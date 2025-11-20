🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.3. Héritage de tables (Table Inheritance) : Concept et limites

## Introduction

L'**héritage de tables** est une fonctionnalité unique de PostgreSQL qui s'inspire de la programmation orientée objet. Elle permet de créer des hiérarchies de tables où une table "enfant" hérite de la structure et des données d'une table "parent". Cette caractéristique fait partie de l'approche "objet-relationnelle" de PostgreSQL, d'où son nom complet : **Système de Gestion de Base de Données Objet-Relationnel (SGBDOR)**.

### Analogie avec la Programmation Orientée Objet

Si vous connaissez la programmation orientée objet (POO), l'héritage de tables fonctionne de manière similaire :

```
En POO (Java/Python) :
┌─────────────────┐
│   Animal        │  (classe parente)
│   - nom         │
│   - age         │
└─────────────────┘
         ▲
         │ hérite de
    ┌────┴─────┐
    │          │
┌───┴───┐   ┌──┴────┐
│ Chien │   │ Chat  │  (classes enfants)
│ - race│   │- poils│
└───────┘   └───────┘

En PostgreSQL :
┌─────────────────┐
│   animaux       │  (table parente)
│   - nom         │
│   - age         │
└─────────────────┘
         ▲
         │ INHERITS
    ┌────┴─────┐
    │          │
┌───┴───┐   ┌──┴────┐
│ chiens│   │ chats │  (tables enfants)
│ - race│   │- poils│
└───────┘   └───────┘
```

### Qu'est-ce que l'Héritage de Tables ?

L'héritage de tables permet à une table enfant de :
- **Hériter automatiquement** de toutes les colonnes de la table parent
- **Ajouter ses propres colonnes** spécifiques
- **Être interrogée via la table parent** (requêtes automatiquement étendues aux enfants)

**Note importante :** Cette fonctionnalité est **spécifique à PostgreSQL** et n'existe pas dans les autres bases de données relationnelles majeures (MySQL, Oracle, SQL Server).

---

## Syntaxe de Base

### Créer une Hiérarchie Simple

```sql
-- Table parent
CREATE TABLE animaux (
    animal_id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    age INTEGER,
    date_arrivee DATE DEFAULT CURRENT_DATE
);

-- Table enfant : hérite de animaux
CREATE TABLE chiens (
    race VARCHAR(50),
    dresse BOOLEAN DEFAULT FALSE
) INHERITS (animaux);

-- Autre table enfant
CREATE TABLE chats (
    type_poil VARCHAR(20),
    declaws BOOLEAN DEFAULT FALSE
) INHERITS (animaux);
```

### Structure Résultante

**Table `animaux` (parent) :**
```
animal_id | nom | age | date_arrivee
```

**Table `chiens` (enfant) :**
```
animal_id | nom | age | date_arrivee | race | dresse
└──────────── hérité de animaux ─────────┘  └─ spécifique ─┘
```

**Table `chats` (enfant) :**
```
animal_id | nom | age | date_arrivee | type_poil | declaws
└──────────── hérité de animaux ─────────┘  └── spécifique ──┘
```

---

## Insertion de Données

### Insérer dans les Tables Enfants

```sql
-- Insérer un chien
INSERT INTO chiens (nom, age, race, dresse)
VALUES ('Rex', 3, 'Berger Allemand', TRUE);

-- Insérer un autre chien
INSERT INTO chiens (nom, age, race, dresse)
VALUES ('Max', 1, 'Golden Retriever', FALSE);

-- Insérer un chat
INSERT INTO chats (nom, age, type_poil, declaws)
VALUES ('Minou', 2, 'long', FALSE);

-- Insérer un autre chat
INSERT INTO chats (nom, age, type_poil, declaws)
VALUES ('Felix', 4, 'court', TRUE);
```

**Important :** Vous insérez directement dans les tables enfants. Les données **ne sont pas physiquement stockées** dans la table parent.

### Peut-on Insérer dans la Table Parent ?

Oui, mais ce n'est généralement **pas recommandé** dans ce modèle :

```sql
-- Possible mais inhabituel dans ce contexte
INSERT INTO animaux (nom, age)
VALUES ('Animal Générique', 5);
```

Cette ligne sera dans `animaux` mais pas dans `chiens` ou `chats`. Cela crée un "animal" sans type spécifique.

---

## Interrogation des Données

### La Magie de l'Héritage : Requêtes Automatiques

L'un des avantages majeurs de l'héritage est que les requêtes sur la table parent **incluent automatiquement** les données des tables enfants.

```sql
-- Requête sur la table parent
SELECT * FROM animaux;
```

**Résultat :**
```
animal_id | nom    | age | date_arrivee | race              | dresse | type_poil | declaws
----------+--------+-----+--------------+-------------------+--------+-----------+---------
1         | Rex    | 3   | 2025-11-20   | Berger Allemand   | t      | NULL      | NULL
2         | Max    | 1   | 2025-11-20   | Golden Retriever  | f      | NULL      | NULL
3         | Minou  | 2   | 2025-11-20   | NULL              | NULL   | long      | f
4         | Felix  | 4   | 2025-11-20   | NULL              | NULL   | court     | t
5         | Animal | 5   | 2025-11-20   | NULL              | NULL   | NULL      | NULL
```

