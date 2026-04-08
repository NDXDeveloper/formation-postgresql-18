🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.3. Dépréciation de MD5 et Migration vers SCRAM-SHA-256

## Introduction

La sécurité informatique est un domaine en constante évolution. Ce qui était considéré comme sûr il y a 20 ans peut devenir vulnérable aujourd'hui avec l'augmentation de la puissance de calcul et les nouvelles techniques d'attaque.

Dans PostgreSQL, **MD5** a longtemps été la méthode d'authentification par mot de passe la plus courante. Cependant, PostgreSQL a officiellement **déprécié MD5** et recommande fortement de migrer vers **SCRAM-SHA-256**, une méthode d'authentification moderne et beaucoup plus sécurisée.

Cette section explique **pourquoi** MD5 est obsolète et **comment** comprendre la migration vers SCRAM-SHA-256.

---

## 🔴 Qu'est-ce que MD5 et Pourquoi est-il Déprécié ?

### Qu'est-ce que MD5 ?

**MD5** (Message Digest 5) est un **algorithme de hachage cryptographique** créé en 1991. Dans le contexte de PostgreSQL, MD5 était utilisé pour :

1. **Stocker les mots de passe** de manière hachée (et non en clair)  
2. **Authentifier les utilisateurs** lors de la connexion

**Comment ça fonctionne (simplifié) :**
```
Mot de passe : "MonMotDePasse123"
           ⬇️ (algorithme MD5)
Hash MD5 : "5f4dcc3b5aa765d61d8327deb882cf99"
```

Le hash est stocké dans PostgreSQL au lieu du mot de passe en clair. Lors de la connexion, PostgreSQL compare le hash du mot de passe fourni avec celui stocké.

### Pourquoi MD5 est-il Problématique ?

#### 1. **Algorithme Obsolète (1991)**

MD5 a été conçu il y a plus de 30 ans. À l'époque, les ordinateurs étaient beaucoup moins puissants.

**Analogie :** C'est comme utiliser un cadenas des années 1990 alors que les cambrioleurs d'aujourd'hui ont des outils bien plus sophistiqués.

#### 2. **Vulnérable aux Attaques par Collisions**

Une **collision** se produit quand deux mots de passe différents produisent le même hash MD5.

```
Mot de passe A : "password123"  ➡️  Hash : "abc123def..."  
Mot de passe B : "p@ssw0rd456"  ➡️  Hash : "abc123def..." (même hash !)  
```

⚠️ **Problème** : Un attaquant peut trouver un mot de passe différent qui produit le même hash et ainsi se connecter.

#### 3. **Attaques par Force Brute Rapides**

Avec les ordinateurs modernes (et surtout les GPU), il est possible de tester des **milliards de combinaisons par seconde** pour casser un hash MD5.

**Exemple de vitesse d'attaque :**
- **MD5** : ~60 milliards de hashs/seconde sur un GPU moderne  
- **SCRAM-SHA-256** : ~1000 hashs/seconde (60 millions de fois plus lent !)

#### 4. **Vulnérable aux Rainbow Tables**

Une **Rainbow Table** est une base de données précalculée contenant des millions de mots de passe et leurs hashs MD5.

**Fonctionnement :**
```
1. L'attaquant vole le hash MD5 : "5f4dcc3b5aa765d61d8327deb882cf99"
2. Il cherche dans sa Rainbow Table
3. Résultat trouvé instantanément : "password"
```

⚠️ **Problème** : Pas besoin de calculer, juste de chercher dans une table !

#### 5. **Absence de "Salt" par Défaut**

Un **salt** est une valeur aléatoire ajoutée au mot de passe avant le hachage pour rendre chaque hash unique.

**Sans salt (MD5 dans PostgreSQL) :**
```
Utilisateur A : "password" ➡️ Hash : "5f4dcc3b5..."  
Utilisateur B : "password" ➡️ Hash : "5f4dcc3b5..." (identique !)  
```

Si un attaquant casse un hash, il casse tous les utilisateurs avec le même mot de passe.

