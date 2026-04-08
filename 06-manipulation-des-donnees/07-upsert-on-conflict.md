🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.7. "Upsert" : La clause ON CONFLICT (DO NOTHING, DO UPDATE)

## Introduction

L'**UPSERT** (contraction de "UPDATE" et "INSERT") est une opération qui tente d'insérer une ligne, et si elle existe déjà (conflit de clé), soit ignore l'insertion, soit met à jour la ligne existante.

PostgreSQL implémente cette fonctionnalité via la clause `ON CONFLICT`, introduite dans PostgreSQL 9.5. C'est l'une des fonctionnalités les plus élégantes et utiles du langage SQL moderne.

### Qu'est-ce qu'un UPSERT ?

Imaginez ces scénarios courants :
- Vous voulez insérer un utilisateur, mais s'il existe déjà, mettre à jour son profil
- Vous comptez des événements : si l'entrée existe, incrémenter le compteur, sinon créer une nouvelle entrée
- Vous importez des données : insérer les nouvelles, mettre à jour les existantes

Sans UPSERT, ces opérations nécessitent plusieurs requêtes et une logique complexe. Avec UPSERT, c'est une seule requête élégante.

---

## 6.7.1. Le problème sans UPSERT

### Approche naïve (qui échoue)

```sql
-- Table d'utilisateurs
CREATE TABLE utilisateurs (
    email VARCHAR(255) PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    derniere_connexion TIMESTAMP
);

-- Première insertion : OK
INSERT INTO utilisateurs (email, nom, prenom, derniere_connexion)  
VALUES ('alice@example.com', 'Dupont', 'Alice', CURRENT_TIMESTAMP);  

-- Deuxième insertion du même email : ERREUR !
INSERT INTO utilisateurs (email, nom, prenom, derniere_connexion)  
VALUES ('alice@example.com', 'Dupont', 'Alice', CURRENT_TIMESTAMP);  
-- ERROR: duplicate key value violates unique constraint "utilisateurs_pkey"
```

### Approche traditionnelle (complexe et inefficace)

#### Méthode 1 : SELECT puis INSERT ou UPDATE

```sql
-- Étape 1 : Vérifier si l'utilisateur existe
SELECT COUNT(*) FROM utilisateurs WHERE email = 'alice@example.com';

-- Étape 2 : Si existe (résultat > 0) → UPDATE, sinon → INSERT
-- En code applicatif (pseudo-code) :
IF count > 0 THEN
    UPDATE utilisateurs
    SET derniere_connexion = CURRENT_TIMESTAMP
    WHERE email = 'alice@example.com';
ELSE
    INSERT INTO utilisateurs (email, nom, prenom, derniere_connexion)
    VALUES ('alice@example.com', 'Dupont', 'Alice', CURRENT_TIMESTAMP);
END IF;
```

**Problèmes** :
- ❌ Deux requêtes minimum (SELECT + INSERT/UPDATE)  
- ⚠️ Race condition : Entre le SELECT et l'INSERT, un autre processus pourrait insérer la même ligne  
- ❌ Code complexe et verbeux  
- ⚠️ Pas atomique

#### Méthode 2 : UPDATE puis INSERT si échec

```sql
-- Étape 1 : Tenter UPDATE
UPDATE utilisateurs  
SET derniere_connexion = CURRENT_TIMESTAMP  
WHERE email = 'alice@example.com';  

-- Étape 2 : Si aucune ligne modifiée, faire INSERT
-- (nécessite vérifier le nombre de lignes affectées dans le code)
IF row_count = 0 THEN
    INSERT INTO utilisateurs (email, nom, prenom, derniere_connexion)
    VALUES ('alice@example.com', 'Dupont', 'Alice', CURRENT_TIMESTAMP);
END IF;
```

