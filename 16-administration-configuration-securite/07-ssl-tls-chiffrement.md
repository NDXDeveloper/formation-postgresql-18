🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.7. SSL/TLS et Chiffrement des Connexions

## Introduction

Jusqu'à présent, nous avons sécurisé **qui** peut se connecter (authentification) et **ce que** les utilisateurs peuvent faire (autorisation, RLS). Mais il reste une question cruciale de sécurité : **comment protéger les données pendant leur transmission** entre le client et le serveur PostgreSQL ?

Par défaut, les connexions PostgreSQL transmettent les données **en clair** sur le réseau. Cela signifie qu'un attaquant qui intercepte le trafic peut lire :
- Les mots de passe
- Les requêtes SQL
- Les résultats des requêtes (données sensibles)

**SSL/TLS** (Secure Sockets Layer / Transport Layer Security) est la solution standard pour **chiffrer** les communications réseau et garantir que personne ne peut espionner vos données en transit.

Cette section explique ce qu'est SSL/TLS, pourquoi c'est essentiel, et comment le configurer dans PostgreSQL.

---

## 🔐 Qu'est-ce que SSL/TLS ?

### Définition Simple

**SSL** (Secure Sockets Layer) et son successeur **TLS** (Transport Layer Security) sont des **protocoles de chiffrement** qui sécurisent les communications sur un réseau.

**En termes simples :**
> "SSL/TLS crée un tunnel sécurisé entre deux ordinateurs, rendant les données illisibles pour quiconque essaie de les intercepter."

### Analogie du Monde Réel

Imaginez deux personnes qui veulent communiquer :

**Sans SSL/TLS (communication en clair) :**
```
Alice → "Mon mot de passe est 123456" → Bob
         ↑ (visible par tous)
    Un espion peut lire ce message !
```

**Avec SSL/TLS (communication chiffrée) :**
```
Alice → "a8f$2#kL9@mP0..." (chiffré) → Bob
         ↑ (incompréhensible pour les espions)
    Un espion voit du charabia illisible !
```

C'est comme envoyer une lettre :
- **Sans SSL/TLS** : Carte postale (tout le monde peut lire)
- **Avec SSL/TLS** : Enveloppe scellée dans un coffre-fort (seul le destinataire peut ouvrir)

### SSL vs TLS : Quelle Différence ?

**Histoire :**
- **SSL** : Protocole original (années 1990)
- **TLS** : Version améliorée et plus sécurisée de SSL (1999+)

**Aujourd'hui :**
- SSL est **obsolète et dangereux** (SSL 2.0 et 3.0 ont des failles critiques)
- **TLS est le standard actuel** (versions TLS 1.2 et TLS 1.3)

⚠️ **Important** : Même si on dit souvent "SSL", on parle en réalité de **TLS**. PostgreSQL supporte TLS, pas SSL.

**Versions TLS :**
- ❌ **TLS 1.0 et 1.1** : Dépréciés, non sécurisés
- ✅ **TLS 1.2** : Standard actuel, sécurisé
- ✅ **TLS 1.3** : Version la plus récente, plus rapide et plus sûre (PostgreSQL 18+)

---

## 🚨 Pourquoi le Chiffrement des Connexions est Crucial

### Les Dangers d'une Connexion Non Chiffrée

#### Danger #1 : Interception du Mot de Passe

**Scénario :**
```sql
-- Connexion sans SSL
psql -h database.example.com -U alice -d production
Password: MonMotDePasse123

-- ⚠️ Le mot de passe transite EN CLAIR sur le réseau
-- Un attaquant avec Wireshark peut le capturer en 5 secondes
```

**Conséquence :** L'attaquant peut se connecter avec vos identifiants.

#### Danger #2 : Vol de Données Sensibles

**Scénario :**
```sql
-- Requête pour récupérer des données confidentielles
SELECT nom, prenom, salaire, numero_secu
FROM employes;

-- ⚠️ Toutes ces données transitent EN CLAIR
-- Un attaquant peut capturer :
-- - 1000 numéros de sécurité sociale
-- - Tous les salaires de l'entreprise
```

**Conséquence :** Violation de la vie privée, non-conformité RGPD.

#### Danger #3 : Injection de Requêtes (Man-in-the-Middle)

**Scénario :**
```
Client → Requête SQL → [ATTAQUANT] → Serveur
                           ↓
                   Modifie la requête
```

