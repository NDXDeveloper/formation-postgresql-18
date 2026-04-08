🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.10. Prepared Statements et Performance (PREPARE, EXECUTE)

## Introduction

Les **Prepared Statements** (requêtes préparées) sont un mécanisme puissant de PostgreSQL qui permet de **préparer une requête une fois** et de l'**exécuter plusieurs fois** avec des paramètres différents, tout en réutilisant le plan d'exécution.

**Analogie** : Imaginez une recette de cuisine. Au lieu de relire toute la recette à chaque fois que vous cuisinez (planification), vous la mémorisez une fois (prepare), puis vous la suivez en changeant juste les ingrédients (paramètres). C'est beaucoup plus rapide !

**Avantages** :
- ⚡ **Performance** : Évite la re-planification des requêtes répétitives  
- 🔒 **Sécurité** : Protection contre les injections SQL  
- 📉 **Réduction de la charge CPU** : Moins de travail pour le planificateur

**Prérequis** : Avoir lu les sections 13.6 (Planificateur) et 13.7 (EXPLAIN ANALYZE).

---

## 1. Comprendre le Cycle de Vie d'une Requête

### 1.1. Requête Ad-hoc (Sans Prepared Statement)

Chaque fois que vous exécutez une requête "classique", PostgreSQL suit ces étapes :

```
1. Parse (Analyse syntaxique)
   ↓
2. Rewrite (Réécriture avec règles/vues)
   ↓
3. Plan (Génération du plan d'exécution)    ← Coûteux !
   ↓
4. Execute (Exécution du plan)
```

**Exemple** :
```sql
-- Exécution 1
SELECT * FROM clients WHERE id = 12345;

-- PostgreSQL fait : Parse → Rewrite → Plan → Execute

-- Exécution 2 (même requête, ID différent)
SELECT * FROM clients WHERE id = 67890;

-- PostgreSQL refait tout : Parse → Rewrite → Plan → Execute
```

**Problème** : Chaque exécution refait la **planification**, même si la requête est structurellement identique !

### 1.2. Avec Prepared Statement

Avec un Prepared Statement, PostgreSQL sépare la préparation de l'exécution :

```
Phase 1 : PREPARE (une seule fois)
   Parse → Rewrite → Plan (optionnel)
   ↓
   Statement préparé stocké en mémoire

Phase 2 : EXECUTE (plusieurs fois)
   Bind parameters → Execute
   (Réutilise le plan ou re-planifie selon stratégie)
```

**Exemple** :
```sql
-- Phase 1 : Préparation (une fois)
PREPARE get_client (INT) AS
    SELECT * FROM clients WHERE id = $1;

-- Phase 2 : Exécutions (multiples)
EXECUTE get_client(12345);   -- Première exécution  
EXECUTE get_client(67890);   -- Réutilise le plan !  
EXECUTE get_client(11111);   -- Réutilise le plan !  
```

**Gain** : Parse et Rewrite ne sont faits qu'**une seule fois** !

---

## 2. Syntaxe : PREPARE et EXECUTE

### 2.1. PREPARE : Créer un Prepared Statement

**Syntaxe** :
```sql
PREPARE nom_statement [(type_param1, type_param2, ...)] AS
    requête_sql;
```

**Exemples** :

#### Exemple 1 : SELECT simple
```sql
PREPARE get_client (INT) AS
    SELECT * FROM clients WHERE id = $1;
```

**Explication** :
- `get_client` : Nom du prepared statement  
- `(INT)` : Type du paramètre (optionnel mais recommandé)  
- `$1` : Placeholder pour le premier paramètre

#### Exemple 2 : Plusieurs paramètres
```sql
PREPARE search_orders (INT, DATE) AS
    SELECT * FROM commandes
    WHERE client_id = $1
      AND date_commande > $2;
```

**Paramètres** :
- `$1` : Premier paramètre (INT = client_id)  
- `$2` : Deuxième paramètre (DATE = date_commande)

#### Exemple 3 : INSERT
```sql
PREPARE insert_client (TEXT, TEXT, INT) AS
    INSERT INTO clients (nom, email, age)
    VALUES ($1, $2, $3)
    RETURNING id;
```

#### Exemple 4 : UPDATE
```sql
PREPARE update_status (TEXT, INT) AS
    UPDATE commandes
    SET statut = $1
    WHERE id = $2;
```

