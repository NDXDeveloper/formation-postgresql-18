🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.1. Stratégies de Scan : Séquentiel vs Index vs Bitmap vs Index-Only

## Introduction

Lorsque PostgreSQL doit récupérer des données dans une table, il dispose de plusieurs **stratégies de scan** (méthodes de lecture) pour accéder aux informations demandées. Le choix de la stratégie utilisée a un impact direct sur les performances de vos requêtes.

Le **planificateur de requêtes** (query planner) de PostgreSQL analyse chaque requête et choisit automatiquement la stratégie la plus efficace en fonction de plusieurs critères :
- La taille de la table
- Le nombre de lignes à récupérer
- La présence d'index
- Les statistiques de la table
- Le coût estimé de chaque méthode

Comprendre ces différentes stratégies vous permettra de :
- Mieux interpréter les plans d'exécution (EXPLAIN)
- Créer des index pertinents
- Optimiser vos requêtes
- Anticiper les problèmes de performance

---

## 1. Sequential Scan (Scan Séquentiel)

### Concept

Le **Sequential Scan** (ou **Seq Scan**) est la stratégie la plus simple : PostgreSQL lit **toutes les lignes de la table**, du début à la fin, de manière séquentielle.

**Analogie** : Imaginez que vous cherchez un nom spécifique dans un annuaire téléphonique. Le scan séquentiel reviendrait à lire l'annuaire page par page, ligne par ligne, depuis la première page jusqu'à la dernière.

### Fonctionnement

```
Table : employes
┌────────────────────────────────┐
│ Bloc 1  │ Ligne 1, 2, 3, ...   │ ← Lecture
│ Bloc 2  │ Ligne N, N+1, ...    │ ← Lecture
│ Bloc 3  │ Ligne M, M+1, ...    │ ← Lecture
│ ...     │ ...                  │ ← Lecture
└────────────────────────────────┘
```

PostgreSQL parcourt chaque **bloc** (page de données, généralement 8 KB) de la table et examine chaque ligne pour déterminer si elle correspond aux critères de la requête.

### Quand PostgreSQL utilise-t-il le Seq Scan ?

Le scan séquentiel est privilégié dans les cas suivants :

1. **Table petite** : Si la table contient peu de données (quelques centaines de lignes), le scan séquentiel est souvent plus rapide qu'un accès via index.

2. **Sélection de nombreuses lignes** : Si votre requête récupère une grande partie de la table (généralement > 5-10%), le scan séquentiel est plus efficace.

   ```sql
   -- Récupère 80% des employés → Seq Scan probable
   SELECT * FROM employes WHERE salaire > 30000;
   ```

3. **Absence d'index** : Si aucun index n'existe sur les colonnes utilisées dans le WHERE.

4. **Table non analysée** : Si les statistiques de la table ne sont pas à jour, PostgreSQL peut choisir un Seq Scan par défaut.

### Avantages

- ✅ **Simple et prévisible** : Pas de structure de données complexe à gérer
- ✅ **Efficace pour grandes sélections** : Lecture continue des blocs, très performant en I/O séquentiel
- ✅ **Pas de maintenance** : Aucun index à créer ou maintenir

### Inconvénients

- ❌ **Lent sur grandes tables** : Si la table contient des millions de lignes et que vous cherchez quelques enregistrements, le Seq Scan lit inutilement toutes les données
- ❌ **Consomme beaucoup d'I/O** : Lit tous les blocs même si peu de lignes correspondent

### Exemple

```sql
-- Table avec 1 million de lignes
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INT,
    montant NUMERIC,
    date_commande DATE
);

-- Requête sans index sur client_id
EXPLAIN SELECT * FROM commandes WHERE client_id = 12345;
```

**Plan d'exécution** :
```
Seq Scan on commandes  (cost=0.00..18334.00 rows=1 width=20)
  Filter: (client_id = 12345)
```

PostgreSQL lit toute la table séquentiellement et filtre les lignes où `client_id = 12345`.

---

## 2. Index Scan (Scan par Index)

### Concept

L'**Index Scan** utilise un **index** pour localiser rapidement les lignes qui correspondent aux critères de recherche, puis accède directement à ces lignes dans la table.

