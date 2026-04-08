🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.7. Stratégies de détection et résolution des deadlocks

## Introduction : Le défi des interblocages

Nous avons vu précédemment ce qu'est un **deadlock** (interblocage) : une situation où deux ou plusieurs transactions s'attendent mutuellement pour obtenir des verrous, créant un cycle sans issue où **personne ne peut progresser**.

**Analogie** : Imaginez deux voitures qui arrivent face à face dans une rue étroite à sens unique. La voiture A ne peut avancer que si B recule. La voiture B ne peut avancer que si A recule. Résultat : **blocage total**.

Dans ce chapitre, nous allons explorer en profondeur :
- 🔍 Comment **détecter** les deadlocks avant qu'ils ne causent des problèmes  
- 🛡️ Comment les **prévenir** par une conception intelligente  
- 🚑 Comment les **résoudre** quand ils se produisent malgré tout  
- 📊 Les **outils** et requêtes pour diagnostiquer les situations de blocage

---

## Rappel : Anatomie d'un deadlock

### Le cycle vicieux

Un deadlock se produit lorsqu'il existe un **cycle dans le graphe d'attente des verrous** :

```
Transaction T1 : détient verrou sur A, attend verrou sur B  
Transaction T2 : détient verrou sur B, attend verrou sur A  

Graphe :  
T1 → B (attend)  
 ↑    ↓
 A    T2 (détient)
 ↑    ↓
(détient) ← B

Cycle : T1 → B → T2 → A → T1 (DEADLOCK !)
```

### Conditions nécessaires pour un deadlock

Un deadlock nécessite **quatre conditions simultanées** (théorème de Coffman) :

1. **Exclusion mutuelle** : Les ressources ne peuvent être détenues que par une transaction à la fois  
2. **Hold and wait** : Une transaction détient des ressources et en attend d'autres  
3. **No preemption** : Les ressources ne peuvent être arrachées de force  
4. **Circular wait** : Il existe un cycle dans le graphe d'attente

**Stratégie de prévention** : Casser au moins l'une de ces conditions.

---

## Le mécanisme de détection de PostgreSQL

### Comment PostgreSQL détecte les deadlocks

PostgreSQL utilise un **détecteur de deadlocks** qui fonctionne périodiquement :

#### 1. Le paramètre deadlock_timeout

```sql
-- Afficher la configuration actuelle
SHOW deadlock_timeout;
-- Valeur par défaut : 1000ms (1 seconde)
```

**Fonctionnement** :
- Lorsqu'une transaction attend un verrou, PostgreSQL démarre un timer
- Si le verrou n'est **pas obtenu** après `deadlock_timeout` millisecondes, PostgreSQL lance la détection
- Le détecteur analyse le **graphe des dépendances** de verrous

**Pourquoi attendre 1 seconde ?**

La détection de deadlocks est **coûteuse** (analyse de graphe). PostgreSQL suppose que la plupart des attentes se résolvent rapidement. Attendre 1 seconde évite de vérifier inutilement.

#### 2. Configuration du deadlock_timeout

```sql
-- Au niveau de la session
SET deadlock_timeout = '2s';  -- 2 secondes

-- Au niveau de la base
ALTER DATABASE ma_base SET deadlock_timeout = '500ms';

-- Au niveau du serveur (postgresql.conf)
deadlock_timeout = 1s
```

**Recommandations** :

| Workload | deadlock_timeout recommandé | Raison |
|----------|----------------------------|--------|
| OLTP (transactions courtes) | 500ms - 1s | Détection rapide |
| OLAP (requêtes longues) | 2s - 5s | Éviter les faux positifs |
| Mixte | 1s (défaut) | Bon équilibre |
| Développement | 100ms - 500ms | Détecter rapidement les bugs |

**⚠️ Attention** : Un `deadlock_timeout` trop court déclenche des vérifications inutiles. Trop long retarde la détection.

#### 3. L'algorithme de détection

```
1. Transaction A attend un verrou depuis > deadlock_timeout
   ↓
2. PostgreSQL construit le graphe des dépendances :
   - Qui attend quoi ?
   - Qui détient quoi ?
   ↓
3. Recherche de cycles dans le graphe (algorithme DFS)
   ↓
4. Cycle trouvé ?
   OUI → DEADLOCK détecté !
        → Choisir une victime (transaction à annuler)
        → Envoyer l'erreur à la victime
   NON → Continuer d'attendre
```

### Le message d'erreur de deadlock

Quand PostgreSQL détecte un deadlock, il envoie cette erreur à l'une des transactions :

```
ERROR:  deadlock detected  
DETAIL:  Process 12345 waits for ShareLock on transaction 67890; blocked by process 12346.  
Process 12346 waits for ShareLock on transaction 67889; blocked by process 12345.  
HINT:  See server log for query details.  
CONTEXT:  while updating tuple (0,42) in relation "comptes"  
```

**Informations contenues** :
- Les **PID** (Process IDs) impliqués
- Le **type de verrou** attendu
- La **relation** (table) concernée
- Le **tuple** (ligne) en conflit

### Logs PostgreSQL pour les deadlocks

Configuration pour logger les deadlocks :

