🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3. Le piège du NULL : Logique ternaire et fonctions de gestion

## Introduction

`NULL` est probablement **le concept le plus mal compris** et le plus problématique en SQL. Il est source d'innombrables bugs, de comportements inattendus et de frustrations pour les développeurs débutants comme expérimentés.

Pourquoi ? Parce que `NULL` ne se comporte pas comme les autres valeurs. Il ne représente ni zéro, ni une chaîne vide, ni faux. `NULL` représente **l'absence d'information** ou **une valeur inconnue**.

Dans ce chapitre, nous allons :
- Comprendre ce qu'est réellement NULL
- Découvrir la logique ternaire (trois états au lieu de deux)
- Explorer comment NULL affecte les opérations
- Maîtriser les fonctions pour gérer NULL (COALESCE, NULLIF)
- Identifier les pièges courants et comment les éviter

---

## Qu'est-ce que NULL ?

### Définition conceptuelle

`NULL` signifie **"valeur inconnue"** ou **"absence de valeur"**. C'est un marqueur spécial qui indique qu'une information n'est pas disponible.

**Exemples de situations où NULL a du sens :**

```sql
-- Un employé sans manager (CEO, par exemple)
INSERT INTO employes (nom, manager_id) VALUES ('Alice', NULL);

-- Un produit dont le prix n'est pas encore fixé
INSERT INTO produits (nom, prix) VALUES ('Nouveau produit', NULL);

-- Un client qui n'a pas fourni son numéro de téléphone
INSERT INTO clients (nom, telephone) VALUES ('Bob', NULL);

-- Une commande pas encore livrée (date de livraison inconnue)
INSERT INTO commandes (numero, date_livraison) VALUES ('CMD123', NULL);
```

### Ce que NULL n'est PAS

**NULL ≠ 0 (zéro)**

```sql
-- Ces trois employés ont des situations DIFFÉRENTES
INSERT INTO employes (nom, salaire) VALUES
    ('Alice', 50000),    -- Alice gagne 50000
    ('Bob', 0),          -- Bob gagne 0 (bénévole ?)
    ('Charlie', NULL);   -- Le salaire de Charlie est inconnu

-- Requête pour comprendre
SELECT nom, salaire,
    CASE
        WHEN salaire IS NULL THEN 'Salaire inconnu'
        WHEN salaire = 0 THEN 'Salaire nul'
        ELSE 'Salaire: ' || salaire
    END as statut
FROM employes;
```

**Résultat :**
```
nom      | salaire | statut
---------|---------|-------------------
Alice    | 50000   | Salaire: 50000
Bob      | 0       | Salaire nul
Charlie  | NULL    | Salaire inconnu
```

**NULL ≠ '' (chaîne vide)**

```sql
-- Ces trois clients ont des situations DIFFÉRENTES
INSERT INTO clients (nom, email) VALUES
    ('Alice', 'alice@example.com'),  -- Alice a fourni un email
    ('Bob', ''),                     -- Bob a fourni une chaîne vide
    ('Charlie', NULL);               -- Charlie n'a pas fourni d'email

-- Vérification
SELECT nom, email,
    CASE
        WHEN email IS NULL THEN 'Email non fourni'
        WHEN email = '' THEN 'Email vide'
        ELSE email
    END as statut_email
FROM clients;
```

**NULL ≠ FALSE (faux)**

```sql
-- NULL n'est pas un booléen
-- NULL signifie "on ne sait pas si c'est vrai ou faux"

SELECT
    TRUE as vrai,
    FALSE as faux,
    NULL as inconnu;
```

---

## La logique ternaire (trois valeurs)

En logique booléenne classique, vous avez deux états : **TRUE** (vrai) ou **FALSE** (faux).

En SQL, avec NULL, vous avez **trois états** :
- **TRUE** : vrai
- **FALSE** : faux
- **NULL** : inconnu

C'est ce qu'on appelle la **logique ternaire** (ou logique à trois valeurs).

### Comparaisons avec NULL

Toute comparaison impliquant NULL retourne **NULL** (pas TRUE, pas FALSE) :

