🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2. Anatomie d'une Instance : Postmaster et Processus d'Arrière-Plan

## 📋 Introduction

Lorsque PostgreSQL démarre, il ne s'agit pas d'un simple programme qui s'exécute. C'est tout un **écosystème de processus** qui travaillent ensemble, chacun avec un rôle spécifique. Comprendre cette architecture vous aidera à diagnostiquer les problèmes, optimiser les performances, et mieux appréhender le fonctionnement interne de PostgreSQL.

Dans cette section, nous allons explorer l'anatomie d'une **instance PostgreSQL** : du processus maître (Postmaster) aux nombreux processus d'arrière-plan qui assurent le bon fonctionnement du système.

---

## 🏢 Qu'est-ce qu'une Instance PostgreSQL ?

### Définition

Une **instance PostgreSQL** est un ensemble de processus et de structures en mémoire qui gèrent une ou plusieurs bases de données. C'est l'environnement d'exécution complet de PostgreSQL sur une machine.

### Analogie de l'Entreprise 🏭

Imaginez PostgreSQL comme une **entreprise** :

```
┌────────────────────────────────────────────────────────┐
│                  ENTREPRISE POSTGRESQL                 │
│                                                        │
│  👔 PDG (Postmaster)                                   │
│     └─ Supervise tout, embauche, coordination          │
│                                                        │
│  👨‍💼 Employés (Backend Processes)                       │
│     └─ Chacun traite les demandes d'un client          │
│                                                        │
│  🔧 Services d'Arrière-Plan (Background Workers)       │
│     └─ Maintenance, nettoyage, logs, etc.              │
│                                                        │
│  📊 Mémoire Partagée (Shared Memory)                   │
│     └─ Espace commun accessible à tous                 │
└────────────────────────────────────────────────────────┘
```

---

## 👔 Le Postmaster : Le Chef d'Orchestre

### Rôle et Responsabilités

