🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4. Structure Physique : Heap Files, TOAST et WAL

## 📋 Introduction

Jusqu'à présent, nous avons exploré l'architecture logique de PostgreSQL : les processus, la mémoire. Mais comment PostgreSQL **stocke-t-il réellement les données sur le disque** ? Comment une simple requête `INSERT` se transforme-t-elle en octets sur le disque dur ?

Dans cette section, nous allons plonger dans la **structure physique** de PostgreSQL en explorant trois concepts fondamentaux :
- **Heap Files** : Comment les tables sont stockées  
- **TOAST** : Comment les grandes valeurs sont gérées  
- **WAL** : Comment PostgreSQL garantit la durabilité des données

Comprendre ces mécanismes vous aidera à optimiser vos bases de données, diagnostiquer les problèmes de performance, et mieux appréhender le fonctionnement interne de PostgreSQL.

---

## 🗂️ Vue d'Ensemble : Du Logique au Physique

### De la Table SQL aux Fichiers Disque

```
NIVEAU LOGIQUE (ce que vous voyez)
    |
    v
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username TEXT,
    email TEXT,
    bio TEXT
);
    |
    v
NIVEAU PHYSIQUE (ce que PostgreSQL stocke)
    |
    ├─> Heap File : /var/lib/postgresql/data/base/16384/24576
    ├─> TOAST Table : /var/lib/postgresql/data/base/16384/24576_toast
    └─> WAL Files : /var/lib/postgresql/data/pg_wal/000000010000000000000001
```

### Analogie de la Bibliothèque 📚

Pour comprendre le stockage physique, imaginez une bibliothèque :

**Tables (Heap Files)** = **Étagères principales**
- Chaque livre (ligne) est rangé séquentiellement
- Les livres sont regroupés dans des "pages" (étagères)

**TOAST** = **Entrepôt annexe**
- Les très gros livres (encyclopédies) sont stockés ailleurs
- On garde juste une référence sur l'étagère principale

**WAL** = **Journal de bord**
- Avant de modifier un livre, on note l'action dans un journal
- Si un problème survient, on peut rejouer les actions depuis le journal

---

## 📦 Heap Files : Le Stockage des Tables

### Qu'est-ce qu'un Heap File ?

Un **Heap File** (fichier de tas) est la structure de données physique où PostgreSQL stocke les **lignes d'une table**. Le terme "heap" signifie que les lignes sont stockées **sans ordre particulier** (contrairement à un index).

### Structure Hiérarchique du Stockage

```
BASE DE DONNÉES
    |
    ├─> TABLESPACE (emplacement physique)
    |       |
    |       └─> DATABASE (répertoire)
    |               |
    |               └─> TABLE (fichier heap)
    |                       |
    |                       └─> PAGES (8 KB chacune)
    |                               |
    |                               └─> TUPLES (lignes)
```

### Localisation Physique

Sur le système de fichiers :

```bash
/var/lib/postgresql/18/main/
    ├─> base/                       # Répertoire des bases
    |   ├─> 1/                      # template1 (OID 1)
    |   ├─> 12345/                  # template0
    |   └─> 16384/                  # Votre base de données
    |       ├─> 24576               # Fichier heap de la table users
    |       ├─> 24576.1             # Suite du fichier (si > 1 GB)
    |       ├─> 24576_fsm           # Free Space Map
    |       ├─> 24576_vm            # Visibility Map
    |       └─> 24577               # Index sur users(email)
    |
    └─> pg_wal/                     # Répertoire WAL
        ├─> 000000010000000000000001
        ├─> 000000010000000000000002
        └─> ...
```

**Note** : Les noms de fichiers sont des **OID (Object Identifier)**, des identifiants internes PostgreSQL.

### Trouver l'OID d'une Table

```sql
-- Obtenir l'OID de la table
SELECT oid, relname, relfilenode  
FROM pg_class  
WHERE relname = 'users';  

-- Résultat
  oid  | relname | relfilenode
-------+---------+-------------
 24576 | users   |       24576
```

Le fichier sera donc : `/base/16384/24576`

---

### Structure d'une Page (8 KB)

Une **page** est l'unité de base du stockage PostgreSQL. Chaque page fait **8 KB** (8192 octets).

