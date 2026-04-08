🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.2. Configuration de l'Authentification (pg_hba.conf)

## Introduction

L'authentification dans PostgreSQL est contrôlée par un fichier de configuration crucial appelé **pg_hba.conf** (HBA signifie "Host-Based Authentication" ou "Authentification basée sur l'hôte"). Ce fichier est le **gardien** de votre base de données : il définit qui peut se connecter, depuis où, à quelle base de données, et comment ils doivent s'authentifier.

> **Pour les débutants** : Imaginez pg_hba.conf comme la liste des règles de sécurité à l'entrée d'un immeuble. Chaque ligne dit : "Si tu viens de tel endroit, pour visiter tel appartement, avec telle identité, alors tu dois montrer tel type de badge". PostgreSQL vérifie ces règles dans l'ordre pour décider si une connexion est autorisée.

---

## Pourquoi pg_hba.conf est-il Crucial ?

### Le Rôle Central dans la Sécurité

```
┌─────────────────────────────────────────────────────────────┐
│                     FLUX DE CONNEXION                       │
└─────────────────────────────────────────────────────────────┘

┌──────────┐                                    ┌──────────┐
│  Client  │                                    │PostgreSQL│
│  (psql)  │                                    │  Server  │
└────┬─────┘                                    └────┬─────┘
     │                                               │
     │ 1. Demande de connexion                       │
     │    (user=alice, db=mydb, from=192.168.1.100)  │
     ├──────────────────────────────────────────────►│
     │                                               │
     │                           2. Lecture pg_hba.conf
     │                              Recherche d'une règle
     │                              correspondante
     │                                               │
     │                              3. Règle trouvée ?
     │                                 ├─ OUI → Quelle méthode ?
     │                                 └─ NON → REJET
     │                                               │
     │ 4. Demande d'authentification                 │
     │    (selon la méthode : password, cert, etc.)  │
     │◄──────────────────────────────────────────────┤
     │                                               │
     │ 5. Fournit les credentials                    │
     ├──────────────────────────────────────────────►│
     │                                               │
     │                              6. Vérification
     │                                 ├─ OK → Connexion accordée
     │                                 └─ KO → REJET
     │                                               │
     │ 7. ✅ Connexion établie (ou ❌ refusée)       │
     │◄──────────────────────────────────────────────┤
     │                                               │
```

### Ce que pg_hba.conf Contrôle

1. **Qui** peut se connecter (utilisateur PostgreSQL)  
2. **D'où** ils peuvent se connecter (adresse IP, réseau)  
3. **À quoi** ils peuvent accéder (base de données spécifique)  
4. **Comment** ils doivent s'authentifier (méthode)

**Sans pg_hba.conf**, PostgreSQL ne peut pas déterminer si une connexion doit être acceptée ou refusée.

---

## Localisation du Fichier pg_hba.conf

### Trouver le Fichier

**Méthode 1 : Depuis PostgreSQL**
```sql
-- Se connecter à PostgreSQL et demander l'emplacement
SHOW hba_file;

-- Résultat typique :
--              hba_file
-- ------------------------------------
--  /var/lib/postgresql/data/pg_hba.conf
```

**Méthode 2 : Emplacements Standards par Système d'Exploitation**

| Système | Chemin Typique |
|---------|----------------|
| **Linux (Debian/Ubuntu)** | `/etc/postgresql/18/main/pg_hba.conf` |
| **Linux (RedHat/CentOS)** | `/var/lib/pgsql/18/data/pg_hba.conf` |
| **Linux (Docker)** | `/var/lib/postgresql/data/pg_hba.conf` |
| **macOS (Homebrew)** | `/opt/homebrew/var/postgresql@18/pg_hba.conf` |
| **Windows** | `C:\Program Files\PostgreSQL\18\data\pg_hba.conf` |

**Méthode 3 : Depuis la ligne de commande**
```bash
# Linux/macOS
sudo -u postgres psql -c "SHOW hba_file;"

# Ou chercher dans le système
find / -name pg_hba.conf 2>/dev/null
```

### Permissions du Fichier

Le fichier pg_hba.conf doit être :
- **Lisible** par l'utilisateur système qui exécute PostgreSQL (généralement `postgres`)  
- **Non modifiable** par les utilisateurs non autorisés

