🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3. CREATE TABLE : Définir une structure

## Introduction

Après avoir compris la hiérarchie logique de PostgreSQL et le concept de schémas, il est temps d'apprendre à créer l'élément central d'une base de données : **la table**.

Une table est la structure qui contient vos données. C'est l'équivalent d'un tableau Excel ou d'une feuille de calcul, mais avec des règles strictes sur ce que chaque colonne peut contenir.

Dans cette section, nous allons apprendre :
- Comment définir une structure de table
- Les éléments constitutifs d'une table
- Les contraintes de base
- Les bonnes pratiques de création de tables

---

## Qu'est-ce qu'une Table ?

### Définition

Une **table** est une collection structurée de données organisées en lignes et colonnes :

- **Colonnes** (ou champs) : Définissent la **structure** de la table. Chaque colonne a un nom, un type de données et des règles optionnelles.  
- **Lignes** (ou enregistrements/tuples) : Contiennent les **données réelles**. Chaque ligne représente une entité ou un événement.

### Analogie

Pensez à une table comme à un formulaire papier :
- Les **colonnes** sont les champs du formulaire (Nom, Prénom, Date de naissance, etc.)
- Les **lignes** sont les formulaires remplis par différentes personnes

```
Table: employes
┌────┬───────────┬──────────┬────────────┬────────┐
│ id │ prenom    │ nom      │ email      │ salaire│
├────┼───────────┼──────────┼────────────┼────────┤
│ 1  │ Alice     │ Martin   │ a@mail.com │ 45000  │  ← Ligne 1
│ 2  │ Bob       │ Dupont   │ b@mail.com │ 50000  │  ← Ligne 2
│ 3  │ Charlie   │ Bernard  │ c@mail.com │ 48000  │  ← Ligne 3
└────┴───────────┴──────────┴────────────┴────────┘
  ↑      ↑          ↑          ↑          ↑
Colonnes (structure définie à la création)
```

---

## Syntaxe de Base de CREATE TABLE

### Forme la plus simple

Voici la syntaxe minimale pour créer une table :

```sql
CREATE TABLE nom_de_la_table (
    nom_colonne1 type_de_donnees,
    nom_colonne2 type_de_donnees,
    nom_colonne3 type_de_donnees
);
```

### Premier exemple concret

Créons une table simple pour stocker des livres :

```sql
CREATE TABLE livres (
    titre VARCHAR(200),
    auteur VARCHAR(100),
    annee_publication INT,
    prix DECIMAL(10, 2)
);
```

Cette commande crée une table nommée `livres` avec 4 colonnes :
- `titre` : Peut contenir jusqu'à 200 caractères  
- `auteur` : Peut contenir jusqu'à 100 caractères  
- `annee_publication` : Nombre entier  
- `prix` : nombre décimal avec 2 décimales (ex. : 19.99)

### Vérifier que la table existe

```sql
-- Dans psql
\dt

-- Voir la structure de la table
\d livres

-- Avec une requête SQL
SELECT * FROM livres;  -- Retourne 0 ligne (table vide)
```

---

## Les Éléments d'une Définition de Colonne

Chaque colonne d'une table peut avoir plusieurs attributs :

```sql
nom_colonne TYPE [CONTRAINTE] [DEFAULT valeur] [COMMENT]
```

Décortiquons chaque élément :

### 1. Nom de la colonne

**Règles de nommage :**
- Commence par une lettre ou un underscore
- Peut contenir des lettres, chiffres et underscores
- Sensible à la casse si entouré de guillemets doubles
- Maximum 63 caractères

```sql
-- Noms valides
CREATE TABLE exemples (
    id INT,
    nom_complet VARCHAR(100),
    date_creation DATE,
    est_actif BOOLEAN
);

-- Avec guillemets (permet caractères spéciaux, mais à éviter)
CREATE TABLE exemple_guillemets (
    "Prénom" VARCHAR(50),  -- Contient un accent
    "Date de Naissance" DATE  -- Contient des espaces
);
```

**Bonnes pratiques :**
- Utilisez des noms en minuscules avec des underscores : `date_creation`
- Évitez les espaces et caractères spéciaux
- Soyez descriptif : `email` plutôt que `e` ou `ml`
- Évitez les mots réservés SQL : `user`, `table`, `select`, etc.

### 2. Type de données

Le type de données définit ce que la colonne peut contenir. Nous verrons les types en détail dans les prochaines sections. Voici les plus courants :

| Type | Description | Exemple |
|------|-------------|---------|
| `INT` / `INTEGER` | Nombre entier (32 bits) | 42, -15, 1000 |
| `BIGINT` | Grand entier (64 bits) | 9 quintillions max |
| `SERIAL` | Entier auto-incrémenté (legacy, voir IDENTITY) | 1, 2, 3, … |
| `VARCHAR(n)` | Texte variable (max n caractères) | 'Bonjour', 'Alice' |
| `TEXT` | Texte de longueur illimitée | Longs articles, descriptions |
| `NUMERIC(p,s)` / `DECIMAL(p,s)` | Nombre décimal précis (exact). `DECIMAL` est un **alias** de `NUMERIC`. | 19.99, 1234.56 |
| `DATE` | Date (sans heure) | 2025-11-19 |
| `TIMESTAMP` | Date et heure (sans fuseau) | 2025-11-19 14:30:00 |
| `TIMESTAMPTZ` | Date et heure avec fuseau (**recommandé**) | 2025-11-19 14:30:00+01 |
| `BOOLEAN` | Vrai ou Faux | TRUE, FALSE |
| `UUID` | Identifiant universellement unique (128 bits) | `01928c5e-…` |
| `JSONB` | JSON binaire indexable | `'{"a": 1}'` |

