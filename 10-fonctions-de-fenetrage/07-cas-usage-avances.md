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

### Alternatives PostgreSQL au Top N

Selon le besoin, deux alternatives PostgreSQL peuvent être préférables aux window functions :

#### `DISTINCT ON` — Idéal pour le Top 1 par groupe

PostgreSQL propose la clause **non-standard** `DISTINCT ON (colonnes)` qui sélectionne la **première ligne** de chaque groupe selon l'`ORDER BY`. Bien plus concis qu'un `ROW_NUMBER + WHERE rn = 1` :

```sql
-- Le produit le plus vendu de chaque catégorie
SELECT DISTINCT ON (categorie)
    categorie, produit, ventes
FROM produits
ORDER BY categorie, ventes DESC;  -- L'ordre détermine quelle ligne est "première"
```

**Limitations** : ne retourne qu'**une ligne par groupe**. Pour N > 1, retomber sur `ROW_NUMBER`.

#### `LATERAL` + `LIMIT` — Performant pour Top N avec peu de groupes

Quand on a une table de groupes (catégories, vendeurs, etc.) et qu'on veut le top N de chacun, `LATERAL` permet à PostgreSQL d'utiliser un index pour ne lire que les N premières lignes par groupe — souvent plus rapide qu'un `ROW_NUMBER` qui doit trier toute la table :

```sql
-- Top 3 produits par catégorie (en supposant un index sur (categorie, ventes DESC))
SELECT c.categorie, p.produit, p.ventes
FROM categories c
CROSS JOIN LATERAL (
    SELECT produit, ventes
    FROM produits
    WHERE categorie = c.categorie
    ORDER BY ventes DESC
    LIMIT 3
) AS p;
```

**Quand utiliser quoi** :
- **`DISTINCT ON`** : top 1 par groupe, syntaxe compacte
- **`LATERAL` + `LIMIT`** : top N avec un bon index et peu de groupes (très performant)
- **`ROW_NUMBER`** : top N en mode "scan complet", flexible, standard SQL

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

⚠️ **Piège classique** : ne pas joindre par `c.date = e.date + INTERVAL '1 day'`. Cette approche casse dès qu'un jour manque (week-end, jour férié, panne de collecte). Il faut **numéroter les lignes** avec `ROW_NUMBER()` puis joindre par numéro consécutif :

```sql
WITH RECURSIVE numerote AS (
    -- On numérote chaque cours selon l'ordre chronologique
    SELECT
        date,
        cours,
        ROW_NUMBER() OVER (ORDER BY date) AS rn
    FROM cours_bourse
),
ema AS (
    -- Cas de base : la première ligne, l'EMA = cours
    SELECT date, cours, rn, cours::numeric AS ema_12
    FROM numerote
    WHERE rn = 1

    UNION ALL

    -- Récursion : EMA_n = cours_n * α + EMA_(n-1) * (1 - α), α = 2/(N+1) = 2/13
    SELECT
        n.date,
        n.cours,
        n.rn,
        (n.cours * (2.0 / 13)) + (e.ema_12 * (11.0 / 13)) AS ema_12
    FROM numerote n
    JOIN ema e ON n.rn = e.rn + 1
)
SELECT date, cours, ROUND(ema_12, 2) AS ema_12
FROM ema
ORDER BY date;
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

⚠️ **Subtilité de référence** : pour détecter une anomalie, la moyenne et l'écart-type doivent porter sur les **autres lignes**, pas inclure la ligne courante (sinon une valeur extrême tire sa propre moyenne et masque l'anomalie). On utilise `1 PRECEDING` comme borne supérieure (ou `EXCLUDE CURRENT ROW` en PG 11+) :

```sql
WITH stats AS (
    SELECT
        date,
        ventes,
        AVG(ventes)    OVER w AS mm7_precedent,
        STDDEV(ventes) OVER w AS ecart_type_7j_precedent
    FROM ventes_quotidiennes
    WINDOW w AS (
        ORDER BY date
        ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING  -- 7 jours précédents, hors aujourd'hui
    )
)
SELECT
    date,
    ventes,
    mm7_precedent,
    ecart_type_7j_precedent,
    CASE
        WHEN ventes > mm7_precedent + 2 * ecart_type_7j_precedent THEN 'PIC ANORMAL'
        WHEN ventes < mm7_precedent - 2 * ecart_type_7j_precedent THEN 'CHUTE ANORMALE'
        ELSE 'NORMAL'
    END AS alerte
