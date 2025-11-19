🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5. Pagination et limitation (LIMIT, OFFSET) : Avantages et limites

## Introduction

Lorsque vous interrogez une base de données, vous ne voulez pas toujours récupérer **toutes** les lignes d'une table. Imaginez un site e-commerce avec 100 000 produits : afficher tous les produits d'un coup serait :
- Très lent (temps de chargement)
- Gourmand en mémoire (côté serveur et client)
- Inutilisable pour l'utilisateur (impossible de tout parcourir)

C'est là qu'interviennent **LIMIT** et **OFFSET**, deux clauses SQL qui permettent de :
- **Limiter** le nombre de résultats retournés
- **Paginer** les résultats (afficher page par page)
- Améliorer les performances en ne chargeant que ce qui est nécessaire

Dans ce chapitre, nous allons explorer en profondeur ces clauses, comprendre leurs avantages et leurs limites, et découvrir des alternatives plus performantes pour certains cas d'usage.

---

## LIMIT : Limiter le nombre de résultats

### Syntaxe de base

```sql
SELECT colonnes
FROM table
WHERE conditions
ORDER BY colonne
LIMIT nombre_lignes;
```

**LIMIT** restreint le nombre de lignes retournées à `nombre_lignes`.

### Exemples simples

```sql
-- Retourner uniquement les 10 premières lignes
SELECT nom, prenom
FROM clients
LIMIT 10;

-- Top 5 des produits les plus chers
SELECT nom_produit, prix
FROM produits
ORDER BY prix DESC
LIMIT 5;

-- Les 3 employés les mieux payés
SELECT nom, salaire
FROM employes
ORDER BY salaire DESC
LIMIT 3;
```

**Résultat avec LIMIT 5 :**
```
nom_produit          | prix
---------------------|-------
MacBook Pro M3       | 2499
iPhone 15 Pro Max    | 1299
iPad Pro 13"         | 1199
Apple Watch Ultra    | 899
AirPods Max          | 599
-- Les autres produits ne sont pas retournés
```

### LIMIT sans ORDER BY : Comportement imprévisible

**⚠️ Attention :** Sans `ORDER BY`, l'ordre des lignes est **non déterministe**.

```sql
-- ❌ Ordre imprévisible (peut changer à chaque exécution)
SELECT nom FROM clients LIMIT 10;

-- ✅ Ordre déterministe
SELECT nom FROM clients
ORDER BY nom ASC
LIMIT 10;
```

**Pourquoi c'est important ?**
- PostgreSQL retourne les lignes dans l'ordre où elles sont physiquement stockées
- Cet ordre peut changer après des UPDATE, DELETE, VACUUM, etc.
- Résultat : vous pourriez obtenir des lignes différentes à chaque exécution

**Règle d'or :** Utilisez **toujours** `ORDER BY` avec `LIMIT` pour des résultats prévisibles.

### Cas d'usage de LIMIT

#### 1. Top N (classements)

```sql
-- Top 10 des ventes
SELECT produit, montant_ventes
FROM statistiques_ventes
ORDER BY montant_ventes DESC
LIMIT 10;

-- 5 articles de blog les plus récents
SELECT titre, date_publication
FROM articles
ORDER BY date_publication DESC
LIMIT 5;

-- 3 employés les plus anciens
SELECT nom, date_embauche
FROM employes
ORDER BY date_embauche ASC
LIMIT 3;
```

#### 2. Échantillonnage rapide

```sql
-- Aperçu rapide de la table (premières lignes)
SELECT * FROM produits LIMIT 20;

-- Exemple de données pour tests
SELECT id, nom, email FROM clients LIMIT 100;
```

#### 3. Vérifications d'existence

```sql
-- Vérifier s'il existe au moins un produit en rupture
SELECT nom_produit
FROM produits
WHERE stock = 0
LIMIT 1;

-- S'il retourne une ligne, au moins un produit est en rupture
```

#### 4. Optimisation de performances

```sql
-- Au lieu de compter toutes les lignes (coûteux)
SELECT COUNT(*) FROM commandes WHERE statut = 'en attente';

-- Si vous voulez juste savoir s'il y en a "beaucoup"
SELECT COUNT(*) FROM (
    SELECT 1 FROM commandes
    WHERE statut = 'en attente'
    LIMIT 1000
) sub;
```

