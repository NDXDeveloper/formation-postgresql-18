🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20bis.1 — Microservices et Bases de Données : Introduction

## Préambule

Bienvenue dans cette section consacrée à l'un des défis les plus importants des architectures modernes : **la gestion des données dans un contexte microservices**.

Si vous avez suivi les chapitres précédents, vous maîtrisez désormais PostgreSQL dans un contexte classique : une application, une base de données, des requêtes SQL. Mais le monde du développement logiciel a considérablement évolué ces dernières années, et avec lui, la façon dont nous concevons et déployons nos applications.

Ce chapitre vous prépare à utiliser PostgreSQL dans les architectures distribuées qui dominent aujourd'hui le paysage technologique des grandes entreprises et des startups en croissance.

---

## L'Évolution des Architectures Applicatives

### L'Ère du Monolithe

Pendant des décennies, la majorité des applications ont été construites selon le modèle **monolithique** : une seule application, déployée comme une unité, accédant à une seule base de données.

```
┌─────────────────────────────────────────────────────────────┐
│                     APPLICATION MONOLITHIQUE                │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Module    │  │   Module    │  │   Module    │          │
│  │ Utilisateurs│  │  Commandes  │  │  Catalogue  │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          │                                  │
│                          ▼                                  │
│                 ┌─────────────────┐                         │
│                 │   PostgreSQL    │                         │
│                 │  (Base unique)  │                         │
│                 └─────────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

Cette architecture présente des avantages indéniables :

| Avantage | Description |
|----------|-------------|
| **Simplicité** | Un seul code source, un seul déploiement |
| **Transactions ACID** | Cohérence garantie par PostgreSQL |
| **Développement rapide** | Pas de coordination entre équipes |
| **Débogage facile** | Tout est au même endroit |

### Les Limites du Monolithe

Cependant, à mesure que les applications grandissent, des problèmes émergent :

| Problème | Impact |
|----------|--------|
| **Couplage fort** | Modifier un module peut casser les autres |
| **Déploiements risqués** | Une petite modification nécessite de redéployer tout |
| **Scalabilité limitée** | Impossible de scaler un seul composant |
| **Équipes bloquées** | Tout le monde travaille sur le même code |
| **Dette technique** | Le code devient un "plat de spaghetti" |
| **Technologie figée** | Difficile d'adopter de nouveaux outils |

Quand une startup de 5 développeurs devient une entreprise de 50 ou 500 ingénieurs, le monolithe devient un frein à l'innovation et à la vélocité.

### L'Émergence des Microservices

Pour répondre à ces défis, l'industrie s'est tournée vers les **architectures microservices** : découper l'application en services indépendants, chacun responsable d'une fonctionnalité métier spécifique.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Service    │     │  Service    │     │  Service    │
│ Utilisateurs│     │  Commandes  │     │  Catalogue  │
│             │     │             │     │             │
│ - API REST  │     │ - API REST  │     │ - API REST  │
│ - Logique   │     │ - Logique   │     │ - Logique   │
│ - Données   │     │ - Données   │     │ - Données   │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       │      Communication via API / Events   │
       └───────────────────┼───────────────────┘
                           │
                   ┌───────┴──────┐
                   │   Clients    │
                   │ (Web, Mobile)│
                   └──────────────┘
```

---

## Qu'est-ce qu'un Microservice ?

### Définition

Un **microservice** est un service logiciel autonome qui :

- Implémente une **fonctionnalité métier** spécifique et cohérente
- Peut être **développé, déployé et scalé indépendamment**
- Communique avec les autres services via des **interfaces bien définies** (APIs, événements)
- Est **géré par une équipe réduite** (souvent selon la "règle des deux pizzas" : une équipe qui peut être nourrie avec deux pizzas)

### Caractéristiques Clés

| Caractéristique | Description |
|-----------------|-------------|
| **Autonomie** | Le service fonctionne de manière indépendante |
| **Responsabilité unique** | Un service = un domaine métier |
| **Décentralisation** | Chaque équipe prend ses propres décisions techniques |
| **Résilience** | La panne d'un service n'arrête pas les autres |
| **Déployabilité** | Mises en production fréquentes et indépendantes |
| **Observabilité** | Chaque service expose ses métriques et logs |

