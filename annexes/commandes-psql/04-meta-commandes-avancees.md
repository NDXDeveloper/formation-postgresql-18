🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Commandes psql Essentielles - Méta-commandes Avancées

## Introduction

Vous maîtrisez maintenant les bases de psql (navigation, configuration, export/import). Il est temps de découvrir des commandes plus avancées qui vous feront gagner énormément de temps et de productivité.

Ces méta-commandes sont souvent méconnues, même par des utilisateurs expérimentés, mais elles peuvent transformer votre façon de travailler avec PostgreSQL.

### Qu'allez-vous apprendre ?

- 🔄 **Exécuter des requêtes répétées** automatiquement  
- ✏️ **Éditer du code SQL** directement dans votre éditeur favori  
- 🔍 **Visualiser le code source** de fonctions et vues  
- 📊 **Manipuler les résultats** de requêtes de manière créative  
- 🕒 **Gérer l'historique** efficacement  
- 🛠️ **Automatiser** des tâches complexes

---

## Exécution Répétée : \watch

### Qu'est-ce que \watch ?

`\watch` exécute la dernière requête de manière répétée, à intervalles réguliers. C'est l'équivalent d'une boucle automatique, parfait pour **surveiller** l'évolution de données en temps réel.

### Syntaxe

```
\watch [SECONDS]
```

**Par défaut** : 2 secondes si aucun intervalle n'est spécifié.

---

### Exemples de Base

#### Surveiller une table en temps réel

```sql
-- Surveiller le nombre de connexions actives
SELECT COUNT(*) as connexions_actives  
FROM pg_stat_activity  
WHERE state = 'active';  

\watch

# Résultat : La requête s'exécute toutes les 2 secondes
# Thu Nov 21 14:23:45 2024 (every 2s)
#  connexions_actives
# --------------------
#                   3
# (1 row)
#
# Thu Nov 21 14:23:47 2024 (every 2s)
#  connexions_actives
# --------------------
#                   5
# (1 row)
```

**Arrêter** : Appuyez sur `Ctrl+C` ou `q`

---

#### Avec intervalle personnalisé

```sql
-- Surveiller toutes les 5 secondes
SELECT NOW() as heure, COUNT(*) as nb_clients  
FROM clients;  

\watch 5
```

---

#### Surveiller une requête complexe

```sql
-- Surveiller les requêtes lentes en cours
SELECT
    pid,
    usename,
    LEFT(query, 50) as query_preview,
    NOW() - query_start as duree
FROM pg_stat_activity  
WHERE state = 'active'  
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duree DESC;

\watch 3
```

---

### Cas d'Usage Pratiques

#### 1. Surveiller une insertion en masse

```sql
-- Terminal 1 : Lancer l'insertion
INSERT INTO logs  
SELECT generate_series(1, 1000000), 'message', NOW();  

-- Terminal 2 : Surveiller la progression
SELECT
    schemaname,
    tablename,
    n_tup_ins as insertions,
    n_tup_upd as updates
FROM pg_stat_user_tables  
WHERE tablename = 'logs';  

\watch 1
```

---

#### 2. Surveiller les locks (verrous)

```sql
-- Voir les verrous actifs en temps réel
SELECT
    pid,
    usename,
    pg_blocking_pids(pid) as blocking_pids,
    query
FROM pg_stat_activity  
WHERE cardinality(pg_blocking_pids(pid)) > 0;  

\watch 2
```

---

#### 3. Surveiller la réplication

```sql
-- État de la réplication (sur le primary)
SELECT
    client_addr,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) as send_lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as replay_lag
FROM pg_stat_replication;

\watch 5
```

---

#### 4. Surveiller l'utilisation du cache

```sql
-- Cache hit ratio en temps réel
SELECT
    SUM(heap_blks_read) as disk_reads,
    SUM(heap_blks_hit) as cache_hits,
    ROUND(
        SUM(heap_blks_hit)::numeric /
        NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0) * 100,
        2
    ) as cache_hit_ratio
FROM pg_statio_user_tables;

\watch 3
```

---

### Astuce avec \watch

Combinez `\watch` avec `\x auto` pour un affichage optimal :

```sql
\x auto
\pset title 'Monitoring des Connexions Actives'

SELECT * FROM pg_stat_activity WHERE state = 'active';
\watch 2
```

---

## Variantes d'Exécution : \gx, \gset, \gexec

### \gx - Exécuter en Mode Étendu

**Syntaxe** : `\gx`

**Description** : Exécute la requête en cours avec l'affichage étendu (`\x`), **sans modifier** le paramètre global.

**Usage** : Quand vous voulez voir **une seule requête** en mode étendu sans activer `\x` pour toutes les suivantes.

**Exemples** :

