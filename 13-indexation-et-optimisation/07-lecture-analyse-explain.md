🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.7. Lecture et Analyse d'un EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)

## Introduction

`EXPLAIN` est l'outil le plus puissant pour comprendre et optimiser vos requêtes PostgreSQL. Il révèle le **plan d'exécution** choisi par le planificateur : comment PostgreSQL va exécuter votre requête, étape par étape.

**Analogie** : EXPLAIN est comme demander à votre GPS de vous montrer l'itinéraire complet avant de partir, avec tous les détails : routes empruntées, temps estimé, péages, etc.

Maîtriser EXPLAIN vous permet de :
- Comprendre pourquoi une requête est lente
- Identifier les index manquants
- Détecter les problèmes de statistiques
- Optimiser les jointures complexes
- Anticiper les problèmes de performance

**Prérequis** : Avoir lu les sections 13.1 à 13.6 sur les stratégies de scan, les index et le planificateur.

---

## 1. EXPLAIN de Base : Le Plan d'Exécution

### 1.1. Syntaxe Minimale

```sql
EXPLAIN SELECT * FROM clients WHERE ville = 'Paris';
```

**Résultat** :
```
Seq Scan on clients  (cost=0.00..2123.00 rows=35000 width=50)
  Filter: ((ville)::text = 'Paris'::text)
```

### 1.2. Anatomie d'une Ligne EXPLAIN

Décortiquons cette ligne :

```
Seq Scan on clients  (cost=0.00..2123.00 rows=35000 width=50)
│                          │              │          │
│                          │              │          └─ Largeur moyenne d'une ligne (octets)
│                          │              └─ Nombre de lignes estimé en sortie
│                          └─ Coût startup..coût total
└─ Type d'opération et table cible
```

#### Le Type d'Opération

**Types courants** :
- `Seq Scan` : Scan séquentiel (lecture complète)  
- `Index Scan` : Scan via index  
- `Index Only Scan` : Toutes données dans l'index  
- `Bitmap Heap Scan` : Scan par bitmap  
- `Nested Loop` : Jointure en boucles imbriquées  
- `Hash Join` : Jointure par hash  
- `Merge Join` : Jointure par fusion

#### Le Coût (cost)

```
cost=0.00..2123.00
      ↑      ↑
      │      └─ Coût total pour obtenir TOUTES les lignes
      └─ Coût de démarrage (obtenir la PREMIÈRE ligne)
```

**Unités** : Coût abstrait (pas des secondes), basé sur :
- Lectures disque
- Calculs CPU
- Utilisation mémoire

**Exemples** :
```
cost=0.00..100.00   → Démarrage instantané, coût total faible  
cost=500.00..600.00 → Démarrage coûteux (ex: tri), puis peu de travail supplémentaire  
```

#### Les Lignes (rows)

Nombre de lignes que le planificateur **estime** en sortie de cette opération.

**Critique** : Si l'estimation est très fausse (10× ou 100× d'écart), c'est un signe de statistiques obsolètes.

#### La Largeur (width)

Taille moyenne d'une ligne en octets. Utilisé pour estimer la mémoire nécessaire.

### 1.3. Plans Imbriqués

Les plans complexes sont **hiérarchiques** :

```sql
EXPLAIN  
SELECT c.nom, o.montant  
FROM clients c  
JOIN commandes o ON c.id = o.client_id  
WHERE c.ville = 'Paris';  
```

**Résultat** :
```
Hash Join  (cost=5000.00..15000.00 rows=10000 width=20)
  Hash Cond: (o.client_id = c.id)
  ->  Seq Scan on commandes o  (cost=0.00..8000.00 rows=100000 width=12)
  ->  Hash  (cost=3500.00..3500.00 rows=35000 width=12)
        ->  Seq Scan on clients c  (cost=0.00..3500.00 rows=35000 width=12)
              Filter: ((ville)::text = 'Paris'::text)
```

