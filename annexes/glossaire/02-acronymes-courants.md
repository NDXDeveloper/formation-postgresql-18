🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Guide des Acronymes PostgreSQL Courants

## Introduction

PostgreSQL, comme tout domaine technique, utilise de nombreux acronymes qui peuvent sembler intimidants au premier abord. Ce guide vous aide à comprendre leur signification et leur contexte d'utilisation, avec des explications accessibles aux débutants.

Les acronymes sont classés par catégorie pour faciliter votre apprentissage.

---

## Acronymes Fondamentaux

### RDBMS
**Relational Database Management System**
(Système de Gestion de Base de Données Relationnelles)

**Définition** : Un logiciel qui gère des bases de données organisées en tables avec des relations entre elles.

**Exemple** : PostgreSQL, MySQL, Oracle, SQL Server sont des RDBMS.

**À retenir** : PostgreSQL est en fait un ORDBMS (Object-Relational), car il supporte aussi des concepts orientés objet.

---

### SQL
**Structured Query Language**
(Langage de Requête Structuré)

**Définition** : Le langage standardisé pour interagir avec les bases de données relationnelles.

**Les 4 catégories principales** :
- **DDL** : Définir la structure  
- **DML** : Manipuler les données  
- **DQL** : Interroger les données  
- **DCL** : Contrôler les permissions

**Exemple** :
```sql
SELECT nom FROM clients WHERE ville = 'Paris';
```

---

### ACID
**Atomicity, Consistency, Isolation, Durability**
(Atomicité, Cohérence, Isolation, Durabilité)

**Définition** : Les quatre propriétés garantissant la fiabilité des transactions.

- **A**tomicity : Tout ou rien  
- **C**onsistency : Toujours conforme aux règles  
- **I**solation : Les transactions ne s'interfèrent pas  
- **D**urability : Les données validées sont permanentes

**Importance** : ACID différencie les "vraies" bases de données des simples systèmes de stockage.

---

## Langages SQL

### DDL
**Data Definition Language**
(Langage de Définition de Données)

**Définition** : Commandes pour créer, modifier ou supprimer la structure de la base.

**Commandes principales** :
- `CREATE` : Créer des objets  
- `ALTER` : Modifier des objets  
- `DROP` : Supprimer des objets  
- `TRUNCATE` : Vider une table

**Exemple** :
```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

ALTER TABLE employes ADD COLUMN email VARCHAR(255);

DROP TABLE ancienne_table;
```

---

### DML
**Data Manipulation Language**
(Langage de Manipulation de Données)

**Définition** : Commandes pour insérer, modifier ou supprimer des données.

**Commandes principales** :
- `INSERT` : Ajouter des lignes  
- `UPDATE` : Modifier des lignes  
- `DELETE` : Supprimer des lignes  
- `MERGE` : Fusionner des données (PG 15+)

**Exemple** :
```sql
INSERT INTO employes (nom, email) VALUES ('Alice', 'alice@example.com');

UPDATE employes SET email = 'alice.new@example.com' WHERE nom = 'Alice';

DELETE FROM employes WHERE id = 99;
```

---

