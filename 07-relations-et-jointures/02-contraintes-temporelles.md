🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2. Nouveauté PostgreSQL 18 : contraintes temporelles (*Temporal Constraints*)

## Introduction

Les **contraintes temporelles** (*temporal constraints*) sont une fonctionnalité majeure introduite dans **PostgreSQL 18** (septembre 2025). Elles permettent de gérer des **périodes de validité** et d'empêcher les **chevauchements temporels** directement au niveau de la base de données, via deux mécanismes :

1. **`UNIQUE` / `PRIMARY KEY … WITHOUT OVERLAPS`** : interdit le chevauchement de périodes pour une même valeur d'identifiant.
2. **`FOREIGN KEY … PERIOD`** : une ligne enfant doit être *couverte* par une (ou plusieurs) ligne(s) parent(s) sur la dimension temporelle.

Ces deux clauses font partie des **fonctionnalités temporelles du standard SQL:2011** et étaient jusqu'ici simulées via des contraintes d'exclusion (`EXCLUDE USING GIST`) et l'extension `btree_gist`.

### Pourquoi cette nouveauté ?

Avant PostgreSQL 18, gérer les chevauchements de périodes nécessitait :
- Des **contraintes d'exclusion** (`EXCLUDE USING gist (…)`) avec une syntaxe peu intuitive ;
- Des **triggers** pour les règles plus complexes ;
- De la **logique applicative** lourde et sujette aux *race conditions*.

Avec PostgreSQL 18, la gestion temporelle devient **déclarative, lisible et conforme au standard**.

---

## 1. Le problème : gérer les périodes et leurs chevauchements

### Cas d'usage typiques

| Scénario | Règle métier | Risque sans contrainte |
|----------|-------------|-----------------------|
| Réservations de salles | Une salle, un seul occupant à la fois | Double réservation |
| Historique de prix | Un seul prix actif à un instant `t` | Ambiguïté de tarification |
| Contrats d'emploi | Un contrat actif par employé à la fois | Cumul d'emploi non détecté |
| Locations de véhicules | Pas deux locataires en même temps | Conflit physique |
| Promotions | Pas de superposition pour un même produit | Réductions cumulées par erreur |

### Le défi : pourquoi `CHECK` ne suffit pas

Une contrainte `CHECK` ne peut interroger **que la ligne en cours d'insertion** :

```sql
CREATE TABLE reservations_naif (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    debut TIMESTAMP NOT NULL,
    fin TIMESTAMP NOT NULL,
    CHECK (fin > debut)  -- ✅ OK : cohérence interne
);

-- ❌ ÉCHEC SILENCIEUX : ces deux lignes se chevauchent mais sont acceptées
INSERT INTO reservations_naif (salle_id, debut, fin) VALUES
    (1, '2025-01-10 14:00', '2025-01-10 16:00'),
    (1, '2025-01-10 15:00', '2025-01-10 17:00');  -- pourtant invalide métier !
```

Le `CHECK` ne peut pas voir les autres lignes : il faut une contrainte qui compare **plusieurs lignes** entre elles.

### Solution pré-PG 18 : `EXCLUDE USING gist`

Avant PG 18, on utilisait une **contrainte d'exclusion** sur un index GiST, en construisant un range à la volée :

```sql
-- Avant PG 18 (toujours valide en PG 18 mais plus verbeux)
CREATE EXTENSION btree_gist;

CREATE TABLE reservations_legacy (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    debut TIMESTAMP NOT NULL,
    fin TIMESTAMP NOT NULL,
    -- "Pas deux lignes où salle_id sont égaux ET les périodes se chevauchent"
    EXCLUDE USING gist (
        salle_id WITH =,
        tsrange(debut, fin, '[)') WITH &&
    )
);
```

Cela fonctionnait, mais la syntaxe `EXCLUDE USING gist (col WITH op, …)` est cryptique pour les nouveaux venus, et l'on doit construire le range dans la déclaration.

---

## 2. Rappel : les types `range` de PostgreSQL

Les contraintes temporelles PG 18 s'appuient sur les **types range** (existants depuis PostgreSQL 9.2). Ils représentent un **intervalle borné** d'une valeur scalaire.

