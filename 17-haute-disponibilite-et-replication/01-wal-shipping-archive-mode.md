🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.1. Le concept de WAL Shipping et archive_mode

## Introduction

Avant de plonger dans les mécanismes de haute disponibilité de PostgreSQL, il est essentiel de comprendre un concept fondamental : le **WAL Shipping**. Ce système constitue la pierre angulaire de la réplication, des sauvegardes avancées et de la récupération après sinistre dans PostgreSQL.

Dans ce chapitre, nous allons explorer en détail ce qu'est le WAL, pourquoi il existe, et comment l'archivage WAL permet de créer des architectures robustes et résilientes.

---

## 1. Qu'est-ce que le WAL ?

### 1.1. Définition et philosophie

**WAL** signifie **Write-Ahead Log** (journal de pré-écriture en français). C'est un mécanisme fondamental dans PostgreSQL qui garantit la durabilité et la cohérence des données.

**Le principe de base :**
> Avant de modifier les données dans les fichiers de données réels, PostgreSQL écrit d'abord ces modifications dans un journal spécial appelé WAL.

### 1.2. Analogie avec un journal de bord

Imaginons que vous teniez un journal personnel :

- **Votre journal (WAL)** : Vous y écrivez tous les événements de votre journée au fur et à mesure qu'ils se produisent  
- **Votre mémoire (fichiers de données)** : Parfois, vous oubliez des détails, mais vous pouvez toujours relire votre journal pour vous souvenir

De la même manière, PostgreSQL :
1. **Écrit d'abord dans le WAL** toutes les modifications (INSERT, UPDATE, DELETE)  
2. **Applique ensuite ces modifications** aux fichiers de données réels (de manière asynchrone)

### 1.3. Pourquoi utiliser un WAL ?

Le WAL répond à plusieurs besoins critiques :

#### a) **Garantir la durabilité (le "D" d'ACID)**

Si PostgreSQL crash au milieu d'une transaction, le WAL permet de :
- Rejouer les modifications validées (COMMIT) qui n'ont pas encore été écrites sur disque
- Annuler les modifications non validées (ROLLBACK)

#### b) **Optimiser les performances**

Écrire dans le WAL est beaucoup plus rapide que modifier directement les fichiers de données :
- Le WAL est écrit **séquentiellement** (comme écrire dans un cahier ligne par ligne)
- Les fichiers de données sont écrits **aléatoirement** (comme modifier des pages dispersées dans un classeur)

Les écritures séquentielles sont 10 à 100 fois plus rapides que les écritures aléatoires sur disque !

#### c) **Permettre la réplication et la haute disponibilité**

Le WAL peut être copié vers d'autres serveurs PostgreSQL qui peuvent alors "rejouer" ces modifications et rester synchronisés avec le serveur principal.

---

## 2. Anatomie du WAL

### 2.1. Structure physique

Les fichiers WAL se trouvent dans le répertoire `pg_wal/` de votre cluster PostgreSQL.

**Caractéristiques :**
- Chaque fichier WAL fait **16 Mo par défaut** (configurable)
- Les fichiers sont nommés selon une convention spécifique : `000000010000000000000001`
- PostgreSQL crée de nouveaux fichiers WAL au fur et à mesure des besoins
- Les anciens fichiers sont recyclés ou supprimés (sauf en mode archive)

### 2.2. Contenu du WAL

Le WAL contient des **enregistrements** (records) qui décrivent :
- Les modifications de données (tuples insérés, modifiés, supprimés)
- Les opérations sur les index
- Les changements de structure (DDL)
- Les informations de transaction (BEGIN, COMMIT, ROLLBACK)

**Format simplifié d'un enregistrement WAL :**
```
[Type d'opération] [Table concernée] [Données avant/après]
```

Exemple conceptuel :
```
INSERT INTO users VALUES (123, 'Alice', 'alice@example.com')  
UPDATE users SET email='alice.new@example.com' WHERE id=123  
DELETE FROM users WHERE id=123  
```

### 2.3. Le processus de checkpoint

PostgreSQL ne peut pas garder indéfiniment le WAL en mémoire. Périodiquement, il effectue un **checkpoint** :

1. **Flush** : Toutes les modifications en mémoire sont écrites sur disque  
2. **Nettoyage** : Les fichiers WAL antérieurs au checkpoint peuvent être recyclés  
3. **Marqueur** : Un enregistrement spécial indique le point de checkpoint dans le WAL

