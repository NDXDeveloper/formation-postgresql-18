🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.4. Extensions de Groupement (ROLLUP, CUBE, GROUPING SETS)

## Introduction : Le Besoin de Sous-Totaux

Dans la section précédente, nous avons appris à utiliser GROUP BY pour calculer des agrégations par groupe. Mais souvent, nous avons besoin de **plusieurs niveaux d'agrégation simultanément** :

- Ventes par produit **et par région**
- Ventes par produit (tous régions confondues)
- Ventes par région (tous produits confondus)
- Ventes totales (grand total)

Sans les extensions de groupement, vous devriez écrire plusieurs requêtes distinctes et les combiner avec UNION ALL. C'est fastidieux, verbeux, et peu performant.

**Les extensions de groupement** (ROLLUP, CUBE, GROUPING SETS) permettent de générer **plusieurs niveaux d'agrégation en une seule requête** !

---

## Le Problème : Rapport Multi-Niveaux

### Exemple de Besoin Métier

Imaginons une table `ventes` :

| id_vente | region   | categorie      | montant |
|----------|----------|----------------|---------|
| 1        | Nord     | Électronique   | 1200    |
| 2        | Nord     | Vêtements      | 350     |
| 3        | Sud      | Électronique   | 800     |
| 4        | Sud      | Vêtements      | 420     |
| 5        | Est      | Électronique   | 950     |
| 6        | Est      | Vêtements      | 280     |

**Besoin** : Un rapport avec :
1. Ventes par région ET catégorie (détail)  
2. Sous-totaux par région (toutes catégories)  
3. Sous-totaux par catégorie (toutes régions)  
4. Grand total général

### Solution Classique (Sans Extensions) : Fastidieuse !

```sql
-- Niveau 1 : Détail région + catégorie
SELECT region, categorie, SUM(montant) AS total  
FROM ventes  
GROUP BY region, categorie  

UNION ALL

-- Niveau 2 : Sous-totaux par région
SELECT region, NULL AS categorie, SUM(montant) AS total  
FROM ventes  
GROUP BY region  

UNION ALL

-- Niveau 3 : Sous-totaux par catégorie
SELECT NULL AS region, categorie, SUM(montant) AS total  
FROM ventes  
GROUP BY categorie  

UNION ALL

-- Niveau 4 : Grand total
SELECT NULL AS region, NULL AS categorie, SUM(montant) AS total  
FROM ventes;  
```

**Problèmes** :
- ❌ Requête longue et répétitive  
- ❌ PostgreSQL scanne la table **4 fois** (très inefficace)  
- ❌ Difficile à maintenir

### Solution Moderne : Les Extensions de Groupement

Les extensions permettent de générer ces niveaux en **une seule passe** !

---

## 1. ROLLUP : Sous-Totaux Hiérarchiques

### Qu'est-ce que ROLLUP ?

**ROLLUP** génère des sous-totaux en suivant une **hiérarchie de gauche à droite** des colonnes de groupement.

### Syntaxe

```sql
SELECT
    colonne1,
    colonne2,
    colonne3,
    SUM(valeur) AS total
FROM table  
GROUP BY ROLLUP(colonne1, colonne2, colonne3);  
```

### Comment ça Fonctionne ?

ROLLUP génère automatiquement les groupements suivants (de gauche à droite) :

1. `GROUP BY colonne1, colonne2, colonne3` (détail complet)  
2. `GROUP BY colonne1, colonne2` (sous-total niveau 2)  
3. `GROUP BY colonne1` (sous-total niveau 1)  
4. Grand total (aucun groupement)

**Analogie** : Imaginez un organigramme d'entreprise :
- Niveau 1 : Employé individuel
- Niveau 2 : Total par équipe
- Niveau 3 : Total par département
- Niveau 4 : Total entreprise

### Exemple Concret : Ventes par Région et Catégorie

```sql
SELECT
    region,
    categorie,
    SUM(montant) AS total_ventes
FROM ventes  
GROUP BY ROLLUP(region, categorie)  
ORDER BY region NULLS LAST, categorie NULLS LAST;  
```

**Résultat :**

| region | categorie      | total_ventes |
|--------|----------------|--------------|
| Est    | Électronique   | 950          |
| Est    | Vêtements      | 280          |
| Est    | NULL           | 1230         | ← Sous-total Est
| Nord   | Électronique   | 1200         |
| Nord   | Vêtements      | 350          |
| Nord   | NULL           | 1550         | ← Sous-total Nord
| Sud    | Électronique   | 800          |
| Sud    | Vêtements      | 420          |
| Sud    | NULL           | 1220         | ← Sous-total Sud
| NULL   | NULL           | 4000         | ← Grand total

