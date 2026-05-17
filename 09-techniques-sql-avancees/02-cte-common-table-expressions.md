🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2. CTE (Common Table Expressions) et l'option MATERIALIZED

## Introduction

Les CTE (Common Table Expressions), également appelées "clauses WITH", sont une fonctionnalité SQL puissante qui permet de créer des résultats intermédiaires nommés et réutilisables dans une requête. Pensez-y comme à des "variables temporaires" pour vos requêtes SQL.

Les CTE améliorent considérablement la **lisibilité**, la **maintenabilité** et parfois les **performances** de vos requêtes SQL complexes.

---

## Qu'est-ce qu'une CTE ?

Une CTE est une requête nommée définie au début d'une instruction SQL principale avec le mot-clé `WITH`. Elle crée un ensemble de résultats temporaire qui existe uniquement le temps de l'exécution de la requête.

### Analogie

Imaginez que vous préparez un plat complexe :

- **Sans CTE** : Vous faites tout en même temps dans une seule grande casserole  
- **Avec CTE** : Vous préparez chaque ingrédient dans un bol séparé et nommé, puis vous les combinez

Les CTE sont les "bols préparatoires" de SQL.

---

## Syntaxe de base

```sql
WITH nom_cte AS (
    -- Requête de définition de la CTE
    SELECT ...
)
-- Requête principale qui utilise la CTE
SELECT ...  
FROM nom_cte  
WHERE ...;  
```

### Premier exemple simple

Sans CTE (avec sous-requête) :

```sql
-- Requête peu lisible avec sous-requête
SELECT
    e.nom,
    e.salaire,
    dept_avg.moyenne
FROM employes e  
INNER JOIN (  
    SELECT
        departement,
        AVG(salaire) AS moyenne
    FROM employes
    GROUP BY departement
) AS dept_avg ON e.departement = dept_avg.departement
WHERE e.salaire > dept_avg.moyenne;
```

Avec CTE (beaucoup plus lisible) :

```sql
-- Même requête avec CTE : claire et structurée
WITH salaires_moyens_dept AS (
    SELECT
        departement,
        AVG(salaire) AS moyenne
    FROM employes
    GROUP BY departement
)
SELECT
    e.nom,
    e.salaire,
    sm.moyenne
FROM employes e  
INNER JOIN salaires_moyens_dept sm ON e.departement = sm.departement  
WHERE e.salaire > sm.moyenne;  
```

**Avantages immédiats :**
- ✅ Le code se lit de haut en bas (logique séquentielle)  
- ✅ La CTE a un nom descriptif (`salaires_moyens_dept`)  
- ✅ On peut facilement tester la CTE isolément  
- ✅ Pas de parenthèses imbriquées complexes

---

## Pourquoi utiliser les CTE ?

### 1. Lisibilité améliorée

Les CTE décomposent une requête complexe en étapes logiques nommées.

```sql
-- Exemple : Analyse de ventes
WITH
-- Étape 1 : Calculer les ventes par produit
ventes_produits AS (
    SELECT
        produit_id,
        SUM(quantite * prix_unitaire) AS total_ventes
    FROM commandes
    GROUP BY produit_id
),
-- Étape 2 : Calculer la moyenne des ventes
moyenne_ventes AS (
    SELECT AVG(total_ventes) AS moyenne
    FROM ventes_produits
)
-- Étape 3 : Trouver les produits au-dessus de la moyenne
SELECT
    p.nom,
    vp.total_ventes,
    mv.moyenne
FROM produits p  
INNER JOIN ventes_produits vp ON p.id = vp.produit_id  
CROSS JOIN moyenne_ventes mv  
WHERE vp.total_ventes > mv.moyenne  
ORDER BY vp.total_ventes DESC;  
```

### 2. Réutilisation dans la même requête

Une CTE peut être référencée plusieurs fois sans la réécrire.

```sql
WITH employes_actifs AS (
    SELECT *
    FROM employes
    WHERE statut = 'actif' AND date_fin IS NULL
)
SELECT
    (SELECT COUNT(*) FROM employes_actifs) AS total_actifs,
    (SELECT AVG(salaire) FROM employes_actifs) AS salaire_moyen,
    (SELECT MAX(anciennete) FROM employes_actifs) AS anciennete_max;
```

### 3. Facilité de maintenance

