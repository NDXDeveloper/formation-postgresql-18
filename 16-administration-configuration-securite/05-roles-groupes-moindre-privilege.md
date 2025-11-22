🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.5. Rôles, Groupes et Principe du Moindre Privilège

## Introduction

Dans PostgreSQL, la gestion des utilisateurs et des permissions repose sur un concept central : les **rôles**. Contrairement à d'autres systèmes de bases de données qui séparent clairement "utilisateurs" et "groupes", PostgreSQL unifie ces deux concepts en un seul : **le rôle**.

Cette section explore :
- Ce qu'est un rôle dans PostgreSQL
- Comment organiser les rôles en hiérarchies (groupes)
- Le principe du moindre privilège (security best practice)
- Comment concevoir une architecture de sécurité robuste

---

## 🎭 Qu'est-ce qu'un Rôle ?

### Définition

Un **rôle** dans PostgreSQL est une entité qui peut :
1. **Se connecter** à la base de données (comme un "utilisateur")
2. **Contenir d'autres rôles** (comme un "groupe")
3. **Posséder des objets** (tables, schémas, etc.)
4. **Avoir des permissions** sur des objets

**En résumé** : Un rôle est à la fois un utilisateur ET un groupe potentiel. C'est un concept unifié.

### Analogie du Monde Réel

Imaginez une entreprise :

**Les employés** (Alice, Bob) sont des **rôles individuels** qui peuvent :
- Se connecter au système (LOGIN)
- Faire certaines actions (permissions)

**Les départements** (Comptabilité, RH, IT) sont aussi des **rôles**, mais :
- Ils ne se connectent pas directement (pas de LOGIN)
- Ils regroupent des employés
- Ils définissent des permissions communes

PostgreSQL fonctionne exactement de la même manière !

---

## 👤 Rôles de Connexion (Utilisateurs)

### Création d'un Rôle de Connexion

Un **rôle de connexion** est ce qu'on appelle traditionnellement un "utilisateur". C'est un rôle qui peut se connecter à la base de données.

**Syntaxe :**
```sql
CREATE ROLE nom_role WITH LOGIN PASSWORD 'mot_de_passe';

-- OU, syntaxe équivalente plus intuitive :
CREATE USER nom_utilisateur WITH PASSWORD 'mot_de_passe';
```

⚠️ **Important** : `CREATE USER` est juste un alias pour `CREATE ROLE WITH LOGIN`. Les deux commandes font exactement la même chose !

**Exemple concret :**
```sql
-- Créer un utilisateur pour une application web
CREATE USER app_web WITH PASSWORD 'P@ssw0rd_S3cur3';

-- Créer un utilisateur pour un analyste
CREATE USER alice_analyst WITH PASSWORD 'Mot_De_Passe_Fort';
```

### Attributs Essentiels d'un Rôle de Connexion

Lors de la création, vous pouvez spécifier plusieurs attributs :

```sql
CREATE ROLE alice WITH
  LOGIN                        -- Peut se connecter
  PASSWORD 'secure_password'   -- Mot de passe
  VALID UNTIL '2026-12-31'     -- Expiration du compte
  CONNECTION LIMIT 10          -- Maximum 10 connexions simultanées
  CREATEDB                     -- Peut créer des bases de données
  CREATEROLE                   -- Peut créer d'autres rôles
  IN ROLE groupe_lecteurs;     -- Membre du groupe 'groupe_lecteurs'
```

**Détail des attributs :**

| Attribut | Description | Exemple d'usage |
|----------|-------------|-----------------|
| `LOGIN` | Autorise la connexion | Utilisateurs humains et applications |
| `NOLOGIN` | Interdit la connexion (rôle groupe) | Groupes de permissions |
| `PASSWORD` | Définit le mot de passe | Sécurité de base |
| `VALID UNTIL` | Date d'expiration du compte | Comptes temporaires |
| `CONNECTION LIMIT` | Nombre max de connexions | Limiter les ressources |
| `SUPERUSER` | Tous les droits (⚠️ dangereux) | Administrateur DBA uniquement |
| `CREATEDB` | Peut créer des bases | Développeurs |
| `CREATEROLE` | Peut créer des rôles | Administrateurs délégués |
| `REPLICATION` | Peut faire de la réplication | Serveurs standby |
| `INHERIT` | Hérite des permissions des groupes | Comportement par défaut |
| `NOINHERIT` | N'hérite pas automatiquement | Sécurité renforcée |

