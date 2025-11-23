🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.4. Principes de Conception d'APIs avec PostgreSQL

## Introduction

La **conception d'API** (Application Programming Interface) avec PostgreSQL consiste à créer une couche logicielle qui permet à vos applications d'interagir de manière structurée, sécurisée et efficace avec votre base de données. Une API bien conçue est la clé d'une application robuste, maintenable et évolutive.

### Métaphore : L'API comme interface de communication

Imaginez PostgreSQL comme une **bibliothèque géante** contenant toutes vos données :

**Sans API (accès direct) :**
```plaintext
Développeur Frontend : "Je veux les utilisateurs"
  → Écrit directement : SELECT * FROM users WHERE active = true;
  → Accès direct à PostgreSQL
  → Risques : injection SQL, permissions incorrectes, requêtes inefficaces

Développeur Mobile : "Je veux les utilisateurs aussi"
  → Écrit : SELECT id, name, email FROM users;
  → Code SQL dupliqué
  → Maintenance cauchemardesque

→ Chaos : SQL dispersé partout, pas de cohérence
```

**Avec API (accès structuré) :**
```plaintext
Développeur Frontend : "Je veux les utilisateurs"
  → Appelle : GET /api/users?active=true
  → API traduit en SQL sécurisé
  → Retourne JSON structuré

Développeur Mobile : "Je veux les utilisateurs aussi"
  → Appelle : GET /api/users?active=true
  → Même endpoint, même logique
  → Cohérence garantie

→ Organisation : API = point d'entrée unique et contrôlé
```

L'API agit comme un **bibliothécaire professionnel** : vous lui demandez ce que vous voulez (en langage simple), il va chercher dans les rayons (PostgreSQL), applique les règles (sécurité, permissions), et vous retourne exactement ce dont vous avez besoin.

## Qu'est-ce qu'une API ?

### Définition simple

Une **API (Application Programming Interface)** est un ensemble de règles et de mécanismes qui permettent à différentes applications de communiquer entre elles. Dans notre contexte, c'est la couche qui se situe entre vos applications clientes (web, mobile, etc.) et votre base de données PostgreSQL.

### Architecture typique

```plaintext
┌─────────────────────────────────────────────────────┐
│                Applications Clientes                │
│  ┌─────────────┐   ┌─────────────┐  ┌─────────────┐ │
│  │   Web App   │   │  Mobile App │  │  Desktop    │ │
│  │  (React)    │   │  (Flutter)  │  │  (Electron) │ │
│  └──────┬──────┘   └──────┬──────┘  └──────┬──────┘ │
└─────────┼─────────────────┼────────────────┼────────┘
          │                 │                │
          │    HTTP/HTTPS (REST, GraphQL)    │
          ▼                 ▼                ▼
┌─────────────────────────────────────────────────────┐
│                    API Layer                        │
│  ┌───────────────────────────────────────────────┐  │
│  │  Routes / Endpoints                           │  │
│  │  - GET  /api/users                            │  │
│  │  - POST /api/users                            │  │
│  │  - GET  /api/users/:id                        │  │
│  └───────────────┬───────────────────────────────┘  │
│                  │                                  │
│  ┌───────────────▼───────────────────────────────┐  │
│  │  Business Logic (Controllers)                 │  │
│  │  - Validation des données                     │  │
│  │  - Authentification / Autorisation            │  │
│  │  - Règles métier                              │  │
│  └───────────────┬───────────────────────────────┘  │
│                  │                                  │
│  ┌───────────────▼───────────────────────────────┐  │
│  │  Data Access Layer (Repository Pattern)       │  │
│  │  - Abstraction de la base de données          │  │
│  │  - Requêtes SQL / ORM                         │  │
│  └───────────────┬───────────────────────────────┘  │
└──────────────────┼──────────────────────────────────┘
                   │
                   │  SQL / Drivers
                   ▼
┌─────────────────────────────────────────────────────┐
│              PostgreSQL Database                    │
│  ┌───────────────────────────────────────────────┐  │
│  │  Tables, Views, Functions, Triggers           │  │
│  │  - users, orders, products, etc.              │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Les couches de l'architecture

#### 1. Couche de présentation (Clients)
```plaintext
Rôle : Interface utilisateur
Technologies : React, Vue, Angular, Flutter, SwiftUI, etc.
Responsabilités :
  - Affichage des données
  - Interaction utilisateur
  - Appels API (HTTP requests)
  - Gestion de l'état local
