🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4. Opérations d'ensemble : UNION, INTERSECT, EXCEPT (ALL)

## Introduction

Les opérations d'ensemble en SQL permettent de **combiner les résultats de plusieurs requêtes** en appliquant des opérations mathématiques sur des ensembles de données. Ces opérations sont directement inspirées de la **théorie des ensembles** en mathématiques.

PostgreSQL supporte trois opérations d'ensemble principales :
- **UNION** : Combine les résultats (union)  
- **INTERSECT** : Garde uniquement les éléments communs (intersection)  
- **EXCEPT** : Soustrait un ensemble d'un autre (différence)

Chacune existe en deux variantes : standard (élimine les doublons) et **ALL** (conserve les doublons).

---

## Rappel : La théorie des ensembles

Avant de plonger dans SQL, rappelons les concepts mathématiques de base.

### Visualisation avec diagrammes de Venn

**Ensemble A** : {1, 2, 3, 4, 5}  
**Ensemble B** : {4, 5, 6, 7, 8}  

```
    A              B
  ┌─────┐      ┌─────┐
  │ 1 2 │      │ 6 7 │
  │  3  │      │  8  │
  │   ┌─┴──────┴─┐   │
  │   │  4  5    │   │
  └───┴──────────┴───┘
       Intersection
```

**Opérations :**

1. **A ∪ B (Union)** : {1, 2, 3, 4, 5, 6, 7, 8}  
   → Tous les éléments de A **OU** B

2. **A ∩ B (Intersection)** : {4, 5}  
   → Éléments présents dans A **ET** B

3. **A − B (Différence)** : {1, 2, 3}  
   → Éléments dans A mais **PAS** dans B

4. **B − A (Différence)** : {6, 7, 8}  
   → Éléments dans B mais **PAS** dans A

---

## Règles générales pour toutes les opérations d'ensemble

### 1. Compatibilité des colonnes

Les requêtes combinées doivent avoir :
- **Même nombre de colonnes**  
- **Types de données compatibles** (dans le même ordre)

```sql
-- ✅ CORRECT : 2 colonnes INTEGER et TEXT dans les deux requêtes
SELECT id, nom FROM employes  
UNION  
SELECT id, nom FROM consultants;  

-- ❌ ERREUR : Nombre de colonnes différent
SELECT id, nom FROM employes  
UNION  
SELECT id FROM consultants;  -- Manque une colonne  

-- ❌ ERREUR : Types incompatibles
SELECT id, nom FROM employes  -- INTEGER, TEXT  
UNION  
SELECT nom, id FROM consultants;  -- TEXT, INTEGER (ordre inversé)  
```

### 2. Les noms de colonnes viennent de la première requête

```sql
SELECT id AS employee_id, nom AS employee_name FROM employes  
UNION  
SELECT id, nom FROM consultants;  

-- Résultat : colonnes nommées "employee_id" et "employee_name"
```

### 3. ORDER BY se place à la fin

Le tri s'applique au résultat final de l'opération d'ensemble.

```sql
SELECT nom FROM employes  
UNION  
SELECT nom FROM consultants  
ORDER BY nom;  -- ✅ Tri du résultat combiné  

-- ❌ ERREUR : ORDER BY dans une sous-requête
SELECT nom FROM employes ORDER BY nom  
UNION  
SELECT nom FROM consultants;  
```

**Exception :** Vous pouvez utiliser ORDER BY dans une sous-requête si vous utilisez LIMIT :

```sql
(SELECT nom FROM employes ORDER BY salaire DESC LIMIT 5)
UNION
(SELECT nom FROM consultants ORDER BY tarif_horaire DESC LIMIT 5)
ORDER BY nom;
```

### 4. Parenthèses pour la clarté

Bien que l'ordre d'évaluation soit défini, les parenthèses améliorent la lisibilité.

```sql
-- Plus clair avec parenthèses
(SELECT nom FROM employes WHERE departement = 'IT')
UNION
(SELECT nom FROM consultants WHERE specialite = 'DevOps')
ORDER BY nom;
```

---

## UNION : Combiner des ensembles

### Définition

