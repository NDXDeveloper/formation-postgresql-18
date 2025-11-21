🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Commandes psql Essentielles - Export/Import

## Introduction

L'import et l'export de données sont des opérations fondamentales dans la gestion de bases de données. Que ce soit pour :
- 📤 **Exporter** des données vers Excel ou un autre système
- 📥 **Importer** des données depuis un fichier CSV
- 💾 **Sauvegarder** des résultats de requêtes
- 🔄 **Migrer** des données entre environnements
- 📊 **Partager** des données avec des collègues

Ce guide vous montre comment utiliser les trois commandes essentielles de psql : `\copy`, `\i` et `\o`.

---

## Redirection de Sortie : \o

### Qu'est-ce que \o ?

`\o` (pour "output") redirige **toute** la sortie de psql vers un fichier au lieu de l'afficher à l'écran. C'est comme dire à psql : "Écris tout ce que tu affiches dans ce fichier".

### Syntaxe

```
\o [FILENAME]
```

**Sans argument** : Revient à l'affichage normal (écran).

---

### Utilisation de Base

#### Rediriger vers un fichier

```bash
# Commencer à écrire dans un fichier
\o /tmp/resultats.txt

# Tout ce qui suit sera écrit dans le fichier
SELECT * FROM clients;
SELECT COUNT(*) FROM commandes;

# Revenir à l'affichage normal
\o

# Vérifier le contenu
\! cat /tmp/resultats.txt
```

**Résultat dans le fichier** :
```
 id |  nom   | prenom |        email
----+--------+--------+----------------------
  1 | Dupont | Alice  | alice.dupont@mail.fr
  2 | Martin | Bob    | bob.martin@mail.fr
(2 rows)

 count
-------
   150
(1 row)
```

---

### Mode Append (Ajout)

Par défaut, `\o` **écrase** le fichier. Pour **ajouter** au fichier existant :

```bash
# Écraser le fichier (défaut)
\o /tmp/log.txt
SELECT 'Premier résultat';
\o

# Ajouter au fichier
\o /tmp/log.txt
SELECT 'Deuxième résultat';
\o
```

**Note** : Sur certains systèmes, vous pouvez utiliser `>>` pour forcer l'ajout :
```bash
\o | cat >> /tmp/log.txt
```

---

### Redirection vers Pipe (Commande Système)

Vous pouvez rediriger vers une commande système :

```bash
# Envoyer directement vers less (paginateur)
\o | less
SELECT * FROM huge_table;
\o

# Envoyer vers grep pour filtrer
\o | grep 'Paris'
SELECT * FROM clients;
\o

# Linux : Copier dans le presse-papier
\o | xclip -selection clipboard
SELECT * FROM clients;
\o

# macOS : Copier dans le presse-papier
\o | pbcopy
SELECT * FROM clients;
\o
```

---

### Exemples Pratiques avec \o

#### Exporter un rapport formaté

```bash
# Configuration pour un rapport propre
\pset border 2
\pset format aligned
\pset title 'Rapport des Ventes - Novembre 2024'

# Rediriger vers fichier
\o /tmp/rapport_ventes.txt

-- Requête de rapport
SELECT
    DATE_TRUNC('day', date_commande) AS jour,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS total_ventes
FROM commandes
WHERE date_commande >= '2024-11-01'
GROUP BY 1
ORDER BY 1;

-- Ajouter des statistiques
\echo ''
\echo '=== STATISTIQUES ==='
SELECT
    'Total' AS type,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS total
FROM commandes
WHERE date_commande >= '2024-11-01';

\o

\echo 'Rapport sauvegardé dans /tmp/rapport_ventes.txt'
```

---

#### Créer un fichier de log de toutes les opérations

```bash
# Enregistrer toutes les opérations dans un log
\o /tmp/session_$(date +%Y%m%d_%H%M%S).log

\echo '=== Session PostgreSQL ==='
\echo 'Date:' `date`
\conninfo

-- Vos opérations...
SELECT version();
\dt

\o

\echo 'Session enregistrée'
```

---

### Limitations de \o

⚠️ **Attention** : `\o` redirige **tout**, y compris :
- Les en-têtes de colonnes
- Les compteurs de lignes "(X rows)"
- Les messages de psql
- Les méta-commandes comme `\dt`