### Vérifier les Rôles Existants

```sql
-- Lister tous les rôles
SELECT rolname, rolcanlogin, rolsuper, rolcreatedb
FROM pg_roles
ORDER BY rolname;

-- OU depuis psql :
\du
```

**Exemple de résultat :**
```
        List of roles
   Role name   | Login | Superuser | Create DB
---------------+-------+-----------+-----------
 alice         | yes   | no        | no
 app_web       | yes   | no        | no
 postgres      | yes   | yes       | yes
 groupe_dev    | no    | no        | yes
```

---

## 👥 Rôles de Groupe (Groupes)

### Concept de Rôle Groupe

Un **rôle groupe** est un rôle qui :
- **N'a PAS** l'attribut `LOGIN` (ne peut pas se connecter)
- **Contient d'autres rôles** (comme membres)
- **Définit des permissions communes** à tous ses membres

**Analogie :** C'est comme un département dans une entreprise. Le département "Comptabilité" n'est pas une personne qui se connecte, mais il regroupe plusieurs comptables.

### Création d'un Rôle Groupe

```sql
-- Créer un groupe sans possibilité de connexion
CREATE ROLE groupe_lecteurs NOLOGIN;

-- Créer un groupe avec des permissions
CREATE ROLE groupe_developpeurs NOLOGIN;
CREATE ROLE groupe_analystes NOLOGIN;
```

**Visualisation :**
```
┌─────────────────────────────────────┐
│    groupe_developpeurs (NOLOGIN)    │
│  ┌───────────────────────────────┐  │
│  │ Permissions :                 │  │
│  │ - SELECT sur toutes les tables│  │
│  │ - INSERT, UPDATE, DELETE      │  │
│  │ - CREATE TABLE                │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
         ↑              ↑
         │              │
    ┌────┴───┐      ┌───┴────┐
    │ alice  │      │  bob   │
    │ (LOGIN)│      │ (LOGIN)│
    └────────┘      └────────┘
```

### Ajouter des Membres à un Groupe

**Méthode 1 : Lors de la création du rôle**
```sql
CREATE USER alice WITH
  PASSWORD 'password'
  IN ROLE groupe_developpeurs;
```

**Méthode 2 : Après la création (recommandé)**
```sql
-- Ajouter alice au groupe
GRANT groupe_developpeurs TO alice;

-- Ajouter plusieurs utilisateurs en une fois
GRANT groupe_developpeurs TO alice, bob, charlie;
```

**Vérification :**
```sql
-- Voir les membres d'un groupe
SELECT
  r.rolname AS membre,
  m.rolname AS groupe
FROM pg_auth_members a
JOIN pg_roles r ON a.member = r.oid
JOIN pg_roles m ON a.roleid = m.oid
WHERE m.rolname = 'groupe_developpeurs';
```

### Retirer un Membre d'un Groupe

```sql
-- Retirer alice du groupe
REVOKE groupe_developpeurs FROM alice;
```

---

## 🔄 Héritage des Permissions

### Le Concept d'Héritage

Par défaut, quand un rôle est membre d'un groupe, il **hérite automatiquement** de toutes les permissions du groupe.

**Exemple :**
```sql
-- 1. Créer un groupe avec des permissions
CREATE ROLE groupe_lecteurs NOLOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO groupe_lecteurs;

-- 2. Créer un utilisateur membre du groupe
CREATE USER alice WITH PASSWORD 'password' IN ROLE groupe_lecteurs;

-- 3. Alice hérite automatiquement des permissions
-- Alice peut maintenant faire SELECT sur toutes les tables !
```

### INHERIT vs NOINHERIT