FROM stats
ORDER BY date;
```

**Logique** : Si la valeur du jour s'écarte de plus de 2 écarts-types de la **moyenne des 7 jours précédents**, c'est anormal. La CTE évite aussi la répétition des window functions.

ℹ️ **Variante PG 11+** avec `EXCLUDE CURRENT ROW` : permet d'écrire `ROWS BETWEEN 7 PRECEDING AND CURRENT ROW EXCLUDE CURRENT ROW` (équivalent fonctionnel ici, mais utile dans des fenêtres centrées).

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

Détecter des séquences de succès/échecs consécutifs avec la technique du **"gaps and islands"** : chaque baisse incrémente un compteur `groupe_sequence`, donc toutes les hausses consécutives partagent le même groupe.

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
        -- Compteur cumulatif : incrémente à chaque jour SANS hausse
        SUM(CASE WHEN en_hausse = 0 THEN 1 ELSE 0 END) OVER (ORDER BY date) AS groupe_sequence
    FROM ventes_avec_tendance
)
SELECT
    date,
    ventes,
    -- On numérote les hausses dans chaque groupe (après filtrage)
    ROW_NUMBER() OVER (PARTITION BY groupe_sequence ORDER BY date) AS jours_consecutifs_hausse
FROM sequences
WHERE en_hausse = 1  -- Filtre avant ROW_NUMBER : la numérotation démarre à 1 pour chaque streak
ORDER BY date;
```

**Pourquoi `WHERE` avant `ROW_NUMBER` ?** Le `WHERE` s'applique sur la CTE `sequences` (étape 1), puis le `ROW_NUMBER` s'applique sur les lignes restantes (étape 2). On obtient ainsi 1, 2, 3, ... pour chaque streak de hausses consécutives.

**Résultat** : Combien de jours consécutifs de hausse pour chaque séquence.

### 3. Calcul de Percentile Mobile

Savoir dans quel percentile se trouve une valeur par rapport aux N derniers jours.

⚠️ **Piège** : `PERCENT_RANK()` est une fonction de rang qui **ignore le frame** — `PERCENT_RANK() OVER (ORDER BY date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)` ne calcule **pas** un percentile mobile, juste une progression linéaire par date. De plus, on ne pourrait pas trier sur `ventes` car alors le percentile serait calculé sur **toute** la série, pas sur la fenêtre 30 jours.

**✅ Solution correcte avec `LATERAL`** : pour chaque ligne, calculer la position relative dans la fenêtre des 29 jours précédents + le jour courant :

```sql
SELECT
    v1.date,
    v1.ventes,
    -- Proportion des 30 derniers jours dont les ventes sont <= aux ventes du jour
    ROUND(stats.rang_dans_fenetre * 100.0 / NULLIF(stats.nb_jours_fenetre, 0), 2) AS percentile_pct
FROM ventes_quotidiennes v1
CROSS JOIN LATERAL (
    SELECT
        COUNT(*) FILTER (WHERE v2.ventes <= v1.ventes) AS rang_dans_fenetre,
        COUNT(*)                                       AS nb_jours_fenetre
    FROM ventes_quotidiennes v2
    WHERE v2.date BETWEEN v1.date - 29 AND v1.date
) AS stats
ORDER BY v1.date;
```

**Interprétation** : Si `percentile_pct = 90`, les ventes du jour sont supérieures ou égales à 90 % des 30 derniers jours (top 10 %).

ℹ️ **Performance** : ce pattern fait une jointure par ligne (coûteux sur de grandes tables). Pour un vrai usage production sur séries longues, envisager une **vue matérialisée** rafraîchie quotidiennement.

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

