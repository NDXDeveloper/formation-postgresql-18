🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.5. Analyse des logs : Configuration et interprétation (log_line_prefix, auto_explain)

## Introduction

Les **logs** (journaux) de PostgreSQL sont comme le journal de bord d'un navire : ils enregistrent tout ce qui se passe sur votre serveur de base de données. Bien configurés et correctement analysés, ils deviennent votre outil de diagnostic le plus puissant.

Dans cette section, nous allons explorer :
- Comment configurer les logs pour capturer les informations essentielles
- Comment personnaliser le format des logs avec `log_line_prefix`
- Comment utiliser `auto_explain` pour identifier automatiquement les requêtes lentes
- Comment interpréter et analyser efficacement vos logs

**Pourquoi c'est crucial ?** Sans logs bien configurés, vous êtes aveugle. Avec eux, vous pouvez :
- Diagnostiquer les erreurs et les ralentissements
- Identifier les requêtes problématiques
- Auditer l'activité (qui fait quoi, quand)
- Analyser les tendances et anticiper les problèmes

---

## Partie 1 : Comprendre les Logs PostgreSQL

### Qu'est-ce qu'un log ?

Un **log** (ou journal) est un fichier texte où PostgreSQL enregistre des événements, des erreurs, et des informations sur son fonctionnement.

**Analogie** : Les logs sont comme les caméras de surveillance d'un magasin. Elles enregistrent tout ce qui se passe, et quand il y a un problème (vol, incident), vous pouvez revenir en arrière et voir exactement ce qui s'est passé.

### Que trouve-t-on dans les logs ?

PostgreSQL peut enregistrer :
- **Erreurs** : Connexions échouées, requêtes invalides, deadlocks  
- **Avertissements** : Problèmes potentiels, dépréciations  
- **Informations** : Démarrage/arrêt du serveur, checkpoints, connexions  
- **Requêtes** : Les requêtes SQL exécutées (si configuré)  
- **Performances** : Temps d'exécution des requêtes lentes  
- **Audit** : Qui s'est connecté, quand, depuis où

### Où sont les logs ?

La localisation des logs dépend de votre configuration :

**Linux (Debian/Ubuntu)** :
```
/var/log/postgresql/postgresql-18-main.log
```

**Linux (Red Hat/CentOS)** :
```
/var/lib/pgsql/18/data/log/
```

**macOS (Homebrew)** :
```
/opt/homebrew/var/log/postgresql@18.log
```

**Windows** :
```
C:\Program Files\PostgreSQL\18\data\log\
```

Pour trouver l'emplacement exact depuis psql :
```sql
SHOW log_directory;  
SHOW log_filename;  
```

### Types de destination des logs

PostgreSQL peut envoyer les logs vers différentes destinations :

1. **stderr** : Sortie d'erreur standard (capture par systemd/journald sur Linux)  
2. **csvlog** : Format CSV structuré (plus facile à analyser)  
3. **jsonlog** : Format JSON (PostgreSQL 15+, idéal pour Elasticsearch/Splunk)  
4. **syslog** : Système de logs Unix/Linux centralisé  
5. **eventlog** : Journal d'événements Windows

**Configuration recommandée** :
```conf
# Dans postgresql.conf
logging_collector = on            # Active la collecte de logs  
log_destination = 'stderr'        # Ou 'csvlog' pour analyse automatisée  
log_directory = 'log'             # Répertoire relatif à PGDATA  
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'  # Un fichier par démarrage  
log_rotation_age = 1d             # Rotation quotidienne  
log_rotation_size = 100MB         # Rotation si > 100 MB  
```

---

## Partie 2 : Configurer les Logs - Les Paramètres Essentiels

### Vue d'ensemble des paramètres de logging

PostgreSQL offre plus de 30 paramètres pour contrôler le logging. Voici les plus importants :

| Paramètre | Description | Valeur recommandée |
|-----------|-------------|-------------------|
| `log_min_messages` | Niveau minimum de messages à logger | `warning` (production) |
| `log_min_error_statement` | Niveau minimum pour logger les requêtes avec erreur | `error` |
| `log_min_duration_statement` | Logger les requêtes plus lentes que X ms | `1000` (1 seconde) |
| `log_connections` | Logger les connexions | `on` |
| `log_disconnections` | Logger les déconnexions | `on` |
| `log_duration` | Logger la durée de chaque requête | `off` (utiliser log_min_duration_statement) |
| `log_statement` | Quelles requêtes logger | `none` ou `ddl` |
| `log_checkpoints` | Logger les checkpoints | `on` |
| `log_lock_waits` | Logger les attentes de locks | `on` |
| `log_temp_files` | Logger la création de fichiers temporaires | `0` (tous) |

### Configuration détaillée : Paramètre par paramètre

#### 1. log_min_messages

Contrôle le niveau minimum de gravité des messages à enregistrer.

