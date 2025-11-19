🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4. Les Types de Données Fondamentaux

## Introduction

Maintenant que vous savez créer des tables dans PostgreSQL, il est temps de comprendre en profondeur les **types de données** que vous pouvez utiliser pour définir vos colonnes. Le choix du bon type de données est crucial car il affecte :

- ✅ **La validité des données** : Certains types empêchent l'insertion de données invalides
- ✅ **Les performances** : Certains types sont plus efficaces en termes de stockage et de vitesse
- ✅ **L'intégrité** : Le bon type garantit que vos données restent cohérentes
- ✅ **La maintenabilité** : Un schéma bien typé est plus facile à comprendre et à maintenir

---

## Qu'est-ce qu'un Type de Données ?

Un **type de données** définit :
1. **Le genre de valeurs** qu'une colonne peut contenir (nombres, texte, dates, etc.)
2. **L'espace de stockage** nécessaire
3. **Les opérations** possibles sur ces valeurs

### Analogie

Pensez aux types de données comme à des conteneurs spécialisés :

```
📦 Type INTEGER    → Boîte pour nombres entiers    → 42, -15, 1000
📦 Type TEXT       → Boîte pour du texte          → "Bonjour", "PostgreSQL"
📦 Type DATE       → Boîte pour des dates         → 2025-11-19
📦 Type BOOLEAN    → Boîte pour vrai/faux         → true, false
```

Chaque "boîte" (type) accepte seulement certaines valeurs et rejette les autres.

---

## Pourquoi les Types de Données sont Importants ?

### 1. Validation Automatique

```sql
-- Créer une table avec types spécifiques
CREATE TABLE personnes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    age INTEGER,
    date_naissance DATE,
    salaire NUMERIC(10, 2)
);

-- ✅ Insertion valide
INSERT INTO personnes (nom, age, date_naissance, salaire)
VALUES ('Alice Martin', 30, '1995-06-15', 45000.50);

-- ❌ Insertion invalide : âge doit être un entier
INSERT INTO personnes (nom, age)
VALUES ('Bob', 'trente');  -- ERREUR : invalid input syntax for type integer

-- ❌ Insertion invalide : date mal formatée
INSERT INTO personnes (nom, date_naissance)
VALUES ('Charlie', '15/06/1995');  -- ERREUR selon configuration datestyle

-- ❌ Insertion invalide : salaire trop précis
INSERT INTO personnes (nom, salaire)
VALUES ('Dave', 45000.123);  -- Arrondi automatiquement à 45000.12
```

**Avantage :** PostgreSQL vous empêche d'insérer des données incorrectes !

### 2. Optimisation du Stockage

Différents types occupent différentes quantités d'espace :

```sql
-- Comparaison de taille de stockage
CREATE TABLE comparaison_tailles (
    petit SMALLINT,      -- 2 octets
    moyen INTEGER,       -- 4 octets
    grand BIGINT,        -- 8 octets
    texte_court VARCHAR(10),  -- Variable (2-11 octets)
    texte_long TEXT,     -- Variable
    nombre_decimal NUMERIC(10,2)  -- Variable
);

-- Exemple : Stocker l'âge (0-120)
-- SMALLINT suffit (2 octets) au lieu de INTEGER (4 octets)
-- Économie : 50% d'espace pour cette colonne

-- Pour 1 million de lignes :
-- SMALLINT : 2 MB
-- INTEGER : 4 MB
-- Économie : 2 MB sur cette seule colonne
```

**Avantage :** Un bon choix de type économise de l'espace disque et améliore les performances !

### 3. Opérations Adaptées

Chaque type supporte des opérations spécifiques :

```sql
-- Opérations sur nombres
SELECT 100 + 50;           -- 150
SELECT 100 / 3;            -- 33 (division entière)
SELECT 100.0 / 3;          -- 33.333...

-- Opérations sur texte
SELECT 'Hello' || ' ' || 'World';  -- 'Hello World'
SELECT UPPER('bonjour');           -- 'BONJOUR'
SELECT LENGTH('PostgreSQL');       -- 10

-- Opérations sur dates
SELECT CURRENT_DATE + 7;              -- Date dans 7 jours
SELECT AGE(DATE '2025-11-19', DATE '1995-06-15');  -- 30 years 5 mons 4 days

-- Opérations sur JSON
SELECT '{"nom": "Alice"}'::JSONB ->> 'nom';  -- 'Alice'

-- ❌ Impossible : mélanger les types incorrectement
SELECT '100' + 50;  -- ERREUR : operator does not exist: text + integer
```

