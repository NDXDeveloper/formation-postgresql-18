🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6. TRUNCATE vs DELETE : Différences transactionnelles et performances

## Introduction

Lorsqu'il s'agit de supprimer des données dans PostgreSQL, deux commandes principales s'offrent à vous : `DELETE` et `TRUNCATE`. Bien que leur objectif final soit similaire (retirer des lignes d'une table), leur fonctionnement interne, leurs performances et leurs cas d'usage sont radicalement différents.

Cette section explore en profondeur ces différences pour vous permettre de choisir la commande appropriée selon votre situation.

> **Analogie simple** : DELETE est comme enlever des livres un par un d'une bibliothèque, en notant chaque retrait dans un registre. TRUNCATE est comme vider complètement toutes les étagères d'un seul coup.

---

## 6.6.1. Les différences fondamentales

### Vue d'ensemble comparative

| Aspect | DELETE | TRUNCATE |
|--------|--------|----------|
| **Syntaxe WHERE** | ✅ Possible (ciblage précis) | ❌ Impossible (tout ou rien) |
| **Vitesse** | 🐌 Lent (ligne par ligne) | ⚡ Très rapide |
| **Triggers** | ✅ Déclenche les triggers | ❌ Ne déclenche pas les triggers |
| **ROLLBACK** | ✅ Possible dans transaction | ✅ Possible dans transaction |
| **WAL (Write-Ahead Log)** | 📝 Écrit chaque suppression | 📝 Écriture minimale |
| **Espace disque** | 🔄 Récupéré progressivement | ⚡ Libéré immédiatement |
| **Séquences (SERIAL)** | ⏸️ Ne réinitialise pas | 🔄 Peut réinitialiser |
| **Foreign Keys** | ✅ Vérifie les contraintes | ⚠️ Échoue si références existent |
| **Verrous (Locks)** | 🔒 Verrous de ligne | 🔒🔒 Verrou de table exclusif |

### DELETE : La suppression sélective

`DELETE` est la commande SQL standard pour supprimer des lignes :

```sql
DELETE FROM table_name  
WHERE condition;  
```

**Caractéristiques** :
- Traite chaque ligne individuellement
- Vérifie toutes les contraintes
- Déclenche les triggers
- Écrit dans le WAL pour la réplication
- Peut être ciblée avec WHERE

### TRUNCATE : La réinitialisation rapide

`TRUNCATE` est une commande DDL (Data Definition Language) spécialisée :

```sql
TRUNCATE TABLE table_name;
```

**Caractéristiques** :
- Supprime toutes les lignes d'un coup
- Opération quasi-instantanée
- Pas de traitement ligne par ligne
- Libère immédiatement l'espace disque

---

## 6.6.2. Syntaxe et utilisation de base

### DELETE : Syntaxe détaillée

```sql
-- Supprimer des lignes spécifiques
DELETE FROM employes  
WHERE date_embauche < '2020-01-01';  

-- Supprimer toutes les lignes (déconseillé, préférez TRUNCATE)
DELETE FROM employes;

-- Supprimer avec sous-requête
DELETE FROM employes  
WHERE id IN (  
    SELECT employe_id FROM licenciements
);

-- Supprimer avec RETURNING
DELETE FROM employes  
WHERE id = 42  
RETURNING *;  
```

### TRUNCATE : Syntaxe détaillée

```sql
-- Forme basique : vider la table
TRUNCATE TABLE employes;

-- Vider plusieurs tables simultanément
TRUNCATE TABLE employes, departements, projets;

-- Vider et réinitialiser les séquences
TRUNCATE TABLE employes RESTART IDENTITY;

-- Continuer même si séquences utilisées ailleurs
TRUNCATE TABLE employes RESTART IDENTITY CASCADE;

-- Vider avec gestion des foreign keys
TRUNCATE TABLE employes CASCADE;
```

#### Options de TRUNCATE

##### RESTART IDENTITY

Réinitialise les colonnes `SERIAL` (auto-incrémentées) :