```bash
# Vérifier les permissions
ls -l /var/lib/postgresql/data/pg_hba.conf

# Résultat attendu :
# -rw------- 1 postgres postgres 4640 Nov 22 10:30 pg_hba.conf
#   ↑         ↑         ↑
#   600    propriétaire groupe
```

---

## Structure du Fichier pg_hba.conf

### Anatomie d'une Ligne

Chaque ligne de pg_hba.conf (hors commentaires) définit une **règle d'authentification** avec la structure suivante :

```
TYPE    DATABASE    USER    ADDRESS    METHOD    [OPTIONS]
```

**Visualisation :**
```
┌────────┬──────────┬──────┬──────────┬────────┬──────────┐
│  TYPE  │ DATABASE │ USER │ ADDRESS  │ METHOD │ OPTIONS  │
├────────┼──────────┼──────┼──────────┼────────┼──────────┤
│ local  │   all    │ all  │    -     │  peer  │    -     │
│ host   │  mydb    │alice │127.0.0.1 │  scram │    -     │
│ hostssl│   all    │  all │ 0.0.0.0/0│  scram │    -     │
└────────┴──────────┴──────┴──────────┴────────┴──────────┘
```

### Les 5 Colonnes Fondamentales

#### 1. TYPE (Type de Connexion)

Définit **comment** le client se connecte au serveur.

| Type | Description | Cas d'usage |
|------|-------------|-------------|
| **local** | Connexion Unix socket (même machine) | Connexions locales via `/var/run/postgresql/.s.PGSQL.5432` |
| **host** | Connexion TCP/IP (avec ou sans SSL) | Connexions réseau (local ou distant) |
| **hostssl** | Connexion TCP/IP avec SSL obligatoire | Connexions sécurisées uniquement |
| **hostnossl** | Connexion TCP/IP sans SSL | Connexions non chiffrées (déconseillé) |
| **hostgssenc** | Connexion avec chiffrement GSSAPI | Environnements Kerberos |

**Exemples :**
```conf
# Connexion locale (Unix socket)
local   all   postgres   peer

# Connexion réseau (TCP/IP)
host    all   all        127.0.0.1/32   scram-sha-256

# Connexion SSL obligatoire
hostssl all   all        0.0.0.0/0      scram-sha-256
```

#### 2. DATABASE (Base de Données)

Définit **quelle(s) base(s) de données** la règle s'applique.

| Valeur | Signification |
|--------|---------------|
| **all** | Toutes les bases de données |
| **nomdb** | Une base spécifique (exemple: `mydb`, `production`) |
| **@fichier** | Liste de bases dans un fichier |
| **sameuser** | Base portant le même nom que l'utilisateur |
| **samerole** | Bases accessibles aux rôles de l'utilisateur |
| **replication** | Connexions de réplication |
| **db1,db2,db3** | Plusieurs bases (séparées par virgules) |

**Exemples :**
```conf
# Toutes les bases
local   all              postgres   peer

# Base spécifique
host    production       appuser    10.0.1.0/24   scram-sha-256

# Plusieurs bases
host    db1,db2,db3      alice      192.168.1.100 scram-sha-256

# Réplication
host    replication      replicator 10.0.2.0/24   scram-sha-256

# Base = username
local   sameuser         all        peer
```

#### 3. USER (Utilisateur PostgreSQL)

Définit **quel(s) utilisateur(s)** PostgreSQL la règle s'applique.

| Valeur | Signification |
|--------|---------------|
| **all** | Tous les utilisateurs |
| **username** | Un utilisateur spécifique |
| **@fichier** | Liste d'utilisateurs dans un fichier |
| **+groupname** | Membres d'un groupe/rôle |
| **user1,user2** | Plusieurs utilisateurs (séparés par virgules) |

**Exemples :**
```conf
# Tous les utilisateurs
host    all   all           127.0.0.1/32   scram-sha-256

# Utilisateur spécifique
local   all   postgres      peer

# Groupe d'utilisateurs
host    all   +developers   10.0.1.0/24    scram-sha-256

# Plusieurs utilisateurs
host    mydb  alice,bob,charlie   192.168.1.0/24   scram-sha-256
```

