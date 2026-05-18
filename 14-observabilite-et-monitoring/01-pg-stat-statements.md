🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.1. La "Super Extension" : pg_stat_statements

## Introduction

Dans le monde de l'observabilité des bases de données, **pg_stat_statements** est souvent considérée comme la "super extension" de PostgreSQL. Pourquoi ? Parce qu'elle vous permet de répondre à une question fondamentale : **"Quelles sont les requêtes qui consomment le plus de ressources dans ma base de données ?"**

Sans cette extension, vous êtes un peu comme un pilote d'avion sans tableau de bord : vous savez que le moteur tourne, mais vous ne savez pas vraiment ce qui se passe sous le capot. Avec pg_stat_statements, vous obtenez une visibilité complète sur les performances de vos requêtes SQL.

### Pourquoi pg_stat_statements est indispensable ?

Imaginez que votre application ralentit. Les utilisateurs se plaignent. Quelle est la cause ? Sans outils, vous devrez :
- Deviner quelles requêtes posent problème
- Analyser manuellement les logs (si vous les avez activés)
- Espérer tomber sur la bonne requête au bon moment

Avec pg_stat_statements, vous pouvez immédiatement identifier :
- Les requêtes les plus lentes
- Les requêtes exécutées le plus fréquemment
- Les requêtes qui consomment le plus de CPU ou d'I/O
- L'évolution des performances dans le temps

C'est un outil de diagnostic essentiel pour tout développeur ou administrateur de bases de données.

---

## Qu'est-ce que pg_stat_statements ?

### Définition simple

**pg_stat_statements** est une extension PostgreSQL qui collecte et stocke des statistiques d'exécution pour toutes les requêtes SQL exécutées sur votre serveur. Elle fonctionne en arrière-plan et agrège automatiquement les informations.

### Ce qu'elle capture

Pour chaque requête unique (normalisée), pg_stat_statements enregistre :

- **Nombre d'exécutions** : Combien de fois la requête a été exécutée  
- **Temps d'exécution** : Temps total, moyen, minimum et maximum  
- **Lignes traitées** : Nombre de lignes lues, insérées, mises à jour ou supprimées  
- **Blocs I/O** : Accès disque (lectures, écritures, cache hits)  
- **Timestamp** : Dernière exécution

### Normalisation des requêtes

Un concept clé à comprendre : **la normalisation**.

Considérez ces deux requêtes :
```sql
SELECT * FROM users WHERE id = 1;  
SELECT * FROM users WHERE id = 42;  
```

Même si les valeurs sont différentes, ce sont structurellement la même requête. pg_stat_statements les **normalise** en remplaçant les valeurs littérales par des placeholders :
```sql
SELECT * FROM users WHERE id = $1;
```

Cela permet d'agréger les statistiques pour toutes les variantes d'une même requête, ce qui est beaucoup plus utile pour l'analyse.

---

## Installation de pg_stat_statements

### Étape 1 : Vérifier la disponibilité de l'extension

pg_stat_statements est fournie avec PostgreSQL par défaut depuis la version 9.2, mais elle doit être activée manuellement.

Pour vérifier si l'extension est disponible sur votre système, connectez-vous à PostgreSQL avec `psql` :

```sql
SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements';
```

Si vous voyez une ligne avec `pg_stat_statements`, l'extension est disponible. Sinon, vous devrez peut-être installer le paquet `postgresql-contrib` :

**Sur Ubuntu/Debian :**
```bash
sudo apt-get install postgresql-contrib-18
```

**Sur CentOS/RHEL :**
```bash
sudo yum install postgresql18-contrib
```

**Sur macOS (avec Homebrew) :**
```bash
brew install postgresql@18
```

### Étape 2 : Modifier le fichier de configuration postgresql.conf

L'extension pg_stat_statements doit être chargée au démarrage du serveur. Pour cela, vous devez modifier le fichier de configuration principal de PostgreSQL : **postgresql.conf**.

#### Localiser postgresql.conf

La localisation dépend de votre installation :
- **Linux** : Généralement dans `/etc/postgresql/18/main/postgresql.conf` ou `/var/lib/pgsql/18/data/postgresql.conf`  
- **macOS (Homebrew)** : `/opt/homebrew/var/postgresql@18/postgresql.conf`  
- **Windows** : `C:\Program Files\PostgreSQL\18\data\postgresql.conf`