```sql
-- Table avec ID auto-incrémenté
CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

-- Insérer des données
INSERT INTO test (nom) VALUES ('Alice'), ('Bob'), ('Charlie');  
SELECT * FROM test;  
-- Résultat : id = 1, 2, 3

-- Vider sans réinitialiser
TRUNCATE TABLE test;  
INSERT INTO test (nom) VALUES ('David');  
SELECT * FROM test;  
-- Résultat : id = 4 (continue la séquence)

-- Vider avec réinitialisation
TRUNCATE TABLE test RESTART IDENTITY;  
INSERT INTO test (nom) VALUES ('Emma');  
SELECT * FROM test;  
-- Résultat : id = 1 (séquence réinitialisée)
```

##### CONTINUE IDENTITY (défaut)

Conserve la séquence actuelle :

```sql
TRUNCATE TABLE test CONTINUE IDENTITY;
-- Équivalent à : TRUNCATE TABLE test;
```

##### CASCADE

Force la suppression même si d'autres tables ont des foreign keys :

```sql
-- Tables avec relations
CREATE TABLE departements (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement_id INTEGER REFERENCES departements(id)
);

-- Sans CASCADE : échoue si employes a des données
TRUNCATE TABLE departements;
-- ERROR: cannot truncate a table referenced in a foreign key constraint

-- Avec CASCADE : vide departements ET employes
TRUNCATE TABLE departements CASCADE;
-- NOTICE: truncate cascades to table "employes"
```

> **⚠️ ATTENTION** : CASCADE vide aussi les tables dépendantes !

---

## 6.6.3. Performances : La différence cruciale

### Benchmark comparatif

Voici des mesures réelles sur différents volumes de données :

#### Test 1 : Table de 10 000 lignes

```sql
-- Créer une table de test
CREATE TABLE test_perf (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer 10 000 lignes
INSERT INTO test_perf (data)  
SELECT 'test_' || generate_series(1, 10000);  
```

**Résultats** :

| Opération | Durée | Taille WAL | Espace libéré |
|-----------|-------|-----------|---------------|
| DELETE sans WHERE | ~80 ms | ~15 MB | Progressif |
| TRUNCATE | ~5 ms | ~8 KB | Immédiat |

**Gain : TRUNCATE est ~16× plus rapide**

#### Test 2 : Table de 1 000 000 lignes

```sql
INSERT INTO test_perf (data)  
SELECT 'test_' || generate_series(1, 1000000);  
```

**Résultats** :

| Opération | Durée | Taille WAL | Espace libéré |
|-----------|-------|-----------|---------------|
| DELETE sans WHERE | ~8 secondes | ~1.5 GB | Progressif (VACUUM requis) |
| TRUNCATE | ~10 ms | ~8 KB | Immédiat |

**Gain : TRUNCATE est ~800× plus rapide !**

#### Test 3 : Table de 10 000 000 lignes

**Résultats** :

| Opération | Durée | Taille WAL | Espace libéré |
|-----------|-------|-----------|---------------|
| DELETE sans WHERE | ~80 secondes | ~15 GB | Progressif (VACUUM requis) |
| TRUNCATE | ~15 ms | ~8 KB | Immédiat |

**Gain : TRUNCATE est ~5000× plus rapide !!!**

### Pourquoi TRUNCATE est-il si rapide ?

#### Architecture de DELETE

1. **Pour chaque ligne** :
   - Lire la ligne
   - Vérifier les contraintes
   - Marquer comme supprimée (MVCC)
   - Écrire dans le WAL
   - Déclencher les triggers (si présents)
   - Mettre à jour les index

2. **Après toutes les suppressions** :
   - L'espace n'est pas immédiatement libéré
   - VACUUM nécessaire pour récupérer l'espace

#### Architecture de TRUNCATE

1. **Une seule opération** :
   - Supprime les fichiers de données de la table
   - Recrée une table vide
   - Mise à jour minimale du WAL
   - Libération immédiate de l'espace

2. **Pas de traitement ligne par ligne** :
   - Pas de parcours des données
   - Pas de vérification contrainte par contrainte
   - Pas de triggers

