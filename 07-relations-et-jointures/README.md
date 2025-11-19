🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7. Relations et Jointures

## Introduction au Chapitre

Bienvenue dans le chapitre le plus important de votre apprentissage de PostgreSQL ! Les **relations** et les **jointures** constituent le cœur même des bases de données relationnelles. C'est ce qui distingue une base de données relationnelle d'un simple fichier Excel ou d'une collection de fichiers.

Dans ce chapitre, vous allez découvrir :
- Comment **structurer vos données** en tables reliées entre elles
- Comment **garantir la cohérence** de vos données avec des contraintes
- Comment **combiner des informations** provenant de plusieurs tables
- Les différents **types de jointures** et quand les utiliser

Maîtriser ces concepts vous permettra de concevoir des bases de données efficaces et d'écrire des requêtes SQL puissantes pour exploiter pleinement vos données.

---

## Qu'est-ce qu'une Base de Données Relationnelle ?

### Le Concept de Relation

Le terme **"relationnel"** dans "base de données relationnelle" ne signifie pas simplement "relier des tables entre elles" (bien que ce soit une partie importante). Il fait référence à un concept mathématique plus profond : chaque table est une **relation** au sens mathématique, c'est-à-dire un ensemble de tuples (lignes) partageant le même schéma (colonnes).

Cependant, dans la pratique quotidienne, quand on parle de "relations" en SQL, on fait généralement référence aux **liens entre les tables**.

### Exemple : Un Système de Commandes

Imaginons un système simple de gestion de commandes en ligne. Nous pourrions avoir :

#### Approche Naïve : Une Seule Table (❌ Mauvaise Pratique)

```
Table : commandes_tout_en_un
┌────┬─────────────┬──────────────────┬──────────┬──────────────┬────────┬──────────┐
│ id │ client_nom  │ client_email     │ client_  │ produit_nom  │ produit│ quantite │
│    │             │                  │ ville    │              │ _prix  │          │
├────┼─────────────┼──────────────────┼──────────┼──────────────┼────────┼──────────┤
│ 1  │ Alice       │ alice@example.com│ Paris    │ Ordinateur   │ 999.99 │ 1        │
│ 2  │ Alice       │ alice@example.com│ Paris    │ Souris       │ 25.00  │ 2        │
│ 3  │ Bob         │ bob@example.com  │ Lyon     │ Clavier      │ 75.00  │ 1        │
│ 4  │ Alice       │ alice@example.com│ Paris    │ Écran        │ 350.00 │ 1        │
└────┴─────────────┴──────────────────┴──────────┴──────────────┴────────┴──────────┘
```

**Problèmes de cette approche** :
1. **Redondance** : Les informations d'Alice (nom, email, ville) sont répétées 3 fois
2. **Incohérence** : Si Alice change d'email, il faut modifier 3 lignes (risque d'oubli)
3. **Gaspillage d'espace** : Les données sont dupliquées inutilement
4. **Anomalies de mise à jour** : Que se passe-t-il si on modifie l'email d'Alice sur une seule ligne ?
5. **Anomalies de suppression** : Si on supprime toutes les commandes d'Alice, on perd toutes ses informations
6. **Anomalies d'insertion** : Comment ajouter un nouveau client qui n'a pas encore commandé ?

#### Approche Relationnelle : Tables Séparées (✅ Bonne Pratique)

```
Table : clients
┌────┬─────────┬──────────────────┬──────────┐
│ id │ nom     │ email            │ ville    │
├────┼─────────┼──────────────────┼──────────┤
│ 1  │ Alice   │ alice@example.com│ Paris    │
│ 2  │ Bob     │ bob@example.com  │ Lyon     │
└────┴─────────┴──────────────────┴──────────┘

Table : produits
┌────┬─────────────┬────────┐
│ id │ nom         │ prix   │
├────┼─────────────┼────────┤
│ 10 │ Ordinateur  │ 999.99 │
│ 20 │ Souris      │ 25.00  │
│ 30 │ Clavier     │ 75.00  │
│ 40 │ Écran       │ 350.00 │
└────┴─────────────┴────────┘

Table : commandes
┌────┬───────────┬────────────┬──────────┬──────────────┐
│ id │ client_id │ produit_id │ quantite │ date_commande│
├────┼───────────┼────────────┼──────────┼──────────────┤
│ 1  │ 1         │ 10         │ 1        │ 2025-01-10   │
│ 2  │ 1         │ 20         │ 2        │ 2025-01-11   │
│ 3  │ 2         │ 30         │ 1        │ 2025-01-12   │
│ 4  │ 1         │ 40         │ 1        │ 2025-01-13   │
└────┴───────────┴────────────┴──────────┴──────────────┘
```

