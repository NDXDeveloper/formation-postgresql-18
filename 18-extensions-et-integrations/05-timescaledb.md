🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.5. TimescaleDB : Séries temporelles et hypertables

## Introduction

TimescaleDB est l'une des extensions les plus populaires et puissantes de PostgreSQL. Elle transforme PostgreSQL en une base de données optimisée pour les **séries temporelles** (time-series data), tout en conservant toute la puissance et la flexibilité du SQL standard. Ce chapitre vous expliquera comment TimescaleDB fonctionne et comment l'utiliser efficacement.

---

## Qu'est-ce qu'une Série Temporelle ?

### Définition Simple

Une **série temporelle** est une séquence de données indexées dans le temps. Autrement dit, ce sont des données qui évoluent au fil du temps et dont l'ordre temporel est crucial.

### Exemples Concrets de Séries Temporelles

**Dans le monde réel, les séries temporelles sont partout** :

1. **Monitoring IT et DevOps**
   - Utilisation CPU, RAM, disque toutes les 10 secondes
   - Temps de réponse HTTP de vos serveurs
   - Nombre de requêtes par minute sur une API
   - Logs d'application avec timestamp

2. **IoT (Internet des Objets)**
   - Température et humidité d'un capteur météo toutes les minutes
   - Position GPS d'un véhicule toutes les secondes
   - Consommation électrique d'un bâtiment toutes les 15 minutes
   - Battements cardiaques d'une montre connectée

3. **Finance et Trading**
   - Prix d'une action boursière toutes les millisecondes
   - Taux de change EUR/USD toutes les secondes
   - Transactions par carte bancaire avec timestamp
   - Cours des cryptomonnaies

4. **E-commerce et Analytics**
   - Visites sur un site web par heure
   - Ventes journalières d'un produit
   - Clics sur une publicité
   - Sessions utilisateurs

5. **Industrie et Machines**
   - Vibrations d'une machine toutes les 100ms
   - Pression dans une canalisation
   - Vitesse de rotation d'un moteur
   - Production d'une usine par heure

### Caractéristiques Communes

Les séries temporelles partagent généralement ces propriétés :

✅ **Horodatage obligatoire** : Chaque mesure a un timestamp  
✅ **Insertion majoritaire** : 95%+ des opérations sont des INSERT  
✅ **Lecture par plages temporelles** : "Données des 7 derniers jours"  
✅ **Volume élevé** : Des millions à des milliards de points de données  
✅ **Données récentes = plus importantes** : Les données anciennes sont moins consultées  
✅ **Agrégations temporelles fréquentes** : Moyennes par heure, sommes par jour, etc.

---

## Pourquoi PostgreSQL Standard N'est Pas Optimal ?

### Les Limites de PostgreSQL pour les Séries Temporelles

PostgreSQL est excellent pour les charges de travail génériques, mais il rencontre des défis avec les séries temporelles massives :

#### 1. Performance d'Insertion

**Problème** : Insérer des millions de lignes par minute dans une table unique devient lent.

```sql
-- Table PostgreSQL classique
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INTEGER NOT NULL,
    temperature NUMERIC,
    humidity NUMERIC
);

-- Après quelques milliards de lignes, les INSERT ralentissent considérablement
-- Index B-Tree devient énorme et coûteux à maintenir
```

#### 2. Requêtes par Plage Temporelle

**Problème** : Interroger une plage de temps spécifique nécessite de scanner toute la table ou un index massif.

```sql
-- Requête typique : données des 7 derniers jours
SELECT AVG(temperature)  
FROM sensor_data  
WHERE time > NOW() - INTERVAL '7 days';  

-- PostgreSQL doit scanner l'index entier ou une grande partie de la table
-- Même avec un index sur 'time', c'est inefficace sur des milliards de lignes
```

#### 3. Maintenance et Rétention des Données

**Problème** : Supprimer les données anciennes (data retention) est coûteux.

```sql
-- Supprimer les données de plus de 1 an
DELETE FROM sensor_data  
WHERE time < NOW() - INTERVAL '1 year';  

-- Cette opération peut prendre des heures et verrouiller la table
-- Génère beaucoup de bloat (espace mort) nécessitant un VACUUM
```

#### 4. Taille de la Table et des Index

**Problème** : Tables et index deviennent gigantesques, affectant toutes les opérations.

```
Table sensor_data : 500 GB  
Index on (time) : 100 GB  
Index on (sensor_id, time) : 120 GB  
Total : 720 GB en mémoire pour être performant  
```

### Le Besoin d'une Solution Spécialisée

C'est exactement pourquoi **TimescaleDB** existe : optimiser PostgreSQL pour ces cas d'usage spécifiques.

---

## Présentation de TimescaleDB

### Qu'est-ce que TimescaleDB ?

**TimescaleDB** est une extension open-source de PostgreSQL qui ajoute :

- ✅ **Partitionnement automatique** par temps (chunking)  
- ✅ **Compression native** des données anciennes  
- ✅ **Fonctions de séries temporelles** (agrégations, interpolation, etc.)  
- ✅ **Rétention automatique** des données (data retention policies)  
- ✅ **Continuous Aggregates** (vues matérialisées incrémentales)  
- ✅ **Support distribué** (TimescaleDB Distributed) pour le scale-out

### Architecture : L'Extension Intelligente

TimescaleDB ne remplace pas PostgreSQL, **il l'augmente** :

```
┌─────────────────────────────────────┐
│      Votre Application              │
│      (utilise du SQL normal)        │
└──────────────┬──────────────────────┘
               │
               │ SQL Standard
               │
┌──────────────▼──────────────────────┐
│         TimescaleDB                 │
│  (extension PostgreSQL)             │
│  - Hypertables                      │
│  - Compression                      │
│  - Fonctions time-series            │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│       PostgreSQL Core               │
│  - ACID, transactions               │
│  - Index, contraintes               │
│  - JOINs, fonctions                 │
└─────────────────────────────────────┘
```

**Avantages majeurs** :

- 🎯 Vous utilisez du SQL standard (aucun nouveau langage à apprendre)  
- 🎯 Compatible avec tous les outils PostgreSQL (psql, pgAdmin, drivers)  
- 🎯 Toutes les fonctionnalités PostgreSQL restent disponibles  
- 🎯 Performances 10x à 100x meilleures pour les séries temporelles

