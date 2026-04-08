🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.4. Partitionnement de tables (Déclaratif : Range, List, Hash)

## Introduction

Le **partitionnement de tables** est l'une des techniques les plus puissantes de PostgreSQL pour gérer efficacement de **très grandes tables** contenant des millions, voire des milliards de lignes. Cette fonctionnalité permet de diviser une grande table en plusieurs tables plus petites appelées **partitions**, tout en présentant une interface unifiée pour l'utilisateur et l'application.

### Analogie Simple

Imaginez une bibliothèque municipale avec un million de livres :

**Sans partitionnement :**
```
┌─────────────────────────────────────────────┐
│  Bibliothèque : 1 million de livres         │
│  Tous mélangés sur une seule grande étagère │
│                                             │
│  Chercher un livre = parcourir 1M de livres │
└─────────────────────────────────────────────┘
```

**Avec partitionnement :**
```
┌─────────────────────────────────────────────┐
│         Bibliothèque (vue unifiée)          │
└─────────────────────────────────────────────┘
        ↓ Organisée en sections
┌──────────┬──────────┬──────────┬──────────┐
│ Section  │ Section  │ Section  │ Section  │
│ 2020     │ 2021     │ 2022     │ 2023     │
│ (250k)   │ (250k)   │ (250k)   │ (250k)   │
└──────────┴──────────┴──────────┴──────────┘

Chercher un livre de 2023 = chercher dans 250k livres au lieu de 1M
```

Le partitionnement fonctionne selon le même principe : diviser pour mieux régner.

---

## Qu'est-ce que le Partitionnement ?

### Définition

Le **partitionnement** consiste à diviser une table logique en plusieurs tables physiques plus petites (partitions) selon un critère défini (date, catégorie, hash, etc.). Pour l'utilisateur et l'application, la table partitionnée reste transparente : on continue d'interroger la table principale, et PostgreSQL se charge de router les requêtes vers les bonnes partitions.

### Architecture

```
Application
    ↓
Requête : SELECT * FROM ventes WHERE date = '2024-05-15'
    ↓
┌─────────────────────────────────────┐
│   ventes (table partitionnée)       │  ← Table parent (virtuelle)
│   Interface unifiée                 │
└─────────────────────────────────────┘
    ↓ PostgreSQL route automatiquement
┌─────────────┐
│ ventes_2024 │  ← Partition contenant les données
│ (1M lignes) │
└─────────────┘

Les partitions ventes_2023 et ventes_2025 sont ignorées
```

**Points clés :**
- La table parent (`ventes`) est une **table virtuelle** qui ne contient pas de données
- Les données sont stockées dans les **partitions** (`ventes_2023`, `ventes_2024`, etc.)
- PostgreSQL **route automatiquement** les requêtes vers les bonnes partitions
- L'application interroge toujours la table parent

---

## Pourquoi Partitionner ?

### Problèmes avec les Très Grandes Tables

Lorsqu'une table atteint des dizaines ou centaines de millions de lignes :

❌ **Performance dégradée**
- Les scans séquentiels deviennent extrêmement lents
- Les index deviennent énormes et moins efficaces
- La mémoire cache ne peut plus contenir les données actives

❌ **Maintenance difficile**
- VACUUM peut prendre des heures
- REINDEX bloque la table pendant longtemps
- Les sauvegardes deviennent massives

❌ **Archivage complexe**
- Supprimer des données anciennes (DELETE) est très lent
- Génère beaucoup de WAL et de bloat

### Solutions Apportées par le Partitionnement

✅ **Amélioration des performances**
- Les requêtes ne scannent que les partitions pertinentes (partition pruning)
- Index plus petits et plus efficaces sur chaque partition
- Meilleure utilisation du cache

✅ **Maintenance simplifiée**
- VACUUM/ANALYZE peut être fait partition par partition
- Moins d'impact sur les performances
- Maintenance parallélisable

✅ **Gestion du cycle de vie**
- Archivage rapide : détacher une partition entière
- Purge instantanée : DROP partition au lieu de DELETE
- Ajout de nouvelles partitions sans impact