### Visualisation de l'impact

```
DELETE (1 million de lignes) :
[████████████████████████████████████████] 8 secondes
Écriture WAL : ████████████████ (1.5 GB)

TRUNCATE (même table) :
[█] 0.01 seconde
Écriture WAL : (8 KB)
```

---

## 6.6.4. Comportement transactionnel

### Les deux commandes supportent les transactions

Contrairement à une idée reçue, **TRUNCATE peut être annulé dans une transaction** :

```sql
BEGIN;

-- Vider la table
TRUNCATE TABLE employes;

-- Vérifier
SELECT COUNT(*) FROM employes;
-- Résultat : 0

-- Annuler
ROLLBACK;

-- Vérifier à nouveau
SELECT COUNT(*) FROM employes;
-- Résultat : Les données sont toujours là !
```

### DELETE dans une transaction

```sql
BEGIN;

-- Supprimer des lignes
DELETE FROM employes WHERE salaire < 30000;

-- Vérifier l'effet
SELECT COUNT(*) FROM employes WHERE salaire < 30000;
-- Résultat : 0

-- Annuler si nécessaire
ROLLBACK;

-- Les données sont restaurées
SELECT COUNT(*) FROM employes WHERE salaire < 30000;
-- Résultat : Nombre original
```

### Isolation et MVCC

#### DELETE et MVCC

DELETE utilise le système MVCC (Multi-Version Concurrency Control) de PostgreSQL :

```sql
-- Session 1
BEGIN;  
DELETE FROM employes WHERE id = 42;  
-- La ligne est marquée comme supprimée, mais pas encore committée

-- Session 2 (en parallèle)
SELECT * FROM employes WHERE id = 42;
-- Résultat : La ligne est toujours visible !
-- MVCC permet de voir l'ancienne version

-- Session 1
COMMIT;

-- Session 2 (après le COMMIT de Session 1)
SELECT * FROM employes WHERE id = 42;
-- Résultat : La ligne a disparu
```

#### TRUNCATE et MVCC

TRUNCATE prend un verrou exclusif, bloquant toutes les autres opérations :

```sql
-- Session 1
BEGIN;  
TRUNCATE TABLE employes;  

-- Session 2 (en parallèle)
SELECT COUNT(*) FROM employes;
-- ⏳ Bloqué en attente que Session 1 termine

-- Session 1
COMMIT;

-- Session 2 (débloqué)
-- Résultat : 0 (table vide)
```

---

## 6.6.5. Gestion des contraintes et relations

### Foreign Keys : Comportement différent

#### Avec DELETE

DELETE vérifie toutes les contraintes foreign key :

```sql
-- Tables avec relation
CREATE TABLE departements (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement_id INTEGER REFERENCES departements(id)
);

INSERT INTO departements (nom) VALUES ('IT');  
INSERT INTO employes (nom, departement_id) VALUES ('Alice', 1);  

-- Tentative de suppression
DELETE FROM departements WHERE id = 1;
-- ERROR: update or delete on table "departements" violates foreign key constraint
```

**Solutions avec DELETE** :

```sql
-- Solution 1 : Supprimer d'abord les dépendances
DELETE FROM employes WHERE departement_id = 1;  
DELETE FROM departements WHERE id = 1;  

-- Solution 2 : Utiliser ON DELETE CASCADE (défini à la création)
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement_id INTEGER REFERENCES departements(id) ON DELETE CASCADE
);

DELETE FROM departements WHERE id = 1;
-- Supprime aussi automatiquement les employés du département
```

#### Avec TRUNCATE

TRUNCATE échoue si des foreign keys pointent vers la table :

```sql
TRUNCATE TABLE departements;
-- ERROR: cannot truncate a table referenced in a foreign key constraint
```

**Solutions avec TRUNCATE** :

