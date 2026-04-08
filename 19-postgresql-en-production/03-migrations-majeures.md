🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.3. Migrations majeures

## Introduction

La migration d'une base de données PostgreSQL d'une version majeure à une autre (par exemple de PostgreSQL 17 vers PostgreSQL 18) est une opération critique qui nécessite une **préparation minutieuse** et une **exécution rigoureuse**. Cette section vous guide à travers les concepts, stratégies et meilleures pratiques pour réussir vos migrations majeures vers PostgreSQL 18.

## Qu'est-ce qu'une migration majeure ?

### Comprendre le versionnage PostgreSQL

PostgreSQL utilise un système de versionnage structuré qui permet de distinguer les différents types de mises à jour.

#### Structure des versions

Depuis PostgreSQL 10, le système de versionnage suit ce format :

```
PostgreSQL [VERSION_MAJEURE].[VERSION_MINEURE]

Exemples :
- PostgreSQL 17.0, 17.1, 17.2 → Version majeure 17
- PostgreSQL 18.0, 18.1, 18.2 → Version majeure 18
```

**Avant PostgreSQL 10**, le format était différent :
```
PostgreSQL [MAJEURE].[MAJEURE].[MINEURE]

Exemples :
- PostgreSQL 9.6.1, 9.6.2 → Version majeure 9.6
- PostgreSQL 9.5.10 → Version majeure 9.5
```

#### Versions majeures vs versions mineures

```
┌────────────────────────────────────────────────────────────┐
│  Versions Majeures vs Mineures                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  VERSION MAJEURE (ex: 17 → 18)                             │
│  ────────────────────────────────                          │
│  • Nouvelles fonctionnalités importantes                   │
│  • Changements de format de stockage interne               │
│  • Améliorations de performance majeures                   │
│  • Possibles incompatibilités avec version précédente      │
│  • Migration complexe (pg_upgrade requis)                  │
│  • Fréquence : 1 fois par an (septembre/octobre)           │
│                                                            │
│  Exemples :                                                │
│  17.x → 18.x ✅ (Migration majeure)                        │
│  16.x → 18.x ✅ (Migration majeure)                        │
│                                                            │
│  ───────────────────────────────────────────────────────── │
│                                                            │
│  VERSION MINEURE (ex: 18.0 → 18.1)                         │
│  ──────────────────────────────                            │
│  • Corrections de bugs                                     │
│  • Patches de sécurité                                     │
│  • Corrections de régressions                              │
│  • Pas de nouvelles fonctionnalités                        │
│  • Compatible avec même version majeure                    │
│  • Migration simple (arrêt/mise à jour/redémarrage)        │
│  • Fréquence : Trimestrielle (environ tous les 3 mois)     │
│                                                            │
│  Exemples :                                                │
│  18.0 → 18.1 ✅ (Mise à jour mineure)                      │
│  18.1 → 18.2 ✅ (Mise à jour mineure)                      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Pourquoi les migrations majeures sont différentes

#### Format de stockage interne

PostgreSQL stocke ses données dans des **fichiers binaires** dont le format évolue entre versions majeures :

```
┌─────────────────────────────────────────────────────────────┐
│  Format de stockage : Incompatibilité entre versions        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PostgreSQL 17                  PostgreSQL 18               │
│  ──────────────                 ──────────────              │
│                                                             │
│  Fichier data/base/16384/12345  Fichier data/base/16384/... │
│  ┌─────────────────┐            ┌─────────────────┐         │
│  │ Format v17      │            │ Format v18      │         │
│  │ [Binary data]   │ ❌ ≠ ❌    │ [Binary data]   │         │
│  │ Structure A     │            │ Structure B     │         │
│  └─────────────────┘            └─────────────────┘         │
│                                                             │
│  ❌ PostgreSQL 18 ne peut PAS lire directement              │
│     les fichiers de PostgreSQL 17                           │
│                                                             │
│  → Nécessite une MIGRATION                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Conséquence** : Vous ne pouvez pas simplement copier les fichiers de données d'une version à l'autre.

#### Comparaison avec d'autres systèmes

**Analogie** : Formats de fichiers incompatibles

```
Microsoft Word :
- Document .doc (Word 2003)
- Document .docx (Word 2010+)
→ Formats différents, conversion nécessaire

PostgreSQL :
- Data files PG 17
- Data files PG 18
→ Formats différents, migration nécessaire
```

### Types de migrations

