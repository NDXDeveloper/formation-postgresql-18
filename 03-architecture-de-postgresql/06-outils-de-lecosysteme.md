🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.6. Outils de l'Écosystème : psql, pgAdmin, DBeaver et Connection Pooling

## 📋 Introduction

PostgreSQL dispose d'un **écosystème riche d'outils** pour interagir avec vos bases de données. Que vous soyez développeur, administrateur système (DBA), ou analyste de données, il existe un outil adapté à vos besoins.

Dans cette section, nous allons explorer :
- **psql** : L'interface en ligne de commande (CLI) officielle
- **pgAdmin** : L'interface graphique (GUI) officielle
- **DBeaver** : Un outil GUI multiplateforme très populaire
- **Connection Pooling** : Optimiser la gestion des connexions avec PgBouncer

Comprendre ces outils vous permettra de choisir le bon outil pour chaque tâche et d'optimiser votre workflow quotidien avec PostgreSQL.

---

## 🗺️ Vue d'Ensemble de l'Écosystème

### Les Différents Types d'Outils

```
ÉCOSYSTÈME POSTGRESQL
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │         OUTILS D'INTERACTION                       │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  CLI (Ligne de commande)                           │  │
│  │  - psql (officiel)                                 │  │
│  │  - pgcli (amélioré avec auto-complétion)           │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  GUI (Interface graphique)                         │  │
│  │  - pgAdmin 4 (officiel)                            │  │
│  │  - DBeaver (multiplateforme)                       │  │
│  │  - DataGrip (JetBrains, payant)                    │  │
│  │  - TablePlus (macOS/Windows)                       │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │         OUTILS D'OPTIMISATION                      │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  Connection Pooling                                │  │
│  │  - PgBouncer (léger, très performant)              │  │
│  │  - PgPool-II (plus de fonctionnalités)             │  │
│  │  - Odyssey (Yandex, moderne)                       │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │         OUTILS D'ADMINISTRATION                    │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  - pg_dump / pg_restore (backup/restore)           │  │
│  │  - pg_basebackup (backup physique)                 │  │
│  │  - Patroni (HA automatisé)                         │  │
│  │  - Barman (backup manager)                         │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Choisir le Bon Outil

| Tâche                          | Outil Recommandé           |
|--------------------------------|----------------------------|
| Développement quotidien        | DBeaver ou pgAdmin         |
| Scripts automatisés            | psql                       |
| Administration serveur         | psql + pgAdmin             |
| Exploration rapide de données  | DBeaver                    |
| Production haute charge        | PgBouncer (pooling)        |
| Debugging SQL                  | psql (EXPLAIN ANALYZE)     |

---

## 💻 psql : L'Interface en Ligne de Commande

### Qu'est-ce que psql ?

**psql** est le client en ligne de commande **officiel** de PostgreSQL. C'est un outil puissant et léger qui permet d'interagir avec PostgreSQL directement depuis le terminal.

### Pourquoi Utiliser psql ?

#### ✅ Avantages

- **Léger** : Aucune interface graphique, très rapide
- **Scriptable** : Parfait pour l'automatisation
- **Universellement disponible** : Installé avec PostgreSQL
- **Puissant** : Accès à toutes les fonctionnalités
- **SSH-friendly** : Fonctionne sur serveur distant
- **Meta-commandes** : Raccourcis pratiques (`\d`, `\l`, etc.)

#### ❌ Inconvénients

- Courbe d'apprentissage (syntaxe à mémoriser)
- Pas de visualisation graphique
- Moins intuitif pour les débutants

### Démarrer avec psql

#### Connexion Basique

```bash
# Connexion locale (utilisateur postgres)
psql -U postgres

# Connexion à une base spécifique
psql -U myuser -d mydatabase

# Connexion à un serveur distant
psql -h hostname -p 5432 -U myuser -d mydatabase

