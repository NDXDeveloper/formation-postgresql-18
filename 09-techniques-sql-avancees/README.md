🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9. Techniques SQL Avancées

## Introduction au chapitre

Félicitations pour être arrivé jusqu'ici ! Vous maîtrisez maintenant les fondamentaux de SQL : vous savez créer des tables, insérer des données, effectuer des requêtes simples, utiliser les jointures et les agrégations. Vous êtes prêt à franchir une nouvelle étape : **les techniques SQL avancées**.

Ce chapitre marque une transition importante dans votre parcours d'apprentissage. Vous allez découvrir des outils puissants qui vous permettront d'exprimer des logiques complexes de manière élégante et performante. Ces techniques sont utilisées quotidiennement par les développeurs et les analystes de données professionnels.

---

## Pourquoi apprendre les techniques SQL avancées ?

### 1. Résoudre des problèmes complexes

Les techniques de base de SQL permettent de répondre à des questions simples :
- "Quels sont tous les clients de Paris ?"
- "Quel est le total des ventes par produit ?"
- "Combien d'employés travaillent dans le département IT ?"

Mais que faire lorsque les questions deviennent plus complexes ?
- "Quels sont les clients dont les achats totaux dépassent la moyenne de leur région ?"
- "Comment afficher tous les employés avec leur manager, le manager de leur manager, et ainsi de suite ?"
- "Comment identifier les produits qui se vendent mieux que tous les autres produits de leur catégorie ?"

**Les techniques SQL avancées vous permettent de répondre à ces questions** sans avoir à écrire des centaines de lignes de code applicatif.

### 2. Améliorer les performances

```sql
-- ❌ Approche naïve : 1000 requêtes dans une boucle applicative
for client in clients:
    orders = execute_query("SELECT * FROM orders WHERE client_id = ?", client.id)
    # Traiter les commandes...

-- ✅ Approche avancée : 1 requête SQL bien conçue
SELECT
    c.*,
    o.*
FROM clients c
LEFT JOIN orders o ON c.id = o.client_id
WHERE c.statut = 'actif';
```

Une requête SQL avancée bien conçue peut remplacer des dizaines ou des centaines de requêtes simples, réduisant drastiquement le temps d'exécution de votre application.

### 3. Écrire du code plus maintenable

```sql
-- Au lieu d'une logique dispersée dans le code applicatif...
WITH clients_actifs AS (
    SELECT * FROM clients WHERE statut = 'actif'
),
commandes_recentes AS (
    SELECT * FROM commandes WHERE date_commande > CURRENT_DATE - INTERVAL '30 days'
)
SELECT
    ca.nom,
    COUNT(cr.id) AS nb_commandes
FROM clients_actifs ca
LEFT JOIN commandes_recentes cr ON ca.id = cr.client_id
GROUP BY ca.id, ca.nom;
```

Les techniques avancées permettent d'exprimer la logique métier **directement dans SQL**, de manière claire et déclarative.

### 4. Exploiter la puissance de PostgreSQL

PostgreSQL est l'un des SGBD les plus riches en fonctionnalités. Ne pas utiliser ses capacités avancées, c'est comme n'utiliser qu'un smartphone pour passer des appels : vous passez à côté de 90% de sa puissance !

---

## Ce que vous allez apprendre

Ce chapitre couvre **six techniques SQL avancées** essentielles. Chacune résout des types de problèmes spécifiques et possède ses propres cas d'usage.

### Vue d'ensemble des techniques