`UNION` combine les résultats de deux requêtes ou plus en **éliminant les doublons**.

### Syntaxe

```sql
SELECT colonnes FROM table1  
UNION  
SELECT colonnes FROM table2  
[UNION
SELECT colonnes FROM table3 ...]
[ORDER BY colonnes];
```

### Exemple simple

```sql
-- Table clients
CREATE TABLE clients (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(100)
);

INSERT INTO clients VALUES
    (1, 'Alice'),
    (2, 'Bob'),
    (3, 'Charlie');

-- Table fournisseurs
CREATE TABLE fournisseurs (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(100)
);

INSERT INTO fournisseurs VALUES
    (1, 'Bob'),        -- Aussi client
    (2, 'Diana'),
    (3, 'Eve');
```

**Requête avec UNION :**

```sql
SELECT nom FROM clients  
UNION  
SELECT nom FROM fournisseurs  
ORDER BY nom;  
```

**Résultat :**

```
nom
--------
Alice  
Bob      ← Présent une seule fois (dédupliqué)  
Charlie  
Diana  
Eve  
```

**Analyse :** Bob apparaît dans les deux tables mais n'est listé qu'une fois grâce à `UNION`.

---

## UNION ALL : Conserver tous les résultats

### Définition

`UNION ALL` combine les résultats **sans éliminer les doublons**. Plus rapide que `UNION` car pas de déduplication.

### Syntaxe

```sql
SELECT colonnes FROM table1  
UNION ALL  
SELECT colonnes FROM table2;  
```

### Exemple

```sql
SELECT nom FROM clients  
UNION ALL  
SELECT nom FROM fournisseurs  
ORDER BY nom;  
```

**Résultat :**

```
nom
--------
Alice  
Bob      ← Apparaît 2 fois  
Bob  
Charlie  
Diana  
Eve  
```

### UNION vs UNION ALL : Comparaison

| Critère | UNION | UNION ALL |
|---------|-------|-----------|
| **Doublons** | Éliminés | Conservés |
| **Performance** | Plus lent (tri + déduplication) | Plus rapide |
| **Cas d'usage** | Listes uniques | Comptages, tous les résultats |
| **Opération interne** | DISTINCT implicite | Simple concaténation |

### Quand utiliser UNION ALL ?

1. **Vous savez qu'il n'y a pas de doublons**
   ```sql
   -- Différentes années, pas de chevauchement possible
   SELECT * FROM ventes_2023
   UNION ALL
   SELECT * FROM ventes_2024;
   ```

2. **Les doublons ont du sens (comptages)**
   ```sql
   -- Compter toutes les transactions
   SELECT COUNT(*) FROM (
       SELECT montant FROM ventes_en_ligne
       UNION ALL
       SELECT montant FROM ventes_en_magasin
   ) AS toutes_ventes;
   ```

3. **Performance critique**
   ```sql
   -- Sur de grandes tables, UNION ALL peut être 2-10× plus rapide
   ```

---

## INTERSECT : Trouver les éléments communs

### Définition

`INTERSECT` retourne uniquement les lignes présentes dans **les deux** (ou tous les) ensembles de résultats.

### Syntaxe

```sql
SELECT colonnes FROM table1  
INTERSECT  
SELECT colonnes FROM table2;  
```

### Exemple : Clients qui sont aussi fournisseurs

```sql
SELECT nom FROM clients  
INTERSECT  
SELECT nom FROM fournisseurs;  
```

**Résultat :**

```
nom
-----
Bob
```

**Explication :** Seul Bob apparaît dans les deux tables.

### Visualisation

```
Clients: {Alice, Bob, Charlie}  
Fournisseurs: {Bob, Diana, Eve}  

Intersection: {Bob}
```

### Exemple avec critères

```sql
-- Employés actifs qui ont aussi été consultants
SELECT email FROM employes WHERE statut = 'actif'  
INTERSECT  
SELECT email FROM historique_consultants;  
```

---

## INTERSECT ALL : Conserver les doublons dans l'intersection

### Définition

`INTERSECT ALL` garde le **nombre minimal d'occurrences** de chaque ligne présente dans tous les ensembles.

### Exemple

