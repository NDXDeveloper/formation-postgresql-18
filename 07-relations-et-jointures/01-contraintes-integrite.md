🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.1. Contraintes d'Intégrité : PK, FK, UNIQUE, CHECK, NOT NULL

## Introduction

Les **contraintes d'intégrité** sont des règles que l'on définit au niveau de la base de données pour garantir la **cohérence**, la **validité** et la **qualité** des données. Elles constituent une première ligne de défense contre les erreurs et les incohérences, bien avant que les données n'atteignent votre application.

En PostgreSQL, ces contraintes sont déclarées lors de la création ou de la modification d'une table, et le système de gestion de base de données (SGBD) se charge automatiquement de les faire respecter. Si une opération (INSERT, UPDATE) viole une contrainte, PostgreSQL rejette l'opération et retourne une erreur.

### Pourquoi utiliser des contraintes ?

1. **Garantie de cohérence** : Les données restent valides et logiques
2. **Prévention des erreurs** : Les bugs applicatifs sont détectés au niveau de la base
3. **Documentation implicite** : Les contraintes documentent les règles métier
4. **Performance** : Le planificateur peut optimiser certaines requêtes grâce aux contraintes
5. **Indépendance applicative** : Les règles s'appliquent quel que soit le client (web, mobile, API, etc.)

---

## 1. NOT NULL - Interdire les Valeurs Nulles

### Concept

La contrainte `NOT NULL` garantit qu'une colonne **ne peut jamais contenir de valeur NULL**. Elle est la plus simple et la plus fondamentale des contraintes.

### Rappel : Qu'est-ce que NULL ?

`NULL` représente l'**absence de valeur** ou une **valeur inconnue**. Ce n'est ni zéro, ni une chaîne vide, ni false. C'est un état spécial qui signifie "pas de données".

```sql
-- Exemple de NULL
SELECT NULL = NULL;  -- Retourne NULL (pas TRUE !)
SELECT NULL IS NULL; -- Retourne TRUE
```

### Syntaxe

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    telephone VARCHAR(20)  -- Peut être NULL (optionnel)
);
```

### Comportement

```sql
-- ✅ Insertion valide
INSERT INTO utilisateurs (nom, prenom, email)
VALUES ('Dupont', 'Marie', 'marie.dupont@example.com');

-- ❌ Échec : colonne 'nom' ne peut pas être NULL
INSERT INTO utilisateurs (prenom, email)
VALUES ('Jean', 'jean@example.com');
-- ERROR: null value in column "nom" violates not-null constraint

-- ✅ Insertion valide avec NULL sur 'telephone' (autorisé)
INSERT INTO utilisateurs (nom, prenom, email, telephone)
VALUES ('Martin', 'Paul', 'paul@example.com', NULL);
```

### Quand utiliser NOT NULL ?

- Sur les colonnes **essentielles** à l'identité de l'entité (nom, email, date de création, etc.)
- Sur les **clés étrangères** (sauf si la relation est optionnelle)
- Par défaut, **privilégiez NOT NULL** sauf si l'absence de valeur a un sens métier

### Ajouter NOT NULL à une colonne existante

```sql
-- Vérifier d'abord qu'il n'y a pas de NULL
SELECT COUNT(*) FROM utilisateurs WHERE telephone IS NULL;

-- Si des NULL existent, les remplacer ou les supprimer
UPDATE utilisateurs SET telephone = 'N/A' WHERE telephone IS NULL;

-- Puis ajouter la contrainte
ALTER TABLE utilisateurs
ALTER COLUMN telephone SET NOT NULL;
```

---

## 2. UNIQUE - Garantir l'Unicité

### Concept

La contrainte `UNIQUE` garantit que **toutes les valeurs d'une colonne (ou d'un groupe de colonnes) sont distinctes**. Aucune duplication n'est autorisée.

### Syntaxe : UNIQUE sur une colonne

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    nom VARCHAR(100) NOT NULL
);
```

### Syntaxe : UNIQUE sur plusieurs colonnes (contrainte composite)

```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    utilisateur_id INTEGER NOT NULL,
    salle_id INTEGER NOT NULL,
    date_reservation DATE NOT NULL,
    heure_debut TIME NOT NULL,
    -- Une salle ne peut être réservée qu'une fois par créneau horaire
    UNIQUE (salle_id, date_reservation, heure_debut)
);
```

