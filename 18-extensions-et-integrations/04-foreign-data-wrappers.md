🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.4. Foreign Data Wrappers (FDW) : PostgreSQL comme Hub de Données

## Introduction

Dans le monde moderne des données, les entreprises utilisent rarement une seule base de données. Vous pourriez avoir :
- Des données clients dans **Oracle**
- Des logs d'application dans des **fichiers CSV**
- Des données produits dans **MySQL**
- Des données analytiques dans **PostgreSQL**

Traditionnellement, pour consolider ces données, vous deviez :
1. Exporter les données depuis chaque source  
2. Transformer les formats (ETL)  
3. Importer dans une base centrale  
4. Gérer la synchronisation

**Les Foreign Data Wrappers (FDW)** offrent une alternative révolutionnaire : transformer PostgreSQL en **hub de données** capable d'interroger directement toutes ces sources comme si elles étaient des tables locales, sans duplication.

### Analogie : PostgreSQL comme traducteur universel

Imaginez que vous êtes dans une conférence internationale :
- Des participants parlent **français** (Oracle)
- D'autres parlent **anglais** (MySQL)
- D'autres parlent **espagnol** (MongoDB)
- Certains communiquent par **documents écrits** (fichiers CSV)

Avec un **traducteur universel** (PostgreSQL + FDW), vous pouvez :
- Poser une question en français
- Le traducteur la transmet dans chaque langue
- Chacun répond dans sa langue
- Le traducteur vous retourne une réponse unifiée en français

PostgreSQL devient ce traducteur universel pour vos données !

---

## Qu'est-ce qu'un Foreign Data Wrapper ?

### Définition

Un **Foreign Data Wrapper (FDW)** est une extension PostgreSQL qui implémente le standard **SQL/MED** (Management of External Data) pour accéder à des sources de données externes.

**En termes simples** : Un FDW est un "adaptateur" qui permet à PostgreSQL de se connecter à une source de données externe et de la traiter comme une table PostgreSQL normale.

### Architecture conceptuelle

```
┌────────────────────────────────────────────────────────────┐
│              Application Utilisateur                       │
│         (Votre code Python, Java, Node.js, etc.)           │
└────────────────────┬───────────────────────────────────────┘
                     │ Connexion standard PostgreSQL
                     │
┌────────────────────▼───────────────────────────────────────┐
│                                                            │
│           PostgreSQL (Hub Central de Données)              │
│                                                            │
│  ┌─────────────┐   ┌─────────────┐  ┌─────────────┐        │
│  │   Tables    │   │   Foreign   │  │   Foreign   │        │
│  │   Locales   │   │   Table 1   │  │   Table 2   │        │
│  │  (natives)  │   │  (virtuelle)│  │  (virtuelle)│        │
│  └─────────────┘   └──────┬──────┘  └──────┬──────┘        │
│                           │                │               │
│                    ┌──────▼────────┐ ┌─────▼──────┐        │
│                    │  postgres_fdw │ │ oracle_fdw │        │
│                    │  (extension)  │ │ (extension)│        │
│                    └──────┬────────┘ └─────┬──────┘        │
└───────────────────────────┼────────────────┼───────────────┘
                            │                │
              ┌─────────────┘                └────────────────┐
              │ Protocole PostgreSQL         Protocole Oracle │
              │                                               │
┌─────────────▼───────────┐           ┌───────────────────────▼─┐
│  PostgreSQL Distant     │           │   Oracle Database       │
│  (Autre serveur)        │           │   (Serveur legacy)      │
│                         │           │                         │
│  ┌─────────────────┐    │           │  ┌────────────────┐     │
│  │  Table Réelle   │    │           │  │  Table Réelle  │     │
│  │  - customers    │    │           │  │  - products    │     │
│  └─────────────────┘    │           │  └────────────────┘     │
└─────────────────────────┘           └─────────────────────────┘
```

### Composants d'un FDW

Pour utiliser un FDW, vous devez configurer trois éléments principaux :

#### 1. Extension FDW

L'extension PostgreSQL qui implémente le protocole de communication avec la source externe.

