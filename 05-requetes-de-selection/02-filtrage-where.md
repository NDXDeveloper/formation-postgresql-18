🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2. Filtrage (WHERE) et logique booléenne

## Introduction

Dans le chapitre précédent, nous avons vu que la clause `WHERE` est exécutée en **deuxième position** dans l'ordre logique d'une requête SQL, juste après le `FROM`. Son rôle est crucial : **filtrer les lignes** pour ne conserver que celles qui satisfont certaines conditions.

Sans filtrage, une requête retourne **toutes** les lignes d'une table. Avec `WHERE`, vous pouvez être précis et sélectif, ce qui est essentiel pour :

- Réduire le volume de données retournées
- Améliorer les performances (moins de données à traiter)
- Obtenir exactement l'information recherchée
- Sécuriser l'accès aux données sensibles

Dans ce chapitre, nous allons explorer en profondeur la clause `WHERE` et la logique booléenne qui la sous-tend.

---

## La clause WHERE : Fondamentaux

### Syntaxe de base

```sql
SELECT colonnes  
FROM table  
WHERE condition;  
```

La `condition` est une **expression booléenne** qui évalue à :
- `TRUE` (vrai) → la ligne est **conservée**  
- `FALSE` (faux) → la ligne est **éliminée**  
- `NULL` → la ligne est **éliminée** (comportement spécial, voir section sur NULL)

### Exemple simple

```sql
SELECT nom, prenom, salaire  
FROM employes  
WHERE salaire > 50000;  
```

**Ce qui se passe :**
1. PostgreSQL charge toutes les lignes de `employes`  
2. Pour chaque ligne, il évalue `salaire > 50000`  
3. Si le résultat est `TRUE`, la ligne est conservée  
4. Toutes les lignes conservées sont retournées avec les colonnes `nom`, `prenom`, `salaire`

---

## Les opérateurs de comparaison

PostgreSQL propose plusieurs opérateurs pour comparer des valeurs.

### Opérateurs numériques et de texte

| Opérateur | Signification | Exemple | Résultat |
|-----------|---------------|---------|----------|
| `=` | Égal à | `age = 30` | Lignes où age vaut exactement 30 |
| `!=` ou `<>` | Différent de | `statut != 'inactif'` | Lignes où statut n'est pas 'inactif' |
| `>` | Supérieur à | `salaire > 60000` | Lignes où salaire est strictement supérieur à 60000 |
| `<` | Inférieur à | `age < 25` | Lignes où age est strictement inférieur à 25 |
| `>=` | Supérieur ou égal | `annee >= 2020` | Lignes où annee est 2020 ou plus |
| `<=` | Inférieur ou égal | `note <= 10` | Lignes où note est 10 ou moins |

**Exemples concrets :**

```sql
-- Employés gagnant exactement 50000
SELECT nom FROM employes WHERE salaire = 50000;

-- Employés n'étant pas dans le département IT
SELECT nom FROM employes WHERE departement <> 'IT';

-- Produits coûtant plus de 100 euros
SELECT nom_produit, prix FROM produits WHERE prix > 100;

-- Clients ayant au moins 18 ans
SELECT nom, age FROM clients WHERE age >= 18;
```

### Comparaison de chaînes de caractères

PostgreSQL compare les chaînes **lexicographiquement** (ordre alphabétique) :

```sql
-- Noms commençant par A, B, C, D, E, F, G, H, I, J
SELECT nom FROM employes WHERE nom < 'K';

-- Noms après 'Martin' dans l'ordre alphabétique
SELECT nom FROM employes WHERE nom > 'Martin';
```

**⚠️ Sensibilité à la casse :**
Par défaut, PostgreSQL est **sensible à la casse** pour les comparaisons :

```sql
-- Ces deux requêtes donnent des résultats différents
WHERE nom = 'Dupont'   -- Trouve uniquement 'Dupont'  
WHERE nom = 'dupont'   -- Trouve uniquement 'dupont'  
```

Pour ignorer la casse, utilisez `ILIKE` ou la fonction `LOWER()` :

```sql
-- Insensible à la casse
WHERE LOWER(nom) = 'dupont'  -- Trouve 'Dupont', 'DUPONT', 'dupont', etc.
```

