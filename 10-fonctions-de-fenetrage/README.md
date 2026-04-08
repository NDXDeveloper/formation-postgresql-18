🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10. Fonctions de Fenêtrage (Window Functions)

## Introduction au Chapitre

Bienvenue dans l'un des chapitres les plus puissants et les plus transformateurs de ce cours sur PostgreSQL ! Les **fonctions de fenêtrage** (ou *window functions* en anglais) représentent une évolution majeure dans la manière d'analyser et de manipuler les données en SQL.

Si vous avez déjà ressenti la frustration de ne pouvoir obtenir à la fois des **détails ligne par ligne** ET des **calculs agrégés** dans la même requête, ce chapitre va changer votre vie de développeur SQL.

## Qu'est-ce qu'une Window Function ?

### Une Révolution dans l'Analyse de Données

Les window functions sont des fonctions qui effectuent des calculs **sur un ensemble de lignes** (une "fenêtre") tout en **conservant les lignes individuelles** dans le résultat.

**L'analogie de la fenêtre** : Imaginez que vous regardez vos données à travers une fenêtre mobile. Cette fenêtre peut :
- Englober toutes les lignes d'un groupe
- Se limiter à quelques lignes autour de la ligne courante
- Glisser le long de vos données ordonnées

À chaque position, vous pouvez effectuer des calculs (sommes, moyennes, classements, etc.) sur ce que vous voyez à travers la fenêtre, **sans fusionner les lignes**.

### Le Problème qu'Elles Résolvent

**Situation classique frustrante** :

Vous avez une table de ventes et vous voulez afficher :
- Chaque vente individuelle (produit, montant)
- Le total des ventes du vendeur
- Le classement de cette vente parmi toutes les ventes du vendeur
- La différence avec la vente précédente

**Avec SQL traditionnel** : Vous devez jongler entre `GROUP BY`, sous-requêtes complexes, jointures de la table sur elle-même, ou traiter les données dans votre code applicatif. Le résultat est souvent lourd, peu lisible et peu performant.

**Avec les window functions** : Une seule requête élégante fait tout cela !

```sql
SELECT
    vendeur,
    produit,
    montant,
    SUM(montant) OVER (PARTITION BY vendeur) AS total_vendeur,
    RANK() OVER (PARTITION BY vendeur ORDER BY montant DESC) AS rang,
    montant - LAG(montant) OVER (PARTITION BY vendeur ORDER BY date) AS variation
FROM ventes;
```

Magique, n'est-ce pas ? Et c'est juste un aperçu de ce que nous allons apprendre.

## Pourquoi les Window Functions sont Essentielles

### 1. Expressivité

Elles permettent d'exprimer des analyses complexes de manière **naturelle et lisible**, là où il faudrait sinon plusieurs niveaux de sous-requêtes ou des jointures alambiquées.

### 2. Performance

PostgreSQL optimise l'exécution des window functions. Un seul passage sur les données peut calculer plusieurs fenêtres simultanément, contrairement aux multiples sous-requêtes qui parcourent les données plusieurs fois.

### 3. Nouvelles Possibilités

Certaines analyses sont presque impossibles avec SQL traditionnel mais deviennent triviales avec les window functions :
- Calculer des moyennes mobiles
- Trouver le top N par catégorie
- Comparer chaque ligne avec la ligne précédente ou suivante
- Calculer des cumuls
- Détecter des tendances et des anomalies

### 4. Standard SQL

Les window functions font partie du standard SQL depuis SQL:2003. Elles sont supportées par tous les SGBD modernes (PostgreSQL, MySQL 8+, SQL Server, Oracle, etc.), ce qui rend vos compétences transférables.

## Domaines d'Application

Les window functions sont particulièrement utiles dans :

### Analyse de Données (Data Analytics)
- Tableaux de bord avec métriques enrichies
- Rapports financiers avec cumuls et comparaisons
- Analyses de tendances et de saisonnalité
- Détection d'anomalies

### Business Intelligence
- Top N produits/vendeurs/clients par catégorie
- Analyse de cohortes
- Segmentation RFM (Recency, Frequency, Monetary)
- Calculs de pourcentages et de contributions

### Finance et Trading
- Moyennes mobiles (SMA, EMA)
- Indicateurs techniques (RSI, MACD, Bollinger Bands)
- Calculs de volatilité
- Détection de points de retournement

### Web Analytics
- Analyse de parcours utilisateur
- Taux de conversion par étapes (funnel analysis)
- Rétention et churn
- Analyse de séquences d'événements

### E-commerce
- Classements de produits
- Recommandations basées sur les achats
- Analyse de panier moyen
- Évolution des prix et promotions

## Vue d'Ensemble du Chapitre

Ce chapitre est structuré de manière progressive pour vous amener du niveau débutant à une maîtrise avancée :

### 10.1. Philosophie : Agréger sans grouper
Le concept fondamental qui sous-tend les window functions. Comprendre **pourquoi** elles existent et ce qui les rend différentes de `GROUP BY`.

