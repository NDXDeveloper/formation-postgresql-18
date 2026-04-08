🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.3. Niveaux d'isolation ANSI (Read Uncommitted, Read Committed, Repeatable Read, Serializable)

## Introduction : Pourquoi les niveaux d'isolation ?

Imaginez que vous êtes dans une bibliothèque où plusieurs personnes consultent et modifient les mêmes livres simultanément. Comment garantir que chacun voit des informations cohérentes ? Comment éviter qu'une personne lise une page pendant qu'une autre est en train de la modifier ?

Les **niveaux d'isolation** définissent les règles qui déterminent **ce qu'une transaction peut voir** des modifications effectuées par d'autres transactions concurrentes. C'est un équilibre constant entre :

- 🔒 **Cohérence** : Garantir que les données lues sont valides et cohérentes  
- ⚡ **Performance** : Permettre un maximum de transactions concurrentes  
- 🎯 **Simplicité** : Éviter les comportements surprenants pour les développeurs

Plus le niveau d'isolation est strict, plus la cohérence est forte, mais plus la performance peut être impactée.

---

## Les quatre niveaux d'isolation ANSI

Le standard SQL ANSI définit quatre niveaux d'isolation, du moins strict au plus strict :

1. **Read Uncommitted** (Lecture non validée)  
2. **Read Committed** (Lecture validée) ⭐ *Niveau par défaut dans PostgreSQL*  
3. **Repeatable Read** (Lecture répétable)  
4. **Serializable** (Sérialisable)

**Note importante** : PostgreSQL ne supporte pas réellement Read Uncommitted. Si vous le demandez, PostgreSQL utilise automatiquement Read Committed à la place.

---

## Les anomalies transactionnelles

Avant de détailler chaque niveau d'isolation, comprenons les trois types d'**anomalies** (comportements problématiques) qu'ils cherchent à éviter :

### 1. Dirty Read (Lecture sale)

Une transaction lit des données **non validées** (non commitées) d'une autre transaction.

**Exemple** :

```sql
-- Transaction A
BEGIN;  
UPDATE comptes SET solde = 0 WHERE id = 1;  -- Pas encore commité !  

-- Transaction B (en parallèle)
BEGIN;  
SELECT solde FROM comptes WHERE id = 1;  
-- Voit : 0 (alors que Transaction A pourrait faire ROLLBACK !)
```

**Problème** : Si Transaction A fait un ROLLBACK, Transaction B a lu des données qui "n'ont jamais existé" réellement.

### 2. Non-Repeatable Read (Lecture non répétable)

Une transaction lit la même ligne **deux fois** et obtient des **résultats différents** car une autre transaction a modifié et validé la ligne entre-temps.

**Exemple** :

```sql
-- Transaction A
BEGIN;  
SELECT solde FROM comptes WHERE id = 1;  -- Voit : 1000€  

-- Transaction B (en parallèle)
BEGIN;  
UPDATE comptes SET solde = 500 WHERE id = 1;  
COMMIT;  

-- Transaction A (suite)
SELECT solde FROM comptes WHERE id = 1;  -- Voit maintenant : 500€ (!!)  
COMMIT;  
```

**Problème** : La même requête dans la même transaction retourne des résultats différents.

### 3. Phantom Read (Lecture fantôme)

Une transaction exécute la même requête **deux fois** et obtient un **nombre différent de lignes** car une autre transaction a inséré ou supprimé des lignes entre-temps.

**Exemple** :

```sql
-- Transaction A
BEGIN;  
SELECT COUNT(*) FROM commandes WHERE client_id = 42;  -- Voit : 5 commandes  

-- Transaction B (en parallèle)
BEGIN;  
INSERT INTO commandes (client_id, montant) VALUES (42, 100);  
COMMIT;  

-- Transaction A (suite)
SELECT COUNT(*) FROM commandes WHERE client_id = 42;  -- Voit maintenant : 6 commandes (!!)  
COMMIT;  
```

