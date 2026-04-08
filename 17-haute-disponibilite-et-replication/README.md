🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17. Haute Disponibilité et Réplication

## Introduction

La **haute disponibilité** (High Availability ou HA) et la **réplication** sont deux piliers essentiels des systèmes de bases de données en production. Dans un monde où les applications doivent être accessibles 24h/24 et 7j/7, où les données doivent être protégées contre toute perte, et où les performances doivent être maintenues même sous une charge intense, ces concepts deviennent non pas un luxe, mais une nécessité absolue.

Ce chapitre vous guidera à travers les différentes stratégies et technologies que PostgreSQL met à votre disposition pour construire des architectures résilientes, performantes et fiables.

---

## 1. Qu'est-ce que la Haute Disponibilité ?

### 1.1. Définition et objectifs

La **haute disponibilité** est la capacité d'un système à rester opérationnel et accessible pendant une très grande partie du temps, malgré les pannes matérielles, logicielles ou les opérations de maintenance.

**Objectif principal :**
> Minimiser les interruptions de service et garantir que votre base de données reste accessible aux utilisateurs, même en cas de problème.

### 1.2. Mesurer la disponibilité : Les "9" (nines)

La disponibilité se mesure généralement en pourcentage, souvent exprimé en nombre de "9" :

| Disponibilité | Downtime par an | Downtime par mois | Downtime par semaine | Classification |
|---------------|-----------------|-------------------|---------------------|----------------|
| 90% ("one nine") | 36,5 jours | 3 jours | 16,8 heures | Inacceptable |
| 99% ("two nines") | 3,65 jours | 7,3 heures | 1,68 heures | Faible |
| 99,9% ("three nines") | 8,76 heures | 43,8 minutes | 10,1 minutes | Basique |
| 99,99% ("four nines") | 52,6 minutes | 4,4 minutes | 1,01 minutes | Haute disponibilité |
| 99,999% ("five nines") | 5,26 minutes | 26 secondes | 6 secondes | Très haute disponibilité |
| 99,9999% ("six nines") | 31,5 secondes | 2,6 secondes | 0,6 secondes | Disponibilité extrême |

**Exemples concrets :**

**99% (deux nines) :**
```
→ Votre site est inaccessible 3,65 jours par an
→ Acceptable pour : blog personnel, site vitrine PME
→ Inacceptable pour : e-commerce, banque en ligne
```

**99,99% (quatre nines) :**
```
→ Votre site est inaccessible ~53 minutes par an
→ Standard pour : applications d'entreprise critiques
→ Coût : Modéré (réplication, monitoring)
```

**99,999% (cinq nines) :**
```
→ Votre site est inaccessible ~5 minutes par an
→ Requis pour : services financiers, télécommunications
→ Coût : Élevé (redondance complète, équipes 24/7)
```

### 1.3. Pourquoi la haute disponibilité est-elle critique ?

#### Impact Business

**E-commerce :**
- 1 heure d'indisponibilité = 100 000€ de chiffre d'affaires perdu (pour un site moyen)
- Perte de confiance des clients
- Impact sur le référencement (SEO)

**Services Financiers :**
- Obligation réglementaire (Basel III, MiFID II)
- Risque de pénalités financières
- Réputation compromise

**SaaS / Cloud :**
- SLA contractuels avec pénalités
- Churn clients (départs)
- Dommage d'image irréversible

**Santé / Urgences :**
- Vies humaines en jeu
- Impossibilité d'accéder aux dossiers médicaux
- Risque juridique majeur

#### Analogie du magasin physique

Imaginez un magasin physique qui ferme de manière imprévisible :
```
Lundi 10h : Fermé 2 heures (panne électrique)  
Mercredi 14h : Fermé 4 heures (problème informatique)  
Samedi 16h : Fermé toute l'après-midi (jour de plus forte affluence)  

Résultat :
→ Les clients vont chez le concurrent
→ Perte de revenus directe
→ Réputation ternie
```

C'est exactement ce qui se passe avec une base de données non disponible.

---

## 2. Les Causes d'Indisponibilité

Comprendre **pourquoi** un système devient indisponible est essentiel pour choisir la bonne stratégie de protection.