### DQL
**Data Query Language**
(Langage d'Interrogation de Données)

**Définition** : Parfois utilisé pour désigner spécifiquement `SELECT`, distinct du DML.

**Commande** :
- `SELECT` : Lire et récupérer des données

**Exemple** :
```sql
SELECT nom, email  
FROM employes  
WHERE ville = 'Lyon'  
ORDER BY nom;  
```

**Note** : Certains considèrent DQL comme faisant partie du DML. La distinction n'est pas universelle.

---

### DCL
**Data Control Language**
(Langage de Contrôle de Données)

**Définition** : Commandes pour gérer les permissions et la sécurité.

**Commandes principales** :
- `GRANT` : Donner des permissions  
- `REVOKE` : Retirer des permissions

**Exemple** :
```sql
-- Donner le droit de lecture
GRANT SELECT ON TABLE clients TO user_reporting;

-- Retirer le droit d'écriture
REVOKE INSERT, UPDATE, DELETE ON TABLE clients FROM user_readonly;
```

---

### TCL
**Transaction Control Language**
(Langage de Contrôle de Transaction)

**Définition** : Commandes pour gérer les transactions.

**Commandes principales** :
- `BEGIN` : Démarrer une transaction  
- `COMMIT` : Valider une transaction  
- `ROLLBACK` : Annuler une transaction  
- `SAVEPOINT` : Créer un point de sauvegarde

**Exemple** :
```sql
BEGIN;
    INSERT INTO commandes (client_id, montant) VALUES (1, 150.00);
    UPDATE clients SET total_achats = total_achats + 150 WHERE id = 1;
COMMIT;
```

---

## Contraintes et Clés

### PK
**Primary Key**
(Clé Primaire)

**Définition** : Une colonne (ou ensemble de colonnes) qui identifie de manière unique chaque ligne d'une table.

**Règles** :
- Toujours unique
- Jamais NULL
- Une seule PK par table

**Exemple** :
```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,  -- PK automatique
    nom VARCHAR(100)
);

-- Ou PK composite
CREATE TABLE commande_produits (
    commande_id INTEGER,
    produit_id INTEGER,
    PRIMARY KEY (commande_id, produit_id)
);
```

---

### FK
**Foreign Key**
(Clé Étrangère)

**Définition** : Une colonne qui référence la clé primaire d'une autre table, créant ainsi une relation.

**But** : Garantir l'intégrité référentielle (pas de références orphelines).

**Exemple** :
```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(id)
);
```

**Actions possibles** :
- `ON DELETE CASCADE` : Supprime les lignes liées  
- `ON DELETE SET NULL` : Met à NULL les références  
- `ON DELETE RESTRICT` : Empêche la suppression (défaut)

---

### UK
**Unique Key**
(Clé Unique)

**Définition** : Une contrainte garantissant que les valeurs d'une colonne sont uniques.

**Différence avec PK** :
- Peut être NULL
- Plusieurs UK possibles par table

**Exemple** :
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE,  -- UK
    username VARCHAR(50) UNIQUE  -- Autre UK
);
```

---

## Architecture et Mécanismes Internes

### MVCC
**Multiversion Concurrency Control**
(Contrôle de Concurrence Multi-Versions)

**Définition** : Le mécanisme de PostgreSQL pour gérer des milliers de connexions simultanées sans blocage.

**Principe** : Créer une nouvelle version de chaque ligne modifiée au lieu de la verrouiller.

**Avantage majeur** :
- Les lectures ne bloquent jamais les écritures
- Les écritures ne bloquent jamais les lectures
- Excellente performance en environnement multi-utilisateurs

**Contrepartie** : Nécessite VACUUM pour nettoyer les anciennes versions.

---

### WAL
**Write-Ahead Log**
(Journal de Pré-Écriture)

**Définition** : Un journal qui enregistre toutes les modifications AVANT qu'elles ne soient appliquées aux fichiers de données.

**Utilités** :
1. **Durabilité** : Récupération après crash  
2. **Performance** : Écriture séquentielle rapide  
3. **Réplication** : Envoi du WAL vers des standby

**Localisation** : Répertoire `pg_wal/` (anciennement `pg_xlog/`)

**Taille typique** : Fichiers de 16 Mo

---

### TOAST
**The Oversized-Attribute Storage Technique**
(Technique de Stockage d'Attributs Surdimensionnés)

**Définition** : Mécanisme automatique pour gérer les valeurs trop grandes (> 2 Ko).

**Actions** :
- Compression des grandes valeurs
- Déplacement dans une table TOAST séparée si nécessaire

**Types concernés** : TEXT, VARCHAR, BYTEA, JSON, JSONB, ARRAY

**Transparent** : L'utilisateur n'a rien à faire, c'est automatique.

---

### XID
**Transaction ID**
(Identifiant de Transaction)

**Définition** : Un numéro unique sur 32 bits attribué à chaque transaction.

**Problème** : Après 4 milliards de transactions, risque de "wraparound".

**Solution** : VACUUM gèle (freeze) les anciennes transactions pour éviter ce problème.

**Monitoring** : Surveiller l'âge des transactions avec `age(datfrozenxid)`.

---

### TID
**Tuple ID**
(Identifiant de Tuple)

**Définition** : L'adresse physique d'une ligne (page, offset) dans la base.

**Format** : `(page_number, item_number)` - Ex: `(0,1)`

**Usage** : Rarement manipulé directement, sauf pour le debugging avancé.

**Accès** :
```sql
SELECT ctid, * FROM ma_table;
```

---

### CTID
**Current Tuple ID**
(Identifiant de Tuple Actuel)

**Définition** : Une colonne système disponible dans toutes les tables, contenant le TID de la ligne.

**Usage** : Debugging, identification physique des lignes.

**Exemple** :
```sql
-- Trouver la position physique d'une ligne
SELECT ctid, id, nom FROM produits WHERE id = 100;
-- Résultat : (15,3) | 100 | 'Ordinateur'
```

**Attention** : Le CTID peut changer après un VACUUM FULL ou UPDATE.

---

## Types d'Index

### B-Tree
**Balanced Tree**
(Arbre Équilibré)

**Définition** : Le type d'index par défaut et le plus polyvalent.

**Structure** : Arbre équilibré permettant des recherches en temps logarithmique O(log n).

**Cas d'usage** :
- Égalité : `WHERE id = 123`
- Comparaisons : `WHERE age > 25`, `WHERE date BETWEEN ...`
- Tri : `ORDER BY nom`
- Recherche de préfixes : `WHERE nom LIKE 'Dup%'`

**Création** :
```sql
CREATE INDEX idx_nom ON clients(nom);  -- B-Tree par défaut
```

---

### GIN
**Generalized Inverted Index**
(Index Inversé Généralisé)

**Définition** : Index spécialisé pour les types de données composés (tableaux, JSONB, full-text).

**Cas d'usage** :
- **Full-Text Search** : Recherche dans du texte  
- **JSONB** : Recherche dans des documents JSON  
- **Arrays** : Recherche dans des tableaux  
- **hstore** : Paires clé-valeur

**Exemple** :
```sql
-- Index GIN pour JSONB
CREATE INDEX idx_data_gin ON produits USING GIN(data);

