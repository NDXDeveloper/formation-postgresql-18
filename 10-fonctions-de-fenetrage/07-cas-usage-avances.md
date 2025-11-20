🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.7. Cas d'usage avancés : Top N par groupe, moyenne mobile

## Introduction

Maintenant que vous maîtrisez toutes les briques des window functions, explorons des **cas d'usage avancés** qui résolvent des problèmes réels rencontrés en production.

Cette section se concentre sur deux patterns particulièrement utiles :

1. **Top N par groupe** : Trouver les N meilleurs/pires éléments dans chaque catégorie
2. **Moyennes mobiles** : Calculer des moyennes glissantes pour détecter des tendances

Nous verrons également d'autres cas d'usage avancés qui combinent plusieurs techniques de window functions.

## Top N par Groupe

### Le Problème

**Scénario typique** : Vous avez des produits dans différentes catégories, et vous voulez les 3 meilleurs produits de chaque catégorie selon les ventes.

**Avec GROUP BY seul** : Impossible ! Vous ne pouvez obtenir qu'un seul maximum par groupe.

**Avec Window Functions** : Facile et élégant.

### Approche 1 : ROW_NUMBER()

La méthode la plus courante et la plus fiable.

```sql
WITH produits_classes AS (
    SELECT
        categorie,
        produit,
        ventes,
        ROW_NUMBER() OVER (
            PARTITION BY categorie
            ORDER BY ventes DESC
        ) AS rang
    FROM produits
)
SELECT categorie, produit, ventes
FROM produits_classes
WHERE rang <= 3
ORDER BY categorie, rang;
```

**Données** :

```
| categorie | produit      | ventes |
|-----------|--------------|--------|
| Laptop    | MacBook Pro  | 5000   |
| Laptop    | ThinkPad X1  | 4500   |
| Laptop    | Dell XPS     | 4200   |
| Laptop    | HP Pavilion  | 3000   |
| Laptop    | Asus Zenbook | 2800   |
| Souris    | MX Master 3  | 1200   |
| Souris    | Magic Mouse  | 800    |
| Souris    | Logitech G   | 750    |
| Souris    | Basic Mouse  | 200    |
```

**Résultat** :

```
| categorie | produit      | ventes |
|-----------|--------------|--------|
| Laptop    | MacBook Pro  | 5000   | ← Top 1 Laptop
| Laptop    | ThinkPad X1  | 4500   | ← Top 2 Laptop
| Laptop    | Dell XPS     | 4200   | ← Top 3 Laptop
| Souris    | MX Master 3  | 1200   | ← Top 1 Souris
| Souris    | Magic Mouse  | 800    | ← Top 2 Souris
| Souris    | Logitech G   | 750    | ← Top 3 Souris
```