### Comportement

```sql
-- ✅ Première insertion
INSERT INTO utilisateurs (email, nom)
VALUES ('alice@example.com', 'Alice');

-- ❌ Échec : email déjà existant
INSERT INTO utilisateurs (email, nom)
VALUES ('alice@example.com', 'Alice Bis');
-- ERROR: duplicate key value violates unique constraint
-- DETAIL: Key (email)=(alice@example.com) already exists.

-- ✅ Insertion valide avec un email différent
INSERT INTO utilisateurs (email, nom)
VALUES ('bob@example.com', 'Bob');
```

### Particularité : UNIQUE et NULL

**Point important** : En PostgreSQL, `NULL` est considéré comme **distinct de NULL**. Donc plusieurs lignes peuvent avoir `NULL` dans une colonne `UNIQUE`.

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    code_barre VARCHAR(50) UNIQUE  -- Peut être NULL
);

-- ✅ Ces deux insertions sont valides (NULL ≠ NULL)
INSERT INTO produits (nom, code_barre) VALUES ('Produit A', NULL);
INSERT INTO produits (nom, code_barre) VALUES ('Produit B', NULL);

-- ❌ Mais celle-ci échouera (duplication de '123456')
INSERT INTO produits (nom, code_barre) VALUES ('Produit C', '123456');
INSERT INTO produits (nom, code_barre) VALUES ('Produit D', '123456');
```

### UNIQUE vs UNIQUE avec NOT NULL

Pour garantir que chaque valeur est unique **et présente**, combinez `UNIQUE` avec `NOT NULL` :

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    -- Équivalent à : email VARCHAR(255) UNIQUE NOT NULL
);
```

### Nommer explicitement une contrainte UNIQUE

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    CONSTRAINT uq_utilisateurs_email UNIQUE (email)
);
```

**Avantage** : Le nom de la contrainte apparaît dans les messages d'erreur, facilitant le débogage.

---

## 3. PRIMARY KEY (PK) - La Clé Primaire

### Concept

La **clé primaire** (Primary Key) est une contrainte qui identifie de manière **unique** chaque ligne d'une table. C'est l'identifiant de référence absolu de l'entité.

Une clé primaire est **automatiquement** :
- `UNIQUE` : Pas de doublons
- `NOT NULL` : Jamais de valeur nulle
- **Indexée** : Un index B-Tree est créé automatiquement

### Règles fondamentales

1. **Une seule clé primaire par table** (mais elle peut être composite)
2. Une PK ne peut **jamais être NULL**
3. Une PK ne change **jamais** (stabilité dans le temps)

### Syntaxe : Clé primaire simple

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);
```

Ou de manière équivalente :

```sql
CREATE TABLE clients (
    id SERIAL,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    PRIMARY KEY (id)
);
```

### Syntaxe : Clé primaire composite

Une clé primaire peut être constituée de **plusieurs colonnes** (rare, mais parfois nécessaire) :

```sql
CREATE TABLE inscriptions_cours (
    etudiant_id INTEGER,
    cours_id INTEGER,
    date_inscription DATE NOT NULL,
    PRIMARY KEY (etudiant_id, cours_id)
);
```

Ici, la combinaison `(etudiant_id, cours_id)` est unique, mais chaque colonne individuellement peut avoir des doublons.

### Types de clés primaires

#### a) Clés Surrogates (Artificielles)

Ce sont des identifiants **générés automatiquement** par la base de données, sans signification métier.

```sql
-- Avec SERIAL (ancienne méthode)
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

-- Avec IDENTITY (méthode moderne, recommandée depuis PostgreSQL 10)
CREATE TABLE produits (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

-- Avec UUID (identifiant universel)
CREATE TABLE transactions (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    montant NUMERIC(10, 2) NOT NULL
);
```

**Avantages** :
- Simples et performants
- Indépendants de la logique métier
- Facilitent les jointures

**Inconvénients** :
- Pas de signification métier
- Potentiel de doublons métier si pas d'autres contraintes UNIQUE

#### b) Clés Naturelles

Ce sont des attributs ayant une **signification métier** (ex : numéro de sécurité sociale, SIRET, ISBN, etc.).

