🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2. Nouveauté PostgreSQL 18 : Contraintes Temporelles (Temporal Constraints)

## Introduction

Les **contraintes temporelles** (Temporal Constraints) sont une fonctionnalité majeure introduite dans **PostgreSQL 18** (septembre 2025). Elles permettent de gérer des **périodes de validité** et d'empêcher les **chevauchements temporels** directement au niveau de la base de données.

Cette nouveauté révolutionnaire simplifie considérablement la gestion des données qui ont une **dimension temporelle**, comme les réservations, les historiques de prix, les contrats, ou les périodes d'emploi.

### Pourquoi cette nouveauté ?

Avant PostgreSQL 18, gérer les chevauchements de périodes nécessitait :
- Des triggers complexes
- Des contraintes CHECK avec des sous-requêtes (interdites)
- De la logique applicative lourde et sujette à erreurs
- Des index GIST avec des extensions tierces (btree_gist)

Avec PostgreSQL 18, la gestion temporelle devient **native**, **simple** et **performante**.

---

## Le Problème : Gérer les Périodes et les Chevauchements

### Cas d'usage typiques

Imaginons ces scénarios courants :

1. **Réservations de salles** : Une salle ne peut pas être réservée par deux personnes en même temps
2. **Historique des prix** : Un produit ne peut avoir qu'un seul prix valide à une date donnée
3. **Contrats d'emploi** : Un employé ne peut avoir qu'un seul contrat actif simultanément
4. **Locations de véhicules** : Une voiture ne peut être louée par deux clients en même temps
5. **Périodes de validité** : Une promotion ne peut pas avoir deux périodes qui se chevauchent

### Le défi avant PostgreSQL 18

```sql
-- Structure classique pour gérer des périodes
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    utilisateur_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    CHECK (date_fin > date_debut)  -- OK : Vérifie la cohérence interne
);
```

**Problème** : Cette table ne peut pas empêcher deux réservations qui se chevauchent !

```sql
-- ✅ Ces deux réservations se chevauchent, mais sont acceptées !
INSERT INTO reservations (salle_id, utilisateur_id, date_debut, date_fin)
VALUES
    (1, 100, '2025-01-10 14:00', '2025-01-10 16:00'),
    (1, 101, '2025-01-10 15:00', '2025-01-10 17:00');  -- Chevauchement !
```

Une contrainte `CHECK` ne peut pas résoudre ce problème car elle ne peut interroger que **la ligne courante**, pas les autres lignes de la table.

### Solutions avant PostgreSQL 18

#### Solution 1 : Extension btree_gist + contrainte d'exclusion

```sql
-- Nécessite l'extension btree_gist
CREATE EXTENSION btree_gist;

CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    -- Contrainte d'exclusion avec opérateur &&
    EXCLUDE USING gist (
        salle_id WITH =,
        tsrange(date_debut, date_fin) WITH &&
    )
);
```

**Inconvénients** :
- Nécessite une extension externe
- Syntaxe complexe pour les débutants
- Moins optimisée que les nouvelles contraintes temporelles natives

#### Solution 2 : Triggers personnalisés

```sql
CREATE OR REPLACE FUNCTION check_reservation_overlap()
RETURNS TRIGGER AS $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM reservations
        WHERE salle_id = NEW.salle_id
          AND id != NEW.id
          AND (date_debut, date_fin) OVERLAPS (NEW.date_debut, NEW.date_fin)
    ) THEN
        RAISE EXCEPTION 'Chevauchement de réservation détecté';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_overlap_before_insert
BEFORE INSERT OR UPDATE ON reservations
FOR EACH ROW EXECUTE FUNCTION check_reservation_overlap();
```

**Inconvénients** :
- Code procédural complexe à maintenir
- Risques de race conditions en haute concurrence
- Performance moins optimale
- Logique métier dispersée dans les triggers

---

## La Solution PostgreSQL 18 : Contraintes Temporelles Natives

PostgreSQL 18 introduit une **syntaxe native** pour gérer les contraintes temporelles de manière élégante et performante.

### Concept : Le Type de Données Range

