🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe F : Impact sur Migration et Compatibilité PostgreSQL 18

## Introduction à la Migration

Migrer vers PostgreSQL 18 signifie passer d'une version plus ancienne (comme PostgreSQL 15, 16 ou 17) vers cette nouvelle version. Cette section vous aide à comprendre ce que cela implique pour vos applications et bases de données existantes.

---

## 🎯 Compatibilité Générale

### Verdict Global : Excellente Rétrocompatibilité

PostgreSQL 18 maintient une **excellente compatibilité** avec les versions précédentes. Cela signifie :

- ✅ **Vos applications fonctionneront** sans modification dans 95% des cas  
- ✅ **Votre code SQL existant** continuera à fonctionner  
- ✅ **Vos scripts et outils** resteront opérationnels  
- ✅ **Vos extensions** seront généralement compatibles

---

## 📊 Matrice de Compatibilité par Version Source

| Version Source | Compatibilité | Difficulté de Migration | Temps Estimé | Recommandation |
|----------------|---------------|-------------------------|--------------|----------------|
| **PostgreSQL 17** | ⭐⭐⭐⭐⭐ Excellente | 🟢 Très facile | 1-2 heures | ✅ Migration simple |
| **PostgreSQL 16** | ⭐⭐⭐⭐⭐ Excellente | 🟢 Facile | 2-4 heures | ✅ Migration recommandée |
| **PostgreSQL 15** | ⭐⭐⭐⭐ Très bonne | 🟡 Moyenne | 4-8 heures | ✅ Migration souhaitable |
| **PostgreSQL 14** | ⭐⭐⭐⭐ Très bonne | 🟡 Moyenne | 1-2 jours | ⚠️ Tester attentivement |
| **PostgreSQL 13 et antérieures** | ⭐⭐⭐ Bonne | 🟠 Plus complexe | 2-5 jours | ⚠️ Audit préalable requis |

> 💡 **Note** : Ces temps sont indicatifs pour une base de données de taille moyenne (< 100 GB). Les très grandes bases demanderont plus de temps.

---

## 🔍 Changements Impactants : Ce Qu'il Faut Savoir

### 1. Data Checksums Activés par Défaut ✅

**Qu'est-ce que c'est ?**
Les checksums sont comme des "codes de vérification" qui permettent à PostgreSQL de détecter si vos données ont été corrompues.

**Impact sur la migration :**
- ✅ **Nouvelles installations** : Activés automatiquement  
- ⚠️ **Migration depuis version antérieure** : Les checksums ne sont PAS activés automatiquement sur les bases migrées  
- 🔧 **Pour les activer après migration** : Nécessite une reconstruction complète (downtime)

**Exemple concret :**
```
# Vérifier si les checksums sont activés
SHOW data_checksums;

# Résultat possible :
# - "on" : Activé (recommandé)
# - "off" : Désactivé (situation après migration)
```

**Recommandation :**
- Pour les **nouvelles bases** : Profiter de l'activation par défaut
- Pour les **migrations** : Planifier l'activation lors d'une fenêtre de maintenance future
- Si vous ne pouvez pas avoir de downtime : Continuer sans checksums (fonctionnement normal)

---

### 2. Configuration du Sous-système I/O 🚀

**Qu'est-ce que c'est ?**
PostgreSQL 18 introduit un nouveau mode d'accès aux données sur disque (I/O asynchrone) beaucoup plus rapide.

**Impact sur la migration :**
- ✅ **Activation automatique** si votre système d'exploitation le supporte (Linux récent, Windows récent)  
- 🔧 **Nouveau paramètre** : `io_method` (valeurs : `'sync'`, `'worker'`, `'io_uring'`)  
- ⚡ **Performances** : Gain de 2-3× sur les opérations disque

**Exemple de configuration :**
```sql
-- Vérifier le mode I/O actuel
SHOW io_method;
-- Résultat : 'worker' (défaut PG 18) ou 'io_uring' (Linux) ou 'sync'

-- Si besoin, forcer le mode synchrone (pour compatibilité)
-- Dans postgresql.conf :
io_method = 'sync'
```

