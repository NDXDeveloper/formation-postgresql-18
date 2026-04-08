🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4. Types de Jointures : INNER, LEFT/RIGHT/FULL OUTER, CROSS

## Introduction

Les **jointures** sont le cœur du langage SQL et des bases de données relationnelles. Elles permettent de **combiner des données provenant de plusieurs tables** en une seule requête, exploitant ainsi les relations entre les tables.

PostgreSQL offre plusieurs types de jointures, chacune répondant à des besoins spécifiques :
- **INNER JOIN** : Seulement les correspondances  
- **LEFT JOIN** : Tout ce qui est à gauche + correspondances  
- **RIGHT JOIN** : Tout ce qui est à droite + correspondances  
- **FULL OUTER JOIN** : Tout ce qui existe dans les deux tables  
- **CROSS JOIN** : Toutes les combinaisons possibles

Dans ce chapitre, nous allons explorer chaque type en détail avec des exemples concrets et des visualisations.

---

## Tables d'Exemple

Pour illustrer les différents types de jointures, nous utiliserons ces deux tables simples :

### Table `clients`

| id | nom      | ville      |
|----|----------|------------|
| 1  | Alice    | Paris      |
| 2  | Bob      | Lyon       |
| 3  | Charlie  | Marseille  |
| 4  | Diana    | Toulouse   |

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    ville VARCHAR(100)
);

INSERT INTO clients (nom, ville) VALUES
    ('Alice', 'Paris'),
    ('Bob', 'Lyon'),
    ('Charlie', 'Marseille'),
    ('Diana', 'Toulouse');
```

### Table `commandes`

| id | client_id | montant | date_commande |
|----|-----------|---------|---------------|
| 101 | 1        | 150.00  | 2025-01-10    |
| 102 | 1        | 220.00  | 2025-01-15    |
| 103 | 2        | 99.50   | 2025-01-12    |
| 104 | 5        | 175.00  | 2025-01-18    |

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER,
    montant NUMERIC(10, 2) NOT NULL,
    date_commande DATE NOT NULL
);

INSERT INTO commandes (client_id, montant, date_commande) VALUES
    (1, 150.00, '2025-01-10'),
    (1, 220.00, '2025-01-15'),
    (2, 99.50, '2025-01-12'),
    (5, 175.00, '2025-01-18');  -- Client 5 n'existe pas !
```

**Points importants à noter** :
- Alice (id=1) a **2 commandes**
- Bob (id=2) a **1 commande**
- Charlie (id=3) et Diana (id=4) n'ont **aucune commande**
- Il existe une commande (id=104) pour un client_id=5 qui **n'existe pas** dans la table clients

Ces situations permettront d'illustrer les différences entre les types de jointures.

---

## 1. INNER JOIN - La Jointure Interne

### Concept

L'**INNER JOIN** (ou simplement **JOIN**) retourne **uniquement les lignes qui ont une correspondance dans les deux tables**. C'est l'intersection entre deux ensembles.

### Diagramme de Venn

```
    Table A          Table B
    ┌─────────┐    ┌─────────┐
    │         │    │         │
    │    ┌────┼────┼────┐    │
    │    │    │    │    │    │
    │    │  ███████████ │    │
    │    │    │    │    │    │
    │    └────┼────┼────┘    │
    │         │    │         │
    └─────────┘    └─────────┘
         ███ = INNER JOIN (intersection)
```

### Syntaxe

```sql
SELECT colonnes  
FROM table_gauche  
INNER JOIN table_droite ON condition_jointure;  

-- "INNER" est optionnel, on peut écrire simplement :
SELECT colonnes  
FROM table_gauche  
JOIN table_droite ON condition_jointure;  
```

### Exemple avec nos tables

```sql
SELECT
    clients.id,
    clients.nom,
    commandes.id AS commande_id,
    commandes.montant,
    commandes.date_commande
FROM clients  
INNER JOIN commandes ON clients.id = commandes.client_id;  
```

### Résultat

| id | nom   | commande_id | montant | date_commande |
|----|-------|-------------|---------|---------------|
| 1  | Alice | 101         | 150.00  | 2025-01-10    |
| 1  | Alice | 102         | 220.00  | 2025-01-15    |
| 2  | Bob   | 103         | 99.50   | 2025-01-12    |

**Observations** :
- ✅ Alice apparaît **2 fois** (elle a 2 commandes)  
- ✅ Bob apparaît **1 fois** (il a 1 commande)  
- ❌ Charlie et Diana **n'apparaissent pas** (aucune commande)  
- ❌ La commande 104 (client_id=5) **n'apparaît pas** (client inexistant)

### Interprétation

L'INNER JOIN ne garde que les lignes où :
```
clients.id = commandes.client_id
```

C'est-à-dire : **Seulement les clients qui ont passé des commandes**.

### Cas d'usage typiques

1. **Afficher les relations existantes** : "Lister tous les clients qui ont commandé"  
2. **Éviter les orphelins** : Exclure les données sans correspondance  
3. **Analyses sur données complètes** : Calculer des statistiques sur des relations confirmées

### Exemple : Calculer le total des commandes par client

```sql
SELECT
    clients.nom,
    COUNT(commandes.id) AS nombre_commandes,
    SUM(commandes.montant) AS total_depense
FROM clients  
INNER JOIN commandes ON clients.id = commandes.client_id  
GROUP BY clients.id, clients.nom;  
```

**Résultat** :

| nom   | nombre_commandes | total_depense |
|-------|------------------|---------------|
| Alice | 2                | 370.00        |
| Bob   | 1                | 99.50         |

