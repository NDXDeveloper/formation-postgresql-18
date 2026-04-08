🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5. Séquences (SEQUENCE) et Génération Automatique

## Introduction

L'une des fonctionnalités les plus courantes dans les bases de données est la génération automatique d'identifiants uniques pour les lignes d'une table. PostgreSQL offre plusieurs mécanismes pour cela, principalement basés sur les **séquences**.

Dans cette section, nous allons explorer :
- **SEQUENCE** : Le mécanisme sous-jacent de génération de nombres  
- **SERIAL** : Le raccourci PostgreSQL pour les identifiants auto-incrémentés  
- **IDENTITY** : Le standard SQL pour la génération automatique (PostgreSQL 10+)
- Les fonctions de manipulation des séquences
- Les bonnes pratiques et pièges à éviter

---

## Qu'est-ce qu'une Séquence ?

### Définition

Une **séquence** (SEQUENCE) est un objet de base de données qui génère une suite de nombres entiers uniques. C'est un **compteur automatique** qui s'incrémente à chaque utilisation.

### Analogie

Imaginez une machine à tickets dans une boulangerie :
- Chaque client appuie sur un bouton
- La machine distribue un numéro : 1, 2, 3, 4...
- Le numéro ne peut jamais être donné deux fois
- Le compteur ne recule jamais

**Une séquence PostgreSQL fonctionne exactement de la même manière.**

### Caractéristiques Importantes

```
✓ Génération automatique de nombres uniques
✓ Incrémentation garantie (pas de doublons)
✓ Indépendante des transactions (pas de rollback)
✓ Thread-safe (utilisable en concurrence)
✓ Peut avoir des "trous" dans la numérotation
```

---

## SERIAL : Le Raccourci PostgreSQL

### Qu'est-ce que SERIAL ?

`SERIAL` n'est **pas un vrai type** de données. C'est un **raccourci** (syntactic sugar) qui crée automatiquement :
1. Une séquence  
2. Une colonne INTEGER avec valeur par défaut provenant de la séquence  
3. Une dépendance entre la colonne et la séquence

### Exemple de Base

```sql
-- Créer une table avec SERIAL
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255)
);

-- PostgreSQL crée automatiquement :
-- 1. Une séquence nommée "utilisateurs_id_seq"
-- 2. Une colonne id de type INTEGER
-- 3. Une valeur par défaut : nextval('utilisateurs_id_seq')
```

### Ce que SERIAL Fait Réellement

Le code ci-dessus est équivalent à :

```sql
-- Version explicite (ce que SERIAL fait automatiquement)
CREATE SEQUENCE utilisateurs_id_seq;

CREATE TABLE utilisateurs (
    id INTEGER NOT NULL DEFAULT nextval('utilisateurs_id_seq'),
    nom VARCHAR(100),
    email VARCHAR(255)
);

ALTER SEQUENCE utilisateurs_id_seq OWNED BY utilisateurs.id;
```

### Les Trois Variantes de SERIAL

PostgreSQL offre trois tailles de SERIAL :

| Type | Type Réel | Taille | Plage | Utilisation |
|------|-----------|--------|-------|-------------|
| `SMALLSERIAL` | SMALLINT | 2 octets | 1 à 32 767 | Très petites tables |
| `SERIAL` | INTEGER | 4 octets | 1 à 2 147 483 647 | **Standard** (2 milliards) |
| `BIGSERIAL` | BIGINT | 8 octets | 1 à 9 × 10¹⁸ | Très grandes tables |

```sql
-- Exemples
CREATE TABLE petite_table (
    id SMALLSERIAL PRIMARY KEY  -- Max 32 767 lignes
);

CREATE TABLE table_normale (
    id SERIAL PRIMARY KEY  -- Max 2 milliards de lignes
);

CREATE TABLE logs_massifs (
    id BIGSERIAL PRIMARY KEY  -- Max 9 quintillions de lignes
);
```

