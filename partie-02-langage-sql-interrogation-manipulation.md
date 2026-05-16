🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 2 : Le Langage SQL - Interrogation et Manipulation

**Niveau** : Intermédiaire  
**Durée estimée** : 12-16 heures  
**Prérequis** : Partie 1 complétée (concepts fondamentaux et objets de base de données)  

---

## Vue d'ensemble

Bienvenue dans la **Partie 2** de cette formation. Après avoir posé les fondations théoriques et architecturales dans la Partie 1, nous entrons maintenant dans le vif du sujet : **le langage SQL**.

SQL (Structured Query Language) est le langage universel des bases de données relationnelles. C'est votre interface principale avec PostgreSQL, que vous soyez développeur écrivant des requêtes applicatives ou administrateur gérant des données. Cette partie vous permettra de passer du statut de débutant à celui d'utilisateur intermédiaire maîtrisant les opérations courantes.

### Pourquoi cette partie est essentielle ?

SQL est bien plus qu'un simple langage de requêtes. C'est un outil de pensée qui vous permet de :

- **Extraire l'information** dont vous avez besoin, de la manière dont vous en avez besoin  
- **Transformer et manipuler** vos données directement dans la base  
- **Établir et maintenir** l'intégrité de vos données via les relations  
- **Agréger et synthétiser** des volumes importants d'informations

Dans cette partie, nous couvrons les fondamentaux du SQL que vous utiliserez quotidiennement :
- **DQL** (*Data Query Language*) : interrogation des données avec `SELECT`  
- **DML** (*Data Manipulation Language*) : écriture des données avec `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `COPY`  
- **Relations et jointures** : conception de schémas relationnels, contraintes d'intégrité et jointures  
- **Agrégation et groupement** : `GROUP BY`, `HAVING`, fonctions statistiques et extensions de groupement

> 📌 **Vocabulaire** : `SELECT` est parfois classé dans le **DML au sens large** (puisqu'il manipule des données en lecture). Le standard SQL et la plupart des SGBD préfèrent le ranger dans le **DQL**, c'est la convention que nous suivons.

---

## Objectifs d'apprentissage

À l'issue de cette deuxième partie, vous serez capable de :

### 📊 Requêtes de sélection (DQL)
- Construire des requêtes SELECT efficaces en comprenant l'ordre d'exécution logique
- Maîtriser le filtrage avancé avec WHERE et la logique booléenne
- Gérer correctement les valeurs NULL et la logique ternaire
- Trier, paginer et dédupliquer vos résultats
- Utiliser DISTINCT et DISTINCT ON de manière appropriée

### ✏️ Manipulation de données (DML)
- Insérer des données efficacement (`INSERT` simple, multiple, `COPY`, et nouveautés PG 18 sur COPY)
- Mettre à jour et supprimer des enregistrements de manière ciblée (`UPDATE`, `DELETE`)
- Utiliser la clause `RETURNING`, avec accès direct à `OLD` et `NEW` (PostgreSQL 18)
- Maîtriser les *upserts* avec `INSERT … ON CONFLICT`
- Exploiter la commande `MERGE` (disponible depuis PG 15, enrichie de `OLD`/`NEW` en PG 18)
- Comprendre les différences entre `TRUNCATE` et `DELETE` (verrous, WAL, transactions)

### 🔗 Relations et jointures
- Définir et maintenir l'intégrité référentielle (PK, FK, contraintes)
- Comprendre la théorie des jointures (produit cartésien, sélection)
- Maîtriser tous les types de jointures (INNER, OUTER, CROSS)
- Utiliser les jointures avancées (LATERAL, self-joins)
- Implémenter des anti-jointures et semi-jointures
- Exploiter les nouvelles contraintes temporelles de PostgreSQL 18

### 📈 Agrégation et groupement
- Utiliser les fonctions d'agrégation standards et statistiques
- Grouper et filtrer avec GROUP BY et HAVING
- Maîtriser les extensions de groupement (ROLLUP, CUBE, GROUPING SETS)
- Appliquer des filtres d'agrégation spécifiques
- Créer des agrégations ordonnées complexes

---

## Structure de cette partie

Cette deuxième partie est organisée en **4 chapitres thématiques** :

### 📖 **Chapitre 5 : Requêtes de Sélection (DQL)**
Le cœur de SQL : extraire des données avec SELECT. Vous apprendrez l'ordre d'exécution logique, le filtrage, la gestion des NULL, le tri, la pagination et la déduplication.

### ✍️ **Chapitre 6 : Manipulation des Données (DML)**
Modifier le contenu de vos tables : `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`. Découverte des nouveautés PostgreSQL 18 (`OLD`/`NEW` dans `RETURNING`, améliorations `COPY`) et des techniques avancées comme les *upserts*.

### 🔄 **Chapitre 7 : Relations et Jointures**
Le fondement du modèle relationnel : contraintes d'intégrité, théorie des jointures, et tous les types de JOIN. Introduction aux contraintes temporelles de PostgreSQL 18.

### 📊 **Chapitre 8 : Agrégation et Groupement**
Synthétiser l'information : fonctions d'agrégation standards (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`), fonctions statistiques (`STDDEV`, `VARIANCE`, `CORR`, `PERCENTILE_*`), `GROUP BY` / `HAVING`, extensions de groupement (`ROLLUP`, `CUBE`, `GROUPING SETS`), filtres d'agrégation avec `FILTER`, et agrégations ordonnées (`STRING_AGG`, `ARRAY_AGG`, `JSON_AGG`, `MODE() WITHIN GROUP`).