**Interprétation :**
- Lignes avec `categorie = NULL` (mais `region ≠ NULL`) : Sous-totaux par région
- Ligne avec `region = NULL` ET `categorie = NULL` : Grand total général

### Niveaux Générés par ROLLUP

Pour `ROLLUP(A, B, C)`, PostgreSQL génère :

```
GROUP BY A, B, C     -- Détail complet  
GROUP BY A, B        -- Sous-total niveau 1  
GROUP BY A           -- Sous-total niveau 2  
Grand total          -- Total général (aucun GROUP BY)  
```

**Nombre de niveaux** : N+1 (où N = nombre de colonnes dans ROLLUP)

### Cas d'Usage Typiques de ROLLUP

**1. Hiérarchies Temporelles**

```sql
-- Ventes par année, mois, jour + sous-totaux
SELECT
    EXTRACT(YEAR FROM date_vente) AS annee,
    EXTRACT(MONTH FROM date_vente) AS mois,
    EXTRACT(DAY FROM date_vente) AS jour,
    SUM(montant) AS ca
FROM ventes  
GROUP BY ROLLUP(  
    EXTRACT(YEAR FROM date_vente),
    EXTRACT(MONTH FROM date_vente),
    EXTRACT(DAY FROM date_vente)
)
ORDER BY annee, mois, jour;
```

**Génère :**
- Ventes par jour
- Sous-totaux par mois
- Sous-totaux par année
- Grand total

**2. Hiérarchies Géographiques**

```sql
-- Ventes par pays, région, ville + sous-totaux
SELECT
    pays,
    region,
    ville,
    SUM(montant) AS ca
FROM ventes  
GROUP BY ROLLUP(pays, region, ville);  
```

**Génère :**
- Détail par ville
- Sous-totaux par région
- Sous-totaux par pays
- Total mondial

**3. Hiérarchies Organisationnelles**

```sql
-- Salaires par département, équipe, poste
SELECT
    departement,
    equipe,
    poste,
    AVG(salaire) AS salaire_moyen,
    COUNT(*) AS nb_employes
FROM employes  
GROUP BY ROLLUP(departement, equipe, poste);  
```

---

## 2. CUBE : Toutes les Combinaisons Possibles

### Qu'est-ce que CUBE ?

**CUBE** génère **toutes les combinaisons possibles** de sous-totaux pour les colonnes spécifiées.

### Différence avec ROLLUP

- **ROLLUP** : Hiérarchique (gauche → droite)  
- **CUBE** : Exhaustif (toutes les combinaisons)

### Syntaxe

```sql
SELECT
    colonne1,
    colonne2,
    SUM(valeur) AS total
FROM table  
GROUP BY CUBE(colonne1, colonne2);  
```

### Comment ça Fonctionne ?

Pour `CUBE(A, B)`, PostgreSQL génère **2² = 4 groupements** :

1. `GROUP BY A, B` (détail complet)  
2. `GROUP BY A` (sous-total sur A uniquement)  
3. `GROUP BY B` (sous-total sur B uniquement)  
4. Grand total (aucun groupement)

Pour `CUBE(A, B, C)` : **2³ = 8 groupements**

**Formule générale** : 2^N groupements (où N = nombre de colonnes)

### Exemple Concret : Ventes par Région et Catégorie

```sql
SELECT
    region,
    categorie,
    SUM(montant) AS total_ventes
FROM ventes  
GROUP BY CUBE(region, categorie)  
ORDER BY region NULLS LAST, categorie NULLS LAST;  
```

**Résultat :**

| region | categorie      | total_ventes |
|--------|----------------|--------------|
| Est    | Électronique   | 950          | ← Détail
| Est    | Vêtements      | 280          | ← Détail
| Est    | NULL           | 1230         | ← Sous-total Est (toutes catégories)
| Nord   | Électronique   | 1200         | ← Détail
| Nord   | Vêtements      | 350          | ← Détail
| Nord   | NULL           | 1550         | ← Sous-total Nord
| Sud    | Électronique   | 800          | ← Détail
| Sud    | Vêtements      | 420          | ← Détail
| Sud    | NULL           | 1220         | ← Sous-total Sud
| NULL   | Électronique   | 2950         | ← Sous-total Électronique (toutes régions) 🆕
| NULL   | Vêtements      | 1050         | ← Sous-total Vêtements (toutes régions) 🆕
| NULL   | NULL           | 4000         | ← Grand total

**Différence avec ROLLUP** :
- CUBE ajoute les lignes avec `region = NULL` mais `categorie ≠ NULL`
- Ce sont des **sous-totaux par catégorie** (toutes régions confondues)

