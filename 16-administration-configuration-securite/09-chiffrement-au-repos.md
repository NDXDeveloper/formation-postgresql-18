🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.9. Chiffrement au Repos (Transparent Data Encryption - TDE)

## Introduction

Nous avons vu dans les sections précédentes comment protéger les données **en transit** avec SSL/TLS : les données sont chiffrées pendant leur voyage sur le réseau entre le client et le serveur.

Mais que se passe-t-il quand les données sont **stockées sur le disque** ? Par défaut, PostgreSQL écrit les données **en clair** dans ses fichiers. Si quelqu'un vole le disque dur, copie les fichiers de la base, ou accède aux sauvegardes, il peut lire toutes les données sans aucune protection.

Le **chiffrement au repos** (Data at Rest Encryption) protège les données stockées sur le disque en les chiffrant. **TDE (Transparent Data Encryption)** est une forme spécifique de chiffrement au repos qui est "transparente" : l'application n'a pas besoin de savoir que les données sont chiffrées.

Cette section explore le chiffrement au repos, son importance, et les différentes solutions disponibles pour PostgreSQL.

---

## 🔒 Chiffrement en Transit vs Chiffrement au Repos

### Vue d'Ensemble

**Schéma conceptuel :**
```
┌─────────────┐                              ┌──────────────┐
│   Client    │ ─── Données en transit ──→   │  Serveur PG  │
│(Application)│       (SSL/TLS 🔐)           │              │
└─────────────┘                              └──────┬───────┘
                                                    │
                                                    ↓
                                             ┌─────────────┐
                                             │ Disque dur  │
                                             │ (Données    │
                                             │  au repos)  │
                                             └─────────────┘
```

### Chiffrement en Transit (Vu Précédemment)

**Protection :** Données pendant la **transmission** sur le réseau

**Menace :** Interception réseau (sniffing, man-in-the-middle)

**Solution :** SSL/TLS

**Exemple d'attaque sans protection :**
```
Attaquant sur le réseau → Wireshark → Capture les requêtes SQL en clair
                                    → Voit "SELECT * FROM credit_cards"
                                    → Vole tous les numéros de cartes
```

### Chiffrement au Repos (Cette Section)

**Protection :** Données **stockées** sur le disque

**Menaces :**
- Vol physique du disque dur ou du serveur
- Copie non autorisée des fichiers de données
- Vol de sauvegardes
- Accès aux disques jetés/recyclés
- Snapshot de VM ou de stockage cloud

**Solution :** TDE ou chiffrement de disque

**Exemple d'attaque sans protection :**
```
Attaquant vole le disque dur → Monte le disque sur son PC
                             → Accède à /var/lib/postgresql/18/main/
                             → Lit directement les fichiers de données
                             → Extrait toutes les informations sensibles
```

### Pourquoi les Deux Sont Nécessaires ?

**Analogie :** Coffre-fort dans un camion blindé

**Chiffrement en transit (SSL/TLS) :**
- C'est le **camion blindé** qui transporte le coffre
- Protège pendant le transport
- Mais une fois arrivé, le contenu est accessible

**Chiffrement au repos (TDE) :**
- C'est le **coffre-fort fermé à clé**
- Même si on vole le coffre, impossible de l'ouvrir sans la clé
- Protection permanente du contenu

**Les deux ensemble :**
> "Données protégées pendant le transport ET une fois stockées"

---

## 🎯 Qu'est-ce que TDE (Transparent Data Encryption) ?

### Définition

**TDE (Transparent Data Encryption)** est une technologie qui chiffre automatiquement les données :
- **Avant** de les écrire sur le disque
- **Après** les avoir lues depuis le disque

**"Transparent"** signifie :
- L'application n'a **rien à changer**
- L'utilisateur ne voit **aucune différence**
- Le chiffrement/déchiffrement est **automatique**

### Fonctionnement Simplifié