| Type | Sous-jacent | Exemple littéral |
|------|-------------|------------------|
| `int4range` | `INTEGER` | `'[1,10)'::int4range` |
| `int8range` | `BIGINT` | `'[1000,2000)'::int8range` |
| `numrange` | `NUMERIC` | `'[0.0,100.0]'::numrange` |
| `tsrange` | `TIMESTAMP` | `'[2025-01-01,2025-12-31)'::tsrange` |
| `tstzrange` | `TIMESTAMPTZ` | `'[2025-01-01 00:00+00,2025-12-31 23:59+00)'::tstzrange` |
| `daterange` | `DATE` | `'[2025-01-01,2025-12-31]'::daterange` |

### Notation des bornes

- `[a, b]` : inclusif des deux côtés
- `[a, b)` : inclusif à gauche, exclusif à droite (**la convention recommandée pour des périodes contiguës**)
- `(a, b)` : exclusif des deux côtés
- `[a, infinity)` : ouvert à droite ; `(-infinity, b]` ouvert à gauche

### Construction et opérateurs utiles

```sql
-- Construction
SELECT tsrange('2025-01-10 14:00', '2025-01-10 16:00', '[)');
-- → ["2025-01-10 14:00:00","2025-01-10 16:00:00")

-- Opérateurs
SELECT tsrange('2025-01-10 14:00', '2025-01-10 16:00', '[)')
    && tsrange('2025-01-10 15:00', '2025-01-10 17:00', '[)');
-- → TRUE (chevauchement)

SELECT tsrange('2025-01-10 14:00', '2025-01-10 16:00', '[)')
    && tsrange('2025-01-10 16:00', '2025-01-10 18:00', '[)');
-- → FALSE (bornes adjacentes : la borne 16:00 est exclue du premier)
```

| Opérateur | Sens | Exemple |
|-----------|------|---------|
| `&&` | Chevauchement | `r1 && r2` |
| `@>` | Contient | `'[1,10)' @> '[2,5)'` |
| `<@` | Est contenu dans | `'[2,5)' <@ '[1,10)'` |
| `-\|-` | Adjacent | `'[1,5)' -\|- '[5,10)'` |
| `<<` / `>>` | Strictement à gauche/droite | `'[1,5)' << '[10,15)'` |

**Pour les contraintes temporelles, l'opérateur clé est `&&`** : la nouvelle syntaxe `WITHOUT OVERLAPS` l'utilise sous le capot.

---

## 3. La syntaxe officielle PG 18 : `WITHOUT OVERLAPS`

### Règle de base

```
UNIQUE | PRIMARY KEY ( col_1, col_2, …, col_range WITHOUT OVERLAPS )
```

**Points clés** :
- La colonne avec `WITHOUT OVERLAPS` doit être **la dernière** de la liste.
- Elle doit être d'un type **`range`** ou **`multirange`**.
- Les autres colonnes sont comparées pour **égalité** (comme dans un `UNIQUE` classique).
- L'index sous-jacent est un **GiST**, pas un B-tree.
- La contrainte doit comporter **au moins deux colonnes** : au moins une colonne d'égalité **et** la colonne range. Une contrainte `WITHOUT OVERLAPS` portant sur le seul range est refusée (`constraint using WITHOUT OVERLAPS needs at least two columns`). Pour interdire tout chevauchement *global*, sans colonne de partitionnement, il faut revenir à `EXCLUDE USING gist (periode WITH &&)`.

### Premier exemple — réservations de salles

```sql
-- ⚠️ btree_gist est nécessaire quand une colonne non-range (ici INTEGER)
-- est combinée avec une colonne range dans la contrainte.
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE reservations_salles (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    utilisateur_id INTEGER NOT NULL,
    -- Une seule colonne range pour la période, plutôt que deux timestamps
    periode tstzrange NOT NULL,
    description TEXT,
    -- La contrainte temporelle : pas de chevauchement par salle
    CONSTRAINT pas_de_chevauchement
        UNIQUE (salle_id, periode WITHOUT OVERLAPS),
    -- Cohérence interne : pas de range vide
    CHECK (NOT isempty(periode))
);
```

### Comportement

