🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.5. Fonctions de valeur (LAG, LEAD, FIRST_VALUE, LAST_VALUE)

## Introduction

Les **fonctions de valeur** sont des window functions spécialisées qui permettent d'accéder à des **valeurs d'autres lignes** depuis la ligne courante, sans avoir besoin de jointures complexes ou de sous-requêtes.

Elles sont essentielles pour :
- **Calculer des variations** (différence avec le jour précédent, croissance)
- **Comparer** une valeur avec la meilleure/pire du groupe
- **Détecter des changements** (premier achat, dernier contact)
- **Analyser des séquences** temporelles ou ordonnées

PostgreSQL propose quatre fonctions principales :

1. **LAG()** : Accède à une ligne **précédente**
2. **LEAD()** : Accède à une ligne **suivante**
3. **FIRST_VALUE()** : Accède à la **première** valeur de la fenêtre
4. **LAST_VALUE()** : Accède à la **dernière** valeur de la fenêtre

## LAG() : Regarder en Arrière

### Principe

`LAG()` permet d'accéder à la valeur d'une colonne dans une ligne **précédente**, selon l'ordre défini par `ORDER BY`.

**Métaphore** : Imaginez que vous lisez une liste de températures jour après jour. `LAG()` vous permet de regarder "quelle était la température hier" tout en regardant celle d'aujourd'hui.

### Syntaxe

```sql
LAG(colonne [, offset [, valeur_par_defaut]]) OVER (
    [PARTITION BY groupe]
    ORDER BY ordre
)
```

**Paramètres** :
- `colonne` : La colonne dont on veut récupérer la valeur
- `offset` : Nombre de lignes en arrière (par défaut : 1)
- `valeur_par_defaut` : Valeur retournée s'il n'y a pas de ligne précédente (par défaut : NULL)

### Exemple de Base

Calculer la différence de température entre aujourd'hui et hier :

```sql
SELECT
    date,
    temperature,
    LAG(temperature) OVER (ORDER BY date) AS temp_hier,
    temperature - LAG(temperature) OVER (ORDER BY date) AS variation
FROM meteo
ORDER BY date;
```

**Données** :

```
| date       | temperature |
|------------|-------------|
| 2025-01-01 | 5           |
| 2025-01-02 | 8           |
| 2025-01-03 | 7           |
| 2025-01-04 | 12          |
```

**Résultat** :

```
| date       | temperature | temp_hier | variation |
|------------|-------------|-----------|-----------|
| 2025-01-01 | 5           | NULL      | NULL      | ← Pas de jour précédent
| 2025-01-02 | 8           | 5         | 3         | ← 8 - 5 = +3
| 2025-01-03 | 7           | 8         | -1        | ← 7 - 8 = -1
| 2025-01-04 | 12          | 7         | 5         | ← 12 - 7 = +5
```

**Explication** :
- Ligne 1 : Pas de ligne précédente → `LAG()` retourne NULL
- Ligne 2 : Ligne précédente = 5 → `LAG()` retourne 5, variation = 8-5 = 3
- Ligne 3 : Ligne précédente = 8 → `LAG()` retourne 8, variation = 7-8 = -1
- Ligne 4 : Ligne précédente = 7 → `LAG()` retourne 7, variation = 12-7 = 5

### Spécifier un Offset

Vous pouvez regarder **plusieurs lignes en arrière** :

```sql
SELECT
    date,
    ventes,
    LAG(ventes, 1) OVER (ORDER BY date) AS ventes_hier,
    LAG(ventes, 7) OVER (ORDER BY date) AS ventes_semaine_derniere,
    LAG(ventes, 30) OVER (ORDER BY date) AS ventes_mois_dernier
FROM ventes_quotidiennes
ORDER BY date;
```

- `LAG(ventes, 1)` : Hier (par défaut)
- `LAG(ventes, 7)` : Il y a 7 jours
- `LAG(ventes, 30)` : Il y a 30 jours

### Valeur par Défaut

Pour éviter les NULL, spécifiez une valeur par défaut :

```sql
SELECT
    date,
    ventes,
    LAG(ventes, 1, 0) OVER (ORDER BY date) AS ventes_hier
FROM ventes_quotidiennes;
```

Maintenant, s'il n'y a pas de ligne précédente, `LAG()` retournera **0** au lieu de NULL.

