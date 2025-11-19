🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.3. Le modèle relationnel (SGBDR) vs NoSQL : Comparaison conceptuelle

## Introduction

Nous avons vu qu'un SGBD permet de gérer des bases de données de manière efficace. Mais saviez-vous qu'il existe **différentes façons d'organiser** les données ?

Les deux grandes familles sont :
1. **Le modèle relationnel** (SGBDR) : données organisées en tables reliées entre elles
2. **Le modèle NoSQL** : données organisées de manière plus flexible

PostgreSQL appartient à la famille des SGBDR (Systèmes de Gestion de Bases de Données Relationnelles), mais possède aussi des capacités NoSQL. Dans cette section, nous allons comprendre les différences fondamentales entre ces deux approches, leurs forces et leurs cas d'usage.

---

## Le Modèle Relationnel (SGBDR)

### Origine et philosophie

Le modèle relationnel a été inventé en **1970 par Edgar F. Codd**, un informaticien chez IBM. Son idée révolutionnaire : organiser les données sous forme de **tables** (aussi appelées "relations" en mathématiques), où chaque ligne représente un enregistrement et chaque colonne un attribut.

### Le concept de table

Une **table** est comme un tableau Excel structuré :

**Table : Clients**
```
┌────┬─────────────┬──────┬───────────────────────────┬────────┐
│ id │ nom         │ age  │ email                     │ ville  │
├────┼─────────────┼──────┼───────────────────────────┼────────┤
│ 1  │ Jean Dupont │ 35   │ jean.dupont@email.com     │ Paris  │
│ 2  │ Marie Martin│ 28   │ marie.martin@email.com    │ Lyon   │
│ 3  │ Luc Bernard │ 42   │ luc.bernard@email.com     │ Paris  │
└────┴─────────────┴──────┴───────────────────────────┴────────┘
```

**Caractéristiques** :
- Chaque **ligne** (ou "tuple") = un enregistrement unique (un client)
- Chaque **colonne** = un attribut spécifique (nom, âge, email...)
- Chaque **cellule** = une valeur unique d'un type précis
- Chaque table a une **clé primaire** (id) qui identifie de manière unique chaque ligne

### Le concept de relation entre tables

L'aspect **"relationnel"** vient du fait que les tables peuvent être **reliées entre elles** via des **clés étrangères**.

**Exemple** : Un système de e-commerce

**Table : Clients**
```
┌────┬─────────────┬───────────────────────────┐
│ id │ nom         │ email                     │
├────┼─────────────┼───────────────────────────┤
│ 1  │ Jean Dupont │ jean.dupont@email.com     │
│ 2  │ Marie Martin│ marie.martin@email.com    │
└────┴─────────────┴───────────────────────────┘
```

**Table : Commandes**
```
┌────┬────────────┬─────────────┬────────────┐
│ id │ date       │ montant     │ client_id  │ ← Clé étrangère
├────┼────────────┼─────────────┼────────────┤
│ 1  │ 2025-11-01 │ 150.00 €    │ 1          │ (réf. Jean)
│ 2  │ 2025-11-05 │ 89.99 €     │ 2          │ (réf. Marie)
│ 3  │ 2025-11-10 │ 250.00 €    │ 1          │ (réf. Jean)
└────┴────────────┴─────────────┴────────────┘
```

**Table : Produits**
```
┌────┬──────────────────┬───────────┐
│ id │ nom              │ prix      │
├────┼──────────────────┼───────────┤
│ 1  │ Ordinateur       │ 800.00 €  │
│ 2  │ Souris           │ 25.00 €   │
│ 3  │ Clavier          │ 65.00 €   │
└────┴──────────────────┴───────────┘
```

**Table : Lignes_Commandes** (table de jonction)
```
┌────┬──────────────┬────────────┬──────────┐
│ id │ commande_id  │ produit_id │ quantite │
├────┼──────────────┼────────────┼──────────┤
│ 1  │ 1            │ 2          │ 2        │
│ 2  │ 1            │ 3          │ 1        │
│ 3  │ 2            │ 1          │ 1        │
└────┴──────────────┴────────────┴──────────┘
```