```sql
CREATE TABLE pays (
    code_iso CHAR(2) PRIMARY KEY,  -- 'FR', 'US', 'DE'...
    nom VARCHAR(100) NOT NULL
);
```

**Avantages** :
- Signification métier claire
- Pas de jointure nécessaire pour afficher l'identifiant

**Inconvénients** :
- Risque de modification dans le temps
- Peuvent être longs ou complexes
- Problèmes de performance si utilisés comme FK partout

### Bonnes pratiques

1. **Privilégiez les clés surrogates** (SERIAL, IDENTITY, UUID) pour la plupart des tables
2. **Ajoutez des contraintes UNIQUE** sur les identifiants métier (email, numéro client, etc.)
3. **Nommez explicitement vos PK** si vous voulez contrôler le nom de l'index

```sql
CREATE TABLE commandes (
    id INTEGER GENERATED ALWAYS AS IDENTITY,
    numero_commande VARCHAR(20) NOT NULL UNIQUE,
    client_id INTEGER NOT NULL,
    date_commande DATE NOT NULL,
    CONSTRAINT pk_commandes PRIMARY KEY (id)
);
```

---

## 4. FOREIGN KEY (FK) - Les Clés Étrangères

### Concept

Une **clé étrangère** (Foreign Key) établit une **relation référentielle** entre deux tables. Elle garantit que la valeur dans une colonne (ou un ensemble de colonnes) correspond à une valeur existante dans la clé primaire (ou UNIQUE) d'une autre table.

C'est le mécanisme fondamental pour modéliser les **relations entre entités**.

### Objectif

- **Intégrité référentielle** : Empêcher les "orphelins" (références vers des enregistrements inexistants)
- **Cohérence des données** : Garantir que les relations sont valides
- **Documentation** : Rendre explicites les liens entre tables

### Syntaxe de base

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL,
    date_commande DATE NOT NULL,
    CONSTRAINT fk_commandes_client
        FOREIGN KEY (client_id)
        REFERENCES clients(id)
);
```

### Syntaxe courte (inline)

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id),
    date_commande DATE NOT NULL
);
```

### Comportement par défaut

```sql
-- ✅ Insertion valide (client_id = 1 existe)
INSERT INTO clients (nom) VALUES ('Alice');  -- id=1
INSERT INTO commandes (client_id, date_commande)
VALUES (1, '2025-01-15');

-- ❌ Échec : client_id = 999 n'existe pas
INSERT INTO commandes (client_id, date_commande)
VALUES (999, '2025-01-15');
-- ERROR: insert or update on table "commandes" violates foreign key constraint
-- DETAIL: Key (client_id)=(999) is not present in table "clients".
```

### Actions ON DELETE et ON UPDATE

PostgreSQL permet de définir le comportement à adopter lorsqu'une ligne référencée est **modifiée** ou **supprimée**.

#### a) ON DELETE

Que se passe-t-il quand on supprime une ligne parente ?

##### NO ACTION (défaut)

Refuse la suppression si des lignes enfants existent.

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id) ON DELETE NO ACTION
);

-- ❌ Échec si des commandes existent pour ce client
DELETE FROM clients WHERE id = 1;
-- ERROR: update or delete on table "clients" violates foreign key constraint
```

##### RESTRICT

Identique à `NO ACTION`, mais vérifié **immédiatement** (pas de différé possible).

```sql
FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE RESTRICT
```

##### CASCADE

**Supprime en cascade** toutes les lignes enfants.

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE
);

-- ✅ Supprime le client ET toutes ses commandes
DELETE FROM clients WHERE id = 1;
```

**⚠️ Attention** : Utilisez `CASCADE` avec précaution. Il peut propager des suppressions massives non intentionnelles.

##### SET NULL

Met la clé étrangère à `NULL` dans les lignes enfants (la colonne doit autoriser NULL).

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER REFERENCES clients(id) ON DELETE SET NULL
    -- client_id peut être NULL
);

-- ✅ Le client est supprimé, client_id devient NULL dans les commandes
DELETE FROM clients WHERE id = 1;
```

##### SET DEFAULT

Met la clé étrangère à sa valeur par défaut.

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER DEFAULT 0 REFERENCES clients(id) ON DELETE SET DEFAULT
);
```

#### b) ON UPDATE