### Comparaison de dates

```sql
-- Employés embauchés après le 1er janvier 2020
SELECT nom, date_embauche  
FROM employes  
WHERE date_embauche > '2020-01-01';  

-- Commandes passées en 2024
SELECT * FROM commandes  
WHERE date_commande >= '2024-01-01'  
  AND date_commande < '2025-01-01';
```

---

## Logique booléenne : AND, OR, NOT

La puissance de `WHERE` réside dans sa capacité à **combiner plusieurs conditions** grâce aux opérateurs logiques.

### L'opérateur AND (ET logique)

`AND` signifie que **toutes les conditions** doivent être vraies.

**Syntaxe :**
```sql
WHERE condition1 AND condition2 AND condition3
```

**Table de vérité :**

| condition1 | condition2 | condition1 AND condition2 |
|------------|------------|---------------------------|
| TRUE       | TRUE       | **TRUE** ✓                |
| TRUE       | FALSE      | FALSE                     |
| FALSE      | TRUE       | FALSE                     |
| FALSE      | FALSE      | FALSE                     |

**Exemples :**

```sql
-- Employés du département IT gagnant plus de 50000
SELECT nom, departement, salaire  
FROM employes  
WHERE departement = 'IT' AND salaire > 50000;  

-- Produits en stock et bon marché
SELECT nom_produit, prix, quantite_stock  
FROM produits  
WHERE quantite_stock > 0 AND prix < 50;  

-- Clients majeurs habitant Paris
SELECT nom, age, ville  
FROM clients  
WHERE age >= 18 AND ville = 'Paris';  

-- Trois conditions en même temps
SELECT nom, salaire, departement, anciennete  
FROM employes  
WHERE salaire > 40000  
  AND departement = 'Ventes'
  AND anciennete >= 5;
```

**Principe :** Plus vous ajoutez de conditions avec `AND`, plus le filtrage est **restrictif** (moins de lignes retournées).

### L'opérateur OR (OU logique)

`OR` signifie qu'**au moins une condition** doit être vraie.

**Syntaxe :**
```sql
WHERE condition1 OR condition2 OR condition3
```

**Table de vérité :**

| condition1 | condition2 | condition1 OR condition2 |
|------------|------------|--------------------------|
| TRUE       | TRUE       | **TRUE** ✓               |
| TRUE       | FALSE      | **TRUE** ✓               |
| FALSE      | TRUE       | **TRUE** ✓               |
| FALSE      | FALSE      | FALSE                    |

**Exemples :**

```sql
-- Employés du département IT ou RH
SELECT nom, departement  
FROM employes  
WHERE departement = 'IT' OR departement = 'RH';  

-- Produits soit très chers soit en rupture de stock
SELECT nom_produit, prix, quantite_stock  
FROM produits  
WHERE prix > 1000 OR quantite_stock = 0;  

-- Commandes urgentes (priorité haute ou très haute)
SELECT numero_commande, priorite  
FROM commandes  
WHERE priorite = 'haute' OR priorite = 'très haute';  
```

**Principe :** Plus vous ajoutez de conditions avec `OR`, plus le filtrage est **permissif** (plus de lignes retournées).

### L'opérateur NOT (NON logique)

`NOT` **inverse** une condition : ce qui était vrai devient faux, et vice-versa.

**Syntaxe :**
```sql
WHERE NOT condition
```

**Table de vérité :**

| condition | NOT condition |
|-----------|---------------|
| TRUE      | **FALSE**     |
| FALSE     | **TRUE**      |
| NULL      | NULL          |

**Exemples :**

```sql
-- Employés qui ne sont PAS dans le département IT
SELECT nom, departement  
FROM employes  
WHERE NOT departement = 'IT';  

-- Équivalent (plus lisible)
SELECT nom, departement  
FROM employes  
WHERE departement != 'IT';  

-- Produits qui ne sont PAS en rupture de stock
SELECT nom_produit, quantite_stock  
FROM produits  
WHERE NOT quantite_stock = 0;  

-- Équivalent
SELECT nom_produit, quantite_stock  
FROM produits  
WHERE quantite_stock != 0;  

-- Utilisation avec des conditions complexes
SELECT nom, age, statut  
FROM clients  
WHERE NOT (age < 18 OR statut = 'bloqué');  
-- Signifie : ni mineur, ni bloqué
```

