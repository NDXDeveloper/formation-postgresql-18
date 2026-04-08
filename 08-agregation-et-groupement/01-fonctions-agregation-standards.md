🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.1. Fonctions d'Agrégation Standards (COUNT, SUM, AVG, MIN, MAX)

## Introduction aux Fonctions d'Agrégation

Les fonctions d'agrégation sont des outils puissants de SQL qui permettent de **calculer une valeur unique à partir d'un ensemble de lignes**. Contrairement aux requêtes classiques qui retournent plusieurs lignes, les agrégations "condensent" les données pour produire des résultats synthétiques.

### Pourquoi les Agrégations ?

Imaginez que vous gérez une base de données de ventes. Plutôt que de parcourir manuellement des milliers de lignes pour connaître :
- Le nombre total de commandes
- Le montant total des ventes
- Le prix moyen d'un produit
- Le produit le plus cher ou le moins cher

Les fonctions d'agrégation font ce travail automatiquement en une seule requête SQL !

---

## Les 5 Fonctions d'Agrégation Fondamentales

PostgreSQL propose cinq fonctions d'agrégation standards qui couvrent la majorité des besoins analytiques de base.

### 1. COUNT() - Compter les Lignes

**Objectif** : Compter le nombre de lignes ou de valeurs non-NULL dans un ensemble de données.

#### Syntaxe

```sql
-- Compter toutes les lignes (y compris les NULL)
COUNT(*)

-- Compter les valeurs non-NULL d'une colonne
COUNT(nom_colonne)

-- Compter les valeurs distinctes (sans doublons)
COUNT(DISTINCT nom_colonne)
```

#### Comportement

- `COUNT(*)` compte **toutes les lignes**, même si certaines colonnes contiennent des valeurs NULL  
- `COUNT(colonne)` compte **uniquement les valeurs non-NULL** de cette colonne  
- `COUNT(DISTINCT colonne)` compte les valeurs uniques (en excluant les doublons et les NULL)

#### Exemples Théoriques

Considérons une table `commandes` :

| id_commande | client_id | montant | date_livraison |
|-------------|-----------|---------|----------------|
| 1           | 101       | 150.00  | 2024-01-15     |
| 2           | 102       | 200.00  | NULL           |
| 3           | 101       | 75.50   | 2024-01-16     |
| 4           | 103       | 300.00  | 2024-01-16     |
| 5           | 102       | 125.00  | NULL           |

**Requêtes :**

```sql
-- Compter le nombre total de commandes
SELECT COUNT(*) AS total_commandes  
FROM commandes;  
-- Résultat : 5
```

```sql
-- Compter les commandes avec une date de livraison renseignée
SELECT COUNT(date_livraison) AS commandes_livrees  
FROM commandes;  
-- Résultat : 3 (les NULL sont ignorés)
```

```sql
-- Compter le nombre de clients distincts ayant passé commande
SELECT COUNT(DISTINCT client_id) AS nombre_clients  
FROM commandes;  
-- Résultat : 3 (clients 101, 102, 103)
```

#### Cas d'Usage Typiques

- **Comptage de lignes** : "Combien d'utilisateurs sont inscrits ?"  
- **Validation de données** : "Combien de lignes ont une valeur NULL dans cette colonne ?"  
- **Analyse de cardinalité** : "Combien de valeurs uniques existe-t-il ?"

---

### 2. SUM() - Additionner les Valeurs

**Objectif** : Calculer la somme totale des valeurs d'une colonne numérique.

#### Syntaxe

```sql
SUM(nom_colonne_numerique)
```

#### Comportement

- Additionne toutes les valeurs **non-NULL** de la colonne
- Les valeurs NULL sont **ignorées** (ne contribuent pas au résultat)
- Retourne NULL si **toutes les valeurs** sont NULL
- Fonctionne uniquement avec des types numériques (INTEGER, NUMERIC, FLOAT, etc.)

#### Exemples Théoriques

Avec la même table `commandes` :

```sql
-- Calculer le chiffre d'affaires total
SELECT SUM(montant) AS ca_total  
FROM commandes;  
-- Résultat : 850.50 (150 + 200 + 75.50 + 300 + 125)
```

