🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.3. CTE Récursives : Parcourir des hiérarchies (arbres, graphes)

## Introduction

Les CTE récursives sont l'une des fonctionnalités les plus puissantes de SQL, permettant de parcourir des structures hiérarchiques (comme des organigrammes, des catégories imbriquées, des arbres de commentaires) et des graphes (comme des réseaux sociaux ou des dépendances).

**Attention :** Ce chapitre est plus avancé, mais nous allons progresser pas à pas avec des exemples concrets et visuels.

---

## Qu'est-ce qu'une structure hiérarchique ?

Une **hiérarchie** est une structure où des éléments sont organisés en niveaux parent-enfant.

### Exemples du quotidien

**Organigramme d'entreprise :**
```
PDG (Alice)
├── Directeur IT (Bob)
│   ├── Dev Senior (Charlie)
│   │   └── Dev Junior (Diana)
│   └── DevOps (Eve)
└── Directeur RH (Frank)
    └── Assistant RH (Grace)
```

**Catégories de produits :**
```
Électronique
├── Informatique
│   ├── Ordinateurs
│   │   ├── Portables
│   │   └── Fixes
│   └── Périphériques
└── Téléphonie
    ├── Smartphones
    └── Accessoires
```

**Système de fichiers :**
```
/
├── home
│   ├── alice
│   └── bob
└── var
    ├── log
    └── tmp
```

---

## Le problème : Comment interroger une hiérarchie en SQL ?

### Structure de données typique

```sql
CREATE TABLE employes (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(100),
    manager_id INTEGER REFERENCES employes(id)
);

INSERT INTO employes VALUES
    (1, 'Alice (PDG)', NULL),           -- Alice n'a pas de manager
    (2, 'Bob (Dir IT)', 1),             -- Bob reporte à Alice
    (3, 'Charlie (Dev Senior)', 2),     -- Charlie reporte à Bob
    (4, 'Diana (Dev Junior)', 3),       -- Diana reporte à Charlie
    (5, 'Eve (DevOps)', 2),             -- Eve reporte à Bob
    (6, 'Frank (Dir RH)', 1),           -- Frank reporte à Alice
    (7, 'Grace (Assistant RH)', 6);     -- Grace reporte à Frank
```

### Questions typiques

1. **Qui sont tous les subordonnés de Bob** (directs et indirects) ?
2. **Quel est le chemin hiérarchique complet de Diana** jusqu'au PDG ?
3. **Quelle est la profondeur de l'organigramme** ?
4. **Quels sont tous les employés au niveau 3** de la hiérarchie ?

**Sans CTE récursive**, ces requêtes sont très difficiles, voire impossibles en SQL standard.

**Avec CTE récursive**, elles deviennent simples ! 🎉

---

## Anatomie d'une CTE Récursive

### Structure générale

```sql
WITH RECURSIVE nom_cte AS (
    -- 1️⃣ TERME D'ANCRAGE (Anchor Term)
    -- Point de départ : requête non récursive
    SELECT ...
    FROM ...
    WHERE condition_de_départ

    UNION ALL  -- ⚠️ Presque toujours UNION ALL, pas UNION

    -- 2️⃣ TERME RÉCURSIF (Recursive Term)
    -- Itération : requête qui référence la CTE elle-même
    SELECT ...
    FROM nom_cte  -- ← Auto-référence !
    INNER JOIN ...
    WHERE condition_de_continuation
)
-- 3️⃣ REQUÊTE FINALE
SELECT * FROM nom_cte;
```

### Les trois composantes essentielles

#### 1. Terme d'ancrage (Anchor)
- Point de départ de la récursion
- Requête **non récursive** (ne référence pas la CTE)
- Exemple : `SELECT * FROM employes WHERE manager_id IS NULL` (les racines)

#### 2. Terme récursif (Recursive)
- Itération de la récursion
- Requête qui **référence la CTE elle-même**
- Exemple : `SELECT e.* FROM employes e JOIN cte ON e.manager_id = cte.id`

#### 3. UNION ALL
- Combine les résultats de chaque itération
- **UNION ALL** (conserve les doublons) est presque toujours utilisé
- UNION (élimine les doublons) peut être utilisé pour éviter les boucles infinies dans les graphes

