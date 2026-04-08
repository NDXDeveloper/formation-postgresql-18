🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20bis.4. Intégration avec Kubernetes

## Introduction

**Kubernetes** est devenu la plateforme standard pour orchestrer les applications conteneurisées. Des startups aux géants de la tech, des milliers d'organisations l'utilisent pour déployer, scaler et gérer leurs applications.

Mais qu'en est-il des bases de données ? Pendant longtemps, la sagesse conventionnelle déconseillait d'exécuter des bases de données "stateful" comme PostgreSQL sur Kubernetes. Cette époque est révolue. Aujourd'hui, avec les bons outils et les bonnes pratiques, PostgreSQL peut non seulement fonctionner sur Kubernetes, mais y prospérer.

Ce chapitre vous guide à travers l'intégration de PostgreSQL avec Kubernetes, des concepts fondamentaux aux architectures de production avancées.

---

## Pourquoi Kubernetes ?

### Le Contexte : L'Évolution du Déploiement

L'infrastructure a considérablement évolué au fil des décennies :

```
ÉVOLUTION DU DÉPLOIEMENT
────────────────────────

1990s-2000s : SERVEURS PHYSIQUES
┌─────────────────────────────────────────────────────────────────┐
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                          │
│  │Serveur 1│  │Serveur 2│  │Serveur 3│   • Coûteux              │
│  │  (App)  │  │  (DB)   │  │ (Web)   │   • Long à provisionner  │
│  └─────────┘  └─────────┘  └─────────┘   • Sous-utilisé         │
└─────────────────────────────────────────────────────────────────┘

2000s-2010s : VIRTUALISATION
┌─────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────────────────────────────┐                    │
│  │              Hyperviseur                    │                    │
│  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐         │   • Meilleure      │
│  │  │ VM1 │  │ VM2 │  │ VM3 │  │ VM4 │         │     utilisation    │
│  │  └─────┘  └─────┘  └─────┘  └─────┘         │   • Plus flexible  │
│  └─────────────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────────┘

2010s-Présent : CONTENEURS + ORCHESTRATION
┌────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    KUBERNETES                           │   │
│  │  ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐     │   │
│  │  │ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C │     │   │
│  │  └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘     │   │
│  │                                                         │   │
│  │  • Déploiement en secondes    • Auto-scaling            │   │
│  │  • Portabilité maximale       • Self-healing            │   │
│  │  • Utilisation optimale       • Infrastructure as Code  │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

### Qu'est-ce que Kubernetes ?

**Kubernetes** (du grec "κυβερνήτης", signifiant "timonier" ou "pilote") est une plateforme open source d'orchestration de conteneurs, initialement développée par Google et maintenant maintenue par la CNCF (Cloud Native Computing Foundation).

En termes simples, Kubernetes :

- **Déploie** vos applications conteneurisées  
- **Scale** automatiquement selon la charge  
- **Répare** les applications défaillantes (self-healing)  
- **Équilibre** le trafic entre les instances  
- **Gère** la configuration et les secrets  
- **Abstrait** l'infrastructure sous-jacente

### Les Avantages de Kubernetes

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AVANTAGES DE KUBERNETES                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  🔄 PORTABILITÉ                                                     │
│     Même plateforme sur AWS, GCP, Azure, on-premise                 │
│     Évite le vendor lock-in                                         │
│                                                                     │
│  📈 SCALABILITÉ                                                     │
│     Scale horizontal automatique                                    │
│     De 1 à 1000 instances sans effort                               │
│                                                                     │
│  🔧 DÉCLARATIF                                                      │
│     "Infrastructure as Code"                                        │
│     Décrivez l'état désiré, Kubernetes s'en occupe                  │
│                                                                     │
│  🛡️ RÉSILIENCE                                                      │
│     Self-healing : redémarre les conteneurs défaillants             │
│     Rolling updates sans downtime                                   │
│                                                                     │
│  🌐 ÉCOSYSTÈME                                                      │
│     Helm, Operators, Service Mesh                                   │
│     Milliers d'outils et intégrations                               │
│                                                                     │
│  👥 STANDARD DE L'INDUSTRIE                                         │
│     Compétences transférables                                       │
│     Large communauté et support                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## PostgreSQL sur Kubernetes : Le Débat

### L'Ancienne Sagesse : "Pets vs Cattle"

Pendant des années, on classait les serveurs en deux catégories :

```
PETS VS CATTLE
──────────────

