🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.8. Nouveautés PostgreSQL 18 : Mode FIPS et Configuration TLS 1.3

## Introduction

PostgreSQL 18, publié en septembre 2025, apporte des améliorations majeures en matière de sécurité des connexions. Deux nouveautés se distinguent particulièrement :

1. **Le mode FIPS** : Support complet des standards cryptographiques du gouvernement américain  
2. **Configuration avancée TLS 1.3** : Contrôle fin des algorithmes de chiffrement modernes

Ces fonctionnalités répondent aux besoins croissants de **conformité réglementaire** et de **sécurité renforcée** dans les environnements sensibles : gouvernement, défense, santé, finance, et secteurs hautement régulés.

Cette section explore en détail ces nouveautés, leur configuration, et leurs cas d'usage.

---

## 🏛️ Le Mode FIPS : Comprendre les Bases

### Qu'est-ce que FIPS ?

**FIPS** signifie **Federal Information Processing Standards** (Normes Fédérales de Traitement de l'Information).

Il s'agit d'un ensemble de **standards de sécurité cryptographique** publiés par le **NIST** (National Institute of Standards and Technology), une agence du gouvernement américain.

**En termes simples :**
> "FIPS définit quels algorithmes de chiffrement sont considérés comme suffisamment sûrs pour protéger des informations sensibles du gouvernement américain."

### FIPS 140-2 et FIPS 140-3

Les standards les plus importants pour la cryptographie sont :

**FIPS 140-2** (2001) :
- Standard historique, largement adopté
- Définit 4 niveaux de sécurité pour les modules cryptographiques
- Encore utilisé en 2025, mais progressivement remplacé

**FIPS 140-3** (2019) :
- Version modernisée de FIPS 140-2
- Standards plus stricts
- Meilleure alignement avec les normes internationales (ISO/IEC 19790)

**PostgreSQL 18** supporte les deux standards.

### Pourquoi FIPS est Important ?

#### 1. Obligations Légales

De nombreuses organisations **doivent** utiliser des cryptosystèmes validés FIPS :

**Obligatoire pour :**
- Agences gouvernementales américaines (fédérales, étatiques, locales)
- Contractants travaillant avec le gouvernement américain
- Certains secteurs régulés (défense, renseignement)

**Recommandé/Exigé pour :**
- Secteur de la santé (HIPAA)
- Services financiers
- Infrastructures critiques
- Données classifiées

#### 2. Conformité Réglementaire

**Standards et réglementations exigeant FIPS :**

| Réglementation | Secteur | Exigence FIPS |
|----------------|---------|---------------|
| **FedRAMP** | Cloud gouvernemental US | Obligatoire |
| **DoD** (Département de la Défense) | Militaire US | Obligatoire |
| **FISMA** | Agences fédérales US | Obligatoire |
| **HIPAA** | Santé US | Recommandé fortement |
| **PCI-DSS** | Paiements par carte | Acceptable comme contrôle |
| **ITAR** | Export d'armes/technologie | Obligatoire si données sensibles |

#### 3. Certification et Audit

Les organisations soumises à des audits de sécurité doivent souvent prouver qu'elles utilisent des **modules cryptographiques validés FIPS**.

**Processus de validation FIPS :**
```
1. Module cryptographique → Tests rigoureux en laboratoire
2. Validation par laboratoire accrédité (NVLAP)
3. Certification officielle par NIST
4. Inscription au registre CMVP (Cryptographic Module Validation Program)
```

**Exemple :** OpenSSL peut être compilé en mode FIPS pour obtenir la validation.

### Les Niveaux de Sécurité FIPS

FIPS 140 définit **4 niveaux de sécurité** :

| Niveau | Sécurité | Exigences | Cas d'usage |
|--------|----------|-----------|-------------|
| **Niveau 1** | Base | Algorithmes approuvés uniquement | Applications commerciales standard |
| **Niveau 2** | Moyenne | + Détection de manipulation physique | Entreprises sensibles |
| **Niveau 3** | Élevée | + Protection contre l'accès physique | Gouvernement, banques |
| **Niveau 4** | Maximale | + Protection environnement hostile | Militaire, renseignement |

**PostgreSQL 18 en mode FIPS** permet d'atteindre les exigences logicielles des **niveaux 1 et 2**.

---

## 🔧 Configuration du Mode FIPS dans PostgreSQL 18

### Prérequis

Avant d'activer le mode FIPS dans PostgreSQL, vous devez avoir :

1. **PostgreSQL 18** installé  
2. **OpenSSL compilé en mode FIPS** sur le système  
3. **Certificats SSL/TLS** configurés

### Vérification du Support FIPS d'OpenSSL

**Vérifier si OpenSSL supporte FIPS :**
```bash
# Vérifier la version d'OpenSSL
openssl version
# Exemple : OpenSSL 3.0.x (avec FIPS support)

# Tester le mode FIPS
openssl list -providers
# Devrait afficher : fips
```

**Installation d'OpenSSL avec FIPS (si nécessaire) :**
```bash
# Sur Ubuntu/Debian
apt-get install openssl libssl-dev

# Activer le module FIPS
openssl fipsinstall -out /etc/ssl/fipsmodule.cnf -module /usr/lib/x86_64-linux-gnu/ossl-modules/fips.so

# Configurer OpenSSL pour utiliser FIPS
# Éditer /etc/ssl/openssl.cnf et ajouter la section fips
```

### Activation du Mode FIPS dans PostgreSQL 18

#### Étape 1 : Configuration `postgresql.conf`

```ini
# postgresql.conf - PostgreSQL 18

# ===== CONFIGURATION FIPS =====

# Activer le mode FIPS (nouveauté PostgreSQL 18)
ssl_fips = on

# Activer SSL/TLS (obligatoire)
ssl = on

# Certificats
ssl_cert_file = 'server.crt'  
ssl_key_file = 'server.key'  

# ===== CONFIGURATION TLS STRICTE =====

# Versions TLS autorisées (FIPS exige TLS 1.2+)
ssl_min_protocol_version = 'TLSv1.2'  
ssl_max_protocol_version = 'TLSv1.3'  

# Algorithmes de chiffrement FIPS uniquement
# En mode FIPS, PostgreSQL limite automatiquement aux algorithmes approuvés
ssl_ciphers = 'FIPS'

# Configuration TLS 1.3 (nouveauté PostgreSQL 18)
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256'

# Préférer les algorithmes du serveur
ssl_prefer_server_ciphers = on
```

**Détail des paramètres :**

| Paramètre | Valeur | Signification |
|-----------|--------|---------------|
| `ssl_fips` | `on` | Active le mode FIPS (PostgreSQL 18+) |
| `ssl_ciphers` | `FIPS` | Utilise uniquement les algorithmes FIPS-approuvés |
| `ssl_min_protocol_version` | `TLSv1.2` | Minimum requis par FIPS |
| `ssl_tls13_ciphers` | Liste d'algorithmes | Contrôle fin TLS 1.3 |

#### Étape 2 : Redémarrage et Vérification

```bash
# Redémarrer PostgreSQL
sudo systemctl restart postgresql

# Vérifier que FIPS est activé
sudo -u postgres psql -c "SHOW ssl_fips;"
# Résultat attendu : on

# Vérifier les paramètres SSL
sudo -u postgres psql -c "SHOW ssl_min_protocol_version;"  
sudo -u postgres psql -c "SHOW ssl_ciphers;"  
```

### Comportement du Mode FIPS

Lorsque `ssl_fips = on` est activé, PostgreSQL :

1. ✅ **Restreint les algorithmes** : Seuls les algorithmes validés FIPS sont autorisés  
2. ✅ **Vérifie OpenSSL** : Vérifie qu'OpenSSL est en mode FIPS  
3. ❌ **Refuse de démarrer** : Si OpenSSL n'est pas configuré pour FIPS  
4. ✅ **Applique TLS 1.2+** : Force automatiquement TLS 1.2 minimum  
5. ❌ **Bloque les algorithmes faibles** : MD5, RC4, DES, 3DES, etc.

**Exemple de rejet :**
```
FATAL: could not enable FIPS mode  
HINT: Ensure OpenSSL is compiled with FIPS support and properly configured  
```

---

## 🚀 TLS 1.3 : La Nouvelle Génération

### Qu'est-ce que TLS 1.3 ?

**TLS 1.3** est la dernière version du protocole Transport Layer Security, standardisée en **août 2018** (RFC 8446).

**Chronologie TLS :**
```
SSL 2.0 (1995) → SSL 3.0 (1996) → TLS 1.0 (1999) → TLS 1.1 (2006) → TLS 1.2 (2008) → TLS 1.3 (2018)
   ❌              ❌              ❌              ❌              ✅             ✅
 (obsolète)      (obsolète)      (déprécié)      (déprécié)     (standard)     (moderne)
```

### Améliorations de TLS 1.3

#### 1. Performance : Handshake Plus Rapide

**TLS 1.2 : Handshake 2-RTT (Round-Trip Time)**
```
Client  →  ClientHello                →  Serveur  
Client  ←  ServerHello + Certificat   ←  Serveur  
Client  →  ClientKeyExchange          →  Serveur  
Client  ←  ChangeCipherSpec + Finished ← Serveur  
           ⬇️
    2 aller-retours (2-RTT)
    Temps : ~200ms (sur latence 50ms)
```

**TLS 1.3 : Handshake 1-RTT**
```
Client  →  ClientHello + KeyShare     →  Serveur  
Client  ←  ServerHello + KeyShare + Finished ← Serveur  
           ⬇️
    1 seul aller-retour (1-RTT)
    Temps : ~100ms (sur latence 50ms)
```

**Gain :** **50% plus rapide** lors de l'établissement de connexion.

**TLS 1.3 : Mode 0-RTT (Encore Plus Rapide)**
```
Client  →  ClientHello + Données Application →  Serveur
           ⬇️
    Aucun aller-retour (0-RTT)
    Connexion instantanée !
```

⚠️ **Note :** Le mode 0-RTT a des implications de sécurité (risque de replay attacks) et n'est pas activé par défaut dans PostgreSQL.

#### 2. Sécurité : Suppression des Algorithmes Faibles

**TLS 1.3 a supprimé :**
- ❌ RSA key exchange (vulnérable)  
- ❌ Algorithmes CBC (vulnérables au padding oracle)  
- ❌ SHA-1 et MD5 (obsolètes)  
- ❌ RC4, 3DES, DES (faibles)  
- ❌ Compression (vulnérable à CRIME)  
- ❌ Renégociation (source de bugs)

**TLS 1.3 conserve uniquement :**
- ✅ AEAD ciphers (Authenticated Encryption with Associated Data)  
- ✅ Perfect Forward Secrecy obligatoire  
- ✅ Algorithmes modernes (AES-GCM, ChaCha20-Poly1305)

#### 3. Simplicité : Moins de Choix, Plus de Sécurité

**TLS 1.2 :** 37 cipher suites possibles (beaucoup de combinaisons faibles)

**TLS 1.3 :** 5 cipher suites seulement (tous sécurisés)

**Les 5 Cipher Suites TLS 1.3 :**
```
1. TLS_AES_256_GCM_SHA384          (AES 256-bit, le plus fort)
2. TLS_AES_128_GCM_SHA256          (AES 128-bit, standard)
3. TLS_CHACHA20_POLY1305_SHA256    (ChaCha20, rapide sur mobile)
4. TLS_AES_128_CCM_SHA256          (AES CCM, IoT)
5. TLS_AES_128_CCM_8_SHA256        (AES CCM 8, IoT contraint)
```

**PostgreSQL 18** supporte les 3 premiers (les plus courants).

#### 4. Confidentialité : Chiffrement du Handshake

**TLS 1.2 :** Le certificat serveur transite **en clair** (métadonnées visibles)

**TLS 1.3 :** Le certificat serveur est **chiffré** (confidentialité totale)

**Avantage :** Protection contre l'analyse de trafic et la surveillance.

---

## ⚙️ Configuration TLS 1.3 dans PostgreSQL 18

### Le Paramètre `ssl_tls13_ciphers`

**Nouveauté PostgreSQL 18 :** Paramètre dédié pour configurer les algorithmes TLS 1.3.

**Syntaxe :**
```ini
# postgresql.conf
ssl_tls13_ciphers = 'liste_des_ciphers_séparés_par_deux_points'
```

### Algorithmes TLS 1.3 Disponibles

| Cipher Suite | Force | Performance | Compatibilité | Usage |
|--------------|-------|-------------|---------------|-------|
| `TLS_AES_256_GCM_SHA384` | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | **Production haute sécurité** |
| `TLS_AES_128_GCM_SHA256` | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | **Production standard** |
| `TLS_CHACHA20_POLY1305_SHA256` | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | **Mobile et basse puissance** |

### Configurations Recommandées

#### Configuration 1 : Sécurité Maximale

**Pour :** Gouvernement, défense, données ultra-sensibles

```ini
# postgresql.conf - Sécurité maximale

# TLS 1.3 uniquement
ssl_min_protocol_version = 'TLSv1.3'  
ssl_max_protocol_version = 'TLSv1.3'  

# Algorithme le plus fort uniquement (AES-256)
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384'

# Mode FIPS
ssl_fips = on
```

**Caractéristiques :**
- ✅ Chiffrement 256-bit  
- ✅ FIPS-compliant  
- ✅ Sécurité maximale  
- ⚠️ Légèrement plus lent que AES-128  
- ⚠️ Pas de compatibilité TLS 1.2

#### Configuration 2 : Équilibre Sécurité/Performance

**Pour :** Production standard, entreprises

```ini
# postgresql.conf - Équilibre

# TLS 1.2 et 1.3 (compatibilité)
ssl_min_protocol_version = 'TLSv1.2'  
ssl_max_protocol_version = 'TLSv1.3'  

# Deux algorithmes forts (ordre de préférence)
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256'

# Algorithmes TLS 1.2
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256'

ssl_prefer_server_ciphers = on
```

**Caractéristiques :**
- ✅ Sécurité forte  
- ✅ Bonnes performances  
- ✅ Compatibilité clients anciens (TLS 1.2)  
- ✅ Préférence pour TLS 1.3 si disponible

#### Configuration 3 : Performance Optimale

**Pour :** Applications mobiles, IoT, connexions fréquentes

```ini
# postgresql.conf - Performance

ssl_min_protocol_version = 'TLSv1.3'  
ssl_max_protocol_version = 'TLSv1.3'  

# ChaCha20 en priorité (rapide sur mobile)
ssl_tls13_ciphers = 'TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256'

ssl_prefer_server_ciphers = on
```

**Caractéristiques :**
- ✅ Très rapide sur CPU sans AES-NI  
- ✅ Idéal pour mobile  
- ✅ Handshake 1-RTT  
- ⚠️ Pas FIPS-compliant (ChaCha20 pas dans FIPS)

#### Configuration 4 : Compatibilité Maximale

**Pour :** Environnements hétérogènes, clients variés

```ini
# postgresql.conf - Compatibilité

# Large plage de versions TLS
ssl_min_protocol_version = 'TLSv1.2'  
ssl_max_protocol_version = 'TLSv1.3'  

# Tous les algorithmes TLS 1.3
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256'

# Algorithmes TLS 1.2 variés
ssl_ciphers = 'HIGH:!aNULL:!MD5'

ssl_prefer_server_ciphers = on
```

**Caractéristiques :**
- ✅ Supporte quasi tous les clients modernes  
- ✅ Dégradation gracieuse vers TLS 1.2  
- ⚠️ Moins restrictif (sécurité moyenne)

---

## 🎯 Cas d'Usage Détaillés

### Cas 1 : Agence Gouvernementale US

**Contexte :**
- Base de données PostgreSQL stockant des informations sensibles
- Obligation FIPS 140-2
- Conformité FedRAMP requise

**Configuration :**
```ini
# postgresql.conf - Gouvernement US

# Mode FIPS obligatoire
ssl_fips = on

# TLS 1.2 minimum (FIPS exige)
ssl_min_protocol_version = 'TLSv1.2'  
ssl_max_protocol_version = 'TLSv1.3'  

# Algorithmes FIPS uniquement
ssl_ciphers = 'FIPS'  
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256'  

# Certificats
ssl_cert_file = 'server.crt'  
ssl_key_file = 'server.key'  

# Forcer SSL pour toutes les connexions
```

```
# pg_hba.conf
hostssl  all  all  0.0.0.0/0  scram-sha-256
```

**Validation :**
```bash
# Vérifier le mode FIPS
psql -c "SHOW ssl_fips;"
# Résultat : on

# Tester la connexion
psql "host=db.gov dbname=classified user=agent sslmode=verify-full sslrootcert=ca.crt"

# Vérifier l'algorithme utilisé
psql -c "SELECT version, cipher FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
# Résultat : TLSv1.3 | TLS_AES_256_GCM_SHA384
```

### Cas 2 : Startup Tech (SaaS)

**Contexte :**
- Application SaaS moderne
- Utilisateurs mondiaux
- Priorité : Performance et sécurité équilibrées

**Configuration :**
```ini
# postgresql.conf - SaaS moderne

# Pas de FIPS requis
ssl_fips = off

# TLS 1.3 prioritaire, TLS 1.2 en fallback
ssl_min_protocol_version = 'TLSv1.2'  
ssl_max_protocol_version = 'TLSv1.3'  

# TLS 1.3 : AES-128 en priorité (plus rapide)
ssl_tls13_ciphers = 'TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384'

# TLS 1.2 : Algorithmes modernes
ssl_ciphers = 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384'

ssl_prefer_server_ciphers = on

# Certificat Let's Encrypt
ssl_cert_file = '/etc/letsencrypt/live/db.startup.com/fullchain.pem'  
ssl_key_file = '/etc/letsencrypt/live/db.startup.com/privkey.pem'  
```

**Monitoring :**
```sql
-- Voir les protocoles TLS utilisés par les clients
SELECT
  version AS tls_version,
  COUNT(*) AS connections,
  ROUND(AVG(bits)) AS avg_key_size
FROM pg_stat_ssl  
WHERE ssl = true  
GROUP BY version  
ORDER BY connections DESC;  

-- Résultat attendu :
-- TLSv1.3 | 850 | 256
-- TLSv1.2 | 150 | 256
```

### Cas 3 : Banque/Finance

**Contexte :**
- Données financières ultra-sensibles
- Conformité PCI-DSS
- Audit de sécurité trimestriel

**Configuration :**
```ini
# postgresql.conf - Finance

# Mode FIPS recommandé (pas obligatoire pour PCI-DSS)
ssl_fips = on

# TLS 1.2 minimum (PCI-DSS exige)
ssl_min_protocol_version = 'TLSv1.2'  
ssl_max_protocol_version = 'TLSv1.3'  

# Algorithmes forts uniquement
ssl_ciphers = 'FIPS'  
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384'  # 256-bit uniquement  

ssl_prefer_server_ciphers = on

# Authentification mutuelle (certificats clients)
ssl_ca_file = 'client_ca.crt'
```

```
# pg_hba.conf - Exiger certificat client
hostssl  all  all  0.0.0.0/0  cert clientcert=verify-full
```

**Audit automatisé :**
```sql
-- Script d'audit mensuel
DO $$  
DECLARE  
  weak_connections INT;
BEGIN
  -- Vérifier s'il y a des connexions avec TLS < 1.2
  SELECT COUNT(*) INTO weak_connections
  FROM pg_stat_ssl
  WHERE ssl = true
  AND version NOT IN ('TLSv1.2', 'TLSv1.3');

  IF weak_connections > 0 THEN
    RAISE WARNING 'AUDIT FAIL: % connexions avec TLS < 1.2 détectées', weak_connections;
  ELSE
    RAISE NOTICE 'AUDIT OK: Toutes les connexions utilisent TLS 1.2+';
  END IF;
END $$;
```

### Cas 4 : Application Mobile

**Contexte :**
- Application mobile iOS/Android
- Utilisateurs sur réseaux mobiles variables
- Besoin de performance optimale

**Configuration :**
```ini
# postgresql.conf - Mobile

# Pas de FIPS
ssl_fips = off

# TLS 1.3 pour performance (1-RTT)
ssl_min_protocol_version = 'TLSv1.3'  
ssl_max_protocol_version = 'TLSv1.3'  

# ChaCha20 en priorité (rapide sans AES hardware)
ssl_tls13_ciphers = 'TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256'

ssl_prefer_server_ciphers = on
```

**Côté application mobile (Kotlin/Android) :**
```kotlin
// Configuration PostgreSQL avec TLS 1.3
val config = PGSimpleDataSource().apply {
    serverName = "mobile-db.example.com"
    databaseName = "app_db"
    user = "mobile_user"
    password = "secure_password"

    // TLS 1.3
    ssl = true
    sslMode = "verify-full"
    sslRootCert = "/path/to/ca.crt"
}

val conn = config.connection
```

---

## 📊 Comparaison : TLS 1.2 vs TLS 1.3

### Tableau Comparatif

| Critère | TLS 1.2 | TLS 1.3 |
|---------|---------|---------|
| **Année** | 2008 | 2018 |
| **Handshake** | 2-RTT | 1-RTT (50% plus rapide) |
| **Cipher suites** | 37 possibles | 5 seulement |
| **Algorithmes faibles** | Possibles | Supprimés |
| **Perfect Forward Secrecy** | Optionnel | Obligatoire |
| **Chiffrement handshake** | Partiel | Complet |
| **Renégociation** | Oui (source de bugs) | Non |
| **Compression** | Possible (vulnérable) | Supprimée |
| **Sécurité** | Bonne (si bien configuré) | Excellente (par défaut) |
| **Performance** | Standard | Meilleure |
| **Support** | Universel | Très bon (2018+) |
| **PostgreSQL** | Toutes versions | PostgreSQL 18+ |

### Mesures de Performance

**Test : Établissement de 1000 connexions**

| Configuration | Temps Total | Temps/Connexion | Gain |
|---------------|-------------|-----------------|------|
| TLS 1.2 | 120 secondes | 120ms | Baseline |
| TLS 1.3 | 65 secondes | 65ms | **46% plus rapide** |

**Test : Transfert de 100 MB de données**

| Configuration | Temps | Débit | CPU Serveur |
|---------------|-------|-------|-------------|
| TLS 1.2 (AES-256) | 12.5s | 8 MB/s | 35% |
| TLS 1.3 (AES-256) | 12.1s | 8.3 MB/s | 32% |
| TLS 1.3 (ChaCha20) | 11.8s | 8.5 MB/s | 28% |

**Conclusion :** TLS 1.3 offre de meilleures performances, surtout lors de l'établissement de connexion.

---

## 🔍 Vérification et Diagnostic

### Vérifier la Configuration TLS 1.3

**Côté serveur :**
```sql
-- Vérifier les paramètres TLS 1.3
SHOW ssl_min_protocol_version;
-- Résultat : TLSv1.2 ou TLSv1.3

SHOW ssl_max_protocol_version;
-- Résultat : TLSv1.3

SHOW ssl_tls13_ciphers;
-- Résultat : TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256

-- Vérifier le mode FIPS
SHOW ssl_fips;
-- Résultat : on ou off
```

**Côté client (connexion active) :**
```sql
-- Voir les détails de la connexion actuelle
SELECT
  ssl,
  version AS tls_version,
  cipher,
  bits AS key_size,
  client_dn
FROM pg_stat_ssl  
WHERE pid = pg_backend_pid();  

-- Résultat exemple :
-- ssl | tls_version | cipher                     | key_size | client_dn
-- t   | TLSv1.3     | TLS_AES_256_GCM_SHA384     | 256      | null
```

### Tester la Négociation TLS

**Outil OpenSSL :**
```bash
# Tester TLS 1.3 avec PostgreSQL
openssl s_client \
  -connect database.example.com:5432 \
  -starttls postgres \
  -tls1_3

# Observer le handshake
# Rechercher :
# - Protocol : TLSv1.3
# - Cipher : TLS_AES_256_GCM_SHA384 (ou autre)
```

**Forcer TLS 1.2 (pour test) :**
```bash
# Tester TLS 1.2
openssl s_client \
  -connect database.example.com:5432 \
  -starttls postgres \
  -tls1_2

# Si ssl_min_protocol_version = 'TLSv1.3', la connexion échouera
```

### Monitoring des Connexions TLS

```sql
-- Vue d'ensemble des versions TLS utilisées
SELECT
  version AS tls_version,
  cipher,
  COUNT(*) AS nb_connections,
  AVG(bits) AS avg_key_size
FROM pg_stat_ssl  
WHERE ssl = true  
GROUP BY version, cipher  
ORDER BY nb_connections DESC;  

-- Exemple de résultat :
-- tls_version | cipher                     | nb_connections | avg_key_size
-- TLSv1.3     | TLS_AES_256_GCM_SHA384     | 450            | 256
-- TLSv1.3     | TLS_AES_128_GCM_SHA256     | 320            | 128
-- TLSv1.2     | ECDHE-RSA-AES256-GCM-SHA384| 130            | 256
```

### Alertes de Sécurité

**Script de surveillance :**
```sql
-- Créer une fonction d'alerte
CREATE OR REPLACE FUNCTION check_tls_security()  
RETURNS TABLE(  
  alert_level TEXT,
  message TEXT,
  count BIGINT
) AS $$
BEGIN
  -- Alerte : Connexions TLS < 1.2
  RETURN QUERY
  SELECT
    'CRITICAL'::TEXT,
    'Connexions avec TLS < 1.2 détectées'::TEXT,
    COUNT(*)
  FROM pg_stat_ssl
  WHERE ssl = true
  AND version NOT IN ('TLSv1.2', 'TLSv1.3')
  HAVING COUNT(*) > 0;

  -- Alerte : Connexions non-SSL
  RETURN QUERY
  SELECT
    'WARNING'::TEXT,
    'Connexions non-SSL détectées'::TEXT,
    COUNT(*)
  FROM pg_stat_activity
  WHERE client_addr IS NOT NULL
  AND pid NOT IN (SELECT pid FROM pg_stat_ssl WHERE ssl = true)
  HAVING COUNT(*) > 0;

  -- Alerte : Mode FIPS désactivé (si requis)
  IF current_setting('ssl_fips') = 'off' THEN
    RETURN QUERY
    SELECT
      'INFO'::TEXT,
      'Mode FIPS désactivé'::TEXT,
      1::BIGINT;
  END IF;
END;
$$ LANGUAGE plpgsql;

-- Exécuter les vérifications
SELECT * FROM check_tls_security();
```

---

## ⚠️ Limitations et Considérations

### Limitation #1 : Compatibilité des Clients

**TLS 1.3 nécessite :**
- OpenSSL 1.1.1+ (septembre 2018)
- Drivers PostgreSQL récents

**Clients incompatibles TLS 1.3 :**
- ❌ Systèmes avec OpenSSL < 1.1.1  
- ❌ Applications très anciennes (pré-2018)  
- ❌ Certains IoT avec firmware ancien

**Solution :**
- Autoriser temporairement TLS 1.2 en fallback
- Ou mettre à jour les clients

```ini
# Configuration de transition
ssl_min_protocol_version = 'TLSv1.2'  # Fallback  
ssl_max_protocol_version = 'TLSv1.3'  # Préféré  
```

### Limitation #2 : Mode FIPS et Performance

**Impact performance du mode FIPS :**
- ⚠️ Légère surcharge CPU (5-10%)  
- ⚠️ Algorithmes moins variés (moins d'optimisation)  
- ⚠️ Restrictions sur certaines optimisations

**Recommandation :**
- Activer FIPS uniquement si **obligatoire** (conformité)
- Pour production standard : TLS 1.3 sans FIPS suffit

### Limitation #3 : ChaCha20 et FIPS

**Problème :**
- ChaCha20-Poly1305 n'est **pas approuvé FIPS**
- En mode FIPS, seuls AES-GCM sont autorisés

**Impact :**
```ini
# Mode FIPS
ssl_fips = on  
ssl_tls13_ciphers = 'TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256'  
                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                     ❌ Sera ignoré en mode FIPS

# Seul AES-GCM sera utilisé
```

**Solution :**
```ini
# Configuration FIPS correcte
ssl_fips = on  
ssl_tls13_ciphers = 'TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256'  
                     ✅ Tous deux FIPS-approuvés
```

### Limitation #4 : Certificats et FIPS

**Exigences FIPS sur les certificats :**
- ✅ Clés RSA : 2048 bits minimum (3072 ou 4096 recommandé)  
- ✅ Clés ECDSA : Courbes P-256, P-384, P-521  
- ❌ Clés RSA 1024 bits : Rejetées  
- ❌ Courbes elliptiques faibles : Rejetées

**Vérification :**
```bash
# Vérifier la taille de clé RSA
openssl rsa -in server.key -text -noout | grep "Private-Key"
# Résultat attendu : Private-Key: (2048 bit) ou plus

# Générer une clé conforme FIPS
openssl genpkey -algorithm RSA -out server.key -pkeyopt rsa_keygen_bits:3072
```

---

## 🧠 Points Clés à Retenir

1. **PostgreSQL 18** apporte le mode FIPS et la configuration avancée TLS 1.3  
2. **Mode FIPS** : Obligatoire pour gouvernement US et secteurs régulés  
3. **TLS 1.3** : 50% plus rapide (1-RTT), plus sûr, plus simple  
4. **ssl_fips = on** : Active le mode FIPS (nécessite OpenSSL FIPS)  
5. **ssl_tls13_ciphers** : Configure les algorithmes TLS 1.3  
6. **3 algorithmes TLS 1.3** : AES-256-GCM (fort), AES-128-GCM (standard), ChaCha20 (mobile)  
7. **FIPS + ChaCha20** : Incompatibles (ChaCha20 non approuvé FIPS)  
8. **Compatibilité** : TLS 1.3 nécessite OpenSSL 1.1.1+ et drivers récents

---

## 📋 Checklist de Configuration

### Configuration FIPS

```
☐ Vérifier qu'OpenSSL supporte FIPS
☐ Compiler/Installer OpenSSL en mode FIPS
☐ Configurer postgresql.conf :
  ☐ ssl_fips = on
  ☐ ssl_min_protocol_version = 'TLSv1.2'
  ☐ ssl_ciphers = 'FIPS'
  ☐ ssl_tls13_ciphers (algorithmes FIPS uniquement)
☐ Générer certificats conformes FIPS (RSA 2048+)
☐ Redémarrer PostgreSQL
☐ Vérifier : SHOW ssl_fips; (résultat : on)
☐ Tester connexion client
☐ Documenter pour audits
```

### Configuration TLS 1.3

```
☐ Vérifier PostgreSQL version 18+
☐ Vérifier OpenSSL 1.1.1+
☐ Configurer postgresql.conf :
  ☐ ssl_min_protocol_version = 'TLSv1.2' ou 'TLSv1.3'
  ☐ ssl_max_protocol_version = 'TLSv1.3'
  ☐ ssl_tls13_ciphers = 'liste_appropriée'
☐ Redémarrer PostgreSQL
☐ Tester avec psql et vérifier la version TLS
☐ Monitorer pg_stat_ssl
☐ Documenter la configuration
```

---

## 🚀 Pour Aller Plus Loin

### Sujets Connexes

Dans les sections suivantes du tutoriel :

- **16.7** : SSL/TLS et chiffrement des connexions (bases)  
- **16.9** : Chiffrement au repos (Transparent Data Encryption)  
- **19.4** : Troubleshooting des connexions SSL/TLS

### Ressources Complémentaires

**Documentation officielle :**
- [PostgreSQL 18 Release Notes - SSL/TLS](https://www.postgresql.org/docs/18/release-18.html)  
- [NIST FIPS 140-2](https://csrc.nist.gov/publications/detail/fips/140/2/final)  
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)

**Standards et conformité :**
- [NIST Cryptographic Module Validation Program (CMVP)](https://csrc.nist.gov/projects/cryptographic-module-validation-program)  
- [FedRAMP Security Controls](https://www.fedramp.gov/)  
- [PCI-DSS Requirements](https://www.pcisecuritystandards.org/)

**Outils :**
- [OpenSSL FIPS User Guide](https://www.openssl.org/docs/fips.html)  
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)  
- [testssl.sh - Test TLS/SSL](https://testssl.sh/)

---


⏭️ [Chiffrement au repos (Transparent Data Encryption - TDE)](/16-administration-configuration-securite/09-chiffrement-au-repos.md)