**Observation :**
- Les lignes de `chiens` apparaissent avec leurs colonnes `race` et `dresse`
- Les lignes de `chats` apparaissent avec leurs colonnes `type_poil` et `declaws`
- Les colonnes non applicables sont NULL

### Requêtes sur les Tables Enfants

```sql
-- Uniquement les chiens
SELECT * FROM chiens;
```

**Résultat :**
```
animal_id | nom | age | date_arrivee | race              | dresse
----------+-----+-----+--------------+-------------------+--------
1         | Rex | 3   | 2025-11-20   | Berger Allemand   | t
2         | Max | 1   | 2025-11-20   | Golden Retriever  | f
```

```sql
-- Uniquement les chats
SELECT * FROM chats;
```

**Résultat :**
```
animal_id | nom    | age | date_arrivee | type_poil | declaws
----------+--------+-----+--------------+-----------+---------
3         | Minou  | 2   | 2025-11-20   | long      | f
4         | Felix  | 4   | 2025-11-20   | court     | t
```

### Exclure les Tables Enfants : ONLY

Pour interroger **uniquement** la table parent sans les enfants :

```sql
SELECT * FROM ONLY animaux;
```

**Résultat :**
```
animal_id | nom    | age | date_arrivee
----------+--------+-----+--------------
5         | Animal | 5   | 2025-11-20
```

Seules les lignes insérées directement dans `animaux` apparaissent.

### Identifier la Table d'Origine : tableoid

PostgreSQL fournit une colonne système spéciale `tableoid` pour identifier de quelle table provient chaque ligne :

```sql
SELECT
    tableoid::regclass AS table_source,
    nom,
    age
FROM animaux;
```

**Résultat :**
```
table_source | nom    | age
-------------+--------+-----
chiens       | Rex    | 3
chiens       | Max    | 1
chats        | Minou  | 2
chats        | Felix  | 4
animaux      | Animal | 5
```

Ceci est très utile pour distinguer les types d'entités dans une requête unifiée.

---

## Cas d'Usage Typiques

### 1. Hiérarchie de Types d'Entités

**Exemple : Système de Gestion de Contenus**

```sql
-- Table parent : tous les contenus
CREATE TABLE contenus (
    contenu_id SERIAL PRIMARY KEY,
    titre VARCHAR(255) NOT NULL,
    auteur_id INTEGER,
    date_creation TIMESTAMPTZ DEFAULT NOW(),
    statut VARCHAR(20) DEFAULT 'brouillon'
);

-- Articles de blog
CREATE TABLE articles (
    contenu TEXT,
    nb_mots INTEGER,
    temps_lecture_min INTEGER,
    categorie VARCHAR(100)
) INHERITS (contenus);

-- Pages statiques
CREATE TABLE pages (
    slug VARCHAR(255) UNIQUE,
    template VARCHAR(50),
    contenu_html TEXT
) INHERITS (contenus);

-- Vidéos
CREATE TABLE videos (
    url_video VARCHAR(500),
    duree_secondes INTEGER,
    miniature_url VARCHAR(500),
    plateforme VARCHAR(50) -- 'youtube', 'vimeo', etc.
) INHERITS (contenus);
```

**Avantages :**
```sql
-- Lister TOUS les contenus, peu importe le type
SELECT contenu_id, titre, date_creation, statut
FROM contenus
ORDER BY date_creation DESC
LIMIT 10;

-- Rechercher dans tous les contenus
SELECT tableoid::regclass as type, titre, auteur_id
FROM contenus
WHERE titre ILIKE '%PostgreSQL%';

-- Statistiques globales
SELECT
    tableoid::regclass AS type_contenu,
    COUNT(*) AS nombre,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS pourcentage
FROM contenus
GROUP BY tableoid::regclass;
```

**Résultat typique :**
```
type_contenu | nombre | pourcentage
-------------+--------+-------------
articles     | 150    | 75.00
pages        | 30     | 15.00
videos       | 20     | 10.00
```

### 2. Séparation Géographique ou Temporelle

**Exemple : Données de Capteurs par Région**

```sql
-- Table parent
CREATE TABLE mesures_capteurs (
    mesure_id SERIAL PRIMARY KEY,
    capteur_id INTEGER NOT NULL,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    valeur NUMERIC(10,2)
);

-- Mesures Europe
CREATE TABLE mesures_europe (
    CHECK (timestamp >= '2025-01-01' AND timestamp < '2026-01-01')
) INHERITS (mesures_capteurs);

-- Mesures Amérique
CREATE TABLE mesures_amerique (
    CHECK (timestamp >= '2025-01-01' AND timestamp < '2026-01-01')
) INHERITS (mesures_capteurs);

-- Mesures Asie
CREATE TABLE mesures_asie (
    CHECK (timestamp >= '2025-01-01' AND timestamp < '2026-01-01')
) INHERITS (mesures_capteurs);
```

