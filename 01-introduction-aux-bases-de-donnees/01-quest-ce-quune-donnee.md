🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.1. Qu'est-ce qu'une donnée ? Qu'est-ce qu'une base de données ?

## Introduction

Bienvenue dans cette première section de notre formation PostgreSQL ! Avant de plonger dans les aspects techniques, il est essentiel de comprendre les concepts fondamentaux qui constituent la base de tout système de gestion de bases de données. Dans ce chapitre, nous allons explorer deux notions essentielles : **la donnée** et **la base de données**.

Ces concepts peuvent sembler abstraits au premier abord, mais vous allez découvrir que vous interagissez avec des données et des bases de données quotidiennement, souvent sans même vous en rendre compte.

---

## Qu'est-ce qu'une donnée ?

### Définition simple

Une **donnée** est un fait brut, observable ou mesurable, qui représente quelque chose du monde réel. C'est l'élément de base de toute information numérique.

Pour le dire autrement : une donnée, c'est simplement **un morceau d'information**.

### Exemples concrets du quotidien

Voici des exemples de données que vous manipulez tous les jours :

- **Votre nom** : « Marie Dupont »
- **Votre âge** : 28
- **Votre adresse e-mail** : marie.dupont@example.com
- **La température extérieure** : 15 °C
- **Le prix d'un produit** : 49,99 €
- **Une date** : 19 novembre 2025
- **Un numéro de téléphone** : 06 12 34 56 78
- **Une photo** : un fichier JPEG de votre dernier voyage

### Les caractéristiques d'une donnée

Une donnée possède plusieurs caractéristiques importantes :

1. **Elle est atomique** : c'est une unité d'information indivisible dans son contexte. Par exemple, « Jean » est une donnée (un prénom), « Dupont » en est une autre (un nom de famille). On évite de mélanger plusieurs informations dans une seule valeur (« Jean Dupont, Paris, 35 ans »).

   > 📐 Ce principe d'atomicité est à la base de la **première forme normale (1NF)**, fondement de la modélisation relationnelle. Il sera approfondi au chapitre 11.

2. **Elle a un type** : les données peuvent être de différentes natures :
   - **Numériques** : entiers (`42`, `-17`) ou décimaux (`3.14`, `49.99`)
   - **Textuelles** : « Bonjour », « PostgreSQL »
   - **Temporelles** : `2025-11-19`, `14:30:00`, intervalles (`1 day 2 hours`)
   - **Booléennes** : Vrai / Faux
   - **Binaires** : images, vidéos, fichiers (type `BYTEA` en PostgreSQL)
   - **Structurées** : JSON, tableaux, géométries…
   - **`NULL`** : l'**absence** de valeur (différent de zéro ou de chaîne vide ; détaillé au chapitre 5)

3. **Elle a un sens dans un contexte** : le nombre « 42 » seul ne signifie pas grand-chose. Mais « 42 ans » (âge), « 42 € » (prix), ou « 42 km » (distance) ont tous un sens différent. C'est le **schéma** de la base (nom de la colonne, type, contrainte) qui donne ce contexte.

### Données vs Information

Il est important de distinguer **donnée** et **information** :

- **Donnée** : Élément brut, sans contexte. Exemple : « 25 », « Paris », « 2025-11-19 »
- **Information** : Donnée avec du contexte et du sens. Exemple : « Marie a 25 ans, elle habite à Paris, et son rendez-vous est le 19 novembre 2025 »

Les données deviennent de l'information lorsqu'elles sont organisées, structurées et mises en relation.

#### La pyramide DIKW (pour aller un peu plus loin)

Les sciences de l'information formalisent cette distinction avec la **pyramide DIKW**, popularisée par Russell Ackoff (1989) :

```
                  ▲
                 ╱ ╲              W — Wisdom (Sagesse)
                ╱ W ╲             « Faut-il agir ainsi ? »
               ╱─────╲
              ╱   K   ╲           K — Knowledge (Connaissance)
             ╱─────────╲          « Comment l'utiliser ? »
            ╱     I     ╲         I — Information
           ╱─────────────╲        « Que signifie cette donnée ? »
          ╱       D       ╲       D — Data (Donnée)
         ╱─────────────────╲      « Quel est ce fait brut ? »
        ─────────────────────
```