🐕 PETS (Animaux de compagnie)                🐄 CATTLE (Bétail)
─────────────────────────────                 ────────────────────

• Serveurs uniques et irremplaçables          • Serveurs interchangeables
• Nommés individuellement                     • Numérotés (server-001, ...)
• Réparés quand malades                       • Remplacés quand défaillants
• Provisionnés manuellement                   • Provisionnés automatiquement

Exemples :                                    Exemples :
• Serveurs de bases de données               • Serveurs web
• Serveurs de fichiers legacy                • Workers de traitement
• Mainframes                                  • Instances cloud

PostgreSQL était considéré comme un "Pet" :
- Données critiques et irremplaçables
- Configuration unique et complexe
- Impossible à redémarrer n'importe où
```

### La Nouvelle Réalité : "Cattle" avec État

Les avancées technologiques ont changé la donne :

```
CE QUI A CHANGÉ
───────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  AVANT (2015)                         MAINTENANT (2024+)            │
│  ───────────                          ──────────────────            │
│                                                                     │
│  Stockage :                           Stockage :                    │
│  • Disques locaux uniquement          • CSI (Container Storage      │
│  • Pas de persistance K8s               Interface)                  │
│                                       • Stockage cloud haute perf   │
│                                       • Snapshots natifs            │
│                                                                     │
│  Kubernetes :                         Kubernetes :                  │
│  • StatefulSets immatures             • StatefulSets matures        │
│  • Pas d'Operators                    • Operators PostgreSQL        │
│  • Networking basique                   production-ready            │
│                                       • Service Mesh                │
│                                                                     │
│  Outils :                             Outils :                      │
│  • Backup manuel                      • Backup automatisé           │
│  • Failover manuel                    • Failover automatique        │
│  • Monitoring ad-hoc                  • Observabilité intégrée      │
│                                                                     │
│  Maturité :                           Maturité :                    │
│  • Peu d'expérience en prod          • Des milliers de clusters     │
│  • Patterns non établis                 en production               │
│                                       • Best practices documentées  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Qui Exécute PostgreSQL sur Kubernetes Aujourd'hui ?

De nombreuses organisations de premier plan :

| Organisation | Échelle | Cas d'Usage |
|--------------|---------|-------------|
| **Zalando** | 1000+ clusters | E-commerce, tous les workloads |
| **GitLab** | Production complète | SaaS et self-hosted |
| **Alibaba** | Massive scale | Cloud et e-commerce |
| **CERN** | Recherche scientifique | Données expérimentales |
| **Startups** | Nombreuses | Simplicité opérationnelle |

---

## Pourquoi Exécuter PostgreSQL sur Kubernetes ?

### Les Arguments Pour

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AVANTAGES DE PG SUR K8S                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✅ UNIFORMITÉ OPÉRATIONNELLE                                       │
│     Une seule plateforme pour tout (apps + DB)                      │
│     Mêmes outils de déploiement et monitoring                       │
│     Réduction de la complexité opérationnelle                       │
│                                                                     │
│  ✅ AUTOMATISATION AVANCÉE                                          │
│     Failover automatique avec les Operators                         │
│     Backups planifiés                                               │
│     Scaling déclaratif                                              │
│                                                                     │
│  ✅ INFRASTRUCTURE AS CODE                                          │
│     Configuration versionnée dans Git                               │
│     Reproductibilité totale                                         │
│     Environnements identiques (dev/staging/prod)                    │
│                                                                     │
│  ✅ PORTABILITÉ                                                     │
│     Même déploiement sur n'importe quel cloud                       │
│     Migration facilitée                                             │
│     Évite le vendor lock-in                                         │
│                                                                     │
│  ✅ EFFICACITÉ DES RESSOURCES                                       │
│     Meilleure utilisation des ressources                            │
│     Bin packing automatique                                         │
│     Réduction des coûts                                             │
│                                                                     │
│  ✅ ENVIRONNEMENTS ÉPHÉMÈRES                                        │
│     Base de données par feature branch                              │
│     Tests d'intégration isolés                                      │
│     Preview environments                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Les Arguments Contre (et Contre-Arguments)

