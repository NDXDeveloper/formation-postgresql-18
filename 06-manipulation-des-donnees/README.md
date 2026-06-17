🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6. Manipulation des Données (DML)

## Introduction générale

Après avoir appris à **créer** des structures de données (tables, colonnes, contraintes) avec le DDL (Data Definition Language), il est temps d'apprendre à **manipuler** les données elles-mêmes. C'est le rôle du **DML** : Data Manipulation Language.

Le DML regroupe l'ensemble des commandes SQL qui permettent de :
- **Insérer** de nouvelles données dans les tables (`INSERT`, `COPY`)  
- **Modifier** des données existantes (`UPDATE`)  
- **Supprimer** des données (`DELETE`, `TRUNCATE`)  
- **Combiner** insertion et mise à jour conditionnelles (`INSERT … ON CONFLICT`, `MERGE`)

Ces opérations constituent le cœur de l'utilisation quotidienne d'une base de données. Comprendre le DML est essentiel pour tout développeur ou administrateur de bases de données.

> 📌 **Note sur SELECT** : la commande `SELECT` est traitée à part dans le **chapitre 5 (DQL — Data Query Language)** car elle **n'écrit pas** de données. Certains auteurs l'incluent dans le DML « au sens large » (puisqu'elle manipule les données en lecture) ; le standard SQL et la plupart des SGBD la classent dans le DQL. Dans cette formation, nous suivons cette séparation : **DQL pour lire**, **DML pour modifier**.

---

## Qu'est-ce que le DML ?

### Définition

Le **DML (Data Manipulation Language)** est un sous-ensemble du langage SQL dédié à la manipulation des données contenues dans les tables d'une base de données. Contrairement au DDL qui modifie la **structure** des objets (CREATE, ALTER, DROP), le DML modifie le **contenu** des tables.

### Les commandes du CRUD

Dans le monde du développement, on parle souvent du **CRUD** (Create, Read, Update, Delete). Ce chapitre couvre les opérations qui **écrivent** dans la base ; la lecture (`SELECT`) a été vue au chapitre 5 :

| Commande SQL | Opération CRUD | Description | Exemple d'usage | Chapitre |
|--------------|---------------|-------------|-----------------|----------|
| `INSERT` / `COPY` | **C**reate | Ajoute de nouvelles lignes | Enregistrer un nouvel utilisateur | **6.1 / 6.2** |
| `SELECT` | **R**ead | Lit et récupère des données | Afficher la liste des produits | 5 (DQL) |
| `UPDATE` | **U**pdate | Modifie des lignes existantes | Changer le prix d'un produit | **6.3** |
| `DELETE` / `TRUNCATE` | **D**elete | Supprime des lignes | Supprimer un compte utilisateur | **6.3 / 6.6** |
| `INSERT … ON CONFLICT` | **C**reate ou **U**pdate | UPSERT atomique | Compteur de vues d'une page | **6.7** |
| `MERGE` | **C**+ **U** + **D** | Synchronisation conditionnelle | Réconcilier deux tables | **6.8** |

### Analogie avec un classeur

Imaginez une base de données comme un classeur de fiches :

- **DDL (`CREATE TABLE`)** : créer un nouveau classeur avec des intercalaires et des colonnes prédéfinies  
- **DML `INSERT`** : ajouter de nouvelles fiches dans le classeur  
- **DQL `SELECT`** : consulter les fiches pour lire les informations (voir chapitre 5)  
- **DML `UPDATE`** : corriger ou mettre à jour une fiche existante  
- **DML `DELETE`** : retirer une fiche du classeur  
- **DML `TRUNCATE`** : vider intégralement le classeur d'un coup

---

## Pourquoi le DML est-il crucial ?

### 1. Le cycle de vie des données

Une application moderne suit généralement ce cycle :

```
┌─────────────┐
│   CREATE    │  DDL : créer la structure (chapitre 4)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   INSERT    │  DML : ajouter des données (6.1, 6.2)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   SELECT    │  DQL : lire les données (chapitre 5)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   UPDATE    │  DML : modifier les données (6.3)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   DELETE    │  DML : supprimer des données (6.3, 6.6)
└─────────────┘
```

Combinées, **lecture (DQL)** et **écriture (DML)** représentent la quasi-totalité des opérations effectuées par une application en production.

### 2. L'interaction quotidienne avec les données

Dans une application web typique, les opérations d'**écriture (DML)** sont minoritaires en volume mais critiques en impact. Voici un ordre de grandeur courant :

| Opération | Part typique | Exemple |
|-----------|--------------|---------|
| `SELECT` (lecture / DQL) | 80–90 % | afficher une page produit, charger un profil utilisateur |
| `INSERT` | 5–10 % | créer un compte, passer une commande |
| `UPDATE` | 3–7 % | mettre à jour un panier, modifier un profil |
| `DELETE` | 1–3 % | supprimer un commentaire, vider le panier |

> 📌 Ces chiffres sont indicatifs : une application analytique peut atteindre 99 % de lectures, tandis qu'un système d'**ingestion** (logs, IoT, ETL) peut au contraire être dominé par les `INSERT` et les `COPY`.

### 3. La base de toute application

Quel que soit le type d'application, le DML est omniprésent. Voici les opérations d'**écriture** typiques dans trois contextes courants :

**E-commerce** :
- `INSERT` : enregistrer une nouvelle commande
- `UPDATE` : mettre à jour le stock après un achat
- `DELETE` : supprimer un article du panier
- `INSERT … ON CONFLICT` : décrémenter un stock partagé sans risquer le doublon

**Réseau social** :
- `INSERT` : publier un nouveau post
- `UPDATE` : modifier un commentaire
- `DELETE` : supprimer une photo

**Gestion d'entreprise** :
- `INSERT` / `COPY` : enregistrer une nouvelle facture, importer en masse depuis l'ERP
- `UPDATE` : corriger une erreur de saisie
- `DELETE` / `TRUNCATE` : archiver des données obsolètes, purger une table de *staging*
- `MERGE` : synchroniser une table miroir avec une source externe

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
- **A**tomicité : tout ou rien  
- **C**ohérence : respect des contraintes  
- **I**solation : les transactions ne s'interfèrent pas  
- **D**urabilité : les données *committées* sont permanentes (garantie via le WAL)

#### Autocommit : la transaction implicite

Hors d'un `BEGIN`, **chaque commande SQL est sa propre transaction implicite** : elle est *committée* automatiquement si elle réussit, ou *rollback* si elle échoue. C'est le mode *autocommit*. Beaucoup de bugs en production viennent de cette confusion :

```sql
-- Hors BEGIN : autocommit. Le DELETE est INSTANTANÉMENT committé.
DELETE FROM comptes WHERE actif = false;
-- Pas de "annulation" possible. Il faut restaurer depuis sauvegarde.

-- Dans BEGIN : transaction explicite. Tant que vous n'avez pas COMMIT,
-- vous pouvez ROLLBACK.
BEGIN;  
DELETE FROM comptes WHERE actif = false;  
-- Vérifier ce qui a été supprimé...
SELECT count(*) FROM comptes WHERE actif = false;  -- 0 (mais visible par vous seul)  
ROLLBACK;  -- Tout est restauré  
```

> 💡 **Réflexe à acquérir** : pour toute opération DML potentiellement destructrice, ouvrez un `BEGIN` avant et un `ROLLBACK` après pour vérifier le périmètre avant de relancer la commande définitivement avec `COMMIT`.

#### Points de sauvegarde (`SAVEPOINT`)

À l'intérieur d'une transaction, on peut poser des **points de sauvegarde** pour annuler partiellement sans tout perdre :

```sql
BEGIN;  
INSERT INTO commandes (client_id, montant) VALUES (42, 100);  

SAVEPOINT avant_lignes;  
INSERT INTO lignes_commande (commande_id, produit_id) VALUES (currval('commandes_id_seq'), 999);  
-- Oups, produit_id 999 n'existe pas !
ROLLBACK TO SAVEPOINT avant_lignes;
-- L'INSERT de la commande est conservé, seul l'INSERT des lignes est annulé.

-- On peut continuer
INSERT INTO lignes_commande (commande_id, produit_id) VALUES (currval('commandes_id_seq'), 1);  
COMMIT;  
```

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

### 4. MVCC et conséquences sur le DML

PostgreSQL implémente l'isolation via **MVCC** (*Multi-Version Concurrency Control*). Comprendre ce mécanisme est essentiel pour ne pas être surpris par le comportement du DML.

#### Principe

Chaque ligne stockée porte deux *timestamps logiques* invisibles, basés sur les **identifiants de transaction** (`xid`) :
- `xmin` : la transaction qui a **créé** cette version
- `xmax` : la transaction qui a **supprimé** cette version (`0` si encore vivante)

Quand vous faites un `UPDATE` ou un `DELETE`, PostgreSQL ne modifie **pas** la ligne en place : il marque l'ancienne version comme « morte pour les futures transactions » (en posant son `xmax`), et — pour un `UPDATE` — écrit une **nouvelle version**. Les anciennes versions restent physiquement présentes tant qu'une transaction ouverte au moment de leur création peut encore les voir.

```sql
-- Voir les pseudo-colonnes système :
SELECT ctid, xmin, xmax, * FROM comptes WHERE id = 1;
--  ctid  | xmin |  xmax  | id | nom   | solde
-- -------+------+--------+----+-------+-------
--  (0,1) |  742 |      0 |  1 | Alice |  1000
```

#### Conséquences pratiques

- **Les lecteurs ne bloquent pas les écrivains** (et inversement) : un `SELECT` ne pose aucun verrou sur les données qu'il lit. Une transaction qui modifie ces données peut progresser en parallèle.
- **Un `UPDATE` est en réalité un `DELETE` + `INSERT` logique** : il génère du *bloat* (espace occupé par d'anciennes versions) jusqu'à ce que `VACUUM` (manuel ou via `autovacuum`) le récupère.
- **Une transaction longue empêche le nettoyage** : tant qu'une transaction ouverte pourrait voir une ancienne version, celle-ci ne peut pas être supprimée. Une transaction oubliée pendant une heure peut faire enfler une table très volatile.

> 💡 **À retenir** : surveillez les transactions longues (`SELECT pid, state, xact_start, query FROM pg_stat_activity WHERE state <> 'idle'`) et configurez `autovacuum` pour qu'il suive le rythme d'écriture de vos tables.

### 5. Niveaux d'isolation des transactions

PostgreSQL propose **trois niveaux d'isolation** standard SQL (les autres sont automatiquement *promus*) :

| Niveau (`SET TRANSACTION ISOLATION LEVEL …`) | Anomalies évitées | Coût |
|----------------------------------------------|-------------------|------|
| `READ COMMITTED` (défaut) | Lectures sales | Très faible |
| `REPEATABLE READ` | + Lectures non-répétables, + lectures fantômes (en pratique grâce au snapshot stable) | Faible à modéré |
| `SERIALIZABLE` | + Anomalies de sérialisation | Modéré ; peut produire des `serialization_failure` à *rejouer* côté applicatif |

En pratique, **`READ COMMITTED` est le bon défaut** pour la plupart des applications transactionnelles. On bascule en `REPEATABLE READ` ou `SERIALIZABLE` quand on veut une cohérence stricte multi-instructions (rapports financiers, *snapshot* d'un agrégat à un instant T).

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- À l'intérieur, tous les SELECT verront le même snapshot, même si
-- d'autres transactions COMMITent entre-temps.
SELECT sum(solde) FROM comptes;
-- ... autres requêtes ...
COMMIT;
```

### 6. Les performances en un coup d'œil

Les coûts ci-dessous sont des **ordres de grandeur** pour fixer les idées — vos chiffres réels dépendront du matériel (SSD/NVMe, RAM, CPU), du nombre d'index, des triggers, et du *fillfactor* de vos tables. Mesurez sur votre propre infrastructure.

| Opération | Coût relatif | Notes |
|-----------|--------------|-------|
| `INSERT` d'**1 ligne** | très faible | Aller-retour réseau souvent dominant |
| 1000 `INSERT` séparés | élevé | N planifications + N allers-retours |
| `INSERT` multi-valeurs (1000 lignes en une requête) | faible | À privilégier |
| `COPY` (CSV ou binaire) | très faible | À privilégier au-delà de quelques milliers de lignes |
| `UPDATE` ciblé par index | très faible | Encore plus rapide si **HOT update** (cf. 6.3) |
| `UPDATE` massif (millions de lignes) | élevé | Génère beaucoup de WAL + dead tuples ; faire **par lots** |
| `DELETE` ciblé par index | très faible | |
| `DELETE` de toute la table | très élevé | Préférer `TRUNCATE` |
| `TRUNCATE` | constant (indépendant du volume) | Verrou exclusif, libère le fichier |

Nous verrons comment optimiser ces opérations dans chaque section.

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

> 🆕 **Attention aux versions** : `MERGE` existe depuis **PG 15**, mais `WHEN NOT MATCHED BY SOURCE` (utilisé ci-dessus) et le support de `RETURNING` avec `MERGE` ne sont arrivés qu'en **PostgreSQL 17**.

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

- [ ] **Ai-je testé sur un environnement de développement / *staging* ?**  
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
| **Ligne (Row)** | Un enregistrement dans une table |
| **Tuple** | Dans PostgreSQL, une **version physique** d'une ligne sur disque. Avec MVCC, une même ligne logique peut exister en plusieurs tuples (versions) tant que d'anciennes transactions y voient encore l'état précédent. |
| **Colonne (Column)** | Un attribut/champ d'une table |
| **Clé primaire (Primary Key)** | Identifiant unique d'une ligne |
| **Clé étrangère (Foreign Key)** | Référence vers une autre table |
| **Contrainte (Constraint)** | Règle de validation des données |
| **Transaction** | Groupe d'opérations atomiques |
| **`COMMIT`** | Validation d'une transaction |
| **`ROLLBACK`** | Annulation d'une transaction |
| **Verrou (Lock)** | Blocage temporaire d'une ressource |
| **MVCC** | *Multi-Version Concurrency Control* — chaque écriture crée une nouvelle version de ligne ; les lecteurs ne bloquent pas les écrivains et inversement |
| **WAL** | *Write-Ahead Log* — journal de transactions garantissant la durabilité (D d'ACID) et la reprise après crash |
| **HOT update** | *Heap-Only Tuple update* — UPDATE optimisé qui réutilise la même page disque quand aucun index n'est touché (voir chapitre 6.3) |

---

## Différences avec d'autres SGBD

Si vous venez d'un autre système de gestion de bases de données, voici les principales différences :

### PostgreSQL vs MySQL / MariaDB

| Aspect | PostgreSQL | MySQL / MariaDB |
|--------|-----------|-----------------|
| **`RETURNING`** | ✅ Supporté (INSERT/UPDATE/DELETE/MERGE) | ❌ MySQL : non — utiliser `LAST_INSERT_ID()`<br>⚠️ MariaDB : `DELETE … RETURNING` (10.0), `INSERT`/`REPLACE … RETURNING` (10.5) ; **pas** `UPDATE … RETURNING` (hors preview 13.0) |
| **UPSERT** | `INSERT … ON CONFLICT` (standard) | `INSERT … ON DUPLICATE KEY UPDATE` (non standard) |
| **Transactions** | Toujours actives, partout | InnoDB : oui ; MyISAM (déprécié) : non |
| **`DELETE` sans `WHERE`** | Autorisé mais dangereux | Idem |
| **`TRUNCATE`** | Très rapide, **transactionnel** (`ROLLBACK` possible) | Très rapide, **non transactionnel** sur InnoDB : commit implicite |

### PostgreSQL vs SQL Server

| Aspect | PostgreSQL | SQL Server |
|--------|-----------|-----------|
| **RETURNING** | `RETURNING` (extension PostgreSQL, **pas** dans le standard SQL) | `OUTPUT` (avec pseudo-tables `INSERTED` / `DELETED`) |
| **`MERGE`** | Depuis PG 15 | Depuis SQL Server 2008 |
| **Séquences / auto-incr.** | `GENERATED … AS IDENTITY` (standard, recommandé) ou `SERIAL` (legacy) | `IDENTITY` (non standard) ou `SEQUENCE` (depuis 2012) |
| **Transactions** | Explicites (mode *autocommit* par défaut) | *Autocommit* par défaut, sauf `SET IMPLICIT_TRANSACTIONS ON` |

### PostgreSQL vs Oracle

| Aspect | PostgreSQL | Oracle |
|--------|-----------|--------|
| **RETURNING** | `RETURNING` directement dans SQL | `RETURNING … INTO` (uniquement en PL/SQL) |
| **`MERGE`** | Depuis PG 15 | Depuis Oracle 9i |
| **Séquences / auto-incr.** | `GENERATED … AS IDENTITY` ou `SEQUENCE` | `SEQUENCE` (historique) ou `GENERATED AS IDENTITY` (12c+) |
| **Transactions** | Mode *autocommit* par défaut (chaque instruction commit) ; `BEGIN` ouvre explicitement une transaction | Pas d'*autocommit* : toute instruction démarre une transaction implicite jusqu'au `COMMIT`/`ROLLBACK` |

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