### 2.1. Pannes matérielles (Hardware Failures)

**Exemples courants :**

1. **Panne de disque dur**
   - Durée de vie moyenne : 3-5 ans
   - Probabilité : ~1-3% par disque et par an
   - Impact : Perte de données si pas de RAID

2. **Panne de RAM**
   - Peut causer de la corruption de données
   - Détection difficile sans ECC (Error-Correcting Code)

3. **Panne d'alimentation**
   - Coupure électrique
   - Défaillance de l'onduleur (UPS)
   - Surtension

4. **Panne réseau**
   - Carte réseau défectueuse
   - Switch tombé
   - Câble débranché (erreur humaine)

5. **Panne CPU / Carte mère**
   - Plus rare mais impact total
   - Remplacement long (plusieurs heures)

**Statistiques :**
Selon une étude de Google (2007) sur des millions de disques :
- Le taux de pannes annuel varie entre 1,7% et 8,6%
- Les pannes arrivent souvent par "vagues" (lots défectueux)
- Les disques jeunes (<1 an) et vieux (>3 ans) sont plus à risque

### 2.2. Pannes logicielles (Software Failures)

**Exemples :**

1. **Bug PostgreSQL**
   - Rare mais possible (surtout sur nouvelles versions)
   - Crash du serveur
   - Corruption de données

2. **Bug système d'exploitation**
   - Kernel panic (Linux)
   - Blue Screen of Death (Windows)

3. **Saturation de ressources**
   - Disque plein (pg_wal/ ou data)
   - Mémoire épuisée (OOM Killer)
   - Connexions saturées (max_connections)

4. **Mise à jour ratée**
   - Migration de version mal préparée
   - Extension incompatible
   - Configuration incorrecte

### 2.3. Erreurs humaines (Human Errors)

**Les plus fréquentes et dangereuses :**

1. **Suppression accidentelle de données**
   ```sql
   -- Oubli du WHERE !
   DELETE FROM customers;  -- Au lieu de DELETE FROM customers WHERE id = 123;
   ```

2. **DROP de table ou base**
   ```sql
   DROP DATABASE production;  -- Pensait être sur "dev"
   ```

3. **Modification de configuration cassante**
   ```ini
   # Oups, trop restrictif
   max_connections = 1
   ```

4. **Commande système destructive**
   ```bash
   rm -rf /var/lib/postgresql/14/main/  # Mauvais répertoire
   ```

**Statistiques alarmantes :**
- 70% des pannes en production sont dues à des erreurs humaines (Gartner)
- 50% des erreurs humaines surviennent pendant les maintenances planifiées

### 2.4. Catastrophes naturelles et sinistres

**Risques physiques :**

1. **Incendie** dans le datacenter  
2. **Inondation** (datacenter sous-sol)  
3. **Tremblement de terre**  
4. **Ouragan / Tempête**  
5. **Coupure électrique généralisée**  
6. **Attentat / Sabotage**

**Exemple réel :**
En 2021, un incendie chez OVH (hébergeur français) a détruit complètement 4 datacenters :
- 3,6 millions de sites web inaccessibles
- Données perdues pour les clients sans backup externe
- Impact économique : plusieurs centaines de millions d'euros

### 2.5. Maintenance planifiée

Même les opérations prévues causent de l'indisponibilité :

- **Mise à jour de PostgreSQL** : Redémarrage requis (5-30 minutes)  
- **Changement de configuration** : Redémarrage parfois nécessaire  
- **Migration de serveur** : Plusieurs heures de downtime  
- **Extension du stockage** : Arrêt du service

---

## 3. Les Métriques de la Haute Disponibilité

Pour évaluer et gérer la HA, on utilise des métriques standardisées :

### 3.1. RTO (Recovery Time Objective)

**Définition :**
> Temps maximum acceptable pour restaurer le service après une panne.

**Exemples :**

| Type d'application | RTO acceptable |
|-------------------|----------------|
| Blog personnel | 24 heures |
| Site e-commerce PME | 1 heure |
| Plateforme bancaire | 15 minutes |
| Service d'urgence médical | 30 secondes |

**Implications techniques :**