**Problème** : De nouvelles lignes "fantômes" apparaissent (ou disparaissent) au milieu de la transaction.

---

## Tableau récapitulatif des niveaux d'isolation

| Niveau d'isolation | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------------------|------------|---------------------|--------------|-------------|
| **Read Uncommitted** | ❌ Possible | ❌ Possible | ❌ Possible | ⚡⚡⚡ Maximale |
| **Read Committed** | ✅ Impossible | ❌ Possible | ❌ Possible | ⚡⚡ Très bonne |
| **Repeatable Read** | ✅ Impossible | ✅ Impossible | ✅ Impossible* | ⚡ Bonne |
| **Serializable** | ✅ Impossible | ✅ Impossible | ✅ Impossible | ⚠️ Variable |

*PostgreSQL va plus loin que le standard ANSI : même Repeatable Read empêche les Phantom Reads grâce au MVCC.

---

## 1. Read Uncommitted (Lecture non validée)

### Définition

Le niveau le moins strict. En théorie, une transaction peut lire les modifications **non validées** d'autres transactions (Dirty Reads).

### Comportement dans PostgreSQL

**Important** : PostgreSQL ne supporte pas réellement Read Uncommitted. Si vous le demandez, PostgreSQL se comporte comme en **Read Committed**.

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- PostgreSQL traite cela comme READ COMMITTED
```

### Pourquoi PostgreSQL ne le supporte pas ?

Le MVCC de PostgreSQL empêche naturellement les Dirty Reads. Il serait techniquement complexe et dangereux de permettre la lecture de données non validées.

### Quand l'utiliser ?

**Jamais** dans PostgreSQL, puisqu'il n'est pas réellement disponible.

---

## 2. Read Committed (Lecture validée) ⭐

### Définition

C'est le **niveau par défaut** dans PostgreSQL. Une transaction ne voit que les données **validées** (commitées) par d'autres transactions.

### Caractéristiques principales

1. ✅ **Pas de Dirty Reads** : On ne lit jamais de données non validées  
2. ❌ **Non-Repeatable Reads possibles** : La même requête peut retourner des résultats différents  
3. ❌ **Phantom Reads possibles** : De nouvelles lignes peuvent apparaître entre deux requêtes  
4. 📸 **Snapshot par requête** : Chaque requête voit un snapshot de la base pris au moment où elle commence

### Comment ça fonctionne

En Read Committed, PostgreSQL prend un **nouveau snapshot au début de chaque requête SQL** dans la transaction.

```sql
BEGIN;  -- Mode Read Committed par défaut

-- Requête 1 : Snapshot pris à 10:00:00
SELECT * FROM produits WHERE categorie = 'Électronique';

-- [Une autre transaction modifie et commit des produits]

-- Requête 2 : NOUVEAU snapshot pris à 10:00:05
SELECT * FROM produits WHERE categorie = 'Électronique';
-- Peut voir les modifications commitées entre les deux requêtes !

COMMIT;
```

### Exemple détaillé : Non-Repeatable Read

**Contexte** : Deux utilisateurs travaillent sur le même compte bancaire.

**Transaction A (consultation de solde)** :
```sql
BEGIN;  -- Read Committed (par défaut)

-- Première lecture
SELECT solde FROM comptes WHERE id = 1;
-- Résultat : 1000€

-- [Attente de 30 secondes...]
```

**Transaction B (retrait d'argent)** :
```sql
BEGIN;

UPDATE comptes SET solde = solde - 200 WHERE id = 1;  
COMMIT;  -- Validé !  
```

**Transaction A (suite)** :
```sql
-- Deuxième lecture (même transaction)
SELECT solde FROM comptes WHERE id = 1;
-- Résultat : 800€ (!!)  <-- Changement !

