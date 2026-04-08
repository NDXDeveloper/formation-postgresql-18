🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.6. DISTINCT et élimination des doublons (DISTINCT ON)

## Introduction

Lorsque vous interrogez une base de données, vous obtenez parfois des **doublons** dans vos résultats. Par exemple, si vous listez toutes les villes où habitent vos clients, une même ville peut apparaître plusieurs fois (autant de fois qu'il y a de clients dans cette ville).

La clause `DISTINCT` permet d'**éliminer ces doublons** et de ne conserver qu'**une seule occurrence** de chaque valeur unique.

Dans ce chapitre, nous allons explorer :
- Le fonctionnement de DISTINCT
- DISTINCT sur une ou plusieurs colonnes
- DISTINCT ON, une extension puissante de PostgreSQL
- Les différences entre DISTINCT et GROUP BY
- Les impacts sur les performances
- Les pièges courants à éviter

---

## Pourquoi des doublons apparaissent ?

### Exemple : Liste des villes

Imaginons une table `clients` :

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    ville VARCHAR(100)
);

INSERT INTO clients (nom, ville) VALUES
    ('Alice', 'Paris'),
    ('Bob', 'Lyon'),
    ('Charlie', 'Paris'),
    ('David', 'Marseille'),
    ('Eve', 'Paris'),
    ('Frank', 'Lyon');
```

Si vous demandez toutes les villes :

```sql
SELECT ville FROM clients;
```

**Résultat :**
```
ville
----------
Paris  
Lyon  
Paris  
Marseille  
Paris  
Lyon  
```

Vous obtenez 6 lignes avec **des doublons** : Paris apparaît 3 fois, Lyon 2 fois.

**Solution avec DISTINCT :**

```sql
SELECT DISTINCT ville FROM clients;
```

**Résultat :**
```
ville
----------
Lyon  
Marseille  
Paris  
```

Maintenant, chaque ville n'apparaît qu'**une seule fois** !

---

## DISTINCT : Syntaxe et fonctionnement

### Syntaxe de base

```sql
SELECT DISTINCT colonne1, colonne2, ...  
FROM table  
WHERE conditions  
ORDER BY colonne;  
```

### Placement de DISTINCT

**DISTINCT** se place **immédiatement après SELECT** :

```sql
-- ✅ Correct
SELECT DISTINCT ville FROM clients;

-- ❌ Incorrect
SELECT ville DISTINCT FROM clients;
```

### Fonctionnement interne

PostgreSQL :
1. Exécute la requête normalement (FROM, WHERE, etc.)  
2. Récupère toutes les lignes  
3. **Élimine les doublons** en comparant toutes les colonnes sélectionnées  
4. Retourne les lignes uniques

**Important :** DISTINCT compare **toutes les colonnes** du SELECT pour déterminer l'unicité.

---

## DISTINCT sur une seule colonne

### Exemple simple

```sql
-- Liste des départements distincts
SELECT DISTINCT departement FROM employes;

-- Liste des catégories de produits distinctes
SELECT DISTINCT categorie FROM produits;

-- Liste des statuts de commandes distincts
SELECT DISTINCT statut FROM commandes;
```

### Avec ORDER BY

Vous pouvez combiner DISTINCT avec ORDER BY :

```sql
-- Villes distinctes, triées alphabétiquement
SELECT DISTINCT ville  
FROM clients  
ORDER BY ville ASC;  
```

**Résultat :**
```
ville
----------
Lyon  
Marseille  
Paris  
```

### Avec WHERE

DISTINCT s'applique **après** le filtrage WHERE :

```sql
-- Villes distinctes des clients actifs uniquement
SELECT DISTINCT ville  
FROM clients  
WHERE statut = 'actif'  
ORDER BY ville;  
```

---

## DISTINCT sur plusieurs colonnes

Lorsque vous utilisez DISTINCT avec **plusieurs colonnes**, PostgreSQL considère qu'une ligne est unique si la **combinaison complète** de toutes les colonnes est unique.

### Exemple

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER,
    produit VARCHAR(50),
    date_commande DATE
);

INSERT INTO commandes (client_id, produit, date_commande) VALUES
    (1, 'Laptop', '2024-01-15'),
    (1, 'Laptop', '2024-01-15'),  -- Doublon exact
    (1, 'Mouse', '2024-01-15'),   -- Produit différent
    (2, 'Laptop', '2024-01-15'),  -- Client différent
    (1, 'Laptop', '2024-02-10');  -- Date différente

-- Combinaisons uniques de client_id et produit
SELECT DISTINCT client_id, produit  
FROM commandes;  
```