✅ **Scalabilité**
- Distribution des données sur plusieurs disques/tablespaces
- Possibilité de parallélisation
- Gestion de croissance continue

---

## Évolution : Héritage vs Partitionnement Déclaratif

### Historique

PostgreSQL a longtemps utilisé **l'héritage de tables** pour simuler le partitionnement (avant PostgreSQL 10). Cette approche avait de nombreuses limitations.

**Ancien système (héritage) :**
```sql
-- Avant PostgreSQL 10
CREATE TABLE ventes (
    vente_id SERIAL PRIMARY KEY,
    date_vente DATE
);

CREATE TABLE ventes_2024 (
    CHECK (date_vente >= '2024-01-01' AND date_vente < '2025-01-01')
) INHERITS (ventes);
```

**Limitations de l'héritage :**
- ❌ Contraintes d'intégrité (PK, FK) non propagées  
- ❌ Index non hérités automatiquement  
- ❌ Gestion manuelle complexe  
- ❌ Routage des données à gérer via triggers

### Partitionnement Déclaratif (PostgreSQL 10+)

Depuis **PostgreSQL 10**, le partitionnement déclaratif est natif et résout tous ces problèmes.

**Nouveau système (déclaratif) :**
```sql
-- PostgreSQL 10+
CREATE TABLE ventes (
    vente_id BIGSERIAL,
    date_vente DATE NOT NULL,
    montant NUMERIC(10,2)
) PARTITION BY RANGE (date_vente);  -- Déclaration du partitionnement

CREATE TABLE ventes_2024 PARTITION OF ventes
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Avantages du partitionnement déclaratif :**
- ✅ Syntaxe simple et claire  
- ✅ Routage automatique des INSERT/UPDATE/DELETE  
- ✅ Index créés automatiquement sur les partitions  
- ✅ Partition pruning natif et optimisé  
- ✅ Support des contraintes (avec limitations)  
- ✅ Maintenance simplifiée

**Recommandation :** Utilisez toujours le **partitionnement déclaratif** pour les nouveaux projets. L'héritage de tables n'est plus recommandé pour le partitionnement.

---

## Les Trois Stratégies de Partitionnement

PostgreSQL propose trois stratégies principales, chacune adaptée à des cas d'usage spécifiques :

### 1. RANGE Partitioning (Par Plage)

**Principe :** Diviser les données selon des plages de valeurs continues.

```sql
CREATE TABLE ventes (...) PARTITION BY RANGE (date_vente);

CREATE TABLE ventes_2023 PARTITION OF ventes
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE ventes_2024 PARTITION OF ventes
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Cas d'usage typiques :**
- 📅 **Dates et timestamps** (le plus courant)  
- 🔢 **Séquences numériques** (IDs séquentiels)  
- 📊 **Scores, notes, valeurs** (ex: 0-100, 100-200, etc.)

**Exemple :** Logs d'application partitionnés par mois, données de ventes par année

### 2. LIST Partitioning (Par Liste)

**Principe :** Diviser les données selon des valeurs discrètes spécifiques.

```sql
CREATE TABLE commandes (...) PARTITION BY LIST (pays);

CREATE TABLE commandes_europe PARTITION OF commandes
    FOR VALUES IN ('FR', 'DE', 'IT', 'ES', 'UK');

CREATE TABLE commandes_asie PARTITION OF commandes
    FOR VALUES IN ('JP', 'CN', 'IN', 'KR');
```

**Cas d'usage typiques :**
- 🌍 **Régions géographiques** (pays, continents)  
- 🏷️ **Catégories** (type de produit, statut)  
- 🎯 **Segments de clients** (B2B, B2C, etc.)  
- 📝 **États** (actif, archivé, supprimé)

**Exemple :** Données utilisateurs par région, produits par catégorie

### 3. HASH Partitioning (Par Hash)

**Principe :** Distribuer les données uniformément via une fonction de hachage.