**Recommandation :** Utilisez `SERIAL` par défaut, `BIGSERIAL` pour les logs et données massives.

---

## Utilisation des Séquences avec SERIAL

### Insertion Automatique

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    prix NUMERIC(10, 2)
);

-- Insertion sans spécifier l'ID
INSERT INTO produits (nom, prix) VALUES ('Ordinateur', 999.99);  
INSERT INTO produits (nom, prix) VALUES ('Souris', 29.99);  
INSERT INTO produits (nom, prix) VALUES ('Clavier', 79.99);  

SELECT * FROM produits;
```

Résultat :
```
 id |     nom     |  prix
----+-------------+--------
  1 | Ordinateur  | 999.99
  2 | Souris      |  29.99
  3 | Clavier     |  79.99
```

PostgreSQL a automatiquement généré les IDs : 1, 2, 3.

### Voir la Séquence Créée

```sql
-- Lister les séquences
\ds

-- Ou avec SQL
SELECT sequencename, last_value  
FROM pg_sequences  
WHERE schemaname = 'public';  

-- Voir les détails d'une séquence
\d produits_id_seq

-- Ou avec SQL
SELECT * FROM produits_id_seq;
```

---

## Créer et Gérer des Séquences Manuellement

### Création Manuelle d'une Séquence

```sql
-- Syntaxe de base
CREATE SEQUENCE nom_sequence;

-- Avec options
CREATE SEQUENCE seq_commandes
    START WITH 1000        -- Commence à 1000
    INCREMENT BY 1         -- Incrémente de 1
    MINVALUE 1000         -- Valeur minimale
    MAXVALUE 9999999      -- Valeur maximale
    CACHE 10;             -- Cache 10 valeurs en mémoire

-- Séquence décrémentante
CREATE SEQUENCE seq_countdown
    START WITH 100
    INCREMENT BY -1       -- Décrémente
    MINVALUE 1
    MAXVALUE 100;

-- Séquence cyclique (recommence à 1 après max)
CREATE SEQUENCE seq_cyclique
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 10
    CYCLE;               -- Recommence après 10
```

### Options de CREATE SEQUENCE

| Option | Description | Valeur par Défaut |
|--------|-------------|-------------------|
| `START WITH n` | Valeur de départ | 1 |
| `INCREMENT BY n` | Pas d'incrémentation | 1 |
| `MINVALUE n` | Valeur minimale | 1 |
| `MAXVALUE n` | Valeur maximale | 9223372036854775807 |
| `CACHE n` | Nombre de valeurs en cache | 1 |
| `CYCLE` | Recommence après max | NO CYCLE |
| `OWNED BY table.column` | Propriété de la séquence | NONE |

### Utiliser une Séquence Manuelle

```sql
-- Créer une séquence pour des numéros de facture
CREATE SEQUENCE seq_numero_facture
    START WITH 2025001
    INCREMENT BY 1;

CREATE TABLE factures (
    id SERIAL PRIMARY KEY,
    numero_facture VARCHAR(20) DEFAULT 'FAC-' || nextval('seq_numero_facture'),
    client_id INTEGER,
    montant NUMERIC(10, 2),
    date_facture DATE DEFAULT CURRENT_DATE
);

-- Insertion
INSERT INTO factures (client_id, montant) VALUES (1, 150.00);  
INSERT INTO factures (client_id, montant) VALUES (2, 250.00);  

SELECT id, numero_facture, montant FROM factures;
```

Résultat :
```
 id | numero_facture | montant
----+----------------+---------
  1 | FAC-2025001   | 150.00
  2 | FAC-2025002   | 250.00
```

---

## Fonctions de Manipulation des Séquences

### nextval() : Obtenir la Prochaine Valeur

```sql
-- nextval() incrémente et retourne la nouvelle valeur
SELECT nextval('seq_numero_facture');  -- 2025003  
SELECT nextval('seq_numero_facture');  -- 2025004  
SELECT nextval('seq_numero_facture');  -- 2025005  

