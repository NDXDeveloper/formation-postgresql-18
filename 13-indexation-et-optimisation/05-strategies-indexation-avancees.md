🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.5. Stratégies d'Indexation Avancées

## Introduction

Dans les chapitres précédents, nous avons découvert les différents **types d'index** (B-Tree, GIN, GiST, BRIN, Hash) et leur utilisation de base. Maintenant, nous allons explorer des **stratégies d'indexation avancées** qui permettent d'optimiser encore davantage les performances de vos requêtes PostgreSQL.

Ces stratégies ne concernent pas tant le *type* d'index à utiliser, mais plutôt **comment créer intelligemment des index** pour répondre à des besoins spécifiques et complexes.

---

## Pourquoi des Stratégies Avancées ?

### Le Problème : Les Index Simples Ne Suffisent Pas Toujours

Imaginez ces scénarios courants :

#### Scénario 1 : Table avec Soft Delete
```sql
-- Table d'utilisateurs avec suppression logique
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    nom VARCHAR(100),
    est_supprime BOOLEAN DEFAULT FALSE,  -- 95% FALSE, 5% TRUE
    date_creation TIMESTAMP
);

-- Index classique sur email
CREATE INDEX idx_email ON utilisateurs(email);
-- Problème : Indexe 100% des lignes, alors qu'on ne cherche que dans les 95% actifs !
```

**Question :** Comment créer un index qui n'indexe que les utilisateurs actifs ?

#### Scénario 2 : Recherches Insensibles à la Casse
```sql
-- Table de produits
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),  -- 'iPhone 15 Pro', 'IPHONE 15 Pro', 'iphone 15 pro'
    prix NUMERIC(10,2)
);

-- Index classique
CREATE INDEX idx_nom ON produits(nom);

-- Requête : Recherche insensible à la casse
SELECT * FROM produits WHERE LOWER(nom) = 'iphone 15 pro';
-- Problème : L'index n'est PAS utilisé (la fonction LOWER() empêche son utilisation) !
```

**Question :** Comment créer un index qui fonctionne avec `LOWER()` ?

#### Scénario 3 : Requête avec SELECT de Nombreuses Colonnes
```sql
-- Requête fréquente
SELECT id, nom, email, telephone, ville
FROM clients
WHERE pays = 'France';

-- Index sur pays
CREATE INDEX idx_pays ON clients(pays);

-- Résultat EXPLAIN :
-- Index Scan using idx_pays on clients
--   Index Cond: (pays = 'France')
--   → Heap Fetches: 15000  ← PostgreSQL doit accéder à la table pour les colonnes !
```

**Question :** Comment éviter ces accès coûteux à la table ?

#### Scénario 4 : Données JSON Semi-Structurées
```sql
-- Table avec métadonnées JSON
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER,
    metadata JSONB  -- {"statut": "livree", "montant": 150.50, ...}
);

-- Requête : Chercher par champ JSON
SELECT * FROM commandes WHERE metadata->>'statut' = 'en_cours';
-- Problème : Scan séquentiel (très lent sur grandes tables) !
```

**Question :** Comment indexer efficacement des données JSON ?

---

## Les Quatre Stratégies Avancées

Ce chapitre couvre quatre stratégies puissantes qui répondent à ces problématiques :

### 13.5.1. Index Partiels (WHERE clause)

**Principe :** Créer un index qui n'indexe qu'un **sous-ensemble** des lignes de la table.

**Syntaxe :**
```sql
CREATE INDEX nom_index ON table(colonne) WHERE condition;
```

**Exemple :**
```sql
-- Indexer uniquement les utilisateurs actifs
CREATE INDEX idx_email_actifs ON utilisateurs(email)
WHERE est_supprime = FALSE;
```

