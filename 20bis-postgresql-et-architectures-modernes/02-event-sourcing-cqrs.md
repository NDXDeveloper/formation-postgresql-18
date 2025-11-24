🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20bis.2 — Event Sourcing et CQRS avec PostgreSQL : Introduction

## Préambule

Dans le chapitre précédent, nous avons exploré comment organiser les bases de données dans une architecture microservices. Nous avons vu les défis de la cohérence transactionnelle et les solutions comme le pattern Saga.

Ce chapitre aborde une approche encore plus fondamentale : **repenser la façon dont nous stockons et manipulons les données**. Au lieu de stocker l'état actuel et de le modifier destructivement, nous allons découvrir comment stocker l'**historique complet des changements** et en dériver l'état.

Cette approche, appelée **Event Sourcing**, combinée au pattern **CQRS** (Command Query Responsibility Segregation), ouvre la porte à des architectures plus robustes, auditables et évolutives.

PostgreSQL, grâce à sa flexibilité (JSONB, LISTEN/NOTIFY, Logical Decoding), est une excellente plateforme pour implémenter ces patterns.

---

## Le Problème : La Perte d'Information

### L'Approche Traditionnelle CRUD

Dans une application classique, nous utilisons le pattern **CRUD** (Create, Read, Update, Delete). Les données sont stockées dans leur état actuel, et chaque modification écrase l'état précédent.

```sql
-- État initial
INSERT INTO accounts (id, owner, balance) VALUES (1, 'Alice', 1000);

-- Après un retrait
UPDATE accounts SET balance = 800 WHERE id = 1;

-- Après un dépôt
UPDATE accounts SET balance = 1300 WHERE id = 1;

-- État final visible : balance = 1300
-- Mais... pourquoi 1300 ? Quand ? Par qui ? On ne sait pas.
```

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Approche CRUD Traditionnelle                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│    État t0          État t1          État t2          État t3          │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐       │
│   │ balance │      │ balance │      │ balance │      │ balance │       │
│   │  1000   │ ───► │   800   │ ───► │  1300   │ ───► │  1100   │       │
│   └─────────┘      └─────────┘      └─────────┘      └─────────┘       │
│        │                │                │                │            │
│        │           ┌────┴────┐      ┌────┴────┐      ┌────┴────┐       │
│        │           │ UPDATE  │      │ UPDATE  │      │ UPDATE  │       │
│        │           │ (écrase)│      │ (écrase)│      │ (écrase)│       │
│        │           └─────────┘      └─────────┘      └─────────┘       │
│        │                                                               │
│        └──────────────────────────────────────────────────────────────►│
│                                                                        │
│                    ⚠️  L'HISTORIQUE EST PERDU                          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Les Limites du CRUD

| Problème | Description |
|----------|-------------|
| **Perte d'historique** | Impossible de savoir comment on est arrivé à l'état actuel |
| **Pas d'audit natif** | Nécessite des tables d'audit séparées, souvent incomplètes |
| **Debugging difficile** | "Pourquoi ce solde est-il négatif ?" - Impossible à retracer |
| **Pas de "time travel"** | Impossible de voir l'état du système à un instant T passé |
| **Intégration complexe** | Comment notifier d'autres systèmes des changements ? |
| **Conflits de concurrence** | Les mises à jour simultanées peuvent perdre des informations |

---

## La Solution : Event Sourcing

### Un Changement de Paradigme

L'**Event Sourcing** propose une approche radicalement différente : au lieu de stocker l'état, nous stockons **les événements** qui ont conduit à cet état.