**Mode INHERIT (par défaut) :**
```sql
CREATE ROLE alice WITH LOGIN PASSWORD 'password' INHERIT;
GRANT groupe_lecteurs TO alice;

-- Alice hérite AUTOMATIQUEMENT des permissions de groupe_lecteurs
```

**Mode NOINHERIT :**
```sql
CREATE ROLE bob WITH LOGIN PASSWORD 'password' NOINHERIT;
GRANT groupe_admin TO bob;

-- Bob NE hérite PAS automatiquement des permissions de groupe_admin
-- Il doit explicitement "devenir" le rôle groupe_admin
```

**Activation manuelle avec NOINHERIT :**
```sql
-- Bob doit explicitement changer de rôle
SET ROLE groupe_admin;

-- Maintenant Bob a les permissions du groupe
SELECT * FROM table_sensible;  -- ✅ Fonctionne

-- Revenir au rôle d'origine
RESET ROLE;

-- Maintenant Bob n'a plus les permissions du groupe
SELECT * FROM table_sensible;  -- ❌ Erreur : permission denied
```

**Quand utiliser NOINHERIT ?**

✅ **Utilisez NOINHERIT pour :**
- Rôles avec permissions très sensibles (admin, superuser)
- Sécurité renforcée : activation explicite requise
- Audit : trace claire de qui utilise quelles permissions

❌ **N'utilisez PAS NOINHERIT pour :**
- Rôles de permissions normales (lecture, écriture)
- Simplifie trop la gestion au quotidien

---

## 🏗️ Hiérarchies de Rôles Avancées

### Groupes Imbriqués

PostgreSQL permet de créer des **hiérarchies complexes** de rôles.

**Exemple : Organisation d'entreprise**
```sql
-- Niveau 1 : Groupes de base
CREATE ROLE groupe_lecteurs NOLOGIN;
CREATE ROLE groupe_editeurs NOLOGIN;

-- Niveau 2 : Groupes métiers (héritent des groupes de base)
CREATE ROLE groupe_analystes NOLOGIN IN ROLE groupe_lecteurs;
CREATE ROLE groupe_developpeurs NOLOGIN IN ROLE groupe_editeurs;

-- Niveau 3 : Groupes avec responsabilités accrues
CREATE ROLE groupe_tech_leads NOLOGIN IN ROLE groupe_developpeurs;
CREATE ROLE groupe_dbas NOLOGIN IN ROLE groupe_tech_leads;

-- Utilisateurs finaux
CREATE USER alice WITH PASSWORD 'pwd' IN ROLE groupe_analystes;
CREATE USER bob WITH PASSWORD 'pwd' IN ROLE groupe_developpeurs;
CREATE USER charlie WITH PASSWORD 'pwd' IN ROLE groupe_dbas;
```

**Visualisation de la hiérarchie :**
```
                    ┌──────────────┐
                    │ groupe_dbas  │ (permissions admin)
                    └──────┬───────┘
                           │
                    ┌──────┴───────────┐
                    │ groupe_tech_leads│ (permissions lead)
                    └──────┬───────────┘
                           │
         ┌─────────────────┴─────────────────┐
         │                                   │
┌────────┴───────────┐            ┌──────────┴─────────┐
│ groupe_developpeurs│            │ groupe_analystes   │
│ (permissions write)│            │ (permissions read) │
└────────┬───────────┘            └──────────┬─────────┘
         │                                   │
┌────────┴───────────┐            ┌──────────┴─────────┐
│  bob (utilisateur) │            │ alice (utilisateur)│
└────────────────────┘            └────────────────────┘
```

**Permissions cumulées :**
- **Alice** : SELECT uniquement (groupe_lecteurs)
- **Bob** : SELECT + INSERT + UPDATE + DELETE (groupe_editeurs)
- **Charlie** : Toutes les permissions de Bob + permissions DBA

### Exemple Pratique : Application Multi-Tenant

**Contexte :** Application SaaS avec plusieurs clients (tenants).