```
PAGE (8 KB = 8192 octets)
┌──────────────────────────────────────────────────────┐
│  PAGE HEADER (24 octets)                             │
│  - LSN (Log Sequence Number)                         │
│  - Checksum                                          │
│  - Flags, etc.                                       │
├──────────────────────────────────────────────────────┤
│  ITEM POINTERS (Array of Line Pointers)              │
│  - Pointeur vers tuple 1                             │
│  - Pointeur vers tuple 2                             │
│  - ...                                               │
├──────────────────────────────────────────────────────┤
│                                                      │
│          ESPACE LIBRE (Free Space)                   │
│                                                      │
├──────────────────────────────────────────────────────┤
│  TUPLE 3 (ligne de données)                          │
├──────────────────────────────────────────────────────┤
│  TUPLE 2 (ligne de données)                          │
├──────────────────────────────────────────────────────┤
│  TUPLE 1 (ligne de données)                          │
└──────────────────────────────────────────────────────┘
```

**Caractéristiques** :
- Les **tuples** (lignes) sont stockés de bas en haut
- Les **pointeurs** sont stockés de haut en bas
- L'**espace libre** se trouve au milieu
- Quand les deux se rencontrent → Page pleine

### Structure d'un Tuple (Ligne)

Chaque ligne (tuple) contient :

```
TUPLE
┌─────────────────────────────────────────────┐
│  TUPLE HEADER (~23 octets)                  │
│  - t_xmin : Transaction qui a créé la ligne │
│  - t_xmax : Transaction qui a supprimé      │
│  - t_cid : Command ID                       │
│  - t_ctid : Localisation (page, offset)     │
│  - Flags, etc.                              │
├─────────────────────────────────────────────┤
│  NULL BITMAP (si colonnes NULL)             │
├─────────────────────────────────────────────┤
│  DONNÉES DES COLONNES                       │
│  - Colonne 1 : id = 123                     │
│  - Colonne 2 : username = 'alice'           │
│  - Colonne 3 : email = 'alice@example.com'  │
│  - Colonne 4 : bio = (référence TOAST)      │
└─────────────────────────────────────────────┘
```

### MVCC et Versioning des Tuples

PostgreSQL utilise **MVCC (Multi-Version Concurrency Control)** : chaque modification crée une **nouvelle version** du tuple.

#### Exemple : UPDATE d'une Ligne

```sql
-- État initial
INSERT INTO users (id, username) VALUES (1, 'alice');
```

**Page après INSERT** :
```
Page 1
├─> Tuple 1 : id=1, username='alice', t_xmin=100, t_xmax=0 (visible)
```

```sql
-- Modification
UPDATE users SET username = 'Alice' WHERE id = 1;
```

**Page après UPDATE** :
```
Page 1
├─> Tuple 1 : id=1, username='alice', t_xmin=100, t_xmax=101 (MORT, marqué comme supprimé)
└─> Tuple 2 : id=1, username='Alice', t_xmin=101, t_xmax=0 (VIVANT, nouvelle version)
```

**Impact** :
- L'ancienne version reste (pour les transactions en cours)
- La nouvelle version est créée
- L'ancienne sera nettoyée par **VACUUM**

### Fragmentation et Bloat

Avec le temps, les pages se remplissent de tuples "morts" :

```
PAGE (avant VACUUM)
┌────────────────────────────────────────┐
│  Tuple 1 : MORT (t_xmax != 0)          │ ← Espace perdu
│  Tuple 2 : VIVANT                      │
│  Tuple 3 : MORT                        │ ← Espace perdu
│  Tuple 4 : VIVANT                      │
│  Tuple 5 : MORT                        │ ← Espace perdu
│  Tuple 6 : VIVANT                      │
└────────────────────────────────────────┘
```

**Problème** : Table "gonflée" (bloat) → Performance dégradée

**Solution** : VACUUM récupère l'espace :

```
PAGE (après VACUUM)
┌────────────────────────────────────────┐
│  Tuple 2 : VIVANT                      │
│  Tuple 4 : VIVANT                      │
│  Tuple 6 : VIVANT                      │
│  ──────── ESPACE LIBRE ────────────    │
└────────────────────────────────────────┘
```

---

### Free Space Map (FSM) et Visibility Map (VM)

PostgreSQL maintient deux fichiers annexes pour chaque table :