Les contraintes temporelles s'appuient sur les **types Range** de PostgreSQL :

| Type Range | Description | Exemple |
|-----------|-------------|---------|
| `int4range` | Intervalle d'entiers | `[1, 10)` |
| `int8range` | Intervalle de grands entiers | `[1000, 2000)` |
| `numrange` | Intervalle de décimaux | `[0.0, 100.0]` |
| `tsrange` | Intervalle de timestamps | `['2025-01-01', '2025-12-31')` |
| `tstzrange` | Intervalle de timestamps avec TZ | `['2025-01-01 00:00+00', '2025-12-31 23:59+00')` |
| `daterange` | Intervalle de dates | `[2025-01-01, 2025-12-31]` |

#### Syntaxe des Ranges

```sql
-- Notation : [inclusif, exclusif)
SELECT '[2025-01-10 14:00, 2025-01-10 16:00)'::tsrange;

-- Inclusif des deux côtés : [a, b]
SELECT '[2025-01-10 14:00, 2025-01-10 16:00]'::tsrange;

-- Exclusif des deux côtés : (a, b)
SELECT '(2025-01-10 14:00, 2025-01-10 16:00)'::tsrange;

-- Ouvert à droite : [a, infinity)
SELECT '[2025-01-01, infinity)'::tsrange;
```

#### Opérateurs sur les Ranges

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `&&` | Chevauchement (overlap) | `'[1,5)' && '[3,7)'` → TRUE |
| `@>` | Contient | `'[1,10)' @> '[2,5)'` → TRUE |
| `<@` | Est contenu dans | `'[2,5)' <@ '[1,10)'` → TRUE |
| `-\|-` | Adjacent | `'[1,5)' -\|- '[5,10)'` → TRUE |
| `<<` | Strictement à gauche | `'[1,5)' << '[10,15)'` → TRUE |
| `>>` | Strictement à droite | `'[10,15)' >> '[1,5)'` → TRUE |

**Le plus important** : L'opérateur `&&` détecte les chevauchements !

```sql
-- Exemples de chevauchement
SELECT '[2025-01-10 14:00, 2025-01-10 16:00)'::tsrange &&
       '[2025-01-10 15:00, 2025-01-10 17:00)'::tsrange;
-- Retourne TRUE (chevauchement de 14:00-16:00 avec 15:00-17:00)

SELECT '[2025-01-10 14:00, 2025-01-10 16:00)'::tsrange &&
       '[2025-01-10 16:00, 2025-01-10 18:00)'::tsrange;
-- Retourne FALSE (pas de chevauchement, fin = début)
```

---

## Syntaxe des Contraintes Temporelles PostgreSQL 18

### Contrainte UNIQUE avec Period

PostgreSQL 18 introduit la syntaxe `WITHOUT OVERLAPS` pour définir une contrainte d'unicité temporelle.

#### Syntaxe de base

```sql
CREATE TABLE nom_table (
    colonne_identifiant TYPE,
    colonne_debut TIMESTAMP,
    colonne_fin TIMESTAMP,
    UNIQUE (colonne_identifiant, PERIOD(colonne_debut, colonne_fin)) WITHOUT OVERLAPS
);
```

**Explication** :
- `UNIQUE` : Garantit l'unicité
- `colonne_identifiant` : La clé sur laquelle on vérifie l'unicité (ex : salle_id)
- `PERIOD(debut, fin)` : Définit la période temporelle
- `WITHOUT OVERLAPS` : Empêche les chevauchements temporels pour une même valeur de `colonne_identifiant`

### Exemple complet : Réservations de salles

```sql
CREATE TABLE reservations_salles (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    utilisateur_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    description TEXT,
    -- Vérification de base : date_fin > date_debut
    CHECK (date_fin > date_debut),
    -- Contrainte temporelle : Pas de chevauchement pour une même salle
    UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS
);
```

### Comportement