COMMIT;
```

**Analyse** : La même requête dans la même transaction retourne des résultats différents. C'est un **Non-Repeatable Read**.

### Exemple détaillé : Phantom Read

**Transaction A (comptage de commandes)** :
```sql
BEGIN;

SELECT COUNT(*) FROM commandes WHERE statut = 'En attente';
-- Résultat : 10 commandes

-- [Attente...]
```

**Transaction B (nouvelle commande)** :
```sql
BEGIN;

INSERT INTO commandes (client_id, statut) VALUES (42, 'En attente');  
COMMIT;  
```

**Transaction A (suite)** :
```sql
SELECT COUNT(*) FROM commandes WHERE statut = 'En attente';
-- Résultat : 11 commandes (!!)  <-- Une ligne "fantôme" est apparue !

COMMIT;
```

**Analyse** : Une nouvelle ligne est apparue entre les deux requêtes identiques. C'est un **Phantom Read**.

### Quand utiliser Read Committed ?

- ✅ **Par défaut pour la plupart des applications**  
- ✅ Applications web classiques (CRUD)  
- ✅ APIs REST standard  
- ✅ Workloads OLTP (transactions courtes)  
- ✅ Quand la performance est prioritaire

- ❌ Rapports nécessitant une cohérence stricte  
- ❌ Calculs financiers complexes avec plusieurs lectures  
- ❌ Logique métier sensible aux changements entre requêtes

### Configuration

C'est le niveau par défaut, donc aucune configuration nécessaire :

```sql
BEGIN;  -- Utilise automatiquement READ COMMITTED
-- ou explicitement :
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 3. Repeatable Read (Lecture répétable)

### Définition

Une transaction voit un **snapshot cohérent** de la base de données pris **au début de la transaction** (au premier SELECT). Les modifications des autres transactions ne sont jamais visibles.

### Caractéristiques principales

1. ✅ **Pas de Dirty Reads**  
2. ✅ **Pas de Non-Repeatable Reads** : Une requête répétée retourne toujours les mêmes résultats  
3. ✅ **Pas de Phantom Reads** (dans PostgreSQL) : Le nombre de lignes reste stable  
4. 📸 **Snapshot unique** : Un seul snapshot pour toute la transaction

### Comment ça fonctionne

En Repeatable Read, PostgreSQL prend un **snapshot au début de la transaction** et le conserve jusqu'à la fin.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Premier SELECT : Snapshot pris ici et FIGÉ pour toute la transaction
SELECT * FROM produits;

-- [Autres transactions modifient et commit des produits]

-- Deuxième SELECT : Voit EXACTEMENT les mêmes données qu'avant
SELECT * FROM produits;  -- Aucun changement visible !

COMMIT;
```

### Exemple détaillé : Protection contre Non-Repeatable Read

Reprenons l'exemple du compte bancaire avec Repeatable Read :

**Transaction A (consultation)** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Première lecture (snapshot pris ici)
SELECT solde FROM comptes WHERE id = 1;
-- Résultat : 1000€
```

**Transaction B (retrait)** :
```sql
BEGIN;

UPDATE comptes SET solde = solde - 200 WHERE id = 1;  
COMMIT;  -- Validé !  
```

**Transaction A (suite)** :
```sql
-- Deuxième lecture (même snapshot figé)
SELECT solde FROM comptes WHERE id = 1;
-- Résultat : TOUJOURS 1000€ !  <-- Aucun changement visible

COMMIT;
```

**Analyse** : Grâce au snapshot figé, Transaction A voit toujours le même solde, même si Transaction B a modifié le compte. C'est la garantie de **lecture répétable**.

### Exemple détaillé : Protection contre Phantom Read

**Transaction A** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*) FROM commandes WHERE statut = 'En attente';
-- Résultat : 10 commandes (snapshot figé)
```

**Transaction B** :
```sql
BEGIN;

