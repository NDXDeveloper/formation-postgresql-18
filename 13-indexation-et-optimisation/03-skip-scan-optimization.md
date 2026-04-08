🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.3. Nouveauté PG 18 : Skip Scan Optimization pour Index Multi-Colonnes

## Introduction

PostgreSQL 18 (septembre 2025) introduit une **amélioration majeure** dans l'utilisation des index composites : le **Skip Scan** (scan par saut). Cette optimisation permet au planificateur d'utiliser efficacement des index multi-colonnes même quand la **première colonne n'apparaît pas dans les conditions WHERE**.

**Impact** : Moins d'index redondants nécessaires, meilleures performances, gestion simplifiée.

**Prérequis** : Avoir lu les sections 13.1 (Stratégies de scan) et 13.2 (Index B-Tree).

---

## 1. Le Problème Avant PostgreSQL 18

### 1.1. La Règle du Leftmost Prefix

Dans PostgreSQL ≤ 17, un index composite ne pouvait être utilisé que si la **première colonne** (leftmost) apparaissait dans le WHERE.

#### Exemple

```sql
-- Création d'un index composite
CREATE INDEX idx_orders_status_date ON orders(status, created_at);
```

**Structure de l'index** (rappel section 13.2) :
```
Index trié par : status PUIS created_at

Feuilles :
['cancelled', '2024-01-15'] → TID1
['cancelled', '2024-02-20'] → TID2
['pending',   '2024-01-10'] → TID3
['pending',   '2024-03-05'] → TID4
['shipped',   '2024-01-25'] → TID5
['shipped',   '2024-02-15'] → TID6
```

### 1.2. Requêtes Utilisables (Avant PG 18)

✅ **Requête 1** : Première colonne présente
```sql
SELECT * FROM orders WHERE status = 'pending';
```
→ **Index utilisé** : PostgreSQL peut chercher directement toutes les lignes avec `status = 'pending'`.

✅ **Requête 2** : Première ET deuxième colonnes présentes
```sql
SELECT * FROM orders  
WHERE status = 'pending' AND created_at > '2024-01-01';  
```
→ **Index utilisé** : Accès optimal via l'index composite.

### 1.3. Requêtes NON Utilisables (Avant PG 18)

❌ **Requête 3** : Seulement la deuxième colonne
```sql
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Problème** : La première colonne `status` n'est pas dans le WHERE.

**Comportement PostgreSQL ≤ 17** :
```
Seq Scan on orders  (cost=0.00..1834.00 rows=50000 width=50)
  Filter: (created_at > '2024-01-01')
```

→ **Sequential Scan** (lecture complète de la table) au lieu d'utiliser l'index !

### 1.4. La Solution "Classique" : Index Redondants

Pour couvrir tous les cas, il fallait créer **plusieurs index** :

```sql
-- Index 1 : Pour status seul ou status+date
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- Index 2 : Pour date seule (redondant !)
CREATE INDEX idx_orders_date ON orders(created_at);
```

**Inconvénients** :
- 🔴 **Double maintenance** : Chaque écriture met à jour 2 index  
- 🔴 **Double espace disque** : La colonne `created_at` est stockée 2 fois  
- 🔴 **Complexité** : Plus d'index à gérer et surveiller

**Analogie** : C'est comme avoir deux annuaires téléphoniques différents pour la même ville, un trié par Nom+Prénom et un autre trié seulement par Date de Naissance, au lieu d'avoir un seul annuaire intelligent.

---

## 2. La Solution : Skip Scan Optimization

### 2.1. Concept

Le **Skip Scan** permet à PostgreSQL de "sauter" (skip) les valeurs de la première colonne de l'index pour accéder directement à la deuxième colonne.

**Analogie** : Imaginez un annuaire téléphonique trié par Ville puis Nom. Avant le Skip Scan, pour trouver tous les "Dupont" sans spécifier la ville, il fallait lire l'annuaire entièrement. Avec le Skip Scan, PostgreSQL peut "sauter" de ville en ville et chercher uniquement les "Dupont" dans chaque section.

### 2.2. Fonctionnement

PostgreSQL parcourt les **valeurs distinctes** de la première colonne, puis pour chaque valeur, il effectue une recherche sur la deuxième colonne.

**Visualisation** :

```
Index : (status, created_at)

Valeurs distinctes de status : ['cancelled', 'pending', 'shipped']

Skip Scan pour : WHERE created_at > '2024-01-01'

