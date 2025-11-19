🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2. Le concept de Schéma (Namespaces) et le search_path

## Introduction

Dans la section précédente, nous avons découvert que les **schémas** sont des espaces de noms qui permettent d'organiser les objets dans une database PostgreSQL. Dans cette section, nous allons approfondir ce concept crucial et comprendre comment PostgreSQL résout les noms d'objets grâce au mécanisme du **search_path**.

---

## Qu'est-ce qu'un Namespace (Espace de Noms) ?

### Définition simple

Un **namespace** (espace de noms) est un conteneur logique qui permet d'éviter les conflits de noms entre différents objets. C'est un concept que l'on retrouve dans de nombreux langages de programmation et systèmes.

### Analogie du monde réel

Imaginez que vous travaillez dans une grande entreprise avec plusieurs départements :

- **Département Marketing** : Il y a un employé nommé "Jean Dupont"
- **Département IT** : Il y a aussi un employé nommé "Jean Dupont"

Comment faire la différence entre les deux ? On utilise le département (namespace) :
- "Jean Dupont du Marketing"
- "Jean Dupont de l'IT"

Dans PostgreSQL, c'est exactement pareil :
- `marketing.clients` (table clients dans le schéma marketing)
- `it.clients` (table clients dans le schéma IT)

### Pourquoi les namespaces sont importants ?

Les namespaces permettent de :

1. **Éviter les conflits de noms** : Deux tables peuvent avoir le même nom si elles sont dans des schémas différents
2. **Organiser logiquement** : Regrouper des objets liés par fonctionnalité ou domaine métier
3. **Gérer les permissions** : Attribuer des droits d'accès par schéma
4. **Isoler le code** : Séparer le code de différentes applications ou modules

---

## Les Schémas PostgreSQL en Détail

### Le schéma `public`

Quand vous créez une nouvelle database dans PostgreSQL, un schéma appelé **`public`** est automatiquement créé. C'est le schéma par défaut où les objets sont créés si vous ne spécifiez rien d'autre.

```sql
-- Ces deux commandes sont équivalentes
CREATE TABLE users (id INT);
CREATE TABLE public.users (id INT);
```

### Créer un schéma

La création d'un schéma est très simple :

```sql
-- Syntaxe de base
CREATE SCHEMA nom_du_schema;

-- Exemple
CREATE SCHEMA ventes;
CREATE SCHEMA ressources_humaines;
CREATE SCHEMA reporting;
```

### Créer un schéma avec un propriétaire

Vous pouvez spécifier quel utilisateur sera le propriétaire du schéma :

```sql
-- Créer un schéma appartenant à un utilisateur spécifique
CREATE SCHEMA ventes AUTHORIZATION vendeur_user;

-- Créer un schéma avec un nom différent du propriétaire
CREATE SCHEMA AUTHORIZATION jean_dupont;
-- Cela crée un schéma nommé "jean_dupont"
```

### Supprimer un schéma

```sql
-- Supprimer un schéma vide
DROP SCHEMA nom_du_schema;

-- Supprimer un schéma et tous ses objets
DROP SCHEMA nom_du_schema CASCADE;

-- Exemples
DROP SCHEMA ventes; -- Échoue si le schéma contient des objets
DROP SCHEMA ventes CASCADE; -- Supprime le schéma et tout son contenu
```

⚠️ **Attention** : `CASCADE` supprime TOUS les objets du schéma. Utilisez avec précaution !

### Lister les schémas

```sql
-- Dans psql
\dn

-- Avec une requête SQL
SELECT schema_name
FROM information_schema.schemata;

-- Voir les schémas avec leur propriétaire
SELECT
    nspname AS schema_name,
    pg_catalog.pg_get_userbyid(nspowner) AS owner
FROM pg_catalog.pg_namespace
ORDER BY schema_name;
```

---

## Utiliser les Schémas

### Syntaxe qualifiée complète

Pour référencer un objet dans un schéma spécifique, vous utilisez la notation **`schema.objet`** :