**Exemple :**
```sql
-- Ce que le client envoie :
SELECT * FROM users WHERE id = 1;

-- Ce que l'attaquant modifie :
SELECT * FROM users WHERE id = 1; DROP TABLE users;--

-- ⚠️ L'attaquant peut exécuter du code malveillant
```

**Conséquence :** Destruction de données, vol, sabotage.

### Où les Attaques Peuvent-elles Avoir Lieu ?

**Réseaux vulnérables :**
- ❌ **Wi-Fi publics** (café, aéroport, hôtel)
- ❌ **Réseaux partagés** (coworking, université)
- ❌ **Internet ouvert** (connexion depuis l'extérieur)
- ⚠️ **Réseaux d'entreprise** (si compromis)

**Même les "réseaux internes" peuvent être vulnérables :**
- Un employé malveillant
- Un ordinateur infecté par un malware
- Un attaquant ayant pénétré le réseau

**Principe de sécurité :**
> "Never trust the network" (Ne jamais faire confiance au réseau)

### Obligations Légales et Conformité

De nombreuses réglementations **imposent** le chiffrement des données :

| Réglementation | Exigence |
|----------------|----------|
| **RGPD** (Europe) | Chiffrement des données personnelles en transit |
| **HIPAA** (USA - Santé) | Chiffrement obligatoire pour données médicales |
| **PCI-DSS** (Cartes de crédit) | Chiffrement fort (TLS 1.2+) obligatoire |
| **SOC 2** (Audit) | Chiffrement des connexions bases de données |
| **ISO 27001** | Protection des données en transit |

⚠️ **Non-conformité = amendes + sanctions + perte de réputation**

---

## 🔧 Comment Fonctionne SSL/TLS ?

### Les Trois Piliers de SSL/TLS

SSL/TLS garantit trois choses essentielles :

#### 1. Confidentialité (Chiffrement)

**Principe :** Les données sont **chiffrées** et illisibles pour les espions.

**Fonctionnement :**
```
Données originales : "SELECT * FROM users"
        ↓ (chiffrement)
Données chiffrées : "a8f2K9mL@P0x7..."
        ↓ (transmission)
Réseau (espions voient du charabia)
        ↓ (réception)
Données chiffrées : "a8f2K9mL@P0x7..."
        ↓ (déchiffrement)
Données originales : "SELECT * FROM users"
```

#### 2. Intégrité (Détection de Modification)

**Principe :** Garantit que les données n'ont **pas été modifiées** en transit.

**Fonctionnement :**
- Un "sceau numérique" (HMAC) est ajouté à chaque message
- Si un attaquant modifie le message, le sceau ne correspond plus
- La connexion est immédiatement interrompue

**Exemple :**
```
Message : "SELECT * FROM users"
Sceau   : "abc123def456"  (calculé sur le message)

→ Transmission sur le réseau

Si l'attaquant modifie en : "DROP TABLE users"
→ Le sceau "abc123def456" ne correspond plus !
→ PostgreSQL détecte la manipulation
→ Connexion fermée
```

#### 3. Authentification (Vérification d'Identité)

**Principe :** Garantit que vous communiquez bien avec le **bon serveur**.

**Problème sans authentification :**
```
Client → pense se connecter à database.example.com
         ↓
    [Serveur IMPOSTEUR]
         ↓
    Vole les identifiants !
```

**Solution avec certificats SSL :**
```
Client → "Prouve que tu es database.example.com"
         ↓
Serveur → [Présente certificat signé par autorité de confiance]
         ↓
Client → ✅ Vérifie le certificat
         ↓
    Connexion sécurisée établie
```

### Flux de Connexion SSL/TLS (Handshake)

**Étapes simplifiées :**

```
1. Client → "Bonjour, je veux une connexion TLS 1.3"
            ↓
2. Serveur → "OK, voici mon certificat" + [Certificat]
            ↓
3. Client → Vérifie le certificat
            - Est-il valide ?
            - N'est-il pas expiré ?
            - Est-il signé par une autorité de confiance ?
            ↓
4. Client & Serveur → Négocient une clé de chiffrement secrète
            ↓
5. 🔒 Connexion chiffrée établie
            ↓
6. Transmission sécurisée des données
```

**Durée :** Moins d'une seconde pour établir la connexion.

---

## 🛠️ Certificats SSL/TLS : Comprendre les Bases

### Qu'est-ce qu'un Certificat ?

Un **certificat SSL/TLS** est comme une **carte d'identité numérique** pour un serveur. Il prouve que le serveur est bien celui qu'il prétend être.

**Contenu d'un certificat :**
- **Nom du serveur** (ex: database.example.com)
- **Clé publique** (pour le chiffrement)
- **Signature numérique** (prouve l'authenticité)
- **Émetteur** (qui a signé ce certificat)
- **Dates de validité** (début et fin)

### Types de Certificats

#### 1. Certificat Auto-Signé (Self-Signed)

**Définition :** Certificat créé et signé par vous-même, pas par une autorité externe.

**Avantages :**
- ✅ Gratuit
- ✅ Facile à créer
- ✅ Parfait pour développement et réseaux internes

**Inconvénients :**
- ❌ Pas reconnu par défaut (avertissements de sécurité)
- ❌ Aucune garantie pour les tiers

**Quand l'utiliser :**
- Développement local
- Réseaux privés internes
- Tests

**Exemple de création :**
```bash
# Créer un certificat auto-signé avec OpenSSL
openssl req -new -x509 -days 365 -nodes \
  -out server.crt \
  -keyout server.key \
  -subj "/CN=database.example.com"
```

#### 2. Certificat Signé par une Autorité de Certification (CA)

**Définition :** Certificat émis et signé par une **autorité de confiance** (CA - Certificate Authority).

**Autorités connues :**
- Let's Encrypt (gratuit)
- DigiCert
- GlobalSign
- Comodo

**Avantages :**
- ✅ Reconnu universellement
- ✅ Aucun avertissement de sécurité
- ✅ Preuve d'identité vérifiée

**Inconvénients :**
- ⚠️ Peut être payant (sauf Let's Encrypt)
- ⚠️ Processus de validation requis

**Quand l'utiliser :**
- Production
- Services accessibles depuis Internet
- Conformité réglementaire

#### 3. Certificat Client (Client Certificate)

**Définition :** Certificat installé côté **client** pour prouver l'identité du client au serveur.

**Usage :**
- Authentification mutuelle (mTLS)
- Sécurité renforcée (au lieu de mot de passe)

**Exemple de flux :**
```
Client → [Présente certificat client]
         ↓
Serveur → Vérifie le certificat
         ✅ "Ce client est bien alice@example.com"
         ↓
    Connexion autorisée (sans mot de passe)
```

---

## ⚙️ Configuration SSL/TLS dans PostgreSQL

### Côté Serveur : Activer SSL

#### Étape 1 : Générer ou Obtenir des Certificats

**Option A : Certificat auto-signé (développement)**
```bash
# Générer une clé privée
openssl genpkey -algorithm RSA -out server.key -pkeyopt rsa_keygen_bits:2048

# Générer le certificat auto-signé
openssl req -new -x509 -key server.key -out server.crt -days 365 \
  -subj "/C=FR/ST=Normandy/L=Rouen/O=MyCompany/CN=postgres.mycompany.com"

# Définir les permissions (IMPORTANT)
chmod 600 server.key
chown postgres:postgres server.key server.crt
```

**Option B : Certificat Let's Encrypt (production)**
```bash
# Installer Certbot
apt-get install certbot

# Obtenir un certificat (nécessite un domaine public)
certbot certonly --standalone -d database.example.com

# Les certificats sont dans /etc/letsencrypt/live/database.example.com/
# - fullchain.pem (certificat)
# - privkey.pem (clé privée)
```

#### Étape 2 : Placer les Certificats

```bash
# Copier les certificats dans le répertoire de données PostgreSQL
cp server.crt /var/lib/postgresql/18/main/
cp server.key /var/lib/postgresql/18/main/

# Permissions strictes (PostgreSQL refusera de démarrer si trop permissif)
chmod 600 /var/lib/postgresql/18/main/server.key
chown postgres:postgres /var/lib/postgresql/18/main/server.*
```

#### Étape 3 : Configurer `postgresql.conf`

```
# postgresql.conf

# Activer SSL
ssl = on

# Certificat du serveur
ssl_cert_file = 'server.crt'

# Clé privée du serveur
ssl_key_file = 'server.key'

# Optionnel : Autorité de certification (pour vérifier les certificats clients)
# ssl_ca_file = 'root.crt'

# Optionnel : Liste de révocation de certificats
# ssl_crl_file = 'root.crl'

# Versions TLS autorisées (sécurité renforcée)
ssl_min_protocol_version = 'TLSv1.2'  # Minimum TLS 1.2
ssl_max_protocol_version = 'TLSv1.3'  # Maximum TLS 1.3 (PostgreSQL 18+)

# Nouveauté PostgreSQL 18 : Configuration TLS 1.3
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256'

# Algorithmes de chiffrement autorisés (TLS 1.2)
ssl_ciphers = 'HIGH:!aNULL:!MD5'

# Préférer les algorithmes du serveur
ssl_prefer_server_ciphers = on
```

**Explication des paramètres :**

| Paramètre | Description | Valeur recommandée |
|-----------|-------------|-------------------|
| `ssl` | Active/Désactive SSL | `on` |
| `ssl_cert_file` | Chemin du certificat serveur | `server.crt` |
| `ssl_key_file` | Chemin de la clé privée | `server.key` |
| `ssl_min_protocol_version` | Version TLS minimale | `TLSv1.2` |
| `ssl_max_protocol_version` | Version TLS maximale | `TLSv1.3` |
| `ssl_ciphers` | Algorithmes de chiffrement | `HIGH:!aNULL:!MD5` |
| `ssl_prefer_server_ciphers` | Préférence serveur | `on` |

#### Étape 4 : Configurer `pg_hba.conf`

**Forcer SSL pour certaines connexions :**

```
# pg_hba.conf

# TYPE  DATABASE  USER      ADDRESS          METHOD

# Connexions locales : pas de SSL nécessaire
local   all       all                        scram-sha-256

# Connexions externes : SSL OBLIGATOIRE
hostssl all       all       0.0.0.0/0        scram-sha-256

# Connexions externes sans SSL : REFUSÉES
# (Commentez ou supprimez les lignes "host" non-SSL)
# host  all       all       0.0.0.0/0        scram-sha-256  ❌
```

**Différence host vs hostssl :**
- `host` : Autorise les connexions avec OU sans SSL
- `hostssl` : **Exige** une connexion SSL (refuse si pas SSL)
- `hostnossl` : Autorise UNIQUEMENT les connexions sans SSL (⚠️ dangereux)

#### Étape 5 : Redémarrer PostgreSQL

```bash
# Redémarrer pour appliquer les changements
sudo systemctl restart postgresql

# Vérifier que SSL est activé
sudo -u postgres psql -c "SHOW ssl;"
# Résultat attendu : on
```

### Côté Client : Se Connecter avec SSL

#### Méthode 1 : psql avec SSL

```bash
# Connexion simple avec SSL
psql "host=database.example.com port=5432 dbname=mydb user=alice sslmode=require"

# Ou avec variables d'environnement
export PGSSLMODE=require
psql -h database.example.com -U alice -d mydb

# Vérifier que la connexion utilise SSL
psql -h database.example.com -U alice -d mydb -c "SELECT ssl_is_used();"
# Résultat : true
```

#### Méthode 2 : URI de Connexion

```bash
# Format URI complet
psql "postgresql://alice@database.example.com:5432/mydb?sslmode=require"
```

#### Méthode 3 : Applications avec Drivers

**Python (psycopg3) :**
```python
import psycopg

# Connexion avec SSL
conn = psycopg.connect(
    "host=database.example.com port=5432 dbname=mydb user=alice password=pwd sslmode=require"
)

# Ou avec paramètres
conn = psycopg.connect(
    host="database.example.com",
    port=5432,
    dbname="mydb",
    user="alice",
    password="pwd",
    sslmode="require"
)
```

**Node.js (node-postgres) :**
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'database.example.com',
  port: 5432,
  database: 'mydb',
  user: 'alice',
  password: 'pwd',
  ssl: {
    rejectUnauthorized: true  // Vérifier le certificat
  }
});
```

**Java (JDBC) :**
```java
String url = "jdbc:postgresql://database.example.com:5432/mydb"
           + "?ssl=true&sslmode=require";
Connection conn = DriverManager.getConnection(url, "alice", "pwd");
```

---

## 🔒 Modes de Connexion SSL

PostgreSQL offre plusieurs **modes SSL** qui définissent le niveau de sécurité de la connexion.

### Tableau des Modes SSL

| Mode | Chiffrement | Vérification Certificat | Sécurité | Usage |
|------|-------------|------------------------|----------|-------|
| `disable` | ❌ Non | ❌ Non | ⚠️ Aucune | Développement local uniquement |
| `allow` | ⚠️ Essaie | ❌ Non | ⚠️ Faible | Déprécié, éviter |
| `prefer` | ✅ Oui si possible | ❌ Non | ⚠️ Moyenne | Par défaut (⚠️ non recommandé) |
| `require` | ✅ Oui (obligatoire) | ❌ Non | ⚠️ Moyenne | Minimum recommandé |
| `verify-ca` | ✅ Oui | ✅ Vérifie CA | ✅ Bonne | Recommandé |
| `verify-full` | ✅ Oui | ✅ Vérifie CA + nom | ✅ Maximale | **Meilleur choix** |

### Détail des Modes

#### `disable` : Aucun SSL

```bash
psql "host=db.example.com dbname=mydb user=alice sslmode=disable"
```

**Comportement :**
- ❌ Aucun chiffrement
- ❌ Données en clair sur le réseau

**Quand l'utiliser :**
- Développement local (localhost)
- Jamais en production !

#### `prefer` : SSL si Disponible (Par Défaut)

```bash
psql "host=db.example.com dbname=mydb user=alice sslmode=prefer"
# ou simplement (prefer est le défaut)
psql "host=db.example.com dbname=mydb user=alice"
```

**Comportement :**
- Essaie de se connecter avec SSL
- Si le serveur ne supporte pas SSL → connexion sans SSL (⚠️ danger)

**Problème :**
- Vulnérable aux attaques de dégradation (downgrade attack)
- Un attaquant peut forcer une connexion non-SSL

⚠️ **Non recommandé en production**

#### `require` : SSL Obligatoire

```bash
psql "host=db.example.com dbname=mydb user=alice sslmode=require"
```

**Comportement :**
- ✅ Exige une connexion SSL
- ✅ Refuse si le serveur ne supporte pas SSL
- ⚠️ N'vérifie PAS le certificat du serveur

**Problème :**
- Vulnérable aux attaques Man-in-the-Middle
- Un imposteur peut présenter n'importe quel certificat

**Quand l'utiliser :**
- Minimum acceptable en production
- Si vous ne pouvez pas vérifier les certificats

#### `verify-ca` : Vérifier l'Autorité de Certification

```bash
psql "host=db.example.com dbname=mydb user=alice sslmode=verify-ca sslrootcert=/path/to/ca.crt"
```

**Comportement :**
- ✅ Exige une connexion SSL
- ✅ Vérifie que le certificat est signé par une CA de confiance
- ⚠️ N'vérifie PAS que le nom du serveur correspond

**Protection :**
- Protège contre les certificats auto-signés non autorisés
- Pas de protection contre un serveur légitime détourné

**Quand l'utiliser :**
- Production avec certificats d'entreprise
- Bonne sécurité mais pas parfaite

#### `verify-full` : Vérification Complète (Recommandé)

```bash
psql "host=db.example.com dbname=mydb user=alice sslmode=verify-full sslrootcert=/path/to/ca.crt"
```

**Comportement :**
- ✅ Exige une connexion SSL
- ✅ Vérifie que le certificat est signé par une CA de confiance
- ✅ Vérifie que le nom du serveur correspond au certificat

**Protection maximale :**
- Protège contre Man-in-the-Middle
- Protège contre les certificats volés
- Protège contre les serveurs imposteurs

**Quand l'utiliser :**
- **Toujours en production** si possible
- Standard de sécurité actuel

**Exemple complet :**
```bash
# Connexion ultra-sécurisée
psql "host=database.example.com \
      port=5432 \
      dbname=production \
      user=app_user \
      sslmode=verify-full \
      sslrootcert=/etc/ssl/certs/ca-bundle.crt"
```

### Fichiers de Certificats Clients

| Paramètre | Fichier | Description |
|-----------|---------|-------------|
| `sslrootcert` | `~/.postgresql/root.crt` | Certificat CA racine (pour vérifier le serveur) |
| `sslcert` | `~/.postgresql/postgresql.crt` | Certificat client (authentification mTLS) |
| `sslkey` | `~/.postgresql/postgresql.key` | Clé privée client |

---

## 🆕 Nouveautés PostgreSQL 18 : SSL/TLS

### 1. Mode FIPS (Federal Information Processing Standards)

**Nouveauté PostgreSQL 18 :** Support du mode FIPS pour les environnements hautement régulés.

**Qu'est-ce que FIPS ?**
- Standard de sécurité cryptographique du gouvernement américain
- Exigé pour les systèmes gouvernementaux et militaires
- Utilise uniquement des algorithmes certifiés FIPS

**Configuration :**
```
# postgresql.conf (PostgreSQL 18+)
ssl_fips = on  # Active le mode FIPS

# Limite les algorithmes à ceux certifiés FIPS
ssl_ciphers = 'FIPS'
```

**Quand l'utiliser :**
- Contrats gouvernementaux US
- Secteurs hautement régulés (défense, santé)
- Conformité stricte requise

### 2. Configuration TLS 1.3 Avancée

**Nouveauté PostgreSQL 18 :** Paramètres dédiés pour TLS 1.3.

```
# postgresql.conf (PostgreSQL 18+)

# Activer TLS 1.3 uniquement
ssl_min_protocol_version = 'TLSv1.3'
ssl_max_protocol_version = 'TLSv1.3'

# Algorithmes de chiffrement TLS 1.3
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256'
```

**Avantages de TLS 1.3 :**
- ✅ **Plus rapide** : Handshake plus court (1-RTT)
- ✅ **Plus sûr** : Algorithmes faibles supprimés
- ✅ **Confidentialité** : Forward secrecy par défaut
- ✅ **Simplicité** : Configuration plus simple

**Algorithmes TLS 1.3 recommandés :**
```
# Ordre de préférence (du plus fort au plus rapide)
TLS_AES_256_GCM_SHA384        # AES 256-bit (le plus fort)
TLS_AES_128_GCM_SHA256        # AES 128-bit (bon compromis)
TLS_CHACHA20_POLY1305_SHA256  # ChaCha20 (rapide sur mobile)
```

### 3. Authentification Mutuelle Améliorée (mTLS)

**Amélioration :** Meilleure intégration avec les certificats clients.

**Configuration :**
```
# pg_hba.conf (PostgreSQL 18)

# Authentification par certificat client uniquement
hostssl  all  all  0.0.0.0/0  cert

# Certificat client + mot de passe (double facteur)
hostssl  all  all  0.0.0.0/0  cert clientcert=verify-full scram-sha-256
```

---

## 🔍 Vérification et Diagnostic

### Vérifier que SSL est Activé

**Côté serveur :**
```sql
-- Vérifier que SSL est activé sur le serveur
SHOW ssl;
-- Résultat attendu : on

-- Voir les paramètres SSL
SHOW ssl_cert_file;
SHOW ssl_key_file;
SHOW ssl_min_protocol_version;
```

**Côté client (connexion active) :**
```sql
-- Vérifier si la connexion actuelle utilise SSL
SELECT ssl_is_used();
-- Résultat : true

-- Voir les détails de la connexion SSL
SELECT * FROM pg_stat_ssl WHERE pid = pg_backend_pid();
-- Affiche : version TLS, cipher, bits, compression, etc.

-- Version plus détaillée
SELECT
  pid,
  ssl,
  version AS tls_version,
  cipher,
  bits,
  client_dn
FROM pg_stat_ssl;
```

### Tester la Connexion SSL avec OpenSSL

```bash
# Tester la connexion SSL au serveur PostgreSQL
openssl s_client -connect database.example.com:5432 -starttls postgres

# Vérifier le certificat
echo | openssl s_client -connect database.example.com:5432 -starttls postgres 2>/dev/null | openssl x509 -noout -text
```

### Logs PostgreSQL pour SSL

**Activer les logs SSL :**
```
# postgresql.conf
log_connections = on
log_disconnections = on

# Logs détaillés (développement uniquement)
log_statement = 'all'
```

**Exemple de logs :**
```
LOG:  connection received: host=192.168.1.100 port=54321
LOG:  connection authorized: user=alice database=mydb SSL enabled (protocol=TLSv1.3, cipher=TLS_AES_256_GCM_SHA384, bits=256)
```

---

## ⚠️ Erreurs Courantes et Solutions

### Erreur #1 : "server does not support SSL"

**Message :**
```
psql: error: connection to server failed:
server does not support SSL, but SSL was required
```

**Causes :**
- SSL n'est pas activé sur le serveur (`ssl = off`)
- `pg_hba.conf` utilise `hostnossl` au lieu de `hostssl`

**Solution :**
```
# postgresql.conf
ssl = on

# pg_hba.conf
hostssl  all  all  0.0.0.0/0  scram-sha-256
```

### Erreur #2 : Permission Denied sur la Clé Privée

**Message :**
```
FATAL: could not load server certificate file "server.crt": Permission denied
FATAL: private key file "server.key" has group or world access
```

**Cause :**
- Permissions trop permissives sur `server.key`

**Solution :**
```bash
# Permissions strictes requises
chmod 600 server.key
chown postgres:postgres server.key

# Vérifier
ls -la server.key
# Résultat attendu : -rw------- postgres postgres
```

### Erreur #3 : Certificat Auto-Signé Rejeté

**Message :**
```
psql: error: connection failed:
SSL error: certificate verify failed
```

**Cause :**
- Mode `verify-ca` ou `verify-full` avec certificat auto-signé non ajouté aux certificats de confiance

**Solution 1 : Ajouter le certificat aux certificats de confiance**
```bash
# Copier le certificat serveur dans les certificats de confiance
cp server.crt ~/.postgresql/root.crt

# Connexion avec vérification
psql "host=db.local dbname=mydb user=alice sslmode=verify-full sslrootcert=~/.postgresql/root.crt"
```

**Solution 2 : Utiliser `require` au lieu de `verify-full` (moins sécurisé)**
```bash
psql "host=db.local dbname=mydb user=alice sslmode=require"
```

### Erreur #4 : Certificat Expiré

**Message :**
```
SSL error: certificate has expired
```

**Solution :**
```bash
# Vérifier la date d'expiration
openssl x509 -in server.crt -noout -dates

# Générer un nouveau certificat
openssl req -new -x509 -days 365 -nodes \
  -out server.crt \
  -keyout server.key \
  -subj "/CN=database.example.com"

# Redémarrer PostgreSQL
sudo systemctl restart postgresql
```

### Erreur #5 : Nom d'Hôte ne Correspond Pas

**Message :**
```
psql: error: server certificate does not match host name
```

**Cause :**
- Connexion à `192.168.1.50` mais certificat pour `database.example.com`
- Mode `verify-full` vérifie la correspondance

**Solution 1 : Se connecter avec le nom du certificat**
```bash
# Mauvais
psql "host=192.168.1.50 sslmode=verify-full ..."

# Bon
psql "host=database.example.com sslmode=verify-full ..."
```

**Solution 2 : Créer un certificat avec SAN (Subject Alternative Names)**
```bash
# Créer un fichier de configuration OpenSSL
cat > san.cnf <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]
CN = database.example.com

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = database.example.com
DNS.2 = db.local
IP.1 = 192.168.1.50
EOF