Pour un export propre de **données seules**, utilisez `\copy` (voir section suivante).

---

## Import/Export de Données : \copy

### Différence entre COPY et \copy

Il existe **deux commandes** similaires mais différentes :

| Commande | Type | Exécution | Permissions | Fichiers |
|----------|------|-----------|-------------|----------|
| `COPY` | SQL | Serveur | Superuser | Serveur uniquement |
| `\copy` | psql | Client | User normal | Client (votre machine) |

**Recommandation** : Utilisez toujours `\copy` sauf si vous avez une raison spécifique d'utiliser `COPY`.

---

### \copy - Syntaxe Générale

```
\copy { table [(columns)] | (query) }
      { FROM | TO } filename
      [ WITH ] [ BINARY ]
               [ DELIMITER 'char' ]
               [ NULL 'string' ]
               [ CSV [ HEADER ] [ QUOTE 'char' ] [ ESCAPE 'char' ] ]
```

**Simplifié pour débutants** :
```bash
# Export
\copy nom_table TO 'fichier.csv' CSV HEADER

# Import
\copy nom_table FROM 'fichier.csv' CSV HEADER
```

---

### Export de Données avec \copy

#### Export Simple (CSV)

```bash
# Exporter toute une table en CSV avec en-têtes
\copy clients TO '/tmp/clients.csv' CSV HEADER

# Résultat : fichier clients.csv créé
```

**Contenu du fichier** :
```csv
id,nom,prenom,email,ville
1,Dupont,Alice,alice.dupont@mail.fr,Paris
2,Martin,Bob,bob.martin@mail.fr,Lyon
3,Bernard,Claire,claire.bernard@mail.fr,Marseille
```

---

#### Export de Colonnes Spécifiques

```bash
# Exporter seulement certaines colonnes
\copy clients (nom, prenom, email) TO '/tmp/contacts.csv' CSV HEADER
```

**Résultat** :
```csv
nom,prenom,email
Dupont,Alice,alice.dupont@mail.fr
Martin,Bob,bob.martin@mail.fr
```

---

#### Export du Résultat d'une Requête

```bash
# Exporter le résultat d'une SELECT
\copy (SELECT nom, prenom FROM clients WHERE ville = 'Paris') TO '/tmp/clients_paris.csv' CSV HEADER

# Requête plus complexe
\copy (
    SELECT
        c.nom,
        c.prenom,
        COUNT(o.id) AS nb_commandes,
        SUM(o.montant_total) AS total_achats
    FROM clients c
    LEFT JOIN commandes o ON c.id = o.client_id
    GROUP BY c.id, c.nom, c.prenom
    ORDER BY total_achats DESC
) TO '/tmp/meilleurs_clients.csv' CSV HEADER
```

---

#### Export TSV (Tab-Separated Values)

```bash
# Utiliser des tabulations comme séparateur
\copy clients TO '/tmp/clients.tsv' DELIMITER E'\t' CSV HEADER

# E'\t' = tabulation (escape sequence)
```

---

#### Export avec Séparateur Personnalisé

```bash
# Séparateur pipe |
\copy clients TO '/tmp/clients.txt' DELIMITER '|' CSV HEADER

# Séparateur point-virgule ; (Excel français)
\copy clients TO '/tmp/clients.csv' DELIMITER ';' CSV HEADER
```

---

#### Gérer les Valeurs NULL

```bash
# Remplacer NULL par une chaîne personnalisée
\copy clients TO '/tmp/clients.csv' CSV HEADER NULL 'N/A'

# Ou vide (défaut)
\copy clients TO '/tmp/clients.csv' CSV HEADER NULL ''
```

**Exemple** :
```csv
id,nom,prenom,email,ville
1,Dupont,Alice,alice@mail.fr,Paris
2,Martin,Bob,bob@mail.fr,N/A
```

---

#### Export Sans En-tête

```bash
# Données uniquement, sans ligne d'en-tête
\copy clients TO '/tmp/clients_no_header.csv' CSV
```

**Résultat** :
```csv
1,Dupont,Alice,alice.dupont@mail.fr,Paris
2,Martin,Bob,bob.martin@mail.fr,Lyon
```

---

### Import de Données avec \copy

#### Import Simple (CSV)

```bash
# Importer un fichier CSV dans une table existante
\copy clients FROM '/tmp/clients.csv' CSV HEADER
```