---

## OFFSET : Sauter des lignes

### Syntaxe de base

```sql
SELECT colonnes
FROM table
WHERE conditions
ORDER BY colonne
LIMIT nombre_lignes
OFFSET nombre_lignes_a_sauter;
```

**OFFSET** indique combien de lignes **ignorer** avant de commencer à retourner des résultats.

### Fonctionnement

```sql
-- Sauter les 10 premières lignes, retourner les 5 suivantes
SELECT nom, prenom
FROM clients
ORDER BY nom ASC
LIMIT 5 OFFSET 10;
```

**Visualisation :**
```
Lignes 1-10  : [ignorées par OFFSET 10]
Lignes 11-15 : [retournées par LIMIT 5]
Lignes 16+   : [ignorées car LIMIT atteint]
```

### Exemples pratiques

```sql
-- Ignorer les 20 premières lignes, retourner les 10 suivantes
SELECT * FROM produits
ORDER BY nom ASC
LIMIT 10 OFFSET 20;

-- Sauter les 100 premières ventes, afficher les 50 suivantes
SELECT * FROM ventes
ORDER BY date_vente DESC
LIMIT 50 OFFSET 100;

-- Ignorer le podium (top 3), afficher les places 4 à 10
SELECT nom, score
FROM classement
ORDER BY score DESC
LIMIT 7 OFFSET 3;
```

### Syntaxe alternative (moins courante)

PostgreSQL supporte aussi la syntaxe `LIMIT ... OFFSET ...` dans cet ordre :

```sql
-- Syntaxe standard (recommandée)
LIMIT 10 OFFSET 20

-- Syntaxe alternative (fonctionne aussi)
OFFSET 20 LIMIT 10

-- Les deux sont équivalentes
```

---

## Pagination : Afficher les résultats page par page

La **pagination** consiste à diviser un grand ensemble de résultats en plusieurs "pages" plus petites.

### Principe de la pagination

**Formule générale :**
```
LIMIT  = taille_page (nombre de résultats par page)
OFFSET = (numero_page - 1) × taille_page
```

**Exemple avec 20 résultats par page :**

```sql
-- Page 1 (résultats 1-20)
SELECT * FROM produits
ORDER BY nom ASC
LIMIT 20 OFFSET 0;

-- Page 2 (résultats 21-40)
SELECT * FROM produits
ORDER BY nom ASC
LIMIT 20 OFFSET 20;

-- Page 3 (résultats 41-60)
SELECT * FROM produits
ORDER BY nom ASC
LIMIT 20 OFFSET 40;

-- Page N
-- OFFSET = (N - 1) × 20
```

### Implémentation pratique

Dans une application, vous recevez généralement :
- `page` : le numéro de page demandé (1, 2, 3, ...)
- `page_size` : le nombre d'éléments par page (10, 20, 50, ...)

**Calcul de OFFSET :**

```python
# Python (avec psycopg ou autre driver)
page = 3          # Page demandée
page_size = 20    # Résultats par page

offset = (page - 1) * page_size  # (3 - 1) × 20 = 40

query = """
    SELECT * FROM produits
    ORDER BY nom ASC
    LIMIT %s OFFSET %s
"""
cursor.execute(query, (page_size, offset))
```

```javascript
// JavaScript/Node.js (avec pg)
const page = 3;
const pageSize = 20;
const offset = (page - 1) * pageSize;

const query = `
    SELECT * FROM produits
    ORDER BY nom ASC
    LIMIT $1 OFFSET $2
`;
const result = await client.query(query, [pageSize, offset]);
```

```sql
-- SQL direct pour la page 5 avec 20 résultats par page
-- Page 5 : OFFSET = (5 - 1) × 20 = 80
SELECT * FROM produits
ORDER BY nom ASC
LIMIT 20 OFFSET 80;
```

### Informations de pagination complètes

Pour une pagination complète, vous devez souvent fournir :
- Les résultats de la page actuelle
- Le nombre total de résultats
- Le nombre total de pages

```sql
-- Requête 1 : Obtenir le nombre total de résultats
SELECT COUNT(*) as total FROM produits;

-- Requête 2 : Obtenir les résultats de la page
SELECT * FROM produits
ORDER BY nom ASC
LIMIT 20 OFFSET 40;

-- Calcul côté application :
-- total_pages = CEIL(total / page_size)
-- has_next_page = current_page < total_pages
-- has_previous_page = current_page > 1
```

