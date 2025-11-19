🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4. La clause RETURNING : Une spécificité puissante

## Introduction

La clause `RETURNING` est l'une des fonctionnalités les plus élégantes et puissantes de PostgreSQL. Elle permet de **récupérer immédiatement** les valeurs des lignes qui viennent d'être insérées, modifiées ou supprimées, le tout dans une seule opération atomique.

Cette capacité distingue PostgreSQL de nombreux autres systèmes de gestion de bases de données et simplifie considérablement de nombreux patterns de développement courants.

### Qu'est-ce que RETURNING ?

`RETURNING` est une clause optionnelle que vous pouvez ajouter à la fin des commandes :
- `INSERT`
- `UPDATE`
- `DELETE`

Elle transforme ces commandes en requêtes qui **retournent des données**, à la manière d'un `SELECT`.

### Syntaxe générale

```sql
-- Avec INSERT
INSERT INTO table (colonnes...)
VALUES (valeurs...)
RETURNING colonne1, colonne2, ...;

-- Avec UPDATE
UPDATE table
SET colonne = valeur
WHERE condition
RETURNING colonne1, colonne2, ...;

-- Avec DELETE
DELETE FROM table
WHERE condition
RETURNING colonne1, colonne2, ...;
```

---

## 6.4.1. RETURNING avec INSERT : Récupérer les données insérées

### Le problème sans RETURNING

Traditionnellement, lorsque vous insérez une ligne dans une table, vous devez effectuer une requête supplémentaire pour récupérer les valeurs générées automatiquement (comme les ID auto-incrémentés).

#### Approche traditionnelle (sans RETURNING)

```sql
-- Étape 1 : Insérer
INSERT INTO employes (nom, prenom, email)
VALUES ('Dupont', 'Marie', 'marie.dupont@example.com');

-- Étape 2 : Récupérer l'ID généré (méthode problématique)
SELECT id FROM employes
WHERE nom = 'Dupont' AND prenom = 'Marie'
ORDER BY id DESC
LIMIT 1;
```

**Problèmes de cette approche** :
1. **Deux requêtes** au lieu d'une (moins performant)
2. **Race condition** : Entre l'INSERT et le SELECT, une autre insertion pourrait survenir
3. **Ambiguïté** : Si plusieurs "Marie Dupont" existent, laquelle est la bonne ?
4. **Complexité** : Code plus verbeux et plus fragile

### La solution avec RETURNING

```sql
-- Une seule requête qui insère ET retourne l'ID
INSERT INTO employes (nom, prenom, email)
VALUES ('Dupont', 'Marie', 'marie.dupont@example.com')
RETURNING id;
```

**Résultat immédiat** :
```
 id
----
 42
```

✅ **Avantages** :
- Une seule requête (atomique)
- Pas de race condition
- Pas d'ambiguïté
- Plus simple et plus lisible

### Retourner plusieurs colonnes

Vous pouvez retourner n'importe quelle colonne, pas seulement l'ID :

```sql
INSERT INTO employes (nom, prenom, email, salaire, date_embauche)
VALUES ('Martin', 'Pierre', 'pierre.martin@example.com', 45000, CURRENT_DATE)
RETURNING id, nom, prenom, date_embauche;
```

**Résultat** :
```
 id |  nom   | prenom |   date_embauche
----+--------+--------+-----------------
 43 | Martin | Pierre | 2025-11-19
```

### RETURNING * : Retourner toutes les colonnes

Pour récupérer toutes les colonnes de la ligne insérée :

```sql
INSERT INTO employes (nom, prenom, email)
VALUES ('Durand', 'Sophie', 'sophie.durand@example.com')
RETURNING *;
```

**Résultat** :
```
 id |  nom   | prenom |           email            | salaire | date_embauche
----+--------+--------+---------------------------+---------+---------------
 44 | Durand | Sophie | sophie.durand@example.com |         | 2025-11-19
```

> **Note** : Les colonnes avec des valeurs par défaut (comme `date_embauche`) sont retournées avec leur valeur calculée.

### RETURNING avec INSERT multiple

`RETURNING` fonctionne aussi avec les insertions multiples :