-- Chaque appel incrémente la séquence
```

**Important :** `nextval()` **consomme** une valeur, même hors transaction.

### currval() : Valeur Courante (Session)

```sql
-- currval() retourne la dernière valeur obtenue par nextval() dans CETTE session
SELECT currval('seq_numero_facture');  -- 2025005

-- ⚠️ Erreur si nextval() n'a jamais été appelé dans cette session
SELECT currval('seq_autre');
-- ERROR: currval of sequence "seq_autre" is not yet defined in this session
```

**Important :** `currval()` est spécifique à la **session** (connexion).

### setval() : Définir une Valeur

```sql
-- setval(sequence, valeur) : définit la valeur de la séquence
SELECT setval('seq_numero_facture', 2030000);

-- Prochaine valeur sera 2030001
SELECT nextval('seq_numero_facture');  -- 2030001

-- setval() avec is_called
SELECT setval('seq_numero_facture', 5000, true);   -- Prochaine valeur : 5001  
SELECT setval('seq_numero_facture', 5000, false);  -- Prochaine valeur : 5000  
```

**Cas d'usage :** Réinitialiser ou synchroniser une séquence.

### lastval() : Dernière Valeur (Toutes Séquences)

```sql
-- lastval() retourne la dernière valeur obtenue (toutes séquences confondues)
SELECT nextval('produits_id_seq');  -- 4  
SELECT nextval('seq_numero_facture');  -- 2030002  
SELECT lastval();  -- 2030002 (dernière valeur appelée)  
```

**Note :** `lastval()` est rarement utilisé.

### Exemple Complet

```sql
-- Créer une séquence
CREATE SEQUENCE demo_seq START WITH 100;

-- État initial
SELECT last_value FROM demo_seq;  -- 100 (valeur courante, pas encore utilisée)

-- Obtenir des valeurs
SELECT nextval('demo_seq');  -- 100 (première utilisation)  
SELECT nextval('demo_seq');  -- 101  
SELECT nextval('demo_seq');  -- 102  

-- Valeur courante dans cette session
SELECT currval('demo_seq');  -- 102

-- Réinitialiser
SELECT setval('demo_seq', 1, false);  -- Recommence à 1

SELECT nextval('demo_seq');  -- 1
```

---

## Modifier une Séquence Existante

```sql
-- ALTER SEQUENCE pour modifier les paramètres
ALTER SEQUENCE produits_id_seq RESTART WITH 1000;

-- Changer l'incrément
ALTER SEQUENCE produits_id_seq INCREMENT BY 10;

-- Changer le maximum
ALTER SEQUENCE produits_id_seq MAXVALUE 999999;

-- Ajouter un cycle
ALTER SEQUENCE produits_id_seq CYCLE;

-- Changer le cache
ALTER SEQUENCE produits_id_seq CACHE 50;

-- Changer le propriétaire
ALTER SEQUENCE produits_id_seq OWNED BY produits.id;

-- Renommer
ALTER SEQUENCE produits_id_seq RENAME TO seq_produits;
```

---

## Réinitialiser une Séquence

### Réinitialiser à une Valeur Spécifique

```sql
-- Réinitialiser à 1
ALTER SEQUENCE produits_id_seq RESTART WITH 1;

-- Ou avec setval
SELECT setval('produits_id_seq', 1, false);
```

### Réinitialiser Automatiquement au Maximum Actuel

```sql
-- Synchroniser la séquence avec la valeur max de la table
SELECT setval('produits_id_seq', (SELECT MAX(id) FROM produits));

-- Ou directement avec ALTER
ALTER SEQUENCE produits_id_seq RESTART WITH (SELECT MAX(id) + 1 FROM produits);
```

**Cas d'usage :** Après import de données avec IDs explicites.

### Réinitialiser Toutes les Séquences d'une Table

```sql
-- Fonction utilitaire pour réinitialiser toutes les séquences
CREATE OR REPLACE FUNCTION reset_sequence(table_name TEXT)  
RETURNS VOID AS $$  
DECLARE  
    seq_name TEXT;
    max_id BIGINT;