```sql
-- Créer une table dans un schéma spécifique
CREATE TABLE ventes.commandes (
    id SERIAL PRIMARY KEY,
    client_id INT,
    montant DECIMAL(10, 2),
    date_commande DATE
);

-- Insérer des données
INSERT INTO ventes.commandes (client_id, montant, date_commande)
VALUES (1, 150.50, '2025-11-19');

-- Sélectionner des données
SELECT * FROM ventes.commandes;

-- Modifier la structure
ALTER TABLE ventes.commandes ADD COLUMN statut VARCHAR(20);

-- Supprimer la table
DROP TABLE ventes.commandes;
```

### Créer plusieurs objets dans le même schéma

```sql
-- Créer le schéma
CREATE SCHEMA gestion_stock;

-- Créer plusieurs tables
CREATE TABLE gestion_stock.produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    reference VARCHAR(50)
);

CREATE TABLE gestion_stock.mouvements (
    id SERIAL PRIMARY KEY,
    produit_id INT REFERENCES gestion_stock.produits(id),
    quantite INT,
    type VARCHAR(20),
    date_mouvement TIMESTAMP
);

CREATE TABLE gestion_stock.inventaires (
    id SERIAL PRIMARY KEY,
    produit_id INT REFERENCES gestion_stock.produits(id),
    quantite_theorique INT,
    quantite_reelle INT,
    date_inventaire DATE
);
```

---

## Le search_path : Comment PostgreSQL Trouve les Objets

### Qu'est-ce que le search_path ?

Le **search_path** est un paramètre de configuration qui indique à PostgreSQL dans quel(s) schéma(s) chercher quand vous **ne spécifiez pas** le nom du schéma dans une requête.

C'est similaire à la variable `PATH` dans les systèmes Unix/Linux, qui indique où chercher les exécutables.

### Analogie

Imaginez que vous cherchez un livre dans une bibliothèque :