```

#### 2. Couche API (Backend)
```plaintext
Rôle : Point d'entrée pour toutes les requêtes
Technologies : Node.js, Python (Flask/Django), Java (Spring), Go, etc.
Responsabilités :
  - Routing (diriger les requêtes)
  - Authentification / Autorisation
  - Validation des entrées
  - Transformation des données
  - Gestion des erreurs
  - Rate limiting
```

#### 3. Couche d'accès aux données
```plaintext
Rôle : Interaction avec PostgreSQL
Technologies : Repository Pattern, ORM (TypeORM, SQLAlchemy), Query Builders
Responsabilités :
  - Exécuter les requêtes SQL
  - Mapper les résultats vers des objets
  - Gestion des transactions
  - Connection pooling
  - Migrations de schéma
```

#### 4. Couche de données (PostgreSQL)
```plaintext
Rôle : Stockage et persistance des données
Responsabilités :
  - Intégrité des données (contraintes)
  - Indexation et performance
  - Gestion des transactions (ACID)
  - Concurrence (MVCC)
  - Sauvegardes et récupération
```

## Pourquoi structurer son API avec PostgreSQL ?

### 1. Séparation des préoccupations (Separation of Concerns)

```plaintext
Sans séparation :
┌─────────────────────────────────────────┐
│  Application Frontend                   │
│  - SELECT * FROM users                  │
│  - UPDATE orders SET status = 'shipped' │
│  - DELETE FROM products WHERE id = 5    │
└─────────────────────────────────────────┘
  → SQL éparpillé dans tout le code
  → Maintenance cauchemardesque
  → Duplication de logique

Avec séparation :
┌─────────────────────────────────────────┐
│  Application Frontend                   │
│  - API.getUsers()                       │
│  - API.updateOrderStatus(id, 'shipped') │
│  - API.deleteProduct(5)                 │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│  API Layer                              │
│  - Logique centralisée                  │
│  - SQL bien structuré                   │
└─────────────────────────────────────────┘
  → Maintenance facile
  → Réutilisabilité
  → Une seule source de vérité
```

### 2. Sécurité renforcée

```plaintext
Accès direct à PostgreSQL :
  Client → PostgreSQL
  Risques :
    ❌ Injection SQL
    ❌ Exposition des credentials
    ❌ Pas de contrôle d'accès granulaire
    ❌ Logs insuffisants

Accès via API :
  Client → API → PostgreSQL
  Avantages :
    ✅ Requêtes paramétrées (prepared statements)
    ✅ Validation des entrées
    ✅ Authentification (JWT, OAuth)
    ✅ Autorisation (RBAC, permissions)
    ✅ Rate limiting
    ✅ Logs et audit complets
```

**Exemple d'injection SQL évitée :**
```plaintext
Sans API (vulnérable) :
  User input : email = "'; DROP TABLE users; --"
  Query directe : SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
  → Catastrophe : Table supprimée !

Avec API (sécurisé) :
  User input : email = "'; DROP TABLE users; --"
  API valide : email contient des caractères suspects → Rejet
  Ou API utilise prepared statement : $1
  Query safe : SELECT * FROM users WHERE email = $1
  Parameter : "'; DROP TABLE users; --"
  → Traité comme une chaîne, pas comme du SQL
  → Aucun risque
```

### 3. Évolutivité et maintenance

```plaintext
Changement de schéma PostgreSQL :
  users.name → users.full_name

Sans API (catastrophe) :
  → Modifier 50 fichiers dans Frontend
  → Modifier 30 fichiers dans Mobile app
  → Modifier 20 fichiers dans Admin panel
  → Risque d'oublis = bugs en production

Avec API (simple) :
  → Modifier uniquement le Repository/DAO
  → Tous les clients continuent de fonctionner
  → Déploiement transparent
```

### 4. Performance optimisée

```plaintext
Clients multiples sans API :
  Frontend : SELECT * FROM users; (charge TOUT)
  Mobile : SELECT * FROM users; (charge TOUT)
  → Bande passante gaspillée
  → Latence élevée
  → Charge inutile sur PostgreSQL

API avec optimisation :
  Frontend : GET /api/users?fields=id,name,email
  Mobile : GET /api/users?fields=id,name (minimaliste)
  → API charge uniquement les colonnes nécessaires
  → SELECT id, name, email FROM users (optimisé)
  → Indexation ciblée
  → Cache possible (Redis)
  → Pagination automatique
