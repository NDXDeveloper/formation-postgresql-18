🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.2. Gestion des Connexions dans les Applications

## Introduction

La **gestion des connexions** est l'un des aspects les plus critiques du développement d'applications utilisant PostgreSQL. Une mauvaise gestion des connexions peut transformer une application parfaitement conçue en un système instable, lent et difficile à maintenir.

Dans cette section, nous allons explorer en profondeur comment gérer efficacement les connexions entre vos applications et PostgreSQL, depuis les concepts fondamentaux jusqu'aux stratégies avancées de production.

---

## Pourquoi la gestion des connexions est-elle cruciale ?

### Le problème : Des connexions mal gérées

Imaginez que vous développez une API REST qui reçoit **100 requêtes par seconde**. Si votre application crée une nouvelle connexion PostgreSQL pour chaque requête :

```
100 requêtes/seconde × 86400 secondes/jour = 8 640 000 connexions par jour !
```

**Conséquences désastreuses** :
- PostgreSQL sera **surchargé** (création/destruction constante de processus)
- Votre application sera **lente** (10-100 ms de latence ajoutée par connexion)
- Vous atteindrez rapidement la limite `max_connections` → **Crash**

### La solution : Une gestion intelligente

Avec une gestion appropriée des connexions :
- ✅ **Performance** : Réutilisation des connexions existantes
- ✅ **Stabilité** : Pas de saturation de PostgreSQL
- ✅ **Scalabilité** : Supporter des milliers de requêtes simultanées
- ✅ **Économie** : Moins de ressources consommées (RAM, CPU)

---

## Qu'est-ce qu'une connexion PostgreSQL ?

### Définition

Une **connexion** est un canal de communication entre votre application et le serveur PostgreSQL qui permet d'échanger des requêtes SQL et leurs résultats.

### Anatomie d'une connexion

```
┌─────────────────────────────────────────────────────────────────┐
│ Cycle de vie d'une connexion                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. INITIATION                                                  │
│     Application envoie demande de connexion                     │
│     ↓                                                           │
│  2. AUTHENTIFICATION                                            │
│     PostgreSQL vérifie credentials (user, password)             │
│     ↓                                                           │
│  3. ÉTABLISSEMENT                                               │
│     Création d'un processus serveur dédié                       │
│     Allocation de mémoire (work_mem, buffers, etc.)             │
│     ↓                                                           │
│  4. UTILISATION                                                 │
│     Échange de requêtes SQL et résultats                        │
│     ↓                                                           │
│  5. FERMETURE                                                   │
│     Libération des ressources                                   │
│     Destruction du processus serveur                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Le coût d'une connexion

Créer une connexion n'est **pas gratuit** :

| Ressource | Coût |
|-----------|------|
| **Temps** | 10-100 millisecondes |
| **CPU** | Création/destruction de processus |
| **Mémoire** | ~10-20 MB par connexion |
| **Réseau** | Négociation SSL, authentification |

**Analogie** : Créer une connexion, c'est comme passer un appel téléphonique :
- **Numérotation** → Initiation
- **Sonnerie** → Authentification
- **"Allô ?"** → Établissement
- **Conversation** → Utilisation
- **"Au revoir"** → Fermeture

Si vous devez dire 100 mots, vous ne raccrochez pas après chaque mot pour rappeler ! Vous gardez la ligne ouverte.

---

## Les différentes approches de gestion des connexions

### Approche 1 : Connexion par requête (Naïve)

**Principe** : Créer une nouvelle connexion pour chaque requête, puis la fermer.

```python
# Code INEFFICACE - NE PAS FAIRE
def get_user(user_id):
    conn = psycopg.connect("postgresql://user:pass@localhost/mydb")
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    user = cursor.fetchone()
    conn.close()
    return user