| Préoccupation | Réalité |
|---------------|---------|
| **"Performances dégradées"** | Avec un stockage approprié (SSD, CSI optimisé), les performances sont comparables. Certains reportent même des améliorations. |
| **"Trop complexe"** | Les Operators réduisent drastiquement la complexité. Un cluster PostgreSQL HA en 30 lignes de YAML. |
| **"Pas production-ready"** | Des milliers de clusters en production. Operators matures depuis 5+ ans. |
| **"Risque de perte de données"** | Stockage persistant, réplication, backups automatisés. Les mêmes protections qu'ailleurs. |
| **"Compétences rares"** | Kubernetes est devenu un standard. PostgreSQL reste PostgreSQL. |

### Quand NE PAS Utiliser Kubernetes pour PostgreSQL

Il existe des cas où d'autres options peuvent être préférables :

```
ALTERNATIVES À CONSIDÉRER
─────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  🏢 TRÈS GRANDES BASES (>10 TB)                                     │
│     Considérer : Bare metal, VM dédiées                             │
│     Raison : Contrôle total sur l'I/O, tuning fin                   │
│                                                                     │
│  💰 BUDGET LIMITÉ + PAS DE K8S EXISTANT                             │
│     Considérer : Services managés (RDS, Cloud SQL)                  │
│     Raison : Pas de coût d'apprentissage K8s                        │
│                                                                     │
│  📊 WORKLOADS OLAP INTENSIFS                                        │
│     Considérer : Solutions spécialisées (Redshift, BigQuery)        │
│     Raison : Optimisées pour l'analytique                           │
│                                                                     │
│  🏛️ ENVIRONNEMENTS TRÈS RÉGULÉS                                     │
│     Considérer : Solutions certifiées spécifiques                   │
│     Raison : Exigences de conformité strictes                       │
│                                                                     │
│  👴 APPLICATIONS LEGACY MONOLITHIQUES                               │
│     Considérer : Garder l'infrastructure existante                  │
│     Raison : Migration risquée pour peu de bénéfice                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Les Défis de PostgreSQL sur Kubernetes

Exécuter une base de données stateful sur une plateforme conçue pour le stateless présente des défis spécifiques.

### Défi 1 : La Persistance des Données

```
LE PROBLÈME DE LA PERSISTANCE
─────────────────────────────

Kubernetes et les conteneurs sont ÉPHÉMÈRES par nature :

┌──────────────────────────────────────────────────────────────────┐
│  Container                                                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │    Filesystem temporaire                                   │  │
│  │    (Perdu à chaque redémarrage)                            │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  PostgreSQL a BESOIN de persistance :                            │
│  • Fichiers de données (/var/lib/postgresql/data)                │
│  • WAL (Write-Ahead Log)                                         │
│  • Configuration                                                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

SOLUTION : PersistentVolumes
────────────────────────────
┌─────────────────────────────────────────────────────────────────┐
│  Container                         PersistentVolume             │
│  ┌────────────────────┐           ┌────────────────────┐        │
│  │                    │           │                    │        │
│  │  PostgreSQL        │◄─────────►│  Stockage durable  │        │
│  │  /var/lib/pg/data  │  (mount)  │  (EBS, GCE PD,     │        │
│  │                    │           │   Azure Disk...)   │        │
│  └────────────────────┘           └────────────────────┘        │
│                                                                 │
│  Le volume survit à la destruction du Pod                       │
└─────────────────────────────────────────────────────────────────┘
```

### Défi 2 : L'Identité Stable

```
LE PROBLÈME DE L'IDENTITÉ
─────────────────────────

Les Pods Kubernetes sont normalement INTERCHANGEABLES :

┌────────────────────────────────────────────────────────────────┐
│  Deployment standard :                                         │
│                                                                │
│  web-abc123  ──► web-xyz789  ──► web-def456                    │
│                                                                │
│  Noms aléatoires, pas d'identité fixe                          │
│  OK pour les applications stateless !                          │
└────────────────────────────────────────────────────────────────┘