**Paramètres importants :**
- `checkpoint_timeout` : Intervalle maximum entre deux checkpoints (par défaut 5 minutes)  
- `max_wal_size` : Taille maximum de WAL avant de déclencher un checkpoint (par défaut 1 Go)

---

## 3. Le concept de WAL Shipping

### 3.1. Qu'est-ce que le WAL Shipping ?

**WAL Shipping** (expédition de WAL) est le processus de **copie des fichiers WAL** d'un serveur PostgreSQL (le serveur primaire) vers un autre emplacement :
- Un serveur de secours (standby)
- Un système de stockage pour archivage
- Un système de sauvegarde

**Métaphore postale :**
> Imaginez que le serveur primaire soit un bureau central qui envoie régulièrement des copies de son journal (WAL) à d'autres bureaux. Ces bureaux peuvent alors reconstituer exactement le même état que le bureau central en "rejouant" les journaux reçus.

### 3.2. Les deux modes de WAL Shipping

#### Mode 1 : **Archivage continu (Continuous Archiving)**

PostgreSQL copie automatiquement les fichiers WAL complétés vers un emplacement d'archivage.

**Cas d'usage :**
- Sauvegardes Point-In-Time Recovery (PITR)
- Récupération après sinistre
- Conformité réglementaire (conservation des journaux)

#### Mode 2 : **Réplication en streaming (Streaming Replication)**

PostgreSQL envoie les enregistrements WAL en temps réel vers des serveurs de secours, **avant même** que le fichier WAL soit complet.

**Cas d'usage :**
- Haute disponibilité
- Load balancing en lecture
- Basculement automatique (failover)

💡 **Note importante :** Ce chapitre se concentre sur l'archivage continu. La réplication en streaming sera abordée dans les chapitres suivants (17.2 et 17.3).

### 3.3. Pourquoi archiver le WAL ?

L'archivage du WAL offre des capacités de récupération exceptionnelles :

**Sans archivage WAL :**
- Vous ne pouvez restaurer qu'au moment exact de votre dernière sauvegarde complète
- Toutes les transactions entre la sauvegarde et l'incident sont perdues

**Avec archivage WAL :**
- Vous pouvez restaurer à **n'importe quel point dans le temps** entre votre sauvegarde de base et le moment présent
- Perte de données minimale (quelques secondes au maximum)

**Exemple concret :**
```
08h00 : Sauvegarde complète de la base
10h30 : Un développeur supprime accidentellement 10 000 lignes
10h35 : Vous détectez le problème

Avec PITR :
→ Vous restaurez la base à 10h29, juste avant l'erreur
→ Perte de données : 0
```

---

## 4. Configuration de l'archive_mode

### 4.1. Activation de l'archivage

L'archivage WAL se configure principalement via trois paramètres dans `postgresql.conf` :

#### Paramètre 1 : `wal_level`

Détermine la quantité d'informations écrites dans le WAL.

**Valeurs possibles :**
- `minimal` : Minimum d'informations (pas de réplication possible)  
- `replica` : Informations suffisantes pour la réplication physique  
- `logical` : Informations pour la réplication logique (le plus verbeux)

**Configuration recommandée pour l'archivage :**
```ini
wal_level = replica
```

💡 Depuis PostgreSQL 9.6, `replica` est la valeur par défaut.

#### Paramètre 2 : `archive_mode`

Active ou désactive l'archivage.

**Valeurs possibles :**
- `off` : Archivage désactivé (par défaut)  
- `on` : Archivage activé  
- `always` : Archivage même en standby (PostgreSQL 9.5+)

**Configuration :**
```ini
archive_mode = on
```

⚠️ **Attention :** Modifier `wal_level` ou `archive_mode` nécessite un **redémarrage** de PostgreSQL.

#### Paramètre 3 : `archive_command`

La commande shell qui sera exécutée pour archiver chaque fichier WAL complété.

**Signature de la commande :**
```bash
archive_command = 'command %p %f'
```

**Variables disponibles :**
- `%p` : Chemin complet du fichier WAL à archiver  
- `%f` : Nom du fichier WAL uniquement

### 4.2. Exemples de commandes d'archivage

#### Exemple 1 : Copie locale simple

```ini
archive_command = 'cp %p /mnt/archive/pg_wal/%f'
```