Charlie et Diana n'apparaissent pas car ils n'ont pas de commandes.

---

## 2. LEFT JOIN (LEFT OUTER JOIN) - La Jointure Externe Gauche

### Concept

Le **LEFT JOIN** retourne **toutes les lignes de la table de gauche**, qu'il y ait ou non une correspondance dans la table de droite. S'il n'y a pas de correspondance, les colonnes de la table de droite seront remplies avec `NULL`.

### Diagramme de Venn

```
    Table A          Table B
    ┌─────────┐    ┌─────────┐
    │         │    │         │
    │████┌────┼────┼────┐    │
    │████│    │    │    │    │
    │████│  ███████████ │    │
    │████│    │    │    │    │
    │████└────┼────┼────┘    │
    │         │    │         │
    └─────────┘    └─────────┘
         ████ = LEFT JOIN (tout A + intersection)
```

### Syntaxe

```sql
SELECT colonnes  
FROM table_gauche  
LEFT JOIN table_droite ON condition_jointure;  

-- Équivalent à :
SELECT colonnes  
FROM table_gauche  
LEFT OUTER JOIN table_droite ON condition_jointure;  
```

**Note** : `OUTER` est optionnel et rarement utilisé.

### Exemple avec nos tables

```sql
SELECT
    clients.id,
    clients.nom,
    clients.ville,
    commandes.id AS commande_id,
    commandes.montant,
    commandes.date_commande
FROM clients  
LEFT JOIN commandes ON clients.id = commandes.client_id;  
```

### Résultat

| id | nom      | ville      | commande_id | montant | date_commande |
|----|----------|------------|-------------|---------|---------------|
| 1  | Alice    | Paris      | 101         | 150.00  | 2025-01-10    |
| 1  | Alice    | Paris      | 102         | 220.00  | 2025-01-15    |
| 2  | Bob      | Lyon       | 103         | 99.50   | 2025-01-12    |
| 3  | Charlie  | Marseille  | NULL        | NULL    | NULL          |
| 4  | Diana    | Toulouse   | NULL        | NULL    | NULL          |

**Observations** :
- ✅ **Tous les clients** apparaissent (même ceux sans commande)  
- ✅ Alice et Bob ont leurs commandes  
- ✅ Charlie et Diana apparaissent avec `NULL` pour les colonnes de commandes  
- ❌ La commande 104 (client_id=5) n'apparaît toujours pas (pas de LEFT JOIN depuis commandes)

### Interprétation

Le LEFT JOIN garantit : **Toutes les lignes de la table de gauche sont présentes**.

### Cas d'usage typiques

1. **Identifier les absences** : "Quels clients n'ont jamais commandé ?"  
2. **Rapports complets** : Afficher tous les éléments, même sans données associées  
3. **Analyses inclusives** : Inclure les éléments sans relation

### Exemple : Trouver les clients sans commande

```sql
SELECT
    clients.id,
    clients.nom
FROM clients  
LEFT JOIN commandes ON clients.id = commandes.client_id  
WHERE commandes.id IS NULL;  
```

**Résultat** :

| id | nom     |
|----|---------|
| 3  | Charlie |
| 4  | Diana   |

**Explication** : On fait un LEFT JOIN puis on filtre les lignes où `commandes.id IS NULL`, c'est-à-dire les clients sans commande.

### Exemple : Statistiques avec tous les clients

```sql
SELECT
    clients.nom,
    COALESCE(COUNT(commandes.id), 0) AS nombre_commandes,
    COALESCE(SUM(commandes.montant), 0) AS total_depense
FROM clients  
LEFT JOIN commandes ON clients.id = commandes.client_id  
GROUP BY clients.id, clients.nom  
ORDER BY total_depense DESC;  
```

**Résultat** :

| nom      | nombre_commandes | total_depense |
|----------|------------------|---------------|
| Alice    | 2                | 370.00        |
| Bob      | 1                | 99.50         |
| Charlie  | 0                | 0.00          |
| Diana    | 0                | 0.00          |

**Note** : Utilisation de `COALESCE()` pour remplacer `NULL` par 0.

---

## 3. RIGHT JOIN (RIGHT OUTER JOIN) - La Jointure Externe Droite

### Concept

Le **RIGHT JOIN** retourne **toutes les lignes de la table de droite**, qu'il y ait ou non une correspondance dans la table de gauche. S'il n'y a pas de correspondance, les colonnes de la table de gauche seront remplies avec `NULL`.

C'est l'inverse symétrique du LEFT JOIN.

### Diagramme de Venn

```
    Table A          Table B
    ┌─────────┐    ┌─────────┐
    │         │    │         │
    │    ┌────┼────┼────┐████│
    │    │    │    │    │████│
    │    │  ███████████ │████│
    │    │    │    │    │████│
    │    └────┼────┼────┘████│
    │         │    │         │
    └─────────┘    └─────────┘
         ████ = RIGHT JOIN (tout B + intersection)
```

### Syntaxe

```sql
SELECT colonnes  
FROM table_gauche  
RIGHT JOIN table_droite ON condition_jointure;  

-- Équivalent à :
SELECT colonnes  
FROM table_gauche  
RIGHT OUTER JOIN table_droite ON condition_jointure;  
```

### Exemple avec nos tables

```sql
SELECT
    clients.id AS client_id,
    clients.nom,
    commandes.id AS commande_id,
    commandes.montant,
    commandes.date_commande
FROM clients  
RIGHT JOIN commandes ON clients.id = commandes.client_id;  
```