Modifier une CTE est simple : un seul endroit à changer.

```sql
WITH donnees_filtrees AS (
    -- Si les critères de filtrage changent,
    -- on ne modifie qu'ici
    SELECT *
    FROM ventes
    WHERE annee = 2024
      AND region = 'Europe'
)
SELECT
    categorie,
    SUM(montant) AS total
FROM donnees_filtrees  
GROUP BY categorie  

UNION ALL

SELECT
    'TOTAL' AS categorie,
    SUM(montant) AS total
FROM donnees_filtrees;
```

### 4. Facilité de test et debug

Vous pouvez tester chaque CTE indépendamment :

```sql
-- Test de la CTE seule
WITH ventes_2024 AS (
    SELECT * FROM ventes WHERE annee = 2024
)
SELECT * FROM ventes_2024 LIMIT 10;

-- Puis l'utiliser dans la requête complète
WITH ventes_2024 AS (
    SELECT * FROM ventes WHERE annee = 2024
)
SELECT categorie, SUM(montant)  
FROM ventes_2024  
GROUP BY categorie;  
```

---

## CTE Multiples

Vous pouvez définir plusieurs CTE séparées par des virgules. Chaque CTE peut référencer les CTE définies avant elle.

### Syntaxe

```sql
WITH
    cte1 AS (
        SELECT ...
    ),
    cte2 AS (
        SELECT ... FROM cte1 ...  -- Peut utiliser cte1
    ),
    cte3 AS (
        SELECT ... FROM cte1, cte2 ...  -- Peut utiliser cte1 et cte2
    )
SELECT ...  
FROM cte3;  
```

### Exemple : Pipeline de transformation de données

```sql
WITH
-- Étape 1 : Nettoyer les données
donnees_nettoyees AS (
    SELECT
        id,
        TRIM(nom) AS nom,
        COALESCE(email, 'non_renseigne@example.com') AS email,
        CASE
            WHEN age < 0 THEN NULL
            WHEN age > 120 THEN NULL
            ELSE age
        END AS age
    FROM clients_brut
),
-- Étape 2 : Enrichir avec des calculs
donnees_enrichies AS (
    SELECT
        *,
        CASE
            WHEN age BETWEEN 18 AND 25 THEN 'Jeune'
            WHEN age BETWEEN 26 AND 50 THEN 'Adulte'
            WHEN age > 50 THEN 'Senior'
            ELSE 'Non défini'
        END AS tranche_age
    FROM donnees_nettoyees
),
-- Étape 3 : Agréger par tranche d'âge
stats_par_tranche AS (
    SELECT
        tranche_age,
        COUNT(*) AS nombre_clients,
        AVG(age) AS age_moyen
    FROM donnees_enrichies
    GROUP BY tranche_age
)
-- Résultat final
SELECT
    tranche_age,
    nombre_clients,
    ROUND(age_moyen, 1) AS age_moyen,
    ROUND(100.0 * nombre_clients / SUM(nombre_clients) OVER (), 2) AS pourcentage
FROM stats_par_tranche  
ORDER BY age_moyen;  
```

**Avantage :** Chaque étape est claire et testable individuellement.

---

## L'option MATERIALIZED

PostgreSQL offre un contrôle fin sur l'optimisation des CTE avec les mots-clés `MATERIALIZED` et `NOT MATERIALIZED`.

### Comportement par défaut (PostgreSQL 12+)

Par défaut, PostgreSQL décide automatiquement s'il faut matérialiser une CTE ou non, en fonction de son utilisation :

- **Une seule utilisation** → La CTE est "inline" (fusionnée dans la requête principale)  
- **Multiples utilisations** → La CTE est matérialisée

### Qu'est-ce que la matérialisation ?

**Matérialiser** signifie exécuter la CTE une fois, stocker son résultat en mémoire (ou sur disque), puis réutiliser ce résultat.

**Ne pas matérialiser** (inline) signifie que la CTE est remplacée par sa définition dans la requête principale, permettant à l'optimiseur de créer un plan d'exécution global optimisé.

### Syntaxe

```sql
-- Forcer la matérialisation
WITH cte_name AS MATERIALIZED (
    SELECT ...
)
SELECT ...;

-- Empêcher la matérialisation (inline)
WITH cte_name AS NOT MATERIALIZED (
    SELECT ...
)
SELECT ...;
```