Calculer la rétention par cohorte (mois d'inscription).

⚠️ **Piège classique** : `EXTRACT(MONTH FROM AGE(...))` ne retourne **que le composant mois** (0-11), pas le **nombre total de mois écoulés**. Pour un écart de 13 mois, ça donne 1 (et non 13) — les cohortes seraient mal agrégées. On combine les composants années + mois, ou on utilise une soustraction directe de `DATE_TRUNC('month', ...)` :

```sql
WITH cohortes AS (
    SELECT
        u.user_id,
        DATE_TRUNC('month', u.date_inscription) AS mois_cohorte,
        DATE_TRUNC('month', a.date_activite) AS mois_activite,
        -- Nombre TOTAL de mois écoulés depuis l'inscription
        (EXTRACT(YEAR  FROM AGE(a.date_activite, u.date_inscription)) * 12
       + EXTRACT(MONTH FROM AGE(a.date_activite, u.date_inscription)))::int AS mois_depuis_inscription
    FROM users u
    LEFT JOIN activites a ON u.user_id = a.user_id
),
retention AS (
    SELECT
        mois_cohorte,
        mois_depuis_inscription,
        COUNT(DISTINCT user_id) AS users_actifs,
        -- La taille initiale = nombre d'utilisateurs au mois 0 de la cohorte
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

Segmentation client classique. **Convention** : un score de **5 = meilleur client** sur la dimension, **1 = moins bon**.

⚠️ **Piège classique** sur le sens de NTILE : `NTILE(5) OVER (ORDER BY x DESC)` attribue le bucket **1 aux plus grandes valeurs** et **5 aux plus petites**. Pour que « bucket 5 = client meilleur », il faut donc :

- **Récence** : `ORDER BY jours_depuis_dernier_achat DESC` → bucket 5 = peu de jours d'inactivité = client récent ✓
- **Fréquence** : `ORDER BY nb_achats ASC` → bucket 5 = beaucoup d'achats = client fréquent ✓
- **Monétaire** : `ORDER BY ca_total ASC` → bucket 5 = gros CA = gros dépensier ✓

```sql
WITH rfm_scores AS (
    SELECT
        client_id,
        MAX(date_achat) AS derniere_achat,
        COUNT(*) AS nb_achats,
        SUM(montant) AS ca_total,
        CURRENT_DATE - MAX(date_achat) AS jours_depuis_dernier_achat,
        -- Bucket 5 = client le plus récent (moins de jours d'inactivité)
        NTILE(5) OVER (ORDER BY CURRENT_DATE - MAX(date_achat) DESC) AS score_recence,
        -- Bucket 5 = client le plus fréquent (le plus d'achats) → ASC pour inverser
        NTILE(5) OVER (ORDER BY COUNT(*) ASC) AS score_frequence,
        -- Bucket 5 = client le plus gros dépensier → ASC pour inverser
        NTILE(5) OVER (ORDER BY SUM(montant) ASC) AS score_montant
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

ℹ️ **Astuce mnémotechnique** : avec `NTILE(N) OVER (ORDER BY x ASC)`, le bucket 1 contient les **plus petites** valeurs et le bucket N les **plus grandes** (intuition « du moins au plus »). Avec `DESC`, c'est l'inverse.

### 7. Analyse de Saisonnalité

Comparer chaque mois à la même période l'année précédente :

⚠️ **Piège** : pour obtenir la moyenne **par mois calendaire** (toutes années confondues), il faut `PARTITION BY EXTRACT(MONTH FROM date)`, **pas** `ORDER BY ...` avec `UNBOUNDED FOLLOWING` (qui donnerait la moyenne globale identique sur toutes les lignes).

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
    -- Moyenne du même mois calendaire sur toutes les années (saisonnalité)
    AVG(ventes) OVER (PARTITION BY EXTRACT(MONTH FROM date)) AS moyenne_mois_tous_ans
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

-- Index unique pour permettre REFRESH CONCURRENTLY (sans bloquer les lectures)
CREATE UNIQUE INDEX idx_vmm_date ON ventes_avec_moyennes_mobiles(date);

-- Rafraîchir quotidiennement
-- ✅ CONCURRENTLY : pas de verrou exclusif, les requêtes continuent à lire l'ancienne version
REFRESH MATERIALIZED VIEW CONCURRENTLY ventes_avec_moyennes_mobiles;

-- ⚠️ Sans CONCURRENTLY : verrou ACCESS EXCLUSIVE pendant tout le rafraîchissement (lectures bloquées)
```

### 2. Limiter la Fenêtre Temporelle (avec Précaution)

```sql
-- ❌ Coûteux : Traite tout l'historique
SELECT
    date,
    AVG(ventes) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS mm7
FROM ventes_quotidiennes;
```

⚠️ **Piège classique** : un simple `WHERE date >= CURRENT_DATE - INTERVAL '1 year'` **tronque la fenêtre** pour les premières lignes du résultat. La moyenne mobile sur 7 jours sera fausse pour les 6 premiers jours (elle ne verra pas les 6 jours antérieurs nécessaires).

**✅ Solution correcte** : étendre la lecture en amont, puis filtrer en sortie via une CTE :

```sql
WITH ventes_avec_mm AS (
    -- Lit 6 jours de plus que la plage cible pour avoir une fenêtre complète
    SELECT
        date,
        ventes,
        AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS mm7
    FROM ventes_quotidiennes
    WHERE date >= CURRENT_DATE - INTERVAL '1 year' - INTERVAL '6 days'
)
SELECT date, ventes, mm7
FROM ventes_avec_mm
WHERE date >= CURRENT_DATE - INTERVAL '1 year'  -- On enlève les 6 jours de "warm-up"
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

Si vous voulez **exactement** N produits par catégorie :

- ✅ Utilisez `ROW_NUMBER()` : garantit exactement N lignes
- ❌ Pas `RANK()` : en cas d'ex-aequo en position N, peut retourner plus de N lignes
- ❌ Pas `DENSE_RANK()` : même problème que `RANK`

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

- ✅ **Top N par groupe** : `ROW_NUMBER()` + `PARTITION BY` + filtre sur rang, ou alternatives PostgreSQL (`DISTINCT ON` pour le top 1, `LATERAL + LIMIT` avec un bon index)  
- ✅ **ROW_NUMBER vs RANK** : `ROW_NUMBER` pour exactement N lignes, `RANK` pour inclure les ex-aequo  
- ✅ **Pagination** : préférer la **keyset pagination** (méthode du seek) à `ROW_NUMBER` ou `OFFSET` sur de grandes tables  
- ✅ **Moyenne mobile** : `AVG()` avec `ROWS BETWEEN N PRECEDING AND CURRENT ROW`  
- ✅ **Moyennes mobiles multiples** : utiles pour détecter les croisements et tendances  
- ✅ **Détection d'anomalies** : moyenne mobile **excluant la ligne courante** (`1 PRECEDING` ou `EXCLUDE CURRENT ROW`) pour ne pas diluer l'écart  
- ✅ **Optimisation** : préférer une CTE avec « warm-up » au lieu d'un `WHERE` qui tronque la fenêtre ; utiliser des index, vues matérialisées (`REFRESH ... CONCURRENTLY`)  
- ✅ **CTE et WINDOW** : améliorent la lisibilité et évitent la duplication  
- ✅ **Z-score** : `(valeur - moyenne) / écart-type` pour détecter les anomalies  
- ✅ **Imbrication interdite** : une window function ne peut pas en contenir une autre — passer par une CTE  
- ✅ **Toujours un tri déterministe** : ajouter une colonne unique en cas d'égalité (`ORDER BY val, id`)

## Tableau Récapitulatif

| Cas d'Usage | Pattern Window Function |
|-------------|------------------------|
| Top 3 par catégorie | `ROW_NUMBER() OVER (PARTITION BY cat ORDER BY val DESC)` filtre <= 3 |
| Moyenne mobile 7 jours | `AVG(val) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` |
| Cumul par groupe | `SUM(val) OVER (PARTITION BY groupe ORDER BY date)` |
| Croissance % vs hier | `(val - LAG(val) OVER (...)) / NULLIF(LAG(val) OVER (...), 0)` |
| Détection anomalies | Comparer à `AVG() ± 2*STDDEV()` sur fenêtre glissante |
| Classement avec ex-aequo | `RANK() OVER (ORDER BY val DESC)` |
| Segmentation RFM | `NTILE(5) OVER (ORDER BY metrique)` |

---

**Fin du Chapitre 10 : Fonctions de Fenêtrage (Window Functions)**

Vous maîtrisez maintenant l'un des outils les plus puissants de PostgreSQL ! Les window functions ouvrent des possibilités d'analyse qui seraient impossibles ou très complexes avec SQL traditionnel.


⏭️ [Modélisation Avancée](/11-modelisation-avancee/README.md)
