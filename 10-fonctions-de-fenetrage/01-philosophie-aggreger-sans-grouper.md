🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.1. Philosophie : Agréger sans grouper

## Introduction aux Fonctions de Fenêtrage

Les **fonctions de fenêtrage** (ou *window functions* en anglais) représentent l'une des fonctionnalités les plus puissantes et élégantes de SQL moderne. Elles permettent de réaliser des calculs sophistiqués qui seraient autrement très complexes, voire impossibles, avec les outils SQL traditionnels.

## Le Problème avec GROUP BY

### Comprendre la limitation

Lorsque vous utilisez `GROUP BY` pour effectuer des agrégations, vous êtes confronté à une contrainte fondamentale : **vous perdez le détail de vos lignes individuelles**.

Prenons un exemple concret. Imaginons une table `ventes` contenant les ventes d'un magasin :

```
| id | vendeur    | produit      | montant |
|----|------------|--------------|---------|
| 1  | Alice      | Ordinateur   | 1200    |
| 2  | Alice      | Souris       | 25      |
| 3  | Bob        | Clavier      | 75      |
| 4  | Bob        | Écran        | 350     |
| 5  | Caroline   | Ordinateur   | 1200    |
```

Si vous voulez connaître le **total des ventes par vendeur**, vous utilisez `GROUP BY` :

```sql
SELECT vendeur, SUM(montant) AS total_ventes
FROM ventes
GROUP BY vendeur;
```

Résultat :

```
| vendeur  | total_ventes |
|----------|--------------|
| Alice    | 1225         |
| Bob      | 425          |
| Caroline | 1200         |
```

**Le problème** : vous obtenez une ligne par vendeur, mais vous avez perdu le détail de chaque vente individuelle. Impossible de voir simultanément :
- Le montant de chaque vente
- Le total des ventes du vendeur

### Le dilemme classique

Que faire si vous voulez afficher **chaque vente avec le total du vendeur** sur la même ligne ?

Avec `GROUP BY`, vous êtes bloqué. Vous devriez :
1. Soit faire une sous-requête complexe
2. Soit faire une jointure de la table sur elle-même
3. Soit traiter les données dans votre code applicatif

**Toutes ces solutions sont lourdes, peu lisibles et peu performantes.**

## La Révolution des Window Functions

### Le concept fondamental

Les **fonctions de fenêtrage** résolvent élégamment ce problème en introduisant un concept révolutionnaire :

> **Effectuer des calculs d'agrégation SANS regrouper les lignes**

Autrement dit, vous pouvez :
- Garder toutes vos lignes individuelles intactes
- Calculer des agrégations (sommes, moyennes, comptages, etc.)
- Afficher les deux côte à côte dans le même résultat

### Exemple fondateur

Reprenons notre exemple précédent. Avec une window function, nous pouvons écrire :

```sql
SELECT
    id,
    vendeur,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur) AS total_vendeur
FROM ventes;
```

Résultat :

```
| id | vendeur  | produit      | montant | total_vendeur |
|----|----------|--------------|---------|---------------|
| 1  | Alice    | Ordinateur   | 1200    | 1225          |
| 2  | Alice    | Souris       | 25      | 1225          |
| 3  | Bob      | Clavier      | 75      | 425           |
| 4  | Bob      | Écran        | 350     | 425           |
| 5  | Caroline | Ordinateur   | 1200    | 1200          |
```

**Magique !** Chaque ligne garde son identité (id, produit, montant), mais affiche également le total des ventes du vendeur.

### Décryptage de la syntaxe

La partie clé est `OVER (PARTITION BY vendeur)` :

- **`OVER`** : Indique qu'on utilise une window function
- **`PARTITION BY vendeur`** : Définit les "groupes" (ou "partitions") sur lesquels calculer l'agrégation, SANS fusionner les lignes

C'est comme si vous disiez : *"Calcule la somme pour chaque vendeur, mais n'élimine pas les lignes individuelles"*

## Comparaison GROUP BY vs Window Functions

### Tableau comparatif

| Aspect | GROUP BY | Window Functions |
|--------|----------|------------------|
| **Nombre de lignes en sortie** | Une ligne par groupe | Toutes les lignes conservées |
| **Détail des données** | Perdu (agrégé) | Conservé |
| **Calculs d'agrégation** | Oui (SUM, AVG, COUNT...) | Oui (mêmes fonctions) |
| **Calculs complexes** | Limités | Étendus (rangs, fenêtres glissantes...) |
| **Lisibilité** | Simple pour agrégation pure | Plus expressive pour analyses avancées |
| **Performance** | Bonne pour groupements simples | Optimisée pour analyses fenêtrées |

### Quand utiliser l'un ou l'autre ?

**Utilisez GROUP BY quand :**
- Vous voulez uniquement des totaux/moyennes par groupe
- Vous n'avez pas besoin du détail ligne par ligne
- Exemple : "Donnez-moi le chiffre d'affaires total par région"