```sql
-- Somme avec une condition WHERE
SELECT SUM(montant) AS ca_janvier  
FROM commandes  
WHERE date_livraison >= '2024-01-01'  
  AND date_livraison < '2024-02-01';
-- Résultat : 525.50 (150 + 75.50 + 300)
```

#### Cas d'Usage Typiques

- **Calcul de totaux financiers** : Chiffre d'affaires, coûts totaux, bénéfices  
- **Métriques quantitatives** : Quantité totale vendue, nombre total de connexions  
- **Agrégation de durées** : Temps total passé sur une tâche

#### Points d'Attention

- **Dépassement de capacité** : Avec de très grands nombres, le type de données peut être dépassé (overflow). PostgreSQL gère généralement bien cela avec le type NUMERIC.  
- **Précision décimale** : Utilisez NUMERIC pour les calculs financiers précis plutôt que FLOAT.

---

### 3. AVG() - Calculer la Moyenne

**Objectif** : Calculer la valeur moyenne (arithmétique) d'une colonne numérique.

#### Syntaxe

```sql
AVG(nom_colonne_numerique)
```

#### Comportement

- Calcule : `SUM(colonne) / COUNT(colonne)`
- Les valeurs NULL sont **ignorées** du calcul
- Retourne un type NUMERIC par défaut (avec décimales)
- Retourne NULL si toutes les valeurs sont NULL

#### Exemples Théoriques

Avec la table `commandes` :

```sql
-- Calculer le montant moyen des commandes
SELECT AVG(montant) AS montant_moyen  
FROM commandes;  
-- Résultat : 170.10 (850.50 / 5)
```

```sql
-- Moyenne arrondie à 2 décimales
SELECT ROUND(AVG(montant), 2) AS montant_moyen_arrondi  
FROM commandes;  
-- Résultat : 170.10
```

#### Cas d'Usage Typiques

- **Analyse de tendance centrale** : Panier moyen, salaire moyen, note moyenne  
- **Benchmarking** : Comparer une valeur à la moyenne  
- **Détection d'anomalies** : Identifier les valeurs très éloignées de la moyenne

#### Moyenne vs Médiane

La moyenne (AVG) est sensible aux valeurs extrêmes. Pour des distributions déséquilibrées, la médiane peut être plus représentative (nous verrons les fonctions statistiques dans une section ultérieure).

Exemple : Si 4 employés gagnent 30 000 € et 1 PDG gagne 500 000 €, la moyenne sera de 130 000 €, ce qui ne reflète pas la réalité de la majorité.

---

### 4. MIN() - Trouver la Valeur Minimale

**Objectif** : Retourner la plus petite valeur d'une colonne.

#### Syntaxe

```sql
MIN(nom_colonne)
```

#### Comportement

- Fonctionne avec **tous les types de données** comparables :
  - Numériques : retourne le plus petit nombre
  - Texte : retourne la première valeur alphabétique (ordre lexicographique)
  - Dates : retourne la date la plus ancienne
  - Booléens : FALSE < TRUE
- Les valeurs NULL sont **ignorées**
- Retourne NULL si toutes les valeurs sont NULL

#### Exemples Théoriques

```sql
-- Trouver la commande la moins chère
SELECT MIN(montant) AS montant_min  
FROM commandes;  
-- Résultat : 75.50
```

```sql
-- Trouver la date de première livraison
SELECT MIN(date_livraison) AS premiere_livraison  
FROM commandes;  
-- Résultat : 2024-01-15
```

Avec une table `produits` :

| id_produit | nom           | prix  | categorie     |
|------------|---------------|-------|---------------|
| 1          | Laptop        | 899   | Électronique  |
| 2          | Souris        | 25    | Accessoires   |
| 3          | Clavier       | 75    | Accessoires   |
| 4          | Écran         | 250   | Électronique  |

```sql
-- Trouver le produit dont le nom vient en premier alphabétiquement
SELECT MIN(nom) AS premier_produit_alpha  
FROM produits;  
-- Résultat : 'Clavier' (ordre alphabétique : C < E < L < S)
```

#### Cas d'Usage Typiques