```sql
-- Table commandes_en_ligne
INSERT INTO commandes_en_ligne (produit) VALUES
    ('Laptop'), ('Laptop'), ('Souris'), ('Clavier');

-- Table commandes_en_magasin
INSERT INTO commandes_en_magasin (produit) VALUES
    ('Laptop'), ('Souris'), ('Souris');
```

**INTERSECT (standard) :**

```sql
SELECT produit FROM commandes_en_ligne  
INTERSECT  
SELECT produit FROM commandes_en_magasin;  
```

**Résultat :**

```
produit
--------
Laptop  
Souris  
```

**INTERSECT ALL :**

```sql
SELECT produit FROM commandes_en_ligne  
INTERSECT ALL  
SELECT produit FROM commandes_en_magasin;  
```

**Résultat :**

```
produit
--------
Laptop   ← 1 occurrence (min de 2 et 1)  
Souris   ← 1 occurrence (min de 1 et 2)  
```

**Logique :**
- Laptop : 2 occurrences dans A, 1 dans B → min(2, 1) = 1
- Souris : 1 occurrence dans A, 2 dans B → min(1, 2) = 1
- Clavier : 1 occurrence dans A, 0 dans B → min(1, 0) = 0 (exclu)

---

## EXCEPT : Différence entre ensembles

### Définition

`EXCEPT` (ou `MINUS` dans certains SGBD) retourne les lignes du premier ensemble qui ne sont **pas** dans le second.

**Important :** L'ordre compte ! `A EXCEPT B ≠ B EXCEPT A`

### Syntaxe

```sql
SELECT colonnes FROM table1  
EXCEPT  
SELECT colonnes FROM table2;  
```

### Exemple : Clients qui ne sont pas fournisseurs

```sql
SELECT nom FROM clients  
EXCEPT  
SELECT nom FROM fournisseurs;  
```

**Résultat :**

```
nom
--------
Alice  
Charlie  
```

**Explication :** Bob est exclu car il est dans les deux tables.

### Exemple inverse : Fournisseurs qui ne sont pas clients

```sql
SELECT nom FROM fournisseurs  
EXCEPT  
SELECT nom FROM clients;  
```

**Résultat :**

```
nom
-----
Diana  
Eve  
```

### Visualisation

```
Clients: {Alice, Bob, Charlie}  
Fournisseurs: {Bob, Diana, Eve}  

Clients EXCEPT Fournisseurs: {Alice, Charlie}  
Fournisseurs EXCEPT Clients: {Diana, Eve}  
```

---

## EXCEPT ALL : Différence avec doublons

### Définition

`EXCEPT ALL` soustrait le nombre d'occurrences du second ensemble du premier.

### Exemple

```sql
-- Table A
INSERT INTO table_a (valeur) VALUES ('X'), ('X'), ('X'), ('Y'), ('Z');

-- Table B
INSERT INTO table_b (valeur) VALUES ('X'), ('X'), ('Z');
```

**EXCEPT (standard) :**

```sql
SELECT valeur FROM table_a  
EXCEPT  
SELECT valeur FROM table_b;  
```

**Résultat :**

```
valeur
------
Y      ← Seul Y n'est pas dans B
```

**EXCEPT ALL :**

```sql
SELECT valeur FROM table_a  
EXCEPT ALL  
SELECT valeur FROM table_b;  
```

**Résultat :**

```
valeur
------
X      ← 3 occurrences dans A - 2 dans B = 1  
Y      ← 1 occurrence dans A - 0 dans B = 1  
```

**Logique :**
- X : 3 dans A, 2 dans B → 3 - 2 = 1 occurrence dans le résultat
- Y : 1 dans A, 0 dans B → 1 - 0 = 1 occurrence
- Z : 1 dans A, 1 dans B → 1 - 1 = 0 (exclu)

---

## Comparaison des opérations

### Tableau récapitulatif