```sql
-- ✅ Première réservation
INSERT INTO reservations_salles (salle_id, utilisateur_id, date_debut, date_fin)
VALUES (1, 100, '2025-01-10 14:00', '2025-01-10 16:00');

-- ✅ Réservation d'une autre salle (même créneau, mais salle différente)
INSERT INTO reservations_salles (salle_id, utilisateur_id, date_debut, date_fin)
VALUES (2, 101, '2025-01-10 14:00', '2025-01-10 16:00');

-- ✅ Réservation de la même salle, mais créneau adjacent (pas de chevauchement)
INSERT INTO reservations_salles (salle_id, utilisateur_id, date_debut, date_fin)
VALUES (1, 102, '2025-01-10 16:00', '2025-01-10 18:00');

-- ❌ ÉCHEC : Chevauchement détecté !
INSERT INTO reservations_salles (salle_id, utilisateur_id, date_debut, date_fin)
VALUES (1, 103, '2025-01-10 15:00', '2025-01-10 17:00');
-- ERROR: conflicting key value violates temporal exclusion constraint
-- DETAIL: Key (salle_id, PERIOD(date_debut, date_fin))=(1, [2025-01-10 15:00, 2025-01-10 17:00))
--         conflicts with existing key (salle_id, PERIOD(date_debut, date_fin))=(1, [2025-01-10 14:00, 2025-01-10 16:00))
```

**Résultat** : PostgreSQL empêche automatiquement les réservations qui se chevauchent pour une même salle !

---

## Cas d'Usage Avancés

### 1. Historique de Prix avec Unicité Temporelle

Un produit ne peut avoir qu'**un seul prix valide** à un moment donné.

```sql
CREATE TABLE historique_prix (
    id SERIAL PRIMARY KEY,
    produit_id INTEGER NOT NULL,
    prix NUMERIC(10, 2) NOT NULL,
    date_debut DATE NOT NULL,
    date_fin DATE,  -- NULL = prix actuel (toujours valide)
    -- Garantit qu'il n'y a qu'un seul prix à une date donnée
    UNIQUE (produit_id, PERIOD(date_debut, COALESCE(date_fin, 'infinity'::date))) WITHOUT OVERLAPS,
    CHECK (date_fin IS NULL OR date_fin > date_debut)
);
```

**Astuce** : Utilisation de `COALESCE(date_fin, 'infinity')` pour gérer les périodes ouvertes (prix actuel sans date de fin).

```sql
-- ✅ Prix initial
INSERT INTO historique_prix (produit_id, prix, date_debut, date_fin)
VALUES (100, 19.99, '2024-01-01', '2024-06-30');

-- ✅ Nouveau prix à partir du 1er juillet
INSERT INTO historique_prix (produit_id, prix, date_debut, date_fin)
VALUES (100, 24.99, '2024-07-01', '2024-12-31');

-- ✅ Prix actuel (ouvert)
INSERT INTO historique_prix (produit_id, prix, date_debut, date_fin)
VALUES (100, 29.99, '2025-01-01', NULL);

-- ❌ ÉCHEC : Chevauchement avec le prix du 1er juillet
INSERT INTO historique_prix (produit_id, prix, date_debut, date_fin)
VALUES (100, 22.00, '2024-06-15', '2024-07-15');
```

### 2. Contrats d'Emploi Sans Chevauchement

Un employé ne peut pas avoir plusieurs contrats actifs simultanément.

```sql
CREATE TABLE contrats_emploi (
    id SERIAL PRIMARY KEY,
    employe_id INTEGER NOT NULL,
    type_contrat VARCHAR(50) NOT NULL,  -- CDI, CDD, Stage, etc.
    date_debut DATE NOT NULL,
    date_fin DATE,  -- NULL pour les CDI en cours
    salaire NUMERIC(10, 2) NOT NULL,
    -- Un employé ne peut avoir qu'un seul contrat actif à la fois
    UNIQUE (employe_id, PERIOD(date_debut, COALESCE(date_fin, 'infinity'::date))) WITHOUT OVERLAPS,
    CHECK (date_fin IS NULL OR date_fin > date_debut)
);
```