```ini
# Dans postgresql.conf

# Logger les deadlocks
log_lock_waits = on  # Logger toutes les attentes de verrous longues

# Logger plus de détails sur les deadlocks
log_line_prefix = '%m [%p] %u@%d '  # timestamp, pid, user, database
```

**Exemple de log** :

```
2024-01-15 10:30:45.123 UTC [12345] user@mydb ERROR:  deadlock detected
2024-01-15 10:30:45.123 UTC [12345] user@mydb DETAIL:  Process 12345 waits for ShareLock on transaction 67890; blocked by process 12346.
	Process 12346 waits for ShareLock on transaction 67889; blocked by process 12345.
	Process 12345: UPDATE comptes SET solde = solde - 100 WHERE id = 'B';
	Process 12346: UPDATE comptes SET solde = solde + 50 WHERE id = 'A';
2024-01-15 10:30:45.123 UTC [12345] user@mydb HINT:  See server log for query details.
2024-01-15 10:30:45.123 UTC [12345] user@mydb CONTEXT:  while updating tuple (0,42) in relation "comptes"
2024-01-15 10:30:45.123 UTC [12345] user@mydb STATEMENT:  UPDATE comptes SET solde = solde - 100 WHERE id = 'B'
```

---

## Stratégies de prévention des deadlocks

La meilleure stratégie est de **prévenir** les deadlocks plutôt que de les gérer après coup.

### 1. Ordre cohérent d'accès aux ressources ⭐⭐⭐

**Principe** : Toujours accéder aux tables et lignes dans le **même ordre** dans toutes les transactions.

#### Pourquoi ça fonctionne ?

Si toutes les transactions accèdent aux ressources dans le même ordre, aucun cycle ne peut se former.

**Exemple de deadlock** :

```sql
-- Transaction A
BEGIN;  
UPDATE comptes SET solde = solde - 100 WHERE id = 'A';  -- Verrou sur A  
-- [pause]
UPDATE comptes SET solde = solde + 100 WHERE id = 'B';  -- Attend verrou sur B  
COMMIT;  

-- Transaction B (en parallèle)
BEGIN;  
UPDATE comptes SET solde = solde - 50 WHERE id = 'B';   -- Verrou sur B  
-- [pause]
UPDATE comptes SET solde = solde + 50 WHERE id = 'A';   -- Attend verrou sur A (DEADLOCK !)  
COMMIT;  
```

**Solution : Ordre alphabétique** :

```sql
-- Transaction A
BEGIN;  
UPDATE comptes SET solde = solde - 100 WHERE id = 'A';  -- Verrou sur A (ordre : A puis B)  
UPDATE comptes SET solde = solde + 100 WHERE id = 'B';  -- Verrou sur B  
COMMIT;  

-- Transaction B
BEGIN;  
UPDATE comptes SET solde = solde + 50 WHERE id = 'A';   -- Verrou sur A (même ordre : A puis B)  
-- Doit attendre que Transaction A libère A
UPDATE comptes SET solde = solde - 50 WHERE id = 'B';  
COMMIT;  

-- Pas de deadlock ! Transaction B attend simplement son tour
```

#### Pattern recommandé : Trier les IDs

```sql
-- ❌ MAUVAIS : Ordre imprévisible
BEGIN;  
UPDATE produits SET stock = stock - 1 WHERE id IN (42, 10, 35);  
COMMIT;  

-- ✅ BON : Trier les IDs avant de verrouiller
BEGIN;
-- Trier par ID pour garantir un ordre cohérent
UPDATE produits SET stock = stock - 1  
WHERE id IN (42, 10, 35)  
ORDER BY id;  -- Ordre : 10, 35, 42  
COMMIT;  
```

#### Avec SELECT FOR UPDATE

```sql
-- Verrouiller plusieurs lignes dans un ordre cohérent
BEGIN;

SELECT * FROM commandes  
WHERE id IN (100, 200, 50, 150)  
ORDER BY id  -- Tri : 50, 100, 150, 200  
FOR UPDATE;  

-- Modifier en toute sécurité
UPDATE commandes SET statut = 'Traité' WHERE id IN (100, 200, 50, 150);

COMMIT;
```

### 2. Transactions courtes et atomiques ⭐⭐⭐

**Principe** : Minimiser la **durée** pendant laquelle les verrous sont détenus.

#### Mauvais exemple : Transaction longue

```sql
-- ❌ TRÈS MAUVAIS
BEGIN;

-- Verrou sur la ligne
UPDATE comptes SET solde = solde - 100 WHERE id = 'A';

-- Traitement long dans l'application (détient le verrou pendant ce temps !)
SELECT pg_sleep(30);  -- 30 secondes !

-- Appel API externe (peut échouer ou prendre du temps)
-- [Requête HTTP vers un service externe]

-- Calculs complexes
-- [Traitement métier lourd]

UPDATE autre_table SET ...;

COMMIT;  -- Libère enfin les verrous après plusieurs minutes !
```

**Problèmes** :
- Les verrous sont détenus **pendant des minutes**
- Autres transactions **bloquées** inutilement
- Probabilité de deadlock **élevée**

#### Bon exemple : Transaction courte