-- Recherche efficace
SELECT * FROM produits WHERE data @> '{"categorie": "electronique"}';
```

**Caractéristique** : Plus lourd mais indispensable pour ces cas.

---

### GiST
**Generalized Search Tree**
(Arbre de Recherche Généralisé)

**Définition** : Framework d'index flexible pour des types de données non-standards.

**Cas d'usage** :
- **PostGIS** : Géométries (points, polygones)  
- **Full-Text Search** : Alternative à GIN  
- **Range types** : `int4range`, `tstzrange`  
- **ltree** : Hiérarchies arborescentes  
- **Recherche de similarité** : pg_trgm

**Exemple spatial** :
```sql
CREATE INDEX idx_location ON restaurants USING GiST(location);

-- Trouver les restaurants dans un rayon
SELECT * FROM restaurants  
WHERE ST_DWithin(location, ST_Point(2.3522, 48.8566), 1000);  
```

---

### BRIN
**Block Range INdex**
(Index de Plage de Blocs)

**Définition** : Index ultra-léger qui stocke seulement les valeurs min/max par bloc de données.

**Idéal pour** :
- Tables massives (> 100 Go)
- Données ordonnées naturellement (logs chronologiques, IoT)
- Colonnes avec forte corrélation physique

**Avantages** :
- Index minuscule (quelques Mo pour des To de données)
- Très rapide à créer et maintenir

**Exemple** :
```sql
CREATE INDEX idx_logs_timestamp ON logs USING BRIN(timestamp);
```

**Quand NE PAS utiliser** : Données non ordonnées ou petites tables.

---

### SP-GiST
**Space-Partitioned Generalized Search Tree**
(Arbre de Recherche Généralisé à Partition d'Espace)

**Définition** : Variante de GiST pour des structures non-équilibrées.

**Cas d'usage** :
- Données géographiques avec clustering irrégulier
- Quadtrees, k-d trees
- Structures de données spécialisées

**Usage** : Plus rare que B-Tree, GIN ou GiST.

---

## Requêtes et Expressions

### CTE
**Common Table Expression**
(Expression de Table Commune)

**Définition** : Une requête temporaire nommée, réutilisable dans une requête principale.

**Syntaxe** : Utilise la clause `WITH`

**Avantages** :
- Améliore la lisibilité
- Permet la récursion
- Peut être matérialisée pour la performance

**Exemple simple** :
```sql
WITH ventes_2024 AS (
    SELECT
        produit_id,
        SUM(quantite) as total_vendu
    FROM commandes
    WHERE date >= '2024-01-01'
    GROUP BY produit_id
)
SELECT
    p.nom,
    v.total_vendu