```
┌─────────────────────────────────────────────────────────┐
│              ÉCRITURE DE DONNÉES                        │
└─────────────────────────────────────────────────────────┘

Application → INSERT INTO users (nom, email)
              VALUES ('Alice', 'alice@example.com')
                            ↓
                    PostgreSQL (en mémoire)
                    Données en clair
                            ↓
                     [MODULE TDE]
                    Chiffrement automatique
                            ↓
                    Données chiffrées
                    "a8f2K9@mL#P0..."
                            ↓
                    Écriture sur disque
                            ↓
                    Fichiers chiffrés


┌─────────────────────────────────────────────────────────┐
│              LECTURE DE DONNÉES                         │
└─────────────────────────────────────────────────────────┘

Application → SELECT * FROM users WHERE nom = 'Alice'
                            ↓
                    Lecture depuis disque
                    Données chiffrées
                    "a8f2K9@mL#P0..."
                            ↓
                     [MODULE TDE]
                    Déchiffrement automatique
                            ↓
                    PostgreSQL (en mémoire)
                    Données en clair
                            ↓
Application ← Résultat : 'Alice', 'alice@example.com'
```

**Point clé :** En mémoire (RAM), les données sont **toujours en clair**. Seules les données sur disque sont chiffrées.

### TDE vs Chiffrement Applicatif

**Chiffrement Applicatif :**
```sql
-- L'application chiffre AVANT d'envoyer à PostgreSQL
INSERT INTO users (nom, email_chiffre)
VALUES ('Alice', encrypt('alice@example.com', 'ma_cle_secrete'));

-- L'application doit déchiffrer APRÈS avoir lu
SELECT nom, decrypt(email_chiffre, 'ma_cle_secrete')
FROM users;
```

**Problèmes :**
- ❌ Modification du code application nécessaire
- ❌ Impossible d'indexer ou de rechercher sur les données chiffrées
- ❌ Complexité accrue
- ❌ Gestion des clés par l'application

**TDE (Transparent) :**
```sql
-- Aucun changement dans l'application
INSERT INTO users (nom, email)
VALUES ('Alice', 'alice@example.com');

SELECT nom, email FROM users WHERE email LIKE '%@example.com';
-- Fonctionne normalement !
```

**Avantages :**
- ✅ Aucune modification de code
- ✅ Index fonctionnels
- ✅ Recherches normales
- ✅ Gestion centralisée des clés

---

## 🚨 Menaces Contre les Données au Repos

### Menace #1 : Vol Physique

**Scénario :** Un serveur est volé dans un datacenter mal sécurisé.

**Sans TDE :**
```
Voleur → Démonte le disque dur
       → Monte le disque sur son ordinateur
       → Lit /var/lib/postgresql/18/main/base/
       → Accède à toutes les tables
       → Extrait les données sensibles
```

**Avec TDE :**
```
Voleur → Démonte le disque dur
       → Monte le disque sur son ordinateur
       → Lit /var/lib/postgresql/18/main/base/
       → Voit uniquement des données chiffrées : "a8f2K9@mL..."
       → ❌ Impossible de déchiffrer sans la clé
```

### Menace #2 : Sauvegardes Non Protégées

**Scénario :** Les sauvegardes sont stockées sur un système de stockage avec accès non restreint.

**Sans TDE :**
```
Attaquant → Accède au serveur de sauvegarde
          → Télécharge pg_dump.sql ou sauvegarde physique
          → Restaure sur son propre serveur PostgreSQL
          → ✅ Accès complet à toutes les données
```

**Avec TDE :**
```
Attaquant → Accède au serveur de sauvegarde
          → Télécharge la sauvegarde chiffrée
          → Tente de restaurer
          → ❌ Impossible sans la clé de chiffrement
```

### Menace #3 : Disques Jetés ou Recyclés

**Scénario :** Un disque dur est remplacé et jeté sans destruction sécurisée.

**Sans TDE :**
```
Disque jeté → Récupéré dans une déchetterie
            → Données "supprimées" sont récupérables
            → Outils de récupération de données
            → ✅ Extraction des données sensibles
```

**Avec TDE :**
```
Disque jeté → Récupéré dans une déchetterie
            → Données chiffrées récupérables
            → ❌ Impossible de déchiffrer sans la clé
```

### Menace #4 : Accès Non Autorisé au Système de Fichiers

**Scénario :** Un administrateur système malveillant ou un attaquant qui a compromis le système d'exploitation.

**Sans TDE :**
```
Attaquant avec accès root → cp -r /var/lib/postgresql/18/main ~/stolen_data
                          → Lit directement les fichiers
                          → ✅ Accès complet
```

**Avec TDE :**
```
Attaquant avec accès root → cp -r /var/lib/postgresql/18/main ~/stolen_data
                          → Fichiers copiés mais chiffrés
                          → ❌ Impossible de lire sans la clé
```

