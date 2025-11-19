🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3. Théorie des Ensembles et Jointures : Produit Cartésien et Sélection

## Introduction

Les **jointures** en SQL reposent sur des concepts mathématiques issus de la **théorie des ensembles** et de l'**algèbre relationnelle**. Comprendre ces fondements théoriques vous permettra de :
- Mieux maîtriser les jointures SQL
- Anticiper les résultats de vos requêtes
- Optimiser vos requêtes complexes
- Déboguer plus efficacement

Dans ce chapitre, nous allons explorer les bases mathématiques qui sous-tendent les opérations relationnelles, en particulier le **produit cartésien** et la **sélection**, qui forment le cœur des jointures.

### Pourquoi la théorie ?

Vous pourriez vous demander : "Pourquoi apprendre la théorie alors que je veux juste écrire du SQL ?"

**Réponse** : Parce que SQL est une implémentation de l'algèbre relationnelle ! En comprenant les concepts sous-jacents, vous ne mémoriserez pas des syntaxes, vous **comprendrez** ce qui se passe réellement. C'est la différence entre conduire une voiture et comprendre comment fonctionne un moteur.

---

## 1. Rappel : La Théorie des Ensembles

### Qu'est-ce qu'un ensemble ?

Un **ensemble** est une collection d'éléments distincts, sans ordre ni duplication.

```
Exemples d'ensembles :
A = {1, 2, 3}
B = {2, 3, 4, 5}
C = {Alice, Bob, Charlie}
```

### Opérations de Base sur les Ensembles

#### a) Union (∪)

L'**union** de deux ensembles contient tous les éléments présents dans **l'un OU l'autre** (ou les deux).

```
A = {1, 2, 3}
B = {3, 4, 5}
A ∪ B = {1, 2, 3, 4, 5}
```

**En SQL** : Opérateur `UNION`

```sql
SELECT id FROM table_a
UNION
SELECT id FROM table_b;
```

#### b) Intersection (∩)

L'**intersection** de deux ensembles contient les éléments présents dans **les deux** ensembles.

```
A = {1, 2, 3}
B = {3, 4, 5}
A ∩ B = {3}
```

**En SQL** : Opérateur `INTERSECT`

```sql
SELECT id FROM table_a
INTERSECT
SELECT id FROM table_b;
```

#### c) Différence (−)

La **différence** A − B contient les éléments de A qui **ne sont pas** dans B.

```
A = {1, 2, 3}
B = {3, 4, 5}
A − B = {1, 2}
B − A = {4, 5}
```

**En SQL** : Opérateur `EXCEPT`

```sql
SELECT id FROM table_a
EXCEPT
SELECT id FROM table_b;
```

### Lien avec les Tables Relationnelles

Dans une base de données relationnelle :
- Une **table** est un ensemble de **tuples** (lignes)
- Un **tuple** est un ensemble de **paires (attribut, valeur)**

```
Table Clients :
{
  (id=1, nom='Alice'),
  (id=2, nom='Bob'),
  (id=3, nom='Charlie')
}
```

Chaque ligne est unique (grâce à la clé primaire), donc une table respecte bien la définition d'un ensemble.

---

## 2. Le Produit Cartésien (×)

### Définition Mathématique

Le **produit cartésien** de deux ensembles A et B, noté **A × B**, est l'ensemble de toutes les **paires possibles** (a, b) où :
- a ∈ A (a appartient à A)
- b ∈ B (b appartient à B)

### Exemple Simple

```
A = {1, 2}
B = {x, y, z}

A × B = {
  (1, x), (1, y), (1, z),
  (2, x), (2, y), (2, z)
}
```

**Nombre de résultats** : |A × B| = |A| × |B| = 2 × 3 = 6

### Visualisation avec des Tables

Prenons deux petites tables :

**Table Couleurs**
| id | nom |
|----|-----|
| 1  | Rouge |
| 2  | Bleu |

**Table Formes**
| id | nom |
|----|----------|
| A  | Cercle   |
| B  | Carré    |
| C  | Triangle |

Le **produit cartésien** Couleurs × Formes donne :

| couleurs.id | couleurs.nom | formes.id | formes.nom |
|-------------|--------------|-----------|------------|
| 1           | Rouge        | A         | Cercle     |
| 1           | Rouge        | B         | Carré      |
| 1           | Rouge        | C         | Triangle   |
| 2           | Bleu         | A         | Cercle     |
| 2           | Bleu         | B         | Carré      |
| 2           | Bleu         | C         | Triangle   |

