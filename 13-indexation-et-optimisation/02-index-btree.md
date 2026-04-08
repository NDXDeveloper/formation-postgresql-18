🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.2. L'index B-Tree : Le couteau suisse (structure et algorithme)

## Introduction

L'**index B-Tree** (Balanced Tree, ou arbre équilibré) est le type d'index **par défaut** dans PostgreSQL et le plus polyvalent. On le surnomme le "couteau suisse" car il répond efficacement à la majorité des besoins d'indexation.

Quand vous créez un index sans préciser de type, PostgreSQL utilise automatiquement un B-Tree :

```sql
-- Ces deux commandes sont équivalentes
CREATE INDEX idx_nom ON clients(nom);  
CREATE INDEX idx_nom ON clients USING BTREE(nom);  
```

### Pourquoi étudier les B-Tree ?

- **Omniprésence** : ~90% des index que vous créerez seront des B-Tree  
- **Performance** : Recherche, insertion, suppression en O(log n)  
- **Polyvalence** : Supporte égalité, comparaisons, tri, ranges  
- **Robustesse** : Structure équilibrée automatiquement

**Prérequis** : Avoir lu la section 13.1 sur les stratégies de scan.

---

## 1. Qu'est-ce qu'un Index ?

### Analogie : L'Index d'un Livre

