🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.4. Index Spécialisés : Introduction et Vue d'Ensemble

## Introduction

Jusqu'à présent, nous avons principalement parlé de l'index **B-Tree**, le type d'index par défaut et le plus universel de PostgreSQL. B-Tree est excellent pour la majorité des cas d'usage, mais PostgreSQL propose également des **index spécialisés** conçus pour des types de données et des patterns de recherche spécifiques.

### L'analogie de la boîte à outils

Imaginez que vous devez effectuer différents travaux :
- **B-Tree** : C'est comme un tournevis multifonction polyvalent qui fonctionne pour la plupart des vis
- **Index spécialisés** : Ce sont des outils spécifiques comme une clé Allen, un tournevis Torx, une perceuse... Chacun excelle dans sa spécialité

De la même manière, PostgreSQL offre différents types d'index, chacun optimisé pour des situations particulières.

### Pourquoi des index spécialisés ?

B-Tree est conçu pour des données **ordonnables et unidimensionnelles** (nombres, texte, dates). Mais que faire quand :
- Vous cherchez dans des **tableaux** ou du **JSON** ?
- Vous avez des **données géospatiales** (points, polygones) ?
- Votre table fait **plusieurs téraoctets** et vous voulez économiser l'espace ?
- Vous ne faites que des **recherches d'égalité stricte** ?
- Vous cherchez des **préfixes** dans du texte ?

C'est là qu'interviennent les index spécialisés !

---

## 1. Vue d'Ensemble des Index Spécialisés

PostgreSQL propose **cinq types d'index spécialisés** principaux :

### 1.1. GIN (Generalized Inverted Index)

**Concept** : Index inversé qui indexe les éléments à l'intérieur des valeurs composites.

**Cas d'usage principaux** :
- 📦 **Arrays** : Recherche dans des tableaux
- 📄 **JSONB** : Recherche dans des documents JSON
- 🔍 **Full-Text Search** : Recherche textuelle avancée

**Exemple typique** :
```sql
-- Trouver tous les articles ayant le tag 'postgresql'
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];
```

**Métaphore** : Comme l'index d'un livre qui liste chaque mot et les pages où il apparaît.

### 1.2. GiST (Generalized Search Tree)

**Concept** : Arbre de recherche généralisé pour données multidimensionnelles.

**Cas d'usage principaux** :
- 🗺️ **Géométrie** : Points, polygones, lignes (PostGIS)
- 🌳 **Hiérarchies** : Arbres avec ltree
- 📝 **Full-Text Search** : Alternative à GIN (meilleur en écriture)
- 📊 **Ranges** : Intervalles temporels ou numériques

**Exemple typique** :
```sql
-- Trouver tous les restaurants dans un rayon de 1 km
SELECT * FROM restaurants
WHERE ST_DWithin(location, mon_point, 1000);
```

**Métaphore** : Comme une carte avec des régions qui peuvent se chevaucher.

### 1.3. BRIN (Block Range Index)

**Concept** : Index ultra-compact qui résume des plages de blocs plutôt que chaque valeur.

**Cas d'usage principaux** :
- 📈 **Tables massives** : Plusieurs Go à plusieurs To
- 📅 **Données chronologiques** : Logs, métriques, time-series
- 🔢 **Données séquentielles** : IDs auto-incrémentés
- 💾 **Économie d'espace** : Index 1000× plus petit que B-Tree

**Exemple typique** :
```sql
-- Logs des 7 derniers jours (table de milliards de lignes)
SELECT * FROM application_logs
WHERE timestamp > NOW() - INTERVAL '7 days';
```

**Métaphore** : Comme une table des matières de livre qui indique "le chapitre 3 couvre les pages 50-100".

### 1.4. Hash

**Concept** : Index basé sur des fonctions de hachage pour égalité stricte.

**Cas d'usage principaux** :
- 🔑 **Égalité uniquement** : WHERE colonne = valeur
- 🎯 **UUID lookup** : Recherche de tokens, sessions
- 🆔 **Clés uniques** : Identifiants, codes