### Utilisation avec PARTITION BY

Le cas le plus courant : comparer avec la ligne précédente **dans chaque groupe** :

```sql
SELECT
    vendeur,
    mois,
    ventes,
    LAG(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY mois
    ) AS ventes_mois_precedent,
    ventes - LAG(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY mois
    ) AS croissance
FROM ventes_mensuelles
ORDER BY vendeur, mois;
```

**Données** :

```
| vendeur | mois    | ventes |
|---------|---------|--------|
| Alice   | 2025-01 | 1000   |
| Alice   | 2025-02 | 1200   |
| Alice   | 2025-03 | 1100   |
| Bob     | 2025-01 | 800    |
| Bob     | 2025-02 | 900    |
| Bob     | 2025-03 | 1000   |
```

**Résultat** :

```
| vendeur | mois    | ventes | ventes_mois_precedent | croissance |
|---------|---------|--------|----------------------|------------|
| Alice   | 2025-01 | 1000   | NULL                 | NULL       |
| Alice   | 2025-02 | 1200   | 1000                 | 200        |
| Alice   | 2025-03 | 1100   | 1200                 | -100       |
| Bob     | 2025-01 | 800    | NULL                 | NULL       | ← Recommence pour Bob
| Bob     | 2025-02 | 900    | 800                  | 100        |
| Bob     | 2025-03 | 1000   | 900                  | 100        |
```

**Point clé** : `LAG()` compare Alice avec Alice, Bob avec Bob, mais jamais Alice avec Bob.

### Cas d'Usage Typiques

#### 1. Croissance Jour après Jour

```sql
SELECT
    date,
    ca_journalier,
    ROUND(
        (ca_journalier - LAG(ca_journalier) OVER (ORDER BY date)) * 100.0
        / LAG(ca_journalier) OVER (ORDER BY date),
        2
    ) AS croissance_pct
FROM ventes
ORDER BY date;
```

#### 2. Détecter les Changements de Statut

```sql
SELECT
    client_id,
    date_statut,
    statut,
    LAG(statut) OVER (
        PARTITION BY client_id
        ORDER BY date_statut
    ) AS statut_precedent
FROM historique_clients
WHERE statut != LAG(statut) OVER (
    PARTITION BY client_id
    ORDER BY date_statut
);  -- Ne garde que les changements
```

#### 3. Temps entre Deux Événements

```sql
SELECT
    utilisateur_id,
    date_connexion,
    date_connexion - LAG(date_connexion) OVER (
        PARTITION BY utilisateur_id
        ORDER BY date_connexion
    ) AS jours_depuis_derniere_connexion
FROM connexions;
```

## LEAD() : Regarder en Avant

### Principe

`LEAD()` est l'inverse de `LAG()` : elle permet d'accéder à la valeur d'une colonne dans une ligne **suivante**.

**Métaphore** : Comme si vous pouviez "voir dans le futur" et connaître la prochaine valeur.

### Syntaxe

```sql
LEAD(colonne [, offset [, valeur_par_defaut]]) OVER (
    [PARTITION BY groupe]
    ORDER BY ordre
)
```

**Identique à LAG**, mais regarde **en avant** au lieu d'en arrière.

### Exemple de Base

Savoir quand aura lieu la prochaine visite d'un client :

```sql
SELECT
    client_id,
    date_visite,
    LEAD(date_visite) OVER (
        PARTITION BY client_id
        ORDER BY date_visite
    ) AS prochaine_visite,
    LEAD(date_visite) OVER (
        PARTITION BY client_id
        ORDER BY date_visite
    ) - date_visite AS jours_avant_prochaine_visite
FROM visites
ORDER BY client_id, date_visite;
```

**Données** :

```
| client_id | date_visite |
|-----------|-------------|
| 1         | 2025-01-01  |
| 1         | 2025-01-15  |
| 1         | 2025-02-10  |
| 2         | 2025-01-05  |
| 2         | 2025-02-20  |
```

**Résultat** :

```
| client_id | date_visite | prochaine_visite | jours_avant_prochaine_visite |
|-----------|-------------|------------------|------------------------------|
| 1         | 2025-01-01  | 2025-01-15       | 14                           |
| 1         | 2025-01-15  | 2025-02-10       | 26                           |
| 1         | 2025-02-10  | NULL             | NULL                         | ← Dernière visite
| 2         | 2025-01-05  | 2025-02-20       | 46                           |
| 2         | 2025-02-20  | NULL             | NULL                         | ← Dernière visite
```

