🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.5. Gestion des verrous (Locks) : Types et Deadlocks

## Introduction : Pourquoi les verrous sont nécessaires

Imaginez une bibliothèque où plusieurs personnes veulent emprunter, retourner et consulter le même livre simultanément. Sans règles, le chaos s'installe : deux personnes prennent le même exemplaire en même temps, quelqu'un modifie la fiche de prêt pendant qu'un autre la lit, etc.

Les **verrous (locks)** sont le mécanisme que PostgreSQL utilise pour **coordonner l'accès concurrent** aux données. Ils garantissent que les opérations se déroulent de manière ordonnée et cohérente, même lorsque des centaines de transactions travaillent simultanément.

**Analogie centrale** : Les verrous sont comme les règles de circulation sur une route :
- 🚦 **Feu rouge** : Stop, attendez (verrou exclusif)  
- 🚦 **Feu vert** : Passez (pas de verrou)  
- 🚦 **Feu orange** : Certains peuvent passer, d'autres attendent (verrous partagés)

---

## MVCC et verrous : Une combinaison puissante

### Comment MVCC et verrous travaillent ensemble

Nous avons vu que PostgreSQL utilise **MVCC** (Multiversion Concurrency Control) pour gérer la concurrence. Mais MVCC ne suffit pas dans tous les cas :

**MVCC gère** :
- ✅ Les lectures concurrentes (multiples versions)  
- ✅ Les lectures pendant les écritures (snapshots)  
- ✅ La visibilité des données

**Les verrous gèrent** :
- 🔒 Les écritures concurrentes sur la **même ligne**  
- 🔒 Les modifications de structure (DDL)  
- 🔒 La coordination entre transactions qui modifient les mêmes données

**Exemple** :

```sql
-- Transaction A
BEGIN;  
UPDATE comptes SET solde = solde - 100 WHERE id = 1;  
-- PostgreSQL pose automatiquement un VERROU sur la ligne id=1

-- Transaction B (en parallèle)
BEGIN;  
UPDATE comptes SET solde = solde + 50 WHERE id = 1;  
-- Doit ATTENDRE que Transaction A libère le verrou
-- Sinon, les deux modifications pourraient s'écraser (Lost Update)
```

Sans verrou, les deux UPDATE pourraient se produire simultanément et créer une incohérence.

---

## Les niveaux de verrous dans PostgreSQL

PostgreSQL utilise un système de verrous **hiérarchique** avec plusieurs niveaux de granularité :

### 1. Verrous de table (Table-level locks)

Appliqués sur une table entière. Plusieurs modes existent.

### 2. Verrous de ligne (Row-level locks)

Appliqués sur des lignes individuelles. Plus fins, permettent plus de concurrence.

### 3. Verrous de page (Page-level locks)

Rarement utilisés directement, gérés en interne.

### 4. Verrous transactionnels (Transaction-level)

Verrous sur les identifiants de transaction pour MVCC.

### 5. Advisory Locks

Verrous applicatifs personnalisés que vous pouvez créer.

---

## Types de verrous de table

PostgreSQL définit **8 modes de verrous de table**, de moins restrictif à plus restrictif :

| Mode | Nom complet | Bloque | Usage typique |
|------|-------------|--------|---------------|
| **ACCESS SHARE** | Lecture simple | ACCESS EXCLUSIVE | SELECT |
| **ROW SHARE** | Lecture avec intention de mise à jour | EXCLUSIVE, ACCESS EXCLUSIVE | SELECT FOR UPDATE |
| **ROW EXCLUSIVE** | Modification de lignes | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | INSERT, UPDATE, DELETE |
| **SHARE UPDATE EXCLUSIVE** | Modification non-concurrente | Lui-même, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | VACUUM, CREATE INDEX CONCURRENTLY |
| **SHARE** | Lecture partagée protégée | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | CREATE INDEX |
| **SHARE ROW EXCLUSIVE** | Lecture partagée exclusive | ROW EXCLUSIVE, SHARE, SHARE UPDATE EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | Rarement utilisé |
| **EXCLUSIVE** | Exclusion des écritures | ROW SHARE, ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | REFRESH MATERIALIZED VIEW |
| **ACCESS EXCLUSIVE** | Exclusion totale | TOUS | DROP TABLE, TRUNCATE, ALTER TABLE (certains cas) |

### Comprendre la compatibilité des verrous

Deux transactions peuvent tenir des verrous sur la même table **si ces verrous sont compatibles**.

**Règle simple** :
- ✅ **ACCESS SHARE** (SELECT) est compatible avec presque tout  
- ✅ Plusieurs **ROW EXCLUSIVE** (UPDATE) sont compatibles entre eux  
- ❌ **ACCESS EXCLUSIVE** est incompatible avec tout

**Exemple de compatibilité** :