#### 4. ADDRESS (Adresse Source)

Définit **d'où** la connexion peut provenir. **Non utilisé pour `local`**.

**Formats supportés :**

| Format | Exemple | Signification |
|--------|---------|---------------|
| **Adresse IP/CIDR** | `192.168.1.100/32` | Une IP spécifique (/32) |
| **Réseau/CIDR** | `10.0.0.0/8` | Un réseau entier |
| **all** | `0.0.0.0/0` | N'importe quelle adresse IPv4 |
| **all** | `::/0` | N'importe quelle adresse IPv6 |
| **Hostname** | `server.company.com` | Résolution DNS (lent, déconseillé) |
| **samehost** | - | Même machine (localhost) |
| **samenet** | - | Même sous-réseau |

**Exemples :**
```conf
# Localhost uniquement
host    all   all   127.0.0.1/32          scram-sha-256  
host    all   all   ::1/128               scram-sha-256  

# Réseau local
host    all   all   192.168.1.0/24        scram-sha-256

# Réseau d'entreprise complet
host    all   all   10.0.0.0/8            scram-sha-256

# N'importe où (ATTENTION : dangereux sans SSL)
hostssl all   all   0.0.0.0/0             scram-sha-256

# Même machine (samehost)
host    all   all   samehost              scram-sha-256
```

**⚠️ Notation CIDR : Rappel**
```
192.168.1.100/32  → Une seule IP (192.168.1.100)
192.168.1.0/24    → 256 IPs (192.168.1.0 à 192.168.1.255)
10.0.0.0/8        → 16,777,216 IPs (10.0.0.0 à 10.255.255.255)
0.0.0.0/0         → Toutes les IPv4
```

#### 5. METHOD (Méthode d'Authentification)

Définit **comment** l'utilisateur doit prouver son identité. Les méthodes détaillées sont couvertes dans les sections suivantes (16.2.1, 16.2.2, 16.2.3), mais voici un aperçu :

| Méthode | Sécurité | Description Brève |
|---------|----------|-------------------|
| **trust** | ⚠️ Aucune | Pas d'authentification (dangereux) |
| **reject** | N/A | Refuser la connexion |
| **scram-sha-256** | ✅ Forte | Standard moderne (recommandé) |
| **md5** | ⚠️ Faible | Déprécié (à éviter) |
| **password** | ❌ Nulle | Obsolète (à éviter) |
| **peer** | ✅ Moyenne | User système = user PostgreSQL |
| **ident** | ✅ Moyenne | Via serveur ident |
| **cert** | ✅ Très forte | Certificats SSL/TLS |
| **ldap** | ✅ Forte | Active Directory |
| **oauth** | ✅ Forte | OAuth 2.0 (PG 18+) |

**Exemples :**
```conf
# Méthode moderne sécurisée
hostssl all   all   0.0.0.0/0   scram-sha-256

# Connexion locale sans mot de passe (peer)
local   all   postgres   peer

# Refuser explicitement
host    all   banned_user   0.0.0.0/0   reject
```

#### 6. OPTIONS (Optionnel)

Certaines méthodes acceptent des options supplémentaires.

**Exemples :**
```conf
# LDAP avec options
host all all 0.0.0.0/0 ldap ldapserver=ad.company.com ldapport=636 ldaptls=1

# SCRAM avec channel binding
hostssl all all 0.0.0.0/0 scram-sha-256 scram_channel_binding=require

# OAuth avec options
host all all 0.0.0.0/0 oauth oauth_issuer_url="https://auth.company.com"
```

---

## L'Ordre des Règles : Premier Match Gagne

### Principe Fondamental

PostgreSQL lit le fichier pg_hba.conf **ligne par ligne, de haut en bas**, et s'arrête à la **première règle qui correspond**.

**Analogie** : C'est comme une liste de filtres dans votre boîte email. Le premier filtre qui correspond s'applique, et les suivants sont ignorés.

### Exemple Illustratif

