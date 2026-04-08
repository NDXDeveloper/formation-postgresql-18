🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Glossaire : Termes PostgreSQL Essentiels

## Introduction

Ce glossaire présente les concepts et termes fondamentaux de PostgreSQL que tout développeur ou administrateur de bases de données doit connaître. Les explications sont volontairement simplifiées pour être accessibles aux débutants, tout en restant précises techniquement.

---

## Concepts Fondamentaux

### ACID

**ACID** est un acronyme qui désigne les quatre propriétés garantissant la fiabilité des transactions dans une base de données :

- **A**tomicité : Une transaction est "tout ou rien". Si une partie échoue, tout est annulé. C'est comme un virement bancaire : soit l'argent est débité ET crédité, soit rien ne se passe.

- **C**ohérence : Les données restent toujours valides selon les règles définies (contraintes). Par exemple, un solde bancaire ne peut jamais être négatif si cette règle est en place.

- **I**solation : Plusieurs transactions simultanées n'interfèrent pas entre elles. Chaque transaction a l'impression d'être seule, même si des milliers d'autres s'exécutent en même temps.

- **D**urabilité : Une fois validée, une transaction est définitivement enregistrée, même en cas de panne électrique ou de crash serveur.

**Exemple concret** : Quand vous effectuez un achat en ligne, ACID garantit que votre paiement et la réservation du produit se font ensemble, que personne ne voit un état incohérent de votre commande, et que votre achat ne sera jamais perdu.

---

### MVCC (Multiversion Concurrency Control)

**MVCC** est le mécanisme qui permet à PostgreSQL de gérer des milliers de connexions simultanées sans bloquer les utilisateurs.

**Le principe** : Au lieu de "verrouiller" une ligne quand quelqu'un la modifie, PostgreSQL crée une nouvelle version de cette ligne. Les autres utilisateurs continuent de voir l'ancienne version jusqu'à ce que la modification soit validée.

**Analogie** : C'est comme un document Google Docs avec historique des versions. Plusieurs personnes peuvent travailler simultanément, et chacun peut voir l'état du document à un instant donné sans que les modifications en cours des autres ne gênent.

**Avantages** :
- Les lectures ne bloquent jamais les écritures
- Les écritures ne bloquent jamais les lectures
- Excellentes performances en environnement multi-utilisateurs