```sql
-- ✅ Première réservation
INSERT INTO reservations_salles (salle_id, utilisateur_id, periode) VALUES
    (1, 100, tstzrange('2025-01-10 14:00+01', '2025-01-10 16:00+01', '[)'));

-- ✅ Salle différente, même créneau : OK
INSERT INTO reservations_salles (salle_id, utilisateur_id, periode) VALUES
    (2, 101, tstzrange('2025-01-10 14:00+01', '2025-01-10 16:00+01', '[)'));

-- ✅ Même salle, créneau adjacent (la borne droite est exclue) : OK
INSERT INTO reservations_salles (salle_id, utilisateur_id, periode) VALUES
    (1, 102, tstzrange('2025-01-10 16:00+01', '2025-01-10 18:00+01', '[)'));

-- ❌ Même salle, créneau qui chevauche : REJETÉ
INSERT INTO reservations_salles (salle_id, utilisateur_id, periode) VALUES
    (1, 103, tstzrange('2025-01-10 15:00+01', '2025-01-10 17:00+01', '[)'));
-- ERROR: conflicting key value violates exclusion constraint "pas_de_chevauchement"
```

PostgreSQL détecte automatiquement le chevauchement et rejette l'insertion.

> 💡 **Pourquoi `btree_gist` ?** Les contraintes `WITHOUT OVERLAPS` s'appuient sur un index **GiST** (les ranges ne sont pas indexables en B-tree). Or `salle_id` est un `INTEGER` : par défaut, GiST ne connaît pas l'opérateur d'égalité sur les types scalaires. L'extension `btree_gist` ajoute le support des opérateurs B-tree (`=`, `<`, etc.) aux index GiST, ce qui permet de mélanger des colonnes scalaires (`salle_id`) avec des colonnes range (`periode`) dans la même contrainte.  
>  
> Si tous les composants de la contrainte sont déjà des ranges, `btree_gist` n'est pas nécessaire.

### `PRIMARY KEY WITHOUT OVERLAPS`

La syntaxe `WITHOUT OVERLAPS` s'utilise aussi sur une **clé primaire** :

```sql
CREATE TABLE tarifs (
    produit_id INTEGER NOT NULL,
    prix NUMERIC(10, 2) NOT NULL,
    validite daterange NOT NULL,
    PRIMARY KEY (produit_id, validite WITHOUT OVERLAPS)
);
```

L'avantage par rapport à `UNIQUE` : pas besoin de définir une colonne `id` artificielle. La paire `(produit_id, validite)` identifie de manière unique chaque tranche de tarif.

---

## 4. Cas d'usage classiques

### a) Historique de prix avec période ouverte

Pour un produit, un seul prix doit être actif à un moment donné. La période courante n'a pas de date de fin (« active jusqu'à nouvel ordre »).

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE historique_prix (
    produit_id INTEGER NOT NULL,
    prix NUMERIC(10, 2) NOT NULL CHECK (prix > 0),
    -- daterange autorise des bornes infinies via 'infinity'
    validite daterange NOT NULL,
    PRIMARY KEY (produit_id, validite WITHOUT OVERLAPS)
);

-- ✅ Trois périodes successives non chevauchantes
INSERT INTO historique_prix VALUES
    (100, 19.99, daterange('2024-01-01', '2024-07-01', '[)')),
    (100, 24.99, daterange('2024-07-01', '2025-01-01', '[)')),
    (100, 29.99, daterange('2025-01-01', NULL, '[)'));  -- borne droite ouverte = infini

-- ❌ Une période qui chevauche est rejetée
INSERT INTO historique_prix VALUES
    (100, 22.00, daterange('2024-06-15', '2024-07-15', '[)'));
```

> 📌 **Astuce : `NULL` comme borne signifie « infini »** dans la construction `daterange(a, b, …)`. C'est plus propre que d'écrire `'infinity'::date`.

### b) Contrats d'emploi sans cumul

```sql
CREATE TABLE contrats (
    employe_id INTEGER NOT NULL,
    type_contrat VARCHAR(20) NOT NULL,
    salaire NUMERIC(10, 2) NOT NULL,
    periode daterange NOT NULL,
    PRIMARY KEY (employe_id, periode WITHOUT OVERLAPS)
);

-- ✅ Un CDD puis un CDI : non chevauchants
INSERT INTO contrats VALUES
    (1, 'CDD', 2500.00, daterange('2024-01-01', '2024-07-01', '[)')),
    (1, 'CDI', 2800.00, daterange('2024-07-01', NULL, '[)'));