**Résultat :**
```
client_id | produit
----------|--------
1         | Laptop
1         | Mouse
2         | Laptop
```

**Explication :**
- (1, 'Laptop') apparaît 3 fois dans les données brutes
- DISTINCT ne garde qu'**une seule** occurrence de cette combinaison
- (1, 'Mouse') et (2, 'Laptop') sont des combinaisons différentes, donc conservées

### Ordre des colonnes dans DISTINCT

L'ordre des colonnes après DISTINCT **n'affecte pas** le résultat, mais affecte l'affichage :

```sql
-- Ces deux requêtes retournent les mêmes combinaisons uniques
SELECT DISTINCT ville, pays FROM clients;  
SELECT DISTINCT pays, ville FROM clients;  

-- Mais l'affichage sera différent (colonnes dans un ordre différent)
```

---

## DISTINCT et NULL

Les valeurs `NULL` sont traitées comme des valeurs **égales** entre elles dans DISTINCT.

### Exemple

```sql
CREATE TABLE employes (
    nom VARCHAR(100),
    manager_id INTEGER
);

INSERT INTO employes (nom, manager_id) VALUES
    ('Alice', NULL),   -- PDG
    ('Bob', NULL),     -- Autre PDG
    ('Charlie', 1),
    ('David', 1),
    ('Eve', 2);

-- Manager IDs distincts
SELECT DISTINCT manager_id FROM employes;
```

**Résultat :**
```
manager_id
-----------
NULL       -- Une seule ligne NULL
1
2
```

**Remarque :** Même si Alice et Bob ont tous les deux `NULL` comme manager_id, DISTINCT ne garde qu'**un seul NULL**.

**Comportement :** `NULL = NULL` est normalement `NULL` (inconnu), mais pour DISTINCT, PostgreSQL traite les NULL comme identiques.

---

## DISTINCT avec COUNT

`COUNT(DISTINCT colonne)` permet de **compter les valeurs uniques**.

### Exemples

```sql
-- Nombre total de clients
SELECT COUNT(*) FROM clients;  -- 100

-- Nombre de villes différentes
SELECT COUNT(DISTINCT ville) FROM clients;  -- 15

-- Nombre de départements différents
SELECT COUNT(DISTINCT departement) FROM employes;

-- Nombre de catégories de produits
SELECT COUNT(DISTINCT categorie) FROM produits;
```

### COUNT(DISTINCT) vs DISTINCT + COUNT(*)

```sql
-- Méthode 1 : COUNT(DISTINCT)
SELECT COUNT(DISTINCT ville) as nb_villes  
FROM clients;  

-- Méthode 2 : DISTINCT + sous-requête
SELECT COUNT(*) as nb_villes  
FROM (  
    SELECT DISTINCT ville FROM clients
) sub;

-- Les deux donnent le même résultat
```

**Recommandation :** Préférez `COUNT(DISTINCT colonne)` pour la simplicité et la performance.

### COUNT(DISTINCT) avec NULL

```sql
-- COUNT(DISTINCT) ignore les NULL
SELECT COUNT(DISTINCT telephone) FROM clients;

-- Si 10 clients n'ont pas de téléphone (NULL)
-- Et que les 90 autres ont 50 numéros différents
-- Résultat : 50 (les NULL ne sont pas comptés)
```

---

## DISTINCT ON : Extension PostgreSQL

`DISTINCT ON` est une **extension puissante** spécifique à PostgreSQL qui permet de sélectionner la **première ligne** de chaque groupe défini par les colonnes spécifiées.

