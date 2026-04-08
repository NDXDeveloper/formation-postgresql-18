🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4. Les Objets de la Base de Données (DDL)

## Introduction au Chapitre

Bienvenue dans ce chapitre fondamental qui va transformer votre compréhension de PostgreSQL ! Jusqu'à présent, nous avons exploré les concepts théoriques et l'architecture du système. Maintenant, nous allons entrer dans le vif du sujet : **comment créer et structurer vos données dans PostgreSQL**.

Ce chapitre est consacré au **DDL (Data Definition Language)**, le langage de définition des données. C'est avec le DDL que vous allez **concevoir** la structure de vos bases de données, créer des tables, définir des types de données, et établir les règles qui garantissent la cohérence de vos informations.

---

## Qu'est-ce que le DDL ?

### Définition

Le **DDL (Data Definition Language)** est l'ensemble des commandes SQL qui permettent de :

- **Créer** des objets de base de données (tables, schémas, index, types, etc.)  
- **Modifier** la structure de ces objets  
- **Supprimer** des objets devenus inutiles

Le DDL s'occupe donc de la **structure** et de l'**architecture** de vos données, pas de leur contenu. C'est un peu comme dessiner les plans d'une maison avant de la construire et de la meubler.

### Les Principales Commandes DDL

Vous allez régulièrement utiliser ces quatre commandes principales :

