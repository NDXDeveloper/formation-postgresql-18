🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe G : Commandes Shell et Scripts Utiles
## Introduction à l'Administration PostgreSQL en Ligne de Commande

---

## Table des Matières

1. [Introduction](#1-introduction)
2. [Pourquoi Utiliser la Ligne de Commande ?](#2-pourquoi-utiliser-la-ligne-de-commande-)
3. [Prérequis](#3-pr%C3%A9requis)
4. [Environnement Shell](#4-environnement-shell)
5. [Organisation de cette Annexe](#5-organisation-de-cette-annexe)
6. [Conventions et Notations](#6-conventions-et-notations)
7. [Premiers Pas](#7-premiers-pas)
8. [Sécurité et Bonnes Pratiques](#8-s%C3%A9curit%C3%A9-et-bonnes-pratiques)

---

## 1. Introduction

### 1.1. Qu'est-ce que cette Annexe ?

Cette annexe est un **guide pratique et complet** des outils en ligne de commande (CLI - Command Line Interface) et des scripts shell pour administrer, surveiller et automatiser PostgreSQL.

**Ce que vous apprendrez :**
- Utiliser les outils PostgreSQL essentiels (pg_ctl, pg_dump, pg_restore, psql)
- Créer des scripts de backup automatisés et fiables
- Monitorer votre système PostgreSQL (métriques, performance, santé)
- Automatiser les tâches répétitives
- Gérer PostgreSQL en production avec confiance

**Ce que cette annexe n'est PAS :**
- ❌ Un cours sur le langage SQL (voir chapitres principaux)
- ❌ Un guide d'installation de PostgreSQL
- ❌ Un tutoriel de programmation Bash avancée (nous restons pragmatiques)

### 1.2. À qui s'adresse cette Annexe ?

**Profils ciblés :**
- **Développeurs** qui souhaitent comprendre l'administration de base
- **DevOps/SRE** qui doivent gérer PostgreSQL en production
- **Administrateurs système** découvrant PostgreSQL
- **Data Engineers** nécessitant automatisation et monitoring

**Niveau de compétence :**
- Débutant : Vous découvrez la ligne de commande Linux
- Intermédiaire : Vous êtes à l'aise avec le terminal
- Avancé : Vous voulez optimiser et automatiser

**⚠️ Note :** Même si vous êtes débutant, ne vous inquiétez pas ! Chaque concept sera expliqué clairement avec des exemples concrets.

### 1.3. Philosophie de cette Annexe

**"Apprendre par la pratique, comprendre par l'explication"**

Cette annexe suit une approche **progressive et pédagogique** :

1. **Comprendre** : Pourquoi cette commande/script existe
2. **Observer** : Voir des exemples concrets
3. **Expliquer** : Décortiquer chaque élément
4. **Appliquer** : Scripts prêts à l'emploi
5. **Adapter** : Personnaliser pour vos besoins

**Exemple de progression :**
```
Script minimal (5 lignes)
    ↓
Script simple avec vérifications (20 lignes)
    ↓
Script robuste avec logs (50 lignes)
    ↓
Script production-ready (150+ lignes)
```

---

## 2. Pourquoi Utiliser la Ligne de Commande ?

### 2.1. Les Limites des Interfaces Graphiques

**Interfaces graphiques (pgAdmin, DBeaver, etc.) :**
- ✅ Faciles pour débuter
- ✅ Visuelles et intuitives
- ✅ Bonnes pour l'exploration

**Mais :**
- ❌ Difficiles à automatiser
- ❌ Lentes pour tâches répétitives
- ❌ Non scriptables
- ❌ Dépendent d'un environnement graphique
- ❌ Consomment plus de ressources

**Ligne de commande :**
- ✅ Automatisation complète
- ✅ Scripts réutilisables
- ✅ Rapide et efficace
- ✅ Fonctionne en SSH distant
- ✅ Essentielle en production
- ✅ Facilite l'intégration CI/CD

### 2.2. Cas d'Usage Concrets

**Scénario 1 : Backup Quotidien**

❌ **Avec interface graphique :**
```
1. Ouvrir pgAdmin
2. Clic droit → Backup
3. Choisir le format
4. Sélectionner le chemin
5. Clic "Backup"
6. Attendre la fin
7. Répéter tous les jours... 😫
```

✅ **Avec ligne de commande :**
```bash
# backup.sh
pg_dump -F c mabase -f backup_$(date +%Y%m%d).dump

# Dans crontab : exécution automatique à 2h
0 2 * * * /usr/local/bin/backup.sh
```

**Résultat :** Backup automatique tous les jours, sans intervention humaine !

**Scénario 2 : Monitoring de 10 Serveurs**

❌ **Avec interface graphique :**
```
Se connecter à chaque serveur
Ouvrir pgAdmin
Vérifier manuellement
Prendre des notes
Répéter pour les 9 autres serveurs... 😫
Temps estimé : 2 heures
```

✅ **Avec ligne de commande :**
```bash
# monitor_all.sh
for server in server{1..10}; do
    ssh $server "psql -c 'SELECT COUNT(*) FROM pg_stat_activity;'"
done

# Temps estimé : 10 secondes
```

**Scénario 3 : Restauration d'Urgence à 3h du Matin**

❌ **Avec interface graphique :**
```
Réveillé en panique
Chercher le laptop
Lancer l'interface graphique
Chercher le bon bouton...
Stress maximum ! 😱
```

✅ **Avec ligne de commande :**
```bash
# Depuis smartphone via SSH
ssh production
./restore_latest.sh
# 30 secondes, retour au lit 😴
```

### 2.3. Avantages en Production

| Critère | Interface Graphique | Ligne de Commande |
|---------|---------------------|-------------------|
| **Automatisation** | ❌ Impossible | ✅ Totale |
| **Rapidité** | 🐌 Lent | ⚡ Instantané |
| **Reproductibilité** | ❌ Erreurs humaines | ✅ Scripts fiables |
| **Monitoring** | ❌ Manuel | ✅ Continu |
| **Documentation** | 📝 À rédiger | ✅ Scripts = doc |
| **CI/CD** | ❌ Incompatible | ✅ Intégration native |
| **SSH Distant** | ❌ Complexe | ✅ Simple |
| **Logs** | ❌ Manuels | ✅ Automatiques |
| **Alertes** | ❌ Manuelles | ✅ Automatiques |

**En production, la ligne de commande n'est pas optionnelle, elle est ESSENTIELLE.**

---

## 3. Prérequis

### 3.1. Connaissances Requises

**Indispensable :**
- ✅ Bases du système Linux/Unix
- ✅ Navigation dans les répertoires (`cd`, `ls`, `pwd`)
- ✅ Utilisation basique de l'éditeur de texte (nano, vim, ou autre)
- ✅ Compréhension du concept de processus

**Recommandé :**
- ⭐ Notions de shell scripting (Bash)
- ⭐ Concept de variables d'environnement
- ⭐ Redirections (`>`, `>>`, `2>&1`)
- ⭐ Pipes (`|`)

**Pas nécessaire (on vous l'expliquera) :**
- ❌ Expertise en scripting Bash
- ❌ Connaissance approfondie de Linux
- ❌ Administration système avancée

### 3.2. Environnement Technique

**Système d'exploitation :**
- Linux (Ubuntu, Debian, CentOS, RHEL)
- macOS (compatible)
- Windows avec WSL2 (Windows Subsystem for Linux)

**PostgreSQL :**
- Version 18 (septembre 2025) - exemples de cette annexe
- Compatible avec versions 12-17 (syntaxe similaire)

**Outils nécessaires :**
```bash
# Vérifier PostgreSQL
psql --version
pg_dump --version

# Vérifier Bash
bash --version

# Outils système recommandés
which cron
which mail  # Pour alertes email
which bc    # Calculatrice en ligne de commande
```

**Accès requis :**
- ✅ Compte utilisateur avec accès PostgreSQL
- ✅ Accès SSH au serveur (si distant)
- ✅ Droits d'écriture dans `/usr/local/bin` (ou équivalent)
- ✅ Possibilité d'éditer le crontab

### 3.3. Installation des Outils PostgreSQL

**Si PostgreSQL est déjà installé, tous les outils CLI sont présents :**

```bash
# Localisation typique
/usr/bin/pg_*              # Debian/Ubuntu
/usr/pgsql-18/bin/pg_*     # CentOS/RHEL
/usr/local/bin/pg_*        # Installation depuis source

# Lister tous les outils PostgreSQL
ls -l /usr/bin/pg_* 2>/dev/null || ls -l /usr/pgsql-*/bin/pg_*
```

**Outils principaux disponibles :**
```
pg_ctl          # Contrôle du serveur
pg_dump         # Backup logique
pg_dumpall      # Backup complet (toutes les bases)
pg_restore      # Restauration
psql            # Client interactif
pg_isready      # Test de connexion
pg_basebackup   # Backup physique
createdb        # Créer une base
dropdb          # Supprimer une base
createuser      # Créer un utilisateur
...et bien d'autres
```

---

## 4. Environnement Shell

### 4.1. Qu'est-ce qu'un Shell ?

Le **shell** est l'interpréteur de commandes qui exécute vos instructions et scripts. Sous Linux/Unix, plusieurs shells existent, mais **Bash** (Bourne Again Shell) est le plus répandu.

**Analogie :** Le shell est comme un traducteur entre vous et le système d'exploitation. Vous lui parlez en "commandes" et il traduit en actions système.

### 4.2. Vérifier votre Shell

```bash
# Quel shell utilisez-vous ?
echo $SHELL

# Résultats possibles :
# /bin/bash   → Bash (le plus courant)
# /bin/zsh    → Zsh (macOS par défaut)
# /bin/sh     → Shell standard (lien vers bash généralement)

# Version de Bash
bash --version
```

**💡 Note :** Tous les scripts de cette annexe sont écrits pour Bash, mais fonctionnent généralement aussi en Zsh.

### 4.3. Variables d'Environnement Importantes

**Variables PostgreSQL :**

```bash
# PGDATA : Répertoire de données PostgreSQL
export PGDATA=/var/lib/postgresql/data

# PGHOST : Hôte PostgreSQL
export PGHOST=localhost

# PGPORT : Port PostgreSQL
export PGPORT=5432

# PGUSER : Utilisateur par défaut
export PGUSER=postgres

# PGDATABASE : Base de données par défaut
export PGDATABASE=postgres

# Afficher les variables
echo "PGDATA=$PGDATA"
env | grep ^PG
```

**Pourquoi c'est important ?**

Avec ces variables définies, vos commandes deviennent plus simples :

```bash
# Sans variables
pg_dump -h localhost -p 5432 -U postgres -d mabase > backup.sql

# Avec variables (si PGHOST, PGPORT, PGUSER définis)
pg_dump mabase > backup.sql
```

**Configuration permanente :**

```bash
# Ajouter dans ~/.bashrc ou ~/.bash_profile
echo 'export PGDATA=/var/lib/postgresql/data' >> ~/.bashrc
echo 'export PGHOST=localhost' >> ~/.bashrc
echo 'export PGPORT=5432' >> ~/.bashrc
echo 'export PGUSER=postgres' >> ~/.bashrc

# Recharger la configuration
source ~/.bashrc
```

### 4.4. Fichier .pgpass (Authentification Automatique)

Le fichier `.pgpass` permet d'éviter de saisir les mots de passe en ligne de commande (essentiel pour l'automatisation).

**Format :**
```
hostname:port:database:username:password
```

**Création :**

```bash
# Créer le fichier
cat > ~/.pgpass << EOF
localhost:5432:*:postgres:monmotdepasse
production.example.com:5432:*:admin:autremotdepasse
EOF

# IMPORTANT : Sécuriser les permissions (obligatoire)
chmod 600 ~/.pgpass

# Vérifier
ls -l ~/.pgpass
# Doit afficher : -rw------- (lecture/écriture pour le propriétaire uniquement)
```

**Utilisation :**

```bash
# Sans .pgpass (demande le mot de passe)
psql -U postgres -d mabase

# Avec .pgpass (connexion automatique)
psql -U postgres -d mabase
# Pas de prompt de mot de passe ! ✅
```

**⚠️ Sécurité :** Ne jamais mettre de mots de passe en clair dans les scripts. Utilisez toujours `.pgpass`.

### 4.5. Structure Typique d'un Script

**Anatomie d'un script Bash pour PostgreSQL :**

```bash
#!/bin/bash
# ==========================================
# Script : mon_script.sh
# Description : Ce que fait le script
# Auteur : Votre nom
# Date : 2025-11-21
# ==========================================

# 1. Configuration (variables)
DATABASE="mabase"
BACKUP_DIR="/var/backups/postgresql"
LOG_FILE="/var/log/mon_script.log"

# 2. Fonctions (code réutilisable)
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

check_postgres() {
    if ! pg_isready -d "$DATABASE" > /dev/null 2>&1; then
        log "❌ PostgreSQL non accessible"
        exit 1
    fi
}

# 3. Script principal
log "=== Démarrage du script ==="

check_postgres

# Votre logique ici
log "Traitement en cours..."

# Code de retour
log "=== Script terminé avec succès ==="
exit 0
```

**Éléments clés :**

1. **Shebang** (`#!/bin/bash`) : Indique quel interpréteur utiliser
2. **Commentaires** : Documenter le script
3. **Variables** : Configuration centralisée
4. **Fonctions** : Réutilisation du code
5. **Logs** : Traçabilité
6. **Codes de retour** : `exit 0` (succès), `exit 1` (erreur)

---

## 5. Organisation de cette Annexe

Cette annexe est organisée en **trois parties principales** :

### 5.1. Partie 1 : Outils PostgreSQL de Base

**Fichier dédié : `postgresql_outils_shell.md`**

**Contenu :**
- **pg_ctl** : Gestion du serveur PostgreSQL
  - Démarrage, arrêt, redémarrage
  - Modes d'arrêt (smart, fast, immediate)
  - Rechargement de configuration
  - Gestion du statut

- **pg_dump** : Sauvegarde logique
  - Formats de sortie (plain, custom, directory, tar)
  - Options essentielles
  - Backup sélectif
  - Parallélisme

- **pg_restore** : Restauration de sauvegardes
  - Restauration complète et sélective
  - Restauration parallèle
  - Gestion des erreurs
  - Optimisation

**Public :** Débutant à Intermédiaire
**Durée estimée :** 2-3 heures de lecture

### 5.2. Partie 2 : Scripts de Backup Automatisés

**Fichier dédié : `postgresql_scripts_backup_automatises.md`**

**Contenu :**
- **Stratégies de backup** : Fréquence, rétention, règle 3-2-1
- **Scripts de base** : Du minimal au robuste
- **Scripts avancés** : Production-ready avec configuration
- **Automatisation avec Cron** : Planification et surveillance
- **Rotation des backups** : Gestion de l'historique (GFS)
- **Destinations multiples** : Local, NAS, Cloud (S3)
- **Monitoring et alertes** : Détection des problèmes
- **Scripts de vérification** : Tests de restauration

**Public :** Intermédiaire à Avancé
**Durée estimée :** 3-4 heures de lecture

### 5.3. Partie 3 : Scripts de Monitoring Système

**Fichier dédié : `postgresql_scripts_monitoring_systeme.md`**

**Contenu :**
- **Introduction au monitoring** : Concepts, métriques, observabilité
- **Métriques système** : CPU, RAM, disque, réseau
- **Métriques PostgreSQL** : Connexions, requêtes, cache, transactions
- **Scripts de monitoring** : Santé, performance, ressources
- **Alertes et seuils** : Gestion des anomalies
- **Tableaux de bord** : Rapports HTML, dashboards
- **Intégration** : Prometheus, Grafana, Slack
- **Bonnes pratiques** : Méthodologie, architecture recommandée

**Public :** Intermédiaire à Avancé
**Durée estimée :** 4-5 heures de lecture

### 5.4. Parcours d'Apprentissage Recommandé

```
Débutant
    │
    ├─→ 1. Lire cette introduction
    │
    ├─→ 2. Maîtriser pg_ctl, pg_dump, pg_restore
    │        (Fichier : postgresql_outils_shell.md)
    │        • Pratiquer les commandes manuellement
    │        • Faire quelques backups/restores
    │
    ├─→ 3. Créer des scripts de backup simples
    │        (Fichier : postgresql_scripts_backup_automatises.md)
    │        • Commencer par scripts minimaux
    │        • Ajouter progressivement fonctionnalités
    │        • Automatiser avec cron
    │
    └─→ 4. Mettre en place le monitoring
             (Fichier : postgresql_scripts_monitoring_systeme.md)
             • Scripts de santé de base
             • Alertes simples
             • Dashboard HTML

Intermédiaire/Avancé
    │
    ├─→ Scripts de backup production
    │        • Configuration centralisée
    │        • Destinations multiples (NAS, S3)
    │        • Rotation GFS
    │        • Tests automatiques
    │
    └─→ Monitoring complet
             • Métriques système et PostgreSQL
             • Historique et tendances
             • Intégration Prometheus/Grafana
             • Alertes intelligentes
```

---

## 6. Conventions et Notations

### 6.1. Conventions Typographiques

**Dans cette annexe, nous utilisons :**

```bash
# Commentaire explicatif
commande --option argument

# ✅ Indique une bonne pratique
# ❌ Indique une mauvaise pratique
# ⚠️ Indique un avertissement
# 💡 Indique un conseil/astuce
```

**Variables :**
- `$VARIABLE` : Variable shell
- `CONSTANTE` : Valeur fixe en majuscules

**Chemins :**
- `/chemin/absolu/fichier.txt` : Chemin complet depuis la racine
- `chemin/relatif/fichier.txt` : Chemin relatif au répertoire actuel
- `~/.fichier` : Fichier dans le répertoire home de l'utilisateur

### 6.2. Symboles dans les Exemples

```bash
$ commande        # $ = Prompt utilisateur normal
# commande        # # = Prompt root (administrateur)

[user@host]$      # Prompt avec utilisateur et machine

# Résultat attendu
Sortie de la commande

# Commentaire explicatif dans le code
```

### 6.3. Codes de Retour

**En Bash, chaque commande retourne un code :**

```bash
# 0 = Succès
# 1-255 = Erreur

# Vérifier le code de retour de la dernière commande
echo $?

# Exemple
pg_dump mabase > backup.sql
if [ $? -eq 0 ]; then
    echo "✅ Backup réussi"
else
    echo "❌ Backup échoué"
fi
```

### 6.4. Niveaux de Difficulté

Chaque script est marqué par un niveau :

- 🟢 **DÉBUTANT** : Script simple, peu de lignes, concepts de base
- 🟡 **INTERMÉDIAIRE** : Script avec gestion d'erreurs, fonctions
- 🔴 **AVANCÉ** : Script complet, production-ready, optimisé

### 6.5. Priorité des Sections

Chaque section est marquée :

- ⭐ **ESSENTIEL** : À lire absolument
- ⭐⭐ **IMPORTANT** : Recommandé fortement
- ⭐⭐⭐ **AVANCÉ** : Pour aller plus loin

---

## 7. Premiers Pas

### 7.1. Votre Premier Script PostgreSQL

**Créons ensemble un script très simple :**

```bash
# 1. Créer le fichier
nano ~/mon_premier_script.sh

# 2. Copier ce contenu
#!/bin/bash
# Mon premier script PostgreSQL

echo "=== Premier Script PostgreSQL ==="
echo "Date : $(date)"
echo ""

# Vérifier PostgreSQL
if pg_isready > /dev/null 2>&1; then
    echo "✅ PostgreSQL est actif"
else
    echo "❌ PostgreSQL est arrêté"
fi

# Lister les bases de données
echo ""
echo "Bases de données disponibles :"
psql -l

echo ""
echo "=== Script terminé ==="

# 3. Sauvegarder et quitter (Ctrl+X, Y, Enter dans nano)

# 4. Rendre exécutable
chmod +x ~/mon_premier_script.sh

# 5. Exécuter
~/mon_premier_script.sh
```

**Résultat attendu :**
```
=== Premier Script PostgreSQL ===
Date : 2025-11-21 14:30:00

✅ PostgreSQL est actif

Bases de données disponibles :
   Name    |  Owner   | Encoding | ...
-----------+----------+----------+-----
 postgres  | postgres | UTF8     | ...
 mabase    | postgres | UTF8     | ...

=== Script terminé ===
```

**Bravo ! Vous venez d'écrire votre premier script PostgreSQL ! 🎉**

### 7.2. Test de Connexion

**Script pour tester la connexion à PostgreSQL :**

```bash
#!/bin/bash
# test_connection.sh

DATABASE="postgres"

echo "Test de connexion à PostgreSQL..."

# Méthode 1 : pg_isready
if pg_isready -d "$DATABASE" > /dev/null 2>&1; then
    echo "✅ Connexion possible"
else
    echo "❌ Connexion impossible"
    exit 1
fi

# Méthode 2 : Requête SQL simple
RESULT=$(psql -d "$DATABASE" -t -c "SELECT 1;" 2>/dev/null | xargs)

if [ "$RESULT" = "1" ]; then
    echo "✅ Requête SQL exécutée avec succès"
else
    echo "❌ Impossible d'exécuter une requête"
    exit 1
fi

# Informations
VERSION=$(psql -d "$DATABASE" -t -c "SELECT version();" | head -1 | xargs)
echo ""
echo "Version : $VERSION"

DATABASES=$(psql -d "$DATABASE" -t -c "SELECT COUNT(*) FROM pg_database WHERE datistemplate = false;" | xargs)
echo "Nombre de bases : $DATABASES"

echo ""
echo "✅ Tout fonctionne correctement !"
```

### 7.3. Ressources pour Apprendre Bash

**Si vous débutez en Bash, voici des ressources :**

**Documentation :**
- Bash Guide for Beginners : https://tldp.org/LDP/Bash-Beginners-Guide/html/
- Advanced Bash-Scripting Guide : https://tldp.org/LDP/abs/html/

**Tutoriels interactifs :**
- https://www.learnshell.org/
- https://www.codecademy.com/learn/learn-the-command-line

**Aide-mémoire :**
```bash
# Aide sur une commande
man pg_dump
pg_dump --help

# Rechercher dans l'historique
history | grep pg_dump

# Complétion automatique (Tab)
pg_<TAB><TAB>  # Liste toutes les commandes pg_*
```

---

## 8. Sécurité et Bonnes Pratiques

### 8.1. Sécurité des Scripts

**⚠️ Règles d'or de sécurité :**

#### 1. Ne JAMAIS mettre de mots de passe dans les scripts

```bash
# ❌ MAUVAIS - Mot de passe en clair
PGPASSWORD="monmotdepasse" pg_dump mabase > backup.sql

# ✅ BON - Utiliser .pgpass
# Fichier ~/.pgpass : localhost:5432:*:postgres:monmotdepasse
# Permissions : chmod 600 ~/.pgpass
pg_dump mabase > backup.sql
```

#### 2. Sécuriser les fichiers de script

```bash
# Scripts contenant de la logique sensible
chmod 700 mon_script.sh       # Lecture/Écriture/Exécution pour le propriétaire uniquement
chown postgres:postgres mon_script.sh

# Backups
chmod 600 backup.dump         # Lecture/Écriture pour le propriétaire uniquement
```

#### 3. Valider les entrées

```bash
# ❌ MAUVAIS - Injection SQL possible
DATABASE=$1
psql -d "$DATABASE" -c "SELECT * FROM users"

# ✅ BON - Valider l'entrée
DATABASE=$1

# Vérifier que la base existe
if ! psql -lqt | cut -d \| -f 1 | grep -qw "$DATABASE"; then
    echo "❌ Base de données '$DATABASE' inexistante"
    exit 1
fi

psql -d "$DATABASE" -c "SELECT * FROM users"
```

#### 4. Éviter les chemins relatifs

```bash
# ❌ MAUVAIS - Chemin relatif
cd backups
pg_dump mabase > backup.sql

# ✅ BON - Chemin absolu
BACKUP_DIR="/var/backups/postgresql"
mkdir -p "$BACKUP_DIR"
pg_dump mabase -f "$BACKUP_DIR/backup.sql"
```

### 8.2. Gestion des Erreurs

**Toujours gérer les erreurs dans les scripts :**

```bash
#!/bin/bash
# Arrêter le script en cas d'erreur
set -e  # Exit si une commande échoue
set -u  # Exit si une variable non définie est utilisée
set -o pipefail  # Exit si une commande dans un pipe échoue

# Alternative : Gestion manuelle
DATABASE="mabase"

if ! pg_dump "$DATABASE" -f backup.sql; then
    echo "❌ Échec du backup"
    # Envoyer une alerte
    echo "Backup failed" | mail -s "Alert" admin@example.com
    exit 1
fi

echo "✅ Backup réussi"
```

### 8.3. Logging et Traçabilité

**Toujours logger les actions importantes :**

```bash
#!/bin/bash
LOG_FILE="/var/log/mes_scripts.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Début du script"

if pg_dump mabase -f backup.sql; then
    log "✅ Backup réussi"
else
    log "❌ Backup échoué"
    exit 1
fi

log "Fin du script"
```

### 8.4. Tests Avant Production

**Checklist avant de mettre un script en production :**

- [ ] Testé manuellement plusieurs fois
- [ ] Testé avec des données réelles (non-production)
- [ ] Gestion des erreurs implémentée
- [ ] Logs configurés
- [ ] Codes de retour appropriés
- [ ] Documentation du script (commentaires)
- [ ] Permissions correctes (chmod, chown)
- [ ] Pas de mots de passe en clair
- [ ] Variables d'environnement vérifiées
- [ ] Chemins absolus utilisés

### 8.5. Principe du Moindre Privilège

**N'exécutez pas tout en root :**

```bash
# ✅ BON - Utilisateur dédié
# Créer un utilisateur pour PostgreSQL
sudo useradd -r -m -d /home/pgbackup -s /bin/bash pgbackup

# Scripts dans son répertoire
sudo mkdir -p /home/pgbackup/scripts
sudo chown pgbackup:pgbackup /home/pgbackup/scripts

# Exécuter en tant que cet utilisateur
sudo -u pgbackup /home/pgbackup/scripts/backup.sh

# Cron de cet utilisateur
sudo crontab -u pgbackup -e
```

### 8.6. Documentation et Commentaires

**Un script bien documenté est un script maintenable :**

```bash
#!/bin/bash
# ===========================================
# Script : backup_production.sh
# Description : Backup quotidien de la base production
# Auteur : Équipe DevOps
# Date création : 2025-11-21
# Dernière modification : 2025-11-21
# Version : 1.0
#
# Usage : ./backup_production.sh
# Cron : 0 2 * * * /usr/local/bin/backup_production.sh
#
# Prérequis :
# - PostgreSQL 18+
# - Fichier ~/.pgpass configuré
# - Espace disque > 50GB
#
# Contacts :
# - Support : devops@example.com
# - Astreinte : +33 6 XX XX XX XX
# ===========================================

# Configuration
DATABASE="production"
BACKUP_DIR="/var/backups/postgresql"

# ... reste du script
```

---

## Conclusion de l'Introduction

### Ce que vous avez appris

- ✅ **Pourquoi la ligne de commande est essentielle** pour PostgreSQL
- ✅ **Les avantages de l'automatisation** (scripts vs GUI)
- ✅ **L'environnement shell** et les variables PostgreSQL
- ✅ **La structure de cette annexe** et son organisation
- ✅ **Les conventions** utilisées dans les exemples
- ✅ **Les premiers pas** avec des scripts simples
- ✅ **Les règles de sécurité** fondamentales

### Prochaines Étapes

**Vous êtes maintenant prêt à :**

1. **Découvrir les outils de base** (pg_ctl, pg_dump, pg_restore)
   → Fichier : `postgresql_outils_shell.md`

2. **Créer des scripts de backup automatisés**
   → Fichier : `postgresql_scripts_backup_automatises.md`

3. **Mettre en place le monitoring**
   → Fichier : `postgresql_scripts_monitoring_systeme.md`

### Points Clés à Retenir

1. **La ligne de commande = Automatisation**
   - Essentielle en production
   - Scripts réutilisables et fiables

2. **Sécurité avant tout**
   - Pas de mots de passe en clair
   - Fichier .pgpass avec permissions 600
   - Gestion des erreurs obligatoire

3. **Progression graduelle**
   - Commencer simple
   - Ajouter fonctionnalités progressivement
   - Tester abondamment

4. **Documentation = Maintenance facilitée**
   - Commenter les scripts
   - Logger les actions
   - Documenter les procédures

### Message Final

**"Le meilleur moment pour automatiser était hier. Le second meilleur moment est maintenant."**

Cette annexe vous donnera tous les outils pour administrer PostgreSQL efficacement en ligne de commande. Que vous soyez débutant ou expérimenté, vous trouverez des scripts adaptés à votre niveau.

**N'ayez pas peur de la ligne de commande. Embrassez-la, et PostgreSQL deviendra votre allié le plus fidèle !** 🚀

---

**Version :** 1.0 - Novembre 2025
**PostgreSQL :** Version 18
**Auteur :** Formation PostgreSQL pour Développeurs et DevOps

---

## Fichiers de cette Annexe

📄 **postgresql_annexe_g_introduction.md** (ce fichier)
   - Introduction générale
   - Prérequis et environnement
   - Premiers pas et bonnes pratiques

📄 **postgresql_outils_shell.md**
   - pg_ctl, pg_dump, pg_restore
   - Commandes essentielles
   - Exemples concrets

📄 **postgresql_scripts_backup_automatises.md**
   - Stratégies de backup
   - Scripts automatisés
   - Rotation et destinations

📄 **postgresql_scripts_monitoring_systeme.md**
   - Monitoring système et PostgreSQL
   - Métriques et alertes
   - Tableaux de bord

---

**Prêt à commencer ? Passez au fichier suivant : `postgresql_outils_shell.md` ! ✨**

⏭️ [pg_ctl, pg_dump, pg_restore](/annexes/commandes-shell-scripts/01-commandes-pg-principales.md)
