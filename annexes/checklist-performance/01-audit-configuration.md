🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Audit de Configuration PostgreSQL
## Guide

---

## Table des Matières

1. [Introduction à l'Audit de Configuration](#1-introduction-%C3%A0-laudit-de-configuration)  
2. [Pourquoi Auditer sa Configuration ?](#2-pourquoi-auditer-sa-configuration-)  
3. [Les Fichiers de Configuration PostgreSQL](#3-les-fichiers-de-configuration-postgresql)  
4. [Méthodologie d'Audit](#4-m%C3%A9thodologie-daudit)  
5. [Paramètres de Mémoire](#5-param%C3%A8tres-de-m%C3%A9moire)  
6. [Paramètres de Performance](#6-param%C3%A8tres-de-performance)  
7. [Configuration du WAL](#7-configuration-du-wal)  
8. [Configuration des Connexions](#8-configuration-des-connexions)  
9. [Autovacuum et Maintenance](#9-autovacuum-et-maintenance)  
10. [Paramètres de Sécurité](#10-param%C3%A8tres-de-s%C3%A9curit%C3%A9)  
11. [Configuration I/O (PostgreSQL 18)](#11-configuration-io-postgresql-18)  
12. [Logging et Observabilité](#12-logging-et-observabilit%C3%A9)  
13. [Checklist d'Audit Complète](#13-checklist-daudit-compl%C3%A8te)  
14. [Outils d'Aide à l'Audit](#14-outils-daide-%C3%A0-laudit)  
15. [Erreurs Courantes de Configuration](#15-erreurs-courantes-de-configuration)  
16. [Conclusion et Bonnes Pratiques](#16-conclusion-et-bonnes-pratiques)

---

## 1. Introduction à l'Audit de Configuration

### Qu'est-ce qu'un Audit de Configuration ?

Un **audit de configuration** est un processus systématique qui consiste à examiner et évaluer les paramètres de configuration de votre instance PostgreSQL pour s'assurer qu'ils sont :

- **Adaptés** à votre cas d'usage (OLTP, OLAP, mixte)  
- **Optimaux** pour vos ressources matérielles disponibles  
- **Sécurisés** selon les meilleures pratiques  
- **Cohérents** entre eux (pas de contradictions)

### À Qui s'Adresse cet Audit ?

Cet audit est destiné à :
- **Développeurs** qui utilisent PostgreSQL dans leurs applications  
- **DevOps/SRE** qui déploient et maintiennent PostgreSQL  
- **DBA débutants** qui veulent comprendre les fondamentaux  
- **Toute personne** responsable d'une instance PostgreSQL en production

### Niveau de Connaissance Requis

Ce guide est conçu pour être accessible aux **débutants**. Nous expliquons chaque concept avant de l'auditer, et nous donnons des explications claires sur l'impact de chaque paramètre.

---

## 2. Pourquoi Auditer sa Configuration ?

### Les Risques d'une Mauvaise Configuration

Une configuration PostgreSQL inadaptée peut entraîner :

1. **Performances Dégradées**
   - Requêtes lentes
   - Temps de réponse élevés
   - Saturation des ressources

2. **Instabilité**
   - Crashes ou redémarrages inopinés
   - Erreurs "out of memory"
   - Corruption de données (dans les cas extrêmes)

3. **Problèmes de Sécurité**
   - Accès non autorisés
   - Authentification faible
   - Logs insuffisants pour détecter les intrusions

4. **Coûts Inutiles**
   - Sur-dimensionnement du matériel
   - Surconsommation de ressources cloud
   - Temps de maintenance excessifs

### Les Bénéfices d'un Audit Régulier

Un audit de configuration régulier permet de :

- **Optimiser les performances** : Exploiter pleinement les ressources disponibles  
- **Prévenir les problèmes** : Détecter les configurations à risque avant qu'elles ne causent des incidents  
- **Économiser des coûts** : Dimensionner correctement les ressources  
- **Améliorer la sécurité** : Identifier et corriger les failles de configuration  
- **Faciliter la maintenance** : Avoir une configuration documentée et cohérente

### Quand Effectuer un Audit ?

Il est recommandé d'auditer sa configuration :

- **Après l'installation initiale** : Vérifier que la configuration par défaut est adaptée  
- **Avant la mise en production** : S'assurer que tout est optimal  
- **Après une mise à jour majeure** : PostgreSQL 17 → 18 par exemple  
- **Après un changement matériel** : Ajout de RAM, changement de disques, etc.  
- **Périodiquement** : Tous les 3 à 6 mois pour les systèmes critiques  
- **Après un incident** : Comprendre si une mauvaise configuration a contribué au problème

---

## 3. Les Fichiers de Configuration PostgreSQL

### Les Trois Fichiers Principaux

PostgreSQL utilise principalement trois fichiers de configuration :

#### 3.1. `postgresql.conf`

C'est le **fichier de configuration principal** de PostgreSQL.

**Localisation typique** :
- Linux : `/etc/postgresql/18/main/postgresql.conf` ou `/var/lib/postgresql/data/postgresql.conf`
- Windows : `C:\Program Files\PostgreSQL\18\data\postgresql.conf`
- Docker : `/var/lib/postgresql/data/postgresql.conf`

**Contenu** :
- Paramètres de mémoire (shared_buffers, work_mem, etc.)
- Paramètres de performance
- Configuration WAL
- Logging
- Chemins de fichiers

**Format** :
```
# Commentaire
parametre = valeur
# autre_parametre = valeur_par_defaut (commenté)
```

#### 3.2. `pg_hba.conf`

Le fichier **Host-Based Authentication** contrôle qui peut se connecter à la base de données.

**Localisation** : Même répertoire que `postgresql.conf`

**Contenu** :
- Règles d'authentification par utilisateur, base de données et adresse IP
- Méthodes d'authentification (trust, md5, scram-sha-256, etc.)

**Format** :
```
# TYPE  DATABASE  USER  ADDRESS      METHOD
local   all       all                scram-sha-256  
host    all       all   127.0.0.1/32 scram-sha-256  
```

#### 3.3. `pg_ident.conf`

Ce fichier gère le **mapping** entre les utilisateurs système et les utilisateurs PostgreSQL (utilisé avec l'authentification "ident").

**Localisation** : Même répertoire que `postgresql.conf`

**Usage** : Moins courant, principalement pour des cas d'authentification avancés.

### Comment Voir la Configuration Active ?

#### Depuis psql

```sql
-- Voir tous les paramètres
SHOW ALL;

-- Voir un paramètre spécifique
SHOW shared_buffers;

-- Voir d'où vient la valeur d'un paramètre
SELECT name, setting, source, sourcefile  
FROM pg_settings  
WHERE name = 'shared_buffers';  
```

#### Localisation du Fichier de Configuration

```sql
-- Trouver le fichier postgresql.conf
SHOW config_file;

-- Trouver le fichier pg_hba.conf
SHOW hba_file;

-- Trouver le répertoire de données
SHOW data_directory;
```

### Hiérarchie des Configurations

PostgreSQL utilise une hiérarchie pour déterminer la valeur finale d'un paramètre :

1. **Valeur par défaut** : Compilée dans PostgreSQL  
2. **postgresql.conf** : Écrase la valeur par défaut  
3. **ALTER SYSTEM** : Écrit dans `postgresql.auto.conf` et écrase postgresql.conf  
4. **ALTER DATABASE/ROLE** : Configuration au niveau base/utilisateur  
5. **SET** : Au niveau session (temporaire)

**Ordre de priorité** : SET > ALTER DATABASE/ROLE > ALTER SYSTEM > postgresql.conf > défaut

---

## 4. Méthodologie d'Audit

### Approche Systématique

Pour auditer efficacement une configuration PostgreSQL, suivez cette méthodologie en 6 étapes :

#### Étape 1 : Collecte d'Informations

Avant de commencer, rassemblez les informations suivantes :

**Sur le Matériel** :
- Quantité de RAM totale
- Nombre de cœurs CPU
- Type de disques (SSD, HDD, NVMe)
- Bande passante I/O

**Sur l'Usage** :
- Type de charge (OLTP, OLAP, mixte)
- Nombre d'utilisateurs/connexions concurrentes
- Taille de la base de données
- Croissance prévue

**Sur l'Environnement** :
- Version PostgreSQL (ici : 18)
- Système d'exploitation
- Déploiement (bare metal, VM, container, cloud)
- Services partagés sur le serveur

#### Étape 2 : Extraction de la Configuration Actuelle

```sql
-- Exporter toute la configuration dans un fichier
COPY (SELECT name, setting, unit, category, short_desc
      FROM pg_settings
      ORDER BY category, name)
TO '/tmp/pg_config_audit.csv'  
WITH CSV HEADER;  
```

Ou utilisez un script shell :
```bash
psql -d postgres -c "SHOW ALL;" > config_audit.txt
```

#### Étape 3 : Analyse par Catégorie

Analysez chaque catégorie de paramètres selon les sections de ce guide (mémoire, performance, WAL, etc.).

#### Étape 4 : Identification des Problèmes

Pour chaque paramètre, posez-vous ces questions :
- La valeur est-elle adaptée à mon matériel ?
- Est-elle cohérente avec mon cas d'usage ?
- Y a-t-il un risque de sécurité ?
- Est-elle documentée/justifiée ?

#### Étape 5 : Priorisation

Classez les problèmes identifiés par :
- **Critique** : Risque de panne ou de perte de données  
- **Important** : Impact significatif sur les performances  
- **Mineur** : Optimisation possible mais impact limité

#### Étape 6 : Documentation et Suivi

Documentez :
- Les valeurs actuelles
- Les valeurs recommandées
- Le raisonnement derrière chaque changement
- Les changements appliqués et leur date

### Principes Directeurs de l'Audit

#### Principe 1 : Pas de Configuration "Universelle"

Il n'existe **pas** de configuration PostgreSQL parfaite qui fonctionne pour tous les cas. La configuration doit être adaptée à :
- Votre matériel spécifique
- Votre charge de travail
- Vos contraintes de disponibilité
- Vos objectifs de performance

#### Principe 2 : Mesurer Avant de Modifier

Ne changez jamais un paramètre sans :
- Comprendre son impact
- Mesurer les performances avant
- Tester le changement
- Mesurer les performances après

#### Principe 3 : Un Changement à la Fois

Lors de l'optimisation, modifiez un seul paramètre à la fois pour pouvoir :
- Isoler l'impact de chaque changement
- Revenir en arrière facilement si nécessaire
- Comprendre les relations entre paramètres

#### Principe 4 : Les Valeurs par Défaut Sont Conservatrices

PostgreSQL utilise des valeurs par défaut très **conservatrices** (pour fonctionner sur de petites machines). Sur des serveurs modernes, ces valeurs sont presque toujours trop basses.

---

## 5. Paramètres de Mémoire

La mémoire est la ressource la plus critique pour les performances de PostgreSQL. Une mauvaise configuration mémoire peut diviser les performances par 10 ou plus.

### 5.1. `shared_buffers`

**Description** : Mémoire partagée utilisée par PostgreSQL pour mettre en cache les données lues depuis les disques.

**Rôle** : C'est le "cache" principal de PostgreSQL. Toutes les lectures et écritures de pages de données passent d'abord par ce buffer.

#### Fonctionnement

Quand PostgreSQL a besoin d'une page de données :
1. Il cherche d'abord dans `shared_buffers` (très rapide)  
2. Si absent, il lit depuis le disque (lent) et met en cache dans `shared_buffers`  
3. Les pages fréquemment utilisées restent en mémoire

#### Valeur par Défaut

```
shared_buffers = 128MB
```

Cette valeur est **beaucoup trop basse** pour un serveur moderne.

#### Recommandations

**Formule générale** :
```
shared_buffers = 25% de la RAM totale
```

**Exemples** :
- Serveur avec 4 GB RAM : `shared_buffers = 1GB`
- Serveur avec 16 GB RAM : `shared_buffers = 4GB`
- Serveur avec 64 GB RAM : `shared_buffers = 16GB`

**Limites** :
- **Minimum** : 128 MB (valeur par défaut, trop faible)  
- **Maximum pratique** : 8-16 GB même sur très gros serveurs
- Au-delà de 16 GB, les gains sont marginaux car le système d'exploitation gère déjà son propre cache (page cache)

#### Cas Particuliers

**Serveurs avec peu de RAM (< 2 GB)** :
```
shared_buffers = 512MB
```

**Serveurs cloud avec RAM limitée** :
```
shared_buffers = 20% de la RAM
```

**Serveurs dédiés avec beaucoup de RAM (> 64 GB)** :
```
shared_buffers = 8GB à 16GB maximum
```

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- PostgreSQL sollicite constamment le disque
- Performances I/O catastrophiques
- Requêtes extrêmement lentes

**Si trop haut** :
- RAM gaspillée (peu de bénéfice au-delà de 25%)
- Risque de compétition avec le cache OS
- Temps de démarrage/arrêt plus longs

#### Comment Vérifier si la Valeur Est Correcte ?

Consultez le **cache hit ratio** :

```sql
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS hit_ratio
FROM pg_statio_user_tables;
```

**Interprétation** :
- `hit_ratio > 0.99` (99%) : Excellent, configuration adaptée  
- `hit_ratio 0.90-0.99` : Correct, possibilité d'amélioration  
- `hit_ratio < 0.90` : Mauvais, `shared_buffers` probablement trop bas

---

### 5.2. `work_mem`

**Description** : Mémoire allouée à **chaque opération** de tri ou de hash pour une requête.

**Rôle** : Utilisée pour les opérations comme `ORDER BY`, `DISTINCT`, jointures hash, agrégations.

#### Fonctionnement

- Chaque **opération** dans une requête (pas chaque requête) peut utiliser jusqu'à `work_mem`
- Une requête complexe peut avoir plusieurs opérations simultanées
- Si l'opération dépasse `work_mem`, PostgreSQL écrit sur disque (très lent)

**Exemple** : Une requête avec un `ORDER BY` et une jointure hash utilise potentiellement `2 × work_mem`

#### Valeur par Défaut

```
work_mem = 4MB
```

Cette valeur est **beaucoup trop basse** pour la plupart des cas.

#### Recommandations

**Formule générale** :
```
work_mem = RAM totale / (max_connections × 2 à 4)
```

Le diviseur dépend de la complexité moyenne de vos requêtes.

**Exemples** :

Pour un serveur avec 16 GB RAM et 100 connexions max :
```
work_mem = 16GB / (100 × 3) = ~50MB
```

**Valeurs typiques par cas d'usage** :

| Cas d'usage | work_mem |
|-------------|----------|
| OLTP léger (requêtes simples) | 10-25 MB |
| OLTP standard | 25-64 MB |
| Mixte OLTP/OLAP | 64-256 MB |
| OLAP (analytics, BI) | 256 MB - 1 GB |
| Data warehouse | 1-4 GB |

#### Cas Particuliers

**Serveurs OLAP/Analytics** :
Avec peu de connexions mais des requêtes complexes :
```
work_mem = 512MB à 2GB
```

**Environnements avec beaucoup de connexions (> 200)** :
```
work_mem = 16MB à 32MB  
max_connections = 200  
```
Utilisez un **connection pooler** (PgBouncer) pour limiter les connexions réelles.

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- Sorts et jointures utilisent le disque (opération "on-disk")
- Performances extrêmement dégradées
- Logs remplis de messages "temporary file: ... "

**Si trop haut** :
- Risque d'épuisement de la RAM
- Swapping (le pire scénario)
- Possibles erreurs "out of memory"

#### Calcul du Risque Mémoire

**Mémoire maximale théorique utilisable** :
```
max_connections × work_mem × complexité_moyenne
```

Cette valeur ne doit **jamais dépasser** la RAM disponible.

#### Comment Détecter un `work_mem` Trop Bas ?

Cherchez dans les logs PostgreSQL :

```bash
grep "temporary file" /var/log/postgresql/postgresql-*.log
```

Si vous voyez des messages comme :
```
LOG: temporary file: path "base/pgsql_tmp/pgsql_tmp12345.6", size 104857600
```

C'est le signe que des opérations débordent sur disque.

**Solution** : Augmentez `work_mem` progressivement (doublez, testez, répétez).

---

### 5.3. `maintenance_work_mem`

**Description** : Mémoire allouée aux opérations de maintenance (VACUUM, CREATE INDEX, ALTER TABLE, etc.).

**Rôle** : Accélère les opérations de maintenance qui sont généralement gourmandes en mémoire.

#### Valeur par Défaut

```
maintenance_work_mem = 64MB
```

Valeur trop basse pour la plupart des bases de données réelles.

#### Recommandations

**Formule générale** :
```
maintenance_work_mem = 5-10% de la RAM totale
```

**Exemples** :
- Serveur avec 8 GB RAM : `maintenance_work_mem = 512MB`
- Serveur avec 16 GB RAM : `maintenance_work_mem = 1GB`
- Serveur avec 64 GB RAM : `maintenance_work_mem = 4-6GB`

**Limites** :
- **Maximum pratique** : 4-8 GB
- Au-delà, les gains sont marginaux

#### Cas Particuliers

**Bases de données avec de très grandes tables** :
```
maintenance_work_mem = 8GB
```

**Environnements avec autovacuum parallèle** :
Attention : chaque worker autovacuum peut utiliser `autovacuum_work_mem` (ou `maintenance_work_mem` si non défini).

```
autovacuum_work_mem = 512MB  
maintenance_work_mem = 2GB  
```

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- Création d'index très lente
- VACUUM inefficace
- Opérations de maintenance qui durent des heures

**Si trop haut** :
- Peu de risques (utilisé uniquement pendant la maintenance)
- Mais peut causer des pics mémoire

#### Comment Vérifier l'Efficacité ?

Observez la durée des opérations de maintenance :

```sql
-- Voir la durée des VACUUM récents
SELECT schemaname, relname, last_vacuum, last_autovacuum  
FROM pg_stat_user_tables  
ORDER BY last_autovacuum DESC NULLS LAST  
LIMIT 10;  
```

Si les VACUUM prennent trop longtemps sur de grandes tables, augmentez `maintenance_work_mem`.

---

### 5.4. `effective_cache_size`

**Description** : **Estimation** de la mémoire disponible pour le cache du système d'exploitation (OS cache) + `shared_buffers`.

**Rôle** : Ce paramètre **n'alloue pas de mémoire**. Il informe simplement le **planificateur de requêtes** de la quantité de données susceptibles d'être en cache.

#### Fonctionnement

Le planificateur utilise cette valeur pour estimer si :
- Les données seront probablement en mémoire (index scan préféré)
- Les données seront probablement sur disque (sequential scan préféré)

#### Valeur par Défaut

```
effective_cache_size = 4GB
```

Cette valeur est souvent sous-estimée sur des serveurs modernes.

#### Recommandations

**Formule générale** :
```
effective_cache_size = 50-75% de la RAM totale
```

**Exemples** :
- Serveur avec 8 GB RAM : `effective_cache_size = 6GB`
- Serveur avec 16 GB RAM : `effective_cache_size = 12GB`
- Serveur avec 64 GB RAM : `effective_cache_size = 48GB`

#### Serveur Dédié vs Partagé

**Serveur dédié PostgreSQL** :
```
effective_cache_size = 75% de la RAM
```

**Serveur partagé** (avec d'autres services) :
```
effective_cache_size = 25-50% de la RAM
```

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- Le planificateur sous-estime le cache
- Préfère les sequential scans même quand un index scan serait plus rapide
- Plans de requêtes sous-optimaux

**Si trop haut** :
- Le planificateur surestime le cache
- Préfère les index scans même quand les données ne sont pas en cache
- Peut causer des performances dégradées sur de grandes tables

**Important** : Une mauvaise valeur ne cause **pas** d'erreur, juste des choix sous-optimaux du planificateur.

---

### 5.5. Tableau Récapitulatif : Configuration Mémoire

| Paramètre | Valeur par défaut | Recommandation | Formule |
|-----------|-------------------|----------------|---------|
| `shared_buffers` | 128 MB | 25% RAM | `RAM × 0.25` (max 16GB) |
| `work_mem` | 4 MB | 25-256 MB | `RAM / (max_connections × 3)` |
| `maintenance_work_mem` | 64 MB | 512 MB - 4 GB | `RAM × 0.05 à 0.10` |
| `effective_cache_size` | 4 GB | 50-75% RAM | `RAM × 0.50 à 0.75` |

#### Exemple Complet : Serveur 16 GB RAM, 100 connexions

```
# Mémoire
shared_buffers = 4GB              # 25% de 16GB  
work_mem = 50MB                   # 16GB / (100 × 3)  
maintenance_work_mem = 1GB        # ~6% de 16GB  
effective_cache_size = 12GB       # 75% de 16GB  
```

---

## 6. Paramètres de Performance

### 6.1. `max_connections`

**Description** : Nombre maximum de connexions simultanées à PostgreSQL.

**Rôle** : Limite le nombre de clients pouvant se connecter en même temps.

#### Valeur par Défaut

```
max_connections = 100
```

#### Pourquoi C'est Important ?

Chaque connexion consomme :
- Environ **10 MB de RAM** pour les structures internes
- Des ressources CPU pour la gestion
- Potentiellement `work_mem` lors de l'exécution de requêtes

**Calcul approximatif** :
```
Mémoire pour connexions = max_connections × 10MB
```

Avec 100 connexions : **~1 GB de RAM** juste pour les connexions.

#### Recommandations

**Serveur web/application OLTP** :
```
max_connections = 100 à 200
```

**Serveur analytics/BI (OLAP)** :
```
max_connections = 20 à 50
```

**Serveur avec connection pooler (PgBouncer)** :
```
max_connections = 50 à 100
```
Le pooler gère les connexions applicatives et maintient peu de connexions réelles à PostgreSQL.

#### Le Problème des Connexions Trop Nombreuses

Au-delà de 200-300 connexions actives simultanées, PostgreSQL souffre de :
- **Contention** : Compétition pour les ressources partagées (locks, buffers)  
- **Context switching** : Le CPU passe son temps à basculer entre processus  
- **Overhead mémoire** : Plusieurs GB de RAM juste pour les connexions

**Symptôme typique** :
- Serveur avec 500 connexions mais performant avec seulement 50 connexions actives
- Augmenter `max_connections` **dégrade** les performances

#### Solution : Connection Pooling

Plutôt que d'augmenter `max_connections`, utilisez un **connection pooler** comme **PgBouncer**.

**Principe** :
- Application crée 500 connexions vers PgBouncer
- PgBouncer maintient seulement 50 connexions vers PostgreSQL
- Les connexions sont réutilisées efficacement

**Configuration typique avec PgBouncer** :
```
# PostgreSQL
max_connections = 100

# PgBouncer
default_pool_size = 25  
max_client_conn = 500  
```

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- Erreur "FATAL: sorry, too many clients already"
- Applications ne peuvent plus se connecter

**Si trop haut** :
- Consommation mémoire excessive
- Contention et dégradation des performances
- Coût par connexion augmente

#### Comment Monitorer les Connexions ?

```sql
-- Nombre de connexions actives
SELECT count(*) FROM pg_stat_activity;

-- Connexions par état
SELECT state, count(*)  
FROM pg_stat_activity  
GROUP BY state;  

-- Connexions par base de données
SELECT datname, count(*)  
FROM pg_stat_activity  
GROUP BY datname;  
```

---

### 6.2. `random_page_cost` et `seq_page_cost`

**Description** : Coûts relatifs utilisés par le planificateur de requêtes pour estimer le coût d'accès aux données.

#### `random_page_cost`

**Rôle** : Coût estimé pour lire une page aléatoire depuis le disque (typiquement via un index).

**Valeur par défaut** :
```
random_page_cost = 4.0
```

Cette valeur a été définie à l'époque des **disques durs rotatifs** (HDD). Sur SSD/NVMe, elle est **obsolète**.

#### `seq_page_cost`

**Rôle** : Coût estimé pour lire une page de façon séquentielle (scan complet de table).

**Valeur par défaut** :
```
seq_page_cost = 1.0
```

Cette valeur sert de **référence** pour les autres coûts.

#### Pourquoi Ces Paramètres Sont Importants ?

Le planificateur de requêtes utilise ces coûts pour décider :
- **Index scan** vs **Sequential scan**
- Quel index utiliser
- Dans quel ordre effectuer les jointures

**Rapport par défaut** :
```
random_page_cost / seq_page_cost = 4.0
```

Cela signifie : "Lire une page aléatoire coûte 4× plus cher qu'une page séquentielle".

#### Recommandations par Type de Stockage

**SSD (Solid State Drive)** :
```
random_page_cost = 1.1  
seq_page_cost = 1.0  
```
Sur SSD, l'accès aléatoire et séquentiel ont des performances similaires.

**NVMe (Plus rapide que SSD)** :
```
random_page_cost = 1.0  
seq_page_cost = 1.0  
```
Sur NVMe, aucune différence significative.

**HDD (Disques durs rotatifs)** :
```
random_page_cost = 4.0  # Valeur par défaut OK  
seq_page_cost = 1.0  
```

**Cloud (EBS Amazon, Azure Disks)** :
```
random_page_cost = 1.5 à 2.0
```
Les performances varient selon le type de stockage cloud.

#### Impact d'une Mauvaise Configuration

**`random_page_cost` trop élevé** (ex: 4.0 sur SSD) :
- Le planificateur évite les index scans
- Préfère les sequential scans même sur petites tables
- Résultat : Requêtes lentes, index inutilisés

**`random_page_cost` trop bas** (ex: 1.0 sur HDD) :
- Le planificateur abuse des index scans
- Trop d'I/O aléatoires sur disques lents
- Résultat : Requêtes encore plus lentes

#### Comment Détecter un Problème ?

Observez les plans de requêtes :

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT * FROM ma_table WHERE colonne_indexee = 'valeur';  
```

Si vous voyez systématiquement des **Seq Scan** alors que vous avez des index, vérifiez `random_page_cost`.

---

### 6.3. `effective_io_concurrency`

**Description** : Nombre de requêtes I/O simultanées que le disque peut gérer efficacement.

**Rôle** : Indique à PostgreSQL combien d'opérations I/O il peut lancer en parallèle (bitmap scans).

#### Valeur par Défaut

```
effective_io_concurrency = 1
```

Cette valeur est **adaptée aux HDD** mais sous-optimale pour SSD/NVMe.

#### Recommandations

**HDD (disques durs)** :
```
effective_io_concurrency = 2
```

**SSD (SATA)** :
```
effective_io_concurrency = 200
```

**NVMe** :
```
effective_io_concurrency = 300 à 1000
```

**Cloud (AWS EBS, Azure Disks)** :
```
effective_io_concurrency = 100 à 300
```
Dépend du type de volume (gp2, gp3, io1, io2).

#### PostgreSQL 18 : Nouveautés I/O

**Nouveau paramètre** : `io_method`

```
# PostgreSQL 18
io_method = 'worker'  # Nouveau : I/O asynchrone (AIO)
                     # Ancienne valeur : 'sync'
```

Le mode `async` peut améliorer les performances I/O de **jusqu'à 3×** sur des charges de travail intensives.

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- PostgreSQL n'exploite pas le parallélisme I/O du disque
- Sous-utilisation des SSD/NVMe

**Si trop haut** :
- Peu de risques sur SSD/NVMe
- Sur HDD, peut causer de la contention

---

### 6.4. `max_parallel_workers_per_gather`

**Description** : Nombre maximum de workers parallèles qu'une requête peut utiliser.

**Rôle** : Permet la **parallélisation des requêtes** pour accélérer les scans et agrégations.

#### Valeur par Défaut

```
max_parallel_workers_per_gather = 2
```

#### Recommandations

**Serveur avec peu de cœurs (4-8)** :
```
max_parallel_workers_per_gather = 2
```

**Serveur avec cœurs moyens (8-16)** :
```
max_parallel_workers_per_gather = 4
```

**Serveur avec beaucoup de cœurs (16+)** :
```
max_parallel_workers_per_gather = 8
```

#### Paramètres Liés

```
max_parallel_workers = 8           # Total de workers parallèles disponibles  
max_worker_processes = 8           # Processus workers totaux (parallèles + autres)  
```

**Relation** :
```
max_parallel_workers ≤ max_worker_processes  
max_parallel_workers_per_gather ≤ max_parallel_workers  
```

#### Quand Est-ce Utilisé ?

La parallélisation est utilisée pour :
- Sequential scans sur grandes tables
- Agrégations (GROUP BY, DISTINCT)
- Jointures sur grandes tables
- CREATE INDEX

**Seuils par défaut** :
- La table doit avoir > 8 MB (configurable avec `min_parallel_table_scan_size`)
- Les index doivent avoir > 512 KB (configurable avec `min_parallel_index_scan_size`)

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- Pas de parallélisation des requêtes
- Requêtes lentes sur grandes tables

**Si trop haut** :
- Contention CPU
- Peut ralentir les petites requêtes concurrentes

---

## 7. Configuration du WAL

Le **WAL (Write-Ahead Log)** est le journal de transactions de PostgreSQL. C'est un composant **critique** pour la durabilité et la réplication.

### 7.1. Qu'est-ce que le WAL ?

Le WAL enregistre **toutes les modifications** avant qu'elles ne soient écrites dans les fichiers de données.

**Principe** :
1. Transaction effectue une modification  
2. Modification écrite dans le WAL (sur disque, rapide)  
3. Plus tard, modification écrite dans les fichiers de données (flush asynchrone)

**Avantages** :
- **Durabilité** : En cas de crash, PostgreSQL rejoue le WAL  
- **Réplication** : Le WAL peut être envoyé à des standby servers  
- **PITR** : Point-In-Time Recovery via archives WAL

### 7.2. `wal_level`

**Description** : Quantité d'informations écrites dans le WAL.

**Valeurs possibles** :
- `minimal` : Minimum (pas de réplication possible)  
- `replica` : Suffisant pour la réplication physique (défaut)  
- `logical` : Nécessaire pour la réplication logique

#### Valeur par Défaut

```
wal_level = replica
```

C'est la valeur **recommandée** pour la plupart des cas.

#### Recommandations

**Production avec réplication** :
```
wal_level = replica
```

**Production avec réplication logique** :
```
wal_level = logical
```

**Test/développement (sans réplication)** :
```
wal_level = replica  # Gardez replica au cas où
```

Ne passez **jamais** en `minimal` en production.

---

### 7.3. `max_wal_size` et `min_wal_size`

**Description** : Contrôle la taille des fichiers WAL avant déclencement d'un checkpoint.

#### `max_wal_size`

**Rôle** : Taille maximale du WAL avant qu'un checkpoint soit forcé.

**Valeur par défaut** :
```
max_wal_size = 1GB
```

Cette valeur est **trop basse** pour des systèmes avec beaucoup d'écritures.

#### Recommandations

**Système avec peu d'écritures (OLTP léger)** :
```
max_wal_size = 2GB
```

**Système avec écritures modérées** :
```
max_wal_size = 4-8GB
```

**Système avec écritures intensives (OLTP chargé, bulk inserts)** :
```
max_wal_size = 16-32GB
```

**Data warehouse (charges batch)** :
```
max_wal_size = 64GB ou plus
```

#### `min_wal_size`

**Rôle** : Taille minimale du WAL conservée entre checkpoints.

**Valeur par défaut** :
```
min_wal_size = 80MB
```

**Recommandation** :
```
min_wal_size = 25% de max_wal_size
```

Exemple :
```
max_wal_size = 8GB  
min_wal_size = 2GB  
```

#### Impact d'une Mauvaise Configuration

**`max_wal_size` trop bas** :
- Checkpoints trop fréquents
- I/O pic lors des checkpoints
- Dégradation des performances d'écriture
- Logs remplis de "checkpoints occurring too frequently"

**`max_wal_size` trop haut** :
- Recovery plus long après un crash
- Espace disque consommé
- Mais généralement peu de risques

#### Comment Détecter un Problème ?

Surveillez les logs :

```bash
grep "checkpoints" /var/log/postgresql/postgresql-*.log
```

Si vous voyez fréquemment :
```
LOG: checkpoints are occurring too frequently (X seconds apart)  
HINT: Consider increasing the configuration parameter "max_wal_size".  
```

Augmentez `max_wal_size`.

---

### 7.4. `checkpoint_timeout`

**Description** : Temps maximum entre deux checkpoints automatiques.

**Rôle** : Force un checkpoint même si `max_wal_size` n'est pas atteint.

#### Valeur par Défaut

```
checkpoint_timeout = 5min
```

Cette valeur peut être trop basse pour des systèmes avec beaucoup d'écritures.

#### Recommandations

**Système OLTP léger** :
```
checkpoint_timeout = 5min  # OK
```

**Système avec écritures modérées** :
```
checkpoint_timeout = 10-15min
```

**Système avec écritures intensives** :
```
checkpoint_timeout = 30min
```

**Balance entre** :
- Recovery time (plus court = checkpoints fréquents)
- Overhead I/O (plus long = checkpoints moins fréquents)

#### Trade-off

**Checkpoints fréquents** :
- ✅ Recovery rapide après crash  
- ❌ Overhead I/O constant

**Checkpoints espacés** :
- ✅ Moins d'overhead I/O  
- ❌ Recovery plus long

---

### 7.5. `checkpoint_completion_target`

**Description** : Fraction de `checkpoint_timeout` sur laquelle étaler l'écriture du checkpoint.

**Rôle** : Évite les pics d'I/O en étalant l'écriture du checkpoint.

#### Valeur par Défaut

```
checkpoint_completion_target = 0.9
```

Cette valeur est **bonne** et recommandée.

#### Fonctionnement

Avec `checkpoint_timeout = 5min` et `checkpoint_completion_target = 0.9` :
- Le checkpoint a 5 minutes pour se terminer
- PostgreSQL étale l'écriture sur `5min × 0.9 = 4.5 minutes`
- Les 30 dernières secondes sont une marge

#### Recommandations

**Valeur standard (recommandée)** :
```
checkpoint_completion_target = 0.9
```

Ne modifiez ce paramètre que si vous rencontrez des problèmes spécifiques.

**Si pics I/O trop importants** :
```
checkpoint_completion_target = 0.95
```

---

### 7.6. Tableau Récapitulatif : Configuration WAL

| Paramètre | Défaut | OLTP léger | OLTP chargé | Data Warehouse |
|-----------|--------|------------|-------------|----------------|
| `wal_level` | replica | replica | replica | replica |
| `max_wal_size` | 1GB | 2-4GB | 8-16GB | 32-64GB |
| `min_wal_size` | 80MB | 500MB | 2-4GB | 8-16GB |
| `checkpoint_timeout` | 5min | 5-10min | 15-30min | 30min |
| `checkpoint_completion_target` | 0.9 | 0.9 | 0.9 | 0.9 |

---

## 8. Configuration des Connexions

### 8.1. `listen_addresses`

**Description** : Adresses IP sur lesquelles PostgreSQL écoute les connexions.

**Valeur par défaut** :
```
listen_addresses = 'localhost'
```

PostgreSQL n'accepte que les connexions locales.

#### Recommandations

**Développement local** :
```
listen_addresses = 'localhost'
```

**Production (avec réseau)** :
```
listen_addresses = '*'
```
Accepte les connexions de toutes les interfaces réseau.

**Production (sécurité renforcée)** :
```
listen_addresses = '192.168.1.10'  # IP spécifique du serveur
```

#### Sécurité

Modifier `listen_addresses` **ne suffit pas** pour la sécurité. Le fichier `pg_hba.conf` contrôle **qui peut réellement se connecter**.

**Bonne pratique** :
1. `listen_addresses = '*'` dans `postgresql.conf`  
2. Règles strictes dans `pg_hba.conf`

---

### 8.2. `port`

**Description** : Port TCP sur lequel PostgreSQL écoute.

**Valeur par défaut** :
```
port = 5432
```

#### Recommandations

**Production standard** :
```
port = 5432
```

**Production avec sécurité par obscurité** :
```
port = 54321  # Port non-standard
```

**Attention** : Changer le port n'apporte **pas** de sécurité réelle. Utilisez plutôt :
- Firewall correctement configuré
- Authentification forte (SCRAM-SHA-256)
- SSL/TLS activé

---

### 8.3. `max_connections` (déjà vu)

Voir section 6.1.

---

### 8.4. `superuser_reserved_connections`

**Description** : Nombre de connexions réservées aux superutilisateurs.

**Rôle** : Garantit qu'un administrateur peut toujours se connecter, même si `max_connections` est atteint.

**Valeur par défaut** :
```
superuser_reserved_connections = 3
```

#### Recommandations

**Production** :
```
superuser_reserved_connections = 3 à 5
```

**Explication** :
Si `max_connections = 100` et `superuser_reserved_connections = 3` :
- 97 connexions disponibles pour les utilisateurs normaux
- 3 connexions réservées pour les superutilisateurs

**Scénario d'urgence** :
Si votre application sature les 97 connexions, vous pouvez toujours vous connecter en tant que superutilisateur pour diagnostiquer.

---

## 9. Autovacuum et Maintenance

Le **VACUUM** est un processus essentiel dans PostgreSQL. Il récupère l'espace des lignes supprimées et prévient des problèmes graves comme le **XID wraparound**.

### 9.1. Pourquoi Autovacuum Est Critique ?

PostgreSQL utilise **MVCC (Multiversion Concurrency Control)** :
- Les UPDATE et DELETE ne suppriment pas physiquement les lignes
- Ils créent de nouvelles versions
- Les anciennes versions (dead tuples) doivent être nettoyées par VACUUM

**Sans VACUUM** :
- La base de données grossit indéfiniment (bloat)
- Les performances se dégradent
- Risque de **transaction ID wraparound** (catastrophique)

### 9.2. `autovacuum`

**Description** : Active ou désactive l'autovacuum automatique.

**Valeur par défaut** :
```
autovacuum = on
```

#### Recommandations

**Production** :
```
autovacuum = on  # NE JAMAIS DÉSACTIVER
```

**⚠️ DANGER : Ne jamais désactiver l'autovacuum en production** ⚠️

Désactiver l'autovacuum peut causer :
- Bloat massif des tables
- Dégradation progressive des performances
- XID wraparound (perte de données possible)

**Seule exception** : Désactivation temporaire pendant un import massif, puis réactivation immédiate + VACUUM manuel.

---

### 9.3. `autovacuum_max_workers`

**Description** : Nombre de processus autovacuum qui peuvent s'exécuter simultanément.

**Valeur par défaut** :
```
autovacuum_max_workers = 3
```

#### PostgreSQL 18 : Nouveautés

**Nouveau paramètre** : `autovacuum_worker_slots`

Permet d'augmenter dynamiquement le nombre de workers autovacuum selon la charge.

```
# PostgreSQL 18
autovacuum_max_workers = 5  
autovacuum_worker_slots = 10  # Nouveau  
```

#### Recommandations

**Base de données avec peu de tables** :
```
autovacuum_max_workers = 3
```

**Base de données avec beaucoup de tables (> 100)** :
```
autovacuum_max_workers = 5 à 8
```

**Serveur avec beaucoup d'écritures** :
```
autovacuum_max_workers = 8 à 16
```

#### Impact d'une Mauvaise Configuration

**Si trop bas** :
- Autovacuum ne suit pas le rythme des modifications
- Accumulation de dead tuples
- Bloat progressif

**Si trop haut** :
- Contention I/O
- Peut ralentir les requêtes utilisateurs

---

### 9.4. `autovacuum_vacuum_scale_factor` et `autovacuum_vacuum_threshold`

**Description** : Détermine quand déclencher un autovacuum sur une table.

#### Formule de Déclenchement

Un autovacuum se déclenche quand :
```
dead_tuples > (autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × nb_lignes_table)
```

#### Valeurs par Défaut

```
autovacuum_vacuum_scale_factor = 0.2  (20%)  
autovacuum_vacuum_threshold = 50  
```

**Exemple** :
Table avec 1,000,000 de lignes :
```
Seuil = 50 + (0.2 × 1,000,000) = 200,050 dead tuples
```

L'autovacuum se déclenche après 200,050 lignes supprimées/modifiées.

#### Recommandations

**Tables avec peu de modifications** :
```
autovacuum_vacuum_scale_factor = 0.2  # Défaut OK
```

**Tables avec beaucoup de modifications (OLTP chargé)** :
```
autovacuum_vacuum_scale_factor = 0.05 à 0.1  
autovacuum_vacuum_threshold = 50  
```

**Très grandes tables (> 100M lignes)** :
```
autovacuum_vacuum_scale_factor = 0.01 à 0.02  
autovacuum_vacuum_threshold = 1000  
```

#### PostgreSQL 18 : Nouveau Paramètre

**`autovacuum_vacuum_max_threshold`** : Plafonne le nombre de dead tuples avant déclenchement (évite les attentes trop longues sur très grandes tables).

```
# PostgreSQL 18
autovacuum_vacuum_max_threshold = 10000000  # 10M dead tuples max
```

---

### 9.5. `autovacuum_naptime`

**Description** : Temps minimum entre deux vérifications autovacuum.

**Valeur par défaut** :
```
autovacuum_naptime = 1min
```

#### Recommandations

**Base de données avec peu de modifications** :
```
autovacuum_naptime = 1min  # Défaut OK
```

**Base de données avec beaucoup de modifications** :
```
autovacuum_naptime = 30s
```

**Serveur avec beaucoup de tables** :
```
autovacuum_naptime = 15s à 30s
```

Plus la valeur est basse, plus l'autovacuum est réactif (mais plus de CPU utilisé).

---

### 9.6. `autovacuum_work_mem`

**Description** : Mémoire utilisée par chaque processus autovacuum.

**Valeur par défaut** :
```
autovacuum_work_mem = -1  # Utilise maintenance_work_mem
```

#### Recommandations

**Standard** :
```
autovacuum_work_mem = -1  # Utilise maintenance_work_mem
```

**Serveurs avec autovacuum parallèle intensif** :
```
autovacuum_work_mem = 512MB à 1GB  
maintenance_work_mem = 2GB  
```

Cela évite que plusieurs workers consomment toute la RAM.

---

### 9.7. Tableau Récapitulatif : Configuration Autovacuum

| Paramètre | Défaut | Peu modif. | Modif. modérées | Beaucoup modif. |
|-----------|--------|-----------|----------------|----------------|
| `autovacuum` | on | on | on | on |
| `autovacuum_max_workers` | 3 | 3 | 5-8 | 8-16 |
| `autovacuum_naptime` | 1min | 1min | 30s | 15-30s |
| `autovacuum_vacuum_scale_factor` | 0.2 | 0.2 | 0.1 | 0.05 |
| `autovacuum_vacuum_threshold` | 50 | 50 | 50 | 50-100 |

---

## 10. Paramètres de Sécurité

### 10.1. `password_encryption`

**Description** : Algorithme de chiffrement des mots de passe.

**Valeur par défaut (PostgreSQL 18)** :
```
password_encryption = scram-sha-256
```

#### Historique

- **PostgreSQL < 10** : `md5` (déprécié, faible)  
- **PostgreSQL 10-13** : `md5` par défaut, `scram-sha-256` disponible  
- **PostgreSQL 14+** : `scram-sha-256` par défaut

#### Recommandations

**PostgreSQL 18** :
```
password_encryption = scram-sha-256
```

**⚠️ Ne jamais utiliser MD5** : Il est vulnérable aux attaques par dictionnaire.

#### Migration de MD5 vers SCRAM

Si vous migrez d'une ancienne version :

1. Modifiez `postgresql.conf` :
```
password_encryption = scram-sha-256
```

2. Les utilisateurs doivent réinitialiser leur mot de passe :
```sql
ALTER USER mon_user PASSWORD 'nouveau_mot_de_passe';
```

3. Mettez à jour `pg_hba.conf` :
```
# Remplacez "md5" par "scram-sha-256"
host all all 0.0.0.0/0 scram-sha-256
```

---

### 10.2. `ssl` et `ssl_*`

**Description** : Active le chiffrement SSL/TLS des connexions.

#### Configuration SSL

**Activer SSL** :
```
ssl = on  
ssl_cert_file = '/etc/ssl/certs/server.crt'  
ssl_key_file = '/etc/ssl/private/server.key'  
ssl_ca_file = '/etc/ssl/certs/ca.crt'  # Optionnel  
```

#### PostgreSQL 18 : Nouveautés TLS

**TLS 1.3 et chiffrements FIPS** :
```
# PostgreSQL 18
ssl_min_protocol_version = 'TLSv1.3'  
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256'  
```

**Mode FIPS** (pour conformité gouvernementale) :
```
ssl_library = 'fips'  # Nouveau dans PG 18
```

#### Recommandations

**Production** :
```
ssl = on  
ssl_min_protocol_version = 'TLSv1.2'  
ssl_prefer_server_ciphers = on  
```

**Production haute sécurité** :
```
ssl = on  
ssl_min_protocol_version = 'TLSv1.3'  
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'  
```

#### Vérifier SSL dans pg_hba.conf

```
# Forcer SSL pour toutes les connexions réseau
hostssl all all 0.0.0.0/0 scram-sha-256
```

---

### 10.3. `log_connections` et `log_disconnections`

**Description** : Enregistre les connexions et déconnexions dans les logs.

**Valeur par défaut** :
```
log_connections = off  
log_disconnections = off  
```

#### Recommandations

**Production** :
```
log_connections = on  
log_disconnections = on  
```

**Utilité** :
- Audit de sécurité (qui se connecte ?)
- Détection d'activités suspectes
- Troubleshooting des problèmes de connexion

---

### 10.4. `log_statement`

**Description** : Quelles requêtes SQL enregistrer dans les logs.

**Valeurs possibles** :
- `none` : Aucune requête  
- `ddl` : Uniquement DDL (CREATE, ALTER, DROP)  
- `mod` : DDL + modifications (INSERT, UPDATE, DELETE)  
- `all` : Toutes les requêtes

**Valeur par défaut** :
```
log_statement = 'none'
```

#### Recommandations

**Production (sécurité standard)** :
```
log_statement = 'ddl'
```
Enregistre les modifications de schéma (important pour l'audit).

**Production (sécurité renforcée)** :
```
log_statement = 'mod'
```
Enregistre toutes les modifications de données.

**Debug/Troubleshooting (temporaire)** :
```
log_statement = 'all'
```

**⚠️ Attention** : `log_statement = 'all'` génère **beaucoup** de logs et peut contenir des informations sensibles (mots de passe dans les requêtes).

---

### 10.5. PostgreSQL 18 : Authentification OAuth 2.0

**Nouveauté majeure** : PostgreSQL 18 supporte nativement OAuth 2.0 pour l'authentification.

#### Configuration pg_hba.conf

```
# Authentification via OAuth 2.0
host all all 0.0.0.0/0 oauth2 issuer=https://auth.example.com
```

#### Avantages

- Intégration avec des fournisseurs d'identité (Okta, Auth0, Azure AD)
- Single Sign-On (SSO)
- Authentification moderne et sécurisée

---

## 11. Configuration I/O (PostgreSQL 18)

PostgreSQL 18 introduit un **nouveau sous-système I/O asynchrone** qui peut améliorer considérablement les performances.

### 11.1. `io_method`

**Description** : Méthode d'I/O utilisée par PostgreSQL.

**Nouveau dans PostgreSQL 18**.

**Valeurs possibles** :
- `sync` : I/O synchrone (ancien comportement)  
- `async` : I/O asynchrone (nouveau, plus rapide)

**Valeur par défaut (PostgreSQL 18)** :
```
io_method = 'worker'
```

#### Performances

Le mode `async` peut améliorer les performances de **jusqu'à 3× sur des charges I/O intensives**.

#### Recommandations

**PostgreSQL 18 en production** :
```
io_method = 'worker'
```

**Si problèmes de stabilité** :
```
io_method = 'sync'  # Retour au comportement classique
```

---

### 11.2. `effective_io_concurrency` (déjà vu)

Voir section 6.3.

**Avec PostgreSQL 18 et `io_method = 'worker'`**, il est recommandé d'augmenter cette valeur :

```
# PostgreSQL 18 avec I/O async
io_method = 'worker'  
effective_io_concurrency = 300  # SSD/NVMe  
```

---

## 12. Logging et Observabilité

### 12.1. `log_destination`

**Description** : Où envoyer les logs.

**Valeurs possibles** :
- `stderr` : Sortie d'erreur standard  
- `csvlog` : Format CSV (idéal pour l'analyse)  
- `syslog` : Syslog système  
- `eventlog` : Event Log Windows

**Valeur par défaut** :
```
log_destination = 'stderr'
```

#### Recommandations

**Production** :
```
log_destination = 'stderr,csvlog'
```

Le format CSV facilite l'analyse avec des outils comme **pgBadger**.

---

### 12.2. `logging_collector`

**Description** : Active la collecte de logs dans des fichiers.

**Valeur par défaut** :
```
logging_collector = off
```

#### Recommandations

**Production** :
```
logging_collector = on  
log_directory = 'log'  
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'  
log_rotation_age = 1d  
log_rotation_size = 100MB  
```

---

### 12.3. `log_min_duration_statement`

**Description** : Enregistre les requêtes qui dépassent une durée donnée.

**Valeur par défaut** :
```
log_min_duration_statement = -1  # Désactivé
```

#### Recommandations

**Production (détection slow queries)** :
```
log_min_duration_statement = 1000  # 1 seconde
```

**Optimisation poussée** :
```
log_min_duration_statement = 100  # 100 ms
```

**Attention** : Une valeur trop basse génère beaucoup de logs.

---

### 12.4. `log_line_prefix`

**Description** : Préfixe ajouté à chaque ligne de log.

**Valeur par défaut** :
```
log_line_prefix = '%m [%p] '
```

#### Recommandations

**Production (observabilité complète)** :
```
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
```

**Format** :
- `%t` : Timestamp  
- `%p` : PID du processus  
- `%u` : Utilisateur  
- `%d` : Base de données  
- `%a` : Nom de l'application  
- `%h` : Adresse IP du client

---

### 12.5. PostgreSQL 18 : Statistiques I/O et WAL par Backend

**Nouveauté** : PostgreSQL 18 enrichit les statistiques disponibles.

```sql
-- Nouvelles colonnes dans pg_stat_activity (PG 18)
SELECT pid, usename, datname,
       backend_type,
       wal_records,  -- Nouveau
       wal_bytes,    -- Nouveau
       io_time       -- Nouveau
FROM pg_stat_activity;
```

Ces statistiques permettent de mieux identifier les backends qui génèrent beaucoup de WAL ou consomment beaucoup d'I/O.

---

### 12.6. `auto_explain`

**Extension** : Enregistre automatiquement les plans d'exécution des requêtes lentes.

#### Configuration

```sql
-- Charger l'extension
ALTER SYSTEM SET shared_preload_libraries = 'auto_explain';

-- Redémarrer PostgreSQL (nécessaire)

-- Configurer auto_explain
ALTER SYSTEM SET auto_explain.log_min_duration = 1000;  -- 1 sec  
ALTER SYSTEM SET auto_explain.log_analyze = on;  
ALTER SYSTEM SET auto_explain.log_buffers = on;  
```

#### Utilité

- Capture automatiquement les plans des requêtes lentes
- Facilite le troubleshooting sans intervention manuelle
- Idéal pour identifier les requêtes problématiques en production

---

## 13. Checklist d'Audit Complète

Voici une checklist complète pour auditer votre configuration PostgreSQL.

### 13.1. Informations Système

Avant de commencer l'audit, collectez :

- [ ] Version PostgreSQL (`SELECT version();`)  
- [ ] Système d'exploitation (`uname -a` ou `ver`)  
- [ ] RAM totale (`free -h` ou Task Manager)  
- [ ] Nombre de cœurs CPU (`lscpu` ou Task Manager)  
- [ ] Type de disques (SSD, NVMe, HDD)  
- [ ] Localisation fichiers config (`SHOW config_file;`)

---

### 13.2. Mémoire

- [ ] `shared_buffers` : Est-il à ~25% de la RAM ? (max 16GB)  
- [ ] `work_mem` : Est-il adapté à `max_connections` ?  
- [ ] `maintenance_work_mem` : Est-il à 5-10% de la RAM ?  
- [ ] `effective_cache_size` : Est-il à 50-75% de la RAM ?  
- [ ] Pas de risque d'épuisement RAM : `max_connections × work_mem × 3 < RAM` ?

**Action si problème** :
- Ajuster `shared_buffers` à 25% RAM
- Calculer `work_mem = RAM / (max_connections × 3)`
- Augmenter `maintenance_work_mem` à 1-2 GB minimum

---

### 13.3. Performance

- [ ] `random_page_cost` : Adapté au type de disque (SSD = 1.1, HDD = 4.0) ?  
- [ ] `effective_io_concurrency` : Adapté au disque (SSD = 200, NVMe = 300-1000) ?  
- [ ] `max_parallel_workers_per_gather` : Adapté au nombre de cœurs ?  
- [ ] `max_parallel_workers` : ≥ `max_parallel_workers_per_gather` ?  
- [ ] `max_worker_processes` : ≥ `max_parallel_workers` ?

**Action si problème** :
- Ajuster `random_page_cost` selon le stockage
- Augmenter `effective_io_concurrency` pour SSD/NVMe
- Activer parallélisme selon CPU disponibles

---

### 13.4. WAL

- [ ] `wal_level` : `replica` ou `logical` (pas `minimal`) ?  
- [ ] `max_wal_size` : Suffisant pour éviter "checkpoints too frequently" ?  
- [ ] `min_wal_size` : ~25% de `max_wal_size` ?  
- [ ] `checkpoint_timeout` : Adapté à la charge d'écriture ?  
- [ ] `checkpoint_completion_target` : 0.9 ?

**Action si problème** :
- Augmenter `max_wal_size` à 4-16 GB selon charge
- Vérifier logs pour "checkpoints too frequently"
- Garder `checkpoint_completion_target = 0.9`

---

### 13.5. Connexions

- [ ] `max_connections` : Adapté au nombre de clients ?  
- [ ] `superuser_reserved_connections` : 3-5 ?  
- [ ] Connection pooling en place (PgBouncer) si > 100 connexions ?  
- [ ] `listen_addresses` : Configuré selon besoin réseau ?  
- [ ] `port` : Standard (5432) ou personnalisé ?

**Action si problème** :
- Limiter `max_connections` à 100-200
- Implémenter PgBouncer si beaucoup de connexions
- Configurer `listen_addresses` selon architecture réseau

---

### 13.6. Autovacuum

- [ ] `autovacuum` : ON (jamais OFF) ?  
- [ ] `autovacuum_max_workers` : Adapté au nombre de tables ?  
- [ ] `autovacuum_naptime` : Adapté à la fréquence de modifications ?  
- [ ] `autovacuum_vacuum_scale_factor` : Adapté à la taille des tables ?  
- [ ] Pas de messages "autovacuum too slow" dans les logs ?

**Action si problème** :
- Vérifier que `autovacuum = on`
- Augmenter `autovacuum_max_workers` à 5-8
- Réduire `autovacuum_vacuum_scale_factor` à 0.05-0.1 pour grandes tables
- Surveiller bloat avec des requêtes de monitoring

---

### 13.7. Sécurité

- [ ] `password_encryption` : `scram-sha-256` (pas MD5) ?  
- [ ] `ssl` : ON en production ?  
- [ ] `ssl_min_protocol_version` : TLSv1.2 minimum ?  
- [ ] `pg_hba.conf` : Règles strictes (pas de `trust` en production) ?  
- [ ] `log_connections` : ON ?  
- [ ] `log_statement` : Au moins `ddl` ?

**Action si problème** :
- Migrer de MD5 vers SCRAM-SHA-256
- Activer SSL avec certificats valides
- Revoir `pg_hba.conf` pour durcir l'authentification
- Activer logs de connexions

---

### 13.8. I/O (PostgreSQL 18)

- [ ] `io_method` : `worker` ou `io_uring` (nouveauté PG 18) ?  
- [ ] `effective_io_concurrency` : Augmenté avec `async` ?  
- [ ] Checksum activés par défaut (PG 18) ?

**Action si problème** :
- Activer `io_method = 'worker'` pour meilleures performances
- Augmenter `effective_io_concurrency` pour SSD

---

### 13.9. Logging

- [ ] `logging_collector` : ON ?  
- [ ] `log_destination` : Inclut `csvlog` ?  
- [ ] `log_min_duration_statement` : Défini (500-1000 ms) ?  
- [ ] `log_line_prefix` : Inclut timestamp, user, database ?  
- [ ] Rotation des logs configurée ?

**Action si problème** :
- Activer `logging_collector = on`
- Configurer rotation des logs (taille et âge)
- Définir `log_min_duration_statement` pour capturer slow queries

---

### 13.10. Extensions et Monitoring

- [ ] Extension `pg_stat_statements` installée et activée ?  
- [ ] `shared_preload_libraries` : Inclut extensions nécessaires ?  
- [ ] Monitoring en place (Prometheus, pgBadger, etc.) ?

**Action si problème** :
```sql
-- Installer pg_stat_statements
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
-- Redémarrer PostgreSQL
CREATE EXTENSION pg_stat_statements;
```

---

## 14. Outils d'Aide à l'Audit

### 14.1. PGTune

**URL** : https://pgtune.leopard.in.ua/

**Description** : Outil en ligne qui génère une configuration PostgreSQL optimisée selon vos ressources et cas d'usage.

**Utilisation** :
1. Indiquez la version PostgreSQL (18)  
2. Spécifiez le type de disque (SSD, HDD, SAN)  
3. Choisissez le cas d'usage (Web, OLTP, Data warehouse, Desktop, Mixed)  
4. Indiquez les ressources (RAM, CPU, connexions)  
5. PGTune génère une configuration recommandée

**Avantages** :
- Rapide et gratuit
- Bon point de départ pour une configuration

**Limites** :
- Configuration générique
- Doit être ajustée selon votre charge réelle

---

### 14.2. pgBadger

**Description** : Analyseur de logs PostgreSQL qui génère des rapports détaillés.

**Installation** :
```bash
# Debian/Ubuntu
apt-get install pgbadger

# Ou via CPAN
cpan App::pgBadger
```

**Utilisation** :
```bash
# Analyser un fichier de logs
pgbadger /var/log/postgresql/postgresql-*.log -o rapport.html

# Ouvrir le rapport
firefox rapport.html
```

**Ce que pgBadger vous montre** :
- Requêtes les plus lentes
- Requêtes les plus fréquentes
- Erreurs et warnings
- Connexions et sessions
- Checkpoints et autovacuum
- Locks et deadlocks

---

### 14.3. pg_stat_statements

**Description** : Extension PostgreSQL qui enregistre les statistiques d'exécution de toutes les requêtes.

**Installation** :
```sql
-- 1. Modifier postgresql.conf
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';

-- 2. Redémarrer PostgreSQL

-- 3. Créer l'extension
CREATE EXTENSION pg_stat_statements;
```

**Utilisation** :
```sql
-- Top 10 requêtes les plus lentes
SELECT query, mean_exec_time, calls  
FROM pg_stat_statements  
ORDER BY mean_exec_time DESC  
LIMIT 10;  

-- Top 10 requêtes les plus fréquentes
SELECT query, calls, total_exec_time  
FROM pg_stat_statements  
ORDER BY calls DESC  
LIMIT 10;  

-- Réinitialiser les statistiques
SELECT pg_stat_statements_reset();
```

---

### 14.4. pg_config

**Description** : Commande qui affiche les options de compilation de PostgreSQL.

**Utilisation** :
```bash
pg_config --version  
pg_config --configure  
pg_config --libdir  
```

Utile pour vérifier comment PostgreSQL a été compilé et où sont les bibliothèques.

---

### 14.5. Scripts SQL de Monitoring

#### Cache Hit Ratio

```sql
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit)  as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

**Objectif** : > 0.99 (99%)

#### Index Usage

```sql
SELECT
    schemaname,
    tablename,
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes  
ORDER BY idx_scan DESC;  
```

**Recherchez** : Index avec `idx_scan = 0` (inutilisés)

#### Table Bloat (estimation)

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables  
WHERE n_dead_tup > 1000  
ORDER BY dead_pct DESC;  
```

**Objectif** : `dead_pct < 5%`

#### Slow Queries (avec pg_stat_statements)

```sql
SELECT
    substring(query, 1, 100) AS query_short,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    round(total_exec_time::numeric, 2) AS total_time_ms
FROM pg_stat_statements  
WHERE mean_exec_time > 100  -- Plus de 100 ms  
ORDER BY mean_exec_time DESC  
LIMIT 20;  
```

---

## 15. Erreurs Courantes de Configuration

### 15.1. `shared_buffers` Trop Bas

**Symptôme** :
- Performances I/O très faibles
- Cache hit ratio < 90%

**Solution** :
```
shared_buffers = 4GB  # Pour 16 GB RAM
```

---

### 15.2. `work_mem` Trop Bas

**Symptôme** :
- Messages "temporary file" dans les logs
- Requêtes avec sorts/jointures très lentes

**Solution** :
```
work_mem = 64MB  # Ajuster selon RAM et connexions
```

---

### 15.3. Checkpoints Trop Fréquents

**Symptôme** :
- Log : "checkpoints are occurring too frequently"
- Pics I/O réguliers

**Solution** :
```
max_wal_size = 8GB  
checkpoint_timeout = 15min  
```

---

### 15.4. Autovacuum Désactivé

**Symptôme** :
- Bloat massif des tables
- Performances qui se dégradent progressivement
- Warnings "transaction ID wraparound"

**Solution** :
```
autovacuum = on  # NE JAMAIS DÉSACTIVER
```

Puis lancez un VACUUM manuel :
```sql
VACUUM VERBOSE ANALYZE;
```

---

### 15.5. Trop de Connexions

**Symptôme** :
- Erreur "FATAL: sorry, too many clients already"
- Performances dégradées avec beaucoup de connexions

**Solution** :
1. Implémentez PgBouncer (connection pooler)  
2. Réduisez `max_connections` :
```
max_connections = 100
```

---

### 15.6. `random_page_cost` Non Adapté au Stockage

**Symptôme** :
- Index inutilisés (avec SSD)
- Préférence pour sequential scans même sur petites tables

**Solution** :
```
random_page_cost = 1.1  # Pour SSD
```

---

### 15.7. Authentification MD5

**Symptôme** :
- `password_encryption = md5` ou `pg_hba.conf` utilise `md5`

**Solution** :
```
# postgresql.conf
password_encryption = scram-sha-256

# pg_hba.conf
host all all 0.0.0.0/0 scram-sha-256
```

Puis réinitialisez les mots de passe :
```sql
ALTER USER mon_user PASSWORD 'nouveau_mot_de_passe';
```

---

### 15.8. Logs Non Configurés

**Symptôme** :
- Impossible de diagnostiquer les problèmes
- Pas de visibilité sur les slow queries

**Solution** :
```
logging_collector = on  
log_destination = 'stderr,csvlog'  
log_min_duration_statement = 1000  
log_line_prefix = '%t [%p]: user=%u,db=%d '  
```

---

## 16. Conclusion et Bonnes Pratiques

### 16.1. Récapitulatif : Les 10 Paramètres Critiques

Voici les 10 paramètres à auditer **en priorité** :

1. **`shared_buffers`** : 25% de la RAM (max 16 GB)  
2. **`work_mem`** : RAM / (max_connections × 3)  
3. **`maintenance_work_mem`** : 5-10% de la RAM  
4. **`effective_cache_size`** : 50-75% de la RAM  
5. **`max_wal_size`** : 4-16 GB selon charge d'écriture  
6. **`random_page_cost`** : 1.1 pour SSD, 4.0 pour HDD  
7. **`autovacuum`** : ON (jamais OFF)  
8. **`max_connections`** : 100-200 max, utilisez PgBouncer au-delà  
9. **`password_encryption`** : scram-sha-256 (pas MD5)  
10. **`ssl`** : ON en production

---

### 16.2. Bonnes Pratiques Générales

#### 1. Documentez Votre Configuration

Maintenez un fichier de documentation expliquant :
- Pourquoi chaque paramètre a sa valeur actuelle
- La date de la dernière modification
- Les tests effectués avant/après

#### 2. Testez Avant de Modifier en Production

- Appliquez les changements sur un environnement de test
- Mesurez les performances avant/après
- Validez qu'il n'y a pas de régression

#### 3. Un Changement à la Fois

Ne modifiez jamais plusieurs paramètres simultanément :
- Impossible d'isoler l'impact de chaque changement
- Difficile de revenir en arrière en cas de problème

#### 4. Surveillez Après les Changements

Après toute modification :
- Surveillez les métriques (CPU, RAM, I/O)
- Vérifiez les logs pour détecter des erreurs
- Observez les temps de réponse des requêtes

#### 5. Ajustez Progressivement

Pour les paramètres critiques comme `work_mem` :
- Commencez par doubler la valeur
- Mesurez l'impact
- Répétez jusqu'à trouver l'optimal

#### 6. Gardez des Backups de Configuration

Avant toute modification :
```bash
cp postgresql.conf postgresql.conf.backup.$(date +%Y%m%d)
```

#### 7. Utilisez `ALTER SYSTEM` avec Prudence

`ALTER SYSTEM` écrit dans `postgresql.auto.conf` et écrase `postgresql.conf`.

**Préférez** :
- Modifier directement `postgresql.conf` (tracé dans Git)
- Ou documenter chaque `ALTER SYSTEM` effectué

#### 8. Planifiez les Redémarrages

Certains paramètres nécessitent un redémarrage de PostgreSQL :
- `shared_buffers`  
- `max_connections`  
- `shared_preload_libraries`

Planifiez ces changements lors d'une fenêtre de maintenance.

#### 9. Auditez Régulièrement

Effectuez un audit complet :
- Tous les 3-6 mois pour les systèmes critiques
- Après chaque changement matériel
- Après chaque mise à jour majeure

#### 10. Formez-vous Continuellement

PostgreSQL évolue constamment (PostgreSQL 18 apporte beaucoup de nouveautés).

**Ressources** :
- Documentation officielle : https://www.postgresql.org/docs/18/
- Blog Percona PostgreSQL : https://www.percona.com/blog/tag/postgresql/
- Chaîne YouTube Cybertec : https://www.youtube.com/@PostgreSQL

---

### 16.3. PostgreSQL 18 : Points d'Attention

Lors de l'audit d'une instance PostgreSQL 18, portez une attention particulière aux nouveautés :

#### Nouveaux Paramètres à Configurer

1. **`io_method = 'worker'`** : Activer pour meilleures performances I/O  
2. **`autovacuum_worker_slots`** : Permet l'ajustement dynamique des workers  
3. **`autovacuum_vacuum_max_threshold`** : Plafonne les dead tuples avant VACUUM  
4. **`ssl_tls13_ciphers`** : Configurer pour TLS 1.3

#### Nouvelles Fonctionnalités à Tester

1. **Checksums activés par défaut** : Vérifier la compatibilité  
2. **OAuth 2.0** : Envisager pour authentification moderne  
3. **Virtual Generated Columns** : Évaluer pour remplacer certaines colonnes calculées  
4. **UUIDv7** : Considérer pour nouveaux identifiants

#### Optimisations Automatiques

PostgreSQL 18 intègre plusieurs optimisations automatiques du planificateur :
- Auto-élimination des self-joins
- Optimisation des OR-clauses
- Skip Scan pour index multi-colonnes

Ces optimisations ne nécessitent **aucune configuration**, mais soyez conscient qu'elles peuvent changer les plans de requêtes.

---

### 16.4. Exemple Complet : Configuration Recommandée

Voici une configuration complète pour un serveur PostgreSQL 18 typique :

```
#------------------------------------------------------------------------------
# POSTGRESQL 18 - CONFIGURATION OPTIMISÉE
# Serveur : 16 GB RAM, 8 CPU cores, SSD NVMe
# Cas d'usage : OLTP web application (mixte lecture/écriture)
# Date : 2025-11-21
#------------------------------------------------------------------------------

#------------------------------------------------------------------------------
# MÉMOIRE
#------------------------------------------------------------------------------
shared_buffers = 4GB                  # 25% de 16 GB  
work_mem = 50MB                       # 16GB / (100 connexions × 3)  
maintenance_work_mem = 1GB            # ~6% de 16 GB  
effective_cache_size = 12GB           # 75% de 16 GB  

#------------------------------------------------------------------------------
# PERFORMANCE
#------------------------------------------------------------------------------
random_page_cost = 1.1                # SSD/NVMe  
seq_page_cost = 1.0  
effective_io_concurrency = 300        # NVMe  
max_parallel_workers_per_gather = 4   # 8 cores disponibles  
max_parallel_workers = 8  
max_worker_processes = 8  

#------------------------------------------------------------------------------
# WAL
#------------------------------------------------------------------------------
wal_level = replica  
max_wal_size = 8GB                    # Charge d'écriture modérée  
min_wal_size = 2GB  
checkpoint_timeout = 15min  
checkpoint_completion_target = 0.9  

#------------------------------------------------------------------------------
# CONNEXIONS
#------------------------------------------------------------------------------
max_connections = 100  
superuser_reserved_connections = 3  
listen_addresses = '*'  
port = 5432  

#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------
autovacuum = on  
autovacuum_max_workers = 5  
autovacuum_naptime = 30s  
autovacuum_vacuum_scale_factor = 0.1  
autovacuum_vacuum_threshold = 50  
autovacuum_work_mem = 512MB  

#------------------------------------------------------------------------------
# SÉCURITÉ
#------------------------------------------------------------------------------
password_encryption = scram-sha-256  
ssl = on  
ssl_min_protocol_version = 'TLSv1.2'  
ssl_prefer_server_ciphers = on  

#------------------------------------------------------------------------------
# I/O (PostgreSQL 18)
#------------------------------------------------------------------------------
io_method = 'worker'                   # Nouveau : I/O asynchrone

#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------
logging_collector = on  
log_destination = 'stderr,csvlog'  
log_directory = 'log'  
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'  
log_rotation_age = 1d  
log_rotation_size = 100MB  

log_min_duration_statement = 1000     # 1 seconde  
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '  
log_connections = on  
log_disconnections = on  
log_statement = 'ddl'  

#------------------------------------------------------------------------------
# EXTENSIONS
#------------------------------------------------------------------------------
shared_preload_libraries = 'pg_stat_statements'

#------------------------------------------------------------------------------
# FIN DE LA CONFIGURATION
#------------------------------------------------------------------------------
```

---

### 16.5. Ressources Complémentaires

#### Documentation Officielle

- **PostgreSQL 18 Documentation** : https://www.postgresql.org/docs/18/  
- **Server Configuration** : https://www.postgresql.org/docs/18/runtime-config.html

#### Outils

- **PGTune** : https://pgtune.leopard.in.ua/  
- **pgBadger** : https://github.com/darold/pgbadger  
- **PgBouncer** : https://www.pgbouncer.org/

#### Livres

- *PostgreSQL: Up and Running* par Regina Obe et Leo Hsu  
- *The Art of PostgreSQL* par Dimitri Fontaine

#### Communautés

- **Reddit** : r/PostgreSQL  
- **Mailing lists** : pgsql-general@postgresql.org  
- **Discord** : PostgreSQL Community Server

---

## Conclusion

L'audit de configuration est une étape **essentielle** pour garantir :
- **Performances optimales** de votre instance PostgreSQL  
- **Stabilité** en production  
- **Sécurité** des données  
- **Utilisation efficace** des ressources matérielles

Avec PostgreSQL 18, de nouvelles opportunités d'optimisation apparaissent :
- **I/O asynchrone** pour des performances jusqu'à 3× supérieures  
- **Autovacuum amélioré** avec ajustements dynamiques  
- **Authentification moderne** avec OAuth 2.0  
- **Checksums par défaut** pour meilleure intégrité

**N'oubliez pas** :
1. Auditez régulièrement (tous les 3-6 mois)  
2. Testez avant de modifier en production  
3. Documentez vos changements  
4. Surveillez les impacts après modifications  
5. Formez-vous continuellement

Avec une configuration bien auditée et optimisée, votre PostgreSQL 18 sera :
- ⚡ **Rapide**  
- 🛡️ **Sécurisé**  
- 💪 **Stable**  
- 📈 **Évolutif**

---


⏭️ [Audit d'indexation](/annexes/checklist-performance/02-audit-indexation.md)