---

## MATERIALIZED : Quand et pourquoi l'utiliser ?

### Cas d'usage 1 : CTE utilisée plusieurs fois

Si une CTE est référencée plusieurs fois et que son calcul est coûteux, la matérialisation évite de la recalculer.

> 💡 **À noter (PG 12+)** : quand une CTE non récursive et sans effet de bord est référencée **deux fois ou plus**, PostgreSQL la matérialise **automatiquement** par défaut — donc le `MATERIALIZED` explicite n'est qu'une garantie/documentation, pas une obligation. À l'inverse, pour une CTE référencée **une seule fois**, le défaut est *inline*.

```sql
-- CTE référencée 3 fois : matérialisée par défaut en PG 12+
WITH stats_ventes AS (
    SELECT
        categorie,
        COUNT(*) AS nb_ventes,
        SUM(montant) AS total
    FROM ventes
    WHERE annee = 2024
    GROUP BY categorie
)
SELECT 'Catégorie A' AS type, * FROM stats_ventes WHERE categorie = 'A'  
UNION ALL  
SELECT 'Catégorie B' AS type, * FROM stats_ventes WHERE categorie = 'B'  
UNION ALL  
SELECT 'Toutes' AS type, * FROM stats_ventes;  
```

**Forme avec `MATERIALIZED` explicite (équivalente, mais auto-documentée) :**

```sql
WITH stats_ventes AS MATERIALIZED (
    SELECT
        categorie,
        COUNT(*) AS nb_ventes,
        SUM(montant) AS total
    FROM ventes
    WHERE annee = 2024
    GROUP BY categorie
)
SELECT 'Catégorie A' AS type, * FROM stats_ventes WHERE categorie = 'A'  
UNION ALL  
SELECT 'Catégorie B' AS type, * FROM stats_ventes WHERE categorie = 'B'  
UNION ALL  
SELECT 'Toutes' AS type, * FROM stats_ventes;  
```

**Résultat :** La requête `stats_ventes` est exécutée **1 fois** au lieu de 3.

### Cas d'usage 2 : Barrière d'optimisation volontaire

Parfois, l'optimiseur de PostgreSQL prend de mauvaises décisions. Matérialiser une CTE force l'exécution dans un ordre spécifique.

```sql
-- L'optimiseur peut mal estimer ce filtre
WITH grands_clients AS MATERIALIZED (
    SELECT client_id
    FROM commandes
    GROUP BY client_id
    HAVING SUM(montant) > 100000
)
SELECT c.nom, gc.client_id  
FROM grands_clients gc  
INNER JOIN clients c ON gc.client_id = c.id;  
```

**Avantage :** Garantit que le filtrage coûteux (`HAVING SUM(montant) > 100000`) se fait avant la jointure.

### Cas d'usage 3 : Éviter des calculs longs répétés

```sql
-- Calculs statistiques complexes réutilisés
WITH statistiques_complexes AS MATERIALIZED (
    SELECT
        produit_id,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY prix) AS mediane_prix,
        PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY prix) AS percentile_90,
        STDDEV(prix) AS ecart_type
    FROM ventes
    GROUP BY produit_id
)
SELECT
    p.nom,
    s.mediane_prix,
    s.percentile_90,
    s.ecart_type,
    CASE
        WHEN v.prix > s.percentile_90 THEN 'Prix élevé'
        WHEN v.prix < s.mediane_prix THEN 'Prix bas'
        ELSE 'Prix normal'
    END AS categorie_prix
FROM ventes v  
INNER JOIN produits p ON v.produit_id = p.id  
INNER JOIN statistiques_complexes s ON v.produit_id = s.produit_id;  
```

---

## NOT MATERIALIZED : Quand et pourquoi l'utiliser ?

### Cas d'usage 1 : Permettre le push-down de filtres

`NOT MATERIALIZED` permet à PostgreSQL de "pousser" les filtres à l'intérieur de la CTE pour un meilleur plan d'exécution.

```sql
-- L'optimiseur peut pousser le WHERE dans la CTE
WITH tous_clients AS NOT MATERIALIZED (
    SELECT * FROM clients
)
SELECT nom, email  
FROM tous_clients  
WHERE ville = 'Paris';  -- Ce filtre peut être appliqué directement sur clients  
```

**Équivalent optimisé automatiquement :**

