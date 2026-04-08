🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.8. MERGE : Consolidation de données (avec OLD/NEW)

## Introduction

`MERGE` est une commande SQL standard (SQL:2003) introduite dans **PostgreSQL 15** avec le support de base (WHEN MATCHED, WHEN NOT MATCHED), puis enrichie dans **PostgreSQL 17** (WHEN NOT MATCHED BY SOURCE, RETURNING) et **PostgreSQL 18** (OLD et NEW dans RETURNING).

Cette commande permet de **synchroniser** des données entre deux tables en effectuant différentes actions (INSERT, UPDATE, DELETE) selon qu'une ligne correspond ou non. C'est l'outil idéal pour les opérations ETL (Extract, Transform, Load), les synchronisations de données, et les imports complexes.

> **Analogie** : MERGE est comme une opération "synchronisation" entre deux dossiers : il copie les nouveaux fichiers, met à jour les fichiers modifiés, et peut supprimer les fichiers obsolètes.

---

## 6.8.1. Qu'est-ce que MERGE ?

### Concept de base

`MERGE` combine trois opérations SQL en une seule commande :
- **INSERT** : Pour les lignes qui existent dans la source mais pas dans la cible  
- **UPDATE** : Pour les lignes qui existent dans les deux tables (correspondance)  
- **DELETE** : Pour les lignes qui existent dans la cible mais plus dans la source

### Terminologie

- **Table cible (target)** : La table que vous voulez modifier  
- **Table source (source)** : La table ou requête contenant les nouvelles données  
- **Condition de correspondance (ON)** : Critère pour déterminer si une ligne correspond  
- **WHEN MATCHED** : Actions à effectuer quand il y a correspondance  
- **WHEN NOT MATCHED** : Actions pour les lignes sans correspondance

### Syntaxe générale

```sql
MERGE INTO table_cible AS target  
USING table_source AS source  
ON condition_de_correspondance  
WHEN MATCHED [AND condition] THEN  
    UPDATE SET colonne = valeur | DELETE
WHEN NOT MATCHED [BY TARGET] [AND condition] THEN
    INSERT (colonnes) VALUES (valeurs)
WHEN NOT MATCHED BY SOURCE [AND condition] THEN
    DELETE
RETURNING ...;
```

---

## 6.8.2. MERGE vs ON CONFLICT : Quelles différences ?

Avant de plonger dans MERGE, comparons-le avec ON CONFLICT (UPSERT) que nous avons vu précédemment.

### Tableau comparatif

| Aspect | ON CONFLICT | MERGE |
|--------|-------------|-------|
| **Standard SQL** | ❌ PostgreSQL spécifique | ✅ SQL:2003 standard |
| **Opérations** | INSERT + UPDATE | INSERT + UPDATE + DELETE |
| **Source de données** | VALUES ou SELECT | Table ou requête |
| **Portabilité** | PostgreSQL uniquement | Portable (SQL Server, Oracle, etc.) |
| **Syntaxe** | Simple et concise | Plus verbeuse mais plus flexible |
| **Suppression** | ❌ Non supportée | ✅ WHEN MATCHED THEN DELETE |
| **Conditions multiples** | ❌ Limitées | ✅ Plusieurs WHEN avec conditions |
| **Cas d'usage** | Upsert simple | Synchronisation complexe |

### Quand utiliser quoi ?

**Utilisez ON CONFLICT quand** :
- ✅ Vous faites un simple upsert (INSERT or UPDATE)  
- ✅ Vous voulez une syntaxe courte et claire  
- ✅ Vous n'avez pas besoin de DELETE  
- ✅ La source est directement dans la requête (VALUES)

**Utilisez MERGE quand** :
- ✅ Vous synchronisez deux tables complètes  
- ✅ Vous avez besoin de DELETE en plus de INSERT/UPDATE  
- ✅ Vous avez des conditions complexes  
- ✅ Vous voulez de la portabilité SQL standard  
- ✅ Vous avez plusieurs sources de données à fusionner

---

## 6.8.3. MERGE basique : INSERT et UPDATE

### Exemple simple de synchronisation

Imaginons deux tables : une table de production et une table de staging pour les imports.

