🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.6. Jointures Latérales (LATERAL JOIN) : Corrélation Avancée

## Introduction

Le **LATERAL JOIN** est une fonctionnalité puissante de PostgreSQL qui permet à une sous-requête ou une fonction de **faire référence aux colonnes de tables précédentes** dans la clause FROM. C'est comme si vous pouviez "regarder en arrière" vers les tables déjà jointes.

### Le Problème que LATERAL Résout

Avec les jointures classiques, chaque table est évaluée **indépendamment**. Vous ne pouvez pas écrire :

```sql
-- ❌ ERREUR : sous-requête ne peut pas référencer clients
SELECT *
FROM clients
CROSS JOIN (
    SELECT * FROM commandes
    WHERE commandes.client_id = clients.id  -- ❌ clients pas accessible
    LIMIT 3
);
```

Avec **LATERAL**, c'est possible :

```sql
-- ✅ FONCTIONNE : LATERAL permet la corrélation
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes
    WHERE commandes.client_id = clients.id  -- ✅ clients accessible !
    LIMIT 3
) AS dernieres_commandes;
```

### Analogie Simple

Imaginez que vous lisez un livre :
- **Jointure classique** : Chaque chapitre est indépendant, vous ne pouvez pas référencer le chapitre précédent
- **LATERAL JOIN** : Chaque chapitre peut faire référence aux chapitres précédents

---

## 1. Concept Fondamental : La Corrélation

### Qu'est-ce que la Corrélation ?

Une requête est **corrélée** quand elle fait référence à des colonnes définies **en dehors** de son contexte immédiat.

#### Exemple Simple : Sous-requête Corrélée

```sql
-- Trouver les clients ayant commandé plus que la moyenne
SELECT nom
FROM clients
WHERE (
    SELECT AVG(montant)
    FROM commandes
    WHERE commandes.client_id = clients.id  -- Corrélation ici
) > 100;
```

**Problème** : Les sous-requêtes corrélées en WHERE sont **lentes** car elles sont exécutées **pour chaque ligne**.

### LATERAL : Corrélation dans FROM

LATERAL permet de placer cette corrélation dans la clause `FROM`, ce qui :
- Rend le code plus lisible
- Permet des opérations impossibles autrement (LIMIT, OFFSET, ORDER BY corrélés)
- Peut améliorer les performances (selon les cas)

---

## 2. Syntaxe de Base

### Forme Générale

```sql
SELECT ...
FROM table_gauche
CROSS JOIN LATERAL (
    sous_requete_qui_reference_table_gauche
) AS alias;

-- Ou avec LEFT JOIN LATERAL
SELECT ...
FROM table_gauche
LEFT JOIN LATERAL (
    sous_requete_qui_reference_table_gauche
) AS alias ON true;
```

**Important** : Le mot-clé `LATERAL` active la corrélation.

### Exemple Minimal

**Tables d'exemple** :

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    ville VARCHAR(100)
);

CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER REFERENCES clients(id),
    montant NUMERIC(10, 2) NOT NULL,
    date_commande DATE NOT NULL
);

INSERT INTO clients (nom, ville) VALUES
    ('Alice', 'Paris'),
    ('Bob', 'Lyon'),
    ('Charlie', 'Marseille');

INSERT INTO commandes (client_id, montant, date_commande) VALUES
    (1, 100, '2025-01-01'),
    (1, 150, '2025-01-05'),
    (1, 200, '2025-01-10'),
    (1, 120, '2025-01-15'),
    (2, 80, '2025-01-03'),
    (2, 90, '2025-01-08');
```

**Objectif** : Pour chaque client, obtenir ses 2 dernières commandes.

```sql
SELECT
    clients.nom,
    dernieres.id AS commande_id,
    dernieres.montant,
    dernieres.date_commande