**Résultat** : 2 × 3 = **6 lignes**

Chaque couleur est combinée avec chaque forme, sans exception.

### En SQL : La Jointure Croisée (CROSS JOIN)

Le produit cartésien en SQL s'écrit avec `CROSS JOIN` (ou une simple virgule entre les tables) :

```sql
-- Syntaxe moderne (recommandée)
SELECT *
FROM couleurs
CROSS JOIN formes;

-- Syntaxe ancienne (équivalente)
SELECT *
FROM couleurs, formes;
```

**Résultat** : Toutes les combinaisons possibles !

### Exemple Concret avec Données Réelles

**Table clients**
| id | nom    |
|----|--------|
| 1  | Alice  |
| 2  | Bob    |

**Table produits**
| id | nom        | prix |
|----|------------|------|
| 10 | Ordinateur | 999  |
| 20 | Souris     | 25   |
| 30 | Clavier    | 75   |

```sql
SELECT
    clients.nom AS client,
    produits.nom AS produit,
    produits.prix
FROM clients
CROSS JOIN produits;
```

**Résultat** : 2 clients × 3 produits = **6 lignes**

| client | produit    | prix |
|--------|------------|------|
| Alice  | Ordinateur | 999  |
| Alice  | Souris     | 25   |
| Alice  | Clavier    | 75   |
| Bob    | Ordinateur | 999  |
| Bob    | Souris     | 25   |
| Bob    | Clavier    | 75   |

### Dangers du Produit Cartésien

⚠️ **Attention** : Le produit cartésien peut rapidement exploser en taille !

```
Table A : 1 000 lignes
Table B : 1 000 lignes
A × B = 1 000 000 lignes !

Table A : 10 000 lignes
Table B : 10 000 lignes
A × B = 100 000 000 lignes ! 💥
```

**Conséquence** :
- Consommation de mémoire énorme
- Temps d'exécution très long
- Saturation du serveur

### Quand Utiliser le CROSS JOIN ?

Le `CROSS JOIN` est rare mais utile dans certains cas :

#### 1. Générer des Combinaisons

```sql
-- Générer toutes les tailles × couleurs pour un catalogue
SELECT
    tailles.nom AS taille,
    couleurs.nom AS couleur
FROM tailles
CROSS JOIN couleurs
ORDER BY tailles.ordre, couleurs.nom;
```

| taille | couleur |
|--------|---------|
| XS     | Bleu    |
| XS     | Noir    |
| XS     | Rouge   |
| S      | Bleu    |
| S      | Noir    |
| S      | Rouge   |
| ...    | ...     |

#### 2. Créer des Séries de Dates

```sql
-- Générer une ligne par jour × par magasin
SELECT
    magasins.id AS magasin_id,
    dates.jour
FROM magasins
CROSS JOIN generate_series(
    '2025-01-01'::date,
    '2025-01-31'::date,
    '1 day'::interval
) AS dates(jour);
```

#### 3. Matrices de Tests

```sql
-- Générer toutes les combinaisons pour des tests
SELECT
    navigateurs.nom AS navigateur,
    os.nom AS systeme,
    resolutions.largeur || 'x' || resolutions.hauteur AS resolution
FROM navigateurs
CROSS JOIN systemes_exploitation AS os
CROSS JOIN resolutions;
```

### Le Produit Cartésien Accidentel

L'erreur la plus fréquente avec les débutants : **oublier la condition de jointure**.

```sql
-- ❌ ERREUR : Produit cartésien accidentel !
SELECT *
FROM clients, commandes;

-- ✅ CORRECT : Jointure avec condition
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

**Symptôme** : Vous obtenez beaucoup plus de lignes que prévu !

---

## 3. La Sélection (σ - Sigma)

### Définition

La **sélection** (ou **restriction**) est une opération qui **filtre les lignes** d'une table selon une condition.

En algèbre relationnelle, on note : **σ_condition(Relation)**

### Exemple

```
Table Produits :
┌────┬─────────────┬───────┐
│ id │ nom         │ prix  │
├────┼─────────────┼───────┤
│ 1  │ Ordinateur  │ 999   │
│ 2  │ Souris      │ 25    │
│ 3  │ Clavier     │ 75    │
│ 4  │ Écran       │ 350   │
└────┴─────────────┴───────┘