# Chaque appel = nouvelle connexion !
user1 = get_user(1)  # Connexion 1 créée puis fermée
user2 = get_user(2)  # Connexion 2 créée puis fermée
user3 = get_user(3)  # Connexion 3 créée puis fermée
```

**Avantages** :
- ✅ Code simple à comprendre
- ✅ Pas de risque de fuite de connexion

**Inconvénients** :
- ❌ **Performance catastrophique** (overhead de 10-100 ms par requête)
- ❌ **Surcharge serveur** (création/destruction constante)
- ❌ **Non scalable** (saturera rapidement `max_connections`)

**Verdict** : ❌ **À ÉVITER ABSOLUMENT en production**

### Approche 2 : Connexion persistante

**Principe** : Créer une connexion au démarrage et la réutiliser pour toutes les requêtes.

```python
# Connexion globale
conn = psycopg.connect("postgresql://user:pass@localhost/mydb")

def get_user(user_id):
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    return cursor.fetchone()

# Même connexion réutilisée
user1 = get_user(1)  # Utilise conn
user2 = get_user(2)  # Utilise conn
user3 = get_user(3)  # Utilise conn
```

**Avantages** :
- ✅ Pas d'overhead de création
- ✅ Performance améliorée

**Inconvénients** :
- ❌ **Une seule connexion** = pas de parallélisme
- ❌ **Thread-unsafe** (problèmes en multi-threading)
- ❌ **Single point of failure** (si connexion coupée, tout s'arrête)

**Verdict** : ⚠️ **OK uniquement pour scripts simples mono-thread**

### Approche 3 : Connection Pooling (RECOMMANDÉE)

**Principe** : Maintenir un **pool** (ensemble) de connexions réutilisables.

```python
from psycopg_pool import ConnectionPool

# Créer un pool au démarrage
pool = ConnectionPool(
    "postgresql://user:pass@localhost/mydb",
    min_size=5,    # Minimum 5 connexions maintenues
    max_size=20    # Maximum 20 connexions
)

def get_user(user_id):
    # Emprunter une connexion du pool
    with pool.connection() as conn:
        with conn.cursor() as cursor:
            cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            return cursor.fetchone()
    # Connexion automatiquement rendue au pool

# Les 3 appels utilisent des connexions du pool
user1 = get_user(1)  # Emprunte connexion A du pool
user2 = get_user(2)  # Emprunte connexion B du pool
user3 = get_user(3)  # Emprunte connexion C du pool
# Toutes les connexions retournent au pool après usage
```

**Avantages** :
- ✅ **Haute performance** (réutilisation)
- ✅ **Scalabilité** (plusieurs connexions en parallèle)
- ✅ **Thread-safe** (conçu pour concurrence)
- ✅ **Économie de ressources** (nombre contrôlé de connexions)

**Inconvénients** :
- ⚠️ Configuration à ajuster (taille du pool)
- ⚠️ Complexité légèrement accrue

**Verdict** : ✅ **APPROCHE RECOMMANDÉE pour toute application en production**

---

## Les niveaux de connection pooling

Il existe **deux niveaux** où implémenter le pooling :

### 1. Pooling côté application

**Principe** : Chaque instance d'application a son propre pool.

```
┌───────────────────────────────────────┐
│ Instance Application 1                │
│                                       │
│  ┌─────────────────────────────────┐  │
│  │ Connection Pool                 │  │
│  │ - Connexion 1                   │  │
│  │ - Connexion 2                   │  │
│  │ - ...                           │  │
│  │ - Connexion 10                  │  │
│  └─────────────────────────────────┘  │
│            │                          │
└────────────┼──────────────────────────┘
             │
             ▼
    ┌────────────────┐
    │  PostgreSQL    │
    └────────────────┘