**Recommandation :**
- ✅ **Par défaut** : Laisser `'worker'` (défaut, multi-plateforme)  
- 🚀 **Linux** : Passer à `'io_uring'` pour performances maximales (kernel 5.1+)  
- ⚠️ **Problème de compatibilité** : Revenir en mode `'sync'`  
- 📊 **Mesurer** : Comparer les performances avant/après migration

---

### 3. Authentification : MD5 Déprécié (Encore Plus) 🔐

**Qu'est-ce que c'est ?**
MD5 est une ancienne méthode de chiffrement des mots de passe, moins sécurisée. PostgreSQL pousse vers SCRAM-SHA-256 (méthode moderne).

**Impact sur la migration :**
- ⚠️ **MD5 fonctionne encore** mais affiche des avertissements dans les logs  
- 🔒 **SCRAM-SHA-256** est maintenant le standard recommandé  
- 📝 **Fichier pg_hba.conf** : Doit être mis à jour

**Migration progressive** :

> ⚠️ **Le champ METHOD de `pg_hba.conf` n'accepte qu'UNE SEULE méthode par ligne**. La syntaxe `scram-sha-256,md5` séparée par virgule **n'existe pas**. Heureusement, la méthode `md5` côté `pg_hba.conf` **accepte aussi les hashes SCRAM stockés en base** — c'est le mécanisme de compatibilité que l'on exploite pendant la transition.

```ini
# Ancien (MD5 uniquement) — à remplacer progressivement
host    all    all    0.0.0.0/0    md5

# Nouveau (SCRAM uniquement) — recommandé à terme
host    all    all    0.0.0.0/0    scram-sha-256
```

**Plan de migration en 4 étapes** :

```ini
# Étape 1 — Configurer le serveur pour générer des hashes SCRAM
# Dans postgresql.conf :
password_encryption = scram-sha-256

# Étape 2 — Laisser pg_hba.conf en 'md5' pour le moment :
#   la méthode 'md5' accepte AUSSI les hashes SCRAM, donc la cohabitation
#   md5/SCRAM dans pg_authid est gérée de manière transparente.
host    all    all    0.0.0.0/0    md5

# Étape 3 — Régénérer chaque mot de passe en SCRAM :
ALTER USER alice PASSWORD 'son-nouveau-mdp';  
ALTER USER bob   PASSWORD 'son-nouveau-mdp';  
-- … pour tous les utilisateurs

# Étape 4 — Une fois tous les hashes en SCRAM (vérifier dans pg_authid.rolpassword),
#   basculer pg_hba.conf en scram-sha-256 strict :
host    all    all    0.0.0.0/0    scram-sha-256
```

**Vérifier où en est chaque rôle** :

```sql
SELECT rolname,
       CASE
         WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'SCRAM'
         WHEN rolpassword LIKE 'md5%'            THEN 'MD5 (à migrer)'
         ELSE 'autre / vide'
       END AS auth_method
FROM pg_authid  
WHERE rolcanlogin  
ORDER BY auth_method, rolname;  
```

**Commandes de migration des utilisateurs :**
```sql
-- Vérifier la méthode actuelle d'un utilisateur
SELECT rolname, rolpassword  
FROM pg_authid  
WHERE rolname = 'mon_utilisateur';  
-- Si commence par "md5" : Ancien format
-- Si commence par "SCRAM-SHA-256" : Nouveau format

-- Migrer un utilisateur vers SCRAM
ALTER USER mon_utilisateur WITH PASSWORD 'nouveau_mot_de_passe';
```

---

### 4. Autovacuum Plus Agressif 🧹

**Qu'est-ce que c'est ?**
L'autovacuum est un processus de nettoyage automatique qui récupère l'espace et maintient les performances.