**Avantages :**
- ✅ Index **beaucoup plus petit** (moins d'espace disque)
- ✅ **Plus rapide** à parcourir
- ✅ **Maintenance réduite** (INSERT/UPDATE moins coûteux)

**Cas d'usage :**
- Soft delete (lignes actives vs supprimées)
- États temporaires (commandes en cours)
- Données exceptionnelles (erreurs, priorités hautes)
- Filtrage sur valeurs spécifiques fréquentes

**Couvert dans :** [13.5.1. Index Partiels](./13-5-1-index-partiels.md)

---

### 13.5.2. Index sur Expressions

**Principe :** Créer un index sur le **résultat d'une fonction ou expression**, pas sur la valeur brute.

**Syntaxe :**
```sql
CREATE INDEX nom_index ON table((expression));
-- Note : doubles parenthèses obligatoires !
```

**Exemple :**
```sql
-- Index sur email en minuscules
CREATE INDEX idx_email_lower ON utilisateurs((LOWER(email)));

-- Requête optimisée
SELECT * FROM utilisateurs WHERE LOWER(email) = 'john@example.com';
-- Utilise l'index !
```

**Avantages :**
- ✅ Recherches **insensibles à la casse**
- ✅ Indexation de **calculs** (prix TTC, surface, âge)
- ✅ Extraction de **composants** (année d'une date, champ JSON)

**Cas d'usage :**
- `LOWER()` / `UPPER()` pour recherches texte
- `DATE()` pour extraire date d'un timestamp
- `data->>'field'` pour extraire champ JSON
- Calculs métier (montant * TVA, longueur * largeur)

**Couvert dans :** [13.5.2. Index sur Expressions](./13-5-2-index-sur-expressions.md)

---

### 13.5.3. Index Multi-Colonnes et INCLUDE (Covering Index)

**Principe 1 : Index Multi-Colonnes** - Indexer **plusieurs colonnes simultanément**.

**Syntaxe :**
```sql
CREATE INDEX nom_index ON table(col1, col2, col3);
```

**Exemple :**
```sql
-- Index composite pour requêtes multi-critères
CREATE INDEX idx_client_date ON commandes(client_id, date_commande DESC);

-- Requête optimisée
SELECT * FROM commandes
WHERE client_id = 12345
ORDER BY date_commande DESC;
-- Utilise l'index pour filtrage ET tri !
```

**Principe 2 : Clause INCLUDE (Covering Index)** - Ajouter des colonnes dans l'index **sans les utiliser pour l'indexation**.

**Syntaxe :**
```sql
CREATE INDEX nom_index ON table(colonnes_indexées)
INCLUDE (colonnes_couvertes);
```

**Exemple :**
```sql
-- Covering index
CREATE INDEX idx_client_covering ON commandes(client_id)
INCLUDE (date_commande, montant, statut);

-- Requête : Tout dans l'index, pas d'accès à la table !
SELECT client_id, date_commande, montant, statut
FROM commandes
WHERE client_id = 12345;
-- Index Only Scan (pas de Heap Fetches) !
```

**Avantages :**
- ✅ **Index Multi-Colonnes :** Optimise requêtes avec plusieurs filtres/tris
- ✅ **INCLUDE :** Élimine les accès à la table (Heap Fetches)
- ✅ **Performances :** Réduction des I/O de 3-10×

**Cas d'usage :**
- Requêtes avec filtrage sur plusieurs colonnes
- SELECT avec colonnes supplémentaires
- Élimination des Heap Fetches

**Couvert dans :** [13.5.3. Index Multi-Colonnes et INCLUDE](./13-5-3-index-multi-colonnes-include.md)

---

### 13.5.4. Index sur Colonnes JSONB (GIN avec jsonb_path_ops)

**Principe :** Utiliser des **index GIN** (Generalized Inverted Index) pour indexer efficacement des données JSON/JSONB.

**Syntaxe :**
```sql
-- GIN standard (tous opérateurs)
CREATE INDEX nom_index ON table USING gin(colonne_jsonb);

-- GIN optimisé (opérateur @> uniquement, 3× plus compact)
CREATE INDEX nom_index ON table USING gin(colonne_jsonb jsonb_path_ops);
```

**Exemple :**
```sql
-- Table avec métadonnées JSON
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    metadata JSONB
);

-- Index GIN optimisé
CREATE INDEX idx_metadata ON produits USING gin(metadata jsonb_path_ops);

-- Requête : Recherche dans JSON
SELECT * FROM produits
WHERE metadata @> '{"marque": "Dell", "en_stock": true}';
-- Utilise l'index GIN !
```

**Avantages :**
- ✅ Recherche **ultra-rapide** dans JSON (50-200× plus rapide)
- ✅ Flexibilité des **données semi-structurées**
- ✅ `jsonb_path_ops` : **3× plus compact** que GIN standard

**Cas d'usage :**
- Métadonnées produits variables
- Logs avec événements JSON
- Attributs dynamiques (CRM, ERP)
- Configuration applicative

**Couvert dans :** [13.5.4. Index sur Colonnes JSONB](./13-5-4-index-jsonb-gin.md)

---

## Comparaison des Stratégies

| Stratégie | Problème Résolu | Gain Typique | Complexité |
|-----------|-----------------|--------------|------------|
| **Index Partiel** | Index sur sous-ensemble | 10-20× plus compact | 🟢 Simple |
| **Index sur Expression** | Fonction dans WHERE | 3-100× plus rapide | 🟡 Moyen |
| **Multi-Colonnes + INCLUDE** | Heap Fetches | 3-10× moins d'I/O | 🟡 Moyen |
| **GIN sur JSONB** | Données semi-structurées | 50-200× plus rapide | 🔴 Complexe |

---

## Quand Utiliser Chaque Stratégie ?

### Arbre de Décision

```
Votre requête filtre sur un sous-ensemble prévisible ?
├─ OUI → Index Partiel (13.5.1)
│   Exemple : WHERE est_actif = TRUE
│
Votre requête utilise une fonction dans WHERE ?
├─ OUI → Index sur Expression (13.5.2)
│   Exemple : WHERE LOWER(email) = '...'
│
Votre requête filtre sur plusieurs colonnes OU a beaucoup de Heap Fetches ?
├─ OUI → Multi-Colonnes + INCLUDE (13.5.3)
│   Exemple : WHERE col1 = X AND col2 = Y, SELECT col1, col2, col3, col4
│
Vos données sont en JSON/JSONB ?
├─ OUI → Index GIN (13.5.4)
│   Exemple : WHERE metadata @> '{"key": "value"}'
│
└─ Index B-Tree classique suffit
```

---

## Combiner les Stratégies

La puissance de ces techniques réside dans leur **combinaison** !

### Exemple 1 : Partiel + Expression

```sql
-- Index sur email en minuscules, uniquement pour utilisateurs actifs
CREATE INDEX idx_email_lower_actifs
ON utilisateurs((LOWER(email)))
WHERE est_supprime = FALSE;
```

**Avantages :**
- Expression : Recherche insensible à la casse
- Partiel : Index 20× plus petit (uniquement actifs)

### Exemple 2 : Multi-Colonnes + INCLUDE + Partiel

```sql
-- L'index ultime pour requêtes sur commandes récentes
CREATE INDEX idx_commandes_ultimate
ON commandes(client_id, date_commande DESC)
INCLUDE (montant, statut)
WHERE date_commande >= '2025-01-01';
```

**Avantages :**
- Multi-colonnes : Filtre + tri optimisés
- INCLUDE : Pas de Heap Fetches
- Partiel : Index compact (seulement 2025)

### Exemple 3 : GIN + Partiel

```sql
-- Index GIN sur metadata JSON, uniquement produits disponibles
CREATE INDEX idx_metadata_disponibles
ON produits USING gin(metadata jsonb_path_ops)
WHERE (metadata->>'disponible')::boolean = TRUE;
```

**Avantages :**
- GIN : Recherche rapide dans JSON
- Partiel : Index plus petit (seulement produits en stock)

---

## Méthodologie : Comment Choisir ?

### Étape 1 : Identifier les Requêtes Lentes

```sql
-- Utiliser pg_stat_statements
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Étape 2 : Analyser avec EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ... FROM ... WHERE ...;
```

**Cherchez :**
- `Seq Scan` → Besoin d'un index
- `Heap Fetches: 1000+` → Candidat pour INCLUDE
- `Filter: (lower(...)` → Index sur expression
- Fonction sur colonne → Index sur expression
- `Rows Removed by Filter: 99%` → Index partiel

### Étape 3 : Choisir la Stratégie Appropriée

| Observation dans EXPLAIN | Stratégie Recommandée |
|---------------------------|------------------------|
| Seq Scan + fonction dans Filter | Index sur expression |
| Seq Scan + condition simple récurrente | Index partiel |
| Index Scan + Heap Fetches élevés | Multi-colonnes + INCLUDE |
| Seq Scan sur colonne JSONB | Index GIN |
| Multiple colonnes dans WHERE | Index multi-colonnes |

### Étape 4 : Tester et Mesurer

```sql
-- Avant
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- Note : Execution Time: 2500 ms, Buffers: shared hit=35000

-- Créer l'index
CREATE INDEX ...;

-- Après
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- Note : Execution Time: 15 ms, Buffers: shared hit=25

-- Gain : 166× plus rapide, 1400× moins d'I/O !
```

### Étape 5 : Monitorer l'Utilisation

```sql
-- Vérifier que l'index est utilisé
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS utilisations,
    idx_tup_read,
    pg_size_pretty(pg_relation_size(indexrelid)) AS taille
FROM pg_stat_user_indexes
WHERE indexname = 'nom_de_votre_index';
```

**Si `idx_scan = 0` après quelques jours :** L'index n'est pas utilisé → À supprimer.

---

## Coûts et Considérations

### Coût de Création

Créer un index bloque les écritures (sauf avec `CONCURRENTLY`) :

```sql
-- Bloque les écritures (rapide mais bloquant)
CREATE INDEX idx ON table(col);

-- Ne bloque pas (plus lent mais production-safe)
CREATE INDEX CONCURRENTLY idx ON table(col);
```

**Temps de création :** Dépend de la taille de la table
- 1M lignes : 30 secondes à 2 minutes
- 10M lignes : 5 à 20 minutes
- 100M lignes : 30 minutes à 2 heures

### Coût de Maintenance

Chaque index a un coût lors des opérations d'écriture :

```sql
-- Sans index : INSERT rapide
INSERT INTO table VALUES (...);  -- 0.5 ms

-- Avec 5 index : INSERT plus lent
INSERT INTO table VALUES (...);  -- 2.5 ms (5× plus lent)
```

**Règle générale :**
- Tables OLTP (beaucoup d'écritures) : Limiter à 3-5 index
- Tables OLAP (lecture principalement) : 10+ index acceptables

### Coût d'Espace Disque

Les index occupent de l'espace :

```sql
-- Vérifier la taille
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS "Taille totale",
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS "Taille table",
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename)) AS "Taille index"
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

**Résultat typique :**
```
Table: commandes
├─ Taille table : 2.5 GB
└─ Taille index : 1.8 GB  ← Les index = 72% de la taille de la table !
```

---

## Pièges Communs à Éviter

### ❌ Piège 1 : Créer Trop d'Index

```sql
-- ❌ MAUVAIS : 10 index sur une table OLTP
CREATE INDEX idx1 ON table(col1);
CREATE INDEX idx2 ON table(col2);
CREATE INDEX idx3 ON table(col3);
-- ... 10 index au total
-- Problème : INSERT/UPDATE très lents !
```

**Solution :** Prioriser les requêtes les plus fréquentes et critiques.

### ❌ Piège 2 : Dupliquer les Index

```sql
-- ❌ MAUVAIS : Index redondants
CREATE INDEX idx1 ON table(a, b, c);
CREATE INDEX idx2 ON table(a, b);     -- Redondant avec idx1 !
CREATE INDEX idx3 ON table(a);        -- Redondant avec idx1 !
```

**Solution :** Un index sur (a, b, c) couvre automatiquement (a, b) et (a).

### ❌ Piège 3 : Index Inutilisés

```sql
-- Index créé mais jamais utilisé
CREATE INDEX idx_ancien ON table(colonne_jamais_requetee);

-- Vérifier après 1 mois
SELECT * FROM pg_stat_user_indexes WHERE indexname = 'idx_ancien';
-- idx_scan = 0  ← Jamais utilisé !
```

**Solution :** Monitorer et supprimer les index inutilisés.

### ❌ Piège 4 : Oublier CONCURRENTLY en Production

```sql
-- ❌ MAUVAIS en production : Bloque la table
CREATE INDEX idx ON table(col);

-- ✅ BON en production : Non bloquant
CREATE INDEX CONCURRENTLY idx ON table(col);
```

### ❌ Piège 5 : Mauvais Ordre dans Index Multi-Colonnes

```sql
-- Table : 1M de commandes
-- Requête : WHERE client_id = X AND statut = 'livree'

-- ❌ MAUVAIS ordre
CREATE INDEX idx_bad ON commandes(statut, client_id);
-- statut: 5 valeurs (peu sélectif)
-- client_id: 50K valeurs (très sélectif)

-- ✅ BON ordre
CREATE INDEX idx_good ON commandes(client_id, statut);
-- client_id en premier = filtrage immédiat et efficace
```

---

## Bonnes Pratiques Générales

### ✅ DO (À Faire)

1. **Analyser avant de créer**
   ```sql
   EXPLAIN (ANALYZE, BUFFERS) -- Toujours vérifier l'impact
   ```

2. **Utiliser CONCURRENTLY en production**
   ```sql
   CREATE INDEX CONCURRENTLY ...
   ```

3. **Documenter les index complexes**
   ```sql
   COMMENT ON INDEX idx IS 'Optimise requête X, réduit temps de 2500ms à 15ms';
   ```

4. **Monitorer l'utilisation**
   ```sql
   -- Requête mensuelle pour identifier index inutilisés
   SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;
   ```

5. **Tester sur données réelles**
   - Ne pas tester uniquement sur table vide
   - Volume de données représentatif

6. **Combiner les stratégies quand pertinent**
   ```sql
   CREATE INDEX idx ON table(col1, col2)
   INCLUDE (col3)
   WHERE condition;
   ```

### ❌ DON'T (À Éviter)

1. **Ne pas créer d'index "au cas où"**
   - Chaque index a un coût
   - Créer uniquement pour requêtes identifiées

2. **Ne pas oublier de supprimer les anciens index**
   ```sql
   DROP INDEX CONCURRENTLY idx_ancien;
   ```

3. **Ne pas créer d'index sur petites tables**
   - < 1000 lignes : Seq Scan souvent plus rapide

4. **Ne pas indexer les colonnes rarement utilisées**

5. **Ne pas créer trop d'index similaires**
   - Analyser les recouvrements possibles

---

## Outils et Ressources

### Extensions PostgreSQL Utiles

```sql
-- pg_stat_statements : Analyser les requêtes
CREATE EXTENSION pg_stat_statements;

-- HypoPG : Tester des index hypothétiques sans les créer
CREATE EXTENSION hypopg;

-- pg_qualstats : Statistiques sur les prédicats WHERE
CREATE EXTENSION pg_qualstats;
```

### Outils Externes

- **pgBadger** : Analyse de logs PostgreSQL
- **pg_hero** : Suggestions d'index et détection de problèmes
- **pgtune** : Aide à la configuration PostgreSQL
- **explain.depesz.com** : Visualiser et analyser EXPLAIN

### Commandes Utiles

```sql
-- Lister tous les index d'une table
\d+ nom_table

-- Taille des index
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE tablename = 'nom_table';

-- Index inutilisés
SELECT
    schemaname || '.' || tablename AS table,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS taille
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Plan du Chapitre 13.5

Ce chapitre est organisé en quatre sections détaillées :

### 📖 13.5.1. Index Partiels (WHERE clause)
- Concept et syntaxe
- Cas d'usage (soft delete, états temporaires)
- Combinaisons avec autres stratégies
- Pièges et bonnes pratiques

### 📖 13.5.2. Index sur Expressions
- Fonctions dans WHERE
- Recherches insensibles à la casse
- Extraction de composants (dates, JSON)
- IMMUTABLE vs VOLATILE
- Alternatives (colonnes générées)

### 📖 13.5.3. Index Multi-Colonnes et INCLUDE
- Index composites
- Ordre des colonnes (règles d'or)
- Clause INCLUDE (covering index)
- Index Only Scan
- Élimination des Heap Fetches

### 📖 13.5.4. Index sur Colonnes JSONB
- JSON vs JSONB
- Index GIN (jsonb_ops vs jsonb_path_ops)
- Opérateurs JSONB (@>, ?, ?|, ?&)
- Performance et benchmarks
- Maintenance des index GIN

---

## Résumé de l'Introduction

### Points Clés à Retenir

1. **Les index simples ne suffisent pas toujours**
   - Besoins complexes nécessitent stratégies avancées

2. **Quatre stratégies principales :**
   - Index partiels (WHERE)
   - Index sur expressions (fonctions)
   - Multi-colonnes + INCLUDE (covering)
   - GIN sur JSONB (données semi-structurées)

3. **Ces stratégies sont combinables**
   - Exemple : Partiel + Expression + INCLUDE

4. **Méthodologie :**
   - Identifier (pg_stat_statements)
   - Analyser (EXPLAIN)
   - Optimiser (créer index adapté)
   - Mesurer (comparer avant/après)
   - Monitorer (pg_stat_user_indexes)

5. **Considérations :**
   - Coût de création (temps)
   - Coût de maintenance (écritures)
   - Coût d'espace (disque)

6. **Bonnes pratiques :**
   - Analyser avant de créer
   - CONCURRENTLY en production
   - Documenter les index complexes
   - Supprimer les index inutilisés

---

## Pour Commencer

Vous êtes maintenant prêt à explorer chaque stratégie en détail. Nous recommandons de les étudier dans l'ordre suivant :

1. **Débutez par les Index Partiels** (13.5.1)
   - Concept le plus simple
   - Gains immédiats importants
   - Peu de risques

2. **Continuez avec les Index sur Expressions** (13.5.2)
   - Résout des problèmes courants (LOWER, dates)
   - Nécessite compréhension des fonctions IMMUTABLE

3. **Approfondissez avec Multi-Colonnes et INCLUDE** (13.5.3)
   - Plus complexe (ordre des colonnes)
   - Gains de performance spectaculaires

4. **Terminez par les Index JSONB** (13.5.4)
   - Cas d'usage spécifique (données JSON)
   - Concept d'index inversé (GIN)

---

## Prochaine Étape

👉 **Commencez par :** 13.5.1. Index Partiels (WHERE clause)

Vous y découvrirez comment créer des index qui n'indexent qu'un sous-ensemble de vos données, réduisant ainsi leur taille de 10-20× et améliorant significativement les performances !

---


⏭️ [Index partiels (WHERE clause)](/13-indexation-et-optimisation/05.1-index-partiels.md)