FROM ventes_2024 v  
JOIN produits p ON v.produit_id = p.id  
ORDER BY v.total_vendu DESC  
LIMIT 10;  
```

**CTE récursive** :
```sql
WITH RECURSIVE subordinates AS (
    -- Cas de base
    SELECT id, nom, manager_id
    FROM employes
    WHERE id = 1

    UNION ALL

    -- Cas récursif
    SELECT e.id, e.nom, e.manager_id
    FROM employes e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

---

### EXPLAIN
**Explain Query Plan**
(Expliquer le Plan de Requête)

**Définition** : Commande qui affiche comment PostgreSQL va exécuter une requête.

**Variantes** :
- `EXPLAIN` : Plan théorique  
- `EXPLAIN ANALYZE` : Exécute et montre les temps réels  
- `EXPLAIN (BUFFERS, ANALYZE)` : Ajoute les statistiques d'I/O

**Exemple** :
```sql
EXPLAIN ANALYZE  
SELECT * FROM clients WHERE ville = 'Paris';  
```

**Lecture** : Identifier les scans séquentiels coûteux, vérifier l'utilisation d'index.

---

### VACUUM
**Vacuum (pas un acronyme !)**

**Définition** : Processus de nettoyage qui récupère l'espace des lignes mortes (dead tuples).

**Variantes** :
- `VACUUM` : Nettoyage standard  
- `VACUUM FULL` : Réorganisation complète (bloque la table)  
- `VACUUM ANALYZE` : Nettoyage + mise à jour des statistiques  
- `autovacuum` : Automatique (recommandé)

**Pourquoi c'est vital** : MVCC crée des anciennes versions qui doivent être nettoyées.

**Commande** :
```sql
VACUUM ANALYZE ma_table;
```

---

### ANALYZE
**Analyze (pas un acronyme !)**

**Définition** : Collecte des statistiques sur le contenu des tables pour l'optimiseur.

**Résultat** : Meilleurs plans d'exécution (choix d'index, ordre de jointures).

**Commande** :
```sql
ANALYZE ma_table;  
ANALYZE;  -- Toutes les tables  
```

**Automatique** : `autovacuum` exécute aussi ANALYZE.

---

## Extensions et Fonctionnalités Avancées

### FDW
**Foreign Data Wrapper**
(Enveloppe de Données Étrangères)

**Définition** : Mécanisme pour accéder à des données externes comme si elles étaient locales.

**Sources supportées** :
- Autres bases PostgreSQL (`postgres_fdw`)
- MySQL (`mysql_fdw`)
- Oracle (`oracle_fdw`)
- Fichiers CSV (`file_fdw`)
- MongoDB, REST APIs, etc.

**Exemple** :
```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER serveur_distant
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'db.example.com', dbname 'production', port '5432');

CREATE FOREIGN TABLE clients_distants (
    id INTEGER,
    nom TEXT
) SERVER serveur_distant OPTIONS (table_name 'clients');

-- Utilisation normale
SELECT * FROM clients_distants WHERE id = 42;
```

**Usage** : Fédération de données, migrations, intégrations.

---

### FTS
**Full-Text Search**
(Recherche en Texte Intégral)

**Définition** : Capacité de recherche avancée dans du texte (comme un moteur de recherche).

**Types spéciaux** :
- `tsvector` : Document indexé  
- `tsquery` : Requête de recherche

**Exemple** :
```sql
-- Créer un index FTS
CREATE INDEX idx_articles_fts  
ON articles USING GIN(to_tsvector('french', contenu));  

-- Rechercher
SELECT titre, contenu  
FROM articles  
WHERE to_tsvector('french', contenu) @@ to_tsquery('french', 'postgresql & performance');  
```

**Fonctionnalités** :
- Stemming (racines des mots)
- Dictionnaires multilingues
- Ranking de pertinence
- Highlighting des résultats

---

### JSONB
**JSON Binary**
(JSON Binaire)

**Définition** : Type de données JSON optimisé, stocké en format binaire décomposé.

**Avantages sur JSON** :
- Plus rapide pour les requêtes
- Indexable avec GIN
- Opérateurs riches

**Exemple** :
```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    data JSONB
);

INSERT INTO produits (data) VALUES
('{"nom": "Laptop", "prix": 999, "specs": {"cpu": "i7", "ram": "16GB"}}');

-- Requêtes JSONB
SELECT * FROM produits WHERE data->>'nom' = 'Laptop';  
SELECT * FROM produits WHERE data->'specs'->>'ram' = '16GB';  
SELECT * FROM produits WHERE data @> '{"prix": 999}';  

-- Index GIN
CREATE INDEX idx_data ON produits USING GIN(data);
```