### Syntaxe

```sql
SELECT DISTINCT ON (colonnes_groupement) colonnes_selection  
FROM table  
ORDER BY colonnes_groupement, colonnes_tri;  
```

**Important :** `ORDER BY` est **fortement recommandé** avec DISTINCT ON pour contrôler quelle ligne est conservée dans chaque groupe.

### Différence entre DISTINCT et DISTINCT ON

**DISTINCT :** Élimine les doublons sur **toutes les colonnes** du SELECT

```sql
SELECT DISTINCT ville, nom FROM clients;
-- Garde les lignes où la combinaison (ville, nom) est unique
```

**DISTINCT ON :** Élimine les doublons sur **les colonnes spécifiées** dans ON, mais retourne **toutes les colonnes** du SELECT

```sql
SELECT DISTINCT ON (ville) ville, nom FROM clients ORDER BY ville;
-- Garde UNE seule ligne par ville (la première selon l'ORDER BY)
-- Mais retourne aussi le nom
```

### Exemple simple

```sql
-- Table d'exemple
CREATE TABLE ventes (
    id SERIAL PRIMARY KEY,
    produit VARCHAR(50),
    vendeur VARCHAR(50),
    montant NUMERIC,
    date_vente DATE
);

INSERT INTO ventes (produit, vendeur, montant, date_vente) VALUES
    ('Laptop', 'Alice', 1200, '2024-01-10'),
    ('Laptop', 'Bob', 1150, '2024-01-15'),
    ('Laptop', 'Charlie', 1300, '2024-01-20'),
    ('Mouse', 'Alice', 25, '2024-01-12'),
    ('Mouse', 'Bob', 30, '2024-01-18'),
    ('Keyboard', 'Alice', 80, '2024-01-14');

-- Pour chaque produit, obtenir la vente la plus récente
SELECT DISTINCT ON (produit)
    produit,
    vendeur,
    montant,
    date_vente
FROM ventes  
ORDER BY produit, date_vente DESC;  
```

**Résultat :**
```
produit  | vendeur | montant | date_vente
---------|---------|---------|------------
Keyboard | Alice   | 80      | 2024-01-14  
Laptop   | Charlie | 1300    | 2024-01-20  -- Vente la plus récente pour Laptop  
Mouse    | Bob     | 30      | 2024-01-18  -- Vente la plus récente pour Mouse  
```

**Explication :**
1. DISTINCT ON (produit) groupe les lignes par produit  
2. ORDER BY produit, date_vente DESC trie par produit puis date décroissante  
3. Pour chaque produit, seule la **première ligne** (= la plus récente) est conservée

### Ordre du ORDER BY avec DISTINCT ON

**Règle importante :** Les colonnes dans `DISTINCT ON` doivent apparaître **en premier** dans `ORDER BY`.

```sql
-- ✅ Correct : produit est en premier dans ORDER BY
SELECT DISTINCT ON (produit) produit, montant  
FROM ventes  
ORDER BY produit, date_vente DESC;  

-- ❌ Erreur : ORDER BY doit commencer par produit
SELECT DISTINCT ON (produit) produit, montant  
FROM ventes  
ORDER BY date_vente DESC, produit;  
-- ERROR: SELECT DISTINCT ON expressions must match initial ORDER BY expressions
```

### DISTINCT ON avec plusieurs colonnes

Vous pouvez grouper sur plusieurs colonnes :

```sql
-- Pour chaque combinaison (produit, vendeur), la vente la plus élevée
SELECT DISTINCT ON (produit, vendeur)
    produit,
    vendeur,
    montant,
    date_vente
FROM ventes  
ORDER BY produit, vendeur, montant DESC;  
```

---

## Cas d'usage de DISTINCT ON

### 1. Dernière entrée par groupe