**Note :** Ce cas d'usage est aujourd'hui **mieux résolu par le partitionnement déclaratif** (voir section "Alternatives").

### 3. Polymorphisme de Données

**Exemple : Système de Paiements**

```sql
-- Table parent
CREATE TABLE transactions (
    transaction_id SERIAL PRIMARY KEY,
    utilisateur_id INTEGER NOT NULL,
    montant NUMERIC(10,2) NOT NULL,
    date_transaction TIMESTAMPTZ DEFAULT NOW(),
    statut VARCHAR(20) DEFAULT 'en_attente'
);

-- Paiements par carte bancaire
CREATE TABLE paiements_carte (
    numero_carte_masque VARCHAR(20),
    type_carte VARCHAR(20), -- 'visa', 'mastercard', etc.
    date_expiration DATE,
    code_autorisation VARCHAR(50)
) INHERITS (transactions);

-- Paiements PayPal
CREATE TABLE paiements_paypal (
    email_paypal VARCHAR(255),
    transaction_paypal_id VARCHAR(100),
    statut_paypal VARCHAR(50)
) INHERITS (transactions);

-- Virements bancaires
CREATE TABLE paiements_virement (
    iban VARCHAR(34),
    bic VARCHAR(11),
    reference_virement VARCHAR(100)
) INHERITS (transactions);
```

**Utilisation :**
```sql
-- Historique complet de toutes les transactions d'un utilisateur
SELECT
    transaction_id,
    tableoid::regclass AS mode_paiement,
    montant,
    date_transaction,
    statut
FROM transactions
WHERE utilisateur_id = 1001
ORDER BY date_transaction DESC;
```

---

## Contraintes et Héritage

### Héritage des Contraintes

Les contraintes sont héritées différemment selon leur type :

```sql
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255) NOT NULL,
    prix NUMERIC(10,2) CHECK (prix > 0),
    stock INTEGER DEFAULT 0
);

CREATE TABLE produits_electronique (
    garantie_mois INTEGER CHECK (garantie_mois >= 12)
) INHERITS (produits);
```

**Comportement :**

| Type de Contrainte | Hérité ? | Notes |
|-------------------|----------|-------|
| **NOT NULL** | ✅ Oui | Hérité automatiquement |
| **CHECK** | ✅ Oui | Hérité et peut être étendu |
| **DEFAULT** | ✅ Oui | Hérité automatiquement |
| **PRIMARY KEY** | ❌ Non | **LIMITATION MAJEURE** |
| **FOREIGN KEY** | ❌ Non | **LIMITATION MAJEURE** |
| **UNIQUE** | ❌ Non | **LIMITATION MAJEURE** |

### Problème : Clés Primaires Non Héritées

```sql
-- La clé primaire dans la table parent...
CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255)
);

-- ...n'est PAS héritée par les enfants
CREATE TABLE employes_temps_plein (
    salaire_annuel NUMERIC(10,2)
) INHERITS (employes);

CREATE TABLE employes_temps_partiel (
    taux_horaire NUMERIC(6,2)
) INHERITS (employes);

-- PROBLÈME : Pas de PK sur les tables enfants !
-- Vous pouvez avoir des doublons d'employe_id entre tables
```

**Conséquence :** Vous devez créer manuellement les contraintes sur chaque table enfant.

```sql
-- Solution : Ajouter manuellement les contraintes
ALTER TABLE employes_temps_plein ADD PRIMARY KEY (employe_id);
ALTER TABLE employes_temps_partiel ADD PRIMARY KEY (employe_id);
```

### Problème : Foreign Keys et Héritage

Les clés étrangères ne fonctionnent **pas bien** avec l'héritage :

```sql
CREATE TABLE departements (
    departement_id SERIAL PRIMARY KEY,
    nom_departement VARCHAR(100)
);

CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement_id INTEGER REFERENCES departements(departement_id)
);

CREATE TABLE employes_temps_plein () INHERITS (employes);

-- PROBLÈME 1 : La FK n'est pas héritée
-- PROBLÈME 2 : On ne peut pas créer de FK vers la table parent "employes"
--              depuis une autre table car elle ne garantit pas l'unicité
--              à travers toute la hiérarchie
```

**Exemple du problème :**

```sql
CREATE TABLE projets (
    projet_id SERIAL PRIMARY KEY,
    nom_projet VARCHAR(255),
    chef_projet_id INTEGER REFERENCES employes(employe_id)  -- INCOMPLET
);

-- Cette FK référence seulement la table parent "employes"
-- Si un chef de projet est dans "employes_temps_plein",
-- la FK ne le trouvera pas !
```

---

## Modification de Structure (ALTER TABLE)

### Ajouter une Colonne à la Table Parent

```sql
ALTER TABLE animaux ADD COLUMN couleur VARCHAR(50);
```

**Effet :** La colonne est **automatiquement ajoutée** à toutes les tables enfants (`chiens`, `chats`).

```sql
-- Vérification
\d chiens
-- Affichera maintenant la colonne "couleur"
```

### Ajouter une Colonne à une Table Enfant

```sql
ALTER TABLE chiens ADD COLUMN puce_electronique VARCHAR(20);
```