FROM clients
CROSS JOIN LATERAL (
    SELECT id, montant, date_commande
    FROM commandes
    WHERE commandes.client_id = clients.id  -- Corrélation
    ORDER BY date_commande DESC
    LIMIT 2
) AS dernieres;
```

**Résultat** :

| nom   | commande_id | montant | date_commande |
|-------|-------------|---------|---------------|
| Alice | 4           | 120     | 2025-01-15    |
| Alice | 3           | 200     | 2025-01-10    |
| Bob   | 6           | 90      | 2025-01-08    |
| Bob   | 5           | 80      | 2025-01-03    |

**Observations** :
- Alice a 4 commandes, on obtient les 2 dernières
- Bob a 2 commandes, on obtient les 2
- Charlie n'a pas de commande, il n'apparaît pas (CROSS JOIN)

---

## 3. CROSS JOIN LATERAL vs LEFT JOIN LATERAL

### CROSS JOIN LATERAL

Équivalent à un **INNER JOIN** : Retourne seulement les lignes où la sous-requête produit des résultats.

```sql
SELECT clients.nom, dernieres.*
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes
    WHERE commandes.client_id = clients.id
    LIMIT 2
) AS dernieres;
```

**Résultat** : Charlie n'apparaît pas (il n'a pas de commande).

### LEFT JOIN LATERAL

Retourne **tous les clients**, même ceux sans résultat dans la sous-requête (avec NULL).

```sql
SELECT
    clients.nom,
    dernieres.id AS commande_id,
    dernieres.montant
FROM clients
LEFT JOIN LATERAL (
    SELECT id, montant
    FROM commandes
    WHERE commandes.client_id = clients.id
    ORDER BY date_commande DESC
    LIMIT 2
) AS dernieres ON true;
```

**Résultat** :

| nom     | commande_id | montant |
|---------|-------------|---------|
| Alice   | 4           | 120     |
| Alice   | 3           | 200     |
| Bob     | 6           | 90      |
| Bob     | 5           | 80      |
| Charlie | NULL        | NULL    |

**Note** : `ON true` est une condition toujours vraie, nécessaire syntaxiquement pour LEFT JOIN.

---

## 4. Cas d'Usage Principaux

### Cas 1 : Top N par Groupe

**Problème classique** : Obtenir les N premiers éléments pour chaque groupe.

#### Exemple : Les 3 Produits les Plus Vendus par Catégorie

**Tables** :

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    categorie_id INTEGER REFERENCES categories(id),
    ventes INTEGER NOT NULL
);

INSERT INTO categories (nom) VALUES ('Électronique'), ('Livres'), ('Vêtements');

INSERT INTO produits (nom, categorie_id, ventes) VALUES
    ('Smartphone', 1, 1500),
    ('Ordinateur', 1, 1200),
    ('Tablette', 1, 800),
    ('Écouteurs', 1, 600),
    ('Roman', 2, 900),
    ('BD', 2, 700),
    ('Essai', 2, 500),
    ('T-shirt', 3, 1100),
    ('Jean', 3, 950);
```

**Requête avec LATERAL** :

```sql
SELECT
    categories.nom AS categorie,
    top_produits.nom AS produit,
    top_produits.ventes
FROM categories
CROSS JOIN LATERAL (
    SELECT nom, ventes
    FROM produits
    WHERE produits.categorie_id = categories.id
    ORDER BY ventes DESC
    LIMIT 3
) AS top_produits
ORDER BY categories.nom, top_produits.ventes DESC;
```

**Résultat** :

| categorie     | produit     | ventes |
|---------------|-------------|--------|
| Électronique  | Smartphone  | 1500   |
| Électronique  | Ordinateur  | 1200   |
| Électronique  | Tablette    | 800    |
| Livres        | Roman       | 900    |
| Livres        | BD          | 700    |
| Livres        | Essai       | 500    |
| Vêtements     | T-shirt     | 1100   |
| Vêtements     | Jean        | 950    |

**Alternative sans LATERAL** (avec Window Functions) :

```sql
SELECT categorie, produit, ventes
FROM (
    SELECT
        c.nom AS categorie,
        p.nom AS produit,
        p.ventes,
        ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY p.ventes DESC) AS rang
    FROM categories c
    INNER JOIN produits p ON c.id = p.categorie_id
) AS ranked
WHERE rang <= 3
ORDER BY categorie, ventes DESC;
```

