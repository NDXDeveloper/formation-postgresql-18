🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4. Tri (ORDER BY) et gestion des NULLs (NULLS FIRST/LAST)

## Introduction

Dans les chapitres précédents, nous avons appris à filtrer les données avec `WHERE`. Maintenant, nous allons apprendre à **organiser** ces données de manière cohérente grâce à la clause `ORDER BY`.

Le tri est une opération fondamentale en SQL qui permet de :
- Présenter les données dans un ordre logique et lisible
- Faciliter l'analyse (du plus petit au plus grand, de A à Z, etc.)
- Identifier rapidement les valeurs extrêmes (top 10, bottom 5, etc.)
- Préparer les données pour la pagination

Comme nous l'avons vu dans le chapitre sur l'ordre d'exécution logique, **ORDER BY est l'une des dernières étapes** (7ème position), juste avant `LIMIT`. Cela signifie que le tri s'applique sur le résultat final après tous les filtres et calculs.

---

## ORDER BY : Les fondamentaux

### Syntaxe de base

```sql
SELECT colonnes  
FROM table  
WHERE conditions  
ORDER BY colonne_de_tri [ASC|DESC];  
```

- `ASC` : Ordre **ascendant** (du plus petit au plus grand, de A à Z) - **par défaut**  
- `DESC` : Ordre **descendant** (du plus grand au plus petit, de Z à A)

### Tri ascendant (ASC)

**Ordre ascendant = croissant = du plus petit au plus grand**

```sql
-- Trier les employés par salaire croissant
SELECT nom, salaire  
FROM employes  
ORDER BY salaire ASC;  

-- ASC est optionnel (c'est le défaut)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire;  -- Équivalent à ORDER BY salaire ASC  
```

**Résultat :**
```
nom      | salaire
---------|--------
Alice    | 30000  
Bob      | 35000  
Charlie  | 45000  
David    | 60000  
Eve      | 75000  
```

**Pour les nombres :** du plus petit au plus grand (30000 → 75000)

**Pour les textes :** ordre alphabétique (A → Z)

```sql
-- Trier les clients par nom alphabétique
SELECT nom, prenom  
FROM clients  
ORDER BY nom ASC;  
```

**Résultat :**
```
nom      | prenom
---------|--------
Dupont   | Jean  
Durand   | Marie  
Martin   | Pierre  
Petit    | Sophie  
```

**Pour les dates :** de la plus ancienne à la plus récente

```sql
-- Trier les commandes de la plus ancienne à la plus récente
SELECT numero_commande, date_commande  
FROM commandes  
ORDER BY date_commande ASC;  
```

**Résultat :**
```
numero_commande | date_commande
----------------|---------------
CMD001          | 2024-01-15  
CMD002          | 2024-02-10  
CMD003          | 2024-03-05  
CMD004          | 2024-04-20  
```

### Tri descendant (DESC)

**Ordre descendant = décroissant = du plus grand au plus petit**

```sql
-- Trier les employés par salaire décroissant (les mieux payés en premier)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC;  
```

**Résultat :**
```
nom      | salaire
---------|--------
Eve      | 75000  
David    | 60000  
Charlie  | 45000  
Bob      | 35000  
Alice    | 30000  
```

**Pour les nombres :** du plus grand au plus petit (75000 → 30000)

**Pour les textes :** ordre alphabétique inverse (Z → A)

```sql
-- Trier les clients par nom inverse
SELECT nom, prenom  
FROM clients  
ORDER BY nom DESC;  
```

**Résultat :**
```
nom      | prenom
---------|--------
Petit    | Sophie  
Martin   | Pierre  
Durand   | Marie  
Dupont   | Jean  
```

**Pour les dates :** de la plus récente à la plus ancienne

```sql
-- Trier les commandes de la plus récente à la plus ancienne
SELECT numero_commande, date_commande  
FROM commandes  
ORDER BY date_commande DESC;  
```

**Résultat :**
```
numero_commande | date_commande
----------------|---------------
CMD004          | 2024-04-20  
CMD003          | 2024-03-05  
CMD002          | 2024-02-10  
CMD001          | 2024-01-15  
```

