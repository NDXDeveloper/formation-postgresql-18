🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.1. Authentification vs Autorisation : Concepts Fondamentaux

## Introduction

Lorsqu'on parle de sécurité dans les bases de données (et dans l'informatique en général), deux concepts reviennent systématiquement : **l'authentification** et **l'autorisation**. Bien que souvent confondus, ces deux mécanismes sont distincts et complémentaires. Comprendre leur différence est essentiel pour sécuriser correctement votre base de données PostgreSQL.

---

## 🔐 Qu'est-ce que l'Authentification ?

### Définition

L'**authentification** répond à la question fondamentale : **"Qui êtes-vous ?"**

C'est le processus par lequel vous **prouvez votre identité** au système. En d'autres termes, c'est la manière dont vous démontrez que vous êtes bien la personne (ou l'application) que vous prétendez être.

### Analogie du Monde Réel

Imaginez que vous arrivez à l'entrée d'un immeuble sécurisé :

- Le gardien vous demande : **"Qui êtes-vous ?"**
- Vous répondez : "Je suis Jean Dupont"
- Le gardien vous demande alors de le **prouver** en montrant votre carte d'identité ou votre badge

C'est exactement ce qu'est l'authentification : **prouver votre identité**.

### Dans PostgreSQL

Dans PostgreSQL, l'authentification se produit lors de la **connexion à la base de données**. Le serveur PostgreSQL demande au client de prouver son identité avant d'autoriser toute action.

**Exemple concret :**
```sql
-- Tentative de connexion depuis la ligne de commande
psql -h localhost -U alice -d ma_base  
Password for user alice: ********  
```

Ici, PostgreSQL demande à l'utilisateur de prouver qu'il est bien "alice" en fournissant un mot de passe.

### Méthodes d'Authentification Courantes

PostgreSQL supporte plusieurs méthodes d'authentification :

1. **Mot de passe** (password, md5, scram-sha-256)
   - L'utilisateur fournit un mot de passe secret
   - C'est la méthode la plus courante

2. **Certificat SSL/TLS** (cert)
   - L'utilisateur présente un certificat numérique
   - Utilisé pour des environnements hautement sécurisés

3. **Trust** (trust)
   - Aucune authentification (⚠️ dangereux en production !)
   - Peut être utilisé en développement local uniquement

4. **LDAP** (ldap)
   - Authentification via un annuaire d'entreprise
   - Centralise la gestion des identités

5. **OAuth 2.0** (nouveauté PostgreSQL 18)
   - Authentification moderne via des fournisseurs tiers
   - Compatible avec Google, Microsoft, etc.

### Ce que l'Authentification NE fait PAS

⚠️ **Important** : L'authentification ne détermine **PAS** ce que vous pouvez faire une fois connecté. Elle vérifie uniquement qui vous êtes, pas ce que vous avez le droit de faire.

---

## 🛡️ Qu'est-ce que l'Autorisation ?

### Définition

L'**autorisation** répond à une question différente : **"Qu'avez-vous le droit de faire ?"**

C'est le processus qui détermine **quelles actions vous êtes autorisé à effectuer** une fois que votre identité a été vérifiée. On parle aussi de **contrôle d'accès** ou de **permissions**.

### Analogie du Monde Réel

Reprenons l'exemple de l'immeuble sécurisé :

- Vous avez **prouvé votre identité** (authentification) avec votre badge
- Le gardien vérifie maintenant : **"À quels étages avez-vous accès ?"**
- Votre badge indique : "Accès autorisé aux étages 1, 2 et 5 uniquement"

C'est l'autorisation : **définir ce que vous pouvez faire**.

### Dans PostgreSQL

Dans PostgreSQL, l'autorisation intervient **après** la connexion réussie. Le système vérifie constamment si vous avez les **droits** nécessaires pour effectuer chaque opération.

**Exemple concret :**
```sql
-- Alice s'est connectée (authentifiée) avec succès
-- Elle tente maintenant de lire la table "employes"

SELECT * FROM employes;
-- ✅ Succès : Alice a le droit SELECT sur cette table

DELETE FROM employes WHERE id = 1;
-- ❌ Erreur : Alice n'a PAS le droit DELETE sur cette table
-- ERROR: permission denied for table employes
```

### Types de Permissions dans PostgreSQL

PostgreSQL gère de nombreux types de permissions :

1. **Permissions sur les tables** :  
   - `SELECT` : Lire les données  
   - `INSERT` : Ajouter de nouvelles lignes  
   - `UPDATE` : Modifier des lignes existantes  
   - `DELETE` : Supprimer des lignes  
   - `TRUNCATE` : Vider une table  
   - `REFERENCES` : Créer des clés étrangères  
   - `TRIGGER` : Créer des triggers

2. **Permissions sur les schémas** :  
   - `USAGE` : Accéder au schéma  
   - `CREATE` : Créer des objets dans le schéma