### Exemple Concret : Application E-Commerce

Une application e-commerce monolithique pourrait être découpée ainsi :

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ARCHITECTURE MICROSERVICES                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐  │
│  │  Users    │ │  Catalog  │ │   Cart    │ │  Orders   │ │ Payments  │  │
│  │  Service  │ │  Service  │ │  Service  │ │  Service  │ │  Service  │  │
│  │           │ │           │ │           │ │           │ │           │  │
│  │ • Comptes │ │ • Produits│ │ • Panier  │ │ • Achat   │ │ • Paiement│  │
│  │ • Profils │ │ • Stock   │ │ • Sessions│ │ • Suivi   │ │ • Refund  │  │
│  │ • Auth    │ │ • Catégor.│ │ • Promos  │ │ • Factures│ │ • Fraude  │  │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘ └───────────┘  │
│                                                                         │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐                │
│  │ Shipping  │ │  Notifs   │ │  Search   │ │  Reviews  │                │
│  │  Service  │ │  Service  │ │  Service  │ │  Service  │                │
│  │           │ │           │ │           │ │           │                │
│  │ • Livr.   │ │ • Emails  │ │ • Elastic │ │ • Notes   │                │
│  │ • Track.  │ │ • SMS     │ │ • Suggest.│ │ • Comments│                │
│  │ • Returns │ │ • Push    │ │ • Filters │ │ • Modérat.│                │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

Chaque service est développé par une équipe dédiée, déployé indépendamment, et peut utiliser les technologies les plus adaptées à son cas d'usage.

---

## Le Défi Central : La Gestion des Données

### Pourquoi les Données Sont-elles si Complexes ?

Dans un monolithe, la gestion des données est relativement simple : une base PostgreSQL, des transactions ACID, des jointures SQL. Tout fonctionne naturellement.

Avec les microservices, **tout se complique** :

```
                    MONOLITHE                          MICROSERVICES

              ┌─────────────────┐            ┌─────────┐ ┌─────────┐ ┌─────────┐
              │   Application   │            │Service A│ │Service B│ │Service C│
              └────────┬────────┘            └────┬────┘ └────┬────┘ └────┬────┘
                       │                          │           │           │
                       ▼                          ▼           ▼           ▼
              ┌─────────────────┐            ┌───────┐   ┌───────┐   ┌───────┐
              │   PostgreSQL    │            │ DB A  │   │ DB B  │   │ DB C  │
              │   (1 base)      │            └───────┘   └───────┘   └───────┘
              └─────────────────┘
                                             Comment faire une jointure
               Jointures faciles !           entre DB A et DB B ? 🤔
               Transactions ACID !
                                             Comment garantir la cohérence
                                             entre DB A, B et C ? 🤔
```

### Les Questions Fondamentales

L'adoption des microservices soulève des questions cruciales concernant les données :

#### 1. Organisation des Bases de Données

> *"Chaque service doit-il avoir sa propre base de données, ou peuvent-ils partager une base commune ?"*

Cette décision fondamentale impacte l'autonomie des équipes, la cohérence des données, et la complexité opérationnelle.

#### 2. Cohérence Transactionnelle

> *"Comment garantir qu'une opération impliquant plusieurs services soit atomique ?"*

Quand une commande e-commerce doit simultanément :
- Créer une commande (Service Orders)
- Réserver le stock (Service Catalog)
- Débiter le client (Service Payments)
- Planifier la livraison (Service Shipping)

...comment s'assurer que soit tout réussit, soit tout est annulé ?

#### 3. Requêtes Transversales

> *"Comment interroger des données réparties sur plusieurs services ?"*

Un tableau de bord qui affiche "le chiffre d'affaires par catégorie de produit et par pays du client" nécessite des données de trois services différents. Comment les rassembler efficacement ?

#### 4. Synchronisation des Données