```

**Bibliothèques populaires** :
- **Python** : psycopg3 (pool intégré), SQLAlchemy
- **Node.js** : node-postgres (pg)
- **Java** : HikariCP, c3p0
- **Go** : pgx/pgxpool
- **.NET** : Npgsql (pool intégré)

**Avantages** :
- ✅ Simple à mettre en place
- ✅ Latence minimale (connexions locales)
- ✅ Pas d'infrastructure supplémentaire

**Limites** :
- ⚠️ Chaque instance a son pool → Multiplication des connexions
- ⚠️ Difficile à gérer globalement en microservices

**Détails** : → Section 20.2.1

### 2. Pooling externe (PgBouncer)

**Principe** : Un proxy centralisé gère toutes les connexions.

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   App 1     │    │   App 2     │    │   App 3     │
│ (30 clients)│    │ (40 clients)│    │ (50 clients)│
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                  120 connexions clientes
                          │
                          ▼
                ┌─────────────────┐
                │   PgBouncer     │
                │   Pool = 20     │
                └────────┬────────┘
                         │
                  20 connexions serveur
                         │
                         ▼
                ┌─────────────────┐
                │   PostgreSQL    │
                └─────────────────┘
```

**Avantages** :
- ✅ **Mutualisation** : 1000 clients → 20 connexions PostgreSQL
- ✅ **Centralisation** : Un seul point de configuration
- ✅ **Scalabilité maximale** : Supporte des milliers de clients

**Limites** :
- ⚠️ Infrastructure supplémentaire à déployer
- ⚠️ Point de défaillance unique (nécessite HA)

**Détails** : → Section 20.2.2

### 3. Approche hybride (RECOMMANDÉE en production)

Combiner les deux niveaux pour le meilleur des deux mondes :

```
┌──────────────────────────────────────────────────────┐
│ Instances Application (chacune a un petit pool)      │
│                                                      │
│  App 1: pool=5    App 2: pool=5    App 3: pool=5     │
└──────────────┬───────────────┬──────────────┬────────┘
               │               │              │
               └───────────────┼──────────────┘
                               │
                         15 connexions
                               │
                               ▼
                      ┌─────────────────┐
                      │   PgBouncer     │
                      │   Pool = 20     │
                      └────────┬────────┘
                               │
                        20 connexions
                               │
                               ▼
                      ┌─────────────────┐
                      │   PostgreSQL    │
                      └─────────────────┘
```

**Avantages** :
- ✅ Latence minimale (pool applicatif local)
- ✅ Scalabilité maximale (PgBouncer global)
- ✅ Résilience (double niveau de protection)

---

## Les problèmes courants et leurs solutions

### Problème 1 : Connection Leaks (Fuites de connexions)

**Qu'est-ce que c'est ?**
Une connexion est empruntée au pool mais **jamais rendue**.

**Symptôme** :
```
Pool starts: 20 connexions disponibles
After 1 hour: 0 connexions disponibles → Application bloquée !
```

**Cause typique** :
```python
# CODE PROBLÉMATIQUE
conn = pool.connection()
cursor = conn.cursor()
cursor.execute("SELECT ...")
return cursor.fetchone()
# OUBLI : ne jamais fermer conn → Fuite !
```

**Solution** : Utiliser des gestionnaires de contexte (`with`, `try-finally`)

**Détails** : → Section 20.2.4

### Problème 2 : Timeouts

**Qu'est-ce que c'est ?**
Délais d'attente dépassés lors de l'obtention ou l'utilisation d'une connexion.

**Types** :
- **Connection timeout** : Impossible de se connecter au serveur
- **Pool timeout** : Pool saturé, pas de connexion disponible
- **Statement timeout** : Requête trop lente

**Symptôme** :
```
TimeoutError: couldn't get a connection after 30.00 sec
```

**Solution** : Configuration appropriée des timeouts + dimensionnement correct

**Détails** : → Section 20.2.4

### Problème 3 : Pool sous-dimensionné

**Symptôme** :
- Timeouts fréquents en production
- Logs montrant beaucoup de requêtes en attente

**Cause** : `pool_size` trop petit pour la charge

**Solution** : Calculer le `pool_size` approprié

**Détails** : → Section 20.2.3

### Problème 4 : Pool sur-dimensionné