**Visualisation des relations** :
```
Clients ──< Commandes ──< Lignes_Commandes >── Produits
   1              N              N                   1

Lecture :
- 1 client peut avoir N commandes
- 1 commande contient N lignes de commande
- 1 produit peut apparaître dans N lignes de commande
```

### Les avantages du modèle relationnel

#### 1. **Structure claire et prévisible**

Les données ont une structure fixe et connue à l'avance. Chaque table a des colonnes définies avec des types précis.

```
Vous savez toujours que :
- Un client a : id, nom, email, ville
- Une commande a : id, date, montant, client_id

Pas de surprise, pas d'ambiguïté !
```

#### 2. **Éviter la redondance (normalisation)**

Les données ne sont stockées qu'une seule fois, ce qui évite les incohérences.

**Mauvaise approche (redondante)** :
```
Commandes (non normalisées) :
┌────┬────────────┬─────────────┬───────────────┬───────────────────────┐
│ id │ date       │ montant     │ client_nom    │ client_email          │
├────┼────────────┼─────────────┼───────────────┼───────────────────────┤
│ 1  │ 2025-11-01 │ 150.00 €    │ Jean Dupont   │ jean.dupont@email.com │
│ 2  │ 2025-11-05 │ 89.99 €     │ Marie Martin  │ marie@email.com       │
│ 3  │ 2025-11-10 │ 250.00 €    │ Jean Dupont   │ jean.dupont@email.com │
└────┴────────────┴─────────────┴───────────────┴───────────────────────┘

Problème : Si Jean change son email, il faut modifier TOUTES ses commandes !
```

**Bonne approche (normalisée)** :
```
L'email de Jean est stocké UNE SEULE FOIS dans la table Clients.
Les commandes référencent juste son id (client_id = 1).
→ Pour changer son email, on modifie une seule ligne !
```

#### 3. **Intégrité des données garantie**

Les **contraintes** garantissent que les données restent cohérentes :

- **PRIMARY KEY** : Chaque ligne est unique
- **FOREIGN KEY** : Une commande ne peut pas référencer un client inexistant
- **UNIQUE** : Deux clients ne peuvent pas avoir le même email
- **CHECK** : L'âge ne peut pas être négatif
- **NOT NULL** : Un client doit obligatoirement avoir un nom

```sql
-- Si vous essayez de créer une commande pour un client inexistant :
INSERT INTO commandes (client_id, montant) VALUES (999, 100.00);

→ ERREUR : Le client 999 n'existe pas !
→ Le SGBD refuse l'opération
```

#### 4. **Langage SQL puissant et standardisé**

SQL permet des requêtes très expressives :

```sql
-- Trouver le montant total des commandes par client
SELECT
    c.nom,
    SUM(co.montant) as total_depense
FROM clients c
JOIN commandes co ON c.id = co.client_id
GROUP BY c.nom
ORDER BY total_depense DESC;

Résultat :
┌─────────────┬───────────────┐
│ nom         │ total_depense │
├─────────────┼───────────────┤
│ Jean Dupont │ 400.00 €      │
│ Marie Martin│ 89.99 €       │
└─────────────┴───────────────┘
```

#### 5. **Transactions ACID**

Le modèle relationnel garantit les propriétés ACID (vu dans la section précédente), essentielles pour les applications critiques (banques, e-commerce).

### Les limites du modèle relationnel

Malgré ses nombreux avantages, le modèle relationnel a certaines limites :

1. **Rigidité du schéma** : Ajouter une colonne à une grande table peut être complexe
2. **Performance pour certains cas** : Les jointures multiples peuvent être coûteuses
3. **Scalabilité horizontale** : Plus difficile à distribuer sur plusieurs serveurs
4. **Données hiérarchiques** : Les structures en arbre/graphe sont moins naturelles

---

## Le Modèle NoSQL

### Origine et philosophie