---

## Philosophie pédagogique

### 🎯 De la théorie à la pratique

Contrairement à la Partie 1 purement conceptuelle, cette partie **introduit des exemples SQL concrets** tout en maintenant une approche théorique. Chaque concept est expliqué avec sa syntaxe, son fonctionnement interne et ses cas d'usage.

### 🧠 Comprendre avant d'exécuter

Nous ne nous contentons pas de montrer la syntaxe SQL. Nous expliquons :
- **Comment** PostgreSQL exécute vos requêtes  
- **Pourquoi** certaines approches sont plus efficaces  
- **Quand** utiliser telle ou telle technique  
- **Quels** pièges éviter

### 🔄 Construction progressive

Chaque chapitre s'appuie sur les précédents :

```
Chapitre 5 (DQL)           Chapitre 6 (DML)          Chapitre 7 (Relations)     Chapitre 8 (Agrégation)
─────────────────          ─────────────────         ──────────────────────     ───────────────────────
SELECT simple              INSERT simple             Contraintes (PK/FK)         COUNT / SUM / AVG
   ↓                          ↓                          ↓                          ↓
Filtrage WHERE             INSERT multiple           Théorie des jointures      Fonctions statistiques
   ↓                          ↓                          ↓                          ↓
NULL et ternaire           COPY, UPDATE, DELETE      INNER, LEFT, RIGHT, FULL   GROUP BY / HAVING
   ↓                          ↓                          ↓                          ↓
Tri, pagination            RETURNING + OLD/NEW       Self-join, LATERAL         ROLLUP, CUBE, GROUPING SETS
   ↓                          ↓                          ↓                          ↓
DISTINCT, DISTINCT ON      UPSERT (ON CONFLICT)      Anti/Semi-jointures        FILTER, agrégations ordonnées
                              ↓
                           MERGE, TRUNCATE
```

Chaque flèche `↓` représente une montée progressive en sophistication. Une fois la base d'un chapitre solide, on bâtit les techniques avancées dessus.

### ⚠️ Pièges courants et bonnes pratiques