```sql
CREATE TABLE exemple_types (
    id SERIAL,
    nom VARCHAR(100),
    description TEXT,
    prix DECIMAL(10, 2),
    date_creation DATE,
    est_disponible BOOLEAN
);
```

### 3. Contraintes (facultatives)

Les contraintes définissent des règles sur les valeurs acceptées. Nous les verrons en détail plus loin.

### 4. Valeur par défaut (facultative)

Vous pouvez spécifier une valeur par défaut avec `DEFAULT` :

```sql
CREATE TABLE utilisateurs (
    id SERIAL,
    nom VARCHAR(100),
    date_inscription DATE DEFAULT CURRENT_DATE,
    est_actif BOOLEAN DEFAULT TRUE,
    points INT DEFAULT 0
);
```

Quand vous insérez une ligne sans spécifier ces colonnes, PostgreSQL utilise les valeurs par défaut :

```sql
INSERT INTO utilisateurs (nom) VALUES ('Alice');
-- PostgreSQL ajoute automatiquement :
-- - date_inscription = date du jour
-- - est_actif = TRUE
-- - points = 0

SELECT * FROM utilisateurs;
```

Résultat :
```
id | nom   | date_inscription | est_actif | points
---+-------+------------------+-----------+-------
1  | Alice | 2025-11-19       | t         | 0
```

---

## Les Contraintes de Base

Les **contraintes** sont des règles qui garantissent l'intégrité de vos données. Voici les plus importantes :

### 1. NOT NULL

**Objectif** : empêcher les valeurs NULL (vides) dans une colonne.

```sql
CREATE TABLE produits (
    id SERIAL,
    nom VARCHAR(200) NOT NULL,  -- Obligatoire
    description TEXT,            -- Facultatif (NULL autorisé)
    prix DECIMAL(10, 2) NOT NULL -- Obligatoire
);

-- Cette insertion fonctionne
INSERT INTO produits (nom, prix) VALUES ('Ordinateur', 999.99);

-- Cette insertion échoue (nom manquant)
INSERT INTO produits (prix) VALUES (500.00);
-- ERROR: null value in column "nom" violates not-null constraint
```

**Quand l'utiliser :**
- Pour les informations essentielles (nom, email, date de création)
- Pour garantir qu'une colonne a toujours une valeur

### 2. DEFAULT

**Objectif** : fournir une valeur par défaut quand aucune valeur n'est spécifiée.

```sql
CREATE TABLE commandes (
    id SERIAL,
    numero_commande VARCHAR(50),
    statut VARCHAR(20) DEFAULT 'EN_ATTENTE',
    date_commande TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    montant DECIMAL(10, 2) NOT NULL
);

-- Sans spécifier statut et date_commande
INSERT INTO commandes (numero_commande, montant)  
VALUES ('CMD-001', 150.50);  

SELECT * FROM commandes;
```

Résultat :
```
id | numero_commande | statut     | date_commande         | montant
---+-----------------+------------+-----------------------+---------
1  | CMD-001         | EN_ATTENTE | 2025-11-19 14:30:00   | 150.50
```

**Valeurs par défaut courantes :**
- Dates : `CURRENT_DATE`, `CURRENT_TIMESTAMP`
- Booléens : `TRUE` ou `FALSE`
- Nombres : `0`
- Texte : `'valeur_defaut'`
- UUID : `gen_random_uuid()` (fonction intégrée depuis PostgreSQL 13)

### 3. PRIMARY KEY (Clé Primaire)

**Objectif** : identifier de manière unique chaque ligne d'une table.

Une clé primaire :
- Ne peut **jamais** être NULL
- Doit être **unique** pour chaque ligne
- Il ne peut y avoir qu'**une seule** clé primaire par table

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL
);

-- Insertion réussie
INSERT INTO clients (nom, email) VALUES ('Alice', 'alice@example.com');  
INSERT INTO clients (nom, email) VALUES ('Bob', 'bob@example.com');  

-- Cette insertion échoue (id dupliqué)
INSERT INTO clients (id, nom, email) VALUES (1, 'Charlie', 'charlie@example.com');
-- ERROR: duplicate key value violates unique constraint "clients_pkey"
```

**Bonnes pratiques :**
- Toute table devrait avoir une clé primaire
- En 2026, préférez `GENERATED ALWAYS AS IDENTITY` (standard SQL) à `SERIAL` (legacy PostgreSQL)
- Pour les systèmes distribués ou exposés via API publique, considérez `UUID` (notamment `UUIDv7` en PG 18)
- Nommez la colonne `id` par convention

**Syntaxes alternatives :**

```sql
-- Méthode 1 : SERIAL (héritage PostgreSQL, encore très utilisé)
CREATE TABLE table1 (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

-- Méthode 2 : IDENTITY (standard SQL, recommandé depuis PG 10)
CREATE TABLE table1_modern (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100)
);

-- Méthode 3 : Contrainte de table
CREATE TABLE table2 (
    id INTEGER GENERATED ALWAYS AS IDENTITY,
    nom VARCHAR(100),
    PRIMARY KEY (id)
);