#### 1. Free Space Map (FSM)

Fichier : `24576_fsm`

**Rôle** : Garde la trace de l'espace libre dans chaque page.

```
FSM
┌──────────────────────────────────┐
│  Page 1 : 20% libre              │
│  Page 2 : 80% libre  ← INSERT ici│
│  Page 3 : 10% libre              │
│  Page 4 : 100% libre             │
└──────────────────────────────────┘
```

**Utilité** : Quand un INSERT arrive, PostgreSQL consulte le FSM pour trouver une page avec assez d'espace libre.

#### 2. Visibility Map (VM)

Fichier : `24576_vm`

**Rôle** : Indique quelles pages contiennent **uniquement** des tuples visibles par toutes les transactions (pas de tuples morts).

```
VM
┌──────────────────────────────────┐
│  Page 1 : ✅ All-visible         │ ← VACUUM peut ignorer
│  Page 2 : ❌ Has dead tuples     │ ← VACUUM doit scanner
│  Page 3 : ✅ All-visible         │
│  Page 4 : ❌ Has dead tuples     │
└──────────────────────────────────┘
```

**Utilité** : VACUUM peut **éviter de scanner** les pages marquées "all-visible" → Plus rapide.

---

### Limite de 1 GB par Fichier

PostgreSQL limite la taille d'un fichier heap à **1 GB**. Au-delà, il crée des fichiers numérotés :

```
Table de 3.5 GB
├─> 24576       (1 GB)
├─> 24576.1     (1 GB)
├─> 24576.2     (1 GB)
└─> 24576.3     (500 MB)
```

**Raison** : Portabilité (certains systèmes de fichiers ont des limites) et gestion plus simple.

---

## 🍞 TOAST : The Oversized-Attribute Storage Technique

### Le Problème des Grandes Valeurs

Les pages PostgreSQL font 8 KB. Mais que se passe-t-il si vous stockez une colonne TEXT de 1 MB ?

```sql
CREATE TABLE articles (
    id SERIAL,
    title TEXT,
    content TEXT  -- Peut faire plusieurs MB !
);

INSERT INTO articles (title, content) VALUES
    ('Mon article', 'Lorem ipsum... [1 MB de texte]');
```

**Problème** : Impossible de stocker 1 MB dans une page de 8 KB !

### La Solution : TOAST

**TOAST** (The Oversized-Attribute Storage Technique) est le mécanisme de PostgreSQL pour gérer les valeurs trop grandes.

### Fonctionnement de TOAST

#### Seuil de TOASTage

PostgreSQL "TOAST" une valeur si elle dépasse **~2 KB** (le seuil exact dépend du contexte).

#### Stratégies TOAST

PostgreSQL peut appliquer 4 stratégies :

1. **PLAIN** : Pas de compression ni TOAST (types simples : INTEGER)  
2. **EXTENDED** : Compression + TOAST si nécessaire (par défaut pour TEXT, BYTEA)  
3. **EXTERNAL** : TOAST sans compression (pour données déjà compressées : JSON, images)  
4. **MAIN** : Compression, mais évite TOAST si possible

### Exemple Concret

```sql
CREATE TABLE documents (
    id SERIAL,
    filename TEXT,
    content BYTEA  -- Fichier binaire (image, PDF, etc.)
);

INSERT INTO documents (filename, content) VALUES
    ('rapport.pdf', <données binaires de 5 MB>);
```

#### Étape 1 : PostgreSQL Analyse la Taille

```
content = 5 MB → Trop grand pour une page !  
Strategy = EXTENDED → Compression tentée  
```

#### Étape 2 : Compression (si possible)

```
Taille originale : 5 MB  
Après compression : 4.8 MB (peu compressible, c'est un PDF)  
```

Toujours trop grand !

#### Étape 3 : TOASTage (Découpage)

```
TOAST divise en chunks de ~2 KB :
├─> Chunk 1 (2 KB)
├─> Chunk 2 (2 KB)
├─> Chunk 3 (2 KB)
├─> ...
└─> Chunk 2457 (1.8 KB)
```

#### Étape 4 : Stockage dans TOAST Table