### Comparaison Visuelle : ROLLUP vs CUBE

**Avec ROLLUP(region, categorie)** :
```
✅ (region, categorie)  -- Détail
✅ (region, NULL)       -- Sous-total par région
✅ (NULL, NULL)         -- Grand total
❌ (NULL, categorie)    -- ABSENT
```

**Avec CUBE(region, categorie)** :
```
✅ (region, categorie)  -- Détail
✅ (region, NULL)       -- Sous-total par région
✅ (NULL, categorie)    -- Sous-total par catégorie 🆕
✅ (NULL, NULL)         -- Grand total
```

### Niveaux Générés par CUBE

Pour `CUBE(A, B, C)` :

```
GROUP BY A, B, C     -- Détail complet  
GROUP BY A, B        -- Sous-total AB  
GROUP BY A, C        -- Sous-total AC  
GROUP BY B, C        -- Sous-total BC  
GROUP BY A           -- Sous-total A  
GROUP BY B           -- Sous-total B  
GROUP BY C           -- Sous-total C  
Grand total          -- Total général  
```

**Total** : 2³ = 8 groupements

### Cas d'Usage Typiques de CUBE

**1. Analyse Croisée de Ventes**

```sql
-- Ventes croisées : produit × région
SELECT
    produit,
    region,
    SUM(montant) AS ca,
    COUNT(*) AS nb_ventes
FROM ventes  
GROUP BY CUBE(produit, region);  
```

**Génère :**
- Ventes par produit et région (détail)
- Total par produit (toutes régions)
- Total par région (tous produits)
- Grand total

**2. Tableau de Bord Multi-Dimensionnel**

```sql
-- Performance par vendeur et trimestre
SELECT
    vendeur_id,
    EXTRACT(QUARTER FROM date_vente) AS trimestre,
    SUM(montant) AS ca,
    COUNT(DISTINCT client_id) AS nb_clients
FROM ventes  
WHERE EXTRACT(YEAR FROM date_vente) = 2024  
GROUP BY CUBE(vendeur_id, EXTRACT(QUARTER FROM date_vente));  
```

**Permet de répondre à :**
- CA par vendeur et trimestre
- CA total par vendeur (tous trimestres)
- CA total par trimestre (tous vendeurs)
- CA annuel total

**3. Analyse A/B Testing**

```sql
-- Conversion par variante et segment utilisateur
SELECT
    variante_test,
    segment_utilisateur,
    COUNT(*) AS nb_utilisateurs,
    SUM(CASE WHEN a_converti THEN 1 ELSE 0 END) AS nb_conversions,
    ROUND(
        100.0 * SUM(CASE WHEN a_converti THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS taux_conversion_pct
FROM tests_ab  
GROUP BY CUBE(variante_test, segment_utilisateur);  
```

---

## 3. GROUPING SETS : Contrôle Total

### Qu'est-ce que GROUPING SETS ?

**GROUPING SETS** vous permet de spécifier **exactement quels groupements** vous voulez, sans générer tous les sous-totaux automatiquement.

C'est la version **"à la carte"** des extensions de groupement.

### Syntaxe

```sql
SELECT
    colonne1,
    colonne2,
    SUM(valeur) AS total
FROM table  
GROUP BY GROUPING SETS (  
    (colonne1, colonne2),  -- Groupement 1
    (colonne1),            -- Groupement 2
    ()                     -- Grand total
);
```

### Comment ça Fonctionne ?

Vous **listez explicitement** chaque combinaison de groupement souhaitée.

### Exemple : Équivalence avec ROLLUP

Ces deux requêtes sont **identiques** :

```sql
-- Avec ROLLUP
SELECT region, categorie, SUM(montant) AS total  
FROM ventes  
GROUP BY ROLLUP(region, categorie);  

-- Équivalent avec GROUPING SETS
SELECT region, categorie, SUM(montant) AS total  
FROM ventes  
GROUP BY GROUPING SETS (  
    (region, categorie),  -- Détail
    (region),             -- Sous-total région
    ()                    -- Grand total
);
```

### Exemple : Équivalence avec CUBE

```sql
-- Avec CUBE
SELECT region, categorie, SUM(montant) AS total  
FROM ventes  
GROUP BY CUBE(region, categorie);  

-- Équivalent avec GROUPING SETS
SELECT region, categorie, SUM(montant) AS total  
FROM ventes  
GROUP BY GROUPING SETS (  
    (region, categorie),  -- Détail
    (region),             -- Sous-total région
    (categorie),          -- Sous-total catégorie
    ()                    -- Grand total
);
```

### Pourquoi Utiliser GROUPING SETS ?

**Flexibilité et Performance** : Vous ne générez que les groupements nécessaires.