Le **Postmaster** est le **processus parent** de toute instance PostgreSQL. C'est le premier processus qui démarre et le dernier qui s'arrête. Son nom vient de « Post Master » (maître de poste), en référence au système de messagerie POSTGRES (l'ancêtre de PostgreSQL).

### Responsabilités Principales du Postmaster

1. **Démarrage et Initialisation**
   - Lance l'instance PostgreSQL
   - Initialise la mémoire partagée
   - Charge la configuration (postgresql.conf)
   - Démarre tous les processus d'arrière-plan

2. **Gestion des Connexions Clients**
   - Écoute sur le port réseau (par défaut : 5432)
   - Accepte les nouvelles connexions
   - Authentifie les clients
   - Crée un processus backend dédié pour chaque connexion

3. **Supervision et Surveillance**
   - Surveille tous les processus enfants
   - Redémarre les processus crashés si nécessaire
   - Gère l'arrêt propre de l'instance

4. **Gestion des Signaux**
   - Répond aux commandes administratives (SIGHUP, SIGTERM, etc.)
   - Recharge la configuration sans redémarrage complet

### Cycle de Vie du Postmaster

```
DÉMARRAGE
    |
    v
[Postmaster démarre]
    |
    ├─> Charge postgresql.conf
    ├─> Initialise la mémoire partagée
    ├─> Démarre les processus d'arrière-plan
    └─> Écoute sur le port 5432
    |
    v
[En Attente de Connexions]
    |
    ├─ Nouvelle connexion → Crée un Backend Process
    ├─ Nouvelle connexion → Crée un Backend Process
    └─ Nouvelle connexion → Crée un Backend Process
    |
    v
[Supervision Continue]
    |
    ├─> Surveille les processus enfants
    ├─> Redémarre les processus crashés
    └─> Traite les signaux administratifs
    |
    v
[ARRÊT sur commande (SIGTERM)]
    |
    ├─> Arrête proprement tous les backends
    ├─> Arrête tous les processus d'arrière-plan
    ├─> Écrit un checkpoint final
    └─> Se termine
```

### Commande pour Voir le Postmaster

Sur Linux, vous pouvez voir le processus Postmaster avec :

```bash
ps aux | grep postgres
```

Résultat typique :
```
postgres  1234  0.0  0.5  /usr/lib/postgresql/18/bin/postgres -D /var/lib/postgresql/18/main
  ^         ^
  |         └─ PID (Process ID) du Postmaster
  └─ Utilisateur propriétaire
```

Le premier processus `postgres` listé est le **Postmaster**. Tous les autres sont ses enfants.

---

## 👨‍💼 Les Backend Processes : Un Processus par Client

### Qu'est-ce qu'un Backend Process ?

Un **Backend Process** (ou « backend ») est un processus **dédié à une connexion client**. Quand un client se connecte (via psql, pgAdmin, ou une application), le Postmaster crée un nouveau processus backend qui servira **exclusivement** ce client.

### Modèle « un processus par connexion »

PostgreSQL utilise le modèle ***process-per-connection*** :

```
CLIENT 1                      BACKEND PROCESS 1
  (psql)  ──────────────────>   (PID 1235)
                                    |
CLIENT 2                            |
  (App Web) ────────────────>  BACKEND PROCESS 2
                                (PID 1236)
                                    |
CLIENT 3                            |
  (pgAdmin) ─────────────────> BACKEND PROCESS 3
                                (PID 1237)
```

**Chaque connexion = 1 processus système indépendant**

### Cycle de Vie d'un Backend Process

```
1. CLIENT demande connexion
       ↓
2. POSTMASTER crée un processus backend (fork)
       ↓
3. BACKEND authentifie le client
       ↓
4. CLIENT envoie des requêtes SQL
       ↓
5. BACKEND traite chaque requête :
   - Parse le SQL
   - Planifie l'exécution
   - Exécute la requête
   - Accède aux données (cache ou disque)
   - Renvoie les résultats
       ↓
6. CLIENT ferme la connexion
       ↓
7. BACKEND se termine
```

> 🍴 **Sous le capot : l'appel système `fork()`**  
>  
> Quand le Postmaster doit créer un backend, il appelle `fork()` — un appel système Unix qui duplique le processus parent. Le nouveau processus enfant hérite :  
> - Du code de PostgreSQL déjà en mémoire (zéro copie grâce au *copy-on-write*)  
> - De l'accès à la mémoire partagée (Shared Buffers, etc.)  
> - Des descripteurs de fichiers ouverts  
>  
> Le child appelle ensuite `exec()`-like en interne pour devenir un backend, puis se charge de l'authentification. Ce mécanisme est **rapide** sur Linux (~1 ms), mais **lent sur Windows** où `fork()` n'existe pas — PostgreSQL doit y simuler ce comportement via `CreateProcess()`, ce qui explique pourquoi PostgreSQL est historiquement moins performant sur Windows pour des charges avec beaucoup de connexions courtes.  
>  
> 👉 **Implication** : sur Windows et pour les workloads à fort taux de connexions/déconnexions (serverless, lambdas), **PgBouncer est encore plus indispensable**.

### Responsabilités d'un Backend Process

1. **Traitement des Requêtes SQL**
   - Parse et analyse syntaxique
   - Optimisation (planificateur)
   - Exécution

2. **Gestion de la Mémoire Locale**
   - Allocation de `work_mem` pour les opérations de tri, hachage
   - Gestion du cache local

3. **Accès aux Données**
   - Lecture depuis le cache partagé (shared buffers)
   - Lecture depuis le disque si nécessaire
   - Écriture des modifications (via WAL)

4. **Gestion des Transactions**
   - BEGIN, COMMIT, ROLLBACK
   - Isolation transactionnelle (MVCC)

### Avantages et Inconvénients du Modèle Process-per-Connection

#### ✅ Avantages

- **Isolation Forte** : Un crash d'un backend n'affecte pas les autres  
- **Sécurité** : Séparation stricte des espaces mémoire  
- **Simplicité** : Pas de gestion complexe de threads

#### ❌ Inconvénients

- **Coût Mémoire** : Chaque processus consomme ~10 MB de RAM minimum  
- **Limite de Connexions** : Chaque processus consomme des ressources système  
- **Overhead de Création** : Fork + authentification prend du temps

**Solution** : Utiliser un **Connection Pooler** (PgBouncer) pour mutualiser les connexions.

---

## 🔧 Les Processus d'Arrière-Plan (Background Workers)

Les **processus d'arrière-plan** sont des processus permanents (ou récurrents) qui effectuent des tâches de maintenance, de surveillance et de gestion interne. Ils sont lancés par le Postmaster au démarrage.

### Vue d'Ensemble des Processus d'Arrière-Plan

```
POSTMASTER (parent)
    |
    ├─── Background Writer (bgwriter)
    ├─── Checkpointer
    ├─── WAL Writer (walwriter)
    ├─── WAL Receiver (sur les replicas)
    ├─── WAL Sender (sur le primary)
    ├─── Autovacuum Launcher
    ├─── Autovacuum Workers (0-N)
    ├─── Logical Replication Workers
    └─── Autres Workers (extensions, etc.)
```

Explorons chacun en détail.

---

### 1. 📝 Background Writer (bgwriter)

#### Rôle

Le **Background Writer** écrit les pages « sales » (*dirty pages*) de la mémoire partagée vers le disque de manière **progressive et continue**.

#### Analogie du Nettoyage 🧹

Imaginez un tableau blanc partagé :
- Les backends écrivent dessus (modifications en mémoire)
- Le bgwriter efface progressivement et recopie sur papier (disque)
- Ainsi, le tableau ne se remplit pas trop, et les backends peuvent continuer à travailler

#### Fonctionnement

```
SHARED BUFFERS (en RAM)
┌──────────────────────────┐
│ Page 1 : PROPRE          │
│ Page 2 : SALE (modified) │ ←── Background Writer
│ Page 3 : SALE (modified) │     identifie et écrit
│ Page 4 : PROPRE          │     ces pages sur disque
│ Page 5 : SALE (modified) │
└──────────────────────────┘
         ↓ (écriture progressive)
      DISQUE
```

#### Pourquoi c'est Important ?

- **Évite les à-coups** : Sans bgwriter, toutes les écritures se feraient lors des checkpoints (pics de latence)  
- **Libère de la mémoire** : Les pages propres peuvent être évincées pour charger de nouvelles données  
- **Performance** : Lisse les écritures disque dans le temps

#### Configuration Clé

```sql
-- Fréquence d'activation (millisecondes)
bgwriter_delay = 200ms

-- Nombre max de buffers écrits par cycle
bgwriter_lru_maxpages = 100

-- Multiplicateur pour ajuster dynamiquement
bgwriter_lru_multiplier = 2.0
```

---

### 2. ✅ Checkpointer

#### Rôle

Le **Checkpointer** effectue des **checkpoints** : des points de cohérence où toutes les données modifiées en mémoire sont garanties d'être écrites sur disque.

#### Analogie de la Sauvegarde 💾

C'est comme une **sauvegarde automatique** dans un jeu vidéo :
- Vous jouez (modifications en mémoire)
- À intervalles réguliers, le jeu sauvegarde votre progression (checkpoint)
- Si le jeu crash, vous repartez du dernier checkpoint

#### Fonctionnement d'un Checkpoint

```
1. DÉCLENCHEMENT (tous les N minutes ou M MB de WAL)
       ↓
2. ÉCRITURE de toutes les pages sales sur disque
       ↓
3. SYNCHRONISATION (fsync) pour garantir la durabilité
       ↓
4. MISE À JOUR des métadonnées de checkpoint
       ↓
5. RECYCLAGE des anciens fichiers WAL
```

#### Pourquoi c'est Important ?

- **Récupération Rapide** : Après un crash, PostgreSQL ne rejoue que les WAL depuis le dernier checkpoint  
- **Gestion du WAL** : Permet de supprimer les anciens fichiers WAL  
- **Cohérence** : Garantit un état cohérent des données sur disque

#### Configuration Clé

```sql
-- Fréquence temporelle (5 min par défaut)
checkpoint_timeout = 5min

-- Taille maximale de WAL avant checkpoint (1 GB par défaut)
max_wal_size = 1GB

-- Avertir si les checkpoints sont trop fréquents
checkpoint_warning = 30s

-- Durée d'étalement de l'écriture (0,5 = 50 % du checkpoint_timeout)
checkpoint_completion_target = 0.9
```

#### Impact Performance

⚠️ **Les checkpoints peuvent causer des pics de latence** si mal configurés :
- Trop fréquents → Overhead constant
- Trop rares → Recovery lente + pics d'I/O

**Best Practice** : Ajuster `max_wal_size` et `checkpoint_completion_target` pour lisser les écritures.

---

### 3. 📜 WAL Writer (walwriter)

#### Rôle

Le **WAL Writer** écrit les enregistrements **WAL (Write-Ahead Log)** depuis les buffers mémoire vers les fichiers WAL sur disque.

#### Le Concept de WAL (Write-Ahead Logging)

Le WAL est un **journal de transactions** : avant d'écrire une modification dans les fichiers de données, PostgreSQL l'écrit d'abord dans le WAL.

```
TRANSACTION                    ÉTAPES D'ÉCRITURE
    |
    v
UPDATE users SET ...
    |
    ├──> 1. Écriture WAL (IMMÉDIATE)
    |       "UPDATE users id=123 name='Alice'"
    |
    └──> 2. Modification en mémoire (Shared Buffers)
             (L'écriture sur disque des données viendra plus tard)
```

#### Pourquoi le WAL ?

- ✅ **Durabilité** : En cas de crash, les transactions commitées sont rejouées depuis le WAL  
- ✅ **Performance** : Écriture séquentielle (WAL) vs aléatoire (données) → Plus rapide  
- ✅ **Réplication** : Les replicas rejouent le WAL pour se synchroniser

#### Fonctionnement du WAL Writer

```
BACKENDS (écrivent dans WAL buffers)
    |
    v
WAL BUFFERS (en mémoire, quelques MB)
    |
    v
WAL WRITER (flush périodique)
    |
    v
FICHIERS WAL sur disque (16 MB chacun)
    /var/lib/postgresql/18/main/pg_wal/000000010000000000000001
```

#### Configuration Clé

```sql
-- Taille des buffers WAL en mémoire (défaut: -1 = 1/32 de shared_buffers)
wal_buffers = 16MB

-- Délai max avant flush (millisecondes)
wal_writer_delay = 200ms

-- Flush automatique tous les N commits (asynchrone)
wal_writer_flush_after = 1MB
```

---

### 4. 🔄 WAL Sender et WAL Receiver (Réplication)

#### WAL Sender (sur le Primary)

Le **WAL Sender** envoie les enregistrements WAL aux serveurs **replicas** (standbys) dans le cadre de la réplication streaming.

```
PRIMARY (Master)
    |
    └─── WAL Sender ───> (streaming) ───> REPLICA (Standby)
                                              |
                                          WAL Receiver
```

#### WAL Receiver (sur le Replica)

Le **WAL Receiver** reçoit le WAL du primary et l'applique localement pour maintenir la réplique à jour.

#### Cas d'Usage

- **Haute Disponibilité (HA)** : Si le primary tombe, un replica peut être promu  
- **Load Balancing** : Les lectures peuvent être réparties sur les replicas  
- **Backup** : Les replicas servent de sauvegarde temps réel

---

### 5. 🧹 Autovacuum Launcher et Workers

#### Autovacuum Launcher

Le **Autovacuum Launcher** est le processus qui **supervise et lance les workers autovacuum** selon les besoins.

#### Autovacuum Workers

Les **Autovacuum Workers** sont les processus qui effectuent réellement les opérations de **VACUUM** et **ANALYZE**.

#### Qu'est-ce que VACUUM ?

**VACUUM** est une opération de maintenance critique qui :
1. **Récupère l'espace** : les lignes supprimées/modifiées laissent des « cadavres » (*dead tuples*)
2. **Prévient le wraparound XID** : PostgreSQL utilise des identifiants de transaction (XID) qui bouclent après 2 milliards  
3. **Met à jour les statistiques** (avec ANALYZE)

#### Analogie du Ramassage des Ordures 🗑️

```
TABLE (comme une rue)
┌──────────────────────────┐
│ Ligne 1 : ACTIVE         │
│ Ligne 2 : MORTE (delete) │ ←── Autovacuum nettoie
│ Ligne 3 : ACTIVE         │
│ Ligne 4 : MORTE (update) │ ←── Récupère l'espace
│ Ligne 5 : ACTIVE         │
└──────────────────────────┘
```

Sans VACUUM :
- ❌ Les tables gonflent (bloat)  
- ❌ Les performances se dégradent  
- ❌ Risque de wraparound (crash du serveur !)

#### Fonctionnement de l'Autovacuum

```
AUTOVACUUM LAUNCHER (surveillance)
    |
    ├─> Surveille les tables (pg_stat_user_tables)
    ├─> Détecte les tables nécessitant VACUUM/ANALYZE
    └─> Lance des workers autovacuum (max = autovacuum_max_workers)
            |
            v
    AUTOVACUUM WORKER 1 → VACUUM table_users
    AUTOVACUUM WORKER 2 → VACUUM table_orders
    AUTOVACUUM WORKER 3 → ANALYZE table_products
```

#### Nouveauté PostgreSQL 18 : Autovacuum Amélioré

- **Ajustements dynamiques** : `autovacuum_worker_slots` permet un pool de workers plus flexible  
- **Nouveau paramètre** : `autovacuum_vacuum_max_threshold` pour limiter les VACUUM massifs

#### Configuration Clé

```sql
-- Activer l'autovacuum (CRITIQUE, toujours à ON)
autovacuum = on

-- Nombre max de workers simultanés (défaut: 3)
autovacuum_max_workers = 3

-- Seuil de déclenchement (50 dead tuples + 20 % de la table)
autovacuum_vacuum_threshold = 50  
autovacuum_vacuum_scale_factor = 0.2  

-- Délai entre les exécutions (1 min)
autovacuum_naptime = 1min
```

⚠️ **JAMAIS désactiver l'autovacuum en production !** Sauf si vous gérez manuellement les VACUUM.

---

### 6. 📊 Statistiques d'Activité (Shared Memory Stats)

#### Rôle

Depuis **PostgreSQL 15**, les statistiques d'activité ne sont plus gérées par un processus séparé (l'ancien « Stats Collector » a été supprimé). Les statistiques sont désormais collectées directement en **mémoire partagée** par les backends eux-mêmes, ce qui est plus performant et plus fiable.