Sélection : σ_prix<100(Produits)

Résultat :
┌────┬─────────┬──────┐
│ id │ nom     │ prix │
├────┼─────────┼──────┤
│ 2  │ Souris  │ 25   │
│ 3  │ Clavier │ 75   │
└────┴─────────┴──────┘
```

### En SQL : La Clause WHERE

La sélection en algèbre relationnelle correspond à la clause `WHERE` en SQL.

```sql
SELECT *
FROM produits
WHERE prix < 100;
```

**Principe** : On part d'une table, et on ne garde que les lignes qui satisfont la condition.

### Conditions Multiples

```sql
-- ET logique (AND)
SELECT *
FROM produits
WHERE prix < 100 AND stock > 0;

-- OU logique (OR)
SELECT *
FROM produits
WHERE categorie = 'Électronique' OR categorie = 'Informatique';

-- Négation (NOT)
SELECT *
FROM produits
WHERE NOT (prix > 1000);
```

### Sélection sur une Jointure

La sélection peut s'appliquer **après** un produit cartésien pour filtrer les résultats.

```sql
-- Produit cartésien + sélection = JOINTURE
SELECT *
FROM clients
CROSS JOIN commandes
WHERE clients.id = commandes.client_id;
```

C'est exactement équivalent à :

```sql
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

---

## 4. De la Théorie à la Pratique : Construire une Jointure

### Le Processus en 3 Étapes

Une jointure classique peut être vue comme :

1. **Produit cartésien** : Générer toutes les combinaisons
2. **Sélection** : Filtrer selon la condition de jointure
3. **Projection** : Sélectionner les colonnes désirées (optionnel)

### Exemple Pas à Pas

**Tables de départ**

**clients**
| id | nom   |
|----|-------|
| 1  | Alice |
| 2  | Bob   |

**commandes**
| id | client_id | montant |
|----|-----------|---------|
| 10 | 1         | 150     |
| 20 | 1         | 200     |
| 30 | 2         | 100     |

#### Étape 1 : Produit Cartésien

```sql
SELECT *
FROM clients
CROSS JOIN commandes;
```

**Résultat intermédiaire** : 2 × 3 = 6 lignes

| clients.id | clients.nom | commandes.id | commandes.client_id | commandes.montant |
|------------|-------------|--------------|---------------------|-------------------|
| 1          | Alice       | 10           | 1                   | 150               |
| 1          | Alice       | 20           | 1                   | 200               |
| 1          | Alice       | 30           | 2                   | 100               |
| 2          | Bob         | 10           | 1                   | 150               |
| 2          | Bob         | 20           | 1                   | 200               |
| 2          | Bob         | 30           | 2                   | 100               |

#### Étape 2 : Sélection (Filtrage)

On applique la condition : `clients.id = commandes.client_id`

```sql
SELECT *
FROM clients
CROSS JOIN commandes
WHERE clients.id = commandes.client_id;
```

**Résultat après sélection** : 3 lignes

| clients.id | clients.nom | commandes.id | commandes.client_id | commandes.montant |
|------------|-------------|--------------|---------------------|-------------------|
| 1          | Alice       | 10           | 1                   | 150               |
| 1          | Alice       | 20           | 1                   | 200               |
| 2          | Bob         | 30           | 2                   | 100               |

Les lignes où `clients.id ≠ commandes.client_id` ont été éliminées.

#### Étape 3 : Projection (Sélection de Colonnes)

On ne garde que les colonnes utiles :

```sql
SELECT
    clients.nom,
    commandes.id AS commande_id,
    commandes.montant
FROM clients
CROSS JOIN commandes
WHERE clients.id = commandes.client_id;
```

**Résultat final** :

| nom   | commande_id | montant |
|-------|-------------|---------|
| Alice | 10          | 150     |
| Alice | 20          | 200     |
| Bob   | 30          | 100     |

### Syntaxe Équivalente avec INNER JOIN