### 10.2. La clause OVER, PARTITION BY et ORDER BY
La syntaxe de base : comment définir une fenêtre, la diviser en groupes, et ordonner les lignes.

### 10.3. Frames de fenêtre : ROWS vs RANGE vs GROUPS
Le contrôle précis de la fenêtre : définir exactement quelles lignes sont incluses dans chaque calcul.

### 10.4. Fonctions de rang (RANK, DENSE_RANK, ROW_NUMBER, NTILE)
Classer, numéroter, et répartir vos données en groupes.

### 10.5. Fonctions de valeur (LAG, LEAD, FIRST_VALUE, LAST_VALUE)
Accéder aux lignes précédentes, suivantes, ou aux extrémités de la fenêtre.

### 10.6. Fonctions d'agrégation en fenêtrage
Utiliser SUM, AVG, COUNT, MIN, MAX et les fonctions statistiques dans un contexte de fenêtrage.

### 10.7. Cas d'usage avancés : Top N par groupe, moyenne mobile
Résoudre des problèmes réels avec des patterns éprouvés en production.

## Ce que Vous Allez Apprendre

À la fin de ce chapitre, vous serez capable de :

- ✅ **Comprendre** la philosophie et le fonctionnement des window functions  
- ✅ **Écrire** des requêtes avec OVER, PARTITION BY, et ORDER BY  
- ✅ **Contrôler** précisément les frames de fenêtre (ROWS, RANGE, GROUPS)  
- ✅ **Utiliser** toutes les fonctions de fenêtrage (rangs, valeurs, agrégations)  
- ✅ **Résoudre** des problèmes complexes : top N, moyennes mobiles, cumuls  
- ✅ **Optimiser** les performances de vos requêtes avec window functions  
- ✅ **Analyser** des données avec des techniques professionnelles

## Prérequis

Pour tirer le meilleur parti de ce chapitre, vous devriez être à l'aise avec :

- Les requêtes `SELECT` de base (Chapitre 5)
- Les jointures (Chapitre 7)
- Les agrégations avec `GROUP BY` et `HAVING` (Chapitre 8)
- Les sous-requêtes et CTE (Chapitre 9)

Si ces concepts ne sont pas encore clairs, nous vous recommandons de les réviser avant de plonger dans les window functions.

## Approche Pédagogique

### Progression Graduelle

Nous avons conçu ce chapitre pour être **accessible aux débutants** tout en allant jusqu'à des **techniques avancées**. Chaque section s'appuie sur les précédentes.

### Exemples Concrets

Chaque concept est illustré par des **exemples avec données et résultats**, pas seulement de la théorie abstraite. Vous verrez exactement ce qui se passe à chaque étape.

### Visualisations

Les window functions sont parfois contre-intuitives. Nous utilisons des **visualisations** et des **tableaux** pour rendre les concepts plus tangibles.

### Comparaisons

Nous comparons systématiquement les window functions avec les approches traditionnelles (GROUP BY, sous-requêtes) pour que vous compreniez **quand** et **pourquoi** les utiliser.

### Cas d'Usage Réels

Au-delà de la syntaxe, nous montrons des **patterns pratiques** utilisés en production : dashboards, analyses de cohortes, détection d'anomalies, etc.

## Terminologie

Avant de commencer, clarifions quelques termes que nous utiliserons tout au long du chapitre :

### Window Function (Fonction de Fenêtrage)
Une fonction SQL qui opère sur un ensemble de lignes (la "fenêtre") et retourne une valeur pour chaque ligne, sans regrouper les lignes.

### Fenêtre (Window)
L'ensemble de lignes sur lequel la fonction opère pour une ligne donnée. La fenêtre peut être :
- Toute la table
- Un groupe de lignes (partition)
- Un sous-ensemble glissant de lignes

### Partition
Un sous-ensemble de lignes défini par `PARTITION BY`. Chaque partition est traitée indépendamment, comme des "mini-tables" virtuelles.

### Frame (Cadre)
Le sous-ensemble précis de la partition utilisé pour le calcul d'une ligne donnée. Le frame peut glisser le long de la partition.

### Clause OVER
Le mot-clé qui indique qu'on utilise une window function. C'est la "signature" des window functions.

## Différences Clés avec GROUP BY

Il est crucial de comprendre cette différence fondamentale :

| Aspect | GROUP BY | Window Functions |
|--------|----------|------------------|
| **Nombre de lignes** | Réduit (une par groupe) | Conserve toutes les lignes |
| **Détail des données** | Perdu (agrégé) | Conservé |
| **Calculs possibles** | Agrégations simples | Agrégations + rangs + comparaisons |
| **Contexte** | Groupe fermé | Fenêtre flexible |

**Exemple illustratif** :

```sql
-- GROUP BY : 3 lignes (une par vendeur)
SELECT vendeur, SUM(ventes) AS total  
FROM ventes  
GROUP BY vendeur;  

-- Window Function : toutes les lignes conservées
SELECT vendeur, produit, ventes,
       SUM(ventes) OVER (PARTITION BY vendeur) AS total_vendeur
FROM ventes;
```