**Avec salt (SCRAM-SHA-256) :**
```
Utilisateur A : "password" + salt aléatoire ➡️ Hash : "a1b2c3d4..."  
Utilisateur B : "password" + salt différent ➡️ Hash : "x9y8z7w6..." (différent !)  
```

Chaque utilisateur a un hash unique, même avec le même mot de passe.

### Déclarations Officielles

- **2017** : PostgreSQL annonce que MD5 est considéré comme non sécurisé  
- **PostgreSQL 10 (2017)** : Introduction de SCRAM-SHA-256  
- **PostgreSQL 14 (2021)** : SCRAM-SHA-256 devient la méthode par défaut  
- **Aujourd'hui** : MD5 est **officiellement déprécié** mais toujours supporté pour la rétrocompatibilité

⚠️ **PostgreSQL recommande fortement** de ne plus utiliser MD5 pour les nouvelles installations et de migrer les systèmes existants.

---

## 🟢 Qu'est-ce que SCRAM-SHA-256 ?

### Définition

**SCRAM-SHA-256** signifie :
- **SCRAM** : Salted Challenge Response Authentication Mechanism  
- **SHA-256** : Secure Hash Algorithm 256-bit

C'est un mécanisme d'authentification moderne standardisé (RFC 7677) et considéré comme **hautement sécurisé**.

### Pourquoi SCRAM-SHA-256 est-il Meilleur ?

#### 1. **Algorithme Moderne et Robuste**

SHA-256 fait partie de la famille SHA-2, conçue par la NSA en 2001 et largement utilisée dans :
- Les certificats SSL/TLS
- La blockchain Bitcoin
- Les signatures numériques

**Niveau de sécurité** : Aucune collision connue à ce jour, considéré comme sûr jusqu'en 2030 minimum.

#### 2. **Salt Automatique et Unique**

Chaque mot de passe reçoit automatiquement un **salt aléatoire unique** lors du stockage.

```
Utilisateur : alice, Mot de passe : "MonMotDePasse"  
Salt aléatoire généré : "a1b2c3d4e5f6..."  
Hash final stocké : SCRAM-SHA-256$4096:a1b2c3d4e5f6...$hash_complexe  
```

**Avantage** : Même avec le même mot de passe, chaque utilisateur a un hash totalement différent.

#### 3. **Itérations Multiples (Key Stretching)**

SCRAM-SHA-256 applique l'algorithme de hachage **plusieurs milliers de fois** (par défaut 4096 itérations dans PostgreSQL).

**Principe :**
```
Mot de passe : "password"  
Itération 1 : hash1 = SHA256(password + salt)  
Itération 2 : hash2 = SHA256(hash1)  
Itération 3 : hash3 = SHA256(hash2)  
...
Itération 4096 : hash_final = SHA256(hash4095)
```

**Avantage** : Rend les attaques par force brute **extrêmement lentes**. Un attaquant doit effectuer 4096 calculs pour tester un seul mot de passe au lieu d'un seul.

#### 4. **Protection contre les Rainbow Tables**

Grâce au salt unique, les Rainbow Tables deviennent **inutiles** :
- Il faudrait créer une Rainbow Table différente pour chaque utilisateur
- Cela rendrait l'attaque complètement impraticable

#### 5. **Mécanisme Challenge-Response**

SCRAM utilise un système de **défi-réponse** qui protège contre les attaques "man-in-the-middle".

**Fonctionnement simplifié :**
```
1. Client : "Je suis alice"
2. Serveur : "Prouve-le ! Voici un défi aléatoire : X"
3. Client : Calcule une réponse basée sur le mot de passe et X
4. Serveur : Vérifie la réponse sans jamais transmettre le mot de passe
```

**Avantage** : Le mot de passe n'est **jamais transmis** sur le réseau, même chiffré.

#### 6. **Résistance aux Attaques par Timing**

SCRAM-SHA-256 est conçu pour être résistant aux **attaques par analyse temporelle** (timing attacks) où un attaquant essaie de deviner le mot de passe en mesurant le temps de réponse.