INSERT INTO commandes (client_id, statut) VALUES (42, 'En attente');  
COMMIT;  
```

**Transaction A (suite)** :
```sql
SELECT COUNT(*) FROM commandes WHERE statut = 'En attente';
-- Résultat : TOUJOURS 10 commandes !  <-- Pas de ligne fantôme

COMMIT;
```

**Analyse** : La nouvelle ligne insérée par Transaction B n'est pas visible dans Transaction A car elle utilise un snapshot figé.

### Cas particulier : Erreurs de sérialisation

Avec Repeatable Read, certaines opérations peuvent échouer avec une **erreur de sérialisation** :

```sql
-- Transaction A
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT solde FROM comptes WHERE id = 1;  -- Voit : 1000€

-- Transaction B (en parallèle)
BEGIN;  
UPDATE comptes SET solde = solde - 200 WHERE id = 1;  
COMMIT;  -- Change le solde à 800€  

-- Transaction A (suite)
UPDATE comptes SET solde = solde + 500 WHERE id = 1;
-- ERREUR: could not serialize access due to concurrent update

ROLLBACK;  -- Obligé d'annuler la transaction
```

**Explication** : Transaction A essaie de modifier une ligne qui a été changée depuis son snapshot. PostgreSQL détecte le conflit et rejette l'opération.

**Solution applicative** : Implémenter une logique de **retry** (réessayer la transaction).

```python
# Pseudo-code
max_retries = 3  
for attempt in range(max_retries):  
    try:
        # Tenter la transaction
        conn.execute("BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ")
        # ... opérations ...
        conn.execute("COMMIT")
        break  # Succès !
    except SerializationError:
        conn.execute("ROLLBACK")
        if attempt == max_retries - 1:
            raise  # Échec après tous les essais
        time.sleep(0.1 * (2 ** attempt))  # Backoff exponentiel
```

### Quand utiliser Repeatable Read ?

- ✅ **Rapports financiers** nécessitant une cohérence stricte  
- ✅ **Analyses et statistiques** où les données ne doivent pas changer  
- ✅ **Calculs complexes** avec plusieurs lectures sur les mêmes données  
- ✅ **Exports de données** nécessitant une vue cohérente  
- ✅ **Audits** où l'on doit garantir qu'on voit un état figé

- ❌ Applications avec beaucoup de contentions (risque d'erreurs de sérialisation)  
- ❌ Opérations simples CRUD qui peuvent tolérer des changements

### Configuration

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- ou via SET pour la session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

---

## 4. Serializable (Sérialisable)

### Définition

Le niveau le plus strict. PostgreSQL **garantit** que l'exécution simultanée de transactions sérialisables produit le **même résultat** que si elles avaient été exécutées **une par une**, en série.

### Caractéristiques principales

1. ✅ **Pas de Dirty Reads**  
2. ✅ **Pas de Non-Repeatable Reads**  
3. ✅ **Pas de Phantom Reads**  
4. ✅ **Pas d'anomalies d'écriture** (Write Skew)  
5. ⚠️ **Erreurs de sérialisation fréquentes** : Les transactions peuvent être rejetées  
6. 🔍 **Détection de conflits** : PostgreSQL analyse les dépendances entre transactions

### Comment ça fonctionne

Serializable utilise une technique appelée **Serializable Snapshot Isolation (SSI)**. PostgreSQL :

1. Prend un snapshot comme en Repeatable Read  
2. **Surveille** les accès en lecture et écriture  
3. **Détecte** les conflits qui pourraient violer la sérialisation  
4. **Rejette** une transaction en cas de conflit avec une erreur

### Exemple : Write Skew (anomalie d'écriture)

Cette anomalie n'est possible ni en Read Committed ni en Repeatable Read, mais Serializable la détecte.

**Contexte** : Une règle métier impose qu'il doit toujours y avoir au moins 1 médecin de garde.

**État initial** :
```sql
-- 2 médecins de garde
SELECT COUNT(*) FROM gardes WHERE en_service = true;
-- Résultat : 2 (Alice et Bob)
```

**Transaction A (Alice quitte)** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- Pas assez strict !

-- Vérifier qu'il y a au moins 2 médecins
SELECT COUNT(*) FROM gardes WHERE en_service = true;  -- Voit : 2

-- Alice peut partir (il restera Bob)
UPDATE gardes SET en_service = false WHERE medecin = 'Alice';

COMMIT;
```