```
┌─────────────────────────────────────────────────────────────┐
│  Les 3 approches principales de migration                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1️⃣  DUMP/RESTORE (Logique)                                 │
│     ────────────────────────                                │
│     PG 17 → pg_dump → Fichier SQL → pg_restore → PG 18      │
│                                                             │
│     • Simple conceptuellement                               │
│     • Lent (heures à jours pour grosses bases)              │
│     • Réorganise les données (élimine le bloat)             │
│     • Downtime important                                    │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  2️⃣  PG_UPGRADE (In-place)                                  │
│     ─────────────────────────                               │
│     PG 17 → pg_upgrade → PG 18                              │
│                                                             │
│     • Rapide (minutes à heures)                             │
│     • Réutilise les fichiers existants                      │
│     • Downtime modéré                                       │
│     • Méthode recommandée                                   │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  3️⃣  RÉPLICATION LOGIQUE (Quasi-zéro downtime)              │
│     ────────────────────────────────────────                │
│     PG 17 ←─ replication logique ─→ PG 18                   │
│            (synchronisation continue)                       │
│                                                             │
│     • Downtime minimal (minutes)                            │
│     • Complexe à mettre en œuvre                            │
│     • Idéal pour systèmes critiques                         │
│     • Coût infrastructure plus élevé                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Pourquoi migrer vers PostgreSQL 18 ?

### Nouveautés et améliorations majeures

PostgreSQL 18 apporte des améliorations significatives qui justifient la migration :

#### 1. Performances I/O (Entrées/Sorties)

```
┌─────────────────────────────────────────────────────────────┐
│  I/O asynchrone (AIO) - Nouveau sous-système                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PostgreSQL 17 :                                            │
│  ───────────────                                            │
│  I/O synchrone → Les opérations attendent une par une       │
│                                                             │
│  Requête 1 → [Lecture disque] → Attente → Résultat          │
│  Requête 2 →          [Attendre requête 1]                  │
│  Requête 3 →                  [Attendre requête 2]          │
│                                                             │
│  PostgreSQL 18 :                                            │
│  ───────────────                                            │
│  I/O asynchrone → Plusieurs opérations en parallèle         │
│                                                             │
│  Requête 1 → [Lecture disque]                               │
│  Requête 2 → [Lecture disque]  ← En même temps !            │
│  Requête 3 → [Lecture disque]                               │
│                                                             │
│  Gain de performance : Jusqu'à 3× plus rapide               │
│  sur les opérations I/O intensives                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Nouvelles fonctionnalités SQL

```sql
-- UUIDv7 natif (avec timestamp intégré)
SELECT uuidv7();
-- b3d9f3e0-7a2c-7890-8000-1234567890ab
-- ↑ Contient un timestamp, trié chronologiquement

-- Colonnes générées virtuelles
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    price NUMERIC(10,2),
    tax_rate NUMERIC(4,2),
    price_with_tax NUMERIC(10,2) GENERATED ALWAYS AS (price * (1 + tax_rate)) VIRTUAL
    -- ↑ Calculée à la volée, pas stockée physiquement
);

-- Contraintes temporelles
ALTER TABLE employees  
ADD CONSTRAINT valid_employment_period  
CHECK (end_date IS NULL OR end_date > start_date);  
```

#### 3. Améliorations de sécurité

```
✅ OAuth 2.0 natif : Authentification moderne
✅ SCRAM passthrough : Amélioration sécurité réplication
✅ Mode FIPS : Conformité gouvernementale
✅ TLS 1.3 par défaut : Chiffrement renforcé
✅ Data Checksums activés par défaut : Détection corruption
```

#### 4. Améliorations d'administration

```
✅ pg_upgrade amélioré : Préservation des statistiques
✅ Option --swap : Rollback facilité
✅ Vérifications parallèles (--jobs) : Migration plus rapide
✅ Autovacuum dynamique : Ajustements automatiques
✅ Statistiques I/O enrichies : Meilleur monitoring
```

#### 5. Optimisations du planificateur

```sql
-- Skip Scan sur index multi-colonnes
-- Avant PG 18 : Obligé de scanner tout l'index
-- PG 18 : Saute intelligemment les valeurs inutiles

CREATE INDEX idx_orders ON orders(status, created_at);

-- Requête optimisée en PG 18
SELECT * FROM orders  
WHERE created_at > '2024-01-01'  
-- PG 18 "saute" les valeurs de status inutiles

-- Auto-élimination des self-joins inutiles
-- Transformation OR en ANY (plus efficace)
-- Optimisation IN (VALUES ...) vers ANY
```