```sql
-- Sans \gx (affichage normal)
SELECT * FROM clients WHERE id = 1;

 id |  nom   | prenom |        email         | ville
----+--------+--------+----------------------+-------
  1 | Dupont | Alice  | alice.dupont@mail.fr | Paris

-- Avec \gx (affichage étendu pour cette requête uniquement)
SELECT * FROM clients WHERE id = 1 \gx

-[ RECORD 1 ]----------------
id     | 1  
nom    | Dupont  
prenom | Alice  
email  | alice.dupont@mail.fr  
ville  | Paris  

-- La requête suivante revient à l'affichage normal
SELECT * FROM clients WHERE id = 2;

 id |  nom   | prenom |       email       | ville
----+--------+--------+-------------------+-------
  2 | Martin | Bob    | bob.martin@mail.fr| Lyon
```

**Astuce** 💡 : Très pratique pour inspecter un enregistrement spécifique sans changer votre configuration globale.

---

### \gset - Stocker les Résultats dans des Variables

**Syntaxe** : `\gset [PREFIX]`

**Description** : Exécute la requête et stocke chaque colonne du résultat dans une variable psql.

**Important** : La requête doit retourner **exactement une ligne**.

**Exemples** :

#### Utilisation de base

```sql
-- Récupérer des informations sur la base
SELECT
    current_database() as db,
    current_user as user,
    version() as version
\gset

-- Maintenant vous avez des variables :
\echo 'Base de données :' :db
\echo 'Utilisateur :' :user
\echo 'Version :' :version

# Résultat :
# Base de données : ma_boutique
# Utilisateur : postgres
# Version : PostgreSQL 18.0...
```

---

#### Avec préfixe

```sql
-- Statistiques d'une table
SELECT
    n_tup_ins as insertions,
    n_tup_upd as updates,
    n_tup_del as deletions
FROM pg_stat_user_tables  
WHERE tablename = 'clients'  
\gset stats_

-- Variables créées : stats_insertions, stats_updates, stats_deletions
\echo 'Insertions :' :stats_insertions
\echo 'Updates :' :stats_updates
\echo 'Deletions :' :stats_deletions
```

---

#### Cas d'usage : Scripts dynamiques

```sql
-- Récupérer l'ID maximum
SELECT MAX(id) as max_id FROM clients \gset

-- Utiliser cette valeur dans une autre requête
SELECT * FROM clients WHERE id > :max_id - 10;

-- Ou pour créer un backup avec l'ID max dans le nom
\! echo "Backup jusqu'à l'ID :max_id" > /tmp/backup_info.txt
```

---

#### Cas d'usage : Calculs intermédiaires

```sql
-- Calculer une statistique
SELECT COUNT(*) * 0.1 as sample_size  
FROM huge_table  
\gset

-- Utiliser pour un échantillon
SELECT * FROM huge_table LIMIT :sample_size;
```

---

### \gexec - Exécuter les Résultats comme SQL

**Syntaxe** : `\gexec`

**Description** : Exécute la requête, puis **exécute chaque ligne du résultat** comme une nouvelle commande SQL.

**Attention** ⚠️ : Puissant mais dangereux si mal utilisé !

**Exemples** :

#### Générer et exécuter des commandes

```sql
-- Générer des commandes VACUUM pour toutes les tables
SELECT 'VACUUM ANALYZE ' || tablename || ';'  
FROM pg_tables  
WHERE schemaname = 'public'  
\gexec

# Résultat : Exécute
# VACUUM ANALYZE clients;
# VACUUM ANALYZE commandes;
# VACUUM ANALYZE produits;
# ...
```

---

#### Créer plusieurs index en une fois

```sql
-- Générer des commandes CREATE INDEX
SELECT
    'CREATE INDEX idx_' || tablename || '_created_at ON ' ||
    tablename || '(created_at);'
FROM information_schema.columns  
WHERE column_name = 'created_at'  
  AND table_schema = 'public'
\gexec
```

---

#### Accorder des permissions en masse

```sql
-- Donner SELECT sur toutes les tables à un utilisateur
SELECT 'GRANT SELECT ON ' || tablename || ' TO readonly_user;'  
FROM pg_tables  
WHERE schemaname = 'public'  
\gexec
```

---

#### Supprimer des tables temporaires

```sql
-- Nettoyer toutes les tables temp_*
SELECT 'DROP TABLE IF EXISTS ' || tablename || ' CASCADE;'  
FROM pg_tables  
WHERE tablename LIKE 'temp_%'  
  AND schemaname = 'public'
\gexec
```

---

**Sécurité** 🔒 : Toujours **vérifier** la requête générée avant d'utiliser `\gexec` :