-- ❌ Stage qui chevauche le CDI : refusé
INSERT INTO contrats VALUES
    (1, 'Stage', 1500.00, daterange('2024-08-01', '2024-12-31', '[)'));
```

### c) Locations de véhicules

```sql
CREATE TABLE locations (
    vehicule_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    periode tstzrange NOT NULL,
    prix_total NUMERIC(10, 2) NOT NULL,
    PRIMARY KEY (vehicule_id, periode WITHOUT OVERLAPS)
);

INSERT INTO locations VALUES
    (42, 100, tstzrange('2025-01-10 09:00+01', '2025-01-15 18:00+01', '[)'), 350.00),
    (42, 101, tstzrange('2025-01-15 18:00+01', '2025-01-20 18:00+01', '[)'), 400.00);

-- ❌ Tentative de location en plein milieu de la première
INSERT INTO locations VALUES
    (42, 102, tstzrange('2025-01-12 09:00+01', '2025-01-17 18:00+01', '[)'), 450.00);
```

---

## 5. `FOREIGN KEY … PERIOD` : clés étrangères temporelles

La clause `PERIOD` permet à une `FOREIGN KEY` de référencer non pas une ligne unique, mais **une couverture temporelle**. Le but : garantir que l'enfant n'existe que sur des intervalles où le parent existe aussi.

### Exemple : abonnements d'une salle (capacité)

Imaginons que chaque salle a une **capacité valide sur une période** (par exemple, elle est ouverte de 2024 à 2027), et qu'on veut interdire toute réservation hors de cette période.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- 1) Table parent : périodes de disponibilité de chaque salle
CREATE TABLE salle_capacite (
    salle_id INTEGER NOT NULL,
    capacite INTEGER NOT NULL,
    validite daterange NOT NULL,
    PRIMARY KEY (salle_id, validite WITHOUT OVERLAPS)
);

INSERT INTO salle_capacite VALUES
    (1, 30, daterange('2024-01-01', '2027-01-01', '[)'));

-- 2) Table enfant : réservations, avec FK temporelle
CREATE TABLE reservations_capacite (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    periode daterange NOT NULL,
    FOREIGN KEY (salle_id, PERIOD periode)
        REFERENCES salle_capacite (salle_id, PERIOD validite)
);

-- ✅ Réservation à l'intérieur de la période de disponibilité
INSERT INTO reservations_capacite (salle_id, periode) VALUES
    (1, daterange('2025-03-01', '2025-03-15', '[)'));

-- ❌ Réservation qui dépasse la période parent
INSERT INTO reservations_capacite (salle_id, periode) VALUES
    (1, daterange('2026-06-01', '2027-12-31', '[)'));
-- ERROR: insert or update on table "reservations_capacite" violates foreign key constraint
```

**Différence cruciale avec une FK classique** :
- Une FK classique vérifie que `(salle_id, validite)` **existe** dans la table parent.
- Une FK temporelle vérifie que `periode` de l'enfant est **entièrement couverte** par l'union des `validite` du parent ayant le même `salle_id`. Le parent peut donc avoir **plusieurs lignes** dont les périodes (combinées) couvrent celle de l'enfant.

### Limitations importantes des FK temporelles

| Action | Supportée sur FK temporelle ? |
|--------|-------------------------------|
| `ON DELETE NO ACTION` (défaut) | ✅ Oui |
| `ON DELETE RESTRICT` | ❌ Non |
| `ON DELETE CASCADE` | ❌ Non |
| `ON DELETE SET NULL` | ❌ Non |
| `ON DELETE SET DEFAULT` | ❌ Non |

> ⚠️ La même restriction vaut pour **`ON UPDATE`** : seul `NO ACTION` est accepté. Un `ON UPDATE CASCADE` (ou `RESTRICT`, `SET NULL`…) est rejeté avec `unsupported ON UPDATE action for foreign key constraint using PERIOD`.

Si vous avez besoin de cascades ou de comportements actifs sur les FK temporelles, il faut implémenter la logique vous-même (typiquement via un trigger ou côté application).

---

## 6. Détecter les chevauchements *a posteriori*

