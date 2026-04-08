🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.7. Gestion des contraintes différées (DEFERRABLE, INITIALLY DEFERRED)

## Introduction

Les **contraintes différées** (deferred constraints) sont une fonctionnalité puissante de PostgreSQL qui permet de reporter la vérification de certaines contraintes d'intégrité jusqu'à la **fin de la transaction**, plutôt que de les vérifier immédiatement après chaque instruction SQL.

### Analogie Simple

Imaginez un comptable qui vérifie les comptes d'une entreprise :

**Vérification immédiate (comportement par défaut) :**
```
Transaction : Transfert de 1000 € du compte A vers le compte B
    ↓
1. Débiter A de 1000 €
   → Vérification : A a-t-il un solde positif ? ✅
    ↓
2. Créditer B de 1000 €
   → Vérification : B a-t-il un solde positif ? ✅
    ↓
Fin de transaction
```

**Vérification différée :**
```
Transaction : Transfert de 1000 € du compte A vers le compte B
    ↓
1. Débiter A de 1000 €
   → Pas de vérification immédiate (temporairement négatif OK)
    ↓
2. Créditer B de 1000 €
   → Pas de vérification immédiate
    ↓
Fin de transaction
   → Vérification globale : Tout est cohérent ? ✅
```

Les contraintes différées permettent à votre base de données d'être **temporairement incohérente** pendant une transaction, à condition qu'elle soit **cohérente à la fin**.

---

## Contexte : Vérification Immédiate (Comportement par Défaut)

### Comment PostgreSQL Vérifie les Contraintes

Par défaut, PostgreSQL vérifie toutes les contraintes d'intégrité **immédiatement** après chaque instruction SQL (INSERT, UPDATE, DELETE).

```sql
-- Créer une table avec contraintes
CREATE TABLE comptes (
    compte_id SERIAL PRIMARY KEY,
    solde NUMERIC(10,2) CHECK (solde >= 0)  -- Contrainte CHECK
);

-- Transaction simple
BEGIN;
    INSERT INTO comptes (solde) VALUES (1000);  -- ✅ OK : solde positif
    UPDATE comptes SET solde = -100 WHERE compte_id = 1;  -- ❌ ERREUR immédiate !
COMMIT;
-- ERROR: new row for relation "comptes" violates check constraint "comptes_solde_check"
```

L'erreur se produit **immédiatement** lors de l'UPDATE, avant la fin de la transaction.

### Problèmes avec la Vérification Immédiate

#### Problème 1 : Modifications Interdépendantes

```sql
CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    manager_id INTEGER REFERENCES employes(employe_id)
);

-- Insérer Alice (pas de manager)
INSERT INTO employes (nom, manager_id) VALUES ('Alice', NULL);  -- ✅ OK

-- Bob veut Alice comme manager, et Alice veut Bob comme manager (temporairement)
BEGIN;
    -- Insérer Bob avec Alice comme manager
    INSERT INTO employes (nom, manager_id) VALUES ('Bob', 1);  -- ✅ OK (Alice existe)

    -- Mettre Alice avec Bob comme manager
    UPDATE employes SET manager_id = 2 WHERE employe_id = 1;  -- ✅ OK (Bob existe maintenant)
COMMIT;
```

Cela fonctionne car les opérations sont séquentielles. Mais imaginez un cas plus complexe :

```sql
-- Échanger les managers de deux employés
BEGIN;
    UPDATE employes SET manager_id = 2 WHERE employe_id = 1;  -- Alice → Bob
    UPDATE employes SET manager_id = 1 WHERE employe_id = 2;  -- Bob → Alice
COMMIT;
```

Cela fonctionne aussi, mais dans certains cas (clés étrangères circulaires, par exemple), c'est impossible avec une vérification immédiate.

#### Problème 2 : Ordre des Opérations Contraint

```sql
CREATE TABLE departements (
    dept_id SERIAL PRIMARY KEY,
    nom_dept VARCHAR(100),
    chef_id INTEGER REFERENCES employes(employe_id)
);

CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    dept_id INTEGER REFERENCES departements(dept_id)
);
```

**Problème :** Dépendance circulaire !
- Pour créer un département, il faut un chef (employé)
- Pour créer un employé, il faut un département

**Sans contraintes différées :**
```sql
BEGIN;
    INSERT INTO departements (nom_dept, chef_id) VALUES ('IT', 1);  -- ❌ Erreur : employé 1 n'existe pas
    INSERT INTO employes (nom, dept_id) VALUES ('Alice', 1);
COMMIT;
```