```sql
-- 1. Afficher d'abord (sans \gexec)
SELECT 'DROP TABLE ' || tablename || ';'  
FROM pg_tables  
WHERE tablename LIKE 'old_%';  

-- 2. Vérifier que c'est bien ce que vous voulez
-- 3. Seulement alors, exécuter avec \gexec
SELECT 'DROP TABLE ' || tablename || ';'  
FROM pg_tables  
WHERE tablename LIKE 'old_%'  
\gexec;
```

---

## Édition et Visualisation de Code

### \sf - Voir le Code Source d'une Fonction

**Syntaxe** : `\sf[+] FUNCTION_NAME`

**Description** : Affiche le code source d'une fonction ou procédure.

**Exemples** :

```sql
-- Créer une fonction d'exemple
CREATE OR REPLACE FUNCTION calculer_total(commande_id INTEGER)  
RETURNS NUMERIC AS $$  
DECLARE  
    total NUMERIC;
BEGIN
    SELECT SUM(prix * quantite) INTO total
    FROM lignes_commandes
    WHERE commande_id = $1;

    RETURN COALESCE(total, 0);
END;
$$ LANGUAGE plpgsql;

-- Voir le code source
\sf calculer_total

# Résultat :
# CREATE OR REPLACE FUNCTION public.calculer_total(commande_id integer)
#  RETURNS numeric
#  LANGUAGE plpgsql
# AS $function$
# DECLARE
#     total NUMERIC;
# BEGIN
#     SELECT SUM(prix * quantite) INTO total
#     FROM lignes_commandes
#     WHERE commande_id = $1;
#
#     RETURN COALESCE(total, 0);
# END;
# $function$
```

---

#### Avec le signe +

```sql
-- Version détaillée avec numéros de ligne
\sf+ calculer_total
```

---

### \sv - Voir la Définition d'une Vue

**Syntaxe** : `\sv[+] VIEW_NAME`

**Description** : Affiche la requête SQL qui définit une vue.

**Exemples** :

```sql
-- Créer une vue
CREATE VIEW ventes_recentes AS  
SELECT  
    c.nom AS client,
    o.date_commande,
    o.montant_total
FROM clients c  
JOIN commandes o ON c.id = o.client_id  
WHERE o.date_commande >= NOW() - INTERVAL '30 days';  

-- Voir la définition
\sv ventes_recentes

# Résultat :
# CREATE OR REPLACE VIEW public.ventes_recentes AS
#  SELECT c.nom AS client,
#     o.date_commande,
#     o.montant_total
#    FROM clients c
#      JOIN commandes o ON c.id = o.client_id
#   WHERE o.date_commande >= (now() - '30 days'::interval)
```

---

### \ef - Éditer une Fonction

**Syntaxe** : `\ef [FUNCTION_NAME]`

**Description** : Ouvre votre éditeur (défini par `EDITOR` ou `PSQL_EDITOR`) pour éditer une fonction.

**Workflow** :
1. `\ef ma_fonction` ouvre l'éditeur avec le code actuel  
2. Vous modifiez le code  
3. Sauvegardez et quittez l'éditeur  
4. psql exécute automatiquement le `CREATE OR REPLACE FUNCTION`

**Exemples** :

```sql
-- Éditer une fonction existante
\ef calculer_total

-- Créer une nouvelle fonction (éditeur vide avec template)
\ef nouvelle_fonction
```

**Configuration de l'éditeur** :

```bash
# Dans votre shell
export PSQL_EDITOR=nano  
export PSQL_EDITOR=vim  
export PSQL_EDITOR="code --wait"  # VS Code  

# Ou dans .psqlrc
\setenv PSQL_EDITOR nano
```

---

### \ev - Éditer une Vue

**Syntaxe** : `\ev [VIEW_NAME]`

**Description** : Comme `\ef` mais pour les vues.

**Exemples** :

```sql
-- Éditer une vue existante
\ev ventes_recentes

-- Créer une nouvelle vue
\ev ma_nouvelle_vue
```

---

## Édition du Buffer de Requête

### Qu'est-ce que le Buffer ?

Le **buffer** est la zone de mémoire où psql stocke votre requête en cours de saisie. Ces commandes vous permettent de manipuler ce buffer.

---

### \e - Éditer dans un Éditeur

**Syntaxe** : `\e [FILENAME]`

**Description** : Ouvre votre éditeur pour écrire/modifier une requête.

**Exemples** :

#### Éditer le buffer actuel

```sql
-- Taper une requête (mais ne pas l'exécuter)
SELECT * FROM clients  
WHERE ville = 'Paris'  

-- Ouvrir dans l'éditeur pour modification
\e

-- Modifiez, sauvegardez, quittez
-- La requête modifiée est maintenant dans le buffer
-- Tapez ; pour l'exécuter
;
```

---

#### Éditer un fichier