---

## Tri sur plusieurs colonnes

PostgreSQL permet de trier sur **plusieurs colonnes** simultanément. Le tri se fait en cascade : d'abord par la première colonne, puis en cas d'égalité, par la deuxième, et ainsi de suite.

### Syntaxe

```sql
ORDER BY colonne1 [ASC|DESC], colonne2 [ASC|DESC], colonne3 [ASC|DESC], ...
```

### Exemple simple

```sql
-- Trier par département (A→Z), puis par salaire décroissant dans chaque département
SELECT nom, departement, salaire  
FROM employes  
ORDER BY departement ASC, salaire DESC;  
```

**Résultat :**
```
nom      | departement | salaire
---------|-------------|--------
Eve      | Finance     | 75000    -- Finance trié par salaire DESC  
David    | Finance     | 60000  
Alice    | IT          | 55000    -- IT trié par salaire DESC  
Bob      | IT          | 45000  
Charlie  | IT          | 40000  
Sophie   | RH          | 50000    -- RH trié par salaire DESC  
Pierre   | RH          | 35000  
```

**Comment ça fonctionne :**
1. PostgreSQL trie d'abord par `departement` en ordre alphabétique (Finance, IT, RH)  
2. **À l'intérieur de chaque département**, il trie par `salaire` décroissant

### Exemple avec trois colonnes

```sql
-- Trier par pays, puis ville, puis nom
SELECT nom, ville, pays  
FROM clients  
ORDER BY pays ASC, ville ASC, nom ASC;  
```

**Résultat :**
```
nom      | ville      | pays
---------|------------|--------
Dupont   | Lyon       | France  
Martin   | Lyon       | France  
Durand   | Paris      | France  
Petit    | Paris      | France  
Schmidt  | Berlin     | Allemagne  
Mueller  | Munich     | Allemagne  
```

**Logique :**
1. Trier par pays (Allemagne, France, ...)  
2. Dans chaque pays, trier par ville  
3. Dans chaque ville, trier par nom

### Mélanger ASC et DESC

Vous pouvez mélanger les ordres de tri pour chaque colonne :

```sql
-- Départements A→Z, mais salaires du plus élevé au plus bas dans chaque département
SELECT nom, departement, salaire  
FROM employes  
ORDER BY departement ASC, salaire DESC;  

-- Année récente en premier, mais dans chaque année, noms A→Z
SELECT nom, annee_embauche  
FROM employes  
ORDER BY annee_embauche DESC, nom ASC;  
```

---

## Les valeurs NULL dans le tri

Les valeurs `NULL` posent une question particulière lors du tri : **où les placer** ? Avant les autres valeurs ? Après ?

### Comportement par défaut de PostgreSQL

**En ordre ascendant (ASC) :**
- PostgreSQL place les `NULL` **en dernier** par défaut

```sql
SELECT nom, salaire  
FROM employes  
ORDER BY salaire ASC;  
```

**Résultat :**
```
nom      | salaire
---------|--------
Alice    | 30000  
Bob      | 40000  
Charlie  | 50000  
David    | NULL     -- NULL en dernier  
Eve      | NULL  
```

**En ordre descendant (DESC) :**
- PostgreSQL place les `NULL` **en premier** par défaut

```sql
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC;  
```

**Résultat :**
```
nom      | salaire
---------|--------
David    | NULL     -- NULL en premier  
Eve      | NULL  
Charlie  | 50000  
Bob      | 40000  
Alice    | 30000  
```

### NULLS FIRST et NULLS LAST

PostgreSQL permet de **contrôler explicitement** où placer les NULL avec les clauses `NULLS FIRST` et `NULLS LAST`.

**Syntaxe :**
```sql
ORDER BY colonne [ASC|DESC] [NULLS FIRST|NULLS LAST]
```

### NULLS FIRST : NULL en premier

```sql
-- Tri ascendant avec NULL en premier
SELECT nom, salaire  
FROM employes  
ORDER BY salaire ASC NULLS FIRST;  
```

**Résultat :**
```
nom      | salaire
---------|--------
David    | NULL     -- NULL d'abord  
Eve      | NULL  
Alice    | 30000  
Bob      | 40000  
Charlie  | 50000  
```