### 2.2. EXECUTE : Exécuter un Prepared Statement

**Syntaxe** :
```sql
EXECUTE nom_statement (valeur1, valeur2, ...);
```

**Exemples** :
```sql
-- Utilisation de get_client
EXECUTE get_client(12345);  
EXECUTE get_client(67890);  

-- Utilisation de search_orders
EXECUTE search_orders(12345, '2024-01-01');  
EXECUTE search_orders(67890, '2024-06-01');  

-- Utilisation de insert_client
EXECUTE insert_client('Dupont', 'dupont@example.com', 35);  
EXECUTE insert_client('Martin', 'martin@example.com', 42);  
```

### 2.3. DEALLOCATE : Supprimer un Prepared Statement

**Syntaxe** :
```sql
DEALLOCATE nom_statement;

-- Ou supprimer tous les prepared statements
DEALLOCATE ALL;
```

**Exemple** :
```sql
DEALLOCATE get_client;
```

**Quand utiliser** :
- Après une série d'exécutions terminée
- Avant de re-préparer avec une nouvelle définition
- En fin de session (automatique de toute façon)

---

## 3. Avantages des Prepared Statements

### 3.1. Performance : Économie de Planification

**Gain mesuré** :

```sql
-- Mesure sans Prepared Statement
\timing on

SELECT * FROM clients WHERE id = 12345;
-- Temps : 0.456 ms (dont ~0.200 ms de planification)

SELECT * FROM clients WHERE id = 67890;
-- Temps : 0.443 ms (dont ~0.198 ms de planification)

-- Total pour 100 exécutions : ~45 ms (dont ~20 ms de planification)
```

**Avec Prepared Statement** :
```sql
PREPARE get_client (INT) AS SELECT * FROM clients WHERE id = $1;

EXECUTE get_client(12345);
-- Première fois : 0.456 ms (planification)

EXECUTE get_client(67890);
-- Deuxième fois : 0.256 ms (pas de re-planification !)

EXECUTE get_client(11111);
-- Troisième fois : 0.252 ms

-- Total pour 100 exécutions : ~26 ms (1× planification + 99× exécution)
```

**Gain** : **~42% plus rapide** (45 ms → 26 ms) !

**Note** : Le gain est plus important pour :
- Requêtes complexes (jointures, agrégations)
- Exécutions très fréquentes (milliers par seconde)
- Requêtes sur tables avec beaucoup d'index

### 3.2. Sécurité : Protection Contre SQL Injection

**Sans Prepared Statement** (❌ Dangereux) :
```python
# Python - Construction de requête par concaténation
user_input = "12345'; DROP TABLE clients; --"

query = f"SELECT * FROM clients WHERE id = {user_input}"
# Résultat : SELECT * FROM clients WHERE id = 12345'; DROP TABLE clients; --
# 💀 Injection SQL !
```

**Avec Prepared Statement** (✅ Sécurisé) :
```python
# Python avec psycopg3
cursor.execute(
    "SELECT * FROM clients WHERE id = %s",  # Placeholder
    (user_input,)  # Paramètre séparé
)
# PostgreSQL traite user_input comme une simple valeur, pas du code SQL
# Pas d'injection possible !
```

**Principe** : Les paramètres sont **escapés automatiquement** et traités comme des données, jamais comme du code.

### 3.3. Réduction de la Charge Serveur

**Métriques** :

| Métrique | Sans Prepared | Avec Prepared | Gain |
|----------|---------------|---------------|------|
| Cycles CPU (planification) | 100% | ~5-10% | 90-95% |
| Temps de réponse moyen | 0.45 ms | 0.26 ms | 42% |
| Requêtes/seconde | 2,200 | 3,800 | +73% |

**Impact sur le serveur** :
- Moins de CPU utilisé par le planificateur
- Plus de requêtes traitées simultanément
- Meilleure scalabilité

---

## 4. Planification : Generic vs Custom Plans

### 4.1. Le Dilemme du Planificateur

PostgreSQL doit faire un choix stratégique :

**Option A : Generic Plan (Plan Générique)**
- Créer un plan d'exécution **unique** pour tous les paramètres
- Avantage : Réutilisé à chaque EXECUTE (très rapide)
- Inconvénient : Peut être suboptimal pour certaines valeurs

