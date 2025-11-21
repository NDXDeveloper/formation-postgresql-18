🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Audit de Requêtes PostgreSQL
## Guide

---

## Table des Matières

1. [Introduction à l'Audit de Requêtes](#1-introduction-%C3%A0-laudit-de-requ%C3%AAtes)
2. [Pourquoi Auditer les Requêtes ?](#2-pourquoi-auditer-les-requ%C3%AAtes-)
3. [Le Cycle de Vie d'une Requête](#3-le-cycle-de-vie-dune-requ%C3%AAte)
4. [Outils d'Audit de Requêtes](#4-outils-daudit-de-requ%C3%AAtes)
5. [EXPLAIN : Comprendre les Plans d'Exécution](#5-explain--comprendre-les-plans-dex%C3%A9cution)
6. [Identification des Requêtes Lentes](#6-identification-des-requ%C3%AAtes-lentes)
7. [Analyse des Patterns de Performance](#7-analyse-des-patterns-de-performance)
8. [Problèmes Courants et Solutions](#8-probl%C3%A8mes-courants-et-solutions)
9. [Optimisation des Requêtes](#9-optimisation-des-requ%C3%AAtes)
10. [Anti-Patterns N+1](#10-anti-patterns-n1)
11. [Requêtes Complexes et Jointures](#11-requ%C3%AAtes-complexes-et-jointures)
12. [Agrégations et Performances](#12-agr%C3%A9gations-et-performances)
13. [PostgreSQL 18 : Nouveautés d'Optimisation](#13-postgresql-18--nouveaut%C3%A9s-doptimisation)
14. [Monitoring en Production](#14-monitoring-en-production)
15. [Checklist d'Audit Complète](#15-checklist-daudit-compl%C3%A8te)
16. [Cas Pratiques d'Optimisation](#16-cas-pratiques-doptimisation)
17. [Conclusion et Bonnes Pratiques](#17-conclusion-et-bonnes-pratiques)

---

## 1. Introduction à l'Audit de Requêtes

### 1.1. Qu'est-ce qu'un Audit de Requêtes ?

Un **audit de requêtes** est un processus systématique d'analyse et d'évaluation des requêtes SQL exécutées sur une base de données PostgreSQL pour :

- **Identifier** les requêtes lentes ou problématiques
- **Comprendre** comment PostgreSQL exécute chaque requête
- **Optimiser** les performances en modifiant les requêtes ou la structure de la base
- **Prévenir** les dégradations futures de performance

### 1.2. Différence avec l'Audit d'Indexation

| Aspect | Audit d'Indexation | Audit de Requêtes |
|--------|-------------------|-------------------|
| **Focus** | Structure (index) | Comportement (requêtes) |
| **Question** | "Ai-je les bons index ?" | "Mes requêtes sont-elles optimales ?" |
| **Action** | Créer/supprimer index | Réécrire requêtes |
| **Niveau** | Infrastructure | Application |

**Complémentarité** : Les deux audits sont complémentaires et doivent être menés ensemble.

### 1.3. À Qui s'Adresse cet Audit ?

- **Développeurs** : Qui écrivent les requêtes SQL
- **DevOps/SRE** : Qui maintiennent les systèmes en production
- **DBA** : Qui optimisent les performances de la base
- **Product Owners** : Qui veulent comprendre les causes des lenteurs

### 1.4. Quand Effectuer un Audit de Requêtes ?

**Déclencheurs d'audit** :
- ✅ Application lente ou temps de réponse dégradé
- ✅ Pics de charge CPU/Mémoire inexpliqués
- ✅ Plaintes utilisateurs sur la lenteur
- ✅ Avant une mise en production majeure
- ✅ Après ajout de nouvelles fonctionnalités
- ✅ Périodiquement (mensuel/trimestriel)

---

## 2. Pourquoi Auditer les Requêtes ?

### 2.1. Les Conséquences de Requêtes Non Optimisées

#### Impact sur les Performances

**Symptômes** :
- Temps de réponse de plusieurs secondes (voire minutes)
- Application qui "rame" ou freeze
- Timeouts fréquents
- Impossibilité de scaler

**Exemple concret** :
```sql
-- Requête non optimisée : 45 secondes
SELECT * FROM commandes WHERE client_id IN (
    SELECT id FROM clients WHERE ville = 'Paris'
);

-- Après optimisation : 0.2 secondes (225× plus rapide)
SELECT c.*
FROM commandes c
JOIN clients cl ON c.client_id = cl.id
WHERE cl.ville = 'Paris';
```

#### Impact sur les Ressources

**Requêtes gourmandes** causent :
- Saturation CPU (100% d'utilisation)
- Consommation mémoire excessive (risque d'OOM)
- I/O disque massif (latence)
- Blocage d'autres requêtes (contention)

**Coût cloud** :
Une requête mal optimisée peut coûter **10× plus cher** en ressources cloud.

#### Impact sur l'Expérience Utilisateur

- ❌ Utilisateurs frustrés
- ❌ Abandon de paniers d'achat
- ❌ Perte de clients
- ❌ Mauvaise réputation

**Règle d'or** :
- Requête simple : < 100 ms
- Requête complexe : < 1 seconde
- Rapport/export : < 10 secondes

### 2.2. Les Bénéfices d'un Audit Régulier

#### Amélioration des Performances

- ✅ Réduction des temps de réponse de **10× à 100×**
- ✅ Meilleure utilisation des ressources
- ✅ Scalabilité améliorée

#### Réduction des Coûts

- ✅ Moins de serveurs nécessaires
- ✅ Factures cloud réduites
- ✅ Moins d'incidents de production

#### Qualité du Code

- ✅ Code SQL plus maintenable
- ✅ Patterns de requêtes documentés
- ✅ Standards d'équipe établis

#### Connaissance Approfondie

- ✅ Compréhension du planificateur PostgreSQL
- ✅ Expertise en optimisation
- ✅ Capacité à prévoir les problèmes

---

## 3. Le Cycle de Vie d'une Requête

### 3.1. Les 5 Phases d'Exécution

Comprendre le cycle de vie d'une requête est essentiel pour l'optimiser.

#### Phase 1 : Parsing (Analyse Syntaxique)

**Rôle** : Vérifier que la requête SQL est syntaxiquement correcte.

```sql
-- Parsing OK
SELECT nom FROM employes WHERE id = 1;

-- Parsing ERROR
SELECT nom FORM employes WHERE id = 1;
--           ^^^^ Erreur de syntaxe
```

**Coût** : Négligeable (< 1 ms)

#### Phase 2 : Rewrite (Réécriture)

**Rôle** : Transformer la requête en utilisant les règles (views, triggers).

**Exemple** :
```sql
-- Vue définie
CREATE VIEW employes_actifs AS
SELECT * FROM employes WHERE actif = true;

-- Requête utilisateur
SELECT * FROM employes_actifs WHERE salaire > 50000;

-- Réécriture interne
SELECT * FROM employes WHERE actif = true AND salaire > 50000;
```

**Coût** : Faible (< 5 ms)

#### Phase 3 : Planning (Planification)

**Rôle** : **Le cœur de l'optimisation**. Le planificateur choisit le meilleur plan d'exécution.

**Décisions prises** :
- Utiliser un index ou faire un sequential scan ?
- Quel ordre pour les jointures ?
- Utiliser des hash joins ou nested loops ?
- Paralléliser l'exécution ?

**Coût** : Variable (1-100 ms selon la complexité)

**Importance** : C'est ici que se joue la performance. Un mauvais plan = requête lente.

#### Phase 4 : Execution (Exécution)

**Rôle** : Exécuter le plan choisi et récupérer les données.

**Opérations** :
- Lecture des pages de données
- Filtrage des lignes
- Jointures
- Tri
- Agrégations

**Coût** : Variable (peut être très élevé)

#### Phase 5 : Result (Retour des Résultats)

**Rôle** : Renvoyer les résultats au client.

**Coût** : Proportionnel au nombre de lignes retournées.

### 3.2. Schéma du Cycle de Vie

```
SQL Query
   ↓
[1. PARSING] ← Syntaxe correcte ?
   ↓
[2. REWRITE] ← Application des règles
   ↓
[3. PLANNING] ← Choix du meilleur plan (CRITIQUE)
   ↓
[4. EXECUTION] ← Exécution du plan
   ↓
[5. RESULT] ← Retour des résultats
   ↓
Client Application
```

### 3.3. Où se Concentrer pour l'Audit ?

**Priorités** :
1. **Planning** (Phase 3) : Le planificateur choisit-il le bon plan ?
2. **Execution** (Phase 4) : Les opérations sont-elles efficaces ?
3. **Result** (Phase 5) : Retourne-t-on trop de données ?

Les phases 1 et 2 sont rarement problématiques.

---

## 4. Outils d'Audit de Requêtes

### 4.1. pg_stat_statements (Essentiel)

**Description** : Extension qui enregistre toutes les requêtes exécutées avec leurs statistiques.

#### Installation

```sql
-- 1. Modifier postgresql.conf
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';

-- 2. Redémarrer PostgreSQL (nécessaire)

-- 3. Créer l'extension
CREATE EXTENSION pg_stat_statements;
```

#### Utilisation

**Requêtes les plus lentes (par temps moyen)** :
```sql
SELECT
    substring(query, 1, 100) AS query_short,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Requêtes les plus coûteuses (par temps total)** :
```sql
SELECT
    substring(query, 1, 100) AS query_short,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(100.0 * total_exec_time / SUM(total_exec_time) OVER (), 2) AS pct_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Requêtes les plus fréquentes** :
```sql
SELECT
    substring(query, 1, 100) AS query_short,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;
```

#### Configuration Recommandée

```sql
-- Nombre de requêtes à tracker (défaut: 5000)
ALTER SYSTEM SET pg_stat_statements.max = 10000;

-- Tracker les requêtes dans les fonctions PL/pgSQL
ALTER SYSTEM SET pg_stat_statements.track = 'all';

-- Sauvegarder les stats à l'arrêt
ALTER SYSTEM SET pg_stat_statements.save = on;
```

#### Réinitialisation

```sql
-- Réinitialiser toutes les statistiques
SELECT pg_stat_statements_reset();
```

**Attention** : Faites cela en connaissance de cause (perte de l'historique).

### 4.2. EXPLAIN et EXPLAIN ANALYZE

**Description** : Commandes pour afficher et analyser les plans d'exécution.

#### EXPLAIN (Estimation)

```sql
EXPLAIN SELECT * FROM employes WHERE nom = 'Dupont';
```

**Résultat** : Plan d'exécution **estimé** (sans exécuter réellement).

**Avantages** :
- ✅ Rapide (pas d'exécution)
- ✅ Sans risque (pas de modification)

**Inconvénients** :
- ❌ Estimations peuvent être inexactes
- ❌ Ne montre pas les temps réels

#### EXPLAIN ANALYZE (Exécution Réelle)

```sql
EXPLAIN ANALYZE SELECT * FROM employes WHERE nom = 'Dupont';
```

**Résultat** : Plan d'exécution avec **temps réels d'exécution**.

**Avantages** :
- ✅ Temps réels mesurés
- ✅ Détecte les écarts estimation vs réalité

**Inconvénients** :
- ❌ Exécute réellement la requête (attention aux DELETE/UPDATE)
- ❌ Plus lent

#### Options d'EXPLAIN

```sql
EXPLAIN (
    ANALYZE true,      -- Exécuter réellement
    BUFFERS true,      -- Montrer l'utilisation des buffers
    VERBOSE true,      -- Informations détaillées
    FORMAT JSON        -- Format JSON (ou TEXT, XML, YAML)
) SELECT ...;
```

**Recommandation** : Toujours utiliser `ANALYZE` et `BUFFERS` pour un audit complet.

### 4.3. PostgreSQL 18 : Améliorations d'EXPLAIN

**Nouveauté** : Affichage automatique des buffers et statistiques I/O enrichies.

```sql
-- PostgreSQL 18 : Plus de détails par défaut
EXPLAIN (ANALYZE) SELECT * FROM employes WHERE nom = 'Dupont';

-- Nouvelles métriques :
-- - I/O timing par opération
-- - WAL generation
-- - Statistiques par backend
```

### 4.4. pg_stat_activity

**Description** : Vue montrant les requêtes **en cours d'exécution**.

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    query,
    query_start,
    state_change
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY query_start;
```

**Cas d'usage** :
- Identifier les requêtes qui tournent depuis longtemps
- Détecter les requêtes bloquées
- Voir la charge en temps réel

#### Tuer une Requête Lente

```sql
-- Annuler une requête (soft)
SELECT pg_cancel_backend(pid);

-- Terminer le processus (hard)
SELECT pg_terminate_backend(pid);
```

**Attention** : `pg_terminate_backend` tue la connexion entière, pas seulement la requête.

### 4.5. auto_explain (Extension)

**Description** : Enregistre automatiquement les plans d'exécution des requêtes lentes.

#### Configuration

```sql
-- Charger l'extension
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements,auto_explain';

-- Redémarrer PostgreSQL

-- Configurer auto_explain
ALTER SYSTEM SET auto_explain.log_min_duration = 1000;  -- 1 seconde
ALTER SYSTEM SET auto_explain.log_analyze = on;
ALTER SYSTEM SET auto_explain.log_buffers = on;
ALTER SYSTEM SET auto_explain.log_timing = on;
ALTER SYSTEM SET auto_explain.log_nested_statements = on;

-- Recharger la configuration
SELECT pg_reload_conf();
```

**Résultat** : Les plans des requêtes > 1 sec sont automatiquement enregistrés dans les logs PostgreSQL.

**Avantage** : Capture passive des requêtes lentes en production.

### 4.6. pgBadger

**Description** : Analyseur de logs PostgreSQL générant des rapports HTML détaillés.

#### Installation

```bash
# Debian/Ubuntu
apt-get install pgbadger

# Ou depuis les sources
cpan App::pgBadger
```

#### Configuration des Logs PostgreSQL

```sql
-- Configuration pour pgBadger
ALTER SYSTEM SET log_min_duration_statement = 0;  -- Tout logger (ou 1000 pour > 1s)
ALTER SYSTEM SET log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h ';
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_temp_files = 0;
ALTER SYSTEM SET log_autovacuum_min_duration = 0;

SELECT pg_reload_conf();
```

#### Génération du Rapport

```bash
# Analyser les logs
pgbadger /var/log/postgresql/postgresql-*.log -o rapport.html

# Ouvrir le rapport
firefox rapport.html
```

**Rapport contient** :
- Top 20 requêtes les plus lentes
- Requêtes les plus fréquentes
- Requêtes les plus coûteuses
- Distribution temporelle
- Locks et deadlocks
- Checkpoints et autovacuum

### 4.7. Tableau Récapitulatif des Outils

| Outil | Utilité | Quand l'utiliser |
|-------|---------|------------------|
| **pg_stat_statements** | Statistiques agrégées | Identifier patterns généraux |
| **EXPLAIN ANALYZE** | Analyse détaillée | Optimiser une requête spécifique |
| **pg_stat_activity** | Requêtes en cours | Debugging temps réel |
| **auto_explain** | Capture automatique | Monitoring passif production |
| **pgBadger** | Rapport global | Audit périodique complet |

---

## 5. EXPLAIN : Comprendre les Plans d'Exécution

### 5.1. Anatomie d'un Plan d'Exécution

Un plan d'exécution est un **arbre d'opérations** que PostgreSQL va effectuer pour exécuter la requête.

#### Exemple Simple

```sql
EXPLAIN ANALYZE
SELECT * FROM employes WHERE nom = 'Dupont';
```

**Résultat** :
```
Index Scan using idx_employes_nom on employes
  (cost=0.42..8.44 rows=1 width=128)
  (actual time=0.023..0.024 rows=1 loops=1)
  Index Cond: (nom = 'Dupont'::text)
Planning Time: 0.123 ms
Execution Time: 0.045 ms
```

### 5.2. Décryptage des Éléments

#### Type de Scan

**Types courants** :
- `Seq Scan` : Balayage séquentiel (lit toute la table)
- `Index Scan` : Utilise un index
- `Index Only Scan` : Lit uniquement l'index (optimal)
- `Bitmap Heap Scan` : Combine plusieurs index
- `Nested Loop` : Jointure en boucles imbriquées
- `Hash Join` : Jointure avec table de hachage
- `Merge Join` : Jointure par fusion (données triées)

#### Cost (Coût Estimé)

```
cost=0.42..8.44
     ^^^^  ^^^^
     start end
```

**Interprétation** :
- `start` : Coût avant de retourner la première ligne
- `end` : Coût total pour toutes les lignes
- **Unité arbitraire** : Compare des coûts relatifs, pas des temps absolus

**Règle** : Plus le coût est bas, mieux c'est.

#### Rows (Lignes Estimées)

```
rows=1
```

**Interprétation** : Le planificateur estime retourner **1 ligne**.

**Attention** : Si l'estimation est très différente de la réalité (`actual rows`), les statistiques sont obsolètes → `ANALYZE` nécessaire.

#### Width (Largeur des Lignes)

```
width=128
```

**Interprétation** : Chaque ligne fait en moyenne **128 octets**.

#### Actual Time (Temps Réel)

```
actual time=0.023..0.024
            ^^^^^  ^^^^^
            start  end
```

**Interprétation** :
- Temps réel mesuré lors de l'exécution
- En millisecondes
- **C'est ça qui compte vraiment !**

#### Loops (Boucles)

```
loops=1
```

**Interprétation** : Cette opération a été exécutée **1 fois**.

**Attention** : Si `loops=1000`, multipliez le temps par 1000 !

**Exemple problématique** :
```
actual time=0.1..0.1 rows=1 loops=10000
```
Temps réel total = `0.1 ms × 10000 = 1000 ms = 1 seconde`

### 5.3. Types de Scans Détaillés

#### Seq Scan (Sequential Scan)

**Description** : Lit toute la table ligne par ligne.

```
Seq Scan on employes  (cost=0.00..1234.56 rows=10000 width=128)
  Filter: (nom = 'Dupont'::text)
```

**Quand est-ce utilisé ?**
- Pas d'index disponible
- Table petite (< 1000 lignes)
- Requête retourne > 10% de la table
- Le planificateur estime que c'est plus rapide

**Problème** : Sur grandes tables, très lent.

**Solution** : Créer un index sur la colonne filtrée.

#### Index Scan

**Description** : Utilise un index pour localiser les lignes.

```
Index Scan using idx_employes_nom on employes  (cost=0.42..8.44 rows=1 width=128)
  Index Cond: (nom = 'Dupont'::text)
```

**Avantages** :
- ✅ Rapide pour retrouver peu de lignes
- ✅ Utilise l'index efficacement

**Inconvénients** :
- ❌ Doit lire l'index **puis** la table
- ❌ Peut être lent si beaucoup de lignes à retourner

#### Index Only Scan

**Description** : Lit **uniquement** l'index, sans accéder à la table.

```
Index Only Scan using idx_employes_nom_include on employes
  (cost=0.42..4.44 rows=1 width=64)
  Index Cond: (nom = 'Dupont'::text)
  Heap Fetches: 0
```

**Conditions** :
- Index contient toutes les colonnes nécessaires (covering index)
- Visibility Map à jour (via VACUUM)

**Avantages** :
- ✅ Le plus rapide
- ✅ Minimal I/O

**Solution** : Utiliser `INCLUDE` dans les index.

```sql
CREATE INDEX idx_covering ON employes(nom) INCLUDE (prenom, salaire);
```

#### Bitmap Heap Scan

**Description** : Combine plusieurs index, crée un bitmap en mémoire, puis lit les pages.

```
Bitmap Heap Scan on employes  (cost=12.34..234.56 rows=50 width=128)
  Recheck Cond: ((ville = 'Paris'::text) OR (ville = 'Lyon'::text))
  ->  BitmapOr  (cost=12.34..12.34 rows=50 width=0)
        ->  Bitmap Index Scan on idx_ville  (cost=0.00..6.17 rows=25 width=0)
              Index Cond: (ville = 'Paris'::text)
        ->  Bitmap Index Scan on idx_ville  (cost=0.00..6.17 rows=25 width=0)
              Index Cond: (ville = 'Lyon'::text)
```

**Quand est-ce utilisé ?**
- Requêtes avec `OR`
- Requêtes avec plusieurs conditions sur index différents
- Nombre modéré de lignes à retourner

**Avantages** :
- ✅ Combine efficacement plusieurs index
- ✅ Réduit les accès disque aléatoires

### 5.4. Types de Jointures

#### Nested Loop (Boucles Imbriquées)

**Description** : Pour chaque ligne de la première table, parcourt la seconde.

```
Nested Loop  (cost=0.42..234.56 rows=100 width=256)
  ->  Seq Scan on commandes  (cost=0.00..12.34 rows=100 width=128)
  ->  Index Scan using clients_pkey on clients  (cost=0.42..2.22 rows=1 width=128)
        Index Cond: (id = commandes.client_id)
```

**Quand est-ce utilisé ?**
- Petite table externe
- Index sur la table interne
- Peu de lignes à joindre

**Complexité** : O(n × m) où n et m sont les tailles des tables.

**Avantages** :
- ✅ Efficace pour petits datasets
- ✅ Commence à retourner des lignes immédiatement

**Inconvénients** :
- ❌ Très lent si beaucoup de lignes

#### Hash Join

**Description** : Crée une table de hachage en mémoire pour la première table, puis la parcourt.

```
Hash Join  (cost=123.45..567.89 rows=1000 width=256)
  Hash Cond: (commandes.client_id = clients.id)
  ->  Seq Scan on commandes  (cost=0.00..234.56 rows=5000 width=128)
  ->  Hash  (cost=67.89..67.89 rows=1000 width=128)
        ->  Seq Scan on clients  (cost=0.00..67.89 rows=1000 width=128)
```

**Quand est-ce utilisé ?**
- Jointures sur de grandes tables
- Pas d'index disponible
- Suffisamment de `work_mem`

**Complexité** : O(n + m)

**Avantages** :
- ✅ Très efficace pour grandes tables
- ✅ Linéaire

**Inconvénients** :
- ❌ Nécessite assez de mémoire (`work_mem`)
- ❌ Ne retourne pas de lignes avant la fin de la construction du hash

#### Merge Join

**Description** : Trie les deux tables, puis les fusionne.

```
Merge Join  (cost=123.45..567.89 rows=1000 width=256)
  Merge Cond: (commandes.client_id = clients.id)
  ->  Sort  (cost=67.89..72.34 rows=5000 width=128)
        Sort Key: commandes.client_id
        ->  Seq Scan on commandes  (cost=0.00..234.56 rows=5000 width=128)
  ->  Sort  (cost=55.56..58.12 rows=1000 width=128)
        Sort Key: clients.id
        ->  Seq Scan on clients  (cost=0.00..67.89 rows=1000 width=128)
```

**Quand est-ce utilisé ?**
- Les deux tables sont déjà triées (ou ont des index)
- Jointure sur égalité

**Complexité** : O(n log n + m log m) pour les tris, puis O(n + m) pour la fusion

**Avantages** :
- ✅ Efficace si données déjà triées
- ✅ Pas de mémoire supplémentaire nécessaire

**Inconvénients** :
- ❌ Coût des tris si non triées

### 5.5. Opérations de Tri

#### Sort

```
Sort  (cost=123.45..126.78 rows=1000 width=128)
  Sort Key: salaire DESC
  Sort Method: quicksort  Memory: 71kB
  ->  Seq Scan on employes  (cost=0.00..67.89 rows=1000 width=128)
```

**Méthodes de tri** :
- `quicksort` : En mémoire (rapide)
- `top-N heapsort` : Tri partiel (LIMIT)
- `external merge` : Sur disque (lent, si dépassement de `work_mem`)

**Si "external merge"** :
```
Sort Method: external merge  Disk: 12345kB
```

**Problème** : Le tri utilise le disque (très lent).

**Solution** : Augmenter `work_mem`.

### 5.6. Buffers (Utilisation Mémoire)

Avec `EXPLAIN (ANALYZE, BUFFERS)` :

```
Buffers: shared hit=123 read=45 dirtied=10 written=5
```

**Signification** :
- `shared hit=123` : 123 pages lues depuis le **cache** (RAM) ✅
- `read=45` : 45 pages lues depuis le **disque** ❌
- `dirtied=10` : 10 pages modifiées
- `written=5` : 5 pages écrites sur disque

**Objectif** : Maximiser `hit`, minimiser `read`.

**Cache Hit Ratio** :
```
hit_ratio = hit / (hit + read)
```

**Objectif** : > 99%

### 5.7. PostgreSQL 18 : Améliorations EXPLAIN

**Nouveautés** :
- Statistiques I/O par backend
- Génération WAL par opération
- Temps passé dans chaque phase
- Auto-affichage des buffers (plus besoin de spécifier)

```sql
-- PostgreSQL 18
EXPLAIN (ANALYZE) SELECT ...;

-- Affiche automatiquement :
-- - Buffers
-- - I/O timing
-- - WAL generation
```

---

## 6. Identification des Requêtes Lentes

### 6.1. Définir "Lent"

**Seuils recommandés** :

| Type de requête | Temps acceptable | Temps lent |
|-----------------|------------------|------------|
| Requête simple (PK) | < 10 ms | > 50 ms |
| Requête avec jointure | < 100 ms | > 500 ms |
| Rapport/agrégation | < 1 sec | > 5 sec |
| Export/batch | < 10 sec | > 30 sec |

**Contexte** : Ces seuils dépendent de votre application.

### 6.2. Méthode 1 : pg_stat_statements

**Top 20 requêtes les plus lentes (moyenne)** :
```sql
SELECT
    queryid,
    substring(query, 1, 100) AS query_short,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- Plus de 100 ms
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Top 20 requêtes les plus coûteuses (temps cumulé)** :
```sql
SELECT
    queryid,
    substring(query, 1, 100) AS query_short,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(100.0 * total_exec_time / SUM(total_exec_time) OVER (), 2) AS pct_total_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Requêtes avec forte variabilité** :
```sql
SELECT
    queryid,
    substring(query, 1, 100) AS query_short,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    ROUND(stddev_exec_time / NULLIF(mean_exec_time, 0), 2) AS coefficient_variation
FROM pg_stat_statements
WHERE calls > 100
  AND stddev_exec_time > mean_exec_time  -- Variabilité élevée
ORDER BY coefficient_variation DESC
LIMIT 20;
```

**Interprétation** : Variabilité élevée peut indiquer :
- Plans d'exécution instables
- Données mal distribuées
- Contention

### 6.3. Méthode 2 : Logs PostgreSQL

#### Configuration

```sql
-- Logger toutes les requêtes > 1 seconde
ALTER SYSTEM SET log_min_duration_statement = 1000;

-- Inclure le temps d'exécution
ALTER SYSTEM SET log_duration = off;  -- Évite la duplication

SELECT pg_reload_conf();
```

#### Analyse des Logs

```bash
# Chercher les requêtes lentes
grep "duration:" /var/log/postgresql/postgresql-*.log | sort -t: -k2 -n

# Exemple de sortie :
# 2025-11-21 10:23:45 UTC [12345]: duration: 5234.567 ms  statement: SELECT ...
```

### 6.4. Méthode 3 : auto_explain

**Configuration** (déjà vue) :
```sql
ALTER SYSTEM SET auto_explain.log_min_duration = 1000;  -- 1 sec
ALTER SYSTEM SET auto_explain.log_analyze = on;
ALTER SYSTEM SET auto_explain.log_buffers = on;
```

**Résultat** : Plans d'exécution des requêtes lentes dans les logs.

### 6.5. Méthode 4 : pg_stat_activity (Temps Réel)

**Requêtes en cours depuis > 5 secondes** :
```sql
SELECT
    pid,
    usename,
    datname,
    query_start,
    NOW() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    substring(query, 1, 100) AS query_short
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '5 seconds'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY query_start;
```

**Si `wait_event_type` est présent** : La requête attend une ressource (I/O, lock, etc.).

### 6.6. Méthode 5 : pgBadger

```bash
pgbadger /var/log/postgresql/postgresql-*.log -o rapport.html
```

**Section "Slowest queries"** : Liste détaillée des requêtes les plus lentes.

---

## 7. Analyse des Patterns de Performance

### 7.1. Pattern 1 : Sequential Scans sur Grandes Tables

**Symptôme dans EXPLAIN** :
```
Seq Scan on large_table  (cost=0.00..123456.78 rows=1000000 width=128)
```

**Problème** : Lit **toute la table** au lieu d'utiliser un index.

**Causes** :
- Aucun index disponible
- Index non utilisé par le planificateur
- Statistiques obsolètes

**Solutions** :
1. Créer un index approprié
2. Mettre à jour les statistiques (`ANALYZE`)
3. Vérifier `random_page_cost`

**Exemple** :
```sql
-- Avant (Seq Scan)
SELECT * FROM commandes WHERE client_id = 123;

-- Créer index
CREATE INDEX idx_commandes_client ON commandes(client_id);

-- Après (Index Scan)
SELECT * FROM commandes WHERE client_id = 123;
```

### 7.2. Pattern 2 : Sous-Requêtes Corrélées

**Symptôme dans EXPLAIN** :
```
Seq Scan on commandes  (cost=0.00..12345.67 rows=1000 width=128)
  Filter: (total > (SubPlan 1))
  SubPlan 1
    ->  Aggregate  (cost=12.34..12.35 rows=1 width=8)
          ->  Seq Scan on commandes c2  (cost=0.00..12.34 rows=1 width=4)
                Filter: (c2.client_id = commandes.client_id)
```

**Problème** : La sous-requête est exécutée **pour chaque ligne** de la table externe.

**Exemple problématique** :
```sql
-- Sous-requête corrélée (lent)
SELECT c.id, c.total,
       (SELECT AVG(c2.total)
        FROM commandes c2
        WHERE c2.client_id = c.client_id) AS avg_client
FROM commandes c;
```

**Solution** : Réécrire avec JOIN ou CTE :
```sql
-- Réécriture avec CTE (rapide)
WITH avg_by_client AS (
    SELECT client_id, AVG(total) AS avg_total
    FROM commandes
    GROUP BY client_id
)
SELECT c.id, c.total, abc.avg_total AS avg_client
FROM commandes c
JOIN avg_by_client abc ON abc.client_id = c.client_id;
```

### 7.3. Pattern 3 : Sorts avec External Merge (Disque)

**Symptôme dans EXPLAIN** :
```
Sort  (cost=123.45..126.78 rows=100000 width=128)
  Sort Key: salaire DESC
  Sort Method: external merge  Disk: 12345kB
```

**Problème** : Le tri déborde sur **disque** (très lent).

**Cause** : `work_mem` trop faible.

**Solutions** :
1. Augmenter `work_mem` globalement
2. Augmenter `work_mem` pour cette session
3. Limiter les résultats (LIMIT)
4. Utiliser un index pour éviter le tri

**Exemple** :
```sql
-- Augmenter work_mem pour cette session
SET work_mem = '256MB';

-- Exécuter la requête
SELECT * FROM employes ORDER BY salaire DESC;

-- Ou créer un index pour éviter le tri
CREATE INDEX idx_employes_salaire ON employes(salaire DESC);
```

### 7.4. Pattern 4 : Nested Loops avec Beaucoup de Lignes

**Symptôme dans EXPLAIN** :
```
Nested Loop  (cost=0.42..123456.78 rows=100000 width=256)
  ->  Seq Scan on commandes  (cost=0.00..234.56 rows=10000 width=128)
  ->  Index Scan using clients_pkey on clients  (cost=0.42..12.34 rows=1 width=128)
```

**Problème** : Nested Loop avec 10,000 lignes externes = 10,000 accès à l'index.

**Solution** : Forcer un Hash Join ou Merge Join.

**Exemple** :
```sql
-- Désactiver nested loop pour cette requête
SET enable_nestloop = off;

SELECT ...;

-- Réactiver
SET enable_nestloop = on;
```

**Mieux** : Améliorer les statistiques ou ajouter un index pour que le planificateur choisisse le bon plan.

### 7.5. Pattern 5 : Trop de Données Retournées

**Symptôme** : `rows=1000000` dans EXPLAIN.

**Problème** : L'application récupère **toutes les lignes** alors qu'elle n'en a besoin que d'une partie.

**Solutions** :
1. Ajouter un `LIMIT`
2. Paginer les résultats
3. Filtrer davantage avec `WHERE`

**Exemple** :
```sql
-- Mauvais : Récupère 1 million de lignes
SELECT * FROM logs;

-- Bon : Pagination
SELECT * FROM logs
ORDER BY timestamp DESC
LIMIT 100 OFFSET 0;
```

### 7.6. Pattern 6 : Fonctions Non Indexées

**Symptôme dans EXPLAIN** :
```
Seq Scan on users  (cost=0.00..1234.56 rows=1000 width=128)
  Filter: (lower(email) = 'user@example.com'::text)
```

**Problème** : `lower(email)` empêche l'utilisation de l'index sur `email`.

**Solution** : Index sur expression.

```sql
-- Créer index sur expression
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- Maintenant : Index Scan
SELECT * FROM users WHERE lower(email) = 'user@example.com';
```

### 7.7. Pattern 7 : Statistiques Obsolètes

**Symptôme dans EXPLAIN** :
```
Hash Join  (cost=123.45..567.89 rows=1000 width=256)
  (actual time=1234.56..5678.90 rows=500000 loops=1)
```

**Problème** : Estimation `rows=1000` très différente de la réalité `actual rows=500000`.

**Cause** : Statistiques PostgreSQL obsolètes.

**Solution** : Exécuter `ANALYZE`.

```sql
-- Pour une table spécifique
ANALYZE commandes;

-- Pour toute la base
ANALYZE;

-- Vérifier les dernières stats
SELECT
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename = 'commandes';
```

---

## 8. Problèmes Courants et Solutions

### 8.1. Problème 1 : Index Non Utilisé

**Symptôme** : Index existant mais `Seq Scan` dans EXPLAIN.

**Causes possibles** :

#### Cause 1a : Statistiques Obsolètes
```sql
ANALYZE table_name;
```

#### Cause 1b : `random_page_cost` Trop Élevé
```sql
-- Pour SSD
SHOW random_page_cost;  -- Si 4.0, trop haut

ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

#### Cause 1c : Requête Retourne > 10% de la Table
Le planificateur préfère parfois `Seq Scan` si beaucoup de lignes sont retournées.

**Solution** : Aucune, c'est le comportement optimal.

#### Cause 1d : Type de Données Incompatible
```sql
-- Index existe sur id (integer)
CREATE INDEX idx_commandes_id ON commandes(id);

-- Mais requête utilise text
SELECT * FROM commandes WHERE id = '123';  -- text, pas integer
```

**Solution** : Cast explicite ou corriger le type dans la requête.
```sql
SELECT * FROM commandes WHERE id = 123;  -- integer
```

### 8.2. Problème 2 : Requête Lente Seulement Parfois

**Symptôme** : Requête rapide la plupart du temps, mais parfois très lente.

**Causes possibles** :

#### Cause 2a : Cache Froid vs Cache Chaud
- **Cache froid** : Première exécution, données sur disque (lent)
- **Cache chaud** : Données en mémoire (rapide)

**Solution** : Augmenter `shared_buffers` ou accepter cette variabilité.

#### Cause 2b : Contention (Locks)
D'autres requêtes verrouillent les données.

**Détection** :
```sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
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

**Solution** : Optimiser les transactions concurrentes ou réduire les locks.

#### Cause 2c : Plans d'Exécution Instables
Paramètres de requête différents entraînent des plans différents.

**Détection** : Variabilité élevée dans `pg_stat_statements` (`stddev_exec_time >> mean_exec_time`).

**Solution** :
- Utiliser `PREPARE` pour fixer le plan
- Ajuster les statistiques
- Réécrire la requête pour être plus stable

### 8.3. Problème 3 : OUT OF MEMORY

**Symptôme** : Erreur `ERROR: out of memory` lors de l'exécution d'une requête.

**Causes possibles** :

#### Cause 3a : `work_mem` Trop Élevé × Trop de Connexions
```
Mémoire totale = max_connections × work_mem × complexité_requête
```

**Solution** : Réduire `work_mem` ou `max_connections`, ou utiliser PgBouncer.

#### Cause 3b : Hash Join Trop Gourmand
```sql
-- Requête crée une énorme table de hash
SELECT ... FROM huge_table1 JOIN huge_table2 ...;
```

**Solution** :
- Augmenter `work_mem` temporairement pour cette requête
- Filtrer davantage avant la jointure
- Ajouter des index

#### Cause 3c : Tri Massif en Mémoire
```sql
SELECT * FROM huge_table ORDER BY random();  -- Très coûteux
```

**Solution** : Éviter les tris aléatoires, limiter les résultats.

### 8.4. Problème 4 : Deadlock (Interblocage)

**Symptôme** : Erreur `ERROR: deadlock detected`.

**Exemple** :
```
Transaction 1:
BEGIN;
UPDATE commandes SET total = 100 WHERE id = 1;  -- Lock commandes(1)
UPDATE clients SET credit = 50 WHERE id = 10;   -- Attend lock clients(10)

Transaction 2:
BEGIN;
UPDATE clients SET credit = 50 WHERE id = 10;   -- Lock clients(10)
UPDATE commandes SET total = 100 WHERE id = 1;  -- Attend lock commandes(1)

→ DEADLOCK
```

**Solutions** :
1. **Ordonner les verrous** : Toujours acquérir les locks dans le même ordre
2. **Transactions courtes** : Réduire la durée des transactions
3. **Isolation niveau** : Ajuster le niveau d'isolation si approprié

### 8.5. Problème 5 : Planificateur Choisit le Mauvais Plan

**Symptôme** : EXPLAIN montre un plan sous-optimal évident.

**Causes** :

#### Cause 5a : Coûts Mal Configurés
```sql
-- Vérifier random_page_cost, seq_page_cost
SHOW random_page_cost;
SHOW seq_page_cost;
```

#### Cause 5b : Statistiques Incorrectes
```sql
ANALYZE table_name;
```

#### Cause 5c : Paramètres de Planification
```sql
-- Augmenter les ressources allouées au planificateur
SET random_page_cost = 1.1;  -- SSD
SET cpu_tuple_cost = 0.01;
SET cpu_index_tuple_cost = 0.005;
SET cpu_operator_cost = 0.0025;
```

**Solution de dernier recours** : Hints (non natif dans PostgreSQL, mais extensions existent comme `pg_hint_plan`).

---

## 9. Optimisation des Requêtes

### 9.1. Principe Général d'Optimisation

**Ordre de priorité** :
1. **Réduire les données scannées** : Filtrer tôt avec WHERE
2. **Utiliser les index** : Créer index appropriés
3. **Éviter les opérations coûteuses** : Sous-requêtes corrélées, fonctions non indexées
4. **Simplifier les jointures** : Moins de tables jointes = plus rapide
5. **Limiter les résultats** : LIMIT, pagination

### 9.2. Technique 1 : Réécrire les Sous-Requêtes

#### Avant : Sous-Requête Corrélée

```sql
-- Lent : Sous-requête exécutée pour chaque ligne
SELECT
    c.id,
    c.nom,
    (SELECT COUNT(*)
     FROM commandes cmd
     WHERE cmd.client_id = c.id) AS nb_commandes
FROM clients c;
```

**Plan d'exécution** : `SubPlan` exécuté N fois.

#### Après : JOIN ou CTE

```sql
-- Rapide : Agrégation puis jointure
SELECT
    c.id,
    c.nom,
    COALESCE(cmd_counts.nb_commandes, 0) AS nb_commandes
FROM clients c
LEFT JOIN (
    SELECT client_id, COUNT(*) AS nb_commandes
    FROM commandes
    GROUP BY client_id
) cmd_counts ON cmd_counts.client_id = c.id;
```

**Gain typique** : 10× à 100× plus rapide.

### 9.3. Technique 2 : EXISTS vs IN vs JOIN

#### Problème avec IN + Sous-Requête

```sql
-- Potentiellement lent
SELECT * FROM clients
WHERE id IN (SELECT client_id FROM commandes WHERE total > 1000);
```

**Problème** : Si la sous-requête retourne beaucoup de lignes, `IN` peut être lent.

#### Solution 1 : EXISTS (Souvent Plus Rapide)

```sql
-- Souvent plus rapide
SELECT * FROM clients c
WHERE EXISTS (
    SELECT 1 FROM commandes cmd
    WHERE cmd.client_id = c.id AND cmd.total > 1000
);
```

**Avantage** : S'arrête dès qu'une ligne est trouvée (pas besoin de toutes les lire).

#### Solution 2 : JOIN (Alternative)

```sql
-- Équivalent avec JOIN
SELECT DISTINCT c.*
FROM clients c
JOIN commandes cmd ON cmd.client_id = c.id
WHERE cmd.total > 1000;
```

**Note** : Le `DISTINCT` est nécessaire pour éviter les doublons.

#### Comparaison

| Technique | Quand l'utiliser |
|-----------|------------------|
| **IN** | Petite liste de valeurs littérales |
| **EXISTS** | Vérifier l'existence (pas besoin des valeurs) |
| **JOIN** | Besoin des colonnes de la sous-requête |

### 9.4. Technique 3 : CTE vs Sous-Requêtes

#### CTE (Common Table Expression)

```sql
-- CTE
WITH commandes_recentes AS (
    SELECT client_id, COUNT(*) AS nb
    FROM commandes
    WHERE date_commande > CURRENT_DATE - INTERVAL '30 days'
    GROUP BY client_id
)
SELECT c.nom, cr.nb
FROM clients c
JOIN commandes_recentes cr ON cr.client_id = c.id;
```

**Avantages** :
- ✅ Plus lisible
- ✅ Peut être référencé plusieurs fois
- ✅ Facilite la maintenance

#### CTE MATERIALIZED (PostgreSQL 12+)

```sql
-- Forcer la matérialisation
WITH commandes_recentes AS MATERIALIZED (
    SELECT client_id, COUNT(*) AS nb
    FROM commandes
    WHERE date_commande > CURRENT_DATE - INTERVAL '30 days'
    GROUP BY client_id
)
SELECT ...;
```

**Quand utiliser MATERIALIZED** :
- CTE utilisé plusieurs fois
- CTE coûteux à calculer
- Éviter la réévaluation

**Quand NE PAS utiliser MATERIALIZED** :
- CTE utilisé une seule fois
- CTE simple
- Le planificateur peut optimiser sans matérialisation

### 9.5. Technique 4 : Éviter SELECT *

#### Problème

```sql
-- Récupère TOUTES les colonnes (même si inutiles)
SELECT * FROM employes WHERE id = 123;
```

**Problèmes** :
- Transfère des données inutiles
- Empêche les Index Only Scans
- Utilise plus de bande passante

#### Solution

```sql
-- Récupérer uniquement les colonnes nécessaires
SELECT id, nom, prenom FROM employes WHERE id = 123;
```

**Avantages** :
- ✅ Moins de données transférées
- ✅ Permet Index Only Scan si covering index
- ✅ Plus rapide

### 9.6. Technique 5 : Utiliser LIMIT et Pagination

#### Problème : Récupérer Toutes les Lignes

```sql
-- Récupère 1 million de lignes
SELECT * FROM logs ORDER BY timestamp DESC;
```

**Problème** : Surcharge mémoire et réseau.

#### Solution : Pagination

```sql
-- Page 1
SELECT * FROM logs
ORDER BY timestamp DESC
LIMIT 100 OFFSET 0;

-- Page 2
SELECT * FROM logs
ORDER BY timestamp DESC
LIMIT 100 OFFSET 100;
```

**Limite de OFFSET** : Sur grandes offsets (OFFSET 1000000), devient lent.

#### Solution Avancée : Keyset Pagination (Cursor)

```sql
-- Page 1
SELECT * FROM logs
ORDER BY timestamp DESC
LIMIT 100;

-- Récupérer le dernier timestamp (ex: 2025-11-21 10:00:00)

-- Page 2
SELECT * FROM logs
WHERE timestamp < '2025-11-21 10:00:00'
ORDER BY timestamp DESC
LIMIT 100;
```

**Avantages** :
- ✅ Performances constantes même sur grandes pages
- ✅ Utilise l'index efficacement

### 9.7. Technique 6 : Batching (Lots)

#### Problème : Trop de Requêtes Individuelles

```sql
-- 1000 requêtes individuelles
for id in ids:
    SELECT * FROM produits WHERE id = id;
```

**Problème** : Overhead réseau et parsing × 1000.

#### Solution : Requête en Lot

```sql
-- 1 seule requête
SELECT * FROM produits WHERE id = ANY(ARRAY[1, 2, 3, ..., 1000]);
```

**Gain** : 10× à 100× plus rapide.

### 9.8. Technique 7 : Préparation de Requêtes (PREPARE)

#### Problème : Parsing Répété

Chaque exécution de requête :
1. Parse la requête
2. Planifie
3. Exécute

**Pour des requêtes répétées**, le parsing est du gaspillage.

#### Solution : PREPARE

```sql
-- Préparer la requête
PREPARE get_employe (int) AS
    SELECT * FROM employes WHERE id = $1;

-- Exécuter (pas de parsing)
EXECUTE get_employe(123);
EXECUTE get_employe(456);
EXECUTE get_employe(789);

-- Libérer
DEALLOCATE get_employe;
```

**Avantages** :
- ✅ Pas de parsing répété
- ✅ Plan d'exécution réutilisé (parfois)
- ✅ Plus rapide

**Utilisation typique** : Dans les drivers (psycopg3, pg pour Node.js, etc.).

---

## 10. Anti-Patterns N+1

### 10.1. Qu'est-ce que le N+1 ?

Le **N+1 problem** est un anti-pattern où une requête initiale récupère N enregistrements, puis N requêtes supplémentaires sont exécutées pour récupérer des données liées.

**Résultat** : N+1 requêtes au lieu d'une seule requête optimisée.

### 10.2. Exemple du Problème

#### Code Application (Python avec ORM)

```python
# Requête 1 : Récupérer tous les clients
clients = Client.query.all()  # 1 requête

# Requêtes 2 à N+1 : Pour chaque client, récupérer ses commandes
for client in clients:
    commandes = client.commandes  # 1 requête par client
    print(f"{client.nom} a {len(commandes)} commandes")
```

**SQL généré** :
```sql
-- Requête 1
SELECT * FROM clients;  -- Retourne 1000 clients

-- Requêtes 2 à 1001
SELECT * FROM commandes WHERE client_id = 1;
SELECT * FROM commandes WHERE client_id = 2;
SELECT * FROM commandes WHERE client_id = 3;
...
SELECT * FROM commandes WHERE client_id = 1000;
```

**Total** : **1001 requêtes** ! 😱

### 10.3. Détection

#### Dans pg_stat_statements

Beaucoup d'appels de la même requête avec paramètres différents :
```sql
SELECT
    query,
    calls
FROM pg_stat_statements
WHERE calls > 1000
ORDER BY calls DESC;
```

Si vous voyez :
```
query: SELECT * FROM commandes WHERE client_id = $1
calls: 10000
```

**Suspect de N+1**.

#### Dans les Logs

```
SELECT * FROM commandes WHERE client_id = 1;
SELECT * FROM commandes WHERE client_id = 2;
SELECT * FROM commandes WHERE client_id = 3;
...
```

Répétition évidente.

### 10.4. Solutions

#### Solution 1 : Eager Loading (ORM)

**Python (SQLAlchemy)** :
```python
# Avec joinedload : 1 seule requête avec JOIN
clients = Client.query.options(joinedload(Client.commandes)).all()

for client in clients:
    commandes = client.commandes  # Pas de requête supplémentaire
    print(f"{client.nom} a {len(commandes)} commandes")
```

**SQL généré** :
```sql
-- 1 seule requête avec JOIN
SELECT clients.*, commandes.*
FROM clients
LEFT JOIN commandes ON commandes.client_id = clients.id;
```

**Résultat** : **1 requête** au lieu de 1001.

#### Solution 2 : Requête Manuelle Optimisée

```python
# 1. Récupérer les clients
clients = Client.query.all()
client_ids = [c.id for c in clients]

# 2. Récupérer toutes les commandes en une seule fois
commandes = Commande.query.filter(Commande.client_id.in_(client_ids)).all()

# 3. Grouper en mémoire
commandes_by_client = {}
for cmd in commandes:
    commandes_by_client.setdefault(cmd.client_id, []).append(cmd)

# 4. Utiliser
for client in clients:
    client_commandes = commandes_by_client.get(client.id, [])
    print(f"{client.nom} a {len(client_commandes)} commandes")
```

**SQL généré** :
```sql
-- Requête 1
SELECT * FROM clients;

-- Requête 2 (1 seule fois)
SELECT * FROM commandes WHERE client_id IN (1, 2, 3, ..., 1000);
```

**Résultat** : **2 requêtes** au lieu de 1001.

#### Solution 3 : Utiliser LATERAL JOIN (PostgreSQL)

```sql
-- Pour chaque client, récupérer ses 5 dernières commandes
SELECT
    c.id,
    c.nom,
    cmd.*
FROM clients c
LEFT JOIN LATERAL (
    SELECT *
    FROM commandes
    WHERE client_id = c.id
    ORDER BY date_commande DESC
    LIMIT 5
) cmd ON true;
```

**Avantage** : Permet des sous-requêtes corrélées **efficaces** dans le FROM.

### 10.5. Impact Performance

**Exemple avec 1000 clients** :

| Approche | Nb Requêtes | Temps |
|----------|-------------|-------|
| N+1 (Lazy Loading) | 1001 | 10 secondes |
| Eager Loading (JOIN) | 1 | 0.1 secondes |
| 2 Requêtes (IN) | 2 | 0.15 secondes |

**Gain** : **100× plus rapide** avec Eager Loading.

---

## 11. Requêtes Complexes et Jointures

### 11.1. Optimiser l'Ordre des Jointures

Le planificateur PostgreSQL essaie de choisir le meilleur ordre, mais comprendre les principes aide.

**Principe** : Joindre d'abord les tables qui **réduisent le plus** le nombre de lignes.

#### Exemple

```sql
-- 3 tables
-- clients : 1,000,000 lignes
-- commandes : 10,000,000 lignes
-- produits : 100,000 lignes

-- Requête
SELECT *
FROM clients c
JOIN commandes cmd ON cmd.client_id = c.id
JOIN produits p ON p.id = cmd.produit_id
WHERE c.pays = 'France'       -- Réduit à 100,000 clients
  AND cmd.statut = 'validé'   -- Réduit à 1,000,000 commandes
  AND p.categorie = 'A';      -- Réduit à 10,000 produits
```

**Ordre optimal** :
1. Filtrer `produits` (100,000 → 10,000)
2. Filtrer `commandes` (10,000,000 → 1,000,000)
3. Joindre avec `clients` filtrés (1,000,000 → 100,000)
4. Joindre les résultats

**Planificateur PostgreSQL** : Calcule automatiquement l'ordre optimal (généralement).

### 11.2. Éviter les Jointures Cartésiennes Accidentelles

#### Problème : Jointure Manquante

```sql
-- ERREUR : Oubli de condition de jointure
SELECT *
FROM clients, commandes
WHERE clients.pays = 'France';
```

**Résultat** : Produit cartésien = 1,000,000 clients × 10,000,000 commandes = 10,000,000,000,000 lignes ! 💥

**Solution** : Toujours inclure les conditions de jointure.

```sql
-- CORRECT
SELECT *
FROM clients c
JOIN commandes cmd ON cmd.client_id = c.id
WHERE c.pays = 'France';
```

### 11.3. Utiliser les Bons Types de Jointures

#### INNER JOIN vs LEFT JOIN

```sql
-- INNER JOIN : Uniquement les clients avec commandes
SELECT c.nom, cmd.total
FROM clients c
INNER JOIN commandes cmd ON cmd.client_id = c.id;

-- LEFT JOIN : Tous les clients, même sans commandes
SELECT c.nom, COALESCE(cmd.total, 0) AS total
FROM clients c
LEFT JOIN commandes cmd ON cmd.client_id = c.id;
```

**Performance** : `INNER JOIN` est souvent plus rapide (moins de lignes).

**Conseil** : Utilisez `INNER JOIN` sauf si vous avez vraiment besoin des lignes non correspondantes.

### 11.4. Jointures sur Index

**Règle** : Les colonnes de jointure **doivent être indexées**.

```sql
-- Jointure sur client_id
SELECT *
FROM commandes cmd
JOIN clients c ON c.id = cmd.client_id;

-- Index nécessaires :
-- clients.id → PRIMARY KEY (auto-indexé)
-- commandes.client_id → DOIT être indexé

CREATE INDEX idx_commandes_client ON commandes(client_id);
```

### 11.5. Filtrer Avant de Joindre

**Principe** : Réduire le nombre de lignes **avant** la jointure.

#### Sous-Optimal

```sql
-- Jointure puis filtrage
SELECT c.nom, cmd.total
FROM clients c
JOIN commandes cmd ON cmd.client_id = c.id
WHERE c.pays = 'France'
  AND cmd.date_commande > '2025-01-01';
```

**Problème** : Joint **toutes** les lignes, puis filtre.

#### Optimal

```sql
-- Filtrage puis jointure
SELECT c.nom, cmd.total
FROM (
    SELECT * FROM clients WHERE pays = 'France'
) c
JOIN (
    SELECT * FROM commandes WHERE date_commande > '2025-01-01'
) cmd ON cmd.client_id = c.id;
```

**Note** : Le planificateur PostgreSQL fait souvent cette optimisation automatiquement (predicate pushdown).

### 11.6. Lateral Joins pour Sous-Requêtes Corrélées

**Cas d'usage** : Récupérer les N meilleurs/derniers éléments pour chaque groupe.

```sql
-- Top 3 commandes par client
SELECT
    c.id,
    c.nom,
    cmd.date_commande,
    cmd.total
FROM clients c
LEFT JOIN LATERAL (
    SELECT *
    FROM commandes
    WHERE client_id = c.id
    ORDER BY total DESC
    LIMIT 3
) cmd ON true;
```

**Avantage** : Beaucoup plus efficace qu'une sous-requête corrélée classique.

---

## 12. Agrégations et Performances

### 12.1. Optimiser GROUP BY

#### Problème : GROUP BY sans Index

```sql
-- Agrégation sur colonne non indexée
SELECT client_id, SUM(total)
FROM commandes
GROUP BY client_id;
```

**EXPLAIN peut montrer** :
```
HashAggregate  (cost=123456.78..123567.89 rows=10000 width=16)
  ->  Seq Scan on commandes  (cost=0.00..100000.00 rows=10000000 width=12)
```

**Problème** : Seq Scan + HashAggregate peut être lent.

#### Solution : Index sur Colonne de GROUP BY

```sql
CREATE INDEX idx_commandes_client ON commandes(client_id);

-- Maintenant potentiellement :
-- Index Scan ou GroupAggregate (plus efficace)
```

### 12.2. Pré-Agrégation avec Vues Matérialisées

Pour des agrégations **très coûteuses** et **peu changeantes** :

```sql
-- Vue matérialisée
CREATE MATERIALIZED VIEW stats_clients AS
SELECT
    client_id,
    COUNT(*) AS nb_commandes,
    SUM(total) AS total_depense,
    AVG(total) AS moyenne_commande
FROM commandes
GROUP BY client_id;

-- Index sur la vue
CREATE INDEX idx_stats_clients ON stats_clients(client_id);

-- Utilisation (très rapide)
SELECT * FROM stats_clients WHERE client_id = 123;

-- Rafraîchissement (quand nécessaire)
REFRESH MATERIALIZED VIEW CONCURRENTLY stats_clients;
```

**Avantages** :
- ✅ Requête instantanée (données pré-calculées)
- ✅ Pas de recalcul à chaque fois

**Inconvénients** :
- ❌ Données pas en temps réel
- ❌ Nécessite rafraîchissement manuel ou planifié

### 12.3. Utiliser FILTER pour Agrégations Conditionnelles

#### Sous-Optimal : CASE dans l'Agrégation

```sql
SELECT
    client_id,
    SUM(CASE WHEN statut = 'validé' THEN total ELSE 0 END) AS total_valide,
    SUM(CASE WHEN statut = 'annulé' THEN total ELSE 0 END) AS total_annule
FROM commandes
GROUP BY client_id;
```

#### Optimal : FILTER Clause

```sql
SELECT
    client_id,
    SUM(total) FILTER (WHERE statut = 'validé') AS total_valide,
    SUM(total) FILTER (WHERE statut = 'annulé') AS total_annule
FROM commandes
GROUP BY client_id;
```

**Avantages** :
- ✅ Plus lisible
- ✅ Légèrement plus performant

### 12.4. Window Functions vs Sous-Requêtes

#### Sous-Optimal : Sous-Requête pour Chaque Calcul

```sql
SELECT
    id,
    total,
    (SELECT AVG(total) FROM commandes) AS moyenne_globale,
    (SELECT MAX(total) FROM commandes) AS max_global
FROM commandes;
```

**Problème** : Sous-requêtes répétées.

#### Optimal : Window Functions

```sql
SELECT
    id,
    total,
    AVG(total) OVER () AS moyenne_globale,
    MAX(total) OVER () AS max_global
FROM commandes;
```

**Avantages** :
- ✅ Un seul scan de la table
- ✅ Plus efficace

### 12.5. Limiter les Agrégations avec HAVING

```sql
-- Agrégation puis filtrage
SELECT client_id, COUNT(*) AS nb
FROM commandes
GROUP BY client_id
HAVING COUNT(*) > 10;
```

**Optimal** : Le planificateur filtre efficacement après l'agrégation.

---

## 13. PostgreSQL 18 : Nouveautés d'Optimisation

### 13.1. Skip Scan pour Index Multi-Colonnes

**Description** : Permet d'utiliser un index multi-colonnes même sans filtrer la première colonne.

**Avant PostgreSQL 18** :
```sql
CREATE INDEX idx_ventes_region_date ON ventes(region, date_vente);

-- Index NON utilisé
SELECT * FROM ventes WHERE date_vente = '2025-01-01';
```

**PostgreSQL 18** :
```sql
-- Index UTILISÉ avec Skip Scan
SELECT * FROM ventes WHERE date_vente = '2025-01-01';
```

**Détection dans EXPLAIN** :
```
Index Skip Scan using idx_ventes_region_date on ventes
  Skip Cond: (date_vente = '2025-01-01'::date)
```

**Impact** : Réduit le besoin de créer des index redondants.

### 13.2. Optimisation OR → ANY

**Description** : Transformation automatique des OR en ANY pour permettre l'utilisation d'index.

**Avant PostgreSQL 18** :
```sql
-- Seq Scan (OR empêche index)
SELECT * FROM produits WHERE id = 1 OR id = 2 OR id = 3;
```

**PostgreSQL 18** :
```sql
-- Transformation automatique en :
SELECT * FROM produits WHERE id = ANY(ARRAY[1, 2, 3]);
-- Index Scan possible
```

**Pas de changement de code nécessaire** : Optimisation automatique.

### 13.3. Auto-Élimination des Self-Joins

**Description** : Le planificateur détecte et élimine les self-joins inutiles.

**Exemple** :
```sql
-- Requête avec self-join redondant
SELECT e1.nom, e1.salaire
FROM employes e1
JOIN employes e2 ON e1.id = e2.id
WHERE e1.salaire > 50000;
```

**PostgreSQL 18** : Simplifie automatiquement en :
```sql
SELECT nom, salaire FROM employes WHERE salaire > 50000;
```

**Détection dans EXPLAIN** :
```
-- Message dans les logs du planificateur
-- "Self-join eliminated"
```

### 13.4. Réorganisation Automatique des DISTINCT

**Description** : Optimise les requêtes avec plusieurs DISTINCT.

**Exemple** :
```sql
SELECT DISTINCT ON (client_id, produit_id) *
FROM ventes
ORDER BY client_id, produit_id, date_vente DESC;
```

**PostgreSQL 18** : Réorganise automatiquement pour utiliser les index efficacement.

### 13.5. EXPLAIN Enrichi

**Nouvelles métriques** :
- Statistiques I/O par opération
- Génération WAL
- Temps par phase
- Buffers automatiquement affichés

```sql
-- PostgreSQL 18
EXPLAIN (ANALYZE) SELECT ...;

-- Affiche automatiquement :
-- I/O Read Time
-- I/O Write Time
-- WAL Bytes
-- Local Buffers
```

---

## 14. Monitoring en Production

### 14.1. Métriques Clés à Surveiller

#### Métrique 1 : Temps de Réponse Moyen

```sql
SELECT
    ROUND(mean_exec_time::numeric, 2) AS avg_response_ms
FROM pg_stat_statements
WHERE query LIKE '%SELECT%'
ORDER BY mean_exec_time DESC
LIMIT 1;
```

**Alerte** : Si > 100 ms pour requêtes simples.

#### Métrique 2 : Requêtes Lentes (> 1 sec)

```sql
SELECT COUNT(*)
FROM pg_stat_statements
WHERE mean_exec_time > 1000;
```

**Alerte** : Si > 10 requêtes lentes.

#### Métrique 3 : Charge Globale (Temps CPU)

```sql
SELECT
    SUM(total_exec_time) AS total_query_time_ms
FROM pg_stat_statements;
```

**Utilisation** : Suivre l'évolution dans le temps.

#### Métrique 4 : Requêtes Actives

```sql
SELECT COUNT(*)
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%';
```

**Alerte** : Si > `max_connections × 0.8`.

#### Métrique 5 : Requêtes Bloquées

```sql
SELECT COUNT(*)
FROM pg_stat_activity
WHERE wait_event_type IS NOT NULL
  AND state = 'active';
```

**Alerte** : Si > 0 pendant longtemps.

### 14.2. Alertes Recommandées

**Prometheus + Alertmanager** :

```yaml
# Requêtes lentes
- alert: SlowQueries
  expr: pg_stat_statements_mean_exec_time_seconds > 1
  for: 5m
  annotations:
    summary: "Requêtes lentes détectées"

# Connexions saturées
- alert: ConnectionsSaturated
  expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
  for: 5m
  annotations:
    summary: "Connexions proches de la limite"
```

### 14.3. Dashboards (Grafana)

**Panels recommandés** :
- Temps de réponse moyen (temps série)
- Top 10 requêtes les plus lentes
- Nombre de requêtes actives
- Nombre de connexions
- Cache hit ratio
- Locks actifs

**Template Grafana** : https://grafana.com/grafana/dashboards/9628

### 14.4. Logs et Analyse Continue

**Configuration logs** :
```sql
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- 1 sec
ALTER SYSTEM SET log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a ';
ALTER SYSTEM SET log_lock_waits = on;
```

**Analyse quotidienne avec pgBadger** :
```bash
# Script cron quotidien
pgbadger /var/log/postgresql/postgresql-$(date -d yesterday +%Y-%m-%d).log \
    -o /var/www/reports/pgbadger-$(date -d yesterday +%Y-%m-%d).html
```

---

## 15. Checklist d'Audit Complète

### 15.1. Phase 1 : Préparation

- [ ] Installer `pg_stat_statements`
- [ ] Configurer `auto_explain`
- [ ] Activer logs avec `log_min_duration_statement`
- [ ] Configurer monitoring (Prometheus/Grafana)
- [ ] Documenter l'architecture et les tables principales

### 15.2. Phase 2 : Collecte de Données

- [ ] Collecter statistiques pendant 1-7 jours (`pg_stat_statements`)
- [ ] Générer rapport pgBadger
- [ ] Exporter les métriques de performance
- [ ] Identifier les heures de pointe

### 15.3. Phase 3 : Identification des Requêtes Problématiques

- [ ] Top 20 requêtes les plus lentes (moyenne)
- [ ] Top 20 requêtes les plus coûteuses (temps total)
- [ ] Requêtes avec forte variabilité
- [ ] Requêtes générant beaucoup de WAL
- [ ] Requêtes avec locks fréquents

### 15.4. Phase 4 : Analyse Détaillée

Pour chaque requête problématique :

- [ ] Exécuter `EXPLAIN (ANALYZE, BUFFERS)`
- [ ] Identifier le type de scan (Seq Scan sur grande table ?)
- [ ] Vérifier utilisation des index
- [ ] Mesurer cache hit ratio
- [ ] Détecter sous-requêtes corrélées
- [ ] Rechercher N+1 patterns

### 15.5. Phase 5 : Optimisation

- [ ] Créer index manquants
- [ ] Réécrire requêtes inefficaces
- [ ] Implémenter eager loading (anti N+1)
- [ ] Ajouter LIMIT/pagination
- [ ] Optimiser jointures
- [ ] Remplacer sous-requêtes par CTE/JOIN

### 15.6. Phase 6 : Validation

- [ ] Mesurer performances avant/après
- [ ] Vérifier plans d'exécution (EXPLAIN)
- [ ] Tester en environnement de staging
- [ ] Valider sous charge (load testing)
- [ ] Vérifier pas de régression sur autres requêtes

### 15.7. Phase 7 : Documentation

- [ ] Documenter chaque optimisation
- [ ] Créer runbook pour requêtes critiques
- [ ] Définir SLA par type de requête
- [ ] Partager best practices avec l'équipe

### 15.8. Phase 8 : Monitoring Continu

- [ ] Mettre en place alertes
- [ ] Planifier audits réguliers (mensuel/trimestriel)
- [ ] Suivre évolution des métriques
- [ ] Réviser stratégie selon croissance

---

## 16. Cas Pratiques d'Optimisation

### 16.1. Cas 1 : Tableau de Bord Lent

#### Problème Initial

**Requête** :
```sql
-- Temps : 15 secondes
SELECT
    c.nom,
    COUNT(cmd.id) AS nb_commandes,
    SUM(cmd.total) AS total_depense
FROM clients c
LEFT JOIN commandes cmd ON cmd.client_id = c.id
WHERE cmd.date_commande > CURRENT_DATE - INTERVAL '30 days'
GROUP BY c.id, c.nom
ORDER BY total_depense DESC
LIMIT 10;
```

**EXPLAIN montre** :
- `Seq Scan` sur `commandes`
- `HashAggregate` coûteux
- Tri (`Sort`) sur disque (external merge)

#### Optimisations Appliquées

**1. Index sur date_commande** :
```sql
CREATE INDEX idx_commandes_date ON commandes(date_commande);
```

**2. Index composite** :
```sql
CREATE INDEX idx_commandes_client_date ON commandes(client_id, date_commande);
```

**3. Réécriture avec CTE** :
```sql
WITH recent_sales AS (
    SELECT client_id, COUNT(*) AS nb, SUM(total) AS total
    FROM commandes
    WHERE date_commande > CURRENT_DATE - INTERVAL '30 days'
    GROUP BY client_id
)
SELECT c.nom, rs.nb AS nb_commandes, rs.total AS total_depense
FROM recent_sales rs
JOIN clients c ON c.id = rs.client_id
ORDER BY rs.total DESC
LIMIT 10;
```

**4. Augmenter work_mem** (si tri sur disque) :
```sql
SET work_mem = '256MB';
```

#### Résultat

**Temps après optimisation** : 0.2 secondes (75× plus rapide).

### 16.2. Cas 2 : Recherche Produits Lente

#### Problème Initial

**Requête** :
```sql
-- Temps : 5 secondes
SELECT *
FROM produits
WHERE lower(nom) LIKE '%postgresql%'
   OR lower(description) LIKE '%postgresql%';
```

**EXPLAIN montre** :
- `Seq Scan` (pas d'index utilisable)
- Fonction `lower()` empêche utilisation index

#### Optimisations Appliquées

**1. Full-Text Search avec GIN** :
```sql
-- Ajouter colonne tsvector
ALTER TABLE produits ADD COLUMN search_vector tsvector;

-- Calculer le vecteur
UPDATE produits SET search_vector =
    to_tsvector('french', coalesce(nom, '') || ' ' || coalesce(description, ''));

-- Index GIN
CREATE INDEX idx_produits_search ON produits USING gin(search_vector);

-- Trigger pour maintenir à jour
CREATE TRIGGER produits_search_update
BEFORE INSERT OR UPDATE ON produits
FOR EACH ROW EXECUTE FUNCTION
    tsvector_update_trigger(search_vector, 'pg_catalog.french', nom, description);
```

**2. Requête optimisée** :
```sql
SELECT *
FROM produits
WHERE search_vector @@ to_tsquery('french', 'postgresql');
```

#### Résultat

**Temps après optimisation** : 0.05 secondes (100× plus rapide).

### 16.3. Cas 3 : Rapport Mensuel Très Lent

#### Problème Initial

**Requête** :
```sql
-- Temps : 120 secondes (2 minutes)
SELECT
    DATE_TRUNC('day', date_commande) AS jour,
    COUNT(*) AS nb_commandes,
    SUM(total) AS chiffre_affaire,
    AVG(total) AS panier_moyen
FROM commandes
WHERE date_commande >= '2025-01-01'
  AND date_commande < '2025-02-01'
GROUP BY DATE_TRUNC('day', date_commande)
ORDER BY jour;
```

**EXPLAIN montre** :
- `Seq Scan` sur 100 millions de lignes
- Agrégation coûteuse

#### Optimisations Appliquées

**1. Index BRIN sur date** (données séquentielles) :
```sql
CREATE INDEX idx_commandes_date_brin ON commandes USING brin(date_commande);
```

**2. Vue matérialisée pour statistiques quotidiennes** :
```sql
CREATE MATERIALIZED VIEW stats_quotidiennes AS
SELECT
    DATE_TRUNC('day', date_commande) AS jour,
    COUNT(*) AS nb_commandes,
    SUM(total) AS chiffre_affaire,
    AVG(total) AS panier_moyen
FROM commandes
GROUP BY DATE_TRUNC('day', date_commande);

CREATE INDEX idx_stats_jour ON stats_quotidiennes(jour);

-- Rafraîchissement quotidien (cron ou pg_cron)
REFRESH MATERIALIZED VIEW CONCURRENTLY stats_quotidiennes;
```

**3. Requête sur la vue** :
```sql
-- Temps : 0.01 secondes
SELECT * FROM stats_quotidiennes
WHERE jour >= '2025-01-01' AND jour < '2025-02-01'
ORDER BY jour;
```

#### Résultat

**Temps après optimisation** : 0.01 secondes (12000× plus rapide).

### 16.4. Cas 4 : N+1 dans API REST

#### Problème Initial

**Code API (Python/Flask)** :
```python
@app.route('/api/clients')
def get_clients():
    clients = Client.query.all()  # Requête 1
    result = []
    for client in clients:
        result.append({
            'id': client.id,
            'nom': client.nom,
            'nb_commandes': len(client.commandes),  # Requête N (lazy load)
            'total_depense': sum(c.total for c in client.commandes)
        })
    return jsonify(result)
```

**Problème** : 1 + N requêtes (N = nombre de clients).

#### Optimisation

**Eager loading avec joinedload** :
```python
@app.route('/api/clients')
def get_clients():
    # 1 seule requête avec JOIN
    clients = Client.query.options(joinedload(Client.commandes)).all()
    result = []
    for client in clients:
        result.append({
            'id': client.id,
            'nom': client.nom,
            'nb_commandes': len(client.commandes),  # Pas de requête
            'total_depense': sum(c.total for c in client.commandes)
        })
    return jsonify(result)
```

**Ou mieux : Agrégation en SQL** :
```python
@app.route('/api/clients')
def get_clients():
    results = db.session.query(
        Client.id,
        Client.nom,
        func.count(Commande.id).label('nb_commandes'),
        func.sum(Commande.total).label('total_depense')
    ).outerjoin(Commande).group_by(Client.id).all()

    return jsonify([{
        'id': r.id,
        'nom': r.nom,
        'nb_commandes': r.nb_commandes or 0,
        'total_depense': r.total_depense or 0
    } for r in results])
```

#### Résultat

**Temps de réponse API** : 3 secondes → 0.3 secondes (10× plus rapide).

---

## 17. Conclusion et Bonnes Pratiques

### 17.1. Les 15 Règles d'Or de l'Audit de Requêtes

#### 1. Mesurez Avant d'Optimiser
**Règle** : Toujours mesurer les performances avant et après optimisation.

**Outil** : `EXPLAIN (ANALYZE, BUFFERS)` + `pg_stat_statements`.

#### 2. Utilisez pg_stat_statements
**Règle** : Installez et activez `pg_stat_statements` sur toute base de production.

**Avantage** : Visibilité complète sur toutes les requêtes.

#### 3. Analysez les Plans d'Exécution
**Règle** : Pour toute requête lente, regardez le plan avec `EXPLAIN ANALYZE`.

**Focus** : Types de scans, jointures, sorts.

#### 4. Indexez Intelligemment
**Règle** : Créez des index sur les colonnes fréquemment filtrées, triées ou jointes.

**Attention** : Pas trop d'index (pénalise les écritures).

#### 5. Évitez les Sous-Requêtes Corrélées
**Règle** : Remplacez par JOIN, CTE ou window functions.

**Gain** : Souvent 10× à 100× plus rapide.

#### 6. Combattez le N+1
**Règle** : Utilisez eager loading dans les ORM.

**Détection** : Beaucoup d'appels de la même requête dans `pg_stat_statements`.

#### 7. Limitez les Résultats
**Règle** : Utilisez `LIMIT` et pagination.

**Évitez** : `SELECT *` sans limite sur grandes tables.

#### 8. Filtrez Tôt
**Règle** : Appliquez les filtres `WHERE` avant les jointures.

**Note** : PostgreSQL optimise souvent automatiquement (predicate pushdown).

#### 9. Préparez les Requêtes Répétées
**Règle** : Utilisez `PREPARE` ou prepared statements dans les drivers.

**Gain** : Élimine le parsing répété.

#### 10. Surveillez les Statistiques
**Règle** : Exécutez `ANALYZE` régulièrement (ou configurez autovacuum correctement).

**Symptôme** : Estimations très différentes de la réalité dans EXPLAIN.

#### 11. Optimisez work_mem
**Règle** : Augmentez `work_mem` si vous voyez des tris "external merge" (disque).

**Formule** : `work_mem = RAM / (max_connections × 3)`.

#### 12. Utilisez les CTEs Judicieusement
**Règle** : CTE pour lisibilité, mais attention à la matérialisation.

**PostgreSQL 12+** : Contrôlez avec `MATERIALIZED` / `NOT MATERIALIZED`.

#### 13. Profilez en Production
**Règle** : Utilisez `auto_explain` pour capturer passivement les requêtes lentes.

**Config** : `auto_explain.log_min_duration = 1000` (1 sec).

#### 14. Documentez les Optimisations
**Règle** : Documentez pourquoi chaque requête a été optimisée.

**Utilité** : Évite les régressions futures, facilite la maintenance.

#### 15. Auditez Régulièrement
**Règle** : Audit mensuel ou trimestriel selon criticité.

**Évolution** : Les patterns d'accès changent avec le temps.

### 17.2. Checklist Rapide

**Avant de déployer une nouvelle fonctionnalité** :
- [ ] Testez les nouvelles requêtes avec `EXPLAIN ANALYZE`
- [ ] Vérifiez l'absence de N+1
- [ ] Validez la présence des index nécessaires
- [ ] Testez sous charge (load testing)
- [ ] Configurez monitoring et alertes

**Audit mensuel** :
- [ ] Top 20 requêtes les plus lentes
- [ ] Identifier nouveaux patterns N+1
- [ ] Vérifier croissance des temps de réponse
- [ ] Analyser rapport pgBadger
- [ ] Mettre à jour documentation

**Investigation d'un problème de performance** :
- [ ] Identifier la requête problématique (`pg_stat_activity`)
- [ ] Analyser le plan (`EXPLAIN ANALYZE`)
- [ ] Vérifier les statistiques (`ANALYZE`)
- [ ] Tester optimisations en staging
- [ ] Déployer et mesurer

### 17.3. PostgreSQL 18 : Points Clés

Les nouveautés PostgreSQL 18 facilitent l'optimisation :

1. **Skip Scan** : Moins d'index redondants nécessaires
2. **Optimisation OR→ANY** : Requêtes avec OR automatiquement optimisées
3. **Auto-élimination self-joins** : Simplification automatique
4. **EXPLAIN enrichi** : Meilleure observabilité

**Action** : Auditez vos requêtes après migration vers PostgreSQL 18 pour bénéficier des optimisations automatiques.

### 17.4. Outils Recommandés

**Analyse** :
- pg_stat_statements (essentiel)
- EXPLAIN ANALYZE (quotidien)
- pgBadger (hebdomadaire/mensuel)

**Monitoring** :
- Prometheus + postgres_exporter
- Grafana (dashboards)
- auto_explain (capture passive)

**Testing** :
- pgbench (load testing)
- HypoPG (test d'index)

### 17.5. Ressources Complémentaires

#### Documentation
- **PostgreSQL 18 Performance** : https://www.postgresql.org/docs/18/performance-tips.html
- **EXPLAIN** : https://www.postgresql.org/docs/18/using-explain.html

#### Livres
- *PostgreSQL Query Performance Tuning* par Henrietta Dombrovskaya
- *High Performance PostgreSQL for Rails* par Andrew Atkinson

#### Communautés
- Reddit : r/PostgreSQL
- Slack : postgres.slack.com
- Discord : PostgreSQL Community

---

## Conclusion Finale

L'audit de requêtes est un **processus continu** qui nécessite :
- **Discipline** : Audits réguliers
- **Outils** : pg_stat_statements, EXPLAIN, monitoring
- **Expertise** : Compréhension du planificateur PostgreSQL
- **Collaboration** : Entre développeurs et DBAs

**Les requêtes optimisées** sont la clé de la performance d'une application PostgreSQL. Investissez du temps dans l'audit et l'optimisation, les gains sont souvent **spectaculaires** (10× à 1000× plus rapide).

Avec **PostgreSQL 18**, le planificateur est plus intelligent que jamais. Mais comprendre comment il fonctionne reste **essentiel** pour écrire des requêtes efficaces et diagnostiquer les problèmes.

**Dernière recommandation** : **Commencez simple**. Optimisez d'abord les requêtes les plus lentes ou les plus fréquentes. L'impact sera maximal avec un effort minimal.

---


⏭️ [Audit de schéma](/annexes/checklist-performance/04-audit-schema.md)