```sql
-- ✅ BON
-- 1. Faire tous les traitements AVANT la transaction
local_data = faire_appel_api()  
resultat_calcul = traitement_complexe(local_data)  

-- 2. Transaction très courte
BEGIN;

UPDATE comptes SET solde = solde - 100 WHERE id = 'A';  
UPDATE autre_table SET valeur = resultat_calcul WHERE id = 1;  

COMMIT;  -- Libère les verrous en quelques millisecondes !
```

#### Pattern recommandé

```
┌─────────────────────────────────────────────────┐
│ Hors transaction (pas de verrous)               │
├─────────────────────────────────────────────────┤
│ • Lire les données nécessaires                  │
│ • Appels API externes                           │
│ • Calculs complexes                             │
│ • Validation métier                             │
│ • Interaction utilisateur                       │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ BEGIN                                           │
├─────────────────────────────────────────────────┤
│ • UPDATE/INSERT/DELETE rapides                  │
│ • Opérations atomiques SQL                      │
│ • Pas de pg_sleep(), pas d'appels externes      │
├─────────────────────────────────────────────────┤
│ COMMIT (< 100ms idéalement)                     │
└─────────────────────────────────────────────────┘
```

### 3. Verrouillage explicite et précoce ⭐⭐

**Principe** : Acquérir **tous les verrous nécessaires au début** de la transaction.

#### Avec LOCK TABLE

```sql
-- Pour des opérations complexes impliquant plusieurs tables
BEGIN;

-- Verrouiller toutes les tables nécessaires IMMÉDIATEMENT
LOCK TABLE comptes IN SHARE ROW EXCLUSIVE MODE;  
LOCK TABLE transactions IN SHARE ROW EXCLUSIVE MODE;  
LOCK TABLE historique IN SHARE ROW EXCLUSIVE MODE;  

-- Maintenant, faire toutes les opérations
UPDATE comptes ...;  
INSERT INTO transactions ...;  
INSERT INTO historique ...;  

COMMIT;
```

**Avantage** : Si le verrou n'est pas disponible, on le sait **tout de suite** (pas après avoir fait la moitié du travail).

#### Avec SELECT ... FOR UPDATE au début

```sql
BEGIN;

-- Verrouiller toutes les lignes nécessaires D'ABORD
SELECT * FROM commandes WHERE id IN (1, 2, 3) ORDER BY id FOR UPDATE;  
SELECT * FROM produits WHERE id IN (10, 20) ORDER BY id FOR UPDATE;  

-- Maintenant, les modifications sont sans risque de deadlock
UPDATE commandes SET statut = 'Validé' WHERE id = 1;  
UPDATE produits SET stock = stock - 1 WHERE id = 10;  
-- etc.

COMMIT;
```

### 4. Utiliser des timeouts ⭐⭐

**Principe** : Configurer des **limites de temps** pour détecter les problèmes rapidement.

#### lock_timeout

Temps maximum pour attendre un verrou :

```sql
-- Au niveau de la session
SET lock_timeout = '5s';  -- 5 secondes max

BEGIN;
-- Si un verrou n'est pas obtenu en 5s → ERROR
UPDATE comptes SET solde = solde - 100 WHERE id = 'A';  
COMMIT;  
```

**Erreur retournée** :

```
ERROR:  canceling statement due to lock timeout
```

#### statement_timeout

Temps maximum d'exécution d'une requête :

```sql
SET statement_timeout = '30s';  -- 30 secondes max

BEGIN;
-- Si la requête prend plus de 30s → ERROR
UPDATE huge_table SET ...;  
COMMIT;  
```

#### Configuration recommandée

```sql
-- Pour les transactions OLTP
SET lock_timeout = '10s';  
SET statement_timeout = '60s';  

-- Pour les transactions batch/reporting
SET lock_timeout = '5m';  
SET statement_timeout = '30m';  
```

### 5. Éviter les transactions interactives ⭐⭐⭐

**Principe** : Ne **jamais** garder une transaction ouverte pendant une interaction utilisateur.

#### Anti-pattern dangereux

```sql
-- ❌ TERRIBLE
BEGIN;

-- Verrouiller des lignes
UPDATE commandes SET statut = 'En modification' WHERE id = 123;

-- [Attendre que l'utilisateur remplisse un formulaire web : peut prendre des MINUTES !]
-- [Pendant ce temps, la ligne est verrouillée]

-- L'utilisateur valide finalement
UPDATE commandes SET montant = nouveau_montant WHERE id = 123;

COMMIT;
```

**Problèmes** :
- Verrous détenus pendant des **minutes** ou **heures**
- Risque énorme de deadlock
- Mauvaise expérience utilisateur (autres utilisateurs bloqués)

#### Pattern recommandé : Optimistic Locking

```sql
-- 1. Lire sans verrouiller (avec numéro de version)
SELECT id, montant, version FROM commandes WHERE id = 123;
-- Résultat : montant=100, version=5

-- [Utilisateur modifie dans l'interface pendant 5 minutes]

-- 2. Au moment de sauvegarder, vérifier la version
BEGIN;

UPDATE commandes  
SET montant = 150, version = version + 1  
WHERE id = 123 AND version = 5;  -- Vérifier que version n'a pas changé  

-- Si affected_rows = 0 → quelqu'un d'autre a modifié
IF NOT FOUND THEN
    ROLLBACK;
    RAISE EXCEPTION 'La commande a été modifiée par quelqu''un d''autre';
END IF;

COMMIT;
```

