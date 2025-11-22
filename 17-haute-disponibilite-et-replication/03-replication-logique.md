🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.3. Réplication Logique (Logical Replication)

## Introduction Générale

La **réplication logique** est l'une des fonctionnalités les plus puissantes et flexibles de PostgreSQL. Introduite dans PostgreSQL 10 et considérablement améliorée dans les versions suivantes, elle permet de copier et synchroniser des données entre différentes bases PostgreSQL de manière sélective et intelligente.

### Qu'est-ce que la Réplication ?

Avant de plonger dans la réplication logique, commençons par définir ce qu'est la réplication en général.

**La réplication de base de données** est le processus de copie et de synchronisation des données d'une base de données source (appelée **master** ou **primary**) vers une ou plusieurs bases de données destination (appelées **replica** ou **standby**).

#### Pourquoi Répliquer ?

Les raisons principales pour mettre en place de la réplication sont :

1. **Haute Disponibilité (HA)** : Si le serveur principal tombe en panne, un replica peut prendre le relais
2. **Répartition de Charge** : Distribuer les lectures sur plusieurs serveurs pour améliorer les performances
3. **Sauvegarde** : Avoir une copie des données à jour en permanence
4. **Analytics et Reporting** : Décharger les requêtes lourdes vers un serveur dédié
5. **Migration** : Déplacer une base de données vers un nouveau serveur sans interruption
6. **Distribution Géographique** : Placer les données près des utilisateurs pour réduire la latence

### Analogie Simple : La Photocopieuse Intelligente

Imaginez une photocopieuse qui peut :
- **Copier seulement certaines pages** d'un document (pas tout)
- **Masquer certaines parties** sensibles avant de copier
- **Copier en temps réel** chaque nouvelle page ajoutée au document source
- **Envoyer les copies** à plusieurs destinations différentes
- **Adapter le format** selon les besoins de chaque destination

C'est exactement ce que fait la réplication logique avec vos données PostgreSQL !

---

## Réplication Physique vs Réplication Logique

PostgreSQL propose deux types principaux de réplication. Comprendre la différence est essentiel pour choisir la bonne approche.

### Réplication Physique (Streaming Replication)

#### Principe de Fonctionnement

La réplication physique copie les **blocs de données bruts** au niveau du système de fichiers. C'est comme faire une copie bit à bit d'un disque dur.

```
┌─────────────────────────────────────────────┐
│         SERVEUR PRIMARY                     │
│                                             │
│  ┌─────────────────────────────────┐        │
│  │  Bloc physique 1 : 0x3F2A...    │        │
│  │  Bloc physique 2 : 0x8B1C...    │───┐    │
│  │  Bloc physique 3 : 0x5D9E...    │   │    │
│  │  ...                            │   │    │
│  └─────────────────────────────────┘   │    │
│                                        │    │
└────────────────────────────────────────┼────┘
                                         │
                    Copie exacte des blocs (WAL)
                                         │
                                         ▼
┌─────────────────────────────────────────────┐
│         SERVEUR STANDBY                     │
│                                             │
│  ┌─────────────────────────────────┐        │
│  │  Bloc physique 1 : 0x3F2A...    │        │
│  │  Bloc physique 2 : 0x8B1C...    │        │
│  │  Bloc physique 3 : 0x5D9E...    │        │
│  │  ...                            │        │
│  └─────────────────────────────────┘        │
│                                             │
│  ⚠️ EXACTEMENT identique au primary         │
└─────────────────────────────────────────────┘
```

**Caractéristiques** :
- ✅ **Très rapide** : Overhead minimal
- ✅ **Simple** : Configuration facile
- ✅ **Complet** : Tout est répliqué automatiquement (DDL, séquences, extensions)
- ❌ **Tout ou rien** : Impossible de sélectionner des tables spécifiques
- ❌ **Même version** : Le primary et standby doivent avoir la même version majeure
- ❌ **Lecture seule** : Le standby ne peut pas accepter d'écritures

### Réplication Logique (Logical Replication)

#### Principe de Fonctionnement