3. **Permissions sur les bases de données** :  
   - `CONNECT` : Se connecter à la base  
   - `CREATE` : Créer des schémas dans la base  
   - `TEMP` : Créer des tables temporaires

4. **Permissions sur les fonctions** :  
   - `EXECUTE` : Exécuter une fonction ou procédure

### Ce que l'Autorisation NE fait PAS

⚠️ **Important** : L'autorisation ne vérifie **PAS** votre identité. Elle suppose que vous avez déjà été authentifié et se concentre uniquement sur vos droits d'accès.

---

## 🔄 La Relation entre Authentification et Autorisation

### Ordre d'Exécution

L'authentification et l'autorisation fonctionnent **toujours dans cet ordre** :

```
1. AUTHENTIFICATION : "Qui êtes-vous ?"
   ⬇️
   ✅ Identité vérifiée
   ⬇️
2. AUTORISATION : "Qu'avez-vous le droit de faire ?"
   ⬇️
   ✅ Action autorisée ou ❌ Action refusée
```

### Pourquoi les Deux sont Nécessaires

**Imaginez un système sans authentification** :
- N'importe qui pourrait se connecter en prétendant être "admin"
- La sécurité serait totalement compromise

**Imaginez un système sans autorisation** :
- Tous les utilisateurs authentifiés auraient les mêmes droits
- Un simple utilisateur pourrait supprimer toute la base de données !

### Exemple Complet

Voici un scénario complet illustrant les deux concepts :

```sql
-- ÉTAPE 1 : AUTHENTIFICATION
-- L'utilisateur "bob" tente de se connecter
psql -h localhost -U bob -d entreprise  
Password: ********  
-- ✅ PostgreSQL vérifie le mot de passe dans pg_authid
-- ✅ Authentification réussie : Bob est connecté

-- ÉTAPE 2 : AUTORISATION (première requête)
SELECT * FROM salaires;
-- ❌ PostgreSQL vérifie : Bob a-t-il SELECT sur "salaires" ?
-- ❌ Résultat : NON
-- ERROR: permission denied for table salaires

-- ÉTAPE 2 : AUTORISATION (deuxième requête)
SELECT * FROM projets;
-- ✅ PostgreSQL vérifie : Bob a-t-il SELECT sur "projets" ?
-- ✅ Résultat : OUI
-- La requête s'exécute avec succès
```

---

## 📊 Tableau Récapitulatif

| Critère | Authentification | Autorisation |
|---------|------------------|--------------|
| **Question posée** | "Qui êtes-vous ?" | "Qu'avez-vous le droit de faire ?" |
| **Moment** | À la connexion | Après chaque action |
| **Objectif** | Vérifier l'identité | Contrôler l'accès aux ressources |
| **Méthode** | Mot de passe, certificat, token | Permissions, rôles, politiques |
| **Fichier de config (PostgreSQL)** | `pg_hba.conf` | Instructions `GRANT`/`REVOKE` |
| **Analogie** | Badge d'identification | Clés d'accès aux pièces |
| **Échec typique** | "Identifiant ou mot de passe incorrect" | "Permission denied for table X" |

---

## 🎯 Cas d'Usage Pratiques

### Cas 1 : Application Web

**Contexte** : Une application web e-commerce avec PostgreSQL

**Authentification** :
- L'application se connecte à PostgreSQL avec un utilisateur dédié : `app_ecommerce`
- Méthode : SCRAM-SHA-256 (mot de passe sécurisé)

**Autorisation** :
- `app_ecommerce` a les droits :  
  - `SELECT`, `INSERT`, `UPDATE` sur la table `commandes`  
  - `SELECT` uniquement sur la table `produits`  
  - `AUCUN droit` sur la table `utilisateurs_admin`

### Cas 2 : Utilisateurs Multiples

**Contexte** : Trois types d'utilisateurs dans une entreprise

| Utilisateur | Authentification | Autorisation |
|-------------|------------------|--------------|
| **Alice (Développeur)** | Mot de passe SCRAM | `SELECT`, `INSERT`, `UPDATE`, `DELETE` sur les tables de développement |
| **Bob (Analyste)** | Certificat SSL | `SELECT` uniquement sur toutes les tables (lecture seule) |
| **Charlie (Admin DBA)** | LDAP entreprise | Tous les droits (superuser) |

### Cas 3 : Service Automatisé

**Contexte** : Un script de sauvegarde automatique

**Authentification** :
- Utilisateur : `backup_service`
- Méthode : Certificat SSL (pas de mot de passe à stocker)

**Autorisation** :
- Droits : `SELECT` sur toutes les tables
- Droit : `pg_dump` (pour effectuer des sauvegardes)
- Pas de droits en écriture (sécurité)

---

## 🧠 Points Clés à Retenir

1. **Authentification = Identité** : "Qui êtes-vous ?"  
2. **Autorisation = Permissions** : "Que pouvez-vous faire ?"  
3. **Ordre séquentiel** : Toujours authentification d'abord, autorisation ensuite  
4. **Indépendance** : Vous pouvez être authentifié mais ne rien avoir le droit de faire  
5. **Complémentarité** : Les deux sont nécessaires pour une sécurité complète  
6. **Granularité** : PostgreSQL permet un contrôle très fin des autorisations (table, colonne, ligne)

