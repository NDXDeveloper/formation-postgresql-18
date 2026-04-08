🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Commandes psql Essentielles - Navigation

## Introduction

`psql` est l'interface en ligne de commande (CLI) officielle de PostgreSQL. C'est un outil puissant qui permet d'interagir avec vos bases de données directement depuis le terminal. Ce guide vous présente toutes les commandes de navigation essentielles pour explorer et comprendre la structure de vos bases de données.

### Qu'est-ce qu'une méta-commande ?

Dans `psql`, les commandes qui commencent par un backslash `\` sont appelées **méta-commandes** (ou commandes internes). Elles ne sont pas du SQL, mais des commandes spécifiques à `psql` pour faciliter votre travail.

**Différence importante** :
```sql
-- Ceci est du SQL (se termine par un point-virgule)
SELECT * FROM clients;

-- Ceci est une méta-commande (pas de point-virgule)
\dt
```

### Convention de notation

Dans ce guide :
- `\commande` : La commande de base  
- `\commande+` : Version détaillée (avec plus d'informations)  
- `\commande PATTERN` : Commande avec un filtre/motif optionnel  
- `[...]` : Paramètre optionnel
- Toutes les commandes sont **insensibles à la casse** mais traditionnellement écrites en minuscules

---

## Navigation de Base

### Se Connecter à une Base de Données

#### \c ou \connect - Changer de base de données

**Syntaxe** :
```
\c[onnect] [DBNAME [USERNAME] [HOST] [PORT]]
```

**Description** : Change la connexion vers une autre base de données, utilisateur, hôte ou port.

**Exemples** :

```bash
# Se connecter à la base "production"
\c production

# Se connecter avec un utilisateur spécifique
\c production admin

# Se connecter à un serveur distant
\c production admin db.example.com 5432

# Afficher la base actuelle (sans arguments)
\c
# Résultat : You are now connected to database "production" as user "admin".
```

**Astuce** 💡 : Utilisez `\conninfo` pour voir les détails de votre connexion actuelle.

```bash
\conninfo
# Résultat : You are connected to database "production" as user "admin"
#            on host "localhost" at port "5432".
```

---

## Explorer les Bases de Données

### \l ou \list - Lister toutes les bases de données

**Syntaxe** :
```
\l[ist][+] [PATTERN]
```

**Description** : Affiche la liste de toutes les bases de données disponibles sur le serveur PostgreSQL.

**Exemples** :

```bash
# Lister toutes les bases
\l

# Sortie typique :
#                                   List of databases
#      Name      |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
# ---------------+----------+----------+-------------+-------------+-----------------------
#  ma_boutique   | postgres | UTF8     | fr_FR.UTF-8 | fr_FR.UTF-8 |
#  postgres      | postgres | UTF8     | fr_FR.UTF-8 | fr_FR.UTF-8 |
#  template0     | postgres | UTF8     | fr_FR.UTF-8 | fr_FR.UTF-8 | =c/postgres          +
#                |          |          |             |             | postgres=CTc/postgres
#  template1     | postgres | UTF8     | fr_FR.UTF-8 | fr_FR.UTF-8 | =c/postgres          +
#                |          |          |             |             | postgres=CTc/postgres
#  test          | postgres | UTF8     | fr_FR.UTF-8 | fr_FR.UTF-8 |
```

**Version détaillée** :
```bash
\l+

# Ajoute des colonnes : Size, Tablespace, Description
```

**Avec filtre** :
```bash
# Lister uniquement les bases commençant par "test"
\l test*

# Lister uniquement "ma_boutique"
\l ma_boutique
```

**Comprendre la sortie** :
- **Name** : Nom de la base  
- **Owner** : Propriétaire (utilisateur qui a créé la base)  
- **Encoding** : Encodage des caractères (UTF8 recommandé)  
- **Collate** : Règles de tri  
- **Ctype** : Règles de classification des caractères  
- **Access privileges** : Permissions (vide = propriétaire a tous les droits)  
- **Size** (avec `+`) : Taille sur disque

**Bases système** :
- `postgres` : Base par défaut (ne pas supprimer)  
- `template0` : Modèle propre (ne jamais modifier)  
- `template1` : Modèle personnalisable
- Les autres sont vos bases utilisateur

---

## Explorer les Schémas

### \dn - Lister les schémas

**Syntaxe** :
```
\dn[+] [PATTERN]
```

**Description** : Affiche tous les schémas (namespaces) de la base de données actuelle.

**Exemples** :

```bash
# Lister tous les schémas
\dn