```sql
INSERT INTO employes (nom, prenom, email, salaire)
VALUES
    ('Petit', 'Anne', 'anne.petit@example.com', 42000),
    ('Roux', 'Thomas', 'thomas.roux@example.com', 44000),
    ('Bernard', 'Julie', 'julie.bernard@example.com', 46000)
RETURNING id, nom, prenom;
```

**Résultat** :
```
 id |  nom    | prenom
----+---------+--------
 45 | Petit   | Anne
 46 | Roux    | Thomas
 47 | Bernard | Julie
```

Chaque ligne insérée retourne ses valeurs !

### Cas d'usage : Récupérer des colonnes calculées

```sql
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER,
    montant_ht NUMERIC(10, 2),
    tva_taux NUMERIC(4, 2) DEFAULT 20.00,
    montant_ttc NUMERIC(10, 2) GENERATED ALWAYS AS (montant_ht * (1 + tva_taux / 100)) STORED,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer et récupérer immédiatement le montant TTC calculé
INSERT INTO commandes (client_id, montant_ht)
VALUES (123, 100.00)
RETURNING id, montant_ht, tva_taux, montant_ttc, date_creation;
```

**Résultat** :
```
 id | montant_ht | tva_taux | montant_ttc |     date_creation
----+------------+----------+-------------+------------------------
  1 |     100.00 |    20.00 |      120.00 | 2025-11-19 14:30:00
```

Vous obtenez immédiatement toutes les valeurs calculées automatiquement par la base de données !

---

## 6.4.2. RETURNING avec UPDATE : Voir les modifications

### Le problème sans RETURNING

Après un UPDATE, vous devez souvent effectuer un SELECT pour vérifier les nouvelles valeurs :

```sql
-- Étape 1 : Modifier
UPDATE employes
SET salaire = salaire * 1.10
WHERE id = 42;

-- Étape 2 : Vérifier
SELECT id, nom, salaire FROM employes WHERE id = 42;
```

### La solution avec RETURNING

```sql
UPDATE employes
SET salaire = salaire * 1.10
WHERE id = 42
RETURNING id, nom, prenom, salaire;
```

**Résultat** :
```
 id |  nom   | prenom | salaire
----+--------+--------+----------
 42 | Dupont | Marie  | 49500.00
```

Vous voyez immédiatement la nouvelle valeur du salaire (45000 * 1.10 = 49500).

### Retourner l'ancienne ET la nouvelle valeur (avec expression)

Vous pouvez utiliser des expressions dans `RETURNING` :

```sql
UPDATE employes
SET salaire = salaire * 1.10
WHERE id = 42
RETURNING
    id,
    nom,
    salaire AS nouveau_salaire,
    salaire / 1.10 AS ancien_salaire;
```

**Résultat** :
```
 id |  nom   | nouveau_salaire | ancien_salaire
----+--------+-----------------+----------------
 42 | Dupont |        49500.00 |       45000.00
```

> **Note** : `salaire / 1.10` recalcule l'ancienne valeur à partir de la nouvelle.

### UPDATE multiple avec RETURNING

Lorsque vous modifiez plusieurs lignes, `RETURNING` retourne toutes les lignes modifiées :

```sql
-- Augmenter tous les développeurs
UPDATE employes
SET salaire = salaire * 1.15
WHERE poste = 'Développeur'
RETURNING id, nom, prenom, salaire, poste;
```

**Résultat** :
```
 id |  nom    | prenom | salaire   |    poste
----+---------+--------+-----------+--------------
 45 | Petit   | Anne   | 48300.00  | Développeur
 48 | Garcia  | Lucas  | 52900.00  | Développeur
 51 | Moreau  | Emma   | 46000.00  | Développeur
```

Vous voyez immédiatement toutes les modifications appliquées !

### Cas d'usage : Audit des modifications

```sql
-- Créer une table d'historique
CREATE TABLE historique_salaires (
    id SERIAL PRIMARY KEY,
    employe_id INTEGER,
    ancien_salaire NUMERIC(10, 2),
    nouveau_salaire NUMERIC(10, 2),
    date_modification TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Utiliser RETURNING pour logger les modifications
WITH modifications AS (
    UPDATE employes
    SET salaire = salaire * 1.10
    WHERE poste = 'Développeur'
    RETURNING id, salaire / 1.10 AS ancien_salaire, salaire AS nouveau_salaire
)
INSERT INTO historique_salaires (employe_id, ancien_salaire, nouveau_salaire)
SELECT id, ancien_salaire, nouveau_salaire
FROM modifications;
```