**Avantages de cette approche** :
1. ✅ **Pas de redondance** : Chaque information n'existe qu'une seule fois
2. ✅ **Cohérence** : Modifier l'email d'Alice se fait en un seul endroit
3. ✅ **Économie d'espace** : Moins de duplication
4. ✅ **Intégrité** : Les données restent cohérentes
5. ✅ **Flexibilité** : On peut ajouter un client sans commande, ou une commande sans tout répéter

**Le lien** : Les colonnes `client_id` et `produit_id` dans la table `commandes` créent les **relations** avec les tables `clients` et `produits`.

---

## Les Trois Types de Relations

Dans une base de données relationnelle, trois types de relations existent entre les tables.

### 1. Relation Un-à-Un (1:1)

**Définition** : Une ligne de la table A est associée à **au maximum une** ligne de la table B, et vice-versa.

**Exemple** : Un employé a un seul bureau attitré, et un bureau est occupé par un seul employé.

```
Table : employes                Table : bureaux
┌────┬─────────┬───────────┐    ┌────┬─────────┬────────┐
│ id │ nom     │ bureau_id │    │ id │ numero  │ etage  │
├────┼─────────┼───────────┤    ├────┼─────────┼────────┤
│ 1  │ Alice   │ 101       │───→│ 101│ B-301   │ 3      │
│ 2  │ Bob     │ 102       │───→│ 102│ B-302   │ 3      │
│ 3  │ Charlie │ 103       │───→│ 103│ B-401   │ 4      │
└────┴─────────┴───────────┘    └────┴─────────┴────────┘
```

**Représentation graphique** :
```
Employé [1] ─────── [1] Bureau
```

**Cas d'usage** :
- Employé → Bureau attitré
- Utilisateur → Profil utilisateur
- Pays → Capitale (une capitale principale par pays)
- Personne → Carte d'identité

**Note** : Les relations 1:1 sont relativement rares. Souvent, on pourrait fusionner les deux tables en une seule, mais on les sépare pour :
- Des raisons de sécurité (isoler les données sensibles)
- Des raisons de performance (table principale légère, détails lourds dans une autre table)
- Des raisons organisationnelles (séparation des préoccupations)

### 2. Relation Un-à-Plusieurs (1:N)

**Définition** : Une ligne de la table A peut être associée à **plusieurs** lignes de la table B, mais une ligne de B n'est associée qu'à **une seule** ligne de A.

**C'est le type de relation le plus courant !**

**Exemple** : Un client peut passer plusieurs commandes, mais chaque commande appartient à un seul client.

```
Table : clients                  Table : commandes
┌────┬─────────┐                 ┌────┬───────────┬──────────┐
│ id │ nom     │                 │ id │ client_id │ montant  │
├────┼─────────┤                 ├────┼───────────┼──────────┤
│ 1  │ Alice   │───┬────────────→│ 1  │ 1         │ 150.00   │
│ 2  │ Bob     │   │   ┌────────→│ 2  │ 1         │ 220.00   │
└────┴─────────┘   │   │   ┌────→│ 3  │ 1         │ 99.00    │
                   │   │   │     │ 4  │ 2         │ 175.00   │
                   └───┴───┴────→│ 5  │ 2         │ 80.00    │
                                 └────┴───────────┴──────────┘
```

**Représentation graphique** :
```
Client [1] ─────── [N] Commandes
```

**Autres exemples** :
- Auteur → Livres (un auteur écrit plusieurs livres)
- Catégorie → Produits (une catégorie contient plusieurs produits)
- Département → Employés (un département a plusieurs employés)
- Pays → Villes (un pays a plusieurs villes)
- Album → Chansons (un album contient plusieurs chansons)