```sql
SELECT nom, email  
FROM clients  
WHERE ville = 'Paris';  
```

### Cas d'usage 2 : CTE simple utilisée une seule fois

Si la CTE n'est qu'un alias pour clarifier le code et n'est utilisée qu'une fois :

```sql
WITH ventes_2024 AS NOT MATERIALIZED (
    SELECT * FROM ventes WHERE annee = 2024
)
SELECT categorie, SUM(montant)  
FROM ventes_2024  
WHERE region = 'Europe'  -- Filtre additionnel  
GROUP BY categorie;  
```

L'optimiseur peut fusionner les conditions :

```sql
-- Plan optimisé automatiquement
SELECT categorie, SUM(montant)  
FROM ventes  
WHERE annee = 2024 AND region = 'Europe'  
GROUP BY categorie;  
```

### Cas d'usage 3 : Éviter la matérialisation de grandes tables

Si une CTE produit des millions de lignes mais que la requête principale ne va utiliser qu'une petite partie :

```sql
WITH toutes_transactions AS NOT MATERIALIZED (
    SELECT * FROM transactions  -- Table de 100M lignes
)
SELECT *  
FROM toutes_transactions  
WHERE id = 12345;  -- Ne récupère qu'1 ligne  
```

**Avec NOT MATERIALIZED :** PostgreSQL peut utiliser l'index sur `id` directement.

**Avec MATERIALIZED :** PostgreSQL chargerait les 100M lignes en mémoire, puis filtrerait.

---

## Comparaison : MATERIALIZED vs NOT MATERIALIZED

| Critère | MATERIALIZED | NOT MATERIALIZED |
|---------|--------------|------------------|
| **Exécution** | CTE exécutée 1 fois, résultat stocké | CTE fusionnée avec requête principale |
| **Mémoire** | Consomme de la mémoire temporaire | Pas de stockage intermédiaire |
| **Optimisation** | Plan figé pour la CTE | Optimiseur peut réorganiser globalement |
| **Utilisation multiple** | ⭐ Excellent (1 seule exécution) | Peut être recalculée plusieurs fois |
| **Filtres additionnels** | Appliqués après matérialisation | Peuvent être "poussés" dans la CTE |
| **Index** | Ne peuvent pas être utilisés sur le résultat | Les index de la table source sont utilisables |
| **Quand utiliser** | CTE coûteuse + réutilisée | CTE simple, clause WHERE externe, 1 utilisation |

---

## Exemples Comparatifs

### Exemple 1 : Filtrage après CTE

```sql
-- Scénario : Grande table, petit filtre final

-- ❌ MAUVAIS : Matérialise toute la table
WITH tous_clients AS MATERIALIZED (
    SELECT * FROM clients  -- 10M lignes
)
SELECT * FROM tous_clients WHERE id = 42;

-- ✅ BON : L'index sur id peut être utilisé
WITH tous_clients AS NOT MATERIALIZED (
    SELECT * FROM clients
)
SELECT * FROM tous_clients WHERE id = 42;
```

### Exemple 2 : Agrégation réutilisée

```sql
-- Scénario : Calcul coûteux utilisé plusieurs fois

-- ✅ BON : Calcule 1 fois
WITH stats AS MATERIALIZED (
    SELECT
        categorie,
        AVG(prix) AS prix_moyen,
        STDDEV(prix) AS ecart_type
    FROM produits
    GROUP BY categorie
)
SELECT
    (SELECT COUNT(*) FROM stats) AS nb_categories,
    (SELECT AVG(prix_moyen) FROM stats) AS moyenne_des_moyennes,
    (SELECT SUM(ecart_type) FROM stats) AS somme_ecarts_type;

-- ❌ MOINS BON : Peut recalculer 3 fois
WITH stats AS NOT MATERIALIZED (
    -- Même requête
)
-- Même SELECT...
```

### Exemple 3 : Pipeline de transformations

```sql
-- Scénario : Étapes séquentielles de nettoyage

WITH  
etape1 AS MATERIALIZED (  
    -- Première transformation lourde
    SELECT *, LOWER(TRIM(nom)) AS nom_clean
    FROM clients_brut
    WHERE date_creation > '2024-01-01'
),
etape2 AS MATERIALIZED (
    -- Enrichissement sur résultat de etape1
    SELECT
        *,
        CASE WHEN nom_clean LIKE '%corp%' THEN 'B2B' ELSE 'B2C' END AS type_client
    FROM etape1
)
SELECT type_client, COUNT(*)  
FROM etape2  
GROUP BY type_client;  
```