# Avec mot de passe (prompt)
psql -U myuser -d mydatabase -W
```

**Syntaxe Générale** :
```bash
psql -h HOST -p PORT -U USER -d DATABASE
```

#### Variables d'Environnement

Pour éviter de taper les mêmes paramètres :

```bash
# Dans ~/.bashrc ou ~/.zshrc
export PGHOST=localhost
export PGPORT=5432
export PGUSER=myuser
export PGDATABASE=mydatabase

# Puis simplement :
psql
```

### Interface psql

Quand vous lancez psql, vous voyez :

```
psql (18.0)
Type "help" for help.

mydatabase=#
```

**Prompt** :
- `mydatabase=#` : Connecté en tant que superuser (`#`)
- `mydatabase=>` : Connecté en tant qu'utilisateur normal (`>`)
- `mydatabase->` : Commande incomplète (attente de `;`)

---

### Meta-Commandes psql

Les **meta-commandes** commencent par `\` et sont des raccourcis très pratiques.

#### Navigation et Exploration

```sql
-- Lister les bases de données
\l
-- ou
\list

-- Se connecter à une autre base
\c mydatabase
-- ou
\connect mydatabase

-- Lister les tables
\dt

-- Lister les tables avec détails (taille)
\dt+

-- Décrire une table (structure)
\d users

-- Décrire une table avec détails
\d+ users

-- Lister les index
\di

-- Lister les vues
\dv

-- Lister les fonctions
\df

-- Lister les séquences
\ds

-- Lister les schémas
\dn
```

#### Affichage et Formatage

```sql
-- Activer l'affichage étendu (vertical)
\x

-- Exemple :
SELECT * FROM users LIMIT 1;

-- Sans \x (horizontal) :
 id | username | email
----+----------+------------------
  1 | alice    | alice@example.com

-- Avec \x (vertical) :
-[ RECORD 1 ]--+------------------
id             | 1
username       | alice
email          | alice@example.com