```sql
SELECT
    NULL = NULL as test1,      -- NULL (pas TRUE !)
    NULL != NULL as test2,     -- NULL
    NULL > 5 as test3,         -- NULL
    NULL < 5 as test4,         -- NULL
    5 = NULL as test5,         -- NULL
    NULL + 10 as test6,        -- NULL
    NULL * 2 as test7,         -- NULL
    NULL || 'texte' as test8;  -- NULL (concaténation)
```

**Résultat :** Toutes ces expressions retournent `NULL`.

**Pourquoi ?**

Pensez-y logiquement :
- Si vous ne connaissez pas une valeur (NULL), comment pouvez-vous dire si elle est égale à une autre ?
- Si vous ne connaissez pas un nombre (NULL), comment pouvez-vous savoir s'il est supérieur à 5 ?
- Si vous ne connaissez pas une valeur (NULL), comment pouvez-vous lui ajouter 10 ?

La réponse est : **vous ne pouvez pas**. Le résultat est donc inconnu (NULL).

### Opérateur spécial : IS NULL et IS NOT NULL

C'est pourquoi SQL fournit des opérateurs spéciaux pour tester NULL :

```sql
-- ❌ NE FONCTIONNE JAMAIS
WHERE colonne = NULL    -- Retourne toujours NULL (évalué comme FALSE en WHERE)
WHERE colonne != NULL   -- Retourne toujours NULL

-- ✅ SYNTAXE CORRECTE
WHERE colonne IS NULL       -- Teste si la valeur est NULL
WHERE colonne IS NOT NULL   -- Teste si la valeur n'est pas NULL
```

**Exemple concret :**

```sql
-- Table d'employés
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    manager_id INTEGER
);

INSERT INTO employes (nom, manager_id) VALUES
    ('Alice', NULL),   -- PDG, pas de manager
    ('Bob', 1),        -- Manager: Alice
    ('Charlie', 1),    -- Manager: Alice
    ('David', 2);      -- Manager: Bob

-- ❌ Cette requête ne retourne AUCUNE ligne
SELECT nom FROM employes WHERE manager_id = NULL;

-- ✅ Cette requête retourne Alice
SELECT nom FROM employes WHERE manager_id IS NULL;

-- ✅ Cette requête retourne Bob, Charlie, David
SELECT nom FROM employes WHERE manager_id IS NOT NULL;
```

---

## NULL et les opérateurs logiques

### AND avec NULL

| A     | B     | A AND B |
|-------|-------|---------|
| TRUE  | TRUE  | TRUE    |
| TRUE  | FALSE | FALSE   |
| TRUE  | NULL  | **NULL**|
| FALSE | TRUE  | FALSE   |
| FALSE | FALSE | FALSE   |
| FALSE | NULL  | **FALSE**|
| NULL  | TRUE  | **NULL**|
| NULL  | FALSE | **FALSE**|
| NULL  | NULL  | **NULL**|

**Règle mnémotechnique :**
- `FALSE AND n'importe quoi` = **FALSE** (car une seule condition fausse suffit)
- `TRUE AND NULL` = **NULL** (on ne sait pas)
- `NULL AND NULL` = **NULL** (on ne sait pas)

**Exemple :**

```sql
SELECT nom, salaire, age
FROM employes
WHERE salaire > 50000 AND age > 30;

-- Si un employé a salaire=60000 mais age=NULL :
-- 60000 > 50000 = TRUE
-- NULL > 30 = NULL
-- TRUE AND NULL = NULL
-- → Cet employé NE sera PAS retourné (NULL est évalué comme FALSE en WHERE)
```

### OR avec NULL

| A     | B     | A OR B  |
|-------|-------|---------|
| TRUE  | TRUE  | TRUE    |
| TRUE  | FALSE | TRUE    |
| TRUE  | NULL  | **TRUE**|
| FALSE | TRUE  | TRUE    |
| FALSE | FALSE | FALSE   |
| FALSE | NULL  | **NULL**|
| NULL  | TRUE  | **TRUE**|
| NULL  | FALSE | **NULL**|
| NULL  | NULL  | **NULL**|

**Règle mnémotechnique :**
- `TRUE OR n'importe quoi` = **TRUE** (car une seule condition vraie suffit)
- `FALSE OR NULL` = **NULL** (on ne sait pas)
- `NULL OR NULL` = **NULL** (on ne sait pas)

