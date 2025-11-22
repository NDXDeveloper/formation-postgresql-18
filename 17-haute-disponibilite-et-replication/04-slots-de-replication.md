🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.4. Slots de réplication : Physical vs Logical

## Introduction

Les **slots de réplication** sont l'un des mécanismes les plus importants mais souvent mal compris de PostgreSQL. Ils jouent un rôle crucial dans la haute disponibilité et la réplication des données.

Dans ce chapitre, nous allons démystifier les slots de réplication, comprendre leur fonctionnement interne, et explorer les différences fondamentales entre les slots physiques et logiques.

---

## 1. Qu'est-ce qu'un slot de réplication ?

### 1.1. Le problème sans les slots

Imaginons une situation de réplication avant l'existence des slots :

```
Serveur Principal              Serveur Standby
     │                              │
     │ Génère WAL                   │
     │ (fichiers 001, 002, 003)     │
     │                              │
     ├──────── 001 ────────────────>│ Reçu et appliqué
     ├──────── 002 ────────────────>│ Reçu et appliqué
     │                              │
     │ Checkpoint !                 │ (Standby déconnecté
     │ → Supprime 001, 002          │  temporairement)
     │                              │
     ├──────── 003 ────────────────>│
     │                              │
     │                              │ Reconnexion
     │                              │ "Je veux 001 et 002"
     │                              │
     X  Fichiers supprimés !        │ ❌ ERREUR
```

**Le problème :**
Le serveur principal ne sait pas que le standby a besoin des anciens fichiers WAL. Il les supprime après un checkpoint, pensant qu'ils ne sont plus nécessaires.

**La conséquence :**
- Le standby ne peut plus se resynchroniser
- Il faut reconstruire entièrement le standby (pg_basebackup)
- Perte de temps et risque pour la haute disponibilité

### 1.2. La solution : Les slots de réplication

Un **slot de réplication** est comme un **marque-page** que le standby place sur le serveur principal :

> "Ne supprime aucun fichier WAL au-delà de ce point, j'en ai encore besoin !"

**Avec un slot :**
```
Serveur Principal              Serveur Standby
     │                              │
     │ Slot: "standby1"             │
     │ Position: WAL 001            │
     │                              │
     ├──────── 001 ────────────────>│ Reçu et appliqué
     ├──────── 002 ────────────────>│ Reçu et appliqué
     │                              │
     │ Checkpoint !                 │ (Standby déconnecté)
     │ Vérifie slot "standby1"      │
     │ → Conserve 001, 002          │
     │ (car le slot dit "j'en ai    │
     │  besoin")                    │
     │                              │
     ├──────── 003 ────────────────>│
     │                              │
     │                              │ Reconnexion
     │<────── "Reprends à 002" ─────┤
     ├──────── 002 ────────────────>│ ✅ Succès !
```

### 1.3. Définition formelle

Un **slot de réplication** est un objet persistant sur le serveur principal qui :

1. **Garde une trace** de la position de réplication d'un consommateur (standby ou abonnement logique)
2. **Empêche la suppression** des fichiers WAL nécessaires à ce consommateur
3. **Survit aux redémarrages** du serveur PostgreSQL
4. **Est identifié par un nom unique**

---

## 2. Les deux types de slots : Physical vs Logical

PostgreSQL propose deux types de slots de réplication, chacun avec des objectifs différents.

### 2.1. Vue d'ensemble comparative