# Sortie typique :
#   List of schemas
#   Name   |  Owner
# ---------+----------
#  public  | postgres
#  ventes  | admin
#  rh      | admin
```

**Version détaillée** :
```bash
\dn+

# Ajoute les colonnes : Access privileges, Description
```

**Avec filtre** :
```bash
# Uniquement les schémas commençant par "prod"
\dn prod*
```

**Rappel** 📖 : Un schéma est comme un dossier qui organise vos tables, vues et autres objets.

---

## Explorer les Tables

### \dt - Lister les tables

**Syntaxe** :
```
\dt[+] [SCHEMA.]PATTERN
```

**Description** : Affiche toutes les tables de la base actuelle (par défaut dans le schéma `public`).

**Exemples** :

```bash
# Lister toutes les tables
\dt

# Sortie typique :
#            List of relations
#  Schema |   Name    | Type  |  Owner
# --------+-----------+-------+----------
#  public | clients   | table | postgres
#  public | commandes | table | postgres
#  public | produits  | table | postgres
```

**Version détaillée** :
```bash
\dt+

# Ajoute : Size, Description
# Sortie :
#  Schema |   Name    | Type  |  Owner   | Size  | Description
# --------+-----------+-------+----------+-------+-------------
#  public | clients   | table | postgres | 64 kB |
#  public | commandes | table | postgres | 1024 MB | Table des commandes
```

**Tables d'un schéma spécifique** :
```bash
# Tables du schéma "ventes"
\dt ventes.*

# Tables du schéma "public" explicitement
\dt public.*
```

**Avec filtre** :
```bash
# Tables commençant par "client"
\dt client*

# Tables contenant "commande"
\dt *commande*
```

**Tous les schémas** :
```bash
# Lister les tables de TOUS les schémas
\dt *.*
```

**Astuce** 💡 : La colonne "Size" avec `\dt+` est très utile pour identifier les grosses tables.

---

### \d - Décrire un objet

**Syntaxe** :
```
\d[+] [SCHEMA.]NAME
```

**Description** : Affiche la structure détaillée d'une table, vue, index, séquence ou autre objet.

**Exemples** :

```bash
# Décrire la table "clients"
\d clients

# Sortie typique :
#                                      Table "public.clients"
#  Column  |          Type          | Collation | Nullable |               Default
# ---------+------------------------+-----------+----------+--------------------------------------
#  id      | integer                |           | not null | nextval('clients_id_seq'::regclass)
#  nom     | character varying(100) |           | not null |
#  email   | character varying(255) |           |          |
#  ville   | character varying(50)  |           |          |
#  created | timestamp              |           |          | now()
# Indexes:
#     "clients_pkey" PRIMARY KEY, btree (id)
#     "clients_email_key" UNIQUE CONSTRAINT, btree (email)
#     "idx_clients_ville" btree (ville)
# Referenced by:
#     TABLE "commandes" CONSTRAINT "commandes_client_id_fkey" FOREIGN KEY (client_id) REFERENCES clients(id)
```

**Version détaillée** :
```bash
\d+ clients

# Ajoute : Storage, Stats target, Description
# Et affiche plus d'informations sur les contraintes
```

**Comprendre la sortie** :
- **Column** : Nom de la colonne  
- **Type** : Type de données  
- **Nullable** : Accepte NULL ? (not null = obligatoire)  
- **Default** : Valeur par défaut  
- **Indexes** : Tous les index sur cette table  
- **Referenced by** : Tables qui ont des FK vers celle-ci  
- **Foreign-key constraints** : FK de cette table vers d'autres

**Autres objets** :
```bash
# Décrire une vue
\d ma_vue

# Décrire un index
\d idx_clients_email