```sql
SELECT
    clients.nom,
    commandes.id AS commande_id,
    commandes.montant
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

**Conclusion** : `INNER JOIN` = Produit cartésien + Sélection (optimisé) !

---

## 5. La Projection (π - Pi)

### Définition

La **projection** est une opération qui **sélectionne des colonnes** d'une table.

En algèbre relationnelle, on note : **π_colonnes(Relation)**

### Exemple

```
Table Produits :
┌────┬─────────────┬───────┬───────┐
│ id │ nom         │ prix  │ stock │
├────┼─────────────┼───────┼───────┤
│ 1  │ Ordinateur  │ 999   │ 5     │
│ 2  │ Souris      │ 25    │ 50    │
│ 3  │ Clavier     │ 75    │ 30    │
└────┴─────────────┴───────┴───────┘

Projection : π_nom,prix(Produits)

Résultat :
┌─────────────┬───────┐
│ nom         │ prix  │
├─────────────┼───────┤
│ Ordinateur  │ 999   │
│ Souris      │ 25    │
│ Clavier     │ 75    │
└─────────────┴───────┘
```

### En SQL : La Clause SELECT

```sql
SELECT nom, prix
FROM produits;
```

**Principe** : On ne garde que les colonnes spécifiées.

### Projection avec Duplication

En théorie des ensembles, un ensemble ne peut contenir de doublons. Mais en SQL, `SELECT` **peut** retourner des doublons (sauf si on utilise `DISTINCT`).

```sql
-- Peut retourner des doublons
SELECT categorie
FROM produits;

-- Élimine les doublons
SELECT DISTINCT categorie
FROM produits;
```

---

## 6. Combinaison : Jointure Complète

### Formule Générale

Une jointure en algèbre relationnelle peut s'écrire :

```
π_colonnes(σ_condition(A × B))
```

**En français** :
1. Produit cartésien de A et B
2. Sélection selon la condition
3. Projection des colonnes désirées

### Exemple Complet

**Objectif** : Afficher le nom des clients et le total de leurs commandes

```sql
-- Version explicite (théorique)
SELECT
    clients.nom,
    SUM(commandes.montant) AS total
FROM clients
CROSS JOIN commandes
WHERE clients.id = commandes.client_id
GROUP BY clients.nom;

-- Version standard (pratique)
SELECT
    clients.nom,
    SUM(commandes.montant) AS total
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id
GROUP BY clients.nom;
```

**Résultat** :

| nom   | total |
|-------|-------|
| Alice | 350   |
| Bob   | 100   |

---

## 7. Optimisation : Pourquoi SQL n'exécute pas Littéralement le Produit Cartésien

### Le Problème de Performance

Si SQL exécutait réellement un produit cartésien complet avant de filtrer, ce serait catastrophique :

```
Table A : 1 000 000 lignes
Table B : 1 000 000 lignes
Produit cartésien : 1 000 000 000 000 lignes (1 trillion !)
```

### Le Rôle de l'Optimiseur de Requêtes

Le **planificateur de requêtes** (Query Planner) de PostgreSQL est intelligent. Il analyse votre requête et choisit la **meilleure stratégie** :

#### Stratégie 1 : Nested Loop (Boucle Imbriquée)

Pour chaque ligne de la table A, chercher les lignes correspondantes dans B.

```
Pour chaque client :
    Chercher ses commandes dans la table commandes
```

**Efficace** : Si une table est petite et l'autre a un index sur la clé de jointure.

#### Stratégie 2 : Hash Join

Créer une table de hachage en mémoire pour une table, puis parcourir l'autre.

```
1. Créer un hash des clients en mémoire (id → client)
2. Parcourir les commandes
3. Pour chaque commande, chercher le client dans le hash
```

**Efficace** : Pour les grandes tables sans index adapté.

#### Stratégie 3 : Merge Join

Trier les deux tables, puis les parcourir en parallèle.

```
1. Trier clients par id
2. Trier commandes par client_id
3. Parcourir les deux en synchronisation
```

**Efficace** : Si les tables sont déjà triées ou si on peut les trier efficacement.

### Voir le Plan d'Exécution

```sql
EXPLAIN ANALYZE
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

**Exemple de résultat** :

```
Hash Join  (cost=15.50..35.88 rows=320 width=68)
  Hash Cond: (commandes.client_id = clients.id)
  ->  Seq Scan on commandes  (cost=0.00..18.20 rows=820 width=40)
  ->  Hash  (cost=12.00..12.00 rows=200 width=28)
        ->  Seq Scan on clients  (cost=0.00..12.00 rows=200 width=28)
```

**Interprétation** : PostgreSQL utilise un **Hash Join**, pas un produit cartésien complet !

---

## 8. Différences entre Théorie et Pratique