### Résultat

| client_id | nom   | commande_id | montant | date_commande |
|-----------|-------|-------------|---------|---------------|
| 1         | Alice | 101         | 150.00  | 2025-01-10    |
| 1         | Alice | 102         | 220.00  | 2025-01-15    |
| 2         | Bob   | 103         | 99.50   | 2025-01-12    |
| NULL      | NULL  | 104         | 175.00  | 2025-01-18    |

**Observations** :
- ✅ **Toutes les commandes** apparaissent  
- ✅ Alice et Bob ont leurs informations  
- ✅ La commande 104 apparaît avec `NULL` pour le client (client_id=5 n'existe pas)  
- ❌ Charlie et Diana n'apparaissent pas (ils n'ont pas de commande)

### Interprétation

Le RIGHT JOIN garantit : **Toutes les lignes de la table de droite sont présentes**.

### Équivalence avec LEFT JOIN

**Important** : `A RIGHT JOIN B` est strictement équivalent à `B LEFT JOIN A`.

```sql
-- Ces deux requêtes sont identiques :

-- Version 1 : RIGHT JOIN
SELECT * FROM clients RIGHT JOIN commandes ON clients.id = commandes.client_id;

-- Version 2 : LEFT JOIN (inversé)
SELECT * FROM commandes LEFT JOIN clients ON commandes.client_id = clients.id;
```

**Convention** : En pratique, on préfère utiliser **LEFT JOIN** car c'est plus intuitif (on lit de gauche à droite). Le RIGHT JOIN est moins utilisé.

### Cas d'usage

1. **Identifier les orphelins** : "Quelles commandes ont un client_id invalide ?"  
2. **Audit d'intégrité** : Détecter les références brisées  
3. **Rapports depuis la table de détail** : Partir des commandes pour remonter aux clients

### Exemple : Trouver les commandes sans client valide

```sql
SELECT
    commandes.id AS commande_id,
    commandes.client_id,
    commandes.montant
FROM clients  
RIGHT JOIN commandes ON clients.id = commandes.client_id  
WHERE clients.id IS NULL;  
```

**Résultat** :

| commande_id | client_id | montant |
|-------------|-----------|---------|
| 104         | 5         | 175.00  |

Cette requête révèle un problème d'intégrité : une commande référence un client qui n'existe pas !

---

## 4. FULL OUTER JOIN - La Jointure Externe Complète

### Concept

Le **FULL OUTER JOIN** retourne **toutes les lignes des deux tables**, qu'il y ait ou non une correspondance. C'est l'union des résultats de LEFT et RIGHT JOIN.

### Diagramme de Venn

```
    Table A          Table B
    ┌─────────┐    ┌─────────┐
    │         │    │         │
    │████┌────┼────┼────┐████│
    │████│    │    │    │████│
    │████│  ███████████ │████│
    │████│    │    │    │████│
    │████└────┼────┼────┘████│
    │         │    │         │
    └─────────┘    └─────────┘
         ████ = FULL OUTER JOIN (union complète)
```

### Syntaxe

```sql
SELECT colonnes  
FROM table_gauche  
FULL OUTER JOIN table_droite ON condition_jointure;  

-- "OUTER" est optionnel :
SELECT colonnes  
FROM table_gauche  
FULL JOIN table_droite ON condition_jointure;  
```

### Exemple avec nos tables

```sql
SELECT
    clients.id AS client_id,
    clients.nom,
    clients.ville,
    commandes.id AS commande_id,
    commandes.montant,
    commandes.date_commande
FROM clients  
FULL OUTER JOIN commandes ON clients.id = commandes.client_id  
ORDER BY clients.id, commandes.id;  
```

### Résultat

| client_id | nom      | ville      | commande_id | montant | date_commande |
|-----------|----------|------------|-------------|---------|---------------|
| 1         | Alice    | Paris      | 101         | 150.00  | 2025-01-10    |
| 1         | Alice    | Paris      | 102         | 220.00  | 2025-01-15    |
| 2         | Bob      | Lyon       | 103         | 99.50   | 2025-01-12    |
| 3         | Charlie  | Marseille  | NULL        | NULL    | NULL          |
| 4         | Diana    | Toulouse   | NULL        | NULL    | NULL          |
| NULL      | NULL     | NULL       | 104         | 175.00  | 2025-01-18    |

**Observations** :
- ✅ **Tous les clients** apparaissent (même sans commande)  
- ✅ **Toutes les commandes** apparaissent (même sans client valide)  
- ✅ Charlie et Diana ont `NULL` pour les commandes  
- ✅ La commande 104 a `NULL` pour le client

### Interprétation

Le FULL OUTER JOIN garantit : **Aucune donnée n'est perdue, tout est visible**.

### Cas d'usage typiques

1. **Audit complet** : Voir toutes les données, correspondances et orphelins  
2. **Réconciliation** : Comparer deux sources de données  
3. **Détection d'anomalies** : Identifier les incohérences dans les deux sens

### Exemple : Rapport complet d'audit

```sql
SELECT
    COALESCE(clients.id, commandes.client_id) AS id,
    clients.nom,
    commandes.id AS commande_id,
    CASE
        WHEN clients.id IS NULL THEN 'Commande orpheline'
        WHEN commandes.id IS NULL THEN 'Client sans commande'
        ELSE 'OK'
    END AS statut
FROM clients  
FULL OUTER JOIN commandes ON clients.id = commandes.client_id;  
```

**Résultat** :