Impossible de satisfaire les deux contraintes en même temps !

---

## Les Contraintes Différées : Solution

### Concept

Une contrainte **DEFERRABLE** (différable) peut avoir sa vérification **reportée** jusqu'à la fin de la transaction.

### Syntaxe de Base

```sql
-- Lors de la création de table
CREATE TABLE nom_table (
    colonne TYPE,
    CONSTRAINT nom_contrainte CHECK (condition) DEFERRABLE INITIALLY DEFERRED
);

-- Lors de l'ajout d'une contrainte
ALTER TABLE nom_table  
ADD CONSTRAINT nom_contrainte CHECK (condition) DEFERRABLE INITIALLY DEFERRED;  
```

**Deux mots-clés importants :**
1. **DEFERRABLE** : La contrainte *peut* être différée (contrôlable)  
2. **INITIALLY DEFERRED** : La contrainte *est* différée par défaut

---

## Les Trois États d'une Contrainte

### 1. NOT DEFERRABLE (Par Défaut)

```sql
CREATE TABLE comptes (
    compte_id SERIAL PRIMARY KEY,
    solde NUMERIC(10,2) CHECK (solde >= 0) NOT DEFERRABLE
);
```

**Comportement :**
- Vérification **toujours immédiate**
- Impossible de différer, même avec SET CONSTRAINTS

**Usage :** C'est le comportement par défaut de PostgreSQL. Contraintes strictes qui ne peuvent jamais être violées temporairement.

### 2. DEFERRABLE INITIALLY IMMEDIATE

```sql
CREATE TABLE comptes (
    compte_id SERIAL PRIMARY KEY,
    solde NUMERIC(10,2) CHECK (solde >= 0) DEFERRABLE INITIALLY IMMEDIATE
);
```

**Comportement :**
- Vérification **immédiate par défaut**
- Mais peut être **différée sur demande** avec `SET CONSTRAINTS ... DEFERRED`

**Usage :** Contraintes normalement strictes, mais avec flexibilité occasionnelle.

### 3. DEFERRABLE INITIALLY DEFERRED

```sql
CREATE TABLE comptes (
    compte_id SERIAL PRIMARY KEY,
    solde NUMERIC(10,2) CHECK (solde >= 0) DEFERRABLE INITIALLY DEFERRED
);
```

**Comportement :**
- Vérification **différée par défaut**
- Peut être rendue **immédiate sur demande** avec `SET CONSTRAINTS ... IMMEDIATE`

**Usage :** Contraintes flexibles, vérifiées en fin de transaction sauf demande contraire.

### Tableau Comparatif

| État | Vérification par défaut | Peut être modifié ? |
|------|------------------------|---------------------|
| **NOT DEFERRABLE** | Immédiate | ❌ Non (toujours immédiat) |
| **DEFERRABLE INITIALLY IMMEDIATE** | Immédiate | ✅ Oui (peut être différé) |
| **DEFERRABLE INITIALLY DEFERRED** | Différée | ✅ Oui (peut être immédiat) |

---

## Utilisation : SET CONSTRAINTS

### Contrôler Dynamiquement la Vérification

La commande `SET CONSTRAINTS` permet de modifier le comportement des contraintes DEFERRABLE **dans la transaction courante**.

```sql
-- Différer toutes les contraintes DEFERRABLE
SET CONSTRAINTS ALL DEFERRED;

-- Différer une contrainte spécifique
SET CONSTRAINTS nom_contrainte DEFERRED;

-- Rendre immédiate une contrainte spécifique
SET CONSTRAINTS nom_contrainte IMMEDIATE;

-- Rendre toutes les contraintes immédiates
SET CONSTRAINTS ALL IMMEDIATE;
```

### Exemple Complet