-- Méthode 4 : Clé primaire composée (plusieurs colonnes)
CREATE TABLE inscriptions (
    etudiant_id INT,
    cours_id INT,
    date_inscription DATE,
    PRIMARY KEY (etudiant_id, cours_id)
);

-- Méthode 5 : 🆕 UUIDv7 (PostgreSQL 18) — idéal pour les API publiques et les systèmes distribués
CREATE TABLE table_distribuee (
    id UUID PRIMARY KEY DEFAULT uuidv7(),
    nom VARCHAR(100)
);
```

> 💡 **`SERIAL` vs `IDENTITY` : que choisir ?**  
>  
> - `SERIAL` est l'ancien mécanisme PostgreSQL : crée une séquence sous le capot, mais moins bien intégré au standard SQL. L'utilisateur peut « tricher » en insérant manuellement une valeur.  
> - `IDENTITY` est conforme au **standard SQL:2003**, plus propre, mieux protégé. Avec `GENERATED ALWAYS`, l'utilisateur **ne peut pas** insérer manuellement une valeur dans cette colonne (sauf à utiliser `OVERRIDING SYSTEM VALUE`).  
> - Pour un nouveau projet en PG 10+, **`IDENTITY` est le choix recommandé** par la communauté PostgreSQL.  
> - Le sujet sera détaillé en section 4.5 (Séquences et génération automatique).

### 4. UNIQUE

**Objectif** : garantir qu'une valeur n'apparaît qu'une seule fois dans la colonne (mais NULL est autorisé plusieurs fois).

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,  -- Un seul email par utilisateur
    telephone VARCHAR(20) UNIQUE         -- Facultatif mais unique si renseigné
);

-- Insertions réussies
INSERT INTO utilisateurs (nom, email) VALUES ('Alice', 'alice@example.com');  
INSERT INTO utilisateurs (nom, email) VALUES ('Bob', 'bob@example.com');  

-- Cette insertion échoue (email déjà existant)
INSERT INTO utilisateurs (nom, email) VALUES ('Charlie', 'alice@example.com');
-- ERROR: duplicate key value violates unique constraint "utilisateurs_email_key"

-- Ces insertions réussissent (telephone NULL autorisé plusieurs fois)
INSERT INTO utilisateurs (nom, email, telephone) VALUES ('Dave', 'dave@example.com', NULL);  
INSERT INTO utilisateurs (nom, email, telephone) VALUES ('Eve', 'eve@example.com', NULL);  
```

**Différence entre UNIQUE et PRIMARY KEY :**

| Caractéristique | PRIMARY KEY | UNIQUE |
|----------------|-------------|--------|
| Valeurs NULL | Interdit | Autorisé |
| Nombre par table | Une seule | Plusieurs possibles |
| Utilisé pour les relations | Oui | Rarement |
| Index créé automatiquement | Oui | Oui |

### 5. CHECK

**Objectif** : définir une condition personnalisée que les valeurs doivent respecter.

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,
    prix DECIMAL(10, 2) NOT NULL CHECK (prix > 0),
    quantite INT NOT NULL CHECK (quantite >= 0),
    note DECIMAL(3, 2) CHECK (note >= 0 AND note <= 5)
);

-- Insertion réussie
INSERT INTO produits (nom, prix, quantite, note)  
VALUES ('Laptop', 999.99, 10, 4.5);  

-- Cette insertion échoue (prix négatif)
INSERT INTO produits (nom, prix, quantite)  
VALUES ('Souris', -10.00, 5);  
-- ERROR: new row violates check constraint "produits_prix_check"

-- Cette insertion échoue (note hors limites)
INSERT INTO produits (nom, prix, quantite, note)  
VALUES ('Clavier', 59.99, 20, 5.5);  
-- ERROR: new row violates check constraint "produits_note_check"
```

**Exemples de contraintes CHECK courantes :**

```sql
-- Vérifier qu'une date est dans le futur
date_evenement DATE CHECK (date_evenement > CURRENT_DATE)

-- Vérifier qu'un email contient un @
email VARCHAR(255) CHECK (email LIKE '%@%')

-- Vérifier qu'une valeur est dans une liste
statut VARCHAR(20) CHECK (statut IN ('actif', 'inactif', 'suspendu'))

-- Vérifier une relation entre colonnes
date_fin DATE CHECK (date_fin > date_debut)
```

### 6. FOREIGN KEY (Clés Étrangères)

**Objectif** : garantir l'**intégrité référentielle** — une valeur doit exister dans une autre table. C'est le mécanisme qui **relie les tables** entre elles.

Une clé étrangère (`FOREIGN KEY`) impose qu'une colonne (ou un groupe de colonnes) corresponde à une clé **`PRIMARY KEY`** (ou **`UNIQUE`**) d'une table **référencée**. PostgreSQL refuse alors d'insérer une ligne « enfant » pointant vers un parent inexistant, et de supprimer un parent encore référencé (selon l'action choisie).

```sql
CREATE TABLE clients (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE commandes (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id),  -- clé étrangère
    montant NUMERIC(10, 2)
);

INSERT INTO clients (nom) VALUES ('Alice');                    -- id = 1
INSERT INTO commandes (client_id, montant) VALUES (1, 99.90);  -- OK : le client 1 existe

-- Client inexistant → rejet
INSERT INTO commandes (client_id, montant) VALUES (999, 10.00);
-- ERROR: insert or update on table "commandes" violates foreign key constraint
```

**Deux syntaxes** :

```sql
-- Syntaxe colonne (la plus courte)
client_id INTEGER REFERENCES clients(id)