| id | nom      | commande_id | statut                 |
|----|----------|-------------|------------------------|
| 1  | Alice    | 101         | OK                     |
| 1  | Alice    | 102         | OK                     |
| 2  | Bob      | 103         | OK                     |
| 3  | Charlie  | NULL        | Client sans commande   |
| 4  | Diana    | NULL        | Client sans commande   |
| 5  | NULL     | 104         | Commande orpheline     |

---

## 5. CROSS JOIN - Le Produit Cartésien

### Concept

Le **CROSS JOIN** retourne le **produit cartésien** des deux tables : toutes les combinaisons possibles entre chaque ligne de la table de gauche et chaque ligne de la table de droite.

**Attention** : Aucune condition de jointure n'est nécessaire (et ne doit pas être spécifiée).

### Diagramme

```
Table A (3 lignes) × Table B (2 lignes) = 6 lignes

A1 ─┬─ B1
    └─ B2
A2 ─┬─ B1
    └─ B2
A3 ─┬─ B1
    └─ B2
```

### Syntaxe

```sql
SELECT colonnes  
FROM table_gauche  
CROSS JOIN table_droite;  

-- Ancienne syntaxe (équivalente) :
SELECT colonnes  
FROM table_gauche, table_droite;  
```

### Exemple avec nos tables

```sql
SELECT
    clients.nom AS client,
    commandes.id AS commande_id,
    commandes.montant
FROM clients  
CROSS JOIN commandes;  
```

### Résultat (16 lignes !)

4 clients × 4 commandes = **16 lignes**

| client   | commande_id | montant |
|----------|-------------|---------|
| Alice    | 101         | 150.00  |
| Alice    | 102         | 220.00  |
| Alice    | 103         | 99.50   |
| Alice    | 104         | 175.00  |
| Bob      | 101         | 150.00  |
| Bob      | 102         | 220.00  |
| Bob      | 103         | 99.50   |
| Bob      | 104         | 175.00  |
| Charlie  | 101         | 150.00  |
| Charlie  | 102         | 220.00  |
| Charlie  | 103         | 99.50   |
| Charlie  | 104         | 175.00  |
| Diana    | 101         | 150.00  |
| Diana    | 102         | 220.00  |
| Diana    | 103         | 99.50   |
| Diana    | 104         | 175.00  |

**Observations** :
- Chaque client est combiné avec chaque commande
- Les correspondances logiques (client_id) sont ignorées
- Le résultat peut être très volumineux !

### Danger : Explosion Combinatoire

```
Table A : 1 000 lignes  
Table B : 1 000 lignes  
CROSS JOIN : 1 000 000 lignes ! 💥  

Table A : 10 000 lignes  
Table B : 10 000 lignes  
CROSS JOIN : 100 000 000 lignes ! 💥💥  
```

### Cas d'usage légitimes

#### 1. Générer des Combinaisons

Créer un catalogue produit avec toutes les variantes :

```sql
CREATE TABLE tailles (nom VARCHAR(10));  
CREATE TABLE couleurs (nom VARCHAR(20));  

INSERT INTO tailles VALUES ('S'), ('M'), ('L'), ('XL');  
INSERT INTO couleurs VALUES ('Rouge'), ('Bleu'), ('Noir');  

-- Générer toutes les combinaisons taille × couleur
SELECT
    tailles.nom AS taille,
    couleurs.nom AS couleur
FROM tailles  
CROSS JOIN couleurs  
ORDER BY tailles.nom, couleurs.nom;  
```

**Résultat** : 4 × 3 = 12 combinaisons

| taille | couleur |
|--------|---------|
| L      | Bleu    |
| L      | Noir    |
| L      | Rouge   |
| M      | Bleu    |
| M      | Noir    |
| M      | Rouge   |
| S      | Bleu    |
| S      | Noir    |
| S      | Rouge   |
| XL     | Bleu    |
| XL     | Noir    |
| XL     | Rouge   |

#### 2. Générer des Séries de Dates

Créer une ligne par jour et par magasin :

```sql
SELECT
    magasins.nom,
    dates.jour
FROM magasins  
CROSS JOIN generate_series(  
    '2025-01-01'::date,
    '2025-01-07'::date,
    '1 day'::interval
) AS dates(jour)
ORDER BY magasins.nom, dates.jour;
```

#### 3. Matrices de Tests

Générer toutes les combinaisons de paramètres de test :

```sql
CREATE TABLE navigateurs (nom VARCHAR(50));  
CREATE TABLE systemes (nom VARCHAR(50));  

INSERT INTO navigateurs VALUES ('Chrome'), ('Firefox'), ('Safari');  
INSERT INTO systemes VALUES ('Windows'), ('macOS'), ('Linux');  

-- Générer toutes les combinaisons pour tests
SELECT
    navigateurs.nom AS navigateur,
    systemes.nom AS systeme
FROM navigateurs  
CROSS JOIN systemes;  
```

**Résultat** : 3 × 3 = 9 combinaisons de tests

---

## 6. Comparaison des Jointures

### Tableau Récapitulatif

| Jointure | Lignes de A | Lignes de B | Correspondances | Résultat |
|----------|-------------|-------------|-----------------|----------|
| **INNER** | Avec correspondance | Avec correspondance | Oui | Intersection |
| **LEFT** | Toutes | Avec correspondance | Oui (NULL si absent) | Tout A |
| **RIGHT** | Avec correspondance | Toutes | Oui (NULL si absent) | Tout B |
| **FULL** | Toutes | Toutes | Oui (NULL si absent) | Union |
| **CROSS** | Toutes | Toutes | Non | Produit cartésien |