La contrainte `WITHOUT OVERLAPS` empêche d'écrire de mauvaises données ; mais sur une base héritée, il faut parfois en **identifier** :

```sql
-- Trouver les paires de réservations qui se chevauchent pour une même salle
SELECT a.id, b.id, a.salle_id, a.periode, b.periode  
FROM reservations_salles a  
JOIN reservations_salles b  
  ON a.salle_id = b.salle_id
 AND a.id < b.id            -- éviter (A,B) et (B,A)
 AND a.periode && b.periode -- l'opérateur de chevauchement
ORDER BY a.salle_id, a.periode;
```

Cette requête est utile pour faire le **nettoyage** d'une table avant d'y ajouter une contrainte `WITHOUT OVERLAPS`.

---

## 6 bis. Requêtes temporelles courantes

Une fois la table peuplée avec des colonnes de type range, plusieurs requêtes deviennent élégantes grâce aux opérateurs natifs.

### a) « Qui occupe la salle à l'instant T ? »

```sql
SELECT salle_id, utilisateur_id, periode  
FROM reservations_salles  
WHERE periode @> '2025-01-10 15:30:00+01'::timestamptz;  
```

L'opérateur `@>` (« contient ») lit littéralement : « la période contient cet instant ». Plus naturel et plus indexable que `'2025-01-10 15:30' BETWEEN debut AND fin`.

### b) « Qui occupe la salle pendant le créneau [14h, 16h) ? »

```sql
SELECT salle_id, utilisateur_id, periode  
FROM reservations_salles  
WHERE periode && tstzrange('2025-01-10 14:00+01', '2025-01-10 16:00+01', '[)');  
```

Avec **un seul opérateur `&&`**, on couvre tous les cas de chevauchement (la réservation commence avant et finit pendant, elle est entièrement incluse, elle englobe le créneau, etc.).

### c) « Trouver les trous dans le planning »

L'opérateur `-` sur les ranges donne la différence (et l'agrégation `range_agg` les unit). Pour trouver les périodes libres d'une salle entre 9h et 18h :

```sql
WITH journee AS (
    SELECT tstzrange('2025-01-10 09:00+01', '2025-01-10 18:00+01', '[)') AS plage
)
SELECT range_agg(occupe.periode) AS plages_occupees,
       -- range_agg renvoie un MULTIRANGE : on convertit la plage de référence
       -- en multirange pour pouvoir soustraire (un « range − multirange » n'existe pas)
       tstzmultirange((SELECT plage FROM journee)) - range_agg(occupe.periode) AS plages_libres
FROM reservations_salles occupe  
WHERE occupe.salle_id = 1  
  AND occupe.periode && (SELECT plage FROM journee);
```

> 📌 `range_agg` (PostgreSQL 14+) consolide plusieurs ranges en un **`multirange`**. Comme la soustraction veut deux opérandes de même nature, on convertit la plage de référence en multirange via `tstzmultirange(...)` : `multirange − multirange` donne les **plages libres**, sans calcul manuel avec des `LEAD`/`LAG`. *(Un `range − multirange` direct lèverait `operator does not exist`.)*

### d) « Quelle est l'occupation cumulée par jour ? »

```sql
SELECT
    date(lower(periode)) AS jour,
    salle_id,
    sum(upper(periode) - lower(periode)) AS duree_occupee
FROM reservations_salles  
WHERE lower(periode) >= '2025-01-01' AND lower(periode) < '2025-02-01'  
GROUP BY date(lower(periode)), salle_id  
ORDER BY jour, salle_id;  
```

`lower(range)` et `upper(range)` accèdent aux bornes du range pour calculer une durée.

---

## 7. Performance et indexation

### Index GiST automatique

`UNIQUE`/`PRIMARY KEY … WITHOUT OVERLAPS` crée automatiquement un **index GiST** (et non B-tree), adapté aux types range. C'est cet index qui implémente la vérification de chevauchement en temps logarithmique.

### Index complémentaires

Pour les **requêtes de lecture** ciblées par identifiant simple (sans dimension temporelle), un index B-tree classique reste pertinent :

```sql
-- Le GiST de la contrainte est efficace pour "chevauchements pour salle X",
-- mais pour "toutes les réservations de l'utilisateur Y", un B-tree classique
-- est utile.
CREATE INDEX idx_resa_utilisateur ON reservations_salles(utilisateur_id);
```