La réplication logique copie les **changements de données au niveau logique** : lignes insérées, mises à jour, supprimées. C'est comme si quelqu'un lisait les modifications et les reproduisait sur une autre base.

```
┌─────────────────────────────────────────────────────┐
│              SERVEUR SOURCE                         │
│                                                     │
│  Données :                                          │
│  ┌──────────────────────────────────┐               │
│  │ Table: commandes                 │               │
│  │ id | client  | montant | date    │               │
│  │ 1  | Alice   | 100.00  | 2025... │               │
│  │ 2  | Bob     | 250.00  | 2025... │───┐           │
│  │ 3  | Charlie | 75.50   | 2025... │   │           │
│  └──────────────────────────────────┘   │           │
│                                         │           │
└─────────────────────────────────────────┼───────────┘
                                          │
       Décodage logique : "INSERT INTO commandes..."
       Transmission des opérations SQL logiques
                                          │
                                          ▼
┌─────────────────────────────────────────────────────┐
│           SERVEUR DESTINATION                       │
│                                                     │
│  ┌──────────────────────────────────┐               │
│  │ Table: commandes                 │               │
│  │ id | client  | montant | date    │               │
│  │ 1  | Alice   | 100.00  | 2025... │               │
│  │ 2  | Bob     | 250.00  | 2025... │               │
│  │ 3  | Charlie | 75.50   | 2025... │               │
│  └──────────────────────────────────┘               │
│                                                     │
│  ✅ Peut avoir d'autres tables différentes          │
│  ✅ Peut avoir des index différents                 │
│  ✅ Peut accepter des écritures sur tables          │
│      non répliquées                                 │
└─────────────────────────────────────────────────────┘
```

**Caractéristiques** :
- ✅ **Sélectif** : Choisir quelles tables, colonnes, lignes répliquer
- ✅ **Flexible** : Versions PostgreSQL différentes possibles
- ✅ **Écritures possibles** : Le replica peut accepter des écritures sur tables non répliquées
- ✅ **Multi-sources** : Un serveur peut recevoir des données de plusieurs sources
- ⚠️ **Plus lent** : Overhead de 10-20% dû au décodage logique
- ⚠️ **Plus complexe** : Configuration et gestion plus élaborées
- ❌ **DDL manuel** : Les changements de schéma ne sont pas automatiques

### Tableau Comparatif Synthétique

| Critère | Réplication Physique | Réplication Logique |
|---------|---------------------|---------------------|
| **Niveau de copie** | Blocs physiques | Lignes logiques (SQL) |
| **Granularité** | ❌ Tout ou rien | ✅ Sélective (tables, colonnes, lignes) |
| **Versions** | ❌ Identiques requises | ✅ Différentes possibles |
| **Performance** | ✅✅ Excellente | ⚠️ Bonne (overhead) |
| **Simplicité** | ✅✅ Simple | ⚠️ Moyenne |
| **Flexibilité** | ❌ Limitée | ✅✅ Très flexible |
| **Écritures replica** | ❌ Non | ✅ Oui (tables non répliquées) |
| **DDL automatique** | ✅ Oui | ❌ Non |
| **Cas d'usage principal** | HA, Failover | Migration, Analytics, Sélectivité |

---

## Comment Fonctionne la Réplication Logique ?

### Vue d'Ensemble du Mécanisme

