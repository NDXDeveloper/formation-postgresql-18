🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.2. Réplication Physique (Streaming Replication)

## Introduction

La **réplication physique** (Physical Replication), également appelée **Streaming Replication**, est le mécanisme principal de réplication de données dans PostgreSQL. C'est une technologie fondamentale pour assurer la **haute disponibilité** (High Availability), la **répartition de charge** (Load Balancing) et la **reprise après sinistre** (Disaster Recovery) de vos bases de données PostgreSQL.

Contrairement aux solutions de réplication basées sur des déclencheurs (triggers) ou des règles applicatives, la réplication physique de PostgreSQL opère au niveau le plus bas de la base de données : elle réplique les **fichiers de données bruts** et les **journaux de transactions** (WAL - Write-Ahead Logs). Cette approche garantit une réplication fidèle, performante et transparente pour les applications.

---

## Qu'est-ce que la Réplication Physique ?

### Définition

La réplication physique est un processus par lequel PostgreSQL maintient **une ou plusieurs copies identiques** d'une base de données en transmettant et en appliquant les modifications au niveau des blocs de données physiques.

**En termes simples :**
- Toutes les modifications effectuées sur le serveur **primary** (principal) sont enregistrées dans des fichiers journaux appelés **WAL** (Write-Ahead Logs)
- Ces fichiers WAL sont ensuite **envoyés en continu** (streaming) vers un ou plusieurs serveurs **standby** (secondaires)
- Les serveurs standby **rejouent** ces WAL pour maintenir leur copie de la base de données à jour
- Le résultat est une **copie bit-à-bit identique** du serveur primary

### Analogie du monde réel

Imaginez un professeur (primary) qui donne un cours magistral. Plutôt que de répéter le cours plusieurs fois, il enregistre audio/vidéo sa présentation. Les étudiants (standbys) reçoivent cet enregistrement en temps réel et peuvent le regarder simultanément. L'enregistrement (WAL) capture **exactement** ce que le professeur dit, sans interprétation ni modification.

---

## Réplication Physique vs Réplication Logique

PostgreSQL propose deux types de réplication : **physique** et **logique**. Il est important de comprendre leurs différences fondamentales.

### Réplication Physique (Physical Replication)

**Niveau de réplication :** Blocs de données bruts (niveau fichier)

**Comment ça fonctionne :**
- Réplication des WAL (Write-Ahead Logs) qui contiennent les modifications au niveau des blocs disque
- Le standby rejoue ces modifications exactement comme le primary les a effectuées
- Réplication **bit-à-bit** : le standby est une copie identique du primary

**Caractéristiques :**
- ✅ **Réplication complète** : Toute la base de données (toutes les bases, tous les schémas, toutes les tables)
- ✅ **Performances élevées** : Overhead minimal, très rapide
- ✅ **Simplicité** : Configuration relativement simple
- ✅ **Fidélité absolue** : Garantie d'identité parfaite entre primary et standby
- ⚠️ **Tout ou rien** : Impossible de répliquer uniquement certaines tables ou bases
- ⚠️ **Version identique requise** : Primary et standby doivent être sur la même version majeure de PostgreSQL
- ⚠️ **Architecture identique** : Même OS, même architecture CPU (x86_64, ARM, etc.)

**Cas d'usage typiques :**
- Haute disponibilité (HA)
- Répartition de charge en lecture (read replicas)
- Disaster Recovery (DR)
- Backup à chaud (hot backup)

### Réplication Logique (Logical Replication)

**Niveau de réplication :** Données logiques (niveau ligne/table)

**Comment ça fonctionne :**
- Réplication des **modifications logiques** (INSERT, UPDATE, DELETE) au niveau des lignes
- Les modifications sont décodées des WAL et converties en opérations SQL
- Le standby applique ces opérations comme des requêtes SQL normales

**Caractéristiques :**
- ✅ **Réplication sélective** : Possibilité de répliquer uniquement certaines tables ou bases
- ✅ **Cross-version** : Peut répliquer entre différentes versions majeures de PostgreSQL
- ✅ **Transformations** : Possibilité de transformer les données pendant la réplication
- ✅ **Bi-directionnelle** : Possibilité de réplication dans les deux sens (avec précautions)
- ⚠️ **Performances moindres** : Overhead plus important que la réplication physique
- ⚠️ **Complexité** : Configuration plus complexe
- ⚠️ **Limitations** : Ne réplique pas les DDL (CREATE, ALTER, DROP), séquences, large objects, etc.

**Cas d'usage typiques :**
- Migrations de version majeure PostgreSQL
- Réplication sélective (sous-ensemble de données)
- Consolidation de plusieurs bases en une seule
- Réplication vers des systèmes hétérogènes

### Tableau Comparatif

| Critère | Réplication Physique | Réplication Logique |
|---------|----------------------|---------------------|
| **Niveau** | Blocs disque (physique) | Lignes/tables (logique) |
| **Granularité** | Toute l'instance | Tables spécifiques |
| **Performance** | ⚡ Très élevée | 🔶 Moyenne |
| **Complexité** | ✅ Simple | ⚠️ Moyenne à élevée |
| **Versions PostgreSQL** | Identiques (même majeure) | Peuvent différer |
| **Architecture** | Identique (OS, CPU) | Peut différer |
| **DDL** | ✅ Répliqué | ❌ Non répliqué |
| **Transformations** | ❌ Impossible | ✅ Possible |
| **Bi-directionnel** | ❌ Non | ✅ Oui (avec précautions) |
| **Cas d'usage principal** | HA, DR, Read Replicas | Migrations, Réplication sélective |