**Problèmes** :
- ❌ Toujours deux requêtes  
- ⚠️ Race condition possible  
- ❌ Logique inversée (on met à jour ce qui n'existe peut-être pas)

#### Méthode 3 : INSERT avec gestion d'exception

```sql
BEGIN;
    -- Tenter INSERT
    INSERT INTO utilisateurs (email, nom, prenom)
    VALUES ('alice@example.com', 'Dupont', 'Alice');
EXCEPTION WHEN unique_violation THEN
    -- Si erreur, faire UPDATE
    UPDATE utilisateurs
    SET derniere_connexion = CURRENT_TIMESTAMP
    WHERE email = 'alice@example.com';
END;
```

**Problèmes** :
- ⚠️ Les exceptions sont coûteuses en performance  
- ❌ Syntaxe lourde  
- ⚠️ Pas disponible en SQL pur (nécessite PL/pgSQL)

---

## 6.7.2. La solution : INSERT ... ON CONFLICT

PostgreSQL propose une solution élégante, performante et atomique :

```sql
INSERT INTO utilisateurs (email, nom, prenom, derniere_connexion)  
VALUES ('alice@example.com', 'Dupont', 'Alice', CURRENT_TIMESTAMP)  
ON CONFLICT (email) DO UPDATE  
SET derniere_connexion = CURRENT_TIMESTAMP;  
```

**Avantages** :
- ✅ Une seule requête (atomique)  
- ✅ Pas de race condition  
- ✅ Performance optimale  
- ✅ Code simple et lisible  
- ✅ SQL standard (pas de code procédural)

### Syntaxe générale

```sql
INSERT INTO table_name (colonnes...)  
VALUES (valeurs...)  
ON CONFLICT (colonne_conflit)  
DO NOTHING | DO UPDATE SET ...  
RETURNING ...;  
```

---

## 6.7.3. ON CONFLICT ... DO NOTHING : Ignorer les doublons

### Concept

`DO NOTHING` signifie : "Si un conflit survient, ne rien faire (ignorer silencieusement l'insertion)".

### Syntaxe basique

```sql
INSERT INTO table_name (colonnes...)  
VALUES (valeurs...)  
ON CONFLICT (colonne_contrainte) DO NOTHING;  
```

### Exemple pratique

```sql
-- Table de tags uniques
CREATE TABLE tags (
    nom VARCHAR(50) PRIMARY KEY,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Première insertion : OK
INSERT INTO tags (nom) VALUES ('postgresql');

-- Deuxième insertion du même tag : ignorée silencieusement
INSERT INTO tags (nom) VALUES ('postgresql')  
ON CONFLICT (nom) DO NOTHING;  

-- Résultat : Le tag existe toujours une seule fois
SELECT * FROM tags WHERE nom = 'postgresql';
```

### Insertion multiple avec DO NOTHING

```sql
-- Insérer plusieurs tags, ignorer les doublons
INSERT INTO tags (nom) VALUES
    ('postgresql'),  -- Existe déjà, ignoré
    ('python'),      -- Nouveau, inséré
    ('javascript'),  -- Nouveau, inséré
    ('postgresql')   -- Doublon dans le batch, ignoré
ON CONFLICT (nom) DO NOTHING;

-- Résultat : Seuls python et javascript sont ajoutés
SELECT * FROM tags ORDER BY nom;
```

### Cas d'usage typiques de DO NOTHING

#### 1. Import de données avec doublons possibles

```sql
-- Importer une liste d'emails sans se soucier des doublons
INSERT INTO newsletter_subscribers (email)  
SELECT email FROM imported_contacts  
ON CONFLICT (email) DO NOTHING;  
```

#### 2. Enregistrement d'événements uniques

```sql
-- Enregistrer qu'un utilisateur a visité une page (une fois maximum)
INSERT INTO page_views (user_id, page_id, date_vue)  
VALUES (123, 456, CURRENT_DATE)  
ON CONFLICT (user_id, page_id, date_vue) DO NOTHING;  
```

#### 3. Création de relations many-to-many sans doublons

```sql
CREATE TABLE utilisateur_roles (
    utilisateur_id INTEGER,
    role_id INTEGER,
    PRIMARY KEY (utilisateur_id, role_id)
);

-- Assigner un rôle, ignorer si déjà assigné
INSERT INTO utilisateur_roles (utilisateur_id, role_id)  
VALUES (42, 1)  
ON CONFLICT (utilisateur_id, role_id) DO NOTHING;  
```

### Vérifier ce qui a été inséré

Avec `RETURNING`, vous pouvez voir quelles lignes ont réellement été insérées :

```sql
-- Insérer avec retour
INSERT INTO tags (nom) VALUES
    ('postgresql'),  -- Existe déjà
    ('rust'),        -- Nouveau
    ('go')           -- Nouveau
ON CONFLICT (nom) DO NOTHING  
RETURNING *;  
```

**Résultat** :
```
  nom  |      date_creation
-------+------------------------
 rust  | 2025-11-19 10:00:00
 go    | 2025-11-19 10:00:00
```

Seules les lignes effectivement insérées sont retournées !

---

## 6.7.4. ON CONFLICT ... DO UPDATE : Mettre à jour en cas de conflit

### Concept

`DO UPDATE` signifie : "Si un conflit survient, mettre à jour la ligne existante au lieu d'insérer".

### Syntaxe basique

```sql
INSERT INTO table_name (colonnes...)  
VALUES (valeurs...)  
ON CONFLICT (colonne_contrainte)  
DO UPDATE SET  
    colonne1 = valeur1,
    colonne2 = valeur2,
    ...;
```

### Exemple simple

```sql
-- Table de compteurs
CREATE TABLE statistiques (
    page VARCHAR(100) PRIMARY KEY,
    vues INTEGER DEFAULT 0,
    derniere_mise_a_jour TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer ou incrémenter le compteur
INSERT INTO statistiques (page, vues, derniere_mise_a_jour)  
VALUES ('accueil', 1, CURRENT_TIMESTAMP)  
ON CONFLICT (page)  
DO UPDATE SET  
    vues = statistiques.vues + 1,
    derniere_mise_a_jour = CURRENT_TIMESTAMP;
```

**Fonctionnement** :
- Si la page n'existe pas : Insère avec vues = 1
- Si la page existe : Incrémente vues de 1 et met à jour la date

### Les mots-clés spéciaux : EXCLUDED

Dans la clause `DO UPDATE`, vous avez accès à deux versions des données :

- **`table_name.colonne`** : Valeur **actuelle** dans la table  
- **`EXCLUDED.colonne`** : Valeur **tentée** d'insertion (celle qui a causé le conflit)

```sql
INSERT INTO statistiques (page, vues)  
VALUES ('accueil', 10)  
ON CONFLICT (page)  
DO UPDATE SET  
    vues = statistiques.vues + EXCLUDED.vues;
    --     └─ Valeur actuelle   └─ Valeur tentée (10)
```

### Exemples avec EXCLUDED

#### Exemple 1 : Mise à jour si la nouvelle valeur est plus récente

```sql
CREATE TABLE produits (
    sku VARCHAR(50) PRIMARY KEY,
    nom VARCHAR(200),
    prix NUMERIC(10, 2),
    derniere_mise_a_jour TIMESTAMP
);

-- Insérer ou mettre à jour si plus récent
INSERT INTO produits (sku, nom, prix, derniere_mise_a_jour)  
VALUES ('PROD-123', 'Ordinateur', 899.99, '2025-11-19 10:00:00')  
ON CONFLICT (sku)  
DO UPDATE SET  
    nom = EXCLUDED.nom,
    prix = EXCLUDED.prix,
    derniere_mise_a_jour = EXCLUDED.derniere_mise_a_jour
WHERE EXCLUDED.derniere_mise_a_jour > produits.derniere_mise_a_jour;
```

La clause `WHERE` est optionnelle et permet de conditionner la mise à jour !

#### Exemple 2 : Conserver le minimum ou maximum

```sql
CREATE TABLE meilleur_score (
    joueur_id INTEGER PRIMARY KEY,
    score INTEGER,
    date_record TIMESTAMP
);

-- Garder seulement le meilleur score
INSERT INTO meilleur_score (joueur_id, score, date_record)  
VALUES (42, 1500, CURRENT_TIMESTAMP)  
ON CONFLICT (joueur_id)  
DO UPDATE SET  
    score = GREATEST(meilleur_score.score, EXCLUDED.score),
    date_record = CASE
        WHEN EXCLUDED.score > meilleur_score.score THEN EXCLUDED.date_record
        ELSE meilleur_score.date_record
    END;
```

Si le nouveau score est meilleur, il est enregistré. Sinon, l'ancien est conservé.

#### Exemple 3 : Fusionner des arrays

```sql
CREATE TABLE utilisateur_tags (
    user_id INTEGER PRIMARY KEY,
    tags TEXT[]
);

-- Ajouter des tags sans doublons
INSERT INTO utilisateur_tags (user_id, tags)  
VALUES (42, ARRAY['postgresql', 'sql'])  
ON CONFLICT (user_id)  
DO UPDATE SET  
    tags = array(
        SELECT DISTINCT unnest(utilisateur_tags.tags || EXCLUDED.tags)
    );
```

Les tags existants et nouveaux sont fusionnés sans doublons.

---

## 6.7.5. Spécifier la contrainte de conflit

### Syntaxe avec nom de colonne

La forme la plus courante :

```sql
ON CONFLICT (colonne)  
ON CONFLICT (colonne1, colonne2, ...)  -- Contrainte composite  
```

**Exemple** :

```sql
CREATE TABLE evenements_uniques (
    user_id INTEGER,
    event_type VARCHAR(50),
    event_date DATE,
    count INTEGER DEFAULT 1,
    PRIMARY KEY (user_id, event_type, event_date)
);

INSERT INTO evenements_uniques (user_id, event_type, event_date, count)  
VALUES (42, 'login', '2025-11-19', 1)  
ON CONFLICT (user_id, event_type, event_date)  
DO UPDATE SET count = evenements_uniques.count + 1;  
```

### Syntaxe avec nom de contrainte

Vous pouvez aussi référencer explicitement une contrainte nommée :

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    username VARCHAR(100),
    CONSTRAINT unique_email UNIQUE (email),
    CONSTRAINT unique_username UNIQUE (username)
);