La réplication logique s'appuie sur plusieurs composants qui travaillent ensemble :

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVEUR SOURCE (Publisher)               │
│                                                             │
│  1. Application écrit dans la base                          │
│     INSERT INTO commandes VALUES (...)  ──┐                 │
│                                           │                 │
│  2. PostgreSQL écrit dans le WAL          │                 │
│     ┌──────────────────┐                  │                 │
│     │  WAL (Write-Ahead│◄─────────────────┘                 │
│     │  Log)            │                                    │
│     └──────────────────┘                                    │
│              │                                              │
│              ▼                                              │
│  3. Décodeur Logique (pgoutput)                             │
│     Lit le WAL et décode les opérations                     │
│     ┌────────────────────────────┐                          │
│     │ "INSERT INTO commandes..." │                          │
│     └────────────────────────────┘                          │
│              │                                              │
│              ▼                                              │
│  4. Publication                                             │
│     Filtre selon les tables/colonnes configurées            │
│     ┌────────────────┐                                      │
│     │ PUBLICATION    │                                      │
│     │ pub_prod       │                                      │
│     └────────────────┘                                      │
│              │                                              │
└──────────────┼──────────────────────────────────────────────┘
               │
               │ Réseau (connexion PostgreSQL)
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                SERVEUR DESTINATION (Subscriber)             │
│                                                             │
│  5. Subscription                                            │
│     Reçoit les changements                                  │
│     ┌────────────────┐                                      │
│     │ SUBSCRIPTION   │                                      │
│     │ sub_prod       │                                      │
│     └────────────────┘                                      │
│              │                                              │
│              ▼                                              │
│  6. Logical Replication Worker                              │
│     Applique les changements dans les tables locales        │
│     ┌──────────────────────────────┐                        │
│     │ INSERT INTO commandes VALUES │                        │
│     │ (...)                        │                        │
│     └──────────────────────────────┘                        │
│              │                                              │
│              ▼                                              │
│  7. Données synchronisées                                   │
│     ┌────────────┐                                          │
│     │ Table      │                                          │
│     │ commandes  │                                          │
│     └────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### Étapes Détaillées du Processus

#### Étape 1 : Écriture dans la Base Source

```sql
-- Une application effectue une modification
INSERT INTO commandes (client_id, montant, date_commande)
VALUES (123, 99.99, NOW());
```

PostgreSQL enregistre cette opération dans le **WAL (Write-Ahead Log)**, un journal séquentiel de toutes les modifications.

#### Étape 2 : Décodage Logique (Logical Decoding)

Le **décodeur logique** (plugin `pgoutput` par défaut) lit le WAL et transforme les modifications binaires en opérations SQL compréhensibles :

```
WAL binaire : 0x4A2F8B3C1D...

↓ Décodage

Opération logique :
INSERT INTO public.commandes (id, client_id, montant, date_commande)
VALUES (5001, 123, 99.99, '2025-11-22 14:30:00');
```

#### Étape 3 : Filtrage par Publication

La **publication** (créée sur le serveur source) détermine quelles données sont disponibles pour la réplication :

```sql
-- Exemple : Publier seulement certaines tables
CREATE PUBLICATION pub_prod
FOR TABLE commandes, clients, produits;

-- Ou avec filtrage avancé (PostgreSQL 15+)
CREATE PUBLICATION pub_prod_france
FOR TABLE commandes WHERE (pays = 'France');
```

Seules les opérations sur les tables publiées sont transmises.

#### Étape 4 : Transmission Réseau

Les changements sont envoyés via une connexion réseau PostgreSQL standard (port 5432) du publisher au subscriber.

#### Étape 5 : Réception par la Subscription

La **subscription** (créée sur le serveur destination) se connecte à la publication et reçoit les changements :

```sql
-- Sur le serveur destination
CREATE SUBSCRIPTION sub_prod
CONNECTION 'host=source.example.com dbname=production user=replicateur'
PUBLICATION pub_prod;
```

#### Étape 6 : Application des Changements

Un **worker de réplication logique** applique les changements dans les tables locales du subscriber :

```sql
-- Le worker exécute (automatiquement) :
INSERT INTO commandes (id, client_id, montant, date_commande)
VALUES (5001, 123, 99.99, '2025-11-22 14:30:00');
```

#### Étape 7 : Synchronisation Continue

Ce processus se répète en **continu** pour chaque modification, maintenant les deux bases synchronisées en quasi temps réel.

---

## Concepts Clés de la Réplication Logique

### 1. Le WAL (Write-Ahead Log)

Le **WAL** est le journal de toutes les transactions PostgreSQL. C'est le cœur de la réplication logique.

**Principe** : Avant de modifier une donnée sur disque, PostgreSQL écrit d'abord la modification dans le WAL. Cela garantit la durabilité (ACID) et permet la réplication.

