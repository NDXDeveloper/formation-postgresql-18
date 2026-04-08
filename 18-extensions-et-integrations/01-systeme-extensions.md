🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.1. Système d'Extensions PostgreSQL (CREATE EXTENSION)

## Introduction

PostgreSQL est bien plus qu'un simple système de gestion de bases de données. L'une de ses forces majeures réside dans son système d'extensions, qui permet d'ajouter des fonctionnalités supplémentaires sans modifier le cœur du système. Ce chapitre vous expliquera comment fonctionnent les extensions et comment les utiliser efficacement.

---

## Qu'est-ce qu'une Extension PostgreSQL ?

### Définition Simple

Une **extension** est un module additionnel qui ajoute de nouvelles fonctionnalités à PostgreSQL. C'est comparable aux plugins d'un navigateur web ou aux applications sur un smartphone : le système de base fonctionne parfaitement seul, mais vous pouvez l'enrichir selon vos besoins spécifiques.

### Pourquoi Utiliser des Extensions ?

Les extensions permettent de :

1. **Étendre les fonctionnalités** : Ajouter des capacités que PostgreSQL ne possède pas nativement (exemple : traitement géospatial, recherche vectorielle)  
2. **Optimiser pour des cas d'usage spécifiques** : Adapter PostgreSQL à des besoins particuliers (séries temporelles, traitement de graphes)  
3. **Maintenir un cœur stable** : Le système principal reste léger et stable, tandis que les fonctionnalités avancées sont optionnelles  
4. **Bénéficier de l'écosystème** : Profiter de milliers d'extensions développées par la communauté mondiale

### Exemples Concrets d'Extensions Populaires

- **PostGIS** : Transforme PostgreSQL en base de données géospatiale capable de gérer des coordonnées GPS, des cartes, des calculs de distances  
- **pgvector** : Permet de stocker et rechercher des vecteurs (embeddings) pour des applications d'intelligence artificielle  
- **pg_stat_statements** : Collecte des statistiques sur toutes les requêtes SQL exécutées (essentiel pour le monitoring)  
- **uuid-ossp** : Génère des identifiants uniques universels (UUID)  
- **hstore** : Permet de stocker des paires clé-valeur directement dans une colonne

---

## Architecture du Système d'Extensions

### Où Sont Stockées les Extensions ?

Le système d'extensions PostgreSQL repose sur plusieurs composants :

#### 1. Les Fichiers d'Extension
Les extensions sont constituées de fichiers installés sur le serveur PostgreSQL :

- **Fichiers de contrôle (.control)** : Contiennent les métadonnées de l'extension (nom, version, dépendances)  
- **Scripts SQL (.sql)** : Définissent les objets créés par l'extension (fonctions, types, tables)  
- **Bibliothèques partagées (.so ou .dll)** : Contiennent du code compilé pour les fonctions performantes

**Emplacement typique** : `/usr/share/postgresql/<version>/extension/`

#### 2. Le Catalogue Système
Une fois installée dans une base de données, l'extension est enregistrée dans le catalogue système :

- **pg_extension** : Liste des extensions installées dans la base  
- **pg_available_extensions** : Extensions disponibles sur le serveur

#### 3. La Portée d'une Extension

**Point important** : Une extension s'installe **par base de données**. Si vous avez trois bases de données différentes, vous devrez installer l'extension dans chacune d'elles séparément.

```
Instance PostgreSQL
├── Base de données "production"
│   ├── Extension PostGIS ✓ (installée)
│   └── Extension pgvector ✗ (non installée)
├── Base de données "development"
│   ├── Extension PostGIS ✗ (non installée)
│   └── Extension pgvector ✓ (installée)
└── Base de données "analytics"
    ├── Extension PostGIS ✓ (installée)
    └── Extension pgvector ✓ (installée)
```

---

## Utilisation des Extensions : Les Commandes Essentielles

### 1. Lister les Extensions Disponibles