**Contrepartie** : Les anciennes versions doivent être nettoyées régulièrement (c'est le rôle de VACUUM).

---

### Transaction

Une **transaction** est un ensemble d'opérations SQL qui doivent toutes réussir ou toutes échouer ensemble.

**Syntaxe de base** :
```sql
BEGIN;
    UPDATE compte SET solde = solde - 100 WHERE id = 1;
    UPDATE compte SET solde = solde + 100 WHERE id = 2;
COMMIT;
```

**États d'une transaction** :
- `BEGIN` : Démarre la transaction  
- `COMMIT` : Valide définitivement les modifications  
- `ROLLBACK` : Annule toutes les modifications depuis le BEGIN  
- `SAVEPOINT` : Crée un point de sauvegarde intermédiaire

**En cas d'erreur**, toute la transaction est automatiquement annulée (rollback automatique).

---

## Architecture et Stockage

### WAL (Write-Ahead Log)

Le **WAL** est un journal qui enregistre TOUTES les modifications avant qu'elles ne soient appliquées aux fichiers de données réels.

**Principe** : "Écrire d'abord dans le journal, appliquer ensuite"

**Pourquoi c'est important** :
1. **Durabilité** : En cas de crash, PostgreSQL peut rejouer le WAL pour récupérer les données  
2. **Performance** : Écrire séquentiellement dans un journal est plus rapide qu'écrire aléatoirement dans les tables  
3. **Réplication** : Le WAL peut être envoyé à d'autres serveurs pour créer des copies en temps réel

**Analogie** : Le WAL est comme un brouillon où vous notez tout ce que vous faites. Si vous renversez votre café sur votre travail final, vous pouvez tout reconstituer grâce à votre brouillon.

**Fichiers WAL** : Stockés dans le répertoire `pg_wal/`, généralement de 16 Mo chacun.

---

### TOAST (The Oversized-Attribute Storage Technique)

**TOAST** est le mécanisme intelligent de PostgreSQL pour gérer les données volumineuses.

**Le problème** : PostgreSQL stocke normalement les lignes sur des pages de 8 Ko. Mais que faire si une colonne contient un texte de 100 Ko ou une image ?

**La solution TOAST** :
- Les petites valeurs (< ~2 Ko) restent dans la table principale
- Les grandes valeurs sont automatiquement compressées et/ou déplacées dans une table TOAST séparée
- Tout cela est transparent pour l'utilisateur

**Stratégies TOAST** :
- `PLAIN` : Pas de compression ni de stockage externe  
- `EXTENDED` : Compression + stockage externe (défaut)  
- `EXTERNAL` : Stockage externe sans compression (pour données déjà compressées)  
- `MAIN` : Compression prioritaire, mais évite le stockage externe

**Types concernés** : TEXT, VARCHAR, BYTEA, JSON, JSONB, ARRAY

---

### Tablespace

Un **tablespace** est un emplacement physique sur le disque où PostgreSQL stocke les données.

**Utilité** :
- Répartir les données sur plusieurs disques pour améliorer les performances
- Placer les tables peu utilisées sur des disques lents et économiques
- Placer les index très sollicités sur des SSD rapides

**Exemple** :
```sql
-- Créer un tablespace sur un SSD rapide
CREATE TABLESPACE fast_storage LOCATION '/mnt/ssd/postgresql';

-- Créer une table sur ce tablespace
CREATE TABLE commandes (...) TABLESPACE fast_storage;
```

---

### Heap

Le **heap** est la structure de stockage par défaut des tables dans PostgreSQL.

**Caractéristiques** :
- Les lignes sont stockées sans ordre particulier
- Les nouvelles lignes sont ajoutées là où il y a de la place
- Aucune organisation physique basée sur une clé

**Note** : C'est différent d'un "heap" en programmation. Ici, ça signifie simplement "tas" (pile désordonnée).

**Conséquence** : Sans index, PostgreSQL doit scanner toute la table pour trouver une ligne (scan séquentiel).

---

## Processus et Architecture

### Postmaster

Le **Postmaster** est le processus principal (maître) de PostgreSQL.

**Rôles** :
- Démarré en premier au lancement de PostgreSQL
- Écoute les connexions entrantes sur le port 5432 (par défaut)
- Crée un processus backend pour chaque nouvelle connexion client
- Gère les processus d'arrière-plan (autovacuum, checkpointer, etc.)

**Commande pour voir le postmaster** :
```bash
ps aux | grep postgres
```

---

### Backend Process

Un **backend** est un processus serveur dédié à une connexion client.

**Fonctionnement** :
- Un backend = Une connexion = Un processus système
- Chaque backend a sa propre mémoire (work_mem, temp_buffers)
- Exécute les requêtes SQL du client
- Meurt quand le client se déconnecte

**Limite** : Le paramètre `max_connections` définit le nombre maximum de backends simultanés (défaut : 100).

---

### Shared Buffers

Les **Shared Buffers** constituent le cache mémoire partagé par tous les processus PostgreSQL.

**Fonctionnement** :
- Cache en RAM des pages de données les plus utilisées
- Partagé entre tous les backends
- Évite des lectures disques coûteuses

**Configuration typique** :
- 25% de la RAM disponible sur un serveur dédié
- Exemple : `shared_buffers = 4GB` pour un serveur avec 16 GB de RAM

**Métrique importante** : Le "cache hit ratio" (taux de succès du cache) doit être > 95%.

---

### Autovacuum

**Autovacuum** est un processus d'arrière-plan qui nettoie automatiquement les tables.

**Pourquoi c'est nécessaire** :
- MVCC crée des versions obsolètes de lignes (dead tuples)
- Ces vieilles versions prennent de l'espace disque
- Elles ralentissent les requêtes

**Actions d'Autovacuum** :
1. **VACUUM** : Récupère l'espace des lignes mortes  
2. **ANALYZE** : Met à jour les statistiques pour l'optimiseur de requêtes  
3. Prévient le "XID wraparound" (problème critique)

**Configuration** : Activé par défaut. Ne JAMAIS le désactiver en production !

---

## Optimisation et Index

### B-Tree (Balanced Tree)

Le **B-Tree** est le type d'index par défaut et le plus utilisé dans PostgreSQL.

**Structure** : Arbre équilibré permettant des recherches très rapides (logarithmiques).

**Cas d'usage** :
- Égalité : `WHERE id = 123`
- Comparaisons : `WHERE age > 25`
- Tri : `ORDER BY nom`
- Plages : `WHERE date BETWEEN '2024-01-01' AND '2024-12-31'`

**Performance** : Pour 1 million de lignes, un B-Tree trouve une ligne en ~20 lectures au lieu de 1 million (scan séquentiel).

---

### GIN (Generalized Inverted Index)

Le **GIN** est un index spécialisé pour les recherches dans des structures complexes.

**Cas d'usage principaux** :
- **Full-Text Search** : Recherche dans du texte  
- **JSONB** : Recherche dans des documents JSON  
- **Arrays** : Recherche dans des tableaux

**Exemple** :
```sql
-- Index pour recherche full-text
CREATE INDEX idx_articles_contenu ON articles USING GIN(to_tsvector('french', contenu));

-- Index pour JSONB
CREATE INDEX idx_data_gin ON table USING GIN(data);
```

**Caractéristique** : Plus lourd qu'un B-Tree, mais indispensable pour ces cas d'usage.

---

### GiST (Generalized Search Tree)

Le **GiST** est un framework d'index flexible pour des types de données non-standards.

**Cas d'usage** :
- **Données géométriques** (PostGIS) : Points, polygones, lignes  
- **Full-Text Search** (alternative à GIN)  
- **Plages** (Range types) : `int4range`, `tstzrange`  
- **Hiérarchies** (ltree) : Structures arborescentes

**Exemple spatial** :
```sql
CREATE INDEX idx_restaurants_location ON restaurants USING GiST(location);
-- Trouver tous les restaurants dans un rayon de 1 km
SELECT * FROM restaurants WHERE ST_DWithin(location, mon_point, 1000);
```

---

### BRIN (Block Range Index)

Le **BRIN** est un index ultra-léger pour les données MASSIVES et ordonnées naturellement.

**Principe** : Au lieu d'indexer chaque ligne, BRIN stocke uniquement les valeurs min/max par bloc de données.

**Idéal pour** :
- Tables de plusieurs To (logs, IoT, time-series)
- Données insérées de manière chronologique
- Quand l'ordre physique correspond à l'ordre logique

**Avantages** :
- Index minuscule (quelques Mo pour des To de données)
- Maintenance quasi-nulle
- Très rapide à créer

**Exemple** :
```sql
-- Table de logs avec 1 milliard de lignes ordonnées par timestamp
CREATE INDEX idx_logs_date ON logs USING BRIN(timestamp);
```

---

## Concurrence et Verrouillage

### Lock (Verrou)

Un **lock** (verrou) empêche des opérations conflictuelles de s'exécuter simultanément.

**Types principaux** :
- **Row-level locks** : Verrouillent des lignes spécifiques  
  - `FOR UPDATE` : Verrouillage exclusif  
  - `FOR SHARE` : Verrouillage partagé

- **Table-level locks** : Verrouillent toute une table  
  - `ACCESS SHARE` : Lecture (peu restrictif)  
  - `EXCLUSIVE` : Modification de structure (très restrictif)

**Voir les verrous actifs** :
```sql
SELECT * FROM pg_locks;
```

---

### Deadlock

Un **deadlock** (interblocage) survient quand deux transactions s'attendent mutuellement.

**Exemple classique** :
```
Transaction 1 : Verrouille ligne A → Attend ligne B  
Transaction 2 : Verrouille ligne B → Attend ligne A  
→ Blocage mutuel !
```

**Gestion par PostgreSQL** :
- Détection automatique après 1 seconde (par défaut)
- Une des transactions est automatiquement annulée (ROLLBACK)
- L'application reçoit une erreur qu'elle doit gérer (retry)

**Prévention** :
- Toujours accéder aux ressources dans le même ordre
- Garder les transactions courtes
- Utiliser des niveaux d'isolation appropriés

---

### Isolation Level (Niveau d'Isolation)

Le **niveau d'isolation** définit le degré de visibilité des modifications entre transactions concurrentes.

**Les 4 niveaux SQL standard** (du moins au plus strict) :

1. **Read Uncommitted** (non supporté par PostgreSQL)
   - Lit les modifications non validées des autres
   - Non fiable

2. **Read Committed** (défaut PostgreSQL)
   - Lit uniquement les modifications validées
   - Chaque requête voit les dernières données commitées
   - Convient à 95% des cas

3. **Repeatable Read**
   - Voit un snapshot fixe au début de la transaction
   - Protège contre les "non-repeatable reads"
   - Idéal pour les rapports consistants

4. **Serializable**
   - Le plus strict : comme si les transactions s'exécutaient en série
   - Détecte et empêche les anomalies complexes
   - Plus lent, mais garantit la cohérence maximale

**Définir le niveau** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

---

## Schéma et Organisation

### Schema

Un **schema** est un namespace (espace de noms) qui regroupe des objets de base de données.

**Analogie** : Un schema est comme un dossier dans un système de fichiers.

**Structure hiérarchique** :
```
Cluster PostgreSQL
└── Database (ex: ma_boutique)
    ├── Schema public (défaut)
    │   ├── Table clients
    │   └── Table commandes
    └── Schema comptabilite
        ├── Table factures
        └── Table paiements
```

**Avantages** :
- Organisation logique (RH, ventes, production...)
- Séparation des environnements (dev, test, prod dans la même DB)
- Gestion fine des permissions

**Exemple** :
```sql
CREATE SCHEMA ventes;  
CREATE TABLE ventes.produits (...);  
```

---

### Search Path

Le **search_path** définit l'ordre de recherche des schemas quand un objet n'est pas qualifié.

**Défaut** : `"$user", public`

**Exemple** :
```sql
-- Sans search_path, il faut qualifier complètement
SELECT * FROM ventes.produits;

-- Avec search_path = ventes, public
SET search_path TO ventes, public;  
SELECT * FROM produits;  -- Cherche d'abord dans "ventes", puis "public"  
```

---

### Sequence

Une **sequence** est un générateur de nombres auto-incrémentés.

**Usage principal** : Générer des identifiants uniques (clés primaires).

**Types** :
- `SERIAL` : Raccourci pour créer une sequence + colonne INTEGER (legacy)  
- `IDENTITY` : Standard SQL moderne (recommandé depuis PG 10)

**Exemple** :
```sql
-- Méthode moderne
CREATE TABLE produits (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom TEXT
);

-- Méthode legacy
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom TEXT
);
```

---

## Maintenance et Administration

### VACUUM

**VACUUM** est le processus de nettoyage qui récupère l'espace disque occupé par les lignes obsolètes.

**Raisons** :
- MVCC crée des "dead tuples" (lignes mortes) à chaque UPDATE/DELETE
- Ces lignes occupent de l'espace mais ne sont plus utilisées
- Sans VACUUM, la base enfle indéfiniment

**Types** :
- `VACUUM` : Nettoyage standard (ne rend pas l'espace à l'OS)  
- `VACUUM FULL` : Nettoyage complet avec réorganisation (bloque la table, éviter en production)  
- `autovacuum` : Automatique, activé par défaut

**Commande** :
```sql
VACUUM ma_table;  
VACUUM ANALYZE ma_table;  -- VACUUM + mise à jour des statistiques  
```

---

### ANALYZE

**ANALYZE** collecte des statistiques sur le contenu des tables pour aider l'optimiseur de requêtes.

**Statistiques collectées** :
- Distribution des valeurs
- Valeurs les plus fréquentes
- Corrélation entre colonnes
- Taille des tables et index

**Résultat** : Le planificateur peut choisir le meilleur plan d'exécution (index vs scan séquentiel, ordre de jointures, etc.).

**Commande** :
```sql
ANALYZE ma_table;  
ANALYZE;  -- Toutes les tables  
```

**Automatisé** : `autovacuum` exécute aussi ANALYZE automatiquement.

---

### Checkpoint

Un **checkpoint** est une opération qui écrit toutes les modifications en mémoire (dirty pages) vers le disque.

**Processus** :
1. Les modifications normales vont d'abord dans le WAL (rapide)  
2. Elles restent en mémoire (shared buffers) jusqu'au checkpoint  
3. Le checkpoint écrit tout sur disque de manière ordonnée  
4. Le WAL peut être recyclé après un checkpoint

**Déclenchement** :
- Automatique toutes les X minutes (`checkpoint_timeout`)
- Automatique si le WAL atteint une certaine taille (`max_wal_size`)
- Manuel : `CHECKPOINT;`

**Impact** : Les checkpoints peuvent causer des pics d'I/O. Configuration importante pour la performance.

---

### XID (Transaction ID)

Le **XID** est un identifiant unique de 32 bits attribué à chaque transaction.

**Problème du XID wraparound** :
- Sur 32 bits, il n'y a que 4 milliards de XID possibles
- Après 2 milliards de transactions, les XID "bouclent" (wraparound)
- Sans précaution, les anciennes données deviendraient "invisibles"

**Solution** : VACUUM gèle (freeze) les anciennes transactions pour éviter le wraparound.

**Monitoring** :
```sql
SELECT age(datfrozenxid) FROM pg_database WHERE datname = 'ma_base';
-- Si > 200 millions, VACUUM est nécessaire
```

**IMPORTANT** : Laisser autovacuum actif pour éviter ce problème critique !

---

## Types de Données Spéciaux

### JSONB

**JSONB** est le type JSON binaire de PostgreSQL, optimisé pour les requêtes.

**Différence JSON vs JSONB** :
- `JSON` : Stocke le texte tel quel (plus lent)  
- `JSONB` : Stocke en format binaire décomposé (plus rapide, plus de fonctionnalités)

**Recommandation** : Toujours utiliser JSONB sauf cas très spécifique.

**Exemple** :
```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    data JSONB
);

-- Requêtes sur JSONB
SELECT * FROM produits WHERE data->>'categorie' = 'electronique';  
SELECT * FROM produits WHERE data @> '{"en_stock": true}';  

-- Index pour performance
CREATE INDEX idx_data ON produits USING GIN(data);
```

---

### Array (Tableau)

PostgreSQL supporte nativement les **tableaux** pour tous les types de données.

**Déclaration** :
```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    tags TEXT[],                    -- Tableau de textes
    scores INTEGER[]                -- Tableau d'entiers
);
```

**Manipulation** :
```sql
-- Insertion
INSERT INTO articles (tags) VALUES (ARRAY['postgresql', 'database', 'sql']);  
INSERT INTO articles (tags) VALUES ('{"tech", "tutorial"}');  

-- Requêtes
SELECT * FROM articles WHERE 'postgresql' = ANY(tags);  
SELECT * FROM articles WHERE tags @> ARRAY['database'];  

-- Index GIN pour recherche rapide
CREATE INDEX idx_tags ON articles USING GIN(tags);
```

---

### UUID

Un **UUID** (Universally Unique Identifier) est un identifiant unique de 128 bits, garanti unique globalement.

**Format** : `550e8400-e29b-41d4-a716-446655440000`

**Avantages** :
- Pas besoin de coordination entre serveurs (distribué)
- Pas de révélation d'informations (contrairement aux séquences)
- Fusion de bases facilitée

**Inconvénients** :
- Plus gros qu'un INTEGER (16 octets vs 4)
- Aléatoire → fragmentation d'index

**Nouveauté PostgreSQL 18 : UUIDv7**
- Version 7 ordonnée par timestamp
- Résout le problème de fragmentation
- Meilleure performance que UUID v4

**Exemple** :
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    -- ou en PG 18
    id UUID DEFAULT uuidv7() PRIMARY KEY,
    nom TEXT
);
```

---

### ENUM

Un **ENUM** est un type défini par l'utilisateur avec une liste fixe de valeurs.

**Avantages** :
- Validation automatique des données
- Économie d'espace (stocké comme entier)
- Lisibilité du code

**Limitations** :
- Difficile à modifier (ajouter/supprimer des valeurs)
- Ordre défini à la création

**Exemple** :
```sql
CREATE TYPE statut_commande AS ENUM ('en_attente', 'payee', 'expediee', 'livree', 'annulee');

CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    statut statut_commande DEFAULT 'en_attente'
);

-- Valide
INSERT INTO commandes (statut) VALUES ('payee');

-- Erreur : valeur invalide
INSERT INTO commandes (statut) VALUES ('invalide');
```

---

## Réplication et Haute Disponibilité

### Replication Slot

Un **replication slot** garantit que le serveur primaire conserve le WAL nécessaire pour un replica, même si ce dernier est temporairement hors ligne.

**Problème sans slot** :
- Le replica déconnecté pendant 2h
- Le primaire recycle le WAL au bout d'1h
- Le replica ne peut plus rattraper son retard → rupture de réplication

**Solution avec slot** :
- Le WAL est conservé jusqu'à ce que le replica le consomme
- Garantit la continuité de la réplication

**Types** :
- **Physical slot** : Pour la réplication physique (streaming)  
- **Logical slot** : Pour la réplication logique (publications/subscriptions)

**Attention** : Un slot peut faire exploser l'espace disque si le replica ne se reconnecte jamais. Monitoring essentiel !

---

### WAL Archiving

Le **WAL archiving** consiste à copier les fichiers WAL vers un stockage externe pour permettre une récupération point-in-time (PITR).

**Configuration de base** :
```
wal_level = replica  (ou logical)  
archive_mode = on  
archive_command = 'cp %p /backup/wal_archives/%f'  
```

**Cas d'usage** :
- Sauvegarde continue (alternative/complément à pg_dump)
- Point-In-Time Recovery : restaurer à un instant T précis
- Réplication vers un standby distant

---

### Streaming Replication

La **streaming replication** est la méthode standard de réplication physique en temps réel.

**Principe** :
- Le serveur standby se connecte au primary
- Il reçoit en continu le flux WAL
- Il applique ces modifications en temps quasi-réel

**Types** :
- **Asynchrone** : Le primary n'attend pas la confirmation du standby (rapide mais risque de perte)  
- **Synchrone** : Le primary attend la confirmation (lent mais sûr)

**Configuration minimale** :
```
# Sur le primary
wal_level = replica  
max_wal_senders = 3  

