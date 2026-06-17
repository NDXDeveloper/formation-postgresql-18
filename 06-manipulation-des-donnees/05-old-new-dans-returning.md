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

| Opération | `OLD` | `NEW` |
|-----------|-------|-------|
| `INSERT` (pur) | ✅ Renvoie `NULL` pour chaque colonne | ✅ Valeurs insérées |
| `INSERT … ON CONFLICT DO UPDATE` | ✅ Ligne préexistante (s'il y a conflit) ou `NULL` (insertion neuve) | ✅ Ligne finale |
| `UPDATE` | ✅ Ligne avant modification | ✅ Ligne après modification |
| `DELETE` | ✅ Ligne supprimée | ✅ Renvoie `NULL` pour chaque colonne |
| `MERGE` (toutes branches) | ✅ Selon l'action effectuée | ✅ Selon l'action effectuée |

> 📌 **Clé importante** : `OLD` et `NEW` sont **toujours accessibles** depuis PG 18, mais peuvent valoir `NULL` selon le type d'opération. Cela permet d'écrire des requêtes **polymorphes** qui marchent pour tout type d'opération (par exemple un `MERGE` dont la branche dépend du `MATCHED`/`NOT MATCHED`) et de tester `OLD IS NULL` pour distinguer une insertion d'une mise à jour.

> 💡 **Renommage des qualificateurs** : on peut renommer `OLD` et `NEW` (par exemple si vous avez réellement une table ou colonne nommée « old ») via la syntaxe :  
>
> ```sql
> UPDATE produits SET prix = prix * 1.10
> RETURNING WITH (OLD AS o, NEW AS n)
>     id, o.prix AS ancien, n.prix AS nouveau;
> ```

### Comportement par défaut (sans qualificateur)

Sans qualificateur `OLD.` ou `NEW.`, PostgreSQL choisit la valeur la plus naturelle :

| Opération | Sans qualificateur → équivalent à |
|-----------|-----------------------------------|
| `INSERT` | `NEW.colonne` (ligne insérée) |
| `UPDATE` | `NEW.colonne` (ligne après modification) |
| `DELETE` | `OLD.colonne` (ligne supprimée) |

Autrement dit, le code existant (avant PG 18) qui écrit `RETURNING id, nom, salaire` **continue de produire exactement le même résultat** en PG 18 — le mécanisme `OLD`/`NEW` est purement additif.

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

## 6.5.6 bis. `RETURNING OLD/NEW` ou trigger : quel mécanisme choisir ?

Les variables `OLD` et `NEW` sont historiquement associées aux **triggers** (depuis longtemps en PostgreSQL). PostgreSQL 18 les rend disponibles dans `RETURNING`. Les deux mécanismes peuvent sembler équivalents pour l'audit ou la comparaison avant/après — mais ils répondent à des besoins différents.

### Tableau comparatif

| Aspect | Trigger `AFTER UPDATE` (PL/pgSQL) | `RETURNING OLD.*, NEW.*` (PG 18+) |
|--------|------------------------------------|-------------------------------------|
| **Périmètre d'activation** | Automatique : toute modification de la table, **quelle que soit la requête** | Seulement la requête qui contient `RETURNING` |
| **Visibilité côté DBA** | Invisible aux applications — garantit l'audit même si le développeur l'oublie | Volontaire : le développeur doit l'écrire explicitement |
| **Overhead** | Permanent (overhead par ligne, même quand pas nécessaire) | Aucun overhead si on n'utilise pas la clause |
| **Logique conditionnelle** | Plein PL/pgSQL : branches, appels de fonctions, exceptions | Expressions SQL uniquement |
| **Destination des données** | N'importe où : autre table, fichier, NOTIFY, appel de fonction externe | Le résultat de la commande (à consommer côté client ou via CTE) |
| **Mode `DEFERRABLE`** | ⚠️ Seulement via un `CONSTRAINT TRIGGER` (`INITIALLY DEFERRED`) — un trigger `AFTER UPDATE` ordinaire n'accepte pas `DEFERRABLE` | ❌ Pas applicable |
| **Tests unitaires** | Difficile : il faut tester côté base | Facile : tout le comportement est dans la requête testée |

### Quand préférer un **trigger**

- L'audit doit être **obligatoire et exhaustif** : aucune modification ne doit y échapper, même si un développeur oublie de l'écrire.
- La logique d'audit est **complexe** (calculs, appels de fonctions stockées, écriture dans plusieurs tables).
- Vous voulez **factoriser** : la même règle s'applique à plusieurs tables — un trigger générique sur chacune est plus simple que copier la clause `RETURNING` partout.
- Vous avez besoin d'un comportement **différé** (à la fin de la transaction, après tout le reste).

### Quand préférer **`RETURNING OLD/NEW`**

- L'audit est **lié à une requête métier précise** (un import, une réorganisation, une migration).
- Vous voulez **enchaîner** la donnée auditée dans une CTE pour la passer à une autre instruction.
- Vous ne voulez **pas** ajouter d'overhead permanent à la table (qui ne sera audité que pour cette opération exceptionnelle).
- Vous cherchez la **lisibilité maximale** : tout est dans la requête, sans aller-retour vers `pg_trigger`.

### Exemple : la même audit avec les deux approches

**Avec trigger** :

```sql
-- Setup une seule fois (le DBA pose le trigger)
CREATE OR REPLACE FUNCTION audit_salaire_trg()  
RETURNS TRIGGER AS $$  
BEGIN  
    IF NEW.salaire IS DISTINCT FROM OLD.salaire THEN
        INSERT INTO audit_salaires (employe_id, ancien, nouveau, ts)
        VALUES (OLD.id, OLD.salaire, NEW.salaire, now());
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_salaire_aft  
AFTER UPDATE OF salaire ON employes  
FOR EACH ROW EXECUTE FUNCTION audit_salaire_trg();  

-- Désormais TOUT UPDATE est audité, peu importe l'auteur :
UPDATE employes SET salaire = salaire * 1.05 WHERE departement_id = 3;
```

**Avec `RETURNING OLD/NEW`** :

```sql
-- À écrire à chaque requête métier qu'on veut auditer
WITH modifs AS (
    UPDATE employes SET salaire = salaire * 1.05
    WHERE departement_id = 3
    RETURNING id, OLD.salaire AS ancien, NEW.salaire AS nouveau
)
INSERT INTO audit_salaires (employe_id, ancien, nouveau, ts)  
SELECT id, ancien, nouveau, now() FROM modifs  
WHERE ancien IS DISTINCT FROM nouveau;  
```

> 💡 **Règle générale** : pour un audit *réglementaire* ou *de sécurité* (« on doit pouvoir prouver que toute modification est tracée »), un **trigger** est plus sûr. Pour une opération ponctuelle ou un pipeline ETL, **`RETURNING OLD/NEW`** est plus simple et plus rapide.

---

## 6.5.7. Utilisation avec MERGE

La commande `MERGE` est disponible depuis **PostgreSQL 15** ; PostgreSQL 17 lui a ajouté la clause `RETURNING`. C'est **PostgreSQL 18** qui complète enfin le tableau en autorisant `OLD` et `NEW` dans le `RETURNING` d'un `MERGE` — extrêmement utile, car `MERGE` peut exécuter une `INSERT`, une `UPDATE` ou une `DELETE` selon la branche `WHEN` choisie pour chaque ligne source.

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

⚠️ **Attention avec de gros volumes** : `RETURNING` matérialise les lignes retournées avant de les renvoyer au client. Sur un `UPDATE` de plusieurs millions de lignes avec `RETURNING OLD.*, NEW.*`, vous pouvez **consommer beaucoup de mémoire et de WAL**. Pour ces volumes, traitez par lots dans une **procédure** (cf. section [6.3](/06-manipulation-des-donnees/03-update-et-delete.md)) :

```sql
CREATE OR REPLACE PROCEDURE augmenter_salaires_par_lots(taille_lot INTEGER)  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    nb_lignes INTEGER;
BEGIN
    LOOP
        -- Sélectionner un lot de lignes non encore traitées
        -- et capturer OLD/NEW pour l'historisation
        WITH lot AS (
            SELECT id FROM employes
            WHERE salaire < 50000
              AND NOT augmente_2025  -- colonne ajoutée pour marquer le traitement
            ORDER BY id
            LIMIT taille_lot
            FOR UPDATE SKIP LOCKED
        ),
        modifies AS (
            UPDATE employes e
            SET salaire = salaire * 1.10,
                augmente_2025 = true
            FROM lot
            WHERE e.id = lot.id
            RETURNING e.id, OLD.salaire AS ancien, NEW.salaire AS nouveau
        )
        INSERT INTO historique_salaires (employe_id, ancien, nouveau)
        SELECT id, ancien, nouveau FROM modifies;

        -- ROW_COUNT = nombre de lignes réellement traitées par le dernier ordre.
        -- (On ne peut pas faire « RETURNING ... INTO tableau » : un RETURNING qui
        --  ramène plusieurs lignes vers une variable lève « query returned more
        --  than one row ». GET DIAGNOSTICS est le mécanisme idiomatique ici.)
        GET DIAGNOSTICS nb_lignes = ROW_COUNT;
        EXIT WHEN nb_lignes = 0;  -- plus aucune ligne à traiter : on sort

        -- Commit après chaque lot : indispensable pour libérer les verrous
        -- et limiter la taille de la transaction.
        COMMIT;
    END LOOP;
END;
$$;

CALL augmenter_salaires_par_lots(10000);
```

> 📌 La présence d'une colonne « marqueur » (`augmente_2025` ici) est essentielle : sans elle, après une première itération, les lignes augmentées pourraient encore satisfaire `salaire < 50000` et seraient ré-augmentées **en boucle infinie**. Alternative : utiliser une colonne `derniere_revalorisation TIMESTAMP` et filtrer dessus.

---

## 6.5.10. Limitations et cas particuliers

### Subtilités à connaître

#### 1. `OLD` dans un `INSERT` pur : retourne `NULL`, sans erreur

Contrairement à ce qu'on pourrait croire, écrire `OLD.colonne` dans le `RETURNING` d'un `INSERT` pur **ne provoque pas d'erreur** : chaque colonne `OLD` vaut simplement `NULL` (il n'y a pas de ligne précédente).