Même principe que `ON DELETE`, mais pour les **mises à jour** de la clé primaire référencée.

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
);

-- Si on modifie l'id du client, client_id est mis à jour dans les commandes
UPDATE clients SET id = 100 WHERE id = 1;
```

**Note** : Modifier une clé primaire est rare et généralement déconseillé. Préférez les clés surrogates immuables.

### Clés étrangères composites

```sql
CREATE TABLE inscriptions (
    etudiant_id INTEGER,
    cours_id INTEGER,
    PRIMARY KEY (etudiant_id, cours_id)
);

CREATE TABLE notes (
    id SERIAL PRIMARY KEY,
    etudiant_id INTEGER,
    cours_id INTEGER,
    note NUMERIC(4, 2),
    FOREIGN KEY (etudiant_id, cours_id)
        REFERENCES inscriptions(etudiant_id, cours_id)
);
```

### Clés étrangères auto-référencées

Une table peut avoir une FK vers elle-même (hiérarchies, arbres).

```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employes(id) ON DELETE SET NULL
);

-- Alice est manager
INSERT INTO employes (nom, manager_id) VALUES ('Alice', NULL);  -- id=1

-- Bob a Alice comme manager
INSERT INTO employes (nom, manager_id) VALUES ('Bob', 1);  -- id=2
```

### Bonnes pratiques

1. **Nommez vos contraintes FK** pour faciliter le débogage :
   ```sql
   CONSTRAINT fk_commandes_client FOREIGN KEY (client_id) REFERENCES clients(id)
   ```

2. **Indexez vos clés étrangères** (PostgreSQL ne le fait pas automatiquement) :
   ```sql
   CREATE INDEX idx_commandes_client_id ON commandes(client_id);
   ```
   **Pourquoi ?** Pour accélérer les jointures et les vérifications de contraintes.

3. **Choisissez ON DELETE avec soin** :
   - `CASCADE` : Pour les relations de composition forte (ex : commande → lignes_commande)
   - `RESTRICT` ou `NO ACTION` : Par défaut, pour éviter les suppressions accidentelles
   - `SET NULL` : Pour les relations optionnelles

4. **Évitez de modifier les PK référencées** : Utilisez `ON UPDATE CASCADE` uniquement si nécessaire.

---

## 5. CHECK - Contraintes de Validation Personnalisées

### Concept

La contrainte `CHECK` permet de définir des **règles de validation personnalisées** sur une ou plusieurs colonnes. C'est un prédicat booléen (une expression qui retourne `TRUE`, `FALSE` ou `NULL`).

### Syntaxe de base

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prix NUMERIC(10, 2) NOT NULL CHECK (prix > 0),
    stock INTEGER NOT NULL CHECK (stock >= 0)
);
```

### Comportement

```sql
-- ✅ Insertion valide
INSERT INTO produits (nom, prix, stock) VALUES ('Ordinateur', 999.99, 10);

-- ❌ Échec : prix négatif
INSERT INTO produits (nom, prix, stock) VALUES ('Souris', -10, 5);
-- ERROR: new row for relation "produits" violates check constraint "produits_prix_check"

-- ❌ Échec : stock négatif
INSERT INTO produits (nom, prix, stock) VALUES ('Clavier', 49.99, -1);
-- ERROR: new row for relation "produits" violates check constraint "produits_stock_check"
```

### Contraintes CHECK nommées

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prix NUMERIC(10, 2) NOT NULL,
    stock INTEGER NOT NULL,
    CONSTRAINT chk_prix_positif CHECK (prix > 0),
    CONSTRAINT chk_stock_non_negatif CHECK (stock >= 0)
);
```

### Contraintes CHECK multi-colonnes

```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    date_debut DATE NOT NULL,
    date_fin DATE NOT NULL,
    CONSTRAINT chk_dates_coherentes CHECK (date_fin >= date_debut)
);

-- ✅ Valide
INSERT INTO reservations (date_debut, date_fin)
VALUES ('2025-01-10', '2025-01-15');

