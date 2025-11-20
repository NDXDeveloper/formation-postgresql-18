🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.6. Advisory Locks : Verrouillage applicatif personnalisé

## Introduction : Au-delà des verrous automatiques

Dans les sections précédentes, nous avons découvert les verrous automatiques que PostgreSQL pose lorsque vous modifiez des données (UPDATE, DELETE, etc.). Ces verrous protègent l'**intégrité des données** au niveau de la base.

Mais imaginez ces situations :

- 🔧 Vous voulez empêcher deux workers de traiter le **même job** en même temps
- 📊 Vous voulez qu'un seul processus génère un **rapport mensuel** à la fois
- 🔄 Vous voulez coordonner des **tâches distribuées** entre plusieurs serveurs
- 🎯 Vous voulez implémenter un **mutex** (exclusion mutuelle) pour une ressource métier

Pour ces cas, les verrous automatiques de PostgreSQL ne suffisent pas. Vous avez besoin de **verrous applicatifs** : les **Advisory Locks**.

**Définition** : Un Advisory Lock est un verrou que **vous** créez et gérez manuellement dans votre application. PostgreSQL fournit le mécanisme, mais c'est à vous de décider quand le prendre et le libérer.

**Analogie** : Les verrous automatiques sont comme les feux de circulation (automatiques, imposés). Les Advisory Locks sont comme réserver une salle de réunion (volontaire, vous décidez quand et pourquoi).

---

## Advisory Locks vs Verrous normaux

### Tableau comparatif

| Caractéristique | Verrous normaux | Advisory Locks |
|----------------|----------------|----------------|
| **Déclenchement** | Automatique (UPDATE, DELETE...) | Manuel (vous appelez une fonction) |
| **Objet protégé** | Tables, lignes physiques | N'importe quoi (concept métier) |
| **PostgreSQL vérifie** | Oui, automatiquement | Non, c'est à vous |
| **Usage** | Intégrité des données | Coordination applicative |
| **Exemple** | Empêcher deux UPDATE sur la même ligne | Empêcher deux jobs identiques |

### Différence fondamentale

**Verrous normaux** :

```sql
BEGIN;
UPDATE produits SET stock = stock - 1 WHERE id = 123;
-- PostgreSQL pose AUTOMATIQUEMENT un verrou sur cette ligne
-- Toute autre transaction doit attendre pour modifier cette ligne
COMMIT;
```

**Advisory Locks** :

```sql
BEGIN;
-- Vous demandez EXPLICITEMENT un verrou sur "l'ID 123"
SELECT pg_advisory_lock(123);
-- PostgreSQL ne sait PAS ce que représente "123"
-- C'est à VOUS de décider que "123" = votre ressource métier
-- D'autres transactions peuvent toujours modifier la table produits
-- Mais si elles respectent le protocole, elles essaieront aussi pg_advisory_lock(123)
COMMIT;
```

**Point clé** : Les Advisory Locks sont "advisory" (consultatifs) car PostgreSQL ne les impose pas. Si votre code ne vérifie pas le verrou, rien n'empêchera l'opération. C'est une **convention** que votre application doit respecter.

---

## Les types d'Advisory Locks

PostgreSQL propose deux catégories d'Advisory Locks :

### 1. Advisory Locks de session (Session-level)

- Actifs pendant toute la **connexion** (session)
- Persistent même après un COMMIT ou ROLLBACK
- Doivent être **explicitement libérés** avec `pg_advisory_unlock`
- Risque : oublier de libérer = verrou éternel jusqu'à la fermeture de la connexion

### 2. Advisory Locks transactionnels (Transaction-level)

- Actifs uniquement pendant la **transaction** en cours
- Libérés **automatiquement** au COMMIT ou ROLLBACK
- Plus sûrs car pas de risque d'oubli
- Recommandés dans la plupart des cas

---

## Fonctions PostgreSQL pour les Advisory Locks

### Vue d'ensemble des fonctions

PostgreSQL fournit plusieurs fonctions pour gérer les Advisory Locks :

| Fonction | Type | Comportement | Retour |
|----------|------|--------------|--------|
| `pg_advisory_lock(key)` | Session | Bloquant (attend) | void |
| `pg_try_advisory_lock(key)` | Session | Non-bloquant | boolean |
| `pg_advisory_unlock(key)` | Session | Libère | boolean |
| `pg_advisory_xact_lock(key)` | Transaction | Bloquant | void |
| `pg_try_advisory_xact_lock(key)` | Transaction | Non-bloquant | boolean |
| `pg_advisory_lock_shared(key)` | Session | Bloquant partagé | void |
| `pg_try_advisory_lock_shared(key)` | Session | Non-bloquant partagé | boolean |
| `pg_advisory_xact_lock_shared(key)` | Transaction | Bloquant partagé | void |