**NoSQL** signifie **"Not Only SQL"** (pas seulement SQL). Ce mouvement est né dans les années 2000 pour répondre aux besoins du web moderne :

- **Volume massif** de données (Big Data)
- **Flexibilité** du schéma (données changeantes)
- **Performance** en lecture/écriture à très grande échelle
- **Distribution** sur de nombreux serveurs

NoSQL ne remplace pas le relationnel, il offre des **alternatives** pour des cas d'usage spécifiques.

### Les 4 grandes familles NoSQL

#### 1. **Bases de données Clé-Valeur** (*Key-Value Store*)

**Concept** : Comme un dictionnaire géant : une clé → une valeur

```
"user:1001" → { nom: "Jean", age: 35 }
"session:abc123" → { connecte: true, expire: 3600 }
"compteur:visites" → 42567
```

**Exemples** : Redis, Memcached, DynamoDB

**Cas d'usage** :
- Cache applicatif (sessions utilisateur)
- Compteurs temps réel
- Files d'attente (queues)

**Avantages** :
- Ultra-rapide (en mémoire)
- Simple à utiliser
- Excellente scalabilité

**Limites** :
- Pas de requêtes complexes
- Pas de relations entre données

#### 2. **Bases de données Documentaires** (*Document Store*)

**Concept** : Stockage de documents JSON/XML/BSON

```json
// Document : Client
{
  "id": "client_1",
  "nom": "Jean Dupont",
  "age": 35,
  "email": "jean@email.com",
  "adresses": [
    {
      "type": "domicile",
      "rue": "10 rue de la Paix",
      "ville": "Paris",
      "code_postal": "75001"
    },
    {
      "type": "travail",
      "rue": "50 avenue des Champs",
      "ville": "Paris",
      "code_postal": "75008"
    }
  ],
  "commandes": [
    {
      "id": "cmd_1",
      "date": "2025-11-01",
      "montant": 150.00,
      "produits": [
        { "nom": "Souris", "quantite": 2 },
        { "nom": "Clavier", "quantite": 1 }
      ]
    }
  ]
}
```

**Exemples** : MongoDB, CouchDB, Firestore

**Cas d'usage** :
- Applications web modernes (APIs REST)
- Catalogues de produits
- Systèmes de gestion de contenu (CMS)

**Avantages** :
- Très flexible (schéma variable)
- Données imbriquées naturelles
- Facile pour les développeurs (JSON = JavaScript)

**Limites** :
- Redondance des données
- Pas de jointures efficaces
- Requêtes complexes limitées

#### 3. **Bases de données Colonnes** (*Column-Family Store*)

**Concept** : Stockage par colonnes plutôt que par lignes

```
Modèle traditionnel (ligne) :
Ligne 1 : id=1, nom="Jean", age=35, ville="Paris"
Ligne 2 : id=2, nom="Marie", age=28, ville="Lyon"

Modèle colonnes :
Colonne "id"    : [1, 2]
Colonne "nom"   : ["Jean", "Marie"]
Colonne "age"   : [35, 28]
Colonne "ville" : ["Paris", "Lyon"]
```

**Exemples** : Cassandra, HBase, ScyllaDB

**Cas d'usage** :
- Data warehousing (analyse de données)
- Séries temporelles massives
- Logs et événements

**Avantages** :
- Très performant pour agrégations
- Compression efficace
- Excellente scalabilité horizontale

**Limites** :
- Complexe à modéliser
- Pas adapté aux transactions ACID
- Courbe d'apprentissage élevée

#### 4. **Bases de données Graphe** (*Graph Database*)

**Concept** : Modélisation des relations comme des graphes (nœuds et arêtes)

```
Réseau social :

    (Alice) ──ami_de──> (Bob)
       │                  │
  travaille_avec       habite_à
       │                  │
       ▼                  ▼
    (Carol)            (Paris)
       │
   aime
       │
       ▼
    (Pizza)
```

**Exemples** : Neo4j, ArangoDB, Amazon Neptune