Pour trouver le chemin exact depuis psql :
```sql
SHOW config_file;
```

#### Éditer postgresql.conf

Ouvrez le fichier avec un éditeur de texte (vous aurez besoin des droits administrateur) :

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

Recherchez la ligne contenant `shared_preload_libraries` (elle peut être commentée avec un `#` au début). Modifiez-la pour inclure `pg_stat_statements` :

```conf
shared_preload_libraries = 'pg_stat_statements'
```

**Si d'autres extensions sont déjà présentes**, ajoutez pg_stat_statements à la liste séparée par des virgules :

```conf
shared_preload_libraries = 'pg_stat_statements,auto_explain'
```

### Étape 3 : Configurer les paramètres de pg_stat_statements

Toujours dans `postgresql.conf`, vous pouvez ajouter des paramètres optionnels pour contrôler le comportement de l'extension. Voici les plus importants :

```conf
# Nombre maximum de requêtes distinctes à suivre (par défaut: 5000)
pg_stat_statements.max = 10000

# Niveau de suivi : 'top' (défaut), 'all' (inclut requêtes imbriquées), 'none'
pg_stat_statements.track = all

# Suivre aussi le temps de planification (par défaut: off — léger overhead)
pg_stat_statements.track_planning = on

# Suivre les commandes utilitaires (VACUUM, ANALYZE, etc.) — par défaut: on
pg_stat_statements.track_utility = on

# Sauvegarder les statistiques sur disque lors de l'arrêt (par défaut: on)
pg_stat_statements.save = on
```

#### Explication des paramètres clés

**pg_stat_statements.max**
- Définit le nombre maximum de requêtes uniques à suivre
- Si ce nombre est dépassé, les requêtes les moins utilisées sont supprimées
- Recommandation : 10 000 pour un serveur de production actif

**pg_stat_statements.track**
- `top` : Suit uniquement les requêtes de niveau supérieur (par défaut)  
- `all` : Suit aussi les requêtes exécutées dans les fonctions PL/pgSQL  
- `none` : Désactive le suivi
- Recommandation : `all` pour une visibilité complète

**pg_stat_statements.track_planning**
- Active le suivi du temps passé à planifier les requêtes
- Utile pour détecter les requêtes avec des plans complexes
- Introduit une légère surcharge de performance
- Recommandation : `on` en production moderne (PostgreSQL 18)

**pg_stat_statements.track_utility**
- Suit les commandes utilitaires (toutes sauf SELECT/INSERT/UPDATE/DELETE/MERGE) : VACUUM, ANALYZE, CREATE INDEX, COPY, etc.
- **Activé par défaut** (`on`) — rien à changer dans la plupart des cas
- Si désactivé, vous ne verriez pas les commandes de maintenance dans les statistiques

### Étape 4 : Redémarrer PostgreSQL

Les modifications de `shared_preload_libraries` nécessitent un **redémarrage complet** du serveur PostgreSQL (un simple reload avec `pg_ctl reload` ne suffit pas).

**Sur Linux (systemd) :**
```bash
sudo systemctl restart postgresql
```

**Sur Linux (pg_ctl) :**
```bash
sudo pg_ctl restart -D /var/lib/pgsql/18/data
```

**Sur macOS (Homebrew) :**
```bash
brew services restart postgresql@18
```

**Vérifier que le redémarrage s'est bien passé :**
```bash
sudo systemctl status postgresql
```

### Étape 5 : Créer l'extension dans votre base de données

Une fois le serveur redémarré, connectez-vous à la base de données où vous souhaitez utiliser pg_stat_statements :

```sql
-- Se connecter à votre base de données
\c ma_base_de_donnees

-- Créer l'extension
CREATE EXTENSION pg_stat_statements;
```

**Important** : L'extension doit être créée dans **chaque base de données** où vous souhaitez collecter des statistiques. Cependant, les statistiques sont partagées au niveau de l'instance entière (toutes les bases).

### Étape 6 : Vérifier l'installation

Pour confirmer que tout fonctionne, exécutez :

```sql
SELECT * FROM pg_stat_statements LIMIT 5;
```

Si vous voyez des lignes de résultats (ou une table vide si aucune requête n'a encore été exécutée depuis l'installation), l'extension est correctement installée !

Vous pouvez aussi vérifier que l'extension est chargée au niveau du serveur :

```sql
SHOW shared_preload_libraries;
```

Vous devriez voir `pg_stat_statements` dans la liste.