-- ❌ Échec : date_fin avant date_debut
INSERT INTO reservations (date_debut, date_fin)
VALUES ('2025-01-20', '2025-01-15');
-- ERROR: new row violates check constraint "chk_dates_coherentes"
```

### Expressions complexes

```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    salaire NUMERIC(10, 2) NOT NULL,
    commission NUMERIC(10, 2),
    -- La commission ne peut pas dépasser 50% du salaire
    CONSTRAINT chk_commission_plafond CHECK (commission IS NULL OR commission <= salaire * 0.5)
);
```

### Contraintes CHECK avec énumérations

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    statut VARCHAR(20) NOT NULL,
    CONSTRAINT chk_statut_valide
        CHECK (statut IN ('en_attente', 'confirmee', 'expediee', 'livree', 'annulee'))
);

-- Alternative : Utiliser un ENUM (plus performant)
CREATE TYPE statut_commande AS ENUM ('en_attente', 'confirmee', 'expediee', 'livree', 'annulee');

CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    statut statut_commande NOT NULL DEFAULT 'en_attente'
);
```

### Contraintes CHECK avec expressions régulières

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    telephone VARCHAR(20),
    -- Vérification basique du format email
    CONSTRAINT chk_email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'),
    -- Vérification format téléphone français
    CONSTRAINT chk_telephone_format CHECK (telephone IS NULL OR telephone ~ '^0[1-9][0-9]{8}$')
);
```

### Limitations importantes

1. **Pas de sous-requêtes** : Une contrainte CHECK ne peut pas interroger d'autres tables
   ```sql
   -- ❌ Interdit
   CHECK (client_id IN (SELECT id FROM clients))
   ```

2. **Pas d'appels à des fonctions volatiles** : Les fonctions comme `CURRENT_DATE`, `NOW()` sont interdites
   ```sql
   -- ❌ Interdit
   CHECK (date_naissance < CURRENT_DATE)
   ```

3. **NULL est toujours valide** : Si l'expression retourne `NULL`, la contrainte est considérée comme satisfaite
   ```sql
   CREATE TABLE test (val INTEGER CHECK (val > 0));
   INSERT INTO test VALUES (NULL);  -- ✅ Accepté
   ```

### Bonnes pratiques

1. **Nommez vos contraintes** pour faciliter le diagnostic :
   ```sql
   CONSTRAINT chk_prix_positif CHECK (prix > 0)
   ```

2. **Utilisez CHECK pour les règles métier simples** :
   - Plages de valeurs (âge entre 18 et 120)
   - Formats de données (code postal, téléphone)
   - Relations entre colonnes de la même ligne

3. **Privilégiez les ENUMs pour les listes de valeurs fixes** (plus performant)

4. **Documentez les contraintes complexes** :
   ```sql
   -- La remise ne peut dépasser 30% du prix
   CONSTRAINT chk_remise_maximale CHECK (remise <= prix * 0.30)
   ```

5. **Ne remplacez pas les validations applicatives** : CHECK complète mais ne remplace pas la validation côté application

---

## Comparaison et Choix des Contraintes

| Contrainte | Objectif | Portée | Exemple d'usage |
|-----------|----------|--------|-----------------|
| **NOT NULL** | Interdire les valeurs nulles | Colonne | Champs obligatoires (nom, email, date) |
| **UNIQUE** | Garantir l'unicité | Colonne(s) | Email, numéro de client, SIRET |
| **PRIMARY KEY** | Identifiant unique de la ligne | Colonne(s) | Clé d'identification (id) |
| **FOREIGN KEY** | Relation entre tables | Colonne(s) | Lien client → commandes |
| **CHECK** | Validation personnalisée | Ligne | Prix > 0, date_fin >= date_debut |

---

## Gestion des Contraintes sur des Tables Existantes

### Ajouter une contrainte

```sql
-- NOT NULL
ALTER TABLE utilisateurs ALTER COLUMN email SET NOT NULL;

-- UNIQUE
ALTER TABLE utilisateurs ADD CONSTRAINT uq_email UNIQUE (email);

-- PRIMARY KEY (seulement si pas de PK existante)
ALTER TABLE utilisateurs ADD PRIMARY KEY (id);

-- FOREIGN KEY
ALTER TABLE commandes
ADD CONSTRAINT fk_client
FOREIGN KEY (client_id) REFERENCES clients(id);

-- CHECK
ALTER TABLE produits
ADD CONSTRAINT chk_prix_positif CHECK (prix > 0);
```

### Supprimer une contrainte

```sql
-- Supprimer une contrainte nommée
ALTER TABLE produits DROP CONSTRAINT chk_prix_positif;

