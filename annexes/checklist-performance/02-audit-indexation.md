🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Audit d'Indexation PostgreSQL
## Guide

---

## Table des Matières

1. [Introduction aux Index](#1-introduction-aux-index)  
2. [Pourquoi Auditer les Index ?](#2-pourquoi-auditer-les-index-)  
3. [Anatomie d'un Index](#3-anatomie-dun-index)  
4. [Types d'Index dans PostgreSQL](#4-types-dindex-dans-postgresql)  
5. [Méthodologie d'Audit d'Indexation](#5-m%C3%A9thodologie-daudit-dindexation)  
6. [Détection des Index Manquants](#6-d%C3%A9tection-des-index-manquants)  
7. [Détection des Index Inutilisés](#7-d%C3%A9tection-des-index-inutilis%C3%A9s)  
8. [Détection des Index Redondants](#8-d%C3%A9tection-des-index-redondants)  
9. [Bloat et Fragmentation des Index](#9-bloat-et-fragmentation-des-index)  
10. [Performance des Index](#10-performance-des-index)  
11. [Stratégies d'Indexation Avancées](#11-strat%C3%A9gies-dindexation-avanc%C3%A9es)  
12. [PostgreSQL 18 : Nouveautés d'Indexation](#12-postgresql-18--nouveaut%C3%A9s-dindexation)  
13. [Outils d'Audit d'Indexation](#13-outils-daudit-dindexation)  
14. [Checklist d'Audit Complète](#14-checklist-daudit-compl%C3%A8te)  
15. [Erreurs Courantes d'Indexation](#15-erreurs-courantes-dindexation)  
16. [Cas Pratiques et Scénarios](#16-cas-pratiques-et-sc%C3%A9narios)  
17. [Conclusion et Bonnes Pratiques](#17-conclusion-et-bonnes-pratiques)

---

## 1. Introduction aux Index

### 1.1. Qu'est-ce qu'un Index ?

Un **index** est une structure de données auxiliaire qui permet d'accélérer la recherche de lignes dans une table.

**Analogie** : Un index dans une base de données est comme l'index d'un livre :
- Sans index, vous devez parcourir toutes les pages pour trouver un sujet (sequential scan)
- Avec un index, vous consultez la page de référence qui vous indique directement où aller (index scan)

### 1.2. Comment Fonctionne un Index ?

**Sans index** :
```sql
SELECT * FROM employes WHERE nom = 'Dupont';
```
PostgreSQL doit lire **toute la table** ligne par ligne jusqu'à trouver 'Dupont'.

**Avec un index sur `nom`** :
```sql
CREATE INDEX idx_employes_nom ON employes(nom);  
SELECT * FROM employes WHERE nom = 'Dupont';  
```
PostgreSQL consulte l'index qui pointe directement vers les lignes contenant 'Dupont'.

### 1.3. Le Trade-off Fondamental des Index

Les index offrent un compromis (trade-off) :

**Avantages** :
- ✅ Lectures beaucoup plus rapides (SELECT)  
- ✅ Tri rapide (ORDER BY)  
- ✅ Jointures accélérées  
- ✅ Contraintes d'unicité (UNIQUE)

**Inconvénients** :
- ❌ Écritures plus lentes (INSERT, UPDATE, DELETE)  
- ❌ Consommation d'espace disque  
- ❌ Coût de maintenance (VACUUM, REINDEX)  
- ❌ Surcharge mémoire

**Règle d'or** : Un bon index accélère les lectures sans trop pénaliser les écritures.

### 1.4. Quand Créer un Index ?

Créez un index quand :
- ✅ La colonne est fréquemment utilisée dans les clauses WHERE  
- ✅ La colonne est utilisée pour des jointures (JOIN)  
- ✅ La colonne est utilisée pour des tris (ORDER BY)  
- ✅ La colonne est utilisée dans des GROUP BY  
- ✅ La table est grande (> 10,000 lignes)  
- ✅ La sélectivité est élevée (peu de doublons)

Ne créez **pas** d'index quand :
- ❌ La table est petite (< 1,000 lignes)  
- ❌ La colonne a peu de valeurs distinctes (ex: booléen)  
- ❌ La table est principalement en écriture (INSERT/UPDATE massif)  
- ❌ La colonne est rarement utilisée dans les requêtes

---

## 2. Pourquoi Auditer les Index ?

### 2.1. Les Problèmes d'une Mauvaise Indexation

Une stratégie d'indexation inadaptée peut causer :

#### Problème 1 : Manque d'Index
**Symptôme** :
- Requêtes très lentes
- Sequential scans sur grandes tables
- Utilisation CPU élevée

**Impact** :
- Temps de réponse de plusieurs secondes
- Saturation du serveur
- Mauvaise expérience utilisateur

#### Problème 2 : Trop d'Index
**Symptôme** :
- Écritures lentes (INSERT, UPDATE, DELETE)
- Espace disque consommé
- Maintenance lourde

**Impact** :
- Débit d'écriture divisé par 2 ou plus
- Coûts de stockage élevés
- Autovacuum surchargé

#### Problème 3 : Index Inutilisés
**Symptôme** :
- Index créés mais jamais utilisés par le planificateur
- Gaspillage de ressources

**Impact** :
- Écritures ralenties inutilement
- Espace disque perdu
- Maintenance inutile

#### Problème 4 : Index Redondants
**Symptôme** :
- Plusieurs index couvrent les mêmes colonnes
- Duplication de données

**Impact** :
- Double pénalité sur les écritures
- Double consommation d'espace

#### Problème 5 : Bloat d'Index
**Symptôme** :
- Index fragmentés
- Taille d'index disproportionnée

**Impact** :
- Performance dégradée progressivement
- I/O excessif

### 2.2. Les Bénéfices d'un Audit d'Indexation

Un audit régulier permet de :

- **Optimiser les performances** : Identifier et combler les lacunes  
- **Réduire les coûts** : Supprimer les index inutiles  
- **Améliorer le débit** : Équilibrer lecture/écriture  
- **Simplifier la maintenance** : Moins d'index à gérer  
- **Prévenir les problèmes** : Détecter le bloat avant qu'il devienne critique

### 2.3. Quand Effectuer un Audit d'Indexation ?

Il est recommandé d'auditer les index :

- **Après le déploiement initial** : Valider la stratégie d'indexation  
- **Mensuellement** : Sur les systèmes critiques en production  
- **Après un changement de charge** : Nouvelles fonctionnalités, pic d'utilisation  
- **Suite à des problèmes de performance** : Requêtes lentes identifiées  
- **Après une migration** : PostgreSQL 17 → 18  
- **Avant un scaling up** : Optimiser avant d'ajouter du matériel

---

## 3. Anatomie d'un Index

### 3.1. Structure Physique

Un index PostgreSQL est composé de :

#### Pages d'Index (Blocks)
- Taille fixe de **8 KB** (comme les pages de tables)
- Contiennent les entrées d'index triées
- Organisées en arbre (B-Tree par défaut)

#### Entrées d'Index
Chaque entrée contient :
- **Clé** : La valeur de la colonne indexée  
- **Pointeur** : Référence vers la ligne de la table (TID - Tuple ID)

**Exemple** : Index sur `employes(nom)`
```
Index :
┌─────────────┬─────────────┐
│ Clé (nom)   │ TID         │
├─────────────┼─────────────┤
│ 'Dupont'    │ (2, 15)     │
│ 'Martin'    │ (1, 42)     │
│ 'Petit'     │ (3, 8)      │
└─────────────┴─────────────┘
```

### 3.2. Types de Scans

#### Sequential Scan (Seq Scan)
- Lit **toute la table** ligne par ligne
- Utilisé quand il n'y a pas d'index ou quand il est plus efficace

**Coût** : Proportionnel à la taille de la table

#### Index Scan
- Consulte d'abord l'index
- Puis lit les lignes pointées dans la table

**Coût** : Proportionnel au nombre de lignes retournées

#### Index Only Scan
- **Nouvelle fonctionnalité optimisée**
- Lit uniquement l'index sans accéder à la table
- Nécessite un index couvrant (covering index)

**Coût** : Minimal, le plus rapide

#### Bitmap Index Scan
- Combine plusieurs index
- Crée un bitmap en mémoire
- Puis lit les pages de la table

**Coût** : Intermédiaire, utilisé pour des requêtes avec OR ou AND

### 3.3. Le Planificateur de Requêtes

Le **planificateur** (query planner) décide quel type de scan utiliser en fonction de :

1. **Statistiques** : Données sur la distribution des valeurs (pg_stats)  
2. **Coût estimé** : Calcul du coût de chaque stratégie  
3. **Sélectivité** : Proportion de lignes retournées

**Formule simplifiée** :
```
Coût Index Scan = (Coût lecture index) + (Coût lecture lignes table)  
Coût Seq Scan = Coût lecture table complète  
```

Le planificateur choisit le scan avec le **coût le plus bas**.

### 3.4. Visibilité Map et Index Only Scan

PostgreSQL maintient une **Visibility Map** :
- Indique quelles pages de table contiennent uniquement des lignes visibles
- Permet les **Index Only Scans** (sans accès à la table)

**Condition** : La Visibility Map doit être à jour (via VACUUM)

---

## 4. Types d'Index dans PostgreSQL

PostgreSQL supporte plusieurs types d'index, chacun adapté à des cas d'usage spécifiques.

### 4.1. B-Tree (Par Défaut)

**Description** : Index équilibré en arbre, le type par défaut et le plus universel.

**Structure** : Arbre équilibré où chaque nœud contient plusieurs clés triées.

**Cas d'usage** :
- Égalité : `WHERE colonne = valeur`
- Comparaisons : `WHERE colonne > valeur`, `WHERE colonne BETWEEN a AND b`
- Tri : `ORDER BY colonne`
- Préfixes de chaîne : `WHERE colonne LIKE 'abc%'`

**Opérateurs supportés** :
```
<, <=, =, >=, >, BETWEEN, IN, IS NULL, IS NOT NULL
```

**Exemple** :
```sql
CREATE INDEX idx_employes_salaire ON employes(salaire);

-- Utilisera l'index
SELECT * FROM employes WHERE salaire > 50000;  
SELECT * FROM employes WHERE salaire BETWEEN 40000 AND 60000;  
SELECT * FROM employes ORDER BY salaire;  
```

**Avantages** :
- ✅ Très polyvalent  
- ✅ Excellentes performances générales  
- ✅ Supporte le tri

**Limites** :
- ❌ Pas optimal pour la recherche de texte intégral  
- ❌ Pas adapté aux données géométriques

---

### 4.2. Hash

**Description** : Index basé sur une fonction de hachage.

**Structure** : Table de hachage distribuant les valeurs dans des buckets.

**Cas d'usage** :
- **Uniquement** égalité stricte : `WHERE colonne = valeur`

**Opérateurs supportés** :
```
= (égalité uniquement)
```

**Exemple** :
```sql
CREATE INDEX idx_employes_email_hash ON employes USING hash(email);

-- Utilisera l'index
SELECT * FROM employes WHERE email = 'jean.dupont@example.com';

-- N'utilisera PAS l'index
SELECT * FROM employes WHERE email LIKE '%@example.com';
```

**Avantages** :
- ✅ Légèrement plus rapide que B-Tree pour l'égalité pure  
- ✅ Taille d'index potentiellement plus petite

**Limites** :
- ❌ **Aucun** support des comparaisons (<, >, BETWEEN)  
- ❌ **Aucun** support du tri  
- ❌ Moins polyvalent que B-Tree

**Quand utiliser Hash ?**
- Presque jamais en pratique
- B-Tree est généralement meilleur ou équivalent

---

### 4.3. GiST (Generalized Search Tree)

**Description** : Structure d'arbre généralisée, extensible à différents types de données.

**Structure** : Arbre non nécessairement équilibré, organisé selon un prédicat.

**Cas d'usage** :
- Données géométriques (via PostGIS)
- Recherche de texte intégral
- Plages et intervalles
- Arbres hiérarchiques (ltree)

**Opérateurs supportés** :
```
<<, &<, &>, >>, <<|, &<|, |&>, |>>, @>, <@, ~=, && (géométrie)
@@ (texte)
```

**Exemple** :
```sql
-- Géométrie avec PostGIS (cas typique GiST)
CREATE INDEX idx_lieux_geom ON lieux USING gist(geom);  
SELECT * FROM lieux WHERE ST_DWithin(geom, point_reference, 1000);  

-- Plages (range types) — GiST est le bon choix
CREATE INDEX idx_resa_periode ON reservations USING gist(periode);  
SELECT * FROM reservations WHERE periode && tstzrange('2026-01-01', '2026-01-31');  

-- Recherche de texte (full-text search)
-- GiST fonctionne, MAIS GIN est presque toujours préférable pour FTS
-- (lookups plus rapides). Voir section 4.4 GIN ci-dessous.
CREATE INDEX idx_documents_texte ON documents USING gist(to_tsvector('french', contenu));  
SELECT * FROM documents WHERE to_tsvector('french', contenu) @@ to_tsquery('postgresql');  
```

**Avantages** :
- ✅ Très flexible et extensible  
- ✅ Optimal pour les données spatiales (PostGIS) et les types `range`  
- ✅ Supporte de nombreux types de données  
- ✅ Écritures plus rapides que GIN (utile si la table est très mise à jour)

**Limites** :
- ❌ Plus lent que B-Tree pour les comparaisons simples  
- ❌ Taille d'index plus importante  
- ❌ Pour le **full-text search** et les **arrays/JSONB**, GIN est plus performant en lecture

---

### 4.4. GIN (Generalized Inverted Index)

**Description** : Index inversé généralisé, optimal pour les valeurs contenant plusieurs éléments.

**Structure** : Index inversé où chaque élément pointe vers les lignes qui le contiennent.

**Cas d'usage** :
- **JSONB** : Recherche dans des documents JSON  
- **Arrays** : Recherche d'éléments dans des tableaux  
- **Full-Text Search** : Recherche de texte intégral (plus rapide que GiST)  
- **hstore** : Paires clé-valeur

**Opérateurs supportés** :
```
@>, <@, && (arrays, jsonb)
@@ (full-text)
? (jsonb)
```

**Exemple** :
```sql
-- JSONB
CREATE INDEX idx_produits_attributs ON produits USING gin(attributs);  
SELECT * FROM produits WHERE attributs @> '{"couleur": "rouge"}';  

-- Array
CREATE INDEX idx_articles_tags ON articles USING gin(tags);  
SELECT * FROM articles WHERE tags @> ARRAY['postgresql', 'index'];  

-- Full-text search
CREATE INDEX idx_documents_fts ON documents USING gin(to_tsvector('french', contenu));  
SELECT * FROM documents WHERE to_tsvector('french', contenu) @@ to_tsquery('base & donnees');  
```

**Avantages** :
- ✅ Très rapide pour la recherche dans JSONB et arrays  
- ✅ Excellent pour full-text search  
- ✅ Taille d'index raisonnable

**Limites** :
- ❌ Écritures plus lentes (maintenance plus coûteuse)  
- ❌ Ne supporte pas toutes les opérations

**GIN vs GiST pour Full-Text** :
- **GIN** : Plus rapide en lecture, plus lent en écriture  
- **GiST** : Plus lent en lecture, plus rapide en écriture

Privilégiez **GIN** pour le full-text sauf si vous avez beaucoup d'écritures.

---

### 4.5. BRIN (Block Range Index)

**Description** : Index compact stockant des résumés par plages de blocs.

**Structure** : Pour chaque plage de blocs (par défaut 128), stocke min et max.

**Cas d'usage** :
- **Très grandes tables** (> 100 GB)
- Données **séquentielles** (ex: timestamps, IDs auto-incrémentés)
- Colonnes avec forte corrélation physique

**Opérateurs supportés** :
```
<, <=, =, >=, >, BETWEEN
```

**Exemple** :
```sql
-- Table de logs avec timestamp séquentiel
CREATE INDEX idx_logs_timestamp_brin ON logs USING brin(timestamp);  
SELECT * FROM logs WHERE timestamp >= '2025-01-01' AND timestamp < '2025-02-01';  
```

**Avantages** :
- ✅ **Très petit** : 1000× plus petit qu'un B-Tree équivalent  
- ✅ Maintenance rapide  
- ✅ Excellent pour les données séquentielles

**Limites** :
- ❌ Moins précis qu'un B-Tree (plus de faux positifs)  
- ❌ Nécessite une corrélation physique forte  
- ❌ Pas adapté aux données aléatoires

**Quand utiliser BRIN ?**
- Tables de logs avec timestamp
- Données de capteurs IoT
- Données de séries temporelles
- Tables de plus de 10 GB avec données séquentielles

---

### 4.6. SP-GiST (Space-Partitioned GiST)

**Description** : Index basé sur le partitionnement de l'espace, pour structures non équilibrées.

**Structure** : Arbre de partitionnement spatial (quadtree, radix tree, etc.)

**Cas d'usage** :
- Adresses IP (inet, cidr)
- Téléphones
- Données géométriques spécifiques
- Structures hiérarchiques non équilibrées

**Exemple** :
```sql
-- Adresses IP
CREATE INDEX idx_connexions_ip ON connexions USING spgist(adresse_ip);  
SELECT * FROM connexions WHERE adresse_ip << inet '192.168.1.0/24';  
```

**Avantages** :
- ✅ Optimal pour certaines structures spécifiques  
- ✅ Plus compact que GiST pour certains types

**Limites** :
- ❌ Cas d'usage très spécifiques  
- ❌ Moins universel que GiST

**Quand utiliser SP-GiST ?**
- Rarement en pratique
- Uniquement si vous avez des données très spécifiques (IP, quadtrees)

---

### 4.7. Tableau Récapitulatif : Types d'Index

| Type | Cas d'usage principal | Taille | Performance | Maintenance |
|------|----------------------|--------|-------------|-------------|
| **B-Tree** | Universel, comparaisons, tri | Moyenne | Excellente | Facile |
| **Hash** | Égalité stricte uniquement | Petite | Bonne | Facile |
| **GiST** | Géométrie, full-text, ltree | Grande | Moyenne | Moyenne |
| **GIN** | JSONB, arrays, full-text | Moyenne | Très bonne | Lente |
| **BRIN** | Grandes tables séquentielles | Très petite | Moyenne | Très rapide |
| **SP-GiST** | IP, structures non équilibrées | Petite | Bonne | Moyenne |

**Recommandation** : Utilisez **B-Tree** par défaut, sauf besoin spécifique.

---

## 5. Méthodologie d'Audit d'Indexation

### 5.1. Approche en 7 Étapes

#### Étape 1 : Inventaire des Index Existants

Listez tous les index de la base de données :

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes  
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')  
ORDER BY schemaname, tablename, indexname;  
```

**Documentation** : Créez un tableau avec :
- Nom de l'index
- Table associée
- Colonnes indexées
- Type d'index
- Taille
- But/justification

#### Étape 2 : Analyse de l'Utilisation

Identifiez quels index sont réellement utilisés :

```sql
SELECT
    schemaname,
    relname       AS tablename,    -- colonne réelle : relname (pas tablename)
    indexrelname  AS indexname,    -- colonne réelle : indexrelname
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes  
ORDER BY idx_scan ASC;  
```

> ℹ️ `pg_stat_user_indexes` expose **`relname`** pour la table porteuse et **`indexrelname`** pour l'index. Les noms `tablename`/`indexname` n'existent que dans la vue `pg_indexes`.

**Indicateurs** :
- `idx_scan = 0` : Index jamais utilisé  
- `idx_scan` faible : Index peu utilisé  
- `idx_scan` élevé : Index utile

#### Étape 3 : Identification des Index Manquants

Analysez les requêtes lentes pour détecter les sequential scans :

```sql
-- Avec pg_stat_statements
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements  
WHERE query LIKE '%Seq Scan%'  
ORDER BY total_exec_time DESC  
LIMIT 20;  
```

#### Étape 4 : Détection des Index Redondants

Cherchez les index qui se chevauchent :

```sql
-- Index sur (a, b) rend inutile un index sur (a) seul
SELECT
    i1.indexrelname AS index1,
    i2.indexrelname AS index2,
    i1.indkey AS columns1,
    i2.indkey AS columns2
FROM pg_index i1  
JOIN pg_index i2 ON i1.indrelid = i2.indrelid  
WHERE i1.indexrelid < i2.indexrelid  
  AND i1.indkey::text LIKE i2.indkey::text || '%';
```

#### Étape 5 : Mesure du Bloat

Calculez le bloat (fragmentation) des index :

```sql
-- Estimation rapide de la taille des index utilisateurs
-- (pour un bloat *précis*, voir la section 9 avec pgstattuple/pgstatindex)
SELECT
    schemaname,
    relname       AS tablename,
    indexrelname  AS indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes  
ORDER BY pg_relation_size(indexrelid) DESC  
LIMIT 20;  
```

> ⚠️ Il n'existe **pas** de formule simple pour estimer le *bloat* d'un B-tree à partir des seules vues système : `pg_relation_size(indexrelid, 'main')` n'est *pas* un indicateur de tuples vivants vs morts. Pour une mesure fiable, utilisez **`pgstatindex()`** (extension `pgstattuple`) — c'est l'objet de la section 9.

#### Étape 6 : Évaluation des Performances

Pour chaque index important :
- Exécutez EXPLAIN ANALYZE sur les requêtes concernées
- Mesurez le temps avec et sans l'index
- Vérifiez que l'index est effectivement utilisé

#### Étape 7 : Plan d'Action

Classez les actions par priorité :

**Haute priorité** :
- Créer les index manquants sur colonnes fréquemment requêtées
- Supprimer les index inutilisés coûteux

**Priorité moyenne** :
- REINDEX des index avec bloat > 30%
- Supprimer les index redondants

**Priorité basse** :
- Optimiser les index existants (index partiels, expressions, etc.)

---

## 6. Détection des Index Manquants

### 6.1. Signes d'Index Manquants

#### Symptôme 1 : Sequential Scans Fréquents

```sql
SELECT
    schemaname,
    relname AS tablename,         -- pg_stat_user_tables expose 'relname'
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_seq_tup_per_scan
FROM pg_stat_user_tables  
WHERE seq_scan > 0  
ORDER BY seq_scan DESC  
LIMIT 20;  
```

**Interprétation** :
- `seq_scan` élevé + `idx_scan` faible : Manque probable d'index  
- `avg_seq_tup_per_scan` faible : Table petite, index peut-être inutile

#### Symptôme 2 : Requêtes Lentes dans pg_stat_statements

```sql
-- Nécessite l'extension pg_stat_statements
SELECT
    substring(query, 1, 100) AS query_short,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements  
WHERE mean_exec_time > 100  -- Plus de 100 ms  
ORDER BY total_exec_time DESC  
LIMIT 20;  
```

Pour chaque requête lente, utilisez `EXPLAIN` pour voir si des sequential scans sont présents.

#### Symptôme 3 : Plans de Requêtes avec Seq Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT * FROM employes WHERE nom = 'Dupont';  
```

Si vous voyez :
```
Seq Scan on employes  (cost=0.00..1234.56 rows=1 width=128)
  Filter: (nom = 'Dupont'::text)
```

Un index sur `nom` accélérerait la requête.

### 6.2. Colonnes à Indexer en Priorité

#### Colonnes dans WHERE

```sql
-- Exemple : Requête fréquente
SELECT * FROM commandes WHERE client_id = 123;

-- Index recommandé
CREATE INDEX idx_commandes_client_id ON commandes(client_id);
```

#### Colonnes de Jointure

```sql
-- Exemple : Jointure fréquente
SELECT *  
FROM commandes c  
JOIN clients cl ON c.client_id = cl.id;  

-- Index recommandés
CREATE INDEX idx_commandes_client_id ON commandes(client_id);
-- (l'index sur clients.id existe déjà via la PRIMARY KEY)
```

#### Colonnes dans ORDER BY

```sql
-- Exemple : Tri fréquent
SELECT * FROM produits ORDER BY prix DESC;

-- Index recommandé
CREATE INDEX idx_produits_prix ON produits(prix DESC);
```

**Note** : L'ordre du tri (ASC/DESC) peut être spécifié dans l'index.

#### Clés Étrangères

**Règle importante** : PostgreSQL ne crée **pas** automatiquement d'index sur les clés étrangères.

```sql
-- Définition de FK
ALTER TABLE commandes  
ADD CONSTRAINT fk_client  
FOREIGN KEY (client_id) REFERENCES clients(id);  

-- Index à créer MANUELLEMENT
CREATE INDEX idx_commandes_client_id ON commandes(client_id);
```

**Sans cet index** : Les jointures et les DELETE sur `clients` seront très lents.

### 6.3. Méthode Systématique : HypoPG

**HypoPG** est une extension permettant de tester des index **hypothétiques** sans les créer réellement.

#### Installation

```sql
CREATE EXTENSION hypopg;
```

#### Utilisation

```sql
-- 1. Créer un index hypothétique
SELECT hypopg_create_index('CREATE INDEX ON employes(nom)');

-- 2. Tester une requête
EXPLAIN SELECT * FROM employes WHERE nom = 'Dupont';

-- 3. Voir si l'index hypothétique est utilisé
-- Si oui, le plan montrera "Index Scan using <oid>"

-- 4. Supprimer les index hypothétiques
SELECT hypopg_reset();
```

**Avantages** :
- ✅ Aucun impact sur la production  
- ✅ Test instantané  
- ✅ Permet de valider avant de créer

---

## 7. Détection des Index Inutilisés

### 7.1. Identification des Index Inutilisés

#### Requête Principale

```sql
SELECT
    schemaname,
    relname       AS table_name,    -- pg_stat_user_indexes : 'relname' (table)
    indexrelname  AS index_name,    -- pg_stat_user_indexes : 'indexrelname'
    idx_scan,
    last_idx_scan,                  -- 🆕 PG 16+ : dernière utilisation (timestamptz)
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_relation_size(indexrelid) AS size_bytes
FROM pg_stat_user_indexes  
WHERE idx_scan = 0  
  AND indexrelname NOT LIKE '%_pkey'  -- Exclure les PK
ORDER BY pg_relation_size(indexrelid) DESC;
```

> 💡 **Astuce PG 16+** : la colonne `last_idx_scan` (`timestamp with time zone`) indique la **date du dernier scan** de l'index. Plus précise que `idx_scan = 0` pour distinguer un index nouveau (jamais scanné encore) d'un index réellement abandonné depuis plusieurs mois. Disponible aussi dans `pg_stat_all_tables` (`last_seq_scan`, `last_idx_scan`).

**Résultat** : Liste des index jamais utilisés, triés par taille.

### 7.2. Critères de Décision

Avant de supprimer un index inutilisé, vérifiez :

#### 1. Âge des Statistiques

```sql
-- Depuis quand les stats sont-elles collectées ?
SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();
```

Si les stats ont été réinitialisées récemment, `idx_scan = 0` peut être temporaire.

#### 2. Contraintes d'Unicité

```sql
-- Vérifier si l'index sert pour une contrainte UNIQUE ou PK
SELECT
    i.relname AS index_name,
    c.contype AS constraint_type
FROM pg_index ix  
JOIN pg_class i ON i.oid = ix.indexrelid  
LEFT JOIN pg_constraint c ON c.conindid = ix.indexrelid  
WHERE i.relname = 'nom_de_lindex';  
```

**Types de contraintes** :
- `p` : PRIMARY KEY  
- `u` : UNIQUE  
- `f` : FOREIGN KEY

**Ne supprimez jamais** un index servant à une contrainte.

#### 3. Index sur Clés Étrangères

Même si `idx_scan = 0`, un index sur FK peut être utile pour :
- Accélérer les DELETE/UPDATE sur la table référencée
- Éviter les locks lors de vérifications d'intégrité

**Recommandation** : Gardez les index sur FK même s'ils semblent inutilisés.

#### 4. Requêtes Saisonnières/Rares

Certaines requêtes sont rares mais critiques :
- Rapports mensuels
- Batch nocturnes
- Analyses ad-hoc

Documentez ces cas et marquez les index comme "utiles malgré faible utilisation".

### 7.3. Procédure de Suppression Sécurisée

**Étape 1** : Documenter l'index
```sql
-- Sauvegarder la définition
SELECT indexdef FROM pg_indexes WHERE indexname = 'nom_index';
```

**Étape 2** : Désactiver temporairement (PostgreSQL 11+)
```sql
-- L'index existe mais n'est plus utilisé par le planificateur
ALTER INDEX nom_index SET (FILLFACTOR = 100);  -- Pas de désactivation directe
-- Alternative : DROP l'index puis recréer si besoin
```

**Note** : PostgreSQL n'a pas de commande native pour désactiver un index. Pour tester :

**Étape 3** : Créer avec CONCURRENTLY pour pouvoir revenir
```sql
-- Conserver la définition pour recréation rapide
CREATE INDEX CONCURRENTLY idx_backup_nomindex ON table(colonne);
```

**Étape 4** : Supprimer l'index inutilisé
```sql
DROP INDEX CONCURRENTLY nom_index;
```

**Étape 5** : Surveiller pendant 1-2 semaines
- Vérifier qu'aucune dégradation de performance
- Vérifier les logs pour des erreurs

**Étape 6** : Si problème, recréer l'index
```sql
-- Utiliser CONCURRENTLY pour ne pas bloquer les écritures
CREATE INDEX CONCURRENTLY nom_index ON table(colonne);
```

### 7.4. Index Protégés (À Ne Jamais Supprimer)

Ne supprimez **jamais** :

1. **Index PRIMARY KEY** : `table_pkey`  
2. **Index UNIQUE** : Servant à contrainte d'unicité  
3. **Index de FK** : Sur colonnes de clés étrangères  
4. **Index système** : Dans `pg_catalog`

---

## 8. Détection des Index Redondants

### 8.1. Types de Redondance

#### Redondance Complète

**Cas 1 : Duplication exacte**
```sql
CREATE INDEX idx1 ON employes(nom);  
CREATE INDEX idx2 ON employes(nom);  -- Redondant !  
```

**Détection** :
```sql
SELECT
    i1.indexrelname AS index1,
    i2.indexrelname AS index2,
    i1.indkey AS columns
FROM pg_index i1  
JOIN pg_index i2 ON i1.indrelid = i2.indrelid  
WHERE i1.indexrelid < i2.indexrelid  
  AND i1.indkey = i2.indkey;
```

#### Redondance Partielle (Left-Prefix)

**Cas 2 : Index multi-colonnes vs simple**
```sql
CREATE INDEX idx_employes_nom_prenom ON employes(nom, prenom);  
CREATE INDEX idx_employes_nom ON employes(nom);  -- Potentiellement redondant  
```

**Règle** : Un index sur `(a, b)` peut souvent remplacer un index sur `(a)`.

**Exceptions** :
- Si l'index sur `(a)` est plus petit et suffit pour la majorité des requêtes
- Si l'index sur `(a, b)` est trop large (pénalise les écritures)

**Détection** :
```sql
SELECT
    i1.indexrelname AS wide_index,
    i2.indexrelname AS narrow_index,
    i1.indkey AS wide_columns,
    i2.indkey AS narrow_columns
FROM pg_index i1  
JOIN pg_index i2 ON i1.indrelid = i2.indrelid  
WHERE i1.indexrelid <> i2.indexrelid  
  AND i1.indkey::text LIKE i2.indkey::text || '%'
  AND array_length(string_to_array(i1.indkey::text, ' '), 1) >
      array_length(string_to_array(i2.indkey::text, ' '), 1);
```

### 8.2. Cas d'Usage : Quand Garder les Deux ?

#### Situation 1 : Différence de Taille Significative

Si l'index multi-colonnes est beaucoup plus large :

```sql
-- Index large (100 MB)
CREATE INDEX idx_large ON logs(timestamp, user_id, action, details);

-- Index petit (10 MB) pour requête fréquente
CREATE INDEX idx_petit ON logs(timestamp);
```

**Justification** : Le petit index est plus rapide pour les requêtes simples.

#### Situation 2 : Index Partiel Spécialisé

```sql
-- Index général
CREATE INDEX idx_commandes_statut ON commandes(statut);

-- Index partiel pour statut spécifique (plus petit, plus rapide)
CREATE INDEX idx_commandes_encours ON commandes(statut)  
WHERE statut = 'en_cours';  
```

**Non redondant** : L'index partiel est plus efficace pour les requêtes sur 'en_cours'.

#### Situation 3 : Ordre de Tri Différent

```sql
-- Tri ascendant
CREATE INDEX idx_produits_prix_asc ON produits(prix ASC);

-- Tri descendant
CREATE INDEX idx_produits_prix_desc ON produits(prix DESC);
```

**Note** : Depuis PostgreSQL 8.3, un seul index suffit généralement (B-Tree peut être parcouru dans les deux sens).

### 8.3. Stratégie de Consolidation

**Étape 1** : Identifier les paires redondantes

**Étape 2** : Analyser l'utilisation
```sql
SELECT
    indexrelname,
    idx_scan,
    idx_tup_read
FROM pg_stat_user_indexes  
WHERE indexrelname IN ('index1', 'index2');  
```

**Étape 3** : Décider lequel garder
- Si `idx_scan` similaire : Garder le plus large
- Si `idx_scan` très différent : Garder le plus utilisé

**Étape 4** : Tester avec HypoPG (si disponible)
```sql
-- Supprimer hypothétiquement l'index
SELECT hypopg_drop_index(oid) FROM hypopg() WHERE indexname = 'index2';

-- Tester les requêtes
EXPLAIN ANALYZE SELECT ...;

-- Si OK, supprimer réellement
DROP INDEX CONCURRENTLY index2;
```

---

## 9. Bloat et Fragmentation des Index

### 9.1. Qu'est-ce que le Bloat ?

Le **bloat** (gonflement) est l'espace perdu dans un index dû à :
- Suppressions de lignes (DELETE)
- Modifications (UPDATE)
- Fragmentations internes

**Causes** :
- MVCC : Les anciennes versions restent temporairement
- VACUUM insuffisant
- Index non maintenus

**Impact** :
- Index plus gros que nécessaire
- Performances dégradées (plus d'I/O)
- Gaspillage d'espace disque

### 9.2. Détection du Bloat

#### Méthode 1 : Estimation Simple

```sql
SELECT
    schemaname,
    relname        AS tablename,
    indexrelname   AS indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_size_pretty(pg_relation_size(relid))      AS table_size,
    ROUND(100 * pg_relation_size(indexrelid)::numeric /
          NULLIF(pg_relation_size(relid), 0), 2) AS index_to_table_ratio
FROM pg_stat_user_indexes  
WHERE schemaname = 'public'  
ORDER BY pg_relation_size(indexrelid) DESC  
LIMIT 20;  
```

> 💡 Pas besoin de jointure sur `pg_class` : la vue `pg_stat_user_indexes` expose déjà `relid` (oid de la table porteuse de l'index), directement utilisable avec `pg_relation_size()`.

**Heuristique** : Si `index_to_table_ratio > 100 %`, investiguer le bloat (ou simplement la sur-indexation).

#### Méthode 2 : Extension pgstattuple

```sql
-- Installation
CREATE EXTENSION pgstattuple;

-- Analyse d'un index
SELECT * FROM pgstatindex('nom_de_lindex');
```

**Colonnes importantes** :
- `leaf_fragmentation` : % de fragmentation (> 50% = mauvais)  
- `avg_leaf_density` : Densité moyenne des pages (< 70% = bloat)

**Interprétation** :
- `avg_leaf_density > 90%` : Excellent  
- `avg_leaf_density 70-90%` : Correct  
- `avg_leaf_density < 70%` : Bloat significatif, REINDEX recommandé

#### Méthode 3 : Requête Complète de Bloat

```sql
-- Mesure réelle du bloat via pgstattuple (à exécuter hors heures de pointe)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    schemaname,
    relname       AS tablename,
    indexrelname  AS indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    (s.avg_leaf_density)::numeric(5,2)           AS avg_leaf_density_pct,
    (s.leaf_fragmentation)::numeric(5,2)         AS leaf_fragmentation_pct
FROM pg_stat_user_indexes ui  
CROSS JOIN LATERAL pgstatindex(format('%I.%I', schemaname, indexrelname)) AS s  
WHERE pg_relation_size(indexrelid) > 1024 * 1024  -- > 1 Mo  
ORDER BY pg_relation_size(indexrelid) DESC  
LIMIT 50;  
```

> ⚠️ `pgstatindex()` lit l'index ligne à ligne. C'est **précis** mais **coûteux** : à éviter sur des index > 10 Go pendant les heures pleines. Pour une estimation grossière, on se contente du ratio taille_index / taille_table (méthode 1).

### 9.3. Solutions au Bloat

#### Solution 1 : VACUUM (Préventif)

```sql
-- VACUUM simple (libère l'espace pour réutilisation interne)
VACUUM employes;

-- VACUUM ANALYZE (met aussi à jour les statistiques)
VACUUM ANALYZE employes;

-- VACUUM VERBOSE (affiche les détails)
VACUUM VERBOSE employes;
```

**Limite** : VACUUM ne réduit **pas** la taille physique, il marque l'espace comme réutilisable.

#### Solution 2 : REINDEX (Correctif)

```sql
-- REINDEX d'un index spécifique
REINDEX INDEX nom_index;

-- REINDEX de tous les index d'une table
REINDEX TABLE employes;

-- REINDEX CONCURRENTLY (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY nom_index;
```

**REINDEX CONCURRENTLY** :
- ✅ N'acquiert pas de verrou exclusif (juste un `ShareUpdateExclusiveLock`)  
- ✅ La table reste accessible en lecture/écriture  
- ❌ Plus lent qu'un REINDEX normal  
- ❌ Nécessite un espace disque temporaire (2× la taille de l'index)  
- ❌ Ne peut PAS s'exécuter dans une transaction explicite (`BEGIN`/`COMMIT`)  
- ⚠️ **En cas d'échec** : laisse un index en état `INVALID` (visible via `pg_index.indisvalid = false`). Il doit être supprimé manuellement avec `DROP INDEX CONCURRENTLY` avant de retenter.

**REINDEX normal** :
- ✅ Plus rapide  
- ❌ Verrou exclusif (table bloquée en écriture)

**Recommandation** : Utilisez toujours `CONCURRENTLY` en production. Surveillez les index `indisvalid = false` après chaque opération (cf. `03-scripts-monitoring-systeme.md`, section « Invalid Indexes »).

```sql
-- Détecter les index laissés en état INVALID après un échec de
-- CREATE INDEX CONCURRENTLY ou REINDEX CONCURRENTLY
SELECT
    n.nspname AS schemaname,
    t.relname AS tablename,
    i.relname AS indexname
FROM pg_index x  
JOIN pg_class i      ON i.oid = x.indexrelid  
JOIN pg_class t      ON t.oid = x.indrelid  
JOIN pg_namespace n  ON n.oid = i.relnamespace  
WHERE NOT x.indisvalid  
  AND n.nspname NOT IN ('pg_catalog', 'information_schema');
```

#### Solution 3 : pg_repack (Extension)

**pg_repack** est une extension qui réorganise tables et index **sans verrouillage**.

```bash
# Installation (exemple Debian/Ubuntu)
apt-get install postgresql-18-repack

# Utilisation
pg_repack -d ma_base -t employes
```

**Avantages** :
- ✅ Aucun verrou exclusif  
- ✅ Réorganise table + index  
- ✅ Supprime le bloat

**Inconvénient** :
- ❌ Nécessite espace disque temporaire (2× la taille)

#### Solution 4 : CREATE INDEX CONCURRENTLY + DROP

Alternative à REINDEX CONCURRENTLY :

```sql
-- 1. Créer un nouvel index identique (peut échouer et laisser un index INVALID)
CREATE INDEX CONCURRENTLY idx_employes_nom_new ON employes(nom);

-- 2. Vérifier qu'il est marqué valide (indisvalid = true)
SELECT i.relname, x.indisvalid  
FROM pg_index x JOIN pg_class i ON i.oid = x.indexrelid  
WHERE i.relname = 'idx_employes_nom_new';  

-- 3. Si invalide → DROP INDEX CONCURRENTLY puis relancer le CREATE INDEX
-- 4. Si valide → supprimer l'ancien index
DROP INDEX CONCURRENTLY idx_employes_nom;

-- 5. Renommer le nouveau (opération métadonnée, instantanée)
ALTER INDEX idx_employes_nom_new RENAME TO idx_employes_nom;
```

**Avantages** :
- ✅ Flexibilité maximale (permet de modifier le type d'index, les colonnes, etc.)  
- ✅ Aucun verrou exclusif à aucune étape  
- ⚠️ **Caveat** : `CREATE INDEX CONCURRENTLY` doit s'exécuter hors transaction (un `BEGIN` invalide la commande).

### 9.4. Planification de la Maintenance

#### Fréquence Recommandée

| Bloat estimé | Action | Fréquence |
|--------------|--------|-----------|
| < 20% | Aucune | - |
| 20-50% | VACUUM | Hebdomadaire |
| > 50% | REINDEX CONCURRENTLY | Mensuel |
| > 80% | REINDEX ou pg_repack | Immédiat |

#### Automatisation

```sql
-- Squelette d'audit hebdomadaire (à compléter avec votre canal d'alerte)
DO $$  
DECLARE  
    idx_record RECORD;
    leaf_density numeric;
BEGIN
    FOR idx_record IN
        SELECT schemaname, relname AS tablename, indexrelname,
               pg_relation_size(indexrelid) AS size
        FROM pg_stat_user_indexes
        WHERE pg_relation_size(indexrelid) > 100 * 1024 * 1024  -- > 100 Mo
    LOOP
        -- Récupérer la densité de feuilles via pgstatindex
        EXECUTE format(
            'SELECT avg_leaf_density FROM pgstatindex(%L)',
            format('%I.%I', idx_record.schemaname, idx_record.indexrelname)
        ) INTO leaf_density;

        IF leaf_density < 50 THEN
            RAISE NOTICE 'Bloat sévère : index %.% (densité=%, taille=% Mo)',
                idx_record.schemaname,
                idx_record.indexrelname,
                round(leaf_density, 1),
                round(idx_record.size / 1024.0 / 1024.0, 1);
        END IF;
    END LOOP;
END $$;
```

> 💡 On qualifie l'index avec `format('%I.%I', schema, indexrelname)` plutôt qu'avec son seul nom : sinon `pgstatindex` cherche dans le `search_path` et peut échouer pour les index hors `public`.

---

## 10. Performance des Index

### 10.1. Métriques de Performance

#### Cache Hit Ratio des Index

```sql
SELECT
    schemaname,
    relname       AS tablename,     -- pg_statio_user_indexes : 'relname'
    indexrelname  AS indexname,     -- pg_statio_user_indexes : 'indexrelname'
    idx_blks_read,
    idx_blks_hit,
    CASE
        WHEN idx_blks_hit + idx_blks_read = 0 THEN 0
        ELSE ROUND(100.0 * idx_blks_hit / (idx_blks_hit + idx_blks_read), 2)
    END AS cache_hit_ratio
FROM pg_statio_user_indexes  
WHERE idx_blks_hit + idx_blks_read > 0  
ORDER BY cache_hit_ratio ASC  
LIMIT 20;  
```

**Objectif** : `cache_hit_ratio > 95%`

**Si < 90%** :
- Index trop gros pour tenir en mémoire
- Workload I/O intensive
- `shared_buffers` ou RAM insuffisante

#### Utilisation vs Taille

```sql
SELECT
    schemaname,
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    CASE
        WHEN idx_scan = 0 THEN 0
        ELSE pg_relation_size(indexrelid) / idx_scan
    END AS bytes_per_scan
FROM pg_stat_user_indexes  
WHERE pg_relation_size(indexrelid) > 10 * 1024 * 1024  -- > 10 MB  
ORDER BY bytes_per_scan DESC  
LIMIT 20;  
```

**Interprétation** :
- `bytes_per_scan` élevé : Index large utilisé rarement (candidat à suppression)  
- `bytes_per_scan` faible : Index efficace

### 10.2. Analyse avec EXPLAIN

#### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)  
SELECT * FROM employes WHERE nom = 'Dupont';  
```

**Sortie type** :
```
Index Scan using idx_employes_nom on public.employes
  (cost=0.42..8.44 rows=1 width=128)
  (actual time=0.023..0.024 rows=1 loops=1)
  Index Cond: (nom = 'Dupont'::text)
  Buffers: shared hit=4
Planning Time: 0.123 ms  
Execution Time: 0.045 ms  
```

**Éléments à analyser** :
- **Type de scan** : Index Scan (bon) vs Seq Scan (mauvais si grande table)  
- **Buffers** : `shared hit=4` signifie 4 pages lues depuis le cache  
- **Execution Time** : Temps réel d'exécution  
- **rows estimé vs actual** : Si très différent, statistiques obsolètes

#### Forcer/Désactiver un Index

**Désactiver les index pour tester** :
```sql
SET enable_indexscan = off;  
EXPLAIN SELECT * FROM employes WHERE nom = 'Dupont';  
SET enable_indexscan = on;  
```

**Forcer un index spécifique** (non recommandé en production) :
PostgreSQL n'a pas de syntaxe pour forcer un index, mais vous pouvez influencer le planificateur.

### 10.3. Index Selectivity (Sélectivité)

La **sélectivité** mesure la capacité d'un index à filtrer efficacement les données.

**Formule** :
```
Sélectivité = Nombre de valeurs distinctes / Nombre total de lignes
```

**Sélectivité élevée (proche de 1)** :
- Peu de doublons
- Index très efficace
- Exemple : email, numéro de sécurité sociale

**Sélectivité faible (proche de 0)** :
- Beaucoup de doublons
- Index peu efficace
- Exemple : sexe (M/F), booléen

#### Calcul de la Sélectivité

```sql
SELECT
    attname AS column_name,
    n_distinct,
    most_common_vals,
    most_common_freqs
FROM pg_stats  
WHERE tablename = 'employes'  
  AND schemaname = 'public'
ORDER BY abs(n_distinct) DESC;
```

**Interprétation** :
- `n_distinct > 0` : Nombre de valeurs distinctes  
- `n_distinct = -1` : Toutes les valeurs sont distinctes (sélectivité maximale)  
- `n_distinct proche de 0` : Peu de valeurs distinctes (sélectivité faible)

**Règle** : N'indexez pas les colonnes avec `n_distinct < 100` (sauf cas particulier).

---

## 11. Stratégies d'Indexation Avancées

### 11.1. Index Multi-Colonnes (Composite)

**Définition** : Index sur plusieurs colonnes.

```sql
CREATE INDEX idx_employes_nom_prenom ON employes(nom, prenom);
```

#### Ordre des Colonnes

**Règle critique** : L'ordre des colonnes dans l'index est **essentiel**.

**Principe** : Placez les colonnes les plus sélectives **en premier**.

**Exemple** :
```sql
-- Bon : nom (sélectif) puis prenom
CREATE INDEX idx_bon ON employes(nom, prenom);

-- Mauvais : prenom puis nom (si prenom est moins sélectif)
CREATE INDEX idx_mauvais ON employes(prenom, nom);
```

#### Utilisation de l'Index Multi-Colonnes

Un index sur `(a, b, c)` peut être utilisé pour :
- ✅ `WHERE a = ... AND b = ... AND c = ...`  
- ✅ `WHERE a = ... AND b = ...`  
- ✅ `WHERE a = ...`  
- ❌ `WHERE b = ...` (colonne non en préfixe)  
- ❌ `WHERE c = ...` (colonne non en préfixe)

**Règle du left-prefix** : L'index est utilisable seulement si les colonnes sont filtrées **de gauche à droite**.

#### PostgreSQL 18 : Skip Scan

**Nouveauté** : PostgreSQL 18 introduit l'optimisation **Skip Scan**.

Avec Skip Scan, un index sur `(a, b)` peut maintenant être utilisé pour `WHERE b = ...` (sans filtrer `a`).

**Exemple** :
```sql
-- Index multi-colonnes
CREATE INDEX idx_commandes_client_date ON commandes(client_id, date_commande);

-- Avant PostgreSQL 18 : Index NON utilisé
SELECT * FROM commandes WHERE date_commande = '2025-01-01';

-- PostgreSQL 18 : Index PEUT être utilisé grâce à Skip Scan
SELECT * FROM commandes WHERE date_commande = '2025-01-01';
```

**Impact** : Réduit le besoin de créer des index redondants.

### 11.2. Index Partiels

**Définition** : Index qui ne contient qu'un sous-ensemble des lignes.

```sql
CREATE INDEX idx_commandes_encours  
ON commandes(date_commande)  
WHERE statut = 'en_cours';  
```

**Avantages** :
- ✅ Plus petit que l'index complet  
- ✅ Plus rapide (moins de données à scanner)  
- ✅ Maintenance plus légère

**Cas d'usage** :
```sql
-- Requête fréquente sur un statut spécifique
SELECT * FROM commandes  
WHERE statut = 'en_cours'  
  AND date_commande > '2025-01-01';
```

L'index partiel est optimal pour cette requête.

**Attention** : L'index partiel n'est utilisé **que si la condition WHERE de la requête correspond exactement à la clause WHERE de l'index**.

#### Exemples d'Index Partiels

**Filtrer les NULL** :
```sql
-- Index uniquement sur les lignes avec email non NULL
CREATE INDEX idx_users_email_not_null ON users(email) WHERE email IS NOT NULL;
```

**Filtrer sur date récente** :
```sql
-- Index uniquement sur les 30 derniers jours
CREATE INDEX idx_logs_recent ON logs(timestamp)  
WHERE timestamp > CURRENT_DATE - INTERVAL '30 days';  
```

**Filtrer par statut actif** :
```sql
-- Index uniquement sur comptes actifs
CREATE INDEX idx_users_actifs ON users(last_login) WHERE actif = true;
```

### 11.3. Index sur Expressions

**Définition** : Index sur le résultat d'une fonction ou expression.

```sql
-- Recherche insensible à la casse
CREATE INDEX idx_employes_nom_lower ON employes(lower(nom));

-- Requête utilisera l'index
SELECT * FROM employes WHERE lower(nom) = 'dupont';
```

#### Cas d'Usage Fréquents

**Fonctions de texte** :
```sql
-- lower() pour recherche case-insensitive
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- substring() pour préfixes
CREATE INDEX idx_produits_code_prefix ON produits(substring(code, 1, 3));
```

**Fonctions de date** :
```sql
-- Extraire l'année
CREATE INDEX idx_commandes_annee ON commandes(EXTRACT(YEAR FROM date_commande));

-- Date sans heure
CREATE INDEX idx_logs_date ON logs(date_trunc('day', timestamp));
```

**Expressions arithmétiques** :
```sql
-- Prix TTC (prix HT × 1.20)
CREATE INDEX idx_produits_ttc ON produits(prix * 1.20);
```

**JSONB** :
```sql
-- Extraire un champ JSON
CREATE INDEX idx_users_prefs_lang ON users((preferences->>'language'));
```

### 11.4. Index INCLUDE (Covering Index)

**Définition** : Index contenant des colonnes supplémentaires non indexées, pour permettre des **Index Only Scans**.

```sql
CREATE INDEX idx_employes_nom_include  
ON employes(nom)  
INCLUDE (prenom, salaire);  
```

**Avantage** : La requête peut être satisfaite entièrement depuis l'index, sans accéder à la table.

```sql
-- Index Only Scan (très rapide)
SELECT nom, prenom, salaire FROM employes WHERE nom = 'Dupont';
```

**Colonnes INCLUDE** :
- Ne participent **pas** à l'ordre de l'index
- Simplement stockées pour être disponibles lors d'un Index Only Scan

**Cas d'usage** :
- Requêtes avec SELECT sur colonnes spécifiques
- Colonnes non sélectives (inutiles pour l'index) mais nécessaires dans le SELECT

#### Exemple Complet

```sql
-- Sans INCLUDE : Index Scan + lecture table
CREATE INDEX idx_commandes_client ON commandes(client_id);  
SELECT client_id, total FROM commandes WHERE client_id = 123;  
-- Plan : Index Scan → Lit index puis table

-- Avec INCLUDE : Index Only Scan
CREATE INDEX idx_commandes_client_include  
ON commandes(client_id)  
INCLUDE (total);  
SELECT client_id, total FROM commandes WHERE client_id = 123;  
-- Plan : Index Only Scan → Lit uniquement index (plus rapide)
```

### 11.5. Index Uniques

**Définition** : Index garantissant l'unicité des valeurs.

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

**Effets** :
- Contrainte d'unicité appliquée
- Amélioration des performances (le planificateur sait qu'une seule ligne sera retournée)

**Note** : Les contraintes PRIMARY KEY et UNIQUE créent automatiquement des index uniques.

### 11.6. Index avec Ordre de Tri

**Définition** : Spécifier l'ordre de tri (`ASC`/`DESC`) et la gestion des `NULL` dans l'index.

```sql
CREATE INDEX idx_produits_prix_desc  
ON produits(prix DESC NULLS LAST);  
```

**Cas d'usage** :

```sql
-- La requête bénéficie directement de l'ordre stocké dans l'index
SELECT * FROM produits ORDER BY prix DESC NULLS LAST LIMIT 10;
```

**Options** :
- `ASC` : ordre croissant (défaut).
- `DESC` : ordre décroissant.
- `NULLS FIRST` : les `NULL` en premier (défaut pour `DESC`).
- `NULLS LAST` : les `NULL` en dernier (défaut pour `ASC`).

> ⚠️ **Précision importante** : depuis PostgreSQL 8.3, un index B-tree peut être parcouru **dans les deux sens**, donc un index `(a ASC)` sert aussi pour `ORDER BY a DESC`. **MAIS** cette symétrie ne joue qu'à l'échelle de **l'index entier** :  
>  
> - Pour `ORDER BY a ASC, b ASC` → l'index `(a ASC, b ASC)` (ou son inverse complet `(a DESC, b DESC)`) est utilisable.  
> - Pour `ORDER BY a ASC, b DESC` → il faut **explicitement** un index `(a ASC, b DESC)` ou `(a DESC, b ASC)`. Un index `(a, b)` (tout ASC) ne suffit pas.  
>  
> Cas typique : pagination ascendante par date + tri descendant par score → `CREATE INDEX … (created_at ASC, score DESC)`.

---

## 12. PostgreSQL 18 : Nouveautés d'Indexation

### 12.1. Skip Scan (B-tree multi-colonnes)

**Le problème historique.** Un index B-tree composite `(a, b)` n'est utilisable efficacement que si la requête filtre sur `a` (la colonne **de tête**). Une requête `WHERE b = …` sans condition sur `a` force un *Sequential Scan*, sauf à créer un index dédié `(b)`.

**Ce que PG 18 apporte.** Le planificateur peut désormais générer un **Skip Scan** : pour chaque valeur distincte de `a`, il « saute » dans l'index jusqu'aux entrées qui satisfont `b = …`. Coût ≈ `nb_valeurs_distinctes(a) × log(N)` plutôt que `N` (scan séquentiel).

**Quand c'est rentable** :
- La colonne de tête a une **cardinalité faible** (peu de valeurs distinctes) — ex. `tenant_id`, `region`, `pays`, statut.
- La colonne suivante est **sélective**.
- Le coût estimé du Skip Scan reste inférieur au *Seq Scan*.

**Exemple** :
```sql
CREATE INDEX idx_ventes_region_date ON ventes(region, date_vente);

-- PG ≤ 17 : Seq Scan (région non filtrée) → lent
-- PG 18  : Skip Scan possible, surtout si peu de régions distinctes
EXPLAIN ANALYZE  
SELECT * FROM ventes WHERE date_vente = DATE '2026-01-01';  
```

**Limites connues** :
- Skip Scan ne supprime *pas* le besoin d'un index `(date_vente)` si vous avez **beaucoup** de régions (le saut devient coûteux).
- Ne s'applique qu'aux index **B-tree** (pas GIN, GiST, BRIN, etc.).
- Vérifier `EXPLAIN` pour confirmer l'utilisation (mention « *Index Scan ... using ...* » + filtre `b` dans la condition d'index).

---

### 12.2. Transformation `OR` → `= ANY (…)`

**Avant PG 18.** Le planificateur convertissait `IN (1,2,3)` en `= ANY (ARRAY[1,2,3])` (plus efficace pour les index), mais **pas** les `OR` équivalents : `col = 1 OR col = 2 OR col = 3` restait littéral et empêchait souvent l'utilisation d'un index.

**PG 18.** Le planificateur **réécrit** automatiquement ces chaînes de `OR` portant sur la **même colonne** avec des constantes en une forme `= ANY (…)`, ce qui permet enfin l'*Index Scan* / *Bitmap Index Scan*.

```sql
-- Écriture utilisateur (PG 18)
SELECT * FROM employes WHERE id = 1 OR id = 2 OR id = 3;

-- Réécriture interne par le planificateur
SELECT * FROM employes WHERE id = ANY (ARRAY[1, 2, 3]);
```

> ℹ️ La transformation se limite aux cas où **tous** les `OR` portent sur la même expression de gauche et des constantes (ou des paramètres) à droite. Un `OR` mixte (`a = 1 OR b = 2`) reste tel quel.

---

### 12.3. Auto-élimination des self-joins

**Description.** Quand le planificateur détecte qu'une jointure d'une table avec elle-même sur sa **clé primaire** ne produit qu'une projection (et que toutes les colonnes utilisées proviennent d'un seul des deux côtés), il **supprime** la jointure.

```sql
-- Self-join redondant (ex. produit par un ORM)
SELECT e1.nom  
FROM employes e1  
JOIN employes e2 ON e1.id = e2.id  
WHERE e1.salaire > 50000;  

-- PG 18 simplifie en interne en :
SELECT nom FROM employes WHERE salaire > 50000;
```

**Impact sur les index** : moins d'accès à l'index PK, moins de tuples manipulés.

**Quand c'est utile** : ORMs, vues empilées, générateurs SQL qui produisent des jointures redondantes.

---

### 12.4. `EXPLAIN (ANALYZE)` plus bavard par défaut

PG 18 intègre désormais les compteurs **BUFFERS** dans la sortie d'`EXPLAIN ANALYZE` **sans avoir à les demander explicitement**. On voit pour chaque nœud combien de blocs ont été lus en cache (`shared hit`), en disque (`shared read`), écrits (`shared written`), etc.

```sql
EXPLAIN ANALYZE  
SELECT * FROM employes WHERE nom = 'Dupont';  

--                                   QUERY PLAN
-- ------------------------------------------------------------------------------
--  Index Scan using idx_employes_nom on employes  (cost=0.43..8.45 rows=1 width=120)
--                                                  (actual time=0.045..0.047 rows=1 loops=1)
--    Index Cond: (nom = 'Dupont'::text)
--    Buffers: shared hit=4
--  Planning Time: 0.123 ms
--  Execution Time: 0.072 ms
```

> 💡 Si vous voulez l'**ancien** comportement (sans BUFFERS), il faut désormais `EXPLAIN (ANALYZE, BUFFERS OFF)`.

---

### 12.5. Plus largement : autres nouveautés indexation

- **NOT NULL en contrainte de 1ʳᵉ classe** (PG 18) : déclarer une colonne `NOT NULL` permet désormais au planificateur de **simplifier** les expressions `IS NULL`/`COALESCE`/jointures, donc d'utiliser mieux les index. Ajouter `NOT NULL NOT VALID` sur une grosse table existante puis `VALIDATE CONSTRAINT` plus tard est désormais *zero-downtime*.
- **Skip Scan + Virtual Generated Columns** : on peut créer un index sur une expression (virtual ou stored). PG 18 permet d'indexer une `STORED` (et non `VIRTUAL`) — pour les `VIRTUAL`, l'index reste impossible.
- **Bénéfice indirect de l'AIO** : `CREATE INDEX CONCURRENTLY` et les `Bitmap Heap Scan` profitent du *prefetch* asynchrone (`io_method = worker | io_uring`).

---

## 13. Outils d'Audit d'Indexation

### 13.1. pg_stat_statements

**Extension essentielle** pour identifier les requêtes nécessitant des index.

#### Installation

```sql
-- 1. Modifier postgresql.conf
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
-- Redémarrer PostgreSQL

-- 2. Créer l'extension
CREATE EXTENSION pg_stat_statements;
```

#### Utilisation pour Audit d'Index

**Requêtes les plus lentes** :
```sql
SELECT
    substring(query, 1, 100) AS query_short,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements  
ORDER BY mean_exec_time DESC  
LIMIT 20;  
```

**Repérer les requêtes lisant beaucoup de blocs disque** (proxy pour *Seq Scan* probable) :

> ⚠️ `pg_stat_statements.query` contient le **texte SQL normalisé**, pas le **plan d'exécution** : un `LIKE '%seq scan%'` n'a aucun sens. Ce qu'on peut faire en revanche, c'est repérer les requêtes qui lisent beaucoup de blocs **shared** (cache + disque), corrélé avec la présence probable d'un *Seq Scan* coûteux.

```sql
SELECT
    substring(query, 1, 100)                                       AS query_short,
    calls,
    round(mean_exec_time::numeric, 2)                              AS avg_ms,
    shared_blks_hit,
    shared_blks_read,
    round(100.0 * shared_blks_read /
          NULLIF(shared_blks_hit + shared_blks_read, 0), 2)        AS miss_pct
FROM pg_stat_statements  
WHERE shared_blks_hit + shared_blks_read > 100000  
ORDER BY shared_blks_read DESC  
LIMIT 20;  
```

**Analyser ensuite avec EXPLAIN** sur les candidats identifiés :
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) <requête identifiée>;
```

> 💡 **PG 18** : la sortie d'`EXPLAIN (ANALYZE)` inclut **automatiquement** les compteurs `BUFFERS` (plus besoin de l'ajouter explicitement).

### 13.2. HypoPG

**Extension pour tester des index hypothétiques**.

#### Installation

```sql
CREATE EXTENSION hypopg;
```

#### Workflow de Test

```sql
-- 1. Créer un index hypothétique
SELECT hypopg_create_index('CREATE INDEX ON employes(nom)');

-- 2. Vérifier les index hypothétiques
SELECT * FROM hypopg();

-- 3. Tester une requête
EXPLAIN SELECT * FROM employes WHERE nom = 'Dupont';

-- 4. Si l'index hypothétique est utilisé et améliore le plan, créer l'index réel
CREATE INDEX CONCURRENTLY idx_employes_nom ON employes(nom);

-- 5. Nettoyer les index hypothétiques
SELECT hypopg_reset();
```

**Avantages** :
- ✅ Aucun impact sur la production  
- ✅ Test instantané  
- ✅ Validation avant création

### 13.3. pgstattuple

**Extension pour analyser le bloat**.

#### Installation

```sql
CREATE EXTENSION pgstattuple;
```

#### Analyse d'Index

```sql
-- Statistiques détaillées d'un index
SELECT * FROM pgstatindex('idx_employes_nom');

-- Colonnes importantes :
-- leaf_fragmentation : % de fragmentation
-- avg_leaf_density : Densité des pages (< 70% = bloat)
-- root_blkno : Bloc racine
```

**Interprétation** :
- `avg_leaf_density > 90%` : Excellent  
- `avg_leaf_density < 70%` : REINDEX recommandé

### 13.4. pg_indexes (Vue Système)

**Vue listant tous les index**.

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes  
WHERE schemaname = 'public'  
ORDER BY tablename, indexname;  
```

### 13.5. pg_stat_user_indexes (Vue Système)

**Vue avec statistiques d'utilisation des index**.

> ℹ️ Cette vue expose **`relname`** (table porteuse) et **`indexrelname`** (index). Les noms `tablename`/`indexname` n'apparaissent que dans la vue `pg_indexes` — confusion courante.

```sql
SELECT
    schemaname,
    relname       AS tablename,
    indexrelname  AS indexname,
    idx_scan,           -- Nombre de scans
    last_idx_scan,      -- 🆕 PG 16+ : timestamp du dernier scan
    idx_tup_read,       -- Tuples lus depuis l'index
    idx_tup_fetch       -- Tuples récupérés depuis la table
FROM pg_stat_user_indexes  
ORDER BY idx_scan DESC;  
```

### 13.6. Scripts SQL Utiles

#### Liste des Index Inutilisés

```sql
SELECT
    schemaname || '.' || relname AS table,
    indexrelname AS index,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    last_idx_scan                                  -- 🆕 PG 16+
FROM pg_stat_user_indexes  
WHERE idx_scan = 0  
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

#### Index les Plus Gros

```sql
SELECT
    schemaname || '.' || relname AS table,
    indexrelname AS index,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan AS scans
FROM pg_stat_user_indexes  
ORDER BY pg_relation_size(indexrelid) DESC  
LIMIT 20;  
```

#### Cache Hit Ratio par Index

```sql
SELECT
    relname       AS table,
    indexrelname  AS index,
    idx_blks_hit + idx_blks_read AS total_reads,
    CASE
        WHEN idx_blks_hit + idx_blks_read = 0 THEN NULL
        ELSE ROUND(100.0 * idx_blks_hit / (idx_blks_hit + idx_blks_read), 2)
    END AS cache_hit_ratio
FROM pg_statio_user_indexes  
WHERE idx_blks_hit + idx_blks_read > 0  
ORDER BY cache_hit_ratio ASC;  
```

#### Taille Totale des Index par Table

```sql
SELECT
    relname AS tablename,
    pg_size_pretty(SUM(pg_relation_size(indexrelid))) AS total_index_size,
    COUNT(*) AS index_count
FROM pg_stat_user_indexes  
GROUP BY schemaname, relname  
ORDER BY SUM(pg_relation_size(indexrelid)) DESC;  
```

---

## 14. Checklist d'Audit Complète

### 14.1. Phase 1 : Inventaire

- [ ] Lister tous les index existants (`pg_indexes`)  
- [ ] Mesurer la taille de chaque index (`pg_relation_size`)  
- [ ] Identifier le type de chaque index (B-Tree, GIN, etc.)  
- [ ] Documenter le but de chaque index

### 14.2. Phase 2 : Utilisation

- [ ] Identifier les index inutilisés (`idx_scan = 0`)  
- [ ] Mesurer la fréquence d'utilisation (`idx_scan`)  
- [ ] Calculer le ratio scans/taille  
- [ ] Identifier les index peu utilisés mais coûteux

### 14.3. Phase 3 : Redondance

- [ ] Détecter les index dupliqués (mêmes colonnes)  
- [ ] Détecter les redondances partielles (left-prefix)  
- [ ] Évaluer si les index redondants sont justifiés  
- [ ] Planifier la consolidation

### 14.4. Phase 4 : Bloat

- [ ] Mesurer le bloat de chaque index (`pgstatindex`)  
- [ ] Identifier les index avec bloat > 30%  
- [ ] Planifier REINDEX CONCURRENTLY pour index > 50% bloat  
- [ ] Vérifier la fréquence de VACUUM

### 14.5. Phase 5 : Performance

- [ ] Mesurer le cache hit ratio par index  
- [ ] Analyser les requêtes lentes (`pg_stat_statements`)  
- [ ] Utiliser EXPLAIN ANALYZE sur requêtes critiques  
- [ ] Identifier les sequential scans sur grandes tables

### 14.6. Phase 6 : Index Manquants

- [ ] Analyser les colonnes fréquentes dans WHERE  
- [ ] Vérifier les index sur clés étrangères  
- [ ] Identifier les colonnes de jointure sans index  
- [ ] Tester avec HypoPG les index candidats

### 14.7. Phase 7 : Stratégies Avancées

- [ ] Évaluer l'opportunité d'index partiels  
- [ ] Identifier les cas pour index sur expressions  
- [ ] Considérer INCLUDE pour covering index  
- [ ] Évaluer le besoin d'index spécialisés (GIN, BRIN)

### 14.8. Phase 8 : Documentation

- [ ] Documenter chaque index et sa justification  
- [ ] Créer un plan de maintenance (REINDEX schedule)  
- [ ] Définir les métriques de monitoring  
- [ ] Établir un calendrier d'audit (mensuel/trimestriel)

---

## 15. Erreurs Courantes d'Indexation

### 15.1. Créer des Index sur Tout

**Erreur** : Indexer toutes les colonnes "au cas où".

```sql
-- Mauvaise pratique
CREATE INDEX idx_employes_col1 ON employes(colonne1);  
CREATE INDEX idx_employes_col2 ON employes(colonne2);  
CREATE INDEX idx_employes_col3 ON employes(colonne3);  
-- ... etc pour 20 colonnes
```

**Problème** :
- Écritures très lentes (chaque INSERT/UPDATE doit maintenir tous les index)
- Espace disque gaspillé
- Maintenance lourde

**Solution** :
- Analyser les requêtes réelles
- Indexer uniquement les colonnes fréquemment filtrées/triées
- Commencer avec peu d'index, ajouter au besoin

### 15.2. Index sur Colonnes à Faible Sélectivité

**Erreur** : Indexer des colonnes booléennes ou avec peu de valeurs distinctes.

```sql
-- Peu utile
CREATE INDEX idx_users_actif ON users(actif);  -- Seulement true/false

-- Peu utile
CREATE INDEX idx_users_sexe ON users(sexe);    -- Seulement 'M'/'F'
```

**Problème** :
- Le planificateur préférera souvent un Seq Scan
- Index rarement utilisé
- Coût de maintenance pour peu de bénéfice

**Exception** : Index partiel si une valeur est très rare :
```sql
-- Justifié si peu d'utilisateurs inactifs
CREATE INDEX idx_users_inactifs ON users(actif) WHERE actif = false;
```

### 15.3. Oublier les Index sur Clés Étrangères

**Erreur** : Ne pas créer d'index sur les colonnes de clés étrangères.

```sql
-- Contrainte FK créée
ALTER TABLE commandes  
ADD CONSTRAINT fk_client  
FOREIGN KEY (client_id) REFERENCES clients(id);  

-- MAIS : Pas d'index sur commandes.client_id !
```

**Problème** :
- Jointures très lentes
- DELETE/UPDATE sur `clients` très lents (vérification d'intégrité)
- Locks excessifs

**Solution** :
```sql
CREATE INDEX idx_commandes_client_id ON commandes(client_id);
```

### 15.4. Index Multi-Colonnes dans le Mauvais Ordre

**Erreur** : Choisir l'ordre des colonnes au hasard sans tenir compte des requêtes ni du type de prédicat.

> ⚠️ La règle simpliste « *colonne la plus sélective d'abord* » est **partiellement fausse**. Ce qui compte réellement, c'est :  
>  
> 1. **L'ordre des prédicats** dans les requêtes (la colonne **de tête** doit être quasi systématiquement filtrée).  
> 2. Le **type d'opérateur** : les colonnes utilisées avec **égalité** (`=`, `IN`) doivent venir avant celles utilisées avec des **plages** (`<`, `>`, `BETWEEN`).  
> 3. La possibilité de **partager l'index** entre plusieurs requêtes (préfixe gauche réutilisable).

**Exemple concret** :

```sql
-- Requêtes principales :
--   (R1) WHERE statut = ?      AND created_at > ?
--   (R2) WHERE statut = ?
--   (R3) WHERE statut = ?      AND user_id = ?

-- ❌ Mauvais : created_at en tête fait perdre R2 et R3
CREATE INDEX bad ON commandes (created_at, statut, user_id);

-- ✅ Bon : statut (égalité, dans toutes les requêtes) en tête,
--          puis user_id (égalité, R3), puis created_at (plage, R1)
CREATE INDEX good ON commandes (statut, user_id, created_at);
```

**Règles pratiques** :

- La **colonne de tête** doit être utilisée dans **toutes** les requêtes qui veulent utiliser l'index (sauf avec **Skip Scan** PG 18 — qui aide mais ne résout pas tout).
- **Égalité avant plage** : `WHERE a = X AND b > Y` ⇒ index `(a, b)`, jamais `(b, a)`.
- Pour le **tri** : si l'`ORDER BY` est sur les mêmes colonnes que les filtres, l'index peut servir aux deux (économise un Sort).
- Avant PG 18, créer parfois un index supplémentaire `(b)` était nécessaire. Avec PG 18 + **Skip Scan**, c'est moins fréquent (cf. section 12.1).

### 15.5. Ne Jamais Faire de REINDEX

**Erreur** : Créer des index mais jamais les maintenir.

**Problème** :
- Bloat s'accumule progressivement
- Performances se dégradent
- Index finissent par être plus lents qu'un Seq Scan

**Solution** :
- Planifier REINDEX régulièrement (mensuel pour index critiques)
- Monitorer le bloat
- Automatiser avec pg_repack ou scripts

### 15.6. Utiliser REINDEX sans CONCURRENTLY en Production

**Erreur** : Utiliser `REINDEX INDEX nom_index` en production.

```sql
-- DANGER : Verrouille la table
REINDEX INDEX idx_employes_nom;
```

**Problème** :
- Table verrouillée en écriture pendant le REINDEX
- Application bloquée
- Indisponibilité

**Solution** :
```sql
-- Toujours utiliser CONCURRENTLY en production
REINDEX INDEX CONCURRENTLY idx_employes_nom;
```

### 15.7. Ignorer les Statistiques Obsolètes

**Erreur** : Créer des index mais ne jamais exécuter ANALYZE.

**Problème** :
- Le planificateur a des statistiques obsolètes
- Mauvais choix d'index
- Plans de requêtes sous-optimaux

**Solution** :
```sql
-- Après création d'index
ANALYZE employes;

-- Ou configurer autovacuum correctement
```

### 15.8. Index sur Fonctions sans les Utiliser dans les Requêtes

**Erreur** : Créer un index sur expression mais ne pas l'utiliser.

```sql
-- Index créé
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- Mais requête n'utilise pas lower()
SELECT * FROM users WHERE email = 'jean.dupont@example.com';
-- Index NON utilisé
```

**Solution** :
```sql
-- Requête doit utiliser la même expression
SELECT * FROM users WHERE lower(email) = lower('jean.dupont@example.com');
-- Index utilisé
```

---

## 16. Cas Pratiques et Scénarios

### 16.1. Scénario 1 : E-Commerce

#### Contexte
- Table `commandes` : 10 millions de lignes
- Table `produits` : 100,000 lignes
- Requêtes fréquentes : Recherche par client, par statut, par date

#### Stratégie d'Indexation

```sql
-- 1. Clé primaire (automatique)
-- commandes.id (PK, B-Tree)

-- 2. Clé étrangère client
CREATE INDEX idx_commandes_client ON commandes(client_id);

-- 3. Index composite pour recherche client + statut
CREATE INDEX idx_commandes_client_statut ON commandes(client_id, statut);

-- 4. Index partiel pour commandes en cours
CREATE INDEX idx_commandes_encours ON commandes(date_commande, montant)  
WHERE statut = 'en_cours';  

-- 5. Index BRIN sur date (données séquentielles)
CREATE INDEX idx_commandes_date_brin ON commandes USING brin(date_commande);

-- 6. Produits : recherche texte
CREATE INDEX idx_produits_nom_gin ON produits USING gin(to_tsvector('french', nom));
```

#### Requêtes Optimisées

```sql
-- Utilise idx_commandes_client_statut
SELECT * FROM commandes  
WHERE client_id = 123 AND statut = 'livré';  

-- Utilise idx_commandes_encours (index partiel)
SELECT * FROM commandes  
WHERE statut = 'en_cours' AND date_commande > CURRENT_DATE - INTERVAL '7 days';  

-- Utilise idx_commandes_date_brin
SELECT COUNT(*) FROM commandes  
WHERE date_commande BETWEEN '2025-01-01' AND '2025-01-31';  
```

### 16.2. Scénario 2 : Application SaaS Multi-tenant

#### Contexte
- Table `utilisateurs` : 5 millions de lignes
- Plusieurs tenants (organisations)
- Recherche fréquente par tenant + email

#### Stratégie d'Indexation

```sql
-- 1. Index composite tenant + email (ordre important)
CREATE INDEX idx_users_tenant_email ON utilisateurs(tenant_id, email);

-- 2. Index unique pour éviter doublons email par tenant
CREATE UNIQUE INDEX idx_users_tenant_email_unique  
ON utilisateurs(tenant_id, lower(email));  

-- 3. Index partiel pour utilisateurs actifs
CREATE INDEX idx_users_actifs ON utilisateurs(tenant_id, last_login)  
WHERE actif = true;  

-- 4. Index GIN sur permissions JSONB
CREATE INDEX idx_users_permissions ON utilisateurs USING gin(permissions);
```

#### Requêtes Optimisées

```sql
-- Utilise idx_users_tenant_email
SELECT * FROM utilisateurs  
WHERE tenant_id = 42 AND email = 'user@example.com';  

-- Utilise idx_users_actifs
SELECT * FROM utilisateurs  
WHERE tenant_id = 42 AND actif = true  
ORDER BY last_login DESC;  

-- Utilise idx_users_permissions
SELECT * FROM utilisateurs  
WHERE permissions @> '{"role": "admin"}';  
```

### 16.3. Scénario 3 : Logs et Séries Temporelles

#### Contexte
- Table `logs` : 1 milliard de lignes
- Croissance continue (10 GB/jour)
- Requêtes principalement sur timestamp récent

#### Stratégie d'Indexation

```sql
-- 1. Index BRIN sur timestamp (très efficace pour grandes tables)
CREATE INDEX idx_logs_timestamp_brin ON logs USING brin(timestamp)  
WITH (pages_per_range = 128);  

-- 2. Index partiel sur derniers 30 jours (hot data)
CREATE INDEX idx_logs_recent ON logs(timestamp, level)  
WHERE timestamp > CURRENT_DATE - INTERVAL '30 days';  

-- 3. Index GIN sur message pour recherche texte
CREATE INDEX idx_logs_message_gin ON logs USING gin(to_tsvector('english', message));

-- 4. Considérer TimescaleDB pour meilleure gestion
-- (extension spécialisée séries temporelles)
```

#### Requêtes Optimisées

```sql
-- Utilise idx_logs_recent (index partiel)
SELECT * FROM logs  
WHERE timestamp > CURRENT_DATE - INTERVAL '1 day'  
  AND level = 'ERROR';

-- Utilise idx_logs_timestamp_brin
SELECT COUNT(*) FROM logs  
WHERE timestamp BETWEEN '2025-01-01' AND '2025-02-01';  
```

### 16.4. Scénario 4 : Recherche Géospatiale

#### Contexte
- Table `lieux` : 1 million de points géographiques
- Requêtes de proximité fréquentes

#### Stratégie d'Indexation (avec PostGIS)

```sql
-- Extension PostGIS
CREATE EXTENSION postgis;

-- 1. Index GiST sur géométrie
CREATE INDEX idx_lieux_geom ON lieux USING gist(geom);

-- 2. Index composite pour filtrage par type + proximité
CREATE INDEX idx_lieux_type_geom ON lieux USING gist(type, geom);

-- 3. Index BRIN si données géographiquement ordonnées
-- (ex: données par ville triées alphabétiquement)
CREATE INDEX idx_lieux_ville_brin ON lieux USING brin(ville);
```

#### Requêtes Optimisées

```sql
-- Utilise idx_lieux_geom
SELECT * FROM lieux  
WHERE ST_DWithin(geom, ST_MakePoint(2.3522, 48.8566), 1000);  

-- Recherche restaurants dans un rayon
SELECT * FROM lieux  
WHERE type = 'restaurant'  
  AND ST_DWithin(geom, ST_MakePoint(2.3522, 48.8566), 1000);
```

---

## 17. Conclusion et Bonnes Pratiques

### 17.1. Les 10 Règles d'Or de l'Indexation

#### 1. Index Parcimonieusement
**Règle** : Créez uniquement les index nécessaires.

**Pourquoi** : Chaque index a un coût (écritures, espace, maintenance).

**Application** : Commencez avec peu d'index, ajoutez selon les besoins réels identifiés par monitoring.

#### 2. Indexez les Clés Étrangères
**Règle** : Créez toujours un index sur les colonnes de FK.

**Pourquoi** : Accélère les jointures et les opérations d'intégrité référentielle.

**Application** : Après chaque création de FK, créez l'index correspondant.

#### 3. Ordre des Colonnes Compte
**Règle** : Dans un index multi-colonnes, placez les colonnes les plus sélectives en premier.

**Application** : Index sur `(nom, prenom)` plutôt que `(prenom, nom)` si `nom` est plus sélectif.

#### 4. Privilégiez les Index Partiels
**Règle** : Pour les requêtes ciblant un sous-ensemble, utilisez des index partiels.

**Avantage** : Plus petits, plus rapides, moins coûteux en maintenance.

**Application** : Index partiel sur `statut = 'actif'` plutôt qu'index complet.

#### 5. Utilisez CONCURRENTLY en Production
**Règle** : Créez et réindexez toujours avec `CONCURRENTLY`.

**Pourquoi** : Évite les verrouillages qui bloqueraient l'application.

**Application** :
```sql
CREATE INDEX CONCURRENTLY ...  
REINDEX INDEX CONCURRENTLY ...  
```

#### 6. Maintenez Vos Index
**Règle** : Planifiez des REINDEX réguliers pour les index critiques.

**Fréquence** : Mensuel pour index avec bloat > 30%.

**Application** : Automatisez avec pg_repack ou scripts planifiés.

#### 7. Surveillez l'Utilisation
**Règle** : Auditez régulièrement les index inutilisés.

**Pourquoi** : Supprimer les index inutilisés améliore les écritures.

**Application** : Requête mensuelle sur `pg_stat_user_indexes` pour identifier `idx_scan = 0`.

#### 8. ANALYZE Après Création
**Règle** : Exécutez toujours ANALYZE après création d'index.

**Pourquoi** : Met à jour les statistiques pour que le planificateur utilise l'index efficacement.

**Application** :
```sql
CREATE INDEX ...  
ANALYZE table_name;  
```

#### 9. Testez Avant de Créer
**Règle** : Utilisez HypoPG ou EXPLAIN pour valider l'utilité d'un index.

**Pourquoi** : Évite de créer des index inutiles.

**Application** : Testez avec index hypothétique, créez seulement si amélioration confirmée.

#### 10. Documentez Vos Décisions
**Règle** : Documentez le but et la justification de chaque index.

**Pourquoi** : Facilite les audits futurs et évite les suppressions accidentelles.

**Application** : Commentaire dans le code ou fichier de documentation séparé.

### 17.2. Checklist Récapitulative Rapide

**Avant de créer un index** :
- [ ] La colonne est-elle fréquemment filtrée/triée ?  
- [ ] La table est-elle suffisamment grande (> 10,000 lignes) ?  
- [ ] La colonne a-t-elle une sélectivité suffisante ?  
- [ ] Un index existant ne couvre-t-il pas déjà ce besoin ?  
- [ ] Ai-je testé avec EXPLAIN ou HypoPG ?

**Audit mensuel** :
- [ ] Identifier les index inutilisés (`idx_scan = 0`)  
- [ ] Mesurer le bloat des index  
- [ ] Analyser les requêtes lentes  
- [ ] Chercher les redondances  
- [ ] Vérifier le cache hit ratio

**Maintenance trimestrielle** :
- [ ] REINDEX des index avec bloat > 30%  
- [ ] Supprimer les index inutilisés confirmés  
- [ ] Consolider les index redondants  
- [ ] Réviser la stratégie d'indexation selon l'évolution de la charge

### 17.3. Récapitulatif des Types d'Index

| Type | Quand l'utiliser | Ne PAS utiliser si |
|------|------------------|--------------------|
| **B-Tree** | Cas général, comparaisons, tri | Recherche texte intégral, géométrie |
| **Hash** | Presque jamais (B-Tree meilleur) | Tout sauf égalité stricte |
| **GIN** | JSONB, arrays, full-text | Écritures très intensives |
| **GiST** | Géométrie, full-text secondaire | Besoin de performance maximale |
| **BRIN** | Très grandes tables séquentielles | Données non ordonnées physiquement |
| **SP-GiST** | IP, structures spécifiques | Cas général |

### 17.4. Formules Utiles

**Sélectivité** :
```
Sélectivité = Nombre de valeurs distinctes / Nombre total de lignes
```

**Coût d'un index** :
```
Coût = (Taille × Maintenance) + (Écritures × Overhead)
```

**Bénéfice d'un index** :
```
Bénéfice = Gain en temps de lecture × Fréquence des lectures
```

**ROI (Return On Investment)** :
```
ROI = Bénéfice / Coût
```

Créez l'index seulement si `ROI > 1`.

### 17.5. PostgreSQL 18 : Points Clés

Les nouveautés PostgreSQL 18 impactent l'indexation :

1. **Skip Scan** : Réduit le besoin d'index redondants  
2. **Optimisation OR** : Transformation automatique OR → ANY  
3. **Auto-élimination self-joins** : Meilleure utilisation des index existants  
4. **EXPLAIN amélioré** : Meilleure observabilité

**Action** : Réévaluez vos index redondants avec PostgreSQL 18, certains peuvent devenir inutiles grâce à Skip Scan.

### 17.6. Ressources Complémentaires

#### Documentation
- **PostgreSQL 18 Indexes** : https://www.postgresql.org/docs/18/indexes.html  
- **EXPLAIN** : https://www.postgresql.org/docs/18/using-explain.html

#### Extensions
- **HypoPG** : https://github.com/HypoPG/hypopg  
- **pg_repack** : https://github.com/reorg/pg_repack  
- **pgstattuple** : https://www.postgresql.org/docs/18/pgstattuple.html

#### Livres
- *PostgreSQL Query Optimization* - plusieurs auteurs  
- *The Art of PostgreSQL* par Dimitri Fontaine

#### Outils
- **pgAdmin** : Interface graphique avec visualisation des index  
- **DBeaver** : IDE avec analyse de performance  
- **Postgres.ai** : Plateforme d'optimisation de requêtes

---

## Conclusion Finale

L'audit d'indexation est un processus **continu et itératif**. Une stratégie d'indexation optimale évolue avec :
- La croissance des données
- L'évolution des patterns d'accès
- Les nouvelles fonctionnalités applicatives
- Les mises à jour PostgreSQL

**Les index sont un outil puissant** mais ils doivent être utilisés avec **discernement**. Trop d'index est aussi néfaste que pas assez.

**Approche recommandée** :
1. **Commencez simple** : Index sur PK, FK, et colonnes fréquemment filtrées  
2. **Mesurez** : Utilisez pg_stat_statements et EXPLAIN  
3. **Ajustez** : Ajoutez/supprimez selon les besoins réels  
4. **Maintenez** : REINDEX régulièrement  
5. **Documentez** : Justifiez chaque index

Avec PostgreSQL 18, le planificateur est plus intelligent que jamais (Skip Scan, optimisations OR, etc.). **Laissez PostgreSQL faire son travail** et concentrez-vous sur les index vraiment stratégiques.

**Dernière recommandation** : **Mesurez toujours** avant et après. Un index est utile seulement s'il améliore **réellement** les performances dans **votre contexte spécifique**.

---


⏭️ [Audit de requêtes](/annexes/checklist-performance/03-audit-requetes.md)