```sql
-- Tri descendant avec NULL en premier (comportement par défaut DESC)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC NULLS FIRST;  
```

**Résultat :**
```
nom      | salaire
---------|--------
David    | NULL     -- NULL d'abord (c'est le défaut pour DESC)  
Eve      | NULL  
Charlie  | 50000  
Bob      | 40000  
Alice    | 30000  
```

### NULLS LAST : NULL en dernier

```sql
-- Tri ascendant avec NULL en dernier (comportement par défaut ASC)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire ASC NULLS LAST;  
```

**Résultat :**
```
nom      | salaire
---------|--------
Alice    | 30000  
Bob      | 40000  
Charlie  | 50000  
David    | NULL     -- NULL à la fin (c'est le défaut pour ASC)  
Eve      | NULL  
```

```sql
-- Tri descendant avec NULL en dernier
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC NULLS LAST;  
```

**Résultat :**
```
nom      | salaire
---------|--------
Charlie  | 50000  
Bob      | 40000  
Alice    | 30000  
David    | NULL     -- NULL à la fin  
Eve      | NULL  
```

### Récapitulatif du comportement par défaut

| Tri | Comportement par défaut | Équivalent explicite |
|-----|-------------------------|----------------------|
| `ORDER BY col ASC` | NULL en dernier | `ORDER BY col ASC NULLS LAST` |
| `ORDER BY col DESC` | NULL en premier | `ORDER BY col DESC NULLS FIRST` |

### Cas d'usage pratiques

#### 1. Priorités de traitement

```sql
-- Afficher les commandes en attente de livraison en premier
-- (date_livraison = NULL signifie "pas encore livrée")
SELECT numero_commande, date_commande, date_livraison  
FROM commandes  
ORDER BY date_livraison ASC NULLS FIRST;  
```

**Résultat :**
```
numero_commande | date_commande | date_livraison
----------------|---------------|----------------
CMD005          | 2024-11-15    | NULL           -- À traiter en priorité  
CMD006          | 2024-11-16    | NULL  
CMD001          | 2024-10-01    | 2024-10-05     -- Déjà livrées  
CMD002          | 2024-10-10    | 2024-10-15  
```

#### 2. Données optionnelles à la fin

```sql
-- Clients avec email renseigné en premier
SELECT nom, email  
FROM clients  
ORDER BY email ASC NULLS LAST;  
```

#### 3. Tri avec plusieurs colonnes et gestion de NULL

```sql
-- Trier par département, puis par salaire
-- Placer les employés sans département à la fin
-- Placer les salaires inconnus à la fin
SELECT nom, departement, salaire  
FROM employes  
ORDER BY  
    departement ASC NULLS LAST,
    salaire DESC NULLS LAST;
```

**Résultat :**
```
nom      | departement | salaire
---------|-------------|--------
Charlie  | Finance     | 60000  
Alice    | Finance     | 50000  
Bob      | IT          | 55000  
David    | IT          | 45000  
Sophie   | IT          | NULL     -- Salaire inconnu à la fin du département  
Pierre   | RH          | 40000  
Marc     | NULL        | 70000    -- Département inconnu à la fin  
Julie    | NULL        | NULL     -- Les deux inconnus à la toute fin  
```

---

## Tri sur des expressions calculées

Vous pouvez trier sur des **calculs** ou **fonctions**, pas seulement sur des colonnes brutes.

### Exemples numériques

```sql
-- Trier par salaire annuel (salaire mensuel * 12)
SELECT nom, salaire, salaire * 12 as salaire_annuel  
FROM employes  
ORDER BY salaire * 12 DESC;  

-- Trier par prix après remise
SELECT nom_produit, prix, remise, prix - remise as prix_final  
FROM produits  
ORDER BY prix - remise ASC;  

-- Trier par valeur absolue (distance à zéro)
SELECT nom, solde_compte  
FROM clients  
ORDER BY ABS(solde_compte) DESC;  
```

### Exemples avec texte

