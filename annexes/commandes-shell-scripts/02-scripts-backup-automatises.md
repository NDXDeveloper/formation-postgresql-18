🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Scripts de Backup Automatisés pour PostgreSQL
## Annexe G.2 - Automatisation des Sauvegardes

---

## Table des Matières

1. [Introduction aux Backups Automatisés](#1-introduction-aux-backups-automatis%C3%A9s)  
2. [Stratégies de Backup](#2-strat%C3%A9gies-de-backup)  
3. [Scripts de Base](#3-scripts-de-base)  
4. [Scripts Avancés pour Production](#4-scripts-avanc%C3%A9s-pour-production)  
5. [Automatisation avec Cron](#5-automatisation-avec-cron)  
6. [Rotation des Backups](#6-rotation-des-backups)  
7. [Destinations de Sauvegarde](#7-destinations-de-sauvegarde)  
8. [Monitoring et Alertes](#8-monitoring-et-alertes)  
9. [Scripts de Vérification](#9-scripts-de-v%C3%A9rification)  
10. [Bonnes Pratiques](#10-bonnes-pratiques)

---

## 1. Introduction aux Backups Automatisés

### 1.1. Pourquoi Automatiser les Backups ?

**Les backups manuels sont risqués :**
- ❌ Oublis fréquents  
- ❌ Incohérences dans le processus  
- ❌ Erreurs humaines  
- ❌ Pas d'historique fiable  
- ❌ Impossible à grande échelle

**Les backups automatisés offrent :**
- ✅ Fiabilité : Exécution régulière garantie  
- ✅ Cohérence : Processus identique à chaque fois  
- ✅ Traçabilité : Logs et historique complets  
- ✅ Sérénité : Dormez tranquille !  
- ✅ Conformité : Respect des politiques de sauvegarde

### 1.2. Qu'est-ce qu'un Script de Backup ?

Un **script de backup** est un programme (généralement en Bash) qui :
1. Se connecte à PostgreSQL  
2. Exporte les données (pg_dump)  
3. Compresse le résultat  
4. Nomme le fichier avec un timestamp  
5. Stocke le backup dans un emplacement sûr  
6. Nettoie les anciens backups (rotation)  
7. Vérifie l'intégrité  
8. Envoie des alertes en cas de problème

**Exemple de workflow automatisé :**

```
[Cron] → Tous les jours à 2h du matin
    ↓
[Script de backup]
    ↓
1. Vérifier que PostgreSQL est actif
2. Créer le backup avec pg_dump
3. Compresser et nommer (db_20251121_020000.dump)
4. Copier vers NAS/Cloud
5. Supprimer backups > 30 jours
6. Vérifier l'intégrité
7. Envoyer email de confirmation
    ↓
[Backup sécurisé et vérifié]
```

### 1.3. Composants d'un Système de Backup Complet

Un système de backup professionnel comprend :

1. **Script principal** : Effectue le backup  
2. **Configuration** : Paramètres centralisés  
3. **Rotation** : Gestion de l'historique  
4. **Vérification** : Validation de l'intégrité  
5. **Stockage** : Multiple destinations (local, distant, cloud)  
6. **Monitoring** : Surveillance et alertes  
7. **Logs** : Traçabilité complète  
8. **Restauration** : Tests réguliers

### 1.4. Prérequis

**Avant de commencer, assurez-vous d'avoir :**
- PostgreSQL 18 installé et fonctionnel
- Accès shell au serveur (SSH)
- Droits d'exécution sur les scripts
- Espace disque suffisant pour les backups
- (Optionnel) Accès à un serveur distant ou cloud

**Connaissances requises :**
- Bases du shell Bash
- Utilisation de `pg_dump` (voir Annexe G.1)
- Notions de cron (sera expliqué ici)

---

## 2. Stratégies de Backup

### 2.1. Fréquences de Backup

**Quelle fréquence choisir ?** Cela dépend de votre tolérance à la perte de données.

| Fréquence | RPO* | Cas d'usage | Coût stockage |
|-----------|------|-------------|---------------|
| **Horaire** | 1 heure | Applications critiques (finance, santé) | Très élevé |
| **4× par jour** | 6 heures | E-commerce, SaaS | Élevé |
| **Quotidienne** | 24 heures | Applications standard | Moyen |
| **Hebdomadaire** | 7 jours | Développement, archives | Faible |

**RPO = Recovery Point Objective : Perte de données maximale acceptable*

**💡 Recommandation standard :**
- **Production** : Backup quotidien + WAL archiving continu (RPO quasi-nul)  
- **Staging** : Backup quotidien  
- **Développement** : Backup hebdomadaire

### 2.2. Stratégie 3-2-1 (La Règle d'Or)

**Principe de la règle 3-2-1 :**
- **3** copies de vos données (1 production + 2 backups)
- Sur **2** supports différents (disque local + NAS/cloud)
- **1** copie hors site (cloud, datacenter distant)

**Exemple concret :**

```
[Base Production]  ← Copie 1 (données actives)
        ↓
[Backup Local]     ← Copie 2 (disque serveur)
        ↓
[Backup NAS]       ← Copie 3 (réseau local, support différent)
        ↓
[Backup Cloud/S3]  ← Copie 4 (hors site, protection maximale)
```

**Pourquoi c'est crucial ?**
- **Disque dur défaillant** → Copie sur NAS disponible  
- **Incendie datacenter** → Copie cloud disponible  
- **Ransomware** → Copie hors site intacte  
- **Erreur humaine** → Plusieurs points de restauration

### 2.3. Stratégie de Rétention (Combien de Temps Garder ?)

**Politique GFS (Grand-Père Père Fils) :**

```
[Quotidiens]     : 7 derniers jours   (Fils)
[Hebdomadaires]  : 4 dernières semaines  (Père)
[Mensuels]       : 12 derniers mois  (Grand-Père)
[Annuels]        : 5-7 dernières années (Archivage)
```

**Exemple de répartition :**
- Lundi-Dimanche : Backups quotidiens (7 fichiers)
- Dimanche uniquement : Backups hebdomadaires (4 fichiers)
- 1er du mois : Backups mensuels (12 fichiers)
- 1er janvier : Backup annuel (7 fichiers)

**Total : ~30 fichiers de backup** au lieu de 365 !

### 2.4. Types de Backups

#### 2.4.1. Backup Complet (Full)

**Définition :** Sauvegarde intégrale de toute la base de données.

**Avantages :**
- ✅ Restauration simple (1 seul fichier)  
- ✅ Autonome et indépendant  
- ✅ Facile à gérer

**Inconvénients :**
- ❌ Consomme beaucoup d'espace  
- ❌ Temps de backup long  
- ❌ Bande passante réseau importante

**Quand utiliser :**
- Bases de données < 500 GB
- Backup quotidien standard
- Obligation de conformité

#### 2.4.2. Backup Incrémental

**Définition :** Sauvegarde uniquement les modifications depuis le dernier backup.

**Avantages :**
- ✅ Rapide à exécuter  
- ✅ Consomme peu d'espace  
- ✅ Peut être exécuté fréquemment

**Inconvénients :**
- ❌ Restauration complexe (chaîne de backups)  
- ❌ Dépendance entre fichiers  
- ❌ Nécessite WAL archiving

**Quand utiliser :**
- Très grandes bases de données (> 1 TB)
- Backup continu (toutes les heures)
- Avec réplication physique

**💡 Note :** PostgreSQL gère l'incrémental via **WAL archiving** (Write-Ahead Logs). Nous nous concentrerons ici sur les backups complets logiques.

### 2.5. Checklist : Définir Votre Stratégie

**Avant de créer vos scripts, répondez à ces questions :**

- [ ] Quelle perte de données maximale est acceptable ? (RPO)  
- [ ] Combien de temps peut durer une restauration ? (RTO)  
- [ ] Quelle est la taille de vos bases de données ?  
- [ ] Combien d'espace disque avez-vous ?  
- [ ] Avez-vous accès à un stockage distant/cloud ?  
- [ ] Qui doit être alerté en cas de problème ?  
- [ ] Quelle est votre politique de rétention ?

**Exemple de stratégie définie :**

```
Application E-commerce "MonShop"
- Base de données : 50 GB
- RPO : 24 heures (backup quotidien)
- RTO : 2 heures
- Rétention : 7 jours (quotidien) + 4 semaines (hebdo) + 12 mois (mensuel)
- Stockage : Local + S3
- Alertes : admin@monshop.com
```

---

## 3. Scripts de Base

### 3.1. Script Minimaliste (Point de Départ)

**Le script de backup le plus simple possible :**

```bash
#!/bin/bash
# backup_minimal.sh

DATABASE="mabase"  
BACKUP_DIR="/var/backups/postgresql"  
TIMESTAMP=$(date +%Y%m%d_%H%M%S)  

# Créer le répertoire si nécessaire
mkdir -p "$BACKUP_DIR"

# Effectuer le backup
pg_dump -F c "$DATABASE" -f "$BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump"

echo "Backup terminé : $BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump"
```

**Ce que fait ce script :**
1. Définit la base à sauvegarder (`mabase`)  
2. Définit le répertoire de destination  
3. Génère un timestamp (ex: `20251121_143000`)  
4. Crée le répertoire si besoin  
5. Exécute `pg_dump` en format custom compressé  
6. Affiche le chemin du backup créé

**Utilisation :**

```bash
# Rendre le script exécutable
chmod +x backup_minimal.sh

# Exécuter
./backup_minimal.sh
```

**Résultat :**

```
/var/backups/postgresql/mabase_20251121_143000.dump
```

### 3.2. Script Simple avec Vérifications

**Amélioration : Ajouter des vérifications de base**

```bash
#!/bin/bash
# backup_simple.sh

# ========================================
# Configuration
# ========================================
DATABASE="mabase"  
BACKUP_DIR="/var/backups/postgresql"  
TIMESTAMP=$(date +%Y%m%d_%H%M%S)  
BACKUP_FILE="$BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump"  

# ========================================
# Fonctions
# ========================================

# Fonction de log simple
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# Fonction de vérification de succès
check_success() {
    if [ $? -eq 0 ]; then
        log "✅ $1 : OK"
        return 0
    else
        log "❌ $1 : ÉCHEC"
        return 1
    fi
}

# ========================================
# Script Principal
# ========================================

log "=== Début du backup de $DATABASE ==="

# Étape 1 : Créer le répertoire
mkdir -p "$BACKUP_DIR"  
check_success "Création du répertoire"  

# Étape 2 : Vérifier que PostgreSQL est accessible
log "Vérification de la connexion à PostgreSQL..."  
if ! pg_isready -d "$DATABASE" > /dev/null 2>&1; then  
    log "❌ ERREUR : Impossible de se connecter à PostgreSQL"
    exit 1
fi  
log "✅ Connexion PostgreSQL OK"  

# Étape 3 : Effectuer le backup
log "Début de l'export de $DATABASE..."  
pg_dump -F c -Z 9 "$DATABASE" -f "$BACKUP_FILE"  
check_success "Export de la base"  

# Étape 4 : Vérifier que le fichier existe et n'est pas vide
if [ -f "$BACKUP_FILE" ] && [ -s "$BACKUP_FILE" ]; then
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "✅ Backup créé : $BACKUP_FILE ($SIZE)"
else
    log "❌ ERREUR : Le fichier de backup est vide ou inexistant"
    exit 1
fi

# Étape 5 : Vérifier l'intégrité du backup
log "Vérification de l'intégrité du backup..."  
if pg_restore --list "$BACKUP_FILE" > /dev/null 2>&1; then  
    log "✅ Backup valide"
else
    log "❌ ATTENTION : Le backup semble corrompu !"
    exit 1
fi

log "=== Backup terminé avec succès ==="  
exit 0  
```

**Améliorations apportées :**
- ✅ Fonction `log()` pour horodater les messages  
- ✅ Vérification de la connectivité PostgreSQL (`pg_isready`)  
- ✅ Vérification de l'existence et taille du fichier  
- ✅ Validation de l'intégrité (`pg_restore --list`)  
- ✅ Codes de retour appropriés (exit 0/1)  
- ✅ Messages clairs et informatifs

### 3.3. Script avec Rotation (Nettoyage Automatique)

**Ajout de la rotation pour éviter de remplir le disque :**

```bash
#!/bin/bash
# backup_avec_rotation.sh

# ========================================
# Configuration
# ========================================
DATABASE="mabase"  
BACKUP_DIR="/var/backups/postgresql"  
RETENTION_DAYS=7  # Nombre de jours à conserver  
TIMESTAMP=$(date +%Y%m%d_%H%M%S)  
BACKUP_FILE="$BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump"  

# ========================================
# Fonctions
# ========================================

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# ========================================
# Script Principal
# ========================================

log "=== Backup de $DATABASE avec rotation ${RETENTION_DAYS} jours ==="

# Créer le répertoire
mkdir -p "$BACKUP_DIR"

# Vérifier PostgreSQL
if ! pg_isready -d "$DATABASE" > /dev/null 2>&1; then
    log "❌ PostgreSQL non accessible"
    exit 1
fi

# Effectuer le backup
log "Création du backup..."  
pg_dump -F c -Z 9 "$DATABASE" -f "$BACKUP_FILE"  

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "✅ Backup créé : $BACKUP_FILE ($SIZE)"

    # Vérifier l'intégrité
    if pg_restore --list "$BACKUP_FILE" > /dev/null 2>&1; then
        log "✅ Backup validé"
    else
        log "⚠️ ATTENTION : Backup potentiellement corrompu"
    fi
else
    log "❌ Échec du backup"
    exit 1
fi

# Rotation : Supprimer les backups plus anciens que RETENTION_DAYS
log "Nettoyage des anciens backups (> $RETENTION_DAYS jours)..."  
BEFORE_COUNT=$(find "$BACKUP_DIR" -name "${DATABASE}_*.dump" | wc -l)  

find "$BACKUP_DIR" -name "${DATABASE}_*.dump" -type f -mtime +$RETENTION_DAYS -delete

AFTER_COUNT=$(find "$BACKUP_DIR" -name "${DATABASE}_*.dump" | wc -l)  
DELETED=$((BEFORE_COUNT - AFTER_COUNT))  

log "🗑️  $DELETED ancien(s) backup(s) supprimé(s)"  
log "📦 Backups restants : $AFTER_COUNT"  

log "=== Backup terminé ==="  
exit 0  
```

**Fonctionnement de la rotation :**

```bash
find "$BACKUP_DIR" -name "${DATABASE}_*.dump" -type f -mtime +$RETENTION_DAYS -delete
```

**Explication :**
- `find "$BACKUP_DIR"` : Cherche dans le répertoire de backup  
- `-name "${DATABASE}_*.dump"` : Fichiers correspondant au pattern (ex: `mabase_*.dump`)  
- `-type f` : Uniquement les fichiers (pas les répertoires)  
- `-mtime +$RETENTION_DAYS` : Modifiés il y a plus de N jours  
- `-delete` : Supprimer les fichiers trouvés

**Exemple avec RETENTION_DAYS=7 :**

```
Jour 1 : mabase_20251115.dump  ← Créé  
Jour 2 : mabase_20251116.dump  ← Créé  
...
Jour 7 : mabase_20251121.dump  ← Créé  
Jour 8 : mabase_20251122.dump  ← Créé  
         mabase_20251115.dump  ← Supprimé (> 7 jours)
```

### 3.4. Script Multi-Bases

**Si vous avez plusieurs bases de données :**

```bash
#!/bin/bash
# backup_multi_bases.sh

# ========================================
# Configuration
# ========================================
BACKUP_DIR="/var/backups/postgresql"  
RETENTION_DAYS=7  
TIMESTAMP=$(date +%Y%m%d_%H%M%S)  

# Liste des bases à sauvegarder
DATABASES=(
    "production"
    "staging"
    "analytics"
)

# ========================================
# Fonctions
# ========================================

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

backup_database() {
    local db="$1"
    local backup_file="$BACKUP_DIR/${db}_${TIMESTAMP}.dump"

    log "📦 Backup de $db..."

    if ! pg_isready -d "$db" > /dev/null 2>&1; then
        log "❌ $db non accessible"
        return 1
    fi

    pg_dump -F c -Z 9 "$db" -f "$backup_file"

    if [ $? -eq 0 ] && [ -s "$backup_file" ]; then
        local size=$(du -h "$backup_file" | cut -f1)
        log "✅ $db : $backup_file ($size)"
        return 0
    else
        log "❌ $db : Échec du backup"
        return 1
    fi
}

# ========================================
# Script Principal
# ========================================

log "=== Backup de ${#DATABASES[@]} base(s) de données ==="

mkdir -p "$BACKUP_DIR"

# Sauvegarder chaque base
SUCCESS_COUNT=0  
FAIL_COUNT=0  

for db in "${DATABASES[@]}"; do
    if backup_database "$db"; then
        ((SUCCESS_COUNT++))
    else
        ((FAIL_COUNT++))
    fi
done

log "📊 Résumé : $SUCCESS_COUNT succès, $FAIL_COUNT échec(s)"

# Rotation globale
log "Nettoyage des anciens backups..."  
for db in "${DATABASES[@]}"; do  
    deleted=$(find "$BACKUP_DIR" -name "${db}_*.dump" -type f -mtime +$RETENTION_DAYS -delete -print | wc -l)
    if [ $deleted -gt 0 ]; then
        log "🗑️  $db : $deleted ancien(s) backup(s) supprimé(s)"
    fi
done

log "=== Backup terminé ==="

# Code de retour
if [ $FAIL_COUNT -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

**Avantages :**
- ✅ Sauvegarde plusieurs bases en une seule exécution  
- ✅ Comptabilisation des succès/échecs  
- ✅ Rotation indépendante par base  
- ✅ Facile à étendre (ajouter des bases dans le tableau)

---

## 4. Scripts Avancés pour Production

### 4.1. Script avec Fichier de Configuration

**Séparer la configuration du code :**

**Fichier : `/etc/postgresql_backup.conf`**

```bash
# Configuration du système de backup PostgreSQL

# Bases de données à sauvegarder (séparées par des espaces)
DATABASES="production staging analytics"

# Répertoire de backup local
BACKUP_DIR="/var/backups/postgresql"

# Répertoire des logs
LOG_DIR="/var/log/postgresql_backup"

# Rétention (jours)
RETENTION_DAILY=7  
RETENTION_WEEKLY=28  
RETENTION_MONTHLY=365  

# Compression (0-9)
COMPRESSION_LEVEL=9

# Parallélisme pour grandes bases
PARALLEL_JOBS=4

# Stockage distant
ENABLE_REMOTE_BACKUP=true  
REMOTE_BACKUP_DIR="/mnt/nas/postgresql_backups"  
S3_BUCKET="s3://my-company-backups/postgresql"  

# Alertes
ENABLE_EMAIL_ALERTS=true  
ALERT_EMAIL="admin@example.com"  
SMTP_SERVER="smtp.example.com"  

# Options avancées
ENABLE_INTEGRITY_CHECK=true  
ENABLE_SIZE_MONITORING=true  
MAX_BACKUP_SIZE_GB=100  
```

**Script : `backup_production.sh`**

```bash
#!/bin/bash
# backup_production.sh

# ========================================
# Chargement de la Configuration
# ========================================

CONFIG_FILE="/etc/postgresql_backup.conf"

if [ ! -f "$CONFIG_FILE" ]; then
    echo "❌ ERREUR : Fichier de configuration introuvable : $CONFIG_FILE"
    exit 1
fi

# Charger la configuration
source "$CONFIG_FILE"

# ========================================
# Variables Globales
# ========================================

TIMESTAMP=$(date +%Y%m%d_%H%M%S)  
DATE_SHORT=$(date +%Y%m%d)  
LOG_FILE="$LOG_DIR/backup_${TIMESTAMP}.log"  

# Créer les répertoires nécessaires
mkdir -p "$BACKUP_DIR" "$LOG_DIR"

# ========================================
# Fonctions
# ========================================

log() {
    local message="[$(date '+%Y-%m-%d %H:%M:%S')] $1"
    echo "$message" | tee -a "$LOG_FILE"
}

log_error() {
    local message="[$(date '+%Y-%m-%d %H:%M:%S')] ❌ ERREUR : $1"
    echo "$message" | tee -a "$LOG_FILE" >&2
}

send_alert() {
    if [ "$ENABLE_EMAIL_ALERTS" = true ]; then
        local subject="$1"
        local body="$2"

        echo "$body" | mail -s "$subject" -S smtp="$SMTP_SERVER" "$ALERT_EMAIL"
        log "📧 Alerte envoyée : $subject"
    fi
}

check_disk_space() {
    local min_space_gb=20
    local available_gb=$(df -BG "$BACKUP_DIR" | tail -1 | awk '{print $4}' | sed 's/G//')

    if [ "$available_gb" -lt "$min_space_gb" ]; then
        log_error "Espace disque insuffisant : ${available_gb}GB disponible (minimum: ${min_space_gb}GB)"
        send_alert "Backup PostgreSQL - Espace Disque Critique" \
                   "Espace disponible : ${available_gb}GB\nMinimum requis : ${min_space_gb}GB"
        return 1
    fi

    log "💾 Espace disque : ${available_gb}GB disponible"
    return 0
}

backup_database() {
    local db="$1"
    local backup_file="$BACKUP_DIR/${db}_${TIMESTAMP}.dump"
    local start_time=$(date +%s)

    log "=========================================="
    log "📦 Début du backup : $db"
    log "=========================================="

    # Vérifier la connectivité
    if ! pg_isready -d "$db" > /dev/null 2>&1; then
        log_error "Base $db non accessible"
        return 1
    fi

    # Taille de la base avant backup
    local db_size=$(psql -t -d "$db" -c "SELECT pg_size_pretty(pg_database_size('$db'))" | xargs)
    log "📊 Taille de la base : $db_size"

    # Effectuer le backup
    log "🔄 Export en cours (compression: $COMPRESSION_LEVEL)..."

    if pg_dump -F c -Z "$COMPRESSION_LEVEL" "$db" -f "$backup_file" 2>> "$LOG_FILE"; then
        local end_time=$(date +%s)
        local duration=$((end_time - start_time))
        local backup_size=$(du -h "$backup_file" | cut -f1)

        log "✅ Backup réussi en ${duration}s"
        log "📦 Fichier : $backup_file"
        log "💾 Taille : $backup_size"

        # Vérifier l'intégrité
        if [ "$ENABLE_INTEGRITY_CHECK" = true ]; then
            log "🔍 Vérification de l'intégrité..."
            if pg_restore --list "$backup_file" > /dev/null 2>&1; then
                log "✅ Intégrité validée"
            else
                log_error "Backup corrompu : $backup_file"
                send_alert "Backup PostgreSQL - Backup Corrompu" \
                           "La base $db a un backup corrompu : $backup_file"
                return 1
            fi
        fi

        # Monitoring de taille
        if [ "$ENABLE_SIZE_MONITORING" = true ]; then
            local size_gb=$(du -BG "$backup_file" | cut -f1 | sed 's/G//')
            if [ "$size_gb" -gt "$MAX_BACKUP_SIZE_GB" ]; then
                log "⚠️ ATTENTION : Backup anormalement volumineux (${size_gb}GB)"
                send_alert "Backup PostgreSQL - Taille Anormale" \
                           "Le backup de $db fait ${size_gb}GB (max: ${MAX_BACKUP_SIZE_GB}GB)"
            fi
        fi

        return 0
    else
        log_error "Échec du backup de $db"
        return 1
    fi
}

rotate_backups() {
    local db="$1"

    log "🗑️  Rotation des backups pour $db..."

    # Quotidiens (garder RETENTION_DAILY jours)
    local deleted=$(find "$BACKUP_DIR" -name "${db}_*.dump" -type f -mtime +$RETENTION_DAILY -delete -print | wc -l)
    if [ $deleted -gt 0 ]; then
        log "  Quotidiens : $deleted supprimé(s)"
    fi

    # Afficher les backups restants
    local remaining=$(find "$BACKUP_DIR" -name "${db}_*.dump" -type f | wc -l)
    log "  📦 Backups restants : $remaining"
}

backup_to_remote() {
    if [ "$ENABLE_REMOTE_BACKUP" != true ]; then
        return 0
    fi

    log "=========================================="
    log "☁️  Copie vers stockage distant"
    log "=========================================="

    # Copie vers NAS
    if [ -n "$REMOTE_BACKUP_DIR" ] && [ -d "$(dirname "$REMOTE_BACKUP_DIR")" ]; then
        log "📡 Copie vers NAS : $REMOTE_BACKUP_DIR"
        mkdir -p "$REMOTE_BACKUP_DIR"

        rsync -av --progress "$BACKUP_DIR/" "$REMOTE_BACKUP_DIR/" >> "$LOG_FILE" 2>&1

        if [ $? -eq 0 ]; then
            log "✅ Copie NAS réussie"
        else
            log_error "Échec de la copie NAS"
        fi
    fi

    # Copie vers S3 (si aws-cli est installé)
    if [ -n "$S3_BUCKET" ] && command -v aws > /dev/null 2>&1; then
        log "📡 Upload vers S3 : $S3_BUCKET"

        for dump_file in "$BACKUP_DIR"/*_${TIMESTAMP}.dump; do
            if [ -f "$dump_file" ]; then
                aws s3 cp "$dump_file" "$S3_BUCKET/$(basename "$dump_file")" >> "$LOG_FILE" 2>&1

                if [ $? -eq 0 ]; then
                    log "✅ Upload S3 : $(basename "$dump_file")"
                else
                    log_error "Échec upload S3 : $(basename "$dump_file")"
                fi
            fi
        done
    fi
}

# ========================================
# Script Principal
# ========================================

log "=========================================="  
log "🚀 Démarrage du backup PostgreSQL"  
log "=========================================="  
log "Configuration : $CONFIG_FILE"  
log "Timestamp : $TIMESTAMP"  
log ""  

# Vérifier l'espace disque
if ! check_disk_space; then
    log_error "Backup annulé : espace disque insuffisant"
    exit 1
fi

# Sauvegarder les objets globaux (rôles, tablespaces)
log "=========================================="  
log "👥 Backup des objets globaux"  
log "=========================================="  

GLOBALS_FILE="$BACKUP_DIR/globals_${TIMESTAMP}.sql"  
if pg_dumpall --globals-only > "$GLOBALS_FILE" 2>> "$LOG_FILE"; then  
    log "✅ Objets globaux sauvegardés : $GLOBALS_FILE"
else
    log_error "Échec du backup des objets globaux"
fi

# Backup de chaque base
SUCCESS_COUNT=0  
FAIL_COUNT=0  

for db in $DATABASES; do
    if backup_database "$db"; then
        ((SUCCESS_COUNT++))
        rotate_backups "$db"
    else
        ((FAIL_COUNT++))
    fi
    echo "" >> "$LOG_FILE"
done

# Copie vers stockage distant
backup_to_remote

# ========================================
# Résumé et Alertes
# ========================================

log "=========================================="  
log "📊 RÉSUMÉ"  
log "=========================================="  
log "Bases sauvegardées : $SUCCESS_COUNT"  
log "Échecs : $FAIL_COUNT"  
log "Durée totale : $SECONDS secondes"  
log "Log complet : $LOG_FILE"  

# Taille totale des backups
TOTAL_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)  
log "💾 Taille totale des backups : $TOTAL_SIZE"  

# Envoyer une alerte si des échecs
if [ $FAIL_COUNT -gt 0 ]; then
    send_alert "Backup PostgreSQL - Échecs Détectés" \
               "$(cat "$LOG_FILE" | grep "ERREUR")"
    log "=========================================="
    exit 1
else
    log "✅ Tous les backups réussis"
    log "=========================================="
    exit 0
fi
```

**Avantages de cette approche :**
- ✅ Configuration centralisée et facilement modifiable  
- ✅ Gestion complète des erreurs  
- ✅ Monitoring et alertes  
- ✅ Logs détaillés  
- ✅ Copie vers stockage distant (NAS, S3)  
- ✅ Vérification d'intégrité  
- ✅ Monitoring de la taille des backups  
- ✅ Backup des objets globaux (rôles)  
- ✅ Prêt pour la production

### 4.2. Script avec Politique GFS (Grand-Père Père Fils)

**Rotation avancée avec conservation hiérarchique :**

```bash
#!/bin/bash
# backup_gfs.sh

# ========================================
# Configuration
# ========================================

DATABASE="production"  
BACKUP_DIR="/var/backups/postgresql"  
DAILY_DIR="$BACKUP_DIR/daily"  
WEEKLY_DIR="$BACKUP_DIR/weekly"  
MONTHLY_DIR="$BACKUP_DIR/monthly"  
YEARLY_DIR="$BACKUP_DIR/yearly"  

TIMESTAMP=$(date +%Y%m%d_%H%M%S)  
DAY_OF_WEEK=$(date +%u)    # 1=Lundi, 7=Dimanche  
DAY_OF_MONTH=$(date +%d)   # 01-31  
DAY_OF_YEAR=$(date +%j)    # 001-365  

# Rétentions
RETENTION_DAILY=7      # 7 jours  
RETENTION_WEEKLY=28    # 4 semaines  
RETENTION_MONTHLY=365  # 12 mois  
RETENTION_YEARLY=2555  # 7 ans  

# ========================================
# Fonctions
# ========================================

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

create_backup() {
    local target_dir="$1"
    local backup_type="$2"

    mkdir -p "$target_dir"

    local backup_file="$target_dir/${DATABASE}_${backup_type}_${TIMESTAMP}.dump"

    log "📦 Création backup $backup_type : $backup_file"

    if pg_dump -F c -Z 9 "$DATABASE" -f "$backup_file" 2>/dev/null; then
        local size=$(du -h "$backup_file" | cut -f1)
        log "✅ Backup $backup_type créé : $size"
        return 0
    else
        log "❌ Échec backup $backup_type"
        return 1
    fi
}

rotate_directory() {
    local dir="$1"
    local retention="$2"
    local type="$3"

    if [ ! -d "$dir" ]; then
        return 0
    fi

    local before=$(find "$dir" -name "*.dump" | wc -l)
    find "$dir" -name "*.dump" -type f -mtime +$retention -delete
    local after=$(find "$dir" -name "*.dump" | wc -l)
    local deleted=$((before - after))

    if [ $deleted -gt 0 ]; then
        log "🗑️  $type : $deleted supprimé(s), $after restant(s)"
    fi
}

# ========================================
# Script Principal
# ========================================

log "=========================================="  
log "🔄 Backup GFS de $DATABASE"  
log "=========================================="  
log "Date : $(date '+%Y-%m-%d %H:%M:%S')"  
log "Jour de la semaine : $DAY_OF_WEEK"  
log "Jour du mois : $DAY_OF_MONTH"  
log ""  

# Déterminer le type de backup selon la date
BACKUP_CREATED=false

# Annuel (1er janvier)
if [ "$DAY_OF_YEAR" = "001" ]; then
    log "📅 Backup ANNUEL"
    create_backup "$YEARLY_DIR" "yearly"
    BACKUP_CREATED=true
fi

# Mensuel (1er du mois)
if [ "$DAY_OF_MONTH" = "01" ] && [ "$BACKUP_CREATED" = false ]; then
    log "📅 Backup MENSUEL"
    create_backup "$MONTHLY_DIR" "monthly"
    BACKUP_CREATED=true
fi

# Hebdomadaire (dimanche)
if [ "$DAY_OF_WEEK" = "7" ] && [ "$BACKUP_CREATED" = false ]; then
    log "📅 Backup HEBDOMADAIRE"
    create_backup "$WEEKLY_DIR" "weekly"
    BACKUP_CREATED=true
fi

# Quotidien (tous les jours)
if [ "$BACKUP_CREATED" = false ]; then
    log "📅 Backup QUOTIDIEN"
    create_backup "$DAILY_DIR" "daily"
fi

log ""  
log "=========================================="  
log "🗑️  Rotation des anciens backups"  
log "=========================================="  

# Rotation de chaque niveau
rotate_directory "$DAILY_DIR" $RETENTION_DAILY "Quotidiens"  
rotate_directory "$WEEKLY_DIR" $RETENTION_WEEKLY "Hebdomadaires"  
rotate_directory "$MONTHLY_DIR" $RETENTION_MONTHLY "Mensuels"  
rotate_directory "$YEARLY_DIR" $RETENTION_YEARLY "Annuels"  

log ""  
log "=========================================="  
log "📊 Résumé des backups"  
log "=========================================="  

# Compter les backups par catégorie
DAILY_COUNT=$(find "$DAILY_DIR" -name "*.dump" 2>/dev/null | wc -l)  
WEEKLY_COUNT=$(find "$WEEKLY_DIR" -name "*.dump" 2>/dev/null | wc -l)  
MONTHLY_COUNT=$(find "$MONTHLY_DIR" -name "*.dump" 2>/dev/null | wc -l)  
YEARLY_COUNT=$(find "$YEARLY_DIR" -name "*.dump" 2>/dev/null | wc -l)  
TOTAL_COUNT=$((DAILY_COUNT + WEEKLY_COUNT + MONTHLY_COUNT + YEARLY_COUNT))  

log "📦 Quotidiens : $DAILY_COUNT"  
log "📦 Hebdomadaires : $WEEKLY_COUNT"  
log "📦 Mensuels : $MONTHLY_COUNT"  
log "📦 Annuels : $YEARLY_COUNT"  
log "📦 TOTAL : $TOTAL_COUNT backups"  

# Taille totale
TOTAL_SIZE=$(du -sh "$BACKUP_DIR" 2>/dev/null | cut -f1)  
log "💾 Espace utilisé : $TOTAL_SIZE"  

log "=========================================="  
log "✅ Backup GFS terminé"  
log "=========================================="  
```

**Avantages de la stratégie GFS :**
- ✅ Historique long sans exploser l'espace disque  
- ✅ Points de restauration fréquents (7 jours)  
- ✅ Archivage à long terme (jusqu'à 7 ans)  
- ✅ Équilibre optimal coût/protection

**Exemple de répartition sur un an :**

```
Janvier 1  : Backup annuel   → yearly/  
Janvier 7  : Backup hebdo    → weekly/  
Janvier 8  : Backup quotidien → daily/  
Janvier 9  : Backup quotidien → daily/  
...
Février 1  : Backup mensuel  → monthly/  
Février 7  : Backup hebdo    → weekly/  
...
```

**Après 1 an, vous aurez :**
- 7 backups quotidiens (dernière semaine)
- 4 backups hebdomadaires (dernier mois)
- 12 backups mensuels (dernière année)
- 1 backup annuel
- **Total : ~24 fichiers** au lieu de 365 !

---

## 5. Automatisation avec Cron

### 5.1. Introduction à Cron

**Qu'est-ce que Cron ?**

Cron est un **planificateur de tâches** sous Linux/Unix qui permet d'exécuter des scripts à intervalles réguliers (quotidien, hebdomadaire, mensuel, etc.).

**Analogie :** Cron est comme un assistant qui exécute vos scripts automatiquement selon un planning que vous définissez.

### 5.2. Syntaxe de Cron

Une ligne cron (crontab) suit ce format :

```
* * * * * commande_à_exécuter
│ │ │ │ │
│ │ │ │ └─── Jour de la semaine (0-7, 0 et 7 = Dimanche)
│ │ │ └───── Mois (1-12)
│ │ └─────── Jour du mois (1-31)
│ └───────── Heure (0-23)
└─────────── Minute (0-59)
```

**Caractères spéciaux :**
- `*` : Toutes les valeurs (chaque minute, chaque heure, etc.)  
- `,` : Liste de valeurs (ex: `1,15,30` = 1er, 15e et 30e)  
- `-` : Plage de valeurs (ex: `1-5` = de 1 à 5)  
- `/` : Intervalle (ex: `*/15` = toutes les 15 minutes)

### 5.3. Exemples de Syntaxe Cron

```bash
# Toutes les heures à la minute 0
0 * * * * /chemin/script.sh

# Tous les jours à 2h du matin
0 2 * * * /chemin/script.sh

# Tous les jours à 2h30
30 2 * * * /chemin/script.sh

# Toutes les 6 heures
0 */6 * * * /chemin/script.sh

# Tous les lundis à 3h
0 3 * * 1 /chemin/script.sh

# Tous les dimanches à 4h
0 4 * * 0 /chemin/script.sh

# Le 1er de chaque mois à 5h
0 5 1 * * /chemin/script.sh

# Du lundi au vendredi à 8h
0 8 * * 1-5 /chemin/script.sh

# Toutes les 15 minutes
*/15 * * * * /chemin/script.sh

# À 2h et 14h tous les jours
0 2,14 * * * /chemin/script.sh
```

### 5.4. Configurer Cron pour les Backups

#### 5.4.1. Éditer le Crontab

**Pour l'utilisateur postgres (recommandé) :**

```bash
# Se connecter en tant que postgres
sudo su - postgres

# Éditer le crontab
crontab -e
```

**Ou en tant que root :**

```bash
sudo crontab -e
```

#### 5.4.2. Exemple de Configuration Complète

**Ouvrir l'éditeur crontab :**

```bash
crontab -e
```

**Ajouter ces lignes :**

```bash
# PostgreSQL Automated Backups
# ========================================

# Variables d'environnement
MAILTO=admin@example.com  
PATH=/usr/local/bin:/usr/bin:/bin  
PGDATA=/var/lib/postgresql/data  

# Backup quotidien à 2h du matin
0 2 * * * /usr/local/bin/backup_production.sh >> /var/log/postgresql_backup/cron.log 2>&1

# Backup hebdomadaire le dimanche à 3h
0 3 * * 0 /usr/local/bin/backup_weekly.sh >> /var/log/postgresql_backup/cron.log 2>&1

# Backup mensuel le 1er à 4h
0 4 1 * * /usr/local/bin/backup_monthly.sh >> /var/log/postgresql_backup/cron.log 2>&1

# Vérification d'intégrité tous les jours à 6h
0 6 * * * /usr/local/bin/verify_backups.sh >> /var/log/postgresql_backup/verify.log 2>&1

# Nettoyage des logs tous les dimanches à 23h
0 23 * * 0 find /var/log/postgresql_backup -name "*.log" -mtime +30 -delete
```

**Explication :**
- `MAILTO` : Envoie les erreurs par email  
- `PATH` : Chemins pour trouver les commandes  
- `>> fichier.log 2>&1` : Redirige sortie standard et erreurs vers un fichier log  
- `-mtime +30` : Supprime les logs de plus de 30 jours

#### 5.4.3. Vérifier les Tâches Cron

```bash
# Lister les tâches cron de l'utilisateur actuel
crontab -l

# Lister les tâches cron de postgres
sudo crontab -u postgres -l

# Lister toutes les tâches cron système
cat /etc/crontab  
ls -la /etc/cron.d/  
```

#### 5.4.4. Logs de Cron

**Vérifier que cron exécute bien les tâches :**

```bash
# Log système de cron (Ubuntu/Debian)
tail -f /var/log/syslog | grep CRON

# Ou (CentOS/RHEL)
tail -f /var/log/cron

# Voir les exécutions récentes
grep CRON /var/log/syslog | tail -20
```

### 5.5. Exemple Complet : Automatisation d'un Backup Quotidien

**Étape 1 : Créer le script**

```bash
sudo nano /usr/local/bin/backup_daily.sh
```

**Contenu du script :**

```bash
#!/bin/bash
# backup_daily.sh - Backup quotidien automatisé

DATABASE="production"  
BACKUP_DIR="/var/backups/postgresql/daily"  
LOG_FILE="/var/log/postgresql_backup/daily_$(date +%Y%m%d).log"  
TIMESTAMP=$(date +%Y%m%d_%H%M%S)  

# Créer les répertoires
mkdir -p "$BACKUP_DIR" "$(dirname "$LOG_FILE")"

# Rediriger toute la sortie vers le log
exec > >(tee -a "$LOG_FILE")  
exec 2>&1  

echo "=========================================="  
echo "Backup quotidien de $DATABASE"  
echo "Date : $(date)"  
echo "=========================================="  

# Backup
pg_dump -F c -Z 9 "$DATABASE" -f "$BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump"

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump" | cut -f1)
    echo "✅ Backup réussi : $SIZE"

    # Rotation (garder 7 jours)
    find "$BACKUP_DIR" -name "*.dump" -mtime +7 -delete
    echo "🗑️ Rotation effectuée"
else
    echo "❌ ÉCHEC du backup"
    exit 1
fi

echo "=========================================="  
echo "Backup terminé"  
echo "=========================================="  
```

**Étape 2 : Rendre le script exécutable**

```bash
sudo chmod +x /usr/local/bin/backup_daily.sh
```

**Étape 3 : Tester le script manuellement**

```bash
sudo /usr/local/bin/backup_daily.sh
```

**Étape 4 : Ajouter au crontab**

```bash
sudo crontab -e
```

**Ajouter :**

```bash
# Backup quotidien à 2h du matin
0 2 * * * /usr/local/bin/backup_daily.sh
```

**Étape 5 : Vérifier l'exécution**

```bash
# Après le premier run (le lendemain à 2h), vérifier :
ls -lh /var/backups/postgresql/daily/  
cat /var/log/postgresql_backup/daily_*.log  
```

### 5.6. Bonnes Pratiques pour Cron

#### ✅ À Faire

1. **Utiliser des chemins absolus**
   ```bash
   # ❌ Mauvais (chemin relatif)
   0 2 * * * backup.sh

   # ✅ Bon (chemin absolu)
   0 2 * * * /usr/local/bin/backup.sh
   ```

2. **Rediriger les sorties vers des logs**
   ```bash
   0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
   ```

3. **Définir les variables d'environnement**
   ```bash
   PATH=/usr/local/bin:/usr/bin:/bin
   PGDATA=/var/lib/postgresql/data
   ```

4. **Tester les scripts avant automation**
   ```bash
   # Toujours tester manuellement d'abord
   /usr/local/bin/backup.sh
   ```

5. **Configurer MAILTO pour les erreurs**
   ```bash
   MAILTO=admin@example.com
   ```

6. **Espacer les tâches pour éviter les conflits**
   ```bash
   # ✅ Bon
   0 2 * * * backup_db1.sh
   0 3 * * * backup_db2.sh

   # ❌ Mauvais (risque de conflit)
   0 2 * * * backup_db1.sh
   0 2 * * * backup_db2.sh
   ```

7. **Utiliser un outil de génération de syntaxe**
   - https://crontab.guru/ (générateur en ligne)

#### ❌ À Éviter

1. **Ne pas tester les scripts en environnement cron**
   - Variables d'environnement différentes
   - Peut fonctionner manuellement mais échouer en cron

2. **Oublier de gérer les erreurs**
   - Toujours vérifier les codes de retour
   - Logger les erreurs

3. **Planifier pendant les heures de pointe**
   - Éviter les backups pendant les périodes d'activité maximale
   - Préférer la nuit ou tôt le matin

4. **Ne pas monitorer l'exécution**
   - Vérifier régulièrement les logs
   - Mettre en place des alertes

---

## 6. Rotation des Backups

### 6.1. Pourquoi la Rotation est Essentielle

**Sans rotation :**
- Disque saturé en quelques semaines/mois
- Performance dégradée
- Coûts de stockage explosifs
- Backup interrompu (plus d'espace)

**Avec rotation :**
- Espace disque maîtrisé
- Coûts prévisibles
- Historique adapté aux besoins
- Système pérenne

### 6.2. Stratégies de Rotation

#### 6.2.1. Rotation Simple (Par Âge)

**Principe :** Supprimer les backups plus vieux que N jours.

```bash
# Supprimer les backups de plus de 7 jours
find /var/backups/postgresql -name "*.dump" -mtime +7 -delete
```

**Avantages :**
- ✅ Simple à implémenter  
- ✅ Prévisible

**Inconvénients :**
- ❌ Tous les backups ont la même importance  
- ❌ Pas d'historique long terme

#### 6.2.2. Rotation par Nombre de Fichiers

**Principe :** Garder les N derniers backups, supprimer les plus anciens.

```bash
#!/bin/bash
# keep_last_n.sh

BACKUP_DIR="/var/backups/postgresql"  
KEEP_COUNT=10  

# Lister les fichiers par date (plus récent en premier)
# Supprimer tout sauf les N premiers
ls -t "$BACKUP_DIR"/*.dump | tail -n +$((KEEP_COUNT + 1)) | xargs -r rm
```

**Avantages :**
- ✅ Contrôle précis du nombre de fichiers  
- ✅ Prévisible en espace disque

**Inconvénients :**
- ❌ Pas adapté à différentes fréquences

#### 6.2.3. Rotation GFS (Grand-Père Père Fils)

**Déjà vu en section 4.2 - Résumé :**

```
Quotidiens   : 7 derniers jours  
Hebdomadaires: 4 dernières semaines  
Mensuels     : 12 derniers mois  
Annuels      : 5-7 dernières années  
```

**Avantages :**
- ✅ Historique long sans exploser l'espace  
- ✅ Adapté aux besoins métier  
- ✅ Équilibre optimal

**Inconvénients :**
- ❌ Plus complexe à mettre en place

#### 6.2.4. Rotation par Taille

**Principe :** Supprimer les anciens backups quand l'espace utilisé dépasse un seuil.

```bash
#!/bin/bash
# rotate_by_size.sh

BACKUP_DIR="/var/backups/postgresql"  
MAX_SIZE_GB=100  # Taille maximale en GB  

# Calculer la taille actuelle
CURRENT_SIZE_GB=$(du -s -BG "$BACKUP_DIR" | cut -f1 | sed 's/G//')

echo "Taille actuelle : ${CURRENT_SIZE_GB}GB / ${MAX_SIZE_GB}GB"

# Si dépassement, supprimer les plus anciens fichiers
if [ "$CURRENT_SIZE_GB" -gt "$MAX_SIZE_GB" ]; then
    echo "⚠️ Dépassement de quota, nettoyage..."

    # Supprimer les plus anciens jusqu'à revenir sous le seuil
    while [ "$CURRENT_SIZE_GB" -gt "$MAX_SIZE_GB" ]; do
        OLDEST_FILE=$(ls -t "$BACKUP_DIR"/*.dump | tail -1)

        if [ -f "$OLDEST_FILE" ]; then
            echo "Suppression : $OLDEST_FILE"
            rm "$OLDEST_FILE"
            CURRENT_SIZE_GB=$(du -s -BG "$BACKUP_DIR" | cut -f1 | sed 's/G//')
        else
            break
        fi
    done

    echo "✅ Nettoyage terminé : ${CURRENT_SIZE_GB}GB"
else
    echo "✅ Espace suffisant"
fi
```

**Avantages :**
- ✅ Garantit de ne jamais saturer le disque  
- ✅ S'adapte à la croissance de la base

**Inconvénients :**
- ❌ Nombre de backups variable  
- ❌ Peut supprimer beaucoup de backups d'un coup

### 6.3. Script de Rotation Avancé

**Combiner plusieurs stratégies :**

```bash
#!/bin/bash
# rotate_advanced.sh

BACKUP_DIR="/var/backups/postgresql"  
LOG_FILE="/var/log/postgresql_backup/rotation.log"  

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=========================================="  
log "🗑️ Rotation des backups PostgreSQL"  
log "=========================================="  

# Compter les backups avant rotation
BEFORE_COUNT=$(find "$BACKUP_DIR" -name "*.dump" | wc -l)  
BEFORE_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)  

log "📊 Avant rotation :"  
log "  Fichiers : $BEFORE_COUNT"  
log "  Taille : $BEFORE_SIZE"  
log ""  

# Stratégie 1 : Supprimer backups corrompus
log "🔍 Recherche de backups corrompus..."  
CORRUPTED_COUNT=0  

for dump_file in "$BACKUP_DIR"/*.dump; do
    if [ -f "$dump_file" ]; then
        if ! pg_restore --list "$dump_file" > /dev/null 2>&1; then
            log "⚠️ Backup corrompu détecté : $(basename "$dump_file")"
            rm "$dump_file"
            ((CORRUPTED_COUNT++))
        fi
    fi
done

if [ $CORRUPTED_COUNT -gt 0 ]; then
    log "🗑️ $CORRUPTED_COUNT backup(s) corrompu(s) supprimé(s)"
else
    log "✅ Aucun backup corrompu"
fi  
log ""  

# Stratégie 2 : Rotation par âge (quotidiens > 7 jours)
log "📅 Rotation des backups quotidiens (> 7 jours)..."  
DAILY_DELETED=$(find "$BACKUP_DIR" -name "*daily*.dump" -mtime +7 -delete -print | wc -l)  
log "🗑️ $DAILY_DELETED quotidien(s) supprimé(s)"  
log ""  

# Stratégie 3 : Garder maximum 10 backups hebdomadaires
log "📅 Rotation des backups hebdomadaires (garder 10 max)..."  
WEEKLY_COUNT=$(ls -t "$BACKUP_DIR"/*weekly*.dump 2>/dev/null | wc -l)  

if [ "$WEEKLY_COUNT" -gt 10 ]; then
    WEEKLY_TO_DELETE=$((WEEKLY_COUNT - 10))
    ls -t "$BACKUP_DIR"/*weekly*.dump | tail -n "$WEEKLY_TO_DELETE" | xargs rm
    log "🗑️ $WEEKLY_TO_DELETE hebdomadaire(s) supprimé(s)"
else
    log "✅ Hebdomadaires OK ($WEEKLY_COUNT/10)"
fi  
log ""  

# Stratégie 4 : Rotation des mensuels (> 1 an)
log "📅 Rotation des backups mensuels (> 365 jours)..."  
MONTHLY_DELETED=$(find "$BACKUP_DIR" -name "*monthly*.dump" -mtime +365 -delete -print | wc -l)  
log "🗑️ $MONTHLY_DELETED mensuel(s) supprimé(s)"  
log ""  

# Stratégie 5 : Vérifier l'espace disque
log "💾 Vérification de l'espace disque..."  
DISK_USAGE=$(df -h "$BACKUP_DIR" | tail -1 | awk '{print $5}' | sed 's/%//')  

if [ "$DISK_USAGE" -gt 90 ]; then
    log "⚠️ ALERTE : Espace disque critique ($DISK_USAGE%)"
    log "Nettoyage d'urgence des backups les plus anciens..."

    # Supprimer les 5 plus anciens backups
    ls -t "$BACKUP_DIR"/*.dump | tail -5 | xargs rm
    log "🗑️ 5 backups supprimés en urgence"
else
    log "✅ Espace disque OK ($DISK_USAGE%)"
fi  
log ""  

# Résumé après rotation
AFTER_COUNT=$(find "$BACKUP_DIR" -name "*.dump" | wc -l)  
AFTER_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)  
DELETED=$((BEFORE_COUNT - AFTER_COUNT))  

log "=========================================="  
log "📊 Après rotation :"  
log "  Fichiers : $AFTER_COUNT (- $DELETED)"  
log "  Taille : $AFTER_SIZE"  
log "=========================================="  
log "✅ Rotation terminée"  
log ""  
```

**Ce script :**
- ✅ Détecte et supprime les backups corrompus  
- ✅ Applique différentes politiques de rétention  
- ✅ Gère les alertes d'espace disque  
- ✅ Log toutes les actions  
- ✅ Affiche un résumé clair

### 6.4. Automatiser la Rotation avec Cron

**Ajouter dans le crontab :**

```bash
# Rotation quotidienne à 23h
0 23 * * * /usr/local/bin/rotate_advanced.sh

# Ou après chaque backup
0 3 * * * /usr/local/bin/backup_daily.sh && /usr/local/bin/rotate_advanced.sh
```

---

## 7. Destinations de Sauvegarde

### 7.1. Stockage Local

**Avantages :**
- ✅ Rapide (accès disque)  
- ✅ Gratuit (pas de coût cloud)  
- ✅ Contrôle total

**Inconvénients :**
- ❌ Vulnérable aux pannes matérielles  
- ❌ Pas de protection contre incendie/désastre  
- ❌ Limité en espace

**Configuration :**

```bash
BACKUP_DIR="/var/backups/postgresql"
```

### 7.2. Stockage NAS (Network Attached Storage)

**Principe :** Sauvegarder vers un serveur de fichiers réseau.

**Avantages :**
- ✅ Séparation physique du serveur de production  
- ✅ Capacité extensible  
- ✅ Redondance RAID  
- ✅ Accessible depuis plusieurs serveurs

**Inconvénients :**
- ❌ Dépend du réseau local  
- ❌ Coût matériel initial

**Script de copie vers NAS :**

```bash
#!/bin/bash
# backup_to_nas.sh

LOCAL_BACKUP_DIR="/var/backups/postgresql"  
NAS_MOUNT_POINT="/mnt/nas"  
NAS_BACKUP_DIR="$NAS_MOUNT_POINT/postgresql_backups"  

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "=========================================="  
log "📡 Copie vers NAS"  
log "=========================================="  

# Vérifier que le NAS est monté
if ! mountpoint -q "$NAS_MOUNT_POINT"; then
    log "❌ ERREUR : NAS non monté à $NAS_MOUNT_POINT"

    # Tenter de monter
    log "Tentative de montage du NAS..."
    mount "$NAS_MOUNT_POINT"

    if [ $? -ne 0 ]; then
        log "❌ Échec du montage du NAS"
        exit 1
    fi

    log "✅ NAS monté avec succès"
fi

# Créer le répertoire sur le NAS
mkdir -p "$NAS_BACKUP_DIR"

# Synchroniser avec rsync
log "Synchronisation en cours..."  
rsync -av --progress \  
    --exclude="*.log" \
    "$LOCAL_BACKUP_DIR/" \
    "$NAS_BACKUP_DIR/"

if [ $? -eq 0 ]; then
    # Statistiques
    LOCAL_SIZE=$(du -sh "$LOCAL_BACKUP_DIR" | cut -f1)
    NAS_SIZE=$(du -sh "$NAS_BACKUP_DIR" | cut -f1)

    log "✅ Synchronisation réussie"
    log "  Local : $LOCAL_SIZE"
    log "  NAS : $NAS_SIZE"
else
    log "❌ Échec de la synchronisation"
    exit 1
fi

log "=========================================="  
log "✅ Copie NAS terminée"  
log "=========================================="  
```

**Montage NFS (exemple) :**

```bash
# /etc/fstab
nas.example.com:/volume1/backups /mnt/nas nfs defaults,_netdev 0 0

# Monter
mount /mnt/nas
```

**Montage CIFS/SMB (Windows share) :**

```bash
# /etc/fstab
//nas.example.com/backups /mnt/nas cifs credentials=/root/.nas_credentials,uid=postgres,gid=postgres 0 0

# Fichier .nas_credentials
# username=backup_user
# password=secure_password

# Monter
mount /mnt/nas
```

### 7.3. Stockage Cloud (AWS S3)

**Avantages :**
- ✅ Stockage illimité  
- ✅ Haute disponibilité (99.999999999%)  
- ✅ Protection géographique  
- ✅ Versioning intégré  
- ✅ Lifecycle policies automatiques

**Inconvénients :**
- ❌ Coûts récurrents  
- ❌ Dépend d'Internet  
- ❌ Transferts lents pour gros backups

**Prérequis : Installer AWS CLI**

```bash
# Ubuntu/Debian
sudo apt-get install awscli

# Configurer AWS CLI
aws configure
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region: eu-west-1
# Default output format: json
```

**Script de backup vers S3 :**

```bash
#!/bin/bash
# backup_to_s3.sh

LOCAL_BACKUP_DIR="/var/backups/postgresql"  
S3_BUCKET="s3://my-company-backups/postgresql"  
LOG_FILE="/var/log/postgresql_backup/s3_sync.log"  

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=========================================="  
log "☁️ Upload vers AWS S3"  
log "=========================================="  
log "Source : $LOCAL_BACKUP_DIR"  
log "Destination : $S3_BUCKET"  
log ""  

# Vérifier AWS CLI
if ! command -v aws > /dev/null 2>&1; then
    log "❌ ERREUR : AWS CLI non installé"
    exit 1
fi

# Vérifier les credentials
if ! aws sts get-caller-identity > /dev/null 2>&1; then
    log "❌ ERREUR : Credentials AWS invalides"
    exit 1
fi

log "✅ Credentials AWS valides"  
log ""  

# Synchroniser vers S3
log "Upload en cours..."  
START_TIME=$(date +%s)  

aws s3 sync "$LOCAL_BACKUP_DIR" "$S3_BUCKET" \
    --storage-class STANDARD_IA \
    --exclude "*.log" \
    --exclude "*.tmp" \
    2>&1 | tee -a "$LOG_FILE"

if [ ${PIPESTATUS[0]} -eq 0 ]; then
    END_TIME=$(date +%s)
    DURATION=$((END_TIME - START_TIME))

    log ""
    log "✅ Upload réussi en ${DURATION}s"

    # Lister les fichiers sur S3
    FILE_COUNT=$(aws s3 ls "$S3_BUCKET" --recursive | wc -l)
    log "📦 Fichiers sur S3 : $FILE_COUNT"
else
    log "❌ Échec de l'upload vers S3"
    exit 1
fi

log "=========================================="  
log "✅ Backup S3 terminé"  
log "=========================================="  
```

**Classe de stockage S3 :**
- `STANDARD` : Accès fréquent (cher)  
- `STANDARD_IA` : Accès peu fréquent (recommandé pour backups)  
- `GLACIER` : Archivage long terme (très peu cher, retrieval lent)

**Lifecycle Policy S3 (optionnel) :**

```json
{
  "Rules": [
    {
      "Id": "Move to Glacier after 30 days",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

### 7.4. Stockage Multi-Destinations

**Combiner plusieurs destinations pour maximum de sécurité :**

```bash
#!/bin/bash
# backup_multi_destinations.sh

DATABASE="production"  
TIMESTAMP=$(date +%Y%m%d_%H%M%S)  

# Destinations
LOCAL_DIR="/var/backups/postgresql"  
NAS_DIR="/mnt/nas/postgresql_backups"  
S3_BUCKET="s3://my-backups/postgresql"  

BACKUP_FILE="$LOCAL_DIR/${DATABASE}_${TIMESTAMP}.dump"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "=========================================="  
log "🌐 Backup Multi-Destinations"  
log "=========================================="  

# Étape 1 : Backup local
log "1️⃣ Création du backup local..."  
mkdir -p "$LOCAL_DIR"  
pg_dump -F c -Z 9 "$DATABASE" -f "$BACKUP_FILE"  

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "✅ Backup local créé : $SIZE"
else
    log "❌ Échec du backup local"
    exit 1
fi

# Étape 2 : Copie vers NAS
log ""  
log "2️⃣ Copie vers NAS..."  

if mountpoint -q "/mnt/nas"; then
    mkdir -p "$NAS_DIR"
    cp "$BACKUP_FILE" "$NAS_DIR/"

    if [ $? -eq 0 ]; then
        log "✅ Copie NAS réussie"
    else
        log "⚠️ Échec copie NAS (non bloquant)"
    fi
else
    log "⚠️ NAS non monté (skip)"
fi

# Étape 3 : Upload vers S3
log ""  
log "3️⃣ Upload vers S3..."  

if command -v aws > /dev/null 2>&1; then
    aws s3 cp "$BACKUP_FILE" "$S3_BUCKET/$(basename "$BACKUP_FILE")" \
        --storage-class STANDARD_IA

    if [ $? -eq 0 ]; then
        log "✅ Upload S3 réussi"
    else
        log "⚠️ Échec upload S3 (non bloquant)"
    fi
else
    log "⚠️ AWS CLI non disponible (skip)"
fi

# Résumé
log ""  
log "=========================================="  
log "📊 Résumé"  
log "=========================================="  
log "✅ Local : $BACKUP_FILE"  

if [ -f "$NAS_DIR/$(basename "$BACKUP_FILE")" ]; then
    log "✅ NAS : Copié"
else
    log "❌ NAS : Échec"
fi

if aws s3 ls "$S3_BUCKET/$(basename "$BACKUP_FILE")" > /dev/null 2>&1; then
    log "✅ S3 : Uploadé"
else
    log "❌ S3 : Échec"
fi

log "=========================================="  
log "✅ Backup multi-destinations terminé"  
log "=========================================="  
```

**Principe 3-2-1 respecté :**
- ✅ 3 copies (local + NAS + S3)  
- ✅ 2 supports (disque local + cloud)  
- ✅ 1 hors site (S3)

---

## 8. Monitoring et Alertes

### 8.1. Pourquoi Monitorer les Backups ?

**Un backup non surveillé = pas de backup**

**Scénarios catastrophes évités par le monitoring :**
- Backup échoue pendant 2 semaines → détecté immédiatement
- Disque plein → alerte avant saturation
- Backup corrompu → détecté et corrigé
- Script mal configuré → identifié rapidement

### 8.2. Métriques à Surveiller

| Métrique | Seuil Alerte | Impact |
|----------|--------------|--------|
| **Succès/Échec** | 1 échec | Critique |
| **Durée backup** | > 2× normale | Avertissement |
| **Taille backup** | > 2× normale | Avertissement |
| **Espace disque** | > 85% | Critique |
| **Âge du dernier backup** | > 36h | Critique |
| **Backups corrompus** | > 0 | Critique |

### 8.3. Script de Monitoring Simple

```bash
#!/bin/bash
# monitor_backups.sh

BACKUP_DIR="/var/backups/postgresql"  
ALERT_EMAIL="admin@example.com"  
MAX_AGE_HOURS=36  # Alerte si dernier backup > 36h  
MIN_DISK_SPACE=15  # GB minimum  

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

send_alert() {
    local subject="$1"
    local body="$2"
    echo "$body" | mail -s "⚠️ $subject" "$ALERT_EMAIL"
    log "📧 Alerte envoyée : $subject"
}

log "=========================================="  
log "🔍 Monitoring des Backups PostgreSQL"  
log "=========================================="  

ALERTS=()

# Vérification 1 : Dernier backup
log "Vérification du dernier backup..."  
LATEST_BACKUP=$(find "$BACKUP_DIR" -name "*.dump" -type f -printf '%T@ %p\n' | sort -rn | head -1 | cut -d' ' -f2)  

if [ -z "$LATEST_BACKUP" ]; then
    ALERTS+=("❌ AUCUN BACKUP TROUVÉ")
    log "❌ Aucun backup trouvé !"
else
    LATEST_AGE_HOURS=$(( ($(date +%s) - $(stat -c %Y "$LATEST_BACKUP")) / 3600 ))
    log "Dernier backup : $(basename "$LATEST_BACKUP") ($LATEST_AGE_HOURS heures)"

    if [ "$LATEST_AGE_HOURS" -gt "$MAX_AGE_HOURS" ]; then
        ALERTS+=("⚠️ Dernier backup trop ancien : ${LATEST_AGE_HOURS}h (max: ${MAX_AGE_HOURS}h)")
        log "⚠️ Backup trop ancien !"
    else
        log "✅ Âge du backup OK"
    fi
fi

# Vérification 2 : Espace disque
log ""  
log "Vérification de l'espace disque..."  
AVAILABLE_GB=$(df -BG "$BACKUP_DIR" | tail -1 | awk '{print $4}' | sed 's/G//')  
USAGE_PCT=$(df -h "$BACKUP_DIR" | tail -1 | awk '{print $5}' | sed 's/%//')  

log "Espace disponible : ${AVAILABLE_GB}GB (${USAGE_PCT}% utilisé)"

if [ "$AVAILABLE_GB" -lt "$MIN_DISK_SPACE" ]; then
    ALERTS+=("❌ Espace disque critique : ${AVAILABLE_GB}GB restant (min: ${MIN_DISK_SPACE}GB)")
    log "❌ Espace disque critique !"
elif [ "$USAGE_PCT" -gt 85 ]; then
    ALERTS+=("⚠️ Disque à ${USAGE_PCT}% (seuil: 85%)")
    log "⚠️ Disque bientôt plein"
else
    log "✅ Espace disque OK"
fi

# Vérification 3 : Intégrité des backups récents
log ""  
log "Vérification de l'intégrité..."  
CORRUPTED=0  

# Vérifier les 3 derniers backups
for dump_file in $(find "$BACKUP_DIR" -name "*.dump" -type f -printf '%T@ %p\n' | sort -rn | head -3 | cut -d' ' -f2); do
    if ! pg_restore --list "$dump_file" > /dev/null 2>&1; then
        ALERTS+=("❌ Backup corrompu : $(basename "$dump_file")")
        log "❌ Corrompu : $(basename "$dump_file")"
        ((CORRUPTED++))
    fi
done

if [ $CORRUPTED -eq 0 ]; then
    log "✅ Intégrité OK"
else
    log "❌ $CORRUPTED backup(s) corrompu(s)"
fi

# Vérification 4 : Nombre de backups
log ""  
log "Vérification du nombre de backups..."  
BACKUP_COUNT=$(find "$BACKUP_DIR" -name "*.dump" -type f | wc -l)  
log "Nombre de backups : $BACKUP_COUNT"  

if [ "$BACKUP_COUNT" -lt 3 ]; then
    ALERTS+=("⚠️ Trop peu de backups : $BACKUP_COUNT (min recommandé: 7)")
    log "⚠️ Peu de backups disponibles"
else
    log "✅ Nombre de backups suffisant"
fi

# Résumé et envoi d'alertes
log ""  
log "=========================================="  
log "📊 RÉSUMÉ"  
log "=========================================="  

if [ ${#ALERTS[@]} -eq 0 ]; then
    log "✅ Tous les contrôles passés avec succès"
    log "=========================================="
    exit 0
else
    log "⚠️ ${#ALERTS[@]} alerte(s) détectée(s) :"
    for alert in "${ALERTS[@]}"; do
        log "  $alert"
    done
    log "=========================================="

    # Envoyer email d'alerte
    ALERT_BODY=$(printf "%s\n" "${ALERTS[@]}")
    send_alert "Monitoring Backups PostgreSQL - Alertes" "$ALERT_BODY"

    exit 1
fi
```

**Automatiser le monitoring :**

```bash
# Dans crontab : vérifier toutes les heures
0 * * * * /usr/local/bin/monitor_backups.sh >> /var/log/postgresql_backup/monitor.log 2>&1
```

### 8.4. Dashboard de Monitoring (Script de Rapport)

```bash
#!/bin/bash
# backup_dashboard.sh

BACKUP_DIR="/var/backups/postgresql"

echo "=========================================="  
echo "📊 DASHBOARD BACKUPS POSTGRESQL"  
echo "=========================================="  
echo "Date : $(date '+%Y-%m-%d %H:%M:%S')"  
echo ""  

# Section 1 : Statistiques générales
echo "📦 STATISTIQUES GÉNÉRALES"  
echo "=========================================="  
TOTAL_BACKUPS=$(find "$BACKUP_DIR" -name "*.dump" | wc -l)  
TOTAL_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)  
echo "Nombre total de backups : $TOTAL_BACKUPS"  
echo "Espace utilisé : $TOTAL_SIZE"  
echo ""  

# Section 2 : Dernier backup
echo "🕒 DERNIER BACKUP"  
echo "=========================================="  
LATEST=$(find "$BACKUP_DIR" -name "*.dump" -type f -printf '%T@ %p\n' | sort -rn | head -1)  
if [ -n "$LATEST" ]; then  
    LATEST_FILE=$(echo "$LATEST" | cut -d' ' -f2)
    LATEST_DATE=$(stat -c %y "$LATEST_FILE" | cut -d. -f1)
    LATEST_SIZE=$(du -h "$LATEST_FILE" | cut -f1)
    LATEST_AGE_HOURS=$(( ($(date +%s) - $(stat -c %Y "$LATEST_FILE")) / 3600 ))

    echo "Fichier : $(basename "$LATEST_FILE")"
    echo "Date : $LATEST_DATE ($LATEST_AGE_HOURS heures)"
    echo "Taille : $LATEST_SIZE"
else
    echo "❌ Aucun backup trouvé"
fi  
echo ""  

# Section 3 : Répartition par type (si GFS)
echo "📅 RÉPARTITION PAR TYPE"  
echo "=========================================="  
DAILY=$(find "$BACKUP_DIR" -name "*daily*.dump" | wc -l)  
WEEKLY=$(find "$BACKUP_DIR" -name "*weekly*.dump" | wc -l)  
MONTHLY=$(find "$BACKUP_DIR" -name "*monthly*.dump" | wc -l)  
YEARLY=$(find "$BACKUP_DIR" -name "*yearly*.dump" | wc -l)  

echo "Quotidiens : $DAILY"  
echo "Hebdomadaires : $WEEKLY"  
echo "Mensuels : $MONTHLY"  
echo "Annuels : $YEARLY"  
echo ""  

# Section 4 : Espace disque
echo "💾 ESPACE DISQUE"  
echo "=========================================="  
df -h "$BACKUP_DIR" | tail -1 | awk '{print "Partition : "$1"\nTaille : "$2"\nUtilisé : "$3" ("$5")\nDisponible : "$4}'  
echo ""  

# Section 5 : Les 5 backups les plus récents
echo "📋 5 BACKUPS LES PLUS RÉCENTS"  
echo "=========================================="  
find "$BACKUP_DIR" -name "*.dump" -type f -printf '%T@ %p\n' | sort -rn | head -5 | while read timestamp file; do  
    filename=$(basename "$file")
    size=$(du -h "$file" | cut -f1)
    date=$(date -d @"${timestamp%.*}" '+%Y-%m-%d %H:%M:%S')
    echo "• $filename - $size - $date"
done  
echo ""  

# Section 6 : Santé
echo "🏥 SANTÉ DES BACKUPS"  
echo "=========================================="  

# Vérifier les 3 derniers
RECENT_BACKUPS=$(find "$BACKUP_DIR" -name "*.dump" -type f -printf '%T@ %p\n' | sort -rn | head -3 | cut -d' ' -f2)  
CORRUPTED_COUNT=0  

for backup in $RECENT_BACKUPS; do
    if pg_restore --list "$backup" > /dev/null 2>&1; then
        echo "✅ $(basename "$backup") : OK"
    else
        echo "❌ $(basename "$backup") : CORROMPU"
        ((CORRUPTED_COUNT++))
    fi
done

if [ $CORRUPTED_COUNT -eq 0 ]; then
    echo ""
    echo "✅ Tous les backups récents sont valides"
else
    echo ""
    echo "⚠️ $CORRUPTED_COUNT backup(s) corrompu(s) détecté(s) !"
fi

echo ""  
echo "=========================================="  
echo "Fin du rapport"  
echo "=========================================="  
```

**Générer un rapport quotidien :**

```bash
# Dans crontab : rapport tous les matins à 9h
0 9 * * * /usr/local/bin/backup_dashboard.sh | mail -s "Rapport Backups PostgreSQL" admin@example.com
```

### 8.5. Intégration avec des Outils de Monitoring

#### 8.5.1. Prometheus + Grafana

**Script d'export de métriques :**

```bash
#!/bin/bash
# export_prometheus_metrics.sh

BACKUP_DIR="/var/backups/postgresql"  
METRICS_FILE="/var/lib/node_exporter/textfile_collector/postgresql_backup.prom"  

# Calculer les métriques
BACKUP_COUNT=$(find "$BACKUP_DIR" -name "*.dump" | wc -l)  
TOTAL_SIZE_BYTES=$(du -sb "$BACKUP_DIR" | cut -f1)  
LATEST_TIMESTAMP=$(find "$BACKUP_DIR" -name "*.dump" -type f -printf '%T@\n' | sort -rn | head -1)  
LATEST_AGE_SECONDS=$(( $(date +%s) - ${LATEST_TIMESTAMP%.*} ))  

# Générer le fichier de métriques Prometheus
cat > "$METRICS_FILE" << EOF
# HELP postgresql_backup_count Number of backup files
# TYPE postgresql_backup_count gauge
postgresql_backup_count $BACKUP_COUNT

# HELP postgresql_backup_size_bytes Total size of backups in bytes
# TYPE postgresql_backup_size_bytes gauge
postgresql_backup_size_bytes $TOTAL_SIZE_BYTES

# HELP postgresql_backup_age_seconds Age of the latest backup in seconds
# TYPE postgresql_backup_age_seconds gauge
postgresql_backup_age_seconds $LATEST_AGE_SECONDS  
EOF  
```

**Automatiser l'export :**

```bash
# Dans crontab : toutes les 5 minutes
*/5 * * * * /usr/local/bin/export_prometheus_metrics.sh
```

#### 8.5.2. Alertes Slack/Discord

```bash
#!/bin/bash
# send_slack_alert.sh

SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

send_slack() {
    local message="$1"
    local color="${2:-#ff0000}"  # Rouge par défaut

    curl -X POST "$SLACK_WEBHOOK_URL" \
        -H 'Content-Type: application/json' \
        -d "{
            \"attachments\": [
                {
                    \"color\": \"$color\",
                    \"title\": \"🔔 Alerte Backup PostgreSQL\",
                    \"text\": \"$message\",
                    \"footer\": \"Serveur: $(hostname)\",
                    \"ts\": $(date +%s)
                }
            ]
        }"
}

# Exemple d'utilisation
if [ $BACKUP_FAILED ]; then
    send_slack "❌ Échec du backup de la base production" "#ff0000"
else
    send_slack "✅ Backup réussi : 50GB en 30min" "#00ff00"
fi
```

---

## 9. Scripts de Vérification

### 9.1. Script de Test de Restauration

**Le test ultime : restaurer le backup**

```bash
#!/bin/bash
# test_restore.sh

BACKUP_FILE="$1"  
TEST_DB="test_restore_$(date +%s)"  

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "=========================================="  
log "🧪 Test de Restauration"  
log "=========================================="  
log "Backup : $BACKUP_FILE"  
log "Base de test : $TEST_DB"  
log ""  

# Vérifier que le backup existe
if [ ! -f "$BACKUP_FILE" ]; then
    log "❌ Fichier de backup introuvable"
    exit 1
fi

# Vérifier l'intégrité
log "1️⃣ Vérification de l'intégrité..."  
if pg_restore --list "$BACKUP_FILE" > /dev/null 2>&1; then  
    log "✅ Backup valide"
else
    log "❌ Backup corrompu"
    exit 1
fi

# Créer une base de test
log ""  
log "2️⃣ Création de la base de test..."  
createdb "$TEST_DB"  

if [ $? -ne 0 ]; then
    log "❌ Impossible de créer la base de test"
    exit 1
fi  
log "✅ Base créée"  

# Restaurer
log ""  
log "3️⃣ Restauration en cours..."  
START_TIME=$(date +%s)  

pg_restore -d "$TEST_DB" -j 4 --no-owner --no-acl "$BACKUP_FILE" 2>&1 | grep -i error

RESTORE_STATUS=${PIPESTATUS[0]}  
END_TIME=$(date +%s)  
DURATION=$((END_TIME - START_TIME))  

if [ $RESTORE_STATUS -eq 0 ]; then
    log "✅ Restauration réussie en ${DURATION}s"
else
    log "⚠️ Restauration terminée avec avertissements"
fi

# Vérifier la base restaurée
log ""  
log "4️⃣ Vérification de la base restaurée..."  

# Compter les tables
TABLE_COUNT=$(psql -t -d "$TEST_DB" -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public'" | xargs)  
log "Nombre de tables : $TABLE_COUNT"  

# Taille de la base
DB_SIZE=$(psql -t -d "$TEST_DB" -c "SELECT pg_size_pretty(pg_database_size('$TEST_DB'))" | xargs)  
log "Taille de la base : $DB_SIZE"  

# Vérifier qu'il y a des données
TOTAL_ROWS=0  
for table in $(psql -t -d "$TEST_DB" -c "SELECT tablename FROM pg_tables WHERE schemaname = 'public'"); do  
    ROW_COUNT=$(psql -t -d "$TEST_DB" -c "SELECT COUNT(*) FROM $table" 2>/dev/null | xargs)
    TOTAL_ROWS=$((TOTAL_ROWS + ROW_COUNT))
done  
log "Nombre total de lignes : $TOTAL_ROWS"  

if [ $TOTAL_ROWS -gt 0 ]; then
    log "✅ Données présentes"
else
    log "⚠️ Aucune donnée trouvée"
fi

# Nettoyage
log ""  
log "5️⃣ Nettoyage..."  
dropdb "$TEST_DB"  
log "✅ Base de test supprimée"  

log ""  
log "=========================================="  
log "✅ Test de restauration terminé avec succès"  
log "=========================================="  
log "Résumé :"  
log "  • Durée restauration : ${DURATION}s"  
log "  • Tables : $TABLE_COUNT"  
log "  • Lignes : $TOTAL_ROWS"  
log "  • Taille : $DB_SIZE"  
log "=========================================="  
```

**Automatiser les tests mensuels :**

```bash
# Dans crontab : test le 1er du mois à 5h
0 5 1 * * /usr/local/bin/test_restore.sh /var/backups/postgresql/latest.dump >> /var/log/postgresql_backup/restore_test.log 2>&1
```

### 9.2. Script de Comparaison Taille Source vs Backup

```bash
#!/bin/bash
# compare_sizes.sh

DATABASE="production"  
BACKUP_FILE="$1"  

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "=========================================="  
log "📊 Comparaison Taille Source vs Backup"  
log "=========================================="  

# Taille de la base source
SOURCE_SIZE_BYTES=$(psql -t -d "$DATABASE" -c "SELECT pg_database_size('$DATABASE')" | xargs)  
SOURCE_SIZE_PRETTY=$(psql -t -d "$DATABASE" -c "SELECT pg_size_pretty(pg_database_size('$DATABASE'))" | xargs)  

# Taille du backup
BACKUP_SIZE_BYTES=$(stat -c %s "$BACKUP_FILE")  
BACKUP_SIZE_PRETTY=$(du -h "$BACKUP_FILE" | cut -f1)  

# Ratio de compression
RATIO=$(echo "scale=2; $BACKUP_SIZE_BYTES * 100 / $SOURCE_SIZE_BYTES" | bc)

log "Base source : $SOURCE_SIZE_PRETTY ($SOURCE_SIZE_BYTES bytes)"  
log "Backup : $BACKUP_SIZE_PRETTY ($BACKUP_SIZE_BYTES bytes)"  
log "Ratio compression : ${RATIO}%"  

# Alertes
if (( $(echo "$RATIO > 150" | bc -l) )); then
    log "⚠️ ATTENTION : Backup plus gros que la source !"
elif (( $(echo "$RATIO < 10" | bc -l) )); then
    log "⚠️ ATTENTION : Compression anormalement élevée"
else
    log "✅ Ratio normal"
fi

log "=========================================="
```

### 9.3. Script de Vérification d'Intégrité Complète

```bash
#!/bin/bash
# verify_all_backups.sh

BACKUP_DIR="/var/backups/postgresql"  
LOG_FILE="/var/log/postgresql_backup/integrity_$(date +%Y%m%d).log"  

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=========================================="  
log "🔍 Vérification Intégrité de TOUS les Backups"  
log "=========================================="  

TOTAL=0  
VALID=0  
CORRUPTED=0  
ERRORS=()  

# Parcourir tous les backups
while IFS= read -r dump_file; do
    ((TOTAL++))
    filename=$(basename "$dump_file")

    log "Vérification : $filename..."

    if pg_restore --list "$dump_file" > /dev/null 2>&1; then
        log "  ✅ OK"
        ((VALID++))
    else
        log "  ❌ CORROMPU"
        ((CORRUPTED++))
        ERRORS+=("$filename")
    fi
done < <(find "$BACKUP_DIR" -name "*.dump" -type f)

# Résumé
log ""  
log "=========================================="  
log "📊 RÉSUMÉ"  
log "=========================================="  
log "Total vérifié : $TOTAL"  
log "Valides : $VALID"  
log "Corrompus : $CORRUPTED"  

if [ $CORRUPTED -gt 0 ]; then
    log ""
    log "⚠️ Backups corrompus détectés :"
    for error in "${ERRORS[@]}"; do
        log "  • $error"
    done
    log ""
    log "❌ Vérification terminée avec erreurs"
    exit 1
else
    log ""
    log "✅ Tous les backups sont valides"
    exit 0
fi
```

---

## 10. Bonnes Pratiques

### 10.1. Checklist de Production

**Avant de mettre en production :**

- [ ] **Scripts testés** manuellement plusieurs fois  
- [ ] **Logs configurés** et centralisés  
- [ ] **Rotation mise en place** (éviter saturation disque)  
- [ ] **Alertes configurées** (email, Slack, etc.)  
- [ ] **Monitoring actif** (dashboard, métriques)  
- [ ] **Stockage distant** configuré (NAS, Cloud)  
- [ ] **Tests de restauration** réguliers automatisés  
- [ ] **Documentation** à jour (runbook)  
- [ ] **Credentials sécurisés** (.pgpass avec chmod 600)  
- [ ] **Politique de rétention** définie (GFS recommandé)  
- [ ] **Backup des objets globaux** (rôles, tablespaces)  
- [ ] **Cron configuré** et vérifié  
- [ ] **Espace disque suffisant** (minimum 2× taille base)

### 10.2. Architecture Recommandée

```
Production PostgreSQL Server
         │
         ├─→ [Backup Quotidien] @ 2h
         │       │
         │       ├─→ Local (/var/backups)
         │       ├─→ NAS (rsync @ 3h)
         │       └─→ S3 (aws s3 sync @ 4h)
         │
         ├─→ [Backup Hebdomadaire] @ Dimanche 3h
         │       └─→ Archivage long terme
         │
         ├─→ [Monitoring] @ Toutes les heures
         │       └─→ Alertes si problème
         │
         └─→ [Test Restauration] @ 1er du mois
                 └─→ Rapport d'intégrité
```

### 10.3. Sécurité

**Protection des Backups :**

```bash
# Permissions strictes
chmod 700 /var/backups/postgresql  
chmod 600 /var/backups/postgresql/*.dump  

# Propriétaire postgres uniquement
chown -R postgres:postgres /var/backups/postgresql

# Chiffrer les backups sensibles (optionnel)
gpg --encrypt --recipient admin@example.com backup.dump
```

**Protection des Credentials :**

```bash
# Fichier .pgpass
chmod 600 ~/.pgpass  
chown postgres:postgres ~/.pgpass  

# Ne JAMAIS mettre de mots de passe dans les scripts
# Utiliser .pgpass ou variables d'environnement
```

### 10.4. Performance

**Optimiser les Backups :**

```bash
# 1. Utiliser le parallélisme
pg_dump -F d -j 8 mabase -f backup/

# 2. Compression adaptée
pg_dump -F c -Z 6 mabase  # Équilibre vitesse/compression

# 3. Exclure les tables volumineuses non critiques
pg_dump -T logs -T cache mabase

# 4. Planifier hors heures de pointe
# Cron à 2h du matin, pas à 14h !

# 5. Utiliser des liens symboliques pour "latest"
ln -sf backup_20251121.dump latest.dump
```

### 10.5. Documentation (Runbook)

**Créer un document de procédure :**

```markdown
# Runbook : Backups PostgreSQL

## Contacts
- Responsable : admin@example.com
- Astreinte : +33 6 XX XX XX XX

## Emplacements
- Scripts : /usr/local/bin/
- Backups : /var/backups/postgresql/
- Logs : /var/log/postgresql_backup/

## Procédures

### Backup Manuel d'Urgence
sudo /usr/local/bin/backup_production.sh

### Restauration d'Urgence
1. Arrêter l'application
2. sudo /usr/local/bin/restore_latest.sh
3. Vérifier l'intégrité
4. Redémarrer l'application

### Problèmes Courants

#### Backup échoue
1. Vérifier l'espace disque : df -h /var/backups
2. Vérifier PostgreSQL : pg_isready
3. Consulter les logs : tail -f /var/log/postgresql_backup/backup.log

#### Disque plein
1. Lancer rotation manuelle : /usr/local/bin/rotate_advanced.sh
2. Si insuffisant, copier vers NAS puis supprimer localement
3. Augmenter la rétention si récurrent

## Tests
- Restauration testée : 1er de chaque mois
- Dernier test : 2025-11-01 ✅
```

### 10.6. Évolution et Amélioration Continue

**Métriques à suivre mensuellement :**
- Taille moyenne des backups (tendance)
- Durée moyenne des backups (dégradation ?)
- Taux de réussite (objectif : 99.9%)
- Espace disque consommé
- Coûts cloud (S3, etc.)

**Questions à se poser :**
- Les backups sont-ils toujours adaptés à la taille de la base ?
- Le temps de restauration est-il acceptable ?
- Les tests de restauration sont-ils réguliers ?
- Les alertes sont-elles pertinentes (pas trop de faux positifs) ?
- La documentation est-elle à jour ?

---

## Conclusion

### Ce que vous avez appris

- ✅ **Stratégies de backup** : Fréquence, rétention, GFS  
- ✅ **Scripts de base à avancés** : Du minimal au production-ready  
- ✅ **Automatisation avec Cron** : Planification et surveillance  
- ✅ **Rotation intelligente** : Gérer l'historique sans saturer  
- ✅ **Destinations multiples** : Local, NAS, Cloud (règle 3-2-1)  
- ✅ **Monitoring et alertes** : Détecter les problèmes avant la catastrophe  
- ✅ **Tests de restauration** : Valider que les backups fonctionnent  
- ✅ **Bonnes pratiques** : Sécurité, performance, documentation

### Points Clés à Retenir

1. **Un backup non testé = pas de backup**
   - Testez régulièrement la restauration

2. **Appliquez la règle 3-2-1**
   - 3 copies, 2 supports, 1 hors site

3. **Automatisez tout**
   - Scripts + Cron + Monitoring = Sérénité

4. **Surveillez vos backups**
   - Alertes immédiates en cas de problème

5. **Documentez**
   - Runbook à jour = Intervention rapide

### Prochaines Étapes

1. **Implémenter un script simple** pour votre environnement  
2. **Ajouter la rotation** pour ne pas saturer le disque  
3. **Configurer Cron** pour automatiser  
4. **Mettre en place les alertes** (email minimum)  
5. **Tester une restauration** manuelle  
6. **Étendre vers le cloud** (S3, etc.)  
7. **Automatiser les tests** mensuels

### Ressources Complémentaires

**Documentation PostgreSQL :**
- pg_dump : https://www.postgresql.org/docs/18/app-pgdump.html
- pg_restore : https://www.postgresql.org/docs/18/app-pgrestore.html
- Backup & Restore : https://www.postgresql.org/docs/18/backup.html

**Outils tiers recommandés :**
- **pgBackRest** : https://pgbackrest.org/  
- **Barman** : https://www.pgbarman.org/  
- **WAL-G** : https://github.com/wal-g/wal-g

**Générateur Cron :**
- https://crontab.guru/

---

**Version :** 1.0 - Novembre 2025  
**PostgreSQL :** Version 18  
**Auteur :** Formation PostgreSQL pour Développeurs et DevOps  

---

## Fin de l'Annexe G.2

Vous disposez maintenant de tous les outils pour mettre en place un système de backup automatisé robuste, fiable et professionnel pour PostgreSQL ! 🚀

**N'oubliez jamais : La meilleure sauvegarde est celle que vous avez testée !** ✅

⏭️ [Scripts de monitoring système](/annexes/commandes-shell-scripts/03-scripts-monitoring-systeme.md)