### Support et cycle de vie

```
┌─────────────────────────────────────────────────────────────┐
│  Politique de support PostgreSQL                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Chaque version majeure est supportée pendant 5 ans         │
│                                                             │
│  Version    Release        Fin de support                   │
│  ────────   ─────────────  ──────────────────               │
│  PG 13      Sept 2020      Novembre 2025                    │
│  PG 14      Sept 2021      Novembre 2026                    │
│  PG 15      Sept 2022      Novembre 2027                    │
│  PG 16      Sept 2023      Novembre 2028                    │
│  PG 17      Sept 2024      Novembre 2029                    │
│  PG 18      Sept 2025      Novembre 2030 (estimé)           │
│                                                             │
│  ⚠️  Après la fin de support :                              │
│  • Plus de patches de sécurité                              │
│  • Plus de corrections de bugs                              │
│  • Risques de sécurité croissants                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Conseil** : Planifier les migrations **avant** la fin de support de votre version actuelle.

## Les défis des migrations majeures

### Risques potentiels

```
┌─────────────────────────────────────────────────────────────┐
│  Risques à gérer lors d'une migration majeure               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  💣 Downtime (indisponibilité)                              │
│     • Service arrêté pendant la migration                   │
│     • Perte de revenus potentielle                          │
│     • Impact utilisateurs                                   │
│                                                             │
│  💣 Incompatibilités applicatives                           │
│     • Fonctions SQL dépréciées                              │
│     • Changements de comportement                           │
│     • Extensions incompatibles                              │
│                                                             │
│  💣 Dégradation de performances                             │
│     • Plans d'exécution différents                          │
│     • Configuration inadaptée                               │
│     • Statistiques manquantes (PG < 18)                     │
│                                                             │
│  💣 Perte de données                                        │
│     • Erreur pendant la migration                           │
│     • Corruption de fichiers                                │
│     • Rollback mal géré                                     │
│                                                             │
│  💣 Problèmes opérationnels                                 │
│     • Équipe non préparée                                   │
│     • Procédures non testées                                │
│     • Monitoring inadapté                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Complexité selon la taille

```
┌─────────────────────────────────────────────────────────────┐
│  Complexité de migration selon la taille                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Petite base (< 10 GB)                                      │
│  🟢 Complexité : Faible                                     │
│  ⏱️  Downtime : 15-30 minutes                               │
│  👥 Équipe : 1-2 personnes                                  │
│  📋 Méthode : pg_dump/restore OU pg_upgrade                 │
│                                                             │
│  Moyenne base (10-100 GB)                                   │
│  🟡 Complexité : Moyenne                                    │
│  ⏱️  Downtime : 30 min - 2 heures                           │
│  👥 Équipe : 2-3 personnes                                  │
│  📋 Méthode : pg_upgrade recommandé                         │
│                                                             │
│  Grosse base (100 GB - 1 TB)                                │
│  🟠 Complexité : Élevée                                     │
│  ⏱️  Downtime : 1-4 heures                                  │
│  👥 Équipe : 3-5 personnes                                  │
│  📋 Méthode : pg_upgrade --swap --jobs                      │
│                                                             │
│  Très grosse base (> 1 TB)                                  │
│  🔴 Complexité : Très élevée                                │
│  ⏱️  Downtime : 4+ heures (sauf réplication logique)        │
│  👥 Équipe : 5+ personnes (DBA, DevOps, Dev)                │
│  📋 Méthode : Réplication logique OU pg_upgrade optimisé    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Stratégies de migration disponibles

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────┐
│  Comparaison des stratégies de migration                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Critère          pg_dump/    pg_upgrade   Replication      │
│                   restore                  Logique          │
│  ──────────────   ─────────   ──────────   ──────────       │
│  Complexité       🟢 Simple   🟡 Moyenne   🔴 Complexe      │
│  Downtime         🔴 Long     🟡 Moyen     🟢 Minimal       │
│  Vitesse          🔴 Lent     🟢 Rapide    🟡 Setup long    │
│  Espace disque    🔴 2×       🟡 1.5×      🔴 2×            │
│  Rollback         🟡 Facile   🟢 Facile    🟢 Très facile   │
│  Réorganisation   ✅ Oui      ❌ Non       ❌ Non           │
│  Tests avant      🟡 Limités  🟢 Oui       🟢 Oui           │
│  Production       ⚠️          ✅          ✅✅              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1. pg_dump/restore : La méthode logique

**Principe** : Exporter les données au format SQL, puis les réimporter.

```
┌─────────────────────────────────────────────────────────────┐
│  Processus pg_dump/restore                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Étape 1 : Export (pg_dump)                                 │
│  ──────────────────────────                                 │
│  PostgreSQL 17 → pg_dump → fichier.sql (ou .dump)           │
│                                                             │
│  CREATE TABLE users (...);                                  │
│  INSERT INTO users VALUES (...);                            │
│  INSERT INTO users VALUES (...);                            │
│  ...                                                        │
│                                                             │
│  Étape 2 : Installation PG 18                               │
│  ──────────────────────────                                 │
│  Installation PostgreSQL 18                                 │
│  Initialisation cluster vide                                │
│                                                             │
│  Étape 3 : Import (pg_restore)                              │
│  ──────────────────────────                                 │
│  fichier.sql → psql → PostgreSQL 18                         │
│                                                             │
│  Exécution des CREATE TABLE                                 │
│  Exécution des INSERT                                       │
│  Création des index                                         │
│  Ajout des contraintes                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Simple à comprendre et à mettre en œuvre  
- ✅ Réorganise les données (élimine le bloat)  
- ✅ Permet de filtrer ou modifier pendant l'import  
- ✅ Indépendant des versions (fonctionne toujours)