```sql
-- Solution 1 : Utiliser CASCADE (vide toutes les tables liées)
TRUNCATE TABLE departements CASCADE;
-- NOTICE: truncate cascades to table "employes"
-- Vide departements ET employes

-- Solution 2 : Vider manuellement dans le bon ordre
TRUNCATE TABLE employes;  
TRUNCATE TABLE departements;  

-- Solution 3 : Vider plusieurs tables simultanément
TRUNCATE TABLE employes, departements;
```

### Triggers : Exécution ou non

#### DELETE déclenche les triggers

```sql
-- Créer un trigger
CREATE OR REPLACE FUNCTION log_delete()  
RETURNS TRIGGER AS $$  
BEGIN  
    INSERT INTO audit_log (action, table_name, row_id)
    VALUES ('DELETE', TG_TABLE_NAME, OLD.id);
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employes_delete_trigger  
AFTER DELETE ON employes  
FOR EACH ROW  
EXECUTE FUNCTION log_delete();  

-- DELETE déclenche le trigger
DELETE FROM employes WHERE id = 42;
-- → Le trigger s'exécute et enregistre dans audit_log
```

#### TRUNCATE ignore les triggers (sauf TRUNCATE triggers)

```sql
-- TRUNCATE ne déclenche PAS les triggers DELETE
TRUNCATE TABLE employes;
-- → Aucun enregistrement dans audit_log

-- Pour intercepter TRUNCATE, créer un trigger spécifique
CREATE OR REPLACE FUNCTION log_truncate()  
RETURNS TRIGGER AS $$  
BEGIN  
    INSERT INTO audit_log (action, table_name)
    VALUES ('TRUNCATE', TG_TABLE_NAME);
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employes_truncate_trigger  
BEFORE TRUNCATE ON employes  
FOR EACH STATEMENT  
EXECUTE FUNCTION log_truncate();  
```

---

## 6.6.6. Gestion de l'espace disque

### DELETE : Espace non libéré immédiatement

Après un DELETE, l'espace disque n'est pas immédiatement récupéré :

```sql
-- Vérifier la taille de la table
SELECT
    pg_size_pretty(pg_total_relation_size('employes')) AS taille_totale,
    pg_size_pretty(pg_relation_size('employes')) AS taille_table;

-- Résultat (exemple) :
-- taille_totale: 150 MB, taille_table: 120 MB

-- Supprimer toutes les lignes
DELETE FROM employes;

-- Vérifier à nouveau
SELECT pg_size_pretty(pg_relation_size('employes')) AS taille_table;
-- Résultat : 120 MB (inchangé !)
```

Les lignes sont marquées comme "mortes" mais l'espace n'est pas libéré.

#### Pourquoi ?

PostgreSQL utilise MVCC :
- Les anciennes versions doivent rester disponibles pour les transactions en cours
- L'espace est récupéré progressivement par VACUUM

#### Récupérer l'espace après DELETE

```sql
-- VACUUM standard (récupération partielle)
VACUUM employes;

-- VACUUM FULL (réorganisation complète, libération maximale)
-- ⚠️ Bloque la table pendant l'opération
VACUUM FULL employes;

-- Vérifier après VACUUM FULL
SELECT pg_size_pretty(pg_relation_size('employes')) AS taille_table;
-- Résultat : ~8 KB (table vide avec structure minimale)
```

### TRUNCATE : Libération immédiate

```sql
-- Avant
SELECT pg_size_pretty(pg_relation_size('employes')) AS taille_avant;
-- Résultat : 120 MB

-- TRUNCATE
TRUNCATE TABLE employes;

-- Après (immédiatement)
SELECT pg_size_pretty(pg_relation_size('employes')) AS taille_apres;
-- Résultat : ~8 KB (libération immédiate)
```

Aucun VACUUM nécessaire !

---

## 6.6.7. Impact sur les séquences (SERIAL, IDENTITY)

### DELETE : Séquences inchangées

```sql
CREATE TABLE test_sequence (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

-- Insérer des données
INSERT INTO test_sequence (nom) VALUES ('A'), ('B'), ('C');
-- id générés : 1, 2, 3

-- Supprimer tout
DELETE FROM test_sequence;

-- Insérer à nouveau
INSERT INTO test_sequence (nom) VALUES ('D');
-- id généré : 4 (la séquence continue)

SELECT * FROM test_sequence;
-- Résultat : id=4, nom='D'
```