### Comparaison Visuelle

```
╔════════════════════════════════════════════════════════════════╗
║                    MD5 vs SCRAM-SHA-256                        ║
╠════════════════════════════════════════════════════════════════╣
║  Critère              │  MD5          │  SCRAM-SHA-256         ║
╠═══════════════════════╪═══════════════╪════════════════════════╣
║  Année de création    │  1991         │  2009 (SCRAM)          ║
║  Salt automatique     │  ❌ Non       │  ✅ Oui (unique)       ║
║  Itérations multiples │  ❌ 1 seule   │  ✅ 4096 par défaut    ║
║  Résistance collisions│  ❌ Faible    │  ✅ Très forte         ║
║  Vitesse attaque GPU  │  ⚠️ Très rapide│  ✅ Très lente        ║
║  Rainbow Tables       │  ❌ Vulnérable│  ✅ Protégé            ║
║  Challenge-Response   │  ❌ Non       │  ✅ Oui                ║
║  Standardisé (RFC)    │  ✅ Oui       │  ✅ Oui (RFC 7677)     ║
║  Recommandé en 2025   │  ❌ Déprécié  │  ✅ Standard actuel    ║
╚════════════════════════════════════════════════════════════════╝
```

---

## 🔄 Comprendre la Migration de MD5 vers SCRAM-SHA-256

### Pourquoi Migrer ?

1. **Sécurité** : MD5 n'est plus considéré comme sûr  
2. **Conformité** : De nombreuses normes de sécurité (PCI-DSS, HIPAA) interdisent MD5  
3. **Bonnes pratiques** : Suivre les recommandations officielles PostgreSQL  
4. **Pérennité** : MD5 pourrait être complètement supprimé dans les futures versions

### Quand Migrer ?

✅ **Migrez dès que possible si :**
- Vous utilisez encore MD5 en production
- Vous stockez des données sensibles
- Votre système est accessible depuis Internet
- Vous devez respecter des normes de conformité

⚠️ **Planifiez la migration si :**
- Vous avez de nombreux utilisateurs
- Vous avez des applications clientes anciennes
- Vous utilisez des outils tiers qui pourraient ne pas supporter SCRAM

### Les Défis de la Migration

#### 1. **Incompatibilité des Hashs**

Les hashs MD5 et SCRAM-SHA-256 sont **incompatibles** :

```
Hash MD5 :          md5abc123def456...  
Hash SCRAM-SHA-256: SCRAM-SHA-256$4096:salt$hash...  
```

**Conséquence** : Il est **impossible** de convertir directement un hash MD5 en hash SCRAM-SHA-256 sans connaître le mot de passe original.

#### 2. **Nécessité de Réinitialiser les Mots de Passe**

Deux approches possibles :

**Approche A : Réinitialisation Manuelle**
- Les utilisateurs doivent définir un nouveau mot de passe
- Peut être contraignant pour de nombreux utilisateurs

**Approche B : Migration Automatique lors de la Connexion**
- Les utilisateurs se connectent normalement avec leur mot de passe actuel
- Lors de la première connexion réussie, le système migre automatiquement vers SCRAM
- Transparent pour l'utilisateur

#### 3. **Compatibilité des Clients**

Tous les clients PostgreSQL ne supportent pas SCRAM-SHA-256 :

**Support SCRAM-SHA-256 :**
- ✅ **psql** : depuis PostgreSQL 10  
- ✅ **libpq** : depuis PostgreSQL 10  
- ✅ **psycopg2** (Python) : version 2.8+ (2019)  
- ✅ **psycopg3** (Python) : toutes versions  
- ✅ **node-postgres** : version 7.0+ (2017)  
- ✅ **JDBC** : version 42.2.0+ (2017)  
- ✅ **Npgsql** (.NET) : version 3.2+ (2017)

❌ **Clients anciens (pré-2017)** : peuvent ne pas supporter SCRAM

### Stratégies de Migration