```
TABLE PRINCIPALE (documents)
┌──────────────────────────────────────────┐
│ id=1, filename='rapport.pdf'             │
│ content=(référence TOAST: OID 54321)     │
└──────────────────────────────────────────┘
        │
        │ (référence)
        v
TOAST TABLE (pg_toast_24576)
┌──────────────────────────────────────────┐
│ chunk_id=54321, chunk_seq=0, data=[...]  │
│ chunk_id=54321, chunk_seq=1, data=[...]  │
│ chunk_id=54321, chunk_seq=2, data=[...]  │
│ ...                                      │
│ chunk_id=54321, chunk_seq=2456, data=[..]│
└──────────────────────────────────────────┘
```

### Impact sur les Performances

#### Lecture d'une Ligne TOASTée

```sql
SELECT * FROM documents WHERE id = 1;
```

**Étapes** :
1. Lecture de la ligne principale (rapide)  
2. Détection de la référence TOAST  
3. Lecture de **tous les chunks** dans la TOAST table (lent !)  
4. Réassemblage des chunks  
5. Décompression (si compressé)  
6. Retour au client

**Coût** : Peut être **10-100× plus lent** qu'une lecture normale.

#### Optimisation : Sélectionner Uniquement les Colonnes Nécessaires

```sql
-- ❌ LENT : Récupère tout le contenu TOAST
SELECT * FROM documents WHERE id = 1;

-- ✅ RAPIDE : Ne récupère que le filename (pas TOASTé)
SELECT id, filename FROM documents WHERE id = 1;
```

**Conseil** : **Toujours sélectionner uniquement les colonnes nécessaires** !

---

### Voir les Tables TOAST

```sql
-- Lister les tables TOAST
SELECT
    c.relname as table_name,
    t.relname as toast_table
FROM pg_class c  
JOIN pg_class t ON c.reltoastrelid = t.oid  
WHERE c.relkind = 'r'  -- Tables régulières  
  AND c.relname = 'documents';

-- Résultat
 table_name |      toast_table
------------+------------------------
 documents  | pg_toast_24576
```

### Configurer la Stratégie TOAST

```sql
-- Voir la stratégie actuelle
SELECT attname, attstorage  
FROM pg_attribute  
WHERE attrelid = 'documents'::regclass  
  AND attname = 'content';

-- Résultat
 attname | attstorage
---------+------------
 content | x          -- x = EXTENDED

-- Changer la stratégie (EXTERNAL = pas de compression)
ALTER TABLE documents  
ALTER COLUMN content SET STORAGE EXTERNAL;  
```

**Cas d'usage EXTERNAL** :
- Données déjà compressées (images JPEG, vidéos, archives ZIP)
- Évite la compression inutile (gain de temps)

---

### Monitoring TOAST

```sql
-- Taille de la table principale vs TOAST
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename)) as toast_size
FROM pg_tables  
WHERE tablename = 'documents';  

-- Résultat
 schemaname | tablename | total_size | table_size | toast_size
------------+-----------+------------+------------+------------
 public     | documents |    125 MB  |      5 MB  |    120 MB
```

**Interprétation** : La majorité des données est TOASTée.

---

## 📜 WAL : Write-Ahead Log

### Qu'est-ce que le WAL ?

Le **WAL (Write-Ahead Log)** est un **journal de transactions** qui enregistre **toutes les modifications** avant qu'elles ne soient écrites dans les fichiers de données.

### Le Principe Write-Ahead Logging

**Règle d'Or** : **Avant de modifier une page de données, on écrit la modification dans le WAL.**

```
TRANSACTION
    |
    v
UPDATE users SET name = 'Alice' WHERE id = 1;
    |
    ├─> 1. Écriture dans WAL (séquentielle, rapide)
    |      "UPDATE users id=1, old='alice', new='Alice'"
    |
    └─> 2. Modification en mémoire (Shared Buffers)
           (L'écriture sur disque viendra plus tard)
```

### Pourquoi le WAL ?

#### 1. **Durabilité (ACID : Durability)**

En cas de crash, PostgreSQL peut **rejouer** le WAL pour restaurer l'état cohérent :

```
SCÉNARIO : Crash du Serveur
    |
    ├─> Avant crash :
    |   - Transaction committée
    |   - WAL écrit sur disque ✅
    |   - Page de données PAS ENCORE écrite sur disque ❌
    |
    ├─> CRASH ! 💥
    |
    └─> Recovery :
        - PostgreSQL redémarre
        - Lit le WAL depuis le dernier checkpoint
        - Rejoue les transactions ✅
        - Les données sont récupérées !
```