# Décrire une séquence
\d clients_id_seq
```

**Astuce** 💡 : `\d` sans argument liste TOUS les objets (tables, vues, séquences, index...)

---

## Explorer les Index

### \di - Lister les index

**Syntaxe** :
```
\di[+] [SCHEMA.]PATTERN
```

**Description** : Affiche tous les index de la base actuelle.

**Exemples** :

```bash
# Lister tous les index
\di

# Sortie typique :
#                       List of relations
#  Schema |        Name         | Type  |  Owner   |   Table
# --------+---------------------+-------+----------+------------
#  public | clients_email_key   | index | postgres | clients
#  public | clients_pkey        | index | postgres | clients
#  public | idx_clients_ville   | index | postgres | clients
#  public | commandes_pkey      | index | postgres | commandes
#  public | idx_commandes_date  | index | postgres | commandes
```

**Version détaillée** :
```bash
\di+

# Ajoute : Size, Description, Method (btree, gin, gist...)
#  Schema |        Name         | Type  |  Owner   |   Table    |  Size  | Description | Method
# --------+---------------------+-------+----------+------------+--------+-------------+--------
#  public | idx_clients_ville   | index | postgres | clients    | 64 kB  |             | btree
#  public | idx_data_gin        | index | postgres | produits   | 256 kB |             | gin
```

**Index d'une table spécifique** :
```bash
# Méthode 1 : Via \d
\d clients
# Affiche les index dans la section "Indexes:"

# Méthode 2 : Filtre sur le nom de la table
\di *clients*
```

**Astuce** 💡 : La colonne "Size" avec `\di+` aide à identifier les index volumineux.

---

## Explorer les Vues

### \dv - Lister les vues

**Syntaxe** :
```
\dv[+] [SCHEMA.]PATTERN
```

**Description** : Affiche toutes les vues de la base actuelle.

**Exemples** :

```bash
# Lister toutes les vues
\dv

# Sortie typique :
#            List of relations
#  Schema |      Name       | Type |  Owner
# --------+-----------------+------+----------
#  public | ventes_2024     | view | postgres
#  public | clients_actifs  | view | postgres
```

**Version détaillée** :
```bash
\dv+

# Ajoute : Size (généralement 0 pour une vue simple), Description
```

**Voir la définition d'une vue** :
```bash
# Méthode 1 : Utiliser \d
\d ventes_2024

# Méthode 2 : Utiliser \d+ (plus détaillé)
\d+ ventes_2024
# Affiche le code SQL de la vue
```

---

### \dm - Lister les vues matérialisées

**Syntaxe** :
```
\dm[+] [SCHEMA.]PATTERN
```

**Description** : Affiche toutes les vues matérialisées.

**Exemples** :

```bash
# Lister toutes les vues matérialisées
\dm

# Sortie typique :
#                   List of relations
#  Schema |        Name         | Type |  Owner
# --------+---------------------+------+----------
#  public | stats_mensuelles    | mat. view | postgres
#  public | rapports_clients    | mat. view | postgres
```

**Version détaillée** :
```bash
\dm+

# Affiche la taille sur disque (importante pour les MV)
```

**Rappel** 📖 : Une vue matérialisée stocke physiquement les données (contrairement à une vue normale qui est juste une requête).

---

## Explorer les Séquences

### \ds - Lister les séquences

**Syntaxe** :
```
\ds[+] [SCHEMA.]PATTERN
```

**Description** : Affiche toutes les séquences (générateurs de nombres auto-incrémentés).

**Exemples** :

```bash
# Lister toutes les séquences
\ds

# Sortie typique :
#               List of relations
#  Schema |       Name        |   Type   |  Owner
# --------+-------------------+----------+----------
#  public | clients_id_seq    | sequence | postgres
#  public | commandes_id_seq  | sequence | postgres
#  public | produits_id_seq   | sequence | postgres
```

**Voir les détails d'une séquence** :
```bash
\d clients_id_seq