```sql
-- Activer l'extension
CREATE EXTENSION postgres_fdw;
```

#### 2. Foreign Server

La définition de la connexion vers la source de données externe (adresse, port, paramètres).

```sql
CREATE SERVER serveur_distant
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host 'db.exemple.com',
        port '5432',
        dbname 'production'
    );
```

#### 3. Foreign Table

Une table **virtuelle** dans PostgreSQL qui représente les données distantes. Elle n'existe pas physiquement dans PostgreSQL mais agit comme un "pointeur" vers la table réelle sur la source externe.

```sql
CREATE FOREIGN TABLE clients_distants (
    id INTEGER,
    nom TEXT,
    email TEXT
)
SERVER serveur_distant  
OPTIONS (  
    schema_name 'public',
    table_name 'customers'
);
```

#### 4. User Mapping (optionnel mais courant)

Les identifiants de connexion pour accéder à la source externe.

```sql
CREATE USER MAPPING FOR utilisateur_local
    SERVER serveur_distant
    OPTIONS (
        user 'admin_distant',
        password 'mot_de_passe'
    );
```

### Terminologie

| Terme | Définition | Exemple |
|-------|------------|---------|
| **Foreign Data Wrapper (FDW)** | Extension qui permet la connexion à une source externe | `postgres_fdw`, `oracle_fdw` |
| **Foreign Server** | Configuration de connexion vers une source externe | Serveur Oracle en production |
| **Foreign Table** | Table virtuelle représentant des données distantes | `clients_oracle`, `logs_csv` |
| **User Mapping** | Identifiants pour se connecter à la source externe | Login/password Oracle |
| **Pushdown** | Délégation du traitement vers la source externe | Filtrage WHERE exécuté sur Oracle |
| **Local Table** | Table PostgreSQL native (stockée localement) | Table classique PostgreSQL |

---

## Standard SQL/MED

### Origine et normalisation

Les Foreign Data Wrappers implémentent le standard **SQL/MED** (Management of External Data), défini dans la norme **SQL:2003**.

**SQL/MED** définit :
- Comment décrire des sources de données externes
- Comment créer des tables qui pointent vers ces sources
- Comment exécuter des requêtes sur ces tables
- L'interface standardisée entre le SGBD et les sources externes