-- Activer le timing (temps d'exécution)
\timing

-- Exemple :
SELECT COUNT(*) FROM users;
Time: 45.123 ms

-- Changer le format de sortie
\pset format aligned  -- Défaut, aligné
\pset format html     -- Format HTML
\pset format csv      -- Format CSV
```

#### Import/Export

```sql
-- Exécuter un fichier SQL
\i /path/to/script.sql

-- Exporter les résultats dans un fichier
\o /path/to/output.txt
SELECT * FROM users;
\o  -- Revenir à stdout

-- Copier depuis/vers un fichier CSV
\copy users TO '/tmp/users.csv' CSV HEADER;
\copy users FROM '/tmp/users.csv' CSV HEADER;
```

#### Aide

```sql
-- Aide générale
\?

-- Aide sur une commande SQL
\h SELECT
\h CREATE TABLE
\h UPDATE

-- Voir les paramètres de configuration
\dconfig

-- Voir les permissions
\dp
```

#### Système

```sql
-- Exécuter une commande shell
\! ls -la

-- Voir l'historique des commandes
\s

-- Sauvegarder l'historique dans un fichier
\s history.sql

-- Quitter psql
\q
-- ou
\quit
-- ou Ctrl+D
```

---

### Exemples d'Utilisation Pratiques

#### Cas 1 : Explorer une Nouvelle Base

```sql
-- Se connecter
psql -U myuser -d newdb

-- Lister les tables
\dt

-- Voir la structure d'une table
\d users

-- Affichage vertical pour mieux voir
\x

-- Regarder quelques exemples de données
SELECT * FROM users LIMIT 3;

-- Compter les lignes
SELECT COUNT(*) FROM users;
```

#### Cas 2 : Exécuter un Script

```bash
# Exécuter un script SQL
psql -U myuser -d mydatabase -f migration.sql

# Avec sortie dans un fichier
psql -U myuser -d mydatabase -f migration.sql > output.log 2>&1
```

#### Cas 3 : Export de Données

```sql
-- Se connecter
psql -U myuser -d mydatabase

-- Export CSV
\copy (SELECT * FROM users WHERE active = true) TO '/tmp/active_users.csv' CSV HEADER;

-- Export formaté
\o /tmp/report.txt
\pset format aligned
SELECT category, COUNT(*), SUM(amount)
FROM orders
GROUP BY category;
\o
```

#### Cas 4 : Debugging de Requête

```sql
-- Activer timing
\timing

-- Activer EXPLAIN automatique
\set AUTOEXPLAIN on

-- Tester la requête
EXPLAIN ANALYZE
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01';

-- Temps affiché : Time: 234.567 ms
```

---

### Configuration de psql

#### Fichier .psqlrc

Vous pouvez personnaliser psql avec un fichier `~/.psqlrc` :

```sql
-- ~/.psqlrc

-- Activer timing par défaut
\timing

-- Prompt personnalisé avec base de données et hôte
\set PROMPT1 '%n@%/%R%# '

-- Historique persistant
\set HISTFILE ~/.psql_history
\set HISTSIZE 10000

-- Affichage null
\pset null '(null)'

-- Activer auto-complétion insensible à la casse
\set COMP_KEYWORD_CASE upper

-- Activer le pager pour longs résultats
\pset pager always
```

---

## 🖥️ pgAdmin 4 : L'Interface Graphique Officielle

### Qu'est-ce que pgAdmin ?

**pgAdmin 4** est l'interface graphique **officielle** pour PostgreSQL. C'est une application web qui peut être utilisée localement ou déployée sur un serveur.

### Caractéristiques Principales

#### ✅ Avantages

- **Officiel** : Maintenu par la communauté PostgreSQL
- **Complet** : Toutes les fonctionnalités PostgreSQL
- **Éditeur SQL** : Avec coloration syntaxique et auto-complétion
- **Visualisation** : Explorer les données facilement
- **Administration** : Gestion des utilisateurs, backups, monitoring
- **Gratuit et Open Source**

#### ❌ Inconvénients

- Interface parfois lourde
- Courbe d'apprentissage
- Consommation mémoire élevée (application Electron)

### Interface pgAdmin

```
┌──────────────────────────────────────────────────────────────┐
│  pgAdmin 4                                         [_][□][×] │
├──────────────────────────────────────────────────────────────┤
│ Fichier  Édition  Outils  Aide                               │
├───────────────┬──────────────────────────────────────────────┤
│               │                                              │
│  🗂️ Serveurs  │         QUERY EDITOR                         │
│   └─ Local   │                                               │
│      ├─ 📁 DB │  SELECT * FROM users                         │
│      │  ├─ 📊 │  WHERE created_at > '2024-01-01'             │
│      │  │ Tbl │  ORDER BY username;                         │
│      │  ├─ 🔧 │                                              │
│      │  │ Func│  [▶ Execute] [💾 Save] [📋 Format]           │
│      │  └─ 👁️ │                                              │
│      │    View│──────────────────────────────────────────────│
│               │  RESULTS                                     │
│               │  ┌────┬──────────┬───────────────────────┐   │
│               │  │ id │ username │ email                 │   │
│               │  ├────┼──────────┼───────────────────────┤   │
│               │  │  1 │ alice    │ alice@example.com     │   │
│               │  │  2 │ bob      │ bob@example.com       │   │
│               │  └────┴──────────┴───────────────────────┘   │
│               │                                              │
└───────────────┴──────────────────────────────────────────────┘
```

### Fonctionnalités Clés

#### 1. Éditeur de Requêtes (Query Tool)

- **Coloration syntaxique** : SQL coloré
- **Auto-complétion** : Suggère tables, colonnes, fonctions
- **EXPLAIN Visualisé** : Voir le plan d'exécution graphiquement
- **Historique** : Retrouver les requêtes précédentes
- **Export** : CSV, JSON, Excel

#### 2. Explorateur d'Objets

Arborescence pour explorer :
- Bases de données
- Schémas
- Tables (avec nombre de lignes)
- Colonnes (avec types)
- Index
- Contraintes
- Triggers
- Fonctions

#### 3. Visualisation de Données

- **Data Grid** : Voir et éditer les données
- **Filtrage** : Filtrer les lignes affichées
- **Tri** : Cliquer sur colonnes pour trier
- **Export** : Exporter les résultats

#### 4. Administration

- **Gestion des utilisateurs** : Créer/modifier rôles
- **Backup/Restore** : Interface graphique pour pg_dump/pg_restore
- **Monitoring** : Voir les processus actifs
- **Configuration** : Éditer postgresql.conf

#### 5. Dashboard

Tableau de bord avec métriques en temps réel :
- Nombre de connexions
- Transactions par seconde
- Cache hit ratio
- Taille des bases de données

---

### Utilisation Typique de pgAdmin

#### Cas 1 : Exploration Rapide

1. **Naviguer** : Cliquer dans l'arborescence jusqu'à la table
2. **Visualiser** : Clic droit → "View/Edit Data" → "All Rows"
3. **Filtrer** : Utiliser les filtres en haut de la grille
4. **Exporter** : Bouton "Download" → CSV

#### Cas 2 : Développement de Requêtes

1. **Ouvrir Query Tool** : Clic droit sur base → "Query Tool"
2. **Écrire requête** : Avec auto-complétion
3. **Exécuter** : F5 ou bouton "Execute"
4. **Analyser** : Bouton "EXPLAIN" pour voir le plan

#### Cas 3 : Administration

1. **Créer un utilisateur** :
   - Serveur → Login/Group Roles → Clic droit → "Create"
   - Remplir le formulaire (nom, mot de passe, permissions)

2. **Faire un backup** :
   - Clic droit sur base → "Backup"
   - Choisir format, emplacement
   - "Backup" → Télécharger

---

## 🦫 DBeaver : L'Outil Universel

### Qu'est-ce que DBeaver ?

**DBeaver** est un outil de gestion de bases de données **multiplateforme** et **multi-SGBD**. Contrairement à pgAdmin qui est spécifique à PostgreSQL, DBeaver supporte des dizaines de bases de données.

### Pourquoi DBeaver ?

#### ✅ Avantages

- **Multi-SGBD** : PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, MongoDB, etc.
- **Interface moderne** : UI élégante et intuitive
- **Performant** : Plus léger que pgAdmin
- **Auto-complétion intelligente** : Très bonne
- **Visualisation ER** : Diagrammes entité-relation automatiques
- **Git intégré** : Versionner vos scripts SQL
- **Edition de données** : Très pratique
- **Gratuit** : Version Community très complète

#### ❌ Inconvénients

- Moins de fonctionnalités admin PostgreSQL spécifiques que pgAdmin
- Version Pro payante pour features avancées

### Interface DBeaver

```
┌─────────────────────────────────────────────────────────────┐
│  DBeaver                                          [_][□][×] │
├─────────────────────────────────────────────────────────────┤
│ File  Edit  View  Database  SQL  Tools  Help                │
├───────────────┬─────────────────────────────────────────────┤
│               │  📝 SQL Editor - users.sql              [×] │
│ 📁 Database   │─────────────────────────────────────────────│
│  Navigator    │                                             │
│               │  SELECT u.username, COUNT(o.id) as orders   │
│ 🐘 PostgreSQL │  FROM users u                               │
│  └─ MyServer  │  LEFT JOIN orders o ON u.id = o.user_id     │
│     └─ mydb   │  GROUP BY u.username                        │
│        ├─ 📊  │  ORDER BY orders DESC;                      │
│        │ Tbl  │                                             │
│        │ ├─us │  [▶ Execute] [📊 Explain] [📁 Format]       │
│        │ ├─or │                                             │
│        │ └─pr │─────────────────────────────────────────────│
│        └─ 🔑  │  📊 Data                                    │
│          Keys │  ┌────────────┬─────────┐                   │
│               │  │ username   │ orders  │                   │
│               │  ├────────────┼─────────┤                   │
│               │  │ alice      │    45   │                   │
│               │  │ bob        │    23   │                   │
│               │  └────────────┴─────────┘                   │
└───────────────┴─────────────────────────────────────────────┘
```

### Fonctionnalités Phares

#### 1. Database Navigator

Arborescence intelligente :
- **Filtrage rapide** : Chercher tables/colonnes
- **Groupement** : Par schéma, par type
- **Statistiques** : Taille, nombre de lignes
- **Icônes colorées** : Distinguer visuellement

#### 2. Éditeur SQL Avancé

- **Auto-complétion contextuelle** : Très intelligente
- **Multi-onglets** : Plusieurs requêtes en parallèle
- **Exécution partielle** : Sélectionner et exécuter
- **Templates** : Snippets SQL réutilisables
- **Formatting** : Formatage automatique (Ctrl+Shift+F)

#### 3. Data Editor

- **Edition inline** : Modifier directement dans la grille
- **Filtres avancés** : Filtre par colonne, recherche
- **Dictionnaires** : Auto-complétion pour foreign keys
- **Import/Export** : CSV, JSON, XML, Excel

#### 4. ER Diagrams

- **Génération automatique** : Diagramme des relations
- **Visualisation** : Voir les foreign keys
- **Export** : Image PNG, PDF

#### 5. Git Integration

- **Version SQL** : Commit/Push vos scripts
- **Comparaison** : Diff entre versions
- **Historique** : Retrouver anciennes versions

---

### Utilisation Typique de DBeaver

#### Cas 1 : Connexion Multi-Bases

```
DBeaver peut gérer simultanément :
├─ PostgreSQL - Production
├─ PostgreSQL - Staging
├─ MySQL - Legacy System
└─ MongoDB - Analytics

→ Un seul outil pour tout !
```

#### Cas 2 : Développement SQL

1. **Ouvrir SQL Editor** : Fichier → New → SQL Script
2. **Écrire** : Auto-complétion intelligente
3. **Formatter** : Ctrl+Shift+F
4. **Exécuter** : Ctrl+Enter
5. **Exporter** : Résultats → Export → CSV

#### Cas 3 : Exploration de Schema

1. **Clic droit sur base** → "View Diagram"
2. **DBeaver génère** un diagramme ER automatique
3. **Voir** toutes les relations (FK)
4. **Exporter** en PNG pour documentation

---

## 🔗 Connection Pooling avec PgBouncer

### Le Problème : Trop de Connexions

Chaque connexion PostgreSQL consomme :
- **~10 MB de RAM minimum**
- **1 processus système**
- **Overhead de création** : ~50-100 ms

#### Exemple Problématique

```
APPLICATION WEB
100 utilisateurs simultanés
Chaque requête crée une connexion

100 connexions × 10 MB = 1 GB RAM ! 🔥
100 processus backend = Overhead CPU

Si 1000 utilisateurs → 10 GB RAM ! 💥
```

**Problème** : Saturation des ressources

---

### La Solution : Connection Pooling

Un **Connection Pooler** est un proxy entre les applications et PostgreSQL qui **mutualise les connexions**.

#### Analogie du Taxi Partagé 🚕

**Sans pooling** :
```
100 clients → 100 taxis → 100 destinations
(Coûteux, inefficace)
```

**Avec pooling** :
```
100 clients → 10 taxis partagés → 100 destinations
(Les clients attendent un taxi libre)
(Beaucoup plus efficace !)
```

---

### PgBouncer : Le Pooler de Référence

**PgBouncer** est le connection pooler le plus populaire pour PostgreSQL.

#### Caractéristiques

- **Léger** : Écrit en C, très performant (~1-2 MB RAM)
- **Rapide** : Overhead quasi-nul (<1% latence)
- **Modes de pooling** : Transaction, Session, Statement
- **Mature** : Utilisé massivement en production
- **Open Source** : Gratuit

### Architecture avec PgBouncer

```
AVANT (sans pooling)
┌──────────────┐                    ┌──────────────┐
│ APPLICATION  │                    │  POSTGRESQL  │
│              │                    │              │
│ 100 threads  ├──> 100 connexions ─┤  100 procs   │
│              │                    │              │
└──────────────┘                    └──────────────┘
                RAM: ~1 GB


APRÈS (avec PgBouncer)
┌──────────────┐    ┌────────────┐    ┌──────────────┐
│ APPLICATION  │    │ PGBOUNCER  │    │  POSTGRESQL  │
│              │    │            │    │              │
│ 100 threads  ├──> │    POOL    ├──> │   10 procs   │
│              │    │  (200 max) │    │              │
└──────────────┘    └────────────┘    └──────────────┘
                                       RAM: ~100 MB ✅

100 clients → 10 connexions réelles
GAIN: 10× moins de ressources !
```

---

### Modes de Pooling

PgBouncer propose 3 modes :

#### 1. Session Pooling (Mode Session)

```
CLIENT connecte → Obtient 1 connexion PostgreSQL dédiée
             → Garde cette connexion jusqu'à déconnexion
             → Connexion retourne au pool

Utilisation : Applications nécessitant des états de session
Exemples : SET variables, TEMP tables, PREPARE statements
```

**Avantages** : Compatible avec tout
**Inconvénients** : Moins de gains (ratio 1:1 pendant la session)

#### 2. Transaction Pooling (Mode Transaction) ⭐

```
CLIENT envoie requête → Obtient connexion pour la transaction
                     → COMMIT/ROLLBACK → Connexion libérée immédiatement
                     → Prochaine requête peut avoir une autre connexion

Utilisation : Applications stateless (web, API)
C'est le MODE RECOMMANDÉ pour la plupart des cas !
```

**Avantages** : Maximum d'efficacité (ratios 10:1 ou plus)
**Inconvénients** : Incompatible avec états de session

**Limitations** :
- ❌ Pas de `SET` persistant
- ❌ Pas de `PREPARE` réutilisable
- ❌ Pas de `TEMP tables`
- ❌ Pas de `LISTEN/NOTIFY`

#### 3. Statement Pooling (Mode Statement)

```
CLIENT envoie 1 requête → Connexion pour cette requête uniquement
                       → Résultat → Connexion libérée

Utilisation : Très rare, peu utilisé
```

**Avantages** : Maximum théorique
**Inconvénients** : Trop restrictif (même pas de transactions multi-statements)

---

### Configuration de PgBouncer

#### Fichier pgbouncer.ini

```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
# Mode de pooling (transaction = recommandé)
pool_mode = transaction

# Écouter sur ce port
listen_port = 6432
listen_addr = *

# Authentification
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Limites de connexions
max_client_conn = 1000        # Max connexions clients
default_pool_size = 20        # Connexions par base
reserve_pool_size = 5         # Connexions de réserve
reserve_pool_timeout = 3      # Timeout réserve (secondes)

# Timeouts
server_idle_timeout = 600     # Ferme connexion idle après 10 min
query_timeout = 0             # Pas de timeout requête (0 = illimité)

# Logs
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
```

#### Fichier userlist.txt

```
"myuser" "scram-sha-256:PASSWORDHASH"
```

---

### Connexion via PgBouncer

#### Application

Au lieu de se connecter directement à PostgreSQL :

```python
# AVANT (direct à PostgreSQL)
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    database="mydb",
    user="myuser",
    password="mypassword"
)