# Sortie :
#                  Sequence "public.clients_id_seq"
#   Type   | Start | Minimum |  Maximum   | Increment | Cycles? | Cache
# ---------+-------+---------+------------+-----------+---------+-------
#  integer |     1 |       1 | 2147483647 |         1 | no      |     1
# Owned by: public.clients.id
```

**Valeur actuelle d'une séquence** :
```sql
-- Ceci est du SQL, pas une méta-commande
SELECT currval('clients_id_seq');  
SELECT last_value FROM clients_id_seq;  
```

---

## Explorer les Fonctions et Procédures

### \df - Lister les fonctions

**Syntaxe** :
```
\df[+] [SCHEMA.]PATTERN
```

**Description** : Affiche toutes les fonctions de la base actuelle.

**Exemples** :

```bash
# Lister toutes les fonctions
\df

# Sortie typique :
#                              List of functions
#  Schema |      Name       | Result data type |    Argument data types     | Type
# --------+-----------------+------------------+----------------------------+------
#  public | calculer_total  | numeric          | commande_id integer        | func
#  public | valider_email   | boolean          | email text                 | func
```

**Version détaillée** :
```bash
\df+

# Ajoute : Volatility, Parallel, Owner, Security, Language, Description
```

**Voir le code d'une fonction** :
```bash
\df+ calculer_total

# Ou plus complet :
\sf calculer_total
# Affiche le code source de la fonction
```

**Filtres utiles** :
```bash
# Fonctions commençant par "calc"
\df calc*

# Fonctions d'un schéma spécifique
\df mon_schema.*
```

---

### \dp ou \z - Afficher les privilèges

**Syntaxe** :
```
\dp[+] [SCHEMA.]PATTERN
\z[+] [SCHEMA.]PATTERN
```

**Description** : Affiche les permissions (Access Control List - ACL) sur les tables et autres objets.

**Exemples** :

```bash
# Lister les privilèges sur toutes les tables
\dp

# Sortie typique :
#                                Access privileges
#  Schema |   Name    | Type  |     Access privileges     | Column privileges | Policies
# --------+-----------+-------+---------------------------+-------------------+----------
#  public | clients   | table | postgres=arwdDxt/postgres+|                   |
#         |           |       | alice=r/postgres          |                   |
#  public | commandes | table | postgres=arwdDxt/postgres |                   |
```

**Comprendre les codes de privilèges** :
- `r` = SELECT (read)  
- `w` = UPDATE (write)  
- `a` = INSERT (append)  
- `d` = DELETE  
- `D` = TRUNCATE  
- `x` = REFERENCES  
- `t` = TRIGGER  
- `=` séparateur entre utilisateur et droits  
- `/` séparateur avant le grantor (qui a donné les droits)

**Exemple d'interprétation** :
```
alice=r/postgres
└─────┬─────┘ │ └──────┬──────┘
   utilisateur │    qui a donné
           droit (r = SELECT)

Signifie : "alice a le droit SELECT, accordé par postgres"
```

**Privilèges sur un objet spécifique** :
```bash
\dp clients
```

---

## Explorer les Utilisateurs et Rôles

### \du - Lister les rôles/utilisateurs

**Syntaxe** :
```
\du[+] [PATTERN]
```

**Description** : Affiche tous les rôles et utilisateurs PostgreSQL.

**Exemples** :

```bash
# Lister tous les rôles
\du

# Sortie typique :
#                                    List of roles
#  Role name |                         Attributes                         | Member of
# -----------+------------------------------------------------------------+-----------
#  admin     | Superuser                                                  | {}
#  alice     |                                                            | {lecteur}
#  bob       | Create DB                                                  | {editeur}
#  postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

**Version détaillée** :
```bash
\du+

# Ajoute : Description
```

**Comprendre les attributs** :
- **Superuser** : Tous les droits (attention !)  
- **Create role** : Peut créer d'autres rôles  
- **Create DB** : Peut créer des bases de données  
- **Replication** : Peut initier la réplication  
- **Bypass RLS** : Ignore la sécurité niveau ligne (Row-Level Security)  
- **Member of** : Groupes/rôles dont cet utilisateur hérite

**Filtrer par nom** :
```bash
\du alice
\du admin*
```

---

## Explorer les Types de Données

### \dT - Lister les types

**Syntaxe** :
```
\dT[+] [SCHEMA.]PATTERN
```

**Description** : Affiche tous les types de données (y compris les types personnalisés).