**Comparaison** :
- **LATERAL** : Plus intuitif, lecture naturelle (pour chaque catégorie, prendre les 3 meilleurs)
- **Window Functions** : Plus performant sur grandes tables, syntaxe plus compacte

### Cas 2 : Calculs Complexes Corrélés

**Exemple** : Pour chaque client, calculer la différence entre sa dernière commande et sa moyenne.

```sql
SELECT
    clients.nom,
    stats.derniere_commande,
    stats.moyenne_commandes,
    stats.derniere_commande - stats.moyenne_commandes AS difference
FROM clients
CROSS JOIN LATERAL (
    SELECT
        (SELECT montant FROM commandes
         WHERE commandes.client_id = clients.id
         ORDER BY date_commande DESC
         LIMIT 1) AS derniere_commande,
        (SELECT AVG(montant) FROM commandes
         WHERE commandes.client_id = clients.id) AS moyenne_commandes
) AS stats
WHERE stats.derniere_commande IS NOT NULL;
```

**Résultat** :

| nom   | derniere_commande | moyenne_commandes | difference |
|-------|-------------------|-------------------|------------|
| Alice | 120               | 142.50            | -22.50     |
| Bob   | 90                | 85.00             | 5.00       |

### Cas 3 : Jointure avec Fonction Renvoyant un Ensemble

LATERAL est particulièrement utile avec des fonctions qui retournent des ensembles (set-returning functions).

#### Exemple : Fonction generate_series

```sql
-- Pour chaque client, générer les 5 prochains mois
SELECT
    clients.nom,
    TO_CHAR(mois.date, 'YYYY-MM') AS mois
FROM clients
CROSS JOIN LATERAL generate_series(
    CURRENT_DATE,
    CURRENT_DATE + INTERVAL '4 months',
    INTERVAL '1 month'
) AS mois(date)
WHERE clients.id <= 2  -- Limiter pour l'exemple
ORDER BY clients.nom, mois.date;
```

**Résultat** :

| nom   | mois    |
|-------|---------|
| Alice | 2025-01 |
| Alice | 2025-02 |
| Alice | 2025-03 |
| Alice | 2025-04 |
| Alice | 2025-05 |
| Bob   | 2025-01 |
| Bob   | 2025-02 |
| Bob   | 2025-03 |
| Bob   | 2025-04 |
| Bob   | 2025-05 |

#### Exemple : Fonction Personnalisée

```sql
-- Créer une fonction qui retourne les commandes d'un client
CREATE OR REPLACE FUNCTION get_client_orders(p_client_id INTEGER, p_limit INTEGER)
RETURNS TABLE (
    commande_id INTEGER,
    montant NUMERIC,
    date_commande DATE
) AS $$
BEGIN
    RETURN QUERY
    SELECT id, montant, date_commande
    FROM commandes
    WHERE client_id = p_client_id
    ORDER BY date_commande DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql;

-- Utiliser avec LATERAL
SELECT
    clients.nom,
    orders.*
FROM clients
CROSS JOIN LATERAL get_client_orders(clients.id, 2) AS orders;
```

### Cas 4 : Enrichissement de Données

**Exemple** : Pour chaque commande, ajouter des statistiques sur le client.

```sql
SELECT
    commandes.id,
    commandes.montant,
    stats.nombre_total_commandes,
    stats.montant_total_depense,
    stats.commande_moyenne
FROM commandes
CROSS JOIN LATERAL (
    SELECT
        COUNT(*) AS nombre_total_commandes,
        SUM(montant) AS montant_total_depense,
        AVG(montant) AS commande_moyenne
    FROM commandes c2
    WHERE c2.client_id = commandes.client_id
) AS stats;
```

**Résultat** : Chaque commande est enrichie avec les statistiques globales du client.

---

## 5. LATERAL avec Plusieurs Tables

Vous pouvez enchaîner plusieurs LATERAL JOIN.

### Exemple : Client → Dernières Commandes → Produits de ces Commandes