**Note** : Toutes ces fonctions existent aussi en version à deux paramètres : `pg_advisory_lock(key1, key2)` pour utiliser deux entiers au lieu d'un seul bigint.

---

## Advisory Locks de session : Détails et exemples

### 1. pg_advisory_lock() : Le verrou bloquant

**Syntaxe** :
```sql
SELECT pg_advisory_lock(key bigint);
SELECT pg_advisory_lock(key1 integer, key2 integer);
```

**Comportement** :
- Si le verrou est **disponible** : l'obtient immédiatement
- Si le verrou est **déjà pris** : **attend** qu'il soit libéré (bloquant)
- Reste actif jusqu'à ce que vous appeliez `pg_advisory_unlock()`

**Exemple simple** :

```sql
-- Session 1
SELECT pg_advisory_lock(42);
-- Verrou obtenu immédiatement ✅

-- [Faire des opérations critiques]
SELECT pg_sleep(10);  -- Simule un traitement de 10 secondes

-- Libérer
SELECT pg_advisory_unlock(42);
```

```sql
-- Session 2 (en parallèle, essaie le même verrou)
SELECT pg_advisory_lock(42);
-- ⏳ BLOQUÉ ! Attend que Session 1 libère
-- Après 10 secondes, Session 1 libère
-- ✅ Obtient enfin le verrou
```

### 2. pg_try_advisory_lock() : Le verrou non-bloquant

**Syntaxe** :
```sql
SELECT pg_try_advisory_lock(key bigint) RETURNS boolean;
```