Étape 1 : Chercher dans status='cancelled' + created_at > '2024-01-01'
          ↓
['cancelled', '2024-01-15'] → TID1 ✓
['cancelled', '2024-02-20'] → TID2 ✓

Étape 2 : SAUTER vers status='pending' + created_at > '2024-01-01'
          ↓
['pending',   '2024-01-10'] → TID3 ✓
['pending',   '2024-03-05'] → TID4 ✓

Étape 3 : SAUTER vers status='shipped' + created_at > '2024-01-01'
          ↓
['shipped',   '2024-01-25'] → TID5 ✓
['shipped',   '2024-02-15'] → TID6 ✓
```

**Résultat** : Toutes les lignes avec `created_at > '2024-01-01'` sont trouvées sans Seq Scan !

---

## 3. Algorithme du Skip Scan

### 3.1. Étapes Détaillées

**Requête** :
```sql
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Index disponible** : `(status, created_at)`

#### Algorithme PostgreSQL 18

**Étape 1** : Identifier les valeurs distinctes de la première colonne
```sql
-- PostgreSQL génère implicitement
SELECT DISTINCT status FROM orders;
-- Résultat : ['cancelled', 'pending', 'shipped']
```

**Étape 2** : Pour chaque valeur distincte, chercher dans l'index
```sql
-- Boucle interne équivalente à :
FOR EACH status_value IN ['cancelled', 'pending', 'shipped'] LOOP
    -- Chercher dans l'index (status_value, created_at > '2024-01-01')
    Index Scan on idx_orders_status_date
      WHERE status = status_value AND created_at > '2024-01-01';
END LOOP;
```

**Étape 3** : Fusionner les résultats

### 3.2. Complexité

**Coût du Skip Scan** = (Nombre de valeurs distinctes × Coût d'un Index Scan)

**Comparaison** :

| Méthode | Coût | Cas favorable |
|---------|------|---------------|
| **Seq Scan** | O(n) | Jamais pour grandes tables |
| **Skip Scan** | O(d × log n) | d petit (peu de valeurs distinctes) |
| **Index direct** | O(log n) | Première colonne dans WHERE |

Où :
- n = Nombre total de lignes
- d = Nombre de valeurs distinctes de la première colonne

**Conclusion** : Skip Scan est efficace si `d` (cardinalité de la première colonne) est **faible**.

---

## 4. Quand PostgreSQL 18 Utilise-t-il le Skip Scan ?

### 4.1. Conditions Nécessaires

Pour que PostgreSQL active le Skip Scan, **toutes** les conditions suivantes doivent être remplies :

1. ✅ **Index composite disponible** : Au moins 2 colonnes  
2. ✅ **Première colonne absente du WHERE** : Seulement la 2ème, 3ème, etc.  
3. ✅ **Cardinalité faible de la 1ère colonne** : Peu de valeurs distinctes (généralement < 100-1000)  
4. ✅ **Sélectivité élevée de la 2ème colonne** : Filtre efficace (< 10% des lignes)  
5. ✅ **Statistiques à jour** : `ANALYZE` exécuté récemment

### 4.2. Exemples de Cas Favorables

#### Exemple 1 : Statut + Date

```sql
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- Requête
SELECT * FROM orders WHERE created_at > '2024-11-01';
```

**Analyse** :
- `status` : 3-5 valeurs distinctes (pending, shipped, cancelled, returned, failed)  
- `created_at` : Très sélectif (ex: dernières 24h = 0.1% des données)

→ ✅ **Skip Scan activé** (PostgreSQL 18)

#### Exemple 2 : Catégorie + Prix

```sql
CREATE INDEX idx_products_category_price ON products(category, price);

-- Requête
SELECT * FROM products WHERE price > 1000;
```

**Analyse** :
- `category` : 10-20 catégories (Electronics, Books, Clothing, etc.)  
- `price > 1000` : Sélectif (ex: 5% des produits)

→ ✅ **Skip Scan activé**

### 4.3. Exemples de Cas Défavorables

#### Exemple 1 : User ID + Timestamp

```sql
CREATE INDEX idx_events_user_timestamp ON events(user_id, timestamp);

-- Requête
SELECT * FROM events WHERE timestamp > '2024-11-01';
```

**Analyse** :
- `user_id` : 1 million de valeurs distinctes (cardinalité élevée)
- Skip Scan = 1M recherches dans l'index

→ ❌ **Skip Scan NON activé** → Seq Scan plus rapide

#### Exemple 2 : Email + Created

```sql
CREATE INDEX idx_users_email_created ON users(email, created_at);

-- Requête (peu sélective)
SELECT * FROM users WHERE created_at > '2020-01-01';
```

**Analyse** :
- `email` : Millions de valeurs distinctes (très élevé)  
- `created_at > '2020-01-01'` : 99% des lignes (peu sélectif)

→ ❌ **Skip Scan NON activé** → Seq Scan plus approprié

---

## 5. Comparaison Avant/Après PostgreSQL 18

### 5.1. Scénario : Table E-Commerce

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT,
    status VARCHAR(20),  -- 4 valeurs : pending, shipped, cancelled, returned
    total_amount NUMERIC,
    created_at TIMESTAMP
);