### Nombre de Lignes Résultantes

Avec nos tables (4 clients, 4 commandes, dont 3 avec correspondance) :

| Jointure | Nombre de lignes | Explication |
|----------|------------------|-------------|
| INNER JOIN | 3 | Seulement Alice (×2) et Bob |
| LEFT JOIN | 5 | 4 clients + Alice en double |
| RIGHT JOIN | 4 | 4 commandes |
| FULL JOIN | 6 | 4 clients + 1 commande orpheline + Alice en double |
| CROSS JOIN | 16 | 4 × 4 = 16 |

### Diagrammes de Venn Comparés

```
INNER JOIN              LEFT JOIN               RIGHT JOIN
┌───┐   ┌───┐          ┌───┐   ┌───┐          ┌───┐   ┌───┐
│   │   │   │          │███│   │   │          │   │   │███│
│ ┌─┼───┼─┐ │          │███┼───┼─┐ │          │ ┌─┼───┼███│
│ │█│███│█│ │          │███│███│█│ │          │ │█│███│███│
│ └─┼───┼─┘ │          │███┼───┼─┘ │          │ └─┼───┼███│
│   │   │   │          │███│   │   │          │   │   │███│
└───┘   └───┘          └───┘   └───┘          └───┘   └───┘

FULL OUTER JOIN         CROSS JOIN
┌───┐   ┌───┐          Toutes les combinaisons
│███│   │███│          A1×B1, A1×B2, A1×B3...
│███┼───┼███│          A2×B1, A2×B2, A2×B3...
│███│███│███│          A3×B1, A3×B2, A3×B3...
│███┼───┼███│          (Produit cartésien complet)
│███│   │███│
└───┘   └───┘
```

---

## 7. Choix du Type de Jointure : Guide de Décision

### Arbre de Décision

```
Question : Que voulez-vous obtenir ?
│
├─ Seulement les correspondances ?
│  └─ ✅ INNER JOIN
│
├─ Tout ce qui est dans la table principale + correspondances ?
│  ├─ Table principale = gauche ? → ✅ LEFT JOIN
│  └─ Table principale = droite ? → ✅ RIGHT JOIN
│
├─ Absolument tout ce qui existe dans les deux tables ?
│  └─ ✅ FULL OUTER JOIN
│
└─ Toutes les combinaisons possibles ?
   └─ ✅ CROSS JOIN (attention à la taille !)
```

### Exemples de Questions → Type de Jointure

| Question | Jointure | Raison |
|----------|----------|--------|
| "Quels clients ont commandé ?" | INNER JOIN | Seulement ceux qui ont commandé |
| "Liste de tous les clients avec leurs commandes (s'ils en ont)" | LEFT JOIN | Tous les clients, commandes optionnelles |
| "Toutes les commandes avec infos client" | LEFT JOIN | Depuis commandes vers clients |
| "Audit complet : tous les clients ET toutes les commandes" | FULL JOIN | Aucune perte d'information |
| "Générer toutes les combinaisons produit × option" | CROSS JOIN | Besoin du produit cartésien |

---

## 8. Exemples Pratiques Avancés

### Exemple 1 : Clients avec et sans Commandes

**Objectif** : Afficher tous les clients avec un indicateur "a commandé" ou "n'a pas commandé"

```sql
SELECT
    clients.nom,
    clients.ville,
    CASE
        WHEN commandes.id IS NOT NULL THEN 'Oui'
        ELSE 'Non'
    END AS a_commande
FROM clients  
LEFT JOIN commandes ON clients.id = commandes.client_id  
GROUP BY clients.id, clients.nom, clients.ville  
ORDER BY clients.nom;  
```

**Résultat** :

| nom      | ville      | a_commande |
|----------|------------|------------|
| Alice    | Paris      | Oui        |
| Bob      | Lyon       | Oui        |
| Charlie  | Marseille  | Non        |
| Diana    | Toulouse   | Non        |

### Exemple 2 : Top Clients par Dépense (Incluant les Zéros)

```sql
SELECT
    clients.nom,
    COALESCE(SUM(commandes.montant), 0) AS total_depense,
    COUNT(commandes.id) AS nombre_commandes
FROM clients  
LEFT JOIN commandes ON clients.id = commandes.client_id  
GROUP BY clients.id, clients.nom  
ORDER BY total_depense DESC;  
```

**Résultat** :

| nom      | total_depense | nombre_commandes |
|----------|---------------|------------------|
| Alice    | 370.00        | 2                |
| Bob      | 99.50         | 1                |
| Charlie  | 0.00          | 0                |
| Diana    | 0.00          | 0                |

### Exemple 3 : Détecter les Incohérences

**Objectif** : Trouver toutes les anomalies (clients sans commande ET commandes sans client)

```sql
SELECT
    COALESCE(clients.nom, 'CLIENT INEXISTANT') AS client,
    COALESCE(commandes.id::TEXT, 'Aucune commande') AS commande,
    CASE
        WHEN clients.id IS NULL THEN '⚠️ Commande orpheline'
        WHEN commandes.id IS NULL THEN 'ℹ️ Client sans commande'
        ELSE '✅ OK'
    END AS statut
FROM clients  
FULL OUTER JOIN commandes ON clients.id = commandes.client_id  
WHERE clients.id IS NULL OR commandes.id IS NULL  
ORDER BY statut;  
```

**Résultat** :