### Différence LAG vs LEAD

**Visualisation** :

```
Données ordonnées : [A] [B] [C] [D]

Pour la ligne C :
- LAG(1)  regarde B (1 ligne avant)
- LEAD(1) regarde D (1 ligne après)

Pour la ligne A :
- LAG(1)  retourne NULL (pas de ligne avant)
- LEAD(1) retourne B

Pour la ligne D :
- LAG(1)  retourne C
- LEAD(1) retourne NULL (pas de ligne après)
```

### Cas d'Usage Typiques

#### 1. Durée d'une Session

```sql
SELECT
    session_id,
    utilisateur_id,
    heure_debut,
    LEAD(heure_debut) OVER (
        PARTITION BY utilisateur_id
        ORDER BY heure_debut
    ) AS heure_session_suivante,
    LEAD(heure_debut) OVER (
        PARTITION BY utilisateur_id
        ORDER BY heure_debut
    ) - heure_debut AS duree_inactivite
FROM sessions;
```

#### 2. Prévision de Rupture de Stock

```sql
SELECT
    date,
    stock,
    ventes_quotidiennes,
    LEAD(stock) OVER (ORDER BY date) AS stock_demain,
    CASE
        WHEN LEAD(stock) OVER (ORDER BY date) < seuil_alerte
        THEN 'ALERTE'
        ELSE 'OK'
    END AS alerte_stock
FROM inventaire;
```

#### 3. Détecter les Séquences

```sql
-- Trouver les séquences de 3 jours consécutifs de croissance
SELECT
    date,
    ventes,
    CASE
        WHEN ventes > LAG(ventes) OVER (ORDER BY date)
         AND LEAD(ventes) OVER (ORDER BY date) > ventes
        THEN 'Tendance haussière'
        ELSE NULL
    END AS tendance
FROM ventes_quotidiennes;
```

## FIRST_VALUE() : La Première Valeur

### Principe

`FIRST_VALUE()` retourne la **première valeur** d'une colonne dans la fenêtre définie.

**Métaphore** : Comme regarder le "point de départ" ou la "référence initiale" de votre analyse.

### Syntaxe

```sql
FIRST_VALUE(colonne) OVER (
    [PARTITION BY groupe]
    ORDER BY ordre
    [frame_clause]
)
```

### Exemple de Base

Comparer chaque valeur au point de départ :

```sql
SELECT
    date,
    cours_action,
    FIRST_VALUE(cours_action) OVER (ORDER BY date) AS cours_initial,
    cours_action - FIRST_VALUE(cours_action) OVER (ORDER BY date) AS evolution,
    ROUND(
        (cours_action - FIRST_VALUE(cours_action) OVER (ORDER BY date)) * 100.0
        / FIRST_VALUE(cours_action) OVER (ORDER BY date),
        2
    ) AS evolution_pct
FROM cours_bourse
ORDER BY date;
```

**Données** :

```
| date       | cours_action |
|------------|--------------|
| 2025-01-01 | 100          |
| 2025-01-02 | 105          |
| 2025-01-03 | 103          |
| 2025-01-04 | 110          |
```

**Résultat** :

```
| date       | cours_action | cours_initial | evolution | evolution_pct |
|------------|--------------|---------------|-----------|---------------|
| 2025-01-01 | 100          | 100           | 0         | 0.00          |
| 2025-01-02 | 105          | 100           | 5         | 5.00          |
| 2025-01-03 | 103          | 100           | 3         | 3.00          |
| 2025-01-04 | 110          | 100           | 10        | 10.00         |
```

**Point clé** : `cours_initial` reste constant (100) pour toutes les lignes car c'est la première valeur.

### Utilisation avec PARTITION BY

Comparer à la première valeur **de chaque groupe** :

```sql
SELECT
    vendeur,
    mois,
    ventes,
    FIRST_VALUE(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY mois
    ) AS ventes_premier_mois,
    ventes - FIRST_VALUE(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY mois
    ) AS progression
FROM ventes_mensuelles
ORDER BY vendeur, mois;
```

**Données** :