**Impact sur la migration :**
- ⚡ **Plus de workers** : PostgreSQL 18 peut lancer plus de processus de nettoyage en parallèle  
- 📊 **Ajustement dynamique** : Le système s'adapte automatiquement à la charge  
- 💻 **Consommation CPU** : Peut augmenter légèrement après migration

**Nouveau paramètre PG 18 — `autovacuum_vacuum_max_threshold`** :

Ce paramètre fixe un **plafond absolu** (en lignes mortes) au-delà duquel `autovacuum` se déclenche, indépendamment du `scale_factor`. Utile pour les très grosses tables où `scale_factor × n_live_tup` deviendrait gigantesque.

```ini
# Dans postgresql.conf

# Nombre de workers autovacuum (statique, ne s'ajuste pas tout seul)
autovacuum_max_workers = 3                  # défaut

# 🆕 PG 18 : plafond absolu sur le déclenchement de VACUUM
autovacuum_vacuum_max_threshold = 100000000 # défaut : 100 millions de lignes mortes
                                            # mettre -1 pour désactiver le plafond
```

> 📌 Ce qui change *vraiment* en PG 18 côté autovacuum, c'est :  
> 1. l'apparition de `autovacuum_vacuum_max_threshold` (plafond absolu) ;  
> 2. une meilleure heuristique de répartition entre tables ;  
> 3. les bénéfices indirects de l'AIO (lectures plus rapides).  
>  
> Le paramètre `autovacuum_max_workers` reste statique : il faut toujours `reload` PostgreSQL pour le modifier.

**Recommandation :**
- ✅ **Laisser les valeurs par défaut** pour la plupart des cas  
- 📊 **Surveiller** la consommation CPU après migration  
- 🔧 **Ajuster si nécessaire** pour les systèmes à ressources limitées

---

### 5. Statistiques du Planificateur Préservées 📈

**Qu'est-ce que c'est ?**
Les statistiques aident PostgreSQL à choisir le meilleur plan d'exécution pour vos requêtes.

**Impact sur la migration :**
- ✅ **Grande amélioration** : Avant PG 18, ces statistiques étaient perdues lors d'un upgrade  
- ⚡ **Performances immédiates** : Plus besoin d'attendre que le système recalcule tout  
- 🎯 **Plans optimaux** dès le démarrage post-migration

**Avant PostgreSQL 18 :**
```
Jour 1 (migration) : Plans d'exécution sous-optimaux ❌  
Jour 2-7 : Amélioration progressive ⚠️  
Jour 8+ : Performances normales ✅  
```

**Avec PostgreSQL 18 :**
```
Jour 1 (migration) : Plans d'exécution optimaux immédiatement ✅
```

**Aucune action requise** : C'est automatique ! 🎉

---

## 🛠️ Stratégies de Migration

### Stratégie 1 : Migration In-Place (Sur Place) avec pg_upgrade

**Principe :**
Utiliser l'outil `pg_upgrade` pour mettre à jour votre instance existante.

**Avantages :**
- ✅ Rapide (minutes à heures au lieu de jours)  
- ✅ Préservation des statistiques (nouveau dans PG18)  
- ✅ Option `--swap` pour migration quasi-instantanée

**Inconvénients :**
- ⚠️ Nécessite un arrêt de service (downtime)  
- ⚠️ Nécessite de l'espace disque (2× la taille de la base dans certains cas)

**Temps d'arrêt estimé :**
- Base < 10 GB : 5-15 minutes
- Base 10-100 GB : 15-60 minutes
- Base 100 GB-1 TB : 1-4 heures
- Base > 1 TB : Utiliser `--swap` (quelques minutes)

**Modes de copie de pg_upgrade** :