**Lecture** : De bas en haut et de droite à gauche (ordre d'exécution)

```
1. Seq Scan sur clients (Filter: ville='Paris')  ← Commence ici
   ↓ Produit 35,000 lignes
2. Hash (construction table de hash)
   ↓
3. Seq Scan sur commandes (en parallèle)
   ↓ Produit 100,000 lignes
4. Hash Join (jointure finale)
   ↓ Produit 10,000 lignes
```

**Indentation** : Plus c'est indenté, plus c'est profond dans l'arbre.

---

## 2. EXPLAIN ANALYZE : La Réalité des Exécutions

### 2.1. Différence avec EXPLAIN Simple

```
EXPLAIN          → Montre le PLAN (estimations)  
EXPLAIN ANALYZE  → EXÉCUTE la requête ET montre les RÉSULTATS RÉELS  
```

**⚠️ ATTENTION** : `EXPLAIN ANALYZE` exécute réellement la requête !
- Pour `SELECT` : Pas de problème
- Pour `INSERT/UPDATE/DELETE` : Modifications réelles (utilisez `BEGIN...ROLLBACK`)

### 2.2. Syntaxe

```sql
EXPLAIN ANALYZE  
SELECT * FROM clients WHERE ville = 'Paris';  
```

**Résultat** :
```
Seq Scan on clients  (cost=0.00..2123.00 rows=35000 width=50)
                     (actual time=0.045..23.456 rows=34987 loops=1)
  Filter: ((ville)::text = 'Paris'::text)
  Rows Removed by Filter: 65013
Planning Time: 0.234 ms  
Execution Time: 25.891 ms  
```

### 2.3. Nouvelles Informations

#### actual time

```
actual time=0.045..23.456
            ↑       ↑
            │       └─ Temps pour obtenir TOUTES les lignes (ms)
            └─ Temps pour obtenir la PREMIÈRE ligne (ms)
```

**Unités** : Millisecondes (ms)

#### rows (réel)

```
(cost=... rows=35000 ...) (actual ... rows=34987 ...)
          ↑ Estimation           ↑ Réalité
```

**Comparaison Critique** :
- Si écart < 10% → ✅ Bonnes statistiques
- Si écart 10-50% → ⚠️ À surveiller
- Si écart > 50% → 🔴 Exécuter `ANALYZE`

#### loops

```
loops=1
```

Nombre de fois que cette opération a été exécutée.

**Exemple avec jointure** :
```
Nested Loop  (actual time=0.123..45.678 rows=1000 loops=1)
  ->  Seq Scan on clients  (actual time=0.012..1.234 rows=100 loops=1)
  ->  Index Scan on commandes  (actual time=0.023..0.345 rows=10 loops=100)
                                                                     ↑
                                            Exécuté 100 fois (1 fois par client)
```

**Calcul du coût réel** :
```
Coût total Index Scan = (0.345 ms × 100 loops) = 34.5 ms
```

#### Rows Removed by Filter

```
Rows Removed by Filter: 65013
```

Nombre de lignes **lues mais rejetées** par le filtre.

**Interprétation** :
- 35,000 lignes passent le filtre (rows=34987)
- 65,013 lignes sont rejetées
- Total lu : 100,000 lignes

**Signe de problème** : Si beaucoup de lignes sont rejetées, un index pourrait aider.

#### Planning Time et Execution Time

```
Planning Time: 0.234 ms    ← Temps de génération du plan  
Execution Time: 25.891 ms  ← Temps d'exécution réel  
```

**Total** : 0.234 + 25.891 = 26.125 ms

**Note** : Pour les Prepared Statements, Planning Time = 0 (plan en cache).

---

## 3. EXPLAIN (ANALYZE, BUFFERS) : Analyse I/O

### 3.1. Comprendre les Buffers

PostgreSQL utilise plusieurs niveaux de cache :
- **Shared Buffers** : Cache PostgreSQL en RAM (paramètre `shared_buffers`)  
- **OS Cache** : Cache du système d'exploitation  
- **Disque** : Lecture physique

`BUFFERS` montre combien de blocs (pages de 8 KB) sont lus depuis chaque niveau.

### 3.2. Syntaxe

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT * FROM clients WHERE ville = 'Paris';  
```

**Résultat** :
```
Seq Scan on clients  (cost=0.00..2123.00 rows=35000 width=50)
                     (actual time=0.045..23.456 rows=34987 loops=1)
  Filter: ((ville)::text = 'Paris'::text)
  Rows Removed by Filter: 65013
  Buffers: shared hit=1234 read=150
Planning Time: 0.234 ms  
Execution Time: 25.891 ms  
```

### 3.3. Interprétation des Buffers

```
Buffers: shared hit=1234 read=150
                   ↑           ↑
                   │           └─ Blocs lus depuis le disque (150 × 8KB = 1.2 MB)
                   └─ Blocs trouvés dans le cache (1234 × 8KB = 9.8 MB)
```

#### Métriques des Buffers

**Types de buffers** :
- `shared hit` : Blocs trouvés dans Shared Buffers (RAM PostgreSQL) → **Très rapide**  
- `shared read` : Blocs lus depuis l'OS ou le disque → **Plus lent**  
- `shared dirtied` : Blocs modifiés (écriture)  
- `shared written` : Blocs écrits sur disque  
- `temp read/written` : Fichiers temporaires (sorts, hashes trop gros)  
- `local hit/read` : Buffers locaux à la session

#### Cache Hit Ratio

**Formule** :
```
Cache Hit Ratio = shared hit / (shared hit + shared read) × 100
```

**Exemple** :
```
Buffers: shared hit=1234 read=150

Hit Ratio = 1234 / (1234 + 150) × 100 = 89.2%
```

**Interprétation** :
- **> 99%** : ✅ Excellent (presque tout en cache)  
- **90-99%** : ✅ Bon (données souvent accédées)  
- **< 90%** : ⚠️ Moyen (augmenter `shared_buffers` ou données froides)  
- **< 50%** : 🔴 Problème (cache trop petit ou scan de données rarement utilisées)

#### Fichiers Temporaires

```
Buffers: shared hit=500 temp read=10000 temp written=10000
                        ↑ Fichiers temporaires sur disque !
```

**Cause** : Opérations (tri, hash) trop volumineuses pour `work_mem`.

**Solution** :
```sql
-- Augmenter work_mem pour cette session
SET work_mem = '256MB';
```

### 3.4. Nouveauté PostgreSQL 18 : BUFFERS par Défaut

Dans PostgreSQL 18, avec `EXPLAIN ANALYZE`, les statistiques de buffers sont affichées **automatiquement** sans avoir à spécifier explicitement `BUFFERS`.

**Avant PG 18** :
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;  -- BUFFERS nécessaire
```

**Avec PG 18** :
```sql
EXPLAIN ANALYZE SELECT ...;  -- BUFFERS inclus automatiquement !
```

**Désactiver si besoin** :
```sql
EXPLAIN (ANALYZE, BUFFERS OFF) SELECT ...;
```

---

## 4. EXPLAIN VERBOSE : Détails Complets

### 4.1. Informations Supplémentaires

`VERBOSE` ajoute :
- Noms complets des colonnes (`table.column`)
- Listes explicites des colonnes projetées (Output)
- Détails des expressions et fonctions
- Schémas qualifiés

### 4.2. Syntaxe

```sql
EXPLAIN (VERBOSE)  
SELECT nom, ville FROM clients WHERE ville = 'Paris';  
```

**Résultat** :
```
Seq Scan on public.clients  (cost=0.00..2123.00 rows=35000 width=50)
  Output: nom, ville
  Filter: ((clients.ville)::text = 'Paris'::text)
```

### 4.3. Utilité

**Cas d'usage** :
1. **Vérifier quelles colonnes sont réellement utilisées**
   ```
   Output: id, nom, ville
   ```
   Si trop de colonnes → Optimiser avec `SELECT` sélectif

2. **Comprendre les expressions complexes**
   ```sql
   EXPLAIN VERBOSE
   SELECT UPPER(nom) AS nom_majuscule, ville
   FROM clients WHERE LENGTH(nom) > 5;
   ```

   **Plan** :
   ```
   Seq Scan on public.clients
     Output: upper((nom)::text), ville
     Filter: (length((clients.nom)::text) > 5)
   ```

3. **Déboguer les CTE et sous-requêtes**
   - Montre les noms intermédiaires des CTE
   - Clarifie les références de colonnes

### 4.4. VERBOSE avec ANALYZE

Combinaison puissante :

```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)  
SELECT nom, ville FROM clients WHERE ville = 'Paris';  
```

**Résultat complet** :
```
Seq Scan on public.clients  (cost=0.00..2123.00 rows=35000 width=50)
                            (actual time=0.045..23.456 rows=34987 loops=1)
  Output: nom, ville
  Filter: ((clients.ville)::text = 'Paris'::text)
  Rows Removed by Filter: 65013
  Buffers: shared hit=1234 read=150
Planning:
  Buffers: shared hit=8 read=2
Planning Time: 0.234 ms  
Execution Time: 25.891 ms  
```

---

## 5. FORMAT : Choisir le Format de Sortie

### 5.1. Formats Disponibles

PostgreSQL supporte 4 formats :
- `TEXT` : Format par défaut (lisible)  
- `JSON` : Format structuré (parsing automatique)  
- `XML` : Format XML (rare)  
- `YAML` : Format YAML (rare)

### 5.2. FORMAT JSON

**Syntaxe** :
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)  
SELECT * FROM clients WHERE ville = 'Paris';  
```

**Résultat** :
```json
[
  {
    "Plan": {
      "Node Type": "Seq Scan",
      "Parallel Aware": false,
      "Relation Name": "clients",
      "Alias": "clients",
      "Startup Cost": 0.00,
      "Total Cost": 2123.00,
      "Plan Rows": 35000,
      "Plan Width": 50,
      "Actual Startup Time": 0.045,
      "Actual Total Time": 23.456,
      "Actual Rows": 34987,
      "Actual Loops": 1,
      "Filter": "((ville)::text = 'Paris'::text)",
      "Rows Removed by Filter": 65013,
      "Shared Hit Blocks": 1234,
      "Shared Read Blocks": 150
    },
    "Planning Time": 0.234,
    "Execution Time": 25.891
  }
]
```

### 5.3. Avantages du Format JSON

#### Pour les Outils

- **Parsing automatique** : Plus besoin de parser du texte  
- **Analyse programmatique** : Scripts Python, Node.js, etc.  
- **Visualisation** : Outils comme PEV (Postgres EXPLAIN Visualizer)

#### Exemple d'Utilisation

**Python** :
```python
import json  
import psycopg2  