```

### 5. Testabilité et qualité

```plaintext
Sans API :
  Test Frontend → Nécessite PostgreSQL réelle
  → Lent, complexe, fragile

Avec API :
  Test Frontend → Mock API
  → Rapide, simple, fiable

  Test API → Base de test dédiée
  → Isolation complète

  Test unitaire Repository → Mock PostgreSQL
  → Tests microscopiques
```

### 6. Multi-plateforme et réutilisabilité

```plaintext
Une API unique sert :
  ✅ Web App (React, Vue, Angular)
  ✅ Mobile App (iOS, Android, Flutter)
  ✅ Desktop App (Electron, Qt)
  ✅ API publique (partenaires)
  ✅ Intégrations (webhooks, zapier)
  ✅ CLI (command line tools)
  ✅ Jobs background (cron, workers)

→ Logique écrite une fois
→ Utilisée partout
→ Maintenance centralisée
```

## Principes fondamentaux de conception

### 1. RESTful Design (ou GraphQL)

**REST (Representational State Transfer)** est le style architectural le plus répandu pour les APIs web.

#### Principes REST

**1.1. Ressources identifiées par URLs**
```plaintext
Ressource : User (utilisateur)
  GET    /api/users          → Liste des utilisateurs
  GET    /api/users/123      → Utilisateur #123
  POST   /api/users          → Créer un utilisateur
  PUT    /api/users/123      → Mettre à jour utilisateur #123
  DELETE /api/users/123      → Supprimer utilisateur #123

Ressource : Orders (commandes)
  GET    /api/orders         → Liste des commandes
  GET    /api/orders/456     → Commande #456
  POST   /api/orders         → Créer une commande

Ressource imbriquée : Orders d'un User
  GET    /api/users/123/orders → Commandes de l'utilisateur #123
```

**1.2. Méthodes HTTP sémantiques**
```plaintext
GET    : Lire des données (safe, idempotent)
POST   : Créer une nouvelle ressource
PUT    : Remplacer complètement une ressource (idempotent)
PATCH  : Modifier partiellement une ressource
DELETE : Supprimer une ressource (idempotent)

Idempotent = Appeler plusieurs fois = même résultat
  GET /users/123 : Toujours le même utilisateur
  DELETE /users/123 : Supprimer 1× ou 10× = même résultat
  POST /users : Créer 10× = 10 utilisateurs différents (PAS idempotent)
```

**1.3. Codes de statut HTTP**
```plaintext
2xx : Succès
  200 OK           : Requête réussie (GET)
  201 Created      : Ressource créée (POST)
  204 No Content   : Succès sans contenu (DELETE)

3xx : Redirection
  301 Moved        : Ressource déplacée définitivement
  304 Not Modified : Ressource non modifiée (cache)

4xx : Erreur client
  400 Bad Request  : Données invalides
  401 Unauthorized : Authentification requise
  403 Forbidden    : Pas les permissions
  404 Not Found    : Ressource inexistante
  409 Conflict     : Conflit (ex: email déjà utilisé)
  422 Unprocessable: Validation échouée

5xx : Erreur serveur
  500 Internal     : Erreur non gérée
  503 Unavailable  : Service temporairement indisponible
```

**1.4. Stateless (sans état)**
```plaintext
Chaque requête doit contenir TOUTES les informations nécessaires :
  ❌ Mauvais : L'API stocke "current_user" en session
  ✅ Bon : Chaque requête inclut un token JWT

Exemple :
  GET /api/users/me
  Headers: Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

  → API décode le token
  → Identifie l'utilisateur
  → Retourne ses données
  → Pas de session côté serveur
```

#### Alternative : GraphQL

```plaintext
REST : Endpoints multiples, données fixes
  GET /api/users/123      → {id, name, email, phone, address, ...}
  → Récupère TOUTES les données (over-fetching)

GraphQL : Endpoint unique, données à la carte
  POST /graphql
  Query: {
    user(id: 123) {
      id
      name
      email
    }
  }
  → Récupère UNIQUEMENT ce qui est demandé

Avantages GraphQL :
  ✅ Pas de over-fetching ni under-fetching
  ✅ Endpoint unique
  ✅ Typage fort
  ✅ Introspection (documentation auto)

