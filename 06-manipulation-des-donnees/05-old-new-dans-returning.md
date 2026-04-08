🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5. Nouveauté PG 18 : Support OLD et NEW dans les clauses RETURNING

## Introduction

PostgreSQL 18 introduit une amélioration majeure de la clause `RETURNING` : la possibilité de référencer explicitement les **anciennes valeurs** (avec `OLD`) et les **nouvelles valeurs** (avec `NEW`) des lignes modifiées ou supprimées.

Cette fonctionnalité révolutionnaire simplifie considérablement de nombreux cas d'usage courants, notamment :
- L'audit des modifications
- Le suivi des changements
- La comparaison avant/après
- L'historisation des données

Avant PostgreSQL 18, obtenir simultanément les anciennes et nouvelles valeurs nécessitait des contournements complexes. Maintenant, c'est simple, élégant et performant.

---

## 6.5.1. Comprendre OLD et NEW : Les concepts de base

### Origine : Les variables de triggers

Les concepts `OLD` et `NEW` proviennent du monde des **triggers** (déclencheurs) dans les bases de données :

- **`OLD`** représente la ligne **avant** la modification  
- **`NEW`** représente la ligne **après** la modification

#### Dans un trigger (rappel)

```sql
CREATE OR REPLACE FUNCTION audit_trigger()  
RETURNS TRIGGER AS $$  
BEGIN  
    -- OLD contient l'ancienne valeur
    -- NEW contient la nouvelle valeur

    INSERT INTO audit_log (table_name, old_value, new_value)
    VALUES (TG_TABLE_NAME, OLD.salaire, NEW.salaire);

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### PostgreSQL 18 : OLD et NEW dans RETURNING

Avec PostgreSQL 18, ces mêmes concepts sont maintenant disponibles **directement dans la clause RETURNING**, sans avoir besoin de créer un trigger !

### Disponibilité selon l'opération

| Opération | OLD disponible | NEW disponible |
|-----------|----------------|----------------|
| **INSERT** | ❌ Non (pas d'ancienne valeur) | ✅ Oui |
| **UPDATE** | ✅ Oui | ✅ Oui |
| **DELETE** | ✅ Oui | ❌ Non (pas de nouvelle valeur) |

Cette logique est intuitive :
- Lors d'un **INSERT**, il n'y a pas de valeur précédente → pas de `OLD`
- Lors d'un **DELETE**, il n'y a plus de valeur après → pas de `NEW`
- Lors d'un **UPDATE**, les deux existent → `OLD` et `NEW` disponibles

---

## 6.5.2. OLD et NEW avec UPDATE : Comparer avant et après

### Le problème dans PostgreSQL 17 et antérieurs

Avant PG 18, pour obtenir les anciennes et nouvelles valeurs lors d'un UPDATE, il fallait :

#### Méthode 1 : Calculer l'ancienne valeur (approximatif et fragile)

```sql
-- PostgreSQL 17 : Calculer l'ancienne valeur à partir de la nouvelle
UPDATE employes  
SET salaire = salaire * 1.10  
WHERE id = 42  
RETURNING  
    id,
    nom,
    salaire AS nouveau_salaire,
    salaire / 1.10 AS ancien_salaire;  -- ⚠️ Calcul inverse approximatif
```

**Problèmes** :
- ❌ Fonctionne seulement si la formule est simple et réversible  
- ❌ Erreurs d'arrondi possibles  
- ❌ Impossible avec des fonctions complexes (CASE, COALESCE, etc.)  
- ❌ Ne fonctionne pas avec des valeurs conditionnelles

#### Méthode 2 : Sélectionner avant de modifier (deux requêtes)

```sql
-- PostgreSQL 17 : Deux requêtes nécessaires
-- Étape 1 : Récupérer les anciennes valeurs
SELECT id, nom, salaire AS ancien_salaire  
FROM employes  
WHERE id = 42;  