**Avantage :** Les types garantissent que vous utilisez les bonnes opérations !

### 4. Lisibilité et Documentation

Un schéma bien typé documente automatiquement votre structure de données :

```sql
-- ✅ BON : Types explicites et appropriés
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    numero_commande VARCHAR(50) UNIQUE NOT NULL,
    client_id INTEGER NOT NULL,
    date_commande TIMESTAMPTZ DEFAULT NOW(),
    montant_total NUMERIC(12, 2) NOT NULL,
    statut VARCHAR(20) DEFAULT 'en_attente'
        CHECK (statut IN ('en_attente', 'confirmee', 'expediee', 'livree')),
    notes TEXT
);

-- En lisant ce schéma, on comprend immédiatement :
-- - id : identifiant auto-incrémenté
-- - numero_commande : code unique (max 50 caractères)
-- - date_commande : moment précis avec fuseau horaire
-- - montant_total : montant financier précis (max 10 chiffres, 2 décimales)
-- - statut : valeur parmi un ensemble limité
-- - notes : texte libre de longueur variable

-- ❌ MAUVAIS : Types vagues
CREATE TABLE commandes_mauvais (
    id INTEGER,
    numero TEXT,
    client TEXT,
    date TEXT,
    montant TEXT,
    statut TEXT,
    notes TEXT
);

-- Impossible de savoir :
-- - Quelles valeurs sont acceptées ?
-- - Quels formats sont attendus ?
-- - Quelles contraintes existent ?
```

---

## Vue d'Ensemble des Catégories de Types

PostgreSQL offre une riche variété de types de données. Voici les principales catégories :

### 1. Types Numériques 🔢

Pour stocker des nombres (entiers, décimaux, flottants).

| Type | Description | Exemple |
|------|-------------|---------|
| `SMALLINT` | Petit entier | -32768 à 32767 |
| `INTEGER` | Entier standard | -2 milliards à +2 milliards |
| `BIGINT` | Grand entier | Très grandes valeurs |
| `SERIAL` | Entier auto-incrémenté | 1, 2, 3, ... |
| `NUMERIC` / `DECIMAL` | Nombre décimal précis | 19.99, 1234.56 |
| `REAL` / `DOUBLE PRECISION` | Nombre flottant | Calculs scientifiques |

**Quand utiliser :**
- Compteurs, quantités : `INTEGER`
- Identifiants auto-incrémentés : `SERIAL`
- **Argent et finances** : `NUMERIC` (jamais FLOAT !)
- Mesures physiques : `REAL` ou `DOUBLE PRECISION`

### 2. Types Textuels 📝

Pour stocker du texte de différentes longueurs.

| Type | Description | Exemple |
|------|-------------|---------|
| `VARCHAR(n)` | Texte variable (max n caractères) | Noms, emails |
| `TEXT` | Texte illimité | Articles, commentaires |
| `CHAR(n)` | Texte fixe (exactement n caractères) | Codes (rarement utilisé) |

**Quand utiliser :**
- Noms, emails, URLs : `VARCHAR(n)` avec limite appropriée
- Contenu long (articles, descriptions) : `TEXT`
- Codes de longueur fixe : `CHAR(n)` (rare)

### 3. Types Temporels ⏰

Pour stocker des dates, heures et durées.

| Type | Description | Exemple |
|------|-------------|---------|
| `DATE` | Date sans heure | 2025-11-19 |
| `TIME` | Heure sans date | 14:30:00 |
| `TIMESTAMP` | Date et heure | 2025-11-19 14:30:00 |
| `TIMESTAMPTZ` | Date/heure avec fuseau horaire | **Recommandé** |
| `INTERVAL` | Durée/intervalle | 3 days, 2 hours |

**Quand utiliser :**
- Dates de naissance, échéances : `DATE`
- **Événements, logs** : `TIMESTAMPTZ` (toujours avec timezone !)
- Durées : `INTERVAL`