**Symptôme** :
- CPU élevé sur PostgreSQL
- Beaucoup de connexions `idle`
- Performance qui se dégrade

**Cause** : `pool_size` trop grand → Surcharge PostgreSQL

**Solution** : Réduire progressivement et monitorer

**Détails** : → Section 20.2.3

### Problème 5 : Saturation de max_connections

**Symptôme** :
```
FATAL: sorry, too many clients already
```

**Cause** : Trop d'applications/instances se connectent directement

**Solution** : Introduire PgBouncer pour mutualiser

**Détails** : → Section 20.2.2

---

## Concepts clés à comprendre

### 1. Pool Size vs Max Connections

**Pool Size** (côté application/PgBouncer) :
```python
pool = ConnectionPool(max_size=20)  # Max 20 connexions dans ce pool
```

**Max Connections** (côté PostgreSQL) :
```conf
max_connections = 100  # PostgreSQL accepte max 100 connexions totales
```

**Relation critique** :
```
Somme de tous les pool_size < max_connections

Exemple :
- 5 instances × pool_size=10 = 50 connexions
- max_connections doit être > 50 (recommandé: 100)
```

**Détails** : → Section 20.2.3

### 2. Connection States (États des connexions)

Une connexion PostgreSQL peut être dans différents états :

| État | Description | Normal ? |
|------|-------------|----------|
| **active** | Exécute une requête | ✅ Oui |
| **idle** | Inactive, en attente | ✅ Oui (si pool) |
| **idle in transaction** | Transaction ouverte mais inactive | ⚠️ Problème potentiel |
| **idle in transaction (aborted)** | Transaction en erreur | ❌ Problème |

**Voir les états** :
```sql
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state;
```

### 3. Transactions et connexions

**Principe important** : Une transaction est toujours liée à **une seule connexion**.

```python
# Transaction commence
with pool.connection() as conn:  # Emprunte connexion A
    conn.execute("BEGIN")
    conn.execute("INSERT INTO users ...")
    conn.execute("UPDATE accounts ...")
    conn.execute("COMMIT")
# Transaction termine, connexion A rendue au pool

# Nouvelle transaction, probablement une autre connexion
with pool.connection() as conn:  # Emprunte connexion B
    conn.execute("BEGIN")
    # ...
```

**Important** : Ne pas partager une connexion entre threads !

### 4. Thread Safety

**Pool thread-safe** : Plusieurs threads peuvent emprunter des connexions en parallèle.

```python
from threading import Thread

def worker(user_id):
    with pool.connection() as conn:  # Thread-safe
        # Chaque thread obtient sa propre connexion
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        return cursor.fetchone()

# 10 threads en parallèle
threads = [Thread(target=worker, args=(i,)) for i in range(10)]
for t in threads:
    t.start()
```

**Important** : Ne jamais partager **une connexion** entre threads, mais le **pool** oui !

---

## Vue d'ensemble des sections suivantes

Cette section 20.2 est organisée en quatre parties progressives :

### 20.2.1. Connection Pooling Côté Application

**Vous apprendrez** :
- Configuration des pools par langage (Python, Node.js, Java, Go, .NET)
- Dimensionnement optimal du pool
- Bonnes pratiques de libération des connexions
- Monitoring et métriques du pool

**Public** : Développeurs d'applications

### 20.2.2. PgBouncer : Transaction vs Session Pooling

**Vous apprendrez** :
- Installation et configuration de PgBouncer
- Modes de pooling (transaction, session, statement)
- Quand utiliser quel mode
- Administration et troubleshooting
- Architecture haute disponibilité

**Public** : Développeurs et DevOps

### 20.2.3. Dimensionnement (max_connections vs pool_size)

**Vous apprendrez** :
- Formules de calcul pour dimensionner correctement
- Relation entre pool_size et max_connections
- Cas d'usage réels avec calculs
- Stratégies d'optimisation
- Checklist de configuration

**Public** : Architectes et DevOps