```sql
-- Table de production (cible)
CREATE TABLE produits (
    sku VARCHAR(50) PRIMARY KEY,
    nom VARCHAR(200),
    prix NUMERIC(10, 2),
    stock INTEGER,
    derniere_maj TIMESTAMP
);

-- Table de staging (source)
CREATE TABLE produits_import (
    sku VARCHAR(50),
    nom VARCHAR(200),
    prix NUMERIC(10, 2),
    stock INTEGER,
    date_import TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Données initiales dans production
INSERT INTO produits (sku, nom, prix, stock, derniere_maj) VALUES
    ('PROD-001', 'Ordinateur', 899.99, 10, '2025-11-01'),
    ('PROD-002', 'Souris', 29.99, 100, '2025-11-01'),
    ('PROD-003', 'Clavier', 79.99, 50, '2025-11-01');

-- Nouvelles données dans staging
INSERT INTO produits_import (sku, nom, prix, stock) VALUES
    ('PROD-002', 'Souris Gaming', 39.99, 120),     -- Existe : UPDATE
    ('PROD-003', 'Clavier Mécanique', 89.99, 45),  -- Existe : UPDATE
    ('PROD-004', 'Écran', 299.99, 20);             -- Nouveau : INSERT
```

### MERGE pour synchroniser

```sql
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED THEN  
    UPDATE SET
        nom = source.nom,
        prix = source.prix,
        stock = source.stock,
        derniere_maj = source.date_import
WHEN NOT MATCHED THEN
    INSERT (sku, nom, prix, stock, derniere_maj)
    VALUES (source.sku, source.nom, source.prix, source.stock, source.date_import);
```

**Résultat dans la table produits** :
```
    sku     |        nom         |  prix  | stock |    derniere_maj
------------+--------------------+--------+-------+---------------------
 PROD-001   | Ordinateur         | 899.99 |    10 | 2025-11-01 ...      (inchangé)
 PROD-002   | Souris Gaming      |  39.99 |   120 | 2025-11-19 ...      (mis à jour)
 PROD-003   | Clavier Mécanique  |  89.99 |    45 | 2025-11-19 ...      (mis à jour)
 PROD-004   | Écran              | 299.99 |    20 | 2025-11-19 ...      (inséré)
```

### Décomposition étape par étape

1. **MERGE INTO produits AS target** : Désigne la table à modifier  
2. **USING produits_import AS source** : Désigne la source des données  
3. **ON target.sku = source.sku** : Condition de correspondance (comme un JOIN)  
4. **WHEN MATCHED THEN UPDATE** : Si la correspondance existe → mise à jour  
5. **WHEN NOT MATCHED THEN INSERT** : Si pas de correspondance → insertion

---

## 6.8.4. Conditions multiples avec AND

Vous pouvez ajouter des conditions supplémentaires à chaque clause WHEN.

### UPDATE seulement si les données ont changé

```sql
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED AND (  
    target.nom != source.nom OR
    target.prix != source.prix OR
    target.stock != source.stock
) THEN
    UPDATE SET
        nom = source.nom,
        prix = source.prix,
        stock = source.stock,
        derniere_maj = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN
    INSERT (sku, nom, prix, stock, derniere_maj)
    VALUES (source.sku, source.nom, source.prix, source.stock, CURRENT_TIMESTAMP);
```

Cela évite de mettre à jour des lignes qui n'ont pas changé, améliorant les performances.

### INSERT seulement si le stock est suffisant

```sql
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED THEN  
    UPDATE SET
        nom = source.nom,
        prix = source.prix,
        stock = source.stock,
        derniere_maj = CURRENT_TIMESTAMP
WHEN NOT MATCHED AND source.stock > 0 THEN
    INSERT (sku, nom, prix, stock, derniere_maj)
    VALUES (source.sku, source.nom, source.prix, source.stock, CURRENT_TIMESTAMP);
```

Les produits avec un stock nul ne sont pas insérés.

---

## 6.8.5. DELETE avec MERGE

Une des forces de MERGE est la possibilité de supprimer des lignes.

### WHEN MATCHED THEN DELETE

Supprimer les produits qui ne sont plus dans le catalogue source :

```sql
-- Scénario : produits_import contient seulement les produits actifs
-- On veut supprimer de produits ceux qui ne sont plus dans produits_import

MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED THEN  
    UPDATE SET
        nom = source.nom,
        prix = source.prix,
        stock = source.stock,
        derniere_maj = CURRENT_TIMESTAMP
WHEN NOT MATCHED BY TARGET THEN
    INSERT (sku, nom, prix, stock, derniere_maj)
    VALUES (source.sku, source.nom, source.prix, source.stock, CURRENT_TIMESTAMP)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
```

**Explication des clauses** :
- `WHEN MATCHED` : Le produit existe dans les deux tables → UPDATE  
- `WHEN NOT MATCHED BY TARGET` : Le produit existe seulement dans la source → INSERT  
- `WHEN NOT MATCHED BY SOURCE` : Le produit existe seulement dans la cible → DELETE (requiert **PostgreSQL 17+**)

### DELETE conditionnel

Supprimer seulement si certaines conditions sont remplies :