- **RTO = 24h** : Backup quotidien + restauration manuelle suffit  
- **RTO = 1h** : Hot standby + monitoring + procédure de failover  
- **RTO = 15 min** : Réplication synchrone + failover automatisé (Patroni)  
- **RTO = 30s** : Multi-master ou réplication quasi-synchrone + détection automatique

### 3.2. RPO (Recovery Point Objective)

**Définition :**
> Quantité maximale de données qu'on peut se permettre de perdre, exprimée en temps.

**Exemples :**

| Type d'application | RPO acceptable | Perte maximale |
|-------------------|----------------|----------------|
| Blog | 24 heures | Articles de la journée |
| E-commerce | 5 minutes | Quelques commandes |
| Banque | 0 secondes | Aucune transaction |
| Réseau social | 1 minute | Quelques posts/likes |

**Implications techniques :**

- **RPO = 24h** : Backup quotidien nocturne  
- **RPO = 1h** : Archivage WAL toutes les heures  
- **RPO = 5 min** : Réplication asynchrone + WAL shipping continu  
- **RPO = 0** : Réplication synchrone (sacrifice de performance)

### 3.3. MTBF (Mean Time Between Failures)

**Définition :**
> Temps moyen entre deux pannes.

**Exemple :**
```
Si un serveur tombe en panne 4 fois par an :
→ MTBF = 365 jours / 4 = 91,25 jours
→ En moyenne, le serveur fonctionne 91 jours avant de tomber en panne
```

**Utilité :**
- Évaluer la fiabilité du matériel
- Planifier les remplacements préventifs
- Dimensionner les garanties et SLA

### 3.4. MTTR (Mean Time To Recovery)

**Définition :**
> Temps moyen nécessaire pour réparer et remettre en service après une panne.

**Exemples :**

| Type de panne | MTTR typique |
|---------------|--------------|
| Redémarrage serveur | 2-5 minutes |
| Remplacement disque (avec RAID) | 30-60 minutes |
| Remplacement serveur complet | 4-8 heures |
| Restauration depuis backup | 1-6 heures |
| Failover automatique | 30 secondes - 2 minutes |

**Calcul de disponibilité :**
```
Disponibilité = MTBF / (MTBF + MTTR)

Exemple :  
MTBF = 90 jours = 2160 heures  
MTTR = 2 heures  

Disponibilité = 2160 / (2160 + 2) = 99,9%
```

---

## 4. Les Stratégies de Haute Disponibilité

PostgreSQL offre plusieurs approches pour atteindre la haute disponibilité, chacune avec ses avantages et compromis.

### 4.1. Vue d'ensemble des stratégies

```
┌────────────────────────────────────────────────────────────┐
│                 Stratégies de HA PostgreSQL                │
└────────────────────────────────────────────────────────────┘

1. Backups & Point-In-Time Recovery (PITR)
   └─ Protection : Erreurs humaines, corruption
   └─ RTO : Heures    RPO : Minutes
   └─ Coût : Faible

2. Réplication Physique (Streaming Replication)
   └─ Protection : Pannes matérielles, catastrophes
   └─ RTO : Minutes   RPO : Secondes
   └─ Coût : Moyen

3. Réplication Logique
   └─ Protection : Migrations, réplication sélective
   └─ RTO : Minutes   RPO : Secondes
   └─ Coût : Moyen-Élevé

4. Pooling de Connexions
   └─ Protection : Surcharge, saturation connexions
   └─ Impact : Améliore la disponibilité sous charge
   └─ Coût : Faible

5. Haute Disponibilité Automatisée (Patroni, Repmgr)
   └─ Protection : Automatisation du failover
   └─ RTO : Secondes  RPO : Secondes
   └─ Coût : Élevé
```

### 4.2. Backups et PITR (Point-In-Time Recovery)

**Principe :**
- Sauvegarde régulière complète (pg_basebackup)
- Archivage continu des fichiers WAL
- Restauration possible à n'importe quel point dans le temps

**Avantages :**
- ✅ Protection contre les erreurs humaines  
- ✅ Protection contre la corruption de données  
- ✅ Conformité réglementaire (audit trail)  
- ✅ Coût faible