| Opération | Action | Doublons éliminés | Cas d'usage typique |
|-----------|--------|-------------------|---------------------|
| **UNION** | A ∪ B | Oui | Listes uniques combinées |
| **UNION ALL** | A + B | Non | Toutes les données, comptages |
| **INTERSECT** | A ∩ B | Oui | Éléments communs (liste unique) |
| **INTERSECT ALL** | A ∩ B (avec occurrences) | Non | Comptage d'éléments communs |
| **EXCEPT** | A − B | Oui | Différences (liste unique) |
| **EXCEPT ALL** | A − B (avec occurrences) | Non | Différences avec multiplicité |

### Diagramme de Venn complet

```
Ensemble A = {1, 2, 3, 4, 5}  
Ensemble B = {4, 5, 6, 7, 8}  

┌─────────────────────────────────────┐
│         A UNION B                   │
│  {1, 2, 3, 4, 5, 6, 7, 8}           │
│                                     │
│  ┌─────────┐      ┌─────────┐       │
│  │ A EXCEPT B    B EXCEPT A │       │
│  │ {1,2,3} │      │ {6,7,8} │       │
│  │         │      │         │       │
│  │    ┌────┴──────┴────┐    │       │
│  │    │  A INTERSECT B │    │       │
│  │    │    {4, 5}      │    │       │
│  └────┴────────────────┴────┘       │
│                                     │
└─────────────────────────────────────┘
```

---

## Cas d'usage pratiques

### 1. Combiner des données de sources multiples (UNION)

**Scénario :** Vous avez des données clients dans plusieurs bases.

```sql
-- Base Europe
SELECT client_id, nom, email, 'Europe' AS region  
FROM clients_europe  
WHERE actif = TRUE  

UNION

-- Base USA
SELECT client_id, nom, email, 'USA' AS region  
FROM clients_usa  
WHERE actif = TRUE  

UNION

-- Base Asie
SELECT client_id, nom, email, 'Asie' AS region  
FROM clients_asie  
WHERE actif = TRUE  

ORDER BY nom;
```

### 2. Identifier les utilisateurs actifs sur plusieurs plateformes (INTERSECT)

**Scénario :** Trouver les utilisateurs présents sur le site web ET l'application mobile.

```sql
SELECT user_id  
FROM utilisateurs_web  
WHERE derniere_connexion > CURRENT_DATE - INTERVAL '30 days'  

INTERSECT

SELECT user_id  
FROM utilisateurs_mobile  
WHERE derniere_connexion > CURRENT_DATE - INTERVAL '30 days';  
```

### 3. Trouver les produits jamais commandés (EXCEPT)

**Scénario :** Identifier les produits du catalogue qui n'ont jamais été achetés.

```sql
SELECT produit_id, nom  
FROM catalogue_produits  

EXCEPT

SELECT DISTINCT produit_id, nom  
FROM commandes  
INNER JOIN catalogue_produits USING (produit_id);  
```

### 4. Audit de synchronisation entre bases (EXCEPT)

**Scénario :** Vérifier quels enregistrements sont dans la base principale mais pas dans la réplique.

```sql
-- Manquant dans la réplique
SELECT id, hash_donnees FROM base_principale  
EXCEPT  
SELECT id, hash_donnees FROM base_replique;  

-- Manquant dans la principale (ne devrait pas arriver)
SELECT id, hash_donnees FROM base_replique  
EXCEPT  
SELECT id, hash_donnees FROM base_principale;  
```

### 5. Analyser des tendances temporelles (UNION ALL)

**Scénario :** Comparer les ventes par mois.

```sql
SELECT
    'Janvier 2024' AS periode,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS total
FROM ventes  
WHERE date_vente BETWEEN '2024-01-01' AND '2024-01-31'  

UNION ALL

SELECT
    'Février 2024',
    COUNT(*),
    SUM(montant)
FROM ventes  
WHERE date_vente BETWEEN '2024-02-01' AND '2024-02-29'  

UNION ALL

SELECT
    'Mars 2024',
    COUNT(*),
    SUM(montant)
FROM ventes  
WHERE date_vente BETWEEN '2024-03-01' AND '2024-03-31'  

ORDER BY periode;
```

### 6. Listes blanches / listes noires

**Scénario :** Exclure les utilisateurs bannis d'une liste de destinataires.