-- Syntaxe contrainte de table (obligatoire pour les clés composites, et nommable)
CONSTRAINT fk_commandes_client FOREIGN KEY (client_id) REFERENCES clients(id)
```

> 💡 Si vous omettez la colonne référencée (`REFERENCES clients`), PostgreSQL utilise la **clé primaire** de la table cible.

#### Actions `ON DELETE` et `ON UPDATE`

Que faire quand le parent référencé est **supprimé** (ou que sa clé est **modifiée**) ? On le précise avec `ON DELETE` / `ON UPDATE` :

| Action | Comportement quand le parent disparaît |
|--------|----------------------------------------|
| `NO ACTION` (défaut) | Rejet si des enfants existent (vérifié en fin d'instruction, donc compatible avec une réaffectation dans la même requête) |
| `RESTRICT` | Rejet immédiat si des enfants existent |
| `CASCADE` | Supprime (ou met à jour) aussi les lignes enfants |
| `SET NULL` | Met la clé étrangère des enfants à `NULL` |
| `SET DEFAULT` | Met la clé étrangère des enfants à leur valeur `DEFAULT` |

```sql
CREATE TABLE lignes_commande (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    commande_id INTEGER NOT NULL
        REFERENCES commandes(id) ON DELETE CASCADE,  -- supprimer la commande supprime ses lignes
    produit VARCHAR(200),
    quantite INTEGER
);
```

> 🆕 **PostgreSQL 15+** : `ON DELETE SET NULL (colonne)` (et `SET DEFAULT (colonne)`) permet de ne réinitialiser **qu'une partie** des colonnes d'une clé étrangère composite.

#### Bonnes pratiques

- **Indexez les colonnes de clé étrangère** : PostgreSQL n'indexe **pas** automatiquement le côté « enfant ». Sans index, chaque suppression d'un parent provoque un *seq scan* de la table enfant.
  ```sql
  CREATE INDEX idx_commandes_client_id ON commandes(client_id);
  ```
- Sur une grande table existante, ajoutez la FK avec `NOT VALID` puis `VALIDATE CONSTRAINT` pour éviter un long verrou (voir section 4.7).
- Une clé étrangère ne peut référencer qu'une clé **`PRIMARY KEY`** ou **`UNIQUE`** de la table cible.

---

## Contraintes Temporelles (🆕 PostgreSQL 18)

PostgreSQL 18 implémente les **contraintes temporelles** du standard SQL:2011 : une clé primaire ou étrangère peut inclure une **période de validité** (un type *range*), de sorte que l'unicité ou la référence ne s'applique qu'à des **périodes qui ne se chevauchent pas**.

### `PRIMARY KEY … WITHOUT OVERLAPS`

`WITHOUT OVERLAPS` rend une clé primaire « temporelle » : deux lignes peuvent partager la même clé classique **à condition** que leurs périodes ne se chevauchent pas.

```sql
-- Prérequis : l'opérateur && (chevauchement) dans un index exige btree_gist
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE occupation_chambre (
    chambre_id INTEGER,
    sejour     DATERANGE,
    -- Une chambre ne peut pas être occupée deux fois sur des périodes qui se chevauchent
    PRIMARY KEY (chambre_id, sejour WITHOUT OVERLAPS)
);

INSERT INTO occupation_chambre VALUES (101, daterange('2025-12-01','2025-12-05'));
INSERT INTO occupation_chambre VALUES (101, daterange('2025-12-05','2025-12-09'));  -- OK : périodes contiguës (bornes [) )

-- Chevauchement sur la même chambre → rejet
INSERT INTO occupation_chambre VALUES (101, daterange('2025-12-03','2025-12-08'));
-- ERROR: conflicting key value violates exclusion constraint "occupation_chambre_pkey"
```

> Avant PG 18, on obtenait ce résultat avec une **contrainte d'exclusion** `EXCLUDE USING GIST (chambre_id WITH =, sejour WITH &&)` (voir section 4.4.3). `WITHOUT OVERLAPS` est la forme **standard**, plus lisible.

### `FOREIGN KEY … PERIOD`

Le pendant côté clé étrangère : `PERIOD` exige que la période de l'enfant soit **couverte** par celle du parent.

```sql
CREATE TABLE reservation (
    id          INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    chambre_id  INTEGER,
    sejour      DATERANGE,
    FOREIGN KEY (chambre_id, PERIOD sejour)
        REFERENCES occupation_chambre (chambre_id, PERIOD sejour)
);
```

**Cas d'usage** : réservations, tarifs valables sur une période, historisation (*versioning*) de données, plannings de ressources.

---

## Les Colonnes Générées (GENERATED COLUMNS)

Depuis PostgreSQL 12, vous pouvez créer des **colonnes dont la valeur est calculée automatiquement** à partir d'autres colonnes de la même ligne. PostgreSQL 18 enrichit ce mécanisme avec un nouveau mode VIRTUAL.

### Deux modes de calcul

| Mode | Comportement | Disponible depuis | Coût |
|------|--------------|-------------------|------|
| `STORED` | Valeur calculée et **stockée sur disque** lors de l'INSERT/UPDATE | PG 12 | Espace disque + écriture |
| `VIRTUAL` 🆕 | Valeur calculée **à chaque lecture**, jamais stockée | **PG 18** | CPU (à la lecture) |

### Colonne générée STORED (PG 12+)

```sql
CREATE TABLE produits (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,
    prix_ht NUMERIC(10, 2) NOT NULL,
    tva NUMERIC(4, 3) DEFAULT 0.200,  -- 20 %
    -- Colonne calculée et stockée sur disque
    prix_ttc NUMERIC(10, 2)
        GENERATED ALWAYS AS (prix_ht * (1 + tva)) STORED
);

