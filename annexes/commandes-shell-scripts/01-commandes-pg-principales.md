🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe G : Commandes Shell et Scripts Utiles
## PostgreSQL 18 - Outils en Ligne de Commande

---

## Introduction

PostgreSQL fournit un ensemble d'outils en ligne de commande (CLI - Command Line Interface) qui permettent de gérer, sauvegarder et restaurer vos bases de données sans passer par une interface graphique. Ces outils sont essentiels pour l'automatisation, les scripts de production et l'administration quotidienne.

Dans cette annexe, nous allons explorer trois outils fondamentaux :

1. **pg_ctl** : Contrôle du serveur PostgreSQL (démarrage, arrêt, redémarrage)
2. **pg_dump** : Sauvegarde logique de bases de données
3. **pg_restore** : Restauration de sauvegardes

Ces outils sont installés automatiquement avec PostgreSQL et sont généralement situés dans le répertoire `bin` de votre installation PostgreSQL.

---

## 1. pg_ctl : Gestion du Serveur PostgreSQL

### 1.1. Qu'est-ce que pg_ctl ?

**pg_ctl** (PostgreSQL Control) est l'utilitaire qui permet de **contrôler le serveur PostgreSQL**. C'est l'équivalent d'un panneau de contrôle pour votre instance de base de données : vous pouvez démarrer, arrêter, redémarrer ou obtenir le statut du serveur.

**Pourquoi est-ce important ?**
- En production, vous devez pouvoir démarrer/arrêter PostgreSQL de manière contrôlée
- Lors de changements de configuration, un redémarrage est souvent nécessaire
- Pour le troubleshooting, il faut pouvoir vérifier l'état du serveur

### 1.2. Syntaxe de Base

```bash
pg_ctl [OPTIONS] ACTION
```

**Où :**
- `OPTIONS` : Paramètres de configuration (chemin des données, mode, timeout, etc.)
- `ACTION` : L'action à effectuer (start, stop, restart, status, etc.)

### 1.3. Actions Principales

#### 1.3.1. Démarrer le Serveur PostgreSQL

```bash
pg_ctl -D /chemin/vers/data start
```

**Explication :**
- `-D /chemin/vers/data` : Spécifie le répertoire de données PostgreSQL (souvent `/var/lib/postgresql/data` sur Linux)
- `start` : Lance le serveur PostgreSQL

**Options utiles pour le démarrage :**

```bash
# Démarrer avec un fichier de log spécifique
pg_ctl -D /var/lib/postgresql/data -l /var/log/postgresql/server.log start

# Démarrer en mode silencieux
pg_ctl -D /var/lib/postgresql/data -s start

# Démarrer en attendant la fin du démarrage
pg_ctl -D /var/lib/postgresql/data -w start
```

**Les options expliquées :**
- `-l <logfile>` : Redirige les logs du serveur vers un fichier
- `-s` : Mode silencieux (moins de sortie console)
- `-w` : Attend (wait) que le serveur soit complètement démarré avant de rendre la main

#### 1.3.2. Arrêter le Serveur PostgreSQL

```bash
pg_ctl -D /chemin/vers/data stop
```

**⚠️ Modes d'arrêt : C'est crucial de comprendre !**

PostgreSQL propose **trois modes d'arrêt**, chacun avec un niveau d'urgence différent :

**a) Smart Shutdown (Mode Intelligent - Par défaut)**

```bash
pg_ctl -D /var/lib/postgresql/data -m smart stop
```

**Comportement :**
- Refuse les **nouvelles connexions**
- Attend que **toutes les sessions actives** se terminent normalement
- Peut prendre du temps si des sessions sont toujours actives
- **Le plus sûr** : aucune perte de données

**Quand l'utiliser :**
- Maintenance programmée
- Shutdown prévu et non urgent
- Quand vous voulez que les utilisateurs terminent leur travail

**b) Fast Shutdown (Mode Rapide)**

```bash
pg_ctl -D /var/lib/postgresql/data -m fast stop
```

**Comportement :**
- Refuse les nouvelles connexions
- **Termine immédiatement** toutes les sessions actives (SIGTERM)
- Les transactions en cours sont **annulées (ROLLBACK)**
- Effectue un checkpoint et s'arrête proprement
- **Arrêt propre et rapide**

**Quand l'utiliser :**
- C'est le mode **recommandé en production** pour un arrêt standard
- Maintenance urgente
- Redémarrage après changement de configuration

**c) Immediate Shutdown (Mode Immédiat)**

```bash
pg_ctl -D /var/lib/postgresql/data -m immediate stop
```

**Comportement :**
- **Tue brutalement** tous les processus PostgreSQL (SIGQUIT)
- **Aucun checkpoint** : équivalent d'un crash
- Au redémarrage, PostgreSQL devra effectuer une **récupération des WAL** (replay)
- **Arrêt le plus rapide mais le plus violent**

**⚠️ Attention :**
- À utiliser **uniquement en cas d'urgence absolue**
- Peut prendre du temps au redémarrage (récupération)
- Ne garantit pas l'intégrité des connexions externes

**Résumé des modes d'arrêt :**

| Mode | Nouvelles connexions | Sessions actives | Transactions | Temps | Sécurité |
|------|---------------------|------------------|--------------|-------|----------|
| **smart** | ❌ Refusées | ✅ Attendre la fin | ✅ Complétées | ⏰ Long | 🛡️ Maximum |
| **fast** | ❌ Refusées | ❌ Terminées | ❌ Annulées | ⏱️ Rapide | 🛡️ Élevé |
| **immediate** | ❌ Refusées | ☠️ Tuées | ☠️ Perdues | ⚡ Immédiat | ⚠️ Moyen |

**Exemple pratique :**

```bash
# Arrêt standard pour une maintenance
pg_ctl -D /var/lib/postgresql/data -m fast stop

# Attendre que l'arrêt soit complet (timeout de 60 secondes)
pg_ctl -D /var/lib/postgresql/data -m fast -t 60 -w stop
```

#### 1.3.3. Redémarrer le Serveur

```bash
pg_ctl -D /chemin/vers/data restart
```

**Équivalent à :**
```bash
pg_ctl -D /chemin/vers/data stop
pg_ctl -D /chemin/vers/data start
```

**Option utile :**

```bash
# Redémarrage rapide avec mode fast
pg_ctl -D /var/lib/postgresql/data -m fast restart
```

**Quand redémarrer ?**
- Après modification du fichier `postgresql.conf` (pour certains paramètres)
- Après installation d'extensions systèmes
- Après un changement majeur de configuration

**💡 Astuce :** Tous les paramètres ne nécessitent pas un redémarrage ! Certains peuvent être rechargés sans interruption (voir `reload` ci-dessous).

#### 1.3.4. Recharger la Configuration (Sans Redémarrage)

```bash
pg_ctl -D /chemin/vers/data reload
```

**Comportement :**
- Relit les fichiers de configuration (`postgresql.conf`, `pg_hba.conf`)
- **Applique les changements sans interrompre le service**
- Ne fonctionne que pour les paramètres marqués comme "reloadable"

**Exemple de paramètres rechargeables :**
- `log_min_duration_statement` : Seuil de logging des requêtes lentes
- `work_mem` : Mémoire de travail pour les requêtes
- `max_connections` : ❌ **Non rechargeable** (nécessite un restart)

**Comment savoir si un paramètre est rechargeable ?**

```sql
-- Dans psql
SELECT name, context, setting
FROM pg_settings
WHERE name = 'work_mem';
```

**Les valeurs de `context` :**
- `postmaster` : Nécessite un **redémarrage complet**
- `sighup` : Rechargeable avec `reload` ✅
- `backend` : Applicable aux nouvelles connexions seulement
- `superuser` / `user` : Modifiable par session

#### 1.3.5. Vérifier le Statut du Serveur

```bash
pg_ctl -D /chemin/vers/data status
```

**Sortie possible :**

```
pg_ctl: server is running (PID: 12345)
/usr/lib/postgresql/18/bin/postgres "-D" "/var/lib/postgresql/data"
```

**Interprétation :**
- `server is running` : Le serveur est actif
- `PID: 12345` : Process ID du processus principal (postmaster)
- Si le serveur est arrêté : `pg_ctl: no server running`

**Utilisation pratique :**