**Exemple complet en pseudo-code :**

```python
def get_paginated_results(page, page_size):
    # 1. Compter le total
    total = db.query("SELECT COUNT(*) FROM produits")[0]

    # 2. Calculer l'offset
    offset = (page - 1) * page_size

    # 3. Récupérer les résultats
    results = db.query(
        "SELECT * FROM produits ORDER BY nom ASC LIMIT %s OFFSET %s",
        (page_size, offset)
    )

    # 4. Calculer les métadonnées
    total_pages = math.ceil(total / page_size)

    return {
        'data': results,
        'pagination': {
            'current_page': page,
            'page_size': page_size,
            'total_results': total,
            'total_pages': total_pages,
            'has_next': page < total_pages,
            'has_previous': page > 1
        }
    }
```

---

## Avantages de LIMIT/OFFSET

### 1. Simplicité d'implémentation

```sql
-- Très facile à comprendre et à implémenter
SELECT * FROM table ORDER BY id LIMIT 20 OFFSET 40;
```

C'est l'approche la plus **simple** et **intuitive** pour la pagination.

### 2. Navigation arbitraire

Vous pouvez **sauter directement** à n'importe quelle page :

```sql
-- Aller directement à la page 100
SELECT * FROM produits
ORDER BY nom
LIMIT 20 OFFSET 1980;  -- (100 - 1) × 20
```

Idéal pour des interfaces avec :
- Sélecteur de page (1, 2, 3, ... 50)
- Lien "Aller à la page X"

### 3. Comptage de pages précis

Avec `COUNT(*)`, vous pouvez calculer le nombre exact de pages :

```sql
SELECT COUNT(*) FROM produits;
-- Total : 5000 produits
-- Page size : 20
-- Nombre de pages : 5000 / 20 = 250 pages
```

### 4. Standard SQL

`LIMIT` et `OFFSET` sont largement supportés par la plupart des SGBD (avec quelques variations syntaxiques).

---

## Limites et problèmes de LIMIT/OFFSET

Malgré sa simplicité, LIMIT/OFFSET présente des **problèmes de performance** et de **cohérence** majeurs.

### 1. Problème de performance : OFFSET élevé

**Le problème :**
PostgreSQL doit **lire et ignorer** toutes les lignes avant l'OFFSET, même si elles ne sont pas retournées.

```sql
-- Page 1 : rapide (0 lignes ignorées)
SELECT * FROM produits ORDER BY id LIMIT 20 OFFSET 0;

-- Page 10 : moins rapide (180 lignes lues et ignorées)
SELECT * FROM produits ORDER BY id LIMIT 20 OFFSET 180;

-- Page 1000 : très lent (19980 lignes lues et ignorées !)
SELECT * FROM produits ORDER BY id LIMIT 20 OFFSET 19980;
```

**Pourquoi ?**
- PostgreSQL doit parcourir toutes les lignes jusqu'à atteindre l'OFFSET
- Plus l'OFFSET est grand, plus c'est lent
- Sur une table de millions de lignes, les dernières pages deviennent **inutilisables**

**Mesure de performance :**

```sql
-- Table avec 1 million de lignes
EXPLAIN ANALYZE
SELECT * FROM large_table
ORDER BY id
LIMIT 20 OFFSET 999980;

-- Résultat possible :
-- Execution Time: 2500 ms  (2.5 secondes juste pour ignorer 999980 lignes !)
```

Comparé à :

```sql
EXPLAIN ANALYZE
SELECT * FROM large_table
ORDER BY id
LIMIT 20 OFFSET 0;

-- Résultat possible :
-- Execution Time: 5 ms
```

**Graphique conceptuel de performance :**
```
Temps (ms)
    ^
    |                                    ╱
2500|                                ╱
    |                            ╱
2000|                        ╱
    |                    ╱
1500|                ╱
    |            ╱
1000|        ╱
    |    ╱
 500|╱
    |
    +----------------------------------------> Numéro de page
    1   10   50  100  500 1000 5000 10000
```

### 2. Problème de cohérence : Données mouvantes