| Mode | Comportement | Vitesse | Sécurité |
|---|---|---|---|
| *par défaut* | Copie complète des fichiers de l'ancien vers le nouveau cluster | Lent | Ancien cluster intact |
| `--link` | *Hard links* sur les fichiers (FS Unix) — pas de duplication | Rapide | Ancien cluster **inutilisable** ensuite |
| `--clone` | Reflink CoW (Btrfs, XFS reflink, APFS…) | Rapide | Ancien cluster reste démarrable |
| **`--swap`** 🆕 PG 18 | **Échange** les répertoires de données puis remplace les fichiers du catalogue | **Le plus rapide** | Ancien cluster **détruit** après commit |

**Exemple complet (mode par défaut, copie)** :

```bash
# 1. Arrêter l'ancien serveur PostgreSQL
pg_ctl stop -D /ancien/data

# 2. Test à blanc (TOUJOURS commencer par --check)
pg_upgrade \
  --old-datadir /ancien/data \
  --new-datadir /nouveau/data \
  --old-bindir  /ancien/bin \
  --new-bindir  /nouveau/bin \
  --jobs 4 \
  --check                   # ← vérifie la compatibilité sans rien modifier

# 3. Migration réelle (si le --check est OK)
pg_upgrade \
  --old-datadir /ancien/data \
  --new-datadir /nouveau/data \
  --old-bindir  /ancien/bin \
  --new-bindir  /nouveau/bin \
  --jobs 4

# 4. Démarrer le nouveau serveur
pg_ctl start -D /nouveau/data

# 5. Exécuter le script d'analyse des stats (généré par pg_upgrade)
./analyze_new_cluster.sh
```

**Avec l'option `--swap` (🆕 PG 18) — la plus rapide** :

```bash
# Test à blanc avec --check + --swap : vérifie aussi les contraintes du mode swap
pg_upgrade \
  --old-datadir /data \
  --new-datadir /data_nouveau \
  --old-bindir  /old/bin \
  --new-bindir  /new/bin \
  --jobs 4 \
  --swap \
  --check

# Migration réelle (après --check OK)
pg_upgrade \
  --old-datadir /data \
  --new-datadir /data_nouveau \
  --old-bindir  /old/bin \
  --new-bindir  /new/bin \
  --jobs 4 \
  --swap
```

> ⚠️ **Contraintes critiques de `--swap`** :  
>  
> - Les deux répertoires (`--old-datadir` et `--new-datadir`) doivent être **sur le même système de fichiers** (le swap déplace les inodes, pas de copie inter-FS).  
> - **Toujours combiner `--swap` avec `--check` lors du test à blanc** : certaines vérifications sont propres à ce mode et n'apparaissent qu'ainsi.  
> - Une fois le point de non-retour franchi, l'ancien cluster est **détruit** : impossible de revenir en arrière sans restaurer un backup. Faire un `pg_basebackup` ou un snapshot juste avant.  
> - Incompatible avec `--link` ou `--clone` (modes mutuellement exclusifs).

---

### Stratégie 2 : Migration Logique (Dump/Restore)

**Principe :**
Exporter toutes les données avec `pg_dump` puis les réimporter dans PostgreSQL 18.

**Avantages :**
- ✅ Très sûr (pas de risque de corruption)  
- ✅ Permet de "nettoyer" la base (réorganisation)  
- ✅ Pas de dépendance entre versions

**Inconvénients :**
- ⚠️ Très long pour les grosses bases  
- ⚠️ Downtime important  
- ⚠️ Nécessite beaucoup d'espace disque

**Temps d'arrêt estimé :**
- Base < 10 GB : 30-60 minutes
- Base 10-100 GB : 2-8 heures
- Base 100 GB-1 TB : 1-3 jours
- Base > 1 TB : Non recommandé (utiliser pg_upgrade)

**Exemple simplifié** :

