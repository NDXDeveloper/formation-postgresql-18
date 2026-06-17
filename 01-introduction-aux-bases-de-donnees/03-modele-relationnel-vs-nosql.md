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

Le modèle relationnel a été inventé en **1970 par Edgar F. Codd**, un informaticien chez IBM, dans son article fondateur *« A Relational Model of Data for Large Shared Data Banks »* publié dans *Communications of the ACM*. Son idée révolutionnaire : organiser les données sous forme de **tables** (formellement appelées *relations* en mathématiques, d'où le nom), où chaque ligne représente un enregistrement et chaque colonne un attribut. Codd reçut le **Prix Turing en 1981** pour cette contribution majeure.

Le modèle s'appuie sur :
- **L'algèbre relationnelle** (sélection, projection, jointure, union, etc.) — bases mathématiques du langage SQL
- **Les formes normales** (1NF, 2NF, 3NF, BCNF) qui guident la conception d'un schéma sans redondance — détaillées au chapitre 11
- **L'indépendance physique/logique** : l'utilisateur travaille sur un modèle logique, le SGBD gère la représentation physique

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
- Chaque **ligne** (ou « tuple ») = un enregistrement unique (un client)
- Chaque **colonne** = un attribut spécifique (nom, âge, email…)
- Chaque **cellule** = une valeur unique d'un type précis
- Chaque table a une **clé primaire** (id) qui identifie de manière unique chaque ligne

### Le concept de relation entre tables

L'aspect **« relationnel »** vient du fait que les tables peuvent être **reliées entre elles** via des **clés étrangères**.

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

Le terme **NoSQL** est apparu en 1998 (Carlo Strozzi), puis a été réintroduit en 2009 par Johan Oskarsson pour désigner les bases de données distribuées non-relationnelles. À l'origine compris comme **« No SQL »** (sans SQL), il a été progressivement réinterprété comme **« Not Only SQL »** (pas seulement SQL) — beaucoup de bases NoSQL proposent aujourd'hui des langages d'interrogation inspirés de SQL.

Ce mouvement est né dans les années 2000 pour répondre aux besoins du web moderne, sous l'impulsion notamment des articles fondateurs de Google sur **Bigtable** (2006) et d'Amazon sur **Dynamo** (2007) :

- **Volume massif** de données (Big Data)
- **Flexibilité** du schéma (données changeantes)
- **Performance** en lecture/écriture à très grande échelle
- **Distribution** sur de nombreux serveurs (sharding, réplication)

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

#### 3. **Bases de données à familles de colonnes** (*Wide-Column Store*)

**Concept** : Les données sont organisées en **familles de colonnes**, où chaque ligne peut avoir un ensemble de colonnes différent. Contrairement au modèle relationnel où chaque ligne a les mêmes colonnes, ici la structure est flexible par ligne.

```
Famille de colonnes "profil" :
  Ligne "user:1" → { nom: "Jean", age: 35, ville: "Paris" }
  Ligne "user:2" → { nom: "Marie", ville: "Lyon", telephone: "06..." }
                    (pas de colonne "age" pour Marie)

Famille de colonnes "activite" :
  Ligne "user:1" → { dernier_achat: "2025-11-01", nb_visites: 42 }
```

> 💡 À ne pas confondre avec les bases **columnar** analytiques (ClickHouse, BigQuery, DuckDB) qui stockent physiquement les données colonne par colonne pour optimiser les agrégations. Ce sont deux concepts différents.

**Exemples** : Cassandra, HBase, Bigtable (Google), ScyllaDB

**Cas d'usage** :
- Données massives distribuées (Big Data)
- Logs et événements à très haut débit d'écriture
- Séries temporelles distribuées

**Avantages** :
- Excellente scalabilité horizontale
- Haut débit en écriture
- Tolérance aux pannes (distribution multi-datacenter)

**Limites** :
- Complexe à modéliser (penser les requêtes avant le schéma)
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
| **Structure** | Tables avec lignes et colonnes | Variable (documents, graphes…) |
| **Relations** | Jointures explicites | Imbrication ou références |
| **Transactions** | ACID garanti | Variable (souvent BASE\*) |
| **Scalabilité** | Verticale par défaut, horizontale via réplication/partitionnement/Citus | Horizontale native (sharding intégré) |
| **Requêtes** | SQL puissant et standardisé | API spécifiques par type |
| **Intégrité** | Forte (contraintes) | Plus faible (responsabilité app) |
| **Cas d'usage** | Transactionnel, cohérence critique | Big Data, flexibilité, performance |
| **Exemples** | PostgreSQL, MySQL, Oracle | MongoDB, Redis, Cassandra, Neo4j |

\*BASE = **B**asically **A**vailable, **S**oft state, **E**ventually consistent (cohérence « à terme »)

> 💡 **Une troisième voie : NewSQL.** Des SGBD comme **CockroachDB**, **YugabyteDB**, **TiDB** ou **Google Spanner** combinent SQL standard et propriétés ACID avec une scalabilité horizontale native (sharding automatique, consensus distribué). Plusieurs sont compatibles avec le protocole PostgreSQL (CockroachDB, YugabyteDB), ce qui permet de réutiliser les drivers et une partie de l'écosystème.

### Le théorème CAP : la clé pour comprendre les compromis

Pour bien choisir entre SQL et NoSQL en environnement distribué, il faut connaître le **théorème CAP**, formulé par Eric Brewer en 2000 et démontré formellement par Gilbert et Lynch en 2002.

Le théorème énonce qu'**un système distribué ne peut garantir simultanément que deux des trois propriétés suivantes** :

| Lettre | Propriété | Définition |
|:------:|-----------|------------|
| **C** | **Consistency** (cohérence) | Tous les nœuds voient les mêmes données au même moment |
| **A** | **Availability** (disponibilité) | Toute requête reçoit une réponse (sans garantie de fraîcheur) |
| **P** | **Partition tolerance** (tolérance au partitionnement) | Le système fonctionne même si le réseau coupe la communication entre nœuds |

En pratique, **P (les coupures réseau) est inévitable** dans un système distribué. Le vrai choix se fait donc entre **C** et **A** :

```
                    P (inévitable)
                       ▲
                      ╱ ╲
                     ╱   ╲
                    ╱     ╲
              CP   ╱       ╲   AP
        ╔════════ ╱─────────╲ ══════════╗
        ║ PostgreSQL,        Cassandra  ║
        ║ MongoDB primary,   DynamoDB   ║
        ║ HBase             CouchDB     ║
        ║ → Cohérence       → Dispo     ║
        ║   stricte           toujours  ║
        ╚═══════════════════════════════╝

(CA n'existe pas vraiment en distribué : il faut abandonner P)
```

- **Systèmes CP** (cohérence prioritaire) : PostgreSQL en mode réplication synchrone, MongoDB par défaut, HBase. En cas de coupure, certains nœuds refusent les requêtes pour éviter d'avoir des données obsolètes.
- **Systèmes AP** (disponibilité prioritaire) : Cassandra, DynamoDB, CouchDB. En cas de coupure, tous les nœuds répondent, quitte à fournir des données légèrement obsolètes (« eventually consistent »).

> 🔄 **Et PACELC ?** Le théorème **PACELC** (Abadi, 2010) complète CAP : *« en cas de Partition (P), il faut choisir entre Availability (A) et Consistency (C) ; sinon (E for Else), il faut choisir entre Latency (L) et Consistency (C). »* Plus pertinent que CAP pour les systèmes cloud modernes.

**Pour PostgreSQL en pratique** : sur un seul serveur (configuration la plus courante), le théorème CAP ne s'applique pas — vous bénéficiez de la cohérence sans compromis. C'est seulement avec la **réplication distribuée** (chapitre 17) que le compromis se pose.

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
   - SQL fonctionne sur PostgreSQL, MySQL, Oracle…

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

> ⚖️ **Réflexe à avoir** : avant de partir sur un NoSQL « parce que c'est tendance », demandez-vous si PostgreSQL ne pourrait pas répondre au besoin. Avec JSONB (documents), pgvector (vectoriel), PostGIS (géospatial), TimescaleDB (séries temporelles), `LISTEN/NOTIFY` (événements), `LATERAL` et CTE récursives (graphes simples), PostgreSQL couvre une très large gamme de cas d'usage tout en gardant les garanties ACID. Un seul SGBD à exploiter, à sauvegarder et à surveiller, c'est un gain opérationnel énorme.

---

## PostgreSQL : Le meilleur des deux mondes ?

### PostgreSQL comme SGBDR

PostgreSQL est avant tout un **SGBDR** puissant et conforme aux standards :
- Tables relationnelles
- SQL complet (jointures, sous-requêtes, CTE…)
- Contraintes d'intégrité (PK, FK, CHECK…)
- Transactions ACID strictes

### PostgreSQL avec des capacités NoSQL

Mais PostgreSQL offre **aussi** des fonctionnalités NoSQL !

#### 1. **Type JSONB : Base de données documentaire**

PostgreSQL propose **deux types JSON** :

| Type | Stockage | Performance lecture | Indexable | Préserve l'ordre des clés / espaces |
|------|----------|---------------------|-----------|--------------------------------------|
| `JSON` | Texte brut | Re-parsing à chaque accès | Non efficacement | Oui |
| `JSONB` | Binaire décomposé | Accès direct, très rapide | Oui (GIN, B-Tree sur expression) | Non |

👉 **Utilisez `JSONB` dans 99 % des cas.** Le type `JSON` (texte) ne se justifie que si vous devez conserver exactement le texte d'origine.

```sql
-- Stocker des documents JSON binaires
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

-- Index GIN pour accélérer les recherches dans le JSON
CREATE INDEX idx_produits_details ON produits USING gin (details);
```

**Avantages** :
- Flexibilité du schéma pour les champs semi-structurés
- Index GIN très performants pour les recherches `?`, `?&`, `@>`
- Combinaison naturelle de relationnel (colonnes fixes) et NoSQL (JSONB)
- Toutes les garanties ACID s'appliquent au JSONB

Le détail des opérateurs JSONB et de leur indexation est vu aux chapitres 4 et 13.

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

- **PostGIS** : Données géospatiales (géographie, géométrie)  
- **pgvector** : Recherche vectorielle (IA, embeddings)  
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

### ❌ Mythe 1 : « NoSQL est plus rapide que SQL »

**Réalité** : cela dépend du cas d'usage.
- Redis est ultra-rapide pour le cache (données en RAM)
- PostgreSQL avec index bien conçus est extrêmement performant
- Pour des requêtes complexes avec jointures, SQL est souvent plus rapide

### ❌ Mythe 2 : « NoSQL remplace SQL »

**Réalité** : Non, ils sont complémentaires.
- SQL reste dominant pour les applications transactionnelles
- Les bases relationnelles dominent toujours largement : dans le classement de popularité **DB-Engines** (2026), les quatre SGBD les plus populaires (Oracle, MySQL, SQL Server, PostgreSQL) sont tous relationnels — MongoDB, premier NoSQL, n'arrive qu'en cinquième position
- PostgreSQL est, selon le *Stack Overflow Developer Survey 2024*, le **SGBD le plus utilisé par les développeurs** (≈ 49 %, pour la deuxième année consécutive) — loin devant le NoSQL le plus populaire, MongoDB (≈ 25 %)
- NoSQL est un outil additionnel, pas un remplacement

### ❌ Mythe 3 : « NoSQL n'a pas besoin de schéma »

**Réalité** : le schéma existe, il est juste implicite.
- Votre application doit quand même connaître la structure des données
- Pas de contraintes = plus de validation côté application
- « Schemaless » ne signifie pas « sans structure »

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

✅ **Il n'y a pas de « meilleur » modèle** : tout dépend du cas d'usage

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