| Caractéristique | Slot Physique | Slot Logique |
|----------------|---------------|--------------|
| **Type de réplication** | Réplication physique (binaire) | Réplication logique (SQL) |
| **Niveau de granularité** | Instance entière | Tables sélectionnées |
| **Contenu répliqué** | Blocs de données bruts | Changements logiques (INSERT/UPDATE/DELETE) |
| **Format** | Binaire (dépendant de l'architecture) | Indépendant de l'architecture |
| **Cas d'usage** | Haute disponibilité, Disaster Recovery | Migrations, Réplication sélective, Multi-master |
| **Overhead** | Faible | Modéré à élevé |
| **Requiert wal_level** | `replica` | `logical` |
| **Transformations possibles** | Non | Oui (filtres, mappings) |

### 2.2. Analogie pour comprendre la différence

**Slot Physique (Physical Slot) :**
> C'est comme copier un disque dur secteur par secteur. Vous obtenez une **copie bit-à-bit exacte** du disque, y compris le système de fichiers, la structure interne, tout. Le disque copié doit fonctionner sur la même architecture matérielle.

**Slot Logique (Logical Slot) :**
> C'est comme exporter des fichiers d'un ordinateur et les importer dans un autre. Vous ne copiez pas la structure du disque, mais seulement le **contenu utile** (les fichiers). Vous pouvez même les importer sur un système d'exploitation différent ou dans une structure de dossiers différente.

---

## 3. Slots de Réplication Physique

### 3.1. Principe de fonctionnement

Les slots physiques sont utilisés dans le cadre de la **réplication en streaming** (Streaming Replication).

#### Architecture

```
┌─────────────────────────────────┐
│   Serveur Principal (Primary)   │
│                                 │
│  ┌────────────────────────────┐ │
│  │  WAL Stream                │ │
│  │  (flux binaire)            │ │
│  └───────────┬────────────────┘ │
│              │                  │
│  ┌───────────▼────────────────┐ │
│  │  Slot: "replica1"          │ │
│  │  Type: Physical            │ │
│  │  Position: 0/3000A48       │ │
│  └────────────────────────────┘ │
└──────────────┬──────────────────┘
               │ Streaming WAL
               │ (binaire)
               ▼
┌──────────────────────────────────┐
│   Serveur Standby (Replica)      │
│                                  │
│  Applique les changements        │
│  binaires → Copie exacte         │
└──────────────────────────────────┘
```

#### Caractéristiques détaillées

1. **Réplication bit-à-bit**
   - Chaque bloc de données modifié est répliqué tel quel
   - Le standby est une copie **byte-perfect** du primary
   - Même structure de fichiers, même layout physique

2. **Dépendance à l'architecture**
   - Primary et Standby doivent avoir :
     - Même version majeure de PostgreSQL
     - Même architecture CPU (x86_64, ARM, etc.)
     - Même endianness (little-endian / big-endian)

3. **Réplication complète**
   - **Toutes** les bases de données sont répliquées
   - **Toutes** les tables de chaque base sont répliquées
   - Impossible de filtrer ou sélectionner

### 3.2. Création d'un slot physique

#### Méthode 1 : Via SQL

```sql
-- Sur le serveur PRIMARY
SELECT pg_create_physical_replication_slot('replica1_slot');
```

**Résultat :**
```
 pg_create_physical_replication_slot
-------------------------------------
 (replica1_slot,0/3000A48)
```

#### Méthode 2 : Via pg_basebackup

```bash
# Lors de la création initiale du standby
pg_basebackup -h primary_host -D /var/lib/postgresql/14/standby \
  -U replication_user -v -P --wal-method=stream \
  --slot=replica1_slot
```

**Avantages de cette méthode :**
- Crée automatiquement le slot
- Garantit la cohérence entre basebackup et slot
- Une seule commande pour tout configurer

### 3.3. Configuration du standby pour utiliser le slot

Dans `postgresql.auto.conf` ou `recovery.conf` (versions anciennes) du standby :

```ini
# Connexion au primary
primary_conninfo = 'host=primary_host port=5432 user=replication_user password=secret'

# Utilisation du slot
primary_slot_name = 'replica1_slot'
```

**Redémarrage du standby :**
```bash
sudo systemctl restart postgresql
```

### 3.4. Vérification et monitoring

#### Voir les slots existants

```sql
SELECT
    slot_name,
    slot_type,
    database,
    active,
    restart_lsn,
    confirmed_flush_lsn
FROM pg_replication_slots;
```

**Exemple de résultat :**
```
   slot_name    | slot_type | database | active | restart_lsn | confirmed_flush_lsn
----------------+-----------+----------+--------+-------------+---------------------
 replica1_slot  | physical  |          | t      | 0/3000A48   |
```

**Interprétation des colonnes :**
- `slot_name` : Nom du slot
- `slot_type` : `physical` ou `logical`
- `active` : `t` (true) si un client est connecté
- `restart_lsn` : Position WAL minimale à conserver
- `confirmed_flush_lsn` : Pour les slots logiques (position confirmée)

#### Vérifier le retard de réplication

```sql
-- Sur le PRIMARY
SELECT
    slot_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag_size
FROM pg_replication_slots
WHERE slot_type = 'physical';
```

**Exemple de résultat :**
```
   slot_name    | lag_bytes |  lag_size
----------------+-----------+------------
 replica1_slot  |  16777216 | 16 MB
```

**Interprétation :**
- `lag_bytes = 0` : Le standby est parfaitement à jour
- `lag_bytes > 0` : Le standby est en retard de X octets de WAL

### 3.5. Avantages des slots physiques

✅ **Performance optimale**
- Overhead minimal sur le primary
- Transfert binaire direct, pas de décodage

✅ **Simplicité**
- Configuration minimale
- Pas de complexité liée au décodage logique

✅ **Fiabilité garantie**
- Le WAL nécessaire ne sera jamais supprimé
- Pas de risque de rupture de réplication due à la suppression de WAL

✅ **Cohérence totale**
- Le standby est une copie exacte, bit par bit

### 3.6. Inconvénients et risques

⚠️ **Risque de saturation du disque**
- Si le standby est déconnecté longtemps, le WAL s'accumule
- Peut remplir le répertoire `pg_wal/` et bloquer le primary
- **Critique** : Nécessite une surveillance active

⚠️ **Pas de granularité**
- Tout ou rien : impossible de répliquer seulement certaines tables
- Toutes les bases de données sont répliquées

⚠️ **Contraintes d'architecture**
- Primary et Standby doivent être identiques (version, architecture)
- Pas de transformation possible

---

## 4. Slots de Réplication Logique

### 4.1. Principe de fonctionnement

Les slots logiques décodent le WAL binaire en **changements logiques** compréhensibles.

#### Architecture

```
┌─────────────────────────────────────────┐
│   Serveur PRIMARY (Publisher)           │
│                                         │
│  ┌────────────────────────────────────┐ │
│  │  WAL Stream (binaire)              │ │
│  └────────────┬───────────────────────┘ │
│               │                         │
│               ▼                         │
│  ┌────────────────────────────────────┐ │
│  │  Logical Decoding                  │ │
│  │  (décodage du WAL)                 │ │
│  └────────────┬───────────────────────┘ │
│               │                         │
│               ▼                         │
│  ┌────────────────────────────────────┐ │
│  │  Changements logiques :            │ │
│  │  INSERT INTO users (id, name)...   │ │
│  │  UPDATE products SET price=...     │ │
│  │  DELETE FROM orders WHERE...       │ │
│  └────────────┬───────────────────────┘ │
│               │                         │
│  ┌────────────▼───────────────────────┐ │
│  │  Slot: "subscriber1"               │ │
│  │  Type: Logical                     │ │
│  │  Plugin: pgoutput                  │ │
│  │  Position: 0/4A2B8C0               │ │
│  └────────────────────────────────────┘ │
└───────────────┬─────────────────────────┘
                │ Changements SQL
                │ (format logique)
                ▼
┌────────────────────────────────────────┐
│   Serveur SUBSCRIBER                   │
│                                        │
│  Applique les changements SQL :        │
│  - Peut transformer les données        │
│  - Peut filtrer (certaines tables)     │
│  - Peut mapper vers schéma différent   │
└────────────────────────────────────────┘
```

#### Caractéristiques détaillées

1. **Décodage logique (Logical Decoding)**
   - Le WAL binaire est décodé en opérations SQL compréhensibles
   - Les changements sont exprimés au niveau des lignes (tuples)
   - Format : INSERT, UPDATE, DELETE avec valeurs

2. **Indépendance architecturale**
   - Publisher et Subscriber peuvent avoir :
     - Versions PostgreSQL différentes (dans certaines limites)
     - Architectures CPU différentes
     - Systèmes d'exploitation différents

3. **Réplication sélective**
   - Choix des tables à répliquer (publications)
   - Filtrage possible au niveau des lignes (row filters)
   - Transformations possibles via triggers

### 4.2. Prérequis : wal_level = logical

Les slots logiques nécessitent un niveau de détail supérieur dans le WAL.

**Configuration requise dans `postgresql.conf` :**
```ini
wal_level = logical
```

**⚠️ Attention :** Modifier `wal_level` nécessite un **redémarrage** de PostgreSQL.

**Vérification :**
```sql
SHOW wal_level;
```

**Résultat attendu :**
```
 wal_level
-----------
 logical
```

### 4.3. Création d'un slot logique

#### Méthode 1 : Création manuelle (pour développement/test)

```sql
-- Sur le serveur qui publiera les données
SELECT pg_create_logical_replication_slot('my_logical_slot', 'pgoutput');
```

**Paramètres :**
- `'my_logical_slot'` : Nom du slot (unique)
- `'pgoutput'` : Plugin de décodage (standard depuis PG 10)

**Résultat :**
```
 pg_create_logical_replication_slot
------------------------------------
 (my_logical_slot,0/4A2B8C0)
```

#### Méthode 2 : Création automatique via PUBLICATION/SUBSCRIPTION

**Sur le Publisher :**
```sql
-- Créer une publication (quelles tables publier)
CREATE PUBLICATION my_publication FOR TABLE users, orders;
```

**Sur le Subscriber :**
```sql
-- Créer une souscription (crée automatiquement le slot)
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host dbname=mydb user=replication_user password=secret'
    PUBLICATION my_publication
    WITH (create_slot = true, slot_name = 'my_logical_slot');
```

**Avantage :** Tout est géré automatiquement (création du slot, synchronisation initiale).

### 4.4. Plugins de décodage logique

PostgreSQL supporte plusieurs plugins pour décoder le WAL :

#### 1. **pgoutput** (Recommandé, par défaut)
- Plugin natif depuis PostgreSQL 10
- Utilisé par la réplication logique native
- Format optimisé pour la performance
- Support complet des types de données PostgreSQL

#### 2. **test_decoding** (Développement/Debug)
- Plugin de test fourni avec PostgreSQL
- Sortie texte lisible par l'humain
- Utile pour comprendre le contenu du WAL
- **Ne pas utiliser en production**

**Exemple d'utilisation :**
```sql
-- Créer un slot avec test_decoding
SELECT pg_create_logical_replication_slot('debug_slot', 'test_decoding');

-- Lire les changements (sortie texte)
SELECT * FROM pg_logical_slot_get_changes('debug_slot', NULL, NULL);
```

**Sortie exemple :**
```
    lsn     |  xid  |                                data
------------+-------+--------------------------------------------------------------------
 0/4A2B8C0  | 1234  | BEGIN 1234
 0/4A2B8D8  | 1234  | table public.users: INSERT: id[integer]:42 name[text]:'Alice'
 0/4A2B9A0  | 1234  | COMMIT 1234
```

#### 3. **wal2json** (Tiers)
- Plugin tiers populaire
- Sortie au format JSON
- Facilite l'intégration avec des systèmes externes
- Utilisé pour Change Data Capture (CDC)

#### 4. **Debezium (CDC platform)**
- Plateforme complète de Change Data Capture
- Utilise un plugin PostgreSQL dédié
- Streaming vers Kafka, Kinesis, etc.
- Cas d'usage : Event Sourcing, CQRS, Analytics temps réel

### 4.5. Monitoring des slots logiques

#### Vue complète des slots

```sql
SELECT
    slot_name,
    plugin,
    slot_type,
    database,
    active,
    restart_lsn,
    confirmed_flush_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_size
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

#### Statistiques avancées (PostgreSQL 14+)

```sql
SELECT
    slot_name,
    spill_txns,      -- Transactions écrites sur disque (trop volumineuses)
    spill_count,     -- Nombre d'écritures sur disque
    spill_bytes,     -- Octets écrits sur disque
    total_txns,      -- Total de transactions décodées
    total_bytes      -- Total d'octets décodés
FROM pg_stat_replication_slots
WHERE slot_type = 'logical';
```

**Interprétation :**
- `spill_txns > 0` : Des transactions sont trop volumineuses pour rester en mémoire
- `spill_bytes` élevé : Considérer d'augmenter `logical_decoding_work_mem`

### 4.6. Avantages des slots logiques

✅ **Flexibilité architecturale**
- Versions PostgreSQL différentes
- Architectures matérielles différentes
- Migrations en ligne facilitées

✅ **Granularité fine**
- Réplication sélective (tables spécifiques)
- Filtrage au niveau des lignes (row filters)
- Possibilité de transformer les données

✅ **Intégration multi-systèmes**
- Réplication vers des bases non-PostgreSQL
- CDC vers systèmes de messagerie (Kafka)
- Synchronisation multi-datacenter

✅ **Cas d'usage avancés**
- Event Sourcing
- CQRS (Command Query Responsibility Segregation)
- Analytics temps réel
- Audit trail

### 4.7. Inconvénients et limitations

⚠️ **Overhead significatif**
- Le décodage logique consomme CPU et mémoire
- Impact sur les performances du publisher (5-15% typique)
- Génération de plus de WAL qu'avec la réplication physique

⚠️ **Limites fonctionnelles**
- Ne réplique pas le DDL (CREATE TABLE, ALTER, etc.) automatiquement
- Ne réplique pas les SEQUENCES (doivent être gérées manuellement)
- Ne réplique pas les LARGE OBJECTS
- Limitations sur certains types (UNLOGGED tables)

⚠️ **Complexité accrue**
- Configuration plus complexe (publications, souscriptions)
- Gestion des conflits de réplication
- Résolution des schémas divergents

⚠️ **Risques de rétention WAL**
- Comme les slots physiques, peut remplir `pg_wal/`
- Particulièrement dangereux si un subscriber est inactif

---

## 5. Gestion et Maintenance des Slots

### 5.1. Supprimer un slot

#### Slot inactif

```sql
-- Suppression simple (si le slot n'est pas utilisé)
SELECT pg_drop_replication_slot('slot_name');
```

#### Slot actif (client connecté)

```sql
-- 1. Identifier le processus utilisant le slot
SELECT
    slot_name,
    active_pid,
    client_addr,
    state
FROM pg_replication_slots
JOIN pg_stat_replication ON pg_replication_slots.active_pid = pg_stat_replication.pid
WHERE slot_name = 'slot_name';

-- 2. Terminer la connexion du client
SELECT pg_terminate_backend(active_pid);

-- 3. Supprimer le slot
SELECT pg_drop_replication_slot('slot_name');
```

**⚠️ Attention :** Supprimer un slot actif peut casser la réplication. Assurez-vous que c'est intentionnel.

### 5.2. Avancer manuellement un slot (PostgreSQL 11+)

Parfois, un slot peut être bloqué sur une position ancienne. Vous pouvez forcer son avancement.

#### Slots physiques

```sql
SELECT pg_replication_slot_advance('slot_name', 'target_lsn');
```

**Exemple :**
```sql
-- Avancer jusqu'à la position WAL actuelle
SELECT pg_replication_slot_advance('replica1_slot', pg_current_wal_lsn());
```

#### Slots logiques

```sql
-- Même syntaxe
SELECT pg_replication_slot_advance('logical_slot', 'target_lsn');
```

**⚠️ Danger :** Avancer un slot signifie **abandonner** les données entre l'ancienne et la nouvelle position. Le replica/subscriber manquera ces changements.

**Cas d'usage légitime :**
- Récupération d'urgence quand un slot bloque le système
- Migration planifiée où les données manquantes seront resynchronisées autrement

### 5.3. Changer les paramètres d'un slot logique (PostgreSQL 13+)

```sql
-- Modifier le plugin d'un slot logique
SELECT pg_alter_replication_slot('slot_name', ARRAY['option', 'value']);
```

**Exemple :**
```sql
-- Activer l'envoi des valeurs binaires (plus compact)
ALTER SUBSCRIPTION my_subscription SET (binary = true);
```

### 5.4. Copier un slot (PostgreSQL 14+)

Utile pour créer un nouveau standby sans perturber un existant.

```sql
SELECT pg_copy_physical_replication_slot('source_slot', 'new_slot');
```

**Cas d'usage :**
- Ajouter un nouveau replica sans impacter l'existant
- Tests de failover sans risque

---

## 6. Surveillance Proactive et Alertes

### 6.1. Métriques critiques à surveiller

#### 1. Retard de réplication (Lag)

```sql
-- Retard en octets et en taille lisible
SELECT
    slot_name,
    active,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag_size
FROM pg_replication_slots;
```

**Seuils d'alerte recommandés :**
- ⚠️ Warning : lag > 500 MB
- 🚨 Critical : lag > 2 GB

#### 2. Taille du répertoire pg_wal/

```sql
-- Taille actuelle de pg_wal/
SELECT
    pg_size_pretty(sum(size)) AS wal_size
FROM pg_ls_waldir();
```

**Seuils d'alerte :**
- ⚠️ Warning : pg_wal/ > 5 GB
- 🚨 Critical : pg_wal/ > 10 GB (ou 80% de l'espace disque disponible)

#### 3. Slots inactifs depuis longtemps

```sql
-- Identifier les slots inactifs
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots
WHERE active = false
ORDER BY lag_bytes DESC;
```

**Action :** Investiguer pourquoi le slot est inactif et envisager sa suppression après validation.

### 6.2. Configuration d'alertes automatiques

#### Exemple avec Prometheus + Alertmanager

**postgres_exporter query :**
```yaml
pg_replication_slot_lag:
  query: |
    SELECT
      slot_name,
      pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
    FROM pg_replication_slots
  metrics:
    - slot_name:
        usage: "LABEL"
        description: "Name of the replication slot"
    - lag_bytes:
        usage: "GAUGE"
        description: "Replication lag in bytes"
```

**Alerte Prometheus :**
```yaml
groups:
  - name: postgresql_replication
    rules:
      - alert: ReplicationSlotLagging
        expr: pg_replication_slot_lag{} > 500000000  # 500 MB
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication slot {{ $labels.slot_name }} is lagging"
          description: "Lag: {{ humanize $value }} bytes"

      - alert: ReplicationSlotInactive
        expr: pg_replication_slot_active == 0
        for: 15m
        labels:
          severity: critical
        annotations:
          summary: "Replication slot {{ $labels.slot_name }} is inactive"
```

### 6.3. Script de monitoring shell

```bash
#!/bin/bash
# check_replication_slots.sh

PGHOST="localhost"
PGUSER="postgres"
LAG_WARNING_MB=500
LAG_CRITICAL_MB=2000

# Vérifier le lag de tous les slots
psql -h "$PGHOST" -U "$PGUSER" -t -A -c "
  SELECT
    slot_name || '|' ||
    COALESCE(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) / 1024 / 1024, 0) || '|' ||
    active
  FROM pg_replication_slots;
" | while IFS='|' read -r slot_name lag_mb active; do

  if [ "$active" = "f" ]; then
    echo "WARNING: Slot $slot_name is INACTIVE"
  fi

  lag_mb_int=$(printf "%.0f" "$lag_mb")

  if [ "$lag_mb_int" -ge "$LAG_CRITICAL_MB" ]; then
    echo "CRITICAL: Slot $slot_name lag is ${lag_mb}MB (threshold: ${LAG_CRITICAL_MB}MB)"
    exit 2
  elif [ "$lag_mb_int" -ge "$LAG_WARNING_MB" ]; then
    echo "WARNING: Slot $slot_name lag is ${lag_mb}MB (threshold: ${LAG_WARNING_MB}MB)"
    exit 1
  fi
done

echo "OK: All replication slots are healthy"
exit 0
```

**Utilisation avec cron :**
```bash
# Exécuter toutes les 5 minutes
*/5 * * * * /usr/local/bin/check_replication_slots.sh || mail -s "PG Replication Alert" admin@example.com
```

---

## 7. Patterns et Architectures Avancées

### 7.1. Combinaison Physical + Logical Slots

Il est possible (et parfois nécessaire) d'avoir les deux types de slots simultanément.

**Cas d'usage :**
```
Primary (Production)
    │
    ├─── Physical Slot → Hot Standby (HA)
    │                     (copie exacte pour failover)
    │
    └─── Logical Slot  → Analytics DB
                          (réplication sélective pour BI)
```

**Configuration :**
```sql
-- Sur le Primary
CREATE ROLE replication_physical WITH REPLICATION LOGIN PASSWORD 'pass1';
CREATE ROLE replication_logical WITH REPLICATION LOGIN PASSWORD 'pass2';

-- Slot physique pour le standby HA
SELECT pg_create_physical_replication_slot('ha_standby_slot');

-- Slot logique pour l'analytics
CREATE PUBLICATION analytics_pub FOR TABLE sales, customers, products;
```

### 7.2. Cascading Replication avec Slots

Les slots permettent des architectures en cascade.

```
Primary
   │ physical slot: "standby1"
   ▼
Standby 1 (cascade_mode)
   │ physical slot: "standby2"
   ▼
Standby 2
```

**Configuration Standby 1 :**
```ini
# postgresql.conf
hot_standby = on

# postgresql.auto.conf
primary_conninfo = 'host=primary ...'
primary_slot_name = 'standby1'
```

**Activer le mode cascade sur Standby 1 :**
```sql
-- Permettre à Standby 1 de créer ses propres slots
ALTER SYSTEM SET max_replication_slots = 5;
```

**Configuration Standby 2 :**
```ini
primary_conninfo = 'host=standby1 ...'
primary_slot_name = 'standby2'
```

### 7.3. Multi-Master avec Logical Replication

La réplication logique bi-directionnelle permet du quasi-multi-master.

```
    Database A              Database B
        │                       │
        │ Logical Slot          │ Logical Slot
        │ Publication           │ Publication
        │                       │
        ├───────────────────────►
        │   Replication         │
        │                       │
        ◄───────────────────────┤
                Replication
```

**⚠️ Attention aux conflits :**
- Insertions concurrentes avec même clé primaire
- Mises à jour du même enregistrement
- Nécessite une stratégie de résolution de conflits

**Solutions :**
- Partitionnement par clé (chaque nœud possède certaines données)
- Timestamps et "Last Write Wins"
- Conflict resolution triggers personnalisés

---

## 8. Problèmes Courants et Dépannage

### 8.1. Problème 1 : "pg_wal/ is full"

**Symptôme :**
```
PANIC: could not write to file "pg_wal/xlogtemp.12345": No space left on device
```

**Diagnostic :**
```sql
-- Identifier les slots problématiques
SELECT
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots
WHERE pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 1073741824  -- > 1 GB
ORDER BY lag DESC;
```

**Solutions :**

**Option 1 : Relancer le replica/subscriber**
```bash
# Sur le standby
sudo systemctl restart postgresql
```

**Option 2 : Avancer le slot (perte de données)**
```sql
-- ⚠️ Utiliser uniquement en dernier recours
SELECT pg_replication_slot_advance('problematic_slot', pg_current_wal_lsn());
```

**Option 3 : Supprimer le slot (recommencer la réplication)**
```sql
SELECT pg_drop_replication_slot('problematic_slot');
-- Puis reconstruire le replica depuis zéro
```

**Prévention :**
- Configurer `max_slot_wal_keep_size` (PostgreSQL 13+)
- Surveiller activement les slots
- Mettre en place des alertes

### 8.2. Problème 2 : "Logical replication slow or stuck"

**Symptôme :**
La réplication logique n'avance plus ou est très lente.

**Diagnostic :**

```sql
-- 1. Vérifier l'activité du slot
SELECT * FROM pg_stat_replication WHERE application_name = 'subscription_name';

-- 2. Vérifier les transactions longues (bloquent le décodage)
SELECT
    pid,
    usename,
    application_name,
    state,
    backend_xid,
    backend_xmin,
    query_start,
    state_change,
    query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL OR backend_xid IS NOT NULL
ORDER BY query_start;

-- 3. Vérifier les spillages (transactions trop volumineuses)
SELECT * FROM pg_stat_replication_slots WHERE spill_txns > 0;
```

**Solutions :**

**Cause 1 : Transaction longue bloquant le catalog**
```sql
-- Identifier et terminer la transaction coupable
SELECT pg_terminate_backend(pid);
```

**Cause 2 : Mémoire insuffisante pour le décodage**
```ini
# postgresql.conf
logical_decoding_work_mem = 256MB  # Augmenter (défaut: 64MB)
```

**Cause 3 : Conflits de réplication**
```sql
-- Sur le Subscriber, vérifier les erreurs
SELECT * FROM pg_stat_subscription;

-- Logs du Subscriber
tail -f /var/log/postgresql/postgresql.log | grep ERROR
```

### 8.3. Problème 3 : "Cannot drop slot: active"

**Symptôme :**
```
ERROR:  replication slot "slot_name" is active for PID 12345
```

**Solution :**
```sql
-- 1. Identifier le PID utilisant le slot
SELECT active_pid FROM pg_replication_slots WHERE slot_name = 'slot_name';

-- 2. Terminer le processus
SELECT pg_terminate_backend(active_pid);

-- 3. Supprimer le slot
SELECT pg_drop_replication_slot('slot_name');
```

### 8.4. Problème 4 : "Slot disappeared after primary failover"

**Symptôme :**
Après un failover, les slots de réplication n'existent plus sur le nouveau primary.

**Cause :**
Par défaut, les slots physiques ne sont **pas répliqués** vers les standbys.

**Solution (PostgreSQL 11+) :**
Activer `hot_standby_feedback` et créer les slots sur tous les nœuds, ou utiliser les **failover slots** (PostgreSQL 17+).

**Configuration pour préserver les slots :**
```ini
# Sur le standby
hot_standby_feedback = on
```

**Créer des slots identiques sur le standby :**
```sql
-- Sur le STANDBY (avant promotion)
SELECT pg_create_physical_replication_slot('replica1_slot');
```

---

## 9. Nouveautés PostgreSQL 18

### 9.1. Statistiques I/O améliorées

PostgreSQL 18 ajoute des métriques détaillées sur les slots de réplication dans `pg_stat_replication_slots`.

**Nouvelles colonnes (exemple hypothétique PG 18) :**
```sql
SELECT
    slot_name,
    total_bytes,      -- Total d'octets décodés
    spill_bytes,      -- Octets écrits sur disque (spillage)
    io_time,          -- Temps passé en I/O
    decode_time       -- Temps passé à décoder le WAL
FROM pg_stat_replication_slots;
```

### 9.2. max_slot_wal_keep_size par défaut plus intelligent

PostgreSQL 18 améliore la gestion automatique de `max_slot_wal_keep_size` pour éviter les saturations.

**Comportement :**
- Détection automatique de l'espace disque disponible
- Limite adaptative basée sur l'espace libre
- Logging préventif quand la limite est approchée

### 9.3. Slots logiques et colonnes virtuelles

Avec l'introduction des colonnes générées virtuelles (Virtual Generated Columns) en PG 18, les slots logiques doivent gérer ces nouvelles colonnes.

**Impact :**
- Les colonnes virtuelles ne sont **pas stockées** physiquement
- Elles sont **recalculées** lors du décodage logique sur le subscriber
- Peut impacter les performances du décodage

---

## 10. Bonnes Pratiques et Recommandations

### 10.1. Configuration de sécurité

#### 1. Limiter le nombre de slots

```ini
# postgresql.conf
max_replication_slots = 10  # Adapter selon vos besoins réels
```

**Rationnel :**
- Prévient la création accidentelle de trop de slots
- Limite l'impact d'un slot bloqué sur le système

#### 2. Configurer max_slot_wal_keep_size (PG 13+)

```ini
# postgresql.conf
max_slot_wal_keep_size = 10GB  # Limite la rétention WAL par slot
```

**Avantages :**
- Empêche un slot de bloquer indéfiniment le système
- Le slot sera "invalidé" si la limite est atteinte
- Prévient la saturation du disque

**⚠️ Attention :** Un slot invalidé nécessitera une resynchronisation complète du replica.

#### 3. Nommer les slots de manière explicite

```sql
-- ❌ Mauvais : nom générique
SELECT pg_create_physical_replication_slot('slot1');

-- ✅ Bon : nom descriptif
SELECT pg_create_physical_replication_slot('prod_standby_paris_slot');
SELECT pg_create_logical_replication_slot('analytics_dwh_slot', 'pgoutput');
```

### 10.2. Performance et optimisation

#### 1. Slots physiques : privilégier pour la HA

Pour la haute disponibilité et le disaster recovery, utilisez **toujours** des slots physiques :
- Performance optimale
- Overhead minimal
- Fiabilité maximale

#### 2. Slots logiques : optimiser le décodage

```ini
# postgresql.conf

# Augmenter la mémoire pour le décodage logique
logical_decoding_work_mem = 256MB

# Réduire les spillages sur disque
max_wal_size = 4GB
```

#### 3. Réplication logique : publier uniquement le nécessaire

```sql
-- ❌ Mauvais : publier toutes les tables
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- ✅ Bon : publier uniquement les tables nécessaires
CREATE PUBLICATION my_pub FOR TABLE users, orders, products;
```

### 10.3. Monitoring et maintenance

#### 1. Automatiser le nettoyage des slots orphelins

```bash
#!/bin/bash
# cleanup_inactive_slots.sh

INACTIVE_THRESHOLD_HOURS=24

# Supprimer les slots inactifs depuis > 24h
psql -c "
  SELECT pg_drop_replication_slot(slot_name)
  FROM pg_replication_slots
  WHERE active = false
    AND slot_type = 'logical'
    AND NOT EXISTS (
      SELECT 1 FROM pg_subscription WHERE subslotname = slot_name
    );
"
```

#### 2. Dashboard de monitoring

**Métriques essentielles à afficher :**
- Nombre de slots actifs vs inactifs
- Lag de réplication par slot (graphique temporel)
- Taille de pg_wal/ (graphique temporel)
- Taux de génération WAL (MB/s)
- Spillages des slots logiques (transactions trop volumineuses)

### 10.4. Documentation opérationnelle

Pour chaque slot créé, documentez :
- **Nom** : Identifiant unique
- **Type** : Physical ou Logical
- **Objectif** : HA, Analytics, CDC, etc.
- **Destination** : Serveur/application consommateur
- **Criticité** : Critical, Important, Nice-to-have
- **Contact** : Équipe ou personne responsable
- **Date de création**
- **Procédure de recréation** en cas de problème

**Exemple de tableau de bord :**

| Slot Name | Type | Purpose | Destination | Criticality | Owner | Created |
|-----------|------|---------|-------------|-------------|-------|---------|
| `ha_paris_slot` | Physical | HA Standby | paris-replica.db | Critical | Ops Team | 2025-01-15 |
| `analytics_slot` | Logical | BI/Analytics | dwh.analytics | Important | Data Team | 2025-02-01 |
| `audit_slot` | Logical | Audit Trail | audit-db | Nice-to-have | Security | 2025-03-10 |

---

## 11. Cas d'Usage Réels

### 11.1. Haute Disponibilité avec Patroni

**Architecture :**
```
┌─────────────────────┐
│   Patroni Cluster   │
│   (avec etcd/Consul)│
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐  ┌─────────┐
│ Primary │  │ Standby │
│ (slots) │  │         │
└─────────┘  └─────────┘
```

**Slots automatiques :**
- Patroni crée et gère automatiquement les slots physiques
- Lors d'un failover, le nouveau primary hérite des slots
- Configuration :

```yaml
# patroni.yml
postgresql:
  parameters:
    max_replication_slots: 5
  slots:
    standby_slot:
      type: physical
```

### 11.2. Change Data Capture (CDC) vers Kafka

**Architecture :**
```
PostgreSQL (Primary)
      │ logical slot: "debezium_slot"
      │ plugin: pgoutput
      ▼
Debezium Connector
      │
      ▼
Apache Kafka
      │
      ├──► Topic: users_changes
      ├──► Topic: orders_changes
      └──► Topic: products_changes
```

**Configuration :**
```sql
-- Sur PostgreSQL
CREATE PUBLICATION debezium_pub FOR TABLE users, orders, products;
```

**Debezium connector config :**
```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-host",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "secret",
    "database.dbname": "production",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_pub",
    "plugin.name": "pgoutput"
  }
}
```

### 11.3. Migration en ligne vers une nouvelle version

**Scénario :** Migrer de PostgreSQL 13 vers PostgreSQL 15 sans downtime.

**Étapes avec réplication logique :**

1. **Préparer la nouvelle instance (PG 15)**
```bash
# Nouvelle instance PG 15
initdb -D /var/lib/postgresql/15/main
```

2. **Créer la publication sur l'ancienne instance (PG 13)**
```sql
-- Sur PG 13
CREATE PUBLICATION migration_pub FOR ALL TABLES;
```

3. **Créer la souscription sur la nouvelle instance (PG 15)**
```sql
-- Sur PG 15
CREATE SUBSCRIPTION migration_sub
  CONNECTION 'host=pg13-host dbname=mydb user=replication_user'
  PUBLICATION migration_pub
  WITH (copy_data = true, create_slot = true, slot_name = 'migration_slot');