```sql
CREATE TABLE evenements (...) PARTITION BY HASH (utilisateur_id);

CREATE TABLE evenements_p0 PARTITION OF evenements
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE evenements_p1 PARTITION OF evenements
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- etc.
```

**Cas d'usage typiques :**
- ⚖️ **Distribution uniforme** (pas de critère naturel)  
- 🔥 **Éviter les hot spots** (répartir la charge)  
- 🚀 **Parallélisation** (traitement concurrent)

**Exemple :** Données IoT, sessions utilisateurs, événements avec forte concurrence

### Comparaison Rapide

| Critère | RANGE | LIST | HASH |
|---------|-------|------|------|
| **Facilité** | ⭐⭐⭐ Simple | ⭐⭐⭐ Simple | ⭐⭐ Modéré |
| **Cas le plus fréquent** | ✅ Oui (dates) | ⚠️ Moyen | ❌ Rare |
| **Archivage facile** | ✅ Excellent | ✅ Bon | ❌ Difficile |
| **Distribution uniforme** | ❌ Variable | ❌ Variable | ✅ Garantie |
| **Requêtes par plage** | ✅ Optimisées | ❌ Scan complet | ❌ Scan complet |

**Note :** Les détails de chaque stratégie sont couverts dans la section **11.4.1**.

---

## Quand Utiliser le Partitionnement ?

### ✅ Vous DEVRIEZ Partitionner Si...

#### 1. Table Très Volumineuse (> 10-100 GB)

```
Taille de table recommandée pour partitionnement :
├─ < 10 GB     : Pas nécessaire (sauf cas spécifique)
├─ 10-100 GB   : Envisager selon cas d'usage
├─ 100 GB-1 TB : Fortement recommandé
└─ > 1 TB      : Indispensable
```

**Exemple :** Table de logs avec 500 millions de lignes (200 GB)

#### 2. Dimension Temporelle Claire

Si vos données ont une composante temps forte :
- Logs d'application
- Historique de transactions
- Mesures de capteurs
- Événements utilisateurs

**Critère :** 80%+ des requêtes filtrent sur une date

#### 3. Besoin d'Archivage Régulier

Suppression régulière de données anciennes :
- Rétention de 30 jours, 6 mois, 2 ans, etc.
- Conformité RGPD (suppression de données)

**Avantage :** Suppression instantanée d'une partition entière vs DELETE lent

#### 4. Maintenance Problématique

Si les opérations de maintenance prennent trop de temps :
- VACUUM dure > 1 heure
- REINDEX bloque trop longtemps
- Backups trop volumineux

**Solution :** Maintenance partition par partition

#### 5. Isolation de Données (Hot vs Cold)

Séparer données actives (hot) et anciennes (cold) :
- Les données récentes sur SSD rapide
- Les données anciennes sur HDD lent
- Optimisation des ressources

#### 6. Requêtes Ciblées sur Sous-ensemble

Si vos requêtes ciblent toujours un sous-ensemble :
- "Ventes du dernier mois"  
- "Commandes de la région EMEA"  
- "Transactions du client X"

**Bénéfice :** Partition pruning élimine les partitions inutiles

### ❌ Vous NE DEVRIEZ PAS Partitionner Si...

#### 1. Table Petite ou Moyenne (< 10 GB)

**Problème :** Overhead du partitionnement > bénéfices

```sql
-- ❌ Inutile : table de 2 GB
CREATE TABLE utilisateurs (...) PARTITION BY RANGE (date_inscription);
```

**Alternative :** Index classiques suffisent

#### 2. Pas de Critère de Partitionnement Clair

Si vous ne savez pas sur quelle colonne partitionner :
- Pas de dimension temporelle
- Pas de catégories naturelles
- Données uniformément distribuées sans critère

**Indication :** Vous hésitez entre plusieurs colonnes

#### 3. Requêtes Touchant Toutes les Données

Si vos requêtes scannent systématiquement toute la table :
- Agrégations globales sans filtre
- Statistiques sur l'ensemble
- Pas de clause WHERE discriminante