**Prérequis** : La table `clients` doit exister avec les bonnes colonnes.

---

#### Import dans des Colonnes Spécifiques

```bash
# Importer seulement certaines colonnes
\copy clients (nom, prenom, email) FROM '/tmp/contacts.csv' CSV HEADER

# Les autres colonnes prendront leur valeur par défaut ou NULL
```

---

#### Import avec Séparateur Personnalisé

```bash
# Import TSV
\copy clients FROM '/tmp/clients.tsv' DELIMITER E'\t' CSV HEADER

# Import avec point-virgule
\copy clients FROM '/tmp/clients.csv' DELIMITER ';' CSV HEADER

# Import avec pipe
\copy clients FROM '/tmp/clients.txt' DELIMITER '|' CSV HEADER
```

---

#### Gérer les Valeurs NULL à l'Import

```bash
# Interpréter 'N/A' comme NULL
\copy clients FROM '/tmp/clients.csv' CSV HEADER NULL 'N/A'

# Interpréter les chaînes vides comme NULL
\copy clients FROM '/tmp/clients.csv' CSV HEADER NULL ''
```

---

#### Import Sans En-tête

Si votre fichier n'a pas de ligne d'en-tête :

```bash
# Ne pas spécifier HEADER
\copy clients FROM '/tmp/clients_no_header.csv' CSV
```

---

### Formats de Fichiers Supportés

#### CSV (Comma-Separated Values)

Le format le plus courant et universel.

**Caractéristiques** :
- Séparateur : virgule `,`
- Guillemets : `"` pour les valeurs contenant des virgules
- Standard : RFC 4180

**Exemple** :
```csv
id,nom,description,prix
1,Produit A,"Description avec, virgule",29.99
2,Produit B,"Description avec ""guillemets""",49.99
```

---

#### TSV (Tab-Separated Values)

Alternative au CSV, utilise des tabulations.

**Avantages** :
- Pas de problème avec les virgules dans les données
- Plus simple pour certains parsers

**Inconvénient** :
- Les tabulations dans les données posent problème

---

#### Format Texte (Text)

Format PostgreSQL natif, non-standard.

```bash
# Export en format text
\copy clients TO '/tmp/clients.txt'

# Import en format text
\copy clients FROM '/tmp/clients.txt'
```

**Caractéristiques** :
- Séparateur : tabulation par défaut
- NULL : représenté par `\N`
- Pas de guillemets

---

#### Format Binaire

Format propriétaire PostgreSQL, très rapide mais non-portable.

```bash
# Export binaire
\copy clients TO '/tmp/clients.bin' BINARY

# Import binaire
\copy clients FROM '/tmp/clients.bin' BINARY
```

**Usage** : Transferts de données entre bases PostgreSQL uniquement.

---

### Gestion des Erreurs à l'Import

#### Erreur : Colonne manquante

```bash
\copy clients FROM '/tmp/clients.csv' CSV HEADER

# ERROR:  column "date_creation" of relation "clients" does not exist
```

**Solutions** :
1. Spécifier les colonnes explicitement
2. Ajouter les colonnes manquantes au fichier
3. Créer les colonnes dans la table

```bash
# Solution 1 : Spécifier les colonnes
\copy clients (nom, prenom, email) FROM '/tmp/clients.csv' CSV HEADER
```

---

#### Erreur : Type incompatible

```bash
\copy commandes FROM '/tmp/commandes.csv' CSV HEADER

# ERROR:  invalid input syntax for type integer: "abc"
```

**Solution** : Nettoyer les données avant import ou utiliser une table de staging.

---

#### Erreur : Contrainte violée

```bash
\copy clients FROM '/tmp/clients.csv' CSV HEADER

# ERROR:  duplicate key value violates unique constraint "clients_email_key"
# DETAIL:  Key (email)=(alice@mail.fr) already exists.
```

**Solutions** :
1. Nettoyer les doublons dans le fichier
2. Utiliser une stratégie ON CONFLICT (nécessite SQL, pas \copy)

---

### Exemples Pratiques Complets

#### Exporter pour Excel

```bash
# Configuration optimale pour Excel (français)
\copy clients TO '/tmp/clients_excel.csv'
    DELIMITER ';'
    CSV HEADER
    ENCODING 'UTF8'

# Excel ouvrira automatiquement ce fichier correctement
```