**Exemple typique** :
```sql
-- Vérifier si une session existe
SELECT * FROM sessions WHERE session_id = 'abc123...';
```

**Métaphore** : Comme un casier postal où chaque nom a un numéro de casier unique.

**Note importante** : En pratique, B-Tree est souvent aussi performant et plus flexible.

### 1.5. SP-GiST (Space-Partitioned GiST)

**Concept** : Arbre non-équilibré pour structures partitionnées dans l'espace.

**Cas d'usage principaux** :
- 🔤 **Préfixes** : Recherches LIKE 'prefix%'
- 🌐 **Réseaux** : Adresses IP, CIDR
- 📍 **Points 2D** : Quadtree pour données spatiales
- 🔗 **Structures hiérarchiques** : Chemins, clés composées

**Exemple typique** :
```sql
-- Autocomplétion d'emails
SELECT * FROM users WHERE email LIKE 'admin%';
```

**Métaphore** : Comme un arbre généalogique où certaines branches sont plus longues que d'autres.

---

## 2. Tableau Comparatif Général

| Index | Taille | Vitesse lecture | Vitesse écriture | Polyvalence | Cas d'usage principal |
|-------|--------|----------------|------------------|-------------|----------------------|
| **B-Tree** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Défaut, usage général |
| **GIN** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | Arrays, JSONB, texte |
| **GiST** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Géométrie, hiérarchies |
| **BRIN** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | Tables massives ordonnées |
| **Hash** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ | Égalité stricte uniquement |
| **SP-GiST** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | Préfixes, réseaux, quadtree |

---

## 3. Choisir le Bon Type d'Index : Guide de Décision

### Étape 1 : Quel est votre type de données ?

```
Type de données :
├─ Scalaire simple (INTEGER, TEXT, DATE) → B-Tree (99% des cas)
├─ Array (TEXT[], INTEGER[]) → GIN
├─ JSONB → GIN (ou GiST si écritures fréquentes)
├─ tsvector (Full-Text) → GIN (ou GiST si écritures fréquentes)
├─ geometry/geography (PostGIS) → GiST (ou SP-GiST pour points)
├─ ltree (hiérarchies) → GiST
├─ INET/CIDR (réseaux) → SP-GiST
├─ range types (tstzrange, numrange) → GiST ou SP-GiST
└─ UUID (lookup strict) → B-Tree (ou Hash)
```

### Étape 2 : Quelle est votre opération de recherche ?

```
Opération principale :
├─ Égalité (=) → B-Tree ou Hash
├─ Comparaison (>, <, >=, <=) → B-Tree uniquement
├─ BETWEEN → B-Tree ou BRIN (si données ordonnées)
├─ ORDER BY → B-Tree uniquement
├─ LIKE 'prefix%' → B-Tree ou SP-GiST
├─ LIKE '%suffix' ou LIKE '%middle%' → GIN avec pg_trgm
├─ @> (contient) pour arrays/JSONB → GIN
├─ && (chevauche) pour arrays/géométrie → GIN ou GiST
├─ Full-Text Search (@@) → GIN ou GiST
└─ Opérations spatiales (ST_*) → GiST ou SP-GiST
```

### Étape 3 : Caractéristiques de vos données

```
Taille et structure :
├─ Table < 1 Go → B-Tree (sauf besoin spécifique)
├─ Table > 100 Go avec données ordonnées chronologiquement → BRIN
├─ Données naturellement ordonnées (timestamp, auto-increment) → BRIN possible
├─ Données aléatoires → B-Tree, GIN, GiST, Hash (selon type)
├─ Beaucoup d'insertions → Éviter GIN, préférer GiST ou BRIN
└─ Données stables (peu de modifications) → GIN acceptable
```

---

## 4. Règles Générales et Bonnes Pratiques

### Règle d'Or : Commencez par B-Tree

