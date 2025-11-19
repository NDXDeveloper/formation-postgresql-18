🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6. Manipulation des Données (DML)

## Introduction générale

Après avoir appris à **créer** des structures de données (tables, colonnes, contraintes) avec le DDL (Data Definition Language), il est temps d'apprendre à **manipuler** les données elles-mêmes. C'est le rôle du **DML** : Data Manipulation Language.

Le DML regroupe l'ensemble des commandes SQL qui permettent de :
- **Insérer** de nouvelles données dans les tables
- **Modifier** des données existantes
- **Supprimer** des données
- **Interroger** les données (bien que SELECT soit parfois classé dans le DQL - Data Query Language)

Ces opérations constituent le cœur de l'utilisation quotidienne d'une base de données. Comprendre le DML est essentiel pour tout développeur ou administrateur de bases de données.

---

## Qu'est-ce que le DML ?

### Définition

Le **DML (Data Manipulation Language)** est un sous-ensemble du langage SQL dédié à la manipulation des données contenues dans les tables d'une base de données. Contrairement au DDL qui modifie la **structure** des objets (CREATE, ALTER, DROP), le DML modifie le **contenu** des tables.

### Les quatre commandes fondamentales

Le DML comprend quatre commandes principales, souvent appelées **CRUD** dans le monde du développement :

| Commande | Opération CRUD | Description | Exemple d'usage |
|----------|---------------|-------------|-----------------|
| **INSERT** | **C**reate | Ajoute de nouvelles lignes | Enregistrer un nouvel utilisateur |
| **SELECT** | **R**ead | Lit et récupère des données | Afficher la liste des produits |
| **UPDATE** | **U**pdate | Modifie des lignes existantes | Changer le prix d'un produit |
| **DELETE** | **D**elete | Supprime des lignes | Supprimer un compte utilisateur |

> **Note** : Certains considèrent SELECT comme faisant partie du DQL (Data Query Language) plutôt que du DML, car il ne modifie pas les données. Dans ce cours, nous l'incluons car il fait partie intégrante de la manipulation des données au sens large.

### Analogie avec un classeur

Imaginez une base de données comme un classeur de fiches :

- **DDL (CREATE TABLE)** : Créer un nouveau classeur avec des intercalaires et des colonnes prédéfinies
- **DML INSERT** : Ajouter de nouvelles fiches dans le classeur
- **DML SELECT** : Consulter les fiches pour lire les informations
- **DML UPDATE** : Corriger ou mettre à jour une fiche existante
- **DML DELETE** : Retirer une fiche du classeur

---

## Pourquoi le DML est-il crucial ?

### 1. Le cycle de vie des données

Une application moderne suit généralement ce cycle :

```
┌─────────────┐
│   CREATE    │  DDL : Créer la structure
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   INSERT    │  DML : Ajouter des données
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   SELECT    │  DML : Lire les données
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   UPDATE    │  DML : Modifier les données
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   DELETE    │  DML : Supprimer les données
└─────────────┘
```

Le DML est utilisé dans **90% des opérations** d'une application en production.

### 2. L'interaction quotidienne avec les données

Voici la fréquence typique des opérations dans une application web :

| Opération | Fréquence | Exemple |
|-----------|-----------|---------|
| **SELECT** | 80-90% | Afficher une page produit, charger un profil utilisateur |
| **INSERT** | 5-10% | Créer un compte, passer une commande |
| **UPDATE** | 3-7% | Mettre à jour un panier, modifier un profil |
| **DELETE** | 1-3% | Supprimer un commentaire, vider le panier |

### 3. La base de toute application

Quel que soit le type d'application, le DML est omniprésent :

**E-commerce** :
- INSERT : Enregistrer une nouvelle commande
- SELECT : Afficher le catalogue de produits
- UPDATE : Mettre à jour le stock après un achat
- DELETE : Supprimer un article du panier

**Réseau social** :
- INSERT : Publier un nouveau post
- SELECT : Afficher le fil d'actualité
- UPDATE : Modifier un commentaire
- DELETE : Supprimer une photo

**Gestion d'entreprise** :
- INSERT : Enregistrer une nouvelle facture
- SELECT : Générer un rapport mensuel
- UPDATE : Corriger une erreur de saisie
- DELETE : Archiver des données obsolètes

---

## Concepts fondamentaux à maîtriser

Avant de plonger dans les détails de chaque commande DML, il est important de comprendre quelques concepts transversaux.

### 1. Les transactions