```sql
CREATE TABLE lignes_commande (
    id SERIAL PRIMARY KEY,
    commande_id INTEGER REFERENCES commandes(id),
    produit_nom VARCHAR(100),
    quantite INTEGER
);

-- Pour chaque client, ses 2 dernières commandes, et les produits
SELECT
    clients.nom AS client,
    dernieres_cmd.commande_id,
    dernieres_cmd.date_commande,
    lignes.produit_nom,
    lignes.quantite
FROM clients
CROSS JOIN LATERAL (
    SELECT id AS commande_id, date_commande
    FROM commandes
    WHERE commandes.client_id = clients.id
    ORDER BY date_commande DESC
    LIMIT 2
) AS dernieres_cmd
CROSS JOIN LATERAL (
    SELECT produit_nom, quantite
    FROM lignes_commande
    WHERE lignes_commande.commande_id = dernieres_cmd.commande_id
) AS lignes
ORDER BY clients.nom, dernieres_cmd.date_commande DESC;
```

**Explication** :
1. Pour chaque client
2. Obtenir ses 2 dernières commandes (LATERAL 1)
3. Pour chaque commande, obtenir ses lignes (LATERAL 2)

---

## 6. Différences avec les Sous-Requêtes Classiques

### Sous-requête Scalaire (dans SELECT)

```sql
-- ❌ Limité : Une seule valeur
SELECT
    clients.nom,
    (SELECT MAX(montant) FROM commandes WHERE commandes.client_id = clients.id) AS max_commande
FROM clients;
```

**Limitation** : Ne peut retourner qu'une seule valeur (scalaire).

### Sous-requête Corrélée (dans WHERE)

```sql
-- ⚠️ Lent : Exécuté pour chaque ligne
SELECT *
FROM clients
WHERE EXISTS (
    SELECT 1 FROM commandes
    WHERE commandes.client_id = clients.id
    AND montant > 100
);
```

**Limitation** : Performance médiocre sur grandes tables.

### LATERAL JOIN

```sql
-- ✅ Flexible : Peut retourner plusieurs lignes et colonnes
SELECT
    clients.nom,
    details.commande_id,
    details.montant,
    details.date_commande
FROM clients
CROSS JOIN LATERAL (
    SELECT id AS commande_id, montant, date_commande
    FROM commandes
    WHERE commandes.client_id = clients.id
    ORDER BY montant DESC
    LIMIT 3
) AS details;
```

**Avantages** :
- ✅ Retourne plusieurs lignes
- ✅ Retourne plusieurs colonnes
- ✅ Permet ORDER BY, LIMIT, OFFSET
- ✅ Plus lisible

---

## 7. LATERAL vs Window Functions

### Quand Utiliser LATERAL ?

**LATERAL est préférable quand** :
- Vous avez besoin d'un **LIMIT par groupe**
- Vous devez appeler une **fonction** avec des paramètres corrélés
- Vous voulez des **calculs complexes** par groupe
- La logique est plus **claire** avec LATERAL

### Quand Utiliser Window Functions ?

**Window Functions sont préférables quand** :
- Vous devez calculer des **rangs, moyennes mobiles, cumuls**
- Vous travaillez sur **toutes les lignes** du groupe
- Les **performances** sont critiques sur grandes tables

### Exemple Comparatif : Top 3 par Catégorie

#### Avec LATERAL

```sql
SELECT cat.nom, top.nom, top.ventes
FROM categories cat
CROSS JOIN LATERAL (
    SELECT nom, ventes
    FROM produits
    WHERE produits.categorie_id = cat.id
    ORDER BY ventes DESC
    LIMIT 3
) AS top;
```

#### Avec Window Functions

```sql
SELECT categorie, nom, ventes
FROM (
    SELECT
        c.nom AS categorie,
        p.nom,
        p.ventes,
        ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY p.ventes DESC) AS rang
    FROM categories c
    JOIN produits p ON c.id = p.categorie_id
) sub
WHERE rang <= 3;
```

**Benchmark** (sur 1 million de lignes) :
- **Window Functions** : ~200ms
- **LATERAL** : ~250ms