---

#### Exporter avec Jointures

```bash
# Export d'un rapport complexe
\copy (
    SELECT
        c.nom AS client,
        c.email,
        o.date_commande,
        o.montant_total,
        o.statut
    FROM clients c
    INNER JOIN commandes o ON c.id = o.client_id
    WHERE o.date_commande >= '2024-01-01'
    ORDER BY o.date_commande DESC
) TO '/tmp/commandes_2024.csv' CSV HEADER
```

---

#### Import Massif avec Table Temporaire

```bash
# 1. Créer une table temporaire pour l'import
CREATE TEMP TABLE temp_clients (
    nom TEXT,
    prenom TEXT,
    email TEXT,
    ville TEXT
);

# 2. Importer dans la table temporaire (tolérante)
\copy temp_clients FROM '/tmp/nouveaux_clients.csv' CSV HEADER

# 3. Nettoyer et valider
DELETE FROM temp_clients WHERE email IS NULL;
DELETE FROM temp_clients WHERE email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$';

# 4. Insérer dans la vraie table
INSERT INTO clients (nom, prenom, email, ville)
SELECT nom, prenom, email, ville
FROM temp_clients
ON CONFLICT (email) DO NOTHING;

# 5. Vérifier les résultats
SELECT
    (SELECT COUNT(*) FROM temp_clients) AS fichier,
    (SELECT COUNT(*) FROM clients WHERE created_at > NOW() - INTERVAL '1 minute') AS inseres;
```

---

#### Backup et Restore de Table

```bash
# Backup
\copy clients TO '/backup/clients_$(date +%Y%m%d).csv' CSV HEADER

# Restore (dans une table vide)
TRUNCATE clients RESTART IDENTITY CASCADE;
\copy clients FROM '/backup/clients_20241121.csv' CSV HEADER
```

---

## Exécution de Scripts : \i

### Qu'est-ce que \i ?

`\i` (pour "input") exécute un fichier SQL comme si vous tapiez son contenu dans psql. C'est l'équivalent de "copier-coller" le contenu d'un fichier.

### Syntaxe

```
\i FILENAME
\ir FILENAME  # Chemin relatif au script actuel
```

---

### Utilisation de Base

#### Créer un Script SQL

```bash
# Créer un fichier script.sql
nano /tmp/script.sql
```

**Contenu du fichier** :
```sql
-- script.sql
-- Script de configuration de la base

BEGIN;

-- Créer des tables
CREATE TABLE IF NOT EXISTS categories (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE IF NOT EXISTS produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,
    categorie_id INTEGER REFERENCES categories(id),
    prix NUMERIC(10, 2)
);

-- Insérer des données
INSERT INTO categories (nom) VALUES ('Électronique'), ('Vêtements'), ('Livres');

INSERT INTO produits (nom, categorie_id, prix) VALUES
    ('Ordinateur', 1, 999.99),
    ('T-shirt', 2, 19.99),
    ('Roman', 3, 12.99);

COMMIT;

\echo 'Script exécuté avec succès !'
```

---

#### Exécuter le Script

```bash
# Depuis psql
\i /tmp/script.sql

# Résultat :
# BEGIN
# CREATE TABLE
# CREATE TABLE
# INSERT 0 3
# INSERT 0 3
# COMMIT
# Script exécuté avec succès !
```

---

### Scripts avec Chemins Relatifs : \ir

```bash
# Structure de fichiers
# /projet/
#   ├── main.sql
#   ├── schemas/
#   │   └── create_tables.sql
#   └── data/
#       └── insert_data.sql
```

**main.sql** :
```sql
-- main.sql
\echo 'Démarrage de l''installation...'

-- Utiliser \ir pour chemin relatif
\ir schemas/create_tables.sql
\ir data/insert_data.sql

\echo 'Installation terminée !'
```

**Exécution** :
```bash
cd /projet
psql -U postgres -d ma_base -f main.sql

# Ou depuis psql déjà connecté
\i /projet/main.sql
```

---

### Scripts Interactifs

Vous pouvez combiner `\i` avec `\prompt` pour des scripts interactifs :