Cette requête fait deux choses en une seule transaction :
1. Modifie les salaires
2. Enregistre l'historique des modifications

### Vérifier combien de lignes ont été modifiées

```sql
-- Obtenir le compte exact des modifications
WITH updated AS (
    UPDATE employes
    SET salaire = salaire * 1.10
    WHERE departement_id = 5
    RETURNING id
)
SELECT COUNT(*) AS nb_modifications FROM updated;
```

**Résultat** :
```
 nb_modifications
------------------
               12
```

---

## 6.4.3. RETURNING avec DELETE : Récupérer les données supprimées

### Le problème sans RETURNING

Avant de supprimer des données, vous devez souvent les sauvegarder :

```sql
-- Étape 1 : Sauvegarder dans une variable ou une table temporaire
SELECT * FROM employes WHERE date_embauche < '2020-01-01';

-- Étape 2 : Supprimer
DELETE FROM employes WHERE date_embauche < '2020-01-01';
```

### La solution avec RETURNING

```sql
DELETE FROM employes
WHERE date_embauche < '2020-01-01'
RETURNING *;
```

**Résultat** : Toutes les lignes supprimées sont retournées :
```
 id |  nom    | prenom |           email              | salaire | date_embauche
----+---------+--------+------------------------------+---------+---------------
  3 | Ancien  | Jean   | jean.ancien@example.com      | 38000   | 2019-05-12
  7 | Vieux   | Marie  | marie.vieux@example.com      | 41000   | 2018-11-03
 12 | Legacy  | Paul   | paul.legacy@example.com      | 39500   | 2019-08-22
```

### Archivage automatique avec RETURNING

Pattern puissant : archiver avant de supprimer, en une seule requête :

```sql
-- Table d'archive
CREATE TABLE employes_archives (LIKE employes INCLUDING ALL);

-- Supprimer et archiver en une seule opération
WITH deleted AS (
    DELETE FROM employes
    WHERE date_embauche < '2020-01-01'
    RETURNING *
)
INSERT INTO employes_archives
SELECT * FROM deleted;
```

**Avantages** :
- ✅ Opération atomique (tout réussit ou tout échoue)
- ✅ Pas de fenêtre temporelle entre SELECT et DELETE
- ✅ Garantit que les données archivées sont exactement celles supprimées
- ✅ Une seule transaction

### Vérifier ce qui va être supprimé

Vous pouvez utiliser RETURNING dans une transaction pour "simuler" une suppression :

```sql
BEGIN;

-- "Simuler" la suppression
DELETE FROM employes
WHERE salaire < 35000
RETURNING id, nom, prenom, salaire;

-- Examiner le résultat
-- Si c'est correct : COMMIT
-- Si erreur : ROLLBACK

ROLLBACK;  -- Annuler (rien n'est supprimé finalement)
```

### Compter les suppressions avec statistiques

```sql
WITH deleted AS (
    DELETE FROM logs
    WHERE date_creation < CURRENT_DATE - INTERVAL '1 year'
    RETURNING *
)
SELECT
    COUNT(*) AS nb_suppressions,
    MIN(date_creation) AS plus_ancien,
    MAX(date_creation) AS plus_recent,
    pg_size_pretty(SUM(pg_column_size(logs.*))::bigint) AS espace_libere
FROM deleted;
```

**Résultat** :
```
 nb_suppressions | plus_ancien | plus_recent | espace_libere
-----------------+-------------+-------------+---------------
           45230 | 2023-01-01  | 2024-11-18  | 15 MB
```

---

## 6.4.4. Expressions et calculs dans RETURNING

### Colonnes calculées

Vous pouvez effectuer des calculs directement dans `RETURNING` :