**Exemple :**

```sql
SELECT nom, statut, date_fin_contrat
FROM employes
WHERE statut = 'actif' OR date_fin_contrat > CURRENT_DATE;

-- Si un employé a statut='inactif' et date_fin_contrat=NULL :
-- 'inactif' = 'actif' = FALSE
-- NULL > CURRENT_DATE = NULL
-- FALSE OR NULL = NULL
-- → Cet employé NE sera PAS retourné
```

### NOT avec NULL

| A     | NOT A |
|-------|-------|
| TRUE  | FALSE |
| FALSE | TRUE  |
| NULL  | **NULL**|

**Exemple :**

```sql
SELECT nom FROM employes WHERE NOT (age > 30);

-- Si un employé a age=NULL :
-- age > 30 = NULL
-- NOT NULL = NULL
-- → Cet employé NE sera PAS retourné
```

### Implication pratique en WHERE

La clause `WHERE` ne retient que les lignes où la condition évalue à **TRUE**.

Si une condition évalue à **NULL** ou **FALSE**, la ligne est **éliminée**.

```sql
-- Table exemple
CREATE TABLE produits (
    nom VARCHAR(100),
    prix NUMERIC,
    remise NUMERIC
);

INSERT INTO produits VALUES
    ('Produit A', 100, 10),
    ('Produit B', 50, NULL),
    ('Produit C', 200, 20);

-- Requête
SELECT nom, prix, remise
FROM produits
WHERE prix - remise > 80;

-- Résultats :
-- Produit A : 100 - 10 = 90 > 80 → TRUE → retourné
-- Produit B : 50 - NULL = NULL > 80 → NULL → PAS retourné
-- Produit C : 200 - 20 = 180 > 80 → TRUE → retourné
```

**Résultat final :**
```
nom        | prix | remise
-----------|------|-------
Produit A  | 100  | 10
Produit C  | 200  | 20
```

Le Produit B est éliminé car `NULL > 80` = `NULL`.

---

## NULL et les agrégations

Les fonctions d'agrégation **ignorent les valeurs NULL** (sauf `COUNT(*)`).

### COUNT

```sql
CREATE TABLE employes (
    nom VARCHAR(100),
    telephone VARCHAR(20)
);

INSERT INTO employes VALUES
    ('Alice', '0601020304'),
    ('Bob', NULL),
    ('Charlie', '0605060708'),
    ('David', NULL);

-- COUNT(*) : compte toutes les lignes
SELECT COUNT(*) FROM employes;  -- Résultat : 4

-- COUNT(colonne) : compte les valeurs NON-NULL
SELECT COUNT(telephone) FROM employes;  -- Résultat : 2

-- COUNT(DISTINCT colonne) : compte les valeurs uniques NON-NULL
SELECT COUNT(DISTINCT telephone) FROM employes;  -- Résultat : 2
```

### SUM, AVG, MIN, MAX

Ces fonctions **ignorent** les NULL :

```sql
CREATE TABLE ventes (
    produit VARCHAR(50),
    montant NUMERIC
);

INSERT INTO ventes VALUES
    ('Produit A', 100),
    ('Produit B', NULL),  -- Vente annulée, montant inconnu
    ('Produit C', 200),
    ('Produit D', NULL);

-- SUM ignore les NULL
SELECT SUM(montant) FROM ventes;  -- Résultat : 300 (pas 300 + NULL + NULL)

-- AVG ignore les NULL
SELECT AVG(montant) FROM ventes;  -- Résultat : 150 (moyenne de 100 et 200)

-- MIN et MAX ignorent les NULL
SELECT MIN(montant) FROM ventes;  -- Résultat : 100
SELECT MAX(montant) FROM ventes;  -- Résultat : 200

-- COUNT(*) compte tout
SELECT COUNT(*) FROM ventes;  -- Résultat : 4

-- COUNT(montant) ignore les NULL
SELECT COUNT(montant) FROM ventes;  -- Résultat : 2
```

**⚠️ Attention au calcul de moyenne :**