```sql
-- Charger et éditer un fichier
\e /tmp/ma_requete.sql

-- Après modification et sauvegarde, le contenu est dans le buffer
-- Exécutez avec ;
;
```

---

#### Créer une requête complexe

```sql
-- Commencez à taper
SELECT
    c.nom,
    COUNT(o.id)
FROM clients c

-- Vous réalisez que c'est complexe, ouvrez l'éditeur
\e

-- Dans l'éditeur, complétez tranquillement :
SELECT
    c.nom,
    COUNT(o.id) as nb_commandes,
    SUM(o.montant_total) as total_achats
FROM clients c  
LEFT JOIN commandes o ON c.id = o.client_id  
WHERE o.date_commande >= NOW() - INTERVAL '1 year'  
GROUP BY c.id, c.nom  
HAVING COUNT(o.id) > 5  
ORDER BY total_achats DESC  
LIMIT 20;  

-- Sauvegardez et quittez, puis exécutez
;
```

---

### \p - Afficher le Buffer

**Syntaxe** : `\p`

**Description** : Affiche le contenu actuel du buffer (la requête en cours).

**Exemples** :

```sql
-- Taper une requête
SELECT * FROM clients WHERE id = 1

-- Voir ce qui est dans le buffer
\p

# Résultat :
# SELECT * FROM clients WHERE id = 1

-- Pratique pour vérifier avant d'exécuter
;
```

---

### \r - Réinitialiser le Buffer

**Syntaxe** : `\r`

**Description** : Vide le buffer (annule la requête en cours).

**Exemples** :

```sql
-- Commencer à taper
SELECT * FROM enormous_table WHERE

-- Oups, mauvaise requête, annuler
\r
Query buffer reset (cleared).

-- Le buffer est vide, recommencez
```

---

### \w - Écrire le Buffer dans un Fichier

**Syntaxe** : `\w FILENAME`

**Description** : Sauvegarde le contenu du buffer dans un fichier.

**Exemples** :

```sql
-- Écrire une belle requête
SELECT
    DATE_TRUNC('month', date_commande) AS mois,
    COUNT(*) AS nb_commandes,
    SUM(montant_total) AS total
FROM commandes  
WHERE date_commande >= '2024-01-01'  
GROUP BY 1  
ORDER BY 1;  

-- Sauvegarder dans un fichier
\w /tmp/rapport_mensuel.sql

-- Le fichier est créé, mais le buffer n'est pas vidé
-- Vous pouvez toujours exécuter la requête
;
```

---

## Gestion de l'Historique

### \s - Afficher/Sauvegarder l'Historique

**Syntaxe** :
```
\s [FILENAME]
```

**Description** : Affiche l'historique des commandes. Avec un fichier, sauvegarde l'historique dedans.

**Exemples** :

#### Afficher l'historique

```sql
-- Voir les dernières commandes
\s

# Résultat :
#    1  SELECT * FROM clients;
#    2  \dt
#    3  SELECT COUNT(*) FROM commandes;
#    4  \s
```

---

#### Sauvegarder l'historique

```sql
-- Sauvegarder toute la session
\s /tmp/ma_session_$(date +%Y%m%d).sql

-- Le fichier contient toutes vos commandes
\! cat /tmp/ma_session_20241121.sql
```

---

#### Rejouer l'historique

```sql
-- 1. Sauvegarder
\s /tmp/session.sql

-- 2. Nettoyer/éditer le fichier si nécessaire
\! nano /tmp/session.sql

-- 3. Rejouer
\i /tmp/session.sql
```

---

### Configuration de l'Historique

```sql
-- Dans .psqlrc

-- Fichier d'historique persistant par base
\set HISTFILE ~/.psql_history- :DBNAME

-- Taille de l'historique
\set HISTSIZE 10000

-- Ignorer les commandes dupliquées
\set HISTCONTROL ignoredups

-- Ignorer les commandes commençant par un espace
\set HISTCONTROL ignorespace

-- Combiner les deux
\set HISTCONTROL ignoreboth
```

---

## Affichage et Formatage Rapide

### \a - Basculer Format Aligned/Unaligned

**Syntaxe** : `\a`

**Description** : Bascule entre format aligned (tableaux) et unaligned (brut).

**Exemples** :

```sql
-- Format aligned (défaut)
SELECT id, nom FROM clients LIMIT 2;

 id |  nom
----+--------
  1 | Dupont
  2 | Martin

-- Basculer en unaligned
\a
Output format is unaligned.

SELECT id, nom FROM clients LIMIT 2;

id|nom
1|Dupont
2|Martin

-- Rebascules
\a
Output format is aligned.
```

---

### \C - Définir le Titre

**Syntaxe** : `\C [TITLE]`

**Description** : Raccourci pour `\pset title`.