Imaginez ce scénario :
1. Un utilisateur consulte la page 1 (résultats 1-20)
2. Pendant qu'il lit, 5 nouvelles lignes sont insérées au début
3. Il clique sur "Page 2" (résultats 21-40)

**Résultat :** Il voit certains éléments **deux fois** (ceux qui ont "glissé" de la page 1 à la page 2).

**Exemple concret :**

```sql
-- État initial : 100 produits, triés par date_creation DESC

-- Page 1 (produits 1-10)
SELECT * FROM produits ORDER BY date_creation DESC LIMIT 10 OFFSET 0;
-- Retourne : P100, P99, P98, P97, P96, P95, P94, P93, P92, P91

-- [5 nouveaux produits sont créés : P101, P102, P103, P104, P105]

-- Page 2 (produits 11-20, mais avec les nouvelles données)
SELECT * FROM produits ORDER BY date_creation DESC LIMIT 10 OFFSET 10;
-- Retourne : P95, P94, P93, P92, P91, P90, P89, P88, P87, P86
--             ↑    ↑    ↑    ↑    ↑  Déjà vus en page 1 !
```

L'utilisateur voit **P95, P94, P93, P92, P91 deux fois** !

**Problème inverse : éléments manqués**

Si des lignes sont **supprimées** entre deux pages, certains éléments peuvent être **sautés** et jamais vus.

### 3. Problème de cohérence : Tri instable

Si votre tri n'est pas **unique** (plusieurs lignes ont la même valeur de tri), l'ordre peut changer :

```sql
-- Tri par prix uniquement (plusieurs produits à 99€)
SELECT * FROM produits ORDER BY prix LIMIT 10 OFFSET 0;  -- Page 1
SELECT * FROM produits ORDER BY prix LIMIT 10 OFFSET 10; -- Page 2

-- L'ordre des produits à 99€ peut différer entre les deux requêtes
-- Résultat : doublons ou éléments manqués
```

**✅ Solution :** Toujours trier avec une colonne **unique** en second critère :

```sql
-- ✅ Tri stable
SELECT * FROM produits
ORDER BY prix ASC, id ASC  -- id est unique
LIMIT 10 OFFSET 10;
```

### 4. Problème de mémoire : COUNT(*) coûteux

Pour afficher "Page X sur Y", vous devez compter le total :

```sql
SELECT COUNT(*) FROM produits WHERE categorie = 'electronique';
```

Sur de très grandes tables, `COUNT(*)` peut être **très lent** (scan complet de la table).

**Alternatives :**
- Estimer le nombre approximatif (pg_class.reltuples)
- Ne pas afficher le nombre total de pages
- Utiliser uniquement "Précédent" / "Suivant"

---

## Alternatives performantes à LIMIT/OFFSET

Pour de grandes tables ou des paginations fréquentes, des alternatives existent.

### 1. Keyset Pagination (Seek Method)

Au lieu d'utiliser OFFSET, on utilise une **clé** (généralement l'ID ou la colonne de tri) pour savoir où reprendre.

**Principe :**
- Retenir la **dernière valeur** de la page précédente
- Demander les lignes **après** cette valeur

**Exemple avec ID :**

```sql
-- Page 1 : premiers 20 produits
SELECT * FROM produits
WHERE id > 0  -- Depuis le début
ORDER BY id ASC
LIMIT 20;

-- Résultat : IDs 1 à 20
-- On retient : last_id = 20

-- Page 2 : 20 produits suivants
SELECT * FROM produits
WHERE id > 20  -- Après le dernier ID de la page précédente
ORDER BY id ASC
LIMIT 20;

-- Résultat : IDs 21 à 40
-- On retient : last_id = 40

-- Page 3 : 20 produits suivants
SELECT * FROM produits
WHERE id > 40
ORDER BY id ASC
LIMIT 20;
```

**Avantages :**
- ✅ **Performances constantes** : toujours rapide, même pour les dernières pages
- ✅ **Pas de problème de données mouvantes** : les insertions n'affectent pas la pagination
- ✅ **Pas besoin de OFFSET**

**Inconvénients :**
- ❌ Impossible de sauter directement à une page arbitraire (pas de "page 50")
- ❌ Uniquement navigation séquentielle (précédent/suivant)
- ❌ Plus complexe à implémenter

