🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.4. Gestion des Autorisations (GRANT/REVOKE)

## Introduction

Imaginez que vous construisez une maison. Vous ne donneriez pas les clés de toutes les pièces à tout le monde, n'est-ce pas ? Certaines personnes ont accès uniquement au salon, d'autres peuvent entrer dans la cuisine, et seuls quelques privilégiés peuvent accéder au coffre-fort. C'est exactement le même principe dans PostgreSQL : contrôler **qui peut faire quoi** sur les données.

Dans cette section, nous allons découvrir le système d'autorisations de PostgreSQL, qui permet de définir précisément quels utilisateurs (rôles) peuvent effectuer quelles opérations sur quels objets. C'est le fondement de la **sécurité** et de la **gouvernance des données**.

---

## Pourquoi la Gestion des Autorisations est Cruciale

### Les Risques Sans Autorisations Appropriées

Sans une gestion rigoureuse des autorisations, votre base de données est exposée à de nombreux risques :

❌ **Accès non autorisé aux données sensibles**
```sql
-- Un développeur junior peut lire les salaires de tous les employés
SELECT * FROM employes;  -- Données sensibles exposées !
```

❌ **Modifications accidentelles ou malveillantes**
```sql
-- Un script mal configuré peut supprimer toutes les données
DELETE FROM clients;  -- Catastrophe !
```

❌ **Violations de conformité**
- RGPD : Accès non contrôlé aux données personnelles
- SOX : Manque de traçabilité des accès
- HIPAA : Exposition de données médicales sensibles

❌ **Perte de contrôle**
- Qui a accès à quoi ? Impossible à dire
- Comment auditer les accès ? Aucune visibilité
- Comment limiter les dégâts d'un compte compromis ? Trop tard

### Les Bénéfices d'une Bonne Gestion

- ✅ **Sécurité renforcée** : Chaque utilisateur n'a accès qu'à ce dont il a besoin  
- ✅ **Conformité réglementaire** : Traçabilité et contrôle des accès  
- ✅ **Limitation des dégâts** : Un compte compromis a un impact limité  
- ✅ **Responsabilisation** : Chacun a des permissions adaptées à son rôle  
- ✅ **Confiance** : Les données sensibles sont protégées

---

## Le Modèle de Sécurité PostgreSQL

### Architecture en Couches

PostgreSQL utilise un modèle de sécurité à **plusieurs niveaux** :

```
┌─────────────────────────────────────────┐
│  1. Authentification (pg_hba.conf)      │  ← Qui peut se connecter ?
├─────────────────────────────────────────┤
│  2. Database                            │  ← Accès à quelle base ?
├─────────────────────────────────────────┤
│  3. Schema                              │  ← Accès à quel namespace ?
├─────────────────────────────────────────┤
│  4. Objets (Tables, Functions, etc.)    │  ← Quelles opérations ?
├─────────────────────────────────────────┤
│  5. Lignes (Row-Level Security)         │  ← Quelles lignes ?
└─────────────────────────────────────────┘
```

Chaque niveau doit être correctement configuré pour que l'utilisateur puisse accéder aux données.

### Authentification vs Autorisation

Il est important de distinguer ces deux concepts :

**Authentification** : "Qui êtes-vous ?"
- Vérification de l'identité (nom d'utilisateur + mot de passe)
- Configurée dans `pg_hba.conf`
- Méthodes : password, scram-sha-256, cert, ldap, etc.

**Autorisation** : "Que pouvez-vous faire ?"
- Gestion des permissions (GRANT/REVOKE)
- Définit les opérations autorisées
- Ce qui est couvert dans cette section

```sql
-- Authentification réussie ✅
-- Mais autorisation insuffisante ❌

-- L'utilisateur peut se connecter
\c mydb user_app
-- Connexion établie

-- Mais ne peut pas accéder aux données
SELECT * FROM clients;
-- ❌ ERROR: permission denied for table clients
```

---

## Les Concepts Fondamentaux

### 1. Les Rôles (Roles)

Dans PostgreSQL, on ne parle pas vraiment "d'utilisateurs" mais de **rôles** (roles). Un rôle peut être :

- **Un utilisateur** : Un rôle avec l'attribut LOGIN qui peut se connecter  
- **Un groupe** : Un rôle sans LOGIN qui regroupe des permissions