**Conclusion** : Window Functions généralement plus rapides, mais LATERAL plus flexible et lisible.

---

## 8. Cas d'Usage Avancés

### Cas 1 : Génération de Séries Personnalisées

**Objectif** : Pour chaque produit, générer des prévisions de ventes mensuelles.

```sql
SELECT
    produits.nom,
    forecast.mois,
    forecast.ventes_prevues
FROM produits
CROSS JOIN LATERAL (
    SELECT
        date,
        produits.ventes * (1 + (RANDOM() * 0.1 - 0.05)) AS ventes_prevues
    FROM generate_series(
        CURRENT_DATE,
        CURRENT_DATE + INTERVAL '5 months',
        INTERVAL '1 month'
    ) AS date
) AS forecast(mois, ventes_prevues)
LIMIT 10;
```

### Cas 2 : Recherche Hiérarchique

**Objectif** : Pour chaque employé, remonter la hiérarchie jusqu'au PDG.

```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    manager_id INTEGER REFERENCES employes(id)
);

-- Pour chaque employé, obtenir sa chaîne hiérarchique
SELECT
    e.nom AS employe,
    hierarchy.niveau,
    hierarchy.manager_nom
FROM employes e
CROSS JOIN LATERAL (
    WITH RECURSIVE chain AS (
        SELECT id, nom, manager_id, 0 AS niveau
        FROM employes
        WHERE id = e.id

        UNION ALL

        SELECT emp.id, emp.nom, emp.manager_id, chain.niveau + 1
        FROM employes emp
        INNER JOIN chain ON emp.id = chain.manager_id
    )
    SELECT niveau, nom AS manager_nom
    FROM chain
    WHERE niveau > 0
) AS hierarchy;
```

### Cas 3 : Calculs de Distance Géographique

**Objectif** : Pour chaque client, trouver les 3 magasins les plus proches.

```sql
CREATE TABLE magasins (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    latitude NUMERIC(10, 8),
    longitude NUMERIC(11, 8)
);

-- Trouver les 3 magasins les plus proches pour chaque client
SELECT
    clients.nom AS client,
    proches.nom AS magasin,
    proches.distance_km
FROM clients
CROSS JOIN LATERAL (
    SELECT
        magasins.nom,
        -- Formule de distance approximative
        111 * SQRT(
            POW(magasins.latitude - clients.latitude, 2) +
            POW(magasins.longitude - clients.longitude, 2)
        ) AS distance_km
    FROM magasins
    ORDER BY distance_km
    LIMIT 3
) AS proches;
```

### Cas 4 : Analyse de Séquences Temporelles

**Objectif** : Pour chaque transaction, calculer le temps écoulé depuis les 3 transactions précédentes.

```sql
CREATE TABLE transactions (
    id SERIAL PRIMARY KEY,
    utilisateur_id INTEGER,
    montant NUMERIC(10, 2),
    date_transaction TIMESTAMP
);

SELECT
    t.id,
    t.montant,
    t.date_transaction,
    precedentes.id AS transaction_precedente_id,
    precedentes.montant AS montant_precedent,
    EXTRACT(EPOCH FROM (t.date_transaction - precedentes.date_transaction)) / 60 AS minutes_ecoulees
FROM transactions t
CROSS JOIN LATERAL (
    SELECT id, montant, date_transaction
    FROM transactions t2
    WHERE t2.utilisateur_id = t.utilisateur_id
      AND t2.date_transaction < t.date_transaction
    ORDER BY t2.date_transaction DESC
    LIMIT 3
) AS precedentes
ORDER BY t.utilisateur_id, t.date_transaction;
```

---

## 9. Performances et Optimisation

### Bonnes Pratiques

#### 1. Indexer les Colonnes de Corrélation

```sql
-- Si vous corrèlez sur client_id
CREATE INDEX idx_commandes_client_id ON commandes(client_id);

-- Index composite si vous triez aussi
CREATE INDEX idx_commandes_client_date ON commandes(client_id, date_commande DESC);
```

#### 2. Limiter le Nombre de Lignes

LATERAL peut générer beaucoup de lignes. Utilisez LIMIT judicieusement.