```sql
INSERT INTO employes (nom, prenom)  
VALUES ('Nouveau', 'Jean')  
RETURNING OLD.id AS ancien_id, NEW.id AS nouveau_id;  
```

**Résultat** :
```
 ancien_id | nouveau_id
-----------+------------
           |         99
```

C'est précisément ce qui permet le pattern `(OLD IS NULL)::boolean AS is_new` pour distinguer une insertion d'une mise à jour dans un `INSERT … ON CONFLICT` ou un `MERGE`.

#### 2. `NEW` dans un `DELETE` pur : retourne `NULL`, sans erreur

De même, `NEW.colonne` dans un `DELETE` renvoie `NULL` :

```sql
DELETE FROM employes  
WHERE id = 42  
RETURNING OLD.id AS supprime_id, NEW.id AS nouveau_id;  
```

**Résultat** :
```
 supprime_id | nouveau_id
-------------+------------
          42 |
```

#### 3. Comportement par défaut sans qualificateur

Si l'on ne préfixe ni par `OLD` ni par `NEW`, PostgreSQL choisit la valeur « sensée » pour l'opération (voir le tableau plus haut). Les codes existants ne changent donc pas de comportement entre PG ≤ 17 et PG 18.

```sql
-- Ces deux UPDATE ... RETURNING retournent strictement la même chose :
UPDATE employes SET salaire = salaire * 1.10 RETURNING salaire;  
UPDATE employes SET salaire = salaire * 1.10 RETURNING NEW.salaire;  
```