```sql
-- Dernier message de chaque conversation
SELECT DISTINCT ON (conversation_id)
    conversation_id,
    message,
    auteur,
    timestamp
FROM messages  
ORDER BY conversation_id, timestamp DESC;  

-- Dernière connexion de chaque utilisateur
SELECT DISTINCT ON (user_id)
    user_id,
    adresse_ip,
    date_connexion
FROM logs_connexion  
ORDER BY user_id, date_connexion DESC;  
```

### 2. Meilleure valeur par groupe

```sql
-- Produit le moins cher par catégorie
SELECT DISTINCT ON (categorie)
    categorie,
    nom_produit,
    prix
FROM produits  
ORDER BY categorie, prix ASC;  

-- Employé le mieux payé par département
SELECT DISTINCT ON (departement)
    departement,
    nom,
    salaire
FROM employes  
ORDER BY departement, salaire DESC;  
```

### 3. Première occurrence par groupe

```sql
-- Premier client de chaque ville (par ordre d'inscription)
SELECT DISTINCT ON (ville)
    ville,
    nom,
    date_inscription
FROM clients  
ORDER BY ville, date_inscription ASC;  

-- Première commande de chaque produit
SELECT DISTINCT ON (produit_id)
    produit_id,
    numero_commande,
    date_commande
FROM commandes  
ORDER BY produit_id, date_commande ASC;  
```

### 4. Déduplication intelligente

```sql
-- Garder la version la plus récente de chaque enregistrement
SELECT DISTINCT ON (email)
    email,
    nom,
    telephone,
    derniere_mise_a_jour
FROM clients_doublons  
ORDER BY email, derniere_mise_a_jour DESC;  
```

---

## DISTINCT vs GROUP BY

Les deux permettent d'éliminer les doublons, mais ils fonctionnent différemment.

### DISTINCT : Élimination simple

```sql
-- Liste des villes uniques
SELECT DISTINCT ville FROM clients;
```

**Caractéristiques :**
- Élimine simplement les doublons
- Pas d'agrégations possibles
- Retourne les valeurs uniques

### GROUP BY : Regroupement avec agrégations

```sql
-- Nombre de clients par ville
SELECT ville, COUNT(*) as nb_clients  
FROM clients  
GROUP BY ville;  
```

**Caractéristiques :**
- Regroupe les lignes
- Permet des agrégations (COUNT, SUM, AVG, etc.)
- Retourne un résultat par groupe

### Équivalence sans agrégation

Pour obtenir uniquement des valeurs distinctes, les deux sont équivalents :

```sql
-- Ces deux requêtes donnent le même résultat
SELECT DISTINCT ville FROM clients;  
SELECT ville FROM clients GROUP BY ville;  
```

**Quelle syntaxe choisir ?**

```sql
-- ✅ Préférez DISTINCT pour la lisibilité
SELECT DISTINCT ville FROM clients;

-- ✅ Utilisez GROUP BY si vous avez des agrégations
SELECT ville, COUNT(*) FROM clients GROUP BY ville;
```

### Performance : DISTINCT vs GROUP BY

En général, les performances sont **similaires** car PostgreSQL optimise les deux de manière comparable.

```sql
-- Performance équivalente
EXPLAIN ANALYZE SELECT DISTINCT ville FROM clients;  
EXPLAIN ANALYZE SELECT ville FROM clients GROUP BY ville;  

-- Les deux utilisent souvent la même stratégie (HashAggregate ou Sort + Unique)
```

---

## DISTINCT et performances

### Impact sur les performances

L'élimination des doublons nécessite de :
1. **Trier** les résultats (pour identifier les doublons consécutifs)  
2. Ou utiliser une **table de hachage** (pour mémoriser les valeurs vues)

Ces opérations ont un **coût** :
- Utilisation de mémoire (work_mem)
- Temps de traitement supplémentaire
- Peut forcer un tri si pas d'index approprié

### Optimisation avec index

Un index peut grandement améliorer les performances de DISTINCT :

```sql
-- Créer un index sur la colonne
CREATE INDEX idx_clients_ville ON clients(ville);

-- Cette requête sera plus rapide
SELECT DISTINCT ville FROM clients;

-- PostgreSQL peut utiliser un "Index-Only Scan" + Unique
```