Une **transaction** est un groupe d'opérations qui doivent toutes réussir ou toutes échouer ensemble.

```sql
BEGIN;  -- Début de la transaction

INSERT INTO comptes (nom, solde) VALUES ('Alice', 1000);
UPDATE comptes SET solde = solde - 100 WHERE nom = 'Alice';
INSERT INTO operations (compte, montant) VALUES ('Alice', -100);

COMMIT;  -- Valider toutes les opérations
-- Ou ROLLBACK; pour tout annuler
```

**Propriétés ACID** :
- **A**tomicité : Tout ou rien
- **C**ohérence : Respect des contraintes
- **I**solation : Les transactions ne s'interfèrent pas
- **D**urabilité : Les données committées sont permanentes

### 2. Les contraintes d'intégrité

Les contraintes protègent vos données lors des opérations DML :

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,              -- Clé primaire : unicité garantie
    email VARCHAR(255) UNIQUE NOT NULL, -- Unique et obligatoire
    age INTEGER CHECK (age >= 18),      -- Contrainte de validation
    departement_id INTEGER REFERENCES departements(id)  -- Clé étrangère
);
```

Les opérations DML doivent respecter ces contraintes :

```sql
-- ❌ Erreur : email NULL
INSERT INTO utilisateurs (age) VALUES (25);

-- ❌ Erreur : age < 18
INSERT INTO utilisateurs (email, age) VALUES ('alice@ex.com', 15);

-- ❌ Erreur : email dupliqué
INSERT INTO utilisateurs (email, age) VALUES ('alice@ex.com', 25);
-- Si alice@ex.com existe déjà
```

### 3. La clause WHERE : Le filtre universel

La clause `WHERE` est le mécanisme de filtrage utilisé par SELECT, UPDATE et DELETE :

```sql
-- Sélectionner des lignes spécifiques
SELECT * FROM produits WHERE prix > 100;

-- Modifier des lignes spécifiques
UPDATE produits SET prix = prix * 0.9 WHERE categorie = 'Soldes';

-- Supprimer des lignes spécifiques
DELETE FROM logs WHERE date_creation < '2024-01-01';
```

> **⚠️ AVERTISSEMENT CRITIQUE** : Sans clause WHERE, l'opération s'applique à **TOUTES** les lignes !

```sql
-- ❌ DANGER : Supprime TOUS les utilisateurs
DELETE FROM utilisateurs;

-- ✅ Correct : Supprime seulement les inactifs
DELETE FROM utilisateurs WHERE statut = 'inactif';
```

### 4. Les performances

Certaines opérations DML peuvent être coûteuses :

| Opération | Volume | Impact |
|-----------|--------|--------|
| INSERT d'une ligne | Léger | ~1ms |
| INSERT de 1000 lignes une par une | Moyen | ~1 seconde |
| INSERT de 1000 lignes en batch | Léger | ~10ms |
| UPDATE d'une ligne (avec index) | Léger | ~1ms |
| UPDATE de 1 million de lignes | Lourd | Minutes |
| DELETE avec WHERE (indexé) | Léger | ~1ms |
| DELETE sans WHERE (table complète) | Très lourd | Minutes/Heures |

Nous verrons comment optimiser ces opérations.

---

## Spécificités PostgreSQL pour le DML

PostgreSQL offre des fonctionnalités avancées qui distinguent sa gestion du DML.

### 1. La clause RETURNING

**Unique à PostgreSQL** (et quelques autres SGBD), RETURNING permet de récupérer les valeurs des lignes insérées, modifiées ou supprimées :

```sql
-- Récupérer l'ID généré
INSERT INTO utilisateurs (nom, email)
VALUES ('Alice', 'alice@example.com')
RETURNING id, nom, email;

-- Voir les valeurs modifiées
UPDATE produits SET prix = prix * 1.1
WHERE categorie = 'Premium'
RETURNING id, nom, prix;

-- Archiver en supprimant
DELETE FROM sessions WHERE date_expiration < NOW()
RETURNING user_id, date_creation;
```

Cette fonctionnalité sera détaillée dans la section 6.4.

### 2. UPSERT : ON CONFLICT

PostgreSQL permet d'insérer une ligne, et si elle existe déjà (conflit), de la mettre à jour ou l'ignorer :

```sql
INSERT INTO statistiques (page, vues)
VALUES ('accueil', 1)
ON CONFLICT (page)
DO UPDATE SET vues = statistiques.vues + 1;
```

Une seule commande remplace :
```sql
-- Traditionnel (2+ requêtes)
SELECT ... FROM statistiques WHERE page = 'accueil';
IF found THEN
    UPDATE ...