```sql
-- ⚠️ Peut générer énormément de lignes
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes WHERE commandes.client_id = clients.id
) AS all_orders;

-- ✅ Limitez pour améliorer les performances
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes
    WHERE commandes.client_id = clients.id
    ORDER BY date_commande DESC
    LIMIT 10
) AS recent_orders;
```

#### 3. Utiliser EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes
    WHERE commandes.client_id = clients.id
    ORDER BY date_commande DESC
    LIMIT 5
) AS recent;
```

Cherchez :
- **Nested Loop** : Normal pour LATERAL
- **Index Scan** : Bon signe (l'index est utilisé)
- **Seq Scan** : Peut indiquer un manque d'index

#### 4. Filtrer AVANT le LATERAL

```sql
-- ✅ Meilleur : Filtrer d'abord les clients
SELECT *
FROM (SELECT * FROM clients WHERE ville = 'Paris') AS clients_paris
CROSS JOIN LATERAL (
    SELECT * FROM commandes
    WHERE commandes.client_id = clients_paris.id
    LIMIT 5
) AS recent;

-- vs

-- ⚠️ Moins efficace : LATERAL pour tous les clients
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes
    WHERE commandes.client_id = clients.id
    LIMIT 5
) AS recent
WHERE clients.ville = 'Paris';
```

### Quand LATERAL Peut Être Lent

1. **Tables volumineuses sans index** sur les colonnes de corrélation
2. **LATERAL sans LIMIT** générant des millions de lignes
3. **Fonctions coûteuses** appelées pour chaque ligne
4. **Sous-requêtes complexes** dans LATERAL

### Alternatives Plus Rapides

Si les performances sont critiques :

```sql
-- Alternative 1 : Window Functions (plus rapide pour Top N)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY client_id ORDER BY date_commande DESC) AS rn
    FROM commandes
) sub WHERE rn <= 5;

-- Alternative 2 : Agrégation avec ARRAY_AGG
SELECT
    clients.nom,
    ARRAY_AGG(commandes.id ORDER BY date_commande DESC) FILTER (WHERE row_num <= 5) AS top_5_commandes
FROM clients
LEFT JOIN (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY client_id ORDER BY date_commande DESC) AS row_num
    FROM commandes
) commandes ON clients.id = commandes.client_id
WHERE commandes.row_num <= 5 OR commandes.row_num IS NULL
GROUP BY clients.id, clients.nom;
```

---

## 10. Pièges et Erreurs Courantes

### Erreur 1 : Oublier le Mot-Clé LATERAL

```sql
-- ❌ ERREUR : Référence invalide à clients
SELECT *
FROM clients
CROSS JOIN (
    SELECT * FROM commandes WHERE commandes.client_id = clients.id
) AS orders;
-- ERROR: invalid reference to FROM-clause entry for table "clients"

-- ✅ CORRECT : Ajouter LATERAL
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes WHERE commandes.client_id = clients.id
) AS orders;
```

### Erreur 2 : Confusion CROSS JOIN vs LEFT JOIN

```sql
-- CROSS JOIN LATERAL : Exclut les clients sans commande
SELECT clients.nom, orders.id
FROM clients
CROSS JOIN LATERAL (
    SELECT id FROM commandes WHERE commandes.client_id = clients.id LIMIT 1
) AS orders;

-- LEFT JOIN LATERAL : Inclut tous les clients
SELECT clients.nom, orders.id
FROM clients
LEFT JOIN LATERAL (
    SELECT id FROM commandes WHERE commandes.client_id = clients.id LIMIT 1
) AS orders ON true;
```

### Erreur 3 : Oublier l'Alias

```sql
-- ❌ ERREUR : Sous-requête doit avoir un alias
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes WHERE commandes.client_id = clients.id
);
-- ERROR: subquery in FROM must have an alias