Avant d'installer une extension, il faut savoir ce qui est disponible sur le serveur.

**Requête SQL** :
```sql
SELECT name, default_version, comment  
FROM pg_available_extensions  
ORDER BY name;  
```

**Ce que vous verrez** :
```
name              | default_version | comment
------------------+-----------------+------------------------------------------
adminpack         | 2.1             | administrative functions for PostgreSQL  
bloom             | 1.0             | bloom access method - signature file...  
btree_gin         | 1.3             | support for indexing common datatypes...  
hstore            | 1.8             | data type for storing sets of key/value...  
pg_stat_statements| 1.11            | track execution statistics of all SQL...  
pgcrypto          | 1.3             | cryptographic functions  
postgis           | 3.5.0           | PostGIS geometry and geography spatial...  
uuid-ossp         | 1.1             | generate universally unique identifiers  
```

### 2. Installer une Extension

**Syntaxe de base** :
```sql
CREATE EXTENSION nom_extension;
```

**Exemple simple** :
```sql
CREATE EXTENSION uuid-ossp;
```

**Avec options** :
```sql
-- Installation dans un schéma spécifique
CREATE EXTENSION hstore SCHEMA public;

-- Installation d'une version précise
CREATE EXTENSION postgis VERSION '3.5.0';

-- Installation conditionnelle (ne génère pas d'erreur si déjà présente)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

**Qui peut installer une extension ?**
Seuls les **superutilisateurs** (ou les utilisateurs avec le privilège `CREATE` sur la base) peuvent installer des extensions. C'est une mesure de sécurité importante.

### 3. Vérifier les Extensions Installées

**Requête SQL** :
```sql
SELECT extname AS "Extension",
       extversion AS "Version",
       nspname AS "Schema"
FROM pg_extension e  
JOIN pg_namespace n ON e.extnamespace = n.oid  
WHERE extname != 'plpgsql'  -- plpgsql est toujours présent par défaut  
ORDER BY extname;  
```

**Résultat typique** :
```
Extension          | Version | Schema
-------------------+---------+--------
hstore             | 1.8     | public  
pg_stat_statements | 1.11    | public  
uuid-ossp          | 1.1     | public  
```

**Commande alternative dans psql** :
```
\dx
```

### 4. Mettre à Jour une Extension

Les extensions évoluent avec le temps. Pour bénéficier des nouvelles fonctionnalités et corrections :

```sql
-- Mise à jour vers la dernière version disponible
ALTER EXTENSION pg_stat_statements UPDATE;

-- Mise à jour vers une version spécifique
ALTER EXTENSION postgis UPDATE TO '3.5.1';
```

**Attention** : Lisez toujours les notes de version avant de mettre à jour, certaines mises à jour peuvent contenir des changements incompatibles.

### 5. Désinstaller une Extension

**Syntaxe de base** :
```sql
DROP EXTENSION nom_extension;
```

**Avec gestion des dépendances** :
```sql
-- Suppression avec tous les objets dépendants (DANGEREUX !)
DROP EXTENSION postgis CASCADE;