**Effet :** La colonne est ajoutée **uniquement** à `chiens`, pas à `animaux` ni aux autres enfants.

### Supprimer une Colonne

```sql
-- Supprimer de la table parent
ALTER TABLE animaux DROP COLUMN couleur;

-- Par défaut, PostgreSQL supprime aussi la colonne des enfants
-- Pour éviter cela (comportement rare) :
ALTER TABLE animaux DROP COLUMN couleur RESTRICT;  -- Échoue si utilisée dans enfants
```

---

## Index et Héritage

### Problème : Les Index Ne Sont Pas Hérités

```sql
-- Créer un index sur la table parent
CREATE INDEX idx_animaux_nom ON animaux(nom);

-- Cet index existe SEULEMENT sur la table "animaux"
-- Il n'existe PAS automatiquement sur "chiens" ou "chats"
```

**Conséquence :** Une requête sur la table parent qui scanne les enfants n'utilisera pas l'index parent.

```sql
-- Cette requête ne peut pas utiliser l'index sur "nom" pour les chiens et chats
SELECT * FROM animaux WHERE nom = 'Rex';
```

**Solution :** Créer les index manuellement sur chaque table enfant.

```sql
CREATE INDEX idx_chiens_nom ON chiens(nom);
CREATE INDEX idx_chats_nom ON chats(nom);
```

### Automatisation avec une Fonction

```sql
-- Fonction pour créer le même index sur toutes les tables d'une hiérarchie
CREATE OR REPLACE FUNCTION create_index_hierarchy(
    parent_table TEXT,
    index_name TEXT,
    columns TEXT
)
RETURNS VOID AS $$
DECLARE
    child_table TEXT;
BEGIN
    -- Index sur le parent
    EXECUTE format('CREATE INDEX IF NOT EXISTS %I ON %I(%s)',
                   index_name, parent_table, columns);

    -- Index sur chaque enfant
    FOR child_table IN
        SELECT c.relname
        FROM pg_inherits i
        JOIN pg_class c ON i.inhrelid = c.oid
        JOIN pg_class p ON i.inhparent = p.oid
        WHERE p.relname = parent_table
    LOOP
        EXECUTE format('CREATE INDEX IF NOT EXISTS %I ON %I(%s)',
                       index_name || '_' || child_table,
                       child_table,
                       columns);
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Utilisation
SELECT create_index_hierarchy('animaux', 'idx_nom', 'nom');
```

---

## Performances et Optimisation

### Constraint Exclusion

PostgreSQL peut optimiser les requêtes sur les hiérarchies en utilisant **constraint exclusion** si vous définissez des contraintes CHECK sur les tables enfants.

```sql
CREATE TABLE mesures (
    mesure_id SERIAL PRIMARY KEY,
    valeur NUMERIC(10,2),
    date_mesure DATE
);

-- Mesures 2023
CREATE TABLE mesures_2023 (
    CHECK (date_mesure >= '2023-01-01' AND date_mesure < '2024-01-01')
) INHERITS (mesures);

-- Mesures 2024
CREATE TABLE mesures_2024 (
    CHECK (date_mesure >= '2024-01-01' AND date_mesure < '2025-01-01')
) INHERITS (mesures);

-- Mesures 2025
CREATE TABLE mesures_2025 (
    CHECK (date_mesure >= '2025-01-01' AND date_mesure < '2026-01-01')
) INHERITS (mesures);

-- Activer constraint exclusion (souvent déjà activé)
SET constraint_exclusion = on;  -- ou 'partition' (recommandé)
```

**Requête optimisée :**

```sql
-- PostgreSQL sait qu'il doit scanner SEULEMENT mesures_2024
EXPLAIN SELECT * FROM mesures WHERE date_mesure = '2024-06-15';
```

**Plan d'exécution :**
```
Append
  ->  Seq Scan on mesures_2024
        Filter: (date_mesure = '2024-06-15'::date)
```

Les tables `mesures_2023` et `mesures_2025` sont **automatiquement exclues** du scan.

### Limites de Performance

⚠️ **Attention :** Si vous avez beaucoup de tables enfants (> 100), les performances peuvent se dégrader :

- Le planificateur doit analyser chaque table
- Overhead dans la planification de requêtes
- Problèmes de scalabilité

**Recommandation :** Pour des cas de partitionnement massif, utilisez le **partitionnement déclaratif** natif de PostgreSQL (depuis la version 10).

---

## Limites et Inconvénients Majeurs

### 1. Contraintes d'Intégrité Référentielle

❌ **Clés primaires et uniques ne traversent pas la hiérarchie**

```sql
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    nom VARCHAR(100)
);

CREATE TABLE clients_particuliers () INHERITS (clients);
CREATE TABLE clients_entreprises () INHERITS (clients);

-- PROBLÈME : On peut insérer le même email dans les deux tables enfants
INSERT INTO clients_particuliers (email, nom) VALUES ('test@example.com', 'Jean');
INSERT INTO clients_entreprises (email, nom) VALUES ('test@example.com', 'ACME Corp');
-- Ces deux insertions RÉUSSISSENT malgré UNIQUE sur la table parent !
```

