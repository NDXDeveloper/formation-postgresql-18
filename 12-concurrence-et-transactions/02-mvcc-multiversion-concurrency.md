🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.2. MVCC (Multiversion Concurrency Control) : Le cœur de PostgreSQL

## Introduction : Le problème de la concurrence

Imaginez une bibliothèque où plusieurs personnes veulent consulter et modifier le même livre en même temps. Comment éviter le chaos ? Comment permettre à quelqu'un de lire le livre pendant qu'un autre le modifie ? C'est exactement le défi que rencontre une base de données lorsque plusieurs utilisateurs travaillent simultanément sur les mêmes données.

Les bases de données doivent résoudre deux problèmes majeurs :

1. **La cohérence** : Garantir que les données restent valides et respectent les règles métier
2. **La concurrence** : Permettre à plusieurs utilisateurs de travailler simultanément sans se bloquer mutuellement

La solution de PostgreSQL à ce défi s'appelle **MVCC : Multiversion Concurrency Control** (Contrôle de Concurrence Multi-versions).

---

## Qu'est-ce que le MVCC ?

### Définition simple

Le **MVCC** est un mécanisme qui permet à PostgreSQL de gérer plusieurs **versions** d'une même ligne de données simultanément. Chaque transaction voit sa propre "version" cohérente de la base de données, comme si elle en avait une photographie privée prise au moment où la transaction a commencé.

### L'approche traditionnelle (sans MVCC)

Dans les systèmes de bases de données plus anciens, la gestion de la concurrence repose sur des **verrous** :

```
Transaction A lit une ligne → La ligne est verrouillée
  ↓
Transaction B essaie de lire la même ligne → Bloquée en attente
  ↓
Transaction A termine → Libère le verrou
  ↓
Transaction B peut enfin lire
```

**Problème** : Les lectures bloquent les écritures et vice-versa. C'est lent et inefficace, surtout pour les applications avec beaucoup de lectures simultanées.

### L'approche MVCC (PostgreSQL)

Avec MVCC, PostgreSQL crée de **nouvelles versions** des lignes au lieu de verrouiller les anciennes :

```
Transaction A lit une ligne (version 1)
  ↓
Transaction B modifie la ligne → Crée une version 2
  ↓
Transaction A continue de voir la version 1 (sa "photographie")
Transaction B voit la version 2 (ses modifications)
  ↓
Les deux transactions travaillent sans se bloquer !
```

**Avantage majeur** : Les lecteurs ne bloquent jamais les écrivains, et les écrivains ne bloquent jamais les lecteurs.

---

## Comment fonctionne MVCC : Les versions de lignes

### Concept de tuple et de version

Dans PostgreSQL, chaque ligne d'une table est appelée un **tuple**. Lorsqu'une ligne est modifiée, PostgreSQL ne remplace pas immédiatement l'ancienne version : il crée une **nouvelle version** du tuple.

**Exemple concret :**

```sql
-- État initial de la table produits
id |   nom    | prix
---|----------|------
1  | Laptop   | 1000
```

Maintenant, deux transactions s'exécutent en parallèle :

**Transaction A (lecture)** :
```sql
BEGIN;
SELECT * FROM produits WHERE id = 1;
-- Voit : Laptop, 1000€
```

**Transaction B (modification)** :
```sql
BEGIN;
UPDATE produits SET prix = 1200 WHERE id = 1;
COMMIT;
```

**Ce qui se passe en interne** :

PostgreSQL crée une **nouvelle version** de la ligne au lieu de modifier l'ancienne :

```
Version 1 (ancienne) : id=1, nom='Laptop', prix=1000  [créée par transaction X]
Version 2 (nouvelle) : id=1, nom='Laptop', prix=1200  [créée par transaction B]
```

**Transaction A continue** :
```sql
-- Transaction A, toujours active
SELECT * FROM produits WHERE id = 1;
-- Voit TOUJOURS : Laptop, 1000€ (version 1)
-- Même après le COMMIT de B !
```

