🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.6. Domaines et Types Personnalisés (CREATE TYPE)

## Introduction

PostgreSQL offre une fonctionnalité puissante : la possibilité de créer vos **propres types de données**. Cette fonctionnalité permet de :

- ✅ **Réutiliser** des définitions de types avec contraintes  
- ✅ **Standardiser** les types de données à travers votre application  
- ✅ **Simplifier** la maintenance du schéma  
- ✅ **Documenter** les intentions métier dans le schéma

Dans cette section, nous allons explorer :
- **DOMAIN** : Types basés sur des types existants avec contraintes personnalisées  
- **COMPOSITE TYPE** : Types structurés avec plusieurs champs  
- **ENUM** : Types énumérés (rappel et approfondissement)  
- **RANGE TYPE** : Types pour représenter des plages de valeurs

---

## 1. Domaines (DOMAIN)

### Qu'est-ce qu'un Domaine ?

Un **domaine** est un type de données personnalisé basé sur un type existant, avec des **contraintes supplémentaires** et une **valeur par défaut** optionnelle.

### Analogie

Pensez à un domaine comme à un **moule personnalisé** :
- Vous prenez un type de base (ex: VARCHAR)
- Vous ajoutez des règles spécifiques (ex: format d'email)
- Vous réutilisez ce moule partout où nécessaire

**Avantage :** Au lieu de répéter les contraintes dans chaque table, vous les définissez une fois.

### Syntaxe de Base

```sql
CREATE DOMAIN nom_domaine AS type_base
    [DEFAULT expression]
    [CONSTRAINT nom_contrainte CHECK (condition)]
    [NOT NULL];
```

### Premier Exemple : Email

```sql
-- ❌ AVANT : Répéter la contrainte partout
CREATE TABLE utilisateurs (
    email VARCHAR(255) CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

CREATE TABLE clients (
    email VARCHAR(255) CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

-- ✅ APRÈS : Créer un domaine réutilisable
CREATE DOMAIN email_address AS VARCHAR(255)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

-- Utiliser le domaine
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email email_address NOT NULL UNIQUE
);

CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    entreprise VARCHAR(200),
    contact_email email_address
);

-- Insertion valide
INSERT INTO utilisateurs (nom, email) VALUES ('Alice', 'alice@example.com');

-- Insertion invalide (erreur)
INSERT INTO utilisateurs (nom, email) VALUES ('Bob', 'email_invalide');
-- ERROR: value for domain email_address violates check constraint
```

**Avantage :** Si vous devez changer la validation, modifiez le domaine une seule fois !

### Exemples de Domaines Utiles

#### Code Postal Français

```sql
CREATE DOMAIN code_postal_fr AS VARCHAR(5)
    CHECK (VALUE ~ '^[0-9]{5}$');

CREATE TABLE adresses (
    id SERIAL PRIMARY KEY,
    rue VARCHAR(200),
    ville VARCHAR(100),
    code_postal code_postal_fr NOT NULL
);

-- OK
INSERT INTO adresses (rue, ville, code_postal)  
VALUES ('10 rue de la Paix', 'Paris', '75001');  

-- Erreur
INSERT INTO adresses (rue, ville, code_postal)  
VALUES ('20 avenue Victor Hugo', 'Lyon', '6900');  -- Manque un chiffre  
```

#### Numéro de Téléphone

```sql
CREATE DOMAIN telephone_fr AS VARCHAR(20)
    CHECK (VALUE ~ '^0[1-9][0-9]{8}$');

CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    telephone telephone_fr
);

-- OK
INSERT INTO contacts (nom, telephone) VALUES ('Alice', '0612345678');

-- Erreur
INSERT INTO contacts (nom, telephone) VALUES ('Bob', '12345');
```

#### Montant Positif

```sql
CREATE DOMAIN montant_positif AS NUMERIC(10, 2)
    CHECK (VALUE >= 0);

CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    prix montant_positif NOT NULL,
    cout_production montant_positif
);

-- OK
INSERT INTO produits (nom, prix, cout_production)  
VALUES ('Laptop', 999.99, 650.00);  

-- Erreur
INSERT INTO produits (nom, prix) VALUES ('Souris', -10.00);
-- ERROR: value for domain montant_positif violates check constraint
```

#### Note sur 20

```sql
CREATE DOMAIN note_sur_20 AS NUMERIC(4, 2)
    CHECK (VALUE >= 0 AND VALUE <= 20);

CREATE TABLE examens (
    id SERIAL PRIMARY KEY,
    etudiant_id INTEGER,
    matiere VARCHAR(100),
    note note_sur_20 NOT NULL
);
```

#### Année Valide

```sql
CREATE DOMAIN annee_valide AS INTEGER
    CHECK (VALUE >= 1900 AND VALUE <= 2100);

CREATE TABLE films (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300),
    annee_sortie annee_valide NOT NULL
);
```

### Domaine avec Valeur par Défaut

```sql
CREATE DOMAIN statut_actif AS VARCHAR(20)
    DEFAULT 'actif'
    CHECK (VALUE IN ('actif', 'inactif', 'suspendu'));

CREATE TABLE abonnements (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    statut statut_actif  -- Valeur par défaut : 'actif'
);

INSERT INTO abonnements (user_id) VALUES (1);  
SELECT * FROM abonnements;  
-- Résultat : statut = 'actif'
```

### Domaine avec NOT NULL

```sql
CREATE DOMAIN email_obligatoire AS VARCHAR(255)
    NOT NULL
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE TABLE newsletters (
    id SERIAL PRIMARY KEY,
    email email_obligatoire  -- NULL interdit
);

-- Erreur
INSERT INTO newsletters (id) VALUES (1);
-- ERROR: domain email_obligatoire does not allow null values
```

### Domaine avec Plusieurs Contraintes

```sql
CREATE DOMAIN username AS VARCHAR(50)
    CHECK (VALUE ~ '^[a-z0-9_]+$')  -- Seulement minuscules, chiffres, underscore
    CHECK (LENGTH(VALUE) >= 3)       -- Minimum 3 caractères
    CHECK (LENGTH(VALUE) <= 30);     -- Maximum 30 caractères

CREATE TABLE utilisateurs_username (
    id SERIAL PRIMARY KEY,
    username username UNIQUE NOT NULL,
    email email_address
);

-- OK
INSERT INTO utilisateurs_username (username, email)  
VALUES ('alice_2025', 'alice@example.com');  

-- Erreur : majuscule
INSERT INTO utilisateurs_username (username, email)  
VALUES ('Alice', 'alice@example.com');  

-- Erreur : trop court
INSERT INTO utilisateurs_username (username, email)  
VALUES ('ab', 'bob@example.com');  
```

### Modifier un Domaine

```sql
-- Ajouter une contrainte
ALTER DOMAIN email_address  
ADD CONSTRAINT email_length CHECK (LENGTH(VALUE) <= 255);  

-- Supprimer une contrainte
ALTER DOMAIN email_address  
DROP CONSTRAINT email_length;  

-- Changer la valeur par défaut
ALTER DOMAIN statut_actif  
SET DEFAULT 'inactif';  

-- Supprimer la valeur par défaut
ALTER DOMAIN statut_actif  
DROP DEFAULT;  

-- Ajouter NOT NULL
ALTER DOMAIN email_address  
SET NOT NULL;  

-- Supprimer NOT NULL
ALTER DOMAIN email_address  
DROP NOT NULL;  

-- Renommer
ALTER DOMAIN email_address RENAME TO adresse_email;
```

### Supprimer un Domaine

```sql
-- Supprimer (échoue si utilisé)
DROP DOMAIN email_address;

-- Supprimer avec CASCADE (supprime les colonnes utilisant ce domaine)
DROP DOMAIN email_address CASCADE;

-- Supprimer si existe
DROP DOMAIN IF EXISTS email_address;
```

### Voir les Domaines

```sql
-- Dans psql
\dD

-- Avec SQL
SELECT
    n.nspname AS schema,
    t.typname AS domain_name,
    pg_catalog.format_type(t.typbasetype, t.typtypmod) AS base_type
FROM pg_catalog.pg_type t  
JOIN pg_catalog.pg_namespace n ON n.oid = t.typnamespace  
WHERE t.typtype = 'd'  
  AND n.nspname = 'public'
ORDER BY domain_name;
```

---

## 2. Types Composites (COMPOSITE TYPE)

### Qu'est-ce qu'un Type Composite ?

Un **type composite** est un type de données structuré contenant **plusieurs champs** de types différents. C'est similaire à une "struct" en C ou une "class" en POO (sans méthodes).

### Syntaxe

```sql
CREATE TYPE nom_type AS (
    champ1 type1,
    champ2 type2,
    champ3 type3
);
```

### Exemple : Adresse

```sql
-- Créer un type composite pour une adresse
CREATE TYPE adresse_type AS (
    rue VARCHAR(200),
    ville VARCHAR(100),
    code_postal VARCHAR(10),
    pays VARCHAR(50)
);

-- Utiliser dans une table
CREATE TABLE personnes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    adresse_principale adresse_type,
    adresse_secondaire adresse_type
);

-- Insertion avec notation ROW
INSERT INTO personnes (nom, adresse_principale) VALUES (
    'Alice Martin',
    ROW('10 rue de la Paix', 'Paris', '75001', 'France')
);

-- Ou avec notation (...)
INSERT INTO personnes (nom, adresse_principale) VALUES (
    'Bob Dupont',
    ('20 avenue Victor Hugo', 'Lyon', '69001', 'France')
);

SELECT * FROM personnes;
```

### Accéder aux Champs d'un Type Composite

```sql
-- Accès avec notation point
SELECT
    nom,
    (adresse_principale).rue,
    (adresse_principale).ville,
    (adresse_principale).code_postal
FROM personnes;

-- Extraire tout le composite
SELECT nom, adresse_principale FROM personnes;

-- Filtrer sur un champ du composite
SELECT * FROM personnes  
WHERE (adresse_principale).ville = 'Paris';  
```

**Note :** Les parenthèses autour du nom de colonne sont nécessaires pour éviter l'ambiguïté.

### Exemple : Coordonnées Géographiques

```sql
CREATE TYPE coordonnees AS (
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    altitude REAL
);

CREATE TABLE lieux_interet (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    position coordonnees
);

INSERT INTO lieux_interet (nom, position) VALUES (
    'Tour Eiffel',
    ROW(48.858844, 2.294351, 324.0)
);

-- Calculer la distance (exemple simplifié)
SELECT
    nom,
    (position).latitude,
    (position).longitude
FROM lieux_interet;
```

### Exemple : Plage de Dates

```sql
CREATE TYPE periode AS (
    date_debut DATE,
    date_fin DATE
);

CREATE TABLE projets (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    phase_conception periode,
    phase_developpement periode,
    phase_test periode
);

INSERT INTO projets (nom, phase_conception, phase_developpement) VALUES (
    'Projet Alpha',
    ROW('2025-01-01', '2025-02-28'),
    ROW('2025-03-01', '2025-06-30')
);

-- Vérifier si une date est dans une période
SELECT nom  
FROM projets  
WHERE CURRENT_DATE BETWEEN  
    (phase_developpement).date_debut AND
    (phase_developpement).date_fin;
```

### Type Composite avec Contraintes

Les types composites eux-mêmes ne peuvent pas avoir de contraintes CHECK, mais vous pouvez :

```sql
-- 1. Utiliser des domaines dans le composite
CREATE DOMAIN date_valide AS DATE
    CHECK (VALUE >= '2000-01-01' AND VALUE <= '2100-12-31');

CREATE TYPE periode_validee AS (
    date_debut date_valide,
    date_fin date_valide
);

-- 2. Ajouter des contraintes au niveau de la table
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    dates periode,
    CHECK ((dates).date_fin >= (dates).date_debut)
);
```

### Modifier un Type Composite

```sql
-- Ajouter un champ
ALTER TYPE adresse_type ADD ATTRIBUTE complement_adresse VARCHAR(100);

-- Supprimer un champ
ALTER TYPE adresse_type DROP ATTRIBUTE complement_adresse;

-- Changer le type d'un champ
ALTER TYPE adresse_type ALTER ATTRIBUTE pays TYPE VARCHAR(100);

-- Renommer un champ
ALTER TYPE adresse_type RENAME ATTRIBUTE pays TO nom_pays;

-- Renommer le type
ALTER TYPE adresse_type RENAME TO adresse;
```

### Tableaux de Types Composites

```sql
-- Un tableau de types composites
CREATE TABLE entreprises (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    adresses adresse_type[]  -- Tableau d'adresses
);

INSERT INTO entreprises (nom, adresses) VALUES (
    'Tech Corp',
    ARRAY[
        ROW('10 rue A', 'Paris', '75001', 'France')::adresse_type,
        ROW('20 rue B', 'Lyon', '69001', 'France')::adresse_type
    ]
);

-- Accéder au tableau
SELECT
    nom,
    adresses[1] AS siege_social,
    (adresses[2]).ville AS ville_succursale
FROM entreprises;
```

### Cas d'Usage des Types Composites

#### 1. Informations de Contact

```sql
CREATE TYPE contact_info AS (
    telephone VARCHAR(20),
    email VARCHAR(255),
    site_web VARCHAR(500)
);

CREATE TABLE fournisseurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    contact contact_info
);
```

#### 2. Plages Horaires

```sql
CREATE TYPE horaire AS (
    heure_debut TIME,
    heure_fin TIME
);

CREATE TABLE disponibilites (
    id SERIAL PRIMARY KEY,
    employe_id INTEGER,
    jour_semaine VARCHAR(20),
    matin horaire,
    apres_midi horaire
);
```

#### 3. Dimensions d'Objets

```sql
CREATE TYPE dimensions AS (
    longueur NUMERIC(10, 2),
    largeur NUMERIC(10, 2),
    hauteur NUMERIC(10, 2),
    unite VARCHAR(10)
);

CREATE TABLE colis (
    id SERIAL PRIMARY KEY,
    numero_tracking VARCHAR(50),
    dimensions dimensions,
    poids NUMERIC(8, 2)
);
```

---

## 3. Types Énumérés (ENUM) - Rappel

Nous avons déjà vu les ENUM en section 4.4.4, mais rappelons les points clés :

```sql
-- Créer un ENUM
CREATE TYPE statut_commande AS ENUM (
    'en_attente',
    'confirmee',
    'expediee',
    'livree',
    'annulee'
);

CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    numero VARCHAR(50),
    statut statut_commande DEFAULT 'en_attente'
);

-- Avantages :
-- ✓ Validation automatique
-- ✓ Stockage efficace (1-4 octets)
-- ✓ Ordre préservé

-- Modifier
ALTER TYPE statut_commande ADD VALUE 'retournee';  
ALTER TYPE statut_commande ADD VALUE 'en_preparation' BEFORE 'expediee';  
```

---

## 4. Types de Plage (RANGE TYPE)

### Qu'est-ce qu'un Range Type ?

PostgreSQL offre des types natifs pour représenter des **plages de valeurs** (intervalles).

### Types de Plage Natifs

| Type | Type de Valeur | Exemple |
|------|----------------|---------|
| `INT4RANGE` | INTEGER | [1, 10) |
| `INT8RANGE` | BIGINT | [1000, 2000] |
| `NUMRANGE` | NUMERIC | [0.0, 100.0) |
| `TSRANGE` | TIMESTAMP | ['2025-01-01', '2025-12-31') |
| `TSTZRANGE` | TIMESTAMPTZ | ['2025-01-01', '2025-12-31') |
| `DATERANGE` | DATE | ['2025-01-01', '2025-12-31') |

### Notation des Plages

```sql
-- [a, b] : Inclusif des deux côtés (fermé)
SELECT '[1, 10]'::INT4RANGE;  -- 1, 2, 3, ..., 9, 10

-- [a, b) : Inclusif à gauche, exclusif à droite (semi-ouvert)
SELECT '[1, 10)'::INT4RANGE;  -- 1, 2, 3, ..., 9 (pas 10)

-- (a, b] : Exclusif à gauche, inclusif à droite
SELECT '(1, 10]'::INT4RANGE;  -- 2, 3, ..., 9, 10 (pas 1)

-- (a, b) : Exclusif des deux côtés (ouvert)
SELECT '(1, 10)'::INT4RANGE;  -- 2, 3, ..., 9

-- Plage vide
SELECT 'empty'::INT4RANGE;

-- Plage infinie
SELECT '[1, )'::INT4RANGE;    -- De 1 à l'infini  
SELECT '(, 10]'::INT4RANGE;   -- De l'infini négatif à 10  
```

### Exemple : Réservations de Salles

```sql
CREATE TABLE salles (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

CREATE TABLE reservations_salles (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER REFERENCES salles(id),
    periode TSTZRANGE NOT NULL,
    reserve_par VARCHAR(100),

    -- Empêcher les chevauchements
    EXCLUDE USING GIST (salle_id WITH =, periode WITH &&)
);

INSERT INTO salles (nom) VALUES ('Salle A'), ('Salle B');

-- Réserver la salle 1
INSERT INTO reservations_salles (salle_id, periode, reserve_par) VALUES (
    1,
    '[2025-11-19 09:00, 2025-11-19 11:00)'::TSTZRANGE,
    'Alice'
);

-- Réservation valide (pas de chevauchement)
INSERT INTO reservations_salles (salle_id, periode, reserve_par) VALUES (
    1,
    '[2025-11-19 11:00, 2025-11-19 13:00)'::TSTZRANGE,
    'Bob'
);

-- Erreur : chevauchement
INSERT INTO reservations_salles (salle_id, periode, reserve_par) VALUES (
    1,
    '[2025-11-19 10:00, 2025-11-19 12:00)'::TSTZRANGE,
    'Charlie'
);
-- ERROR: conflicting key value violates exclusion constraint
```

### Opérateurs sur les Plages

```sql
-- Contient
SELECT '[1, 10]'::INT4RANGE @> 5;  -- true (5 est dans [1,10])  
SELECT '[1, 10]'::INT4RANGE @> 15; -- false  

-- Est contenu dans
SELECT 5 <@ '[1, 10]'::INT4RANGE;  -- true

-- Chevauche (intersecte)
SELECT '[1, 5]'::INT4RANGE && '[3, 8]'::INT4RANGE;  -- true  
SELECT '[1, 5]'::INT4RANGE && '[6, 10]'::INT4RANGE; -- false  

-- Adjacent
SELECT '[1, 5]'::INT4RANGE -|- '[5, 10]'::INT4RANGE; -- true  
SELECT '[1, 5]'::INT4RANGE -|- '[6, 10]'::INT4RANGE; -- false  

-- Union
SELECT '[1, 5]'::INT4RANGE + '[3, 8]'::INT4RANGE;  -- [1, 8]

-- Intersection
SELECT '[1, 10]'::INT4RANGE * '[5, 15]'::INT4RANGE;  -- [5, 10]

-- Différence
SELECT '[1, 10]'::INT4RANGE - '[5, 15]'::INT4RANGE;  -- [1, 5)
```

### Fonctions sur les Plages

```sql
-- Limites
SELECT lower('[1, 10]'::INT4RANGE);  -- 1  
SELECT upper('[1, 10]'::INT4RANGE);  -- 10  

-- Est vide ?
SELECT isempty('[1, 1]'::INT4RANGE);  -- true  
SELECT isempty('[1, 10]'::INT4RANGE); -- false  

-- Borne inférieure/supérieure est inclusive ?
SELECT lower_inc('[1, 10]'::INT4RANGE);  -- true  
SELECT upper_inc('[1, 10)'::INT4RANGE);  -- false  

-- Borne inférieure/supérieure est infinie ?
SELECT lower_inf('[1, )'::INT4RANGE);   -- false  
SELECT upper_inf('[1, )'::INT4RANGE);   -- true  
```

### Cas d'Usage des Range Types

#### 1. Tarification par Plages de Quantité

```sql
CREATE TABLE tarifs (
    id SERIAL PRIMARY KEY,
    produit_id INTEGER,
    quantite_range INT4RANGE,
    prix_unitaire NUMERIC(10, 2),

    -- Empêcher les chevauchements pour un même produit
    EXCLUDE USING GIST (produit_id WITH =, quantite_range WITH &&)
);

INSERT INTO tarifs (produit_id, quantite_range, prix_unitaire) VALUES
    (1, '[1, 10)'::INT4RANGE, 10.00),   -- 1-9 unités : 10€
    (1, '[10, 50)'::INT4RANGE, 9.00),   -- 10-49 unités : 9€
    (1, '[50, )'::INT4RANGE, 8.00);     -- 50+ unités : 8€

-- Trouver le prix pour 25 unités
SELECT prix_unitaire  
FROM tarifs  
WHERE produit_id = 1  
  AND quantite_range @> 25;
-- Résultat : 9.00
```

#### 2. Périodes d'Abonnement

```sql
CREATE TABLE abonnements (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    type VARCHAR(50),
    periode DATERANGE NOT NULL,

    -- Un utilisateur ne peut avoir qu'un abonnement actif à la fois
    EXCLUDE USING GIST (user_id WITH =, periode WITH &&)
);

INSERT INTO abonnements (user_id, type, periode) VALUES
    (1, 'Premium', '[2025-01-01, 2025-06-30]'::DATERANGE);

-- Vérifier si l'abonnement est actif aujourd'hui
SELECT * FROM abonnements  
WHERE user_id = 1  
  AND periode @> CURRENT_DATE;
```

#### 3. Plages d'Âges

```sql
CREATE TABLE categories_age (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    plage_age INT4RANGE
);

INSERT INTO categories_age (nom, plage_age) VALUES
    ('Enfant', '[0, 13)'::INT4RANGE),
    ('Adolescent', '[13, 18)'::INT4RANGE),
    ('Adulte', '[18, 65)'::INT4RANGE),
    ('Senior', '[65, )'::INT4RANGE);

-- Trouver la catégorie pour un âge donné
SELECT nom FROM categories_age  
WHERE plage_age @> 30;  
-- Résultat : Adulte
```

### Créer un Type de Plage Personnalisé

```sql
-- Créer un type de plage pour un type personnalisé
CREATE TYPE prix_range AS RANGE (
    subtype = NUMERIC,
    subtype_diff = float8mi  -- Fonction de différence
);

-- Utilisation
CREATE TABLE segments_prix (
    id SERIAL PRIMARY KEY,
    categorie VARCHAR(50),
    plage_prix prix_range
);

INSERT INTO segments_prix (categorie, plage_prix) VALUES
    ('Budget', '[0, 50)'::prix_range),
    ('Milieu', '[50, 200)'::prix_range),
    ('Premium', '[200, )'::prix_range);
```

---

## Bonnes Pratiques

### 1. Domaines : Centraliser les Contraintes

```sql
-- ✅ BON : Un domaine réutilisable
CREATE DOMAIN email AS VARCHAR(255)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

-- Utiliser partout
CREATE TABLE t1 (email email);  
CREATE TABLE t2 (contact_email email);  

-- ❌ MAUVAIS : Répéter les contraintes
CREATE TABLE t1 (email VARCHAR(255) CHECK (...));  
CREATE TABLE t2 (email VARCHAR(255) CHECK (...));  
```

### 2. Types Composites : Pour Données Cohésives

```sql
-- ✅ BON : Grouper les données liées
CREATE TYPE adresse AS (
    rue TEXT,
    ville TEXT,
    code_postal TEXT
);

-- ❌ MOINS BON : Colonnes séparées répétées
CREATE TABLE personnes (
    adresse1_rue TEXT,
    adresse1_ville TEXT,
    adresse1_cp TEXT,
    adresse2_rue TEXT,
    adresse2_ville TEXT,
    adresse2_cp TEXT
);
```

### 3. ENUM : Pour Ensembles Stables

```sql
-- ✅ BON : Valeurs stables
CREATE TYPE jour_semaine AS ENUM ('Lundi', 'Mardi', ...);

-- ❌ MAUVAIS : Valeurs changeantes
-- CREATE TYPE pays AS ENUM ('France', 'Allemagne', ...);
-- Mieux : table de référence
```

### 4. Range : Pour Intervalles et Périodes

```sql
-- ✅ BON : Utiliser RANGE pour périodes
CREATE TABLE reservations (
    periode TSTZRANGE,
    EXCLUDE USING GIST (periode WITH &&)
);

-- ❌ MOINS BON : Deux colonnes séparées
CREATE TABLE reservations (
    date_debut TIMESTAMPTZ,
    date_fin TIMESTAMPTZ
    -- Difficile d'empêcher les chevauchements
);
```

### 5. Documenter les Types Personnalisés

```sql
-- ✅ BON : Commenter les types
CREATE DOMAIN email AS VARCHAR(255) CHECK (...);  
COMMENT ON DOMAIN email IS 'Adresse email validée selon RFC 5322';  

CREATE TYPE adresse AS (...);  
COMMENT ON TYPE adresse IS 'Adresse postale complète';  
```

---

## Inspection des Types Personnalisés

### Lister les Domaines

```sql
-- Dans psql
\dD

-- Avec SQL
SELECT
    n.nspname AS schema,
    t.typname AS domain_name,
    pg_catalog.format_type(t.typbasetype, t.typtypmod) AS base_type
FROM pg_catalog.pg_type t  
JOIN pg_catalog.pg_namespace n ON n.oid = t.typnamespace  
WHERE t.typtype = 'd'  
ORDER BY domain_name;  
```

### Lister les Types Composites

```sql
-- Dans psql
\dT

-- Avec SQL
SELECT
    n.nspname AS schema,
    t.typname AS type_name
FROM pg_catalog.pg_type t  
JOIN pg_catalog.pg_namespace n ON n.oid = t.typnamespace  
WHERE t.typtype = 'c'  
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY type_name;
```

### Voir la Définition d'un Type Composite

```sql
-- Dans psql
\d+ nom_type

-- Avec SQL
SELECT
    a.attname AS field_name,
    pg_catalog.format_type(a.atttypid, a.atttypmod) AS field_type
FROM pg_catalog.pg_type t  
JOIN pg_catalog.pg_attribute a ON a.attrelid = t.typrelid  
WHERE t.typname = 'nom_type'  
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY a.attnum;
```

### Lister les ENUM

```sql
-- Valeurs d'un ENUM
SELECT
    e.enumlabel
FROM pg_enum e  
JOIN pg_type t ON e.enumtypid = t.oid  
WHERE t.typname = 'nom_enum'  
ORDER BY e.enumsortorder;  
```

---

## Exemples Complets

### Système de Gestion d'Événements

```sql
-- Domaines
CREATE DOMAIN email AS VARCHAR(255)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE DOMAIN prix_positif AS NUMERIC(10, 2)
    CHECK (VALUE >= 0);

-- Types composites
CREATE TYPE adresse AS (
    rue VARCHAR(200),
    ville VARCHAR(100),
    code_postal VARCHAR(10),
    pays VARCHAR(50)
);

CREATE TYPE contact_info AS (
    telephone VARCHAR(20),
    email email,
    site_web VARCHAR(500)
);

-- ENUM
CREATE TYPE statut_evenement AS ENUM (
    'planifie',
    'confirme',
    'en_cours',
    'termine',
    'annule'
);

-- Tables utilisant les types personnalisés
CREATE TABLE lieux (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    adresse adresse,
    contact contact_info,
    capacite INTEGER
);

CREATE TABLE evenements (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300),
    lieu_id INTEGER REFERENCES lieux(id),
    periode TSTZRANGE NOT NULL,
    prix_entree prix_positif,
    statut statut_evenement DEFAULT 'planifie',

    EXCLUDE USING GIST (lieu_id WITH =, periode WITH &&)
);

-- Insertions
INSERT INTO lieux (nom, adresse, contact, capacite) VALUES (
    'Grande Salle',
    ROW('10 avenue des Champs', 'Paris', '75008', 'France')::adresse,
    ROW('0123456789', 'contact@salle.fr', 'https://salle.fr')::contact_info,
    500
);

INSERT INTO evenements (titre, lieu_id, periode, prix_entree) VALUES (
    'Concert de Jazz',
    1,
    '[2025-12-20 20:00, 2025-12-20 23:00)'::TSTZRANGE,
    45.00
);
```

---

## Récapitulatif

### Types Personnalisés Disponibles

| Type | Objectif | Exemple |
|------|----------|---------|
| **DOMAIN** | Type basé sur existant + contraintes | `email`, `code_postal_fr` |
| **COMPOSITE** | Plusieurs champs structurés | `adresse`, `coordonnees` |
| **ENUM** | Ensemble fixe de valeurs | `statut_commande` |
| **RANGE** | Plages/intervalles de valeurs | `TSTZRANGE`, `NUMRANGE` |

### Commandes Essentielles

```sql
-- Créer un domaine
CREATE DOMAIN email AS VARCHAR(255) CHECK (...);

-- Créer un type composite
CREATE TYPE adresse AS (rue TEXT, ville TEXT);

-- Créer un ENUM
CREATE TYPE statut AS ENUM ('actif', 'inactif');

-- Modifier
ALTER DOMAIN email ADD CONSTRAINT ...;  
ALTER TYPE adresse ADD ATTRIBUTE complement TEXT;  
ALTER TYPE statut ADD VALUE 'suspendu';  

-- Supprimer
DROP DOMAIN email;  
DROP TYPE adresse;  
DROP TYPE statut;  
```

### Quand Utiliser Quoi ?

| Besoin | Solution |
|--------|----------|
| Validation réutilisable | **DOMAIN** |
| Grouper des champs liés | **COMPOSITE TYPE** |
| Ensemble de valeurs fixes | **ENUM** |
| Intervalles, périodes | **RANGE TYPE** |
| Validation simple | CHECK constraint |

---

## Conclusion

Les types personnalisés de PostgreSQL sont des outils puissants pour :

- ✅ **Réutiliser** des définitions et contraintes  
- ✅ **Standardiser** votre schéma  
- ✅ **Documenter** les intentions métier  
- ✅ **Simplifier** la maintenance

**Recommandations :**
- Utilisez les **domaines** pour les validations répétitives (email, téléphone, etc.)
- Utilisez les **types composites** pour regrouper des données cohésives
- Utilisez les **ENUM** pour des ensembles stables de valeurs
- Utilisez les **RANGE** pour gérer des intervalles et empêcher les chevauchements

Dans la prochaine section, nous explorerons la gestion des modifications de schéma avec ALTER et DROP, et comment gérer le verrouillage lors des modifications de structure.

---


⏭️ [Gestion des modifications de schéma (ALTER, DROP) et verrouillage](/04-objets-de-la-base-de-donnees/07-modifications-de-schema.md)