**Exemples** :

```sql
\C 'Rapport des Ventes 2024'

SELECT * FROM ventes WHERE annee = 2024;

#      Rapport des Ventes 2024
#  mois  | total
# -------+--------
#  01    | 15000
#  02    | 18000
```

---

### \H - Basculer Sortie HTML

**Syntaxe** : `\H`

**Description** : Bascule entre format normal et HTML.

**Exemples** :

```sql
\H
Output format is html.

SELECT * FROM clients LIMIT 2;

# <table border="1">
#   <tr><th>id</th><th>nom</th></tr>
#   <tr><td>1</td><td>Dupont</td></tr>
#   <tr><td>2</td><td>Martin</td></tr>
# </table>

-- Rebascule
\H
Output format is aligned.
```

---

## Commandes Système et Navigation

### \cd - Changer de Répertoire

**Syntaxe** : `\cd [DIRECTORY]`

**Description** : Change le répertoire de travail de psql (comme `cd` en shell).

**Exemples** :

```sql
-- Aller dans /tmp
\cd /tmp

-- Vérifier
\! pwd
# /tmp

-- Revenir au home
\cd ~

-- Ou simplement
\cd
```

**Usage** : Pratique avant d'utiliser `\i`, `\o`, ou `\copy` avec des chemins relatifs.

---

### \! - Exécuter une Commande Shell

**Syntaxe** : `\! [COMMAND]`

**Description** : Exécute une commande système.

**Exemples avancés** :

```sql
-- Voir les processus PostgreSQL
\! ps aux | grep postgres

-- Voir l'espace disque
\! df -h | grep postgres

-- Créer un répertoire de backup
\! mkdir -p /backup/$(date +%Y%m%d)

-- Éditer un fichier rapidement
\! nano /tmp/config.txt

-- Rechercher dans les logs
\! tail -f /var/log/postgresql/postgresql.log | grep ERROR

-- Lancer un script Python
\! python3 /scripts/analyse.py
```

---

### \q ou \quit - Quitter psql

**Syntaxe** : `\q` ou `\quit` ou `Ctrl+D`

**Description** : Quitte psql.

**Exemples** :

```sql
-- Quitter normalement
\q

-- Ou
\quit

-- Ou presser Ctrl+D
```

**Astuce** 💡 : Si vous avez une transaction en cours, psql vous avertira.

---

## Variables psql Avancées

### Variables Spéciales

PostgreSQL psql a plusieurs variables spéciales qui contrôlent son comportement.

#### ON_ERROR_STOP

```sql
-- Arrêter l'exécution sur erreur (important pour les scripts)
\set ON_ERROR_STOP on

BEGIN;
    INSERT INTO clients (nom) VALUES ('Test');
    INSERT INTO bad_table (x) VALUES (1);  -- Erreur ici
    -- Le script s'arrête, ROLLBACK automatique
COMMIT;
```

---

#### ON_ERROR_ROLLBACK

```sql
-- Rollback automatique sur erreur dans une transaction
\set ON_ERROR_ROLLBACK on

BEGIN;
    INSERT INTO clients (nom) VALUES ('Alice');
    INSERT INTO clients (nom) VALUES (NULL);  -- Erreur
    -- Rollback automatique du dernier INSERT uniquement
    INSERT INTO clients (nom) VALUES ('Bob');  -- Réussit
COMMIT;
```

---

#### ECHO

```sql
-- Afficher toutes les commandes exécutées
\set ECHO all

SELECT * FROM clients;
# Affiche d'abord : SELECT * FROM clients;
# Puis le résultat

-- Afficher seulement les requêtes
\set ECHO queries

-- Afficher seulement les erreurs
\set ECHO errors
```

---

#### QUIET

```sql
-- Mode silencieux (pas de messages informatifs)
\set QUIET on

-- Désactiver
\set QUIET off
```

---

#### AUTOCOMMIT

```sql
-- Désactiver l'auto-commit (chaque commande devient une transaction)
\set AUTOCOMMIT off

INSERT INTO clients (nom) VALUES ('Test');
-- Pas encore committé, besoin de COMMIT; explicite

COMMIT;

-- Réactiver
\set AUTOCOMMIT on
```

---

### Variables Utilisateur Personnalisées

#### Définir et Utiliser des Variables

```sql
-- Définir des variables
\set ma_limite 100
\set ma_ville 'Paris'
\set date_debut '2024-01-01'

-- Utiliser (numérique sans quotes)
SELECT * FROM produits LIMIT :ma_limite;

-- Utiliser (chaîne avec quotes)
SELECT * FROM clients WHERE ville = :'ma_ville';

-- Utiliser dans une requête complexe
SELECT * FROM commandes  
WHERE date_commande >= :'date_debut'::date  
LIMIT :ma_limite;  
```