-- 10 millions de commandes
-- Distribution :
--   pending: 10%
--   shipped: 70%
--   cancelled: 15%
--   returned: 5%

CREATE INDEX idx_orders_status_date ON orders(status, created_at);
```

### 5.2. Requête Problématique

```sql
-- Trouver les commandes des 7 derniers jours
SELECT * FROM orders  
WHERE created_at > NOW() - INTERVAL '7 days';  
```

### 5.3. PostgreSQL ≤ 17 : Seq Scan

**Plan d'exécution** :
```
Seq Scan on orders  (cost=0.00..250000.00 rows=50000 width=100)
  Filter: (created_at > (now() - '7 days'::interval))
  Rows Removed by Filter: 9950000
```

**Analyse** :
- Scan de 10M de lignes
- 50 000 lignes retournées (0.5%)
- Temps : ~8 secondes (disque rotatif) ou ~2 secondes (SSD)

**Solution classique** : Créer un index redondant
```sql
CREATE INDEX idx_orders_date ON orders(created_at);  -- Index supplémentaire !
```

### 5.4. PostgreSQL 18 : Skip Scan

**Plan d'exécution** :
```
Index Scan using idx_orders_status_date on orders
  (cost=0.56..15234.00 rows=50000 width=100)
  Index Cond: (created_at > (now() - '7 days'::interval))
  Skip Scan: status
  Loops: 4
```

**Analyse** :
- Skip Scan sur 4 valeurs de `status`
- Accès direct via l'index
- Temps : ~200 ms (SSD)

→ **Gain de 10-40× en performance** ! 🚀

### 5.5. Tableau Comparatif

| Critère | PostgreSQL ≤ 17 | PostgreSQL 18 |
|---------|-----------------|---------------|
| **Stratégie** | Seq Scan | Skip Scan |
| **Lignes scannées** | 10 000 000 | ~200 000 (via index) |
| **Temps (SSD)** | ~2 secondes | ~200 ms |
| **Index nécessaires** | 2 (redondants) | 1 (optimisé) |
| **Maintenance** | Double | Simple |
| **Espace disque** | 2× colonnes indexées | 1× colonnes indexées |

---

## 6. Configuration et Contrôle

### 6.1. Activation par Défaut

Le Skip Scan est **activé par défaut** dans PostgreSQL 18.

Vérification :
```sql
SHOW enable_indexskipscan;
-- Résultat : on
```

### 6.2. Désactivation (Test/Debug)

Pour forcer un Seq Scan (comparaison de performances) :

```sql
-- Désactiver temporairement
SET enable_indexskipscan = off;

EXPLAIN ANALYZE  
SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days';  

-- Réactiver
SET enable_indexskipscan = on;
```

### 6.3. Forcer le Skip Scan (Rarement Nécessaire)

Si PostgreSQL ne choisit pas le Skip Scan alors qu'il devrait :

```sql
-- Vérifier les statistiques
ANALYZE orders;

-- Ajuster le coût du Seq Scan (rendre le Skip Scan plus attractif)
SET seq_page_cost = 1.5;  -- Valeur par défaut : 1.0  
SET random_page_cost = 3.0;  -- Valeur par défaut : 4.0  
```

---

## 7. Cas d'Usage Avancés

### 7.1. Index à 3 Colonnes

```sql
CREATE INDEX idx_orders_triple ON orders(status, payment_method, created_at);
```

**Requêtes bénéficiant du Skip Scan** :

✅ Seulement la 3ème colonne
```sql
SELECT * FROM orders WHERE created_at > '2024-11-01';
-- Skip sur status ET payment_method
```

✅ 2ème et 3ème colonnes
```sql
SELECT * FROM orders  
WHERE payment_method = 'credit_card' AND created_at > '2024-11-01';  
-- Skip seulement sur status
```

❌ Seulement la 2ème colonne (Skip Scan peu probable)
```sql
SELECT * FROM orders WHERE payment_method = 'credit_card';
-- Skip sur status, mais moins efficace qu'un index direct
```

### 7.2. Combinaison avec d'Autres Optimisations

#### Index Partial + Skip Scan

```sql
-- Index partiel sur commandes récentes
CREATE INDEX idx_orders_recent ON orders(status, created_at)  
WHERE created_at > '2024-01-01';  