```sql
-- ✅ Premier contrat CDD
INSERT INTO contrats_emploi (employe_id, type_contrat, date_debut, date_fin, salaire)
VALUES (1, 'CDD', '2024-01-01', '2024-06-30', 2500.00);

-- ✅ CDI à partir du 1er juillet (pas de chevauchement)
INSERT INTO contrats_emploi (employe_id, type_contrat, date_debut, date_fin, salaire)
VALUES (1, 'CDI', '2024-07-01', NULL, 2800.00);

-- ❌ ÉCHEC : Impossible de créer un contrat qui chevauche le CDI
INSERT INTO contrats_emploi (employe_id, type_contrat, date_debut, date_fin, salaire)
VALUES (1, 'Stage', '2024-08-01', '2024-12-31', 1500.00);
```

### 3. Locations de Véhicules

Une voiture ne peut être louée par plusieurs clients en même temps.

```sql
CREATE TABLE locations_vehicules (
    id SERIAL PRIMARY KEY,
    vehicule_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    date_heure_debut TIMESTAMP NOT NULL,
    date_heure_fin TIMESTAMP NOT NULL,
    prix_total NUMERIC(10, 2) NOT NULL,
    -- Un véhicule ne peut être loué par deux clients en même temps
    UNIQUE (vehicule_id, PERIOD(date_heure_debut, date_heure_fin)) WITHOUT OVERLAPS,
    CHECK (date_heure_fin > date_heure_debut)
);
```

```sql
-- ✅ Location du véhicule 42 du 10 au 15 janvier
INSERT INTO locations_vehicules (vehicule_id, client_id, date_heure_debut, date_heure_fin, prix_total)
VALUES (42, 100, '2025-01-10 09:00', '2025-01-15 18:00', 350.00);

-- ✅ Location du même véhicule après la première location
INSERT INTO locations_vehicules (vehicule_id, client_id, date_heure_debut, date_heure_fin, prix_total)
VALUES (42, 101, '2025-01-15 18:00', '2025-01-20 18:00', 400.00);

-- ❌ ÉCHEC : Chevauchement avec la première location
INSERT INTO locations_vehicules (vehicule_id, client_id, date_heure_debut, date_heure_fin, prix_total)
VALUES (42, 102, '2025-01-12 09:00', '2025-01-17 18:00', 450.00);
```

### 4. Promotions Sans Chevauchement

Une promotion pour un produit ne peut pas avoir plusieurs périodes actives qui se chevauchent.

```sql
CREATE TABLE promotions (
    id SERIAL PRIMARY KEY,
    produit_id INTEGER NOT NULL,
    code_promo VARCHAR(50) NOT NULL,
    pourcentage_reduction INTEGER NOT NULL,
    date_debut DATE NOT NULL,
    date_fin DATE NOT NULL,
    -- Une seule promotion active à la fois pour un produit
    UNIQUE (produit_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS,
    CHECK (date_fin >= date_debut),
    CHECK (pourcentage_reduction > 0 AND pourcentage_reduction <= 100)
);
```

```sql
-- ✅ Promotion de janvier
INSERT INTO promotions (produit_id, code_promo, pourcentage_reduction, date_debut, date_fin)
VALUES (200, 'WINTER2025', 20, '2025-01-01', '2025-01-31');

-- ✅ Promotion de février (pas de chevauchement)
INSERT INTO promotions (produit_id, code_promo, pourcentage_reduction, date_debut, date_fin)
VALUES (200, 'SPRING2025', 15, '2025-02-01', '2025-02-28');

-- ❌ ÉCHEC : Chevauchement avec la promotion de janvier
INSERT INTO promotions (produit_id, code_promo, pourcentage_reduction, date_debut, date_fin)
VALUES (200, 'MEGA2025', 30, '2025-01-15', '2025-02-15');
```

---

## Contraintes Temporelles Multi-Colonnes

Vous pouvez combiner **plusieurs colonnes d'identification** avec une période temporelle.

### Exemple : Réservations de Salles par Bâtiment

```sql
CREATE TABLE reservations_salles_batiment (
    id SERIAL PRIMARY KEY,
    batiment_id INTEGER NOT NULL,
    salle_id INTEGER NOT NULL,
    utilisateur_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    -- Unicité sur la combinaison (batiment_id, salle_id) + période
    UNIQUE (batiment_id, salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS,
    CHECK (date_fin > date_debut)
);
```