**Pourquoi MATERIALIZED ici ?**
- Chaque étape dépend de la précédente
- On veut éviter de recalculer les transformations
- Ordre d'exécution séquentiel souhaité

---

## Comment choisir : Arbre de décision

```
La CTE est-elle utilisée plusieurs fois ?
│
├─ OUI → MATERIALIZED (sauf si très légère)
│
└─ NON
    │
    ├─ La CTE est-elle très coûteuse (agrégations, calculs) ?
    │   │
    │   ├─ OUI → MATERIALIZED
    │   └─ NON → Continuer
    │
    └─ Y a-t-il des filtres dans la requête principale ?
        │
        ├─ OUI → NOT MATERIALIZED (pour push-down)
        └─ NON → Laisser PostgreSQL décider (défaut)
```

---

## Performance : Mesurer l'impact

### Utiliser EXPLAIN ANALYZE

```sql
-- Tester avec MATERIALIZED
EXPLAIN ANALYZE  
WITH ma_cte AS MATERIALIZED (  
    SELECT * FROM grande_table WHERE condition
)
SELECT * FROM ma_cte WHERE autre_condition;

-- Tester avec NOT MATERIALIZED
EXPLAIN ANALYZE  
WITH ma_cte AS NOT MATERIALIZED (  
    SELECT * FROM grande_table WHERE condition
)
SELECT * FROM ma_cte WHERE autre_condition;
```

### Interpréter les résultats

**Avec MATERIALIZED :**
```
CTE Scan on ma_cte  (cost=... rows=...)
  CTE ma_cte
    -> Seq Scan on grande_table  (cost=...)
```

**Avec NOT MATERIALIZED :**
```
Seq Scan on grande_table  (cost=...)
  Filter: (condition AND autre_condition)
```

**Observez :**
- Temps d'exécution total
- Nombre de lignes traitées
- Utilisation des index

---

## CTE vs Sous-requêtes : Quand utiliser quoi ?

### Quand préférer les CTE

1. **Requête complexe avec multiples étapes**
   ```sql
   WITH
       etape1 AS (...),
       etape2 AS (...),
       etape3 AS (...)
   SELECT * FROM etape3;
   ```

2. **Réutilisation d'un résultat**
   ```sql
   WITH calcul AS (...)
   SELECT * FROM calcul
   UNION ALL
   SELECT * FROM calcul WHERE ...;
   ```

3. **Lisibilité et maintenance**
   ```sql
   WITH
       clients_actifs AS (...),
       commandes_recentes AS (...)
   SELECT ...;
   ```

4. **Documentation du code**
   - Les noms de CTE documentent l'intention

### Quand préférer les sous-requêtes

1. **Sous-requête scalaire simple**
   ```sql
   SELECT nom, (SELECT AVG(salaire) FROM employes) AS moyenne
   FROM employes;
   ```

2. **EXISTS / NOT EXISTS**
   ```sql
   SELECT * FROM clients c
   WHERE EXISTS (SELECT 1 FROM commandes WHERE client_id = c.id);
   ```

3. **Sous-requête utilisée une seule fois dans FROM**
   ```sql
   -- Si très simple, sous-requête acceptable
   SELECT * FROM (SELECT * FROM t WHERE x > 10) AS sub;
   ```

---

## CTE vs Vues vs Tables Temporaires

| Caractéristique | CTE | Vue | Table Temporaire |
|-----------------|-----|-----|------------------|
| **Durée de vie** | Requête unique | Permanente | Session ou transaction |
| **Indexation** | Non | Possible sur tables sous-jacentes | Oui |
| **Réutilisation** | Dans la même requête | Entre requêtes | Entre requêtes |
| **Performance** | Optimisée par requête | Dépend des tables | Peut être optimisée |
| **Cas d'usage** | Décomposition logique | Abstraction permanente | Travail temporaire complexe |

---

## Bonnes Pratiques

### ✅ À faire

1. **Nommer clairement les CTE**
   ```sql
   -- ✅ BON
   WITH salaires_au_dessus_moyenne AS (...)

   -- ❌ ÉVITER
   WITH cte1 AS (...)
   ```

