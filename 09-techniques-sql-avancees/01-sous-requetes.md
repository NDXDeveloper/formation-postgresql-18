🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1. Sous-requêtes (scalaires, vectorielles, table) et Performance

## Introduction

Les sous-requêtes (ou *subqueries* en anglais) sont des requêtes SQL imbriquées à l'intérieur d'une autre requête SQL. Elles permettent de décomposer des problèmes complexes en étapes plus simples et constituent un outil puissant pour interroger des bases de données relationnelles.

Dans ce chapitre, nous allons explorer les trois types principaux de sous-requêtes et comprendre leurs implications en termes de performance.

---

## Qu'est-ce qu'une sous-requête ?

Une sous-requête est simplement une requête `SELECT` placée entre parenthèses et utilisée dans une autre requête SQL. Elle peut apparaître dans différentes clauses : `SELECT`, `FROM`, `WHERE`, `HAVING`, etc.

### Exemple simple

```sql
-- Trouver les employés qui gagnent plus que le salaire moyen
SELECT nom, salaire
FROM employes
WHERE salaire > (SELECT AVG(salaire) FROM employes);
```

Dans cet exemple, `(SELECT AVG(salaire) FROM employes)` est une sous-requête qui calcule le salaire moyen, et cette valeur est ensuite utilisée dans la condition `WHERE`.

---

## Les trois types de sous-requêtes

### 1. Sous-requêtes Scalaires

Une sous-requête scalaire retourne **une seule valeur** (une ligne, une colonne). C'est le type le plus simple et le plus facile à comprendre.

#### Caractéristiques

- Retourne exactement **1 ligne** et **1 colonne**
- Peut être utilisée partout où une valeur unique est attendue
- Se comporte comme une constante dans la requête principale

#### Exemples d'utilisation

**Dans la clause WHERE :**

```sql
-- Trouver les produits plus chers que le produit le moins cher de la catégorie 'Électronique'
SELECT nom, prix
FROM produits
WHERE prix > (
    SELECT MIN(prix)
    FROM produits
    WHERE categorie = 'Électronique'
);
```

**Dans la clause SELECT :**

```sql
-- Afficher chaque employé avec l'écart par rapport au salaire moyen
SELECT
    nom,
    salaire,
    salaire - (SELECT AVG(salaire) FROM employes) AS ecart_moyenne
FROM employes;
```

**Dans la clause HAVING :**

```sql
-- Trouver les départements dont la masse salariale dépasse la moyenne
SELECT
    departement,
    SUM(salaire) AS masse_salariale
FROM employes
GROUP BY departement
HAVING SUM(salaire) > (
    SELECT AVG(total)
    FROM (
        SELECT SUM(salaire) AS total
        FROM employes
        GROUP BY departement
    ) AS dept_totals
);
```

#### ⚠️ Attention : Sous-requête scalaire et cardinalité

Si votre sous-requête retourne plus d'une ligne, PostgreSQL génèrera une erreur :

```sql
-- ❌ ERREUR : Cette sous-requête retourne plusieurs lignes
SELECT nom
FROM employes
WHERE salaire = (SELECT salaire FROM employes WHERE departement = 'IT');
```

**Solution :** Utilisez des opérateurs adaptés comme `IN`, `ANY`, `ALL` pour les résultats multi-lignes.

---

### 2. Sous-requêtes Vectorielles (ou de Ligne)

Une sous-requête vectorielle retourne **plusieurs lignes** mais **une seule colonne** (un vecteur de valeurs). Elle est souvent utilisée avec les opérateurs `IN`, `ANY`, `ALL`, `EXISTS`.

#### Caractéristiques

- Retourne **n lignes** et **1 colonne**
- Utilisée principalement avec des opérateurs de comparaison d'ensemble
- Très courante dans les requêtes de filtrage

#### Opérateurs associés

##### `IN` : Appartenance à un ensemble

```sql
-- Trouver les employés travaillant dans des départements situés à Paris
SELECT nom, departement
FROM employes
WHERE departement IN (
    SELECT nom_departement
    FROM departements
    WHERE ville = 'Paris'
);
```

