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
-- Transaction A, toujours active (niveau Read Committed par défaut)
SELECT * FROM produits WHERE id = 1;
-- Voit : Laptop, 1200€ (version 2) ← Voit le COMMIT de B !
```

> ⚠️ **Attention au niveau d'isolation** : En **Read Committed** (défaut de PostgreSQL), chaque **instruction** prend un nouveau snapshot. Donc après le COMMIT de B, un nouveau SELECT dans A verra le prix mis à jour (1200€).  
>  
> En **Repeatable Read**, Transaction A verrait TOUJOURS 1000€ pendant toute sa durée, car elle utilise un snapshot figé au début de la transaction. C'est ce niveau qui se comporte comme "une photo prise au début de la transaction".  
>  
> Les niveaux d'isolation seront détaillés dans la section 12.3.

---

## Les métadonnées de visibilité : xmin et xmax

Pour savoir quelle version d'une ligne est visible pour quelle transaction, PostgreSQL associe des **métadonnées de visibilité** à chaque tuple.

### Les colonnes système cachées

Chaque ligne dans PostgreSQL possède des colonnes système invisibles (qui n'apparaissent pas dans un `SELECT *`) :

| Colonne | Description |
|---------|-------------|
| **xmin** | XID de la transaction qui a **créé** cette version du tuple |
| **xmax** | XID de la transaction qui a **invalidé** cette version (UPDATE ou DELETE). Vaut **`0`** (`InvalidTransactionId`) tant que la version n'a été ni modifiée ni supprimée |
| **ctid** | Identifiant physique du tuple sous la forme `(page, offset)` |
| **tableoid** | OID de la table qui contient ce tuple (utile pour les requêtes sur des hiérarchies d'héritage / partitionnement) |
| **cmin** / **cmax** | Numéros de commande à l'intérieur de la transaction (utiles surtout pour les triggers et fonctions PL/pgSQL) |

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

**« Cette version du tuple est-elle visible pour ma transaction ? »**

La réponse dépend de l'état (commit / abort / encore en cours) des transactions `xmin` et `xmax`, comparé au **snapshot** de la transaction en cours. En simplifiant fortement, une version est visible si :

1. **`xmin` est validé (committé)** ET ne fait pas partie des transactions en cours dans le snapshot → la création est visible.  
2. **ET** soit `xmax = 0` (jamais supprimé), soit `xmax` est encore en cours ou a été annulé (`ROLLBACK`), soit `xmax` a été validé après le snapshot → la suppression n'est pas visible.

> ⚠️ **La règle réelle est plus complexe** : PostgreSQL consulte le **commit log** (`pg_xact`, anciennement `pg_clog`) pour connaître l'état de chaque XID. La comparaison directe `xmin < xid_courant` est trompeuse à cause du wraparound des XID 32 bits : PostgreSQL utilise donc l'arithmétique modulaire et la notion d'« âge » de transaction, pas une comparaison numérique brute. Les snapshots conservent en plus la liste explicite des XID encore actifs au moment où ils ont été pris.

**Analogie** : Imaginez que vous regardez une vidéo. Chaque transaction voit la base de données à un « instant T » spécifique (son snapshot). Les modifications faites par des transactions postérieures à cet instant sont invisibles pour cette transaction.

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

**État interne** (tant qu'aucun `VACUUM` n'est passé, **les quatre versions cohabitent** sur le disque) :

```
Version 1 :  xmin=100, xmax=105               (id=1, salaire=50000)  – morte  
Version 2 :  xmin=105, xmax=110               (id=1, salaire=55000)  – morte  
Version 3 :  xmin=110, xmax=115 (DELETE)      (id=1, salaire=60000)  – morte  
```

**Important** : Le `DELETE` ne supprime pas physiquement la ligne ! Il marque simplement `xmax` de la version actuelle (V3) avec le XID 115 pour indiquer qu'elle est supprimée. Les anciennes versions (V1, V2) sont toujours présentes dans le fichier de la table — elles le restent jusqu'à ce que `VACUUM` constate qu'aucune transaction active ne peut plus les voir et libère leur espace.

---

## Les niveaux d'isolation et MVCC

Le MVCC permet à PostgreSQL d'implémenter différents **niveaux d'isolation** des transactions, qui définissent quel degré de cohérence une transaction peut garantir.

### Les quatre niveaux d'isolation ANSI

La norme ANSI SQL définit quatre niveaux d'isolation. Le tableau ci-dessous indique les anomalies **autorisées par la norme** pour chaque niveau :

| Niveau | Dirty Read | Non-Repeatable Read | Phantom Read |
|--------|------------|---------------------|--------------|
| **Read Uncommitted** | autorisé | autorisé | autorisé |
| **Read Committed** | interdit | autorisé | autorisé |
| **Repeatable Read** | interdit | interdit | autorisé |
| **Serializable** | interdit | interdit | interdit |

**Spécificités de PostgreSQL** :

- PostgreSQL n'implémente que **trois** niveaux distincts. Demander `READ UNCOMMITTED` est accepté par le parseur mais traité **exactement comme `READ COMMITTED`** : grâce à MVCC, PostgreSQL ne montre **jamais** de données non validées (pas de *dirty read* possible, quel que soit le niveau).  
- En `REPEATABLE READ`, PostgreSQL est **plus strict que la norme** : il empêche aussi les *phantom reads* (lignes nouvelles apparues entre deux lectures), grâce au snapshot figé pour toute la transaction. On parle parfois pour PostgreSQL de *snapshot isolation*.  
- `SERIALIZABLE` est implémenté via **SSI** (*Serializable Snapshot Isolation*) : pas de verrouillage prédicatif, mais une détection de conflits qui peut abandonner une transaction avec l'erreur `serialization_failure` (SQLSTATE `40001`) — il faut alors la **rejouer** côté applicatif.

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

Un **snapshot** (instantané) est une description compacte de l'état des transactions à un instant précis. Concrètement, PostgreSQL stocke :

1. **`xmin`** : le plus petit XID encore **actif** (toute transaction antérieure est forcément terminée — committée ou annulée).  
2. **`xmax`** : un XID juste au-dessus du plus grand XID **déjà assigné** au moment du snapshot (toute transaction ≥ `xmax` est postérieure et donc invisible).  
3. **`xip_list`** : la liste explicite des XID **encore actifs** dans l'intervalle `[xmin, xmax[`.

Le snapshot ne contient **pas** la liste de toutes les transactions committées — ce serait gigantesque. C'est l'inverse : il liste les transactions à **ignorer**. Pour les XID qui ne sont ni dans `xip_list` ni ≥ `xmax`, PostgreSQL va consulter le **commit log** (`pg_xact`) pour savoir si elles ont été committées ou annulées.

Vous pouvez observer le snapshot courant :

```sql
SELECT pg_current_snapshot();
-- Format : xmin:xmax:xip_list, p. ex. "1234:1240:1236,1238"
```

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

Le principal inconvénient : tant que `VACUUM` n'est pas passé, **PostgreSQL conserve les anciennes versions** des lignes, ce qui consomme de l'espace disque.

```sql
-- Une ligne, mettons que sa taille effective sur disque est ~ 50 octets
-- (header de tuple ≈ 24 octets + colonnes + alignement)
INSERT INTO produits VALUES (1, 'Laptop', 1000);

-- Boucler 1000 UPDATE sur cette même ligne
-- (par exemple via un client qui répète la requête)
UPDATE produits SET prix = prix + 1 WHERE id = 1;
-- … répété 1000 fois …

-- Tant qu'aucun VACUUM n'est passé, la table contient ~1001 versions
-- de cette ligne. La taille passe d'environ 50 octets utiles à
-- ~50 ko de stockage réel — c'est le BLOAT.
```

**Atténuations** :
- `VACUUM` (automatique via autovacuum) nettoie les versions mortes (voir section suivante).  
- Si l'`UPDATE` ne touche **aucune colonne indexée** et qu'il y a de la place dans la page, PostgreSQL peut faire un **HOT update** (cf. section précédente) : pas d'écriture dans les index et nettoyage à la volée.  
- Réduire `fillfactor` réserve de l'espace dans chaque page pour favoriser HOT et le pruning local.

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

**Attention** : `VACUUM` ne rend **généralement pas** l'espace disque au système d'exploitation. Il marque l'espace comme réutilisable pour de futures insertions.

> 💡 **Exception** : si les **dernières pages** du fichier de table sont entièrement vides après le nettoyage, VACUUM peut les tronquer et restituer cet espace à l'OS. Ce comportement est piloté en PG 18 par la GUC `vacuum_truncate` (et par le paramètre du même nom au niveau de chaque table). Pour récupérer l'espace au milieu du fichier, seul `VACUUM FULL` (ou `pg_repack` côté outillage) fonctionne.

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

**Nouveautés PostgreSQL 18 pour l'autovacuum** :

- `autovacuum_worker_slots` (nouveau) : nombre maximal de slots de workers réservés. `autovacuum_max_workers` peut désormais être ajusté **à chaud** jusqu'à cette borne, sans redémarrage du serveur.  
- `autovacuum_vacuum_max_threshold` (nouveau) : seuil **absolu** en nombre de tuples morts qui déclenche l'autovacuum, en plus du pourcentage. Très utile sur les très grosses tables où 20% représente des millions de lignes.  
- `vacuum_max_eager_freeze_failure_rate` (nouveau) : autorise un *eager freezing* — geler quelques pages déjà `all-visible` pendant un VACUUM normal — pour réduire le coût d'un futur freeze massif lié au wraparound.  
- `vacuum_truncate` (promu en GUC) : contrôle au niveau serveur la troncation de la fin de fichier lors du VACUUM (option qui n'existait jusque-là qu'au niveau table).  
- `VACUUM` et `ANALYZE` traitent désormais **par défaut** les tables filles d'un héritage / partitionnement. Pour retrouver l'ancien comportement (ne traiter que la table parente), utiliser le nouveau mot-clé : `VACUUM ONLY parent_table;` ou `ANALYZE ONLY parent_table;`.  
- Nouvelles colonnes dans `pg_stat_all_tables` : `total_vacuum_time`, `total_autovacuum_time`, `total_analyze_time`, `total_autoanalyze_time` pour mesurer précisément le coût de la maintenance.

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

**Automatique** : autovacuum effectue automatiquement le freezing quand nécessaire, piloté par un trio de paramètres :

```ini
# Dans postgresql.conf

# Âge minimum d'un XID pour qu'un VACUUM le frige (50 M par défaut)
vacuum_freeze_min_age = 50000000

# Quand la "table age" dépasse ce seuil, un VACUUM agressif fige tout (150 M défaut)
vacuum_freeze_table_age = 150000000

# Si l'âge d'une table dépasse ce seuil, autovacuum est FORCÉ à passer
# pour éviter le wraparound — même si l'autovacuum est désactivé !
autovacuum_freeze_max_age = 200000000  # 200 M par défaut, max 2 G
```

**Étapes de l'alerte côté serveur** (sur le XID restant avant wraparound) :

| XID restants | Comportement du serveur |
|--------------|------------------------|
| < 40 M | Warning dans les logs : `WARNING: database "X" must be vacuumed within N transactions` |
| < 3 M | Le serveur **refuse les écritures qui consomment des XID** : `ERROR: database is not accepting commands that assign new XIDs to avoid wraparound data loss in database "X"`. Les transactions purement en lecture seule continuent de fonctionner. |
| 0 | Wraparound : perte de données silencieuse — ce niveau **ne devrait jamais être atteint** grâce aux protections ci-dessus |

**Surveillance proactive** :

```sql
-- Âge maximum par base (XID consommés depuis le dernier freeze)
SELECT datname,
       age(datfrozenxid)                  AS xid_age,
       2000000000 - age(datfrozenxid)     AS marge_avant_wraparound
  FROM pg_database
 ORDER BY xid_age DESC;
```

Surveillez cette métrique : si `xid_age` approche les 1,5 milliards sur une base de production, c'est l'heure de planifier un `VACUUM FREEZE` manuel ou de booster l'autovacuum.

> 💡 **Nouveauté à connaître** : la communauté travaille depuis plusieurs versions à un passage en XID 64 bits qui supprimerait définitivement le risque de wraparound. En PG 18, le format reste **32 bits** ; le passage est attendu pour une version future.

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

**Maintenant, ouvrons deux transactions** — Transaction A en **Repeatable Read** pour figer son snapshot et bien voir l'isolation MVCC :

**Terminal 1 (Transaction A)** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;  
SELECT pg_current_xact_id();  -- Retourne : 201 (XID8 sur une nouvelle base)  
SELECT * FROM comptes WHERE id = 1;  
-- Alice, 1000.00  → fixe le snapshot de A à ce moment
```

**Terminal 2 (Transaction B)** :
```sql
BEGIN;  
SELECT pg_current_xact_id();  -- Retourne : 202  
UPDATE comptes SET solde = 1500.00 WHERE id = 1;  
COMMIT;  
```

**Vérifier les versions** (dans une 3ᵉ session, après le COMMIT de B) :
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
-- Le snapshot de A a été pris avant le COMMIT de B (XID 202),
-- donc A continue de voir l'ancienne version, même si elle a été
-- "marquée" supprimée par B.
```

> 💡 **Si A avait été en Read Committed** (défaut), le second SELECT aurait pris un **nouveau snapshot** et donc vu `1500.00` (le COMMIT de B est désormais visible). C'est précisément la différence entre les deux niveaux que vous verrez en détail dans la section 12.3.

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

### HOT updates : éviter d'écrire dans les index

Un `UPDATE` classique en MVCC crée une nouvelle version de tuple, et **chaque index** sur la table doit recevoir un nouvel enregistrement pointant vers cette nouvelle version — coûteux en E/S, surtout sur les tables très indexées.

PostgreSQL optimise ce cas avec le **HOT update** (*Heap-Only Tuple update*) : si l'UPDATE remplit deux conditions, il évite complètement de toucher aux index.

**Conditions pour qu'un UPDATE soit HOT** :

1. **Aucune colonne indexée n'est modifiée** par l'UPDATE.  
2. La **nouvelle version tient dans la même page** que l'ancienne (PostgreSQL réserve un peu d'espace libre dans chaque page via le `fillfactor`).

**Comment ça marche** : la nouvelle version est chaînée à l'ancienne via un pointeur intra-page (`t_ctid`). Les index continuent de pointer vers l'ancien tuple, qui sert de « tête de chaîne » ; PostgreSQL suit la chaîne pour atteindre la version actuelle.

**Gains concrets** :
- Pas de mise à jour des index → moins d'écritures dans le WAL et sur disque.
- Le nettoyage des versions mortes peut se faire à la volée (HOT pruning) sans VACUUM complet.

**Pour favoriser les HOT updates** sur une table très modifiée, réduisez le `fillfactor` (par défaut 100% pour les tables, 90% pour les index B-tree) afin de laisser de la place dans chaque page :

```sql
ALTER TABLE compteurs SET (fillfactor = 80);
-- Important : ce paramètre n'affecte que les nouvelles pages.
-- Les pages déjà pleines à 100% ne seront pas redécoupées tant que
-- la table n'est pas réécrite (VACUUM FULL, CLUSTER ou pg_repack).
```

> 💡 **Vérifier** : la colonne `n_tup_hot_upd` de `pg_stat_user_tables` indique combien d'UPDATE ont pu utiliser HOT. Un ratio élevé `n_tup_hot_upd / n_tup_upd` signe une table bien optimisée.

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

### Comparaison avec InnoDB (MySQL)

InnoDB implémente lui aussi **MVCC**, mais avec une approche **différente** : les anciennes versions des lignes sont stockées dans un **undo log** (segment de rollback) plutôt qu'à côté de la version courante dans la table elle-même.

**InnoDB (MySQL)** :
- Stockage : version courante en place + anciennes versions dans l'undo log  
- Lecteurs et écrivains ne se bloquent pas (lecture cohérente non bloquante)  
- L'undo log doit être purgé régulièrement, sinon il grossit  
- Les UPDATE in-place sont possibles : moins de fragmentation de table, mais croissance de l'undo

**PostgreSQL (MVCC)** :
- Stockage : toutes les versions cohabitent dans les pages de la table (xmin/xmax sur chaque tuple)  
- Lecteurs et écrivains ne se bloquent pas  
- Bloat de la table elle-même : VACUUM doit nettoyer les versions mortes  
- Pas de modification in-place : tout UPDATE crée une nouvelle version

> 💡 **Les deux moteurs adoptent la philosophie « les lecteurs ne bloquent pas les écrivains »** — la différence se situe dans le stockage et le nettoyage. PostgreSQL paye en bloat de table ; InnoDB paye en croissance de l'undo. Ce sont deux compromis valides du même paradigme MVCC.

Les SGBD qui utilisent réellement un **verrouillage pessimiste** par défaut pour les lectures (lectures bloquantes lorsqu'il y a une écriture) sont aujourd'hui rares — c'est par exemple le comportement historique de SQL Server sans l'option `READ_COMMITTED_SNAPSHOT`, ou le mode `LOCK TABLES` explicite sur les vieux moteurs MyISAM.

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

- 🔑 MVCC = Multiversion Concurrency Control = plusieurs versions d'une ligne coexistent dans la table  
- 🔑 `xmin` = transaction qui a **créé** la version ; `xmax` = transaction qui l'a **invalidée** (UPDATE ou DELETE)  
- 🔑 Les lecteurs ne bloquent jamais les écrivains, et inversement  
- 🔑 Un **snapshot** = `xmin:xmax:xip_list` (voir `pg_current_snapshot()`)  
- 🔑 VACUUM nettoie les versions mortes (dead tuples) ; autovacuum le fait automatiquement  
- 🔑 Le **bloat** est l'effet secondaire du MVCC — surveillez `n_dead_tup`  
- 🔑 **HOT update** = pas d'écriture des index si aucune colonne indexée n'est modifiée et que la nouvelle version tient dans la page (favoriser via `fillfactor < 100`)  
- 🔑 Évitez les transactions longues : elles empêchent VACUUM et bloquent le freezing anti-wraparound  
- 🔑 PG 18 : nouveautés autovacuum (`autovacuum_worker_slots`, `autovacuum_vacuum_max_threshold`, `vacuum_max_eager_freeze_failure_rate`)

⏭️ [Niveaux d'isolation ANSI (Read Uncommitted, Read Committed, Repeatable Read, Serializable)](/12-concurrence-et-transactions/03-niveaux-isolation-ansi.md)