BEGIN
    seq_name := table_name || '_id_seq';
    EXECUTE format('SELECT COALESCE(MAX(id), 0) FROM %I', table_name) INTO max_id;
    EXECUTE format('SELECT setval(%L, %s)', seq_name, max_id);
END;
$$ LANGUAGE plpgsql;

-- Utilisation
SELECT reset_sequence('produits');
```

---

## IDENTITY : Le Standard SQL (PostgreSQL 10+)

### Qu'est-ce que IDENTITY ?

`IDENTITY` est le **standard SQL** pour la génération automatique de colonnes. PostgreSQL 10+ le supporte comme alternative à `SERIAL`.

### Différences : SERIAL vs IDENTITY

| Aspect | SERIAL | IDENTITY |
|--------|--------|----------|
| Standard SQL | ❌ Non (spécifique PostgreSQL) | ✅ Oui (SQL:2003) |
| Type de données | Raccourci (macro) | Propriété de colonne |
| Insertion manuelle | Toujours possible | Contrôlable (ALWAYS/BY DEFAULT) |
| Portabilité | Faible | Meilleure |
| Recommandation | Legacy (mais très utilisé) | **Préféré pour nouveau code** |

### IDENTITY ALWAYS : Génération Stricte

```sql
-- GENERATED ALWAYS AS IDENTITY : insertion manuelle interdite
CREATE TABLE employes (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255)
);

-- Insertion normale (OK)
INSERT INTO employes (nom, email) VALUES ('Alice', 'alice@example.com');  -- id = 1

-- Tentative d'insertion manuelle (ERREUR)
INSERT INTO employes (id, nom, email) VALUES (100, 'Bob', 'bob@example.com');
-- ERROR: cannot insert into column "id"

-- Pour forcer quand même (déconseillé)
INSERT INTO employes (id, nom, email)  
OVERRIDING SYSTEM VALUE  
VALUES (100, 'Bob', 'bob@example.com');  
```

### IDENTITY BY DEFAULT : Génération Flexible

```sql
-- GENERATED BY DEFAULT AS IDENTITY : insertion manuelle autorisée
CREATE TABLE produits_identity (
    id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    nom VARCHAR(200),
    prix NUMERIC(10, 2)
);

-- Insertion automatique
INSERT INTO produits_identity (nom, prix) VALUES ('Produit A', 10.00);  -- id = 1

-- Insertion manuelle (OK avec BY DEFAULT)
INSERT INTO produits_identity (id, nom, prix) VALUES (100, 'Produit B', 20.00);

-- Prochaine insertion automatique
INSERT INTO produits_identity (nom, prix) VALUES ('Produit C', 30.00);  -- id = 2 (pas 101 !)

SELECT * FROM produits_identity ORDER BY id;
```

Résultat :
```
 id  |    nom     | prix
-----+------------+-------
   1 | Produit A  | 10.00
   2 | Produit C  | 30.00
 100 | Produit B  | 20.00
```

**⚠️ Important :** L'insertion manuelle n'affecte pas la séquence automatique !

### Options de IDENTITY

```sql
-- Avec options personnalisées
CREATE TABLE commandes_identity (
    id INTEGER GENERATED ALWAYS AS IDENTITY (
        START WITH 1000
        INCREMENT BY 1
        MINVALUE 1000
        MAXVALUE 999999
        CACHE 20
    ) PRIMARY KEY,
    numero VARCHAR(50)
);

-- Vérifier la séquence créée
SELECT * FROM pg_sequences WHERE sequencename LIKE 'commandes_identity%';
```

### Modifier une Colonne IDENTITY

```sql
-- Redémarrer la séquence
ALTER TABLE employes ALTER COLUMN id RESTART WITH 100;