```sql
-- Créer une table avec contrainte DEFERRABLE
CREATE TABLE comptes (
    compte_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    solde NUMERIC(10,2),
    CONSTRAINT check_solde_positif CHECK (solde >= 0) DEFERRABLE INITIALLY IMMEDIATE
);

-- Insérer des données
INSERT INTO comptes (nom, solde) VALUES ('Alice', 1000), ('Bob', 500);

-- Transaction avec vérification immédiate (par défaut)
BEGIN;
    UPDATE comptes SET solde = -100 WHERE nom = 'Alice';  -- ❌ ERREUR immédiate
ROLLBACK;

-- Transaction avec vérification différée
BEGIN;
    SET CONSTRAINTS check_solde_positif DEFERRED;

    UPDATE comptes SET solde = -100 WHERE nom = 'Alice';  -- ✅ OK temporairement
    UPDATE comptes SET solde = 600 WHERE nom = 'Bob';     -- Compensation
    UPDATE comptes SET solde = 200 WHERE nom = 'Alice';   -- Correction

COMMIT;  -- ✅ Vérification ici : tout est valide
```

---

## Cas d'Usage Pratiques

### Cas 1 : Références Circulaires

**Problème :** Table avec auto-référence circulaire.

```sql
-- Table employés avec hiérarchie
CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    manager_id INTEGER,
    CONSTRAINT fk_manager FOREIGN KEY (manager_id)
        REFERENCES employes(employe_id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Échanger les managers de deux employés
BEGIN;
    -- Alice (ID 1) devient manager de Bob (ID 2)
    -- Bob (ID 2) devient manager de Alice (ID 1)

    UPDATE employes SET manager_id = 2 WHERE employe_id = 1;  -- OK temporairement
    UPDATE employes SET manager_id = 1 WHERE employe_id = 2;  -- OK temporairement

COMMIT;  -- Vérification finale : les deux employés existent, OK
```

**Sans DEFERRABLE :** Impossible, car la première UPDATE violerait temporairement la contrainte.

### Cas 2 : Dépendances Bidirectionnelles

**Problème :** Deux tables qui se référencent mutuellement.

```sql
-- Départements et employés avec dépendances croisées
CREATE TABLE departements (
    dept_id SERIAL PRIMARY KEY,
    nom_dept VARCHAR(100),
    chef_id INTEGER  -- Référence vers employes (ajoutée après)
);

CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    dept_id INTEGER,
    CONSTRAINT fk_dept FOREIGN KEY (dept_id)
        REFERENCES departements(dept_id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Ajouter la contrainte FK après coup (sinon erreur circulaire)
ALTER TABLE departements  
ADD CONSTRAINT fk_chef FOREIGN KEY (chef_id)  
    REFERENCES employes(employe_id)
    DEFERRABLE INITIALLY DEFERRED;

-- Créer un département avec son chef
BEGIN;
    -- Insérer le département sans chef
    INSERT INTO departements (nom_dept, chef_id) VALUES ('IT', NULL);

    -- Insérer l'employé avec le département
    INSERT INTO employes (nom, dept_id) VALUES ('Alice', 1);

    -- Définir Alice comme chef du département IT
    UPDATE departements SET chef_id = 1 WHERE dept_id = 1;

COMMIT;  -- Vérification : tout est cohérent
```

**Alternative avec SET CONSTRAINTS :**
```sql
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    -- Insérer dans l'ordre qu'on veut
    INSERT INTO departements (nom_dept, chef_id) VALUES ('IT', 1);  -- Alice n'existe pas encore
    INSERT INTO employes (nom, dept_id) VALUES ('Alice', 1);        -- OK

COMMIT;
```

### Cas 3 : Contraintes UNIQUE Temporairement Violées

```sql
-- Table avec contrainte UNIQUE différable
CREATE TABLE codes_produits (
    code_id SERIAL PRIMARY KEY,
    ancien_code VARCHAR(50),
    nouveau_code VARCHAR(50),
    CONSTRAINT unique_nouveau_code UNIQUE (nouveau_code)
        DEFERRABLE INITIALLY DEFERRED
);

-- Données initiales
INSERT INTO codes_produits (ancien_code, nouveau_code) VALUES
    ('OLD001', 'NEW001'),
    ('OLD002', 'NEW002'),
    ('OLD003', 'NEW003');

-- Réorganiser les codes (rotation circulaire)
BEGIN;
    SET CONSTRAINTS unique_nouveau_code DEFERRED;

    UPDATE codes_produits SET nouveau_code = 'TEMP001' WHERE ancien_code = 'OLD001';
    UPDATE codes_produits SET nouveau_code = 'NEW001' WHERE ancien_code = 'OLD002';
    UPDATE codes_produits SET nouveau_code = 'NEW002' WHERE ancien_code = 'OLD003';
    UPDATE codes_produits SET nouveau_code = 'NEW003' WHERE ancien_code = 'OLD001';

COMMIT;  -- Vérification : tous les codes sont uniques
```