### Versions et Licences

**TimescaleDB Community (Apache 2.0)** :
- Gratuit et open-source
- Hypertables, compression, continuous aggregates
- Parfait pour la plupart des cas d'usage

**TimescaleDB Enterprise** :
- Fonctionnalités avancées (multi-node, replication policies)
- Support commercial

**TimescaleDB Cloud** :
- Service managé (SaaS)
- Hébergé et géré par Timescale

---

## Le Concept d'Hypertable

### Qu'est-ce qu'une Hypertable ?

Une **hypertable** est l'abstraction centrale de TimescaleDB. C'est une table PostgreSQL normale vue de l'extérieur, mais qui est automatiquement partitionnée en **chunks** (morceaux) en coulisses.

### Visualisation : Table Normale vs Hypertable

#### Table PostgreSQL Standard

```
┌───────────────────────────────────────────┐
│          sensor_data                      │
│  (Une seule table, tous les temps)        │
│                                           │
│  2023-01-01 ... 2025-11-22                │
│  5 milliards de lignes                    │
│  500 GB                                   │
└───────────────────────────────────────────┘
```

**Problème** : Toutes les données dans une seule structure monumentale.

#### Hypertable TimescaleDB

```
┌─────────────────────────────────────────────────────┐
│         sensor_data (Hypertable)                    │
│         Vue logique unifiée                         │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┬────────────┐
        │             │             │            │
   ┌────▼────┐  ┌─────▼───┐   ┌─────▼───┐   ┌────▼────┐
   │ Chunk 1 │  │ Chunk 2 │   │ Chunk 3 │   │ Chunk 4 │
   │ Nov 15  │  │ Nov 16  │   │ Nov 17  │   │ Nov 18  │
   │ 10M row │  │ 11M row │   │ 10M row │   │  9M row │
   └─────────┘  └─────────┘   └─────────┘   └─────────┘
        ↓            ↓              ↓              ↓
   Compression   Compression      Active         Active
    (20:1)        (20:1)      (non compressé) (non compressé)
```

**Avantages** :

1. **Insertion rapide** : Seulement dans le chunk actif (récent)  
2. **Requêtes efficaces** : Seuls les chunks concernés sont lus  
3. **Compression sélective** : Chunks anciens compressés, récents non  
4. **Suppression rapide** : Suppression d'un chunk entier = DROP TABLE (instantané)  
5. **Maintenance légère** : VACUUM/REINDEX opère chunk par chunk

### Partitionnement Automatique par Temps

TimescaleDB crée automatiquement des chunks basés sur un **intervalle de temps** que vous définissez :

**Exemples d'intervalles** :

- Données très fréquentes (IoT, logs) : **1 jour par chunk**
- Données modérées (metrics) : **7 jours par chunk**
- Données éparses (analytics) : **1 mois par chunk**

```sql
-- Création d'une hypertable avec chunks de 7 jours
SELECT create_hypertable('sensor_data', by_range('time'), chunk_time_interval => INTERVAL '7 days');
```

TimescaleDB :
- ✅ Crée automatiquement les chunks au fur et à mesure  
- ✅ Route les INSERT vers le bon chunk  
- ✅ Optimise les SELECT pour ne lire que les chunks nécessaires  
- ✅ Applique la compression sur les chunks anciens

---

## Installation de TimescaleDB

### Prérequis