-- Requête sur période récente
SELECT * FROM orders  
WHERE created_at > '2024-11-01';  
```

→ Skip Scan sur un index plus petit = encore plus rapide !

#### Covering Index + Skip Scan

```sql
CREATE INDEX idx_orders_covering ON orders(status, created_at)  
INCLUDE (total_amount);  

-- Index-Only Scan possible avec Skip Scan
SELECT total_amount  
FROM orders  
WHERE created_at > '2024-11-01';  
```

→ **Combinaison ultime** : Skip Scan + Index-Only Scan = performance maximale !

---

## 8. Impact sur la Conception des Index

### 8.1. Avant PostgreSQL 18 : Stratégie Défensive

**Approche** : Créer plusieurs index pour couvrir tous les cas

```sql
-- Index 1 : Status + Date
CREATE INDEX idx1 ON orders(status, created_at);

-- Index 2 : Date seule (redondant)
CREATE INDEX idx2 ON orders(created_at);

-- Index 3 : Customer + Date
CREATE INDEX idx3 ON orders(customer_id, created_at);

-- Index 4 : Date seule (encore !) pour une autre requête
-- → Confusion et maintenance complexe
```

**Problèmes** :
- 🔴 Redondance massive  
- 🔴 Maintenance lourde (4 index à maintenir)  
- 🔴 Espace disque × 4  
- 🔴 Écritures ralenties

### 8.2. Avec PostgreSQL 18 : Stratégie Optimisée

**Approche** : Un seul index composite bien conçu

```sql
-- Index unique couvrant plusieurs cas
CREATE INDEX idx_orders_optimized ON orders(status, created_at);
```

**Couverture** :
- ✅ `WHERE status = '...'`  
- ✅ `WHERE status = '...' AND created_at > '...'`  
- ✅ `WHERE created_at > '...'` → **Skip Scan !**

**Avantages** :
- ✅ Un seul index à maintenir  
- ✅ Espace disque divisé par 2-3  
- ✅ Écritures plus rapides  
- ✅ Gestion simplifiée

### 8.3. Nouvelle Règle de Conception

**Avant PG 18** : "Première colonne = colonne la plus utilisée"

**Avec PG 18** : "Première colonne = colonne avec **faible cardinalité**"

**Exemple** :

Table `events` avec :
- `user_id` : 1M valeurs distinctes  
- `event_type` : 10 valeurs distinctes  
- `timestamp` : Continu

**Avant PG 18** :
```sql
-- user_id en premier (plus utilisé)
CREATE INDEX idx_events ON events(user_id, timestamp);

-- Index séparé nécessaire pour timestamp seul
CREATE INDEX idx_events_ts ON events(timestamp);
```

**Avec PG 18** :
```sql
-- event_type en premier (faible cardinalité → Skip Scan efficace)
CREATE INDEX idx_events ON events(event_type, timestamp);

-- Couvre :
-- - WHERE event_type = '...'
-- - WHERE event_type = '...' AND timestamp > '...'
-- - WHERE timestamp > '...' → Skip Scan sur 10 valeurs seulement !
```

---

## 9. Monitoring et Analyse

### 9.1. Détecter l'Utilisation du Skip Scan

#### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT * FROM orders  
WHERE created_at > NOW() - INTERVAL '7 days';  
```

**Plan avec Skip Scan** :
```
Index Scan using idx_orders_status_date on orders
  (cost=0.56..15234.00 rows=50000 width=100) (actual time=0.123..195.456 rows=50123 loops=1)
  Index Cond: (created_at > (now() - '7 days'::interval))
  Skip Scan: status
  Loops: 4
  Buffers: shared hit=1234
Planning Time: 0.234 ms  
Execution Time: 197.891 ms  
```