**Exemples** :

```bash
# Lister tous les types
\dT

# Sortie typique :
#                          List of data types
#  Schema |        Name         |      Description
# --------+---------------------+------------------------
#  public | email_address       |
#  public | statut_commande     |
#  public | telephone           |
```

**Version détaillée** :
```bash
\dT+

# Ajoute plus d'informations selon le type
```

**Voir la définition d'un type** :
```bash
\dT+ statut_commande

# Pour un ENUM, affiche les valeurs
\dT+ statut_commande

#     List of data types
#  Schema |      Name       | Internal name | Size  | Elements | Owner    | Access privileges | Description
# --------+-----------------+---------------+-------+----------+----------+-------------------+-------------
#  public | statut_commande | statut_commande| 4    | en_attente+| postgres |                  |
#         |                 |               |       | expediee  +|          |                  |
#         |                 |               |       | livree    +|          |                  |
#         |                 |               |       | annulee   |          |                  |
```

**Filtres utiles** :
```bash
# Types commençant par "statut"
\dT statut*
```

---

## Explorer les Extensions

### \dx - Lister les extensions

**Syntaxe** :
```
\dx[+] [PATTERN]
```

**Description** : Affiche toutes les extensions installées dans la base actuelle.

**Exemples** :

```bash
# Lister toutes les extensions
\dx

# Sortie typique :
#                              List of installed extensions
#       Name       | Version |   Schema   |                   Description
# -----------------+---------+------------+--------------------------------------------------
#  pg_stat_statements | 1.10    | public     | track planning and execution statistics of all SQL
#  plpgsql         | 1.0     | pg_catalog | PL/pgSQL procedural language
#  uuid-ossp       | 1.1     | public     | generate universally unique identifiers (UUIDs)
```

**Version détaillée** :
```bash
\dx+

# Affiche tous les objets fournis par chaque extension
\dx+ pg_stat_statements

# Sortie :
# Objects in extension "pg_stat_statements"
#            Object description
# -----------------------------------------
#  function pg_stat_statements(boolean)
#  function pg_stat_statements_reset()
#  view pg_stat_statements
```

**Astuce** 💡 : Utilisez `\dx` régulièrement pour voir quelles extensions sont disponibles dans votre base.

---

## Explorer les Tablespaces

### \db - Lister les tablespaces

**Syntaxe** :
```
\db[+] [PATTERN]
```

**Description** : Affiche tous les tablespaces (emplacements de stockage).

**Exemples** :

```bash
# Lister tous les tablespaces
\db

# Sortie typique :
#        List of tablespaces
#     Name    |  Owner   | Location
# ------------+----------+----------
#  pg_default | postgres |
#  pg_global  | postgres |
#  fast_ssd   | postgres | /mnt/ssd/postgresql
```

**Version détaillée** :
```bash
\db+

# Ajoute : Access privileges, Options, Size, Description
```

**Rappel** 📖 : Un tablespace définit où PostgreSQL stocke physiquement les données sur le disque.

---

## Commandes de Recherche Globale

### \d sans argument - Lister tous les objets

**Syntaxe** :
```
\d[+]
```

**Description** : Affiche TOUS les types d'objets (tables, vues, séquences, index...).

**Exemples** :

```bash
# Lister tout
\d

# Sortie typique :
#                   List of relations
#  Schema |        Name         |   Type   |  Owner
# --------+---------------------+----------+----------
#  public | clients             | table    | postgres
#  public | clients_id_seq      | sequence | postgres
#  public | clients_pkey        | index    | postgres
#  public | commandes           | table    | postgres
#  public | ventes_2024         | view     | postgres
#  public | stats_mensuelles    | mat. view| postgres
```

**Astuce** 💡 : C'est souvent le point de départ pour explorer une base inconnue !

---

### Wildcards (Caractères jokers)

**Syntaxe des patterns** :

- `*` : N'importe quelle séquence de caractères  
- `?` : Un caractère quelconque  
- `[abc]` : Un caractère parmi a, b ou c

**Exemples** :