conn = psycopg2.connect("dbname=mabase")  
cur = conn.cursor()  

cur.execute("EXPLAIN (ANALYZE, FORMAT JSON) SELECT * FROM clients WHERE ville = 'Paris'")  
plan = json.loads(cur.fetchone()[0])  

execution_time = plan[0]["Execution Time"]  
actual_rows = plan[0]["Plan"]["Actual Rows"]  

print(f"Temps d'exécution: {execution_time} ms")  
print(f"Lignes retournées: {actual_rows}")  
```

**Node.js** :
```javascript
const { Client } = require('pg');  
const client = new Client({ database: 'mabase' });  

await client.connect();  
const result = await client.query(  
  "EXPLAIN (ANALYZE, FORMAT JSON) SELECT * FROM clients WHERE ville = 'Paris'"
);

const plan = result.rows[0]["QUERY PLAN"][0];  
console.log(`Execution Time: ${plan["Execution Time"]} ms`);  
console.log(`Actual Rows: ${plan.Plan["Actual Rows"]}`);  
```

### 5.4. Outils de Visualisation

#### PEV (Postgres EXPLAIN Visualizer)

**URL** : https://explain.dalibo.com/

**Utilisation** :
1. Exécuter : `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...`  
2. Copier le JSON  
3. Coller sur https://explain.dalibo.com/  
4. Visualisation interactive !

**Avantages** :
- Graphique en arbre
- Colorisation selon les coûts
- Identification visuelle des goulots d'étranglement
- Pourcentages de temps par nœud

#### pgAdmin

pgAdmin inclut un visualiseur graphique d'EXPLAIN :
1. Onglet "Query Tool"  
2. Bouton "Explain" ou "Explain Analyze"  
3. Visualisation sous forme d'arbre

---

## 6. Lecture de Plans Complexes

### 6.1. Exemple Complet : Jointure Multi-Tables

```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT  
    c.nom AS client,
    p.nom AS produit,
    o.quantite,
    o.montant