```sql
-- Trier par longueur du nom (du plus court au plus long)
SELECT nom, LENGTH(nom) as longueur  
FROM clients  
ORDER BY LENGTH(nom) ASC;  

-- Trier alphabétiquement sans tenir compte de la casse
SELECT nom  
FROM clients  
ORDER BY LOWER(nom) ASC;  

-- Trier par nom de famille (après l'espace)
SELECT nom_complet  
FROM clients  
ORDER BY SPLIT_PART(nom_complet, ' ', 2) ASC;  
```

### Exemples avec dates

```sql
-- Trier par ancienneté (différence avec aujourd'hui)
SELECT nom, date_embauche,
       CURRENT_DATE - date_embauche as anciennete_jours
FROM employes  
ORDER BY date_embauche ASC;  -- Les plus anciens en premier  

-- Trier par âge
SELECT nom, date_naissance,
       EXTRACT(YEAR FROM AGE(date_naissance)) as age
FROM clients  
ORDER BY date_naissance ASC;  -- Les plus âgés en premier  

-- Trier par mois (ignorer l'année)
SELECT nom, date_naissance  
FROM clients  
ORDER BY EXTRACT(MONTH FROM date_naissance) ASC;  
```

### Utiliser les alias dans ORDER BY

Une particularité importante : **ORDER BY peut utiliser les alias** définis dans `SELECT` (contrairement à WHERE ou GROUP BY).

```sql
-- ✅ Ceci fonctionne
SELECT
    nom,
    salaire * 12 as salaire_annuel
FROM employes  
ORDER BY salaire_annuel DESC;  -- Utilise l'alias  

-- Équivalent (plus verbeux)
SELECT
    nom,
    salaire * 12 as salaire_annuel
FROM employes  
ORDER BY salaire * 12 DESC;  -- Répète l'expression  
```

**Pourquoi ça fonctionne ?**

Comme nous l'avons vu dans le chapitre 5.1, `ORDER BY` est exécuté **après** `SELECT`, donc les alias existent déjà.

---

## Tri par position de colonne

PostgreSQL permet de trier en référençant la **position** d'une colonne dans le `SELECT` (numérotée à partir de 1).

```sql
-- Trier par la 2ème colonne du SELECT
SELECT nom, salaire, departement  
FROM employes  
ORDER BY 2 DESC;  -- 2 = salaire  

-- Équivalent
SELECT nom, salaire, departement  
FROM employes  
ORDER BY salaire DESC;  
```

**Tri sur plusieurs colonnes par position :**

```sql
SELECT nom, departement, salaire  
FROM employes  
ORDER BY 2 ASC, 3 DESC;  -- departement ASC, salaire DESC  
```

**⚠️ Attention :**

Cette syntaxe est **déconseillée** en production car :
- Moins lisible (que représente "2" ?)
- Fragile (si vous ajoutez/supprimez une colonne, les numéros changent)
- Difficile à maintenir

**Recommandation :** Utilisez les noms de colonnes explicites.

```sql
-- ❌ À éviter
ORDER BY 2, 3 DESC

-- ✅ Préférez
ORDER BY departement ASC, salaire DESC
```

---

## Tri avec CASE : Ordre personnalisé

Parfois, vous voulez un ordre de tri qui ne suit pas l'ordre alphabétique ou numérique naturel. Vous pouvez utiliser `CASE` pour définir un **ordre personnalisé**.

### Exemple : Priorités personnalisées

```sql
-- Trier les tickets par priorité : critique > haute > moyenne > basse
SELECT titre, priorite  
FROM tickets  
ORDER BY  
    CASE priorite
        WHEN 'critique' THEN 1
        WHEN 'haute' THEN 2
        WHEN 'moyenne' THEN 3
        WHEN 'basse' THEN 4
        ELSE 5  -- Inconnu à la fin
    END ASC;
```

**Résultat :**
```
titre                | priorite
---------------------|----------
Bug serveur          | critique  
Erreur paiement      | critique  
Lenteur application  | haute  
Demande nouvelle vue | moyenne  
Amélioration design  | basse  
```

### Exemple : Statuts personnalisés