```sql
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED THEN  
    UPDATE SET
        nom = source.nom,
        prix = source.prix,
        stock = source.stock
WHEN NOT MATCHED BY TARGET THEN
    INSERT (sku, nom, prix, stock)
    VALUES (source.sku, source.nom, source.prix, source.stock)
WHEN NOT MATCHED BY SOURCE AND target.stock = 0 THEN
    DELETE;
```

Supprime les produits obsolètes uniquement s'ils ont un stock nul.

### Exemple : Synchronisation complète d'un catalogue

```sql
-- Table source : catalogue actuel du fournisseur
CREATE TABLE catalogue_fournisseur (
    reference VARCHAR(50),
    designation VARCHAR(200),
    prix_achat NUMERIC(10, 2),
    disponible BOOLEAN
);

-- Synchronisation complète
MERGE INTO produits AS target  
USING catalogue_fournisseur AS source  
ON target.sku = source.reference  
WHEN MATCHED AND source.disponible = true THEN  
    UPDATE SET
        nom = source.designation,
        prix = source.prix_achat * 1.5,  -- Marge de 50%
        derniere_maj = CURRENT_TIMESTAMP
WHEN MATCHED AND source.disponible = false THEN
    DELETE  -- Supprimer les produits indisponibles
WHEN NOT MATCHED AND source.disponible = true THEN
    INSERT (sku, nom, prix, stock, derniere_maj)
    VALUES (source.reference, source.designation, source.prix_achat * 1.5, 0, CURRENT_TIMESTAMP);
```

---

## 6.8.6. OLD et NEW dans MERGE (PostgreSQL 18)

La clause RETURNING avec MERGE a été introduite dans **PostgreSQL 17**. **PostgreSQL 18** y ajoute le support de OLD et NEW, permettant de distinguer et comparer les valeurs avant et après l'opération.

### Comprendre OLD et NEW dans MERGE

| Situation | OLD | NEW |
|-----------|-----|-----|
| **WHEN MATCHED THEN UPDATE** | ✅ Valeurs avant UPDATE | ✅ Valeurs après UPDATE |
| **WHEN MATCHED THEN DELETE** | ✅ Valeurs supprimées | ❌ NULL |
| **WHEN NOT MATCHED THEN INSERT** | ❌ NULL | ✅ Valeurs insérées |

### Exemple basique avec OLD et NEW

```sql
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED THEN  
    UPDATE SET
        prix = source.prix,
        stock = source.stock
WHEN NOT MATCHED THEN
    INSERT (sku, nom, prix, stock)
    VALUES (source.sku, source.nom, source.prix, source.stock)
RETURNING
    CASE
        WHEN OLD.sku IS NULL THEN 'INSERT'
        WHEN NEW.sku IS NULL THEN 'DELETE'
        ELSE 'UPDATE'
    END AS action,
    COALESCE(OLD.sku, NEW.sku) AS sku,
    OLD.prix AS ancien_prix,
    NEW.prix AS nouveau_prix,
    NEW.prix - COALESCE(OLD.prix, 0) AS variation_prix;
```

**Résultat possible** :
```
 action |    sku    | ancien_prix | nouveau_prix | variation_prix
--------+-----------+-------------+--------------+----------------
 UPDATE | PROD-002  |       29.99 |        39.99 |          10.00
 UPDATE | PROD-003  |       79.99 |        89.99 |          10.00
 INSERT | PROD-004  |             |       299.99 |         299.99
```

### Distinguer les trois types d'opérations

```sql
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED AND source.stock > 0 THEN  
    UPDATE SET
        nom = source.nom,
        prix = source.prix,
        stock = source.stock
WHEN MATCHED AND source.stock = 0 THEN
    DELETE
WHEN NOT MATCHED THEN
    INSERT (sku, nom, prix, stock)
    VALUES (source.sku, source.nom, source.prix, source.stock)
RETURNING
    CASE
        WHEN OLD.sku IS NULL THEN 'INSERTED'
        WHEN NEW.sku IS NULL THEN 'DELETED'
        ELSE 'UPDATED'
    END AS operation,
    COALESCE(OLD.sku, NEW.sku) AS sku,
    OLD.nom AS ancien_nom,
    NEW.nom AS nouveau_nom;
```

**Résultat** :
```
 operation |    sku    |    ancien_nom    |     nouveau_nom
-----------+-----------+------------------+---------------------
 UPDATED   | PROD-002  | Souris           | Souris Gaming
 DELETED   | PROD-005  | Webcam           |
 INSERTED  | PROD-006  |                  | Microphone
```

---

## 6.8.7. Cas d'usage pratiques

### Cas 1 : Synchronisation avec audit automatique