-- Changer les options
ALTER TABLE employes ALTER COLUMN id  
SET INCREMENT BY 10;  

-- Changer de ALWAYS à BY DEFAULT
ALTER TABLE employes ALTER COLUMN id  
SET GENERATED BY DEFAULT;  

-- Supprimer IDENTITY
ALTER TABLE employes ALTER COLUMN id  
DROP IDENTITY;  
```

---

## Comportements Importants des Séquences

### 1. Les Séquences ne Sont Pas Transactionnelles

```sql
BEGIN;
    INSERT INTO produits (nom, prix) VALUES ('Test 1', 10.00);  -- Consomme id = 4
    INSERT INTO produits (nom, prix) VALUES ('Test 2', 20.00);  -- Consomme id = 5
ROLLBACK;  -- Annulation de la transaction

-- Les IDs 4 et 5 sont perdus (trou dans la séquence)
INSERT INTO produits (nom, prix) VALUES ('Test 3', 30.00);  -- id = 6 (pas 4 !)

SELECT * FROM produits;
-- Résultat : IDs 1, 2, 3, 6 (pas de 4 ni 5)
```

**C'est normal !** Les séquences ne reculent jamais, même en cas de ROLLBACK.

### 2. Trous dans la Numérotation

Les trous sont **normaux et acceptables** :

```sql
-- Causes de trous :
-- 1. ROLLBACK de transactions
-- 2. DELETE de lignes
-- 3. Erreurs d'insertion
-- 4. Cache de séquences

CREATE TABLE test_trous (
    id SERIAL PRIMARY KEY,
    valeur VARCHAR(50)
);

INSERT INTO test_trous (valeur) VALUES ('A');  -- id = 1  
INSERT INTO test_trous (valeur) VALUES ('B');  -- id = 2  
DELETE FROM test_trous WHERE id = 2;  
INSERT INTO test_trous (valeur) VALUES ('C');  -- id = 3 (pas 2 !)  