**Sans WAL** : Perte de données garantie !

#### 2. **Performance**

Écriture WAL (séquentielle) **beaucoup plus rapide** qu'écriture directe (aléatoire) :

```
ÉCRITURE DIRECTE (sans WAL)
    UPDATE users id=1 → Écriture aléatoire sur disque (page 12345)
    UPDATE users id=99 → Écriture aléatoire sur disque (page 54321)
    UPDATE orders id=5 → Écriture aléatoire sur disque (page 99999)

    Coût : 3 × recherche disque (~15-30 ms)

AVEC WAL
    UPDATE users id=1   ┐
    UPDATE users id=99  ├─> Écriture SÉQUENTIELLE dans WAL (~3 ms total)
    UPDATE orders id=5  ┘

    Coût : 1 × écriture séquentielle (~3 ms)
```

**Gain** : **5-10× plus rapide** !

#### 3. **Réplication**

Les serveurs **replicas** rejouent le WAL du serveur **primary** pour rester synchronisés.

```
PRIMARY                          REPLICA
    |                               |
    ├─> Modifications               |
    |   (INSERT, UPDATE, DELETE)    |
    |                               |
    ├─> Écriture WAL                |
    |                               |
    └─> Envoi WAL (streaming) ─────>├─> Réception WAL
                                    |
                                    └─> Rejoue WAL → Base à jour
```

---

### Structure du WAL

#### Fichiers WAL

Le WAL est stocké dans le répertoire `pg_wal/` :

```bash
/var/lib/postgresql/18/main/pg_wal/
├─> 000000010000000000000001   (16 MB)
├─> 000000010000000000000002   (16 MB)
├─> 000000010000000000000003   (16 MB)
├─> 000000010000000000000004   (16 MB)
└─> ...
```

**Caractéristiques** :
- Chaque fichier fait **16 MB**
- Nommage : Timeline + Log Segment Number
- Écriture séquentielle

#### Nomenclature des Fichiers WAL

```
000000010000000000000042
│      │ │      │ │      │
│      │ │      │ └──────┴─> Segment Number (hex)
│      │ └──────┴─────────> Log File Number (hex)
└──────┴───────────────────> Timeline ID (incremental après failover)
```

#### LSN : Log Sequence Number

Chaque enregistrement WAL a un **LSN (Log Sequence Number)** unique :

```
LSN Format : 0/15D5E88

0/       15D5E88
│        │
│        └──> Offset dans le segment
└──────────> Numéro de fichier WAL
```

Le LSN est **strictement croissant** et permet de :
- Identifier une position dans le WAL
- Comparer deux états (lequel est plus récent ?)
- Mesurer la distance entre primary et replica (lag)

---

### Contenu des Enregistrements WAL

Un enregistrement WAL contient :

```
WAL RECORD
┌──────────────────────────────────────────────────────┐
│  Header                                              │
│  - LSN                                               │
│  - Type d'opération (INSERT, UPDATE, etc.)           │
│  - Transaction ID                                    │
├──────────────────────────────────────────────────────┤
│  Données                                             │
│  - Page modifiée (block number)                      │
│  - Offset dans la page                               │
│  - Image complète de la page (FPI, après checkpoint) │
│  - Données modifiées (pour REDO au recovery)         │
└──────────────────────────────────────────────────────┘
```

### Cycle de Vie du WAL

```
1. TRANSACTION DÉBUT
       ↓
2. MODIFICATION EN MÉMOIRE (Shared Buffers)
       ↓
3. ÉCRITURE DANS WAL BUFFERS (en RAM)
       ↓
4. COMMIT → FLUSH WAL SUR DISQUE (fsync)
       ↓
5. CONFIRMATION AU CLIENT ("SUCCESS")
       ↓
6. Background Writer écrit les pages sales
       ↓
7. CHECKPOINT → Toutes les pages sur disque
       ↓
8. RECYCLAGE/SUPPRESSION des anciens fichiers WAL
```

### Configuration du WAL

#### Paramètres Principaux