```
Transaction :
UPDATE commandes SET statut = 'payee' WHERE id = 100;

┌─────────────────────────────────────────┐
│ 1. Écriture dans le WAL (instantané)    │
│    "UPDATE commandes id=100..."         │
└─────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│ 2. Modification sur disque (différé)    │
│    Mise à jour de la page de données    │
└─────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│ 3. COMMIT : Transaction validée         │
└─────────────────────────────────────────┘
```

Pour la réplication logique, PostgreSQL doit être configuré avec `wal_level = logical`, qui enregistre suffisamment d'informations dans le WAL pour reconstruire les modifications au niveau logique.

### 2. Les Publications (Publisher Side)

Une **publication** est un objet sur le serveur source qui définit quelles données sont disponibles pour la réplication.

**Analogie** : C'est comme créer un "catalogue" ou un "menu" des données que vous êtes prêt à partager.

**Caractéristiques** :
- Créée sur le serveur **source** (publisher)
- Peut contenir une ou plusieurs tables
- Peut filtrer par colonnes (PostgreSQL 15+)
- Peut filtrer par lignes avec WHERE (PostgreSQL 15+)
- Peut spécifier quelles opérations répliquer (INSERT, UPDATE, DELETE, TRUNCATE)

### 3. Les Subscriptions (Subscriber Side)

Une **subscription** est un objet sur le serveur destination qui se connecte à une publication et reçoit les données.

**Analogie** : C'est comme "s'abonner" à un journal ou à une newsletter.

**Caractéristiques** :
- Créée sur le serveur **destination** (subscriber)
- Se connecte à une publication via une chaîne de connexion
- Gère automatiquement la copie initiale des données
- Maintient la synchronisation continue

### 4. Les Slots de Réplication

Un **slot de réplication** est un mécanisme qui garantit qu'aucun changement ne sera perdu.

**Principe** : Le slot "mémorise" jusqu'où le subscriber a lu le WAL, permettant au publisher de conserver les données WAL nécessaires même si le subscriber est temporairement déconnecté.

```
┌─────────────────────────────────────────────────┐
│  WAL du Publisher                               │
│                                                 │
│  [===Déjà lu===][===Slot保持===][===Nouveau===]  │
│                 ▲                               │
│                 │                               │
│            Position du slot                     │
│         (subscriber a lu jusqu'ici)             │
│                                                 │
│  Le WAL avant cette position peut être supprimé │
│  Le WAL après doit être conservé                │
└─────────────────────────────────────────────────┘
```

**⚠️ Important** : Un slot actif peut faire grossir le WAL indéfiniment si le subscriber est arrêté longtemps, risquant de saturer le disque.

### 5. Le Décodeur Logique (Logical Decoding)

Le **décodeur logique** est le composant qui lit le WAL binaire et le transforme en opérations SQL compréhensibles.

PostgreSQL utilise le plugin **pgoutput** par défaut pour la réplication logique, mais d'autres plugins existent pour des cas d'usage spécifiques (comme `wal2json` pour exporter en JSON).

---

## Avantages de la Réplication Logique

### 1. Flexibilité Maximale

**Réplication Sélective** :
```sql
-- Répliquer seulement 3 tables sur 100
CREATE PUBLICATION pub_metier
FOR TABLE commandes, clients, produits;
```

**Filtrage par Colonnes** :
```sql
-- Répliquer sans les colonnes sensibles
CREATE PUBLICATION pub_anonymized
FOR TABLE clients (id, nom, ville, pays);
-- email, telephone, adresse sont exclus
```

**Filtrage par Lignes** :
```sql
-- Répliquer seulement les données d'une région
CREATE PUBLICATION pub_europe
FOR TABLE commandes WHERE (region = 'Europe');
```

### 2. Migration sans Interruption

La réplication logique est idéale pour migrer une base de données sans downtime :

```
Jour J-30 : Configuration de la réplication
         ↓
Jour J-1  : Validation (données synchronisées)
         ↓
Jour J    : Bascule (quelques secondes d'interruption)
         ↓
Jour J+1  : Ancien serveur désactivé
```

### 3. Compatibilité Multi-Versions

Vous pouvez répliquer entre différentes versions de PostgreSQL :