SELECT * FROM test_trous;
-- Résultat : 1, 3 (trou à 2)
```

**Ne cherchez pas à combler les trous !** Ce n'est pas nécessaire ni recommandé.

### 3. Insertion Manuelle d'ID

```sql
CREATE TABLE test_manuel (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

INSERT INTO test_manuel (nom) VALUES ('Auto 1');  -- id = 1  
INSERT INTO test_manuel (nom) VALUES ('Auto 2');  -- id = 2  

-- Insertion manuelle
INSERT INTO test_manuel (id, nom) VALUES (100, 'Manuel 100');

-- La séquence n'est PAS mise à jour automatiquement
INSERT INTO test_manuel (nom) VALUES ('Auto 3');  -- id = 3 (pas 101 !)

-- Problème potentiel : conflit futur
-- Quand la séquence atteindra 100...
```

**Solution :** Synchroniser la séquence après insertion manuelle :

```sql
SELECT setval('test_manuel_id_seq', (SELECT MAX(id) FROM test_manuel));
```

### 4. Concurrence et Performance

Les séquences sont **thread-safe** et conçues pour la concurrence :

```sql
-- Plusieurs sessions peuvent appeler nextval() simultanément
-- Chaque session obtient un ID unique, sans conflit

-- Le CACHE améliore les performances
CREATE SEQUENCE seq_rapide
    CACHE 100;  -- Pré-alloue 100 valeurs en mémoire

-- Utile pour insertion massive
```

**Trade-off :** Un cache élevé améliore les performances mais peut créer plus de trous en cas de crash serveur.

---

## Cas d'Usage Avancés

### 1. Numéros de Commande Formatés

```sql
-- Séquence pour numéros de commande
CREATE SEQUENCE seq_cmd START WITH 1;

CREATE TABLE commandes_format (
    id SERIAL PRIMARY KEY,
    numero_commande VARCHAR(50) DEFAULT
        'CMD-' || TO_CHAR(CURRENT_DATE, 'YYYY') || '-' ||
        LPAD(nextval('seq_cmd')::TEXT, 6, '0'),
    client_id INTEGER,
    montant NUMERIC(10, 2)
);

-- Insertion
INSERT INTO commandes_format (client_id, montant) VALUES (1, 100.00);  
INSERT INTO commandes_format (client_id, montant) VALUES (2, 200.00);  

SELECT id, numero_commande, montant FROM commandes_format;
-- Résultats :
-- 1 | CMD-2025-000001 | 100.00
-- 2 | CMD-2025-000002 | 200.00
```

### 2. Séquence Partagée Entre Tables

```sql
-- Une séquence pour plusieurs tables
CREATE SEQUENCE seq_transactions START WITH 1;

CREATE TABLE ventes (
    id INTEGER DEFAULT nextval('seq_transactions') PRIMARY KEY,
    montant NUMERIC(10, 2)
);

CREATE TABLE remboursements (
    id INTEGER DEFAULT nextval('seq_transactions') PRIMARY KEY,
    montant NUMERIC(10, 2)
);

-- Les IDs sont uniques à travers les deux tables
INSERT INTO ventes (montant) VALUES (100);        -- id = 1  
INSERT INTO remboursements (montant) VALUES (50); -- id = 2  
INSERT INTO ventes (montant) VALUES (200);        -- id = 3  
```

**Cas d'usage :** Numérotation unique pour transactions multi-types.

### 3. Séquence avec Reset Annuel

```sql
-- Séquence qui recommence chaque année
CREATE TABLE factures_annuelles (
    id SERIAL PRIMARY KEY,
    numero INTEGER,
    annee INTEGER,
    montant NUMERIC(10, 2),
    created_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(annee, numero)
);

-- Fonction pour obtenir le prochain numéro
CREATE OR REPLACE FUNCTION prochain_numero_facture()  
RETURNS INTEGER AS $$  
DECLARE  
    annee_courante INTEGER := EXTRACT(YEAR FROM CURRENT_DATE);
    prochain_num INTEGER;
BEGIN
    SELECT COALESCE(MAX(numero), 0) + 1
    INTO prochain_num
    FROM factures_annuelles
    WHERE annee = annee_courante;

    RETURN prochain_num;
END;
$$ LANGUAGE plpgsql;

-- Utilisation
INSERT INTO factures_annuelles (numero, annee, montant)  
VALUES (prochain_numero_facture(), EXTRACT(YEAR FROM CURRENT_DATE), 150.00);  
```

### 4. Identifiants Composites

```sql
-- ID composite avec préfixe
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) GENERATED ALWAYS AS (
        'DOC-' || LPAD(id::TEXT, 8, '0')
    ) STORED,
    titre VARCHAR(300)
);

INSERT INTO documents (titre) VALUES ('Document 1');  
INSERT INTO documents (titre) VALUES ('Document 2');  

SELECT id, code, titre FROM documents;
-- 1 | DOC-00000001 | Document 1
-- 2 | DOC-00000002 | Document 2
```

---

## Problèmes Courants et Solutions

### Problème 1 : Séquence Désynchronisée

```sql
-- Symptôme : Erreur de contrainte unique après import
INSERT INTO produits (id, nom, prix) VALUES (50, 'Import', 10.00);  
INSERT INTO produits (nom, prix) VALUES ('Nouveau', 20.00);  
-- ERROR: duplicate key value violates unique constraint
-- La séquence génère un ID déjà utilisé

-- Solution : Synchroniser la séquence
SELECT setval('produits_id_seq', (SELECT MAX(id) FROM produits));
```

### Problème 2 : Atteindre la Limite

```sql
-- SERIAL (INTEGER) : limite à ~2 milliards
-- Si vous atteignez cette limite...

-- Solution 1 : Migrer vers BIGSERIAL
ALTER TABLE logs ALTER COLUMN id TYPE BIGINT;
-- Puis recréer la séquence en BIGINT