### EXPLAIN pour comprendre

```sql
EXPLAIN ANALYZE  
SELECT DISTINCT ville FROM clients;  
```

**Plans d'exécution possibles :**

**1. HashAggregate (rapide, utilise la mémoire) :**
```
HashAggregate  (cost=... rows=10)
  ->  Seq Scan on clients  (cost=...)
```

**2. Sort + Unique (si pas assez de mémoire) :**
```
Unique  (cost=...)
  ->  Sort  (cost=...)
        ->  Seq Scan on clients  (cost=...)
```

**3. Index-Only Scan + Unique (optimal avec index) :**
```
Unique  (cost=...)
  ->  Index Only Scan using idx_clients_ville on clients
```

### Limiter DISTINCT à ce qui est nécessaire

```sql
-- ❌ Inefficace : DISTINCT sur beaucoup de colonnes
SELECT DISTINCT * FROM large_table;

-- ✅ Efficace : DISTINCT uniquement sur les colonnes nécessaires
SELECT DISTINCT categorie FROM large_table;

-- ❌ Inefficace : DISTINCT inutile
SELECT DISTINCT id FROM clients;  -- id est unique, DISTINCT ne sert à rien

-- ✅ Efficace : Pas de DISTINCT si la colonne est déjà unique
SELECT id FROM clients;
```

---

## Pièges et erreurs courantes

### 1. DISTINCT avec ORDER BY sur colonne non sélectionnée

```sql
-- ❌ ERREUR (sur certains SGBD, pas PostgreSQL)
SELECT DISTINCT ville  
FROM clients  
ORDER BY date_inscription;  
-- ERROR: for SELECT DISTINCT, ORDER BY expressions must appear in select list

-- PostgreSQL autorise cela, mais c'est ambigu
-- Quelle date_inscription choisir quand plusieurs clients ont la même ville ?

-- ✅ Solution : inclure la colonne de tri
SELECT DISTINCT ville, date_inscription  
FROM clients  
ORDER BY date_inscription;  
```

### 2. Confondre DISTINCT et DISTINCT ON

```sql
-- DISTINCT : élimine les doublons sur TOUTES les colonnes
SELECT DISTINCT ville, nom FROM clients;
-- Retourne les combinaisons (ville, nom) uniques

-- DISTINCT ON : élimine les doublons sur ville uniquement
SELECT DISTINCT ON (ville) ville, nom FROM clients ORDER BY ville;
-- Retourne UNE ligne par ville (avec le nom de l'un des clients)
```

### 3. DISTINCT sur des colonnes avec NULL

```sql
-- Les NULL sont traités comme égaux
SELECT DISTINCT telephone FROM clients;

-- Si 100 clients ont telephone = NULL
-- Une seule ligne NULL sera retournée
```

### 4. DISTINCT inutile

```sql
-- ❌ DISTINCT inutile si la colonne est PRIMARY KEY ou UNIQUE
SELECT DISTINCT id FROM clients;  -- id est déjà unique !

-- ✅ Pas besoin de DISTINCT
SELECT id FROM clients;

-- ❌ DISTINCT inutile si la requête retourne déjà des valeurs uniques
SELECT DISTINCT departement, SUM(salaire)  
FROM employes  
GROUP BY departement;  
-- GROUP BY garantit déjà l'unicité de departement, DISTINCT ne sert à rien
```

### 5. DISTINCT avec agrégations

```sql
-- ❌ Confusion : DISTINCT ne s'applique pas aux agrégations
SELECT DISTINCT departement, COUNT(*)  
FROM employes  
GROUP BY departement;  
-- DISTINCT ici n'a aucun effet, GROUP BY garantit déjà l'unicité

-- ✅ Si vous voulez compter les valeurs distinctes
SELECT departement, COUNT(DISTINCT poste)  
FROM employes  
GROUP BY departement;  
```

### 6. DISTINCT ON sans ORDER BY approprié

