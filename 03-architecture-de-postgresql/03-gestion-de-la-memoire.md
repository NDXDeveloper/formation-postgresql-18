🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3. Gestion de la Mémoire : Shared Buffers vs Local Memory

## 📋 Introduction

La **gestion de la mémoire** est l'un des aspects les plus critiques des performances de PostgreSQL. Comprendre comment PostgreSQL utilise la RAM vous permettra d'optimiser vos configurations et d'éviter les pièges courants qui dégradent les performances.

PostgreSQL utilise deux types de mémoire distincts :
- **Shared Buffers** : Mémoire partagée accessible à tous les processus  
- **Local Memory** : Mémoire privée propre à chaque processus backend

Dans cette section, nous allons explorer ces deux types de mémoire, leur rôle, et comment les configurer efficacement.

---

## 🧠 Vue d'Ensemble : L'Architecture Mémoire de PostgreSQL

### Le Modèle à Deux Niveaux

```
┌────────────────────────────────────────────────────────────┐
│                    SYSTÈME D'EXPLOITATION                  │
│                         (RAM Totale)                       │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         MÉMOIRE PARTAGÉE (Shared Memory)            │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │        SHARED BUFFERS (Cache de pages)        │  │   │
│  │  │           Configuration: 4 GB                 │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  │  │ WAL Buffers │ Lock Table │ Autres structures  │  │   │
│  └─────────────────────────────────────────────────────┘   │
│         ↑            ↑            ↑            ↑           │
│         │            │            │            │           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Backend  │ │ Backend  │ │ Backend  │ │ Backend  │       │
│  │ Process  │ │ Process  │ │ Process  │ │ Process  │       │
│  │    1     │ │    2     │ │    3     │ │    4     │       │
│  ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤       │
│  │  LOCAL   │ │  LOCAL   │ │  LOCAL   │ │  LOCAL   │       │
│  │  MEMORY  │ │  MEMORY  │ │  MEMORY  │ │  MEMORY  │       │
│  │ work_mem │ │ work_mem │ │ work_mem │ │ work_mem │       │
│  │temp_buffs│ │temp_buffs│ │temp_buffs│ │temp_buffs│       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Analogie de la Bibliothèque 📚

Pour mieux comprendre, imaginez une bibliothèque universitaire :

**Shared Buffers** = **Salle de Lecture Commune**
- Espace partagé par tous les étudiants
- Contient les livres les plus consultés (cache)
- Tout le monde peut y accéder et lire
- Limitée en taille (il faut choisir quels livres y mettre)

**Local Memory** = **Bureaux Individuels**
- Chaque étudiant a son propre bureau
- Espace de travail privé pour prendre des notes, faire des calculs
- Ne peut être utilisé que par un seul étudiant à la fois
- Taille variable selon le travail à effectuer

---

## 🗄️ Shared Buffers : La Mémoire Partagée

### Qu'est-ce que les Shared Buffers ?

Les **Shared Buffers** constituent le **cache principal de PostgreSQL**. C'est une zone de mémoire RAM où PostgreSQL stocke les pages de données (8 KB chacune) pour éviter des lectures disque coûteuses.

### Structure d'une Page de Données

```
SHARED BUFFERS (en mémoire)
┌─────────────────────────────────────────┐
│  Page 1 : users (id=1-100)              │ ◄─── Backend 1 lit ici
│  Page 2 : users (id=101-200)            │
│  Page 3 : orders (id=1-50)              │ ◄─── Backend 2 lit ici
│  Page 4 : products (id=1-80)            │
│  Page 5 : users_email_idx               │ ◄─── Backend 3 lit l'index
│  ...                                    │
│  Page N : (vide, disponible)            │
└─────────────────────────────────────────┘
         ↕
      DISQUE
   /var/lib/postgresql/data/
```

### Fonctionnement du Cache

#### 1. Lecture d'une Page (Cache Miss)

```
ÉTAPE 1 : Backend demande une page
    Backend → "Je veux la page 12345 de la table users"