# Sur le standby
primary_conninfo = 'host=primary port=5432 user=replication'
```

---

### Logical Replication

La **réplication logique** réplique les modifications au niveau logique (lignes changées) plutôt que physique (blocs disque).

**Avantages sur la réplication physique** :
- Réplication sélective (certaines tables seulement)
- Réplication vers une version différente de PostgreSQL
- Réplication bidirectionnelle possible
- Le standby peut être accessible en écriture

**Concepts** :
- **Publication** : Sur le serveur source, définit ce qui est répliqué  
- **Subscription** : Sur le serveur destination, consomme les modifications

**Exemple** :
```sql
-- Sur le primary
CREATE PUBLICATION ma_pub FOR TABLE clients, commandes;

-- Sur le standby
CREATE SUBSCRIPTION ma_sub
    CONNECTION 'host=primary dbname=ma_base'
    PUBLICATION ma_pub;
```

---

## Extensions

### Extension

Une **extension** est un module qui ajoute des fonctionnalités à PostgreSQL.

**Installation** :
```sql
CREATE EXTENSION nom_extension;
```

**Extensions populaires** :
- `pg_stat_statements` : Statistiques de requêtes  
- `pgcrypto` : Fonctions cryptographiques  
- `uuid-ossp` : Génération d'UUID  
- `hstore` : Paires clé-valeur  
- `PostGIS` : Données géospatiales  
- `pg_trgm` : Recherche de similarité de texte

**Lister les extensions disponibles** :
```sql
SELECT * FROM pg_available_extensions;
```

---

### Foreign Data Wrapper (FDW)

Un **FDW** permet d'accéder à des données externes comme si elles étaient des tables PostgreSQL locales.

**Cas d'usage** :
- Requêter une autre base PostgreSQL distante
- Intégrer MySQL, Oracle, MongoDB dans des jointures
- Lire des fichiers CSV comme des tables
- Interroger des APIs REST

**Exemple avec postgres_fdw** :
```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER serveur_distant
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'autre-serveur.com', dbname 'autre_base', port '5432');