**Comportement** :
- Si disponible : l'obtient et retourne **true**
- Si déjà pris : retourne immédiatement **false** (n'attend pas)

**Exemple** :

```sql
-- Session 1
SELECT pg_try_advisory_lock(42);
-- Résultat : true (obtenu)

-- Session 2 (en parallèle)
SELECT pg_try_advisory_lock(42);
-- Résultat : false (déjà pris, n'attend pas)

-- Session 2 peut réagir
DO $$
BEGIN
    IF pg_try_advisory_lock(42) THEN
        RAISE NOTICE 'Verrou obtenu, je traite';
        -- [Traitement]
        PERFORM pg_advisory_unlock(42);
    ELSE
        RAISE NOTICE 'Verrou déjà pris, je passe mon tour';
    END IF;
END $$;
```

### 3. pg_advisory_unlock() : Libération

**Syntaxe** :
```sql
SELECT pg_advisory_unlock(key bigint) RETURNS boolean;
```

**Retour** :
- **true** : Verrou libéré avec succès
- **false** : Vous ne déteniez pas ce verrou (erreur)

**Important** : Vous devez libérer **exactement** le nombre de fois que vous avez acquis le verrou.

```sql
-- Acquérir deux fois
SELECT pg_advisory_lock(42);
SELECT pg_advisory_lock(42);  -- Même session, même verrou

-- Libérer deux fois
SELECT pg_advisory_unlock(42);  -- true
SELECT pg_advisory_unlock(42);  -- true

-- Libérer une troisième fois
SELECT pg_advisory_unlock(42);  -- false (plus détenu)
```

### 4. Verrous partagés (Shared)

Les verrous partagés permettent à **plusieurs sessions** d'obtenir le même verrou, mais bloquent les verrous **exclusifs**.

**Fonctions** :
- `pg_advisory_lock_shared(key)` : Obtenir un verrou partagé
- `pg_advisory_unlock_shared(key)` : Libérer un verrou partagé

**Exemple** :

```sql
-- Session A
SELECT pg_advisory_lock_shared(42);
-- ✅ Obtenu en mode PARTAGÉ

-- Session B
SELECT pg_advisory_lock_shared(42);
-- ✅ Obtenu aussi en mode PARTAGÉ (compatible)

-- Session C
SELECT pg_advisory_lock(42);  -- Mode EXCLUSIF
-- ⏳ BLOQUÉ ! Doit attendre que A et B libèrent
```

**Usage typique** : Plusieurs lecteurs, un écrivain (comme les read-write locks).

---

## Advisory Locks transactionnels : Plus sûrs

### 1. pg_advisory_xact_lock() : Le verrou transactionnel bloquant

**Syntaxe** :
```sql
SELECT pg_advisory_xact_lock(key bigint);
```

**Avantage majeur** : Libération **automatique** au COMMIT ou ROLLBACK.

**Exemple** :

```sql
BEGIN;

-- Obtenir le verrou
SELECT pg_advisory_xact_lock(42);
-- ✅ Obtenu

-- [Faire des opérations]
UPDATE jobs SET status = 'En cours' WHERE id = 42;

COMMIT;
-- Le verrou est automatiquement libéré ! Pas besoin de pg_advisory_unlock
```

**Cas d'erreur** :

```sql
BEGIN;

SELECT pg_advisory_xact_lock(42);

-- Une erreur se produit
UPDATE jobs SET invalid_column = 'value' WHERE id = 42;
-- ERROR!

ROLLBACK;
-- Le verrou est quand même libéré automatiquement ✅
```

### 2. pg_try_advisory_xact_lock() : Version non-bloquante

```sql
BEGIN;

IF pg_try_advisory_xact_lock(42) THEN
    -- Traiter le job
    UPDATE jobs SET status = 'Traité' WHERE id = 42;
    COMMIT;
ELSE
    -- Un autre worker traite déjà ce job
    ROLLBACK;
END IF;
```

### Pourquoi préférer les verrous transactionnels ?

- ✅ **Pas de fuite de verrous** : Libération automatique garantie
- ✅ **Plus simple** : Pas besoin de gérer la libération manuellement
- ✅ **Plus sûr** : En cas d'erreur, pas de verrou orphelin
- ✅ **Recommandé** pour la plupart des cas d'usage

⚠️ **Limitation** : Ne convient pas si vous avez besoin d'un verrou qui persiste entre plusieurs transactions.

---

## Utiliser deux clés (key1, key2)

PostgreSQL permet d'utiliser **deux entiers de 32 bits** au lieu d'un seul bigint de 64 bits. Cela facilite la création de verrous sur des "tuples" d'identifiants.

### Exemple : Verrou par (table, ligne)

```sql
-- Définir des constantes pour les tables
-- Table "users" = 1, Table "orders" = 2, etc.

-- Verrouiller l'utilisateur 42 dans la table users (1, 42)
SELECT pg_advisory_xact_lock(1, 42);

-- Verrouiller la commande 100 dans la table orders (2, 100)
SELECT pg_advisory_xact_lock(2, 100);
```

### Exemple : Verrou par (tenant_id, resource_id)

```sql
-- Application multi-tenant
-- Tenant 5, Ressource 789
SELECT pg_advisory_xact_lock(5, 789);

-- Tenant 5, Ressource 790
SELECT pg_advisory_xact_lock(5, 790);

-- Chaque tenant peut avoir ses propres verrous indépendants
```

### Conversion depuis une chaîne de caractères

Si votre ID est une chaîne, vous pouvez la convertir en entier :

```sql
-- Utiliser un hash de la chaîne
SELECT pg_advisory_xact_lock(hashtext('mon-job-unique'));

-- Ou pour UUID
SELECT pg_advisory_xact_lock(
    ('x' || substr(uuid_column::text, 1, 8))::bit(32)::int,
    ('x' || substr(uuid_column::text, 10, 8))::bit(32)::int
);
```

---

## Cas d'usage pratiques

### Cas 1 : File d'attente de jobs (Job Queue)

**Problème** : Plusieurs workers doivent traiter des jobs sans traiter le même job deux fois.

**Solution avec Advisory Lock transactionnel** :

```sql
-- Chaque worker exécute ceci en boucle
DO $$
DECLARE
    job_record RECORD;
BEGIN
    -- Trouver un job en attente
    SELECT * INTO job_record
    FROM jobs
    WHERE status = 'pending'
    ORDER BY priority DESC, created_at ASC
    LIMIT 1;

    -- Si aucun job, terminer
    IF NOT FOUND THEN
        RETURN;
    END IF;

    -- Essayer d'obtenir le verrou sur ce job
    IF pg_try_advisory_xact_lock(job_record.id) THEN
        -- Verrou obtenu ! Ce worker traite ce job

        -- Marquer comme en cours
        UPDATE jobs SET status = 'processing', started_at = NOW()
        WHERE id = job_record.id;

        -- [TRAITEMENT DU JOB ICI]
        -- ...

        -- Marquer comme terminé
        UPDATE jobs SET status = 'completed', completed_at = NOW()
        WHERE id = job_record.id;

        RAISE NOTICE 'Job % traité avec succès', job_record.id;
    ELSE
        -- Un autre worker traite déjà ce job
        RAISE NOTICE 'Job % déjà pris par un autre worker', job_record.id;
    END IF;
END $$;
```

**Avantage** : Aucun job n'est traité deux fois, même avec 100 workers en parallèle.

### Cas 2 : Génération de rapport unique

**Problème** : Un rapport mensuel doit être généré une seule fois, même si plusieurs processus essaient de le lancer.

**Solution avec Advisory Lock de session** :

```sql
-- Script de génération de rapport
DO $$
DECLARE
    report_id INTEGER := 202401;  -- Janvier 2024
BEGIN
    -- Essayer d'obtenir le verrou
    IF pg_try_advisory_lock(report_id) THEN
        BEGIN
            RAISE NOTICE 'Génération du rapport % en cours...', report_id;

            -- [GÉNÉRATION DU RAPPORT]
            INSERT INTO reports (month, data) VALUES (report_id, ...);

            RAISE NOTICE 'Rapport % terminé', report_id;
        EXCEPTION
            WHEN OTHERS THEN
                RAISE NOTICE 'Erreur lors de la génération: %', SQLERRM;
        END;

        -- Libérer le verrou
        PERFORM pg_advisory_unlock(report_id);
    ELSE
        RAISE NOTICE 'Le rapport % est déjà en cours de génération', report_id;
    END IF;
END $$;
```

**Résultat** : Si 10 processus lancent ce script simultanément, un seul génère le rapport, les autres sortent immédiatement.

### Cas 3 : Migration de données one-shot

**Problème** : Une migration de données doit s'exécuter une seule fois dans un cluster avec plusieurs instances de l'application.

**Solution** :

```sql
-- Fonction de migration avec protection
CREATE OR REPLACE FUNCTION run_migration_v2() RETURNS void AS $$
DECLARE
    migration_id BIGINT := 20240115001;  -- ID unique de la migration
BEGIN
    -- Essayer d'obtenir le verrou (non-bloquant)
    IF NOT pg_try_advisory_lock(migration_id) THEN
        RAISE NOTICE 'Migration déjà exécutée ou en cours';
        RETURN;
    END IF;

    BEGIN
        -- Vérifier si déjà exécutée
        IF EXISTS (SELECT 1 FROM migrations WHERE id = migration_id) THEN
            RAISE NOTICE 'Migration déjà exécutée';
            PERFORM pg_advisory_unlock(migration_id);
            RETURN;
        END IF;

        -- Exécuter la migration
        RAISE NOTICE 'Début de la migration %', migration_id;

        -- [CODE DE MIGRATION ICI]
        ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

        -- Enregistrer
        INSERT INTO migrations (id, executed_at) VALUES (migration_id, NOW());

        RAISE NOTICE 'Migration % terminée', migration_id;
    EXCEPTION
        WHEN OTHERS THEN
            RAISE EXCEPTION 'Erreur migration: %', SQLERRM;
    END;

    -- Libérer le verrou
    PERFORM pg_advisory_unlock(migration_id);
END;
$$ LANGUAGE plpgsql;

-- Exécuter
SELECT run_migration_v2();
```

### Cas 4 : Rate limiting distribué

**Problème** : Limiter le nombre d'opérations par utilisateur (par exemple, max 10 requêtes API par minute).

**Solution** :

```sql
CREATE OR REPLACE FUNCTION check_rate_limit(user_id INTEGER, max_requests INTEGER)
RETURNS boolean AS $$
DECLARE
    lock_key BIGINT;
    current_count INTEGER;
BEGIN
    -- Créer une clé unique : user_id + minute actuelle
    lock_key := user_id::BIGINT * 100000000 + EXTRACT(EPOCH FROM date_trunc('minute', NOW()))::BIGINT;

    -- Obtenir le verrou
    PERFORM pg_advisory_lock(lock_key);

    BEGIN
        -- Compter les requêtes dans cette minute
        SELECT COUNT(*) INTO current_count
        FROM api_requests
        WHERE user_id = user_id
          AND created_at >= date_trunc('minute', NOW());

        IF current_count >= max_requests THEN
            -- Limite atteinte
            PERFORM pg_advisory_unlock(lock_key);
            RETURN false;
        ELSE
            -- Enregistrer cette requête
            INSERT INTO api_requests (user_id, created_at) VALUES (user_id, NOW());
            PERFORM pg_advisory_unlock(lock_key);
            RETURN true;
        END IF;
    EXCEPTION
        WHEN OTHERS THEN
            PERFORM pg_advisory_unlock(lock_key);
            RAISE;
    END;
END;
$$ LANGUAGE plpgsql;

-- Utilisation
SELECT check_rate_limit(42, 10);  -- true ou false
```

### Cas 5 : Leader election (élection de leader)

**Problème** : Dans un cluster de workers, un seul doit être le "leader" à un moment donné.

**Solution** :

```sql
CREATE OR REPLACE FUNCTION try_become_leader(worker_id INTEGER)
RETURNS boolean AS $$
DECLARE
    leader_lock_key BIGINT := 999999;  -- ID fixe pour le rôle de leader
BEGIN
    IF pg_try_advisory_lock(leader_lock_key) THEN
        -- Ce worker est maintenant le leader
        RAISE NOTICE 'Worker % est devenu le leader', worker_id;
        RETURN true;
    ELSE
        -- Un autre worker est déjà leader
        RAISE NOTICE 'Worker % n''est pas le leader', worker_id;
        RETURN false;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Worker 1
SELECT try_become_leader(1);  -- true (devient leader)

-- Worker 2 (en parallèle)
SELECT try_become_leader(2);  -- false (pas leader)

-- Worker 1 libère (démission ou crash de connexion)
SELECT pg_advisory_unlock(999999);

-- Worker 2 peut maintenant devenir leader
SELECT try_become_leader(2);  -- true
```

---

## Surveillance et debugging des Advisory Locks

### Lister tous les Advisory Locks actifs

```sql
SELECT
    locktype,
    classid,
    objid,
    objsubid,
    virtualtransaction,
    pid,
    mode,
    granted,
    fastpath
FROM pg_locks
WHERE locktype = 'advisory'
ORDER BY pid;
```

**Résultat exemple** :

```
locktype | classid | objid | objsubid | pid   | mode        | granted
---------|---------|-------|----------|-------|-------------|--------
advisory | 0       | 42    | 1        | 12345 | ExclusiveLock | t
advisory | 0       | 100   | 1        | 12346 | ShareLock     | t
```

### Identifier les Advisory Locks par session

```sql
SELECT
    l.pid,
    l.locktype,
    l.classid,
    l.objid,
    l.mode,
    l.granted,
    a.usename,
    a.application_name,
    a.client_addr,
    a.query_start,
    a.state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.locktype = 'advisory';
```

### Trouver qui bloque qui (Advisory Locks)

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked.objid AS lock_key
FROM pg_locks blocked
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked.pid
JOIN pg_locks blocking ON blocking.objid = blocked.objid
    AND blocking.locktype = blocked.locktype
    AND blocking.pid != blocked.pid
    AND blocking.granted = true
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking.pid
WHERE blocked.locktype = 'advisory'
  AND NOT blocked.granted;
```

### Tuer une session qui détient un Advisory Lock

```sql
-- Identifier le PID
SELECT pid, objid FROM pg_locks WHERE locktype = 'advisory' AND objid = 42;

-- Terminer la session (libère automatiquement tous ses verrous)
SELECT pg_terminate_backend(12345);
```

---

## Bonnes pratiques

### 1. Préférer les verrous transactionnels (xact)

```sql
-- ✅ BON : Libération automatique
BEGIN;
SELECT pg_advisory_xact_lock(42);
-- ...
COMMIT;  -- Verrou libéré automatiquement

-- ⚠️ MOINS BON : Risque d'oubli
SELECT pg_advisory_lock(42);
-- ...
-- Si erreur avant pg_advisory_unlock → verrou éternel !
SELECT pg_advisory_unlock(42);
```

### 2. Utiliser des IDs de clé significatifs

```sql
-- ❌ MAUVAIS : Magic numbers
SELECT pg_advisory_lock(42);
SELECT pg_advisory_lock(123);
-- Que représentent ces nombres ?

-- ✅ BON : Constantes nommées
DO $$
DECLARE
    LOCK_MONTHLY_REPORT CONSTANT BIGINT := 1000001;
    LOCK_DAILY_SYNC CONSTANT BIGINT := 1000002;
BEGIN
    PERFORM pg_advisory_xact_lock(LOCK_MONTHLY_REPORT);
    -- Clair et maintenable
END $$;
```

### 3. Documenter votre schéma de clés

Créez une table de documentation :

```sql
CREATE TABLE advisory_lock_registry (
    lock_key BIGINT PRIMARY KEY,
    lock_name VARCHAR(100) NOT NULL,
    description TEXT,
    usage_pattern TEXT
);

INSERT INTO advisory_lock_registry VALUES
(1000001, 'monthly_report_generation', 'Empêche la génération multiple du rapport mensuel', 'Utilise le lock_key = 1000000 + YYYYMM'),
(1000002, 'daily_sync', 'Synchronisation quotidienne avec le système externe', 'Utilise le lock_key = 1000000 + jour de l''année'),
(2000000, 'job_processing', 'Base pour les verrous de jobs', 'Utilise 2000000 + job_id');
```

### 4. Gérer les timeouts

```sql
-- Définir un timeout pour éviter les blocages éternels
SET lock_timeout = '30s';

BEGIN;
-- Si le verrou n'est pas disponible en 30s, erreur
SELECT pg_advisory_xact_lock(42);
COMMIT;
```

### 5. Toujours libérer dans un bloc EXCEPTION

Pour les verrous de session :

```sql
DO $$
BEGIN
    -- Obtenir le verrou
    PERFORM pg_advisory_lock(42);

    BEGIN
        -- Code qui peut échouer
        PERFORM risque_erreur();
    EXCEPTION
        WHEN OTHERS THEN
            -- Libérer même en cas d'erreur
            PERFORM pg_advisory_unlock(42);
            RAISE;
    END;

    -- Libération normale
    PERFORM pg_advisory_unlock(42);
END $$;
```

### 6. Éviter les deadlocks avec Advisory Locks

Même règle que pour les verrous normaux : **ordre cohérent d'acquisition**.

```sql
-- ❌ MAUVAIS : Ordre différent
-- Transaction A
SELECT pg_advisory_lock(1);
SELECT pg_advisory_lock(2);

-- Transaction B
SELECT pg_advisory_lock(2);  -- Ordre inversé !
SELECT pg_advisory_lock(1);

-- ✅ BON : Toujours le même ordre
-- Transaction A
SELECT pg_advisory_lock(1);
SELECT pg_advisory_lock(2);

-- Transaction B
SELECT pg_advisory_lock(1);  -- Même ordre
SELECT pg_advisory_lock(2);
```

### 7. Implémenter un retry avec backoff

```python
import time
import psycopg2

def execute_with_advisory_lock(conn, lock_key, operation, max_retries=5):
    """
    Exécute une opération avec un advisory lock et retry si occupé
    """
    for attempt in range(max_retries):
        try:
            cur = conn.cursor()

            # Essayer d'obtenir le verrou (transactionnel)
            cur.execute("BEGIN")
            cur.execute("SELECT pg_try_advisory_xact_lock(%s)", (lock_key,))
            locked = cur.fetchone()[0]

            if locked:
                # Verrou obtenu, exécuter l'opération
                operation(cur)
                conn.commit()
                return True
            else:
                # Verrou occupé
                conn.rollback()

                if attempt == max_retries - 1:
                    raise Exception(f"Impossible d'obtenir le verrou après {max_retries} tentatives")

                # Attente avec backoff exponentiel
                wait_time = 0.1 * (2 ** attempt)
                time.sleep(wait_time)

        except Exception as e:
            conn.rollback()
            raise

    return False

# Utilisation
def process_job(cursor):
    cursor.execute("UPDATE jobs SET status = 'processing' WHERE id = 42")
    # ... traitement ...
    cursor.execute("UPDATE jobs SET status = 'completed' WHERE id = 42")

execute_with_advisory_lock(conn, lock_key=42, operation=process_job)
```

---

## Patterns avancés

### Pattern 1 : Verrou avec expiration (TTL)

PostgreSQL n'a pas de TTL natif pour les Advisory Locks, mais vous pouvez l'implémenter :

```sql
CREATE TABLE lock_registry (
    lock_key BIGINT PRIMARY KEY,
    acquired_at TIMESTAMPTZ NOT NULL,
    acquired_by VARCHAR(100),
    ttl_seconds INTEGER DEFAULT 300  -- 5 minutes
);

CREATE OR REPLACE FUNCTION acquire_lock_with_ttl(
    p_lock_key BIGINT,
    p_acquirer VARCHAR(100),
    p_ttl_seconds INTEGER DEFAULT 300
)
RETURNS boolean AS $$
BEGIN
    -- Nettoyer les verrous expirés
    DELETE FROM lock_registry
    WHERE lock_key = p_lock_key
      AND acquired_at < NOW() - (ttl_seconds || ' seconds')::INTERVAL;

    -- Essayer d'obtenir le verrou PostgreSQL
    IF pg_try_advisory_lock(p_lock_key) THEN
        -- Enregistrer
        INSERT INTO lock_registry (lock_key, acquired_at, acquired_by, ttl_seconds)
        VALUES (p_lock_key, NOW(), p_acquirer, p_ttl_seconds)
        ON CONFLICT (lock_key) DO UPDATE
        SET acquired_at = NOW(), acquired_by = p_acquirer, ttl_seconds = p_ttl_seconds;

        RETURN true;
    ELSE
        RETURN false;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### Pattern 2 : Compteur de sémaphore (limiter à N concurrents)

Permettre jusqu'à N workers simultanés :

```sql
CREATE TABLE semaphore_state (
    semaphore_key VARCHAR(100) PRIMARY KEY,
    max_concurrent INTEGER NOT NULL,
    current_count INTEGER DEFAULT 0,
    CHECK (current_count <= max_concurrent)
);

CREATE OR REPLACE FUNCTION acquire_semaphore(
    p_key VARCHAR(100),
    p_max INTEGER
)
RETURNS boolean AS $$
DECLARE
    v_lock_key BIGINT;
BEGIN
    -- Utiliser un advisory lock pour protéger le compteur
    v_lock_key := hashtext(p_key);
    PERFORM pg_advisory_lock(v_lock_key);

    BEGIN
        -- Créer ou mettre à jour l'entrée
        INSERT INTO semaphore_state (semaphore_key, max_concurrent, current_count)
        VALUES (p_key, p_max, 0)
        ON CONFLICT (semaphore_key) DO NOTHING;

        -- Vérifier si on peut incrémenter
        IF (SELECT current_count < max_concurrent FROM semaphore_state WHERE semaphore_key = p_key) THEN
            UPDATE semaphore_state
            SET current_count = current_count + 1
            WHERE semaphore_key = p_key;

            PERFORM pg_advisory_unlock(v_lock_key);
            RETURN true;
        ELSE
            PERFORM pg_advisory_unlock(v_lock_key);
            RETURN false;
        END IF;
    EXCEPTION
        WHEN OTHERS THEN
            PERFORM pg_advisory_unlock(v_lock_key);
            RAISE;
    END;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION release_semaphore(p_key VARCHAR(100))
RETURNS void AS $$
DECLARE
    v_lock_key BIGINT;
BEGIN
    v_lock_key := hashtext(p_key);
    PERFORM pg_advisory_lock(v_lock_key);

    UPDATE semaphore_state
    SET current_count = GREATEST(0, current_count - 1)
    WHERE semaphore_key = p_key;

    PERFORM pg_advisory_unlock(v_lock_key);
END;
$$ LANGUAGE plpgsql;

-- Utilisation : max 3 workers simultanés
SELECT acquire_semaphore('api_heavy_task', 3);  -- true
SELECT acquire_semaphore('api_heavy_task', 3);  -- true
SELECT acquire_semaphore('api_heavy_task', 3);  -- true
SELECT acquire_semaphore('api_heavy_task', 3);  -- false (limite atteinte)

-- Libérer
SELECT release_semaphore('api_heavy_task');
```

### Pattern 3 : Queue FIFO avec priorité

```sql
CREATE TABLE task_queue (
    task_id SERIAL PRIMARY KEY,
    task_data JSONB,
    priority INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    locked_at TIMESTAMPTZ,
    locked_by VARCHAR(100)
);

CREATE OR REPLACE FUNCTION dequeue_task(worker_id VARCHAR(100))
RETURNS TABLE(task_id INTEGER, task_data JSONB) AS $$
DECLARE
    v_task RECORD;
BEGIN
    -- Trouver la prochaine tâche
    SELECT t.task_id, t.task_data INTO v_task
    FROM task_queue t
    WHERE t.status = 'pending'
    ORDER BY t.priority DESC, t.created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED;  -- Ignorer les tâches déjà verrouillées

    IF NOT FOUND THEN
        RETURN;
    END IF;

    -- Essayer d'obtenir un advisory lock
    IF pg_try_advisory_xact_lock(v_task.task_id) THEN
        -- Marquer comme verrouillée
        UPDATE task_queue
        SET status = 'processing',
            locked_at = NOW(),
            locked_by = worker_id
        WHERE task_queue.task_id = v_task.task_id;

        RETURN QUERY SELECT v_task.task_id, v_task.task_data;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Worker récupère une tâche
BEGIN;
SELECT * FROM dequeue_task('worker-1');
-- Traiter la tâche
UPDATE task_queue SET status = 'completed' WHERE task_id = ...;
COMMIT;
```

---

## Comparaison : Advisory Locks vs autres mécanismes

| Mécanisme | Scope | Performance | Complexité | Use Case |
|-----------|-------|-------------|------------|----------|
| **Advisory Locks** | Database-level | ⚡⚡⚡ Excellent | 🔧 Simple | Coordination entre workers DB |
| **Verrous applicatifs (Redis)** | External | ⚡⚡ Bon (réseau) | 🔧🔧 Moyen | Coordination entre services |
| **Database triggers** | Database-level | ⚡ Moyen | 🔧🔧🔧 Complexe | Validation automatique |
| **Optimistic locking (version)** | Application-level | ⚡⚡⚡ Excellent | 🔧🔧 Moyen | Détection de conflits |
| **Distributed locks (etcd, ZooKeeper)** | Cluster-level | ⚡⚡ Bon | 🔧🔧🔧 Complexe | Coordination distribuée |

**Quand utiliser Advisory Locks** :

- ✅ Vos workers sont tous connectés à la même base PostgreSQL
- ✅ Vous voulez une solution simple sans dépendance externe
- ✅ La performance est importante (pas de latence réseau)
- ✅ Vous gérez des jobs, reports, migrations

**Quand NE PAS utiliser Advisory Locks** :

- ❌ Coordination entre services qui n'utilisent pas PostgreSQL
- ❌ Architecture microservices avec bases séparées
- ❌ Besoin de locks qui survivent à la perte de connexion DB
- ❌ Locks inter-datacenter (géo-distribution)

---

## Troubleshooting courant

### Problème 1 : Verrous qui ne sont jamais libérés

**Symptôme** :
```sql
SELECT * FROM pg_locks WHERE locktype = 'advisory';
-- Des verrous vieux de plusieurs heures/jours
```

**Causes** :
- Oubli de `pg_advisory_unlock()` sur un verrou de session
- Exception non gérée qui empêche l'unlock
- Worker/connexion qui crash sans fermer proprement

**Solution** :
```sql
-- Identifier le PID
SELECT pid, objid, age(NOW(), query_start)
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE locktype = 'advisory'
ORDER BY query_start;

-- Tuer la session si nécessaire
SELECT pg_terminate_backend(pid_problematique);

-- Prévention : toujours utiliser pg_advisory_xact_lock si possible
```

### Problème 2 : Deadlock avec Advisory Locks

**Symptôme** :
```
ERROR: deadlock detected
```

**Cause** : Acquisition dans un ordre différent

**Solution** : Ordre cohérent (voir bonnes pratiques)

### Problème 3 : Performance dégradée

**Symptôme** : Ralentissements généraux

**Diagnostic** :
```sql
-- Trop de sessions en attente d'advisory locks ?
SELECT COUNT(*)
FROM pg_locks
WHERE locktype = 'advisory' AND NOT granted;
```

**Solution** :
- Réduire la durée de détention des locks
- Utiliser `pg_try_advisory_lock` au lieu de la version bloquante
- Revoir la stratégie de verrouillage

---

## Résumé et checklist

### Fonctions essentielles

| Usage | Fonction recommandée |
|-------|---------------------|
| Verrou exclusif transactionnel | `pg_advisory_xact_lock(key)` |
| Essai non-bloquant transactionnel | `pg_try_advisory_xact_lock(key)` |
| Verrou partagé transactionnel | `pg_advisory_xact_lock_shared(key)` |
| Verrou session (déconseillé) | `pg_advisory_lock(key)` + `pg_advisory_unlock(key)` |

### Checklist de bonnes pratiques

- ✅ Utiliser les verrous **transactionnels** (`xact`) par défaut
- ✅ Documenter vos **schémas de clés** (table de registry)
- ✅ Utiliser des **constantes nommées** pour les clés
- ✅ Gérer les **exceptions** avec libération garantie
- ✅ Acquérir les verrous dans un **ordre cohérent**
- ✅ Définir des **timeouts** appropriés
- ✅ **Monitorer** les advisory locks actifs
- ✅ Utiliser `pg_try_advisory_lock` pour éviter les blocages
- ✅ Implémenter des **retries** avec backoff exponentiel
- ✅ **Tester** vos scénarios de concurrence

### Erreurs à éviter

- ❌ Oublier de libérer un verrou de session
- ❌ Utiliser des magic numbers sans documentation
- ❌ Acquérir des verrous dans un ordre incohérent
- ❌ Garder un verrou pendant des opérations longues
- ❌ Ne pas gérer les exceptions/erreurs
- ❌ Ne pas monitorer les verrous actifs
- ❌ Utiliser les advisory locks pour des use cases inappropriés

---

## Conclusion

Les **Advisory Locks** sont un outil puissant pour implémenter une **coordination applicative** au sein de PostgreSQL. Ils vous permettent de :

- ✅ Coordonner des workers distribués simplement
- ✅ Implémenter des mutex et sémaphores
- ✅ Garantir l'unicité d'exécution de tâches
- ✅ Créer des files d'attente robustes
- ✅ Gérer des élections de leader

**Avantages** :
- 🚀 Performance excellente (pas de latence réseau)
- 🔧 Simplicité (pas de dépendance externe)
- 🛡️ Atomicité garantie par PostgreSQL
- 🔍 Observable (pg_locks)

**Points clés à retenir** :

1. Les advisory locks sont **consultatifs** : c'est à vous de les respecter
2. Préférez les verrous **transactionnels** (libération automatique)
3. Documentez votre **schéma de clés**
4. Suivez les **bonnes pratiques** (ordre, timeouts, monitoring)
5. C'est idéal pour coordonner des **workers PostgreSQL**, moins pour l'inter-service

Dans les prochaines sections du chapitre 12, nous continuerons à explorer la concurrence dans PostgreSQL, en abordant des sujets comme les stratégies de résolution de conflits et les patterns de haute concurrence.

---

**Points clés à retenir :**

- 🔑 Advisory Locks = verrous applicatifs personnalisés
- 🔑 Deux types : session (manuel) et transactionnel (auto-libération)
- 🔑 pg_advisory_xact_lock() = version recommandée (transactionnel)
- 🔑 Parfait pour job queues, élections de leader, migrations
- 🔑 Clés numériques : utilisez des constantes documentées
- 🔑 Monitoring via pg_locks WHERE locktype = 'advisory'
- 🔑 Respecter l'ordre d'acquisition pour éviter les deadlocks
- 🔑 Alternative simple à Redis/ZooKeeper pour la coordination DB

⏭️ [Stratégies de détection et résolution des deadlocks](/12-concurrence-et-transactions/07-strategies-detection-resolution-deadlocks.md)