**Avantages** :
- Résultat prévisible (toujours exactement N lignes par groupe, s'il y en a assez)
- Pas d'ambiguïté en cas d'égalité

**Inconvénients** :
- En cas d'égalité, une des lignes est choisie arbitrairement

### Approche 2 : RANK()

Si vous voulez **inclure les ex-aequo**.

```sql
WITH produits_classes AS (
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
SELECT categorie, produit, ventes, rang
FROM produits_classes
WHERE rang <= 3
ORDER BY categorie, rang;
```

**Données avec égalités** :

```
| categorie | produit      | ventes |
|-----------|--------------|--------|
| Laptop    | MacBook Pro  | 5000   |
| Laptop    | ThinkPad X1  | 4500   |
| Laptop    | Dell XPS     | 4500   | ← Égalité avec ThinkPad
| Laptop    | HP Pavilion  | 4200   |
| Laptop    | Asus Zenbook | 2800   |
```

**Résultat** :

```
| categorie | produit      | ventes | rang |
|-----------|--------------|--------|------|
| Laptop    | MacBook Pro  | 5000   | 1    |
| Laptop    | ThinkPad X1  | 4500   | 2    |
| Laptop    | Dell XPS     | 4500   | 2    | ← Ex-aequo en 2ème position
| Laptop    | HP Pavilion  | 4200   | 4    | ← Rang 4 (pas 3)
```

**Avantages** :
- Inclut tous les ex-aequo
- Plus "équitable" si les égalités sont importantes

**Inconvénients** :
- Peut retourner plus de N lignes par groupe
- Peut "sauter" des rangs

### Approche 3 : DENSE_RANK()

Pour un classement sans sauts.

```sql
WITH produits_classes AS (
    SELECT
        categorie,
        produit,
        ventes,
        DENSE_RANK() OVER (
            PARTITION BY categorie
            ORDER BY ventes DESC
        ) AS rang
    FROM produits
)
SELECT categorie, produit, ventes, rang
FROM produits_classes
WHERE rang <= 3
ORDER BY categorie, rang;
```

Avec les mêmes données, le HP Pavilion aurait le rang **3** (pas 4).

### Choix de la Fonction

| Besoin | Fonction | Résultat |
|--------|----------|----------|
| Exactement N lignes par groupe | `ROW_NUMBER()` | 3 lignes garanties |
| Inclure tous les ex-aequo | `RANK()` | ≥ 3 lignes possibles |
| Classement compact | `DENSE_RANK()` | ≥ 3 lignes, rangs continus |

**Recommandation générale** : `ROW_NUMBER()` pour la prévisibilité, `RANK()` si les égalités comptent.

### Top N avec Conditions Multiples

Trier d'abord par ventes, puis par prix en cas d'égalité :

```sql
WITH produits_classes AS (
    SELECT
        categorie,
        produit,
        ventes,
        prix,
        ROW_NUMBER() OVER (
            PARTITION BY categorie
            ORDER BY ventes DESC, prix ASC  -- Prix le plus bas en cas d'égalité
        ) AS rang
    FROM produits
)
SELECT categorie, produit, ventes, prix
FROM produits_classes
WHERE rang <= 3
ORDER BY categorie, rang;
```

### Bottom N (Les N Pires)

Pour trouver les 3 **pires** produits :

```sql
WITH produits_classes AS (
    SELECT
        categorie,
        produit,
        ventes,
        ROW_NUMBER() OVER (
            PARTITION BY categorie
            ORDER BY ventes ASC  -- ← ASC au lieu de DESC
        ) AS rang
    FROM produits
)
SELECT categorie, produit, ventes
FROM produits_classes
WHERE rang <= 3
ORDER BY categorie, rang;
```

### Top N Global + Top N par Groupe

Combiner les deux : top 10 global + top 3 par catégorie :

```sql
WITH classements AS (
    SELECT
        categorie,
        produit,
        ventes,
        ROW_NUMBER() OVER (ORDER BY ventes DESC) AS rang_global,
        ROW_NUMBER() OVER (
            PARTITION BY categorie
            ORDER BY ventes DESC
        ) AS rang_categorie
    FROM produits
)
SELECT
    categorie,
    produit,
    ventes,
    rang_global,
    rang_categorie
FROM classements
WHERE rang_global <= 10 OR rang_categorie <= 3
ORDER BY ventes DESC;
```

Cela retourne les produits qui sont :
- Soit dans le top 10 global
- Soit dans le top 3 de leur catégorie

## Moyennes Mobiles

### Le Concept

Une **moyenne mobile** (ou **moving average**) est la moyenne calculée sur une fenêtre **glissante** de N périodes.

**Utilité** :
- Lisser les variations aléatoires (bruit)
- Détecter les tendances
- Signaux d'achat/vente en trading
- Prévisions simples

### Moyenne Mobile Simple (SMA - Simple Moving Average)

#### Sur N Jours

```sql
SELECT
    date,
    ventes,
    AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS mm7,
    AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS mm30
FROM ventes_quotidiennes
ORDER BY date;
```

**Données** :

```
| date       | ventes |
|------------|--------|
| 2025-01-01 | 100    |
| 2025-01-02 | 120    |
| 2025-01-03 | 90     |
| 2025-01-04 | 110    |
| 2025-01-05 | 130    |
| 2025-01-06 | 105    |
| 2025-01-07 | 115    |
| 2025-01-08 | 125    |
```

**Résultat (extrait)** :

```
| date       | ventes | mm7   |
|------------|--------|-------|
| 2025-01-01 | 100    | 100.0 | ← 1 jour
| 2025-01-02 | 120    | 110.0 | ← (100+120)/2
| 2025-01-03 | 90     | 103.3 | ← (100+120+90)/3
| ...        | ...    | ...   |
| 2025-01-07 | 115    | 110.0 | ← Moyenne des 7 derniers jours
| 2025-01-08 | 125    | 113.6 | ← Fenêtre glisse
```

**Observation** : La MM7 "lisse" les variations brutales.

#### Visualisation de la Fenêtre Glissante

```
Jour 1 : [100]                           → MM = 100
Jour 2 : [100, 120]                      → MM = 110
Jour 3 : [100, 120, 90]                  → MM = 103.3
...
Jour 7 : [100, 120, 90, 110, 130, 105, 115] → MM = 110
Jour 8 : [120, 90, 110, 130, 105, 115, 125] → MM = 113.6
         ↑ Le 100 du jour 1 est sorti de la fenêtre
```

### Moyenne Mobile par Groupe

Calculer une moyenne mobile **par vendeur** :

```sql
SELECT
    vendeur,
    date,
    ventes,
    AVG(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS mm7_vendeur
FROM ventes
ORDER BY vendeur, date;
```

Chaque vendeur a sa propre moyenne mobile indépendante.

### Moyenne Mobile Centrée

Au lieu de regarder en arrière, regarder **autour** de la date :

```sql
SELECT
    date,
    temperature,
    AVG(temperature) OVER (
        ORDER BY date
        ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING
    ) AS mm7_centree
FROM meteo
ORDER BY date;
```

**Utilité** : Lissage plus équilibré, mais nécessite de connaître les données futures (utile pour l'analyse historique, pas pour la prédiction en temps réel).

### Moyennes Mobiles Multiples (Croisement)

Stratégie classique en trading : détecter le croisement de deux moyennes mobiles.

```sql
SELECT
    date,
    cours,
    AVG(cours) OVER (
        ORDER BY date
        ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
    ) AS mm20,
    AVG(cours) OVER (
        ORDER BY date
        ROWS BETWEEN 49 PRECEDING AND CURRENT ROW
    ) AS mm50,
    CASE
        WHEN AVG(cours) OVER (
            ORDER BY date
            ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) > AVG(cours) OVER (
            ORDER BY date
            ROWS BETWEEN 49 PRECEDING AND CURRENT ROW
        )
        THEN 'Tendance haussière'
        ELSE 'Tendance baissière'
    END AS signal
FROM cours_bourse
ORDER BY date;
```

**Signal** :
- MM20 > MM50 : Tendance **haussière** (signal d'achat)
- MM20 < MM50 : Tendance **baissière** (signal de vente)

### Moyenne Mobile Exponentielle (EMA - Conceptuel)

**Note** : PostgreSQL n'a pas de fonction EMA native, mais on peut l'implémenter.

L'EMA donne plus de poids aux valeurs **récentes** :

```sql
WITH RECURSIVE ema AS (
    -- Première valeur : prix initial
    SELECT
        date,
        cours,
        cours AS ema_12
    FROM cours_bourse
    WHERE date = (SELECT MIN(date) FROM cours_bourse)

    UNION ALL

    -- Calcul récursif
    SELECT
        c.date,
        c.cours,
        (c.cours * (2.0 / (12 + 1))) + (e.ema_12 * (1 - (2.0 / (12 + 1)))) AS ema_12
    FROM cours_bourse c
    JOIN ema e ON c.date = e.date + INTERVAL '1 day'
)
SELECT * FROM ema ORDER BY date;
```

**Formule EMA** : `EMA_aujourd'hui = (Cours * α) + (EMA_hier * (1-α))`

Où `α = 2 / (N + 1)` (facteur de lissage)

**Complexité** : L'EMA nécessite une approche récursive, moins intuitive. Pour la plupart des cas, la SMA suffit.

### Moyenne Mobile Pondérée (WMA)

Donner des poids différents aux jours (le plus récent a le poids le plus élevé) :

```sql
SELECT
    date,
    cours,
    (
        cours * 5 +
        LAG(cours, 1) OVER (ORDER BY date) * 4 +
        LAG(cours, 2) OVER (ORDER BY date) * 3 +
        LAG(cours, 3) OVER (ORDER BY date) * 2 +
        LAG(cours, 4) OVER (ORDER BY date) * 1
    ) / 15.0 AS wma_5
FROM cours_bourse;
```

Poids : Jour actuel (5), J-1 (4), J-2 (3), J-3 (2), J-4 (1). Total = 15.

**Plus réactive** que la SMA simple aux changements récents.

### Détecter les Anomalies avec Moyenne Mobile

```sql
SELECT
    date,
    ventes,
    AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS mm7,
    STDDEV(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS ecart_type_7j,
    CASE
        WHEN ventes > AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) + 2 * STDDEV(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        )
        THEN 'PIC ANORMAL'
        WHEN ventes < AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) - 2 * STDDEV(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        )
        THEN 'CHUTE ANORMALE'
        ELSE 'NORMAL'
    END AS alerte
FROM ventes_quotidiennes
ORDER BY date;
```

**Logique** : Si une valeur s'écarte de plus de 2 écarts-types de la moyenne mobile, c'est anormal.

## Autres Cas d'Usage Avancés

### 1. Analyse de Gap (Écarts entre Lignes Consécutives)

Détecter les écarts importants entre valeurs consécutives :

```sql
SELECT
    date,
    cours,
    LAG(cours) OVER (ORDER BY date) AS cours_veille,
    cours - LAG(cours) OVER (ORDER BY date) AS gap,
    ROUND(
        (cours - LAG(cours) OVER (ORDER BY date)) * 100.0 /
        NULLIF(LAG(cours) OVER (ORDER BY date), 0),
        2
    ) AS gap_pct,
    CASE
        WHEN ABS((cours - LAG(cours) OVER (ORDER BY date)) * 100.0 /
                 NULLIF(LAG(cours) OVER (ORDER BY date), 0)) > 5
        THEN 'GAP SIGNIFICATIF'
        ELSE 'Normal'
    END AS analyse
FROM cours_bourse
ORDER BY date;
```

**Usage** : En bourse, les "gaps" (écarts d'ouverture) sont des signaux importants.

### 2. Séquences et Streaks

Détecter des séquences de succès/échecs consécutifs :

```sql
WITH ventes_avec_tendance AS (
    SELECT
        date,
        ventes,
        LAG(ventes) OVER (ORDER BY date) AS ventes_veille,
        CASE
            WHEN ventes > LAG(ventes) OVER (ORDER BY date) THEN 1
            ELSE 0
        END AS en_hausse
    FROM ventes_quotidiennes
),
sequences AS (
    SELECT
        date,
        ventes,
        en_hausse,
        SUM(CASE WHEN en_hausse = 0 THEN 1 ELSE 0 END) OVER (ORDER BY date) AS groupe_sequence
    FROM ventes_avec_tendance
)
SELECT
    date,
    ventes,
    en_hausse,
    ROW_NUMBER() OVER (PARTITION BY groupe_sequence ORDER BY date) AS jours_consecutifs_hausse
FROM sequences
WHERE en_hausse = 1
ORDER BY date;
```

**Résultat** : Combien de jours consécutifs de hausse pour chaque séquence.

### 3. Calcul de Percentile Mobile

Savoir dans quel percentile se trouve une valeur par rapport aux N derniers jours :

```sql
SELECT
    date,
    ventes,
    PERCENT_RANK() OVER (
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS percentile_30j
FROM ventes_quotidiennes;
```

**Interprétation** : Si `percentile_30j = 0.9`, les ventes d'aujourd'hui sont dans le top 10% des 30 derniers jours.

### 4. Premier et Dernier par Groupe avec Détails

Trouver la première et dernière commande de chaque client avec tous les détails :

```sql
WITH commandes_classees AS (
    SELECT
        client_id,
        date_commande,
        montant,
        produit,
        ROW_NUMBER() OVER (
            PARTITION BY client_id
            ORDER BY date_commande
        ) AS numero_commande,
        COUNT(*) OVER (PARTITION BY client_id) AS total_commandes
    FROM commandes
)
SELECT
    client_id,
    date_commande,
    montant,
    produit,
    CASE
        WHEN numero_commande = 1 THEN 'Première commande'
        WHEN numero_commande = total_commandes THEN 'Dernière commande'
        ELSE 'Commande intermédiaire'
    END AS type_commande
FROM commandes_classees
WHERE numero_commande = 1 OR numero_commande = total_commandes
ORDER BY client_id, numero_commande;
```

### 5. Analyse de Cohorte Avancée

Calculer la rétention par cohorte (mois d'inscription) :

```sql
WITH cohortes AS (
    SELECT
        u.user_id,
        DATE_TRUNC('month', u.date_inscription) AS mois_cohorte,
        DATE_TRUNC('month', a.date_activite) AS mois_activite,
        EXTRACT(MONTH FROM AGE(a.date_activite, u.date_inscription)) AS mois_depuis_inscription
    FROM users u
    LEFT JOIN activites a ON u.user_id = a.user_id
),
retention AS (
    SELECT
        mois_cohorte,
        mois_depuis_inscription,
        COUNT(DISTINCT user_id) AS users_actifs,
        FIRST_VALUE(COUNT(DISTINCT user_id)) OVER (
            PARTITION BY mois_cohorte
            ORDER BY mois_depuis_inscription
        ) AS taille_cohorte_initiale
    FROM cohortes
    GROUP BY mois_cohorte, mois_depuis_inscription
)
SELECT
    mois_cohorte,
    mois_depuis_inscription,
    users_actifs,
    taille_cohorte_initiale,
    ROUND(
        users_actifs * 100.0 / NULLIF(taille_cohorte_initiale, 0),
        2
    ) AS taux_retention_pct
FROM retention
ORDER BY mois_cohorte, mois_depuis_inscription;
```

### 6. Analyse RFM (Recency, Frequency, Monetary)

Segmentation client classique :

```sql
WITH rfm_scores AS (
    SELECT
        client_id,
        MAX(date_achat) AS derniere_achat,
        COUNT(*) AS nb_achats,
        SUM(montant) AS ca_total,
        CURRENT_DATE - MAX(date_achat) AS jours_depuis_dernier_achat,
        NTILE(5) OVER (ORDER BY CURRENT_DATE - MAX(date_achat) DESC) AS score_recence,
        NTILE(5) OVER (ORDER BY COUNT(*) DESC) AS score_frequence,
        NTILE(5) OVER (ORDER BY SUM(montant) DESC) AS score_montant
    FROM achats
    GROUP BY client_id
)
SELECT
    client_id,
    jours_depuis_dernier_achat,
    nb_achats,
    ca_total,
    score_recence,
    score_frequence,
    score_montant,
    score_recence + score_frequence + score_montant AS score_rfm_total,
    CASE
        WHEN score_recence = 5 AND score_frequence = 5 AND score_montant = 5
        THEN 'Champions'
        WHEN score_recence >= 4 AND score_frequence >= 4
        THEN 'Clients fidèles'
        WHEN score_recence >= 3 AND score_montant >= 4
        THEN 'Gros dépensiers'
        WHEN score_recence <= 2
        THEN 'À risque'
        ELSE 'Nécessite attention'
    END AS segment
FROM rfm_scores
ORDER BY score_rfm_total DESC;
```

### 7. Analyse de Saisonnalité

Comparer chaque mois à la même période l'année précédente :

```sql
SELECT
    date,
    ventes,
    LAG(ventes, 12) OVER (ORDER BY date) AS ventes_annee_precedente,
    ventes - LAG(ventes, 12) OVER (ORDER BY date) AS variation_annuelle,
    ROUND(
        (ventes - LAG(ventes, 12) OVER (ORDER BY date)) * 100.0 /
        NULLIF(LAG(ventes, 12) OVER (ORDER BY date), 0),
        2
    ) AS croissance_annuelle_pct,
    AVG(ventes) OVER (
        ORDER BY EXTRACT(MONTH FROM date)
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS moyenne_mois_tous_ans
FROM ventes_mensuelles
WHERE date >= CURRENT_DATE - INTERVAL '3 years'
ORDER BY date;
```

### 8. Détection de Changement de Niveau (Change Point Detection)

Identifier quand une métrique change significativement de niveau :

```sql
WITH stats_glissantes AS (
    SELECT
        date,
        metrique,
        AVG(metrique) OVER (
            ORDER BY date
            ROWS BETWEEN 29 PRECEDING AND 1 PRECEDING
        ) AS moyenne_avant,
        AVG(metrique) OVER (
            ORDER BY date
            ROWS BETWEEN CURRENT ROW AND 29 FOLLOWING
        ) AS moyenne_apres,
        STDDEV(metrique) OVER (
            ORDER BY date
            ROWS BETWEEN 29 PRECEDING AND 1 PRECEDING
        ) AS ecart_type_avant
    FROM metriques
)
SELECT
    date,
    metrique,
    moyenne_avant,
    moyenne_apres,
    ABS(moyenne_apres - moyenne_avant) / NULLIF(ecart_type_avant, 0) AS significativite,
    CASE
        WHEN ABS(moyenne_apres - moyenne_avant) / NULLIF(ecart_type_avant, 0) > 2
        THEN 'CHANGEMENT SIGNIFICATIF'
        ELSE 'Stable'
    END AS alerte
FROM stats_glissantes
WHERE ecart_type_avant IS NOT NULL
ORDER BY date;
```

## Optimisations et Bonnes Pratiques

### 1. Utiliser des Vues Matérialisées

Pour des calculs coûteux répétés :

```sql
CREATE MATERIALIZED VIEW ventes_avec_moyennes_mobiles AS
SELECT
    date,
    ventes,
    AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS mm7,
    AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS mm30
FROM ventes_quotidiennes;

-- Créer un index
CREATE INDEX idx_vmm_date ON ventes_avec_moyennes_mobiles(date);

-- Rafraîchir quotidiennement
REFRESH MATERIALIZED VIEW ventes_avec_moyennes_mobiles;
```

### 2. Limiter la Fenêtre Temporelle

```sql
-- ❌ Coûteux : Traite toute l'historique
SELECT
    date,
    AVG(ventes) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS mm7
FROM ventes_quotidiennes;

-- ✅ Optimisé : Filtre d'abord
SELECT
    date,
    AVG(ventes) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS mm7
FROM ventes_quotidiennes
WHERE date >= CURRENT_DATE - INTERVAL '1 year'
ORDER BY date;
```

### 3. Pré-agréger les Données

Pour des analyses sur de très longues périodes :

```sql
-- Pré-agrégation quotidienne
CREATE TABLE ventes_quotidiennes_agg AS
SELECT
    date,
    SUM(montant) AS ventes_journalieres,
    COUNT(*) AS nb_transactions,
    AVG(montant) AS panier_moyen
FROM transactions
GROUP BY date;

-- Ensuite, calcul de moyennes mobiles sur données agrégées
SELECT
    date,
    ventes_journalieres,
    AVG(ventes_journalieres) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS mm7
FROM ventes_quotidiennes_agg;
```

### 4. Index Appropriés

```sql
-- Pour optimiser les window functions avec PARTITION BY et ORDER BY
CREATE INDEX idx_ventes_categorie_date
ON ventes(categorie, date);

-- Permet d'accélérer :
SELECT
    categorie,
    date,
    AVG(montant) OVER (PARTITION BY categorie ORDER BY date)
FROM ventes;
```

### 5. Clause WINDOW Réutilisable

```sql
SELECT
    date,
    ventes,
    AVG(ventes) OVER w AS mm7,
    STDDEV(ventes) OVER w AS volatilite_7j,
    MAX(ventes) OVER w AS max_7j,
    MIN(ventes) OVER w AS min_7j
FROM ventes_quotidiennes
WINDOW w AS (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW);
```

## Pièges à Éviter

### 1. Fenêtres Trop Larges sur Grandes Tables

```sql
-- ❌ Très coûteux sur 10M de lignes
AVG(ventes) OVER (ORDER BY date ROWS BETWEEN 364 PRECEDING AND CURRENT ROW)

-- ✅ Limiter d'abord avec WHERE
SELECT ... FROM ventes WHERE date >= '2024-01-01'
```

### 2. Ne pas Gérer les Données Manquantes

```sql
-- ❌ La moyenne mobile peut être faussée par des NULL
AVG(ventes) OVER (...)

-- ✅ Filtrer ou remplacer les NULL
AVG(COALESCE(ventes, 0)) OVER (...)
```

### 3. Oublier le Tri Secondaire pour ROW_NUMBER()

```sql
-- ❌ Ordre instable en cas d'égalité
ROW_NUMBER() OVER (PARTITION BY categorie ORDER BY ventes DESC)

-- ✅ Ajouter un tri unique
ROW_NUMBER() OVER (PARTITION BY categorie ORDER BY ventes DESC, produit_id)
```

### 4. Mauvaise Utilisation de RANK vs ROW_NUMBER

```sql
-- Si vous voulez EXACTEMENT 3 produits par catégorie
-- ✅ Utilisez ROW_NUMBER()
-- ❌ PAS RANK() (peut retourner plus de 3)
```

### 5. Calculs Redondants

```sql
-- ❌ Calcul de la même window plusieurs fois
ventes / SUM(ventes) OVER (PARTITION BY categorie),
AVG(ventes) OVER (PARTITION BY categorie)

-- ✅ Utiliser une CTE
WITH stats AS (
    SELECT
        *,
        SUM(ventes) OVER (PARTITION BY categorie) AS total_cat,
        AVG(ventes) OVER (PARTITION BY categorie) AS moyenne_cat
    FROM ventes
)
SELECT *, ventes / total_cat AS part_categorie FROM stats;
```

## Exemples Complets de Production

### Exemple 1 : Dashboard de Performance Vendeur

```sql
WITH stats_vendeur AS (
    SELECT
        vendeur_id,
        DATE_TRUNC('month', date_vente) AS mois,
        SUM(montant) AS ca_mensuel,
        COUNT(*) AS nb_ventes,
        AVG(montant) AS panier_moyen
    FROM ventes
    WHERE date_vente >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY vendeur_id, DATE_TRUNC('month', date_vente)
),
metriques_enrichies AS (
    SELECT
        vendeur_id,
        mois,
        ca_mensuel,
        nb_ventes,
        panier_moyen,
        -- Rang du mois
        ROW_NUMBER() OVER (PARTITION BY vendeur_id ORDER BY mois) AS numero_mois,
        -- Top vendeurs du mois
        RANK() OVER (PARTITION BY mois ORDER BY ca_mensuel DESC) AS rang_mois,
        -- Évolutions
        LAG(ca_mensuel) OVER (PARTITION BY vendeur_id ORDER BY mois) AS ca_mois_precedent,
        ca_mensuel - LAG(ca_mensuel) OVER (PARTITION BY vendeur_id ORDER BY mois) AS variation_mois,
        -- Moyennes mobiles
        AVG(ca_mensuel) OVER (
            PARTITION BY vendeur_id
            ORDER BY mois
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS mm3_mois,
        -- Cumul annuel
        SUM(ca_mensuel) OVER (
            PARTITION BY vendeur_id, EXTRACT(YEAR FROM mois)
            ORDER BY mois
        ) AS ca_cumule_annuel,
        -- Stats globales
        AVG(ca_mensuel) OVER (PARTITION BY vendeur_id) AS ca_moyen_vendeur,
        STDDEV(ca_mensuel) OVER (PARTITION BY vendeur_id) AS volatilite_vendeur
    FROM stats_vendeur
)
SELECT
    v.nom AS vendeur,
    m.mois,
    m.ca_mensuel,
    m.nb_ventes,
    m.panier_moyen,
    m.rang_mois,
    m.variation_mois,
    ROUND(
        m.variation_mois * 100.0 / NULLIF(m.ca_mois_precedent, 0),
        2
    ) AS croissance_pct,
    m.mm3_mois,
    m.ca_cumule_annuel,
    ROUND(
        (m.ca_mensuel - m.ca_moyen_vendeur) / NULLIF(m.volatilite_vendeur, 0),
        2
    ) AS z_score,
    CASE
        WHEN m.ca_mensuel > m.ca_moyen_vendeur + m.volatilite_vendeur
        THEN 'Excellent mois'
        WHEN m.ca_mensuel < m.ca_moyen_vendeur - m.volatilite_vendeur
        THEN 'Mois difficile'
        ELSE 'Performance normale'
    END AS evaluation
FROM metriques_enrichies m
JOIN vendeurs v ON m.vendeur_id = v.id
ORDER BY m.mois DESC, m.rang_mois;
```

### Exemple 2 : Système de Recommandation Simple

Produits fréquemment achetés ensemble :

```sql
WITH paires_produits AS (
    SELECT
        c1.produit_id AS produit_a,
        c2.produit_id AS produit_b,
        COUNT(*) AS nb_achats_ensemble
    FROM commande_items c1
    JOIN commande_items c2
        ON c1.commande_id = c2.commande_id
        AND c1.produit_id < c2.produit_id
    GROUP BY c1.produit_id, c2.produit_id
    HAVING COUNT(*) >= 5  -- Au moins 5 achats conjoints
),
produits_recommandes AS (
    SELECT
        produit_a,
        produit_b,
        nb_achats_ensemble,
        ROW_NUMBER() OVER (
            PARTITION BY produit_a
            ORDER BY nb_achats_ensemble DESC
        ) AS rang_recommandation
    FROM paires_produits
)
SELECT
    pa.nom AS produit_source,
    pb.nom AS produit_recommande,
    pr.nb_achats_ensemble,
    pr.rang_recommandation
FROM produits_recommandes pr
JOIN produits pa ON pr.produit_a = pa.id
JOIN produits pb ON pr.produit_b = pb.id
WHERE pr.rang_recommandation <= 5  -- Top 5 recommandations
ORDER BY pr.produit_a, pr.rang_recommandation;
```

## Points Clés à Retenir

- ✅ **Top N par groupe** : Utilisez ROW_NUMBER() avec PARTITION BY puis filtrez sur le rang
- ✅ **ROW_NUMBER vs RANK** : ROW_NUMBER pour exactement N lignes, RANK pour inclure les ex-aequo
- ✅ **Moyenne mobile** : AVG() avec ROWS BETWEEN N PRECEDING AND CURRENT ROW
- ✅ **Moyennes mobiles multiples** : Utiles pour détecter les croisements et tendances
- ✅ **Optimisation** : Filtrez d'abord avec WHERE, utilisez des index, considérez les vues matérialisées
- ✅ **CTE et WINDOW** : Améliorent la lisibilité et évitent la duplication
- ✅ **Z-score** : (valeur - moyenne) / écart-type pour détecter les anomalies
- ✅ **Toujours un tri déterministe** : Ajoutez une colonne unique en cas d'égalité

## Tableau Récapitulatif

| Cas d'Usage | Pattern Window Function |
|-------------|------------------------|
| Top 3 par catégorie | `ROW_NUMBER() OVER (PARTITION BY cat ORDER BY val DESC)` filtre <= 3 |
| Moyenne mobile 7 jours | `AVG(val) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` |
| Cumul par groupe | `SUM(val) OVER (PARTITION BY groupe ORDER BY date)` |
| Croissance % vs hier | `(val - LAG(val) OVER (...)) / LAG(val) OVER (...)` |
| Détection anomalies | Comparer à `AVG() ± 2*STDDEV()` sur fenêtre glissante |
| Classement avec ex-aequo | `RANK() OVER (ORDER BY val DESC)` |
| Segmentation RFM | `NTILE(5) OVER (ORDER BY metrique)` |

---

**Fin du Chapitre 10 : Fonctions de Fenêtrage (Window Functions)**

Vous maîtrisez maintenant l'un des outils les plus puissants de PostgreSQL ! Les window functions ouvrent des possibilités d'analyse qui seraient impossibles ou très complexes avec SQL traditionnel.


⏭️ [Modélisation Avancée](/11-modelisation-avancee/README.md)