CREATE USER MAPPING FOR postgres
    SERVER serveur_distant
    OPTIONS (user 'remote_user', password 'secret');

CREATE FOREIGN TABLE clients_distants (
    id INTEGER,
    nom TEXT
) SERVER serveur_distant OPTIONS (table_name 'clients');

-- Utilisation normale
SELECT * FROM clients_distants WHERE id = 123;
```

---

## Termes Divers

### DDL (Data Definition Language)

**DDL** regroupe les commandes qui définissent la structure de la base de données.

**Commandes principales** :
- `CREATE` : Créer (table, index, schema...)  
- `ALTER` : Modifier (ajouter/supprimer colonnes, changer types...)  
- `DROP` : Supprimer  
- `TRUNCATE` : Vider une table (DDL car plus violent que DELETE)

**Exemple** :
```sql
CREATE TABLE produits (id SERIAL PRIMARY KEY, nom TEXT);  
ALTER TABLE produits ADD COLUMN prix NUMERIC(10,2);  
DROP TABLE ancienne_table;  
```

---

### DML (Data Manipulation Language)

**DML** regroupe les commandes qui manipulent les données.

**Commandes** :
- `SELECT` : Lire  
- `INSERT` : Insérer  
- `UPDATE` : Modifier  
- `DELETE` : Supprimer

**Exemple** :
```sql
INSERT INTO produits (nom, prix) VALUES ('Ordinateur', 999.99);  
UPDATE produits SET prix = 899.99 WHERE nom = 'Ordinateur';  
DELETE FROM produits WHERE prix > 10000;  
```

---

### DQL (Data Query Language)

**DQL** désigne parfois spécifiquement `SELECT` pour le distinguer des autres commandes DML.

---

### DCL (Data Control Language)

**DCL** gère les permissions et la sécurité.

**Commandes** :
- `GRANT` : Donner des permissions  
- `REVOKE` : Retirer des permissions

**Exemple** :
```sql
GRANT SELECT ON TABLE clients TO user_reporting;  
REVOKE INSERT ON TABLE clients FROM user_readonly;  
```

---

### Constraint (Contrainte)

Une **constraint** est une règle qui garantit l'intégrité des données.

**Types** :
- **PRIMARY KEY** : Identifiant unique non-null  
- **FOREIGN KEY** : Référence vers une autre table  
- **UNIQUE** : Valeur unique (peut être NULL)  
- **CHECK** : Condition personnalisée  
- **NOT NULL** : Valeur obligatoire

**Exemple** :
```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id),
    montant NUMERIC CHECK (montant > 0),
    email TEXT UNIQUE
);
```

---

### Trigger

Un **trigger** est une fonction qui s'exécute automatiquement lors d'événements sur une table (INSERT, UPDATE, DELETE).

**Cas d'usage** :
- Audit automatique (qui a modifié quoi et quand)
- Validation complexe
- Synchronisation vers d'autres tables
- Calculs automatiques

**Exemple simple** :
```sql
CREATE OR REPLACE FUNCTION maj_timestamp()  
RETURNS TRIGGER AS $$  
BEGIN  
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_maj_timestamp
    BEFORE UPDATE ON produits
    FOR EACH ROW
    EXECUTE FUNCTION maj_timestamp();