ÉTAPE 2 : Recherche dans Shared Buffers
    Shared Buffers → "Page 12345 non trouvée" ❌ (CACHE MISS)

ÉTAPE 3 : Lecture depuis le disque
    Disque → Lecture de la page 12345 (LENT : ~5-10 ms)

ÉTAPE 4 : Stockage dans Shared Buffers
    Page 12345 est mise en cache

ÉTAPE 5 : Retour au Backend
    Backend reçoit la page
```

**Coût** : ~5-10 millisecondes (latence disque)

#### 2. Lecture d'une Page (Cache Hit)

```
ÉTAPE 1 : Backend demande une page
    Backend → "Je veux la page 12345 de la table users"

ÉTAPE 2 : Recherche dans Shared Buffers
    Shared Buffers → "Page 12345 trouvée !" ✅ (CACHE HIT)

ÉTAPE 3 : Retour direct au Backend
    Backend reçoit la page immédiatement
```

**Coût** : ~0.001-0.01 millisecondes (accès RAM)

**Vitesse** : **500 à 1000 fois plus rapide** qu'une lecture disque !

### Le Cache Hit Ratio : Métrique Essentielle

Le **Cache Hit Ratio** mesure le pourcentage de lectures satisfaites depuis le cache :

```
Cache Hit Ratio = (Lectures en cache / Total lectures) × 100
```

**Objectif** : > 95% (idéalement > 99%)

#### Exemple de Calcul

```sql
-- Vérifier le cache hit ratio
SELECT
    sum(heap_blks_read) as disk_reads,
    sum(heap_blks_hit) as cache_reads,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 AS cache_hit_ratio
FROM pg_statio_user_tables;
```

Résultat :
```
 disk_reads | cache_reads | cache_hit_ratio
------------+-------------+-----------------
     10000  |   990000    |      99.00%     ← Excellent !
```

**Interprétation** :
- **> 99%** : Excellent, les shared_buffers sont bien dimensionnés  
- **95-99%** : Bon, peut-être augmenter légèrement  
- **< 95%** : Problème, augmenter shared_buffers ou optimiser les requêtes

### Politique d'Éviction (Replacement Policy)

Quand les Shared Buffers sont pleins et qu'une nouvelle page doit être chargée, PostgreSQL doit **évincer** une page existante.

#### Algorithme Clock Sweep (Horloge)

PostgreSQL utilise une variante de l'algorithme **LRU (Least Recently Used)** appelée **Clock Sweep** :

```
┌──────────────────────────────────────────────┐
│  Shared Buffers (circulaire)                 │
│                                              │
│   [Page A, usage=2] ◄──── Pointeur (horloge) │
│   [Page B, usage=1]                          │
│   [Page C, usage=0] ← Sera évincée           │
│   [Page D, usage=2]                          │
│   [Page E, usage=1]                          │
│   ...                                        │
└──────────────────────────────────────────────┘
```

**Principe** :
1. Chaque page a un compteur d'usage (0 à 5)  
2. À chaque accès, le compteur augmente (max 5)  
3. Le "balayeur" (clock sweep) parcourt les pages  
4. Il décrémente les compteurs  
5. Quand compteur = 0, la page peut être évincée

**Avantage** : Les pages fréquemment utilisées restent en cache.

### Pages "Sales" (Dirty Pages)

Une page est **"sale"** (dirty) si elle a été modifiée en mémoire mais pas encore écrite sur disque.

```
SHARED BUFFERS
┌───────────────────────────────────────┐
│  Page 1 : PROPRE (clean)  │           │
│  Page 2 : SALE (dirty) 🔴 │ Modifiée  │
│  Page 3 : SALE (dirty) 🔴 │ Modifiée  │
│  Page 4 : PROPRE (clean)  │           │
└───────────────────────────────────────┘
         ↓ (Background Writer)
      DISQUE