**Exemple avec une colonne de tri :**

```sql
-- Tri par date de création, puis ID
-- Page 1
SELECT * FROM articles
WHERE (date_creation, id) > ('1900-01-01', 0)  -- Depuis le début
ORDER BY date_creation DESC, id DESC
LIMIT 20;

-- Dernière ligne : date_creation='2024-11-15', id=150
-- Page 2
SELECT * FROM articles
WHERE (date_creation, id) < ('2024-11-15', 150)
ORDER BY date_creation DESC, id DESC
LIMIT 20;
```

**⚠️ Syntaxe avec tuple comparison :**
PostgreSQL supporte la comparaison de tuples, très pratique pour keyset pagination.

### 2. Cursors (côté serveur)

Les **cursors** permettent de "garder une position" côté serveur dans un ensemble de résultats.

**Exemple :**

```sql
-- Déclarer un cursor
BEGIN;
DECLARE my_cursor CURSOR FOR
    SELECT * FROM produits ORDER BY id;

-- Récupérer les 20 premières lignes
FETCH 20 FROM my_cursor;

-- Récupérer les 20 suivantes
FETCH 20 FROM my_cursor;

-- Récupérer les 20 suivantes
FETCH 20 FROM my_cursor;

-- Fermer le cursor
CLOSE my_cursor;
COMMIT;
```

**Avantages :**
- ✅ Performances constantes
- ✅ État maintenu côté serveur

**Inconvénients :**
- ❌ Complexe à gérer avec HTTP (stateless)
- ❌ Nécessite de maintenir une connexion ouverte
- ❌ Pas adapté aux applications web modernes

### 3. Pagination avec timestamp