- PostgreSQL 12, 13, 14, 15, 16, 17 ou 18
- Droits d'administration système (pour installer l'extension)
- Droits de superutilisateur PostgreSQL (pour activer l'extension)

### Installation sur Ubuntu/Debian

```bash
# 1. Ajouter le dépôt Timescale
sudo sh -c "echo 'deb https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -c -s) main' > /etc/apt/sources.list.d/timescaledb.list"

# 2. Importer la clé GPG
wget --quiet -O - https://packagecloud.io/timescale/timescaledb/gpgkey | sudo apt-key add -

# 3. Mettre à jour et installer
sudo apt update  
sudo apt install timescaledb-2-postgresql-18  

# 4. Configurer TimescaleDB (recommandé)
sudo timescaledb-tune --quiet --yes

# 5. Redémarrer PostgreSQL
sudo systemctl restart postgresql
```

### Installation sur Red Hat/CentOS/Fedora

```bash
# 1. Ajouter le dépôt
sudo tee /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
[timescale_timescaledb]
name=timescale_timescaledb  
baseurl=https://packagecloud.io/timescale/timescaledb/el/$(rpm -E %{rhel})/\$basearch  
repo_gpgcheck=1  
gpgcheck=0  
enabled=1  
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey  
sslverify=1  
sslcacert=/etc/pki/tls/certs/ca-bundle.crt  
EOL  

# 2. Installer
sudo yum install timescaledb-2-postgresql-18

# 3. Configurer
sudo timescaledb-tune --quiet --yes

# 4. Redémarrer
sudo systemctl restart postgresql
```

### Installation sur macOS (Homebrew)

```bash
# Installer TimescaleDB
brew install timescaledb

# Configurer
timescaledb-tune --quiet --yes

# Redémarrer PostgreSQL
brew services restart postgresql
```

### Installation avec Docker

```bash
# Pull de l'image officielle
docker pull timescale/timescaledb:latest-pg18

# Lancer un conteneur
docker run -d --name timescaledb \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=password \
  timescale/timescaledb:latest-pg18
```

### Configuration Manuelle (si timescaledb-tune non utilisé)

Éditez `postgresql.conf` :

```ini
# Ajouter timescaledb au préchargement
shared_preload_libraries = 'timescaledb'

# Recommandations TimescaleDB
max_background_workers = 16  
max_parallel_workers = 8  
```

**Redémarrez PostgreSQL** après modification.

### Activation dans la Base de Données

```sql
-- Se connecter à la base
psql -U postgres -d mydb

-- Activer l'extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Vérifier l'installation
\dx timescaledb
```

**Résultat attendu** :
```
Name        | Version | Schema  | Description
------------+---------+---------+--------------------------------
timescaledb | 2.15.0  | public  | Enables scalable inserts and...
```

---

## Créer et Utiliser une Hypertable

### Étape 1 : Créer une Table Standard

Commencez par créer une table PostgreSQL normale :

```sql
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INTEGER NOT NULL,
    temperature NUMERIC(5,2),
    humidity NUMERIC(5,2),
    location TEXT
);
```

**Points importants** :

- ✅ La colonne de temps doit être de type `TIMESTAMP`, `TIMESTAMPTZ`, ou `DATE`  
- ✅ La colonne de temps doit avoir la contrainte `NOT NULL`  
- ✅ Il n'est pas nécessaire de définir de clé primaire à ce stade

### Étape 2 : Convertir en Hypertable

```sql
-- Convertir la table en hypertable
SELECT create_hypertable('sensor_data', by_range('time'));
```

**Avec options** :

```sql
-- Spécifier l'intervalle des chunks (défaut : 7 jours)
SELECT create_hypertable(
    'sensor_data',
    by_range('time'),
    chunk_time_interval => INTERVAL '1 day'
);

-- Avec partitionnement spatial supplémentaire (optionnel)
SELECT create_hypertable(
    'sensor_data',
    by_range('time'),
    partitioning_column => 'sensor_id',
    number_partitions => 4,
    chunk_time_interval => INTERVAL '1 day'
);
```

**Explication des paramètres** :

- `'sensor_data'` : Nom de la table à convertir  
- `by_range('time')` : Colonne de temps pour le partitionnement  
- `chunk_time_interval` : Durée de chaque chunk (1 jour, 7 jours, 1 mois, etc.)  
- `partitioning_column` : Colonne supplémentaire pour le partitionnement spatial (optionnel)  
- `number_partitions` : Nombre de partitions spatiales (si partitionnement spatial)

### Étape 3 : Utiliser comme une Table Normale

**Insertion de données** :

```sql
-- Insertion simple
INSERT INTO sensor_data (time, sensor_id, temperature, humidity, location)  
VALUES  
    (NOW(), 1, 22.5, 45.0, 'Room A'),
    (NOW(), 2, 23.1, 47.5, 'Room B');

-- Insertion en masse
INSERT INTO sensor_data (time, sensor_id, temperature, humidity, location)  
SELECT  
    NOW() - INTERVAL '1 hour' * generate_series(1, 10000),
    (random() * 100)::INTEGER,
    20 + (random() * 10)::NUMERIC(5,2),
    40 + (random() * 20)::NUMERIC(5,2),
    'Room ' || (random() * 10)::INTEGER;
```

**Requêtes SQL standard** :

```sql
-- Moyenne de température par capteur sur les dernières 24h
SELECT
    sensor_id,
    AVG(temperature) AS avg_temp,
    COUNT(*) AS measurements
FROM sensor_data  
WHERE time > NOW() - INTERVAL '24 hours'  
GROUP BY sensor_id  
ORDER BY avg_temp DESC;  

-- Évolution horaire
SELECT
    time_bucket('1 hour', time) AS hour,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM sensor_data  
WHERE time > NOW() - INTERVAL '7 days'  
GROUP BY hour  
ORDER BY hour DESC;  
```

**L'hypertable se comporte comme une table PostgreSQL normale** : vous utilisez INSERT, SELECT, UPDATE, DELETE classiques.

### Étape 4 : Ajouter des Index (Recommandé)

```sql
-- Index sur le temps (créé automatiquement)
-- TimescaleDB crée automatiquement un index sur la colonne de temps

-- Index composite pour améliorer les filtres fréquents
CREATE INDEX idx_sensor_time ON sensor_data (sensor_id, time DESC);

-- Index pour les recherches par localisation
CREATE INDEX idx_location ON sensor_data (location, time DESC);
```

**Bonnes pratiques d'indexation** :

- ✅ Toujours inclure la colonne de temps dans les index composites  
- ✅ Utilisez `DESC` sur le temps si vous requêtez souvent les données récentes  
- ✅ N'ajoutez des index que sur les colonnes réellement utilisées dans les WHERE

---

## Fonctions Spécifiques aux Séries Temporelles

TimescaleDB ajoute des fonctions SQL puissantes pour les séries temporelles.

### 1. time_bucket() : Agrégation Temporelle

`time_bucket()` regroupe les données par intervalles de temps réguliers. C'est comme `date_trunc()` mais plus flexible.

**Syntaxe** :
```sql
time_bucket(interval, timestamp_column)
```

**Exemples** :

```sql
-- Température moyenne par heure
SELECT
    time_bucket('1 hour', time) AS hour,
    AVG(temperature) AS avg_temp
FROM sensor_data  
WHERE time > NOW() - INTERVAL '1 day'  
GROUP BY hour  
ORDER BY hour;  

-- Comptage par intervalle de 5 minutes
SELECT
    time_bucket('5 minutes', time) AS bucket,
    COUNT(*) AS events
FROM sensor_data  
WHERE time > NOW() - INTERVAL '1 hour'  
GROUP BY bucket  
ORDER BY bucket DESC;  

-- Par jour
SELECT
    time_bucket('1 day', time) AS day,
    sensor_id,
    MAX(temperature) - MIN(temperature) AS temp_range
FROM sensor_data  
WHERE time > NOW() - INTERVAL '30 days'  
GROUP BY day, sensor_id  
ORDER BY day DESC;  
```

**Alignement des buckets** :

```sql
-- Aligner les buckets sur une heure spécifique (ex: 9h00)
SELECT
    time_bucket('1 hour', time, TIMESTAMPTZ '2025-01-01 09:00:00') AS hour,
    AVG(temperature)
FROM sensor_data  
GROUP BY hour;  
```

### 2. first() et last() : Première et Dernière Valeur

Récupérer la première ou dernière valeur dans un groupe temporel.

```sql
-- Première et dernière température de chaque capteur par heure
SELECT
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    first(temperature, time) AS first_temp,
    last(temperature, time) AS last_temp
FROM sensor_data  
WHERE time > NOW() - INTERVAL '1 day'  
GROUP BY hour, sensor_id  
ORDER BY hour DESC;  
```

### 3. histogram() : Distribution de Valeurs

Créer un histogramme de distribution.

```sql
-- Distribution des températures en 10 buckets
SELECT histogram(temperature, 20.0, 30.0, 10)  
FROM sensor_data  
WHERE time > NOW() - INTERVAL '1 day';  
```

### 4. interpolate() : Interpolation Linéaire

Combler les valeurs manquantes par interpolation.

```sql
-- Créer une série temporelle complète avec interpolation
SELECT
    time_bucket_gapfill('5 minutes', time) AS bucket,
    sensor_id,
    interpolate(AVG(temperature)) AS temperature
FROM sensor_data  
WHERE time > NOW() - INTERVAL '1 hour'  
  AND sensor_id = 1
GROUP BY bucket, sensor_id  
ORDER BY bucket;  
```

**Note** : `time_bucket_gapfill()` remplit les trous temporels et `interpolate()` calcule les valeurs manquantes.

### 5. Fonctions d'Agrégation Spécialisées

```sql
-- Calculer des statistiques hyperspécifiques
SELECT
    time_bucket('1 hour', time) AS hour,
    approx_percentile(0.95, percentile_agg(temperature)) AS p95_temp,
    approx_percentile(0.99, percentile_agg(temperature)) AS p99_temp
FROM sensor_data  
WHERE time > NOW() - INTERVAL '1 day'  
GROUP BY hour;  
```

---

## Compression des Données

### Pourquoi Compresser ?

Les données de séries temporelles sont hautement compressibles car :

- Les valeurs évoluent graduellement (température de 22.1°C à 22.2°C)
- Les timestamps sont prévisibles (intervalles réguliers)
- Les métadonnées se répètent (même sensor_id pendant des heures)

**Gains typiques** : 10x à 20x de réduction de taille !

### Activer la Compression

#### Étape 1 : Définir une Politique de Compression

```sql
-- Compresser les chunks de plus de 7 jours
ALTER TABLE sensor_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id',
    timescaledb.compress_orderby = 'time DESC'
);
```

**Paramètres** :

- `timescaledb.compress` : Active la compression sur la table  
- `compress_segmentby` : Colonnes pour regrouper les données (ex: sensor_id, location)  
- `compress_orderby` : Ordre de tri dans les segments compressés (généralement temps décroissant)

#### Étape 2 : Ajouter une Politique Automatique

```sql
-- Compresser automatiquement après 7 jours
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');
```

**Ce qui se passe** :

1. TimescaleDB exécute régulièrement un job de compression  
2. Les chunks de plus de 7 jours sont compressés automatiquement  
3. Les données restent interrogeables normalement

### Compression Manuelle

```sql
-- Compresser un chunk spécifique
SELECT compress_chunk('_timescaledb_internal._hyper_1_1_chunk');

-- Compresser tous les chunks avant une date
SELECT compress_chunk(i)  
FROM show_chunks('sensor_data', older_than => INTERVAL '30 days') i;  
```

### Décompresser (si nécessaire)

```sql
-- Décompresser un chunk pour permettre les UPDATE/DELETE
SELECT decompress_chunk('_timescaledb_internal._hyper_1_1_chunk');
```

**Note** : Les chunks compressés sont **en lecture seule**. Pour modifier des données, il faut d'abord décompresser.

### Vérifier la Compression

```sql
-- Statistiques de compression
SELECT
    chunk_name,
    before_compression_total_bytes,
    after_compression_total_bytes,
    before_compression_total_bytes::FLOAT / after_compression_total_bytes AS compression_ratio
FROM chunk_compression_stats('sensor_data')  
ORDER BY chunk_name;  
```

**Résultat typique** :
```
chunk_name         | before_bytes | after_bytes | ratio
-------------------+--------------+-------------+-------
_hyper_1_1_chunk   | 536870912    | 26843545    | 20.0
_hyper_1_2_chunk   | 536870912    | 28932456    | 18.5
```

---

## Rétention des Données (Data Retention)

### Le Problème

Les séries temporelles accumulent énormément de données. Stocker indéfiniment n'est pas viable :

- Coût de stockage croissant
- Performances dégradées
- Données anciennes rarement consultées

### Solution : Politique de Rétention

Définissez une durée de conservation, après quoi les données sont automatiquement supprimées.

```sql
-- Supprimer automatiquement les données de plus de 90 jours
SELECT add_retention_policy('sensor_data', INTERVAL '90 days');
```

**Comment ça marche** :

- TimescaleDB exécute un job périodique (par défaut toutes les heures)
- Les chunks entiers plus anciens que la rétention sont **DROP**pés (suppression instantanée)
- Pas de VACUUM coûteux, pas de bloat

### Gestion des Politiques

```sql
-- Lister toutes les politiques sur une hypertable
SELECT * FROM timescaledb_information.jobs  
WHERE hypertable_name = 'sensor_data';  

-- Modifier une politique de rétention
SELECT alter_job(
    1001,  -- job_id (trouvé ci-dessus)
    schedule_interval => INTERVAL '6 hours'
);

-- Supprimer une politique
SELECT remove_retention_policy('sensor_data');
```

### Rétention Manuelle

```sql
-- Supprimer manuellement les chunks anciens
SELECT drop_chunks('sensor_data', older_than => INTERVAL '1 year');

-- Supprimer avec clause WHERE (plus sélectif)
SELECT drop_chunks('sensor_data', older_than => INTERVAL '6 months', cascade_to_materializations => false);
```

---

## Continuous Aggregates (Agrégations Continues)

### Le Problème

Calculer des agrégations sur des milliards de lignes est lent :

```sql
-- Cette requête peut prendre plusieurs minutes
SELECT
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp
FROM sensor_data  
WHERE time > NOW() - INTERVAL '1 year'  
GROUP BY hour, sensor_id;  
```

### La Solution : Continuous Aggregates

Une **continuous aggregate** est une vue matérialisée spéciale qui :

- ✅ Se rafraîchit automatiquement et incrémentalement  
- ✅ Ne recalcule que les nouvelles données (pas tout)  
- ✅ Peut être interrogée comme une table normale  
- ✅ Améliore les performances de 10x à 1000x

### Créer une Continuous Aggregate

```sql
-- Vue matérialisée des moyennes horaires
CREATE MATERIALIZED VIEW sensor_hourly  
WITH (timescaledb.continuous) AS  
SELECT  
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp,
    COUNT(*) AS measurements
FROM sensor_data  
GROUP BY hour, sensor_id;  
```

### Ajouter une Politique de Rafraîchissement

```sql
-- Rafraîchir toutes les heures
SELECT add_continuous_aggregate_policy('sensor_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

**Paramètres** :

- `start_offset` : Début de la fenêtre à rafraîchir (3h dans le passé)  
- `end_offset` : Fin de la fenêtre (1h dans le passé, pour laisser les données récentes se stabiliser)  
- `schedule_interval` : Fréquence d'exécution

### Interroger la Continuous Aggregate

```sql
-- Requête instantanée sur les agrégations pré-calculées
SELECT
    hour,
    sensor_id,
    avg_temp
FROM sensor_hourly  
WHERE hour > NOW() - INTERVAL '7 days'  
ORDER BY hour DESC;  
```

**Performance** : Cette requête est des ordres de grandeur plus rapide qu'agréger les données brutes.

### Continuous Aggregates en Cascade

Vous pouvez créer des agrégations sur d'autres agrégations :

```sql
-- Agrégation horaire (déjà créée)
CREATE MATERIALIZED VIEW sensor_hourly ...

-- Agrégation journalière basée sur l'horaire
CREATE MATERIALIZED VIEW sensor_daily  
WITH (timescaledb.continuous) AS  
SELECT  
    time_bucket('1 day', hour) AS day,
    sensor_id,
    AVG(avg_temp) AS avg_temp,  -- Moyenne des moyennes horaires
    MAX(max_temp) AS max_temp,
    MIN(min_temp) AS min_temp
FROM sensor_hourly  
GROUP BY day, sensor_id;  
```

### Rafraîchissement Manuel

```sql
-- Rafraîchir manuellement une plage temporelle
CALL refresh_continuous_aggregate('sensor_hourly',
    '2025-11-01',
    '2025-11-22'
);
```

---

## Performance et Optimisation

### Choisir le Bon Intervalle de Chunk

**Règle générale** :

- **Petites tables** (< 1 million lignes/jour) : 7 jours par chunk  
- **Tables moyennes** (1-10 millions/jour) : 1 jour par chunk  
- **Tables massives** (> 10 millions/jour) : 12 heures ou moins

**Exemple de décision** :

```
Cas d'usage : 100 capteurs, 1 mesure/seconde chacun  
Données par jour : 100 * 60 * 60 * 24 = 8,64 millions de lignes/jour  

Recommandation : chunk_time_interval = 1 jour
```

### Modifier l'Intervalle de Chunk

```sql
-- Changer l'intervalle pour les futurs chunks
SELECT set_chunk_time_interval('sensor_data', INTERVAL '1 day');
```

**Note** : Cela n'affecte que les nouveaux chunks. Les anciens restent inchangés.

### Index Stratégiques

```sql
-- Index sur les colonnes fréquemment filtrées
CREATE INDEX idx_sensor_location_time ON sensor_data (sensor_id, location, time DESC);

-- Index partiel pour les cas spécifiques
CREATE INDEX idx_high_temp ON sensor_data (time DESC)  
WHERE temperature > 30.0;  
```

### Partitionnement Spatial (Space Partitioning)

Pour des charges distribuées sur plusieurs dimensions :

```sql
-- Partitionner par temps ET par sensor_id
SELECT create_hypertable(
    'sensor_data',
    by_range('time'),
    partitioning_column => 'sensor_id',
    number_partitions => 4,
    chunk_time_interval => INTERVAL '1 day'
);
```

**Cas d'usage** : Quand vos requêtes filtrent souvent sur une autre colonne que le temps (ex: sensor_id, user_id, device_id).

### Monitoring des Performances

```sql
-- Voir la taille des chunks
SELECT
    chunk_name,
    range_start,
    range_end,
    pg_size_pretty(total_bytes) AS size,
    pg_size_pretty(index_bytes) AS index_size
FROM chunks_detailed_size('sensor_data')  
ORDER BY range_start DESC  
LIMIT 10;  

-- Statistiques d'insertion
SELECT
    hypertable_name,
    num_chunks,
    num_dimensions
FROM timescaledb_information.hypertables  
WHERE hypertable_name = 'sensor_data';  
```

---

## Cas d'Usage Réels et Patterns

### 1. Monitoring d'Infrastructure (DevOps)

**Scénario** : Collecter des métriques système de 1000 serveurs toutes les 10 secondes.

```sql
-- Table des métriques
CREATE TABLE system_metrics (
    time TIMESTAMPTZ NOT NULL,
    hostname TEXT NOT NULL,
    cpu_percent NUMERIC(5,2),
    memory_percent NUMERIC(5,2),
    disk_io_read BIGINT,
    disk_io_write BIGINT
);

-- Conversion en hypertable
SELECT create_hypertable('system_metrics', by_range('time'), chunk_time_interval => INTERVAL '1 day');

-- Index pour requêtes par hostname
CREATE INDEX idx_hostname_time ON system_metrics (hostname, time DESC);

-- Agrégation continue : moyennes par minute
CREATE MATERIALIZED VIEW system_metrics_1min  
WITH (timescaledb.continuous) AS  
SELECT  
    time_bucket('1 minute', time) AS bucket,
    hostname,
    AVG(cpu_percent) AS avg_cpu,
    AVG(memory_percent) AS avg_memory,
    SUM(disk_io_read) AS total_read,
    SUM(disk_io_write) AS total_write
FROM system_metrics  
GROUP BY bucket, hostname;  

-- Rétention : 90 jours pour les données brutes, 1 an pour les agrégations
SELECT add_retention_policy('system_metrics', INTERVAL '90 days');  
SELECT add_retention_policy('system_metrics_1min', INTERVAL '1 year');  

-- Compression après 3 jours
ALTER TABLE system_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'hostname'
);
SELECT add_compression_policy('system_metrics', INTERVAL '3 days');
```

### 2. IoT et Capteurs

**Scénario** : 10 000 capteurs de température/humidité rapportant toutes les 5 minutes.

```sql
-- Table des mesures IoT
CREATE TABLE iot_sensors (
    time TIMESTAMPTZ NOT NULL,
    device_id TEXT NOT NULL,
    sensor_type TEXT NOT NULL,
    value NUMERIC,
    battery_level INTEGER,
    signal_strength INTEGER
);