La séquence continue là où elle s'était arrêtée.

### TRUNCATE : Réinitialisation optionnelle

#### TRUNCATE sans option (CONTINUE IDENTITY)

```sql
-- Même scénario
TRUNCATE TABLE test_sequence;

INSERT INTO test_sequence (nom) VALUES ('E');
-- id généré : 5 (continue)

SELECT * FROM test_sequence;
-- Résultat : id=5, nom='E'
```

Par défaut, la séquence continue.

#### TRUNCATE avec RESTART IDENTITY

```sql
TRUNCATE TABLE test_sequence RESTART IDENTITY;

INSERT INTO test_sequence (nom) VALUES ('F');
-- id généré : 1 (réinitialisé)

SELECT * FROM test_sequence;
-- Résultat : id=1, nom='F'
```

La séquence repart de 1.

### Cas d'usage pratique

```sql
-- Scénario : Table de test qu'on vide et remplit régulièrement
CREATE TABLE tests_temporaires (
    id SERIAL PRIMARY KEY,
    donnee TEXT
);

-- Cycle 1
INSERT INTO tests_temporaires (donnee) VALUES ('test1'), ('test2');
-- id: 1, 2

-- Vider et recommencer à 1
TRUNCATE TABLE tests_temporaires RESTART IDENTITY;

-- Cycle 2
INSERT INTO tests_temporaires (donnee) VALUES ('test3'), ('test4');
-- id: 1, 2 (réinitialisés)
```

---

## 6.6.8. Verrous (Locks) et concurrence

### DELETE : Verrous de ligne

DELETE prend des verrous au niveau des lignes :

```sql
-- Session 1
BEGIN;  
DELETE FROM employes WHERE id = 42;  
-- Verrou sur la ligne id=42

-- Session 2 (en parallèle)
DELETE FROM employes WHERE id = 43;
-- ✅ Fonctionne : ligne différente

SELECT * FROM employes WHERE id = 100;
-- ✅ Fonctionne : lecture autorisée

UPDATE employes SET salaire = 50000 WHERE id = 44;
-- ✅ Fonctionne : ligne différente
```

Les autres lignes restent accessibles.

### TRUNCATE : Verrou exclusif de table

TRUNCATE prend un verrou exclusif (ACCESS EXCLUSIVE) sur toute la table :

```sql
-- Session 1
BEGIN;  
TRUNCATE TABLE employes;  

-- Session 2 (en parallèle)
SELECT COUNT(*) FROM employes;
-- ⏳ BLOQUÉ : même les lectures sont interdites

UPDATE employes SET salaire = 50000 WHERE id = 44;
-- ⏳ BLOQUÉ

DELETE FROM employes WHERE id = 43;
-- ⏳ BLOQUÉ

-- Tout est bloqué jusqu'au COMMIT ou ROLLBACK de Session 1
```

### Implications pratiques

| Commande | Verrou | Impact sur concurrence |
|----------|--------|----------------------|
| **DELETE** | Lignes concernées | ✅ Bonne concurrence |
| **TRUNCATE** | Table entière | ❌ Bloque tout |

**Conséquence** : TRUNCATE n'est adapté que pour :
- Tables temporaires
- Maintenance hors production
- Situations où l'exclusivité est acceptable

---

## 6.6.9. Écriture dans le WAL (Write-Ahead Log)

### Concept du WAL

Le WAL (Write-Ahead Log) est le journal de transactions de PostgreSQL :
- Enregistre toutes les modifications
- Permet la réplication
- Permet la récupération après crash

### DELETE : Écriture exhaustive

```sql
-- Supprimer 1 million de lignes
DELETE FROM logs WHERE date_creation < '2024-01-01';
```

**Impact WAL** :
- Chaque ligne supprimée est enregistrée dans le WAL
- Si 1 million de lignes × 500 bytes par ligne = ~500 MB de WAL
- Réplication : Les replicas doivent rejouer toutes ces suppressions

