🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.6. Fonctions d'agrégation en fenêtrage

## Introduction

Nous avons déjà utilisé des fonctions d'agrégation comme `SUM()`, `AVG()` et `COUNT()` dans les sections précédentes. Maintenant, approfondissons leur utilisation dans le contexte des window functions et explorons **toutes** les possibilités qu'elles offrent.

Les **fonctions d'agrégation en fenêtrage** combinent le meilleur des deux mondes :
- La puissance des agrégations (sommes, moyennes, comptages)
- La flexibilité des window functions (conservation des lignes, fenêtres glissantes)

Cette section couvre :
- Les fonctions d'agrégation standards (SUM, AVG, COUNT, MIN, MAX)
- Les fonctions statistiques (STDDEV, VARIANCE, CORR)
- Les agrégations avec frames personnalisés
- Les techniques avancées de calcul

## Les Fonctions d'Agrégation Standards

### Rappel : Agrégation Classique vs Window Function

**Agrégation classique (GROUP BY)** :
```sql
SELECT vendeur, SUM(montant) AS total  
FROM ventes  
GROUP BY vendeur;  
```
→ Une ligne par vendeur, perte du détail

**Agrégation en fenêtrage** :
```sql
SELECT vendeur, produit, montant,
       SUM(montant) OVER (PARTITION BY vendeur) AS total_vendeur
FROM ventes;
```
→ Toutes les lignes conservées, total affiché sur chaque ligne

### SUM() : Sommes et Cumuls

#### Somme Totale

```sql
SELECT
    produit,
    ventes,
    SUM(ventes) OVER () AS total_global
FROM produits;
```

**Données** :
```
| produit    | ventes |
|------------|--------|
| Laptop     | 5000   |
| Souris     | 1000   |
| Clavier    | 800    |
| Écran      | 3000   |
```

**Résultat** :
```
| produit    | ventes | total_global |
|------------|--------|--------------|
| Laptop     | 5000   | 9800         |
| Souris     | 1000   | 9800         |
| Clavier    | 800    | 9800         |
| Écran      | 3000   | 9800         |
```

#### Somme par Groupe

```sql
SELECT
    categorie,
    produit,
    prix,
    SUM(prix) OVER (PARTITION BY categorie) AS total_categorie
FROM produits;
```

**Données** :
```
| categorie | produit      | prix |
|-----------|--------------|------|
| Hardware  | Laptop       | 1000 |
| Hardware  | Desktop      | 800  |
| Hardware  | Tablet       | 500  |
| Software  | Office       | 150  |
| Software  | Antivirus    | 50   |
```

**Résultat** :
```
| categorie | produit      | prix | total_categorie |
|-----------|--------------|------|-----------------|
| Hardware  | Laptop       | 1000 | 2300            |
| Hardware  | Desktop      | 800  | 2300            |
| Hardware  | Tablet       | 500  | 2300            |
| Software  | Office       | 150  | 200             |
| Software  | Antivirus    | 50   | 200             |
```

#### Cumul (Running Total)

Le cas d'usage le plus populaire de `SUM()` avec window functions :

```sql
SELECT
    date,
    ventes,
    SUM(ventes) OVER (ORDER BY date) AS cumul
FROM ventes_quotidiennes  
ORDER BY date;  
```

**Données** :
```
| date       | ventes |
|------------|--------|
| 2025-01-01 | 100    |
| 2025-01-02 | 150    |
| 2025-01-03 | 120    |
| 2025-01-04 | 200    |
```

**Résultat** :
```
| date       | ventes | cumul |
|------------|--------|-------|
| 2025-01-01 | 100    | 100   | ← 100
| 2025-01-02 | 150    | 250   | ← 100+150
| 2025-01-03 | 120    | 370   | ← 100+150+120
| 2025-01-04 | 200    | 570   | ← 100+150+120+200
```

#### Cumul par Groupe

```sql
SELECT
    vendeur,
    date,
    ventes,
    SUM(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY date
    ) AS cumul_vendeur
FROM ventes  
ORDER BY vendeur, date;  
```

#### Total Glissant (Rolling Sum)

Somme des N derniers jours :

```sql
SELECT
    date,
    ventes,
    SUM(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS total_7j
FROM ventes_quotidiennes;
```

**Résultat** : Chaque ligne affiche la somme des 7 derniers jours (ou moins si on est au début).

### AVG() : Moyennes

#### Moyenne Globale

```sql
SELECT
    produit,
    prix,
    AVG(prix) OVER () AS prix_moyen_global,
    prix - AVG(prix) OVER () AS ecart_moyenne
FROM produits;
```

**Données** :
```
| produit    | prix |
|------------|------|
| Laptop     | 1000 |
| Souris     | 30   |
| Clavier    | 50   |
| Écran      | 400  |
```

**Résultat** :
```
| produit    | prix | prix_moyen_global | ecart_moyenne |
|------------|------|-------------------|---------------|
| Laptop     | 1000 | 370               | 630           |
| Souris     | 30   | 370               | -340          |
| Clavier    | 50   | 370               | -320          |
| Écran      | 400  | 370               | 30            |
```

#### Moyenne par Catégorie

```sql
SELECT
    categorie,
    produit,
    prix,
    AVG(prix) OVER (PARTITION BY categorie) AS prix_moyen_categorie,
    CASE
        WHEN prix > AVG(prix) OVER (PARTITION BY categorie)
        THEN 'Au-dessus de la moyenne'
        ELSE 'En-dessous de la moyenne'
    END AS position
FROM produits;
```