**Explication :** La contrainte UNIQUE n'est vérifiée que **dans chaque table individuellement**, pas à travers la hiérarchie.

### 2. Foreign Keys Problématiques

❌ **Impossible de créer des FK fiables vers une hiérarchie**

```sql
-- Table référençant la hiérarchie
CREATE TABLE commandes (
    commande_id SERIAL PRIMARY KEY,
    client_id INTEGER REFERENCES clients(client_id),  -- INCOMPLET !
    montant NUMERIC(10,2)
);

-- Si client_id=42 est dans "clients_entreprises", la FK ne le trouvera pas
-- car elle cherche seulement dans "clients" (table parent)
```

### 3. Index Non Hérités

❌ **Les index doivent être créés manuellement sur chaque table**

Cela crée une maintenance importante et des risques d'oubli.

### 4. Manque de Support dans les ORMs

❌ **La plupart des ORMs ne supportent pas bien l'héritage**

- SQLAlchemy (Python) : Support limité
- Hibernate (Java) : Préfère d'autres stratégies
- Entity Framework (.NET) : Pas de support natif
- ActiveRecord (Ruby) : Support minimal

### 5. Migrations Complexes

❌ **Les migrations de schéma sont compliquées**

Changer la structure de la hiérarchie après coup peut être très difficile.

### 6. Confusion dans les Requêtes

❌ **Comportement parfois non intuitif**

```sql
-- Cette requête compte-t-elle les enfants ?
SELECT COUNT(*) FROM animaux;  -- Oui, inclut chiens et chats

-- Celle-ci ?
SELECT COUNT(*) FROM ONLY animaux;  -- Non, seulement le parent

-- Il faut toujours se souvenir du comportement
```

### 7. Statistiques et ANALYZE

❌ **ANALYZE doit être exécuté sur chaque table**

```sql
-- Pas suffisant :
ANALYZE animaux;

-- Il faut aussi :
ANALYZE chiens;
ANALYZE chats;
```

Les statistiques ne sont pas agrégées automatiquement.

---

## Alternatives Modernes

### 1. Partitionnement Déclaratif (Recommandé)

Depuis PostgreSQL 10, le **partitionnement déclaratif** est l'alternative recommandée pour la plupart des cas d'usage d'héritage.

```sql
-- Au lieu de l'héritage :
CREATE TABLE mesures (
    mesure_id BIGSERIAL,
    date_mesure DATE NOT NULL,
    valeur NUMERIC(10,2)
) PARTITION BY RANGE (date_mesure);

-- Partitions
CREATE TABLE mesures_2023 PARTITION OF mesures
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE mesures_2024 PARTITION OF mesures
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE mesures_2025 PARTITION OF mesures
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

**Avantages du partitionnement vs héritage :**
- ✅ Contraintes d'intégrité fonctionnent correctement
- ✅ Index hérités automatiquement
- ✅ Partition pruning optimisé
- ✅ Support natif et bien documenté
- ✅ Meilleure intégration avec les outils

**Voir :** Chapitre 11.4 pour les détails complets sur le partitionnement.

### 2. Approche "Single Table Inheritance"

Stocker tous les types dans une seule table avec une colonne discriminante.

```sql
CREATE TABLE animaux (
    animal_id SERIAL PRIMARY KEY,
    type_animal VARCHAR(20) CHECK (type_animal IN ('chien', 'chat', 'autre')),
    nom VARCHAR(100),
    age INTEGER,
    -- Colonnes spécifiques (peuvent être NULL selon le type)
    race VARCHAR(50),            -- Pour chiens
    dresse BOOLEAN,              -- Pour chiens
    type_poil VARCHAR(20),       -- Pour chats
    declaws BOOLEAN              -- Pour chats
);

-- Index partiel pour chaque type
CREATE INDEX idx_animaux_chiens ON animaux(nom, race) WHERE type_animal = 'chien';
CREATE INDEX idx_animaux_chats ON animaux(nom, type_poil) WHERE type_animal = 'chat';
```

**Avantages :**
- ✅ Une seule table = simplicité
- ✅ Toutes les contraintes fonctionnent
- ✅ Requêtes simples
- ✅ Support ORM natif

**Inconvénients :**
- ❌ Beaucoup de colonnes potentiellement NULL
- ❌ Moins "propre" conceptuellement

### 3. Approche "Class Table Inheritance"

Tables séparées reliées par des clés étrangères.

```sql
-- Table principale
CREATE TABLE animaux (
    animal_id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    age INTEGER,
    date_arrivee DATE DEFAULT CURRENT_DATE
);

-- Tables spécialisées avec FK
CREATE TABLE chiens (
    chien_id INTEGER PRIMARY KEY REFERENCES animaux(animal_id) ON DELETE CASCADE,
    race VARCHAR(50),
    dresse BOOLEAN DEFAULT FALSE
);