```
┌─────────────────────────────────────────────────────────┐
│         Techniques SQL Avancées (Chapitre 9)            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  9.1  Sous-requêtes                                     │
│       ├─ Scalaires : une seule valeur                   │
│       ├─ Vectorielles : plusieurs lignes                │
│       └─ Tables : résultats multi-colonnes              │
│                                                         │
│  9.2  CTE (Common Table Expressions)                    │
│       └─ WITH ... AS : requêtes nommées et lisibles     │
│                                                         │
│  9.3  CTE Récursives                                    │
│       └─ Parcourir des hiérarchies et graphes           │
│                                                         │
│  9.4  Opérations d'ensemble                             │
│       ├─ UNION : combiner des résultats                 │
│       ├─ INTERSECT : trouver les éléments communs       │
│       └─ EXCEPT : calculer les différences              │
│                                                         │
│  9.5  CASE : Logique conditionnelle                     │
│       └─ IF/ELSE directement dans SQL                   │
│                                                         │
│  9.6  Optimisation OR → ANY (PostgreSQL 18)             │
│       └─ Performances automatiquement améliorées        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Progression pédagogique

### Niveau de difficulté

Les six techniques sont présentées par difficulté croissante :

| Technique | Difficulté | Prérequis | Utilisation |
|-----------|------------|-----------|-------------|
| **CASE** | ⭐ Facile | SELECT, WHERE | Très fréquent |
| **Opérations d'ensemble** | ⭐⭐ Moyen | SELECT de base | Fréquent |
| **Sous-requêtes** | ⭐⭐ Moyen | SELECT, WHERE | Très fréquent |
| **CTE** | ⭐⭐⭐ Avancé | Sous-requêtes | Fréquent |
| **Optimisation OR→ANY** | ⭐⭐⭐ Avancé | WHERE, index | Automatique (PG 18) |
| **CTE Récursives** | ⭐⭐⭐⭐ Expert | CTE | Moins fréquent |

**Recommandation :** Suivez l'ordre des chapitres, chaque technique s'appuyant sur les précédentes.

### Temps d'apprentissage estimé

- **Lecture complète du chapitre** : 8-10 heures
- **Compréhension avec expérimentation** : 15-20 heures
- **Maîtrise pratique** : 40-60 heures (avec pratique régulière)

---

## Les six techniques en un coup d'œil

### 9.1. Sous-requêtes

**Concept :** Une requête SQL imbriquée dans une autre requête.

**Exemple simple :**
```sql
-- Trouver les employés qui gagnent plus que la moyenne
SELECT nom, salaire
FROM employes
WHERE salaire > (SELECT AVG(salaire) FROM employes);
                 └─────── sous-requête ──────┘
```

**Quand l'utiliser :**
- Filtrer sur un résultat calculé
- Comparer à des agrégats
- Décomposer des requêtes complexes

---

### 9.2. CTE (Common Table Expressions)

**Concept :** Créer des résultats intermédiaires nommés et réutilisables avec `WITH`.

**Exemple simple :**
```sql
-- Décomposer une requête complexe en étapes claires
WITH salaires_moyens AS (
    SELECT departement, AVG(salaire) AS moyenne
    FROM employes
    GROUP BY departement
)
SELECT e.nom, e.salaire, sm.moyenne
FROM employes e
JOIN salaires_moyens sm ON e.departement = sm.departement;
```

**Quand l'utiliser :**
- Améliorer la lisibilité
- Éviter la répétition
- Structurer des requêtes complexes

---

### 9.3. CTE Récursives

**Concept :** Une CTE qui se référence elle-même pour parcourir des structures hiérarchiques.

**Exemple simple :**
```sql
-- Parcourir un organigramme d'entreprise
WITH RECURSIVE hierarchie AS (
    -- Point de départ : le PDG
    SELECT id, nom, manager_id, 1 AS niveau
    FROM employes WHERE manager_id IS NULL

    UNION ALL

    -- Récursion : les subordonnés
    SELECT e.id, e.nom, e.manager_id, h.niveau + 1
    FROM employes e
    JOIN hierarchie h ON e.manager_id = h.id
)
SELECT * FROM hierarchie ORDER BY niveau;
```

**Quand l'utiliser :**
- Organigrammes et hiérarchies
- Arbres de catégories
- Réseaux et graphes
- Bill of Materials (BOM)

---

### 9.4. Opérations d'ensemble

**Concept :** Combiner les résultats de plusieurs requêtes avec des opérations mathématiques d'ensembles.

**Exemple simple :**
```sql
-- Combiner deux listes
SELECT nom FROM clients
UNION
SELECT nom FROM fournisseurs;

-- Trouver les éléments communs
SELECT email FROM clients
INTERSECT
SELECT email FROM newsletter_subscribers;

-- Calculer la différence
SELECT email FROM clients
EXCEPT
SELECT email FROM liste_bannis;
```

**Quand l'utiliser :**
- Fusionner des données de sources multiples
- Identifier les éléments communs
- Calculer des différences

---

### 9.5. CASE : Logique conditionnelle

**Concept :** L'équivalent SQL du `if/else` des langages de programmation.

**Exemple simple :**
```sql
-- Catégoriser dynamiquement
SELECT
    nom,
    salaire,
    CASE
        WHEN salaire < 40000 THEN 'Junior'
        WHEN salaire < 70000 THEN 'Intermédiaire'
        ELSE 'Senior'
    END AS niveau
FROM employes;
```

**Quand l'utiliser :**
- Catégorisation dynamique
- Calculs conditionnels
- Transformation de données
- Pivot de données

---

### 9.6. Optimisation OR → ANY (PostgreSQL 18)

**Concept :** PostgreSQL 18 transforme automatiquement les conditions `OR` multiples en opérateur `ANY` pour de meilleures performances.

**Exemple :**
```sql
-- Votre code (inchangé)
SELECT * FROM produits
WHERE categorie = 'A' OR categorie = 'B' OR categorie = 'C';