##### `NOT IN` : Non-appartenance à un ensemble

```sql
-- Trouver les produits qui n'ont jamais été commandés
SELECT nom
FROM produits
WHERE id NOT IN (
    SELECT produit_id
    FROM commandes
);
```

**⚠️ Piège avec NULL :** `NOT IN` peut produire des résultats inattendus si la sous-requête contient des `NULL`. Préférez `NOT EXISTS` dans ce cas.

##### `ANY` / `SOME` : Comparaison avec au moins une valeur

```sql
-- Trouver les employés qui gagnent plus que n'importe quel employé du département 'Support'
SELECT nom, salaire
FROM employes
WHERE salaire > ANY (
    SELECT salaire
    FROM employes
    WHERE departement = 'Support'
);
```

Équivalent à : "plus que le minimum"

##### `ALL` : Comparaison avec toutes les valeurs

```sql
-- Trouver les employés qui gagnent plus que tous les employés du département 'Support'
SELECT nom, salaire
FROM employes
WHERE salaire > ALL (
    SELECT salaire
    FROM employes
    WHERE departement = 'Support'
);
```

Équivalent à : "plus que le maximum"

##### `EXISTS` / `NOT EXISTS` : Test d'existence

```sql
-- Trouver les clients qui ont passé au moins une commande
SELECT nom
FROM clients c
WHERE EXISTS (
    SELECT 1
    FROM commandes
    WHERE client_id = c.id
);
```

**💡 Astuce :** `EXISTS` ne retourne que `TRUE` ou `FALSE`, donc `SELECT 1`, `SELECT *` ou `SELECT NULL` ont la même performance. Par convention, on utilise souvent `SELECT 1`.

##### Différence entre `IN` et `EXISTS`

```sql
-- Avec IN (charge tous les IDs en mémoire)
SELECT nom
FROM clients
WHERE id IN (SELECT client_id FROM commandes);

-- Avec EXISTS (teste l'existence ligne par ligne, souvent plus performant)
SELECT nom
FROM clients c
WHERE EXISTS (SELECT 1 FROM commandes WHERE client_id = c.id);
```

**Règle générale :**
- `EXISTS` est souvent plus performant pour les grandes tables
- `IN` est plus lisible pour les petits ensembles
- `EXISTS` gère mieux les `NULL`

---

### 3. Sous-requêtes de Table (ou Dérivées)

Une sous-requête de table retourne **plusieurs lignes et plusieurs colonnes**, créant une table temporaire qui peut être utilisée comme source de données dans la clause `FROM`.

#### Caractéristiques

- Retourne **n lignes** et **m colonnes** (une table complète)
- Doit avoir un alias (obligatoire en SQL)
- Se comporte comme une table virtuelle dans la requête principale

#### Syntaxe de base

```sql
SELECT colonnes
FROM (
    SELECT ... FROM ... WHERE ...
) AS alias_obligatoire
WHERE conditions;
```

#### Exemples d'utilisation

**Agrégation sur agrégation :**

```sql
-- Calculer la moyenne des masses salariales par département
SELECT AVG(masse_salariale) AS moyenne_masse_salariale
FROM (
    SELECT departement, SUM(salaire) AS masse_salariale
    FROM employes
    GROUP BY departement
) AS dept_salaires;
```

**Filtrage après agrégation :**

```sql
-- Trouver les catégories de produits avec un prix moyen > 100
SELECT categorie, prix_moyen
FROM (
    SELECT
        categorie,
        AVG(prix) AS prix_moyen
    FROM produits
    GROUP BY categorie
) AS cat_stats
WHERE prix_moyen > 100;
```

**Jointures sur des résultats agrégés :**

```sql
-- Joindre les informations employés avec la moyenne de leur département
SELECT
    e.nom,
    e.salaire,
    d.salaire_moyen_dept,
    e.salaire - d.salaire_moyen_dept AS ecart
FROM employes e
INNER JOIN (
    SELECT
        departement,
        AVG(salaire) AS salaire_moyen_dept
    FROM employes
    GROUP BY departement
) AS d ON e.departement = d.departement;
```