**Inconvénients :**
- ❌ RTO élevé (plusieurs heures pour restaurer)  
- ❌ Nécessite une procédure manuelle  
- ❌ Ne protège pas contre les pannes en temps réel

**Cas d'usage :**
- Couche de protection de base (obligatoire pour TOUS les systèmes)
- Applications non critiques (RTO > 1 heure acceptable)
- Protection contre le "DROP TABLE" accidentel

### 4.3. Réplication Physique (Streaming Replication)

**Principe :**
- Serveur principal (Primary) envoie les modifications binaires (WAL)
- Serveur(s) secondaire(s) (Standby) applique ces modifications
- Copie bit-à-bit identique

**Architecture simple :**
```
┌─────────────────┐
│    Primary      │  (Lecture/Écriture)
│   (Production)  │
└────────┬────────┘
         │ WAL Stream
         │ (binaire)
    ┌────┴─────┐
    ▼          ▼
┌─────────┐ ┌─────────┐
│Standby 1│ │Standby 2│  (Lecture seule)
│ (Paris) │ │(Londres)│
└─────────┘ └─────────┘
```

**Modes :**

1. **Asynchrone** (par défaut)
   - Le primary n'attend pas la confirmation du standby
   - Performance maximale
   - RPO : quelques secondes (risque de perte de données si crash)

2. **Synchrone**
   - Le primary attend la confirmation du standby avant COMMIT
   - RPO = 0 (aucune perte de données)
   - Performance impactée (latence réseau)

**Avantages :**
- ✅ RTO faible (quelques minutes avec failover manuel)  
- ✅ RPO faible (secondes en async, 0 en sync)  
- ✅ Standbys utilisables en lecture (load balancing)  
- ✅ Protection contre pannes matérielles et catastrophes

**Inconvénients :**
- ❌ Ne protège PAS contre les erreurs humaines (répliquées instantanément)  
- ❌ Architecture identique requise (version, CPU)  
- ❌ Complexité de configuration

**Cas d'usage :**
- Applications critiques (e-commerce, SaaS)
- Architecture multi-datacenter
- Disaster Recovery (DR)

### 4.4. Réplication Logique

**Principe :**
- Décodage du WAL en changements logiques (INSERT, UPDATE, DELETE)
- Réplication sélective (certaines tables uniquement)
- Versions PostgreSQL différentes possibles

**Architecture :**
```
┌──────────────────┐
│  Primary (PG 14) │
│                  │
│  Publication:    │
│  - users         │
│  - orders        │
└────────┬─────────┘
         │ Changements SQL
         │ (format logique)
    ┌────┴─────┐
    ▼          ▼
┌──────────┐ ┌──────────────┐
│Secondary │ │  Analytics   │
│ (PG 15)  │ │  (PG 14)     │
│          │ │  tables: *   │
└──────────┘ └──────────────┘
```

**Avantages :**
- ✅ Réplication sélective (tables spécifiques)  
- ✅ Versions PostgreSQL différentes  
- ✅ Transformations possibles (ETL-like)  
- ✅ Architectures hétérogènes

**Inconvénients :**
- ❌ Overhead supérieur (décodage + encodage)  
- ❌ Ne réplique pas le DDL automatiquement  
- ❌ Gestion des conflits plus complexe  
- ❌ Ne réplique pas les SEQUENCES ni LARGE OBJECTS

**Cas d'usage :**
- Migrations en ligne (upgrade de version)
- Réplication vers data warehouse analytique
- Multi-master (avec précautions)
- Synchronisation inter-régions avec filtrage

### 4.5. Pooling de Connexions (PgBouncer, PgPool-II)

**Principe :**
- Pool de connexions réutilisables
- Évite la création/destruction coûteuse de connexions
- Limite le nombre de connexions PostgreSQL

**Architecture :**
```
Applications (1000 clients)
         │
         ▼
┌──────────────────┐
│    PgBouncer     │  Pool: 100 connexions
│  (Connection     │
│   Pooler)        │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   PostgreSQL     │  max_connections = 100
│   (Primary)      │
└──────────────────┘
```

**Modes de pooling :**

1. **Session pooling**
   - Connexion assignée pour toute la session
   - Compatible avec toutes les fonctionnalités PG
   - Réduction modérée des connexions