Tout au long de cette partie, nous mettons en évidence :
- Les erreurs fréquentes des débutants
- Les subtilités du SQL (logique ternaire, NULL, ordre d'exécution)
- Les bonnes pratiques PostgreSQL
- Les optimisations à connaître

---

## Nouveautés PostgreSQL 18 abordées

Cette deuxième partie présente plusieurs innovations majeures de PostgreSQL 18 :

### 🆕 DML amélioré
- **`OLD` et `NEW` dans `RETURNING`** : accès direct aux valeurs avant et après modification — disponible pour `UPDATE`, `DELETE`, `INSERT … ON CONFLICT` et `MERGE`. Renvoie `NULL` côté `OLD` pour un `INSERT` pur (et côté `NEW` pour un `DELETE` pur), ce qui permet le pattern `OLD IS NULL` pour distinguer les opérations dans un `MERGE`.
- **Améliorations `COPY`** : `COPY FROM` lisant un fichier ne reconnaît plus `\.` comme marqueur de fin (le piège silencieux du tronquage est éliminé) ; nouvelles options `REJECT_LIMIT` et `LOG_VERBOSITY = 'silent'` qui complètent `ON_ERROR = 'ignore'` (PG 17) ; `COPY TO` accepte enfin les vues matérialisées peuplées.
- **Contraintes `NOT NULL` repensées** : les contraintes `NOT NULL` deviennent des objets de première classe dans `pg_constraint` (avec un nom et un identifiant) et acceptent `NOT VALID`, ce qui permet de les ajouter sans bloquer la table sur une longue validation.

### 🔗 Contraintes évoluées
- **Contraintes temporelles** (*temporal constraints*) : nouvelles syntaxes `UNIQUE … WITHOUT OVERLAPS` et `PRIMARY KEY … WITHOUT OVERLAPS` sur des colonnes de type `range`, ainsi que `FOREIGN KEY … PERIOD` pour les clés étrangères temporelles. Le standard SQL:2011 enfin natif, sans `EXCLUDE USING gist` à écrire à la main.

> 📌 **À propos de `MERGE`** : la commande `MERGE` n'est pas nouvelle — elle existe depuis **PostgreSQL 15**, et `RETURNING` y a été ajouté en PG 17. Ce qui est nouveau en PG 18, c'est le support de `OLD`/`NEW` dans son `RETURNING`.

Ces nouveautés seront expliquées en détail dans les chapitres correspondants.

> 💡 D'autres nouveautés PostgreSQL 18 liées au SQL (transformation automatique de prédicats `OR` chaînés en `= ANY(…)`, *B-tree Skip Scan* qui permet enfin d'utiliser un index multicolonne sans condition sur la colonne de tête, sous-système d'E/S asynchrone, etc.) sont couvertes dans les Parties 3 et 4 de la formation.

---

## Public cible et niveau

### 👥 Pour qui ?

- **Développeurs** écrivant des requêtes applicatives quotidiennes  
- **Data Analysts** interrogeant des bases de données  
- **DevOps** devant manipuler des données en production  
- **Architectes** concevant des schémas de bases de données
- Toute personne ayant complété la Partie 1 ou possédant des bases solides en concepts de BDD

### 📋 Prérequis

**Requis** :
- Partie 1 complétée (ou connaissances équivalentes)
- Compréhension du modèle relationnel
- Connaissance des types de données PostgreSQL
- Capacité à créer des tables avec CREATE TABLE

**Recommandé** :
- Accès à une instance PostgreSQL 18 pour expérimenter
- Familiarité avec un client SQL (psql, DBeaver, pgAdmin)

---

## Ce que vous ne trouverez pas (encore) ici

Cette partie couvre le SQL **intermédiaire**. Les sujets suivants seront abordés dans les parties ultérieures :

### ➡️ Partie 3 (SQL Avancé)
- Sous-requêtes complexes et CTE
- CTE récursives
- Fonctions de fenêtrage (Window Functions)
- Techniques SQL avancées

### ➡️ Partie 4 (Performance)
- Indexation approfondie
- Optimisation de requêtes
- Analyse avec EXPLAIN
- Transactions et concurrence

### ➡️ Partie 5 (Production)
- Extensions et intégrations
- Déploiement en production
- Monitoring et troubleshooting

---

## Approche pédagogique recommandée

### 📖 Lecture active

Pour chaque chapitre :
1. **Lisez attentivement** la théorie  
2. **Visualisez** mentalement les opérations décrites  
3. **Notez** les concepts clés et les pièges  
4. **Expérimentez** (si possible) avec vos propres exemples

### 🧪 Expérimentation (fortement recommandée)

Bien que cette formation soit théorique, l'expérimentation est cruciale pour le SQL :

```sql
-- Créez une base de test
CREATE DATABASE formation_test;

-- Créez des tables d'exemple (syntaxe SQL standard recommandée)
CREATE TABLE utilisateurs (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    date_inscription DATE
);

-- Expérimentez avec les concepts appris
```

### 📝 Prenez des notes structurées

Organisez vos notes par chapitre :
- Syntaxe des commandes clés
- Pièges à éviter
- Cas d'usage réels
- Questions à approfondir

### 🔄 Révision régulière

Le SQL est un langage de pratique. Revenez régulièrement sur les chapitres précédents pour consolider vos acquis.

---

## Conseils pour progresser efficacement

### 💡 Comprenez l'ordre d'exécution

La **clé pour maîtriser SQL** est de comprendre que **l'ordre d'écriture n'est pas l'ordre d'exécution**. Le moteur évalue les clauses dans un ordre logique précis :

| Étape | Clause | Effet |
|:-----:|--------|-------|
| 1 | `FROM` + `JOIN` | Charge les tables sources et les combine |
| 2 | `WHERE` | Filtre les **lignes individuelles** |
| 3 | `GROUP BY` | Regroupe les lignes restantes |
| 4 | `HAVING` | Filtre les **groupes** (peut donc utiliser des agrégats) |
| 5 | `SELECT` | Évalue les expressions de projection et les agrégats |
| 5 bis | *Window functions* | Calculées **après** les agrégats mais **avant** `DISTINCT` |
| 6 | `DISTINCT` | Élimine les doublons sur le résultat projeté |
| 7 | `UNION`/`INTERSECT`/`EXCEPT` | Combine avec les autres requêtes |
| 8 | `ORDER BY` | Trie le résultat final (peut utiliser les alias du `SELECT`) |
| 9 | `LIMIT`/`OFFSET`/`FETCH` | Limite le nombre de lignes retournées |

**Conséquences pratiques fréquemment piégeantes** :

```sql
-- ❌ ERREUR : alias non utilisable dans WHERE (SELECT est évalué APRÈS WHERE)
SELECT prix_ttc, prix_ht * 1.20 AS prix_ttc FROM produits WHERE prix_ttc > 100;
--                                                              ^^^^^^^^^
--           ERROR: column "prix_ttc" does not exist

-- ✅ Solutions :
-- 1) Répéter l'expression
SELECT prix_ht * 1.20 AS prix_ttc FROM produits WHERE prix_ht * 1.20 > 100;

-- 2) Encapsuler dans une sous-requête / CTE (vu en partie 3)
WITH calcul AS (SELECT *, prix_ht * 1.20 AS prix_ttc FROM produits)  
SELECT prix_ttc FROM calcul WHERE prix_ttc > 100;  
```

> 💡 **Asymétrie utile à mémoriser** : les alias de `SELECT` sont **utilisables dans `ORDER BY`** (évalué après) et — extension PostgreSQL — dans `GROUP BY`, **mais pas dans `WHERE` ni `HAVING`**.

Cette compréhension est approfondie dans le Chapitre 5 (section dédiée à l'ordre d'exécution logique).

### ⚠️ Maîtrisez la logique NULL

Les valeurs NULL sont une source constante de confusion :
- `NULL = NULL` → `NULL` (pas TRUE !)  
- `NULL <> NULL` → `NULL` (pas TRUE !)
- Utilisez `IS NULL` et `IS NOT NULL`

### 🎯 Privilégiez la lisibilité

Écrivez du SQL **lisible** avant d'écrire du SQL « intelligent » :
- Indentez correctement
- Utilisez des alias explicites
- Commentez les requêtes complexes
- Décomposez avec des CTE (Partie 3)

### 🚀 Pensez en ensembles

SQL est un langage **ensembliste**, pas procédural :
- Ne pensez pas « pour chaque ligne, faire… »
- Pensez « sur cet ensemble de lignes, appliquer… »

#### Le déclic ensembliste : un exemple frappant

**Problème** : remettre tous les salaires d'un département à 30 000 € s'ils sont en-dessous, et n'y toucher sinon.

**Approche procédurale (mentale)** :
```
pour chaque employé du département IT :
    si son salaire < 30000 :
        modifier son salaire à 30000
```

**Approche ensembliste (SQL)** — une seule instruction, une seule passe, atomique :
```sql
UPDATE employes  
SET salaire = 30000  
WHERE departement = 'IT' AND salaire < 30000;  
```

Quand on vient d'un langage procédural (Python, Java, JavaScript), la tentation est forte de :
1. Lire les lignes en boucle,
2. Tester chaque ligne,
3. Faire un `UPDATE` par ligne.

Cela peut être **1 000 fois plus lent** (un aller-retour réseau par ligne, pas de plan d'exécution global, pas d'index utilisé efficacement). La pensée ensembliste est ce qui sépare un développeur qui « sait écrire du SQL » d'un développeur qui sait l'**exploiter**.

#### Anti-pattern à bannir dès maintenant : la boucle applicative

```python
# ❌ N+1 queries : 1 SELECT + N UPDATE
for emp_id in db.query("SELECT id FROM employes WHERE departement = 'IT'"):
    salaire = db.query("SELECT salaire FROM employes WHERE id = %s", emp_id)[0]
    if salaire < 30000:
        db.query("UPDATE employes SET salaire = 30000 WHERE id = %s", emp_id)
```

Cette forme produit `2 × N + 1` requêtes (sur 10 000 employés : 20 001 allers-retours). La version SQL ensembliste : **1 seule requête**. C'est le réflexe à acquérir.

---

## Schéma fil rouge de la partie 2

Pour rendre les exemples concrets et progressifs, plusieurs chapitres de cette partie utilisent un **mini-domaine e-commerce** récurrent :

```
┌──────────┐       1:N        ┌───────────┐
│ Clients  │──────────────────│ Commandes │
└──────────┘                  └───────────┘
                                    │ 1:N
                                    ▼
                              ┌──────────────────┐
                              │ Lignes_commande  │
                              └──────────────────┘
                                    │ N:1
                                    ▼
                              ┌──────────┐
                              │ Produits │
                              └──────────┘
```

**Mise en place minimale** (à recréer si vous voulez expérimenter) :

```sql
CREATE TABLE clients (
    id    INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom   VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    ville VARCHAR(100)
);

CREATE TABLE produits (
    id    INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom   VARCHAR(100) NOT NULL,
    prix  NUMERIC(10, 2) NOT NULL CHECK (prix > 0),
    stock INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE commandes (
    id            INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    client_id     INTEGER NOT NULL REFERENCES clients(id),
    date_commande DATE NOT NULL DEFAULT CURRENT_DATE,
    statut        VARCHAR(20) NOT NULL DEFAULT 'en_attente'
);

CREATE TABLE lignes_commande (
    id            INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    commande_id   INTEGER NOT NULL REFERENCES commandes(id) ON DELETE CASCADE,
    produit_id    INTEGER NOT NULL REFERENCES produits(id),
    quantite      INTEGER NOT NULL CHECK (quantite > 0),
    prix_unitaire NUMERIC(10, 2) NOT NULL,
    UNIQUE (commande_id, produit_id)
);
```

Ce schéma met en jeu :
- les **trois types de relations** (1:1 implicitement, 1:N entre clients et commandes, N:M entre commandes et produits via `lignes_commande`) ;
- les **contraintes** principales (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `NOT NULL`) ;
- des **valeurs par défaut** et de la **génération automatique** (`IDENTITY`, `CURRENT_DATE`) ;
- des **types** variés (entier, texte, numérique exact, date) ;
- un cas concret pour pratiquer les jointures et agrégations du chapitre 7 et 8.

> 📌 Les chapitres adaptent ce schéma à leurs besoins (en ajoutant par exemple `montant` à `commandes` pour les exemples d'agrégation), mais la trame reste reconnaissable.

---

## Structure type des chapitres

Chaque chapitre de cette partie suit une structure cohérente :

### 1️⃣ Introduction et contexte
Pourquoi ce sujet est important et comment il s'inscrit dans l'ensemble.

### 2️⃣ Concepts fondamentaux
Théorie, syntaxe de base et exemples simples.

### 3️⃣ Techniques intermédiaires
Cas d'usage plus complexes et subtilités.

### 4️⃣ Nouveautés PostgreSQL 18
Fonctionnalités spécifiques à la version 18 (si applicable).

### 5️⃣ Pièges et bonnes pratiques
Erreurs courantes à éviter et recommandations.

### 6️⃣ Résumé et points clés
Récapitulatif des concepts essentiels à retenir.

---

## Conventions SQL utilisées

### 📝 Style de code

Dans cette formation, nous utilisons les conventions suivantes :

```sql
-- Mots-clés SQL en MAJUSCULES
SELECT nom, prenom  
FROM utilisateurs  
WHERE age > 18  
ORDER BY nom ASC;  

-- Indentation pour la lisibilité
SELECT
    u.nom,
    u.email,
    COUNT(c.id) AS nb_commandes
FROM utilisateurs u  
LEFT JOIN commandes c ON u.id = c.user_id  
GROUP BY u.id, u.nom, u.email  
HAVING COUNT(c.id) > 0;  
```

### 🔤 Nommage

- **Tables** : pluriel, minuscules, séparées par underscores (`utilisateurs`, `commandes_lignes`)  
- **Colonnes** : singulier, snake_case (`date_creation`, `prix_unitaire`)  
- **Alias** : courts et significatifs (`u` pour users, `c` pour commandes)

### 💬 Commentaires

```sql
-- Commentaire sur une ligne

/*
   Commentaire sur
   plusieurs lignes
*/
```

---

## Outils recommandés pour cette partie

### 🖥️ Clients SQL

**psql** (CLI) :
```bash
psql -U postgres -d ma_base
\timing  -- Activer le timing
\x       -- Format étendu
```

**DBeaver** (GUI) :
- Éditeur SQL avec autocomplétion
- Visualisation des résultats
- Générateur de diagrammes ER

**pgAdmin** (GUI officiel) :
- Interface web complète
- Outils d'administration intégrés

### 📊 Jeux de données d'entraînement

Pour expérimenter, utilisez des jeux de données publics :
- **Pagila** : Base de données de location de films (clone de Sakila)  
- **DVD Rental** : Exemple classique PostgreSQL  
- **Northwind** : Base de données commerciale

### 🔧 Outils de formatage

- **pgFormatter** : Formater automatiquement votre SQL  
- **SQLFluff** : Linter SQL avec règles de style

---

## Après cette partie

Une fois cette deuxième partie maîtrisée, vous aurez une base solide en SQL intermédiaire. Vous serez prêt à aborder :

### ➡️ **Partie 3 : SQL Avancé et Modélisation**
Sous-requêtes, CTE récursives, fonctions de fenêtrage, partitionnement, modélisation avancée.

### ➡️ **Partie 4 : Administration, Performance et Architecture Avancée**
Transactions MVCC, indexation, observabilité, programmation serveur, haute disponibilité.

### ➡️ **Partie 5 : Écosystème, Production et Ouverture**
Extensions (PostGIS, pgvector), déploiement production, architectures modernes, intégrations.

---

## Ressources complémentaires

### 📚 Documentation PostgreSQL
- [SQL Commands Reference](https://www.postgresql.org/docs/18/sql-commands.html)  
- [Data Types](https://www.postgresql.org/docs/18/datatype.html)  
- [Functions and Operators](https://www.postgresql.org/docs/18/functions.html)

### 📖 Livres recommandés
- ***PostgreSQL : Up and Running*** — Regina Obe et Leo Hsu (O'Reilly)  
- ***The Art of PostgreSQL*** — Dimitri Fontaine  
- ***SQL Antipatterns*** — Bill Karwin (Pragmatic Bookshelf)

### 🎓 Pratique en ligne
- [PostgreSQL Exercises](https://pgexercises.com/)  
- [SQL Zoo](https://sqlzoo.net/)  
- [LeetCode SQL Problems](https://leetcode.com/problemset/database/)

### 👥 Communauté
- [Stack Overflow — PostgreSQL](https://stackoverflow.com/questions/tagged/postgresql)  
- [PostgreSQL Community Slack](https://pgtreats.info/slack-invite) (lien d'invitation officiel)  
- [Reddit r/PostgreSQL](https://www.reddit.com/r/PostgreSQL/)  
- [Listes de diffusion officielles](https://www.postgresql.org/list/) (le canal de communication historique du projet)

---

## Checklist avant de commencer

Avant d'entamer le Chapitre 5, assurez-vous de :

- ✅ Avoir complété la Partie 1 (ou avoir les connaissances équivalentes)  
- ✅ Comprendre la hiérarchie logique (Database → Schema → Table)  
- ✅ Savoir créer des tables avec les types de données de base  
- ✅ Avoir accès à une instance PostgreSQL (recommandé)  
- ✅ Être familier avec un client SQL (psql, DBeaver ou pgAdmin)  
- ✅ Avoir un environnement de test pour expérimenter

---

## Mot de conclusion

Le SQL est un langage à la fois **simple et profond**. Sa syntaxe de base est accessible, mais sa maîtrise demande de la pratique et une compréhension des concepts sous-jacents.

Cette partie vous donnera les outils pour écrire des requêtes efficaces, maintenir l'intégrité de vos données et extraire l'information dont vous avez besoin. Le SQL est un investissement durable : ces compétences vous serviront tout au long de votre carrière, quel que soit le SGBD utilisé.

N'oubliez pas : **le meilleur moyen d'apprendre SQL est de l'écrire**. Lisez, comprenez, puis expérimentez !

**Prêt à interroger des données ? Allons-y ! 🚀**

---


⏭️ [Requêtes de Sélection (DQL)](/05-requetes-de-selection/README.md)