FROM commandes o  
JOIN clients c ON o.client_id = c.id  
JOIN produits p ON o.produit_id = p.id  
WHERE c.ville = 'Paris'  
  AND o.date_commande > '2024-01-01'
ORDER BY o.montant DESC  
LIMIT 100;  
```

**Plan d'exécution** :
```
Limit  (cost=25234.56..25234.81 rows=100 width=50)
       (actual time=156.789..156.891 rows=100 loops=1)
  Buffers: shared hit=5678 read=234
  ->  Sort  (cost=25234.56..25484.56 rows=100000 width=50)
            (actual time=156.785..156.823 rows=100 loops=1)
        Sort Key: o.montant DESC
        Sort Method: top-N heapsort  Memory: 32kB
        Buffers: shared hit=5678 read=234
        ->  Hash Join  (cost=8500.00..20000.00 rows=100000 width=50)
                       (actual time=45.123..145.678 rows=95432 loops=1)
              Hash Cond: (o.produit_id = p.id)
              Buffers: shared hit=5678 read=234
              ->  Hash Join  (cost=4000.00..12000.00 rows=100000 width=42)
                             (actual time=23.456..89.012 rows=95432 loops=1)
                    Hash Cond: (o.client_id = c.id)
                    Buffers: shared hit=3456 read=123
                    ->  Seq Scan on commandes o  (cost=0.00..6000.00 rows=150000 width=30)
                                                 (actual time=0.012..34.567 rows=150234 loops=1)
                          Filter: (date_commande > '2024-01-01'::date)
                          Rows Removed by Filter: 50000
                          Buffers: shared hit=2345 read=100
                    ->  Hash  (cost=3500.00..3500.00 rows=35000 width=16)
                              (actual time=23.234..23.234 rows=35123 loops=1)
                          Buckets: 65536  Batches: 1  Memory Usage: 2048kB
                          Buffers: shared hit=1111 read=23
                          ->  Seq Scan on clients c  (cost=0.00..3500.00 rows=35000 width=16)
                                                     (actual time=0.023..18.765 rows=35123 loops=1)
                                Filter: ((ville)::text = 'Paris'::text)
                                Rows Removed by Filter: 64877
                                Buffers: shared hit=1111 read=23
              ->  Hash  (cost=3000.00..3000.00 rows=100000 width=20)
                        (actual time=21.567..21.567 rows=100000 loops=1)
                    Buckets: 131072  Batches: 1  Memory Usage: 6144kB
                    Buffers: shared hit=2222 read=111
                    ->  Seq Scan on produits p  (cost=0.00..3000.00 rows=100000 width=20)
                                                (actual time=0.008..12.345 rows=100000 loops=1)
                          Buffers: shared hit=2222 read=111
Planning:
  Buffers: shared hit=12