**Avantages :**
- Simple à configurer
- Rapide

**Inconvénients :**
- Pas de protection contre la panne du serveur (même machine)
- Pas de vérification d'intégrité

#### Exemple 2 : Copie avec vérification (recommandé)

```ini
archive_command = 'test ! -f /mnt/archive/pg_wal/%f && cp %p /mnt/archive/pg_wal/%f'
```

Cette commande vérifie que le fichier n'existe pas déjà avant de le copier, évitant ainsi d'écraser un fichier existant.

#### Exemple 3 : Archivage distant avec rsync

```ini
archive_command = 'rsync -a %p backup-server:/archive/pg_wal/%f'
```

**Avantages :**
- Protection contre la panne du serveur (machine distante)
- Transfert optimisé

#### Exemple 4 : Archivage vers le Cloud (AWS S3)

```ini
archive_command = 'aws s3 cp %p s3://my-bucket/pg-archives/%f'
```

**Avantages :**
- Durabilité extrême (99.999999999% avec S3)
- Archivage géo-distribué automatique
- Pas de gestion d'infrastructure de stockage

#### Exemple 5 : Archivage avec pgBackRest

```ini
archive_command = 'pgbackrest --stanza=my-db archive-push %p'
```

**Avantages :**
- Compression automatique
- Vérification d'intégrité
- Gestion avancée des archives

### 4.3. Codes de retour et gestion des erreurs

**Important :** PostgreSQL considère l'archivage réussi **uniquement si** la commande retourne un code de sortie 0.

**Comportement en cas d'échec :**
- PostgreSQL **réessaiera** périodiquement d'archiver le fichier
- Les fichiers WAL **ne seront pas supprimés** tant qu'ils ne sont pas archivés
- Si l'archivage échoue trop longtemps, `pg_wal/` peut se remplir et **bloquer la base**

**Bonnes pratiques :**
1. Toujours tester votre `archive_command` manuellement avant de l'activer  
2. Surveiller l'espace disque de `pg_wal/`  
3. Mettre en place des alertes sur les échecs d'archivage  
4. Prévoir un mécanisme de nettoyage d'urgence

### 4.4. Configuration complète d'exemple

Voici un exemple de configuration complète pour l'archivage :

```ini
# postgresql.conf

# Niveau de verbosité du WAL
wal_level = replica

# Activation de l'archivage
archive_mode = on

# Commande d'archivage (copie vers un NAS)
archive_command = 'rsync -a --timeout=60 %p /mnt/backup/pg_wal/%f'

# Taille maximale de WAL avant checkpoint (optionnel)
max_wal_size = 2GB

# Intervalle maximum entre deux checkpoints (optionnel)
checkpoint_timeout = 10min

# Compression du WAL (PostgreSQL 14+) - optionnel
wal_compression = on
```

---

## 5. Le cycle de vie d'un fichier WAL en mode archive

Comprendre le cycle de vie d'un fichier WAL est essentiel pour bien appréhender le système.

### 5.1. Étape 1 : Création et écriture

```
Transaction → Modifications → Écriture dans le WAL en mémoire (buffers)
```

Les modifications sont d'abord écrites dans les buffers WAL en mémoire.

### 5.2. Étape 2 : Flush sur disque

```
WAL buffers → pg_wal/000000010000000000000001 (en cours d'écriture)
```

Lors d'un COMMIT ou lorsque les buffers sont pleins, les données sont écrites physiquement sur disque.

### 5.3. Étape 3 : Segment complété

```
Fichier WAL rempli (16 Mo atteint) → Passage au segment suivant
```

PostgreSQL ferme le fichier WAL actuel et commence à écrire dans un nouveau.

### 5.4. Étape 4 : Archivage

```
archive_command est exécuté
→ Copie vers l'emplacement d'archivage
→ Code de retour 0 (succès)
```

Le fichier WAL complété est copié vers l'emplacement d'archivage.

### 5.5. Étape 5 : Nettoyage ou recyclage

Une fois archivé avec succès :
- Le fichier peut être **recyclé** (renommé et réutilisé pour un nouveau segment)
- Ou **supprimé** si PostgreSQL a suffisamment de segments disponibles

---

## 6. Surveillance et maintenance

### 6.1. Vérifier l'état de l'archivage

#### Requête SQL pour vérifier le statut

