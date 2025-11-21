🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.2. Vues système essentielles : Introduction

## Qu'est-ce qu'une vue système ?

Dans PostgreSQL, les **vues système** sont des vues SQL spéciales créées et maintenues automatiquement par PostgreSQL lui-même. Elles vous donnent une fenêtre en temps réel sur l'état interne de votre base de données : ce qui se passe actuellement, les performances, la santé générale, et la structure de vos objets.

Imaginez ces vues comme les instruments de bord d'une voiture : le compteur de vitesse, la jauge d'essence, le témoin de température du moteur, etc. Elles vous permettent de surveiller ce qui se passe "sous le capot" de PostgreSQL sans avoir à comprendre tous les détails de son fonctionnement interne.

**En d'autres termes :** Les vues système transforment des informations internes complexes de PostgreSQL en données SQL facilement interrogeables.

## Pourquoi les vues système sont essentielles ?

Sans les vues système, administrer et optimiser PostgreSQL serait extrêmement difficile. Elles vous permettent de :

### 1. **Diagnostiquer les problèmes en temps réel**
- "Pourquoi ma requête est-elle si lente ?"
- "Qui bloque qui dans ma base de données ?"
- "Combien de connexions actives ai-je ?"
- "Y a-t-il des transactions zombies ?"

### 2. **Optimiser les performances**
- Identifier les requêtes les plus coûteuses
- Trouver les tables qui manquent d'index
- Détecter les tables fragmentées nécessitant un VACUUM
- Mesurer le taux de succès du cache (cache hit ratio)

### 3. **Surveiller la santé de la base**
- Suivre la consommation de ressources (CPU, I/O, mémoire)
- Détecter les anomalies avant qu'elles ne deviennent critiques
- Vérifier que les maintenances automatiques (autovacuum, autoanalyze) fonctionnent
- Identifier les corruptions potentielles de données

### 4. **Planifier la capacité**
- Estimer la croissance des tables
- Anticiper les besoins en ressources
- Identifier les goulots d'étranglement
- Planifier les migrations et upgrades

### 5. **Comprendre la structure**
- Explorer toutes les tables, colonnes, index
- Analyser les dépendances entre objets
- Auditer les permissions et la sécurité
- Générer de la documentation automatiquement

## Les catégories de vues système

PostgreSQL organise ses vues système en plusieurs catégories selon leur fonction :

### Vues de statistiques d'activité (`pg_stat_*`)

Ces vues collectent et exposent des statistiques sur l'activité de la base de données. Elles sont cumulatives depuis le démarrage du serveur (ou la dernière réinitialisation).

**Exemples :**
- `pg_stat_activity` : Activité en cours (connexions, requêtes actives)
- `pg_stat_database` : Statistiques par base de données
- `pg_stat_user_tables` : Statistiques par table utilisateur
- `pg_stat_user_indexes` : Statistiques par index utilisateur

### Vues de statistiques I/O (`pg_statio_*`)

Ces vues se concentrent spécifiquement sur les opérations d'entrée/sortie (lectures/écritures disque vs cache).

**Exemples :**
- `pg_statio_user_tables` : Stats I/O par table
- `pg_statio_user_indexes` : Stats I/O par index

### Vues de verrous (`pg_locks`)

Cette vue unique montre tous les verrous (locks) actuellement actifs ou en attente dans le système. Cruciale pour diagnostiquer les blocages.

### Vues de catalogue système (`pg_catalog`)

Le catalogue système contient toutes les métadonnées sur la structure de votre base de données : tables, colonnes, contraintes, types, fonctions, etc.

**Exemples :**
- `pg_class` : Tables, index, séquences, vues
- `pg_attribute` : Colonnes
- `pg_constraint` : Contraintes (PK, FK, CHECK)
- `pg_index` : Index

### Vues simplifiées

PostgreSQL fournit également des vues "user-friendly" qui simplifient l'accès aux informations :

**Exemples :**
- `pg_tables` : Liste simple des tables
- `pg_indexes` : Liste simple des index
- `pg_views` : Liste des vues
- `pg_roles` : Utilisateurs et rôles

## Vue d'ensemble des vues que nous allons étudier

