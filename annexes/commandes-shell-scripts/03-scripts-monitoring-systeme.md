🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Scripts de Monitoring Système pour PostgreSQL
## Annexe G.3 - Surveillance et Observabilité

---

## Table des Matières

1. [Introduction au Monitoring](#1-introduction-au-monitoring)
2. [Métriques Système (OS)](#2-m%C3%A9triques-syst%C3%A8me-os)
3. [Métriques PostgreSQL](#3-m%C3%A9triques-postgresql)
4. [Scripts de Monitoring de Base](#4-scripts-de-monitoring-de-base)
5. [Scripts de Monitoring Avancés](#5-scripts-de-monitoring-avanc%C3%A9s)
6. [Monitoring des Performances](#6-monitoring-des-performances)
7. [Monitoring des Ressources](#7-monitoring-des-ressources)
8. [Monitoring de la Santé](#8-monitoring-de-la-sant%C3%A9)
9. [Alertes et Seuils](#9-alertes-et-seuils)
10. [Tableaux de Bord et Rapports](#10-tableaux-de-bord-et-rapports)
11. [Intégration avec des Outils](#11-int%C3%A9gration-avec-des-outils)
12. [Bonnes Pratiques](#12-bonnes-pratiques)

---

## 1. Introduction au Monitoring

### 1.1. Qu'est-ce que le Monitoring ?

Le **monitoring** (surveillance) est l'action de **collecter, analyser et visualiser** des métriques sur le fonctionnement d'un système. Pour PostgreSQL, cela signifie surveiller :

- **Le système d'exploitation** : CPU, RAM, disque, réseau
- **PostgreSQL lui-même** : connexions, requêtes, performances
- **Les bases de données** : taille, activité, santé
- **Les applications** : comportement des utilisateurs, erreurs

**Analogie :** Le monitoring, c'est comme le tableau de bord d'une voiture :
- Vitesse (charge CPU)
- Essence (espace disque)
- Température (load average)
- Voyants d'alerte (erreurs, seuils dépassés)

### 1.2. Pourquoi Monitorer PostgreSQL ?

**Sans monitoring :**
- ❌ Pannes non détectées avant qu'il ne soit trop tard
- ❌ Performances dégradées sans explication
- ❌ Problèmes récurrents non identifiés
- ❌ Capacité dépassée sans anticipation
- ❌ Temps de résolution long (pas de visibilité)

**Avec monitoring :**
- ✅ Détection proactive des problèmes
- ✅ Anticipation des pannes (tendances)
- ✅ Optimisation basée sur des données réelles
- ✅ Conformité (SLA, audits)
- ✅ Historique pour post-mortem

### 1.3. Les 3 Piliers de l'Observabilité

**1. Métriques (Metrics)**
- Valeurs numériques mesurées dans le temps
- Exemple : CPU à 75%, 150 connexions actives, 2.5s de latence

**2. Logs**
- Événements textuels horodatés
- Exemple : "ERROR: duplicate key", "Connection from 192.168.1.10"

**3. Traces**
- Suivi d'une requête à travers le système
- Exemple : Requête → Parsing → Planification → Exécution → Résultat

**Ce tutoriel se concentre sur les métriques (1).**

### 1.4. Types de Monitoring

#### 1.4.1. Monitoring Infrastructure (System-Level)

**Ce qui est surveillé :**
- CPU (utilisation, load average)
- RAM (utilisation, swap)
- Disque (I/O, espace, latence)
- Réseau (bande passante, erreurs)

**Outils :** `top`, `vmstat`, `iostat`, `sar`

#### 1.4.2. Monitoring Application (PostgreSQL-Level)

**Ce qui est surveillé :**
- Connexions (actives, idle)
- Requêtes (lentes, bloquées)
- Transactions (commits, rollbacks)
- Cache (hit ratio)
- Réplication (lag)

**Outils :** `pg_stat_*` views, `pg_stat_statements`

#### 1.4.3. Monitoring Business (Application-Level)

**Ce qui est surveillé :**
- Nombre d'utilisateurs actifs
- Transactions métier par seconde
- Taux d'erreur applicatif
- SLA respectés

**Outils :** APM (Application Performance Monitoring)

### 1.5. Méthodologie de Monitoring

**Les 4 Golden Signals (Google SRE) :**

1. **Latency** (Latence)
   - Temps de réponse des requêtes
   - Objectif : < 100ms pour 95% des requêtes

2. **Traffic** (Trafic)
   - Nombre de requêtes par seconde
   - Charge du système

3. **Errors** (Erreurs)
   - Taux d'échec des requêtes
   - Connexions refusées

4. **Saturation** (Saturation)
   - Ressources proches de la limite
   - CPU > 80%, Disque > 90%

### 1.6. Fréquence de Collecte

| Métrique | Fréquence | Rétention |
|----------|-----------|-----------|
| **CPU, RAM** | 10-30s | 30 jours |
| **Disque I/O** | 30-60s | 30 jours |
| **Connexions** | 10-30s | 30 jours |
| **Requêtes lentes** | Continue | 7 jours |
| **Cache hit ratio** | 1-5 min | 90 jours |
| **Taille des bases** | 1 heure | 1 an |
| **Logs d'erreur** | Continue | 30 jours |

**💡 Principe :** Plus la métrique est volatile, plus la fréquence doit être élevée.

---

## 2. Métriques Système (OS)

### 2.1. CPU (Processeur)

#### 2.1.1. Métriques Clés

**Utilisation CPU (%) :**
- `user` : Processus utilisateur (PostgreSQL)
- `system` : Noyau Linux
- `iowait` : Attente I/O (disque)
- `idle` : Inactivité

**Load Average :**
- Nombre moyen de processus en attente
- Sur 1, 5, 15 minutes
- Seuil : Load > Nombre de CPU = Surcharge

**Contexte Switch :**
- Nombre de changements de processus
- Élevé = Système surchargé

#### 2.1.2. Commandes de Collecte

```bash
# Utilisation CPU en temps réel
top -b -n 1 | grep "Cpu(s)"

# Load average
uptime
# Exemple : load average: 2.50, 1.75, 1.20

# CPU par cœur
mpstat -P ALL 1 1

# Utilisation CPU de PostgreSQL
ps aux | grep postgres | awk '{sum+=$3} END {print sum"%"}'

# Top 10 processus PostgreSQL
ps aux | grep postgres | sort -k3 -r | head -10
```

#### 2.1.3. Script de Monitoring CPU

```bash
#!/bin/bash
# monitor_cpu.sh

get_cpu_usage() {
    top -b -n 1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1
}

get_load_average() {
    uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//'
}

get_cpu_count() {
    nproc
}

# Collecte
CPU_USAGE=$(get_cpu_usage)
LOAD_1MIN=$(get_load_average)
CPU_COUNT=$(get_cpu_count)
LOAD_PER_CPU=$(echo "scale=2; $LOAD_1MIN / $CPU_COUNT" | bc)

echo "=========================================="
echo "CPU Monitoring"
echo "=========================================="
echo "Utilisation : ${CPU_USAGE}%"
echo "Load Average (1min) : $LOAD_1MIN"
echo "Nombre de CPUs : $CPU_COUNT"
echo "Load par CPU : $LOAD_PER_CPU"

# Alerte si load > CPU count
if (( $(echo "$LOAD_1MIN > $CPU_COUNT" | bc -l) )); then
    echo "⚠️ ALERTE : System overloaded (Load: $LOAD_1MIN > CPUs: $CPU_COUNT)"
fi

# Alerte si CPU > 90%
if (( $(echo "$CPU_USAGE > 90" | bc -l) )); then
    echo "⚠️ ALERTE : CPU usage critical ($CPU_USAGE%)"
fi
```

### 2.2. Mémoire (RAM)

#### 2.2.1. Métriques Clés

**Utilisation RAM :**
- `total` : Mémoire totale
- `used` : Mémoire utilisée
- `free` : Mémoire libre
- `available` : Mémoire disponible (inclut cache)
- `cached` : Cache disque (réutilisable)
- `buffers` : Buffers noyau

**Swap :**
- Mémoire virtuelle sur disque
- Swap élevé = Problème de RAM

**OOM Killer :**
- Linux tue des processus si RAM saturée
- À surveiller dans les logs

#### 2.2.2. Commandes de Collecte

```bash
# Vue globale
free -h

# Détails
cat /proc/meminfo

# Utilisation mémoire de PostgreSQL
ps aux | grep postgres | awk '{sum+=$4} END {print sum"%"}'

# Top 10 processus par mémoire
ps aux | sort -k4 -r | head -10

# Vérifier swap
swapon --show
vmstat 1 5

# Vérifier OOM Killer dans les logs
dmesg | grep -i "killed process"
journalctl -k | grep -i "killed process"
```

#### 2.2.3. Script de Monitoring RAM

```bash
#!/bin/bash
# monitor_ram.sh

get_memory_info() {
    free -b | awk 'NR==2 {printf "%.2f %.2f %.2f", $2/1024/1024/1024, $3/1024/1024/1024, $7/1024/1024/1024}'
}

get_swap_info() {
    free -b | awk 'NR==3 {printf "%.2f %.2f", $2/1024/1024/1024, $3/1024/1024/1024}'
}

# Collecte
read TOTAL_RAM USED_RAM AVAILABLE_RAM <<< $(get_memory_info)
read TOTAL_SWAP USED_SWAP <<< $(get_swap_info)

USAGE_PCT=$(echo "scale=2; $USED_RAM * 100 / $TOTAL_RAM" | bc)
AVAILABLE_PCT=$(echo "scale=2; $AVAILABLE_RAM * 100 / $TOTAL_RAM" | bc)

echo "=========================================="
echo "Memory Monitoring"
echo "=========================================="
echo "Total RAM : ${TOTAL_RAM}GB"
echo "Used RAM : ${USED_RAM}GB (${USAGE_PCT}%)"
echo "Available RAM : ${AVAILABLE_RAM}GB (${AVAILABLE_PCT}%)"
echo ""
echo "Total Swap : ${TOTAL_SWAP}GB"
echo "Used Swap : ${USED_SWAP}GB"

# Alertes
if (( $(echo "$AVAILABLE_PCT < 10" | bc -l) )); then
    echo "⚠️ ALERTE : RAM disponible critique (${AVAILABLE_PCT}%)"
fi

if (( $(echo "$USED_SWAP > 0.1" | bc -l) )); then
    echo "⚠️ ALERTE : Swap utilisé (${USED_SWAP}GB) - Risque de performance"
fi

# Vérifier OOM Killer
OOM_COUNT=$(dmesg | grep -c "Out of memory")
if [ $OOM_COUNT -gt 0 ]; then
    echo "❌ CRITIQUE : $OOM_COUNT événement(s) OOM Killer détecté(s)"
fi
```

### 2.3. Disque (Storage)

#### 2.3.1. Métriques Clés

**Espace Disque :**
- Utilisation (%)
- Espace libre
- Inodes disponibles

**I/O Disque :**
- IOPS (Input/Output Operations Per Second)
- Throughput (MB/s)
- Latence (ms)
- Queue length (files d'attente)

**Nouveauté PG 18 : I/O asynchrone (AIO)**
- Performance I/O jusqu'à 3× meilleure
- Monitoring spécifique requis

#### 2.3.2. Commandes de Collecte

```bash
# Espace disque
df -h

# Utilisation par répertoire
du -sh /var/lib/postgresql/*

# I/O en temps réel
iostat -x 1 5

# I/O par processus
iotop -o

# Statistiques I/O détaillées
vmstat -d

# Latence disque
ioping -c 10 /var/lib/postgresql
```

#### 2.3.3. Script de Monitoring Disque

```bash
#!/bin/bash
# monitor_disk.sh

PGDATA="/var/lib/postgresql/data"
WAL_DIR="$PGDATA/pg_wal"
BACKUP_DIR="/var/backups/postgresql"

get_disk_usage() {
    local path="$1"
    df -BG "$path" | tail -1 | awk '{print $2, $3, $4, $5}' | sed 's/G//g' | sed 's/%//'
}

echo "=========================================="
echo "Disk Monitoring"
echo "=========================================="

# PGDATA
echo "📁 PGDATA : $PGDATA"
read TOTAL USED FREE USAGE_PCT <<< $(get_disk_usage "$PGDATA")
echo "  Total : ${TOTAL}GB"
echo "  Used : ${USED}GB (${USAGE_PCT}%)"
echo "  Free : ${FREE}GB"

if [ $USAGE_PCT -gt 90 ]; then
    echo "  ⚠️ ALERTE : Disque presque plein (${USAGE_PCT}%)"
elif [ $USAGE_PCT -gt 80 ]; then
    echo "  ⚠️ WARNING : Disque à ${USAGE_PCT}%"
else
    echo "  ✅ Espace disque OK"
fi

# Taille des bases
echo ""
echo "📊 Taille des bases de données :"
psql -t -c "
    SELECT
        datname,
        pg_size_pretty(pg_database_size(datname)) as size
    FROM pg_database
    WHERE datistemplate = false
    ORDER BY pg_database_size(datname) DESC;
" | head -10

# WAL
echo ""
echo "📝 WAL Directory : $WAL_DIR"
WAL_SIZE=$(du -sh "$WAL_DIR" 2>/dev/null | cut -f1)
WAL_COUNT=$(find "$WAL_DIR" -type f 2>/dev/null | wc -l)
echo "  Taille : $WAL_SIZE"
echo "  Fichiers : $WAL_COUNT"

# Backups
echo ""
echo "💾 Backup Directory : $BACKUP_DIR"
if [ -d "$BACKUP_DIR" ]; then
    BACKUP_SIZE=$(du -sh "$BACKUP_DIR" 2>/dev/null | cut -f1)
    BACKUP_COUNT=$(find "$BACKUP_DIR" -name "*.dump" 2>/dev/null | wc -l)
    echo "  Taille : $BACKUP_SIZE"
    echo "  Backups : $BACKUP_COUNT"
else
    echo "  ⚠️ Répertoire inexistant"
fi

# I/O Stats (si iostat disponible)
echo ""
echo "📈 I/O Statistics :"
if command -v iostat > /dev/null 2>&1; then
    iostat -x 1 2 | tail -n +4 | grep -E "^[a-z]" | head -3
else
    echo "  iostat non disponible (installer sysstat)"
fi

# Inodes
echo ""
echo "🔢 Inodes :"
df -i "$PGDATA" | tail -1 | awk '{print "  Used: "$3" / "$2" ("$5")"}'
```

### 2.4. Réseau

#### 2.4.1. Métriques Clés

**Bande Passante :**
- RX (Reception) : Bytes/s reçus
- TX (Transmission) : Bytes/s envoyés

**Connexions :**
- ESTABLISHED : Connexions actives
- TIME_WAIT : Connexions en fermeture
- LISTEN : Ports en écoute

**Erreurs :**
- Packets dropped
- Collisions
- Errors

#### 2.4.2. Commandes de Collecte

```bash
# Statistiques réseau
ifconfig

# Bande passante en temps réel
iftop

# Connexions TCP
netstat -tan | grep ESTABLISHED | wc -l

# Connexions PostgreSQL (port 5432)
netstat -tan | grep :5432 | wc -l

# Top connexions par IP
netstat -tan | grep :5432 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10

# Statistiques détaillées
ss -s
```

#### 2.4.3. Script de Monitoring Réseau

```bash
#!/bin/bash
# monitor_network.sh

PG_PORT=5432

echo "=========================================="
echo "Network Monitoring"
echo "=========================================="

# Connexions PostgreSQL
PG_CONNECTIONS=$(netstat -tan | grep ":$PG_PORT" | grep ESTABLISHED | wc -l)
echo "🔌 Connexions PostgreSQL : $PG_CONNECTIONS"

# Top 5 IPs connectées
echo ""
echo "📊 Top 5 IPs connectées :"
netstat -tan | grep ":$PG_PORT" | grep ESTABLISHED | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -5 | while read count ip; do
    echo "  $ip : $count connexion(s)"
done

# État des connexions
echo ""
echo "📈 État des connexions TCP :"
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Bande passante (si disponible)
echo ""
echo "📡 Bande passante :"
if command -v ifstat > /dev/null 2>&1; then
    ifstat 1 1 | tail -1
else
    # Alternative avec /proc/net/dev
    cat /proc/net/dev | grep -v "lo:" | grep ":" | head -1 | awk '{print "  RX: "$2" bytes, TX: "$10" bytes"}'
fi

# Vérifier erreurs réseau
echo ""
echo "❌ Erreurs réseau :"
netstat -i | awk 'NR>2 {print $1": RX-ERR="$3", TX-ERR="$7}' | grep -v "^lo:"
```

---

## 3. Métriques PostgreSQL

### 3.1. Vues Statistiques (pg_stat_*)

PostgreSQL fournit de nombreuses **vues système** pour le monitoring :

| Vue | Description |
|-----|-------------|
| `pg_stat_activity` | Activité en cours (connexions, requêtes) |
| `pg_stat_database` | Statistiques par base de données |
| `pg_stat_user_tables` | Statistiques par table |
| `pg_stat_user_indexes` | Statistiques par index |
| `pg_stat_bgwriter` | Statistiques du background writer |
| `pg_stat_replication` | État de la réplication |
| `pg_locks` | Verrous actifs |
| `pg_stat_statements` | Requêtes exécutées (extension) |

**Nouveautés PostgreSQL 18 :**
- `pg_stat_io` : Statistiques I/O détaillées par backend
- Statistiques VACUUM/ANALYZE enrichies
- Métriques WAL par backend

### 3.2. Connexions

#### 3.2.1. Métriques Clés

- **Connexions actives** : Requêtes en cours
- **Connexions idle** : Connectées mais inactives
- **Connexions idle in transaction** : Transaction ouverte sans activité (⚠️ problème)
- **Max connections** : Limite configurée

#### 3.2.2. Requêtes de Monitoring

```sql
-- Nombre de connexions par état
SELECT
    state,
    COUNT(*) as count
FROM pg_stat_activity
GROUP BY state
ORDER BY count DESC;

/*
Résultat typique :
     state          | count
--------------------+-------
 active             |     5
 idle               |    12
 idle in transaction|     2
*/

-- Connexions par base de données
SELECT
    datname,
    COUNT(*) as connections
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;

-- Connexions par application
SELECT
    application_name,
    COUNT(*) as count
FROM pg_stat_activity
WHERE application_name IS NOT NULL
GROUP BY application_name
ORDER BY count DESC;

-- Top 10 utilisateurs connectés
SELECT
    usename,
    COUNT(*) as connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC
LIMIT 10;

-- Connexions anciennes (> 1 heure)
SELECT
    pid,
    usename,
    datname,
    state,
    NOW() - backend_start as connection_age,
    NOW() - state_change as idle_time
FROM pg_stat_activity
WHERE backend_start < NOW() - INTERVAL '1 hour'
ORDER BY backend_start;

-- Connexions bloquées
SELECT
    pid,
    usename,
    datname,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
AND wait_event_type != 'Activity';

-- Utilisation vs limite
SELECT
    (SELECT COUNT(*) FROM pg_stat_activity) as current_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') as max_connections,
    ROUND(100.0 * (SELECT COUNT(*) FROM pg_stat_activity) /
          (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2) as usage_pct;
```

#### 3.2.3. Script de Monitoring Connexions

```bash
#!/bin/bash
# monitor_connections.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Connections Monitoring"
echo "=========================================="

# Connexions par état
echo "📊 Connexions par état :"
psql -d "$DB_NAME" -t -c "
    SELECT
        COALESCE(state, 'unknown') as state,
        COUNT(*) as count
    FROM pg_stat_activity
    GROUP BY state
    ORDER BY count DESC;
" | while read state count; do
    echo "  $state : $count"
done

# Total et limite
echo ""
echo "📈 Utilisation :"
psql -d "$DB_NAME" -t -c "
    SELECT
        'Current: ' || COUNT(*) || ' / Max: ' ||
        (SELECT setting FROM pg_settings WHERE name = 'max_connections') ||
        ' (' || ROUND(100.0 * COUNT(*) / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2) || '%)'
    FROM pg_stat_activity;
"

# Connexions idle in transaction (problème potentiel)
IDLE_IN_TXN=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'idle in transaction';
" | xargs)

if [ $IDLE_IN_TXN -gt 0 ]; then
    echo ""
    echo "⚠️ WARNING : $IDLE_IN_TXN connexion(s) 'idle in transaction'"

    # Détails
    psql -d "$DB_NAME" -c "
        SELECT
            pid,
            usename,
            datname,
            NOW() - state_change as idle_duration,
            LEFT(query, 50) as query_preview
        FROM pg_stat_activity
        WHERE state = 'idle in transaction'
        ORDER BY state_change;
    "
fi

# Connexions anciennes
echo ""
echo "⏰ Connexions anciennes (> 1h) :"
OLD_CONNECTIONS=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity WHERE backend_start < NOW() - INTERVAL '1 hour';
" | xargs)

echo "  $OLD_CONNECTIONS connexion(s)"

if [ $OLD_CONNECTIONS -gt 10 ]; then
    echo "  ⚠️ Nombre élevé de connexions anciennes"
fi
```

### 3.3. Requêtes

#### 3.3.1. Extension pg_stat_statements

**Installation (si pas déjà activée) :**

```sql
-- Dans postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- Redémarrer PostgreSQL, puis :
CREATE EXTENSION pg_stat_statements;
```

**Pourquoi c'est essentiel ?**
- Collecte des statistiques de TOUTES les requêtes exécutées
- Temps d'exécution, nombre d'appels, rows retournées
- **L'outil #1 pour l'optimisation des performances**

#### 3.3.2. Requêtes de Monitoring

```sql
-- Top 10 requêtes les plus lentes (temps total)
SELECT
    queryid,
    LEFT(query, 60) as query_preview,
    calls,
    ROUND(total_exec_time::numeric, 2) as total_time_ms,
    ROUND(mean_exec_time::numeric, 2) as avg_time_ms,
    ROUND((100.0 * total_exec_time / SUM(total_exec_time) OVER())::numeric, 2) as pct_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top 10 requêtes les plus fréquentes
SELECT
    calls,
    ROUND(mean_exec_time::numeric, 2) as avg_time_ms,
    LEFT(query, 60) as query_preview
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Requêtes avec temps moyen élevé
SELECT
    calls,
    ROUND(mean_exec_time::numeric, 2) as avg_time_ms,
    ROUND(max_exec_time::numeric, 2) as max_time_ms,
    LEFT(query, 60) as query_preview
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- > 1 seconde
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Requêtes en cours d'exécution
SELECT
    pid,
    usename,
    datname,
    NOW() - query_start as duration,
    state,
    LEFT(query, 80) as query
FROM pg_stat_activity
WHERE state = 'active'
AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration DESC;

-- Requêtes longues (> 30 secondes)
SELECT
    pid,
    usename,
    NOW() - query_start as duration,
    LEFT(query, 100) as query
FROM pg_stat_activity
WHERE state = 'active'
AND query_start < NOW() - INTERVAL '30 seconds'
AND query NOT LIKE '%pg_stat_activity%';
```

#### 3.3.3. Script de Monitoring Requêtes

```bash
#!/bin/bash
# monitor_queries.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Queries Monitoring"
echo "=========================================="

# Vérifier pg_stat_statements
PSS_INSTALLED=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_extension WHERE extname = 'pg_stat_statements';
" | xargs)

if [ $PSS_INSTALLED -eq 0 ]; then
    echo "⚠️ WARNING : pg_stat_statements non installé"
    echo "Installation : CREATE EXTENSION pg_stat_statements;"
    echo ""
fi

# Requêtes actives
ACTIVE_QUERIES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active' AND query NOT LIKE '%pg_stat_activity%';
" | xargs)

echo "🔄 Requêtes actives : $ACTIVE_QUERIES"

# Requêtes longues
LONG_QUERIES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity
    WHERE state = 'active'
    AND query_start < NOW() - INTERVAL '30 seconds'
    AND query NOT LIKE '%pg_stat_activity%';
" | xargs)

echo "⏱️  Requêtes longues (>30s) : $LONG_QUERIES"

if [ $LONG_QUERIES -gt 0 ]; then
    echo ""
    echo "⚠️ Détails des requêtes longues :"
    psql -d "$DB_NAME" -c "
        SELECT
            pid,
            usename,
            ROUND(EXTRACT(EPOCH FROM (NOW() - query_start))::numeric, 2) as duration_sec,
            LEFT(query, 80) as query_preview
        FROM pg_stat_activity
        WHERE state = 'active'
        AND query_start < NOW() - INTERVAL '30 seconds'
        AND query NOT LIKE '%pg_stat_activity%'
        ORDER BY query_start;
    "
fi

# Top 5 requêtes lentes (si pg_stat_statements disponible)
if [ $PSS_INSTALLED -gt 0 ]; then
    echo ""
    echo "🐌 Top 5 requêtes les plus lentes :"
    psql -d "$DB_NAME" -c "
        SELECT
            calls,
            ROUND(total_exec_time::numeric, 2) as total_ms,
            ROUND(mean_exec_time::numeric, 2) as avg_ms,
            LEFT(query, 70) as query_preview
        FROM pg_stat_statements
        ORDER BY mean_exec_time DESC
        LIMIT 5;
    "
fi
```

### 3.4. Performance (Cache Hit Ratio)

#### 3.4.1. Qu'est-ce que le Cache Hit Ratio ?

Le **cache hit ratio** mesure le pourcentage de lectures servies depuis la RAM (cache) plutôt que depuis le disque.

**Formule :**
```
Cache Hit Ratio = (Blocks read from cache / Total blocks read) × 100
```

**Objectif :** > 99% (idéalement > 99.5%)

**Interprétation :**
- **99.5%** : Excellent ✅
- **95-99%** : Correct ⚠️
- **< 95%** : Problème de configuration (augmenter `shared_buffers`) ❌

#### 3.4.2. Requêtes de Monitoring

```sql
-- Cache Hit Ratio global
SELECT
    SUM(heap_blks_read) as disk_reads,
    SUM(heap_blks_hit) as cache_hits,
    SUM(heap_blks_hit) + SUM(heap_blks_read) as total_reads,
    ROUND(
        100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0),
        2
    ) as cache_hit_ratio
FROM pg_statio_user_tables;

-- Cache Hit Ratio par base de données
SELECT
    datname,
    ROUND(
        100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0),
        2
    ) as cache_hit_ratio,
    blks_hit,
    blks_read
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY cache_hit_ratio;

-- Cache Hit Ratio par table
SELECT
    schemaname,
    tablename,
    ROUND(
        100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0),
        2
    ) as cache_hit_ratio,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as table_size
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 10;

-- Buffer allocation stats
SELECT
    buffers_alloc,
    buffers_backend,
    buffers_backend_fsync,
    buffers_checkpoint,
    buffers_clean
FROM pg_stat_bgwriter;
```

#### 3.4.3. Script de Monitoring Cache

```bash
#!/bin/bash
# monitor_cache.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Cache Monitoring"
echo "=========================================="

# Cache Hit Ratio global
echo "📊 Cache Hit Ratio Global :"
psql -d "$DB_NAME" -t -c "
    SELECT
        ROUND(
            100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0),
            2
        ) || '%' as cache_hit_ratio
    FROM pg_statio_user_tables;
" | xargs

CACHE_RATIO=$(psql -d "$DB_NAME" -t -c "
    SELECT
        ROUND(
            100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0),
            2
        )
    FROM pg_statio_user_tables;
" | xargs)

if (( $(echo "$CACHE_RATIO < 95" | bc -l) )); then
    echo "❌ CRITIQUE : Cache hit ratio faible ($CACHE_RATIO%)"
    echo "   Action recommandée : Augmenter shared_buffers"
elif (( $(echo "$CACHE_RATIO < 99" | bc -l) )); then
    echo "⚠️ WARNING : Cache hit ratio sous-optimal ($CACHE_RATIO%)"
else
    echo "✅ Cache hit ratio excellent ($CACHE_RATIO%)"
fi

# Cache par base
echo ""
echo "📈 Cache Hit Ratio par Base :"
psql -d "$DB_NAME" -c "
    SELECT
        datname,
        ROUND(
            100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0),
            2
        ) as hit_ratio_pct,
        blks_hit as cache_hits,
        blks_read as disk_reads
    FROM pg_stat_database
    WHERE datname NOT IN ('template0', 'template1')
    ORDER BY hit_ratio_pct;
"

# Shared buffers configuration
echo ""
echo "⚙️  Configuration Shared Buffers :"
psql -d "$DB_NAME" -t -c "
    SELECT setting || ' (' || unit || ')' FROM pg_settings WHERE name = 'shared_buffers';
" | xargs
```

### 3.5. Transactions

#### 3.5.1. Métriques Clés

- **Commits** : Transactions validées
- **Rollbacks** : Transactions annulées
- **XID Wraparound** : Risque de saturation des IDs de transaction
- **Locks** : Verrous actifs
- **Deadlocks** : Blocages mutuels

#### 3.5.2. Requêtes de Monitoring

```sql
-- Statistiques de transactions par base
SELECT
    datname,
    xact_commit as commits,
    xact_rollback as rollbacks,
    ROUND(
        100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0),
        2
    ) as rollback_pct,
    deadlocks
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY xact_commit DESC;

-- Age des transactions (XID)
SELECT
    datname,
    age(datfrozenxid) as xid_age,
    ROUND(100.0 * age(datfrozenxid) / 2000000000, 2) as pct_to_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Verrous actifs
SELECT
    locktype,
    mode,
    COUNT(*) as count
FROM pg_locks
GROUP BY locktype, mode
ORDER BY count DESC;

-- Requêtes bloquées
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Deadlocks
SELECT
    datname,
    deadlocks,
    conflicts,
    temp_files,
    temp_bytes
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY deadlocks DESC;
```

### 3.6. Réplication (si configurée)

#### 3.6.1. Métriques Clés

- **Replication Lag** : Retard de réplication
- **WAL Position** : Position dans les logs
- **Replication Slot** : Slots de réplication
- **Sync State** : État de synchronisation

#### 3.6.2. Requêtes de Monitoring

```sql
-- État de la réplication
SELECT
    client_addr,
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as lag_bytes,
    EXTRACT(EPOCH FROM (NOW() - write_lag)) as write_lag_sec,
    EXTRACT(EPOCH FROM (NOW() - flush_lag)) as flush_lag_sec,
    EXTRACT(EPOCH FROM (NOW() - replay_lag)) as replay_lag_sec
FROM pg_stat_replication;

-- Slots de réplication
SELECT
    slot_name,
    plugin,
    slot_type,
    database,
    active,
    restart_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) as lag_bytes
FROM pg_replication_slots;

-- WAL archiving
SELECT
    archived_count,
    failed_count,
    last_archived_time,
    last_failed_time
FROM pg_stat_archiver;
```

---

## 4. Scripts de Monitoring de Base

### 4.1. Script de Santé Générale

```bash
#!/bin/bash
# health_check.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Health Check"
echo "=========================================="
echo "Date : $(date '+%Y-%m-%d %H:%M:%S')"
echo ""

# 1. PostgreSQL est-il actif ?
echo "1️⃣ Service Status"
if pg_isready -d "$DB_NAME" > /dev/null 2>&1; then
    echo "✅ PostgreSQL is running"
else
    echo "❌ PostgreSQL is DOWN"
    exit 1
fi

# 2. Version
echo ""
echo "2️⃣ Version"
psql -d "$DB_NAME" -t -c "SELECT version();" | head -1

# 3. Uptime
echo ""
echo "3️⃣ Uptime"
psql -d "$DB_NAME" -t -c "SELECT NOW() - pg_postmaster_start_time() as uptime;" | xargs

# 4. Connexions
echo ""
echo "4️⃣ Connexions"
CURRENT_CONN=$(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_activity;" | xargs)
MAX_CONN=$(psql -d "$DB_NAME" -t -c "SELECT setting FROM pg_settings WHERE name = 'max_connections';" | xargs)
CONN_PCT=$(echo "scale=2; 100 * $CURRENT_CONN / $MAX_CONN" | bc)

echo "Current: $CURRENT_CONN / Max: $MAX_CONN (${CONN_PCT}%)"

if (( $(echo "$CONN_PCT > 80" | bc -l) )); then
    echo "⚠️ WARNING: Connexions proches de la limite"
fi

# 5. Taille des bases
echo ""
echo "5️⃣ Database Sizes"
psql -d "$DB_NAME" -c "
    SELECT
        datname,
        pg_size_pretty(pg_database_size(datname)) as size
    FROM pg_database
    WHERE datistemplate = false
    ORDER BY pg_database_size(datname) DESC;
"

# 6. Cache Hit Ratio
echo ""
echo "6️⃣ Cache Hit Ratio"
CACHE_RATIO=$(psql -d "$DB_NAME" -t -c "
    SELECT
        ROUND(
            100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0),
            2
        )
    FROM pg_statio_user_tables;
" | xargs)

echo "Cache Hit Ratio: ${CACHE_RATIO}%"

if (( $(echo "$CACHE_RATIO < 95" | bc -l) )); then
    echo "❌ CRITICAL: Cache hit ratio too low"
elif (( $(echo "$CACHE_RATIO < 99" | bc -l) )); then
    echo "⚠️ WARNING: Cache hit ratio could be better"
else
    echo "✅ Excellent cache performance"
fi

# 7. Requêtes longues
echo ""
echo "7️⃣ Long Running Queries"
LONG_QUERIES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity
    WHERE state = 'active'
    AND query_start < NOW() - INTERVAL '1 minute'
    AND query NOT LIKE '%pg_stat_activity%';
" | xargs)

echo "Queries running > 1 minute: $LONG_QUERIES"

if [ $LONG_QUERIES -gt 0 ]; then
    echo "⚠️ WARNING: Long running queries detected"
fi

# 8. Espace disque
echo ""
echo "8️⃣ Disk Space"
PGDATA=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" | xargs)
DISK_USAGE=$(df -h "$PGDATA" | tail -1 | awk '{print $5}' | sed 's/%//')

echo "PGDATA: $PGDATA"
echo "Usage: ${DISK_USAGE}%"

if [ $DISK_USAGE -gt 90 ]; then
    echo "❌ CRITICAL: Disk almost full"
elif [ $DISK_USAGE -gt 80 ]; then
    echo "⚠️ WARNING: Disk filling up"
else
    echo "✅ Disk space OK"
fi

# 9. XID Wraparound
echo ""
echo "9️⃣ XID Wraparound Risk"
psql -d "$DB_NAME" -c "
    SELECT
        datname,
        age(datfrozenxid) as xid_age,
        ROUND(100.0 * age(datfrozenxid) / 2000000000, 2) as pct_to_wraparound
    FROM pg_database
    WHERE age(datfrozenxid) > 1000000000
    ORDER BY age(datfrozenxid) DESC;
"

# 10. Résumé
echo ""
echo "=========================================="
echo "✅ Health Check Complete"
echo "=========================================="
```

### 4.2. Script de Collecte de Métriques

```bash
#!/bin/bash
# collect_metrics.sh

DB_NAME="postgres"
METRICS_FILE="/var/log/postgresql_metrics.csv"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Créer le fichier avec header si inexistant
if [ ! -f "$METRICS_FILE" ]; then
    echo "timestamp,connections,cache_hit_ratio,disk_usage_pct,cpu_usage_pct,active_queries,long_queries" > "$METRICS_FILE"
fi

# Collecter métriques
CONNECTIONS=$(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_activity;" | xargs)

CACHE_RATIO=$(psql -d "$DB_NAME" -t -c "
    SELECT COALESCE(
        ROUND(100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2),
        0
    )
    FROM pg_statio_user_tables;
" | xargs)

PGDATA=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" | xargs)
DISK_USAGE=$(df "$PGDATA" | tail -1 | awk '{print $5}' | sed 's/%//')

CPU_USAGE=$(top -b -n 1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)

ACTIVE_QUERIES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active' AND query NOT LIKE '%pg_stat_activity%';
" | xargs)

LONG_QUERIES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity
    WHERE state = 'active'
    AND query_start < NOW() - INTERVAL '30 seconds'
    AND query NOT LIKE '%pg_stat_activity%';
" | xargs)

# Écrire dans le fichier
echo "$TIMESTAMP,$CONNECTIONS,$CACHE_RATIO,$DISK_USAGE,$CPU_USAGE,$ACTIVE_QUERIES,$LONG_QUERIES" >> "$METRICS_FILE"

echo "Metrics collected at $TIMESTAMP"
```

**Automatiser avec cron :**

```bash
# Collecter toutes les 5 minutes
*/5 * * * * /usr/local/bin/collect_metrics.sh
```

---

## 5. Scripts de Monitoring Avancés

### 5.1. Script de Monitoring Complet

```bash
#!/bin/bash
# monitor_complete.sh

DB_NAME="postgres"
LOG_FILE="/var/log/postgresql_monitor.log"
ALERT_EMAIL="admin@example.com"

# Seuils d'alerte
ALERT_CPU=80
ALERT_DISK=85
ALERT_CONNECTIONS_PCT=80
ALERT_CACHE_HIT=95
ALERT_LONG_QUERIES=5

ALERTS=()

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

send_alert() {
    if [ ${#ALERTS[@]} -gt 0 ]; then
        ALERT_BODY=$(printf "%s\n" "${ALERTS[@]}")
        echo "$ALERT_BODY" | mail -s "⚠️ PostgreSQL Monitoring Alert" "$ALERT_EMAIL"
        log "📧 Alert email sent"
    fi
}

log "=========================================="
log "PostgreSQL Complete Monitoring"
log "=========================================="

# 1. Service Status
log "Checking PostgreSQL service..."
if ! pg_isready -d "$DB_NAME" > /dev/null 2>&1; then
    ALERTS+=("❌ CRITICAL: PostgreSQL is DOWN")
    log "❌ PostgreSQL is DOWN"
    send_alert
    exit 1
fi
log "✅ PostgreSQL is running"

# 2. CPU
CPU_USAGE=$(top -b -n 1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
log "CPU Usage: ${CPU_USAGE}%"
if (( $(echo "$CPU_USAGE > $ALERT_CPU" | bc -l) )); then
    ALERTS+=("⚠️ CPU usage high: ${CPU_USAGE}% (threshold: ${ALERT_CPU}%)")
fi

# 3. Disk
PGDATA=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" | xargs)
DISK_USAGE=$(df "$PGDATA" | tail -1 | awk '{print $5}' | sed 's/%//')
log "Disk Usage: ${DISK_USAGE}%"
if [ $DISK_USAGE -gt $ALERT_DISK ]; then
    ALERTS+=("⚠️ Disk usage high: ${DISK_USAGE}% (threshold: ${ALERT_DISK}%)")
fi

# 4. Connections
CURRENT_CONN=$(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_activity;" | xargs)
MAX_CONN=$(psql -d "$DB_NAME" -t -c "SELECT setting FROM pg_settings WHERE name = 'max_connections';" | xargs)
CONN_PCT=$(echo "scale=2; 100 * $CURRENT_CONN / $MAX_CONN" | bc)
log "Connections: $CURRENT_CONN / $MAX_CONN (${CONN_PCT}%)"
if (( $(echo "$CONN_PCT > $ALERT_CONNECTIONS_PCT" | bc -l) )); then
    ALERTS+=("⚠️ Connections high: ${CONN_PCT}% (threshold: ${ALERT_CONNECTIONS_PCT}%)")
fi

# 5. Cache Hit Ratio
CACHE_RATIO=$(psql -d "$DB_NAME" -t -c "
    SELECT COALESCE(
        ROUND(100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2),
        100
    )
    FROM pg_statio_user_tables;
" | xargs)
log "Cache Hit Ratio: ${CACHE_RATIO}%"
if (( $(echo "$CACHE_RATIO < $ALERT_CACHE_HIT" | bc -l) )); then
    ALERTS+=("⚠️ Cache hit ratio low: ${CACHE_RATIO}% (threshold: ${ALERT_CACHE_HIT}%)")
fi

# 6. Long Queries
LONG_QUERIES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity
    WHERE state = 'active'
    AND query_start < NOW() - INTERVAL '1 minute'
    AND query NOT LIKE '%pg_stat_activity%';
" | xargs)
log "Long Queries (>1min): $LONG_QUERIES"
if [ $LONG_QUERIES -gt $ALERT_LONG_QUERIES ]; then
    ALERTS+=("⚠️ Long queries: $LONG_QUERIES (threshold: $ALERT_LONG_QUERIES)")
fi

# 7. Idle in Transaction
IDLE_IN_TXN=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'idle in transaction';
" | xargs)
log "Idle in Transaction: $IDLE_IN_TXN"
if [ $IDLE_IN_TXN -gt 0 ]; then
    ALERTS+=("⚠️ Connections idle in transaction: $IDLE_IN_TXN")
fi

# 8. Locks
LOCKS=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_locks WHERE granted = false;
" | xargs)
log "Blocked Locks: $LOCKS"
if [ $LOCKS -gt 0 ]; then
    ALERTS+=("⚠️ Blocked locks detected: $LOCKS")
fi

# 9. Replication Lag (si applicable)
REPLICATION_COUNT=$(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_replication;" | xargs)
if [ $REPLICATION_COUNT -gt 0 ]; then
    MAX_LAG=$(psql -d "$DB_NAME" -t -c "
        SELECT COALESCE(MAX(EXTRACT(EPOCH FROM (NOW() - replay_lag))), 0)
        FROM pg_stat_replication;
    " | xargs)
    log "Max Replication Lag: ${MAX_LAG}s"

    if (( $(echo "$MAX_LAG > 60" | bc -l) )); then
        ALERTS+=("⚠️ Replication lag high: ${MAX_LAG}s (threshold: 60s)")
    fi
fi

# 10. Database Sizes (top 3)
log ""
log "Top 3 Database Sizes:"
psql -d "$DB_NAME" -t -c "
    SELECT
        datname || ': ' || pg_size_pretty(pg_database_size(datname))
    FROM pg_database
    WHERE datistemplate = false
    ORDER BY pg_database_size(datname) DESC
    LIMIT 3;
" | while read line; do
    log "  $line"
done

# Résumé et alertes
log ""
log "=========================================="
if [ ${#ALERTS[@]} -eq 0 ]; then
    log "✅ All checks passed - System healthy"
else
    log "⚠️ ${#ALERTS[@]} alert(s) detected:"
    for alert in "${ALERTS[@]}"; do
        log "  $alert"
    done
    send_alert
fi
log "=========================================="
```

**Automatiser :**

```bash
# Toutes les 5 minutes
*/5 * * * * /usr/local/bin/monitor_complete.sh
```

### 5.2. Script de Monitoring avec Historique

```bash
#!/bin/bash
# monitor_with_history.sh

DB_NAME="postgres"
HISTORY_DIR="/var/lib/postgresql_history"
TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
DATE=$(date '+%Y%m%d')

mkdir -p "$HISTORY_DIR"

# Fichiers d'historique
CONNECTIONS_FILE="$HISTORY_DIR/connections_${DATE}.csv"
CACHE_FILE="$HISTORY_DIR/cache_${DATE}.csv"
QUERIES_FILE="$HISTORY_DIR/queries_${DATE}.csv"
DISK_FILE="$HISTORY_DIR/disk_${DATE}.csv"

# Headers si fichiers n'existent pas
[ ! -f "$CONNECTIONS_FILE" ] && echo "timestamp,total,active,idle,idle_in_transaction,max_connections,usage_pct" > "$CONNECTIONS_FILE"
[ ! -f "$CACHE_FILE" ] && echo "timestamp,cache_hit_ratio,heap_blks_read,heap_blks_hit" > "$CACHE_FILE"
[ ! -f "$QUERIES_FILE" ] && echo "timestamp,active_queries,long_queries,blocked_queries" > "$QUERIES_FILE"
[ ! -f "$DISK_FILE" ] && echo "timestamp,total_gb,used_gb,free_gb,usage_pct" > "$DISK_FILE"

# Collecte Connexions
CONN_STATS=$(psql -d "$DB_NAME" -t -c "
    SELECT
        COUNT(*) as total,
        COUNT(*) FILTER (WHERE state = 'active') as active,
        COUNT(*) FILTER (WHERE state = 'idle') as idle,
        COUNT(*) FILTER (WHERE state = 'idle in transaction') as idle_in_txn,
        (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') as max_conn,
        ROUND(100.0 * COUNT(*) / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2) as usage_pct
    FROM pg_stat_activity;
" | xargs | tr ' ' ',')

echo "$TIMESTAMP,$CONN_STATS" >> "$CONNECTIONS_FILE"

# Collecte Cache
CACHE_STATS=$(psql -d "$DB_NAME" -t -c "
    SELECT
        COALESCE(ROUND(100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2), 100),
        SUM(heap_blks_read),
        SUM(heap_blks_hit)
    FROM pg_statio_user_tables;
" | xargs | tr ' ' ',')

echo "$TIMESTAMP,$CACHE_STATS" >> "$CACHE_FILE"

# Collecte Requêtes
QUERY_STATS=$(psql -d "$DB_NAME" -t -c "
    SELECT
        COUNT(*) FILTER (WHERE state = 'active' AND query NOT LIKE '%pg_stat_activity%') as active,
        COUNT(*) FILTER (WHERE state = 'active' AND query_start < NOW() - INTERVAL '30 seconds' AND query NOT LIKE '%pg_stat_activity%') as long,
        (SELECT COUNT(*) FROM pg_locks WHERE NOT granted) as blocked
    FROM pg_stat_activity;
" | xargs | tr ' ' ',')

echo "$TIMESTAMP,$QUERY_STATS" >> "$QUERIES_FILE"

# Collecte Disque
PGDATA=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" | xargs)
DISK_STATS=$(df -BG "$PGDATA" | tail -1 | awk '{print $2","$3","$4","$5}' | sed 's/G//g' | sed 's/%//')

echo "$TIMESTAMP,$DISK_STATS" >> "$DISK_FILE"

echo "Metrics collected at $TIMESTAMP"

# Nettoyage des vieux fichiers (> 30 jours)
find "$HISTORY_DIR" -name "*.csv" -mtime +30 -delete
```

**Analyser l'historique :**

```bash
#!/bin/bash
# analyze_history.sh

HISTORY_DIR="/var/lib/postgresql_history"
DATE=$(date '+%Y%m%d')

echo "=========================================="
echo "PostgreSQL Historical Analysis"
echo "=========================================="

# Connexions - Max today
echo "📊 Connexions (today):"
if [ -f "$HISTORY_DIR/connections_${DATE}.csv" ]; then
    MAX_CONN=$(tail -n +2 "$HISTORY_DIR/connections_${DATE}.csv" | cut -d',' -f2 | sort -rn | head -1)
    AVG_CONN=$(tail -n +2 "$HISTORY_DIR/connections_${DATE}.csv" | cut -d',' -f2 | awk '{sum+=$1} END {print int(sum/NR)}')
    echo "  Max: $MAX_CONN"
    echo "  Avg: $AVG_CONN"
fi

# Cache - Min today
echo ""
echo "📊 Cache Hit Ratio (today):"
if [ -f "$HISTORY_DIR/cache_${DATE}.csv" ]; then
    MIN_CACHE=$(tail -n +2 "$HISTORY_DIR/cache_${DATE}.csv" | cut -d',' -f2 | sort -n | head -1)
    AVG_CACHE=$(tail -n +2 "$HISTORY_DIR/cache_${DATE}.csv" | cut -d',' -f2 | awk '{sum+=$1} END {printf "%.2f", sum/NR}')
    echo "  Min: ${MIN_CACHE}%"
    echo "  Avg: ${AVG_CACHE}%"
fi

# Queries - Max long queries
echo ""
echo "📊 Long Queries (today):"
if [ -f "$HISTORY_DIR/queries_${DATE}.csv" ]; then
    MAX_LONG=$(tail -n +2 "$HISTORY_DIR/queries_${DATE}.csv" | cut -d',' -f3 | sort -rn | head -1)
    echo "  Max: $MAX_LONG"
fi

# Disk - Trend
echo ""
echo "📊 Disk Usage (today):"
if [ -f "$HISTORY_DIR/disk_${DATE}.csv" ]; then
    FIRST_USAGE=$(tail -n +2 "$HISTORY_DIR/disk_${DATE}.csv" | head -1 | cut -d',' -f5)
    LAST_USAGE=$(tail -n 1 "$HISTORY_DIR/disk_${DATE}.csv" | cut -d',' -f5)
    DIFF=$(echo "$LAST_USAGE - $FIRST_USAGE" | bc)
    echo "  Start: ${FIRST_USAGE}%"
    echo "  Current: ${LAST_USAGE}%"
    echo "  Trend: ${DIFF}%"

    if (( $(echo "$DIFF > 1" | bc -l) )); then
        echo "  ⚠️ Disk usage increasing significantly"
    fi
fi

echo "=========================================="
```

---

## 6. Monitoring des Performances

### 6.1. Script de Profiling des Requêtes

```bash
#!/bin/bash
# profile_queries.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Query Profiling"
echo "=========================================="

# Vérifier pg_stat_statements
PSS=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_extension WHERE extname = 'pg_stat_statements';
" | xargs)

if [ $PSS -eq 0 ]; then
    echo "❌ pg_stat_statements not installed"
    echo "Run: CREATE EXTENSION pg_stat_statements;"
    exit 1
fi

# Top 10 requêtes par temps total
echo ""
echo "🐌 Top 10 Slowest Queries (Total Time):"
psql -d "$DB_NAME" -c "
    SELECT
        calls,
        ROUND(total_exec_time::numeric, 2) as total_ms,
        ROUND(mean_exec_time::numeric, 2) as avg_ms,
        ROUND((100.0 * total_exec_time / SUM(total_exec_time) OVER())::numeric, 2) as pct_total,
        LEFT(query, 70) as query_preview
    FROM pg_stat_statements
    ORDER BY total_exec_time DESC
    LIMIT 10;
"

# Top 10 requêtes les plus fréquentes
echo ""
echo "🔥 Top 10 Most Frequent Queries:"
psql -d "$DB_NAME" -c "
    SELECT
        calls,
        ROUND(mean_exec_time::numeric, 2) as avg_ms,
        ROUND(total_exec_time::numeric, 2) as total_ms,
        LEFT(query, 70) as query_preview
    FROM pg_stat_statements
    ORDER BY calls DESC
    LIMIT 10;
"

# Requêtes avec temps moyen élevé
echo ""
echo "⏱️  Queries with High Average Time (>1s):"
psql -d "$DB_NAME" -c "
    SELECT
        calls,
        ROUND(mean_exec_time::numeric, 2) as avg_ms,
        ROUND(max_exec_time::numeric, 2) as max_ms,
        LEFT(query, 70) as query_preview
    FROM pg_stat_statements
    WHERE mean_exec_time > 1000
    ORDER BY mean_exec_time DESC
    LIMIT 10;
"

# Statistiques globales
echo ""
echo "📊 Global Statistics:"
psql -d "$DB_NAME" -t -c "
    SELECT
        'Total Queries: ' || SUM(calls) || CHR(10) ||
        'Total Time: ' || ROUND(SUM(total_exec_time)/1000, 2) || 's' || CHR(10) ||
        'Avg Time: ' || ROUND(AVG(mean_exec_time), 2) || 'ms'
    FROM pg_stat_statements;
"

echo "=========================================="
```

### 6.2. Script d'Analyse des Index

```bash
#!/bin/bash
# analyze_indexes.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Index Analysis"
echo "=========================================="

# Index non utilisés
echo "🗑️  Unused Indexes (0 scans):"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        indexname,
        pg_size_pretty(pg_relation_size(schemaname||'.'||indexname)) as index_size
    FROM pg_stat_user_indexes
    WHERE idx_scan = 0
    AND schemaname NOT IN ('pg_catalog', 'information_schema')
    ORDER BY pg_relation_size(schemaname||'.'||indexname) DESC
    LIMIT 10;
"

# Index rarement utilisés
echo ""
echo "⚠️  Rarely Used Indexes (<10 scans):"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        indexname,
        idx_scan,
        pg_size_pretty(pg_relation_size(schemaname||'.'||indexname)) as index_size
    FROM pg_stat_user_indexes
    WHERE idx_scan > 0 AND idx_scan < 10
    AND schemaname NOT IN ('pg_catalog', 'information_schema')
    ORDER BY idx_scan
    LIMIT 10;
"

# Index les plus utilisés
echo ""
echo "✅ Most Used Indexes:"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        indexname,
        idx_scan,
        idx_tup_read,
        idx_tup_fetch,
        pg_size_pretty(pg_relation_size(schemaname||'.'||indexname)) as index_size
    FROM pg_stat_user_indexes
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    ORDER BY idx_scan DESC
    LIMIT 10;
"

# Taille totale des index
echo ""
echo "📊 Total Index Size:"
psql -d "$DB_NAME" -t -c "
    SELECT pg_size_pretty(SUM(pg_relation_size(schemaname||'.'||indexname)))
    FROM pg_stat_user_indexes
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema');
"

# Index invalides
echo ""
echo "❌ Invalid Indexes:"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        indexname
    FROM pg_indexes
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    AND indexname IN (
        SELECT indexrelname
        FROM pg_index i
        JOIN pg_class c ON i.indexrelid = c.oid
        WHERE NOT indisvalid
    );
"

echo "=========================================="
```

### 6.3. Script d'Analyse du Bloat

```bash
#!/bin/bash
# analyze_bloat.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Bloat Analysis"
echo "=========================================="

# Table bloat estimation
echo "🗂️  Table Bloat (Top 10):"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
        n_dead_tup,
        n_live_tup,
        ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 0
    ORDER BY n_dead_tup DESC
    LIMIT 10;
"

# Tables nécessitant VACUUM
echo ""
echo "🧹 Tables Needing VACUUM (>10% dead tuples):"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        n_dead_tup,
        n_live_tup,
        ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct,
        last_vacuum,
        last_autovacuum
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 0
    AND (100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0)) > 10
    ORDER BY dead_pct DESC;
"

# Index bloat estimation
echo ""
echo "📇 Index Bloat (estimated):"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        indexname,
        pg_size_pretty(pg_relation_size(schemaname||'.'||indexname)) as index_size,
        idx_scan,
        idx_tup_read,
        idx_tup_fetch
    FROM pg_stat_user_indexes
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    ORDER BY pg_relation_size(schemaname||'.'||indexname) DESC
    LIMIT 10;
"

# Recommandations
echo ""
echo "💡 Recommendations:"

BLOATED_TABLES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_user_tables
    WHERE (100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0)) > 10;
" | xargs)

if [ $BLOATED_TABLES -gt 0 ]; then
    echo "  ⚠️ $BLOATED_TABLES table(s) with significant bloat"
    echo "  Action: Run VACUUM ANALYZE on these tables"
    echo "  Command: VACUUM ANALYZE;"
else
    echo "  ✅ No significant bloat detected"
fi

echo "=========================================="
```

---

## 7. Monitoring des Ressources

### 7.1. Script de Monitoring Complet des Ressources

```bash
#!/bin/bash
# monitor_resources.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Resource Monitoring"
echo "=========================================="
echo "Date: $(date '+%Y-%m-%d %H:%M:%S')"
echo ""

# CPU
echo "🖥️  CPU:"
CPU_USAGE=$(top -b -n 1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
LOAD=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
CPU_COUNT=$(nproc)
echo "  Usage: ${CPU_USAGE}%"
echo "  Load (1min): $LOAD"
echo "  CPUs: $CPU_COUNT"
echo "  Load per CPU: $(echo "scale=2; $LOAD / $CPU_COUNT" | bc)"

# Memory
echo ""
echo "💾 Memory:"
free -h | awk 'NR==2 {print "  Total: "$2"\n  Used: "$3" ("$3/$2*100"%)\n  Available: "$7}'

SWAP_USED=$(free -h | awk 'NR==3 {print $3}')
if [ "$SWAP_USED" != "0B" ]; then
    echo "  ⚠️ Swap used: $SWAP_USED"
fi

# Disk
echo ""
echo "💿 Disk:"
PGDATA=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" | xargs)
df -h "$PGDATA" | tail -1 | awk '{print "  Filesystem: "$1"\n  Total: "$2"\n  Used: "$3" ("$5")\n  Available: "$4}'

# Network
echo ""
echo "🌐 Network:"
PG_PORT=$(psql -d "$DB_NAME" -t -c "SHOW port;" | xargs)
TOTAL_CONNECTIONS=$(netstat -tan | grep ":$PG_PORT" | wc -l)
ESTABLISHED=$(netstat -tan | grep ":$PG_PORT" | grep ESTABLISHED | wc -l)
echo "  Total connections to port $PG_PORT: $TOTAL_CONNECTIONS"
echo "  Established: $ESTABLISHED"

# PostgreSQL Memory Usage
echo ""
echo "🐘 PostgreSQL Memory:"
PG_MEM=$(ps aux | grep postgres | awk '{sum+=$6} END {print sum/1024}')
echo "  Total: ${PG_MEM} MB"

SHARED_BUFFERS=$(psql -d "$DB_NAME" -t -c "SHOW shared_buffers;" | xargs)
WORK_MEM=$(psql -d "$DB_NAME" -t -c "SHOW work_mem;" | xargs)
MAINTENANCE_WORK_MEM=$(psql -d "$DB_NAME" -t -c "SHOW maintenance_work_mem;" | xargs)

echo "  shared_buffers: $SHARED_BUFFERS"
echo "  work_mem: $WORK_MEM"
echo "  maintenance_work_mem: $MAINTENANCE_WORK_MEM"

# I/O Stats
echo ""
echo "📊 I/O Statistics:"
if command -v iostat > /dev/null 2>&1; then
    iostat -x 1 2 | tail -n +4 | grep -E "^[a-z]" | head -1 | awk '{print "  Device: "$1"\n  Reads/s: "$4"\n  Writes/s: "$5"\n  Util%: "$NF}'
else
    echo "  iostat not available (install sysstat)"
fi

# Resource Limits
echo ""
echo "⚙️  PostgreSQL Configuration:"
psql -d "$DB_NAME" -c "
    SELECT
        name,
        setting,
        unit,
        context
    FROM pg_settings
    WHERE name IN (
        'max_connections',
        'shared_buffers',
        'work_mem',
        'maintenance_work_mem',
        'effective_cache_size',
        'max_worker_processes',
        'max_parallel_workers'
    )
    ORDER BY name;
"

echo "=========================================="
```

### 7.2. Script de Prédiction de Croissance

```bash
#!/bin/bash
# predict_growth.sh

DB_NAME="postgres"

echo "=========================================="
echo "PostgreSQL Growth Prediction"
echo "=========================================="

# Taille actuelle des bases
echo "📊 Current Database Sizes:"
psql -d "$DB_NAME" -c "
    SELECT
        datname,
        pg_size_pretty(pg_database_size(datname)) as current_size,
        pg_database_size(datname) as size_bytes
    FROM pg_database
    WHERE datistemplate = false
    ORDER BY pg_database_size(datname) DESC;
"

# Si historique disponible
HISTORY_DIR="/var/lib/postgresql_history"
DATE=$(date '+%Y%m%d')
YESTERDAY=$(date -d "yesterday" '+%Y%m%d')

if [ -f "$HISTORY_DIR/disk_${YESTERDAY}.csv" ] && [ -f "$HISTORY_DIR/disk_${DATE}.csv" ]; then
    YESTERDAY_SIZE=$(tail -1 "$HISTORY_DIR/disk_${YESTERDAY}.csv" | cut -d',' -f3)
    TODAY_SIZE=$(tail -1 "$HISTORY_DIR/disk_${DATE}.csv" | cut -d',' -f3)
    DAILY_GROWTH=$(echo "$TODAY_SIZE - $YESTERDAY_SIZE" | bc)

    echo ""
    echo "📈 Growth Analysis:"
    echo "  Yesterday: ${YESTERDAY_SIZE}GB"
    echo "  Today: ${TODAY_SIZE}GB"
    echo "  Daily growth: ${DAILY_GROWTH}GB"

    if (( $(echo "$DAILY_GROWTH > 0" | bc -l) )); then
        # Projection
        CURRENT_FREE=$(tail -1 "$HISTORY_DIR/disk_${DATE}.csv" | cut -d',' -f4)
        DAYS_UNTIL_FULL=$(echo "scale=0; $CURRENT_FREE / $DAILY_GROWTH" | bc)

        echo ""
        echo "📅 Prediction:"
        echo "  Current free space: ${CURRENT_FREE}GB"
        echo "  Days until full (at current rate): $DAYS_UNTIL_FULL days"

        if [ $DAYS_UNTIL_FULL -lt 30 ]; then
            echo "  ⚠️ WARNING: Disk will be full in less than a month!"
        elif [ $DAYS_UNTIL_FULL -lt 90 ]; then
            echo "  ⚠️ NOTICE: Monitor disk usage closely"
        else
            echo "  ✅ Sufficient space for near future"
        fi
    fi
fi

# Top tables par taille
echo ""
echo "📦 Top 10 Largest Tables:"
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
        pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as indexes_size
    FROM pg_tables
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
    LIMIT 10;
"

echo "=========================================="
```

---

## 8. Monitoring de la Santé

### 8.1. Script de Diagnostic Complet

```bash
#!/bin/bash
# diagnostic_complete.sh

DB_NAME="postgres"
REPORT_FILE="/tmp/postgresql_diagnostic_$(date +%Y%m%d_%H%M%S).txt"

exec > >(tee -a "$REPORT_FILE")
exec 2>&1

echo "=========================================="
echo "PostgreSQL Complete Diagnostic Report"
echo "=========================================="
echo "Generated: $(date '+%Y-%m-%d %H:%M:%S')"
echo "Hostname: $(hostname)"
echo ""

# 1. Service Status
echo "===== 1. SERVICE STATUS ====="
if pg_isready -d "$DB_NAME" > /dev/null 2>&1; then
    echo "✅ PostgreSQL is running"

    # Version et uptime
    psql -d "$DB_NAME" -c "SELECT version();"
    echo ""
    psql -d "$DB_NAME" -t -c "SELECT 'Uptime: ' || (NOW() - pg_postmaster_start_time());"
else
    echo "❌ PostgreSQL is DOWN"
    exit 1
fi
echo ""

# 2. System Resources
echo "===== 2. SYSTEM RESOURCES ====="
echo "--- CPU ---"
top -b -n 1 | grep "Cpu(s)"
uptime
echo ""

echo "--- Memory ---"
free -h
echo ""

echo "--- Disk ---"
PGDATA=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" | xargs)
df -h "$PGDATA"
echo ""

# 3. PostgreSQL Configuration
echo "===== 3. POSTGRESQL CONFIGURATION ====="
psql -d "$DB_NAME" -c "
    SELECT
        name,
        setting,
        unit,
        source
    FROM pg_settings
    WHERE name IN (
        'max_connections',
        'shared_buffers',
        'effective_cache_size',
        'work_mem',
        'maintenance_work_mem',
        'checkpoint_completion_target',
        'wal_buffers',
        'default_statistics_target',
        'random_page_cost',
        'effective_io_concurrency',
        'max_worker_processes',
        'max_parallel_workers_per_gather',
        'max_parallel_workers'
    )
    ORDER BY name;
"
echo ""

# 4. Connections
echo "===== 4. CONNECTIONS ====="
psql -d "$DB_NAME" -c "
    SELECT
        state,
        COUNT(*) as count
    FROM pg_stat_activity
    GROUP BY state
    ORDER BY count DESC;
"
echo ""

# 5. Database Sizes
echo "===== 5. DATABASE SIZES ====="
psql -d "$DB_NAME" -c "
    SELECT
        datname,
        pg_size_pretty(pg_database_size(datname)) as size,
        age(datfrozenxid) as xid_age
    FROM pg_database
    WHERE datistemplate = false
    ORDER BY pg_database_size(datname) DESC;
"
echo ""

# 6. Top Tables
echo "===== 6. TOP 10 LARGEST TABLES ====="
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
        n_live_tup,
        n_dead_tup
    FROM pg_stat_user_tables
    ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
    LIMIT 10;
"
echo ""

# 7. Performance Metrics
echo "===== 7. PERFORMANCE METRICS ====="
echo "--- Cache Hit Ratio ---"
psql -d "$DB_NAME" -c "
    SELECT
        'Cache Hit Ratio: ' ||
        ROUND(
            100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0),
            2
        ) || '%'
    FROM pg_statio_user_tables;
"
echo ""

echo "--- Transaction Statistics ---"
psql -d "$DB_NAME" -c "
    SELECT
        datname,
        xact_commit,
        xact_rollback,
        ROUND(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2) as rollback_pct,
        deadlocks
    FROM pg_stat_database
    WHERE datname NOT IN ('template0', 'template1')
    ORDER BY xact_commit DESC;
"
echo ""

# 8. Slow Queries
echo "===== 8. ACTIVE QUERIES ====="
psql -d "$DB_NAME" -c "
    SELECT
        pid,
        usename,
        datname,
        state,
        ROUND(EXTRACT(EPOCH FROM (NOW() - query_start))::numeric, 2) as duration_sec,
        LEFT(query, 100) as query_preview
    FROM pg_stat_activity
    WHERE state = 'active'
    AND query NOT LIKE '%pg_stat_activity%'
    ORDER BY query_start;
"
echo ""

# 9. Locks
echo "===== 9. LOCKS ====="
psql -d "$DB_NAME" -c "
    SELECT
        locktype,
        mode,
        COUNT(*) as count
    FROM pg_locks
    GROUP BY locktype, mode
    ORDER BY count DESC;
"
echo ""

# 10. Bloat
echo "===== 10. BLOAT ANALYSIS ====="
psql -d "$DB_NAME" -c "
    SELECT
        schemaname,
        tablename,
        n_dead_tup,
        n_live_tup,
        ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 1000
    ORDER BY dead_pct DESC
    LIMIT 10;
"
echo ""

# 11. Replication (if applicable)
echo "===== 11. REPLICATION STATUS ====="
REPLICATION_COUNT=$(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_replication;" | xargs)
if [ $REPLICATION_COUNT -gt 0 ]; then
    psql -d "$DB_NAME" -c "
        SELECT
            application_name,
            client_addr,
            state,
            sync_state,
            pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes
        FROM pg_stat_replication;
    "
else
    echo "No replication configured"
fi
echo ""

# 12. Recommendations
echo "===== 12. RECOMMENDATIONS ====="

# Check cache hit ratio
CACHE_RATIO=$(psql -d "$DB_NAME" -t -c "
    SELECT COALESCE(
        ROUND(100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2),
        100
    )
    FROM pg_statio_user_tables;
" | xargs)

if (( $(echo "$CACHE_RATIO < 95" | bc -l) )); then
    echo "⚠️ Cache hit ratio is low (${CACHE_RATIO}%)"
    echo "   Recommendation: Increase shared_buffers"
fi

# Check connections
CONN_PCT=$(psql -d "$DB_NAME" -t -c "
    SELECT ROUND(100.0 * COUNT(*) / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2)
    FROM pg_stat_activity;
" | xargs)

if (( $(echo "$CONN_PCT > 80" | bc -l) )); then
    echo "⚠️ Connection usage is high (${CONN_PCT}%)"
    echo "   Recommendation: Consider connection pooling (PgBouncer)"
fi

# Check bloat
BLOATED_TABLES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_user_tables
    WHERE (100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0)) > 10;
" | xargs)

if [ $BLOATED_TABLES -gt 0 ]; then
    echo "⚠️ $BLOATED_TABLES table(s) have significant bloat (>10% dead tuples)"
    echo "   Recommendation: Run VACUUM ANALYZE"
fi

# Check XID age
MAX_XID_AGE=$(psql -d "$DB_NAME" -t -c "
    SELECT MAX(age(datfrozenxid)) FROM pg_database;
" | xargs)

if [ $MAX_XID_AGE -gt 1500000000 ]; then
    echo "❌ XID age is dangerously high ($MAX_XID_AGE)"
    echo "   Recommendation: Run VACUUM FREEZE immediately"
elif [ $MAX_XID_AGE -gt 1000000000 ]; then
    echo "⚠️ XID age is elevated ($MAX_XID_AGE)"
    echo "   Recommendation: Schedule VACUUM FREEZE soon"
fi

echo ""
echo "=========================================="
echo "Report saved to: $REPORT_FILE"
echo "=========================================="
```

---

## 9. Alertes et Seuils

### 9.1. Script de Gestion des Alertes

```bash
#!/bin/bash
# alert_manager.sh

DB_NAME="postgres"
ALERT_LOG="/var/log/postgresql_alerts.log"
ALERT_EMAIL="admin@example.com"

# Seuils configurables
THRESHOLD_CPU=80
THRESHOLD_MEMORY=85
THRESHOLD_DISK=85
THRESHOLD_CONNECTIONS=80
THRESHOLD_CACHE_HIT=95
THRESHOLD_LONG_QUERIES=5
THRESHOLD_REPLICATION_LAG=60

ALERTS=()
CRITICAL_ALERTS=()

log_alert() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" >> "$ALERT_LOG"
}

check_cpu() {
    local cpu_usage=$(top -b -n 1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)

    if (( $(echo "$cpu_usage > $THRESHOLD_CPU" | bc -l) )); then
        ALERTS+=("CPU usage high: ${cpu_usage}% (threshold: ${THRESHOLD_CPU}%)")
        log_alert "WARNING" "CPU usage: ${cpu_usage}%"

        if (( $(echo "$cpu_usage > 95" | bc -l) )); then
            CRITICAL_ALERTS+=("CPU usage critical: ${cpu_usage}%")
            log_alert "CRITICAL" "CPU usage: ${cpu_usage}%"
        fi
    fi
}

check_memory() {
    local mem_usage=$(free | awk 'NR==2 {print int($3*100/$2)}')

    if [ $mem_usage -gt $THRESHOLD_MEMORY ]; then
        ALERTS+=("Memory usage high: ${mem_usage}% (threshold: ${THRESHOLD_MEMORY}%)")
        log_alert "WARNING" "Memory usage: ${mem_usage}%"
    fi

    # Check swap
    local swap_used=$(free -m | awk 'NR==3 {print $3}')
    if [ $swap_used -gt 100 ]; then
        ALERTS+=("Swap being used: ${swap_used}MB")
        log_alert "WARNING" "Swap usage: ${swap_used}MB"
    fi
}

check_disk() {
    local pgdata=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" 2>/dev/null | xargs)
    local disk_usage=$(df "$pgdata" 2>/dev/null | tail -1 | awk '{print $5}' | sed 's/%//')

    if [ -n "$disk_usage" ] && [ $disk_usage -gt $THRESHOLD_DISK ]; then
        ALERTS+=("Disk usage high: ${disk_usage}% (threshold: ${THRESHOLD_DISK}%)")
        log_alert "WARNING" "Disk usage: ${disk_usage}%"

        if [ $disk_usage -gt 95 ]; then
            CRITICAL_ALERTS+=("Disk usage critical: ${disk_usage}%")
            log_alert "CRITICAL" "Disk usage: ${disk_usage}%"
        fi
    fi
}

check_postgresql() {
    if ! pg_isready -d "$DB_NAME" > /dev/null 2>&1; then
        CRITICAL_ALERTS+=("PostgreSQL is DOWN")
        log_alert "CRITICAL" "PostgreSQL is DOWN"
        return 1
    fi

    # Connections
    local conn_pct=$(psql -d "$DB_NAME" -t -c "
        SELECT ROUND(100.0 * COUNT(*) / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2)
        FROM pg_stat_activity;
    " 2>/dev/null | xargs)

    if [ -n "$conn_pct" ] && (( $(echo "$conn_pct > $THRESHOLD_CONNECTIONS" | bc -l) )); then
        ALERTS+=("Connection usage high: ${conn_pct}% (threshold: ${THRESHOLD_CONNECTIONS}%)")
        log_alert "WARNING" "Connection usage: ${conn_pct}%"
    fi

    # Cache hit ratio
    local cache_ratio=$(psql -d "$DB_NAME" -t -c "
        SELECT COALESCE(
            ROUND(100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2),
            100
        )
        FROM pg_statio_user_tables;
    " 2>/dev/null | xargs)

    if [ -n "$cache_ratio" ] && (( $(echo "$cache_ratio < $THRESHOLD_CACHE_HIT" | bc -l) )); then
        ALERTS+=("Cache hit ratio low: ${cache_ratio}% (threshold: ${THRESHOLD_CACHE_HIT}%)")
        log_alert "WARNING" "Cache hit ratio: ${cache_ratio}%"
    fi

    # Long queries
    local long_queries=$(psql -d "$DB_NAME" -t -c "
        SELECT COUNT(*) FROM pg_stat_activity
        WHERE state = 'active'
        AND query_start < NOW() - INTERVAL '1 minute'
        AND query NOT LIKE '%pg_stat_activity%';
    " 2>/dev/null | xargs)

    if [ -n "$long_queries" ] && [ $long_queries -gt $THRESHOLD_LONG_QUERIES ]; then
        ALERTS+=("Long running queries: $long_queries (threshold: $THRESHOLD_LONG_QUERIES)")
        log_alert "WARNING" "Long queries: $long_queries"
    fi

    # Idle in transaction
    local idle_in_txn=$(psql -d "$DB_NAME" -t -c "
        SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'idle in transaction';
    " 2>/dev/null | xargs)

    if [ -n "$idle_in_txn" ] && [ $idle_in_txn -gt 0 ]; then
        ALERTS+=("Connections idle in transaction: $idle_in_txn")
        log_alert "WARNING" "Idle in transaction: $idle_in_txn"
    fi

    # Replication lag
    local replication_count=$(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_replication;" 2>/dev/null | xargs)
    if [ -n "$replication_count" ] && [ $replication_count -gt 0 ]; then
        local max_lag=$(psql -d "$DB_NAME" -t -c "
            SELECT COALESCE(MAX(EXTRACT(EPOCH FROM (NOW() - replay_lag))), 0)
            FROM pg_stat_replication;
        " 2>/dev/null | xargs)

        if [ -n "$max_lag" ] && (( $(echo "$max_lag > $THRESHOLD_REPLICATION_LAG" | bc -l) )); then
            ALERTS+=("Replication lag high: ${max_lag}s (threshold: ${THRESHOLD_REPLICATION_LAG}s)")
            log_alert "WARNING" "Replication lag: ${max_lag}s"
        fi
    fi
}

send_alerts() {
    if [ ${#CRITICAL_ALERTS[@]} -gt 0 ]; then
        local subject="🚨 CRITICAL: PostgreSQL Alerts"
        local body="CRITICAL ALERTS:\n$(printf "%s\n" "${CRITICAL_ALERTS[@]}")\n\n"

        if [ ${#ALERTS[@]} -gt 0 ]; then
            body+="OTHER ALERTS:\n$(printf "%s\n" "${ALERTS[@]}")"
        fi

        echo -e "$body" | mail -s "$subject" "$ALERT_EMAIL"
        log_alert "EMAIL" "Critical alert email sent"

    elif [ ${#ALERTS[@]} -gt 0 ]; then
        local subject="⚠️ PostgreSQL Alerts"
        local body=$(printf "%s\n" "${ALERTS[@]}")

        echo "$body" | mail -s "$subject" "$ALERT_EMAIL"
        log_alert "EMAIL" "Alert email sent"
    fi
}

# Exécution
check_cpu
check_memory
check_disk
check_postgresql

# Résumé
if [ ${#CRITICAL_ALERTS[@]} -gt 0 ] || [ ${#ALERTS[@]} -gt 0 ]; then
    echo "=========================================="
    echo "PostgreSQL Alert Summary"
    echo "=========================================="
    echo "Time: $(date '+%Y-%m-%d %H:%M:%S')"
    echo ""

    if [ ${#CRITICAL_ALERTS[@]} -gt 0 ]; then
        echo "🚨 CRITICAL ALERTS:"
        printf "%s\n" "${CRITICAL_ALERTS[@]}"
        echo ""
    fi

    if [ ${#ALERTS[@]} -gt 0 ]; then
        echo "⚠️ WARNINGS:"
        printf "%s\n" "${ALERTS[@]}"
        echo ""
    fi

    send_alerts
    echo "=========================================="
    exit 1
else
    echo "✅ All checks passed - No alerts"
    exit 0
fi
```

**Automatiser :**

```bash
# Vérifier toutes les 5 minutes
*/5 * * * * /usr/local/bin/alert_manager.sh
```

---

## 10. Tableaux de Bord et Rapports

### 10.1. Script de Rapport Quotidien

```bash
#!/bin/bash
# daily_report.sh

DB_NAME="postgres"
REPORT_DATE=$(date '+%Y-%m-%d')
REPORT_FILE="/var/reports/postgresql_daily_${REPORT_DATE}.html"

mkdir -p "$(dirname "$REPORT_FILE")"

# Générer HTML
cat > "$REPORT_FILE" << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>PostgreSQL Daily Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { color: #336791; }
        h2 { color: #447799; border-bottom: 2px solid #336791; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th { background-color: #336791; color: white; padding: 10px; text-align: left; }
        td { padding: 8px; border: 1px solid #ddd; }
        tr:nth-child(even) { background-color: #f2f2f2; }
        .ok { color: green; font-weight: bold; }
        .warning { color: orange; font-weight: bold; }
        .critical { color: red; font-weight: bold; }
        .metric { background-color: #f9f9f9; padding: 15px; margin: 10px 0; border-left: 4px solid #336791; }
    </style>
</head>
<body>
EOF

echo "<h1>PostgreSQL Daily Report</h1>" >> "$REPORT_FILE"
echo "<p>Date: $REPORT_DATE</p>" >> "$REPORT_FILE"
echo "<p>Server: $(hostname)</p>" >> "$REPORT_FILE"

# Section 1: Summary
echo "<h2>1. Summary</h2>" >> "$REPORT_FILE"
echo "<div class='metric'>" >> "$REPORT_FILE"
echo "<p><strong>PostgreSQL Status:</strong> " >> "$REPORT_FILE"
if pg_isready -d "$DB_NAME" > /dev/null 2>&1; then
    echo "<span class='ok'>✅ Running</span></p>" >> "$REPORT_FILE"
else
    echo "<span class='critical'>❌ DOWN</span></p>" >> "$REPORT_FILE"
fi

UPTIME=$(psql -d "$DB_NAME" -t -c "SELECT NOW() - pg_postmaster_start_time();" 2>/dev/null | xargs)
echo "<p><strong>Uptime:</strong> $UPTIME</p>" >> "$REPORT_FILE"
echo "</div>" >> "$REPORT_FILE"

# Section 2: Resource Usage
echo "<h2>2. Resource Usage</h2>" >> "$REPORT_FILE"
echo "<table>" >> "$REPORT_FILE"
echo "<tr><th>Resource</th><th>Current</th><th>Status</th></tr>" >> "$REPORT_FILE"

# CPU
CPU_USAGE=$(top -b -n 1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
CPU_STATUS="ok"
[ $(echo "$CPU_USAGE > 80" | bc) -eq 1 ] && CPU_STATUS="warning"
[ $(echo "$CPU_USAGE > 95" | bc) -eq 1 ] && CPU_STATUS="critical"
echo "<tr><td>CPU</td><td>${CPU_USAGE}%</td><td class='$CPU_STATUS'>$(echo $CPU_STATUS | tr '[:lower:]' '[:upper:]')</td></tr>" >> "$REPORT_FILE"

# Memory
MEM_USAGE=$(free | awk 'NR==2 {print int($3*100/$2)}')
MEM_STATUS="ok"
[ $MEM_USAGE -gt 80 ] && MEM_STATUS="warning"
[ $MEM_USAGE -gt 95 ] && MEM_STATUS="critical"
echo "<tr><td>Memory</td><td>${MEM_USAGE}%</td><td class='$MEM_STATUS'>$(echo $MEM_STATUS | tr '[:lower:]' '[:upper:]')</td></tr>" >> "$REPORT_FILE"

# Disk
PGDATA=$(psql -d "$DB_NAME" -t -c "SHOW data_directory;" 2>/dev/null | xargs)
DISK_USAGE=$(df "$PGDATA" 2>/dev/null | tail -1 | awk '{print $5}' | sed 's/%//')
DISK_STATUS="ok"
[ $DISK_USAGE -gt 80 ] && DISK_STATUS="warning"
[ $DISK_USAGE -gt 90 ] && DISK_STATUS="critical"
echo "<tr><td>Disk</td><td>${DISK_USAGE}%</td><td class='$DISK_STATUS'>$(echo $DISK_STATUS | tr '[:lower:]' '[:upper:]')</td></tr>" >> "$REPORT_FILE"

echo "</table>" >> "$REPORT_FILE"

# Section 3: Database Statistics
echo "<h2>3. Database Statistics</h2>" >> "$REPORT_FILE"
echo "<table>" >> "$REPORT_FILE"
echo "<tr><th>Database</th><th>Size</th><th>Connections</th><th>Transactions</th></tr>" >> "$REPORT_FILE"

psql -d "$DB_NAME" -t -c "
    SELECT
        d.datname,
        pg_size_pretty(pg_database_size(d.datname)),
        (SELECT COUNT(*) FROM pg_stat_activity WHERE datname = d.datname),
        s.xact_commit + s.xact_rollback
    FROM pg_database d
    LEFT JOIN pg_stat_database s ON d.datname = s.datname
    WHERE d.datistemplate = false
    ORDER BY pg_database_size(d.datname) DESC;
" 2>/dev/null | while read line; do
    echo "<tr><td>$(echo "$line" | awk '{print $1}')</td><td>$(echo "$line" | awk '{print $2}')</td><td>$(echo "$line" | awk '{print $3}')</td><td>$(echo "$line" | awk '{print $4}')</td></tr>" >> "$REPORT_FILE"
done

echo "</table>" >> "$REPORT_FILE"

# Section 4: Performance Metrics
echo "<h2>4. Performance Metrics</h2>" >> "$REPORT_FILE"

CACHE_RATIO=$(psql -d "$DB_NAME" -t -c "
    SELECT COALESCE(
        ROUND(100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2),
        100
    )
    FROM pg_statio_user_tables;
" 2>/dev/null | xargs)

echo "<div class='metric'>" >> "$REPORT_FILE"
echo "<p><strong>Cache Hit Ratio:</strong> ${CACHE_RATIO}%</p>" >> "$REPORT_FILE"
echo "</div>" >> "$REPORT_FILE"

# Section 5: Recommendations
echo "<h2>5. Recommendations</h2>" >> "$REPORT_FILE"
echo "<ul>" >> "$REPORT_FILE"

# Check cache
if (( $(echo "$CACHE_RATIO < 95" | bc -l) )); then
    echo "<li class='warning'>⚠️ Cache hit ratio is low. Consider increasing shared_buffers.</li>" >> "$REPORT_FILE"
fi

# Check bloat
BLOATED_TABLES=$(psql -d "$DB_NAME" -t -c "
    SELECT COUNT(*) FROM pg_stat_user_tables
    WHERE (100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0)) > 10;
" 2>/dev/null | xargs)

if [ $BLOATED_TABLES -gt 0 ]; then
    echo "<li class='warning'>⚠️ $BLOATED_TABLES table(s) have significant bloat. Run VACUUM ANALYZE.</li>" >> "$REPORT_FILE"
fi

# If no issues
if (( $(echo "$CACHE_RATIO >= 95" | bc -l) )) && [ $BLOATED_TABLES -eq 0 ]; then
    echo "<li class='ok'>✅ No issues detected. System running optimally.</li>" >> "$REPORT_FILE"
fi

echo "</ul>" >> "$REPORT_FILE"

# Footer
echo "<hr>" >> "$REPORT_FILE"
echo "<p><small>Generated: $(date '+%Y-%m-%d %H:%M:%S')</small></p>" >> "$REPORT_FILE"
echo "</body></html>" >> "$REPORT_FILE"

echo "Report generated: $REPORT_FILE"

# Envoyer par email
if command -v mail > /dev/null 2>&1; then
    mail -s "PostgreSQL Daily Report - $REPORT_DATE" \
         -a "Content-Type: text/html" \
         admin@example.com < "$REPORT_FILE"
    echo "Report sent via email"
fi
```

**Automatiser :**

```bash
# Tous les jours à 8h
0 8 * * * /usr/local/bin/daily_report.sh
```

---

## 11. Intégration avec des Outils

### 11.1. Export Prometheus

```bash
#!/bin/bash
# export_prometheus.sh

DB_NAME="postgres"
METRICS_DIR="/var/lib/node_exporter/textfile_collector"
METRICS_FILE="$METRICS_DIR/postgresql.prom"

mkdir -p "$METRICS_DIR"

# Générer métriques Prometheus
cat > "$METRICS_FILE" << EOF
# HELP postgresql_up PostgreSQL is up and running
# TYPE postgresql_up gauge
postgresql_up $(pg_isready -d "$DB_NAME" > /dev/null 2>&1 && echo 1 || echo 0)

# HELP postgresql_connections_total Total number of connections
# TYPE postgresql_connections_total gauge
postgresql_connections_total $(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_activity;" 2>/dev/null | xargs || echo 0)

# HELP postgresql_connections_max Maximum number of connections
# TYPE postgresql_connections_max gauge
postgresql_connections_max $(psql -d "$DB_NAME" -t -c "SELECT setting FROM pg_settings WHERE name = 'max_connections';" 2>/dev/null | xargs || echo 0)

# HELP postgresql_cache_hit_ratio Cache hit ratio percentage
# TYPE postgresql_cache_hit_ratio gauge
postgresql_cache_hit_ratio $(psql -d "$DB_NAME" -t -c "SELECT COALESCE(ROUND(100.0 * SUM(heap_blks_hit) / NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2), 100) FROM pg_statio_user_tables;" 2>/dev/null | xargs || echo 0)

# HELP postgresql_database_size_bytes Database size in bytes
# TYPE postgresql_database_size_bytes gauge
EOF

# Taille des bases
psql -d "$DB_NAME" -t -c "
    SELECT
        'postgresql_database_size_bytes{database=\"' || datname || '\"} ' || pg_database_size(datname)
    FROM pg_database
    WHERE datistemplate = false;
" 2>/dev/null >> "$METRICS_FILE"

# Queries actives
ACTIVE_QUERIES=$(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active' AND query NOT LIKE '%pg_stat_activity%';" 2>/dev/null | xargs || echo 0)

cat >> "$METRICS_FILE" << EOF

# HELP postgresql_active_queries Number of active queries
# TYPE postgresql_active_queries gauge
postgresql_active_queries $ACTIVE_QUERIES

# HELP postgresql_long_queries Number of long running queries
# TYPE postgresql_long_queries gauge
postgresql_long_queries $(psql -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active' AND query_start < NOW() - INTERVAL '1 minute' AND query NOT LIKE '%pg_stat_activity%';" 2>/dev/null | xargs || echo 0)
EOF

echo "Prometheus metrics exported to $METRICS_FILE"
```

**Automatiser :**

```bash
# Toutes les 30 secondes
* * * * * /usr/local/bin/export_prometheus.sh
* * * * * sleep 30 && /usr/local/bin/export_prometheus.sh
```

### 11.2. Webhook Slack

```bash
#!/bin/bash
# send_slack.sh

SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
MESSAGE="$1"
COLOR="${2:-#ff0000}"

send_slack() {
    curl -X POST "$SLACK_WEBHOOK_URL" \
        -H 'Content-Type: application/json' \
        -d "{
            \"attachments\": [
                {
                    \"color\": \"$COLOR\",
                    \"title\": \"🐘 PostgreSQL Monitoring\",
                    \"text\": \"$MESSAGE\",
                    \"footer\": \"Server: $(hostname)\",
                    \"ts\": $(date +%s)
                }
            ]
        }"
}

send_slack
```

---

## 12. Bonnes Pratiques

### 12.1. Checklist de Monitoring

**Configuration initiale :**
- [ ] pg_stat_statements installé et activé
- [ ] Collecte de métriques système (CPU, RAM, disque)
- [ ] Scripts de monitoring automatisés (cron)
- [ ] Seuils d'alerte configurés
- [ ] Alertes email/Slack configurées
- [ ] Historique des métriques conservé (30+ jours)
- [ ] Dashboard mis en place

**Surveillance quotidienne :**
- [ ] Vérifier uptime PostgreSQL
- [ ] Consulter le nombre de connexions
- [ ] Vérifier cache hit ratio
- [ ] Identifier requêtes lentes
- [ ] Vérifier espace disque
- [ ] Consulter les logs d'erreur

**Surveillance hebdomadaire :**
- [ ] Analyser tendances de croissance
- [ ] Identifier index non utilisés
- [ ] Vérifier bloat des tables
- [ ] Analyser historique des performances
- [ ] Réviser les seuils d'alerte

**Surveillance mensuelle :**
- [ ] Audit complet des performances
- [ ] Planification de capacité
- [ ] Révision de la configuration
- [ ] Test des alertes
- [ ] Revue des logs

### 12.2. Métriques Essentielles (Golden Signals)

**À surveiller en priorité :**

1. **Latency** (Latence)
   - Temps de réponse des requêtes
   - Objectif : < 100ms (95th percentile)

2. **Traffic** (Trafic)
   - Connexions actives
   - Requêtes par seconde
   - Objectif : < 80% de max_connections

3. **Errors** (Erreurs)
   - Connexions refusées
   - Requêtes échouées
   - Deadlocks
   - Objectif : 0 erreur critique

4. **Saturation** (Saturation)
   - CPU > 80%
   - Disque > 85%
   - Cache hit ratio < 95%
   - Objectif : Ressources < 80%

### 12.3. Architecture de Monitoring Recommandée

```
[PostgreSQL Server]
        │
        ├─→ [Scripts de collecte] (cron toutes les 5 min)
        │       │
        │       ├─→ Métriques système (CPU, RAM, disque)
        │       ├─→ Métriques PostgreSQL (connexions, cache, requêtes)
        │       └─→ Logs d'événements
        │
        ├─→ [Stockage des métriques]
        │       ├─→ CSV local (historique 30 jours)
        │       ├─→ Base de données de métriques (Prometheus/InfluxDB)
        │       └─→ Logs centralisés (ELK Stack)
        │
        ├─→ [Alertes]
        │       ├─→ Email (critiques)
        │       ├─→ Slack/Discord (warnings)
        │       └─→ PagerDuty (incidents majeurs)
        │
        └─→ [Visualisation]
                ├─→ Grafana (dashboards temps réel)
                ├─→ Rapports HTML quotidiens
                └─→ Rapports mensuels détaillés
```

### 12.4. Outils Recommandés

**Open Source :**
- **Prometheus + Grafana** : Métriques et dashboards
- **pgBadger** : Analyse de logs
- **pg_stat_monitor** : Extension de monitoring avancé
- **pgwatch2** : Solution complète de monitoring
- **check_postgres** : Nagios/Icinga checks

**Commercial :**
- **Datadog** : Monitoring complet
- **New Relic** : APM avec PostgreSQL
- **Percona Monitoring and Management (PMM)** : Gratuit, complet

**Cloud :**
- **AWS CloudWatch** (pour RDS)
- **Azure Monitor** (pour Azure Database)
- **Google Cloud Monitoring** (pour Cloud SQL)

---

## Conclusion

### Ce que vous avez appris

- ✅ **Introduction au monitoring** : Concepts, méthodologie, 3 piliers
- ✅ **Métriques système** : CPU, RAM, disque, réseau
- ✅ **Métriques PostgreSQL** : Connexions, requêtes, cache, transactions
- ✅ **Scripts de base** : Health check, collecte de métriques
- ✅ **Scripts avancés** : Monitoring complet, historique
- ✅ **Performance** : Profiling, index, bloat
- ✅ **Ressources** : Monitoring complet, prédiction de croissance
- ✅ **Santé** : Diagnostic complet, recommandations
- ✅ **Alertes** : Seuils, gestion, notifications
- ✅ **Rapports** : Dashboard HTML, rapports quotidiens
- ✅ **Intégration** : Prometheus, Slack, outils tiers
- ✅ **Bonnes pratiques** : Checklist, métriques essentielles

### Points Clés à Retenir

1. **Monitorer proactivement, pas réactivement**
   - Détecter les problèmes avant qu'ils n'arrivent

2. **Les 4 Golden Signals**
   - Latency, Traffic, Errors, Saturation

3. **Automatiser la collecte**
   - Scripts + cron = surveillance continue

4. **Définir des seuils adaptés**
   - Éviter les faux positifs

5. **Conserver l'historique**
   - Analyse des tendances et prédiction

6. **Alerter intelligemment**
   - Email pour critiques, Slack pour warnings

7. **Visualiser les métriques**
   - Dashboard pour vue d'ensemble

### Prochaines Étapes

1. **Mettre en place les scripts de base**
2. **Configurer les alertes email**
3. **Automatiser avec cron**
4. **Collecter l'historique pendant 30 jours**
5. **Analyser les tendances**
6. **Mettre en place Prometheus + Grafana**
7. **Optimiser les seuils**

### Ressources Complémentaires

**Documentation PostgreSQL :**
- Monitoring : https://www.postgresql.org/docs/18/monitoring.html
- Statistics Views : https://www.postgresql.org/docs/18/monitoring-stats.html

**Extensions recommandées :**
- **pg_stat_statements** : Analyse des requêtes
- **pg_stat_kcache** : Métriques système
- **auto_explain** : Log des plans d'exécution

**Outils :**
- **Grafana Dashboards** : https://grafana.com/grafana/dashboards/
- **pgwatch2** : https://github.com/cybertec-postgresql/pgwatch2
- **check_postgres** : https://bucardo.org/check_postgres/


---

## Fin de l'Annexe G.3

Vous disposez maintenant d'un arsenal complet de scripts de monitoring pour surveiller PostgreSQL à tous les niveaux : système, application et business. La clé du succès : **automatiser, collecter, analyser, alerter** ! 🚀

**Un système bien surveillé est un système fiable.** ✅

⏭️ Retour au [Sommaire](/SOMMAIRE.md)