**Transaction B (Bob quitte, EN PARALLÈLE)** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Vérifier qu'il y a au moins 2 médecins
SELECT COUNT(*) FROM gardes WHERE en_service = true;  -- Voit : 2 aussi !

-- Bob peut partir (il restera Alice... pense-t-il)
UPDATE gardes SET en_service = false WHERE medecin = 'Bob';

COMMIT;
```

**Résultat catastrophique** :
```sql
SELECT COUNT(*) FROM gardes WHERE en_service = true;
-- Résultat : 0 (!!)  <-- Violation de la règle métier !
```

Les deux transactions ont réussi, mais le résultat viole la contrainte métier. C'est un **Write Skew**.

**Avec Serializable** :

```sql
-- Transaction A
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*) FROM gardes WHERE en_service = true;  -- 2  
UPDATE gardes SET en_service = false WHERE medecin = 'Alice';  

COMMIT;  -- Succès

-- Transaction B (en parallèle)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*) FROM gardes WHERE en_service = true;  -- 2  
UPDATE gardes SET en_service = false WHERE medecin = 'Bob';  

COMMIT;
-- ERREUR: could not serialize access due to read/write dependencies
-- among transactions
```

PostgreSQL **détecte** que les deux transactions ont lu les mêmes données et effectué des écritures conflictuelles, et rejette Transaction B.

### Exemple détaillé : Transfert bancaire concurrent

**Règle métier** : Le solde total de deux comptes liés doit rester positif.

**État initial** :
```sql
compte_A : 100€  
compte_B : 100€  
Total : 200€  
```

**Transaction 1 (Transfert A→B)** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Vérifier que A a assez de fonds
SELECT solde FROM comptes WHERE id = 'A';  -- 100€

-- Transférer 150€ de A vers B
UPDATE comptes SET solde = solde - 150 WHERE id = 'A';  -- A = -50€  
UPDATE comptes SET solde = solde + 150 WHERE id = 'B';  -- B = 250€  

COMMIT;
```

**Transaction 2 (Transfert B→A, EN PARALLÈLE)** :
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Vérifier que B a assez de fonds
SELECT solde FROM comptes WHERE id = 'B';  -- 100€

-- Transférer 150€ de B vers A
UPDATE comptes SET solde = solde - 150 WHERE id = 'B';  -- B = -50€  
UPDATE comptes SET solde = solde + 150 WHERE id = 'A';  -- A = 250€  

COMMIT;
-- ERREUR: could not serialize access
```

PostgreSQL détecte que les deux transactions créeraient une incohérence et rejette l'une d'elles.

### Coût de performance

Le niveau Serializable a un coût :

1. **Overhead de surveillance** : PostgreSQL doit traquer toutes les lectures et écritures  
2. **Erreurs de sérialisation** : Nécessite une logique de retry dans l'application  
3. **Faux positifs** : Parfois, PostgreSQL rejette des transactions qui auraient été sûres

**Impact sur les performances** :

- ⚡ Lecture seule : Impact minimal  
- ⚡ Faible contention : Impact modéré  
- ⚠️ Haute contention : Impact significatif (beaucoup de retries)

### Quand utiliser Serializable ?

- ✅ **Logique métier complexe** avec des règles d'intégrité sophistiquées  
- ✅ **Transactions financières critiques**  
- ✅ **Systèmes où la cohérence absolue est requise**  
- ✅ **Protection contre le Write Skew**  
- ✅ **Applications où les erreurs de logique sont plus coûteuses que les retries**

- ❌ Applications haute performance avec beaucoup de contentions  
- ❌ Workloads OLTP simples  
- ❌ Applications qui ne peuvent pas gérer les retries

### Configuration

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- ou via SET
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- ou globalement (postgresql.conf)
default_transaction_isolation = 'serializable'
```