```bash
# 1. Sauvegarde complète du cluster source
#    --globals-only ne suffit pas : on veut TOUT (rôles + bases + tablespaces + objets globaux)
sudo -u postgres pg_dumpall -h ancien_serveur -p 5432 \
    --no-password \
    --file=backup_complet.sql

# 2. Sur le nouveau serveur PG 18 : restauration via psql
#    -d postgres : on se connecte à la base 'postgres' (cible neutre, les autres bases
#                  sont (re)créées par les CREATE DATABASE contenus dans le dump)
#    -v ON_ERROR_STOP=1 : on stoppe à la première erreur (pas le comportement par défaut)
sudo -u postgres psql -h nouveau_serveur -p 5432 \
    -d postgres \
    -v ON_ERROR_STOP=1 \
    -f backup_complet.sql

# 3. Une fois la restauration terminée, lancer ANALYZE pour générer les stats
sudo -u postgres vacuumdb -h nouveau_serveur --all --analyze-only --jobs=4
```

> 💡 **Astuce volumétrie** : pour les bases > 50 Go, `pg_dump -F d -j N` (format directory + parallélisme) sur chaque base **individuellement**, puis `pg_dumpall --globals-only` pour les rôles/tablespaces, est généralement **plus rapide** qu'un `pg_dumpall` monolithique.

---

### Stratégie 3 : Réplication Logique (Zero-Downtime)

**Principe :**
Créer un serveur PostgreSQL 18 et y répliquer les données en continu, puis basculer.

**Avantages :**
- ✅ Zéro downtime (ou quelques secondes)  
- ✅ Possibilité de revenir en arrière facilement  
- ✅ Test de la nouvelle version en production

**Inconvénients :**
- ⚠️ Configuration complexe  
- ⚠️ Nécessite PostgreSQL 10+ comme source  
- ⚠️ Certaines limitations (séquences, DDL)

**Temps d'arrêt estimé :**
- Toutes tailles : < 30 secondes (bascule applicative)

**Étapes simplifiées :**
```sql
-- Sur l'ancien serveur (source)
CREATE PUBLICATION ma_publication FOR ALL TABLES;

-- Sur le nouveau serveur PostgreSQL 18 (cible)
CREATE SUBSCRIPTION ma_subscription
  CONNECTION 'host=ancien_serveur dbname=mabase'
  PUBLICATION ma_publication;

-- Attendre la synchronisation complète
-- Puis basculer les applications vers le nouveau serveur
```

---

## 📋 Checklist de Pré-Migration

Avant de migrer, assurez-vous de vérifier ces points :

### ✅ Préparation Technique

- [ ] **Sauvegarde complète** : Avoir une sauvegarde récente et testée  
- [ ] **Espace disque** : Vérifier l'espace disponible (minimum 2× la taille de la base)  
- [ ] **Version compatible** : Confirmer que vous êtes sur PostgreSQL 10+ minimum  
- [ ] **Extensions** : Vérifier la compatibilité de toutes les extensions tierces  
- [ ] **Outils** : S'assurer que pgAdmin, drivers applicatifs, etc. supportent PG18

### ✅ Préparation Applicative

- [ ] **Inventaire** : Lister toutes les applications qui se connectent à la base  
- [ ] **Tests** : Préparer un environnement de test avec PostgreSQL 18  
- [ ] **Drivers** : Mettre à jour les drivers de connexion (libpq, JDBC, psycopg3, etc.)  
- [ ] **Scripts** : Tester tous les scripts de maintenance et de déploiement  
- [ ] **Documentation** : Documenter la procédure de rollback (retour arrière)

### ✅ Préparation Organisationnelle

- [ ] **Fenêtre de maintenance** : Planifier un créneau avec faible activité  
- [ ] **Communication** : Informer les utilisateurs et équipes  
- [ ] **Équipe** : S'assurer d'avoir du support technique disponible  
- [ ] **Plan B** : Avoir une procédure de rollback documentée  
- [ ] **Monitoring** : Préparer les outils de surveillance post-migration

---

## 🧪 Tests Post-Migration Essentiels

Une fois la migration effectuée, effectuez ces vérifications :

### 1. Tests Fonctionnels de Base