### Combinaison de AND, OR et NOT

Vous pouvez combiner ces opérateurs pour créer des conditions complexes :

```sql
-- Employés IT ou RH, mais seulement ceux gagnant plus de 50000
SELECT nom, departement, salaire  
FROM employes  
WHERE (departement = 'IT' OR departement = 'RH')  
  AND salaire > 50000;

-- Produits chers ou en rupture, mais pas les produits obsolètes
SELECT nom_produit, prix, quantite_stock, statut  
FROM produits  
WHERE (prix > 1000 OR quantite_stock = 0)  
  AND NOT statut = 'obsolète';

-- Clients parisiens majeurs ou lyonnais de tout âge
SELECT nom, ville, age  
FROM clients  
WHERE (ville = 'Paris' AND age >= 18)  
   OR ville = 'Lyon';
```

---

## Priorité des opérateurs : Les parenthèses sont essentielles

Les opérateurs logiques ont un **ordre de priorité** :

1. **NOT** (priorité la plus haute)  
2. **AND**  
3. **OR** (priorité la plus basse)

Cela peut créer des résultats inattendus si vous n'utilisez pas de parenthèses.

### Exemple problématique

```sql
-- SANS parenthèses (ambigu)
SELECT nom, departement, salaire  
FROM employes  
WHERE departement = 'IT' OR departement = 'RH' AND salaire > 50000;  
```

**Comment PostgreSQL interprète cette requête ?**

En raison de la priorité des opérateurs (`AND` avant `OR`), cela devient :

```sql
WHERE departement = 'IT' OR (departement = 'RH' AND salaire > 50000)
```

**Résultat :**
- **TOUS** les employés IT (quel que soit leur salaire)
- Les employés RH gagnant plus de 50000

**Ce n'est probablement pas ce que vous vouliez !**

### Solution : Utiliser des parenthèses

```sql
-- AVEC parenthèses (clair)
SELECT nom, departement, salaire  
FROM employes  
WHERE (departement = 'IT' OR departement = 'RH') AND salaire > 50000;  
```

**Résultat :**
- Les employés IT **gagnant plus de 50000**
- Les employés RH **gagnant plus de 50000**

### Règle d'or

> **Utilisez TOUJOURS des parenthèses** pour clarifier vos intentions, même si ce n'est pas strictement nécessaire. Votre code sera plus lisible et moins sujet aux erreurs.

**Exemples de bonnes pratiques :**

```sql
-- ✅ Clair et explicite
WHERE (age >= 18 AND ville = 'Paris') OR (age >= 21 AND ville = 'New York')

-- ❌ Ambigu (même si techniquement correct)
WHERE age >= 18 AND ville = 'Paris' OR age >= 21 AND ville = 'New York'

-- ✅ Facile à comprendre
WHERE NOT (statut = 'inactif' OR statut = 'suspendu')

-- ❌ Moins clair
WHERE NOT statut = 'inactif' AND NOT statut = 'suspendu'
```

---

## Opérateurs avancés

### IN : Appartenance à une liste

`IN` vérifie si une valeur appartient à un **ensemble de valeurs**.

**Syntaxe :**
```sql
WHERE colonne IN (valeur1, valeur2, valeur3, ...)
```

**Exemples :**

```sql
-- Employés des départements IT, RH ou Finance
SELECT nom, departement  
FROM employes  
WHERE departement IN ('IT', 'RH', 'Finance');  

-- Équivalent (mais plus verbeux)
SELECT nom, departement  
FROM employes  
WHERE departement = 'IT'  
   OR departement = 'RH'
   OR departement = 'Finance';

-- Produits avec des IDs spécifiques
SELECT * FROM produits  
WHERE id IN (10, 25, 42, 103);  

-- Clients habitant dans plusieurs villes
SELECT nom, ville  
FROM clients  
WHERE ville IN ('Paris', 'Lyon', 'Marseille', 'Toulouse');  
```