### Menace #5 : Snapshots de VM

**Scénario :** Dans un environnement cloud/virtualisé, snapshots de VM conservés sans protection.

**Sans TDE :**
```
Administrateur cloud → Crée un snapshot de la VM
                     → Exporte le snapshot
                     → Monte le snapshot
                     → ✅ Accès à toutes les données PostgreSQL
```

**Avec TDE :**
```
Administrateur cloud → Crée un snapshot de la VM
                     → Exporte le snapshot
                     → Monte le snapshot
                     → ❌ Données chiffrées, inaccessibles sans clé
```

---

## 📊 État de TDE dans PostgreSQL

### Situation Actuelle (PostgreSQL 18)

**Important :** PostgreSQL **n'a pas de TDE natif intégré** dans le noyau principal.

**Historique :**
- TDE a été **proposé** et **discuté** pendant des années dans la communauté PostgreSQL
- Plusieurs implémentations ont été développées mais pas encore intégrées au core
- C'est une fonctionnalité **très demandée** mais techniquement complexe

**Pourquoi TDE n'est pas encore dans le core ?**
1. **Complexité technique** : Impacte beaucoup de composants (WAL, buffers, réplication)
2. **Performance** : Le chiffrement a un coût CPU
3. **Gestion des clés** : Architecture complexe requise
4. **Réplication** : Doit fonctionner avec la réplication logique et physique

**Bonne nouvelle :** Des solutions existent via :
- Extensions PostgreSQL tierces
- Chiffrement au niveau système de fichiers
- Chiffrement au niveau disque/volume
- Forks PostgreSQL avec TDE intégré

### Comparaison avec Autres SGBD

| SGBD | TDE Natif | Depuis |
|------|-----------|--------|
| **Oracle Database** | ✅ Oui | 2004 (10g) |
| **Microsoft SQL Server** | ✅ Oui | 2008 |
| **MySQL Enterprise** | ✅ Oui | 2015 (5.7.11) |
| **MariaDB** | ✅ Oui | 2016 (10.1.3) |
| **PostgreSQL** | ❌ Non (core) | N/A |

**PostgreSQL rattrape son retard** avec des solutions tierces de qualité et un TDE natif potentiellement dans les futures versions.

---

## 🛠️ Solutions de Chiffrement au Repos pour PostgreSQL

### Solution 1 : Chiffrement au Niveau Disque (LUKS)

**Principe :** Chiffrer le **disque ou la partition** entière où PostgreSQL stocke ses données.

#### Fonctionnement

```
┌──────────────────────────────────────────────┐
│         Système d'exploitation               │
│  ┌────────────────────────────────────┐      │
│  │       PostgreSQL                   │      │
│  │  (écrit en clair)                  │      │
│  └────────────┬───────────────────────┘      │
│               ↓                              │
│  ┌────────────────────────────────────┐      │
│  │   Système de fichiers (ext4, xfs)  │      │
│  └────────────┬───────────────────────┘      │
│               ↓                              │
│  ┌────────────────────────────────────┐      │
│  │   LUKS (chiffrement de disque)     │      │
│  │   Chiffre/déchiffre automatiquement│      │
│  └────────────┬───────────────────────┘      │
│               ↓                              │
│  ┌────────────────────────────────────┐      │
│  │   Disque physique (chiffré)        │      │
│  │   Données illisibles               │      │
│  └────────────────────────────────────┘      │
└──────────────────────────────────────────────┘
```

#### Avantages

- ✅ **Simple** : Pas de modification de PostgreSQL
- ✅ **Universel** : Protège tout (PostgreSQL + logs + config)
- ✅ **Mature** : LUKS est utilisé depuis des années
- ✅ **Performance** : Accélération matérielle (AES-NI)
- ✅ **Gratuit** : Intégré dans Linux

#### Inconvénients

- ❌ **Protection limitée** : Ne protège que si le serveur est éteint
- ❌ **Clé en mémoire** : Une fois démarré, les données sont accessibles
- ❌ **Pas de granularité** : Tout ou rien (impossible de chiffrer seulement certaines tables)
- ❌ **Sauvegardes** : Les sauvegardes logiques (pg_dump) ne sont pas chiffrées

#### Configuration (Linux avec LUKS)