```sql
INSERT INTO produits (nom, prix_ht, tva_taux)
VALUES ('Ordinateur', 800.00, 20.00)
RETURNING
    id,
    nom,
    prix_ht,
    tva_taux,
    prix_ht * (1 + tva_taux / 100) AS prix_ttc,
    prix_ht * (tva_taux / 100) AS montant_tva;
```

**Résultat** :
```
 id |    nom     | prix_ht | tva_taux | prix_ttc | montant_tva
----+------------+---------+----------+----------+-------------
  1 | Ordinateur |  800.00 |    20.00 |   960.00 |      160.00
```

### Fonctions et conversions

```sql
INSERT INTO utilisateurs (nom, prenom, email)
VALUES ('DUPONT', 'marie', 'MARIE.DUPONT@EXAMPLE.COM')
RETURNING
    id,
    INITCAP(nom) AS nom_formate,
    INITCAP(prenom) AS prenom_formate,
    LOWER(email) AS email_normalise,
    LENGTH(nom) AS longueur_nom;
```

**Résultat** :
```
 id | nom_formate | prenom_formate | email_normalise              | longueur_nom
----+-------------+----------------+------------------------------+--------------
  1 | Dupont      | Marie          | marie.dupont@example.com     |            6
```

### Concaténations et transformations

```sql
UPDATE employes
SET
    nom = UPPER(nom),
    prenom = UPPER(prenom)
WHERE id = 42
RETURNING
    id,
    nom || ' ' || prenom AS nom_complet,
    LOWER(prenom || '.' || nom || '@company.com') AS email_genere,
    CURRENT_TIMESTAMP AS date_modification;
```

### CASE expressions dans RETURNING

```sql
UPDATE employes
SET salaire = salaire * 1.10
WHERE poste = 'Développeur'
RETURNING
    id,
    nom,
    salaire,
    CASE
        WHEN salaire >= 60000 THEN 'Senior'
        WHEN salaire >= 45000 THEN 'Confirmé'
        ELSE 'Junior'
    END AS niveau;
```

---

## 6.4.5. Cas d'usage avancés

### Pattern 1 : Gestion de file d'attente (Queue)

```sql
CREATE TABLE job_queue (
    id SERIAL PRIMARY KEY,
    job_type VARCHAR(50),
    payload JSONB,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP
);

-- Récupérer et marquer un job comme "en cours" en une seule opération atomique
UPDATE job_queue
SET
    status = 'processing',
    processed_at = CURRENT_TIMESTAMP
WHERE id = (
    SELECT id
    FROM job_queue
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED  -- Évite les conflits entre workers
)
RETURNING id, job_type, payload;
```

Ce pattern garantit qu'un seul worker récupère chaque job, sans risque de doublon.

### Pattern 2 : Incrémentation atomique de compteur

```sql
CREATE TABLE compteurs (
    id VARCHAR(50) PRIMARY KEY,
    valeur INTEGER DEFAULT 0
);

-- Incrémenter et récupérer la nouvelle valeur en une opération
UPDATE compteurs
SET valeur = valeur + 1
WHERE id = 'facture'
RETURNING valeur;
```

**Résultat** :
```
 valeur
--------
   1043
```

Utilisable pour générer des numéros de facture uniques et séquentiels.

### Pattern 3 : Swap de valeurs

```sql
-- Échanger les positions de deux éléments
WITH updated AS (
    UPDATE items
    SET position = CASE
        WHEN id = 5 THEN (SELECT position FROM items WHERE id = 8)
        WHEN id = 8 THEN (SELECT position FROM items WHERE id = 5)
    END
    WHERE id IN (5, 8)
    RETURNING id, position
)
SELECT * FROM updated ORDER BY id;
```

### Pattern 4 : Upsert avec RETURNING

Combiner `INSERT ... ON CONFLICT` avec `RETURNING` :

```sql
INSERT INTO statistiques (page, vues, derniere_visite)
VALUES ('accueil', 1, CURRENT_TIMESTAMP)
ON CONFLICT (page)
DO UPDATE SET
    vues = statistiques.vues + 1,
    derniere_visite = CURRENT_TIMESTAMP
RETURNING page, vues, derniere_visite;
```

**Résultat** (si la page existait déjà avec 42 vues) :
```
  page   | vues |    derniere_visite
---------+------+------------------------
 accueil |   43 | 2025-11-19 15:30:00
```