-- Supprimer NOT NULL
ALTER TABLE utilisateurs ALTER COLUMN telephone DROP NOT NULL;
```

### Désactiver temporairement une contrainte

```sql
-- Désactiver (attention : dangereux, à utiliser avec précaution)
ALTER TABLE commandes DISABLE TRIGGER ALL;

-- Réactiver
ALTER TABLE commandes ENABLE TRIGGER ALL;
```

**⚠️ Important** : PostgreSQL ne permet pas de désactiver les contraintes CHECK et FK directement. Il faut les supprimer puis les recréer. Utilisez les triggers pour les désactivations temporaires.

---

## Visualiser les Contraintes

### Lister les contraintes d'une table

```sql
-- Dans psql
\d nom_table

-- Requête SQL
SELECT
    conname AS nom_contrainte,
    contype AS type,
    CASE contype
        WHEN 'p' THEN 'PRIMARY KEY'
        WHEN 'f' THEN 'FOREIGN KEY'
        WHEN 'u' THEN 'UNIQUE'
        WHEN 'c' THEN 'CHECK'
    END AS type_contrainte
FROM pg_constraint
WHERE conrelid = 'nom_table'::regclass;
```

### Types de contraintes (contype)

- `p` : PRIMARY KEY
- `f` : FOREIGN KEY
- `u` : UNIQUE
- `c` : CHECK
- `t` : TRIGGER constraint
- `x` : EXCLUSION constraint

---

## Ordre de Création et Dépendances

### Créer les tables dans le bon ordre

```sql
-- 1. Table parente (sans FK)
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

-- 2. Table enfant (avec FK vers clients)
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id),
    date_commande DATE NOT NULL
);

-- 3. Table petite-fille (avec FK vers commandes)
CREATE TABLE lignes_commande (
    id SERIAL PRIMARY KEY,
    commande_id INTEGER NOT NULL REFERENCES commandes(id) ON DELETE CASCADE,
    produit VARCHAR(100) NOT NULL,
    quantite INTEGER NOT NULL CHECK (quantite > 0)
);
```

### Gérer les dépendances circulaires

Si deux tables se référencent mutuellement, créez-les en deux étapes :

```sql
-- Étape 1 : Créer les tables sans FK
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    departement_id INTEGER  -- Sans FK pour l'instant
);

CREATE TABLE departements (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    chef_id INTEGER  -- Sans FK pour l'instant
);

-- Étape 2 : Ajouter les FK
ALTER TABLE employes
ADD CONSTRAINT fk_employes_departement
FOREIGN KEY (departement_id) REFERENCES departements(id);