```sql
-- Groupe de base : lecture uniquement
CREATE ROLE app_readonly NOLOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;

-- Groupe par client
CREATE ROLE tenant_acme NOLOGIN IN ROLE app_readonly;
CREATE ROLE tenant_globex NOLOGIN IN ROLE app_readonly;

-- Permissions spécifiques par tenant (Row-Level Security serait mieux ici)
-- Pour l'exemple, on donne accès à des schémas dédiés
GRANT SELECT ON ALL TABLES IN SCHEMA acme_data TO tenant_acme;
GRANT SELECT ON ALL TABLES IN SCHEMA globex_data TO tenant_globex;

-- Utilisateurs par client
CREATE USER acme_app WITH PASSWORD 'pwd' IN ROLE tenant_acme;
CREATE USER globex_app WITH PASSWORD 'pwd' IN ROLE tenant_globex;
```

---

## 🛡️ Le Principe du Moindre Privilège

### Définition

Le **principe du moindre privilège** (Principle of Least Privilege - PoLP) stipule qu'un utilisateur, programme ou processus ne doit avoir **que les permissions strictement nécessaires** pour accomplir sa tâche, et rien de plus.

**En d'autres termes :**
> "Ne donnez jamais plus de droits que le minimum absolument requis."

### Pourquoi C'est Important ?

**Réduction de la Surface d'Attaque :**
- Si un compte est compromis, les dégâts sont limités
- Un attaquant ne peut faire que ce que le compte autorise

**Limitation des Erreurs Humaines :**
- Un développeur ne peut pas accidentellement supprimer toute la production
- Les scripts automatisés ne peuvent pas causer de dommages excessifs

**Conformité et Audit :**
- Facilite la traçabilité des actions
- Répond aux exigences réglementaires (RGPD, SOC 2, etc.)

### Exemple : Mauvaise Pratique vs Bonne Pratique

#### ❌ Mauvaise Pratique : Superuser Partout

```sql
-- MAUVAIS : Donner tous les droits à l'application
CREATE USER app_web WITH SUPERUSER PASSWORD 'password';

-- Conséquences catastrophiques possibles :
-- - L'application peut DROP DATABASE
-- - Peut modifier pg_hba.conf
-- - Peut créer d'autres superusers
-- - Si piratée : contrôle total du serveur
```

#### ✅ Bonne Pratique : Permissions Minimales

```sql
-- BON : Permissions strictement nécessaires
CREATE USER app_web WITH PASSWORD 'secure_password';

-- Connexion à la base uniquement
GRANT CONNECT ON DATABASE production TO app_web;

-- Utilisation du schéma public
GRANT USAGE ON SCHEMA public TO app_web;

-- Opérations CRUD sur les tables nécessaires
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE users TO app_web;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE orders TO app_web;
GRANT SELECT ON TABLE products TO app_web;  -- Lecture seule

-- Utilisation des séquences pour les ID auto-incrémentés
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_web;

-- Résultat : app_web peut faire son travail, mais RIEN D'AUTRE
```

**Conséquences positives :**
- Si l'application est piratée, l'attaquant est limité aux tables spécifiées
- Impossible de DROP DATABASE ou de modifier le schéma
- Impossible de lire les tables sensibles (ex: admin_logs, salaries)

---

## 📋 Scénarios Pratiques et Modèles de Rôles

### Modèle 1 : Application Web Standard

**Architecture :**
- Application web classique (CRUD)
- Base de données PostgreSQL
- Plusieurs environnements (dev, staging, prod)

**Structure des rôles :**

```sql
-- 1. GROUPE DE BASE : Lecture seule
CREATE ROLE app_readonly NOLOGIN;
GRANT CONNECT ON DATABASE myapp TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;

-- 2. GROUPE : Opérations complètes (CRUD)
CREATE ROLE app_readwrite NOLOGIN IN ROLE app_readonly;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;

-- 3. UTILISATEUR PRODUCTION (lecture/écriture limitée)
CREATE USER app_prod WITH PASSWORD 'prod_password' IN ROLE app_readwrite;

-- 4. UTILISATEUR DÉVELOPPEMENT (plus de droits)
CREATE USER app_dev WITH PASSWORD 'dev_password' CREATEDB IN ROLE app_readwrite;

-- 5. ANALYSTE (lecture seule)
CREATE USER analyst_alice WITH PASSWORD 'analyst_pwd' IN ROLE app_readonly;
```

