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
2. **Vous dites "Allô"** (salutation initiale)
3. **Vous posez votre question** (requête)
4. **Votre interlocuteur répond** (réponse)
5. **Vous dites "Au revoir"** (fermeture de connexion)

PostgreSQL fonctionne de manière similaire, mais de façon beaucoup plus structurée.

---

## 📡 Le Wire Protocol de PostgreSQL (Version 3.2)

### Qu'est-ce que le Wire Protocol ?

Le **Wire Protocol** (littéralement "protocole du câble") est le langage de communication standardisé entre un client PostgreSQL et le serveur. La version actuelle est **3.2**, introduite dans PostgreSQL 18.

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
- Le client envoie une **requête SQL** encapsulée dans un message de type "Query"
- Le message contient :
  - Le type de message (Query = 'Q')
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
3. **CommandComplete** : Le serveur signale la fin (ex: "SELECT 2" pour 2 lignes)
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
│  Type de Message (1 byte)           │  Ex: 'Q' pour Query, 'D' pour DataRow
├─────────────────────────────────────┤
│  Longueur du Message (4 bytes)      │  Taille totale en octets
├─────────────────────────────────────┤
│  Contenu du Message (variable)      │  Données spécifiques au message
└─────────────────────────────────────┘
```

### Exemple Concret : Message "Query"

```
Type:     'Q' (Query)
Longueur: 30 octets
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

---

## 🔐 Sécurité et Authentification dans le Wire Protocol

### Méthodes d'Authentification Supportées

Lors de l'établissement de la connexion, plusieurs méthodes d'authentification sont possibles :

1. **trust** : Pas d'authentification (dangereux, seulement pour dev local)
2. **password** : Mot de passe en clair (déprécié)
3. **md5** : Hash MD5 du mot de passe (déprécié)
4. **scram-sha-256** : Standard moderne sécurisé (recommandé)
5. **cert** : Certificat SSL/TLS client
6. **OAuth 2.0** : Nouveau dans PostgreSQL 18

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

**Avantage** : Le mot de passe n'est jamais transmis en clair sur le réseau.

---

## 🚀 Optimisations du Wire Protocol 3.2

PostgreSQL 18 a introduit plusieurs améliorations dans le Wire Protocol 3.2 :

### 1. **Compression des Données** (optionnelle)

Les résultats volumineux peuvent être compressés avant transmission :

```
Sans compression:    Client <────── [1 MB de données] ────── Serveur
Avec compression:    Client <────── [200 KB compressé] ───── Serveur
```

**Avantage** : Réduction de la bande passante réseau (utile pour les connexions lentes)

### 2. **Pipelining Amélioré**

Le client peut envoyer plusieurs requêtes sans attendre la réponse de la première :

```
TRADITIONNEL (séquentiel):
  Requête 1 ───> Attente ───> Réponse 1 ───> Requête 2 ───> Attente ───> Réponse 2

PIPELINÉ:
  Requête 1 ─┐
  Requête 2 ─┼──> Traitement concurrent ──> Réponse 1 + 2
  Requête 3 ─┘
```

**Avantage** : Réduction de la latence totale

### 3. **Messages d'Erreur Enrichis**

Les erreurs renvoient désormais plus d'informations structurées :

```
Ancien format:
  ERROR: column "username" does not exist

Nouveau format (Wire Protocol 3.2):
  ERROR: column "username" does not exist
  DETAIL: Perhaps you meant to reference column "user_name"
  HINT: Try using double quotes if the name contains special characters
  POSITION: 8 (dans la requête)
  FILE: parse_relation.c
  LINE: 3425
```

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
   - Requête préparée : 3 round-trips (Parse, Bind, Execute)
   - Transaction : +2 round-trips (BEGIN, COMMIT)

### Bonnes Pratiques pour Minimiser la Latence

- ✅ **Utiliser des requêtes préparées** (pour les requêtes répétées)
- ✅ **Regrouper les opérations** dans une transaction
- ✅ **Limiter les résultats** avec `LIMIT` / pagination
- ✅ **Utiliser le pipelining** si supporté par le driver
- ✅ **Connecter via socket Unix** pour les applications locales
- ✅ **Activer la compression** pour les gros volumes de données

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

### Optimisations PostgreSQL 18
- ✅ Compression des données
- ✅ Pipelining amélioré
- ✅ Messages d'erreur enrichis

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