```conf
# pg_hba.conf

# Ligne 1 : Admin local
local   all   postgres   peer

# Ligne 2 : Développeurs depuis le réseau local
host    all   +developers   192.168.1.0/24   scram-sha-256

# Ligne 3 : Utilisateur Alice depuis n'importe où
hostssl all   alice         0.0.0.0/0        scram-sha-256

# Ligne 4 : Tous les autres refusés
host    all   all           0.0.0.0/0        reject
```

**Scénarios de connexion :**

**Scénario 1 :**
```
Connexion : local, user=postgres, db=mydb
→ Correspond à la ligne 1
→ Méthode : peer
→ ✅ Accepté si user système = postgres
```

**Scénario 2 :**
```
Connexion : host, user=bob (membre de developers), from=192.168.1.50, db=testdb
→ Ne correspond PAS à la ligne 1 (pas local)
→ Correspond à la ligne 2 (IP dans 192.168.1.0/24 + membre de developers)
→ Méthode : scram-sha-256
→ ✅ Accepté si mot de passe correct
```

**Scénario 3 :**
```
Connexion : hostssl, user=alice, from=203.0.113.10, db=production
→ Ne correspond PAS à la ligne 1 (pas local)
→ Ne correspond PAS à la ligne 2 (IP hors réseau local)
→ Correspond à la ligne 3 (user=alice, SSL)
→ Méthode : scram-sha-256
→ ✅ Accepté si mot de passe correct
```

**Scénario 4 :**
```
Connexion : host, user=charlie, from=203.0.113.20, db=mydb
→ Ne correspond à aucune ligne 1-3
→ Correspond à la ligne 4 (catch-all)
→ Méthode : reject
→ ❌ REFUSÉ explicitement
```

### ⚠️ Piège Courant : Ordre Incorrect

**❌ Configuration Incorrecte :**
```conf
# ERREUR : Règle trop permissive en premier
host    all   all        0.0.0.0/0         scram-sha-256

# Cette règle ne sera JAMAIS atteinte
host    all   +admins    192.168.1.0/24    cert
```

Ici, la deuxième ligne est inutile car toutes les connexions correspondent déjà à la première ligne.

**✅ Configuration Correcte :**
```conf
# Règles spécifiques d'abord
host    all   +admins    192.168.1.0/24    cert

# Règle générale après
host    all   all        0.0.0.0/0         scram-sha-256
```

**Principe d'or :** Toujours placer les règles **les plus spécifiques en premier**, et les plus générales à la fin.

---

## Exemple de Fichier pg_hba.conf Complet et Commenté

```conf
# PostgreSQL Client Authentication Configuration File
# ===================================================
#
# Format : TYPE  DATABASE  USER  ADDRESS  METHOD  [OPTIONS]

# ──────────────────────────────────────────────────────────
# 1. CONNEXIONS LOCALES (Unix sockets)
# ──────────────────────────────────────────────────────────

# Admin local : peer (user système = user PostgreSQL)
local   all   postgres   peer

# Autres utilisateurs locaux : peer également
local   all   all        peer

# ──────────────────────────────────────────────────────────
# 2. CONNEXIONS LOCALHOST (TCP/IP sur 127.0.0.1)
# ──────────────────────────────────────────────────────────

# IPv4 localhost
host    all   all   127.0.0.1/32   scram-sha-256

# IPv6 localhost
host    all   all   ::1/128        scram-sha-256

# ──────────────────────────────────────────────────────────
# 3. RÉPLICATION (Streaming Replication)
# ──────────────────────────────────────────────────────────

# Réplication depuis les standby servers
host    replication   replicator   10.0.2.10/32   scram-sha-256  
host    replication   replicator   10.0.2.11/32   scram-sha-256  

# ──────────────────────────────────────────────────────────
# 4. CONNEXIONS RÉSEAU INTERNE (Réseau d'entreprise)
# ──────────────────────────────────────────────────────────

# Admins : certificats obligatoires
hostssl all   +admins        10.0.1.0/24   cert   clientcert=verify-full

# Développeurs : SCRAM depuis le réseau dev
host    all   +developers    10.0.3.0/24   scram-sha-256

# Applications : comptes de service
host    production   app_user     10.0.4.0/24   scram-sha-256

# ──────────────────────────────────────────────────────────
# 5. CONNEXIONS VPN (Utilisateurs distants)
# ──────────────────────────────────────────────────────────

# Utilisateurs via VPN : SSL obligatoire
hostssl all   all   192.168.100.0/24   scram-sha-256

# ──────────────────────────────────────────────────────────
# 6. CONNEXIONS EXTERNES (Internet)
# ──────────────────────────────────────────────────────────

# Comptes externes : SSL + SCRAM
hostssl all   +external_users   0.0.0.0/0   scram-sha-256

# ──────────────────────────────────────────────────────────
# 7. RÈGLE DE REJET (Catch-all)
# ──────────────────────────────────────────────────────────

# Tout le reste est refusé explicitement
host    all   all   0.0.0.0/0   reject  
host    all   all   ::/0        reject  
```