| client              | commande        | statut                   |
|---------------------|-----------------|--------------------------|
| Charlie             | Aucune commande | ℹ️ Client sans commande |
| Diana               | Aucune commande | ℹ️ Client sans commande |
| CLIENT INEXISTANT   | 104             | ⚠️ Commande orpheline   |

### Exemple 4 : Planification de Créneaux Horaires

**Objectif** : Générer tous les créneaux de 1h entre 9h et 17h pour chaque jour de la semaine

```sql
-- Table des jours
CREATE TABLE jours_semaine (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(20)
);

INSERT INTO jours_semaine VALUES
    (1, 'Lundi'), (2, 'Mardi'), (3, 'Mercredi'),
    (4, 'Jeudi'), (5, 'Vendredi');

-- Générer tous les créneaux
SELECT
    jours.nom AS jour,
    horaires.heure::TIME AS heure
FROM jours_semaine jours  
CROSS JOIN generate_series(  
    '09:00'::TIME,
    '17:00'::TIME,
    '1 hour'::INTERVAL
) AS horaires(heure)
ORDER BY jours.id, horaires.heure;
```

**Résultat** : 5 jours × 9 heures = 45 créneaux

| jour      | heure    |
|-----------|----------|
| Lundi     | 09:00:00 |
| Lundi     | 10:00:00 |
| Lundi     | 11:00:00 |
| ...       | ...      |
| Vendredi  | 17:00:00 |

---

## 9. Jointures Multiples

### Chaîner Plusieurs Jointures

Vous pouvez combiner plusieurs tables en enchaînant les jointures :

```sql
SELECT colonnes  
FROM table_a  
INNER JOIN table_b ON condition_ab  
LEFT JOIN table_c ON condition_bc  
INNER JOIN table_d ON condition_cd;  
```

### Exemple : Système de Commandes Complet

**Tables** :
- `clients` : Les clients  
- `commandes` : Les commandes  
- `lignes_commande` : Détail des produits commandés  
- `produits` : Informations sur les produits

```sql
SELECT
    clients.nom AS client,
    commandes.date_commande,
    produits.nom AS produit,
    lignes_commande.quantite,
    lignes_commande.prix_unitaire,
    (lignes_commande.quantite * lignes_commande.prix_unitaire) AS sous_total
FROM clients  
INNER JOIN commandes ON clients.id = commandes.client_id  
INNER JOIN lignes_commande ON commandes.id = lignes_commande.commande_id  
INNER JOIN produits ON lignes_commande.produit_id = produits.id  
WHERE clients.id = 1  
ORDER BY commandes.date_commande;  
```

### Mélanger les Types de Jointures

```sql
-- Tous les clients, leurs commandes (si existent),
-- et les produits de ces commandes
SELECT
    clients.nom,
    commandes.id AS commande_id,
    produits.nom AS produit
FROM clients  
LEFT JOIN commandes ON clients.id = commandes.client_id  
LEFT JOIN lignes_commande ON commandes.id = lignes_commande.commande_id  
LEFT JOIN produits ON lignes_commande.produit_id = produits.id;  
```

**Important** : L'ordre des jointures compte ! Si vous faites un LEFT JOIN suivi d'un INNER JOIN, le INNER JOIN peut annuler l'effet du LEFT JOIN.

### Règle : Cohérence des Jointures

```sql
-- ✅ BON : LEFT JOIN en cascade
FROM clients  
LEFT JOIN commandes ON ...  
LEFT JOIN lignes_commande ON ...  

-- ⚠️ ATTENTION : LEFT puis INNER
FROM clients  
LEFT JOIN commandes ON ...  
INNER JOIN lignes_commande ON ...  -- Peut exclure des clients !  
```

Le INNER JOIN sur `lignes_commande` ne retournera que les lignes où `lignes_commande` existe, annulant partiellement l'effet du LEFT JOIN précédent.

---

## 10. Jointures avec Alias et Qualification

### Pourquoi Utiliser des Alias ?

1. **Clarté** : Noms plus courts  
2. **Éviter les ambiguïtés** : Quand deux tables ont des colonnes du même nom  
3. **Lisibilité** : Code plus facile à lire

### Syntaxe

```sql
SELECT
    c.nom AS nom_client,
    cmd.date_commande,
    cmd.montant
FROM clients AS c  -- Alias "c"  
INNER JOIN commandes AS cmd ON c.id = cmd.client_id;  -- Alias "cmd"  
```

**Note** : Le mot-clé `AS` est optionnel, on peut écrire simplement :

```sql
FROM clients c  
INNER JOIN commandes cmd ON c.id = cmd.client_id;  
```

### Jointures Auto-Référencées (Self-Join)

Une table peut être jointe à elle-même en utilisant des alias différents :

```sql
-- Table employés avec hiérarchie
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    manager_id INTEGER REFERENCES employes(id)
);

-- Trouver les employés avec leur manager
SELECT
    e.nom AS employe,
    m.nom AS manager
FROM employes e  
LEFT JOIN employes m ON e.manager_id = m.id;  
```

**Résultat** :

| employe | manager |
|---------|---------|
| Alice   | NULL    |
| Bob     | Alice   |
| Charlie | Alice   |
| Diana   | Bob     |

---

## 11. Performances et Optimisation

### Bonnes Pratiques pour les Jointures

#### 1. Indexer les Clés de Jointure

```sql
-- Créer des index sur les colonnes de jointure
CREATE INDEX idx_commandes_client_id ON commandes(client_id);  
CREATE INDEX idx_lignes_commande_commande_id ON lignes_commande(commande_id);  
```

**Pourquoi ?** Les jointures utilisent ces colonnes pour trouver les correspondances. Un index accélère drastiquement les recherches.