---

### UUID
**Universally Unique Identifier**
(Identifiant Unique Universel)

**Définition** : Identifiant de 128 bits, garanti unique globalement.

**Format** : `550e8400-e29b-41d4-a716-446655440000`

**Avantages** :
- Pas de coordination nécessaire entre serveurs
- Sécurité (pas d'énumération séquentielle)
- Fusion de bases facilitée

**Versions** :
- **UUID v4** : Aléatoire (classique)  
- **UUID v7** : Ordonné par timestamp (nouveau en PG 18)

**Exemple** :
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    nom TEXT
);
```

---

### PITR
**Point-In-Time Recovery**
(Récupération à un Point dans le Temps)

**Définition** : Capacité de restaurer la base à n'importe quel instant précis.

**Nécessite** :
- WAL archiving activé
- Sauvegarde de base (pg_basebackup)
- Fichiers WAL archivés

**Cas d'usage** :
- "Annuler" une erreur humaine (suppression accidentelle)
- Récupérer l'état avant une panne
- Audits et conformité

**Exemple** :
```bash
# Restaurer à 14h30 précises aujourd'hui
pg_ctl start -D /data/pgdata \
    -o "--recovery-target-time='2024-11-20 14:30:00'"
```

---

### HA
**High Availability**
(Haute Disponibilité)

**Définition** : Architecture garantissant un temps de disponibilité maximal (99.9%+).

**Composants PostgreSQL** :
- Réplication (streaming, logique)
- Failover automatique (Patroni, repmgr)
- Load balancing (HAProxy, PgPool-II)
- Monitoring et alerting

**Objectifs** :
- **RTO** (Recovery Time Objective) : Temps maximum de panne acceptable  
- **RPO** (Recovery Point Objective) : Perte de données maximale acceptable

---

### RLS
**Row-Level Security**
(Sécurité au Niveau des Lignes)

**Définition** : Mécanisme pour filtrer automatiquement les lignes selon l'utilisateur connecté.

**Cas d'usage** :
- Applications multi-tenant
- Données sensibles (un utilisateur ne voit que ses données)
- Conformité RGPD

**Exemple** :
```sql
-- Activer RLS sur la table
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Politique : les users ne voient que leurs documents
CREATE POLICY user_documents ON documents
    FOR SELECT
    USING (user_id = current_user_id());

-- Politique : un manager voit tout son équipe
CREATE POLICY manager_team ON documents
    FOR SELECT
    USING (
        team_id IN (
            SELECT team_id FROM teams WHERE manager_id = current_user_id()
        )
    );
```

---

### SSL/TLS
**Secure Sockets Layer / Transport Layer Security**
(Couche de Sockets Sécurisés / Sécurité de la Couche de Transport)

**Définition** : Protocoles de chiffrement des communications réseau.

**Usage PostgreSQL** :
- Chiffrer les connexions entre client et serveur
- Authentification par certificat

**Configuration** :
```
# postgresql.conf
ssl = on  
ssl_cert_file = 'server.crt'  
ssl_key_file = 'server.key'  
ssl_ca_file = 'root.crt'  
```

**Note** : TLS est la version moderne de SSL (SSL est obsolète mais le nom reste).

---

## Réplication et Clustering

### OLTP
**OnLine Transaction Processing**
(Traitement de Transactions en Ligne)

**Définition** : Charge de travail orientée transactions courtes et fréquentes (lecture/écriture).

**Caractéristiques** :
- Nombreuses connexions simultanées
- Requêtes simples et rapides
- Modifications fréquentes
- Besoin de faible latence

**Exemples** :
- Site e-commerce
- Application bancaire
- CRM

**Optimisation** : Index B-Tree, connexions poolées, MVCC.

---

### OLAP
**OnLine Analytical Processing**
(Traitement Analytique en Ligne)

**Définition** : Charge de travail orientée analyse et reporting (lecture intensive).

**Caractéristiques** :
- Peu de connexions
- Requêtes complexes et longues
- Agrégations massives
- Peu de modifications

**Exemples** :
- Data warehouse
- Tableaux de bord analytiques
- Rapports financiers

**Optimisation** : Vues matérialisées, index BRIN, partitionnement, columnar storage.

---

### WAL-G
**Write-Ahead Log - Go** (implémentation)

**Définition** : Outil de sauvegarde et archivage WAL pour PostgreSQL, écrit en Go.

**Fonctionnalités** :
- Sauvegardes continues et incrémentielles
- Compression et chiffrement
- Support cloud (S3, GCS, Azure)
- PITR facilité

**Alternative** : pgBackRest (autre outil populaire)

---

### SCRAM
**Salted Challenge Response Authentication Mechanism**
(Mécanisme d'Authentification par Défi-Réponse Salé)

**Définition** : Méthode d'authentification sécurisée moderne pour PostgreSQL.

**Variante** : **SCRAM-SHA-256** (recommandé)

**Avantages sur MD5** (obsolète) :
- Plus sécurisé cryptographiquement
- Résistant aux attaques par rejeu
- Salt unique par utilisateur

**Configuration** :
```
# pg_hba.conf
host    all    all    0.0.0.0/0    scram-sha-256
```

**Migration** :
```sql
-- Définir la méthode par défaut
ALTER SYSTEM SET password_encryption = 'scram-sha-256';

