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

Dans cette partie, nous couvrons les quatre sous-langages SQL fondamentaux que vous utiliserez quotidiennement :
- **DQL** (Data Query Language) : Interrogation avec SELECT
- **DML** (Data Manipulation Language) : INSERT, UPDATE, DELETE
- **Relations** : Jointures et intégrité référentielle
- **Agrégation** : GROUP BY et fonctions statistiques

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
- Insérer des données efficacement (simple, multiple, COPY)
- Mettre à jour et supprimer des enregistrements de manière ciblée
- Utiliser la puissante clause RETURNING (avec OLD/NEW en PG 18)
- Maîtriser les "upserts" avec ON CONFLICT
- Exploiter la nouvelle commande MERGE de PostgreSQL
- Comprendre les différences entre TRUNCATE et DELETE

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
Modifier le contenu de vos tables : INSERT, UPDATE, DELETE, TRUNCATE. Découverte des nouveautés PostgreSQL 18 (OLD/NEW dans RETURNING, améliorations COPY) et des techniques avancées comme les upserts.

### 🔄 **Chapitre 7 : Relations et Jointures**
Le fondement du modèle relationnel : contraintes d'intégrité, théorie des jointures, et tous les types de JOIN. Introduction aux contraintes temporelles de PostgreSQL 18.

### 📊 **Chapitre 8 : Agrégation et Groupement**
Synthétiser l'information : fonctions d'agrégation, GROUP BY, HAVING, et techniques avancées (ROLLUP, CUBE, filtres d'agrégation).

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
1. SELECT simple → Filtrage → Tri → Pagination
2. Manipulations de base → Techniques avancées (upserts, MERGE)
3. Contraintes → Jointures simples → Jointures complexes
4. Agrégations de base → Extensions avancées

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
- **Support OLD et NEW dans RETURNING** : Accès aux valeurs avant/après modification dans UPDATE
- **Améliorations COPY** : Gestion optimisée du marqueur de fin `\.` en CSV
- **MERGE avec OLD/NEW** : Consolidation de données avec accès aux anciennes valeurs

### 🔗 Contraintes évoluées
- **Contraintes temporelles (Temporal Constraints)** : Validation de périodes de temps et chevauchements
- Gestion native des périodes de validité dans les données

### ⚡ Optimisations du planificateur
- **Optimisation des OR-clauses** : Transformation automatique en ANY pour de meilleures performances
- Amélioration du traitement des conditions multiples

Ces nouveautés seront expliquées en détail dans les chapitres correspondants.

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

-- Créez des tables d'exemple
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
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

La **clé pour maîtriser SQL** est de comprendre comment PostgreSQL exécute vos requêtes. L'ordre d'écriture n'est pas l'ordre d'exécution :

**Ordre d'écriture** :
```
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY
```

**Ordre d'exécution logique** :
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

Cette compréhension sera approfondie dans le Chapitre 5.

### ⚠️ Maîtrisez la logique NULL

Les valeurs NULL sont une source constante de confusion :
- `NULL = NULL` → `NULL` (pas TRUE !)
- `NULL <> NULL` → `NULL` (pas TRUE !)
- Utilisez `IS NULL` et `IS NOT NULL`

### 🎯 Privilégiez la lisibilité

Écrivez du SQL **lisible** avant d'écrire du SQL "intelligent" :
- Indentez correctement
- Utilisez des alias explicites
- Commentez les requêtes complexes
- Décomposez avec des CTE (Partie 3)

### 🚀 Pensez en ensembles

SQL est un langage **ensembliste**, pas procédural :
- Ne pensez pas "pour chaque ligne, faire..."
- Pensez "sur cet ensemble de lignes, appliquer..."

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
Features spécifiques à la version 18 (si applicable).

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
- **"PostgreSQL: Up and Running"** - Regina Obe, Leo Hsu (O'Reilly)
- **"The Art of PostgreSQL"** - Dimitri Fontaine
- **"SQL Antipatterns"** - Bill Karwin (Pragmatic Bookshelf)

### 🎓 Pratique en ligne
- [PostgreSQL Exercises](https://pgexercises.com/)
- [SQL Zoo](https://sqlzoo.net/)
- [LeetCode SQL Problems](https://leetcode.com/problemset/database/)

### 👥 Communauté
- [Stack Overflow - PostgreSQL](https://stackoverflow.com/questions/tagged/postgresql)
- [PostgreSQL Slack](https://postgres-slack.herokuapp.com/)
- [Reddit r/PostgreSQL](https://www.reddit.com/r/PostgreSQL/)

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