Inconvénients GraphQL :
  ❌ Complexité accrue
  ❌ Caching plus difficile
  ❌ Courbe d'apprentissage
  ❌ Nécessite un resolver pour chaque champ
```

### 2. Validation des données

**Principe :** Valider TOUTES les entrées utilisateur avant d'atteindre PostgreSQL.

```plaintext
Flux de validation :
  Client → API → Validation → PostgreSQL

Niveaux de validation :
  1. Client-side : UX (feedback immédiat)
  2. API-side : OBLIGATOIRE (sécurité)
  3. Database-side : Dernier rempart (contraintes PostgreSQL)
```

**Exemple :**
```plaintext
POST /api/users
Body: {
  "email": "invalid-email",
  "age": -5,
  "username": "a"
}

Validations API :
  ❌ email : Pas un email valide
  ❌ age : Doit être > 0
  ❌ username : Minimum 3 caractères

Response :
  Status: 422 Unprocessable Entity
  Body: {
    "errors": [
      {"field": "email", "message": "Invalid email format"},
      {"field": "age", "message": "Must be positive"},
      {"field": "username", "message": "Minimum 3 characters"}
    ]
  }

→ Aucune requête envoyée à PostgreSQL
→ PostgreSQL protégé des données invalides
```

### 3. Authentification et autorisation

**Authentification** : Qui êtes-vous ?
**Autorisation** : Que pouvez-vous faire ?

#### Mécanismes d'authentification courants

**3.1. JWT (JSON Web Token)**
```plaintext
Flow :
  1. User login : POST /api/auth/login {email, password}
  2. API vérifie credentials dans PostgreSQL
  3. API génère JWT : eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
  4. Client stocke JWT (localStorage, cookie)
  5. Chaque requête : Header Authorization: Bearer <JWT>
  6. API vérifie JWT → Identifie l'utilisateur

Avantages :
  ✅ Stateless (pas de session serveur)
  ✅ Scalable (load balancing facile)
  ✅ Contient des infos (user_id, roles)

Inconvénients :
  ❌ Révocation difficile (token valide jusqu'à expiration)
  ❌ Taille (plus gros qu'un session ID)
```

**3.2. Session-based**
```plaintext
Flow :
  1. User login : POST /api/auth/login
  2. API crée session dans PostgreSQL (ou Redis)
  3. API retourne session ID (cookie)
  4. Client envoie cookie automatiquement
  5. API vérifie session dans PostgreSQL

Avantages :
  ✅ Révocation facile (DELETE session)
  ✅ Cookie sécurisé (HttpOnly, Secure)

Inconvénients :
  ❌ Stateful (session stockée serveur)
  ❌ Scalabilité complexe (session partagée)
```

**3.3. OAuth 2.0 / OpenID Connect**
```plaintext
Pour déléguer l'authentification :
  "Se connecter avec Google"
  "Se connecter avec GitHub"

Flow :
  1. Redirection vers provider (Google)
  2. User s'authentifie chez Google
  3. Google redirige vers votre app avec code
  4. App échange code contre access_token
  5. App utilise access_token pour accéder aux données
```

#### Autorisation

```plaintext
RBAC (Role-Based Access Control) :
  Users ont des Roles
  Roles ont des Permissions

Exemple :
  User "john@example.com"
    → Role "admin"
      → Permissions ["users:read", "users:write", "users:delete"]

  User "jane@example.com"
    → Role "viewer"
      → Permissions ["users:read"]

API vérifie :
  DELETE /api/users/123
  → Authentification : Qui êtes-vous ? → john@example.com
  → Autorisation : Avez-vous "users:delete" ? → Oui (admin)
  → Exécution : DELETE FROM users WHERE id = 123
```

### 4. Gestion des erreurs

**Principe :** Retourner des erreurs structurées et exploitables.

#### Format d'erreur standardisé

```plaintext
Bon format d'erreur :
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with ID 123 not found",
    "details": {
      "user_id": 123,
      "timestamp": "2025-11-23T14:30:00Z"
    }
  }
}

Status: 404 Not Found
```

#### Mapping des erreurs PostgreSQL

```plaintext
PostgreSQL → API :

1. Unique violation (23505)
   → Status: 409 Conflict
   → Message: "Email already exists"

2. Foreign key violation (23503)
   → Status: 400 Bad Request
   → Message: "Referenced user does not exist"