**Sans DEFERRABLE :** La première UPDATE créerait un doublon temporaire avec 'NEW001', erreur immédiate.

### Cas 4 : Import de Données avec Dépendances

```sql
-- Tables pour import de données
CREATE TABLE categories (
    categorie_id INTEGER PRIMARY KEY,
    nom_categorie VARCHAR(100)
);

CREATE TABLE produits (
    produit_id INTEGER PRIMARY KEY,
    nom_produit VARCHAR(255),
    categorie_id INTEGER,
    CONSTRAINT fk_categorie FOREIGN KEY (categorie_id)
        REFERENCES categories(categorie_id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Import sans ordre particulier (données d'un fichier CSV)
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    -- Insérer les produits d'abord (catégories pas encore là)
    INSERT INTO produits (produit_id, nom_produit, categorie_id) VALUES
        (1, 'Ordinateur', 10),
        (2, 'Souris', 10),
        (3, 'Livre SQL', 20);

    -- Insérer les catégories après
    INSERT INTO categories (categorie_id, nom_categorie) VALUES
        (10, 'Informatique'),
        (20, 'Livres');

COMMIT;  -- Vérification : toutes les FK sont satisfaites
```

### Cas 5 : Transactions Complexes de Migration

```sql
-- Migration de données entre deux structures
CREATE TABLE ancien_schema (
    id SERIAL PRIMARY KEY,
    data TEXT,
    ref_id INTEGER
);

CREATE TABLE nouveau_schema (
    id SERIAL PRIMARY KEY,
    info TEXT,
    lien_id INTEGER,
    CONSTRAINT fk_lien FOREIGN KEY (lien_id)
        REFERENCES nouveau_schema(id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Migration avec auto-références
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    -- Copier les données avec références temporairement incorrectes
    INSERT INTO nouveau_schema (id, info, lien_id)
    SELECT id, data, ref_id FROM ancien_schema;

    -- Ajuster les références
    UPDATE nouveau_schema n1
    SET lien_id = n2.id
    FROM nouveau_schema n2
    WHERE n1.lien_id = (SELECT id FROM ancien_schema WHERE ref_id = n2.id);

COMMIT;
```

---

## Types de Contraintes Supportées

### Contraintes Différables

| Type de Contrainte | DEFERRABLE Supporté ? | Notes |
|--------------------|----------------------|-------|
| **PRIMARY KEY** | ✅ Oui | Peut être différé |
| **UNIQUE** | ✅ Oui | Peut être différé |
| **FOREIGN KEY** | ✅ Oui | Le plus couramment différé |
| **CHECK** | ✅ Oui | Peut être différé |
| **EXCLUDE** | ✅ Oui | PostgreSQL uniquement |

### Contraintes Non-Différables

| Type | Pourquoi ? |
|------|-----------|
| **NOT NULL** | ❌ Ne peut pas être différé (contrainte de colonne, pas de table) |

**Note importante :** NOT NULL n'est **jamais** différable car c'est une propriété de colonne, pas une contrainte de table.

---

## Syntaxe Complète

### Lors de la Création de Table

```sql
-- Clé primaire différable
CREATE TABLE exemple1 (
    id INTEGER PRIMARY KEY DEFERRABLE INITIALLY DEFERRED
);

-- Clé étrangère différable
CREATE TABLE exemple2 (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER REFERENCES exemple2(id) DEFERRABLE INITIALLY IMMEDIATE
);

-- Contrainte UNIQUE différable
CREATE TABLE exemple3 (
    id INTEGER PRIMARY KEY,
    code VARCHAR(50) UNIQUE DEFERRABLE INITIALLY DEFERRED
);

-- Contrainte CHECK différable
CREATE TABLE exemple4 (
    id INTEGER PRIMARY KEY,
    solde NUMERIC(10,2),
    CONSTRAINT check_solde CHECK (solde >= 0) DEFERRABLE INITIALLY IMMEDIATE
);
```

### Lors de l'Ajout de Contraintes