**Exemple : Groupements Asymétriques**

```sql
-- Je veux uniquement :
-- - Détail (region, categorie)
-- - Sous-total par catégorie (MAIS PAS par région)
-- - Grand total

SELECT
    region,
    categorie,
    SUM(montant) AS total
FROM ventes  
GROUP BY GROUPING SETS (  
    (region, categorie),  -- Détail
    (categorie),          -- Sous-total catégorie uniquement
    ()                    -- Grand total
);
-- Pas de sous-total par région !
```

**Impossible avec ROLLUP ou CUBE seuls** : ROLLUP génèrerait les sous-totaux par région, CUBE aussi.

### Cas d'Usage Typiques de GROUPING SETS

**1. Groupements Personnalisés**

```sql
-- Rapport spécifique : ventes par (produit, mois) + totaux par produit
SELECT
    produit,
    EXTRACT(MONTH FROM date_vente) AS mois,
    SUM(montant) AS ca
FROM ventes  
GROUP BY GROUPING SETS (  
    (produit, EXTRACT(MONTH FROM date_vente)),
    (produit)
);
-- Pas de totaux par mois ni de grand total
```

**2. Combinaison de Hiérarchies Différentes**

```sql
-- Ventes par (année, mois) + ventes par (région)
SELECT
    EXTRACT(YEAR FROM date_vente) AS annee,
    EXTRACT(MONTH FROM date_vente) AS mois,
    region,
    SUM(montant) AS ca
FROM ventes  
GROUP BY GROUPING SETS (  
    (EXTRACT(YEAR FROM date_vente), EXTRACT(MONTH FROM date_vente)),
    (region)
);
-- Deux hiérarchies indépendantes
```

**3. Optimisation de Requêtes Complexes**

Au lieu de plusieurs UNION ALL, une seule requête avec GROUPING SETS :

```sql
-- Besoin : totaux par produit, totaux par région, grand total
-- (mais PAS de détail produit × région)

SELECT
    produit,
    region,
    SUM(montant) AS ca
FROM ventes  
GROUP BY GROUPING SETS (  
    (produit),
    (region),
    ()
);
-- Seulement 3 groupements au lieu de 4 avec CUBE
```

---

## 4. La Fonction GROUPING() : Identifier les Sous-Totaux

### Le Problème : NULL Ambigus

Regardez ce résultat :

| region | categorie | total_ventes |
|--------|-----------|--------------|
| Nord   | NULL      | 1550         |

**Question** : Ce NULL signifie-t-il :
- A) Sous-total pour la région Nord (toutes catégories) ?
- B) Ventes dans la région Nord pour des produits sans catégorie ?

**Réponse** : Impossible à dire sans contexte !

### La Solution : GROUPING()

La fonction **GROUPING()** retourne :
- **1** si la colonne est NULL **à cause d'un sous-total** (agrégation)  
- **0** si la colonne a sa valeur réelle (ou est NULL dans les données)

### Syntaxe

```sql
SELECT
    region,
    categorie,
    SUM(montant) AS total,
    GROUPING(region) AS est_subtotal_region,
    GROUPING(categorie) AS est_subtotal_categorie
FROM ventes  
GROUP BY ROLLUP(region, categorie);  
```

**Résultat :**

| region | categorie    | total | est_subtotal_region | est_subtotal_categorie |
|--------|--------------|-------|---------------------|------------------------|
| Est    | Électronique | 950   | 0                   | 0                      |
| Est    | Vêtements    | 280   | 0                   | 0                      |
| Est    | NULL         | 1230  | 0                   | 1                      | ← Sous-total catégorie
| Nord   | Électronique | 1200  | 0                   | 0                      |
| Nord   | Vêtements    | 350   | 0                   | 0                      |
| Nord   | NULL         | 1550  | 0                   | 1                      | ← Sous-total catégorie
| Sud    | Électronique | 800   | 0                   | 0                      |
| Sud    | Vêtements    | 420   | 0                   | 0                      |
| Sud    | NULL         | 1220  | 0                   | 1                      | ← Sous-total catégorie
| NULL   | NULL         | 4000  | 1                   | 1                      | ← Grand total

### Utilisation Pratique : Étiqueter les Lignes

```sql
SELECT
    CASE
        WHEN GROUPING(region) = 1 AND GROUPING(categorie) = 1
            THEN 'TOTAL GÉNÉRAL'
        WHEN GROUPING(categorie) = 1
            THEN 'Sous-total ' || region
        ELSE region
    END AS region_label,
    CASE
        WHEN GROUPING(categorie) = 1
            THEN 'Toutes catégories'
        ELSE categorie
    END AS categorie_label,
    SUM(montant) AS total_ventes
FROM ventes  
GROUP BY ROLLUP(region, categorie)  
ORDER BY  
    GROUPING(region),
    region NULLS LAST,
    GROUPING(categorie),
    categorie NULLS LAST;
```

