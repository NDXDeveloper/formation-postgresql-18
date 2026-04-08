🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe C : Requêtes SQL de Référence
## Guide d'Introduction et d'Utilisation

---

## Table des Matières

1. [Introduction aux Requêtes de Référence](#1-introduction-aux-requ%C3%AAtes-de-r%C3%A9f%C3%A9rence)  
2. [Comment Utiliser Ces Requêtes](#2-comment-utiliser-ces-requ%C3%AAtes)  
3. [Organisation et Catégories](#3-organisation-et-cat%C3%A9gories)  
4. [Outils et Environnement](#4-outils-et-environnement)  
5. [Bonnes Pratiques d'Utilisation](#5-bonnes-pratiques-dutilisation)  
6. [Index des Requêtes Disponibles](#6-index-des-requ%C3%AAtes-disponibles)

---

## 1. Introduction aux Requêtes de Référence

### 1.1. Qu'est-ce que l'Annexe des Requêtes SQL de Référence ?

Cette annexe est une **bibliothèque de requêtes SQL prêtes à l'emploi** pour :

1. **Administrer** votre base de données PostgreSQL  
2. **Monitorer** les performances et la santé du système  
3. **Analyser** les données et les statistiques  
4. **Diagnostiquer** les problèmes rapidement  
5. **Optimiser** les performances

Ces requêtes ont été soigneusement sélectionnées et testées pour vous aider dans les tâches quotidiennes de gestion et d'optimisation de PostgreSQL.

### 1.2. Pourquoi Cette Annexe Est Essentielle ?

#### Pour les Débutants

- **Apprentissage accéléré** : Voir des requêtes SQL réelles et professionnelles  
- **Copier-coller direct** : Pas besoin de réinventer la roue  
- **Comprendre les concepts** : Chaque requête est expliquée

#### Pour les Utilisateurs Intermédiaires

- **Gain de temps** : Requêtes optimisées et éprouvées  
- **Meilleures pratiques** : Apprendre des patterns efficaces  
- **Référence rapide** : Trouver la bonne requête en quelques secondes

#### Pour les Experts

- **Base de départ** : Personnaliser selon vos besoins spécifiques  
- **Validation** : Confirmer vos approches  
- **Documentation** : Partager avec votre équipe

### 1.3. Ce Que Cette Annexe N'Est PAS

- ❌ **Pas un tutoriel SQL complet** : Pour apprendre SQL, référez-vous aux parties principales du cours  
- ❌ **Pas exhaustif** : Impossible de couvrir tous les cas d'usage  
- ❌ **Pas un remplacement de la documentation** : Consultez toujours la doc PostgreSQL officielle pour les détails

- ✅ **C'est une boîte à outils pratique** pour les situations courantes  
- ✅ **Un point de départ** pour vos propres requêtes  
- ✅ **Une référence rapide** pour les tâches d'administration

---

## 2. Comment Utiliser Ces Requêtes

### 2.1. Structure de Chaque Requête

Chaque requête de référence suit cette structure standard :

```sql
-- ========================================
-- TITRE DE LA REQUÊTE
-- ========================================
-- Description : Ce que fait cette requête
-- Cas d'usage : Quand l'utiliser
-- Fréquence recommandée : Quotidien/Hebdomadaire/Mensuel/À la demande
-- ========================================

SELECT
    -- Colonnes avec commentaires explicatifs
    colonne1,                    -- Commentaire sur colonne1
    colonne2,                    -- Commentaire sur colonne2
    fonction(colonne3) AS alias  -- Commentaire sur le calcul
FROM
    table_source
WHERE
    condition = 'valeur'
ORDER BY
    colonne1 DESC
LIMIT 20;

-- ========================================
-- INTERPRÉTATION DES RÉSULTATS
-- ========================================
-- Valeur normale : Description
-- Seuil d'alerte : Description
-- Action recommandée : Description
-- ========================================
```

### 2.2. Adaptation des Requêtes à Votre Contexte

La plupart des requêtes nécessitent une **personnalisation** :

#### Remplacer les Noms d'Objets

```sql
-- Exemple générique
SELECT * FROM ma_table WHERE status = 'active';

-- Adaptez avec vos vrais noms
SELECT * FROM orders WHERE status = 'pending';
```

#### Ajuster les Seuils

```sql
-- Requête avec seuil par défaut
WHERE n_dead_tup > n_live_tup * 0.2  -- 20% de bloat

-- Ajustez selon votre tolérance
WHERE n_dead_tup > n_live_tup * 0.1  -- 10% de bloat (plus strict)
```

#### Modifier les Limites

```sql
-- Par défaut : Top 20
LIMIT 20;

-- Ajustez selon vos besoins
LIMIT 50;  -- Top 50
-- Ou supprimez la limite pour tout voir
```

### 2.3. Sauvegarder Vos Requêtes Personnalisées

Il est recommandé de créer votre propre **bibliothèque de requêtes** :

#### Option 1 : Fichiers SQL Organisés

```
/sql-queries/
├── admin/
│   ├── locks.sql
│   ├── bloat.sql
│   └── index_usage.sql
├── monitoring/
│   ├── cache_hit_ratio.sql
│   └── slow_queries.sql
└── analysis/
    ├── table_sizes.sql
    └── statistics.sql
```

#### Option 2 : Vues dans la Base de Données

```sql
-- Créer un schéma dédié
CREATE SCHEMA IF NOT EXISTS monitoring;

-- Créer des vues pour vos requêtes fréquentes
CREATE VIEW monitoring.top_tables_by_size AS  
SELECT  
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables  
WHERE schemaname = 'public'  
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC  
LIMIT 20;  

-- Utilisation simple
SELECT * FROM monitoring.top_tables_by_size;
```

#### Option 3 : Snippets dans Votre Éditeur

La plupart des éditeurs (VS Code, DataGrip, pgAdmin) supportent les **snippets** :

```json
// Exemple de snippet VS Code
{
  "Top Tables by Size": {
    "prefix": "pgtop",
    "body": [
      "SELECT",
      "    schemaname,",
      "    tablename,",
      "    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size",
      "FROM pg_tables",
      "WHERE schemaname = '${1:public}'",
      "ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC",
      "LIMIT ${2:20};"
    ],
    "description": "Liste les tables les plus volumineuses"
  }
}
```

### 2.4. Exécution Sécurisée des Requêtes

#### Requêtes en Lecture Seule (Sûres)

- ✅ Toutes les requêtes **SELECT** sont sûres  
- ✅ Elles ne modifient pas les données  
- ✅ Peuvent être exécutées en production sans risque

#### Requêtes de Maintenance (Attention)

- ⚠️ **VACUUM**, **ANALYZE**, **REINDEX** peuvent consommer des ressources  
- ⚠️ Exécutez-les pendant les heures creuses  
- ⚠️ **VACUUM FULL** pose un verrou exclusif : **NE PAS** exécuter en production

#### Requêtes Administratives (Dangereuses)

- 🔴 **pg_terminate_backend()** : Tue une connexion  
- 🔴 **DROP INDEX** : Supprime un index  
- 🔴 **ALTER TABLE** : Modifie la structure

**Règle d'or** : Toujours tester en staging avant production !

---

## 3. Organisation et Catégories

### 3.1. Vue d'Ensemble des Catégories

Les requêtes sont organisées en **trois catégories principales** :

```
Annexe C : Requêtes SQL de Référence
│
├── 1. Requêtes d'Administration
│   ├── Gestion des Locks (Verrous)
│   ├── Détection et Analyse du Bloat
│   └── Surveillance de l'Utilisation des Index
│
├── 2. Requêtes de Monitoring
│   ├── Cache Hit Ratio
│   ├── Slow Queries (Requêtes Lentes)
│   └── Métriques Complémentaires
│
└── 3. Requêtes d'Analyse
    ├── Taille des Objets (Tables, Index, Bases)
    ├── Statistiques des Tables
    ├── Statistiques des Colonnes
    └── Analyse de la Fragmentation
```

### 3.2. Catégorie 1 : Requêtes d'Administration

**Objectif** : Gérer les aspects opérationnels quotidiens de PostgreSQL

#### Locks (Verrous)

- Identifier les transactions bloquantes
- Détecter les deadlocks
- Résoudre les contentions

**Cas d'usage** :
- Application figée, utilisateurs bloqués
- Timeout sur les requêtes
- Diagnostic de performance soudaine

#### Bloat (Gonflement)

- Mesurer l'espace mort dans les tables
- Détecter le bloat dans les index
- Planifier les opérations de nettoyage

**Cas d'usage** :
- Table qui grossit anormalement
- Performances qui se dégradent dans le temps
- Optimisation de l'espace disque

#### Index Usage (Utilisation des Index)

- Identifier les index inutilisés
- Détecter les index redondants
- Mesurer l'efficacité des index

**Cas d'usage** :
- Optimisation des performances INSERT/UPDATE
- Réduction de l'empreinte disque
- Amélioration du cache hit ratio

**📄 Document détaillé** : `requetes-administration-postgresql.md`

### 3.3. Catégorie 2 : Requêtes de Monitoring

**Objectif** : Surveiller la santé et les performances en temps réel

#### Cache Hit Ratio

- Mesurer l'efficacité du buffer cache
- Identifier les tables mal cachées
- Optimiser la configuration de la mémoire

**Cas d'usage** :
- Validation de la configuration `shared_buffers`
- Détection de problèmes d'I/O
- Optimisation du coût cloud (I/O)

#### Slow Queries (Requêtes Lentes)

- Identifier les requêtes les plus chronophages
- Analyser les patterns de performance
- Prioriser les optimisations

**Cas d'usage** :
- Diagnostic de lenteur applicative
- Optimisation proactive
- Analyse post-déploiement

#### Métriques Complémentaires

- Connexions actives
- Checkpoints et WAL
- Risque de XID wraparound
- Réplication lag

**Cas d'usage** :
- Tableau de bord de santé
- Alertes automatisées
- Planification de capacité

**📄 Document détaillé** : `requetes-monitoring-postgresql.md`

### 3.4. Catégorie 3 : Requêtes d'Analyse

**Objectif** : Comprendre la structure et la croissance des données

#### Taille des Objets

- Mesurer la taille des bases, tables, index
- Suivre la croissance dans le temps
- Prévoir les besoins en stockage

**Cas d'usage** :
- Planification de capacité
- Optimisation des coûts
- Identification des candidats au partitionnement

#### Statistiques des Tables

- Analyser l'activité (lectures/écritures)
- Comprendre les patterns d'accès
- Valider l'efficacité des index

**Cas d'usage** :
- Optimisation de schéma
- Choix de stratégie de cache
- Décisions d'architecture

#### Statistiques des Colonnes

- Cardinalité et distribution
- Corrélation physique/logique
- Valeurs les plus fréquentes

**Cas d'usage** :
- Optimisation d'index
- Compréhension du modèle de données
- Validation des estimations du planificateur

#### Fragmentation

- Mesurer le bloat précis avec pgstattuple
- Analyser la fragmentation des index
- Planifier les opérations de maintenance

**Cas d'usage** :
- Optimisation d'espace disque
- Amélioration des performances
- Maintenance préventive

**📄 Document détaillé** : `requetes-analyse-postgresql.md`

---

## 4. Outils et Environnement

### 4.1. Clients PostgreSQL Recommandés

#### psql (Ligne de Commande)

**Avantages** :
- Léger et rapide
- Scriptable
- Disponible partout

**Installation** :
```bash
# Debian/Ubuntu
sudo apt-get install postgresql-client

# macOS
brew install postgresql

# Windows
# Télécharger depuis postgresql.org
```

**Configuration recommandée** (`.psqlrc`) :
```sql
-- Activer le timing des requêtes
\timing on

-- Affichage étendu automatique pour résultats larges
\x auto

-- Historique des commandes
\set HISTSIZE 10000

-- Pager automatique
\pset pager always

-- Format de date lisible
\pset null '(null)'
```

#### pgAdmin 4 (Interface Graphique)

**Avantages** :
- Interface visuelle intuitive
- Éditeur SQL avec coloration syntaxique
- Visualisation des données
- Outils de monitoring intégrés

**Téléchargement** : https://www.pgadmin.org/

#### DBeaver (Multi-plateformes)

**Avantages** :
- Support de multiples SGBD
- Éditeur puissant avec autocomplétion
- Visualisation ERD
- Export de données flexible

**Téléchargement** : https://dbeaver.io/

#### DataGrip (JetBrains - Payant)

**Avantages** :
- Autocomplétion intelligente
- Refactoring SQL
- Intégration Git
- Debugging de fonctions

**Téléchargement** : https://www.jetbrains.com/datagrip/

### 4.2. Extensions PostgreSQL Nécessaires

Certaines requêtes nécessitent des extensions :

#### pg_stat_statements (ESSENTIEL)

**Utilité** : Tracking de toutes les requêtes exécutées

```sql
-- Installation (une fois par base)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Configuration dans postgresql.conf
shared_preload_libraries = 'pg_stat_statements'  
pg_stat_statements.max = 10000  
pg_stat_statements.track = all  
```

**Nécessite** : Redémarrage de PostgreSQL

#### pgstattuple (Recommandé)

**Utilité** : Analyse précise du bloat

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
```

**Attention** : Les fonctions pgstattuple font un scan complet, utilisez hors production.

### 4.3. Permissions Requises

#### Requêtes en Lecture Seule

**Permissions minimum** :
```sql
-- Créer un utilisateur monitoring
CREATE ROLE monitoring_user LOGIN PASSWORD 'secure_password';

-- Donner accès aux vues système
GRANT CONNECT ON DATABASE ma_base TO monitoring_user;  
GRANT USAGE ON SCHEMA pg_catalog TO monitoring_user;  
GRANT SELECT ON ALL TABLES IN SCHEMA pg_catalog TO monitoring_user;  

-- Accès à pg_stat_statements
GRANT EXECUTE ON FUNCTION pg_stat_statements_reset() TO monitoring_user;
```

#### Requêtes d'Administration

**Permissions requises** :
```sql
-- Pour pg_terminate_backend(), pg_cancel_backend()
GRANT pg_signal_backend TO admin_user;

-- Pour VACUUM, ANALYZE
GRANT ma_table TO admin_user;

-- Pour REINDEX
-- Nécessite ALTER privilège ou être owner
```

### 4.4. Configuration de l'Environnement

#### Variables d'Environnement

```bash
# .bashrc ou .zshrc
export PGHOST=localhost  
export PGPORT=5432  
export PGDATABASE=ma_base  
export PGUSER=mon_user  
# Ne pas mettre PGPASSWORD, utilisez .pgpass

# Fichier .pgpass (Linux/macOS: ~/.pgpass, Windows: %APPDATA%\postgresql\pgpass.conf)
# Format: hostname:port:database:username:password
localhost:5432:ma_base:mon_user:mot_de_passe
```

**Permissions .pgpass** :
```bash
chmod 600 ~/.pgpass
```

#### Alias Utiles

```bash
# .bashrc
alias pgdev='psql -h localhost -U dev_user -d dev_db'  
alias pgprod='psql -h prod-server -U readonly_user -d prod_db'  
alias pgtop='psql -c "SELECT * FROM monitoring.database_health"'  
```

---

## 5. Bonnes Pratiques d'Utilisation

### 5.1. Avant d'Exécuter une Requête

#### Checklist de Sécurité

- ✅ **Lecture seule ?** : Requête SELECT → Sûr  
- ✅ **Environnement** : Production ou Staging ?  
- ✅ **Impact** : Requête coûteuse (pgstattuple) ?  
- ✅ **Backup** : Si modification de structure (ALTER, DROP)  
- ✅ **Permissions** : Ai-je les droits nécessaires ?

#### Test en Staging

Pour toute modification :
1. Tester en développement d'abord  
2. Valider en staging  
3. Appliquer en production avec un plan de rollback

### 5.2. Interprétation des Résultats

#### Contexte Important

Les métriques n'ont de sens que dans **votre contexte** :

- **Cache hit ratio 85%** :
  - Peut être acceptable pour un data warehouse (OLAP)
  - Problématique pour une API transactionnelle (OLTP)

- **Table de 500 GB** :
  - Normal pour des logs
  - Énorme pour une table de configuration

#### Tendances > Valeurs Absolues

Une métrique isolée dit peu de choses. Surveillez les **tendances** :

```
Jour 1 : Cache hit ratio = 97%  
Jour 2 : Cache hit ratio = 97%  
Jour 3 : Cache hit ratio = 95%  
Jour 4 : Cache hit ratio = 92%  ← Tendance à la baisse, enquêter !  
```

#### Corrélation

Cherchez les **corrélations** entre métriques :

- Bloat élevé + Cache hit ratio faible → VACUUM nécessaire
- Slow queries + Locks élevés → Contention
- Croissance rapide + Disk full → Archivage urgent

### 5.3. Documentation de Vos Actions

**Tenir un journal** des requêtes exécutées et de leurs effets :

```markdown
## Journal d'Administration PostgreSQL

### 2025-11-21 14:30 - Diagnostic de Lenteur

**Problème** : Application lente depuis 13h00  
**Requêtes exécutées** :  
1. Cache hit ratio global → 97% (OK)
2. Slow queries top 20 → Requête sur orders identifiée (avg: 2.5s)
3. EXPLAIN ANALYZE sur la requête lente → Seq Scan détecté

**Action prise** :
```sql
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
```

**Résultat** : Requête passée de 2.5s à 45ms (55× plus rapide)  
**Status** : ✅ Résolu  
```

### 5.4. Automatisation et Alerting

#### Scripts de Monitoring Automatisés

```bash
#!/bin/bash
# check_database_health.sh

# Configuration
PGDATABASE="ma_base"  
ALERT_EMAIL="admin@example.com"  
CACHE_HIT_THRESHOLD=95  

# Vérifier cache hit ratio
CACHE_HIT=$(psql $PGDATABASE -t -c "
    SELECT round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2)
    FROM pg_stat_database
    WHERE datname = current_database();
")

if (( $(echo "$CACHE_HIT < $CACHE_HIT_THRESHOLD" | bc -l) )); then
    echo "ALERTE : Cache hit ratio = $CACHE_HIT% (seuil: $CACHE_HIT_THRESHOLD%)" | \
        mail -s "PostgreSQL Alert" $ALERT_EMAIL
fi
```

#### Planification avec Cron

```bash
# crontab -e

# Tous les jours à 2h du matin
0 2 * * * /path/to/check_database_health.sh

# Toutes les heures
0 * * * * /path/to/check_slow_queries.sh

# Tous les lundis à 3h
0 3 * * 1 /path/to/weekly_bloat_check.sh
```

### 5.5. Versionning de Vos Requêtes

Utilisez Git pour versionner vos scripts SQL :

```bash
# Structure de répertoire
sql-library/
├── README.md
├── admin/
│   ├── 01_locks.sql
│   ├── 02_bloat.sql
│   └── 03_index_usage.sql
├── monitoring/
│   ├── 01_cache_hit_ratio.sql
│   └── 02_slow_queries.sql
└── analysis/
    └── 01_table_sizes.sql

# Git
git init  
git add .  
git commit -m "Initial SQL library"  
```

**Avantages** :
- Historique des modifications
- Travail collaboratif
- Rollback facile

---

## 6. Index des Requêtes Disponibles

### 6.1. Requêtes d'Administration

📄 **Document complet** : `requetes-administration-postgresql.md`

#### Gestion des Locks

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Locks actifs | Liste tous les verrous en cours | À la demande |
| Transactions bloquantes | Identifie qui bloque qui | À la demande |
| Deadlocks en cours | Détecte les interblocages | À la demande |

#### Détection du Bloat

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Bloat par table | Estime le % de bloat | Hebdomadaire |
| Bloat précis (pgstattuple) | Mesure exacte | Mensuel |
| Bloat des index | Fragmentation des index | Hebdomadaire |

#### Utilisation des Index

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Index inutilisés | idx_scan = 0 | Mensuel |
| Index redondants | Doublons détectés | Mensuel |
| Taux d'utilisation | Ratio index vs seq scan | Hebdomadaire |

### 6.2. Requêtes de Monitoring

📄 **Document complet** : `requetes-monitoring-postgresql.md`

#### Cache Hit Ratio

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Cache hit ratio global | Par base de données | Quotidien |
| Cache hit ratio par table | Identification des tables | Hebdomadaire |
| Cache hit ratio par index | Performance des index | Hebdomadaire |

#### Slow Queries

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Top 20 slow queries (temps total) | Cumul le plus élevé | Quotidien |
| Top 20 slow queries (temps moyen) | Temps moyen le plus élevé | Quotidien |
| Requêtes en cours | Temps réel | À la demande |
| Requêtes avec I/O élevé | Lectures disque | Hebdomadaire |

#### Métriques Complémentaires

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Connexions actives | État des connexions | Quotidien |
| Taille des tables | Top 20 | Quotidien |
| Checkpoints | Fréquence et timing | Hebdomadaire |
| XID wraparound risk | Risque de wraparound | Hebdomadaire |
| Réplication lag | Retard des replicas | Quotidien (si réplication) |

### 6.3. Requêtes d'Analyse

📄 **Document complet** : `requetes-analyse-postgresql.md`

#### Taille des Objets

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Taille de toutes les bases | Vue d'ensemble | Hebdomadaire |
| Top 20 tables par taille | Les plus volumineuses | Quotidien |
| Ratio table/index | Distribution de taille | Hebdomadaire |
| Croissance dans le temps | Tracking historique | Quotidien |
| Taille par schéma | Si multi-schémas | Hebdomadaire |

#### Statistiques des Tables

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Vue d'ensemble statistiques | n_live_tup, n_dead_tup | Quotidien |
| Tables les plus actives (lecture) | seq_scan, idx_scan | Hebdomadaire |
| Tables les plus actives (écriture) | INSERT, UPDATE, DELETE | Hebdomadaire |
| Statistiques I/O | Cache vs disque | Hebdomadaire |

#### Statistiques des Colonnes

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Cardinalité des colonnes | Valeurs distinctes | Mensuel |
| Distribution MCV | Valeurs les plus fréquentes | Mensuel |
| Corrélation physique | Ordre des données | Mensuel |
| Colonnes volumineuses | avg_width élevé | Mensuel |

#### Analyse de la Fragmentation

| Requête | Description | Fréquence |
|---------|-------------|-----------|
| Fragmentation (pgstattuple) | Mesure précise | Mensuel |
| Estimation bloat rapide | Via pg_stat | Hebdomadaire |
| Fragmentation index | pgstatindex | Mensuel |

---

## 7. Prochaines Étapes

### 7.1. Par Où Commencer ?

Si vous débutez avec PostgreSQL, voici un **parcours recommandé** :

#### Semaine 1 : Familiarisation

1. Installer un client SQL (psql ou pgAdmin)  
2. Se connecter à votre base  
3. Exécuter les requêtes de base :
   - Taille de la base
   - Liste des tables
   - Cache hit ratio global

#### Semaine 2-3 : Monitoring

1. Installer pg_stat_statements  
2. Explorer les slow queries  
3. Mettre en place un suivi quotidien du cache hit ratio  
4. Créer vos premières vues de monitoring

#### Semaine 4 : Administration

1. Comprendre les locks  
2. Analyser le bloat de vos tables principales  
3. Auditer l'utilisation des index  
4. Planifier votre première maintenance (VACUUM, ANALYZE)

#### Mois 2 : Automatisation

1. Créer des scripts de monitoring  
2. Planifier des vérifications automatiques (cron)  
3. Mettre en place des alertes  
4. Documenter vos processus

### 7.2. Ressources Complémentaires

#### Documentation PostgreSQL Officielle

- [Monitoring](https://www.postgresql.org/docs/18/monitoring.html)  
- [System Catalogs](https://www.postgresql.org/docs/18/catalogs.html)  
- [Statistics](https://www.postgresql.org/docs/18/planner-stats.html)

#### Livres Recommandés

- **PostgreSQL: Up and Running** (Regina Obe, Leo Hsu)  
- **The Art of PostgreSQL** (Dimitri Fontaine)  
- **Mastering PostgreSQL** (Hans-Jürgen Schönig)

#### Blogs et Sites Web

- [PostgreSQL Wiki](https://wiki.postgresql.org/)  
- [Planet PostgreSQL](https://planet.postgresql.org/) - Agrégateur de blogs  
- [Percona PostgreSQL Blog](https://www.percona.com/blog/category/postgresql/)  
- [2ndQuadrant Blog](https://www.2ndquadrant.com/en/blog/)

#### Communautés

- Reddit : [r/PostgreSQL](https://www.reddit.com/r/PostgreSQL/)
- Discord : Communauté PostgreSQL francophone
- Stack Overflow : Tag `postgresql`
- Mailing lists : pgsql-general@postgresql.org

### 7.3. Contribuer à Cette Annexe

Cette annexe est un **document vivant**. Vos contributions sont les bienvenues :

- **Nouvelles requêtes** : Partagez vos trouvailles  
- **Améliorations** : Optimisations, clarifications  
- **Corrections** : Erreurs, typos  
- **Cas d'usage** : Exemples réels

---

## Résumé

### Points Clés

- ✅ **Bibliothèque prête à l'emploi** : Requêtes testées et documentées  
- ✅ **Trois catégories** : Administration, Monitoring, Analyse  
- ✅ **Personnalisables** : Adaptez à votre contexte  
- ✅ **Documents détaillés** : 3 fichiers spécialisés disponibles  
- ✅ **Bonnes pratiques** : Sécurité, documentation, automatisation

### Structure des Documents

```
Annexe C : Requêtes SQL de Référence
│
├── 📄 requetes-sql-reference-introduction.md (ce fichier)
│   └── Vue d'ensemble et guide d'utilisation
│
├── 📄 requetes-administration-postgresql.md
│   ├── Locks (Verrous)
│   ├── Bloat (Gonflement)
│   └── Index Usage
│
├── 📄 requetes-monitoring-postgresql.md
│   ├── Cache Hit Ratio
│   ├── Slow Queries
│   └── Métriques Complémentaires
│
└── 📄 requetes-analyse-postgresql.md
    ├── Taille des Objets
    ├── Statistiques des Tables
    ├── Statistiques des Colonnes
    └── Analyse de la Fragmentation
```

### Prochaine Lecture

📖 Consultez maintenant les documents détaillés selon vos besoins :

- **Problème immédiat** (app figée, lenteur) → `requetes-administration-postgresql.md`  
- **Suivi de santé quotidien** → `requetes-monitoring-postgresql.md`  
- **Planification et croissance** → `requetes-analyse-postgresql.md`

---


⏭️ [Requêtes d'administration (locks, bloat, index usage)](/annexes/requetes-sql-reference/01-requetes-administration.md)