---

#### Variables d'Environnement

```sql
-- Accéder aux variables shell
\setenv USER_NAME 'alice'

-- Ou lire depuis le shell
\set current_user `whoami`
\set current_date `date +%Y-%m-%d`

\echo 'Exécuté par :' :current_user
\echo 'Date :' :current_date
```

---

#### Conditions avec Variables

```sql
-- Définir une variable de mode
\set mode 'production'

-- Utiliser dans des scripts conditionnels
\if :{?mode}
    \if :mode = 'production'
        \echo 'Mode PRODUCTION détecté'
        \set ON_ERROR_STOP on
    \else
        \echo 'Mode DÉVELOPPEMENT'
    \endif
\else
    \echo 'Variable mode non définie'
\endif
```

---

## Commandes d'Information Avancées

### \errverbose - Détails de la Dernière Erreur

**Syntaxe** : `\errverbose`

**Description** : Affiche la dernière erreur avec tous ses détails (codes, emplacement dans le code source).

**Exemples** :

```sql
-- Provoquer une erreur
SELECT * FROM table_inexistante;

# ERROR:  relation "table_inexistante" does not exist
# LINE 1: SELECT * FROM table_inexistante;
#                       ^

-- Voir tous les détails
\errverbose

# ERROR:  42P01: relation "table_inexistante" does not exist
# LINE 1: SELECT * FROM table_inexistante;
#                       ^
# LOCATION:  parserOpenTable, parse_relation.c:1180
```

**Usage** : Déboguer des erreurs complexes, comprendre la source exacte du problème.

---

### \dconfig - Configuration Serveur

**Syntaxe** : `\dconfig[+] [PATTERN]`

**Description** : Affiche les paramètres de configuration du serveur.

**Exemples** :

```sql
-- Tous les paramètres
\dconfig

-- Paramètres commençant par "shared"
\dconfig shared*

-- Version détaillée
\dconfig+ shared_buffers

#        List of configuration parameters
#      Name       | Setting | Unit |   Context
# ----------------+---------+------+--------------
#  shared_buffers | 128MB   |      | postmaster
```

---

### \drds - Paramètres par Rôle/Base

**Syntaxe** : `\drds [ROLE_PATTERN [DATABASE_PATTERN]]`

**Description** : Affiche les paramètres spécifiques définis par rôle et/ou base de données.

**Exemples** :

```sql
-- Tous les paramètres spécifiques
\drds

-- Paramètres pour un utilisateur
\drds alice

-- Paramètres pour une base
\drds * production
```

---

## Techniques Avancées de Scripting

### Scripts avec Conditions

```sql
-- script_conditionnel.sql

-- Vérifier si une table existe
\set table_exists false

SELECT EXISTS (
    SELECT 1 FROM information_schema.tables
    WHERE table_name = 'clients'
) as exists \gset table_

\if :table_exists
    \echo 'Table clients existe'
    SELECT COUNT(*) FROM clients;
\else
    \echo 'Table clients n''existe pas, création...'
    CREATE TABLE clients (id SERIAL PRIMARY KEY, nom TEXT);
\endif
```

---

### Scripts avec Boucles (via \gexec)

```sql
-- Créer 10 tables de test
SELECT 'CREATE TABLE test_' || i || ' (id SERIAL, data TEXT);'  
FROM generate_series(1, 10) as i  
\gexec

-- Les remplir avec des données
SELECT 'INSERT INTO test_' || i || ' (data) SELECT md5(random()::text) FROM generate_series(1, 1000);'  
FROM generate_series(1, 10) as i  
\gexec

-- Les analyser
SELECT 'ANALYZE test_' || i || ';'  
FROM generate_series(1, 10) as i  
\gexec
```

---

### Scripts avec Rapports

```sql
-- rapport_complet.sql

\set QUIET on
\pset footer off
\pset border 2

\o /tmp/rapport_$(date +%Y%m%d).txt

\echo '╔════════════════════════════════════════╗'
\echo '║   RAPPORT QUOTIDIEN                    ║'
\echo '╚════════════════════════════════════════╝'
\echo ''

\echo '=== STATISTIQUES GÉNÉRALES ==='
SELECT
    (SELECT COUNT(*) FROM clients) as nb_clients,
    (SELECT COUNT(*) FROM commandes) as nb_commandes,
    (SELECT SUM(montant_total) FROM commandes) as total_ventes;

\echo ''
\echo '=== TOP 10 CLIENTS ==='
SELECT
    c.nom,
    COUNT(o.id) as nb_commandes,
    SUM(o.montant_total) as total
FROM clients c  
JOIN commandes o ON c.id = o.client_id  
GROUP BY c.id, c.nom  
ORDER BY total DESC  
LIMIT 10;  

\echo ''
\echo '=== PERFORMANCE BASE ==='
\dconfig+ shared_buffers
\dconfig+ work_mem

\echo ''
\echo 'Rapport généré le :' `date`

\o
\set QUIET off
\echo 'Rapport sauvegardé dans /tmp/rapport_$(date +%Y%m%d).txt'
```

