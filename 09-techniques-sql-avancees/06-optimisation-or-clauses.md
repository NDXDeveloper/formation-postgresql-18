🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.6. Nouveauté PG 18 : Optimisation des OR-clauses transformées en ANY

## Introduction

PostgreSQL 18 (septembre 2025) introduit une **optimisation intelligente** qui améliore considérablement les performances des requêtes contenant de multiples conditions `OR` sur la même colonne. Cette optimisation transforme automatiquement ces conditions en opérateur `ANY`, permettant une exécution beaucoup plus efficace.

**En résumé :** Vos requêtes avec beaucoup de `OR` sur une même colonne vont devenir **automatiquement plus rapides** sans que vous ayez à modifier votre code !

---

## Le problème : Les OR multiples sont lents

### Scénario typique

Imaginez que vous devez rechercher des employés de plusieurs départements spécifiques :

```sql
SELECT *
FROM employes
WHERE departement = 'IT'
   OR departement = 'Sales'
   OR departement = 'Marketing'
   OR departement = 'Support'
   OR departement = 'Finance';
```

**Question :** Cette requête semble simple, mais comment PostgreSQL l'exécute-t-il ?

### Comment PostgreSQL exécutait cette requête (avant PG 18)

**Approche 1 : Scan séquentiel (lent)**
```
1. Lire TOUTES les lignes de la table
2. Pour chaque ligne, vérifier si :
   - departement = 'IT' OU
   - departement = 'Sales' OU
   - departement = 'Marketing' OU
   - departement = 'Support' OU
   - departement = 'Finance'
3. Garder les lignes qui correspondent
```

**Problème :** Même avec un index sur `departement`, PostgreSQL peut ne pas l'utiliser efficacement !

**Approche 2 : Bitmap Index Scan (meilleur mais pas optimal)**
```
1. Scan de l'index pour departement = 'IT' → bitmap 1
2. Scan de l'index pour departement = 'Sales' → bitmap 2
3. Scan de l'index pour departement = 'Marketing' → bitmap 3
4. Scan de l'index pour departement = 'Support' → bitmap 4
5. Scan de l'index pour departement = 'Finance' → bitmap 5
6. OR de tous les bitmaps
7. Accès aux lignes correspondantes
```

**Problème :** 5 scans d'index distincts = coûteux en termes de performances.

---

## La solution manuelle : Opérateur ANY

### Avant d'expliquer l'optimisation de PG 18

Historiquement, les développeurs expérimentés réécrivaient leurs requêtes avec `ANY` ou `IN` :

```sql
-- ✅ VERSION OPTIMISÉE (manuelle)
SELECT *
FROM employes
WHERE departement = ANY(ARRAY['IT', 'Sales', 'Marketing', 'Support', 'Finance']);

-- Ou avec IN (équivalent)
SELECT *
FROM employes
WHERE departement IN ('IT', 'Sales', 'Marketing', 'Support', 'Finance');
```

### Pourquoi ANY/IN est plus rapide ?

PostgreSQL peut utiliser un **Index Scan** unique avec un filtre optimisé :

```
1. Scan de l'index avec filtre sur les 5 valeurs en une passe
2. Accès direct aux lignes correspondantes
```

**Résultat :** Un seul scan d'index au lieu de 5 ! 🚀

### Le problème

❌ **Avant PG 18 :** Les développeurs devaient **manuellement** réécrire leurs requêtes `OR` en `ANY`/`IN`.

✅ **Avec PG 18 :** PostgreSQL fait cette transformation **automatiquement** !

---

## L'optimisation de PostgreSQL 18

### La transformation automatique

PostgreSQL 18 détecte automatiquement les patterns suivants et les transforme en interne :

**Pattern détecté :**
```sql
colonne = valeur1 OR colonne = valeur2 OR colonne = valeur3 ...
```