# APRÈS (via PgBouncer)
conn = psycopg2.connect(
    host="localhost",
    port=6432,  # ← Port PgBouncer !
    database="mydb",
    user="myuser",
    password="mypassword"
)
```

**Transparent** : L'application ne voit pas la différence !

---

### Monitoring de PgBouncer

#### Console PgBouncer

```bash
# Se connecter à la console admin PgBouncer
psql -h localhost -p 6432 -U pgbouncer pgbouncer

# Commandes utiles
SHOW POOLS;
SHOW CLIENTS;
SHOW SERVERS;
SHOW STATS;
```

#### SHOW POOLS

```
pgbouncer=# SHOW POOLS;

 database | user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used
----------+--------+-----------+------------+-----------+---------+---------
 mydb     | myuser |        45 |          0 |        18 |       2 |      20

Légende :
- cl_active : Clients actifs (requêtes en cours)
- cl_waiting : Clients en attente d'une connexion
- sv_active : Connexions PostgreSQL actives
- sv_idle : Connexions PostgreSQL idle (pool)
```

**Interprétation** :
- 45 clients actifs → 18 connexions PostgreSQL réelles
- Ratio : 45:18 = 2.5:1 → Bon pooling !

#### SHOW CLIENTS

```
pgbouncer=# SHOW CLIENTS;

 type | user   | database | state  | addr          | port
