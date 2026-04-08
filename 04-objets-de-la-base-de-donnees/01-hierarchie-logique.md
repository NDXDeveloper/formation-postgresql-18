🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1. Hiérarchie logique : Instance → Database → Schema → Table

## Introduction

Dans PostgreSQL, l'organisation des données suit une structure hiérarchique bien définie. Comprendre cette hiérarchie est essentiel pour travailler efficacement avec PostgreSQL. Pensez à cette structure comme à l'organisation de fichiers sur votre ordinateur : vous avez un disque dur, des dossiers, des sous-dossiers, et enfin des fichiers.

Dans PostgreSQL, cette hiérarchie se décompose en quatre niveaux principaux :

```
Instance PostgreSQL
    └── Database (Base de données)
        └── Schema (Schéma)
            └── Table (Table)
```

Parcourons chaque niveau de cette hiérarchie, du plus haut au plus bas.

---

## 1. L'Instance PostgreSQL (Le Serveur)

### Qu'est-ce qu'une instance ?

Une **instance PostgreSQL** représente un serveur de base de données en cours d'exécution. C'est le niveau le plus élevé de la hiérarchie. Une instance est un processus (ou ensemble de processus) qui gère l'accès aux données stockées sur le disque.

### Analogie

Imaginez un immeuble de bureaux. L'instance PostgreSQL, c'est tout l'immeuble. C'est la structure physique qui contient tout le reste.

### Caractéristiques d'une instance

- **Une seule instance par installation** : Généralement, vous avez une instance PostgreSQL qui tourne sur un serveur (bien qu'il soit techniquement possible d'en avoir plusieurs sur le même serveur, avec des ports différents).  
- **Port d'écoute** : Par défaut, PostgreSQL écoute sur le port **5432**.  
- **Processus principal** : Le processus principal s'appelle `postmaster` ou `postgres` selon le contexte.  
- **Fichiers de configuration** : L'instance possède ses propres fichiers de configuration (`postgresql.conf`, `pg_hba.conf`).  
- **Mémoire partagée** : Tous les utilisateurs connectés à cette instance partagent certaines zones mémoire (comme le `shared_buffers`).

### Exemple visuel

```
┌─────────────────────────────────────┐
│   Instance PostgreSQL               │
│   (Serveur sur localhost:5432)      │
│                                     │
│   ┌─────────────────────────────┐   │
│   │   Database 1                │   │
│   │   Database 2                │   │
│   │   Database 3                │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### Ce qu'il faut retenir

> Une instance PostgreSQL = Un serveur de base de données qui peut héberger plusieurs bases de données.

---

## 2. La Database (Base de Données)

### Qu'est-ce qu'une database ?

Une **database** (base de données) est un conteneur logique qui regroupe un ensemble de schémas, tables, fonctions et autres objets liés à une application ou un projet spécifique. C'est le deuxième niveau de la hiérarchie.

### Analogie

Si l'instance est un immeuble, une database est un **étage entier** de cet immeuble. Chaque étage est indépendant des autres et peut avoir sa propre organisation interne.

### Caractéristiques d'une database

- **Isolation** : Les databases sont **isolées** les unes des autres. Vous ne pouvez pas effectuer de requête SQL qui joint directement des tables de deux databases différentes (sauf avec des outils avancés comme les Foreign Data Wrappers).  
- **Connexion unique** : Quand vous vous connectez à PostgreSQL, vous vous connectez toujours à **une seule database à la fois**.  
- **Databases système** : Par défaut, PostgreSQL crée trois databases système :  
  - `postgres` : Database par défaut, utilisée pour les connexions administratives  
  - `template0` : Template immuable pour créer de nouvelles databases  
  - `template1` : Template modifiable pour créer de nouvelles databases

### Cas d'usage typiques

- **Séparation par projet** : Une database pour chaque application (ex: `ecommerce_db`, `blog_db`, `analytics_db`)  
- **Séparation par environnement** : Une database pour le développement, une pour les tests, une pour la production  
- **Séparation par client** : Dans une application multi-tenant, chaque client peut avoir sa propre database

### Exemple de création

```sql
-- Créer une nouvelle database
CREATE DATABASE ma_boutique;

-- Se connecter à cette database (dans psql)
\c ma_boutique