**Indicateurs** :
- `Skip Scan: status` → Confirmation du Skip Scan  
- `Loops: 4` → 4 valeurs distinctes de `status` parcourues  
- `Buffers: shared hit` → Lecture depuis le cache (performant)

#### Comparaison avec Seq Scan

```sql
-- Forcer Seq Scan pour comparaison
SET enable_indexskipscan = off;

EXPLAIN (ANALYZE, BUFFERS)  
SELECT * FROM orders  
WHERE created_at > NOW() - INTERVAL '7 days';  
```

**Plan sans Skip Scan** :
```
Seq Scan on orders
  (cost=0.00..250000.00 rows=50000 width=100) (actual time=0.012..2145.678 rows=50123 loops=1)
  Filter: (created_at > (now() - '7 days'::interval))
  Rows Removed by Filter: 9949877
  Buffers: shared hit=125000
Planning Time: 0.087 ms  
Execution Time: 2147.234 ms  
```

**Comparaison** :
- Skip Scan : **197 ms**
- Seq Scan : **2147 ms**
- **Gain : 10.8×** ! 🚀

### 9.2. Statistiques des Index

Vérifier la cardinalité de la première colonne :

```sql
SELECT
    tablename,
    attname AS column_name,
    n_distinct,
    null_frac,
    avg_width
FROM pg_stats  
WHERE tablename = 'orders' AND attname = 'status';  
```

**Résultat** :
```
 tablename | column_name | n_distinct | null_frac | avg_width
-----------|-------------|------------|-----------|----------
 orders    | status      | 4          | 0         | 9
```

**Interprétation** :
- `n_distinct = 4` → Très faible (Skip Scan efficace)  
- `null_frac = 0` → Pas de NULL  
- `avg_width = 9` → Petite taille

→ ✅ Candidat idéal pour Skip Scan !

---

## 10. Limitations et Pièges

### 10.1. Cardinalité Élevée

Si la première colonne a trop de valeurs distinctes, Skip Scan devient **contre-productif**.

**Exemple** :
```sql
CREATE INDEX idx_users_email_created ON users(email, created_at);

-- 1 million d'emails distincts
SELECT * FROM users WHERE created_at > '2024-11-01';
```

→ Skip Scan sur 1M valeurs = 1M recherches = **plus lent qu'un Seq Scan !**

**Solution** : Créer un index direct sur `created_at` ou réorganiser l'index avec une colonne à faible cardinalité en premier.

### 10.2. Requêtes avec OR

Skip Scan fonctionne moins bien avec des conditions OR complexes :

```sql
SELECT * FROM orders  
WHERE created_at > '2024-11-01' OR status = 'pending';  
```

→ PostgreSQL peut préférer un Bitmap Scan ou Seq Scan.

### 10.3. Statistiques Obsolètes

Si `ANALYZE` n'a pas été exécuté récemment, PostgreSQL peut sous-estimer la cardinalité de la première colonne et choisir un Seq Scan.

**Solution** :
```sql
ANALYZE orders;
```

### 10.4. Coût du Planificateur

Le calcul du Skip Scan a un léger surcoût au niveau du planificateur (quelques microsecondes).

**Impact** : Négligeable pour des requêtes d'exécution > 10ms, mais peut se remarquer pour des requêtes ultra-rapides (<1ms).

---

## 11. Migration depuis PostgreSQL ≤ 17

### 11.1. Audit des Index Existants

Identifier les index redondants qui peuvent être supprimés :

```sql
-- Trouver les index avec chevauchement potentiel
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes  
WHERE schemaname = 'public'  
ORDER BY tablename, indexname;  
```

**Rechercher** :
- Index composites `(col1, col2)`
- Index simples `(col2)` sur la même table

→ Si `col1` a une faible cardinalité, l'index simple peut être **supprimé** après migration vers PG 18.

### 11.2. Tests de Performance

**Avant suppression** :

```sql
-- 1. Activer timing
\timing on

-- 2. Tester avec index actuel
EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at > '2024-11-01';

-- 3. Simuler la suppression (désactiver temporairement)
UPDATE pg_index  
SET indisvalid = false  
WHERE indexrelid = 'idx_orders_date'::regclass;  

-- 4. Re-tester
EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at > '2024-11-01';

-- 5. Réactiver si nécessaire
UPDATE pg_index  
SET indisvalid = true  
WHERE indexrelid = 'idx_orders_date'::regclass;  
```

**Si performances équivalentes ou meilleures** → Supprimez l'index redondant !

### 11.3. Plan de Migration