3. Not null violation (23502)
   → Status: 400 Bad Request
   → Message: "Field 'email' is required"

4. Connection timeout
   → Status: 503 Service Unavailable
   → Message: "Database temporarily unavailable"

→ Jamais exposer les erreurs brutes PostgreSQL au client !
```

### 5. Performance et optimisation

#### 5.1. Pagination

```plaintext
Mauvais (charge tout) :
  GET /api/users
  → SELECT * FROM users
  → 1 million de lignes retournées ❌

Bon (pagination) :
  GET /api/users?page=1&limit=50
  → SELECT * FROM users LIMIT 50 OFFSET 0
  → 50 lignes retournées ✅

Meilleur (cursor-based) :
  GET /api/users?cursor=last_id&limit=50
  → SELECT * FROM users WHERE id > last_id LIMIT 50
  → Plus efficace pour grandes tables ✅✅
```

#### 5.2. Filtering, Sorting, Searching

```plaintext
GET /api/users?status=active&sort=created_at:desc&search=john

API traduit en :
  SELECT * FROM users
  WHERE status = 'active'
    AND (name ILIKE '%john%' OR email ILIKE '%john%')
  ORDER BY created_at DESC
  LIMIT 50;

→ Permettre aux clients de filtrer côté serveur
→ Éviter de charger toutes les données
```

#### 5.3. Eager loading vs Lazy loading

```plaintext
Lazy loading (N+1 problem) :
  GET /api/users
  → SELECT * FROM users (1 requête)
  → Pour chaque user : SELECT * FROM orders WHERE user_id = ? (N requêtes)
  → Total : 1 + N requêtes ❌ Lent !

Eager loading (JOIN) :
  GET /api/users?include=orders
  → SELECT users.*, orders.*
     FROM users
     LEFT JOIN orders ON users.id = orders.user_id
  → Total : 1 requête ✅ Rapide !
```

#### 5.4. Caching

```plaintext
Sans cache :
  GET /api/users/123
  → PostgreSQL query chaque fois
  → Latence : 50ms

Avec cache (Redis) :
  GET /api/users/123
  → Vérifier cache Redis
  → Si hit : Retour immédiat (latence : 2ms) ✅
  → Si miss : Query PostgreSQL + stocker en cache

Stratégies :
  - Cache-Aside : App gère le cache
  - Write-Through : Write en DB + cache
  - Write-Behind : Write en cache, async en DB
  - TTL : Expiration automatique (ex: 5 minutes)
```

### 6. Versioning de l'API

**Principe :** Permettre l'évolution de l'API sans casser les clients existants.

```plaintext
Stratégies de versioning :

1. URL versioning (recommandé) :
   /api/v1/users
   /api/v2/users
   → Clair, explicite

2. Header versioning :
   GET /api/users
   Header: Accept: application/vnd.myapp.v2+json
   → Flexible, mais moins visible

3. Query parameter :
   GET /api/users?version=2
   → Simple, mais peut être oublié

Quand incrémenter la version ?
  - Breaking changes : v1 → v2
  - Nouvelles fonctionnalités (compatibles) : v1 reste, v2 ajoute
  - Bug fixes : pas de nouvelle version
```

### 7. Documentation

**Principe :** Une API sans documentation est une API inutilisable.

#### Formats de documentation

**OpenAPI (Swagger)**
```plaintext
Fichier YAML/JSON décrivant :
  - Tous les endpoints
  - Paramètres (path, query, body)
  - Schémas de données
  - Codes de réponse
  - Exemples

→ Génération automatique d'interface interactive (Swagger UI)
→ Génération de clients (SDK)
```

**Documentation narrative**
```plaintext
README.md complet avec :
  - Guide de démarrage rapide
  - Authentification
  - Exemples de requêtes (curl, fetch, axios)
  - Cas d'usage
  - FAQ / Troubleshooting
```

**Postman Collection**
```plaintext
Collection de requêtes pré-configurées :
  → Import dans Postman
  → Test immédiat de l'API
  → Partage avec l'équipe
```

## Les quatre piliers de cette section

Cette section **20.4. Principes de conception d'APIs avec PostgreSQL** est organisée autour de quatre concepts clés qui, ensemble, forment les fondations d'une API robuste et professionnelle :

### 20.4.1. Repository Pattern
**Objectif :** Abstraire l'accès aux données PostgreSQL.

```plaintext
Ce que vous apprendrez :
  - Séparer logique métier et accès aux données
  - Créer une couche d'abstraction réutilisable
  - Faciliter les tests et la maintenance
  - Pattern CRUD et méthodes personnalisées

