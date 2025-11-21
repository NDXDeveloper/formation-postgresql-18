🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Audit de Schéma PostgreSQL
## Guide

---

## Table des Matières
1. [Introduction](#introduction)
2. [Pourquoi Auditer son Schéma ?](#pourquoi-auditer-son-sch%C3%A9ma-)
3. [Les Piliers d'un Audit de Schéma](#les-piliers-dun-audit-de-sch%C3%A9ma)
4. [Audit des Tables](#audit-des-tables)
5. [Audit des Contraintes](#audit-des-contraintes)
6. [Audit des Index](#audit-des-index)
7. [Audit des Types de Données](#audit-des-types-de-donn%C3%A9es)
8. [Audit de la Normalisation](#audit-de-la-normalisation)
9. [Audit de Sécurité](#audit-de-s%C3%A9curit%C3%A9)
10. [Audit de Performance](#audit-de-performance)
11. [Checklist Complète](#checklist-compl%C3%A8te)
12. [Conclusion](#conclusion)

---

## Introduction

Un **audit de schéma** est une analyse systématique de la structure de votre base de données PostgreSQL. Il permet d'identifier les problèmes de conception, les inefficacités et les risques potentiels avant qu'ils ne deviennent critiques en production.

### Qu'est-ce qu'un Schéma ?

Dans PostgreSQL, un schéma est un **espace de noms** qui contient des objets de base de données comme des tables, des vues, des index et des fonctions. Le schéma définit la structure logique et organisationnelle de vos données.

### À qui s'adresse cet Audit ?

- Développeurs débutants qui créent leur première base de données
- Équipes DevOps responsables de la maintenance
- Architectes qui veulent valider une conception
- DBAs qui prennent en charge une base existante

---

## Pourquoi Auditer son Schéma ?

### Les Enjeux d'un Schéma Bien Conçu

Un schéma mal conçu peut entraîner :

**Performance Dégradée**
- Requêtes lentes qui prennent plusieurs secondes au lieu de millisecondes
- Consommation excessive de mémoire et CPU
- Saturation des disques durs

**Problèmes de Maintenance**
- Difficultés à faire évoluer la structure
- Migrations complexes et risquées
- Code applicatif difficile à maintenir

**Risques de Sécurité**
- Failles d'accès non autorisé
- Fuites de données sensibles
- Violations de conformité (RGPD, etc.)

**Intégrité des Données Compromise**
- Données incohérentes ou corrompues
- Doublons non contrôlés
- Relations orphelines

### Quand Effectuer un Audit ?

- **Avant la mise en production** : Valider la conception initiale
- **Périodiquement en production** : Tous les 3-6 mois pour détecter la dérive
- **Après une migration** : Vérifier que tout est correct
- **En cas de problèmes de performance** : Identifier la cause racine
- **Lors de la reprise d'un projet existant** : Comprendre l'état actuel

---

## Les Piliers d'un Audit de Schéma

Un audit complet examine sept dimensions principales :

### 1. Structure des Tables
La façon dont vos tables sont organisées et définies.

### 2. Contraintes d'Intégrité
Les règles qui garantissent la cohérence des données.

### 3. Stratégie d'Indexation
Comment vos données sont indexées pour les recherches rapides.

### 4. Choix des Types de Données
L'adéquation entre les données stockées et leur type.

### 5. Niveau de Normalisation
L'équilibre entre normalisation et dénormalisation.

### 6. Sécurité et Permissions
Qui peut accéder à quoi et comment.

### 7. Optimisation pour la Performance
Les aspects qui impactent la vitesse d'exécution.

---

## Audit des Tables

### 1. Présence de Clés Primaires

**Qu'est-ce qu'une Clé Primaire ?**

Une clé primaire (PRIMARY KEY ou PK) identifie de manière **unique** chaque ligne d'une table. C'est l'identifiant fondamental de vos enregistrements.

**Points à Vérifier**

✅ **Toutes les tables doivent avoir une clé primaire**
```sql
-- Exemple de table AVEC clé primaire (BON)
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    nom VARCHAR(100)
);
```

❌ **Table sans clé primaire (MAUVAIS)**
```sql
CREATE TABLE logs (
    timestamp TIMESTAMP,
    message TEXT,
    level VARCHAR(10)
);
-- Risque : doublons possibles, pas d'identifiant unique
```

**Problèmes Causés par l'Absence de PK**

- Impossible de cibler précisément une ligne pour UPDATE/DELETE
- Difficultés avec les relations (foreign keys)
- Problèmes de réplication
- Performances dégradées sur les jointures

**Types de Clés Primaires**

1. **Entier Auto-incrémenté (SERIAL/BIGSERIAL)**
   - Simple et performant
   - Recommandé pour la plupart des cas

2. **UUID/UUIDv7**
   - Utile pour les systèmes distribués
   - Évite les collisions entre bases
   - PostgreSQL 18 introduit UUIDv7 (avec timestamp)

3. **Clé Composite (Multiple colonnes)**
   - Pour des relations many-to-many
   - Exemple : (user_id, product_id) dans une table de commandes

4. **Clé Naturelle**
   - Utilise une donnée métier (email, numéro de sécurité sociale)
   - À éviter généralement (les données métier changent)

### 2. Conventions de Nommage

**Pourquoi c'est Important ?**

Des noms cohérents facilitent :
- La compréhension du schéma
- La maintenance du code
- Le travail en équipe

**Bonnes Pratiques**

✅ **Noms Clairs et Explicites**
```sql
-- BON
CREATE TABLE commandes_clients (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT NOT NULL,
    date_commande TIMESTAMP NOT NULL,
    montant_total NUMERIC(10,2)
);

-- MAUVAIS
CREATE TABLE cmd (
    i BIGSERIAL PRIMARY KEY,
    c BIGINT NOT NULL,
    d TIMESTAMP NOT NULL,
    m NUMERIC(10,2)
);
```

✅ **Conventions Standards**

- **Tables** : Pluriel, snake_case → `utilisateurs`, `commandes_clients`
- **Colonnes** : Singulier, snake_case → `nom`, `date_creation`
- **Clés Primaires** : `id` ou `table_id` → `id`, `utilisateur_id`
- **Clés Étrangères** : `table_id` → `client_id`, `produit_id`
- **Index** : `idx_table_colonne` → `idx_commandes_client_id`
- **Contraintes** :
  - FK : `fk_table_colonne` → `fk_commandes_client_id`
  - Unique : `uk_table_colonne` → `uk_utilisateurs_email`
  - Check : `ck_table_condition` → `ck_commandes_montant_positif`

### 3. Colonnes de Métadonnées (Audit Trail)

**Qu'est-ce qu'un Audit Trail ?**

Ce sont des colonnes qui enregistrent automatiquement :
- Quand un enregistrement a été créé
- Quand il a été modifié
- Qui l'a créé/modifié (optionnel)

**Colonnes Recommandées**

```sql
CREATE TABLE articles (
    id BIGSERIAL PRIMARY KEY,
    titre VARCHAR(200) NOT NULL,
    contenu TEXT,

    -- Colonnes d'audit (RECOMMANDÉ)
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by BIGINT,  -- Optionnel : référence à utilisateurs.id
    updated_by BIGINT   -- Optionnel
);
```

**Avantages**

- Traçabilité complète des modifications
- Debugging facilité (savoir quand un bug est apparu)
- Conformité réglementaire (RGPD, audit financier)
- Analyses temporelles possibles

**Automatisation avec des Triggers**

PostgreSQL permet d'automatiser la mise à jour de `updated_at` :

```sql
-- Fonction trigger pour updated_at
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Appliquer le trigger
CREATE TRIGGER trg_articles_updated_at
    BEFORE UPDATE ON articles
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();
```

### 4. Soft Delete vs Hard Delete

**Qu'est-ce que le Soft Delete ?**

Au lieu de **supprimer physiquement** une ligne (hard delete), on la marque comme "supprimée" avec un flag.

**Approche Hard Delete (Suppression Physique)**

```sql
-- Suppression définitive
DELETE FROM utilisateurs WHERE id = 42;
-- La ligne n'existe plus dans la base
```

❌ **Problèmes** :
- Perte définitive des données
- Impossible de restaurer
- Problèmes si des relations existent
- Pas de traçabilité

**Approche Soft Delete (Suppression Logique)**

```sql
-- Ajout d'une colonne deleted_at
ALTER TABLE utilisateurs
ADD COLUMN deleted_at TIMESTAMP;

-- "Suppression" logique
UPDATE utilisateurs
SET deleted_at = NOW()
WHERE id = 42;

-- Requêtes sur données actives
SELECT * FROM utilisateurs
WHERE deleted_at IS NULL;
```

✅ **Avantages** :
- Récupération possible
- Audit trail complet
- Relations préservées
- Conformité légale (conservation des données)

**Quand Utiliser Quoi ?**

| Situation | Recommandation |
|-----------|----------------|
| Données métier critiques | Soft Delete |
| Données personnelles (RGPD) | Hard Delete après délai |
| Données temporaires (logs) | Hard Delete |
| Données financières | Soft Delete (légal) |

### 5. Colonnes NOT NULL

**Qu'est-ce que NOT NULL ?**

Une contrainte qui **interdit** les valeurs NULL dans une colonne.

**Principe de Base**

✅ **Colonnes obligatoires → NOT NULL**

```sql
CREATE TABLE produits (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,        -- Obligatoire
    description TEXT,                  -- Optionnel (NULL autorisé)
    prix NUMERIC(10,2) NOT NULL,      -- Obligatoire
    stock INTEGER NOT NULL DEFAULT 0  -- Obligatoire avec valeur par défaut
);
```

**Pourquoi c'est Important ?**

1. **Clarté de l'intention** : Indique explicitement les champs obligatoires
2. **Intégrité des données** : Évite les valeurs manquantes non prévues
3. **Performance** : PostgreSQL peut optimiser les requêtes
4. **Évite les bugs** : Moins de gestion de NULL dans le code applicatif

**Le Piège du NULL**

Le NULL en SQL n'est pas une valeur, c'est l'**absence de valeur**. Cela cause des comportements contre-intuitifs :

```sql
-- NULL n'est pas égal à NULL !
SELECT NULL = NULL;  -- Retourne NULL (pas TRUE)

-- La logique ternaire
SELECT * FROM produits WHERE prix <> 100;
-- N'inclut PAS les lignes où prix IS NULL
```

**Audit des NOT NULL**

Points à vérifier :
- Les clés étrangères doivent souvent être NOT NULL
- Les colonnes métier critiques doivent être NOT NULL
- Les dates importantes (created_at) doivent être NOT NULL
- Attention aux colonnes qui "devraient" être obligatoires mais ne le sont pas

---

## Audit des Contraintes

Les contraintes garantissent l'**intégrité référentielle** et la **cohérence des données**. Ce sont les règles métier encodées dans la base de données.

### 1. Clés Étrangères (Foreign Keys)

**Qu'est-ce qu'une Clé Étrangère ?**

Une clé étrangère (FK) établit une **relation** entre deux tables et garantit que la valeur existe dans la table référencée.

**Exemple Sans Foreign Key (MAUVAIS)**

```sql
CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT,  -- Aucune garantie que ce client existe !
    montant NUMERIC(10,2)
);

-- Problème : On peut insérer n'importe quel client_id
INSERT INTO commandes (client_id, montant)
VALUES (999999, 100.00);  -- client_id 999999 n'existe peut-être pas
```

**Exemple Avec Foreign Key (BON)**

```sql
CREATE TABLE clients (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT NOT NULL,
    montant NUMERIC(10,2) NOT NULL,

    -- Foreign Key
    CONSTRAINT fk_commandes_client
        FOREIGN KEY (client_id)
        REFERENCES clients(id)
);

-- PostgreSQL refuse maintenant les client_id invalides
INSERT INTO commandes (client_id, montant)
VALUES (999999, 100.00);
-- ERROR: insert or update on table "commandes" violates foreign key constraint
```

**Actions en Cascade**

Les Foreign Keys peuvent définir un comportement automatique :

```sql
-- ON DELETE CASCADE : Supprime les commandes si le client est supprimé
CONSTRAINT fk_commandes_client
    FOREIGN KEY (client_id)
    REFERENCES clients(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE;

-- ON DELETE SET NULL : Met client_id à NULL
ON DELETE SET NULL

-- ON DELETE RESTRICT : Interdit la suppression (par défaut)
ON DELETE RESTRICT

-- ON DELETE NO ACTION : Similaire à RESTRICT
ON DELETE NO ACTION
```

**Quand Utiliser Quoi ?**

| Action | Cas d'Usage |
|--------|-------------|
| CASCADE | Relations parents-enfants strictes (ex: facture → lignes_facture) |
| SET NULL | Relations optionnelles (ex: commande → commercial_responsable) |
| RESTRICT | Protection contre suppression accidentelle (par défaut) |
| NO ACTION | Vérification différée (dans une transaction) |

**Audit des Foreign Keys**

Points à vérifier :
- ✅ Toutes les colonnes `xxx_id` ont-elles une FK ?
- ✅ Les actions ON DELETE sont-elles appropriées ?
- ✅ Les FK sont-elles indexées ? (Performance !)
- ✅ Les FK sont-elles nommées explicitement ?

### 2. Contraintes UNIQUE

**Qu'est-ce qu'une Contrainte UNIQUE ?**

Garantit qu'aucune valeur ne se répète dans une colonne (ou combinaison de colonnes).

**Exemples d'Usage**

```sql
CREATE TABLE utilisateurs (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    username VARCHAR(50) NOT NULL,

    -- UNIQUE sur une colonne
    CONSTRAINT uk_utilisateurs_email UNIQUE (email),
    CONSTRAINT uk_utilisateurs_username UNIQUE (username)
);

-- Tentative de doublon refusée
INSERT INTO utilisateurs (email, username)
VALUES ('john@example.com', 'john');  -- OK

INSERT INTO utilisateurs (email, username)
VALUES ('john@example.com', 'john2'); -- ERROR: duplicate key value
```

**UNIQUE Composite (Multiple Colonnes)**

```sql
CREATE TABLE inscriptions_cours (
    id BIGSERIAL PRIMARY KEY,
    etudiant_id BIGINT NOT NULL,
    cours_id BIGINT NOT NULL,

    -- Un étudiant ne peut s'inscrire qu'une seule fois à un cours
    CONSTRAINT uk_inscriptions_etudiant_cours
        UNIQUE (etudiant_id, cours_id)
);
```

**UNIQUE vs PRIMARY KEY**

| Aspect | PRIMARY KEY | UNIQUE |
|--------|-------------|--------|
| Nombre par table | 1 seule | Plusieurs possibles |
| NULL autorisé | Non | Oui (mais un seul NULL) |
| Crée un index | Oui | Oui |
| Usage | Identifiant principal | Unicité métier |

**Audit des UNIQUE**

Points à vérifier :
- ✅ Email, username, numéro de série → doivent être UNIQUE
- ✅ Les combinaisons métier uniques sont-elles protégées ?
- ✅ Les contraintes UNIQUE sont-elles nommées explicitement ?

### 3. Contraintes CHECK

**Qu'est-ce qu'une Contrainte CHECK ?**

Permet de définir une **condition métier** que chaque ligne doit respecter.

**Exemples Pratiques**

```sql
CREATE TABLE produits (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,
    prix NUMERIC(10,2) NOT NULL,
    stock INTEGER NOT NULL,
    reduction_pct NUMERIC(3,2),

    -- Le prix doit être positif
    CONSTRAINT ck_produits_prix_positif
        CHECK (prix > 0),

    -- Le stock ne peut pas être négatif
    CONSTRAINT ck_produits_stock_positif
        CHECK (stock >= 0),

    -- La réduction entre 0% et 100%
    CONSTRAINT ck_produits_reduction_valide
        CHECK (reduction_pct >= 0 AND reduction_pct <= 1)
);

-- Tentatives invalides refusées
INSERT INTO produits (nom, prix, stock, reduction_pct)
VALUES ('Laptop', -500, 10, 0.2);
-- ERROR: violates check constraint "ck_produits_prix_positif"

INSERT INTO produits (nom, prix, stock, reduction_pct)
VALUES ('Laptop', 500, -5, 0.2);
-- ERROR: violates check constraint "ck_produits_stock_positif"
```

**Contraintes CHECK Avancées**

```sql
CREATE TABLE evenements (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(200) NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,

    -- La date de fin doit être après la date de début
    CONSTRAINT ck_evenements_dates_coherentes
        CHECK (date_fin > date_debut)
);

CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    age INTEGER,
    salaire NUMERIC(10,2),

    -- Age entre 18 et 70 ans
    CONSTRAINT ck_employees_age_valide
        CHECK (age >= 18 AND age <= 70),

    -- Salaire minimum
    CONSTRAINT ck_employees_salaire_minimum
        CHECK (salaire >= 1500)
);
```

**Avantages des Contraintes CHECK**

1. **Intégrité au niveau de la base** : Impossible d'insérer des données invalides
2. **Documentation du schéma** : Les règles métier sont explicites
3. **Centralisation** : Pas besoin de valider dans chaque application
4. **Performance** : Validation au plus proche des données

**Limites des Contraintes CHECK**

- Ne peuvent pas référencer d'autres tables (utiliser un trigger pour ça)
- Ne peuvent pas utiliser de fonctions non-déterministes (NOW(), RANDOM())
- Peuvent ralentir les insertions massives

**Audit des CHECK**

Points à vérifier :
- ✅ Les valeurs numériques ont-elles des limites logiques ?
- ✅ Les pourcentages sont-ils entre 0 et 100 ?
- ✅ Les dates de fin sont-elles après les dates de début ?
- ✅ Les statuts sont-ils limités à des valeurs valides ?

### 4. Contraintes NOT NULL (Rappel)

Déjà couvert dans la section Tables, mais c'est aussi une contrainte d'intégrité fondamentale.

### 5. Contraintes DEFAULT

**Qu'est-ce qu'une Valeur par Défaut ?**

Une valeur automatiquement insérée si aucune valeur n'est fournie.

**Exemples d'Usage**

```sql
CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT NOT NULL,
    statut VARCHAR(20) NOT NULL DEFAULT 'en_attente',
    date_commande TIMESTAMP NOT NULL DEFAULT NOW(),
    priorite INTEGER NOT NULL DEFAULT 1,
    remise NUMERIC(5,2) NOT NULL DEFAULT 0.00
);

-- Insertion sans spécifier les colonnes avec DEFAULT
INSERT INTO commandes (client_id)
VALUES (42);
-- statut = 'en_attente', date_commande = NOW(), priorite = 1, remise = 0.00
```

**Bonnes Pratiques DEFAULT**

✅ **Valeurs Sensées**
```sql
-- BON : Valeur par défaut logique
created_at TIMESTAMP NOT NULL DEFAULT NOW()
is_active BOOLEAN NOT NULL DEFAULT TRUE
quantity INTEGER NOT NULL DEFAULT 1

-- MAUVAIS : Valeur par défaut qui n'a pas de sens
prix NUMERIC(10,2) NOT NULL DEFAULT 0  -- Un prix à 0 ?
```

✅ **Combiner NOT NULL + DEFAULT**

Cela rend la colonne toujours définie mais avec une valeur par défaut :
```sql
statut VARCHAR(20) NOT NULL DEFAULT 'actif'
```

**Audit des DEFAULT**

Points à vérifier :
- ✅ `created_at` a-t-il `DEFAULT NOW()` ?
- ✅ Les booléens ont-ils une valeur par défaut claire ?
- ✅ Les statuts ont-ils un état initial par défaut ?
- ✅ Les valeurs par défaut sont-elles logiques métier ?

---

## Audit des Index

Les index sont cruciaux pour les performances, mais mal gérés, ils peuvent aussi les dégrader.

### 1. Présence d'Index Essentiels

**Quelles Colonnes Doivent Être Indexées ?**

✅ **TOUJOURS Indexer**
1. **Clés primaires** : Automatique (PostgreSQL le fait)
2. **Clés étrangères** : TRÈS IMPORTANT (pas automatique !)
3. **Colonnes dans WHERE fréquents** : Filtres courants
4. **Colonnes dans JOIN** : Améliore les jointures
5. **Colonnes dans ORDER BY** : Tri rapide

**Le Problème des FK Non Indexées**

```sql
-- Sans index sur client_id
CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(id)
);

-- Requête LENTE (scan complet de la table)
SELECT * FROM commandes WHERE client_id = 42;
-- Temps : Proportionnel au nombre total de commandes

-- Avec index (RAPIDE)
CREATE INDEX idx_commandes_client_id ON commandes(client_id);

-- Même requête maintenant RAPIDE
SELECT * FROM commandes WHERE client_id = 42;
-- Temps : Constant, quelques millisecondes
```

**Impact Performance**

Sans index sur FK :
- Requêtes lentes sur les relations
- Problèmes lors de DELETE CASCADE (scan complet)
- Locks prolongés
- Saturation CPU

### 2. Index Inutilisés ou Redondants

**Index Inutilisés**

Des index qui existent mais ne sont jamais utilisés par les requêtes.

**Comment les Identifier ?**

PostgreSQL suit l'utilisation des index dans `pg_stat_user_indexes` :

```sql
-- Requête pour trouver les index inutilisés
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS utilisations,
    pg_size_pretty(pg_relation_size(indexrelid)) AS taille
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Jamais utilisé
    AND indexrelid NOT IN (
        SELECT indexrelid
        FROM pg_constraint
        WHERE contype IN ('p', 'u')  -- Exclure PK et UNIQUE
    )
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Pourquoi les Supprimer ?**

1. **Consommation d'espace disque** : Peut être significative
2. **Ralentissement des écritures** : Chaque INSERT/UPDATE/DELETE doit mettre à jour tous les index
3. **Consommation mémoire** : Index chargés en cache inutilement

**Index Redondants**

Des index qui se chevauchent ou sont en doublon.

**Exemples de Redondance**

```sql
-- Index 1 : Sur une seule colonne
CREATE INDEX idx_users_email ON users(email);

-- Index 2 : Multi-colonnes commençant par email
CREATE INDEX idx_users_email_created ON users(email, created_at);

-- Le deuxième index REND LE PREMIER REDONDANT
-- PostgreSQL peut utiliser idx_users_email_created même pour des requêtes
-- qui filtrent uniquement sur email
```

**Cas Particulier : Index Partiels**

Parfois, la "redondance" est justifiée :

```sql
-- Index 1 : Complet
CREATE INDEX idx_orders_status ON orders(status);

-- Index 2 : Partiel sur statut "pending" uniquement
CREATE INDEX idx_orders_status_pending
ON orders(status)
WHERE status = 'pending';

-- Pas de redondance : Le 2ème est plus petit et plus rapide pour les
-- requêtes WHERE status = 'pending'
```

### 3. Index Multi-Colonnes Mal Ordonnés

**L'Ordre des Colonnes Compte !**

Dans un index multi-colonnes, l'ordre est critique :

```sql
-- Index A
CREATE INDEX idx_orders_client_date ON orders(client_id, order_date);

-- Index B
CREATE INDEX idx_orders_date_client ON orders(order_date, client_id);
```

**Ces deux index ne sont PAS interchangeables !**

**Règle d'Utilisation**

Un index `(A, B, C)` peut être utilisé pour :
- `WHERE A = ...` ✅
- `WHERE A = ... AND B = ...` ✅
- `WHERE A = ... AND B = ... AND C = ...` ✅
- `WHERE B = ...` ❌ (sauf avec Skip Scan en PG 18)
- `WHERE C = ...` ❌

**Ordre Optimal**

1. **Colonnes avec égalité** (`=`) en premier
2. **Colonnes triées** (`ORDER BY`) en dernier
3. **Colonnes les plus sélectives** en premier (celles qui filtrent le plus)

```sql
-- BON : Filtre d'abord sur client (très sélectif), puis date
CREATE INDEX idx_orders_client_date ON orders(client_id, order_date);

-- Usage optimal
SELECT * FROM orders
WHERE client_id = 42
ORDER BY order_date DESC;
```

### 4. Sur-Indexation

**Le Problème de Trop d'Index**

Avoir trop d'index cause :
1. **Ralentissement des écritures** : Chaque modification doit mettre à jour tous les index
2. **Consommation mémoire et disque** : Peut devenir majeure
3. **Ralentissement du planificateur** : Trop d'options à évaluer

**Symptômes de Sur-Indexation**

- Dizaines d'index sur une seule table
- Temps d'INSERT/UPDATE très lents
- Espace disque qui explose
- Index rarement ou jamais utilisés

**Équilibre à Trouver**

| Type de Table | Nombre d'Index Recommandé |
|---------------|---------------------------|
| Table de référence (lecture seule) | Autant que nécessaire |
| Table OLTP (mix lecture/écriture) | 3-7 index |
| Table d'écriture intensive | 1-3 index (PK + FK) |

### 5. Types d'Index Inappropriés

**PostgreSQL propose plusieurs types d'index**

**B-Tree (Par Défaut)**
- Usage : Comparaisons (=, <, >, <=, >=, BETWEEN)
- Idéal pour : La plupart des cas

**Hash**
- Usage : Égalité stricte uniquement (=)
- Idéal pour : Rarement utilisé (B-Tree est généralement meilleur)

**GIN (Generalized Inverted Index)**
- Usage : Recherche dans tableaux, JSONB, full-text
- Idéal pour : Données complexes

```sql
-- Index GIN sur colonne JSONB
CREATE INDEX idx_products_attributes
ON products USING GIN (attributes);

-- Recherche dans JSONB rapide
SELECT * FROM products
WHERE attributes @> '{"color": "red"}';
```

**GiST (Generalized Search Tree)**
- Usage : Données géométriques, texte, ltree
- Idéal pour : PostGIS, recherche géospatiale

**BRIN (Block Range Index)**
- Usage : Données séquentielles (dates, timestamps)
- Idéal pour : Tables massives avec ordre naturel

```sql
-- BRIN pour table de logs avec millions de lignes
CREATE INDEX idx_logs_timestamp
ON logs USING BRIN (timestamp);
-- Très compact (quelques Ko pour des millions de lignes)
```

**Audit des Types d'Index**

Points à vérifier :
- ✅ Les colonnes JSONB ont-elles des index GIN ?
- ✅ Les grandes tables avec dates ont-elles des index BRIN ?
- ✅ Les index Hash sont-ils vraiment nécessaires ?

---

## Audit des Types de Données

Le choix du bon type de données est crucial pour :
- L'intégrité des données
- Les performances
- L'espace disque
- La clarté du schéma

### 1. VARCHAR vs TEXT

**La Question Éternelle**

PostgreSQL ne fait AUCUNE différence de performance entre :
- `VARCHAR` (avec ou sans limite)
- `TEXT`

**Recommandations**

✅ **Utiliser TEXT par Défaut**
```sql
-- Préférer
description TEXT

-- Plutôt que
description VARCHAR(1000)  -- Limite arbitraire
```

✅ **Utiliser VARCHAR(n) Quand la Limite est Métier**
```sql
-- BON : Contrainte métier réelle
code_postal VARCHAR(5)
code_pays VARCHAR(2)  -- ISO 3166-1 alpha-2

-- BON : Contrainte d'affichage
titre VARCHAR(200)  -- Titre de page web
```

**Problèmes Courants**

❌ **Limites Arbitraires**
```sql
-- MAUVAIS : Pourquoi 255 ? Pourquoi pas 256 ?
nom VARCHAR(255)

-- MIEUX
nom TEXT
-- OU avec contrainte CHECK si vraiment nécessaire
nom VARCHAR(100) CHECK (length(nom) >= 2)
```

### 2. INTEGER vs BIGINT

**Choisir la Bonne Taille**

| Type | Plage | Espace | Usage |
|------|-------|--------|-------|
| SMALLINT | -32,768 à 32,767 | 2 octets | Petites valeurs |
| INTEGER | -2 milliards à 2 milliards | 4 octets | Usage général |
| BIGINT | -9×10^18 à 9×10^18 | 8 octets | Grandes valeurs |

**Règle Générale**

✅ **Utiliser BIGINT pour les Clés Primaires**
```sql
-- BON : Prévient l'épuisement des ID
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- BIGINT auto-incrémenté
    ...
);
```

✅ **Utiliser INTEGER pour les Compteurs**
```sql
-- BON : Suffisant pour la plupart des compteurs
views_count INTEGER NOT NULL DEFAULT 0,
stock_quantity INTEGER NOT NULL DEFAULT 0
```

**Le Problème de l'Épuisement des ID**

```sql
-- INTEGER (4 octets) : max 2,147,483,647
CREATE TABLE articles (
    id SERIAL PRIMARY KEY  -- INTEGER auto-incrémenté
);

-- Si vous insérez 1000 articles/jour :
-- 2,147,483,647 / 1000 = 2,147,483 jours = 5,884 ans → OK

-- Mais si vous insérez 10,000 articles/jour :
-- 2,147,483,647 / 10,000 = 214,748 jours = 588 ans → OK

-- Mais pour une table de logs avec 1,000,000 lignes/jour :
-- 2,147,483,647 / 1,000,000 = 2,147 jours = 5,8 ans → PROBLÈME !
```

**Migration Ultérieure Douloureuse**

Migrer de `INTEGER` vers `BIGINT` sur une table avec des millions de lignes est **très coûteux** :
- Réécriture complète de la table
- Verrous exclusifs prolongés
- Downtime potentiel

**Conclusion : BIGINT par Défaut pour les PK**

### 3. NUMERIC vs FLOAT

**Différences Fondamentales**

**NUMERIC (ou DECIMAL)**
- Précision exacte
- Usage : Argent, calculs financiers
- Syntaxe : `NUMERIC(precision, scale)`

```sql
-- NUMERIC(10, 2) : 10 chiffres total, 2 après la virgule
-- Exemple : 12345678.90
prix NUMERIC(10, 2)
```

**FLOAT / DOUBLE PRECISION**
- Précision approximative (virgule flottante)
- Usage : Calculs scientifiques, coordonnées GPS
- Erreurs d'arrondi possibles

**Exemple de Problème avec FLOAT**

```sql
-- Avec FLOAT
SELECT 0.1::FLOAT + 0.2::FLOAT;
-- Résultat : 0.30000000000000004 (!)

-- Avec NUMERIC
SELECT 0.1::NUMERIC + 0.2::NUMERIC;
-- Résultat : 0.3 (exact)
```

**Règle d'Or**

✅ **Argent → NUMERIC**
```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    montant NUMERIC(10, 2) NOT NULL,  -- Toujours NUMERIC
    taux_tva NUMERIC(5, 4) NOT NULL   -- Exemple : 0.2000 (20%)
);
```

✅ **Sciences/Stats → FLOAT**
```sql
CREATE TABLE mesures (
    id BIGSERIAL PRIMARY KEY,
    temperature DOUBLE PRECISION,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION
);
```

### 4. Date et Temps

**Types de Dates en PostgreSQL**

| Type | Stocke | Usage |
|------|--------|-------|
| DATE | Date uniquement | Anniversaires, dates historiques |
| TIME | Heure uniquement | Horaires d'ouverture |
| TIMESTAMP | Date + Heure | Logs, événements |
| TIMESTAMPTZ | Date + Heure + Fuseau | Recommandé (gère les fuseaux) |
| INTERVAL | Durée | Calculs de durées |

**TIMESTAMP vs TIMESTAMPTZ**

✅ **TOUJOURS Utiliser TIMESTAMPTZ**
```sql
-- BON
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

-- MAUVAIS
created_at TIMESTAMP NOT NULL DEFAULT NOW()
```

**Pourquoi TIMESTAMPTZ ?**

1. **Gestion Automatique des Fuseaux Horaires**
   - Stocke en UTC en interne
   - Convertit automatiquement selon le fuseau du client

2. **Évite les Bugs de Fuseaux Horaires**
   - Pas de confusion lors de changements d'heure (DST)
   - Comparaisons correctes entre dates de différents fuseaux

3. **Interopérabilité**
   - Standard international
   - Facile à utiliser avec des APIs

**Exemple de Problème avec TIMESTAMP**

```sql
-- Avec TIMESTAMP (sans fuseau)
INSERT INTO events (name, event_time)
VALUES ('Meeting', '2025-03-15 14:00:00');
-- Question : 14h dans quel fuseau horaire ? 🤔

-- Avec TIMESTAMPTZ (recommandé)
INSERT INTO events (name, event_time)
VALUES ('Meeting', '2025-03-15 14:00:00+01:00');  -- CET explicite
-- OU
VALUES ('Meeting', '2025-03-15 13:00:00+00:00');  -- UTC explicite
```

### 5. Types Spéciaux PostgreSQL

**JSON vs JSONB**

| Type | Stockage | Performance | Usage |
|------|----------|-------------|-------|
| JSON | Texte brut | Lent | Rarement utilisé |
| JSONB | Binaire | Rapide | Toujours préférer |

```sql
-- TOUJOURS utiliser JSONB
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    attributes JSONB  -- Pas JSON
);

-- Index GIN sur JSONB
CREATE INDEX idx_products_attributes
ON products USING GIN (attributes);

-- Requêtes rapides
SELECT * FROM products
WHERE attributes @> '{"color": "red"}';
```

**UUID vs UUIDv7 (Nouveauté PG 18)**

```sql
-- UUID classique (v4)
id UUID DEFAULT gen_random_uuid()
-- Problème : Pas d'ordre temporel, fragmentation d'index

-- UUIDv7 (PG 18 - recommandé)
id UUID DEFAULT gen_random_uuid_v7()
-- Avantage : Contient un timestamp, meilleur pour les index B-Tree
```

**ENUM vs VARCHAR**

```sql
-- ENUM : Type personnalisé
CREATE TYPE statut_commande AS ENUM ('en_attente', 'traitee', 'annulee');

CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    statut statut_commande NOT NULL DEFAULT 'en_attente'
);
```

✅ **Avantages ENUM** :
- Validation automatique
- Stockage compact
- Clarté du schéma

❌ **Inconvénients ENUM** :
- Difficile à modifier (ajout de valeurs)
- Nécessite une migration pour changer

**Alternative : VARCHAR + CHECK**
```sql
statut VARCHAR(20) NOT NULL
    CHECK (statut IN ('en_attente', 'traitee', 'annulee'))
    DEFAULT 'en_attente'
```

Plus flexible mais moins efficace en espace.

---

## Audit de la Normalisation

La normalisation organise les données pour **minimiser la redondance** et **maximiser l'intégrité**.

### Les Formes Normales (Rappel Simplifié)

**1NF (Première Forme Normale)**
- Pas de colonnes multi-valuées
- Chaque cellule contient une valeur atomique

**2NF (Deuxième Forme Normale)**
- 1NF + Pas de dépendances partielles sur la clé primaire

**3NF (Troisième Forme Normale)**
- 2NF + Pas de dépendances transitives

**BCNF (Boyce-Codd)**
- Version renforcée de 3NF

### Violations Courantes

#### Violation 1NF : Colonnes Multi-Valuées

❌ **Mauvais Design**
```sql
CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT,
    produits TEXT  -- "Laptop, Mouse, Keyboard" ← VIOLATION 1NF
);
```

✅ **Bon Design**
```sql
CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT
);

CREATE TABLE lignes_commandes (
    id BIGSERIAL PRIMARY KEY,
    commande_id BIGINT NOT NULL REFERENCES commandes(id),
    produit_id BIGINT NOT NULL REFERENCES produits(id),
    quantite INTEGER NOT NULL
);
```

#### Violation 3NF : Dépendances Transitives

❌ **Mauvais Design**
```sql
CREATE TABLE employes (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement_id BIGINT,
    departement_nom VARCHAR(100),  -- REDONDANCE
    departement_responsable VARCHAR(100)  -- REDONDANCE
);
-- Si le département change de nom, il faut MAJ tous les employés
```

✅ **Bon Design**
```sql
CREATE TABLE departements (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    responsable VARCHAR(100)
);

CREATE TABLE employes (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement_id BIGINT REFERENCES departements(id)
);
```

### Dénormalisation Stratégique

**Quand Dénormaliser ?**

La sur-normalisation peut causer des problèmes de performance. Parfois, la dénormalisation est justifiée.

**Cas d'Usage Légitimes**

1. **Colonnes Calculées pour Performance**
```sql
CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT,
    montant_total NUMERIC(10, 2),  -- Dénormalisé (somme des lignes)
    nb_articles INTEGER  -- Dénormalisé (compte des lignes)
);
-- Évite des calculs coûteux sur chaque requête
-- À maintenir via triggers
```

2. **Colonnes de Recherche Dénormalisées**
```sql
CREATE TABLE articles (
    id BIGSERIAL PRIMARY KEY,
    titre VARCHAR(200),
    auteur_id BIGINT REFERENCES auteurs(id),
    auteur_nom VARCHAR(100),  -- Dénormalisé pour performance
    search_vector TSVECTOR  -- Dénormalisé pour full-text search
);
-- Évite des JOINs dans les recherches fréquentes
```

3. **Colonnes Matérialisées pour Analytics**
```sql
CREATE TABLE statistiques_journalieres (
    date DATE PRIMARY KEY,
    total_ventes NUMERIC(12, 2),  -- Agrégat dénormalisé
    nb_commandes INTEGER,
    panier_moyen NUMERIC(10, 2)
);
-- Données pré-calculées pour dashboards
```

**Règles de Dénormalisation**

1. ✅ Dénormaliser seulement si un besoin de performance est prouvé
2. ✅ Maintenir la cohérence via triggers ou batch jobs
3. ✅ Documenter clairement la dénormalisation
4. ✅ Considérer d'abord les vues matérialisées

---

## Audit de Sécurité

La sécurité au niveau du schéma est souvent négligée mais critique.

### 1. Gestion des Rôles et Permissions

**Principe du Moindre Privilège**

Chaque utilisateur/application ne doit avoir que les permissions strictement nécessaires.

**Anti-Pattern Courant**

❌ **Tout le monde est superuser**
```sql
-- MAUVAIS : L'application a tous les droits
GRANT ALL PRIVILEGES ON DATABASE mydb TO app_user;
```

✅ **Permissions Granulaires**
```sql
-- Créer un rôle pour l'application
CREATE ROLE app_user LOGIN PASSWORD 'secure_password';

-- Permissions minimales
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE ON TABLE users TO app_user;
GRANT SELECT ON TABLE products TO app_user;  -- Lecture seule

-- Permissions sur séquences (pour SERIAL/IDENTITY)
GRANT USAGE ON SEQUENCE users_id_seq TO app_user;
```

**Rôles Spécialisés**

```sql
-- Rôle lecture seule pour analytics
CREATE ROLE analytics_readonly;
GRANT CONNECT ON DATABASE mydb TO analytics_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analytics_readonly;

-- Rôle admin pour migrations
CREATE ROLE migrations_admin;
GRANT ALL PRIVILEGES ON DATABASE mydb TO migrations_admin;
```

### 2. Row-Level Security (RLS)

**Qu'est-ce que le RLS ?**

Permet de filtrer les lignes automatiquement selon l'utilisateur connecté.

**Cas d'Usage : Application Multi-Tenant**

```sql
-- Table avec colonne tenant_id
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    titre VARCHAR(200),
    contenu TEXT
);

-- Activer RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Politique : Chaque tenant ne voit que ses documents
CREATE POLICY tenant_isolation ON documents
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

-- L'application définit le tenant au début de chaque session
SET app.current_tenant_id = 42;

-- Toutes les requêtes sont automatiquement filtrées
SELECT * FROM documents;
-- Ne retourne QUE les documents du tenant 42
```

**Avantages RLS**

1. Sécurité au niveau base de données (pas seulement applicatif)
2. Impossible d'oublier un WHERE tenant_id = ...
3. Protection contre les injections SQL
4. Centralisation de la logique de sécurité

### 3. Colonnes Sensibles

**Protection des Données Personnelles (RGPD)**

Certaines colonnes doivent avoir une attention particulière :
- Noms, prénoms
- Emails
- Numéros de téléphone
- Adresses
- Données de santé
- Données financières

**Stratégies de Protection**

1. **Chiffrement au Repos** (TDE - Transparent Data Encryption)
2. **Chiffrement Applicatif** (avant insertion dans la base)
3. **Pseudonymisation** (remplacer par des identifiants)
4. **Hachage** (pour données non-réversibles comme mots de passe)

**Exemple : Mots de Passe**

❌ **JAMAIS Stocker en Clair**
```sql
-- INACCEPTABLE
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255),
    password TEXT  -- DANGER : Mot de passe en clair
);
```

✅ **Toujours Hasher**
```sql
-- BON : Hash bcrypt
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255),
    password_hash TEXT  -- Hash bcrypt/argon2
);