**Option B : Custom Plan (Plan Spécifique)**
- Créer un nouveau plan **pour chaque valeur** de paramètre
- Avantage : Plan optimal pour chaque valeur
- Inconvénient : Re-planification à chaque fois (lent)

### 4.2. Stratégie PostgreSQL

PostgreSQL utilise une **stratégie adaptative** :

**Algorithme** :
1. **5 premières exécutions** : Custom Plans (plans spécifiques)  
2. Après 5 exécutions : Calcul du **coût moyen** des custom plans  
3. **6ème exécution** : Création d'un Generic Plan  
4. **Comparaison** : Generic Plan vs coût moyen des custom plans  
5. **Décision** :
   - Si Generic Plan ≤ coût moyen × 1.1 → Utiliser Generic Plan
   - Sinon → Continuer avec Custom Plans

### 4.3. Exemple : Distribution Uniforme

**Scénario** : Table clients avec 100,000 lignes, répartition uniforme par ville (200 villes, 500 clients par ville)

```sql
CREATE INDEX idx_clients_ville ON clients(ville);

PREPARE get_clients_by_city (TEXT) AS
    SELECT * FROM clients WHERE ville = $1;
```

**Exécutions** :
```sql
-- 1ère exécution : Custom Plan
EXECUTE get_clients_by_city('Paris');
-- Plan : Index Scan (coût estimé: 100)

-- 2ème à 5ème : Custom Plans
EXECUTE get_clients_by_city('Lyon');  
EXECUTE get_clients_by_city('Marseille');  
EXECUTE get_clients_by_city('Toulouse');  
EXECUTE get_clients_by_city('Nice');  
-- Coût moyen : 100

-- 6ème exécution : Generic Plan créé
EXECUTE get_clients_by_city('Bordeaux');
-- Generic Plan : Index Scan (coût estimé: 105)
-- 105 ≤ 100 × 1.1 (110) → Generic Plan accepté !

-- 7ème et suivantes : Generic Plan réutilisé
EXECUTE get_clients_by_city('Lille');
-- Réutilise Generic Plan (pas de re-planification)
```

**Résultat** : Generic Plan utilisé (bon choix, distribution uniforme).

### 4.4. Exemple : Distribution Inégale

**Scénario** : Table commandes avec 1 million de lignes, répartition très inégale par statut :
- 'shipped' : 950,000 lignes (95%)  
- 'pending' : 45,000 lignes (4.5%)  
- 'cancelled' : 5,000 lignes (0.5%)

```sql
CREATE INDEX idx_commandes_statut ON commandes(statut);

PREPARE get_orders_by_status (TEXT) AS
    SELECT * FROM commandes WHERE statut = $1;
```

**Exécutions** :
```sql
-- 1ère à 5ème : Custom Plans
EXECUTE get_orders_by_status('pending');    -- Custom Plan : Index Scan (coût: 200)  
EXECUTE get_orders_by_status('cancelled');  -- Custom Plan : Index Scan (coût: 50)  
EXECUTE get_orders_by_status('pending');    -- Custom Plan : Index Scan (coût: 200)  
EXECUTE get_orders_by_status('cancelled');  -- Custom Plan : Index Scan (coût: 50)  
EXECUTE get_orders_by_status('pending');    -- Custom Plan : Index Scan (coût: 200)  
-- Coût moyen : (200 + 50 + 200 + 50 + 200) / 5 = 140

-- 6ème exécution : Generic Plan créé
EXECUTE get_orders_by_status('shipped');
-- Generic Plan : Seq Scan (coût: 25000)
-- 25000 > 140 × 1.1 (154) → Generic Plan REJETÉ !

-- 7ème et suivantes : Custom Plans continuent
EXECUTE get_orders_by_status('pending');
-- Custom Plan : Index Scan (planification à chaque fois)
```

**Résultat** : Custom Plans utilisés (bon choix, distribution inégale).

**Explication** : Pour 'shipped' (95% des données), le meilleur plan est un Seq Scan, mais pour 'pending' et 'cancelled', c'est un Index Scan. Le Generic Plan choisirait Seq Scan (médiane), ce qui serait catastrophique pour 'pending' et 'cancelled'.

### 4.5. Visualiser les Plans

#### Voir quel plan est utilisé