**NOT IN :**

```sql
-- Employés n'étant PAS dans IT, RH ou Finance
SELECT nom, departement  
FROM employes  
WHERE departement NOT IN ('IT', 'RH', 'Finance');  

-- Produits dont l'ID n'est pas dans la liste
SELECT * FROM produits  
WHERE id NOT IN (10, 25, 42, 103);  
```

**⚠️ Attention avec NULL :**

```sql
-- Si la liste contient NULL, NOT IN peut donner des résultats inattendus
WHERE departement NOT IN ('IT', 'RH', NULL)
-- Aucune ligne ne sera retournée à cause de la logique ternaire !
-- Nous verrons cela en détail dans la section 5.3
```

### BETWEEN : Intervalle de valeurs

`BETWEEN` vérifie si une valeur est **comprise entre** deux bornes (incluses).

**Syntaxe :**
```sql
WHERE colonne BETWEEN valeur_min AND valeur_max
```

**Important :** Les bornes sont **incluses** (c'est un intervalle fermé).

**Exemples :**

```sql
-- Employés gagnant entre 40000 et 60000 (inclus)
SELECT nom, salaire  
FROM employes  
WHERE salaire BETWEEN 40000 AND 60000;  

-- Équivalent
SELECT nom, salaire  
FROM employes  
WHERE salaire >= 40000 AND salaire <= 60000;  

-- Produits avec un prix entre 10 et 50 euros
SELECT nom_produit, prix  
FROM produits  
WHERE prix BETWEEN 10 AND 50;  

-- Commandes passées en 2024
SELECT * FROM commandes  
WHERE date_commande BETWEEN '2024-01-01' AND '2024-12-31';  

-- Étudiants ayant entre 18 et 25 ans
SELECT nom, age FROM etudiants  
WHERE age BETWEEN 18 AND 25;  
```

**NOT BETWEEN :**

```sql
-- Salaires en dehors de la fourchette 40000-60000
SELECT nom, salaire  
FROM employes  
WHERE salaire NOT BETWEEN 40000 AND 60000;  

-- Équivalent
SELECT nom, salaire  
FROM employes  
WHERE salaire < 40000 OR salaire > 60000;  
```

**⚠️ Ordre important :**

```sql
-- ✅ Correct
WHERE age BETWEEN 18 AND 65

-- ❌ Incorrect (aucun résultat)
WHERE age BETWEEN 65 AND 18  -- min doit être inférieur à max !
```

### LIKE : Recherche de motifs (pattern matching)

`LIKE` permet de rechercher des chaînes de caractères selon un **motif** (pattern).

**Caractères spéciaux :**
- `%` : Représente **zéro, un ou plusieurs caractères**  
- `_` : Représente **exactement un caractère**

**Syntaxe :**
```sql
WHERE colonne LIKE 'motif'
```

**Exemples avec % :**

```sql
-- Noms commençant par 'Dup'
SELECT nom FROM employes  
WHERE nom LIKE 'Dup%';  
-- Trouve : Dupont, Dupuis, Duprès, etc.

-- Noms se terminant par 'ard'
SELECT nom FROM employes  
WHERE nom LIKE '%ard';  
-- Trouve : Bernard, Gérard, Richard, etc.

-- Noms contenant 'mar' (n'importe où)
SELECT nom FROM employes  
WHERE nom LIKE '%mar%';  
-- Trouve : Martin, Lamarche, Dumarché, Marie, etc.

-- Emails du domaine gmail.com
SELECT email FROM clients  
WHERE email LIKE '%@gmail.com';  

-- Produits dont le nom contient 'ordinateur'
SELECT nom_produit FROM produits  
WHERE nom_produit LIKE '%ordinateur%';  
```

**Exemples avec _ :**

```sql
-- Codes produit format : 3 lettres + 2 chiffres (ex: ABC12)
SELECT code FROM produits  
WHERE code LIKE '___##';  -- 3 underscores + 2 caractères  

-- Numéros de téléphone format : 06-##-##-##-##
SELECT telephone FROM clients  
WHERE telephone LIKE '06-__-__-__-__';  

-- Codes postaux parisiens (750## ou 751##)
SELECT ville, code_postal FROM adresses  
WHERE code_postal LIKE '75%';  
```

**Combiner % et _ :**

```sql
-- Prénoms de 5 lettres commençant par 'M'
SELECT prenom FROM employes  
WHERE prenom LIKE 'M____';  -- M + 4 underscores  
-- Trouve : Marie, Marco, Manon, etc.

-- Emails du format : prénom.nom@entreprise.com
SELECT email FROM employes  
WHERE email LIKE '%._@%.com';  
```

**ILIKE : Insensible à la casse (spécifique PostgreSQL)**

```sql
-- Recherche insensible à la casse (PostgreSQL uniquement)
SELECT nom FROM employes  
WHERE nom ILIKE 'dup%';  
-- Trouve : Dupont, DUPONT, dupont, DuPont, etc.
```

**NOT LIKE :**

```sql
-- Employés dont le nom ne commence PAS par 'A'
SELECT nom FROM employes  
WHERE nom NOT LIKE 'A%';  

-- Emails qui ne sont PAS des Gmail
SELECT email FROM clients  
WHERE email NOT LIKE '%@gmail.com';  
```

**Échapper les caractères spéciaux :**

Si votre motif contient littéralement `%` ou `_`, utilisez `ESCAPE` :

```sql
-- Rechercher des produits avec un nom contenant littéralement '50%'
SELECT nom FROM produits  
WHERE nom LIKE '%50!%%' ESCAPE '!';  
-- Le ! signale que le % suivant est littéral

-- Rechercher des fichiers .txt (underscore littéral)
SELECT nom_fichier FROM documents  
WHERE nom_fichier LIKE '%!_.txt' ESCAPE '!';  
```

### SIMILAR TO et expressions régulières

PostgreSQL supporte aussi les expressions régulières pour des recherches plus complexes :

```sql
-- Avec SIMILAR TO (SQL standard)
SELECT nom FROM employes  
WHERE nom SIMILAR TO '(Martin|Dupont|Bernard)';  

-- Avec ~ (expression régulière POSIX, spécifique PostgreSQL)
SELECT email FROM clients  
WHERE email ~ '^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$';  

-- Insensible à la casse avec ~*
SELECT nom FROM employes  
WHERE nom ~* '^dup';  -- Commence par 'dup' (insensible à la casse)  
```

---

## Opérateurs de NULL

### IS NULL et IS NOT NULL

Les valeurs `NULL` représentent **l'absence de données** ou **une valeur inconnue**. Elles nécessitent des opérateurs spéciaux.

**⚠️ Erreur fréquente :**

```sql
-- ❌ NE FONCTIONNE PAS
WHERE colonne = NULL   -- Toujours faux !  
WHERE colonne != NULL  -- Toujours faux !  
```

**✅ Syntaxe correcte :**

```sql
-- Lignes où la colonne est NULL
WHERE colonne IS NULL

-- Lignes où la colonne n'est pas NULL
WHERE colonne IS NOT NULL
```

**Exemples :**

```sql
-- Employés sans manager assigné
SELECT nom, manager_id  
FROM employes  
WHERE manager_id IS NULL;  

-- Produits dont le prix est défini
SELECT nom_produit, prix  
FROM produits  
WHERE prix IS NOT NULL;  

-- Clients ayant renseigné leur email
SELECT nom, email  
FROM clients  
WHERE email IS NOT NULL AND email != '';  

-- Commandes en attente de livraison (date_livraison pas encore définie)
SELECT numero_commande, date_commande, date_livraison  
FROM commandes  
WHERE date_livraison IS NULL;  
```

**Combiner avec d'autres conditions :**

```sql
-- Employés IT sans manager ou avec un salaire > 70000
SELECT nom, departement, manager_id, salaire  
FROM employes  
WHERE departement = 'IT'  
  AND (manager_id IS NULL OR salaire > 70000);

-- Produits disponibles (prix défini et stock > 0)
SELECT nom_produit, prix, quantite_stock  
FROM produits  
WHERE prix IS NOT NULL  
  AND quantite_stock > 0;
```

---

## Filtrage sur des expressions calculées

Vous pouvez utiliser des **expressions** dans la clause `WHERE`, pas seulement des colonnes :

```sql
-- Employés dont le salaire annuel dépasse 600000
SELECT nom, salaire  
FROM employes  
WHERE salaire * 12 > 600000;  

-- Produits avec une remise de plus de 20%
SELECT nom_produit, prix, prix_reduit  
FROM produits  
WHERE ((prix - prix_reduit) / prix) > 0.20;  

-- Clients dont le nom complet (prénom + nom) dépasse 20 caractères
SELECT prenom, nom  
FROM clients  
WHERE LENGTH(prenom || ' ' || nom) > 20;  

-- Commandes passées dans les 30 derniers jours
SELECT * FROM commandes  
WHERE date_commande > CURRENT_DATE - INTERVAL '30 days';  

-- Employés embauchés il y a plus de 5 ans
SELECT nom, date_embauche  
FROM employes  
WHERE date_embauche < CURRENT_DATE - INTERVAL '5 years';  
```

**Fonctions dans WHERE :**

```sql
-- Recherche insensible à la casse avec LOWER()
SELECT nom, email  
FROM clients  
WHERE LOWER(email) = 'jean.dupont@example.com';  

-- Extraire l'année d'une date
SELECT nom, date_embauche  
FROM employes  
WHERE EXTRACT(YEAR FROM date_embauche) = 2023;  

-- Nom contenant exactement 5 caractères
SELECT nom FROM employes  
WHERE LENGTH(nom) = 5;  

-- Prix arrondi à l'euro supérieur
SELECT nom_produit, prix  
FROM produits  
WHERE CEIL(prix) > 100;  
```

---

## Cas d'usage avancés

### Recherche de plusieurs motifs

```sql
-- Produits contenant 'ordinateur' OU 'laptop' OU 'PC'
SELECT nom_produit  
FROM produits  
WHERE nom_produit ILIKE '%ordinateur%'  
   OR nom_produit ILIKE '%laptop%'
   OR nom_produit ILIKE '%PC%';

-- Avec expression régulière (plus concis)
SELECT nom_produit  
FROM produits  
WHERE nom_produit ~* 'ordinateur|laptop|PC';  
```

### Filtrage sur plages multiples

```sql
-- Salaires dans plusieurs fourchettes
SELECT nom, salaire  
FROM employes  
WHERE (salaire BETWEEN 30000 AND 40000)  
   OR (salaire BETWEEN 60000 AND 80000)
   OR (salaire > 100000);

-- Âges : mineurs ou seniors
SELECT nom, age  
FROM clients  
WHERE age < 18 OR age >= 65;  
```

### Filtrage complexe avec dates

```sql
-- Commandes du dernier trimestre
SELECT * FROM commandes  
WHERE date_commande >= DATE_TRUNC('quarter', CURRENT_DATE - INTERVAL '3 months')  
  AND date_commande < DATE_TRUNC('quarter', CURRENT_DATE);

-- Employés embauchés en janvier de n'importe quelle année
SELECT nom, date_embauche  
FROM employes  
WHERE EXTRACT(MONTH FROM date_embauche) = 1;  

-- Événements du week-end
SELECT titre, date_evenement  
FROM evenements  
WHERE EXTRACT(DOW FROM date_evenement) IN (0, 6);  -- 0=dimanche, 6=samedi  
```

### Exclusions multiples

```sql
-- Tous les départements sauf IT, RH et Finance
SELECT nom, departement  
FROM employes  
WHERE departement NOT IN ('IT', 'RH', 'Finance');  

-- Produits actifs (ni obsolètes, ni discontinués, ni en rupture)
SELECT nom_produit, statut  
FROM produits  
WHERE statut NOT IN ('obsolète', 'discontinué')  
  AND quantite_stock > 0;
```

---

## Performances et bonnes pratiques

### 1. WHERE est votre meilleur ami pour les performances

Plus vous filtrez tôt avec `WHERE`, moins PostgreSQL a de données à traiter :

```sql
-- ✅ Excellent : filtre avant traitement
SELECT AVG(salaire)  
FROM employes  
WHERE departement = 'IT';  

-- ❌ Moins bon : traite toutes les données puis filtre
SELECT departement, AVG(salaire)  
FROM employes  
GROUP BY departement  
HAVING departement = 'IT';  
```

### 2. Utilisez les index

Les colonnes fréquemment utilisées dans `WHERE` devraient être **indexées** :

```sql
-- Si vous faites souvent :
WHERE email = 'user@example.com'

-- Créez un index :
CREATE INDEX idx_clients_email ON clients(email);
```

### 3. Évitez les fonctions sur les colonnes indexées

```sql
-- ❌ Ne peut pas utiliser l'index sur 'nom'
WHERE UPPER(nom) = 'DUPONT';

-- ✅ Peut utiliser l'index (si créé)
WHERE nom = 'Dupont' OR nom = 'DUPONT' OR nom = 'dupont';

-- Ou créez un index fonctionnel :
CREATE INDEX idx_clients_nom_upper ON clients(UPPER(nom));  
WHERE UPPER(nom) = 'DUPONT';  -- Peut maintenant utiliser l'index  
```

### 4. IN vs OR : quelle est la différence ?

En termes de performances, `IN` et `OR` sont équivalents (PostgreSQL les optimise de la même façon) :

```sql
-- Ces deux requêtes ont les mêmes performances
WHERE ville IN ('Paris', 'Lyon', 'Marseille')  
WHERE ville = 'Paris' OR ville = 'Lyon' OR ville = 'Marseille'  
```

Utilisez `IN` pour la **lisibilité** quand vous avez plusieurs valeurs.

### 5. LIKE avec % au début est coûteux

```sql
-- ❌ Lent : ne peut pas utiliser d'index standard
WHERE nom LIKE '%dupont%';

-- ✅ Rapide : peut utiliser un index
WHERE nom LIKE 'dupont%';

-- Pour rechercher dans tout le texte efficacement, utilisez :
-- - Un index GIN avec pg_trgm (trigrammes)
-- - Full-Text Search (tsvector/tsquery)
```

### 6. Combinez les conditions intelligemment

```sql
-- ❌ Moins efficace (évalue toutes les conditions)
WHERE condition_rare AND condition_frequente;

-- ✅ Plus efficace (ordre optimisé)
WHERE condition_frequente AND condition_rare;
```

PostgreSQL optimise généralement cela automatiquement, mais c'est une bonne pratique à garder en tête.

---

## Pièges courants et erreurs à éviter

### 1. Oublier les parenthèses avec AND/OR

```sql
-- ❌ Ambigu
WHERE age > 18 OR age < 65 AND ville = 'Paris'

-- ✅ Clair
WHERE (age > 18 OR age < 65) AND ville = 'Paris'
```

### 2. Utiliser = NULL au lieu de IS NULL

```sql
-- ❌ Ne fonctionne jamais
WHERE manager_id = NULL

-- ✅ Correct
WHERE manager_id IS NULL
```

### 3. NOT IN avec des NULL

```sql
-- ⚠️ Peut donner des résultats inattendus
WHERE departement NOT IN ('IT', 'RH', NULL)
-- Aucune ligne retournée à cause de NULL !

-- ✅ Solution
WHERE departement NOT IN ('IT', 'RH') OR departement IS NULL
```

### 4. Comparaison de chaînes avec espaces

```sql
-- Ces deux valeurs sont différentes :
WHERE nom = 'Dupont'   -- 6 caractères  
WHERE nom = 'Dupont '  -- 7 caractères (espace à la fin)  

-- Solution : nettoyer avec TRIM()
WHERE TRIM(nom) = 'Dupont'
```

### 5. Confusion entre BETWEEN et </>

```sql
-- BETWEEN inclut les bornes
WHERE age BETWEEN 18 AND 65  -- 18 et 65 inclus

-- Si vous voulez exclure les bornes :
WHERE age > 18 AND age < 65  -- 18 et 65 exclus
```

### 6. LIKE sensible à la casse

```sql
-- ❌ Ne trouve pas 'DUPONT' ni 'dupont'
WHERE nom LIKE 'Dupont%'

-- ✅ Solution PostgreSQL
WHERE nom ILIKE 'Dupont%'

-- ✅ Solution standard SQL
WHERE LOWER(nom) LIKE LOWER('Dupont%')
```

---

## Récapitulatif des opérateurs

| Catégorie | Opérateurs | Usage |
|-----------|------------|-------|
| **Comparaison** | `=`, `!=`, `<>`, `>`, `<`, `>=`, `<=` | Égalité, inégalité, ordre |
| **Logique** | `AND`, `OR`, `NOT` | Combiner des conditions |
| **Appartenance** | `IN`, `NOT IN` | Vérifier si dans une liste |
| **Intervalle** | `BETWEEN`, `NOT BETWEEN` | Vérifier si dans un intervalle |
| **Motifs** | `LIKE`, `ILIKE`, `NOT LIKE`, `SIMILAR TO`, `~`, `~*` | Recherche de patterns |
| **NULL** | `IS NULL`, `IS NOT NULL` | Tester les valeurs NULL |
| **Existence** | `EXISTS`, `NOT EXISTS` | Sous-requêtes (voir plus tard) |

---

## Exemples synthétiques

### Recherche multi-critères typique

```sql
SELECT
    nom,
    prenom,
    email,
    ville,
    age
FROM clients  
WHERE  
    -- Critère géographique
    ville IN ('Paris', 'Lyon', 'Marseille')
    -- Critère d'âge
    AND age BETWEEN 25 AND 45
    -- Critère de contact
    AND email IS NOT NULL
    -- Critère de statut
    AND statut_compte = 'actif'
    -- Critère d'engagement
    AND (derniere_connexion > CURRENT_DATE - INTERVAL '30 days'
         OR nb_commandes > 5)
ORDER BY derniere_connexion DESC;
```

### Recherche de produits

```sql
SELECT
    nom_produit,
    categorie,
    prix,
    quantite_stock
FROM produits  
WHERE  
    -- Catégories acceptées
    categorie IN ('Électronique', 'Informatique', 'Téléphonie')
    -- Prix dans une fourchette
    AND prix BETWEEN 100 AND 1000
    -- En stock
    AND quantite_stock > 0
    -- Pas en fin de vie
    AND statut != 'obsolète'
    -- Recherche textuelle
    AND (nom_produit ILIKE '%samsung%'
         OR nom_produit ILIKE '%apple%')
ORDER BY prix ASC;
```

---

## Points clés à retenir

1. **WHERE filtre les lignes individuelles** avant tout traitement ultérieur (GROUP BY, etc.)

2. **Les opérateurs logiques ont une priorité** : NOT > AND > OR. Utilisez des parenthèses !

3. **IN est équivalent à plusieurs OR**, mais plus lisible pour des listes longues

4. **BETWEEN inclut les bornes** (intervalle fermé)

5. **LIKE utilise % et _** pour le pattern matching  
   - `%` = zéro ou plusieurs caractères  
   - `_` = exactement un caractère

6. **NULL nécessite IS NULL / IS NOT NULL**, jamais `= NULL`

7. **ILIKE est insensible à la casse** (spécifique PostgreSQL)

8. **Filtrer tôt améliore les performances** : placez les conditions restrictives dans WHERE

9. **Les parenthèses clarifient les intentions** : utilisez-les généreusement

10. **Les expressions calculées sont possibles** dans WHERE, mais attention aux performances avec les fonctions sur colonnes indexées

---

## Conclusion

La clause `WHERE` et la logique booléenne sont les fondations du filtrage en SQL. Maîtriser ces concepts vous permet de :

- Extraire précisément les données dont vous avez besoin
- Optimiser les performances de vos requêtes
- Combiner des conditions complexes de manière lisible
- Éviter les erreurs courantes (NULL, priorité d'opérateurs, etc.)

Dans le prochain chapitre, nous approfondirons le comportement spécial des valeurs `NULL`, qui représente un défi particulier en SQL et nécessite une compréhension de la **logique ternaire**.

---


⏭️ [Le piège du NULL : Logique ternaire et fonctions de gestion (COALESCE, NULLIF)](/05-requetes-de-selection/03-piege-du-null.md)