Pour des flux chronologiques (fils d'actualité, logs, etc.) :

```sql
-- Page 1 : les 20 derniers messages
SELECT * FROM messages
WHERE timestamp < NOW()
ORDER BY timestamp DESC
LIMIT 20;

-- Dernier message : timestamp = '2024-11-19 10:30:00'

-- Page 2 : les 20 messages précédents
SELECT * FROM messages
WHERE timestamp < '2024-11-19 10:30:00'
ORDER BY timestamp DESC
LIMIT 20;
```

**Idéal pour :**
- Fils d'actualité (Twitter, Facebook)
- Historique de messages (chat)
- Logs système

### 4. Pagination "infinie" (Infinite Scroll)

Au lieu de pages numérotées, charger automatiquement plus de résultats en scrollant :

```sql
-- Charger les 50 premiers éléments
SELECT * FROM produits ORDER BY id LIMIT 50;

-- Quand l'utilisateur arrive en bas, charger les 50 suivants
SELECT * FROM produits WHERE id > 50 ORDER BY id LIMIT 50;

-- Et ainsi de suite
SELECT * FROM produits WHERE id > 100 ORDER BY id LIMIT 50;
```

**Avantages :**
- ✅ Expérience utilisateur fluide
- ✅ Pas besoin de compter le total
- ✅ Peut utiliser keyset pagination (performant)

---

## Bonnes pratiques et recommandations

### 1. Toujours utiliser ORDER BY avec LIMIT/OFFSET

```sql
-- ❌ Ordre imprévisible
SELECT * FROM produits LIMIT 20;

-- ✅ Ordre déterministe
SELECT * FROM produits ORDER BY id ASC LIMIT 20;
```

### 2. Trier sur une colonne unique

```sql
-- ❌ Risque de doublons/omissions (prix n'est pas unique)
SELECT * FROM produits ORDER BY prix LIMIT 20 OFFSET 20;

-- ✅ Tri stable avec colonne unique en second critère
SELECT * FROM produits ORDER BY prix ASC, id ASC LIMIT 20 OFFSET 20;
```

### 3. Créer des index sur les colonnes de tri

```sql
-- Index pour ORDER BY + pagination
CREATE INDEX idx_produits_prix_id ON produits(prix, id);

-- La requête suivante sera beaucoup plus rapide
SELECT * FROM produits
ORDER BY prix ASC, id ASC
LIMIT 20 OFFSET 1000;
```

### 4. Limiter la valeur maximale d'OFFSET

Pour éviter les abus et les problèmes de performance :

```sql
-- Côté application : limiter les pages accessibles
max_page = 100
max_offset = (max_page - 1) * page_size

if offset > max_offset:
    return error("Page trop élevée, utilisez les filtres de recherche")
```

### 5. Utiliser keyset pagination pour les grandes tables

```sql
-- Pour une table avec des millions de lignes
-- ❌ Lent avec OFFSET élevé
SELECT * FROM logs ORDER BY timestamp DESC LIMIT 100 OFFSET 900000;

-- ✅ Rapide avec keyset
SELECT * FROM logs
WHERE timestamp < :last_timestamp
ORDER BY timestamp DESC
LIMIT 100;
```

### 6. Cacher le nombre total de pages si coûteux

```sql
-- Au lieu de calculer COUNT(*) à chaque fois
-- Utilisez une estimation
SELECT reltuples::BIGINT as estimate
FROM pg_class
WHERE relname = 'produits';

-- Ou ne montrez pas le total
-- Interface : [Précédent] [Suivant] au lieu de "Page 5 sur 1000"
```

### 7. Valider les paramètres d'entrée

```python
# Validation côté application
def paginate(page, page_size):
    # Limites raisonnables
    page = max(1, min(page, 1000))  # Max 1000 pages
    page_size = max(10, min(page_size, 100))  # Entre 10 et 100

    offset = (page - 1) * page_size

    # Requête sécurisée
    return db.query(
        "SELECT * FROM produits ORDER BY id LIMIT %s OFFSET %s",
        (page_size, offset)
    )
```

---

## Comparaison des méthodes de pagination

| Critère | LIMIT/OFFSET | Keyset Pagination | Cursors |
|---------|--------------|-------------------|---------|
| **Simplicité** | ✅ Très simple | ⚠️ Moyenne | ❌ Complexe |
| **Performance page 1** | ✅ Rapide | ✅ Rapide | ✅ Rapide |
| **Performance dernières pages** | ❌ Très lent | ✅ Rapide | ✅ Rapide |
| **Navigation arbitraire** | ✅ Oui (page 50) | ❌ Non | ❌ Non |
| **Données mouvantes** | ❌ Problèmes | ✅ Pas de problème | ✅ Pas de problème |
| **Comptage total** | ✅ Possible | ⚠️ Difficile | ⚠️ Difficile |
| **Stateless (HTTP)** | ✅ Oui | ✅ Oui | ❌ Non |
| **Cas d'usage** | Petites tables, navigation par page | Grandes tables, navigation séquentielle | Applications desktop |

---

## Exemples d'implémentation complète

### Pagination classique (LIMIT/OFFSET)

```sql
-- Fonction PostgreSQL pour pagination
CREATE OR REPLACE FUNCTION get_products_page(
    p_page INTEGER,
    p_page_size INTEGER
)
RETURNS TABLE (
    id INTEGER,
    nom VARCHAR,
    prix NUMERIC,
    total_count BIGINT,
    total_pages INTEGER
) AS $$
DECLARE
    v_offset INTEGER;
    v_total BIGINT;
BEGIN
    -- Calculer l'offset
    v_offset := (p_page - 1) * p_page_size;

    -- Compter le total
    SELECT COUNT(*) INTO v_total FROM produits;

    -- Retourner les résultats paginés avec métadonnées
    RETURN QUERY
    SELECT
        p.id,
        p.nom,
        p.prix,
        v_total as total_count,
        CEIL(v_total::NUMERIC / p_page_size)::INTEGER as total_pages
    FROM produits p
    ORDER BY p.id ASC
    LIMIT p_page_size
    OFFSET v_offset;
END;
$$ LANGUAGE plpgsql;

-- Utilisation
SELECT * FROM get_products_page(3, 20);  -- Page 3, 20 résultats
```

### Keyset pagination

```sql
-- Fonction pour keyset pagination
CREATE OR REPLACE FUNCTION get_products_after(
    p_last_id INTEGER,
    p_page_size INTEGER
)
RETURNS TABLE (
    id INTEGER,
    nom VARCHAR,
    prix NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        p.id,
        p.nom,
        p.prix
    FROM produits p
    WHERE p.id > p_last_id
    ORDER BY p.id ASC
    LIMIT p_page_size;
END;
$$ LANGUAGE plpgsql;

-- Utilisation
-- Page 1 (from start)
SELECT * FROM get_products_after(0, 20);

-- Page 2 (after last ID from page 1)
SELECT * FROM get_products_after(20, 20);

-- Page 3
SELECT * FROM get_products_after(40, 20);
```

---

## Cas d'usage selon le type d'application

### 1. Site e-commerce (catalogue produits)

**Recommandation :** LIMIT/OFFSET pour < 10 000 produits, keyset au-delà

```sql
-- Navigation par page (1, 2, 3, ..., 50)
SELECT * FROM produits
WHERE categorie = 'electronique'
ORDER BY prix ASC, id ASC
LIMIT 20 OFFSET :offset;

-- Avec index approprié
CREATE INDEX idx_produits_cat_prix ON produits(categorie, prix, id);
```

### 2. Réseau social (fil d'actualité)

**Recommandation :** Keyset pagination (infinite scroll)

```sql
-- Charger les nouveaux posts
SELECT * FROM posts
WHERE timestamp < :last_seen_timestamp
ORDER BY timestamp DESC
LIMIT 50;
```

### 3. Tableau de bord analytique

**Recommandation :** LIMIT sans pagination (afficher top/bottom N)

```sql
-- Top 20 des vendeurs
SELECT vendeur, SUM(montant) as total
FROM ventes
GROUP BY vendeur
ORDER BY total DESC
LIMIT 20;

-- Pas besoin de pagination, c'est un classement fixe
```

### 4. Export de données

**Recommandation :** Streaming ou batch processing (pas de pagination)

```sql
-- Utiliser COPY pour exporter massivement
COPY (
    SELECT * FROM commandes
    WHERE date > '2024-01-01'
    ORDER BY id
) TO '/tmp/export.csv' WITH CSV HEADER;
```

### 5. API REST

**Recommandation :** Keyset pagination + links hypermedia

```json
{
  "data": [...],
  "pagination": {
    "next": "/api/products?after_id=120",
    "previous": "/api/products?before_id=100"
  }
}
```

---

## Points clés à retenir

1. **LIMIT restreint le nombre de résultats**
   - Simple et intuitif
   - Toujours utiliser avec ORDER BY

2. **OFFSET saute les N premières lignes**
   - Permet la pagination
   - Performance se dégrade avec des valeurs élevées

3. **Pagination classique : LIMIT/OFFSET**
   - Formule : `OFFSET = (page - 1) × page_size`
   - Simple mais problématique à grande échelle

4. **Problèmes de LIMIT/OFFSET :**
   - Performance dégradée avec OFFSET élevé
   - Doublons/omissions si données mouvantes
   - COUNT(*) coûteux sur grandes tables

5. **Keyset pagination : alternative performante**
   - `WHERE id > last_id` au lieu de OFFSET
   - Performance constante
   - Uniquement navigation séquentielle

6. **Bonnes pratiques :**
   - Toujours trier avec une colonne unique
   - Créer des index sur colonnes de tri
   - Limiter OFFSET maximum
   - Valider les paramètres d'entrée

7. **Choisir la méthode selon le contexte :**
   - LIMIT/OFFSET : petites tables, navigation par page
   - Keyset : grandes tables, navigation séquentielle
   - Cursors : applications desktop/batch
   - Infinite scroll : réseaux sociaux, flux

---

## Conclusion

LIMIT et OFFSET sont des outils puissants pour contrôler le nombre de résultats retournés et implémenter la pagination. Leur simplicité les rend idéaux pour débuter et pour les petites à moyennes tables.

Cependant, pour des applications à grande échelle avec des millions de lignes, les limitations de performance de OFFSET deviennent un problème réel. Dans ces cas, des alternatives comme la **keyset pagination** offrent des performances bien supérieures, au prix d'une complexité accrue.

Le choix de la méthode dépend de :
- La taille de vos tables
- Le type de navigation souhaité (pages numérotées vs séquentielle)
- Les contraintes de performance
- L'expérience utilisateur visée

Gardez à l'esprit qu'il n'y a pas de solution unique : adaptez votre stratégie de pagination au contexte de votre application.

---


⏭️ [DISTINCT et élimination des doublons (DISTINCT ON)](/05-requetes-de-selection/06-distinct-elimination-doublons.md)