-- Hypertable avec partitionnement spatial
SELECT create_hypertable(
    'iot_sensors',
    by_range('time'),
    partitioning_column => 'device_id',
    number_partitions => 4,
    chunk_time_interval => INTERVAL '7 days'
);

-- Agrégation horaire par type de capteur
CREATE MATERIALIZED VIEW iot_hourly  
WITH (timescaledb.continuous) AS  
SELECT  
    time_bucket('1 hour', time) AS hour,
    device_id,
    sensor_type,
    AVG(value) AS avg_value,
    MAX(value) AS max_value,
    MIN(value) AS min_value,
    AVG(battery_level) AS avg_battery
FROM iot_sensors  
GROUP BY hour, device_id, sensor_type;  

-- Alerte : batterie faible
SELECT device_id, AVG(battery_level) AS avg_battery  
FROM iot_sensors  
WHERE time > NOW() - INTERVAL '1 day'  
GROUP BY device_id  
HAVING AVG(battery_level) < 20;  
```

### 3. Analytics E-commerce

**Scénario** : Événements de tracking web (page views, clicks, purchases).

```sql
-- Table des événements
CREATE TABLE web_events (
    time TIMESTAMPTZ NOT NULL,
    user_id UUID,
    session_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    page_url TEXT,
    product_id INTEGER,
    revenue NUMERIC(10,2)
);