```sql
PREPARE stmt AS SELECT * FROM clients WHERE ville = $1;

-- Exécutions avec EXPLAIN
EXPLAIN EXECUTE stmt('Paris');
```

**Résultat (Custom Plan)** :
```
Index Scan using idx_clients_ville on clients
  Index Cond: (ville = 'Paris'::text)
```

**Après 6 exécutions (Generic Plan)** :
```
Index Scan using idx_clients_ville on clients
  Index Cond: (ville = $1)   ← Utilise $1 (generic)
```

#### Forcer un Generic Plan

```sql
-- Forcer l'utilisation immédiate d'un Generic Plan
SET plan_cache_mode = 'force_generic_plan';

PREPARE stmt AS SELECT * FROM clients WHERE ville = $1;  
EXECUTE stmt('Paris');  -- Utilise Generic Plan dès la 1ère fois  
```

#### Forcer des Custom Plans

```sql
-- Forcer des Custom Plans (jamais de generic)
SET plan_cache_mode = 'force_custom_plan';

PREPARE stmt AS SELECT * FROM clients WHERE ville = $1;  
EXECUTE stmt('Paris');  -- Custom Plan à chaque fois  
```

**Valeurs de plan_cache_mode** :
- `auto` (défaut) : Stratégie adaptative (5 custom → décision)  
- `force_generic_plan` : Toujours generic  
- `force_custom_plan` : Toujours custom

---

## 5. Cas d'Usage Appropriés

### 5.1. Quand Utiliser les Prepared Statements

✅ **Cas d'usage idéaux** :

#### 1. Requêtes OLTP Répétitives

**Exemple** : Application web avec milliers de requêtes identiques par seconde

```sql
-- Connexion au pool
PREPARE get_user_profile (INT) AS
    SELECT * FROM users WHERE id = $1;

-- Exécuté des milliers de fois
EXECUTE get_user_profile(12345);  
EXECUTE get_user_profile(67890);  
...
```

**Gain** : Énorme sur le throughput (requêtes/seconde).

#### 2. APIs RESTful

**Exemple** : Endpoint GET /clients/:id

```python
# Préparation (au démarrage de l'app)
cursor.execute("PREPARE get_client (INT) AS SELECT * FROM clients WHERE id = $1")

# Requêtes HTTP
@app.route('/clients/<int:client_id>')
def get_client(client_id):
    cursor.execute("EXECUTE get_client(%s)", (client_id,))
    return jsonify(cursor.fetchone())
```

#### 3. Batch Operations

**Exemple** : Import de données

```python
# Préparation
cursor.execute("""
    PREPARE insert_order (INT, NUMERIC, DATE) AS
    INSERT INTO commandes (client_id, montant, date_commande)
    VALUES ($1, $2, $3)
""")

# Insertion de 100,000 lignes
for order in orders:
    cursor.execute(
        "EXECUTE insert_order(%s, %s, %s)",
        (order['client_id'], order['montant'], order['date'])
    )
```

**Gain** : 30-50% plus rapide qu'INSERT classiques.

#### 4. Microservices avec Connexions Longues

Si votre service maintient des connexions ouvertes au pool PostgreSQL :

```java
// Connexion persistante
Connection conn = pool.getConnection();

// Préparation (une fois)
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM products WHERE category = ?"
);

// Réutilisation
stmt.setString(1, "Electronics");  
ResultSet rs = stmt.executeQuery();  

stmt.setString(1, "Books");  
rs = stmt.executeQuery();  
```

### 5.2. Quand NE PAS Utiliser les Prepared Statements

❌ **Cas d'usage inappropriés** :

#### 1. Requêtes Ad-hoc / Analytiques

```sql
-- Requête complexe unique
SELECT
    date_trunc('month', created_at) AS month,
    status,
    COUNT(*),
    AVG(amount)
FROM orders  
WHERE created_at > '2024-01-01'  
  AND country IN ('FR', 'BE', 'CH')
GROUP BY 1, 2  
ORDER BY 1 DESC, 2;  
```

**Raison** : Exécutée une seule fois, pas de gain à préparer.

#### 2. Requêtes Dynamiques

```python
# Colonnes et filtres variables
columns = ['id', 'nom', 'email']  # Variable  
filters = {'ville': 'Paris', 'age': 35}  # Variable  

# Construction dynamique nécessaire
query = f"SELECT {', '.join(columns)} FROM clients WHERE ..."
```