#### Où sont les Statistiques ?

Les statistiques sont accessibles via les vues système `pg_stat_*` :

```sql
-- Activité en temps réel
SELECT * FROM pg_stat_activity;

-- Statistiques par table
SELECT * FROM pg_stat_user_tables;

-- Statistiques par index
SELECT * FROM pg_stat_user_indexes;

-- Statistiques par base de données
SELECT * FROM pg_stat_database;
```

#### Importance pour les Développeurs

Ces statistiques sont **essentielles** pour :
- **Monitoring** : Détecter les problèmes de performance  
- **Optimisation** : Identifier les requêtes lentes  
- **Planification** : Le planificateur utilise ces stats pour optimiser les requêtes

> 💡 **Évolution historique** : avant PostgreSQL 15, un processus dédié « Stats Collector » recevait les statistiques via des sockets UDP et les écrivait dans des fichiers temporaires. Ce mécanisme a été remplacé par un accès direct en mémoire partagée, éliminant les problèmes de perte de statistiques et améliorant les performances.

#### Nouveauté PostgreSQL 18

- `pg_stat_io` (introduit en PG 16) reporte désormais les I/O **en octets** plutôt qu'en pages — plus pratique pour l'analyse
- Les statistiques sont maintenant disponibles **par backend** (via `pg_stat_get_backend_io()` et `pg_stat_get_backend_wal()`), pas seulement globales
- Les statistiques WAL ont été consolidées dans `pg_stat_io`