- **Recherche de bornes** : Prix minimum, date la plus ancienne  
- **Validation de seuils** : Vérifier si une valeur minimale est atteinte  
- **Analyse temporelle** : Première occurrence d'un événement

---

### 5. MAX() - Trouver la Valeur Maximale

**Objectif** : Retourner la plus grande valeur d'une colonne.

#### Syntaxe

```sql
MAX(nom_colonne)
```

#### Comportement

- Fonctionne avec **tous les types de données** comparables (comme MIN)
  - Numériques : retourne le plus grand nombre
  - Texte : retourne la dernière valeur alphabétique
  - Dates : retourne la date la plus récente
  - Booléens : TRUE > FALSE
- Les valeurs NULL sont **ignorées**
- Retourne NULL si toutes les valeurs sont NULL

#### Exemples Théoriques

```sql
-- Trouver la commande la plus chère
SELECT MAX(montant) AS montant_max  
FROM commandes;  
-- Résultat : 300.00
```

```sql
-- Trouver la date de dernière livraison
SELECT MAX(date_livraison) AS derniere_livraison  
FROM commandes;  
-- Résultat : 2024-01-16
```

```sql
-- Trouver le produit le plus cher par catégorie (nécessite GROUP BY)
SELECT categorie, MAX(prix) AS prix_max  
FROM produits  
GROUP BY categorie;  
-- Résultat :
-- Électronique | 899
-- Accessoires  | 75
```

#### Cas d'Usage Typiques

- **Identification de records** : Meilleur score, prix le plus élevé  
- **Analyse temporelle** : Dernière transaction, date la plus récente  
- **Détection de pics** : Valeur maximale atteinte dans une période

---

## Combiner Plusieurs Agrégations

Il est très courant de combiner plusieurs fonctions d'agrégation dans une même requête pour obtenir une vue d'ensemble.

### Exemple : Statistiques Globales

```sql
SELECT
    COUNT(*) AS total_commandes,
    COUNT(DISTINCT client_id) AS nombre_clients,
    SUM(montant) AS ca_total,
    AVG(montant) AS panier_moyen,
    MIN(montant) AS commande_min,
    MAX(montant) AS commande_max
FROM commandes;
```

**Résultat théorique :**

| total_commandes | nombre_clients | ca_total | panier_moyen | commande_min | commande_max |
|-----------------|----------------|----------|--------------|--------------|--------------|
| 5               | 3              | 850.50   | 170.10       | 75.50        | 300.00       |

Cette requête fournit un **tableau de bord instantané** des commandes en une seule ligne !

---

## Gestion des Valeurs NULL dans les Agrégations

### Comportement Général

**Règle d'or** : Les fonctions d'agrégation (sauf COUNT(*)) **ignorent les valeurs NULL**.

### Tableau Récapitulatif

| Fonction       | Traitement des NULL               | Résultat si toutes les valeurs sont NULL |
|----------------|-----------------------------------|------------------------------------------|
| COUNT(*)       | Compte toutes les lignes          | 0                                        |
| COUNT(colonne) | Ignore les NULL                   | 0                                        |
| SUM(colonne)   | Ignore les NULL                   | NULL                                     |
| AVG(colonne)   | Ignore les NULL                   | NULL                                     |
| MIN(colonne)   | Ignore les NULL                   | NULL                                     |
| MAX(colonne)   | Ignore les NULL                   | NULL                                     |

### Exemple Pratique

Table `ventes_mensuelles` :

| mois    | ventes   |
|---------|----------|
| Janvier | 1000     |
| Février | NULL     |
| Mars    | 1500     |
| Avril   | NULL     |

```sql
SELECT
    COUNT(*) AS nb_lignes,                -- 4
    COUNT(ventes) AS nb_ventes_renseignees, -- 2
    SUM(ventes) AS total,                 -- 2500
    AVG(ventes) AS moyenne                -- 1250 (2500 / 2, pas 2500 / 4 !)
FROM ventes_mensuelles;
```

**Point clé** : La moyenne est calculée sur les valeurs non-NULL uniquement. Si vous voulez traiter les NULL comme des zéros, utilisez `COALESCE` :