---

## Modifications et Rechargement

### Éditer le Fichier

**⚠️ Important :** Toujours faire une sauvegarde avant modification !

```bash
# Sauvegarder l'original
sudo cp /var/lib/postgresql/data/pg_hba.conf \
        /var/lib/postgresql/data/pg_hba.conf.backup

# Éditer avec votre éditeur préféré
sudo nano /var/lib/postgresql/data/pg_hba.conf
# ou
sudo vim /var/lib/postgresql/data/pg_hba.conf
```

### Vérifier la Syntaxe (PostgreSQL 18)

PostgreSQL 18 offre un outil pour valider pg_hba.conf avant de l'activer :

```bash
# Vérifier la syntaxe (ne modifie rien)
sudo -u postgres pg_ctl -D /var/lib/postgresql/data check

# Ou utiliser l'outil dédié
sudo -u postgres pg_hba_file_rules /var/lib/postgresql/data/pg_hba.conf
```

### Recharger la Configuration

Après modification, PostgreSQL doit **recharger** le fichier. Trois méthodes :

**Méthode 1 : Via SQL (Recommandée)**
```sql
-- Depuis une connexion PostgreSQL
SELECT pg_reload_conf();

-- Résultat :
--  pg_reload_conf
-- ----------------
--  t
-- (Pas de déconnexion des clients existants)
```

**Méthode 2 : Via pg_ctl**
```bash
sudo -u postgres pg_ctl reload -D /var/lib/postgresql/data
```

**Méthode 3 : Via systemctl (Linux)**
```bash
# Reload (sans interruption)
sudo systemctl reload postgresql

# Ou
sudo service postgresql reload
```

**⚠️ NE PAS utiliser restart** : Cela déconnecterait tous les clients !

```bash
# ❌ À éviter (sauf si nécessaire)
sudo systemctl restart postgresql
```

### Vérifier l'Application des Changements

```sql
-- Voir quand le fichier a été rechargé
SELECT pg_postmaster_start_time(), pg_conf_load_time();

-- Résultat :
--  pg_postmaster_start_time |   pg_conf_load_time
-- --------------------------+------------------------
--  2025-11-22 10:00:00+00   | 2025-11-22 14:30:00+00
--                  ↑                    ↑
--            Démarrage           Dernier reload
```

---

## Tester votre Configuration

### Script de Test

```bash
#!/bin/bash
# test_pg_hba.sh - Tester différentes connexions

echo "=== Test 1 : Connexion locale (peer) ==="  
psql -U postgres -c "SELECT current_user;"  

echo "=== Test 2 : Connexion TCP localhost (SCRAM) ==="  
PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d mydb -c "SELECT current_user;"  

echo "=== Test 3 : Connexion réseau (SCRAM) ==="  
PGPASSWORD=mypassword psql -h 192.168.1.10 -U appuser -d production -c "SELECT current_user;"  

echo "=== Test 4 : Connexion SSL (SCRAM) ==="  
PGPASSWORD=mypassword psql "host=pg.company.com user=alice dbname=mydb sslmode=require" -c "SELECT current_user;"  
```

### Interpréter les Erreurs

**Erreur 1 : Aucune règle correspondante**
```
FATAL: no pg_hba.conf entry for host "192.168.1.100", user "alice", database "mydb", SSL off
```

**Cause :** Aucune ligne de pg_hba.conf ne correspond à cette connexion.

**Solution :** Ajouter une règle correspondante :
```conf
host    mydb   alice   192.168.1.100/32   scram-sha-256
```