-- PostgreSQL 18 optimise automatiquement en :
SELECT * FROM produits
WHERE categorie = ANY(ARRAY['A', 'B', 'C']);
```

**Bénéfice :**
- 2× à 8× plus rapide automatiquement
- Aucun changement de code requis
- Utilisation optimale des index

---

## Comment aborder ce chapitre

### Pour les débutants

Si vous découvrez SQL :
1. ✅ Commencez par **CASE** (9.5) - le plus accessible
2. ✅ Puis **Opérations d'ensemble** (9.4) - concepts mathématiques simples
3. ✅ Ensuite **Sous-requêtes** (9.1) - fondation essentielle
4. ✅ Suivez avec **CTE** (9.2) - améliore la lisibilité
5. ⚠️ Abordez **CTE Récursives** (9.3) après avoir bien compris les CTE
6. ℹ️ Lisez **OR→ANY** (9.6) pour comprendre les optimisations PG 18

### Pour les développeurs expérimentés

Si vous connaissez déjà SQL :
1. Parcourez rapidement les sections que vous maîtrisez
2. Concentrez-vous sur les **nouveautés PostgreSQL 18**
3. Approfondissez les **aspects performance**
4. Explorez les **cas d'usage avancés**

### Conseils d'apprentissage

#### 1. Expérimentez avec des données

Créez des tables de test simples :

```sql
-- Table d'exemple simple
CREATE TABLE employes_test (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement VARCHAR(50),
    salaire NUMERIC(10, 2),
    manager_id INTEGER REFERENCES employes_test(id)
);

-- Insérer quelques données
INSERT INTO employes_test (nom, departement, salaire, manager_id) VALUES
    ('Alice', 'IT', 75000, NULL),
    ('Bob', 'IT', 55000, 1),
    ('Charlie', 'Sales', 65000, NULL),
    ('Diana', 'Sales', 45000, 3);
```

#### 2. Utilisez EXPLAIN pour comprendre

```sql
-- Voir comment PostgreSQL exécute votre requête
EXPLAIN ANALYZE
SELECT ...;
```

#### 3. Commencez simple, puis complexifiez

```sql
-- Étape 1 : Version simple
SELECT nom FROM employes;

-- Étape 2 : Ajouter une sous-requête
SELECT nom FROM employes
WHERE salaire > (SELECT AVG(salaire) FROM employes);

-- Étape 3 : Convertir en CTE pour la lisibilité
WITH salaire_moyen AS (
    SELECT AVG(salaire) AS moyenne FROM employes
)
SELECT nom FROM employes, salaire_moyen
WHERE salaire > moyenne;
```

#### 4. Comparez les approches

Souvent, plusieurs techniques peuvent résoudre le même problème. Comparez-les !

```sql
-- Approche 1 : Sous-requête
SELECT * FROM employes
WHERE departement IN (SELECT DISTINCT departement FROM managers);

-- Approche 2 : JOIN
SELECT DISTINCT e.*
FROM employes e
JOIN managers m ON e.departement = m.departement;

-- Approche 3 : INTERSECT
SELECT departement FROM employes
INTERSECT
SELECT departement FROM managers;
```

---

## Liens avec les autres chapitres

### Prérequis (déjà vus)

- **Chapitre 5** : Requêtes de sélection (DQL)
- **Chapitre 6** : Manipulation de données (DML)
- **Chapitre 7** : Relations et jointures
- **Chapitre 8** : Agrégation et groupement

### Préparation aux chapitres suivants

Les techniques de ce chapitre sont essentielles pour :
- **Chapitre 10** : Fonctions de fenêtrage (Window Functions)
- **Chapitre 11** : Modélisation avancée
- **Chapitre 13** : Optimisation et indexation

---

## Terminologie importante

Avant de commencer, familiarisez-vous avec ces termes :

| Terme | Définition | Exemple |
|-------|------------|---------|
| **Sous-requête** | Requête imbriquée dans une autre | `WHERE x > (SELECT ...)` |
| **CTE** | Résultat intermédiaire nommé | `WITH temp AS (...)` |
| **Récursion** | Processus qui se référence lui-même | CTE qui joint sur elle-même |
| **Ensemble** | Collection de valeurs uniques | {1, 2, 3} |
| **Union** | Combiner deux ensembles | A ∪ B |
| **Intersection** | Éléments communs | A ∩ B |
| **Différence** | Éléments dans A mais pas B | A − B |
| **Expression** | Calcul qui retourne une valeur | `CASE WHEN ... END` |

---

## Philosophie de PostgreSQL

PostgreSQL adopte une approche **déclarative** du traitement des données :

### Impératif vs Déclaratif

**Approche impérative** (code applicatif) :
```python
# Dire COMMENT faire
results = []
for employee in employees:
    if employee.salary > average_salary:
        results.append(employee)