SELECT create_hypertable('web_events', by_range('time'), chunk_time_interval => INTERVAL '1 day');

-- Dashboard temps réel : agrégations par 5 minutes
CREATE MATERIALIZED VIEW web_events_5min  
WITH (timescaledb.continuous) AS  
SELECT  
    time_bucket('5 minutes', time) AS bucket,
    event_type,
    COUNT(*) AS event_count,
    COUNT(DISTINCT session_id) AS unique_sessions,
    SUM(revenue) AS total_revenue
FROM web_events  
GROUP BY bucket, event_type;  

-- Rafraîchissement toutes les 5 minutes
SELECT add_continuous_aggregate_policy('web_events_5min',
    start_offset => INTERVAL '15 minutes',
    end_offset => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes'
);

-- Requête : Conversion funnel sur la dernière heure
SELECT
    event_type,
    COUNT(DISTINCT session_id) AS sessions
FROM web_events  
WHERE time > NOW() - INTERVAL '1 hour'  
GROUP BY event_type  
ORDER BY event_type;  
```

### 4. Finance et Trading

**Scénario** : Prix de milliers d'actifs mis à jour toutes les secondes.

```sql
-- Table des ticks de marché
CREATE TABLE market_ticks (
    time TIMESTAMPTZ NOT NULL,
    symbol TEXT NOT NULL,
    price NUMERIC(20,8),
    volume BIGINT,
    bid NUMERIC(20,8),
    ask NUMERIC(20,8)
);