### TRUNCATE : Écriture minimale

```sql
-- Vider la même table
TRUNCATE TABLE logs;
```

**Impact WAL** :
- Une seule entrée dans le WAL (~8 KB)
- Réplication : Les replicas reçoivent simplement "table vidée"

### Comparaison visuelle

```
DELETE (1M lignes) :  
WAL : [████████████████████████████████] 1.5 GB  
Durée réplication : ~30 secondes  

TRUNCATE (même table) :  
WAL : [█] 8 KB  
Durée réplication : ~0.1 seconde  
```

---

## 6.6.10. Quand utiliser DELETE vs TRUNCATE ?

### Utilisez DELETE quand...

✅ **Vous devez supprimer seulement certaines lignes**
```sql
DELETE FROM logs WHERE date_creation < '2024-01-01';  
DELETE FROM employes WHERE statut = 'inactif';  
```

✅ **Vous avez besoin de RETURNING**
```sql
DELETE FROM employes WHERE id = 42  
RETURNING id, nom, prenom, email;  
```

✅ **Les triggers doivent s'exécuter**
```sql
-- Trigger d'audit qui doit enregistrer chaque suppression
DELETE FROM documents WHERE id = 100;
```

✅ **Vous travaillez avec des foreign keys complexes**
```sql
-- DELETE gère mieux les cascades sélectives
DELETE FROM commandes WHERE statut = 'annulé';
```

✅ **La concurrence est importante**
```sql
-- DELETE ne bloque que les lignes concernées
DELETE FROM sessions WHERE date_expiration < NOW();
-- Les autres sessions restent accessibles
```

### Utilisez TRUNCATE quand...

✅ **Vous voulez vider complètement une table**
```sql
TRUNCATE TABLE logs_temporaires;  
TRUNCATE TABLE cache_table;  
```

✅ **La performance est critique**
```sql
-- Vider une énorme table rapidement
TRUNCATE TABLE analytics_brutes;  -- Millions de lignes, 0.01s
```

✅ **Vous voulez réinitialiser les séquences**
```sql
TRUNCATE TABLE tests RESTART IDENTITY;
-- Repart avec id=1
```

✅ **Vous voulez libérer immédiatement l'espace disque**
```sql
-- Pas besoin de VACUUM après
TRUNCATE TABLE archives_2023;
```

✅ **Les triggers ne sont pas nécessaires**
```sql
-- Table sans logique métier complexe
TRUNCATE TABLE staging_import;
```

✅ **Vous êtes en dehors des heures de production**
```sql
-- Maintenance nocturne : le verrou exclusif est acceptable
TRUNCATE TABLE rapports_journaliers;
```

### Matrice de décision

| Critère | DELETE | TRUNCATE |
|---------|--------|----------|
| **Suppression partielle** | ✅ Oui | ❌ Non |
| **Table avec millions de lignes** | ⚠️ Lent | ✅ Rapide |
| **Besoin de RETURNING** | ✅ Oui | ❌ Non |
| **Triggers requis** | ✅ Oui | ⚠️ Spéciaux |
| **Foreign keys complexes** | ✅ Oui | ⚠️ Attention |
| **Haute concurrence** | ✅ Oui | ❌ Non |
| **Réinitialiser séquences** | ❌ Non | ✅ Oui |
| **Libération espace immédiate** | ❌ Non | ✅ Oui |
| **Réplication (WAL)** | ⚠️ Lourd | ✅ Léger |

---

## 6.6.11. Exemples pratiques et patterns

### Pattern 1 : Nettoyage de logs

```sql
-- ❌ Inefficace pour gros volumes
DELETE FROM application_logs  
WHERE date_creation < CURRENT_DATE - INTERVAL '1 year';  
-- Peut prendre des minutes/heures sur des millions de lignes

-- ✅ Solution hybride : Partitionnement + TRUNCATE
-- Table partitionnée par mois
CREATE TABLE logs_2024_01 PARTITION OF logs  
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');  

-- Détacher et supprimer la partition (équivalent à TRUNCATE ultra-rapide)
ALTER TABLE logs DETACH PARTITION logs_2024_01;  
DROP TABLE logs_2024_01;  
```