CREATE TABLE chats (
    chat_id INTEGER PRIMARY KEY REFERENCES animaux(animal_id) ON DELETE CASCADE,
    type_poil VARCHAR(20),
    declaws BOOLEAN DEFAULT FALSE
);
```

**Avantages :**
- ✅ Structure propre et normalisée
- ✅ Intégrité référentielle complète
- ✅ Pas de colonnes inutiles
- ✅ Bien supporté par les ORMs

**Inconvénients :**
- ❌ Nécessite des JOINs pour récupérer toutes les infos
- ❌ Plus complexe à interroger

### 4. Utilisation de JSONB pour Polymorphisme

Stocker les attributs spécifiques dans une colonne JSONB.

```sql
CREATE TABLE animaux (
    animal_id SERIAL PRIMARY KEY,
    type_animal VARCHAR(20),
    nom VARCHAR(100),
    age INTEGER,
    attributs_specifiques JSONB  -- Attributs variables selon le type
);

-- Index GIN pour recherche dans JSONB
CREATE INDEX idx_animaux_attributs ON animaux USING GIN (attributs_specifiques);

-- Insérer un chien
INSERT INTO animaux (type_animal, nom, age, attributs_specifiques) VALUES (
    'chien',
    'Rex',
    3,
    '{"race": "Berger Allemand", "dresse": true}'
);

-- Insérer un chat
INSERT INTO animaux (type_animal, nom, age, attributs_specifiques) VALUES (
    'chat',
    'Minou',
    2,
    '{"type_poil": "long", "declaws": false}'
);
```

**Avantages :**
- ✅ Très flexible
- ✅ Facile d'ajouter de nouveaux attributs
- ✅ Une seule table

**Inconvénients :**
- ❌ Moins de validation au niveau base de données
- ❌ Requêtes JSONB moins performantes que colonnes

**Voir :** Chapitre 11.2 pour les détails sur JSONB.

---

## Quand Utiliser l'Héritage de Tables ?

### ✅ Cas d'Usage Légitimes (Rares)

L'héritage de tables peut être approprié dans des cas **très spécifiques** :

#### 1. Partitionnement Manuel (Ancien PostgreSQL)

Si vous êtes bloqué sur PostgreSQL < 10 sans partitionnement déclaratif.

**Note :** Aujourd'hui, **migrez vers le partitionnement déclaratif**.

#### 2. Prototypage Rapide

Pour des preuves de concept où l'intégrité référentielle n'est pas critique.

```sql
-- Prototype : hiérarchie de documents
CREATE TABLE documents (doc_id SERIAL, titre TEXT);
CREATE TABLE articles () INHERITS (documents);
CREATE TABLE pages () INHERITS (documents);
CREATE TABLE fichiers () INHERITS (documents);
```

#### 3. Logging et Données Non-Critiques

Pour des logs où les contraintes strictes ne sont pas nécessaires.

```sql
CREATE TABLE logs_app (
    log_id BIGSERIAL,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    niveau VARCHAR(10),
    message TEXT
);

CREATE TABLE logs_2024_q1 (
    CHECK (timestamp >= '2024-01-01' AND timestamp < '2024-04-01')
) INHERITS (logs_app);

CREATE TABLE logs_2024_q2 (
    CHECK (timestamp >= '2024-04-01' AND timestamp < '2024-07-01')
) INHERITS (logs_app);
```

**Mais encore une fois :** Le partitionnement déclaratif est meilleur.

### ❌ Quand NE PAS Utiliser l'Héritage

#### 1. Relations Métier Critiques

```sql
-- ❌ ÉVITER : Clients/Commandes avec héritage
CREATE TABLE clients (...) ;
CREATE TABLE clients_premium () INHERITS (clients);
CREATE TABLE commandes (client_id REFERENCES clients...);  -- PROBLÈME FK
```

**Alternative :** Class Table Inheritance ou Single Table.

#### 2. Besoin de Contraintes Strictes

Si l'unicité, les FK, et l'intégrité sont critiques → **NE PAS** utiliser l'héritage.

#### 3. Compatibilité ORM Requise

La plupart des ORMs modernes ne supportent pas bien l'héritage PostgreSQL.

#### 4. Partitionnement de Données

Utilisez le **partitionnement déclaratif** à la place.

---

## Migration depuis l'Héritage

Si vous avez déjà une hiérarchie par héritage et souhaitez migrer, voici les stratégies :

### Vers le Partitionnement Déclaratif

```sql
-- Étape 1 : Créer la nouvelle table partitionnée
CREATE TABLE mesures_new (
    mesure_id BIGSERIAL,
    date_mesure DATE NOT NULL,
    valeur NUMERIC(10,2),
    PRIMARY KEY (mesure_id, date_mesure)
) PARTITION BY RANGE (date_mesure);

-- Étape 2 : Créer les partitions
CREATE TABLE mesures_2024 PARTITION OF mesures_new
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Étape 3 : Copier les données
INSERT INTO mesures_new SELECT * FROM mesures_old_2024;

-- Étape 4 : Basculer les noms (dans une transaction)
BEGIN;
    ALTER TABLE mesures RENAME TO mesures_old_backup;
    ALTER TABLE mesures_new RENAME TO mesures;
COMMIT;

-- Étape 5 : Mettre à jour les références dans l'application