### En Théorie (Algèbre Relationnelle)

1. **Pas d'ordre** : Les lignes n'ont pas d'ordre
2. **Pas de doublons** : Les ensembles n'ont pas de doublons
3. **Pas de NULL** : NULL n'existe pas en théorie pure

### En Pratique (SQL)

1. **Ordre possible** : `ORDER BY` définit un ordre
2. **Doublons possibles** : Sauf si on utilise `DISTINCT`
3. **NULL existe** : Avec une logique ternaire complexe

### Multiensembles (Bags)

SQL travaille en réalité avec des **multiensembles** (bags), qui autorisent les doublons.

```sql
-- Peut retourner : {1, 1, 2, 3, 3, 3}
SELECT categorie_id FROM produits;

-- Retourne un ensemble : {1, 2, 3}
SELECT DISTINCT categorie_id FROM produits;
```

---

## 9. Types de Jointures : Au-delà du Produit Cartésien

Le produit cartésien et la sélection forment la base, mais SQL offre des jointures plus sophistiquées :

### Vue d'Ensemble

| Type de Jointure | Opération Algébrique | Description |
|------------------|---------------------|-------------|
| **CROSS JOIN** | A × B | Produit cartésien complet |
| **INNER JOIN** | σ(A × B) | Produit cartésien + sélection |
| **LEFT JOIN** | A ⟕ B | Toutes les lignes de A + correspondances de B |
| **RIGHT JOIN** | A ⟖ B | Toutes les lignes de B + correspondances de A |
| **FULL JOIN** | A ⟗ B | Toutes les lignes de A et B |

### INNER JOIN : Intersection

```sql
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

**Résultat** : Seulement les clients qui **ont** des commandes.

### LEFT JOIN : Inclusion de la Table de Gauche

```sql
SELECT *
FROM clients
LEFT JOIN commandes ON clients.id = commandes.client_id;
```

**Résultat** : **Tous** les clients, même ceux sans commande (NULL pour les colonnes de commandes).

### FULL JOIN : Union

```sql
SELECT *
FROM clients
FULL OUTER JOIN commandes ON clients.id = commandes.client_id;
```

**Résultat** : Tous les clients ET toutes les commandes (avec NULL si pas de correspondance).

Nous explorerons ces types de jointures en détail dans la section suivante (7.4).

---

## 10. Exemples Pratiques de Produit Cartésien + Sélection

### Exemple 1 : Catalogue Produits

Créer un catalogue avec toutes les combinaisons taille × couleur.

```sql
CREATE TABLE tailles (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(10) NOT NULL
);

CREATE TABLE couleurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(20) NOT NULL
);

INSERT INTO tailles (nom) VALUES ('XS'), ('S'), ('M'), ('L'), ('XL');
INSERT INTO couleurs (nom) VALUES ('Rouge'), ('Bleu'), ('Noir'), ('Blanc');

-- Générer toutes les combinaisons
SELECT
    tailles.nom AS taille,
    couleurs.nom AS couleur
FROM tailles
CROSS JOIN couleurs
ORDER BY tailles.id, couleurs.nom;
```

**Résultat** : 5 tailles × 4 couleurs = **20 combinaisons**

| taille | couleur |
|--------|---------|
| XS     | Bleu    |
| XS     | Blanc   |
| XS     | Noir    |
| XS     | Rouge   |
| S      | Bleu    |
| ...    | ...     |

### Exemple 2 : Planning de Réunions

Trouver les créneaux où deux personnes sont disponibles.

```sql
CREATE TABLE disponibilites_alice (
    jour DATE,
    heure_debut TIME,
    heure_fin TIME
);

CREATE TABLE disponibilites_bob (
    jour DATE,
    heure_debut TIME,
    heure_fin TIME
);

-- Trouver les créneaux communs
SELECT
    alice.jour,
    GREATEST(alice.heure_debut, bob.heure_debut) AS debut,
    LEAST(alice.heure_fin, bob.heure_fin) AS fin
FROM disponibilites_alice alice
CROSS JOIN disponibilites_bob bob
WHERE alice.jour = bob.jour
  AND alice.heure_debut < bob.heure_fin
  AND bob.heure_debut < alice.heure_fin;
```

### Exemple 3 : Distances entre Villes

Calculer les distances entre toutes les paires de villes.

```sql
CREATE TABLE villes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8)
);