**En cas de doute, utilisez B-Tree.** C'est l'index le plus :
- ✅ **Polyvalent** : Supporte presque toutes les opérations
- ✅ **Mature** : 40+ ans d'optimisations
- ✅ **Fiable** : Comportement prévisible et bien compris
- ✅ **Performant** : Excellent pour 90% des cas

Ne passez à un index spécialisé que si vous avez une **raison mesurée**.

### Quand envisager un index spécialisé ?

**Considérez un index spécialisé si :**
1. ✅ B-Tree **ne supporte pas** l'opération (ex: @>, &&, ST_Within)
2. ✅ Vous avez des **benchmarks** montrant un gain significatif
3. ✅ Vous avez une **contrainte d'espace** (BRIN pour tables massives)
4. ✅ Le **type de données** le nécessite (JSONB, geometry)

### Processus de décision recommandé

```
1. Identifier le besoin
   ↓
2. Commencer par B-Tree
   ↓
3. Mesurer les performances (EXPLAIN ANALYZE)
   ↓
4. Si insuffisant, identifier l'index spécialisé approprié
   ↓
5. Créer l'index spécialisé en parallèle (CONCURRENTLY)
   ↓
6. Comparer les performances
   ↓
7. Garder le meilleur, supprimer l'autre
   ↓
8. Documenter le choix
```

---

## 5. Exemples Concrets par Scénario

### Scénario 1 : Blog avec système de tags

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre TEXT NOT NULL,
    contenu TEXT,
    tags TEXT[],  -- Array de tags
    date_publication TIMESTAMPTZ DEFAULT NOW()
);

-- Index spécialisés appropriés :
CREATE INDEX idx_tags_gin ON articles USING GIN (tags);  -- Pour @>
CREATE INDEX idx_date_brin ON articles USING BRIN (date_publication);  -- Si volume massif
CREATE INDEX idx_contenu_gin ON articles USING GIN (to_tsvector('french', contenu));  -- Full-text

-- Requêtes optimisées :
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];  -- Utilise GIN
SELECT * FROM articles WHERE date_publication > NOW() - INTERVAL '30 days';  -- Utilise BRIN
SELECT * FROM articles WHERE to_tsvector('french', contenu) @@ to_tsquery('french', 'database');  -- Utilise GIN
```

### Scénario 2 : Application de livraison géolocalisée

```sql
CREATE TABLE restaurants (
    id SERIAL PRIMARY KEY,
    nom TEXT NOT NULL,
    location geometry(Point, 4326),  -- PostGIS
    zone_livraison geometry(Polygon, 4326)
);

-- Index spatial GiST
CREATE INDEX idx_location_gist ON restaurants USING GIST (location);
CREATE INDEX idx_zone_gist ON restaurants USING GIST (zone_livraison);

-- Requêtes spatiales :
SELECT * FROM restaurants
WHERE ST_DWithin(location, ST_Point(2.35, 48.86), 5000);  -- Dans 5 km

SELECT * FROM restaurants
WHERE ST_Within(ST_Point(2.35, 48.86), zone_livraison);  -- Point dans polygone
```

### Scénario 3 : Système de logs massif (IoT/Metrics)

```sql
CREATE TABLE sensor_data (
    sensor_id INTEGER,
    timestamp TIMESTAMPTZ NOT NULL,
    temperature NUMERIC(5,2),
    humidity NUMERIC(5,2)
);

-- BRIN pour économiser l'espace (table de plusieurs To)
CREATE INDEX idx_timestamp_brin ON sensor_data USING BRIN (timestamp);

-- Requêtes de time-series :
SELECT AVG(temperature)
FROM sensor_data
WHERE timestamp > NOW() - INTERVAL '24 hours';  -- BRIN élimine 99% des blocs
```

### Scénario 4 : API avec catalogue produits flexible

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    sku TEXT UNIQUE NOT NULL,
    specifications JSONB,
    price NUMERIC(10,2)
);

-- GIN pour recherches JSONB
CREATE INDEX idx_specs_gin ON products USING GIN (specifications jsonb_path_ops);

-- Requêtes JSON :
SELECT * FROM products
WHERE specifications @> '{"brand": "Samsung", "screen_size": 6.5}';  -- GIN optimal
```