```sql
-- Statistiques I/O globales (PG 16+)
SELECT * FROM pg_stat_io;

-- Statistiques WAL globales (PG 14+)
SELECT * FROM pg_stat_wal;

-- Nouveauté PG 18 : par backend
SELECT pid, pg_stat_get_backend_io(pid) FROM pg_stat_activity;
```

---

### 7. 🔌 Logical Replication Workers

#### Rôle

Les **Logical Replication Workers** gèrent la **réplication logique** : réplication sélective de certaines tables ou bases de données (contrairement à la réplication physique qui réplique tout).

#### Cas d'Usage

- **Migration** : Répliquer depuis PostgreSQL vers une autre base  
- **Réplication Sélective** : Répliquer uniquement certaines tables  
- **Consolidation** : Agréger des données de plusieurs bases

#### Fonctionnement

```
PRIMARY
    |
    └── Publication (tables à répliquer)
            |
            v (streaming logique)
REPLICA
    |
    └── Subscription (reçoit et applique)
```

---

### 8. 📤 Archiver Process

#### Rôle

Le **Archiver Process** (ou *archiver*) copie les fichiers WAL terminés vers un emplacement d'archivage (disque réseau, stockage objet, etc.). C'est la base technique de :
- **PITR** (*Point-In-Time Recovery*) : restaurer la base à un instant précis
- **Réplication asynchrone retardée** : un réplica peut rejouer le WAL avec un délai contrôlé
- **Backup logique long-terme** : conserver l'historique des modifications