| Niveau | Définition | Exemple |
|--------|------------|---------|
| **D** — Donnée | Fait brut, sans contexte | `38.5` |
| **I** — Information | Donnée + contexte | « Le patient a 38,5 °C de fièvre » |
| **K** — Connaissance | Information appliquée | « Une fièvre > 38 °C nécessite surveillance » |
| **W** — Sagesse | Connaissance évaluée | « Avec cet historique, je consulte un médecin » |

👉 Une base de données stocke des **données**. C'est l'application (et son utilisateur) qui les transforme progressivement en information, connaissance, puis sagesse.

### Données structurées, semi-structurées et non structurées

Au-delà de leur type, les données se distinguent aussi par leur **degré d'organisation**. Cette classification est essentielle : elle détermine en grande partie le type de base de données le plus adapté pour les stocker.

| Catégorie | Description | Exemples | Stockage typique |
|-----------|-------------|----------|------------------|
| **Structurées** | Organisées selon un schéma strict et prédéfini (lignes et colonnes typées) | Table de clients, transactions bancaires, catalogue produits | SGBDR (PostgreSQL, MySQL…) |
| **Semi-structurées** | Dotées d'une structure souple et auto-descriptive (la structure est portée par la donnée elle-même) | JSON, XML, YAML | JSONB (PostgreSQL), MongoDB |
| **Non structurées** | Sans modèle prédéfini ; contenu libre | Texte libre, images, vidéos, audio, PDF | Stockage de fichiers ou objet (S3), `BYTEA` |

La majorité des données produites aujourd'hui (une proportion souvent estimée autour de **80 %**) sont non structurées — d'où l'importance des outils capables de les exploiter.

> 🐘 **La polyvalence de PostgreSQL** : il couvre les trois catégories. Le modèle **relationnel** pour les données structurées, le type **JSONB** pour le semi-structuré, et la **recherche plein texte** (*Full-Text Search*) ou le type **BYTEA** pour exploiter et stocker des contenus non structurés.

### Pourquoi les données sont-elles importantes ?

Dans notre monde numérique, les données sont devenues le **pétrole du 21e siècle** (expression popularisée par Clive Humby en 2006) :

- Les entreprises utilisent les données pour **prendre des décisions**
- Les applications web et mobiles **stockent vos préférences** sous forme de données
- Les systèmes de santé **enregistrent votre dossier médical**
- Les réseaux sociaux **conservent vos publications et interactions**
- Les banques **gèrent vos transactions financières**

Sans un moyen efficace de stocker, organiser et récupérer ces données, nos applications modernes ne pourraient tout simplement pas fonctionner.

---

## Qu'est-ce qu'une base de données ?

### Définition simple

Une **base de données** (en anglais : *database*) est un **ensemble organisé de données** structurées de manière à faciliter leur stockage, leur gestion, leur recherche et leur utilisation.

Pensez à une base de données comme à une **bibliothèque numérique** où chaque livre est classé de manière logique, avec un système de catalogage qui permet de retrouver rapidement l'information recherchée.

### Pourquoi avons-nous besoin de bases de données ?

Imaginez que vous gérez une petite entreprise avec 10 clients. Vous pourriez noter leurs informations dans un cahier ou un fichier Excel :

```
Client 1 : Jean Dupont, 35 ans, jean.dupont@email.com, Paris  
Client 2 : Marie Martin, 28 ans, marie.martin@email.com, Lyon  
…
```

C'est gérable ! Mais maintenant, imaginez avec **10 000 clients**, ou **1 million de clients**, et que vous devez :

- Retrouver rapidement les informations d'un client spécifique
- Modifier l'adresse d'un client
- Lister tous les clients de Paris
- Calculer l'âge moyen de vos clients
- Gérer plusieurs utilisateurs qui accèdent aux données en même temps
- Assurer que les données ne soient pas perdues en cas de panne

Avec un simple fichier texte ou tableur, cela devient rapidement **impossible** ou **très lent**. C'est là qu'intervient la base de données !

### Les avantages d'une base de données

Une base de données offre de nombreux avantages par rapport à un simple fichier :

#### 1. **Organisation structurée**

Les données sont organisées de manière logique, généralement sous forme de **tables** (comme des tableaux Excel améliorés) :