```sql
SELECT
    AVG(COALESCE(ventes, 0)) AS moyenne_avec_nulls_comme_zero
FROM ventes_mensuelles;
-- Résultat : 625 (2500 / 4)
```

---

## Agrégations avec Filtrage (WHERE)

Les agrégations peuvent être combinées avec la clause WHERE pour calculer des statistiques sur un sous-ensemble de données.

### Exemple

```sql
-- Statistiques uniquement sur les commandes livrées
SELECT
    COUNT(*) AS commandes_livrees,
    AVG(montant) AS panier_moyen_livrees
FROM commandes  
WHERE date_livraison IS NOT NULL;  
-- Résultat :
-- commandes_livrees | 3
-- panier_moyen_livrees | 175.17 (525.50 / 3)
```

**Ordre d'exécution** :
1. PostgreSQL applique d'abord le filtre WHERE  
2. Puis calcule les agrégations sur les lignes sélectionnées

---

## Agrégations sur Toutes les Lignes vs Par Groupe

### Sans GROUP BY : Agrégation Globale

Quand aucune clause GROUP BY n'est présente, les fonctions d'agrégation s'appliquent à **toutes les lignes** de la table (après filtrage WHERE si présent).

```sql
-- Une seule ligne de résultat : statistique globale
SELECT AVG(montant) AS moyenne_globale  
FROM commandes;  
```

### Avec GROUP BY : Agrégations Par Groupe

La clause GROUP BY permet de calculer des agrégations **pour chaque groupe de valeurs**. Nous verrons cela en détail dans la section suivante (8.3), mais voici un aperçu :

```sql
-- Une ligne de résultat par client
SELECT
    client_id,
    COUNT(*) AS nb_commandes,
    SUM(montant) AS total_depense
FROM commandes  
GROUP BY client_id;  
```

**Résultat théorique :**

| client_id | nb_commandes | total_depense |
|-----------|--------------|---------------|
| 101       | 2            | 225.50        |
| 102       | 2            | 325.00        |
| 103       | 1            | 300.00        |

---

## Expressions dans les Agrégations

Les fonctions d'agrégation peuvent s'appliquer à des **expressions calculées**, pas uniquement à des colonnes simples.

### Exemples

```sql
-- Calculer la somme des montants TTC (montant HT * 1.20)
SELECT SUM(montant * 1.20) AS ca_ttc  
FROM commandes;  
-- Résultat : 1020.60
```

```sql
-- Compter les commandes supérieures à 100€
SELECT COUNT(*) AS commandes_importantes  
FROM commandes  
WHERE montant > 100;  
-- Alternative avec CASE dans COUNT :
SELECT COUNT(CASE WHEN montant > 100 THEN 1 END) AS commandes_importantes  
FROM commandes;  
```

```sql
-- Moyenne des montants après réduction de 10%
SELECT AVG(montant * 0.90) AS panier_moyen_promo  
FROM commandes;  
```

---

## Bonnes Pratiques et Pièges à Éviter

### ✅ Bonnes Pratiques

1. **Nommer les résultats avec AS** : Toujours utiliser un alias clair pour les colonnes agrégées
   ```sql
   SELECT COUNT(*) AS total  -- ✅ Clair
   -- vs
   SELECT COUNT(*)           -- ❌ Nom par défaut "count" peu explicite
   ```

2. **Utiliser DISTINCT pour éliminer les doublons** :
   ```sql
   SELECT COUNT(DISTINCT client_id) AS clients_uniques
   FROM commandes;
   ```

3. **Gérer les NULL explicitement** :
   ```sql
   -- Si vous voulez traiter les NULL comme des zéros
   SELECT SUM(COALESCE(montant, 0)) AS total
   FROM ventes;
   ```

4. **Précision des types** :
   ```sql
   -- Pour les calculs financiers, forcer NUMERIC
   SELECT SUM(montant::NUMERIC) AS total_precis
   FROM commandes;
   ```

### ❌ Pièges à Éviter

1. **Mélanger colonnes agrégées et non-agrégées sans GROUP BY** :
   ```sql
   -- ❌ ERREUR : client_id n'est pas agrégé
   SELECT client_id, COUNT(*)
   FROM commandes;

   -- ✅ Correct avec GROUP BY
   SELECT client_id, COUNT(*)
   FROM commandes
   GROUP BY client_id;
   ```