**Raison** : Structure de requête changeante, impossible de réutiliser un plan.

#### 3. Connexions Courtes (Connection Pool avec Rotation)

Si chaque requête utilise une nouvelle connexion du pool :

```python
# Chaque requête = nouvelle connexion
with pool.connection() as conn:
    conn.execute("SELECT * FROM clients WHERE id = %s", (12345,))
# Connexion rendue au pool
```

**Raison** : Les prepared statements sont liés à une session. Si la session change, le prepared statement est perdu.

#### 4. Requêtes avec Distribution Très Inégale

Si vous savez que les valeurs de paramètres ont des distributions très différentes :

```sql
-- Mauvais candidat
PREPARE search (TEXT) AS SELECT * FROM logs WHERE message LIKE $1;

-- Exécutions
EXECUTE search('%ERROR%');    -- 0.1% des logs → Index Scan optimal  
EXECUTE search('%INFO%');     -- 99% des logs → Seq Scan optimal  

-- Generic Plan choisirait un compromis suboptimal
```

**Solution** : Utiliser des requêtes séparées ou `plan_cache_mode = 'force_custom_plan'`.

---

## 6. Prepared Statements dans les Drivers

### 6.1. Python : psycopg3

**Prepared Statements automatiques** :

```python
import psycopg

# Connexion
with psycopg.connect("dbname=mabase") as conn:
    with conn.cursor() as cur:
        # Préparation automatique au 6ème execute
        for i in range(10):
            cur.execute(
                "SELECT * FROM clients WHERE id = %s",
                (i,)
            )
            # 1-5 : Custom Plans
            # 6+  : Generic Plan (si approprié)
```

**Préparation explicite** :

```python
# Préparation manuelle
cur.execute("PREPARE get_client (INT) AS SELECT * FROM clients WHERE id = $1")

# Exécution
cur.execute("EXECUTE get_client(%s)", (12345,))
```

**Configuration** :

```python
# Forcer Custom Plans
conn.execute("SET plan_cache_mode = 'force_custom_plan'")

# Ou forcer Generic Plans
conn.execute("SET plan_cache_mode = 'force_generic_plan'")
```

### 6.2. Node.js : node-postgres (pg)

**Prepared Statements automatiques** :

```javascript
const { Client } = require('pg');  
const client = new Client({ database: 'mabase' });  

await client.connect();

// Préparation automatique (nommage basé sur hash de la requête)
for (let i = 0; i < 10; i++) {
    await client.query(
        'SELECT * FROM clients WHERE id = $1',
        [i]
    );
}
```

**Préparation explicite** :

```javascript
// Nom explicite
const query = {
    name: 'get-client',
    text: 'SELECT * FROM clients WHERE id = $1',
    values: [12345]
};

await client.query(query);

// Réutilisation (même nom)
query.values = [67890];  
await client.query(query);  
```

### 6.3. Java : JDBC

**Prepared Statements natifs** :

```java
Connection conn = DriverManager.getConnection("jdbc:postgresql://localhost/mabase");

// Préparation
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM clients WHERE id = ?"
);

// Exécutions
pstmt.setInt(1, 12345);  
ResultSet rs = pstmt.executeQuery();  

pstmt.setInt(1, 67890);  
rs = pstmt.executeQuery();  

// Nettoyage
pstmt.close();
```

### 6.4. Go : pgx

**Prepared Statements automatiques** :

```go
import "github.com/jackc/pgx/v5"

conn, _ := pgx.Connect(context.Background(), "postgres://localhost/mabase")

// Préparation automatique après 5 exécutions
for i := 0; i < 10; i++ {
    var name string
    err := conn.QueryRow(
        context.Background(),
        "SELECT nom FROM clients WHERE id = $1",
        i,
    ).Scan(&name)
}
```

**Préparation explicite** :

```go
// Préparation manuelle
_, err := conn.Prepare(context.Background(), "get-client",
    "SELECT * FROM clients WHERE id = $1")

// Exécution
rows, err := conn.Query(context.Background(), "get-client", 12345)
```

---

## 7. Monitoring et Debugging

### 7.1. Voir les Prepared Statements Actifs

```sql
-- Vue système
SELECT
    name,
    statement,
    parameter_types,
    from_sql
FROM pg_prepared_statements;
```

