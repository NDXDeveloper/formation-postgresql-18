🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1. Le Modèle Client-Serveur et le Protocole Réseau (Wire Protocol 3.2)

## 📋 Introduction

Avant de plonger dans les détails techniques de PostgreSQL, il est essentiel de comprendre comment fonctionne la communication entre une application et le serveur de base de données. Cette section explique les fondements du modèle client-serveur et comment PostgreSQL organise ses échanges de données.

---

## 🔍 Qu'est-ce que le Modèle Client-Serveur ?

### Définition Simple

Le **modèle client-serveur** est une architecture informatique qui sépare deux rôles distincts :

- **Le Client** : C'est l'application ou l'outil qui fait une demande (par exemple, votre application web, psql, pgAdmin, etc.)  
- **Le Serveur** : C'est le système qui reçoit la demande, la traite, et renvoie une réponse (dans notre cas, PostgreSQL)

### Analogie du Restaurant 🍽️

Pour mieux comprendre, imaginez un restaurant :

```
┌─────────────┐                           ┌─────────────┐
│   CLIENT    │                           │   SERVEUR   │
│             │                           │             │
│  (Vous, le  │  ──── Commande ────────>  │   (Le chef  │
│   client)   │                           │ et cuisine) │
│             │  <──── Plat préparé ────  │             │
└─────────────┘                           └─────────────┘
```

- **Vous** (le client) passez une commande  
- **Le chef** (le serveur) prépare votre plat  
- **Le serveur** vous ramène le plat préparé  
- **Vous ne voyez pas** ce qui se passe en cuisine (abstraction)

### Application à PostgreSQL

Avec PostgreSQL, le principe est identique :

```
┌──────────────────┐                    ┌──────────────────┐
│  APPLICATION     │                    │   POSTGRESQL     │
│                  │                    │                  │
│ - Application    │ ── Requête SQL ──> │ - Analyse SQL    │
│   Web/Mobile     │                    │ - Accès disque   │
│ - Script Python  │ <─── Résultats ─── │ - Traitement     │
│ - psql (CLI)     │                    │ - Réponse        │
│ - pgAdmin        │                    │                  │
└──────────────────┘                    └──────────────────┘
```

---

## 🌐 Pourquoi un Modèle Client-Serveur ?

### 1. **Séparation des Responsabilités**

- Le **client** se concentre sur l'interface utilisateur et la logique métier
- Le **serveur** se concentre sur la gestion des données, la sécurité, et l'intégrité

### 2. **Centralisation des Données**

```
        Client 1 (Web)
              ↓
        Client 2 (Mobile)  ──────> SERVEUR PostgreSQL (Base unique)
              ↓
        Client 3 (Desktop)
```

Tous les clients accèdent à la **même base de données centralisée**, garantissant la cohérence.

### 3. **Sécurité et Contrôle**

- Le serveur contrôle **qui** peut accéder à **quoi**
- Les données sensibles restent sur le serveur
- Les clients n'ont pas accès direct aux fichiers de données

### 4. **Performance et Optimisation**

- Le serveur peut être optimisé spécifiquement pour la gestion de données
- Le client peut être léger et rapide
- La charge de calcul est répartie intelligemment

---

## 🔌 Comment Client et Serveur Communiquent-ils ?

### Le Protocole Réseau

Un **protocole** est un ensemble de règles qui définit comment deux systèmes communiquent. Pour PostgreSQL, ce protocole s'appelle le **Wire Protocol** (protocole filaire).

### Analogie de la Conversation Téléphonique 📞

Quand vous téléphonez, vous suivez un protocole implicite :

1. **Vous composez** le numéro (établissement de connexion)
2. **Vous dites « Allô »** (salutation initiale)
3. **Vous posez votre question** (requête)
4. **Votre interlocuteur répond** (réponse)
5. **Vous dites « Au revoir »** (fermeture de connexion)

PostgreSQL fonctionne de manière similaire, mais de façon beaucoup plus structurée.

---

## 📡 Le Wire Protocol de PostgreSQL (Version 3.2)

### Qu'est-ce que le Wire Protocol ?

Le **Wire Protocol** (littéralement « protocole du câble ») est le langage de communication standardisé entre un client PostgreSQL et le serveur. La version actuelle est **3.2**, introduite dans PostgreSQL 18 — c'est la première mise à jour majeure du protocole depuis la version 3.0 de PostgreSQL 7.4 en 2003.