### Index partiel sur l'« actuel »

Si la quasi-totalité des requêtes ne porte que sur les périodes courantes ou futures, un index partiel réduit la taille de l'index :

```sql
-- ⚠️ Le prédicat d'un index partiel doit être IMMUTABLE : « now() » y est
--    REFUSÉ (« functions in index predicate must be marked IMMUTABLE »),
--    car c'est une fonction STABLE. On fige donc une date « charnière »
--    constante, qu'on actualise périodiquement en recréant l'index.
CREATE INDEX idx_resa_futures
    ON reservations_salles USING gist (salle_id, periode)
    WHERE upper(periode) > '2025-01-01'::timestamptz;
```

---

## 8. Migration depuis `EXCLUDE USING gist`

Si votre base utilise déjà des `EXCLUDE` avec deux colonnes timestamp, voici le plan de migration :

### a) Ancienne structure (PG ≤ 17)

```sql
CREATE EXTENSION btree_gist;

CREATE TABLE reservations_old (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    debut TIMESTAMPTZ NOT NULL,
    fin   TIMESTAMPTZ NOT NULL,
    EXCLUDE USING gist (
        salle_id WITH =,
        tstzrange(debut, fin, '[)') WITH &&
    )
);
```

### b) Nouvelle structure (PG 18+)

```sql
CREATE TABLE reservations_new (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    periode TSTZRANGE NOT NULL,
    UNIQUE (salle_id, periode WITHOUT OVERLAPS)
);
```

### c) Procédure de migration

```sql
BEGIN;

-- Créer la nouvelle table
CREATE TABLE reservations_new (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    salle_id INTEGER NOT NULL,
    periode TSTZRANGE NOT NULL,
    UNIQUE (salle_id, periode WITHOUT OVERLAPS)
);

-- Copier les données en convertissant deux timestamps en un range
INSERT INTO reservations_new (salle_id, periode)  
SELECT salle_id, tstzrange(debut, fin, '[)') FROM reservations_old;  

-- Bascule
ALTER TABLE reservations_old RENAME TO reservations_old_bak;  
ALTER TABLE reservations_new RENAME TO reservations_old;  -- on garde le nom métier  

COMMIT;

-- Plus tard, après validation :
-- DROP TABLE reservations_old_bak;
```

**Avantages de la nouvelle approche** :
- ✅ Syntaxe **déclarative** et lisible
- ✅ Conforme au **standard SQL:2011**
- ✅ Le range est **une donnée**, pas une construction à recalculer dans chaque requête de lecture
- ✅ Les **FK temporelles** deviennent possibles
- ✅ Mieux supporté par les ORM et outils de migration récents

**Inconvénients** :
- Une seule colonne `periode` au lieu de deux : si vos requêtes existantes interrogent `debut` ou `fin` directement, il faut les adapter (`lower(periode)`, `upper(periode)`).

---

## 9. Limitations et points d'attention

### a) Types acceptés

`WITHOUT OVERLAPS` exige une colonne **range** ou **multirange**. Vous ne pouvez **pas** écrire `… periode WITHOUT OVERLAPS` sur une colonne `TIMESTAMP` ou `INTEGER` : il faut une colonne de type `tstzrange`, `daterange`, `int8range`, etc.

### b) `NULL` dans la colonne range

Si la colonne range est `NULL`, la ligne **n'est jamais en conflit** avec une autre (puisqu'il n'y a pas de période à comparer). Pour interdire ce cas :

```sql
periode TSTZRANGE NOT NULL  -- ← déclarer NOT NULL
```

C'est la valeur par défaut recommandée pour une table de réservation.

### c) Ranges vides

En **PostgreSQL 18, un range vide (`'empty'::tstzrange`) est automatiquement rejeté** dès qu'il vise une colonne couverte par `WITHOUT OVERLAPS` :

```
ERROR:  empty WITHOUT OVERLAPS value found in column "periode" in relation "reservations_salles"
```

Aucune protection supplémentaire n'est donc requise pour ce cas précis. Un `CHECK (NOT isempty(periode))` reste néanmoins recommandé comme **garde-fou explicite** : il documente l'intention, renvoie un message d'erreur plus parlant, et continue de protéger la table si la contrainte `WITHOUT OVERLAPS` venait à être retirée :