### 6. Réduire la granularité des verrous ⭐

**Principe** : Verrouiller uniquement ce qui est **strictement nécessaire**.

#### Mauvais : Verrouiller toute la table

```sql
-- ❌ MAUVAIS
BEGIN;  
LOCK TABLE produits IN EXCLUSIVE MODE;  -- Bloque TOUT  
UPDATE produits SET prix = prix * 1.1 WHERE categorie = 'Électronique';  
COMMIT;  
```

#### Bon : Verrouiller seulement les lignes nécessaires

```sql
-- ✅ BON
BEGIN;
-- Verrouiller seulement les lignes concernées
SELECT * FROM produits WHERE categorie = 'Électronique' FOR UPDATE;  
UPDATE produits SET prix = prix * 1.1 WHERE categorie = 'Électronique';  
COMMIT;  
```

### 7. Utiliser NOWAIT ou SKIP LOCKED ⭐⭐

**Principe** : Éviter d'attendre indéfiniment.

#### FOR UPDATE NOWAIT

```sql
BEGIN;

-- Essayer d'obtenir le verrou, échouer immédiatement si occupé
SELECT * FROM produits WHERE id = 1 FOR UPDATE NOWAIT;
-- Si la ligne est déjà verrouillée → ERROR immédiate (pas d'attente)

-- Traiter l'erreur dans l'application
EXCEPTION
    WHEN lock_not_available THEN
        -- Réessayer plus tard ou abandonner
        RAISE NOTICE 'Ressource occupée';
        ROLLBACK;

COMMIT;
```