**backup_table.sql** :
```sql
-- backup_table.sql
\prompt 'Nom de la table à sauvegarder : ' table_name
\prompt 'Chemin de sauvegarde : ' backup_path

\echo 'Sauvegarde de :table_name vers :backup_path'

\copy :table_name TO :'backup_path' CSV HEADER

\echo 'Sauvegarde terminée !'
```

**Utilisation** :
```bash
\i backup_table.sql
# Nom de la table à sauvegarder : clients
# Chemin de sauvegarde : /tmp/clients_backup.csv
# Sauvegarde de clients vers /tmp/clients_backup.csv
# Sauvegarde terminée !
```

---

### Scripts avec Gestion d'Erreurs

```sql
-- setup_safe.sql
-- Script avec gestion d'erreurs

-- Arrêter sur erreur
\set ON_ERROR_STOP on

BEGIN;

\echo 'Création des tables...'
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL
);

\echo 'Insertion des données de test...'
INSERT INTO users (username) VALUES ('alice'), ('bob')
ON CONFLICT (username) DO NOTHING;

COMMIT;

\echo 'Setup terminé avec succès'
```

---

### Scripts de Migration

**migration_001_add_users.sql** :
```sql
-- Migration #001 : Ajout table users
-- Date : 2024-11-21
-- Auteur : Dev Team

\echo '=== Migration 001 : Début ==='

BEGIN;

-- Vérification pré-migration
DO $$
BEGIN
    IF EXISTS (SELECT 1 FROM information_schema.tables WHERE table_name = 'users') THEN
        RAISE EXCEPTION 'Table users existe déjà. Migration annulée.';
    END IF;
END $$;

-- Création
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Index
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);

-- Enregistrer la migration
CREATE TABLE IF NOT EXISTS migrations (
    id SERIAL PRIMARY KEY,
    version VARCHAR(50) UNIQUE NOT NULL,
    applied_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO migrations (version) VALUES ('001_add_users');

COMMIT;

\echo '=== Migration 001 : Succès ==='
```

**Exécution** :
```bash
\i migrations/migration_001_add_users.sql
```

---

### Scripts de Déploiement Complet

**deploy.sql** :
```sql
-- deploy.sql
-- Déploiement complet de l'application

\set ON_ERROR_STOP on

\echo ''
\echo '╔════════════════════════════════════════╗'
\echo '║   DÉPLOIEMENT APPLICATION v1.0         ║'
\echo '╚════════════════════════════════════════╝'
\echo ''

-- 1. Schémas
\echo '→ Création des schémas...'
\ir schemas/create_schemas.sql

-- 2. Types et domaines
\echo '→ Création des types...'
\ir types/custom_types.sql

-- 3. Tables
\echo '→ Création des tables...'
\ir tables/create_tables.sql

-- 4. Index
\echo '→ Création des index...'
\ir indexes/create_indexes.sql

-- 5. Fonctions
\echo '→ Création des fonctions...'
\ir functions/utility_functions.sql

-- 6. Triggers
\echo '→ Création des triggers...'
\ir triggers/audit_triggers.sql

-- 7. Vues
\echo '→ Création des vues...'
\ir views/reporting_views.sql

-- 8. Permissions
\echo '→ Configuration des permissions...'
\ir security/grants.sql

-- 9. Données de référence
\echo '→ Chargement des données...'
\ir data/reference_data.sql

\echo ''
\echo '✓ Déploiement terminé avec succès !'
\echo ''
```

---

## Combinaison \o et \copy

### Exporter des Résultats Formatés ET Données Brutes

```bash
# 1. Rapport formaté pour humains
\o /tmp/rapport_humain.txt
\pset border 2
\pset title 'Rapport des Ventes'

SELECT
    DATE_TRUNC('month', date_commande) AS mois,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS total
FROM commandes
GROUP BY 1
ORDER BY 1;

\o

# 2. Données brutes pour traitement
\copy (
    SELECT
        DATE_TRUNC('month', date_commande) AS mois,
        COUNT(*) AS nb_commandes,
        SUM(montant_total) AS total
    FROM commandes
    GROUP BY 1
    ORDER BY 1
) TO '/tmp/rapport_data.csv' CSV HEADER
```

---

## Cas d'Usage Réels

### Migration d'une Table vers un Autre Serveur