#### Stratégie 1 : Migration Complète Immédiate

**Principe :**
- Basculer tout le système vers SCRAM-SHA-256 en une seule fois
- Réinitialiser tous les mots de passe

**Avantages :**
- ✅ Sécurité maximale immédiate  
- ✅ Simplicité de gestion (un seul système)

**Inconvénients :**
- ❌ Interruption de service possible  
- ❌ Contraignant pour les utilisateurs

**Quand l'utiliser :**
- Petit nombre d'utilisateurs
- Possibilité de coordination
- Système non critique temporairement

#### Stratégie 2 : Migration Progressive (Cohabitation)

**Principe :**
- Autoriser temporairement MD5 ET SCRAM-SHA-256
- Migrer les utilisateurs progressivement
- Désactiver MD5 une fois tous les utilisateurs migrés

**Configuration `pg_hba.conf` en mode mixte :**
```
# Autoriser MD5 et SCRAM temporairement
host    all    all    0.0.0.0/0    scram-sha-256,md5
```

**Avantages :**
- ✅ Pas d'interruption de service  
- ✅ Migration en douceur  
- ✅ Temps d'adaptation pour les utilisateurs

**Inconvénients :**
- ❌ Période de transition avec MD5 toujours actif  
- ❌ Gestion plus complexe temporairement

**Quand l'utiliser :**
- Grand nombre d'utilisateurs
- Système en production critique
- Applications clientes multiples

#### Stratégie 3 : Migration Automatique Transparente

**Principe :**
- Les utilisateurs MD5 se connectent normalement
- Lors de la première connexion, le système migre automatiquement vers SCRAM
- Après quelques semaines/mois, désactiver MD5

**Mécanisme :**
```
1. Utilisateur se connecte avec mot de passe MD5
2. PostgreSQL vérifie le mot de passe (hash MD5)
3. Si succès, PostgreSQL régénère automatiquement un hash SCRAM
4. Prochaine connexion : utilisation de SCRAM directement
```

**Avantages :**
- ✅ Totalement transparent pour l'utilisateur  
- ✅ Aucune action requise des utilisateurs  
- ✅ Migration automatique et progressive

**Inconvénients :**
- ❌ Nécessite du développement (trigger ou script)  
- ❌ Fenêtre de vulnérabilité jusqu'à migration complète

**Quand l'utiliser :**
- Très grand nombre d'utilisateurs
- Système avec connexions fréquentes
- Ressources de développement disponibles

---

## 📋 Processus de Migration (Vue d'Ensemble)

### Phase 1 : Préparation

1. **Audit des clients existants**
   - Vérifier que tous les drivers/clients supportent SCRAM
   - Identifier les applications à mettre à jour

2. **Inventaire des utilisateurs**
   - Lister tous les utilisateurs PostgreSQL
   - Identifier ceux utilisant MD5

3. **Plan de communication**
   - Informer les utilisateurs de la migration
   - Prévoir un support en cas de problème

4. **Environnement de test**
   - Tester la migration dans un environnement de pré-production
   - Valider le bon fonctionnement des applications

### Phase 2 : Configuration PostgreSQL

1. **Modifier `postgresql.conf`**
   ```
   # Définir SCRAM comme méthode par défaut
   password_encryption = 'scram-sha-256'
   ```

2. **Modifier `pg_hba.conf` (mode transition)**
   ```
   # Avant (MD5 uniquement)
   host    all    all    0.0.0.0/0    md5

   # Pendant transition (MD5 + SCRAM)
   host    all    all    0.0.0.0/0    scram-sha-256,md5

   # Après migration (SCRAM uniquement)
   host    all    all    0.0.0.0/0    scram-sha-256
   ```

3. **Recharger la configuration**
   ```
   # Rechargement sans interruption
   SELECT pg_reload_conf();
   ```

### Phase 3 : Migration des Utilisateurs

#### Option A : Réinitialisation Manuelle