### Pattern 5 : Batch processing avec statistiques

```sql
WITH processed AS (
    UPDATE commandes
    SET
        statut = 'traitee',
        date_traitement = CURRENT_TIMESTAMP
    WHERE statut = 'en_attente'
      AND date_creation < CURRENT_TIMESTAMP - INTERVAL '1 hour'
    RETURNING id, montant, client_id
)
SELECT
    COUNT(*) AS nb_commandes_traitees,
    SUM(montant) AS montant_total,
    COUNT(DISTINCT client_id) AS nb_clients_concernes
FROM processed;
```

**Résultat** :
```
 nb_commandes_traitees | montant_total | nb_clients_concernes
-----------------------+---------------+----------------------
                   147 |      89234.50 |                   82
```

---

## 6.4.6. RETURNING dans les applications

### Utilisation avec les drivers

La plupart des drivers PostgreSQL supportent `RETURNING` nativement.

#### Python (psycopg3)

```python
import psycopg

conn = psycopg.connect("dbname=mabase")
cur = conn.cursor()

# Insérer et récupérer l'ID
cur.execute("""
    INSERT INTO employes (nom, prenom, email, salaire)
    VALUES (%s, %s, %s, %s)
    RETURNING id, date_embauche
""", ('Dupont', 'Marie', 'marie@example.com', 45000))

resultat = cur.fetchone()
print(f"ID créé : {resultat[0]}, Date : {resultat[1]}")

conn.commit()
```

#### Node.js (node-postgres)

```javascript
const { Pool } = require('pg');
const pool = new Pool();

async function creerEmploye(nom, prenom, email, salaire) {
    const result = await pool.query(
        `INSERT INTO employes (nom, prenom, email, salaire)
         VALUES ($1, $2, $3, $4)
         RETURNING id, nom, prenom, date_embauche`,
        [nom, prenom, email, salaire]
    );

    return result.rows[0]; // { id: 42, nom: 'Dupont', ... }
}
```

#### Java (JDBC)

```java
String sql = "INSERT INTO employes (nom, prenom, email, salaire) " +
             "VALUES (?, ?, ?, ?) RETURNING id, date_embauche";

PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, "Dupont");
stmt.setString(2, "Marie");
stmt.setString(3, "marie@example.com");
stmt.setBigDecimal(4, new BigDecimal("45000"));

ResultSet rs = stmt.executeQuery();
if (rs.next()) {
    int id = rs.getInt("id");
    Date dateEmbauche = rs.getDate("date_embauche");
    System.out.println("ID créé : " + id);
}
```

### Avantages pour les applications

1. **Réduction des round-trips réseau** : Une seule requête au lieu de deux
2. **Atomicité garantie** : Pas de risque de race condition
3. **Code plus simple** : Moins de gestion d'état
4. **Meilleures performances** : Moins de latence réseau

---

## 6.4.7. Combinaison avec CTE (Common Table Expressions)

Les CTE permettent d'enchaîner plusieurs opérations en utilisant `RETURNING`.

### Exemple : Pipeline de traitement

```sql
WITH nouveaux_employes AS (
    -- Étape 1 : Insérer les nouveaux employés
    INSERT INTO employes (nom, prenom, email, departement_id)
    VALUES
        ('Nouveau', 'Alice', 'alice@example.com', 1),
        ('Nouveau', 'Bob', 'bob@example.com', 1)
    RETURNING id, nom, prenom, departement_id
),
affectations_creees AS (
    -- Étape 2 : Créer leurs affectations
    INSERT INTO affectations (employe_id, projet_id, date_debut)
    SELECT id, 101, CURRENT_DATE
    FROM nouveaux_employes
    RETURNING employe_id, projet_id
)
-- Étape 3 : Retourner un résumé
SELECT
    COUNT(*) AS nb_employes_crees,
    COUNT(*) AS nb_affectations_creees
FROM nouveaux_employes, affectations_creees;
```

### Exemple : Réorganisation avec historique