**Table : Clients**
```
┌────┬─────────────┬──────┬───────────────────────────┬────────┐
│ ID │ Nom         │ Âge  │ Email                     │ Ville  │
├────┼─────────────┼──────┼───────────────────────────┼────────┤
│ 1  │ Jean Dupont │ 35   │ jean.dupont@email.com     │ Paris  │
│ 2  │ Marie Martin│ 28   │ marie.martin@email.com    │ Lyon   │
│ 3  │ Luc Bernard │ 42   │ luc.bernard@email.com     │ Paris  │
└────┴─────────────┴──────┴───────────────────────────┴────────┘
```

#### 2. **Rapidité de recherche**

Grâce aux **index** (comme l'index d'un livre), une base de données peut retrouver une information parmi des millions d'enregistrements en quelques millisecondes.

> ⚡ **Ordre de grandeur** : sans index, retrouver une ligne dans 1 million d'enregistrements nécessite jusqu'à **1 000 000 comparaisons** (lecture séquentielle, complexité *O(N)*). Avec un index B-Tree, il suffit d'environ **20 comparaisons** (complexité *O(log N)*). Sur 1 milliard de lignes, on passe d'un milliard d'opérations à seulement ~30. C'est ce gain logarithmique qui rend les bases de données aussi performantes — l'indexation est explorée en détail au chapitre 13.

#### 3. **Intégrité des données**

La base de données garantit que les données restent **cohérentes** et **valides**. Par exemple :
- Un âge ne peut pas être négatif
- Une adresse e-mail doit avoir un format valide
- On ne peut pas supprimer un client s'il a encore des commandes en cours

#### 4. **Accès concurrent**

Plusieurs utilisateurs peuvent lire et modifier les données **en même temps** sans conflit, grâce à des mécanismes de gestion de la concurrence.

#### 5. **Sécurité**

La base de données contrôle **qui peut accéder à quelles données** et **quelles opérations sont autorisées** (lecture, écriture, suppression).

#### 6. **Sauvegarde et récupération**

En cas de panne, les bases de données offrent des mécanismes pour **restaurer les données** et éviter toute perte.

### Exemples de bases de données dans la vie réelle

Vous utilisez des bases de données tous les jours, souvent sans le savoir :

| Application | Type de données stockées |
|------------|--------------------------|
| **Facebook** | Profils utilisateurs, publications, photos, commentaires, likes |
| **Netflix** | Films, séries, historique de visionnage, recommandations |
| **Banque en ligne** | Comptes, transactions, virements, cartes bancaires |
| **Amazon** | Produits, commandes, avis clients, historique d'achats |
| **Gmail** | E-mails, contacts, pièces jointes, dossiers |
| **Spotify** | Chansons, artistes, playlists, historique d'écoute |
| **Uber** | Chauffeurs, passagers, trajets, tarifs, positions GPS |

Chacune de ces applications repose sur une ou plusieurs bases de données pour fonctionner.

### Types de bases de données

Il existe différents types de bases de données, adaptés à différents besoins :

#### 1. **Bases de données relationnelles (SGBDR)**

C'est le type le plus courant et celui sur lequel nous allons nous concentrer dans cette formation. Les données sont organisées en **tables** reliées entre elles par des **relations**.

**Exemples** : PostgreSQL, MySQL, Oracle, SQL Server

**Cas d'usage** : Applications de gestion (e-commerce, CRM, ERP), systèmes bancaires, sites web traditionnels

#### 2. **Bases de données NoSQL**

Conçues pour des données non structurées ou semi-structurées, avec une flexibilité accrue. On distingue quatre grandes familles :

- **Documents** : MongoDB, CouchDB (documents JSON/BSON)
- **Clé-valeur** : Redis, DynamoDB, etcd (paires clé/valeur simples)
- **Wide-column (familles de colonnes)** : Cassandra, HBase, Bigtable (lignes à colonnes variables, optimisées écritures massives)
- **Graphes** : Neo4j, ArangoDB (nœuds et relations interconnectés)

**Cas d'usage** : Big data, réseaux sociaux, systèmes temps réel, IoT

> 💡 **Attention à la confusion** : les bases « wide-column » (comme Cassandra) sont des bases NoSQL orientées **lignes** avec colonnes flexibles. Elles ne sont **pas** la même chose que les bases « columnar » (présentées juste après), qui partitionnent réellement chaque colonne pour l'analytique.

#### 3. **Bases de données columnar (orientées colonnes)**