**Pagination complexe (Top N par groupe) :**

```sql
-- Obtenir les 3 employés les mieux payés de chaque département
SELECT *
FROM (
    SELECT
        nom,
        departement,
        salaire,
        ROW_NUMBER() OVER (PARTITION BY departement ORDER BY salaire DESC) AS rang
    FROM employes
) AS ranked_employees
WHERE rang <= 3;
```

---

## Sous-requêtes Corrélées vs Non-Corrélées

### Sous-requêtes Non-Corrélées

Une sous-requête non-corrélée est **indépendante** de la requête externe. Elle est exécutée **une seule fois** et son résultat est réutilisé.

```sql
-- La sous-requête est exécutée 1 fois
SELECT nom, salaire
FROM employes
WHERE salaire > (SELECT AVG(salaire) FROM employes);
```

**Avantages :**
- Plus performante (exécutée une seule fois)
- Plus facile à comprendre et déboguer
- Peut être testée indépendamment

---

### Sous-requêtes Corrélées

Une sous-requête corrélée **dépend** de la requête externe. Elle référence une colonne de la table externe et est potentiellement exécutée **pour chaque ligne** de la requête principale.

```sql
-- La sous-requête est exécutée pour chaque employé
SELECT e1.nom, e1.salaire
FROM employes e1
WHERE e1.salaire > (
    SELECT AVG(e2.salaire)
    FROM employes e2
    WHERE e2.departement = e1.departement  -- ← Corrélation
);
```

**Caractéristiques :**
- Référence une colonne de la requête externe (`e1.departement`)
- Potentiellement exécutée N fois (une fois par ligne)
- Plus difficile à optimiser pour le SGBD

**Exemple avec EXISTS (très courant) :**

```sql
-- Trouver les clients qui ont commandé au moins un produit de catégorie 'Électronique'
SELECT c.nom
FROM clients c
WHERE EXISTS (
    SELECT 1
    FROM commandes co
    INNER JOIN produits p ON co.produit_id = p.id
    WHERE co.client_id = c.id  -- ← Corrélation
      AND p.categorie = 'Électronique'
);
```

---

## Performance des Sous-requêtes

### Facteurs influençant la performance

#### 1. Type de sous-requête

| Type | Performance | Remarque |
|------|-------------|----------|
| Scalaire non-corrélée | ⭐⭐⭐⭐⭐ Excellente | Exécutée 1 fois |
| Scalaire corrélée | ⭐⭐ Médiocre | Exécutée N fois |
| Vectorielle (IN) | ⭐⭐⭐ Bonne | Optimisable en semi-join |
| Vectorielle (EXISTS) | ⭐⭐⭐⭐ Très bonne | Souvent optimal |
| Table (FROM) | ⭐⭐⭐⭐ Très bonne | Comme une CTE inline |

#### 2. Corrélation

Les sous-requêtes corrélées sont plus coûteuses car elles peuvent être exécutées de nombreuses fois. PostgreSQL essaie de les optimiser mais ce n'est pas toujours possible.

**Exemple de problème :**

```sql
-- ❌ Sous-requête corrélée potentiellement lente
SELECT
    p.nom,
    (SELECT COUNT(*) FROM commandes WHERE produit_id = p.id) AS nb_commandes
FROM produits p;
```

**Solution optimisée avec jointure :**

```sql
-- ✅ Jointure plus performante
SELECT
    p.nom,
    COALESCE(c.nb_commandes, 0) AS nb_commandes
FROM produits p
LEFT JOIN (
    SELECT produit_id, COUNT(*) AS nb_commandes
    FROM commandes
    GROUP BY produit_id
) c ON p.id = c.produit_id;
```

#### 3. Indexation

Les sous-requêtes bénéficient grandement des index sur les colonnes utilisées dans les conditions de jointure ou de filtrage.