2. **Transaction pooling** (recommandé)
   - Connexion rendue au pool après chaque transaction
   - Réduction maximale des connexions
   - Incompatible avec certaines fonctionnalités (LISTEN/NOTIFY, cursors)

3. **Statement pooling**
   - Connexion rendue après chaque requête
   - Très agressif, rarement utilisé

**Avantages :**
- ✅ Améliore dramatiquement la capacité de charge  
- ✅ Protège contre la saturation de connexions  
- ✅ Réduit la consommation mémoire  
- ✅ Transparence pour l'application

**Inconvénients :**
- ❌ Point de défaillance unique (si pas redondé)  
- ❌ Complexité ajoutée dans l'architecture  
- ❌ Debugging plus difficile

**Cas d'usage :**
- Applications web avec nombreux clients concurrents
- Serverless (AWS Lambda, Azure Functions)
- Protection contre connection storms

### 4.6. Haute Disponibilité Automatisée

**Solutions :**

1. **Patroni**
   - Gestionnaire de cluster HA open-source
   - Utilise etcd/Consul/ZooKeeper pour le consensus
   - Failover automatique en ~30 secondes

2. **Repmgr**
   - Suite d'outils de gestion de réplication
   - Failover automatique ou manuel
   - Plus simple que Patroni

3. **Cloud-Native (AWS RDS, Azure, GCP)**
   - HA entièrement gérée par le provider
   - Failover automatique transparent
   - Configuration simplifiée

**Architecture Patroni (exemple) :**
```
┌──────────────────────────────────┐
│  etcd / Consul / ZooKeeper       │  (Consensus distribué)
│  (Qui est le leader ?)           │
└──────────┬───────────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐  ┌─────────┐
│Patroni  │  │Patroni  │
│  +      │  │  +      │
│Primary  │  │Standby  │
└─────────┘  └─────────┘
     ▲           │
     │           │
     └───────────┘
    Failover automatique
    si Primary down
```

**Avantages :**
- ✅ RTO minimal (30 secondes - 2 minutes)  
- ✅ Aucune intervention humaine requise  
- ✅ Détection automatique des pannes  
- ✅ Reconfiguration automatique du cluster

**Inconvénients :**
- ❌ Complexité très élevée  
- ❌ Nécessite une expertise spécifique  
- ❌ Infrastructure additionnelle (consensus service)  
- ❌ Coût élevé (équipe, matériel)

**Cas d'usage :**
- Applications critiques 24/7
- Exigence de RTO < 5 minutes
- Budget IT conséquent

---

## 5. Choisir la Bonne Stratégie

### 5.1. Matrice de décision

| Critère | Backup/PITR | Réplication Async | Réplication Sync | HA Automatisée |
|---------|-------------|-------------------|------------------|----------------|
| **RTO** | Heures | Minutes | Minutes | Secondes |
| **RPO** | Minutes | Secondes | 0 | 0 |
| **Coût** | € | €€ | €€ | €€€€ |
| **Complexité** | Faible | Moyenne | Moyenne | Très élevée |
| **Erreurs humaines** | ✅ | ❌ | ❌ | ❌ |
| **Pannes matérielles** | ⚠️ | ✅ | ✅ | ✅ |
| **Catastrophes** | ✅ | ✅ | ✅ | ✅ |
| **Lecture distribuée** | ❌ | ✅ | ✅ | ✅ |

### 5.2. Recommandations par type d'application

#### Startup / MVP
```
Stratégie : Backup/PITR uniquement
- Backup quotidien automatisé
- Archivage WAL vers S3/GCS
- RPO : 1 heure, RTO : 4 heures
- Coût : ~100€/mois
```

#### PME / Application d'entreprise
```
Stratégie : Réplication Async + Backup/PITR
- 1 Primary + 1 Standby async
- Backup quotidien
- Archivage WAL continu
- RPO : 30 secondes, RTO : 15 minutes
- Coût : ~500€/mois
```

#### Startup en croissance / SaaS
```
Stratégie : Réplication Sync + Backup/PITR + Pooling
- 1 Primary + 2 Standbys (1 sync, 1 async)
- PgBouncer pour gérer la charge
- Backup quotidien + PITR
- RPO : 0, RTO : 10 minutes
- Coût : ~2000€/mois
```