Dans ce chapitre, nous allons explorer en détail les cinq vues système les plus importantes pour l'observabilité et l'administration de PostgreSQL :

### 14.2.1. pg_stat_activity : Activité en cours

**Ce que vous y trouverez :**
- Toutes les connexions actuellement actives
- Les requêtes en cours d'exécution
- L'état de chaque session (active, idle, idle in transaction)
- Les temps d'exécution
- Les événements d'attente (si une session est bloquée)

**Cas d'usage typiques :**
- "Ma base est lente, que se passe-t-il ?"
- "Combien de connexions ai-je ?"
- "Qui exécute cette longue requête ?"
- "Y a-t-il des transactions qui ne se terminent jamais ?"

**Métaphore :** `pg_stat_activity` est comme un panneau d'affichage en temps réel dans un aéroport qui montre tous les vols en cours.

### 14.2.2. pg_stat_database : Métriques par base

**Ce que vous y trouverez :**
- Statistiques cumulatives par base de données
- Nombre de transactions (commits, rollbacks)
- Cache hit ratio (taux de succès du cache)
- Nombre de lignes lues, insérées, modifiées, supprimées
- Deadlocks détectés
- Fichiers temporaires créés

**Cas d'usage typiques :**
- "Quelle base consomme le plus de ressources ?"
- "Mon cache est-il bien dimensionné ?"
- "Y a-t-il beaucoup de rollbacks ?"
- "Combien de transactions par seconde ?"

**Métaphore :** `pg_stat_database` est comme le tableau de bord de votre voiture qui montre le kilométrage total, la consommation moyenne, etc.

### 14.2.3. pg_stat_user_tables : Statistiques de tables

**Ce que vous y trouverez :**
- Statistiques détaillées par table
- Scans séquentiels vs scans d'index
- Nombre de lignes vivantes vs mortes (bloat)
- Informations sur VACUUM et ANALYZE
- HOT updates (optimisation des mises à jour)

**Cas d'usage typiques :**
- "Quelle table manque d'index ?"
- "Quelle table a besoin d'un VACUUM ?"
- "Mes statistiques sont-elles à jour ?"
- "Y a-t-il beaucoup de lignes mortes ?"

**Métaphore :** `pg_stat_user_tables` est comme un rapport de santé individuel pour chaque table, montrant son état et ses besoins de maintenance.

### 14.2.4. pg_locks : Verrous actifs

**Ce que vous y trouverez :**
- Tous les verrous (locks) actuellement posés
- Qui détient quel verrou
- Qui attend quel verrou (blocages)
- Types de verrous (lecture, écriture, exclusif)
- Ressources verrouillées (tables, lignes, transactions)

**Cas d'usage typiques :**
- "Pourquoi ma requête attend-elle ?"
- "Qui bloque qui ?"
- "Y a-t-il un deadlock ?"
- "Quelle transaction monopolise cette table ?"

**Métaphore :** `pg_locks` est comme un registre montrant qui utilise quelle ressource et qui attend son tour.

### 14.2.5. pg_catalog : Métadonnées système

**Ce que vous y trouverez :**
- Toute la structure de la base de données
- Définition de toutes les tables, colonnes, types
- Contraintes (PK, FK, CHECK, UNIQUE)
- Index, fonctions, triggers, vues
- Dépendances entre objets
- Permissions et propriétaires

**Cas d'usage typiques :**
- "Quelle est la structure de cette table ?"
- "Quelles vues dépendent de cette table ?"
- "Quels sont tous les index de cette table ?"
- "Quelles tables n'ont pas de clé primaire ?"

**Métaphore :** `pg_catalog` est comme le plan architectural complet de votre base de données, documentant chaque élément et leurs relations.

## Comment ces vues fonctionnent ensemble

Ces vues ne sont pas isolées : elles sont complémentaires et souvent utilisées ensemble pour obtenir une vision complète.

**Exemple de workflow de diagnostic :**

1. **Problème détecté :** "La base est lente"

2. **Étape 1 - pg_stat_activity :**
   - Identifier les requêtes longues en cours
   - Vérifier s'il y a des blocages

3. **Étape 2 - pg_locks :**
   - Si des blocages existent, identifier qui bloque qui
   - Décider de tuer la transaction bloquante ou d'attendre