**Implémentation** : La table "plusieurs" (N) contient une **clé étrangère** pointant vers la table "un" (1).

### 3. Relation Plusieurs-à-Plusieurs (N:M)

**Définition** : Une ligne de la table A peut être associée à **plusieurs** lignes de la table B, et vice-versa.

**Exemple** : Un étudiant peut s'inscrire à plusieurs cours, et un cours peut avoir plusieurs étudiants.

```
Table : etudiants              Table : cours
┌────┬──────────┐              ┌────┬─────────────────┐
│ id │ nom      │              │ id │ nom             │
├────┼──────────┤              ├────┼─────────────────┤
│ 1  │ Alice    │              │ 10 │ Mathématiques   │
│ 2  │ Bob      │              │ 20 │ Physique        │
│ 3  │ Charlie  │              │ 30 │ Informatique    │
└────┴──────────┘              └────┴─────────────────┘

      ↓           Table de jonction           ↓

          Table : inscriptions
          ┌────┬─────────────┬──────────┐
          │ id │ etudiant_id │ cours_id │
          ├────┼─────────────┼──────────┤
          │ 1  │ 1           │ 10       │  Alice → Maths
          │ 2  │ 1           │ 20       │  Alice → Physique
          │ 3  │ 2           │ 10       │  Bob → Maths
          │ 4  │ 2           │ 30       │  Bob → Info
          │ 5  │ 3           │ 20       │  Charlie → Physique
          │ 6  │ 3           │ 30       │  Charlie → Info
          └────┴─────────────┴──────────┘
```

**Représentation graphique** :
```
Étudiants [N] ─────── [M] Cours
              (via table de jonction)
```

**Implémentation** : On utilise une **table de jonction** (ou table associative) qui contient deux clés étrangères, une vers chaque table.

**Autres exemples** :
- Acteurs ↔ Films (un acteur joue dans plusieurs films, un film a plusieurs acteurs)
- Tags ↔ Articles (un article peut avoir plusieurs tags, un tag peut être sur plusieurs articles)
- Produits ↔ Commandes (un produit peut être dans plusieurs commandes, une commande contient plusieurs produits)
- Auteurs ↔ Livres (dans certains cas, un livre peut avoir plusieurs auteurs)
- Projets ↔ Employés (un employé travaille sur plusieurs projets, un projet a plusieurs employés)

---

## Pourquoi Séparer les Données en Tables ?

### 1. Éliminer la Redondance (Normalisation)

**Problème** : Si on stocke l'adresse complète d'un client dans chaque commande, que se passe-t-il quand le client déménage ?

**Solution** : Stocker l'adresse une seule fois dans la table `clients`, et référencer le client dans la table `commandes`.

### 2. Garantir l'Intégrité des Données

**Problème** : Si un utilisateur entre "Paris" dans une commande et "paris" dans une autre, comment retrouver toutes les commandes de Paris ?

**Solution** : Créer une table `villes` avec un ID unique, et utiliser cet ID dans les autres tables.

### 3. Faciliter les Mises à Jour

**Problème** : Si le prix d'un produit change, faut-il modifier toutes les commandes passées ?

**Solution** : Non ! Les commandes gardent une copie du prix au moment de l'achat, mais les nouvelles commandes utilisent le prix actuel depuis la table `produits`.

### 4. Permettre des Requêtes Complexes

Avec des tables séparées, on peut facilement répondre à des questions comme :
- "Quels clients ont commandé plus de 1000€ ?"
- "Quels produits n'ont jamais été vendus ?"
- "Quel est le top 10 des clients par chiffre d'affaires ?"

C'est là que les **jointures** entrent en jeu !

---

## Les Jointures : Recombiner les Données

### Le Problème

Nous avons séparé nos données en plusieurs tables (bonne pratique !), mais maintenant nous voulons les **recombiner** pour afficher des informations complètes.

**Question** : Comment afficher le nom du client avec chaque commande ?

### La Solution : Les Jointures (JOINs)

Les **jointures** permettent de combiner des lignes de plusieurs tables en fonction d'une **condition de correspondance**.

**Exemple simple** :