```sql
-- Transaction A
BEGIN;  
SELECT * FROM produits;  -- Pose un verrou ACCESS SHARE  
-- N'empêche pas les autres de lire ou modifier

-- Transaction B (en parallèle, compatible)
BEGIN;  
UPDATE produits SET prix = 100 WHERE id = 1;  -- Pose un verrou ROW EXCLUSIVE  
-- Fonctionne ! Les deux verrous sont compatibles

-- Transaction C (en parallèle, compatible aussi)
BEGIN;  
SELECT * FROM produits;  -- Pose aussi un ACCESS SHARE  
-- Fonctionne !
```

**Exemple d'incompatibilité** :

```sql
-- Transaction A
BEGIN;  
TRUNCATE TABLE produits;  -- Pose un verrou ACCESS EXCLUSIVE  
-- Bloque TOUT accès à la table !

-- Transaction B (en parallèle)
BEGIN;  
SELECT * FROM produits;  -- Pose un ACCESS SHARE  
-- BLOQUÉE ! Doit attendre que A termine
```

---

## Verrous de ligne (Row-Level Locks)

Les verrous de ligne sont **plus granulaires** : ils ne bloquent qu'une ligne spécifique, pas toute la table.

### Les quatre modes de verrous de ligne

| Mode | Commande | Bloque | Permet |
|------|----------|--------|--------|
| **FOR UPDATE** | SELECT ... FOR UPDATE | Autres FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE, FOR KEY SHARE | Lectures simples |
| **FOR NO KEY UPDATE** | SELECT ... FOR NO KEY UPDATE | FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE | FOR KEY SHARE, lectures |
| **FOR SHARE** | SELECT ... FOR SHARE | FOR UPDATE, FOR NO KEY UPDATE | FOR SHARE, FOR KEY SHARE, lectures |
| **FOR KEY SHARE** | SELECT ... FOR KEY SHARE | FOR UPDATE | FOR NO KEY UPDATE, FOR SHARE, FOR KEY SHARE, lectures |

### 1. FOR UPDATE : Le verrou exclusif

```sql
BEGIN;

SELECT * FROM commandes WHERE id = 123 FOR UPDATE;
-- Verrouille la ligne id=123 en mode EXCLUSIF
-- Autres transactions ne peuvent ni la modifier, ni la verrouiller

-- Traitement...
UPDATE commandes SET statut = 'Traité' WHERE id = 123;

COMMIT;  -- Libère le verrou
```

**Usage typique** : Empêcher les modifications concurrentes pendant un traitement critique.

**Exemple concret** : Réservation de place

```sql
-- Transaction A (Alice réserve)
BEGIN;

-- Verrouiller la place pour vérifier et modifier
SELECT disponible FROM places WHERE id = 42 FOR UPDATE;
-- Si disponible = true

UPDATE places SET disponible = false, client_id = 'Alice' WHERE id = 42;

COMMIT;

-- Transaction B (Bob essaie en parallèle)
BEGIN;

SELECT disponible FROM places WHERE id = 42 FOR UPDATE;
-- BLOQUÉ ! Doit attendre qu'Alice termine
-- Quand Alice commit, Bob obtient enfin le verrou
-- Mais maintenant disponible = false, donc Bob ne peut pas réserver

COMMIT;
```

### 2. FOR SHARE : Verrou partagé de lecture

```sql
BEGIN;

SELECT * FROM produits WHERE id = 1 FOR SHARE;
-- Verrouille la ligne en mode PARTAGÉ
-- Autres peuvent aussi lire (FOR SHARE)
-- Mais personne ne peut modifier (FOR UPDATE/UPDATE bloqués)

-- Lecture garantie stable
-- ...

COMMIT;
```

**Usage typique** : Garantir qu'une ligne ne changera pas pendant votre traitement, tout en permettant aux autres de la lire.

### 3. FOR NO KEY UPDATE : Mise à jour non-clé

Permet de mettre à jour des colonnes **non-clés** (pas la clé primaire) sans bloquer complètement les références de clés étrangères.

```sql
BEGIN;

SELECT * FROM produits WHERE id = 1 FOR NO KEY UPDATE;
-- Verrouille pour mise à jour, mais les FK peuvent toujours référencer

UPDATE produits SET description = 'Nouveau texte' WHERE id = 1;
-- OK car 'description' n'est pas une clé

COMMIT;
```

### 4. FOR KEY SHARE : Verrou faible

Le moins restrictif. Empêche uniquement les modifications de clé primaire.

```sql
BEGIN;

SELECT * FROM produits WHERE id = 1 FOR KEY SHARE;
-- Verrou très faible
-- Permet même des UPDATE de colonnes non-clés

COMMIT;
```

### Comparaison visuelle des verrous de ligne

```
Restrictivité croissante :  
FOR KEY SHARE < FOR SHARE < FOR NO KEY UPDATE < FOR UPDATE  

Lectures simples (SELECT)
    ↓
FOR KEY SHARE (empêche UPDATE de PK)
    ↓
FOR SHARE (empêche tout UPDATE)
    ↓
FOR NO KEY UPDATE (empêche UPDATE de PK et verrous exclusifs)
    ↓
FOR UPDATE (empêche tout verrouillage et modification)
```