```sql
-- Au lieu de UPDATE, nous INSÉRONS des événements immuables
INSERT INTO events (stream_id, event_type, data, occurred_at) VALUES
('account-1', 'AccountOpened',   '{"owner": "Alice", "initial_balance": 1000}', '2025-01-01 10:00'),
('account-1', 'MoneyWithdrawn',  '{"amount": 200, "reason": "ATM"}',            '2025-01-01 14:30'),
('account-1', 'MoneyDeposited',  '{"amount": 500, "source": "salary"}',         '2025-01-15 09:00'),
('account-1', 'MoneyWithdrawn',  '{"amount": 200, "reason": "transfer"}',       '2025-01-20 16:45');

-- L'état actuel (balance = 1100) se CALCULE en rejouant les événements
-- 1000 - 200 + 500 - 200 = 1100
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Event Sourcing                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│    Event 1           Event 2           Event 3           Event 4        │
│   ┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐   │
│   │ Account   │     │ Money     │     │ Money     │     │ Money     │   │
│   │ Opened    │ ──► │ Withdrawn │ ──► │ Deposited │ ──► │ Withdrawn │   │
│   │ +1000     │     │ -200      │     │ +500      │     │ -200      │   │
│   │ 01/01     │     │ 01/01     │     │ 15/01     │     │ 20/01     │   │
│   └───────────┘     └───────────┘     └───────────┘     └───────────┘   │
│        │                 │                 │                 │          │
│        ▼                 ▼                 ▼                 ▼          │
│     balance           balance           balance           balance       │
│      =1000             =800             =1300             =1100         │
│                                                                         │
│   ✅ HISTORIQUE COMPLET PRÉSERVÉ                                        │
│   ✅ Chaque événement : quoi, quand, pourquoi, par qui                  │
│   ✅ Immutable : rien n'est jamais modifié ou supprimé                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Définition Formelle

> **Event Sourcing** : Pattern architectural où l'état d'une application est déterminé par une séquence d'événements. Chaque changement d'état est capturé comme un événement immuable dans un journal append-only.

### Les Concepts Clés

| Concept | Description |
|---------|-------------|
| **Event** | Fait qui s'est produit, décrit au passé (ex: `OrderPlaced`, `PaymentReceived`) |
| **Event Store** | Base de données optimisée pour stocker les événements |
| **Stream** | Séquence ordonnée d'événements pour une entité (ex: tous les événements d'une commande) |
| **Projection** | Vue dérivée calculée en rejouant les événements |
| **Snapshot** | Cache de l'état à un instant T pour optimiser la reconstruction |
| **Rehydration** | Processus de reconstruction de l'état depuis les événements |

---

## CQRS : Séparer Lecture et Écriture

### Le Problème des Modèles Unifiés

Dans une application CRUD classique, le même modèle de données sert à la fois pour :
- Les **écritures** (INSERT, UPDATE, DELETE)
- Les **lectures** (SELECT, rapports, recherche)

Cela crée des compromis : un schéma optimisé pour les écritures est rarement optimal pour les lectures complexes, et vice versa.

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Modèle CRUD Unifié                                 │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│     Écritures (OLTP)                    Lectures (OLAP)                │
│   ┌───────────────────┐              ┌───────────────────┐             │
│   │ INSERT très simple│              │ SELECT complexe   │             │
│   │ UPDATE rapide     │              │ avec 10 JOINs,    │             │
│   │ Normalisé (3NF)   │              │ agrégations,      │             │
│   │                   │              │ full-text search  │             │
│   └─────────┬─────────┘              └─────────┬─────────┘             │
│             │                                  │                       │
│             └──────────────┬───────────────────┘                       │
│                            │                                           │
│                            ▼                                           │
│                   ┌─────────────────┐                                  │
│                   │  Même table,    │                                  │
│                   │  même schéma,   │                                  │
│                   │  même index     │                                  │
│                   │                 │                                  │
│                   │   COMPROMIS !   │                                  │
│                   └─────────────────┘                                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### La Solution CQRS

**CQRS** (Command Query Responsibility Segregation) sépare les modèles de lecture et d'écriture :

- **Commands** : Opérations qui modifient l'état (write model)
- **Queries** : Opérations qui lisent l'état (read model)

```
┌────────────────────────────────────────────────────────────────────────┐
│                              CQRS                                      │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│                         Application                                    │
│                             │                                          │
│              ┌──────────────┴──────────────┐                           │
│              │                             │                           │
│              ▼                             ▼                           │
│     ┌─────────────────┐           ┌─────────────────┐                  │
│     │    Commands     │           │     Queries     │                  │
│     │                 │           │                 │                  │
│     │ • PlaceOrder    │           │ • GetOrderById  │                  │
│     │ • CancelOrder   │           │ • SearchOrders  │                  │
│     │ • UpdateAddress │           │ • GetDashboard  │                  │
│     └────────┬────────┘           └────────┬────────┘                  │
│              │                             │                           │
│              ▼                             ▼                           │
│     ┌─────────────────┐           ┌─────────────────┐                  │
│     │   Write Model   │           │   Read Model    │                  │
│     │                 │           │                 │                  │
│     │ • Normalisé     │  ──────►  │ • Dénormalisé   │                  │
│     │ • Event Store   │  Sync     │ • Optimisé      │                  │
│     │ • Transactionnel│           │ • Pré-calculé   │                  │
│     └─────────────────┘           └─────────────────┘                  │
│                                                                        │
│     Optimisé pour               Optimisé pour les                      │
│     l'intégrité                 performances de lecture                │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Définition Formelle