------+--------+----------+--------+---------------+------
 C    | myuser | mydb     | active | 192.168.1.10  | 54321
 C    | myuser | mydb     | idle   | 192.168.1.11  | 54322
```

#### SHOW STATS

```
pgbouncer=# SHOW STATS;

 database | total_requests | total_received | total_sent | avg_req_time
----------+----------------+----------------+------------+--------------
 mydb     |       1234567  |      123456 MB |  98765 MB  |      15 ms
```

---

### Quand Utiliser PgBouncer ?

#### ✅ Cas d'Usage Favorables

1. **Applications Web/API** (stateless)
   - Beaucoup de connexions courtes
   - Gain : 5-10× moins de ressources

2. **Microservices**
   - Chaque service a son pool de connexions
   - PgBouncer centralise

3. **Serverless (Lambda, Cloud Functions)**
   - Connexions éphémères
   - Essentiel pour éviter la saturation

4. **Haute concurrence**
   - 100+ utilisateurs simultanés
   - PostgreSQL limité à ~200-500 connexions max

#### ⚠️ Cas où PgBouncer Peut Poser Problème

1. **Applications avec états de session**
   - TEMP tables
   - PREPARE statements
   - SET variables

   **Solution** : Utiliser `pool_mode = session` (moins de gains)

2. **LISTEN/NOTIFY**
   - Pas supporté en mode transaction

3. **Long-running queries**
   - Bloque une connexion du pool

---

## 📊 Comparaison des Outils

### psql vs pgAdmin vs DBeaver

| Critère                | psql          | pgAdmin 4      | DBeaver         |
|------------------------|---------------|----------------|-----------------|
| **Type**               | CLI           | GUI Web        | GUI Desktop     |
| **Performance**        | ⭐⭐⭐⭐⭐       | ⭐⭐⭐           | ⭐⭐⭐⭐          |
| **Courbe apprentissage**| Moyenne      | Élevée         | Faible          |
| **Scriptable**         | ✅ Excellent   | ❌             | ⚠️ Limité       |
| **Multi-SGBD**         | ❌            | ❌             | ✅ Excellent    |
| **Visualisation**      | ❌            | ⭐⭐⭐⭐         | ⭐⭐⭐⭐⭐        |
| **Admin PostgreSQL**   | ⭐⭐⭐⭐⭐       | ⭐⭐⭐⭐⭐        | ⭐⭐⭐           |
| **SSH/Distant**        | ✅ Parfait     | ⚠️ Web         | ✅ Bon          |
| **Export/Import**      | ⭐⭐⭐⭐        | ⭐⭐⭐⭐         | ⭐⭐⭐⭐⭐        |
| **Gratuit**            | ✅            | ✅             | ✅ (Community)  |

### Recommandations par Profil

#### Développeur Backend

```
Quotidien : DBeaver (confort, multi-DB)
Scripts : psql (automatisation)
Debugging : psql + EXPLAIN ANALYZE
```

#### Développeur Full-Stack

```
Quotidien : DBeaver (intuitif, rapide)
Exploration : pgAdmin (features PostgreSQL complètes)
```

#### DBA (Administrateur)

```
Administration : pgAdmin + psql
Monitoring : psql + scripts
Backup/Restore : psql (pg_dump, pg_restore)
```

#### Data Analyst

```
Exploration : DBeaver (ER diagrams, export Excel)
Requêtes : DBeaver (auto-complétion, visualisation)
```

#### DevOps/SRE

```
Automatisation : psql (scripts, CI/CD)
Debugging production : psql (SSH)
Monitoring : Grafana + pg_stat_statements
```

---

## 🎯 Bonnes Pratiques

### Utilisation des Outils

#### 1. Développement Local

```
Configuration recommandée :
├─ DBeaver : Interface quotidienne
├─ psql : Scripts et tests rapides
└─ PgBouncer : Si simulation production
```

#### 2. Production

```
Configuration recommandée :
├─ PgBouncer : OBLIGATOIRE pour haute charge
├─ psql : Administration via SSH
├─ pgAdmin : Interface admin (si nécessaire)
└─ Monitoring : Prometheus + Grafana
```

#### 3. CI/CD

```bash
# Utiliser psql pour migrations automatiques
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -f migrations/001_create_users.sql
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -f migrations/002_add_indexes.sql
```

### Sécurité

#### psql

```bash
# Ne JAMAIS mettre le mot de passe en ligne de commande
# ❌ MAUVAIS :
psql -U myuser -p mypassword  # Visible dans l'historique !