#### Entreprise critique / E-commerce majeur
```
Stratégie : HA Automatisée (Patroni) + Multi-datacenter
- Cluster Patroni 3 nœuds minimum
- Réplication synchrone
- PgBouncer redondé (HAProxy)
- Multi-datacenter (DR)
- Équipe DevOps dédiée
- RPO : 0, RTO : 30 secondes
- Coût : ~10000€/mois
```

#### Banque / Service critique
```
Stratégie : HA Full-Stack avec tous les niveaux
- Patroni + Multi-datacenter + Multi-région
- Réplication synchrone quorum-based
- Backup continu + PITR + archivage long terme
- Monitoring 24/7 + Astreintes
- Tests de failover hebdomadaires
- RPO : 0, RTO : 15 secondes
- Coût : ~50000€/mois
```

### 5.3. L'approche par couches (Defense in Depth)

**Recommandation universelle :** Ne jamais dépendre d'une seule stratégie !

```
Couche 1 : BACKUPS (obligatoire)
  └─ Protection contre erreurs humaines et corruption

Couche 2 : RÉPLICATION (recommandée)
  └─ Protection contre pannes matérielles

Couche 3 : MONITORING & ALERTING (obligatoire)
  └─ Détection proactive des problèmes

Couche 4 : TESTS RÉGULIERS (critique)
  └─ Validation que les protections fonctionnent

Couche 5 : PROCÉDURES DOCUMENTÉES (essentielle)
  └─ Runbooks pour réagir rapidement
```

---

## 6. Ce que Vous Allez Apprendre

Ce chapitre 17 est organisé en plusieurs sous-chapitres qui couvrent progressivement tous les aspects de la haute disponibilité :

### 6.1. Fondamentaux (Sections 17.1 à 17.4)

**17.1. Le concept de WAL Shipping et archive_mode**
- Comprendre le Write-Ahead Log (WAL)
- Configuration de l'archivage continu
- Base pour PITR et réplication
- Surveillance et maintenance

**17.2. Réplication Physique (Streaming Replication)**
- Configuration Primary/Standby
- Modes synchrone vs asynchrone
- Hot Standby (lecture sur replica)
- Cascading replication

**17.3. Réplication Logique**
- Publications et Subscriptions
- Cas d'usage et limitations
- Réplication sélective
- Gestion des conflits

**17.4. Slots de réplication : Physical vs Logical**
- Prévention de la perte de WAL
- Différences entre slots physiques et logiques
- Monitoring et maintenance
- Risques et mitigation

### 6.2. Opérations (Sections 17.5 à 17.6)

**17.5. Failover et Promotion**
- Promotion manuelle d'un standby
- Détection de panne
- Procédures de basculement
- Reconstruction après failover

**17.6. Architectures HA complètes**
- Patroni : HA automatisée avec consensus
- Repmgr : Gestion de réplication simplifiée
- Architectures multi-datacenter
- Load balancing avec HAProxy/PgPool-II

---

## 7. Prérequis et Préparation

### 7.1. Connaissances requises

Pour aborder ce chapitre sereinement, vous devriez être à l'aise avec :

✅ **PostgreSQL de base :**
- Installation et configuration
- Fichier postgresql.conf et pg_hba.conf
- Notion de cluster et data directory

✅ **Administration système :**
- Linux : fichiers, permissions, services (systemd)
- Réseau : IP, ports, pare-feu
- Stockage : disques, partitions, montages

✅ **SQL :**
- Requêtes de base (SELECT, INSERT, UPDATE, DELETE)
- Notions de transactions (COMMIT, ROLLBACK)

### 7.2. Environnement de test

Pour expérimenter (en dehors de ce tutoriel théorique), il est recommandé d'avoir :

**Option 1 : Machines virtuelles locales**
```
- 3 VM Linux (Ubuntu 22.04 / Debian 12)
- 2 CPU, 4 GB RAM chacune
- Réseau privé entre les VMs
```

**Option 2 : Cloud (AWS, GCP, Azure)**
```
- 3 instances t3.small (ou équivalent)
- Réseau VPC privé
- Coût : ~100€/mois (pensez à éteindre quand inutilisé !)
```