**Résultat plus lisible :**

| region_label     | categorie_label    | total_ventes |
|------------------|--------------------|--------------|
| Est              | Électronique       | 950          |
| Est              | Vêtements          | 280          |
| Sous-total Est   | Toutes catégories  | 1230         |
| Nord             | Électronique       | 1200         |
| Nord             | Vêtements          | 350          |
| Sous-total Nord  | Toutes catégories  | 1550         |
| Sud              | Électronique       | 800          |
| Sud              | Vêtements          | 420          |
| Sous-total Sud   | Toutes catégories  | 1220         |
| TOTAL GÉNÉRAL    | Toutes catégories  | 4000         |

### GROUPING() avec Plusieurs Colonnes

```sql
-- Identifier le type de ligne selon le niveau d'agrégation
SELECT
    region,
    categorie,
    SUM(montant) AS total,
    CASE
        WHEN GROUPING(region) = 0 AND GROUPING(categorie) = 0 THEN 'Détail'
        WHEN GROUPING(region) = 0 AND GROUPING(categorie) = 1 THEN 'Sous-total région'
        WHEN GROUPING(region) = 1 AND GROUPING(categorie) = 0 THEN 'Sous-total catégorie'
        WHEN GROUPING(region) = 1 AND GROUPING(categorie) = 1 THEN 'Grand total'
    END AS niveau_agregation
FROM ventes  
GROUP BY CUBE(region, categorie);  
```

---

## Exemples Avancés par Domaine

### 📊 E-commerce : Analyse Multi-Dimensionnelle

```sql
-- Rapport de ventes : produit × région × mois
SELECT
    COALESCE(produit, 'TOUS PRODUITS') AS produit,
    COALESCE(region, 'TOUTES RÉGIONS') AS region,
    COALESCE(TO_CHAR(DATE_TRUNC('month', date_vente), 'YYYY-MM'), 'TOUS MOIS') AS mois,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca,
    ROUND(AVG(montant), 2) AS panier_moyen
FROM ventes  
WHERE date_vente >= '2024-01-01'  
GROUP BY CUBE(  
    produit,
    region,
    DATE_TRUNC('month', date_vente)
)
ORDER BY
    GROUPING(produit),
    produit NULLS LAST,
    GROUPING(region),
    region NULLS LAST,
    GROUPING(DATE_TRUNC('month', date_vente)),
    mois NULLS LAST;
```

**Génère 8 niveaux d'analyse** (2³) pour répondre à toutes les questions :
- Détail complet
- Totaux par produit et région
- Totaux par produit et mois
- Totaux par région et mois
- Totaux par produit
- Totaux par région
- Totaux par mois
- Grand total

### 🏥 Santé : Statistiques Hospitalières

```sql
-- Admissions par service, gravité, période
SELECT
    CASE
        WHEN GROUPING(service) = 1 THEN 'TOUS SERVICES'
        ELSE service
    END AS service,
    CASE
        WHEN GROUPING(niveau_gravite) = 1 THEN 'TOUTES GRAVITÉS'
        ELSE niveau_gravite
    END AS gravite,
    CASE
        WHEN GROUPING(EXTRACT(QUARTER FROM date_admission)) = 1 THEN 'TOUTE ANNÉE'
        ELSE 'T' || EXTRACT(QUARTER FROM date_admission)::TEXT
    END AS trimestre,
    COUNT(*) AS nb_admissions,
    ROUND(AVG(duree_sejour_jours), 1) AS duree_moyenne_sejour,
    GROUPING(service) AS sub_service,
    GROUPING(niveau_gravite) AS sub_gravite,
    GROUPING(EXTRACT(QUARTER FROM date_admission)) AS sub_trimestre
FROM admissions  
WHERE EXTRACT(YEAR FROM date_admission) = 2024  
GROUP BY ROLLUP(  
    service,
    niveau_gravite,
    EXTRACT(QUARTER FROM date_admission)
)
ORDER BY
    sub_service,
    service NULLS LAST,
    sub_gravite,
    gravite NULLS LAST,
    sub_trimestre,
    trimestre NULLS LAST;
```

### 💰 Finance : Analyse de Portefeuille