Transaction A voit toujours la version 1 car c'est la version qui existait quand elle a commencé. C'est comme si elle avait une photo de la base de données prise au début de sa transaction.

---

## Les métadonnées de visibilité : xmin et xmax

Pour savoir quelle version d'une ligne est visible pour quelle transaction, PostgreSQL associe des **métadonnées de visibilité** à chaque tuple.

### Les colonnes système cachées

Chaque ligne dans PostgreSQL possède des colonnes système invisibles (que vous ne voyez pas normalement) :

| Colonne | Description |
|---------|-------------|
| **xmin** | ID de la transaction qui a **créé** cette version du tuple |
| **xmax** | ID de la transaction qui a **supprimé/modifié** cette version (0 si toujours active) |
| **ctid** | Identifiant physique du tuple (page, offset) |

### Voir ces colonnes cachées

Vous pouvez afficher ces colonnes avec une requête spéciale :

```sql
SELECT xmin, xmax, ctid, * FROM produits;
```

Résultat possible :

```
xmin | xmax | ctid  | id |   nom    | prix
-----|------|-------|----|----------|------
100  |  0   | (0,1) | 1  | Laptop   | 1000
101  |  0   | (0,2) | 2  | Souris   | 25
```

**Interprétation** :
- La première ligne a été créée par la transaction 100 (xmin = 100)
- Elle n'a pas encore été modifiée ou supprimée (xmax = 0)
- Elle est stockée dans la page 0, position 1 (ctid = (0,1))

### Comment PostgreSQL détermine la visibilité

Pour chaque ligne, PostgreSQL se pose cette question :

**"Est-ce que cette version du tuple est visible pour ma transaction ?"**

La réponse dépend de plusieurs règles :

