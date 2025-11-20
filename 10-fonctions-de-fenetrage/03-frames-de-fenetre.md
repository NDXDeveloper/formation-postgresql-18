🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.3. Frames de fenêtre : ROWS vs RANGE vs GROUPS

## Introduction

Jusqu'à présent, nous avons vu comment `PARTITION BY` et `ORDER BY` définissent le contexte de nos window functions. Mais il existe un troisième élément encore plus précis pour contrôler **exactement quelles lignes** sont incluses dans chaque calcul : les **frames de fenêtre**.

Les frames permettent de définir avec précision :
- Combien de lignes avant la ligne courante inclure
- Combien de lignes après la ligne courante inclure
- Comment gérer les valeurs égales (ties)

C'est l'outil qui vous permet de créer des **moyennes mobiles**, des **fenêtres glissantes personnalisées**, et bien d'autres analyses sophistiquées.

## Le Concept de Frame (Cadre)

### Qu'est-ce qu'un Frame ?

Un **frame** est un **sous-ensemble de la partition** utilisé pour le calcul d'une ligne donnée.

**Analogie** : Imaginez que vous regardez une vidéo (la partition). Le frame est comme une "fenêtre temporelle" que vous pouvez déplacer :
- Vous pouvez regarder les 3 dernières secondes
- Vous pouvez regarder de la seconde 0 jusqu'à maintenant
- Vous pouvez regarder 2 secondes avant et 2 secondes après la seconde actuelle

### Comportement par Défaut

Quand vous utilisez `ORDER BY` sans spécifier de frame explicite :

```sql
SUM(montant) OVER (PARTITION BY vendeur ORDER BY date_vente)
```

PostgreSQL utilise un frame **par défaut** :
- **Du début de la partition jusqu'à la ligne courante** (incluse)
- C'est équivalent à : `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

C'est ce qui crée l'effet de **cumul** que nous avons vu précédemment.

### Sans ORDER BY

Sans `ORDER BY`, le frame inclut **toute la partition** :

```sql
SUM(montant) OVER (PARTITION BY vendeur)
```

Équivalent à : `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`

## Les Trois Types de Frames

PostgreSQL propose trois façons de définir un frame :

1. **ROWS** : Compte les lignes physiques
2. **RANGE** : Regroupe les lignes ayant la même valeur (dans ORDER BY)
3. **GROUPS** : Travaille par groupes de valeurs égales (PostgreSQL 11+)

### Vue d'Ensemble

| Type | Base de calcul | Gestion des égalités | Usage typique |
|------|----------------|---------------------|---------------|
| **ROWS** | Nombre de lignes | Strictement physique | Moyennes mobiles, N lignes avant/après |
| **RANGE** | Plage de valeurs | Groupe les égalités | Cumuls, tous les montants égaux ensemble |
| **GROUPS** | Groupes d'égalités | Par groupe complet | Analyse par paliers |

Voyons chacun en détail.

## ROWS : Comptage Physique de Lignes

### Principe

`ROWS` compte les lignes **physiquement**, une par une, indépendamment de leurs valeurs.

### Syntaxe de Base

```sql
fonction OVER (
    ORDER BY colonne
    ROWS BETWEEN debut AND fin
)
```

### Les Bornes Possibles

| Borne | Signification |
|-------|---------------|
| `UNBOUNDED PRECEDING` | Début de la partition |
| `N PRECEDING` | N lignes avant la ligne courante |
| `CURRENT ROW` | La ligne courante |
| `N FOLLOWING` | N lignes après la ligne courante |
| `UNBOUNDED FOLLOWING` | Fin de la partition |

### Exemple 1 : Les 3 Dernières Lignes

```sql
SELECT
    date_vente,
    montant,
    AVG(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moyenne_mobile_3j
FROM ventes
ORDER BY date_vente;
```

**Données** :

```
| date_vente | montant |
|------------|---------|
| 2025-01-01 | 100     |
| 2025-01-02 | 150     |
| 2025-01-03 | 200     |
| 2025-01-04 | 120     |
| 2025-01-05 | 180     |
```

**Résultat** :