```sql
-- Moyenne calculée par PostgreSQL (ignore NULL)
SELECT AVG(montant) FROM ventes;  -- 150

-- Si vous calculez manuellement avec SUM/COUNT :
SELECT SUM(montant) / COUNT(*) FROM ventes;  -- 300 / 4 = 75 ❌ FAUX

-- Calcul correct
SELECT SUM(montant) / COUNT(montant) FROM ventes;  -- 300 / 2 = 150 ✓
```

---

## Les fonctions de gestion de NULL

PostgreSQL fournit plusieurs fonctions pour gérer les NULL de manière élégante.

### COALESCE : Remplacer NULL par une valeur par défaut

`COALESCE` retourne la **première valeur non-NULL** de sa liste d'arguments.

**Syntaxe :**
```sql
COALESCE(valeur1, valeur2, valeur3, ..., valeur_par_defaut)
```

**Fonctionnement :**
- Si `valeur1` n'est pas NULL → retourne `valeur1`
- Sinon, si `valeur2` n'est pas NULL → retourne `valeur2`
- Sinon, si `valeur3` n'est pas NULL → retourne `valeur3`
- ... et ainsi de suite
- Si toutes les valeurs sont NULL → retourne `valeur_par_defaut`

**Exemples simples :**

```sql
SELECT COALESCE(NULL, 'Valeur par défaut');  -- Résultat : 'Valeur par défaut'

SELECT COALESCE('Valeur', 'Défaut');  -- Résultat : 'Valeur'

SELECT COALESCE(NULL, NULL, 'Troisième', 'Défaut');  -- Résultat : 'Troisième'

SELECT COALESCE(NULL, NULL, NULL);  -- Résultat : NULL (aucune valeur non-NULL)
```

**Cas d'usage pratiques :**

#### 1. Affichage de valeurs par défaut

```sql
SELECT
    nom,
    COALESCE(telephone, 'Non renseigné') as telephone,
    COALESCE(email, 'Non renseigné') as email
FROM clients;

-- Si Alice n'a pas de téléphone :
-- nom: Alice, telephone: Non renseigné
```

#### 2. Calculs avec valeurs NULL

```sql
-- Calculer un prix final avec remise optionnelle
SELECT
    nom_produit,
    prix,
    remise,
    prix - COALESCE(remise, 0) as prix_final
FROM produits;

-- Si remise est NULL, on utilise 0 pour le calcul
-- Sinon, on utilise la vraie valeur de remise
```

#### 3. Valeurs de substitution en cascade

```sql
-- Utiliser le téléphone mobile, sinon fixe, sinon un message
SELECT
    nom,
    COALESCE(tel_mobile, tel_fixe, 'Aucun téléphone') as contact
FROM clients;
```

#### 4. Agrégations avec valeur par défaut

```sql
-- Salaire moyen ou message si aucune donnée
SELECT
    departement,
    COALESCE(AVG(salaire)::TEXT, 'Aucune donnée') as salaire_moyen
FROM employes
GROUP BY departement;
```

#### 5. Jointures avec valeurs manquantes

```sql
-- Afficher le nom du manager ou 'PDG' si NULL
SELECT
    e.nom as employe,
    COALESCE(m.nom, 'PDG') as manager
FROM employes e
LEFT JOIN employes m ON e.manager_id = m.id;
```

#### 6. Formulaires et valeurs par défaut

```sql
-- Insérer des valeurs avec défauts si non fournies
INSERT INTO preferences (user_id, theme, langue, notifications)
VALUES (
    123,
    COALESCE(:theme, 'clair'),           -- 'clair' par défaut
    COALESCE(:langue, 'fr'),             -- 'fr' par défaut
    COALESCE(:notifications, true)       -- true par défaut
);
```

### NULLIF : Convertir une valeur en NULL

`NULLIF` retourne **NULL** si deux valeurs sont égales, sinon retourne la première valeur.

**Syntaxe :**
```sql
NULLIF(valeur1, valeur2)
```

**Fonctionnement :**
- Si `valeur1 = valeur2` → retourne **NULL**
- Sinon → retourne `valeur1`

**Pourquoi est-ce utile ?**