-- Réinitialiser les mots de passe
ALTER USER mon_user PASSWORD 'nouveau_mot_de_passe';
```

---

## Administration et Outils

### DBA
**DataBase Administrator**
(Administrateur de Base de Données)

**Définition** : Rôle professionnel responsable de la gestion, maintenance et optimisation des bases de données.

**Responsabilités** :
- Installation et configuration
- Monitoring et tuning des performances
- Sauvegardes et récupération
- Sécurité et gestions des accès
- Haute disponibilité
- Planification de capacité

---

### CLI
**Command-Line Interface**
(Interface en Ligne de Commande)

**Définition** : Interface texte pour interagir avec un système.

**PostgreSQL CLI** : `psql`

**Exemples de commandes** :
```bash
# Se connecter
psql -h localhost -U postgres -d ma_base

# Exécuter une requête
psql -c "SELECT version();"

# Exécuter un script
psql -f mon_script.sql
```

---

### GUI
**Graphical User Interface**
(Interface Graphique Utilisateur)

**Définition** : Interface visuelle pour interagir avec PostgreSQL.

**Outils populaires** :
- **pgAdmin** : L'officiel, complet mais parfois lourd  
- **DBeaver** : Multi-SGBD, moderne, gratuit  
- **DataGrip** : JetBrains (payant), très puissant  
- **TablePlus** : Mac/Windows, élégant et rapide  
- **Postico** : Mac seulement, simple et efficace

---

### ETL
**Extract, Transform, Load**
(Extraire, Transformer, Charger)

**Définition** : Processus de migration et transformation de données.

**Étapes** :
1. **Extract** : Extraire des données sources  
2. **Transform** : Nettoyer, transformer, enrichir  
3. **Load** : Charger dans la base cible

**Outils avec PostgreSQL** :
- Apache Airflow
- Talend
- Pentaho
- Scripts Python (pandas + psycopg3)

---

### CDC
**Change Data Capture**
(Capture de Changements de Données)

**Définition** : Technique pour identifier et capturer les changements dans la base en temps réel.

**Usage PostgreSQL** :
- Logical Replication
- Logical Decoding
- Extensions : Debezium, wal2json

**Cas d'usage** :
- Synchronisation vers un data warehouse
- Event sourcing
- Audit trail
- Cache invalidation

---

### CRUD
**Create, Read, Update, Delete**
(Créer, Lire, Mettre à jour, Supprimer)

**Définition** : Les quatre opérations de base sur les données.

**Correspondance SQL** :
- **C**reate → `INSERT`  
- **R**ead → `SELECT`  
- **U**pdate → `UPDATE`  
- **D**elete → `DELETE`

**Exemple** :
```sql
-- Create
INSERT INTO produits (nom, prix) VALUES ('Laptop', 999);

-- Read
SELECT * FROM produits WHERE id = 1;

-- Update
UPDATE produits SET prix = 899 WHERE id = 1;