**Erreur 2 : Authentification échouée**
```
FATAL: password authentication failed for user "alice"
```

**Cause :** La règle existe, mais l'authentification a échoué (mauvais mot de passe, certificat invalide, etc.).

**Solution :** Vérifier les credentials fournis.

**Erreur 3 : SSL requis mais non fourni**
```
FATAL: no pg_hba.conf entry for host "192.168.1.100", user "alice", database "mydb", SSL off
```

**Cause :** La règle utilise `hostssl` mais le client se connecte sans SSL.

**Solution :** Ajouter `sslmode=require` dans la connexion :
```bash
psql "host=pg.company.com user=alice dbname=mydb sslmode=require"
```

---

## Bonnes Pratiques de Configuration

### 1. Principe du Moindre Privilège ✅

```conf
# ❌ Trop permissif
host    all   all   0.0.0.0/0   trust

# ✅ Restrictif et sécurisé
hostssl production   app_user   10.0.4.100/32   scram-sha-256
```

### 2. Règles Spécifiques d'Abord ✅

```conf
# ✅ Correct : Spécifique → Général
host    all   +admins      10.0.1.0/24   cert  
host    all   +developers  10.0.2.0/24   scram-sha-256  
host    all   all          0.0.0.0/0     reject  

# ❌ Incorrect : Général en premier rend le reste inutile
host    all   all          0.0.0.0/0     scram-sha-256  
host    all   +admins      10.0.1.0/24   cert  # Jamais atteint !  
```

### 3. Toujours Utiliser SSL pour les Connexions Réseau ✅

```conf
# ✅ SSL obligatoire
hostssl all   all   0.0.0.0/0   scram-sha-256

# ❌ Sans SSL (dangereux)
host    all   all   0.0.0.0/0   scram-sha-256
```

### 4. Limiter les Réseaux Autorisés ✅

```conf
# ❌ Trop large
host    all   all   0.0.0.0/0   scram-sha-256

# ✅ Limité aux réseaux connus
host    all   all   10.0.0.0/8         scram-sha-256  # Réseau interne  
hostssl all   all   192.168.100.0/24   scram-sha-256  # VPN  
```

### 5. Utiliser des Groupes/Rôles ✅

```conf
# ❌ Répétitif
host    all   alice    10.0.1.0/24   scram-sha-256  
host    all   bob      10.0.1.0/24   scram-sha-256  
host    all   charlie  10.0.1.0/24   scram-sha-256  

# ✅ Groupe
host    all   +developers   10.0.1.0/24   scram-sha-256
```

```sql
-- Créer le groupe
CREATE ROLE developers;

-- Ajouter les membres
GRANT developers TO alice, bob, charlie;
```

### 6. Rejeter Explicitement en Fin de Fichier ✅

```conf
# ... règles normales ...

# Catch-all : rejeter tout le reste
host    all   all   0.0.0.0/0   reject  
host    all   all   ::/0        reject  
```

### 7. Commenter Abondamment ✅

```conf
# ──────────────────────────────────────────────────────────
# PRODUCTION DATABASE - Accès depuis le réseau applicatif
# Contact : ops@company.com
# Dernière modification : 2025-11-22 par alice
# ──────────────────────────────────────────────────────────
host    production   app_user   10.0.4.0/24   scram-sha-256
```

### 8. Versionner pg_hba.conf ✅

```bash
# Initialiser un repo Git
cd /var/lib/postgresql/data  
git init  
git add pg_hba.conf postgresql.conf  
git commit -m "Initial configuration"  

# Après chaque modification
git add pg_hba.conf  
git commit -m "Allow developers from 10.0.3.0/24"  

# Voir l'historique
git log pg_hba.conf
```

---

## Stratégies de Configuration par Environnement

### Développement Local

```conf
# Confiance totale en local (développement uniquement)
local   all   all   trust  
host    all   all   127.0.0.1/32   trust  
host    all   all   ::1/128        trust  

# Rejeter tout le reste
host    all   all   0.0.0.0/0   reject
```

### Staging / Test