#### 1. **CREATE**
Crée un nouvel objet dans la base de données.

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);
```

#### 2. **ALTER**
Modifie un objet existant.

```sql
ALTER TABLE utilisateurs  
ADD COLUMN email VARCHAR(255);  
```

#### 3. **DROP**
Supprime un objet de la base de données.

```sql
DROP TABLE utilisateurs_temp;
```

#### 4. **TRUNCATE**
Vide une table de toutes ses données (techniquement un mix DDL/DML).

```sql
TRUNCATE TABLE logs;
```

### DDL vs DML vs DQL

Il est important de comprendre la distinction entre les différents types de commandes SQL :

| Catégorie | Signification | Rôle | Exemples de Commandes |
|-----------|---------------|------|----------------------|
| **DDL** | Data Definition Language | Définir la **structure** | CREATE, ALTER, DROP |
| **DML** | Data Manipulation Language | Manipuler les **données** | INSERT, UPDATE, DELETE |
| **DQL** | Data Query Language | Interroger les **données** | SELECT |
| **DCL** | Data Control Language | Gérer les **permissions** | GRANT, REVOKE |
| **TCL** | Transaction Control Language | Gérer les **transactions** | BEGIN, COMMIT, ROLLBACK |

**Analogie simple :**
- **DDL** = Construire les étagères et les tiroirs d'une bibliothèque  
- **DML** = Ranger, déplacer ou retirer des livres  
- **DQL** = Chercher et consulter des livres  
- **DCL** = Décider qui peut accéder à quelles étagères  
- **TCL** = Garantir que tous les livres sont bien rangés avant de partir

Dans ce chapitre, nous nous concentrons exclusivement sur le **DDL**.

---

## Pourquoi le DDL est-il Crucial ?

### 1. La Fondation de Votre Projet

Le DDL définit **l'architecture** de votre base de données. Une bonne conception initiale vous fera gagner des mois de travail et d'optimisation plus tard. Une mauvaise conception peut rendre votre application lente, difficile à maintenir, et coûteuse à faire évoluer.

**Exemple concret :**
- Une table mal conçue sans index appropriés peut transformer une requête de 10ms en 10 secondes
- Des types de données inappropriés peuvent gaspiller de l'espace disque et ralentir les performances
- Des contraintes manquantes peuvent conduire à des données incohérentes et corrompues

### 2. La Qualité des Données

Le DDL vous permet de définir des **règles de validation** au niveau de la base de données :

- Garantir qu'un email est unique
- S'assurer qu'un prix est toujours positif
- Empêcher la suppression d'un utilisateur si des commandes lui sont associées

Ces règles, appelées **contraintes**, sont votre première ligne de défense contre les données invalides.

### 3. La Documentation Vivante

Votre schéma de base de données (la structure définie par le DDL) est une **documentation vivante** de votre modèle de données. En examinant les tables, leurs colonnes et leurs relations, un développeur peut comprendre la logique métier de votre application.

### 4. La Performance

Les choix que vous faites au niveau du DDL ont un impact direct sur la performance :

- Choix des types de données (INTEGER vs BIGINT, TEXT vs VARCHAR)
- Définition des index
- Partitionnement des tables
- Utilisation de types spécialisés (JSONB, ARRAY, UUID)

---

## Les Objets de Base de Données dans PostgreSQL

PostgreSQL est un SGBD **objet-relationnel**, ce qui signifie qu'il supporte une grande variété d'objets de base de données. Voici les principaux que nous allons explorer dans ce chapitre :

### Objets Fondamentaux

#### 1. **Database (Base de Données)**
Le conteneur de plus haut niveau pour vos données. Chaque instance PostgreSQL peut héberger plusieurs bases de données complètement isolées.

#### 2. **Schema (Schéma)**
Un namespace (espace de noms) qui organise les objets à l'intérieur d'une base de données. C'est comme des dossiers dans un système de fichiers.

#### 3. **Table**
L'objet central qui stocke vos données sous forme de lignes et de colonnes. C'est l'équivalent d'une feuille Excel structurée.

#### 4. **Column (Colonne)**
Un champ dans une table, avec un nom et un type de données. Par exemple : `nom VARCHAR(100)`.

#### 5. **Constraint (Contrainte)**
Une règle qui garantit l'intégrité des données : clés primaires, clés étrangères, contraintes UNIQUE, CHECK, NOT NULL.

### Objets Complémentaires

#### 6. **Sequence (Séquence)**
Un générateur de nombres automatiques, souvent utilisé pour les identifiants.

#### 7. **Index**
Une structure de données qui accélère les recherches (nous les verrons en détail dans les chapitres avancés).

#### 8. **View (Vue)**
Une requête stockée qui se comporte comme une table virtuelle.

#### 9. **Materialized View (Vue Matérialisée)**
Une vue dont les résultats sont physiquement stockés (cache de requête).

#### 10. **Domain (Domaine)**
Un type de données personnalisé basé sur un type existant, avec des contraintes spécifiques.

#### 11. **Type (Type Personnalisé)**
PostgreSQL permet de créer vos propres types de données (ENUM, COMPOSITE, etc.).

#### 12. **Function (Fonction) et Procedure (Procédure)**
Du code réutilisable stocké dans la base de données (nous les verrons dans les chapitres avancés).

#### 13. **Trigger (Déclencheur)**
Une action automatique exécutée en réponse à certains événements (INSERT, UPDATE, DELETE).

---

## L'Approche de Ce Chapitre

Ce chapitre est structuré de manière progressive pour vous accompagner du plus simple au plus avancé :

### Phase 1 : Comprendre la Hiérarchie (Section 4.1)
Vous allez d'abord comprendre comment les objets s'organisent de manière hiérarchique dans PostgreSQL : Instance → Database → Schema → Table.

### Phase 2 : Maîtriser les Schémas (Section 4.2)
Les schémas sont un concept unique et puissant de PostgreSQL. Vous apprendrez à les utiliser pour organiser vos tables.

### Phase 3 : Créer des Tables (Section 4.3)
Nous entrerons dans le vif du sujet avec la commande CREATE TABLE et tous ses détails.

### Phase 4 : Les Types de Données (Section 4.4)
Un tour d'horizon complet des types de données disponibles dans PostgreSQL, des plus basiques (INTEGER, TEXT) aux plus spécialisés (JSONB, UUID, ARRAY).

### Phase 5 : Séquences et Génération Automatique (Section 4.5)
Comment générer automatiquement des identifiants et des numéros de série.

### Phase 6 : Types Personnalisés (Section 4.6)
Créer vos propres types de données pour modéliser des concepts métier spécifiques.

### Phase 7 : Modifier et Supprimer (Section 4.7)
Gérer les modifications de schéma avec ALTER et DROP, et comprendre l'impact du verrouillage.

---

## Ce que Vous Allez Apprendre

À la fin de ce chapitre, vous serez capable de :

- ✅ Comprendre l'organisation hiérarchique de PostgreSQL  
- ✅ Créer des bases de données et des schémas  
- ✅ Concevoir des tables avec les types de données appropriés  
- ✅ Définir des contraintes pour garantir l'intégrité des données  
- ✅ Utiliser des séquences pour générer des identifiants automatiquement  
- ✅ Créer des types personnalisés adaptés à vos besoins métier  
- ✅ Modifier la structure de vos tables en toute sécurité  
- ✅ Choisir les bons types de données pour optimiser l'espace et les performances

---

## Quelques Principes Importants à Garder en Tête

### 1. La Simplicité d'Abord

Commencez toujours par une conception simple. N'ajoutez de la complexité que si vous en avez vraiment besoin. Une table avec 50 colonnes est probablement un signe de mauvaise conception.

### 2. La Normalisation (Mais Pas Trop)

Évitez la redondance des données (nous verrons la normalisation dans les chapitres avancés), mais ne sacrifiez pas la performance pour une normalisation parfaite. Parfois, un peu de dénormalisation est utile.

### 3. Penser à Long Terme

Les choix de conception initiaux sont difficiles à changer une fois que vous avez des millions de lignes de données. Prenez le temps de bien réfléchir à votre modèle.

### 4. Documenter Vos Choix

PostgreSQL permet d'ajouter des commentaires sur les tables et colonnes :

```sql
COMMENT ON TABLE utilisateurs IS 'Table stockant les comptes utilisateurs de l''application';  
COMMENT ON COLUMN utilisateurs.email IS 'Email unique de l''utilisateur, utilisé pour la connexion';  
```

Utilisez cette fonctionnalité pour documenter vos décisions de conception.

### 5. Tester Avant de Déployer

Testez toujours vos modifications de schéma sur un environnement de développement ou de staging avant de les appliquer en production.

---

## Conventions de Nommage

Avant de commencer à créer des objets, établissons quelques conventions de nommage courantes (vous pouvez adapter selon vos préférences) :

### Tables
- **Pluriel ou Singulier ?** Les deux approches existent. Choisissez une convention et restez cohérent.
  - Pluriel : `utilisateurs`, `commandes`, `produits`
  - Singulier : `utilisateur`, `commande`, `produit`
- **Casse :** Préférez les minuscules avec underscores (snake_case)  
  - ✅ `utilisateurs`, `details_commande`  
  - ❌ `Utilisateurs`, `detailsCommande`, `UtilisateursDetails`

### Colonnes
- **Descriptives et claires :** `prenom`, `date_naissance`, `montant_total`  
- **Évitez les abréviations obscures :** Préférez `numero_telephone` à `num_tel`  
- **ID de clé primaire :** Généralement `id` ou `nom_table_id`

### Contraintes
- **Clés primaires :** `pk_nom_table` (ex: `pk_utilisateurs`)  
- **Clés étrangères :** `fk_table_colonne` (ex: `fk_commandes_utilisateur`)  
- **Contraintes UNIQUE :** `uk_table_colonne` (ex: `uk_utilisateurs_email`)  
- **Contraintes CHECK :** `ck_table_condition` (ex: `ck_produits_prix_positif`)

### Index
- **Index standards :** `idx_table_colonne` (ex: `idx_utilisateurs_email`)  
- **Index composites :** `idx_table_col1_col2` (ex: `idx_commandes_date_statut`)

---

## Les Outils pour Travailler avec le DDL

### 1. **psql** - Le Client en Ligne de Commande

L'outil officiel PostgreSQL pour exécuter des commandes SQL.

```bash
psql -U utilisateur -d ma_base
```

Commandes utiles dans psql :
- `\dt` : Lister les tables  
- `\d nom_table` : Décrire une table  
- `\dn` : Lister les schémas  
- `\l` : Lister les bases de données

### 2. **pgAdmin** - Interface Graphique

Un outil graphique puissant qui permet de visualiser et manipuler votre base de données sans écrire de SQL.

### 3. **DBeaver** - Client Universel

Un client multi-bases de données très populaire, avec un excellent support PostgreSQL.

### 4. **Scripts SQL et Migrations**

Dans un projet professionnel, les modifications de schéma sont généralement gérées par des outils de migration comme :
- **Flyway** (Java)  
- **Liquibase** (Multi-langages)  
- **Alembic** (Python)  
- **migrate** (Go)  
- **Prisma** (Node.js)

---

## Mode de Pensée : Modéliser Avant de Coder

Avant de commencer à écrire du DDL, prenez le temps de **modéliser** vos données :

### 1. Identifier les Entités
Quels sont les "objets" principaux de votre application ?
- Utilisateurs
- Produits
- Commandes
- Catégories

### 2. Identifier les Attributs
Quelles informations devez-vous stocker pour chaque entité ?
- Utilisateur : nom, prénom, email, date_inscription
- Produit : nom, description, prix, stock

### 3. Identifier les Relations
Comment ces entités sont-elles liées entre elles ?
- Un utilisateur peut passer plusieurs commandes (1-N)
- Une commande contient plusieurs produits (N-N, via une table de jonction)
- Un produit appartient à une catégorie (N-1)

### 4. Dessiner un Diagramme
Utilisez un outil de modélisation (draw.io, Lucidchart, dbdiagram.io) ou simplement du papier pour visualiser votre modèle.

### 5. Traduire en DDL
Seulement ensuite, traduisez votre modèle en commandes CREATE TABLE.

---

## Un Mot sur les Transactions et le DDL

**Point important :** Dans PostgreSQL, contrairement à d'autres SGBD (comme MySQL), les commandes DDL sont **transactionnelles**.

Cela signifie que vous pouvez faire :

```sql
BEGIN;