SELECT create_hypertable('market_ticks', by_range('time'), chunk_time_interval => INTERVAL '1 day');

-- Index pour requêtes par symbole
CREATE INDEX idx_symbol_time ON market_ticks (symbol, time DESC);

-- Agrégation : Bougies OHLCV (Open, High, Low, Close, Volume) par minute
CREATE MATERIALIZED VIEW ohlcv_1min  
WITH (timescaledb.continuous) AS  
SELECT  
    time_bucket('1 minute', time) AS bucket,
    symbol,
    first(price, time) AS open,
    MAX(price) AS high,
    MIN(price) AS low,
    last(price, time) AS close,
    SUM(volume) AS volume
FROM market_ticks  
GROUP BY bucket, symbol;  

-- Compression agressive après 1 jour
ALTER TABLE market_ticks SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('market_ticks', INTERVAL '1 day');

-- Requête : Volatilité sur 24h
SELECT
    symbol,
    (MAX(price) - MIN(price)) / AVG(price) * 100 AS volatility_pct
FROM market_ticks  
WHERE time > NOW() - INTERVAL '24 hours'  
GROUP BY symbol  
ORDER BY volatility_pct DESC  
LIMIT 20;  
```

---

## Migration depuis PostgreSQL Standard

### Scénario

Vous avez déjà une table PostgreSQL avec des millions de lignes et voulez migrer vers TimescaleDB.

### Méthode 1 : Migration In-Place (Table Vide ou Petite)

```sql
-- 1. Installer TimescaleDB
CREATE EXTENSION timescaledb;

-- 2. Convertir la table existante (ATTENTION : table doit être vide ou petite)
SELECT create_hypertable('sensor_data', by_range('time'));
```

**⚠️ Limitations** :

- La table ne doit pas avoir de clé primaire ou unique incluant la colonne de temps
- La table ne doit pas avoir de foreign keys pointant vers elle (pour l'instant)
- La conversion peut prendre du temps si la table est volumineuse

### Méthode 2 : Migration avec Reconstruction (Recommandé pour Tables Volumineuses)

```sql
-- 1. Créer une nouvelle table avec le même schéma
CREATE TABLE sensor_data_new (LIKE sensor_data INCLUDING ALL);

-- 2. Convertir en hypertable
SELECT create_hypertable('sensor_data_new', by_range('time'), chunk_time_interval => INTERVAL '1 day');

-- 3. Copier les données (par lots pour éviter les verrous)
INSERT INTO sensor_data_new  
SELECT * FROM sensor_data  
WHERE time >= '2024-01-01' AND time < '2024-02-01';  

-- Répéter pour chaque mois...

-- 4. Renommer
BEGIN;  
ALTER TABLE sensor_data RENAME TO sensor_data_old;  
ALTER TABLE sensor_data_new RENAME TO sensor_data;  
COMMIT;  