**Avantages de la standardisation** :
- ✅ Approche cohérente entre différents SGBD  
- ✅ Portabilité des concepts (même si l'implémentation diffère)  
- ✅ Écosystème riche de FDW développés par la communauté

### Implémentation PostgreSQL

PostgreSQL a introduit le support SQL/MED dans **PostgreSQL 9.1** (2011) et l'a considérablement amélioré depuis :

| Version PostgreSQL | Amélioration FDW |
|-------------------|------------------|
| **9.1** (2011) | Introduction SQL/MED et `file_fdw` |
| **9.3** (2013) | Introduction `postgres_fdw` avec capacités d'écriture |
| **9.6** (2016) | Amélioration des jointures distantes et tri distant |
| **10** (2017) | **Pushdown d'agrégations** (révolution performance) |
| **11** (2018) | Parallélisation des requêtes sur foreign tables |
| **13** (2020) | Amélioration du partitionnement avec FDW |
| **14** (2021) | Bulk insert plus rapide |
| **16** (2023) | Amélioration du pushdown et optimisations |
| **18** (2025) | I/O asynchrone, amélioration des performances |

---

## Écosystème des FDW

### FDW officiels (fournis avec PostgreSQL)

#### file_fdw

**Source** : Fichiers locaux (CSV, texte)

**Cas d'usage** :
- Lire des exports CSV
- Analyser des logs texte
- Validation de données avant import

**Exemple** :
```sql
CREATE FOREIGN TABLE ventes_csv (
    date DATE,
    produit TEXT,
    montant NUMERIC
)
SERVER serveur_fichiers  
OPTIONS (  
    filename '/data/ventes.csv',
    format 'csv',
    header 'true'
);
```

### FDW tiers populaires

#### postgres_fdw

**Source** : Autres serveurs PostgreSQL

**Cas d'usage** :
- Fédération de bases PostgreSQL multiples
- Migration progressive
- Sharding manuel
- Reporting consolidé

**Maintenu par** : Communauté PostgreSQL (qualité production)

#### oracle_fdw

**Source** : Oracle Database

**Cas d'usage** :
- Migration d'Oracle vers PostgreSQL
- Intégration hybride Oracle/PostgreSQL
- Accès aux données Oracle legacy

**Maintenu par** : Laurenz Albe (EnterpriseDB)

**Prérequis** : Oracle Instant Client

#### mysql_fdw

**Source** : MySQL / MariaDB

**Cas d'usage** :
- Migration de MySQL vers PostgreSQL
- Data warehouse multi-SGBD
- Intégration d'applications MySQL existantes

**Maintenu par** : EnterpriseDB

**Prérequis** : Bibliothèques MySQL client

### FDW pour bases NoSQL

#### mongo_fdw

**Source** : MongoDB

**Cas d'usage** :
- Interroger MongoDB avec SQL
- Joindre données MongoDB avec PostgreSQL
- Reporting sur données NoSQL

```sql
-- Exemple conceptuel
SELECT
    u.nom,
    COUNT(c.commande_id) AS nb_commandes
FROM users_postgres u  
INNER JOIN commandes_mongo c ON u.id = c.user_id  
GROUP BY u.nom;  
```

#### redis_fdw

**Source** : Redis (cache key-value)

**Cas d'usage** :
- Interroger le cache Redis avec SQL
- Analyser les données en cache
- Debugging et monitoring

### FDW pour APIs et services

#### multicorn

**Source** : APIs REST, services web, formats personnalisés

**Particularité** : Framework Python pour créer des FDW personnalisés

**Cas d'usage** :
- API REST comme table SQL
- Elasticsearch
- LDAP
- Services cloud (S3, etc.)

**Exemple conceptuel** :
```sql
-- Créer un FDW pour API REST GitHub
CREATE FOREIGN TABLE github_repos (
    name TEXT,
    stars INTEGER,
    language TEXT
)
SERVER github_api  
OPTIONS (endpoint 'https://api.github.com/repos');  
```

#### www_fdw

**Source** : Pages web (scraping)

**Cas d'usage** :
- Extraction de données depuis pages HTML
- Monitoring de sites web
- Agrégation de données publiques

### FDW pour formats de fichiers

#### ogr_fdw (PostGIS)

**Source** : Formats géospatiaux (Shapefile, GeoJSON, KML)

**Cas d'usage** :
- Données SIG (Systèmes d'Information Géographique)
- Cartographie
- Analyse spatiale

#### parquet_fdw

**Source** : Fichiers Parquet (format Big Data)

**Cas d'usage** :
- Interroger des data lakes
- Intégration avec Hadoop/Spark
- Analytics sur données columnar

### Tableau récapitulatif

| FDW | Source | Officiel ? | Complexité | Cas d'usage principal |
|-----|--------|-----------|------------|----------------------|
| **file_fdw** | Fichiers CSV/texte | ✅ Oui | 🟢 Simple | Import, validation données |
| **postgres_fdw** | PostgreSQL | ✅ Oui | 🟢 Simple | Fédération, migration |
| **oracle_fdw** | Oracle | ❌ Tiers | 🟡 Moyen | Migration Oracle |
| **mysql_fdw** | MySQL/MariaDB | ❌ Tiers | 🟡 Moyen | Migration/intégration MySQL |
| **mongo_fdw** | MongoDB | ❌ Tiers | 🟡 Moyen | NoSQL → SQL |
| **redis_fdw** | Redis | ❌ Tiers | 🟡 Moyen | Cache → SQL |
| **multicorn** | APIs, custom | ❌ Tiers | 🔴 Complexe | APIs REST, services custom |
| **parquet_fdw** | Parquet | ❌ Tiers | 🟡 Moyen | Data lakes, Big Data |

---

## Cas d'Usage des FDW

### 1. PostgreSQL comme Hub de Données (Data Hub)

**Scénario** : Entreprise avec multiples sources de données

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│              PostgreSQL Hub Central                      │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Clients  │  │ Produits │  │ Commandes│  │  Logs    │  │
│  │ (Oracle) │  │ (MySQL)  │  │ (local)  │  │  (CSV)   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │             │             │             │        │
│       │             │             │             │        │
└───────┼─────────────┼─────────────┼─────────────┼────────┘
        │             │             │             │
        │ oracle_fdw  │ mysql_fdw   │ native      │ file_fdw
        │             │             │             │
   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐   ┌────▼─────┐
   │ Oracle  │   │  MySQL  │   │   PG    │   │  Fichiers│
   │   DB    │   │   DB    │   │  Tables │   │   CSV    │
   └─────────┘   └─────────┘   └─────────┘   └──────────┘
```

**Avantages** :
- ✅ Point d'entrée unique pour toutes les données  
- ✅ Requêtes SQL standard sur toutes les sources  
- ✅ Pas de duplication de données  
- ✅ Données toujours à jour (temps réel)

**Exemple de requête consolidée** :

```sql
-- Joindre données de 3 sources différentes
SELECT
    c.nom AS client,           -- Oracle
    p.nom_produit,             -- MySQL
    cmd.montant,               -- PostgreSQL local
    l.date_livraison           -- Fichier CSV
FROM clients_oracle c  
INNER JOIN commandes_locales cmd ON c.id = cmd.client_id  
INNER JOIN produits_mysql p ON cmd.produit_id = p.id  
LEFT JOIN livraisons_csv l ON cmd.id = l.commande_id  
WHERE cmd.date_commande >= '2024-01-01';  
```

### 2. Migration Progressive de Bases de Données

**Scénario** : Migration d'Oracle vers PostgreSQL sans interruption

**Phase 1 : État initial**
```
Application → Oracle (100% des données)
```

**Phase 2 : Configuration FDW**
```
Application → PostgreSQL
                  ↓
            oracle_fdw → Oracle (100% des données)
```

**Phase 3 : Migration table par table**
```
Application → PostgreSQL
                  ├─ Tables migrées (20%)
                  └─ oracle_fdw → Oracle (80% restant)
```

**Phase 4 : Finalisation**
```
Application → PostgreSQL (100% des données)
              Archive → Oracle (historique en lecture seule)
```

**Avantages** :
- ✅ Migration sans interruption de service  
- ✅ Retour arrière possible à tout moment  
- ✅ Tests en conditions réelles  
- ✅ Apprentissage progressif

### 3. Data Warehouse et Reporting

**Scénario** : Consolider des données depuis plusieurs sources pour l'analytique

```
┌─────────────────────────────────────────────────────────────┐
│          PostgreSQL Data Warehouse                          │
│                                                             │
│  Requêtes analytiques, dashboards, reporting                │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │ Ventes (MySQL│   │ Stocks (PG)  │   │ CRM (Oracle) │     │
│  │     FDW)     │   │   distant    │   │    FDW       │     │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘     │
└─────────┼──────────────────┼──────────────────┼─────────────┘
          │                  │                  │
    ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼────┐
    │MySQL Prod │      │ PostgreSQL│      │  Oracle  │
    │  (OLTP)   │      │   (OLTP)  │      │  (CRM)   │
    └───────────┘      └───────────┘      └──────────┘
```

**Stratégie** :
1. Connexion via FDW pour accès temps réel  
2. Vues matérialisées pour cache des agrégations  
3. Rafraîchissement planifié (quotidien, horaire)

```sql
-- Vue matérialisée consolidée
CREATE MATERIALIZED VIEW ventes_consolidees AS  
SELECT  
    date_trunc('day', v.date_vente) AS jour,
    p.categorie,
    SUM(v.montant) AS ca,
    COUNT(*) AS nb_ventes
FROM ventes_mysql v  
INNER JOIN produits_locaux p ON v.produit_id = p.id  
GROUP BY jour, p.categorie;  

-- Index pour performance
CREATE INDEX idx_ventes_cons_jour ON ventes_consolidees(jour);

-- Rafraîchir quotidiennement
REFRESH MATERIALIZED VIEW ventes_consolidees;
```

### 4. Architecture Microservices

**Scénario** : Chaque microservice a sa propre base, mais certains ont besoin de lire les données d'autres services

```
┌─────────────────────────────────────────────────────────────┐
│                   Service Facturation                       │
│                   (PostgreSQL)                              │
│                                                             │
│  Données locales : factures, paiements                      │
│                                                             │
│  Foreign tables (lecture seule) :                           │
│  ├─ clients (depuis Service Clients)                        │
│  └─ commandes (depuis Service Commandes)                    │
└─────────────────────────────────────────────────────────────┘
         │                              │
         │ postgres_fdw                 │ postgres_fdw
         │ (lecture seule)              │ (lecture seule)
         │                              │
    ┌────▼─────────┐              ┌─────▼────────┐
    │  Service     │              │   Service    │
    │  Clients     │              │  Commandes   │
    │ (PostgreSQL) │              │ (PostgreSQL) │
    └──────────────┘              └──────────────┘
```

**Principe** :
- Chaque service possède ses données (ownership)
- Accès en lecture aux données d'autres services via FDW
- Évite les appels API synchrones pour les lectures
- Maintient le découplage

### 5. Sharding Manuel

**Scénario** : Partitionner horizontalement une très grande table sur plusieurs serveurs

```
┌─────────────────────────────────────────────────────────────┐
│               PostgreSQL Coordinateur                       │
│                                                             │
│  Vue partitionnée : clients (virtuelle)                     │
│  ├─ clients_shard1 → Serveur 1 (clients A-M)                │
│  └─ clients_shard2 → Serveur 2 (clients N-Z)                │
└─────────────────────────────────────────────────────────────┘
              │                           │
              │ postgres_fdw              │ postgres_fdw
              │                           │
         ┌────▼─────┐               ┌─────▼────┐
         │ Serveur 1│               │ Serveur 2│
         │ Shard A-M│               │ Shard N-Z│
         └──────────┘               └──────────┘
```

```sql
-- Créer les foreign tables pour chaque shard
CREATE FOREIGN TABLE clients_shard1 (
    id INTEGER,
    nom TEXT,
    -- ...
)
SERVER serveur1  
OPTIONS (schema_name 'public', table_name 'clients');  

CREATE FOREIGN TABLE clients_shard2 (
    -- même structure
)
SERVER serveur2  
OPTIONS (schema_name 'public', table_name 'clients');  

-- Créer une vue unifiée
CREATE VIEW clients AS  
SELECT * FROM clients_shard1  
UNION ALL  
SELECT * FROM clients_shard2;  

-- L'application interroge simplement 'clients'
SELECT * FROM clients WHERE nom = 'Dupont';
```

### 6. Intégration de Fichiers Externes

**Scénario** : Fichiers CSV quotidiens de partenaires

```sql
-- Fichier du jour
CREATE FOREIGN TABLE commandes_partenaire_aujourdhui (
    commande_id TEXT,
    montant NUMERIC,
    date_commande DATE
)
SERVER serveur_fichiers  
OPTIONS (  
    filename '/imports/commandes_2025-11-23.csv',
    format 'csv',
    header 'true'
);

-- Validation avant import
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE montant IS NULL) AS invalides,
    SUM(montant) AS ca_total
FROM commandes_partenaire_aujourdhui;

-- Import si validation OK
INSERT INTO commandes_locales  
SELECT * FROM commandes_partenaire_aujourdhui  
WHERE montant IS NOT NULL AND montant > 0;  
```

---

## Avantages des FDW

### 1. Unification des données

✅ **Point d'accès unique** : Une seule interface SQL pour toutes vos données

✅ **Langage standard** : SQL pour tout, quelle que soit la source

✅ **Simplicité applicative** : L'application ne voit qu'une seule base PostgreSQL

### 2. Pas de duplication

✅ **Données toujours à jour** : Accès direct aux sources, pas de synchronisation

✅ **Économie de stockage** : Pas besoin de dupliquer des téraoctets de données

✅ **Cohérence garantie** : Pas de problème de données obsolètes

### 3. Flexibilité

✅ **Migration progressive** : Migrez table par table, sans interruption

✅ **Architecture hybride** : Conservez les systèmes legacy le temps nécessaire

✅ **Expérimentation** : Testez sans engagement (les foreign tables sont virtuelles)

### 4. Performance (avec optimisations)

✅ **Query Pushdown** : Le filtrage et l'agrégation se font à la source

✅ **Transfert minimal** : Seules les données nécessaires sont transférées

✅ **Parallélisation** : Requêtes parallèles sur plusieurs sources

### 5. Écosystème riche

✅ **Nombreux FDW disponibles** : Bases SQL, NoSQL, fichiers, APIs

✅ **Communauté active** : Nouveaux FDW régulièrement

✅ **Création personnalisée** : multicorn permet de créer vos propres FDW

---

## Limitations et Considérations

### 1. Performance réseau

⚠️ **Latence** : Chaque requête traverse le réseau

⚠️ **Bande passante** : Les gros volumes peuvent saturer le réseau

**Mitigation** :
- Utiliser le query pushdown (chapitre suivant)
- Matérialiser les données fréquemment consultées
- Filtrer au maximum côté distant

### 2. Garanties transactionnelles

⚠️ **Pas de 2PC par défaut** : Les transactions distribuées ne sont pas garanties à 100%

**Exemple de risque** :
```sql
BEGIN;  
UPDATE table_locale SET statut = 'validé';  
UPDATE table_distante SET stock = stock - 1;  
COMMIT;  
-- Si le COMMIT échoue, une des deux opérations pourrait être validée, l'autre non
```

**Mitigation** :
- Activer `two_phase_commit` pour postgres_fdw
- Gérer l'idempotence au niveau applicatif
- Utiliser des patterns Saga pour les microservices

### 3. Fonctionnalités limitées

⚠️ **Pas de DDL** : Vous ne pouvez pas modifier la structure des foreign tables

```sql
-- ❌ Impossible
ALTER TABLE clients_distants ADD COLUMN telephone TEXT;
```

⚠️ **Pas d'index locaux** : Les foreign tables ne peuvent pas avoir d'index locaux

**Mitigation** :
- Créer des index sur les serveurs distants
- Utiliser des vues matérialisées pour indexation locale

### 4. Complexité opérationnelle

⚠️ **Multiples points de défaillance** : Si une source externe est down, les requêtes échouent

⚠️ **Monitoring distribué** : Il faut surveiller chaque source externe

⚠️ **Sécurité distribuée** : Gérer les permissions sur chaque système

**Mitigation** :
- Monitoring centralisé (Prometheus, Grafana)
- Gestion de secrets (Vault, AWS Secrets Manager)
- Fallback sur vues matérialisées si source indisponible

### 5. Dépendance aux FDW tiers

⚠️ **Maintenance** : Les FDW tiers peuvent ne plus être maintenus

⚠️ **Compatibilité** : Problèmes de compatibilité entre versions PostgreSQL et FDW

⚠️ **Bugs** : Moins testés que les fonctionnalités PostgreSQL natives

**Mitigation** :
- Privilégier les FDW officiels (postgres_fdw, file_fdw)
- Choisir des FDW activement maintenus (vérifier GitHub)
- Avoir un plan de secours (import classique)

---

## Comparaison : FDW vs Alternatives

### FDW vs ETL traditionnel

| Critère | Foreign Data Wrappers | ETL (Extract, Transform, Load) |
|---------|----------------------|--------------------------------|
| **Fraîcheur des données** | ✅ Temps réel | ❌ Batch (retard) |
| **Duplication** | ✅ Aucune | ❌ Données dupliquées |
| **Complexité** | 🟡 Moyenne | 🔴 Élevée (outils ETL) |
| **Performance lecture** | ⚠️ Latence réseau | ✅ Rapide (local) |
| **Performance écriture** | 🟡 Moyenne | ✅ Optimisé (bulk) |
| **Transformations complexes** | ⚠️ Limitées | ✅ Illimitées |
| **Historisation** | ❌ Non natif | ✅ Facile (snapshots) |
| **Cas d'usage idéal** | Requêtes ponctuelles, migration | Data warehouse, analytics |

**Conclusion** : Les FDW et l'ETL sont **complémentaires**, pas en opposition.

### FDW vs Réplication

| Critère | Foreign Data Wrappers | Réplication (Logique/Physique) |
|---------|----------------------|-------------------------------|
| **Fraîcheur des données** | ✅ Temps réel exact | 🟡 Quasi temps réel (lag) |
| **Bi-directionnel** | ✅ Oui (lecture/écriture) | ⚠️ Dépend (souvent lecture seule) |
| **Overhead serveur source** | ⚠️ Requêtes directes | 🟢 Minimal (streaming WAL) |
| **Stockage** | ✅ Aucune duplication | ❌ Duplication complète |
| **Complexité setup** | 🟢 Simple | 🟡 Moyenne à élevée |
| **Performance** | ⚠️ Latence réseau | ✅ Excellent (local) |
| **Sélectivité** | ✅ Tables spécifiques | ⚠️ Tout ou filtrage limité |
| **Cas d'usage idéal** | Accès ponctuel, fédération | HA, read replicas, DR |

**Conclusion** : Utilisez la réplication pour la haute disponibilité et les performances critiques, les FDW pour la flexibilité et l'intégration.

### FDW vs API applicatif

| Critère | Foreign Data Wrappers | API REST / GraphQL |
|---------|----------------------|-------------------|
| **Langage** | ✅ SQL standard | ❌ Langage spécifique |
| **Agrégations** | ✅ Facile (SQL) | 🟡 Complexe (logique applicative) |
| **Jointures** | ✅ SQL natif | ❌ Multiple appels + merge |
| **Cache** | 🟡 Vues matérialisées | ✅ Cache applicatif (Redis) |
| **Sécurité** | 🟡 Database-level | ✅ Fine-grained (JWT, OAuth) |
| **Scalabilité** | ⚠️ Limité par DB | ✅ Horizontal scaling |
| **Découplage** | ⚠️ Couplage fort (SQL) | ✅ Découplage (contrat API) |
| **Cas d'usage idéal** | Analytics, reporting | Applications web, microservices |

**Conclusion** : Les APIs restent préférables pour les architectures microservices en production, les FDW excellent pour l'analytique et le reporting.

---

## Bonnes Pratiques Générales

### 1. Choisir le bon FDW

✅ **Privilégier les FDW officiels** quand possible (postgres_fdw, file_fdw)

✅ **Vérifier la maturité** des FDW tiers :
- Activité GitHub (derniers commits)
- Nombre d'utilisateurs (stars, forks)
- Documentation
- Support communautaire

✅ **Tester en développement** avant production

### 2. Sécurité

✅ **Principe du moindre privilège** : Créer des utilisateurs dédiés avec permissions minimales

```sql
-- Sur la source distante
CREATE USER fdw_readonly WITH PASSWORD 'mot_de_passe_fort';  
GRANT CONNECT ON DATABASE prod TO fdw_readonly;  
GRANT SELECT ON ALL TABLES IN SCHEMA public TO fdw_readonly;  
```

✅ **Chiffrement** : Utiliser SSL/TLS pour les connexions

```sql
CREATE SERVER serveur_distant
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host 'db.exemple.com',
        port '5432',
        dbname 'prod',
        sslmode 'require'  -- Force SSL
    );