```bash
# Sur le serveur source
psql -h source.example.com -U postgres -d prod

\copy clients TO '/tmp/clients_export.csv' CSV HEADER
\copy commandes TO '/tmp/commandes_export.csv' CSV HEADER

# Transférer les fichiers (exemple avec scp)
\! scp /tmp/*.csv user@destination.example.com:/tmp/

# Sur le serveur destination
psql -h destination.example.com -U postgres -d prod

\copy clients FROM '/tmp/clients_export.csv' CSV HEADER
\copy commandes FROM '/tmp/commandes_export.csv' CSV HEADER
```

---

### Backup Sélectif

```bash
# Script de backup sélectif
-- backup_important.sql

\set backup_dir '/backup/daily'

\echo 'Backup des tables critiques...'

\copy clients TO :'backup_dir'/clients.csv CSV HEADER
\copy commandes TO :'backup_dir'/commandes.csv CSV HEADER
\copy produits TO :'backup_dir'/produits.csv CSV HEADER

\! date > :backup_dir/backup_date.txt

\echo 'Backup terminé !'
```

---

### Import avec Validation

```sql
-- import_with_validation.sql

BEGIN;

-- Table temporaire
CREATE TEMP TABLE staging_clients (
    nom TEXT,
    prenom TEXT,
    email TEXT,
    telephone TEXT
);

-- Import
\echo 'Import des données...'
\copy staging_clients FROM '/tmp/nouveaux_clients.csv' CSV HEADER

-- Validation
\echo 'Validation des données...'

-- Compter les problèmes
SELECT
    COUNT(*) FILTER (WHERE nom IS NULL OR nom = '') AS sans_nom,
    COUNT(*) FILTER (WHERE email IS NULL OR email = '') AS sans_email,
    COUNT(*) FILTER (WHERE email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$') AS email_invalide
FROM staging_clients;

-- Supprimer les invalides
DELETE FROM staging_clients
WHERE nom IS NULL OR nom = ''
   OR email IS NULL OR email = ''
   OR email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$';

-- Insérer les valides
INSERT INTO clients (nom, prenom, email, telephone)
SELECT nom, prenom, email, telephone
FROM staging_clients
ON CONFLICT (email) DO UPDATE
SET
    nom = EXCLUDED.nom,
    prenom = EXCLUDED.prenom,
    telephone = EXCLUDED.telephone,
    updated_at = NOW();

-- Rapport
SELECT
    (SELECT COUNT(*) FROM staging_clients) AS importes,
    (SELECT COUNT(*) FROM clients WHERE updated_at > NOW() - INTERVAL '1 minute') AS traites;

COMMIT;
```

---

### Export Multi-Formats

```sql
-- export_multi_format.sql

\set export_date `date +%Y%m%d`

\echo 'Export multi-formats...'

-- 1. CSV pour Excel
\copy (SELECT * FROM clients) TO '/exports/clients_:export_date.csv'
    DELIMITER ';' CSV HEADER

-- 2. TSV pour analyse
\copy (SELECT * FROM clients) TO '/exports/clients_:export_date.tsv'
    DELIMITER E'\t' CSV HEADER

-- 3. Binaire pour PostgreSQL
\copy clients TO '/exports/clients_:export_date.bin' BINARY

-- 4. JSON (via requête)
\o /exports/clients_:export_date.json
SELECT json_agg(row_to_json(clients)) FROM clients;
\o

\echo 'Exports terminés :'
\! ls -lh /exports/clients_:export_date.*
```

---

## Bonnes Pratiques

### ✅ À Faire

#### 1. Toujours utiliser des chemins absolus

```bash
# ✅ Bon
\copy clients FROM '/home/user/data/clients.csv' CSV HEADER

# ❌ Éviter (peut être ambigu)
\copy clients FROM 'clients.csv' CSV HEADER
```

---

#### 2. Spécifier HEADER pour les CSV

```bash
# ✅ Explicite et clair
\copy clients TO '/tmp/clients.csv' CSV HEADER

# ❌ Sans HEADER, difficile à réutiliser
\copy clients TO '/tmp/clients.csv' CSV
```

---

#### 3. Utiliser des noms de fichiers avec date

```bash
# ✅ Bon
\copy clients TO '/backup/clients_20241121.csv' CSV HEADER

# ❌ Risque d'écrasement
\copy clients TO '/backup/clients.csv' CSV HEADER
```