-- 5. Vérifier et supprimer l'ancienne table
DROP TABLE sensor_data_old;
```

### Méthode 3 : Migration Progressive (Zero Downtime)

```sql
-- 1. Créer la nouvelle hypertable
CREATE TABLE sensor_data_ts (LIKE sensor_data);  
SELECT create_hypertable('sensor_data_ts', by_range('time'));  

-- 2. Configurer la réplication logique (PostgreSQL 10+)
-- Ou utiliser un trigger pour dupliquer les INSERT

-- 3. Copier les données historiques en arrière-plan
INSERT INTO sensor_data_ts  
SELECT * FROM sensor_data  
WHERE time < NOW() - INTERVAL '1 hour'  
ORDER BY time;  

-- 4. Basculer l'application vers la nouvelle table
-- (sans downtime si réplication configurée)

-- 5. Supprimer l'ancienne table quand sûr
DROP TABLE sensor_data;
```

---

## Limitations et Considérations

### Limitations Techniques

#### 1. UPDATE et DELETE sur Données Compressées

**Problème** : Les chunks compressés sont en lecture seule.

**Solution** : Décompresser temporairement ou éviter les UPDATE/DELETE sur données anciennes.

```sql
-- Décompresser pour permettre les modifications
SELECT decompress_chunk('_timescaledb_internal._hyper_1_1_chunk');

-- Modifier
UPDATE sensor_data SET temperature = 25.0 WHERE ...;

-- Recompresser
SELECT compress_chunk('_timescaledb_internal._hyper_1_1_chunk');
```

#### 2. Foreign Keys

Les foreign keys vers une hypertable sont **limités** :

- ❌ Pas de FK pointant *vers* une colonne non-temps d'une hypertable  
- ✅ FK pointant *depuis* une hypertable vers une table normale (OK)

**Workaround** : Utiliser des contraintes applicatives ou des triggers.

#### 3. Triggers sur Hypertables

Les triggers fonctionnent, mais :

- Peuvent impacter les performances d'insertion (surtout BEFORE triggers)
- Évaluez si la logique peut être déportée en applicatif

#### 4. Contraintes UNIQUE

Les contraintes UNIQUE doivent inclure la colonne de partitionnement (temps).

```sql
-- ❌ Impossible
ALTER TABLE sensor_data ADD CONSTRAINT unique_sensor UNIQUE (sensor_id);

-- ✅ Possible
ALTER TABLE sensor_data ADD CONSTRAINT unique_sensor_time UNIQUE (sensor_id, time);
```

### Considérations Opérationnelles

#### 1. Surveillance des Jobs

Les politiques de compression, rétention, et refresh sont des jobs en arrière-plan.

```sql
-- Vérifier l'état des jobs
SELECT * FROM timescaledb_information.job_stats;

-- Voir les jobs échoués
SELECT * FROM timescaledb_information.job_stats  
WHERE last_run_status = 'Failure';  
```

#### 2. Ressources Serveur

- **CPU** : Compression et continuous aggregates consomment du CPU  
- **I/O** : Écritures intensives nécessitent des disques rapides (SSD)  
- **Mémoire** : Configurez `shared_buffers` et `work_mem` généreusement

#### 3. Backup et Restauration

**Logique (pg_dump)** :

```bash
# Dump avec TimescaleDB
pg_dump -d mydb -Fc -f backup.dump