### 4. Types Booléens ✓

Pour stocker des valeurs vrai/faux.

| Type | Valeurs | Exemple |
|------|---------|---------|
| `BOOLEAN` | TRUE, FALSE, NULL | est_actif, est_publie |

**Quand utiliser :**
- Flags, états binaires : `BOOLEAN`
- Ne jamais utiliser VARCHAR('Y'/'N') ou INTEGER (0/1) !

### 5. Types Spécifiques PostgreSQL ⭐

Types avancés uniques à PostgreSQL.

| Type | Description | Cas d'Usage |
|------|-------------|-------------|
| `JSONB` | Documents JSON | Données flexibles, APIs |
| `ARRAY` | Tableaux | Tags, listes |
| `UUID` | Identifiants uniques | IDs distribués |
| `ENUM` | Valeurs énumérées | Statuts, catégories |
| `INET` / `CIDR` | Adresses IP | Réseaux |

**Quand utiliser :**
- Données semi-structurées : `JSONB`
- Listes simples : `ARRAY`
- IDs globaux, API publiques : `UUID`
- Ensembles de valeurs fixes : `ENUM`

### 6. Types Géométriques et Spécialisés 🗺️

Types pour cas d'usage spécifiques.

| Type | Description |
|------|-------------|
| `POINT`, `LINE`, `POLYGON` | Géométrie 2D |
| `hstore` | Paires clé-valeur (legacy) |
| `ltree` | Hiérarchies/arbres |
| `BYTEA` | Données binaires |
| `XML` | Documents XML |

**Note :** Ces types sont moins courants mais très utiles dans des contextes spécifiques.

---

## Tableau Récapitulatif : Choisir le Bon Type

Voici un guide rapide pour choisir le bon type selon vos besoins :

| Je veux stocker... | Type Recommandé | Exemple |
|-------------------|-----------------|---------|
| Un identifiant unique auto-incrémenté | `SERIAL` ou `UUID` | `id SERIAL PRIMARY KEY` |
| Un âge | `SMALLINT` | `age SMALLINT` |
| Une quantité, un compteur | `INTEGER` | `quantite INTEGER` |
| Un **prix, un montant d'argent** | `NUMERIC(10, 2)` | `prix NUMERIC(10, 2)` |
| Un pourcentage | `NUMERIC(5, 2)` | `taux NUMERIC(5, 2)` |
| Une mesure physique approximative | `REAL` | `temperature REAL` |
| Un nom, prénom | `VARCHAR(100)` | `nom VARCHAR(100)` |
| Un email | `VARCHAR(255)` | `email VARCHAR(255)` |
| Une URL | `VARCHAR(500)` | `url VARCHAR(500)` |
| Un article, commentaire long | `TEXT` | `contenu TEXT` |
| Une date de naissance | `DATE` | `date_naissance DATE` |
| Un moment précis avec heure | `TIMESTAMPTZ` | `created_at TIMESTAMPTZ` |
| Une durée | `INTERVAL` | `duree INTERVAL` |
| Un flag vrai/faux | `BOOLEAN` | `est_actif BOOLEAN` |
| Des données flexibles (API) | `JSONB` | `metadata JSONB` |
| Une liste de tags | `TEXT[]` | `tags TEXT[]` |
| Un identifiant global unique | `UUID` | `id UUID` |
| Un statut parmi des valeurs fixes | `ENUM` ou `VARCHAR` + CHECK | `statut statut_enum` |

---

## Principes de Base pour Choisir un Type

### Principe 1 : Choisir le Type le Plus Restrictif

```sql
-- ✅ BON : Type restrictif
CREATE TABLE produits (
    age_minimum SMALLINT CHECK (age_minimum >= 0 AND age_minimum <= 18)
);

-- ❌ MOINS BON : Type trop large
CREATE TABLE produits_large (
    age_minimum BIGINT  -- Peut stocker des milliards, pas nécessaire pour un âge
);
```

**Pourquoi ?** Un type restrictif :
- Empêche les erreurs
- Économise de l'espace
- Rend les intentions claires

### Principe 2 : Préférer les Types Natifs