ELSE
    INSERT ...
END IF;
```

Nous verrons cela en détail dans la section 6.7.

### 3. COPY : Import/Export massif

Pour des volumes importants, PostgreSQL offre `COPY`, bien plus rapide que des INSERT multiples :

```sql
-- Importer 1 million de lignes en quelques secondes
COPY employes FROM '/data/employes.csv' WITH (FORMAT csv, HEADER true);
```

Détaillé dans les sections 6.1 et 6.2.

### 4. MERGE (PostgreSQL 15+)

Synchronisation de tables avec INSERT, UPDATE et DELETE combinés :

```sql
MERGE INTO produits_cible AS target
USING produits_source AS source
ON target.sku = source.sku
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

Nous l'explorerons dans la section 6.8.

### 5. Support OLD et NEW dans RETURNING (PostgreSQL 18)

Nouveauté majeure de PostgreSQL 18 :

```sql
UPDATE employes
SET salaire = salaire * 1.10
WHERE departement = 'IT'
RETURNING
    id,
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire,
    NEW.salaire - OLD.salaire AS augmentation;
```

Détaillé dans la section 6.5.

---

## Organisation de ce chapitre

Ce chapitre est organisé de manière progressive, du simple au complexe :

### Section 6.1 : INSERT - Ajouter des données
- Insertion simple et multiple
- Import massif avec COPY
- Gestion des valeurs par défaut et séquences

### Section 6.2 : Nouveautés PostgreSQL 18 - COPY
- Amélioration de la gestion du marqueur `\.` en CSV
- Performances accrues
- Meilleure robustesse

### Section 6.3 : UPDATE et DELETE - Modifier et supprimer
- Syntaxe et précautions critiques
- Importance de la clause WHERE
- Gestion des transactions

### Section 6.4 : RETURNING - Récupérer les données modifiées
- Utilisation avec INSERT, UPDATE, DELETE
- Patterns avancés
- Cas d'usage pratiques

### Section 6.5 : OLD et NEW dans RETURNING (PostgreSQL 18)
- Comparer avant/après
- Audit automatique
- Validation des modifications

### Section 6.6 : TRUNCATE vs DELETE
- Différences fondamentales
- Comparaison de performances
- Quand utiliser quoi

### Section 6.7 : UPSERT avec ON CONFLICT
- INSERT ou UPDATE en une seule commande
- DO NOTHING vs DO UPDATE
- Cas d'usage (compteurs, cache, synchronisation)

### Section 6.8 : MERGE - Consolidation de données
- Synchronisation de tables
- INSERT + UPDATE + DELETE combinés
- OLD et NEW avec MERGE (PostgreSQL 18)

---

## Bonnes pratiques générales DML

Avant de commencer, voici quelques règles d'or à toujours garder en tête.

### 1. Toujours tester avec SELECT d'abord

```sql
-- ❌ Dangereux : Modifier directement
UPDATE clients SET statut = 'inactif' WHERE derniere_visite < '2024-01-01';

-- ✅ Sécurisé : Vérifier d'abord
SELECT id, nom, derniere_visite
FROM clients
WHERE derniere_visite < '2024-01-01';
-- Vérifier les résultats, puis UPDATE
```

### 2. Utiliser des transactions pour les modifications critiques

```sql
BEGIN;

-- Opérations DML
UPDATE comptes SET solde = solde - 100 WHERE id = 1;
UPDATE comptes SET solde = solde + 100 WHERE id = 2;
INSERT INTO operations (type, montant) VALUES ('transfert', 100);

-- Vérifier que tout est OK
SELECT * FROM comptes WHERE id IN (1, 2);

-- Si OK : COMMIT, sinon : ROLLBACK
COMMIT;
```

### 3. Privilégier les opérations par lots

```sql
-- ❌ Lent : 1000 requêtes
FOR i IN 1..1000 LOOP
    INSERT INTO logs (message) VALUES ('Log ' || i);
END LOOP;

-- ✅ Rapide : 1 requête
INSERT INTO logs (message)
SELECT 'Log ' || generate_series(1, 1000);
```

### 4. Faire attention aux verrous