# Restore (installer TimescaleDB d'abord sur la cible)
pg_restore -d mydb backup.dump
```

**Physique (pg_basebackup)** : Fonctionne normalement.

**Point-in-Time Recovery (PITR)** : Compatible sans problème.

---

## Ressources et Documentation

### Documentation Officielle

- **Site officiel** : https://www.timescale.com/  
- **Documentation** : https://docs.timescale.com/  
- **GitHub** : https://github.com/timescale/timescaledb  
- **Forum communautaire** : https://www.timescale.com/forum

### Tutoriels et Guides

- **Getting Started** : https://docs.timescale.com/getting-started/  
- **API Reference** : https://docs.timescale.com/api/  
- **Best Practices** : https://docs.timescale.com/use-timescale/latest/best-practices/

### Outils Complémentaires

- **Promscale** : Stockage long-terme pour Prometheus  
- **Grafana** : Visualisation (source de données TimescaleDB native)  
- **Telegraf** : Collecte de métriques vers TimescaleDB  
- **TimescaleDB Toolkit** : Extension avec fonctions avancées (hyperfunctions)

### Support et Communauté

- **Slack** : https://slack.timescale.com/  
- **Stack Overflow** : Tag [timescaledb]  
- **Reddit** : r/PostgreSQL (beaucoup de discussions TimescaleDB)

---

## Comparaison : TimescaleDB vs Alternatives

### TimescaleDB vs InfluxDB

| Critère | TimescaleDB | InfluxDB |
|---------|-------------|----------|
| **Langage** | SQL standard | InfluxQL/Flux |
| **Base** | PostgreSQL | Propre DB |
| **Relationnel** | ✅ Oui (joins, FK) | ❌ Limité |
| **Compression** | ✅ 10-20x | ✅ 10x |
| **Écosystème** | PostgreSQL entier | Propre écosystème |
| **Licence** | Apache 2.0 (base) | MIT (OSS) / Propriétaire |
| **Cas d'usage** | Séries temporelles + relationnel | Séries temporelles pures |

**Verdict** : TimescaleDB si vous avez besoin de SQL et de fonctionnalités relationnelles. InfluxDB si vous voulez une solution dédiée time-series.

### TimescaleDB vs Cassandra

| Critère | TimescaleDB | Cassandra |
|---------|-------------|-----------|
| **Architecture** | Monolithique (scale-up) | Distribuée (scale-out) |
| **Consistance** | ACID fort | Eventual consistency |
| **Langage** | SQL | CQL |
| **Complexité** | Simple | Élevée |
| **Performance** | Excellente (single-node) | Excellente (multi-node) |

**Verdict** : TimescaleDB pour la simplicité et des volumes < 100 TB. Cassandra pour du multi-datacenter à échelle planétaire.

### TimescaleDB vs PostgreSQL Natif (Partitions Déclaratives)

| Critère | TimescaleDB | PostgreSQL Partitionné |
|---------|-------------|------------------------|
| **Gestion partitions** | Automatique | Manuelle |
| **Compression** | Native | Externe (TOAST) |
| **Continuous Aggregates** | ✅ Oui | ❌ Non |
| **Fonctions time-series** | ✅ Riches | ❌ Basiques |
| **Rétention automatique** | ✅ Oui | ❌ Manuelle |

**Verdict** : TimescaleDB est une évolution naturelle de PostgreSQL pour les séries temporelles, avec beaucoup moins d'effort manuel.

---

## Bonnes Pratiques - Récapitulatif

### 1. Design de Schema

✅ **À faire** :
- Utilisez `TIMESTAMPTZ` pour la colonne de temps (gestion des fuseaux horaires)
- Choisissez un `chunk_time_interval` adapté à votre débit
- Ajoutez des index sur les colonnes fréquemment filtrées avec temps

❌ **À éviter** :
- Ne pas inclure trop de colonnes (pensez normalisation)
- Éviter les contraintes UNIQUE sans la colonne de temps
- Ne pas utiliser de types exotiques non nécessaires

### 2. Insertion de Données

✅ **À faire** :
- Insérez par lots (batch) pour meilleures performances
- Utilisez `COPY` pour imports massifs
- Assurez-vous que les timestamps sont dans un ordre croissant (ou proche)

❌ **À éviter** :
- Insertions ligne par ligne en haute fréquence (utilisez des buffers)
- Insertions désordonnées temporellement (impacte le partitionnement)
- Transactions trop longues (verrouillage)

### 3. Requêtes

✅ **À faire** :
- Filtrez toujours sur la colonne de temps
- Utilisez `time_bucket()` pour les agrégations temporelles
- Créez des continuous aggregates pour les dashboards

❌ **À éviter** :
- Requêtes sans filtre temporel (scan complet de l'hypertable)
- Agrégations lourdes sur données brutes non compressées
- Trop de JOINs complexes sur hypertables massives

### 4. Maintenance

✅ **À faire** :
- Configurez la compression après quelques jours
- Définissez des politiques de rétention claires
- Surveillez les jobs (compression, retention, refresh)
- Testez régulièrement vos backups

❌ **À éviter** :
- Laisser les données s'accumuler sans compression ni rétention
- Oublier de monitorer la croissance du disque
- Négliger les logs d'erreur des jobs

### 5. Performance

✅ **À faire** :
- Ajustez `shared_buffers`, `work_mem` pour votre charge
- Utilisez des SSD pour les I/O intensives
- Créez des continuous aggregates pour les rapports fréquents
- Partitionnez spatialement si filtres sur plusieurs dimensions

❌ **À éviter** :
- Sous-dimensionner le serveur (CPU, RAM, I/O)
- Négliger l'optimisation des index
- Compresser des chunks trop récents (données encore actives)

---

## Conclusion

TimescaleDB transforme PostgreSQL en une plateforme exceptionnelle pour les séries temporelles, en conservant toute la puissance du SQL relationnel. Les concepts clés à retenir :

### Points Essentiels

- 🎯 **Hypertables** : Tables partitionnées automatiquement en chunks temporels  
- 🎯 **Compression** : Réduction de 10-20x de la taille avec lecture transparente  
- 🎯 **Continuous Aggregates** : Vues matérialisées incrémentales pour performances extrêmes  
- 🎯 **Rétention automatique** : Suppression instantanée des données anciennes  
- 🎯 **Fonctions time-series** : `time_bucket()`, `first()`, `last()`, interpolation, etc.

### Quand Utiliser TimescaleDB ?

✅ **OUI, utilisez TimescaleDB si** :
- Vous avez des données horodatées (logs, métriques, IoT, analytics)
- Vous insérez massivement et lisez par plages temporelles
- Vous voulez combiner relationnel (JOINs) et time-series
- Vous aimez SQL et l'écosystème PostgreSQL

❌ **NON, peut-être pas si** :
- Vos données ne sont pas temporelles (utilisez PostgreSQL standard)
- Vous avez besoin de scale-out multi-datacenter (Cassandra, CockroachDB)
- Votre charge est majoritairement des UPDATE/DELETE (pas optimal pour time-series)

### Prochaines Étapes

1. **Installez TimescaleDB** sur un environnement de développement  
2. **Créez votre première hypertable** avec vos données  
3. **Expérimentez** avec compression et continuous aggregates  
4. **Évaluez les performances** par rapport à PostgreSQL standard  
5. **Déployez progressivement** en production

TimescaleDB est mature, stable, et largement adopté dans l'industrie (Cisco, IBM, Walmart, etc.). C'est un choix solide pour toute application nécessitant des capacités de séries temporelles à l'échelle.

---

## Annexe : Commandes de Référence Rapide

### Installation

```bash
# Ubuntu/Debian
sudo apt install timescaledb-2-postgresql-18

# Configuration
sudo timescaledb-tune --quiet --yes  
sudo systemctl restart postgresql  
```

### Création Hypertable

```sql
CREATE EXTENSION timescaledb;

CREATE TABLE my_table (time TIMESTAMPTZ NOT NULL, ...);  
SELECT create_hypertable('my_table', by_range('time'), chunk_time_interval => INTERVAL '1 day');  
```

### Compression

```sql
ALTER TABLE my_table SET (timescaledb.compress, timescaledb.compress_segmentby = 'device_id');  
SELECT add_compression_policy('my_table', INTERVAL '7 days');  
```

### Rétention

```sql
SELECT add_retention_policy('my_table', INTERVAL '90 days');
```

### Continuous Aggregate

```sql
CREATE MATERIALIZED VIEW my_agg  
WITH (timescaledb.continuous) AS  
SELECT time_bucket('1 hour', time) AS hour, AVG(value)  
FROM my_table  
GROUP BY hour;  

SELECT add_continuous_aggregate_policy('my_agg',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

### Monitoring

```sql
-- Taille des chunks
SELECT * FROM chunks_detailed_size('my_table');

-- Jobs actifs
SELECT * FROM timescaledb_information.jobs;

-- Statistiques compression
SELECT * FROM chunk_compression_stats('my_table');
```

⏭️ [pgvector : IA, embeddings et recherche vectorielle](/18-extensions-et-integrations/06-pgvector.md)