```sql
-- Niveau de détail du WAL
wal_level = replica  -- minimal | replica | logical

-- Taille max avant checkpoint automatique
max_wal_size = 1GB

-- Taille min à conserver
min_wal_size = 80MB

-- Méthode de synchronisation (durabilité vs performance)
fsync = on                      -- TOUJOURS ON en production !  
synchronous_commit = on         -- on | off | remote_apply  

-- Compression du WAL (PG 15+)
wal_compression = on
```

#### ⚠️ Compromis : Durabilité vs Performance

##### synchronous_commit = on (par défaut)

```
Transaction → WAL écrit sur disque → fsync → Confirmation client
```
- **Avantage** : Durabilité garantie  
- **Inconvénient** : Plus lent (~1-5 ms de latence par transaction)

##### synchronous_commit = off

```
Transaction → WAL écrit en mémoire → Confirmation client immédiate
             ↓
          (flush asynchrone, max 200 ms plus tard)
```
- **Avantage** : Transactions beaucoup plus rapides  
- **Inconvénient** : Risque de perte de quelques transactions récentes en cas de crash

**Cas d'usage** : Logs, données analytiques non critiques.

---

### Archivage du WAL

Pour des sauvegardes et la réplication, PostgreSQL peut **archiver** les anciens fichiers WAL :

```
pg_wal/
├─> 000000010000000000000001  (actif)
├─> 000000010000000000000002  (actif)
└─> 000000010000000000000003  (complet)
        │
        └─> Copié vers archive :
            /backup/wal_archive/000000010000000000000003
```

#### Configuration de l'Archivage

```sql
-- Activer l'archivage
archive_mode = on

-- Commande d'archivage
archive_command = 'cp %p /backup/wal_archive/%f'
```

**Utilité** :
- Point-In-Time Recovery (PITR)
- Réplication avec décalage
- Audit et forensics

---

### Monitoring du WAL

#### 1. Localisation WAL Actuelle

```sql
-- Position LSN actuelle
SELECT pg_current_wal_lsn();

-- Résultat
 pg_current_wal_lsn
--------------------
 0/15D5E88
```

#### 2. Génération de WAL

```sql
-- Statistiques WAL globales
SELECT * FROM pg_stat_wal;

-- Résultat
 wal_records | wal_fpi | wal_bytes  | wal_buffers_full
-------------+---------+------------+------------------
    1234567  |   45678 | 1234567890 |               12
```

**Interprétation** :
- `wal_records` : Nombre d'enregistrements WAL  
- `wal_fpi` : Full Page Images (après checkpoint)  
- `wal_bytes` : Octets de WAL générés  
- `wal_buffers_full` : Combien de fois les buffers WAL étaient pleins (à minimiser)

#### 3. Taille du Répertoire WAL

```sql
-- Taille du répertoire pg_wal
SELECT pg_size_pretty(
    sum(size)
) as wal_size
FROM pg_ls_waldir();

-- Résultat
 wal_size
----------
   256 MB
```

#### 4. Nouveauté PostgreSQL 18 : Statistiques I/O WAL

```sql
-- Statistiques I/O détaillées par backend
SELECT * FROM pg_stat_io  
WHERE context = 'wal';  
```

---

### Point-In-Time Recovery (PITR)

Le WAL permet de restaurer la base à **n'importe quel point dans le temps** :

```
SCÉNARIO : Suppression accidentelle
    |
    ├─> 10:00 AM : Backup complet (base backup)
    ├─> 10:30 AM : Modifications normales
    ├─> 11:00 AM : DROP TABLE users; (ERREUR !) 💥
    ├─> 11:15 AM : Découverte de l'erreur
    |
    └─> Recovery :
        - Restaurer le backup de 10:00 AM
        - Rejouer le WAL jusqu'à 10:59 AM (avant le DROP)
        - Base restaurée à 10:59 AM ✅
```

**Commande** :
```bash
# Restaurer au temps spécifié
recovery_target_time = '2024-11-19 10:59:00'
```

---

## 🔄 Comment Tout Cela Fonctionne Ensemble

### Cycle Complet d'une Transaction