```sql
-- Trier les commandes par statut : en cours > en attente > livrée > annulée
SELECT numero_commande, statut  
FROM commandes  
ORDER BY  
    CASE statut
        WHEN 'en cours' THEN 1
        WHEN 'en attente' THEN 2
        WHEN 'livrée' THEN 3
        WHEN 'annulée' THEN 4
    END ASC,
    date_commande DESC;  -- Puis par date décroissante
```

### Exemple : Jours de la semaine

```sql
-- Trier par jour de la semaine (lundi en premier)
SELECT evenement, jour_semaine  
FROM calendrier  
ORDER BY  
    CASE jour_semaine
        WHEN 'lundi' THEN 1
        WHEN 'mardi' THEN 2
        WHEN 'mercredi' THEN 3
        WHEN 'jeudi' THEN 4
        WHEN 'vendredi' THEN 5
        WHEN 'samedi' THEN 6
        WHEN 'dimanche' THEN 7
    END ASC;
```

### Exemple : Tri mixte (certaines valeurs en premier)

```sql
-- Mettre les produits "featured" en premier, puis trier par prix
SELECT nom_produit, est_featured, prix  
FROM produits  
ORDER BY  
    CASE
        WHEN est_featured = TRUE THEN 0  -- Featured en premier
        ELSE 1
    END ASC,
    prix ASC;  -- Puis par prix croissant
```

---

## Tri et performances

### L'impact du tri sur les performances

Le tri est une opération **coûteuse** en termes de performances, surtout sur de grandes tables.

**Pourquoi ?**
- PostgreSQL doit charger toutes les lignes (après filtrage WHERE)
- Les trier en mémoire (ou sur disque si trop volumineux)
- Puis retourner le résultat

**Complexité algorithmique :** O(n log n) dans le meilleur cas

### Optimiser le tri avec des index

Un **index** sur la colonne de tri peut considérablement améliorer les performances.

```sql
-- Créer un index sur la colonne de tri
CREATE INDEX idx_employes_salaire ON employes(salaire);

-- Cette requête sera beaucoup plus rapide
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC  
LIMIT 10;  
```

**Comment ça fonctionne ?**
- Sans index : PostgreSQL doit trier toute la table
- Avec index : PostgreSQL peut lire directement dans l'ordre de l'index

**Index pour tri ascendant vs descendant :**

PostgreSQL peut utiliser un index dans les deux sens :

```sql
CREATE INDEX idx_employes_salaire ON employes(salaire);

-- Les deux requêtes peuvent utiliser l'index
SELECT * FROM employes ORDER BY salaire ASC;   -- ✅ Index utilisable  
SELECT * FROM employes ORDER BY salaire DESC;  -- ✅ Index utilisable (sens inverse)  
```

**Index composites pour tri multi-colonnes :**

```sql
-- Index sur deux colonnes dans l'ordre de tri
CREATE INDEX idx_employes_dept_salaire ON employes(departement, salaire DESC);

-- Cette requête peut utiliser l'index efficacement
SELECT * FROM employes  
ORDER BY departement ASC, salaire DESC;  
```

### Tri avec LIMIT : Optimisation majeure

Combiner `ORDER BY` avec `LIMIT` permet d'optimiser considérablement les performances :

```sql
-- Top 10 des salaires (PostgreSQL n'a pas besoin de tout trier)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC  
LIMIT 10;  
```

PostgreSQL peut utiliser un algorithme de **top-N heap sort** qui est beaucoup plus efficace qu'un tri complet.

### Quand éviter le tri

Si possible, évitez le tri sur :
- Des expressions complexes sans index
- De très grandes tables sans index approprié
- Des colonnes de type texte très longues

**Alternatives :**
- Trier côté application si le volume est faible
- Utiliser une vue matérialisée pré-triée
- Créer des index appropriés

---

## Cas d'usage pratiques

### 1. Top N (les plus grands)

```sql
-- Top 10 des employés les mieux payés
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC  
LIMIT 10;  

-- Top 5 des produits les plus chers
SELECT nom_produit, prix  
FROM produits  
ORDER BY prix DESC  
LIMIT 5;  
```

### 2. Bottom N (les plus petits)

