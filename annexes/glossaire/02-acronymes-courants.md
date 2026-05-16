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

**Actions possibles** (`ON DELETE` et `ON UPDATE`) :
- `NO ACTION` *(défaut)* : vérifie la contrainte **à la fin de la transaction** — laisse une fenêtre pour corriger l'incohérence avant le `COMMIT`.
- `RESTRICT` : vérifie **immédiatement**, refuse la suppression/mise à jour sans attendre la fin de transaction.
- `CASCADE` : applique la même opération sur les lignes référençantes (DELETE → DELETE en cascade, UPDATE → UPDATE de la FK).
- `SET NULL` : met les colonnes référençantes à `NULL` (la colonne FK doit être nullable).
- `SET DEFAULT` : met les colonnes référençantes à leur valeur `DEFAULT`.

> 💡 `NO ACTION` et `RESTRICT` se comportent quasi-identiquement, mais `NO ACTION` permet de fixer la cohérence en cours de transaction (par exemple : `DELETE` du parent suivi d'un `INSERT` réparateur avant le `COMMIT`). En pratique, on choisit `NO ACTION` (laisser le moteur valider à la fin) ou `CASCADE` (propagation explicite) selon le besoin métier.

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

**Définition** : Un numéro unique sur 32 bits attribué à chaque transaction (espace total : ~4 milliards).

**Problème : « wraparound »** : PostgreSQL utilise une comparaison **modulaire** des XIDs (passé/futur). Seule la moitié de l'espace est « visible à un instant T » — soit **~2 milliards d'XIDs**. Au-delà de cet âge, les transactions anciennes deviendraient subitement « futures » : *catastrophe*. PostgreSQL refuse alors d'accepter de nouvelles transactions et bascule en lecture seule pour protéger les données (`autovacuum_freeze_max_age` déclenche un VACUUM forcé bien avant, à 200 M par défaut).

**Solution** : VACUUM **gèle** (freeze) les anciennes transactions (`vacuum_freeze_min_age`, défaut 50 M) : leurs XID sont remplacés par le marqueur spécial `FrozenTransactionId` qui n'a plus à participer à la comparaison modulaire. La table peut alors continuer à vivre indéfiniment.

**Monitoring** : Surveiller l'âge maximal avec `age(datfrozenxid)` (par base) et `age(relfrozenxid)` (par table). Le seuil dur est ~2 milliards ; l'autovacuum doit avoir agi bien avant.

```sql
SELECT datname, age(datfrozenxid),
       ROUND(100.0 * age(datfrozenxid) / 2000000000, 2) AS pct_to_wraparound
FROM pg_database  
ORDER BY age(datfrozenxid) DESC;  
```

---

### TID / CTID
**Tuple ID / *Current Tuple ID***

**Définition** :
- `tid` est le **type de données** qui représente l'adresse physique d'une ligne, sous la forme `(page, item)`.
- `ctid` est la **colonne système** présente dans toutes les tables ordinaires (sauf vues, FDW, etc.), de type `tid`, qui expose cette adresse pour la ligne courante.

**Format** : `(numéro_de_page, offset_dans_la_page)` — ex. `(15,3)` = 4ᵉ entrée (offset 1-indexé) de la 16ᵉ page de 8 ko.

**Exemple** :
```sql
SELECT ctid, id, nom FROM produits WHERE id = 100;
-- ctid  | id  | nom
--(15,3) | 100 | Ordinateur
```

**Cas d'usage légitimes** :
- Diagnostic de fragmentation (corrélation `ctid` ↔ valeurs).
- Suppression rapide de doublons identifiés (`DELETE … WHERE ctid IN (…)`).
- Inspection bas-niveau avec `pageinspect`.

**⚠️ Attention** : un `ctid` n'est **pas stable**. Toute opération qui réécrit la ligne le change : `UPDATE` (sauf HOT update sur la même page), `VACUUM FULL`, `CLUSTER`, `pg_repack`. Ne jamais stocker un `ctid` comme clé applicative.

---

### LSN
**Log Sequence Number**
(Numéro de Séquence du Journal)

**Définition** : Position absolue **en octets** dans le flux WAL, exprimée en hexadécimal sous la forme `XXX/YYYYYYYY` (haut 32 bits / bas 32 bits). Le type SQL associé est `pg_lsn`.

**Usage** :
- Identifier un point de reprise (`pg_basebackup`, PITR).
- Mesurer le **retard de réplication** (résultat en octets) :
  - **Côté primaire** : `pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)` via `pg_stat_replication`.
  - **Côté standby** : `pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn())` (retard d'application local).
- Suivre la progression du WAL : `pg_wal_lsn_diff(a, b)` retourne le nombre d'octets entre deux LSN.

**Exemple** :
```sql
SELECT pg_current_wal_lsn();                        -- ex. 7/3A2C8000

-- Nom du fichier WAL contenant ce LSN (segment de 16 Mo par défaut,
-- ajustable via wal_segment_size à l'initdb depuis PG 11)
SELECT pg_walfile_name(pg_current_wal_lsn());       -- 000000010000000700000003
--                                                    └TLID┘└─high LSN─┘└─seg─┘

-- Volume WAL généré entre deux instants
SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), '7/3A000000'::pg_lsn)
) AS wal_genere;
```

---

### HBA
**Host-Based Authentication**
(Authentification Basée sur l'Hôte)

**Définition** : Mécanisme PostgreSQL qui décide *qui peut se connecter, depuis où, à quelle base, avec quelle méthode*, configuré via le fichier **`pg_hba.conf`**.

**Format d'une ligne** :
```
# TYPE   DATABASE   USER     ADDRESS         METHOD
local    all        all                      peer  
host     app        alice    10.0.0.0/24     scram-sha-256  
host     all        all      0.0.0.0/0       reject  
hostssl  all        all      0.0.0.0/0       scram-sha-256  
```

**Règles** :
- Évalué **dans l'ordre** : la première règle qui *matche* s'applique.
- `local` = socket Unix ; `host` = TCP/IP ; `hostssl` = TCP/IP **avec** SSL ; `hostnossl` = TCP/IP **sans** SSL.
- Méthodes courantes : `trust`, `reject`, `peer`, `md5` (legacy), `scram-sha-256` (recommandé), `cert`, `ldap`, `oauth` 🆕 (PG 18).

**Recharger après modification** :
```sql
SELECT pg_reload_conf();   -- ou pg_ctl reload
```

---

### HOT
***Heap-Only Tuple***

**Définition** : Optimisation PostgreSQL qui permet à un `UPDATE` de **rester dans la même page** sans toucher aux index, à condition qu'aucune colonne indexée ne change et qu'il reste de la place dans la page.

**Bénéfices** :
- Pas de réécriture des entrées d'index → écritures réduites.
- Les anciennes versions sont nettoyées au passage par le mécanisme de *HOT chain pruning*, sans attendre `VACUUM`.

**Levier de configuration** : le **`fillfactor`** d'une table fréquemment mise à jour. Le baisser (ex. `fillfactor = 90`) laisse de l'espace dans chaque page pour favoriser les HOT updates.

```sql
ALTER TABLE compteurs SET (fillfactor = 80);
```

---

### SSI
***Serializable Snapshot Isolation***

**Définition** : Implémentation PostgreSQL du niveau d'isolation `SERIALIZABLE` (depuis PG 9.1). Au lieu de poser des verrous prédicats coûteux, SSI :

1. travaille en *snapshot isolation* (comme `REPEATABLE READ`) ;
2. trace les dépendances de lecture/écriture entre transactions concurrentes ;
3. annule l'une des transactions impliquées dans un cycle de dépendances pouvant produire une anomalie de sérialisation (erreur SQLSTATE `40001`).

**Conséquence pour l'application** : prévoir une logique de *retry* sur l'erreur `serialization_failure` (`40001`).

---

### BGW
***Background Worker***

**Définition** : Processus serveur démarré et supervisé par le postmaster, destiné à des tâches asynchrones internes ou fournies par des extensions.

**Exemples natifs** : autovacuum *launcher*/*workers*, *parallel workers* (exécution parallèle de requêtes), *logical replication workers*, *walwriter*, *checkpointer*, *walsender*/*walreceiver*.

**Exemples d'extensions** : `pg_cron`, `pg_partman`, `pg_stat_kcache`, `pglogical`.

**Paramètres clés** :
- `max_worker_processes` (défaut 8)
- `max_parallel_workers` (sous-ensemble dédié aux requêtes parallèles)
- `max_parallel_workers_per_gather`

---

### AIO
***Asynchronous I/O***

🆕 **Nouveauté PostgreSQL 18** : sous-système d'entrées/sorties asynchrones pour les lectures (et certaines écritures). Au lieu d'attendre une page disque, le backend délègue la demande et continue à travailler.

**Trois implémentations sélectionnables** :
- `worker` : *background workers* dédiés (portable, défaut).
- `io_uring` : interface Linux moderne (kernel ≥ 5.1), plus efficace quand disponible.
- `sync` : *fallback* bloquant (comportement antérieur à PG 18).

**Paramètre** : `io_method = worker | io_uring | sync` dans `postgresql.conf`.

**Impact attendu** : meilleurs débits sur les workloads dominés par les lectures séquentielles et le *prefetch* (OLAP, *bitmap heap scans*, `VACUUM`).

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

**Définition** : Identifiant de 128 bits, garanti unique globalement (probabilité de collision négligeable).

**Format** : `550e8400-e29b-41d4-a716-446655440000`

**Avantages** :
- Pas de coordination nécessaire entre serveurs
- Sécurité (pas d'énumération séquentielle exploitable)
- Fusion de bases facilitée

**Versions courantes en pratique** :

| Version | Caractéristique | Génération PostgreSQL |
|---|---|---|
| **v4** | 122 bits aléatoires | `gen_random_uuid()` (natif PG 13+) ou `uuid_generate_v4()` (extension `uuid-ossp`) |
| **v7** | 48 bits timestamp Unix ms + 74 bits aléatoires (RFC 9562) | `uuidv7()` 🆕 **natif PG 18** |

🆕 **Nouveauté PG 18** : `uuidv7()` est intégrée au noyau, **aucune extension** à charger. Les UUID v7 sont **triés naturellement par instant de génération**, ce qui réduit la fragmentation d'index B-tree à l'insertion (problème classique de l'UUID v4).

**Exemple PG 18** :
```sql
CREATE TABLE evenements (
    id      UUID DEFAULT uuidv7() PRIMARY KEY,
    payload JSONB,
    cree_le TIMESTAMPTZ DEFAULT now()
);
```

**Exemple portable (PG 13+)** :
```sql
CREATE TABLE utilisateurs (
    id  UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    nom TEXT
);
```

---

### PITR
**Point-In-Time Recovery**
(Récupération à un Point dans le Temps)

**Définition** : Capacité de restaurer la base à n'importe quel instant précis entre la dernière base de sauvegarde et la dernière transaction archivée.

**Prérequis** :
- `wal_level = replica` (ou `logical`)
- `archive_mode = on` et `archive_command` (ou `archive_library`) qui copie les segments WAL vers un stockage durable
- Une base de sauvegarde cohérente (`pg_basebackup`)
- Les segments WAL archivés couvrant la fenêtre cible

**Cas d'usage** :
- « Annuler » une erreur humaine (suppression accidentelle)
- Récupérer l'état avant une panne logique
- Audits et conformité

**Procédure (PG 12+)** :

1. Restaurer la base de sauvegarde dans un répertoire vide (`pg_basebackup` ou copie + `restore_command`).
2. Configurer la cible dans `postgresql.conf` (ou `postgresql.auto.conf`) :
   ```
   restore_command       = 'cp /backup/wal/%f %p'
   recovery_target_time  = '2026-05-15 14:30:00+02'
   recovery_target_action = 'pause'   # ou 'promote'
   ```
3. Créer le fichier marqueur `recovery.signal` à la racine du `PGDATA`.
4. Démarrer PostgreSQL : `pg_ctl start -D /data/pgdata`.
5. Une fois la cible atteinte, basculer en mode normal : `SELECT pg_wal_replay_resume();` puis `pg_promote()` si besoin.

> 💡 Depuis PostgreSQL 12, le fichier historique `recovery.conf` a disparu : tout passe par les paramètres `recovery_target_*` du `postgresql.conf` et le marqueur `recovery.signal`.

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
ALTER TABLE documents FORCE  ROW LEVEL SECURITY;  -- s'applique aussi au propriétaire  

-- L'identité applicative passe par un GUC custom 'app.user_id' positionné par
-- l'application après connexion (SET LOCAL app.user_id = '42'). Le 2e paramètre
-- 'true' rend la lecture tolérante quand le GUC n'est pas défini (renvoie NULL).
-- (current_user_id() n'est PAS une fonction PostgreSQL standard.)

-- Politique : les users ne voient que leurs documents
CREATE POLICY user_documents ON documents
    FOR SELECT
    USING (user_id = current_setting('app.user_id', true)::int);

-- Politique : un manager voit toute son équipe
CREATE POLICY manager_team ON documents
    FOR SELECT
    USING (
        team_id IN (
            SELECT team_id FROM teams
            WHERE manager_id = current_setting('app.user_id', true)::int
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

**Définition** : Outil tiers de sauvegarde et d'archivage WAL pour PostgreSQL, écrit en Go (le nom n'est pas un acronyme officiel : il dérive de l'outil historique **WAL-E**, dont WAL-G est la réimplémentation performante).

**Fonctionnalités** :
- Sauvegardes de base complètes et **incrémentielles différentielles** (block-level)
- Archivage WAL continu avec compression (lz4, zstd, brotli) et chiffrement
- Backends de stockage : S3, GCS, Azure Blob, Swift, fichier local
- Restauration PITR (intègre la chaîne `restore_command`)

**Alternatives populaires** :
- **pgBackRest** (en C, très utilisé en entreprise, parallélisme natif)
- **Barman** (en Python, l'outil historique de 2ndQuadrant/EDB)
- **pg_probackup** (en C, fork de pg_arman par Postgres Pro)

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
-- Export — ⚠️ COPY (SQL) lit/écrit côté SERVEUR (superuser ou rôle pg_write_server_files)
COPY (SELECT * FROM clients) TO '/tmp/clients.csv' CSV HEADER;

-- Import
COPY clients FROM '/tmp/clients.csv' CSV HEADER;
```

```sql
-- Équivalent côté CLIENT (psql) — pas de privilège spécial, fichier local au poste
\copy (SELECT * FROM clients) TO '/tmp/clients.csv' CSV HEADER
\copy clients FROM '/tmp/clients.csv' CSV HEADER
```

> 💡 **Piège classique** : `COPY` (SQL) accède au système de fichiers du **serveur** ; `\copy` (méta-commande psql) accède à celui du **client**. Les deux ont la même syntaxe SQL, mais des privilèges et chemins différents.

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