# Générer le certificat avec SAN
openssl req -new -x509 -days 365 -nodes \
  -config san.cnf \
  -out server.crt \
  -keyout server.key
```

---

## 🎯 Bonnes Pratiques

### 1. Utiliser TLS 1.2+ Uniquement

```
# postgresql.conf
ssl_min_protocol_version = 'TLSv1.2'  # Minimum
ssl_max_protocol_version = 'TLSv1.3'  # Maximum (PostgreSQL 18+)
```

❌ **Ne jamais autoriser :**
- SSL 2.0, SSL 3.0 (failles critiques)
- TLS 1.0, TLS 1.1 (dépréciés, non sécurisés)

### 2. Utiliser des Algorithmes de Chiffrement Forts

```
# postgresql.conf

# Algorithmes recommandés (TLS 1.2)
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256'

# Algorithmes recommandés (TLS 1.3) - PostgreSQL 18+
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256'
```

❌ **Éviter :**
- MD5, RC4, DES, 3DES (faibles)
- Algorithmes NULL (aucun chiffrement)
- Export ciphers (très faibles)

### 3. Forcer SSL en Production

```
# pg_hba.conf - PRODUCTION

# ❌ MAUVAIS : Permet les connexions non-SSL
host    all    all    0.0.0.0/0    scram-sha-256