```sql
-- Tous les utilisateurs actifs
SELECT email FROM utilisateurs WHERE statut = 'actif'

EXCEPT

-- Sauf les bannis
SELECT email FROM utilisateurs_bannis;
```

---

## Combinaisons multiples

Vous pouvez chaîner plusieurs opérations d'ensemble.

### Ordre de priorité

**Sans parenthèses**, l'ordre d'évaluation est de gauche à droite, avec cette priorité :
1. `INTERSECT` (priorité la plus haute)  
2. `UNION` et `EXCEPT` (même priorité, gauche à droite)

### Exemple : Trois ensembles

```sql
-- Clients actifs OU anciens employés SAUF bannis
SELECT email FROM clients WHERE actif = TRUE

UNION

SELECT email FROM anciens_employes

EXCEPT

SELECT email FROM liste_bannis

ORDER BY email;
```

### Utiliser des parenthèses pour clarifier

```sql
-- Plus explicite avec parenthèses
(
    SELECT email FROM clients WHERE actif = TRUE
    UNION
    SELECT email FROM anciens_employes
)
EXCEPT  
SELECT email FROM liste_bannis  

ORDER BY email;
```

### Exemple complexe : Analyse multi-critères

```sql
-- Utilisateurs VIP : (Achat > 1000€ OU Membre depuis > 5 ans) ET Pas d'incidents
(
    -- Gros acheteurs
    SELECT user_id FROM achats
    GROUP BY user_id
    HAVING SUM(montant) > 1000

    UNION

    -- Membres anciens
    SELECT user_id FROM utilisateurs
    WHERE date_inscription < CURRENT_DATE - INTERVAL '5 years'
)
INTERSECT
(
    -- Tous les utilisateurs
    SELECT user_id FROM utilisateurs

    EXCEPT

    -- Sauf ceux avec incidents
    SELECT user_id FROM incidents
)
ORDER BY user_id;
```

---

## Performance et optimisation

### 1. UNION vs UNION ALL : Impact mesurable

```sql
-- Test de performance
EXPLAIN ANALYZE  
SELECT nom FROM grande_table_1  
UNION  -- Déduplication coûteuse  
SELECT nom FROM grande_table_2;  

EXPLAIN ANALYZE  
SELECT nom FROM grande_table_1  
UNION ALL  -- Simple concaténation  
SELECT nom FROM grande_table_2;  
```

**Observation typique :**
- `UNION` : Unique + Sort (coûteux sur grandes tables)  
- `UNION ALL` : Append (très rapide)

**Règle :** Si vous **savez** qu'il n'y a pas de doublons (tables partitionnées par date, par exemple), utilisez `UNION ALL`.

### 2. Index et opérations d'ensemble

Les index sur les tables sources peuvent accélérer les requêtes individuelles, mais les opérations d'ensemble elles-mêmes ne bénéficient pas directement des index.

**Optimisation :**
```sql
-- ✅ Filtrer AVANT l'opération d'ensemble
SELECT email FROM clients  
WHERE actif = TRUE  -- Index sur actif  
UNION  
SELECT email FROM prospects  
WHERE statut = 'qualifié';  -- Index sur statut  

-- ❌ Filtrer APRÈS (moins efficace)
SELECT email FROM (
    SELECT email, actif FROM clients
    UNION
    SELECT email, actif FROM prospects
) AS combine
WHERE actif = TRUE;
```

### 3. Utiliser des CTE pour la lisibilité

```sql
WITH  
clients_actifs AS (  
    SELECT email FROM clients WHERE actif = TRUE
),
prospects_qualifies AS (
    SELECT email FROM prospects WHERE statut = 'qualifié'
),
liste_bannis AS (
    SELECT email FROM bannis
)
-- Opérations d'ensemble sur les CTE
SELECT email FROM clients_actifs  
UNION  
SELECT email FROM prospects_qualifies  
EXCEPT  
SELECT email FROM liste_bannis  
ORDER BY email;  
```

### 4. Matérialiser les résultats coûteux

Si vous utilisez le même ensemble plusieurs fois :