```sql
SELECT
    clients.nom AS client,
    commandes.id AS numero_commande,
    commandes.montant
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

**Résultat** :

```
┌─────────┬──────────────────┬──────────┐
│ client  │ numero_commande  │ montant  │
├─────────┼──────────────────┼──────────┤
│ Alice   │ 1                │ 150.00   │
│ Alice   │ 2                │ 220.00   │
│ Alice   │ 3                │ 99.00    │
│ Bob     │ 4                │ 175.00   │
│ Bob     │ 5                │ 80.00    │
└─────────┴──────────────────┴──────────┘
```

**Magic !** Les données des deux tables sont combinées en une seule vue.

### Les Différents Types de Jointures

PostgreSQL offre plusieurs types de jointures :

1. **INNER JOIN** : Retourne seulement les lignes qui ont une correspondance dans les deux tables
2. **LEFT JOIN** : Retourne toutes les lignes de la table de gauche, même sans correspondance
3. **RIGHT JOIN** : Retourne toutes les lignes de la table de droite, même sans correspondance
4. **FULL OUTER JOIN** : Retourne toutes les lignes des deux tables
5. **CROSS JOIN** : Produit cartésien (toutes les combinaisons)
6. **SELF JOIN** : Une table jointe à elle-même

Nous explorerons chacun de ces types en détail dans les sections suivantes.

---

## Structure de ce Chapitre

Ce chapitre est organisé en sections progressives pour vous guider du débutant à l'expert :

### 7.1. Contraintes d'Intégrité (PK, FK, UNIQUE, CHECK, NOT NULL)
Vous apprendrez à **garantir la cohérence** de vos données en définissant des règles au niveau de la base de données.

### 7.2. Nouveauté PostgreSQL 18 : Contraintes Temporelles
Découvrez comment gérer les **périodes de validité** et éviter les chevauchements temporels nativement.

### 7.3. Théorie des Ensembles et Jointures
Comprenez les **fondements mathématiques** : produit cartésien, sélection, projection.

### 7.4. Types de Jointures (INNER, LEFT, RIGHT, FULL, CROSS)
Maîtrisez **tous les types de jointures** et sachez quand utiliser chacun.

### 7.5. Self-Joins
Apprenez à **joindre une table à elle-même** pour gérer les hiérarchies et comparaisons.

### 7.6. Jointures Latérales (LATERAL JOIN)
Explorez la **corrélation avancée** pour des requêtes sophistiquées.

### 7.7. Anti-Jointures et Semi-Jointures (NOT EXISTS, NOT IN, EXCEPT)
Trouvez ce qui **existe** ou **n'existe pas** dans vos données.

---

## Concepts Clés à Maîtriser

À la fin de ce chapitre, vous serez capable de :

### 1. Concevoir un Schéma Relationnel
- ✅ Identifier les entités de votre domaine métier
- ✅ Déterminer les relations entre ces entités (1:1, 1:N, N:M)
- ✅ Normaliser vos tables pour éliminer la redondance
- ✅ Définir les clés primaires et étrangères appropriées

### 2. Garantir l'Intégrité des Données
- ✅ Utiliser les contraintes PRIMARY KEY et FOREIGN KEY
- ✅ Appliquer des contraintes UNIQUE, NOT NULL, CHECK
- ✅ Comprendre les actions CASCADE, RESTRICT, SET NULL
- ✅ Gérer les contraintes temporelles (PostgreSQL 18)

### 3. Écrire des Jointures Efficaces
- ✅ Choisir le bon type de jointure selon vos besoins
- ✅ Combiner plusieurs tables en une seule requête
- ✅ Éviter les pièges courants (produit cartésien accidentel, NULL dans les jointures)
- ✅ Optimiser les performances avec les bons index

### 4. Résoudre des Problèmes Complexes
- ✅ Trouver les clients sans commande (anti-jointure)
- ✅ Obtenir les top N par groupe (LATERAL JOIN)
- ✅ Gérer les hiérarchies (self-join)
- ✅ Combiner des données de sources multiples

---

## Prérequis

Avant de commencer ce chapitre, assurez-vous de maîtriser :

- ✅ Les bases de SQL (SELECT, FROM, WHERE)
- ✅ La création de tables (CREATE TABLE)
- ✅ L'insertion de données (INSERT)
- ✅ Les types de données de base (INTEGER, VARCHAR, DATE, etc.)

Si vous avez des doutes, n'hésitez pas à réviser les chapitres précédents.

---

## Conseils pour Bien Apprendre

### 1. Visualisez les Données

Avant d'écrire une jointure complexe, **dessinez** vos tables et leurs relations sur papier. Cela vous aidera à comprendre quelle jointure utiliser.

```
Clients ──[1:N]── Commandes ──[N:M]── Produits
   │
   └──[1:1]── Profils