**Exemple de résultat** :
```
 name        | statement                              | parameter_types | from_sql
-------------|----------------------------------------|-----------------|----------
 get_client  | SELECT * FROM clients WHERE id = $1    | {integer}       | t
 insert_ord  | INSERT INTO commandes VALUES ($1, $2)  | {integer,numeric}| t
```

### 7.2. Statistiques d'Utilisation

```sql
-- Avec pg_stat_statements
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements  
WHERE query LIKE 'EXECUTE%'  
ORDER BY total_exec_time DESC  
LIMIT 10;  
```

### 7.3. Debugging : Plan Choisi

```sql
-- Voir quel plan est utilisé
EXPLAIN (ANALYZE, VERBOSE) EXECUTE get_client(12345);
```

**Indicateurs** :

**Generic Plan** :
```
Index Scan using idx_clients_id on clients
  Index Cond: (id = $1)   ← Paramètre $1 (non évalué)
```

**Custom Plan** :
```
Index Scan using idx_clients_id on clients
  Index Cond: (id = 12345)   ← Valeur concrète
```

### 7.4. Comparer Generic vs Custom

**Test** :

```sql
-- Préparer
PREPARE stmt AS SELECT * FROM commandes WHERE statut = $1;

-- Forcer Custom Plan
SET plan_cache_mode = 'force_custom_plan';  
EXPLAIN ANALYZE EXECUTE stmt('pending');  
-- Noter le temps d'exécution

-- Forcer Generic Plan
SET plan_cache_mode = 'force_generic_plan';  
EXPLAIN ANALYZE EXECUTE stmt('pending');  
-- Comparer le temps

-- Remettre en auto
SET plan_cache_mode = 'auto';
```

---

## 8. Pièges et Limitations

### 8.1. Piège 1 : Prepared Statements Liés à la Session

**Problème** :

```python
# Connexion 1
with pool.connection() as conn1:
    conn1.execute("PREPARE stmt AS SELECT * FROM clients WHERE id = $1")
    conn1.execute("EXECUTE stmt(%s)", (12345,))
# Connexion rendue au pool

# Connexion 2 (différente)
with pool.connection() as conn2:
    conn2.execute("EXECUTE stmt(%s)", (67890,))
    # ❌ ERROR: prepared statement "stmt" does not exist
```

**Solution** : Les drivers gèrent cela automatiquement en préparant dans chaque nouvelle session.

### 8.2. Piège 2 : Mémoire des Prepared Statements

Chaque prepared statement consomme de la mémoire :

```sql
-- Limite par défaut : 250 plans en cache par session
SHOW max_prepared_transactions;
```

**Problème** : Trop de prepared statements différents → Fuite mémoire

**Exemple** :
```python
# ❌ Mauvais : Noms uniques à chaque fois
for i in range(10000):
    cursor.execute(f"PREPARE stmt_{i} AS SELECT * FROM clients WHERE id = $1")
    # 10,000 prepared statements en mémoire !
```

**Solution** : Réutiliser les mêmes noms ou laisser le driver gérer.

### 8.3. Piège 3 : Plans Obsolètes après ANALYZE

**Scénario** :

```sql
-- Préparation avec statistiques obsolètes
PREPARE stmt AS SELECT * FROM clients WHERE ville = $1;  
EXECUTE stmt('Paris');  -- Generic Plan créé basé sur anciennes stats  

-- Mise à jour des données (changement de distribution)
-- Maintenant 95% des clients sont à Paris

ANALYZE clients;

EXECUTE stmt('Paris');  -- Utilise encore l'ancien Generic Plan !
```

**Solution** : Re-préparer après ANALYZE significatif

```sql
DEALLOCATE stmt;  
PREPARE stmt AS SELECT * FROM clients WHERE ville = $1;  
```

Ou simplement fermer/rouvrir la session (prepared statements sont liés à la session).

### 8.4. Piège 4 : Transactions et Prepared Statements

**Attention** : Les prepared statements survivent aux transactions :

```sql
BEGIN;  
PREPARE stmt AS SELECT * FROM clients WHERE id = $1;  
ROLLBACK;  

-- Le prepared statement existe encore !
EXECUTE stmt(12345);  -- ✅ Fonctionne
```