2. **Décomposer les requêtes complexes**
   ```sql
   WITH
       donnees_nettoyees AS (...),
       donnees_agregees AS (...),
       donnees_enrichies AS (...)
   SELECT * FROM donnees_enrichies;
   ```

3. **Tester chaque CTE individuellement**
   ```sql
   -- Étape 1 : Tester la CTE
   WITH ma_cte AS (...)
   SELECT * FROM ma_cte LIMIT 10;

   -- Étape 2 : Utiliser dans la requête finale
   ```

4. **Utiliser MATERIALIZED pour les réutilisations**
   ```sql
   WITH stats AS MATERIALIZED (
       -- Calcul coûteux
   )
   SELECT * FROM stats  -- Utilisé plusieurs fois
   UNION ALL
   SELECT * FROM stats WHERE ...;
   ```

5. **Commenter les CTE complexes**
   ```sql
   WITH
       -- Étape 1 : Filtrer les clients actifs (1M lignes → 100K lignes)
       clients_actifs AS (...),
       -- Étape 2 : Agréger par région
       stats_region AS (...)
   SELECT ...;
   ```

### ❌ À éviter

1. **CTE inutiles**
   ```sql
   -- ❌ INUTILE
   WITH tous AS (SELECT * FROM table)
   SELECT * FROM tous;

   -- ✅ DIRECT
   SELECT * FROM table;
   ```

2. **Trop de niveaux d'imbrication**
   ```sql
   -- ❌ Difficile à lire
   WITH a AS (...),
        b AS (SELECT * FROM a),
        c AS (SELECT * FROM b),
        d AS (SELECT * FROM c)
   SELECT * FROM d;
   ```

3. **Matérialiser des résultats énormes non réutilisés**
   ```sql
   -- ❌ Consomme beaucoup de mémoire inutilement
   WITH huge AS MATERIALIZED (
       SELECT * FROM table_100M_lignes
   )
   SELECT * FROM huge WHERE id = 1;  -- Utilise 1 ligne sur 100M
   ```

4. **Oublier que les CTE sont réévaluées dans les sous-requêtes**
   ```sql
   WITH ma_cte AS (...)
   SELECT
       x,
       (SELECT COUNT(*) FROM ma_cte) AS total  -- Réévaluation !
   FROM ma_cte;
   ```

---

## Cas d'Usage Avancés

### 1. Déduplication avec ROW_NUMBER

```sql
WITH deduplique AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY date_creation DESC) AS rn
    FROM clients
)
SELECT *  
FROM deduplique  
WHERE rn = 1;  
```

### 2. Calculs en cascade

```sql
WITH  
ventes_journalieres AS (  
    SELECT
        DATE(date_vente) AS jour,
        SUM(montant) AS total_jour
    FROM ventes
    GROUP BY DATE(date_vente)
),
ventes_cumulees AS (
    SELECT
        jour,
        total_jour,
        SUM(total_jour) OVER (ORDER BY jour) AS cumul
    FROM ventes_journalieres
)
SELECT
    jour,
    total_jour,
    cumul,
    cumul / (SELECT SUM(total_jour) FROM ventes_journalieres) * 100 AS pourcentage_cumul
FROM ventes_cumulees  
ORDER BY jour;  
```

### 3. Pivot de données

```sql
WITH ventes_par_mois AS (
    SELECT
        produit,
        EXTRACT(MONTH FROM date_vente) AS mois,
        SUM(montant) AS total
    FROM ventes
    WHERE EXTRACT(YEAR FROM date_vente) = 2024
    GROUP BY produit, EXTRACT(MONTH FROM date_vente)
)
SELECT
    produit,
    SUM(CASE WHEN mois = 1 THEN total ELSE 0 END) AS janvier,
    SUM(CASE WHEN mois = 2 THEN total ELSE 0 END) AS fevrier,
    SUM(CASE WHEN mois = 3 THEN total ELSE 0 END) AS mars
    -- ... autres mois
FROM ventes_par_mois  
GROUP BY produit;  
```

### 4. Comparaison avec période précédente