```

### 2. Commencez Simple

Ne cherchez pas à écrire une jointure de 5 tables dès le début. Commencez par joindre 2 tables, vérifiez le résultat, puis ajoutez une 3ème table, etc.

### 3. Utilisez des Alias

Les alias rendent vos requêtes plus lisibles :

```sql
-- ❌ Difficile à lire
SELECT clients.nom, commandes.montant
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;

-- ✅ Plus lisible
SELECT c.nom, cmd.montant
FROM clients AS c
INNER JOIN commandes AS cmd ON c.id = cmd.client_id;
```

### 4. Testez sur des Petits Échantillons

Avant de lancer une jointure sur des millions de lignes, testez d'abord sur quelques lignes avec `LIMIT` :

```sql
SELECT ...
FROM table1
JOIN table2 ON ...
LIMIT 10;
```

### 5. Utilisez EXPLAIN

Pour comprendre comment PostgreSQL exécute votre requête :

```sql
EXPLAIN ANALYZE
SELECT ...
FROM table1
JOIN table2 ON ...;
```

---

## Avertissements et Pièges Courants

### ⚠️ Piège 1 : Le Produit Cartésien Accidentel

**Erreur** : Oublier la condition de jointure.

```sql
-- ❌ DANGER : Produit cartésien !
SELECT *
FROM clients, commandes;
-- Si clients a 1000 lignes et commandes 5000 → 5 000 000 lignes !
```

**Solution** : Toujours spécifier la condition ON.

```sql
-- ✅ Correct
SELECT *
FROM clients
INNER JOIN commandes ON clients.id = commandes.client_id;
```

### ⚠️ Piège 2 : Colonnes Ambiguës

**Erreur** : Ne pas qualifier les colonnes quand deux tables ont des colonnes du même nom.

```sql
-- ❌ Erreur si les deux tables ont une colonne "id"
SELECT id, nom
FROM clients
JOIN commandes ON clients.id = commandes.client_id;
-- ERROR: column reference "id" is ambiguous
```

**Solution** : Toujours qualifier ou utiliser des alias.

```sql
-- ✅ Correct
SELECT clients.id, clients.nom, commandes.id AS commande_id
FROM clients
JOIN commandes ON clients.id = commandes.client_id;
```

### ⚠️ Piège 3 : NULL dans les Jointures

**Attention** : `NULL = NULL` retourne `NULL` (pas `TRUE`), donc les lignes avec NULL dans la clé de jointure ne correspondront jamais.

```sql
-- Si client_id est NULL, la ligne ne sera jamais jointe
SELECT *
FROM commandes
JOIN clients ON commandes.client_id = clients.id;
```

### ⚠️ Piège 4 : Mauvais Type de Jointure

Utiliser `INNER JOIN` quand on voulait `LEFT JOIN` (et vice-versa) est une erreur fréquente.

- **INNER JOIN** exclut les lignes sans correspondance
- **LEFT JOIN** les inclut (avec NULL)

**Conseil** : Posez-vous toujours la question : "Est-ce que je veux inclure les lignes sans correspondance ?"

---

## Terminologie Importante

Avant de continuer, assurons-nous de bien comprendre ces termes :

| Terme | Définition | Exemple |
|-------|------------|---------|
| **Clé Primaire (PK)** | Identifiant unique d'une ligne | `clients.id` |
| **Clé Étrangère (FK)** | Référence vers la PK d'une autre table | `commandes.client_id` |
| **Table Parent** | Table référencée par une FK | `clients` (référencé par commandes) |
| **Table Enfant** | Table contenant une FK | `commandes` (contient client_id) |
| **Relation** | Lien entre deux tables | clients → commandes |
| **Cardinalité** | Type de relation (1:1, 1:N, N:M) | clients [1:N] commandes |
| **Jointure** | Opération combinant des tables | INNER JOIN, LEFT JOIN, etc. |
| **Condition de jointure** | Critère de correspondance | `ON clients.id = commandes.client_id` |
| **Intégrité référentielle** | Cohérence des références entre tables | Un client_id doit exister dans clients |

---

## Exemple Fil Rouge

Tout au long de ce chapitre, nous utiliserons un exemple cohérent : **un système de e-commerce simplifié**.

### Schéma de Base

```sql
-- Clients
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    ville VARCHAR(100)
);