-- Lister toutes les databases
\l
```

### Exemple visuel

```
┌─────────────────────────────────────────────┐
│   Instance PostgreSQL                       │
│                                             │
│   ┌──────────────────┐  ┌────────────────┐  │
│   │ Database: blog   │  │ Database: shop │  │
│   │                  │  │                │  │
│   │  Schema public   │  │  Schema public │  │
│   │  Schema admin    │  │  Schema orders │  │
│   └──────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────┘
```

### Ce qu'il faut retenir

> Une database = Un espace isolé contenant des schémas et des données, dédié à une application ou un projet.

---

## 3. Le Schema (Schéma)

### Qu'est-ce qu'un schema ?

Un **schema** (schéma) est un espace de noms (namespace) à l'intérieur d'une database. C'est un conteneur logique qui regroupe des objets de base de données : tables, vues, fonctions, types, etc. C'est le troisième niveau de la hiérarchie.

### Analogie

Si la database est un étage d'immeuble, un schema est un **bureau** ou une **salle** dans cet étage. Chaque bureau contient ses propres meubles, documents, etc.

### Pourquoi utiliser des schémas ?

Les schémas permettent :

1. **Organisation logique** : Regrouper des objets liés (ex: toutes les tables de facturation dans un schéma `billing`)  
2. **Éviter les conflits de noms** : Vous pouvez avoir une table `users` dans le schéma `public` et une autre table `users` dans le schéma `admin`  
3. **Gestion des permissions** : Attribuer des droits d'accès différents par schéma  
4. **Isolation fonctionnelle** : Séparer différentes parties d'une application

### Le schéma `public`

Par défaut, PostgreSQL crée un schéma appelé **`public`** dans chaque nouvelle database. C'est le schéma par défaut où les objets sont créés si vous ne spécifiez pas de schéma explicitement.

```sql
-- Ces deux commandes sont équivalentes
CREATE TABLE users (id INT);  
CREATE TABLE public.users (id INT);  
```

### Le search_path

PostgreSQL utilise un concept appelé **`search_path`** pour déterminer dans quel(s) schéma(s) chercher les objets quand vous ne spécifiez pas le nom du schéma.

```sql
-- Voir le search_path actuel
SHOW search_path;
-- Résultat typique : "$user", public

-- Modifier le search_path
SET search_path TO mon_schema, public;
```

Avec ce `search_path`, PostgreSQL cherchera d'abord dans `mon_schema`, puis dans `public`.

### Exemples d'utilisation

```sql
-- Créer un nouveau schéma
CREATE SCHEMA ventes;

-- Créer une table dans ce schéma
CREATE TABLE ventes.commandes (
    id SERIAL PRIMARY KEY,
    montant DECIMAL(10, 2)
);

-- Accéder à la table en spécifiant le schéma
SELECT * FROM ventes.commandes;

-- Lister tous les schémas
\dn
```

### Organisation typique avec schémas

Voici un exemple d'organisation courante :

```
Database: ecommerce
│
├── Schema: public (schéma par défaut)
│   ├── Table: products
│   └── Table: categories
│
├── Schema: sales
│   ├── Table: orders
│   ├── Table: order_items
│   └── Table: invoices
│
├── Schema: users
│   ├── Table: accounts
│   ├── Table: profiles
│   └── Table: sessions
│
└── Schema: reporting
    ├── View: sales_summary
    └── View: user_statistics
```

### Exemple visuel

```
┌───────────────────────────────────────────────┐
│   Database: ecommerce                         │
│                                               │
│   ┌─────────────┐  ┌─────────────┐            │
│   │ Schema:     │  │ Schema:     │            │
│   │ public      │  │ sales       │            │
│   │             │  │             │            │
│   │ ┌─────────┐ │  │ ┌─────────┐ │            │
│   │ │ users   │ │  │ │ orders  │ │            │
│   │ │ products│ │  │ │ invoices│ │            │
│   │ └─────────┘ │  │ └─────────┘ │            │
│   └─────────────┘  └─────────────┘            │
└───────────────────────────────────────────────┘
```

### Ce qu'il faut retenir

> Un schema = Un espace de noms permettant d'organiser et de regrouper des objets (tables, vues, fonctions) au sein d'une database.

---

## 4. La Table (Table)

### Qu'est-ce qu'une table ?

Une **table** est la structure fondamentale qui stocke réellement les données dans PostgreSQL. C'est le dernier niveau de notre hiérarchie, celui où les informations sont physiquement enregistrées.

### Analogie

Si le schema est un bureau, une table est comme un **classeur** ou un **fichier Excel** dans ce bureau. Chaque classeur contient des données organisées en lignes et colonnes.

### Structure d'une table

Une table est composée de :

- **Colonnes** (aussi appelées attributs ou champs) : Définissent la structure des données (nom, type, contraintes)  
- **Lignes** (aussi appelées tuples ou enregistrements) : Contiennent les données réelles

### Exemple de table

```sql
-- Créer une table simple
CREATE TABLE public.utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    date_inscription TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer des données
INSERT INTO utilisateurs (nom, email)  
VALUES ('Alice Dupont', 'alice@example.com');  