```
PostgreSQL 13  ────→  PostgreSQL 18
  (source)              (destination)

Permet de tester la nouvelle version avec vos données réelles
avant de migrer complètement
```

### 4. Architecture Distribuée

**Scénario : Agrégation Multi-Sources**
```
┌─────────────┐
│ Filiale A   │──┐
└─────────────┘  │
                 │
┌─────────────┐  │    ┌─────────────────┐
│ Filiale B   │──┼───→│  Data Warehouse │
└─────────────┘  │    │    (Central)    │
                 │    └─────────────────┘
┌─────────────┐  │
│ Filiale C   │──┘
└─────────────┘

Chaque filiale publie ses données
Le data warehouse les agrège toutes
```

### 5. Isolation des Charges (OLTP vs OLAP)

```
┌─────────────────┐
│  Base OLTP      │  ← Applications (écritures intensives)
│  (Production)   │
└─────────────────┘
         │
         │ Réplication
         ▼
┌─────────────────┐
│  Base OLAP      │  ← Analytics (lectures lourdes)
│  (Reporting)    │
└─────────────────┘

Les requêtes analytics n'impactent pas la production
```

---

## Cas d'Usage Typiques

### Cas d'Usage 1 : Migration de Base de Données

**Problème** : Migrer d'un ancien serveur vers un nouveau sans interruption.

**Solution avec Réplication Logique** :
1. Configurer la réplication vers le nouveau serveur
2. Attendre la synchronisation complète
3. Basculer l'application (downtime < 1 minute)
4. Valider et désactiver l'ancien serveur

**Avantage** : Zero-downtime migration.

### Cas d'Usage 2 : Environnement de Développement Réaliste

**Problème** : Les développeurs ont besoin de données réalistes mais sans données sensibles.

**Solution** :
```sql
-- Répliquer seulement les données non sensibles
CREATE PUBLICATION pub_dev_safe
FOR TABLE produits, categories, fournisseurs;
-- clients, commandes, paiements NE SONT PAS répliqués

-- Ou avec anonymisation (via triggers sur la destination)
```

### Cas d'Usage 3 : Conformité RGPD

**Problème** : Les données européennes doivent rester en Europe.

**Solution** :
```sql
-- Serveur EU : Répliquer seulement les données EU
CREATE PUBLICATION pub_eu_only
FOR TABLE commandes WHERE (pays IN ('FR', 'DE', 'IT', 'ES'));

-- Les données des autres régions ne quittent jamais leur zone
```

### Cas d'Usage 4 : Analytics et Business Intelligence

**Problème** : Les requêtes analytics ralentissent la production.

**Solution** : Base OLAP dédiée pour le reporting
- Réplication des tables de faits et dimensions
- Agrégations et vues matérialisées sur le serveur analytics
- Zéro impact sur la production

### Cas d'Usage 5 : Upgrade de Version PostgreSQL

**Problème** : Passer de PostgreSQL 13 à 18 sans risque.

**Solution** :
1. Configurer réplication 13 → 18
2. Tester l'application contre PG 18
3. Valider les performances
4. Basculer en production
5. Rollback facile si problème (retour vers PG 13)

---

## Prérequis et Configuration Initiale

### Configuration Serveur Source (Publisher)

#### 1. Paramètres PostgreSQL (`postgresql.conf`)

```ini
# Niveau de WAL requis pour la réplication logique
wal_level = logical

# Nombre maximum de slots de réplication
# (au moins 1 par subscription)
max_replication_slots = 10

# Nombre de processus WAL sender
# (au moins 1 par subscription active)
max_wal_senders = 10

# Optionnel : Limite de rétention WAL (PostgreSQL 13+)
max_slot_wal_keep_size = 50GB
```

**⚠️ Important** : Modifier `wal_level` nécessite un **redémarrage** de PostgreSQL.

#### 2. Authentification (`pg_hba.conf`)

```
# Autoriser la connexion pour la réplication
# depuis les serveurs subscribers
host    replication    replicateur    10.0.0.0/8    scram-sha-256
host    production     replicateur    10.0.0.0/8    scram-sha-256
```

#### 3. Utilisateur de Réplication