```bash
# Tables commençant par "client"
\dt client*

# Tables finissant par "log"
\dt *log

# Tables contenant "temp"
\dt *temp*

# Tables avec un pattern complexe
\dt client_[0-9]*
```

---

## Commandes de Recherche Avancées

### \dE - Lister les tables externes (Foreign Tables)

**Syntaxe** :
```
\dE[+] [SCHEMA.]PATTERN
```

**Description** : Affiche les tables étrangères (Foreign Data Wrappers).

**Exemples** :

```bash
# Lister toutes les tables externes
\dE

# Sortie typique :
#                List of relations
#  Schema |      Name       | Type  |  Owner   | Server
# --------+-----------------+-------+----------+---------
#  public | clients_distants| foreign table | postgres | serveur_externe
```

---

### \des - Lister les serveurs externes

**Syntaxe** :
```
\des[+] [PATTERN]
```

**Description** : Affiche les serveurs FDW configurés.

**Exemples** :

```bash
# Lister tous les serveurs externes
\des

# Sortie typique :
#            List of foreign servers
#      Name       |  Owner   | Foreign-data wrapper
# ----------------+----------+----------------------
#  serveur_mysql  | postgres | mysql_fdw
#  serveur_oracle | postgres | oracle_fdw
```

---

### \dew - Lister les wrappers FDW

**Syntaxe** :
```
\dew[+] [PATTERN]
```

**Description** : Affiche les Foreign Data Wrappers disponibles.

**Exemples** :

```bash
# Lister tous les FDW
\dew

# Sortie typique :
#          List of foreign-data wrappers
#     Name     |  Owner   |       Handler       |      Validator
# -------------+----------+---------------------+----------------------
#  file_fdw    | postgres | file_fdw_handler    | file_fdw_validator
#  postgres_fdw| postgres | postgres_fdw_handler| postgres_fdw_validator
```

---

### \dF - Lister les configurations de recherche plein texte

**Syntaxe** :
```
\dF[+] [PATTERN]
```

**Description** : Affiche les configurations Full-Text Search.

**Exemples** :

```bash
# Lister les configurations FTS
\dF

# Sortie typique :
#             List of text search configurations
#  Schema |    Name    |              Description
# --------+------------+---------------------------------------
#  public | english    | configuration for english language
#  public | french     | configuration for french language
#  public | simple     | simple configuration
```

---

### \do - Lister les opérateurs

**Syntaxe** :
```
\do[+] [SCHEMA.]PATTERN
```

**Description** : Affiche les opérateurs définis.

**Exemples** :

```bash
# Lister tous les opérateurs
\do

# Rechercher les opérateurs JSONB
\do @>
\do ?

# Sortie partielle :
#                            List of operators
#  Schema | Name | Left arg type | Right arg type | Result type |    Function
# --------+------+---------------+----------------+-------------+-----------------
#  public | @>   | jsonb         | jsonb          | boolean     | jsonb_contains
#  public | ?    | jsonb         | text           | boolean     | jsonb_exists
```

---

### \da - Lister les agrégats

**Syntaxe** :
```
\da[+] [SCHEMA.]PATTERN
```

**Description** : Affiche les fonctions d'agrégation.

**Exemples** :

```bash
# Lister tous les agrégats
\da

# Sortie partielle :
#                            List of aggregate functions
#  Schema | Name  | Result data type | Argument data types |     Description
# --------+-------+------------------+---------------------+---------------------
#  public | avg   | numeric          | numeric             |
#  public | count | bigint           | "any"               |
#  public | max   | same as input    | VARIADIC "any"      |
#  public | min   | same as input    | VARIADIC "any"      |
#  public | sum   | same as input    | numeric             |
```

---

### \dA - Lister les méthodes d'accès (Access Methods)

**Syntaxe** :
```
\dA[+] [PATTERN]
```

**Description** : Affiche les méthodes d'accès aux index (btree, gin, gist, etc.).

**Exemples** :

```bash
# Lister toutes les méthodes d'accès
\dA

# Sortie typique :
#          List of access methods
#  Name  | Type  |      Handler
# -------+-------+-------------------
#  brin  | Index | brinhandler
#  btree | Index | bthandler
#  gin   | Index | ginhandler
#  gist  | Index | gisthandler
#  hash  | Index | hashhandler
#  spgist| Index | spghandler
```