**Comportement** : Une salle dans un bâtiment donné ne peut être réservée qu'une seule fois par période, mais la même salle dans un autre bâtiment peut l'être.

```sql
-- ✅ Réservation salle 10 du bâtiment A
INSERT INTO reservations_salles_batiment
    (batiment_id, salle_id, utilisateur_id, date_debut, date_fin)
VALUES (1, 10, 100, '2025-01-10 14:00', '2025-01-10 16:00');

-- ✅ Réservation salle 10 du bâtiment B (même salle, autre bâtiment)
INSERT INTO reservations_salles_batiment
    (batiment_id, salle_id, utilisateur_id, date_debut, date_fin)
VALUES (2, 10, 101, '2025-01-10 14:00', '2025-01-10 16:00');

-- ❌ ÉCHEC : Chevauchement pour la salle 10 du bâtiment A
INSERT INTO reservations_salles_batiment
    (batiment_id, salle_id, utilisateur_id, date_debut, date_fin)
VALUES (1, 10, 102, '2025-01-10 15:00', '2025-01-10 17:00');
```

---

## Contraintes Temporelles et Primary Keys

Vous pouvez également définir une **clé primaire temporelle**, garantissant l'unicité absolue sur une période.

### Syntaxe

```sql
CREATE TABLE reservations_primaire (
    salle_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    utilisateur_id INTEGER NOT NULL,
    -- Clé primaire temporelle
    PRIMARY KEY (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS,
    CHECK (date_fin > date_debut)
);
```

**Différence avec UNIQUE** :
- `PRIMARY KEY` : Identifie uniquement chaque "tranche temporelle"
- Pas besoin d'un `id SERIAL` séparé si la clé temporelle suffit

```sql
-- ✅ Insertion valide
INSERT INTO reservations_primaire
VALUES (1, '2025-01-10 14:00', '2025-01-10 16:00', 100);

-- ✅ Autre période pour la même salle
INSERT INTO reservations_primaire
VALUES (1, '2025-01-10 16:00', '2025-01-10 18:00', 101);

-- ❌ ÉCHEC : Duplication de la clé primaire temporelle
INSERT INTO reservations_primaire
VALUES (1, '2025-01-10 14:00', '2025-01-10 16:00', 102);
```

---

## Intégration avec les Foreign Keys

Les contraintes temporelles peuvent être combinées avec des **clés étrangères** pour modéliser des relations complexes.

### Exemple : Réservations avec Validation de Disponibilité

```sql
-- Table des salles
CREATE TABLE salles (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    capacite INTEGER NOT NULL
);

-- Table des périodes de maintenance (la salle est indisponible)
CREATE TABLE maintenance_salles (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL REFERENCES salles(id),
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    raison TEXT,
    -- Pas de chevauchement de maintenances
    UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS,
    CHECK (date_fin > date_debut)
);

-- Table des réservations
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL REFERENCES salles(id),
    utilisateur_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    -- Pas de chevauchement de réservations
    UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS,
    CHECK (date_fin > date_debut)
);
```

**Limitation** : PostgreSQL 18 ne peut pas (encore) vérifier automatiquement qu'une réservation ne chevauche pas une période de maintenance. Cela nécessiterait une logique applicative ou des triggers supplémentaires.

---

## Performance et Indexation

### Index Automatiques

PostgreSQL 18 crée automatiquement un **index spécialisé** pour les contraintes temporelles, optimisé pour détecter les chevauchements.

```sql
-- Après création de la table avec contrainte temporelle
\d reservations_salles

-- Vous verrez un index nommé automatiquement, par exemple :
-- "reservations_salles_salle_id_date_debut_date_fin_excl" EXCLUDE USING gist (...)
```

### Type d'Index : GIST (Generalized Search Tree)

Les contraintes temporelles utilisent des **index GIST**, particulièrement efficaces pour :
- Les requêtes de chevauchement (`&&`)
- Les requêtes spatiales et temporelles
- Les recherches sur des ranges