### Scénario 5 : Système d'autocomplétion

```sql
CREATE TABLE search_terms (
    id SERIAL PRIMARY KEY,
    term TEXT NOT NULL,
    popularity INTEGER
);

-- SP-GiST pour préfixes
CREATE INDEX idx_term_spgist ON search_terms USING SPGIST (term);

-- Autocomplétion :
SELECT term FROM search_terms
WHERE term LIKE 'postgr%'  -- SP-GiST très efficace
ORDER BY popularity DESC
LIMIT 10;
```

---

## 6. Comprendre les Opérateurs Spécialisés

Chaque type d'index supporte des **opérateurs spécifiques**. Voici les plus courants :

### Opérateurs B-Tree (standard)
```sql
=, <>, <, <=, >, >=, BETWEEN, IN, IS NULL, ORDER BY
```

### Opérateurs GIN
```sql
-- Arrays
@>  (contient)
<@  (est contenu dans)
&&  (chevauche)

-- JSONB
@>  (contient)
?   (a la clé)
?&  (a toutes les clés)
?|  (a au moins une clé)

-- Full-Text
@@  (correspond à la recherche)
```

### Opérateurs GiST
```sql
-- Géométrie (PostGIS)
&&  (chevauchement de boîtes)
@>  (contient)
<@  (est contenu dans)
~=  (identique)
ST_*  (fonctions spatiales)

-- ltree
@>  (est un ancêtre)
<@  (est un descendant)
~   (correspond au pattern)
```

### Opérateurs SP-GiST
```sql
-- Texte
LIKE 'prefix%'  (préfixe)
^@  (commence par)

-- Réseaux
<<  (strictement contenu dans)
<<=  (contenu dans ou égal)
>>  (strictement contient)
>>=  (contient ou égal)
```

### Opérateurs BRIN
```sql
=, <, <=, >, >=, BETWEEN
-- Identiques à B-Tree mais sur plages de blocs
```

### Opérateurs Hash
```sql
=  (égalité uniquement)
-- Rien d'autre !
```

---

## 7. Performance et Taille : Ce qu'il Faut Savoir

### Taille des index (pour 1 million de lignes typiques)

```
Colonne INTEGER (4 bytes) :
├─ B-Tree : ~20 MB
├─ Hash :   ~20 MB
├─ BRIN :   ~40 KB (500× plus petit !)
└─ Index inutile : 0 MB (parfois c'est le mieux !)

Colonne TEXT (emails) :
├─ B-Tree : ~40 MB
├─ GIN (avec trigrams) : ~80 MB
├─ SP-GiST : ~45 MB
└─ BRIN : Inadapté (données non ordonnées)

Colonne JSONB (documents) :
├─ GIN (jsonb_ops) : ~100 MB
├─ GIN (jsonb_path_ops) : ~60 MB (plus compact)
├─ GiST : ~120 MB
└─ B-Tree : Impossible

Colonne geometry (points PostGIS) :
├─ GiST : ~50 MB
├─ SP-GiST : ~45 MB (si distribution uniforme)
└─ B-Tree : Impossible
```

### Vitesse de recherche (ordres de grandeur)

```
Recherche d'égalité sur 10M lignes :
├─ Sans index : ~500 ms (scan séquentiel)
├─ B-Tree : ~0.05 ms
├─ Hash : ~0.04 ms
├─ GIN/GiST : ~0.1 ms (selon type)
└─ BRIN : ~20 ms (mais index 1000× plus petit)

Recherche de range sur 10M lignes :
├─ Sans index : ~500 ms
├─ B-Tree : ~5 ms
├─ BRIN : ~30 ms (données ordonnées)
└─ Hash : Impossible

Recherche full-text sur 1M documents :
├─ Sans index : ~2000 ms
├─ B-Tree LIKE : ~500 ms (préfixe uniquement)
├─ GIN : ~2 ms
└─ GiST : ~10 ms (plus rapide en écriture)
```