---

## Comparaison pratique des niveaux

### Scénario : Lecture répétée du même solde

```sql
-- Table initiale
CREATE TABLE comptes (id INT PRIMARY KEY, solde NUMERIC(10,2));  
INSERT INTO comptes VALUES (1, 1000.00);  
```

| Action | Read Committed | Repeatable Read | Serializable |
|--------|----------------|-----------------|--------------|
| Transaction A : SELECT solde | 1000€ | 1000€ (snapshot) | 1000€ (snapshot) |
| Transaction B : UPDATE solde = 500, COMMIT | (OK) | (OK) | (OK) |
| Transaction A : SELECT solde | **500€** (voit le changement) | **1000€** (snapshot figé) | **1000€** (snapshot figé) |
| Transaction A : UPDATE solde = solde + 100 | 600€ ✅ | ❌ Erreur sérialisation | ❌ Erreur sérialisation |

### Scénario : Comptage avec insertions concurrentes

| Action | Read Committed | Repeatable Read | Serializable |
|--------|----------------|-----------------|--------------|
| Transaction A : SELECT COUNT(*) | 10 | 10 (snapshot) | 10 (snapshot) |
| Transaction B : INSERT nouvelle ligne, COMMIT | (OK) | (OK) | (OK) |
| Transaction A : SELECT COUNT(*) | **11** (phantom) | **10** (figé) | **10** (figé) |

---

## Comment choisir le bon niveau d'isolation ?

### Arbre de décision

```
Votre application nécessite-t-elle une cohérence absolue avec règles complexes ?
    ├── OUI → SERIALIZABLE (+ logique de retry)
    └── NON ↓

Vos rapports/calculs nécessitent-ils un snapshot figé ?
    ├── OUI → REPEATABLE READ
    └── NON ↓

Application web/API standard avec transactions courtes ?
    ├── OUI → READ COMMITTED (défaut) ⭐
    └── NON → Analyser plus en détail
```

### Recommandations par type d'application

| Type d'application | Niveau recommandé | Raison |
|-------------------|-------------------|--------|
| API REST CRUD | **Read Committed** | Performance, simplicité |
| E-commerce (panier) | **Read Committed** | Transactions courtes |
| Transferts bancaires | **Serializable** | Cohérence critique |
| Rapports financiers | **Repeatable Read** | Snapshot figé |
| Analytics / BI | **Repeatable Read** | Données cohérentes |
| Inventaire multi-entrepôts | **Serializable** | Règles complexes |
| Blog / CMS | **Read Committed** | Cohérence relaxée OK |

---

## Configuration des niveaux d'isolation

### 1. Au niveau de la transaction

```sql
-- Syntaxe complète
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Syntaxe abrégée
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Avec START TRANSACTION
START TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### 2. Au niveau de la session

```sql
-- Pour toute la session en cours
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Vérifier le niveau actuel
SHOW default_transaction_isolation;
```

### 3. Au niveau global (postgresql.conf)

```ini
# Définir le niveau par défaut pour toutes les connexions
default_transaction_isolation = 'read committed'  # Valeurs : read committed, repeatable read, serializable
```

### 4. Au niveau de l'utilisateur ou de la base

```sql
-- Pour un utilisateur spécifique
ALTER USER mon_user SET default_transaction_isolation = 'repeatable read';

-- Pour une base de données
ALTER DATABASE ma_base SET default_transaction_isolation = 'serializable';
```

---

## Gestion des erreurs de sérialisation

### Types d'erreurs

En Repeatable Read et Serializable, vous pouvez rencontrer ces erreurs :

```
ERROR: could not serialize access due to concurrent update  
ERROR: could not serialize access due to read/write dependencies among transactions  
```

### Pattern de retry recommandé

```python
import time  
import psycopg2  
from psycopg2.extensions import ISOLATION_LEVEL_REPEATABLE_READ  