```sql
-- Table d'audit
CREATE TABLE audit_merge (
    id SERIAL PRIMARY KEY,
    operation VARCHAR(10),
    table_name VARCHAR(100),
    sku VARCHAR(50),
    ancien_prix NUMERIC(10, 2),
    nouveau_prix NUMERIC(10, 2),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- MERGE avec audit
WITH merge_results AS (
    MERGE INTO produits AS target
    USING produits_import AS source
    ON target.sku = source.sku
    WHEN MATCHED THEN
        UPDATE SET
            nom = source.nom,
            prix = source.prix,
            stock = source.stock
    WHEN NOT MATCHED THEN
        INSERT (sku, nom, prix, stock)
        VALUES (source.sku, source.nom, source.prix, source.stock)
    RETURNING
        CASE
            WHEN OLD.sku IS NULL THEN 'INSERT'
            ELSE 'UPDATE'
        END AS operation,
        COALESCE(OLD.sku, NEW.sku) AS sku,
        OLD.prix AS ancien_prix,
        NEW.prix AS nouveau_prix
)
INSERT INTO audit_merge (operation, table_name, sku, ancien_prix, nouveau_prix)  
SELECT operation, 'produits', sku, ancien_prix, nouveau_prix  
FROM merge_results;  
```

### Cas 2 : Import ETL avec validation

```sql
-- Table de staging avec données brutes
CREATE TABLE staging_clients (
    id_externe VARCHAR(50),
    nom VARCHAR(100),
    email VARCHAR(255),
    telephone VARCHAR(20),
    valide BOOLEAN
);

-- Table de production
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    id_externe VARCHAR(50) UNIQUE,
    nom VARCHAR(100),
    email VARCHAR(255),
    telephone VARCHAR(20),
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    derniere_maj TIMESTAMP
);

-- Import avec validation
MERGE INTO clients AS target  
USING staging_clients AS source  
ON target.id_externe = source.id_externe  
WHEN MATCHED AND source.valide = true THEN  
    UPDATE SET
        nom = source.nom,
        email = source.email,
        telephone = source.telephone,
        derniere_maj = CURRENT_TIMESTAMP
WHEN MATCHED AND source.valide = false THEN
    DELETE
WHEN NOT MATCHED AND source.valide = true THEN
    INSERT (id_externe, nom, email, telephone, derniere_maj)
    VALUES (source.id_externe, source.nom, source.email, source.telephone, CURRENT_TIMESTAMP)
RETURNING
    CASE
        WHEN OLD.id IS NULL THEN 'CREATED'
        WHEN NEW.id IS NULL THEN 'REMOVED'
        ELSE 'UPDATED'
    END AS action,
    NEW.id AS client_id,
    NEW.nom AS nom_client;
```

### Cas 3 : Synchronisation de cache avec TTL

```sql
-- Table de cache
CREATE TABLE cache (
    key VARCHAR(255) PRIMARY KEY,
    value TEXT,
    created_at TIMESTAMP,
    expires_at TIMESTAMP
);

-- Table source avec nouvelles valeurs
CREATE TABLE cache_updates (
    key VARCHAR(255),
    value TEXT,
    ttl_seconds INTEGER DEFAULT 3600
);

-- Synchroniser le cache
MERGE INTO cache AS target  
USING cache_updates AS source  
ON target.key = source.key  
WHEN MATCHED THEN  
    UPDATE SET
        value = source.value,
        expires_at = CURRENT_TIMESTAMP + (source.ttl_seconds || ' seconds')::INTERVAL
WHEN NOT MATCHED THEN
    INSERT (key, value, created_at, expires_at)
    VALUES (
        source.key,
        source.value,
        CURRENT_TIMESTAMP,
        CURRENT_TIMESTAMP + (source.ttl_seconds || ' seconds')::INTERVAL
    )
WHEN NOT MATCHED BY SOURCE AND target.expires_at < CURRENT_TIMESTAMP THEN
    DELETE  -- Supprimer les entrées expirées
RETURNING
    NEW.key,
    CASE
        WHEN OLD.key IS NULL THEN 'CACHED'
        WHEN NEW.key IS NULL THEN 'EXPIRED'
        ELSE 'REFRESHED'
    END AS status;
```

### Cas 4 : Gestion d'inventaire avec seuils