**Avantage** : Pas de risque de deadlock (pas d'attente).

#### FOR UPDATE SKIP LOCKED

```sql
-- Pour des queues de travail
BEGIN;

-- Prendre la prochaine tâche disponible en sautant les verrouillées
SELECT * FROM jobs  
WHERE status = 'pending'  
ORDER BY priority DESC, created_at ASC  
LIMIT 1  
FOR UPDATE SKIP LOCKED;  

-- Si aucune tâche disponible (toutes verrouillées) → résultat vide

IF FOUND THEN
    -- Traiter la tâche
    UPDATE jobs SET status = 'processing' WHERE id = ...;
END IF;

COMMIT;
```

**Cas d'usage parfait** : Workers multiples qui traitent des jobs en parallèle.

---

## Stratégies de détection (Monitoring)

### 1. Surveiller les métriques de deadlocks

#### Compter les deadlocks

```sql
-- Nombre de deadlocks détectés depuis le dernier redémarrage
SELECT
    datname,
    deadlocks
FROM pg_stat_database  
WHERE datname = current_database();  
```

**Exemple de résultat** :

```
datname   | deadlocks
----------|----------
mydb      | 47
```

**Interprétation** :
- 0-10 : Normal, occasionnel
- 10-100 : À surveiller, peut indiquer un problème de conception
- >100 : Problème sérieux, action requise

#### Taux de deadlocks

```sql
-- Ratio deadlocks / transactions
SELECT
    datname,
    deadlocks,
    xact_commit + xact_rollback AS total_transactions,
    ROUND(100.0 * deadlocks / NULLIF(xact_commit + xact_rollback, 0), 4) AS deadlock_ratio
FROM pg_stat_database  
WHERE datname NOT IN ('template0', 'template1');  
```

**Seuils recommandés** :
- < 0.01% : Excellent
- 0.01-0.1% : Acceptable
- > 0.1% : Problématique

### 2. Identifier les transactions en attente de verrous

#### Vue pg_locks : Les verrous actifs

```sql
-- Tous les verrous non accordés (en attente)
SELECT
    pl.pid,
    pl.locktype,
    pl.relation::regclass AS table_name,
    pl.mode,
    pl.granted,
    psa.usename,
    psa.query,
    psa.state,
    age(now(), psa.query_start) AS waiting_time
FROM pg_locks pl  
LEFT JOIN pg_stat_activity psa ON pl.pid = psa.pid  
WHERE NOT pl.granted  
ORDER BY psa.query_start;  
```

#### Identifier qui bloque qui

```sql
-- Requête complète pour voir les blocages
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement,
    blocked_activity.application_name AS blocked_app,
    age(now(), blocked_activity.query_start) AS blocked_duration
FROM pg_catalog.pg_locks blocked_locks  
JOIN pg_catalog.pg_stat_activity blocked_activity  
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

#### Fonction PostgreSQL simplifiée

```sql
-- Voir qui bloque le processus 12345
SELECT pg_blocking_pids(12345);
-- Retourne : {12346, 12347}  (les PID qui bloquent)
```

### 3. Alertes automatiques

#### Script de monitoring (à exécuter périodiquement)

```sql
-- Créer une fonction d'alerte
CREATE OR REPLACE FUNCTION check_deadlock_alert()  
RETURNS TABLE(severity TEXT, message TEXT) AS $$  
BEGIN  
    -- Alerte si trop de deadlocks récents
    IF (SELECT deadlocks FROM pg_stat_database WHERE datname = current_database()) > 100 THEN
        RETURN QUERY SELECT 'CRITICAL'::TEXT, 'Plus de 100 deadlocks détectés'::TEXT;
    END IF;

    -- Alerte si transactions en attente trop longtemps
    IF EXISTS (
        SELECT 1 FROM pg_locks l
        JOIN pg_stat_activity a ON l.pid = a.pid
        WHERE NOT l.granted
          AND age(now(), a.query_start) > interval '5 minutes'
    ) THEN
        RETURN QUERY SELECT 'WARNING'::TEXT, 'Transactions en attente depuis plus de 5 minutes'::TEXT;
    END IF;

    RETURN QUERY SELECT 'OK'::TEXT, 'Aucun problème détecté'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Exécuter
SELECT * FROM check_deadlock_alert();
```

#### Intégration avec Prometheus/Grafana

```sql
-- Exposer les métriques pour postgres_exporter
-- pg_stat_database_deadlocks
SELECT datname, deadlocks FROM pg_stat_database;

-- pg_locks_waiting
SELECT COUNT(*) AS waiting_locks FROM pg_locks WHERE NOT granted;
```

**Dashboard Grafana recommandé** :
- Graphe : Nombre de deadlocks au fil du temps
- Jauge : Transactions en attente
- Liste : Top 10 des requêtes bloquées
- Alerte : Si deadlocks/hour > seuil

---

## Stratégies de résolution (Quand un deadlock arrive)

### 1. PostgreSQL résout automatiquement

**Rappel** : PostgreSQL **détecte et résout automatiquement** les deadlocks :

1. Détection du cycle  
2. Choix d'une **victime** (transaction à annuler)  
3. Envoi de l'erreur à la victime  
4. Les autres transactions continuent

**Vous devez** : Implémenter une logique de **retry** dans votre application.

### 2. Pattern de retry automatique

#### Python (psycopg2)

```python
import time  
import psycopg2  
from psycopg2.extensions import TransactionRollbackError  

def execute_with_deadlock_retry(conn, operation, max_retries=5):
    """
    Exécute une opération avec retry automatique en cas de deadlock
    """
    for attempt in range(max_retries):
        try:
            # Tenter l'opération
            cursor = conn.cursor()
            operation(cursor)
            conn.commit()
            return True  # Succès

        except TransactionRollbackError as e:
            # Deadlock détecté
            conn.rollback()

            # Logger
            print(f"Deadlock détecté (tentative {attempt + 1}/{max_retries})")

            if attempt == max_retries - 1:
                # Dernière tentative échouée
                raise Exception(f"Échec après {max_retries} tentatives") from e

            # Backoff exponentiel avec jitter
            wait_time = (0.1 * (2 ** attempt)) * (1 + random.uniform(0, 0.3))
            print(f"Attente de {wait_time:.2f}s avant retry...")
            time.sleep(wait_time)

        except Exception as e:
            # Autre erreur (pas un deadlock)
            conn.rollback()
            raise

    return False

# Utilisation
def mon_transfert(cursor):
    # Accès dans un ordre cohérent
    cursor.execute("UPDATE comptes SET solde = solde - 100 WHERE id = 'A'")
    cursor.execute("UPDATE comptes SET solde = solde + 100 WHERE id = 'B'")

execute_with_deadlock_retry(conn, mon_transfert)
```

#### Java (JDBC)

```java
public boolean executeWithDeadlockRetry(
    Connection conn,
    SqlOperation operation,
    int maxRetries
) {
    for (int attempt = 0; attempt < maxRetries; attempt++) {
        try {
            operation.execute(conn);
            conn.commit();
            return true;

        } catch (SQLException e) {
            // Code d'erreur PostgreSQL pour deadlock : 40P01
            if ("40P01".equals(e.getSQLState())) {
                conn.rollback();

                if (attempt == maxRetries - 1) {
                    throw new RuntimeException(
                        "Échec après " + maxRetries + " tentatives", e
                    );
                }

                // Backoff exponentiel
                long waitMs = (long) (100 * Math.pow(2, attempt));
                Thread.sleep(waitMs);

            } else {
                conn.rollback();
                throw e;
            }
        }
    }
    return false;
}
```

#### Node.js (node-postgres)

```javascript
async function executeWithDeadlockRetry(client, operation, maxRetries = 5) {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            await client.query('BEGIN');
            await operation(client);
            await client.query('COMMIT');
            return true;  // Succès

        } catch (error) {
            await client.query('ROLLBACK');

            // Code d'erreur PostgreSQL pour deadlock : 40P01
            if (error.code === '40P01') {
                console.log(`Deadlock détecté (tentative ${attempt + 1}/${maxRetries})`);

                if (attempt === maxRetries - 1) {
                    throw new Error(`Échec après ${maxRetries} tentatives`);
                }

                // Backoff exponentiel
                const waitMs = 100 * Math.pow(2, attempt);
                await new Promise(resolve => setTimeout(resolve, waitMs));

            } else {
                throw error;  // Autre erreur
            }
        }
    }
    return false;
}