---

#### 4. Valider avant import massif

```sql
-- ✅ Toujours tester d'abord
CREATE TEMP TABLE test_import (...);
\copy test_import FROM '/data/huge_file.csv' CSV HEADER LIMIT 100
-- Vérifier que tout est OK
-- Puis importer en vrai
```

---

#### 5. Documenter vos scripts

```sql
-- ✅ Script auto-documenté
-- Nom: import_clients.sql
-- But: Importer les nouveaux clients depuis le CRM
-- Auteur: Jean Dupont
-- Date: 2024-11-21
-- Usage: psql -f import_clients.sql
--
-- Prérequis:
--   - Table clients doit exister
--   - Fichier /data/clients.csv doit être présent
--

\echo 'Import des clients...'
\copy clients FROM '/data/clients.csv' CSV HEADER
```

---

### ❌ À Éviter

#### 1. Import sans validation

```bash
# ❌ Dangereux : import direct en production
\copy clients FROM '/tmp/unknown_data.csv' CSV HEADER

# ✅ Mieux : table temporaire d'abord
CREATE TEMP TABLE staging AS SELECT * FROM clients LIMIT 0;
\copy staging FROM '/tmp/unknown_data.csv' CSV HEADER
-- Valider, puis insérer
```

---

#### 2. Écraser des fichiers importants

```bash
# ❌ Risque de perte de données
\o /tmp/important_data.txt
SELECT * FROM autre_chose;  -- Écrase le fichier !
\o
```

---

#### 3. Chemins avec espaces non-quotés

```bash
# ❌ Erreur sur les chemins avec espaces
\copy clients TO '/backup/mes fichiers/clients.csv' CSV HEADER

# ✅ Solution
\copy clients TO '/backup/mes_fichiers/clients.csv' CSV HEADER
```

---

#### 4. Oublier le HEADER

```bash
# ❌ Fichier difficile à réutiliser
\copy clients TO '/tmp/clients.csv' CSV

# ✅ Avec HEADER, plus facile
\copy clients TO '/tmp/clients.csv' CSV HEADER
```

---

## Dépannage des Erreurs Courantes

### Erreur : Permission denied

```bash
\copy clients TO '/root/clients.csv' CSV HEADER
# ERROR:  could not open file "/root/clients.csv" for writing: Permission denied
```

**Solution** : Utiliser un répertoire où vous avez les droits (ex: `/tmp/`, votre home).

---

### Erreur : No such file or directory

```bash
\copy clients FROM '/tmp/fichier_inexistant.csv' CSV HEADER
# ERROR:  /tmp/fichier_inexistant.csv: No such file or directory
```

**Solutions** :
1. Vérifier que le fichier existe : `\! ls -l /tmp/fichier_inexistant.csv`
2. Vérifier le chemin
3. Vérifier les permissions

---

### Erreur : Invalid byte sequence

```bash
\copy clients FROM '/tmp/clients.csv' CSV HEADER
# ERROR:  invalid byte sequence for encoding "UTF8"
```

**Solution** : Spécifier l'encodage correct
```bash
\copy clients FROM '/tmp/clients.csv' CSV HEADER ENCODING 'LATIN1'
```

---

### Erreur : Extra data after last expected column

```bash
\copy clients FROM '/tmp/clients.csv' CSV HEADER
# ERROR:  extra data after last expected column
# CONTEXT:  COPY clients, line 5: "Dupont,Alice,alice@mail.fr,extra_column"
```

**Solutions** :
1. Fichier a plus de colonnes que la table
2. Spécifier les colonnes explicitement
3. Corriger le fichier

```bash
# Solution : spécifier les colonnes
\copy clients (nom, prenom, email) FROM '/tmp/clients.csv' CSV HEADER
```

---

## Commandes Avancées

### \copy avec Programme Externe

```bash
# Importer depuis un programme
\copy clients FROM PROGRAM 'curl https://api.example.com/clients.csv' CSV HEADER

# Exporter vers un programme
\copy clients TO PROGRAM 'gzip > /backup/clients.csv.gz' CSV HEADER

# Filtrer à la volée
\copy (SELECT * FROM logs WHERE created > NOW() - INTERVAL '1 day')
    TO PROGRAM 'gzip > /backup/logs_today.csv.gz' CSV HEADER
```