```
1. CLIENT envoie : INSERT INTO users VALUES (1, 'Alice');
       ↓
2. BACKEND crée un enregistrement WAL
       ↓
3. WAL écrit dans WAL BUFFERS (en RAM)
       ↓
4. Modification appliquée en SHARED BUFFERS (page marquée "dirty")
       ↓
5. CLIENT envoie : COMMIT;
       ↓
6. FLUSH WAL sur disque (fsync) → Durabilité garantie ✅
       ↓
7. Confirmation au CLIENT : "INSERT 0 1"
       ↓
8. (Plus tard) Background Writer écrit la page dirty sur disque
       ↓
9. (Plus tard) Checkpointer force toutes les pages dirty sur disque
       ↓
10. WAL peut être recyclé après checkpoint
```

### Exemple avec TOAST et WAL

```sql
INSERT INTO documents (filename, content)  
VALUES ('big_file.pdf', <5 MB de données>);  
```

**Étapes** :
```
1. Analyse : content trop grand (5 MB)
2. Compression tentée
3. Découpage en chunks TOAST
4. ÉCRITURE WAL :
   - Record : INSERT dans documents
   - Record : INSERT chunk 1 dans TOAST table
   - Record : INSERT chunk 2 dans TOAST table
   - ... (2457 records)
5. Modification en mémoire (Shared Buffers)
6. COMMIT → Flush WAL
7. Confirmation client
8. Background tasks écrivent sur disque
```

---

## 📊 Impact sur les Performances

### Table Bloat (Fragmentation)

**Cause** : Mises à jour fréquentes sans VACUUM régulier

```sql
-- Détecter le bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as actual_size,
    round(100 * (pg_total_relation_size(schemaname||'.'||tablename)::numeric -
                 pg_relation_size(schemaname||'.'||tablename)::numeric) /
          NULLIF(pg_total_relation_size(schemaname||'.'||tablename)::numeric, 0), 2) as bloat_ratio
FROM pg_tables  
WHERE schemaname = 'public'  
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;  
```

**Solution** :
```sql
-- VACUUM régulier (automatique via autovacuum)
VACUUM ANALYZE users;

-- VACUUM FULL (réorganise complètement, verrouillant)
VACUUM FULL users;  -- ⚠️ Attention : lock exclusif !
```

### WAL Generation Excessive

**Symptômes** :
- Répertoire pg_wal/ qui grossit continuellement
- Replicas en lag
- I/O disque élevé

**Causes** :
- Mises à jour massives
- Checkpoints trop fréquents
- Pas d'archivage configuré

**Solutions** :
```sql
-- Augmenter max_wal_size
max_wal_size = 4GB

-- Augmenter checkpoint_timeout
checkpoint_timeout = 15min

-- Activer la compression WAL
wal_compression = on

-- Batch les modifications
BEGIN;
    UPDATE massive_table SET status = 'processed';
COMMIT;
```

### TOAST Performance

**Problème** : Requêtes lentes sur tables avec grandes colonnes

**Solutions** :
```sql
-- 1. Sélectionner uniquement les colonnes nécessaires
SELECT id, filename FROM documents;  -- Pas de TOAST

-- 2. Utiliser EXTERNAL pour données incompressibles
ALTER TABLE documents ALTER COLUMN content SET STORAGE EXTERNAL;

-- 3. Partitionner si possible
-- 4. Externaliser les fichiers (S3, filesystem) et stocker uniquement les URLs
```

---

## 🛠️ Outils de Diagnostic

### 1. pg_relation_filepath

```sql
-- Trouver l'emplacement physique d'une table
SELECT pg_relation_filepath('users');

-- Résultat
 pg_relation_filepath
----------------------
 base/16384/24576
```

### 2. pageinspect Extension

```sql
-- Installer l'extension
CREATE EXTENSION pageinspect;

-- Examiner une page
SELECT * FROM heap_page_items(get_raw_page('users', 0));

-- Voir les tuples dans une page
SELECT lp, t_xmin, t_xmax, t_ctid, t_data  
FROM heap_page_items(get_raw_page('users', 0))  
LIMIT 5;  
```

### 3. pg_waldump

Outil en ligne de commande pour inspecter le WAL :