-- Étape 2 : Modifier et récupérer les nouvelles valeurs
UPDATE employes  
SET salaire = salaire * 1.10  
WHERE id = 42  
RETURNING id, nom, salaire AS nouveau_salaire;  
```

**Problèmes** :
- ❌ Deux requêtes = deux allers-retours réseau  
- ⚠️ Race condition possible entre SELECT et UPDATE  
- ❌ Pas atomique  
- ❌ Code plus complexe

### La solution avec PostgreSQL 18 : OLD et NEW

```sql
-- PostgreSQL 18 : Simple, direct et atomique
UPDATE employes  
SET salaire = salaire * 1.10  
WHERE id = 42  
RETURNING  
    id,
    nom,
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire;
```

**Résultat** :
```
 id |  nom   | ancien_salaire | nouveau_salaire
----+--------+----------------+-----------------
 42 | Dupont |       45000.00 |        49500.00
```

✅ **Avantages** :
- Simple et lisible
- Atomique (une seule opération)
- Valeurs exactes (pas d'approximation)
- Fonctionne avec n'importe quelle transformation

### Exemples pratiques

#### Augmentation de salaire avec différence

```sql
UPDATE employes  
SET salaire = salaire * 1.15  
WHERE poste = 'Développeur'  
RETURNING  
    id,
    nom,
    prenom,
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire,
    NEW.salaire - OLD.salaire AS augmentation;
```

**Résultat** :
```
 id |  nom    | prenom | ancien_salaire | nouveau_salaire | augmentation
----+---------+--------+----------------+-----------------+--------------
 12 | Martin  | Alice  |       42000.00 |        48300.00 |      6300.00
 18 | Durand  | Marc   |       46000.00 |        52900.00 |      6900.00
 25 | Bernard | Emma   |       40000.00 |        46000.00 |      6000.00
```

#### Modification avec calcul du pourcentage

```sql
UPDATE employes  
SET salaire = CASE  
    WHEN salaire < 40000 THEN salaire * 1.20
    WHEN salaire < 50000 THEN salaire * 1.15
    ELSE salaire * 1.10
END  
WHERE departement_id = 5  
RETURNING  
    id,
    nom,
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire,
    ROUND((NEW.salaire - OLD.salaire) / OLD.salaire * 100, 2) AS pourcentage_augmentation;
```

**Résultat** :
```
 id |  nom   | ancien_salaire | nouveau_salaire | pourcentage_augmentation
----+--------+----------------+-----------------+--------------------------
 15 | Petit  |       38000.00 |        45600.00 |                    20.00
 22 | Moreau |       45000.00 |        51750.00 |                    15.00
 31 | Roux   |       55000.00 |        60500.00 |                    10.00
```

#### Modification de plusieurs colonnes

```sql
UPDATE employes  
SET  
    salaire = salaire * 1.10,
    poste = 'Senior ' || poste,
    date_promotion = CURRENT_DATE
WHERE anciennete >= 5  
RETURNING  
    id,
    nom,
    OLD.poste AS ancien_poste,
    NEW.poste AS nouveau_poste,
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire;
```

**Résultat** :
```
 id |   nom   |  ancien_poste  |    nouveau_poste      | ancien_salaire | nouveau_salaire
----+---------+----------------+-----------------------+----------------+-----------------
  8 | Laurent | Développeur    | Senior Développeur    |       48000.00 |        52800.00
 14 | Garcia  | Analyste       | Senior Analyste       |       45000.00 |        49500.00