```sql
-- ⚠️ Résultat imprévisible
SELECT DISTINCT ON (produit) produit, vendeur, montant  
FROM ventes;  
-- Quelle ligne sera conservée pour chaque produit ? Imprévisible !

-- ✅ Toujours spécifier ORDER BY
SELECT DISTINCT ON (produit) produit, vendeur, montant  
FROM ventes  
ORDER BY produit, montant DESC;  
-- Maintenant on sait : la vente la plus élevée pour chaque produit
```

---

## Exemples pratiques complets

### Exemple 1 : Rapport de ventes par produit

```sql
-- Obtenir, pour chaque produit :
-- - Le nombre total de ventes
-- - La vente la plus élevée
-- - Le dernier vendeur

-- Étape 1 : Agrégation classique
SELECT
    produit,
    COUNT(*) as nb_ventes,
    MAX(montant) as meilleure_vente
FROM ventes  
GROUP BY produit;  

-- Résultat :
-- produit  | nb_ventes | meilleure_vente
-- Laptop   | 3         | 1300
-- Mouse    | 2         | 30

-- Étape 2 : Ajouter le dernier vendeur (nécessite DISTINCT ON)
WITH derniere_vente AS (
    SELECT DISTINCT ON (produit)
        produit,
        vendeur as dernier_vendeur,
        date_vente
    FROM ventes
    ORDER BY produit, date_vente DESC
),
stats AS (
    SELECT
        produit,
        COUNT(*) as nb_ventes,
        MAX(montant) as meilleure_vente
    FROM ventes
    GROUP BY produit
)
SELECT
    s.produit,
    s.nb_ventes,
    s.meilleure_vente,
    dv.dernier_vendeur
FROM stats s  
JOIN derniere_vente dv ON s.produit = dv.produit;  
```

**Résultat :**
```
produit  | nb_ventes | meilleure_vente | dernier_vendeur
---------|-----------|-----------------|----------------
Laptop   | 3         | 1300            | Charlie  
Mouse    | 2         | 30              | Bob  
Keyboard | 1         | 80              | Alice  
```

### Exemple 2 : Déduplication d'emails

```sql
-- Table avec doublons
CREATE TABLE emails_importes (
    email VARCHAR(100),
    nom VARCHAR(100),
    date_import TIMESTAMP
);

-- Garder l'import le plus récent pour chaque email
SELECT DISTINCT ON (email)
    email,
    nom,
    date_import
FROM emails_importes  
ORDER BY email, date_import DESC;  

-- Ou insérer dans une table propre
INSERT INTO emails_uniques (email, nom)  
SELECT DISTINCT ON (email)  
    email,
    nom
FROM emails_importes  
ORDER BY email, date_import DESC  
ON CONFLICT (email) DO NOTHING;  
```

### Exemple 3 : Dashboard avec dernières valeurs

```sql
-- Pour chaque capteur, afficher la dernière mesure
SELECT DISTINCT ON (capteur_id)
    capteur_id,
    nom_capteur,
    valeur,
    timestamp,
    CASE
        WHEN valeur > seuil_alerte THEN 'ALERTE'
        WHEN valeur > seuil_warning THEN 'ATTENTION'
        ELSE 'OK'
    END as statut
FROM mesures m  
JOIN capteurs c ON m.capteur_id = c.id  
ORDER BY capteur_id, timestamp DESC;  
```

---

## Alternatives à DISTINCT

### 1. GROUP BY pour agrégations

```sql
-- Au lieu de
SELECT DISTINCT ville FROM clients;

-- Si vous voulez aussi compter
SELECT ville, COUNT(*) FROM clients GROUP BY ville;
```

### 2. Window Functions pour contexte

```sql
-- Au lieu de DISTINCT ON pour obtenir le top 1
SELECT DISTINCT ON (categorie) categorie, nom_produit, prix  
FROM produits  
ORDER BY categorie, prix DESC;  

-- Avec window function (plus flexible)
SELECT *  
FROM (  
    SELECT
        categorie,
        nom_produit,
        prix,
        ROW_NUMBER() OVER (PARTITION BY categorie ORDER BY prix DESC) as rang
    FROM produits
) sub
WHERE rang = 1;
```