**Cas d'usage** :
- Réseaux sociaux (relations entre personnes)
- Systèmes de recommandation
- Détection de fraude
- Gestion de connaissances

**Avantages** :
- Requêtes de relations très rapides
- Modélisation naturelle des graphes
- Traversée de relations efficace

**Limites** :
- Pas adapté aux données tabulaires simples
- Complexité de modélisation
- Moins mature que SQL

---

## Comparaison : Relationnel vs NoSQL

### Tableau comparatif

| Critère | Relationnel (SGBDR) | NoSQL |
|---------|---------------------|-------|
| **Schéma** | Fixe et défini à l'avance | Flexible et dynamique |
| **Structure** | Tables avec lignes et colonnes | Variable (documents, graphes...) |
| **Relations** | Jointures explicites | Imbrication ou références |
| **Transactions** | ACID garanti | Variable (souvent BASE*) |
| **Scalabilité** | Verticale (serveur plus puissant) | Horizontale (plus de serveurs) |
| **Requêtes** | SQL puissant et standardisé | API spécifiques par type |
| **Intégrité** | Forte (contraintes) | Plus faible (responsabilité app) |
| **Cas d'usage** | Transactionnel, cohérence critique | Big Data, flexibilité, performance |
| **Exemples** | PostgreSQL, MySQL, Oracle | MongoDB, Redis, Cassandra, Neo4j |

*BASE = **B**asically **A**vailable, **S**oft state, **E**ventually consistent (cohérence "à terme")

### Quand utiliser le modèle relationnel (SGBDR) ?

✅ **Utilisez un SGBDR comme PostgreSQL si** :

1. **Vos données ont une structure claire et stable**
   - Exemple : Système de facturation, gestion RH, inventaire

2. **L'intégrité des données est critique**
   - Exemple : Banque, santé, comptabilité

3. **Vous avez besoin de transactions ACID**
   - Exemple : Paiements, réservations, transferts

4. **Vous faites des requêtes complexes avec jointures**
   - Exemple : Rapports financiers, analyses croisées

5. **Vos données sont relationnelles par nature**
   - Exemple : E-commerce (clients ↔ commandes ↔ produits)

6. **Vous voulez un langage standardisé (SQL)**
   - SQL fonctionne sur PostgreSQL, MySQL, Oracle...

### Quand utiliser NoSQL ?

✅ **Utilisez NoSQL si** :

1. **Votre schéma change fréquemment**
   - Exemple : Startup en phase d'expérimentation

2. **Vous avez des données massives et distribuées**
   - Exemple : Logs applicatifs, IoT, analytics

3. **La performance en lecture/écriture est prioritaire**
   - Exemple : Cache, sessions, compteurs temps réel

4. **Vos données sont hiérarchiques ou en graphe**
   - Exemple : Réseau social, arbre de catégories

5. **Vous travaillez avec des documents JSON**
   - Exemple : API REST moderne, microservices

6. **Vous avez besoin de scalabilité horizontale massive**
   - Exemple : Application mondiale, millions d'utilisateurs

---

## PostgreSQL : Le meilleur des deux mondes ?

### PostgreSQL comme SGBDR

PostgreSQL est avant tout un **SGBDR** puissant et conforme aux standards :
- Tables relationnelles
- SQL complet (jointures, sous-requêtes, CTE...)
- Contraintes d'intégrité (PK, FK, CHECK...)
- Transactions ACID strictes

### PostgreSQL avec des capacités NoSQL

Mais PostgreSQL offre **aussi** des fonctionnalités NoSQL !

#### 1. **Type JSONB : Base de données documentaire**

```sql
-- Stocker des documents JSON
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    details JSONB  -- Type NoSQL !
);

INSERT INTO produits (nom, details) VALUES
(
    'Ordinateur portable',
    '{
        "marque": "Dell",
        "processeur": "Intel i7",
        "ram": 16,
        "stockage": {"type": "SSD", "capacite": 512},
        "prix": 899.99,
        "disponible": true
    }'::jsonb
);

-- Requêter le JSON avec des opérateurs spéciaux
SELECT nom, details->>'marque' as marque
FROM produits
WHERE (details->'ram')::int >= 16;
```