**Création d'un volume chiffré :**
```bash
# Créer un volume chiffré avec LUKS
cryptsetup luksFormat /dev/sdb1

# Ouvrir le volume (demande la passphrase)
cryptsetup luksOpen /dev/sdb1 postgresql_encrypted

# Créer un système de fichiers
mkfs.ext4 /dev/mapper/postgresql_encrypted

# Monter le volume
mount /dev/mapper/postgresql_encrypted /var/lib/postgresql

# PostgreSQL utilisera automatiquement ce volume chiffré
```

**Automatisation au démarrage (avec gestion de clé) :**
```bash
# Stocker la clé dans un fichier sécurisé (attention à la sécurité !)
# Mieux : utiliser un HSM ou un service de gestion de clés
echo "ma_passphrase_ultra_secrete" > /root/.luks_key
chmod 600 /root/.luks_key

# Ajouter à /etc/crypttab pour ouverture automatique
echo "postgresql_encrypted /dev/sdb1 /root/.luks_key luks" >> /etc/crypttab
```

#### Quand Utiliser LUKS

**✅ Utilisez LUKS pour :**
- Protection contre le vol physique du serveur
- Compliance minimale requise
- Simplicité maximale

**❌ N'utilisez PAS LUKS seul pour :**
- Protection contre administrateurs malveillants (clé en mémoire)
- Sauvegardes chiffrées (nécessite chiffrement additionnel)
- Granularité au niveau table/colonne

### Solution 2 : Chiffrement au Niveau Système de Fichiers

**Exemples :** ZFS encryption, Btrfs encryption

**Similaire à LUKS** mais au niveau du système de fichiers.

**Avantages supplémentaires :**
- ✅ Chiffrement par dataset/filesystem (plus granulaire que LUKS)
- ✅ Snapshots chiffrés
- ✅ Meilleure intégration avec fonctionnalités avancées (compression, déduplication)

**Exemple avec ZFS :**
```bash
# Créer un pool ZFS chiffré
zfs create -o encryption=aes-256-gcm \
           -o keyformat=passphrase \
           -o keylocation=prompt \
           tank/postgresql

# Monter et utiliser
zfs mount tank/postgresql
# PostgreSQL peut maintenant utiliser /tank/postgresql
```

### Solution 3 : Extensions TDE Tierces

#### Cybertec PostgreSQL TDE

**Description :** Extension développée par Cybertec offrant TDE complet.

**Fonctionnalités :**
- ✅ Chiffrement transparent des données
- ✅ Chiffrement du WAL
- ✅ Gestion des clés intégrée
- ✅ Compatible avec réplication

**Limitation :**
- ⚠️ Extension commerciale (payante)
- ⚠️ Nécessite version patchée de PostgreSQL

**Site web :** https://www.cybertec-postgresql.com/en/products/cybertec-postgresql-transparent-data-encryption/

#### pgcrypto (Chiffrement au Niveau Colonne)

**Description :** Extension PostgreSQL officielle pour chiffrement applicatif.

**Important :** Ce n'est **PAS du TDE** (pas transparent), mais une option de chiffrement.

**Exemple :**
```sql
-- Activer l'extension
CREATE EXTENSION pgcrypto;

-- Chiffrer des données
INSERT INTO users (nom, email_chiffre)
VALUES ('Alice', pgp_sym_encrypt('alice@example.com', 'cle_secrete'));

-- Déchiffrer
SELECT nom, pgp_sym_decrypt(email_chiffre, 'cle_secret') AS email
FROM users;
```

**Limitations :**
- ❌ Pas transparent (modification du code)
- ❌ Pas d'index sur données chiffrées
- ❌ Performance réduite
- ❌ Complexité applicative

### Solution 4 : Forks PostgreSQL avec TDE Intégré

#### EnterpriseDB (EDB) Postgres Advanced Server

**Description :** Fork commercial de PostgreSQL avec TDE natif.

**Fonctionnalités :**
- ✅ TDE complètement intégré
- ✅ Chiffrement des données et du WAL
- ✅ Gestion avancée des clés
- ✅ Compatible avec tous les outils PostgreSQL

**Limitation :**
- ⚠️ Solution commerciale (licence payante)
- ⚠️ Pas du PostgreSQL "vanilla"

#### Autres Forks