#### Fonctionnement

```
pg_wal/                          /backup/wal_archive/
├─> 000000010000000000000001     ├─> 000000010000000000000001  ← Archivé
├─> 000000010000000000000002     └─> 000000010000000000000002  ← Archivé
├─> 000000010000000000000003 ────────────────────────────────────┐
│   (en cours d'écriture)                                        │
└─> 000000010000000000000004     ARCHIVER                        │
                                  └─> Copie chaque fichier WAL  ─┘
                                      terminé via archive_command
```

#### Configuration

```ini
# postgresql.conf
archive_mode = on                          # Active l'archiver  
archive_command = 'cp %p /backup/wal/%f'   # Commande exécutée pour chaque WAL terminé  
# Ou plus robuste : pgBackRest, WAL-G, Barman
```

Le placeholder `%p` est le chemin du fichier source, `%f` est le nom du fichier seul. **L'archive_command doit retourner 0 en cas de succès** ; un échec bloque l'archiver et fait grossir `pg_wal/`.

> ⚠️ **Piège classique** : si `archive_command` échoue silencieusement, le répertoire `pg_wal/` peut se remplir jusqu'à saturer le disque et **bloquer toute écriture** sur la base. Toujours monitorer `pg_stat_archiver` (`failed_count`, `last_failed_time`).