**Résultat :** Aucun bénéfice du partition pruning

#### 4. Trop de Partitions Prévues (> 1000)

**Problème :** Overhead du planificateur PostgreSQL

```
Nombre de partitions :
├─ 10-100   : Excellent
├─ 100-500  : Bon
├─ 500-1000 : Acceptable
└─ > 1000   : Problématique (dégradation performances)
```

**Alternative :** Revoir la granularité (ex: mensuel au lieu de quotidien)

#### 5. Complexité Non Justifiée

Le partitionnement ajoute de la complexité :
- Gestion des partitions
- Scripts de maintenance
- Formation de l'équipe

**Question :** Le gain justifie-t-il l'effort ?

#### 6. Forte Concurrence en Écriture sur Partition Unique

Si toutes les écritures vont dans la même partition :
- Inserts massifs dans partition actuelle
- Hot spot non résolu

**Alternative :** Considérer HASH partitioning ou sharding

---

## Syntaxe de Base du Partitionnement Déclaratif

### Étape 1 : Créer la Table Parent

```sql
CREATE TABLE nom_table (
    colonne1 TYPE,
    colonne2 TYPE,
    ...
) PARTITION BY {RANGE | LIST | HASH} (colonne_partition);
```

**Points importants :**
- La table parent ne contient **aucune donnée** physiquement
- Elle définit la structure commune à toutes les partitions
- La colonne de partitionnement doit être **NOT NULL** (recommandé)

### Étape 2 : Créer les Partitions

**Pour RANGE :**
```sql
CREATE TABLE nom_partition PARTITION OF nom_table
    FOR VALUES FROM (valeur_min) TO (valeur_max);
```

**Pour LIST :**
```sql
CREATE TABLE nom_partition PARTITION OF nom_table
    FOR VALUES IN (valeur1, valeur2, ...);
```

**Pour HASH :**
```sql
CREATE TABLE nom_partition PARTITION OF nom_table
    FOR VALUES WITH (MODULUS nombre_total, REMAINDER reste);
```

### Étape 3 : Utiliser la Table Normalement

```sql
-- Les INSERT sont routés automatiquement vers la bonne partition
INSERT INTO nom_table VALUES (...);

-- Les SELECT interrogent toutes les partitions (avec pruning)
SELECT * FROM nom_table WHERE ...;

-- Les UPDATE/DELETE fonctionnent normalement
UPDATE nom_table SET ... WHERE ...;  
DELETE FROM nom_table WHERE ...;  
```

**Transparence totale :** L'application n'a pas besoin de connaître les partitions

---

## Exemple Complet Étape par Étape

Créons une table de ventes partitionnée par année.

### Étape 1 : Créer la Table Partitionnée

```sql
-- Table parent
CREATE TABLE ventes (
    vente_id BIGSERIAL,
    date_vente DATE NOT NULL,
    client_id INTEGER NOT NULL,
    produit_id INTEGER NOT NULL,
    quantite INTEGER NOT NULL,
    montant NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (date_vente);
```

**Note :** `date_vente` est la colonne de partitionnement.

### Étape 2 : Créer les Partitions

```sql
-- Partition pour 2023
CREATE TABLE ventes_2023 PARTITION OF ventes
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

-- Partition pour 2024
CREATE TABLE ventes_2024 PARTITION OF ventes
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Partition pour 2025
CREATE TABLE ventes_2025 PARTITION OF ventes
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

**Important :** Les bornes sont **[min, max)** (min inclusif, max exclusif)

### Étape 3 : Créer les Index

```sql
-- Index sur la table parent → créé automatiquement sur toutes les partitions
CREATE INDEX idx_ventes_date ON ventes(date_vente);  
CREATE INDEX idx_ventes_client ON ventes(client_id);  
CREATE INDEX idx_ventes_montant ON ventes(montant);  
```

**Vérification :**
```sql
-- Lister les index par partition
SELECT
    schemaname,
    tablename,
    indexname
