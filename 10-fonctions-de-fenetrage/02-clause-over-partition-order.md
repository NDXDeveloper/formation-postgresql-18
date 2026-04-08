🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.2. La clause OVER, PARTITION BY et ORDER BY

## Introduction

Maintenant que vous comprenez la philosophie des window functions (agréger sans grouper), il est temps de maîtriser leur syntaxe. Les trois éléments clés sont :

- **OVER** : La clause qui transforme une fonction d'agrégation en window function  
- **PARTITION BY** : Pour définir les groupes de calcul  
- **ORDER BY** : Pour définir l'ordre dans chaque fenêtre

Explorons chacun de ces éléments en détail.

## La Clause OVER : Le Déclencheur

### Syntaxe de Base

La clause `OVER` est ce qui distingue une fonction d'agrégation classique d'une window function :

```sql
-- Agrégation classique (une ligne en sortie)
SELECT SUM(montant) FROM ventes;

-- Window function (toutes les lignes conservées)
SELECT montant, SUM(montant) OVER () FROM ventes;
```

### OVER () Vide : La Fenêtre Globale

Quand vous utilisez `OVER ()` sans paramètres, la fenêtre englobe **toutes les lignes** du résultat :

```sql
SELECT
    vendeur,
    produit,
    montant,
    SUM(montant) OVER () AS total_global
FROM ventes;
```

Résultat :

```
| vendeur  | produit      | montant | total_global |
|----------|--------------|---------|--------------|
| Alice    | Ordinateur   | 1200    | 2850         |
| Alice    | Souris       | 25      | 2850         |
| Bob      | Clavier      | 75      | 2850         |
| Bob      | Écran        | 350     | 2850         |
| Caroline | Ordinateur   | 1200    | 2850         |
```

**Observation** : Le `total_global` est identique sur chaque ligne (2850 = somme de tous les montants).

### Cas d'Usage de OVER ()

Cette fenêtre globale est utile pour calculer des **pourcentages du total** :

```sql
SELECT
    vendeur,
    produit,
    montant,
    SUM(montant) OVER () AS total_global,
    ROUND(montant * 100.0 / SUM(montant) OVER (), 2) AS pourcentage_global
FROM ventes;
```

Résultat :

```
| vendeur  | produit      | montant | total_global | pourcentage_global |
|----------|--------------|---------|--------------|-------------------|
| Alice    | Ordinateur   | 1200    | 2850         | 42.11             |
| Alice    | Souris       | 25      | 2850         | 0.88              |
| Bob      | Clavier      | 75      | 2850         | 2.63              |
| Bob      | Écran        | 350     | 2850         | 12.28             |
| Caroline | Ordinateur   | 1200    | 2850         | 42.11             |
```

Chaque vente est exprimée en pourcentage du chiffre d'affaires total.

## PARTITION BY : Diviser pour Mieux Calculer

### Le Concept de Partition

`PARTITION BY` divise vos données en **groupes logiques** (appelés "partitions") et effectue le calcul **séparément** pour chaque groupe.

**Analogie** : Imaginez que vous triez des cartes par couleur (cœur, carreau, pique, trèfle). Chaque couleur forme une partition. Vous pouvez ensuite compter les cartes dans chaque couleur sans mélanger les groupes.

### Syntaxe

```sql
fonction OVER (PARTITION BY colonne)
```

### Exemple Fondamental

Calculer le total des ventes **par vendeur** :

```sql
SELECT
    vendeur,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur) AS total_vendeur
FROM ventes;
```

Résultat :

```
| vendeur  | produit      | montant | total_vendeur |
|----------|--------------|---------|---------------|
| Alice    | Ordinateur   | 1200    | 1225          |
| Alice    | Souris       | 25      | 1225          |
| Bob      | Clavier      | 75      | 425           |
| Bob      | Écran        | 350     | 425           |
| Caroline | Ordinateur   | 1200    | 1200          |
```

**Explication** :
- Pour les lignes d'Alice, la fenêtre contient uniquement les ventes d'Alice → total = 1225
- Pour les lignes de Bob, la fenêtre contient uniquement les ventes de Bob → total = 425
- Pour Caroline, une seule vente → total = 1200

### PARTITION BY Multiple

Vous pouvez partitionner sur **plusieurs colonnes** :

```sql
SELECT
    vendeur,
    region,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur, region) AS total_vendeur_region
FROM ventes_regionales;
```

Cela crée une partition unique pour chaque **combinaison** vendeur + région.

**Exemple** :
- (Alice, Paris) forme une partition  
- (Alice, Lyon) forme une autre partition  
- (Bob, Paris) forme une troisième partition, etc.

### PARTITION BY vs GROUP BY : Rappel