---

## Verrous automatiques vs explicites

### Verrous automatiques

PostgreSQL pose **automatiquement** des verrous lors des opérations courantes :

```sql
-- SELECT : Pose automatiquement ACCESS SHARE sur la table
SELECT * FROM produits;

-- UPDATE : Pose automatiquement ROW EXCLUSIVE sur la table + verrou de ligne
UPDATE produits SET prix = 100 WHERE id = 1;

-- DELETE : Pareil que UPDATE
DELETE FROM produits WHERE id = 1;

-- INSERT : Pose ROW EXCLUSIVE sur la table
INSERT INTO produits (nom, prix) VALUES ('Nouveau', 50);

-- ALTER TABLE : Pose ACCESS EXCLUSIVE
ALTER TABLE produits ADD COLUMN description TEXT;
```

### Verrous explicites

Vous pouvez demander des verrous manuellement :

#### LOCK TABLE

```sql
BEGIN;

-- Verrouiller explicitement une table
LOCK TABLE produits IN ACCESS EXCLUSIVE MODE;
-- Personne d'autre ne peut accéder à la table

-- Faire vos opérations
UPDATE produits SET prix = prix * 1.1;

COMMIT;
```

**Modes disponibles** :

```sql
LOCK TABLE ma_table IN ACCESS SHARE MODE;  
LOCK TABLE ma_table IN ROW SHARE MODE;  
LOCK TABLE ma_table IN ROW EXCLUSIVE MODE;  
LOCK TABLE ma_table IN SHARE UPDATE EXCLUSIVE MODE;  
LOCK TABLE ma_table IN SHARE MODE;  
LOCK TABLE ma_table IN SHARE ROW EXCLUSIVE MODE;  
LOCK TABLE ma_table IN EXCLUSIVE MODE;  
LOCK TABLE TABLE ma_table IN ACCESS EXCLUSIVE MODE;  
```

**Quand utiliser LOCK TABLE** :
- ✅ Opérations en batch nécessitant un accès exclusif  
- ✅ Éviter les deadlocks dans des scénarios complexes  
- ⚠️ Rarement nécessaire (PostgreSQL gère bien automatiquement)

#### SELECT ... FOR UPDATE / FOR SHARE

```sql
BEGIN;

-- Verrouiller des lignes spécifiques
SELECT * FROM commandes WHERE client_id = 42 FOR UPDATE;
-- Verrouille toutes les commandes du client 42

-- Faire vos modifications
UPDATE commandes SET statut = 'En cours' WHERE client_id = 42;

COMMIT;
```

---

## Deadlocks (Interblocages)

### Qu'est-ce qu'un deadlock ?

Un **deadlock** (interblocage) se produit lorsque deux (ou plus) transactions s'attendent **mutuellement** pour des verrous, créant une situation de blocage circulaire où **aucune ne peut progresser**.

**Analogie** : Deux voitures arrivent face à face dans une rue étroite à sens unique. Chacune attend que l'autre recule. Personne ne peut avancer : c'est un interblocage.

### Exemple simple de deadlock

**Contexte** : Deux comptes bancaires (A et B).

**Transaction 1** :
```sql
BEGIN;

-- Verrouiller le compte A
UPDATE comptes SET solde = solde - 100 WHERE id = 'A';
-- ✅ Obtient le verrou sur la ligne A

-- [Petit délai]
```

**Transaction 2 (en parallèle)** :
```sql
BEGIN;

-- Verrouiller le compte B
UPDATE comptes SET solde = solde - 50 WHERE id = 'B';
-- ✅ Obtient le verrou sur la ligne B

-- [Petit délai]
```

**Transaction 1 (suite)** :
```sql
-- Essayer de verrouiller le compte B
UPDATE comptes SET solde = solde + 100 WHERE id = 'B';
-- ⏳ BLOQUÉ ! Transaction 2 détient le verrou sur B
-- Attend que Transaction 2 libère...
```

**Transaction 2 (suite)** :
```sql
-- Essayer de verrouiller le compte A
UPDATE comptes SET solde = solde + 50 WHERE id = 'A';
-- ⏳ BLOQUÉ ! Transaction 1 détient le verrou sur A
-- Attend que Transaction 1 libère...
```

**Situation finale** : DEADLOCK ! 🔴

```
Transaction 1 : détient A, attend B  
Transaction 2 : détient B, attend A  

     ┌───────────────┐
     │ Transaction 1 │
     │   détient A   │
     │   attend B    │
     └───────┬───────┘
             │
             ↓
     ┌───────────────┐
     │   Ligne B     │
     │ (détenue par  │
     │ Transaction 2)│
     └───────┬───────┘
             │
             ↓
     ┌───────────────┐
     │ Transaction 2 │
     │   détient B   │
     │   attend A    │
     └───────┬───────┘
             │
             ↓
     ┌───────────────┐
     │   Ligne A     │
     │ (détenue par  │
     │ Transaction 1)│
     └───────────────┘
             │
             │ Cycle !
             └─────────┐
                       │
                       ↓
               ♾️ DEADLOCK
```