4. **Étape 3 - pg_stat_user_tables :**
   - Vérifier si les tables interrogées ont des index appropriés
   - Vérifier si elles ont besoin de VACUUM/ANALYZE

5. **Étape 4 - pg_stat_database :**
   - Vérifier le cache hit ratio global
   - Vérifier s'il y a beaucoup de fichiers temporaires (manque de mémoire)

6. **Étape 5 - pg_catalog :**
   - Examiner la structure des tables problématiques
   - Vérifier les index existants et leur pertinence

## Prérequis et permissions

### Qui peut voir quoi ?

Les vues système ont des règles de visibilité :

**Superutilisateurs :**
- Peuvent voir TOUTES les informations de TOUTES les sessions/objets

**Utilisateurs normaux :**
- `pg_stat_activity` : Voient uniquement leurs propres sessions
- `pg_stat_database` : Voient les statistiques de leur base actuelle
- `pg_stat_user_tables` : Voient les statistiques de leurs propres tables
- `pg_locks` : Voient uniquement leurs propres verrous
- `pg_catalog` : Voient uniquement les objets qu'ils possèdent ou auxquels ils ont accès

**Rôle pg_read_all_stats :**

PostgreSQL 10+ a introduit un rôle spécial qui permet à un utilisateur non-superutilisateur de voir toutes les statistiques :

```sql
-- Donner accès complet aux statistiques
GRANT pg_read_all_stats TO mon_utilisateur;
```

### Configuration requise

Certaines statistiques nécessitent une configuration spécifique dans `postgresql.conf` :

```ini
# Suivi de l'activité (généralement activé par défaut)
track_activities = on
track_counts = on

# Suivi des temps I/O (à activer pour pg_stat_database)
track_io_timing = on

# Suivi des fonctions (optionnel)
track_functions = all
```

Après modification du fichier de configuration :
```sql
SELECT pg_reload_conf();  -- Recharge sans redémarrage
```

## Concepts de base à comprendre

### 1. Statistiques cumulatives vs instantanées

**Statistiques cumulatives :**
- Accumulées depuis le démarrage du serveur (ou dernier reset)
- Exemples : `pg_stat_database`, `pg_stat_user_tables`
- Utiles pour les tendances et l'analyse historique

**Vues instantanées :**
- État actuel à l'instant T
- Exemples : `pg_stat_activity`, `pg_locks`
- Utiles pour le diagnostic en temps réel

### 2. Réinitialisation des statistiques

Les statistiques cumulatives peuvent être réinitialisées :

```sql
-- Réinitialiser les stats de la base actuelle
SELECT pg_stat_reset();

-- Réinitialiser les stats d'une fonction spécifique
SELECT pg_stat_reset_single_function_counters(func_oid);
```

**⚠️ Attention :** La réinitialisation supprime l'historique. Faites-le rarement et de manière réfléchie.

### 3. Fréquence de rafraîchissement

Les vues sont mises à jour en temps réel (ou presque) :
- `pg_stat_activity` : Mise à jour instantanée
- `pg_locks` : Mise à jour instantanée
- `pg_stat_database`, `pg_stat_user_tables` : Mise à jour toutes les quelques secondes

### 4. Impact sur les performances

Interroger les vues système a un coût, mais il est généralement négligeable :

- **Impact faible :** `pg_stat_activity`, `pg_stat_database`
- **Impact modéré :** `pg_stat_user_tables` (si des milliers de tables)
- **Impact potentiellement élevé :** Requêtes complexes sur `pg_catalog` avec beaucoup de jointures

**Recommandation :** Sur des systèmes très chargés, évitez d'interroger ces vues en boucle rapide (< 1 seconde). Privilégiez un monitoring toutes les 5-10 secondes.

## Outils qui utilisent ces vues

De nombreux outils d'administration PostgreSQL reposent sur ces vues système :

### 1. Outils en ligne de commande
- **psql** : Commandes méta (\\d, \\dt, \\di) interrogent `pg_catalog`
- **pg_top** : Monitore `pg_stat_activity` en temps réel