```

**Processus d'écriture** :
1. **Backend** modifie la page en mémoire → Page devient "sale"  
2. **Background Writer** écrit progressivement les pages sales  
3. **Checkpointer** force l'écriture de toutes les pages sales périodiquement

### Configuration des Shared Buffers

#### Paramètre Principal : `shared_buffers`

```sql
-- Taille des shared buffers (nécessite redémarrage)
shared_buffers = 4GB
```

#### Règles de Dimensionnement

**Environnement Serveur Dédié** :
```
shared_buffers = 25% de la RAM totale
```

**Exemples** :
- RAM = 16 GB → `shared_buffers = 4GB`
- RAM = 32 GB → `shared_buffers = 8GB`
- RAM = 64 GB → `shared_buffers = 16GB`

**⚠️ Attention** : Ne pas dépasser ~40% de la RAM totale

**Pourquoi pas 100% ?**
- PostgreSQL utilise aussi le cache du système d'exploitation (OS cache)
- Il faut de la RAM pour les processus locaux (work_mem)
- Il faut de la RAM pour le système d'exploitation

#### Double Buffering : Shared Buffers + OS Cache

```
LECTURE D'UNE PAGE
       │
       ├─> Recherche dans Shared Buffers
       │        │
       │        ├─> TROUVÉE ? → Retour rapide ✅
       │        │
       │        └─> NON TROUVÉE
       │                │
       └────────────────┴─> Recherche dans OS Cache
                             │
                             ├─> TROUVÉE ? → Lecture rapide (pas d'I/O disque)
                             │
                             └─> NON TROUVÉE → Lecture DISQUE (lent)
```

**Avantage** : Deux niveaux de cache augmentent les chances d'éviter le disque.

---

## 💼 Local Memory : La Mémoire Privée

### Qu'est-ce que la Local Memory ?

La **Local Memory** est la mémoire **privée** allouée à chaque **backend process**. Contrairement aux Shared Buffers, cette mémoire n'est pas partagée et est utilisée pour les opérations temporaires propres à chaque connexion.

### Composants de la Local Memory

```
BACKEND PROCESS
┌──────────────────────────────────────┐
│  LOCAL MEMORY                        │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  work_mem                      │  │ ← Tri, Hash, etc.
│  │  (Opérations temporaires)      │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  maintenance_work_mem          │  │ ← VACUUM, CREATE INDEX
│  │  (Opérations maintenance)      │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  temp_buffers                  │  │ ← Tables temporaires
│  │  (Tables TEMP)                 │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  Overhead & Stack              │  │ ← ~10 MB
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

---

### 1. `work_mem` : Le Plus Important

#### Rôle

`work_mem` est la quantité de mémoire qu'un **backend peut utiliser** pour des **opérations temporaires** avant de recourir au disque (fichiers temporaires).

#### Opérations Utilisant work_mem

1. **Tri (ORDER BY, DISTINCT)**  
2. **Hachage (Hash Joins, Hash Aggregates)**  
3. **Opérations Set (UNION, INTERSECT)**  
4. **Bitmap Index Scans**  
5. **Merge Joins**

#### Exemple Concret : Tri en Mémoire vs Disque

**Requête** :
```sql
SELECT * FROM users ORDER BY created_at;
```

##### Cas 1 : Tri en Mémoire (work_mem suffisant)

```
work_mem = 64 MB  
Taille des données à trier = 50 MB  

ÉTAPES :
1. Backend charge les données (50 MB)
2. Tri effectué EN MÉMOIRE (rapide)
3. Résultats retournés

Temps : ~100 ms
```

##### Cas 2 : Tri sur Disque (work_mem insuffisant)

```
work_mem = 4 MB  
Taille des données à trier = 50 MB  

ÉTAPES :
1. Backend commence à charger les données
2. Mémoire saturée (4 MB < 50 MB)
3. PostgreSQL utilise des fichiers TEMPORAIRES sur disque
4. Tri externe (multi-passes) sur disque
5. Résultats retournés

Temps : ~5000 ms (50× plus lent !)
```

**EXPLAIN indiquera** :
```
Sort Method: external merge  Disk: 50000kB  ← Avertissement !
```

**Objectif** : Éviter les sorts sur disque en augmentant `work_mem`.

#### Configuration de work_mem

```sql
-- Valeur par défaut PostgreSQL : 4 MB (souvent insuffisant en production)
-- Valeur recommandée selon le workload :
work_mem = 64MB

-- Modifier pour la session en cours (sans redémarrage)
SET work_mem = '256MB';

-- Modifier pour une requête spécifique
SET LOCAL work_mem = '1GB';  
SELECT * FROM huge_table ORDER BY column;  
RESET work_mem;  
```

#### ⚠️ Piège Critique : Multiplication de work_mem

**ATTENTION** : `work_mem` peut être utilisé **plusieurs fois** dans une seule requête !

##### Exemple 1 : Requête avec 2 Sorts

```sql
EXPLAIN ANALYZE  
SELECT * FROM  
  (SELECT * FROM users ORDER BY created_at) u
  JOIN
  (SELECT * FROM orders ORDER BY order_date) o
  ON u.id = o.user_id;
```

**Mémoire utilisée** :
- 1er tri (users) : 1× work_mem
- 2ème tri (orders) : 1× work_mem
- Hash Join : 1× work_mem
- **TOTAL** : 3× work_mem

##### Exemple 2 : Connexions Multiples

```
100 connexions actives
work_mem = 256 MB

Mémoire potentielle = 100 × 256 MB = 25.6 GB ! 🔥
```

**Risque** : **Out of Memory (OOM)** → Le système tue PostgreSQL !

#### Règles de Dimensionnement pour work_mem

**Approche Conservative** :
```
work_mem = RAM Totale / (max_connections × 2)
```

**Exemples** :
- RAM = 16 GB, max_connections = 100
  - work_mem = 16 GB / (100 × 2) = **80 MB**

**Approche Analytique (OLAP)** :
- Moins de connexions, requêtes complexes
- work_mem = **256 MB à 1 GB**

**Approche Transactionnelle (OLTP)** :
- Beaucoup de connexions, requêtes simples
- work_mem = **16 MB à 64 MB**

**Best Practice** :
- Démarrer avec une valeur conservatrice globalement
- Augmenter dynamiquement pour des requêtes spécifiques

---

### 2. `maintenance_work_mem` : Opérations de Maintenance

#### Rôle

`maintenance_work_mem` est utilisée pour les **opérations de maintenance** :
- **VACUUM**  
- **CREATE INDEX**  
- **ALTER TABLE**  
- **FOREIGN KEY checks**

#### Différence avec work_mem

```
work_mem           →  Requêtes utilisateur (SELECT, JOIN, etc.)  
maintenance_work_mem → Opérations admin (VACUUM, INDEX, etc.)  
```

**Avantage** : On peut allouer **beaucoup plus** de mémoire sans risque, car ces opérations sont rares et contrôlées.

#### Configuration

```sql
-- Valeur recommandée : 1-2 GB
maintenance_work_mem = 1GB

-- Maximum par worker autovacuum
autovacuum_work_mem = 512MB  -- (défaut: -1 = utilise maintenance_work_mem)
```

#### Exemple : CREATE INDEX

**Sans maintenance_work_mem suffisant** :
```
CREATE INDEX idx_users_email ON users(email);

Temps : 5 minutes (utilise disque)
```

**Avec maintenance_work_mem = 2GB** :
```
CREATE INDEX idx_users_email ON users(email);

Temps : 30 secondes (tout en mémoire) ✅
```

---

### 3. `temp_buffers` : Tables Temporaires

#### Rôle

`temp_buffers` est la mémoire allouée pour les **tables temporaires** (créées avec `CREATE TEMP TABLE`).

#### Utilisation

```sql
-- Créer une table temporaire
CREATE TEMP TABLE temp_results AS  
SELECT * FROM users WHERE active = true;  

-- Utiliser la table temporaire
SELECT * FROM temp_results WHERE created_at > '2024-01-01';

-- La table disparaît à la fin de la session
```

**Stockage** :
1. D'abord en RAM (temp_buffers)  
2. Si débordement → Fichiers temporaires sur disque

#### Configuration

```sql
-- Valeur par défaut : 8 MB (généralement suffisant)
temp_buffers = 16MB
```

**Note** : Rarement besoin d'augmenter, sauf usage intensif de tables temporaires.

---

### 4. `effective_cache_size` : Pas de la Vraie Mémoire !

#### Rôle

`effective_cache_size` n'alloue **AUCUNE mémoire réelle**. C'est juste un **indicateur** pour le planificateur de requêtes sur la quantité de cache disponible (PostgreSQL + OS).

#### Utilité

Le planificateur utilise cette valeur pour **estimer** si les données seront probablement en cache ou sur disque :

```
effective_cache_size élevé
    → Planificateur pense que les données sont en cache
    → Favorise les Index Scans (rapides si en cache)

effective_cache_size faible
    → Planificateur pense que les données sont sur disque
    → Favorise les Sequential Scans (plus efficaces si disque)
```

#### Configuration

```sql
-- Recommandation : 50-75% de la RAM totale
effective_cache_size = 12GB  -- Pour un serveur de 16 GB RAM
```

**Règle** :
```
effective_cache_size = shared_buffers + (RAM libre pour OS cache)
```

---

## 📊 Calcul de la Consommation Mémoire Totale

### Formule Générale

```
RAM Totale PostgreSQL =
    shared_buffers
    + (max_connections × (work_mem + temp_buffers + overhead))
    + maintenance_work_mem (pour les opérations en cours)
    + WAL buffers
    + 500 MB (overhead système)
```

### Exemple Réaliste

**Configuration** :
```ini
shared_buffers = 4GB  
work_mem = 64MB  
maintenance_work_mem = 1GB  
temp_buffers = 8MB  
max_connections = 100  
```

**Calcul** :
```
Shared Memory = 4 GB

Local Memory par connexion = 64 MB (work_mem)
                            + 8 MB (temp_buffers)
                            + 10 MB (overhead)
                            = 82 MB

Total Local Memory (100 connexions) = 100 × 82 MB = 8.2 GB

Maintenance = 1 GB (seulement si VACUUM/INDEX en cours)

Total Maximum = 4 GB + 8.2 GB + 1 GB = 13.2 GB
```

**⚠️ Attention** : Ce calcul représente le **pire cas** (toutes les connexions utilisent work_mem simultanément).

---

## 🔍 Monitoring de la Mémoire

### 1. Cache Hit Ratio (Shared Buffers)

```sql
-- Cache Hit Ratio global
SELECT
    sum(blks_hit) / (sum(blks_hit) + sum(blks_read)) * 100 AS cache_hit_ratio
FROM pg_stat_database;
```

**Objectif** : > 99%

### 2. Identifier les Sorts sur Disque

```sql
-- Activer le logging des sorts externes
SET log_temp_files = 0;  -- Log tous les fichiers temporaires
```

Puis dans les logs :
```
LOG: temporary file: path "base/pgsql_tmp/pgsql_tmp12345.0", size 52428800  
STATEMENT: SELECT * FROM users ORDER BY created_at;  
```

**Indication** : work_mem trop faible pour cette requête.

### 3. Vérifier l'Utilisation Mémoire des Backends

```sql
-- Backends actifs et leur activité
SELECT
    pid,
    usename,
    application_name,
    state,
    query_start,
    substring(query, 1, 50) as query_preview
FROM pg_stat_activity  
WHERE state != 'idle'  
ORDER BY query_start;  
```

### 4. Statistiques par Requête (pg_stat_statements)

```sql
-- Activer l'extension
CREATE EXTENSION pg_stat_statements;

-- Requêtes avec le plus d'I/O disque (cache misses)
SELECT
    substring(query, 1, 100) as query,
    calls,
    shared_blks_hit,
    shared_blks_read,
    shared_blks_read / (shared_blks_hit + shared_blks_read)::float * 100 as cache_miss_ratio
FROM pg_stat_statements  
WHERE shared_blks_read > 0  
ORDER BY shared_blks_read DESC  
LIMIT 10;  
```

---

## ⚙️ Configuration Optimale par Type de Workload

### 1. OLTP (Online Transaction Processing)

**Caractéristiques** :
- Beaucoup de petites transactions
- Nombreuses connexions (100-500)
- Requêtes simples et rapides

**Configuration** :
```ini
shared_buffers = 4GB          # 25% RAM  
work_mem = 16MB               # Petite valeur (beaucoup de connexions)  
maintenance_work_mem = 1GB  
effective_cache_size = 12GB   # 75% RAM  
max_connections = 200  
```

### 2. OLAP (Online Analytical Processing)

**Caractéristiques** :
- Peu de connexions (5-20)
- Requêtes complexes avec grands sorts/jointures
- Longs traitements analytiques

**Configuration** :
```ini
shared_buffers = 8GB          # 50% RAM  
work_mem = 512MB              # Grande valeur (peu de connexions)  
maintenance_work_mem = 4GB  
effective_cache_size = 14GB  
max_connections = 20  
```

### 3. Mixed Workload (Mixte)

**Configuration équilibrée** :
```ini
shared_buffers = 6GB  
work_mem = 64MB  
maintenance_work_mem = 2GB  
effective_cache_size = 12GB  
max_connections = 100  
```

### 4. Développement Local

**Configuration légère** :
```ini
shared_buffers = 512MB  
work_mem = 4MB  
maintenance_work_mem = 256MB  
effective_cache_size = 2GB  
max_connections = 20  
```

---

## 🚨 Problèmes Courants et Solutions

### Problème 1 : Out of Memory (OOM)

**Symptômes** :
- PostgreSQL tué par le système (OOM Killer)
- Logs : `Out of memory`
- Serveur instable

**Causes** :
- `work_mem` trop élevé × beaucoup de connexions  
- `shared_buffers` trop grand
- Trop de connexions simultanées

**Solutions** :
```sql
-- Réduire work_mem
work_mem = 16MB

-- Réduire max_connections + utiliser PgBouncer
max_connections = 50

-- Vérifier shared_buffers (max 40% RAM)
shared_buffers = 4GB  -- Pour 16 GB RAM
```

### Problème 2 : Cache Hit Ratio Faible

**Symptômes** :
- Cache hit ratio < 95%
- Requêtes lentes
- I/O disque élevé

**Causes** :
- `shared_buffers` trop petit
- Base de données plus grande que la RAM
- Requêtes scannant beaucoup de données

**Solutions** :
```sql
-- Augmenter shared_buffers (max 25-40% RAM)
shared_buffers = 8GB

-- Ajouter des index appropriés
CREATE INDEX idx_users_email ON users(email);

-- Limiter les résultats
SELECT * FROM users LIMIT 1000;
```

### Problème 3 : Sorts Fréquents sur Disque

**Symptômes** :
- Logs remplis de `temporary file: path ...`
- Requêtes avec `ORDER BY` très lentes
- EXPLAIN montre `Sort Method: external merge Disk`

**Causes** :
- `work_mem` trop faible
- Requêtes triant beaucoup de données

**Solutions** :
```sql
-- Augmenter work_mem pour cette session
SET work_mem = '256MB';

-- Ou globalement (avec prudence)
ALTER SYSTEM SET work_mem = '128MB';

-- Limiter les données à trier
SELECT * FROM users ORDER BY created_at LIMIT 100;
```

### Problème 4 : VACUUM/INDEX Lents

**Symptômes** :
- `CREATE INDEX` prend des heures  
- `VACUUM` très lent
- Table bloat

**Cause** :
- `maintenance_work_mem` trop faible

**Solution** :
```sql
-- Augmenter maintenance_work_mem
maintenance_work_mem = 2GB

-- Ou temporairement pour une session
SET maintenance_work_mem = '4GB';  
CREATE INDEX idx_huge_table ON huge_table(column);  
```

---

## 🎯 Checklist de Configuration Mémoire

### Démarrage Rapide (Serveur de 16 GB RAM)

```ini
# postgresql.conf

# === MÉMOIRE PARTAGÉE ===
shared_buffers = 4GB                    # 25% RAM

# === MÉMOIRE LOCALE ===
work_mem = 64MB                         # Ajuster selon workload  
maintenance_work_mem = 1GB              # Pour VACUUM/INDEX  
temp_buffers = 8MB                      # Rarement modifié  

# === PLANIFICATEUR ===
effective_cache_size = 12GB             # 75% RAM

# === AUTRES ===
max_connections = 100                   # Ajuster + utiliser PgBouncer
```

### Validation de la Configuration

```sql
-- Vérifier les paramètres actuels
SHOW shared_buffers;  
SHOW work_mem;  
SHOW maintenance_work_mem;  
SHOW effective_cache_size;  

-- Vérifier le cache hit ratio
SELECT
    sum(blks_hit) / (sum(blks_hit) + sum(blks_read)) * 100 AS cache_hit_ratio
FROM pg_stat_database;
```

---

## 📝 Résumé des Concepts Clés

### Shared Buffers (Mémoire Partagée)

- ✅ **Cache principal** de PostgreSQL  
- ✅ Partagé entre **tous les processus**  
- ✅ Stocke les **pages de données** (8 KB)  
- ✅ Dimensionnement : **25% de la RAM** (serveur dédié)  
- ✅ Objectif : **Cache Hit Ratio > 99%**

### Local Memory (Mémoire Privée)

- ✅ **Privée** à chaque backend process  
- ✅ `work_mem` : Sorts, Hash, Joins → **16-64 MB** (OLTP), **256 MB-1 GB** (OLAP)  
- ✅ `maintenance_work_mem` : VACUUM, INDEX → **1-4 GB**  
- ✅ `temp_buffers` : Tables temporaires → **8-16 MB**  
- ✅ `effective_cache_size` : Indicateur planificateur → **50-75% RAM**

### Calcul Mémoire Totale

```
RAM Totale = shared_buffers + (max_connections × work_mem) + overhead
```

### Pièges à Éviter

- ❌ Ne pas sous-estimer la multiplication de `work_mem`  
- ❌ Ne pas allouer > 40% RAM aux `shared_buffers`  
- ❌ Ne pas ignorer les sorts sur disque (logs `temporary file`)  
- ❌ Ne pas négliger `maintenance_work_mem` pour de grosses bases

---

## 🎓 Points à Retenir pour les Débutants

1. **Shared Buffers = Cache** : Plus c'est grand (dans la limite), plus c'est rapide.

2. **work_mem se multiplie** : 100 connexions × 64 MB = 6.4 GB potentiel !

3. **Le disque est 1000× plus lent que la RAM** : Éviter les sorts/hash sur disque à tout prix.

4. **Cache Hit Ratio > 99%** : C'est l'indicateur #1 à surveiller.

5. **effective_cache_size n'alloue PAS de mémoire** : C'est juste un hint pour le planificateur.

6. **PgBouncer est votre ami** : Connection pooling pour réduire la consommation mémoire.

7. **Commencer conservateur** : Augmenter progressivement en observant les métriques.

8. **Monitorer en continu** : Logs, pg_stat_statements, cache hit ratio.

---

## 🔗 Prochaine Étape

Maintenant que vous comprenez **comment PostgreSQL gère sa mémoire**, la section suivante explorera **la structure physique des données** : comment PostgreSQL stocke les données sur disque (Heap files, TOAST, WAL).


⏭️ [Structure physique : Heap files, TOAST et WAL](/03-architecture-de-postgresql/04-structure-physique.md)