**Utilisez Window Functions quand :**
- Vous voulez garder le détail ET avoir des agrégations
- Vous voulez comparer chaque ligne à un total ou une moyenne
- Vous devez calculer des classements, des moyennes mobiles, des écarts
- Exemple : "Montrez-moi chaque vente avec le pourcentage qu'elle représente du total du vendeur"

## Cas d'Usage Typiques

### 1. Pourcentage du total

Calculer la part de chaque vente dans le total du vendeur :

```sql
SELECT
    vendeur,
    produit,
    montant,
    ROUND(montant * 100.0 / SUM(montant) OVER (PARTITION BY vendeur), 2) AS pourcentage
FROM ventes;
```

### 2. Comparaison à la moyenne

Voir si chaque vente est au-dessus ou en-dessous de la moyenne du vendeur :

```sql
SELECT
    vendeur,
    produit,
    montant,
    AVG(montant) OVER (PARTITION BY vendeur) AS moyenne_vendeur,
    montant - AVG(montant) OVER (PARTITION BY vendeur) AS ecart_moyenne
FROM ventes;
```

### 3. Cumuls et totaux glissants

Afficher le cumul des ventes pour chaque vendeur (nous verrons cela en détail plus tard) :

```sql
SELECT
    vendeur,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur ORDER BY id) AS cumul
FROM ventes;
```

## La "Fenêtre" : Un Concept Visuel

### Pourquoi "Window" ?

Le terme *"window"* (fenêtre) n'est pas anodin. Imaginez que vous regardez vos données à travers une **fenêtre mobile** :

- La fenêtre peut englober **toutes les lignes d'un groupe** (comme dans nos exemples avec `PARTITION BY`)
- La fenêtre peut être **restreinte** à quelques lignes autour de la ligne courante
- La fenêtre peut **glisser** le long des données (pour des moyennes mobiles, par exemple)

### Analogie visuelle

Pensez à une feuille de calcul Excel :
- Vous avez vos données ligne par ligne
- Vous pouvez ajouter une colonne qui calcule un total pour un sous-ensemble de lignes
- Mais les lignes individuelles restent visibles

Les window functions font exactement cela, mais de manière déclarative et optimisée en SQL.

## Avantages des Window Functions

### 1. **Lisibilité**

Au lieu d'écrire des sous-requêtes complexes imbriquées, vous avez une requête claire et linéaire.

### 2. **Performance**

PostgreSQL optimise l'exécution des window functions. Un seul passage sur les données peut calculer plusieurs fenêtres simultanément.

### 3. **Expressivité**

Vous pouvez exprimer des analyses qui seraient très difficiles autrement :
- Classements (top N par catégorie)
- Comparaisons avec la ligne précédente/suivante
- Moyennes mobiles
- Percentiles et distributions

### 4. **Maintenabilité**

Le code est plus facile à comprendre et à modifier. Pas de jointures alambiquées ou de sous-requêtes multi-niveaux.

## Limitations et Précautions

### Ce que les window functions NE font PAS

1. **Elles ne filtrent pas les lignes** : Si vous voulez filtrer, utilisez `WHERE` avant ou `HAVING` dans une requête englobante
2. **Elles ne modifient pas les données** : Les window functions sont en lecture seule (comme toutes les fonctions dans SELECT)
3. **Elles ne peuvent pas être dans WHERE** : Les calculs de fenêtrage se font après le filtrage

### Performance

Bien que généralement performantes, les window functions peuvent devenir coûteuses sur de très grandes tables sans indexation appropriée. Surveillez toujours les temps d'exécution avec `EXPLAIN ANALYZE`.

## Transition vers la Suite

Maintenant que vous comprenez la **philosophie** des window functions, nous allons explorer :

- **La clause OVER** en détail et ses options
- **PARTITION BY** pour définir les groupes
- **ORDER BY** pour définir l'ordre dans les fenêtres
- **Les frames de fenêtre** (ROWS, RANGE, GROUPS) pour contrôler précisément l'étendue des calculs
- **Les fonctions spécialisées** : rangs, décalages, valeurs précédentes/suivantes

Chaque concept s'appuie sur cette idée fondamentale : **calculer sur un ensemble de lignes tout en conservant le détail individuel**.

## Résumé

Les **window functions** révolutionnent l'analyse de données en SQL en permettant :

- ✅ D'agréger sans perdre le détail des lignes
- ✅ De comparer chaque ligne à un groupe ou un total
- ✅ D'effectuer des calculs sophistiqués (rangs, moyennes mobiles) de manière déclarative
- ✅ D'écrire des requêtes plus lisibles et maintenables

**Principe clé** : `FONCTION_AGREGATION OVER (PARTITION BY colonne)` calcule l'agrégation pour chaque groupe défini par `colonne`, mais conserve toutes les lignes.

---


⏭️ [La clause OVER, PARTITION BY et ORDER BY](/10-fonctions-de-fenetrage/02-clause-over-partition-order.md)