> **CQRS** : Pattern architectural qui sépare les opérations de lecture (Queries) des opérations d'écriture (Commands), permettant d'optimiser indépendamment chaque côté.

---

## Event Sourcing + CQRS : Le Duo Puissant

### Synergie Naturelle

Event Sourcing et CQRS se combinent naturellement :

- L'**Event Store** stocke les événements (write model)
- Les **Projections** dérivent des vues optimisées (read models)
- Les événements synchronisent les deux modèles

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Event Sourcing + CQRS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                            Application                                  │
│                                │                                        │
│               ┌────────────────┴────────────────┐                       │
│               │                                 │                       │
│               ▼                                 ▼                       │
│      ┌─────────────────┐               ┌─────────────────┐              │
│      │    Commands     │               │     Queries     │              │
│      │                 │               │                 │              │
│      │ PlaceOrder()    │               │ GetOrder()      │              │
│      │ CancelOrder()   │               │ SearchOrders()  │              │
│      └────────┬────────┘               └────────┬────────┘              │
│               │                                 │                       │
│               ▼                                 ▼                       │
│      ┌─────────────────┐               ┌─────────────────┐              │
│      │   Event Store   │               │   Read Models   │              │
│      │   (Write)       │               │   (Projections) │              │
│      │                 │               │                 │              │
│      │ ┌─────────────┐ │               │ ┌─────────────┐ │              │
│      │ │OrderCreated │ │    Events     │ │orders_view  │ │              │
│      │ │ItemAdded    │ │ ───────────►  │ │(dénormalisé)│ │              │
│      │ │OrderShipped │ │    Sync       │ ├─────────────┤ │              │
│      │ │...          │ │               │ │search_index │ │              │
│      │ └─────────────┘ │               │ │(Elasticsearch)│              │
│      └─────────────────┘               │ ├─────────────┤ │              │
│                                        │ │dashboard_   │ │              │
│      Source de Vérité                  │ │cache (Redis)│ │              │
│      (append-only)                     │ └─────────────┘ │              │
│                                        └─────────────────┘              │
│                                                                         │
│                                        Vues optimisées                  │
│                                        pour chaque usage                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Avantages de la Combinaison

| Avantage | Description |
|----------|-------------|
| **Historique complet** | Chaque action est enregistrée comme événement |
| **Audit natif** | L'Event Store EST le journal d'audit |
| **Flexibilité des lectures** | Créer N projections pour N cas d'usage |
| **Scalabilité** | Read models peuvent être dupliqués et distribués |
| **Évolutivité** | Ajouter de nouvelles projections sans toucher aux données sources |
| **Debugging** | Rejouer les événements pour comprendre un bug |
| **Time Travel** | Reconstruire l'état à n'importe quel instant passé |

---

## Quand Utiliser Event Sourcing et CQRS ?

### Cas d'Usage Idéaux

| Domaine | Pourquoi Event Sourcing ? |
|---------|---------------------------|
| **Finance & Banque** | Audit obligatoire, traçabilité des transactions |
| **E-commerce** | Historique des commandes, paniers, comportements |
| **Logistique** | Tracking complet des colis, états successifs |
| **Santé** | Dossiers médicaux, historique des traitements |
| **Systèmes collaboratifs** | Historique des modifications (comme Git) |
| **Gaming** | Replay, détection de triche, statistiques |
| **IoT** | Séries temporelles d'événements capteurs |

### Signaux que Vous en Avez Besoin

Vous devriez considérer Event Sourcing si :

- ✅ L'audit et la traçabilité sont des exigences métier
- ✅ Vous avez besoin de "défaire" ou "rejouer" des actions
- ✅ Le domaine métier est naturellement événementiel
- ✅ Vous avez des besoins de reporting complexes sur l'historique
- ✅ Plusieurs systèmes doivent réagir aux mêmes changements
- ✅ La conformité réglementaire exige un historique immuable

### Quand Éviter

Event Sourcing n'est pas toujours approprié :