### Comment PostgreSQL détecte les deadlocks

PostgreSQL a un **détecteur de deadlocks** qui s'exécute automatiquement :

1. **Timeout** : Par défaut, toutes les secondes (paramètre `deadlock_timeout`)  
2. **Détection** : PostgreSQL analyse le graphe des attentes de verrous  
3. **Résolution** : Si un cycle est détecté, PostgreSQL **annule** l'une des transactions

**Message d'erreur** :

```
ERROR: deadlock detected  
DETAIL: Process 12345 waits for ShareLock on transaction 67890; blocked by process 12346.  
Process 12346 waits for ShareLock on transaction 67889; blocked by process 12345.  
HINT: See server log for query details.  
```

La transaction annulée reçoit cette erreur et doit faire un **ROLLBACK**.

### Configuration du détecteur

```sql
-- Dans postgresql.conf ou via SET

-- Temps d'attente avant de vérifier les deadlocks (défaut: 1 seconde)
deadlock_timeout = 1s

-- Temps maximum d'attente pour un verrou avant erreur (0 = infini)
lock_timeout = 30s  -- Exemple: 30 secondes max

-- Temps maximum pour une instruction (0 = infini)
statement_timeout = 60s  -- Exemple: 1 minute max
```

---

## Types de deadlocks

### 1. Deadlock simple (2 transactions)

Le cas classique vu précédemment : T1 attend T2, T2 attend T1.

### 2. Deadlock circulaire (3+ transactions)

```
T1 détient A, attend B  
T2 détient B, attend C  
T3 détient C, attend A  

Cycle : T1 → T2 → T3 → T1
```

**Exemple** :

```sql
-- Transaction 1
BEGIN;  
UPDATE t1 SET ... WHERE id = 1;  -- Verrouille t1 ligne 1  
-- [attend]
UPDATE t2 SET ... WHERE id = 1;  -- Veut t2 ligne 1 (détenue par T2)

-- Transaction 2
BEGIN;  
UPDATE t2 SET ... WHERE id = 1;  -- Verrouille t2 ligne 1  
-- [attend]
UPDATE t3 SET ... WHERE id = 1;  -- Veut t3 ligne 1 (détenue par T3)

-- Transaction 3
BEGIN;  
UPDATE t3 SET ... WHERE id = 1;  -- Verrouille t3 ligne 1  
-- [attend]
UPDATE t1 SET ... WHERE id = 1;  -- Veut t1 ligne 1 (détenue par T1)

-- DEADLOCK !
```

### 3. Deadlock table vs ligne

```sql
-- Transaction 1
BEGIN;  
UPDATE produits SET prix = 100 WHERE id = 1;  -- Verrou ligne  
-- [attend]
ALTER TABLE produits ADD COLUMN description TEXT;  -- Veut verrou table (T2 l'a)

-- Transaction 2
BEGIN;  
ALTER TABLE produits ADD COLUMN stock INT;  -- Verrou table (ACCESS EXCLUSIVE)  
-- [attend]
UPDATE produits SET stock = 10 WHERE id = 1;  -- Veut verrou ligne (T1 l'a)

-- DEADLOCK !
```

### 4. Deadlock avec clés étrangères

```sql
-- Transaction 1
BEGIN;  
UPDATE commandes SET montant = 500 WHERE id = 100;  -- Verrouille commande 100  
-- [attend]
INSERT INTO lignes_commande (commande_id, produit_id)  
VALUES (200, 1);  -- Veut verrouiller commande 200 (FK, détenue par T2)  

-- Transaction 2
BEGIN;  
UPDATE commandes SET montant = 300 WHERE id = 200;  -- Verrouille commande 200  
-- [attend]
INSERT INTO lignes_commande (commande_id, produit_id)  
VALUES (100, 2);  -- Veut verrouiller commande 100 (FK, détenue par T1)  

-- DEADLOCK !
```

---

## Prévenir les deadlocks

### 1. Ordre cohérent d'accès aux ressources ⭐

**Règle d'or** : Toujours accéder aux tables/lignes dans le **même ordre** dans toutes les transactions.

**❌ Mauvais exemple** (ordre différent) :

```sql
-- Transaction A
UPDATE comptes SET ... WHERE id = 'A';  -- A puis B  
UPDATE comptes SET ... WHERE id = 'B';  

-- Transaction B
UPDATE comptes SET ... WHERE id = 'B';  -- B puis A (ordre inversé!)  
UPDATE comptes SET ... WHERE id = 'A';  

-- Risque de deadlock !
```

**✅ Bon exemple** (ordre identique) :