```bash
# Vérifier si PostgreSQL est actif avant un script
if pg_ctl -D /var/lib/postgresql/data status > /dev/null 2>&1; then
    echo "PostgreSQL est actif"
else
    echo "PostgreSQL est arrêté"
fi
```

#### 1.3.6. Promouvoir un Standby (Réplication)

```bash
pg_ctl -D /chemin/vers/data promote
```

**Contexte :**
- Utilisé dans les architectures de **haute disponibilité**
- Transforme un serveur **standby** (réplique) en serveur **primary** (principal)
- Crucial lors d'un **failover** (bascule après panne du serveur principal)

**Exemple de scénario :**
1. Serveur Primary tombe en panne
2. L'administrateur décide de promouvoir le Standby
3. `pg_ctl promote` → Le standby devient le nouveau primary
4. Les applications se reconnectent au nouveau primary

### 1.4. Options Globales Importantes

#### 1.4.1. Répertoire de Données (-D)

```bash
pg_ctl -D /var/lib/postgresql/data [ACTION]
```

**Alternative : Variable d'environnement PGDATA**

```bash
# Dans votre .bashrc ou script
export PGDATA=/var/lib/postgresql/data

# Puis simplement :
pg_ctl start
pg_ctl stop
pg_ctl status
```

**💡 Astuce :** Définir `PGDATA` simplifie grandement vos commandes !

#### 1.4.2. Timeout (-t)

```bash
pg_ctl -D /var/lib/postgresql/data -t 120 stop
```