- **Sans search_path** : Vous devez dire exactement "Je veux le livre X dans la salle Y, dans l'étagère Z"
- **Avec search_path** : Vous dites "Je veux le livre X" et le bibliothécaire cherche automatiquement dans les salles que vous avez configurées (d'abord la salle A, puis la salle B, etc.)

### Voir le search_path actuel

```sql
-- Méthode 1 : Avec SHOW
SHOW search_path;

-- Résultat typique
-- "$user", public
```

Ce résultat signifie :
1. PostgreSQL cherche d'abord dans un schéma qui porte le nom de l'utilisateur connecté (si ce schéma existe)
2. Puis dans le schéma `public`

### Comportement par défaut

```sql
-- Si votre search_path est : "$user", public
-- Et que vous êtes connecté en tant que "alice"

-- Cette requête
SELECT * FROM produits;

-- PostgreSQL cherche dans cet ordre :
-- 1. alice.produits (si le schéma "alice" existe)
-- 2. public.produits
-- 3. Si aucune table n'est trouvée → ERREUR
```

### Modifier le search_path

#### Pour la session en cours

```sql
-- Définir un nouveau search_path pour la session actuelle
SET search_path TO ventes, public;

-- Maintenant PostgreSQL cherchera d'abord dans "ventes", puis dans "public"
```

#### Pour un utilisateur spécifique

```sql
-- Modifier le search_path par défaut pour un utilisateur
ALTER USER mon_utilisateur SET search_path TO ventes, ressources_humaines, public;
```

#### Pour une database entière

```sql
-- Modifier le search_path par défaut pour une database
ALTER DATABASE ma_base SET search_path TO ventes, public;
```

#### Restaurer le search_path par défaut

```sql
-- Revenir au search_path par défaut
SET search_path TO "$user", public;

-- Ou simplement se reconnecter
```

---

## Exemples Pratiques du search_path

### Exemple 1 : Schémas avec des tables de même nom

```sql
-- Créer deux schémas
CREATE SCHEMA france;
CREATE SCHEMA belgique;

-- Créer une table "clients" dans chaque schéma
CREATE TABLE france.clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    pays VARCHAR(50) DEFAULT 'France'
);

CREATE TABLE belgique.clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    pays VARCHAR(50) DEFAULT 'Belgique'
);

-- Insérer des données
INSERT INTO france.clients (nom) VALUES ('Jean Dupont');
INSERT INTO belgique.clients (nom) VALUES ('Marc Dubois');

-- Voir le search_path actuel
SHOW search_path;
-- Résultat : "$user", public

-- Requête sans qualification → ERREUR (table pas dans le search_path)
SELECT * FROM clients;
-- ERROR: relation "clients" does not exist

-- Modifier le search_path pour inclure "france"
SET search_path TO france, public;

-- Maintenant cette requête fonctionne et retourne les clients français
SELECT * FROM clients;
-- Résultat :
-- id | nom         | pays
-- ---+-------------+--------
-- 1  | Jean Dupont | France

-- Changer le search_path pour "belgique"
SET search_path TO belgique, public;

-- La même requête retourne maintenant les clients belges
SELECT * FROM clients;
-- Résultat :
-- id | nom         | pays
-- ---+-------------+----------
-- 1  | Marc Dubois | Belgique

-- On peut toujours accéder aux deux avec des noms qualifiés
SELECT * FROM france.clients;
SELECT * FROM belgique.clients;
```

### Exemple 2 : Ordre de priorité dans le search_path

```sql
-- Créer des schémas
CREATE SCHEMA prioritaire;
CREATE SCHEMA secondaire;

-- Créer une table "produits" dans les deux
CREATE TABLE prioritaire.produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    type VARCHAR(50) DEFAULT 'PREMIUM'
);

CREATE TABLE secondaire.produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    type VARCHAR(50) DEFAULT 'STANDARD'
);

-- Insérer des données
INSERT INTO prioritaire.produits (nom) VALUES ('Produit A');
INSERT INTO secondaire.produits (nom) VALUES ('Produit B');

-- Définir le search_path avec "prioritaire" en premier
SET search_path TO prioritaire, secondaire, public;

-- Cette requête trouve la table dans "prioritaire" (premier schéma)
SELECT * FROM produits;
-- Résultat :
-- id | nom       | type
-- ---+-----------+---------
-- 1  | Produit A | PREMIUM

-- Inverser l'ordre
SET search_path TO secondaire, prioritaire, public;

-- Maintenant la même requête trouve la table dans "secondaire"
SELECT * FROM produits;
-- Résultat :
-- id | nom       | type
-- ---+-----------+----------
-- 1  | Produit B | STANDARD
```

### Exemple 3 : Application pratique avec modules

```sql
-- Créer des schémas pour différents modules d'une application
CREATE SCHEMA auth;      -- Module d'authentification
CREATE SCHEMA blog;      -- Module de blog
CREATE SCHEMA ecommerce; -- Module e-commerce

-- Tables du module auth
CREATE TABLE auth.users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    password_hash VARCHAR(255),
    email VARCHAR(255)
);

CREATE TABLE auth.sessions (
    id UUID PRIMARY KEY,
    user_id INT REFERENCES auth.users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tables du module blog
CREATE TABLE blog.posts (
    id SERIAL PRIMARY KEY,
    author_id INT, -- Référence vers auth.users
    title VARCHAR(200),
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE blog.comments (
    id SERIAL PRIMARY KEY,
    post_id INT REFERENCES blog.posts(id),
    author_id INT, -- Référence vers auth.users
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tables du module ecommerce
CREATE TABLE ecommerce.products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10, 2)
);

CREATE TABLE ecommerce.orders (
    id SERIAL PRIMARY KEY,
    user_id INT, -- Référence vers auth.users
    total DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Définir le search_path selon le contexte
-- Pour travailler sur le blog :
SET search_path TO blog, auth, public;

-- Pour travailler sur l'e-commerce :
SET search_path TO ecommerce, auth, public;

-- "auth" est toujours dans le search_path car les utilisateurs
-- sont partagés entre tous les modules
```

---

## Le Schéma Spécial `$user`

### Qu'est-ce que `$user` ?

Dans le search_path, **`$user`** est un placeholder qui est remplacé par le nom de l'utilisateur actuellement connecté.

```sql
-- Si le search_path est : "$user", public
-- Et que vous êtes connecté en tant que "alice"
-- PostgreSQL cherchera dans : alice, public

-- Si vous êtes connecté en tant que "bob"
-- PostgreSQL cherchera dans : bob, public
```

### Utilité de `$user`

Ce mécanisme permet de créer des environnements isolés pour chaque utilisateur :

```sql
-- Créer un schéma pour chaque utilisateur
CREATE SCHEMA alice;
CREATE SCHEMA bob;

-- Créer une table "privée" pour chaque utilisateur
CREATE TABLE alice.notes (
    id SERIAL PRIMARY KEY,
    contenu TEXT
);

CREATE TABLE bob.notes (
    id SERIAL PRIMARY KEY,
    contenu TEXT
);

-- Avec le search_path par défaut ("$user", public)

-- Quand Alice se connecte et exécute :
SELECT * FROM notes;
-- PostgreSQL cherche dans alice.notes

-- Quand Bob se connecte et exécute :
SELECT * FROM notes;
-- PostgreSQL cherche dans bob.notes
```

---

## Les Schémas Système

PostgreSQL possède plusieurs schémas système qui contiennent des métadonnées et des fonctions internes :

### 1. `pg_catalog`

Le schéma **`pg_catalog`** contient toutes les tables système et les fonctions internes de PostgreSQL.

```sql
-- Ce schéma est TOUJOURS dans le search_path (implicitement en premier)
-- C'est pourquoi vous pouvez utiliser des fonctions sans préfixe

SELECT current_timestamp; -- Équivalent à pg_catalog.current_timestamp
SELECT version();         -- Équivalent à pg_catalog.version()

-- Exemples de tables système dans pg_catalog
SELECT * FROM pg_catalog.pg_tables;      -- Liste de toutes les tables
SELECT * FROM pg_catalog.pg_namespace;   -- Liste de tous les schémas
SELECT * FROM pg_catalog.pg_class;       -- Liste de tous les objets
```

### 2. `information_schema`

Le schéma **`information_schema`** fournit une vue standardisée (ANSI SQL) des métadonnées.

```sql
-- Liste de toutes les tables (vue standard SQL)
SELECT * FROM information_schema.tables;

-- Liste de toutes les colonnes
SELECT * FROM information_schema.columns;

-- Liste de toutes les contraintes
SELECT * FROM information_schema.table_constraints;
```

### 3. `pg_toast`

Le schéma **`pg_toast`** contient les données TOAST (The Oversized-Attribute Storage Technique) pour stocker les valeurs trop grandes.

Vous n'avez généralement pas besoin d'interagir directement avec ce schéma.

---

## Bonnes Pratiques avec les Schémas

### 1. Ne pas tout mettre dans `public`

❌ **Mauvaise pratique** :
```sql
-- Tout dans public
CREATE TABLE public.users (...);
CREATE TABLE public.products (...);
CREATE TABLE public.orders (...);
CREATE TABLE public.invoices (...);
CREATE TABLE public.blog_posts (...);
-- ... 50 autres tables
```

✅ **Bonne pratique** :
```sql
-- Organisation par domaine
CREATE SCHEMA auth;
CREATE SCHEMA ecommerce;
CREATE SCHEMA blog;
CREATE SCHEMA billing;

CREATE TABLE auth.users (...);
CREATE TABLE ecommerce.products (...);
CREATE TABLE ecommerce.orders (...);
CREATE TABLE billing.invoices (...);
CREATE TABLE blog.posts (...);
```

### 2. Utiliser des noms de schémas descriptifs

❌ **Mauvaise pratique** :
```sql
CREATE SCHEMA s1;
CREATE SCHEMA s2;
CREATE SCHEMA app;
CREATE SCHEMA data;
```

✅ **Bonne pratique** :
```sql
CREATE SCHEMA user_management;
CREATE SCHEMA inventory;
CREATE SCHEMA reporting;
CREATE SCHEMA audit;
```

### 3. Toujours spécifier le schéma dans le code de production

Pour plus de clarté et éviter les ambiguïtés, spécifiez toujours le schéma dans votre code applicatif :

✅ **Recommandé** :
```sql
SELECT * FROM ecommerce.products;
INSERT INTO auth.users (username, email) VALUES (...);
UPDATE billing.invoices SET status = 'paid' WHERE id = 123;
```

❓ **À éviter en production** :
```sql
SELECT * FROM products;  -- Dans quel schéma ?
INSERT INTO users (...);  -- Quel schéma ?
```

### 4. Documenter votre organisation de schémas

Créez un document (ou un commentaire dans votre code) qui décrit l'organisation :

```sql
-- Documentation des schémas
COMMENT ON SCHEMA auth IS 'Module d''authentification et gestion des utilisateurs';
COMMENT ON SCHEMA ecommerce IS 'Module e-commerce : produits, paniers, commandes';
COMMENT ON SCHEMA billing IS 'Module de facturation et paiements';
COMMENT ON SCHEMA reporting IS 'Vues et tables pour le reporting';

-- Voir les commentaires
SELECT
    nspname AS schema_name,
    obj_description(oid, 'pg_namespace') AS description
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%'
  AND nspname != 'information_schema';
```

### 5. Configurer le search_path au niveau de l'application

Plutôt que de modifier le search_path manuellement, configurez-le au niveau de votre application :

```python
# Exemple en Python avec psycopg2
import psycopg2

conn = psycopg2.connect(
    "dbname=mydb user=myuser password=mypass"
)

# Définir le search_path pour cette connexion
cursor = conn.cursor()
cursor.execute("SET search_path TO ecommerce, auth, public")
```

```javascript
// Exemple en Node.js avec pg
const { Pool } = require('pg');

const pool = new Pool({
  user: 'myuser',
  host: 'localhost',
  database: 'mydb',
  password: 'mypass',
  port: 5432,
});

// Définir le search_path pour chaque connexion
pool.on('connect', (client) => {
  client.query('SET search_path TO ecommerce, auth, public');
});
```

---

## Cas d'Usage Courants

### Cas 1 : Séparation par environnement

```sql
-- Créer des schémas pour différents environnements
CREATE SCHEMA dev;
CREATE SCHEMA staging;
CREATE SCHEMA prod;

-- Structure identique dans chaque schéma
CREATE TABLE dev.users (...);
CREATE TABLE staging.users (...);
CREATE TABLE prod.users (...);

-- Basculer entre environnements via search_path
SET search_path TO dev, public;   -- Développement
SET search_path TO staging, public; -- Tests
SET search_path TO prod, public;  -- Production
```

### Cas 2 : Versioning de schéma

```sql
-- Plusieurs versions de votre schéma cohabitent
CREATE SCHEMA app_v1;
CREATE SCHEMA app_v2;
CREATE SCHEMA app_v3;

-- Chaque version a sa propre structure
CREATE TABLE app_v1.users (...); -- Structure ancienne
CREATE TABLE app_v2.users (...); -- Structure avec modifications
CREATE TABLE app_v3.users (...); -- Dernière structure

-- Les anciennes versions de l'application utilisent app_v1
-- Les nouvelles versions utilisent app_v3
```

### Cas 3 : Multi-tenant avec schémas

```sql
-- Créer un schéma par client
CREATE SCHEMA client_acme;
CREATE SCHEMA client_globex;
CREATE SCHEMA client_initech;

-- Structure identique pour chaque client
CREATE TABLE client_acme.data (...);
CREATE TABLE client_globex.data (...);
CREATE TABLE client_initech.data (...);

-- L'application sélectionne le bon schéma selon le client connecté
-- SET search_path TO client_acme, public;
```

### Cas 4 : Extensions et outils tiers

```sql
-- Installer des extensions dans un schéma dédié
CREATE SCHEMA extensions;

-- Installer PostGIS dans ce schéma
CREATE EXTENSION IF NOT EXISTS postgis SCHEMA extensions;

-- Installer pg_trgm dans ce schéma
CREATE EXTENSION IF NOT EXISTS pg_trgm SCHEMA extensions;

-- Ajouter "extensions" au search_path
SET search_path TO public, extensions;
```

---

## Gérer les Permissions par Schéma

Les schémas permettent une gestion fine des permissions :

```sql
-- Créer des rôles (utilisateurs)
CREATE ROLE vendeur LOGIN PASSWORD 'secret123';
CREATE ROLE comptable LOGIN PASSWORD 'secret456';
CREATE ROLE admin LOGIN PASSWORD 'secret789';

-- Créer des schémas
CREATE SCHEMA ventes;
CREATE SCHEMA comptabilite;

-- Permissions pour le vendeur
GRANT USAGE ON SCHEMA ventes TO vendeur;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA ventes TO vendeur;

-- Permissions pour le comptable
GRANT USAGE ON SCHEMA comptabilite TO comptable;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA comptabilite TO comptable;

-- Permissions pour l'admin (accès partout)
GRANT USAGE ON SCHEMA ventes TO admin;
GRANT USAGE ON SCHEMA comptabilite TO admin;
GRANT ALL ON ALL TABLES IN SCHEMA ventes TO admin;
GRANT ALL ON ALL TABLES IN SCHEMA comptabilite TO admin;

-- Le vendeur ne peut PAS accéder au schéma comptabilite
-- Le comptable ne peut PAS accéder au schéma ventes
```

---

## Pièges Courants et Comment les Éviter

### Piège 1 : Oublier le schéma dans les contraintes de clés étrangères

❌ **Problème** :
```sql
CREATE TABLE ventes.commandes (
    id SERIAL PRIMARY KEY,
    client_id INT REFERENCES clients(id)  -- Cherche dans le search_path !
);
```

✅ **Solution** :
```sql
CREATE TABLE ventes.commandes (
    id SERIAL PRIMARY KEY,
    client_id INT REFERENCES ventes.clients(id)  -- Schéma explicite
);
```

### Piège 2 : Confusion avec plusieurs schémas dans le search_path

```sql
-- Si vous avez le même objet dans plusieurs schémas du search_path
SET search_path TO schema1, schema2, public;

-- Cette requête peut donner des résultats inattendus
SELECT * FROM users;  -- Utilise schema1.users ou schema2.users ?
```

✅ **Solution** : Toujours qualifier les noms en cas de doute :
```sql
SELECT * FROM schema1.users;
```

### Piège 3 : search_path non défini pour les fonctions

Les fonctions SQL héritent du search_path au moment de leur exécution, ce qui peut causer des problèmes :

```sql
CREATE FUNCTION get_user_count() RETURNS INT AS $$
    SELECT COUNT(*) FROM users;  -- Quel schéma ?
$$ LANGUAGE SQL;
```

✅ **Solution** : Toujours qualifier ou définir le search_path dans la fonction :
```sql
CREATE FUNCTION get_user_count() RETURNS INT AS $$
    SELECT COUNT(*) FROM public.users;
$$ LANGUAGE SQL;

-- Ou utiliser SET pour la fonction
CREATE FUNCTION get_user_count() RETURNS INT AS $$
    SELECT COUNT(*) FROM users;
$$ LANGUAGE SQL SET search_path = public;
```

### Piège 4 : Privilèges manquants sur le schéma

```sql
-- Créer un schéma
CREATE SCHEMA my_schema;

-- Un autre utilisateur essaie d'y accéder
-- SET ROLE other_user;
SELECT * FROM my_schema.table1;  -- ERROR: permission denied for schema my_schema
```

✅ **Solution** : Accorder les privilèges nécessaires :
```sql
GRANT USAGE ON SCHEMA my_schema TO other_user;
GRANT SELECT ON ALL TABLES IN SCHEMA my_schema TO other_user;
```

---

## Visualisation Complète

Voici une représentation visuelle de l'interaction entre schémas et search_path :

```
Database: ecommerce
│
├── Schema: auth (schéma d'authentification)
│   ├── Table: users
│   └── Table: sessions
│
├── Schema: sales (schéma de ventes)
│   ├── Table: orders
│   ├── Table: products
│   └── Table: customers
│
├── Schema: reporting (schéma de reporting)
│   ├── View: sales_summary
│   └── View: customer_stats
│
└── Schema: public (schéma par défaut)
    └── Table: temp_data

search_path = "sales, auth, public"

Requête: SELECT * FROM orders;
         ↓
Recherche dans sales.orders ✓ TROUVÉ
         ↓
Retourne les données de sales.orders

Requête: SELECT * FROM users;
         ↓
Recherche dans sales.users ✗ Pas trouvé
         ↓
Recherche dans auth.users ✓ TROUVÉ
         ↓
Retourne les données de auth.users

Requête: SELECT * FROM temp_data;
         ↓
Recherche dans sales.temp_data ✗ Pas trouvé
         ↓
Recherche dans auth.temp_data ✗ Pas trouvé
         ↓
Recherche dans public.temp_data ✓ TROUVÉ
         ↓
Retourne les données de public.temp_data

Requête: SELECT * FROM sales_summary;
         ↓
Recherche dans sales.sales_summary ✗ Pas trouvé
         ↓
Recherche dans auth.sales_summary ✗ Pas trouvé
         ↓
Recherche dans public.sales_summary ✗ Pas trouvé
         ↓
ERROR: relation "sales_summary" does not exist
```

---

## Récapitulatif

### Points Clés à Retenir

1. **Les schémas sont des namespaces** : Ils permettent d'organiser et d'isoler les objets dans une database.

2. **Le search_path détermine où PostgreSQL cherche** : C'est une liste ordonnée de schémas dans laquelle PostgreSQL cherche les objets non qualifiés.

3. **`$user` est un placeholder** : Il est remplacé par le nom de l'utilisateur connecté.

4. **`public` est le schéma par défaut** : Créé automatiquement dans chaque database.

5. **`pg_catalog` est toujours prioritaire** : Il contient les fonctions et tables système et est toujours cherché en premier (implicitement).

6. **Spécifiez le schéma pour plus de clarté** : En production, utilisez toujours `schema.table` pour éviter les ambiguïtés.

### Commandes Essentielles

```sql
-- Créer un schéma
CREATE SCHEMA nom_schema;

-- Supprimer un schéma
DROP SCHEMA nom_schema CASCADE;

-- Voir le search_path
SHOW search_path;

-- Modifier le search_path (session)
SET search_path TO schema1, schema2, public;

-- Modifier le search_path (utilisateur)
ALTER USER mon_user SET search_path TO schema1, public;

-- Lister les schémas
\dn
SELECT schema_name FROM information_schema.schemata;

-- Voir les tables d'un schéma
\dt nom_schema.*
```

### Bonnes Pratiques Résumées

| ✅ À FAIRE | ❌ À ÉVITER |
|-----------|------------|
| Organiser par domaine métier | Tout mettre dans `public` |
| Noms de schémas descriptifs | Noms génériques (s1, s2, data) |
| Qualifier les objets en production | Dépendre uniquement du search_path |
| Documenter l'organisation | Laisser sans documentation |
| Gérer les permissions par schéma | Donner tous les droits à tous |
| Configurer le search_path dans l'app | Modifier manuellement à chaque fois |

---

## Conclusion

Les schémas et le search_path sont des concepts fondamentaux de PostgreSQL qui vous permettent de :

- **Organiser** vos objets de manière logique et maintenable
- **Éviter les conflits** de noms entre différents modules
- **Gérer les permissions** de manière granulaire
- **Simplifier** l'accès aux objets avec le search_path
- **Isoler** différentes parties de votre application

Maîtriser ces concepts est essentiel pour concevoir des applications PostgreSQL robustes et évolutives. Dans la prochaine section, nous verrons comment créer des tables et définir leur structure avec les différents types de données disponibles.

---


⏭️ [CREATE TABLE : Définir une structure](/04-objets-de-la-base-de-donnees/03-create-table.md)