---

## Navigation Entre Schémas

### Comprendre le Search Path

Le `search_path` détermine dans quels schémas PostgreSQL cherche les objets.

**Voir le search_path actuel** :
```sql
SHOW search_path;
-- Résultat typique : "$user", public
```

**Ou via méta-commande** :
```bash
\drds
# Affiche tous les paramètres, y compris search_path
```

**Modifier temporairement** :
```sql
SET search_path TO ventes, public;
```

**Maintenant, les commandes cherchent d'abord dans "ventes"** :
```bash
# Ceci cherche d'abord dans "ventes", puis "public"
\dt

# Pour forcer un schéma spécifique
\dt public.*
\dt ventes.*
```

---

## Astuces de Navigation

### Combiner les Commandes

**Exemple 1 : Explorer une nouvelle base** :
```bash
# 1. Se connecter
\c nouvelle_base

# 2. Voir les schémas
\dn

# 3. Voir toutes les tables
\dt

# 4. Voir une table spécifique
\d clients

# 5. Voir les index de cette table (déjà dans \d)
# Ou lister tous les index
\di
```

**Exemple 2 : Audit de sécurité** :
```bash
# 1. Lister les utilisateurs
\du

# 2. Voir les permissions sur les tables
\dp

# 3. Voir les permissions sur une table spécifique
\dp clients
```

**Exemple 3 : Analyse de performance** :
```bash
# 1. Voir les tables et leur taille
\dt+

# 2. Voir les index et leur taille
\di+

# 3. Voir les vues matérialisées (peuvent être grosses)
\dm+
```

---

### Utiliser grep pour Filtrer

Vous pouvez combiner les méta-commandes avec `grep` dans le shell :

```bash
# Depuis le shell (pas dans psql)
psql -U postgres -d ma_base -c "\dt" | grep client

# Ou depuis psql avec \!
\! psql -c "\dt" | grep temp
```

---

### Auto-complétion

`psql` offre une auto-complétion puissante avec la touche `Tab` :

```bash
# Taper \d puis Tab → liste toutes les commandes \d*
\d<Tab>

# Taper \dt cli puis Tab → complète avec les tables commençant par "cli"
\dt cli<Tab>

# Taper \d clients puis Tab → affiche les colonnes de la table
\d clients.<Tab>
```

---

## Tableau Récapitulatif des Commandes de Navigation

| Commande | Description | Exemple |
|----------|-------------|---------|
| `\l` | Lister les bases de données | `\l+` |
| `\c` | Se connecter à une base | `\c ma_base` |
| `\dn` | Lister les schémas | `\dn+` |
| `\dt` | Lister les tables | `\dt ventes.*` |
| `\d` | Décrire un objet | `\d+ clients` |
| `\di` | Lister les index | `\di+ *clients*` |
| `\dv` | Lister les vues | `\dv+` |
| `\dm` | Lister les vues matérialisées | `\dm+` |
| `\ds` | Lister les séquences | `\ds` |
| `\df` | Lister les fonctions | `\df+ calc*` |
| `\du` | Lister les utilisateurs/rôles | `\du+` |
| `\dp` ou `\z` | Afficher les privilèges | `\dp clients` |
| `\dT` | Lister les types | `\dT+` |
| `\dx` | Lister les extensions | `\dx+` |
| `\db` | Lister les tablespaces | `\db+` |
| `\dE` | Lister les tables externes | `\dE` |
| `\des` | Lister les serveurs FDW | `\des` |
| `\dew` | Lister les FDW | `\dew` |
| `\dF` | Lister les configs FTS | `\dF` |
| `\do` | Lister les opérateurs | `\do @>` |
| `\da` | Lister les agrégats | `\da` |
| `\dA` | Lister les méthodes d'accès | `\dA` |

**Astuce** 💡 : Presque toutes les commandes acceptent `+` pour plus de détails et un pattern pour filtrer !

---

## Commandes d'Information Système

### \conninfo - Informations de connexion

```bash
\conninfo
# Résultat : You are connected to database "ma_base" as user "postgres"
#            on host "localhost" (address "127.0.0.1") at port "5432".
```