```sql
-- ⚠️ Peut bloquer d'autres transactions
BEGIN;
UPDATE produits SET prix = prix * 1.1;  -- Verrouille toute la table
-- ... longue opération ...
COMMIT;

-- ✅ Mieux : Traiter par lots
UPDATE produits SET prix = prix * 1.1 WHERE id BETWEEN 1 AND 1000;
-- Pause
UPDATE produits SET prix = prix * 1.1 WHERE id BETWEEN 1001 AND 2000;
-- Etc.
```

### 5. Documenter les modifications en production

```sql
-- ❌ Pas de trace
UPDATE employes SET departement_id = 5;

-- ✅ Avec documentation
-- MAINTENANCE 2025-11-19 : Réorganisation départements
-- Responsable : Jean Dupont
-- Ticket : JIRA-1234
BEGIN;
UPDATE employes SET departement_id = 5 WHERE departement_id = 3;
-- Vérification
SELECT COUNT(*) FROM employes WHERE departement_id = 5;
COMMIT;
```

---

## Checklist de sécurité DML

Avant toute opération DML en production, vérifiez :

- [ ] **Ai-je testé sur un environnement de développement/staging ?**
- [ ] **Ai-je vérifié avec SELECT ce qui sera affecté ?**
- [ ] **Ma clause WHERE est-elle correcte ?**
- [ ] **Suis-je dans une transaction (BEGIN) si nécessaire ?**
- [ ] **Ai-je une sauvegarde récente ?**
- [ ] **Les contraintes d'intégrité seront-elles respectées ?**
- [ ] **L'impact sur les performances est-il acceptable ?**
- [ ] **Les autres utilisateurs/applications sont-ils informés ?**
- [ ] **Ai-je un plan de rollback en cas de problème ?**
- [ ] **L'opération est-elle documentée (qui, quand, pourquoi) ?**

---

## Terminologie importante

Avant de continuer, assurez-vous de comprendre ces termes :

| Terme | Définition |
|-------|------------|
| **Ligne (Row/Tuple)** | Un enregistrement dans une table |
| **Colonne (Column)** | Un attribut/champ d'une table |
| **Clé primaire (Primary Key)** | Identifiant unique d'une ligne |
| **Clé étrangère (Foreign Key)** | Référence vers une autre table |
| **Contrainte (Constraint)** | Règle de validation des données |
| **Transaction** | Groupe d'opérations atomiques |
| **COMMIT** | Validation d'une transaction |
| **ROLLBACK** | Annulation d'une transaction |
| **Verrou (Lock)** | Blocage temporaire d'une ressource |
| **MVCC** | Multi-Version Concurrency Control |
| **WAL** | Write-Ahead Log (journal de transactions) |

---

## Différences avec d'autres SGBD

Si vous venez d'un autre système de gestion de bases de données, voici les principales différences :

### PostgreSQL vs MySQL

| Aspect | PostgreSQL | MySQL |
|--------|-----------|-------|
| **RETURNING** | ✅ Supporté | ❌ Non (utiliser LAST_INSERT_ID()) |
| **UPSERT** | `ON CONFLICT` | `ON DUPLICATE KEY UPDATE` |
| **Transactions** | Toujours activées | InnoDB : Oui, MyISAM : Non |
| **DELETE sans WHERE** | Autorisé mais dangereux | Idem |
| **TRUNCATE** | Très rapide | Très rapide |

### PostgreSQL vs SQL Server

| Aspect | PostgreSQL | SQL Server |
|--------|-----------|-----------|
| **RETURNING** | `RETURNING` | `OUTPUT` |
| **MERGE** | Depuis PG 15 | Depuis SQL Server 2008 |
| **Séquences** | `SERIAL` / `IDENTITY` | `IDENTITY` |
| **Transactions** | Explicites ou implicites | Implicites par défaut |

### PostgreSQL vs Oracle

| Aspect | PostgreSQL | Oracle |
|--------|-----------|--------|
| **RETURNING** | `RETURNING` | `RETURNING INTO` (PL/SQL) |
| **MERGE** | Depuis PG 15 | Depuis Oracle 9i |
| **Séquences** | `SERIAL` / Sequences | Sequences |
| **Transactions** | Explicites | Explicites |

---

## Ce que vous allez apprendre

À la fin de ce chapitre, vous serez capable de :

