🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.4. Fonctions de rang (RANK, DENSE_RANK, ROW_NUMBER, NTILE)

## Introduction

Les **fonctions de rang** sont parmi les window functions les plus populaires et les plus utiles. Elles permettent de **classer** et **numéroter** les lignes selon un ordre défini, ce qui est essentiel pour de nombreuses analyses :

- Top N par catégorie (meilleurs vendeurs, produits les plus vendus)
- Classements et podiums
- Pagination avancée
- Distribution en quantiles
- Élimination de doublons

PostgreSQL propose quatre fonctions de rang principales, chacune avec un comportement spécifique face aux égalités :

1. **ROW_NUMBER()** : Numérotation stricte, toujours unique  
2. **RANK()** : Classement avec "sauts" en cas d'égalité  
3. **DENSE_RANK()** : Classement sans "sauts" en cas d'égalité  
4. **NTILE()** : Répartition en groupes (quantiles, quartiles, etc.)

## ROW_NUMBER() : Numérotation Simple

### Principe

`ROW_NUMBER()` attribue un **numéro séquentiel unique** à chaque ligne, en suivant l'ordre défini par `ORDER BY`.

**Caractéristique clé** : Même si deux lignes ont des valeurs identiques, elles reçoivent des numéros **différents**.

### Syntaxe

```sql
ROW_NUMBER() OVER (
    [PARTITION BY colonne_groupe]
    ORDER BY colonne_tri
)
```

### Exemple de Base

```sql
SELECT
    vendeur,
    montant,
    ROW_NUMBER() OVER (ORDER BY montant DESC) AS numero
FROM ventes;
```

**Données** :

```
| vendeur  | montant |
|----------|---------|
| Alice    | 1200    |
| Caroline | 1200    | ← Même montant qu'Alice
| Bob      | 350     |
| David    | 150     |
| Eve      | 150     | ← Même montant que David
```

**Résultat** :

```
| vendeur  | montant | numero |
|----------|---------|--------|
| Alice    | 1200    | 1      |
| Caroline | 1200    | 2      | ← Numéro différent malgré l'égalité
| Bob      | 350     | 3      |
| David    | 150     | 4      |
| Eve      | 150     | 5      |
```

**Observation** : Alice et Caroline ont le même montant (1200), mais des numéros différents (1 et 2). L'ordre entre elles est **arbitraire** (dépend de l'ordre physique des lignes ou d'un tri secondaire).

### Utilisation avec PARTITION BY

Le cas d'usage le plus puissant : numéroter **par groupe** :

```sql
SELECT
    categorie,
    produit,
    prix,
    ROW_NUMBER() OVER (
        PARTITION BY categorie
        ORDER BY prix DESC
    ) AS rang_categorie
FROM produits;
```

**Données** :

```
| categorie | produit      | prix  |
|-----------|--------------|-------|
| Laptop    | MacBook      | 2000  |
| Laptop    | ThinkPad     | 1500  |
| Laptop    | Chromebook   | 500   |
| Souris    | MX Master    | 80    |
| Souris    | Magic Mouse  | 70    |
| Souris    | Basic        | 10    |
```

**Résultat** :

```
| categorie | produit      | prix  | rang_categorie |
|-----------|--------------|-------|----------------|
| Laptop    | MacBook      | 2000  | 1              |
| Laptop    | ThinkPad     | 1500  | 2              |
| Laptop    | Chromebook   | 500   | 3              |
| Souris    | MX Master    | 80    | 1              | ← Recommence à 1
| Souris    | Magic Mouse  | 70    | 2              |
| Souris    | Basic        | 10    | 3              |
```

**Point clé** : La numérotation recommence à 1 pour chaque catégorie.

### Cas d'Usage Typiques

#### 1. Top N par Catégorie

Trouver les 3 meilleurs produits de chaque catégorie :

```sql
SELECT * FROM (
    SELECT
        categorie,
        produit,
        prix,
        ROW_NUMBER() OVER (
            PARTITION BY categorie
            ORDER BY prix DESC
        ) AS rang
    FROM produits
) AS classement
WHERE rang <= 3;
```