-- Étape 6 : Supprimer l'ancienne hiérarchie après validation
DROP TABLE mesures_old_backup CASCADE;
```

### Vers Class Table Inheritance

```sql
-- Créer la nouvelle structure
CREATE TABLE animaux_new (
    animal_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    age INTEGER
);

CREATE TABLE chiens_new (
    chien_id INTEGER PRIMARY KEY REFERENCES animaux_new(animal_id),
    race VARCHAR(50)
);

-- Migrer les données
INSERT INTO animaux_new (animal_id, nom, age)
SELECT animal_id, nom, age FROM chiens;

INSERT INTO chiens_new (chien_id, race)
SELECT animal_id, race FROM chiens;

-- Basculer
```

---

## Inspection de la Hiérarchie

### Requêtes Système Utiles

```sql
-- Lister toutes les tables enfants d'une table parent
SELECT
    c.relname AS child_table,
    p.relname AS parent_table
FROM pg_inherits i
JOIN pg_class c ON i.inhrelid = c.oid
JOIN pg_class p ON i.inhparent = p.oid
WHERE p.relname = 'animaux';
```

**Résultat :**
```
child_table | parent_table
------------+--------------
chiens      | animaux
chats       | animaux
```

### Hiérarchie Complète

```sql
-- Obtenir toute la hiérarchie avec profondeur
WITH RECURSIVE hierarchy AS (
    -- Tables sans parent (racines)
    SELECT
        c.oid,
        c.relname,
        0 AS niveau,
        ARRAY[c.relname] AS chemin
    FROM pg_class c
    WHERE c.relkind = 'r'
      AND NOT EXISTS (
          SELECT 1 FROM pg_inherits WHERE inhrelid = c.oid
      )

    UNION ALL

    -- Tables enfants
    SELECT
        c.oid,
        c.relname,
        h.niveau + 1,
        h.chemin || c.relname
    FROM pg_class c
    JOIN pg_inherits i ON c.oid = i.inhrelid
    JOIN hierarchy h ON i.inhparent = h.oid
)
SELECT
    REPEAT('  ', niveau) || relname AS hierarchie,
    niveau,
    chemin
FROM hierarchy
ORDER BY chemin;
```

### Compter les Lignes par Table

```sql
-- Fonction pour compter les lignes dans toute une hiérarchie
CREATE OR REPLACE FUNCTION count_hierarchy(parent_table TEXT)
RETURNS TABLE(table_name TEXT, row_count BIGINT) AS $$
DECLARE
    child_table TEXT;