```sql
-- Cette requête sera rapide si un index existe sur commandes.client_id
SELECT nom
FROM clients c
WHERE EXISTS (
    SELECT 1 FROM commandes WHERE client_id = c.id
);
```

**Recommandation :** Créez des index sur les colonnes de corrélation.

---

### Comparaison : Sous-requête vs Jointure

Souvent, une sous-requête peut être réécrite avec une jointure. Voici les considérations :

#### Quand préférer une sous-requête

1. **Clarté du code :** Logique plus évidente
2. **Test d'existence :** `EXISTS` est explicite et optimisé
3. **Agrégation dans SELECT :** Calcul par ligne

```sql
-- Plus lisible avec EXISTS
SELECT nom
FROM clients c
WHERE EXISTS (SELECT 1 FROM commandes WHERE client_id = c.id);
```

#### Quand préférer une jointure

1. **Performance :** Souvent plus rapide (une seule passe)
2. **Récupération de colonnes multiples :** Plus naturel
3. **Agrégations globales :** Moins de sous-requêtes imbriquées

```sql
-- Plus performant avec JOIN
SELECT c.nom, COUNT(co.id) AS nb_commandes
FROM clients c
LEFT JOIN commandes co ON c.id = co.client_id
GROUP BY c.id, c.nom;
```

---

### Optimisations PostgreSQL

PostgreSQL possède un optimiseur de requêtes sophistiqué qui peut transformer automatiquement certaines sous-requêtes :

#### 1. Semi-Join et Anti-Join

PostgreSQL peut convertir `IN` et `EXISTS` en semi-jointures efficaces :

```sql
-- Ces deux requêtes peuvent produire le même plan d'exécution
SELECT * FROM clients WHERE id IN (SELECT client_id FROM commandes);
SELECT * FROM clients c WHERE EXISTS (SELECT 1 FROM commandes WHERE client_id = c.id);
```

#### 2. Subquery Pull-Up

Certaines sous-requêtes dans `FROM` sont "remontées" et fusionnées avec la requête principale :

```sql
-- Peut être optimisée en une seule requête par PostgreSQL
SELECT nom
FROM (
    SELECT nom, salaire FROM employes WHERE actif = TRUE
) AS e
WHERE salaire > 50000;
```

#### 3. Matérialisation

Pour les sous-requêtes coûteuses utilisées plusieurs fois, PostgreSQL peut les matérialiser (calculer une fois et stocker temporairement).

---

### Comment analyser la performance

Utilisez `EXPLAIN ANALYZE` pour comprendre comment PostgreSQL exécute vos sous-requêtes :

```sql
EXPLAIN ANALYZE
SELECT nom
FROM clients c
WHERE EXISTS (
    SELECT 1 FROM commandes WHERE client_id = c.id
);
```

**Éléments à surveiller :**
- **Nested Loop** : Peut indiquer une corrélation ligne par ligne
- **Hash Semi Join** / **Merge Semi Join** : Optimisations efficaces
- **SubPlan** : Sous-requête corrélée (attention si coût élevé)
- **InitPlan** : Sous-requête exécutée une fois avant la requête principale (bon signe)

---

## Bonnes Pratiques

### ✅ À faire

1. **Privilégier `EXISTS` pour les tests d'existence**
   ```sql
   -- Préférer ceci
   WHERE EXISTS (SELECT 1 FROM ...)
   -- À cela
   WHERE id IN (SELECT id FROM ...)
   ```

2. **Éviter les sous-requêtes corrélées dans SELECT**
   ```sql
   -- ❌ Éviter
   SELECT nom, (SELECT COUNT(*) FROM commandes WHERE client_id = c.id)
   FROM clients c;

   -- ✅ Préférer
   SELECT c.nom, COUNT(co.id)
   FROM clients c
   LEFT JOIN commandes co ON c.id = co.client_id
   GROUP BY c.id, c.nom;
   ```

3. **Indexer les colonnes de corrélation**

4. **Tester avec `EXPLAIN ANALYZE`**