```sql
SELECT
    archived_count,        -- Nombre de WAL archivés avec succès
    last_archived_wal,     -- Dernier WAL archivé
    last_archived_time,    -- Horodatage du dernier archivage
    failed_count,          -- Nombre d'échecs d'archivage
    last_failed_wal,       -- Dernier WAL en échec
    last_failed_time       -- Horodatage du dernier échec
FROM pg_stat_archiver;
```

**Interprétation :**
- `failed_count` devrait être 0 en production  
- `last_archived_time` devrait être récent (quelques minutes)

#### Vérifier l'espace disque de pg_wal/

```bash
# Taille totale du répertoire WAL
du -sh /var/lib/postgresql/14/main/pg_wal/

# Nombre de fichiers WAL
ls -l /var/lib/postgresql/14/main/pg_wal/ | grep -v "archive_status" | wc -l
```

**Alerte si :**
- La taille dépasse 2-3× `max_wal_size`
- Le nombre de fichiers augmente continuellement

### 6.2. Diagnostiquer les problèmes d'archivage

#### Symptôme 1 : pg_wal/ se remplit

**Causes possibles :**
1. La commande d'archivage échoue  
2. Le stockage de destination est plein  
3. Problème réseau (pour archivage distant)  
4. Permissions insuffisantes

**Diagnostic :**
```bash
# Vérifier les logs PostgreSQL
tail -f /var/log/postgresql/postgresql-14-main.log | grep archive

# Tester manuellement la commande d'archivage
su - postgres
/path/to/archive_command /path/to/wal/file testfile
echo $?  # Doit retourner 0
```

#### Symptôme 2 : Archivage lent

**Causes possibles :**
1. Bande passante réseau insuffisante  
2. Stockage de destination lent (disque, réseau)  
3. Compression CPU-intensive

**Solution :**
- Utiliser un stockage local temporaire avec copie asynchrone
- Activer `wal_compression` pour réduire la taille
- Paralléliser l'archivage avec des outils comme pgBackRest

### 6.3. Métriques à surveiller

Pour un système de production robuste, surveillez :

| Métrique | Seuil d'alerte | Action |
|----------|---------------|---------|
| `pg_stat_archiver.failed_count` | > 0 | Investiguer immédiatement |
| Taille de `pg_wal/` | > 3× `max_wal_size` | Vérifier l'archivage |
| `last_archived_time` | > 5 minutes | Vérifier la connectivité |
| Espace disque archive | < 20% libre | Nettoyer les anciennes archives |
| Génération WAL/seconde | Tendance à la hausse | Optimiser les écritures |

---

## 7. Cas d'usage et architectures

### 7.1. Architecture de base : Sauvegarde PITR

```
┌──────────────────┐
│  PostgreSQL      │
│   (Primary)      │
│                  │
│  ┌────────────┐  │
│  │  pg_wal/   │  │
│  └─────┬──────┘  │
└────────┼─────────┘
         │ archive_command
         ▼
┌─────────────────┐
│  Archive        │
│  Storage        │
│  (NFS/S3/etc)   │
└─────────────────┘
```

**Bénéfices :**
- Récupération point-in-time
- Protection contre les erreurs humaines
- Conformité réglementaire

### 7.2. Architecture avancée : Multi-destinations

```
┌──────────────────┐
│  PostgreSQL      │
└────────┬─────────┘
         │
         ├─────────────┐
         │             │
         ▼             ▼
┌─────────────┐  ┌──────────────┐
│  Local      │  │  Remote      │
│  Archive    │  │  Archive     │
│  (Fast)     │  │  (Secure)    │
└─────────────┘  └──────────────┘
```

**Implémentation avec un script personnalisé :**
```bash
#!/bin/bash
# archive_multi.sh

WAL_PATH=$1  
WAL_FILE=$2  

# Copie locale rapide
cp "$WAL_PATH" /mnt/local/archive/"$WAL_FILE" || exit 1

# Copie distante asynchrone (en arrière-plan)
rsync -a "$WAL_PATH" backup-server:/archive/"$WAL_FILE" &

exit 0
```

### 7.3. Architecture haute disponibilité

L'archivage WAL est la base de la réplication :

```
┌──────────────────┐
│  Primary         │
└────────┬─────────┘
         │ WAL Shipping
         ├─────────────┐
         ▼             ▼
┌─────────────┐  ┌──────────────┐
│  Standby 1  │  │  Standby 2   │
│  (Hot)      │  │  (Warm)      │
└─────────────┘  └──────────────┘
```