-- Produit cartésien pour calculer toutes les distances
SELECT
    v1.nom AS ville_depart,
    v2.nom AS ville_arrivee,
    -- Formule de distance (simplifiée)
    SQRT(
        POW(v1.latitude - v2.latitude, 2) +
        POW(v1.longitude - v2.longitude, 2)
    ) AS distance
FROM villes v1
CROSS JOIN villes v2
WHERE v1.id < v2.id  -- Éviter les doublons (Paris->Lyon = Lyon->Paris)
ORDER BY distance;
```

---

## 11. Pièges et Erreurs Courantes

### Erreur 1 : Oublier la Condition de Jointure

```sql
-- ❌ Produit cartésien accidentel
SELECT *
FROM clients, commandes;

-- Résultat : 1000 clients × 5000 commandes = 5 000 000 lignes !
```

**Symptôme** : Requête très lente, résultat gigantesque.

**Solution** : Toujours spécifier la condition de jointure.

```sql
-- ✅ Correct
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

### Erreur 2 : Jointure sur Plusieurs Tables Sans Ordre

```sql
-- ⚠️ Dangereux si mal écrit
SELECT *
FROM clients, commandes, produits;
```

Si aucune condition n'est spécifiée, c'est un produit cartésien triple !

```
10 clients × 100 commandes × 1000 produits = 1 000 000 lignes !
```

**Solution** : Utiliser `JOIN` explicite avec conditions.

```sql
-- ✅ Correct
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id
INNER JOIN lignes_commande ON commandes.id = lignes_commande.commande_id
INNER JOIN produits ON lignes_commande.produit_id = produits.id;
```

### Erreur 3 : Confusion entre INNER et CROSS JOIN

```sql
-- Ces deux requêtes sont DIFFÉRENTES

-- 1. CROSS JOIN : Toutes les combinaisons
SELECT * FROM clients CROSS JOIN commandes;

-- 2. INNER JOIN : Seulement les correspondances
SELECT * FROM clients INNER JOIN commandes ON clients.id = commandes.client_id;
```

---

## 12. Visualisation des Concepts

### Diagramme : Du Produit Cartésien à la Jointure

```
┌─────────────────────────────────────────────────────┐
│  Étape 1 : PRODUIT CARTÉSIEN (A × B)                │
│  ┌───────┐     ┌───────┐                            │
│  │   A   │  ×  │   B   │  → Toutes les combinaisons │
│  └───────┘     └───────┘                            │
│        ↓                                            │
│  Résultat : |A| × |B| lignes                        │
└─────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│  Étape 2 : SÉLECTION (σ)                            │
│  Filtrer selon condition : A.id = B.id_ref          │
│        ↓                                            │
│  Résultat : Seulement les lignes qui correspondent  │
└─────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│  Étape 3 : PROJECTION (π)                           │
│  Sélectionner les colonnes désirées                 │
│        ↓                                            │
│  Résultat final : Jointure complète                 │
└─────────────────────────────────────────────────────┘
```

### Représentation Graphique : Tables et Jointures

```
Table A (Clients)        Table B (Commandes)
┌────┬───────┐           ┌────┬────────────┬─────────┐
│ id │ nom   │           │ id │ client_id  │ montant │
├────┼───────┤           ├────┼────────────┼─────────┤
│ 1  │ Alice │    ┌─────→│ 10 │     1      │   150   │
│ 2  │ Bob   │────┤      │ 20 │     1      │   200   │
│ 3  │ Clara │    └─────→│ 30 │     2      │   100   │
└────┴───────┘           └────┴────────────┴─────────┘

Produit cartésien : 3 × 3 = 9 lignes
Après sélection (id = client_id) : 3 lignes
```

---

## 13. Résumé des Opérations d'Algèbre Relationnelle

| Opération | Symbole | SQL | Description |
|-----------|---------|-----|-------------|
| **Sélection** | σ | WHERE | Filtre les lignes |
| **Projection** | π | SELECT colonnes | Sélectionne les colonnes |
| **Union** | ∪ | UNION | Combine deux tables |
| **Intersection** | ∩ | INTERSECT | Lignes communes |
| **Différence** | − | EXCEPT | Lignes dans A mais pas B |
| **Produit cartésien** | × | CROSS JOIN | Toutes les combinaisons |
| **Jointure naturelle** | ⋈ | INNER JOIN | Produit cartésien + sélection |
| **Renommage** | ρ | AS | Renomme colonnes/tables |