```conf
# Admin local
local   all   postgres   peer

# Développeurs depuis le réseau test
host    all   +developers   10.0.10.0/24   scram-sha-256

# Applications de test
host    staging   test_app_user   10.0.11.0/24   scram-sha-256

# Rejeter le reste
host    all   all   0.0.0.0/0   reject
```

### Production

```conf
# Admin local uniquement (pas de connexion distante pour postgres)
local   all   postgres   peer

# Réplication
hostssl replication   replicator   10.0.2.0/24   scram-sha-256

# Admins : Certificats + IP restreinte
hostssl all   +admins   10.0.1.0/24   cert   clientcert=verify-full

# Applications : Comptes dédiés + IP restreinte
hostssl production   app_user   10.0.4.100/32   scram-sha-256

# Monitoring
host    all   monitoring_user   10.0.5.50/32   scram-sha-256

# Rejeter TOUT le reste (important!)
host    all   all   0.0.0.0/0   reject  
host    all   all   ::/0        reject  
```

---

## Debugging et Logs

### Activer les Logs d'Authentification

```conf
# postgresql.conf

# Logger toutes les tentatives de connexion
log_connections = on  
log_disconnections = on  

# Logger les échecs d'authentification
log_failed_connections = on  # PostgreSQL 18+

# Détail des messages
log_line_prefix = '%t [%p] %u@%d from %h '

# Niveau de log
log_min_messages = info
```

### Analyser les Logs

**Exemple de log avec succès :**
```
2025-11-22 14:30:00 UTC [12345] alice@mydb from 192.168.1.100 LOG: connection authorized: user=alice database=mydb SSL enabled (protocol=TLSv1.3, cipher=TLS_AES_256_GCM_SHA384, bits=256)
```

**Exemple de log avec échec :**
```
2025-11-22 14:31:00 UTC [12346] bob@production from 203.0.113.50 FATAL: no pg_hba.conf entry for host "203.0.113.50", user "bob", database "production", SSL off
```

**Analyse :**
- **User** : bob  
- **Database** : production  
- **IP** : 203.0.113.50  
- **SSL** : Non  
- **Problème** : Aucune règle pg_hba.conf ne correspond

**Solution :** Ajouter une règle ou vérifier que l'utilisateur se connecte avec SSL.

### Identifier les Règles Appliquées

PostgreSQL 18 introduit une vue système pour voir les règles pg_hba.conf :

```sql
-- Voir toutes les règles pg_hba.conf
SELECT * FROM pg_hba_file_rules;

-- Résultat :
--  line_number | type  | database | user_name |  address   |    auth_method
-- -------------+-------+----------+-----------+------------+-------------------
--      10      | local | {all}    | {postgres}| NULL       | peer
--      15      | host  | {all}    | {all}     | 127.0.0.1  | scram-sha-256
--      20      | hostssl| {all}   | {all}     | 0.0.0.0/0  | scram-sha-256
```

```sql
-- Trouver quelle règle s'applique à une connexion
SELECT * FROM pg_hba_file_rules  
WHERE 'mydb' = ANY(database)  
  AND 'alice' = ANY(user_name);
```

---

## Cas d'Usage Avancés

### 1. Différentes Bases, Différentes Méthodes

```conf
# Base de production : Sécurité maximale
hostssl production   app_user      10.0.4.0/24   cert

# Base de développement : SCRAM standard
host    development  +developers   10.0.3.0/24   scram-sha-256

# Base de test : Trust en local
local   test         all           trust
```

### 2. Multi-Tenant avec Isolation

```conf
# Tenant A : Réseau dédié
host    tenant_a_db   tenant_a_user   10.1.0.0/16   scram-sha-256

# Tenant B : Réseau dédié
host    tenant_b_db   tenant_b_user   10.2.0.0/16   scram-sha-256

# Tenant C : Réseau dédié
host    tenant_c_db   tenant_c_user   10.3.0.0/16   scram-sha-256

# Interdire les cross-tenants
host    tenant_a_db   tenant_b_user   0.0.0.0/0     reject  
host    tenant_b_db   tenant_a_user   0.0.0.0/0     reject  
```

### 3. Maintenance Windows