-- Conflit sur la contrainte email
INSERT INTO utilisateurs (email, username)  
VALUES ('alice@example.com', 'alice42')  
ON CONFLICT ON CONSTRAINT unique_email  
DO UPDATE SET username = EXCLUDED.username;  

-- Conflit sur la contrainte username
INSERT INTO utilisateurs (email, username)  
VALUES ('bob@example.com', 'alice42')  
ON CONFLICT ON CONSTRAINT unique_username  
DO NOTHING;  
```

### Contraintes partielles (index avec WHERE)

Si vous avez un index partiel (unique avec une clause WHERE) :

```sql
-- Index unique partiel : email unique seulement pour les utilisateurs actifs
CREATE UNIQUE INDEX unique_active_email  
ON utilisateurs (email)  
WHERE statut = 'actif';  

-- ON CONFLICT doit inclure la condition de l'index
INSERT INTO utilisateurs (email, username, statut)  
VALUES ('alice@example.com', 'alice42', 'actif')  
ON CONFLICT (email) WHERE statut = 'actif'  
DO UPDATE SET username = EXCLUDED.username;  
```

---

## 6.7.6. Cas d'usage pratiques

### Cas 1 : Cache / Dictionnaire de valeurs

```sql
CREATE TABLE cache (
    key VARCHAR(255) PRIMARY KEY,
    value TEXT,
    expire_at TIMESTAMP
);