```
| vendeur | mois    | ventes |
|---------|---------|--------|
| Alice   | 2025-01 | 1000   |
| Alice   | 2025-02 | 1200   |
| Alice   | 2025-03 | 1100   |
| Bob     | 2025-01 | 800    |
| Bob     | 2025-02 | 900    |
| Bob     | 2025-03 | 1000   |
```

**Résultat** :

```
| vendeur | mois    | ventes | ventes_premier_mois | progression |
|---------|---------|--------|---------------------|-------------|
| Alice   | 2025-01 | 1000   | 1000                | 0           |
| Alice   | 2025-02 | 1200   | 1000                | 200         |
| Alice   | 2025-03 | 1100   | 1000                | 100         |
| Bob     | 2025-01 | 800    | 800                 | 0           |
| Bob     | 2025-02 | 900    | 800                 | 100         |
| Bob     | 2025-03 | 1000   | 800                 | 200         |
```

**Point clé** : Pour Alice, le premier mois est 1000. Pour Bob, c'est 800.

### Frame et FIRST_VALUE

Par défaut, `FIRST_VALUE()` utilise le frame :
```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

Cela signifie "du début de la partition jusqu'à la ligne courante", ce qui fait que `FIRST_VALUE()` retourne toujours la première valeur de la partition.

Vous pouvez modifier le frame pour obtenir la première valeur d'une **fenêtre glissante** :

```sql
SELECT
    date,
    temperature,
    FIRST_VALUE(temperature) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS temp_debut_semaine
FROM meteo;
```

Cela retourne la température du **début** de la fenêtre de 7 jours glissante.

### Cas d'Usage Typiques

#### 1. Progression depuis le Début de l'Année

```sql
SELECT
    date,
    ventes,
    FIRST_VALUE(ventes) OVER (
        PARTITION BY EXTRACT(YEAR FROM date)
        ORDER BY date
    ) AS ventes_1er_jour_annee,
    ventes - FIRST_VALUE(ventes) OVER (
        PARTITION BY EXTRACT(YEAR FROM date)
        ORDER BY date
    ) AS progression_annuelle
FROM ventes_quotidiennes;
```

#### 2. Comparer au Prix d'Ouverture

```sql
-- Bourse : comparer au cours d'ouverture de la journée
SELECT
    timestamp,
    cours,
    FIRST_VALUE(cours) OVER (
        PARTITION BY DATE(timestamp)
        ORDER BY timestamp
    ) AS cours_ouverture,
    cours - FIRST_VALUE(cours) OVER (
        PARTITION BY DATE(timestamp)
        ORDER BY timestamp
    ) AS variation_journee
FROM cours_temps_reel;
```

#### 3. Référence par Catégorie

```sql
-- Comparer chaque prix au prix du leader de la catégorie
SELECT
    categorie,
    produit,
    prix,
    FIRST_VALUE(produit) OVER (
        PARTITION BY categorie
        ORDER BY prix DESC
    ) AS produit_premium,
    FIRST_VALUE(prix) OVER (
        PARTITION BY categorie
        ORDER BY prix DESC
    ) AS prix_premium,
    ROUND((prix * 100.0) / FIRST_VALUE(prix) OVER (
        PARTITION BY categorie
        ORDER BY prix DESC
    ), 2) AS pourcentage_du_premium
FROM produits;
```

## LAST_VALUE() : La Dernière Valeur

### Principe

`LAST_VALUE()` retourne la **dernière valeur** d'une colonne dans la fenêtre définie.

**⚠️ ATTENTION** : `LAST_VALUE()` est **piégeuse** ! Son comportement par défaut surprend souvent les débutants.

### Syntaxe

```sql
LAST_VALUE(colonne) OVER (
    [PARTITION BY groupe]
    ORDER BY ordre
    [frame_clause]
)
```

### Le Piège de LAST_VALUE

#### Comportement par Défaut (Surprenant)

```sql
SELECT
    date,
    ventes,
    LAST_VALUE(ventes) OVER (ORDER BY date) AS derniere_vente