INSERT INTO utilisateurs (nom, email)  
VALUES ('Bob Martin', 'bob@example.com');  

-- Consulter les données
SELECT * FROM utilisateurs;
```

Résultat :

```
 id |     nom      |        email         |    date_inscription
----+--------------+----------------------+-------------------------
  1 | Alice Dupont | alice@example.com    | 2025-11-19 10:30:00
  2 | Bob Martin   | bob@example.com      | 2025-11-19 10:30:05
```

### Nom qualifié complet (Fully Qualified Name)

Le nom complet d'une table suit cette syntaxe :

```
database.schema.table
```

Cependant, dans une requête SQL, vous ne pouvez pas spécifier la database (puisque vous êtes déjà connecté à une database). Vous utilisez donc :

```
schema.table
```

Exemples :

```sql
-- Forme complète avec schéma
SELECT * FROM public.utilisateurs;

-- Forme courte (utilise le search_path)
SELECT * FROM utilisateurs;

-- Avec un schéma explicite différent
SELECT * FROM ventes.commandes;
```

### Exemple visuel d'une table

```
Table: utilisateurs (dans public)
┌────┬──────────────┬───────────────────┬─────────────────────┐
│ id │     nom      │       email       │  date_inscription   │
├────┼──────────────┼───────────────────┼─────────────────────┤
│ 1  │ Alice Dupont │ alice@example.com │ 2025-11-19 10:30:00 │
│ 2  │ Bob Martin   │ bob@example.com   │ 2025-11-19 10:30:05 │
└────┴──────────────┴───────────────────┴─────────────────────┘
```

### Ce qu'il faut retenir

> Une table = La structure qui contient réellement les données, organisées en lignes et colonnes, au sein d'un schéma.

---

## Récapitulatif de la Hiérarchie

Voici un récapitulatif visuel complet de la hiérarchie :

```
┌───────────────────────────────────────────────────────────┐
│  INSTANCE PostgreSQL (localhost:5432)                     │
│  └─ Serveur de base de données                            │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │  DATABASE: ecommerce                               │   │
│  │  └─ Conteneur logique isolé                        │   │
│  │                                                    │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  SCHEMA: public                              │  │   │
│  │  │  └─ Espace de noms par défaut                │  │   │
│  │  │                                              │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │  TABLE: utilisateurs                   │  │  │   │
│  │  │  │  └─ Structure de stockage de données   │  │  │   │
│  │  │  │                                        │  │  │   │
│  │  │  │  Colonnes: id, nom, email, date        │  │  │   │
│  │  │  │  Lignes: Données réelles des users     │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  │                                              │  │   │
│  │  │  TABLE: produits                             │  │   │
│  │  │  TABLE: commandes                            │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  │                                                    │   │
│  │  SCHEMA: ventes                                    │   │
│  │  SCHEMA: statistiques                              │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  DATABASE: blog                                           │
│  DATABASE: analytics                                      │
└───────────────────────────────────────────────────────────┘
```

---

## Exemple Pratique Complet

Voyons un exemple concret qui illustre toute la hiérarchie :

```sql
-- 1. Se connecter à l'instance (via psql)
psql -h localhost -p 5432 -U postgres

-- 2. Créer une nouvelle DATABASE
CREATE DATABASE entreprise;

-- 3. Se connecter à cette database
\c entreprise

-- 4. Créer plusieurs SCHEMAS
CREATE SCHEMA ressources_humaines;  
CREATE SCHEMA comptabilite;  
CREATE SCHEMA production;  

-- 5. Créer des TABLES dans différents schémas

-- Table dans le schéma ressources_humaines
CREATE TABLE ressources_humaines.employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    poste VARCHAR(100),
    salaire DECIMAL(10, 2)
);