**Matrice des permissions :**
```
┌──────────────────┬──────────┬────────┬──────────┬──────────┬─────────┐
│ Utilisateur      │ CONNECT  │ SELECT │ INSERT   │ UPDATE   │ DELETE  │
├──────────────────┼──────────┼────────┼──────────┼──────────┼─────────┤
│ app_prod         │    ✅    │   ✅   │    ✅    │    ✅    │   ✅    │
│ app_dev          │    ✅    │   ✅   │    ✅    │    ✅    │   ✅    │
│ analyst_alice    │    ✅    │   ✅   │    ❌    │    ❌    │   ❌    │
└──────────────────┴──────────┴────────┴──────────┴──────────┴─────────┘
```

### Modèle 2 : Équipe de Données (Data Team)

**Architecture :**
- Data warehouse / Data lake
- Analystes de données
- Data scientists
- Ingénieurs données (ETL)

**Structure des rôles :**

```sql
-- 1. GROUPE : Lecture Analytics
CREATE ROLE data_readers NOLOGIN;
GRANT CONNECT ON DATABASE analytics_db TO data_readers;
GRANT USAGE ON SCHEMA public, analytics, staging TO data_readers;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO data_readers;

-- 2. GROUPE : Ingénieurs Données (ETL)
CREATE ROLE data_engineers NOLOGIN IN ROLE data_readers;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA staging TO data_engineers;
GRANT CREATE ON SCHEMA staging TO data_engineers;

-- 3. GROUPE : Data Scientists (+ tables temporaires)
CREATE ROLE data_scientists NOLOGIN IN ROLE data_readers;
GRANT TEMP ON DATABASE analytics_db TO data_scientists;
GRANT CREATE ON SCHEMA public TO data_scientists;

-- 4. Utilisateurs individuels
CREATE USER alice_analyst WITH PASSWORD 'pwd' IN ROLE data_readers;
CREATE USER bob_engineer WITH PASSWORD 'pwd' IN ROLE data_engineers;
CREATE USER charlie_scientist WITH PASSWORD 'pwd' IN ROLE data_scientists;
```

**Qui peut faire quoi :**
```
alice_analyst (Analyste) :
  ✅ Lire toutes les données analytics
  ❌ Modifier les données
  ❌ Créer des tables

bob_engineer (Ingénieur ETL) :
  ✅ Lire toutes les données analytics
  ✅ Modifier les données de staging
  ✅ Créer des tables dans staging
  ❌ Modifier analytics (en production)

charlie_scientist (Data Scientist) :
  ✅ Lire toutes les données analytics
  ✅ Créer des tables temporaires
  ✅ Créer des tables dans public (expérimentation)
  ❌ Modifier staging ou analytics
```

### Modèle 3 : Microservices

**Architecture :**
- Application découpée en microservices
- Chaque service a son propre utilisateur PostgreSQL
- Isolation des permissions par service

**Structure des rôles :**

```sql
-- Schémas par domaine
CREATE SCHEMA users_schema;
CREATE SCHEMA orders_schema;
CREATE SCHEMA inventory_schema;
CREATE SCHEMA payments_schema;

-- Service Users (gestion utilisateurs)
CREATE ROLE service_users NOLOGIN;
GRANT USAGE ON SCHEMA users_schema TO service_users;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA users_schema TO service_users;

CREATE USER api_users_service WITH PASSWORD 'pwd' IN ROLE service_users;

-- Service Orders (gestion commandes)
CREATE ROLE service_orders NOLOGIN;
GRANT USAGE ON SCHEMA orders_schema, users_schema TO service_orders;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA orders_schema TO service_orders;
GRANT SELECT ON TABLE users_schema.users TO service_orders;  -- Lecture seule users

CREATE USER api_orders_service WITH PASSWORD 'pwd' IN ROLE service_orders;

-- Service Inventory (gestion stock)
CREATE ROLE service_inventory NOLOGIN;
GRANT USAGE ON SCHEMA inventory_schema TO service_inventory;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA inventory_schema TO service_inventory;

CREATE USER api_inventory_service WITH PASSWORD 'pwd' IN ROLE service_inventory;

-- Service Payments (paiements - critique !)
CREATE ROLE service_payments NOLOGIN;
GRANT USAGE ON SCHEMA payments_schema, orders_schema TO service_payments;
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA payments_schema TO service_payments;
GRANT SELECT ON TABLE orders_schema.orders TO service_payments;  -- Lecture seule orders

CREATE USER api_payments_service WITH PASSWORD 'pwd' IN ROLE service_payments;
```