```sql
-- Table principale d'inventaire
CREATE TABLE inventaire (
    produit_id INTEGER PRIMARY KEY,
    quantite INTEGER,
    seuil_alerte INTEGER DEFAULT 10,
    statut VARCHAR(20)
);

-- Mise à jour de l'inventaire depuis les mouvements
CREATE TABLE mouvements_stock (
    produit_id INTEGER,
    ajustement INTEGER  -- Positif = entrée, négatif = sortie
);

-- Synchroniser l'inventaire
MERGE INTO inventaire AS target  
USING (  
    SELECT
        produit_id,
        SUM(ajustement) AS ajustement_total
    FROM mouvements_stock
    GROUP BY produit_id
) AS source
ON target.produit_id = source.produit_id  
WHEN MATCHED THEN  
    UPDATE SET
        quantite = target.quantite + source.ajustement_total,
        statut = CASE
            WHEN target.quantite + source.ajustement_total <= target.seuil_alerte THEN 'ALERTE'
            WHEN target.quantite + source.ajustement_total = 0 THEN 'RUPTURE'
            ELSE 'OK'
        END
WHEN NOT MATCHED THEN
    INSERT (produit_id, quantite, statut)
    VALUES (source.produit_id, source.ajustement_total, 'OK')
RETURNING
    NEW.produit_id,
    OLD.quantite AS ancienne_quantite,
    NEW.quantite AS nouvelle_quantite,
    NEW.statut,
    CASE
        WHEN OLD.statut != NEW.statut THEN true
        ELSE false
    END AS changement_statut;
```

---

## 6.8.8. Sources multiples et requêtes complexes

MERGE accepte n'importe quelle requête SQL comme source, pas seulement une table simple.

### Source avec jointure

```sql
-- Mettre à jour les produits avec des informations de plusieurs tables
MERGE INTO produits AS target  
USING (  
    SELECT
        p.sku,
        p.nom,
        p.prix_achat * (1 + c.marge / 100.0) AS prix_vente,
        s.quantite AS stock
    FROM produits_fournisseur p
    JOIN categories c ON p.categorie_id = c.id
    LEFT JOIN stock_entrepot s ON p.sku = s.sku
) AS source
ON target.sku = source.sku  
WHEN MATCHED THEN  
    UPDATE SET
        nom = source.nom,
        prix = source.prix_vente,
        stock = COALESCE(source.stock, 0)
WHEN NOT MATCHED THEN
    INSERT (sku, nom, prix, stock)
    VALUES (source.sku, source.nom, source.prix_vente, COALESCE(source.stock, 0));
```

### Source avec agrégation

```sql
-- Mettre à jour les statistiques clients
MERGE INTO client_stats AS target  
USING (  
    SELECT
        client_id,
        COUNT(*) AS nb_commandes,
        SUM(montant) AS montant_total,
        MAX(date_commande) AS derniere_commande
    FROM commandes
    WHERE date_commande >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY client_id
) AS source
ON target.client_id = source.client_id  
WHEN MATCHED THEN  
    UPDATE SET
        nb_commandes = source.nb_commandes,
        montant_total = source.montant_total,
        derniere_commande = source.derniere_commande,
        date_maj = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN
    INSERT (client_id, nb_commandes, montant_total, derniere_commande, date_maj)
    VALUES (
        source.client_id,
        source.nb_commandes,
        source.montant_total,
        source.derniere_commande,
        CURRENT_TIMESTAMP
    )
RETURNING
    NEW.client_id,
    OLD.nb_commandes AS anciennes_commandes,
    NEW.nb_commandes AS nouvelles_commandes;
```

### Source avec UNION

```sql
-- Fusionner les données de plusieurs sources
MERGE INTO contacts AS target  
USING (  
    SELECT email, nom, prenom, 'CRM' AS source FROM crm_contacts
    UNION
    SELECT email, nom, prenom, 'Newsletter' AS source FROM newsletter_subscribers
    UNION
    SELECT email, nom, prenom, 'Support' AS source FROM support_tickets
) AS source
ON target.email = source.email  
WHEN MATCHED THEN  
    UPDATE SET
        nom = COALESCE(source.nom, target.nom),
        prenom = COALESCE(source.prenom, target.prenom),
        derniere_source = source.source,
        derniere_maj = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN
    INSERT (email, nom, prenom, source, date_creation)
    VALUES (source.email, source.nom, source.prenom, source.source, CURRENT_TIMESTAMP);
```

---

## 6.8.9. Plusieurs clauses WHEN du même type

PostgreSQL permet d'avoir plusieurs clauses WHEN du même type avec des conditions différentes.

### Exemple : Actions différentes selon les conditions

```sql
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
-- Première condition MATCHED : mise à jour normale
WHEN MATCHED AND source.prix > target.prix THEN
    UPDATE SET
        prix = source.prix,
        stock = source.stock,
        raison_maj = 'Augmentation de prix'
-- Deuxième condition MATCHED : mise à jour avec alerte
WHEN MATCHED AND source.prix < target.prix * 0.8 THEN
    UPDATE SET
        prix = source.prix,
        stock = source.stock,
        raison_maj = 'Baisse importante (>20%)',
        alerte = true
-- Troisième condition MATCHED : mise à jour simple
WHEN MATCHED THEN
    UPDATE SET
        prix = source.prix,
        stock = source.stock,
        raison_maj = 'Mise à jour standard'
-- Insertion des nouveaux produits
WHEN NOT MATCHED THEN
    INSERT (sku, nom, prix, stock, raison_maj)
    VALUES (source.sku, source.nom, source.prix, source.stock, 'Nouveau produit');
```