```sql
WITH  
ventes_courantes AS (  
    SELECT
        categorie,
        SUM(montant) AS total
    FROM ventes
    WHERE date_vente >= '2024-01-01' AND date_vente < '2024-02-01'
    GROUP BY categorie
),
ventes_precedentes AS (
    SELECT
        categorie,
        SUM(montant) AS total
    FROM ventes
    WHERE date_vente >= '2023-12-01' AND date_vente < '2024-01-01'
    GROUP BY categorie
)
SELECT
    COALESCE(c.categorie, p.categorie) AS categorie,
    c.total AS mois_courant,
    p.total AS mois_precedent,
    -- NULLIF(p.total, 0) protège contre la division par zéro :
    -- si une catégorie n'existait pas le mois précédent, evolution_pct = NULL
    ROUND((c.total - p.total) / NULLIF(p.total, 0) * 100, 2) AS evolution_pct
FROM ventes_courantes c  
FULL OUTER JOIN ventes_precedentes p ON c.categorie = p.categorie  
ORDER BY evolution_pct DESC NULLS LAST;  
```

---

## Limitations et Contraintes

### 1. Pas d'index sur les CTE matérialisées

Les CTE matérialisées sont des résultats temporaires en mémoire : impossible d'y créer des index.

**Workaround :** Utilisez des tables temporaires si vous avez besoin d'index.

```sql
-- Alternative avec table temporaire
CREATE TEMP TABLE stats_temp AS  
SELECT categorie, AVG(prix) AS prix_moyen  
FROM produits  
GROUP BY categorie;  

CREATE INDEX idx_cat ON stats_temp(categorie);

SELECT * FROM stats_temp WHERE categorie = 'Électronique';
```

### 2. CTE récursives toujours matérialisées

Les CTE récursives (voir chapitre 9.3) sont **toujours matérialisées implicitement** par PostgreSQL (elles ne peuvent pas être *inlinées*). Le mot-clé `MATERIALIZED` y est donc redondant — accepté syntaxiquement, sans effet.

### 3. Scope limité à la requête

Une CTE n'existe que dans la requête qui la définit.

```sql
WITH ma_cte AS (SELECT * FROM t1)  
SELECT * FROM ma_cte;  -- ✅ OK  

-- Nouvelle requête
SELECT * FROM ma_cte;  -- ❌ ERREUR : ma_cte n'existe plus
```

---

## CTE en écriture (data-modifying CTEs)

PostgreSQL est l'un des rares SGBD qui autorise des **`INSERT` / `UPDATE` / `DELETE` / `MERGE` directement dans une clause `WITH`**. Combinées à `RETURNING`, ces CTE permettent d'enchaîner plusieurs modifications atomiquement dans **une seule instruction**.

### Syntaxe

```sql
WITH operation_nommee AS (
    INSERT INTO … VALUES (…) RETURNING colonnes
    -- ou UPDATE … SET … RETURNING colonnes
    -- ou DELETE FROM … RETURNING colonnes
)
SELECT … FROM operation_nommee;
```

### Cas d'usage 1 — Archivage atomique (déplacer une ligne)

```sql
-- Déplacer les commandes anciennes vers une table d'archive,
-- en UNE seule instruction (atomique grâce à la transaction implicite)
WITH supprimees AS (
    DELETE FROM commandes
    WHERE date_commande < CURRENT_DATE - INTERVAL '5 years'
    RETURNING *
)
INSERT INTO commandes_archive
SELECT * FROM supprimees;
```

### Cas d'usage 2 — Audit en même temps qu'une mise à jour

🆕 **PG 18** introduit `OLD.*` / `NEW.*` dans `RETURNING`, ce qui rend l'audit avant/après trivial :

```sql
-- Augmenter les salaires + tracer l'opération dans un journal d'audit (PG 18+)
WITH augmentes AS (
    UPDATE employes
       SET salaire = salaire * 1.05
     WHERE evaluation = 'Excellent'
    RETURNING id, OLD.salaire AS ancien, NEW.salaire AS nouveau
)
INSERT INTO audit_salaires (employe_id, ancien, nouveau, date_modif)
SELECT id, ancien, nouveau, NOW()
FROM augmentes;
```

> 💡 **Avant PG 18**, on devait recalculer l'ancien salaire via la formule inverse (`salaire / 1.05`) — fragile car sujet aux arrondis flottants. La syntaxe `OLD.col` retourne la valeur *exacte* avant modification.

### Cas d'usage 3 — Upsert avec dispatch sur deux tables