**Avantages de cette approche :**
- ✅ Isolation : Chaque service ne voit que ses données
- ✅ Sécurité : Compromission d'un service = dégâts limités
- ✅ Audit : Traçabilité claire des actions par service
- ✅ Évolutivité : Facile d'ajouter de nouveaux services

---

## ⚙️ Gestion des Permissions par Défaut

### Le Problème des Nouvelles Tables

Imaginez ce scénario :
```sql
-- Vous donnez des permissions à un groupe
GRANT SELECT ON ALL TABLES IN SCHEMA public TO groupe_lecteurs;

-- Plus tard, vous créez une nouvelle table
CREATE TABLE nouvelle_table (id INT, data TEXT);

-- ❌ PROBLÈME : groupe_lecteurs N'A PAS accès à nouvelle_table !
-- Il faut re-exécuter le GRANT...
```

### La Solution : ALTER DEFAULT PRIVILEGES

**ALTER DEFAULT PRIVILEGES** permet de définir des permissions **automatiques** pour les **futurs objets**.

**Syntaxe :**
```sql
ALTER DEFAULT PRIVILEGES
  IN SCHEMA nom_schema
  FOR ROLE role_createur
  GRANT permissions ON TABLES TO role_beneficiaire;
```

**Exemple concret :**
```sql
-- Définir les permissions par défaut pour les futures tables
ALTER DEFAULT PRIVILEGES
  IN SCHEMA public
  FOR ROLE postgres  -- Rôle qui crée les objets
  GRANT SELECT ON TABLES TO groupe_lecteurs;

-- Maintenant, toute table créée par 'postgres' dans 'public'
-- donnera automatiquement SELECT à 'groupe_lecteurs'

-- Test :
CREATE TABLE test_table (id INT);

-- groupe_lecteurs a automatiquement accès !
```

**Exemple complet pour une application :**
```sql
-- Groupe de l'application
CREATE ROLE app_role NOLOGIN;

-- Permissions sur les objets existants
GRANT USAGE ON SCHEMA public TO app_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_role;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_role;

-- Permissions sur les FUTURS objets
ALTER DEFAULT PRIVILEGES
  IN SCHEMA public
  FOR ROLE postgres
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_role;

ALTER DEFAULT PRIVILEGES
  IN SCHEMA public
  FOR ROLE postgres
  GRANT USAGE ON SEQUENCES TO app_role;

-- Utilisateur de l'application
CREATE USER app_web WITH PASSWORD 'pwd' IN ROLE app_role;
```

---

## 🔍 Audit et Vérification des Permissions

### Vérifier les Permissions d'un Rôle

```sql
-- Permissions sur les tables
SELECT
  schemaname,
  tablename,
  has_table_privilege('alice', schemaname||'.'||tablename, 'SELECT') AS can_select,
  has_table_privilege('alice', schemaname||'.'||tablename, 'INSERT') AS can_insert,
  has_table_privilege('alice', schemaname||'.'||tablename, 'UPDATE') AS can_update,
  has_table_privilege('alice', schemaname||'.'||tablename, 'DELETE') AS can_delete
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;
```

### Lister les Membres d'un Groupe

```sql
-- Méthode 1 : Via pg_auth_members
SELECT
  r.rolname AS membre,
  m.rolname AS groupe
FROM pg_auth_members a
JOIN pg_roles r ON a.member = r.oid
JOIN pg_roles m ON a.roleid = m.oid
WHERE m.rolname = 'groupe_developpeurs';

-- Méthode 2 : Depuis psql
\du groupe_developpeurs
```