Parfois, certaines valeurs "spéciales" doivent être traitées comme NULL (par exemple, 0, -1, '', 'N/A', etc.).

**Exemples simples :**

```sql
SELECT NULLIF(5, 5);       -- Résultat : NULL (5 = 5)
SELECT NULLIF(5, 10);      -- Résultat : 5 (5 ≠ 10)
SELECT NULLIF('', '');     -- Résultat : NULL ('' = '')
SELECT NULLIF('abc', '');  -- Résultat : 'abc' ('abc' ≠ '')
```

**Cas d'usage pratiques :**

#### 1. Éviter la division par zéro

```sql
-- ❌ Division par zéro → ERREUR
SELECT 10 / 0;  -- ERROR: division by zero

-- ✅ Avec NULLIF
SELECT 10 / NULLIF(0, 0);  -- Résultat : NULL (pas d'erreur)

-- Cas réel : calculer un pourcentage
SELECT
    nom_produit,
    ventes,
    stock,
    (ventes * 100.0 / NULLIF(stock, 0))::NUMERIC(10,2) as pourcentage_vendu
FROM produits;

-- Si stock = 0, le résultat sera NULL au lieu d'une erreur
```

#### 2. Convertir des chaînes vides en NULL

```sql
-- Certaines applications insèrent '' au lieu de NULL
-- Normaliser ces données :
UPDATE clients
SET email = NULLIF(email, '')
WHERE email = '';

-- Ou en requête de sélection
SELECT
    nom,
    NULLIF(email, '') as email_propre
FROM clients;
```

#### 3. Convertir des valeurs sentinelles en NULL

```sql
-- Anciennes bases de données utilisent parfois -1 ou 0 pour "non défini"
SELECT
    nom_produit,
    NULLIF(stock, -1) as stock_reel,
    NULLIF(delai_livraison, 0) as delai_reel
FROM produits;

-- stock=-1 devient NULL
-- delai_livraison=0 devient NULL
```

#### 4. Nettoyage de données importées

```sql
-- Fichiers CSV avec valeurs spéciales
SELECT
    nom,
    NULLIF(telephone, 'N/A') as telephone,
    NULLIF(ville, 'Unknown') as ville,
    NULLIF(age, -1) as age
FROM donnees_importees;
```

### Combiner COALESCE et NULLIF

Ces deux fonctions se complètent souvent :

```sql
-- Convertir les chaînes vides en NULL, puis fournir un défaut
SELECT
    nom,
    COALESCE(NULLIF(email, ''), 'aucun@email.com') as email
FROM clients;

-- Diviser avec protection contre zéro, puis fournir un défaut
SELECT
    produit,
    COALESCE(ventes / NULLIF(stock, 0), 0) as ratio
FROM statistiques;

-- Étapes :
-- 1. NULLIF(stock, 0) : si stock=0, retourne NULL
-- 2. ventes / NULL = NULL
-- 3. COALESCE(NULL, 0) = 0
```

---

## Autres fonctions utiles

### GREATEST et LEAST

Ces fonctions ignorent NULL **par défaut dans PostgreSQL** :

```sql
-- GREATEST retourne la plus grande valeur (ignore NULL)
SELECT GREATEST(10, NULL, 5, 20);  -- Résultat : 20

SELECT GREATEST(NULL, NULL);  -- Résultat : NULL (tous NULL)

-- LEAST retourne la plus petite valeur (ignore NULL)
SELECT LEAST(10, NULL, 5, 20);  -- Résultat : 5

-- Cas d'usage : prendre le prix le plus bas entre plusieurs options
SELECT
    nom_produit,
    LEAST(prix_normal, prix_promo, prix_membre) as meilleur_prix
FROM produits;
```

### CASE pour une logique complexe

`CASE` permet une gestion très flexible de NULL :

```sql
SELECT
    nom,
    salaire,
    CASE
        WHEN salaire IS NULL THEN 'Salaire non défini'
        WHEN salaire = 0 THEN 'Bénévole'
        WHEN salaire < 30000 THEN 'Junior'
        WHEN salaire < 60000 THEN 'Confirmé'
        ELSE 'Senior'
    END as categorie
FROM employes;
```

### IS DISTINCT FROM