```

**Approche déclarative** (SQL) :
```sql
-- Dire CE QUE l'on veut
SELECT * FROM employes
WHERE salaire > (SELECT AVG(salaire) FROM employes);
```

**Avantages du déclaratif :**
- ✅ Plus concis
- ✅ Optimisé automatiquement par PostgreSQL
- ✅ Fonctionne sur des millions de lignes
- ✅ Indépendant du langage de programmation

---

## Objectifs d'apprentissage

À la fin de ce chapitre, vous serez capable de :

### Compétences techniques

- ✅ Écrire et comprendre des sous-requêtes de tous types
- ✅ Utiliser les CTE pour structurer vos requêtes
- ✅ Parcourir des hiérarchies avec des CTE récursives
- ✅ Combiner des résultats avec UNION, INTERSECT, EXCEPT
- ✅ Implémenter une logique conditionnelle avec CASE
- ✅ Comprendre les optimisations automatiques de PostgreSQL 18

### Compétences conceptuelles

- ✅ Identifier quelle technique utiliser selon le problème
- ✅ Évaluer la performance de différentes approches
- ✅ Lire et comprendre du SQL avancé écrit par d'autres
- ✅ Décomposer un problème complexe en requêtes SQL

### Compétences pratiques

- ✅ Optimiser des requêtes lentes
- ✅ Rendre le code SQL plus maintenable
- ✅ Résoudre des problèmes métier complexes en SQL
- ✅ Utiliser EXPLAIN pour analyser les performances

---

## Structure des sous-chapitres

Chaque technique est présentée selon le même format :

1. **Introduction** : Qu'est-ce que c'est et pourquoi c'est utile
2. **Syntaxe** : Comment l'écrire
3. **Exemples progressifs** : Du simple au complexe
4. **Cas d'usage pratiques** : Situations réelles
5. **Performance** : Comprendre l'impact
6. **Pièges courants** : Erreurs à éviter
7. **Bonnes pratiques** : Recommandations
8. **Résumé** : Points clés à retenir

---

## Prêt à commencer ?

Vous avez maintenant une vue d'ensemble complète de ce qui vous attend dans ce chapitre. Les techniques SQL avancées vont transformer votre façon d'écrire des requêtes et d'interagir avec les bases de données.

**Conseil :** Ne cherchez pas à tout maîtriser immédiatement. Chaque technique nécessite de la pratique. Avancez à votre rythme, expérimentez, et n'hésitez pas à revenir sur les chapitres précédents si nécessaire.

### Points importants à garder en tête

1. 🎯 **L'objectif n'est pas d'utiliser toutes les techniques partout**, mais de choisir la bonne technique pour chaque problème

2. 📊 **La performance compte**, mais la lisibilité aussi - un code maintenable est souvent plus important qu'un gain de 10ms

3. 🔄 **Plusieurs solutions existent** pour le même problème - apprenez à les comparer

4. 🛠️ **PostgreSQL fait beaucoup d'optimisations automatiquement** - concentrez-vous sur la clarté du code

5. 📚 **Ces techniques sont universelles** - elles fonctionnent dans la plupart des SGBD relationnels (avec quelques variations)

---

## Ressources complémentaires

### Documentation PostgreSQL officielle

- [Queries](https://www.postgresql.org/docs/current/queries.html)
- [SELECT](https://www.postgresql.org/docs/current/sql-select.html)
- [WITH Queries (CTE)](https://www.postgresql.org/docs/current/queries-with.html)

### Outils utiles pour l'apprentissage

- **psql** : Client en ligne de commande PostgreSQL
- **pgAdmin** : Interface graphique pour PostgreSQL
- **DBeaver** : Client universel multi-bases
- **EXPLAIN Visualizer** : Comprendre les plans d'exécution

### Environnement de test recommandé

```sql
-- Créer une base de données de test
CREATE DATABASE sql_avance_test;

-- Se connecter
\c sql_avance_test

-- Créer des tables d'exemple
-- (voir les exemples dans chaque sous-chapitre)
```

---

## Prochaine étape

Vous êtes maintenant prêt à plonger dans la première technique : **les sous-requêtes** !

👉 **Continuez avec le Chapitre 9.1 : Sous-requêtes (scalaires, vectorielles, table) et performance**

---


⏭️ [Sous-requêtes (scalaires, vectorielles, table) et performance](/09-techniques-sql-avancees/01-sous-requetes.md)