-- Solution 2 : Ajouter CYCLE (rarement approprié)
ALTER SEQUENCE logs_id_seq CYCLE;
```

### Problème 3 : Performance avec Insertions Massives

```sql
-- Problème : nextval() appelé trop souvent

-- Solution : Augmenter le CACHE
ALTER SEQUENCE produits_id_seq CACHE 1000;

-- Ou : Pré-allouer des IDs
CREATE TEMP TABLE temp_ids AS  
SELECT nextval('produits_id_seq') AS id  
FROM generate_series(1, 100000);  

-- Utiliser ces IDs pour l'insertion
```

---

## Supprimer une Séquence

```sql
-- Supprimer une séquence
DROP SEQUENCE seq_numero_facture;

-- Supprimer avec CASCADE (supprime les dépendances)
DROP SEQUENCE seq_numero_facture CASCADE;

-- Supprimer si elle existe
DROP SEQUENCE IF EXISTS seq_numero_facture;

-- Note : Supprimer une table supprime automatiquement sa séquence SERIAL
DROP TABLE produits;  -- Supprime aussi produits_id_seq
```

---

## Bonnes Pratiques

### 1. Préférer SERIAL pour Simplicité

```sql
-- ✅ Simple et efficace
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(300)
);
```

### 2. Utiliser BIGSERIAL pour Logs et Données Massives

```sql
-- ✅ Pour tables qui grossissent rapidement
CREATE TABLE logs_application (
    id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ,
    message TEXT
);
```

### 3. Utiliser IDENTITY pour Code Standard SQL

```sql
-- ✅ Pour portabilité vers autres SGBD
CREATE TABLE clients (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100)
);
```

### 4. Ne Pas Réutiliser les IDs

```sql
-- ❌ MAUVAIS : Essayer de combler les trous
-- Ne perdez pas de temps à ça !

-- ✅ BON : Accepter les trous
-- Les IDs ne sont pas censés être consécutifs
```

### 5. Synchroniser Après Import

```sql
-- ✅ Toujours synchroniser après import de données
COPY produits FROM 'data.csv';  
SELECT setval('produits_id_seq', (SELECT MAX(id) FROM produits));  
```

### 6. Documenter les Séquences Personnalisées

```sql
-- ✅ Commenter les séquences spéciales
CREATE SEQUENCE seq_factures START WITH 2025001;  
COMMENT ON SEQUENCE seq_factures IS 'Numérotation des factures - Format: FAC-YYYYNNNNN';  
```

### 7. Utiliser Cache pour Performance

```sql
-- ✅ Cache élevé pour tables à forte insertion
CREATE SEQUENCE seq_logs
    CACHE 1000;

-- ⚠️ Mais attention : cache trop élevé = plus de trous en cas de crash
```

---

## Inspection et Monitoring

### Voir Toutes les Séquences

```sql
-- Méthode 1 : psql
\ds

-- Méthode 2 : Vue pg_sequences
SELECT
    schemaname,
    sequencename,
    last_value,
    max_value,
    increment_by,
    cache_size
FROM pg_sequences  
WHERE schemaname = 'public'  
ORDER BY sequencename;  
```

### Voir la Valeur Actuelle

```sql
-- Valeur actuelle (non consommée)
SELECT last_value FROM nom_sequence;

-- Ou
SELECT * FROM nom_sequence;
```

### Trouver la Séquence d'une Colonne

```sql
-- Trouver quelle séquence alimente une colonne
SELECT pg_get_serial_sequence('nom_table', 'nom_colonne');

-- Exemple
SELECT pg_get_serial_sequence('produits', 'id');
-- Résultat : public.produits_id_seq
```

### Vérifier la Progression

```sql
-- Voir combien d'IDs restent disponibles
SELECT
    sequencename,
    last_value,
    max_value,
    max_value - last_value AS ids_restants,
    ROUND(100.0 * last_value / max_value, 2) AS pourcentage_utilise