```

---

## 6.5.3. OLD et NEW avec DELETE : Archiver les données supprimées

### Le problème dans PostgreSQL 17

Lors d'une suppression, si vous vouliez archiver les données avec des informations additionnelles (comme la date de suppression), c'était complexe :

```sql
-- PostgreSQL 17 : Approche laborieuse
WITH to_delete AS (
    SELECT *, CURRENT_TIMESTAMP AS date_suppression
    FROM employes
    WHERE date_embauche < '2020-01-01'
),
archived AS (
    INSERT INTO employes_archives
    SELECT * FROM to_delete
    RETURNING *
)
DELETE FROM employes  
WHERE date_embauche < '2020-01-01';  
```

Cette approche avait des limitations et n'était pas vraiment atomique.

### La solution avec PostgreSQL 18

Avec OLD, vous pouvez directement référencer les valeurs supprimées :

```sql
-- PostgreSQL 18 : Simple et élégant
WITH deleted AS (
    DELETE FROM employes
    WHERE date_embauche < '2020-01-01'
    RETURNING
        OLD.id,
        OLD.nom,
        OLD.prenom,
        OLD.email,
        OLD.salaire,
        OLD.date_embauche,
        CURRENT_TIMESTAMP AS date_suppression,
        CURRENT_USER AS supprime_par
)
INSERT INTO employes_archives  
SELECT * FROM deleted;  
```

> **Note** : Avec DELETE, seul `OLD` est disponible car il n'y a pas de "nouvelle" valeur après suppression.

### Exemples pratiques

#### Suppression avec statistiques

```sql
DELETE FROM logs  
WHERE date_creation < CURRENT_DATE - INTERVAL '1 year'  
RETURNING  
    OLD.id,
    OLD.date_creation,
    OLD.niveau,
    CURRENT_DATE - OLD.date_creation AS age_en_jours;
```

**Résultat** :
```
   id   | date_creation |  niveau  | age_en_jours
--------+---------------+----------+--------------
 102345 | 2023-05-12    | INFO     |          557
 102346 | 2023-05-12    | DEBUG    |          557
 102347 | 2023-05-13    | WARNING  |          556