Optimisées pour l'analyse de grandes quantités de données (OLAP, data warehousing). Contrairement aux bases NoSQL wide-column, ces bases partitionnent verticalement chaque colonne, ce qui permet une compression et des agrégations bien plus efficaces sur de gros volumes.

**Exemples** : ClickHouse, Amazon Redshift, Google BigQuery, DuckDB, Snowflake

**Cas d'usage** : Analytique, business intelligence, reporting, data warehousing

#### 4. **Bases de données en mémoire**

Stockent les données en RAM pour des performances ultra-rapides.

**Exemples** : Redis, Memcached

**Cas d'usage** : Cache, sessions utilisateur, compteurs temps réel

#### 5. **Bases de données vectorielles**

Spécialisées dans le stockage et la recherche de **vecteurs** (embeddings), ces données numériques de haute dimension produites par les modèles d'IA. Indispensables pour les applications modernes d'intelligence artificielle (RAG, recherche sémantique).

**Exemples** : Pinecone, Weaviate, Qdrant, Milvus, Chroma — et **PostgreSQL avec l'extension pgvector**

**Cas d'usage** : Recherche sémantique, systèmes de recommandation, RAG (Retrieval-Augmented Generation), détection de similarité

> 🐘 PostgreSQL est l'un des rares SGBDR à couvrir nativement **plusieurs** de ces familles : relationnel (par défaut), documents (JSONB), géospatial (PostGIS), vectoriel (pgvector), séries temporelles (TimescaleDB). C'est l'une des raisons de sa popularité.

---

## Analogie pour bien comprendre

Imaginons une **bibliothèque municipale** :

| Concept | Bibliothèque | Base de données |
|---------|--------------|-----------------|
| **Base de données** | Toute la bibliothèque | L'ensemble de la base |
| **Table** | Une étagère thématique (Romans, Sciences, etc.) | Une table (Clients, Produits, etc.) |
| **Enregistrement (ligne)** | Un livre individuel | Une ligne de table |
| **Champ / Colonne** | Catégorie d'information (Titre, Auteur, ISBN) | Une colonne d'une table (Nom, Email, Âge) |
| **Donnée (valeur)** | Une information précise du livre (« Les Misérables ») | Une cellule (valeur unique dans une ligne) |
| **Index** | Le catalogue de recherche | Index de base de données |
| **Bibliothécaire** | Personne qui gère les livres | SGBD (PostgreSQL) |
| **Lecteur** | Personne qui consulte les livres | Application / Utilisateur |

Tout comme vous ne stockeriez pas 10 000 livres en vrac dans une cave (impossible de retrouver quoi que ce soit !), vous ne stockez pas des millions de données dans un simple fichier texte. Vous utilisez une base de données, qui est comme une bibliothèque ultra-moderne et automatisée.

---

## Récapitulatif

### Points clés à retenir

✅ **Une donnée** est un fait brut mesurable (nom, âge, prix, date, etc.) ; mise en contexte, elle devient de l'**information**

✅ **Une base de données** est un ensemble organisé de données, structuré pour faciliter le stockage, la recherche et la gestion

✅ **Les bases de données résolvent des problèmes** : rapidité, organisation, intégrité, sécurité, concurrence

✅ **Les bases de données relationnelles** (comme PostgreSQL) organisent les données en tables reliées entre elles

✅ **Nous utilisons des bases de données** quotidiennement via les applications web, mobiles, bancaires, etc.

### Ce que vous avez appris

Dans cette section, vous avez découvert :

- La définition d'une donnée et ses caractéristiques
- La différence entre donnée et information
- Ce qu'est une base de données et pourquoi elle est nécessaire
- Les avantages d'une base de données par rapport à un simple fichier
- Les différents types de bases de données existants
- Des exemples concrets d'utilisation dans le monde réel

### Et maintenant ?

Maintenant que vous comprenez ce qu'est une donnée et une base de données, nous allons découvrir dans la section suivante le **SGBD (Système de Gestion de Bases de Données)**, qui est l'outil logiciel qui permet de créer, gérer et interroger ces bases de données.

PostgreSQL est précisément un SGBD, et l'un des plus puissants et populaires au monde !

---

**Prochaine section** : 1.2. Le concept de SGBD (Système de Gestion de Bases de Données)

⏭️ [Le concept de SGBD (Système de Gestion de Bases de Données)](/01-introduction-aux-bases-de-donnees/02-concept-de-sgbd.md)