```

---

### View (Vue)

Une **view** est une requête SQL sauvegardée et réutilisable comme une table virtuelle.

**Avantages** :
- Simplifie les requêtes complexes
- Masque la complexité aux utilisateurs
- Centralise la logique métier
- Améliore la sécurité (exposer certaines colonnes seulement)

**Exemple** :
```sql
CREATE VIEW ventes_2024 AS  
SELECT  
    p.nom AS produit,
    SUM(c.quantite) AS total_vendu
FROM commandes c  
JOIN produits p ON c.produit_id = p.id  
WHERE c.date >= '2024-01-01'  
GROUP BY p.nom;  

-- Utilisation
SELECT * FROM ventes_2024;
```

---

### Materialized View

Une **materialized view** est une vue dont le résultat est physiquement stocké sur disque.

**Différence avec une vue normale** :
- Vue normale : Exécute la requête à chaque fois
- Vue matérialisée : Stocke le résultat, le lit rapidement

**Contrepartie** : Il faut rafraîchir manuellement les données.

**Exemple** :
```sql
CREATE MATERIALIZED VIEW stats_mensuelles AS  
SELECT  
    DATE_TRUNC('month', date) AS mois,
    SUM(montant) AS total
FROM commandes  
GROUP BY 1;  

-- Créer un index sur la vue matérialisée
CREATE INDEX ON stats_mensuelles(mois);