### 9. 🌐 Autres Background Workers

PostgreSQL supporte des **workers personnalisés** via les extensions :

#### pg_cron
- Worker pour planifier des tâches SQL (comme un cron Unix)

#### pglogical
- Réplication logique avancée

#### Extensions Custom
- Les extensions peuvent démarrer leurs propres workers

---

## 🧠 Gestion de la Mémoire Partagée (Shared Memory)

### Qu'est-ce que la Shared Memory ?

La **mémoire partagée** est une zone de RAM accessible à **tous les processus PostgreSQL**. C'est la « mémoire commune » de l'entreprise.

```
SHARED MEMORY
┌────────────────────────────────────┐
│  SHARED BUFFERS (cache de pages)   │ ← Backends lisent/écrivent
│  WAL BUFFERS (journal)             │ ← WAL Writer écrit
│  LOCK TABLE (verrous)              │ ← Coordination
│  PROC ARRAY (processus actifs)     │ ← État des transactions
└────────────────────────────────────┘
         ↑         ↑         ↑
     Backend 1  Backend 2  Backend 3
```

### Composants Principaux

#### 1. Shared Buffers (Cache de Pages)

**Taille** : paramètre `shared_buffers` (défaut : 128 Mo ; recommandé : 25 % de la RAM)

C'est le **cache de pages de données** :
- Les backends lisent d'abord ici avant d'aller sur disque
- Les modifications sont d'abord faites ici
- Background Writer et Checkpointer écrivent sur disque

**Impact Performance** : Plus c'est grand, plus le cache hit ratio est élevé → Moins d'I/O disque

#### 2. WAL Buffers

**Taille** : Paramètre `wal_buffers` (défaut: 1/32 de shared_buffers)

Buffer temporaire pour les enregistrements WAL avant écriture disque.

#### 3. Lock Table

Stocke les **verrous actifs** (row locks, table locks, etc.) pour coordonner les transactions concurrentes.

#### 4. Proc Array

Liste des **processus actifs** et leurs états (actif, idle, en transaction, etc.)

---

## 🔍 Observer les Processus PostgreSQL en Action

### Avec la Commande `ps`

```bash
ps -ef | grep postgres
```

Sortie typique :
```
postgres  1234     1  /usr/lib/postgresql/18/bin/postgres -D /data  ← Postmaster  
postgres  1235  1234  postgres: checkpointer                         ← Checkpointer  
postgres  1236  1234  postgres: background writer                    ← Background Writer  
postgres  1237  1234  postgres: walwriter                            ← WAL Writer  
postgres  1238  1234  postgres: autovacuum launcher                  ← Autovacuum Launcher  
postgres  1240  1234  postgres: alice mydb 192.168.1.10(54321) idle  ← Backend (client Alice)  
postgres  1241  1234  postgres: bob analytics 192.168.1.11(54322) SELECT ← Backend (client Bob)  
```

**Interprétation** :
- **PID 1234** : Postmaster (parent de tous)  
- **PPID 1234** : Tous les autres processus ont 1234 comme parent  
- **Backend** : Indique utilisateur, base, IP client, état (idle, SELECT, INSERT, etc.)

### Avec `pg_stat_activity`

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query
FROM pg_stat_activity  
WHERE state != 'idle';  
```

Résultat :
```
 pid  | usename | application_name | client_addr | state  |        query
------+---------+------------------+-------------+--------+---------------------
 1240 | alice   | psql             | 192.168.1.10| active | SELECT * FROM users;
 1241 | bob     | DBeaver          | 192.168.1.11| idle   | <IDLE>