### 3. EXISTS pour vérification d'existence

```sql
-- Au lieu de compter les DISTINCT
SELECT COUNT(DISTINCT ville) FROM clients WHERE pays = 'France';

-- Si vous voulez juste savoir s'il y a au moins une ville
SELECT EXISTS (
    SELECT 1 FROM clients WHERE pays = 'France'
);
```

---

## Récapitulatif des différences

| Aspect | DISTINCT | DISTINCT ON | GROUP BY |
|--------|----------|-------------|----------|
| **Standard SQL** | ✅ Oui | ❌ Non (PostgreSQL) | ✅ Oui |
| **Élimine doublons** | ✅ Oui (toutes colonnes) | ✅ Oui (colonnes ON) | ✅ Oui (colonnes GROUP BY) |
| **Agrégations** | ❌ Non | ❌ Non | ✅ Oui (COUNT, SUM, etc.) |
| **Contrôle ligne conservée** | ❌ Non | ✅ Oui (avec ORDER BY) | ❌ Non (sauf avec agrégation) |
| **Syntaxe** | Simple | Moyenne | Moyenne |
| **Cas d'usage** | Liste unique simple | Top 1 par groupe | Statistiques par groupe |

---

## Points clés à retenir

1. **DISTINCT élimine les doublons**
   - Compare toutes les colonnes du SELECT
   - Garde une seule occurrence de chaque combinaison unique

2. **DISTINCT s'applique à toutes les colonnes**  
   - `SELECT DISTINCT col1, col2` → unicité sur (col1, col2)

3. **NULL sont considérés égaux**
   - Plusieurs NULL deviennent un seul NULL avec DISTINCT

4. **COUNT(DISTINCT) compte les valeurs uniques**
   - Ignore les NULL
   - Utile pour statistiques

5. **DISTINCT ON est spécifique à PostgreSQL**
   - Garde une seule ligne par groupe
   - Nécessite ORDER BY pour être prévisible
   - Très puissant pour "top 1 par groupe"

6. **ORDER BY doit commencer par les colonnes de DISTINCT ON**
   - Les expressions DISTINCT ON doivent correspondre au début de ORDER BY

7. **DISTINCT a un coût**
   - Nécessite tri ou hashage
   - Index peuvent optimiser
   - N'utilisez pas si déjà unique

8. **DISTINCT vs GROUP BY**
   - DISTINCT : valeurs uniques simples
   - GROUP BY : regroupement avec agrégations

9. **Évitez DISTINCT inutile**
   - Sur PRIMARY KEY ou UNIQUE
   - Après GROUP BY (déjà unique)

10. **Toujours ORDER BY avec DISTINCT ON**
    - Sinon résultat imprévisible
    - Spécifier quelle ligne garder

---

## Conclusion

`DISTINCT` est un outil simple mais puissant pour éliminer les doublons dans vos résultats. Sa syntaxe est intuitive et son comportement prévisible : il compare toutes les colonnes sélectionnées et garde une seule occurrence de chaque combinaison unique.

`DISTINCT ON`, extension PostgreSQL, va plus loin en permettant de contrôler précisément quelle ligne conserver dans chaque groupe. C'est particulièrement utile pour obtenir "la dernière valeur", "le meilleur prix", ou "la première occurrence" par catégorie.

Quelques conseils finaux :
- Utilisez DISTINCT quand vous avez besoin de valeurs uniques simples
- Utilisez GROUP BY quand vous avez besoin d'agrégations
- Utilisez DISTINCT ON pour le "top 1" par groupe
- Optimisez avec des index sur les colonnes concernées
- Évitez DISTINCT quand il est inutile (colonnes déjà uniques)

Avec ces connaissances, vous pouvez maintenant éliminer efficacement les doublons et construire des requêtes qui retournent exactement les données dont vous avez besoin.

---


⏭️ [Manipulation des Données (DML)](/06-manipulation-des-donnees/README.md)