-- Rafraîchir les données
REFRESH MATERIALIZED VIEW stats_mensuelles;
```

**Cas d'usage** : Rapports coûteux calculés périodiquement (nuit, toutes les heures...).

---

### Connection Pooling

Le **connection pooling** est une technique qui réutilise les connexions à la base de données au lieu d'en créer une nouvelle à chaque requête.

**Problème** :
- Créer une connexion PostgreSQL est coûteux (authentification, initialisation...)
- Dans une application web avec 1000 requêtes/seconde, créer 1000 connexions simultanées = crash

**Solution** :
- Un pool maintient 20 connexions ouvertes
- Les requêtes "empruntent" une connexion disponible
- La connexion retourne au pool après usage

**Outils** :
- **PgBouncer** : Pooler externe ultra-léger (recommandé)  
- **Pgpool-II** : Plus complet (load balancing, cache...)
- Pooling intégré dans l'application (HikariCP en Java, connexion pools en Python...)

---

### Prepared Statement

Un **prepared statement** est une requête SQL compilée et mise en cache par le serveur.

**Avantages** :
- **Performance** : La requête n'est analysée qu'une fois  
- **Sécurité** : Protection automatique contre l'injection SQL

**Exemple en SQL** :
```sql
PREPARE get_produit (INTEGER) AS
    SELECT * FROM produits WHERE id = $1;