```sql
-- 10 produits les moins chers
SELECT nom_produit, prix  
FROM produits  
ORDER BY prix ASC  
LIMIT 10;  

-- 5 employés les moins anciens
SELECT nom, date_embauche  
FROM employes  
ORDER BY date_embauche DESC  
LIMIT 5;  
```

### 3. Pagination

```sql
-- Page 1 (résultats 1-20)
SELECT nom, email  
FROM clients  
ORDER BY nom ASC  
LIMIT 20 OFFSET 0;  

-- Page 2 (résultats 21-40)
SELECT nom, email  
FROM clients  
ORDER BY nom ASC  
LIMIT 20 OFFSET 20;  

-- Page 3 (résultats 41-60)
SELECT nom, email  
FROM clients  
ORDER BY nom ASC  
LIMIT 20 OFFSET 40;  
```

**⚠️ Important :** Pour la pagination, il est crucial d'avoir un **tri stable et reproductible**.

```sql
-- ❌ Mauvais : l'ordre peut changer entre les pages si plusieurs personnes ont le même nom
SELECT nom, email  
FROM clients  
ORDER BY nom ASC  
LIMIT 20 OFFSET 20;  

-- ✅ Bon : ajouter un tri secondaire sur l'ID (unique)
SELECT nom, email  
FROM clients  
ORDER BY nom ASC, id ASC  
LIMIT 20 OFFSET 20;  
```

### 4. Dernières entrées

```sql
-- 20 dernières commandes
SELECT numero_commande, date_commande, montant  
FROM commandes  
ORDER BY date_commande DESC  
LIMIT 20;  

-- 10 derniers clients inscrits
SELECT nom, date_inscription  
FROM clients  
ORDER BY date_inscription DESC  
LIMIT 10;  
```

### 5. Tri alphabétique avec accents

```sql
-- Tri alphabétique en ignorant les accents (collation)
SELECT nom  
FROM clients  
ORDER BY nom COLLATE "fr_FR";  

-- Tri insensible à la casse
SELECT nom  
FROM clients  
ORDER BY LOWER(nom) ASC;  
```

### 6. Tri par pertinence (recherche)

```sql
-- Recherche avec tri par pertinence
SELECT nom, description,
       CASE
           WHEN nom ILIKE 'ordinateur%' THEN 1          -- Commence par
           WHEN nom ILIKE '%ordinateur%' THEN 2         -- Contient
           WHEN description ILIKE '%ordinateur%' THEN 3 -- Dans description
           ELSE 4
       END as pertinence
FROM produits  
WHERE nom ILIKE '%ordinateur%' OR description ILIKE '%ordinateur%'  
ORDER BY pertinence ASC, nom ASC;  
```

### 7. Tri géographique (distance)

```sql
-- Clients triés par distance à un point (nécessite PostGIS ou calcul manuel)
SELECT nom, ville,
       SQRT(POW(latitude - 48.8566, 2) + POW(longitude - 2.3522, 2)) as distance
FROM clients  
ORDER BY distance ASC  
LIMIT 10;  
```

---

## Tri aléatoire : RANDOM()

PostgreSQL permet de trier de manière **aléatoire** avec `RANDOM()`.

```sql
-- Sélectionner 10 produits au hasard
SELECT nom_produit, prix  
FROM produits  
ORDER BY RANDOM()  
LIMIT 10;  

-- Tirer un employé au sort
SELECT nom  
FROM employes  
ORDER BY RANDOM()  
LIMIT 1;  
```

**⚠️ Attention aux performances :**
- `ORDER BY RANDOM()` est **très lent** sur de grandes tables
- PostgreSQL doit générer un nombre aléatoire pour chaque ligne, puis trier

**Alternative plus performante pour de grandes tables :**

```sql
-- Utiliser une sous-requête avec TABLESAMPLE
SELECT nom_produit, prix  
FROM produits TABLESAMPLE SYSTEM (10);  -- ~10% de lignes aléatoires  

-- Ou avec un ID aléatoire si vous avez des IDs séquentiels
SELECT nom_produit, prix  
FROM produits  
WHERE id = (  
    SELECT id
    FROM produits
    ORDER BY RANDOM()
    LIMIT 1
);
```