ALTER TABLE departements
ADD CONSTRAINT fk_departements_chef
FOREIGN KEY (chef_id) REFERENCES employes(id);
```

---

## Erreurs Courantes et Solutions

### Erreur 1 : Violation de NOT NULL

```
ERROR: null value in column "email" violates not-null constraint
```

**Solution** : Fournir une valeur pour la colonne ou la rendre nullable si l'absence de valeur a un sens métier.

### Erreur 2 : Violation de UNIQUE

```
ERROR: duplicate key value violates unique constraint "utilisateurs_email_key"
DETAIL: Key (email)=(alice@example.com) already exists.
```

**Solutions** :
- Vérifier si l'enregistrement existe avant d'insérer
- Utiliser `ON CONFLICT` pour gérer les doublons
- Modifier la valeur pour qu'elle soit unique

### Erreur 3 : Violation de FOREIGN KEY

```
ERROR: insert or update on table "commandes" violates foreign key constraint "fk_client"
DETAIL: Key (client_id)=(999) is not present in table "clients".
```

**Solutions** :
- Vérifier que la valeur référencée existe
- Créer d'abord l'enregistrement parent
- Utiliser une transaction pour insérer parent et enfant ensemble

### Erreur 4 : Violation de CHECK

```
ERROR: new row for relation "produits" violates check constraint "chk_prix_positif"
DETAIL: Failing row contains (10, Souris, -5.00, 20).
```

**Solution** : Corriger la valeur pour qu'elle respecte la règle métier.

---

## Résumé et Bonnes Pratiques

### Principes Généraux

1. **Utilisez les contraintes généreusement** : Elles sont votre première ligne de défense contre les données invalides

2. **Nommez vos contraintes** : Facilitez le diagnostic et la maintenance
   ```sql
   CONSTRAINT fk_commandes_client FOREIGN KEY (client_id) REFERENCES clients(id)
   ```

3. **Indexez les Foreign Keys** : PostgreSQL ne le fait pas automatiquement
   ```sql
   CREATE INDEX idx_commandes_client_id ON commandes(client_id);
   ```

4. **Privilégiez NOT NULL par défaut** : NULL devrait être l'exception, pas la règle

5. **Documentez les contraintes complexes** : Un commentaire peut sauver des heures de débogage
   ```sql
   COMMENT ON CONSTRAINT chk_dates_coherentes ON reservations
   IS 'La date de fin doit être postérieure ou égale à la date de début';
   ```

6. **Testez vos contraintes** : Écrivez des tests unitaires pour vérifier qu'elles fonctionnent comme prévu

### Checklist de Conception

Lors de la création d'une table, posez-vous ces questions :

- ✅ Quelle est la clé primaire ? (Préférez SERIAL ou IDENTITY)
- ✅ Quelles colonnes sont obligatoires ? (NOT NULL)
- ✅ Quelles colonnes doivent être uniques ? (UNIQUE)
- ✅ Quelles sont les relations avec d'autres tables ? (FOREIGN KEY)
- ✅ Quelles validations métier puis-je encoder ? (CHECK)
- ✅ Ai-je besoin d'index sur les FK pour les performances ?

### Exemple Complet

Voici un exemple réunissant toutes les contraintes :

```sql
CREATE TABLE commandes (
    -- Clé primaire avec génération automatique
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    -- Numéro de commande unique (identifiant métier)
    numero_commande VARCHAR(20) NOT NULL,

    -- Clé étrangère vers clients avec cascade à la suppression
    client_id INTEGER NOT NULL,

    -- Dates avec validation
    date_commande DATE NOT NULL DEFAULT CURRENT_DATE,
    date_livraison_prevue DATE,

    -- Montant avec validation
    montant_total NUMERIC(10, 2) NOT NULL,

    -- Statut avec énumération
    statut VARCHAR(20) NOT NULL DEFAULT 'en_attente',

    -- Contraintes
    CONSTRAINT uq_numero_commande UNIQUE (numero_commande),
    CONSTRAINT fk_commandes_client
        FOREIGN KEY (client_id)
        REFERENCES clients(id)
        ON DELETE RESTRICT,
    CONSTRAINT chk_montant_positif
        CHECK (montant_total > 0),
    CONSTRAINT chk_statut_valide
        CHECK (statut IN ('en_attente', 'confirmee', 'expediee', 'livree', 'annulee')),
    CONSTRAINT chk_date_livraison_coherente
        CHECK (date_livraison_prevue IS NULL OR date_livraison_prevue >= date_commande)
);

-- Index sur la FK pour les performances
CREATE INDEX idx_commandes_client_id ON commandes(client_id);

-- Index sur le statut pour les requêtes fréquentes
CREATE INDEX idx_commandes_statut ON commandes(statut);
```

---

## Conclusion

Les **contraintes d'intégrité** sont essentielles pour bâtir une base de données robuste et fiable. Elles transforment votre SGBD en un gardien actif de la qualité de vos données.

### Points clés à retenir

1. **NOT NULL** : Rendez vos colonnes non nullables par défaut
2. **UNIQUE** : Évitez les doublons sur les identifiants métier
3. **PRIMARY KEY** : Chaque table doit avoir un identifiant unique
4. **FOREIGN KEY** : Garantissez l'intégrité référentielle entre tables
5. **CHECK** : Encodez les règles métier au niveau de la base

En combinant ces contraintes intelligemment, vous créez un **système de données auto-documenté** où les règles métier sont explicites, appliquées automatiquement, et indépendantes de votre code applicatif.

**Prochain sujet** : Dans la section suivante (7.2), nous découvrirons les **nouveautés PostgreSQL 18** concernant les contraintes temporelles (Temporal Constraints), une avancée majeure pour gérer les données historiques et les périodes de validité.

⏭️ [Nouveauté PG 18 : Contraintes temporelles (Temporal Constraints)](/07-relations-et-jointures/02-contraintes-temporelles.md)