### Requêtes Optimisées

```sql
-- Rechercher toutes les réservations d'une salle pour une période donnée
EXPLAIN ANALYZE
SELECT * FROM reservations_salles
WHERE salle_id = 1
  AND tsrange(date_debut, date_fin) && '[2025-01-10 12:00, 2025-01-10 18:00)';

-- L'index GIST sera utilisé automatiquement
```

### Recommandation : Index Supplémentaires

Pour optimiser les requêtes fréquentes, ajoutez des index B-Tree classiques :

```sql
-- Index sur salle_id pour les recherches par salle
CREATE INDEX idx_reservations_salle_id ON reservations_salles(salle_id);

-- Index sur utilisateur_id pour les recherches par utilisateur
CREATE INDEX idx_reservations_utilisateur_id ON reservations_salles(utilisateur_id);

-- Index sur date_debut pour les recherches par date
CREATE INDEX idx_reservations_date_debut ON reservations_salles(date_debut);
```

---

## Migration depuis les Anciennes Versions

### Avant PostgreSQL 18 : Contraintes d'Exclusion

Si vous aviez utilisé des contraintes d'exclusion avec `btree_gist` :

```sql
-- Ancienne méthode (PostgreSQL < 18)
CREATE EXTENSION btree_gist;

CREATE TABLE reservations_old (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    EXCLUDE USING gist (
        salle_id WITH =,
        tsrange(date_debut, date_fin) WITH &&
    )
);
```

### Nouvelle Syntaxe PostgreSQL 18

```sql
-- Nouvelle méthode (PostgreSQL 18+)
CREATE TABLE reservations_new (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS
);
```

**Avantages** :
- ✅ Plus lisible et intuitive
- ✅ Standard SQL:2011 (Temporal Features)
- ✅ Pas besoin d'extension externe
- ✅ Meilleures performances
- ✅ Meilleur support par les ORM et outils

### Processus de Migration

```sql
-- 1. Créer la nouvelle table avec la syntaxe PG18
CREATE TABLE reservations_new (
    id SERIAL PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    utilisateur_id INTEGER NOT NULL,
    UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS
);

-- 2. Copier les données
INSERT INTO reservations_new
SELECT * FROM reservations_old;

-- 3. Renommer les tables
BEGIN;
ALTER TABLE reservations_old RENAME TO reservations_old_backup;
ALTER TABLE reservations_new RENAME TO reservations;
COMMIT;

-- 4. Vérifier et supprimer l'ancienne table
DROP TABLE reservations_old_backup;
```

---

## Limitations et Considérations

### 1. Types de Données Supportés

Les contraintes temporelles PostgreSQL 18 fonctionnent avec :
- ✅ `TIMESTAMP` / `TIMESTAMPTZ`
- ✅ `DATE`
- ✅ Types Range (`tsrange`, `tstzrange`, `daterange`)

### 2. NULL dans les Périodes

Si `date_debut` ou `date_fin` est `NULL`, la contrainte **ne s'applique pas** pour cette ligne.

```sql
-- ✅ Ces deux insertions sont acceptées (NULL ignore la contrainte)
INSERT INTO reservations (salle_id, date_debut, date_fin)
VALUES (1, NULL, NULL);

INSERT INTO reservations (salle_id, date_debut, date_fin)
VALUES (1, NULL, NULL);
```

**Solution** : Utilisez `NOT NULL` sur les colonnes de période si vous voulez toujours valider :

```sql
CREATE TABLE reservations (
    salle_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,  -- Obligatoire
    date_fin TIMESTAMP NOT NULL,    -- Obligatoire
    UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS
);
```

### 3. Périodes Ouvertes (Sans Fin)

Pour gérer des périodes ouvertes (ex : contrat en cours), utilisez `COALESCE` :

```sql
UNIQUE (employe_id, PERIOD(date_debut, COALESCE(date_fin, 'infinity'::date))) WITHOUT OVERLAPS
```

### 4. Modification de Périodes

Mettre à jour une période peut déclencher une violation si cela crée un chevauchement :