**Concept :**
```sql
-- Pour chaque utilisateur, définir un nouveau mot de passe
-- PostgreSQL utilisera automatiquement SCRAM (grâce à password_encryption)
ALTER USER alice PASSWORD 'nouveau_mot_de_passe_secure';
```

**Vérification du type de hash :**
```sql
-- Vérifier qu'un utilisateur utilise SCRAM
SELECT rolname,
       CASE
         WHEN rolpassword LIKE 'md5%' THEN 'MD5'
         WHEN rolpassword LIKE 'SCRAM-SHA-256%' THEN 'SCRAM-SHA-256'
         ELSE 'Autre'
       END AS password_type
FROM pg_authid  
WHERE rolname = 'alice';  
```

#### Option B : Migration Automatique

**Concept :**
- L'utilisateur se connecte avec son mot de passe actuel (MD5)
- Un trigger ou une fonction capture le mot de passe lors de la connexion
- Le système régénère automatiquement un hash SCRAM

⚠️ **Note** : Cette approche nécessite du développement personnalisé (extension PostgreSQL ou logique applicative).

### Phase 4 : Validation et Monitoring

1. **Vérifier la migration de tous les utilisateurs**
   ```sql
   -- Lister les utilisateurs encore en MD5
   SELECT rolname
   FROM pg_authid
   WHERE rolpassword LIKE 'md5%';
   ```

2. **Tester les connexions**
   - Vérifier que tous les utilisateurs peuvent se connecter
   - Valider les applications clientes

3. **Surveiller les logs**
   - Vérifier qu'il n'y a pas d'erreurs d'authentification
   - Identifier les éventuels clients incompatibles

### Phase 5 : Finalisation

1. **Désactiver MD5 dans `pg_hba.conf`**
   ```
   # Retirer md5 de la liste
   host    all    all    0.0.0.0/0    scram-sha-256
   ```

2. **Recharger la configuration**
   ```sql
   SELECT pg_reload_conf();
   ```

3. **Vérification finale**
   - Tous les utilisateurs sont en SCRAM-SHA-256
   - Aucune connexion MD5 autorisée
   - Documentation mise à jour

---

## 🛡️ Bonnes Pratiques de Sécurité

### 1. **Utiliser SCRAM-SHA-256 Partout**

```
✅ FAIRE :
- Nouvelles installations : SCRAM-SHA-256 par défaut
- Systèmes existants : Migrer dès que possible

❌ NE PAS FAIRE :
- Utiliser MD5 pour "faciliter" l'installation
- Reporter la migration indéfiniment
```

### 2. **Augmenter le Nombre d'Itérations (Pour Experts)**

Par défaut, PostgreSQL utilise 4096 itérations. Vous pouvez augmenter ce nombre pour plus de sécurité (au prix de performances CPU légèrement réduites).

**Contexte :**
- Serveurs puissants : 8192 ou 16384 itérations
- Données ultra-sensibles : jusqu'à 32768 itérations

⚠️ **Attention** : Plus d'itérations = plus de CPU utilisé lors des connexions

### 3. **Imposer des Mots de Passe Forts**

Même avec SCRAM-SHA-256, un mot de passe faible reste vulnérable.

**Recommandations :**
- Longueur minimale : 12 caractères
- Complexité : majuscules, minuscules, chiffres, symboles
- Utiliser un gestionnaire de mots de passe
- Éviter les mots du dictionnaire

### 4. **Combiner avec SSL/TLS**

SCRAM-SHA-256 protège les mots de passe, mais les données transitent toujours sur le réseau.

**Protection complète :**
```
SCRAM-SHA-256 : Protège l'authentification
       +
  SSL/TLS     : Chiffre toutes les communications
       =
Sécurité Maximale
```

### 5. **Auditer Régulièrement**

```sql
-- Script de vérification mensuelle
SELECT
  rolname,
  CASE
    WHEN rolpassword LIKE 'SCRAM-SHA-256%' THEN '✅ SCRAM'
    WHEN rolpassword LIKE 'md5%' THEN '⚠️ MD5 (à migrer)'
    ELSE '❓ Autre'
  END AS auth_method,
  rolconnlimit,
  rolvaliduntil
FROM pg_authid  
WHERE rolcanlogin = true  
ORDER BY rolname;  
```