- ❌ Applications CRUD simples sans besoin d'historique
- ❌ Équipes sans expérience du pattern (courbe d'apprentissage)
- ❌ Domaines où les données peuvent/doivent être supprimées (RGPD complexe)
- ❌ Systèmes avec très faible complexité métier
- ❌ Contraintes de time-to-market très serrées

---

## PostgreSQL : Une Excellente Plateforme

### Pourquoi PostgreSQL pour Event Sourcing ?

PostgreSQL offre toutes les fonctionnalités nécessaires :

| Fonctionnalité | Utilité pour Event Sourcing |
|----------------|----------------------------|
| **JSONB** | Stocker les payloads d'événements de façon flexible |
| **Séquences** | Garantir l'ordre des événements |
| **Transactions ACID** | Cohérence de l'Event Store |
| **LISTEN/NOTIFY** | Notifications temps réel des nouveaux événements |
| **Logical Decoding** | CDC pour alimenter les projections |
| **Partitionnement** | Gérer de gros volumes d'événements |
| **Index GIN** | Recherche efficace dans les événements JSON |

### Comparaison avec des Event Stores Dédiés

| Critère | PostgreSQL | EventStoreDB | Kafka |
|---------|------------|--------------|-------|
| **Complexité** | Vous le connaissez déjà | Nouveau système | Nouveau système |
| **Transactions** | ACID complet | ACID par stream | At-least-once |
| **Requêtes SQL** | Oui | Limité | Non |
| **Projections** | À implémenter | Intégrées | Via Kafka Streams |
| **Écosystème** | Immense | Spécialisé | Immense |
| **Opérations** | Bien maîtrisées | Spécifiques | Complexes |

Pour de nombreuses équipes, PostgreSQL représente le meilleur compromis : fonctionnalités suffisantes sans ajouter de nouvelle technologie à maîtriser.

---

## Ce que Vous Allez Apprendre

Ce chapitre est structuré en quatre sections progressives :

### 20bis.2.1 — Event Store Pattern

Comment construire un Event Store robuste avec PostgreSQL :

- Structure de la table des événements
- Gestion des streams et du versioning
- Contrôle de concurrence optimiste
- Snapshots pour les performances
- Reconstruction de l'état (rehydration)

### 20bis.2.2 — NOTIFY/LISTEN pour Événements Temps Réel

Utiliser le mécanisme pub/sub natif de PostgreSQL :

- Notifications instantanées des nouveaux événements
- Intégration avec les triggers
- Architecture WebSocket pour les clients
- Limites et cas d'usage appropriés

### 20bis.2.3 — Logical Decoding et Change Data Capture

Capturer les changements au niveau du WAL :

- Configuration du Logical Decoding
- Plugins (pgoutput, wal2json)
- Slots de réplication
- Alimenter des systèmes externes

### 20bis.2.4 — Debezium pour Streaming d'Événements

La plateforme CDC de référence pour la production :

- Architecture Debezium + Kafka
- Configuration avancée
- Transformations de messages
- Monitoring et opérations

---

## Architecture de Référence

Voici l'architecture que nous allons construire au fil des sections :

```
┌─────────────────────────────────────────────────────────────────────────┐
│                Architecture Event Sourcing Complète                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────┐                                                   │
│   │   Application   │                                                   │
│   │   (Commands)    │                                                   │
│   └────────┬────────┘                                                   │
│            │                                                            │
│            ▼                                                            │
│   ┌─────────────────┐      ┌─────────────────┐                          │
│   │   Event Store   │      │  NOTIFY/LISTEN  │   Section 20bis.2.1      │
│   │   (PostgreSQL)  │─────►│  (temps réel)   │   + 20bis.2.2            │
│   └────────┬────────┘      └────────┬────────┘                          │
│            │                        │                                   │
│            │                        ▼                                   │
│            │               ┌─────────────────┐                          │
│            │               │   WebSockets    │                          │
│            │               │   (Dashboards)  │                          │
│            │               └─────────────────┘                          │
│            │                                                            │
│            ▼                                                            │
│   ┌─────────────────┐                                                   │
│   │ Logical Decoding│      Section 20bis.2.3                            │
│   │      (WAL)      │                                                   │
│   └────────┬────────┘                                                   │
│            │                                                            │
│            ▼                                                            │
│   ┌─────────────────┐      ┌─────────────────┐                          │
│   │    Debezium     │─────►│     Kafka       │   Section 20bis.2.4      │
│   │   (CDC)         │      │    Topics       │                          │
│   └─────────────────┘      └────────┬────────┘                          │
│                                     │                                   │
│            ┌────────────────────────┼────────────────────────┐          │
│            │                        │                        │          │
│            ▼                        ▼                        ▼          │
│   ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐ │
│   │  Projection 1   │      │  Projection 2   │      │  Projection 3   │ │
│   │  (PostgreSQL    │      │  (Elasticsearch │      │  (Redis         │ │
│   │   Read Model)   │      │   Search Index) │      │   Cache)        │ │
│   └─────────────────┘      └─────────────────┘      └─────────────────┘ │
│            │                        │                        │          │
│            └────────────────────────┼────────────────────────┘          │
│                                     │                                   │
│                                     ▼                                   │
│                            ┌─────────────────┐                          │
│                            │   Application   │                          │
│                            │   (Queries)     │                          │
│                            └─────────────────┘                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Prérequis pour ce Chapitre

Avant d'aborder ces sujets, assurez-vous de maîtriser :

| Concept | Chapitres de référence |
|---------|----------------------|
| Transactions et ACID | Chapitre 1.4, Chapitre 12 |
| JSONB et opérateurs JSON | Chapitre 4.4.4 |
| Triggers et fonctions PL/pgSQL | Chapitre 15 |
| Index GIN | Chapitre 13.4.1 |
| Séquences | Chapitre 4.5 |
| Configuration WAL | Chapitre 16.13.3 |

---

## Vocabulaire Essentiel

| Terme | Définition |
|-------|------------|
| **Event** | Fait immutable qui s'est produit (toujours au passé) |
| **Event Store** | Base de données spécialisée pour stocker les événements |
| **Stream** | Séquence d'événements pour une entité donnée |
| **Aggregate** | Entité métier dont l'état est défini par ses événements |
| **Projection** | Vue matérialisée dérivée des événements |
| **Snapshot** | Cache de l'état à un instant T pour optimiser |
| **Command** | Intention de modifier l'état (peut être rejetée) |
| **Query** | Demande de lecture de l'état |
| **Rehydration** | Reconstruction de l'état en rejouant les événements |
| **Eventual Consistency** | Les projections sont cohérentes "à terme" |
| **CDC** | Change Data Capture — capture des modifications |
| **Idempotence** | Une opération peut être rejouée sans effet supplémentaire |

---

## Un Avertissement Important

### La Complexité a un Coût

Event Sourcing et CQRS ajoutent de la complexité architecturale :

| Défi | Description |
|------|-------------|
| **Courbe d'apprentissage** | Nouveau paradigme pour l'équipe |
| **Eventual consistency** | Les projections ne sont pas immédiatement à jour |
| **Versioning des événements** | Gérer les changements de format |
| **Volume de données** | Les événements s'accumulent indéfiniment |
| **RGPD et droit à l'oubli** | Complexe avec des données immutables |
| **Debugging différent** | Nouvelle façon de diagnostiquer les problèmes |

### Commencer Petit

Notre recommandation :

1. **Ne pas tout migrer d'un coup** — Commencez par un domaine limité
2. **Choisir un cas d'usage clair** — Où l'audit est déjà une exigence
3. **Former l'équipe** — Investir dans la compréhension des patterns
4. **Itérer** — Affiner l'implémentation au fil des retours

---

## Conclusion de l'Introduction

L'**Event Sourcing** et le **CQRS** représentent un changement de paradigme dans la façon de concevoir les systèmes de données. Au lieu de stocker et modifier l'état, nous capturons l'histoire complète sous forme d'événements immutables.

Cette approche offre des avantages uniques : audit natif, debugging facilité, flexibilité des lectures, et intégration naturelle avec les architectures événementielles.

PostgreSQL, avec son écosystème riche (JSONB, LISTEN/NOTIFY, Logical Decoding), permet d'implémenter ces patterns sans ajouter de nouvelles technologies à votre stack.

Dans les sections suivantes, nous allons construire pas à pas une architecture Event Sourcing complète, de l'Event Store aux projections en temps réel.

---

## Points Clés à Retenir

- **CRUD** : Stocke l'état actuel, perd l'historique
- **Event Sourcing** : Stocke les événements, dérive l'état
- **CQRS** : Sépare les modèles de lecture et d'écriture
- **Event Store** : Base append-only pour les événements
- **Projection** : Vue dérivée optimisée pour un cas d'usage
- **PostgreSQL** : Excellente plateforme grâce à JSONB, NOTIFY, Logical Decoding
- **Complexité** : À utiliser quand les bénéfices justifient le coût

---


⏭️ [Event Store pattern](/20bis-postgresql-et-architectures-modernes/02.1-event-store-pattern.md)