```sql
-- Réservation initiale
INSERT INTO reservations (salle_id, date_debut, date_fin)
VALUES (1, '2025-01-10 14:00', '2025-01-10 16:00');

-- Autre réservation
INSERT INTO reservations (salle_id, date_debut, date_fin)
VALUES (1, '2025-01-10 16:00', '2025-01-10 18:00');

-- ❌ ÉCHEC : Étendre la première réservation crée un chevauchement
UPDATE reservations
SET date_fin = '2025-01-10 17:00'
WHERE salle_id = 1 AND date_debut = '2025-01-10 14:00';
```

**Solution** : Vérifiez les chevauchements avant de modifier, ou utilisez des transactions.

### 5. Performance sur Grandes Tables

Sur des tables avec **millions de lignes**, les contraintes temporelles peuvent ralentir les insertions/modifications. Optimisations :

1. **Partitionnement** : Partitionnez par période (par mois, par année)
2. **Archivage** : Déplacez les anciennes réservations vers une table d'archive
3. **Index sélectifs** : Utilisez des index partiels si pertinent

```sql
-- Index partiel : Seulement les réservations futures
CREATE INDEX idx_reservations_futures
ON reservations(salle_id, date_debut)
WHERE date_fin >= CURRENT_TIMESTAMP;
```

---

## Bonnes Pratiques

### 1. Nommez vos Contraintes

```sql
CREATE TABLE reservations (
    salle_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    CONSTRAINT uq_reservations_salle_periode
        UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS
);
```

**Avantage** : Messages d'erreur explicites.

### 2. Combinez avec des CHECK pour la Cohérence

```sql
CREATE TABLE reservations (
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    -- Vérification de base : date_fin après date_debut
    CHECK (date_fin > date_debut),
    -- Contrainte temporelle
    UNIQUE (salle_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS
);
```

### 3. Documentez vos Contraintes

```sql
COMMENT ON CONSTRAINT uq_reservations_salle_periode ON reservations
IS 'Empêche les réservations qui se chevauchent pour une même salle';
```

### 4. Utilisez des Transactions pour les Opérations Complexes

```sql
BEGIN;

-- Annuler une réservation existante
DELETE FROM reservations WHERE id = 42;

-- Créer deux nouvelles réservations dans le même créneau
INSERT INTO reservations (salle_id, date_debut, date_fin)
VALUES (1, '2025-01-10 14:00', '2025-01-10 15:30');

INSERT INTO reservations (salle_id, date_debut, date_fin)
VALUES (1, '2025-01-10 15:30', '2025-01-10 17:00');

COMMIT;
```

### 5. Testez Systématiquement

Écrivez des tests pour vérifier que vos contraintes fonctionnent :

```sql
-- Test 1 : Insertion normale
INSERT INTO reservations (salle_id, date_debut, date_fin)
VALUES (1, '2025-01-10 14:00', '2025-01-10 16:00');

-- Test 2 : Chevauchement (doit échouer)
DO $$
BEGIN
    INSERT INTO reservations (salle_id, date_debut, date_fin)
    VALUES (1, '2025-01-10 15:00', '2025-01-10 17:00');
    RAISE EXCEPTION 'Test échoué : Chevauchement non détecté';
EXCEPTION WHEN unique_violation THEN
    RAISE NOTICE 'Test réussi : Chevauchement correctement bloqué';
END $$;
```

---

## Comparaison : Avant vs Après PostgreSQL 18

| Aspect | PostgreSQL < 18 | PostgreSQL 18 |
|--------|----------------|---------------|
| **Syntaxe** | Complexe (EXCLUDE USING GIST) | Simple (UNIQUE ... WITHOUT OVERLAPS) |
| **Extension requise** | Oui (btree_gist) | Non (natif) |
| **Lisibilité** | Faible | Excellente |
| **Standard SQL** | Non | Oui (SQL:2011) |
| **Performance** | Bonne | Meilleure (optimisé) |
| **Support ORM** | Limité | En cours d'adoption |
| **Maintenance** | Complexe (triggers) | Simple (déclaratif) |