-- Exemple d'insertion (côté application)
-- password_hash = bcrypt.hash('my_password', rounds=12)
INSERT INTO users (email, password_hash)
VALUES ('user@example.com', '$2b$12$...');
```

### 4. Audit Trail et Logs

**Traçabilité des Modifications**

Pour des raisons de sécurité et de conformité, il faut souvent savoir :
- Qui a modifié quoi ?
- Quand ?
- Quelle était l'ancienne valeur ?

**Approche 1 : Colonnes d'Audit**

```sql
CREATE TABLE commandes (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT,
    montant NUMERIC(10, 2),

    -- Audit trail
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by BIGINT REFERENCES users(id),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Approche 2 : Table d'Historique**

```sql
-- Table principale
CREATE TABLE produits (
    id BIGSERIAL PRIMARY KEY,
    nom VARCHAR(200),
    prix NUMERIC(10, 2)
);

-- Table d'historique (toutes les versions)
CREATE TABLE produits_historique (
    id BIGSERIAL PRIMARY KEY,
    produit_id BIGINT NOT NULL,
    nom VARCHAR(200),
    prix NUMERIC(10, 2),
    operation VARCHAR(10),  -- INSERT, UPDATE, DELETE
    modified_by BIGINT,
    modified_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Trigger pour alimenter l'historique
CREATE OR REPLACE FUNCTION log_produit_change()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO produits_historique (produit_id, nom, prix, operation)
        VALUES (OLD.id, OLD.nom, OLD.prix, 'DELETE');
    ELSE
        INSERT INTO produits_historique (produit_id, nom, prix, operation)
        VALUES (NEW.id, NEW.nom, NEW.prix, TG_OP);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_produit_audit
AFTER INSERT OR UPDATE OR DELETE ON produits
FOR EACH ROW EXECUTE FUNCTION log_produit_change();
```

---

## Audit de Performance

### 1. Tables Sans Statistiques

**Qu'est-ce que ANALYZE ?**

ANALYZE collecte des statistiques sur les données pour aider le planificateur à choisir les meilleurs plans d'exécution.

**Symptômes de Statistiques Obsolètes**

- Plans d'exécution sous-optimaux
- Index inutilisés alors qu'ils devraient l'être
- Scans séquentiels au lieu d'index scans

**Vérifier les Statistiques**

```sql
-- Dernière analyse par table
SELECT
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY last_analyze NULLS FIRST;
```

**Solution : Autovacuum**

PostgreSQL lance automatiquement ANALYZE via l'autovacuum. Vérifier qu'il est actif :

```sql
SHOW autovacuum;  -- Doit être 'on'
```

### 2. Bloat (Fragmentation)

**Qu'est-ce que le Bloat ?**

Avec MVCC, PostgreSQL ne supprime pas immédiatement les anciennes versions. Cela crée du "bloat" : de l'espace mort.

**Impact du Bloat**

- Tables et index gonflés
- Performance dégradée (plus de pages à lire)
- Espace disque gaspillé

**Détection du Bloat**

Extensions recommandées :
- `pgstattuple` : Analyse détaillée
- Scripts communautaires

**Prévention du Bloat**

1. ✅ Autovacuum correctement configuré
2. ✅ VACUUM réguliers sur tables à forte écriture
3. ✅ Éviter les transactions très longues (bloquent le VACUUM)

### 3. Partitionnement Manquant

**Quand Partitionner ?**

Tables avec :
- Des millions de lignes
- Données temporelles (logs, événements)
- Requêtes qui filtrent sur une colonne (date, région)

**Avantages du Partitionnement**

1. **Partition Pruning** : Ignore les partitions non pertinentes
2. **Maintenance facilitée** : Supprimer des données = DROP partition
3. **Requêtes plus rapides** : Scans sur moins de données

**Exemple : Partitionnement par Date**

```sql
-- Table parent
CREATE TABLE logs (
    id BIGSERIAL,
    timestamp TIMESTAMPTZ NOT NULL,
    message TEXT,
    level VARCHAR(10)
) PARTITION BY RANGE (timestamp);

-- Partitions mensuelles
CREATE TABLE logs_2025_01 PARTITION OF logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE logs_2025_02 PARTITION OF logs
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Requête automatiquement optimisée
SELECT * FROM logs
WHERE timestamp >= '2025-01-15' AND timestamp < '2025-01-20';
-- PostgreSQL scan UNIQUEMENT logs_2025_01
```

### 4. Colonnes Calculées Non Optimisées

**Problème : Calculs Répétitifs**

```sql
-- Calcul répété à chaque requête
SELECT
    prix,
    quantite,
    prix * quantite AS total  -- Recalculé à chaque fois
FROM lignes_commandes
WHERE prix * quantite > 100;  -- Pas d'index possible
```

**Solution 1 : Colonne Stockée**

```sql
ALTER TABLE lignes_commandes
ADD COLUMN total NUMERIC(10, 2)
    GENERATED ALWAYS AS (prix * quantite) STORED;

-- Index possible
CREATE INDEX idx_lignes_total ON lignes_commandes(total);

-- Requête optimisée
SELECT * FROM lignes_commandes WHERE total > 100;
```

**Solution 2 : Colonnes Virtuelles (PG 18)**

```sql
-- Nouveauté PostgreSQL 18
ALTER TABLE lignes_commandes
ADD COLUMN total NUMERIC(10, 2)
    GENERATED ALWAYS AS (prix * quantite);  -- VIRTUAL par défaut

-- Pas de stockage, calcul à la volée, mais index possible
```

### 5. Configuration Inadaptée

**Paramètres Critiques**

```sql
-- shared_buffers : Cache PostgreSQL (15-25% de la RAM)
shared_buffers = 4GB

-- work_mem : Mémoire par opération de tri (trop bas = disk sorts)
work_mem = 64MB

-- maintenance_work_mem : Opérations de maintenance (VACUUM, CREATE INDEX)
maintenance_work_mem = 512MB

-- effective_cache_size : Indique la RAM disponible pour l'OS (50-75% RAM)
effective_cache_size = 16GB
```

**Outils d'Aide**

- **PGTune** : Génère une config selon votre matériel
- **pg_stat_statements** : Identifie les requêtes lentes

---

## Checklist Complète

### ✅ Structure des Tables

- [ ] Toutes les tables ont une clé primaire
- [ ] Les clés primaires sont BIGSERIAL (ou UUID)
- [ ] Conventions de nommage cohérentes (snake_case)
- [ ] Colonnes d'audit trail présentes (created_at, updated_at)
- [ ] Stratégie soft delete vs hard delete définie
- [ ] Colonnes NOT NULL là où nécessaire
- [ ] Valeurs DEFAULT logiques et cohérentes

### ✅ Contraintes d'Intégrité

- [ ] Foreign Keys présentes sur toutes les relations
- [ ] Actions CASCADE/RESTRICT appropriées sur les FK
- [ ] Contraintes UNIQUE sur colonnes métier (email, username)
- [ ] Contraintes CHECK pour validation métier
- [ ] Contraintes nommées explicitement
- [ ] Contraintes temporelles (PG 18) si pertinentes

### ✅ Indexation

- [ ] Index sur toutes les clés étrangères
- [ ] Index sur colonnes fréquemment filtrées (WHERE)
- [ ] Index sur colonnes de jointure
- [ ] Index multi-colonnes correctement ordonnés
- [ ] Pas d'index inutilisés ou redondants
- [ ] Type d'index approprié (B-Tree, GIN, BRIN)
- [ ] Index partiels pour sous-ensembles fréquents

### ✅ Types de Données

- [ ] TEXT préféré sauf contrainte métier
- [ ] BIGINT pour les clés primaires
- [ ] NUMERIC pour les montants financiers
- [ ] TIMESTAMPTZ pour les dates
- [ ] JSONB (pas JSON) pour données semi-structurées
- [ ] UUID ou UUIDv7 si distribué
- [ ] Types appropriés pour la précision nécessaire

### ✅ Normalisation

- [ ] Pas de colonnes multi-valuées (1NF)
- [ ] Pas de dépendances partielles (2NF)
- [ ] Pas de dépendances transitives (3NF)
- [ ] Dénormalisation justifiée et documentée
- [ ] Redondance maintenue par triggers si nécessaire

### ✅ Sécurité

- [ ] Principe du moindre privilège appliqué
- [ ] Rôles granulaires définis
- [ ] Row-Level Security (RLS) si multi-tenant
- [ ] Colonnes sensibles protégées
- [ ] Mots de passe hashés (jamais en clair)
- [ ] Audit trail pour traçabilité
- [ ] Permissions documentées

### ✅ Performance

- [ ] Statistiques à jour (ANALYZE)
- [ ] Autovacuum activé et configuré
- [ ] Bloat surveillé et géré
- [ ] Partitionnement sur grandes tables temporelles
- [ ] Colonnes calculées stockées ou virtuelles
- [ ] Configuration PostgreSQL optimisée
- [ ] Vues matérialisées pour agrégats coûteux

### ✅ Maintenance et Monitoring

- [ ] pg_stat_statements installé
- [ ] Logs configurés et analysés
- [ ] Monitoring des métriques vitales actif
- [ ] Sauvegardes régulières testées
- [ ] Stratégie de migration définie
- [ ] Documentation du schéma à jour

---

## Conclusion

L'audit de schéma est une **discipline continue**, pas un événement ponctuel. Un schéma bien conçu est :

- ✅ **Performant** : Requêtes rapides, index appropriés
- ✅ **Intègre** : Contraintes qui garantissent la cohérence
- ✅ **Sécurisé** : Permissions granulaires, données protégées
- ✅ **Maintenable** : Conventions claires, documentation
- ✅ **Évolutif** : Prêt pour la croissance future

### Prochaines Étapes

1. **Auditer votre schéma actuel** avec cette checklist
2. **Prioriser les problèmes** : Sécurité > Performance > Conventions
3. **Planifier les corrections** : Migrations par étapes
4. **Automatiser** : Scripts de validation, monitoring continu
5. **Documenter** : Rationale des choix, patterns utilisés

### Ressources Complémentaires

- Documentation PostgreSQL officielle
- Extensions utiles : pgstattuple, pg_stat_statements, HypoPG
- Outils de visualisation : pgAdmin, DBeaver, dbdiagram.io
- Communauté : pgsql-general, Reddit r/PostgreSQL

---


⏭️ [Nouveautés PostgreSQL 18 en un Coup d'Œil](/annexes/nouveautes-pg18/README.md)