Ce sujet sera approfondi dans les chapitres 17.2 et 17.3.

---

## 8. Bonnes pratiques et recommandations

### 8.1. Sécurité de l'archivage

1. **Chiffrez les archives** si elles contiennent des données sensibles :
   ```ini
   archive_command = 'openssl enc -aes-256-cbc -salt -in %p -out /archive/%f.enc -pass file:/path/to/passphrase'
   ```

2. **Limitez les permissions** sur le répertoire d'archivage :
   ```bash
   chmod 700 /mnt/archive/pg_wal/
   chown postgres:postgres /mnt/archive/pg_wal/
   ```

3. **Utilisez des connexions sécurisées** (SSH, TLS) pour l'archivage distant

### 8.2. Performances et optimisation

1. **Activez la compression WAL** (PostgreSQL 14+) :
   ```ini
   wal_compression = on
   ```
   Réduction typique : 40-60% de la taille du WAL

2. **Optimisez `max_wal_size`** selon votre charge :
   - OLTP léger : 1-2 GB
   - OLTP intense : 4-8 GB
   - ETL/Batch : 16-32 GB

3. **Utilisez un stockage local rapide** pour `pg_wal/` :
   - SSD ou NVMe recommandé
   - Évitez les montages réseau (NFS) pour `pg_wal/`

### 8.3. Rétention et nettoyage des archives

Les archives WAL peuvent rapidement consommer de l'espace disque. Planifiez une stratégie de rétention :

**Exemple de stratégie :**
- Conserver 7 jours d'archives complètes (pour PITR quotidien)
- Archiver mensuellement vers un stockage froid (Glacier, Azure Archive)
- Supprimer les archives > 30 jours

**Script de nettoyage (cron) :**
```bash
#!/bin/bash
# cleanup_old_wal.sh

ARCHIVE_DIR="/mnt/archive/pg_wal"  
RETENTION_DAYS=7  

# Supprimer les fichiers WAL plus vieux que RETENTION_DAYS
find "$ARCHIVE_DIR" -name "0*" -type f -mtime +$RETENTION_DAYS -delete

# Logger l'opération
echo "$(date): Cleaned WAL archives older than $RETENTION_DAYS days" >> /var/log/pg_cleanup.log
```

**⚠️ Important :** Ne supprimez JAMAIS des archives WAL nécessaires pour les standbys actifs !

### 8.4. Tests réguliers

L'archivage WAL n'a de valeur que si vous pouvez restaurer :

1. **Testez votre procédure de restauration** au moins trimestriellement  
2. **Documentez le processus** étape par étape  
3. **Mesurez les temps** de restauration (RTO - Recovery Time Objective)  
4. **Vérifiez l'intégrité** des archives régulièrement

### 8.5. Monitoring et alerting

Configurez des alertes pour :
- Échec d'archivage (failed_count > 0)
- Retard d'archivage (last_archived_time > 5 min)
- Espace disque insuffisant (pg_wal/ ou archive)
- Génération anormale de WAL (pic soudain)

**Exemple avec Prometheus/Alertmanager :**
```yaml
- alert: PostgreSQLArchivingFailing
  expr: pg_stat_archiver_failed_count > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "WAL archiving is failing on {{ $labels.instance }}"
```

---

## 9. Dépannage : Scénarios courants

### Scénario 1 : "Le répertoire pg_wal/ est plein"

**Symptômes :**
```
PANIC: could not write to file "pg_wal/xlogtemp.12345": No space left on device
```

**Cause :** L'archivage échoue, les fichiers WAL s'accumulent.

**Solution d'urgence :**
```bash
# 1. Vérifier l'espace disque
df -h /var/lib/postgresql/14/main/pg_wal/

# 2. Identifier le problème d'archivage
tail -100 /var/log/postgresql/postgresql-14-main.log | grep -i archive

# 3. Si nécessaire, archiver manuellement et temporairement
su - postgres  
cd /var/lib/postgresql/14/main/pg_wal/  
for f in 0*; do  
  if [ -f "$f" ]; then
    cp "$f" /mnt/backup/emergency/ && rm "$f"
  fi
done

# 4. Corriger la cause racine (commande d'archivage, espace disque, etc.)
```

### Scénario 2 : "Comment restaurer à un point précis ?"