### Caractéristiques du Wire Protocol 3.2

1. **Protocole Binaire** : Les données sont transmises en format binaire (plus efficace que du texte)  
2. **Orienté Message** : La communication se fait par échange de messages typés  
3. **Stateful** : Le serveur maintient l'état de la connexion (transaction en cours, etc.)  
4. **Basé sur TCP/IP** : Utilise le protocole réseau TCP pour garantir la fiabilité

---

## 🔄 Le Cycle de Communication Client-Serveur

### Étape 1 : Établissement de la Connexion

```
CLIENT                                          SERVEUR
  |                                               |
  |──── 1. Demande de connexion (TCP) ──────────> |
  |                                               |
  | <──── 2. Acceptation de connexion ────────────|
  |                                               |
  |──── 3. Informations d'authentification ─────> |
  |          (utilisateur, mot de passe)          |
  |                                               |
  | <──── 4. Authentification réussie ────────────|
  |        (Prêt à recevoir des requêtes)         |
```

**Ce qui se passe :**
- Le client initie une connexion TCP vers le serveur (généralement sur le port **5432**)
- Le serveur accepte la connexion
- Le client envoie ses identifiants (nom d'utilisateur, mot de passe, base de données cible)
- Le serveur vérifie les droits d'accès (via `pg_hba.conf`)
- Si tout est OK, la connexion est établie

### Étape 2 : Envoi d'une Requête

```
CLIENT                                          SERVEUR
  |                                               |
  |──── Message "Query" ──────────────────────>   |
  |     Contenu: "SELECT * FROM users;"           |
  |                                               |
```

**Ce qui se passe :**
- Le client envoie une **requête SQL** encapsulée dans un message de type « Query »
- Le message contient :
  - Le type de message (Query = `'Q'`)
  - La longueur du message
  - La requête SQL en texte

### Étape 3 : Traitement et Réponse

```
CLIENT                                          SERVEUR
  |                                               |
  | <──── Message "RowDescription" ───────────────|
  |       (Description des colonnes résultats)    |
  |                                               |
  | <──── Message "DataRow" ───────────────────── |
  |       (Ligne 1 de résultats)                  |
  |                                               |
  | <──── Message "DataRow" ───────────────────── |
  |       (Ligne 2 de résultats)                  |
  |                                               |
  | <──── Message "CommandComplete" ──────────────|
  |       (Fin de la requête, nb lignes)          |
  |                                               |
  | <──── Message "ReadyForQuery" ────────────────|
  |       (Prêt pour une nouvelle requête)        |
```

**Ce qui se passe :**
1. **RowDescription** : Le serveur décrit les colonnes (nom, type, taille)  
2. **DataRow** : Le serveur envoie chaque ligne de résultat  
3. **CommandComplete** : le serveur signale la fin (ex. : `SELECT 2` pour 2 lignes)
4. **ReadyForQuery** : Le serveur indique qu'il est prêt pour une nouvelle requête

### Étape 4 : Fermeture de la Connexion

```
CLIENT                                          SERVEUR
  |                                               |
  |──── Message "Terminate" ───────────────────>  |
  |                                               |
  | <──── Fermeture TCP ──────────────────────────|
```

---

## 📦 Anatomie d'un Message du Wire Protocol

Chaque message échangé suit une structure précise :

```
┌─────────────────────────────────────┐
│  Type de Message (1 byte)           │  Ex. : 'Q' pour Query, 'D' pour DataRow
├─────────────────────────────────────┤
│  Longueur du Message (4 bytes)      │  Ces 4 octets + le contenu (hors octet de type)
├─────────────────────────────────────┤
│  Contenu du Message (variable)      │  Données spécifiques au message
└─────────────────────────────────────┘
```

### Exemple Concret : Message « Query »

```
Type:     'Q' (Query)  
Longueur: 25 octets (le champ « longueur » compte ses propres 4 octets + les 21 octets de la chaîne)  
Contenu:  "SELECT * FROM users;\0"  ← \0 = caractère null de fin  
```

### Principaux Types de Messages

#### Messages Client → Serveur

| Type | Nom              | Description                                  |
|------|------------------|----------------------------------------------|
| `Q`  | Query            | Envoi d'une requête SQL simple               |
| `P`  | Parse            | Prépare une requête (Prepared Statement)     |
| `B`  | Bind             | Lie des paramètres à une requête préparée    |
| `E`  | Execute          | Exécute une requête préparée                 |
| `X`  | Terminate        | Demande de fermeture de connexion            |

#### Messages Serveur → Client

| Type | Nom              | Description                                  |
|------|------------------|----------------------------------------------|
| `T`  | RowDescription   | Description des colonnes du résultat         |
| `D`  | DataRow          | Une ligne de données                         |
| `C`  | CommandComplete  | Fin de l'exécution de commande               |
| `Z`  | ReadyForQuery    | Serveur prêt pour nouvelle requête           |
| `E`  | ErrorResponse    | Message d'erreur                             |
| `N`  | NoticeResponse   | Message d'avertissement                      |

### Le Protocole Étendu : Parse / Bind / Execute

Le message `Q` (Query simple) suffit pour exécuter une requête, mais expose à deux problèmes :
1. **Injection SQL** si les paramètres sont concaténés dans la chaîne SQL
2. **Re-parsing** à chaque appel même pour des requêtes identiques (gaspillage CPU)

Le **protocole étendu** résout les deux en séparant la requête en trois étapes :

```
CLIENT                                          SERVEUR
  |                                               |
  |──── Parse ('P') ───────────────────────────>  |
  |     "INSERT INTO users(name) VALUES ($1)"     |
  |     ↑ La requête est analysée et planifiée    |
  |                                               |
  |──── Bind ('B') ────────────────────────────>  |
  |     $1 = "Alice"                              |
  |     ↑ Les paramètres sont liés à la requête   |
  |                                               |
  |──── Execute ('E') ─────────────────────────>  |
  |     ↑ La requête est exécutée                 |
  |                                               |
  | <──── CommandComplete ('C') ──────────────────|
```

**Avantages** :
- ✅ **Anti-injection SQL** : les valeurs des paramètres ne sont jamais interprétées comme du SQL
- ✅ **Réutilisation** : la même `Parse` peut être suivie de plusieurs `Bind`/`Execute` avec des valeurs différentes
- ✅ **Plans préparés** : PostgreSQL peut mettre en cache le plan d'exécution (`generic_plan`)
- ✅ **Types explicites** : pas de conversion ambiguë

C'est ce que font automatiquement la plupart des drivers modernes (psycopg3, pgx, JDBC) quand vous utilisez des requêtes paramétrées :

```python
# psycopg3 — utilise automatiquement le protocole étendu
cur.execute("INSERT INTO users(name) VALUES (%s)", ("Alice",))
#                                            ↑ $1 côté serveur
```

> ⚠️ **Implication pour PgBouncer en mode `transaction`** : les prepared statements *nommées* étaient historiquement incompatibles avec ce mode (perdues entre transactions). Depuis PgBouncer 1.21+ (2023), un support natif des prepared statements en mode transaction a été ajouté — détail au chapitre 16.

---

## 🔐 Sécurité et Authentification dans le Wire Protocol

### Méthodes d'Authentification Supportées

Lors de l'établissement de la connexion, plusieurs méthodes d'authentification sont possibles :

1. **trust** : Pas d'authentification (dangereux, seulement pour dev local)  
2. **peer** / **ident** : Identité de l'utilisateur du système d'exploitation (souvent le défaut en local sur Linux — c'est ce qui explique que `sudo -u postgres psql` fonctionne sans mot de passe)  
3. **password** : Mot de passe en clair (déprécié)  
4. **md5** : Hash MD5 du mot de passe (déprécié)  
5. **scram-sha-256** : Standard moderne sécurisé (recommandé)  
6. **cert** : Certificat SSL/TLS client  
7. **OAuth 2.0** : Nouveau dans PostgreSQL 18

D'autres méthodes existent pour l'intégration en entreprise : **LDAP**, **GSSAPI**/**Kerberos**, **RADIUS**, **PAM**. La configuration de tout cela (fichier `pg_hba.conf`) est détaillée au **chapitre 16**.

### Flux d'Authentification SCRAM-SHA-256

```
CLIENT                                          SERVEUR
  |                                               |
  |──── 1. Demande connexion ──────────────────>  |
  |                                               |
  | <──── 2. Challenge SCRAM ─────────────────────|
  |         (Méthode: SCRAM-SHA-256)              |
  |                                               |
  |──── 3. Réponse client (Hash) ─────────────>   |
  |                                               |
  | <──── 4. Vérification serveur ────────────────|
  |         (OK ou Erreur)                        |
```

**Avantage** : le mot de passe n'est jamais transmis en clair sur le réseau.

### Chiffrement en Transit : TLS/SSL

L'authentification protège l'identité, mais pas le contenu des requêtes et résultats. Pour cela, PostgreSQL supporte **TLS** (souvent encore appelé « SSL » par tradition).

**Le mode `sslmode` côté client** détermine comment l'application traite TLS :

| `sslmode` | Comportement | Quand l'utiliser |
|-----------|--------------|------------------|
| `disable` | Pas de TLS | Jamais en production, jamais sur internet |
| `allow` | Essaie sans TLS, puis avec | Dépannage |
| `prefer` | Essaie TLS, retombe en clair si refusé | Défaut historique, peu sûr |
| `require` | TLS obligatoire, mais ne vérifie pas le certificat | Réseau interne de confiance |
| `verify-ca` | TLS + vérification de la chaîne de certification | Production standard |
| `verify-full` | TLS + vérif chaîne **et** correspondance du nom d'hôte | **Recommandé en production**, surtout sur internet |

> 🔐 **Bonne pratique** : utilisez **`verify-full`** dès que possible. Sans cette vérification, un attaquant en position d'homme-du-milieu (MITM) peut présenter son propre certificat TLS et intercepter vos données — même si le mot de passe SCRAM reste protégé, vos requêtes et résultats sont lisibles.

**Configuration côté serveur** (`postgresql.conf`) :

```ini
ssl = on  
ssl_cert_file = '/etc/postgresql/server.crt'  
ssl_key_file  = '/etc/postgresql/server.key'  
ssl_ca_file   = '/etc/postgresql/root.crt'  # Pour valider les certificats clients (cert auth)  
ssl_min_protocol_version = 'TLSv1.2'         # Refuser TLS 1.0 / 1.1 (vulnérables)  
```

**Nouveauté PostgreSQL 18** : paramètre `ssl_tls13_ciphers` pour configurer finement les suites cryptographiques TLS 1.3 (utile pour la conformité FIPS, environnements gouvernementaux ou financiers).

---

## 🚀 Nouveautés du Wire Protocol 3.2 (PostgreSQL 18)

PostgreSQL 18 a introduit la version 3.2 du protocole, la première mise à jour depuis la version 3.0 de PostgreSQL 7.4 (2003).

### Clés d'Annulation 256 bits (Sécurité Renforcée)

La nouveauté principale du Protocol 3.2 est le passage des clés d'annulation de requêtes (*cancel keys*) de 32 bits à **256 bits**.

**Contexte** : Quand un client veut annuler une requête en cours (`Ctrl+C` dans psql), il envoie une *cancel request* contenant une clé secrète. Avec 32 bits, cette clé était vulnérable aux attaques par force brute — un attaquant sur le même réseau pouvait deviner la clé et annuler les requêtes d'autres utilisateurs.

```
Protocole 3.0 (ancien) :
  Cancel Key : 32 bits → ~4 milliards de combinaisons (force brute possible)

Protocole 3.2 (nouveau) :
  Cancel Key : 256 bits → résistant à la force brute
```

**Compatibilité** : Le protocole est rétrocompatible — les anciens clients continuent de fonctionner avec les nouveaux serveurs, et vice versa.

### Rappel : Le Pipelining (disponible depuis PostgreSQL 14)

Bien que ce ne soit pas une nouveauté du Protocol 3.2, le **pipelining** mérite d'être mentionné car il optimise fortement les performances réseau :

```
TRADITIONNEL (séquentiel):
  Requête 1 ───> Attente ───> Réponse 1 ───> Requête 2 ───> Attente ───> Réponse 2

PIPELINÉ:
  Requête 1 ─┐
  Requête 2 ─┼──> Traitement concurrent ──> Réponse 1 + 2
  Requête 3 ─┘
```

**Avantage** : Réduction de la latence totale, particulièrement bénéfique sur les connexions à haute latence.

### Messages d'Erreur Structurés

PostgreSQL fournit des messages d'erreur très détaillés avec des champs structurés :

```
ERROR: column "username" does not exist  
DETAIL: Perhaps you meant to reference column "user_name"  
HINT: Try using double quotes if the name contains special characters  
POSITION: 8 (position dans la requête)  
```

> 💡 Ces champs structurés (DETAIL, HINT, POSITION) sont disponibles depuis longtemps dans PostgreSQL, mais sont souvent méconnus. Les drivers modernes permettent d'accéder à chacun de ces champs individuellement pour un meilleur diagnostic.

---

## 🌍 Connexion Réseau : Local vs Distant

### Connexion Locale (Socket Unix)

Sur Linux/Mac, PostgreSQL peut utiliser des **sockets Unix** pour les connexions locales :

```
CLIENT (même machine)          SERVEUR PostgreSQL
         |                            |
         └─── Socket Unix ────────────┘
              /var/run/postgresql/.s.PGSQL.5432
```

**Avantages** :
- Plus rapide (pas de stack TCP/IP)
- Plus sécurisé (pas d'exposition réseau)
- Idéal pour les applications sur le même serveur

### Connexion Réseau TCP/IP

Pour les connexions distantes :

```
CLIENT (Machine A)                   SERVEUR PostgreSQL (Machine B)
   App Python                             Port 5432
      |                                       |
      └────── Internet/Réseau local ──────────┘
              TCP/IP (IPv4 ou IPv6)
```

**Configuration** :
- Le serveur doit écouter sur l'interface réseau (`listen_addresses = '*'`)
- Le pare-feu doit autoriser le port 5432
- Le fichier `pg_hba.conf` doit autoriser les connexions distantes

### Connection Strings : exprimer une connexion en une chaîne

PostgreSQL accepte deux formats pour exprimer une URL de connexion. Tous les drivers majeurs (`libpq`, `psycopg`, JDBC, `pgx`, Npgsql…) les comprennent.

**Format URI (recommandé, lisible)** :

```
postgresql://utilisateur:motdepasse@hote:port/base?param1=valeur1&param2=valeur2
```

**Exemples** :

```bash
# Connexion simple
postgresql://alice@localhost/mydb

# Avec mot de passe et port
postgresql://alice:secret@db.example.com:5432/production

# Avec paramètres : SSL obligatoire, timeout, application name
postgresql://alice:secret@db.example.com/prod?sslmode=require&connect_timeout=10&application_name=batch_job

# Socket Unix (chemin du répertoire via le paramètre host)
postgresql:///mydb?host=/var/run/postgresql

# Adresse IPv6 (entre crochets)
postgresql://alice@[::1]:5432/mydb

# Multi-hôtes (failover automatique côté client)
postgresql://alice@host1,host2,host3:5432/mydb?target_session_attrs=primary
```

**Format key-value (historique, parfois plus lisible pour beaucoup d'options)** :

```
host=db.example.com port=5432 dbname=production user=alice password=secret sslmode=require
```

**Paramètres utiles** :

| Paramètre | Rôle |
|-----------|------|
| `sslmode` | `disable`, `prefer`, `require`, `verify-ca`, `verify-full` |
| `connect_timeout` | Timeout en secondes pour établir la connexion |
| `application_name` | Nom apparaissant dans `pg_stat_activity` (très utile pour le debug) |
| `target_session_attrs` | `any`, `read-write`, `read-only`, `primary`, `standby` (failover) |
| `options` | Options de session, ex. `-c search_path=app,public` |

> 💡 **`application_name` est sous-utilisé** : positionnez-le systématiquement dans vos applications (`SET application_name = 'mon_service'` ou via la connection string). Lors d'un incident en production, savoir quelle application a lancé une requête bloquante vous fait gagner un temps fou.

---

## 🛠️ Outils Utilisant le Wire Protocol

Tous les outils PostgreSQL communiquent via le Wire Protocol :

### 1. **psql** (CLI officiel)
```bash
$ psql -h localhost -p 5432 -U mon_user -d ma_base
```
→ Établit une connexion Wire Protocol

### 2. **pgAdmin** (Interface graphique)
→ Utilise le Wire Protocol pour toutes les opérations

### 3. **Drivers de Programmation**
- **Python** : `psycopg3`  
- **Node.js** : `pg` (node-postgres)  
- **Java** : `JDBC`  
- **Go** : `pgx`

Tous implémentent le Wire Protocol pour communiquer avec PostgreSQL.

### 4. **PgBouncer** (Connection Pooler)
→ Se positionne entre clients et serveur, parle Wire Protocol des deux côtés

```
Clients ───> PgBouncer ───> PostgreSQL
         (Wire Protocol)  (Wire Protocol)
```

---

## 📊 Performance et Latence

### Facteurs Impactant la Performance du Wire Protocol

1. **Latence Réseau**
   - Local (socket) : < 1 ms
   - Même datacenter : 1-5 ms
   - Même région (cloud) : 5-20 ms
   - Cross-région : 50-200+ ms

2. **Taille des Résultats**
   - 10 lignes : Quasi-instantané
   - 10 000 lignes : Impact significatif
   - 1 million lignes : Utiliser pagination/streaming

3. **Nombre de Round-Trips**
   - 1 requête simple : 1 round-trip
   - Requête préparée : souvent **1 seul** round-trip — les drivers groupent Parse/Bind/Execute/Sync dans un même envoi, et réutilisent le `Parse` mis en cache pour les exécutions suivantes
   - Transaction explicite : +1 round-trip pour `BEGIN`, +1 pour `COMMIT`

### Bonnes Pratiques pour Minimiser la Latence

- ✅ **Utiliser des requêtes préparées** (pour les requêtes répétées)  
- ✅ **Regrouper les opérations** dans une transaction  
- ✅ **Limiter les résultats** avec `LIMIT` / pagination  
- ✅ **Utiliser le pipelining** si supporté par le driver  
- ✅ **Connecter via socket Unix** pour les applications locales  
- ✅ **Récupérer les gros résultats par curseur** (`DECLARE … CURSOR` / `FETCH`) plutôt qu'en un seul bloc *(PostgreSQL n'a pas de compression native du protocole ; passez par TLS ou un tunnel SSH si vous devez compresser)*

---

## 🔍 Débogage : Capturer le Trafic Wire Protocol

Pour les utilisateurs avancés, il est possible d'observer le Wire Protocol en action.

### Avec `tcpdump` (Linux)

```bash
sudo tcpdump -i lo -X port 5432
```

Cela capture tous les paquets sur le port 5432 et affiche leur contenu.

### Avec Wireshark

Wireshark peut **décoder le protocole PostgreSQL** et afficher les messages de manière lisible :

```
Message: Query
  Type: Q
  SQL: SELECT * FROM users WHERE id = 1;

Message: RowDescription
  Columns: 4
    - id (INTEGER)
    - username (TEXT)
    - email (TEXT)
    - created_at (TIMESTAMP)

Message: DataRow
  Values: [1, "alice", "alice@example.com", "2024-01-15"]

Message: CommandComplete
  Status: SELECT 1
```

---

## 📝 Résumé des Concepts Clés

### Le Modèle Client-Serveur
- ✅ Sépare l'application (client) de la base de données (serveur)  
- ✅ Centralise les données et la sécurité  
- ✅ Permet à plusieurs clients d'accéder à la même base

### Le Wire Protocol 3.2
- ✅ Langage de communication standardisé entre client et serveur  
- ✅ Basé sur l'échange de messages typés  
- ✅ Utilise TCP/IP (réseau) ou Unix Socket (local)  
- ✅ Supporte l'authentification sécurisée (SCRAM-SHA-256)

### Cycle de Communication
1. **Connexion** : Établissement et authentification  
2. **Requête** : Envoi de commandes SQL  
3. **Réponse** : Réception des résultats  
4. **Fermeture** : Terminaison de la connexion

### Nouveauté PostgreSQL 18 (Protocol 3.2)
- ✅ Clés d'annulation 256 bits (sécurité renforcée)  
- ✅ Rétrocompatibilité complète avec les anciens clients

---

## 🎯 Points à Retenir pour les Débutants

1. **Vous n'avez pas besoin de maîtriser le Wire Protocol** pour utiliser PostgreSQL au quotidien. Les drivers et outils s'en occupent pour vous.

2. **Comprendre le modèle client-serveur** vous aide à :
   - Mieux concevoir vos applications
   - Diagnostiquer les problèmes de connexion
   - Optimiser les performances réseau

3. **Le Wire Protocol est transparent** : vous écrivez du SQL, le driver gère la communication.

4. **La sécurité commence à la connexion** : utilisez toujours des méthodes d'authentification sécurisées (SCRAM-SHA-256) et SSL/TLS pour les connexions distantes.

5. **La performance dépend du réseau** : privilégiez les connexions locales (socket Unix) quand c'est possible.

---

## 🔗 Prochaine Étape

Maintenant que vous comprenez **comment** client et serveur communiquent, la section suivante explorera **l'architecture interne de PostgreSQL** : les processus, la gestion mémoire, et le stockage physique des données.


⏭️ [Anatomie d'une instance : Postmaster et processus d'arrière-plan](/03-architecture-de-postgresql/02-anatomie-dune-instance.md)