CREATE TABLE test1 (id INTEGER);  
CREATE TABLE test2 (id INTEGER);  
ALTER TABLE test1 ADD COLUMN nom VARCHAR(100);  

ROLLBACK; -- Annule toutes les créations
```

Cette caractéristique est extrêmement utile pour tester des modifications de schéma en toute sécurité.

**Exception :** Certaines opérations DDL ne peuvent pas être annulées, comme `CREATE DATABASE` et `DROP DATABASE`.

---

## Avertissement : Production vs Développement

Les commandes DDL que vous allez apprendre dans ce chapitre peuvent être exécutées librement dans un environnement de développement. Cependant, en **production**, certaines opérations peuvent :

- Bloquer l'accès à la base de données
- Prendre beaucoup de temps sur de grandes tables
- Causer des interruptions de service

Nous aborderons ces aspects en détail dans la section 4.7 (Verrouillage) et dans les chapitres avancés sur l'administration.

**Règle d'or :** Testez toujours vos modifications de schéma sur un environnement de test avant de les appliquer en production.

---

## Structure du Chapitre

Voici un aperçu des sections que nous allons couvrir :

1. **4.1. Hiérarchie logique** : Instance → Database → Schema → Table  
2. **4.2. Le concept de Schéma** : Namespaces et search_path  
3. **4.3. CREATE TABLE** : Créer votre première table  
4. **4.4. Les types de données** : Du simple INTEGER au complexe JSONB  
5. **4.5. Séquences** : Génération automatique d'identifiants  
6. **4.6. Domaines et types personnalisés** : Créer vos propres types  
7. **4.7. Gestion des modifications** : ALTER, DROP, et verrouillage

---

## Votre Premier Exercice Mental

Avant de plonger dans les détails techniques, prenez un moment pour réfléchir à un projet simple que vous aimeriez créer. Par exemple : un blog, une application de gestion de tâches, ou un catalogue de produits.

Essayez de répondre à ces questions :
1. Quelles sont les 3-5 entités principales ?  
2. Quelles informations devez-vous stocker pour chaque entité ?  
3. Comment ces entités sont-elles reliées ?

Gardez ce projet en tête tout au long du chapitre. Vous pourrez ainsi appliquer mentalement les concepts au fur et à mesure que nous les découvrons.

---

## Conclusion de l'Introduction

Le DDL est le fondement de tout travail avec PostgreSQL. C'est avec ces commandes que vous allez donner vie à votre modèle de données, structurer vos informations, et garantir leur intégrité.

Ce chapitre va vous donner toutes les clés pour créer des bases de données robustes, performantes et évolutives. Prenez le temps de bien comprendre chaque section, car ces connaissances vous accompagneront tout au long de votre carrière de développeur ou d'administrateur de bases de données.

**Prêt à commencer ?** Passons maintenant à la section 4.1 où nous allons explorer la hiérarchie logique des objets dans PostgreSQL : Instance → Database → Schema → Table.

---


⏭️ [Hiérarchie logique : Instance → Database → Schema → Table](/04-objets-de-la-base-de-donnees/01-hierarchie-logique.md)