#### Moyenne Mobile (Moving Average)

L'une des analyses les plus utiles en finance et data science :

```sql
SELECT
    date,
    cours,
    AVG(cours) OVER (
        ORDER BY date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ) AS moyenne_mobile_20j,
    AVG(cours) OVER (
        ORDER BY date
        ROWS BETWEEN 49 PRECEDING AND CURRENT ROW
    ) AS moyenne_mobile_50j
FROM cours_bourse  
ORDER BY date;  
```

**Usage** : Détection de tendances (quand la MM20 croise la MM50, c'est un signal).

#### Moyenne sur Fenêtre Temporelle (par jours calendaires)

```sql
SELECT
    date,
    temperature,
    AVG(temperature) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
    ) AS temp_moyenne_7j
FROM meteo;
```

Utilise `RANGE` pour une moyenne sur les **7 jours calendaires** (le jour courant + les 6 jours précédents). Si certains jours ont plusieurs mesures et d'autres aucune, on agrège correctement par plage de dates.

⚠️ **Attention au comptage** : `INTERVAL '6 days' PRECEDING AND CURRENT ROW` couvre **7 jours calendaires** (du jour `D-6` au jour `D` inclus). De même, `INTERVAL '7 days' PRECEDING` couvrirait **8 jours**. Toujours soustraire 1 pour obtenir le nombre de jours voulu.

ℹ️ **Note** : il s'agit d'une moyenne **arithmétique** sur les valeurs de la fenêtre, **pas** d'une moyenne pondérée (où les valeurs récentes auraient un poids supérieur). Pour une vraie moyenne pondérée, voir la section WMA au chapitre 10.7.

### COUNT() : Comptages

#### Nombre Total

```sql
SELECT
    produit,
    COUNT(*) OVER () AS nb_produits_total
FROM produits;
```

Chaque ligne affiche le nombre total de produits.

#### Nombre par Groupe

```sql
SELECT
    categorie,
    produit,
    COUNT(*) OVER (PARTITION BY categorie) AS nb_produits_categorie
FROM produits;
```

**Données** :
```
| categorie | produit   |
|-----------|-----------|
| Laptop    | MacBook   |
| Laptop    | ThinkPad  |
| Laptop    | Dell XPS  |
| Souris    | MX Master |
| Souris    | Magic     |
```

**Résultat** :
```
| categorie | produit   | nb_produits_categorie |
|-----------|-----------|----------------------|
| Laptop    | MacBook   | 3                    |
| Laptop    | ThinkPad  | 3                    |
| Laptop    | Dell XPS  | 3                    |
| Souris    | MX Master | 2                    |
| Souris    | Magic     | 2                    |
```

#### Comptage Cumulé

```sql
SELECT
    date,
    nouveau_client,
    COUNT(*) OVER (ORDER BY date) AS nb_clients_cumule
FROM inscriptions  
ORDER BY date;  
```

Affiche le nombre total de clients acquis depuis le début.

#### Comptage avec DISTINCT

⚠️ **Attention** : `COUNT(DISTINCT)` ne fonctionne **pas** directement dans une window function en PostgreSQL.

```sql
-- ❌ NE FONCTIONNE PAS
COUNT(DISTINCT categorie) OVER (PARTITION BY vendeur)

-- ✅ WORKAROUND : Sous-requête
SELECT
    vendeur,
    date,
    ventes,
    (SELECT COUNT(DISTINCT categorie)
     FROM ventes v2
     WHERE v2.vendeur = v1.vendeur) AS nb_categories
FROM ventes v1;
```

#### Comptage Glissant

```sql
SELECT
    date,
    COUNT(*) OVER (
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS nb_jours_actifs_30j
FROM ventes_quotidiennes;
```

#### Comptage Conditionnel avec FILTER

PostgreSQL supporte la clause **`FILTER (WHERE ...)`** sur les agrégations en window function (mais **pas** sur les fonctions de rang comme ROW_NUMBER ou LAG). C'est une alternative **plus lisible et plus rapide** que `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` :

```sql
-- ✅ Avec FILTER (élégant et performant)
SELECT
    vendeur,
    date,
    montant,
    COUNT(*) FILTER (WHERE statut = 'paye')   OVER (PARTITION BY vendeur) AS nb_payees,
    COUNT(*) FILTER (WHERE statut = 'annule') OVER (PARTITION BY vendeur) AS nb_annulees,
    SUM(montant) FILTER (WHERE statut = 'paye') OVER (PARTITION BY vendeur) AS ca_paye
FROM ventes;
```

**Équivalent classique** (plus verbeux) :

```sql
SELECT
    vendeur,
    date,
    montant,
    SUM(CASE WHEN statut = 'paye'   THEN 1 ELSE 0 END) OVER (PARTITION BY vendeur) AS nb_payees,
    SUM(CASE WHEN statut = 'annule' THEN 1 ELSE 0 END) OVER (PARTITION BY vendeur) AS nb_annulees,
    SUM(CASE WHEN statut = 'paye'   THEN montant ELSE 0 END) OVER (PARTITION BY vendeur) AS ca_paye
FROM ventes;
```

⚠️ **Restriction** : la documentation PG 18 précise *"Only window functions that are aggregates accept a FILTER clause"*. Donc :

- ✅ `SUM/AVG/COUNT/MIN/MAX/STDDEV ... FILTER (...) OVER (...)` : valide
- ❌ `ROW_NUMBER() FILTER (...) OVER (...)` : interdit
- ❌ `LAG(x) FILTER (...) OVER (...)` : interdit

#### FILTER avec Frame Glissant

`FILTER` se combine librement avec un frame personnalisé :

```sql
-- Nombre de jours avec ventes > 1000 dans les 7 derniers jours
SELECT
    date,
    ventes,
    COUNT(*) FILTER (WHERE ventes > 1000)
        OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS nb_bons_jours_7j,
    AVG(ventes) FILTER (WHERE ventes > 0)
        OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moyenne_jours_actifs_7j
FROM ventes_quotidiennes;
```

ℹ️ **Note** : la condition `WHERE` du `FILTER` ne peut **pas** contenir une autre window function (même règle que pour WHERE classique). Pour comparer à une valeur d'une autre ligne, calculez-la d'abord dans une CTE puis appliquez `FILTER` dessus.

### MIN() et MAX() : Extremums

#### Minimum et Maximum Globaux

```sql
SELECT
    produit,
    prix,
    MIN(prix) OVER () AS prix_min_global,
    MAX(prix) OVER () AS prix_max_global,
    ROUND((prix - MIN(prix) OVER ()) * 100.0
          / (MAX(prix) OVER () - MIN(prix) OVER ()), 2) AS position_pct
FROM produits;
```

**Résultat** : Chaque produit voit où il se situe entre le moins cher et le plus cher (0% à 100%).

#### Par Catégorie

```sql
SELECT
    categorie,
    produit,
    prix,
    MIN(prix) OVER (PARTITION BY categorie) AS moins_cher_categorie,
    MAX(prix) OVER (PARTITION BY categorie) AS plus_cher_categorie,
    CASE
        WHEN prix = MAX(prix) OVER (PARTITION BY categorie)
        THEN 'Le plus cher'
        WHEN prix = MIN(prix) OVER (PARTITION BY categorie)
        THEN 'Le moins cher'
        ELSE 'Intermédiaire'
    END AS positionnement
FROM produits;
```

#### Maximum Glissant (All-Time High)

Trouver le record historique jusqu'à chaque point :

```sql
SELECT
    date,
    cours,
    MAX(cours) OVER (ORDER BY date) AS record_historique,
    cours - MAX(cours) OVER (ORDER BY date) AS drawdown
FROM cours_bourse  
ORDER BY date;  
```

**Données** :
```
| date       | cours |
|------------|-------|
| 2025-01-01 | 100   |
| 2025-01-02 | 105   |
| 2025-01-03 | 102   |
| 2025-01-04 | 110   |
| 2025-01-05 | 108   |
```

**Résultat** :
```
| date       | cours | record_historique | drawdown |
|------------|-------|-------------------|----------|
| 2025-01-01 | 100   | 100               | 0        |
| 2025-01-02 | 105   | 105               | 0        |
| 2025-01-03 | 102   | 105               | -3       | ← Sous le record
| 2025-01-04 | 110   | 110               | 0        | ← Nouveau record
| 2025-01-05 | 108   | 110               | -2       |
```

Le `drawdown` montre l'écart par rapport au plus haut historique.

#### Minimum Glissant

```sql
SELECT
    date,
    stock,
    MIN(stock) OVER (
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS stock_min_30j
FROM inventaire;
```

Utile pour détecter les niveaux critiques de stock.

## Fonctions Statistiques

### STDDEV() : Écart-Type

Mesure la dispersion des valeurs autour de la moyenne.

```sql
SELECT
    vendeur,
    ventes,
    AVG(ventes) OVER (PARTITION BY vendeur) AS moyenne,
    STDDEV(ventes) OVER (PARTITION BY vendeur) AS ecart_type,
    CASE
        WHEN ventes > AVG(ventes) OVER (PARTITION BY vendeur) +
                      STDDEV(ventes) OVER (PARTITION BY vendeur)
        THEN 'Outlier haut'
        WHEN ventes < AVG(ventes) OVER (PARTITION BY vendeur) -
                      STDDEV(ventes) OVER (PARTITION BY vendeur)
        THEN 'Outlier bas'
        ELSE 'Normal'
    END AS classification
FROM ventes;
```

**Usage** : Détection d'anomalies (valeurs à plus d'1 écart-type sont inhabituelles).