---

## 14. Bonnes Pratiques

### 1. Toujours Utiliser JOIN Explicite

```sql
-- ❌ À éviter (ancienne syntaxe)
SELECT *
FROM clients, commandes
WHERE clients.id = commandes.client_id;

-- ✅ Recommandé (syntaxe moderne)
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

**Pourquoi ?**
- Plus lisible
- Moins de risque d'erreur
- Séparation claire entre jointure et filtrage

### 2. Nommer les Alias de Tables

```sql
-- ✅ Bon
SELECT
    c.nom AS client,
    cmd.montant
FROM clients AS c
INNER JOIN commandes AS cmd ON c.id = cmd.client_id;
```

**Avantages** :
- Code plus court
- Plus facile à lire
- Évite les ambiguïtés

### 3. Vérifier le Nombre de Lignes Attendu

Avant d'exécuter une jointure sur de grandes tables :

```sql
-- Vérifier les tailles
SELECT COUNT(*) FROM table_a;
SELECT COUNT(*) FROM table_b;

-- Estimer le résultat
-- INNER JOIN : ≤ MIN(count_a, count_b)
-- LEFT JOIN : = count_a
-- CROSS JOIN : = count_a × count_b (danger !)
```

### 4. Utiliser EXPLAIN pour Comprendre

```sql
EXPLAIN ANALYZE
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

Cela vous montrera la stratégie choisie par PostgreSQL.

---

## 15. Exercices de Compréhension (Théoriques)

### Question 1 : Calcul de Taille

Soit :
- Table A : 50 lignes
- Table B : 30 lignes

Combien de lignes retournent les opérations suivantes ?

a) `A CROSS JOIN B`
b) `A INNER JOIN B ON A.id = B.a_id` (supposons 20 correspondances)
c) `A LEFT JOIN B ON A.id = B.a_id`

**Réponses** :
a) 50 × 30 = **1500 lignes**
b) **20 lignes** (seulement les correspondances)
c) **50 lignes** (toutes les lignes de A)

### Question 2 : Opérations Algébriques

Traduire en SQL :
```
π_nom,prix(σ_prix>100(Produits))
```

**Réponse** :
```sql
SELECT nom, prix
FROM produits
WHERE prix > 100;
```

### Question 3 : Identifier le Type de Jointure

Quelle opération algébrique correspond à :
```sql
SELECT *
FROM employes
CROSS JOIN departements
WHERE employes.dept_id = departements.id;
```

**Réponse** :
- Produit cartésien (`CROSS JOIN`)
- Suivi d'une sélection (`WHERE`)
- = `INNER JOIN`

---

## Conclusion

### Points Clés à Retenir

1. **Produit cartésien** : Toutes les combinaisons possibles (A × B)
2. **Sélection** : Filtre les lignes selon une condition (WHERE)
3. **Projection** : Sélectionne les colonnes (SELECT)
4. **INNER JOIN** = Produit cartésien + Sélection (optimisé)
5. **L'optimiseur** ne fait pas littéralement un produit cartésien complet

### La Magie de l'Algèbre Relationnelle

Comprendre que :
```
INNER JOIN = Produit cartésien + Sélection
```

Vous permet de :
- Anticiper les résultats de vos requêtes
- Déboguer les jointures complexes
- Comprendre pourquoi certaines requêtes sont lentes
- Optimiser vos schémas de base de données

### Prochaines Étapes

Dans la section suivante **(7.4 Types de jointures)**, nous explorerons en détail :
- **INNER JOIN** : Intersection
- **LEFT/RIGHT JOIN** : Jointures externes
- **FULL OUTER JOIN** : Union complète
- **CROSS JOIN** : Cas d'usage pratiques
- **SELF-JOIN** : Joindre une table à elle-même

Vous verrez comment ces différentes jointures correspondent à des opérations ensemblistes et comment les utiliser efficacement dans vos applications.

---

**Résumé en une phrase** : Toute jointure SQL est, conceptuellement, un produit cartésien suivi d'une sélection, mais PostgreSQL est assez intelligent pour ne jamais l'exécuter littéralement de cette façon !

⏭️ [Types de jointures : INNER, LEFT/RIGHT/FULL OUTER, CROSS](/07-relations-et-jointures/04-types-de-jointures.md)