> *"Comment garder les données cohérentes entre services qui dupliquent certaines informations ?"*

Si le service Orders stocke une copie du nom du produit (pour éviter d'appeler le service Catalog à chaque affichage), comment mettre à jour cette copie quand le produit est renommé ?

---

## Les Grandes Stratégies

Face à ces défis, plusieurs stratégies et patterns ont émergé. Ce chapitre est structuré autour des trois approches fondamentales :

### 20bis.1.1 — Database per Service vs Shared Database

La première décision architecturale concerne l'organisation physique des bases de données :

| Approche | Principe |
|----------|----------|
| **Shared Database** | Tous les services partagent une même base PostgreSQL |
| **Database per Service** | Chaque service possède sa propre base de données |

Nous explorerons les avantages, inconvénients et cas d'usage de chaque approche.

### 20bis.1.2 — Distributed Transactions et Saga Pattern

Quand chaque service a sa propre base, les transactions ACID classiques ne fonctionnent plus. Nous étudierons :

| Pattern | Description |
|---------|-------------|
| **Two-Phase Commit** | Protocole classique de transaction distribuée |
| **Saga Pattern** | Séquence de transactions locales avec compensations |
| **Orchestration vs Chorégraphie** | Deux façons d'implémenter les Sagas |

### 20bis.1.3 — Foreign Data Wrappers pour la Fédération

Pour les requêtes transversales et le reporting, PostgreSQL offre une solution élégante :

| Fonctionnalité | Utilité |
|----------------|---------|
| **Foreign Data Wrappers** | Accéder à des données distantes comme des tables locales |
| **postgres_fdw** | Fédérer plusieurs bases PostgreSQL |
| **Vues matérialisées** | Cacher les résultats des requêtes fédérées |

---

## Prérequis pour ce Chapitre

Avant d'aborder ces sujets, assurez-vous de maîtriser :

| Concept | Chapitres de référence |
|---------|----------------------|
| Transactions et ACID | Chapitre 1.4, Chapitre 12 |
| Contraintes d'intégrité | Chapitre 7.1 |
| Jointures SQL | Chapitre 7 |
| Schémas PostgreSQL | Chapitre 4.2 |
| Rôles et permissions | Chapitre 16.4, 16.5 |
| Vues et vues matérialisées | Chapitre 11.5 |

Si certains de ces concepts vous semblent flous, nous vous encourageons à revisiter les chapitres correspondants avant de continuer.

---

## Vocabulaire Essentiel

Avant de plonger dans les détails, familiarisons-nous avec le vocabulaire clé :

| Terme | Définition |
|-------|------------|
| **Microservice** | Service logiciel autonome implémentant une fonctionnalité métier |
| **Bounded Context** | Périmètre fonctionnel et technique d'un service (concept DDD) |
| **API Gateway** | Point d'entrée unique qui route les requêtes vers les services |
| **Service Mesh** | Infrastructure de communication entre services |
| **Event-Driven** | Architecture basée sur l'échange d'événements asynchrones |
| **Eventual Consistency** | Cohérence garantie à terme, pas immédiatement |
| **CQRS** | Séparation des modèles de lecture et d'écriture |
| **Saga** | Transaction distribuée composée de transactions locales |
| **Outbox Pattern** | Technique pour publier des événements de manière fiable |
| **FDW** | Foreign Data Wrapper — accès à des données externes |

---

## Ce que Vous Allez Apprendre

À la fin de ce chapitre, vous serez capable de :

✅ **Choisir** l'architecture de données adaptée à votre contexte (shared vs per-service)

✅ **Implémenter** des transactions distribuées avec le pattern Saga

✅ **Configurer** des Foreign Data Wrappers pour fédérer plusieurs bases PostgreSQL

✅ **Concevoir** une base de reporting qui agrège des données de multiples services

✅ **Comprendre** les compromis entre cohérence, disponibilité et performance

✅ **Éviter** les pièges classiques des architectures microservices

---

## Avertissement : La Complexité a un Coût

Avant de vous lancer dans les microservices, un avertissement important s'impose.

### Les Microservices Ne Sont Pas Toujours la Solution

L'architecture microservices résout certains problèmes mais en crée d'autres :

| Avantage gagné | Complexité ajoutée |
|----------------|-------------------|
| Scalabilité indépendante | Infrastructure plus complexe |
| Autonomie des équipes | Coordination distribuée |
| Déploiements isolés | Observabilité distribuée |
| Flexibilité technologique | Cohérence des données difficile |
| Résilience aux pannes | Debugging plus complexe |

### Quand Rester sur un Monolithe

Un monolithe bien conçu (parfois appelé "monolithe modulaire") reste souvent le meilleur choix pour :

- Les équipes de moins de 10-15 développeurs
- Les startups en phase de validation produit
- Les applications avec des exigences de cohérence forte
- Les projets avec des contraintes budgétaires

> *"Si vous ne pouvez pas construire un monolithe bien structuré, qu'est-ce qui vous fait croire que vous pouvez construire un ensemble de microservices bien structurés ?"*
> — Simon Brown

### L'Approche Progressive

La sagesse recommande souvent de :

1. **Commencer** par un monolithe modulaire bien structuré
2. **Identifier** les domaines qui bénéficieraient d'une extraction
3. **Extraire** progressivement les services quand le besoin se fait sentir
4. **Éviter** de sur-architecturer dès le départ

---

## Structure des Sous-Chapitres

Ce chapitre est divisé en trois sous-sections complémentaires :

```
20bis.1 — Microservices et Bases de Données (cette introduction)
    │
    ├── 20bis.1.1 — Database per Service vs Shared Database
    │       │
    │       ├── Avantages et inconvénients de chaque approche
    │       ├── Variante : Schema per Service
    │       ├── Guide de décision
    │       └── Bonnes pratiques PostgreSQL
    │
    ├── 20bis.1.2 — Distributed Transactions et Saga Pattern
    │       │
    │       ├── Pourquoi les transactions ACID ne suffisent plus
    │       ├── Two-Phase Commit et ses limites
    │       ├── Le pattern Saga (Orchestration vs Chorégraphie)
    │       ├── Pattern Outbox pour la fiabilité
    │       └── Gestion des erreurs et idempotence
    │
    └── 20bis.1.3 — Foreign Data Wrappers pour la Fédération
            │
            ├── Architecture des FDW
            ├── postgres_fdw en détail
            ├── Optimisation et pushdown
            ├── Autres FDW (file, mysql, oracle, mongo)
            └── Patterns d'architecture (reporting, migration)
```

Chaque sous-section est conçue pour être lue indépendamment, mais nous recommandons de les parcourir dans l'ordre pour une compréhension optimale.

---

## Conclusion de l'Introduction

La gestion des données dans une architecture microservices est un sujet vaste et nuancé. Il n'existe pas de solution universelle : chaque décision implique des compromis entre simplicité, cohérence, performance et autonomie.

PostgreSQL, grâce à sa robustesse, sa flexibilité et ses fonctionnalités avancées (schémas, FDW, réplication logique, JSONB), est un excellent choix pour accompagner cette transition. Que vous optiez pour une base partagée, des bases distribuées, ou une approche hybride, PostgreSQL s'adaptera à vos besoins.

Dans les sections suivantes, nous explorerons chaque stratégie en détail, avec des exemples concrets et des recommandations pratiques pour tirer le meilleur parti de PostgreSQL dans vos architectures modernes.

---

## Points Clés à Retenir

- **Microservices** : Services autonomes, chacun responsable d'un domaine métier
- **Défi principal** : La gestion des données distribuées (cohérence, transactions, requêtes)
- **Trois stratégies clés** : Organisation des bases, transactions distribuées, fédération
- **Complexité** : Les microservices ne sont pas toujours la bonne solution
- **PostgreSQL** : Parfaitement adapté aux architectures modernes grâce à sa flexibilité

---


⏭️ [Database per service vs Shared database](/20bis-postgresql-et-architectures-modernes/01.1-database-per-service.md)