Opérateur spécial qui traite NULL comme une valeur comparable :

```sql
-- Comparaison classique
SELECT NULL = NULL;    -- Résultat : NULL
SELECT NULL != NULL;   -- Résultat : NULL

-- Avec IS DISTINCT FROM
SELECT NULL IS DISTINCT FROM NULL;  -- Résultat : FALSE (ils sont "égaux")
SELECT 5 IS DISTINCT FROM NULL;     -- Résultat : TRUE (ils sont différents)
SELECT 5 IS DISTINCT FROM 5;        -- Résultat : FALSE

-- Cas d'usage : détecter des changements (même si NULL)
SELECT * FROM employes_nouveau
WHERE (salaire, departement) IS DISTINCT FROM
      (SELECT salaire, departement FROM employes_ancien WHERE id = employes_nouveau.id);
```

---

## Les pièges courants avec NULL

### Piège 1 : NOT IN avec NULL

C'est l'un des pièges les plus vicieux de SQL.

```sql
-- Table de départements autorisés
CREATE TABLE dept_autorises (dept VARCHAR(50));
INSERT INTO dept_autorises VALUES ('IT'), ('RH'), (NULL);

-- Trouver les employés n'étant PAS dans les départements autorisés
SELECT nom, departement
FROM employes
WHERE departement NOT IN (SELECT dept FROM dept_autorises);
```

**Résultat attendu :** Les employés des départements non listés (Ventes, Finance, etc.)

**Résultat réel :** **AUCUNE LIGNE** ! 😱

**Pourquoi ?**

`NOT IN` est équivalent à une série de comparaisons avec `!=` et `AND` :

```sql
WHERE departement NOT IN ('IT', 'RH', NULL)

-- Est équivalent à :
WHERE departement != 'IT'
  AND departement != 'RH'
  AND departement != NULL

-- Or, departement != NULL retourne toujours NULL
-- TRUE AND TRUE AND NULL = NULL
-- Donc aucune ligne n'est retenue
```

**✅ Solution 1 : Exclure les NULL de la sous-requête**

```sql
WHERE departement NOT IN (SELECT dept FROM dept_autorises WHERE dept IS NOT NULL)
```

**✅ Solution 2 : Utiliser NOT EXISTS**

```sql
WHERE NOT EXISTS (
    SELECT 1 FROM dept_autorises
    WHERE dept = employes.departement
)
```

### Piège 2 : Agrégations et NULL dans GROUP BY

```sql
CREATE TABLE commandes (
    client_id INTEGER,
    montant NUMERIC
);

INSERT INTO commandes VALUES
    (1, 100),
    (1, 200),
    (2, 150),
    (NULL, 50),  -- Commande sans client
    (NULL, 75);

SELECT client_id, SUM(montant)
FROM commandes
GROUP BY client_id;
```

**Résultat :**
```
client_id | sum
----------|-----
1         | 300
2         | 150
NULL      | 125  -- Les NULL sont groupés ensemble !
```

Les valeurs NULL sont **traitées comme égales** lors du regroupement.

### Piège 3 : UNIQUE et NULL

Dans PostgreSQL, **plusieurs NULL sont autorisés** dans une colonne UNIQUE :

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE
);

-- Ces trois insertions fonctionnent !
INSERT INTO utilisateurs (email) VALUES (NULL);
INSERT INTO utilisateurs (email) VALUES (NULL);
INSERT INTO utilisateurs (email) VALUES (NULL);

-- Mais celle-ci échoue (doublon non-NULL)
INSERT INTO utilisateurs (email) VALUES ('test@example.com');
INSERT INTO utilisateurs (email) VALUES ('test@example.com');  -- ❌ ERREUR
```

**Pourquoi ?** Selon le standard SQL, NULL != NULL, donc plusieurs NULL ne violent pas la contrainte UNIQUE.

**Si vous voulez interdire les NULL multiples :**

```sql
-- Ajoutez une contrainte NOT NULL
email VARCHAR(100) UNIQUE NOT NULL