```sql
CREATE TEMP TABLE emails_actifs AS  
SELECT email FROM clients WHERE actif = TRUE  
UNION  
SELECT email FROM prospects WHERE statut = 'qualifié';  

CREATE INDEX idx_email ON emails_actifs(email);

-- Utiliser ensuite
SELECT * FROM emails_actifs WHERE email LIKE '%@example.com';
```

---

## UNION vs JOIN : Quelle différence ?

C'est une confusion courante pour les débutants.

### UNION : Combinaison verticale (lignes)

```sql
SELECT nom FROM employes     -- Résultat : 100 lignes  
UNION  
SELECT nom FROM consultants  -- Résultat : 50 lignes  
-- Total : ~150 lignes (moins si doublons)
```

**Résultat :** Plus de **lignes** (empilement vertical).

### JOIN : Combinaison horizontale (colonnes)

```sql
SELECT e.nom, d.nom_departement  
FROM employes e  
INNER JOIN departements d ON e.dept_id = d.id  
-- Total : 100 lignes avec plus de colonnes
```

**Résultat :** Plus de **colonnes** (extension horizontale).

### Visualisation

```
UNION (vertical) :  
Table 1    →    Résultat  
Alice            Alice  
Bob              Bob  
                 Charlie  ← De Table 2
Table 2          Diana    ← De Table 2  
Charlie  
Diana  


JOIN (horizontal) :  
Table 1    +    Table 2    →    Résultat  
Alice  1        1  IT           Alice  IT  
Bob    2        2  RH           Bob    RH  
```

---

## Alternatives aux opérations d'ensemble

### UNION → FULL OUTER JOIN

```sql
-- Avec UNION
SELECT nom FROM clients  
UNION  
SELECT nom FROM fournisseurs;  

-- Équivalent avec FULL OUTER JOIN (si clé commune)
SELECT COALESCE(c.nom, f.nom) AS nom  
FROM clients c  
FULL OUTER JOIN fournisseurs f ON c.id = f.id;  
```

### INTERSECT → INNER JOIN

```sql
-- Avec INTERSECT
SELECT email FROM clients  
INTERSECT  
SELECT email FROM newsletter_subscribers;  

-- Équivalent avec INNER JOIN
SELECT DISTINCT c.email  
FROM clients c  
INNER JOIN newsletter_subscribers n ON c.email = n.email;  
```

### EXCEPT → NOT EXISTS

```sql
-- Avec EXCEPT
SELECT email FROM clients  
EXCEPT  
SELECT email FROM bannis;  

-- Équivalent avec NOT EXISTS
SELECT email  
FROM clients c  
WHERE NOT EXISTS (  
    SELECT 1 FROM bannis b WHERE b.email = c.email
);
```

**Performances :** Les équivalences avec JOIN peuvent être plus rapides avec les bons index.

---

## Pièges courants et erreurs

### 1. Nombre de colonnes incompatible

```sql
-- ❌ ERREUR
SELECT id, nom FROM employes  
UNION  
SELECT id FROM consultants;  -- Manque "nom"  
```

**Erreur PostgreSQL :**
```
ERROR: each UNION query must have the same number of columns
```

**Solution :**
```sql
-- ✅ CORRECT
SELECT id, nom FROM employes  
UNION  
SELECT id, nom FROM consultants;  

-- ✅ ALTERNATIVE : Ajouter NULL si colonne manquante
SELECT id, nom FROM employes  
UNION  
SELECT id, NULL AS nom FROM consultants;  
```

### 2. Types de données incompatibles

```sql
-- ❌ ERREUR
SELECT id, nom FROM employes  -- INTEGER, TEXT  
UNION  
SELECT nom, id FROM consultants;  -- TEXT, INTEGER (ordre inversé)  
```

**Solution :**
```sql
-- ✅ CORRECT : Même ordre
SELECT id, nom FROM employes  
UNION  
SELECT id, nom FROM consultants;  
```

### 3. Oublier ORDER BY à la fin

```sql
-- ❌ ERREUR
SELECT nom FROM employes ORDER BY nom  
UNION  
SELECT nom FROM consultants;  
```

**Solution :**
```sql
-- ✅ CORRECT
SELECT nom FROM employes  
UNION  
SELECT nom FROM consultants  
ORDER BY nom;  -- À la fin !  
```