Planning Time: 1.234 ms  
Execution Time: 157.234 ms  
```

### 6.2. Analyse Pas à Pas

#### Étape 1 : Identifier l'Ordre d'Exécution

**Ordre** : De bas en haut, de droite à gauche (profondeur d'abord).

```
1. Seq Scan sur clients (Filter: ville='Paris')         → 35,123 lignes
2. Hash de clients                                       → Table de hash
3. Seq Scan sur commandes (Filter: date > '2024-01-01') → 150,234 lignes
4. Hash Join (clients × commandes)                       → 95,432 lignes
5. Seq Scan sur produits                                 → 100,000 lignes
6. Hash de produits                                      → Table de hash
7. Hash Join (résultat × produits)                       → 95,432 lignes
8. Sort (ORDER BY montant DESC)                          → Tri des 95,432 lignes
9. Limit 100                                             → Seulement 100 lignes
```

#### Étape 2 : Identifier les Goulots d'Étranglement

**Temps par opération** :

| Opération | Temps (ms) | % du total |
|-----------|------------|------------|
| Seq Scan commandes | 34.567 | 22% |
| Hash Join 1 | 89.012 - 34.567 = 54.445 | 35% |
| Hash Join 2 | 145.678 - 89.012 = 56.666 | 36% |
| Sort | 156.785 - 145.678 = 11.107 | 7% |
| **Total** | **157.234** | **100%** |

**Goulots identifiés** :
1. **Hash Join 2** (36%) : Jointure avec produits  
2. **Hash Join 1** (35%) : Jointure avec clients  
3. **Seq Scan commandes** (22%) : Lecture de la table

#### Étape 3 : Analyser les Estimations vs Réalité

| Nœud | Estimation | Réel | Écart |
|------|------------|------|-------|
| Clients (Filter Paris) | 35,000 | 35,123 | ✅ 0.3% |
| Commandes (Filter date) | 150,000 | 150,234 | ✅ 0.15% |
| Hash Join 1 | 100,000 | 95,432 | ✅ 4.6% |
| Hash Join 2 | 100,000 | 95,432 | ✅ 4.6% |

→ ✅ Excellentes estimations ! Statistiques à jour.

#### Étape 4 : Analyser les Buffers

**Cache Hit Ratio Global** :
```
Total shared hit = 5678  
Total shared read = 234  

Hit Ratio = 5678 / (5678 + 234) × 100 = 96.0%
```

→ ✅ Excellent ! Presque tout est en cache.

**Par table** :
- Commandes : hit=2345 read=100 → 95.9% hit ratio
- Clients : hit=1111 read=23 → 98.0% hit ratio
- Produits : hit=2222 read=111 → 95.2% hit ratio

#### Étape 5 : Identifier les Optimisations Possibles

**Optimisation 1** : Index sur `commandes.date_commande`

```
Seq Scan on commandes (Filter: date > '2024-01-01')  
Rows Removed by Filter: 50,000  
```

25% des lignes rejetées → Index bénéfique !

```sql
CREATE INDEX idx_commandes_date ON commandes(date_commande);
```

**Optimisation 2** : Index partiel sur `clients.ville`

```
Seq Scan on clients (Filter: ville='Paris')  
Rows Removed by Filter: 64,877  
```

35% des clients à Paris → Index partiel intéressant :

```sql
CREATE INDEX idx_clients_paris ON clients(id) WHERE ville = 'Paris';
```

**Optimisation 3** : Sort optimisé avec LIMIT

```
Sort Method: top-N heapsort  Memory: 32kB
```

Le planificateur utilise déjà l'algorithme optimal (top-N heapsort) !
→ ✅ Rien à faire ici.

---

## 7. Patterns Courants et Interprétation

### 7.1. Sequential Scan

```
Seq Scan on large_table  (cost=0.00..250000.00 rows=1000000 width=100)
                         (actual time=0.012..2345.678 rows=1000000 loops=1)
  Buffers: shared hit=50000 read=10000
```

**Interprétation** :
- Lecture complète de la table
- `shared read=10000` → 10,000 blocs (80 MB) lus depuis le disque

**Bon signe si** :
- Requête récupère >10% de la table
- Pas d'index disponible
- `correlation` élevée pour un range scan

**Mauvais signe si** :
- Requête récupère <1% de la table
- Index existe mais non utilisé → Vérifier statistiques

### 7.2. Index Scan

```
Index Scan using idx_client_id on orders  (cost=0.43..234.56 rows=100 width=50)
                                          (actual time=0.045..1.234 rows=98 loops=1)
  Index Cond: (client_id = 12345)
  Buffers: shared hit=15 read=3
```

**Interprétation** :
- Utilisation efficace de l'index
- 15 blocs en cache, 3 lus depuis le disque
- Estimation vs réel : 100 vs 98 → ✅ Excellent

**Bon signe si** :
- `shared hit` >> `shared read` (cache chaud)
- Estimation proche de la réalité

### 7.3. Index Only Scan

```
Index Only Scan using idx_covering on orders  (cost=0.43..123.45 rows=100 width=12)
                                              (actual time=0.023..0.567 rows=98 loops=1)
  Index Cond: (client_id = 12345)
  Heap Fetches: 0
  Buffers: shared hit=10