```sql
-- Créer un utilisateur (rôle avec LOGIN)
CREATE ROLE alice LOGIN PASSWORD 'secure_password';

-- Créer un groupe (rôle sans LOGIN)
CREATE ROLE lecteurs;

-- Ajouter alice au groupe lecteurs
GRANT lecteurs TO alice;
```

**Analogie** :
- Un **rôle utilisateur** = une personne avec un badge d'accès
- Un **rôle groupe** = un niveau d'autorisation (VIP, Personnel, Visiteur)

### 2. Les Privilèges (Privileges)

Un privilège est une **permission spécifique** d'effectuer une opération sur un objet.

Types de privilèges courants :

| Privilège | Description | Exemple |
|-----------|-------------|---------|
| **SELECT** | Lire les données | `SELECT * FROM clients` |
| **INSERT** | Ajouter des données | `INSERT INTO clients VALUES (...)` |
| **UPDATE** | Modifier des données | `UPDATE clients SET email = ...` |
| **DELETE** | Supprimer des données | `DELETE FROM clients WHERE ...` |
| **TRUNCATE** | Vider une table | `TRUNCATE TABLE clients` |
| **REFERENCES** | Créer des FK | `FOREIGN KEY (client_id) REFERENCES clients` |
| **TRIGGER** | Créer des triggers | `CREATE TRIGGER ... ON clients` |
| **USAGE** | Utiliser un schéma/séquence | Accéder aux objets d'un schéma |
| **CREATE** | Créer des objets | `CREATE TABLE ...` dans un schéma |
| **CONNECT** | Se connecter | Connexion à une base de données |
| **EXECUTE** | Exécuter une fonction | `SELECT ma_fonction()` |

### 3. Les Objets (Objects)

Les privilèges s'appliquent à différents types d'objets :

```
Base de données (DATABASE)
    └── Schéma (SCHEMA)
            ├── Table (TABLE)
            ├── Vue (VIEW)
            ├── Séquence (SEQUENCE)
            ├── Fonction (FUNCTION)
            ├── Type (TYPE)
            └── Domaine (DOMAIN)
```

### 4. Le Principe du Moindre Privilège

**Règle d'or** : N'accordez que les permissions **strictement nécessaires**, rien de plus.

❌ **Mauvaise pratique** :
```sql
-- Donner tous les droits à tout le monde
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO PUBLIC;
```

✅ **Bonne pratique** :
```sql
-- Permissions granulaires selon le besoin
GRANT SELECT ON TABLE clients TO lecteur_commercial;  
GRANT SELECT, INSERT, UPDATE ON TABLE commandes TO application_web;  
```

**Avantages** :
- 🔒 Limite l'impact d'un compte compromis  
- 🎯 Facilite les audits de sécurité  
- 📊 Rend les responsabilités claires  
- 🛡️ Réduit la surface d'attaque

---

## GRANT : Accorder des Permissions

### Concept

`GRANT` est la commande qui permet d'**accorder** des privilèges à un rôle sur un objet.

### Syntaxe Générale

```sql
GRANT privilege_type ON object_type object_name TO role_name;
```

### Anatomie d'une Commande GRANT

```sql
GRANT  SELECT, INSERT          -- 1. Quels privilèges ?  
ON     TABLE clients           -- 2. Sur quel type d'objet ?  
TO     app_backend;            -- 3. À quel rôle ?  
```

Décomposons :
1. **GRANT** : Le verbe d'action (accorder)  
2. **SELECT, INSERT** : Les privilèges spécifiques  
3. **ON TABLE** : Le type d'objet concerné  
4. **clients** : Le nom de l'objet  
5. **TO** : Destinataire  
6. **app_backend** : Le rôle qui reçoit les privilèges

### Exemples Simples

```sql
-- Lecture seule
GRANT SELECT ON TABLE produits TO lecteur;

-- Lecture et écriture
GRANT SELECT, INSERT, UPDATE ON TABLE commandes TO application;

-- Tous les privilèges
GRANT ALL PRIVILEGES ON TABLE employes TO administrateur;

-- Sur plusieurs tables
GRANT SELECT ON TABLE clients, commandes, produits TO reporting;
```

### Illustration : Avant / Après GRANT

**Avant GRANT** :
```sql
-- Se connecter en tant que lecteur
\c mydb lecteur

SELECT * FROM clients;
-- ❌ ERROR: permission denied for table clients
```

**Après GRANT** :
```sql
-- En tant que propriétaire ou superuser
GRANT SELECT ON TABLE clients TO lecteur;

-- Se connecter en tant que lecteur
\c mydb lecteur

SELECT * FROM clients;
-- ✅ Fonctionne !
```