### 20.2.4. Connection Leaks et Timeouts

**Vous apprendrez** :
- Détecter et corriger les fuites de connexions
- Configurer les différents timeouts
- Patterns de résilience (retry, circuit breaker)
- Diagnostic et troubleshooting en production
- Solutions d'urgence

**Public** : Tous (particulièrement important !)

---

## Prérequis et outils

### Connaissances requises

**Minimum** :
- Bases de programmation dans un langage (Python, Java, Node.js, etc.)
- Compréhension basique de SQL
- Notions de connexion client-serveur

**Recommandé** :
- Expérience avec une application web ou API
- Notions de multi-threading/concurrence
- Compréhension basique de PostgreSQL

### Outils nécessaires

**Pour suivre les exemples** :
- PostgreSQL 12+ installé (18 pour les nouveautés)
- Un langage de programmation au choix
- Un client PostgreSQL (psql, pgAdmin, DBeaver)

**Pour la production** :
- Monitoring : Prometheus + Grafana (recommandé)
- Pooler externe : PgBouncer
- Logs : pgBadger ou équivalent

---

## Méthodologie de lecture

### Parcours recommandé

**Si vous débutez** :
```
1. Lire cette introduction complètement
2. Section 20.2.1 (Connection Pooling Applicatif)
3. Tester avec votre langage préféré
4. Section 20.2.4 (Leaks et Timeouts) - CRUCIAL
5. Section 20.2.3 (Dimensionnement) si vous scalez
6. Section 20.2.2 (PgBouncer) si architecture complexe
```

**Si vous êtes expérimenté** :
```
1. Parcourir cette introduction rapidement
2. Section 20.2.2 (PgBouncer) si vous ne l'utilisez pas encore
3. Section 20.2.3 (Dimensionnement) pour optimiser
4. Section 20.2.4 (Leaks et Timeouts) pour résoudre des problèmes
```

**Si vous avez un problème en production** :
```
→ Aller directement à la Section 20.2.4 (Leaks et Timeouts)
```

### Comment tirer le meilleur parti

**✅ À faire** :
- Tester les exemples de code dans votre environnement
- Adapter les configurations à votre contexte
- Monitorer les métriques avant/après changements
- Documenter vos choix de configuration

**❌ À éviter** :
- Copier-coller sans comprendre
- Appliquer en production sans tester
- Ignorer les sections de monitoring
- Négliger les cas d'erreur

---

## Checklist avant de commencer

Avant de plonger dans les sections détaillées, assurez-vous de :

```
☐ Avoir une instance PostgreSQL de test disponible
☐ Connaître votre langage de programmation cible
☐ Avoir installé le driver PostgreSQL pour votre langage
☐ Pouvoir exécuter des requêtes SQL
☐ Avoir accès aux logs PostgreSQL (si possible)
☐ Avoir un outil de monitoring basique (htop, top, ou équivalent)
```

**Optionnel mais recommandé** :
```
☐ PgBouncer installé (pour section 20.2.2)
☐ Prometheus + Grafana (pour monitoring avancé)
☐ pgBadger (pour analyse de logs)
☐ Environnement de test isolé (Docker, VM, etc.)
```

---

## Premiers pas : Configuration minimale

Avant de commencer les sections détaillées, voici une configuration minimale pour démarrer :

### PostgreSQL (postgresql.conf)

```conf
# Connexions
max_connections = 100                           # Par défaut, souvent suffisant
shared_buffers = 256MB                          # Ajuster selon RAM
work_mem = 4MB                                  # Par opération

# Logs (pour debugging)
log_connections = on
log_disconnections = on
log_line_prefix = '%t [%p] %u@%d '

# Timeouts de sécurité
idle_in_transaction_session_timeout = 300000    # 5 minutes
statement_timeout = 60000                       # 1 minute (ajuster selon besoin)
```

### Application - Python (exemple minimal)