**Option 3 : Docker (pour tests rapides)**
```
- Docker Compose avec 3 conteneurs PostgreSQL
- Réseau bridge personnalisé
- Volumes persistants
```

### 7.3. Versions PostgreSQL

Les concepts de ce chapitre s'appliquent à partir de **PostgreSQL 10** (introduction de la réplication logique native).

**Versions recommandées :**
- PostgreSQL 14+ : Améliorations de performance et monitoring
- PostgreSQL 15+ : Amélioration de la réplication logique
- **PostgreSQL 18** : I/O asynchrone, optimisations majeures (focus de ce tutoriel)

---

## 8. Avertissements et Bonnes Pratiques

### 8.1. Avertissements importants

⚠️ **La haute disponibilité ne remplace PAS les backups**
- Même avec réplication, vous devez avoir des backups réguliers
- La réplication réplique aussi les erreurs (DROP TABLE, suppression de données)

⚠️ **Testez, testez, testez !**
- Un plan de HA non testé est un plan qui échouera en production
- Organisez des "game days" : simulez des pannes volontairement

⚠️ **Documentez tout**
- Procédures de failover écrites et à jour
- Runbooks accessibles 24/7
- Contacts d'astreinte clairement définis

⚠️ **Surveillez proactivement**
- Le monitoring n'est pas optionnel
- Alertes configurées sur toutes les métriques critiques
- Tests des alertes réguliers

⚠️ **La réplication synchrone a un coût**
- Latence réseau impacte directement les performances
- Peut diviser le débit par 2 ou plus
- À utiliser avec discernement

### 8.2. Checklist avant de commencer

Avant d'implémenter une stratégie HA en production :

**Infrastructure :**
- [ ] Serveurs redondants disponibles  
- [ ] Réseau stable et surveillé  
- [ ] Stockage avec redondance (RAID, SAN)  
- [ ] Monitoring en place (Prometheus, Grafana, etc.)

**PostgreSQL :**
- [ ] Version supportée (ideally LTS)  
- [ ] Configuration optimisée  
- [ ] Backups testés et fonctionnels  
- [ ] Procédure de restauration documentée

**Équipe :**
- [ ] Compétences PostgreSQL et Linux  
- [ ] Astreintes organisées (24/7 pour HA critique)  
- [ ] Formation sur les outils (Patroni, Repmgr)  
- [ ] Runbooks écrits et validés

**Processus :**
- [ ] Tests de basculement planifiés  
- [ ] Métriques RTO/RPO définies  
- [ ] SLA contractualisés (si applicable)  
- [ ] Post-mortem après chaque incident

---

## 9. Conclusion de l'Introduction

La **haute disponibilité** n'est pas un produit que l'on achète, mais un **état** que l'on atteint grâce à une combinaison de :
- Technologies appropriées (réplication, pooling, clustering)
- Architecture résiliente (redondance, géo-distribution)
- Processus rigoureux (monitoring, tests, documentation)
- Culture d'équipe (proactivité, amélioration continue)

PostgreSQL offre tous les outils nécessaires pour construire des architectures hautement disponibles, de la simple réplication à deux nœuds jusqu'aux clusters multi-datacenter automatisés.

Dans les sections suivantes, nous allons explorer chacune de ces technologies en détail, avec des explications accessibles, des exemples concrets, et les meilleures pratiques éprouvées par des années de production.

**Rappelez-vous :**
> "La meilleure stratégie HA est celle que vous comprenez, que vous testez régulièrement, et qui correspond réellement à vos besoins métier."

Ne visez pas la perfection théorique (six nines à tout prix), mais l'équilibre optimal entre disponibilité, coût, et complexité pour **votre** contexte spécifique.

---

## Prêt à commencer ?

Passons maintenant aux fondamentaux en commençant par le **WAL Shipping** et l'**archive_mode**, la pierre angulaire de toute stratégie de haute disponibilité avec PostgreSQL.

---


⏭️ [Le concept de WAL Shipping et archive_mode](/17-haute-disponibilite-et-replication/01-wal-shipping-archive-mode.md)