```sql
-- Transaction A
UPDATE comptes SET ... WHERE id = 'A';  -- A puis B  
UPDATE comptes SET ... WHERE id = 'B';  

-- Transaction B
UPDATE comptes SET ... WHERE id = 'A';  -- A puis B (même ordre)  
UPDATE comptes SET ... WHERE id = 'B';  

-- Pas de deadlock possible !
```

**Astuce** : Trier les IDs par ordre croissant

```sql
-- Si vous devez modifier plusieurs comptes
SELECT * FROM comptes WHERE id IN ('B', 'A', 'C') ORDER BY id FOR UPDATE;
-- Résultat ordonné : A, B, C (toujours le même ordre)
```

### 2. Transactions courtes

Plus une transaction est courte, moins elle a de chances de rentrer en conflit avec d'autres.

```sql
-- ❌ MAUVAIS : Transaction longue
BEGIN;  
UPDATE comptes SET ... WHERE id = 1;  
-- [Traitement long dans l'application : 30 secondes]
-- [Appels API externes]
-- [Calculs complexes]
UPDATE autre_table SET ...;  
COMMIT;  

-- ✅ BON : Transaction courte
-- [Faire tous les calculs AVANT]
BEGIN;  
UPDATE comptes SET ... WHERE id = 1;  
UPDATE autre_table SET ...;  
COMMIT;  -- Rapide !  
```

### 3. Utiliser des timeouts

```sql
-- Au niveau de la session
SET lock_timeout = '10s';  
SET statement_timeout = '30s';  

BEGIN;
-- Si un verrou n'est pas obtenu en 10s → erreur
-- Si la requête prend plus de 30s → erreur
UPDATE ...;  
COMMIT;  
```

### 4. Verrouillage explicite avec LOCK TABLE

Pour des opérations complexes, verrouillez les tables **au début** dans le bon ordre :

```sql
BEGIN;

-- Verrouiller toutes les tables nécessaires immédiatement
LOCK TABLE comptes IN SHARE ROW EXCLUSIVE MODE;  
LOCK TABLE transactions IN SHARE ROW EXCLUSIVE MODE;  

-- Faire toutes les opérations
UPDATE comptes ...;  
INSERT INTO transactions ...;  

COMMIT;
```

### 5. Utiliser SELECT ... FOR UPDATE NOWAIT

Au lieu d'attendre indéfiniment, échouer immédiatement si le verrou n'est pas disponible :

```sql
BEGIN;

SELECT * FROM produits WHERE id = 1 FOR UPDATE NOWAIT;
-- Si la ligne est déjà verrouillée → erreur immédiate
-- Pas d'attente → pas de risque de deadlock

COMMIT;

-- Gérer l'erreur dans l'application
EXCEPTION WHEN lock_not_available THEN
    -- Réessayer plus tard
```

**Variante avec timeout** :

```sql
BEGIN;

SELECT * FROM produits WHERE id = 1 FOR UPDATE SKIP LOCKED;
-- Saute les lignes déjà verrouillées
-- Utile pour des queues de travail

COMMIT;
```

### 6. Réduire le nombre de verrous

```sql
-- ❌ MAUVAIS : Verrouiller ligne par ligne dans une boucle
BEGIN;  
FOR each_id IN (SELECT id FROM comptes WHERE ...) LOOP  
    UPDATE comptes SET ... WHERE id = each_id;
END LOOP;  
COMMIT;  

-- ✅ BON : Une seule opération SQL
BEGIN;  
UPDATE comptes SET ... WHERE id IN (SELECT id FROM comptes WHERE ...);  
COMMIT;  
```

### 7. Éviter les transactions interactives

```sql
-- ❌ TRÈS MAUVAIS : Transaction ouverte pendant une interaction utilisateur
BEGIN;  
UPDATE comptes SET statut = 'En modification' WHERE id = 1;  

-- [Attente de validation utilisateur : peut prendre des minutes !]
-- [Pendant ce temps, le verrou est détenu]

UPDATE comptes SET solde = nouveau_solde WHERE id = 1;  
COMMIT;  

-- ✅ BON : Pas de transaction pendant l'interaction
-- 1. Lire les données
SELECT * FROM comptes WHERE id = 1;

-- 2. [Interaction utilisateur]

-- 3. Transaction courte pour écrire
BEGIN;  
UPDATE comptes SET solde = nouveau_solde WHERE id = 1;  
COMMIT;  
```

---

## Diagnostiquer les deadlocks

### Surveiller les deadlocks dans les logs

PostgreSQL enregistre les deadlocks dans ses logs si configuré :

```ini
# Dans postgresql.conf
log_lock_waits = on  # Logger les attentes de verrous longues  
deadlock_timeout = 1s  # Temps avant de vérifier les deadlocks  
```

**Exemple de log** :