-- Delete
DELETE FROM produits WHERE id = 1;
```

---

### ORM
**Object-Relational Mapping**
(Mappage Objet-Relationnel)

**Définition** : Technique/bibliothèque qui convertit entre objets du code et tables de base de données.

**ORMs populaires** :
- **Python** : SQLAlchemy, Django ORM, Peewee  
- **Java** : Hibernate, JPA  
- **JavaScript/Node** : Sequelize, TypeORM, Prisma  
- **Ruby** : ActiveRecord  
- **PHP** : Eloquent (Laravel), Doctrine  
- **.NET** : Entity Framework

**Avantages** :
- Abstraction de la base de données
- Moins de SQL manuel
- Protection contre l'injection SQL

**Inconvénients** :
- Peut générer du SQL non-optimal
- Perte de contrôle fin
- Courbe d'apprentissage

---

## Acronymes de Performance

### QPS
**Queries Per Second**
(Requêtes par Seconde)

**Définition** : Métrique mesurant le nombre de requêtes exécutées par seconde.

**Benchmark** :
- Petit serveur : 100-1000 QPS
- Serveur moyen : 1000-10000 QPS
- Gros serveur : 10000+ QPS

**Facteurs** :
- Complexité des requêtes
- Utilisation d'index
- Configuration serveur
- Connection pooling

---

### TPS
**Transactions Per Second**
(Transactions par Seconde)

**Définition** : Nombre de transactions validées (COMMIT) par seconde.

**Différence avec QPS** : Une transaction peut contenir plusieurs requêtes.

**Outil de benchmark** : pgbench
```bash
pgbench -c 10 -j 2 -t 1000 ma_base
```

---

### IOPS
**Input/Output Operations Per Second**
(Opérations d'Entrée/Sortie par Seconde)

**Définition** : Métrique disque mesurant le nombre d'opérations de lecture/écriture par seconde.

**Importance PostgreSQL** :
- Tables non-cached nécessitent des lectures disque
- WAL nécessite des écritures disque
- Checkpoints provoquent des écritures massives

**Typique** :
- HDD classique : 100-200 IOPS
- SSD SATA : 10 000-100 000 IOPS
- NVMe SSD : 100 000-1 000 000+ IOPS

---

### RTO
**Recovery Time Objective**
(Objectif de Temps de Récupération)

**Définition** : Durée maximale acceptable de panne (downtime).

**Exemples** :
- Site vitrine : RTO = 4 heures
- E-commerce : RTO = 15 minutes
- Banque : RTO = 30 secondes

**Solutions PostgreSQL** :
- Réplication synchrone
- Failover automatique (Patroni)
- Hot standby

---

### RPO
**Recovery Point Objective**
(Objectif de Point de Récupération)

**Définition** : Perte de données maximale acceptable (en temps).

**Exemples** :
- Blog : RPO = 24 heures
- E-commerce : RPO = 5 minutes
- Banque : RPO = 0 (aucune perte acceptable)

**Solutions PostgreSQL** :
- PITR (RPO = quelques secondes à minutes)
- Réplication synchrone (RPO = 0)
- WAL archiving fréquent

---

## Formats et Standards

### JSON
**JavaScript Object Notation**
(Notation d'Objet JavaScript)

**Définition** : Format de données texte, léger et lisible.

**Usage PostgreSQL** :
- Types `json` et `jsonb`
- Stockage de données semi-structurées
- Communication API REST

**Exemple** :
```json
{
  "nom": "Alice",
  "age": 30,
  "competences": ["PostgreSQL", "Python", "Docker"]
}
```

---

### CSV
**Comma-Separated Values**
(Valeurs Séparées par des Virgules)

**Définition** : Format de fichier texte simple pour données tabulaires.

**Usage PostgreSQL** :
```sql
-- Export
COPY (SELECT * FROM clients) TO '/tmp/clients.csv' CSV HEADER;

-- Import
COPY clients FROM '/tmp/clients.csv' CSV HEADER;
```

**Alternative** : TSV (Tab-Separated Values)

---

### XML
**eXtensible Markup Language**
(Langage de Balisage Extensible)

**Définition** : Format de données structuré, plus verbeux que JSON.

**Support PostgreSQL** :
- Type de données `xml`
- Fonctions XPath pour requêtes
- Génération et parsing XML

**Usage** : Moins populaire que JSON aujourd'hui, mais encore utilisé (legacy, SOAP, certaines industries).

---

### UTF-8
**Unicode Transformation Format - 8 bits**
(Format de Transformation Unicode - 8 bits)

**Définition** : Encodage de caractères universel, supportant toutes les langues.

**PostgreSQL** : Encodage par défaut et recommandé.

**Avantages** :
- Support de tous les caractères (émojis inclus 🎉)
- Compatible ASCII
- Standard web

**Définir à la création** :
```sql
CREATE DATABASE ma_base
    ENCODING 'UTF8'
    LC_COLLATE = 'fr_FR.UTF-8'
    LC_CTYPE = 'fr_FR.UTF-8';