```sql
WITH anciens AS (
    -- Supprimer les anciens employés et les archiver
    DELETE FROM employes
    WHERE date_depart IS NOT NULL
    RETURNING *
),
archives AS (
    -- Les insérer dans la table d'archives
    INSERT INTO employes_archives
    SELECT * FROM anciens
    RETURNING id, nom, prenom
)
-- Retourner un rapport
SELECT
    COUNT(*) AS nb_archives,
    STRING_AGG(nom || ' ' || prenom, ', ') AS employes_archives
FROM archives;
```

---

## 6.4.8. Limitations et considérations

### Ce que RETURNING ne peut pas faire

#### 1. Pas de RETURNING avec TRUNCATE

```sql
-- ❌ Ceci ne fonctionne pas
TRUNCATE TABLE employes RETURNING *;
-- ERROR: syntax error
```

`TRUNCATE` ne supporte pas `RETURNING` car il ne traite pas les lignes individuellement.

#### 2. Limité aux lignes directement affectées

`RETURNING` ne retourne que les lignes de la table directement modifiée, pas les lignes affectées par cascade.

```sql
-- Supprimer un département (avec ON DELETE CASCADE sur employes)
DELETE FROM departements
WHERE id = 5
RETURNING *;

-- Retourne uniquement la ligne du département
-- Ne retourne PAS les employés supprimés en cascade
```

### Considérations de performance

#### Impact mémoire

`RETURNING` stocke toutes les lignes retournées en mémoire avant de les renvoyer :

```sql
-- ⚠️ Attention : peut consommer beaucoup de mémoire
UPDATE employes
SET salaire = salaire * 1.10
RETURNING *;
-- Si 1 million d'employés → 1 million de lignes en mémoire
```

Pour de très gros volumes, préférez traiter par lots :

```sql
-- Traitement par lots de 1000
UPDATE employes
SET salaire = salaire * 1.10
WHERE id IN (
    SELECT id FROM employes
    WHERE salaire < 50000
    LIMIT 1000
)
RETURNING id, salaire;
```

#### Index et RETURNING

`RETURNING` n'ajoute pas de surcoût significatif par rapport à l'opération elle-même (INSERT/UPDATE/DELETE).

---

## 6.4.9. Comparaison avec d'autres SGBD

### Support de RETURNING

| SGBD | Support RETURNING | Notes |
|------|------------------|-------|
| **PostgreSQL** | ✅ Complet | INSERT, UPDATE, DELETE |
| **Oracle** | ✅ Partiel | Clause RETURNING INTO (PL/SQL) |
| **SQL Server** | ✅ Via OUTPUT | Clause OUTPUT (syntaxe différente) |
| **MySQL** | ❌ Non | Pas de support natif |
| **SQLite** | ✅ Depuis v3.35 | RETURNING ajouté récemment |

### Équivalents dans d'autres SGBD

#### SQL Server (OUTPUT)

```sql
-- SQL Server utilise OUTPUT au lieu de RETURNING
INSERT INTO employes (nom, prenom)
OUTPUT INSERTED.id, INSERTED.nom, INSERTED.prenom
VALUES ('Dupont', 'Marie');
```

#### Oracle (RETURNING INTO)

```sql
-- Oracle utilise RETURNING INTO (en PL/SQL)
DECLARE
    v_id NUMBER;
BEGIN
    INSERT INTO employes (nom, prenom)
    VALUES ('Dupont', 'Marie')
    RETURNING id INTO v_id;
END;
```

#### MySQL (pas de support direct)

```sql
-- MySQL nécessite deux requêtes
INSERT INTO employes (nom, prenom)
VALUES ('Dupont', 'Marie');

SELECT LAST_INSERT_ID() AS id;
```

---

## 6.4.10. Bonnes pratiques

### 1. Utiliser RETURNING systématiquement pour les ID générés

```sql
-- ✅ Bonne pratique
INSERT INTO employes (nom, prenom)
VALUES ('Dupont', 'Marie')
RETURNING id;

-- ❌ À éviter
INSERT INTO employes (nom, prenom)
VALUES ('Dupont', 'Marie');
SELECT currval('employes_id_seq');  -- Fragile et non standard
```

### 2. Retourner uniquement les colonnes nécessaires