```
2024-01-15 10:30:45.123 UTC [12345] ERROR:  deadlock detected
2024-01-15 10:30:45.123 UTC [12345] DETAIL:  Process 12345 waits for ShareLock on transaction 67890; blocked by process 12346.
        Process 12346 waits for ShareLock on transaction 67889; blocked by process 12345.
        Process 12345: UPDATE comptes SET solde = solde - 100 WHERE id = 'B';
        Process 12346: UPDATE comptes SET solde = solde + 50 WHERE id = 'A';
2024-01-15 10:30:45.123 UTC [12345] HINT:  See server log for query details.
2024-01-15 10:30:45.123 UTC [12345] CONTEXT:  while updating tuple (0,42) in relation "comptes"
```

### Identifier les transactions en attente de verrous

```sql
-- Vue pg_locks : Tous les verrous actifs
SELECT
    pid,
    locktype,
    relation::regclass AS table_name,
    mode,
    granted
FROM pg_locks  
WHERE NOT granted;  -- Verrous non accordés (en attente)  
```

### Identifier qui bloque qui

```sql
-- Requête complète pour voir les blocages
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query,
    blocked_activity.application_name AS blocked_app
FROM pg_catalog.pg_locks blocked_locks  
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid  
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
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid  
WHERE NOT blocked_locks.granted;  
```

### Fonction PostgreSQL pour voir les blocages (simplifié)

```sql
-- Identifier les PID qui bloquent d'autres transactions
SELECT pg_blocking_pids(12345);
-- Retourne un tableau des PID qui bloquent le processus 12345
-- Exemple : {12346, 12347}
```

### Tuer une transaction bloquante

Si une transaction bloque tout et ne répond plus :

```sql
-- Terminer gentiment (laisse la transaction faire ROLLBACK)
SELECT pg_cancel_backend(12346);

-- Tuer brutalement (si pg_cancel_backend ne fonctionne pas)
SELECT pg_terminate_backend(12346);
```

**⚠️ Attention** : `pg_terminate_backend` ferme la connexion brutalement. À utiliser en dernier recours.

---

## Advisory Locks (Verrous applicatifs)

Les **advisory locks** sont des verrous que vous créez **manuellement** pour coordonner des opérations dans votre application. PostgreSQL ne les utilise pas automatiquement : c'est à vous de les gérer.

### Pourquoi utiliser des advisory locks ?

- ✅ Coordonner des tâches distribuées (jobs, workers)  
- ✅ Implémenter des mutex au niveau base de données  
- ✅ Empêcher l'exécution simultanée d'une opération critique  
- ✅ Créer des sémaphores ou des compteurs partagés

### Types d'advisory locks

#### 1. Advisory locks de session

Actifs pendant toute la session (connexion).

```sql
-- Obtenir un verrou exclusif
SELECT pg_advisory_lock(123);
-- Si le verrou est déjà pris, ATTEND qu'il soit libéré

-- [Faire des opérations critiques]

-- Libérer le verrou
SELECT pg_advisory_unlock(123);
```

**Variante non-bloquante** :

```sql
-- Essayer d'obtenir le verrou sans attendre
SELECT pg_try_advisory_lock(123);
-- Retourne : true (succès) ou false (déjà pris)

IF success THEN
    -- Faire les opérations
    SELECT pg_advisory_unlock(123);
ELSE
    -- Le verrou est déjà pris, abandonner ou réessayer
END IF;
```

#### 2. Advisory locks transactionnels

Libérés automatiquement à la fin de la transaction.

```sql
BEGIN;

-- Obtenir un verrou transactionnel
SELECT pg_advisory_xact_lock(456);
-- Sera libéré automatiquement au COMMIT ou ROLLBACK

-- [Opérations]

COMMIT;  -- Le verrou est libéré automatiquement
```

### Utiliser deux nombres pour les clés

PostgreSQL permet d'utiliser deux entiers de 32 bits au lieu d'un seul de 64 bits :

```sql
-- Verrou basé sur (table_id, row_id)
SELECT pg_advisory_lock(42, 100);
-- Équivalent à un verrou sur le "tuple" (42, 100)

-- Libération
SELECT pg_advisory_unlock(42, 100);
```

### Exemple pratique : Job queue distribuée

Empêcher deux workers de traiter le même job :

```sql
-- Worker 1 essaie de prendre un job
BEGIN;

-- Essayer d'obtenir le verrou sur le job 789
SELECT pg_try_advisory_xact_lock(789);
-- Retourne : true

-- Si true, traiter le job
UPDATE jobs SET status = 'En cours' WHERE id = 789;

-- [Traitement du job]

UPDATE jobs SET status = 'Terminé' WHERE id = 789;

COMMIT;  -- Verrou libéré automatiquement


-- Worker 2 essaie en parallèle
BEGIN;

SELECT pg_try_advisory_xact_lock(789);
-- Retourne : false (déjà pris par Worker 1)

-- Passer au job suivant

COMMIT;
```

### Verrous partagés vs exclusifs