---

## ⚠️ Pièges et Erreurs Courantes

### Erreur #1 : Oublier de Mettre à Jour `pg_hba.conf`

**Symptôme :**
```
psql: error: FATAL: password authentication failed for user "alice"
```

**Cause :**
`pg_hba.conf` autorise uniquement MD5, mais l'utilisateur a un hash SCRAM.

**Solution :**
```
# Dans pg_hba.conf, ajouter scram-sha-256
host    all    all    0.0.0.0/0    scram-sha-256
```

### Erreur #2 : Driver Client Incompatible

**Symptôme :**
```
ERROR: authentication method "scram-sha-256" not supported
```

**Cause :**
Le driver client est trop ancien (pré-2017).

**Solution :**
- Mettre à jour le driver vers une version récente
- Si impossible, utiliser temporairement MD5 en mode mixte

### Erreur #3 : Ne Pas Recharger la Configuration

**Symptôme :**
Les modifications dans `pg_hba.conf` ou `postgresql.conf` ne sont pas prises en compte.

**Solution :**
```sql
-- Recharger la configuration
SELECT pg_reload_conf();

-- OU depuis le shell
pg_ctl reload
```

### Erreur #4 : Confondre `password_encryption` et `pg_hba.conf`

**Important à comprendre :**

- **`password_encryption`** (postgresql.conf) : Définit comment les **nouveaux** mots de passe sont **stockés**  
- **`pg_hba.conf`** : Définit quelles méthodes d'authentification sont **acceptées** lors de la **connexion**

**Exemple de confusion :**
```
# postgresql.conf
password_encryption = 'scram-sha-256'  # Nouveaux hash en SCRAM

# pg_hba.conf
host  all  all  0.0.0.0/0  md5  # Mais seules les connexions MD5 sont autorisées !
```

**Résultat :** Les nouveaux utilisateurs auront un hash SCRAM mais ne pourront pas se connecter car `pg_hba.conf` n'autorise que MD5 !

**Correction :**
```
# pg_hba.conf
host  all  all  0.0.0.0/0  scram-sha-256,md5  # Autoriser les deux
```

---

## 📊 Tableau Récapitulatif de Migration

| Étape | Action | Fichier Concerné | Impact | Interruption |
|-------|--------|------------------|--------|--------------|
| 1 | Vérifier compatibilité clients | N/A | Aucun | Non |
| 2 | Modifier `password_encryption` | `postgresql.conf` | Futurs utilisateurs | Non |
| 3 | Ajouter SCRAM à `pg_hba.conf` | `pg_hba.conf` | Autorise SCRAM | Non |
| 4 | Recharger configuration | `SELECT pg_reload_conf()` | Applique les changements | Non |
| 5 | Migrer les utilisateurs | SQL (`ALTER USER`) | Utilisateurs individuels | Non |
| 6 | Retirer MD5 de `pg_hba.conf` | `pg_hba.conf` | Tous les utilisateurs MD5 | Possible |
| 7 | Recharger configuration | `SELECT pg_reload_conf()` | MD5 désactivé | Non |

---

## 🎯 Cas d'Usage Pratiques

### Cas 1 : Startup avec Nouvelle Installation

**Contexte :** Nouvelle application, nouvelle base de données PostgreSQL 18.

**Recommandation :**
```
✅ Utiliser SCRAM-SHA-256 dès le départ
✅ Ne jamais activer MD5
✅ Configurer SSL/TLS en même temps
```

**Configuration :**
```
# postgresql.conf
password_encryption = 'scram-sha-256'

# pg_hba.conf
hostssl    all    all    0.0.0.0/0    scram-sha-256
```

### Cas 2 : PME avec 50 Utilisateurs

**Contexte :** PostgreSQL existant avec MD5, 50 utilisateurs internes.

**Stratégie recommandée :** Migration Progressive