5. **Utiliser des CTEs pour la lisibilité des sous-requêtes complexes**
   ```sql
   WITH salaires_dept AS (
       SELECT departement, AVG(salaire) AS moyenne
       FROM employes
       GROUP BY departement
   )
   SELECT e.nom, e.salaire
   FROM employes e
   INNER JOIN salaires_dept sd ON e.departement = sd.departement
   WHERE e.salaire > sd.moyenne;
   ```

### ❌ À éviter

1. **NOT IN avec possibilité de NULL**
   ```sql
   -- ❌ Peut retourner 0 résultat si la sous-requête contient NULL
   WHERE id NOT IN (SELECT client_id FROM commandes)

   -- ✅ Utiliser NOT EXISTS à la place
   WHERE NOT EXISTS (SELECT 1 FROM commandes WHERE client_id = clients.id)
   ```

2. **Sous-requêtes scalaires corrélées répétées**

3. **Sous-requêtes non nécessaires**
   ```sql
   -- ❌ Inutile
   SELECT * FROM (SELECT * FROM employes) AS e;

   -- ✅ Direct
   SELECT * FROM employes;
   ```

4. **Oublier les alias pour les sous-requêtes dans FROM**

---

## Cas d'Usage Courants

### 1. Top N par groupe

```sql
-- Les 2 produits les plus vendus par catégorie
SELECT categorie, nom, ventes
FROM (
    SELECT
        categorie,
        nom,
        ventes,
        ROW_NUMBER() OVER (PARTITION BY categorie ORDER BY ventes DESC) AS rang
    FROM produits
) AS ranked
WHERE rang <= 2;
```

### 2. Comparaison avec agrégat de groupe

```sql
-- Employés gagnant plus que la moyenne de leur département
SELECT e.nom, e.salaire, e.departement
FROM employes e
WHERE e.salaire > (
    SELECT AVG(salaire)
    FROM employes
    WHERE departement = e.departement
);
```

### 3. Élimination de doublons complexes

```sql
-- Garder uniquement le dernier enregistrement pour chaque client
SELECT *
FROM commandes c1
WHERE c1.date_commande = (
    SELECT MAX(date_commande)
    FROM commandes c2
    WHERE c2.client_id = c1.client_id
);
```

### 4. Filtrage sur agrégats

```sql
-- Départements avec plus de 10 employés
SELECT nom
FROM departements
WHERE id IN (
    SELECT departement_id
    FROM employes
    GROUP BY departement_id
    HAVING COUNT(*) > 10
);
```

---

## Résumé

| Type | Retour | Utilisation | Performance |
|------|--------|-------------|-------------|
| **Scalaire** | 1 ligne, 1 colonne | WHERE, SELECT, HAVING | Excellente si non-corrélée |
| **Vectorielle** | N lignes, 1 colonne | IN, ANY, ALL, EXISTS | Bonne (EXISTS optimal) |
| **Table** | N lignes, M colonnes | FROM | Très bonne |

**Points clés à retenir :**

1. Les sous-requêtes permettent de décomposer des problèmes complexes
2. `EXISTS` est généralement plus performant que `IN` pour les grandes tables
3. Évitez les sous-requêtes corrélées dans `SELECT` si possible
4. Les index sur les colonnes de corrélation sont cruciaux
5. Utilisez `EXPLAIN ANALYZE` pour valider vos optimisations
6. Parfois, une jointure est plus performante qu'une sous-requête
7. PostgreSQL optimise intelligemment de nombreuses sous-requêtes automatiquement

---

## Pour aller plus loin

- **CTEs (Common Table Expressions)** : Alternative souvent plus lisible aux sous-requêtes complexes
- **Window Functions** : Pour éviter certaines sous-requêtes corrélées
- **Lateral Joins** : Pour corréler efficacement des sous-requêtes de table
- **Subquery Pull-Up** : Détails internes de l'optimiseur PostgreSQL

---

**Prochain chapitre :** 9.2. CTE (Common Table Expressions) et l'option MATERIALIZED

---


⏭️ [CTE (Common Table Expressions) et l'option MATERIALIZED](/09-techniques-sql-avancees/02-cte-common-table-expressions.md)