```sql
-- Verrou EXCLUSIF (par défaut)
SELECT pg_advisory_lock(123);
-- Personne d'autre ne peut obtenir ce verrou

-- Verrou PARTAGÉ
SELECT pg_advisory_lock_shared(123);
-- Plusieurs processus peuvent obtenir le verrou partagé
-- Mais un verrou exclusif est bloqué
```

### Lister les advisory locks actifs

```sql
SELECT
    locktype,
    classid,
    objid,
    pid,
    mode,
    granted
FROM pg_locks  
WHERE locktype = 'advisory';  
```

---

## Bonnes pratiques générales sur les verrous

### 1. Comprendre les verrous automatiques

Connaissez quel type de verrou pose chaque opération :

```sql
SELECT → ACCESS SHARE (lecture)  
UPDATE/DELETE → ROW EXCLUSIVE (table) + verrou ligne  
INSERT → ROW EXCLUSIVE (table)  
CREATE INDEX → SHARE  
TRUNCATE/DROP → ACCESS EXCLUSIVE  
ALTER TABLE → Varie selon l'opération  
```

### 2. Éviter les opérations DDL en production aux heures de pointe

```sql
-- ❌ DANGEREUX en production active
ALTER TABLE produits ADD COLUMN description TEXT;
-- Pose un verrou ACCESS EXCLUSIVE → bloque TOUT

-- ✅ MEILLEUR : Pendant une fenêtre de maintenance
-- ou utiliser CREATE INDEX CONCURRENTLY (pour les index)
```

### 3. Utiliser CREATE INDEX CONCURRENTLY

```sql
-- ❌ Bloque les écritures
CREATE INDEX idx_produits_nom ON produits(nom);

-- ✅ N'empêche pas les écritures (plus lent, mais non-bloquant)
CREATE INDEX CONCURRENTLY idx_produits_nom ON produits(nom);
```

### 4. Monitorer les locks longs

```sql
-- Trouver les verrous qui durent longtemps
SELECT
    pid,
    usename,
    pg_blocking_pids(pid) as blocked_by,
    query as query_text,
    age(clock_timestamp(), query_start) AS duration
FROM pg_stat_activity  
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'  
  AND state = 'active';
```

### 5. Configurer des alertes

Surveillez :
- Nombre de transactions en attente de verrous
- Fréquence des deadlocks
- Durée des verrous
- Transactions en idle in transaction

```sql
-- Transactions en "idle in transaction" (danger !)
SELECT
    pid,
    usename,
    state,
    query,
    age(clock_timestamp(), state_change) AS idle_duration
FROM pg_stat_activity  
WHERE state = 'idle in transaction'  
  AND age(clock_timestamp(), state_change) > interval '5 minutes';
```

### 6. Implémenter une logique de retry

```python
import time  
import psycopg2  

def execute_with_deadlock_retry(conn, operation, max_retries=3):
    """
    Exécute une opération avec retry en cas de deadlock
    """
    for attempt in range(max_retries):
        try:
            operation(conn)
            conn.commit()
            return True

        except psycopg2.extensions.TransactionRollbackError as e:
            # Deadlock détecté
            conn.rollback()

            if attempt == max_retries - 1:
                raise Exception(f"Deadlock après {max_retries} tentatives") from e

            # Attente avec backoff exponentiel
            wait_time = 0.1 * (2 ** attempt)
            time.sleep(wait_time)

        except Exception as e:
            conn.rollback()
            raise

# Utilisation
def mon_transfert(conn):
    cur = conn.cursor()
    # Accès dans un ordre cohérent (A puis B)
    cur.execute("UPDATE comptes SET solde = solde - 100 WHERE id = 'A'")
    cur.execute("UPDATE comptes SET solde = solde + 100 WHERE id = 'B'")

execute_with_deadlock_retry(conn, mon_transfert)
```

---

## Cas d'étude : Scénarios réels

### Cas 1 : E-commerce - Gestion de stock

**Problème** : Deux clients achètent le dernier article simultanément.

**Solution avec verrous** :

```sql
BEGIN;

-- Verrouiller la ligne pour vérifier et décrémenter atomiquement
SELECT stock FROM produits WHERE id = 123 FOR UPDATE;

-- Vérifier le stock
IF stock >= quantite_demandee THEN
    UPDATE produits
    SET stock = stock - quantite_demandee
    WHERE id = 123;

    -- Créer la commande
    INSERT INTO commandes ...;
ELSE
    RAISE EXCEPTION 'Stock insuffisant';
END IF;

COMMIT;
```

### Cas 2 : Compteur distribué

**Problème** : Incrémenter un compteur depuis plusieurs workers.

**❌ Mauvaise approche** (Lost Update) :

```sql
-- Worker 1
SELECT compteur FROM stats WHERE id = 1;  -- Lit : 100
-- [Calcul : 100 + 1]
UPDATE stats SET compteur = 101 WHERE id = 1;

-- Worker 2 (en parallèle)
SELECT compteur FROM stats WHERE id = 1;  -- Lit : 100 aussi !
-- [Calcul : 100 + 1]
UPDATE stats SET compteur = 101 WHERE id = 1;  -- Écrase !

-- Résultat final : 101 au lieu de 102
```