// Utilisation
await executeWithDeadlockRetry(client, async (client) => {
    await client.query("UPDATE comptes SET solde = solde - 100 WHERE id = 'A'");
    await client.query("UPDATE comptes SET solde = solde + 100 WHERE id = 'B'");
});
```

### 3. Stratégie de backoff

**Types de backoff** :

#### Backoff exponentiel simple

```
Tentative 1 : Attendre 100ms  
Tentative 2 : Attendre 200ms  
Tentative 3 : Attendre 400ms  
Tentative 4 : Attendre 800ms  
Tentative 5 : Attendre 1600ms  
```

#### Backoff exponentiel avec jitter (recommandé)

```python
import random

wait_time = base_delay * (2 ** attempt) * (1 + random.uniform(0, 0.3))
```

**Pourquoi le jitter ?** : Éviter que plusieurs transactions qui ont échoué en même temps ne réessaient toutes exactement au même moment (thundering herd).

#### Backoff avec cap

```python
wait_time = min(max_delay, base_delay * (2 ** attempt))
# Exemple : max 5 secondes
```

### 4. Décider du nombre de retries

| Type de transaction | Retries recommandés | Raison |
|---------------------|--------------------|---------|
| Transaction critique (paiement) | 5-10 | Doit absolument aboutir |
| Transaction normale (CRUD) | 3-5 | Équilibre |
| Transaction non-critique (analytics) | 1-2 | Échec acceptable |
| Background job | 10-20 | Peut réessayer longtemps |

---

## Outils de diagnostic avancés

### 1. Extension pg_stat_statements

Identifier les requêtes qui causent des deadlocks :

```sql
-- Installer l'extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Trouver les requêtes les plus souvent impliquées dans des deadlocks
-- (via l'analyse des logs et corrélation avec les requêtes)
SELECT
    queryid,
    substring(query, 1, 100) AS query_text,
    calls,
    total_exec_time,
    mean_exec_time
FROM pg_stat_statements  
ORDER BY calls DESC  
LIMIT 20;  
```

### 2. Fonction personnalisée de diagnostic

```sql
CREATE OR REPLACE FUNCTION diagnose_blocking()  
RETURNS TABLE(  
    blocked_pid INT,
    blocked_user TEXT,
    blocked_query TEXT,
    blocking_pid INT,
    blocking_user TEXT,
    blocking_query TEXT,
    wait_duration INTERVAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        blocked.pid AS blocked_pid,
        blocked_act.usename AS blocked_user,
        blocked_act.query AS blocked_query,
        blocking.pid AS blocking_pid,
        blocking_act.usename AS blocking_user,
        blocking_act.query AS blocking_query,
        age(now(), blocked_act.query_start) AS wait_duration
    FROM pg_locks blocked
    JOIN pg_stat_activity blocked_act ON blocked_act.pid = blocked.pid
    JOIN pg_locks blocking ON blocking.relation = blocked.relation
        AND blocking.pid != blocked.pid
        AND blocking.granted = true
    JOIN pg_stat_activity blocking_act ON blocking_act.pid = blocking.pid
    WHERE NOT blocked.granted
    ORDER BY blocked_act.query_start;
END;
$$ LANGUAGE plpgsql;

-- Utiliser
SELECT * FROM diagnose_blocking();
```

### 3. Script de monitoring automatique

```bash
#!/bin/bash
# deadlock_monitor.sh

# Se connecter à PostgreSQL et vérifier les deadlocks
DEADLOCKS=$(psql -U postgres -d mydb -tAc "
    SELECT deadlocks FROM pg_stat_database WHERE datname = 'mydb'
")

# Seuil d'alerte
THRESHOLD=50

if [ "$DEADLOCKS" -gt "$THRESHOLD" ]; then
    echo "ALERTE : $DEADLOCKS deadlocks détectés (seuil : $THRESHOLD)"

    # Envoyer une notification (email, Slack, etc.)
    # curl -X POST https://hooks.slack.com/... -d "Deadlocks: $DEADLOCKS"

    # Logger les détails
    psql -U postgres -d mydb -c "
        SELECT * FROM diagnose_blocking()
    " >> /var/log/postgresql/deadlocks_$(date +%Y%m%d).log
fi
```

### 4. Analyser les logs avec pgBadger

```bash
# Analyser les logs PostgreSQL pour identifier les patterns de deadlocks
pgbadger /var/log/postgresql/postgresql.log --prefix '%t [%p] %u@%d' -o report.html

# Ouvrir report.html et aller à la section "Locks"
```

---

## Patterns et anti-patterns

### ✅ Patterns recommandés

#### 1. Transaction Coordinator Pattern

```sql
-- Pour des opérations complexes impliquant plusieurs ressources
CREATE OR REPLACE FUNCTION transfer_money(
    from_account INT,
    to_account INT,
    amount NUMERIC
)
RETURNS void AS $$  
DECLARE  
    account1 INT;
    account2 INT;
BEGIN
    -- Assurer un ordre cohérent (le plus petit ID d'abord)
    IF from_account < to_account THEN
        account1 := from_account;
        account2 := to_account;
    ELSE
        account1 := to_account;
        account2 := from_account;
    END IF;

    -- Verrouiller dans l'ordre
    PERFORM * FROM comptes WHERE id = account1 FOR UPDATE;
    PERFORM * FROM comptes WHERE id = account2 FOR UPDATE;

    -- Effectuer le transfert
    UPDATE comptes SET solde = solde - amount WHERE id = from_account;
    UPDATE comptes SET solde = solde + amount WHERE id = to_account;
END;
$$ LANGUAGE plpgsql;
```

#### 2. Batch Update Pattern

```sql
-- Au lieu de mettre à jour ligne par ligne
-- ❌ MAUVAIS
FOR each_row IN SELECT id FROM produits WHERE categorie = 'X' LOOP
    UPDATE produits SET prix = prix * 1.1 WHERE id = each_row.id;
END LOOP;

-- ✅ BON : Une seule opération
UPDATE produits  
SET prix = prix * 1.1  
WHERE categorie = 'X';  
```

#### 3. Try-Lock Pattern

```sql
-- Essayer d'obtenir le verrou, abandonner si occupé
DO $$  
BEGIN  
    IF pg_try_advisory_xact_lock(123) THEN
        -- Traiter
        UPDATE jobs SET status = 'processing' WHERE id = 123;
    ELSE
        -- Occupé, passer au suivant
        RAISE NOTICE 'Job déjà en cours de traitement';
    END IF;
END $$;
```

### ❌ Anti-patterns à éviter

#### 1. The Death Spiral Anti-Pattern

```sql
-- ❌ TERRIBLE : Boucle infinie de retries sans limite
LOOP
    BEGIN
        -- Tenter l'opération
        UPDATE ...;
        EXIT;  -- Sortir si succès
    EXCEPTION
        WHEN deadlock_detected THEN
            -- Réessayer immédiatement, sans limite !
            -- Peut créer un storm de requêtes
    END;
END LOOP;
```

#### 2. The Hold Everything Anti-Pattern

```sql
-- ❌ TRÈS MAUVAIS : Verrouiller toutes les tables "au cas où"
BEGIN;  
LOCK TABLE table1 IN ACCESS EXCLUSIVE MODE;  
LOCK TABLE table2 IN ACCESS EXCLUSIVE MODE;  
LOCK TABLE table3 IN ACCESS EXCLUSIVE MODE;  
-- Bloque TOUTE l'application !
UPDATE table1 SET ... WHERE id = 1;  -- Une seule ligne !  
COMMIT;  
```

#### 3. The Random Order Anti-Pattern

```sql
-- ❌ MAUVAIS : Ordre aléatoire basé sur le hash ou l'insertion
UPDATE produits SET ...  
WHERE id IN (  
    SELECT id FROM produits_temp ORDER BY random()
);
```

---

## Checklist complète de prévention

### Avant le développement

- [ ] Former l'équipe sur les deadlocks et leur prévention  
- [ ] Établir des conventions de codage (ordre d'accès, timeouts)  
- [ ] Documenter les patterns recommandés

### Pendant le développement

- [ ] Toujours accéder aux tables/lignes dans un ordre cohérent  
- [ ] Trier les IDs avant SELECT FOR UPDATE ou UPDATE multiple  
- [ ] Garder les transactions aussi courtes que possible  
- [ ] Ne jamais ouvrir de transaction pendant une interaction utilisateur  
- [ ] Configurer des timeouts appropriés (lock_timeout, statement_timeout)  
- [ ] Implémenter une logique de retry avec backoff exponentiel  
- [ ] Utiliser NOWAIT ou SKIP LOCKED quand approprié  
- [ ] Éviter les SELECT FOR UPDATE inutiles

### Avant la mise en production

- [ ] Tester les scénarios de concurrence (charge)  
- [ ] Vérifier que les retries fonctionnent correctement  
- [ ] Configurer le monitoring (pg_stat_database, logs)  
- [ ] Définir des alertes (deadlock_threshold)  
- [ ] Créer un runbook pour gérer les incidents de deadlock  
- [ ] Activer log_lock_waits dans postgresql.conf

### En production

- [ ] Surveiller le nombre de deadlocks quotidiennement  
- [ ] Analyser les logs avec pgBadger hebdomadairement  
- [ ] Identifier et corriger les hotspots (tables/lignes très contendues)  
- [ ] Revoir les transactions longues (> 1 seconde)  
- [ ] Optimiser les patterns qui causent des deadlocks répétés

---

## Cas réels et solutions

### Cas 1 : E-commerce - Gestion du stock

**Problème** : Deadlocks fréquents lors du Black Friday avec des milliers de clients achetant simultanément.

**Cause** :
```sql
-- Transaction A
UPDATE produits SET stock = stock - 1 WHERE id = 100;  
UPDATE produits SET stock = stock - 1 WHERE id = 50;  

-- Transaction B (ordre inversé)
UPDATE produits SET stock = stock - 1 WHERE id = 50;  
UPDATE produits SET stock = stock - 1 WHERE id = 100;  
```

**Solution** :
```sql
-- Trier les IDs dans le panier AVANT de mettre à jour
WITH sorted_items AS (
    SELECT unnest(ARRAY[100, 50]) AS product_id
    ORDER BY product_id
)
UPDATE produits SET stock = stock - 1  
WHERE id IN (SELECT product_id FROM sorted_items);  
```

**Résultat** : Deadlocks réduits de 95%.

### Cas 2 : Plateforme de réservation

**Problème** : Deadlocks sur les tables `reservations` et `places`.

**Cause** : Transactions longues gardant des verrous pendant la validation de carte bancaire.

**Solution** :
```sql
-- Avant : Transaction longue
BEGIN;  
UPDATE places SET disponible = false WHERE id = 42;  
-- [Validation carte bancaire : 5 secondes]
INSERT INTO reservations ...;  
COMMIT;  

-- Après : Pré-validation, transaction courte
-- 1. Valider la carte HORS transaction
resultat_paiement = valider_carte(carte_bancaire)

-- 2. Transaction ultra-courte
BEGIN;  
UPDATE places SET disponible = false WHERE id = 42;  
INSERT INTO reservations (paiement_valide = true) ...;  
COMMIT;  
```

**Résultat** : Deadlocks éliminés, throughput multiplié par 3.

### Cas 3 : Système bancaire

**Problème** : Deadlocks sur les transferts entre comptes.

**Solution complète avec fonction** :
```sql
CREATE OR REPLACE FUNCTION safe_transfer(
    p_from_id INT,
    p_to_id INT,
    p_amount NUMERIC
)
RETURNS void AS $$  
DECLARE  
    v_first_id INT;
    v_second_id INT;
BEGIN
    -- Déterminer l'ordre (toujours du plus petit au plus grand)
    IF p_from_id < p_to_id THEN
        v_first_id := p_from_id;
        v_second_id := p_to_id;
    ELSE
        v_first_id := p_to_id;
        v_second_id := p_from_id;
    END IF;

    -- Verrouiller dans l'ordre
    PERFORM solde FROM comptes WHERE id = v_first_id FOR UPDATE;
    PERFORM solde FROM comptes WHERE id = v_second_id FOR UPDATE;

    -- Vérifier le solde
    IF (SELECT solde FROM comptes WHERE id = p_from_id) < p_amount THEN
        RAISE EXCEPTION 'Solde insuffisant';
    END IF;

    -- Effectuer le transfert
    UPDATE comptes SET solde = solde - p_amount WHERE id = p_from_id;
    UPDATE comptes SET solde = solde + p_amount WHERE id = p_to_id;

    -- Logger la transaction
    INSERT INTO historique_transferts (from_id, to_id, amount, date)
    VALUES (p_from_id, p_to_id, p_amount, NOW());
END;
$$ LANGUAGE plpgsql;

-- Utilisation avec retry
SELECT safe_transfer(100, 200, 50.00);
```

---

## Conclusion

La gestion des deadlocks dans PostgreSQL nécessite une approche **proactive** combinant prévention, détection et résolution :

### Les 3 piliers

1. **PRÉVENTION** (priorité #1)
   - Ordre cohérent d'accès ⭐⭐⭐
   - Transactions courtes ⭐⭐⭐
   - Timeouts appropriés ⭐⭐

2. **DÉTECTION** (surveillance)
   - Monitoring des métriques
   - Analyse des logs
   - Alertes automatiques

3. **RÉSOLUTION** (résilience)
   - Retry automatique avec backoff
   - Code d'erreur 40P01 géré
   - Logs détaillés

### Règles d'or

- ✅ **TOUJOURS** accéder aux ressources dans le même ordre  
- ✅ **TOUJOURS** garder les transactions courtes (< 100ms idéal)  
- ✅ **TOUJOURS** implémenter une logique de retry  
- ✅ **TOUJOURS** configurer des timeouts  
- ✅ **JAMAIS** garder une transaction ouverte pendant une interaction utilisateur  
- ✅ **JAMAIS** faire d'appels externes dans une transaction  
- ✅ **MONITORER** continuellement les deadlocks en production

### Impact de la prévention

Une bonne stratégie de prévention peut réduire les deadlocks de **90-99%** dans la plupart des applications. Les deadlocks résiduels (1-10%) sont gérés par les retries automatiques.

### Pour aller plus loin

- Tester vos transactions sous charge avec des outils comme pgbench
- Analyser régulièrement vos logs avec pgBadger
- Revoir périodiquement les hotspots de contention
- Former votre équipe aux bonnes pratiques

Les deadlocks sont **inévitables** dans un système concurrent, mais avec les bonnes stratégies, ils deviennent un **inconvénient mineur** plutôt qu'un problème majeur.

---

**Points clés à retenir :**

- 🔑 PostgreSQL détecte et résout automatiquement les deadlocks  
- 🔑 deadlock_timeout = 1s (défaut) déclenche la détection  
- 🔑 Ordre cohérent d'accès = prévention #1  
- 🔑 Transactions courtes = moins de risque  
- 🔑 Implémenter TOUJOURS un retry avec backoff exponentiel  
- 🔑 Code erreur 40P01 = deadlock dans votre application  
- 🔑 Monitorer pg_stat_database.deadlocks  
- 🔑 Utiliser lock_timeout et statement_timeout  
- 🔑 NOWAIT / SKIP LOCKED pour éviter les attentes  
- 🔑 Tester sous charge avant la mise en production

⏭️ [Indexation et Optimisation](/13-indexation-et-optimisation/README.md)