---

## REVOKE : Révoquer des Permissions

### Concept

`REVOKE` est l'opposé de `GRANT` : il permet de **retirer** des privilèges précédemment accordés.

### Syntaxe Générale

```sql
REVOKE privilege_type ON object_type object_name FROM role_name;
```

### Exemples

```sql
-- Retirer un privilège spécifique
REVOKE INSERT ON TABLE clients FROM application;

-- Retirer plusieurs privilèges
REVOKE INSERT, UPDATE ON TABLE clients FROM application;

-- Retirer tous les privilèges
REVOKE ALL PRIVILEGES ON TABLE clients FROM application;
```

### Illustration : Avant / Après REVOKE

**Avant REVOKE** :
```sql
-- L'utilisateur peut insérer
INSERT INTO clients (nom) VALUES ('Test');
-- ✅ Fonctionne
```

**Après REVOKE** :
```sql
-- Retirer le privilège INSERT
REVOKE INSERT ON TABLE clients FROM application;

-- L'utilisateur essaie d'insérer
INSERT INTO clients (nom) VALUES ('Test');
-- ❌ ERROR: permission denied for table clients
```

---

## Le Cycle de Vie des Permissions

### Workflow Typique

```
1. Création du rôle
   CREATE ROLE mon_utilisateur LOGIN PASSWORD '...';

2. Attribution des permissions
   GRANT SELECT ON TABLE ... TO mon_utilisateur;

3. Utilisation normale
   [L'utilisateur accède aux données]

4. Modification des permissions (si besoin)
   GRANT INSERT ON TABLE ... TO mon_utilisateur;
   REVOKE SELECT ON TABLE ... FROM mon_utilisateur;

5. Fin de vie
   REVOKE ALL PRIVILEGES ... FROM mon_utilisateur;
   DROP ROLE mon_utilisateur;
```

### Les Permissions Sont Additives

Important : Les permissions sont **additives** (cumulatives), pas exclusives.

```sql
-- Accorder SELECT
GRANT SELECT ON TABLE clients TO utilisateur;

-- Ajouter INSERT (n'enlève pas SELECT)
GRANT INSERT ON TABLE clients TO utilisateur;

-- L'utilisateur a maintenant SELECT + INSERT ✅
```

Pour retirer une permission, il faut explicitement REVOKE :

```sql
-- Retirer INSERT (SELECT reste)
REVOKE INSERT ON TABLE clients FROM utilisateur;

-- Maintenant : SELECT uniquement
```

---

## PUBLIC : Le Rôle Spécial

### Qu'est-ce que PUBLIC ?

`PUBLIC` est un **pseudo-rôle** qui représente **tous les utilisateurs** de la base de données. C'est l'équivalent de "tout le monde".

```sql
-- Accorder SELECT à tout le monde
GRANT SELECT ON TABLE statistiques_publiques TO PUBLIC;

-- Révoquer de tout le monde
REVOKE SELECT ON TABLE donnees_sensibles FROM PUBLIC;
```

### Comportements par Défaut

⚠️ **Attention** : PostgreSQL accorde certaines permissions à PUBLIC par défaut :

```sql
-- Par défaut, sur une nouvelle base de données :
-- - PUBLIC a CONNECT
-- - PUBLIC a TEMP (créer des tables temporaires)

-- Par défaut, sur le schéma public :
-- - PUBLIC a CREATE
-- - PUBLIC a USAGE

-- Par défaut, sur les nouvelles fonctions :
-- - PUBLIC a EXECUTE
```

### Sécuriser PUBLIC (Recommandé en Production)

```sql
-- Révoquer les permissions par défaut dangereuses
REVOKE ALL ON DATABASE mydb FROM PUBLIC;  
REVOKE CREATE ON SCHEMA public FROM PUBLIC;  

-- Puis accorder explicitement selon les besoins
GRANT CONNECT ON DATABASE mydb TO app_backend;  
GRANT USAGE ON SCHEMA public TO app_backend;  
```

---

## La Hiérarchie des Permissions

### Dépendances Entre Niveaux

Pour accéder à une table, un utilisateur doit avoir :

```
✅ CONNECT sur la database
    ↓
✅ USAGE sur le schema
    ↓
✅ SELECT (ou autre) sur la table
```

Si **un seul** de ces niveaux manque, l'accès est refusé.

### Exemple Complet