---

## 8. Pièges Communs à Éviter

### Piège 1 : Créer trop d'index

```sql
-- ❌ PROBLÈME : Index sur tout
CREATE INDEX idx1 ON table(col1);
CREATE INDEX idx2 ON table(col2);
CREATE INDEX idx3 ON table(col3);
CREATE INDEX idx4 ON table(col1, col2);
CREATE INDEX idx5 ON table(col1, col3);
-- ... 10 index de plus

-- Conséquences :
-- - Insertions/updates très lentes
-- - Beaucoup d'espace disque gaspillé
-- - Maintenance coûteuse

-- ✅ SOLUTION : Index ciblés et mesurés
-- Créer uniquement les index utilisés par les requêtes fréquentes
-- Surveiller avec pg_stat_user_indexes
```

### Piège 2 : Mauvais choix de type d'index

```sql
-- ❌ PROBLÈME : BRIN sur données aléatoires
CREATE TABLE users (email TEXT);
-- Insertions dans le désordre
CREATE INDEX idx_email_brin ON users USING BRIN (email);  -- Inefficace !

-- ✅ SOLUTION : B-Tree pour données non ordonnées
CREATE INDEX idx_email_btree ON users (email);
```

### Piège 3 : Oublier ANALYZE après création

```sql
-- ❌ PROBLÈME
CREATE INDEX idx_new ON table(column);
-- L'index existe mais le planificateur ne sait pas comment l'utiliser

-- ✅ SOLUTION
CREATE INDEX idx_new ON table(column);
ANALYZE table;  -- Met à jour les statistiques
```

### Piège 4 : Ne jamais vérifier l'utilisation

```sql
-- ❌ PROBLÈME : Index créés "au cas où" jamais supprimés
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Jamais utilisé
  AND schemaname = 'public';

-- ✅ SOLUTION : Supprimer les index inutilisés
DROP INDEX CONCURRENTLY idx_inutilise;
```

### Piège 5 : Index spécialisé quand B-Tree suffit

```sql
-- ❌ PROBLÈME : Complexifier inutilement
CREATE INDEX idx_id_hash ON users USING HASH (id);

-- ✅ SOLUTION : B-Tree est aussi rapide et plus flexible
CREATE INDEX idx_id_btree ON users (id);
```

---

## 9. Méthodologie de Test et Validation

### Processus recommandé

```sql
-- 1. Identifier la requête lente
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];  -- 500 ms

-- 2. EXPLAIN ANALYZE sans index
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];
-- Seq Scan : 500 ms

-- 3. Créer l'index candidat (CONCURRENTLY en production)
CREATE INDEX CONCURRENTLY idx_tags_gin ON articles USING GIN (tags);

-- 4. ANALYZE la table
ANALYZE articles;

-- 5. EXPLAIN ANALYZE avec index
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];
-- Bitmap Index Scan using idx_tags_gin : 2 ms (250× plus rapide !)

-- 6. Vérifier l'utilisation sur période
SELECT
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE indexname = 'idx_tags_gin';

-- 7. Décider : garder ou supprimer
-- Si utilisé et utile → garder
-- Si jamais utilisé après 1 mois → supprimer
```

---

## 10. Checklist Avant de Créer un Index Spécialisé

### ✅ Questions à se poser

- [ ] **Ai-je essayé B-Tree d'abord ?**
- [ ] **L'opération est-elle supportée par B-Tree ?**
- [ ] **Ai-je mesuré les performances actuelles ?**
- [ ] **Le type de données nécessite-t-il un index spécialisé ?**
- [ ] **La requête est-elle fréquente ?**
- [ ] **Ai-je assez d'espace disque ?**
- [ ] **Ai-je considéré l'impact sur les écritures ?**
- [ ] **Puis-je tester en préproduction d'abord ?**
- [ ] **Ai-je documenté la raison de ce choix ?**