```
| date_vente | montant | moyenne_mobile_3j |
|------------|---------|-------------------|
| 2025-01-01 | 100     | 100.00            | ← 1 ligne : [100]
| 2025-01-02 | 150     | 125.00            | ← 2 lignes : [100, 150]
| 2025-01-03 | 200     | 150.00            | ← 3 lignes : [100, 150, 200]
| 2025-01-04 | 120     | 156.67            | ← 3 lignes : [150, 200, 120]
| 2025-01-05 | 180     | 166.67            | ← 3 lignes : [200, 120, 180]
```

**Explication** :
- Ligne 1 : Seulement 1 ligne disponible avant → moyenne de 100
- Ligne 2 : 2 lignes disponibles → moyenne de (100+150)/2
- Ligne 3 : 3 lignes disponibles → moyenne de (100+150+200)/3
- Ligne 4 : 3 lignes (fenêtre glisse) → moyenne de (150+200+120)/3
- Ligne 5 : 3 lignes (fenêtre glisse) → moyenne de (200+120+180)/3

### Exemple 2 : Fenêtre Centrée

```sql
SELECT
    jour,
    temperature,
    AVG(temperature) OVER (
        ORDER BY jour
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS temp_lissee
FROM meteo
ORDER BY jour;
```

Cette fenêtre inclut :
- 1 ligne avant
- La ligne courante
- 1 ligne après

**Données** :

```
| jour       | temperature |
|------------|-------------|
| 2025-01-01 | 5           |
| 2025-01-02 | 8           |
| 2025-01-03 | 12          |
| 2025-01-04 | 10          |
```

**Résultat** :

```
| jour       | temperature | temp_lissee |
|------------|-------------|-------------|
| 2025-01-01 | 5           | 6.5         | ← (5+8)/2 (pas de ligne avant)
| 2025-01-02 | 8           | 8.33        | ← (5+8+12)/3
| 2025-01-03 | 12          | 10.00       | ← (8+12+10)/3
| 2025-01-04 | 10          | 11.00       | ← (12+10)/2 (pas de ligne après)
```

### Exemple 3 : Du Début à la Ligne Courante (Cumul)

```sql
SELECT
    date_vente,
    montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumul
FROM ventes
ORDER BY date_vente;
```

C'est le **cumul classique** : chaque ligne additionne toutes les lignes précédentes plus elle-même.

### Forme Raccourcie

Certaines syntaxes sont simplifiables :

```sql
-- Forme complète
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Forme raccourcie (équivalent)
ROWS UNBOUNDED PRECEDING
```

```sql
-- Forme complète
ROWS BETWEEN 3 PRECEDING AND CURRENT ROW

-- Forme raccourcie
ROWS 3 PRECEDING
```

## RANGE : Plage de Valeurs

### Principe

`RANGE` groupe automatiquement les lignes qui ont la **même valeur** dans la colonne `ORDER BY`.

### Différence Clé avec ROWS

**ROWS** : Compte physiquement les lignes, même si elles ont la même valeur.

**RANGE** : Traite toutes les lignes ayant la même valeur comme un **groupe indivisible**.

### Exemple Démonstratif

Imaginez ces données :

```
| id | date_vente | montant |
|----|------------|---------|
| 1  | 2025-01-01 | 100     |
| 2  | 2025-01-01 | 150     | ← Même date que ligne 1
| 3  | 2025-01-02 | 200     |
| 4  | 2025-01-03 | 120     |
```

#### Avec ROWS

```sql
SELECT
    id, date_vente, montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumul_rows
FROM ventes
ORDER BY date_vente, id;
```

**Résultat** :

```
| id | date_vente | montant | cumul_rows |
|----|------------|---------|------------|
| 1  | 2025-01-01 | 100     | 100        | ← Juste ligne 1
| 2  | 2025-01-01 | 150     | 250        | ← Lignes 1+2
| 3  | 2025-01-02 | 200     | 450        | ← Lignes 1+2+3
| 4  | 2025-01-03 | 120     | 570        | ← Toutes les lignes
```

Les lignes 1 et 2 ont des cumuls **différents** même si elles ont la même date.

#### Avec RANGE

```sql
SELECT
    id, date_vente, montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumul_range
FROM ventes
ORDER BY date_vente, id;
```

**Résultat** :

```
| id | date_vente | montant | cumul_range |
|----|------------|---------|-------------|
| 1  | 2025-01-01 | 100     | 250         | ← Toutes les lignes du 01/01
| 2  | 2025-01-01 | 150     | 250         | ← Toutes les lignes du 01/01
| 3  | 2025-01-02 | 200     | 450         | ← Jusqu'au 02/01
| 4  | 2025-01-03 | 120     | 570         | ← Jusqu'au 03/01
```