```

4. **Attendre la synchronisation complète**
```sql
-- Sur PG 15
SELECT * FROM pg_stat_subscription;
```

5. **Basculer l'application vers PG 15**
```bash
# Changer la connexion de l'application
APPLICATION_DB_HOST=pg15-host
```

6. **Nettoyer**
```sql
-- Sur PG 15
DROP SUBSCRIPTION migration_sub;

-- Sur PG 13
SELECT pg_drop_replication_slot('migration_slot');
```

---

## 12. Résumé et Points Clés

### Ce qu'il faut retenir 🔑

#### Slots de Réplication en Général
1. **Empêchent la suppression du WAL** nécessaire aux consommateurs (replicas, subscribers)
2. **Persistent entre les redémarrages** de PostgreSQL
3. **Peuvent saturer pg_wal/** si mal surveillés → **Critique à monitorer**

#### Slots Physiques
- ✅ **Usage** : Haute disponibilité, Disaster Recovery
- ✅ **Avantages** : Performance, Simplicité, Fiabilité
- ⚠️ **Limitation** : Tout ou rien (pas de granularité)

#### Slots Logiques
- ✅ **Usage** : Migrations, CDC, Analytics, Réplication sélective
- ✅ **Avantages** : Flexibilité, Transformations, Multi-versions
- ⚠️ **Limitations** : Overhead, Pas de DDL, Conflits potentiels

#### Configuration Minimale

**Slot Physique :**
```sql
SELECT pg_create_physical_replication_slot('my_slot');
```
```ini
# Sur le standby
primary_slot_name = 'my_slot'
```

**Slot Logique :**
```ini
# Sur le primary
wal_level = logical
```
```sql
CREATE PUBLICATION my_pub FOR TABLE t1, t2;
CREATE SUBSCRIPTION my_sub CONNECTION '...' PUBLICATION my_pub;
```

#### Monitoring Essentiel
```sql
-- Vérifier le lag
SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
FROM pg_replication_slots;