```sql
-- Performance par classe d'actifs et secteur
SELECT
    COALESCE(classe_actif, 'TOUTES CLASSES') AS classe_actif,
    COALESCE(secteur, 'TOUS SECTEURS') AS secteur,
    COUNT(DISTINCT ticker) AS nb_actifs,
    SUM(valeur_position) AS valeur_totale,
    ROUND(AVG(rendement_ytd_pct), 2) AS rendement_moyen_pct,
    ROUND(STDDEV(rendement_ytd_pct), 2) AS volatilite_pct
FROM portefeuille  
GROUP BY GROUPING SETS (  
    (classe_actif, secteur),  -- Détail complet
    (classe_actif),           -- Par classe d'actifs
    ()                        -- Portefeuille global
)
-- Exclut volontairement les totaux par secteur seul
ORDER BY
    GROUPING(classe_actif),
    classe_actif NULLS LAST,
    GROUPING(secteur),
    secteur NULLS LAST;
```

### 🌐 SaaS : Métriques d'Utilisation

```sql
-- Usage API par client, endpoint, période
WITH usage_periode AS (
    SELECT
        client_id,
        endpoint,
        DATE_TRUNC('week', timestamp) AS semaine,
        COUNT(*) AS nb_requetes,
        AVG(latence_ms) AS latence_moyenne,
        SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) AS nb_erreurs
    FROM logs_api
    WHERE timestamp >= NOW() - INTERVAL '4 weeks'
    GROUP BY client_id, endpoint, DATE_TRUNC('week', timestamp)
)
SELECT
    COALESCE(client_id::TEXT, 'TOUS CLIENTS') AS client_id,
    COALESCE(endpoint, 'TOUS ENDPOINTS') AS endpoint,
    COALESCE(TO_CHAR(semaine, 'YYYY-WW'), 'TOUTES SEMAINES') AS semaine,
    SUM(nb_requetes) AS total_requetes,
    ROUND(AVG(latence_moyenne), 2) AS latence_moy_ms,
    SUM(nb_erreurs) AS total_erreurs,
    ROUND(100.0 * SUM(nb_erreurs) / NULLIF(SUM(nb_requetes), 0), 2) AS taux_erreur_pct
FROM usage_periode  
GROUP BY CUBE(client_id, endpoint, semaine)  
HAVING SUM(nb_requetes) > 0  -- Exclure les groupes vides  
ORDER BY  
    GROUPING(client_id),
    client_id NULLS LAST,
    GROUPING(endpoint),
    endpoint NULLS LAST,
    GROUPING(semaine),
    semaine NULLS LAST;
```

---

## Comparaison des Trois Extensions

| Aspect | ROLLUP | CUBE | GROUPING SETS |
|--------|--------|------|---------------|
| **Type de sous-totaux** | Hiérarchiques | Toutes combinaisons | Personnalisés |
| **Nombre de groupements** | N+1 | 2^N | Selon spécification |
| **Performance** | ⚡ Rapide | 🐢 Plus lent (+ de groupes) | ⚡ Optimal (seulement ce qui est nécessaire) |
| **Flexibilité** | ❌ Limitée | ⚠️ Moyenne | ✅ Totale |
| **Complexité SQL** | ✅ Simple | ✅ Simple | ⚠️ Plus verbeux |
| **Usage typique** | Hiérarchies naturelles | Analyses croisées | Besoins spécifiques |
| **Exemple** | Année → Mois → Jour | Toutes combinaisons | Sur-mesure |

### Tableau de Décision

**Utilisez ROLLUP si :**
- ✅ Vos dimensions ont une hiérarchie naturelle (date, géographie, organisation)  
- ✅ Vous voulez des sous-totaux de gauche à droite  
- ✅ Exemple : Pays → Région → Ville

**Utilisez CUBE si :**
- ✅ Vous voulez analyser toutes les relations possibles  
- ✅ Vos dimensions sont indépendantes (produit, région, période)  
- ✅ Vous créez un tableau de bord complet  
- ⚠️ Attention : Peut générer beaucoup de lignes !

**Utilisez GROUPING SETS si :**
- ✅ Vous avez besoin de combinaisons spécifiques uniquement  
- ✅ Vous optimisez les performances (éviter calculs inutiles)  
- ✅ ROLLUP ou CUBE généreraient trop de données  
- ✅ Vous remplacez plusieurs UNION ALL

---

## Combinaison d'Extensions

Vous pouvez **combiner** plusieurs extensions dans la même requête !

### Exemple : ROLLUP Partiel + Groupement Simple

```sql
-- Hiérarchie temporelle (ROLLUP) × Catégorie (groupement simple)
SELECT
    EXTRACT(YEAR FROM date_vente) AS annee,
    EXTRACT(MONTH FROM date_vente) AS mois,
    categorie,
    SUM(montant) AS ca
FROM ventes  
GROUP BY  
    ROLLUP(EXTRACT(YEAR FROM date_vente), EXTRACT(MONTH FROM date_vente)),
    categorie;
```