**Analogie** : C'est comme utiliser l'index alphabétique à la fin d'un livre. Vous cherchez le mot "PostgreSQL" dans l'index, qui vous indique les numéros de pages (4, 23, 156), puis vous allez directement à ces pages.

### Fonctionnement

```
Index B-Tree sur client_id
┌─────────────────────┐
│ 10001 → Bloc 5      │
│ 12345 → Bloc 42     │ ← Recherche dans l'index
│ 15678 → Bloc 103    │
└─────────────────────┘
         ↓
Table : commandes
┌────────────────────────────────┐
│ Bloc 42 │ client_id=12345 ...  │ ← Accès direct
└────────────────────────────────┘
```

1. **Recherche dans l'index** : PostgreSQL parcourt l'index (généralement un arbre B-Tree) pour trouver les valeurs correspondantes
2. **Récupération des pointeurs** : L'index contient des pointeurs (TID - Tuple ID) vers les lignes de la table
3. **Accès aux lignes** : PostgreSQL accède directement aux blocs de la table contenant les lignes

### Quand PostgreSQL utilise-t-il l'Index Scan ?

1. **Sélection de peu de lignes** : Si la requête récupère une petite portion de la table (< 5-10%)

   ```sql
   -- Récupère 0.01% des lignes → Index Scan probable
   SELECT * FROM commandes WHERE client_id = 12345;
   ```

2. **Index sur les colonnes du WHERE** : Un index doit exister sur les colonnes utilisées dans les conditions

3. **Ordre de tri demandé** : Si la requête utilise ORDER BY sur une colonne indexée

   ```sql
   SELECT * FROM employes ORDER BY nom LIMIT 10;
   -- Si index sur 'nom' → Index Scan
   ```

### Avantages

- ✅ **Très rapide pour sélections ciblées** : Accès direct aux lignes pertinentes
- ✅ **Réduit considérablement l'I/O** : Ne lit que les blocs nécessaires
- ✅ **Efficace pour les tris** : Si l'index correspond à l'ORDER BY

### Inconvénients