# ✅ BON : Exige SSL
hostssl all    all    0.0.0.0/0    scram-sha-256
```

### 4. Utiliser verify-full Côté Client

```python
# Python - Connexion ultra-sécurisée
conn = psycopg.connect(
    host="database.example.com",
    dbname="production",
    user="app_user",
    password="secure_password",
    sslmode="verify-full",
    sslrootcert="/etc/ssl/certs/ca-bundle.crt"
)
```

### 5. Renouveler les Certificats Régulièrement

**Cycle de vie recommandé :**
- **Let's Encrypt** : Renouvellement automatique tous les 90 jours
- **Certificats d'entreprise** : Renouvellement annuel
- **Certificats auto-signés** : Renouvellement tous les 6-12 mois

**Automation avec Let's Encrypt :**
```bash
# Cron job pour renouvellement automatique
0 0 * * * certbot renew --quiet --deploy-hook "systemctl reload postgresql"
```

### 6. Monitorer les Connexions SSL

```sql
-- Voir toutes les connexions SSL actives
SELECT
  usename,
  application_name,
  client_addr,
  ssl,
  version AS tls_version,
  cipher
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE ssl = true;

-- Alerter si connexions non-SSL détectées
SELECT COUNT(*) AS non_ssl_connections
FROM pg_stat_activity
WHERE client_addr IS NOT NULL
AND pid NOT IN (SELECT pid FROM pg_stat_ssl WHERE ssl = true);
```

### 7. Documentation et Procédures

**Documenter :**
- Localisation des certificats
- Procédure de renouvellement
- Contacts pour les certificats
- Dates d'expiration
- Procédure de récupération en cas de problème

---

## 🧠 Points Clés à Retenir

1. **SSL/TLS chiffre les communications** entre client et serveur PostgreSQL
2. **Trois piliers** : Confidentialité (chiffrement) + Intégrité (détection modification) + Authentification (vérification identité)
3. **TLS 1.2+ uniquement** : Désactiver TLS 1.0, 1.1 et tous les SSL
4. **PostgreSQL 18** apporte TLS 1.3, mode FIPS, et configuration avancée
5. **Modes SSL** : `verify-full` est le plus sûr pour la production
6. **Certificats** : Auto-signés (dev) vs Signés CA (production)
7. **Configuration double** : Serveur (`postgresql.conf` + certificats) + Client (`sslmode`)
8. **Forcer SSL** : Utiliser `hostssl` dans `pg_hba.conf`

---

## 📊 Checklist de Configuration SSL

### Configuration Serveur

```
☐ Générer ou obtenir des certificats SSL
☐ Placer server.crt et server.key dans le data directory
☐ Définir permissions 600 sur server.key
☐ Configurer postgresql.conf :
  ☐ ssl = on
  ☐ ssl_cert_file = 'server.crt'
  ☐ ssl_key_file = 'server.key'
  ☐ ssl_min_protocol_version = 'TLSv1.2'
  ☐ ssl_ciphers = 'HIGH:!aNULL:!MD5'