**Planning :**
```
Semaine 1 : Communication et préparation  
Semaine 2 : Mode mixte (MD5 + SCRAM)  
Semaine 3-4 : Migration des utilisateurs par département  
Semaine 5 : Vérification et tests  
Semaine 6 : Désactivation MD5  
```

### Cas 3 : Plateforme SaaS avec Milliers d'Utilisateurs

**Contexte :** Application SaaS, milliers d'utilisateurs, 24/7 critique.

**Stratégie recommandée :** Migration Automatique Transparente

**Approche :**
1. Déployer un mécanisme de migration automatique lors de la connexion  
2. Activer le mode mixte (MD5 + SCRAM)  
3. Laisser les utilisateurs migrer naturellement pendant 3-6 mois  
4. Identifier et contacter les utilisateurs restants  
5. Désactiver MD5 avec communication préalable

---

## 🧠 Points Clés à Retenir

1. **MD5 est obsolète** : Vulnérable aux attaques modernes, officiellement déprécié  
2. **SCRAM-SHA-256 est le standard** : Recommandé par PostgreSQL depuis 2021  
3. **Sécurité multi-couches** : Salt unique + itérations multiples + challenge-response  
4. **Migration nécessaire** : Impossible de convertir directement les hashs MD5  
5. **Stratégies multiples** : Choisir selon la taille et la criticité du système  
6. **Compatibilité clients** : Vérifier que tous les drivers supportent SCRAM (2017+)  
7. **Configuration double** : `password_encryption` (stockage) + `pg_hba.conf` (connexion)  
8. **Cohabitation temporaire possible** : MD5 et SCRAM peuvent coexister pendant la transition

---

## 🔍 Vérifications Post-Migration

### Checklist de Validation

```sql
-- ✅ 1. Tous les utilisateurs sont en SCRAM
SELECT COUNT(*) FROM pg_authid  
WHERE rolcanlogin = true AND rolpassword LIKE 'md5%';  
-- Résultat attendu : 0

-- ✅ 2. password_encryption est configuré
SHOW password_encryption;
-- Résultat attendu : scram-sha-256

-- ✅ 3. pg_hba.conf n'autorise plus MD5
-- Vérifier manuellement le fichier

-- ✅ 4. Les connexions fonctionnent
-- Tester avec psql et les applications

-- ✅ 5. Logs sans erreur d'authentification
-- Vérifier les logs PostgreSQL
```

---

## 🚀 Pour Aller Plus Loin

### Sujets Connexes

Dans les sections suivantes du tutoriel :

- **16.2** : Configuration détaillée de `pg_hba.conf`  
- **16.4** : Gestion des autorisations (`GRANT`/`REVOKE`)  
- **16.7** : SSL/TLS et chiffrement des connexions  
- **16.8** : Nouveautés PostgreSQL 18 - OAuth 2.0 et TLS 1.3

### Ressources Externes

- **RFC 7677** : Spécification SCRAM-SHA-256  
- **PostgreSQL Documentation** : [Password Authentication](https://www.postgresql.org/docs/current/auth-password.html)  
- **OWASP** : Bonnes pratiques de stockage de mots de passe  
- **NIST Guidelines** : Standards de sécurité des mots de passe

---

## 📚 Glossaire

| Terme | Définition |
|-------|------------|
| **Hash** | Empreinte numérique unique d'une donnée (ici, un mot de passe) |
| **Salt** | Valeur aléatoire ajoutée au mot de passe avant hachage |
| **Collision** | Deux entrées différentes produisant le même hash |
| **Rainbow Table** | Base de données précalculée de hashs pour accélérer les attaques |
| **Key Stretching** | Technique consistant à appliquer le hachage de nombreuses fois |
| **Challenge-Response** | Mécanisme où le serveur envoie un défi, le client répond avec une preuve |
| **Itération** | Une application de l'algorithme de hachage |

---


⏭️ [Gestion des autorisations (GRANT/REVOKE)](/16-administration-configuration-securite/04-gestion-des-autorisations.md)