```conf
# Pendant la maintenance : Admins seulement
host    all   +admins        10.0.1.0/24     scram-sha-256

# Bloquer temporairement les utilisateurs normaux
# (Commenter/Décommenter selon besoin)
# host    all   +normal_users  10.0.0.0/8      scram-sha-256

# Rejeter tout le reste
host    all   all            0.0.0.0/0       reject
```

---

## Checklist de Sécurité pg_hba.conf

### Avant la Mise en Production

- [ ] **SSL activé** pour toutes les connexions réseau (`hostssl`)  
- [ ] **Pas de `trust`** sauf en local pour postgres  
- [ ] **Pas de `password` ou `md5`** (utiliser `scram-sha-256`)  
- [ ] **Adresses IP restreintes** (pas de `0.0.0.0/0` sans SSL)  
- [ ] **Règle de rejet finale** (`reject` en catch-all)  
- [ ] **Admin `postgres` accessible uniquement en local**  
- [ ] **Comptes applicatifs avec IP dédiées**  
- [ ] **Logs d'authentification activés**  
- [ ] **Fichier versionné** (Git)  
- [ ] **Sauvegarde de l'ancien fichier** avant modification  
- [ ] **Tests de connexion** effectués  
- [ ] **Documentation à jour** (commentaires dans le fichier)

### Audit Périodique (Trimestriel)

```bash
# Script d'audit pg_hba.conf
#!/bin/bash

echo "=== Audit pg_hba.conf ==="

# 1. Chercher les méthodes faibles
echo "❌ Méthodes faibles trouvées :"  
grep -E "trust|password|md5" /var/lib/postgresql/data/pg_hba.conf  

# 2. Chercher les règles trop permissives
echo "⚠️  Règles 0.0.0.0/0 trouvées :"  
grep "0.0.0.0/0" /var/lib/postgresql/data/pg_hba.conf  

# 3. Vérifier les connexions sans SSL
echo "⚠️  Connexions host (non-SSL) trouvées :"  
grep "^host " /var/lib/postgresql/data/pg_hba.conf | grep -v "127.0.0.1"  

# 4. Compter les règles
echo "📊 Nombre total de règles :"  
grep -v "^#" /var/lib/postgresql/data/pg_hba.conf | grep -v "^$" | wc -l  

echo "=== Fin de l'audit ==="
```

---

## Ressources et Documentation

### Documentation Officielle

- **PostgreSQL Authentication** : https://www.postgresql.org/docs/18/auth-pg-hba-conf.html  
- **Client Authentication** : https://www.postgresql.org/docs/18/client-authentication.html

### Outils

- **pgAdmin** : Interface graphique avec éditeur pg_hba.conf intégré  
- **pg_hba_file_rules** : Vue système (PG 18+) pour inspecter les règles  
- **pg_hba.conf generator** : Outils en ligne pour générer des configurations

### Exemples de Configurations

Disponibles dans :
```
/usr/share/postgresql/18/pg_hba.conf.sample
```

---

## Conclusion

Le fichier **pg_hba.conf** est la pierre angulaire de la sécurité de PostgreSQL. Une configuration correcte est essentielle pour :

1. **Protéger vos données** contre les accès non autorisés  
2. **Permettre les connexions légitimes** sans friction  
3. **Auditer et tracer** les accès à la base de données  
4. **Se conformer** aux standards de sécurité (PCI-DSS, GDPR, etc.)

**Points clés à retenir :**

- 📝 **Structure** : TYPE - DATABASE - USER - ADDRESS - METHOD  
- 🔢 **Ordre** : Premier match gagne (règles spécifiques avant générales)  
- 🔄 **Rechargement** : `pg_reload_conf()` sans redémarrage  
- 🔒 **Sécurité** : SSL + SCRAM-SHA-256 en production  
- 🚫 **Rejection** : Toujours terminer par une règle de rejet

Dans les sections suivantes, nous explorerons en détail chaque **méthode d'authentification** (SCRAM, certificats, LDAP, OAuth) ainsi que les **nouveautés de PostgreSQL 18** qui renforcent encore davantage la sécurité de l'authentification.

---


⏭️ [Méthodes : trust, password, md5, scram-sha-256, cert, ldap](/16-administration-configuration-securite/02.1-methodes-authentification.md)