Les clauses sont évaluées dans l'ordre, et la première qui correspond est exécutée.

---

## 6.8.10. Performance et optimisation

### Index appropriés

MERGE nécessite des index sur les colonnes de jointure (clause ON) :

```sql
-- ✅ Index sur la colonne de jointure
CREATE INDEX idx_produits_sku ON produits(sku);  
CREATE INDEX idx_import_sku ON produits_import(sku);  

-- MERGE sera efficace
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
...;
```

### Comparaison de performance

Pour synchroniser 100 000 lignes entre deux tables :

| Méthode | Durée | Complexité |
|---------|-------|-----------|
| DELETE + INSERT | ~15 secondes | Perd toutes les données temporairement |
| SELECT puis UPDATE/INSERT individuels | ~45 secondes | Nombreuses requêtes |
| MERGE | ~3 secondes | Une seule requête optimisée |

### Batch processing pour très gros volumes

Pour des millions de lignes, traiter par lots :

```sql
-- Traiter par lots de 100 000 lignes
DO $$  
DECLARE  
    batch_size INTEGER := 100000;
    offset_val INTEGER := 0;
    affected_rows INTEGER;
BEGIN
    LOOP
        MERGE INTO produits AS target
        USING (
            SELECT *
            FROM produits_import
            ORDER BY sku
            LIMIT batch_size OFFSET offset_val
        ) AS source
        ON target.sku = source.sku
        WHEN MATCHED THEN
            UPDATE SET
                prix = source.prix,
                stock = source.stock
        WHEN NOT MATCHED THEN
            INSERT (sku, nom, prix, stock)
            VALUES (source.sku, source.nom, source.prix, source.stock);

        GET DIAGNOSTICS affected_rows = ROW_COUNT;
        EXIT WHEN affected_rows = 0;

        offset_val := offset_val + batch_size;
        RAISE NOTICE 'Traité % lignes', offset_val;
    END LOOP;
END $$;
```

---

## 6.8.11. Limitations et considérations

### Limitation 1 : Une seule table cible

```sql
-- ❌ Impossible : MERGE ne peut modifier qu'une seule table à la fois
MERGE INTO produits, categories AS target
...  -- ERROR: syntax error
```

### Limitation 2 : Pas de RETURNING dans les sous-clauses

```sql
-- ❌ RETURNING seulement à la fin du MERGE
MERGE INTO produits AS target  
USING source  
ON condition  
WHEN MATCHED THEN  
    UPDATE SET ... RETURNING id  -- ERROR
WHEN NOT MATCHED THEN
    INSERT ... RETURNING id;  -- ERROR

-- ✅ RETURNING à la fin
MERGE INTO produits AS target
...
RETURNING id, nom;  -- OK
```

### Limitation 3 : OLD/NEW disponibles uniquement dans RETURNING

```sql
-- ❌ Impossible d'utiliser OLD dans WHERE
MERGE INTO produits AS target  
USING source  
ON target.sku = source.sku  
WHEN MATCHED AND OLD.prix < source.prix THEN  -- ERROR  
    UPDATE SET prix = source.prix;

-- ✅ Utiliser les noms de table
WHEN MATCHED AND target.prix < source.prix THEN
    UPDATE SET prix = source.prix;
```

### Considération : Verrous

MERGE prend des verrous sur les lignes affectées :

```sql
-- Session 1
BEGIN;  
MERGE INTO produits AS target  
USING source  
ON target.sku = source.sku  
...;
-- Lignes verrouillées jusqu'au COMMIT

-- Session 2 (en parallèle)
UPDATE produits SET prix = 99 WHERE sku = 'PROD-001';
-- ⏳ Attendra si PROD-001 est affecté par le MERGE de Session 1
```

---

## 6.8.12. MERGE vs Autres approches

### Comparaison complète

| Approche | Complexité | Performance | Flexibilité | Standard SQL |
|----------|-----------|-------------|-------------|-------------|
| **SELECT + UPDATE/INSERT** | ❌ Haute | ⚠️ Moyenne | ✅ Totale | ✅ Oui |
| **ON CONFLICT** | ✅ Basse | ✅ Haute | ⚠️ Limitée | ❌ Non |
| **MERGE** | ⚠️ Moyenne | ✅ Haute | ✅ Haute | ✅ Oui |