```sql
-- Créer un utilisateur dédié avec privilèges appropriés
CREATE ROLE replicateur WITH LOGIN REPLICATION PASSWORD 'mot_de_passe_securise';

-- Accorder les permissions sur les tables à répliquer
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicateur;
GRANT USAGE ON SCHEMA public TO replicateur;
```

### Configuration Serveur Destination (Subscriber)

#### 1. Paramètres PostgreSQL (`postgresql.conf`)

```ini
# Nombre de workers pour appliquer les changements
max_logical_replication_workers = 4

# Nombre de workers par subscription
max_sync_workers_per_subscription = 2

# Workers en parallèle (PostgreSQL 16+)
max_parallel_apply_workers_per_subscription = 2
```

#### 2. Structure de Base

Les **tables doivent exister** sur le subscriber avant de créer la subscription, avec :
- Même nom de table
- Mêmes colonnes (ou un sous-ensemble si filtrage)
- **Clés primaires obligatoires** sur les tables répliquées

```sql
-- Exemple : Créer la même structure
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INT NOT NULL,
    montant NUMERIC(10,2),
    date_commande TIMESTAMP DEFAULT NOW()
);
```

---

## Architecture de Référence

### Architecture Simple : Master-Replica

```
┌──────────────────────────────────────────────┐
│         Serveur MASTER (Publisher)           │
│                                              │
│  • Écritures : ✅ Autorisées                 │
│  • Lectures  : ✅ Autorisées                 │
│                                              │
│  Applications écrivent ici                   │
└──────────────────────────────────────────────┘
                    │
                    │ Réplication Logique
                    │ (Unidirectionnelle)
                    ▼
┌──────────────────────────────────────────────┐
│         Serveur REPLICA (Subscriber)         │
│                                              │
│  • Écritures : ❌ Interdites (sur tables     │
│                   répliquées)                │
│  • Lectures  : ✅ Autorisées                 │
│                                              │
│  Analytics, Reporting lisent ici             │
└──────────────────────────────────────────────┘
```

### Architecture Avancée : Multi-Sources

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Source A   │   │  Source B   │   │  Source C   │
│  (Région 1) │   │  (Région 2) │   │  (Région 3) │
└─────────────┘   └─────────────┘   └─────────────┘
       │                 │                 │
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                         │ 3 Subscriptions
                         ▼
              ┌──────────────────┐
              │  Data Warehouse  │
              │    (Central)     │
              └──────────────────┘

Agrégation de données depuis plusieurs sources
```

### Architecture Cloud-Native

```
┌────────────────────────────────────────────┐
│   On-Premise Data Center                   │
│   PostgreSQL 15                            │
└────────────────────────────────────────────┘
                    │
                    │ Migration progressive
                    ▼
┌────────────────────────────────────────────┐
│   AWS RDS PostgreSQL 18                    │
│   (ou Azure, GCP)                          │
└────────────────────────────────────────────┘

Transition cloud avec réplication logique
```

---

## Monitoring et Observabilité

### Vues Système Essentielles

#### 1. État des Publications (Serveur Source)

```sql
-- Lister toutes les publications
SELECT * FROM pg_publication;

-- Voir les tables publiées
SELECT * FROM pg_publication_tables;

-- Détails d'une publication
SELECT
    pubname,
    pubinsert,
    pubupdate,
    pubdelete,
    pubtruncate
FROM pg_publication
WHERE pubname = 'pub_prod';
```

#### 2. État des Subscriptions (Serveur Destination)

```sql
-- Lister toutes les subscriptions
SELECT
    subname,
    subenabled,
    subconninfo,
    subslotname,
    subpublications
FROM pg_subscription;

-- État de réplication en temps réel
SELECT
    subname,
    pid,
    received_lsn,
    latest_end_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    pg_size_pretty(
        pg_wal_lsn_diff(latest_end_lsn, received_lsn)
    ) AS replication_lag
FROM pg_stat_subscription;
```

#### 3. Slots de Réplication (Serveur Source)

```sql
-- Surveiller les slots
SELECT
    slot_name,
    slot_type,
    database,
    active,
    restart_lsn,
    confirmed_flush_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS wal_retained
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