-- Mettre en cache (ou rafraîchir si existe)
INSERT INTO cache (key, value, expire_at)  
VALUES ('user:42:profile', '{"name": "Alice", "age": 30}', NOW() + INTERVAL '1 hour')  
ON CONFLICT (key)  
DO UPDATE SET  
    value = EXCLUDED.value,
    expire_at = EXCLUDED.expire_at;
```

### Cas 2 : Agrégation en temps réel

```sql
CREATE TABLE ventes_journalieres (
    produit_id INTEGER,
    date_vente DATE,
    quantite_vendue INTEGER DEFAULT 0,
    montant_total NUMERIC(12, 2) DEFAULT 0,
    PRIMARY KEY (produit_id, date_vente)
);

-- Enregistrer une vente
INSERT INTO ventes_journalieres (produit_id, date_vente, quantite_vendue, montant_total)  
VALUES (123, CURRENT_DATE, 5, 249.95)  
ON CONFLICT (produit_id, date_vente)  
DO UPDATE SET  
    quantite_vendue = ventes_journalieres.quantite_vendue + EXCLUDED.quantite_vendue,
    montant_total = ventes_journalieres.montant_total + EXCLUDED.montant_total;
```

Chaque vente met à jour les totaux du jour automatiquement.

### Cas 3 : Synchronisation de données

```sql
-- Table principale
CREATE TABLE contacts (
    email VARCHAR(255) PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    telephone VARCHAR(20),
    derniere_synchro TIMESTAMP
);

-- Synchroniser avec une source externe
INSERT INTO contacts (email, nom, prenom, telephone, derniere_synchro)  
SELECT email, nom, prenom, telephone, CURRENT_TIMESTAMP  
FROM contacts_externes  
ON CONFLICT (email)  
DO UPDATE SET  
    nom = EXCLUDED.nom,
    prenom = EXCLUDED.prenom,
    telephone = EXCLUDED.telephone,
    derniere_synchro = EXCLUDED.derniere_synchro;
```

### Cas 4 : Présence utilisateur / Heartbeat

```sql
CREATE TABLE utilisateurs_en_ligne (
    user_id INTEGER PRIMARY KEY,
    derniere_activite TIMESTAMP,
    ip_address INET,
    user_agent TEXT
);

-- Mettre à jour la présence
INSERT INTO utilisateurs_en_ligne (user_id, derniere_activite, ip_address, user_agent)  
VALUES (42, CURRENT_TIMESTAMP, '192.168.1.100', 'Mozilla/5.0...')  
ON CONFLICT (user_id)  
DO UPDATE SET  
    derniere_activite = EXCLUDED.derniere_activite,
    ip_address = EXCLUDED.ip_address,
    user_agent = EXCLUDED.user_agent;