- **Percona Distribution for PostgreSQL** : Envisage d'intégrer TDE
- **Amazon RDS PostgreSQL** : Chiffrement au repos via AWS KMS
- **Google Cloud SQL PostgreSQL** : Chiffrement automatique
- **Azure Database for PostgreSQL** : Chiffrement avec clés managées

### Solution 5 : Chiffrement Cloud-Native

#### AWS RDS PostgreSQL

**Fonctionnalité :** Encryption at rest avec AWS KMS (Key Management Service).

**Activation :**
```bash
# Créer une instance RDS avec chiffrement
aws rds create-db-instance \
  --db-instance-identifier my-encrypted-postgres \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --master-username admin \
  --master-user-password securepassword \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc-123
```

**Avantages :**
- ✅ Totalement managé (pas de gestion manuelle)
- ✅ Rotation automatique des clés
- ✅ Intégration avec IAM
- ✅ Sauvegardes automatiquement chiffrées

**Inconvénients :**
- ❌ Vendor lock-in
- ❌ Coût additionnel
- ❌ Moins de contrôle

#### Google Cloud SQL

**Similaire à AWS :** Chiffrement automatique avec Cloud KMS.

#### Azure Database for PostgreSQL

**Similaire à AWS/GCP :** Chiffrement avec Azure Key Vault.

---

## 🔑 Gestion des Clés de Chiffrement

### L'Importance de la Gestion des Clés

**Problème fondamental :**
> "Les données sont aussi sécurisées que la clé de chiffrement."

**Si la clé est compromise, le chiffrement est inutile.**

### Architecture de Gestion des Clés

#### Hiérarchie de Clés (Key Hierarchy)

**Architecture typique :**
```
┌──────────────────────────────────────────┐
│   Master Key (Clé Maîtresse)             │
│   - Stockée dans HSM ou KMS              │
│   - Rarement utilisée directement        │
│   - Durée de vie : années                │
└──────────────┬───────────────────────────┘
               │ (chiffre)
               ↓
┌──────────────────────────────────────────┐
│   Data Encryption Key (DEK)              │
│   - Chiffre les données réelles          │
│   - Stockée chiffrée par Master Key      │
│   - Durée de vie : mois/années           │
└──────────────┬───────────────────────────┘
               │ (chiffre)
               ↓
┌──────────────────────────────────────────┐
│   Données PostgreSQL                     │
│   - Fichiers de base de données          │
│   - WAL                                  │
│   - Sauvegardes                          │
└──────────────────────────────────────────┘
```

**Avantage :** Pour changer la clé, on peut re-chiffrer seulement la DEK (rapide) plutôt que toutes les données (très lent).

### Options de Stockage des Clés

#### Option 1 : Fichier sur Disque (⚠️ Faible Sécurité)

**Principe :** Stocker la clé dans un fichier protégé.

```bash
# Créer une clé
openssl rand -base64 32 > /etc/postgresql/encryption.key
chmod 600 /etc/postgresql/encryption.key
chown postgres:postgres /etc/postgresql/encryption.key
```

**Avantages :**
- ✅ Simple
- ✅ Gratuit

**Inconvénients :**
- ❌ Clé sur le même serveur que les données
- ❌ Vulnérable si le serveur est compromis
- ❌ Pas de rotation facile

**Quand l'utiliser :**
- Développement uniquement
- Protection minimale contre vol de disque

#### Option 2 : Variables d'Environnement (⚠️ Faible Sécurité)

**Principe :** Passer la clé via variable d'environnement.

```bash
# Définir la clé
export PGENCRYPTION_KEY="ma_cle_secrete_base64"

# Démarrer PostgreSQL
sudo -u postgres PGENCRYPTION_KEY=$PGENCRYPTION_KEY pg_ctl start
```

**Problème :** Les variables d'environnement sont visibles avec `ps` ou `/proc/`.

#### Option 3 : HSM (Hardware Security Module) ✅

**Principe :** Dispositif matériel dédié pour stocker et gérer les clés.

**Caractéristiques :**
- ✅ Clé ne quitte jamais le HSM
- ✅ Résistant aux attaques physiques
- ✅ Certification FIPS 140-2 Level 3 ou 4
- ✅ Rotation de clés facilitée

**Exemples de HSM :**
- Thales Luna HSM
- Utimaco HSM
- Gemalto SafeNet

**Inconvénients :**
- ❌ Très coûteux (10 000$ - 100 000$+)
- ❌ Complexité de mise en place