2. **Oublier que AVG ignore les NULL** :
   - Si vous avez des NULL et voulez les compter comme zéros, utilisez COALESCE

3. **Confondre COUNT(*) et COUNT(colonne)** :
   ```sql
   -- Différent si la colonne contient des NULL !
   SELECT COUNT(*), COUNT(date_livraison)
   FROM commandes;
   ```

4. **Dépassement de capacité avec SUM sur de grandes tables** :
   - Utilisez NUMERIC pour éviter les overflows sur de très grands nombres

---

## Performances des Agrégations

### Facteurs d'Impact

1. **Taille de la table** : Plus il y a de lignes, plus l'agrégation prend du temps  
2. **Index** : Les index peuvent accélérer certaines agrégations (MIN/MAX surtout)  
3. **Filtrage WHERE** : Réduire le nombre de lignes avant agrégation améliore les performances  
4. **Type de données** : Les types numériques sont plus rapides que les types texte

### Optimisation MIN/MAX avec Index

MIN et MAX peuvent être **extrêmement rapides** si un index existe sur la colonne :

```sql
-- Si un index existe sur montant, PostgreSQL peut récupérer
-- directement la première/dernière valeur de l'index (O(1))
SELECT MIN(montant), MAX(montant)  
FROM commandes;  
```

### Agrégations vs Scan Complet

- **COUNT(*)** nécessite un scan complet (parcourir toutes les lignes)  
- **SUM, AVG** nécessitent aussi un scan complet  
- **MIN/MAX** peuvent utiliser un index pour une récupération directe

---

## Récapitulatif : Tableau de Référence Rapide

| Fonction | Objectif | Traite les NULL | Type de Résultat | Exemple |
|----------|----------|-----------------|------------------|---------|
| COUNT(*) | Compter toutes les lignes | Compte tout | BIGINT | `SELECT COUNT(*) FROM table` |
| COUNT(col) | Compter valeurs non-NULL | Ignore NULL | BIGINT | `SELECT COUNT(email) FROM users` |
| SUM(col) | Additionner les valeurs | Ignore NULL | Type de la colonne | `SELECT SUM(prix) FROM produits` |
| AVG(col) | Calculer la moyenne | Ignore NULL | NUMERIC | `SELECT AVG(note) FROM evaluations` |
| MIN(col) | Trouver le minimum | Ignore NULL | Type de la colonne | `SELECT MIN(date_creation) FROM articles` |
| MAX(col) | Trouver le maximum | Ignore NULL | Type de la colonne | `SELECT MAX(temperature) FROM mesures` |

---

## À Retenir

1. **Les agrégations condensent plusieurs lignes en une seule valeur**  
2. **Les NULL sont généralement ignorés** (sauf COUNT(*))  
3. **COUNT(*) compte les lignes, COUNT(colonne) compte les non-NULL**  
4. **Toujours nommer les colonnes agrégées avec AS**  
5. **MIN/MAX fonctionnent sur tous types comparables** (pas uniquement numériques)  
6. **Les agrégations nécessitent GROUP BY pour calculer par groupe** (section suivante)  
7. **Les expressions sont acceptées dans les fonctions d'agrégation**

---

## Prochaines Étapes

Maintenant que vous maîtrisez les fonctions d'agrégation standards, vous êtes prêt pour :

- **Section 8.2** : Fonctions d'agrégation statistiques (STDDEV, VARIANCE, PERCENTILE)  
- **Section 8.3** : GROUP BY et HAVING pour des agrégations par groupe  
- **Section 8.4** : Extensions de groupement (ROLLUP, CUBE, GROUPING SETS)

Les agrégations sont un pilier fondamental de l'analyse de données en SQL. Avec ces cinq fonctions, vous pouvez déjà répondre à une multitude de questions analytiques !

⏭️ [Fonctions d'agrégation statistiques (STDDEV, VARIANCE, CORR, PERCENTILE)](/08-agregation-et-groupement/02-fonctions-agregation-statistiques.md)