INSERT INTO produits (nom, prix_ht) VALUES ('Livre', 10.00);  
SELECT * FROM produits;  
-- id | nom   | prix_ht | tva   | prix_ttc
-- ---+-------+---------+-------+----------
-- 1  | Livre | 10.00   | 0.200 | 12.00      ← calculé automatiquement
```

**Avantages STORED** : lecture instantanée, indexable.  
**Inconvénient** : occupe de l'espace disque, recalculé à chaque UPDATE des colonnes sources.  

### Colonne générée VIRTUAL (🆕 PG 18)

```sql
CREATE TABLE produits_v2 (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,
    prix_ht NUMERIC(10, 2) NOT NULL,
    tva NUMERIC(4, 3) DEFAULT 0.200,
    -- Colonne calculée à chaque lecture, sans stockage
    prix_ttc NUMERIC(10, 2)
        GENERATED ALWAYS AS (prix_ht * (1 + tva)) VIRTUAL
);
```

**Avantages VIRTUAL** :
- Zéro espace disque supplémentaire
- Toujours cohérent (recalculé à chaque accès)
- Idéal pour les calculs simples consultés peu fréquemment

**Inconvénients** :
- CPU à chaque lecture
- Non indexable directement (utiliser un index sur expression à la place)
- Ne peut pas référencer d'autres colonnes générées

### Cas d'usage typiques

| Cas | Mode recommandé | Pourquoi |
|-----|-----------------|----------|
| Colonne fréquemment lue et filtrée | `STORED` | Indexable, lecture rapide |
| Calcul lourd (concaténation, agrégation, sous-requête) | `STORED` | Évite de recalculer à chaque accès |
| Colonne simple (multiplication, soustraction) | `VIRTUAL` | Économie d'espace, négligeable en CPU |
| Schéma en évolution, économie de stockage | `VIRTUAL` | Modifiable plus facilement |
| Migration depuis MySQL (qui supporte VIRTUAL natif) | `VIRTUAL` | Compatibilité directe |

> ⚠️ **Restrictions** : une colonne générée ne peut pas avoir de `DEFAULT`, ne peut pas être utilisée dans une clé primaire (sauf `STORED`), et son expression doit être *immuable* (pas de fonctions volatiles comme `random()` ou `now()`).

---

## Exemples Complets et Progressifs

### Exemple 1 : Table Simple

Une table pour gérer des tâches (TODO list) :

```sql
CREATE TABLE taches (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(200) NOT NULL,
    description TEXT,
    est_terminee BOOLEAN DEFAULT FALSE,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer des données
INSERT INTO taches (titre, description)  
VALUES ('Apprendre PostgreSQL', 'Lire le tutoriel complet');  

INSERT INTO taches (titre, est_terminee)  
VALUES ('Faire les courses', TRUE);  

-- Consulter
SELECT * FROM taches;
```

### Exemple 2 : Table avec Contraintes

Une table pour gérer des employés :

```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    numero_employe VARCHAR(20) UNIQUE NOT NULL,
    prenom VARCHAR(50) NOT NULL,
    nom VARCHAR(50) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    date_embauche DATE NOT NULL DEFAULT CURRENT_DATE,
    salaire DECIMAL(10, 2) NOT NULL CHECK (salaire > 0),
    departement VARCHAR(100),
    est_actif BOOLEAN DEFAULT TRUE,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertion valide
INSERT INTO employes (numero_employe, prenom, nom, email, salaire, departement)  
VALUES ('EMP001', 'Alice', 'Martin', 'alice.martin@company.com', 45000.00, 'IT');  

-- Insertion avec valeurs par défaut
INSERT INTO employes (numero_employe, prenom, nom, email, salaire)  
VALUES ('EMP002', 'Bob', 'Dupont', 'bob.dupont@company.com', 50000.00);  
```

### Exemple 3 : Table avec Contrainte Multi-Colonnes

Une table pour des articles de blog avec contrainte CHECK sur plusieurs colonnes :

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300) NOT NULL,
    slug VARCHAR(300) UNIQUE NOT NULL,
    contenu TEXT NOT NULL,
    statut VARCHAR(20) DEFAULT 'brouillon'
        CHECK (statut IN ('brouillon', 'publie', 'archive')),
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    date_publication TIMESTAMP,
    nombre_vues INT DEFAULT 0 CHECK (nombre_vues >= 0),

    -- Contrainte : si publié, date_publication doit être renseignée
    CHECK (
        (statut = 'publie' AND date_publication IS NOT NULL) OR
        (statut != 'publie')
    )
);

-- Insertion d'un brouillon (OK)
INSERT INTO articles (titre, slug, contenu)  
VALUES ('Mon premier article', 'mon-premier-article', 'Contenu du premier article');  

-- Insertion d'un article publié avec date (OK)
INSERT INTO articles (titre, slug, contenu, statut, date_publication)  
VALUES ('Article publié', 'article-publie', 'Contenu publié', 'publie', CURRENT_TIMESTAMP);  

-- Tentative de publier sans date (ERREUR)
INSERT INTO articles (titre, slug, contenu, statut)  
VALUES ('Article sans date', 'article-sans-date', 'Contenu', 'publie');  
-- ERROR: check constraint violated
```

---

## Options Avancées de CREATE TABLE

### Spécifier le schéma

```sql
-- Créer la table dans un schéma spécifique
CREATE TABLE blog.articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300) NOT NULL
);

-- Créer le schéma s'il n'existe pas, puis la table
CREATE SCHEMA IF NOT EXISTS blog;  
CREATE TABLE blog.articles (  
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300) NOT NULL
);
```

### IF NOT EXISTS

Éviter une erreur si la table existe déjà :

```sql
-- Sans IF NOT EXISTS (erreur si existe déjà)
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200)
);

-- Avec IF NOT EXISTS (pas d'erreur si existe déjà)
CREATE TABLE IF NOT EXISTS produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200)
);
```

### TEMPORARY TABLE

Créer une table temporaire (disparaît à la fin de la session) :

```sql
-- Table temporaire (supprimée automatiquement à la déconnexion)
CREATE TEMPORARY TABLE temp_calculs (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    resultat NUMERIC(10, 2)
);

-- Ou avec l'alias TEMP
CREATE TEMP TABLE temp_stats (
    categorie VARCHAR(50),
    total INTEGER
);
```

#### Contrôle du cycle de vie : clause `ON COMMIT`

Une table temporaire peut être configurée pour réagir au `COMMIT` d'une transaction :

| Option | Comportement |
|--------|--------------|
| `ON COMMIT PRESERVE ROWS` (défaut) | Les données restent jusqu'à la fin de la **session** |
| `ON COMMIT DELETE ROWS` | Les données sont vidées à chaque `COMMIT` (la table reste) |
| `ON COMMIT DROP` | La table est **supprimée** au prochain `COMMIT` (utile pour une seule transaction) |

```sql
BEGIN;  
CREATE TEMP TABLE batch_traitement (id INT, payload JSONB)  
    ON COMMIT DROP;
-- … traitements …
COMMIT;
-- batch_traitement n'existe plus
```

**Utilité :**
- Stockage temporaire pendant des traitements complexes
- Tests sans affecter les données permanentes
- Calculs intermédiaires, agrégations multi-étapes
- Sessions PL/pgSQL avec données isolées par utilisateur

> 💡 **À savoir** : les tables temporaires vivent dans un schéma `pg_temp_<N>` propre à chaque session. Elles ne sont **pas répliquées**, ne déclenchent **aucune réplication logique**, et ne sont pas visibles depuis les autres sessions.

### Table UNLOGGED

Table non journalisée (plus rapide en écriture, mais **vidée** après un crash et **non répliquée** en streaming) :

```sql
CREATE UNLOGGED TABLE cache_donnees (
    cle VARCHAR(100) PRIMARY KEY,
    valeur TEXT,
    expiration TIMESTAMPTZ
);
```

**Caractéristiques importantes :**

| Aspect | UNLOGGED | Table normale |
|--------|----------|---------------|
| WAL généré | ❌ Non | ✅ Oui |
| Performance INSERT/UPDATE | **2 à 4× plus rapide** | Référence |
| Survit à un crash propre (shutdown) | ✅ Oui | ✅ Oui |
| Survit à un crash brutal (kill -9, panne) | ❌ Non (vidée au redémarrage) | ✅ Oui |
| Répliquée par streaming replication | ❌ Non | ✅ Oui |
| Répliquée par logical replication | ❌ Non | ✅ Oui |
| Référençable par une FK **depuis une table permanente** | ❌ Non¹ | ✅ Oui |

> ¹ Une **table permanente ne peut pas avoir de clé étrangère vers une table `UNLOGGED`** (`ERROR: constraints on permanent tables may reference only permanent tables`) — logique, puisqu'une table `UNLOGGED` est vidée après un crash, ce qui casserait l'intégrité référentielle. L'inverse est permis : une table `UNLOGGED` peut référencer une table permanente.

**Cas d'usage** : caches applicatifs, tables de staging ETL, indexes inversés rebuilt-able, agrégats reconstructibles.

⚠️ **Attention** : les données sont perdues en cas de crash du serveur. Ne stockez jamais des données critiques en `UNLOGGED`.

```sql
-- Bascule possible dans les deux sens
ALTER TABLE cache_donnees SET LOGGED;    -- réécrit la table, génère du WAL  
ALTER TABLE ma_table SET UNLOGGED;       -- réécrit la table, abandonne le WAL  
```

---

## Nommage des Contraintes

Par défaut, PostgreSQL génère des noms automatiques pour les contraintes. Vous pouvez les nommer explicitement :

```sql
CREATE TABLE utilisateurs (
    id SERIAL,
    email VARCHAR(255),
    age INT,

    -- Nommer explicitement les contraintes
    CONSTRAINT pk_utilisateurs PRIMARY KEY (id),
    CONSTRAINT uk_utilisateurs_email UNIQUE (email),
    CONSTRAINT ck_utilisateurs_age CHECK (age >= 18)
);
```

**Avantages :**
- Messages d'erreur plus clairs
- Plus facile à référencer dans ALTER TABLE
- Meilleure documentation

**Convention de nommage (préfixes courants) :**
- `pk_` : Primary Key (ex. : `pk_utilisateurs`)
- `fk_` : Foreign Key (ex. : `fk_commandes_client`)
- `uk_` : Unique Key (ex. : `uk_utilisateurs_email`)
- `ck_` : Check constraint (ex. : `ck_produits_prix_positif`)
- `idx_` : Index (ex. : `idx_commandes_date`)
- `seq_` : Séquence (ex. : `seq_factures`)
- `trg_` : Trigger (ex. : `trg_audit_modifications`)
- `df_` : Contrainte de DEFAULT explicite (rare en PG)

---

## Commandes Utiles pour Inspecter les Tables

### Dans psql (console PostgreSQL)

```sql
-- Lister toutes les tables du schéma actuel
\dt

-- Lister toutes les tables (tous schémas)
\dt *.*

-- Voir la structure détaillée d'une table
\d nom_table
\d+ nom_table  -- Version détaillée avec plus d'infos

-- Voir uniquement les contraintes
\d+ nom_table

-- Lister les index
\di
```

### Avec des requêtes SQL

```sql
-- Lister toutes les tables de la database actuelle
SELECT table_schema, table_name  
FROM information_schema.tables  
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')  
ORDER BY table_schema, table_name;  

-- Voir les colonnes d'une table
SELECT
    column_name,
    data_type,
    character_maximum_length,
    is_nullable,
    column_default
FROM information_schema.columns  
WHERE table_name = 'nom_table';  

-- Voir les contraintes d'une table
SELECT
    constraint_name,
    constraint_type
FROM information_schema.table_constraints  
WHERE table_name = 'nom_table';  
```

---

## Modifier une Table Existante (Aperçu)

Bien que nous n'entrions pas dans les détails ici, sachez que vous pouvez modifier une table après sa création avec `ALTER TABLE` :

```sql
-- Ajouter une colonne
ALTER TABLE employes ADD COLUMN telephone VARCHAR(20);

-- Supprimer une colonne
ALTER TABLE employes DROP COLUMN telephone;

-- Modifier le type d'une colonne
ALTER TABLE employes ALTER COLUMN salaire TYPE DECIMAL(12, 2);

-- Ajouter une contrainte
ALTER TABLE employes ADD CONSTRAINT ck_salaire_positif CHECK (salaire > 0);

-- Renommer une colonne
ALTER TABLE employes RENAME COLUMN prenom TO first_name;

-- Renommer une table
ALTER TABLE employes RENAME TO employees;
```

Nous verrons ces commandes en détail dans une section ultérieure.

---

## Bonnes Pratiques de Création de Tables

### 1. Toujours définir une clé primaire

✅ **Recommandé :**
```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);
```

❌ **À éviter :**
```sql
CREATE TABLE clients (
    nom VARCHAR(100)
);
```

### 2. Utiliser des noms de colonnes explicites

✅ **Recommandé :**
```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    date_commande DATE,
    montant_total DECIMAL(10, 2),
    statut_livraison VARCHAR(50)
);
```

❌ **À éviter :**
```sql
CREATE TABLE commandes (
    i INT PRIMARY KEY,
    d DATE,
    m DECIMAL(10, 2),
    s VARCHAR(50)
);
```

### 3. Définir NOT NULL pour les colonnes obligatoires

✅ **Recommandé :**
```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    nom VARCHAR(100) NOT NULL
);
```

### 4. Utiliser des types appropriés

✅ **Recommandé :**
```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    prix DECIMAL(10, 2),  -- Montants financiers
    quantite INT,         -- Nombres entiers
    description TEXT      -- Texte long
);
```

❌ **À éviter :**
```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    prix VARCHAR(50),     -- Prix en texte ? Mauvaise idée !
    quantite VARCHAR(10), -- Quantité en texte ? Non !
    description VARCHAR(100) -- Trop court pour une description
);
```

### 5. Ajouter des valeurs par défaut sensées

✅ **Recommandé :**
```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300) NOT NULL,
    statut VARCHAR(20) DEFAULT 'brouillon',
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    nombre_vues INT DEFAULT 0
);
```

### 6. Utiliser CHECK pour valider les données

✅ **Recommandé :**
```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,
    prix DECIMAL(10, 2) CHECK (prix > 0),
    note DECIMAL(3, 2) CHECK (note BETWEEN 0 AND 5)
);
```

### 7. Documenter avec des commentaires

```sql
-- Créer la table
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

-- Ajouter des commentaires
COMMENT ON TABLE employes IS 'Table contenant tous les employés de l''entreprise';  
COMMENT ON COLUMN employes.id IS 'Identifiant unique de l''employé';  
COMMENT ON COLUMN employes.nom IS 'Nom complet de l''employé';  

-- Voir les commentaires
\d+ employes
```

### 8. Organiser avec des schémas

✅ **Recommandé :**
```sql
CREATE SCHEMA ressources_humaines;  
CREATE SCHEMA ventes;  
CREATE SCHEMA comptabilite;  

CREATE TABLE ressources_humaines.employes (...);  
CREATE TABLE ventes.commandes (...);  
CREATE TABLE comptabilite.factures (...);  
```

---

## Exemples de Tables Courantes

### Table Utilisateurs (Application Web)

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    mot_de_passe_hash VARCHAR(255) NOT NULL,
    prenom VARCHAR(100),
    nom VARCHAR(100),
    telephone VARCHAR(20),
    est_actif BOOLEAN DEFAULT TRUE,
    est_verifie BOOLEAN DEFAULT FALSE,
    date_inscription TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    derniere_connexion TIMESTAMP,

    CONSTRAINT ck_email_format CHECK (email LIKE '%@%')
);
```

### Table Produits (E-commerce)

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    reference VARCHAR(50) UNIQUE NOT NULL,
    nom VARCHAR(300) NOT NULL,
    description TEXT,
    prix_unitaire DECIMAL(10, 2) NOT NULL,
    quantite_stock INT NOT NULL DEFAULT 0 CHECK (quantite_stock >= 0),
    categorie VARCHAR(100),
    est_actif BOOLEAN DEFAULT TRUE,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    date_modification TIMESTAMP,

    CONSTRAINT ck_prix_positif CHECK (prix_unitaire > 0)
);
```

