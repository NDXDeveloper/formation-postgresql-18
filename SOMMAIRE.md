# 📚 Sommaire - Formation PostgreSQL 18
**Un guide progressif pour découvrir et approfondir PostgreSQL**

---

## **Partie 1 : Introduction et Concepts Fondamentaux (Débutant)**

### 1. [Introduction aux Bases de Données](01-introduction-aux-bases-de-donnees/README.md)
- 1.1 [Qu'est-ce qu'une donnée ? Qu'est-ce qu'une base de données ?](01-introduction-aux-bases-de-donnees/01-quest-ce-quune-donnee.md)
- 1.2 [Le concept de SGBD (Système de Gestion de Bases de Données)](01-introduction-aux-bases-de-donnees/02-concept-de-sgbd.md)
- 1.3 [Le modèle relationnel (SGBDR) vs NoSQL : Comparaison conceptuelle](01-introduction-aux-bases-de-donnees/03-modele-relationnel-vs-nosql.md)
- 1.4 [Le concept de transaction et les propriétés ACID](01-introduction-aux-bases-de-donnees/04-transactions-et-proprietes-acid.md)

### 2. [Présentation de PostgreSQL](02-presentation-de-postgresql/README.md)
- 2.1 [Histoire et philosophie : Le SGBD "objet-relationnel"](02-presentation-de-postgresql/01-histoire-et-philosophie.md)
- 2.2 [Gestion des versions et cycle de vie (PostgreSQL 18 : Septembre 2025)](02-presentation-de-postgresql/02-gestion-des-versions-et-cycle-de-vie.md)
- 2.3 [Cas d'utilisation et positionnement dans l'industrie](02-presentation-de-postgresql/03-cas-utilisation-et-positionnement.md)
- 2.4 [Licence et communauté : L'écosystème open-source](02-presentation-de-postgresql/04-licence-et-communaute.md)
- 2.5 [PostgreSQL vs MySQL, Oracle, SQL Server : Forces et différences](02-presentation-de-postgresql/05-postgresql-vs-autres-sgbd.md)

### 3. [Architecture de PostgreSQL](03-architecture-de-postgresql/README.md)
- 3.1 [Le modèle client-serveur et le protocole réseau (Wire Protocol 3.2)](03-architecture-de-postgresql/01-modele-client-serveur.md)
- 3.2 [Anatomie d'une instance : Postmaster et processus d'arrière-plan](03-architecture-de-postgresql/02-anatomie-dune-instance.md)
- 3.3 [Gestion de la mémoire : Shared Buffers vs Local Memory](03-architecture-de-postgresql/03-gestion-de-la-memoire.md)
- 3.4 [Structure physique : Heap files, TOAST et WAL](03-architecture-de-postgresql/04-structure-physique.md)
- **3.5 [Nouveauté PG 18 : Le sous-système I/O asynchrone (AIO)](03-architecture-de-postgresql/05-sous-systeme-io-asynchrone.md)** 🆕
- 3.6 [Outils de l'écosystème : psql, pgAdmin, DBeaver et Connection Pooling](03-architecture-de-postgresql/06-outils-de-lecosysteme.md)

### 4. [Les Objets de la Base de Données (DDL)](04-objets-de-la-base-de-donnees/README.md)
- 4.1 [Hiérarchie logique : Instance → Database → Schema → Table](04-objets-de-la-base-de-donnees/01-hierarchie-logique.md)
- 4.2 [Le concept de Schéma (Namespaces) et le search_path](04-objets-de-la-base-de-donnees/02-concept-de-schema.md)
- 4.3 [CREATE TABLE : Définir une structure](04-objets-de-la-base-de-donnees/03-create-table.md)
- 4.4 [Les types de données fondamentaux](04-objets-de-la-base-de-donnees/04-types-de-donnees-fondamentaux.md)
    - 4.4.1 [Numériques (INTEGER, SERIAL/IDENTITY, NUMERIC, FLOAT)](04-objets-de-la-base-de-donnees/04.1-types-numeriques.md)
    - 4.4.2 [Texte (VARCHAR, TEXT, CHAR)](04-objets-de-la-base-de-donnees/04.2-types-texte.md)
    - 4.4.3 [Temporels (DATE, TIMESTAMP, TIMESTAMPTZ, INTERVAL)](04-objets-de-la-base-de-donnees/04.3-types-temporels.md)
    - 4.4.4 [Spécifiques PostgreSQL (JSON/JSONB, ARRAYS, UUID/UUIDv7, ENUMS)](04-objets-de-la-base-de-donnees/04.4-types-specifiques-postgresql.md)
    - 4.4.5 [Types géométriques, hstore et ltree](04-objets-de-la-base-de-donnees/04.5-types-geometriques-hstore-ltree.md)
    - 4.4.6 [Types binaires (BYTEA) et XML](04-objets-de-la-base-de-donnees/04.6-types-binaires-xml.md)
- 4.5 [Séquences (SEQUENCE) et génération automatique](04-objets-de-la-base-de-donnees/05-sequences-et-generation-automatique.md)
- 4.6 [Domaines et types personnalisés (CREATE TYPE)](04-objets-de-la-base-de-donnees/06-domaines-et-types-personnalises.md)
- 4.7 [Gestion des modifications de schéma (ALTER, DROP) et verrouillage](04-objets-de-la-base-de-donnees/07-modifications-de-schema.md)

---

## **Partie 2 : Le Langage SQL - Interrogation et Manipulation (Intermédiaire)**

### 5. [Requêtes de Sélection (DQL)](05-requetes-de-selection/README.md)
- 5.1 [L'ordre d'exécution logique d'une requête SQL](05-requetes-de-selection/01-ordre-execution-logique.md)
- 5.2 [Filtrage (WHERE) et logique booléenne](05-requetes-de-selection/02-filtrage-where.md)
- 5.3 [Le piège du NULL : Logique ternaire et fonctions de gestion (COALESCE, NULLIF)](05-requetes-de-selection/03-piege-du-null.md)
- 5.4 [Tri (ORDER BY) et gestion des NULLs (NULLS FIRST/LAST)](05-requetes-de-selection/04-tri-order-by.md)
- 5.5 [Pagination et limitation (LIMIT, OFFSET) : Avantages et limites](05-requetes-de-selection/05-pagination-et-limitation.md)
- 5.6 [DISTINCT et élimination des doublons (DISTINCT ON)](05-requetes-de-selection/06-distinct-elimination-doublons.md)

### 6. [Manipulation des Données (DML)](06-manipulation-des-donnees/README.md)
- 6.1 [INSERT : Insertion simple, multiple, et import via COPY](06-manipulation-des-donnees/01-insert-et-copy.md)
- **6.2 [Nouveauté PG 18 : Améliorations COPY et gestion du marqueur de fin \. en CSV](06-manipulation-des-donnees/02-ameliorations-copy-pg18.md)** 🆕
- 6.3 [UPDATE et DELETE : Ciblage et précautions](06-manipulation-des-donnees/03-update-et-delete.md)
- 6.4 [La clause RETURNING : Une spécificité puissante](06-manipulation-des-donnees/04-clause-returning.md)
- **6.5 [Nouveauté PG 18 : Support OLD et NEW dans les clauses RETURNING](06-manipulation-des-donnees/05-old-new-dans-returning.md)** 🆕
- 6.6 [TRUNCATE vs DELETE : Différences transactionnelles et performances](06-manipulation-des-donnees/06-truncate-vs-delete.md)
- 6.7 ["Upsert" : La clause ON CONFLICT (DO NOTHING, DO UPDATE)](06-manipulation-des-donnees/07-upsert-on-conflict.md)
- 6.8 [MERGE : Consolidation de données (avec OLD/NEW)](06-manipulation-des-donnees/08-merge-consolidation.md)

### 7. [Relations et Jointures](07-relations-et-jointures/README.md)
- 7.1 [Contraintes d'intégrité : PK, FK, UNIQUE, CHECK, NOT NULL](07-relations-et-jointures/01-contraintes-integrite.md)
- **7.2 [Nouveauté PG 18 : Contraintes temporelles (Temporal Constraints)](07-relations-et-jointures/02-contraintes-temporelles.md)** 🆕
- 7.3 [Théorie des ensembles et jointures : Produit cartésien et sélection](07-relations-et-jointures/03-theorie-des-ensembles.md)
- 7.4 [Types de jointures : INNER, LEFT/RIGHT/FULL OUTER, CROSS](07-relations-et-jointures/04-types-de-jointures.md)
- 7.5 [Self-Joins : Joindre une table à elle-même](07-relations-et-jointures/05-self-joins.md)
- 7.6 [Jointures latérales (LATERAL JOIN) : Corrélation avancée](07-relations-et-jointures/06-lateral-join.md)
- 7.7 [Anti-jointures et Semi-jointures (NOT EXISTS, NOT IN, EXCEPT)](07-relations-et-jointures/07-anti-jointures-semi-jointures.md)

### 8. [Agrégation et Groupement](08-agregation-et-groupement/README.md)
- 8.1 [Fonctions d'agrégation standards (COUNT, SUM, AVG, MIN, MAX)](08-agregation-et-groupement/01-fonctions-agregation-standards.md)
- 8.2 [Fonctions d'agrégation statistiques (STDDEV, VARIANCE, CORR, PERCENTILE)](08-agregation-et-groupement/02-fonctions-agregation-statistiques.md)
- 8.3 [GROUP BY et HAVING : Filtrage post-agrégation](08-agregation-et-groupement/03-group-by-et-having.md)
- 8.4 [Extensions de groupement (ROLLUP, CUBE, GROUPING SETS)](08-agregation-et-groupement/04-extensions-de-groupement.md)
- 8.5 [Filtres d'agrégation (FILTER clause)](08-agregation-et-groupement/05-filtres-agregation.md)
- 8.6 [Agrégations ordonnées (WITHIN GROUP, string_agg)](08-agregation-et-groupement/06-agregations-ordonnees.md)

---

## **Partie 3 : SQL Avancé et Modélisation (Avancé)**

### 9. [Techniques SQL Avancées](09-techniques-sql-avancees/README.md)
- 9.1 [Sous-requêtes (scalaires, vectorielles, table) et performance](09-techniques-sql-avancees/01-sous-requetes.md)
- 9.2 [CTE (Common Table Expressions) et l'option MATERIALIZED](09-techniques-sql-avancees/02-cte-common-table-expressions.md)
- 9.3 [CTE Récursives : Parcourir des hiérarchies (arbres, graphes)](09-techniques-sql-avancees/03-cte-recursives.md)
- 9.4 [Opérations d'ensemble : UNION, INTERSECT, EXCEPT (ALL)](09-techniques-sql-avancees/04-operations-ensemble.md)
- 9.5 [CASE expressions et logique conditionnelle](09-techniques-sql-avancees/05-case-expressions.md)
- **9.6 [Nouveauté PG 18 : Optimisation des OR-clauses transformées en ANY](09-techniques-sql-avancees/06-optimisation-or-clauses.md)** 🆕

### 10. [Fonctions de Fenêtrage (Window Functions)](10-fonctions-de-fenetrage/README.md)
- 10.1 [Philosophie : Agréger sans grouper](10-fonctions-de-fenetrage/01-philosophie-aggreger-sans-grouper.md)
- 10.2 [La clause OVER, PARTITION BY et ORDER BY](10-fonctions-de-fenetrage/02-clause-over-partition-order.md)
- 10.3 [Frames de fenêtre : ROWS vs RANGE vs GROUPS](10-fonctions-de-fenetrage/03-frames-de-fenetre.md)
- 10.4 [Fonctions de rang (RANK, DENSE_RANK, ROW_NUMBER, NTILE)](10-fonctions-de-fenetrage/04-fonctions-de-rang.md)
- 10.5 [Fonctions de valeur (LAG, LEAD, FIRST_VALUE, LAST_VALUE)](10-fonctions-de-fenetrage/05-fonctions-de-valeur.md)
- 10.6 [Fonctions d'agrégation en fenêtrage](10-fonctions-de-fenetrage/06-fonctions-agregation-fenetrage.md)
- 10.7 [Cas d'usage avancés : Top N par groupe, moyenne mobile](10-fonctions-de-fenetrage/07-cas-usage-avances.md)

### 11. [Modélisation Avancée](11-modelisation-avancee/README.md)
- 11.1 [Normalisation (1NF à 3NF/BCNF) vs Dénormalisation stratégique](11-modelisation-avancee/01-normalisation-vs-denormalisation.md)
- 11.2 [Modélisation JSONB : Quand utiliser le NoSQL dans SQL](11-modelisation-avancee/02-modelisation-jsonb.md)
- 11.3 [Héritage de tables (Table Inheritance) : Concept et limites](11-modelisation-avancee/03-heritage-de-tables.md)
- 11.4 [Partitionnement de tables (Déclaratif : Range, List, Hash)](11-modelisation-avancee/04-partitionnement-de-tables.md)
    - 11.4.1 [Stratégies de partitionnement](11-modelisation-avancee/04.1-strategies-partitionnement.md)
    - 11.4.2 [Partition Pruning et Partition-wise Join](11-modelisation-avancee/04.2-partition-pruning-wise-join.md)
    - 11.4.3 [Détachement et attachement de partitions](11-modelisation-avancee/04.3-detachement-attachement-partitions.md)
- 11.5 [Vues et Vues Matérialisées : Refresh et indexation](11-modelisation-avancee/05-vues-et-vues-materialisees.md)
- **11.6 [Nouveauté PG 18 : Colonnes générées virtuelles (Virtual Generated Columns)](11-modelisation-avancee/06-colonnes-generees-virtuelles.md)** 🆕
- 11.7 [Gestion des contraintes différées (DEFERRABLE, INITIALLY DEFERRED)](11-modelisation-avancee/07-contraintes-differees.md)

---

## **Partie 4 : Administration, Performance et Architecture Avancée (Avancé)**

### 12. [Concurrence et Transactions](12-concurrence-et-transactions/README.md)
- 12.1 [Cycle de vie d'une transaction (BEGIN, SAVEPOINT, COMMIT, ROLLBACK)](12-concurrence-et-transactions/01-cycle-de-vie-transaction.md)
- 12.2 [MVCC (Multiversion Concurrency Control) : Le cœur de PostgreSQL](12-concurrence-et-transactions/02-mvcc-multiversion-concurrency.md)
- 12.3 [Niveaux d'isolation ANSI (Read Uncommitted, Read Committed, Repeatable Read, Serializable)](12-concurrence-et-transactions/03-niveaux-isolation-ansi.md)
- 12.4 [Anomalies transactionnelles : Dirty Read, Non-Repeatable Read, Phantom Read](12-concurrence-et-transactions/04-anomalies-transactionnelles.md)
- 12.5 [Gestion des verrous (Locks) : Types et Deadlocks](12-concurrence-et-transactions/05-gestion-des-verrous.md)
- 12.6 [Advisory Locks : Verrouillage applicatif personnalisé](12-concurrence-et-transactions/06-advisory-locks.md)
- 12.7 [Stratégies de détection et résolution des deadlocks](12-concurrence-et-transactions/07-strategies-detection-resolution-deadlocks.md)

### 13. [Indexation et Optimisation](13-indexation-et-optimisation/README.md)
- 13.1 [Stratégies de scan : Séquentiel vs Index vs Bitmap vs Index-Only](13-indexation-et-optimisation/01-strategies-de-scan.md)
- 13.2 [L'index B-Tree : Le couteau suisse (structure et algorithme)](13-indexation-et-optimisation/02-index-btree.md)
- **13.3 [Nouveauté PG 18 : Skip Scan optimization pour index multi-colonnes](13-indexation-et-optimisation/03-skip-scan-optimization.md)** 🆕
- 13.4 [Index spécialisés](13-indexation-et-optimisation/04-index-specialises.md)
    - 13.4.1 [GIN (Generalized Inverted Index) : JSONB, Full-Text, Arrays](13-indexation-et-optimisation/04.1-gin-generalized-inverted-index.md)
    - 13.4.2 [GiST (Generalized Search Tree) : Géométrie, Texte, ltree](13-indexation-et-optimisation/04.2-gist-generalized-search-tree.md)
    - 13.4.3 [BRIN (Block Range Index) : Données séquentielles massives](13-indexation-et-optimisation/04.3-brin-block-range-index.md)
    - 13.4.4 [Hash : Égalité stricte](13-indexation-et-optimisation/04.4-hash-egalite-stricte.md)
    - 13.4.5 [SP-GiST (Space-Partitioned GiST) : Structures non-équilibrées](13-indexation-et-optimisation/04.5-spgist-space-partitioned.md)
- 13.5 [Stratégies d'indexation avancées](13-indexation-et-optimisation/05-strategies-indexation-avancees.md)
    - 13.5.1 [Index partiels (WHERE clause)](13-indexation-et-optimisation/05.1-index-partiels.md)
    - 13.5.2 [Index sur expressions (CREATE INDEX ON table((lower(col))))](13-indexation-et-optimisation/05.2-index-sur-expressions.md)
    - 13.5.3 [Index multi-colonnes et INCLUDE (covering index)](13-indexation-et-optimisation/05.3-index-multi-colonnes-include.md)
    - 13.5.4 [Index sur colonnes JSONB (GIN avec jsonb_path_ops)](13-indexation-et-optimisation/05.4-index-sur-colonnes-jsonb.md)
- 13.6 [Le planificateur de requêtes et les statistiques (pg_stats)](13-indexation-et-optimisation/06-planificateur-et-statistiques.md)
- 13.7 [Lecture et analyse d'un EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)](13-indexation-et-optimisation/07-lecture-analyse-explain.md)
- **13.8 [Nouveauté PG 18 : Améliorations d'EXPLAIN avec affichage automatique des buffers](13-indexation-et-optimisation/08-ameliorations-explain-pg18.md)** 🆕
- **13.9 [Nouveauté PG 18 : Optimisations du planificateur](13-indexation-et-optimisation/09-optimisations-planificateur-pg18.md)** 🆕
    - 13.9.1 [Auto-élimination des self-joins](13-indexation-et-optimisation/09.1-auto-elimination-self-joins.md)
    - 13.9.2 [Réorganisation automatique des colonnes DISTINCT](13-indexation-et-optimisation/09.2-reorganisation-colonnes-distinct.md)
    - 13.9.3 [Optimisation IN (VALUES ...) vers ANY](13-indexation-et-optimisation/09.3-optimisation-in-values-vers-any.md)
- 13.10 [Prepared Statements et performance (PREPARE, EXECUTE)](13-indexation-et-optimisation/10-prepared-statements.md)
- 13.11 [Maintenance des index : REINDEX, VACUUM FULL](13-indexation-et-optimisation/11-maintenance-des-index.md)

### 14. [Observabilité et Monitoring](14-observabilite-et-monitoring/README.md)
- 14.1 [La "Super Extension" : pg_stat_statements (installation et configuration)](14-observabilite-et-monitoring/01-pg-stat-statements.md)
- 14.2 [Vues système essentielles](14-observabilite-et-monitoring/02-vues-systeme-essentielles.md)
    - 14.2.1 [pg_stat_activity : Activité en cours](14-observabilite-et-monitoring/02.1-pg-stat-activity.md)
    - 14.2.2 [pg_stat_database : Métriques par base](14-observabilite-et-monitoring/02.2-pg-stat-database.md)
    - 14.2.3 [pg_stat_user_tables : Statistiques de tables](14-observabilite-et-monitoring/02.3-pg-stat-user-tables.md)
    - 14.2.4 [pg_locks : Verrous actifs](14-observabilite-et-monitoring/02.4-pg-locks.md)
    - 14.2.5 [pg_catalog : Métadonnées système](14-observabilite-et-monitoring/02.5-pg-catalog.md)
- **14.3 [Nouveauté PG 18 : Statistiques de VACUUM et ANALYZE dans pg_stat_all_tables](14-observabilite-et-monitoring/03-statistiques-vacuum-analyze-pg18.md)** 🆕
- **14.4 [Nouveauté PG 18 : Statistiques I/O et WAL par backend](14-observabilite-et-monitoring/04-statistiques-io-wal-backend-pg18.md)** 🆕
- 14.5 [Analyse des logs : Configuration et interprétation (log_line_prefix, auto_explain)](14-observabilite-et-monitoring/05-analyse-des-logs.md)
- 14.6 [Métriques vitales](14-observabilite-et-monitoring/06-metriques-vitales.md)
    - 14.6.1 [Cache Hit Ratio (Buffer Cache)](14-observabilite-et-monitoring/06.1-cache-hit-ratio.md)
    - 14.6.2 [Table et Index Bloat](14-observabilite-et-monitoring/06.2-table-index-bloat.md)
    - 14.6.3 [I/O Wait et Disk Latency](14-observabilite-et-monitoring/06.3-io-wait-disk-latency.md)
    - 14.6.4 [Connexions actives et pools saturés](14-observabilite-et-monitoring/06.4-connexions-actives-pools.md)
    - 14.6.5 [Checkpoints et WAL generation](14-observabilite-et-monitoring/06.5-checkpoints-wal-generation.md)
- 14.7 [Outils de monitoring](14-observabilite-et-monitoring/07-outils-de-monitoring.md)
    - 14.7.1 [pgBadger : Analyse de logs](14-observabilite-et-monitoring/07.1-pgbadger.md)
    - 14.7.2 [pg_stat_kcache : Métriques système (CPU, I/O)](14-observabilite-et-monitoring/07.2-pg-stat-kcache.md)
    - 14.7.3 [Prometheus + postgres_exporter + Grafana](14-observabilite-et-monitoring/07.3-prometheus-postgres-exporter-grafana.md)
    - 14.7.4 [Solutions cloud natives (CloudWatch, Azure Monitor, GCP)](14-observabilite-et-monitoring/07.4-solutions-cloud-natives.md)

### 15. [Programmation Serveur (PL/pgSQL)](15-programmation-serveur/README.md)
- 15.1 [Fonctions SQL et PL/pgSQL : Différences et cas d'usage](15-programmation-serveur/01-fonctions-sql-et-plpgsql.md)
- 15.2 [Catégories de volatilité : VOLATILE, STABLE, IMMUTABLE](15-programmation-serveur/02-categories-de-volatilite.md)
- 15.3 [Procédures stockées et gestion transactionnelle (CALL, COMMIT/ROLLBACK)](15-programmation-serveur/03-procedures-stockees.md)
- 15.4 [Triggers (BEFORE, AFTER, INSTEAD OF)](15-programmation-serveur/04-triggers.md)
    - 15.4.1 [Triggers par ligne (FOR EACH ROW)](15-programmation-serveur/04.1-triggers-par-ligne.md)
    - 15.4.2 [Triggers par instruction (FOR EACH STATEMENT)](15-programmation-serveur/04.2-triggers-par-instruction.md)
    - 15.4.3 [Variables spéciales (NEW, OLD, TG_OP)](15-programmation-serveur/04.3-variables-speciales.md)
- 15.5 [Event Triggers : Surveillance DDL](15-programmation-serveur/05-event-triggers.md)
- 15.6 [Gestion des exceptions (BEGIN...EXCEPTION...END)](15-programmation-serveur/06-gestion-des-exceptions.md)
- 15.7 [Autres langages procéduraux](15-programmation-serveur/07-autres-langages-proceduraux.md)
    - 15.7.1 [PL/Python (plpython3u)](15-programmation-serveur/07.1-plpython.md)
    - 15.7.2 [PL/Perl (plperlu)](15-programmation-serveur/07.2-plperl.md)
    - 15.7.3 [PL/v8 (JavaScript) - mention](15-programmation-serveur/07.3-plv8.md)

### 16. [Administration, Configuration et Sécurité](16-administration-configuration-securite/README.md)
- 16.1 [Authentification vs Autorisation : Concepts fondamentaux](16-administration-configuration-securite/01-authentification-vs-autorisation.md)
- 16.2 [Configuration de l'authentification (pg_hba.conf)](16-administration-configuration-securite/02-configuration-authentification.md)
    - 16.2.1 [Méthodes : trust, password, md5, scram-sha-256, cert, ldap](16-administration-configuration-securite/02.1-methodes-authentification.md)
    - **16.2.2 [Nouveauté PG 18 : Authentification OAuth 2.0](16-administration-configuration-securite/02.2-authentification-oauth-pg18.md)** 🆕
    - **16.2.3 [Nouveauté PG 18 : SCRAM passthrough avec postgres_fdw et dblink](16-administration-configuration-securite/02.3-scram-passthrough-pg18.md)** 🆕
- 16.3 [Dépréciation de MD5 et migration vers SCRAM-SHA-256](16-administration-configuration-securite/03-depreciation-md5-migration-scram.md)
- 16.4 [Gestion des autorisations (GRANT/REVOKE)](16-administration-configuration-securite/04-gestion-des-autorisations.md)
    - 16.4.1 [Permissions sur objets (TABLE, SEQUENCE, FUNCTION)](16-administration-configuration-securite/04.1-permissions-sur-objets.md)
    - 16.4.2 [Permissions de schéma et database](16-administration-configuration-securite/04.2-permissions-schema-database.md)
    - 16.4.3 [Default privileges (ALTER DEFAULT PRIVILEGES)](16-administration-configuration-securite/04.3-default-privileges.md)
- 16.5 [Rôles, Groupes et principe du moindre privilège](16-administration-configuration-securite/05-roles-groupes-moindre-privilege.md)
- 16.6 [Row-Level Security (RLS) : Politiques par ligne](16-administration-configuration-securite/06-row-level-security.md)
- 16.7 [SSL/TLS et chiffrement des connexions](16-administration-configuration-securite/07-ssl-tls-chiffrement.md)
- **16.8 [Nouveauté PG 18 : Mode FIPS et configuration TLS 1.3 (ssl_tls13_ciphers)](16-administration-configuration-securite/08-mode-fips-tls13-pg18.md)** 🆕
- 16.9 [Chiffrement au repos (Transparent Data Encryption - TDE)](16-administration-configuration-securite/09-chiffrement-au-repos.md)
- 16.10 [Maintenance vitale](16-administration-configuration-securite/10-maintenance-vitale.md)
    - 16.10.1 [VACUUM : Récupération d'espace et prévention XID wraparound](16-administration-configuration-securite/10.1-vacuum.md)
    - 16.10.2 [ANALYZE : Mise à jour des statistiques du planificateur](16-administration-configuration-securite/10.2-analyze.md)
    - **16.10.3 [Nouveauté PG 18 : Autovacuum et ajustements dynamiques (autovacuum_worker_slots)](16-administration-configuration-securite/10.3-autovacuum-ajustements-pg18.md)** 🆕
    - **16.10.4 [Nouveauté PG 18 : Nouveau paramètre autovacuum_vacuum_max_threshold](16-administration-configuration-securite/10.4-autovacuum-vacuum-max-threshold-pg18.md)** 🆕
- 16.11 [Sauvegardes et restauration](16-administration-configuration-securite/11-sauvegardes-et-restauration.md)
    - 16.11.1 [Sauvegardes logiques (pg_dump, pg_dumpall)](16-administration-configuration-securite/11.1-sauvegardes-logiques.md)
    - 16.11.2 [Sauvegardes physiques (pg_basebackup)](16-administration-configuration-securite/11.2-sauvegardes-physiques.md)
    - 16.11.3 [Point-In-Time Recovery (PITR) et WAL archiving](16-administration-configuration-securite/11.3-pitr-wal-archiving.md)
    - 16.11.4 [Stratégies de sauvegarde 3-2-1](16-administration-configuration-securite/11.4-strategies-sauvegarde-321.md)
- **16.12 [Nouveauté PG 18 : Data Checksums activés par défaut (--no-data-checksums pour désactiver)](16-administration-configuration-securite/12-data-checksums-pg18.md)** 🆕
- 16.13 [Tuning et Configuration](16-administration-configuration-securite/13-tuning-et-configuration.md)
    - 16.13.1 [Paramètres critiques (shared_buffers, work_mem, maintenance_work_mem, effective_cache_size)](16-administration-configuration-securite/13.1-parametres-critiques.md)
    - **16.13.2 [Nouveauté PG 18 : Configuration du sous-système I/O (io_method='sync' vs 'async')](16-administration-configuration-securite/13.2-configuration-io-pg18.md)** 🆕
    - 16.13.3 [Configuration WAL (wal_level, max_wal_size, checkpoint_timeout)](16-administration-configuration-securite/13.3-configuration-wal.md)
    - 16.13.4 [Configuration autovacuum](16-administration-configuration-securite/13.4-configuration-autovacuum.md)
    - 16.13.5 [PGTune et outils d'aide à la configuration](16-administration-configuration-securite/13.5-pgtune-outils-aide.md)
    - 16.13.6 [Connection Pooling avec PgBouncer (transaction vs session)](16-administration-configuration-securite/13.6-connection-pooling-pgbouncer.md)

### 17. [Haute Disponibilité et Réplication](17-haute-disponibilite-et-replication/README.md)
- 17.1 [Le concept de WAL Shipping et archive_mode](17-haute-disponibilite-et-replication/01-wal-shipping-archive-mode.md)
- 17.2 [Réplication Physique (Streaming Replication)](17-haute-disponibilite-et-replication/02-replication-physique.md)
    - 17.2.1 [Configuration Primary/Standby](17-haute-disponibilite-et-replication/02.1-configuration-primary-standby.md)
    - 17.2.2 [Synchronous vs Asynchronous](17-haute-disponibilite-et-replication/02.2-synchronous-vs-asynchronous.md)
    - 17.2.3 [Cascading Replication](17-haute-disponibilite-et-replication/02.3-cascading-replication.md)
- 17.3 [Réplication Logique (Logical Replication)](17-haute-disponibilite-et-replication/03-replication-logique.md)
    - 17.3.1 [Publications et Subscriptions](17-haute-disponibilite-et-replication/03.1-publications-subscriptions.md)
    - 17.3.2 [Cas d'usage : Migrations, Selective replication](17-haute-disponibilite-et-replication/03.2-cas-usage-migrations.md)
    - 17.3.3 [Limitations et considérations](17-haute-disponibilite-et-replication/03.3-limitations-considerations.md)
- 17.4 [Slots de réplication : Physical vs Logical](17-haute-disponibilite-et-replication/04-slots-de-replication.md)
- 17.5 [Failover et Promotion](17-haute-disponibilite-et-replication/05-failover-et-promotion.md)
    - 17.5.1 [Promotion manuelle (pg_ctl promote)](17-haute-disponibilite-et-replication/05.1-promotion-manuelle.md)
    - 17.5.2 [Patroni : HA automatisé avec consensus (etcd, Consul)](17-haute-disponibilite-et-replication/05.2-patroni-ha-automatise.md)
    - 17.5.3 [Repmgr : Gestion de réplication simplifiée](17-haute-disponibilite-et-replication/05.3-repmgr.md)
- 17.6 [Architectures HA](17-haute-disponibilite-et-replication/06-architectures-ha.md)
    - 17.6.1 [Synchrone vs Asynchrone : Trade-offs](17-haute-disponibilite-et-replication/06.1-synchrone-vs-asynchrone-tradeoffs.md)
    - 17.6.2 [Quorum-based commit](17-haute-disponibilite-et-replication/06.2-quorum-based-commit.md)
    - 17.6.3 [Load balancing avec HAProxy ou PgPool-II](17-haute-disponibilite-et-replication/06.3-load-balancing.md)

---

## **Partie 5 : Écosystème, Production et Ouverture**

### 18. [Extensions et Intégrations](18-extensions-et-integrations/README.md)
- 18.1 [Système d'extensions PostgreSQL (CREATE EXTENSION)](18-extensions-et-integrations/01-systeme-extensions.md)
- 18.2 [PostGIS : La référence spatiale mondiale](18-extensions-et-integrations/02-postgis.md)
    - 18.2.1 [Types géométriques (Point, LineString, Polygon)](18-extensions-et-integrations/02.1-types-geometriques.md)
    - 18.2.2 [Index spatiaux (GiST, SP-GiST)](18-extensions-et-integrations/02.2-index-spatiaux.md)
    - 18.2.3 [Fonctions spatiales (ST_Distance, ST_Intersects)](18-extensions-et-integrations/02.3-fonctions-spatiales.md)
- 18.3 [Full-Text Search Avancé](18-extensions-et-integrations/03-full-text-search.md)
    - 18.3.1 [tsvector et tsquery](18-extensions-et-integrations/03.1-tsvector-tsquery.md)
    - 18.3.2 [Ranking et pondération (ts_rank, setweight)](18-extensions-et-integrations/03.2-ranking-ponderation.md)
    - 18.3.3 [Dictionnaires et langues multiples](18-extensions-et-integrations/03.3-dictionnaires-langues.md)
    - 18.3.4 [Index GIN pour la performance](18-extensions-et-integrations/03.4-index-gin-performance.md)
- 18.4 [Foreign Data Wrappers (FDW) : PostgreSQL comme hub de données](18-extensions-et-integrations/04-foreign-data-wrappers.md)
    - 18.4.1 [postgres_fdw : Fédération PostgreSQL](18-extensions-et-integrations/04.1-postgres-fdw.md)
    - 18.4.2 [file_fdw, oracle_fdw, mysql_fdw](18-extensions-et-integrations/04.2-autres-fdw.md)
    - 18.4.3 [Pushdown d'opérations et performance](18-extensions-et-integrations/04.3-pushdown-operations.md)
- 18.5 [TimescaleDB : Séries temporelles et hypertables](18-extensions-et-integrations/05-timescaledb.md)
- 18.6 [pgvector : IA, embeddings et recherche vectorielle](18-extensions-et-integrations/06-pgvector.md)
    - 18.6.1 [Recherche de similarité (cosine, L2)](18-extensions-et-integrations/06.1-recherche-similarite.md)
    - 18.6.2 [Index HNSW et IVFFlat](18-extensions-et-integrations/06.2-index-hnsw-ivfflat.md)
    - 18.6.3 [Cas d'usage : RAG, Semantic Search](18-extensions-et-integrations/06.3-cas-usage-rag.md)
- 18.7 [Autres extensions notables](18-extensions-et-integrations/07-autres-extensions.md)
    - 18.7.1 [pg_cron : Planification de tâches](18-extensions-et-integrations/07.1-pg-cron.md)
    - 18.7.2 [pg_partman : Gestion automatisée de partitions](18-extensions-et-integrations/07.2-pg-partman.md)
    - 18.7.3 [pg_stat_kcache : Métriques système](18-extensions-et-integrations/07.3-pg-stat-kcache.md)
    - 18.7.4 [HypoPG : Indexation hypothétique](18-extensions-et-integrations/07.4-hypopg.md)
    - 18.7.5 [pg_repack : Réorganisation sans verrous](18-extensions-et-integrations/07.5-pg-repack.md)

### 19. [PostgreSQL en Production](19-postgresql-en-production/README.md)
- 19.1 [Stratégies de déploiement](19-postgresql-en-production/01-strategies-de-deploiement.md)
    - 19.1.1 [Bare Metal : Configuration optimale](19-postgresql-en-production/01.1-bare-metal.md)
    - 19.1.2 [Virtual Machines (VM)](19-postgresql-en-production/01.2-virtual-machines.md)
    - 19.1.3 [Conteneurs (Docker, Podman)](19-postgresql-en-production/01.3-conteneurs.md)
    - 19.1.4 [Kubernetes (StatefulSets, Operators : Zalando, CloudNativePG)](19-postgresql-en-production/01.4-kubernetes.md)
- 19.2 [PostgreSQL dans le Cloud](19-postgresql-en-production/02-postgresql-dans-le-cloud.md)
    - 19.2.1 [AWS RDS et Aurora PostgreSQL](19-postgresql-en-production/02.1-aws-rds-aurora.md)
    - 19.2.2 [Azure Database for PostgreSQL](19-postgresql-en-production/02.2-azure-database.md)
    - 19.2.3 [Google Cloud SQL et AlloyDB](19-postgresql-en-production/02.3-google-cloud-sql-alloydb.md)
    - 19.2.4 [Managed vs Self-Hosted : Trade-offs](19-postgresql-en-production/02.4-managed-vs-self-hosted.md)
- 19.3 [Migrations majeures](19-postgresql-en-production/03-migrations-majeures.md)
    - **19.3.1 [Nouveauté PG 18 : pg_upgrade amélioré et préservation des statistiques](19-postgresql-en-production/03.1-pg-upgrade-ameliore-pg18.md)** 🆕
    - **19.3.2 [Nouveauté PG 18 : Option --swap pour upgrade rapide](19-postgresql-en-production/03.2-option-swap-pg18.md)** 🆕
    - **19.3.3 [Nouveauté PG 18 : Vérifications parallèles (--jobs)](19-postgresql-en-production/03.3-verifications-paralleles-pg18.md)** 🆕
    - 19.3.4 [Stratégies Blue/Green et Logical Replication](19-postgresql-en-production/03.4-strategies-blue-green.md)
    - 19.3.5 [Tests de migration et validation](19-postgresql-en-production/03.5-tests-migration-validation.md)
- 19.4 [Troubleshooting et Crises](19-postgresql-en-production/04-troubleshooting-et-crises.md)
    - 19.4.1 [Diagnostic des verrous (pg_locks, pg_blocking_pids)](19-postgresql-en-production/04.1-diagnostic-verrous.md)
    - 19.4.2 [Saturation des ressources (CPU, RAM, I/O)](19-postgresql-en-production/04.2-saturation-ressources.md)
    - 19.4.3 [Transaction Wraparound (XID exhaustion) et prévention](19-postgresql-en-production/04.3-transaction-wraparound.md)
    - 19.4.4 [Corruption de données et Checksums](19-postgresql-en-production/04.4-corruption-donnees-checksums.md)
    - 19.4.5 [Slow queries et Query tuning](19-postgresql-en-production/04.5-slow-queries-tuning.md)
    - 19.4.6 [Connection storms et pooling](19-postgresql-en-production/04.6-connection-storms-pooling.md)
- 19.5 [Disaster Recovery (DR)](19-postgresql-en-production/05-disaster-recovery.md)
    - 19.5.1 [RTO et RPO : Définir les objectifs](19-postgresql-en-production/05.1-rto-rpo-objectifs.md)
    - 19.5.2 [Tests de restauration réguliers](19-postgresql-en-production/05.2-tests-restauration.md)
    - 19.5.3 [Géo-réplication et Multi-Region](19-postgresql-en-production/05.3-geo-replication-multi-region.md)
- 19.6 [Checklist de mise en production (Best Practices)](19-postgresql-en-production/06-checklist-mise-en-production.md)
    - 19.6.1 [Hardening sécurité](19-postgresql-en-production/06.1-hardening-securite.md)
    - 19.6.2 [Configuration optimale](19-postgresql-en-production/06.2-configuration-optimale.md)
    - 19.6.3 [Monitoring et alerting](19-postgresql-en-production/06.3-monitoring-alerting.md)
    - 19.6.4 [Backup et DR](19-postgresql-en-production/06.4-backup-et-dr.md)
    - 19.6.5 [Documentation runbooks](19-postgresql-en-production/06.5-documentation-runbooks.md)

### 20. [Drivers, Connexion Applicative et Bonnes Pratiques](20-drivers-connexion-applicative/README.md)
- 20.1 [Drivers populaires](20-drivers-connexion-applicative/01-drivers-populaires.md)
    - 20.1.1 [Python : psycopg3 (psycopg2 legacy)](20-drivers-connexion-applicative/01.1-python-psycopg.md)
    - 20.1.2 [Node.js : node-postgres (pg), Prisma](20-drivers-connexion-applicative/01.2-nodejs-node-postgres.md)
    - 20.1.3 [Java : JDBC, HikariCP, R2DBC](20-drivers-connexion-applicative/01.3-java-jdbc-hikari.md)
    - 20.1.4 [Go : pgx, GORM](20-drivers-connexion-applicative/01.4-go-pgx-gorm.md)
    - 20.1.5 [.NET : Npgsql, Entity Framework Core](20-drivers-connexion-applicative/01.5-dotnet-npgsql.md)
- 20.2 [Gestion des connexions dans les applications](20-drivers-connexion-applicative/02-gestion-des-connexions.md)
    - 20.2.1 [Connection pooling côté application](20-drivers-connexion-applicative/02.1-connection-pooling-application.md)
    - 20.2.2 [PgBouncer : Transaction vs Session pooling](20-drivers-connexion-applicative/02.2-pgbouncer-transaction-session.md)
    - 20.2.3 [Dimensionnement (max_connections vs pool_size)](20-drivers-connexion-applicative/02.3-dimensionnement-connexions.md)
    - 20.2.4 [Connection leaks et timeouts](20-drivers-connexion-applicative/02.4-connection-leaks-timeouts.md)
- 20.3 [Patterns anti-corruption](20-drivers-connexion-applicative/03-patterns-anti-corruption.md)
    - 20.3.1 [N+1 queries : Détection et correction (JOIN, LATERAL)](20-drivers-connexion-applicative/03.1-n-plus-1-queries.md)
    - 20.3.2 [ORM vs SQL brut : Quand utiliser quoi](20-drivers-connexion-applicative/03.2-orm-vs-sql-brut.md)
    - 20.3.3 [Lazy loading vs Eager loading](20-drivers-connexion-applicative/03.3-lazy-vs-eager-loading.md)
    - 20.3.4 [Batching et bulk operations](20-drivers-connexion-applicative/03.4-batching-bulk-operations.md)
- 20.4 [Principes de conception d'APIs avec PostgreSQL](20-drivers-connexion-applicative/04-conception-apis.md)
    - 20.4.1 [Repository pattern](20-drivers-connexion-applicative/04.1-repository-pattern.md)
    - 20.4.2 [Database migrations (Flyway, Liquibase, Alembic)](20-drivers-connexion-applicative/04.2-database-migrations.md)
    - 20.4.3 [Schema versioning](20-drivers-connexion-applicative/04.3-schema-versioning.md)
    - 20.4.4 [Zero-downtime deployments](20-drivers-connexion-applicative/04.4-zero-downtime-deployments.md)

### 20bis. [PostgreSQL et Architectures Modernes](20bis-postgresql-et-architectures-modernes/README.md)
- 20bis.1 [Microservices et bases de données](20bis-postgresql-et-architectures-modernes/01-microservices-et-bdd.md)
    - 20bis.1.1 [Database per service vs Shared database](20bis-postgresql-et-architectures-modernes/01.1-database-per-service.md)
    - 20bis.1.2 [Distributed transactions et Saga pattern](20bis-postgresql-et-architectures-modernes/01.2-distributed-transactions-saga.md)
    - 20bis.1.3 [Foreign Data Wrappers pour la fédération](20bis-postgresql-et-architectures-modernes/01.3-fdw-federation.md)
- 20bis.2 [Event Sourcing et CQRS avec PostgreSQL](20bis-postgresql-et-architectures-modernes/02-event-sourcing-cqrs.md)
    - 20bis.2.1 [Event Store pattern](20bis-postgresql-et-architectures-modernes/02.1-event-store-pattern.md)
    - 20bis.2.2 [NOTIFY/LISTEN pour événements temps réel](20bis-postgresql-et-architectures-modernes/02.2-notify-listen.md)
    - 20bis.2.3 [Logical Decoding et Change Data Capture (CDC)](20bis-postgresql-et-architectures-modernes/02.3-logical-decoding-cdc.md)
    - 20bis.2.4 [Debezium pour streaming d'événements](20bis-postgresql-et-architectures-modernes/02.4-debezium.md)
- 20bis.3 [PostgreSQL en architecture serverless](20bis-postgresql-et-architectures-modernes/03-postgresql-serverless.md)
    - 20bis.3.1 [Neon, Supabase, PlanetScale alternatives](20bis-postgresql-et-architectures-modernes/03.1-neon-supabase-planetscale.md)
    - 20bis.3.2 [Connection pooling serverless (PgBouncer, RDS Proxy)](20bis-postgresql-et-architectures-modernes/03.2-connection-pooling-serverless.md)
    - 20bis.3.3 [Cold starts et gestion des connexions](20bis-postgresql-et-architectures-modernes/03.3-cold-starts-connexions.md)
- 20bis.4 [Intégration avec Kubernetes](20bis-postgresql-et-architectures-modernes/04-integration-kubernetes.md)
    - 20bis.4.1 [StatefulSets : Persistance et identité](20bis-postgresql-et-architectures-modernes/04.1-statefulsets.md)
    - 20bis.4.2 [Operators : Zalando, CloudNativePG, Crunchy](20bis-postgresql-et-architectures-modernes/04.2-operators.md)
    - 20bis.4.3 [Backup automation (Velero, native solutions)](20bis-postgresql-et-architectures-modernes/04.3-backup-automation.md)
    - 20bis.4.4 [Service mesh et observabilité](20bis-postgresql-et-architectures-modernes/04.4-service-mesh-observabilite.md)

### 21. [Conclusion et Perspectives](21-conclusion-et-perspectives/README.md)
- 21.1 [Résumé des concepts clés par niveau](21-conclusion-et-perspectives/01-resume-concepts-cles.md)
    - 21.1.1 [Débutant : SQL, DDL, DML, Contraintes](21-conclusion-et-perspectives/01.1-niveau-debutant.md)
    - 21.1.2 [Intermédiaire : Jointures, Agrégation, Indexation](21-conclusion-et-perspectives/01.2-niveau-intermediaire.md)
    - 21.1.3 [Avancé : MVCC, Réplication, Optimisation, Production](21-conclusion-et-perspectives/01.3-niveau-avance.md)
- 21.2 [L'avenir de PostgreSQL : Tendances et évolutions](21-conclusion-et-perspectives/02-avenir-de-postgresql.md)
    - 21.2.1 [Performances I/O et parallélisation](21-conclusion-et-perspectives/02.1-performances-io-parallelisation.md)
    - 21.2.2 [IA et Machine Learning intégrés](21-conclusion-et-perspectives/02.2-ia-machine-learning.md)
    - 21.2.3 [Cloud-native et distributed PostgreSQL](21-conclusion-et-perspectives/02.3-cloud-native-distributed.md)
    - 21.2.4 [Columnar storage (Hydra, pg_analytics)](21-conclusion-et-perspectives/02.4-columnar-storage.md)
- **21.3 [Les apports majeurs de PostgreSQL 18](21-conclusion-et-perspectives/03-apports-majeurs-pg18.md)** 🆕
    - 21.3.1 [I/O asynchrone (jusqu'à 3× plus rapide)](21-conclusion-et-perspectives/03.1-io-asynchrone.md)
    - 21.3.2 [UUIDv7 et colonnes virtuelles](21-conclusion-et-perspectives/03.2-uuidv7-colonnes-virtuelles.md)
    - 21.3.3 [OAuth et sécurité moderne](21-conclusion-et-perspectives/03.3-oauth-securite-moderne.md)
    - 21.3.4 [Upgrade simplifié avec préservation statistiques](21-conclusion-et-perspectives/03.4-upgrade-simplifie.md)
- 21.4 [Ressources pour aller plus loin](21-conclusion-et-perspectives/04-ressources-pour-aller-plus-loin.md)
    - 21.4.1 [Documentation officielle PostgreSQL](21-conclusion-et-perspectives/04.1-documentation-officielle.md)
    - 21.4.2 [Livres recommandés (PostgreSQL: Up and Running, Art of PostgreSQL)](21-conclusion-et-perspectives/04.2-livres-recommandes.md)
    - 21.4.3 [Communautés (pgsql-general, Reddit r/PostgreSQL, Discord)](21-conclusion-et-perspectives/04.3-communautes.md)
    - 21.4.4 [Conférences (PGConf, FOSDEM, PostgreSQL sessions)](21-conclusion-et-perspectives/04.4-conferences.md)
    - 21.4.5 [Blogs techniques (2ndQuadrant, Percona, CrunchyData)](21-conclusion-et-perspectives/04.5-blogs-techniques.md)
- 21.5 [Roadmap de montée en compétence](21-conclusion-et-perspectives/05-roadmap-montee-en-competence.md)
    - 21.5.1 [Parcours Développeur (0-6 mois, 6-12 mois, 12-24 mois)](21-conclusion-et-perspectives/05.1-parcours-developpeur.md)
    - 21.5.2 [Parcours DevOps/SRE](21-conclusion-et-perspectives/05.2-parcours-devops-sre.md)
    - 21.5.3 [Parcours DBA](21-conclusion-et-perspectives/05.3-parcours-dba.md)
    - 21.5.4 [Certifications PostgreSQL (EnterpriseDB, Percona)](21-conclusion-et-perspectives/05.4-certifications.md)

---

## **Annexes (Optionnelles)**

### A. [Glossaire des Termes Techniques](annexes/glossaire/README.md)
- A.1 [Termes PostgreSQL essentiels (ACID, MVCC, WAL, TOAST, etc.)](annexes/glossaire/01-termes-postgresql-essentiels.md)
- A.2 [Acronymes courants (FK, PK, CTE, FDW, GIN, GiST, etc.)](annexes/glossaire/02-acronymes-courants.md)

### B. [Commandes psql Essentielles](annexes/commandes-psql/README.md)
- B.1 [Navigation (\l, \c, \dt, \d+, \df, etc.)](annexes/commandes-psql/01-navigation.md)
- B.2 [Configuration (\x, \timing, \pset)](annexes/commandes-psql/02-configuration.md)
- B.3 [Export/Import (\copy, \i, \o)](annexes/commandes-psql/03-export-import.md)
- B.4 [Meta-commandes avancées](annexes/commandes-psql/04-meta-commandes-avancees.md)

### C. [Requêtes SQL de Référence](annexes/requetes-sql-reference/README.md)
- C.1 [Requêtes d'administration (locks, bloat, index usage)](annexes/requetes-sql-reference/01-requetes-administration.md)
- C.2 [Requêtes de monitoring (cache hit ratio, slow queries)](annexes/requetes-sql-reference/02-requetes-monitoring.md)
- C.3 [Requêtes d'analyse (statistiques, tables sizes)](annexes/requetes-sql-reference/03-requetes-analyse.md)

### D. [Configuration de Référence par Cas d'Usage](annexes/configuration-reference/README.md)
- D.1 [OLTP (High concurrency, low latency)](annexes/configuration-reference/01-configuration-oltp.md)
- D.2 [OLAP (Data warehouse, analytics)](annexes/configuration-reference/02-configuration-olap.md)
- D.3 [Mixed workload (équilibre OLTP/OLAP)](annexes/configuration-reference/03-configuration-mixed-workload.md)
- D.4 [Développement local](annexes/configuration-reference/04-configuration-developpement-local.md)

### E. [Checklist de Performance](annexes/checklist-performance/README.md)
- E.1 [Audit de configuration](annexes/checklist-performance/01-audit-configuration.md)
- E.2 [Audit d'indexation](annexes/checklist-performance/02-audit-indexation.md)
- E.3 [Audit de requêtes](annexes/checklist-performance/03-audit-requetes.md)
- E.4 [Audit de schéma](annexes/checklist-performance/04-audit-schema.md)

### F. [Nouveautés PostgreSQL 18 en un Coup d'Œil](annexes/nouveautes-pg18/README.md) 🆕
- F.1 [Tableau récapitulatif des features majeures](annexes/nouveautes-pg18/01-tableau-recapitulatif.md)
- F.2 [Impact sur migration et compatibilité](annexes/nouveautes-pg18/02-impact-migration-compatibilite.md)
- F.3 [Recommandations d'adoption](annexes/nouveautes-pg18/03-recommandations-adoption.md)

### G. [Commandes Shell et Scripts Utiles](annexes/commandes-shell-scripts/README.md)
- G.1 [pg_ctl, pg_dump, pg_restore](annexes/commandes-shell-scripts/01-commandes-pg-principales.md)
- G.2 [Scripts de backup automatisés](annexes/commandes-shell-scripts/02-scripts-backup-automatises.md)
- G.3 [Scripts de monitoring système](annexes/commandes-shell-scripts/03-scripts-monitoring-systeme.md)

---

## **Points Forts de Cette Formation**

✅ **Progressive** : Parcours structuré du débutant à l'expert
✅ **Complète** : 21 chapitres + 1 chapitre bonus couvrant tous les aspects
✅ **À jour** : Intégration complète des nouveautés PostgreSQL 18 🆕
✅ **Professionnelle** : Production, monitoring, HA, architectures modernes
✅ **Moderne** : Cloud, Kubernetes, IA, architectures distribuées
✅ **Pratique** : Annexes et références techniques détaillées

**Durée estimée** : 40-60 heures de formation théorique
**Public** : Développeurs, DevOps, DBAs débutants à avancés
**Prérequis** : Bases en SQL et systèmes Unix/Linux

---

**Version** : 1.0 - Novembre 2025
**PostgreSQL** : Version 18 (Septembre 2025)
**Licence** : Creative Commons BY-NC-SA 4.0