def execute_with_retry(conn, operation, max_retries=5):
    """
    Exécute une opération avec retry automatique en cas d'erreur de sérialisation
    """
    for attempt in range(max_retries):
        try:
            conn.set_isolation_level(ISOLATION_LEVEL_REPEATABLE_READ)

            with conn.cursor() as cur:
                operation(cur)

            conn.commit()
            return True  # Succès

        except psycopg2.extensions.TransactionRollbackError as e:
            # Erreur de sérialisation
            conn.rollback()

            if attempt == max_retries - 1:
                # Dernier essai échoué
                raise Exception(f"Échec après {max_retries} tentatives") from e

            # Attente avec backoff exponentiel
            wait_time = 0.1 * (2 ** attempt)
            time.sleep(wait_time)

        except Exception as e:
            # Autre erreur
            conn.rollback()
            raise

# Utilisation
def mon_transfert(cursor):
    cursor.execute("UPDATE comptes SET solde = solde - 100 WHERE id = 1")
    cursor.execute("UPDATE comptes SET solde = solde + 100 WHERE id = 2")

execute_with_retry(conn, mon_transfert)
```

### Bonnes pratiques de retry

1. ✅ **Limiter le nombre de tentatives** (3-5 typiquement)  
2. ✅ **Utiliser un backoff exponentiel** pour éviter la congestion  
3. ✅ **Logger les retries** pour surveiller la contention  
4. ✅ **Considérer un circuit breaker** si trop d'échecs  
5. ⚠️ **Attention aux effets de bord** : L'opération ne doit pas avoir d'effets non-transactionnels

---

## Bonnes pratiques

### 1. Commencez par Read Committed

Pour la plupart des applications, le niveau par défaut est suffisant et offre le meilleur équilibre performance/cohérence.

### 2. Utilisez Repeatable Read pour les rapports

```sql
-- Rapport financier mensuel
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT
    SUM(montant) as total_ventes,
    COUNT(*) as nb_commandes,
    AVG(montant) as panier_moyen
FROM commandes  
WHERE date >= '2024-01-01' AND date < '2024-02-01';  

-- Autres requêtes du rapport...

COMMIT;
```

### 3. Utilisez Serializable pour la logique critique

```sql
-- Virement bancaire avec vérification de solde
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Vérifier solde suffisant
SELECT solde FROM comptes WHERE id = source_id;

-- Effectuer le transfert
UPDATE comptes SET solde = solde - montant WHERE id = source_id;  
UPDATE comptes SET solde = solde + montant WHERE id = dest_id;  