**Recommandation** : dès qu'on a besoin de l'ancienne valeur ou de comparer avant/après, **soyez explicite** :

```sql
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
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name VARCHAR(100),
    operation VARCHAR(10),
    row_id INTEGER,
    changes JSONB,
    user_name VARCHAR(100),
    moment TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Fonction utilitaire : calcule la "diff" entre deux JSONB
-- (PostgreSQL n'offre pas de jsonb_diff natif — voici une implémentation simple
--  qui ne retient que les clés dont la valeur a changé en passant de old à new)
CREATE OR REPLACE FUNCTION jsonb_diff(old_data JSONB, new_data JSONB)  
RETURNS JSONB AS $$  
    SELECT COALESCE(jsonb_object_agg(key, value), '{}'::jsonb)
    FROM jsonb_each(new_data)
    WHERE old_data IS NULL
       OR old_data -> key IS DISTINCT FROM value;
$$ LANGUAGE sql IMMUTABLE;

-- Utilisation avec UPDATE : un seul appel SQL fait la mise à jour ET l'audit
WITH updated AS (
    UPDATE employes
    SET
        salaire = salaire * 1.10,
        poste = 'Senior ' || poste
    WHERE id = 42
    RETURNING
        NEW.id AS row_id,
        to_jsonb(OLD) AS old_data,
        to_jsonb(NEW) AS new_data
)
INSERT INTO audit_universel (table_name, operation, row_id, changes, user_name)  
SELECT  
    'employes',
    'UPDATE',
    row_id,
    jsonb_build_object(
        'old', old_data,
        'new', new_data,
        'diff', jsonb_diff(old_data, new_data)
    ),
    CURRENT_USER
FROM updated;
```