```bash
# Lire un fichier WAL
pg_waldump /var/lib/postgresql/18/main/pg_wal/000000010000000000000001

# Résultat
rmgr: Heap        len (rec/tot):     59/    59, tx:        742, lsn: 0/015D5E50, prev 0/015D5E18, desc: INSERT off 3, blkref #0: rel 1663/16384/24576 blk 0  
rmgr: Heap        len (rec/tot):     60/    60, tx:        743, lsn: 0/015D5E8B, prev 0/015D5E50, desc: UPDATE off 3 xmax 743, blkref #0: rel 1663/16384/24576 blk 0  
```

---

## 🎯 Checklist de Bonnes Pratiques

### Configuration de Base

```ini
# === WAL Configuration ===
wal_level = replica  
max_wal_size = 2GB  
min_wal_size = 80MB  
wal_compression = on  
checkpoint_completion_target = 0.9  

# === Durabilité ===
fsync = on  
synchronous_commit = on  # off seulement pour données non critiques  

# === Archivage (si PITR nécessaire) ===
archive_mode = on  
archive_command = 'cp %p /backup/wal_archive/%f'  
```

### Maintenance Régulière

```sql
-- Activer autovacuum (par défaut)
autovacuum = on

-- VACUUM manuel si nécessaire
VACUUM ANALYZE;

-- Surveiller le bloat régulièrement
-- (requête de monitoring ci-dessus)

-- Vérifier la taille du WAL
SELECT pg_size_pretty(sum(size)) FROM pg_ls_waldir();
```

### Optimisations

- ✅ **Sélectionner uniquement les colonnes nécessaires** (éviter TOAST)  
- ✅ **Utiliser STORAGE EXTERNAL** pour données incompressibles  
- ✅ **Batch les modifications** (BEGIN...COMMIT)  
- ✅ **Monitorer le cache hit ratio** (Shared Buffers)  
- ✅ **Configurer max_wal_size** selon le workload  
- ✅ **Archiver le WAL** pour PITR

---

## 📝 Résumé des Concepts Clés

### Heap Files

- ✅ **Structure de base** pour stocker les tables  
- ✅ **Pages de 8 KB** contenant des tuples (lignes)  
- ✅ **MVCC** : Les updates créent de nouvelles versions  
- ✅ **Bloat** : Fragmentation nécessitant VACUUM  
- ✅ **FSM/VM** : Métadonnées pour optimiser l'accès

### TOAST

- ✅ **Gère les grandes valeurs** (> 2 KB)  
- ✅ **Compression + Découpage** en chunks  
- ✅ **Stockage externe** dans une table TOAST dédiée  
- ✅ **Impact performance** : Sélectionner uniquement les colonnes nécessaires

### WAL

- ✅ **Journal de transactions** pour durabilité  
- ✅ **Write-Ahead** : Écriture WAL avant modification données  
- ✅ **Performance** : Écriture séquentielle vs aléatoire  
- ✅ **Réplication** : Streaming WAL vers replicas  
- ✅ **PITR** : Point-In-Time Recovery  
- ✅ **Fichiers de 16 MB** dans pg_wal/

---

## 🎓 Points à Retenir pour les Débutants

1. **PostgreSQL stocke les données en pages de 8 KB** : Unité de base du stockage.

2. **MVCC crée des versions multiples** : Les updates ne modifient pas en place, ils créent de nouvelles versions.

3. **VACUUM est vital** : Sans lui, les tables "gonflent" et les performances se dégradent.

4. **TOAST gère automatiquement les grandes valeurs** : Mais cela a un coût en performance.

5. **Sélectionner uniquement les colonnes nécessaires** : Évite les lectures TOAST inutiles.

6. **Le WAL garantit la durabilité** : En cas de crash, rien n'est perdu si le WAL est écrit.

7. **Write-Ahead = Performance** : Écriture séquentielle beaucoup plus rapide.

8. **fsync = on TOUJOURS en production** : Ne jamais le désactiver !

9. **Monitorer le bloat et le WAL** : Indicateurs clés de santé de la base.

---

## 🔗 Prochaine Étape

Maintenant que vous comprenez **comment PostgreSQL stocke physiquement les données**, la section suivante explorera les **nouveautés de PostgreSQL 18**, notamment le **sous-système I/O asynchrone (AIO)** qui révolutionne les performances d'accès disque.


⏭️ [Nouveauté PG 18 : Le sous-système I/O asynchrone (AIO)](/03-architecture-de-postgresql/05-sous-systeme-io-asynchrone.md)