---

## Configuration avancée (optionnelle pour débutants)

### Ajuster la taille mémoire allouée

pg_stat_statements utilise de la mémoire partagée pour stocker les statistiques. La quantité allouée est proportionnelle à `pg_stat_statements.max` — chaque entrée occupe quelques centaines d'octets. Le calcul est fait automatiquement au démarrage du serveur (mémoire dimensionnée à partir de `pg_stat_statements.max`), aucune autre variable n'est à régler.

### Ajuster la granularité de la normalisation

Par défaut, pg_stat_statements normalise complètement les requêtes. Vous pouvez contrôler ce comportement avec le paramètre `pg_stat_statements.track` :

```conf
# Niveau de suivi (par défaut: 'top')
pg_stat_statements.track = 'all'  # Recommandé pour la plupart des cas
```

---

## Évolutions récentes des colonnes (PG 17 / PG 18)

Au fil des versions, pg_stat_statements s'est enrichie de nouvelles colonnes utiles. Quelques points à connaître quand on lit une formation/script ancien :

### Renommage important en PG 17

Les colonnes de temps I/O ont été **scindées par type de bloc** : `blk_read_time` et `blk_write_time` n'existent plus telles quelles. Elles sont maintenant :

| Type de bloc | Lecture | Écriture |
|--------------|---------|----------|
| **Partagés** (shared buffers) | `shared_blk_read_time` | `shared_blk_write_time` |
| **Locaux** (temp tables session) | `local_blk_read_time` | `local_blk_write_time` |
| **Temporaires** (tris, hash) | `temp_blk_read_time` | `temp_blk_write_time` |

Toute requête qui utilisait `blk_read_time + blk_write_time` doit donc être adaptée.

### Nouvelles colonnes PG 17

- **`stats_since`** : timestamp de création/réinitialisation de l'entrée (utile pour calculer un taux : `calls / (now() - stats_since)`).
- **`minmax_stats_since`** : timestamp du dernier reset des colonnes `min_*_time` / `max_*_time` (via `pg_stat_statements_reset(..., minmax_only := true)`).
- Tracking I/O scindé par type de bloc (voir ci-dessus).

### Nouvelles colonnes PG 18