-- Table dans le schéma comptabilite
CREATE TABLE comptabilite.factures (
    id SERIAL PRIMARY KEY,
    numero VARCHAR(50),
    montant DECIMAL(10, 2),
    date_emission DATE
);

-- Table dans le schéma production
CREATE TABLE production.produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    reference VARCHAR(50),
    stock INT
);

-- 6. Insérer des données
INSERT INTO ressources_humaines.employes (nom, prenom, poste, salaire)  
VALUES ('Durand', 'Sophie', 'Développeuse', 45000);  

INSERT INTO comptabilite.factures (numero, montant, date_emission)  
VALUES ('F-2025-001', 1500.00, '2025-11-19');  

INSERT INTO production.produits (nom, reference, stock)  
VALUES ('Ordinateur portable', 'PC-2025-15', 50);  

-- 7. Requêtes avec nom qualifié complet
SELECT * FROM ressources_humaines.employes;  
SELECT * FROM comptabilite.factures;  
SELECT * FROM production.produits;  
```

### Navigation dans la hiérarchie avec psql

```sql
-- Lister toutes les databases de l'instance
\l

-- Se connecter à une database spécifique
\c entreprise

-- Lister tous les schémas de la database actuelle
\dn

-- Lister toutes les tables de tous les schémas
\dt *.*

-- Lister les tables d'un schéma spécifique
\dt ressources_humaines.*

-- Voir la structure d'une table
\d ressources_humaines.employes
```

---

## Points Clés à Retenir

### Hiérarchie en résumé

| Niveau | Nom | Description | Isolation |
|--------|-----|-------------|-----------|
| 1 | **Instance** | Serveur PostgreSQL en cours d'exécution | Processus séparés |
| 2 | **Database** | Conteneur logique pour une application | Forte (pas de requêtes inter-databases) |
| 3 | **Schema** | Espace de noms pour organiser les objets | Moyenne (accessible via search_path) |
| 4 | **Table** | Structure de stockage des données | Aucune (accessible partout dans le schéma) |

### Règles importantes

1. **Connexion à une database** : Vous devez vous connecter à une database spécifique. Vous ne pouvez pas être "connecté à l'instance" sans être dans une database.

2. **Pas de requêtes inter-databases** : Vous ne pouvez pas faire une jointure entre deux tables de databases différentes dans une requête SQL standard.

3. **Schéma par défaut** : Si vous ne spécifiez pas de schéma, PostgreSQL utilise le schéma `public` par défaut (selon votre `search_path`).

4. **Nommage explicite** : Pour plus de clarté, il est recommandé de toujours spécifier le schéma : `schema.table` plutôt que juste `table`.

5. **Organisation logique** : Utilisez les schémas pour organiser votre database de manière logique (par fonctionnalité, par module, par domaine métier).

### Bonnes pratiques

- **Évitez d'avoir trop de databases** : Préférez utiliser plusieurs schémas dans une même database plutôt que plusieurs databases.  
- **Nommez vos schémas de manière descriptive** : `ventes`, `rh`, `production` plutôt que `schema1`, `schema2`.  
- **Documentez votre organisation** : Maintenez une documentation claire de l'organisation de vos databases, schémas et tables.  
- **Utilisez le schéma `public` avec parcimonie** : Pour les gros projets, créez des schémas dédiés plutôt que de tout mettre dans `public`.

---

## Analogie Finale

Pour bien comprendre la hiérarchie complète, voici une dernière analogie :

```
Instance PostgreSQL    =  Immeuble (le bâtiment entier)
    ↓
Database              =  Étage (chaque étage est isolé)
    ↓
Schema                =  Bureau/Salle dans l'étage
    ↓
Table                 =  Classeur dans le bureau
    ↓
Lignes et Colonnes    =  Documents dans le classeur
```

---

## Conclusion

La compréhension de cette hiérarchie est fondamentale pour travailler efficacement avec PostgreSQL. Elle vous permet de :

- **Organiser vos données** de manière logique et maintenable  
- **Gérer les permissions** finement (au niveau database, schéma ou table)  
- **Naviguer** efficacement dans votre système de gestion de base de données  
- **Éviter les conflits** de noms en utilisant les schémas  
- **Planifier votre architecture** de données selon vos besoins

Dans les prochaines sections, nous verrons comment créer et manipuler concrètement ces objets, en commençant par les tables et leurs types de données.

---


⏭️ [Le concept de Schéma (Namespaces) et le search_path](/04-objets-de-la-base-de-donnees/02-concept-de-schema.md)