---

### Export avec STDIN/STDOUT

```bash
# Export vers stdout (affichage)
\copy clients TO STDOUT CSV HEADER

# Import depuis stdin
echo "1,Test,User,test@mail.fr" | psql -c "\copy clients FROM STDIN CSV"
```

---

## Scripts Utilitaires

### Script de Backup Automatique

**daily_backup.sh** :
```bash
#!/bin/bash

DATE=$(date +%Y%m%d)
BACKUP_DIR="/backup/$DATE"
mkdir -p "$BACKUP_DIR"

psql -U postgres -d production <<EOF
\o $BACKUP_DIR/backup.log

\echo 'Backup du $(date)'
\echo '===================='

\copy clients TO '$BACKUP_DIR/clients.csv' CSV HEADER
\copy commandes TO '$BACKUP_DIR/commandes.csv' CSV HEADER
\copy produits TO '$BACKUP_DIR/produits.csv' CSV HEADER

\echo ''
\echo 'Tailles des exports :'
\! ls -lh $BACKUP_DIR/*.csv

\echo ''
\echo 'Backup terminé avec succès'
\o
EOF

# Compression
cd "$BACKUP_DIR"
tar czf "../backup_$DATE.tar.gz" *.csv
rm *.csv

echo "Backup sauvegardé dans /backup/backup_$DATE.tar.gz"
```

---

### Script d'Import Robuste

**import_robust.sql** :
```sql
\set ON_ERROR_STOP on

BEGIN;

-- Log
CREATE TEMP TABLE import_log (
    timestamp TIMESTAMP DEFAULT NOW(),
    message TEXT
);

-- Staging
CREATE TEMP TABLE staging AS SELECT * FROM clients LIMIT 0;

-- Import
INSERT INTO import_log (message) VALUES ('Début import...');

\copy staging FROM '/data/clients.csv' CSV HEADER

-- Validation
INSERT INTO import_log (message)
VALUES ('Lignes importées: ' || (SELECT COUNT(*) FROM staging)::TEXT);

-- Nettoyage
DELETE FROM staging WHERE email IS NULL;

INSERT INTO import_log (message)
VALUES ('Lignes invalides supprimées: ' ||
        (SELECT COUNT(*) FROM staging WHERE email IS NULL)::TEXT);

-- Insertion
INSERT INTO clients SELECT * FROM staging
ON CONFLICT (email) DO NOTHING;

INSERT INTO import_log (message)
VALUES ('Lignes insérées: ' || (SELECT COUNT(*) FROM clients
        WHERE created_at > NOW() - INTERVAL '1 second')::TEXT);

-- Rapport
SELECT * FROM import_log ORDER BY timestamp;

COMMIT;
```

---

## Résumé des Commandes

| Commande | Usage | Exemple |
|----------|-------|---------|
| `\o FILE` | Rediriger sortie | `\o /tmp/results.txt` |
| `\o` | Sortie normale | `\o` |
| `\copy TO` | Export données | `\copy clients TO '/tmp/c.csv' CSV HEADER` |
| `\copy FROM` | Import données | `\copy clients FROM '/tmp/c.csv' CSV HEADER` |
| `\i FILE` | Exécuter script | `\i /scripts/setup.sql` |
| `\ir FILE` | Script relatif | `\ir ../data.sql` |

---

## Conclusion

L'import/export de données est une compétence essentielle. Les points clés :

🎯 **Essentiels** :
- `\copy` pour importer/exporter des données (CSV)
- `\o` pour sauvegarder des résultats formatés
- `\i` pour exécuter des scripts SQL

📋 **Formats** :
- CSV avec HEADER pour la compatibilité
- TSV pour éviter les problèmes de virgules
- Binaire pour performance PostgreSQL-à-PostgreSQL

🛡️ **Sécurité** :
- Toujours valider avant import massif
- Utiliser des tables temporaires (staging)
- Documenter vos scripts

💡 **Astuces** :
- Noms de fichiers avec dates
- Chemins absolus
- Gestion d'erreurs dans les scripts
- Logs et audits

---

**Prochaines sections** :
- Meta-commandes avancées
- Scripts et automatisation
- Intégration avec des outils externes

---


⏭️ [Meta-commandes avancées](/annexes/commandes-psql/04-meta-commandes-avancees.md)