### Table Commandes

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    numero_commande VARCHAR(50) UNIQUE NOT NULL,
    client_id INT NOT NULL,  -- Sera une clé étrangère (vue plus tard)
    date_commande TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    date_livraison_prevue DATE,
    montant_total DECIMAL(12, 2) NOT NULL CHECK (montant_total >= 0),
    statut VARCHAR(50) DEFAULT 'en_attente'
        CHECK (statut IN ('en_attente', 'confirme', 'expedie', 'livre', 'annule')),
    adresse_livraison TEXT NOT NULL,
    notes TEXT
);
```

### Table Articles de Blog

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300) NOT NULL,
    slug VARCHAR(300) UNIQUE NOT NULL,
    contenu TEXT NOT NULL,
    resume VARCHAR(500),
    auteur_id INT NOT NULL,  -- Clé étrangère
    categorie VARCHAR(100),
    tags TEXT[],  -- Array de tags
    statut VARCHAR(20) DEFAULT 'brouillon'
        CHECK (statut IN ('brouillon', 'publie', 'archive')),
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    date_modification TIMESTAMP,
    date_publication TIMESTAMP,
    nombre_vues INT DEFAULT 0 CHECK (nombre_vues >= 0),

    CONSTRAINT ck_slug_format CHECK (slug ~ '^[a-z0-9-]+$')
);
```