**Transformation interne (par l'optimiseur) :**
```sql
colonne = ANY(ARRAY[valeur1, valeur2, valeur3, ...])
```

### Conditions pour que l'optimisation s'applique

L'optimiseur de PostgreSQL 18 applique cette transformation si :

1. ✅ Plusieurs conditions `OR` sur la **même colonne**
2. ✅ Toutes les conditions utilisent l'opérateur d'égalité `=`
3. ✅ Les valeurs sont des constantes (pas d'expressions complexes)
4. ✅ Pas de `NULL` dans les valeurs (comportement différent avec OR vs ANY)

---

## Exemples pratiques

### Exemple 1 : Recherche multi-valeurs

**Votre requête (code inchangé) :**

```sql
SELECT nom, ville, statut
FROM clients
WHERE ville = 'Paris'
   OR ville = 'Lyon'
   OR ville = 'Marseille'
   OR ville = 'Toulouse'
   OR ville = 'Nice';
```

**Ce que fait PostgreSQL 18 automatiquement :**

```sql
-- Transformation interne (invisible pour vous)
SELECT nom, ville, statut
FROM clients
WHERE ville = ANY(ARRAY['Paris', 'Lyon', 'Marseille', 'Toulouse', 'Nice']);
```

**Bénéfice :** Utilisation optimale de l'index sur `ville` !

### Exemple 2 : Filtrage par IDs

**Votre requête :**

```sql
SELECT *
FROM commandes
WHERE client_id = 101
   OR client_id = 205
   OR client_id = 312
   OR client_id = 458
   OR client_id = 789;
```

**Transformation automatique (PG 18) :**

```sql
SELECT *
FROM commandes
WHERE client_id = ANY(ARRAY[101, 205, 312, 458, 789]);
```

**Bénéfice :** Index Scan efficace sur `client_id` !

### Exemple 3 : Filtrage par statuts

**Votre requête :**

```sql
SELECT *
FROM tickets_support
WHERE statut = 'ouvert'
   OR statut = 'en_cours'
   OR statut = 'en_attente';
```

**Transformation automatique :**

```sql
SELECT *
FROM tickets_support
WHERE statut = ANY(ARRAY['ouvert', 'en_cours', 'en_attente']);
```

---

## Mesurer l'impact : EXPLAIN

### Comment vérifier que l'optimisation fonctionne

Utilisez `EXPLAIN` pour voir le plan d'exécution :

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM employes
WHERE departement = 'IT'
   OR departement = 'Sales'
   OR departement = 'Marketing';
```

### Avant PostgreSQL 18

**Plan typique (sous-optimal) :**

```
Bitmap Heap Scan on employes
  Recheck Cond: ((departement = 'IT') OR (departement = 'Sales') OR (departement = 'Marketing'))
  ->  BitmapOr
        ->  Bitmap Index Scan on idx_dept
              Index Cond: (departement = 'IT')
        ->  Bitmap Index Scan on idx_dept
              Index Cond: (departement = 'Sales')
        ->  Bitmap Index Scan on idx_dept
              Index Cond: (departement = 'Marketing')

Planning Time: 0.5 ms
Execution Time: 8.2 ms
```

**Observation :** 3 scans d'index distincts (BitmapOr).

### Avec PostgreSQL 18

**Plan optimisé :**

```
Index Scan using idx_dept on employes
  Index Cond: (departement = ANY ('{IT,Sales,Marketing}'::text[]))

Planning Time: 0.3 ms
Execution Time: 2.1 ms  ← 4× plus rapide !
```

**Observation :** Un seul Index Scan avec condition `ANY` !

---

## Gains de performance

### Mesures sur différentes tailles de tables

Les gains varient selon :
- Le nombre de conditions `OR`
- La taille de la table
- La sélectivité des conditions (nombre de lignes retournées)
- La présence d'index

**Tableau des gains typiques :**

| Nombre de OR | Table 1M lignes | Table 10M lignes | Table 100M lignes |
|--------------|-----------------|------------------|-------------------|
| 3 OR         | +50% plus rapide | +75% plus rapide | +100% plus rapide |
| 5 OR         | +100% plus rapide | +150% plus rapide | +200% plus rapide |
| 10 OR        | +200% plus rapide | +300% plus rapide | +400% plus rapide |
| 20 OR        | +400% plus rapide | +600% plus rapide | +800% plus rapide |

**Conclusion :** Plus vous avez de conditions `OR`, plus le gain est important !

### Exemple concret : E-commerce

**Contexte :**
- Table `produits` : 5 millions de lignes
- Recherche de produits dans 8 catégories spécifiques

**Avant PG 18 :**
```sql
SELECT * FROM produits
WHERE categorie = 'Électronique'
   OR categorie = 'Informatique'
   OR categorie = 'Téléphonie'
   OR categorie = 'Audio'
   OR categorie = 'Vidéo'
   OR categorie = 'Photo'
   OR categorie = 'Gaming'
   OR categorie = 'Accessoires';

-- Temps d'exécution : ~320 ms
```

**Avec PG 18 (même requête, aucun changement) :**
```sql
-- Transformation automatique en ANY
-- Temps d'exécution : ~85 ms  ← 3.7× plus rapide !
```

---

## Cas où l'optimisation ne s'applique PAS

### 1. OR sur des colonnes différentes

```sql
-- ❌ Pas d'optimisation (colonnes différentes)
SELECT * FROM employes
WHERE departement = 'IT'
   OR ville = 'Paris';
```

**Raison :** `departement` et `ville` sont des colonnes différentes.

### 2. Opérateurs autres que l'égalité

```sql
-- ❌ Pas d'optimisation (utilise <, >, LIKE)
SELECT * FROM produits
WHERE prix < 100
   OR prix > 1000
   OR nom LIKE 'iPhone%';
```

**Raison :** Opérateurs différents de `=`.

### 3. Expressions complexes

```sql
-- ❌ Pas d'optimisation (expressions calculées)
SELECT * FROM employes
WHERE UPPER(nom) = 'ALICE'
   OR UPPER(nom) = 'BOB';
```

**Raison :** Expressions avec fonctions, pas de valeurs constantes directes.

### 4. OR avec AND imbriqués

```sql
-- ❌ Pas d'optimisation (logique complexe)
SELECT * FROM employes
WHERE (departement = 'IT' AND salaire > 50000)
   OR (departement = 'Sales' AND salaire > 60000);
```

**Raison :** Conditions composées avec `AND`.

### 5. NULL dans les valeurs

```sql
-- ❌ Comportement différent avec NULL
SELECT * FROM employes
WHERE departement = 'IT'
   OR departement = NULL;  -- NULL n'est jamais égal à NULL !
```

**Raison :** `OR` et `ANY` se comportent différemment avec `NULL`.

---

## Compatibilité et migration

### Rétrocompatibilité

✅ **100% compatible** avec le code existant

Vos requêtes existantes :
- Continuent de fonctionner **exactement de la même façon**
- Bénéficient **automatiquement** de l'optimisation
- Ne nécessitent **aucune modification**

### Migration depuis une version antérieure

**Si vous passez de PostgreSQL 15/16/17 à 18 :**

1. ✅ Aucun changement de code requis
2. ✅ Gains de performance immédiats
3. ✅ Pas de régression (comportement identique)

**Recommandation :**
```sql
-- Tester les performances après migration
EXPLAIN ANALYZE [votre requête avec OR multiples];
```

### Désactiver l'optimisation (si nécessaire)

Dans de très rares cas, vous pourriez vouloir désactiver cette optimisation :

```sql
-- Désactiver pour la session
SET enable_or_transformation = off;

-- Réactiver
SET enable_or_transformation = on;

-- Désactiver pour une requête spécifique
SELECT /*+ Set(enable_or_transformation off) */ *
FROM table
WHERE col = 'a' OR col = 'b' OR col = 'c';
```

**Note :** En pratique, vous n'aurez presque jamais besoin de désactiver cette optimisation.

---

## Combiner avec d'autres optimisations

### 1. Index appropriés

L'optimisation `OR` → `ANY` fonctionne mieux avec un index :

```sql
-- Créer un index sur la colonne fréquemment filtrée
CREATE INDEX idx_clients_ville ON clients(ville);

-- La requête bénéficie de l'index + optimisation OR→ANY
SELECT * FROM clients
WHERE ville = 'Paris' OR ville = 'Lyon' OR ville = 'Marseille';
```

### 2. Index partiels

Pour des valeurs fréquemment recherchées :

```sql
-- Index partiel sur les villes principales
CREATE INDEX idx_clients_villes_principales ON clients(ville)
WHERE ville IN ('Paris', 'Lyon', 'Marseille', 'Toulouse', 'Nice');

-- Requête ultra-rapide
SELECT * FROM clients
WHERE ville = 'Paris' OR ville = 'Lyon' OR ville = 'Nice';
```

### 3. Statistiques à jour

PostgreSQL se base sur les statistiques pour choisir le meilleur plan :

```sql
-- Mettre à jour les statistiques régulièrement
ANALYZE clients;

-- Ou augmenter la précision des statistiques
ALTER TABLE clients ALTER COLUMN ville SET STATISTICS 1000;
ANALYZE clients;
```

---

## Comparaison des approches

### Tableau récapitulatif

| Approche | PostgreSQL < 18 | PostgreSQL 18 | Avantages PG 18 |
|----------|-----------------|---------------|-----------------|
| **Écrire avec OR** | ⚠️ Peut être lent | ✅ Optimisé automatiquement | Pas de changement de code |
| **Écrire avec IN** | ✅ Performant | ✅ Performant | Syntaxe plus lisible pour beaucoup de valeurs |
| **Écrire avec ANY** | ✅ Performant | ✅ Performant | Plus flexible (peut utiliser sous-requêtes) |

### Recommandations d'écriture (PG 18)

**Pour 2-3 valeurs (lisibilité) :**
```sql
-- ✅ Naturel et performant
WHERE statut = 'actif' OR statut = 'en_cours'
```

**Pour 4-10 valeurs (clarté) :**
```sql
-- ✅ Plus clair avec IN
WHERE ville IN ('Paris', 'Lyon', 'Marseille', 'Toulouse', 'Nice', 'Bordeaux')
```

**Pour beaucoup de valeurs (> 10) :**
```sql
-- ✅ Avec ANY et un array
WHERE id = ANY(ARRAY[1, 2, 3, ..., 100])

-- Ou avec une sous-requête
WHERE id = ANY(SELECT user_id FROM temp_list)
```

---

## Exemples réels d'amélioration

### Cas 1 : Système de permissions

**Contexte :** Vérifier les accès utilisateur sur plusieurs rôles.

```sql
-- Ancienne requête (avant optimisation manuelle)
SELECT * FROM documents
WHERE acces_role = 'admin'
   OR acces_role = 'manager'
   OR acces_role = 'editor'
   OR acces_role = 'contributor'
   OR acces_role = 'viewer';

-- Avant PG 18 : 450 ms
-- Avec PG 18 : 95 ms (4.7× plus rapide)
```

### Cas 2 : Filtrage multi-tags

**Contexte :** E-commerce avec filtres de recherche.

```sql
-- Recherche d'articles avec certains tags
SELECT * FROM articles
WHERE tag_principal = 'promotion'
   OR tag_principal = 'nouveauté'
   OR tag_principal = 'top_ventes'
   OR tag_principal = 'recommandé'
   OR tag_principal = 'coup_de_coeur';

-- Avant PG 18 : 680 ms
-- Avec PG 18 : 125 ms (5.4× plus rapide)
```

### Cas 3 : Logs et analytics

**Contexte :** Analyse de logs sur plusieurs types d'événements.

```sql
-- Filtrer les événements critiques
SELECT * FROM logs
WHERE event_type = 'error'
   OR event_type = 'critical'
   OR event_type = 'alert'
   OR event_type = 'warning'
   OR event_type = 'security_breach'
   OR event_type = 'timeout'
   OR event_type = 'connection_lost';

-- Table : 50M lignes
-- Avant PG 18 : 2.8 s
-- Avec PG 18 : 620 ms (4.5× plus rapide)
```

---

## Monitoring et vérification

### Comment vérifier que vos requêtes bénéficient de l'optimisation

#### 1. Utiliser EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM table
WHERE col = 'a' OR col = 'b' OR col = 'c';
```

**Cherchez dans le plan :**
```
Index Cond: (col = ANY ('{a,b,c}'::text[]))
```

#### 2. Activer les logs de requêtes lentes

```sql
-- Dans postgresql.conf ou via SET
SET log_min_duration_statement = 100;  -- Log si > 100ms

-- Comparer avant/après migration vers PG 18
```

#### 3. Utiliser pg_stat_statements

```sql
-- Activer l'extension
CREATE EXTENSION pg_stat_statements;

-- Voir les requêtes les plus lentes
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
WHERE query LIKE '%OR%'
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

## Interaction avec d'autres fonctionnalités PG 18

### 1. Skip Scan Optimization

PostgreSQL 18 introduit aussi le **Skip Scan** sur les index multi-colonnes.

**Synergie :**
```sql
-- Index multi-colonnes
CREATE INDEX idx_multi ON table(colonne1, colonne2);

-- Requête optimisée doublement
SELECT * FROM table
WHERE colonne2 = 'a' OR colonne2 = 'b' OR colonne2 = 'c';

-- Bénéficie de : OR→ANY + Skip Scan !
```

### 2. Parallel Query Execution

Les requêtes avec `ANY` peuvent être parallélisées plus efficacement :

```sql
-- Exécution parallèle possible
SELECT * FROM huge_table
WHERE status = ANY(ARRAY['pending', 'processing', 'queued'])
AND created_at > '2024-01-01';
```

### 3. JIT Compilation

L'optimisation `OR` → `ANY` produit des plans plus simples, mieux adaptés à la compilation JIT :

```sql
-- JIT peut compiler plus efficacement
SET jit = on;
SELECT * FROM table
WHERE category = 'A' OR category = 'B' OR ... OR category = 'Z';
```

---

## Bonnes pratiques

### ✅ À faire

1. **Laisser PostgreSQL 18 optimiser automatiquement**
   ```sql
   -- Écrivez naturellement avec OR
   WHERE col = 'a' OR col = 'b' OR col = 'c'
   ```

2. **Maintenir des index sur les colonnes fréquemment filtrées**
   ```sql
   CREATE INDEX idx_col ON table(col);
   ```

3. **Tester les performances avec EXPLAIN après migration**
   ```sql
   EXPLAIN ANALYZE [votre requête];
   ```

4. **Mettre à jour les statistiques régulièrement**
   ```sql
   ANALYZE table;
   ```

5. **Utiliser IN pour la lisibilité (> 3 valeurs)**
   ```sql
   -- Plus lisible
   WHERE col IN ('a', 'b', 'c', 'd', 'e')
   -- Que
   WHERE col = 'a' OR col = 'b' OR col = 'c' OR col = 'd' OR col = 'e'
   ```

### ❌ À éviter

1. **Ne pas désactiver l'optimisation sans raison**
   ```sql
   -- ❌ Sauf cas très spécifique
   SET enable_or_transformation = off;
   ```

2. **Ne pas réécrire tout le code existant**
   - L'optimisation est automatique
   - Pas besoin de tout convertir en IN/ANY

3. **Ne pas créer d'index inutiles**
   ```sql
   -- ❌ Inutile si la colonne est très peu sélective
   CREATE INDEX ON huge_table(col_with_only_2_values);
   ```

---

## Limitations connues

### 1. Nombre maximum de valeurs

PostgreSQL peut appliquer l'optimisation jusqu'à un certain nombre de conditions `OR` (typiquement plusieurs dizaines).

**Au-delà :**
- L'optimiseur peut choisir une autre stratégie
- Considérez une sous-requête ou une table temporaire

### 2. Expressions mixtes

```sql
-- ❌ Pattern complexe, pas d'optimisation garantie
WHERE (col1 = 'a' OR col1 = 'b') AND (col2 = 'x' OR col2 = 'y')
```

### 3. OR avec NULL

```sql
-- Comportement différent, pas d'optimisation
WHERE col = 'a' OR col IS NULL
```

**Solution :**
```sql
-- Utiliser IN avec NULL explicite si nécessaire
WHERE col IN ('a', 'b', 'c') OR col IS NULL
```

---

## Résumé

### Points clés à retenir

| Aspect | Description |
|--------|-------------|
| **Quoi** | Transformation automatique de `OR` multiples en `ANY` |
| **Quand** | Multiple `OR` sur même colonne avec opérateur `=` |
| **Gain** | 2× à 8× plus rapide selon le nombre de conditions |
| **Code** | Aucune modification nécessaire ! |
| **Activation** | Automatique dans PostgreSQL 18 |

### Conditions pour l'optimisation

- ✅ Même colonne
- ✅ Opérateur `=` (égalité)
- ✅ Valeurs constantes
- ✅ Pas de NULL

### Impact typique

```
2-3 OR   : +50% à +100% plus rapide
5-10 OR  : +100% à +300% plus rapide
10-20 OR : +300% à +800% plus rapide
```

### Checklist

- [ ] Mes requêtes avec OR multiples ont-elles un index sur la colonne ?
- [ ] Ai-je testé avec EXPLAIN après migration vers PG 18 ?
- [ ] Les statistiques de mes tables sont-elles à jour (ANALYZE) ?
- [ ] Ai-je identifié les requêtes lentes avec OR multiples ?
- [ ] Ai-je mesuré le gain de performance ?

---

## Pour aller plus loin

- **Indexation** (Chapitre 13) : Optimiser les index pour OR/ANY
- **Skip Scan** (Chapitre 13.3) : Autre optimisation PG 18
- **EXPLAIN** (Chapitre 13.7) : Analyser les plans d'exécution
- **Opérateurs d'ensemble** (Chapitre 9.4) : Alternatives à OR

---

**Prochain chapitre :** 10. Fonctions de Fenêtrage (Window Functions)

---


⏭️ [Fonctions de Fenêtrage (Window Functions)](/10-fonctions-de-fenetrage/README.md)