---

## Premier exemple simple : Lister tous les subordonnés

### Objectif
Trouver tous les employés qui reportent à Bob (id=2), directement ou indirectement.

### Visualisation

```
Bob (2) ← Point de départ
├── Charlie (3) ← Itération 1
│   └── Diana (4) ← Itération 2
└── Eve (5) ← Itération 1
```

### Code SQL

```sql
WITH RECURSIVE subordonnes AS (
    -- 1️⃣ ANCRAGE : On commence avec Bob
    SELECT id, nom, manager_id, 1 AS niveau
    FROM employes
    WHERE id = 2  -- Bob

    UNION ALL

    -- 2️⃣ RÉCURSIF : On trouve les subordonnés de chaque personne
    SELECT e.id, e.nom, e.manager_id, s.niveau + 1
    FROM employes e
    INNER JOIN subordonnes s ON e.manager_id = s.id
)
SELECT * FROM subordonnes;
```

### Résultat

```
id |        nom         | manager_id | niveau
---+--------------------+------------+--------
 2 | Bob (Dir IT)       |          1 |      1
 3 | Charlie (Dev Senior)|         2 |      2
 5 | Eve (DevOps)       |          2 |      2
 4 | Diana (Dev Junior) |          3 |      3
```

### Comment ça fonctionne ? (Pas à pas)

**Itération 0 (Ancrage) :**
- PostgreSQL exécute le terme d'ancrage
- Résultat : Bob (id=2)
- Ce résultat devient le contenu temporaire de `subordonnes`

**Itération 1 (Récursion) :**
- PostgreSQL joint `employes` avec `subordonnes` (qui contient Bob)
- Trouve : Charlie (manager_id=2) et Eve (manager_id=2)
- Ajoute ces résultats à `subordonnes`

**Itération 2 (Récursion) :**
- PostgreSQL joint `employes` avec les nouveaux résultats (Charlie et Eve)
- Trouve : Diana (manager_id=3)
- Ajoute Diana à `subordonnes`

**Itération 3 (Récursion) :**
- PostgreSQL joint `employes` avec Diana
- Aucun employé n'a Diana comme manager
- **Condition d'arrêt atteinte** : la récursion se termine

**Résultat final :**
- PostgreSQL retourne tous les résultats accumulés

---

## Exemple 2 : Trouver la hiérarchie complète (de bas en haut)

### Objectif
Pour Diana (id=4), afficher le chemin hiérarchique complet jusqu'à la racine (PDG).

### Visualisation

```
Diana (4) → Charlie (3) → Bob (2) → Alice (1)
```

### Code SQL

```sql
WITH RECURSIVE hierarchie AS (
    -- 1️⃣ ANCRAGE : On commence avec Diana
    SELECT id, nom, manager_id, 1 AS niveau
    FROM employes
    WHERE id = 4  -- Diana

    UNION ALL

    -- 2️⃣ RÉCURSIF : On remonte vers les managers
    SELECT e.id, e.nom, e.manager_id, h.niveau + 1
    FROM employes e
    INNER JOIN hierarchie h ON e.id = h.manager_id  -- ← Inverser la jointure
)
SELECT
    niveau,
    nom,
    REPEAT('  ', niveau - 1) || '└─ ' || nom AS arborescence
FROM hierarchie
ORDER BY niveau DESC;  -- Du PDG vers Diana
```

### Résultat

```
niveau |        nom         |        arborescence
-------+--------------------+---------------------------
     4 | Alice (PDG)        | └─ Alice (PDG)
     3 | Bob (Dir IT)       |   └─ Bob (Dir IT)
     2 | Charlie (Dev Senior)|     └─ Charlie (Dev Senior)
     1 | Diana (Dev Junior) |       └─ Diana (Dev Junior)
```

---

## Exemple 3 : Arbre complet avec chemins

### Objectif
Afficher toute la hiérarchie avec le chemin complet de chaque employé.

### Code SQL