Ce sera détaillé dans le chapitre sur PITR, mais voici le principe :

1. Restaurer une sauvegarde de base (pg_basebackup)  
2. Placer les archives WAL dans le répertoire approprié  
3. Créer un fichier `recovery.signal`  
4. Configurer `restore_command` dans `postgresql.auto.conf`  
5. Démarrer PostgreSQL qui rejouera les WAL jusqu'au point souhaité

### Scénario 3 : "L'archivage est lent et bloque la production"

**Diagnostic :**
```sql
SELECT * FROM pg_stat_archiver;
```

Si `last_archived_time` est ancien malgré une activité élevée :

**Solutions :**
1. Utiliser un archivage local + copie asynchrone  
2. Augmenter `max_wal_size` pour espacer les checkpoints  
3. Optimiser le réseau ou le stockage de destination  
4. Utiliser pgBackRest ou WAL-G (archivage parallèle)

---

## 10. Évolutions et nouveautés (PostgreSQL 18)

PostgreSQL 18 apporte des améliorations notables dans la gestion du WAL :

### 10.1. Sous-système I/O asynchrone (AIO)

PostgreSQL 18 introduit un nouveau système d'I/O asynchrone qui améliore significativement les performances d'écriture WAL.

**Configuration :**
```ini
io_method = 'worker'  # Défaut PG 18 (ou 'io_uring' sur Linux 5.1+)
```

**Gains de performance :**
- Jusqu'à **3× plus rapide** pour les charges d'écriture intensives
- Réduction de la latence des COMMIT

### 10.2. Data Checksums activés par défaut

Les checksums de données détectent la corruption :
```bash
# Avant PG 18 : opt-in
initdb --data-checksums

# PG 18 : activé par défaut
initdb
# (utiliser --no-data-checksums pour désactiver)
```

**Impact sur l'archivage :**
- Détection automatique de la corruption pendant le replay WAL
- Meilleure fiabilité des standbys

### 10.3. Améliorations de COPY

Les améliorations de COPY en PostgreSQL 18 génèrent moins de WAL pour les insertions massives, réduisant la pression sur l'archivage.

---

## 11. Résumé et points clés

### Ce qu'il faut retenir 🔑

1. **Le WAL est le journal de modifications** de PostgreSQL, garantissant durabilité et cohérence

2. **WAL Shipping = copie des fichiers WAL** vers un autre emplacement pour :
   - Sauvegardes Point-In-Time Recovery (PITR)
   - Réplication et haute disponibilité
   - Récupération après sinistre

3. **Configuration minimale de l'archivage :**
   ```ini
   wal_level = replica
   archive_mode = on
   archive_command = 'command %p %f'
   ```

4. **La commande d'archivage doit retourner 0** pour indiquer le succès, sinon PostgreSQL réessaiera

5. **Surveillez pg_stat_archiver** pour détecter les problèmes d'archivage avant qu'ils ne deviennent critiques

6. **Testez régulièrement vos restaurations** - un backup non testé est un backup qui n'existe pas

7. **Planifiez la rétention et le nettoyage** des archives pour éviter la saturation du stockage

### Prochaines étapes

Maintenant que vous comprenez l'archivage WAL, vous êtes prêt à explorer :
- **17.2** : Réplication Physique (Streaming Replication)  
- **17.3** : Réplication Logique  
- **Chapitre 16.11** : PITR (Point-In-Time Recovery) en pratique

---

## Ressources complémentaires

### Documentation officielle
- [PostgreSQL: Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal.html)  
- [PostgreSQL: Continuous Archiving and Point-in-Time Recovery (PITR)](https://www.postgresql.org/docs/current/continuous-archiving.html)

### Outils recommandés
- **pgBackRest** : Solution de backup/restore avancée avec archivage WAL optimisé  
- **WAL-G** : Outil d'archivage WAL avec support cloud natif (S3, Azure, GCS)  
- **Barman** : Backup and Recovery Manager pour PostgreSQL

### Lectures avancées
- "PostgreSQL: Up and Running" (O'Reilly) - Chapitre sur la réplication
- Blog 2ndQuadrant : Articles sur WAL internals
- Percona Blog : Cas d'usage d'archivage WAL en production

---


⏭️ [Réplication Physique (Streaming Replication)](/17-haute-disponibilite-et-replication/02-replication-physique.md)