PostgreSQL a BESOIN d'identité stable :

┌────────────────────────────────────────────────────────────────┐
│  Cluster PostgreSQL :                                          │
│                                                                │
│  postgres-0 (Primary)     ──► Toujours le même rôle            │
│       │                                                        │
│       ├── postgres-1 (Replica)  ──► Sait où répliquer          │
│       │                                                        │
│       └── postgres-2 (Replica)  ──► Sait où répliquer          │
│                                                                │
│  Noms prévisibles, rôles définis                               │
└────────────────────────────────────────────────────────────────┘

SOLUTION : StatefulSets
───────────────────────
• Noms de Pods ordonnés et prévisibles (postgres-0, postgres-1, ...)
• Stockage dédié par Pod
• Démarrage et arrêt ordonnés
```

### Défi 3 : Le Réseau

```
LE PROBLÈME DU RÉSEAU
─────────────────────

Les applications doivent trouver PostgreSQL :

┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Application ──?──► Où est le Primary PostgreSQL ?             │
│                                                                │
│  • L'IP des Pods change à chaque redémarrage                   │
│  • Le Primary peut changer (failover)                          │
│  • Les Replicas doivent être découvrables pour les lectures    │
│                                                                │
└────────────────────────────────────────────────────────────────┘

SOLUTIONS : Services Kubernetes
───────────────────────────────

┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Service "postgres-primary"  ──► Toujours vers le Primary      │
│  (ClusterIP stable)                                            │
│                                                                │
│  Service "postgres-replica"  ──► Load balance vers Replicas    │
│  (ClusterIP stable)                                            │
│                                                                │
│  Service Headless            ──► Accès direct à chaque Pod     │
│  (DNS par Pod)                   postgres-0.postgres-svc       │
│                                  postgres-1.postgres-svc       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Défi 4 : La Haute Disponibilité

```
LE PROBLÈME DE LA HA
────────────────────

Que se passe-t-il quand le Primary tombe ?

┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  postgres-0 (Primary)  ──► CRASH !                             │
│       │                                                        │
│       ├── postgres-1 (Replica)  ──► Qui devient Primary ?      │
│       │                                                        │
│       └── postgres-2 (Replica)  ──► Comment le savoir ?        │
│                                                                │
│  Kubernetes seul ne gère PAS le failover PostgreSQL            │
│  Il ne sait pas quel replica promouvoir                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘

SOLUTION : Operators avec Patroni/Leader Election
─────────────────────────────────────────────────

┌────────────────────────────────────────────────────────────────┐
│                                                                │
│                    OPERATOR                                    │
│                       │                                        │
│                       │ Surveille et orchestre                 │
│                       ▼                                        │
│  postgres-0 (Primary)  ──► Tombe                               │
│                                                                │
│  OPERATOR détecte la panne (quelques secondes)                 │
│       │                                                        │
│       ├──► Élit postgres-1 comme nouveau Primary               │
│       │                                                        │
│       └──► Reconfigure postgres-2 pour répliquer               │
│            depuis postgres-1                                   │
│                                                                │
│  Failover automatique en ~10-30 secondes                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Défi 5 : Les Backups et la Restauration

```
LE PROBLÈME DES BACKUPS
───────────────────────

┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Questions critiques :                                         │
│                                                                │
│  • Comment sauvegarder PostgreSQL automatiquement ?            │
│  • Où stocker les backups ? (pas dans le cluster !)            │
│  • Comment restaurer rapidement ?                              │
│  • Comment faire du Point-In-Time Recovery ?                   │
│                                                                │
│  Kubernetes seul ne fournit pas ces fonctionnalités            │
│                                                                │
└────────────────────────────────────────────────────────────────┘

SOLUTIONS :
───────────