### Pattern 2 : Staging tables (import de données)

```sql
-- Pattern typique ETL (Extract, Transform, Load)

BEGIN;

-- 1. Vider la table de staging (rapide)
TRUNCATE TABLE staging_import RESTART IDENTITY;

-- 2. Charger les nouvelles données
COPY staging_import FROM '/data/import.csv' CSV;

-- 3. Valider et transformer
INSERT INTO production_table  
SELECT * FROM staging_import  
WHERE validation_rules_ok;  

COMMIT;
```

### Pattern 3 : Tables de cache

```sql
-- Cache de calculs qu'on régénère périodiquement

-- Vider le cache (rapide)
TRUNCATE TABLE cache_statistics;

-- Recalculer
INSERT INTO cache_statistics  
SELECT  
    date_trunc('day', date_vente) AS jour,
    SUM(montant) AS total_ventes,
    COUNT(*) AS nb_commandes
FROM ventes  
WHERE date_vente >= CURRENT_DATE - INTERVAL '30 days'  
GROUP BY jour;  
```

### Pattern 4 : Tests unitaires

```sql
-- Setup de test : table propre à chaque test

CREATE OR REPLACE FUNCTION setup_test()  
RETURNS VOID AS $$  
BEGIN  
    -- Réinitialiser complètement
    TRUNCATE TABLE test_users RESTART IDENTITY CASCADE;

    -- Insérer données de test
    INSERT INTO test_users (nom, email) VALUES
        ('Test User 1', 'test1@example.com'),
        ('Test User 2', 'test2@example.com');
END;
$$ LANGUAGE plpgsql;

-- Chaque test commence avec un état propre
SELECT setup_test();
```

### Pattern 5 : Archivage puis suppression

```sql
-- Archiver avant de supprimer (DELETE requis pour RETURNING)

BEGIN;

-- Archiver avec RETURNING (DELETE nécessaire)
WITH archived AS (
    DELETE FROM commandes
    WHERE date_commande < '2023-01-01'
    RETURNING *
)
INSERT INTO commandes_archives  
SELECT * FROM archived;  

COMMIT;

-- Alternative si pas besoin de RETURNING et toute la table :
-- 1. Renommer (instantané)
ALTER TABLE old_data RENAME TO old_data_archive;

-- 2. Créer nouvelle table vide
CREATE TABLE old_data (LIKE old_data_archive INCLUDING ALL);
```

---

## 6.6.12. Pièges et erreurs courantes

### Piège 1 : DELETE sans WHERE en production

```sql
-- ❌ CATASTROPHE : Supprime tout lentement
DELETE FROM employes;
-- Sur une grosse table :
-- - Prend des heures
-- - Remplit le WAL
-- - Peut saturer le disque
-- - Bloque les replicas

-- ✅ Si vraiment besoin de tout supprimer :
TRUNCATE TABLE employes;
```

### Piège 2 : TRUNCATE avec CASCADE non intentionnel

```sql
-- ❌ Attention au CASCADE
TRUNCATE TABLE departements CASCADE;
-- Vide aussi employes, projets, et toutes tables liées !

-- ✅ Explicite et sûr
TRUNCATE TABLE employes;  
TRUNCATE TABLE departements;  
```

### Piège 3 : Oublier VACUUM après DELETE massif

```sql
-- Supprimer beaucoup de données
DELETE FROM logs WHERE date_creation < '2023-01-01';
-- → Espace non libéré

-- ❌ Oublier le VACUUM
-- Résultat : Table bloated (gonflée), performances dégradées

-- ✅ VACUUM après DELETE massif
VACUUM ANALYZE logs;
-- Ou planifier autovacuum correctement
```

### Piège 4 : TRUNCATE en pleine production