```sql
-- Créer un utilisateur
CREATE ROLE app_user LOGIN PASSWORD 'secure_pass';

-- ❌ Sans permissions, rien ne fonctionne
\c mydb app_user
SELECT * FROM public.clients;
-- ERROR: permission denied

-- ✅ Accorder dans l'ordre
-- Niveau 1 : Database
GRANT CONNECT ON DATABASE mydb TO app_user;

-- Niveau 2 : Schema
GRANT USAGE ON SCHEMA public TO app_user;

-- Niveau 3 : Table
GRANT SELECT ON TABLE public.clients TO app_user;

-- Maintenant ça fonctionne !
\c mydb app_user
SELECT * FROM public.clients;  -- ✅ OK
```

---

## Les Propriétaires d'Objets

### Concept de Propriété

Chaque objet dans PostgreSQL a un **propriétaire** (owner) qui :
- A **tous les privilèges** sur cet objet automatiquement
- Peut accorder ou révoquer des privilèges à d'autres
- Peut modifier ou supprimer l'objet

```sql
-- Créer une table (le créateur devient propriétaire)
CREATE TABLE ma_table (id INTEGER);

-- Voir le propriétaire
\dt+ ma_table

-- Le propriétaire peut tout faire sans GRANT explicite
SELECT * FROM ma_table;  -- ✅ OK  
DROP TABLE ma_table;     -- ✅ OK  
```

### Changer le Propriétaire

```sql
-- Transférer la propriété à un autre rôle
ALTER TABLE ma_table OWNER TO nouveau_proprietaire;
```

### Propriétaire vs Privilèges

**Propriétaire** :
- Droits **complets** et **permanents**
- Ne peut pas être révoqué (sauf changement de propriétaire)
- Responsable de l'objet

**Privilèges accordés** :
- Droits **spécifiques** et **révocables**
- Peuvent être retirés à tout moment
- Accès contrôlé

---

## WITH GRANT OPTION : Déléguer l'Attribution

### Concept

`WITH GRANT OPTION` permet à un utilisateur de **transmettre** les privilèges qu'il a reçus à d'autres utilisateurs.

```sql
-- Alice reçoit SELECT avec possibilité de le transmettre
GRANT SELECT ON TABLE clients TO alice WITH GRANT OPTION;

-- Maintenant Alice peut donner SELECT à Bob
-- (en se connectant en tant que Alice)
GRANT SELECT ON TABLE clients TO bob;
```

### Cas d'Usage

C'est utile dans les organisations hiérarchiques :
- Un manager peut gérer les permissions de son équipe
- Un chef de projet peut donner accès aux ressources de son projet
- Délégation contrôlée de la gestion des permissions

### Attention : Chaîne de Responsabilité

```sql
-- Admin → Manager (WITH GRANT OPTION)
GRANT SELECT ON TABLE data TO manager WITH GRANT OPTION;

-- Manager → Developer
-- (connecté en tant que manager)
GRANT SELECT ON TABLE data TO developer;

-- Si on révoque le privilège du manager :
REVOKE SELECT ON TABLE data FROM manager CASCADE;
-- CASCADE supprime aussi les privilèges accordés par le manager
-- → developer perd aussi SELECT !
```

---

## Les Différents Niveaux d'Objets

PostgreSQL gère les permissions sur plusieurs types d'objets. Cette section introduit les concepts principaux ; les détails seront couverts dans les sous-sections suivantes.

### 1. Niveau Database

Permissions sur les bases de données elles-mêmes :
- **CONNECT** : Se connecter à la base  
- **CREATE** : Créer de nouveaux schémas  
- **TEMPORARY** : Créer des tables temporaires

```sql
GRANT CONNECT ON DATABASE production TO app_user;
```

### 2. Niveau Schema

Permissions sur les schémas (namespaces) :
- **USAGE** : Accéder aux objets du schéma  
- **CREATE** : Créer des objets dans le schéma

```sql
GRANT USAGE ON SCHEMA app_data TO app_user;
```

### 3. Niveau Objets

Permissions sur les objets individuels :
- **Tables** : SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER  
- **Sequences** : USAGE, SELECT, UPDATE  
- **Functions** : EXECUTE  
- **Types** : USAGE

```sql
GRANT SELECT, INSERT ON TABLE clients TO app_user;  
GRANT USAGE ON SEQUENCE clients_id_seq TO app_user;  
GRANT EXECUTE ON FUNCTION calculer_total() TO app_user;  
```

---

## Permissions par Défaut (Aperçu)