---

## 🚀 Pour Aller Plus Loin

Dans les sections suivantes du tutoriel, nous explorerons :

- **16.2** : Configuration détaillée de l'authentification (`pg_hba.conf`)  
- **16.3** : Migration vers SCRAM-SHA-256 (méthode moderne et sécurisée)  
- **16.4** : Gestion fine des autorisations avec `GRANT` et `REVOKE`  
- **16.5** : Rôles et groupes : simplifier la gestion des permissions  
- **16.6** : Row-Level Security (RLS) : autorisation au niveau des lignes

---

## 📝 Notes pour les Développeurs

### Bonne Pratique #1 : Principe du Moindre Privilège

Donnez toujours **le minimum de droits nécessaires** :

```sql
-- ❌ MAUVAIS : Donner tous les droits
GRANT ALL PRIVILEGES ON DATABASE entreprise TO app_user;

-- ✅ BON : Donner uniquement les droits nécessaires
GRANT SELECT, INSERT, UPDATE ON TABLE commandes TO app_user;
```

### Bonne Pratique #2 : Utilisateurs Dédiés

Créez des utilisateurs spécifiques pour chaque application :

```sql
-- ❌ MAUVAIS : Utiliser le superuser "postgres"
psql -U postgres

-- ✅ BON : Créer un utilisateur dédié avec droits limités
CREATE USER app_web WITH PASSWORD 'mot_de_passe_fort';  
GRANT CONNECT ON DATABASE ma_base TO app_web;  
GRANT USAGE ON SCHEMA public TO app_web;  
GRANT SELECT, INSERT ON TABLE articles TO app_web;  
```

### Bonne Pratique #3 : Séparer les Environnements

Utilisez des comptes différents selon l'environnement :

- **Développement** : Droits larges pour faciliter le travail  
- **Staging** : Droits similaires à la production  
- **Production** : Droits minimaux et strictement contrôlés

---

## ⚠️ Erreurs Courantes à Éviter

### Erreur #1 : Confondre Authentification et Autorisation

```sql
-- Un utilisateur dit : "Je n'arrive pas à me connecter !"
-- Est-ce un problème d'authentification ou d'autorisation ?

-- Si le message est :
"password authentication failed for user alice"
-- ➡️ C'est un problème d'AUTHENTIFICATION

-- Si le message est :
"permission denied for table employes"
-- ➡️ C'est un problème d'AUTORISATION (l'utilisateur est connecté !)
```

### Erreur #2 : Méthode "trust" en Production

```
# pg_hba.conf
# ❌ DANGEREUX : N'importe qui peut se connecter sans mot de passe !
local   all   all   trust

# ✅ SÉCURISÉ : Exiger une authentification
local   all   all   scram-sha-256
```

### Erreur #3 : Donner des Droits de Superuser

```sql
-- ❌ MAUVAIS : Donner tous les pouvoirs
ALTER USER app_user WITH SUPERUSER;

-- ✅ BON : Donner uniquement les droits nécessaires
GRANT SELECT, INSERT, UPDATE ON TABLE ma_table TO app_user;
```

---

## 🎓 Quiz de Compréhension

### Question 1
**Un utilisateur se connecte avec succès mais ne peut pas lire la table "clients". Est-ce un problème d'authentification ou d'autorisation ?**

<details>
<summary>Réponse</summary>

**Autorisation** : L'utilisateur a réussi à se connecter (authentification OK), mais il n'a pas les droits de lecture (SELECT) sur la table "clients".
</details>

### Question 2
**Vous obtenez l'erreur "password authentication failed". Est-ce un problème d'authentification ou d'autorisation ?**

<details>
<summary>Réponse</summary>

**Authentification** : Le système n'a pas pu vérifier l'identité de l'utilisateur. L'utilisateur n'a même pas pu se connecter, donc la question de l'autorisation ne se pose pas encore.
</details>

### Question 3
**Que se passe-t-il si vous avez des droits (autorisation) mais aucune méthode d'authentification configurée ?**

<details>
<summary>Réponse</summary>

**Vous ne pourrez pas vous connecter**. L'authentification vient toujours en premier. Sans pouvoir prouver votre identité, vous ne pourrez jamais utiliser vos droits.
</details>

---

## 📚 Ressources Complémentaires

- **Documentation PostgreSQL** : [Client Authentication](https://www.postgresql.org/docs/current/client-authentication.html)  
- **Documentation PostgreSQL** : [Database Roles](https://www.postgresql.org/docs/current/user-manag.html)  
- **Best Practices** : [PostgreSQL Security Best Practices](https://www.postgresql.org/docs/current/security.html)

---


⏭️ [Configuration de l'authentification (pg_hba.conf)](/16-administration-configuration-securite/02-configuration-authentification.md)