FROM ventes
ORDER BY date;
```

**Données** :

```
| date       | ventes |
|------------|--------|
| 2025-01-01 | 100    |
| 2025-01-02 | 200    |
| 2025-01-03 | 150    |
| 2025-01-04 | 300    |
```

**Résultat (surprenant !)** :

```
| date       | ventes | derniere_vente |
|------------|--------|----------------|
| 2025-01-01 | 100    | 100            | ← Pas 300 !
| 2025-01-02 | 200    | 200            | ← Pas 300 !
| 2025-01-03 | 150    | 150            | ← Pas 300 !
| 2025-01-04 | 300    | 300            | ← OK
```

**Pourquoi ?** Le frame par défaut est `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, ce qui signifie "du début jusqu'à la ligne courante". Donc `LAST_VALUE()` retourne la **dernière valeur de la ligne courante**, c'est-à-dire... la ligne courante elle-même !

#### Solution : Spécifier le Frame Correct

Pour obtenir la **vraie** dernière valeur de la partition, utilisez :

```sql
SELECT
    date,
    ventes,
    LAST_VALUE(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS derniere_vente
FROM ventes
ORDER BY date;
```

**Résultat (correct)** :

```
| date       | ventes | derniere_vente |
|------------|--------|----------------|
| 2025-01-01 | 100    | 300            | ← Correct !
| 2025-01-02 | 200    | 300            |
| 2025-01-03 | 150    | 300            |
| 2025-01-04 | 300    | 300            |
```

**Explication** : `UNBOUNDED FOLLOWING` force la fenêtre à inclure **toutes** les lignes jusqu'à la fin, donc `LAST_VALUE()` peut accéder à la vraie dernière valeur.

### Forme Recommandée

**Toujours** utiliser cette forme avec `LAST_VALUE()` :