### Le Problème des Nouveaux Objets

Lorsque vous créez un nouvel objet, il n'hérite **pas** automatiquement des permissions accordées sur les objets existants.

```sql
-- Accorder SELECT sur toutes les tables existantes
GRANT SELECT ON ALL TABLES IN SCHEMA public TO lecteur;

-- Créer une nouvelle table
CREATE TABLE nouvelle_table (id INTEGER);

-- lecteur ne peut PAS y accéder ! 😱
-- (Aucune permission sur cette nouvelle table)
```

### La Solution : ALTER DEFAULT PRIVILEGES

Pour automatiser l'attribution des permissions sur les **futurs objets** :

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public  
GRANT SELECT ON TABLES TO lecteur;  

-- Maintenant, toute nouvelle table aura automatiquement SELECT pour lecteur
```

Ce sujet crucial sera détaillé dans la section **16.4.3. Default privileges**.

---

## Outils de Vérification

### Vérifier les Permissions Accordées

#### Commandes psql

```sql
-- Lister les tables avec leurs permissions
\dp
\z

-- Lister les schémas avec permissions
\dn+

-- Lister les bases de données avec permissions
\l+
```

#### Requêtes SQL

```sql
-- Permissions sur une table spécifique
SELECT grantee, privilege_type  
FROM information_schema.table_privileges  
WHERE table_name = 'clients';  

-- Permissions d'un rôle spécifique
SELECT table_name, privilege_type  
FROM information_schema.table_privileges  
WHERE grantee = 'app_user';  
```

### Fonctions Utiles

PostgreSQL fournit des fonctions pour tester les permissions :

```sql
-- Est-ce que l'utilisateur peut SELECT sur cette table ?
SELECT has_table_privilege('app_user', 'clients', 'SELECT');
-- Retourne : true ou false

-- Est-ce que l'utilisateur peut se connecter à cette base ?
SELECT has_database_privilege('app_user', 'mydb', 'CONNECT');

-- Est-ce que l'utilisateur peut utiliser ce schéma ?
SELECT has_schema_privilege('app_user', 'public', 'USAGE');
```

---

## Scénarios d'Erreurs Courants

### Erreur 1 : "permission denied for table"

```sql
SELECT * FROM clients;
-- ERROR: permission denied for table clients
```

**Cause** : Aucun privilège SELECT sur la table

**Solution** :
```sql
GRANT SELECT ON TABLE clients TO mon_utilisateur;
```

### Erreur 2 : "permission denied for schema"

```sql
SELECT * FROM app_data.users;
-- ERROR: permission denied for schema app_data
```

**Cause** : Pas de USAGE sur le schéma

**Solution** :
```sql
GRANT USAGE ON SCHEMA app_data TO mon_utilisateur;  
GRANT SELECT ON TABLE app_data.users TO mon_utilisateur;  
```

### Erreur 3 : "permission denied for database"

```sql
\c production
-- FATAL: permission denied for database "production"
```

**Cause** : Pas de CONNECT sur la database

**Solution** :
```sql
GRANT CONNECT ON DATABASE production TO mon_utilisateur;
```

### Erreur 4 : "permission denied for sequence"

```sql
INSERT INTO clients (nom) VALUES ('Test');
-- ERROR: permission denied for sequence clients_id_seq
```

**Cause** : La table a une colonne SERIAL, mais pas de USAGE sur la séquence

**Solution** :
```sql
GRANT USAGE ON SEQUENCE clients_id_seq TO mon_utilisateur;
-- Ou sur toutes les séquences :
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO mon_utilisateur;
```

---

## Stratégies de Gestion des Permissions

### 1. Approche par Rôles (Recommandée)

Créez des rôles groupes par fonction, pas par personne :

```sql
-- Rôles groupes
CREATE ROLE lecteurs;  
CREATE ROLE editeurs;  
CREATE ROLE administrateurs;  

-- Permissions sur les groupes
GRANT SELECT ON ALL TABLES IN SCHEMA public TO lecteurs;  
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO editeurs;  
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO administrateurs;  

-- Utilisateurs individuels
CREATE ROLE alice LOGIN PASSWORD 'pass';  
CREATE ROLE bob LOGIN PASSWORD 'pass';  