---

## Supprimer une Table

### Syntaxe de base

```sql
-- Supprimer une table
DROP TABLE nom_table;

-- Supprimer si elle existe (pas d'erreur si elle n'existe pas)
DROP TABLE IF EXISTS nom_table;

-- Supprimer plusieurs tables
DROP TABLE table1, table2, table3;

-- Supprimer avec CASCADE (supprime aussi les objets dépendants)
DROP TABLE nom_table CASCADE;
```

### Exemples

```sql
-- Créer une table de test
CREATE TABLE test (id SERIAL PRIMARY KEY, nom VARCHAR(50));

-- La supprimer
DROP TABLE test;

-- Essayer de la supprimer à nouveau (erreur)
DROP TABLE test;
-- ERROR: table "test" does not exist

-- Version sans erreur
DROP TABLE IF EXISTS test;  -- Pas d'erreur, juste un NOTICE
```

⚠️ **Attention** : `DROP TABLE` est **irréversible**. Toutes les données sont perdues !

---

## Récapitulatif

### Structure de CREATE TABLE

```sql
CREATE TABLE [IF NOT EXISTS] [schema.]nom_table (
    colonne1 TYPE [NOT NULL] [DEFAULT valeur] [UNIQUE] [CHECK (condition)],
    colonne2 TYPE [NOT NULL] [DEFAULT valeur],
    ...
    PRIMARY KEY (colonne),
    UNIQUE (colonne),
    CHECK (condition sur plusieurs colonnes)
);
```