```sql
-- Inserer les commandes ; si elles concernent un produit en rupture,
-- déclencher en plus un réapprovisionnement.
WITH nouvelles AS (
    INSERT INTO commandes (client_id, produit_id, quantite)
    VALUES (42, 17, 3), (42, 18, 1)
    RETURNING produit_id, quantite
),
ruptures AS (
    SELECT n.produit_id, n.quantite
    FROM nouvelles n
    JOIN produits p USING (produit_id)
    WHERE p.stock < n.quantite
)
INSERT INTO commandes_reappro (produit_id, quantite_a_commander)
SELECT produit_id, quantite * 10 FROM ruptures;
```

### Règles et pièges importants

1. **Ordre d'exécution non garanti** : toutes les CTE en écriture d'une même instruction voient le **même snapshot** des données (état avant l'instruction). Une CTE `UPDATE` ne « voit » donc pas les changements d'une autre CTE `DELETE` de la même `WITH`.

   ```sql
   -- ⚠️ Le DELETE et l'UPDATE travaillent sur le même snapshot
   WITH suppr AS (DELETE FROM t WHERE x = 1 RETURNING *),
        upd  AS (UPDATE t SET x = 2 WHERE x = 1 RETURNING *)
   SELECT * FROM suppr UNION ALL SELECT * FROM upd;
   -- Comportement non-déterministe : à éviter
   ```

2. **`RETURNING` est obligatoire** pour exploiter le résultat de la CTE plus loin dans la requête.

3. **Une CTE en écriture est toujours exécutée** (matérialisée), même si elle n'est référencée nulle part dans la requête principale — sa présence dans la `WITH` suffit à déclencher l'opération.

4. **Triggers `BEFORE`/`AFTER`** : se déclenchent normalement comme pour un `INSERT`/`UPDATE`/`DELETE` simple.

5. Sur PG 18 : `MERGE` est également utilisable dans une CTE, avec `RETURNING` (PG 17+) qui expose l'action (`INSERT`/`UPDATE`/`DELETE`) via `merge_action()`.

---

## Résumé

### Points clés à retenir

1. **CTE = Clarté** : Décompose des requêtes complexes en étapes nommées  
2. **Réutilisable** : Une CTE peut être référencée plusieurs fois  
3. **MATERIALIZED** : Calcule une fois et stocke (bon pour réutilisation)  
4. **NOT MATERIALIZED** : Fusionne avec requête principale (bon pour optimisation)  
5. **PostgreSQL 12+** : Décision automatique intelligente par défaut  
6. **Test avec EXPLAIN** : Toujours mesurer l'impact des choix

### Quand utiliser quoi ?

| Situation | Recommandation |
|-----------|----------------|
| CTE utilisée 2+ fois | `MATERIALIZED` |
| CTE simple, 1 utilisation, filtres externes | `NOT MATERIALIZED` |
| Pipeline de transformations complexes | `MATERIALIZED` |
| CTE pour lisibilité seulement | Laisser défaut ou `NOT MATERIALIZED` |
| Résultat énorme, utilisation partielle | `NOT MATERIALIZED` |
| Calculs coûteux (agrégats, fonctions fenêtre) | `MATERIALIZED` si réutilisé |

### Checklist d'utilisation

- [ ] La CTE améliore-t-elle vraiment la lisibilité ?  
- [ ] Ai-je testé la CTE indépendamment ?  
- [ ] Ai-je donné un nom descriptif ?  
- [ ] Ai-je vérifié avec EXPLAIN ANALYZE les performances ?  
- [ ] Ai-je choisi MATERIALIZED ou NOT MATERIALIZED de façon éclairée ?

---

## Pour aller plus loin

- **CTE Récursives** (Chapitre 9.3) : Parcourir hiérarchies et graphes  
- **Window Functions** (Chapitre 10) : Souvent combinées avec CTE  
- **Vues Matérialisées** (Chapitre 11.5) : Pour persister des résultats entre requêtes  
- **Query Optimization** (Chapitre 13) : Approfondir l'analyse de performance

---

**Prochain chapitre :** 9.3. CTE Récursives : Parcourir des hiérarchies (arbres, graphes)

---


⏭️ [CTE Récursives : Parcourir des hiérarchies (arbres, graphes)](/09-techniques-sql-avancees/03-cte-recursives.md)