**Explication :**
- `-t 120` : Attendre jusqu'à 120 secondes pour que l'action se termine
- Par défaut : 60 secondes
- Si le timeout est dépassé, `pg_ctl` abandonne (mais PostgreSQL continue l'action)

**Utilisation :**
- Serveurs avec beaucoup de connexions actives
- Shutdown smart qui peut prendre du temps
- Scripts automatisés nécessitant un timeout explicite

#### 1.4.3. Attendre ou Non (-w / -W)

```bash
# Attendre la fin de l'action (Wait)
pg_ctl -D /var/lib/postgresql/data -w start

# Ne pas attendre (No Wait)
pg_ctl -D /var/lib/postgresql/data -W start
```

**Différence :**
- `-w` : La commande **attend** que PostgreSQL soit complètement démarré/arrêté
- `-W` : La commande rend la main **immédiatement** (comportement asynchrone)

**Quand utiliser `-w` ?**
- Dans des scripts où l'étape suivante dépend de PostgreSQL
- Pour être sûr que le serveur est prêt avant de continuer

**Exemple dans un script :**

```bash
#!/bin/bash
echo "Démarrage de PostgreSQL..."
pg_ctl -D /var/lib/postgresql/data -w start

if [ $? -eq 0 ]; then
    echo "PostgreSQL démarré avec succès"
    # Continuer avec d'autres commandes
else
    echo "Échec du démarrage"
    exit 1
fi
```

### 1.5. Fichiers Importants Utilisés par pg_ctl

#### 1.5.1. postmaster.pid

**Emplacement :** `$PGDATA/postmaster.pid`

**Contenu :**
- PID du processus postmaster
- Répertoire de données
- Timestamp de démarrage
- Port d'écoute
- Socket Unix

**Utilité :**
- pg_ctl lit ce fichier pour déterminer si PostgreSQL est actif
- Permet de savoir quel processus est le serveur PostgreSQL

**⚠️ Attention :**
- Si ce fichier existe alors que PostgreSQL n'est pas actif → **Problème !**
- Peut indiquer un crash ou un arrêt brutal
- Solution : Supprimer manuellement le fichier (après vérification)

#### 1.5.2. postgresql.conf

**Emplacement :** `$PGDATA/postgresql.conf`

**Rôle :**
- Fichier de configuration principal de PostgreSQL
- Contient tous les paramètres du serveur

**Rechargé par :**
- `pg_ctl reload` (pour les paramètres rechargeables)
- `pg_ctl restart` (pour tous les paramètres)

### 1.6. Exemples de Scripts Pratiques

#### 1.6.1. Script de Démarrage Sécurisé

```bash
#!/bin/bash
# start_postgresql.sh

PGDATA="/var/lib/postgresql/data"
LOGFILE="/var/log/postgresql/server.log"

echo "Vérification du statut de PostgreSQL..."
if pg_ctl -D "$PGDATA" status > /dev/null 2>&1; then
    echo "PostgreSQL est déjà actif."
    exit 0
fi

echo "Démarrage de PostgreSQL..."
pg_ctl -D "$PGDATA" -l "$LOGFILE" -w start

if [ $? -eq 0 ]; then
    echo "✅ PostgreSQL démarré avec succès"
    pg_ctl -D "$PGDATA" status
else
    echo "❌ Échec du démarrage de PostgreSQL"
    echo "Consultez le fichier de log : $LOGFILE"
    exit 1
fi
```

#### 1.6.2. Script d'Arrêt Sécurisé avec Retry

```bash
#!/bin/bash
# stop_postgresql.sh

PGDATA="/var/lib/postgresql/data"

echo "Arrêt de PostgreSQL (mode fast)..."
pg_ctl -D "$PGDATA" -m fast -t 120 -w stop

if [ $? -eq 0 ]; then
    echo "✅ PostgreSQL arrêté avec succès"
    exit 0
else
    echo "⚠️ L'arrêt fast a échoué ou expiré"
    echo "Tentative d'arrêt immediate..."
    pg_ctl -D "$PGDATA" -m immediate stop

    if [ $? -eq 0 ]; then
        echo "✅ PostgreSQL arrêté (mode immediate)"
        exit 0
    else
        echo "❌ Impossible d'arrêter PostgreSQL"
        exit 1
    fi
fi
```

#### 1.6.3. Script de Redémarrage avec Backup de Config

```bash
#!/bin/bash
# restart_postgresql.sh

PGDATA="/var/lib/postgresql/data"
BACKUP_DIR="/var/backups/postgresql/config"

# Créer le répertoire de backup si nécessaire
mkdir -p "$BACKUP_DIR"

echo "Sauvegarde de la configuration actuelle..."
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
cp "$PGDATA/postgresql.conf" "$BACKUP_DIR/postgresql.conf.$TIMESTAMP"
cp "$PGDATA/pg_hba.conf" "$BACKUP_DIR/pg_hba.conf.$TIMESTAMP"

echo "Redémarrage de PostgreSQL..."
pg_ctl -D "$PGDATA" -m fast -w restart

if [ $? -eq 0 ]; then
    echo "✅ PostgreSQL redémarré avec succès"
    pg_ctl -D "$PGDATA" status
else
    echo "❌ Échec du redémarrage"
    echo "Les backups de configuration sont disponibles dans : $BACKUP_DIR"
    exit 1
fi
```

### 1.7. Bonnes Pratiques avec pg_ctl

#### ✅ À Faire

1. **Toujours utiliser `-m fast` pour les arrêts en production**
   - Équilibre parfait entre rapidité et sécurité
   - Évite les pertes de données

2. **Définir la variable d'environnement PGDATA**
   - Simplifie les commandes
   - Évite les erreurs de chemin

3. **Utiliser des timeouts explicites dans les scripts**
   - Évite les blocages infinis
   - Permet une meilleure gestion des erreurs

4. **Logger les actions dans un fichier**
   - Facilite le troubleshooting
   - Conserve un historique des actions

5. **Vérifier le statut avant toute action**
   - Évite les tentatives de démarrage d'un serveur déjà actif
   - Prévient les erreurs dans les scripts

#### ❌ À Éviter

1. **Ne jamais utiliser `-m immediate` en production normale**
   - Uniquement pour les urgences absolues
   - Peut entraîner une récupération longue

2. **Ne pas ignorer les codes de retour**
   - Toujours vérifier `$?` dans les scripts
   - Gérer les échecs proprement

3. **Ne pas modifier manuellement `postmaster.pid`**
   - Laissez PostgreSQL gérer ce fichier
   - Suppression manuelle uniquement si le serveur est vraiment arrêté

4. **Ne pas utiliser `kill -9` sur les processus PostgreSQL**
   - Équivalent à un crash brutal
   - Utilisez toujours `pg_ctl`

### 1.8. Dépannage Courant

#### Problème : "pg_ctl: no server running"

**Cause possible :**
- Le serveur est réellement arrêté
- Le fichier `postmaster.pid` n'existe pas ou est invalide

**Solution :**
```bash
# Vérifier les processus
ps aux | grep postgres

# Si des processus existent, il y a incohérence
# Vérifier les logs
tail -f /var/log/postgresql/server.log
```

#### Problème : "pg_ctl: could not start server"

**Causes possibles :**
1. Port déjà utilisé
2. Permissions incorrectes sur le répertoire de données
3. Erreur de configuration dans `postgresql.conf`

**Solution :**
```bash
# Vérifier les logs détaillés
cat /var/log/postgresql/server.log

# Vérifier les permissions
ls -la /var/lib/postgresql/data

# Vérifier le port
netstat -tulpn | grep 5432
```

#### Problème : Timeout lors de l'arrêt

**Cause :**
- Trop de connexions actives
- Transactions longues en cours

**Solution :**
```bash
# Augmenter le timeout
pg_ctl -D /var/lib/postgresql/data -t 300 -m fast stop

# Ou forcer l'arrêt (en dernier recours)
pg_ctl -D /var/lib/postgresql/data -m immediate stop
```

---

## 2. pg_dump : Sauvegarde Logique de Bases de Données

### 2.1. Qu'est-ce que pg_dump ?

**pg_dump** est l'outil de **sauvegarde logique** de PostgreSQL. Il exporte le contenu d'une base de données sous forme de commandes SQL ou dans un format personnalisé compressé.

**Différence avec une sauvegarde physique :**
- **Logique** (pg_dump) : Exporte les données et la structure en SQL → Portable, lisible
- **Physique** (pg_basebackup) : Copie les fichiers binaires du disque → Plus rapide, mais moins flexible

**Quand utiliser pg_dump ?**
- Sauvegardes régulières de bases de données
- Migration vers une autre version de PostgreSQL
- Copie d'une base de production vers un environnement de développement
- Backup avant une opération risquée (migration de schéma, mise à jour majeure)
- Export de données pour archivage ou partage

### 2.2. Syntaxe de Base

```bash
pg_dump [OPTIONS] [DBNAME]
```

**Exemple le plus simple :**

```bash
pg_dump mabase > backup.sql
```

**Ce que cela fait :**
- Se connecte à la base `mabase`
- Génère toutes les commandes SQL nécessaires pour recréer la base
- Redirige la sortie vers le fichier `backup.sql`

### 2.3. Formats de Sortie

pg_dump propose **quatre formats** différents :

#### 2.3.1. Format Plain Text (SQL)

```bash
pg_dump -F p mabase > backup.sql
# ou simplement (p = plain est le défaut)
pg_dump mabase > backup.sql
```

**Caractéristiques :**
- Fichier texte lisible contenant des commandes SQL
- **Avantages :**
  - Lisible et éditable
  - Peut être restauré avec `psql`
  - Peut être versionné avec Git
- **Inconvénients :**
  - Non compressé → Fichiers volumineux
  - Restauration séquentielle uniquement
  - Pas de restauration sélective

**Contenu typique d'un backup SQL :**

```sql
--
-- PostgreSQL database dump
--

SET statement_timeout = 0;
SET lock_timeout = 0;
SET client_encoding = 'UTF8';

CREATE TABLE utilisateurs (
    id integer NOT NULL,
    nom character varying(100),
    email character varying(255)
);

COPY utilisateurs (id, nom, email) FROM stdin;
1	Alice	alice@example.com
2	Bob	bob@example.com
\.

ALTER TABLE ONLY utilisateurs
    ADD CONSTRAINT utilisateurs_pkey PRIMARY KEY (id);
```

**Restauration :**

```bash
psql mabase < backup.sql
```

#### 2.3.2. Format Custom (Recommandé)

```bash
pg_dump -F c mabase > backup.dump
# ou avec extension explicite
pg_dump -F c mabase -f backup.dump
```

**Caractéristiques :**
- Format binaire compressé propriétaire à PostgreSQL
- **Avantages :** 🌟
  - **Compressé automatiquement** → Gain d'espace considérable
  - **Restauration sélective** → Restaurer uniquement certaines tables
  - **Restauration parallèle** → Utilisation de plusieurs processus
  - **Plus flexible** pour la restauration avec pg_restore
- **Inconvénients :**
  - Non lisible (binaire)
  - Nécessite pg_restore pour restaurer

**💡 Recommandation :** C'est le format **le plus utilisé en production** !

#### 2.3.3. Format Directory

```bash
pg_dump -F d mabase -f backup_dir/
```

**Caractéristiques :**
- Crée un **répertoire** contenant plusieurs fichiers
- Chaque table/objet est dans un fichier séparé
- **Avantages :**
  - Restauration parallèle automatique
  - Idéal pour très grandes bases de données
  - Permet une compression efficace
- **Inconvénients :**
  - Plusieurs fichiers à gérer
  - Nécessite pg_restore

#### 2.3.4. Format Tar

```bash
pg_dump -F t mabase > backup.tar
```

**Caractéristiques :**
- Archive tar contenant les données
- Similaire au format custom mais en tar standard
- **Avantages :**
  - Archive unique
  - Compatible avec les outils tar standard
- **Inconvénients :**
  - Pas de compression intégrée (mais peut être compressé avec gzip)
  - Moins flexible que le format custom

**Tableau récapitulatif :**

| Format | Option | Extension | Lisible | Compressé | Restauration parallèle | Recommandé pour |
|--------|--------|-----------|---------|-----------|----------------------|-----------------|
| **Plain** | `-F p` | `.sql` | ✅ Oui | ❌ Non | ❌ Non | Dev, Versioning |
| **Custom** | `-F c` | `.dump` | ❌ Non | ✅ Oui | ✅ Oui | **Production** 🌟 |
| **Directory** | `-F d` | `<dir>/` | ❌ Non | ✅ Oui | ✅ Oui | Grandes bases |
| **Tar** | `-F t` | `.tar` | ❌ Non | ❌ Non* | ❌ Non | Archivage |

*Peut être compressé manuellement avec gzip/bzip2

### 2.4. Options Essentielles

#### 2.4.1. Connexion à la Base de Données

```bash
# Spécifier la base de données
pg_dump -d mabase > backup.sql

# Spécifier l'hôte
pg_dump -h localhost -d mabase > backup.sql

# Spécifier le port
pg_dump -p 5432 -d mabase > backup.sql

# Spécifier l'utilisateur
pg_dump -U postgres -d mabase > backup.sql

# Tout ensemble
pg_dump -h db.example.com -p 5432 -U admin -d production > backup.sql
```

**💡 Astuce :** Vous pouvez aussi utiliser les variables d'environnement :

```bash
export PGHOST=localhost
export PGPORT=5432
export PGUSER=postgres
export PGDATABASE=mabase

# Puis simplement
pg_dump > backup.sql
```

#### 2.4.2. Authentification

**Mot de passe :**

```bash
# Demander le mot de passe de manière interactive
pg_dump -U postgres -W mabase > backup.sql

# Ou utiliser un fichier .pgpass pour automatisation
# ~/.pgpass : hostname:port:database:username:password
# Exemple : localhost:5432:mabase:postgres:monmotdepasse
# Permissions : chmod 600 ~/.pgpass

pg_dump -U postgres mabase > backup.sql  # Pas de prompt
```

**Fichier .pgpass (pour automatisation) :**

```bash
# Créer le fichier
cat > ~/.pgpass << EOF
localhost:5432:*:postgres:motdepasse_postgres
db.example.com:5432:production:admin:motdepasse_admin
EOF

# Sécuriser les permissions (OBLIGATOIRE)
chmod 600 ~/.pgpass

# Maintenant les backups automatiques fonctionnent sans interaction
pg_dump -U postgres mabase > backup.sql
```

#### 2.4.3. Compression

```bash
# Format custom : compression automatique
pg_dump -F c -Z 9 mabase -f backup.dump

# -Z 9 : Niveau de compression maximum (0-9)
# 0 = pas de compression
# 9 = compression maximum (plus lent, plus petit)
# Défaut : 6
```

**Compression manuelle avec plain text :**

```bash
# Avec gzip
pg_dump mabase | gzip > backup.sql.gz

# Avec bzip2 (meilleure compression, plus lent)
pg_dump mabase | bzip2 > backup.sql.bz2

# Avec xz (excellente compression)
pg_dump mabase | xz > backup.sql.xz
```

**Comparaison de taille typique (pour une base de 1 GB) :**

```
backup.sql           : 1000 MB  (100%)
backup.sql.gz        :  250 MB  (25%)
backup.dump (-Z 6)   :  220 MB  (22%)
backup.dump (-Z 9)   :  200 MB  (20%)
backup.sql.bz2       :  180 MB  (18%)
backup.sql.xz        :  150 MB  (15%)
```

#### 2.4.4. Backup Sélectif

**Sauvegarder uniquement certaines tables :**

```bash
# Une seule table
pg_dump -t utilisateurs mabase > backup_utilisateurs.sql

# Plusieurs tables
pg_dump -t utilisateurs -t commandes -t produits mabase > backup_tables.sql

# Avec pattern (wildcard)
pg_dump -t 'log_*' mabase > backup_logs.sql
# Sauvegarde toutes les tables commençant par "log_"
```

**Exclure certaines tables :**

```bash
# Exclure une table
pg_dump -T logs mabase > backup_sans_logs.sql

# Exclure plusieurs tables
pg_dump -T logs -T temp_data -T cache mabase > backup.sql

# Exclure avec pattern
pg_dump -T 'temp_*' mabase > backup.sql
```

**Sauvegarder uniquement un schéma :**

```bash
# Un seul schéma
pg_dump -n public mabase > backup_public.sql

# Plusieurs schémas
pg_dump -n public -n sales -n hr mabase > backup_schemas.sql

# Exclure un schéma
pg_dump -N audit mabase > backup_sans_audit.sql
```

#### 2.4.5. Options de Contenu

**Données uniquement (sans structure) :**

```bash
pg_dump --data-only mabase > data.sql
# ou
pg_dump -a mabase > data.sql
```

**Utilité :**
- Restaurer les données dans une base déjà créée
- Migration de données entre schémas identiques
- Rafraîchir les données de développement

**Structure uniquement (sans données) :**

```bash
pg_dump --schema-only mabase > schema.sql
# ou
pg_dump -s mabase > schema.sql
```

**Utilité :**
- Créer une copie de la structure pour un nouvel environnement
- Versioning du schéma dans Git
- Documentation de la base

**Exemple de workflow :**

```bash
# 1. Backup de la structure en Git pour versioning
pg_dump --schema-only mabase > schema.sql
git add schema.sql
git commit -m "Update database schema"

# 2. Backup des données séparément (non versionné, plus gros)
pg_dump --data-only -F c mabase -f data.dump
```

### 2.5. Sauvegarde de Toute une Instance

**pg_dumpall** : Sauvegarder toutes les bases de données :

```bash
pg_dumpall > cluster_backup.sql
```

**Contenu sauvegardé :**
- **Toutes les bases de données** de l'instance
- **Rôles et utilisateurs** (CREATE ROLE)
- **Tablespaces**
- **Attributs globaux** (configurations au niveau cluster)

**⚠️ Important :** pg_dumpall produit uniquement du SQL plain text (pas de format custom).

**Options utiles :**

```bash
# Sauvegarder uniquement les rôles
pg_dumpall --roles-only > roles.sql

# Sauvegarder uniquement les tablespaces
pg_dumpall --tablespaces-only > tablespaces.sql

# Sauvegarder les définitions globales uniquement
pg_dumpall --globals-only > globals.sql

# Puis sauvegarder chaque base séparément en format custom
pg_dump -F c base1 -f base1.dump
pg_dump -F c base2 -f base2.dump
```

**Stratégie recommandée pour backup complet :**

```bash
#!/bin/bash
# Backup complet d'une instance PostgreSQL

BACKUP_DIR="/var/backups/postgresql/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Sauvegarde des rôles et objets globaux..."
pg_dumpall --globals-only > "$BACKUP_DIR/globals.sql"

echo "Sauvegarde des bases de données..."
for DB in $(psql -t -c "SELECT datname FROM pg_database WHERE NOT datistemplate AND datname != 'postgres'")
do
    echo "  - Backup de $DB"
    pg_dump -F c "$DB" -f "$BACKUP_DIR/${DB}.dump"
done

echo "✅ Backup terminé dans : $BACKUP_DIR"
```

### 2.6. Backup Parallèle (Grande Performance)

Pour les **très grandes bases de données**, pg_dump peut utiliser plusieurs processus en parallèle (uniquement avec format directory) :

```bash
# Backup avec 4 processus parallèles
pg_dump -F d -j 4 grande_base -f backup_dir/
```

**Explication :**
- `-j 4` : Utilise 4 processus (jobs) en parallèle
- Divise le travail entre les processus
- Peut réduire le temps de backup de 70-80% !

**💡 Recommandation :**
- Nombre de jobs = Nombre de cœurs CPU disponibles
- Typiquement entre 4 et 8 pour des performances optimales
- Au-delà de 8, les gains diminuent

**Exemple de gain :**

```
Base de 100 GB :
- pg_dump standard    : 2 heures
- pg_dump -j 4        : 30 minutes (4× plus rapide)
- pg_dump -j 8        : 20 minutes (6× plus rapide)
```

### 2.7. Exemples de Scripts de Backup

#### 2.7.1. Script de Backup Simple avec Rotation

```bash
#!/bin/bash
# backup_simple.sh

DATABASE="mabase"
BACKUP_DIR="/var/backups/postgresql"
RETENTION_DAYS=7

# Créer le nom de fichier avec timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump"

echo "Début du backup de $DATABASE..."

# Effectuer le backup
pg_dump -F c -Z 9 "$DATABASE" -f "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "✅ Backup réussi : $BACKUP_FILE"

    # Calculer la taille
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    echo "Taille : $SIZE"

    # Nettoyer les anciens backups (plus de RETENTION_DAYS jours)
    echo "Nettoyage des anciens backups (> $RETENTION_DAYS jours)..."
    find "$BACKUP_DIR" -name "${DATABASE}_*.dump" -mtime +$RETENTION_DAYS -delete

    echo "✅ Rotation des backups terminée"
else
    echo "❌ Échec du backup"
    exit 1
fi
```

#### 2.7.2. Script de Backup Avancé avec Logging

```bash
#!/bin/bash
# backup_avance.sh

# Configuration
DATABASE="production"
BACKUP_DIR="/var/backups/postgresql"
LOG_DIR="/var/log/postgresql_backup"
RETENTION_DAYS=30
ALERT_EMAIL="admin@example.com"

# Créer les répertoires si nécessaires
mkdir -p "$BACKUP_DIR" "$LOG_DIR"

# Noms de fichiers
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DATABASE}_${TIMESTAMP}.dump"
LOG_FILE="$LOG_DIR/backup_${TIMESTAMP}.log"

# Fonction de log
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Fonction d'alerte
send_alert() {
    local subject="$1"
    local message="$2"
    echo "$message" | mail -s "$subject" "$ALERT_EMAIL"
}

# Début du backup
log "=== Début du backup de $DATABASE ==="

# Vérifier la connectivité
if ! pg_isready -d "$DATABASE" > /dev/null 2>&1; then
    log "❌ ERREUR : Impossible de se connecter à PostgreSQL"
    send_alert "Échec Backup PostgreSQL" "La connexion à PostgreSQL a échoué"
    exit 1
fi

# Taille de la base avant backup
DB_SIZE=$(psql -t -c "SELECT pg_size_pretty(pg_database_size('$DATABASE'))")
log "Taille de la base : $DB_SIZE"

# Effectuer le backup avec temps d'exécution
START_TIME=$(date +%s)
pg_dump -F c -Z 9 "$DATABASE" -f "$BACKUP_FILE" 2>> "$LOG_FILE"
BACKUP_STATUS=$?
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

if [ $BACKUP_STATUS -eq 0 ]; then
    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "✅ Backup réussi en ${DURATION}s"
    log "Fichier : $BACKUP_FILE"
    log "Taille : $BACKUP_SIZE"

    # Vérifier l'intégrité du backup
    log "Vérification de l'intégrité..."
    if pg_restore --list "$BACKUP_FILE" > /dev/null 2>&1; then
        log "✅ Backup valide"
    else
        log "⚠️ ATTENTION : Le backup semble corrompu !"
        send_alert "Backup PostgreSQL Corrompu" "Le backup $BACKUP_FILE semble invalide"
    fi

    # Rotation des backups
    log "Nettoyage des anciens backups..."
    DELETED_COUNT=$(find "$BACKUP_DIR" -name "${DATABASE}_*.dump" -mtime +$RETENTION_DAYS -delete -print | wc -l)
    log "Supprimés : $DELETED_COUNT ancien(s) backup(s)"

    # Rotation des logs
    find "$LOG_DIR" -name "backup_*.log" -mtime +$RETENTION_DAYS -delete

    log "=== Backup terminé avec succès ==="
else
    log "❌ ÉCHEC du backup (code: $BACKUP_STATUS)"
    send_alert "ÉCHEC Backup PostgreSQL" "Le backup de $DATABASE a échoué. Consultez $LOG_FILE"
    exit 1
fi
```

#### 2.7.3. Script de Backup Multi-Bases avec Parallélisme

```bash
#!/bin/bash
# backup_multi_bases.sh

BACKUP_DIR="/var/backups/postgresql/$(date +%Y%m%d_%H%M%S)"
PARALLEL_JOBS=4
RETENTION_DAYS=14

mkdir -p "$BACKUP_DIR"

echo "=== Backup de toutes les bases PostgreSQL ==="
echo "Répertoire : $BACKUP_DIR"
echo "Jobs parallèles : $PARALLEL_JOBS"

# Sauvegarder les objets globaux
echo "Sauvegarde des rôles et objets globaux..."
pg_dumpall --globals-only > "$BACKUP_DIR/globals.sql"

# Obtenir la liste des bases (hors templates et postgres)
DATABASES=$(psql -t -c "SELECT datname FROM pg_database WHERE NOT datistemplate AND datname != 'postgres'")

# Fonction de backup
backup_database() {
    local db="$1"
    local output="$BACKUP_DIR/${db}.dump"
    echo "  Backup de $db..."
    pg_dump -F c -Z 9 "$db" -f "$output"
    if [ $? -eq 0 ]; then
        size=$(du -h "$output" | cut -f1)
        echo "    ✅ $db : $size"
    else
        echo "    ❌ $db : ÉCHEC"
    fi
}

export -f backup_database
export BACKUP_DIR

# Exécuter les backups en parallèle avec GNU parallel ou xargs
if command -v parallel > /dev/null 2>&1; then
    # Avec GNU parallel (recommandé)
    echo "$DATABASES" | parallel -j "$PARALLEL_JOBS" backup_database {}
else
    # Fallback avec xargs
    echo "$DATABASES" | xargs -P "$PARALLEL_JOBS" -I {} bash -c "backup_database '{}'"
fi

# Créer une archive tar compressée du tout
echo "Création de l'archive finale..."
tar -czf "$BACKUP_DIR.tar.gz" -C "$(dirname "$BACKUP_DIR")" "$(basename "$BACKUP_DIR")"
echo "✅ Archive : $BACKUP_DIR.tar.gz"

# Nettoyage
rm -rf "$BACKUP_DIR"

# Rotation
echo "Rotation des anciens backups..."
find "$(dirname "$BACKUP_DIR")" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "=== Backup terminé ==="
```

### 2.8. Bonnes Pratiques pour pg_dump

#### ✅ À Faire

1. **Toujours utiliser le format custom (-F c) en production**
   - Compression automatique
   - Flexibilité maximale pour la restauration

2. **Automatiser les backups avec cron**
   ```bash
   # Exemple dans crontab : backup tous les jours à 2h du matin
   0 2 * * * /usr/local/bin/backup_postgresql.sh
   ```

3. **Tester régulièrement la restauration**
   - Un backup non testé = pas de backup
   - Restaurer sur une base de test mensuelle

4. **Mettre en place une rotation**
   - Conserver les backups sur 7-30 jours selon votre politique
   - Ne pas remplir le disque !

5. **Vérifier l'intégrité des backups**
   ```bash
   pg_restore --list backup.dump > /dev/null
   # Si retour = 0 → backup valide
   ```

6. **Stocker les backups hors du serveur de production**
   - NAS, cloud (S3, Azure Blob), serveur de backup distant
   - Principe 3-2-1 : 3 copies, 2 supports différents, 1 hors site

7. **Monitorer la taille et la durée des backups**
   - Détecter les anomalies (base qui grossit trop vite)
   - Ajuster la stratégie si nécessaire

#### ❌ À Éviter

1. **Ne pas utiliser pg_dump sur une base en cours de migration/upgrade**
   - Peut capturer un état incohérent

2. **Ne pas oublier de sauvegarder les objets globaux (rôles)**
   - pg_dump ne sauvegarde pas les utilisateurs/rôles
   - Utilisez pg_dumpall --globals-only

3. **Ne pas dépendre uniquement de pg_dump**
   - Combiner avec des backups physiques (pg_basebackup)
   - Mettre en place une réplication

4. **Ne pas ignorer les erreurs**
   - Toujours vérifier le code de retour
   - Logger et alerter en cas d'échec

### 2.9. Dépannage pg_dump

#### Problème : "pg_dump: error: connection to server failed"

**Solution :**
```bash
# Vérifier la connectivité
pg_isready -h localhost -p 5432

# Vérifier pg_hba.conf
cat /etc/postgresql/18/main/pg_hba.conf

# Tester la connexion
psql -h localhost -U postgres -d mabase -c "SELECT 1"
```

#### Problème : "pg_dump: error: query failed: server closed the connection"

**Cause :** Timeout ou problème de mémoire côté serveur

**Solution :**
```bash
# Augmenter les timeouts
export PGCONNECT_TIMEOUT=300
pg_dump mabase > backup.sql

# Ou diviser le backup en tables
pg_dump -t grande_table mabase > grande_table.sql
pg_dump -T grande_table mabase > rest.sql
```

#### Problème : Backup trop lent

**Solutions :**
1. Utiliser le format directory avec parallélisme
   ```bash
   pg_dump -F d -j 4 mabase -f backup_dir/
   ```

2. Augmenter la compression (si CPU disponible)
   ```bash
   pg_dump -F c -Z 1 mabase -f backup.dump  # Moins de compression, plus rapide
   ```

3. Exclure les tables volumineuses non critiques
   ```bash
   pg_dump -T logs -T temp_data mabase -f backup.dump
   ```

---

## 3. pg_restore : Restauration de Sauvegardes

### 3.1. Qu'est-ce que pg_restore ?

**pg_restore** est l'outil qui permet de **restaurer des backups** créés par pg_dump aux formats **custom**, **directory** ou **tar**.

**⚠️ Important :** pg_restore ne peut PAS restaurer les fichiers SQL plain text. Pour ces derniers, utilisez `psql` :

```bash
# Pour restaurer un backup SQL plain text
psql mabase < backup.sql
```

**Pourquoi utiliser pg_restore ?**
- Restauration **sélective** (seulement certaines tables)
- Restauration **parallèle** (beaucoup plus rapide)
- **Flexibilité** : réordonner les objets, gérer les erreurs
- **Options avancées** : restauration incrémentale, skip d'objets

### 3.2. Syntaxe de Base

```bash
pg_restore [OPTIONS] BACKUP_FILE
```

**Exemple le plus simple :**

```bash
pg_restore -d mabase backup.dump
```

**Ce que cela fait :**
- Lit le fichier `backup.dump` (format custom)
- Restaure tout le contenu dans la base `mabase`

### 3.3. Modes de Restauration

#### 3.3.1. Restauration vers une Base Existante

```bash
# Restaurer dans une base déjà créée
createdb nouvelle_base
pg_restore -d nouvelle_base backup.dump
```

**⚠️ Attention :**
- La base de destination doit exister
- Les objets existants peuvent causer des conflits

#### 3.3.2. Restauration avec Création de Base (-C)

```bash
# Restaurer en créant la base automatiquement
pg_restore -C -d postgres backup.dump
```

**Explication :**
- `-C` : Crée la base de données avant de restaurer
- `-d postgres` : Se connecte d'abord à la base `postgres` (base système)
- La base est créée avec le même nom que dans le backup

**💡 Astuce :** C'est le mode le plus simple pour une restauration complète !

#### 3.3.3. Restauration avec Clean (-c)

```bash
# Supprimer les objets existants avant restauration
pg_restore -c -d mabase backup.dump
```

**Comportement :**
- Exécute des `DROP TABLE`, `DROP INDEX`, etc. avant les CREATE
- Permet de "nettoyer" une base avant restauration
- ⚠️ Peut causer des erreurs si les objets n'existent pas

**Combinaison puissante :**

```bash
# Créer la base ET nettoyer (si elle existe déjà)
pg_restore -C -c -d postgres backup.dump
```

### 3.4. Restauration Sélective

#### 3.4.1. Lister le Contenu d'un Backup

**Avant de restaurer, vous pouvez inspecter le contenu :**

```bash
pg_restore --list backup.dump
```

**Sortie typique :**

```
;
; Archive created at 2025-11-21 10:30:00 UTC
;     dbname: mabase
;     TOC Entries: 245
;     Compression: 9
;     Dump Version: 1.15-0
;     Format: CUSTOM
;

3; 2615 16385 SCHEMA - public postgres
4; 0 0 COMMENT - SCHEMA public postgres
5; 3079 16386 EXTENSION - plpgsql
205; 1259 16387 TABLE public utilisateurs postgres
206; 1259 16388 SEQUENCE public utilisateurs_id_seq postgres
207; 0 16387 SEQUENCE OWNED BY public utilisateurs_id_seq postgres
...
```

**Éléments importants :**
- **TOC ID** (première colonne) : Identifiant unique de chaque objet
- **Type d'objet** : TABLE, INDEX, CONSTRAINT, etc.
- **Nom de l'objet**
- **Propriétaire**

#### 3.4.2. Restaurer Uniquement Certaines Tables

```bash
# Restaurer une seule table
pg_restore -d mabase -t utilisateurs backup.dump

# Restaurer plusieurs tables
pg_restore -d mabase -t utilisateurs -t commandes backup.dump
```

#### 3.4.3. Restaurer Uniquement un Schéma

```bash
# Restaurer uniquement le schéma "sales"
pg_restore -d mabase -n sales backup.dump
```

#### 3.4.4. Restaurer en Utilisant un Fichier de Liste

**Pour un contrôle précis sur CE qui est restauré :**

```bash
# 1. Créer un fichier de liste du backup
pg_restore --list backup.dump > backup.list

# 2. Éditer backup.list : commenter (;) les lignes à ignorer
# Exemple : commenter la table "logs"
# ;205; 1259 16389 TABLE public logs postgres

# 3. Restaurer uniquement ce qui reste dans la liste
pg_restore -d mabase -L backup.list backup.dump
```

**Cas d'usage :**
- Restaurer tout sauf certaines tables volumineuses
- Restaurer uniquement la structure (commenter les COPY)
- Restaurer dans un ordre spécifique

### 3.5. Restauration Parallèle (Performance)

**Pour les gros backups, la restauration parallèle est ESSENTIELLE :**

```bash
# Restaurer avec 4 processus parallèles
pg_restore -d mabase -j 4 backup.dump
```

**Explication :**
- `-j 4` : Utilise 4 processus (jobs) simultanés
- Divise les tables/index entre les processus
- Peut réduire le temps de restauration de 70-80% !

**⚠️ Limitation :**
- Fonctionne uniquement avec les formats **custom** et **directory**
- Nécessite plusieurs objets à restaurer (inefficace pour une seule table)

**💡 Recommandation :**
- Nombre de jobs = Nombre de cœurs CPU (typiquement 4-8)
- Très efficace pour restaurer index et contraintes

**Exemple de gain :**

```
Restauration d'un backup de 100 GB :
- pg_restore standard  : 3 heures
- pg_restore -j 4      : 45 minutes (4× plus rapide)
- pg_restore -j 8      : 30 minutes (6× plus rapide)
```

### 3.6. Options de Contenu

#### 3.6.1. Restaurer Uniquement la Structure (--schema-only)

```bash
pg_restore -d mabase --schema-only backup.dump
```

**Restaure :**
- Tables (structure)
- Index
- Contraintes
- Séquences
- Fonctions, Triggers, etc.

**N'importe PAS les données.**

**Cas d'usage :**
- Créer un environnement de développement vide
- Cloner la structure vers une nouvelle base
- Tester des migrations de schéma

#### 3.6.2. Restaurer Uniquement les Données (--data-only)

```bash
pg_restore -d mabase --data-only backup.dump
```

**Restaure uniquement les données**, pas la structure.

**⚠️ Prérequis :** La structure doit déjà exister dans la base de destination.

**Cas d'usage :**
- Rafraîchir les données d'une base de développement
- Importer des données dans un schéma existant
- Migration de données entre schémas identiques

#### 3.6.3. Restaurer Uniquement les Index et Contraintes

```bash
# Restaurer tout sauf les index
pg_restore -d mabase --section=pre-data --section=data backup.dump

# Puis restaurer les index en parallèle
pg_restore -d mabase --section=post-data -j 8 backup.dump
```

**Explication des sections :**
- **pre-data** : Structure des tables, séquences, types
- **data** : Données (COPY)
- **post-data** : Index, contraintes, triggers

**💡 Stratégie optimale :**
1. Restaurer pre-data + data (rapide, sans index)
2. Restaurer post-data en parallèle (créer les index en //)

**Exemple complet :**

```bash
# Étape 1 : Restaurer structure + données
echo "Restauration structure et données..."
pg_restore -d mabase --section=pre-data --section=data backup.dump

# Étape 2 : Restaurer index en parallèle (très rapide !)
echo "Création des index en parallèle..."
pg_restore -d mabase --section=post-data -j 8 backup.dump

echo "✅ Restauration terminée"
```

### 3.7. Gestion des Erreurs

#### 3.7.1. Continuer en Cas d'Erreur (--single-transaction OFF)

Par défaut, pg_restore s'arrête à la première erreur. Pour continuer :

```bash
# Ne pas s'arrêter aux erreurs
pg_restore -d mabase --no-owner --no-acl backup.dump
```

**Options utiles :**
- `--no-owner` : Ignore les propriétaires d'objets (évite les erreurs si l'utilisateur n'existe pas)
- `--no-acl` : Ignore les permissions (GRANT/REVOKE)
- `--if-exists` : Ajoute IF EXISTS aux DROP (évite erreurs si objet n'existe pas)

#### 3.7.2. Mode Transaction Unique

```bash
# Restaurer en une seule transaction (tout ou rien)
pg_restore -d mabase --single-transaction backup.dump
```

**Comportement :**
- Toute la restauration = 1 transaction
- Si une erreur survient → **ROLLBACK complet**
- Base laissée dans état initial

**⚠️ Attention :**
- Impossible avec `-j` (parallèle)
- Consomme beaucoup de mémoire pour gros backups
- Utile pour s'assurer de la cohérence

### 3.8. Options de Connexion

**Identiques à pg_dump :**

```bash
# Spécifier hôte, port, utilisateur
pg_restore -h db.example.com -p 5432 -U admin -d mabase backup.dump

# Avec mot de passe interactif
pg_restore -U admin -W -d mabase backup.dump

# En utilisant .pgpass pour automatisation
pg_restore -U admin -d mabase backup.dump
```

### 3.9. Exemples de Scripts de Restauration

#### 3.9.1. Script de Restauration Simple

```bash
#!/bin/bash
# restore_simple.sh

BACKUP_FILE="$1"
TARGET_DB="$2"

if [ -z "$BACKUP_FILE" ] || [ -z "$TARGET_DB" ]; then
    echo "Usage: $0 <backup_file> <target_database>"
    exit 1
fi

# Vérifier que le backup existe
if [ ! -f "$BACKUP_FILE" ]; then
    echo "❌ Erreur : Fichier de backup introuvable : $BACKUP_FILE"
    exit 1
fi

echo "Restauration de $BACKUP_FILE vers $TARGET_DB..."

# Vérifier si la base existe
if psql -lqt | cut -d \| -f 1 | grep -qw "$TARGET_DB"; then
    echo "⚠️  La base $TARGET_DB existe déjà"
    read -p "Voulez-vous la supprimer et la recréer ? (o/N) " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Oo]$ ]]; then
        dropdb "$TARGET_DB"
        createdb "$TARGET_DB"
    else
        echo "Restauration dans la base existante..."
    fi
else
    echo "Création de la base $TARGET_DB..."
    createdb "$TARGET_DB"
fi

# Restaurer avec parallélisme
echo "Restauration en cours..."
pg_restore -d "$TARGET_DB" -j 4 --no-owner --no-acl "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "✅ Restauration réussie !"

    # Afficher des informations sur la base restaurée
    echo ""
    echo "Statistiques de la base restaurée :"
    psql -d "$TARGET_DB" -c "
        SELECT
            schemaname,
            COUNT(*) as nb_tables
        FROM pg_tables
        WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
        GROUP BY schemaname;
    "
else
    echo "❌ Échec de la restauration"
    exit 1
fi
```

#### 3.9.2. Script de Restauration Avancé avec Optimisation

```bash
#!/bin/bash
# restore_optimized.sh

BACKUP_FILE="$1"
TARGET_DB="$2"

if [ -z "$BACKUP_FILE" ] || [ -z "$TARGET_DB" ]; then
    echo "Usage: $0 <backup_file> <target_database>"
    exit 1
fi

echo "=== Restauration Optimisée ==="
echo "Source : $BACKUP_FILE"
echo "Destination : $TARGET_DB"
echo ""

# Vérifier le backup
echo "1. Vérification du backup..."
if ! pg_restore --list "$BACKUP_FILE" > /dev/null 2>&1; then
    echo "❌ Le fichier de backup semble corrompu"
    exit 1
fi
echo "✅ Backup valide"

# Créer la base
echo ""
echo "2. Création de la base de destination..."
dropdb --if-exists "$TARGET_DB"
createdb "$TARGET_DB"

# Configuration temporaire pour optimiser la restauration
echo ""
echo "3. Configuration des paramètres d'optimisation..."
psql -d "$TARGET_DB" << EOF
-- Désactiver les triggers et contraintes temporairement
SET session_replication_role = replica;

-- Augmenter les paramètres de performance
SET maintenance_work_mem = '2GB';
SET max_parallel_workers_per_gather = 4;

-- Désactiver autovacuum pendant la restauration
ALTER TABLE pg_catalog.pg_class SET (autovacuum_enabled = false);
EOF

# Étape 1 : Restaurer structure + données (sans index)
echo ""
echo "4. Restauration de la structure et des données..."
START_TIME=$(date +%s)
pg_restore -d "$TARGET_DB" \
    --section=pre-data \
    --section=data \
    --no-owner \
    --no-acl \
    "$BACKUP_FILE"

DATA_TIME=$(($(date +%s) - START_TIME))
echo "✅ Données restaurées en ${DATA_TIME}s"

# Étape 2 : Créer les index en parallèle (le plus efficace)
echo ""
echo "5. Création des index en parallèle..."
START_TIME=$(date +%s)
pg_restore -d "$TARGET_DB" \
    --section=post-data \
    -j 8 \
    --no-owner \
    --no-acl \
    "$BACKUP_FILE"

INDEX_TIME=$(($(date +%s) - START_TIME))
echo "✅ Index créés en ${INDEX_TIME}s"

# Réactiver la configuration normale
echo ""
echo "6. Réactivation de la configuration normale..."
psql -d "$TARGET_DB" << EOF
-- Réactiver les triggers
SET session_replication_role = DEFAULT;

-- Réactiver autovacuum
ALTER TABLE pg_catalog.pg_class SET (autovacuum_enabled = true);
EOF

# ANALYZE pour mettre à jour les statistiques
echo ""
echo "7. Analyse de la base (mise à jour des statistiques)..."
psql -d "$TARGET_DB" -c "ANALYZE;"

# Résumé
TOTAL_TIME=$((DATA_TIME + INDEX_TIME))
echo ""
echo "=== Restauration Terminée ==="
echo "Temps total : ${TOTAL_TIME}s"
echo "  - Données : ${DATA_TIME}s"
echo "  - Index : ${INDEX_TIME}s"

# Afficher les informations
echo ""
echo "Informations sur la base restaurée :"
psql -d "$TARGET_DB" << EOF
SELECT
    pg_size_pretty(pg_database_size('$TARGET_DB')) as taille_totale,
    (SELECT COUNT(*) FROM pg_tables WHERE schemaname = 'public') as nb_tables,
    (SELECT COUNT(*) FROM pg_indexes WHERE schemaname = 'public') as nb_index;
EOF

echo ""
echo "✅ Restauration terminée avec succès !"
```

#### 3.9.3. Script de Restauration Sélective

```bash
#!/bin/bash
# restore_selective.sh

BACKUP_FILE="$1"
TARGET_DB="$2"

# Liste des tables à restaurer (une par ligne)
TABLES_TO_RESTORE=(
    "utilisateurs"
    "commandes"
    "produits"
)

# Liste des tables à exclure
TABLES_TO_EXCLUDE=(
    "logs"
    "temp_data"
    "cache"
)

echo "=== Restauration Sélective ==="
echo "Backup : $BACKUP_FILE"
echo "Base : $TARGET_DB"
echo ""

# Créer la base
createdb "$TARGET_DB"

# Restaurer la structure complète
echo "1. Restauration de la structure..."
pg_restore -d "$TARGET_DB" --schema-only "$BACKUP_FILE"

# Restaurer uniquement les tables sélectionnées
echo ""
echo "2. Restauration des tables sélectionnées..."
for table in "${TABLES_TO_RESTORE[@]}"; do
    echo "  - Restauration de $table..."
    pg_restore -d "$TARGET_DB" --data-only -t "$table" "$BACKUP_FILE"
done

echo ""
echo "✅ Restauration sélective terminée"
```

### 3.10. Stratégies de Restauration

#### 3.10.1. Restauration Rapide (Production)

**Objectif : Minimiser le downtime**

```bash
# 1. Restaurer dans une base temporaire
pg_restore -C -d postgres -j 8 backup.dump

# 2. Swap rapide des bases
psql -c "
    ALTER DATABASE production RENAME TO production_old;
    ALTER DATABASE production_restore RENAME TO production;
"

# 3. Nettoyage ultérieur
# dropdb production_old
```

#### 3.10.2. Restauration de Test

**Objectif : Valider un backup sans impacter la production**

```bash
# Restaurer dans une base de test
pg_restore -C -d postgres backup.dump

# Renommer pour clarifier
psql -c "ALTER DATABASE mabase RENAME TO mabase_test;"

# Valider avec des requêtes
psql -d mabase_test -c "SELECT COUNT(*) FROM utilisateurs;"
```

#### 3.10.3. Restauration Point-in-Time (Partielle)

**Si vous voulez restaurer l'état d'avant une erreur :**

```bash
# Restaurer un backup plus ancien
pg_restore -C -d postgres backup_hier.dump

# Puis appliquer les WAL jusqu'à un timestamp
# (Nécessite WAL archiving - voir chapitre Sauvegardes)
```

### 3.11. Bonnes Pratiques pour pg_restore

#### ✅ À Faire

1. **Toujours utiliser le parallélisme (-j) pour les gros backups**
   - Divise le temps par 4-8
   - `-j 4` ou `-j 8` selon vos CPU

2. **Restaurer en deux phases pour optimiser**
   ```bash
   # Phase 1 : Structure + Données
   pg_restore --section=pre-data --section=data -d mabase backup.dump

   # Phase 2 : Index en parallèle
   pg_restore --section=post-data -j 8 -d mabase backup.dump
   ```

3. **Utiliser --no-owner et --no-acl pour portabilité**
   - Évite les erreurs de permissions
   - Utile pour restaurer d'un environnement à l'autre

4. **Toujours vérifier le backup avant restauration**
   ```bash
   pg_restore --list backup.dump
   ```

5. **Tester la restauration dans un environnement de test d'abord**
   - Valider le processus
   - Chronométrer pour planifier un downtime réaliste

6. **Désactiver les triggers pendant la restauration (si possible)**
   ```sql
   SET session_replication_role = replica;  -- Désactive triggers
   -- ... restauration ...
   SET session_replication_role = DEFAULT;   -- Réactive
   ```

7. **Lancer un ANALYZE après restauration**
   ```bash
   psql -d mabase -c "ANALYZE;"
   ```

#### ❌ À Éviter

1. **Ne pas restaurer directement en production sans test**
   - Toujours tester d'abord

2. **Ne pas ignorer les erreurs sans les analyser**
   - Certaines sont critiques

3. **Ne pas oublier de restaurer les objets globaux**
   ```bash
   # Restaurer d'abord les rôles
   psql -f globals.sql

   # Puis la base
   pg_restore -d mabase backup.dump
   ```

4. **Ne pas sous-estimer le temps de restauration**
   - Un backup de 100 GB peut prendre 1-3h à restaurer
   - Planifier en conséquence

### 3.12. Dépannage pg_restore

#### Problème : "pg_restore: error: could not execute query: ERROR:  role 'xxx' does not exist"

**Cause :** L'utilisateur propriétaire des objets n'existe pas

**Solution :**
```bash
# Option 1 : Créer l'utilisateur manquant
createuser xxx

# Option 2 : Ignorer les propriétaires
pg_restore --no-owner -d mabase backup.dump
```

#### Problème : "pg_restore: error: could not execute query: ERROR:  relation 'xxx' already exists"

**Cause :** La base contient déjà des objets

**Solution :**
```bash
# Option 1 : Nettoyer avant restauration
pg_restore -c -d mabase backup.dump

# Option 2 : Restaurer dans une base vide
dropdb mabase
createdb mabase
pg_restore -d mabase backup.dump
```

#### Problème : Restauration très lente

**Cause :** Index créés de manière séquentielle

**Solution :**
```bash
# Créer les index en parallèle
pg_restore -d mabase --section=pre-data --section=data backup.dump
pg_restore -d mabase --section=post-data -j 8 backup.dump
```

#### Problème : Manque de mémoire

**Cause :** Backup trop gros, paramètres insuffisants

**Solution :**
```sql
-- Augmenter temporairement
ALTER SYSTEM SET maintenance_work_mem = '2GB';
SELECT pg_reload_conf();

-- Puis restaurer
```

---

## 4. Comparaison Récapitulative

### 4.1. Tableau de Comparaison des Outils

| Outil | Fonction | Niveau | Portée | Format Output |
|-------|----------|--------|--------|---------------|
| **pg_ctl** | Contrôle du serveur | Système | Serveur complet | N/A |
| **pg_dump** | Backup logique | Base de données | 1 base | SQL, Custom, Directory, Tar |
| **pg_dumpall** | Backup complet | Cluster | Toutes les bases + rôles | SQL uniquement |
| **pg_restore** | Restauration | Base de données | Flexible | N/A (lit Custom/Dir/Tar) |
| **psql** | Restauration SQL | Base de données | Scripts SQL | N/A (exécute SQL) |

### 4.2. Quand Utiliser Quel Outil ?

**Gestion du Serveur :**
- Démarrer/Arrêter/Redémarrer PostgreSQL → **pg_ctl**
- Recharger la configuration → **pg_ctl reload**
- Vérifier le statut → **pg_ctl status**

**Sauvegardes :**
- Backup d'une base (production) → **pg_dump -F c -j 4**
- Backup de toutes les bases → **pg_dumpall** + **pg_dump** (par base)
- Backup quotidien automatisé → **pg_dump -F c** + script cron
- Export de données pour migration → **pg_dump --data-only**
- Versioning du schéma → **pg_dump --schema-only** (SQL)

**Restaurations :**
- Restauration rapide (production) → **pg_restore -j 8**
- Restauration sélective (quelques tables) → **pg_restore -t table1 -t table2**
- Restauration depuis SQL plain text → **psql < backup.sql**
- Clone de base → **pg_dump | psql autre_base**
- Restauration de test → **pg_restore** dans base temporaire

---

## 5. Conclusion et Bonnes Pratiques Globales

### 5.1. Checklist de Production

#### Pour pg_ctl :
- [ ] Définir `PGDATA` dans l'environnement
- [ ] Utiliser `-m fast` pour les arrêts standards
- [ ] Toujours logger les démarrages/arrêts
- [ ] Vérifier le statut avant toute action
- [ ] Utiliser `-w` dans les scripts pour synchronisation

#### Pour pg_dump :
- [ ] Automatiser avec cron
- [ ] Utiliser format custom (-F c) en production
- [ ] Activer compression maximale (-Z 9)
- [ ] Paralléliser pour grandes bases (-j)
- [ ] Mettre en place rotation des backups
- [ ] Tester régulièrement la restauration
- [ ] Stocker hors site (NAS, S3, etc.)
- [ ] Monitorer taille et durée des backups
- [ ] Sauvegarder aussi les objets globaux (pg_dumpall --globals-only)

#### Pour pg_restore :
- [ ] Toujours vérifier le backup avant (--list)
- [ ] Utiliser parallélisme (-j)
- [ ] Restaurer en deux phases (data puis index)
- [ ] Utiliser --no-owner --no-acl pour portabilité
- [ ] Tester dans environnement de test d'abord
- [ ] Lancer ANALYZE après restauration
- [ ] Documenter le processus (runbook)

### 5.2. Stratégie de Backup Complète

**Backup Quotidien :**
```bash
# 02:00 - Backup de chaque base
0 2 * * * /usr/local/bin/backup_daily.sh

# Script backup_daily.sh :
# - pg_dumpall --globals-only
# - pg_dump -F c -Z 9 -j 4 pour chaque base
# - Rotation 30 jours
# - Upload vers S3/NAS
```

**Backup Hebdomadaire :**
```bash
# Dimanche 03:00 - Backup complet avec vérification
0 3 * * 0 /usr/local/bin/backup_weekly.sh

# Script backup_weekly.sh :
# - Backup complet
# - Test de restauration sur base test
# - Rapport d'intégrité
# - Archivage long terme
```

### 5.3. Récupération après Sinistre

**Scénario : Perte complète du serveur**

```bash
# 1. Installer PostgreSQL sur nouveau serveur
apt-get install postgresql-18

# 2. Restaurer les objets globaux
psql -f globals.sql

# 3. Restaurer chaque base
for dump in /backups/*.dump; do
    db=$(basename "$dump" .dump)
    pg_restore -C -d postgres -j 8 "$dump"
done

# 4. Vérifier l'intégrité
psql -l
# Tester les connexions applicatives
```

### 5.4. Automatisation Complète

**Script de monitoring et backup combiné :**

```bash
#!/bin/bash
# postgres_maintenance.sh

# 1. Vérifier que PostgreSQL est actif
if ! pg_ctl status > /dev/null 2>&1; then
    echo "❌ PostgreSQL est arrêté !"
    # Alerter
    exit 1
fi

# 2. Effectuer les backups
/usr/local/bin/backup_all.sh

# 3. Vérifier l'espace disque
USAGE=$(df -h /var/backups | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$USAGE" -gt 80 ]; then
    echo "⚠️  Espace disque critique : ${USAGE}%"
    # Nettoyer les vieux backups
    find /var/backups/postgresql -mtime +7 -delete
fi

# 4. Monitorer la taille des bases
psql -c "
    SELECT datname, pg_size_pretty(pg_database_size(datname))
    FROM pg_database
    WHERE NOT datistemplate;
"

# 5. Vérifier les connexions
CONN_COUNT=$(psql -t -c "SELECT count(*) FROM pg_stat_activity")
echo "Connexions actives : $CONN_COUNT"

echo "✅ Maintenance terminée"
```

### 5.5. Ressources Additionnelles

**Documentation officielle :**
- pg_ctl: https://www.postgresql.org/docs/18/app-pg-ctl.html
- pg_dump: https://www.postgresql.org/docs/18/app-pgdump.html
- pg_restore: https://www.postgresql.org/docs/18/app-pgrestore.html

**Outils complémentaires :**
- **pgBackRest** : Solution de backup complète (physique + logique)
- **Barman** : Backup and Recovery Manager pour PostgreSQL
- **WAL-E / WAL-G** : Archivage continu des WAL vers S3
- **pg_back** : Script de backup simplifié

---

## Fin de l'Annexe G

**Points Clés à Retenir :**

1. **pg_ctl** contrôle le cycle de vie du serveur PostgreSQL
   - `start`, `stop -m fast`, `restart`, `reload`, `status`

2. **pg_dump** crée des backups logiques
   - Format custom (-F c) recommandé en production
   - Parallélisme (-j) pour grandes bases
   - Rotation et stockage hors site essentiels

3. **pg_restore** restaure depuis formats custom/directory/tar
   - Restauration parallèle (-j) divise le temps par 4-8
   - Restauration en deux phases (data puis index) optimale
   - --no-owner --no-acl pour portabilité

4. **Automatisation** = Sécurité
   - Scripts + cron + monitoring + alertes
   - Tester régulièrement les restaurations
   - Stratégie 3-2-1 pour les backups

Ces trois outils sont **fondamentaux** pour l'administration quotidienne de PostgreSQL. Maîtrisez-les et vos bases de données seront bien protégées ! 🚀

---


⏭️ [Scripts de backup automatisés](/annexes/commandes-shell-scripts/02-scripts-backup-automatises.md)