---

## Exemples d'Utilisation Avancée

### 1. Gestion de Versions de Documents

Gérer les versions d'un document avec des périodes de validité.

```sql
CREATE TABLE versions_documents (
    document_id INTEGER NOT NULL,
    version INTEGER NOT NULL,
    contenu TEXT NOT NULL,
    auteur_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    date_fin TIMESTAMP,
    -- Une seule version active à la fois pour un document
    UNIQUE (document_id, PERIOD(date_debut, COALESCE(date_fin, 'infinity'::timestamp))) WITHOUT OVERLAPS
);
```

### 2. Créneaux de Disponibilité de Personnel

Gérer les disponibilités d'employés sans chevauchement.

```sql
CREATE TABLE disponibilites_employes (
    employe_id INTEGER NOT NULL,
    date_debut TIMESTAMP NOT NULL,
    date_fin TIMESTAMP NOT NULL,
    type_disponibilite VARCHAR(20) NOT NULL,  -- 'disponible', 'conges', 'maladie'
    -- Pas de chevauchement de périodes pour un employé
    UNIQUE (employe_id, PERIOD(date_debut, date_fin)) WITHOUT OVERLAPS,
    CHECK (date_fin > date_debut)
);
```

### 3. Tarification Dynamique avec Historique

```sql
CREATE TABLE tarifs_zones (
    zone_id INTEGER NOT NULL,
    type_vehicule VARCHAR(50) NOT NULL,
    tarif_horaire NUMERIC(10, 2) NOT NULL,
    date_debut DATE NOT NULL,
    date_fin DATE,
    -- Un seul tarif valide par zone et type de véhicule à un moment donné
    UNIQUE (zone_id, type_vehicule, PERIOD(date_debut, COALESCE(date_fin, 'infinity'::date))) WITHOUT OVERLAPS,
    CHECK (tarif_horaire > 0)
);
```

---

## Résumé

### Points Clés

1. **Nouveauté majeure** : Les contraintes temporelles sont natives dans PostgreSQL 18
2. **Syntaxe simple** : `UNIQUE (col, PERIOD(debut, fin)) WITHOUT OVERLAPS`
3. **Gestion automatique** : Détection et blocage des chevauchements
4. **Performance optimisée** : Index GIST automatiques
5. **Standard SQL** : Conforme à SQL:2011 (Temporal Features)

### Avantages

- ✅ **Simplicité** : Plus besoin d'extensions ou de triggers complexes
- ✅ **Fiabilité** : Garantie au niveau de la base de données
- ✅ **Performance** : Optimisé nativement
- ✅ **Maintenabilité** : Code déclaratif et lisible
- ✅ **Portabilité** : Standard SQL (compatible avec d'autres SGBD modernes)

### Cas d'Usage Idéaux

- 📅 Réservations (salles, véhicules, équipements)
- 💰 Historiques de prix et tarifs
- 📝 Contrats et périodes de validité
- 🎟️ Promotions temporelles
- 👤 Disponibilités de personnel
- 📚 Versions de documents
- 🏢 Locations et baux

---

## Conclusion

Les **contraintes temporelles** de PostgreSQL 18 représentent une avancée majeure pour la gestion des données temporelles. Ce qui nécessitait auparavant des extensions complexes, des triggers personnalisés, ou une logique applicative lourde est désormais géré de manière **native, simple et performante**.

Cette fonctionnalité transforme PostgreSQL en un outil encore plus puissant pour les applications modernes qui manipulent des périodes, des historiques et des réservations.

**Recommandation** : Si votre application gère des périodes temporelles avec des risques de chevauchement, **migrez vers PostgreSQL 18** et adoptez cette syntaxe moderne. Vous gagnerez en simplicité, en fiabilité et en performance.

---

**Prochain sujet** : Dans la section suivante (7.3), nous explorerons la **théorie des ensembles et les jointures**, fondement essentiel de SQL pour relier les tables entre elles.

⏭️ [Théorie des ensembles et jointures : Produit cartésien et sélection](/07-relations-et-jointures/03-theorie-des-ensembles.md)