```

**État possible** :
- `active` : Exécute une requête  
- `idle` : Connexion ouverte, en attente  
- `idle in transaction` : Transaction ouverte mais inactive (⚠️ attention aux locks !)  
- `idle in transaction (aborted)` : Transaction en erreur

---

## ⚡ Impact sur les Performances

### Nombre de Connexions vs Ressources

Chaque backend consomme :
- **~10 MB de RAM** minimum (`work_mem` + overhead)  
- **1 slot de connexion** (limite: `max_connections`)  
- **Ressources CPU** lors de l'exécution

#### Exemple de Calcul de RAM

```
Configuration :
- shared_buffers = 4 GB
- work_mem = 64 MB
- max_connections = 100

RAM minimale requise :
= shared_buffers + (work_mem × max_connections) + overhead OS
= 4 GB + (64 MB × 100) + 2 GB
= 4 GB + 6.4 GB + 2 GB
= 12.4 GB
```

⚠️ **Attention** : Trop de connexions simultanées saturent le serveur !

### Solution : Connection Pooling

Utiliser un **pooler** comme **PgBouncer** :

```
100 CLIENTS                  PGBOUNCER                  POSTGRESQL
    |                            |                           |
    ├──> Connexion 1 ─┐          |                           |
    ├──> Connexion 2 ─┤          |                           |
    ├──> Connexion 3 ─┼──> POOL ─┼──> 10 connexions ─────────┤
    ├──> Connexion 4 ─┤   (200)  |    (max_connections=50)   |
    └──> ...          ─┘         |                           |
```

**Avantage** : 100 clients → Seulement 10 connexions réelles à PostgreSQL

---

## 🛠️ Gestion des Processus (Administration)

### Démarrer PostgreSQL

```bash
# Via systemd (recommandé)
sudo systemctl start postgresql

# Manuellement (avec pg_ctl)
pg_ctl -D /var/lib/postgresql/18/main start
```

### Arrêter PostgreSQL

```bash
# Arrêt propre (attend la fin des requêtes)
sudo systemctl stop postgresql

# Arrêt rapide (SIGINT, annule les transactions en cours)
pg_ctl -D /var/lib/postgresql/18/main stop -m fast

# Arrêt d'urgence (SIGQUIT, sans nettoyage, recovery au redémarrage)
pg_ctl -D /var/lib/postgresql/18/main stop -m immediate
```

### Recharger la Configuration (sans redémarrage)

```bash
# Envoie SIGHUP au Postmaster
sudo systemctl reload postgresql

# Ou
pg_ctl reload
```

**Impact** : certains paramètres sont rechargés à chaud (ex. : `work_mem`), d'autres nécessitent un redémarrage complet (ex. : `shared_buffers`). Pour savoir lequel, consultez la colonne `context` dans `pg_settings` (`postmaster` = redémarrage requis, `sighup` = reload, `user` = SET dans la session).

### Les Signaux Unix : la communication avec PostgreSQL

PostgreSQL utilise les **signaux Unix** pour piloter ses processus à distance. C'est le mécanisme sous-jacent derrière `pg_ctl` et `systemctl`. Connaître les signaux principaux aide à comprendre les logs et à diagnostiquer.

| Signal | Envoyé au | Effet | Équivalent `pg_ctl` |
|--------|-----------|-------|---------------------|
| **`SIGHUP`** | Postmaster | Recharge `postgresql.conf` + `pg_hba.conf` (à chaud) | `pg_ctl reload` |
| **`SIGTERM`** | Postmaster | Arrêt « smart » : attend la fin des transactions en cours | `pg_ctl stop` (par défaut) |
| **`SIGINT`** | Postmaster | Arrêt « fast » : annule les transactions, débranche les clients | `pg_ctl stop -m fast` |
| **`SIGQUIT`** | Postmaster | Arrêt « immediate » : crash simulé, recovery au redémarrage | `pg_ctl stop -m immediate` |
| **`SIGTERM`** | Backend (PID) | Termine cette connexion proprement | `pg_terminate_backend(pid)` |
| **`SIGINT`** | Backend (PID) | Annule la requête en cours (sans fermer la connexion) | `pg_cancel_backend(pid)` |

**Exemple en pratique** :

```bash
# Recharger la configuration
sudo systemctl reload postgresql
# Équivalent : kill -HUP <pid_postmaster>

# Voir le PID du Postmaster
cat /var/lib/postgresql/18/main/postmaster.pid | head -1