Pourquoi c'est important :
  → Foundation de toute API bien structurée
  → Découplage code métier ↔ base de données
  → Testabilité et évolutivité
```

### 20.4.2. Database Migrations
**Objectif :** Gérer l'évolution du schéma PostgreSQL dans le temps.

```plaintext
Ce que vous apprendrez :
  - Versionner votre schéma comme du code
  - Flyway, Liquibase, Alembic
  - Migrations UP et DOWN
  - Synchronisation entre environnements

Pourquoi c'est important :
  → Traçabilité de l'évolution du schéma
  → Collaboration d'équipe facilitée
  → Déploiements automatisés
  → Rollback possible
```

### 20.4.3. Schema Versioning
**Objectif :** Stratégies de versionnement et compatibilité.

```plaintext
Ce que vous apprendrez :
  - Sequential, Timestamp, Semantic Versioning
  - Backward/Forward compatibility
  - Expand-Migrate-Contract pattern
  - Gestion multi-environnements

Pourquoi c'est important :
  → Coordination API ↔ Database
  → Déploiements sans casse
  → Communication claire entre équipes
  → SLA et disponibilité
```

### 20.4.4. Zero-Downtime Deployments
**Objectif :** Déployer sans interruption de service.

```plaintext
Ce que vous apprendrez :
  - Techniques PostgreSQL sans verrous
  - Blue-Green et Rolling deployments
  - Monitoring et rollback strategy
  - Cas d'études réels

Pourquoi c'est important :
  → 99.99%+ uptime
  → Déploiements fréquents possibles
  → Expérience utilisateur optimale
  → Compétitivité business
```

## Synergie des quatre piliers

```plaintext
Repository Pattern
    ↓ Fournit une abstraction propre
Database Migrations
    ↓ Gèrent l'évolution du schéma
Schema Versioning
    ↓ Coordonne versions API ↔ DB
Zero-Downtime Deployments
    ↓ Applique tout ça en production sans interruption

Résultat :
  ✅ API robuste et maintenable
  ✅ Schéma évolutif et versionné
  ✅ Déploiements sécurisés et fréquents
  ✅ Équipe confiante et productive
```

## Technologies courantes

### Frameworks API (Backend)

**Node.js / TypeScript**
```plaintext
- Express.js : Minimaliste, flexible
- Nest.js : Structuré, TypeScript-first, inspiré d'Angular
- Fastify : Performance maximale
- tRPC : Type-safe, pas de validation runtime

ORM/Query Builders :
  - TypeORM : Complet, décorateurs
  - Prisma : Modern, type-safe, migrations intégrées
  - Knex.js : Query builder SQL pur
```

**Python**
```plaintext
- Flask : Micro-framework, flexible
- FastAPI : Moderne, async, validation Pydantic, OpenAPI auto
- Django : Full-featured, admin panel inclus

ORM :
  - SQLAlchemy : Le plus complet
  - Django ORM : Intégré, simple
  - Tortoise ORM : Async-first
```

**Java**
```plaintext
- Spring Boot : Standard d'entreprise
- Quarkus : Cloud-native, GraalVM
- Micronaut : Performance, microservices

ORM :
  - Hibernate : Standard JPA
  - jOOQ : Type-safe SQL
  - Spring Data JPA : Simplification Spring
```

**Go**
```plaintext
- Gin : Rapide, minimaliste
- Echo : Simple, performant
- Fiber : Inspiré d'Express

Libraries :
  - GORM : ORM complet
  - sqlx : Extensions SQL natives
  - pgx : Driver PostgreSQL natif et performant