**Génère :**
- Détail : (année, mois, catégorie)
- Sous-total : (année, catégorie) - pour chaque année, toutes catégories
- Total : (catégorie) - pour toutes années, toutes catégories

### Exemple : Plusieurs GROUPING SETS

```sql
-- Ventes par (produit, région) + par (client, mois)
SELECT
    produit,
    region,
    client_id,
    EXTRACT(MONTH FROM date_vente) AS mois,
    SUM(montant) AS ca
FROM ventes  
GROUP BY GROUPING SETS (  
    (produit, region),
    (client_id, EXTRACT(MONTH FROM date_vente))
);
-- Deux axes d'analyse indépendants
```

---

## Performance et Optimisation

### Coût de Calcul

Les extensions de groupement sont **plus efficaces** que plusieurs UNION ALL :

**Approche Naïve (4 scans de table) :**
```sql
SELECT region, SUM(montant) FROM ventes GROUP BY region  
UNION ALL  
SELECT categorie, SUM(montant) FROM ventes GROUP BY categorie  
UNION ALL  
SELECT NULL, SUM(montant) FROM ventes;  
-- PostgreSQL scanne la table 3 fois !
```

**Approche Optimisée (1 scan) :**
```sql
SELECT region, categorie, SUM(montant)  
FROM ventes  
GROUP BY GROUPING SETS ((region), (categorie), ());  
-- PostgreSQL scanne la table UNE SEULE fois !
```

### Plan d'Exécution

```sql
EXPLAIN ANALYZE  
SELECT region, categorie, SUM(montant)  
FROM ventes  
GROUP BY ROLLUP(region, categorie);  
```

**Résultat typique :**
```
MixedAggregate  (cost=25.50..30.75 rows=10 width=20)
  Group Key: region, categorie
  Group Key: region
  Group Key: ()
  ->  Sort  (cost=25.50..27.00 rows=600 width=12)
        Sort Key: region, categorie
        ->  Seq Scan on ventes  (cost=0.00..10.00 rows=600 width=12)
```

**MixedAggregate** : PostgreSQL calcule tous les niveaux en une seule passe !

### Optimisations

1. **Index sur les colonnes de groupement**
   ```sql
   CREATE INDEX idx_ventes_region_categorie
   ON ventes(region, categorie);
   ```

2. **Limiter le nombre de dimensions dans CUBE**
   - CUBE(A, B, C, D) génère 2⁴ = 16 groupements
   - Préférer GROUPING SETS avec seulement les combinaisons nécessaires

3. **Filtrer avec WHERE avant groupement**
   ```sql
   SELECT region, categorie, SUM(montant)
   FROM ventes
   WHERE date_vente >= '2024-01-01'  -- Réduit le volume
   GROUP BY CUBE(region, categorie);
   ```

4. **Matérialiser pour les rapports fréquents**
   ```sql
   CREATE MATERIALIZED VIEW ventes_cube AS
   SELECT
       region,
       categorie,
       DATE_TRUNC('month', date_vente) AS mois,
       SUM(montant) AS ca,
       COUNT(*) AS nb_ventes
   FROM ventes
   GROUP BY CUBE(region, categorie, DATE_TRUNC('month', date_vente));

   CREATE INDEX idx_ventes_cube_region ON ventes_cube(region);
   CREATE INDEX idx_ventes_cube_categorie ON ventes_cube(categorie);
   ```

---

## Formatage et Présentation des Résultats

### Remplacer les NULL par des Labels

```sql
SELECT
    COALESCE(region, '*** TOUTES RÉGIONS ***') AS region,
    COALESCE(categorie, '*** TOUTES CATÉGORIES ***') AS categorie,
    TO_CHAR(SUM(montant), 'FM999,999.00€') AS total_formate
FROM ventes  
GROUP BY ROLLUP(region, categorie)  
ORDER BY  
    region NULLS LAST,
    categorie NULLS LAST;
```

### Indenter les Sous-Totaux

```sql
SELECT
    CASE
        WHEN GROUPING(region) = 1 THEN 'TOTAL GÉNÉRAL'
        WHEN GROUPING(categorie) = 1 THEN '  Sous-total ' || region
        ELSE '    ' || region || ' - ' || categorie
    END AS libelle,
    SUM(montant) AS montant
FROM ventes  
GROUP BY ROLLUP(region, categorie);  
```

**Résultat :**
```
    Est - Électronique       | 950
    Est - Vêtements          | 280
  Sous-total Est             | 1230
    Nord - Électronique      | 1200
    Nord - Vêtements         | 350
  Sous-total Nord            | 1550
    Sud - Électronique       | 800
    Sud - Vêtements          | 420
  Sous-total Sud             | 1220
TOTAL GÉNÉRAL                | 4000
```