#### 2. Utiliser EXPLAIN pour Comprendre

```sql
EXPLAIN ANALYZE  
SELECT *  
FROM clients  
INNER JOIN commandes ON clients.id = commandes.client_id;  
```

Cela montre :
- Le type de jointure utilisé (Nested Loop, Hash Join, Merge Join)
- Si les index sont utilisés
- Le coût estimé et réel

#### 3. Filtrer Avant de Joindre (WHERE)

```sql
-- ⚠️ Moins efficace : Jointure puis filtrage
SELECT *  
FROM clients  
INNER JOIN commandes ON clients.id = commandes.client_id  
WHERE clients.ville = 'Paris';  

-- ✅ Plus efficace : Filtrage puis jointure (si possible)
SELECT *  
FROM (SELECT * FROM clients WHERE ville = 'Paris') c  
INNER JOIN commandes ON c.id = commandes.client_id;  
```

**Note** : L'optimiseur de PostgreSQL fait souvent cette optimisation automatiquement.

#### 4. Attention au CROSS JOIN

Évitez les CROSS JOIN accidentels :

```sql
-- ❌ Danger : Produit cartésien accidentel
SELECT * FROM clients, commandes;

-- ✅ Toujours spécifier la condition
SELECT * FROM clients INNER JOIN commandes ON clients.id = commandes.client_id;
```

#### 5. Choisir le Bon Type de Jointure

- Utilisez **INNER JOIN** si vous n'avez besoin que des correspondances
- N'utilisez pas **FULL OUTER JOIN** si un **LEFT JOIN** suffit
- Évitez les jointures inutiles

---

## 12. Pièges et Erreurs Courantes

### Erreur 1 : Colonnes Ambiguës

```sql
-- ❌ Erreur : colonne "id" est ambiguë
SELECT id, nom, montant  
FROM clients  
INNER JOIN commandes ON clients.id = commandes.client_id;  
-- ERROR: column reference "id" is ambiguous
```

**Solution** : Qualifier les colonnes

```sql
-- ✅ Correct
SELECT clients.id, clients.nom, commandes.montant  
FROM clients  
INNER JOIN commandes ON clients.id = commandes.client_id;  
```

### Erreur 2 : Oublier la Condition de Jointure

```sql
-- ❌ Produit cartésien accidentel
SELECT * FROM clients INNER JOIN commandes;
-- ERROR: syntax error at or near ";"
```

**Solution** : Toujours spécifier `ON condition`

```sql
-- ✅ Correct
SELECT * FROM clients INNER JOIN commandes ON clients.id = commandes.client_id;
```

### Erreur 3 : Confusion INNER vs LEFT JOIN

```sql
-- Avec INNER JOIN : Seulement Alice et Bob (3 lignes)
SELECT COUNT(*) FROM clients INNER JOIN commandes ON clients.id = commandes.client_id;
-- Résultat : 3

-- Avec LEFT JOIN : Tous les clients (5 lignes, Alice comptée 2 fois)
SELECT COUNT(*) FROM clients LEFT JOIN commandes ON clients.id = commandes.client_id;
-- Résultat : 5
```

**Solution** : Comprendre la différence et choisir le bon type.

### Erreur 4 : NULL dans les Conditions de Jointure

```sql
-- Si client_id est NULL, la ligne ne correspondra JAMAIS
-- même avec LEFT JOIN
SELECT *  
FROM commandes  
LEFT JOIN clients ON commandes.client_id = clients.id;  
```

**Important** : `NULL = NULL` retourne `NULL` (pas `TRUE`), donc les lignes avec `NULL` ne correspondent jamais.

### Erreur 5 : Mauvais Ordre des Jointures

```sql
-- ⚠️ Peut donner un résultat inattendu
FROM clients  
LEFT JOIN commandes ON ...  
INNER JOIN produits ON ...  -- Peut filtrer les clients sans commande !  
```

**Solution** : Utiliser LEFT JOIN en cascade si vous voulez garder tous les clients.

---

## 13. Comparaison Visuelle des Résultats

Reprenons nos tables et comparons visuellement les résultats :

### Tables de Référence

**clients** : 4 lignes
```
1-Alice, 2-Bob, 3-Charlie, 4-Diana
```

**commandes** : 4 lignes
```
101(client_id=1), 102(client_id=1), 103(client_id=2), 104(client_id=5)
```

### Résultats par Type de Jointure

```
INNER JOIN (3 lignes) :
┌─────────┬─────────────┐
│ Client  │ Commande    │
├─────────┼─────────────┤
│ Alice   │ 101         │
│ Alice   │ 102         │
│ Bob     │ 103         │
└─────────┴─────────────┘

LEFT JOIN (5 lignes) :
┌─────────┬─────────────┐
│ Client  │ Commande    │
├─────────┼─────────────┤
│ Alice   │ 101         │
│ Alice   │ 102         │
│ Bob     │ 103         │
│ Charlie │ NULL        │
│ Diana   │ NULL        │
└─────────┴─────────────┘

RIGHT JOIN (4 lignes) :
┌─────────┬─────────────┐
│ Client  │ Commande    │
├─────────┼─────────────┤
│ Alice   │ 101         │
│ Alice   │ 102         │
│ Bob     │ 103         │
│ NULL    │ 104         │
└─────────┴─────────────┘

FULL OUTER JOIN (6 lignes) :
┌─────────┬─────────────┐
│ Client  │ Commande    │
├─────────┼─────────────┤
│ Alice   │ 101         │
│ Alice   │ 102         │
│ Bob     │ 103         │
│ Charlie │ NULL        │
│ Diana   │ NULL        │
│ NULL    │ 104         │
└─────────┴─────────────┘

CROSS JOIN (16 lignes) :
┌─────────┬─────────────┐
│ Client  │ Commande    │
├─────────┼─────────────┤
│ Alice   │ 101         │
│ Alice   │ 102         │
│ Alice   │ 103         │
│ Alice   │ 104         │
│ Bob     │ 101         │
│ Bob     │ 102         │
│ Bob     │ 103         │
│ Bob     │ 104         │
│ Charlie │ 101         │
│ Charlie │ 102         │
│ Charlie │ 103         │
│ Charlie │ 104         │
│ Diana   │ 101         │
│ Diana   │ 102         │
│ Diana   │ 103         │
│ Diana   │ 104         │
└─────────┴─────────────┘
```