**Inconvénients** :
- ❌ Très lent sur grosses bases (jours pour plusieurs To)  
- ❌ Downtime important  
- ❌ Nécessite 2× l'espace disque  
- ❌ Perte des statistiques (ANALYZE requis après)

**Quand l'utiliser** :
- Bases < 50 GB
- Downtime flexible
- Besoin de réorganisation des données
- Migration très ancienne (PG 9.x → PG 18)

### 2. pg_upgrade : La méthode in-place

**Principe** : Migrer la base en réutilisant les fichiers de données existants.

```
┌─────────────────────────────────────────────────────────────┐
│  Processus pg_upgrade                                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Avant :                                                    │
│  ───────                                                    │
│  /var/lib/postgresql/17/main ← Cluster actif (PG 17)        │
│                                                             │
│  Pendant pg_upgrade :                                       │
│  ────────────────────                                       │
│  1. Vérification de compatibilité                           │
│  2. Création structure PG 18                                │
│  3. Copie/Link des fichiers de données                      │
│  4. Mise à jour des métadonnées                             │
│  5. (PG 18) Préservation des statistiques                   │
│                                                             │
│  Après :                                                    │
│  ───────                                                    │
│  /var/lib/postgresql/18/main ← Nouveau cluster actif        │
│  /var/lib/postgresql/17/main ← Ancien cluster (à garder)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Rapide (minutes à quelques heures)  
- ✅ Downtime réduit  
- ✅ Statistiques préservées (PG 18)  
- ✅ Option --swap pour rollback facile (PG 18)  
- ✅ Parallélisation avec --jobs (PG 18)

**Inconvénients** :
- ❌ Ne réorganise pas les données (bloat conservé)  
- ❌ Nécessite espace disque supplémentaire (~50%)  
- ❌ Plus complexe que dump/restore  
- ❌ Teste la migration à l'aveugle (sauf --check)

**Quand l'utiliser** :
- Bases moyennes à grosses (> 50 GB)
- Downtime limité (< 4 heures)
- Production standard
- **Méthode recommandée dans la plupart des cas**

### 3. Réplication logique : La méthode haute disponibilité

**Principe** : Synchroniser continuellement deux clusters pendant que l'ancien reste en production.

```
┌─────────────────────────────────────────────────────────────┐
│  Processus Réplication Logique                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Phase 1 : Setup (J-7)                                      │
│  ─────────────────────                                      │
│  PG 17 (Production) ──replication──→ PG 18 (Standby)        │
│         ↑                                                   │
│    Applications                                             │
│                                                             │
│  Phase 2 : Synchronisation (J-7 à J-Day)                    │
│  ────────────────────────────────────────                   │
│  Les deux clusters restent synchronisés                     │
│  Tests possibles sur PG 18 sans impact sur production       │
│                                                             │
│  Phase 3 : Cutover (J-Day, 5-15 min)                        │
│  ─────────────────────────────────────                      │
│  PG 17 (Standby)         PG 18 (Production)                 │
│                                 ↑                           │
│                            Applications                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Downtime minimal (5-30 minutes)  
- ✅ Tests possibles avant bascule  
- ✅ Rollback instantané  
- ✅ Idéal pour systèmes critiques