Les lignes 1 et 2 ont le **même cumul** (250) car elles ont la même date. `RANGE` inclut **toutes** les lignes ayant la date 01/01 dans le frame.

### Quand Utiliser RANGE

`RANGE` est utile quand vous voulez que les **valeurs égales soient traitées ensemble** :

- Cumuls par date (toutes les transactions du même jour comptent ensemble)
- Classements sans "sauts" en cas d'égalité
- Agrégations sur des périodes (tous les montants d'une même période)

### Limites de RANGE

1. **Colonne ORDER BY unique** : `RANGE` ne fonctionne bien qu'avec une seule colonne dans `ORDER BY`
2. **Type de données** : La colonne doit être numérique ou temporelle pour les offsets (ex: `RANGE 7 PRECEDING` pour 7 jours)

### RANGE avec Offset

Vous pouvez spécifier un **offset en valeur** (pas en nombre de lignes) :

```sql
SELECT
    date_vente,
    montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) AS total_7_derniers_jours
FROM ventes
ORDER BY date_vente;
```

Cela inclut **toutes les lignes dont la date est dans les 7 jours précédents** (peu importe le nombre de lignes).

**Données** :

```
| date_vente | montant |
|------------|---------|
| 2025-01-01 | 100     |
| 2025-01-05 | 150     |
| 2025-01-08 | 200     |
| 2025-01-10 | 120     |
```

**Résultat** :

```
| date_vente | montant | total_7_derniers_jours |
|------------|---------|------------------------|
| 2025-01-01 | 100     | 100                    | ← Seulement 01/01
| 2025-01-05 | 150     | 250                    | ← 01/01 + 05/01 (dans 7j)
| 2025-01-08 | 200     | 450                    | ← 01/01 + 05/01 + 08/01
| 2025-01-10 | 120     | 470                    | ← 05/01 + 08/01 + 10/01 (01/01 trop ancien)
```

## GROUPS : Par Groupes d'Égalités

### Principe

`GROUPS` est un hybride entre `ROWS` et `RANGE` :
- Il compte les **groupes** de valeurs égales (comme RANGE)
- Mais permet de spécifier un **nombre de groupes** (comme ROWS compte les lignes)

### Exemple

```sql
SELECT
    date_vente,
    montant,
    SUM(montant) OVER (
        ORDER BY date_vente
        GROUPS BETWEEN 1 PRECEDING AND CURRENT ROW
    ) AS total_2_groupes
FROM ventes
ORDER BY date_vente;
```

**Données** :

```
| date_vente | montant |
|------------|---------|
| 2025-01-01 | 100     | ← Groupe 1
| 2025-01-01 | 150     | ← Groupe 1 (même date)
| 2025-01-02 | 200     | ← Groupe 2
| 2025-01-03 | 120     | ← Groupe 3
| 2025-01-03 | 180     | ← Groupe 3 (même date)
```

**Résultat** :

```
| date_vente | montant | total_2_groupes |
|------------|---------|-----------------|
| 2025-01-01 | 100     | 250             | ← Groupe 1 (100+150)
| 2025-01-01 | 150     | 250             | ← Groupe 1
| 2025-01-02 | 200     | 450             | ← Groupes 1+2 (250+200)
| 2025-01-03 | 120     | 500             | ← Groupes 2+3 (200+300)
| 2025-01-03 | 180     | 500             | ← Groupes 2+3
```

**Explication** :
- `1 PRECEDING` signifie "1 groupe avant" + le groupe courant
- Toutes les lignes d'un même groupe ont la même valeur calculée

### Quand Utiliser GROUPS

`GROUPS` est utile pour des analyses par **paliers** ou **niveaux** :
- Comparer le groupe actuel avec les N groupes précédents
- Analyser des catégories (prix bas, moyen, élevé)
- Travailler avec des données discrètes par nature

## Syntaxe Complète des Frames

### Structure Générale

```sql
fonction OVER (
    [PARTITION BY ...]
    ORDER BY colonne
    {ROWS | RANGE | GROUPS} BETWEEN debut AND fin
)
```

### Bornes Disponibles

```sql
UNBOUNDED PRECEDING    -- Début de la partition
n PRECEDING            -- N unités avant (lignes/valeurs/groupes)
CURRENT ROW            -- Ligne courante
n FOLLOWING            -- N unités après
UNBOUNDED FOLLOWING    -- Fin de la partition
```

### Exemples de Frames Courants

#### 1. Moyenne Mobile sur 7 Lignes

```sql
AVG(valeur) OVER (
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)
```

→ Moyenne des 7 dernières lignes (6 avant + courante)

#### 2. Total Cumulé

```sql
SUM(montant) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

→ Cumul depuis le début

#### 3. Fenêtre Centrée 5 Lignes

```sql
AVG(temperature) OVER (
    ORDER BY date
    ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
)
```

→ Moyenne de 5 lignes : 2 avant + courante + 2 après

#### 4. Total sur 30 Jours Glissants

```sql
SUM(montant) OVER (
    ORDER BY date
    RANGE BETWEEN INTERVAL '30 days' PRECEDING AND CURRENT ROW
)
```

→ Toutes les transactions des 30 derniers jours

#### 5. Toute la Partition

```sql
MAX(montant) OVER (
    PARTITION BY vendeur
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

→ Maximum sur toute la partition (sans ORDER BY, c'est le comportement par défaut)

## Comportements par Défaut

### Avec ORDER BY, Sans Frame Explicite

```sql
SUM(montant) OVER (ORDER BY date)
```

**Équivaut à** :

```sql
SUM(montant) OVER (
    ORDER BY date
    RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

→ Cumul en utilisant RANGE (groupe les égalités)

### Sans ORDER BY

```sql
SUM(montant) OVER (PARTITION BY vendeur)
```

**Équivaut à** :

```sql
SUM(montant) OVER (
    PARTITION BY vendeur
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

→ Total de toute la partition

## Cas d'Usage Pratiques

### 1. Moyenne Mobile (Moving Average)

**Objectif** : Lisser les variations journalières en calculant une moyenne sur 7 jours.

```sql
SELECT
    date,
    nb_visites,
    AVG(nb_visites) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moyenne_mobile_7j
FROM analytics
ORDER BY date;
```

**Usage** : Détecter les tendances en éliminant le bruit.

### 2. Différence avec la Ligne Précédente

**Objectif** : Calculer la croissance jour après jour.

```sql
SELECT
    date,
    ca_journalier,
    ca_journalier - LAG(ca_journalier) OVER (ORDER BY date) AS croissance
FROM ventes
ORDER BY date;
```

Note : `LAG()` est une fonction spécifique aux window functions (vue au chapitre 10.5).

### 3. Cumul depuis le Début de l'Année

**Objectif** : Voir le chiffre d'affaires accumulé année par année.

```sql
SELECT
    date,
    montant,
    SUM(montant) OVER (
        PARTITION BY EXTRACT(YEAR FROM date)
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumul_annuel
FROM ventes
ORDER BY date;
```

### 4. Somme Glissante sur 3 Mois

**Objectif** : Total des ventes sur les 3 derniers mois (90 jours).

```sql
SELECT
    date,
    montant,
    SUM(montant) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '90 days' PRECEDING AND CURRENT ROW
    ) AS total_3_mois
FROM ventes
ORDER BY date;
```

### 5. Comparaison avec le Meilleur du Mois

**Objectif** : Comparer chaque vente au maximum du mois.

```sql
SELECT
    date,
    montant,
    MAX(montant) OVER (
        PARTITION BY DATE_TRUNC('month', date)
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS max_mois
FROM ventes
ORDER BY date;
```

## Tableau Comparatif Complet

| Aspect | ROWS | RANGE | GROUPS |
|--------|------|-------|--------|
| **Unité de mesure** | Lignes physiques | Plage de valeurs | Groupes de valeurs égales |
| **Gestion des égalités** | Aucune | Groupe toutes les égalités | Compte les groupes |
| **N PRECEDING** | N lignes avant | N unités de valeur avant | N groupes avant |
| **Offset numérique** | Toujours possible | Seulement si type compatible | Toujours possible |
| **Offset temporel** | Non | Oui (INTERVAL) | Non |
| **Déterminisme** | Toujours | Peut dépendre de l'ordre physique | Peut dépendre de l'ordre |
| **Usage typique** | Moyennes mobiles | Cumuls, fenêtres temporelles | Analyses par paliers |

## Erreurs Courantes

### 1. Oublier ORDER BY avec un Frame

```sql
-- ❌ ERREUR : Frame sans ORDER BY
SELECT montant,
       SUM(montant) OVER (
           ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
       )
FROM ventes;

-- ✅ CORRECT : ORDER BY est obligatoire
SELECT montant,
       SUM(montant) OVER (
           ORDER BY date
           ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
       )
FROM ventes;
```

**Pourquoi ?** Sans `ORDER BY`, le concept de "précédent" ou "suivant" n'a pas de sens.

### 2. Confondre ROWS et RANGE

```sql
-- Si vous voulez exactement 7 lignes
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- Pas RANGE (qui groupera les égalités)
RANGE BETWEEN 6 PRECEDING AND CURRENT ROW
```

### 3. Utiliser un Offset Temporel avec ROWS

```sql
-- ❌ ERREUR : INTERVAL avec ROWS
ROWS BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW

-- ✅ CORRECT : INTERVAL avec RANGE
RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
```

### 4. Bornes Inversées

```sql
-- ❌ ERREUR : La borne de début est après la fin
ROWS BETWEEN CURRENT ROW AND 3 PRECEDING

-- ✅ CORRECT
ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
```

### 5. Omettre BETWEEN avec Deux Bornes

```sql
-- ❌ ERREUR : Syntaxe incomplète
ROWS 3 PRECEDING AND CURRENT ROW

-- ✅ CORRECT
ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
```

## Performance et Bonnes Pratiques

### 1. Choisir le Bon Type de Frame

- **ROWS** : Plus performant car simple à calculer
- **RANGE** : Peut être plus coûteux, surtout avec des offsets temporels
- **GROUPS** : Coût intermédiaire

### 2. Utiliser des Index

Pour optimiser les frames avec `ORDER BY`, créez des index sur les colonnes de tri :

```sql
CREATE INDEX idx_ventes_date ON ventes(date_vente);
```

### 3. Limiter la Taille des Frames

Des frames trop larges (ex: `1000 PRECEDING`) peuvent être coûteux. Si possible :
- Utilisez des frames raisonnables (10-100 lignes)
- Filtrez les données en amont avec `WHERE`
- Considérez une agrégation pré-calculée pour des fenêtres très larges

### 4. Réutiliser les Mêmes Frames

PostgreSQL optimise les requêtes avec plusieurs window functions utilisant le même frame :

```sql
SELECT
    date,
    montant,
    SUM(montant) OVER w AS total,
    AVG(montant) OVER w AS moyenne,
    COUNT(*) OVER w AS nb
FROM ventes
WINDOW w AS (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW);
```

La clause `WINDOW` définit un frame réutilisable, évitant la duplication de code.

## Points Clés à Retenir

- ✅ **ROWS** compte les lignes physiquement, une par une
- ✅ **RANGE** groupe les valeurs égales et permet des offsets temporels
- ✅ **GROUPS** compte les groupes de valeurs égales
- ✅ Sans frame explicite + ORDER BY → Par défaut : `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- ✅ `N PRECEDING` / `N FOLLOWING` pour définir la taille de la fenêtre
- ✅ `UNBOUNDED` pour inclure tout depuis le début ou jusqu'à la fin
- ✅ Les frames nécessitent `ORDER BY` pour définir "avant" et "après"
- ✅ Utilisez la clause `WINDOW` pour définir des frames réutilisables

## Résumé Visuel

```sql
SELECT
    colonne,
    fonction(colonne2) OVER (
        [PARTITION BY groupe]
        ORDER BY ordre
        {ROWS | RANGE | GROUPS} BETWEEN debut AND fin
    ) AS resultat
FROM table;

-- Bornes possibles :
-- debut/fin = UNBOUNDED PRECEDING
--           | N PRECEDING
--           | CURRENT ROW
--           | N FOLLOWING
--           | UNBOUNDED FOLLOWING
```

**Formes raccourcies** :
```sql
ROWS 3 PRECEDING              → ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
ROWS UNBOUNDED PRECEDING      → ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

---


⏭️ [Fonctions de rang (RANK, DENSE_RANK, ROW_NUMBER, NTILE)](/10-fonctions-de-fenetrage/04-fonctions-de-rang.md)