```sql
WITH RECURSIVE arbre AS (
    -- 1️⃣ ANCRAGE : Racines (employés sans manager)
    SELECT
        id,
        nom,
        manager_id,
        1 AS niveau,
        nom AS chemin,  -- Le chemin commence avec le nom de la racine
        ARRAY[id] AS ids_chemin  -- Tableau des IDs du chemin
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    -- 2️⃣ RÉCURSIF : Enfants de chaque nœud
    SELECT
        e.id,
        e.nom,
        e.manager_id,
        a.niveau + 1,
        a.chemin || ' → ' || e.nom,  -- Concaténation du chemin
        a.ids_chemin || e.id  -- Ajout de l'ID au tableau
    FROM employes e
    INNER JOIN arbre a ON e.manager_id = a.id
)
SELECT
    niveau,
    REPEAT('  ', niveau - 1) || nom AS nom_indente,
    chemin
FROM arbre
ORDER BY ids_chemin;  -- Tri par chemin pour ordre hiérarchique
```

### Résultat

```
niveau | nom_indente              | chemin
-------+--------------------------+-----------------------------------------------
     1 | Alice (PDG)              | Alice (PDG)
     2 |   Bob (Dir IT)           | Alice (PDG) → Bob (Dir IT)
     3 |     Charlie (Dev Senior) | Alice (PDG) → Bob (Dir IT) → Charlie (Dev Senior)
     4 |       Diana (Dev Junior) | Alice (PDG) → Bob (Dir IT) → Charlie (Dev Senior) → Diana (Dev Junior)
     3 |     Eve (DevOps)         | Alice (PDG) → Bob (Dir IT) → Eve (DevOps)
     2 |   Frank (Dir RH)         | Alice (PDG) → Frank (Dir RH)
     3 |     Grace (Assistant RH) | Alice (PDG) → Frank (Dir RH) → Grace (Assistant RH)
```

---

## Exemple 4 : Catégories de produits avec profondeur

### Structure de données

```sql
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(100),
    parent_id INTEGER REFERENCES categories(id)
);

INSERT INTO categories VALUES
    (1, 'Électronique', NULL),
    (2, 'Informatique', 1),
    (3, 'Téléphonie', 1),
    (4, 'Ordinateurs', 2),
    (5, 'Périphériques', 2),
    (6, 'Portables', 4),
    (7, 'Fixes', 4),
    (8, 'Smartphones', 3),
    (9, 'Accessoires', 3);
```

### Requête : Afficher l'arbre avec la profondeur

```sql
WITH RECURSIVE arbre_categories AS (
    -- Racines
    SELECT
        id,
        nom,
        parent_id,
        0 AS profondeur,
        nom AS chemin_complet
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Récursion
    SELECT
        c.id,
        c.nom,
        c.parent_id,
        ac.profondeur + 1,
        ac.chemin_complet || ' / ' || c.nom
    FROM categories c
    INNER JOIN arbre_categories ac ON c.parent_id = ac.id
)
SELECT
    REPEAT('  ', profondeur) || '├─ ' || nom AS arborescence,
    profondeur,
    chemin_complet
FROM arbre_categories
ORDER BY chemin_complet;
```

### Résultat

```
arborescence                | profondeur | chemin_complet
---------------------------+------------+------------------------------------------
├─ Électronique            |          0 | Électronique
  ├─ Informatique          |          1 | Électronique / Informatique
    ├─ Ordinateurs         |          2 | Électronique / Informatique / Ordinateurs
      ├─ Fixes             |          3 | Électronique / Informatique / Ordinateurs / Fixes
      ├─ Portables         |          3 | Électronique / Informatique / Ordinateurs / Portables
    ├─ Périphériques       |          2 | Électronique / Informatique / Périphériques
  ├─ Téléphonie            |          1 | Électronique / Téléphonie
    ├─ Accessoires         |          2 | Électronique / Téléphonie / Accessoires
    ├─ Smartphones         |          2 | Électronique / Téléphonie / Smartphones
```

---

## Exemple 5 : Parcourir un graphe (réseau social)

### Différence entre arbre et graphe

- **Arbre** : Chaque nœud a **un seul parent** (ou aucun)
- **Graphe** : Un nœud peut avoir **plusieurs parents** ou connexions

### Structure de données : Amis sur un réseau social