### VARIANCE() : Variance

La variance est l'écart-type au carré :

```sql
SELECT
    categorie,
    produit,
    prix,
    VARIANCE(prix) OVER (PARTITION BY categorie) AS variance_prix
FROM produits;
```

### Écart-Type Glissant

```sql
SELECT
    date,
    cours,
    STDDEV(cours) OVER (
        ORDER BY date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ) AS volatilite_20j
FROM cours_bourse;
```

**Usage** : Mesure de la volatilité en finance (plus l'écart-type est élevé, plus les prix fluctuent).

### Fonctions de Corrélation (Avancé)

PostgreSQL propose des fonctions de corrélation statistique :

#### CORR() : Coefficient de Corrélation

Mesure la relation linéaire entre deux variables (-1 à +1).

**Esprit window function** (chaque produit voit la corrélation de sa catégorie) :

```sql
SELECT
    categorie,
    produit,
    prix,
    ventes,
    CORR(prix, ventes) OVER (PARTITION BY categorie) AS correlation_categorie
FROM produits;
```

**Esprit agrégation classique** (une ligne par catégorie) — souvent plus lisible si on n'a pas besoin du détail :

```sql
SELECT categorie, CORR(prix, ventes) AS correlation
FROM produits
GROUP BY categorie;
```

- **Corrélation positive** (proche de +1) : quand le prix augmente, les ventes augmentent
- **Corrélation négative** (proche de -1) : quand le prix augmente, les ventes diminuent
- **Pas de corrélation** (proche de 0) : aucun lien linéaire

#### COVAR_POP() et COVAR_SAMP() : Covariance

```sql
SELECT
    mois,
    temperature,
    ventes_glaces,
    COVAR_POP(temperature, ventes_glaces) OVER (PARTITION BY mois) AS covariance_mois
FROM donnees_meteo_ventes;
```

- `COVAR_POP` : covariance de la **population** (divise par N)
- `COVAR_SAMP` : covariance de l'**échantillon** (divise par N-1, à utiliser sur des sous-ensembles statistiques)

### PERCENTILE_CONT() et PERCENTILE_DISC() : Percentiles

Trouver la valeur au Nième percentile.

⚠️ **Limitation importante de PostgreSQL** : `PERCENTILE_CONT` et `PERCENTILE_DISC` sont des **ordered-set aggregates** (avec clause `WITHIN GROUP`). La documentation officielle précise :

> "Ordered-set and hypothetical-set aggregates cannot presently be used as window functions."

Donc cette syntaxe est **rejetée** par PostgreSQL :

```sql
-- ❌ INTERDIT : ordered-set aggregate + OVER
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY prix) OVER (PARTITION BY categorie)
```

**✅ Solutions correctes** :

**Approche 1 : GROUP BY classique** (calcul par catégorie, une ligne par catégorie) :

```sql
SELECT
    categorie,
    PERCENTILE_CONT(0.5)  WITHIN GROUP (ORDER BY prix) AS mediane,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY prix) AS q1,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY prix) AS q3
FROM produits
GROUP BY categorie;
```

**Approche 2 : JOIN avec un CTE agrégé** (si on veut conserver le détail par produit) :

```sql
WITH stats_categorie AS (
    SELECT
        categorie,
        PERCENTILE_CONT(0.5)  WITHIN GROUP (ORDER BY prix) AS mediane,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY prix) AS q1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY prix) AS q3
    FROM produits
    GROUP BY categorie
)
SELECT
    p.produit,
    p.prix,
    s.mediane,
    s.q1,
    s.q3,
    CASE
        WHEN p.prix < s.q1 THEN 'Bas (<= Q1)'
        WHEN p.prix < s.mediane THEN 'Moyen-bas'
        WHEN p.prix < s.q3 THEN 'Moyen-haut'
        ELSE 'Haut (>= Q3)'
    END AS quartile_qualitatif
FROM produits p
JOIN stats_categorie s USING (categorie);
```

**Différence entre les deux fonctions** :
- `PERCENTILE_CONT` : interpolation linéaire (valeur **continue**, peut ne pas exister dans la table)  
- `PERCENTILE_DISC` : valeur **discrète** existante (renvoie une vraie ligne)

## Agrégations avec Frames Personnalisés

### Somme sur Fenêtre Temporelle

```sql
SELECT
    date,
    ventes,
    SUM(ventes) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW
    ) AS ventes_30j
FROM ventes_quotidiennes;
```

Somme des ventes sur **30 jours calendaires** : le jour courant + les 29 jours précédents. Pour avoir exactement N jours, utiliser `INTERVAL 'N-1 days' PRECEDING`.

### Moyenne Centrée

```sql
SELECT
    date,
    temperature,
    AVG(temperature) OVER (
        ORDER BY date
        ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING
    ) AS temp_lissee
FROM meteo;
```

Moyenne de 7 jours : 3 avant + jour courant + 3 après (lisse les variations).

### Maximum dans une Fenêtre Glissante

```sql
-- Hypothèse : 1 ligne par semaine dans cours_bourse
SELECT
    date,
    cours,
    MAX(cours) OVER (
        ORDER BY date
        ROWS BETWEEN 52 PRECEDING AND 1 PRECEDING
    ) AS plus_haut_52_semaines
FROM cours_bourse;
```

Compare le cours actuel au maximum des 52 semaines **précédentes** (exclut la semaine courante grâce à `1 PRECEDING` comme borne supérieure).

💡 **Comptage du frame** : `BETWEEN N PRECEDING AND M PRECEDING` couvre `N - M + 1` lignes (ici `52 - 1 + 1 = 52`). Si vous voulez les 52 dernières semaines **y compris** la courante, utilisez `BETWEEN 51 PRECEDING AND CURRENT ROW`.

### Comptage Non-NULL

```sql
SELECT
    date,
    temperature,
    COUNT(temperature) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS nb_mesures_disponibles_7j
FROM meteo;
```

`COUNT(colonne)` compte uniquement les valeurs non-NULL.

## Techniques Avancées

### 1. Multiples Agrégations Simultanées

```sql
SELECT
    vendeur,
    date,
    ventes,
    SUM(ventes) OVER w AS total,
    AVG(ventes) OVER w AS moyenne,
    COUNT(*) OVER w AS nb_ventes,
    MIN(ventes) OVER w AS min,
    MAX(ventes) OVER w AS max,
    STDDEV(ventes) OVER w AS ecart_type
FROM ventes  
WINDOW w AS (PARTITION BY vendeur)  
ORDER BY vendeur, date;  
```

**Astuce** : La clause `WINDOW` évite la duplication de code.

### 2. Calcul de Pourcentages

#### Pourcentage du Total

```sql
SELECT
    categorie,
    produit,
    ventes,
    SUM(ventes) OVER () AS total_global,
    ROUND(ventes * 100.0 / SUM(ventes) OVER (), 2) AS pct_global,
    ROUND(ventes * 100.0 / SUM(ventes) OVER (PARTITION BY categorie), 2) AS pct_categorie
FROM produits;
```

#### Contribution Cumulée

```sql
SELECT
    produit,
    ventes,
    SUM(ventes) OVER (ORDER BY ventes DESC) AS cumul,
    ROUND(
        SUM(ventes) OVER (ORDER BY ventes DESC) * 100.0 /
        SUM(ventes) OVER (),
        2
    ) AS pct_cumule
FROM produits  
ORDER BY ventes DESC;  
```

**Usage** : Analyse de Pareto (règle des 80/20).

### 3. Comparaisons Temporelles

```sql
SELECT
    date,
    ventes,
    AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moyenne_7j,
    ventes - AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS ecart_moyenne_7j,
    CASE
        WHEN ventes > 1.2 * AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) THEN 'Pic'
        WHEN ventes < 0.8 * AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) THEN 'Creux'
        ELSE 'Normal'
    END AS analyse
FROM ventes_quotidiennes;
```

### 4. Z-Score (Standardisation)

Mesure combien d'écarts-types une valeur est éloignée de la moyenne :

```sql
SELECT
    produit,
    prix,
    AVG(prix) OVER () AS moyenne,
    STDDEV(prix) OVER () AS ecart_type,
    ROUND(
        (prix - AVG(prix) OVER ()) /
        NULLIF(STDDEV(prix) OVER (), 0),
        2
    ) AS z_score
FROM produits;
```

**Interprétation** :
- Z-score = 0 : À la moyenne
- Z-score > 2 ou < -2 : Valeur extrême (rare)

### 5. Index de Performance

Normaliser une performance par rapport au début :

```sql
SELECT
    date,
    cours,
    FIRST_VALUE(cours) OVER (ORDER BY date) AS cours_initial,
    ROUND(
        cours * 100.0 / FIRST_VALUE(cours) OVER (ORDER BY date),
        2
    ) AS indice_performance
FROM cours_bourse  
ORDER BY date;  
```

Si `indice_performance = 110`, le cours a progressé de 10% depuis le début.

### 6. Bandes de Bollinger (Finance)

Indicateur technique utilisant moyenne mobile et écart-type :

```sql
SELECT
    date,
    cours,
    AVG(cours) OVER w AS mm20,
    AVG(cours) OVER w + 2 * STDDEV_POP(cours) OVER w AS bande_haute,
    AVG(cours) OVER w - 2 * STDDEV_POP(cours) OVER w AS bande_basse,
    CASE
        WHEN cours > AVG(cours) OVER w + 2 * STDDEV_POP(cours) OVER w
        THEN 'Surachat'
        WHEN cours < AVG(cours) OVER w - 2 * STDDEV_POP(cours) OVER w
        THEN 'Survente'
        ELSE 'Neutre'
    END AS signal
FROM cours_bourse  
WINDOW w AS (ORDER BY date ROWS BETWEEN 19 PRECEDING AND CURRENT ROW)  
ORDER BY date;  
```

> ℹ️ On utilise **`STDDEV_POP`** (écart-type de **population**, diviseur `N`) plutôt que `STDDEV`/`STDDEV_SAMP` (diviseur `N-1`) : c'est la convention des Bandes de Bollinger telles que définies par John Bollinger. L'écart reste faible sur 20 périodes (facteur √(20/19) ≈ 1,026), mais autant rester fidèle à la définition.

### 7. Taux de Croissance Composé (CAGR Annualisé)

⚠️ **Deux pièges à éviter** :

1. **Frame par défaut** : avec `ORDER BY date`, `COUNT(*) OVER (ORDER BY date)` donne un **cumul** (1, 2, 3...), pas le total. Pour le nombre total constant, il faut **omettre `ORDER BY`** ou utiliser `UNBOUNDED FOLLOWING`.

2. **CAGR ≠ taux mensuel** : le CAGR (*Compound Annual Growth Rate*) est par définition **annuel**. Sur des données mensuelles, il faut **annualiser** en élevant le facteur de croissance à la puissance `12 / (nb_mois - 1)`. Sinon on obtient un taux mensuel composé, pas un CAGR.

```sql
WITH donnees AS (
    SELECT
        date,
        ca_mensuel,
        FIRST_VALUE(ca_mensuel) OVER (ORDER BY date) AS ca_initial,
        LAST_VALUE(ca_mensuel) OVER (
            ORDER BY date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS ca_final,
        COUNT(*) OVER () AS nb_mois  -- Total sans ORDER BY (constant sur toutes les lignes)
    FROM ventes_mensuelles
)
SELECT
    date,
    ca_mensuel,
    -- CAGR annualisé : on annualise en élevant à la puissance 12/(nb_mois-1)
    ROUND(
        (POWER(
            ca_final::numeric / NULLIF(ca_initial, 0),
            12.0 / NULLIF(nb_mois - 1, 0)  -- Annualisation
        ) - 1) * 100,
        4
    ) AS cagr_annuel_pct
FROM donnees;
```

ℹ️ **Pour un taux mensuel composé** (croissance moyenne par mois) à la place, utiliser `1.0 / NULLIF(nb_mois - 1, 0)` (sans annualisation). Ces deux valeurs sont liées : `(1 + taux_mensuel)^12 - 1 = CAGR_annuel`.

## Cas d'Usage Pratiques Complets

### 1. Tableau de Bord Exécutif

Cet exemple utilise `PARTITION BY EXTRACT(YEAR FROM mois)` pour les calculs "annuels" : le cumul et le pourcentage redémarrent à chaque année — robuste même sans le `WHERE` final.

```sql
SELECT
    mois,
    ca_mensuel,
    -- Comparaisons temporelles
    LAG(ca_mensuel) OVER (ORDER BY mois) AS ca_mois_precedent,
    ca_mensuel - LAG(ca_mensuel) OVER (ORDER BY mois) AS var_mois,
    -- Cumul depuis le début de l'année (redémarre chaque 1er janvier)
    SUM(ca_mensuel) OVER (
        PARTITION BY EXTRACT(YEAR FROM mois)
        ORDER BY mois
    ) AS ca_cumule_annuel,
    -- Moyenne mobile 3 mois
    AVG(ca_mensuel) OVER (
        ORDER BY mois
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS mm3,
    -- Statistiques sur l'année
    AVG(ca_mensuel)    OVER (PARTITION BY EXTRACT(YEAR FROM mois)) AS moyenne_annuelle,
    STDDEV(ca_mensuel) OVER (PARTITION BY EXTRACT(YEAR FROM mois)) AS volatilite_annuelle,
    -- Record du CA jusqu'au mois courant (depuis le début, toutes années confondues)
    MAX(ca_mensuel) OVER (ORDER BY mois) AS record_ca_jusqua_ce_mois,
    -- Part de chaque mois dans le CA annuel
    ROUND(
        ca_mensuel * 100.0 / NULLIF(
            SUM(ca_mensuel) OVER (PARTITION BY EXTRACT(YEAR FROM mois)),
            0
        ),
        2
    ) AS pct_annuel
FROM ventes_mensuelles  
ORDER BY mois;  
```

### 2. Analyse de Cohortes

```sql
WITH cohortes AS (
    SELECT
        client_id,
        DATE_TRUNC('month', MIN(date_inscription)) AS mois_cohorte,
        date_achat
    FROM clients c
    JOIN achats a ON c.id = a.client_id
    GROUP BY client_id, date_achat
)
SELECT
    mois_cohorte,
    DATE_TRUNC('month', date_achat) AS mois_achat,
    COUNT(DISTINCT client_id) AS nb_clients_actifs,
    FIRST_VALUE(COUNT(DISTINCT client_id)) OVER (
        PARTITION BY mois_cohorte
        ORDER BY DATE_TRUNC('month', date_achat)
    ) AS taille_cohorte_initiale,
    ROUND(
        COUNT(DISTINCT client_id) * 100.0 /
        FIRST_VALUE(COUNT(DISTINCT client_id)) OVER (
            PARTITION BY mois_cohorte
            ORDER BY DATE_TRUNC('month', date_achat)
        ),
        2
    ) AS taux_retention_pct
FROM cohortes  
GROUP BY mois_cohorte, DATE_TRUNC('month', date_achat)  
ORDER BY mois_cohorte, DATE_TRUNC('month', date_achat);  
```

### 3. Détection d'Anomalies Statistiques

```sql
WITH stats AS (
    SELECT
        date,
        nb_visiteurs,
        AVG(nb_visiteurs) OVER w AS moyenne,
        STDDEV(nb_visiteurs) OVER w AS ecart_type
    FROM analytics
    WINDOW w AS (
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND 1 PRECEDING
    )
)
SELECT
    date,
    nb_visiteurs,
    moyenne,
    ecart_type,
    (nb_visiteurs - moyenne) / NULLIF(ecart_type, 0) AS z_score,
    CASE
        WHEN ABS((nb_visiteurs - moyenne) / NULLIF(ecart_type, 0)) > 3
        THEN 'ANOMALIE CRITIQUE'
        WHEN ABS((nb_visiteurs - moyenne) / NULLIF(ecart_type, 0)) > 2
        THEN 'ALERTE'
        ELSE 'NORMAL'
    END AS statut
FROM stats  
WHERE date >= CURRENT_DATE - INTERVAL '90 days'  
ORDER BY date;  
```

### 4. Analyse ABC avec Cumuls

```sql
WITH produits_tries AS (
    SELECT
        produit,
        ca_annuel,
        SUM(ca_annuel) OVER (ORDER BY ca_annuel DESC) AS ca_cumule,
        SUM(ca_annuel) OVER () AS ca_total
    FROM produits
)
SELECT
    produit,
    ca_annuel,
    ca_cumule,
    ROUND(ca_cumule * 100.0 / ca_total, 2) AS pct_cumule,
    CASE
        WHEN ca_cumule * 100.0 / ca_total <= 80 THEN 'A'
        WHEN ca_cumule * 100.0 / ca_total <= 95 THEN 'B'
        ELSE 'C'
    END AS classe_abc
FROM produits_tries  
ORDER BY ca_annuel DESC;  
```

### 5. Momentum et Tendances

⚠️ **Règle importante** : PostgreSQL **interdit l'imbrication** d'une window function dans une autre window function (`ERROR: window function calls cannot be nested`). Pour calculer une moyenne mobile sur la variation `cours - LAG(cours)`, il faut passer par une CTE qui calcule d'abord la variation, puis applique l'agrégation fenêtrée dessus :

```sql
WITH variations AS (
    SELECT
        date,
        cours,
        cours - LAG(cours) OVER (ORDER BY date) AS variation_j
    FROM cours_bourse
)
SELECT
    date,
    cours,
    -- Moyennes mobiles
    AVG(cours) OVER (ORDER BY date ROWS BETWEEN 9 PRECEDING AND CURRENT ROW) AS mm10,
    AVG(cours) OVER (ORDER BY date ROWS BETWEEN 49 PRECEDING AND CURRENT ROW) AS mm50,
    -- Momentum
    cours - LAG(cours, 10) OVER (ORDER BY date) AS momentum_10j,
    -- RSI simplifié (gains/pertes moyens sur 14 jours)
    AVG(GREATEST(variation_j, 0)) OVER (
        ORDER BY date ROWS BETWEEN 13 PRECEDING AND CURRENT ROW
    ) AS gains_moyens_14j,
    AVG(GREATEST(-variation_j, 0)) OVER (
        ORDER BY date ROWS BETWEEN 13 PRECEDING AND CURRENT ROW
    ) AS pertes_moyennes_14j,
    -- Signal
    CASE
        WHEN AVG(cours) OVER (ORDER BY date ROWS BETWEEN 9 PRECEDING AND CURRENT ROW) >
             AVG(cours) OVER (ORDER BY date ROWS BETWEEN 49 PRECEDING AND CURRENT ROW)
        THEN 'Tendance haussière'
        ELSE 'Tendance baissière'
    END AS signal_tendance
FROM variations  
ORDER BY date;  
```

## Optimisation et Performance

### Utiliser des CTE pour la Lisibilité

Au lieu de répéter les mêmes window functions :

```sql
-- ❌ Code répétitif
SELECT
    ventes,
    ventes - AVG(ventes) OVER (PARTITION BY vendeur) AS ecart1,
    (ventes - AVG(ventes) OVER (PARTITION BY vendeur)) /
        STDDEV(ventes) OVER (PARTITION BY vendeur) AS z_score
FROM ventes;

-- ✅ Plus lisible avec CTE
WITH stats AS (
    SELECT
        vendeur,
        ventes,
        AVG(ventes) OVER (PARTITION BY vendeur) AS moyenne,
        STDDEV(ventes) OVER (PARTITION BY vendeur) AS ecart_type
    FROM ventes
)
SELECT
    vendeur,
    ventes,
    ventes - moyenne AS ecart,
    (ventes - moyenne) / NULLIF(ecart_type, 0) AS z_score
FROM stats;
```

### Réutiliser les Windows

```sql
SELECT
    date,
    ventes,
    SUM(ventes) OVER w AS total,
    AVG(ventes) OVER w AS moyenne,
    MAX(ventes) OVER w AS maximum,
    COUNT(*) OVER w AS nb
FROM ventes  
WINDOW w AS (PARTITION BY EXTRACT(YEAR FROM date) ORDER BY date);  
```

### Limiter les Frames Larges

```sql
-- ❌ Coûteux si la table est grande
SUM(ventes) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- ✅ Si vous n'avez besoin que des 90 derniers jours
SUM(ventes) OVER (ORDER BY date ROWS BETWEEN 89 PRECEDING AND CURRENT ROW)
```

### Index sur les Colonnes de Tri

```sql
-- Pour optimiser :
SUM(ventes) OVER (PARTITION BY region ORDER BY date)

-- Créez :
CREATE INDEX idx_ventes_region_date ON ventes(region, date);
```

## Pièges et Erreurs Courantes

### 1. Division par Zéro

```sql
-- ❌ Peut planter
ventes / SUM(ventes) OVER ()

-- ✅ Protégez avec NULLIF
ventes / NULLIF(SUM(ventes) OVER (), 0)
```

### 2. Frames par Défaut avec ORDER BY

```sql
-- Attention : frame par défaut = RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
SUM(ventes) OVER (ORDER BY date)

-- Soyez explicite si vous voulez toute la partition :
SUM(ventes) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
```

ℹ️ **Rappel** : le mode `RANGE` par défaut regroupe **les ex-aequo** sur la clé d'ORDER BY (cf. chapitre 10.3). Si plusieurs lignes ont la même date, leur cumul sera identique.

### 3. NULL dans les Calculs

⚠️ **Important** : les fonctions d'agrégation SQL **ignorent automatiquement les NULL** (sauf `COUNT(*)`). Donc :

- `SUM(prix)`, `AVG(prix)`, `MIN(prix)`, `MAX(prix)` : ignorent les NULL
- `COUNT(prix)` : compte uniquement les valeurs non-NULL
- `COUNT(*)` : compte **toutes** les lignes (NULL inclus)

Le vrai piège est dans les **opérations arithmétiques** sur les NULL : `prix - AVG(prix)` retourne `NULL` si `prix` lui-même vaut `NULL` (et non si la moyenne est NULL).

```sql
-- Cas 1 : prix = NULL pour cette ligne → resultat = NULL
prix - AVG(prix) OVER ()

-- Solution 1 : utiliser COALESCE sur la colonne pour neutraliser les NULL
COALESCE(prix, 0) - AVG(prix) OVER ()
-- ⚠️ Mais attention : cela modifie aussi la "ligne courante" en 0

-- Solution 2 : filtrer les NULL en amont
SELECT prix - AVG(prix) OVER () AS ecart
FROM produits
WHERE prix IS NOT NULL;
```

ℹ️ **Cas particulier** : `AVG(...)` ne retourne `NULL` que si **toutes** les valeurs sont NULL. Dans ce cas, `COALESCE(AVG(prix) OVER (), 0)` peut être utile.

### 4. Window Function sur le Résultat d'un GROUP BY

On **peut** mélanger `GROUP BY` et window functions dans la **même** requête, à condition que la window prenne en argument une **expression agrégée** : après le regroupement, une colonne brute n'est plus accessible.

```sql
-- ❌ ERREUR : `ventes` (brut) n'est ni groupé ni agrégé dans la window function
SELECT
    categorie,
    SUM(ventes) AS total_categorie,
    SUM(ventes) OVER () AS total_global
FROM ventes
GROUP BY categorie;
-- ERROR: column "ventes.ventes" must appear in the GROUP BY clause
--        or be used in an aggregate function

-- ✅ CORRECT (directement, sans sous-requête) : imbriquer l'agrégat DANS la window
SELECT
    categorie,
    SUM(ventes) AS total_categorie,
    SUM(SUM(ventes)) OVER () AS total_global   -- window SUR l'agrégat de chaque groupe
FROM ventes
GROUP BY categorie;
```

`SUM(SUM(ventes)) OVER ()` se lit « la window `SUM` appliquée à l'agrégat `SUM(ventes)` de chaque catégorie » : elle additionne donc les totaux par catégorie pour donner le total global, **sans CTE**. Une **CTE** reste néanmoins une alternative souvent plus lisible quand la requête se complexifie :

```sql
WITH totaux_cat AS (
    SELECT categorie, SUM(ventes) AS total_categorie
    FROM ventes
    GROUP BY categorie
)
SELECT
    categorie,
    total_categorie,
    SUM(total_categorie) OVER () AS total_global
FROM totaux_cat;
```

## Points Clés à Retenir

- ✅ **SUM()** pour cumuls et totaux glissants  
- ✅ **AVG()** pour moyennes mobiles et comparaisons  
- ✅ **COUNT()** pour comptages et fréquences  
- ✅ **MIN()/MAX()** pour records et extremums  
- ✅ **STDDEV()/VARIANCE()** pour volatilité et dispersion  
- ✅ **FILTER (WHERE ...)** pour comptages/sommes conditionnels élégants  
- ✅ Utilisez **frames personnalisés** pour fenêtres glissantes  
- ✅ Combinez plusieurs agrégations avec la clause **WINDOW**  
- ✅ Protégez contre les **divisions par zéro** avec NULLIF  
- ✅ Les agrégations **ignorent les NULL** automatiquement (sauf `COUNT(*)`)  
- ✅ Les index sur **PARTITION BY et ORDER BY** améliorent les performances  
- ✅ Les **ordered-set aggregates** (PERCENTILE_CONT/DISC, MODE) ne fonctionnent pas en OVER

## Tableau de Choix Rapide

| Besoin | Fonction à Utiliser |
|--------|---------------------|
| Cumul chronologique | `SUM() OVER (ORDER BY date)` |
| Moyenne mobile | `AVG() OVER (... ROWS BETWEEN N PRECEDING AND CURRENT ROW)` |
| Total par groupe | `SUM() OVER (PARTITION BY groupe)` |
| Pourcentage du total | `valeur / NULLIF(SUM(valeur) OVER (), 0)` |
| Record historique | `MAX() OVER (ORDER BY date)` |
| Volatilité | `STDDEV() OVER (...)` |
| Comptage conditionnel | `COUNT(*) FILTER (WHERE condition) OVER (...)` |
| Détection d'anomalies | Z-score avec AVG() et STDDEV() |
| Analyse de tendance | Moyennes mobiles multiples |

## Résumé Visuel

```sql
-- Template général pour agrégations en fenêtrage
SELECT
    colonnes,
    FONCTION_AGREGATION(colonne)
        [FILTER (WHERE condition)]    -- Filtrage conditionnel (optionnel)
        OVER (
            [PARTITION BY groupe]     -- Groupes séparés (optionnel)
            [ORDER BY ordre]          -- Ordre de calcul (optionnel)
            [frame_specification]     -- Fenêtre précise (optionnel)
        ) AS resultat
FROM nom_table;

-- Utilisables en window function (OVER) :
-- SUM, AVG, COUNT, MIN, MAX                     -- General-purpose
-- STDDEV, VARIANCE, STDDEV_SAMP, VAR_SAMP       -- Statistical
-- CORR, COVAR_POP, COVAR_SAMP, REGR_*           -- Statistical (multi-args)

-- NON utilisables en OVER (ordered-set aggregates) :
-- PERCENTILE_CONT, PERCENTILE_DISC, MODE        -- Utiliser GROUP BY à la place

-- FILTER (WHERE ...) : valide UNIQUEMENT sur les agrégations en window function
-- (pas sur ROW_NUMBER, RANK, LAG, LEAD, etc.)
```

---


⏭️ [Cas d'usage avancés : Top N par groupe, moyenne mobile](/10-fonctions-de-fenetrage/07-cas-usage-avances.md)