EXECUTE get_produit(42);  
EXECUTE get_produit(99);  
```

**En pratique** : Les drivers (psycopg2, node-pg...) utilisent automatiquement des prepared statements.

---

## Conclusion

Ce glossaire couvre les concepts essentiels pour comprendre et utiliser PostgreSQL efficacement. Chaque terme est interconnecté avec les autres, formant l'écosystème riche et puissant de PostgreSQL.

**Points clés à retenir** :
- **ACID** garantit la fiabilité des transactions  
- **MVCC** permet une excellente concurrence sans blocages  
- **WAL** assure la durabilité et permet la réplication  
- **Index** (B-Tree, GIN, GiST, BRIN) optimisent les performances selon les cas d'usage  
- **VACUUM** et **ANALYZE** sont essentiels pour la santé de la base  
- **Réplication** et **HA** assurent la disponibilité en production

---

**Ressources pour approfondir** :
- Documentation officielle PostgreSQL : https://www.postgresql.org/docs/
- Wiki PostgreSQL : https://wiki.postgresql.org/
- Glossaire officiel : https://www.postgresql.org/docs/current/glossary.html

---


⏭️ [Acronymes courants (FK, PK, CTE, FDW, GIN, GiST, etc.)](/annexes/glossaire/02-acronymes-courants.md)