- **`parallel_workers_to_launch`** : nombre de workers parallèles que le planner a demandé pour cette requête (cumulé sur tous les appels).
- **`parallel_workers_launched`** : nombre de workers parallèles réellement lancés. Si `launched < to_launch`, la requête a été bridée par `max_parallel_workers` ou `max_parallel_workers_per_gather`.
- **`wal_buffers_full`** : nombre de fois où le buffer WAL a dû être vidé en cours d'exécution parce qu'il était plein (signal indicateur d'un `wal_buffers` sous-dimensionné).
- **`jit_deform_count`** / **`jit_deform_time`** : statistiques JIT pour la désérialisation des tuples (compilation JIT du tuple deforming).

**Exemple — détecter une requête bridée par le pool de workers :**

```sql
SELECT
    LEFT(query, 60)                         AS query_preview,
    calls,
    parallel_workers_to_launch              AS demandes,
    parallel_workers_launched               AS lances,
    parallel_workers_to_launch
        - parallel_workers_launched         AS manquants
FROM pg_stat_statements  
WHERE parallel_workers_to_launch > 0  
ORDER BY manquants DESC  
LIMIT 10;  
```

**Exemple — repérer un `wal_buffers` sous-dimensionné :**

```sql
SELECT
    LEFT(query, 80) AS query_preview,
    calls,
    wal_records,
    pg_size_pretty(wal_bytes) AS wal_size,
    wal_buffers_full
FROM pg_stat_statements  
WHERE wal_buffers_full > 0  
ORDER BY wal_buffers_full DESC  
LIMIT 10;  
```

Un `wal_buffers_full` élevé sur des requêtes générant beaucoup de WAL suggère d'augmenter `wal_buffers` (souvent : passer du défaut `-1` à `64MB` ou plus).

---

## Vérification de la configuration

Une fois l'installation terminée, voici un récapitulatif des vérifications à effectuer :

### 1. Vérifier que l'extension est chargée
```sql
SHOW shared_preload_libraries;
-- Devrait afficher : pg_stat_statements (et éventuellement d'autres extensions)
```

### 2. Vérifier que l'extension existe dans la base de données
```sql
\dx pg_stat_statements
-- ou
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';
```

### 3. Vérifier que des statistiques sont collectées
```sql
SELECT COUNT(*) FROM pg_stat_statements;
-- Devrait retourner un nombre > 0 si des requêtes ont été exécutées
```

### 4. Vérifier les paramètres actuels
```sql
SELECT name, setting, unit  
FROM pg_settings  
WHERE name LIKE 'pg_stat_statements%';  
```

Exemple de sortie :
```
               name                | setting | unit
-----------------------------------+---------+------
 pg_stat_statements.max            | 10000   |
 pg_stat_statements.save           | on      |
 pg_stat_statements.track          | all     |
 pg_stat_statements.track_planning | on      |
 pg_stat_statements.track_utility  | on      |
```

---

## Dépannage : Problèmes courants

### Problème 1 : "ERROR: could not load library pg_stat_statements"

**Cause** : L'extension n'est pas installée sur le système ou le chemin n'est pas correct.

**Solution** :
- Installez le paquet `postgresql-contrib` (voir Étape 1)
- Vérifiez que le fichier `pg_stat_statements.so` existe dans le répertoire des bibliothèques PostgreSQL :
  ```bash
  sudo find /usr -name pg_stat_statements.so
  ```

### Problème 2 : "FATAL: could not access file 'pg_stat_statements': No such file"

**Cause** : Le fichier de bibliothèque partagée n'est pas dans le bon répertoire.

**Solution** : Réinstallez le paquet contrib ou vérifiez votre installation PostgreSQL.

### Problème 3 : L'extension ne collecte pas de statistiques

**Cause** : L'extension n'est pas créée dans la base de données.

**Solution** :
```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Problème 4 : "shared_preload_libraries" n'a pas d'effet

**Cause** : Le serveur n'a pas été redémarré après la modification de postgresql.conf.

**Solution** : Redémarrez PostgreSQL (pas seulement reload) :
```bash
sudo systemctl restart postgresql
```

### Problème 5 : Permission denied lors de la création de l'extension

**Cause** : Vous n'avez pas les droits superuser.

**Solution** : Connectez-vous en tant que superuser (postgres) :
```bash
sudo -u postgres psql
```

---

## Sécurité et bonnes pratiques

### Qui peut voir les statistiques ?

Par défaut, seuls les **superusers** peuvent consulter toutes les statistiques de pg_stat_statements. Les utilisateurs normaux ne voient que leurs propres requêtes.

Pour donner accès à un utilisateur non-superuser (par exemple, un outil de monitoring) :

```sql
GRANT pg_read_all_stats TO mon_utilisateur_monitoring;
```

### Impact sur les performances

pg_stat_statements a un **impact minimal** sur les performances (généralement < 5% de overhead). Cependant :
- Évitez de définir `pg_stat_statements.max` à des valeurs très élevées (> 50 000) sans raison
- Le paramètre `track_planning` peut ajouter un léger overhead supplémentaire

### Nettoyer les anciennes statistiques

Les statistiques s'accumulent indéfiniment jusqu'à atteindre `pg_stat_statements.max`. Pour réinitialiser les statistiques :

```sql
SELECT pg_stat_statements_reset();
```

**Attention** : Cette commande supprime **toutes** les statistiques collectées depuis le dernier redémarrage ou reset.

Pour réinitialiser uniquement les statistiques d'une requête spécifique (PostgreSQL 12+) :

```sql
SELECT pg_stat_statements_reset(userid, dbid, queryid);
```

À partir de **PostgreSQL 15**, on peut aussi ne réinitialiser que les compteurs min/max (utile pour mesurer une période sans perdre l'historique complet) :

```sql
-- Reset SEULEMENT des min_*_time / max_*_time, garde le reste
SELECT pg_stat_statements_reset(minmax_only := true);

-- Équivalent en arguments positionnels
SELECT pg_stat_statements_reset(0, 0, 0, true);
```

Le timestamp précédent du reset min/max est renvoyé par la fonction, et stocké aussi dans `minmax_stats_since` (PG 17+) de chaque entrée.

### Surveiller l'état de l'extension : `pg_stat_statements_info`

Une vue souvent oubliée mais critique : **`pg_stat_statements_info`** (PG 14+) qui décrit l'état de l'extension elle-même.

```sql
SELECT * FROM pg_stat_statements_info;
```

| Colonne | Signification |
|---------|---------------|
| `dealloc` | Nombre de fois où PostgreSQL a dû **supprimer des entrées** (LRU) parce que `pg_stat_statements.max` était atteint |
| `stats_reset` | Date du dernier reset des statistiques |

**Pourquoi c'est important ?** Si `dealloc` croît, cela signifie que **votre `pg_stat_statements.max` est trop petit** : des requêtes sont évincées avant que vous ayez le temps de les voir, et certaines statistiques sont incomplètes.

```sql
-- Alerter si dealloc commence à croître
SELECT
    dealloc,
    stats_reset,
    age(NOW(), stats_reset) AS depuis_reset,
    CASE
        WHEN dealloc = 0 THEN '🟢 OK — max suffisant'
        WHEN dealloc < 100 THEN '🟡 Surveiller — quelques évictions'
        ELSE '🔴 max trop petit, augmenter pg_stat_statements.max'
    END AS statut
FROM pg_stat_statements_info;
```

> 💡 **Diagnostic concret** : si `dealloc > 0` quelques minutes après un reset, c'est que `pg_stat_statements.max` (défaut 5000 en PG 18) est dépassé. Doubler la valeur (`max = 10000`) et redémarrer le serveur.

---

## Résumé : Checklist d'installation

Voici un récapitulatif rapide des étapes d'installation :

- [ ] **Étape 1** : Vérifier que le paquet contrib est installé  
- [ ] **Étape 2** : Ajouter `pg_stat_statements` à `shared_preload_libraries` dans postgresql.conf  
- [ ] **Étape 3** : Configurer les paramètres optionnels (max, track, track_planning, track_utility)  
- [ ] **Étape 4** : Redémarrer PostgreSQL  
- [ ] **Étape 5** : Créer l'extension avec `CREATE EXTENSION pg_stat_statements;`  
- [ ] **Étape 6** : Vérifier avec `SELECT * FROM pg_stat_statements LIMIT 5;`

---

## Exemple de configuration complète

Voici un exemple de configuration recommandée pour PostgreSQL 18 dans `postgresql.conf` :

```conf
# === Chargement de l'extension ===
shared_preload_libraries = 'pg_stat_statements'

# === Configuration de pg_stat_statements ===
pg_stat_statements.max = 10000              # Nombre max de requêtes distinctes  
pg_stat_statements.track = all              # Suit toutes les requêtes (y compris dans les fonctions)  
pg_stat_statements.track_planning = on      # Inclut le temps de planification  
pg_stat_statements.track_utility = on       # Suit VACUUM, ANALYZE, etc.  
pg_stat_statements.save = on                # Sauvegarde sur disque à l'arrêt  
```

Après avoir édité ce fichier, n'oubliez pas de redémarrer PostgreSQL !

---

## Prochaines étapes

Maintenant que pg_stat_statements est installé et configuré, vous êtes prêt à l'utiliser pour analyser les performances de vos requêtes. Dans les sections suivantes (14.2 et au-delà), nous verrons :

- Comment interroger pg_stat_statements pour trouver les requêtes lentes
- Comment identifier les requêtes qui consomment le plus de ressources
- Comment combiner pg_stat_statements avec d'autres vues système (pg_stat_activity, pg_locks)
- Comment utiliser pg_stat_statements dans vos outils de monitoring (Grafana, pgBadger)

L'extension est maintenant votre alliée principale pour comprendre et optimiser les performances de votre base de données PostgreSQL !

---

**🎯 Points clés à retenir :**

1. **pg_stat_statements** est une extension essentielle pour le monitoring des performances SQL  
2. Elle **normalise** les requêtes pour agréger les statistiques de manière pertinente  
3. L'installation nécessite une modification de `postgresql.conf` et un **redémarrage** du serveur  
4. L'extension capture automatiquement le temps d'exécution, les I/O, et le nombre d'exécutions  
5. Elle a un **impact minimal** sur les performances (< 5%)  
6. Elle doit être créée dans chaque base de données avec `CREATE EXTENSION`  
7. Les paramètres recommandés pour PostgreSQL 18 : `max=10000`, `track=all`, `track_planning=on`

**💡 Conseil de pro :**
Installez pg_stat_statements dès le début de tout nouveau projet PostgreSQL, même en développement. Vous serez reconnaissant de l'avoir fait lorsque vous devrez optimiser vos requêtes plus tard !

⏭️ [Vues système essentielles](/14-observabilite-et-monitoring/02-vues-systeme-essentielles.md)