```sql
-- Vérifier la version
SELECT version();
-- Doit afficher : PostgreSQL 18.x

-- Vérifier la disponibilité
SELECT now();

-- Tester une requête simple
SELECT COUNT(*) FROM ma_table_principale;
```

### 2. Tests de Performances

```sql
-- Activer le timing pour mesurer
\timing on

-- Exécuter vos requêtes les plus fréquentes
SELECT * FROM commandes WHERE date > CURRENT_DATE - 30;

-- Comparer avec les temps d'avant migration
```

### 3. Tests de Connectivité Applicative

- [ ] Tester chaque application se connectant à la base  
- [ ] Vérifier les logs d'erreurs applicatives  
- [ ] Valider les fonctionnalités critiques métier  
- [ ] Tester les jobs automatisés et scripts cron

### 4. Vérifications Système

```sql
-- Vérifier l'autovacuum
SELECT * FROM pg_stat_progress_vacuum;

-- Vérifier les connexions
SELECT COUNT(*) FROM pg_stat_activity;

-- Vérifier les extensions
SELECT * FROM pg_extension;

-- Vérifier la réplication (si applicable)
SELECT * FROM pg_stat_replication;
```

---

## ⚠️ Problèmes Courants et Solutions

### Problème 1 : "MD5 authentication is deprecated"

**Symptôme :**
Messages d'avertissement dans les logs concernant MD5.

**Solution :** suivre le plan de migration progressif décrit plus haut (section « Plan de migration progressif md5 → scram-sha-256 »). Le principe — rappelé ici :

```bash
# pg_hba.conf : conserver la méthode 'md5' pendant la transition.
# IMPORTANT : pg_hba.conf n'accepte qu'UNE SEULE méthode par ligne
# (la syntaxe 'scram-sha-256,md5' n'existe pas).
# La méthode 'md5' accepte aussi les hashes SCRAM stockés en base,
# donc on garde 'md5' le temps de re-hasher tous les mots de passe.
host all all 0.0.0.0/0 md5
```

```sql
-- 1. Côté serveur : produire les nouveaux hashes en SCRAM-SHA-256
ALTER SYSTEM SET password_encryption = 'scram-sha-256';  
SELECT pg_reload_conf();  

-- 2. Demander aux utilisateurs de RE-SAISIR leur mot de passe
--    (ALTER ROLE … PASSWORD 'xxx' avec password_encryption=scram-sha-256
--     stocke un hash SCRAM dans pg_authid)
ALTER USER utilisateur1 WITH PASSWORD 'nouveau_mdp';

-- 3. Une fois TOUS les utilisateurs migrés, basculer pg_hba.conf
--    sur 'scram-sha-256' pour refuser explicitement les vrais md5.
-- host all all 0.0.0.0/0 scram-sha-256
```

---

### Problème 2 : Performances Initiales Dégradées

**Symptôme :**
Requêtes plus lentes juste après la migration.

**Solution :**
```sql
-- Forcer la mise à jour des statistiques
ANALYZE VERBOSE;

-- Pour chaque table importante
VACUUM ANALYZE nom_table;

-- Vérifier que les index sont valides
SELECT schemaname, tablename, indexname  
FROM pg_indexes  
WHERE schemaname NOT IN ('pg_catalog', 'information_schema');  
```

---

### Problème 3 : Extensions Non Disponibles

**Symptôme :**
Erreur "extension X does not exist" après migration.

**Solution :**
```sql
-- Lister les extensions disponibles
SELECT * FROM pg_available_extensions  
ORDER BY name;  

-- Si l'extension manque, l'installer
-- Sur le système (en tant que root/admin)
apt-get install postgresql-18-extension-nom  -- Debian/Ubuntu  
yum install postgresql18-extension-nom       -- RedHat/CentOS  

-- Puis dans PostgreSQL
CREATE EXTENSION nom_extension;
```

---

### Problème 4 : Consommation Mémoire Élevée

**Symptôme :**
PostgreSQL utilise plus de RAM qu'avant.