```sql
-- ✅ Efficace : Retourne seulement ce qui est nécessaire
UPDATE employes
SET salaire = salaire * 1.10
WHERE id = 42
RETURNING id, salaire;

-- ⚠️ Moins efficace : Retourne tout alors qu'on n'en a pas besoin
UPDATE employes
SET salaire = salaire * 1.10
WHERE id = 42
RETURNING *;
```

### 3. Combiner avec des transactions pour la sécurité

```sql
BEGIN;

WITH deleted AS (
    DELETE FROM anciens_logs
    WHERE date_creation < '2024-01-01'
    RETURNING *
)
INSERT INTO logs_archives
SELECT * FROM deleted;

-- Vérifier le résultat avant de valider
SELECT COUNT(*) FROM logs_archives
WHERE date_creation < '2024-01-01';

COMMIT;
```

### 4. Documenter l'utilisation de RETURNING

```sql
-- RETURNING utilisé ici pour audit et traçabilité
-- Toutes les suppressions sont loggées automatiquement
WITH deleted AS (
    DELETE FROM utilisateurs
    WHERE derniere_connexion < CURRENT_DATE - INTERVAL '2 years'
    RETURNING id, email, derniere_connexion
)
INSERT INTO audit_suppressions (utilisateur_id, email, date_suppression)
SELECT id, email, CURRENT_TIMESTAMP
FROM deleted;
```

---

## Récapitulatif

### Avantages de RETURNING

| Avantage | Description |
|----------|-------------|
| **Atomicité** | Une seule opération au lieu de deux |
| **Performance** | Réduit les round-trips réseau |
| **Simplicité** | Code plus clair et plus concis |
| **Sécurité** | Pas de race condition |
| **Flexibilité** | Supporte expressions et calculs |

### Quand utiliser RETURNING ?

✅ **Utilisez RETURNING quand** :
- Vous avez besoin de récupérer des ID générés automatiquement
- Vous voulez vérifier les valeurs après modification
- Vous devez archiver des données avant suppression
- Vous implémentez des patterns atomiques (queues, compteurs)
- Vous voulez simplifier votre code applicatif

❌ **N'utilisez pas RETURNING (ou avec précaution) quand** :
- Vous modifiez des millions de lignes (impact mémoire)
- Vous n'avez pas besoin des données retournées
- Vous utilisez un SGBD qui ne supporte pas cette fonctionnalité

### Syntaxe rapide de référence

```sql
-- INSERT
INSERT INTO table (colonnes...)
VALUES (valeurs...)
RETURNING id, colonne1, colonne2;

-- UPDATE
UPDATE table
SET colonne = valeur
WHERE condition
RETURNING id, colonne1, colonne2;

-- DELETE
DELETE FROM table
WHERE condition
RETURNING id, colonne1, colonne2;

-- RETURNING avec expressions
... RETURNING
    id,
    colonne1 * 1.1 AS nouveau_calcul,
    UPPER(colonne2) AS texte_maj,
    CURRENT_TIMESTAMP AS timestamp_operation;

-- RETURNING avec CTE
WITH operation AS (
    INSERT INTO table1 VALUES (...)
    RETURNING id
)
INSERT INTO table2
SELECT id FROM operation;
```

---

## Conclusion

La clause `RETURNING` est l'une des fonctionnalités qui font de PostgreSQL un système de gestion de bases de données particulièrement élégant et puissant. Elle simplifie considérablement de nombreux patterns de développement courants, améliore les performances en réduisant les allers-retours réseau, et renforce la sécurité en garantissant l'atomicité des opérations.

**Points clés à retenir** :

1. `RETURNING` transforme INSERT, UPDATE et DELETE en requêtes qui retournent des données
2. Permet de récupérer les valeurs générées automatiquement en une seule opération
3. Supporte les expressions, fonctions et calculs
4. Facilite l'implémentation de patterns avancés (queues, audit, archivage)
5. Réduit la complexité du code applicatif et améliore les performances

En maîtrisant `RETURNING`, vous écrirez du code PostgreSQL plus efficace, plus sûr et plus élégant.

---


⏭️ [Nouveauté PG 18 : Support OLD et NEW dans les clauses RETURNING](/06-manipulation-des-donnees/05-old-new-dans-returning.md)