**Inconvénients** :
- ❌ Complexe à mettre en œuvre  
- ❌ Nécessite 2 serveurs en parallèle  
- ❌ Coût infrastructure élevé  
- ❌ Limitations (séquences, DDL, etc.)

**Quand l'utiliser** :
- Services critiques 24/7
- Downtime < 30 minutes impératif
- Budget infrastructure disponible
- Équipe avec expertise réplication

## Préparation d'une migration majeure

### Planification générale

```
┌─────────────────────────────────────────────────────────────┐
│  Timeline type d'une migration majeure                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  J-60  📋 Planification                                     │
│        • Choix de la stratégie                              │
│        • Constitution de l'équipe                           │
│        • Identification des risques                         │
│                                                             │
│  J-45  📚 Documentation                                     │
│        • État des lieux (versions, extensions, taille)      │
│        • Runbook de migration                               │
│        • Plan de rollback                                   │
│                                                             │
│  J-30  🧪 Tests en DEV                                      │
│        • Migration test                                     │
│        • Vérification compatibilité                         │
│        • Tests applications                                 │
│                                                             │
│  J-14  🔬 Tests en STAGING                                  │
│        • Clone production                                   │
│        • Migration complète                                 │
│        • Tests de charge                                    │
│        • Validation performances                            │
│                                                             │
│  J-7   ✅ Validation finale                                 │
│        • Go/No-Go meeting                                   │
│        • Communication utilisateurs                         │
│        • Préparation équipe                                 │
│                                                             │
│  J-Day 🚀 Migration PRODUCTION                              │
│        • Fenêtre de maintenance                             │
│        • Exécution procédures                               │
│        • Validation post-migration                          │
│                                                             │
│  J+1   📊 Monitoring intensif                               │
│        • Surveillance 24/7                                  │
│        • Validation performances                            │
│        • Support utilisateurs                               │
│                                                             │
│  J+7   🎯 Stabilisation                                     │
│        • Analyse métriques                                  │
│        • Optimisations si besoin                            │
│        • Documentation retour d'expérience                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Checklist de préparation

```
□ ANALYSE
  □ Version actuelle identifiée
  □ Taille de la base mesurée
  □ Extensions listées et compatibilité vérifiée
  □ Contraintes de downtime définies
  □ Budget alloué

□ ÉQUIPE
  □ Chef de projet désigné
  □ DBA / Expert PostgreSQL disponible
  □ DevOps / Infrastructure préparés
  □ Développeurs informés
  □ Support utilisateurs prêt

□ ENVIRONNEMENTS
  □ DEV avec PostgreSQL 18 disponible
  □ STAGING avec clone production
  □ Production sauvegardée

□ STRATÉGIE
  □ Méthode de migration choisie
  □ Date et fenêtre de maintenance fixées
  □ Plan de communication rédigé
  □ Procédures de rollback documentées

□ TESTS
  □ Migration test en DEV réussie
  □ Migration test en STAGING réussie
  □ Applications testées sur PG 18
  □ Performances validées
  □ Rollback testé

□ DOCUMENTATION
  □ Runbook de migration rédigé
  □ Scripts préparés et testés
  □ Procédures de validation définies
  □ Contacts d'urgence listés
```

### État des lieux préalable

Avant toute migration, documenter l'état actuel :

```sql
-- Script d'état des lieux
-- state_assessment.sql

-- 1. Version actuelle
SELECT version();

-- 2. Taille de la base
SELECT
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database  
ORDER BY pg_database_size(pg_database.datname) DESC;  

-- 3. Nombre d'objets
SELECT
    'Tables' AS object_type,
    count(*) AS count
FROM pg_tables  
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')  
UNION ALL  
SELECT 'Index', count(*) FROM pg_indexes  
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')  
UNION ALL  
SELECT 'Sequences', count(*) FROM pg_sequences  
UNION ALL  
SELECT 'Functions', count(*) FROM pg_proc  
WHERE pronamespace::regnamespace::text NOT IN ('pg_catalog', 'information_schema');  

-- 4. Extensions installées
SELECT
    extname AS extension,
    extversion AS version