**Niveaux disponibles (du moins au plus grave)** :
- `DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1` : Débogage très verbeux  
- `INFO` : Informations générales  
- `NOTICE` : Notifications  
- `WARNING` : Avertissements  
- `ERROR` : Erreurs (transaction annulée)  
- `LOG` : Messages importants pour l'administrateur  
- `FATAL` : Erreur grave (connexion fermée)  
- `PANIC` : Erreur critique (arrêt du serveur)

**Configuration recommandée** :
```conf
# Développement
log_min_messages = info

# Production
log_min_messages = warning
```

**Pourquoi ?** En production, `info` génère trop de logs. `warning` capture les problèmes importants sans noyer dans les détails.

#### 2. log_min_duration_statement

**Le paramètre le plus important pour la performance !**

Logger toutes les requêtes qui prennent plus de X millisecondes.

```conf
# Logger les requêtes de plus de 1 seconde
log_min_duration_statement = 1000

# Logger toutes les requêtes (ATTENTION : très verbeux)
log_min_duration_statement = 0

# Ne rien logger (par défaut)
log_min_duration_statement = -1
```

**Recommandation** :
- **Développement** : 100-500 ms  
- **Production** : 1000-5000 ms (1-5 secondes)  
- **Analyse ponctuelle** : 0 (toutes les requêtes)

**Exemple de log généré** :
```
2025-11-21 14:32:45 CET [12345]: LOG:  duration: 1234.567 ms  statement: SELECT * FROM orders WHERE created_at > '2025-01-01'
```

#### 3. log_connections et log_disconnections

Enregistre toutes les connexions et déconnexions.

```conf
log_connections = on  
log_disconnections = on  
```

**Utilité** :
- Audit de sécurité (qui se connecte, depuis où)
- Détection d'attaques par force brute
- Identification de connection leaks (connexions jamais fermées)

**Exemple de logs** :
```
2025-11-21 14:30:12 CET [12340]: LOG:  connection received: host=192.168.1.50 port=54321
2025-11-21 14:30:12 CET [12340]: LOG:  connection authorized: user=webapp database=production application_name=MyApp
2025-11-21 14:35:45 CET [12340]: LOG:  disconnection: session time: 0:05:33.123 user=webapp database=production host=192.168.1.50 port=54321
```

#### 4. log_statement

Contrôle quelles **catégories** de requêtes sont loggées.

```conf
# Ne rien logger (par défaut, recommandé)
log_statement = 'none'

# Logger uniquement les DDL (CREATE, ALTER, DROP)
log_statement = 'ddl'

# Logger DDL + DML avec modifications (INSERT, UPDATE, DELETE)
log_statement = 'mod'

# Logger TOUTES les requêtes (ATTENTION : très verbeux)
log_statement = 'all'
```

**Recommandation** :
- **Production normale** : `none` (utiliser `log_min_duration_statement` à la place)  
- **Audit strict** : `ddl` ou `mod`  
- **Débogage** : `all` (temporairement)

**Différence avec log_min_duration_statement** :
- `log_statement` : Basé sur le **type** de requête  
- `log_min_duration_statement` : Basé sur la **durée** d'exécution

#### 5. log_checkpoints

Logger les checkpoints (synchronisation de la mémoire vers le disque).

```conf
log_checkpoints = on
```

**Exemple de log** :
```
2025-11-21 14:40:00 CET [12330]: LOG:  checkpoint starting: time
2025-11-21 14:40:15 CET [12330]: LOG:  checkpoint complete: wrote 2456 buffers (15.0%); 0 WAL file(s) added, 0 removed, 3 recycled; write=12.345 s, sync=2.123 s, total=15.123 s; sync files=45, longest=0.456 s, average=0.047 s; distance=45678 kB, estimate=45678 kB
```

**Utilité** : Identifier si les checkpoints prennent trop de temps (signe de problème I/O).

#### 6. log_lock_waits

Logger quand un processus attend un verrou (lock) depuis plus de `deadlock_timeout` (1 seconde par défaut).

```conf
log_lock_waits = on  
deadlock_timeout = 1s  
```

**Exemple de log** :
```
2025-11-21 14:45:23 CET [12355]: LOG:  process 12355 still waiting for ShareLock on transaction 123456 after 1000.123 ms
2025-11-21 14:45:23 CET [12355]: DETAIL:  Process holding the lock: 12350. Wait queue: 12355.
2025-11-21 14:45:23 CET [12355]: STATEMENT:  UPDATE accounts SET balance = balance + 100 WHERE id = 42
```

**Utilité** : Identifier les contentions de locks et les deadlocks potentiels.

#### 7. log_temp_files

Logger la création de fichiers temporaires (quand une requête ne peut pas tenir en mémoire).

```conf
# Logger tous les fichiers temporaires
log_temp_files = 0

# Logger uniquement ceux > 10 MB
log_temp_files = 10240  # En kB
```

**Exemple de log** :
```
2025-11-21 14:50:00 CET [12360]: LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp12360.0", size 524288000
2025-11-21 14:50:00 CET [12360]: STATEMENT:  SELECT * FROM huge_table ORDER BY created_at
```