```

**Interprétation** :
- **Heap Fetches: 0** → Données entièrement dans l'index ! ✅
- Aucun accès à la table → Ultra rapide
- Seulement 10 blocs d'index lus

**Mauvais signe si** :
```
Heap Fetches: 500
```
→ Visibility Map obsolète → Exécuter `VACUUM`

### 7.4. Bitmap Heap Scan

```
Bitmap Heap Scan on orders  (cost=1234.56..5678.90 rows=50000 width=50)
                            (actual time=23.456..89.012 rows=49876 loops=1)
  Recheck Cond: (client_id >= 10000 AND client_id <= 15000)
  Heap Blocks: exact=5000
  Buffers: shared hit=5100 read=50
  ->  Bitmap Index Scan on idx_client_id  (cost=0.00..1222.06 rows=50000 width=0)
                                          (actual time=22.345..22.345 rows=49876 loops=1)
        Index Cond: (client_id >= 10000 AND client_id <= 15000)
        Buffers: shared hit=100 read=5
```

**Interprétation** :
- 2 phases : Construction bitmap (22 ms) puis lecture heap (67 ms)
- `Heap Blocks: exact=5000` → 5000 blocs différents à lire
- Recheck nécessaire car bitmap travaille au niveau bloc

**Bon signe si** :
- Nombre de `Heap Blocks` << nombre total de blocs de la table
- Alternative efficace entre Index Scan et Seq Scan

### 7.5. Hash Join

```
Hash Join  (cost=15000.00..50000.00 rows=100000 width=50)
           (actual time=45.123..234.567 rows=98765 loops=1)
  Hash Cond: (o.client_id = c.id)
  Buffers: shared hit=5000 read=500
  ->  Seq Scan on orders o  (cost=0.00..30000.00 rows=1000000 width=40)
                            (actual time=0.012..123.456 rows=1000000 loops=1)
        Buffers: shared hit=4000 read=400
  ->  Hash  (cost=12000.00..12000.00 rows=100000 width=20)
            (actual time=44.567..44.567 rows=98765 loops=1)
        Buckets: 131072  Batches: 1  Memory Usage: 6144kB
        Buffers: shared hit=1000 read=100
        ->  Seq Scan on clients c  (cost=0.00..12000.00 rows=100000 width=20)
                                   (actual time=0.008..23.456 rows=98765 loops=1)
              Buffers: shared hit=1000 read=100
```

**Interprétation** :
- Phase 1 : Construction de la table de hash (clients) → 44 ms
- Phase 2 : Scan de orders et probe dans la hash table → 190 ms
- `Batches: 1` → Tout tient en mémoire (work_mem suffisant) ✅  
- `Memory Usage: 6144kB` → 6 MB utilisés

**Mauvais signe si** :
```
Batches: 4  
Buffers: temp read=10000 temp written=10000  
```
→ Hash table trop grosse, déborde sur disque (fichiers temporaires) → Augmenter `work_mem`

### 7.6. Nested Loop

```
Nested Loop  (cost=0.43..5678.90 rows=1000 width=50)
             (actual time=0.123..45.678 rows=987 loops=1)
  Buffers: shared hit=3000 read=50
  ->  Index Scan using idx_clients_ville on clients c  (cost=0.43..123.45 rows=100 width=20)
                                                       (actual time=0.045..1.234 rows=98 loops=1)
        Index Cond: (ville = 'Paris')
        Buffers: shared hit=50 read=5
  ->  Index Scan using idx_orders_client on orders o  (cost=0.43..55.34 rows=10 width=40)
                                                      (actual time=0.023..0.456 rows=10 loops=98)
        Index Cond: (client_id = c.id)
        Buffers: shared hit=2950 read=45
```

**Interprétation** :
- Boucle externe : 98 clients à Paris
- Boucle interne : Pour CHAQUE client, recherche ses commandes (loops=98)
- Total : 98 × 10 = 980 lignes (proche de actual rows=987) ✅

**Bon signe si** :
- Outer side (table externe) est petit (< 1000 lignes)
- Inner side utilise un index
- Estimation vs réel cohérent

**Mauvais signe si** :
```
->  Seq Scan on orders o  (actual time=0.012..234.567 rows=10 loops=10000)
                                                               ↑
                                                    10,000 itérations !
```
→ 10,000 Seq Scans → Catastrophique ! Ajouter un index.

### 7.7. Sort

```
Sort  (cost=25000.00..27500.00 rows=100000 width=50)
      (actual time=123.456..145.678 rows=100000 loops=1)
  Sort Key: created_at DESC
  Sort Method: external merge  Disk: 8192kB
  Buffers: shared hit=3000 read=500, temp read=1024 temp written=1024