**Recommandation générale :**
- **Utilisez la réplication physique** pour la haute disponibilité, la reprise après sinistre, et les read replicas (95% des cas d'usage)
- **Utilisez la réplication logique** pour les migrations de version majeure, la réplication sélective, ou les architectures distribuées complexes

---

## Concepts Fondamentaux

Avant de plonger dans la configuration, il est essentiel de comprendre les concepts clés de la réplication physique.

### 1. Les Rôles : Primary et Standby

#### Primary (Serveur Principal)

**Définition :** Le serveur primary est le serveur **maître** qui accepte les opérations d'écriture (INSERT, UPDATE, DELETE).

**Caractéristiques :**
- ✅ Accepte les lectures (SELECT) et les écritures (INSERT, UPDATE, DELETE)
- ✅ Génère les WAL (Write-Ahead Logs)
- ✅ Envoie les WAL aux serveurs standby en temps réel
- ✅ Source de vérité : c'est la version "officielle" de la base de données

**Processus impliqués :**
- **WAL Writer** : Écrit les WAL sur disque
- **WAL Sender** : Envoie les WAL aux standbys (un processus par standby connecté)
- **Checkpointer** : Synchronise périodiquement les données en mémoire avec le disque

#### Standby (Serveur Secondaire)

**Définition :** Le serveur standby est un serveur **esclave** qui maintient une copie à jour de la base de données en rejouant les WAL reçus du primary.

**Caractéristiques :**
- ✅ Accepte les lectures (SELECT) en mode **Hot Standby**
- ❌ Refuse les écritures (read-only)
- ✅ Reçoit et rejoue les WAL du primary
- ✅ Peut devenir primary en cas de promotion (failover)

**Processus impliqués :**
- **WAL Receiver** : Reçoit les WAL du primary via la connexion réseau
- **Startup Process** : Rejoue les WAL pour mettre à jour la base de données
- **WAL Sender** (en cascade) : Si configuré, peut envoyer les WAL à d'autres standbys

**Modes de standby :**

**Hot Standby (recommandé)**
- Le standby accepte les connexions en lecture seule
- Les utilisateurs peuvent exécuter des SELECT
- Idéal pour répartir les charges de lecture

**Warm Standby (obsolète)**
- Le standby n'accepte aucune connexion
- Utilisé uniquement pour la haute disponibilité (failover)
- Moins courant aujourd'hui (Hot Standby est préféré)

### 2. WAL (Write-Ahead Logs)

#### Qu'est-ce que les WAL ?

Les **WAL** (Write-Ahead Logs) sont des fichiers journaux qui enregistrent **toutes les modifications** apportées à la base de données **avant** qu'elles ne soient effectivement écrites dans les fichiers de données.

**Principe du Write-Ahead Logging :**
1. Une transaction modifie des données (INSERT, UPDATE, DELETE)
2. PostgreSQL écrit **d'abord** ces modifications dans les WAL (en mémoire, puis sur disque)
3. La transaction est validée (COMMIT)
4. **Plus tard**, lors d'un checkpoint, PostgreSQL écrit les modifications des WAL dans les fichiers de données

**Pourquoi cette approche ?**
- ✅ **Atomicité** : Garantit qu'une transaction est soit complètement validée, soit complètement annulée
- ✅ **Durabilité** : Les données validées ne sont jamais perdues (même en cas de crash)
- ✅ **Performance** : Écriture séquentielle des WAL (très rapide) vs écriture aléatoire dans les fichiers de données (plus lent)
- ✅ **Récupération** : Permet de récupérer la base de données après un crash en rejouant les WAL

#### Structure des WAL

**Localisation :**
- Répertoire : `pg_wal/` (anciennement `pg_xlog/` avant PostgreSQL 10)
- Exemple : `/var/lib/postgresql/18/main/pg_wal/`

**Format :**
- Fichiers binaires de **16 MB** par défaut (configurable)
- Nommage : `000000010000000000000001`, `000000010000000000000002`, etc.
- Écriture séquentielle : quand un fichier est plein, PostgreSQL passe au suivant

**Contenu :**
- Modifications au niveau des blocs de données (pages)
- Informations de transaction (BEGIN, COMMIT, ABORT)
- Métadonnées (checkpoints, snapshots)
- Opérations DDL (CREATE TABLE, ALTER, etc.)

**Cycle de vie :**
1. **Création** : Un segment WAL est créé quand nécessaire
2. **Écriture** : Les transactions écrivent leurs modifications
3. **Archivage** (optionnel) : Le segment est copié vers un emplacement de sauvegarde
4. **Réplication** : Le segment est envoyé aux standbys
5. **Recyclage** : Après un checkpoint, le segment peut être recyclé (renommé et réutilisé)

#### WAL et Réplication

Dans le contexte de la réplication :
- Le primary génère les WAL normalement (pour la durabilité)
- **En plus**, il envoie ces WAL en continu (streaming) aux standbys
- Les standbys reçoivent et rejouent ces WAL pour rester à jour
- C'est cette transmission continue de WAL qui constitue la **Streaming Replication**

### 3. Streaming Replication

#### Définition

Le **Streaming Replication** est le mécanisme par lequel les WAL sont transmis **en continu** du primary au standby via une connexion réseau TCP/IP.

**Pourquoi "Streaming" ?**
- Les WAL sont envoyés **au fur et à mesure** de leur génération (en temps réel)
- Pas d'attente qu'un segment WAL complet (16 MB) soit écrit
- Latence minimale : quelques millisecondes entre primary et standby

#### Architecture de Streaming

```
Primary                                    Standby
┌─────────────────────┐                  ┌─────────────────────┐
│                     │                  │                     │
│  Transaction        │                  │                     │
│       ↓             │                  │                     │
│  ┌─────────────┐    │                  │                     │
│  │ WAL Buffer  │    │                  │                     │
│  └──────┬──────┘    │                  │                     │
│         ↓           │                  │                     │
│  ┌─────────────┐    │                  │                     │
│  │ WAL Writer  │    │                  │                     │
│  └──────┬──────┘    │                  │                     │
│         ↓           │                  │                     │
│  ┌─────────────┐    │    Streaming     │  ┌─────────────┐    │
│  │   pg_wal/   │◄───┼──────────────────┼──┤WAL Receiver │    │
│  └──────┬──────┘    │    (TCP/IP)      │  └──────┬──────┘    │
│         ↓           │                  │         ↓           │
│  ┌─────────────┐    │                  │  ┌─────────────┐    │
│  │ WAL Sender  ├────┼──────────────────┼─>│   pg_wal/   │    │
│  └─────────────┘    │                  │  └──────┬──────┘    │
│                     │                  │         ↓           │
└─────────────────────┘                  │  ┌─────────────┐    │
                                         │  │   Startup   │    │
                                         │  │   Process   │    │
                                         │  └──────┬──────┘    │
                                         │         ↓           │
                                         │  ┌─────────────┐    │
                                         │  │  Database   │    │
                                         │  └─────────────┘    │
                                         │                     │
                                         └─────────────────────┘
```

**Étapes du processus :**

1. **Sur le Primary :**
   - Une transaction modifie des données
   - Les modifications sont écrites dans le WAL buffer (mémoire)
   - Le WAL writer écrit les WAL sur disque (`pg_wal/`)
   - Le WAL sender lit ces WAL et les envoie au standby via TCP/IP

2. **Transmission réseau :**
   - Connexion TCP/IP persistante entre primary et standby
   - Port par défaut : 5432 (le même que PostgreSQL)
   - Peut être chiffré avec SSL/TLS

3. **Sur le Standby :**
   - Le WAL receiver reçoit les WAL via la connexion réseau
   - Il écrit ces WAL dans le répertoire `pg_wal/` du standby
   - Le startup process lit ces WAL et les rejoue (replay)
   - Les modifications sont appliquées à la base de données du standby

#### Avantages du Streaming

- ⚡ **Latence minimale** : Les modifications sont transmises en millisecondes
- 🔄 **Temps réel** : Le standby est quasiment synchronisé avec le primary
- 🔌 **Simple** : Utilise une connexion réseau standard (TCP/IP)
- 📊 **Monitoring** : Vues système pour surveiller l'état de réplication
- 🛡️ **Résilient** : Gère automatiquement les reconnexions en cas de perte réseau

### 4. Hot Standby

#### Définition

Le **Hot Standby** est une fonctionnalité qui permet aux serveurs standby d'accepter des **connexions en lecture seule** pendant qu'ils rejouent les WAL.

**Sans Hot Standby (Warm Standby) :**
```
Standby : Rejouant les WAL... [Aucune connexion acceptée]
```

**Avec Hot Standby :**
```
Standby : Rejouant les WAL... [Accepte SELECT, pas INSERT/UPDATE/DELETE]
```

#### Avantages

**1. Répartition de charge (Load Balancing)**
- Les requêtes de lecture (SELECT) peuvent être dirigées vers les standbys
- Le primary est libéré pour traiter les écritures
- Scalabilité horizontale en lecture : ajoutez plus de standbys pour plus de capacité

**2. Reporting et Analytics**
- Exécuter des requêtes lourdes d'analytics sur un standby dédié
- Aucun impact sur le primary et les utilisateurs en production
- Idéal pour les tableaux de bord, rapports, BI

**3. Utilisation optimale des ressources**
- Les standbys ne servent plus uniquement de "secours passifs"
- Ils participent activement à la charge de travail

#### Limitations du Hot Standby

⚠️ **Lecture seule stricte**
- Aucune écriture (INSERT, UPDATE, DELETE) n'est autorisée
- Tentative d'écriture → Erreur : `cannot execute INSERT in a read-only transaction`

⚠️ **Conflits de réplication**
- Si une requête sur le standby est en conflit avec un WAL entrant (ex: lecture d'une ligne que le primary veut supprimer), PostgreSQL peut :
  - Annuler la requête sur le standby (comportement par défaut)
  - Attendre un délai configurable (`max_standby_streaming_delay`)
  - Continuer le replay après le délai (sacrifie la requête)

⚠️ **Lag de réplication**
- Les données sur le standby peuvent être légèrement en retard (quelques millisecondes à quelques secondes)
- Les lectures peuvent retourner des données "obsolètes" si le standby n'a pas encore appliqué les derniers WAL

### 5. Replication Slots

#### Définition

Un **replication slot** (emplacement de réplication) est un mécanisme qui garantit que le primary **conserve** les WAL nécessaires pour un standby, même si ce standby est temporairement déconnecté.

**Sans slot de réplication :**
```
Primary génère WAL → WAL archivés → WAL recyclés après checkpoint
Si standby déconnecté > durée de rétention → WAL perdus → Standby ne peut plus se reconnecter
```

**Avec slot de réplication :**
```
Primary génère WAL → Vérifie les slots → Conserve les WAL tant que le standby ne les a pas reçus
```

#### Avantages

✅ **Protection contre la perte de WAL**
- Même si le standby est en panne pendant des heures/jours, les WAL nécessaires sont conservés
- Le standby peut se reconnecter et rattraper son retard

✅ **Facilite les opérations de maintenance**
- Possibilité de redémarrer un standby pour maintenance sans risque
- Pas besoin de recalculer précisément `wal_keep_size`

✅ **Monitoring intégré**
- Vues système pour surveiller l'état des slots
- Identification facile des standbys en retard

#### Inconvénients

⚠️ **Accumulation de WAL**
- Si un standby ne se reconnecte jamais, les WAL s'accumulent indéfiniment
- Peut remplir le disque du primary si non surveillé

⚠️ **Gestion manuelle**
- Les slots doivent être supprimés manuellement si un standby est définitivement retiré
- Nécessite une surveillance active

#### Types de Slots

**Physical Replication Slot** (pour la réplication physique)
- Conserve les WAL au niveau des blocs physiques
- Utilisé pour la Streaming Replication classique

**Logical Replication Slot** (pour la réplication logique)
- Conserve les changements logiques décodés
- Utilisé pour la réplication logique (hors sujet de ce chapitre)

---

## Modes de Réplication : Synchrone vs Asynchrone

Un choix fondamental lors de la configuration de la réplication est le **mode de synchronisation**.

### Réplication Asynchrone (par défaut)

**Fonctionnement :**
- Le primary valide une transaction dès qu'elle est écrite localement (sur son propre disque)
- Les WAL sont envoyés au standby **en arrière-plan**
- Le primary n'attend **pas** la confirmation du standby

**Avantages :**
- ⚡ Performances maximales (latence minimale)
- 🔌 Résilient : le primary continue si le standby tombe
- 🌍 Adapté aux réplications géographiquement distantes

**Inconvénient :**
- ⚠️ Risque de perte de données : si le primary tombe avant l'envoi des WAL, les transactions non répliquées sont perdues

### Réplication Synchrone

**Fonctionnement :**
- Le primary **attend** la confirmation du standby avant de valider une transaction
- Les WAL doivent être écrits sur le disque du standby avant que le primary réponde "COMMIT" à l'application
- Garantie de **zero data loss**

**Avantages :**
- ✅ Aucune perte de données
- ✅ Cohérence forte entre primary et standby

**Inconvénients :**
- 🐌 Latence augmentée (chaque transaction attend le réseau)
- ⚠️ Dépendance au standby : si le standby tombe, le primary peut bloquer

**Note :** Les modes synchrone et asynchrone sont détaillés dans le chapitre 17.2.2.

---

## Cascading Replication (Réplication en Cascade)

La **réplication en cascade** permet à un standby de servir lui-même de source de réplication pour d'autres standbys.

**Architecture simple :**
```
Primary → Standby 1 → Standby 2 (cascade)
```

**Avantages :**
- Réduit la charge sur le primary (moins de connexions WAL sender)
- Permet de construire des architectures géographiquement distribuées
- Scalabilité : un primary peut alimenter des dizaines de standbys via des intermédiaires

**Note :** La réplication en cascade est détaillée dans le chapitre 17.2.3.

---

## Cas d'Usage de la Réplication Physique

### 1. Haute Disponibilité (High Availability - HA)

**Objectif :** Minimiser le temps d'indisponibilité en cas de panne du serveur primary.

**Architecture typique :**
```
Primary (Datacenter A)
 └─> Standby Hot (Datacenter A ou B) [Synchrone ou Asynchrone]
```

**Processus de failover :**
1. Le primary tombe en panne (panne matérielle, crash, etc.)
2. Le standby est **promu** en nouveau primary (promotion)
3. Les applications sont redirigées vers le nouveau primary
4. RTO (Recovery Time Objective) : Quelques secondes à quelques minutes

**Avantages :**
- ✅ Continuité de service
- ✅ Automatisable (avec Patroni, repmgr, etc.)
- ✅ Aucune perte de données (si réplication synchrone)

### 2. Répartition de Charge en Lecture (Read Replicas)

**Objectif :** Scalabilité horizontale pour les charges de lecture (SELECT).

**Architecture typique :**
```
Primary (Écritures)
 ├─> Standby Read 1 (Lectures)
 ├─> Standby Read 2 (Lectures)
 └─> Standby Read 3 (Lectures)
```

**Répartition de charge :**
- Les applications écrivent sur le primary
- Les applications lisent depuis les standbys (via un load balancer ou DNS round-robin)
- Chaque standby peut traiter des milliers de connexions en lecture

**Cas d'usage typiques :**
- Applications web avec beaucoup plus de lectures que d'écritures
- APIs REST en lecture seule
- Tableaux de bord et reporting

**Avantages :**
- ✅ Scalabilité : Ajouter des standbys pour augmenter la capacité de lecture
- ✅ Isolation : Les requêtes lourdes sur les standbys n'impactent pas le primary
- ✅ Performance : Le primary est dédié aux écritures

### 3. Disaster Recovery (DR)

**Objectif :** Protéger les données contre des catastrophes majeures (incendie, inondation, coupure électrique régionale).

**Architecture typique :**
```
Primary (Paris)
 └─> Standby DR (Londres ou New York) [Asynchrone]
```

**Caractéristiques :**
- Standby dans un datacenter ou une région géographique différente
- Réplication asynchrone (pour gérer les latences réseau élevées)
- RPO (Recovery Point Objective) : Quelques secondes à quelques minutes (selon le lag)

**Processus de DR :**
1. Catastrophe au datacenter principal
2. Promotion du standby DR en primary
3. Redirection des applications vers le nouveau primary
4. Reconstruction du datacenter principal en parallèle

**Avantages :**
- ✅ Protection contre les catastrophes régionales
- ✅ Conformité réglementaire (exigences de géo-réplication)

### 4. Backup à Chaud (Hot Backup)

**Objectif :** Effectuer des sauvegardes (pg_dump, pg_basebackup) sans impacter le primary.

**Architecture typique :**
```
Primary (Production)
 └─> Standby Backup (Dédié aux sauvegardes)
```

**Processus :**
- Les sauvegardes sont effectuées sur le standby (pas le primary)
- Le primary continue de traiter les transactions normalement
- Aucun verrou, aucune charge supplémentaire sur le primary

**Avantages :**
- ✅ Sauvegardes sans impact sur la production
- ✅ Possibilité de sauvegardes fréquentes (ex: toutes les heures)

### 5. Environnements de Test et Développement

**Objectif :** Fournir des environnements de dev/test avec des données réelles (anonymisées).

**Architecture typique :**
```
Primary (Production)
 └─> Standby Intermediate
      ├─> Standby Dev
      ├─> Standby Test
      └─> Standby Staging
```

**Avantages :**
- ✅ Données de production en dev/test (après anonymisation)
- ✅ Isolation : Les environnements non-production ne surchargent pas le primary
- ✅ Facilité de rafraîchissement : Recréer un standby est simple

---

## Prérequis Techniques

Avant de configurer la réplication physique, assurez-vous que votre infrastructure répond aux exigences suivantes :

### 1. Infrastructure Réseau

**Connectivité :**
- Les serveurs primary et standby doivent pouvoir communiquer via TCP/IP
- Port par défaut : **5432** (configurable)
- Firewall configuré pour autoriser le trafic entre primary et standby

**Bande passante :**
- Dépend du volume d'écritures sur le primary
- Règle approximative : Volume WAL/s × 1.2 (overhead réseau)
- Exemple : Si le primary génère 50 MB/s de WAL, prévoir au moins 60 MB/s de bande passante

**Latence :**
- **Réplication asynchrone** : Tolérance élevée (< 100ms acceptable)
- **Réplication synchrone** : Latence faible requise (< 10ms recommandé)
- Mesurer la latence : `ping <standby_ip>`

### 2. Configuration Matérielle

**Serveur Primary :**
- CPU : Selon la charge applicative (4-16+ cores typiques)
- RAM : 25-40% de la taille de la base de données (pour `shared_buffers`)
- Disque : SSD recommandé, RAID 10 pour performance et résilience
- Espace disque WAL : 50-100 GB dédiés à `pg_wal/`

**Serveur Standby :**
- **Recommandation :** Configuration identique ou supérieure au primary
- Si utilisé uniquement pour HA (failover) : Peut être identique
- Si utilisé pour read replicas (lectures intensives) : Peut nécessiter plus de RAM/CPU

**Espace disque :**
- Taille de la base de données + marge pour croissance
- Espace pour les WAL : 50-100 GB minimum

### 3. Système d'Exploitation

**Versions identiques recommandées :**
- Même distribution Linux (ex: Ubuntu 24.04, CentOS 9, Debian 12)
- Même architecture CPU (x86_64 ou ARM)
- Même version de PostgreSQL (obligatoire pour la réplication physique)

**Packages requis :**
- PostgreSQL 18 installé sur tous les serveurs
- OpenSSL (pour SSL/TLS)
- Outils réseau (ping, telnet, nc) pour diagnostics

### 4. PostgreSQL

**Version :**
- **Primary et Standby doivent être sur la même version majeure** (ex: tous PostgreSQL 18.x)
- La réplication entre versions majeures différentes (ex: 17 vers 18) n'est **pas supportée** avec la réplication physique
- Pour cross-version : utiliser la réplication logique

**Configuration minimale :**
- `wal_level = replica` (sur le primary)
- `max_wal_senders > 0` (sur le primary)
- Utilisateur avec privilège `REPLICATION`

---

## Architecture de Référence

Voici quelques architectures de réplication physique couramment utilisées en production.

### Architecture 1 : HA Simple (1 Primary + 1 Standby)

```
┌──────────────────────┐
│   Primary            │
│   (Read/Write)       │
│   Paris              │
└───────────┬──────────┘
            │
            │ Streaming (Sync ou Async)
            │
            ↓
┌──────────────────────┐
│   Standby Hot        │
│   (Read-Only)        │
│   Paris ou Londres   │
└──────────────────────┘
```

**Cas d'usage :** Petites à moyennes applications nécessitant une HA de base.

**Configuration :**
- Réplication synchrone si zero data loss requis
- Réplication asynchrone si performance prioritaire

### Architecture 2 : HA + Read Replicas

```
┌──────────────────────┐
│   Primary            │
│   (Read/Write)       │
└───────────┬──────────┘
            │
            ├──────────────────────┐
            │                      │
            ↓                      ↓
┌──────────────────────┐  ┌──────────────────────┐
│   Standby HA         │  │   Standby Read       │
│   (Hot - Failover)   │  │   (Hot - Lectures)   │
│   Synchrone          │  │   Asynchrone         │
└──────────────────────┘  └──────────────────────┘
```

**Cas d'usage :** Applications avec haute disponibilité ET répartition de charge en lecture.

**Configuration :**
- Standby HA en synchrone (zero data loss)
- Standby Read en asynchrone (performance)

### Architecture 3 : Multi-Datacenter avec Cascade

```
Datacenter Paris (Primary)
┌──────────────────────┐
│   Primary            │
│   (Read/Write)       │
└───────────┬──────────┘
            │
            ├───────────────────────────────┐
            │                               │
            ↓                               ↓
┌──────────────────────┐      ┌──────────────────────┐
│   Standby Local      │      │  Standby Londres     │
│   (Paris - Hot)      │      │  (Intermédiaire)     │
└──────────────────────┘      └────────┬─────────────┘
                                       │
                              ┌────────┴────────┐
                              │                 │
                              ↓                 ↓
                    ┌──────────────────┐  ┌──────────────────┐
                    │ Standby Cascade  │  │ Standby Cascade  │
                    │ (Londres - Hot)  │  │ (Londres - Hot)  │
                    └──────────────────┘  └──────────────────┘
```

**Cas d'usage :** Grande application multi-région avec DR géographique.

**Configuration :**
- Paris → Londres : Asynchrone (latence réseau)
- Londres : Cascade pour répartir la charge localement

### Architecture 4 : Production + Non-Production

```
┌──────────────────────┐
│   Primary            │
│   (Production)       │
└───────────┬──────────┘
            │
            ├──────────────────────┬──────────────────────┐
            │                      │                      │
            ↓                      ↓                      ↓
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│   Standby HA         │  │  Standby Reporting   │  │  Standby Backup      │
│   (Sync - Failover)  │  │  (Async - BI)        │  │  (Async - Backups)   │
└──────────────────────┘  └───────────┬──────────┘  └──────────────────────┘
                                      │
                          ┌───────────┴───────────┐
                          │                       │
                          ↓                       ↓
                  ┌──────────────────┐    ┌──────────────────┐
                  │  Standby Dev     │    │  Standby Test    │
                  │  (Cascade)       │    │  (Cascade)       │
                  └──────────────────┘    └──────────────────┘
```

**Cas d'usage :** Séparation production/non-production avec isolation complète.

**Configuration :**
- Standby HA protège la production (sync)
- Standbys Dev/Test en cascade (isolés de la production)

---

## Vue d'Ensemble du Processus de Configuration

Configurer la réplication physique implique plusieurs étapes sur les serveurs primary et standby. Voici un aperçu avant d'entrer dans les détails (chapitres suivants).

### Sur le Serveur Primary

1. **Créer un utilisateur de réplication**
   - Utilisateur dédié avec privilège `REPLICATION`
   - Exemple : `CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD '***'`

2. **Configurer `postgresql.conf`**
   - `wal_level = replica`
   - `max_wal_senders = 5` (ou plus selon le nombre de standbys)
   - `wal_keep_size = 1GB` (ou utiliser des slots de réplication)
   - Optionnel : `synchronous_standby_names` (pour réplication synchrone)

3. **Configurer `pg_hba.conf`**
   - Autoriser les connexions de réplication depuis les standbys
   - Exemple : `host replication replicator 192.168.1.20/32 scram-sha-256`

4. **Créer des slots de réplication** (recommandé)
   - `SELECT pg_create_physical_replication_slot('standby_slot_1');`

5. **Redémarrer PostgreSQL**
   - Appliquer les modifications de configuration

### Sur le Serveur Standby

1. **Créer une copie initiale du primary** (Base Backup)
   - Utiliser `pg_basebackup` pour copier la base de données du primary
   - Exemple : `pg_basebackup -h <primary_ip> -D /var/lib/postgresql/18/main -U replicator -P -v -R -X stream`

2. **Configurer `postgresql.conf`**
   - `hot_standby = on` (pour accepter les lectures)
   - `primary_conninfo = 'host=<primary_ip> user=replicator password=***'`
   - `primary_slot_name = 'standby_slot_1'` (si slot utilisé)

3. **Vérifier le fichier `standby.signal`**
   - Créé automatiquement par `pg_basebackup -R`
   - Indique à PostgreSQL de démarrer en mode standby

4. **Démarrer PostgreSQL**
   - Le standby se connecte au primary et commence à recevoir/rejouer les WAL

### Vérification

- **Sur le primary :** `SELECT * FROM pg_stat_replication;` (voir le standby connecté)
- **Sur le standby :** `SELECT pg_is_in_recovery();` (doit retourner `true`)
- **Test de bout en bout :** Insérer des données sur le primary, vérifier qu'elles apparaissent sur le standby

---

## Nouveautés PostgreSQL 18 pour la Réplication

PostgreSQL 18 (sortie en septembre 2025) apporte plusieurs améliorations significatives à la réplication physique.

### 1. Sous-système I/O Asynchrone (AIO)

**Description :** PostgreSQL 18 introduit un nouveau sous-système d'I/O asynchrone qui améliore significativement les performances de lecture/écriture des WAL.

**Avantages pour la réplication :**
- ⚡ Réduction du lag de réplication de 20-30%
- ⚡ Meilleure utilisation des disques NVMe
- ⚡ Jusqu'à 3× plus rapide sur certaines charges de travail

**Configuration :**
```ini
io_method = async  # 'sync' (ancien comportement) ou 'async' (nouveau)
```

### 2. Améliorations de COPY et Marqueur CSV

**Description :** Amélioration de la commande `COPY` pour le bulk loading, réduisant la génération de WAL.

**Avantages pour la réplication :**
- Moins de WAL générés lors des imports massifs
- Réduction de la charge réseau entre primary et standby
- Lag réduit pendant les opérations de bulk loading

### 3. Support OLD et NEW dans RETURNING

**Description :** Les clauses `RETURNING` peuvent maintenant accéder aux anciennes et nouvelles valeurs des lignes (OLD et NEW).

**Avantages pour la réplication :**
- Facilite le debugging et l'audit lors de modifications répliquées
- Permet des triggers plus sophistiqués sur les standbys (après promotion)

### 4. Contraintes Temporelles (Temporal Constraints)

**Description :** Support natif des contraintes sur les périodes de temps (validité temporelle).

**Avantages pour la réplication :**
- Réplication fidèle des contraintes temporelles complexes
- Pas besoin d'extensions tierces pour gérer l'historisation

### 5. Améliorations d'EXPLAIN avec Buffers Automatiques

**Description :** `EXPLAIN` affiche maintenant automatiquement les statistiques de buffers sans nécessiter l'option explicite `BUFFERS`.

**Avantages pour la réplication :**
- Diagnostic plus facile des problèmes de performance sur les standbys
- Identification rapide des requêtes consommant beaucoup d'I/O

### 6. Optimisations du Planificateur

**Description :** Plusieurs optimisations du planificateur de requêtes :
- Auto-élimination des self-joins
- Skip Scan pour index multi-colonnes
- Transformation OR → ANY

**Avantages pour la réplication :**
- Réduction de la charge sur le primary → moins de WAL générés
- Meilleures performances sur les standbys (lectures)

### 7. Statistiques I/O et WAL par Backend

**Description :** Nouvelles statistiques détaillées sur les opérations I/O et WAL par processus backend.

**Avantages pour la réplication :**
- Monitoring précis de la réplication (WAL sender, WAL receiver)
- Identification des goulots d'étranglement réseau ou disque

### 8. Améliorations d'Autovacuum

**Description :**
- Nouveau paramètre `autovacuum_vacuum_max_threshold`
- Ajustements dynamiques des workers (`autovacuum_worker_slots`)

**Avantages pour la réplication :**
- Moins de bloat sur les standbys (après promotion)
- Gestion automatique du vacuum sur les standbys promus

### 9. Data Checksums Activés par Défaut

**Description :** Les checksums de données sont maintenant activés par défaut lors de l'initialisation d'une nouvelle instance (`initdb`).

**Avantages pour la réplication :**
- Détection automatique de corruptions de données pendant la réplication
- Protection accrue contre les corruptions silencieuses
- Nécessaire pour `pg_basebackup` avec vérification d'intégrité

**Note :** Pour désactiver (non recommandé) : `initdb --no-data-checksums`

### 10. pg_upgrade Amélioré avec Préservation des Statistiques

**Description :** `pg_upgrade` préserve maintenant les statistiques du planificateur, réduisant le temps de stabilisation après upgrade.

**Avantages pour la réplication :**
- Migrations plus rapides de standbys
- Pas de dégradation des performances après promotion d'un standby upgradé

**Option swap pour upgrade rapide :**
```bash
pg_upgrade --old-datadir=/old --new-datadir=/new --swap
```

### 11. OAuth 2.0 pour l'Authentification

**Description :** Support natif de l'authentification OAuth 2.0 pour les connexions PostgreSQL.

**Avantages pour la réplication :**
- Authentification moderne et centralisée pour les connexions de réplication
- Intégration avec des systèmes d'identité (Azure AD, Okta, etc.)
- Rotation des credentials simplifiée

**Configuration :**
```ini
# pg_hba.conf
host replication replicator 192.168.1.20/32 oauth
```

### 12. SCRAM Passthrough avec postgres_fdw et dblink

**Description :** Transmission transparente de l'authentification SCRAM via Foreign Data Wrappers.

**Avantages pour la réplication :**
- Authentification sécurisée pour les cascades complexes
- Pas besoin de stocker les mots de passe en clair dans les FDW

---

## Outils et Utilitaires pour la Réplication

PostgreSQL et son écosystème fournissent plusieurs outils pour faciliter la gestion de la réplication.

### 1. Outils Natifs PostgreSQL

**pg_basebackup**
- Crée une copie physique d'un serveur PostgreSQL
- Utilisé pour initialiser un standby
- Supporte SSL, compression, et streaming des WAL

**pg_receivewal**
- Reçoit et archive les WAL en continu depuis un serveur
- Utile pour les sauvegardes continues (PITR)

**pg_rewind**
- Resynchronise un ancien primary avec un nouveau primary après failover
- Évite de refaire un `pg_basebackup` complet

**pg_upgrade**
- Upgrade d'une version majeure PostgreSQL à une autre
- PostgreSQL 18 : Préserve les statistiques, option `--swap`

### 2. Extensions de Monitoring

**pg_stat_statements**
- Extension pour monitorer les requêtes exécutées
- Utile sur les standbys pour analyser les lectures

**pg_stat_kcache**
- Statistiques système (CPU, I/O) par requête
- Diagnostic des goulots d'étranglement sur les standbys

**HypoPG**
- Création d'index "hypothétiques" pour tester leur impact
- Utile pour optimiser les standbys dédiés aux lectures

### 3. Outils de Haute Disponibilité

**Patroni**
- Orchestrateur HA automatique pour PostgreSQL
- Gère automatiquement les failovers, élection de leader
- Intégration avec etcd, Consul, Zookeeper

**Repmgr**
- Gestionnaire de réplication et failover
- Plus simple que Patroni, moins de dépendances
- Outils de ligne de commande intuitifs

**PgBouncer**
- Connection pooler léger
- Réduit la charge de connexions sur primary et standbys
- Modes transaction, session, statement

### 4. Outils de Monitoring et Observabilité

**pgBadger**
- Analyseur de logs PostgreSQL
- Génère des rapports HTML détaillés sur l'activité

**Prometheus + postgres_exporter**
- Collecte de métriques PostgreSQL
- Intégration avec Grafana pour visualisation

**Grafana**
- Dashboards pour visualiser les métriques de réplication
- Templates communautaires disponibles

**pg_top / pg_activity**
- Équivalents PostgreSQL de `top`
- Monitoring en temps réel des processus

### 5. Outils Cloud et Managés

**AWS RDS for PostgreSQL**
- Service managé avec réplication physique intégrée
- Failover automatique, sauvegardes automatisées

**Azure Database for PostgreSQL**
- Service managé Microsoft
- Réplication inter-régions disponible

**Google Cloud SQL**
- Service managé Google Cloud
- Réplication haute disponibilité et read replicas

**Supabase, Neon, Crunchy Data**
- Alternatives PostgreSQL cloud-native
- Réplication et HA gérées automatiquement

---

## Ressources et Apprentissage

### Documentation Officielle

- **PostgreSQL 18 Documentation :** [https://www.postgresql.org/docs/18/](https://www.postgresql.org/docs/18/)
- **High Availability, Load Balancing, and Replication :** [https://www.postgresql.org/docs/18/high-availability.html](https://www.postgresql.org/docs/18/high-availability.html)
- **Warm Standby and Streaming Replication :** [https://www.postgresql.org/docs/18/warm-standby.html](https://www.postgresql.org/docs/18/warm-standby.html)

### Livres Recommandés

- **"PostgreSQL: Up and Running"** par Regina Obe et Leo Hsu
- **"Mastering PostgreSQL 13"** par Hans-Jürgen Schönig (principes valables pour v18)
- **"The Art of PostgreSQL"** par Dimitri Fontaine

### Blogs et Sites Techniques

- **2ndQuadrant Blog :** [https://www.2ndquadrant.com/en/blog/](https://www.2ndquadrant.com/en/blog/)
- **Percona Blog :** [https://www.percona.com/blog/](https://www.percona.com/blog/)
- **Crunchy Data Blog :** [https://www.crunchydata.com/blog](https://www.crunchydata.com/blog)
- **PostgreSQL Wiki :** [https://wiki.postgresql.org/](https://wiki.postgresql.org/)

### Communautés

- **pgsql-general :** Liste de diffusion officielle PostgreSQL
- **Reddit r/PostgreSQL :** [https://www.reddit.com/r/PostgreSQL/](https://www.reddit.com/r/PostgreSQL/)
- **PostgreSQL Discord :** Communauté active pour discussions en temps réel
- **Stack Overflow :** Tag `postgresql` pour questions techniques

### Conférences

- **PGConf :** Conférence annuelle mondiale PostgreSQL
- **PostgreSQL Sessions :** Événements réguliers en France et Europe
- **FOSDEM :** Track PostgreSQL/Database chaque année à Bruxelles

---

## Conclusion

La réplication physique (Streaming Replication) est un pilier fondamental de toute infrastructure PostgreSQL de production. Elle offre :

- ✅ **Haute disponibilité** : Continuité de service en cas de panne
- ✅ **Scalabilité** : Read replicas pour répartir les charges de lecture
- ✅ **Disaster Recovery** : Protection géographique des données
- ✅ **Flexibilité** : Multiples architectures possibles (synchrone, asynchrone, cascade)
- ✅ **Performance** : Overhead minimal, latence en millisecondes
- ✅ **Simplicité** : Configuration relativement accessible

**Les chapitres suivants détailleront :**

- **17.2.1. Configuration Primary/Standby** : Guide pas à pas pour configurer votre première réplication
- **17.2.2. Synchronous vs Asynchronous** : Choisir le bon mode de synchronisation selon vos besoins
- **17.2.3. Cascading Replication** : Construire des architectures multi-niveaux scalables

Avec PostgreSQL 18 et ses améliorations significatives (I/O asynchrone, checksums par défaut, OAuth), la réplication physique est plus performante et sécurisée que jamais. Maîtriser cette technologie est essentiel pour tout développeur ou administrateur PostgreSQL travaillant sur des systèmes de production critiques.

---


⏭️ [Configuration Primary/Standby](/17-haute-disponibilite-et-replication/02.1-configuration-primary-standby.md)