-- Affectation aux groupes
GRANT lecteurs TO alice;  
GRANT editeurs TO bob;  
```

**Avantages** :
- 🎯 Gestion centralisée  
- 🔄 Facilité de maintenance  
- 📊 Clarté des responsabilités  
- ⚡ Changements rapides

### 2. Approche par Environnement

Différenciez les permissions selon l'environnement :

```sql
-- Développement : permissif
ALTER DEFAULT PRIVILEGES IN SCHEMA public  
GRANT ALL PRIVILEGES ON TABLES TO developpeurs;  

-- Staging : intermédiaire
ALTER DEFAULT PRIVILEGES IN SCHEMA public  
GRANT SELECT, INSERT, UPDATE ON TABLES TO app_staging;  

-- Production : restrictif
ALTER DEFAULT PRIVILEGES IN SCHEMA public  
GRANT SELECT, INSERT, UPDATE ON TABLES TO app_production;  
-- DELETE nécessite une approbation séparée
```

### 3. Approche par Schéma

Isolez les permissions par schéma :

```sql
-- Schéma public : données partagées
GRANT USAGE ON SCHEMA public TO PUBLIC;  
GRANT SELECT ON ALL TABLES IN SCHEMA public TO PUBLIC;  

-- Schéma app_internal : données de l'application
GRANT USAGE ON SCHEMA app_internal TO app_backend;  
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA app_internal TO app_backend;  

-- Schéma admin : données sensibles
GRANT USAGE ON SCHEMA admin TO administrateurs;  
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA admin TO administrateurs;  
```

---

## Checklist de Sécurité

Avant de mettre une base de données en production, vérifiez :

### Niveau Database
- [ ] CONNECT révoqué de PUBLIC  
- [ ] CONNECT accordé uniquement aux rôles nécessaires  
- [ ] CREATE révoqué de PUBLIC (sauf cas spécifiques)

### Niveau Schema
- [ ] USAGE accordé uniquement aux rôles nécessaires  
- [ ] CREATE sur le schéma public révoqué de PUBLIC  
- [ ] Schémas séparés pour différentes fonctions

### Niveau Objets
- [ ] Aucune table accessible par PUBLIC (sauf données vraiment publiques)  
- [ ] Permissions granulaires (pas de ALL PRIVILEGES sauf pour admin)  
- [ ] USAGE accordé sur les séquences nécessaires  
- [ ] EXECUTE sur fonctions révoqué de PUBLIC

### Niveau Configuration
- [ ] Default privileges configurés pour les nouveaux objets  
- [ ] Rôles groupes créés et utilisés  
- [ ] Documentation des permissions  
- [ ] Audit régulier des accès

---

## Plan de Cette Section

Cette section 16.4 est organisée en trois parties complémentaires :

### **16.4.1. Permissions sur Objets (TABLE, SEQUENCE, FUNCTION)**
Détaille les permissions spécifiques à chaque type d'objet :
- Comment accorder SELECT, INSERT, UPDATE sur les tables
- Gérer les permissions sur les séquences (SERIAL)
- Contrôler l'exécution des fonctions
- Permissions au niveau des colonnes

### **16.4.2. Permissions de Schéma et Database**
Explique les permissions aux niveaux supérieurs :
- CONNECT sur les databases
- USAGE et CREATE sur les schémas
- La hiérarchie complète des permissions
- Sécurisation du schéma public

### **16.4.3. Default Privileges (ALTER DEFAULT PRIVILEGES)**
Automatise l'attribution des permissions :
- Configurer les permissions par défaut
- Éviter les oublis sur les nouveaux objets
- Intégration dans les pipelines CI/CD
- Bonnes pratiques d'automatisation

---

## Conclusion de l'Introduction

La gestion des autorisations avec GRANT et REVOKE est un pilier fondamental de la sécurité PostgreSQL. Elle repose sur des principes simples mais puissants :

- ✅ **Principe du moindre privilège** : Donnez le minimum nécessaire  
- ✅ **Granularité** : Contrôle précis au niveau database, schéma, objet  
- ✅ **Hiérarchie** : Les permissions se cumulent à travers les niveaux  
- ✅ **Révocabilité** : Tout privilège accordé peut être retiré  
- ✅ **Auditabilité** : Vérification et traçabilité des accès

Les sections suivantes vous fourniront tous les détails pratiques pour maîtriser ces concepts et les appliquer efficacement dans vos projets.

**Prêt à plonger dans les détails ?** Commençons par les permissions sur les objets ! 🚀

---


⏭️ [Permissions sur objets (TABLE, SEQUENCE, FUNCTION)](/16-administration-configuration-securite/04.1-permissions-sur-objets.md)