---

## 14. Récapitulatif : Quand Utiliser Quelle Jointure ?

### Scénarios Courants

| Scénario | Jointure | Exemple |
|----------|----------|---------|
| Liste des clients ayant commandé | INNER JOIN | Rapport de ventes actives |
| Tous les clients avec leur total de dépense | LEFT JOIN | Tableau de bord CRM |
| Vérifier les commandes sans client valide | RIGHT JOIN | Audit d'intégrité |
| Rapport complet de réconciliation | FULL OUTER JOIN | Migration de données |
| Générer toutes les variantes produit | CROSS JOIN | Catalogue e-commerce |
| Hiérarchie employés/managers | LEFT JOIN (self) | Organigramme |

### Formules de Choix Rapide

```
Besoin de tout A ?
  Oui → LEFT JOIN ou FULL JOIN
  Non → INNER JOIN ou RIGHT JOIN

Besoin de tout B ?
  Oui → RIGHT JOIN ou FULL JOIN
  Non → INNER JOIN ou LEFT JOIN

Besoin de A ET B complets ?
  Oui → FULL OUTER JOIN
  Non → INNER, LEFT ou RIGHT JOIN

Besoin de toutes les combinaisons ?
  Oui → CROSS JOIN (attention !)
  Non → Autre type de JOIN
```

---

## 15. Exercices de Réflexion (Sans Code)

### Question 1 : Diagnostic

Vous exécutez cette requête :
```sql
SELECT COUNT(*) FROM clients INNER JOIN commandes ON clients.id = commandes.client_id;
```
Résultat : 3 lignes

Puis :
```sql
SELECT COUNT(*) FROM clients LEFT JOIN commandes ON clients.id = commandes.client_id;
```
Résultat : 5 lignes

**Que pouvez-vous en déduire ?**

**Réponse** :
- Il y a 3 correspondances (clients ayant commandé)
- Il y a 2 clients sans commande
- Un client a commandé 2 fois (3 correspondances pour probablement 2 clients)

### Question 2 : Choix de Jointure

Vous devez créer un rapport montrant :
- Tous les produits du catalogue
- Le nombre de fois qu'ils ont été vendus (0 si jamais vendu)

**Quelle jointure utilisez-vous ?**

**Réponse** : `LEFT JOIN` depuis `produits` vers `lignes_commande`, car vous voulez tous les produits même s'ils n'ont jamais été vendus.

### Question 3 : Optimisation

Une requête est très lente :
```sql
SELECT * FROM table_a CROSS JOIN table_b WHERE table_a.id = table_b.a_id;
```

**Comment l'améliorer ?**

**Réponse** : Remplacer par `INNER JOIN` :
```sql
SELECT * FROM table_a INNER JOIN table_b ON table_a.id = table_b.a_id;
```

---

## Conclusion

### Points Clés à Retenir

1. **INNER JOIN** : Intersection (seulement les correspondances)  
2. **LEFT JOIN** : Tout ce qui est à gauche + correspondances  
3. **RIGHT JOIN** : Tout ce qui est à droite + correspondances (rarement utilisé)  
4. **FULL OUTER JOIN** : Union (tout ce qui existe)  
5. **CROSS JOIN** : Produit cartésien (toutes les combinaisons)

### La Règle d'Or

**Choisissez toujours la jointure la plus restrictive qui répond à vos besoins.**

- Si vous n'avez besoin que des correspondances → **INNER JOIN**
- Si vous devez inclure les éléments sans correspondance → **LEFT/RIGHT/FULL JOIN**
- Si vous devez générer des combinaisons → **CROSS JOIN** (avec précaution)

### Bonnes Pratiques Finales

1. ✅ Toujours qualifier les colonnes ambiguës  
2. ✅ Utiliser des alias pour la lisibilité  
3. ✅ Indexer les colonnes de jointure  
4. ✅ Préférer INNER JOIN quand c'est suffisant  
5. ✅ Utiliser EXPLAIN pour comprendre les performances  
6. ✅ Tester avec de petits échantillons d'abord  
7. ⚠️ Attention aux CROSS JOIN sur grandes tables

### Prochaines Étapes

Dans les sections suivantes, nous explorerons :
- **7.5 Self-Joins** : Joindre une table à elle-même  
- **7.6 LATERAL JOIN** : Jointures corrélées avancées  
- **7.7 Anti-jointures et Semi-jointures** : NOT EXISTS, NOT IN, EXCEPT

Vous êtes maintenant équipé pour maîtriser les jointures, l'une des compétences les plus importantes en SQL !

⏭️ [Self-Joins : Joindre une table à elle-même](/07-relations-et-jointures/05-self-joins.md)