---

### \password - Changer le mot de passe

```bash
# Changer son propre mot de passe
\password

# Changer le mot de passe d'un autre utilisateur (si superuser)
\password alice
```

---

### \encoding - Afficher/définir l'encodage

```bash
# Voir l'encodage actuel
\encoding

# Changer l'encodage (côté client uniquement)
\encoding UTF8
```

---

## Workflow Typique d'Exploration

Voici un workflow recommandé pour explorer une base PostgreSQL inconnue :

```bash
# 1. Se connecter
psql -U username -d database_name

# 2. Vérifier où vous êtes
\conninfo

# 3. Voir la structure globale : quels schémas ?
\dn

# 4. Voir toutes les tables
\dt

# 5. Pour chaque table importante, voir sa structure
\d+ table_importante

# 6. Voir les relations (FK) entre tables
# Elles apparaissent dans \d+ sous "Foreign-key constraints" et "Referenced by"

# 7. Voir les index (performance)
\di+

# 8. Voir les extensions installées
\dx

# 9. Voir les utilisateurs et leurs droits
\du
\dp

# 10. Si vous avez des vues matérialisées (pour l'analytique)
\dm+
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Utilisez `+` systématiquement** quand vous explorez une nouvelle base
   ```bash
   \dt+  # Pas juste \dt
   \di+  # Pas juste \di
   ```

2. **Commencez large, puis affinez**
   ```bash
   \dt           # Toutes les tables
   \dt client*   # Tables commençant par "client"
   \d clients    # Structure de la table "clients"
   ```

3. **Vérifiez toujours les FK et index**
   ```bash
   \d+ ma_table
   # Regardez les sections "Indexes:" et "Foreign-key constraints:"
   ```

4. **Explorez les extensions**
   ```bash
   \dx+  # Pour voir ce que chaque extension apporte
   ```

5. **Documentez vos découvertes**
   - Notez les tables importantes
   - Notez les schémas et leur utilité
   - Notez les conventions de nommage

### ❌ À Éviter

1. **Ne pas assumer la structure**
   - Toujours vérifier avec `\d` avant d'écrire du SQL

2. **Ne pas ignorer les schémas**
   - Une base peut avoir plusieurs schémas
   - Vérifiez avec `\dn` et utilisez `schema.table` si nécessaire

3. **Ne pas oublier les vues matérialisées**
   - Elles peuvent contenir des données importantes
   - Listez-les avec `\dm`

4. **Ne pas négliger les permissions**
   - Vérifiez avec `\dp` avant de donner des accès

---

## Commandes d'Aide

### \? - Aide sur les méta-commandes

```bash
# Aide complète sur toutes les méta-commandes
\?

# Catégories :
#   General
#   Query Buffer
#   Input/Output
#   Informational
#   Formatting
#   Connection
#   Operating System
#   Variables
#   Large Objects
```

### \h - Aide SQL

```bash
# Aide sur une commande SQL
\h SELECT
\h CREATE TABLE
\h ALTER TABLE

# Liste toutes les commandes SQL disponibles
\h
```

---

## Conclusion

Maîtriser les commandes de navigation de `psql` est essentiel pour travailler efficacement avec PostgreSQL. Ces commandes vous permettent de :

- ✅ **Explorer** rapidement la structure d'une base  
- ✅ **Comprendre** les relations entre les objets  
- ✅ **Diagnostiquer** les problèmes de performance  
- ✅ **Auditer** les permissions et la sécurité  
- ✅ **Naviguer** efficacement entre schémas et bases

**Mémo des commandes essentielles à retenir** :
- `\l` : Bases de données  
- `\c` : Se connecter  
- `\dt` : Tables  
- `\d` : Décrire un objet  
- `\di` : Index  
- `\du` : Utilisateurs  
- `\dx` : Extensions  
- `\?` : Aide

**Prochaines sections** :
- Configuration et personnalisation de psql
- Export/Import avec psql
- Meta-commandes avancées

---


⏭️ [Configuration (\x, \timing, \pset)](/annexes/commandes-psql/02-configuration.md)