**Utilité** : Détecter les requêtes qui nécessitent trop de mémoire (à optimiser ou augmenter `work_mem`).

---

## Partie 3 : log_line_prefix - Personnaliser le Format des Logs

### Qu'est-ce que log_line_prefix ?

`log_line_prefix` est un **template** qui définit le format de chaque ligne de log. C'est comme personnaliser l'affichage d'un relevé bancaire : vous choisissez quelles informations apparaissent et dans quel ordre.

### Format par défaut

Par défaut, PostgreSQL utilise un format minimal :
```conf
log_line_prefix = '%m [%p] '
```

Résultat :
```
2025-11-21 14:32:45 CET [12345] LOG:  statement: SELECT 1
```

**Problème** : Pas assez d'informations pour un diagnostic efficace !

### Les codes de formatage disponibles

Voici les principaux codes (« placeholders ») utilisables dans `log_line_prefix` :

| Code | Description | Exemple |
|------|-------------|---------|
| `%a` | Nom de l'application | `MyApp` |
| `%u` | Nom d'utilisateur | `webapp` |
| `%d` | Nom de la base de données | `production` |
| `%r` | Nom d'hôte/port du client | `192.168.1.50:54321` |
| `%h` | Nom d'hôte du client uniquement | `192.168.1.50` |
| `%p` | Process ID (PID) | `12345` |
| `%t` | Timestamp (sans millisecondes) | `2025-11-21 14:32:45 CET` |
| `%m` | Timestamp (avec millisecondes) | `2025-11-21 14:32:45.123 CET` |
| `%n` | Timestamp (Unix epoch) | `1732195965.123` |
| `%i` | Command tag (type de commande) | `SELECT`, `UPDATE` |
| `%e` | SQLSTATE (code d'erreur) | `42P01` |
| `%c` | Session ID | `66e3d4f2.1234` |
| `%l` | Numéro de ligne de log par session | `1`, `2`, `3`... |
| `%s` | Timestamp de début de session | `2025-11-21 14:30:00 CET` |
| `%v` | Virtual transaction ID | `3/1234` |
| `%x` | Transaction ID (XID) | `123456` |
| `%q` | Pas de sortie (utilisé pour des conditions) | - |
| `%%` | Caractère `%` littéral | `%` |

### Configuration recommandée pour la production

Voici une configuration complète et équilibrée :

```conf
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

**Résultat** :
```
2025-11-21 14:32:45 CET [12345]: [1-1] user=webapp,db=production,app=MyApp,client=192.168.1.50 LOG:  statement: SELECT * FROM users WHERE id = 42
```

**Avantages** :
- ✅ Timestamp précis (`%t`)  
- ✅ PID du processus (`%p`)  
- ✅ Numéro de ligne (`%l`)  
- ✅ Contexte complet : utilisateur, base, application, client  
- ✅ Facile à parser avec des outils d'analyse

### Configuration enrichie avec transaction ID

Pour le débogage avancé ou l'audit strict :

```conf
log_line_prefix = '%m [%p] %q[user=%u,db=%d,app=%a,host=%h] [vxid=%v,txid=%x] '
```

**Résultat** :
```
2025-11-21 14:32:45.123 CET [12345] [user=webapp,db=production,app=MyApp,host=192.168.1.50] [vxid=3/1234,txid=123456] LOG:  statement: UPDATE accounts SET balance = balance + 100
```

**Informations supplémentaires** :
- Transaction virtuelle (`vxid`) : Pour corréler plusieurs requêtes d'une même transaction
- Transaction ID (`txid`) : L'identifiant de transaction unique

### Configuration pour analyse automatisée (CSV-like)

Si vous voulez parser les logs avec des scripts :

```conf
log_line_prefix = '%m|%p|%u|%d|%a|%h|%l|'
```

**Résultat** :
```
2025-11-21 14:32:45 CET|12345|webapp|production|MyApp|192.168.1.50|1|LOG:  statement: SELECT 1
```

**Avantage** : Facile à parser avec `awk`, `cut`, ou des scripts Python.

### Exemples de configurations par cas d'usage

#### Développement local (minimal)
```conf
log_line_prefix = '%t [%p] %q%u@%d '
```

#### Production (équilibré)
```conf
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

#### Audit de sécurité (maximal)
```conf
log_line_prefix = '%m [%p] %q[user=%u,db=%d,app=%a,host=%r,session=%c,line=%l] '
```

#### Analyse automatisée (structuré)
```conf
log_destination = 'csvlog'  # Utiliser le format CSV natif plutôt que log_line_prefix
```

---

## Partie 4 : auto_explain - L'Extension Magique

### Qu'est-ce qu'auto_explain ?

**auto_explain** est une extension PostgreSQL qui **capture automatiquement les plans d'exécution** (EXPLAIN) des requêtes lentes et les écrit dans les logs.

**Analogie** : Imaginez un médecin qui prend automatiquement votre température chaque fois que vous avez de la fièvre. auto_explain fait la même chose pour vos requêtes : il "radiographie" automatiquement celles qui sont lentes.

**Pourquoi c'est génial ?**
- ❌ Sans auto_explain : Vous devez deviner quelles requêtes sont lentes, puis manuellement faire `EXPLAIN ANALYZE`  
- ✅ Avec auto_explain : PostgreSQL le fait automatiquement et vous avez l'historique dans les logs

### Installation et activation

#### Étape 1 : Charger l'extension au démarrage

Éditez `postgresql.conf` :

```conf
# Charger auto_explain au démarrage du serveur
shared_preload_libraries = 'auto_explain,pg_stat_statements'  # Si vous avez déjà pg_stat_statements

# Si c'est la seule extension
shared_preload_libraries = 'auto_explain'
```

**Important** : Nécessite un **redémarrage** du serveur PostgreSQL.

```bash
sudo systemctl restart postgresql
```

#### Étape 2 : Configurer auto_explain

Toujours dans `postgresql.conf`, ajoutez les paramètres de configuration :

```conf
# === Configuration auto_explain ===

# Activer auto_explain (on par défaut si chargé)
auto_explain.log_min_duration = 1000  # Logger les requêtes > 1 seconde (en ms)

# Niveau de détail du plan d'exécution
auto_explain.log_analyze = on         # Inclure les statistiques d'exécution réelles  
auto_explain.log_buffers = on         # Inclure les statistiques de buffers  
auto_explain.log_timing = on          # Inclure le timing détaillé par nœud  
auto_explain.log_triggers = on        # Inclure les triggers  
auto_explain.log_verbose = off        # Mode verbose (très verbeux)  

# Format du plan
auto_explain.log_format = 'text'      # Options: 'text', 'xml', 'json', 'yaml'

# Inclure les requêtes imbriquées (dans les fonctions)
auto_explain.log_nested_statements = on

# Échantillonnage (pour réduire l'overhead)
auto_explain.sample_rate = 1.0        # 1.0 = 100% des requêtes, 0.1 = 10%
```

#### Étape 3 : Recharger la configuration

```bash
sudo systemctl reload postgresql
# Ou depuis psql :
SELECT pg_reload_conf();
```

**Note** : Contrairement au chargement initial, les modifications de paramètres ne nécessitent qu'un reload, pas un redémarrage.

### Configuration recommandée par environnement

#### Développement
```conf
auto_explain.log_min_duration = 100   # 100 ms  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
auto_explain.log_timing = on  
auto_explain.sample_rate = 1.0        # Tout logger  
```

#### Production
```conf
auto_explain.log_min_duration = 5000  # 5 secondes  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
auto_explain.log_timing = off         # Réduit l'overhead  
auto_explain.sample_rate = 0.1        # Échantillonner 10% des requêtes lentes  
```

#### Analyse ponctuelle (troubleshooting)
```conf
auto_explain.log_min_duration = 0     # Tout logger (temporairement !)  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
auto_explain.log_verbose = on  
auto_explain.sample_rate = 1.0  
```

### Paramètres détaillés

#### auto_explain.log_min_duration

Seuil de durée (en millisecondes) à partir duquel une requête est loggée.

```conf
# Logger les requêtes de plus de 5 secondes
auto_explain.log_min_duration = 5000

# Logger toutes les requêtes (ATTENTION : très verbeux)
auto_explain.log_min_duration = 0

# Désactiver auto_explain
auto_explain.log_min_duration = -1
```

**Recommandation** : Commencez avec 5000 ms (5 secondes) en production, ajustez selon vos besoins.

#### auto_explain.log_analyze

Active `EXPLAIN ANALYZE` (exécution réelle) au lieu de `EXPLAIN` simple (estimation).

```conf
auto_explain.log_analyze = on  # Recommandé
```

**Différence** :
- `EXPLAIN` : Estimation du planificateur (rapide, pas d'exécution)  
- `EXPLAIN ANALYZE` : Exécution réelle + statistiques (plus lent, plus précis)

**Impact** : Ajoute environ 10-20% d'overhead sur la requête loggée.

#### auto_explain.log_buffers

Inclut les statistiques de buffers (cache hits/misses).

```conf
auto_explain.log_buffers = on  # Fortement recommandé
```

**Utilité** : Voir combien de données ont été lues depuis le disque vs le cache.

#### auto_explain.log_timing

Inclut le timing détaillé pour chaque nœud du plan.

```conf
# Production : off (réduit l'overhead)
auto_explain.log_timing = off

# Développement : on (plus de détails)
auto_explain.log_timing = on
```

**Impact** : Peut ajouter 5-10% d'overhead supplémentaire.

#### auto_explain.sample_rate

Échantillonne seulement un pourcentage des requêtes lentes.

```conf
# Logger 100% des requêtes lentes
auto_explain.sample_rate = 1.0

# Logger seulement 10% des requêtes lentes (réduit l'overhead)
auto_explain.sample_rate = 0.1

# Logger 1% (pour très haute charge)
auto_explain.sample_rate = 0.01
```

**Utilité** : Réduire l'overhead et la taille des logs en haute charge.

### Exemple de log auto_explain

Voici ce que vous verrez dans les logs quand auto_explain capture une requête :

```
2025-11-21 15:30:45 CET [12567]: [1-1] user=webapp,db=production,app=MyApp,client=192.168.1.50 LOG:  duration: 1234.567 ms  plan:
Query Text: SELECT o.*, u.email FROM orders o JOIN users u ON o.user_id = u.id WHERE o.created_at > '2025-01-01' ORDER BY o.created_at DESC LIMIT 100  
Hash Join  (cost=1234.56..45678.90 rows=5000 width=200) (actual time=45.123..1200.456 rows=4876 loops=1)  
  Hash Cond: (o.user_id = u.id)
  Buffers: shared hit=12345 read=6789
  ->  Index Scan using orders_created_at_idx on orders o  (cost=0.43..34567.89 rows=5000 width=150) (actual time=0.123..800.456 rows=4876 loops=1)
        Index Cond: (created_at > '2025-01-01 00:00:00'::timestamp without time zone)
        Buffers: shared hit=8901 read=3456
  ->  Hash  (cost=987.65..987.65 rows=10000 width=50) (actual time=44.123..44.123 rows=9876 loops=1)
        Buckets: 16384  Batches: 1  Memory Usage: 456kB
        Buffers: shared hit=3444 read=3333
        ->  Seq Scan on users u  (cost=0.00..987.65 rows=10000 width=50) (actual time=0.012..25.678 rows=9876 loops=1)
              Buffers: shared hit=3444 read=3333
Planning Time: 2.345 ms  
Execution Time: 1234.567 ms  
```

**Informations capturées** :
- Durée totale : 1234.567 ms
- Texte de la requête complet
- Plan d'exécution avec temps réel par nœud
- Statistiques de buffers (cache hits vs disk reads)
- Temps de planification vs exécution

---

## Partie 5 : Interpréter les Logs

### Anatomie d'une ligne de log

Reprenons notre configuration recommandée :
```conf
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

**Exemple de ligne** :
```
2025-11-21 15:45:23 CET [12678]: [1-1] user=webapp,db=production,app=MyApp,client=192.168.1.50 ERROR:  duplicate key value violates unique constraint "users_email_key"
```

**Décomposition** :
1. `2025-11-21 15:45:23 CET` : Timestamp exact  
2. `[12678]` : Process ID (PID) du backend  
3. `[1-1]` : Numéro de ligne de log pour cette session  
4. `user=webapp` : Utilisateur PostgreSQL  
5. `db=production` : Base de données  
6. `app=MyApp` : Nom de l'application  
7. `client=192.168.1.50` : Adresse IP du client  
8. `ERROR:` : Niveau de gravité  
9. Le message d'erreur

### Niveaux de gravité et leur signification

| Niveau | Signification | Action requise |
|--------|---------------|----------------|
| `DEBUG` | Information de débogage | Aucune (développement uniquement) |
| `INFO` | Information générale | Aucune |
| `NOTICE` | Notification | Aucune (sauf si récurrent) |
| `WARNING` | Avertissement | Investiguer si fréquent |
| `ERROR` | Erreur (transaction annulée) | Vérifier l'application |
| `LOG` | Message administratif important | Noter, analyser si nécessaire |
| `FATAL` | Erreur grave (connexion fermée) | Urgence - investiguer immédiatement |
| `PANIC` | Erreur critique (serveur arrêté) | Urgence maximale - incident majeur |

### Exemples d'erreurs courantes et leur interprétation

#### Erreur 1 : Violation de contrainte unique

```
ERROR:  duplicate key value violates unique constraint "users_email_key"  
DETAIL:  Key (email)=(user@example.com) already exists.  
STATEMENT:  INSERT INTO users (email, name) VALUES ('user@example.com', 'John Doe')  
```

**Interprétation** :
- L'application tente d'insérer un email qui existe déjà
- La contrainte `users_email_key` empêche les doublons

**Actions** :
- Vérifier la logique applicative (valider avant d'insérer)
- Utiliser `INSERT ... ON CONFLICT` (upsert)

#### Erreur 2 : Deadlock détecté

```
ERROR:  deadlock detected  
DETAIL:  Process 12678 waits for ShareLock on transaction 123456; blocked by process 12679.  
Process 12679 waits for ShareLock on transaction 123457; blocked by process 12678.  
HINT:  See server log for query details.  
STATEMENT:  UPDATE accounts SET balance = balance - 100 WHERE id = 42  
```

**Interprétation** :
- Deux transactions se bloquent mutuellement
- PostgreSQL a détecté le deadlock et annulé une transaction

**Actions** :
- Revoir l'ordre des opérations dans les transactions
- Utiliser des verrous explicites (`SELECT ... FOR UPDATE`)

#### Erreur 3 : Fichier temporaire créé

```
LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp12678.0", size 524288000  
STATEMENT:  SELECT * FROM huge_table ORDER BY created_at  
```

**Interprétation** :
- La requête nécessite 500 MB de fichiers temporaires (sort sur disque)
- La mémoire allouée (`work_mem`) n'est pas suffisante

**Actions** :
- Augmenter `work_mem` pour cette session ou globalement
- Ajouter un index sur `created_at`
- Limiter les résultats avec `LIMIT`

#### Erreur 4 : Connexion échouée

```
FATAL:  password authentication failed for user "webapp"  
DETAIL:  Connection matched pg_hba.conf line 95: "host all all 192.168.1.0/24 scram-sha-256"  
```

**Interprétation** :
- Tentative de connexion avec un mauvais mot de passe
- Si fréquent : possible attaque par force brute

**Actions** :
- Vérifier les identifiants de l'application
- Si attaque : bloquer l'IP ou utiliser fail2ban

#### Erreur 5 : Lock wait

```
LOG:  process 12678 still waiting for ShareLock on transaction 123456 after 1000.123 ms  
DETAIL:  Process holding the lock: 12679. Wait queue: 12678.  
STATEMENT:  UPDATE orders SET status = 'shipped' WHERE id = 42  
```

**Interprétation** :
- Le processus 12678 attend un lock détenu par 12679
- Attente de plus de 1 seconde

**Actions** :
- Identifier la requête longue (processus 12679)
- Terminer la transaction bloquante si nécessaire

### Patterns à surveiller dans les logs

#### Pattern 1 : Connexions fréquentes (connection storm)

```bash
# Compter les connexions par minute
grep "connection authorized" postgresql.log | \
  cut -d' ' -f1-2 | uniq -c | sort -rn
```

**Signe de problème** : Plus de 100 connexions/minute de la même source.

**Causes** :
- Pas de connection pooling
- Application qui ouvre/ferme trop souvent

#### Pattern 2 : Requêtes lentes récurrentes

```bash
# Identifier les requêtes > 5 secondes
grep "duration:" postgresql.log | \
  awk '$6 > 5000' | \
  cut -d':' -f4- | sort | uniq -c | sort -rn | head -20
```

**Actions** :
- Optimiser ces requêtes (index, réécriture)
- Utiliser `auto_explain` pour analyser

#### Pattern 3 : Erreurs en cascade

```
ERROR:  canceling statement due to user request  
ERROR:  current transaction is aborted, commands ignored until end of transaction block  
ERROR:  current transaction is aborted, commands ignored until end of transaction block  
```

**Interprétation** :
- Une erreur dans une transaction
- L'application continue d'envoyer des commandes sans gérer l'erreur

**Actions** :
- Améliorer la gestion d'erreurs dans le code applicatif

---

## Partie 6 : Outils d'Analyse des Logs

### pgBadger - L'Outil Incontournable

**pgBadger** est un analyseur de logs PostgreSQL qui génère des rapports HTML détaillés.

#### Installation

```bash
# Debian/Ubuntu
sudo apt-get install pgbadger

# macOS
brew install pgbadger

# Ou depuis CPAN
cpan App::pgBadger
```

#### Utilisation basique

```bash
# Analyser un fichier de log
pgbadger /var/log/postgresql/postgresql-18-main.log

# Analyser plusieurs fichiers
pgbadger /var/log/postgresql/postgresql-*.log

# Spécifier le fichier de sortie
pgbadger -o report.html /var/log/postgresql/postgresql-18-main.log

# Analyser une période spécifique
pgbadger --begin "2025-11-20 00:00:00" --end "2025-11-21 00:00:00" postgresql.log
```

#### Rapports générés

pgBadger génère un rapport HTML avec :
- **Statistiques globales** : Nombre de requêtes, durée totale, erreurs  
- **Requêtes les plus lentes** : Top N avec temps moyen/max  
- **Requêtes les plus fréquentes** : Top N avec nombre d'appels  
- **Requêtes normalisées** : Agrégation des requêtes similaires  
- **Erreurs et avertissements** : Liste complète avec contexte  
- **Connexions et sessions** : Patterns de connexion  
- **Checkpoints et autovacuum** : Activité de maintenance  
- **Graphiques temporels** : Évolution au fil du temps

#### Configuration pour pgBadger

Pour des résultats optimaux, configurez PostgreSQL pour générer des logs compatibles :

```conf
# Dans postgresql.conf
log_min_duration_statement = 0       # Logger toutes les requêtes (temporairement)  
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '  
log_checkpoints = on  
log_connections = on  
log_disconnections = on  
log_duration = off  
log_lock_waits = on  
log_temp_files = 0  
log_autovacuum_min_duration = 0  
```

**Attention** : `log_min_duration_statement = 0` génère BEAUCOUP de logs. À utiliser temporairement pour analyse.

### Autres outils d'analyse

#### 1. grep et scripts shell

**Exemple : Top 10 des erreurs**
```bash
grep "ERROR:" postgresql.log | \
  cut -d':' -f5- | sort | uniq -c | sort -rn | head -10
```

**Exemple : Temps moyen des requêtes par heure**
```bash
grep "duration:" postgresql.log | \
  awk '{print $1, $7}' | \
  awk '{h=substr($1,1,13); sum[h]+=$2; count[h]++}
       END {for (h in sum) print h, sum[h]/count[h]}' | sort
```

#### 2. Splunk / ELK Stack

Pour les grandes entreprises, les logs peuvent être envoyés vers :
- **Elasticsearch + Logstash + Kibana (ELK)**  
- **Splunk**  
- **Datadog Logs**  
- **Graylog**

Configuration pour JSON logs (PostgreSQL 15+) :
```conf
log_destination = 'jsonlog'  
logging_collector = on  
```

#### 3. Extensions PostgreSQL

**pg_stat_statements** : Statistiques agrégées des requêtes (vu précédemment)

```sql
SELECT
    query,
    calls,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements  
ORDER BY mean_exec_time DESC  
LIMIT 10;  
```

---

## Partie 7 : Bonnes Pratiques et Recommandations

### 1. Configuration par environnement

#### Développement
```conf
log_min_messages = info  
log_min_duration_statement = 100  
log_statement = 'all'  # Temporairement  
log_connections = on  
log_disconnections = on  

auto_explain.log_min_duration = 100  
auto_explain.log_analyze = on  
```

#### Production
```conf
log_min_messages = warning  
log_min_duration_statement = 5000  # 5 secondes  
log_statement = 'ddl'  
log_connections = on  
log_disconnections = on  
log_checkpoints = on  
log_lock_waits = on  
log_temp_files = 10240  # > 10 MB  

auto_explain.log_min_duration = 5000  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
auto_explain.sample_rate = 0.1  # 10%  
```

### 2. Rotation des logs

Sans rotation, les logs peuvent remplir le disque !

```conf
# Rotation automatique
logging_collector = on  
log_rotation_age = 1d          # Nouveau fichier chaque jour  
log_rotation_size = 100MB      # Nouveau fichier si > 100 MB  
log_truncate_on_rotation = on  # Écraser l'ancien fichier  

# Nom de fichier avec date
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
```

**Ou utiliser logrotate** (Linux) :

```bash
# /etc/logrotate.d/postgresql
/var/log/postgresql/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 postgres postgres
    sharedscripts
    postrotate
        /usr/bin/systemctl reload postgresql
    endscript
}
```

### 3. Surveiller la taille des logs

```bash
# Vérifier la taille du répertoire de logs
du -sh /var/log/postgresql/

# Surveiller en temps réel
watch -n 5 'du -sh /var/log/postgresql/'
```

**Alerte si** : Croissance > 1 GB/jour de manière inattendue.

### 4. Activer/Désactiver temporairement le logging verbeux

Pour une analyse ponctuelle sans redémarrage :

```sql
-- Augmenter temporairement le niveau de log
ALTER SYSTEM SET log_min_duration_statement = 0;  
SELECT pg_reload_conf();  

-- Analyser pendant quelques minutes...

-- Revenir à la normale
ALTER SYSTEM SET log_min_duration_statement = 5000;  
SELECT pg_reload_conf();  
```

**Ou au niveau session** :

```sql
-- Uniquement pour cette session
SET log_min_duration_statement = 0;  
SET client_min_messages = debug;  

-- Exécuter les requêtes à analyser...

-- Revenir à la normale
RESET log_min_duration_statement;  
RESET client_min_messages;  
```

### 5. Sécurité et confidentialité

**Attention** : Les logs peuvent contenir des données sensibles !

```conf
# Ne pas logger les mots de passe
log_statement = 'none'  # Évite de logger les CREATE USER avec mot de passe

# Restreindre l'accès aux logs
chmod 600 /var/log/postgresql/*.log  
chown postgres:postgres /var/log/postgresql/*.log  
```

**Anonymiser les logs** pour les partager :

```bash
# Remplacer les IPs
sed 's/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/XXX.XXX.XXX.XXX/g' \
  postgresql.log > postgresql-anonymized.log
```

### 6. Checklist de configuration des logs

- [ ] `logging_collector = on`  
- [ ] `log_destination` configuré (stderr ou csvlog)  
- [ ] `log_line_prefix` personnalisé avec contexte suffisant  
- [ ] `log_min_duration_statement` adapté à votre charge  
- [ ] `log_connections = on`  
- [ ] `log_disconnections = on`  
- [ ] `log_checkpoints = on`  
- [ ] `log_lock_waits = on`  
- [ ] `log_temp_files` configuré  
- [ ] Rotation des logs activée  
- [ ] `auto_explain` configuré et chargé  
- [ ] Tests de génération et lecture des logs  
- [ ] pgBadger installé et testé  
- [ ] Monitoring de la taille des logs

---

## Partie 8 : Dépannage et Questions Fréquentes

### Q1 : Les logs ne sont pas générés

**Solutions** :
```sql
-- Vérifier que le logging_collector est activé
SHOW logging_collector;

-- Vérifier le répertoire de logs
SHOW log_directory;  
SHOW log_filename;  

-- Vérifier les permissions
ls -la /var/log/postgresql/
```

### Q2 : auto_explain ne fonctionne pas

**Solutions** :
```sql
-- Vérifier que l'extension est chargée
SHOW shared_preload_libraries;

-- Vérifier la configuration
SHOW auto_explain.log_min_duration;

-- Forcer une requête lente pour tester
SELECT pg_sleep(2);
```

### Q3 : Les logs sont trop volumineux

**Solutions** :
1. Augmenter `log_min_duration_statement` (moins de requêtes loggées)  
2. Réduire `auto_explain.sample_rate` (échantillonnage)  
3. Désactiver `log_statement = 'all'`  
4. Configurer une rotation plus agressive

### Q4 : Comment analyser les logs en temps réel ?

```bash
# Suivre les logs en direct
tail -f /var/log/postgresql/postgresql-18-main.log

# Filtrer les erreurs uniquement
tail -f /var/log/postgresql/postgresql-18-main.log | grep ERROR

# Filtrer les requêtes lentes
tail -f /var/log/postgresql/postgresql-18-main.log | grep "duration:"
```

### Q5 : Comment exporter les logs vers un outil centralisé ?

**Option 1 : Filebeat (ELK)**
```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  paths:
    - /var/log/postgresql/*.log
output.elasticsearch:
  hosts: ["localhost:9200"]
```

**Option 2 : rsyslog**
```conf
# Dans postgresql.conf
log_destination = 'syslog'  
syslog_facility = 'LOCAL0'  
syslog_ident = 'postgres'  
```

---

## Conclusion

La configuration et l'analyse des logs PostgreSQL sont des compétences essentielles pour tout développeur ou administrateur de bases de données.

### Points clés à retenir

1. **log_line_prefix** : Personnalisez pour avoir le contexte nécessaire  
2. **log_min_duration_statement** : Le paramètre le plus important pour la performance  
3. **auto_explain** : Capture automatiquement les plans des requêtes lentes  
4. **pgBadger** : Outil d'analyse indispensable  
5. **Rotation** : Essentielle pour éviter de remplir le disque  
6. **Équilibre** : Loggez suffisamment, mais pas trop (overhead et taille)

### Configuration minimale recommandée

```conf
# Logging de base
logging_collector = on  
log_destination = 'stderr'  
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '  
log_min_duration_statement = 5000  
log_connections = on  
log_disconnections = on  
log_checkpoints = on  
log_lock_waits = on  
log_temp_files = 10240  

# Rotation
log_rotation_age = 1d  
log_rotation_size = 100MB  

# auto_explain
shared_preload_libraries = 'auto_explain'  
auto_explain.log_min_duration = 5000  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
auto_explain.sample_rate = 0.1  
```

### Workflow recommandé

1. **Configurer** les logs dès le début du projet  
2. **Monitorer** régulièrement avec pgBadger ou équivalent  
3. **Analyser** les patterns et anomalies  
4. **Optimiser** les requêtes identifiées comme problématiques  
5. **Itérer** : Ajuster la configuration selon vos besoins

### Ressources complémentaires

- [Documentation PostgreSQL - Error Reporting and Logging](https://www.postgresql.org/docs/18/runtime-config-logging.html)  
- [pgBadger Documentation](https://pgbadger.darold.net/)  
- [Guide auto_explain](https://www.postgresql.org/docs/18/auto-explain.html)  
- [PostgreSQL Log Analysis Best Practices](https://wiki.postgresql.org/wiki/Logging_Difficult_Queries)

---

**💡 Astuce finale** : Créez un script de configuration standardisé pour vos projets :

```bash
#!/bin/bash
# configure_logging.sh

cat >> /etc/postgresql/18/main/postgresql.conf <<EOF

# === Logging Configuration ===
logging_collector = on  
log_destination = 'stderr'  
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '  
log_min_duration_statement = 5000  
log_connections = on  
log_disconnections = on  
log_checkpoints = on  
log_lock_waits = on  
log_temp_files = 10240  
log_rotation_age = 1d  
log_rotation_size = 100MB  

# === auto_explain ===
shared_preload_libraries = 'auto_explain'  
auto_explain.log_min_duration = 5000  
auto_explain.log_analyze = on  
auto_explain.log_buffers = on  
auto_explain.sample_rate = 0.1  
EOF  

systemctl restart postgresql  
echo "Logging configuration complete!"  
```

**🎯 Objectif** : Des logs informatifs, des diagnostics rapides, des applications optimisées !

⏭️ [Métriques vitales](/14-observabilite-et-monitoring/06-metriques-vitales.md)