**Solution :**
```sql
-- Vérifier la configuration mémoire
SHOW shared_buffers;  
SHOW work_mem;  
SHOW maintenance_work_mem;  

-- Ajuster si nécessaire dans postgresql.conf
-- Règle générale : shared_buffers = 25% de la RAM totale
shared_buffers = 4GB           -- Exemple pour 16GB RAM  
work_mem = 64MB                -- Par opération de tri  
maintenance_work_mem = 512MB   -- Pour VACUUM, CREATE INDEX  
```

```bash
# Redémarrer PostgreSQL (shared_buffers nécessite un restart, pas un reload)
# pg_ctl exige le répertoire de données via -D (ou la variable PGDATA)
pg_ctl -D "$PGDATA" restart
# Ou avec systemd :
# sudo systemctl restart postgresql
```

---

## 🎯 Recommandations par Profil

### Pour les Développeurs 👨‍💻

**Priorité : Compatibilité applicative**

- ✅ Mettre à jour les drivers (psycopg3, node-pg, JDBC, etc.)  
- ✅ Tester toutes les fonctionnalités en environnement de dev  
- ✅ Profiter des nouvelles fonctionnalités (colonnes virtuelles, UUIDv7)  
- ⚠️ Vérifier les requêtes avec OR multiples (optimisées automatiquement)

**Timeline recommandée :**
- Mois 1 : Environnement de développement en PG18
- Mois 2 : Tests intensifs et correction de bugs
- Mois 3 : Migration staging/pré-production
- Mois 4-6 : Migration production

---

### Pour les DevOps/SRE 🔧

**Priorité : Stabilité et performances**

- ✅ Planifier la stratégie de migration (pg_upgrade vs réplication)  
- ✅ Configurer le monitoring post-migration  
- ✅ Valider les sauvegardes et procédures de restauration  
- ✅ Documenter la procédure de rollback  
- ⚠️ Surveiller l'autovacuum (plus agressif)

**Timeline recommandée :**
- Mois 1 : Audit de compatibilité et tests
- Mois 2-3 : Migration environnements non-prod
- Mois 4-6 : Migration production par étapes

---

### Pour les DBA 👨‍🔬

**Priorité : Intégrité et optimisation**

- ✅ Activer les data checksums sur nouvelles installations  
- ✅ Migrer MD5 vers SCRAM-SHA-256  
- ✅ Optimiser la configuration I/O (async)  
- ✅ Profiter des statistiques préservées  
- ⚠️ Surveiller le bloat et l'espace disque

**Timeline recommandée :**
- Semaine 1-2 : Analyse de compatibilité détaillée
- Semaine 3-4 : Tests de charge et benchmarks
- Mois 2 : Migration d'une base pilote
- Mois 3-6 : Migration progressive des autres bases

---

## 📊 Tableau Récapitulatif : Impact par Fonctionnalité

| Fonctionnalité | Impact Migration | Action Requise | Urgence |
|----------------|------------------|----------------|---------|
| **I/O Asynchrone** | 🟢 Transparent | Aucune (automatique) | ✅ Aucune |
| **Data Checksums** | 🟡 Partiel | Activer manuellement sur bases migrées | 🔶 Souhaitable |
| **SCRAM vs MD5** | 🟠 Moyen | Migrer progressivement | ⚠️ Important |
| **Autovacuum** | 🟢 Transparent | Surveiller CPU | ✅ Aucune |
| **pg_upgrade --swap** | 🟢 Transparent | Utiliser pour grosses bases | ✅ Recommandé |
| **Statistiques préservées** | 🟢 Transparent | Aucune (automatique) | ✅ Aucune |
| **OAuth 2.0** | 🟢 Optionnel | Configurer si souhaité | 🔵 Optionnel |
| **UUIDv7** | 🟢 Optionnel | Modifier code pour utilisation | 🔵 Optionnel |
| **Colonnes Virtuelles** | 🟢 Optionnel | Modifier schéma pour utilisation | 🔵 Optionnel |