> 📌 **Note sur `jsonb_diff`** : PostgreSQL n'offre pas de fonction `jsonb_diff` native. L'implémentation ci-dessus ne couvre que les clés modifiées au premier niveau et marque comme « modifiée » toute clé absente de `old_data`. Pour un diff complet (suppressions de clés, comparaison récursive sur les objets imbriqués), il existe des extensions comme [jsonb_deep_diff](https://github.com/dbgate/dbgate/blob/master/packages/api/src/utility/jsonbDiff.ts) ou des implémentations PL/pgSQL plus élaborées — adaptez selon votre besoin réel.

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

| Opération | `OLD` (avant) | `NEW` (après) | Usage typique |
|-----------|---------------|---------------|---------------|
| `INSERT` | ✅ vaut `NULL` partout | ✅ ligne insérée | Récupérer les valeurs générées |
| `UPDATE` | ✅ ligne avant | ✅ ligne après | Comparer avant / après |
| `DELETE` | ✅ ligne supprimée | ✅ vaut `NULL` partout | Archiver les données supprimées |
| `INSERT … ON CONFLICT DO UPDATE` | ✅ ligne préexistante si conflit, `NULL` sinon | ✅ ligne finale | Distinguer INSERT pur d'UPDATE via `OLD IS NULL` |
| `MERGE` | ✅ selon la branche | ✅ selon la branche | Auditer une consolidation polymorphe |

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

- ✅ `OLD` = ligne avant modification (`NULL` pour un `INSERT` pur)  
- ✅ `NEW` = ligne après modification (`NULL` pour un `DELETE` pur)  
- ✅ `UPDATE` : les deux portent des valeurs réelles, c'est le cas le plus riche  
- ✅ `INSERT … ON CONFLICT` et `MERGE` : on peut tester `OLD IS NULL` pour reconnaître une insertion vs une mise à jour dans la même requête  
- ✅ Sans qualificateur `OLD.` / `NEW.`, PostgreSQL choisit la valeur la plus naturelle — donc **tout code antérieur à PG 18 continue de fonctionner à l'identique**

Cette fonctionnalité place PostgreSQL encore plus en avant comme base de données offrant les outils les plus avancés et élégants pour la manipulation de données.

---


⏭️ [TRUNCATE vs DELETE : Différences transactionnelles et performances](/06-manipulation-des-donnees/06-truncate-vs-delete.md)