---

### Scripts Interactifs Avancés

```sql
-- backup_interactif.sql

\prompt 'Type de backup (full/incremental) : ' backup_type
\prompt 'Destination : ' backup_dest

\if :backup_type = 'full'
    \echo 'Backup COMPLET vers :' :backup_dest

    \copy clients TO :'backup_dest'/clients.csv CSV HEADER
    \copy commandes TO :'backup_dest'/commandes.csv CSV HEADER
    \copy produits TO :'backup_dest'/produits.csv CSV HEADER

    \echo 'Backup complet terminé'
\else
    \echo 'Backup INCRÉMENTAL vers :' :backup_dest

    \prompt 'Date depuis (YYYY-MM-DD) : ' since_date

    \copy (SELECT * FROM clients WHERE updated_at >= :'since_date')
        TO :'backup_dest'/clients_incr.csv CSV HEADER
    \copy (SELECT * FROM commandes WHERE updated_at >= :'since_date')
        TO :'backup_dest'/commandes_incr.csv CSV HEADER

    \echo 'Backup incrémental terminé'
\endif
```

---

## Large Objects (Objets Volumineux)

Les Large Objects sont un système de PostgreSQL pour stocker des fichiers volumineux (images, PDFs, vidéos).

### \lo_import - Importer un Fichier

**Syntaxe** : `\lo_import FILENAME [COMMENT]`

**Exemples** :

```sql
-- Importer une image
\lo_import '/tmp/photo.jpg' 'Photo de profil'

-- Résultat : lo 16385
-- (16385 est l'OID du large object)
```

---

### \lo_export - Exporter un Large Object

**Syntaxe** : `\lo_export OID FILENAME`

**Exemples** :

```sql
-- Exporter un large object
\lo_export 16385 '/tmp/photo_exported.jpg'
```

---

### \lo_list - Lister les Large Objects

**Syntaxe** : `\lo_list`

**Exemples** :

```sql
\lo_list

#  Large objects
#   ID   | Owner | Description
# -------+-------+------------------
#  16385 | postgres | Photo de profil
#  16386 | postgres | Document PDF
```

---

### \lo_unlink - Supprimer un Large Object

**Syntaxe** : `\lo_unlink OID`

**Exemples** :

```sql
-- Supprimer le large object
\lo_unlink 16385
```

---

## Astuces et Techniques Avancées

### 1. Créer des Alias avec des Fonctions Shell

Dans votre `.psqlrc` :

```sql
-- Alias pour voir les tables avec leur taille
\set show_sizes 'SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||''.''||tablename)) AS size FROM pg_tables WHERE schemaname = ''public'' ORDER BY pg_total_relation_size(schemaname||''.''||tablename) DESC;'

-- Usage : :show_sizes
```

---

### 2. Benchmark avec \watch et \timing

```sql
\timing on

-- Requête à benchmarker
SELECT COUNT(*) FROM huge_table WHERE condition;

\watch 0.1

-- Voir les variations de performance en temps réel
-- Ctrl+C pour arrêter
```

---

### 3. Debug avec \echo et Variables

```sql
-- debug_script.sql

\set DEBUG true

\if :DEBUG
    \echo 'DEBUG: Début du script'
\endif

SELECT COUNT(*) FROM clients \gset nb_

\if :DEBUG
    \echo 'DEBUG: Nombre de clients trouvés :' :nb_count
\endif

-- ... reste du script
```

---

### 4. Pagination Manuelle avec Variables

```sql
\set page_size 20
\set current_page 0

-- Page 1
SELECT * FROM produits  
ORDER BY id  
LIMIT :page_size  
OFFSET :current_page * :page_size;  

-- Page suivante
\set current_page 1
SELECT * FROM produits  
ORDER BY id  
LIMIT :page_size  
OFFSET :current_page * :page_size;  
```

---

### 5. Validation de Scripts avec \set ON_ERROR_STOP