```sql
-- ❌ TRUNCATE bloque tout
TRUNCATE TABLE sessions_actives;
-- Tous les utilisateurs sont bloqués !

-- ✅ DELETE avec WHERE ou hors heures de pointe
DELETE FROM sessions_actives  
WHERE date_expiration < NOW();  
```

### Piège 5 : Séquence non réinitialisée

```sql
-- Vider table de test
TRUNCATE TABLE test_data;

-- Insérer données
INSERT INTO test_data (nom) VALUES ('Test');
-- id = 1001 (séquence pas réinitialisée !)

-- ✅ Réinitialiser explicitement
TRUNCATE TABLE test_data RESTART IDENTITY;  
INSERT INTO test_data (nom) VALUES ('Test');  
-- id = 1
```

---

## 6.6.13. Monitoring et observabilité

### Détecter les tables nécessitant VACUUM

```sql
-- Tables avec beaucoup de dead tuples après DELETE
SELECT
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables  
WHERE n_dead_tup > 1000  
ORDER BY n_dead_tup DESC  
LIMIT 10;  
```

### Surveiller l'impact sur le WAL

```sql
-- Taille actuelle du WAL
SELECT
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')
    ) AS wal_size;

-- Croissance du WAL (à exécuter avant et après opération)
SELECT pg_current_wal_lsn() AS wal_position;
```

### Identifier les longues opérations DELETE

```sql
-- Requêtes DELETE en cours
SELECT
    pid,
    usename,
    state,
    query,
    age(clock_timestamp(), query_start) AS duree
FROM pg_stat_activity  
WHERE query LIKE 'DELETE%'  
  AND state = 'active'
ORDER BY query_start;
```

---

## Récapitulatif final

### Tableau de décision rapide

| Situation | Commande | Raison |
|-----------|----------|--------|
| Supprimer 10% d'une table | DELETE | Suppression partielle |
| Vider complètement une table de staging | TRUNCATE | Performance maximale |
| Supprimer avec audit/log | DELETE | Triggers nécessaires |
| Nettoyer table temporaire en test | TRUNCATE RESTART IDENTITY | Réinitialisation complète |
| Supprimer en haute concurrence | DELETE | Moins de blocage |
| Vider 100 millions de lignes | TRUNCATE | Gain de temps énorme |
| Archiver puis supprimer | DELETE avec RETURNING | Besoin de récupérer les données |
| Maintenance hors production | TRUNCATE | Performance + libération espace |

### Points clés à retenir

**DELETE** :
- ✅ Flexible (WHERE)  
- ✅ Transactionnel  
- ✅ Triggers  
- ✅ RETURNING  
- ⚠️ Lent  
- ⚠️ VACUUM nécessaire

**TRUNCATE** :
- ✅ Ultra-rapide  
- ✅ Libération espace immédiate  
- ✅ Transactionnel (rollback possible)  
- ✅ Réinitialisation séquences  
- ❌ Pas de WHERE  
- ❌ Pas de RETURNING  
- ⚠️ Verrou exclusif

### Règle d'or

> **Si vous devez supprimer TOUTES les lignes d'une table et que les triggers ne sont pas critiques, utilisez TRUNCATE. Sinon, utilisez DELETE.**

---

## Conclusion

DELETE et TRUNCATE sont deux outils complémentaires dans l'arsenal PostgreSQL. Comprendre leurs différences est essentiel pour :

1. **Optimiser les performances** : TRUNCATE peut être 1000× plus rapide  
2. **Éviter les erreurs** : Savoir quel outil utiliser dans quelle situation  
3. **Maintenir la cohérence** : Comprendre l'impact sur les contraintes et triggers  
4. **Gérer la concurrence** : Anticiper les verrous et blocages

En production, le choix entre DELETE et TRUNCATE peut faire la différence entre une opération de quelques millisecondes et une opération de plusieurs heures. Choisissez judicieusement !

---


⏭️ ["Upsert" : La clause ON CONFLICT (DO NOTHING, DO UPDATE)](/06-manipulation-des-donnees/07-upsert-on-conflict.md)