```sql
CREATE TABLE utilisateurs (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(100)
);

CREATE TABLE amities (
    utilisateur_id INTEGER REFERENCES utilisateurs(id),
    ami_id INTEGER REFERENCES utilisateurs(id),
    PRIMARY KEY (utilisateur_id, ami_id)
);

INSERT INTO utilisateurs VALUES
    (1, 'Alice'), (2, 'Bob'), (3, 'Charlie'),
    (4, 'Diana'), (5, 'Eve'), (6, 'Frank');

INSERT INTO amities VALUES
    (1, 2), (2, 1),  -- Alice ↔ Bob
    (2, 3), (3, 2),  -- Bob ↔ Charlie
    (3, 4), (4, 3),  -- Charlie ↔ Diana
    (1, 5), (5, 1),  -- Alice ↔ Eve
    (5, 6), (6, 5);  -- Eve ↔ Frank
```

### Requête : Trouver tous les amis à distance N

**Question :** Qui sont tous les amis d'Alice jusqu'à 3 degrés de séparation ?

```sql
WITH RECURSIVE reseau AS (
    -- ANCRAGE : Alice elle-même (degré 0)
    SELECT
        id,
        nom,
        0 AS degre,
        ARRAY[id] AS chemin_ids  -- Pour éviter les cycles
    FROM utilisateurs
    WHERE id = 1  -- Alice

    UNION  -- ⚠️ UNION (pas ALL) pour éviter les doublons/cycles

    -- RÉCURSIF : Amis des amis
    SELECT
        u.id,
        u.nom,
        r.degre + 1,
        r.chemin_ids || u.id
    FROM reseau r
    INNER JOIN amities a ON r.id = a.utilisateur_id
    INNER JOIN utilisateurs u ON a.ami_id = u.id
    WHERE
        r.degre < 3  -- Limiter à 3 degrés
        AND NOT (u.id = ANY(r.chemin_ids))  -- Éviter les cycles
)
SELECT DISTINCT  -- Au cas où
    degre,
    nom
FROM reseau
WHERE degre > 0  -- Exclure Alice elle-même
ORDER BY degre, nom;
```

### Résultat

```
degre | nom
------+---------
    1 | Bob
    1 | Eve
    2 | Charlie
    2 | Frank
    3 | Diana
```