-- Suppression conditionnelle (ne génère pas d'erreur si absente)
DROP EXTENSION IF EXISTS old_extension;
```

**⚠️ Avertissement** : La clause `CASCADE` supprimera TOUS les objets qui dépendent de l'extension (tables, fonctions, etc.). Soyez extrêmement prudent !

---

## Comprendre les Dépendances d'Extensions

### Qu'est-ce qu'une Dépendance ?

Certaines extensions ont besoin d'autres extensions pour fonctionner. C'est ce qu'on appelle une **dépendance**.

**Exemple réel** : L'extension **postgis_topology** dépend de **postgis**. Vous devez d'abord installer postgis avant de pouvoir installer postgis_topology.

### Installation Automatique des Dépendances

PostgreSQL gère automatiquement les dépendances lors de l'installation :

```sql
-- Tente d'installer postgis_topology
CREATE EXTENSION postgis_topology;
```

**Si postgis n'est pas installé**, PostgreSQL retournera une erreur :
```
ERROR: required extension "postgis" is not installed  
HINT: Use CREATE EXTENSION ... CASCADE to install required extensions too.  
```

**Solution avec CASCADE** :
```sql
CREATE EXTENSION postgis_topology CASCADE;
-- Cela installera automatiquement postgis, puis postgis_topology
```

### Visualiser les Dépendances

```sql
SELECT e1.extname AS "Extension",
       e2.extname AS "Depends on"
FROM pg_extension e1  
LEFT JOIN pg_depend d ON d.objid = e1.oid AND d.deptype = 'e'  
LEFT JOIN pg_extension e2 ON d.refobjid = e2.oid  
WHERE e1.extname != 'plpgsql'  
ORDER BY e1.extname;  
```

---

## Les Schémas et les Extensions

### Pourquoi les Schémas Comptent ?

Lorsqu'une extension est installée, tous ses objets (fonctions, tables, types) sont créés dans un **schéma**. Par défaut, c'est le schéma `public`, mais vous pouvez choisir.

### Installation dans un Schéma Spécifique

**Cas d'usage** : Vous voulez isoler les objets d'une extension dans un schéma dédié pour une meilleure organisation.

```sql
-- Créer d'abord le schéma
CREATE SCHEMA extensions;

-- Installer l'extension dans ce schéma
CREATE EXTENSION hstore SCHEMA extensions;
```

**Accès aux fonctions** :
```sql
-- Avec préfixe de schéma
SELECT extensions.hstore('key1', 'value1');

-- Ou en ajustant le search_path
SET search_path TO public, extensions;  
SELECT hstore('key1', 'value1');  
```

### Extensions et Relocatabilité

Certaines extensions sont **relocatables** : vous pouvez les déplacer d'un schéma à un autre après installation.

```sql
-- Déplacer une extension relocatable
ALTER EXTENSION hstore SET SCHEMA my_schema;
```

**Extensions non-relocatables** : Certaines extensions (comme PostGIS) créent tellement d'objets interconnectés qu'elles ne peuvent pas être déplacées. Cette information est visible dans le fichier `.control` de l'extension.

---

## Extensions Intégrées vs Extensions Externes

### Extensions Contrib (Intégrées)

PostgreSQL inclut un ensemble d'extensions appelées **contrib** (contribution). Elles sont maintenues par l'équipe PostgreSQL et installées automatiquement avec le serveur (mais pas activées par défaut).

**Exemples d'extensions contrib populaires** :

- **pg_stat_statements** : Statistiques de requêtes  
- **pgcrypto** : Fonctions cryptographiques  
- **btree_gin**, **btree_gist** : Types d'index avancés  
- **uuid-ossp** : Génération d'UUID  
- **hstore** : Stockage clé-valeur  
- **citext** : Type texte insensible à la casse  
- **pg_trgm** : Recherche de similarité de texte (trigrams)

**Activation** :
```sql
CREATE EXTENSION pg_stat_statements;
-- Aucune installation système supplémentaire nécessaire
```

### Extensions Tierces (Externes)

Ce sont des extensions développées par la communauté ou des entreprises tierces. Elles nécessitent une installation système avant de pouvoir être utilisées.

**Exemples d'extensions tierces majeures** :

- **PostGIS** : Géospatial  
- **TimescaleDB** : Séries temporelles  
- **pgvector** : Recherche vectorielle/IA  
- **Citus** : Distribution et parallélisation  
- **pg_partman** : Gestion automatisée de partitions

**Installation en deux étapes** :

1. **Installation système** (avec apt, yum, ou depuis les sources) :
```bash
# Exemple Ubuntu/Debian
sudo apt install postgresql-16-postgis-3
```

2. **Activation dans la base** :
```sql
CREATE EXTENSION postgis;
```

---

## Vérification et Exploration d'une Extension

### Découvrir les Objets Créés

Une fois une extension installée, vous pouvez explorer ce qu'elle a créé.

**Lister les fonctions d'une extension** :
```sql
SELECT p.proname AS "Function Name",
       pg_get_function_identity_arguments(p.oid) AS "Arguments"
FROM pg_proc p  
JOIN pg_depend d ON d.objid = p.oid  
JOIN pg_extension e ON d.refobjid = e.oid  
WHERE e.extname = 'uuid-ossp'  
  AND d.deptype = 'e'
ORDER BY p.proname;
```

**Résultat pour uuid-ossp** :
```
Function Name      | Arguments
-------------------+------------------
uuid_generate_v1   |  
uuid_generate_v1mc |  
uuid_generate_v3   | namespace uuid, name text  
uuid_generate_v4   |  
uuid_generate_v5   | namespace uuid, name text  
```

**Commande psql simplifiée** :
```sql
\dx+ uuid-ossp
```

### Tester une Extension

**Exemple avec uuid-ossp** :
```sql
-- Génération d'un UUID v4 (aléatoire)
SELECT uuid_generate_v4();

-- Résultat typique :
--  uuid_generate_v4
-- --------------------------------------
--  a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11
```

**Exemple avec pg_trgm (similarité de texte)** :
```sql
CREATE EXTENSION pg_trgm;

-- Calculer la similarité entre deux chaînes
SELECT similarity('PostgreSQL', 'Postgres');
-- Résultat : 0.55555557 (55% de similarité)

-- Trouver des textes similaires
SELECT word  
FROM words_table  
WHERE word % 'PostgreSQL'  -- Opérateur de similarité  
ORDER BY similarity(word, 'PostgreSQL') DESC;  
```

---

## Bonnes Pratiques

### 1. Installer Uniquement ce Dont Vous Avez Besoin

**Principe** : N'installez que les extensions que vous utilisez réellement. Chaque extension ajoute des objets à votre base et peut avoir un impact sur la maintenance.

❌ **À éviter** :
```sql
-- Installer toutes les extensions "au cas où"
CREATE EXTENSION postgis;  
CREATE EXTENSION timescaledb;  
CREATE EXTENSION pgvector;  
CREATE EXTENSION pg_partman;  
-- ... alors que vous n'utilisez aucune de ces fonctionnalités
```

✅ **Recommandé** :
```sql
-- Installer uniquement ce qui est nécessaire pour votre application
CREATE EXTENSION IF NOT EXISTS pg_stat_statements; -- Monitoring  
CREATE EXTENSION IF NOT EXISTS uuid-ossp; -- Génération d'identifiants  
```

### 2. Utiliser IF NOT EXISTS

Cette clause évite les erreurs si l'extension est déjà installée, ce qui est utile dans les scripts de déploiement.

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- Pas d'erreur même si déjà installé
```

### 3. Documenter les Extensions Utilisées

Maintenez une documentation claire des extensions installées dans chaque base :

```sql
-- Script de documentation (à exécuter)
SELECT
    extname AS "Extension",
    extversion AS "Version",
    nspname AS "Schema",
    CASE
        WHEN extname IN ('pg_stat_statements', 'uuid-ossp', 'pgcrypto')
        THEN 'Built-in (contrib)'
        ELSE 'External'
    END AS "Type"
FROM pg_extension e  
JOIN pg_namespace n ON e.extnamespace = n.oid  
WHERE extname != 'plpgsql'  
ORDER BY extname;  
```

**Créez également un fichier README** listant :
- Les extensions utilisées
- Leur rôle dans votre application
- Les versions compatibles
- Les instructions d'installation système si nécessaire

### 4. Tester les Mises à Jour dans un Environnement de Test

Avant de mettre à jour une extension en production :

1. Créez un environnement de test identique  
2. Testez la mise à jour  
3. Vérifiez que toutes vos requêtes fonctionnent toujours  
4. Lisez le changelog de l'extension  
5. Seulement ensuite, appliquez en production

### 5. Gérer les Extensions dans le Contrôle de Version

**Incluez dans votre dépôt Git** :
```
migrations/
  ├── 001_create_schema.sql
  ├── 002_install_extensions.sql  ← Liste des extensions
  ├── 003_create_tables.sql
  └── ...
```

**Exemple de 002_install_extensions.sql** :
```sql
-- Extensions nécessaires pour l'application
-- Version: 1.0
-- Date: 2025-11-22

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";  
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";  
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  

-- PostGIS (nécessite installation système préalable)
-- sudo apt install postgresql-18-postgis-3
CREATE EXTENSION IF NOT EXISTS "postgis";
```

### 6. Attention aux Extensions Expérimentales

Certaines extensions sont en développement actif et peuvent :
- Changer leur API
- Introduire des bugs
- Ne pas avoir de support de migration

**Évaluez avant d'adopter** :
- Maturité du projet (âge, versions stables)
- Activité de la communauté (GitHub commits, issues)
- Documentation disponible
- Tests et couverture de code
- Adoption dans l'industrie

---

## Comprendre les Versions d'Extensions

### Format de Versionnement

Les extensions suivent généralement le **Semantic Versioning** (SemVer) : `MAJEUR.MINEUR.PATCH`

- **MAJEUR** : Changements incompatibles avec les versions précédentes  
- **MINEUR** : Nouvelles fonctionnalités, rétrocompatibles  
- **PATCH** : Corrections de bugs, rétrocompatibles

**Exemple** : `postgis 3.5.1`
- 3 = version majeure
- 5 = version mineure
- 1 = patch

### Scripts de Mise à Jour

Les extensions incluent des scripts de migration entre versions :

```
/usr/share/postgresql/16/extension/
  ├── postgis--3.5.0.sql          # Installation initiale v3.5.0
  ├── postgis--3.5.0--3.5.1.sql   # Migration 3.5.0 → 3.5.1
  ├── postgis--3.5.1--3.5.2.sql   # Migration 3.5.1 → 3.5.2
  └── postgis.control             # Métadonnées
```

PostgreSQL utilise automatiquement ces scripts lors d'un `ALTER EXTENSION UPDATE`.

### Vérifier la Version Installée

```sql
SELECT extname, extversion  
FROM pg_extension  
WHERE extname = 'postgis';  
```

**Comparer avec la version disponible** :
```sql
SELECT name, installed_version, default_version  
FROM pg_available_extensions  
WHERE name = 'postgis';  
```

---

## Sécurité et Extensions

### Principes de Sécurité

Les extensions peuvent exécuter du code arbitraire sur le serveur. C'est pourquoi :

1. **Seuls les superutilisateurs** peuvent installer des extensions (par défaut)  
2. **Les extensions de confiance** peuvent être installées par des non-superutilisateurs dans PostgreSQL 13+  
3. **Auditez le code** des extensions tierces avant installation en production

### Extensions de Confiance (Trusted Extensions)

Depuis PostgreSQL 13, certaines extensions sont marquées comme "trusted" et peuvent être installées par des utilisateurs avec le privilège `CREATE` sur la base.

**Vérifier si une extension est trusted** :
```sql
SELECT name, trusted  
FROM pg_available_extensions  
WHERE name = 'pg_stat_statements';  
```

**Extensions trusted communes** :
- pg_stat_statements
- pgcrypto
- uuid-ossp
- pg_trgm
- btree_gin

### Recommandations de Sécurité

1. **Utilisez uniquement des sources fiables** : Extensions officielles contrib, PGXN (PostgreSQL Extension Network), dépôts GitHub bien maintenus  
2. **Vérifiez les signatures** quand disponibles  
3. **Lisez le code source** des extensions tierces (surtout si elles contiennent du C)  
4. **Testez en isolation** avant la production  
5. **Suivez les CVE** (Common Vulnerabilities and Exposures) des extensions que vous utilisez

---

## Dépannage Courant

### Problème 1 : Extension Non Disponible

**Symptôme** :
```sql
CREATE EXTENSION postgis;
-- ERROR: could not open extension control file "/usr/share/postgresql/16/extension/postgis.control": No such file or directory
```

**Cause** : L'extension n'est pas installée au niveau système.

**Solution** :
```bash
# Ubuntu/Debian
sudo apt install postgresql-16-postgis-3

# Red Hat/CentOS
sudo yum install postgis35_16

# macOS (Homebrew)
brew install postgis
```

### Problème 2 : Conflit de Versions

**Symptôme** :
```sql
ALTER EXTENSION postgis UPDATE;
-- ERROR: extension "postgis" has no update path from version "3.5.0" to version "3.6.0"
```

**Cause** : Le script de migration entre ces deux versions n'existe pas.

**Solution** : Mise à jour incrémentale ou réinstallation
```sql
-- Option 1 : Vérifier les chemins disponibles
SELECT * FROM pg_extension_update_paths('postgis')  
WHERE source = '3.5.0';  

-- Option 2 : Mise à jour par étapes
ALTER EXTENSION postgis UPDATE TO '3.5.1';  
ALTER EXTENSION postgis UPDATE TO '3.5.2';  
-- ... jusqu'à la version souhaitée
```

### Problème 3 : Dépendances Non Satisfaites

**Symptôme** :
```sql
DROP EXTENSION postgis;
-- ERROR: cannot drop extension postgis because other objects depend on it
-- DETAIL: extension postgis_topology depends on extension postgis
```

**Solution** :
```sql
-- Option 1 : Supprimer les dépendances d'abord
DROP EXTENSION postgis_topology;  
DROP EXTENSION postgis;  

-- Option 2 : Utiliser CASCADE (ATTENTION : supprime TOUT)
DROP EXTENSION postgis CASCADE;
```

### Problème 4 : Extension Manquante Après Restauration

**Symptôme** : Après un `pg_restore`, les fonctions d'extension ne fonctionnent plus.

**Cause** : `pg_dump` n'inclut pas les objets d'extension (par design), il génère des commandes `CREATE EXTENSION`.

**Solution** : Assurez-vous que les extensions soient installées au niveau système sur le serveur cible avant de restaurer.

```bash
# 1. Sur le serveur cible, installer les extensions système
sudo apt install postgresql-16-postgis-3

# 2. Restaurer le dump (les CREATE EXTENSION seront exécutés)
pg_restore -d mydb backup.dump
```

---

## Extensions et Performance

### Impact sur les Performances

**Extensions légères** (peu d'impact) :
- uuid-ossp : Juste quelques fonctions
- pgcrypto : Utilisé à la demande
- pg_trgm : Index supplémentaires uniquement si créés

**Extensions avec impact** :
- **pg_stat_statements** : Surcharge de 5-10% pour collecter toutes les statistiques (mais indispensable pour le monitoring)  
- **TimescaleDB** : Restructure le stockage des données (hypertables)  
- **PostGIS** : Ajoute des milliers de fonctions et types

### Optimiser l'Utilisation des Extensions

1. **pg_stat_statements** : Configurez le nombre de requêtes suivies
```sql
-- Dans postgresql.conf
pg_stat_statements.max = 10000  -- Nombre max de requêtes distinctes suivies  
pg_stat_statements.track = top  -- 'all', 'top', 'none'  
```

2. **PostGIS** : Utilisez les index spatiaux
```sql
CREATE INDEX idx_spatial ON my_table USING GIST(geom);
```

3. **pgvector** : Choisissez le bon type d'index selon vos besoins
```sql
-- Pour petites données : index exact
CREATE INDEX ON items USING ivfflat (embedding vector_cosine_ops);

-- Pour grandes données : index approximatif (HNSW)
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops);
```

---

## Exemples d'Utilisation Réels

### Exemple 1 : UUID pour les Identifiants

**Problème** : Vous voulez des identifiants uniques distribués sans coordination.

**Solution** :
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Créer une table avec identifiant UUID
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertion automatique d'UUID
INSERT INTO users (username, email)  
VALUES ('alice', 'alice@example.com');  

SELECT id FROM users;
-- Résultat : a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11
```

### Exemple 2 : Recherche Approximative avec pg_trgm

**Problème** : Recherche de produits avec typos ou variations.

**Solution** :
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Table de produits
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

-- Index pour recherche de similarité
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- Recherche floue
SELECT name, similarity(name, 'PostgreSQL') AS sim  
FROM products  
WHERE name % 'PostgreSQL'  -- Opérateur de similarité  
ORDER BY sim DESC  
LIMIT 10;  
```

### Exemple 3 : Cryptographie avec pgcrypto

**Problème** : Hacher des mots de passe de manière sécurisée.

**Solution** :
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Table utilisateurs avec mot de passe haché
CREATE TABLE secure_users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    password_hash TEXT NOT NULL
);

-- Insérer un utilisateur avec mot de passe haché
INSERT INTO secure_users (username, password_hash)  
VALUES ('bob', crypt('my_secure_password', gen_salt('bf')));  -- bcrypt  

-- Vérifier un mot de passe
SELECT username  
FROM secure_users  
WHERE username = 'bob'  
  AND password_hash = crypt('my_secure_password', password_hash);
-- Retourne 'bob' si le mot de passe est correct
```

---

## Ressources Additionnelles

### Documentation Officielle

- **Manuel PostgreSQL** : https://www.postgresql.org/docs/current/extend-extensions.html  
- **PGXN (PostgreSQL Extension Network)** : https://pgxn.org/  
- **Liste contrib** : https://www.postgresql.org/docs/current/contrib.html

### Extensions Incontournables à Connaître

| Extension | Catégorie | Description |
|-----------|-----------|-------------|
| pg_stat_statements | Monitoring | Statistiques de requêtes |
| PostGIS | Géospatial | Données géographiques |
| pgvector | IA/ML | Recherche vectorielle |
| TimescaleDB | Séries temporelles | Optimisation time-series |
| pg_trgm | Recherche texte | Similarité et recherche floue |
| uuid-ossp | Identifiants | Génération UUID |
| pgcrypto | Sécurité | Cryptographie |
| hstore | NoSQL | Clé-valeur dans colonnes |
| pg_cron | Automatisation | Planification de tâches |
| pg_partman | Maintenance | Gestion partitions |

### Communauté et Support

- **Mailing list pgsql-general** : Pour poser des questions  
- **Stack Overflow** : Tag [postgresql] + [postgresql-extensions]  
- **GitHub** : Beaucoup d'extensions ont leur dépôt officiel  
- **Reddit** : r/PostgreSQL

---

## Conclusion

Le système d'extensions de PostgreSQL est l'une des raisons majeures de son succès. Il permet :

- ✅ **Flexibilité** : Adaptez PostgreSQL à vos besoins précis  
- ✅ **Stabilité** : Le cœur reste stable, les innovations se font en périphérie  
- ✅ **Communauté** : Des milliers de développeurs contribuent des extensions  
- ✅ **Performance** : N'activez que ce dont vous avez besoin

**Principes à retenir** :

1. Les extensions s'installent **par base de données**  
2. Seuls les **superutilisateurs** peuvent les installer (sauf trusted extensions)  
3. Utilisez `IF NOT EXISTS` dans vos scripts  
4. Documentez les extensions utilisées  
5. Testez les mises à jour avant la production  
6. N'installez que ce dont vous avez réellement besoin

Dans les prochains chapitres, nous explorerons en détail certaines extensions majeures comme **PostGIS** (18.2), **Full-Text Search** (18.3) et **pgvector** (18.6).

---


⏭️ [PostGIS : La référence spatiale mondiale](/18-extensions-et-integrations/02-postgis.md)