Dans le premier cas, vous perdez le détail des produits. Dans le second, chaque ligne de vente affiche son propre détail ET le total du vendeur.

## Compatibilité et Versions

Les window functions sont disponibles dans PostgreSQL depuis la **version 8.4** (2009). Toutes les fonctionnalités que nous allons voir sont disponibles dans PostgreSQL 9.x et supérieures.

**PostgreSQL 18** (version couverte par ce cours) apporte des optimisations supplémentaires :
- Amélioration du planificateur pour les window functions
- Meilleure performance sur les grandes fenêtres
- Optimisation des auto-éliminations (self-joins)

## Conseils pour Apprendre

### 1. Pratiquez avec de Petites Données d'Abord

Créez de petites tables de test (5-10 lignes) pour **voir** exactement ce que fait chaque fonction. C'est plus instructif que de travailler sur de grandes tables où les résultats sont difficiles à interpréter.

### 2. Utilisez EXPLAIN pour Comprendre

N'hésitez pas à utiliser `EXPLAIN` pour voir comment PostgreSQL exécute vos requêtes avec window functions. Cela vous aidera à comprendre les performances.

### 3. Comparez avec GROUP BY

Quand vous apprenez une nouvelle window function, essayez d'obtenir le même résultat avec `GROUP BY` et des sous-requêtes. Vous apprécierez d'autant plus la simplicité des window functions !

### 4. Construisez Progressivement

Commencez simple (par exemple, juste `OVER ()`), puis ajoutez `PARTITION BY`, puis `ORDER BY`, puis des frames. Ne sautez pas les étapes.

### 5. Expérimentez

Les window functions sont un terrain de jeu fantastique. N'ayez pas peur d'expérimenter, de combiner plusieurs fonctions, d'essayer différents ordres.

## Un Mot sur la Performance

Les window functions sont généralement **très performantes** dans PostgreSQL. Cependant :

- ✅ **Bonne nouvelle** : PostgreSQL optimise intelligemment les calculs de fenêtres multiples  
- ✅ **Index utiles** : Les colonnes dans `PARTITION BY` et `ORDER BY` bénéficient d'index  
- ⚠️ **Attention** : Les fenêtres très larges (ex: `UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) sur de très grandes tables peuvent être coûteuses

Nous aborderons les aspects de performance en détail dans chaque section.

## Format des Exemples

Dans tout ce chapitre, nous suivrons ce format pour les exemples :

1. **Le problème** : Ce que nous voulons accomplir  
2. **La requête SQL** : Avec window functions  
3. **Les données d'entrée** : Sous forme de tableau  
4. **Le résultat** : Avec explications ligne par ligne si nécessaire  
5. **L'analyse** : Pourquoi ça fonctionne

Exemple de format :

```sql
SELECT
    colonne1,
    colonne2,
    FONCTION() OVER (...) AS resultat
FROM table;
```

**Données** :
```
| colonne1 | colonne2 |
|----------|----------|
| ...      | ...      |
```

**Résultat** :
```
| colonne1 | colonne2 | resultat |
|----------|----------|----------|
| ...      | ...      | ...      |
```

**Explication** : Ce qui se passe et pourquoi.

## Ressources Complémentaires

Une fois ce chapitre complété, vous pourrez approfondir avec :

- **Documentation officielle PostgreSQL** : Section "Window Functions"  
- **PostgreSQL Up and Running** (livre) : Chapitre sur les window functions  
- **Modern SQL** (site web) : Excellentes explications sur les window functions  
- **Use The Index, Luke** : Aspects de performance

## Mindset pour ce Chapitre

Les window functions peuvent sembler déroutantes au début, surtout si vous êtes habitué à penser en termes de `GROUP BY`. Voici le bon état d'esprit :

- 🧠 **Pensez "vue d'ensemble + détail"** : Vous voulez garder les détails ET avoir une vue d'ensemble  
- 🧠 **Pensez "fenêtre glissante"** : Imaginez une fenêtre qui se déplace sur vos données  
- 🧠 **Pensez "contexte"** : Chaque ligne voit son contexte (les lignes autour d'elle)  
- 🧠 **Pensez "sans fusion"** : Contrairement à GROUP BY, les lignes ne fusionnent jamais

## Prêt à Commencer ?

Les window functions vont transformer votre façon de penser SQL. Elles sont un outil **indispensable** pour tout développeur ou analyste qui travaille sérieusement avec des données.

Ce chapitre est dense mais structuré pour être **progressif** et **accessible**. Prenez votre temps, expérimentez avec les exemples, et n'hésitez pas à relire les sections qui vous semblent complexes.

**Conseil final** : Créez-vous une petite base de données de test avec des données que vous comprenez bien (par exemple, vos propres achats, un budget personnel, des scores de jeux). Les window functions deviennent beaucoup plus intuitives quand vous les utilisez sur des données qui ont du sens pour vous.

---

**Allons-y ! Commençons par comprendre la philosophie fondamentale des window functions dans la section suivante.**


⏭️ [Philosophie : Agréger sans grouper](/10-fonctions-de-fenetrage/01-philosophie-aggreger-sans-grouper.md)