```sql
-- Ajouter FK différable
ALTER TABLE produits  
ADD CONSTRAINT fk_categorie FOREIGN KEY (categorie_id)  
    REFERENCES categories(categorie_id)
    DEFERRABLE INITIALLY DEFERRED;

-- Ajouter contrainte CHECK différable
ALTER TABLE comptes  
ADD CONSTRAINT check_solde_positif CHECK (solde >= 0)  
    DEFERRABLE INITIALLY IMMEDIATE;

-- Ajouter contrainte UNIQUE différable
ALTER TABLE codes  
ADD CONSTRAINT unique_code UNIQUE (code)  
    DEFERRABLE INITIALLY DEFERRED;
```

### Modifier une Contrainte Existante

```sql
-- ❌ Impossible : on ne peut pas modifier directement une contrainte

-- ✅ Solution : Supprimer et recréer
ALTER TABLE produits DROP CONSTRAINT fk_categorie;

ALTER TABLE produits  
ADD CONSTRAINT fk_categorie FOREIGN KEY (categorie_id)  
    REFERENCES categories(categorie_id)
    DEFERRABLE INITIALLY DEFERRED;
```

---

## Comportement dans les Transactions

### Portée de SET CONSTRAINTS

```sql
-- SET CONSTRAINTS est limité à la transaction courante
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;
    -- Contraintes différées dans cette transaction
COMMIT;

-- Transaction suivante : retour au comportement par défaut (INITIALLY)
BEGIN;
    -- Contraintes selon leur définition INITIALLY
COMMIT;
```

### Vérification à la Fin de Transaction

```sql
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    -- Opérations qui violent temporairement des contraintes
    INSERT INTO employes (nom, manager_id) VALUES ('Alice', 999);  -- Manager inexistant

    -- Si on ne corrige pas avant COMMIT...
COMMIT;  -- ❌ ERREUR : violation de contrainte FK détectée ici
```

**Message d'erreur typique :**
```
ERROR: insert or update on table "employes" violates foreign key constraint "fk_manager"  
DETAIL: Key (manager_id)=(999) is not present in table "employes".  
```

### ROLLBACK et Contraintes Différées

```sql
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    -- Violations temporaires
    UPDATE comptes SET solde = -500 WHERE compte_id = 1;

    -- Décision : annuler la transaction
ROLLBACK;  -- Aucune vérification de contraintes (transaction annulée)
```

---

## Différences avec les Autres SGBD

### PostgreSQL vs MySQL

| Fonctionnalité | PostgreSQL | MySQL |
|----------------|------------|-------|
| **Contraintes différées** | ✅ Support complet | ❌ Pas de support |
| **SET CONSTRAINTS** | ✅ Oui | ❌ Non |
| **FK DEFERRABLE** | ✅ Oui | ❌ Non |

MySQL vérifie **toujours** les contraintes immédiatement, sans possibilité de différer.

### PostgreSQL vs Oracle

| Fonctionnalité | PostgreSQL | Oracle |
|----------------|------------|--------|
| **Contraintes différées** | ✅ Support complet | ✅ Support complet |
| **SET CONSTRAINTS** | ✅ Oui | ✅ Oui (syntaxe identique) |
| **Comportement par défaut** | NOT DEFERRABLE | DEFERRABLE INITIALLY IMMEDIATE |

Oracle et PostgreSQL ont des comportements similaires, mais Oracle rend plus de contraintes DEFERRABLE par défaut.

### PostgreSQL vs SQL Server

| Fonctionnalité | PostgreSQL | SQL Server |
|----------------|------------|------------|
| **Contraintes différées** | ✅ Support complet | ⚠️ Support partiel |
| **SET CONSTRAINTS** | ✅ Oui | ❌ Non (utilise CHECK_CONSTRAINTS) |

SQL Server a une approche différente avec `SET CHECK_CONSTRAINTS ON/OFF`, qui désactive complètement les contraintes.

---

## Performance et Considérations

### Impact sur les Performances

**Contraintes immédiates (NOT DEFERRABLE) :**
- ✅ Vérification rapide après chaque instruction  
- ✅ Erreurs détectées immédiatement  
- ❌ Peut ralentir les opérations massives

**Contraintes différées (DEFERRABLE INITIALLY DEFERRED) :**
- ✅ Pas de vérification pendant la transaction (plus rapide pour opérations massives)  
- ❌ Vérification groupée à la fin (peut être coûteuse si beaucoup de violations)  
- ❌ Erreurs découvertes tardivement

### Benchmark Typique