FROM pg_sequences  
WHERE sequencename LIKE '%_id_seq';  
```

---

## Séquences et Réplication

### Considérations pour la Réplication

```sql
-- En réplication, les séquences peuvent causer des problèmes

-- Solution 1 : Utiliser des plages d'IDs par serveur
-- Serveur 1 : 1, 11, 21, 31... (INCREMENT BY 10, START 1)
-- Serveur 2 : 2, 12, 22, 32... (INCREMENT BY 10, START 2)

CREATE SEQUENCE seq_serveur1
    START WITH 1
    INCREMENT BY 10;

CREATE SEQUENCE seq_serveur2
    START WITH 2
    INCREMENT BY 10;

-- Solution 2 : Utiliser UUID au lieu de SERIAL
CREATE TABLE items_distribues (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nom VARCHAR(200)
);
```

---

## Comparaison Finale : SERIAL vs IDENTITY vs UUID

| Aspect | SERIAL | IDENTITY | UUID |
|--------|--------|----------|------|
| Standard SQL | ❌ Non | ✅ Oui | ✅ Oui |
| Taille | 4/8 octets | 4/8 octets | 16 octets |
| Séquentialité | ✅ Oui | ✅ Oui | ❌ Non (v4) / ✅ Oui (v7) |
| Distribution | ❌ Difficile | ❌ Difficile | ✅ Facile |
| Performance | ✅✅ Excellente | ✅✅ Excellente | ✅ Bonne |
| Lisibilité | ✅✅ Très bonne | ✅✅ Très bonne | ❌ Moyenne |
| **Usage recommandé** | Standard PostgreSQL | Standard SQL | Systèmes distribués |

---

## Récapitulatif

### Points Clés

1. **SEQUENCE** : Générateur de nombres automatique  
2. **SERIAL** : Raccourci PostgreSQL (crée séquence + colonne)  
3. **IDENTITY** : Standard SQL (préféré pour nouveau code)  
4. **Trous normaux** : Les séquences ne reculent jamais  
5. **Non transactionnel** : ROLLBACK ne restitue pas les IDs  
6. **Thread-safe** : Utilisable en concurrence

### Commandes Essentielles

```sql
-- Créer avec SERIAL
CREATE TABLE t (id SERIAL PRIMARY KEY);

-- Créer avec IDENTITY
CREATE TABLE t (id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY);

-- Créer séquence manuelle
CREATE SEQUENCE seq START WITH 1000;

-- Utiliser séquence
SELECT nextval('seq');  
SELECT currval('seq');  
SELECT setval('seq', 5000);  

-- Synchroniser
SELECT setval('seq', (SELECT MAX(id) FROM table));

-- Réinitialiser
ALTER SEQUENCE seq RESTART WITH 1;
```

### Quand Utiliser Quoi ?

| Besoin | Solution |
|--------|----------|
| ID simple, standard | `SERIAL` |
| Code portable SQL | `IDENTITY` |
| Très grande table | `BIGSERIAL` |
| Système distribué | `UUID v7` |
| Numérotation spéciale | `SEQUENCE` manuelle |

---

## Conclusion

Les séquences sont un mécanisme fondamental de PostgreSQL pour générer des identifiants uniques automatiquement. Comprendre leur fonctionnement vous permet de :

- ✅ Créer des clés primaires efficaces  
- ✅ Éviter les erreurs courantes  
- ✅ Optimiser les performances  
- ✅ Gérer correctement les cas complexes

**Recommandation moderne :** Pour les nouveaux projets, utilisez `IDENTITY` pour la conformité au standard SQL, ou `UUID v7` (PostgreSQL 18+) pour les systèmes distribués.

Dans la prochaine section, nous explorerons les domaines et types personnalisés, qui permettent de créer vos propres types de données réutilisables.

---


⏭️ [Domaines et types personnalisés (CREATE TYPE)](/04-objets-de-la-base-de-donnees/06-domaines-et-types-personnalises.md)