### Exemple : Même opération avec les trois approches

#### Approche 1 : SELECT + UPDATE/INSERT (traditionnel)

```sql
-- Complexe et verbeux
DO $$  
DECLARE  
    rec RECORD;
BEGIN
    FOR rec IN SELECT * FROM produits_import LOOP
        IF EXISTS (SELECT 1 FROM produits WHERE sku = rec.sku) THEN
            UPDATE produits
            SET prix = rec.prix, stock = rec.stock
            WHERE sku = rec.sku;
        ELSE
            INSERT INTO produits (sku, nom, prix, stock)
            VALUES (rec.sku, rec.nom, rec.prix, rec.stock);
        END IF;
    END LOOP;
END $$;
```

#### Approche 2 : ON CONFLICT (simple mais limité)

```sql
-- Simple mais ne gère pas les suppressions
INSERT INTO produits (sku, nom, prix, stock)  
SELECT sku, nom, prix, stock  
FROM produits_import  
ON CONFLICT (sku)  
DO UPDATE SET  
    prix = EXCLUDED.prix,
    stock = EXCLUDED.stock;
```

#### Approche 3 : MERGE (équilibré)

```sql
-- Flexible et performant
MERGE INTO produits AS target  
USING produits_import AS source  
ON target.sku = source.sku  
WHEN MATCHED THEN  
    UPDATE SET prix = source.prix, stock = source.stock
WHEN NOT MATCHED THEN
    INSERT (sku, nom, prix, stock)
    VALUES (source.sku, source.nom, source.prix, source.stock)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
```

---

## 6.8.13. Patterns avancés avec OLD/NEW

### Pattern 1 : Rapport de synchronisation détaillé

```sql
WITH merge_report AS (
    MERGE INTO produits AS target
    USING produits_import AS source
    ON target.sku = source.sku
    WHEN MATCHED THEN
        UPDATE SET
            nom = source.nom,
            prix = source.prix,
            stock = source.stock
    WHEN NOT MATCHED THEN
        INSERT (sku, nom, prix, stock)
        VALUES (source.sku, source.nom, source.prix, source.stock)
    WHEN NOT MATCHED BY SOURCE THEN
        DELETE
    RETURNING
        CASE
            WHEN OLD.sku IS NULL THEN 'INSERT'
            WHEN NEW.sku IS NULL THEN 'DELETE'
            ELSE 'UPDATE'
        END AS operation,
        COALESCE(OLD.sku, NEW.sku) AS sku,
        OLD.prix AS ancien_prix,
        NEW.prix AS nouveau_prix,
        OLD.stock AS ancien_stock,
        NEW.stock AS nouveau_stock
)
SELECT
    operation,
    COUNT(*) AS nombre,
    SUM(CASE WHEN operation = 'UPDATE' THEN (nouveau_prix - ancien_prix) * nouveau_stock ELSE 0 END) AS impact_valeur_stock
FROM merge_report  
GROUP BY operation;  
```

**Résultat** :
```
 operation | nombre | impact_valeur_stock
-----------+--------+---------------------
 INSERT    |     15 |                   0
 UPDATE    |     82 |             2450.75
 DELETE    |      3 |                   0
```

### Pattern 2 : Alertes sur modifications importantes

```sql
WITH merge_changes AS (
    MERGE INTO produits AS target
    USING produits_import AS source
    ON target.sku = source.sku
    WHEN MATCHED THEN
        UPDATE SET prix = source.prix, stock = source.stock
    WHEN NOT MATCHED THEN
        INSERT (sku, nom, prix, stock)
        VALUES (source.sku, source.nom, source.prix, source.stock)
    RETURNING
        NEW.sku,
        NEW.nom,
        OLD.prix AS ancien_prix,
        NEW.prix AS nouveau_prix,
        ABS(NEW.prix - COALESCE(OLD.prix, 0)) / NULLIF(OLD.prix, 0) AS variation_pct
)
SELECT
    sku,
    nom,
    ancien_prix,
    nouveau_prix,
    ROUND(variation_pct * 100, 2) AS variation_pourcentage
FROM merge_changes  
WHERE variation_pct > 0.20  -- Plus de 20% de variation  
ORDER BY variation_pct DESC;  
```

### Pattern 3 : Historisation automatique