```sql
-- Test sur 100 000 insertions

-- Avec contrainte immédiate
BEGIN;
    -- Temps : 12 secondes
    -- Vérification : 100 000 vérifications individuelles
COMMIT;

-- Avec contrainte différée
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;
    -- Temps : 8 secondes (33% plus rapide)
    -- Vérification : 1 vérification groupée à la fin
COMMIT;
```

**Gain :** Les contraintes différées sont généralement plus rapides pour les opérations en masse.

### Quand Utiliser les Contraintes Différées

#### ✅ Utiliser DEFERRABLE Si...

1. **Dépendances circulaires** entre tables ou lignes  
2. **Import de données** massif avec dépendances complexes  
3. **Réorganisation** de données existantes  
4. **Migrations** de schéma complexes  
5. **Échange de valeurs** (swap) entre lignes

#### ❌ Éviter DEFERRABLE Si...

1. **Intégrité critique** en temps réel (ex: soldes bancaires)  
2. **Contraintes simples** sans dépendances circulaires  
3. **Performance** : vérification immédiate suffisamment rapide  
4. **Simplicité** : pas de besoin de flexibilité

---

## Bonnes Pratiques

### 1. Utiliser DEFERRABLE Uniquement Quand Nécessaire

```sql
-- ❌ Pas nécessaire : pas de dépendances circulaires
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY DEFERRABLE INITIALLY DEFERRED,  -- Inutile
    nom VARCHAR(100)
);

-- ✅ Bon : contrainte simple, immédiate suffit
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY,  -- Par défaut : NOT DEFERRABLE
    nom VARCHAR(100)
);
```

### 2. Préférer INITIALLY IMMEDIATE pour Sécurité

```sql
-- ✅ Recommandé : immédiat par défaut, différable si besoin
ALTER TABLE produits  
ADD CONSTRAINT fk_categorie FOREIGN KEY (categorie_id)  
    REFERENCES categories(categorie_id)
    DEFERRABLE INITIALLY IMMEDIATE;

-- ⚠️ Moins sûr : différé par défaut (risque d'oublier les vérifications)
ALTER TABLE produits  
ADD CONSTRAINT fk_categorie FOREIGN KEY (categorie_id)  
    REFERENCES categories(categorie_id)
    DEFERRABLE INITIALLY DEFERRED;
```

**Raison :** INITIALLY IMMEDIATE détecte les erreurs plus tôt, et on peut explicitement différer quand nécessaire.

### 3. Documenter les Contraintes DEFERRABLE

```sql
CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    manager_id INTEGER,
    CONSTRAINT fk_manager FOREIGN KEY (manager_id)
        REFERENCES employes(employe_id)
        DEFERRABLE INITIALLY IMMEDIATE
);

COMMENT ON CONSTRAINT fk_manager ON employes IS
'Contrainte FK vers employes (auto-référence).
DEFERRABLE car permet l''échange de managers entre employés.  
Utiliser SET CONSTRAINTS fk_manager DEFERRED si nécessaire.';  
```

### 4. Tester les Transactions Complexes

```sql
-- Script de test
DO $$  
BEGIN  
    -- Test de la logique avec contraintes différées
    SET CONSTRAINTS ALL DEFERRED;

    -- Opérations
    UPDATE employes SET manager_id = 2 WHERE employe_id = 1;
    UPDATE employes SET manager_id = 1 WHERE employe_id = 2;

    -- Vérification manuelle avant COMMIT
    PERFORM * FROM employes WHERE manager_id NOT IN (SELECT employe_id FROM employes);

    IF FOUND THEN
        RAISE EXCEPTION 'Violation de FK détectée avant COMMIT';
    END IF;
END $$;
```

### 5. Gérer les Erreurs Gracieusement

```python
# Exemple Python avec psycopg3
import psycopg

try:
    with psycopg.connect("dbname=mydb") as conn:
        with conn.cursor() as cur:
            cur.execute("BEGIN")
            cur.execute("SET CONSTRAINTS ALL DEFERRED")

            # Opérations complexes
            cur.execute("UPDATE employes SET manager_id = %s WHERE employe_id = %s", (2, 1))
            cur.execute("UPDATE employes SET manager_id = %s WHERE employe_id = %s", (1, 2))

            cur.execute("COMMIT")
except psycopg.errors.ForeignKeyViolation as e:
    print(f"Erreur de contrainte FK : {e}")
    conn.rollback()
except Exception as e:
    print(f"Erreur inattendue : {e}")
    conn.rollback()
```

### 6. Nommer les Contraintes