**Étape 1** : Upgrade vers PostgreSQL 18
```bash
pg_upgrade --check  
pg_upgrade --link  
```

**Étape 2** : Analyser toutes les tables
```sql
VACUUM ANALYZE;
```

**Étape 3** : Tester les requêtes critiques
```sql
-- Vérifier que Skip Scan est activé
EXPLAIN SELECT * FROM orders WHERE created_at > '2024-11-01';
```

**Étape 4** : Supprimer les index redondants progressivement
```sql
-- Supprimer avec précaution (1 par 1, en surveillant)
DROP INDEX CONCURRENTLY idx_orders_date;
```

**Étape 5** : Monitoring post-migration
- Surveiller les temps de réponse
- Vérifier les plans d'exécution
- Analyser les métriques via `pg_stat_statements`

---

## 12. Exemples Complets

### 12.1. E-Commerce : Commandes

```sql
-- Table avec 10M de commandes
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    status VARCHAR(20) NOT NULL,  -- 4 valeurs
    payment_method VARCHAR(20),   -- 5 valeurs
    total_amount NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Index composite optimisé pour PG 18
CREATE INDEX idx_orders_opt ON orders(status, payment_method, created_at);

-- Requête 1 : Commandes récentes (Skip sur status + payment_method)
SELECT * FROM orders WHERE created_at > '2024-11-15';

-- Requête 2 : Par statut + date (Index direct)
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-11-01';

-- Requête 3 : Par méthode de paiement + date (Skip sur status)
SELECT * FROM orders  
WHERE payment_method = 'credit_card' AND created_at > '2024-11-01';  
```

**Résultat** : Un seul index couvre les 3 requêtes ! ✅

### 12.2. IoT : Événements Capteurs

```sql
-- Table avec 100M d'événements
CREATE TABLE sensor_events (
    id BIGSERIAL PRIMARY KEY,
    sensor_type VARCHAR(20) NOT NULL,  -- 8 types
    location_id INT NOT NULL,          -- 50 localisations
    value NUMERIC,
    timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- Index composite IoT-optimisé
CREATE INDEX idx_events_opt ON sensor_events(sensor_type, location_id, timestamp);

-- Requête 1 : Tous les événements récents (Skip sur sensor_type + location_id)
SELECT * FROM sensor_events WHERE timestamp > NOW() - INTERVAL '1 hour';

-- Requête 2 : Type spécifique (Skip sur location_id)
SELECT * FROM sensor_events  
WHERE sensor_type = 'temperature' AND timestamp > NOW() - INTERVAL '1 hour';  

-- Requête 3 : Type + Location (Index direct)
SELECT * FROM sensor_events  
WHERE sensor_type = 'temperature'  
  AND location_id = 42
  AND timestamp > NOW() - INTERVAL '1 hour';
```

**Économie** : 3 index évités grâce au Skip Scan ! ✅

---

## Points Clés à Retenir

🔑 **Nouveauté PostgreSQL 18** : Skip Scan permet d'utiliser des index composites même sans la première colonne.

🔑 **Efficacité** : Optimal quand la première colonne a une **faible cardinalité** (< 100-1000 valeurs distinctes).

🔑 **Économie d'index** : Réduit le besoin d'index redondants → Moins de maintenance, moins d'espace disque.

🔑 **Performance** : Gain de 10-100× par rapport au Seq Scan pour des requêtes sélectives.

🔑 **Conception d'index** : Nouvelle règle : placer en premier la colonne à **faible cardinalité**, pas nécessairement la plus utilisée.

🔑 **Activation automatique** : Skip Scan activé par défaut, PostgreSQL choisit intelligemment.

🔑 **Monitoring** : Vérifier avec `EXPLAIN` et surveiller `Loops` pour voir le nombre de "sauts".

🔑 **Limitation** : Inefficace si cardinalité de la première colonne > quelques milliers.

---

## Ressources pour Aller Plus Loin

- **Documentation PostgreSQL 18** : [Release Notes - Index Skip Scan](https://www.postgresql.org/docs/18/release-18.html)  
- **Section précédente** : 13.2. L'index B-Tree : Le couteau suisse  
- **Section suivante** : 13.4. Index spécialisés (GIN, GiST, BRIN, Hash, SP-GiST)  
- **Blog Officiel** : "PostgreSQL 18: Skip Scan Optimization Explained"

---


⏭️ [Index spécialisés](/13-indexation-et-optimisation/04-index-specialises.md)