```

**Interprétation** :
- Tri de 100,000 lignes
- `Sort Method: external merge` → Tri déborde sur disque 🔴  
- `Disk: 8192kB` → 8 MB de fichiers temporaires

**Sort Methods** :
- `quicksort  Memory: 2048kB` → ✅ Tri en mémoire (rapide)  
- `top-N heapsort  Memory: 128kB` → ✅ Tri optimisé pour LIMIT  
- `external merge  Disk: 8192kB` → 🔴 Tri sur disque (lent)

**Solution pour external merge** :
```sql
SET work_mem = '64MB';  -- Augmenter pour tenir en mémoire
```

---

## 8. Cas Pratiques de Débogage

### 8.1. Cas 1 : Requête Lente Sans Raison Apparente

**Requête** :
```sql
SELECT * FROM orders WHERE created_at > '2024-01-01';
-- Temps : 5 secondes (devrait être <100ms)
```

**Diagnostic** :
```sql
EXPLAIN (ANALYZE, BUFFERS)  
SELECT * FROM orders WHERE created_at > '2024-01-01';  
```

**Plan** :
```
Seq Scan on orders  (cost=0.00..250000.00 rows=1000000 width=100)
                    (actual time=0.012..4567.890 rows=5000 loops=1)
  Filter: (created_at > '2024-01-01')
  Rows Removed by Filter: 995000
  Buffers: shared hit=120000
```

**Problème identifié** :
- 995,000 lignes rejetées (99.5%)
- Seq Scan alors que seulement 0.5% des lignes correspondent

**Solution** :
```sql
CREATE INDEX idx_orders_date ON orders(created_at);  
ANALYZE orders;  
```

**Résultat après index** :
```
Index Scan using idx_orders_date on orders  (cost=0.43..234.56 rows=5000 width=100)
                                            (actual time=0.045..12.345 rows=5000 loops=1)
  Index Cond: (created_at > '2024-01-01')
  Buffers: shared hit=150 read=20
```

Temps : **12 ms** au lieu de 5000 ms → **Gain de 417×** ! 🚀

### 8.2. Cas 2 : Estimations Très Fausses

**Requête** :
```sql
SELECT * FROM products WHERE category = 'Electronics';
```

**Plan** :
```
Seq Scan on products  (cost=0.00..5000.00 rows=100 width=50)
                      (actual time=0.012..234.567 rows=500000 loops=1)
  Filter: (category = 'Electronics')
  Rows Removed by Filter: 50000
```

**Problème** :
- **Estimation** : 100 lignes (0.02%)  
- **Réalité** : 500,000 lignes (91%)  
- **Écart** : 5000× ! 🔴

**Cause** : Statistiques obsolètes (avant, Electronics était rare)

**Solution** :
```sql
ANALYZE products;
```

**Nouveau plan** :
```
Seq Scan on products  (cost=0.00..5000.00 rows=500000 width=50)
                      (actual time=0.012..234.567 rows=500000 loops=1)
  Filter: (category = 'Electronics')
  Rows Removed by Filter: 50000
```

Estimation corrigée (500,000) ! Le planificateur sait maintenant que Seq Scan est approprié.

### 8.3. Cas 3 : Fichiers Temporaires Excessifs

**Requête** :
```sql
SELECT DISTINCT user_id FROM events ORDER BY user_id;
```

**Plan** :
```
Unique  (cost=500000.00..550000.00 rows=1000000 width=4)
        (actual time=5678.901..8901.234 rows=1000000 loops=1)
  Buffers: shared hit=50000, temp read=50000 temp written=50000
  ->  Sort  (cost=500000.00..525000.00 rows=10000000 width=4)
            (actual time=5678.890..7890.123 rows=10000000 loops=1)
        Sort Key: user_id
        Sort Method: external merge  Disk: 400000kB
        Buffers: shared hit=50000, temp read=50000 temp written=50000
```

**Problème** :
- `Disk: 400000kB` → **400 MB** de fichiers temporaires ! 🔴
- Sort Method: external merge → Tri sur disque (très lent)

**Solution** :
```sql
-- Augmenter work_mem pour cette requête
SET work_mem = '512MB';

-- Ou globalement
ALTER SYSTEM SET work_mem = '256MB';  
SELECT pg_reload_conf();  
```

**Alternative** : Créer un index
```sql
CREATE INDEX idx_events_user ON events(user_id);
```

**Nouveau plan (avec index)** :
```
Index Only Scan using idx_events_user on events
  (cost=0.43..250000.00 rows=1000000 width=4)
  (actual time=0.045..123.456 rows=1000000 loops=1)
  Heap Fetches: 0
  Buffers: shared hit=10000