-- Vérifier les slots inactifs
SELECT * FROM pg_replication_slots WHERE active = false;
```

#### Protection contre la saturation (PG 13+)
```ini
max_slot_wal_keep_size = 10GB  # Limite la rétention par slot
```

### Prochaines Étapes

Maintenant que vous maîtrisez les slots de réplication :
- **17.5** : Failover et Promotion (gestion des basculements)
- **17.6** : Architectures HA complètes (Patroni, Repmgr)
- **Chapitre 18** : Extensions et cas d'usage avancés

---

## Ressources Complémentaires

### Documentation Officielle
- [PostgreSQL: Replication Slots](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-SLOTS)
- [PostgreSQL: Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [PostgreSQL: Logical Decoding](https://www.postgresql.org/docs/current/logicaldecoding.html)

### Outils de l'Écosystème
- **Patroni** : HA cluster avec gestion automatique des slots
- **Debezium** : Plateforme CDC complète
- **pgBackRest** : Gestion des sauvegardes et slots
- **Repmgr** : Gestion de réplication simplifiée

### Articles et Blogs Techniques
- 2ndQuadrant: "Understanding Replication Slots"
- Percona: "Logical Replication in PostgreSQL"
- Cybertec: "Monitoring Replication Slots"

### Livres Recommandés
- "PostgreSQL: Up and Running" - Chapitre Replication
- "Mastering PostgreSQL 13" - Advanced Replication Topics

---


⏭️ [Failover et Promotion](/17-haute-disponibilite-et-replication/05-failover-et-promotion.md)