- ❌ **Coût de maintenance** : Les index doivent être mis à jour à chaque INSERT/UPDATE/DELETE
- ❌ **Espace disque supplémentaire** : Les index occupent de l'espace
- ❌ **Random I/O** : Accès aléatoires aux blocs de la table (plus lent que l'I/O séquentiel)
- ❌ **Inefficace pour grandes sélections** : Si trop de lignes correspondent, le Seq Scan devient plus rapide

### Exemple

```sql
-- Création d'un index
CREATE INDEX idx_commandes_client ON commandes(client_id);

-- Requête avec index
EXPLAIN SELECT * FROM commandes WHERE client_id = 12345;
```

**Plan d'exécution** :
```
Index Scan using idx_commandes_client on commandes  (cost=0.42..8.44 rows=1 width=20)
  Index Cond: (client_id = 12345)
```

PostgreSQL utilise l'index pour accéder directement aux lignes.

---

## 3. Bitmap Scan (Scan par Bitmap)

### Concept

Le **Bitmap Scan** est une stratégie **hybride** entre le Seq Scan et l'Index Scan. Il se déroule en deux phases :

1. **Bitmap Index Scan** : Construction d'une bitmap (carte de bits) en mémoire indiquant quels blocs contiennent des lignes pertinentes
2. **Bitmap Heap Scan** : Lecture séquentielle des blocs identifiés

**Analogie** : Imaginez que vous cherchez plusieurs mots dans un livre. Au lieu d'aller directement à chaque page individuellement (Index Scan), vous créez d'abord une liste de toutes les pages contenant au moins un des mots, puis vous lisez ces pages dans l'ordre (plus efficace).

### Fonctionnement

```
Phase 1 : Bitmap Index Scan
Index B-Tree sur client_id
┌─────────────────────┐
│ 12345 → Bloc 42     │ ← Scan de l'index
│ 12346 → Bloc 42     │
│ 12347 → Bloc 103    │
└─────────────────────┘
         ↓
Bitmap en mémoire
┌────────────────────┐
│ Bloc 42  : 1       │ (contient des lignes)
│ Bloc 43  : 0       │
│ Bloc 103 : 1       │
└────────────────────┘

Phase 2 : Bitmap Heap Scan
Table : commandes
┌────────────────────────────────┐
│ Bloc 42  │ Lignes avec 12345   │ ← Lecture séquentielle
│ Bloc 103 │ Lignes avec 12347   │ ← Lecture séquentielle
└────────────────────────────────┘
```

### Quand PostgreSQL utilise-t-il le Bitmap Scan ?

Le Bitmap Scan est utilisé dans les situations intermédiaires :

1. **Sélection modérée** : Trop de lignes pour un Index Scan efficient, mais trop peu pour un Seq Scan (généralement 1-10% de la table)

2. **Plusieurs index combinés** : Lorsque plusieurs conditions utilisent différents index

   ```sql
   -- Deux index : idx_client et idx_date
   SELECT * FROM commandes
   WHERE client_id = 12345 AND date_commande > '2024-01-01';
   ```

   PostgreSQL peut créer deux bitmaps (une par index) et les combiner avec des opérations AND/OR.

3. **Réduction des Random I/O** : Le Bitmap Scan transforme les accès aléatoires (Random I/O) en accès plus séquentiels, ce qui est plus performant

### Avantages

- ✅ **Combine plusieurs index** : Permet d'utiliser plusieurs index simultanément (BitmapAnd, BitmapOr)
- ✅ **Optimise l'I/O** : Lit chaque bloc une seule fois, dans l'ordre séquentiel
- ✅ **Bon compromis** : Plus rapide qu'un Seq Scan, moins de Random I/O qu'un Index Scan

### Inconvénients

- ❌ **Mémoire requise** : La bitmap doit être stockée en mémoire (limitée par `work_mem`)
- ❌ **Moins précis** : Travaille au niveau du bloc, pas de la ligne (perd de la granularité)
- ❌ **Complexité** : Plus complexe à comprendre et à analyser

### Exemple

```sql
EXPLAIN SELECT * FROM commandes WHERE client_id BETWEEN 10000 AND 12000;
```

**Plan d'exécution** :
```
Bitmap Heap Scan on commandes  (cost=104.00..5234.50 rows=5000 width=20)
  Recheck Cond: ((client_id >= 10000) AND (client_id <= 12000))
  ->  Bitmap Index Scan on idx_commandes_client  (cost=0.00..102.75 rows=5000 width=0)
        Index Cond: ((client_id >= 10000) AND (client_id <= 12000))
```

1. **Bitmap Index Scan** : Parcourt l'index et construit la bitmap
2. **Bitmap Heap Scan** : Lit les blocs identifiés dans la table
3. **Recheck Cond** : Vérifie à nouveau la condition car la bitmap travaille au niveau bloc

---

## 4. Index-Only Scan (Scan Index Seul)

### Concept

L'**Index-Only Scan** est une optimisation puissante : PostgreSQL récupère **toutes les données nécessaires directement depuis l'index**, sans jamais accéder à la table.

**Analogie** : Vous cherchez uniquement le numéro de téléphone d'une personne. Au lieu de consulter l'annuaire complet (table), vous pouvez obtenir l'information directement depuis l'index alphabétique (index) si celui-ci contient déjà le numéro.

### Fonctionnement

```
Index B-Tree sur (nom, prenom, telephone)
┌──────────────────────────────────┐
│ Dupont, Marie, 0601020304        │ ← Toutes les données sont ici
│ Martin, Paul, 0612345678         │
│ Durand, Alice, 0698765432        │
└──────────────────────────────────┘
         ↓
    Réponse directe
    (pas d'accès à la table)

Table : clients
┌────────────────────────────────┐
│ (non accédée)                  │
└────────────────────────────────┘
```

### Conditions pour un Index-Only Scan

Pour qu'un Index-Only Scan soit possible, **toutes** les conditions suivantes doivent être remplies :

1. **Colonnes SELECT dans l'index** : Toutes les colonnes demandées dans le SELECT doivent être présentes dans l'index

   ```sql
   -- Index sur (client_id, date_commande)

   -- ✅ Index-Only Scan possible
   SELECT client_id, date_commande FROM commandes WHERE client_id = 12345;

   -- ❌ Index-Only Scan impossible (montant n'est pas dans l'index)
   SELECT client_id, montant FROM commandes WHERE client_id = 12345;
   ```

2. **Visibility Map à jour** : PostgreSQL utilise une **Visibility Map** pour savoir si un tuple dans un bloc est visible par toutes les transactions. Si le VACUUM n'a pas été exécuté récemment, PostgreSQL doit vérifier la table (Heap Fetches).

3. **Index couvrant** : L'index doit "couvrir" toutes les colonnes nécessaires (on parle de **covering index**)

### Covering Index avec INCLUDE

PostgreSQL permet de créer des **index couvrants** avec la clause `INCLUDE` :

```sql
-- Index qui inclut des colonnes supplémentaires
CREATE INDEX idx_commandes_covering ON commandes(client_id) INCLUDE (montant, date_commande);

-- Index-Only Scan possible
SELECT client_id, montant, date_commande
FROM commandes
WHERE client_id = 12345;
```

**Différence avec index multi-colonnes** :
- **Index multi-colonnes** : `CREATE INDEX ON table(col1, col2, col3)` → col2 et col3 sont utilisables pour le filtrage
- **Index INCLUDE** : `CREATE INDEX ON table(col1) INCLUDE (col2, col3)` → col2 et col3 sont stockées mais pas utilisables pour le filtrage (seulement pour l'Index-Only Scan)

### Quand PostgreSQL utilise-t-il l'Index-Only Scan ?

1. **Requêtes analytiques simples** :
   ```sql
   SELECT COUNT(*) FROM commandes WHERE client_id = 12345;
   ```

2. **Projections limitées** :
   ```sql
   SELECT nom, prenom FROM clients WHERE nom = 'Dupont';
   -- Si index sur (nom, prenom)
   ```

3. **Agrégations sur colonnes indexées** :
   ```sql
   SELECT MAX(date_commande) FROM commandes WHERE client_id = 12345;
   ```

### Avantages

- ✅ **Performance maximale** : Pas d'accès disque à la table
- ✅ **Moins d'I/O** : Seul l'index est lu
- ✅ **Cache efficace** : Les index sont généralement plus petits et restent en cache

### Inconvénients

- ❌ **Index plus volumineux** : Les index couvrants (INCLUDE) prennent plus d'espace
- ❌ **Maintenance plus coûteuse** : Index plus gros à maintenir
- ❌ **Dépendance au VACUUM** : Nécessite une Visibility Map à jour

### Exemple

```sql
-- Création d'un index couvrant
CREATE INDEX idx_commandes_io ON commandes(client_id) INCLUDE (montant);

-- Requête
EXPLAIN (ANALYZE, BUFFERS)
SELECT client_id, montant FROM commandes WHERE client_id = 12345;
```

**Plan d'exécution** :
```
Index Only Scan using idx_commandes_io on commandes  (cost=0.42..8.44 rows=1 width=12)
  Index Cond: (client_id = 12345)
  Heap Fetches: 0
```

**Heap Fetches: 0** indique que PostgreSQL n'a pas eu besoin d'accéder à la table (Index-Only Scan pur).

Si `Heap Fetches > 0`, cela signifie que PostgreSQL a dû vérifier certaines lignes dans la table (Visibility Map pas à jour) → Exécutez VACUUM.

---

## Comparaison des Stratégies

| Stratégie | Cas d'usage | Performance | I/O | Conditions |
|-----------|-------------|-------------|-----|------------|
| **Seq Scan** | Grande sélection (>10%) ou petite table | Lent sur grandes tables | Séquentiel (efficace) | Aucun index requis |
| **Index Scan** | Petite sélection (<5%) | Très rapide | Aléatoire (Random) | Index sur colonnes WHERE |
| **Bitmap Scan** | Sélection modérée (1-10%) ou multi-index | Bon compromis | Semi-séquentiel | Index disponibles |
| **Index-Only Scan** | Colonnes SELECT dans l'index | Optimal | Minimal (index seul) | Index couvrant + VACUUM |

---

## Comment Choisir la Bonne Stratégie ?

Le **planificateur PostgreSQL** choisit automatiquement la stratégie optimale en fonction :
- Des **statistiques de la table** (mises à jour par ANALYZE)
- Du **coût estimé** de chaque stratégie
- De la **sélectivité** de vos conditions WHERE
- De la **présence et qualité des index**

### Bonnes Pratiques

1. **Créez des index intelligents** :
   - Index sur les colonnes fréquemment utilisées dans WHERE
   - Index couvrants (INCLUDE) pour les requêtes répétitives

2. **Maintenez les statistiques à jour** :
   ```sql
   ANALYZE nom_table;
   ```

3. **Exécutez VACUUM régulièrement** :
   ```sql
   VACUUM ANALYZE nom_table;
   ```
   Cela met à jour la Visibility Map et permet les Index-Only Scans.

4. **Analysez vos plans d'exécution** :
   ```sql
   EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
   ```

5. **Ne sur-indexez pas** : Chaque index a un coût en écriture et en espace disque

---

## Exemple Pratique de Transition entre Stratégies

Imaginons une table `commandes` avec 1 million de lignes.

### Scénario 1 : Recherche d'un client spécifique

```sql
-- 1 seule ligne retournée (0.0001%)
SELECT * FROM commandes WHERE client_id = 12345;
```

**Sans index** → **Seq Scan** (lit 1M lignes)
```
Seq Scan on commandes  (cost=0.00..18334.00 rows=1 width=20)
```

**Avec index** → **Index Scan** (lit 1 ligne)
```sql
CREATE INDEX idx_client ON commandes(client_id);
```
```
Index Scan using idx_client on commandes  (cost=0.42..8.44 rows=1 width=20)
```

### Scénario 2 : Recherche d'un range de clients

```sql
-- 50,000 lignes retournées (5%)
SELECT * FROM commandes WHERE client_id BETWEEN 10000 AND 15000;
```

**Avec index** → **Bitmap Scan** (compromis)
```
Bitmap Heap Scan on commandes  (cost=1040.50..12450.00 rows=50000 width=20)
  ->  Bitmap Index Scan on idx_client
```

### Scénario 3 : Agrégation simple

```sql
-- Seulement 2 colonnes nécessaires
SELECT client_id, COUNT(*)
FROM commandes
WHERE client_id = 12345
GROUP BY client_id;
```

**Avec index couvrant** → **Index-Only Scan**
```sql
CREATE INDEX idx_client_covering ON commandes(client_id) INCLUDE (id);
```
```
GroupAggregate
  ->  Index Only Scan using idx_client_covering on commandes
        Heap Fetches: 0
```

---

## Points Clés à Retenir

🔑 **Le planificateur choisit automatiquement** : PostgreSQL optimise pour vous, mais vous devez créer les bons index.

🔑 **Seq Scan n'est pas mauvais** : Pour grandes sélections ou petites tables, c'est souvent optimal.

🔑 **Index Scan pour sélections ciblées** : Idéal pour récupérer peu de lignes (<5%).

🔑 **Bitmap Scan combine plusieurs index** : Très utile pour des requêtes avec plusieurs conditions.

🔑 **Index-Only Scan = Performance maximale** : Nécessite des index couvrants et un VACUUM régulier.

🔑 **Analysez avec EXPLAIN** : C'est votre meilleur ami pour comprendre et optimiser vos requêtes.

🔑 **Maintenez vos statistiques** : `ANALYZE` et `VACUUM` sont essentiels pour de bonnes décisions du planificateur.

---

## Ressources pour Aller Plus Loin

- **Documentation officielle PostgreSQL** : [Query Planning](https://www.postgresql.org/docs/current/planner-stats.html)
- **EXPLAIN et ANALYZE** : Chapitre 13.7 de cette formation
- **Indexation avancée** : Chapitres 13.2 à 13.5 de cette formation

---


⏭️ [L'index B-Tree : Le couteau suisse (structure et algorithme)](/13-indexation-et-optimisation/02-index-btree.md)