FROM pg_extension  
ORDER BY extname;  

-- 5. Configuration importante
SELECT
    name,
    setting,
    unit
FROM pg_settings  
WHERE name IN (  
    'max_connections',
    'shared_buffers',
    'work_mem',
    'maintenance_work_mem',
    'wal_level',
    'max_wal_senders',
    'max_replication_slots'
)
ORDER BY name;

-- 6. Activité actuelle
SELECT
    count(*) AS active_connections,
    count(*) FILTER (WHERE state = 'active') AS active_queries
FROM pg_stat_activity;
```

### Sauvegardes critiques

**AVANT toute migration, effectuer une sauvegarde complète** :

```bash
#!/bin/bash
# pre_migration_backup.sh

DATE=$(date +%Y%m%d_%H%M%S)  
BACKUP_DIR="/backup/pre_migration_pg18_${DATE}"  

echo "🔄 Sauvegarde pré-migration vers PostgreSQL 18"  
echo "Destination : $BACKUP_DIR"  

# 1. Sauvegarde physique complète (pg_basebackup)
echo "📦 Sauvegarde physique en cours..."  
pg_basebackup -D "$BACKUP_DIR/physical" -Fp -Xs -P -v  

# 2. Sauvegarde logique (pg_dumpall)
echo "📄 Sauvegarde logique en cours..."  
pg_dumpall > "$BACKUP_DIR/logical/dumpall_${DATE}.sql"  

# 3. Sauvegarde configuration
echo "⚙️  Sauvegarde configuration..."  
mkdir -p "$BACKUP_DIR/config"  
cp /etc/postgresql/17/main/postgresql.conf "$BACKUP_DIR/config/"  
cp /etc/postgresql/17/main/pg_hba.conf "$BACKUP_DIR/config/"  

# 4. Documentation état
echo "📊 Documentation état système..."  
psql -f state_assessment.sql > "$BACKUP_DIR/state_before_migration.txt"  

# 5. Vérification intégrité
echo "✅ Vérification intégrité sauvegarde..."  
du -sh "$BACKUP_DIR"  

echo "✅ Sauvegarde terminée : $BACKUP_DIR"  
echo "⚠️  Conserver cette sauvegarde jusqu'à validation complète de la migration"  
```

## Les nouveautés PostgreSQL 18 pour les migrations

Cette introduction ne détaille pas toutes les nouveautés (voir sections suivantes), mais voici un aperçu des améliorations qui facilitent les migrations :

### 1. pg_upgrade amélioré (Section 19.3.1)

```
✅ Préservation des statistiques
   → Plus besoin de ANALYZE après migration
   → Performances immédiates
   → Économie de temps (heures sur grosses bases)
```

### 2. Option --swap (Section 19.3.2)

```
✅ Rollback facilité
   → Retour arrière simple et rapide
   → Sécurité accrue
   → Confiance pour production
```

### 3. Vérifications parallèles --jobs (Section 19.3.3)

```
✅ Phase --check parallélisée
   → Validation 5-10× plus rapide
   → Tests itératifs possibles
   → Gain de temps en préparation
```

### 4. Stratégies avancées (Section 19.3.4)

```
✅ Blue/Green + Logical Replication
   → Downtime minimal (< 30 min)
   → Tests avant bascule
   → Idéal pour production critique
```

### 5. Tests et validation (Section 19.3.5)

```
✅ Méthodologies complètes
   → Tests automatisés
   → Validation progressive
   → Confiance maximale