---

## Combiner ORDER BY avec d'autres clauses

### ORDER BY après GROUP BY

```sql
-- Salaire moyen par département, trié du plus élevé au plus bas
SELECT departement, AVG(salaire) as salaire_moyen  
FROM employes  
GROUP BY departement  
ORDER BY salaire_moyen DESC;  

-- Nombre de commandes par client, trié par nombre décroissant
SELECT client_id, COUNT(*) as nb_commandes  
FROM commandes  
GROUP BY client_id  
ORDER BY nb_commandes DESC  
LIMIT 10;  
```

### ORDER BY avec HAVING

```sql
-- Départements avec plus de 5 employés, triés par nombre d'employés
SELECT departement, COUNT(*) as nb_employes  
FROM employes  
GROUP BY departement  
HAVING COUNT(*) > 5  
ORDER BY nb_employes DESC;  
```

### ORDER BY avec DISTINCT

```sql
-- Villes distinctes, triées alphabétiquement
SELECT DISTINCT ville  
FROM clients  
ORDER BY ville ASC;  

-- Départements distincts avec tri personnalisé
SELECT DISTINCT departement  
FROM employes  
ORDER BY  
    CASE departement
        WHEN 'Direction' THEN 1
        WHEN 'IT' THEN 2
        ELSE 3
    END ASC,
    departement ASC;
```

### ORDER BY avec jointures

```sql
-- Employés avec leur manager, triés par département puis nom
SELECT
    e.nom as employe,
    e.departement,
    m.nom as manager
FROM employes e  
LEFT JOIN employes m ON e.manager_id = m.id  
ORDER BY e.departement ASC, e.nom ASC;  

-- Commandes avec client, triées par date décroissante
SELECT
    c.numero_commande,
    c.date_commande,
    cl.nom as client
FROM commandes c  
JOIN clients cl ON c.client_id = cl.id  
ORDER BY c.date_commande DESC;  
```

---

## Erreurs courantes et pièges

### 1. Oublier ORDER BY et obtenir un ordre imprévisible

```sql
-- ❌ L'ordre n'est PAS garanti
SELECT nom FROM employes;

-- ✅ Si vous voulez un ordre spécifique, spécifiez-le
SELECT nom FROM employes ORDER BY nom ASC;
```

**Important :** Sans `ORDER BY`, l'ordre des lignes est **imprévisible** et peut changer entre deux exécutions de la même requête.

### 2. Supposer un ordre par défaut

```sql
-- ❌ Ne présumez jamais que les résultats seront triés par ID
SELECT * FROM produits;

-- ✅ Spécifiez explicitement
SELECT * FROM produits ORDER BY id ASC;
```

### 3. Tri sur des colonnes non sélectionnées

Vous pouvez trier sur des colonnes qui ne sont pas dans le `SELECT` :

```sql
-- ✅ Ceci est valide
SELECT nom, email  
FROM clients  
ORDER BY date_inscription DESC;  -- date_inscription n'est pas dans SELECT  

-- Le tri fonctionne même si la colonne n'est pas affichée
```

**Exception avec DISTINCT :**

```sql
-- ❌ ERREUR avec DISTINCT
SELECT DISTINCT ville  
FROM clients  
ORDER BY nom ASC;  -- Erreur : nom n'est pas dans SELECT DISTINCT  

-- ✅ Solution
SELECT DISTINCT ville, nom  
FROM clients  
ORDER BY nom ASC;  
```

### 4. Tri sur des agrégations sans alias

```sql
-- ✅ Avec alias (recommandé)
SELECT departement, AVG(salaire) as salaire_moyen  
FROM employes  
GROUP BY departement  
ORDER BY salaire_moyen DESC;  

-- ✅ Sans alias (plus verbeux)
SELECT departement, AVG(salaire)  
FROM employes  
GROUP BY departement  
ORDER BY AVG(salaire) DESC;  

-- ❌ Erreur : ne pas répéter l'alias dans GROUP BY
SELECT departement, AVG(salaire) as salaire_moyen  
FROM employes  
GROUP BY departement  
ORDER BY AVG(salaire_moyen) DESC;  -- Erreur : AVG(alias) n'existe pas  
```