COMMIT;
-- Si erreur de sérialisation → retry automatique par l'application
```

### 4. Gardez les transactions courtes

Plus une transaction est longue, plus le risque d'erreur de sérialisation augmente (en Repeatable Read et Serializable).

```sql
-- ❌ MAUVAIS : Transaction longue
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;  
SELECT * FROM huge_table;  -- Lecture de millions de lignes  
-- [Traitement long dans l'application]
UPDATE some_table SET ...;  
COMMIT;  -- Risque élevé d'erreur  

-- ✅ BON : Transactions courtes
-- Lecture en dehors de la transaction
SELECT * FROM huge_table;
-- [Traitement]

-- Transaction courte pour l'écriture
BEGIN;  
UPDATE some_table SET ...;  
COMMIT;  
```

### 5. Testez les erreurs de sérialisation

Assurez-vous que votre application gère correctement les erreurs de sérialisation, surtout en Repeatable Read et Serializable.

### 6. Surveillez les métriques

```sql
-- Nombre de transactions annulées par des erreurs de sérialisation
SELECT
    datname,
    xact_rollback as rollbacks,
    xact_commit as commits,
    ROUND(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2) as rollback_ratio
FROM pg_stat_database  
WHERE datname NOT IN ('template0', 'template1', 'postgres');  
```

Si le ratio de rollback est élevé, considérez :
- Réduire le niveau d'isolation
- Optimiser la logique pour réduire les contentions
- Raccourcir les transactions

---

## Cas d'usage réels

### Cas 1 : Système de réservation de places

**Problème** : Éviter la double réservation de la même place.

**Solution avec Serializable** :

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Vérifier disponibilité
SELECT disponible FROM places WHERE id = 42;

-- Réserver si disponible
UPDATE places SET disponible = false, client_id = 123 WHERE id = 42 AND disponible = true;

COMMIT;
-- En cas de conflit, retry automatique
```

### Cas 2 : Dashboard analytique

**Problème** : Afficher des statistiques cohérentes pour un rapport.

**Solution avec Repeatable Read** :

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Toutes ces requêtes voient les mêmes données
SELECT COUNT(*) FROM ventes WHERE date = CURRENT_DATE;  
SELECT SUM(montant) FROM ventes WHERE date = CURRENT_DATE;  
SELECT AVG(montant) FROM ventes WHERE date = CURRENT_DATE;  

COMMIT;
```

### Cas 3 : API REST simple

**Problème** : CRUD standard sans logique complexe.

**Solution avec Read Committed** (défaut) :

```sql
-- Pas besoin de spécifier explicitement
BEGIN;

UPDATE produits SET stock = stock - 1 WHERE id = 456;  
INSERT INTO commandes (produit_id, quantite) VALUES (456, 1);  

COMMIT;
```

---

## Conclusion

Les niveaux d'isolation sont un outil puissant pour contrôler le comportement des transactions concurrentes dans PostgreSQL. Voici les points clés à retenir :

### Résumé des niveaux

| Niveau | Quand l'utiliser | Avantages | Inconvénients |
|--------|------------------|-----------|---------------|
| **Read Committed** | Par défaut, applications standard | ⚡ Performance maximale, Simplicité | ⚠️ Non-Repeatable/Phantom Reads |
| **Repeatable Read** | Rapports, analytics, exports | ✅ Snapshot cohérent, Pas de surprises | ⚠️ Erreurs sérialisation possibles |
| **Serializable** | Logique critique, règles complexes | ✅ Cohérence absolue garantie | ⚠️ Performance impactée, Beaucoup de retries |

### Règles d'or

1. 🎯 **Commencez simple** : Read Committed couvre 80% des besoins  
2. 📊 **Rapports = Repeatable Read** : Pour une vue cohérente  
3. 💰 **Critique = Serializable** : Quand la cohérence est non négociable  
4. 🔄 **Implémentez les retries** : Indispensables pour Repeatable Read et Serializable  
5. ⏱️ **Transactions courtes** : Minimisez la durée pour réduire les conflits

### Pour aller plus loin

Dans la prochaine section (12.4), nous approfondirons les **anomalies transactionnelles** en détail : Dirty Read, Non-Repeatable Read, Phantom Read, et Write Skew.

---

**Points clés à retenir :**

- 🔑 4 niveaux d'isolation : Read Uncommitted, Read Committed, Repeatable Read, Serializable  
- 🔑 PostgreSQL par défaut = Read Committed (bon équilibre)  
- 🔑 Repeatable Read = snapshot figé pour toute la transaction  
- 🔑 Serializable = cohérence maximale mais erreurs de sérialisation possibles  
- 🔑 Plus strict = plus cohérent mais plus d'erreurs à gérer  
- 🔑 Toujours implémenter une logique de retry pour RR et Serializable  
- 🔑 Le choix dépend de votre logique métier, pas de la performance seule

⏭️ [Anomalies transactionnelles : Dirty Read, Non-Repeatable Read, Phantom Read](/12-concurrence-et-transactions/04-anomalies-transactionnelles.md)