### 2. Interfaces graphiques
- **pgAdmin** : Dashboards basés sur `pg_stat_*`
- **DBeaver** : Moniteur d'activité utilisant `pg_stat_activity`
- **DataGrip** : Utilise `pg_catalog` pour l'autocomplétion

### 3. Monitoring et observabilité
- **Prometheus + postgres_exporter** : Exporte toutes les métriques `pg_stat_*`
- **Grafana** : Visualise les métriques exportées
- **pgBadger** : Analyse les logs ET les statistiques
- **Datadog, New Relic** : Collectent automatiquement ces métriques

### 4. Solutions cloud
- **AWS RDS Performance Insights** : Basé sur `pg_stat_activity` et `pg_stat_statements`
- **Azure Database Insights** : Utilise les vues système
- **Google Cloud SQL Insights** : Idem

Tous ces outils ne font que formuler des requêtes SQL sur les vues que vous allez apprendre à utiliser directement !

## Approche pédagogique de ce chapitre

Dans les sections suivantes, pour chaque vue système, nous suivrons cette structure :

### 1. Introduction et concepts
- Qu'est-ce que cette vue ?
- Pourquoi est-elle importante ?
- Quand l'utiliser ?

### 2. Colonnes principales
- Détail de chaque colonne importante
- Interprétation des valeurs
- Exemples concrets

### 3. Requêtes pratiques
- Requêtes prêtes à l'emploi
- Cas d'usage réels
- Combinaisons avec d'autres vues

### 4. Cas d'usage en production
- Scénarios réels de diagnostic
- Problèmes courants et solutions
- Workflow d'investigation

### 5. Monitoring et alertes
- Métriques critiques à surveiller
- Seuils d'alerte recommandés
- Intégration avec les outils de monitoring

### 6. Bonnes pratiques
- Ce qu'il faut faire et ne pas faire
- Optimisations
- Pièges courants à éviter

## Progression recommandée

Si vous êtes débutant, voici l'ordre recommandé pour étudier ces vues :

### Niveau 1 : Essentiel (début)
1. **pg_stat_activity** : Comprendre ce qui se passe en ce moment
2. **pg_catalog** (bases) : Lister tables et colonnes
3. **pg_stat_database** : Vue d'ensemble de la santé

### Niveau 2 : Intermédiaire
4. **pg_stat_user_tables** : Optimisation et maintenance
5. **pg_catalog** (avancé) : Contraintes, index, dépendances

### Niveau 3 : Avancé
6. **pg_locks** : Diagnostic des blocages et deadlocks

Cette progression suit une logique de complexité croissante et de fréquence d'utilisation.

## Combinaisons puissantes de vues

Certaines requêtes nécessitent de combiner plusieurs vues pour obtenir une information complète :

### Exemple 1 : Activité + Blocages
```sql
-- Identifier qui bloque qui avec les requêtes
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocker.pid AS blocker_pid,
    blocker.query AS blocker_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocker_locks ON blocked_locks.locktype = blocker_locks.locktype
    AND blocked_locks.database IS NOT DISTINCT FROM blocker_locks.database
    AND blocked_locks.relation IS NOT DISTINCT FROM blocker_locks.relation
    -- ... (autres conditions de jointure)
JOIN pg_stat_activity blocker ON blocker_locks.pid = blocker.pid
WHERE NOT blocked_locks.granted;
```

### Exemple 2 : Statistiques + Structure
```sql
-- Tables les plus actives avec leur taille
SELECT
    s.relname AS table_name,
    s.seq_scan + s.idx_scan AS total_scans,
    s.n_tup_ins + s.n_tup_upd + s.n_tup_del AS total_modifications,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS taille
FROM pg_stat_user_tables s
JOIN pg_class c ON s.relname = c.relname
ORDER BY total_scans DESC
LIMIT 20;
```

### Exemple 3 : Activité + Base + Tables
```sql
-- Vue complète de l'activité sur une table spécifique
SELECT
    a.pid,
    a.usename,
    a.state,
    a.query,
    l.mode AS lock_mode,
    s.n_live_tup,
    s.n_dead_tup
FROM pg_stat_activity a
LEFT JOIN pg_locks l ON a.pid = l.pid AND l.relation = 'ma_table'::regclass
LEFT JOIN pg_stat_user_tables s ON s.relname = 'ma_table'
WHERE a.query LIKE '%ma_table%'
   OR l.relation IS NOT NULL;
```