```sql
-- validation_script.sql

\set ON_ERROR_STOP on

BEGIN;

\echo 'Test 1: Insertion...'
INSERT INTO test_table (value) VALUES ('test1');

\echo 'Test 2: Update...'
UPDATE test_table SET value = 'test2' WHERE value = 'test1';

\echo 'Test 3: Contrainte...'
INSERT INTO test_table (id) VALUES (1);  -- Devrait échouer si existe

ROLLBACK;

\echo 'Tous les tests sont OK (en rollback)'
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Utilisez \watch pour surveiller** plutôt que des boucles shell
   ```sql
   SELECT * FROM monitoring_view;
   \watch 5
   ```

2. **Testez avant \gexec** en affichant d'abord
   ```sql
   -- 1. Voir les commandes
   SELECT 'DROP TABLE ' || tablename FROM pg_tables WHERE...;

   -- 2. Vérifier
   -- 3. Exécuter
   SELECT 'DROP TABLE ' || tablename FROM pg_tables WHERE... \gexec
   ```

3. **Utilisez \gset pour scripts dynamiques**
   ```sql
   SELECT MAX(id) as max_id FROM table \gset
   -- Puis utiliser :max_id
   ```

4. **Documentez vos variables dans les scripts**
   ```sql
   -- Variables configurables
   \set backup_path '/backup/daily'
   \set retention_days 30
   ```

5. **Utilisez \sf et \sv pour apprendre**
   ```sql
   -- Voir comment une fonction système est faite
   \sf pg_size_pretty
   ```

---

### ❌ À Éviter

1. **Ne pas vérifier avant \gexec**
   ```sql
   -- ❌ Dangereux
   SELECT 'DROP DATABASE ' || datname FROM pg_database \gexec
   ```

2. **Oublier ON_ERROR_STOP dans les scripts**
   ```sql
   -- ❌ Continue même sur erreur
   \i script.sql

   -- ✅ Arrête sur erreur
   \set ON_ERROR_STOP on
   \i script.sql
   ```

3. **Utiliser \watch sur des requêtes lourdes**
   ```sql
   -- ❌ Va surcharger le serveur
   SELECT * FROM huge_table JOIN another_huge_table...;
   \watch 1
   ```

4. **Modifier directement avec \ef sans backup**
   ```sql
   -- ✅ D'abord sauvegarder
   \sf ma_fonction_importante > /backup/fonction.sql
   -- Puis éditer
   \ef ma_fonction_importante
   ```

---

## Résumé des Commandes Avancées

| Commande | Description | Exemple |
|----------|-------------|---------|
| `\watch [N]` | Exécution répétée | `SELECT COUNT(*) FROM clients; \watch 5` |
| `\gx` | Exécuter en mode étendu | `SELECT * FROM clients WHERE id = 1 \gx` |
| `\gset [P]` | Stocker dans variables | `SELECT MAX(id) as m FROM t \gset` |
| `\gexec` | Exécuter résultats comme SQL | `SELECT 'VACUUM ' || t FROM pg_tables \gexec` |
| `\sf FUNC` | Voir code fonction | `\sf ma_fonction` |
| `\sv VIEW` | Voir définition vue | `\sv ma_vue` |
| `\ef FUNC` | Éditer fonction | `\ef ma_fonction` |
| `\ev VIEW` | Éditer vue | `\ev ma_vue` |
| `\e [FILE]` | Éditer dans éditeur | `\e /tmp/query.sql` |
| `\p` | Afficher buffer | `\p` |
| `\r` | Réinitialiser buffer | `\r` |
| `\w FILE` | Écrire buffer | `\w /tmp/query.sql` |
| `\s [FILE]` | Historique | `\s /tmp/history.sql` |
| `\cd [DIR]` | Changer répertoire | `\cd /tmp` |
| `\! CMD` | Commande shell | `\! ls -la` |
| `\errverbose` | Détails erreur | `\errverbose` |

---

## Conclusion

Les méta-commandes avancées de psql transforment votre productivité. Points clés :

🎯 **Surveillance** :  
- `\watch` pour monitoring en temps réel
- Parfait pour développement et debugging

⚡ **Automatisation** :  
- `\gexec` pour générer et exécuter du SQL  
- `\gset` pour scripts dynamiques
- Variables pour scripts réutilisables

✏️ **Édition** :  
- `\ef` et `\ev` pour éditer fonctions/vues  
- `\e` pour requêtes complexes  
- `\sf` et `\sv` pour apprendre

📊 **Scripts** :
- Conditions avec `\if`
- Variables utilisateur personnalisées
- `ON_ERROR_STOP` pour robustesse

💡 **Pro Tips** :
- Combinez plusieurs commandes pour des workflows puissants
- Créez des alias dans `.psqlrc`
- Documentez vos scripts avec `\echo`
- Testez toujours avant d'utiliser `\gexec`

---

**Vous avez maintenant une boîte à outils complète pour psql !**

Les 5 guides psql :
1. ✅ Introduction et glossaire  
2. ✅ Navigation  
3. ✅ Configuration  
4. ✅ Export/Import  
5. ✅ Méta-commandes avancées (celui-ci)

---


⏭️ [Requêtes SQL de Référence](/annexes/requetes-sql-reference/README.md)