**Raison** : Les prepared statements sont au niveau **session**, pas **transaction**.

### 8.5. Piège 5 : Changement de Schéma

**Scénario** :

```sql
-- Préparation
PREPARE stmt AS SELECT * FROM clients WHERE id = $1;

-- Modification du schéma
ALTER TABLE clients ADD COLUMN new_field VARCHAR(100);

-- Exécution
EXECUTE stmt(12345);
-- ✅ Fonctionne, mais ne voit pas new_field
```

**Solution** : Re-préparer après changements de schéma (ALTER TABLE, DROP/CREATE INDEX).

---

## 9. Bonnes Pratiques

### 9.1. Nommer les Prepared Statements de Manière Significative

❌ **Mauvais** :
```sql
PREPARE s1 AS SELECT ...;  
PREPARE stmt AS SELECT ...;  
PREPARE q AS SELECT ...;  
```

✅ **Bon** :
```sql
PREPARE get_client_by_id AS SELECT ...;  
PREPARE search_orders_by_date AS SELECT ...;  
PREPARE insert_new_user AS INSERT ...;  
```

### 9.2. Utiliser les Drivers Modernes

Les drivers modernes gèrent automatiquement :
- Préparation différée (après 5 exécutions)
- Nettoyage des prepared statements
- Gestion du cycle de vie

**Recommandé** :
- Python : `psycopg3` (pas psycopg2)
- Node.js : `pg` avec préparation automatique
- Java : JDBC natif
- Go : `pgx`

### 9.3. Nettoyer les Prepared Statements Inutilisés

```python
# Après une série d'opérations
cursor.execute("DEALLOCATE ALL")
```

Ou laisser le pool de connexions gérer (recommandé).

### 9.4. Monitorer les Prepared Statements

**Requête de monitoring** :

```sql
-- Nombre de prepared statements par session
SELECT
    pid,
    usename,
    application_name,
    COUNT(*) as nb_prepared
FROM pg_prepared_statements  
JOIN pg_stat_activity USING (pid)  
GROUP BY pid, usename, application_name  
ORDER BY nb_prepared DESC;  
```

**Alerte** : Si nb_prepared > 100 par session → Investiguer (possible fuite).

### 9.5. Tester les Performances

**Benchmark** :

```python
import time

# Sans prepared statement
start = time.time()  
for i in range(1000):  
    cursor.execute("SELECT * FROM clients WHERE id = %s", (i,))
duration_without = time.time() - start

# Avec prepared statement
cursor.execute("PREPARE stmt AS SELECT * FROM clients WHERE id = $1")  
start = time.time()  
for i in range(1000):  
    cursor.execute("EXECUTE stmt(%s)", (i,))
duration_with = time.time() - start

print(f"Sans: {duration_without:.2f}s")  
print(f"Avec: {duration_with:.2f}s")  
print(f"Gain: {(1 - duration_with/duration_without) * 100:.1f}%")  
```

---

## 10. Alternatives et Comparaisons

### 10.1. Prepared Statements vs Query Cache (MySQL)

**MySQL** : Cache des résultats de requêtes

**PostgreSQL Prepared Statements** : Cache des **plans** d'exécution, pas des résultats

**Différences** :

| Aspect | MySQL Query Cache | PG Prepared Statements |
|--------|-------------------|------------------------|
| **Cache** | Résultats | Plans d'exécution |
| **Invalidation** | À chaque UPDATE | Persist jusqu'à DEALLOCATE |
| **Mémoire** | Partagée (global) | Par session |
| **Paramètres** | Non (cache exact) | Oui (paramétrisable) |
| **Scalabilité** | Problèmes (contention) | Excellente |

**Note** : MySQL a déprécié Query Cache (MySQL 8.0+).

### 10.2. Prepared Statements vs Vues Matérialisées

**Vues Matérialisées** : Pré-calcul de résultats

**Prepared Statements** : Pré-planification de requêtes

**Cas d'usage** :
- Vue Matérialisée : Résultats complexes rarement modifiés
- Prepared Statement : Requêtes fréquentes avec paramètres changeants

### 10.3. Prepared Statements vs Fonctions PL/pgSQL

**Fonctions** : Logique métier côté serveur