| Aspect | GROUP BY | PARTITION BY |
|--------|----------|--------------|
| **Élimine les lignes** | Oui (agrège) | Non (conserve tout) |
| **Crée des groupes** | Oui | Oui |
| **Utilisé dans** | SELECT avec agrégation | Clause OVER |
| **Peut être combiné** | Avec HAVING | Avec ORDER BY dans OVER |

**Point clé** : `PARTITION BY` fait la même chose que `GROUP BY` (diviser en groupes), mais **sans fusionner les lignes**.

## ORDER BY dans OVER : L'Ordre Compte

### Pourquoi ORDER BY dans une Window Function ?

Ajouter `ORDER BY` dans la clause `OVER` change fondamentalement le comportement de certaines fonctions. Cela définit :

1. **L'ordre de traitement** des lignes dans chaque partition  
2. **La portée de la fenêtre** (par défaut : du début jusqu'à la ligne courante)

### Syntaxe

```sql
fonction OVER (PARTITION BY colonne ORDER BY autre_colonne)
```

### Exemple sans ORDER BY

```sql
SELECT
    vendeur,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur) AS total_vendeur
FROM ventes;
```

Résultat :

```
| vendeur  | produit      | montant | total_vendeur |
|----------|--------------|---------|---------------|
| Alice    | Ordinateur   | 1200    | 1225          |
| Alice    | Souris       | 25      | 1225          |
| Bob      | Clavier      | 75      | 425           |
| Bob      | Écran        | 350     | 425           |
| Caroline | Ordinateur   | 1200    | 1200          |
```

Le `total_vendeur` est **identique pour toutes les lignes d'Alice** (1225).

### Exemple AVEC ORDER BY : Le Cumul

```sql
SELECT
    vendeur,
    date_vente,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur ORDER BY date_vente) AS cumul_vendeur
FROM ventes  
ORDER BY vendeur, date_vente;  
```

Supposons ces données :

```
| vendeur | date_vente | produit    | montant |
|---------|------------|------------|---------|
| Alice   | 2025-01-01 | Souris     | 25      |
| Alice   | 2025-01-02 | Ordinateur | 1200    |
| Bob     | 2025-01-01 | Clavier    | 75      |
| Bob     | 2025-01-03 | Écran      | 350     |
```

Résultat :

```
| vendeur | date_vente | produit    | montant | cumul_vendeur |
|---------|------------|------------|---------|---------------|
| Alice   | 2025-01-01 | Souris     | 25      | 25            |
| Alice   | 2025-01-02 | Ordinateur | 1200    | 1225          |
| Bob     | 2025-01-01 | Clavier    | 75      | 75            |
| Bob     | 2025-01-03 | Écran      | 350     | 425           |
```

**Observation** : Le `cumul_vendeur` **évolue** ligne par ligne !

- Ligne 1 (Alice, 01/01) : cumul = 25 (juste cette vente)
- Ligne 2 (Alice, 02/01) : cumul = 25 + 1200 = 1225 (cumul des deux ventes d'Alice jusqu'ici)
- Ligne 3 (Bob, 01/01) : cumul = 75 (début de la partition de Bob)
- Ligne 4 (Bob, 03/01) : cumul = 75 + 350 = 425 (cumul des deux ventes de Bob)

### La Fenêtre Glissante (Running Window)

Avec `ORDER BY`, la fenêtre devient **glissante** par défaut :

- Pour chaque ligne, PostgreSQL calcule l'agrégation depuis le **début de la partition** jusqu'à la **ligne courante**
- C'est ce qui crée l'effet de cumul

**Visualisation** :

```
Partition d'Alice (ordonnée par date) :
┌─────────────┬─────────┐
│ Ligne 1     │ Ligne 2 │
│ (Souris)    │ (Ordi)  │
└─────────────┴─────────┘

Ligne 1 : Fenêtre = [Ligne 1]           → SUM = 25  
Ligne 2 : Fenêtre = [Ligne 1, Ligne 2]  → SUM = 25 + 1200 = 1225  
```

### ORDER BY Multiple

Vous pouvez trier sur plusieurs colonnes :

```sql
SUM(montant) OVER (PARTITION BY vendeur ORDER BY date_vente, id)
```

Cela garantit un tri déterministe même si plusieurs lignes ont la même date.

### ORDER BY DESC

Vous pouvez inverser l'ordre :

```sql
SUM(montant) OVER (PARTITION BY vendeur ORDER BY date_vente DESC)
```

Cela calcule le cumul **depuis la date la plus récente**.

## Combinaisons Possibles

### OVER () - Fenêtre Globale

```sql
SUM(montant) OVER ()
```

- **Pas de partition** : Toutes les lignes ensemble  
- **Pas d'ordre** : Le calcul porte sur tout l'ensemble

**Usage** : Total global, pourcentage du total général

### OVER (PARTITION BY ...) - Partitions Sans Ordre

```sql
SUM(montant) OVER (PARTITION BY vendeur)
```

- **Avec partitions** : Groupes séparés  
- **Sans ordre** : Le calcul porte sur toute la partition

**Usage** : Total par groupe, moyenne par catégorie

### OVER (ORDER BY ...) - Ordre Global

```sql
SUM(montant) OVER (ORDER BY date_vente)
```

- **Pas de partition** : Toutes les lignes ensemble  
- **Avec ordre** : Fenêtre glissante depuis le début

**Usage** : Cumul global chronologique

### OVER (PARTITION BY ... ORDER BY ...) - Le Plus Complet

```sql
SUM(montant) OVER (PARTITION BY vendeur ORDER BY date_vente)
```

- **Avec partitions** : Groupes séparés  
- **Avec ordre** : Fenêtre glissante dans chaque groupe

**Usage** : Cumul par groupe, classement par catégorie

## Cas d'Usage Pratiques

### 1. Calculer le Pourcentage du Total par Groupe

```sql
SELECT
    vendeur,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur) AS total_vendeur,
    ROUND(montant * 100.0 / SUM(montant) OVER (PARTITION BY vendeur), 2) AS pct_vendeur
FROM ventes;
```

Chaque vente est exprimée en pourcentage du total de son vendeur.

### 2. Calculer un Cumul Temporel

```sql
SELECT
    date_vente,
    produit,
    montant,
    SUM(montant) OVER (ORDER BY date_vente) AS cumul_chronologique
FROM ventes  
ORDER BY date_vente;  
```

Affiche le chiffre d'affaires cumulé au fil du temps.

### 3. Comparer à la Moyenne de son Groupe

```sql
SELECT
    vendeur,
    produit,
    montant,
    AVG(montant) OVER (PARTITION BY vendeur) AS moyenne_vendeur,
    montant - AVG(montant) OVER (PARTITION BY vendeur) AS ecart_moyenne
FROM ventes;
```

Permet de voir si une vente est au-dessus ou en-dessous de la moyenne du vendeur.

### 4. Calculer le Nombre de Ventes par Vendeur

```sql
SELECT
    vendeur,
    produit,
    montant,
    COUNT(*) OVER (PARTITION BY vendeur) AS nb_ventes_vendeur
FROM ventes;
```

Chaque ligne affiche le nombre total de ventes réalisées par son vendeur.

### 5. Trouver le Montant Maximum par Vendeur

```sql
SELECT
    vendeur,
    produit,
    montant,
    MAX(montant) OVER (PARTITION BY vendeur) AS meilleure_vente_vendeur
FROM ventes;
```

Permet de comparer chaque vente à la meilleure vente du vendeur.

## Différence entre ORDER BY dans OVER et ORDER BY Final

**Attention** : Il y a DEUX types de `ORDER BY` différents !

### ORDER BY dans OVER

```sql
SELECT
    vendeur,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur ORDER BY date_vente) AS cumul
FROM ventes
-- Pas de ORDER BY ici !
```

- Définit **l'ordre de calcul** dans la fenêtre
- Ne change **pas l'ordre d'affichage** des lignes du résultat
- Utilisé par la window function

### ORDER BY Final (clause SELECT)

```sql
SELECT
    vendeur,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur) AS total
FROM ventes  
ORDER BY vendeur, montant DESC;  -- ORDER BY final  
```

- Définit **l'ordre d'affichage** des lignes dans le résultat
- Appliqué **après** le calcul des window functions
- N'affecte **pas** les calculs de fenêtre

### Les Deux Ensemble

Vous pouvez (et devez souvent) utiliser les deux :

```sql
SELECT
    vendeur,
    date_vente,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur ORDER BY date_vente) AS cumul
FROM ventes  
ORDER BY vendeur, date_vente;  -- Pour un affichage lisible  
```

1. `ORDER BY date_vente` dans `OVER` : Définit le cumul chronologique  
2. `ORDER BY vendeur, date_vente` final : Affiche les résultats dans un ordre compréhensible

## Fonctions d'Agrégation Disponibles

Toutes les fonctions d'agrégation classiques fonctionnent dans `OVER` :

| Fonction | Description | Exemple |
|----------|-------------|---------|
| `SUM()` | Somme | `SUM(montant) OVER (...)` |
| `AVG()` | Moyenne | `AVG(montant) OVER (...)` |
| `COUNT()` | Comptage | `COUNT(*) OVER (...)` |
| `MIN()` | Minimum | `MIN(montant) OVER (...)` |
| `MAX()` | Maximum | `MAX(montant) OVER (...)` |
| `STDDEV()` | Écart-type | `STDDEV(montant) OVER (...)` |
| `VARIANCE()` | Variance | `VARIANCE(montant) OVER (...)` |

### Fonctions Spécifiques aux Window Functions

Il existe aussi des fonctions qui **n'existent que** pour les window functions :
- `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` : Classement (chapitre 10.4)  
- `LAG()`, `LEAD()` : Accès aux lignes précédentes/suivantes (chapitre 10.5)  
- `FIRST_VALUE()`, `LAST_VALUE()` : Première/dernière valeur d'une fenêtre (chapitre 10.5)

Nous les verrons en détail dans les sections suivantes.

## Performances et Optimisation

### Ordre d'Évaluation

Les window functions sont évaluées **après** :
1. `FROM` et `JOIN`  
2. `WHERE`  
3. `GROUP BY` et `HAVING`

Et **avant** :
- `ORDER BY` final  
- `LIMIT` et `OFFSET`

### Utilisation des Index

PostgreSQL peut utiliser des index pour optimiser :
- Les `PARTITION BY` (index sur les colonnes de partition)
- Les `ORDER BY` dans `OVER` (index sur les colonnes de tri)

**Bonne pratique** : Si vous partitionnez souvent sur une colonne (ex: `vendeur_id`), créez un index sur cette colonne.

### Calculs Multiples Optimisés

PostgreSQL optimise intelligemment les requêtes avec plusieurs window functions :

```sql
SELECT
    vendeur,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur) AS total,
    AVG(montant) OVER (PARTITION BY vendeur) AS moyenne,
    COUNT(*) OVER (PARTITION BY vendeur) AS nb_ventes
FROM ventes;
```

Les trois fonctions utilisent la **même partition**, PostgreSQL ne parcourt les données qu'**une seule fois**.

## Erreurs Courantes

### 1. Oublier OVER

```sql
-- ❌ ERREUR : Mélange agrégation et colonnes individuelles
SELECT vendeur, montant, SUM(montant)  
FROM ventes;  

-- ✅ CORRECT : Avec OVER
SELECT vendeur, montant, SUM(montant) OVER ()  
FROM ventes;  
```

### 2. Utiliser Window Function dans WHERE

```sql
-- ❌ ERREUR : Window functions après WHERE
SELECT vendeur, montant  
FROM ventes  
WHERE SUM(montant) OVER (PARTITION BY vendeur) > 1000;  

-- ✅ CORRECT : Utiliser une sous-requête ou CTE
SELECT * FROM (
    SELECT vendeur, montant,
           SUM(montant) OVER (PARTITION BY vendeur) AS total
    FROM ventes
) AS sub
WHERE total > 1000;
```

### 3. Confondre les Deux ORDER BY

```sql
-- Ceci calcule un cumul chronologique MAIS affiche dans un ordre aléatoire
SELECT vendeur, date_vente, montant,
       SUM(montant) OVER (ORDER BY date_vente) AS cumul
FROM ventes;

-- Mieux : Ajouter un ORDER BY final pour l'affichage
SELECT vendeur, date_vente, montant,
       SUM(montant) OVER (ORDER BY date_vente) AS cumul
FROM ventes  
ORDER BY date_vente;  
```

## Points Clés à Retenir

- ✅ **OVER ()** transforme une fonction d'agrégation en window function  
- ✅ **PARTITION BY** divise les données en groupes sans éliminer les lignes  
- ✅ **ORDER BY dans OVER** crée une fenêtre glissante (par défaut : du début à la ligne courante)  
- ✅ On peut combiner PARTITION BY et ORDER BY : `OVER (PARTITION BY x ORDER BY y)`  
- ✅ ORDER BY dans OVER ≠ ORDER BY final de la requête  
- ✅ Toutes les fonctions d'agrégation (SUM, AVG, COUNT...) fonctionnent dans OVER  
- ✅ PostgreSQL optimise les calculs multiples sur la même partition

## Résumé Visuel

```sql
SELECT
    colonne1,
    colonne2,
    FONCTION_AGREGATION(colonne3) OVER (
        PARTITION BY colonne_groupe    -- Divise en groupes (optionnel)
        ORDER BY colonne_ordre         -- Définit l'ordre (optionnel)
    ) AS resultat
FROM table  
ORDER BY colonne_affichage;            -- Ordre d'affichage (optionnel)  
```

**Composants** :
- `FONCTION_AGREGATION` : SUM, AVG, COUNT, MIN, MAX, etc.  
- `PARTITION BY` : Crée des groupes séparés  
- `ORDER BY` (dans OVER) : Définit l'ordre de traitement, crée une fenêtre glissante  
- `ORDER BY` (final) : Définit l'ordre d'affichage des résultats

---


⏭️ [Frames de fenêtre : ROWS vs RANGE vs GROUPS](/10-fonctions-de-fenetrage/03-frames-de-fenetre.md)