FROM pg_indexes  
WHERE tablename LIKE 'ventes%'  
ORDER BY tablename, indexname;  
```

**Résultat :**
```
tablename    | indexname
-------------+---------------------------
ventes_2023  | ventes_2023_date_idx  
ventes_2023  | ventes_2023_client_idx  
ventes_2023  | ventes_2023_montant_idx  
ventes_2024  | ventes_2024_date_idx  
...
```

### Étape 4 : Insérer des Données

```sql
-- Insertion dans la table parent
INSERT INTO ventes (date_vente, client_id, produit_id, quantite, montant)  
VALUES  
    ('2023-05-15', 1001, 501, 2, 150.00),
    ('2024-08-22', 1002, 502, 1, 200.00),
    ('2025-11-10', 1003, 503, 5, 350.00);

-- PostgreSQL route automatiquement chaque ligne vers sa partition
```

**Vérification du routage :**
```sql
SELECT COUNT(*) FROM ventes_2023;  -- 1  
SELECT COUNT(*) FROM ventes_2024;  -- 1  
SELECT COUNT(*) FROM ventes_2025;  -- 1  
SELECT COUNT(*) FROM ventes;       -- 3 (total)  
```

### Étape 5 : Interroger les Données

```sql
-- Requête avec partition pruning
EXPLAIN SELECT * FROM ventes WHERE date_vente = '2024-06-15';
```

**Plan d'exécution :**
```
Seq Scan on ventes_2024 ventes
  Filter: (date_vente = '2024-06-15'::date)
```

PostgreSQL scanne **uniquement** `ventes_2024`, pas les autres partitions !

```sql
-- Requête sans filtre sur la colonne de partitionnement
EXPLAIN SELECT * FROM ventes WHERE client_id = 1001;
```

**Plan d'exécution :**
```
Append
  -> Seq Scan on ventes_2023 ventes_1
       Filter: (client_id = 1001)
  -> Seq Scan on ventes_2024 ventes_2
       Filter: (client_id = 1001)
  -> Seq Scan on ventes_2025 ventes_3
       Filter: (client_id = 1001)
```

Sans filtre sur `date_vente`, toutes les partitions sont scannées.

---

## Contraintes et Limitations

### 1. Clés Primaires et Uniques

**Limitation :** Les clés primaires et contraintes UNIQUE doivent **inclure la colonne de partitionnement**.

```sql
-- ❌ Erreur : PK sans colonne de partitionnement
CREATE TABLE ventes (
    vente_id BIGSERIAL PRIMARY KEY,  -- Erreur !
    date_vente DATE NOT NULL,
    montant NUMERIC(10,2)
) PARTITION BY RANGE (date_vente);
-- ERROR: primary key constraint must include all partitioning columns

-- ✅ Solution : PK composite incluant la colonne de partitionnement
CREATE TABLE ventes (
    vente_id BIGSERIAL,
    date_vente DATE NOT NULL,
    montant NUMERIC(10,2),
    PRIMARY KEY (vente_id, date_vente)  -- OK
) PARTITION BY RANGE (date_vente);
```

**Raison :** L'unicité doit être vérifiable localement dans chaque partition.

### 2. Clés Étrangères

**Limitations :**
- ✅ Une partition **peut** référencer une table normale (FK sortante)  
- ❌ Une table normale **ne peut pas** facilement référencer une table partitionnée (FK entrante)

```sql
-- ✅ OK : FK de partition vers table normale
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

CREATE TABLE ventes (
    vente_id BIGSERIAL,
    date_vente DATE NOT NULL,
    client_id INTEGER REFERENCES clients(client_id),  -- OK
    montant NUMERIC(10,2)
) PARTITION BY RANGE (date_vente);

