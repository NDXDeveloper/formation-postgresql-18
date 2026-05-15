🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1. Histoire et Philosophie : Le SGBD « objet-relationnel »

## Introduction

PostgreSQL n'est pas un simple système de gestion de bases de données parmi d'autres. C'est un projet avec une histoire riche, une philosophie unique et une approche distinctive qui le distingue de ses concurrents. Dans ce chapitre, nous allons explorer les origines de PostgreSQL, comprendre ce qui fait sa particularité et découvrir pourquoi on le qualifie de SGBD « objet-relationnel ».

---

## Les Origines : Du Projet POSTGRES à PostgreSQL

### Les Années 1970 : INGRES

L'histoire de PostgreSQL commence bien avant sa création officielle. Dans les années 1970, à l'Université de Californie à Berkeley, le professeur **Michael Stonebraker** et son équipe développent un système de bases de données appelé **INGRES** (*INteractive GRaphics REtrieval System*).

INGRES était l'un des premiers systèmes de gestion de bases de données relationnelles, concurrent direct d'Oracle et de System R d'IBM. Ce projet a posé les bases de nombreux concepts que nous utilisons encore aujourd'hui.

> 🏆 Michael Stonebraker recevra plus tard, en **2014, le Prix Turing** (l'équivalent du Prix Nobel en informatique) pour ses contributions fondamentales aux systèmes de bases de données — INGRES, POSTGRES et plusieurs autres SGBD ont contribué à cette reconnaissance.

### 1986 : Naissance de POSTGRES

Après le succès d'INGRES, Michael Stonebraker et son équipe à Berkeley identifient plusieurs limitations des systèmes relationnels de l'époque. En 1986, ils lancent un nouveau projet ambitieux : **POSTGRES** (*POST inGRES*). Le projet est présenté dans l'article fondateur *« The Design of POSTGRES »* (Stonebraker & Rowe, 1986).

**Les objectifs de POSTGRES étaient révolutionnaires pour l'époque :**

- **Extension du modèle relationnel** : ajouter la possibilité de définir des types de données personnalisés (*Abstract Data Types*, ADT)
- **Support des données complexes** : gérer des données plus sophistiquées que les simples nombres et textes
- **Règles et déclencheurs** : automatiser des actions en réponse à des événements
- **Requêtes sur des données historiques** : voyager dans le temps avec les données (*time travel*)

> 🧩 **Pourquoi ces objectifs ? Le problème du « impedance mismatch »**  
>  
> Dans les années 1980, les développeurs souffraient d'un problème récurrent appelé *« object-relational impedance mismatch »* : leurs langages de programmation devenaient orientés objet (C++, Smalltalk…), mais leurs bases de données restaient strictement relationnelles, avec des types primitifs limités (entiers, chaînes, dates). Pour stocker un objet complexe comme une **adresse postale** ou une **géométrie 3D**, il fallait l'éclater en plusieurs colonnes plates, puis le reconstruire en mémoire — code répétitif, bugs subtils, performances dégradées.  
>  
> POSTGRES proposait une réponse radicale : **rendre la base de données elle-même extensible** pour qu'elle accepte ces objets complexes nativement. C'est ce qui distinguait POSTGRES de tous les SGBD concurrents de l'époque — et qui rend PostgreSQL toujours unique aujourd'hui, avec ses types JSON, géométriques, vectoriels, etc.

Le projet POSTGRES a été développé entre 1986 et 1994 à l'Université de Berkeley, financé par des agences gouvernementales (DARPA, ARO, NSF) et des sponsors privés.

> 📚 **Héritage commercial de POSTGRES**  
>  
> En 1992, Stonebraker fonda **Illustra Information Technologies** pour commercialiser POSTGRES. Illustra fut racheté par **Informix** en 1996, qui intégra son moteur extensible dans Informix Universal Server. **IBM** racheta Informix en 2001, et plusieurs concepts d'extensibilité ont depuis irrigué DB2. La descendance commerciale de POSTGRES est donc présente dans toute l'industrie, en plus du PostgreSQL open source que nous utilisons aujourd'hui.

### 1994-1995 : L'Arrivée du SQL

Initialement, POSTGRES utilisait son propre langage de requête appelé **POSTQUEL**. Bien que puissant, ce langage propriétaire limitait l'adoption du système.

En **1994**, deux étudiants de Berkeley, **Andrew Yu** et **Jolly Chen**, remplacent l'interpréteur POSTQUEL par un interpréteur **SQL** standard. La première version publique de **Postgres95** (v0.01) est diffusée le **5 mai 1995** sur le web.

Cette évolution marque un tournant majeur : Postgres devient compatible avec le standard de l'industrie tout en conservant ses innovations uniques.

### 1996 : PostgreSQL Devient Open Source

En 1996, il devient évident que le nom « Postgres95 » ne survivra pas au passage du millénaire. Le projet est officiellement renommé **PostgreSQL** — un nom qui reflète le lien entre l'ancêtre POSTGRES et l'arrivée du SQL. La prononciation officielle est *« post-GRES-Q-L »* (en quatre syllabes), souvent abrégé en *« Postgres »* à l'oral.

**Événement crucial :** le code source est publié sous une licence open source très permissive (licence de type BSD, appelée aujourd'hui *PostgreSQL License*). Cette décision permet :

- La création d'une communauté mondiale de développeurs
- L'utilisation gratuite dans des projets commerciaux
- Des améliorations rapides grâce aux contributions de milliers de développeurs

### L'Évolution Continue (1996 - Aujourd'hui)

Depuis 1996, PostgreSQL a connu une évolution constante avec une nouvelle version majeure chaque année. Chaque version apporte son lot d'innovations, d'optimisations et de nouvelles fonctionnalités.

**Quelques jalons importants :**

- **PostgreSQL 7.1 (2001)** : introduction du WAL (*Write-Ahead Logging*), garantissant une durabilité robuste
- **PostgreSQL 8.0 (2005)** : support natif Windows, Point-In-Time Recovery (PITR), savepoints
- **PostgreSQL 9.0 (2010)** : réplication en streaming, Hot Standby
- **PostgreSQL 9.4 (2014)** : type JSONB (JSON binaire indexable)
- **PostgreSQL 10 (2017)** : partitionnement déclaratif natif, réplication logique
- **PostgreSQL 11 (2018)** : procédures stockées (CALL, transactions imbriquées), JIT compilation
- **PostgreSQL 12 (2019)** : améliorations massives des performances, colonnes générées (stockées)
- **PostgreSQL 13 (2020)** : déduplication B-Tree, vacuum parallèle
- **PostgreSQL 14 (2021)** : pipeline mode du protocole, améliorations JSON, *predicate locks* en logical replication
- **PostgreSQL 15 (2022)** : MERGE SQL standard, compression LZ4 pour TOAST
- **PostgreSQL 16 (2023)** : parallélisation accrue, réplication logique depuis un standby
- **PostgreSQL 17 (2024)** : *incremental backup*, optimisations VACUUM, MERGE amélioré
- **PostgreSQL 18 (sept. 2025)** : I/O asynchrone (AIO), colonnes virtuelles, OAuth 2.0, UUIDv7, contraintes temporelles, data checksums par défaut

---

## La Philosophie PostgreSQL

### Un Projet Communautaire

PostgreSQL n'appartient à aucune entreprise. C'est un projet **communautaire** géré par le **PostgreSQL Global Development Group** (PGDG), un groupe international de volontaires et d'experts.

**Cette structure apporte plusieurs avantages :**

1. **Indépendance** : Aucune entreprise ne contrôle le projet ou sa direction  
2. **Stabilité** : Le projet ne peut pas être racheté ou abandonné  
3. **Transparence** : Toutes les discussions et décisions sont publiques  
4. **Qualité** : Les contributions sont rigoureusement examinées par des pairs

### Les Valeurs Fondamentales

PostgreSQL est guidé par des principes forts qui influencent chaque décision :

#### 1. **Conformité aux Standards**

PostgreSQL respecte scrupuleusement les standards SQL (ANSI SQL, ISO/IEC 9075). Contrairement à certains concurrents qui prennent des libertés avec la norme, PostgreSQL privilégie la conformité.

**Exemple pratique :**
- Traitement rigoureux des valeurs NULL
- Respect des niveaux d'isolation transactionnelle
- Support complet des types de données SQL standard

#### 2. **Extensibilité**

PostgreSQL est conçu pour être étendu. Vous pouvez ajouter :
- Vos propres types de données
- Vos propres fonctions
- Vos propres opérateurs
- Vos propres langages de programmation

Cette extensibilité permet à PostgreSQL de s'adapter à des besoins très variés sans modifier le cœur du système.

#### 3. **Fiabilité et Intégrité des Données**

La priorité absolue de PostgreSQL est de **ne jamais perdre ou corrompre vos données**.

Cela se traduit par :
- Support complet des propriétés ACID (Atomicité, Cohérence, Isolation, Durabilité)
- Mécanismes de récupération robustes
- Validation stricte des contraintes d'intégrité
- Checksums de données (activés par défaut depuis PostgreSQL 18)

#### 4. **Performance ET Correction**

PostgreSQL ne sacrifie jamais la correction des données pour la performance. Si un choix doit être fait, l'intégrité prime.

Cependant, cela ne signifie pas que PostgreSQL est lent. Au contraire, il est reconnu pour ses excellentes performances, mais jamais au détriment de la fiabilité.

#### 5. **Licence Permissive**

La licence PostgreSQL (similaire à BSD/MIT) est très permissive. Vous pouvez :
- Utiliser PostgreSQL gratuitement, même en production commerciale
- Modifier le code source sans obligation de partager vos modifications
- Intégrer PostgreSQL dans des produits propriétaires

---

## PostgreSQL : Un SGBD « objet-relationnel »

### Qu'est-ce qu'un Modèle Relationnel ?

Avant de comprendre le terme « objet-relationnel », rappelons ce qu'est un modèle **relationnel** classique.

**Un SGBD relationnel organise les données en :**

- **Tables** : Structures à deux dimensions (lignes et colonnes)  
- **Lignes** : Représentent des enregistrements individuels  
- **Colonnes** : Représentent des attributs avec des types de données spécifiques  
- **Relations** : Liens entre les tables via des clés (primaires et étrangères)

**Exemple simple :**

```
Table : Clients
+----+-----------+------------------+
| id | nom       | email            |
+----+-----------+------------------+
| 1  | Dupont    | dupont@email.fr  |
| 2  | Martin    | martin@email.fr  |
+----+-----------+------------------+
```

Les types de données sont limités aux types de base : nombres, textes, dates, etc.

### Les Limites du Modèle Relationnel Pur

Le modèle relationnel classique a certaines limitations :

1. **Types de données rigides** : Difficile de représenter des structures complexes  
2. **Absence d'héritage** : Pas de hiérarchie entre les types  
3. **Données plates** : Difficile de représenter des objets imbriqués  
4. **Extensibilité limitée** : Pas facile d'ajouter de nouveaux types

### Le Modèle Objet : L'Autre Approche

Les bases de données **orientées objet** adoptent une philosophie différente inspirée de la programmation orientée objet :

- **Objets complexes** : Possibilité d'imbriquer des structures  
- **Héritage** : Les objets peuvent hériter de propriétés  
- **Encapsulation** : Regroupement de données et comportements  
- **Polymorphisme** : Un objet peut avoir plusieurs formes

**Problème :** Les bases de données purement orientées objet n'ont jamais vraiment décollé dans l'industrie. Le modèle relationnel reste dominant.

### PostgreSQL : Le Meilleur des Deux Mondes

PostgreSQL se positionne comme un SGBD **objet-relationnel**, combinant les avantages des deux approches :

#### 1. **Base Relationnelle Solide**

PostgreSQL reste fondamentalement un SGBD relationnel :
- Tables, lignes, colonnes
- SQL standard
- Contraintes d'intégrité (clés primaires, étrangères)
- Transactions ACID

#### 2. **Extensions Orientées Objet**

PostgreSQL ajoute des capacités orientées objet :

##### a) **Types de Données Personnalisés**

Vous pouvez créer vos propres types de données adaptés à votre domaine métier. PostgreSQL propose plusieurs niveaux de personnalisation :

**Niveau 1 — `DOMAIN` : un type existant + contraintes**

Le plus simple : étendre un type primitif avec des règles métier.

```sql
-- Définir un domaine « code postal français »
CREATE DOMAIN code_postal_fr AS VARCHAR(5)
    CHECK (VALUE ~ '^[0-9]{5}$');

-- Utiliser ce domaine dans une table
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom TEXT,
    cp code_postal_fr  -- Validation automatique du format !
);

-- Tentative d'insertion invalide
INSERT INTO clients (nom, cp) VALUES ('Dupont', 'ABCDE');
-- ERROR: value for domain code_postal_fr violates check constraint
```

**Niveau 2 — `CREATE TYPE` (composite) : objet métier complet**

```sql
-- Créer un type composite pour une adresse
CREATE TYPE adresse AS (
    rue TEXT,
    ville TEXT,
    code_postal code_postal_fr,  -- Réutilise notre domaine !
    pays TEXT
);

-- Utiliser ce type dans une table
CREATE TABLE clients_v2 (
    id SERIAL PRIMARY KEY,
    nom TEXT,
    adresse adresse  -- Type personnalisé composite
);
```

**Niveau 3 — `CREATE TYPE` (base) : type entièrement nouveau en C**

Pour les performances maximales, on peut écrire un type en C avec ses fonctions d'entrée/sortie, opérateurs, comparateurs. C'est ainsi que sont construits PostGIS (géométries), pgvector (embeddings), hstore (clé-valeur)…

##### b) **Types Composites**

PostgreSQL permet de créer des structures de données complexes, comme des objets imbriqués.

**Exemple :**
- Une colonne peut contenir un tableau de valeurs (type ARRAY)
- Une colonne peut contenir un document JSON complexe (type JSONB)
- Une colonne peut contenir une structure géométrique (type GEOMETRY avec PostGIS)

##### c) **Héritage de Tables**

PostgreSQL supporte l'héritage entre tables (bien que cette fonctionnalité soit moins utilisée aujourd'hui au profit du partitionnement).

**Concept :**
```sql
-- Table parente
CREATE TABLE personnes (
    id SERIAL PRIMARY KEY,
    nom TEXT,
    prenom TEXT
);

-- Table enfant héritant de personnes
CREATE TABLE employes (
    numero_employe INTEGER,
    salaire NUMERIC
) INHERITS (personnes);
```

La table `employes` hérite automatiquement des colonnes de `personnes`.

##### d) **Fonctions et Opérateurs Personnalisés**

Vous pouvez définir vos propres fonctions et opérateurs qui travaillent avec vos types personnalisés.

**Exemple conceptuel :**
- Créer une fonction de calcul de distance entre deux adresses
- Créer un opérateur `@@` pour la recherche plein texte
- Créer une fonction d'agrégation personnalisée

#### 3. **Extensibilité Maximale**

Cette approche objet-relationnelle rend PostgreSQL extrêmement extensible :

**Extensions célèbres :**

- **PostGIS** : Transforme PostgreSQL en base de données géospatiale de classe mondiale  
- **TimescaleDB** : Optimise PostgreSQL pour les séries temporelles  
- **pgvector** : Ajoute la recherche vectorielle pour l'IA et les embeddings  
- **pg_trgm** : Recherche floue et similarité de texte  
- **hstore** : Stockage clé-valeur dans une colonne

Ces extensions ne sont possibles que grâce à l'architecture objet-relationnelle de PostgreSQL.

---

## Pourquoi Cette Philosophie Est Importante ?

### 1. **Adaptabilité**

L'approche objet-relationnelle permet à PostgreSQL de s'adapter à une grande variété de cas d'usage :

- Applications web traditionnelles (e-commerce, CRM)
- Systèmes d'information géographique (SIG)
- Analyse de données et data warehousing
- Applications IoT et séries temporelles
- Intelligence artificielle et machine learning
- Recherche scientifique

**Un seul SGBD, de multiples usages.**

### 2. **Pérennité**

En restant fidèle aux standards SQL tout en innovant, PostgreSQL garantit :

- **Compatibilité** : Votre code SQL reste valide d'une version à l'autre  
- **Portabilité** : Facilité de migration depuis/vers d'autres SGBD relationnels  
- **Investissement sûr** : Vos compétences SQL restent pertinentes

### 3. **Innovation Sans Rupture**

PostgreSQL innove constamment (I/O asynchrone, colonnes virtuelles, optimisations…) sans casser la compatibilité avec le code existant.

### 4. **Écosystème Riche**

La philosophie d'extensibilité a créé un écosystème d'extensions, d'outils et de solutions qui enrichissent PostgreSQL sans alourdir le cœur du système.

### 5. **Une Philosophie « Unix » Appliquée aux Bases de Données**

PostgreSQL applique au monde des SGBD ce que Unix a apporté au monde des systèmes d'exploitation :

| Principe Unix | Application dans PostgreSQL |
|---------------|------------------------------|
| *« Do one thing and do it well »* | Le cœur fait l'essentiel : transactions ACID, planification, stockage. Les fonctionnalités spécialisées (géospatial, IA, séries temporelles) sont déléguées aux extensions. |
| Composition d'outils simples | `CREATE EXTENSION postgis;` ajoute un monde entier de capacités, sans toucher au cœur. |
| Configuration explicite plutôt que magique | `postgresql.conf` est lisible et auditable, chaque paramètre a une signification documentée. |
| Stabilité avant la mode | Les fonctionnalités sont mûrement réfléchies. Pas de course aux annonces marketing. |

Cette philosophie explique pourquoi PostgreSQL **vieillit bien** : du code écrit pour PostgreSQL 9 continue de fonctionner sur PostgreSQL 18, sans réécriture massive.

---

## PostgreSQL Aujourd'hui

### Position dans l'Industrie

Aujourd'hui, PostgreSQL est reconnu comme l'un des SGBD les plus avancés et les plus fiables au monde.

**Quelques faits marquants :**

- **DB-Engines** : régulièrement classé parmi les 4 premiers SGBD mondiaux
- **Stack Overflow Developer Survey** : le SGBD le plus utilisé par les développeurs depuis 2023 (≈ 49 % en 2024)
- **Adoption massive** : utilisé par Apple, Instagram, Spotify, Reddit, Twitch, et des milliers d'autres entreprises
- **Cloud providers** : support natif par AWS (RDS, Aurora), Azure (Azure Database for PostgreSQL), Google Cloud (Cloud SQL, AlloyDB), et tous les grands fournisseurs

### Une Communauté Mondiale

Le succès de PostgreSQL repose sur une communauté active et diversifiée :

- **Développeurs** : Des centaines de contributeurs actifs dans le monde entier  
- **Entreprises** : Des sociétés comme Crunchy Data, EDB, Percona, Timescale qui financent le développement  
- **Utilisateurs** : Des millions d'installations à travers le monde  
- **Événements** : Conférences PGConf sur tous les continents

---

## Comparaison Philosophique avec D'autres SGBD

Pour mieux comprendre la particularité de PostgreSQL, comparons sa philosophie avec celle d'autres systèmes :

### PostgreSQL vs MySQL

**MySQL** :
- Historiquement axé sur la rapidité et la simplicité
- Compromis parfois sur la conformité SQL et l'intégrité
- Propriété d'Oracle (bien qu'open source)
- Plusieurs moteurs de stockage (InnoDB, MyISAM…)