```

### Cas 5 : Gestion de sessions

```sql
CREATE TABLE sessions (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id INTEGER,
    data JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Créer ou mettre à jour une session
INSERT INTO sessions (session_id, user_id, data, updated_at)  
VALUES ('abc123...', 42, '{"cart": ["item1", "item2"]}', CURRENT_TIMESTAMP)  
ON CONFLICT (session_id)  
DO UPDATE SET  
    data = EXCLUDED.data,
    updated_at = EXCLUDED.updated_at;
```

---

## 6.7.7. Combinaison avec RETURNING

RETURNING fonctionne parfaitement avec ON CONFLICT et permet de distinguer les insertions des mises à jour.

### Savoir ce qui a été fait

```sql
-- Version sans distinction
INSERT INTO statistiques (page, vues)  
VALUES ('accueil', 1)  
ON CONFLICT (page)  
DO UPDATE SET vues = statistiques.vues + 1  
RETURNING page, vues;  

-- Résultat : On voit la valeur finale mais pas l'action effectuée
```

### Distinguer INSERT et UPDATE (avec PostgreSQL 18)

Avec PostgreSQL 18 et le support de OLD/NEW :

```sql
INSERT INTO statistiques (page, vues, derniere_maj)  
VALUES ('accueil', 1, CURRENT_TIMESTAMP)  
ON CONFLICT (page)  
DO UPDATE SET  
    vues = statistiques.vues + 1,
    derniere_maj = CURRENT_TIMESTAMP
RETURNING
    page,
    CASE
        WHEN OLD.page IS NULL THEN 'INSERT'
        ELSE 'UPDATE'
    END AS action,
    COALESCE(OLD.vues, 0) AS anciennes_vues,
    NEW.vues AS nouvelles_vues;
```

**Résultat possible** :
```
  page   | action | anciennes_vues | nouvelles_vues
---------+--------+----------------+----------------
 accueil | UPDATE |             42 |             43
```

### Pattern : Log des upserts

```sql
WITH upserted AS (
    INSERT INTO produits (sku, nom, prix, stock)
    VALUES ('PROD-123', 'Ordinateur', 899.99, 10)
    ON CONFLICT (sku)
    DO UPDATE SET
        nom = EXCLUDED.nom,
        prix = EXCLUDED.prix,
        stock = EXCLUDED.stock
    RETURNING
        sku,
        nom,
        prix,
        CASE WHEN OLD.sku IS NULL THEN 'created' ELSE 'updated' END AS action
)
INSERT INTO audit_produits (sku, action, timestamp)  
SELECT sku, action, CURRENT_TIMESTAMP  
FROM upserted;  
```

---

## 6.7.8. Performance et optimisation

### Comparaison de performance

#### Approche traditionnelle (2 requêtes)

```sql
-- Mesure de temps pour 10 000 opérations

-- Approche 1 : SELECT puis INSERT ou UPDATE
DO $$  
DECLARE  
    i INTEGER;
BEGIN
    FOR i IN 1..10000 LOOP
        -- SELECT
        IF EXISTS (SELECT 1 FROM stats WHERE id = i) THEN
            -- UPDATE
            UPDATE stats SET counter = counter + 1 WHERE id = i;
        ELSE
            -- INSERT
            INSERT INTO stats (id, counter) VALUES (i, 1);
        END IF;
    END LOOP;
END $$;

-- Durée : ~15 secondes
```

#### Approche UPSERT (1 requête)

```sql
-- Mesure de temps pour 10 000 opérations

DO $$  
DECLARE  
    i INTEGER;
BEGIN
    FOR i IN 1..10000 LOOP
        -- UPSERT
        INSERT INTO stats (id, counter)
        VALUES (i, 1)
        ON CONFLICT (id)
        DO UPDATE SET counter = stats.counter + 1;
    END LOOP;
END $$;

-- Durée : ~5 secondes
```

**Gain : 3× plus rapide**

### Batch UPSERT (encore plus rapide)

```sql
-- Au lieu de 10 000 requêtes, une seule avec 10 000 lignes
INSERT INTO stats (id, counter)  
SELECT generate_series(1, 10000), 1  
ON CONFLICT (id)  
DO UPDATE SET counter = stats.counter + 1;  

-- Durée : ~0.5 seconde (30× plus rapide que l'approche traditionnelle)
```

### Optimisations

#### 1. Index appropriés

`ON CONFLICT` nécessite un index unique sur la ou les colonnes de conflit :

```sql
-- ✅ Index unique obligatoire
CREATE UNIQUE INDEX idx_email ON utilisateurs (email);

INSERT INTO utilisateurs (email, nom)  
VALUES ('alice@example.com', 'Alice')  
ON CONFLICT (email) DO NOTHING;  -- Fonctionne  

-- ❌ Sans index unique : erreur
ON CONFLICT (nom) DO NOTHING;  -- ERROR: there is no unique constraint matching...
```

#### 2. Préférer les batch operations

```sql
-- ❌ Lent : Plusieurs requêtes
FOR each row IN data:
    INSERT ... ON CONFLICT ... DO UPDATE;

-- ✅ Rapide : Une seule requête
INSERT INTO table  
SELECT * FROM unnest(array_of_data)  
ON CONFLICT (key) DO UPDATE SET ...;  
```

#### 3. Utiliser des CTE pour la clarté

```sql
WITH data_to_upsert AS (
    SELECT
        email,
        nom,
        prenom,
        derniere_connexion
    FROM staging_table
    WHERE date_import = CURRENT_DATE
)
INSERT INTO utilisateurs (email, nom, prenom, derniere_connexion)  
SELECT * FROM data_to_upsert  
ON CONFLICT (email)  
DO UPDATE SET  
    nom = EXCLUDED.nom,
    prenom = EXCLUDED.prenom,
    derniere_connexion = EXCLUDED.derniere_connexion;
```

---

## 6.7.9. Limitations et pièges

### Limitation 1 : Ne fonctionne qu'avec UNIQUE ou PRIMARY KEY

```sql
-- ❌ Impossible : colonne sans contrainte unique
CREATE TABLE logs (
    id SERIAL,
    message TEXT,
    level VARCHAR(20)
);

INSERT INTO logs (message, level)  
VALUES ('Error occurred', 'ERROR')  
ON CONFLICT (level) DO NOTHING;  
-- ERROR: there is no unique or exclusion constraint matching the ON CONFLICT specification

-- ✅ Solution : Ajouter une contrainte unique
ALTER TABLE logs ADD CONSTRAINT unique_message_level UNIQUE (message, level);
```

### Limitation 2 : WHERE ne peut référencer que la table cible

```sql
-- ❌ Impossible : WHERE référençant une autre table
INSERT INTO produits (sku, prix)  
VALUES ('PROD-123', 99.99)  
ON CONFLICT (sku)  
DO UPDATE SET prix = EXCLUDED.prix  
WHERE EXCLUDED.prix < (SELECT prix_max FROM config);  -- Erreur  

-- ✅ Solution : Utiliser une sous-requête dans SET
INSERT INTO produits (sku, prix)  
VALUES ('PROD-123', 99.99)  
ON CONFLICT (sku)  
DO UPDATE SET prix = CASE  
    WHEN EXCLUDED.prix < (SELECT prix_max FROM config) THEN EXCLUDED.prix
    ELSE produits.prix
END;
```

### Piège 1 : Oublier EXCLUDED

```sql
-- ❌ Erreur courante : Utiliser directement les valeurs
INSERT INTO stats (page, vues)  
VALUES ('accueil', 100)  
ON CONFLICT (page)  
DO UPDATE SET vues = vues + 100;  -- Quelle variable "vues" ?  

-- ✅ Correct : Être explicite
INSERT INTO stats (page, vues)  
VALUES ('accueil', 100)  
ON CONFLICT (page)  
DO UPDATE SET vues = stats.vues + EXCLUDED.vues;  
```

### Piège 2 : Conflit sur la mauvaise contrainte

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    username VARCHAR(100) UNIQUE
);

-- Conflit sur email, mais on spécifie username !
INSERT INTO utilisateurs (email, username)  
VALUES ('alice@example.com', 'alice')  
ON CONFLICT (username) DO NOTHING;  
-- Si le conflit est sur email, l'erreur se produit quand même !

-- ✅ Solution : Gérer les deux contraintes
INSERT INTO utilisateurs (email, username)  
VALUES ('alice@example.com', 'alice')  
ON CONFLICT (email) DO UPDATE SET username = EXCLUDED.username  
ON CONFLICT (username) DO UPDATE SET email = EXCLUDED.email;  
-- ERROR: syntax error (impossible d'avoir deux ON CONFLICT)

-- ✅ Vraie solution : Deux requêtes ou logique métier
```

### Piège 3 : Performances avec de nombreux index

```sql
-- Table avec beaucoup d'index uniques
CREATE TABLE entite_complexe (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) UNIQUE,
    ref_externe VARCHAR(50) UNIQUE,
    email VARCHAR(255) UNIQUE,
    telephone VARCHAR(20) UNIQUE
);

-- Chaque UPSERT doit vérifier tous les index
INSERT INTO entite_complexe (code, ref_externe, email, telephone)  
VALUES ('C001', 'REF001', 'test@ex.com', '0102030405')  
ON CONFLICT (code) DO UPDATE SET ...;  
-- → Vérifie aussi ref_externe, email, telephone (surcoût)
```

---

## 6.7.10. Patterns avancés

### Pattern 1 : Upsert conditionnel avec logique complexe

```sql
-- Mettre à jour seulement si les données sont plus récentes
INSERT INTO produits (sku, nom, prix, maj_timestamp)  
VALUES ('PROD-123', 'Nouveau nom', 99.99, '2025-11-19 12:00:00')  
ON CONFLICT (sku)  
DO UPDATE SET  
    nom = CASE
        WHEN EXCLUDED.maj_timestamp > produits.maj_timestamp THEN EXCLUDED.nom
        ELSE produits.nom
    END,
    prix = CASE
        WHEN EXCLUDED.maj_timestamp > produits.maj_timestamp THEN EXCLUDED.prix
        ELSE produits.prix
    END,
    maj_timestamp = GREATEST(produits.maj_timestamp, EXCLUDED.maj_timestamp)
WHERE EXCLUDED.maj_timestamp > produits.maj_timestamp;
```

### Pattern 2 : Upsert avec calculs agrégés

```sql
-- Table de métriques agrégées
CREATE TABLE metriques_horaires (
    metric_name VARCHAR(100),
    hour TIMESTAMP,
    count BIGINT DEFAULT 0,
    sum_value NUMERIC(15, 2) DEFAULT 0,
    min_value NUMERIC(15, 2),
    max_value NUMERIC(15, 2),
    PRIMARY KEY (metric_name, hour)
);

-- Intégrer une nouvelle mesure
INSERT INTO metriques_horaires (metric_name, hour, count, sum_value, min_value, max_value)  
VALUES ('temperature', date_trunc('hour', CURRENT_TIMESTAMP), 1, 23.5, 23.5, 23.5)  
ON CONFLICT (metric_name, hour)  
DO UPDATE SET  
    count = metriques_horaires.count + 1,
    sum_value = metriques_horaires.sum_value + EXCLUDED.sum_value,
    min_value = LEAST(metriques_horaires.min_value, EXCLUDED.min_value),
    max_value = GREATEST(metriques_horaires.max_value, EXCLUDED.max_value);
```

### Pattern 3 : Upsert avec JSONB merge

```sql
-- Fusionner des données JSON
CREATE TABLE user_preferences (
    user_id INTEGER PRIMARY KEY,
    preferences JSONB DEFAULT '{}'::jsonb
);

-- Ajouter/mettre à jour des préférences
INSERT INTO user_preferences (user_id, preferences)  
VALUES (42, '{"theme": "dark", "language": "fr"}'::jsonb)  
ON CONFLICT (user_id)  
DO UPDATE SET  
    preferences = user_preferences.preferences || EXCLUDED.preferences;
    -- L'opérateur || fusionne les JSONB
```

### Pattern 4 : Idempotence garantie

```sql
-- Importer des données de manière idempotente
-- (peut être rejoué sans effets de bord)

CREATE TABLE import_log (
    import_id UUID PRIMARY KEY,
    filename VARCHAR(255),
    imported_at TIMESTAMP,
    row_count INTEGER
);

-- Enregistrer l'import de manière idempotente
INSERT INTO import_log (import_id, filename, imported_at, row_count)  
VALUES (  
    'a1b2c3d4-...',  -- UUID fixe pour cet import
    'data_2025-11-19.csv',
    CURRENT_TIMESTAMP,
    10000
)
ON CONFLICT (import_id) DO NOTHING;
-- Si rejoué, n'importe pas → idempotent
```

### Pattern 5 : Race-safe counters

```sql
-- Compteur thread-safe sans verrous
CREATE TABLE counters (
    name VARCHAR(50) PRIMARY KEY,
    value BIGINT DEFAULT 0
);

-- Incrémenter de manière atomique
INSERT INTO counters (name, value)  
VALUES ('page_views', 1)  
ON CONFLICT (name)  
DO UPDATE SET value = counters.value + EXCLUDED.value;  

-- Peut être appelé en parallèle par plusieurs processus sans problème
```

---

## 6.7.11. Comparaison avec MERGE (PostgreSQL 18)

PostgreSQL 18 introduit la commande `MERGE`, qui est une alternative SQL standard à ON CONFLICT.

### MERGE vs ON CONFLICT

| Aspect | ON CONFLICT | MERGE |
|--------|-------------|-------|
| **Standard SQL** | ❌ PostgreSQL spécifique | ✅ Standard SQL:2016 |
| **Simplicité** | ✅ Syntaxe simple | ⚠️ Plus verbeux |
| **Flexibilité** | ⚠️ Limitée à INSERT | ✅ Peut gérer DELETE aussi |
| **Performance** | ✅ Très performant | ✅ Comparable |
| **Cas d'usage** | Upsert simple | Synchronisation complexe |

### Équivalence

```sql
-- Avec ON CONFLICT
INSERT INTO stats (page, vues)  
VALUES ('accueil', 1)  
ON CONFLICT (page)  
DO UPDATE SET vues = stats.vues + 1;  

-- Avec MERGE (PostgreSQL 18)
MERGE INTO stats AS target  
USING (VALUES ('accueil', 1)) AS source (page, vues)  
ON target.page = source.page  
WHEN MATCHED THEN  
    UPDATE SET vues = target.vues + source.vues
WHEN NOT MATCHED THEN
    INSERT (page, vues) VALUES (source.page, source.vues);
```

### Quand utiliser quoi ?

**Utilisez ON CONFLICT** :
- ✅ Pour des upserts simples et directs  
- ✅ Quand la syntaxe courte est préférable  
- ✅ Pour de meilleures performances sur des cas simples

**Utilisez MERGE** :
- ✅ Pour de la synchronisation complexe (INSERT/UPDATE/DELETE)  
- ✅ Pour la portabilité SQL standard  
- ✅ Quand plusieurs sources doivent être fusionnées

---

## Récapitulatif

### Syntaxe de référence rapide

```sql
-- DO NOTHING : Ignorer les doublons
INSERT INTO table (colonnes)  
VALUES (valeurs)  
ON CONFLICT (colonne_unique) DO NOTHING;  

-- DO UPDATE : Mettre à jour en cas de conflit
INSERT INTO table (col1, col2, col3)  
VALUES (val1, val2, val3)  
ON CONFLICT (colonne_unique)  
DO UPDATE SET  
    col2 = EXCLUDED.col2,
    col3 = table.col3 + EXCLUDED.col3;

-- Avec WHERE conditionnel
INSERT INTO table (colonnes)  
VALUES (valeurs)  
ON CONFLICT (colonne_unique)  
DO UPDATE SET colonnes = valeurs  
WHERE condition;  

-- Avec RETURNING
INSERT INTO table (colonnes)  
VALUES (valeurs)  
ON CONFLICT (colonne_unique)  
DO UPDATE SET colonnes = valeurs  
RETURNING *;  

-- Contrainte composite
ON CONFLICT (col1, col2, col3) DO ...

-- Par nom de contrainte
ON CONFLICT ON CONSTRAINT nom_contrainte DO ...
```

### Tableau de décision

| Besoin | Solution |
|--------|----------|
| Ignorer les doublons | `ON CONFLICT ... DO NOTHING` |
| Remplacer les anciennes valeurs | `ON CONFLICT ... DO UPDATE SET col = EXCLUDED.col` |
| Incrémenter un compteur | `DO UPDATE SET counter = table.counter + EXCLUDED.counter` |
| Garder le max/min | `DO UPDATE SET val = GREATEST/LEAST(...)` |
| Fusionner des données | `DO UPDATE SET jsonb_col = table.jsonb_col \|\| EXCLUDED.jsonb_col` |
| Mise à jour conditionnelle | `DO UPDATE SET ... WHERE condition` |

### Points clés à retenir

1. **UPSERT = INSERT or UPDATE** en une seule opération atomique  
2. **ON CONFLICT** nécessite un index UNIQUE ou PRIMARY KEY  
3. **DO NOTHING** ignore silencieusement les doublons  
4. **DO UPDATE** met à jour la ligne existante  
5. **EXCLUDED** référence les valeurs tentées d'insertion  
6. **WHERE** optionnel conditionne la mise à jour  
7. **RETURNING** fonctionne avec ON CONFLICT  
8. Bien plus performant que SELECT puis INSERT/UPDATE

---

## Conclusion

La clause `ON CONFLICT` est l'une des fonctionnalités les plus utiles et élégantes de PostgreSQL moderne. Elle transforme des opérations qui nécessitaient auparavant plusieurs requêtes, de la logique applicative complexe et des risques de race conditions en une seule commande SQL simple, atomique et performante.

**Bénéfices principaux** :
- ✅ **Simplicité** : Une ligne de code au lieu de dizaines  
- ✅ **Atomicité** : Opération garantie sans race condition  
- ✅ **Performance** : 3× à 10× plus rapide que les approches traditionnelles  
- ✅ **Lisibilité** : Intent clair dans le code SQL  
- ✅ **Sécurité** : Pas besoin de gestion d'exception complexe

Maîtriser `ON CONFLICT` est essentiel pour tout développeur travaillant avec PostgreSQL, que ce soit pour de l'import de données, de la synchronisation, du caching, ou de l'agrégation en temps réel.

---


⏭️ [MERGE : Consolidation de données (avec OLD/NEW)](/06-manipulation-des-donnees/08-merge-consolidation.md)