-- Ou utilisez un index unique avec filtrage (PostgreSQL)
CREATE UNIQUE INDEX idx_users_email ON utilisateurs(email) WHERE email IS NOT NULL;
-- Permet un seul NULL, mais pas de doublons non-NULL
```

### Piège 4 : ORDER BY et NULL

Par défaut, PostgreSQL place les NULL **en dernier** lors du tri ascendant :

```sql
SELECT nom, salaire
FROM employes
ORDER BY salaire ASC;
```

**Résultat :**
```
nom      | salaire
---------|--------
Alice    | 30000
Bob      | 50000
Charlie  | 70000
David    | NULL    -- NULL en dernier
Eve      | NULL
```

**Contrôler le placement des NULL :**

```sql
-- NULL en premier
ORDER BY salaire ASC NULLS FIRST

-- NULL en dernier (par défaut pour ASC)
ORDER BY salaire ASC NULLS LAST

-- NULL en premier (par défaut pour DESC)
ORDER BY salaire DESC NULLS FIRST

-- NULL en dernier
ORDER BY salaire DESC NULLS LAST
```

### Piège 5 : Concaténation avec NULL

```sql
SELECT 'Bonjour ' || NULL || ' monde';  -- Résultat : NULL

-- Toute la chaîne devient NULL !

-- ✅ Solution avec COALESCE
SELECT 'Bonjour ' || COALESCE(prenom, '') || ' ' || nom
FROM clients;
```

### Piège 6 : Calculs avec NULL

```sql
-- Un seul NULL rend tout le calcul NULL
SELECT prix * quantite * (1 - COALESCE(remise_pct, 0) / 100) as total
FROM lignes_commande;

-- Si quantite est NULL, total sera NULL
-- Si remise_pct est NULL, on utilise 0
```

---

## Stratégies pour gérer NULL

### 1. Éviter NULL quand c'est possible

**Utiliser des valeurs par défaut :**

```sql
CREATE TABLE employes (
    nom VARCHAR(100) NOT NULL,
    telephone VARCHAR(20) DEFAULT '',  -- Chaîne vide au lieu de NULL
    actif BOOLEAN DEFAULT TRUE,        -- TRUE par défaut
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**⚠️ Mais attention :** Une chaîne vide n'est pas la même chose que NULL. Réfléchissez à la sémantique :
- `NULL` = "information non fournie"
- `''` = "information fournie, mais vide"

### 2. Utiliser NOT NULL quand une valeur est obligatoire

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,        -- Un produit DOIT avoir un nom
    prix NUMERIC(10,2) NOT NULL,      -- Un produit DOIT avoir un prix
    description TEXT,                 -- Optionnel (peut être NULL)
    image_url VARCHAR(500)            -- Optionnel
);
```

### 3. Fournir des valeurs par défaut dans les requêtes

```sql
-- À la sélection
SELECT
    nom,
    COALESCE(telephone, 'Non renseigné'),
    COALESCE(email, 'Non renseigné'),
    COALESCE(adresse, 'Non renseignée')
FROM clients;