-- Produits
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prix NUMERIC(10, 2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0
);

-- Commandes
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL REFERENCES clients(id),
    date_commande DATE NOT NULL DEFAULT CURRENT_DATE,
    statut VARCHAR(20) DEFAULT 'en_attente'
);

-- Lignes de commande (relation N:M entre commandes et produits)
CREATE TABLE lignes_commande (
    id SERIAL PRIMARY KEY,
    commande_id INTEGER NOT NULL REFERENCES commandes(id),
    produit_id INTEGER NOT NULL REFERENCES produits(id),
    quantite INTEGER NOT NULL CHECK (quantite > 0),
    prix_unitaire NUMERIC(10, 2) NOT NULL
);
```

### Diagramme des Relations

```
┌──────────┐       1:N        ┌───────────┐
│ Clients  │──────────────────│ Commandes │
└──────────┘                  └───────────┘
                                    │
                                    │ 1:N
                                    ↓
                              ┌──────────────────┐
                              │ Lignes_commande  │
                              └──────────────────┘
                                    │
                                    │ N:1
                                    ↓
                              ┌──────────┐
                              │ Produits │
                              └──────────┘
```

Ce schéma sera notre fil conducteur pour illustrer tous les concepts de ce chapitre.

---

## À Vous de Jouer !

Vous êtes maintenant prêt à plonger dans le monde fascinant des relations et jointures. Chaque section de ce chapitre vous apportera de nouvelles compétences et vous rapprochera de la maîtrise de PostgreSQL.

**Rappel de la structure** :
1. 🔒 Contraintes d'intégrité (garantir la cohérence)
2. ⏰ Contraintes temporelles PostgreSQL 18 (nouveauté)
3. 📐 Théorie mathématique (comprendre les fondements)
4. 🔗 Types de jointures (combiner les données)
5. 🪞 Self-joins (relations hiérarchiques)
6. ↔️ LATERAL JOIN (corrélation avancée)
7. ⛔ Anti-jointures (trouver ce qui n'existe pas)

**Conseil final** : Ne cherchez pas à tout mémoriser. Concentrez-vous sur la **compréhension des concepts**, la **pratique** viendra naturellement.

Bonne chance, et que les jointures soient avec vous ! 🚀

---

## Ce que Vous Allez Apprendre

### Compétences Techniques

À la fin de ce chapitre, vous saurez :
- ✅ Modéliser des relations entre entités
- ✅ Implémenter des contraintes d'intégrité
- ✅ Écrire des jointures simples et complexes
- ✅ Choisir le bon type de jointure
- ✅ Optimiser les performances de vos jointures
- ✅ Déboguer les erreurs de jointure
- ✅ Utiliser les fonctionnalités avancées (LATERAL, EXISTS)

### Compétences Analytiques

Vous serez capable de :
- ✅ Analyser un schéma de base de données
- ✅ Identifier les relations entre tables
- ✅ Diagnostiquer les problèmes d'intégrité
- ✅ Concevoir des schémas efficaces et maintenables
- ✅ Répondre à des questions métier complexes avec SQL

### Mindset du Développeur

Vous développerez :
- ✅ Une pensée relationnelle (comment structurer les données)
- ✅ Une approche méthodique (résoudre des problèmes étape par étape)
- ✅ Un sens critique (évaluer différentes approches)
- ✅ Une rigueur (garantir la qualité des données)

---

**Prêt ?** Commençons par la première section : les contraintes d'intégrité !

⏭️ [Contraintes d'intégrité : PK, FK, UNIQUE, CHECK, NOT NULL](/07-relations-et-jointures/01-contraintes-integrite.md)