### Permissions Effectives d'un Utilisateur

```sql
-- Voir toutes les permissions d'un utilisateur (via groupes)
WITH RECURSIVE role_tree AS (
  -- Rôle de départ
  SELECT oid, rolname, 0 AS level
  FROM pg_roles
  WHERE rolname = 'alice'

  UNION ALL

  -- Rôles parents (groupes)
  SELECT pr.oid, pr.rolname, rt.level + 1
  FROM pg_roles pr
  JOIN pg_auth_members am ON pr.oid = am.roleid
  JOIN role_tree rt ON am.member = rt.oid
)
SELECT DISTINCT rolname, level
FROM role_tree
ORDER BY level, rolname;
```

### Auditer les Superusers

```sql
-- Lister tous les superusers (⚠️ devrait être minimal !)
SELECT
  rolname,
  rolsuper,
  rolcreatedb,
  rolcreaterole,
  rolcanlogin
FROM pg_roles
WHERE rolsuper = true
ORDER BY rolname;
```

---

## ⚠️ Erreurs Courantes et Pièges

### Erreur #1 : Confondre ROLE et USER

```sql
-- ❌ Confusion fréquente
CREATE USER mon_groupe NOLOGIN;  -- Incohérent !

-- ✅ Correct
CREATE ROLE mon_groupe NOLOGIN;  -- Groupe
CREATE USER mon_utilisateur WITH PASSWORD 'pwd';  -- Utilisateur
```

**Clarification :**
- `CREATE ROLE` : Terme générique (avec ou sans LOGIN)
- `CREATE USER` : Alias pour `CREATE ROLE WITH LOGIN`

### Erreur #2 : Oublier GRANT USAGE sur le Schéma

```sql
-- L'utilisateur a SELECT sur la table, mais...
GRANT SELECT ON TABLE public.users TO alice;

-- ❌ Alice ne peut toujours pas accéder !
-- Pourquoi ? Elle n'a pas USAGE sur le schéma 'public'

-- ✅ Solution
GRANT USAGE ON SCHEMA public TO alice;
GRANT SELECT ON TABLE public.users TO alice;
```

**Règle :** Pour accéder à un objet, il faut :
1. Permission sur le **schéma** (USAGE)
2. Permission sur l'**objet** (SELECT, INSERT, etc.)

### Erreur #3 : Ne Pas Utiliser DEFAULT PRIVILEGES

```sql
-- ❌ Donner des permissions uniquement sur les tables existantes
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_role;

-- Les futures tables ne seront PAS accessibles !

-- ✅ Solution : Combiner les deux
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO app_role;
```

### Erreur #4 : Trop de Superusers

```sql
-- ❌ DANGEREUX : Donner SUPERUSER à tout le monde
ALTER USER alice WITH SUPERUSER;
ALTER USER bob WITH SUPERUSER;
ALTER USER app_web WITH SUPERUSER;

-- ✅ BON : Un seul superuser (postgres), le reste avec permissions ciblées
-- alice : CREATEROLE pour gérer les utilisateurs
-- bob : CREATEDB pour créer des bases
-- app_web : Permissions CRUD strictement limitées
```

**Règle d'or :** Maximum 1-2 superusers par instance (DBA uniquement).

### Erreur #5 : Permissions Trop Larges

```sql
-- ❌ Donner tous les droits sur toute la base
GRANT ALL PRIVILEGES ON DATABASE prod TO app_web;

-- ✅ Permissions granulaires
GRANT CONNECT ON DATABASE prod TO app_web;
GRANT USAGE ON SCHEMA public TO app_web;
GRANT SELECT, INSERT, UPDATE ON TABLE users, orders TO app_web;
```

---

## 🧠 Points Clés à Retenir

1. **Rôle = Utilisateur OU Groupe** : Dans PostgreSQL, c'est le même concept unifié
2. **LOGIN vs NOLOGIN** :
   - LOGIN : Peut se connecter (utilisateur)
   - NOLOGIN : Ne peut pas se connecter (groupe)