-- À l'insertion
INSERT INTO produits (nom, prix, remise)
VALUES ('Nouveau produit', 99.99, COALESCE(:remise_input, 0));
```

### 4. Normaliser les NULL à l'insertion

```sql
-- Transformer les chaînes vides en NULL
INSERT INTO clients (nom, email, telephone)
VALUES (
    'Alice',
    NULLIF(TRIM(:email), ''),      -- Si email vide, mettre NULL
    NULLIF(TRIM(:telephone), '')   -- Si telephone vide, mettre NULL
);
```

### 5. Documenter la signification de NULL

```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,

    -- NULL signifie "pas de manager" (cas du PDG)
    manager_id INTEGER REFERENCES employes(id),

    -- NULL signifie "salaire non négocié" ou "données confidentielles"
    salaire NUMERIC(10,2),

    -- NULL signifie "date de fin de contrat non définie" (CDI)
    date_fin_contrat DATE
);
```

---

## Comparaison : NULL vs valeurs sentinelles

Certains systèmes utilisent des **valeurs sentinelles** au lieu de NULL :
- `-1` pour "non défini"
- `0` pour "inconnu"
- `''` pour "pas de texte"
- `'1900-01-01'` pour "pas de date"

**Avantages de NULL :**
- Sémantiquement correct
- Pas de confusion avec des valeurs réelles
- Fonctions d'agrégation les ignorent automatiquement
- Standard SQL

**Inconvénients de NULL :**
- Logique ternaire complexe
- Pièges avec NOT IN, comparaisons, etc.
- Nécessite IS NULL / IS NOT NULL

**Recommandation :** Utilisez NULL pour représenter l'absence de données, mais soyez conscient des pièges.

---

## Exemple synthétique : Rapport avec gestion de NULL

```sql
-- Générer un rapport de ventes avec gestion complète de NULL
SELECT
    -- Nom du vendeur avec défaut
    COALESCE(v.nom, 'Vendeur inconnu') as vendeur,

    -- Département avec défaut
    COALESCE(v.departement, 'Non assigné') as departement,

    -- Nombre de ventes (COUNT(*) ne compte pas NULL)
    COUNT(*) as nb_ventes,

    -- Montant total (SUM ignore NULL)
    COALESCE(SUM(c.montant), 0) as total_ventes,

    -- Montant moyen (AVG ignore NULL)
    COALESCE(AVG(c.montant)::NUMERIC(10,2), 0) as vente_moyenne,

    -- Commission avec défaut si NULL
    COALESCE(v.taux_commission, 0) as taux_commission,

    -- Calcul sécurisé de la commission
    COALESCE(SUM(c.montant), 0) * COALESCE(v.taux_commission, 0) / 100 as commission_totale,

    -- Dernière vente avec message si aucune
    COALESCE(MAX(c.date_vente)::TEXT, 'Aucune vente') as derniere_vente

FROM vendeurs v
LEFT JOIN commandes c ON v.id = c.vendeur_id
GROUP BY v.id, v.nom, v.departement, v.taux_commission
ORDER BY total_ventes DESC NULLS LAST;
```

---

## Points clés à retenir

1. **NULL ≠ 0, NULL ≠ '', NULL ≠ FALSE**
   - NULL signifie "valeur inconnue" ou "absence de valeur"

2. **Logique ternaire : TRUE, FALSE, NULL**
   - Toute opération avec NULL retourne généralement NULL

3. **WHERE élimine les NULL**
   - Seules les lignes où la condition = TRUE sont retenues
   - NULL et FALSE sont éliminés

4. **Utilisez IS NULL et IS NOT NULL**
   - Jamais `= NULL` ou `!= NULL`

5. **Les agrégations ignorent NULL**
   - Sauf `COUNT(*)` qui compte toutes les lignes

6. **COALESCE pour les valeurs par défaut**
   - `COALESCE(valeur, defaut)` retourne la première valeur non-NULL

7. **NULLIF pour convertir en NULL**
   - `NULLIF(valeur, sentinelle)` retourne NULL si égales

8. **Attention à NOT IN avec NULL**
   - Peut ne retourner aucune ligne de manière inattendue

9. **NULL dans UNIQUE permet des doublons**
   - Plusieurs NULL sont autorisés

10. **Documentez la signification de NULL**
    - Dans votre schéma, précisez ce que NULL représente

---

## Conclusion

NULL est un concept fondamental mais déroutant en SQL. Sa logique ternaire (TRUE/FALSE/NULL) crée des comportements contre-intuitifs qui peuvent générer des bugs subtils.

Les clés pour maîtriser NULL :
- **Comprendre** que NULL signifie "inconnu", pas "zéro" ou "vide"
- **Utiliser** `IS NULL` et `IS NOT NULL` pour les tests
- **Anticiper** le comportement de NULL dans les opérations et les agrégations
- **Gérer** NULL explicitement avec `COALESCE`, `NULLIF` et `CASE`
- **Éviter** NULL quand une valeur est obligatoire (contrainte `NOT NULL`)
- **Documenter** la signification de NULL dans votre modèle de données

Avec ces connaissances, vous êtes maintenant équipé pour éviter les pièges courants et écrire des requêtes SQL robustes qui gèrent correctement l'absence de données.

---


⏭️ [Tri (ORDER BY) et gestion des NULLs (NULLS FIRST/LAST)](/05-requetes-de-selection/04-tri-order-by.md)