```

### Outils de développement API

**Documentation**
```plaintext
- Swagger UI / OpenAPI : Standard de facto
- Postman : Test et documentation
- Insomnia : Alternative à Postman
- ReDoc : Documentation élégante depuis OpenAPI
```

**Testing**
```plaintext
- Postman : Tests automatisés
- REST Client (VS Code) : Fichiers .http
- k6 : Load testing
- Artillery : Performance testing
```

**Monitoring**
```plaintext
- Prometheus + Grafana : Métriques
- Datadog : APM complet
- New Relic : Monitoring applicatif
- Sentry : Error tracking
```

## Bonnes pratiques récapitulatives

### ✅ À faire systématiquement

1. **Validation stricte** : Valider toutes les entrées utilisateur
2. **Authentification** : Sécuriser tous les endpoints (sauf publics)
3. **Rate limiting** : Prévenir les abus (ex: 100 req/min/IP)
4. **Logging** : Tracer toutes les opérations importantes
5. **Pagination** : Jamais retourner toutes les données
6. **Codes HTTP sémantiques** : 200, 201, 400, 401, 404, 500...
7. **HTTPS** : Toujours en production
8. **CORS** : Configurer correctement (pas `*` en prod)
9. **Versioning** : Anticiper l'évolution (v1, v2)
10. **Documentation** : OpenAPI/Swagger à jour

### ❌ À éviter absolument

1. **SQL brut dans les controllers** : Utiliser Repository
2. **Exposer les IDs séquentiels** : Préférer UUID ou obfuscation
3. **Retourner des stack traces** : Masquer les erreurs internes
4. **Pas de limite de requête** : Pagination obligatoire
5. **Mots de passe en clair** : Hash (bcrypt, argon2)
6. **Transactions mal gérées** : Toujours commit ou rollback
7. **N+1 queries** : Utiliser eager loading (JOIN)
8. **Pas de tests** : Au minimum smoke tests
9. **Credentials hardcodés** : Variables d'environnement
10. **Ignorer les migrations** : Schéma versionné obligatoire

## Métriques de qualité d'API

### Comment mesurer la qualité de votre API ?

**Performance**
```plaintext
- Latence P95 : < 100ms (idéal)
- Latence P99 : < 500ms (acceptable)
- Throughput : > 1000 req/sec (selon contexte)
- Temps de connexion PostgreSQL : < 10ms
```

**Fiabilité**
```plaintext
- Uptime : > 99.9% (43 min downtime/mois)
- Taux d'erreur : < 0.1%
- Taux de succès : > 99.9%
```

**Sécurité**
```plaintext
- Vulnérabilités : 0 critique, 0 haute
- Authentification : 100% des endpoints privés
- Rate limiting : Activé et testé
- Audit logs : Complets et archivés
```

**Maintenabilité**
```plaintext
- Couverture de tests : > 80%
- Documentation : Complète et à jour
- Temps de correction bugs : < 24h (critiques)
- Temps d'ajout feature : Prévisible
```

## Conclusion

La conception d'API avec PostgreSQL est un art qui combine **architecture logicielle**, **sécurité**, **performance** et **maintenabilité**. Les quatre piliers que nous allons explorer dans les sections suivantes (Repository Pattern, Database Migrations, Schema Versioning, Zero-Downtime Deployments) forment un système cohérent qui vous permettra de construire des APIs robustes, évolutives et professionnelles.

### Principes à retenir

1. **Abstraction** : Séparer couches (API ↔ Business ↔ Data ↔ PostgreSQL)
2. **Sécurité** : Valider, authentifier, autoriser, logguer
3. **Performance** : Paginer, indexer, cacher, optimiser
4. **Évolutivité** : Versionner, migrer, déployer sans interruption
5. **Qualité** : Tester, documenter, monitorer, améliorer

### Roadmap de lecture

```plaintext
Vous êtes ici → 20.4. Introduction aux principes d'API

Prochaines étapes recommandées :
  1. → 20.4.1. Repository Pattern
     Comprendre l'abstraction de l'accès aux données

  2. → 20.4.2. Database Migrations
     Versionner et gérer l'évolution du schéma

  3. → 20.4.3. Schema Versioning
     Stratégies de compatibilité et coordination

  4. → 20.4.4. Zero-Downtime Deployments
     Déployer en production sans interruption

Lectures complémentaires :
  - 20.3. Patterns anti-corruption
  - 19. PostgreSQL en Production
  - 16. Administration et Sécurité
```

Ces concepts ne sont pas juste théoriques : ils sont **appliqués quotidiennement** par les entreprises tech leaders (Stripe, GitHub, Airbnb, Netflix, etc.) pour gérer des APIs à l'échelle de millions d'utilisateurs tout en maintenant une disponibilité maximale et une qualité de code exceptionnelle.

Plongeons maintenant dans le premier pilier : le **Repository Pattern**.

---

⏭️ [Repository pattern](/20-drivers-connexion-applicative/04.1-repository-pattern.md)