**Explication :**
- **Degré 1** : Amis directs d'Alice → Bob, Eve
- **Degré 2** : Amis des amis → Charlie (ami de Bob), Frank (ami d'Eve)
- **Degré 3** : Amis de degré 2 → Diana (ami de Charlie)

---

## Détection de cycles et prévention de boucles infinies

### Le danger des boucles infinies

Dans les graphes, les cycles sont courants. Sans précaution, une CTE récursive peut boucler indéfiniment.

**Exemple de cycle :**
```
A → B → C → A  (cycle de 3 nœuds)
```

### Stratégie 1 : Utiliser UNION (sans ALL)

`UNION` élimine automatiquement les doublons, empêchant de revisiter les mêmes nœuds.

```sql
WITH RECURSIVE parcours AS (
    SELECT id, nom FROM noeuds WHERE id = 1
    UNION  -- ← Pas ALL !
    SELECT n.id, n.nom
    FROM noeuds n
    INNER JOIN liens l ON n.id = l.destination
    INNER JOIN parcours p ON l.source = p.id
)
SELECT * FROM parcours;
```

**Inconvénient :** Plus lent (tri nécessaire pour éliminer les doublons).

### Stratégie 2 : Suivre le chemin avec un tableau

Maintenir un tableau des nœuds visités et vérifier qu'on ne revisite pas.

```sql
WITH RECURSIVE parcours AS (
    SELECT
        id,
        nom,
        ARRAY[id] AS chemin_ids  -- Tableau des IDs visités
    FROM noeuds
    WHERE id = 1

    UNION ALL

    SELECT
        n.id,
        n.nom,
        p.chemin_ids || n.id
    FROM noeuds n
    INNER JOIN liens l ON n.id = l.destination
    INNER JOIN parcours p ON l.source = p.id
    WHERE NOT (n.id = ANY(p.chemin_ids))  -- ← Vérification anti-cycle
)
SELECT * FROM parcours;
```

**Avantage :** Plus rapide, contrôle fin.

### Stratégie 3 : Limiter la profondeur

Ajouter une limite explicite au nombre d'itérations.

```sql
WITH RECURSIVE parcours AS (
    SELECT id, nom, 0 AS profondeur
    FROM noeuds WHERE id = 1

    UNION ALL

    SELECT n.id, n.nom, p.profondeur + 1
    FROM noeuds n
    INNER JOIN liens l ON n.id = l.destination
    INNER JOIN parcours p ON l.source = p.id
    WHERE p.profondeur < 10  -- ← Limite de profondeur
)
SELECT * FROM parcours;
```

---

## Cas d'Usage Avancés

### 1. Calculer la profondeur maximale d'un arbre

```sql
WITH RECURSIVE profondeurs AS (
    SELECT id, nom, 1 AS profondeur
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.nom, p.profondeur + 1
    FROM employes e
    INNER JOIN profondeurs p ON e.manager_id = p.id
)
SELECT MAX(profondeur) AS profondeur_max
FROM profondeurs;
```

### 2. Trouver toutes les feuilles d'un arbre

Les feuilles sont les nœuds sans enfants.

```sql
WITH RECURSIVE arbre AS (
    SELECT id, nom, manager_id
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.nom, e.manager_id
    FROM employes e
    INNER JOIN arbre a ON e.manager_id = a.id
)
SELECT a.id, a.nom
FROM arbre a
LEFT JOIN employes e ON e.manager_id = a.id
WHERE e.id IS NULL;  -- Pas d'enfants
```

### 3. Calculer des agrégats hiérarchiques

**Exemple :** Calculer la masse salariale totale de chaque manager (incluant tous ses subordonnés).

```sql
WITH RECURSIVE hierarchie AS (
    SELECT
        id,
        nom,
        manager_id,
        salaire,
        salaire AS masse_salariale_totale
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.id,
        e.nom,
        e.manager_id,
        e.salaire,
        e.salaire AS masse_salariale_totale
    FROM employes e
    INNER JOIN hierarchie h ON e.manager_id = h.id
)
-- Agrégation post-récursion
SELECT
    manager.id,
    manager.nom,
    SUM(subordonnes.salaire) AS masse_salariale_equipe
FROM hierarchie manager
LEFT JOIN hierarchie subordonnes
    ON subordonnes.chemin_ids @> ARRAY[manager.id]  -- Contient l'ID du manager
GROUP BY manager.id, manager.nom;
```

### 4. Matérialiser un arbre dans une table dénormalisée

```sql
CREATE TABLE employes_denormalises AS
WITH RECURSIVE arbre AS (
    SELECT
        id,
        nom,
        manager_id,
        1 AS niveau,
        nom AS chemin_complet,
        id::TEXT AS chemin_ids
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.id,
        e.nom,
        e.manager_id,
        a.niveau + 1,
        a.chemin_complet || ' → ' || e.nom,
        a.chemin_ids || '/' || e.id
    FROM employes e
    INNER JOIN arbre a ON e.manager_id = a.id
)
SELECT * FROM arbre;

-- Créer un index pour les requêtes rapides
CREATE INDEX idx_niveau ON employes_denormalises(niveau);
CREATE INDEX idx_chemin ON employes_denormalises USING GIN(chemin_ids);
```

---

## Exemple Complet : Bill of Materials (BOM)

Les BOM (nomenclatures) sont très courantes en industrie : une pièce complexe est composée de sous-pièces, elles-mêmes composées de sous-sous-pièces, etc.

### Structure de données

```sql
CREATE TABLE pieces (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(100),
    prix_unitaire NUMERIC(10, 2)
);

CREATE TABLE compositions (
    piece_parent_id INTEGER REFERENCES pieces(id),
    piece_composant_id INTEGER REFERENCES pieces(id),
    quantite INTEGER,
    PRIMARY KEY (piece_parent_id, piece_composant_id)
);

INSERT INTO pieces VALUES
    (1, 'Vélo', 0),  -- Prix calculé
    (2, 'Cadre', 100),
    (3, 'Roue', 50),
    (4, 'Pneu', 15),
    (5, 'Jante', 30),
    (6, 'Rayons (lot)', 5),
    (7, 'Guidon', 25),
    (8, 'Selle', 40);

INSERT INTO compositions VALUES
    (1, 2, 1),  -- 1 Vélo = 1 Cadre
    (1, 3, 2),  -- 1 Vélo = 2 Roues
    (1, 7, 1),  -- 1 Vélo = 1 Guidon
    (1, 8, 1),  -- 1 Vélo = 1 Selle
    (3, 4, 1),  -- 1 Roue = 1 Pneu
    (3, 5, 1),  -- 1 Roue = 1 Jante
    (3, 6, 1);  -- 1 Roue = 1 lot de Rayons
```

### Requête : BOM explosée avec coût total

```sql
WITH RECURSIVE bom AS (
    -- ANCRAGE : Pièce de départ (le vélo)
    SELECT
        1 AS piece_finale_id,
        id AS piece_id,
        nom,
        prix_unitaire,
        1 AS quantite_totale,
        0 AS niveau,
        nom AS chemin
    FROM pieces
    WHERE id = 1  -- Vélo

    UNION ALL

    -- RÉCURSIF : Composants de chaque pièce
    SELECT
        b.piece_finale_id,
        p.id,
        p.nom,
        p.prix_unitaire,
        b.quantite_totale * c.quantite,  -- Multiplication en cascade
        b.niveau + 1,
        b.chemin || ' → ' || p.nom
    FROM bom b
    INNER JOIN compositions c ON b.piece_id = c.piece_parent_id
    INNER JOIN pieces p ON c.piece_composant_id = p.id
)
SELECT
    REPEAT('  ', niveau) || nom AS structure,
    quantite_totale AS qte,
    prix_unitaire AS prix_unit,
    quantite_totale * prix_unitaire AS cout_total,
    chemin
FROM bom
WHERE niveau > 0  -- Exclure la pièce finale elle-même
ORDER BY chemin;
```

### Résultat

```
structure        | qte | prix_unit | cout_total | chemin
-----------------+-----+-----------+------------+--------------------
  Cadre          |   1 |    100.00 |     100.00 | Vélo → Cadre
  Guidon         |   1 |     25.00 |      25.00 | Vélo → Guidon
  Roue           |   2 |     50.00 |     100.00 | Vélo → Roue
    Jante        |   2 |     30.00 |      60.00 | Vélo → Roue → Jante
    Pneu         |   2 |     15.00 |      30.00 | Vélo → Roue → Pneu
    Rayons (lot) |   2 |      5.00 |      10.00 | Vélo → Roue → Rayons (lot)
  Selle          |   1 |     40.00 |      40.00 | Vélo → Selle

-- Coût total du vélo : 100 + 25 + 60 + 30 + 10 + 40 = 265 €
```

### Calculer le coût total

```sql
WITH RECURSIVE bom AS (
    -- Même requête que ci-dessus
    ...
)
SELECT
    'Vélo complet' AS article,
    SUM(quantite_totale * prix_unitaire) AS cout_total
FROM bom
WHERE niveau > 0;
```

**Résultat :** `265.00 €`

---

## Performance et Optimisation

### 1. Limiter la profondeur de récursion

PostgreSQL a une limite par défaut de **100 niveaux de récursion**. Vous pouvez l'ajuster avec un paramètre :

```sql
-- Augmenter la limite (session actuelle)
SET max_recursion_depth = 1000;

-- Ou dans la requête elle-même
WITH RECURSIVE arbre AS (
    ...
)
OPTIONS (max_recursion_depth 500)
SELECT * FROM arbre;
```

**Attention :** Une limite trop élevée peut causer des problèmes de mémoire.

### 2. Indexer les colonnes de jointure

Les CTE récursives font de nombreuses jointures. Des index sur les colonnes clés sont **essentiels**.

```sql
-- Pour une structure parent-enfant
CREATE INDEX idx_manager ON employes(manager_id);
CREATE INDEX idx_parent ON categories(parent_id);

-- Pour un graphe
CREATE INDEX idx_source ON liens(source);
CREATE INDEX idx_destination ON liens(destination);
```

### 3. Éviter les colonnes inutiles

Ne sélectionnez que les colonnes nécessaires dans la CTE récursive.

```sql
-- ❌ MOINS EFFICACE
WITH RECURSIVE arbre AS (
    SELECT * FROM employes WHERE ...  -- Toutes les colonnes
    UNION ALL
    SELECT e.* FROM employes e ...
)
SELECT * FROM arbre;

-- ✅ PLUS EFFICACE
WITH RECURSIVE arbre AS (
    SELECT id, nom, manager_id FROM employes WHERE ...
    UNION ALL
    SELECT e.id, e.nom, e.manager_id FROM employes e ...
)
SELECT * FROM arbre;
```

### 4. Utiliser UNION vs UNION ALL intelligemment

- **UNION ALL** : Plus rapide (pas de déduplication)
- **UNION** : Empêche les cycles mais plus lent

**Recommandation :** Utilisez `UNION ALL` avec détection de cycle manuelle (tableau de chemin).

### 5. Matérialiser pour réutilisation

Si vous utilisez le même arbre plusieurs fois, matérialisez-le dans une table temporaire :

```sql
CREATE TEMP TABLE arbre_materialise AS
WITH RECURSIVE arbre AS (
    ...
)
SELECT * FROM arbre;

CREATE INDEX ON arbre_materialise(id);
CREATE INDEX ON arbre_materialise(parent_id);

-- Utilisez ensuite arbre_materialise dans vos requêtes
SELECT * FROM arbre_materialise WHERE niveau = 3;
```

---

## Analyse avec EXPLAIN

```sql
EXPLAIN ANALYZE
WITH RECURSIVE arbre AS (
    SELECT id, nom, manager_id, 1 AS niveau
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.nom, e.manager_id, a.niveau + 1
    FROM employes e
    INNER JOIN arbre a ON e.manager_id = a.id
)
SELECT * FROM arbre;
```

### Éléments à surveiller dans le plan

- **Recursive CTE** : Confirme que PostgreSQL utilise la récursion
- **WorkTable Scan** : Scan de la table de travail temporaire (résultats intermédiaires)
- **Nested Loop** ou **Hash Join** : Type de jointure utilisée à chaque itération
- **Execution Time** : Temps total (attention aux arbres profonds)

---

## Limitations et Considérations

### 1. Pas de matérialisation explicite

Les CTE récursives ne peuvent pas être déclarées `MATERIALIZED` ou `NOT MATERIALIZED`.

### 2. Mémoire

Les résultats intermédiaires sont stockés en mémoire (ou sur disque si trop gros). Des arbres énormes peuvent saturer la mémoire.

### 3. Performances sur des graphes très connectés

Dans des graphes denses (beaucoup de connexions), les CTE récursives peuvent devenir lentes. Alternatives :
- Extension `pg_graph` (si disponible)
- Matérialiser l'arbre dans une table
- Algorithmes spécialisés côté application

### 4. Ordre d'exécution

L'ordre des résultats dans une CTE récursive n'est **pas garanti** sans `ORDER BY` explicite dans la requête finale.

---

## Comparaison avec d'autres approches

### CTE Récursive vs Closures (tables de clôture transitive)

**Closure table** : Table précalculée de toutes les relations ancêtre-descendant.

```sql
CREATE TABLE employes_closure (
    ancetre_id INTEGER,
    descendant_id INTEGER,
    profondeur INTEGER,
    PRIMARY KEY (ancetre_id, descendant_id)
);

-- Pré-remplir avec toutes les relations
INSERT INTO employes_closure
WITH RECURSIVE ...;
```

| Approche | Avantages | Inconvénients |
|----------|-----------|---------------|
| **CTE Récursive** | Pas de stockage additionnel, toujours à jour | Calcul à chaque requête |
| **Closure Table** | Requêtes très rapides (simple SELECT) | Maintenance complexe (INSERT/UPDATE/DELETE) |

**Recommandation :**
- **CTE Récursive** : Données qui changent fréquemment, structures petites/moyennes
- **Closure Table** : Données stables, requêtes fréquentes, grandes hiérarchies

### CTE Récursive vs ltree (extension PostgreSQL)

**ltree** : Extension PostgreSQL pour stocker des hiérarchies sous forme de chemins.

```sql
CREATE EXTENSION ltree;

CREATE TABLE categories_ltree (
    id INTEGER PRIMARY KEY,
    nom VARCHAR(100),
    chemin ltree  -- Ex: 'Électronique.Informatique.Ordinateurs'
);

CREATE INDEX idx_chemin_gist ON categories_ltree USING GIST (chemin);

-- Requête ultra-rapide
SELECT * FROM categories_ltree
WHERE chemin <@ 'Électronique.Informatique';  -- Tous les descendants
```

**Avantage ltree :** Performance supérieure pour les requêtes hiérarchiques.

**Inconvénient ltree :** Nécessite de maintenir le chemin à jour lors des modifications.

---

## Bonnes Pratiques

### ✅ À faire

1. **Toujours inclure une condition d'arrêt**
   ```sql
   WHERE niveau < 100  -- Limite explicite
   WHERE NOT (id = ANY(chemin_ids))  -- Détection de cycle
   ```

2. **Utiliser des index sur les colonnes de jointure**
   ```sql
   CREATE INDEX idx_parent ON table(parent_id);
   ```

3. **Ajouter un niveau/profondeur pour le monitoring**
   ```sql
   SELECT ..., niveau + 1 AS niveau
   ```

4. **Tester avec de petits ensembles de données d'abord**

5. **Utiliser UNION ALL sauf si cycles attendus**

6. **Documenter les CTE récursives**
   ```sql
   -- Cette CTE parcourt l'organigramme à partir du PDG (id=1)
   -- et calcule la profondeur de chaque employé
   WITH RECURSIVE ...
   ```

### ❌ À éviter

1. **Oublier la condition d'arrêt**
   → Risque de boucle infinie

2. **Pas d'index sur parent_id / manager_id**
   → Performances catastrophiques

3. **Sélectionner toutes les colonnes inutilement**
   → Gaspillage mémoire

4. **Utiliser UNION quand UNION ALL suffit**
   → Perte de performance inutile

5. **Négliger les cycles dans les graphes**
   → Boucles infinies garanties

---

## Résumé

### Points clés à retenir

| Concept | Description |
|---------|-------------|
| **CTE Récursive** | Requête qui se référence elle-même pour parcourir des hiérarchies |
| **Terme d'ancrage** | Point de départ (racines) |
| **Terme récursif** | Itération qui joint la CTE avec elle-même |
| **UNION ALL** | Combine les résultats de chaque itération |
| **Condition d'arrêt** | INDISPENSABLE pour éviter les boucles infinies |
| **Applications** | Organigrammes, catégories, BOM, réseaux sociaux, graphes |

### Structure type

```sql
WITH RECURSIVE nom AS (
    -- 1. ANCRAGE (non récursif)
    SELECT colonnes, 0 AS niveau
    FROM table
    WHERE condition_racine

    UNION ALL

    -- 2. RÉCURSIF (auto-référence)
    SELECT colonnes, n.niveau + 1
    FROM table t
    INNER JOIN nom n ON t.parent = n.id
    WHERE condition_continuation  -- ← IMPORTANT !
)
SELECT * FROM nom;
```

### Checklist d'utilisation

- [ ] Ai-je défini un terme d'ancrage clair ?
- [ ] Ai-je une condition d'arrêt explicite ?
- [ ] Ai-je géré les cycles potentiels (graphes) ?
- [ ] Les colonnes de jointure sont-elles indexées ?
- [ ] Ai-je testé avec un petit jeu de données ?
- [ ] Ai-je vérifié avec EXPLAIN ANALYZE ?

---

## Pour aller plus loin

- **ltree Extension** : Hiérarchies optimisées avec chemins matérialisés
- **Closure Tables** : Pré-calcul de toutes les relations
- **Graph Databases** : Neo4j, RedisGraph pour graphes complexes
- **Window Functions** : Alternative pour certains calculs hiérarchiques

---

**Prochain chapitre :** 9.4. Opérations d'ensemble : UNION, INTERSECT, EXCEPT

---


⏭️ [Opérations d'ensemble : UNION, INTERSECT, EXCEPT (ALL)](/09-techniques-sql-avancees/04-operations-ensemble.md)