**Légende :**
- 🟢 Transparent : Aucun impact négatif  
- 🟡 Partiel : Impact limité et gérable  
- 🟠 Moyen : Nécessite attention et planification

---

## 🚀 Feuille de Route Type de Migration

### Phase 1 : Préparation (2-4 semaines)

**Semaine 1-2 : Audit**
- Inventaire des versions, extensions, taille des bases
- Identification des dépendances applicatives
- Évaluation de la compatibilité

**Semaine 3-4 : Planification**
- Choix de la stratégie de migration
- Planification des fenêtres de maintenance
- Préparation de l'environnement de test

---

### Phase 2 : Tests (3-6 semaines)

**Semaine 5-6 : Tests Fonctionnels**
- Installation PostgreSQL 18 en environnement de test
- Migration d'une copie de la base de production
- Tests fonctionnels applicatifs

**Semaine 7-9 : Tests de Performance**
- Benchmarks de performance (avant/après)
- Tests de charge
- Identification des régressions éventuelles

**Semaine 10 : Tests de Rollback**
- Simulation de problèmes
- Tests de restauration
- Validation de la procédure de retour arrière

---

### Phase 3 : Migration Production (1-4 semaines)

**Option A : Downtime Acceptable**
- Fenêtre de maintenance planifiée
- Migration avec pg_upgrade
- Durée : quelques heures

**Option B : Zero Downtime Requis**
- Mise en place réplication logique
- Bascule progressive des applications
- Durée : 1-2 semaines

---

### Phase 4 : Post-Migration (2-4 semaines)

**Semaine 1-2 : Surveillance Active**
- Monitoring intensif
- Correction d'anomalies
- Optimisation fine

**Semaine 3-4 : Stabilisation**
- Retour à la normale
- Documentation des leçons apprises
- Bilan de migration

---

## 💡 Conseils d'Expert

### DO ✅

1. **Toujours avoir une sauvegarde** : Avant toute migration, avoir une sauvegarde complète testée  
2. **Tester en dev d'abord** : Ne jamais migrer directement en production  
3. **Commencer petit** : Migrer d'abord les bases non-critiques  
4. **Surveiller attentivement** : Les 48 premières heures sont critiques  
5. **Documenter tout** : Chaque étape, chaque problème rencontré

### DON'T ❌

1. **Ne pas précipiter** : Une migration mal préparée peut causer des interruptions majeures  
2. **Ne pas ignorer les warnings** : Les avertissements dans les logs sont importants  
3. **Ne pas oublier les extensions** : Vérifier qu'elles sont toutes compatibles  
4. **Ne pas négliger les tests** : Les tests de charge sont essentiels  
5. **Ne pas migrer pendant les pics** : Choisir une période de faible activité

---

## 🎓 Conclusion

La migration vers PostgreSQL 18 est généralement **simple et sûre** grâce à :

- ✅ L'excellente rétrocompatibilité  
- ✅ Les outils de migration améliorés (pg_upgrade avec --swap)  
- ✅ La préservation des statistiques  
- ✅ Les gains de performance immédiats

**Trois règles d'or :**
1. **Préparer** : Audit, tests, planification  
2. **Tester** : En dev, en staging, puis en production  
3. **Surveiller** : Monitoring actif post-migration

Avec une préparation adéquate, votre migration sera un succès ! 🎉

---

## 📚 Ressources Complémentaires

- **Chapitre 19.3** : Détails techniques sur pg_upgrade  
- **Chapitre 14** : Observabilité et monitoring post-migration  
- **Chapitre 16** : Configuration et tuning pour PostgreSQL 18  
- **Documentation officielle** : [postgresql.org/docs/18/upgrading.html](https://www.postgresql.org/docs/18/upgrading.html)

---


⏭️ [Recommandations d'adoption](/annexes/nouveautes-pg18/03-recommandations-adoption.md)