3. **Héritage par défaut** : Les membres d'un groupe héritent automatiquement des permissions (sauf si NOINHERIT)
4. **Hiérarchies possibles** : Les groupes peuvent contenir d'autres groupes
5. **Principe du moindre privilège** : Ne jamais donner plus de permissions que nécessaire
6. **DEFAULT PRIVILEGES** : Essentiel pour les futures tables
7. **SUPERUSER = Danger** : À utiliser avec extrême parcimonie
8. **Permissions à deux niveaux** : Schéma (USAGE) + Objet (SELECT, etc.)

---

## 📊 Checklist de Sécurité

### Audit Mensuel

```
☐ Vérifier le nombre de superusers (devrait être ≤ 2)
☐ Lister les utilisateurs inactifs depuis 90+ jours
☐ Vérifier les rôles avec CREATEROLE ou CREATEDB
☐ Auditer les permissions sur les tables sensibles
☐ Vérifier les DEFAULT PRIVILEGES sont configurés
☐ Contrôler les CONNECTION LIMIT
☐ Examiner les rôles avec VALID UNTIL expiré
```

### À la Création de Chaque Nouveau Rôle

```
☐ Définir un mot de passe fort (SCRAM-SHA-256)
☐ Limiter les connexions (CONNECTION LIMIT)
☐ Définir une date d'expiration si temporaire (VALID UNTIL)
☐ Assigner au(x) groupe(s) approprié(s)
☐ Vérifier l'héritage (INHERIT ou NOINHERIT)
☐ Ne PAS donner SUPERUSER sauf absolue nécessité
☐ Documenter la raison d'être du rôle
☐ Tester les permissions dans un environnement de test
```

---

## 🚀 Pour Aller Plus Loin

### Sujets Connexes

Dans les sections suivantes du tutoriel :

- **16.4** : Gestion des autorisations (`GRANT`/`REVOKE`)
- **16.6** : Row-Level Security (RLS) - Permissions au niveau des lignes
- **16.7** : SSL/TLS et chiffrement des connexions
- **16.10** : Maintenance et gestion des utilisateurs

### Concepts Avancés

**Row-Level Security (RLS) :**
- Filtrer les données visibles au niveau des lignes
- Chaque utilisateur ne voit que "ses" données
- Exemple : SaaS multi-tenant

**Audit Logging :**
- Extension pgaudit pour tracer toutes les actions
- Log des connexions, requêtes, modifications DDL

**Roles Dynamiques :**
- SET ROLE pour changer de rôle temporairement
- Élévation de privilèges temporaire et contrôlée

---

## 📚 Ressources Complémentaires

- **Documentation PostgreSQL** : [Database Roles](https://www.postgresql.org/docs/current/user-manag.html)
- **Documentation PostgreSQL** : [Role Membership](https://www.postgresql.org/docs/current/role-membership.html)
- **Documentation PostgreSQL** : [Default Privileges](https://www.postgresql.org/docs/current/ddl-priv.html#DDL-PRIV-DEFAULT)
- **OWASP** : [Principle of Least Privilege](https://owasp.org/www-community/Access_Control)
- **NIST** : Cybersecurity Framework - Access Control

---

## 📝 Glossaire

| Terme | Définition |
|-------|------------|
| **Rôle** | Entité PostgreSQL qui peut être un utilisateur ET/OU un groupe |
| **LOGIN** | Attribut permettant à un rôle de se connecter |
| **NOLOGIN** | Rôle qui ne peut pas se connecter (groupe uniquement) |
| **INHERIT** | Héritage automatique des permissions des groupes parents |
| **NOINHERIT** | Nécessite une activation explicite (`SET ROLE`) |
| **SUPERUSER** | Rôle avec tous les privilèges (à éviter) |
| **Principe du moindre privilège** | Donner uniquement les permissions strictement nécessaires |
| **DEFAULT PRIVILEGES** | Permissions automatiques pour les futurs objets |
| **GRANT** | Donner une permission |
| **REVOKE** | Retirer une permission |

---


⏭️ [Row-Level Security (RLS) : Politiques par ligne](/16-administration-configuration-securite/06-row-level-security.md)