```

---

## Acronymes de Sécurité

### RBAC
**Role-Based Access Control**
(Contrôle d'Accès Basé sur les Rôles)

**Définition** : Modèle de sécurité où les permissions sont attribuées à des rôles, et les utilisateurs héritent des rôles.

**PostgreSQL** :
```sql
-- Créer des rôles
CREATE ROLE lecteur;  
CREATE ROLE editeur;  

-- Attribuer permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO lecteur;  
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO editeur;  

-- Créer un utilisateur avec un rôle
CREATE USER alice WITH PASSWORD 'secret';  
GRANT lecteur TO alice;  

-- User Bob a les deux rôles
CREATE USER bob WITH PASSWORD 'secret';  
GRANT lecteur, editeur TO bob;  
```

---

### MFA
**Multi-Factor Authentication**
(Authentification Multi-Facteurs)

**Définition** : Sécurité renforcée nécessitant plusieurs preuves d'identité.

**PostgreSQL** :
- Pas de support MFA natif dans PostgreSQL lui-même
- Implémenté au niveau application
- Ou via proxy (pgBouncer avec auth externe)
- Nouveauté PG 18 : Support OAuth 2.0 (peut inclure MFA)

---

### GDPR / RGPD
**General Data Protection Regulation / Règlement Général sur la Protection des Données**

**Définition** : Réglementation européenne sur la protection des données personnelles.

**Implications PostgreSQL** :
- Chiffrement des données sensibles
- Pseudonymisation/anonymisation
- Droit à l'oubli (suppression complète)
- Audit logs (qui a accédé à quelles données)
- RLS pour isolation des données

---

## Concepts Cloud

### RDS
**Relational Database Service**
(Service de Base de Données Relationnelle)

**Définition** : Service managé AWS pour PostgreSQL (et autres SGBD).

**Avantages** :
- Pas de gestion d'infrastructure
- Sauvegardes automatiques
- Haute disponibilité intégrée
- Scaling facilité

**Équivalents** :
- **Google Cloud** : Cloud SQL  
- **Azure** : Azure Database for PostgreSQL  
- **DigitalOcean** : Managed Databases

---

### IaaS / PaaS / SaaS
**Infrastructure/Platform/Software as a Service**
(Infrastructure/Plateforme/Logiciel en tant que Service)

**Niveaux d'abstraction** :

- **IaaS** : Vous gérez PostgreSQL sur une VM (EC2, Google Compute)  
- **PaaS** : Service managé gère PostgreSQL (RDS, Cloud SQL)  
- **SaaS** : Application complète utilisant PostgreSQL en backend (invisible pour vous)

---

## Conclusion

Ce guide couvre les acronymes les plus courants dans l'univers PostgreSQL. La maîtrise de ce vocabulaire facilite grandement la lecture de documentation, la communication avec d'autres développeurs, et la compréhension des concepts avancés.

### Points clés à retenir

**Fondamentaux** :
- SQL, ACID, RDBMS sont les bases
- DDL, DML, DQL, DCL : les catégories de commandes

**Architecture** :
- MVCC, WAL, TOAST : les mécanismes internes essentiels
- PK, FK, UK : les contraintes garantissant l'intégrité

**Performance** :
- Index : B-Tree, GIN, GiST, BRIN selon les besoins
- CTE pour la lisibilité et la réutilisation
- EXPLAIN pour comprendre les performances

**Administration** :
- VACUUM et ANALYZE pour la maintenance
- HA, RTO, RPO pour la disponibilité
- RLS, RBAC, SSL/TLS pour la sécurité

**Écosystème** :
- FDW, FTS, JSONB : fonctionnalités avancées
- ORM, ETL, CDC : intégrations applicatives
- Cloud : RDS, IaaS, PaaS

### Pour aller plus loin

Consultez les autres sections de la formation :
- Glossaire des termes techniques (concepts détaillés)
- Commandes psql essentielles
- Requêtes SQL de référence
- Configuration par cas d'usage

---

**Astuce** : Créez vos propres fiches mnémotechniques pour les acronymes que vous utilisez le plus souvent !

---


⏭️ [Commandes psql Essentielles](/annexes/commandes-psql/README.md)