# Annuler une requête trop longue depuis psql
SELECT pg_cancel_backend(12345);
```

> 🚨 **`SIGKILL` (kill -9) : à ne JAMAIS utiliser !**  
>  
> Contrairement aux signaux ci-dessus, `SIGKILL` est non-interceptable : il tue brutalement le processus sans nettoyage. Sur un backend, cela force le Postmaster à **tuer tous les autres backends** et à passer en mode recovery (par sécurité, on ne sait pas si la mémoire partagée a été corrompue). Sur le Postmaster lui-même, c'est un crash brutal qui force une recovery au redémarrage. Toujours préférer `SIGTERM` (gracieux).

### Tuer un Backend Spécifique

⚠️ **À utiliser avec précaution !**

```sql
-- Terminer proprement une connexion
SELECT pg_terminate_backend(1240);

-- Annuler la requête en cours (sans tuer la connexion)
SELECT pg_cancel_backend(1240);
```

**Cas d'usage** :
- Requête bloquante qui dure trop longtemps
- Connexion zombie en `idle in transaction`

---

## 🚨 Pannes et Recovery

### Que se passe-t-il en cas de Crash ?

Si un **backend crash** :
1. Le Postmaster détecte la mort du processus  
2. Le Postmaster **tue tous les autres backends** (pour éviter la corruption)  
3. PostgreSQL entre en **recovery mode**  
4. Le serveur rejoue les enregistrements WAL depuis le dernier checkpoint  
5. Une fois la recovery terminée, le serveur redémarre normalement

### Analogie de la Sécurité Incendie 🔥

Si un étage prend feu dans un immeuble :
1. L'alarme sonne (Postmaster détecte le crash)  
2. Tout le monde évacue (tous les backends sont tués)  
3. Les pompiers interviennent (recovery depuis les logs)  
4. L'immeuble rouvre une fois sécurisé (redémarrage)

### Pourquoi Tuer Tous les Backends ?

PostgreSQL utilise la **mémoire partagée**. Si un processus corrompt cette mémoire, tous les processus sont potentiellement affectés. C'est une **mesure de sécurité** pour garantir l'intégrité des données.

---

## 📝 Résumé des Concepts Clés

### L'Instance PostgreSQL

- ✅ Un **écosystème de processus** travaillant ensemble  
- ✅ Coordonnés par le **Postmaster** (processus parent)  
- ✅ Partagent une **zone mémoire commune** (shared memory)

### Le Postmaster

- ✅ Lance et supervise tous les processus  
- ✅ Gère les connexions clientes  
- ✅ Assure la cohérence et la stabilité

### Les Backend Processes

- ✅ **Un processus par connexion client**  
- ✅ Traite les requêtes SQL  
- ✅ Isole les clients les uns des autres

### Les Processus d'Arrière-Plan

| Processus             | Rôle Principal                                    |
|-----------------------|---------------------------------------------------|
| **Background Writer** | Écrit progressivement les pages sales sur disque  |
| **Checkpointer**      | Crée des points de cohérence (checkpoints)        |
| **WAL Writer**        | Écrit le journal WAL sur disque                   |
| **WAL Sender**        | Envoie le WAL aux replicas (réplication)          |
| **WAL Receiver**      | Reçoit le WAL (sur les replicas)                  |
| **Autovacuum Launcher**| Lance les workers autovacuum                     |
| **Autovacuum Workers**| Nettoient les tables (VACUUM/ANALYZE)             |

---

## 🎯 Points à Retenir pour les Débutants

1. **PostgreSQL n'est pas un seul processus**, mais une famille de processus collaborant ensemble.

2. **Le Postmaster est le chef** : il gère tout et surveille tout.

3. **Chaque connexion client = 1 processus backend** : attention aux limites de `max_connections`.

4. **Les processus d'arrière-plan sont invisibles mais essentiels** : ils assurent la maintenance, la performance et la durabilité.

5. **La mémoire partagée est critique** : c'est le « tableau blanc commun » de tous les processus.

6. **Jamais désactiver l'autovacuum** : c'est vital pour la santé de la base.

7. **En cas de crash d'un backend, tous sont tués** : c'est une mesure de sécurité pour protéger l'intégrité des données.

8. **Utilisez un connection pooler** (PgBouncer) pour les applications avec beaucoup de connexions.

---

## 🔗 Prochaine Étape

Maintenant que vous comprenez **comment PostgreSQL s'organise en processus**, la section suivante explorera **comment PostgreSQL gère sa mémoire** : la différence entre mémoire partagée et mémoire locale, et comment optimiser ces paramètres.


⏭️ [Gestion de la mémoire : Shared Buffers vs Local Memory](/03-architecture-de-postgresql/03-gestion-de-la-memoire.md)