**Avantages** :
- Flexibilité du schéma pour certains champs
- Index GIN pour performance sur JSON
- Possibilité de combiner relationnel et NoSQL

#### 2. **Type ARRAY : Tableaux**

```sql
-- Stocker des tableaux directement
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(200),
    tags TEXT[]  -- Tableau de tags
);

INSERT INTO articles (titre, tags) VALUES
('PostgreSQL 18 est sorti', ARRAY['database', 'postgresql', 'release']);

-- Rechercher dans les tableaux
SELECT * FROM articles
WHERE 'postgresql' = ANY(tags);
```

#### 3. **Extensions pour d'autres modèles**

- **PostGIS** : Données géospatiales (graphe géométrique)
- **pg_vector** : Recherche vectorielle (IA, embeddings)
- **ltree** : Hiérarchies et arbres
- **hstore** : Paires clé-valeur

### Stratégie hybride recommandée

Pour la plupart des applications, une approche **hybride** est idéale :

```
┌─────────────────────────────────────────────────────┐
│              Votre Application                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────┐     ┌──────────────────┐   │
│  │  PostgreSQL         │     │  Redis (Cache)   │   │
│  │  (Relationnel)      │     │  (Clé-Valeur)    │   │
│  │                     │     │                  │   │
│  │  - Clients          │     │  - Sessions      │   │
│  │  - Commandes        │     │  - Compteurs     │   │
│  │  - Produits (JSONB) │     │  - Cache queries │   │
│  │  - Transactions     │     │                  │   │
│  └─────────────────────┘     └──────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Principe** :
- **PostgreSQL** pour les données structurées et transactionnelles
- **Redis** pour le cache et les données éphémères
- **JSONB dans PostgreSQL** pour la flexibilité ponctuelle

---

## Mythes et réalités

### ❌ Mythe 1 : "NoSQL est plus rapide que SQL"

**Réalité** : Cela dépend du cas d'usage.
- Redis est ultra-rapide pour le cache (données en RAM)
- PostgreSQL avec index bien conçus est extrêmement performant
- Pour des requêtes complexes avec jointures, SQL est souvent plus rapide

### ❌ Mythe 2 : "NoSQL remplace SQL"

**Réalité** : Non, ils sont complémentaires.
- SQL reste dominant pour applications transactionnelles
- 70%+ des entreprises utilisent des SGBDR
- NoSQL est un outil additionnel, pas un remplacement

### ❌ Mythe 3 : "NoSQL n'a pas besoin de schéma"

**Réalité** : Le schéma existe, il est juste implicite.
- Votre application doit quand même connaître la structure des données
- Pas de contraintes = plus de validation côté application
- "Schemaless" ne signifie pas "sans structure"

### ✅ Réalité : Choisissez l'outil adapté au besoin

- **E-commerce classique** → PostgreSQL (ou MySQL)
- **Cache applicatif** → Redis
- **Logs et métriques** → ClickHouse ou Elasticsearch
- **Réseau social** → Combinaison SQL + Graphe (Neo4j)
- **Application temps réel** → Firestore ou MongoDB + Redis

---

## Exemple concret : Système de blog

Voyons comment modéliser un blog dans les deux approches :

### Approche relationnelle (PostgreSQL)

```sql
-- Table des utilisateurs
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255) UNIQUE
);

-- Table des articles
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(200),
    contenu TEXT,
    auteur_id INT REFERENCES utilisateurs(id),
    date_publication TIMESTAMP
);

-- Table des commentaires
CREATE TABLE commentaires (
    id SERIAL PRIMARY KEY,
    article_id INT REFERENCES articles(id),
    auteur_id INT REFERENCES utilisateurs(id),
    texte TEXT,
    date_creation TIMESTAMP
);

-- Requête : Articles avec leurs commentaires
SELECT
    a.titre,
    u.nom as auteur,
    c.texte as commentaire,
    c.date_creation