```sql
-- ❌ Mauvais : nom généré automatiquement
ALTER TABLE produits  
ADD FOREIGN KEY (categorie_id) REFERENCES categories(categorie_id)  
DEFERRABLE;  

-- ✅ Bon : nom explicite
ALTER TABLE produits  
ADD CONSTRAINT fk_produits_categories FOREIGN KEY (categorie_id)  
    REFERENCES categories(categorie_id)
    DEFERRABLE INITIALLY IMMEDIATE;

-- Permet un contrôle précis
SET CONSTRAINTS fk_produits_categories DEFERRED;
```

---

## Exemples Avancés

### Exemple 1 : Arbre Hiérarchique avec Réorganisation

```sql
-- Arbre organisationnel
CREATE TABLE organisation (
    node_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    parent_id INTEGER,
    CONSTRAINT fk_parent FOREIGN KEY (parent_id)
        REFERENCES organisation(node_id)
        DEFERRABLE INITIALLY IMMEDIATE
);

-- Données initiales
INSERT INTO organisation (nom, parent_id) VALUES
    ('PDG', NULL),
    ('DG Ventes', 1),
    ('DG Tech', 1),
    ('Manager Ventes', 2),
    ('Dev Lead', 3);

-- Réorganiser : échanger deux branches
BEGIN;
    SET CONSTRAINTS fk_parent DEFERRED;

    -- Échanger les parents des branches
    UPDATE organisation SET parent_id = 3 WHERE node_id = 4;  -- Manager Ventes → DG Tech
    UPDATE organisation SET parent_id = 2 WHERE node_id = 5;  -- Dev Lead → DG Ventes

COMMIT;
```

### Exemple 2 : Gestion de Versions avec Références

```sql
-- Système de versioning de documents
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    titre VARCHAR(255),
    version INTEGER,
    version_precedente INTEGER,
    CONSTRAINT fk_version_prec FOREIGN KEY (version_precedente)
        REFERENCES documents(doc_id)
        DEFERRABLE INITIALLY DEFERRED
);

-- Créer une nouvelle version en référençant l'ancienne
BEGIN;
    -- Insérer nouvelle version (référence l'ancienne qui va être modifiée)
    INSERT INTO documents (titre, version, version_precedente)
    VALUES ('Mon Document', 2, 1);

    -- Mettre à jour l'ancienne version pour pointer vers la nouvelle
    UPDATE documents SET version_precedente = 2 WHERE doc_id = 1;

COMMIT;
```

### Exemple 3 : Swap de Clés Primaires

```sql
-- Tables avec relations complexes
CREATE TABLE utilisateurs (
    user_id INTEGER PRIMARY KEY DEFERRABLE INITIALLY IMMEDIATE,
    username VARCHAR(100) UNIQUE,
    email VARCHAR(255)
);

CREATE TABLE sessions (
    session_id SERIAL PRIMARY KEY,
    user_id INTEGER,
    CONSTRAINT fk_user FOREIGN KEY (user_id)
        REFERENCES utilisateurs(user_id)
        DEFERRABLE INITIALLY IMMEDIATE
);

-- Échanger les user_id de deux utilisateurs
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    -- Temporairement, les user_id seront en conflit
    UPDATE utilisateurs SET user_id = 999 WHERE user_id = 1;
    UPDATE utilisateurs SET user_id = 1 WHERE user_id = 2;
    UPDATE utilisateurs SET user_id = 2 WHERE user_id = 999;

    -- Mettre à jour les FK
    UPDATE sessions SET user_id = 1 WHERE user_id = 2;
    UPDATE sessions SET user_id = 2 WHERE user_id = 999;

COMMIT;
```

---

## Dépannage

### Erreur : Contrainte Non-Différable

```sql
-- Tentative de différer une contrainte non-différable
BEGIN;
    SET CONSTRAINTS fk_produit_categorie DEFERRED;
ROLLBACK;
-- ERROR: constraint "fk_produit_categorie" is not deferrable
```

**Solution :** Recréer la contrainte avec DEFERRABLE
```sql
ALTER TABLE produits DROP CONSTRAINT fk_produit_categorie;

ALTER TABLE produits  
ADD CONSTRAINT fk_produit_categorie FOREIGN KEY (categorie_id)  
    REFERENCES categories(categorie_id)
    DEFERRABLE INITIALLY IMMEDIATE;
```

### Erreur : Violation à la Fin de Transaction