```sql
-- ✅ BON : Type natif approprié
CREATE TABLE utilisateurs (
    date_inscription TIMESTAMPTZ DEFAULT NOW(),
    est_actif BOOLEAN DEFAULT TRUE
);

-- ❌ MAUVAIS : Simuler des types avec TEXT
CREATE TABLE utilisateurs_mauvais (
    date_inscription TEXT,  -- '2025-11-19' en texte ?
    est_actif TEXT          -- 'Y' ou 'N' en texte ?
);
```

**Pourquoi ?** Les types natifs :
- Sont validés automatiquement
- Supportent des opérations spécifiques
- Sont plus performants

### Principe 3 : Penser à l'Évolution

```sql
-- ✅ BON : Prévoir la croissance
CREATE TABLE logs (
    id BIGSERIAL PRIMARY KEY,  -- Grande table potentielle
    timestamp TIMESTAMPTZ,
    message TEXT
);

-- ❌ RISQUÉ : Type trop petit
CREATE TABLE logs_risque (
    id SERIAL PRIMARY KEY,  -- Limite à 2 milliards de lignes
    message VARCHAR(100)    -- Trop court pour certains messages
);
```

**Conseil :** Anticipez la croissance de vos données.

### Principe 4 : Documenter avec les Types

```sql
-- ✅ BON : Types auto-documentants
CREATE TABLE commandes (
    montant_ht NUMERIC(10, 2) NOT NULL,  -- Clair : montant financier
    montant_tva NUMERIC(10, 2) NOT NULL,
    montant_ttc NUMERIC(10, 2) NOT NULL,
    date_commande TIMESTAMPTZ DEFAULT NOW(),
    date_livraison_estimee DATE
);

-- Le schéma raconte l'histoire des données !
```

---

## Erreurs Courantes à Éviter

### ❌ Erreur 1 : Utiliser VARCHAR pour Tout

```sql
-- ❌ MAUVAIS
CREATE TABLE mauvaise_table (
    age VARCHAR(10),
    prix VARCHAR(20),
    date VARCHAR(50),
    est_actif VARCHAR(5)
);

-- ✅ BON
CREATE TABLE bonne_table (
    age INTEGER,
    prix NUMERIC(10, 2),
    date DATE,
    est_actif BOOLEAN
);
```

### ❌ Erreur 2 : Utiliser FLOAT pour l'Argent

```sql
-- ❌ CATASTROPHIQUE : FLOAT pour l'argent
CREATE TABLE mauvais_prix (
    prix FLOAT
);

INSERT INTO mauvais_prix VALUES (0.1 + 0.2);
SELECT * FROM mauvais_prix;
-- Résultat : 0.30000000000000004 (erreur d'arrondi !)

-- ✅ CORRECT : NUMERIC pour l'argent
CREATE TABLE bon_prix (
    prix NUMERIC(10, 2)
);

INSERT INTO bon_prix VALUES (0.1 + 0.2);
SELECT * FROM bon_prix;
-- Résultat : 0.30 (exact !)
```

**Règle d'or :** JAMAIS FLOAT/REAL pour l'argent !

### ❌ Erreur 3 : Types Trop Larges

```sql
-- ❌ GASPILLAGE : Type trop grand
CREATE TABLE personnes (
    age BIGINT  -- Peut stocker jusqu'à 9 × 10^18 !
);

-- ✅ APPROPRIÉ
CREATE TABLE personnes_correct (
    age SMALLINT CHECK (age >= 0 AND age <= 150)
);
```

### ❌ Erreur 4 : Ignorer les Fuseaux Horaires

```sql
-- ❌ PROBLÉMATIQUE : Sans timezone
CREATE TABLE events (
    event_time TIMESTAMP
);

-- ✅ RECOMMANDÉ : Avec timezone
CREATE TABLE events_correct (
    event_time TIMESTAMPTZ
);
```

---

## Comment Lire la Suite de ce Chapitre

Ce chapitre 4.4 est organisé en sous-sections détaillées, chacune explorant une catégorie de types :

1. **4.4.1. Types Numériques** → Nombres entiers, décimaux, flottants
2. **4.4.2. Types Texte** → VARCHAR, TEXT, CHAR
3. **4.4.3. Types Temporels** → DATE, TIMESTAMP, INTERVAL
4. **4.4.4. Types Spécifiques PostgreSQL** → JSONB, ARRAY, UUID, ENUM
5. **4.4.5. Types Géométriques, hstore et ltree** → Cas spécialisés
6. **4.4.6. Types Binaires et XML** → BYTEA, XML