```

✅ **Gestion des secrets** : Ne pas stocker les mots de passe en dur

- Utiliser `.pgpass` pour les credentials
- Gestionnaires de secrets (Vault, AWS Secrets Manager)
- Rotation régulière des mots de passe

### 3. Performance

✅ **Maximiser le pushdown** : Écrire des requêtes qui peuvent être poussées (voir chapitre suivant)

✅ **Limiter les colonnes** : Sélectionner uniquement les colonnes nécessaires

✅ **Utiliser LIMIT** : Toujours pour les explorations

✅ **Index sur sources distantes** : Créer les index appropriés sur les serveurs distants

✅ **Matérialiser si nécessaire** : Vues matérialisées pour données fréquemment consultées

### 4. Monitoring

✅ **Surveiller les requêtes** : pg_stat_statements pour identifier les lentes

✅ **Monitorer les connexions** : Éviter la saturation des connexions sur les sources

✅ **Alerting** : Alertes si sources distantes indisponibles

### 5. Documentation

✅ **Documenter la topologie** : Schéma des dépendances entre sources

✅ **Documenter les foreign tables** : Quelle table virtuelle pointe vers quelle source

✅ **Runbooks** : Procédures en cas de panne d'une source

---

## Quand Utiliser les FDW ?

### ✅ Utilisez les FDW quand :

- Vous devez **interroger occasionnellement** des données distantes
- Vous faites une **migration progressive** d'un SGBD vers PostgreSQL
- Vous construisez un **hub de données / data warehouse**
- Vous avez besoin de **jointures entre sources hétérogènes**
- Les données changent fréquemment et vous avez besoin du **temps réel**
- Vous voulez **éviter la duplication** de données
- Vous prototypez une **architecture de données**

### ❌ N'utilisez PAS les FDW quand :

- Les **performances sont critiques** (< 10 ms) → Préférez la réplication
- Vous avez besoin de **transactions distribuées strictes** → Utilisez un système de transactions distribuées (XA) ou patterns Saga
- Le **volume de données transféré est énorme** (> 1 Go par requête) → Préférez ETL batch
- La **latence réseau est élevée** (> 100 ms) → Matérialisez les données localement
- Vous construisez une **architecture microservices** en production → Préférez les APIs

---

## Prochaines Étapes

Ce chapitre a introduit les concepts fondamentaux des Foreign Data Wrappers et leur écosystème. Les chapitres suivants approfondiront :

### 18.4.1. postgres_fdw : Fédération PostgreSQL
- Configuration détaillée
- Cas d'usage avancés
- Gestion des transactions

### 18.4.2. file_fdw, oracle_fdw, mysql_fdw
- Spécificités de chaque FDW
- Configuration et prérequis
- Migration de bases de données

### 18.4.3. Pushdown d'opérations et performance
- Optimisation des requêtes
- Query pushdown
- Benchmarking et monitoring

---

## Résumé

### Points clés à retenir

✅ Les **Foreign Data Wrappers (FDW)** permettent à PostgreSQL d'accéder à des sources de données externes comme si elles étaient des tables locales

✅ PostgreSQL devient un **hub de données** unifié pour interroger SQL, NoSQL, fichiers, APIs, etc.

✅ Les FDW implémentent le **standard SQL/MED** pour la gestion de données externes

✅ L'écosystème FDW est **riche** : postgres_fdw, oracle_fdw, mysql_fdw, mongo_fdw, file_fdw, et bien d'autres

✅ Les cas d'usage principaux sont : **fédération de données**, **migration progressive**, **data warehouse**, **microservices**

✅ Les FDW ont des **avantages** (temps réel, pas de duplication) et des **limitations** (latence, transactions distribuées)

✅ Les FDW sont **complémentaires** avec ETL, réplication et APIs, pas en opposition

⚠️ Toujours **tester les performances** et **maximiser le pushdown** (chapitre suivant)

### Vocabulaire essentiel

| Terme | Définition courte |
|-------|------------------|
| **FDW** | Extension pour accéder à des sources externes |
| **Foreign Server** | Configuration de connexion vers une source |
| **Foreign Table** | Table virtuelle pointant vers des données externes |
| **Pushdown** | Délégation du traitement vers la source (chapitre suivant) |
| **SQL/MED** | Standard SQL pour Management of External Data |

---


⏭️ [postgres_fdw : Fédération PostgreSQL](/18-extensions-et-integrations/04.1-postgres-fdw.md)