```python
from psycopg_pool import ConnectionPool

# Pool de connexions basique
pool = ConnectionPool(
    "postgresql://user:password@localhost:5432/mydb",
    min_size=2,
    max_size=10,
    timeout=30
)

# Utilisation correcte avec context manager
def get_data(query):
    with pool.connection() as conn:
        with conn.cursor() as cursor:
            cursor.execute(query)
            return cursor.fetchall()

# Fermer le pool à l'arrêt de l'application
# pool.close()
```

### Vérification - SQL de monitoring

```sql
-- Voir les connexions actuelles
SELECT count(*) as total,
       count(*) FILTER (WHERE state = 'active') as active,
       count(*) FILTER (WHERE state = 'idle') as idle
FROM pg_stat_activity
WHERE datname = current_database();

-- Voir l'utilisation de max_connections
SELECT
    count(*) as used,
    current_setting('max_connections')::int as max,
    (count(*)::float / current_setting('max_connections')::int * 100)::numeric(5,2) as percent
FROM pg_stat_activity;
```

---

## Ressources complémentaires

### Documentation officielle

- [PostgreSQL Connection Strings](https://www.postgresql.org/docs/current/libpq-connect.html)
- [PostgreSQL Runtime Config - Connections](https://www.postgresql.org/docs/current/runtime-config-connection.html)

### Drivers et bibliothèques

- **Python** : [psycopg3](https://www.psycopg.org/psycopg3/) | [SQLAlchemy](https://www.sqlalchemy.org/)
- **Node.js** : [node-postgres](https://node-postgres.com/)
- **Java** : [HikariCP](https://github.com/brettwooldridge/HikariCP)
- **Go** : [pgx](https://github.com/jackc/pgx)
- **.NET** : [Npgsql](https://www.npgsql.org/)

### Outils externes

- **PgBouncer** : [https://www.pgbouncer.org/](https://www.pgbouncer.org/)
- **pgBadger** : [https://pgbadger.darold.net/](https://pgbadger.darold.net/)
- **pg_stat_statements** : Extension PostgreSQL pour statistiques de requêtes

### Articles recommandés

- [HikariCP - About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing) (applicable à tous les langages)
- [PostgreSQL Connection Pooling](https://www.postgresql.org/docs/current/pgbouncer.html)

---

## Points clés à retenir

### ✨ Concepts fondamentaux

1. **Une connexion = ressource coûteuse** (temps, CPU, mémoire)
2. **Connection pooling = réutilisation** de connexions existantes
3. **Deux niveaux de pooling** : application (local) et externe (PgBouncer)
4. **Configuration critique** : `pool_size` et `max_connections` doivent être cohérents

### 🎯 Règles d'or

1. **Toujours utiliser un connection pool** en production
2. **Toujours fermer/rendre les connexions** (with, try-finally, etc.)
3. **Monitorer en continu** l'utilisation du pool
4. **Dimensionner selon la charge** réelle, pas des estimations

### ⚠️ Erreurs à éviter

1. ❌ Créer une connexion par requête
2. ❌ Oublier de fermer les connexions (leaks)
3. ❌ Pool trop grand (surcharge PostgreSQL)
4. ❌ Pool trop petit (timeouts)
5. ❌ Ignorer les métriques de monitoring

---

## Prêt à commencer ?

Vous avez maintenant une vue d'ensemble complète de la gestion des connexions. Choisissez votre parcours :

### 👉 Développeurs d'applications
**Commencez par** : Section 20.2.1 - Connection Pooling Côté Application
Apprenez à implémenter correctement le pooling dans votre code.

### 👉 Architectes / DevOps
**Commencez par** : Section 20.2.2 - PgBouncer
Découvrez comment centraliser et optimiser la gestion des connexions.

### 👉 Troubleshooters
**Commencez par** : Section 20.2.4 - Connection Leaks et Timeouts
Résolvez les problèmes de connexion en production.

---


⏭️ [Connection pooling côté application](/20-drivers-connexion-applicative/02.1-connection-pooling-application.md)