```sql
-- Table d'historique
CREATE TABLE produits_historique (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(50),
    operation VARCHAR(10),
    ancien_prix NUMERIC(10, 2),
    nouveau_prix NUMERIC(10, 2),
    ancien_stock INTEGER,
    nouveau_stock INTEGER,
    date_modification TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- MERGE avec historisation
WITH merged AS (
    MERGE INTO produits AS target
    USING produits_import AS source
    ON target.sku = source.sku
    WHEN MATCHED THEN
        UPDATE SET prix = source.prix, stock = source.stock
    WHEN NOT MATCHED THEN
        INSERT (sku, nom, prix, stock)
        VALUES (source.sku, source.nom, source.prix, source.stock)
    RETURNING
        CASE
            WHEN OLD.sku IS NULL THEN 'INSERT'
            ELSE 'UPDATE'
        END AS operation,
        COALESCE(OLD.sku, NEW.sku) AS sku,
        OLD.prix AS ancien_prix,
        NEW.prix AS nouveau_prix,
        OLD.stock AS ancien_stock,
        NEW.stock AS nouveau_stock
)
INSERT INTO produits_historique (sku, operation, ancien_prix, nouveau_prix, ancien_stock, nouveau_stock)  
SELECT sku, operation, ancien_prix, nouveau_prix, ancien_stock, nouveau_stock  
FROM merged  
WHERE operation = 'UPDATE';  -- Historiser seulement les modifications  
```

---

## Récapitulatif

### Syntaxe de référence complète

```sql
MERGE INTO table_cible AS target  
USING table_source AS source  
ON condition_jointure  

-- Plusieurs WHEN MATCHED possibles
WHEN MATCHED AND condition1 THEN
    UPDATE SET colonne = valeur
WHEN MATCHED AND condition2 THEN
    DELETE
WHEN MATCHED THEN
    UPDATE SET colonne = valeur

-- WHEN NOT MATCHED BY TARGET (dans source mais pas dans cible)
WHEN NOT MATCHED BY TARGET AND condition THEN
    INSERT (colonnes) VALUES (valeurs)

-- WHEN NOT MATCHED BY SOURCE (dans cible mais pas dans source)
WHEN NOT MATCHED BY SOURCE AND condition THEN
    UPDATE SET colonne = valeur | DELETE

-- RETURNING avec OLD et NEW (PostgreSQL 18)
RETURNING
    CASE
        WHEN OLD.id IS NULL THEN 'INSERT'
        WHEN NEW.id IS NULL THEN 'DELETE'
        ELSE 'UPDATE'
    END AS operation,
    OLD.colonne AS ancienne_valeur,
    NEW.colonne AS nouvelle_valeur;
```

### Tableau de décision : Quand utiliser MERGE

| Besoin | Utilisez MERGE ? |
|--------|-----------------|
| Simple INSERT or UPDATE | ⚠️ ON CONFLICT plus simple |
| Synchronisation complète de tables | ✅ Parfait |
| INSERT + UPDATE + DELETE | ✅ Idéal |
| Conditions complexes multiples | ✅ Très flexible |
| Besoin OLD et NEW dans RETURNING | ✅ PostgreSQL 18+ |
| Portabilité SQL standard | ✅ SQL:2003 |
| Performance critique | ✅ Optimisé |
| Une seule source de données simple | ⚠️ ON CONFLICT peut suffire |

### Points clés à retenir

1. **MERGE = Synchronisation** : Idéal pour maintenir deux tables en sync  
2. **Trois opérations** : INSERT, UPDATE, DELETE en une seule commande  
3. **Conditions flexibles** : Plusieurs WHEN avec des AND  
4. **OLD et NEW (PG 18)** : Distinguer et comparer avant/après dans RETURNING  
5. **Standard SQL** : Portable vers d'autres SGBD  
6. **Performance** : Bien plus rapide que des requêtes séparées  
7. **Source flexible** : Accepte n'importe quelle requête SELECT

---

## Conclusion

`MERGE` est l'outil de choix pour les opérations de synchronisation de données complexes. Introduit dans PostgreSQL 15 et enrichi dans PostgreSQL 18 avec le support de OLD/NEW, il offre :

**Avantages principaux** :
- ✅ **Simplicité** : Une commande au lieu de multiples requêtes  
- ✅ **Atomicité** : Toutes les opérations dans une seule transaction  
- ✅ **Flexibilité** : INSERT + UPDATE + DELETE avec conditions  
- ✅ **Performance** : Optimisé pour les synchronisations de masse  
- ✅ **Standard** : SQL:2003, portable vers d'autres SGBD  
- ✅ **Traçabilité** : OLD/NEW dans RETURNING pour l'audit complet

**Cas d'usage idéaux** :
- ETL et pipelines de données
- Synchronisation de catalogues
- Import de données externes
- Gestion de cache distribué
- Réplication de données entre systèmes

Avec PostgreSQL 18 et le support de OLD/NEW dans RETURNING, MERGE devient encore plus puissant pour l'audit, l'historisation et le reporting des changements de données.

---


⏭️ [Relations et Jointures](/07-relations-et-jointures/README.md)