```

## Comparaison des méthodes : Quel outil pour quelle situation ?

```
┌─────────────────────────────────────────────────────────────┐
│  Arbre de décision : Quelle méthode choisir ?               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│             Besoin de migrer PG 17 → PG 18                  │
│                         |                                   │
│                         |                                   │
│              Quelle taille de base ?                        │
│           /              |              \                   │
│        < 10 GB       10-100 GB        > 100 GB              │
│          |               |               |                  │
│     pg_dump/restore      |               |                  │
│     OU pg_upgrade        |               |                  │
│          |               |               |                  │
│          |        Quel downtime          |                  │
│          |        acceptable ?           |                  │
│          |      /              \         |                  │
│          |   < 1h             > 1h       |                  │
│          |    |                |         |                  │
│          | pg_upgrade      pg_dump       |                  │
│          | --swap          (réorg)       |                  │
│          |                               |                  │
│          |                     Downtime acceptable ?        │
│          |                    /                    \        │
│          |                < 30 min              > 1h        │
│          |                   |                    |         │
│          |             Réplication          pg_upgrade      │
│          |             Logique              --swap          │
│          |             (Blue/Green)         --jobs          │
│          |                                                  │
└─────────────────────────────────────────────────────────────┘
```

### Recommandations par profil

```
┌─────────────────────────────────────────────────────────────┐
│  Recommandations selon le profil d'entreprise               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  🏢 Startup / PME                                           │
│  Base : < 500 GB                                            │
│  Budget : Limité                                            │
│  Équipe : 1-3 personnes                                     │
│  → pg_upgrade --swap --jobs                                 │
│                                                             │
│  🏢 Entreprise moyenne                                      │
│  Base : 500 GB - 2 TB                                       │
│  Budget : Moyen                                             │
│  Équipe : 3-10 personnes                                    │
│  → pg_upgrade --swap --jobs                                 │
│     OU Réplication logique (si < 30 min requis)             │
│                                                             │
│  🏢 Grande entreprise / FinTech                             │
│  Base : > 2 TB                                              │
│  Budget : Confortable                                       │
│  Équipe : 10+ personnes                                     │
│  Criticité : Maximale                                       │
│  → Réplication logique (Blue/Green)                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Structure de cette section

Cette section 19.3 est organisée en plusieurs sous-sections détaillées :

### 19.3.1. Nouveauté PG 18 : pg_upgrade amélioré et préservation des statistiques
- Fonctionnement de pg_upgrade
- Préservation automatique des statistiques
- Impact sur les performances post-migration
- Comparaison PG 17 vs PG 18

### 19.3.2. Nouveauté PG 18 : Option --swap pour upgrade rapide
- Concept de swap de répertoires
- Comparaison avec --copy et --link
- Procédure de rollback facilitée
- Cas d'usage et recommandations

### 19.3.3. Nouveauté PG 18 : Vérifications parallèles (--jobs)
- Parallélisation de la phase --check
- Optimisation du nombre de workers
- Gains de performance mesurés
- Configuration selon le matériel

### 19.3.4. Stratégies Blue/Green et Logical Replication
- Déploiement Blue/Green
- Réplication logique pour downtime minimal
- Configuration et mise en œuvre
- Cas d'usage pour production critique

### 19.3.5. Tests de migration et validation
- Environnements de test (DEV, STAGING, PROD)
- Tests pré-migration, pendant et post-migration
- Outils de test et automatisation
- Méthodologies de validation

## Points clés à retenir

```
✅ Une migration majeure nécessite une préparation rigoureuse
✅ PostgreSQL 18 facilite grandement les migrations
✅ Plusieurs stratégies existent, choisir selon le contexte
✅ Les tests sont essentiels (DEV → STAGING → PROD)
✅ pg_upgrade est la méthode recommandée dans la plupart des cas
✅ La réplication logique offre le downtime minimal
✅ Toujours avoir un plan de rollback testé
✅ La sauvegarde complète est obligatoire avant migration
```

## Prochaines étapes

Les sections suivantes détaillent chaque aspect de la migration :

1. **Lisez 19.3.1** : Comprenez la préservation des statistiques (gain majeur PG 18)  
2. **Lisez 19.3.2** : Maîtrisez l'option --swap (sécurité et rollback)  
3. **Lisez 19.3.3** : Optimisez avec --jobs (rapidité des vérifications)  
4. **Lisez 19.3.4** : Explorez Blue/Green (downtime minimal)  
5. **Lisez 19.3.5** : Appliquez les méthodologies de test (succès garanti)

---

**Note** : Une migration majeure peut sembler intimidante, mais avec une bonne préparation et les bons outils (notamment les nouveautés de PostgreSQL 18), elle devient un processus maîtrisable. L'essentiel est de ne pas se précipiter, de tester abondamment, et de suivre les procédures éprouvées. Les sections suivantes vous guident pas à pas dans cette démarche.

**Conseil** : La clé d'une migration réussie réside dans trois mots : **Préparation, Tests, Documentation**. Investissez 80% de votre temps dans ces phases, et les 20% restants (la migration elle-même) se dérouleront sans encombre.

⏭️ [Nouveauté PG 18 : pg_upgrade amélioré et préservation des statistiques](/19-postgresql-en-production/03.1-pg-upgrade-ameliore-pg18.md)