**Conseil de lecture :**
- Lisez d'abord **4.4.1**, **4.4.2** et **4.4.3** (fondamentaux, utilisés dans 90% des projets)
- Explorez **4.4.4** (types modernes très utiles)
- Consultez **4.4.5** et **4.4.6** selon vos besoins spécifiques

---

## Exemple Complet : Schéma Bien Typé

Voici un exemple de table utilisant divers types de manière appropriée :

```sql
CREATE TABLE utilisateurs (
    -- Identifiant
    id SERIAL PRIMARY KEY,
    uuid UUID DEFAULT gen_random_uuid() UNIQUE,

    -- Informations personnelles
    prenom VARCHAR(100) NOT NULL,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL CHECK (email LIKE '%@%'),
    date_naissance DATE CHECK (date_naissance > '1900-01-01'),
    age INTEGER GENERATED ALWAYS AS (
        EXTRACT(YEAR FROM AGE(CURRENT_DATE, date_naissance))
    ) STORED,

    -- Authentification
    password_hash BYTEA NOT NULL,

    -- État et préférences
    est_actif BOOLEAN DEFAULT TRUE,
    est_verifie BOOLEAN DEFAULT FALSE,
    preferences JSONB DEFAULT '{}'::JSONB,
    roles TEXT[] DEFAULT ARRAY['user']::TEXT[],

    -- Métadonnées
    date_inscription TIMESTAMPTZ DEFAULT NOW(),
    derniere_connexion TIMESTAMPTZ,
    ip_derniere_connexion INET,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Ce schéma utilise :
-- - SERIAL pour l'ID auto-incrémenté
-- - UUID pour un identifiant global
-- - VARCHAR avec limites pour les textes courts
-- - DATE pour la date de naissance
-- - INTEGER généré pour l'âge (calculé automatiquement)
-- - BYTEA pour le hash du mot de passe
-- - BOOLEAN pour les flags
-- - JSONB pour des préférences flexibles
-- - ARRAY pour une liste de rôles
-- - TIMESTAMPTZ pour les horodatages (avec timezone !)
-- - INET pour les adresses IP
```

---

## Récapitulatif

### Points Clés à Retenir

1. **Le type de données définit** ce qu'une colonne peut contenir
2. **Choisir le bon type** améliore validité, performance et maintenabilité
3. **Types principaux** : Numériques, Texte, Temporels, Booléens, Spécialisés
4. **Règle d'or** : Type le plus restrictif approprié
5. **Pour l'argent** : TOUJOURS NUMERIC (jamais FLOAT)
6. **Pour les horodatages** : TIMESTAMPTZ (avec timezone)

### Commandes pour Explorer les Types

```sql
-- Voir tous les types disponibles
SELECT typname, typlen, typcategory
FROM pg_type
WHERE typtype = 'b'  -- Types de base
ORDER BY typname;

-- Type d'une valeur
SELECT pg_typeof(123);           -- integer
SELECT pg_typeof('texte');       -- unknown (devient text)
SELECT pg_typeof(123.45);        -- numeric
SELECT pg_typeof(NOW());         -- timestamp with time zone

-- Voir la structure d'une table
\d nom_table

-- Taille de stockage
SELECT pg_column_size(colonne) FROM table;
```

---

## Prochaines Étapes

Maintenant que vous comprenez l'importance des types de données et avez une vue d'ensemble, plongeons dans les détails !

La prochaine section **4.4.1. Types Numériques** explore en profondeur :
- Les différents types d'entiers (SMALLINT, INTEGER, BIGINT)
- Les identifiants auto-incrémentés (SERIAL, IDENTITY)
- Les nombres décimaux précis (NUMERIC, DECIMAL)
- Les nombres à virgule flottante (REAL, DOUBLE PRECISION)

**Prêt à devenir un expert des types de données PostgreSQL ? C'est parti !** 🚀

---


⏭️ [Numériques (INTEGER, SERIAL/IDENTITY, NUMERIC, FLOAT)](/04-objets-de-la-base-de-donnees/04.1-types-numeriques.md)