**Quand l'utiliser :**
- Données ultra-sensibles (gouvernement, finance)
- Exigences FIPS 140-2 Level 3+

#### Option 4 : KMS Cloud (Key Management Service) ✅

**Principe :** Service managé pour gérer les clés de chiffrement.

**Exemples :**
- **AWS KMS** (Key Management Service)
- **Google Cloud KMS**
- **Azure Key Vault**
- **HashiCorp Vault**

**Avantages :**
- ✅ Managé (pas d'infrastructure HSM à gérer)
- ✅ Rotation automatique
- ✅ Audit et logs
- ✅ Intégration IAM
- ✅ Prix raisonnable

**Exemple avec AWS KMS :**
```bash
# Récupérer la clé depuis KMS au démarrage
aws kms decrypt \
  --ciphertext-blob fileb://encrypted_key.bin \
  --output text \
  --query Plaintext | base64 --decode > /tmp/encryption.key
```

**Quand l'utiliser :**
- Production cloud
- Balance coût/sécurité optimale

#### Option 5 : HashiCorp Vault ✅

**Principe :** Solution open-source pour gestion centralisée des secrets et clés.

**Fonctionnalités :**
- ✅ Stockage sécurisé des clés
- ✅ Rotation automatique
- ✅ Audit complet
- ✅ Intégration avec PostgreSQL
- ✅ Multi-cloud

**Exemple d'intégration :**
```bash
# Récupérer la clé depuis Vault
vault kv get -field=encryption_key secret/postgresql/production > /tmp/encryption.key

# Utiliser avec PostgreSQL
# (dépend de la solution TDE choisie)
```

---

## 📋 Comparaison des Solutions

### Tableau Récapitulatif

| Solution | Complexité | Coût | Sécurité | Performance | Granularité | Recommandé |
|----------|------------|------|----------|-------------|-------------|------------|
| **LUKS (disque)** | Faible | Gratuit | Moyenne | Excellente | Disque entier | Dev, PME |
| **ZFS/Btrfs encryption** | Moyenne | Gratuit | Moyenne | Excellente | Filesystem | PME, serveurs modernes |
| **TDE Cybertec** | Moyenne | Payant | Élevée | Bonne | Table/database | Entreprise |
| **EDB Postgres** | Faible | Payant | Élevée | Excellente | Transparente | Grande entreprise |
| **AWS RDS** | Très faible | Moyen | Élevée | Excellente | Instance | Cloud AWS |
| **pgcrypto** | Élevée | Gratuit | Variable | Moyenne | Colonne | Cas spécifiques |

### Matrice de Décision

**Quel niveau de protection recherchez-vous ?**

```
┌─────────────────────────────────────────────────────────┐
│ Protection contre vol physique uniquement               │
│ → LUKS ou ZFS encryption                                │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Protection + sauvegardes chiffrées                      │
│ → LUKS + chiffrement des sauvegardes séparé             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Protection complète + conformité                        │
│ → TDE commercial (Cybertec, EDB) ou Cloud managé        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Niveau gouvernemental / défense                         │
│ → TDE + HSM + FIPS + audits                             │
└─────────────────────────────────────────────────────────┘
```

**Quel est votre environnement ?**

```
Bare Metal / VM on-premise → LUKS, ZFS, TDE commercial
Cloud AWS → RDS with KMS
Cloud GCP → Cloud SQL with Cloud KMS
Cloud Azure → Azure Database with Key Vault
Multi-cloud → HashiCorp Vault + solution TDE
```

**Quel est votre budget ?**

```
Budget limité (< 1000€/mois) → LUKS, ZFS (gratuit)
Budget moyen (1000-5000€/mois) → Cloud managé (RDS, Cloud SQL)
Budget élevé (> 5000€/mois) → Solution TDE commercial + HSM
```

---

## 🎯 Bonnes Pratiques

### 1. Ne Jamais Stocker la Clé avec les Données

**❌ Mauvaise pratique :**
```bash
# Clé dans le même répertoire que les données
/var/lib/postgresql/18/main/encryption.key
/var/lib/postgresql/18/main/base/
```

**✅ Bonne pratique :**
```bash
# Clé dans un emplacement séparé
/etc/postgresql/encryption.key (avec permissions 600)
# OU mieux : clé dans KMS/Vault
```

### 2. Chiffrer les Sauvegardes

**Même avec TDE activé**, chiffrez les sauvegardes séparément :

```bash
# Sauvegarde avec pg_dump + chiffrement GPG
pg_dump production | gzip | gpg --encrypt --recipient admin@example.com > backup.sql.gz.gpg

# Ou avec age (outil moderne)
pg_dump production | age -r age1... > backup.sql.age
```

### 3. Rotation Régulière des Clés

**Fréquence recommandée :**
- Master Key : Tous les 1-2 ans
- Data Encryption Keys : Tous les 6-12 mois
- Après un incident de sécurité : Immédiatement

**Avec KMS/Vault :**
```bash
# Rotation automatique dans AWS KMS
aws kms enable-key-rotation --key-id abc-123

# Vault rotation
vault write database/rotate-root/postgres-prod
```

### 4. Tester la Récupération

**Scénario catastrophe :** Perte de la clé de chiffrement

**Procédure de test trimestrielle :**
```
1. Simuler la perte de la clé
2. Tenter une restauration depuis sauvegarde
3. Vérifier que la procédure de récupération fonctionne
4. Documenter les problèmes rencontrés
5. Mettre à jour la documentation
```

### 5. Audit et Monitoring

**Logs à surveiller :**
- Accès aux clés de chiffrement
- Échecs de déchiffrement
- Modifications de configuration TDE
- Rotations de clés

**Exemple de monitoring avec Vault :**
```bash
# Vérifier les accès à la clé
vault audit list
vault audit enable file file_path=/var/log/vault_audit.log
```

### 6. Conformité et Documentation

**Documents requis :**
- Architecture de chiffrement
- Procédure de gestion des clés
- Plan de rotation
- Plan de récupération d'urgence
- Matrice de responsabilités

**Audits réguliers :**
- Vérifier que le chiffrement est actif
- Tester les sauvegardes chiffrées
- Valider les accès aux clés
- Revoir les logs d'audit

---

## ⚠️ Limitations et Considérations

### Limitation #1 : Performance

**Impact du chiffrement :**
- Surcharge CPU : 5-15% selon l'algorithme
- Latence légèrement accrue (microseconds)
- Throughput réduit de 5-10%

**Mitigation :**
- Utiliser AES-NI (accélération matérielle)
- Algorithmes optimisés (AES-GCM)
- Dimensionner le CPU en conséquence

**Benchmark typique :**
```
Sans chiffrement : 10 000 requêtes/sec
Avec LUKS :        9 500 requêtes/sec (-5%)
Avec TDE :         9 000 requêtes/sec (-10%)
```

### Limitation #2 : Données en Mémoire Non Chiffrées

**Important :** TDE ne protège **pas** les données en mémoire (RAM).

**Implication :**
- Un attaquant avec accès root peut dumper la mémoire
- Les données sensibles sont visibles dans les buffers
- Protection contre vol de RAM (peu probable mais possible)

**Solution partielle :**
- Chiffrer la mémoire swap
- Désactiver les core dumps
- Utiliser des outils anti-forensics

### Limitation #3 : Complexité de Gestion

**Défis opérationnels :**
- Gestion des clés complexe
- Procédures de rotation
- Récupération en cas de perte de clé
- Formation des équipes

**Recommandation :** Commencer simple (LUKS) puis évoluer vers des solutions plus avancées.

### Limitation #4 : Pas de Protection contre Administrateurs

**Si l'administrateur a accès à :**
- La base de données PostgreSQL (en cours d'exécution)
- ET la clé de chiffrement

**Alors :** Il peut accéder aux données.

**Solution :** Séparation des privilèges (separation of duties)
- Admin système ≠ DBA ≠ Gestionnaire des clés

---

## 🧠 Points Clés à Retenir

1. **Chiffrement au repos** protège les données stockées sur disque
2. **TDE** = chiffrement transparent (aucune modification application)
3. **PostgreSQL core** n'a pas de TDE natif (mais c'est en discussion)
4. **Solutions disponibles** : LUKS, ZFS, TDE commercial, Cloud managé
5. **LUKS** = simple et gratuit, protection de base
6. **Cloud (RDS, Cloud SQL)** = chiffrement managé, recommandé pour le cloud
7. **Gestion des clés** est critique : KMS/Vault recommandés
8. **Chiffrer les sauvegardes** séparément est essentiel
9. **Performance** : impact 5-15% (acceptable pour la plupart des cas)
10. **Rotation des clés** régulière et tests de récupération obligatoires

---

## 📋 Checklist de Mise en Place

### Évaluation Initiale

```
☐ Identifier les données sensibles nécessitant chiffrement
☐ Évaluer les exigences de conformité (RGPD, HIPAA, PCI-DSS)
☐ Déterminer le niveau de sécurité requis
☐ Évaluer le budget disponible
☐ Choisir la solution appropriée
```

### Implémentation (Exemple LUKS)

```
☐ Créer un volume chiffré LUKS
☐ Configurer l'ouverture automatique (avec gestion de clé sécurisée)
☐ Migrer les données PostgreSQL vers le volume chiffré
☐ Tester que PostgreSQL fonctionne correctement
☐ Vérifier les performances
☐ Configurer les sauvegardes chiffrées
☐ Tester la restauration depuis sauvegarde
```

### Gestion des Clés

```
☐ Choisir un système de gestion de clés (fichier, KMS, Vault, HSM)
☐ Générer la clé maîtresse
☐ Stocker la clé en lieu sûr
☐ Créer une copie de sauvegarde de la clé (coffre-fort)
☐ Documenter la procédure de récupération
☐ Définir un calendrier de rotation
☐ Former les équipes
```

### Validation et Production

```
☐ Tests de charge avec chiffrement activé
☐ Mesurer l'impact performance
☐ Simuler une perte de clé et récupération
☐ Valider que les sauvegardes sont chiffrées
☐ Configurer le monitoring et les alertes
☐ Documenter la configuration complète
☐ Former les équipes opérationnelles
☐ Planifier les audits réguliers
```

---

## 🚀 Pour Aller Plus Loin

### Sujets Connexes

Dans les sections suivantes du tutoriel :

- **16.7** : SSL/TLS et chiffrement des connexions (en transit)
- **16.11** : Sauvegardes et restauration
- **19.4** : Troubleshooting et monitoring de sécurité

### Ressources Complémentaires

**Documentation et Standards :**
- [PostgreSQL Security Best Practices](https://www.postgresql.org/docs/current/encryption-options.html)
- [NIST Guidelines on Cryptography](https://csrc.nist.gov/publications/sp800)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)

**Solutions Commerciales :**
- [Cybertec PostgreSQL TDE](https://www.cybertec-postgresql.com/en/products/cybertec-postgresql-transparent-data-encryption/)
- [EDB Postgres Advanced Server](https://www.enterprisedb.com/products/edb-postgres-advanced-server-secure-ha-oracle-compatible)
- [Percona Distribution for PostgreSQL](https://www.percona.com/software/postgresql-distribution)

**Open Source :**
- [LUKS Cryptsetup](https://gitlab.com/cryptsetup/cryptsetup)
- [HashiCorp Vault](https://www.vaultproject.io/)
- [ZFS Encryption](https://openzfs.github.io/openzfs-docs/man/8/zfs-load-key.8.html)

**Blogs et Tutoriels :**
- [Encrypting PostgreSQL Data at Rest](https://www.percona.com/blog/)
- [AWS RDS Encryption](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)
- [Google Cloud SQL Encryption](https://cloud.google.com/sql/docs/postgres/data-encryption)

---

## 🔮 Futur de TDE dans PostgreSQL

### Travaux en Cours

**PostgreSQL 19+ (Futur) :**
- Plusieurs propositions de TDE natif en discussion
- Développement actif par la communauté
- Patches soumis et en cours de review

**Challenges techniques :**
- Performance acceptable
- Compatibilité avec réplication
- Gestion des clés standardisée
- Support des outils existants

### Évolution Probable

**Court terme (1-2 ans) :**
- Solutions tierces matures (Cybertec, EDB)
- Adoption croissante du cloud managé
- Standardisation des pratiques

**Moyen terme (3-5 ans) :**
- TDE natif potentiellement intégré au core
- Meilleure intégration avec KMS standards
- Outils de migration simplifiés

**Long terme (5+ ans) :**
- TDE standard dans PostgreSQL
- Chiffrement homomorphe (calcul sur données chiffrées)
- Solutions hybrides cloud/on-premise

---


⏭️ [Maintenance vitale](/16-administration-configuration-securite/10-maintenance-vitale.md)