### 4. Confondre EXCEPT et NOT IN avec NULL

```sql
-- Piège avec NULL dans EXCEPT
SELECT id FROM table_a  -- Contient 1, 2, 3, NULL  
EXCEPT  
SELECT id FROM table_b;  -- Contient 2, 3  

-- Résultat : 1, NULL (NULL est conservé dans EXCEPT)
```

**Avec NOT IN (différent) :**
```sql
-- NULL pose problème avec NOT IN
SELECT id FROM table_a  
WHERE id NOT IN (SELECT id FROM table_b);  

-- Résultat : 1 seulement (NULL est exclu à cause de la logique ternaire)
```

### 5. Performance de UNION quand UNION ALL suffirait

```sql
-- ❌ LENT : Déduplication inutile
SELECT * FROM ventes_2023  
UNION  
SELECT * FROM ventes_2024;  -- Pas de chevauchement possible !  

-- ✅ RAPIDE
SELECT * FROM ventes_2023  
UNION ALL  
SELECT * FROM ventes_2024;  
```

---

## Exemples avancés

### 1. Rapport multi-sources avec étiquettes

```sql
SELECT
    'Client' AS type,
    nom,
    email,
    date_creation
FROM clients

UNION ALL

SELECT
    'Prospect' AS type,
    nom,
    email,
    date_creation
FROM prospects

UNION ALL

SELECT
    'Partenaire' AS type,
    nom,
    email,
    date_signature AS date_creation
FROM partenaires

ORDER BY date_creation DESC;
```

### 2. Analyse de cohérence entre environnements

```sql
-- Différences entre production et staging
(
    SELECT 'Prod seulement' AS source, id, nom
    FROM production.utilisateurs
    EXCEPT
    SELECT 'Prod seulement', id, nom
    FROM staging.utilisateurs
)
UNION ALL
(
    SELECT 'Staging seulement', id, nom
    FROM staging.utilisateurs
    EXCEPT
    SELECT 'Staging seulement', id, nom
    FROM production.utilisateurs
)
ORDER BY source, id;
```

### 3. Top N par catégorie consolidé

```sql
-- Top 5 par catégorie, toutes catégories
(
    SELECT 'Électronique' AS categorie, nom, ventes
    FROM produits
    WHERE categorie = 'Électronique'
    ORDER BY ventes DESC
    LIMIT 5
)
UNION ALL
(
    SELECT 'Mode', nom, ventes
    FROM produits
    WHERE categorie = 'Mode'
    ORDER BY ventes DESC
    LIMIT 5
)
UNION ALL
(
    SELECT 'Alimentation', nom, ventes
    FROM produits
    WHERE categorie = 'Alimentation'
    ORDER BY ventes DESC
    LIMIT 5
)
ORDER BY categorie, ventes DESC;
```

### 4. Construction de rapports dynamiques

```sql
-- Totaux par ligne + ligne de total général
SELECT
    departement,
    SUM(salaire) AS total
FROM employes  
GROUP BY departement  

UNION ALL

SELECT
    'TOTAL GÉNÉRAL' AS departement,
    SUM(salaire)
FROM employes

ORDER BY
    CASE WHEN departement = 'TOTAL GÉNÉRAL' THEN 1 ELSE 0 END,
    departement;
```

### 5. Détection d'anomalies entre tables

```sql
-- Commandes sans client correspondant
SELECT
    'Commande orpheline' AS anomalie,
    c.id,
    c.client_id
FROM commandes c  
LEFT JOIN clients cl ON c.client_id = cl.id  
WHERE cl.id IS NULL  

UNION ALL

-- Clients sans commande
SELECT
    'Client sans commande',
    cl.id,
    NULL
FROM clients cl  
LEFT JOIN commandes c ON cl.id = c.client_id  
WHERE c.id IS NULL  

ORDER BY anomalie, id;
```

---

## Bonnes pratiques

### ✅ À faire

1. **Utiliser UNION ALL quand les doublons n'existent pas**
   ```sql
   -- Tables partitionnées par année
   SELECT * FROM ventes_2023
   UNION ALL
   SELECT * FROM ventes_2024;
   ```