# ✅ BON :
# Utiliser .pgpass
echo "localhost:5432:mydb:myuser:mypassword" >> ~/.pgpass
chmod 600 ~/.pgpass
psql -U myuser -d mydb

# Ou variable d'environnement (scripts)
export PGPASSWORD=mypassword
psql -U myuser -d mydb
```

#### PgBouncer

```ini
# Toujours utiliser SCRAM-SHA-256 (pas MD5 !)
auth_type = scram-sha-256

# Limiter les connexions clients
max_client_conn = 1000

# Firewall : Ouvrir port 6432 uniquement pour serveurs d'application
# iptables -A INPUT -p tcp --dport 6432 -s 192.168.1.0/24 -j ACCEPT
```

---

## 📝 Résumé des Concepts Clés

### Outils d'Interaction

| Outil      | Type | Force Principale              |
|------------|------|-------------------------------|
| **psql**   | CLI  | Scriptable, léger, universel  |
| **pgAdmin**| GUI  | Admin complète, officiel      |
| **DBeaver**| GUI  | Multi-DB, moderne, intuitif   |

### Connection Pooling

- ✅ **PgBouncer** : Proxy mutualisant les connexions
- ✅ **Transaction Mode** : Mode recommandé (stateless)
- ✅ **Gains** : 5-10× moins de ressources PostgreSQL
- ✅ **Transparent** : Applications ne voient pas la différence

### Modes de Pooling

| Mode          | Connexion Réutilisée       | Cas d'Usage                |
|---------------|----------------------------|----------------------------|
| **Session**   | Pendant toute la session   | Applications avec état     |
| **Transaction** | À chaque transaction     | Applications stateless ⭐  |
| **Statement** | À chaque requête           | Rare, très restrictif      |

---

## 🎓 Points à Retenir pour les Débutants

1. **psql est votre ami** : Apprendre les meta-commandes (`\d`, `\dt`, `\x`) vous fera gagner beaucoup de temps.

2. **DBeaver pour commencer** : Interface intuitive, parfait pour débuter et explorer.

3. **pgAdmin pour l'administration** : Toutes les features PostgreSQL, interface d'admin complète.

4. **PgBouncer est essentiel en production** : Ne pas l'utiliser = saturation garantie avec plus de 50 utilisateurs.

5. **Transaction mode pour le pooling** : C'est le mode recommandé pour 95% des applications web/API.

6. **Un outil par tâche** :
   - Développement → DBeaver
   - Scripts → psql
   - Admin → pgAdmin
   - Production → PgBouncer

7. **Automatiser avec psql** : Pour CI/CD, migrations, backups → psql est imbattable.

8. **Monitorer PgBouncer** : `SHOW POOLS` pour voir l'efficacité du pooling.

---

## 🔗 Prochaine Étape

Maintenant que vous connaissez les **outils de l'écosystème PostgreSQL**, la prochaine partie de la formation abordera **le langage SQL** : comment interroger et manipuler les données avec les requêtes de sélection (DQL) et de manipulation (DML).

---

**Note** : Ce tutoriel couvre les outils principaux, mais l'écosystème PostgreSQL est riche. Explorez également : pgcli (psql amélioré), TablePlus, DataGrip, Postico, et bien d'autres selon vos préférences !

⏭️ [Les Objets de la Base de Données (DDL)](/04-objets-de-la-base-de-donnees/README.md)