1. Operators avec backup intégré (pgBackRest, Barman, WAL-G)
2. Velero pour backup des ressources K8s + volumes
3. Snapshots de PersistentVolumes
```

---

## Les Composants Clés

Pour exécuter PostgreSQL sur Kubernetes avec succès, plusieurs composants sont nécessaires :

```
┌──────────────────────────────────────────────────────────────────────┐
│               STACK POSTGRESQL SUR KUBERNETES                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                    ┌─────────────────────┐                           │
│                    │     OPERATOR        │  Automatisation           │
│                    │  (CloudNativePG,    │  complète                 │
│                    │   Zalando, Crunchy) │                           │
│                    └──────────┬──────────┘                           │
│                               │                                      │
│                               │ Gère                                 │
│                               ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │                     STATEFULSET                             │     │
│  │                                                             │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │     │
│  │  │ postgres-0  │  │ postgres-1  │  │ postgres-2  │          │     │
│  │  │  (Primary)  │  │  (Replica)  │  │  (Replica)  │          │     │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │     │
│  │         │                │                │                 │     │
│  │  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐          │     │
│  │  │     PVC     │  │     PVC     │  │     PVC     │          │     │
│  │  │   (data)    │  │   (data)    │  │   (data)    │          │     │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │     │
│  │                                                             │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                               │                                      │
│                               │                                      │
│  ┌────────────────────────────┼─────────────────────────────────┐    │
│  │                     SERVICES                                 │    │
│  │                            │                                 │    │
│  │  ┌─────────────────┐  ┌────┴─────────────┐  ┌─────────────┐  │    │
│  │  │ postgres-rw     │  │ postgres-ro      │  │ postgres-   │  │    │
│  │  │ (Primary)       │  │ (Replicas)       │  │ headless    │  │    │
│  │  └─────────────────┘  └──────────────────┘  └─────────────┘  │    │
│  │                                                              │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                    COMPOSANTS ADDITIONNELS                    │   │
│  │                                                               │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐               │   │
│  │  │ ConfigMaps │  │  Secrets   │  │  Backups   │               │   │
│  │  │ (config)   │  │ (passwords)│  │ (S3, GCS)  │               │   │
│  │  └────────────┘  └────────────┘  └────────────┘               │   │
│  │                                                               │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐               │   │
│  │  │ Monitoring │  │ PgBouncer  │  │ NetworkPol │               │   │
│  │  │ (Prometheus│  │ (Pooling)  │  │ (Sécurité) │               │   │
│  │  └────────────┘  └────────────┘  └────────────┘               │   │
│  │                                                               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Vue d'Ensemble des Sous-Chapitres

Ce chapitre sur l'intégration de PostgreSQL avec Kubernetes se décompose en quatre sections détaillées :

### 20bis.4.1. StatefulSets : Persistance et Identité

Nous explorerons la ressource fondamentale pour les applications stateful :

- Différences entre Deployment et StatefulSet
- Identité stable des Pods
- Stockage persistant avec PersistentVolumeClaims
- Ordre de déploiement et de suppression
- Configuration réseau avec Services Headless

### 20bis.4.2. Operators : Zalando, CloudNativePG, Crunchy

Nous comparerons les trois principaux Operators PostgreSQL :

- **Zalando Postgres Operator** : simplicité et maturité  
- **CloudNativePG** : architecture Kubernetes-native  
- **Crunchy PGO** : fonctionnalités enterprise
- Critères de choix et cas d'usage

### 20bis.4.3. Backup Automation (Velero, Native Solutions)

Nous couvrirons les stratégies de sauvegarde :

- Solutions natives PostgreSQL (pgBackRest, Barman, WAL-G)
- Velero pour le backup Kubernetes
- Snapshots de volumes
- Point-In-Time Recovery (PITR)
- Stratégies de rétention et tests de restauration

### 20bis.4.4. Service Mesh et Observabilité

Nous aborderons l'exploitation en production :

- Service Meshes (Istio, Linkerd) avec PostgreSQL
- Monitoring avec Prometheus et Grafana
- Collecte de logs
- Traces distribuées
- Alerting

---

## Prérequis pour ce Chapitre

### Connaissances Requises

Pour tirer le meilleur parti de ce chapitre, vous devriez avoir :

- **Bases PostgreSQL** : Installation, configuration, requêtes SQL  
- **Concepts Docker** : Images, conteneurs, volumes  
- **Notions Kubernetes de base** : Pods, Deployments, Services

### Connaissances Utiles (mais pas obligatoires)

- Expérience avec kubectl
- Familiarité avec YAML
- Notions de réseaux (TCP/IP, DNS)
- Connaissance d'un cloud provider (AWS, GCP, Azure)

### Ce que Vous Allez Apprendre

À la fin de ce chapitre, vous serez capable de :

1. **Comprendre** pourquoi et quand utiliser PostgreSQL sur Kubernetes  
2. **Déployer** un cluster PostgreSQL haute disponibilité  
3. **Choisir** l'Operator adapté à vos besoins  
4. **Configurer** les backups automatisés  
5. **Monitorer** et diagnostiquer les problèmes  
6. **Sécuriser** les communications avec un Service Mesh

---

## Environnement de Travail Recommandé

Pour suivre ce chapitre, vous pouvez utiliser :

### Option 1 : Cluster Local (Développement)

```bash
# Minikube
minikube start --cpus=4 --memory=8192 --disk-size=50g

# Kind (Kubernetes in Docker)
kind create cluster --name postgres-lab

# k3d (k3s in Docker)
k3d cluster create postgres-lab
```

### Option 2 : Cluster Cloud (Production-like)

- **AWS EKS** : Amazon Elastic Kubernetes Service  
- **GKE** : Google Kubernetes Engine  
- **AKS** : Azure Kubernetes Service  
- **DigitalOcean Kubernetes**

### Outils Essentiels

```bash
# kubectl - CLI Kubernetes
# https://kubernetes.io/docs/tasks/tools/

# Helm - Package manager Kubernetes
# https://helm.sh/docs/intro/install/

# k9s - Interface terminal (optionnel mais recommandé)
# https://k9scli.io/
```

---

## Un Mot sur la Complexité

Déployer PostgreSQL sur Kubernetes peut sembler intimidant au premier abord. Il y a beaucoup de concepts à maîtriser : StatefulSets, PersistentVolumes, Operators, Services...

**La bonne nouvelle** : vous n'avez pas besoin de tout construire vous-même. Les Operators PostgreSQL encapsulent des années d'expertise et de bonnes pratiques. En quelques lignes de YAML, vous pouvez obtenir un cluster PostgreSQL :

- Haute disponibilité
- Backups automatisés
- Monitoring intégré
- Failover automatique

```yaml
# Exemple avec CloudNativePG : un cluster HA en 15 lignes
apiVersion: postgresql.cnpg.io/v1  
kind: Cluster  
metadata:  
  name: my-postgres
spec:
  instances: 3
  storage:
    size: 50Gi
  backup:
    barmanObjectStore:
      destinationPath: s3://my-backups
```

Les sections suivantes vous guideront pas à pas, du plus fondamental au plus avancé.

---

## Conclusion de l'Introduction

PostgreSQL sur Kubernetes n'est plus une expérimentation risquée, c'est une pratique établie et éprouvée en production par des organisations du monde entier.

Les avantages sont significatifs :
- **Uniformité** : une plateforme pour tout  
- **Automatisation** : failover, backups, scaling  
- **Portabilité** : même déploiement partout  
- **Infrastructure as Code** : reproductibilité totale

Les défis existent mais sont résolus par l'écosystème :
- **StatefulSets** pour l'identité et la persistance  
- **Operators** pour l'automatisation du cycle de vie  
- **Services** pour le réseau  
- **Outils** pour les backups et le monitoring

Les sections suivantes vous donneront toutes les connaissances nécessaires pour déployer et opérer PostgreSQL sur Kubernetes avec confiance.

---

## Ressources Préliminaires

### Documentation Officielle

- [Kubernetes Documentation](https://kubernetes.io/docs/)  
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

### Tutoriels d'Introduction

- Kubernetes : Concepts fondamentaux
- Kubernetes : Tutoriel interactif officiel
- Docker : Getting Started

### Communautés

- Kubernetes Slack (#postgresql channel)
- PostgreSQL Slack
- Reddit r/kubernetes, r/PostgreSQL

---


⏭️ [StatefulSets : Persistance et identité](/20bis-postgresql-et-architectures-modernes/04.1-statefulsets.md)