### Ajouter des Indicateurs Visuels

```sql
SELECT
    COALESCE(region, '🌍 MONDIAL') AS region,
    COALESCE(categorie, '📦 TOUTES') AS categorie,
    SUM(montant) AS ca,
    CASE
        WHEN GROUPING(region) = 1 AND GROUPING(categorie) = 1 THEN '🏆 TOTAL'
        WHEN GROUPING(categorie) = 1 THEN '📊 Sous-total'
        ELSE '📍 Détail'
    END AS type_ligne
FROM ventes  
GROUP BY ROLLUP(region, categorie);  
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Utiliser GROUPING() pour identifier les niveaux**
   ```sql
   SELECT
       region,
       SUM(montant) AS ca,
       GROUPING(region) AS est_total
   FROM ventes
   GROUP BY ROLLUP(region);
   ```

2. **Ajouter des labels explicites avec COALESCE ou CASE**
   ```sql
   SELECT
       COALESCE(region, 'TOUTES RÉGIONS') AS region,
       SUM(montant) AS ca
   FROM ventes
   GROUP BY ROLLUP(region);
   ```

3. **Utiliser GROUPING SETS pour les besoins spécifiques**
   - Plus efficace que CUBE si vous ne voulez pas toutes les combinaisons

4. **Ordonner avec GROUPING() pour grouper les sous-totaux**
   ```sql
   ORDER BY
       GROUPING(region),
       region NULLS LAST,
       GROUPING(categorie),
       categorie NULLS LAST;
   ```

5. **Limiter le nombre de dimensions dans CUBE**
   - Maximum 3-4 dimensions (sinon explosion combinatoire)

### ❌ À Éviter

1. **CUBE sur trop de dimensions**
   ```sql
   -- ❌ Génère 2⁵ = 32 groupements !
   GROUP BY CUBE(col1, col2, col3, col4, col5);

   -- ✅ Utiliser GROUPING SETS à la place
   GROUP BY GROUPING SETS (
       (col1, col2),
       (col3, col4),
       ()
   );
   ```

2. **Oublier de gérer les NULL dans les résultats**
   - Toujours utiliser COALESCE, NULLIF ou CASE pour clarifier

3. **Utiliser ROLLUP quand CUBE est nécessaire**
   ```sql
   -- ❌ ROLLUP ne donnera pas les totaux par catégorie
   GROUP BY ROLLUP(region, categorie);

   -- ✅ CUBE si vous voulez les deux axes
   GROUP BY CUBE(region, categorie);
   ```

4. **Ne pas vérifier le nombre de lignes générées**
   ```sql
   -- Toujours vérifier combien de lignes seront générées
   SELECT COUNT(*) FROM (
       SELECT 1
       FROM ventes
       GROUP BY CUBE(col1, col2, col3)
   ) sub;
   ```

---

## À Retenir

1. **ROLLUP** génère des sous-totaux **hiérarchiques** (gauche → droite)  
2. **CUBE** génère **toutes les combinaisons** de sous-totaux (2^N groupements)  
3. **GROUPING SETS** permet de spécifier **exactement** quels groupements vous voulez  
4. Les extensions sont **plus performantes** que plusieurs UNION ALL (un seul scan)  
5. **GROUPING()** identifie les lignes de sous-totaux (1 = agrégation, 0 = valeur réelle)  
6. Utilisez **COALESCE** ou **CASE** pour rendre les NULL lisibles  
7. **ROLLUP** : Hiérarchies naturelles (date, géographie)  
8. **CUBE** : Analyses croisées exhaustives  
9. **GROUPING SETS** : Optimisation et sur-mesure  
10. Attention à l'explosion combinatoire avec **CUBE** sur beaucoup de colonnes

---

## Prochaines Étapes

Vous maîtrisez maintenant les extensions de groupement ! Prochains sujets :

- **Section 8.5** : Filtres d'agrégation (FILTER clause) - Agrégations conditionnelles  
- **Section 8.6** : Agrégations ordonnées (WITHIN GROUP, string_agg)  
- **Section 10** : Window Functions - Le summum de l'analyse SQL !

Les extensions de groupement (ROLLUP, CUBE, GROUPING SETS) sont des outils **extrêmement puissants** pour créer des rapports multi-niveaux et des tableaux de bord analytiques. Elles transforment des requêtes complexes avec multiples UNION ALL en requêtes élégantes et performantes !

⏭️ [Filtres d'agrégation (FILTER clause)](/08-agregation-et-groupement/05-filtres-agregation.md)