FROM articles a
JOIN utilisateurs u ON a.auteur_id = u.id
JOIN commentaires c ON c.article_id = a.id
WHERE a.id = 1;
```

### Approche NoSQL documentaire (MongoDB)

```javascript
// Document imbriqué
{
  "_id": "article_1",
  "titre": "Introduction à PostgreSQL",
  "contenu": "PostgreSQL est un SGBDR...",
  "auteur": {
    "id": "user_1",
    "nom": "Jean Dupont",
    "email": "jean@email.com"
  },
  "date_publication": "2025-11-19T10:00:00Z",
  "commentaires": [
    {
      "id": "comment_1",
      "auteur": {
        "id": "user_2",
        "nom": "Marie Martin"
      },
      "texte": "Excellent article !",
      "date_creation": "2025-11-19T11:30:00Z"
    },
    {
      "id": "comment_2",
      "auteur": {
        "id": "user_3",
        "nom": "Luc Bernard"
      },
      "texte": "Très instructif.",
      "date_creation": "2025-11-19T12:00:00Z"
    }
  ],
  "tags": ["database", "postgresql", "tutorial"]
}

// Requête : Un seul document contient tout
db.articles.findOne({ _id: "article_1" })
```

**Comparaison** :

| Critère | Relationnel | NoSQL Document |
|---------|-------------|----------------|
| **Requête** | 3 jointures | 1 requête simple |
| **Redondance** | Minimale | Importante (nom auteur répété) |
| **Modification** | Si Jean change de nom, 1 seule ligne à modifier | Il faut modifier tous les articles et commentaires |
| **Cohérence** | Garantie par FK | Responsabilité de l'application |
| **Complexité app** | SQL gère les jointures | Application gère la dénormalisation |

---

## Récapitulatif

### Points clés à retenir

✅ **Le modèle relationnel (SGBDR)** organise les données en tables reliées avec des contraintes d'intégrité strictes

✅ **NoSQL** offre des alternatives flexibles (documents, clé-valeur, colonnes, graphes) pour des cas d'usage spécifiques

✅ **PostgreSQL est relationnel** mais possède aussi des capacités NoSQL (JSONB, ARRAYS, extensions)

✅ **Il n'y a pas de "meilleur" modèle** : tout dépend du cas d'usage

✅ **Une approche hybride** (PostgreSQL + Redis, par exemple) est souvent optimale

✅ **SQL reste dominant** pour les applications transactionnelles et les données structurées

### Critères de choix

**Choisissez Relationnel (PostgreSQL) si** :
- Structure claire et stable
- Intégrité critique
- Transactions ACID nécessaires
- Requêtes complexes fréquentes

**Choisissez NoSQL si** :
- Schéma très flexible requis
- Scalabilité horizontale massive
- Performance lecture/écriture extrême
- Données non relationnelles (documents, graphes)

**Utilisez les deux si** :
- PostgreSQL pour données transactionnelles
- Redis pour cache
- Le meilleur des deux mondes !

### Ce que vous avez appris

Dans cette section, vous avez découvert :

- Les principes du modèle relationnel (tables, relations, normalisation)
- Les 4 familles NoSQL et leurs cas d'usage
- Les avantages et limites de chaque approche
- Comment PostgreSQL combine relationnel et NoSQL
- Comment choisir le bon modèle pour votre projet
- Des exemples concrets de modélisation

### Et maintenant ?

Maintenant que vous comprenez les différents modèles de bases de données et que vous savez pourquoi PostgreSQL est un excellent choix, nous allons dans la section suivante découvrir le **concept de transaction** et les **propriétés ACID** qui font la force des SGBDR.

Ces concepts sont fondamentaux pour comprendre comment PostgreSQL garantit la cohérence et la fiabilité de vos données, même en cas de problème.

---

**Prochaine section** : 1.4. Le concept de transaction et les propriétés ACID

⏭️ [Le concept de transaction et les propriétés ACID](/01-introduction-aux-bases-de-donnees/04-transactions-et-proprietes-acid.md)