-- ❌ Problématique : FK vers table partitionnée
CREATE TABLE paiements (
    paiement_id SERIAL PRIMARY KEY,
    vente_id BIGINT REFERENCES ventes(vente_id)  -- Difficile
);
```

**Workaround :** Partitionner les deux tables de la même manière.

### 3. Triggers BEFORE ROW

Les triggers BEFORE ROW sur la table parent peuvent modifier la ligne, ce qui peut **changer la partition cible**. Cela fonctionne, mais peut être surprenant.

### 4. Partition DEFAULT

Vous pouvez créer une partition DEFAULT pour capturer les valeurs qui ne correspondent à aucune partition :

```sql
CREATE TABLE ventes_autres PARTITION OF ventes DEFAULT;
```

**Usage :** Utile en développement, à éviter en production (peut devenir un goulot d'étranglement).

---

## Performance : À Quoi S'Attendre ?

### Gains Typiques

**Requêtes avec filtre sur colonne de partitionnement :**
```
Sans partitionnement : 5000 ms (scan de 100M lignes)  
Avec partitionnement : 500 ms (scan de 10M lignes dans 1 partition)  
Gain : 10× plus rapide  
```

**Maintenance :**
```
VACUUM sur table de 100 GB : 2 heures  
VACUUM sur partition de 10 GB : 12 minutes × 10 = 2 heures au total  
Avantage : Peut être fait partition par partition sans tout bloquer  
```

**Archivage :**
```
DELETE FROM ventes WHERE date < '2020-01-01' : 30 minutes + bloat  
DROP TABLE ventes_2019 : < 1 seconde, pas de bloat  
Gain : Quasi-instantané  
```

### Overhead

**Plan-time overhead :** Le planificateur doit examiner les métadonnées de toutes les partitions.

```
Nombre de partitions | Overhead planification
---------------------|-------------------------
10                   | Négligeable (<1ms)
100                  | Faible (~5-10ms)
500                  | Moyen (~20-50ms)
1000+                | Significatif (>100ms)
```

**Recommandation :** Garder le nombre de partitions < 500 si possible.

---

## Outils et Monitoring

### Lister les Partitions

```sql
-- Vue des partitions
SELECT
    nmsp_parent.nspname AS schema,
    parent.relname AS table_parent,
    child.relname AS partition,
    pg_get_expr(child.relpartbound, child.oid) AS limites
FROM pg_inherits  
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid  
JOIN pg_class child ON pg_inherits.inhrelid = child.oid  
JOIN pg_namespace nmsp_parent ON parent.relnamespace = nmsp_parent.oid  
WHERE parent.relname = 'ventes'  
ORDER BY limites;  
```

### Taille des Partitions

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS taille,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS taille_table,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename)) AS taille_index
FROM pg_tables  
WHERE tablename LIKE 'ventes_%'  
ORDER BY tablename;  
```

### Statistiques d'Utilisation

```sql
SELECT
    schemaname,
    tablename,
    n_live_tup AS lignes_actives,
    n_dead_tup AS lignes_mortes,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables  
WHERE tablename LIKE 'ventes_%'  
ORDER BY tablename;  
```

---

## Bonnes Pratiques Générales

### 1. Planification

- ✅ Choisir la bonne stratégie (RANGE pour dates, LIST pour catégories, HASH pour distribution)  
- ✅ Définir une granularité appropriée (mensuelle > quotidienne pour la plupart des cas)  
- ✅ Créer les partitions à l'avance (3-6 mois dans le futur)  
- ✅ Définir une politique de rétention claire

### 2. Création

- ✅ Utiliser des noms cohérents (ex: `table_2024_01`, `table_2024_02`)  
- ✅ Créer les index sur la table parent (propagation automatique)  
- ✅ Documenter la stratégie de partitionnement  
- ✅ Tester avec des données réalistes

### 3. Maintenance

- ✅ Automatiser la création de nouvelles partitions  
- ✅ Automatiser l'archivage/suppression des anciennes partitions  
- ✅ ANALYZE régulièrement (surtout les partitions actives)  
- ✅ Surveiller la taille et l'utilisation des partitions

### 4. Requêtes

- ✅ Toujours filtrer sur la colonne de partitionnement quand possible  
- ✅ Utiliser EXPLAIN pour vérifier le partition pruning  
- ✅ Éviter les fonctions sur la colonne de partitionnement dans WHERE

### 5. Monitoring