Imaginez un dictionnaire de 1000 pages sans index alphabétique. Pour trouver le mot "PostgreSQL", vous devriez :
- Lire la page 1, puis la page 2, puis la page 3...
- Cela pourrait prendre plusieurs minutes (équivalent d'un **Seq Scan**)

Avec un index alphabétique :
- Vous ouvrez l'index à la lettre "P"
- Vous trouvez "PostgreSQL → page 742"
- Vous allez directement à la page 742 (équivalent d'un **Index Scan**)

**Un index dans PostgreSQL est exactement cela** : une structure de données auxiliaire qui permet de retrouver rapidement des lignes dans une table.

### Structure Générale

```
Table (Heap)                      Index B-Tree
┌─────────────────────┐          ┌─────────────────────┐
│ Ligne 1             │          │ Valeur → Pointeur   │
│ Ligne 2             │    ←─────│ "Alice" → Ligne 156 │
│ ...                 │          │ "Bob"   → Ligne 42  │
│ Ligne 1M            │          │ "Zoe"   → Ligne 891 │
└─────────────────────┘          └─────────────────────┘
```

L'index stocke :
- **Clés** : Les valeurs de la colonne indexée (triées)  
- **Pointeurs** : Les TID (Tuple ID) vers les lignes de la table

---

## 2. Structure d'un B-Tree : L'Arbre Équilibré

### Concept de Base

Un **B-Tree** (Balanced Tree) est une structure d'arbre où :
- Toutes les feuilles sont à la **même profondeur** (équilibré)
- Chaque nœud contient **plusieurs clés** (pas juste une comme un arbre binaire)
- Les clés sont **triées** dans chaque nœud

**Analogie** : Pensez à une bibliothèque organisée en sections → étagères → livres. Chaque niveau vous rapproche de votre livre cible.

### Anatomie d'un B-Tree PostgreSQL

```
                    [Nœud Racine (Root)]
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
    [Nœud Interne]   [Nœud Interne]   [Nœud Interne]
        │                 │                 │
    ┌───┼───┐         ┌───┼───┐         ┌───┼───┐
    │   │   │         │   │   │         │   │   │
[Feuille] [Feuille] [Feuille] [Feuille] [Feuille] [Feuille]
```

#### Les Composants

1. **Nœud Racine (Root Node)** : Point d'entrée de l'arbre  
2. **Nœuds Internes (Internal Nodes)** : Nœuds de navigation  
3. **Nœuds Feuilles (Leaf Nodes)** : Contiennent les données réelles (clés + pointeurs vers la table)

### Exemple Concret : Index sur les Noms

Imaginons une table `clients` avec un index sur la colonne `nom` :

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(50),
    email VARCHAR(100)
);

CREATE INDEX idx_clients_nom ON clients(nom);
```

**Structure du B-Tree** (simplifié) :

```
                    [Nœud Racine]
                        [M]
                    /           \
                   /             \
          [Nœud Interne]      [Nœud Interne]
              [D]                 [T]
          /         \           /         \
         /           \         /           \
    [Feuille]   [Feuille] [Feuille]  [Feuille]
    ┌──────┐    ┌──────┐  ┌──────┐    ┌──────┐
    │Alice │    │David │  │Marie │    │Zoe   │
    │→TID1 │    │→TID3 │  │→TID2 │    │→TID4 │
    │      │    │      │  │      │    │      │
    │Bob   │    │Emma  │  │Paul  │    │      │
    │→TID5 │    │→TID7 │  │→TID9 │    │      │
    └──────┘    └──────┘  └──────┘    └──────┘
```

**Lecture** :
- La racine contient "M" qui sépare les noms < M et ≥ M
- Le nœud interne gauche contient "D" qui sépare < D et ≥ D
- Les feuilles contiennent les clés réelles (noms) et les pointeurs (TID) vers les lignes de la table

---

## 3. Algorithme de Recherche dans un B-Tree

### Recherche d'une Valeur

**Objectif** : Trouver toutes les lignes où `nom = 'David'`

#### Étape par Étape

```sql
SELECT * FROM clients WHERE nom = 'David';
```

**1. Démarrage à la racine**

```
[Nœud Racine] : [M]
  ↓
  Comparaison : 'David' < 'M' ?
  ✓ Oui → Descendre à gauche
```

**2. Traversée des nœuds internes**

```
[Nœud Interne] : [D]
  ↓
  Comparaison : 'David' < 'D' ?
  ✗ Non → Descendre à droite
```

**3. Arrivée à la feuille**

```
[Feuille] : [David → TID3, Emma → TID7]
  ↓
  Recherche : 'David' trouvé !
  Résultat : TID3
```

**4. Accès à la table (Heap)**

```
PostgreSQL utilise TID3 pour accéder directement à la ligne dans la table
→ Récupération de (id, nom, email) de David
```

### Complexité

- **Hauteur de l'arbre** : O(log n)  
- **Recherche dans un nœud** : O(log k) où k = nombre de clés dans le nœud (recherche binaire)  
- **Complexité totale** : O(log n)

**Traduction** : Même avec 1 million de lignes, PostgreSQL ne fait que ~3-4 sauts pour trouver une valeur !

```
Nombre de lignes   | Hauteur B-Tree | Accès maximum
-------------------|----------------|---------------
1 000              | 2              | 2-3 lectures
100 000            | 3              | 3-4 lectures
10 000 000         | 4              | 4-5 lectures
1 000 000 000      | 5              | 5-6 lectures
```

**Comparaison avec Seq Scan** :
- Seq Scan sur 1M lignes : 1 000 000 lectures
- B-Tree sur 1M lignes : ~4 lectures

→ Gain de **250 000 fois** ! 🚀

---

## 4. Algorithme d'Insertion dans un B-Tree

### Insérer une Nouvelle Valeur

**Objectif** : Insérer `nom = 'Clara'` dans l'index

#### Étape 1 : Trouver la Feuille Appropriée

Même processus qu'une recherche :

```
[Racine] : [M]
  'Clara' < 'M' → Gauche

[Nœud Interne] : [D]
  'Clara' < 'D' → Gauche

[Feuille] : [Alice, Bob]
  ← Insertion de 'Clara' ici
```

#### Étape 2 : Insertion dans la Feuille

**Cas simple** : La feuille a de la place

```
Avant :
[Feuille] : [Alice → TID1, Bob → TID5]

Après :
[Feuille] : [Alice → TID1, Bob → TID5, Clara → TID8]
```

Les clés restent **triées** (Alice < Bob < Clara).

#### Étape 3 : Split en Cas de Débordement

**Problème** : Si la feuille est pleine (nombre maximum de clés atteint), PostgreSQL effectue un **split** (division).

**Avant split** :
```
[Feuille pleine] : [Alice, Bob, Clara, David, Emma]
```

**Après split** :
```
[Feuille 1] : [Alice, Bob, Clara]
[Feuille 2] : [David, Emma]

Le nœud parent reçoit la clé médiane 'David' pour séparer les deux feuilles
```

**Propagation** : Si le nœud parent est aussi plein, il est divisé à son tour. Ce processus peut remonter jusqu'à la racine.

**Rééquilibrage automatique** : Le B-Tree reste toujours **équilibré** grâce à ces mécanismes de split. Toutes les feuilles restent à la même profondeur.

---

## 5. Opérations Supportées par les B-Tree

### 5.1. Égalité (=)

```sql
SELECT * FROM clients WHERE nom = 'Alice';
```

✅ **B-Tree est optimal** : Recherche directe de la valeur.

### 5.2. Comparaisons (<, <=, >, >=)

```sql
SELECT * FROM commandes WHERE date_commande > '2024-01-01';
```

✅ **B-Tree fonctionne** : Les clés sont triées, PostgreSQL peut naviguer efficacement.

**Algorithme** :
1. Trouver la première clé ≥ '2024-01-01'  
2. Scanner séquentiellement les feuilles suivantes jusqu'à la fin

### 5.3. Ranges (BETWEEN)

```sql
SELECT * FROM employes WHERE salaire BETWEEN 30000 AND 50000;
```

✅ **B-Tree excelle** : Trouve le début du range, puis scan séquentiel jusqu'à la fin.

### 5.4. Tri (ORDER BY)

```sql
SELECT * FROM clients ORDER BY nom;
```

✅ **B-Tree est parfait** : Les clés sont déjà triées dans l'index !

PostgreSQL peut lire l'index dans l'ordre sans tri supplémentaire.

### 5.5. Pattern Matching (LIKE avec préfixe)

```sql
-- ✅ B-Tree utilisable
SELECT * FROM clients WHERE nom LIKE 'Dup%';

-- ❌ B-Tree NON utilisable
SELECT * FROM clients WHERE nom LIKE '%dup%';
```

**Explication** :
- Préfixe `'Dup%'` : PostgreSQL peut chercher toutes les valeurs commençant par "Dup" (triées ensemble)
- Joker au début `'%dup%'` : Impossible de naviguer dans l'arbre car les valeurs ne sont pas regroupées

### 5.6. Recherche NULL

```sql
SELECT * FROM clients WHERE email IS NULL;
```

✅ **B-Tree indexe les NULL** : Contrairement à certains SGBD, PostgreSQL stocke les NULL dans les B-Tree.

### 5.7. Min/Max Optimisé

```sql
SELECT MAX(date_creation) FROM articles;
```

✅ **B-Tree ultra-rapide** : PostgreSQL accède directement à la dernière feuille de l'index (O(log n) au lieu de O(n)).

---

## 6. Index Composites (Multi-Colonnes)

### Concept

Un **index composite** porte sur **plusieurs colonnes** :

```sql
CREATE INDEX idx_clients_nom_prenom ON clients(nom, prenom);
```

### Structure : Tri Hiérarchique

```
Index trié par : nom PUIS prenom

Feuilles :
[Dupont, Alice → TID1]
[Dupont, Marie → TID3]  ← Même nom, tri sur prenom
[Dupont, Paul  → TID5]
[Martin, Alice → TID2]
[Martin, Bob   → TID7]
```

**Principe** : Tri primaire sur `nom`, puis tri secondaire sur `prenom` en cas d'égalité.

### Règle de Leftmost Prefix

L'index `(nom, prenom)` peut être utilisé pour :

✅ **Utilisable** :
```sql
WHERE nom = 'Dupont'                     -- Colonne 1 seule  
WHERE nom = 'Dupont' AND prenom = 'Alice' -- Colonnes 1 + 2  
WHERE nom LIKE 'Dup%'                    -- Préfixe colonne 1  
```

❌ **NON utilisable** :
```sql
WHERE prenom = 'Alice'                   -- Saut de colonne 1  
WHERE nom = 'Dupont' OR prenom = 'Alice' -- OR entre colonnes  
```

**Analogie** : Un annuaire téléphonique trié par Nom puis Prénom. Vous pouvez chercher par Nom seul, mais pas par Prénom seul.

### Ordre des Colonnes : Critère Crucial

```sql
-- ❓ Quel ordre choisir ?
CREATE INDEX idx_a ON commandes(client_id, date_commande);  
CREATE INDEX idx_b ON commandes(date_commande, client_id);  
```

**Règle d'or** : Mettez en **premier** la colonne :
1. La plus **sélective** (qui filtre le plus de données)  
2. La plus **fréquemment utilisée seule** dans les WHERE

**Exemple** :

```sql
-- Requête fréquente
SELECT * FROM commandes WHERE client_id = 12345;

-- Requête rare
SELECT * FROM commandes WHERE date_commande = '2024-01-01';
```

→ Choisissez `(client_id, date_commande)` : client_id est plus sélectif et utilisé seul.

---

## 7. Avantages des B-Tree

### ✅ Polyvalence

Supporte presque toutes les opérations : égalité, comparaisons, ranges, tri, MIN/MAX.

### ✅ Performance Logarithmique

O(log n) pour recherche, insertion, suppression → Scalabilité exceptionnelle.

### ✅ Équilibrage Automatique

Vous n'avez rien à faire : PostgreSQL maintient l'arbre équilibré automatiquement.

### ✅ Support du Tri

Lecture de l'index dans l'ordre → Pas de tri supplémentaire pour ORDER BY.

### ✅ Range Queries Efficaces

Idéal pour `BETWEEN`, `>`, `<`, dates, etc.

### ✅ Stabilité

Structure mature, testée depuis des décennies, très fiable.

---

## 8. Limites des B-Tree

### ❌ Pas Optimal pour Full-Text Search

Pour rechercher du texte (`'%mot%'`), utilisez :
- **GIN** (Generalized Inverted Index) avec Full-Text Search
- Extension **pg_trgm** pour LIKE '%...%'

```sql
-- Inefficace avec B-Tree
SELECT * FROM articles WHERE contenu LIKE '%PostgreSQL%';

-- Efficace avec GIN + tsvector
CREATE INDEX idx_fts ON articles USING GIN(to_tsvector('french', contenu));  
SELECT * FROM articles WHERE to_tsvector('french', contenu) @@ to_tsquery('PostgreSQL');  
```

### ❌ Pas Optimal pour Données Géométriques

Pour des points, polygones, distances spatiales, utilisez :
- **GiST** (Generalized Search Tree)
- Extension **PostGIS**

```sql
CREATE INDEX idx_geo ON restaurants USING GIST(localisation);  
SELECT * FROM restaurants WHERE ST_DWithin(localisation, ST_Point(2.3522, 48.8566), 1000);  
```

### ❌ Coût de Maintenance

Chaque `INSERT`, `UPDATE`, `DELETE` doit maintenir l'index :
- Insertions peuvent provoquer des splits
- Updates peuvent nécessiter des réorganisations
- Deletions laissent des espaces vides (bloat)

**Impact** : Plus vous avez d'index, plus les écritures sont lentes.

### ❌ Espace Disque

Les B-Tree occupent de l'espace :

```sql
-- Vérifier la taille d'un index
SELECT pg_size_pretty(pg_relation_size('idx_clients_nom'));
-- Résultat : 42 MB
```

**Règle** : Un index B-Tree fait généralement **10-30% de la taille de la table** indexée.

### ❌ Index Bloat

Avec le temps, les index accumulent des **espaces vides** (bloat) :
- Suppressions laissent des trous
- Updates fragmentent les pages

**Solution** : `REINDEX` périodique

```sql
REINDEX INDEX idx_clients_nom;
-- Ou pour toute la table
REINDEX TABLE clients;
```

---

## 9. Quand Utiliser un B-Tree ?

### ✅ Cas d'Usage Idéaux

1. **Clés primaires et uniques**
   ```sql
   CREATE TABLE users (id SERIAL PRIMARY KEY);  -- B-Tree automatique
   ```

2. **Colonnes de jointure**
   ```sql
   CREATE INDEX idx_orders_user ON orders(user_id);
   ```

3. **Colonnes dans WHERE avec égalité ou comparaisons**
   ```sql
   WHERE date_creation > '2024-01-01'
   WHERE prix BETWEEN 10 AND 50
   WHERE statut = 'actif'
   ```

4. **Colonnes dans ORDER BY**
   ```sql
   ORDER BY nom, prenom
   ```

5. **Colonnes avec ranges de dates ou numériques**
   ```sql
   WHERE timestamp BETWEEN '2024-01-01' AND '2024-12-31'
   ```

### ❌ Quand NE PAS Utiliser un B-Tree

1. **Full-text search** → Utilisez GIN  
2. **Géométrie/Spatial** → Utilisez GiST (PostGIS)  
3. **Arrays complexes** → Utilisez GIN  
4. **JSONB avec recherches imbriquées** → Utilisez GIN  
5. **Données séquentielles massives (time-series)** → Utilisez BRIN  
6. **Colonnes avec très peu de valeurs distinctes** (faible cardinalité)
   ```sql
   -- Mauvais : colonne booléenne
   CREATE INDEX idx_actif ON users(actif);  -- 2 valeurs possibles seulement

   -- Mieux : index partiel
   CREATE INDEX idx_actifs ON users(id) WHERE actif = true;
   ```

---

## 10. Optimisations Avancées des B-Tree

### 10.1. Index Partiels (Partial Index)

Indexer seulement un **sous-ensemble** de la table :

```sql
-- Index uniquement les commandes non traitées
CREATE INDEX idx_commandes_pending ON commandes(date_creation)  
WHERE statut = 'pending';  
```

**Avantages** :
- Index plus petit → Moins d'espace, plus rapide
- Maintenance plus légère
- Cache plus efficace

**Cas d'usage** : Requêtes fréquentes sur un sous-ensemble spécifique.

### 10.2. Index sur Expressions

Indexer le **résultat d'une fonction** :

```sql
-- Recherche insensible à la casse
CREATE INDEX idx_email_lower ON users(LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

**Important** : La requête doit utiliser **exactement la même expression** que l'index.

### 10.3. Covering Index (INCLUDE)

Inclure des colonnes supplémentaires dans l'index pour permettre des **Index-Only Scans** :

```sql
CREATE INDEX idx_commandes_covering ON commandes(client_id) INCLUDE (montant, date_commande);

-- Index-Only Scan possible
SELECT client_id, montant, date_commande  
FROM commandes  
WHERE client_id = 12345;  
```

**Voir aussi** : Section 13.1 sur l'Index-Only Scan.

### 10.4. Nouveauté PostgreSQL 18 : Skip Scan Optimization

PostgreSQL 18 introduit une optimisation permettant d'utiliser efficacement des **index composites** même quand la première colonne n'est pas dans le WHERE.

**Avant PostgreSQL 18** :

```sql
CREATE INDEX idx_composite ON orders(status, created_at);

-- ❌ Index NON utilisé (colonne 1 absente)
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Avec PostgreSQL 18 (Skip Scan)** :

```sql
-- ✅ Index UTILISÉ malgré l'absence de 'status'
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

PostgreSQL "saute" (skip) les valeurs de `status` pour accéder directement à `created_at`.

**Avantage** : Moins d'index nécessaires, meilleure utilisation des index existants.

---

## 11. Maintenance des B-Tree

### 11.1. Surveillance de la Santé des Index

#### Vérifier la Taille

```sql
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes  
WHERE tablename = 'clients';  
```

#### Détecter le Bloat

Le **bloat** (gonflement) est l'espace inutilisé dans un index.

```sql
-- Extension pgstattuple (à installer)
CREATE EXTENSION pgstattuple;

SELECT
    indexname,
    round(100.0 * (1 - (avg_leaf_density / 100)), 1) AS bloat_pct
FROM pgstatindex('idx_clients_nom');
```

**Seuil d'alerte** : Bloat > 30% → Envisagez un REINDEX.

### 11.2. REINDEX : Reconstruire un Index

```sql
-- Reconstruire un index spécifique
REINDEX INDEX idx_clients_nom;

-- Reconstruire tous les index d'une table
REINDEX TABLE clients;

-- Reconstruire tous les index d'une base (attention : verrous)
REINDEX DATABASE mabase;
```

**⚠️ Attention** : `REINDEX` pose un **verrou exclusif** (bloque les écritures). Utilisez `REINDEX CONCURRENTLY` (PostgreSQL 12+) pour éviter le downtime :

```sql
REINDEX INDEX CONCURRENTLY idx_clients_nom;
```

### 11.3. VACUUM : Récupérer l'Espace

```sql
-- Nettoyer les espaces morts dans la table et ses index
VACUUM ANALYZE clients;

-- Vacuum plus agressif (réécrit complètement, verrous exclusifs)
VACUUM FULL clients;
```

**Différence** :
- `VACUUM` : Marque l'espace comme réutilisable (pas de verrou exclusif)  
- `VACUUM FULL` : Réécrit complètement (libère réellement l'espace, mais bloque la table)

### 11.4. Automatisation : Autovacuum

PostgreSQL exécute automatiquement `VACUUM` en arrière-plan via le processus **autovacuum**.

Configuration recommandée (dans `postgresql.conf`) :

```ini
autovacuum = on  
autovacuum_max_workers = 3  
autovacuum_naptime = 1min  
```

**Vérifier l'activité autovacuum** :

```sql
SELECT
    schemaname,
    relname,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables  
WHERE relname = 'clients';  
```

---

## 12. Comparaison B-Tree vs Autres Structures

| Critère | B-Tree | Hash | GIN | GiST | BRIN |
|---------|--------|------|-----|------|------|
| **Égalité (=)** | ✅ Excellent | ✅ Excellent | ⚠️ Selon données | ⚠️ Selon données | ❌ Médiocre |
| **Comparaisons (<, >)** | ✅ Excellent | ❌ Non supporté | ❌ Non supporté | ⚠️ Selon données | ✅ Bon |
| **Ranges (BETWEEN)** | ✅ Excellent | ❌ Non supporté | ❌ Non supporté | ⚠️ Selon données | ✅ Bon |
| **Tri (ORDER BY)** | ✅ Optimal | ❌ Non supporté | ❌ Non supporté | ❌ Non supporté | ❌ Non supporté |
| **LIKE 'prefix%'** | ✅ Bon | ❌ Non supporté | ✅ Avec pg_trgm | ⚠️ Selon config | ❌ Non supporté |
| **LIKE '%suffix'** | ❌ Non supporté | ❌ Non supporté | ✅ Avec pg_trgm | ⚠️ Selon config | ❌ Non supporté |
| **Full-Text Search** | ❌ Inefficace | ❌ Non supporté | ✅ Excellent | ⚠️ Possible | ❌ Non supporté |
| **Arrays/JSONB** | ⚠️ Limité | ❌ Non supporté | ✅ Excellent | ✅ Bon | ❌ Non supporté |
| **Géométrie** | ❌ Non supporté | ❌ Non supporté | ❌ Non supporté | ✅ Excellent | ❌ Non supporté |
| **Taille** | Moyenne | Petite | Grande | Grande | Très petite |
| **Maintenance** | Modérée | Faible | Élevée | Élevée | Très faible |

**Conclusion** : B-Tree est le **meilleur choix par défaut** pour la majorité des cas d'usage.

---

## 13. Exemple Complet : De la Création à l'Utilisation

### Scénario : Application E-Commerce

```sql
-- Table de commandes
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,                    -- B-Tree automatique (PK)
    client_id INT NOT NULL,
    montant NUMERIC(10,2),
    statut VARCHAR(20) DEFAULT 'pending',
    date_creation TIMESTAMP DEFAULT NOW()
);

-- Index 1 : Recherche par client (fréquent)
CREATE INDEX idx_commandes_client ON commandes(client_id);

-- Index 2 : Recherche par date (pour rapports)
CREATE INDEX idx_commandes_date ON commandes(date_creation DESC);

-- Index 3 : Composite pour requêtes mixtes
CREATE INDEX idx_commandes_client_date ON commandes(client_id, date_creation DESC);

-- Index 4 : Partiel sur commandes en attente (optimisation)
CREATE INDEX idx_commandes_pending ON commandes(client_id, date_creation)  
WHERE statut = 'pending';  

-- Index 5 : Covering index pour dashboard
CREATE INDEX idx_commandes_covering ON commandes(client_id)  
INCLUDE (montant, date_creation, statut);  
```

### Requêtes et Plans d'Exécution

**Requête 1** : Commandes d'un client

```sql
EXPLAIN SELECT * FROM commandes WHERE client_id = 12345;
```

**Plan** :
```
Index Scan using idx_commandes_client on commandes
  Index Cond: (client_id = 12345)
```

**Requête 2** : Commandes d'un client par date

```sql
EXPLAIN SELECT * FROM commandes  
WHERE client_id = 12345  
ORDER BY date_creation DESC;  
```

**Plan** :
```
Index Scan using idx_commandes_client_date on commandes
  Index Cond: (client_id = 12345)
```

Index composite utilisé pour filtrage ET tri !

**Requête 3** : Dashboard (Index-Only Scan)

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT client_id, montant, date_creation  
FROM commandes  
WHERE client_id = 12345;  
```

**Plan** :
```
Index Only Scan using idx_commandes_covering on commandes
  Index Cond: (client_id = 12345)
  Heap Fetches: 0
```

Toutes les données viennent de l'index, pas d'accès à la table !

---

## Points Clés à Retenir

🔑 **B-Tree = Type d'index par défaut** : Utilisé dans 90% des cas.

🔑 **Structure équilibrée** : Toutes les feuilles à la même profondeur → Performances prévisibles O(log n).

🔑 **Polyvalent** : Égalité, comparaisons, ranges, tri, MIN/MAX.

🔑 **Index composites** : Ordre des colonnes crucial (leftmost prefix rule).

🔑 **Maintenance nécessaire** : REINDEX et VACUUM réguliers pour éviter le bloat.

🔑 **Optimisations avancées** : Index partiels, sur expressions, INCLUDE pour covering.

🔑 **PostgreSQL 18** : Skip Scan optimization pour meilleure utilisation des index composites.

🔑 **Limitations** : Pas optimal pour full-text search, géométrie, ou arrays complexes.

---

## Exercice de Réflexion (Sans Pratique)

Analysez les requêtes suivantes et déterminez si un B-Tree serait approprié :

1. `SELECT * FROM articles WHERE titre LIKE '%PostgreSQL%'`
   **Réponse** : ❌ B-Tree inefficace (joker au début). Utilisez GIN avec pg_trgm.

2. `SELECT * FROM users WHERE date_inscription > '2024-01-01' ORDER BY date_inscription`
   **Réponse** : ✅ B-Tree parfait (range + tri).

3. `SELECT * FROM products WHERE tags @> ARRAY['electronics', 'sale']`
   **Réponse** : ❌ B-Tree limité pour arrays. Utilisez GIN.

4. `SELECT MAX(prix) FROM products WHERE categorie = 'laptops'`
   **Réponse** : ✅ B-Tree excellent (index composite sur `(categorie, prix)` serait optimal).

---

## Ressources pour Aller Plus Loin

- **Documentation PostgreSQL** : [B-Tree Index Type](https://www.postgresql.org/docs/current/btree.html)  
- **Section précédente** : 13.1. Stratégies de scan  
- **Section suivante** : 13.3. Nouveauté PG 18 : Skip Scan optimization  
- **Indexation avancée** : Chapitres 13.4 (GIN, GiST, BRIN, Hash)

---


⏭️ [Nouveauté PG 18 : Skip Scan optimization pour index multi-colonnes](/13-indexation-et-optimisation/03-skip-scan-optimization.md)