```

#### Archivage enrichi

```sql
WITH deleted AS (
    DELETE FROM sessions_expiries
    WHERE date_expiration < CURRENT_TIMESTAMP
    RETURNING
        OLD.id AS session_id,
        OLD.user_id,
        OLD.date_creation,
        OLD.date_expiration,
        CURRENT_TIMESTAMP AS date_nettoyage,
        OLD.date_expiration - OLD.date_creation AS duree_vie
)
INSERT INTO sessions_archives  
SELECT * FROM deleted;  
```

---

## 6.5.4. Cas d'usage : Audit et historisation

### Pattern 1 : Table d'audit complète

Créer automatiquement un historique de toutes les modifications :

```sql
-- Table d'audit
CREATE TABLE audit_employes (
    audit_id SERIAL PRIMARY KEY,
    operation VARCHAR(10),
    employe_id INTEGER,
    nom_avant VARCHAR(100),
    nom_apres VARCHAR(100),
    salaire_avant NUMERIC(10, 2),
    salaire_apres NUMERIC(10, 2),
    modifie_par VARCHAR(100),
    date_modification TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Modification avec audit automatique
WITH updated AS (
    UPDATE employes
    SET
        nom = UPPER(nom),
        salaire = salaire * 1.10
    WHERE departement_id = 3
    RETURNING
        OLD.id AS employe_id,
        OLD.nom AS nom_avant,
        NEW.nom AS nom_apres,
        OLD.salaire AS salaire_avant,
        NEW.salaire AS salaire_apres
)
INSERT INTO audit_employes (
    operation,
    employe_id,
    nom_avant,
    nom_apres,
    salaire_avant,
    salaire_apres,
    modifie_par
)
SELECT
    'UPDATE',
    employe_id,
    nom_avant,
    nom_apres,
    salaire_avant,
    salaire_apres,
    CURRENT_USER
FROM updated;
```

**Résultat dans audit_employes** :
```
 audit_id | operation | employe_id | nom_avant | nom_apres | salaire_avant | salaire_apres |  modifie_par  |  date_modification
----------+-----------+------------+-----------+-----------+---------------+---------------+---------------+---------------------
       47 | UPDATE    |         12 | dupont    | DUPONT    |      45000.00 |      49500.00 | admin         | 2025-11-19 10:30:00
       48 | UPDATE    |         18 | martin    | MARTIN    |      48000.00 |      52800.00 | admin         | 2025-11-19 10:30:00
```

### Pattern 2 : Historique des prix

Tracker l'évolution des prix de produits :

```sql
CREATE TABLE historique_prix (
    id SERIAL PRIMARY KEY,
    produit_id INTEGER,
    ancien_prix NUMERIC(10, 2),
    nouveau_prix NUMERIC(10, 2),
    variation_absolue NUMERIC(10, 2),
    variation_pourcentage NUMERIC(5, 2),
    date_changement TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Mise à jour des prix avec historisation
WITH updated AS (
    UPDATE produits
    SET prix = prix * 1.05  -- Augmentation de 5%
    WHERE categorie = 'Électronique'
    RETURNING
        OLD.id AS produit_id,
        OLD.prix AS ancien_prix,
        NEW.prix AS nouveau_prix,
        NEW.prix - OLD.prix AS variation_absolue,
        ROUND((NEW.prix - OLD.prix) / OLD.prix * 100, 2) AS variation_pourcentage
)
INSERT INTO historique_prix (
    produit_id,
    ancien_prix,
    nouveau_prix,
    variation_absolue,
    variation_pourcentage
)
SELECT * FROM updated;
```

### Pattern 3 : Détection de modifications significatives

Alerter uniquement sur les changements importants :

```sql
-- Identifier les modifications de plus de 20%
WITH updated AS (
    UPDATE employes
    SET salaire = nouveau_salaire_calcule
    WHERE conditions
    RETURNING
        OLD.id,
        OLD.nom,
        OLD.salaire AS ancien_salaire,
        NEW.salaire AS nouveau_salaire,
        ABS(NEW.salaire - OLD.salaire) / OLD.salaire AS variation_ratio
)
SELECT
    id,
    nom,
    ancien_salaire,
    nouveau_salaire,
    ROUND(variation_ratio * 100, 2) AS variation_pct
FROM updated  
WHERE variation_ratio > 0.20;  -- Plus de 20% de variation  
```

---

## 6.5.5. Cas d'usage : Validation et contrôles

### Validation des modifications

Vérifier que les modifications respectent certaines règles métier :

```sql
-- S'assurer qu'aucun salaire ne diminue
WITH updated AS (
    UPDATE employes
    SET salaire = recalcul_salaire(id)
    WHERE departement_id = 5
    RETURNING
        OLD.id,
        OLD.nom,
        OLD.salaire AS ancien_salaire,
        NEW.salaire AS nouveau_salaire
)
SELECT
    COUNT(*) AS nb_modifications,
    COUNT(*) FILTER (WHERE nouveau_salaire < ancien_salaire) AS nb_baisses,
    COUNT(*) FILTER (WHERE nouveau_salaire = ancien_salaire) AS nb_stables,
    COUNT(*) FILTER (WHERE nouveau_salaire > ancien_salaire) AS nb_hausses
FROM updated;
```

**Résultat** :
```
 nb_modifications | nb_baisses | nb_stables | nb_hausses
------------------+------------+------------+------------
               45 |          0 |          8 |         37
```

Si `nb_baisses > 0`, vous pouvez annuler la transaction (ROLLBACK).

### Contrôle des limites

```sql
-- Vérifier qu'aucune augmentation ne dépasse 50%
BEGIN;

WITH updated AS (
    UPDATE employes
    SET salaire = salaire * facteur_augmentation
    WHERE evaluation = 'Excellente'
    RETURNING
        OLD.id,
        OLD.nom,
        OLD.salaire AS ancien_salaire,
        NEW.salaire AS nouveau_salaire,
        (NEW.salaire - OLD.salaire) / OLD.salaire AS ratio
)
SELECT
    id,
    nom,
    ancien_salaire,
    nouveau_salaire,
    ratio
FROM updated  
WHERE ratio > 0.50;  

-- Si des lignes sont retournées, l'augmentation est trop importante
-- → ROLLBACK
-- Sinon → COMMIT
```

---

## 6.5.6. Comparaison avec les approches antérieures

### Tableau comparatif

| Aspect | PostgreSQL 17 | PostgreSQL 18 (OLD/NEW) |
|--------|--------------|------------------------|
| **Nombre de requêtes** | 2 (SELECT + UPDATE) | 1 (UPDATE avec RETURNING) |
| **Atomicité** | ⚠️ Fenêtre de temps | ✅ Complètement atomique |
| **Race conditions** | ⚠️ Possibles | ✅ Impossibles |
| **Calculs inverses** | ⚠️ Nécessaires (approximatifs) | ✅ Valeurs exactes |
| **Complexité code** | ❌ Élevée | ✅ Simple |
| **Performance** | ⚠️ 2× round-trips réseau | ✅ 1× round-trip |
| **Lisibilité** | ⚠️ Moyenne | ✅ Excellente |

### Exemple de migration

#### Avant PostgreSQL 18

```sql
-- Code complexe avec CTE et calculs inverses
WITH current_values AS (
    SELECT id, nom, salaire, poste
    FROM employes
    WHERE departement_id = 5
),
updated AS (
    UPDATE employes e
    SET salaire = CASE
        WHEN salaire < 40000 THEN salaire * 1.20
        WHEN salaire < 50000 THEN salaire * 1.15
        ELSE salaire * 1.10
    END
    WHERE departement_id = 5
    RETURNING id, nom, salaire, poste
)
SELECT
    u.id,
    u.nom,
    c.salaire AS ancien_salaire,  -- Depuis le SELECT initial
    u.salaire AS nouveau_salaire,
    u.salaire - c.salaire AS difference
FROM updated u  
JOIN current_values c ON u.id = c.id;  
```

**Problèmes** :
- CTE complexe
- JOIN nécessaire
- Risque si les données changent entre les deux étapes

#### Après PostgreSQL 18

```sql
-- Code simple et direct
UPDATE employes  
SET salaire = CASE  
    WHEN salaire < 40000 THEN salaire * 1.20
    WHEN salaire < 50000 THEN salaire * 1.15
    ELSE salaire * 1.10
END  
WHERE departement_id = 5  
RETURNING  
    id,
    nom,
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire,
    NEW.salaire - OLD.salaire AS difference;
```

**Avantages** :
- ✅ Simple et lisible  
- ✅ Atomique  
- ✅ Pas de JOIN nécessaire  
- ✅ Valeurs garanties cohérentes

---

## 6.5.7. Utilisation avec MERGE (PostgreSQL 18)

PostgreSQL 18 introduit également la commande `MERGE`, et OLD/NEW sont disponibles dans sa clause RETURNING !

### Syntaxe MERGE avec OLD et NEW

```sql
MERGE INTO employes t  
USING employes_import s ON t.email = s.email  
WHEN MATCHED THEN  
    UPDATE SET
        salaire = s.salaire,
        poste = s.poste
WHEN NOT MATCHED THEN
    INSERT (nom, prenom, email, salaire, poste)
    VALUES (s.nom, s.prenom, s.email, s.salaire, s.poste)
RETURNING
    CASE
        WHEN OLD.id IS NULL THEN 'INSERT'
        ELSE 'UPDATE'
    END AS operation,
    OLD.id AS ancien_id,
    NEW.id AS nouveau_id,
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire;
```

**Résultat** :
```
 operation | ancien_id | nouveau_id | ancien_salaire | nouveau_salaire
-----------+-----------+------------+----------------+-----------------
 UPDATE    |        42 |         42 |       45000.00 |        48000.00
 INSERT    |           |        89 |                |        35000.00
 UPDATE    |        55 |         55 |       52000.00 |        54000.00
```

Vous pouvez distinguer les INSERT des UPDATE grâce à `OLD.id IS NULL` !

---

## 6.5.8. Expressions et calculs avec OLD et NEW

### Calculs de différences

```sql
UPDATE stocks  
SET quantite = quantite - ventes_du_jour  
WHERE date_vente = CURRENT_DATE  
RETURNING  
    produit_id,
    OLD.quantite AS stock_avant,
    NEW.quantite AS stock_apres,
    OLD.quantite - NEW.quantite AS quantite_vendue,
    CASE
        WHEN NEW.quantite < 10 THEN 'ALERTE: Stock faible'
        WHEN NEW.quantite < 50 THEN 'ATTENTION: Réapprovisionner'
        ELSE 'OK'
    END AS statut_stock;
```

### Calculs de ratios et pourcentages

```sql
UPDATE metriques  
SET valeur = nouvelle_valeur_calculee  
WHERE date_mesure = CURRENT_DATE  
RETURNING  
    metrique_id,
    OLD.valeur AS valeur_precedente,
    NEW.valeur AS valeur_actuelle,
    NEW.valeur - OLD.valeur AS variation_absolue,
    ROUND((NEW.valeur - OLD.valeur) / OLD.valeur * 100, 2) AS variation_pct,
    CASE
        WHEN ABS(NEW.valeur - OLD.valeur) / OLD.valeur > 0.10 THEN 'Variation significative'
        ELSE 'Variation normale'
    END AS analyse;
```

### Concaténations et transformations

```sql
UPDATE employes  
SET  
    nom = UPPER(nom),
    prenom = INITCAP(prenom),
    email = LOWER(email)
WHERE departement_id = 5  
RETURNING  
    id,
    OLD.nom || ' -> ' || NEW.nom AS transformation_nom,
    OLD.prenom || ' -> ' || NEW.prenom AS transformation_prenom,
    OLD.email || ' -> ' || NEW.email AS transformation_email;
```

**Résultat** :
```
 id | transformation_nom | transformation_prenom | transformation_email
----+--------------------+-----------------------+------------------------------------------
 12 | dupont -> DUPONT   | marie -> Marie        | Marie.Dupont@Ex.com -> marie.dupont@ex.com
 18 | martin -> MARTIN   | PIERRE -> Pierre      | P.Martin@ex.com -> p.martin@ex.com
```

---

## 6.5.9. Performance et considérations techniques

### Impact sur les performances

L'utilisation de OLD et NEW dans RETURNING a un impact minimal sur les performances :

| Aspect | Impact |
|--------|--------|
| **Overhead mémoire** | Négligeable (valeurs déjà en mémoire) |
| **CPU** | Aucun surcoût (pas de calcul supplémentaire) |
| **I/O** | Identique (données déjà lues pour UPDATE/DELETE) |
| **Réseau** | Léger (plus de données retournées) |

### Optimisations internes

PostgreSQL 18 optimise l'accès aux valeurs OLD et NEW :

1. **Pas de copie supplémentaire** : Les valeurs OLD et NEW sont déjà présentes en mémoire pendant l'opération  
2. **Accès direct** : Pas de requête supplémentaire nécessaire  
3. **MVCC friendly** : Compatible avec le système MVCC de PostgreSQL

### Recommandations

✅ **Utilisez OLD et NEW** :
- Pour les audits et historiques
- Pour les validations
- Pour les comparaisons avant/après
- Pour simplifier le code

⚠️ **Attention avec de gros volumes** :
- RETURNING stocke toutes les lignes en mémoire
- Pour des millions de lignes, traitez par lots

```sql
-- ✅ Bon pour quelques milliers de lignes
UPDATE employes  
SET salaire = salaire * 1.10  
RETURNING OLD.salaire, NEW.salaire;  

-- ⚠️ Pour des millions de lignes, préférez par lots
DO $$  
DECLARE  
    batch_size INTEGER := 10000;
BEGIN
    LOOP
        WITH updated AS (
            UPDATE employes
            SET salaire = salaire * 1.10
            WHERE id IN (
                SELECT id FROM employes
                WHERE salaire < 50000
                LIMIT batch_size
                FOR UPDATE SKIP LOCKED
            )
            RETURNING OLD.salaire, NEW.salaire
        )
        INSERT INTO historique_salaires
        SELECT * FROM updated;

        EXIT WHEN NOT FOUND;
    END LOOP;
END $$;
```

---

## 6.5.10. Limitations et cas particuliers

### Limitations connues

#### 1. Pas de OLD avec INSERT

```sql
-- ❌ Erreur : OLD n'existe pas pour INSERT
INSERT INTO employes (nom, prenom)  
VALUES ('Nouveau', 'Jean')  
RETURNING OLD.id, NEW.id;  
-- ERROR: column "old" does not exist
```

**Solution** : Utilisez simplement les noms de colonnes ou NEW :
```sql
-- ✅ Correct
INSERT INTO employes (nom, prenom)  
VALUES ('Nouveau', 'Jean')  
RETURNING id, nom, prenom;  -- ou NEW.id, NEW.nom, NEW.prenom  
```

#### 2. Pas de NEW avec DELETE

```sql
-- ❌ Erreur : NEW n'existe pas pour DELETE
DELETE FROM employes  
WHERE id = 42  
RETURNING OLD.id, NEW.id;  
-- ERROR: column "new" does not exist
```

**Solution** : Utilisez OLD :
```sql
-- ✅ Correct
DELETE FROM employes  
WHERE id = 42  
RETURNING OLD.id, OLD.nom, OLD.prenom;  
```

#### 3. Ambiguïté possible

Si vous ne précisez ni OLD ni NEW, PostgreSQL utilise la valeur NEW par défaut (pour UPDATE) :

```sql
-- Ces deux requêtes sont équivalentes
UPDATE employes SET salaire = salaire * 1.10  
RETURNING salaire;  -- Retourne NEW.salaire (nouvelle valeur)  

UPDATE employes SET salaire = salaire * 1.10  
RETURNING NEW.salaire;  -- Explicite : nouvelle valeur  
```

**Recommandation** : Soyez explicite pour la clarté du code :

```sql
-- ✅ Clair et sans ambiguïté
UPDATE employes SET salaire = salaire * 1.10  
RETURNING  
    OLD.salaire AS ancien_salaire,
    NEW.salaire AS nouveau_salaire;
```

---

## 6.5.11. Exemples complets et patterns avancés

### Pattern complet : Système d'audit universel

```sql
-- Table d'audit générique
CREATE TABLE audit_universel (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    operation VARCHAR(10),
    row_id INTEGER,
    changes JSONB,
    user_name VARCHAR(100),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Fonction d'audit réutilisable avec OLD et NEW
CREATE OR REPLACE FUNCTION audit_changes(
    p_table_name TEXT,
    p_operation TEXT,
    p_row_id INTEGER,
    p_old_data JSONB,
    p_new_data JSONB
) RETURNS VOID AS $$
BEGIN
    INSERT INTO audit_universel (table_name, operation, row_id, changes, user_name)
    VALUES (
        p_table_name,
        p_operation,
        p_row_id,
        jsonb_build_object(
            'old', p_old_data,
            'new', p_new_data,
            'diff', jsonb_diff(p_old_data, p_new_data)
        ),
        CURRENT_USER
    );
END;
$$ LANGUAGE plpgsql;

-- Utilisation avec UPDATE
WITH updated AS (
    UPDATE employes
    SET
        salaire = salaire * 1.10,
        poste = 'Senior ' || poste
    WHERE id = 42
    RETURNING
        NEW.id,
        to_jsonb(OLD.*) AS old_data,
        to_jsonb(NEW.*) AS new_data
)
SELECT audit_changes(
    'employes',
    'UPDATE',
    id,
    old_data,
    new_data
) FROM updated;
```

### Pattern : Synchronisation bidirectionnelle

```sql
-- Synchroniser deux tables en gardant trace des différences
WITH sync AS (
    UPDATE table_principale
    SET valeur = table_source.valeur
    FROM table_source
    WHERE table_principale.id = table_source.id
      AND table_principale.valeur <> table_source.valeur
    RETURNING
        table_principale.id,
        OLD.valeur AS valeur_avant_sync,
        NEW.valeur AS valeur_apres_sync,
        table_source.derniere_modification
)
INSERT INTO log_synchronisation (
    table_name,
    row_id,
    valeur_avant,
    valeur_apres,
    source_timestamp
)
SELECT
    'table_principale',
    id,
    valeur_avant_sync,
    valeur_apres_sync,
    derniere_modification
FROM sync;
```

---

## Récapitulatif

### Tableau de référence rapide

| Opération | OLD disponible | NEW disponible | Usage typique |
|-----------|----------------|----------------|---------------|
| INSERT | ❌ Non | ✅ Oui | Récupérer valeurs générées |
| UPDATE | ✅ Oui | ✅ Oui | Comparer avant/après |
| DELETE | ✅ Oui | ❌ Non | Archiver données supprimées |
| MERGE | ✅ Oui | ✅ Oui | Distinguer INSERT/UPDATE |

### Syntaxe de référence

```sql
-- UPDATE avec OLD et NEW
UPDATE table  
SET colonne = nouvelle_valeur  
WHERE condition  
RETURNING  
    OLD.colonne AS ancienne_valeur,
    NEW.colonne AS nouvelle_valeur,
    NEW.colonne - OLD.colonne AS difference;

-- DELETE avec OLD
DELETE FROM table  
WHERE condition  
RETURNING  
    OLD.id,
    OLD.colonne1,
    OLD.colonne2;

-- MERGE avec OLD et NEW
MERGE INTO target t  
USING source s ON t.id = s.id  
WHEN MATCHED THEN UPDATE SET ...  
WHEN NOT MATCHED THEN INSERT ...  
RETURNING  
    CASE WHEN OLD.id IS NULL THEN 'INSERT' ELSE 'UPDATE' END AS action,
    OLD.colonne AS ancienne_valeur,
    NEW.colonne AS nouvelle_valeur;
```

### Avantages clés de cette fonctionnalité

| Avantage | Description |
|----------|-------------|
| **Simplicité** | Code plus court et plus lisible |
| **Atomicité** | Une seule opération, garantie de cohérence |
| **Performance** | Pas de requête supplémentaire nécessaire |
| **Précision** | Valeurs exactes (pas d'approximation) |
| **Sécurité** | Pas de race condition |
| **Flexibilité** | Calculs et comparaisons directs |

---

## Conclusion

Le support de OLD et NEW dans les clauses RETURNING est l'une des améliorations les plus significatives de PostgreSQL 18. Cette fonctionnalité :

1. **Simplifie radicalement** de nombreux patterns courants (audit, historique, validation)  
2. **Améliore les performances** en éliminant les requêtes supplémentaires  
3. **Renforce la robustesse** en garantissant l'atomicité des opérations  
4. **Enrichit le langage SQL** avec des capacités proches des triggers, mais plus simples

**Points clés à retenir** :

- ✅ OLD = valeurs avant modification  
- ✅ NEW = valeurs après modification  
- ✅ UPDATE : OLD et NEW disponibles  
- ✅ DELETE : seulement OLD  
- ✅ INSERT : seulement NEW (ou nom de colonne simple)  
- ✅ MERGE : OLD et NEW pour distinguer les opérations

Cette fonctionnalité place PostgreSQL encore plus en avant comme base de données offrant les outils les plus avancés et élégants pour la manipulation de données.

---


⏭️ [TRUNCATE vs DELETE : Différences transactionnelles et performances](/06-manipulation-des-donnees/06-truncate-vs-delete.md)