```sql
LAST_VALUE(colonne) OVER (
    [PARTITION BY groupe]
    ORDER BY ordre
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

### Utilisation avec PARTITION BY

Obtenir la dernière valeur **de chaque groupe** :

```sql
SELECT
    vendeur,
    mois,
    ventes,
    LAST_VALUE(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY mois
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS ventes_dernier_mois
FROM ventes_mensuelles
ORDER BY vendeur, mois;
```

**Données** :

```
| vendeur | mois    | ventes |
|---------|---------|--------|
| Alice   | 2025-01 | 1000   |
| Alice   | 2025-02 | 1200   |
| Alice   | 2025-03 | 1100   |
| Bob     | 2025-01 | 800    |
| Bob     | 2025-02 | 900    |
| Bob     | 2025-03 | 1000   |
```

**Résultat** :

```
| vendeur | mois    | ventes | ventes_dernier_mois |
|---------|---------|--------|---------------------|
| Alice   | 2025-01 | 1000   | 1100                |
| Alice   | 2025-02 | 1200   | 1100                |
| Alice   | 2025-03 | 1100   | 1100                |
| Bob     | 2025-01 | 800    | 1000                |
| Bob     | 2025-02 | 900    | 1000                |
| Bob     | 2025-03 | 1000   | 1000                |
```

### Cas d'Usage Typiques

#### 1. Comparer au Dernier Connu

```sql
SELECT
    date,
    objectif,
    LAST_VALUE(objectif) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS objectif_final,
    ROUND(
        (objectif * 100.0) / LAST_VALUE(objectif) OVER (
            ORDER BY date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ),
        2
    ) AS pourcentage_objectif_final
FROM objectifs_annuels;
```

#### 2. Trouver le Prix Final

```sql
-- Quel est le prix final d'un produit dans sa catégorie ?
SELECT
    categorie,
    date_modification,
    prix,
    LAST_VALUE(prix) OVER (
        PARTITION BY categorie
        ORDER BY date_modification
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS prix_actuel
FROM historique_prix;
```

#### 3. Distance au Dernier Événement

```sql
SELECT
    client_id,
    date_achat,
    montant,
    LAST_VALUE(date_achat) OVER (
        PARTITION BY client_id
        ORDER BY date_achat
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS date_dernier_achat,
    CURRENT_DATE - LAST_VALUE(date_achat) OVER (
        PARTITION BY client_id
        ORDER BY date_achat
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS jours_depuis_dernier_achat
FROM achats;
```

## Combinaisons et Comparaisons

### Les Quatre Fonctions Ensemble

```sql
SELECT
    date,
    ventes,
    LAG(ventes) OVER (ORDER BY date) AS ventes_hier,
    LEAD(ventes) OVER (ORDER BY date) AS ventes_demain,
    FIRST_VALUE(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS premiere_vente,
    LAST_VALUE(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS derniere_vente
FROM ventes
ORDER BY date;
```

**Données** :

```
| date       | ventes |
|------------|--------|
| 2025-01-01 | 100    |
| 2025-01-02 | 200    |
| 2025-01-03 | 150    |
| 2025-01-04 | 300    |
```

**Résultat** :

```
| date       | ventes | ventes_hier | ventes_demain | premiere_vente | derniere_vente |
|------------|--------|-------------|---------------|----------------|----------------|
| 2025-01-01 | 100    | NULL        | 200           | 100            | 300            |
| 2025-01-02 | 200    | 100         | 150           | 100            | 300            |
| 2025-01-03 | 150    | 200         | 300           | 100            | 300            |
| 2025-01-04 | 300    | 150         | NULL          | 100            | 300            |
```

### Tableau Comparatif

| Fonction | Direction | Offset | Valeur Retournée | Frame par Défaut |
|----------|-----------|--------|------------------|------------------|
| **LAG()** | Arrière | Paramétrable | Ligne N précédente | N/A (direct access) |
| **LEAD()** | Avant | Paramétrable | Ligne N suivante | N/A (direct access) |
| **FIRST_VALUE()** | Début | N/A | Première de la fenêtre | UNBOUNDED PRECEDING → CURRENT ROW |
| **LAST_VALUE()** | Fin | N/A | Dernière de la fenêtre | UNBOUNDED PRECEDING → CURRENT ROW ⚠️ |

## Techniques Avancées

### 1. Calculer la Croissance Moyenne sur N Périodes

```sql
SELECT
    date,
    ventes,
    AVG(
        (ventes - LAG(ventes) OVER (ORDER BY date)) * 100.0
        / LAG(ventes) OVER (ORDER BY date)
    ) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS croissance_moyenne_7j
FROM ventes_quotidiennes;
```

### 2. Détecter les Pics et Creux

```sql
SELECT
    date,
    cours,
    CASE
        WHEN cours > LAG(cours) OVER (ORDER BY date)
         AND cours > LEAD(cours) OVER (ORDER BY date)
        THEN 'PIC'
        WHEN cours < LAG(cours) OVER (ORDER BY date)
         AND cours < LEAD(cours) OVER (ORDER BY date)
        THEN 'CREUX'
        ELSE 'NORMAL'
    END AS point_remarquable
FROM cours_bourse;
```

### 3. Amplitude de la Fenêtre

```sql
SELECT
    date,
    temperature,
    LAST_VALUE(temperature) OVER w - FIRST_VALUE(temperature) OVER w AS amplitude_semaine,
    MAX(temperature) OVER w AS max_semaine,
    MIN(temperature) OVER w AS min_semaine
FROM meteo
WINDOW w AS (
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
);
```

### 4. Combiner avec Agrégations

```sql
SELECT
    vendeur,
    trimestre,
    ventes,
    AVG(ventes) OVER (PARTITION BY vendeur) AS moyenne_vendeur,
    ventes - FIRST_VALUE(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY trimestre
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS progression_depuis_debut,
    LAST_VALUE(ventes) OVER (
        PARTITION BY vendeur
        ORDER BY trimestre
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) - ventes AS ecart_avec_fin
FROM ventes_trimestrielles;
```

### 5. Fenêtres Glissantes Personnalisées

```sql
-- Première et dernière valeur d'une fenêtre de 30 jours
SELECT
    date,
    ventes,
    FIRST_VALUE(ventes) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '30 days' PRECEDING AND CURRENT ROW
    ) AS debut_fenetre_30j,
    LAST_VALUE(ventes) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '30 days' PRECEDING AND CURRENT ROW
    ) AS fin_fenetre_30j
FROM ventes;
```

## Gestion des NULL

### NULL dans LAG/LEAD

Si la colonne contient des NULL, `LAG()` et `LEAD()` les retournent tel quel :

```sql
SELECT
    date,
    temperature,
    LAG(temperature) OVER (ORDER BY date) AS temp_hier
FROM meteo;
```

Si la température d'hier était NULL, `temp_hier` sera NULL.

### Utiliser COALESCE

Pour gérer les NULL, utilisez `COALESCE()` :

```sql
SELECT
    date,
    temperature,
    COALESCE(
        LAG(temperature) OVER (ORDER BY date),
        temperature
    ) AS temp_hier_ou_aujourdhui
FROM meteo;
```

Si `LAG()` retourne NULL (première ligne ou valeur manquante), utilise la température du jour.

### Valeur par Défaut dans LAG/LEAD

Alternative : spécifier directement la valeur par défaut :

```sql
LAG(temperature, 1, 0) OVER (ORDER BY date)  -- 0 si NULL
```

## Performances

### Optimisation avec Index

Ces fonctions bénéficient d'index sur les colonnes `ORDER BY` et `PARTITION BY` :

```sql
-- Pour optimiser :
LAG(ventes) OVER (PARTITION BY vendeur ORDER BY date)

-- Créez un index :
CREATE INDEX idx_ventes_vendeur_date ON ventes(vendeur, date);
```

### Réutiliser les Windows

Utilisez la clause `WINDOW` pour éviter la duplication :

```sql
SELECT
    date,
    ventes,
    LAG(ventes) OVER w AS ventes_hier,
    LEAD(ventes) OVER w AS ventes_demain,
    AVG(ventes) OVER w AS moyenne
FROM ventes
WINDOW w AS (ORDER BY date);
```

### Coût Relatif

Ces fonctions sont généralement **très performantes** :
1. `LAG()` / `LEAD()` : Accès direct, très rapide
2. `FIRST_VALUE()` : Simple, rapide
3. `LAST_VALUE()` avec `UNBOUNDED FOLLOWING` : Plus coûteuse (doit regarder toute la partition)

## Erreurs Courantes

### 1. Oublier ORDER BY

```sql
-- ❌ ERREUR : Pas d'ordre défini
LAG(ventes) OVER (PARTITION BY vendeur)

-- ✅ CORRECT
LAG(ventes) OVER (PARTITION BY vendeur ORDER BY date)
```

Sans `ORDER BY`, les concepts de "précédent" et "suivant" n'ont pas de sens.

### 2. Utiliser LAST_VALUE sans Frame Correct

```sql
-- ❌ INCORRECT : Retourne la ligne courante
LAST_VALUE(ventes) OVER (ORDER BY date)

-- ✅ CORRECT
LAST_VALUE(ventes) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

### 3. Confondre LAG et LEAD

```sql
-- Pour calculer la croissance :
ventes - LAG(ventes) OVER (ORDER BY date)  -- ✅ Croissance par rapport à hier

-- Pas :
ventes - LEAD(ventes) OVER (ORDER BY date)  -- ❌ Différence avec demain
```

### 4. Filtrer Directement sur Window Functions

```sql
-- ❌ ERREUR : Window function dans WHERE
SELECT * FROM ventes
WHERE LAG(ventes) OVER (ORDER BY date) < ventes;

-- ✅ CORRECT : Sous-requête
SELECT * FROM (
    SELECT *, LAG(ventes) OVER (ORDER BY date) AS ventes_hier
    FROM ventes
) AS sub
WHERE ventes_hier < ventes;
```

### 5. Ne pas Gérer les NULL

```sql
-- ❌ Peut causer des NULL inattendus
ventes - LAG(ventes) OVER (ORDER BY date)

-- ✅ Gérer explicitement
ventes - COALESCE(LAG(ventes) OVER (ORDER BY date), ventes)
```

## Cas d'Usage Pratiques Complets

### 1. Tableau de Bord de Croissance

```sql
SELECT
    date,
    ventes,
    LAG(ventes, 1) OVER (ORDER BY date) AS hier,
    LAG(ventes, 7) OVER (ORDER BY date) AS semaine_derniere,
    ventes - LAG(ventes, 1) OVER (ORDER BY date) AS var_jour,
    ventes - LAG(ventes, 7) OVER (ORDER BY date) AS var_semaine,
    ROUND(
        (ventes - LAG(ventes, 1) OVER (ORDER BY date)) * 100.0
        / NULLIF(LAG(ventes, 1) OVER (ORDER BY date), 0),
        2
    ) AS pct_jour,
    FIRST_VALUE(ventes) OVER (
        PARTITION BY EXTRACT(YEAR FROM date)
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS debut_annee
FROM ventes_quotidiennes
ORDER BY date;
```

### 2. Analyse de Rétention Client

```sql
WITH achats_numerotes AS (
    SELECT
        client_id,
        date_achat,
        montant,
        ROW_NUMBER() OVER (PARTITION BY client_id ORDER BY date_achat) AS numero_achat,
        LAG(date_achat) OVER (PARTITION BY client_id ORDER BY date_achat) AS achat_precedent
    FROM achats
)
SELECT
    client_id,
    date_achat,
    numero_achat,
    date_achat - achat_precedent AS jours_depuis_precedent,
    CASE
        WHEN numero_achat = 1 THEN 'Premier achat'
        WHEN date_achat - achat_precedent <= 30 THEN 'Client fidèle'
        WHEN date_achat - achat_precedent <= 90 THEN 'Client régulier'
        ELSE 'Client réactivé'
    END AS type_achat
FROM achats_numerotes
ORDER BY client_id, date_achat;
```

### 3. Détection d'Anomalies

```sql
SELECT
    date,
    ventes,
    AVG(ventes) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
    ) AS moyenne_7j_precedents,
    CASE
        WHEN ventes > 2 * AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
        ) THEN 'PIC ANORMAL'
        WHEN ventes < 0.5 * AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING
        ) THEN 'CHUTE ANORMALE'
        ELSE 'NORMAL'
    END AS alerte
FROM ventes_quotidiennes;
```

### 4. Évolution de Position dans un Classement

```sql
WITH classement AS (
    SELECT
        semaine,
        joueur,
        score,
        RANK() OVER (PARTITION BY semaine ORDER BY score DESC) AS rang
    FROM scores
)
SELECT
    semaine,
    joueur,
    score,
    rang,
    LAG(rang) OVER (PARTITION BY joueur ORDER BY semaine) AS rang_precedent,
    LAG(rang) OVER (PARTITION BY joueur ORDER BY semaine) - rang AS progression,
    FIRST_VALUE(rang) OVER (
        PARTITION BY joueur
        ORDER BY semaine
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS meilleur_rang
FROM classement
ORDER BY semaine, rang;
```

## Points Clés à Retenir

- ✅ **LAG()** accède à une ligne **précédente** (N lignes en arrière)
- ✅ **LEAD()** accède à une ligne **suivante** (N lignes en avant)
- ✅ **FIRST_VALUE()** accède à la **première** valeur de la fenêtre
- ✅ **LAST_VALUE()** accède à la **dernière** valeur (⚠️ spécifier `UNBOUNDED FOLLOWING`)
- ✅ Toutes nécessitent **ORDER BY** pour définir l'ordre
- ✅ Utilisez des **valeurs par défaut** ou `COALESCE()` pour gérer les NULL
- ✅ Combinez avec **PARTITION BY** pour des analyses par groupe
- ✅ Créez des **index** sur les colonnes ORDER BY pour la performance

## Tableau de Choix Rapide

| Besoin | Fonction à Utiliser |
|--------|---------------------|
| Calculer la croissance jour/jour | `LAG()` |
| Détecter la prochaine valeur | `LEAD()` |
| Comparer au point de départ | `FIRST_VALUE()` |
| Comparer au point d'arrivée | `LAST_VALUE()` |
| Calculer la différence avec hier | `LAG()` |
| Prévoir le prochain événement | `LEAD()` |
| Analyser la progression depuis le début | `FIRST_VALUE()` |
| Mesurer l'écart avec l'objectif final | `LAST_VALUE()` |

## Résumé Visuel

```sql
-- LAG : Regarder en arrière
LAG(colonne, N, defaut) OVER (ORDER BY date)

-- LEAD : Regarder en avant
LEAD(colonne, N, defaut) OVER (ORDER BY date)

-- FIRST_VALUE : Première de la fenêtre
FIRST_VALUE(colonne) OVER (
    ORDER BY date
    [ROWS BETWEEN ... AND ...]
)

-- LAST_VALUE : Dernière de la fenêtre (ATTENTION AU FRAME !)
LAST_VALUE(colonne) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

**Représentation visuelle** :
```
Partition ordonnée : [A] [B] [C] [D] [E]

Pour la ligne C :
- LAG(1)        → B (ligne précédente)
- LEAD(1)       → D (ligne suivante)
- FIRST_VALUE() → A (première de la partition)
- LAST_VALUE()  → E (dernière de la partition, si frame correct)
```

---


⏭️ [Fonctions d'agrégation en fenêtrage](/10-fonctions-de-fenetrage/06-fonctions-agregation-fenetrage.md)