```sql
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    INSERT INTO produits (nom, categorie_id) VALUES ('Test', 999);
    -- Aucune erreur ici (différé)

COMMIT;  -- ❌ ERREUR ICI
-- ERROR: insert or update on table "produits" violates foreign key constraint
```

**Solution :** Corriger les violations avant COMMIT
```sql
BEGIN;
    SET CONSTRAINTS ALL DEFERRED;

    INSERT INTO produits (nom, categorie_id) VALUES ('Test', 999);

    -- Vérification manuelle
    SELECT * FROM produits p
    LEFT JOIN categories c ON p.categorie_id = c.categorie_id
    WHERE c.categorie_id IS NULL;

    -- Correction
    UPDATE produits SET categorie_id = 1 WHERE categorie_id = 999;

COMMIT;  -- ✅ OK
```

### Vérifier le Statut d'une Contrainte

```sql
-- Lister toutes les contraintes et leur statut
SELECT
    conname AS constraint_name,
    contype AS constraint_type,
    condeferrable AS is_deferrable,
    condeferred AS is_deferred_by_default
FROM pg_constraint  
WHERE conrelid = 'employes'::regclass;  
```

**Résultat :**
```
constraint_name | constraint_type | is_deferrable | is_deferred_by_default
----------------+-----------------+---------------+------------------------
fk_manager      | f               | t             | f  
pk_employe_id   | p               | f             | f  
```

- `constraint_type` : 'f' = FK, 'p' = PK, 'u' = UNIQUE, 'c' = CHECK  
- `is_deferrable` : 't' = DEFERRABLE, 'f' = NOT DEFERRABLE  
- `is_deferred_by_default` : 't' = INITIALLY DEFERRED, 'f' = INITIALLY IMMEDIATE

---

## Conclusion

### Points Clés à Retenir

1. **Les contraintes différées permettent des violations temporaires**
   - Vérification reportée à la fin de la transaction
   - Utile pour dépendances circulaires

2. **Trois états possibles**
   - NOT DEFERRABLE : Toujours immédiat (défaut)
   - DEFERRABLE INITIALLY IMMEDIATE : Immédiat par défaut, différable sur demande
   - DEFERRABLE INITIALLY DEFERRED : Différé par défaut

3. **SET CONSTRAINTS contrôle le comportement**
   - Limité à la transaction courante
   - Permet de différer/rendre immédiat dynamiquement

4. **Cas d'usage principaux**
   - Références circulaires (auto-références, tables croisées)
   - Import de données avec dépendances
   - Réorganisation de données
   - Migrations complexes

5. **Bonnes pratiques**
   - Utiliser uniquement quand nécessaire
   - Préférer INITIALLY IMMEDIATE pour sécurité
   - Documenter les contraintes DEFERRABLE
   - Tester les transactions complexes

### Checklist d'Utilisation

Avant d'utiliser une contrainte DEFERRABLE :

- [ ] **Y a-t-il des dépendances circulaires ?** (auto-référence, FK croisées)  
- [ ] **L'ordre des opérations est-il contraint ?** (impossible de satisfaire immédiatement)  
- [ ] **Import de données** avec références complexes ?  
- [ ] **Réorganisation** nécessitant des états transitoires invalides ?  
- [ ] La contrainte peut-elle rester **NOT DEFERRABLE** ? (préférer si possible)

### Recommandation Générale

**Stratégie par défaut :**
1. Créer les contraintes **NOT DEFERRABLE** (défaut)  
2. Identifier les contraintes problématiques lors du développement  
3. Convertir en **DEFERRABLE INITIALLY IMMEDIATE** uniquement si nécessaire  
4. Utiliser `SET CONSTRAINTS ... DEFERRED` explicitement dans les transactions complexes

Les contraintes différées sont un outil puissant mais à utiliser avec discernement. Elles ajoutent de la flexibilité au prix d'une complexité accrue et d'erreurs potentiellement détectées plus tard.

---

**Fin du Chapitre 11.7**

Les contraintes différées sont une fonctionnalité avancée de PostgreSQL qui offre une flexibilité essentielle pour gérer des situations complexes où l'ordre des opérations est contraint par des dépendances circulaires. Utilisez-les judicieusement pour résoudre des problèmes d'intégrité référentielle qui seraient autrement impossibles à gérer.

⏭️ [Concurrence et Transactions](/12-concurrence-et-transactions/README.md)