**PostgreSQL** :
- Priorité à la conformité, l'intégrité et la fiabilité
- Performances excellentes sans compromis sur la correction
- Communautaire, indépendant
- Architecture unifiée et cohérente

### PostgreSQL vs Oracle

**Oracle** :
- SGBD commercial propriétaire très coûteux
- Riche en fonctionnalités d'entreprise
- Écosystème verrouillé
- Support commercial garanti

**PostgreSQL** :
- Open source et gratuit
- Fonctionnalités comparables (parfois supérieures)
- Liberté et indépendance
- Support communautaire + options commerciales

### PostgreSQL vs MongoDB (NoSQL)

**MongoDB** :
- Base de données orientée documents (NoSQL)
- Schéma flexible
- Simplicité pour certains cas d'usage

**PostgreSQL** :
- Relationnel + support JSON/JSONB natif
- Structure et flexibilité en même temps
- Requêtes SQL puissantes sur JSON
- Intégrité transactionnelle forte

**Note** : PostgreSQL avec JSONB offre souvent le meilleur des deux mondes !

---

## Conclusion

L'histoire et la philosophie de PostgreSQL expliquent sa réussite actuelle :

**Les Clés du Succès :**

1. **Héritage académique** : Des fondations solides posées par des chercheurs de Berkeley  
2. **Open source** : Une licence permissive qui encourage l'adoption et la contribution  
3. **Communauté** : Une gouvernance communautaire qui garantit indépendance et qualité  
4. **Innovation** : Une approche objet-relationnelle qui permet l'extensibilité  
5. **Rigueur** : Un respect strict des standards et de l'intégrité des données  
6. **Pragmatisme** : Des fonctionnalités qui répondent aux besoins réels

PostgreSQL n'est pas simplement un SGBD parmi d'autres. C'est un projet avec une vision, une histoire et des valeurs qui se reflètent dans chaque aspect de sa conception et de son évolution.

En tant que développeur ou DevOps, comprendre cette philosophie vous aidera à mieux appréhender les choix de conception de PostgreSQL et à utiliser le système de manière optimale.

---

## Points Clés à Retenir

- ✅ PostgreSQL est issu du projet POSTGRES de l'Université de Berkeley (1986)  
- ✅ Il est open source depuis 1996 avec une licence très permissive  
- ✅ C'est un projet communautaire indépendant de toute entreprise  
- ✅ Il combine le modèle relationnel avec des capacités orientées objet  
- ✅ Sa philosophie privilégie : conformité, extensibilité, fiabilité, intégrité  
- ✅ PostgreSQL est aujourd'hui l'un des SGBD les plus avancés et respectés

---

**Prochaine section : 2.2. Gestion des versions et cycle de vie (PostgreSQL 18 : septembre 2025)**

⏭️ [Gestion des versions et cycle de vie (PostgreSQL 18 : septembre 2025)](/02-presentation-de-postgresql/02-gestion-des-versions-et-cycle-de-vie.md)