1. **Si xmin > ID de ma transaction** → La ligne a été créée APRÈS mon début → **Invisible**
2. **Si xmin < ID de ma transaction et xmax = 0** → Ligne créée avant moi et toujours active → **Visible**
3. **Si xmax > 0 et xmax < ID de ma transaction** → Ligne supprimée avant mon début → **Invisible**
4. **Si xmax > ID de ma transaction** → Ligne supprimée après mon début → **Visible** (je vois l'ancienne version)

**Analogie** : Imaginez que vous regardez une vidéo. Chaque transaction voit la base de données à un "instant T" spécifique. Les modifications faites après cet instant sont invisibles pour cette transaction.

---

## Exemple détaillé : Cycle de vie d'une ligne

Suivons le cycle de vie d'une ligne à travers plusieurs modifications :

### Étape 1 : Insertion initiale

```sql
-- Transaction 100
BEGIN;
INSERT INTO employes (id, nom, salaire) VALUES (1, 'Alice', 50000);
COMMIT;
```

**État interne** :

```
Version 1:
  xmin = 100 (créée par transaction 100)
  xmax = 0   (toujours active)
  Données : id=1, nom='Alice', salaire=50000
```

### Étape 2 : Première modification

```sql
-- Transaction 105
BEGIN;
UPDATE employes SET salaire = 55000 WHERE id = 1;
COMMIT;
```

**État interne** :

```
Version 1 (ancienne):
  xmin = 100
  xmax = 105 (invalidée par transaction 105)
  Données : id=1, nom='Alice', salaire=50000

Version 2 (nouvelle):
  xmin = 105 (créée par transaction 105)
  xmax = 0   (toujours active)
  Données : id=1, nom='Alice', salaire=55000
```

PostgreSQL conserve **les deux versions** temporairement !

### Étape 3 : Deuxième modification

```sql
-- Transaction 110
BEGIN;
UPDATE employes SET salaire = 60000 WHERE id = 1;
COMMIT;
```

**État interne** :

```
Version 1 (très ancienne):
  xmin = 100
  xmax = 105
  Données : id=1, nom='Alice', salaire=50000

Version 2 (ancienne):
  xmin = 105
  xmax = 110 (invalidée par transaction 110)
  Données : id=1, nom='Alice', salaire=55000

Version 3 (actuelle):
  xmin = 110 (créée par transaction 110)
  xmax = 0   (toujours active)
  Données : id=1, nom='Alice', salaire=60000
```

Maintenant, il y a **trois versions** de la même ligne !

### Étape 4 : Suppression

```sql
-- Transaction 115
BEGIN;
DELETE FROM employes WHERE id = 1;
COMMIT;
```

**État interne** :

```
Version 3:
  xmin = 110
  xmax = 115 (supprimée par transaction 115)
  Données : id=1, nom='Alice', salaire=60000
```

**Important** : Le `DELETE` ne supprime pas physiquement la ligne ! Il marque simplement xmax pour indiquer qu'elle est supprimée.

---

## Les niveaux d'isolation et MVCC

Le MVCC permet à PostgreSQL d'implémenter différents **niveaux d'isolation** des transactions, qui définissent quel degré de cohérence une transaction peut garantir.

### Les quatre niveaux d'isolation ANSI

PostgreSQL supporte trois des quatre niveaux d'isolation (Read Uncommitted est traité comme Read Committed) :

| Niveau | Dirty Read | Non-Repeatable Read | Phantom Read |
|--------|------------|---------------------|--------------|
| **Read Uncommitted** | ❌ Possible | ❌ Possible | ❌ Possible |
| **Read Committed** | ✅ Impossible | ❌ Possible | ❌ Possible |
| **Repeatable Read** | ✅ Impossible | ✅ Impossible | ❌ Possible* |
| **Serializable** | ✅ Impossible | ✅ Impossible | ✅ Impossible |

*PostgreSQL va plus loin et empêche aussi les Phantom Reads en Repeatable Read.

### 1. Read Committed (par défaut)

C'est le niveau par défaut dans PostgreSQL. Chaque **requête** dans une transaction voit un snapshot de la base de données pris **au moment de la requête**.

```sql
-- Transaction A
BEGIN;  -- Commence à 10:00:00

SELECT salaire FROM employes WHERE id = 1;
-- Voit : 50000€

-- [Pendant ce temps, Transaction B modifie et commit la ligne]

SELECT salaire FROM employes WHERE id = 1;  -- Même transaction, nouvelle requête
-- Voit : 55000€ (la modification de B est visible !)

COMMIT;
```

**Caractéristique** : Les lectures peuvent voir des données différentes au fil de la transaction (Non-Repeatable Read).

### 2. Repeatable Read

La transaction voit un snapshot de la base de données pris **au début de la transaction** (au premier SELECT).

```sql
-- Transaction A
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT salaire FROM employes WHERE id = 1;
-- Voit : 50000€

-- [Transaction B modifie et commit]

SELECT salaire FROM employes WHERE id = 1;  -- Même transaction
-- Voit : TOUJOURS 50000€ (snapshot figé)

COMMIT;
```

**Caractéristique** : Les lectures sont cohérentes durant toute la transaction.

### 3. Serializable

Le niveau le plus strict. PostgreSQL garantit que l'exécution simultanée de transactions sérialisables produit le même résultat que si elles avaient été exécutées une par une, en série.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Si un conflit de sérialisation est détecté, PostgreSQL
-- annule la transaction avec une erreur
```

**Prix à payer** : Risque accru d'erreurs de sérialisation et de transactions annulées.

---

## Snapshots et visibilité des transactions

### Qu'est-ce qu'un snapshot ?

Un **snapshot** (instantané) est une vue cohérente de la base de données à un instant précis. Il contient :

1. **L'ID de transaction actuelle**
2. **La liste des transactions en cours** au moment où le snapshot est pris
3. **La liste des transactions validées (committed)**

Grâce à ces informations, PostgreSQL peut déterminer quelles versions de lignes sont visibles pour votre transaction.

### Comment les snapshots fonctionnent

Quand une transaction démarre (ou quand une requête commence en Read Committed), PostgreSQL prend un snapshot :

```
Snapshot au moment T:
  - Transaction actuelle : 120
  - Transactions en cours : [118, 119]
  - Transactions validées avant T : [1...117]
```

**Règles de visibilité** :

- ✅ Les lignes créées par des transactions < 118 et validées sont **visibles**
- ❌ Les lignes créées par les transactions 118, 119 (en cours) sont **invisibles**
- ❌ Les lignes créées par des transactions > 120 (futures) sont **invisibles**

---

## Les avantages du MVCC

### 1. Haute concurrence

Le principal avantage : **les lecteurs ne bloquent jamais les écrivains**, et vice-versa.

```sql
-- Transaction A (lecture)
BEGIN;
SELECT * FROM produits;  -- Lit des millions de lignes

-- Transaction B (écriture) - En parallèle !
BEGIN;
UPDATE produits SET prix = prix * 1.1;  -- Ne bloque PAS
COMMIT;
```

Les deux transactions s'exécutent sans se bloquer mutuellement.

### 2. Cohérence des lectures

Une transaction voit toujours une vue cohérente de la base de données. Pas de lectures partielles ou incohérentes.

### 3. Pas de lecture de données "sales" (Dirty Reads)

Une transaction ne voit JAMAIS les modifications non validées d'une autre transaction.

```sql
-- Transaction A
BEGIN;
UPDATE produits SET prix = 0 WHERE id = 1;  -- Pas encore commitée

-- Transaction B
BEGIN;
SELECT prix FROM produits WHERE id = 1;
-- Voit l'ANCIEN prix, pas 0 !
```

### 4. Performance pour les lectures

Les opérations de lecture sont extrêmement rapides car elles ne nécessitent généralement pas de verrous.

---

## Les inconvénients du MVCC

### 1. Bloat (gonflement) des tables

Le principal inconvénient : PostgreSQL garde les anciennes versions des lignes, ce qui consomme de l'espace disque.

```sql
-- État initial : 1 ligne, 100 octets
INSERT INTO produits VALUES (1, 'Laptop', 1000);

-- Après 1000 modifications
UPDATE produits SET prix = prix + 1;  -- Répété 1000 fois

-- La table contient maintenant 1001 versions de la même ligne !
-- Espace utilisé : 100KB au lieu de 100 octets
```

**Solution** : Le processus `VACUUM` nettoie les anciennes versions (voir section suivante).

### 2. Overhead de stockage

Chaque tuple nécessite des métadonnées supplémentaires (xmin, xmax, etc.), ce qui augmente légèrement la taille de stockage.

### 3. Complexité du VACUUM

Le processus de nettoyage (VACUUM) doit être géré et configuré correctement, sinon les performances se dégradent.

### 4. Limitations sur les modifications

Les UPDATE et DELETE ne modifient pas en place : ils créent de nouvelles versions. Pour des tables très fréquemment modifiées, cela peut créer beaucoup de versions temporaires.

---

## VACUUM : Le nettoyeur du MVCC

### Pourquoi VACUUM est nécessaire

Reprenons notre exemple précédent avec trois versions d'une ligne :

```
Version 1: xmin=100, xmax=105, salaire=50000
Version 2: xmin=105, xmax=110, salaire=55000
Version 3: xmin=110, xmax=0,   salaire=60000
```

Si plus aucune transaction n'a besoin de voir les versions 1 et 2 (parce que toutes les transactions actives ont un ID > 110), ces versions sont **mortes** (dead tuples) et peuvent être supprimées pour libérer de l'espace.

### Le rôle de VACUUM

`VACUUM` est le processus qui :

1. **Identifie les tuples morts** (versions de lignes qui ne seront jamais plus visibles par aucune transaction)
2. **Marque leur espace comme réutilisable**
3. **Met à jour les statistiques** pour le planificateur de requêtes

**Attention** : `VACUUM` ne rend généralement pas l'espace disque au système d'exploitation. Il marque simplement l'espace comme réutilisable pour de futures insertions.

### Deux types de VACUUM

#### 1. VACUUM (standard)

```sql
VACUUM;              -- Vacuum toutes les tables
VACUUM produits;     -- Vacuum une table spécifique
```

- Fonctionne en **mode non-bloquant**
- Marque l'espace comme réutilisable
- N'empêche pas les opérations concurrentes
- Exécuté automatiquement par **autovacuum**

#### 2. VACUUM FULL

```sql
VACUUM FULL produits;
```

- Réécrit **complètement** la table
- Supprime physiquement les tuples morts et **rend l'espace au système**
- **Prend un verrou exclusif** sur la table (bloque les lectures et écritures !)
- Très lent sur les grandes tables
- À utiliser seulement en cas de bloat extrême

### Autovacuum : Le vacuum automatique

PostgreSQL dispose d'un démon **autovacuum** qui fonctionne en arrière-plan et lance automatiquement VACUUM sur les tables qui en ont besoin.

**Configuration** (dans postgresql.conf) :

```ini
autovacuum = on  # Activé par défaut

# Seuil de déclenchement
autovacuum_vacuum_threshold = 50       # Nb minimum de tuples morts
autovacuum_vacuum_scale_factor = 0.2   # 20% de tuples morts

# Calcul : VACUUM se déclenche si :
# dead_tuples > threshold + (scale_factor * total_tuples)
# Exemple : table de 1000 lignes → 50 + (0.2 * 1000) = 250 tuples morts
```

**Nouveauté PostgreSQL 18** : Autovacuum a été amélioré avec de nouveaux paramètres dynamiques pour mieux gérer les grandes tables.

---

## Transaction ID Wraparound : Un danger silencieux

### Le problème

PostgreSQL utilise des **identifiants de transaction sur 32 bits**, ce qui signifie qu'il peut avoir au maximum 2^32 (environ 4 milliards) d'IDs de transaction avant de "boucler" et de recommencer à 0.

Si PostgreSQL ne nettoie pas les anciennes transactions, il peut arriver un moment où :

```
Transaction actuelle : 4 000 000 000
Ancienne transaction non nettoyée : 100

Après wraparound :
Transaction actuelle : 5 (wraparound ! On repart de 0)
Ancienne transaction : 100 (semble maintenant FUTURE !)
```

**Résultat catastrophique** : Les données anciennes deviennent subitement "invisibles" !

### La solution : Freezing

Pour éviter ce problème, PostgreSQL "gèle" (freeze) les anciennes transactions en remplaçant leur xmin par une valeur spéciale (**FrozenXID**) qui est considérée comme "toujours visible".

```sql
-- VACUUM marque les tuples anciens comme "frozen"
VACUUM FREEZE produits;
```

**Automatique** : Autovacuum effectue automatiquement le freezing quand nécessaire.

**Paramètre important** :

```ini
# Dans postgresql.conf
vacuum_freeze_min_age = 50000000  # Age minimum avant freezing
```

**Signal d'alarme** : PostgreSQL avertit lorsqu'il approche du wraparound et peut même empêcher de nouvelles transactions en cas de danger imminent !

---

## Visualisation du MVCC en action

### Exemple pratique complet

Créons une table et observons MVCC :

```sql
-- Créer une table test
CREATE TABLE comptes (
    id INT PRIMARY KEY,
    nom TEXT,
    solde NUMERIC(10,2)
);

INSERT INTO comptes VALUES (1, 'Alice', 1000.00);

-- Afficher les métadonnées
SELECT xmin, xmax, ctid, * FROM comptes;
```

Résultat :

```
xmin | xmax | ctid  | id | nom   | solde
-----|------|-------|----| ------|--------
200  | 0    | (0,1) | 1  | Alice | 1000.00
```

**Maintenant, ouvrons deux transactions :**

**Terminal 1 (Transaction A)** :
```sql
BEGIN;
SELECT pg_current_xact_id();  -- Retourne : 201
SELECT * FROM comptes WHERE id = 1;
-- Alice, 1000.00
```

**Terminal 2 (Transaction B)** :
```sql
BEGIN;
SELECT pg_current_xact_id();  -- Retourne : 202
UPDATE comptes SET solde = 1500.00 WHERE id = 1;
COMMIT;
```

**Vérifier les versions** :
```sql
SELECT xmin, xmax, ctid, * FROM comptes;
```

Résultat après le UPDATE et avant le VACUUM :

```
xmin | xmax | ctid  | id | nom   | solde
-----|------|-------|----|-------|--------
200  | 202  | (0,1) | 1  | Alice | 1000.00  -- Ancienne version
202  | 0    | (0,2) | 1  | Alice | 1500.00  -- Nouvelle version
```

**Terminal 1 (Transaction A, toujours active)** :
```sql
SELECT * FROM comptes WHERE id = 1;
-- Voit TOUJOURS : Alice, 1000.00 (version avec xmin=200, xmax=202)
-- Car Transaction A (ID=201) a commencé avant Transaction B (ID=202)
```

---

## MVCC et les index

### Les index et les versions multiples

Un détail important : **les index ne stockent pas xmin/xmax**. Ils pointent vers **toutes les versions** d'un tuple.

```
Index sur 'id':
  id=1 → pointe vers (0,1) ET (0,2)
```

Quand PostgreSQL utilise un index, il doit :
1. Suivre le pointeur de l'index
2. Vérifier **toutes les versions** du tuple
3. Appliquer les règles de visibilité pour trouver la bonne version

**Conséquence** : Un index sur une table avec beaucoup de dead tuples peut causer des ralentissements car PostgreSQL doit vérifier de nombreuses versions inutiles.

### Index-Only Scans et VM (Visibility Map)

Pour optimiser les Index-Only Scans, PostgreSQL utilise une **Visibility Map** :

- Une carte qui indique quelles pages de la table contiennent uniquement des tuples visibles par toutes les transactions
- Permet d'éviter d'accéder à la table principale dans certains cas

---

## Résumé des concepts clés

### Architecture MVCC

```
Ligne logique (vue utilisateur)
           |
           | (peut avoir plusieurs versions physiques)
           ↓
Version 1: xmin=100, xmax=105, données=[...]
Version 2: xmin=105, xmax=0,   données=[...]
           ↑
           | (règles de visibilité)
           |
Transactions :
  - Transaction 103 → voit Version 1
  - Transaction 106 → voit Version 2
```

### Flux de vie d'une modification

```
UPDATE table SET col = val
           ↓
Création nouvelle version (xmin = transaction actuelle)
           ↓
Marquage ancienne version (xmax = transaction actuelle)
           ↓
COMMIT
           ↓
Ancienne version devient "dead tuple" (après que toutes les
transactions qui pouvaient la voir se terminent)
           ↓
VACUUM identifie et nettoie les dead tuples
           ↓
Espace réutilisable pour futures insertions
```

---

## Bonnes pratiques liées au MVCC

### 1. Surveillez le bloat

```sql
-- Requête pour détecter le bloat
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
  n_dead_tup,
  n_live_tup,
  round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### 2. Configurez autovacuum correctement

Pour les tables très actives, ajustez les seuils :

```sql
ALTER TABLE ma_table_tres_active SET (
  autovacuum_vacuum_scale_factor = 0.05,  -- Plus agressif (5% au lieu de 20%)
  autovacuum_vacuum_threshold = 1000
);
```

### 3. Évitez les transactions longues

Les transactions qui restent ouvertes longtemps empêchent VACUUM de nettoyer les tuples morts :

```sql
-- MAUVAIS : Transaction ouverte pendant des heures
BEGIN;
SELECT * FROM huge_table;
-- [Application fait un traitement long...]
COMMIT;  -- Des heures plus tard !
```

**Impact** : Pendant ce temps, aucun dead tuple créé après le BEGIN ne peut être nettoyé, causant du bloat.

### 4. Utilisez VACUUM ANALYZE régulièrement

```sql
-- Après de grosses modifications
VACUUM ANALYZE ma_table;
```

### 5. Surveillez l'âge des transactions

```sql
-- Identifier les transactions anciennes
SELECT
  pid,
  usename,
  state,
  age(backend_xid) AS xid_age,
  age(backend_xmin) AS xmin_age,
  query
FROM pg_stat_activity
WHERE backend_xid IS NOT NULL OR backend_xmin IS NOT NULL
ORDER BY greatest(age(backend_xid), age(backend_xmin)) DESC;
```

---

## MVCC vs autres approches

### Comparaison avec le verrouillage pessimiste (MySQL/InnoDB)

**MySQL (mode par défaut)** :
- Verrous au niveau ligne pour les modifications
- Lecteurs et écrivains peuvent se bloquer mutuellement
- Moins de bloat, mais plus de contentions

**PostgreSQL (MVCC)** :
- Versions multiples
- Lecteurs et écrivains ne se bloquent pas
- Plus de bloat, mais meilleure concurrence

### Quand MVCC excelle

- ✅ Applications avec **beaucoup de lectures** simultanées
- ✅ Applications **OLTP** (Online Transaction Processing)
- ✅ Rapports/analytics en parallèle avec des modifications
- ✅ Besoin de **cohérence de lecture** stricte

### Quand MVCC peut poser problème

- ❌ Tables avec **très nombreuses modifications** (high update rate)
- ❌ Transactions **très longues** qui empêchent le nettoyage
- ❌ Ressources disque **limitées** (bloat peut être problématique)

---

## Conclusion

Le **MVCC** est l'innovation majeure qui fait de PostgreSQL un SGBD performant et fiable pour les environnements hautement concurrents. En maintenant plusieurs versions des données, PostgreSQL permet :

- ✅ **Aucun blocage entre lecteurs et écrivains**
- ✅ **Cohérence de lecture** garantie
- ✅ **Niveaux d'isolation** flexibles
- ✅ **Performance** excellente pour les workloads mixtes

Mais cette puissance a un prix :

- ⚠️ **Bloat** à surveiller
- ⚠️ **VACUUM** à configurer correctement
- ⚠️ **Transaction wraparound** à comprendre

Comprendre MVCC est essentiel pour :
- Diagnostiquer les problèmes de performance
- Configurer correctement PostgreSQL
- Concevoir des schémas de données efficaces
- Optimiser les workloads transactionnels

Dans la prochaine section (12.3), nous approfondirons les **niveaux d'isolation** et les **anomalies transactionnelles** qu'ils permettent d'éviter.

---

**Points clés à retenir :**

- 🔑 MVCC = Multiversion Concurrency Control = Plusieurs versions d'une ligne
- 🔑 xmin = transaction créatrice, xmax = transaction suppressive
- 🔑 Les lecteurs ne bloquent jamais les écrivains
- 🔑 VACUUM nettoie les versions mortes (dead tuples)
- 🔑 Le bloat est l'effet secondaire du MVCC
- 🔑 Les snapshots déterminent quelle version est visible
- 🔑 Évitez les transactions longues pour permettre le nettoyage

⏭️ [Niveaux d'isolation ANSI (Read Uncommitted, Read Committed, Repeatable Read, Serializable)](/12-concurrence-et-transactions/03-niveaux-isolation-ansi.md)