**✅ Bonne approche** (Atomique) :

```sql
-- Les deux workers
UPDATE stats SET compteur = compteur + 1 WHERE id = 1;
-- PostgreSQL gère l'atomicité
-- Résultat correct : 102
```

### Cas 3 : File d'attente de jobs

**Problème** : Plusieurs workers doivent traiter des jobs sans conflit.

**Solution avec SKIP LOCKED** :

```sql
-- Chaque worker exécute ceci
BEGIN;

-- Prendre un job disponible en sautant les verrouillés
SELECT * FROM jobs  
WHERE status = 'En attente'  
ORDER BY priority DESC, created_at ASC  
LIMIT 1  
FOR UPDATE SKIP LOCKED;  

-- Si un job est trouvé
IF FOUND THEN
    -- Le traiter
    UPDATE jobs SET status = 'En cours' WHERE id = job_id;

    -- [Traitement]

    UPDATE jobs SET status = 'Terminé' WHERE id = job_id;
END IF;

COMMIT;
```

---

## Résumé et checklist

### Types de verrous essentiels

| Type | Niveau | Usage | Obtention |
|------|--------|-------|-----------|
| **ACCESS SHARE** | Table | SELECT | Automatique |
| **ROW EXCLUSIVE** | Table | INSERT/UPDATE/DELETE | Automatique |
| **ACCESS EXCLUSIVE** | Table | DDL (DROP, TRUNCATE) | Automatique |
| **FOR UPDATE** | Ligne | Modification exclusive | Explicite (SELECT FOR UPDATE) |
| **FOR SHARE** | Ligne | Lecture protégée | Explicite (SELECT FOR SHARE) |
| **Advisory Lock** | Applicatif | Coordination custom | Explicite (pg_advisory_lock) |

### Checklist anti-deadlock

- ✅ Accéder aux ressources dans un **ordre cohérent** (trier les IDs)  
- ✅ Garder les transactions **courtes**  
- ✅ Éviter les transactions **interactives**  
- ✅ Utiliser **NOWAIT** ou **SKIP LOCKED** quand approprié  
- ✅ Configurer des **timeouts** (lock_timeout, statement_timeout)  
- ✅ Implémenter une **logique de retry** avec backoff exponentiel  
- ✅ **Monitorer** les deadlocks dans les logs  
- ✅ Préférer les opérations **atomiques SQL** au Read-Modify-Write  
- ✅ Éviter les **DDL** pendant les heures de pointe

### Signaux d'alarme

- 🚨 Transactions en "idle in transaction" pour longtemps  
- 🚨 Nombreux processus en attente de verrous  
- 🚨 Deadlocks fréquents dans les logs  
- 🚨 Lock timeout dépassé régulièrement  
- 🚨 Requêtes bloquées pendant plusieurs minutes  
- 🚨 Opérations DDL en production active

---

## Conclusion

Les verrous sont un mécanisme essentiel de PostgreSQL pour coordonner l'accès concurrent aux données. Bien comprendre leur fonctionnement vous permet de :

- ✅ Concevoir des transactions robustes sans Lost Updates  
- ✅ Éviter les deadlocks par une approche méthodique  
- ✅ Diagnostiquer et résoudre les problèmes de performance liés aux blocages  
- ✅ Utiliser les bons types de verrous pour chaque situation  
- ✅ Implémenter une coordination applicative avec Advisory Locks

**Principes clés** :

1. **MVCC** gère la visibilité, **verrous** gèrent les modifications concurrentes  
2. Les verrous de **ligne** permettent plus de concurrence que les verrous de **table**  
3. Les **deadlocks** sont détectés et résolus automatiquement, mais il faut les **prévenir**  
4. L'**ordre d'accès cohérent** est la meilleure défense contre les deadlocks  
5. Les **advisory locks** offrent une flexibilité pour la coordination applicative

Dans la prochaine section (12.6), nous explorerons les **Advisory Locks en profondeur** et des patterns avancés de coordination.

---

**Points clés à retenir :**

- 🔑 MVCC + Verrous = Gestion complète de la concurrence  
- 🔑 8 modes de verrous de table, du moins au plus restrictif  
- 🔑 FOR UPDATE = verrou exclusif de ligne  
- 🔑 Deadlock = attente circulaire, détecté automatiquement  
- 🔑 Ordre cohérent d'accès = prévention #1 des deadlocks  
- 🔑 Transactions courtes = moins de conflits  
- 🔑 SKIP LOCKED = utile pour les queues de travail  
- 🔑 Advisory locks = coordination applicative personnalisée

⏭️ [Advisory Locks : Verrouillage applicatif personnalisé](/12-concurrence-et-transactions/06-advisory-locks.md)