### 5. NULL et pagination

```sql
-- ⚠️ Attention : si vous ne gérez pas les NULL, ils peuvent perturber la pagination
SELECT nom, date_derniere_connexion  
FROM clients  
ORDER BY date_derniere_connexion DESC  -- NULL en premier par défaut  
LIMIT 20;  

-- Les premières pages pourraient être remplies de NULL !

-- ✅ Solution : mettre les NULL à la fin
SELECT nom, date_derniere_connexion  
FROM clients  
ORDER BY date_derniere_connexion DESC NULLS LAST  
LIMIT 20;  
```

---

## Récapitulatif des clauses ORDER BY

| Clause | Description | Exemple |
|--------|-------------|---------|
| `ORDER BY col` | Tri ascendant (défaut) | `ORDER BY nom` |
| `ORDER BY col ASC` | Tri ascendant explicite | `ORDER BY salaire ASC` |
| `ORDER BY col DESC` | Tri descendant | `ORDER BY date DESC` |
| `ORDER BY col1, col2` | Tri multi-colonnes | `ORDER BY dept, nom` |
| `ORDER BY col NULLS FIRST` | NULL en premier | `ORDER BY age ASC NULLS FIRST` |
| `ORDER BY col NULLS LAST` | NULL en dernier | `ORDER BY age DESC NULLS LAST` |
| `ORDER BY expression` | Tri sur calcul | `ORDER BY salaire * 12` |
| `ORDER BY alias` | Tri sur alias SELECT | `ORDER BY salaire_annuel` |
| `ORDER BY CASE ...` | Ordre personnalisé | `ORDER BY CASE WHEN ...` |
| `ORDER BY RANDOM()` | Ordre aléatoire | `ORDER BY RANDOM()` |

---

## Points clés à retenir

1. **ORDER BY est exécuté en 7ème position** (avant LIMIT)
   - Après WHERE, GROUP BY, HAVING, SELECT

2. **ASC = ascendant (par défaut), DESC = descendant**
   - ASC : petit → grand, A → Z, ancien → récent
   - DESC : grand → petit, Z → A, récent → ancien

3. **Tri multi-colonnes en cascade**
   - Trier d'abord par col1, puis col2 en cas d'égalité

4. **NULL par défaut :**
   - ASC : NULL en dernier (NULLS LAST)
   - DESC : NULL en premier (NULLS FIRST)

5. **Contrôle explicite avec NULLS FIRST/LAST**
   - Utilisez-les pour un comportement prévisible

6. **ORDER BY peut utiliser les alias**
   - Contrairement à WHERE et GROUP BY

7. **Tri sur expressions et fonctions possible**  
   - `ORDER BY LENGTH(nom)`, `ORDER BY salaire * 12`

8. **Sans ORDER BY, l'ordre est imprévisible**
   - Toujours spécifier ORDER BY si l'ordre importe

9. **Performances : privilégier les index**
   - Index sur colonnes de tri fréquentes

10. **LIMIT avec ORDER BY est optimisé**
    - PostgreSQL utilise des algorithmes efficaces

---

## Conclusion

Le tri avec `ORDER BY` est une fonctionnalité fondamentale de SQL qui permet d'organiser les résultats de manière cohérente et lisible. Combiné avec la gestion explicite des NULL via `NULLS FIRST` et `NULLS LAST`, vous avez un contrôle total sur la présentation de vos données.

Points essentiels à maîtriser :
- Comprendre ASC (croissant) et DESC (décroissant)
- Trier sur plusieurs colonnes simultanément
- Gérer le placement des valeurs NULL
- Optimiser avec des index pour de meilleures performances
- Utiliser ORDER BY systématiquement quand l'ordre compte

Dans le prochain chapitre, nous explorerons la **pagination** avec `LIMIT` et `OFFSET`, qui s'appuient fortement sur un tri stable et cohérent.

---


⏭️ [Pagination et limitation (LIMIT, OFFSET) : Avantages et limites](/05-requetes-de-selection/05-pagination-et-limitation.md)