-- ✅ CORRECT
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes WHERE commandes.client_id = clients.id
) AS orders;  -- Alias obligatoire
```

### Erreur 4 : Mauvais Ordre des Jointures

```sql
-- ❌ ERREUR : LATERAL fait référence à une table qui vient après
SELECT *
FROM commandes
CROSS JOIN LATERAL (
    SELECT * FROM clients WHERE clients.id = commandes.client_id
) AS client_info
CROSS JOIN orders;  -- orders n'est pas défini ici
```

**Règle** : LATERAL ne peut référencer que les tables déclarées **avant** lui dans FROM.

### Erreur 5 : Corrélation Circulaire

```sql
-- ❌ Impossible : Référence circulaire
SELECT *
FROM clients
CROSS JOIN LATERAL (
    SELECT * FROM commandes
    WHERE commandes.client_id = orders.some_column  -- orders pas encore défini
) AS orders;
```

---

## 11. Compatibilité et Standards SQL

### Support PostgreSQL

- **Introduit** : PostgreSQL 9.3 (2013)
- **Standard SQL** : SQL:2003 (partie de la norme)
- **Maturité** : Stable et largement utilisé

### Autres SGBD

| SGBD | Support LATERAL | Notes |
|------|-----------------|-------|
| PostgreSQL | ✅ Oui (depuis 9.3) | Support complet |
| MySQL | ❌ Non | Utilisez des sous-requêtes corrélées |
| SQL Server | ✅ Oui (CROSS APPLY / OUTER APPLY) | Syntaxe différente |
| Oracle | ✅ Oui (LATERAL) | Depuis Oracle 12c |
| SQLite | ❌ Non | Pas de support |

### Équivalents dans d'autres SGBD

#### SQL Server : CROSS APPLY / OUTER APPLY

```sql
-- PostgreSQL
SELECT * FROM clients
CROSS JOIN LATERAL (
    SELECT TOP 5 * FROM commandes WHERE commandes.client_id = clients.id
) AS orders;

-- SQL Server équivalent
SELECT * FROM clients
CROSS APPLY (
    SELECT TOP 5 * FROM commandes WHERE commandes.client_id = clients.id
) AS orders;
```

---

## 12. Exemples Pratiques Complets

### Exemple 1 : Tableau de Bord E-Commerce

**Objectif** : Pour chaque client, afficher :
- Dernière commande
- Commande la plus élevée
- Nombre total de commandes

```sql
SELECT
    clients.id,
    clients.nom,
    last_order.date_commande AS derniere_commande_date,
    last_order.montant AS derniere_commande_montant,
    top_order.montant AS commande_max,
    stats.total_commandes,
    stats.montant_total
FROM clients
LEFT JOIN LATERAL (
    SELECT date_commande, montant
    FROM commandes
    WHERE commandes.client_id = clients.id
    ORDER BY date_commande DESC
    LIMIT 1
) AS last_order ON true
LEFT JOIN LATERAL (
    SELECT montant
    FROM commandes
    WHERE commandes.client_id = clients.id
    ORDER BY montant DESC
    LIMIT 1
) AS top_order ON true
LEFT JOIN LATERAL (
    SELECT
        COUNT(*) AS total_commandes,
        COALESCE(SUM(montant), 0) AS montant_total
    FROM commandes
    WHERE commandes.client_id = clients.id
) AS stats ON true
ORDER BY clients.nom;
```

### Exemple 2 : Recommandations de Produits

**Objectif** : Pour chaque produit consulté, recommander 5 produits similaires (même catégorie, prix proche).

```sql
SELECT
    p.nom AS produit_consulte,
    recommandations.nom AS produit_recommande,
    recommandations.prix
FROM produits p
CROSS JOIN LATERAL (
    SELECT nom, prix
    FROM produits p2
    WHERE p2.categorie_id = p.categorie_id
      AND p2.id != p.id
      AND ABS(p2.prix - p.prix) <= 50
    ORDER BY ABS(p2.prix - p.prix), p2.ventes DESC
    LIMIT 5
) AS recommandations
WHERE p.id = 1;  -- Produit consulté
```

### Exemple 3 : Analyse de Rétention Client

**Objectif** : Pour chaque mois, trouver les clients qui ont commandé ce mois-là et le mois suivant (rétention).

```sql
WITH mois_commandes AS (
    SELECT DISTINCT
        DATE_TRUNC('month', date_commande) AS mois,
        client_id
    FROM commandes
)
SELECT
    m1.mois,
    COUNT(m1.client_id) AS clients_ce_mois,
    COUNT(retention.client_id) AS clients_retour_mois_suivant,
    ROUND(COUNT(retention.client_id)::NUMERIC / COUNT(m1.client_id) * 100, 2) AS taux_retention