- ✅ **Insérer** des données efficacement (simple, multiple, massif)
- ✅ **Modifier** des données en toute sécurité avec UPDATE
- ✅ **Supprimer** des données de manière contrôlée
- ✅ **Utiliser RETURNING** pour récupérer les valeurs modifiées
- ✅ **Comparer** les valeurs OLD et NEW (PostgreSQL 18)
- ✅ **Choisir** entre DELETE et TRUNCATE selon le contexte
- ✅ **Maîtriser UPSERT** avec ON CONFLICT
- ✅ **Synchroniser** des tables avec MERGE
- ✅ **Optimiser** les performances des opérations DML
- ✅ **Éviter** les erreurs courantes et dangereuses
- ✅ **Gérer** les transactions pour garantir l'intégrité

---

## Conventions utilisées dans ce chapitre

Pour faciliter la lecture, nous utilisons les conventions suivantes :

### Annotations des exemples

```sql
-- ✅ Bonne pratique : À faire
SELECT * FROM users WHERE id = 42;

-- ⚠️ Attention : Potentiellement dangereux
UPDATE users SET active = false;

-- ❌ Mauvaise pratique : À éviter
DELETE FROM users;  -- Sans WHERE !
```

### Icônes de mise en garde

> **💡 Astuce** : Conseil pratique pour améliorer votre code

> **⚠️ Attention** : Point important nécessitant vigilance

> **❌ Erreur courante** : Piège fréquent à éviter

> **🚀 Performance** : Optimisation pour meilleures performances

> **🆕 PostgreSQL 18** : Nouveauté de la version 18

### Code SQL

Les exemples sont complets et testables :

```sql
-- Création de table exemple
CREATE TABLE exemple (
    id SERIAL PRIMARY KEY,
    valeur TEXT
);

-- Opération DML
INSERT INTO exemple (valeur) VALUES ('test');

-- Vérification
SELECT * FROM exemple;
```

---

## Prérequis

Avant de commencer ce chapitre, vous devriez :

1. **Connaître les bases du SQL** : SELECT simple, types de données basiques
2. **Avoir créé des tables** : Comprendre CREATE TABLE
3. **Comprendre les contraintes** : PRIMARY KEY, UNIQUE, NOT NULL
4. **Avoir accès à PostgreSQL** : Version 13+ recommandée, 18 pour les dernières fonctionnalités

Si vous n'êtes pas à l'aise avec ces prérequis, nous vous recommandons de réviser les chapitres précédents.

---

## Progression recommandée

Nous recommandons de suivre les sections dans l'ordre, car elles sont conçues de manière progressive :

```
6.1 INSERT           → Fondamental
     ↓
6.2 COPY (PG 18)     → Import massif
     ↓
6.3 UPDATE/DELETE    → Modifications
     ↓
6.4 RETURNING        → Récupération avancée
     ↓
6.5 OLD/NEW (PG 18)  → Comparaisons
     ↓
6.6 TRUNCATE         → Alternative à DELETE
     ↓
6.7 ON CONFLICT      → UPSERT
     ↓
6.8 MERGE            → Synchronisation complète
```

Chaque section s'appuie sur les connaissances des sections précédentes.

---

## Ressources additionnelles

Pour approfondir vos connaissances :

**Documentation officielle PostgreSQL** :
- [INSERT](https://www.postgresql.org/docs/current/sql-insert.html)
- [UPDATE](https://www.postgresql.org/docs/current/sql-update.html)
- [DELETE](https://www.postgresql.org/docs/current/sql-delete.html)
- [MERGE](https://www.postgresql.org/docs/current/sql-merge.html)

**Outils utiles** :
- **psql** : Interface en ligne de commande
- **pgAdmin** : Interface graphique
- **DBeaver** : Client SQL universel

---

## Conclusion de l'introduction

Le DML est le cœur battant de toute application utilisant une base de données. Maîtriser ces commandes est essentiel pour :

- **Développer** des applications robustes et performantes
- **Maintenir** l'intégrité des données
- **Optimiser** les performances
- **Éviter** les erreurs catastrophiques

PostgreSQL offre des fonctionnalités DML parmi les plus avancées du marché, avec des innovations continues (comme OLD/NEW dans PostgreSQL 18). Ce chapitre vous donnera toutes les clés pour les maîtriser.

> **Rappel important** : Avec un grand pouvoir vient une grande responsabilité. Les commandes DML, en particulier UPDATE et DELETE, peuvent avoir des conséquences irréversibles. Soyez toujours vigilant, testez sur des environnements de développement, et utilisez les transactions.

Prêt à commencer ? Direction la section 6.1 pour apprendre à insérer des données !

---


⏭️ [INSERT : Insertion simple, multiple, et import via COPY](/06-manipulation-des-donnees/01-insert-et-copy.md)