```

Plus besoin de tri (index déjà trié) ! Temps divisé par 100.

---

## 9. Bonnes Pratiques

### 9.1. Workflow d'Analyse

1. **Commencer simple** : `EXPLAIN ANALYZE`  
2. **Ajouter BUFFERS** : `EXPLAIN (ANALYZE, BUFFERS)`  
3. **Si nécessaire, VERBOSE** : Pour colonnes et expressions  
4. **Comparer estimations vs réalité** : Écart > 50% → ANALYZE  
5. **Identifier les goulots** : Nœuds avec le plus de temps  
6. **Optimiser progressivement** : Une amélioration à la fois

### 9.2. Checklist de Diagnostic

- [ ] Estimations cohérentes ? (< 10% d'écart)  
- [ ] Cache Hit Ratio > 95% ?  
- [ ] Pas de fichiers temporaires ?  
- [ ] Index utilisés quand approprié ?  
- [ ] Pas de Nested Loop avec outer side > 1000 lignes ?  
- [ ] Sorts en mémoire (quicksort ou top-N heapsort) ?  
- [ ] Heap Fetches = 0 pour Index Only Scans ?

### 9.3. Commandes Utiles

```sql
-- Analyse complète
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS, TIMING) SELECT ...;

-- Format JSON pour outils
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;

-- Désactiver temporairement l'exécution (juste le plan)
BEGIN;  
EXPLAIN ANALYZE INSERT/UPDATE/DELETE ...;  
ROLLBACK;  

-- Comparer plans avec différentes configurations
BEGIN;  
SET enable_seqscan = off;  
EXPLAIN ANALYZE SELECT ...;  
ROLLBACK;  
```

### 9.4. Ce qu'il Faut Éviter

❌ **Ne pas exécuter EXPLAIN ANALYZE sur des UPDATE/DELETE en production sans BEGIN...ROLLBACK**

❌ **Ne pas ignorer les estimations fausses** (> 50% d'écart)

❌ **Ne pas sur-optimiser** : Si une requête prend 5ms, pas besoin de descendre à 2ms

❌ **Ne pas créer des index sans analyser EXPLAIN** : Vérifier d'abord qu'ils seront utilisés

---

## 10. Outils Complémentaires

### 10.1. auto_explain

Extension qui log automatiquement les plans des requêtes lentes.

**Configuration** (dans `postgresql.conf`) :
```ini
shared_preload_libraries = 'auto_explain'  
auto_explain.log_min_duration = 1000  # Log si > 1 seconde  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
auto_explain.log_timing = on  
auto_explain.log_nested_statements = on  
```

**Redémarrage nécessaire** :
```bash
sudo systemctl restart postgresql
```

**Utilité** : Capture automatiquement les requêtes lentes en production.

### 10.2. pg_stat_statements

Extension qui collecte des statistiques agrégées sur toutes les requêtes.

```sql
CREATE EXTENSION pg_stat_statements;

-- Top 10 requêtes les plus lentes (temps total)
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements  
ORDER BY total_exec_time DESC  
LIMIT 10;  
```

**Combinaison avec EXPLAIN** :
1. Identifier requêtes lentes avec `pg_stat_statements`  
2. Copier la requête  
3. Analyser avec `EXPLAIN ANALYZE`

### 10.3. pgBadger

Outil d'analyse de logs PostgreSQL.

**Installation** :
```bash
# Debian/Ubuntu
sudo apt install pgbadger

# Configuration PostgreSQL pour logs détaillés
log_min_duration_statement = 0  
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '  
log_checkpoints = on  
log_connections = on  
log_disconnections = on  
log_lock_waits = on  
log_temp_files = 0  
```

**Génération du rapport** :
```bash
pgbadger /var/log/postgresql/postgresql-*.log -o report.html
```

**Contenu** :
- Top requêtes lentes
- Graphiques de charge
- Analyse des locks
- Checkpoints, connexions, etc.

---

## Points Clés à Retenir

🔑 **EXPLAIN = Outil indispensable** : Comprendre et optimiser toutes vos requêtes.

🔑 **EXPLAIN vs EXPLAIN ANALYZE** : Le premier estime, le second exécute et mesure.

🔑 **BUFFERS révèle les I/O** : Cache Hit Ratio > 95% = bon, fichiers temp = problème.

🔑 **Comparer estimations vs réalité** : Écart > 50% → Exécuter `ANALYZE`.

🔑 **Identifier les goulots** : Nœud avec le plus de temps = priorité d'optimisation.

🔑 **Loops** : Multiplier le temps par le nombre de loops pour le coût réel.

🔑 **Format JSON** : Pour automatisation et visualisation (PEV, scripts).

🔑 **VERBOSE** : Utile pour comprendre les colonnes et expressions complexes.

🔑 **PostgreSQL 18** : BUFFERS inclus automatiquement avec ANALYZE.

🔑 **auto_explain en production** : Capture automatique des requêtes lentes.

---

## Ressources pour Aller Plus Loin

- **Documentation PostgreSQL** : [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)  
- **Visualiseur PEV** : https://explain.dalibo.com/  
- **Section précédente** : 13.6. Le planificateur de requêtes et les statistiques  
- **Section suivante** : 13.8. Nouveauté PG 18 : Améliorations d'EXPLAIN

---


⏭️ [Nouveauté PG 18 : Améliorations d'EXPLAIN avec affichage automatique des buffers](/13-indexation-et-optimisation/08-ameliorations-explain-pg18.md)