### Métriques Clés à Surveiller

| Métrique | Seuil Recommandé | Signification |
|----------|------------------|---------------|
| **Lag de réplication** | < 5 secondes | Délai entre source et destination |
| **WAL retenu par slot** | < 10 GB | Espace disque consommé |
| **Subscription active** | true | Réplication fonctionnelle |
| **Workers actifs** | > 0 | Processus de réplication en cours |
| **Erreurs dans les logs** | 0 | Problèmes de réplication |

### Dashboard de Monitoring Recommandé

```
┌────────────────────────────────────────────┐
│   PostgreSQL Logical Replication Monitor   │
├────────────────────────────────────────────┤
│                                            │
│  Replication Lag:  [====    ] 2.3 sec      │
│  WAL Retained:     [==      ] 3.2 GB       │
│  Active Workers:   ✅ 4/4                  │
│  Subscription:     ✅ ACTIVE               │
│                                            │
│  Last Update:      2025-11-22 14:35:00     │
│  Status:           🟢 HEALTHY              │
│                                            │
└────────────────────────────────────────────┘
```

---

## Considérations Importantes

### Performances

**Impact sur le Serveur Source** :
- Overhead CPU : ~10-20% pour le décodage logique
- Overhead I/O : WAL retention et lecture
- Bande passante réseau : Dépend du volume de transactions

**Impact sur le Serveur Destination** :
- CPU : Application des changements
- I/O : Écritures dans les tables
- Latence typique : 1-5 secondes en conditions normales

### Sécurité

**Bonnes Pratiques** :
- Utiliser SSL/TLS pour les connexions (`sslmode=require`)
- Créer un utilisateur dédié avec privilèges minimaux
- Restreindre les IPs dans `pg_hba.conf`
- Utiliser SCRAM-SHA-256 pour l'authentification (pas MD5)
- PostgreSQL 18 : Support OAuth 2.0 pour authentification moderne

### Limitations à Connaître

**Ce qui N'est PAS répliqué** :
- ❌ Modifications de schéma (DDL)
- ❌ Valeurs de séquences
- ❌ Large Objects (BLOB via `pg_largeobject`)
- ❌ Tables sans clé primaire (par défaut)
- ❌ Tables temporaires et non journalisées

Ces limitations seront détaillées dans la section suivante.

---

## Nouveautés PostgreSQL 18 pour la Réplication Logique

PostgreSQL 18 (septembre 2025) apporte des améliorations significatives :

### 1. Performances Améliorées

- **Décodage logique optimisé** : Réduction de l'overhead CPU
- **Parallélisation étendue** : Meilleure utilisation des workers
- **Compression améliorée** : Réduction de la bande passante réseau

### 2. Nouvelles Fonctionnalités

- **Authentification OAuth 2.0** : Intégration moderne des identités
- **Statistiques étendues** : Nouvelles métriques dans `pg_stat_subscription`
- **Gestion améliorée des conflits** : Nouvelles options de résolution

### 3. Fiabilité

- **Data Checksums par défaut** : Détection de corruption automatique
- **Améliorations du décodage WAL** : Moins d'erreurs en cas de charge élevée

---

## Conclusion de l'Introduction

La **réplication logique** est un outil puissant qui offre une flexibilité inégalée pour :
- Migrer des bases de données sans interruption
- Créer des architectures distribuées sophistiquées
- Isoler les charges OLTP et OLAP
- Respecter les contraintes réglementaires (RGPD)
- Tester de nouvelles versions PostgreSQL en toute sécurité

Contrairement à la réplication physique qui est "tout ou rien", la réplication logique vous permet de **choisir précisément** quelles données répliquer, vers quelles destinations, et comment.

Dans les sections suivantes, nous allons explorer en détail :
- **17.3.1** : Publications et Subscriptions (mécanismes fondamentaux)
- **17.3.2** : Cas d'usage concrets (migrations, réplication sélective)
- **17.3.3** : Limitations et considérations (ce qu'il faut savoir)

---


⏭️ [Publications et Subscriptions](/17-haute-disponibilite-et-replication/03.1-publications-subscriptions.md)