- ✅ Alerter si une partition dépasse une taille cible  
- ✅ Monitorer l'équilibre entre partitions  
- ✅ Vérifier régulièrement les statistiques

---

## Checklist : Dois-je Partitionner ?

Avant de décider de partitionner, répondez à ces questions :

### Questions Techniques

- [ ] Ma table dépasse (ou dépassera bientôt) **10 GB** ?  
- [ ] Mes requêtes scannent souvent **> 100 000 lignes** ?  
- [ ] Les index actuels sont **> 1 GB** ?  
- [ ] VACUUM/ANALYZE prend **> 30 minutes** ?

### Questions Métier

- [ ] Les données ont une **dimension temporelle** claire ?  
- [ ] Je supprime régulièrement des **données anciennes** ?  
- [ ] Les requêtes ciblent généralement un **sous-ensemble** (ex: dernier mois) ?  
- [ ] J'ai besoin d'**isoler** certaines données (hot/cold, régions) ?

### Questions d'Infrastructure

- [ ] Mon équipe peut gérer la **complexité supplémentaire** ?  
- [ ] J'ai les **ressources** pour automatiser la gestion ?  
- [ ] Le gain de performance justifie l'**effort de mise en place** ?

**Résultat :**
- **6+ réponses "oui"** → Partitionnement fortement recommandé  
- **3-5 réponses "oui"** → Envisager sérieusement  
- **< 3 réponses "oui"** → Probablement pas nécessaire pour l'instant

---

## Structure du Chapitre 11.4

Ce chapitre est organisé en plusieurs sections :

**11.4.1. Stratégies de partitionnement**
- RANGE : Partitionnement par plage (dates, séquences)
- LIST : Partitionnement par liste (catégories, régions)
- HASH : Partitionnement par hash (distribution uniforme)
- Sous-partitionnement (multi-niveaux)

**11.4.2. Partition Pruning et Partition-wise Join**
- Comment PostgreSQL élimine les partitions inutiles
- Optimisations de jointure entre tables partitionnées
- Configuration et monitoring

**11.4.3. Détachement et attachement de partitions**
- Gérer dynamiquement les partitions
- Archivage et purge rapide
- Maintenance sans interruption de service

---

## Conclusion de l'Introduction

Le **partitionnement de tables** est une fonctionnalité puissante de PostgreSQL qui permet de gérer efficacement des tables volumineuses. Les points clés à retenir :

### Points Essentiels

1. **Le partitionnement divise une grande table en partitions plus petites**
   - Table parent virtuelle
   - Partitions physiques contenant les données
   - Transparence pour l'application

2. **Trois stratégies principales**
   - RANGE : Pour dates et séquences (le plus courant)
   - LIST : Pour catégories et valeurs discrètes
   - HASH : Pour distribution uniforme

3. **Bénéfices majeurs**
   - Performance : Partition pruning élimine les partitions inutiles
   - Maintenance : Opérations plus rapides et moins impactantes
   - Gestion du cycle de vie : Archivage/purge facilités

4. **Quand l'utiliser**
   - Tables > 10-100 GB
   - Dimension temporelle claire
   - Besoin d'archivage régulier
   - Maintenance problématique

5. **Limitations**
   - Contraintes PK/UNIQUE doivent inclure colonne de partitionnement
   - FK entrantes complexes
   - Overhead si trop de partitions (> 1000)

### Prochaines Étapes

Dans les sections suivantes, nous explorerons :
- Les détails de chaque stratégie de partitionnement
- Les optimisations automatiques de PostgreSQL
- La gestion dynamique des partitions

Le partitionnement n'est pas une solution miracle, mais quand il est bien utilisé, il transforme radicalement les performances et la maintenabilité de grandes bases de données.

---

**Fin de l'Introduction au Chapitre 11.4**

Vous êtes maintenant prêt à explorer les stratégies de partitionnement en détail dans la section 11.4.1.

⏭️ [Stratégies de partitionnement](/11-modelisation-avancee/04.1-strategies-partitionnement.md)