☐ Configurer pg_hba.conf (utiliser hostssl)
☐ Redémarrer PostgreSQL
☐ Vérifier : SHOW ssl; (résultat : on)
☐ Tester connexion SSL avec psql
```

### Configuration Client

```
☐ Définir sslmode (minimum 'require', idéal 'verify-full')
☐ Si verify-full : Obtenir le certificat CA racine
☐ Placer le certificat CA dans ~/.postgresql/root.crt
☐ Tester la connexion
☐ Vérifier avec SELECT ssl_is_used();
☐ Documenter la configuration
```

### Maintenance Continue

```
☐ Surveiller les dates d'expiration des certificats
☐ Configurer le renouvellement automatique (Let's Encrypt)
☐ Auditer les connexions (SSL vs non-SSL)
☐ Vérifier les logs pour erreurs SSL
☐ Réviser les algorithmes de chiffrement annuellement
☐ Tester la récupération en cas de problème certificat
```

---

## 🚀 Pour Aller Plus Loin

### Sujets Connexes

Dans les sections suivantes du tutoriel :

- **16.2** : Configuration de l'authentification (`pg_hba.conf`)
- **16.8** : Nouveautés PostgreSQL 18 - OAuth 2.0
- **16.9** : Chiffrement au repos (Transparent Data Encryption - TDE)

### Concepts Avancés

**Authentification Mutuelle (mTLS) :**
- Client et serveur présentent tous deux un certificat
- Authentification forte sans mot de passe
- Standard dans les architectures zero-trust

**Rotation Automatique de Certificats :**
- Outils : cert-manager (Kubernetes), Vault (HashiCorp)
- Automatisation complète du cycle de vie

**Perfect Forward Secrecy (PFS) :**
- Même si la clé privée est compromise, les sessions passées restent sécurisées
- Activé par défaut avec TLS 1.3

---

## 📚 Ressources Complémentaires

- **Documentation PostgreSQL** : [SSL Support](https://www.postgresql.org/docs/current/ssl-tcp.html)
- **Documentation PostgreSQL** : [Secure TCP/IP Connections](https://www.postgresql.org/docs/current/runtime-config-connection.html#RUNTIME-CONFIG-CONNECTION-SSL)
- **Let's Encrypt** : [Certbot Documentation](https://certbot.eff.org/)
- **Mozilla SSL Configuration Generator** : [ssl-config.mozilla.org](https://ssl-config.mozilla.org/)
- **SSL Labs** : [Tester la configuration SSL/TLS](https://www.ssllabs.com/ssltest/)

---


⏭️ [Nouveauté PG 18 : Mode FIPS et configuration TLS 1.3 (ssl_tls13_ciphers)](/16-administration-configuration-securite/08-mode-fips-tls13-pg18.md)