### Contraintes Essentielles

| Contrainte | Objectif | Exemple |
|------------|----------|---------|
| `NOT NULL` | Valeur obligatoire | `email VARCHAR(255) NOT NULL` |
| `DEFAULT` | Valeur par défaut | `statut VARCHAR(20) DEFAULT 'actif'` |
| `PRIMARY KEY` | Identifiant unique | `id SERIAL PRIMARY KEY` |
| `UNIQUE` | Valeur unique (NULL autorisé) | `email VARCHAR(255) UNIQUE` |
| `CHECK` | Validation personnalisée | `age INT CHECK (age >= 18)` |

### Commandes Utiles

```sql
-- Créer une table
CREATE TABLE nom_table (...);

-- Voir la structure
\d nom_table

-- Lister les tables
\dt

-- Supprimer une table
DROP TABLE nom_table;

-- Commenter une table
COMMENT ON TABLE nom_table IS 'Description';
```

---

## Conclusion

Créer une table dans PostgreSQL est une opération fondamentale qui nécessite de :

1. **Définir la structure** : Quelles colonnes avec quels types  
2. **Choisir les contraintes appropriées** : NOT NULL, UNIQUE, CHECK, etc.  
3. **Définir une clé primaire** : Toujours !  
4. **Utiliser des valeurs par défaut** : Pour simplifier les insertions  
5. **Suivre les conventions de nommage** : Pour la lisibilité et la maintenance

Une bonne conception de tables est cruciale pour :
- **L'intégrité des données** : Les contraintes empêchent les données invalides  
- **Les performances** : Les bons types de données optimisent le stockage et les requêtes  
- **La maintenabilité** : Une structure claire facilite les évolutions

Dans les prochaines sections, nous explorerons en détail les différents types de données disponibles dans PostgreSQL, ce qui vous permettra de créer des tables encore plus adaptées à vos besoins.

---


⏭️ [Les types de données fondamentaux](/04-objets-de-la-base-de-donnees/04-types-de-donnees-fondamentaux.md)