BEGIN
    -- Compter dans la table parent (ONLY)
    RETURN QUERY EXECUTE format(
        'SELECT %L::TEXT, COUNT(*)::BIGINT FROM ONLY %I',
        parent_table, parent_table
    );

    -- Compter dans chaque table enfant
    FOR child_table IN
        SELECT c.relname
        FROM pg_inherits i
        JOIN pg_class c ON i.inhrelid = c.oid
        JOIN pg_class p ON i.inhparent = p.oid
        WHERE p.relname = parent_table
    LOOP
        RETURN QUERY EXECUTE format(
            'SELECT %L::TEXT, COUNT(*)::BIGINT FROM %I',
            child_table, child_table
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Utilisation
SELECT * FROM count_hierarchy('animaux');
```

**Résultat :**
```
table_name | row_count
-----------+-----------
animaux    | 1
chiens     | 2
chats      | 2
```

---

## Bonnes Pratiques si Vous Utilisez l'Héritage

Si vous devez absolument utiliser l'héritage, suivez ces pratiques :

### 1. Documenter Clairement

```sql
-- Documenter la hiérarchie et ses limitations
COMMENT ON TABLE animaux IS
'Table parent de la hiérarchie animaux.
ATTENTION : Les contraintes PK/FK ne traversent pas la hiérarchie.
Utiliser tableoid::regclass pour identifier le type.';

COMMENT ON TABLE chiens IS
'Hérite de animaux. Stocke les chiens spécifiquement.';
```

### 2. Créer les Contraintes Manuellement

```sql
-- Ne pas oublier les PK sur les enfants
ALTER TABLE chiens ADD PRIMARY KEY (animal_id);
ALTER TABLE chats ADD PRIMARY KEY (animal_id);

-- Contraintes CHECK pour éviter les conflits
ALTER TABLE chiens ADD CONSTRAINT chk_type CHECK (TRUE);
```

### 3. Créer les Index sur Tous les Niveaux

```sql
-- Script pour créer les index partout
CREATE INDEX idx_animaux_nom ON animaux(nom);
CREATE INDEX idx_chiens_nom ON chiens(nom);
CREATE INDEX idx_chats_nom ON chats(nom);
```

### 4. Utiliser tableoid Systématiquement

```sql
-- Toujours distinguer le type dans les requêtes
SELECT
    tableoid::regclass AS type,
    animal_id,
    nom,
    age
FROM animaux
ORDER BY tableoid, nom;
```

### 5. ANALYZE Régulièrement

```sql
-- Script de maintenance
DO $$
DECLARE
    tbl TEXT;
BEGIN
    FOR tbl IN
        SELECT tablename
        FROM pg_tables
        WHERE schemaname = 'public'
          AND tablename IN ('animaux', 'chiens', 'chats')
    LOOP
        EXECUTE 'ANALYZE ' || tbl;
        RAISE NOTICE 'Analyzed %', tbl;
    END LOOP;
END $$;
```

### 6. Tester les Migrations

Avant de déployer :
```sql
-- Test dans une transaction
BEGIN;
    ALTER TABLE animaux ADD COLUMN nouvelle_col TEXT;
    -- Vérifier que les enfants l'ont aussi
    SELECT column_name FROM information_schema.columns
    WHERE table_name IN ('chiens', 'chats')
      AND column_name = 'nouvelle_col';
ROLLBACK;  -- ou COMMIT si OK
```

---

## Comparaison : Héritage vs Alternatives

### Tableau Récapitulatif

| Critère | Héritage | Partitionnement | Class Table | Single Table | JSONB |
|---------|----------|----------------|-------------|--------------|-------|
| **Intégrité PK/FK** | ❌ Limitée | ✅ Complète | ✅ Complète | ✅ Complète | ✅ Complète |
| **Performance requêtes** | ⚠️ Variable | ✅ Excellente | ⚠️ JOINs requis | ✅ Bonne | ⚠️ Variable |
| **Flexibilité schéma** | ✅ Bonne | ❌ Structure fixe | ⚠️ Moyenne | ❌ Colonnes fixes | ✅ Excellente |
| **Complexité** | ⚠️ Moyenne | ⚠️ Moyenne | ⚠️ Moyenne-Haute | ✅ Faible | ⚠️ Moyenne |
| **Support ORM** | ❌ Faible | ✅ Bon | ✅ Excellent | ✅ Excellent | ✅ Bon |
| **Maintenance** | ❌ Difficile | ✅ Bonne | ✅ Bonne | ✅ Facile | ✅ Facile |
| **Cas d'usage** | Logs, legacy | Partitionnement temps | Hiérarchies métier | Types simples | Données variables |

---

## Conclusion

### Points Clés à Retenir

1. **L'héritage de tables est une fonctionnalité unique de PostgreSQL**, mais elle a des **limitations importantes**
2. **Les contraintes PK, FK et UNIQUE ne traversent pas la hiérarchie** - c'est la limitation majeure
3. **Le partitionnement déclaratif** est l'alternative moderne recommandée pour la plupart des cas
4. **Les ORMs ne supportent pas bien l'héritage** - problème pour les applications modernes
5. **L'héritage peut être utile dans des cas très spécifiques**, mais rarement en production

### Recommandations Générales

**Pour les nouveaux projets :**
- ❌ **Évitez l'héritage de tables** sauf cas très spécifique
- ✅ Utilisez le **partitionnement déclaratif** pour partitionner
- ✅ Utilisez **Class Table Inheritance** avec FK pour hiérarchies métier
- ✅ Utilisez **Single Table** avec colonne discriminante pour types simples
- ✅ Utilisez **JSONB** pour données vraiment variables

**Pour les projets existants :**
- Évaluez les limitations actuelles
- Planifiez une migration si l'intégrité référentielle est critique
- Documentez clairement les contraintes

### Vision Future

PostgreSQL continue d'améliorer le **partitionnement déclaratif**, qui est l'évolution naturelle de l'héritage pour les cas de partitionnement. Les cas d'usage légitimes de l'héritage de tables deviennent de plus en plus rares.

**Citation de la documentation PostgreSQL :**
> "Table inheritance is often not what you want. Consider using declarative partitioning instead."

Cette phrase résume bien la position actuelle de la communauté PostgreSQL sur l'héritage de tables.

---

## Ressources pour Aller Plus Loin

### Documentation Officielle

- [PostgreSQL Table Inheritance](https://www.postgresql.org/docs/current/ddl-inherit.html)
- [Declarative Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)

### Articles et Discussions

- "Why Table Inheritance is (Usually) a Bad Idea" - Divers blogs PostgreSQL
- Discussions sur pgsql-general mailing list
- Articles de 2ndQuadrant, Percona sur les alternatives

### Prochains Chapitres

- **11.4. Partitionnement de tables** : L'alternative moderne et recommandée
- **11.2. Modélisation JSONB** : Pour la flexibilité de schéma

---

**Fin du Chapitre 11.3**

Ce chapitre vous a présenté l'héritage de tables, une fonctionnalité historique de PostgreSQL. Bien qu'elle offre certaines possibilités intéressantes, ses limitations importantes font qu'elle est rarement recommandée pour de nouveaux projets. Privilégiez le partitionnement déclaratif ou les approches relationnelles classiques pour vos hiérarchies de données.

Dans le prochain chapitre, nous explorerons le **partitionnement de tables**, qui est aujourd'hui la solution recommandée pour la plupart des cas d'usage qui auraient pu historiquement utiliser l'héritage.

⏭️ [Partitionnement de tables (Déclaratif : Range, List, Hash)](/11-modelisation-avancee/04-partitionnement-de-tables.md)