```sql
CREATE FUNCTION get_client(client_id INT)  
RETURNS TABLE(nom TEXT, email TEXT) AS $$  
BEGIN  
    RETURN QUERY SELECT nom, email FROM clients WHERE id = client_id;
END;
$$ LANGUAGE plpgsql;

-- Utilisation
SELECT * FROM get_client(12345);
```

**Comparaison** :

| Aspect | Prepared Statement | Fonction |
|--------|-------------------|----------|
| **Emplacement** | Session | Base de données |
| **Réutilisation** | Même session | Toutes sessions |
| **Logique** | SQL simple | SQL + logique procédurale |
| **Performance** | Plan en cache | Plan en cache + overhead fonction |

**Recommandation** :
- Prepared Statements : Requêtes SQL simples répétitives
- Fonctions : Logique métier complexe, réutilisée globalement

---

## 11. PostgreSQL 18 : Améliorations des Prepared Statements

### 11.1. Meilleure Heuristique Generic vs Custom

PostgreSQL 18 améliore l'algorithme de décision entre Generic et Custom plans :

**Amélioration** : Prise en compte de la **variance** des coûts

**Avant PG 18** :
```
Décision basée sur : Generic cost ≤ Average custom cost × 1.1
```

**Avec PG 18** :
```
Décision basée sur : Generic cost ≤ Average custom cost × 1.1
                   ET Variance custom costs < seuil
```

**Impact** : Si les custom plans ont des coûts très variables (haute variance), PostgreSQL 18 préfère continuer avec Custom Plans plutôt que risquer un Generic Plan suboptimal.

### 11.2. Statistiques Enrichies

```sql
-- Nouvelle colonne dans pg_prepared_statements (PG 18)
SELECT
    name,
    generic_plans,    -- Nombre d'exécutions avec generic plan
    custom_plans,     -- Nombre d'exécutions avec custom plan
    last_plan_type    -- 'generic' ou 'custom'
FROM pg_prepared_statements;
```

**Utilité** : Monitoring précis de la stratégie choisie.

### 11.3. Optimisation des Paramètres Array

PostgreSQL 18 améliore les prepared statements avec paramètres de type ARRAY :

```sql
PREPARE search_cities (TEXT[]) AS
    SELECT * FROM clients WHERE ville = ANY($1);

-- Meilleures estimations dans PG 18
EXECUTE search_cities(ARRAY['Paris', 'Lyon', 'Marseille']);
```

---

## Points Clés à Retenir

🔑 **Prepared Statements = Préparer une fois, exécuter plusieurs fois** avec paramètres différents.

🔑 **Gain de performance** : 30-50% pour requêtes répétitives (économie de planification).

🔑 **Sécurité** : Protection automatique contre les injections SQL.

🔑 **Syntaxe** : `PREPARE nom (types) AS requête` puis `EXECUTE nom(valeurs)`.

🔑 **Stratégie adaptative** : 5 custom plans → décision generic vs custom basée sur coûts.

🔑 **Generic Plan** : Un plan réutilisé pour toutes les valeurs (rapide mais parfois suboptimal).

🔑 **Custom Plan** : Un plan par valeur (optimal mais re-planification à chaque fois).

🔑 **Cas d'usage idéaux** : OLTP, APIs, batch operations, connexions longues.

🔑 **À éviter** : Requêtes ad-hoc, dynamiques, connexions courtes, distributions très inégales.

🔑 **Drivers modernes** : Gèrent automatiquement les prepared statements (psycopg3, node-postgres, pgx).

🔑 **Monitoring** : `pg_prepared_statements`, `pg_stat_statements`.

🔑 **PostgreSQL 18** : Meilleure heuristique generic vs custom, statistiques enrichies.

---

## Ressources pour Aller Plus Loin

- **Documentation PostgreSQL** : [PREPARE](https://www.postgresql.org/docs/current/sql-prepare.html) et [EXECUTE](https://www.postgresql.org/docs/current/sql-execute.html)  
- **Article approfondi** : [Prepared Statements and Plan Caching](https://www.postgresql.org/docs/current/sql-prepare.html#SQL-PREPARE-NOTES)  
- **Section précédente** : 13.9. Optimisations du planificateur  
- **Section suivante** : 13.11. Maintenance des index (REINDEX, VACUUM FULL)

---


⏭️ [Maintenance des index : REINDEX, VACUUM FULL](/13-indexation-et-optimisation/11-maintenance-des-index.md)