#### 2. Éliminer les Doublons

Garder seulement la première occurrence de chaque doublon :

```sql
DELETE FROM clients  
WHERE id IN (  
    SELECT id FROM (
        SELECT
            id,
            ROW_NUMBER() OVER (
                PARTITION BY email
                ORDER BY date_creation
            ) AS rn
        FROM clients
    ) AS sub
    WHERE rn > 1
);
```

Cela conserve le client le plus ancien pour chaque email.

#### 3. Pagination Avancée

Alternative à `OFFSET` (plus performante sur de grandes tables) :

```sql
SELECT * FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (ORDER BY date_creation) AS rn
    FROM articles
) AS numbered
WHERE rn BETWEEN 21 AND 30;  -- Page 3 (10 articles par page)
```

## RANK() : Classement avec Sauts

### Principe

`RANK()` attribue le **même rang** aux lignes ayant des **valeurs égales**, puis effectue un **saut** dans la numérotation.

**Métaphore** : Comme aux Jeux Olympiques, si deux athlètes arrivent ex-aequo en 1ère position, ils sont tous deux classés 1er, et le suivant est classé 3ème (il n'y a pas de 2ème).

### Syntaxe

```sql
RANK() OVER (
    [PARTITION BY colonne_groupe]
    ORDER BY colonne_tri
)
```

### Exemple Démonstratif

```sql
SELECT
    vendeur,
    montant,
    RANK() OVER (ORDER BY montant DESC) AS rang
FROM ventes;
```

**Données** :

```
| vendeur  | montant |
|----------|---------|
| Alice    | 1200    |
| Caroline | 1200    | ← Égalité avec Alice
| Bob      | 350     |
| David    | 150     |
| Eve      | 150     | ← Égalité avec David
```

**Résultat** :

```
| vendeur  | montant | rang |
|----------|---------|------|
| Alice    | 1200    | 1    |
| Caroline | 1200    | 1    | ← Même rang qu'Alice
| Bob      | 350     | 3    | ← Saut : 2 personnes en position 1
| David    | 150     | 4    |
| Eve      | 150     | 4    | ← Même rang que David
```

**Observation** :
- Alice et Caroline (égalité) : rang 1
- Bob : rang **3** (pas 2, car 2 personnes occupent la 1ère place)
- David et Eve (égalité) : rang 4

### Différence Clé : RANK vs ROW_NUMBER

| Fonction | Alice (1200) | Caroline (1200) | Bob (350) | David (150) | Eve (150) |
|----------|--------------|-----------------|-----------|-------------|-----------|
| **RANK()** | 1 | 1 | **3** | 4 | 4 |
| **ROW_NUMBER()** | 1 | **2** | 3 | 4 | **5** |

- `RANK()` : Égalités = même rang, puis **saut**  
- `ROW_NUMBER()` : Chaque ligne a un numéro **unique**

### Utilisation Pratique

#### Trouver les Ex-Aequo

```sql
SELECT
    joueur,
    score,
    RANK() OVER (ORDER BY score DESC) AS classement
FROM resultats  
WHERE RANK() OVER (ORDER BY score DESC) = 1;  -- Tous les 1ers  
```

#### Top N avec Égalités Incluses

Si vous voulez les "3 meilleurs" mais que le 3ème est ex-aequo avec d'autres, `RANK()` les inclut tous :

```sql
SELECT * FROM (
    SELECT
        produit,
        ventes,
        RANK() OVER (ORDER BY ventes DESC) AS rang
    FROM produits
) AS classement
WHERE rang <= 3;
```

Si 3 produits sont à égalité en 3ème position, ils seront tous inclus (vous aurez plus de 3 lignes).

## DENSE_RANK() : Classement Sans Sauts

### Principe

`DENSE_RANK()` fonctionne comme `RANK()`, mais **sans sauts** dans la numérotation. Les rangs sont **consécutifs**.

**Métaphore** : Comme un classement de films par note. Si deux films ont 5 étoiles (rang 1), le suivant avec 4 étoiles est rang 2 (pas 3).

### Syntaxe

```sql
DENSE_RANK() OVER (
    [PARTITION BY colonne_groupe]
    ORDER BY colonne_tri
)
```

### Exemple Démonstratif

```sql
SELECT
    vendeur,
    montant,
    DENSE_RANK() OVER (ORDER BY montant DESC) AS rang_dense
FROM ventes;
```

**Données** : (mêmes que précédemment)

```
| vendeur  | montant |
|----------|---------|
| Alice    | 1200    |
| Caroline | 1200    |
| Bob      | 350     |
| David    | 150     |
| Eve      | 150     |
```

**Résultat** :

```
| vendeur  | montant | rang_dense |
|----------|---------|------------|
| Alice    | 1200    | 1          |
| Caroline | 1200    | 1          | ← Même rang
| Bob      | 350     | 2          | ← Pas de saut !
| David    | 150     | 3          | ← Consécutif
| Eve      | 150     | 3          |
```

**Observation** : Les rangs sont 1, 2, 3 (pas de trou), même avec des égalités.

### Comparaison des Trois Fonctions

| Fonction | Alice (1200) | Caroline (1200) | Bob (350) | David (150) | Eve (150) |
|----------|--------------|-----------------|-----------|-------------|-----------|
| **ROW_NUMBER()** | 1 | 2 | 3 | 4 | 5 |
| **RANK()** | 1 | 1 | **3** | 4 | 4 |
| **DENSE_RANK()** | 1 | 1 | **2** | **3** | 3 |

**Résumé** :
- `ROW_NUMBER()` : 1, 2, 3, 4, 5 (toujours unique)  
- `RANK()` : 1, 1, **3**, 4, 4 (avec sauts)  
- `DENSE_RANK()` : 1, 1, **2**, **3**, 3 (sans sauts)

### Quand Utiliser DENSE_RANK

Utilisez `DENSE_RANK()` quand :
- Vous voulez un classement **compact** sans trous
- Le nombre de "niveaux" distincts est important
- Vous créez des catégories (A, B, C...) basées sur le rang

**Exemple** : Attribuer des badges "Or", "Argent", "Bronze" :

```sql
SELECT
    vendeur,
    ventes,
    DENSE_RANK() OVER (ORDER BY ventes DESC) AS niveau,
    CASE DENSE_RANK() OVER (ORDER BY ventes DESC)
        WHEN 1 THEN 'Or'
        WHEN 2 THEN 'Argent'
        WHEN 3 THEN 'Bronze'
        ELSE 'Participant'
    END AS badge
FROM vendeurs;
```

## NTILE() : Répartition en Groupes

### Principe

`NTILE(n)` divise les lignes en **n groupes** (ou "buckets") de taille approximativement égale, et attribue un numéro de groupe (de 1 à n).

**Métaphore** : Diviser une classe en 4 groupes de niveau (quartiles) : les 25% meilleurs, les 25% suivants, etc.

### Syntaxe

```sql
NTILE(nombre_de_groupes) OVER (
    [PARTITION BY colonne_groupe]
    ORDER BY colonne_tri
)
```

### Exemple : Quartiles

Diviser les vendeurs en 4 groupes de performance :

```sql
SELECT
    vendeur,
    ventes,
    NTILE(4) OVER (ORDER BY ventes DESC) AS quartile
FROM vendeurs;
```

**Données** (12 vendeurs) :

```
| vendeur | ventes |
|---------|--------|
| Alice   | 10000  |
| Bob     | 9500   |
| Carol   | 9000   |
| David   | 8500   | ← Fin Q1 (3 personnes)
| Eve     | 8000   |
| Frank   | 7500   |
| Grace   | 7000   | ← Fin Q2 (3 personnes)
| Henry   | 6500   |
| Irene   | 6000   |
| Jack    | 5500   | ← Fin Q3 (3 personnes)
| Kelly   | 5000   |
| Leo     | 4500   |
| Mia     | 4000   | ← Fin Q4 (3 personnes)
```

**Résultat** :

```
| vendeur | ventes | quartile |
|---------|--------|----------|
| Alice   | 10000  | 1        | ← Top 25%
| Bob     | 9500   | 1        |
| Carol   | 9000   | 1        |
| David   | 8500   | 2        | ← 25-50%
| Eve     | 8000   | 2        |
| Frank   | 7500   | 2        |
| Grace   | 7000   | 3        | ← 50-75%
| Henry   | 6500   | 3        |
| Irene   | 6000   | 3        |
| Jack    | 5500   | 4        | ← Bottom 25%
| Kelly   | 5000   | 4        |
| Leo     | 4500   | 4        |
| Mia     | 4000   | 4        |
```

**Point clé** : `NTILE(4)` crée 4 groupes de 3 personnes chacun (12 ÷ 4 = 3).

### Gestion des Divisions Inégales

Si le nombre de lignes n'est pas divisible par n, les premiers groupes auront **une ligne de plus** :

**Exemple** : 10 lignes divisées en 4 groupes (NTILE(4))

```
10 ÷ 4 = 2 reste 2
→ Les 2 premiers groupes auront 3 lignes
→ Les 2 derniers groupes auront 2 lignes
```

**Résultat** :
- Groupe 1 : 3 lignes
- Groupe 2 : 3 lignes
- Groupe 3 : 2 lignes
- Groupe 4 : 2 lignes

### Cas d'Usage de NTILE

#### 1. Segmentation Client (Déciles)

Diviser les clients en 10 groupes selon leur valeur :

```sql
SELECT
    client_id,
    ca_total,
    NTILE(10) OVER (ORDER BY ca_total DESC) AS decile
FROM clients;
```

- Décile 1 : Top 10% des clients (VIP)
- Décile 10 : Bottom 10% (clients à faible valeur)

#### 2. Analyse ABC

Classement ABC en 3 catégories :

```sql
SELECT
    produit,
    ventes,
    NTILE(3) OVER (ORDER BY ventes DESC) AS categorie_abc,
    CASE NTILE(3) OVER (ORDER BY ventes DESC)
        WHEN 1 THEN 'A - Haute valeur'
        WHEN 2 THEN 'B - Valeur moyenne'
        WHEN 3 THEN 'C - Faible valeur'
    END AS classification
FROM produits;
```

#### 3. Distribution Équitable

Répartir équitablement des tâches entre des travailleurs :

```sql
SELECT
    tache_id,
    duree_estimee,
    NTILE(5) OVER (ORDER BY duree_estimee) AS worker_id
FROM taches;
```

Attribue chaque tâche à un worker (1 à 5) en équilibrant la charge.

## Comparaison Complète des Quatre Fonctions

### Tableau Récapitulatif

| Fonction | Gestion des Égalités | Numéros Uniques | Sauts | Groupes | Usage Principal |
|----------|---------------------|-----------------|-------|---------|-----------------|
| **ROW_NUMBER()** | Ignore (arbitraire) | Oui | Non | Non | Numérotation unique, pagination |
| **RANK()** | Même rang | Non | Oui | Non | Classements sportifs, podiums |
| **DENSE_RANK()** | Même rang | Non | Non | Non | Classements par niveaux |
| **NTILE()** | Distribue équitablement | Non | N/A | Oui | Segmentation, quantiles |

### Exemple avec Toutes les Fonctions

```sql
SELECT
    vendeur,
    ventes,
    ROW_NUMBER() OVER (ORDER BY ventes DESC) AS row_num,
    RANK() OVER (ORDER BY ventes DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY ventes DESC) AS dense_rank,
    NTILE(3) OVER (ORDER BY ventes DESC) AS tertile
FROM vendeurs;
```

**Données** :

```
| vendeur | ventes |
|---------|--------|
| Alice   | 1000   |
| Bob     | 1000   | ← Égalité
| Carol   | 800    |
| David   | 600    |
| Eve     | 600    | ← Égalité
| Frank   | 400    |
```

**Résultat** :

```
| vendeur | ventes | row_num | rank | dense_rank | tertile |
|---------|--------|---------|------|------------|---------|
| Alice   | 1000   | 1       | 1    | 1          | 1       |
| Bob     | 1000   | 2       | 1    | 1          | 1       |
| Carol   | 800    | 3       | 3    | 2          | 2       |
| David   | 600    | 4       | 4    | 3          | 2       |
| Eve     | 600    | 5       | 4    | 3          | 3       |
| Frank   | 400    | 6       | 6    | 4          | 3       |
```

**Observations** :
- `row_num` : 1, 2, 3, 4, 5, 6 (tous uniques)  
- `rank` : 1, 1, **3**, 4, 4, **6** (avec sauts)  
- `dense_rank` : 1, 1, **2**, 3, 3, **4** (sans sauts)  
- `tertile` : 1, 1, 2, 2, 3, 3 (6 lignes ÷ 3 groupes = 2 par groupe)

## Techniques Avancées

### 1. Rang par Catégorie (Combinaison avec PARTITION BY)

Le plus souvent, vous utiliserez ces fonctions **avec PARTITION BY** pour classer **dans chaque groupe** :

```sql
SELECT
    region,
    vendeur,
    ventes,
    RANK() OVER (
        PARTITION BY region
        ORDER BY ventes DESC
    ) AS rang_regional
FROM vendeurs;
```

Cela donne le classement de chaque vendeur **au sein de sa région**.

### 2. Filtrer sur le Rang (Sous-requête ou CTE)

Pour obtenir seulement le top 3 par catégorie :

```sql
WITH classement AS (
    SELECT
        categorie,
        produit,
        ventes,
        RANK() OVER (
            PARTITION BY categorie
            ORDER BY ventes DESC
        ) AS rang
    FROM produits
)
SELECT *  
FROM classement  
WHERE rang <= 3;  
```

**Important** : Vous ne pouvez pas filtrer directement dans `WHERE` avec une window function. Utilisez une sous-requête ou CTE.

### 3. Percentiles Personnalisés

Calculer dans quel percentile se trouve chaque valeur :

```sql
SELECT
    client_id,
    ca_annuel,
    NTILE(100) OVER (ORDER BY ca_annuel DESC) AS percentile,
    CASE
        WHEN NTILE(100) OVER (ORDER BY ca_annuel DESC) <= 10 THEN 'Top 10%'
        WHEN NTILE(100) OVER (ORDER BY ca_annuel DESC) <= 25 THEN 'Top 25%'
        WHEN NTILE(100) OVER (ORDER BY ca_annuel DESC) <= 50 THEN 'Top 50%'
        ELSE 'Bottom 50%'
    END AS segment
FROM clients;
```

### 4. Médiane et Quartiles

Bien que PostgreSQL ait des fonctions dédiées (`PERCENTILE_CONT`), vous pouvez approximer avec `NTILE()` :

```sql
-- Trouver la valeur médiane (approximative)
SELECT AVG(ventes) AS mediane  
FROM (  
    SELECT ventes,
           NTILE(2) OVER (ORDER BY ventes) AS moitie
    FROM produits
) AS sub
WHERE moitie = 1;  -- Première moitié
```

### 5. Éliminer les Doublons Basés sur un Critère

Garder seulement la ligne avec le rang 1 pour chaque groupe de doublons :

```sql
DELETE FROM commandes  
WHERE id NOT IN (  
    SELECT id
    FROM (
        SELECT
            id,
            ROW_NUMBER() OVER (
                PARTITION BY client_id, produit_id, date_commande
                ORDER BY heure_creation
            ) AS rn
        FROM commandes
    ) AS dedupe
    WHERE rn = 1
);
```

## Ordre de Tri et Déterminisme

### Problème du Tri Instable

Avec `ROW_NUMBER()`, si plusieurs lignes ont la même valeur dans `ORDER BY`, l'ordre entre elles est **non déterministe** :

```sql
-- Ordre potentiellement instable
ROW_NUMBER() OVER (ORDER BY ventes DESC)
```

Si deux vendeurs ont le même nombre de ventes, l'un pourrait être #1 et l'autre #2, mais cet ordre peut **changer** entre les exécutions.

### Solution : Tri Multi-Colonnes

Ajoutez une colonne de tri secondaire **unique** pour garantir un ordre déterministe :

```sql
-- Ordre déterministe
ROW_NUMBER() OVER (ORDER BY ventes DESC, vendeur_id)
```

Maintenant, en cas d'égalité sur `ventes`, le `vendeur_id` départage.

### Importance en Production

C'est **crucial** pour :
- **Pagination** : Éviter que des lignes apparaissent deux fois ou disparaissent entre les pages  
- **Suppression de doublons** : Garantir que c'est toujours la même ligne qui est conservée  
- **Tests** : Résultats reproductibles

## Performances et Index

### Optimisation avec Index

Les fonctions de rang bénéficient grandement d'index sur les colonnes utilisées dans `ORDER BY` :

```sql
-- Si vous faites souvent :
RANK() OVER (PARTITION BY categorie ORDER BY prix DESC)

-- Créez un index :
CREATE INDEX idx_produits_cat_prix ON produits(categorie, prix DESC);
```

### Coût de Calcul

L'ordre de coût (du moins cher au plus cher) :

1. `ROW_NUMBER()` : Simple compteur  
2. `DENSE_RANK()` : Détection des valeurs distinctes  
3. `RANK()` : Comptage de lignes précédentes à égalité  
4. `NTILE()` : Division et distribution

**Bonne pratique** : Toutes ces fonctions sont généralement très performantes. Privilégiez la **clarté** et la **justesse** du résultat plutôt que des micro-optimisations.

## Erreurs Courantes

### 1. Oublier ORDER BY

```sql
-- ❌ ERREUR : Rang sans ordre défini
RANK() OVER (PARTITION BY categorie)

-- ✅ CORRECT
RANK() OVER (PARTITION BY categorie ORDER BY prix DESC)
```

Sans `ORDER BY`, le rang n'a pas de sens.

### 2. Utiliser Window Functions dans WHERE

```sql
-- ❌ ERREUR : Window function dans WHERE
SELECT * FROM produits  
WHERE RANK() OVER (ORDER BY prix DESC) <= 3;  

-- ✅ CORRECT : Utiliser une sous-requête
SELECT * FROM (
    SELECT *, RANK() OVER (ORDER BY prix DESC) AS rang
    FROM produits
) AS sub
WHERE rang <= 3;
```

### 3. Confondre les Fonctions

Choisir la mauvaise fonction selon le besoin :

- Besoin de **numéros uniques** → `ROW_NUMBER()`
- Besoin de **classement sportif** (ex-aequo) → `RANK()` ou `DENSE_RANK()`
- Besoin de **groupes de taille égale** → `NTILE()`

### 4. Oublier PARTITION BY pour des Rangs par Groupe

```sql
-- ❌ Rang global (pas ce qu'on veut souvent)
RANK() OVER (ORDER BY ventes DESC)

-- ✅ Rang par région
RANK() OVER (PARTITION BY region ORDER BY ventes DESC)
```

### 5. Supposer un Ordre avec ROW_NUMBER sans Tri Secondaire

```sql
-- ❌ Ordre instable en cas d'égalité
ROW_NUMBER() OVER (ORDER BY ventes DESC)

-- ✅ Ordre déterministe
ROW_NUMBER() OVER (ORDER BY ventes DESC, id)
```

## Cas d'Usage Pratiques Complets

### 1. Top 3 Produits par Catégorie

```sql
WITH produits_classes AS (
    SELECT
        categorie,
        nom_produit,
        ventes,
        RANK() OVER (
            PARTITION BY categorie
            ORDER BY ventes DESC
        ) AS rang
    FROM produits
)
SELECT categorie, nom_produit, ventes  
FROM produits_classes  
WHERE rang <= 3  
ORDER BY categorie, rang;  
```

### 2. Pagination Performante

```sql
-- Page 5 (lignes 41-50)
SELECT * FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (ORDER BY date_creation DESC, id) AS rn
    FROM articles
) AS paginated
WHERE rn BETWEEN 41 AND 50;
```

### 3. Segmentation Client RFM Simplifiée

```sql
SELECT
    client_id,
    derniere_commande,
    nb_commandes,
    ca_total,
    NTILE(5) OVER (ORDER BY derniere_commande DESC) AS recence_score,
    NTILE(5) OVER (ORDER BY nb_commandes DESC) AS frequence_score,
    NTILE(5) OVER (ORDER BY ca_total DESC) AS montant_score
FROM clients;
```

### 4. Déduplication avec Critère de Priorité

```sql
-- Garder l'email le plus récent de chaque personne
DELETE FROM emails  
WHERE id IN (  
    SELECT id FROM (
        SELECT
            id,
            ROW_NUMBER() OVER (
                PARTITION BY personne_id
                ORDER BY date_envoi DESC
            ) AS rn
        FROM emails
    ) AS numbered
    WHERE rn > 1  -- Supprimer tous sauf le plus récent
);
```

### 5. Analyse des Performances par Quartile

```sql
WITH vendeurs_quartiles AS (
    SELECT
        vendeur_id,
        nom,
        ventes_totales,
        NTILE(4) OVER (ORDER BY ventes_totales DESC) AS quartile
    FROM vendeurs
)
SELECT
    quartile,
    COUNT(*) AS nb_vendeurs,
    AVG(ventes_totales) AS ventes_moyennes,
    MIN(ventes_totales) AS ventes_min,
    MAX(ventes_totales) AS ventes_max
FROM vendeurs_quartiles  
GROUP BY quartile  
ORDER BY quartile;  
```

## Points Clés à Retenir

- ✅ **ROW_NUMBER()** : Numérotation unique pour chaque ligne  
- ✅ **RANK()** : Classement avec sauts en cas d'égalité (1, 1, 3, 4, 4, 6)  
- ✅ **DENSE_RANK()** : Classement sans sauts (1, 1, 2, 3, 3, 4)  
- ✅ **NTILE(n)** : Répartition en n groupes de taille approximativement égale  
- ✅ Toutes nécessitent **ORDER BY** (sauf cas très rares)  
- ✅ Combinez avec **PARTITION BY** pour des rangs par groupe  
- ✅ Utilisez une **sous-requête ou CTE** pour filtrer sur les rangs  
- ✅ Ajoutez un **tri secondaire unique** pour garantir le déterminisme

## Tableau de Choix Rapide

| Besoin | Fonction à Utiliser |
|--------|---------------------|
| Numérotation unique pour pagination | `ROW_NUMBER()` |
| Top N par catégorie | `ROW_NUMBER()` ou `RANK()` |
| Podium sportif avec ex-aequo | `RANK()` |
| Classement par niveaux (A, B, C) | `DENSE_RANK()` |
| Segmentation en quantiles | `NTILE()` |
| Éliminer les doublons | `ROW_NUMBER()` |
| Distribution équitable | `NTILE()` |

## Résumé Visuel

```sql
-- Les quatre fonctions côte à côte
SELECT
    colonne,
    valeur,
    ROW_NUMBER() OVER (ORDER BY valeur DESC) AS row_num,    -- 1,2,3,4,5,6
    RANK()       OVER (ORDER BY valeur DESC) AS rank,       -- 1,1,3,4,4,6
    DENSE_RANK() OVER (ORDER BY valeur DESC) AS dense_rank, -- 1,1,2,3,3,4
    NTILE(3)     OVER (ORDER BY valeur DESC) AS tertile     -- 1,1,2,2,3,3
FROM table;

-- Avec partitionnement par groupe
SELECT
    groupe,
    valeur,
    RANK() OVER (
        PARTITION BY groupe
        ORDER BY valeur DESC
    ) AS rang_dans_groupe
FROM table;
```

---


⏭️ [Fonctions de valeur (LAG, LEAD, FIRST_VALUE, LAST_VALUE)](/10-fonctions-de-fenetrage/05-fonctions-de-valeur.md)