### ✅ Actions post-création

- [ ] Exécuter `ANALYZE table`
- [ ] Vérifier avec `EXPLAIN ANALYZE`
- [ ] Mesurer le gain réel de performance
- [ ] Documenter l'index (COMMENT)
- [ ] Monitorer l'utilisation (pg_stat_user_indexes)
- [ ] Planifier la maintenance (REINDEX si nécessaire)

---

## 11. Ressources et Prochaines Étapes

### Documentation officielle PostgreSQL 18

- [Index Types](https://www.postgresql.org/docs/18/indexes-types.html)
- [Indexes and Operator Classes](https://www.postgresql.org/docs/18/indexes-opclass.html)
- [GIN Indexes](https://www.postgresql.org/docs/18/gin.html)
- [GiST Indexes](https://www.postgresql.org/docs/18/gist.html)
- [BRIN Indexes](https://www.postgresql.org/docs/18/brin.html)
- [Hash Indexes](https://www.postgresql.org/docs/18/hash-intro.html)
- [SP-GiST Indexes](https://www.postgresql.org/docs/18/spgist.html)

### Prochains chapitres de ce tutoriel

Dans les sections suivantes, nous explorerons en détail chaque type d'index spécialisé :

1. **13.4.1. GIN** : Index inversés pour arrays, JSONB et full-text search
2. **13.4.2. GiST** : Index pour géométrie, hiérarchies et données multidimensionnelles
3. **13.4.3. BRIN** : Index ultra-compacts pour tables massives ordonnées
4. **13.4.4. Hash** : Index pour égalité stricte uniquement
5. **13.4.5. SP-GiST** : Index pour structures partitionnées et préfixes

Chaque section détaillera :
- Le principe de fonctionnement
- Les cas d'usage concrets
- Les syntaxes et opérateurs
- Les comparaisons de performance
- Les bonnes pratiques
- Les pièges à éviter

---

## Conclusion

Les **index spécialisés** de PostgreSQL sont des outils puissants qui étendent les capacités d'indexation bien au-delà du B-Tree classique. Chacun a été conçu pour exceller dans des domaines spécifiques :

### Récapitulatif des forces

- **GIN** : Roi des structures composites (arrays, JSON, texte)
- **GiST** : Champion du spatial et du multidimensionnel
- **BRIN** : Maître de l'économie d'espace pour données massives
- **Hash** : Spécialiste de l'égalité stricte (mais B-Tree souvent suffisant)
- **SP-GiST** : Expert des structures hiérarchiques non-équilibrées

### Philosophie d'utilisation

**Simplicité d'abord** : Commencez toujours par B-Tree, le couteau suisse fiable et polyvalent.

**Spécialisation quand nécessaire** : Passez à un index spécialisé uniquement quand :
1. Le type de données l'exige (JSONB, geometry)
2. B-Tree ne supporte pas l'opération
3. Les gains de performance sont mesurés et significatifs

**Mesure et validation** : Utilisez toujours `EXPLAIN ANALYZE` et `pg_stat_user_indexes` pour valider vos choix.

**Documentation** : Expliquez pourquoi vous avez choisi cet index, pour vous (futur) et vos collègues.

### Pour aller plus loin

Les sections suivantes de ce tutoriel vous guideront dans l'utilisation pratique de chaque type d'index spécialisé, avec des exemples concrets, des benchmarks et des recommandations basées sur l'expérience.

**Prêt à explorer les index spécialisés ?** Commençons par GIN, l'index inversé qui révolutionne la recherche dans les structures composites ! 🚀

---


⏭️ [GIN (Generalized Inverted Index) : JSONB, Full-Text, Arrays](/13-indexation-et-optimisation/04.1-gin-generalized-inverted-index.md)