2. **Filtrer avant l'opération d'ensemble**
   ```sql
   -- ✅ Efficace
   SELECT email FROM clients WHERE actif = TRUE
   UNION
   SELECT email FROM prospects WHERE statut = 'qualifié';
   ```

3. **Utiliser des CTE pour la lisibilité**
   ```sql
   WITH a AS (...), b AS (...)
   SELECT * FROM a
   UNION
   SELECT * FROM b;
   ```

4. **Nommer clairement les colonnes dans la première requête**
   ```sql
   SELECT id AS utilisateur_id, nom AS nom_complet FROM clients
   UNION
   SELECT id, nom FROM prospects;
   -- Colonnes nommées : utilisateur_id, nom_complet
   ```

5. **Ajouter une colonne "source" pour traçabilité**
   ```sql
   SELECT 'DB1' AS source, * FROM db1.table
   UNION ALL
   SELECT 'DB2', * FROM db2.table;
   ```

### ❌ À éviter

1. **UNION quand UNION ALL suffit**  
   → Perte de performance inutile

2. **Opérations d'ensemble sur de très grandes tables sans filtrage**
   ```sql
   -- ❌ Lent
   SELECT * FROM huge_table_1
   UNION
   SELECT * FROM huge_table_2;

   -- ✅ Mieux
   SELECT * FROM huge_table_1 WHERE date > '2024-01-01'
   UNION
   SELECT * FROM huge_table_2 WHERE date > '2024-01-01';
   ```

3. **Oublier ORDER BY pour un résultat déterministe**

4. **Confondre UNION (vertical) et JOIN (horizontal)**

5. **Utiliser EXCEPT avec des colonnes comportant beaucoup de NULL sans attention**

---

## Résumé

### Points clés à retenir

| Opération | Fonction | Élimine doublons | Équivalent mathématique |
|-----------|----------|------------------|------------------------|
| **UNION** | Combine ensembles | Oui | A ∪ B |
| **UNION ALL** | Combine tout | Non | A + B |
| **INTERSECT** | Éléments communs | Oui | A ∩ B |
| **INTERSECT ALL** | Communs avec occurrences | Non | min(A, B) |
| **EXCEPT** | Différence | Oui | A − B |
| **EXCEPT ALL** | Différence avec occurrences | Non | A − B (par occurrence) |

### Règles essentielles

1. **Même nombre de colonnes** et types compatibles  
2. **ORDER BY à la fin** de toutes les opérations  
3. **UNION ALL plus rapide** que UNION (pas de tri)  
4. **EXCEPT n'est pas commutatif** : A EXCEPT B ≠ B EXCEPT A  
5. **Utiliser parenthèses** pour clarifier les combinaisons multiples

### Quand utiliser quoi ?

- **Listes combinées uniques** → UNION  
- **Toutes les données** → UNION ALL  
- **Éléments présents partout** → INTERSECT  
- **Éléments absents du second ensemble** → EXCEPT  
- **Audit/comparaison** → EXCEPT des deux côtés

### Checklist

- [ ] Mes requêtes ont-elles le même nombre de colonnes ?  
- [ ] Les types de données sont-ils compatibles ?  
- [ ] Ai-je besoin d'éliminer les doublons ou pas ?  
- [ ] Ai-je placé ORDER BY à la fin ?  
- [ ] Ai-je filtré les données AVANT l'opération d'ensemble ?  
- [ ] Ai-je testé avec EXPLAIN ANALYZE ?

---

## Pour aller plus loin

- **Jointures** (Chapitre 7) : Combinaisons horizontales vs verticales  
- **Sous-requêtes** (Chapitre 9.1) : Alternatives avec IN, EXISTS  
- **CTE** (Chapitre 9.2) : Améliorer la lisibilité des combinaisons complexes  
- **Window Functions** (Chapitre 10) : Agrégations sans UNION

---

**Prochain chapitre :** 9.5. CASE expressions et logique conditionnelle

---


⏭️ [CASE expressions et logique conditionnelle](/09-techniques-sql-avancees/05-case-expressions.md)