## Ressources et documentation

Pour approfondir votre compréhension des vues système :

### Documentation officielle PostgreSQL
- [Monitoring Database Activity](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [System Catalogs](https://www.postgresql.org/docs/current/catalogs.html)
- [System Views](https://www.postgresql.org/docs/current/views-overview.html)

### Commandes psql utiles
```sql
-- Lister toutes les vues système
\dv pg_catalog.*

-- Décrire une vue spécifique
\d+ pg_stat_activity

-- Lister toutes les fonctions système
\df pg_catalog.*
```

### Livres recommandés
- **PostgreSQL: Up and Running** (O'Reilly) : Chapitre sur le monitoring
- **The Art of PostgreSQL** (Dimitri Fontaine) : Utilisation avancée du catalogue

### Blogs et ressources en ligne
- Blog officiel PostgreSQL
- 2ndQuadrant blog (maintenant EDB)
- Percona Database Performance Blog
- CrunchyData blog

## Préparation pour les sections suivantes

Avant de plonger dans les détails de chaque vue, assurez-vous d'avoir :

### 1. Accès à une instance PostgreSQL
- Version 12+ recommandée (idéalement PostgreSQL 18)
- Droits suffisants (superuser ou pg_read_all_stats)
- Possibilité d'exécuter des requêtes

### 2. Un client SQL
- **psql** (ligne de commande)
- **pgAdmin** (interface graphique)
- **DBeaver** (multi-plateforme)
- Ou tout autre client de votre choix

### 3. Configuration recommandée
```ini
# Dans postgresql.conf
track_activities = on
track_counts = on
track_io_timing = on
track_functions = all
```

### 4. Quelques données de test
Pour mieux comprendre les vues, il est utile d'avoir :
- Quelques tables avec des données
- Des requêtes en cours d'exécution
- Un peu d'activité dans la base

Vous pouvez créer une base de test simple :
```sql
CREATE DATABASE test_monitoring;
\c test_monitoring

CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT now()
);

INSERT INTO test_table (data)
SELECT 'test_' || i
FROM generate_series(1, 10000) AS i;

CREATE INDEX idx_data ON test_table(data);
```

## Terminologie importante

Avant de continuer, familiarisez-vous avec ces termes clés :

- **Backend** : Un processus PostgreSQL gérant une connexion client
- **PID** : Process ID, identifiant unique d'un processus
- **OID** : Object Identifier, identifiant unique d'un objet dans le catalogue
- **Transaction** : Unité atomique de travail (BEGIN...COMMIT/ROLLBACK)
- **VACUUM** : Opération de nettoyage des lignes mortes
- **ANALYZE** : Collecte de statistiques pour le planificateur
- **Cache Hit Ratio** : Pourcentage de lectures satisfaites par le cache plutôt que le disque
- **Bloat** : Espace gaspillé par des lignes mortes non VACUUMées
- **Lock** : Verrou posé sur une ressource
- **Deadlock** : Blocage circulaire entre transactions

## Conclusion de l'introduction

Les vues système sont vos yeux et oreilles dans PostgreSQL. Elles transforment un système complexe en informations exploitables. Sans elles, l'administration et l'optimisation de PostgreSQL seraient comme piloter un avion sans instruments de bord.

Dans les sections suivantes, vous allez apprendre à :
- ✅ Diagnostiquer les problèmes de performance en quelques requêtes
- ✅ Identifier les goulots d'étranglement avant qu'ils ne deviennent critiques
- ✅ Optimiser vos bases de données avec des données factuelles
- ✅ Automatiser la surveillance et la maintenance
- ✅ Comprendre en profondeur comment PostgreSQL fonctionne

**Chaque vue système est un outil puissant.** Maîtrisez-les, et vous maîtriserez PostgreSQL.

---

**Prêt à commencer ?** Passons maintenant à la première vue essentielle : `pg_stat_activity`, qui vous montrera en temps réel tout ce qui se passe dans votre base de données.

⏭️ [pg_stat_activity : Activité en cours](/14-observabilite-et-monitoring/02.1-pg-stat-activity.md)