FROM mois_commandes m1
LEFT JOIN LATERAL (
    SELECT client_id
    FROM mois_commandes m2
    WHERE m2.client_id = m1.client_id
      AND m2.mois = m1.mois + INTERVAL '1 month'
) AS retention ON true
GROUP BY m1.mois
ORDER BY m1.mois;
```

---

## 13. Récapitulatif : LATERAL en 10 Points

1. **LATERAL permet la corrélation** : Une sous-requête peut référencer les tables précédentes
2. **Syntaxe** : `CROSS JOIN LATERAL` ou `LEFT JOIN LATERAL`
3. **Cas d'usage principal** : Top N par groupe
4. **Avantages** :
   - Plus lisible que les sous-requêtes corrélées en WHERE
   - Permet LIMIT, ORDER BY corrélés
   - Retourne plusieurs lignes et colonnes
5. **CROSS JOIN LATERAL** = INNER (exclut lignes sans résultat)
6. **LEFT JOIN LATERAL** = LEFT (inclut toutes les lignes avec NULL)
7. **Performances** : Indexer les colonnes de corrélation
8. **Alternative** : Window Functions (souvent plus rapides pour Top N)
9. **Compatibilité** : PostgreSQL 9.3+, SQL:2003
10. **Équivalent SQL Server** : CROSS APPLY / OUTER APPLY

---

## 14. Tableau de Décision : Quelle Technique Utiliser ?

| Besoin | Technique Recommandée | Raison |
|--------|-----------------------|--------|
| Top N par groupe | LATERAL ou Window Functions | Dépend de la complexité |
| Rang par groupe | Window Functions | Plus performant |
| Appeler une fonction avec paramètres corrélés | LATERAL | Seule option |
| Sous-requête scalaire (1 valeur) | SELECT (sous-requête) | Plus simple |
| Filtre EXISTS / NOT EXISTS | WHERE EXISTS | Plus performant |
| Calculs cumulatifs | Window Functions | Plus adapté |
| Logique complexe par groupe | LATERAL | Plus flexible |

---

## Conclusion

### Points Clés à Retenir

1. **LATERAL** transforme une sous-requête en "consciente" des tables précédentes
2. C'est une **jointure corrélée** dans la clause FROM
3. Indispensable pour **Top N par groupe avec LIMIT**
4. Très utile avec les **fonctions retournant des ensembles**
5. **Alternative** : Window Functions (souvent plus rapides)
6. **Toujours indexer** les colonnes de corrélation

### Quand Utiliser LATERAL ?

✅ **Utilisez LATERAL** quand :
- Vous devez limiter les résultats par groupe (Top N)
- Vous appelez une fonction avec des paramètres corrélés
- La logique est plus claire avec une sous-requête corrélée
- Vous avez besoin de plusieurs colonnes/lignes par corrélation

⚠️ **Considérez des alternatives** quand :
- Window Functions peuvent faire le travail
- Les performances sont critiques sur très grandes tables
- La requête devient trop complexe

### Prochain Sujet

Dans la section suivante **(7.7 Anti-jointures et Semi-jointures)**, nous explorerons les techniques pour trouver ce qui **n'existe pas** ou **existe** dans une autre table, avec NOT EXISTS, NOT IN, et EXCEPT.

LATERAL JOIN est une arme puissante dans votre arsenal SQL. Bien utilisé, il simplifie des requêtes complexes et ouvre des possibilités impossibles autrement !

⏭️ [Anti-jointures et Semi-jointures (NOT EXISTS, NOT IN, EXCEPT)](/07-relations-et-jointures/07-anti-jointures-semi-jointures.md)