```sql
CHECK (NOT isempty(periode))
```

### d) `RESTRICT`/`CASCADE` non supportés sur FK temporelles

Comme vu plus haut, seul `ON DELETE NO ACTION` (le défaut) est autorisé sur les `FOREIGN KEY … PERIOD`. Si vous avez besoin de cascades temporelles, utilisez un trigger applicatif.

### e) Index lourd sur très grandes tables

Le GiST est plus coûteux à maintenir qu'un B-tree. Sur une table d'historique avec des millions de lignes, le partitionnement par période (mensuel, annuel) reste indispensable.

---

## 10. Comparatif : avant et après PG 18

| Aspect | `EXCLUDE USING gist` (avant) | `WITHOUT OVERLAPS` (PG 18) |
|--------|------------------------------|----------------------------|
| Syntaxe | `EXCLUDE USING gist (col WITH =, range WITH &&)` | `UNIQUE (col, range WITHOUT OVERLAPS)` |
| Extension `btree_gist` | Requise dès qu'on mélange scalaire et range | Idem (inchangé) |
| Standard SQL | Extension PostgreSQL | **SQL:2011** |
| Lisibilité | Faible | Excellente |
| `PRIMARY KEY` temporelle | Possible via index unique séparé | Directement avec `PRIMARY KEY … WITHOUT OVERLAPS` |
| `FOREIGN KEY` temporelle | Impossible nativement (triggers ad hoc) | `FOREIGN KEY … PERIOD` |
| Schéma stocké | Deux colonnes `debut`/`fin` souvent | Une colonne `range` |

---

## 11. Bonnes pratiques

1. **Stocker les périodes comme des ranges**, pas comme deux colonnes séparées. C'est plus naturel, et l'opérateur `&&` devient utilisable partout.
2. **Toujours rendre la colonne range `NOT NULL`** sur les tables où la période est intrinsèque (réservation, contrat, location).
3. **Exclure les ranges vides avec `CHECK (NOT isempty(periode))`** comme garde-fou explicite. *(Dans une contrainte `WITHOUT OVERLAPS`, PostgreSQL 18 rejette déjà nativement les ranges vides — voir la section 9c ; le `CHECK` documente l'intention et renvoie un message plus parlant.)*
4. **Préférer la convention `[)`** (inclusif à gauche, exclusif à droite) : elle rend les périodes contiguës (`[8h,10h)` puis `[10h,12h)`) sans chevauchement ni trou.
5. **Documenter la contrainte** :
   ```sql
   COMMENT ON CONSTRAINT pas_de_chevauchement ON reservations_salles
   IS 'Interdit deux réservations qui se chevauchent pour la même salle.';
   ```
6. **Combiner avec une `FOREIGN KEY` classique** vers la table des entités référencées (salle, employé) — la contrainte temporelle vérifie le chevauchement, mais pas l'existence de l'entité parente sans FK explicite.

---

## 12. Résumé

### Points clés

1. **PostgreSQL 18 standardise** la gestion des contraintes temporelles avec `WITHOUT OVERLAPS` et `PERIOD`.
2. **La syntaxe utilise une colonne de type range** (pas deux colonnes scalaires séparées).
3. `WITHOUT OVERLAPS` détecte les **chevauchements** ; `PERIOD` (sur FK) vérifie la **couverture**.
4. L'extension **`btree_gist`** reste requise dès qu'on mélange une colonne scalaire avec une colonne range dans la contrainte.
5. Les **actions référentielles** sur FK temporelles sont limitées à `NO ACTION`.

### Cas d'usage idéaux

- 📅 Réservations, plannings, créneaux
- 💰 Historiques de prix et tarifs
- 📝 Contrats et périodes de validité
- 🎟️ Promotions temporelles
- 👤 Disponibilités et absences
- 📚 Versions de documents avec dates de validité
- 🏢 Baux, locations, abonnements

### À retenir en une ligne

> `UNIQUE (entite_id, periode_range WITHOUT OVERLAPS)` remplace, en plus simple et plus standard, les contraintes d'exclusion historiques.

---

⏭️ [Théorie des ensembles et jointures : produit cartésien et sélection](/07-relations-et-jointures/03-theorie-des-ensembles.md)
