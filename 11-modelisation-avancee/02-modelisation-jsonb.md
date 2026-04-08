🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.2. Modélisation JSONB : Quand utiliser le NoSQL dans SQL

## Introduction

PostgreSQL est une base de données **relationnelle**, mais elle possède une capacité unique : intégrer des fonctionnalités **NoSQL** au sein même de son système relationnel. Cette hybridation se fait principalement grâce au type de données **JSONB**, qui permet de stocker et manipuler des documents JSON de manière performante.

### Qu'est-ce que JSON ?

**JSON** (JavaScript Object Notation) est un format de données textuelles léger et lisible par l'humain, utilisé pour représenter des structures de données complexes. Il est devenu le standard de facto pour l'échange de données sur le web.

**Exemple de document JSON :**

```json
{
  "nom": "Alice Dupont",
  "age": 32,
  "email": "alice.dupont@example.com",
  "adresse": {
    "rue": "15 rue de la Paix",
    "ville": "Paris",
    "code_postal": "75002"
  },
  "hobbies": ["lecture", "voyage", "photographie"],
  "actif": true
}
```

### JSON vs JSONB dans PostgreSQL

PostgreSQL propose **deux types** pour stocker du JSON :

| Caractéristique | JSON | JSONB |
|----------------|------|-------|
| **Stockage** | Texte brut | Format binaire décomposé |
| **Performance en écriture** | Plus rapide | Légèrement plus lent (conversion) |
| **Performance en lecture** | Plus lent | Beaucoup plus rapide |
| **Indexation** | Non supportée | Supportée (GIN) |
| **Ordre des clés** | Préservé | Non garanti |
| **Doublons de clés** | Conservés | Dernière valeur gardée |
| **Espaces blancs** | Conservés | Supprimés |
| **Recommandation** | Rarement utilisé | **À privilégier** |

**Règle d'or :** Utilisez **JSONB** dans 99% des cas. N'utilisez JSON que si vous devez absolument préserver l'ordre exact des clés et les espaces.

### Pourquoi JSONB est-il si performant ?

Lorsque vous insérez des données JSONB, PostgreSQL :
1. **Parse** le JSON  
2. **Décompose** la structure en format binaire optimisé  
3. **Stocke** de manière efficace pour un accès rapide

Lors de la lecture :
1. Pas besoin de parser à nouveau  
2. Accès direct aux éléments  
3. Opérations natives ultra-rapides

---

## Le Dilemme : Relationnel vs JSONB

### Le Monde Relationnel Classique

Dans une base de données relationnelle pure, chaque entité et relation est modélisée par des tables et des colonnes structurées.

**Exemple : Profil utilisateur**

```sql
-- Modèle relationnel strict
CREATE TABLE utilisateurs (
    utilisateur_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(255),
    date_naissance DATE
);

CREATE TABLE adresses (
    adresse_id SERIAL PRIMARY KEY,
    utilisateur_id INTEGER REFERENCES utilisateurs(utilisateur_id),
    type_adresse VARCHAR(20), -- 'principale', 'livraison', 'facturation'
    rue VARCHAR(255),
    ville VARCHAR(100),
    code_postal VARCHAR(10),
    pays VARCHAR(100)
);

CREATE TABLE telephones (
    telephone_id SERIAL PRIMARY KEY,
    utilisateur_id INTEGER REFERENCES utilisateurs(utilisateur_id),
    type_telephone VARCHAR(20), -- 'mobile', 'fixe', 'bureau'
    numero VARCHAR(20)
);

CREATE TABLE preferences_utilisateur (
    preference_id SERIAL PRIMARY KEY,
    utilisateur_id INTEGER REFERENCES utilisateurs(utilisateur_id),
    cle VARCHAR(100),
    valeur TEXT
);
```

**Avantages du modèle relationnel :**
- ✅ Structure rigide et validée  
- ✅ Intégrité référentielle garantie  
- ✅ Recherches et jointures optimisées  
- ✅ Normalisation et absence de redondance

**Inconvénients :**
- ❌ Rigidité : ajout d'un champ = modification de schéma (ALTER TABLE)  
- ❌ Complexité pour données variables  
- ❌ Nombreuses jointures pour reconstituer un objet complet  
- ❌ Inadapté pour données semi-structurées ou évolutives

### L'Approche JSONB

Avec JSONB, vous pouvez stocker des structures complexes et flexibles dans une seule colonne.

```sql
-- Modèle hybride avec JSONB
CREATE TABLE utilisateurs (
    utilisateur_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    profil JSONB
);

-- Exemple d'insertion
INSERT INTO utilisateurs (nom, email, profil) VALUES (
    'Alice Dupont',
    'alice.dupont@example.com',
    '{
        "prenom": "Alice",
        "age": 32,
        "date_naissance": "1992-05-15",
        "adresses": [
            {
                "type": "principale",
                "rue": "15 rue de la Paix",
                "ville": "Paris",
                "code_postal": "75002",
                "pays": "France"
            },
            {
                "type": "livraison",
                "rue": "8 avenue des Champs",
                "ville": "Lyon",
                "code_postal": "69001",
                "pays": "France"
            }
        ],
        "telephones": [
            {"type": "mobile", "numero": "+33601020304"},
            {"type": "fixe", "numero": "+33143567890"}
        ],
        "preferences": {
            "newsletter": true,
            "notifications_push": false,
            "theme": "sombre",
            "langue": "fr"
        },
        "reseaux_sociaux": {
            "twitter": "@alicedupont",
            "linkedin": "alice-dupont-pro"
        }
    }'
);
```

**Avantages de JSONB :**
- ✅ Flexibilité : structure évolutive sans migration  
- ✅ Simplicité : un seul objet, pas de jointures  
- ✅ Adapté aux données variables entre enregistrements  
- ✅ Idéal pour APIs REST (JSON natif)

**Inconvénients :**
- ❌ Moins de contrôle sur la structure (validation à faire côté application)  
- ❌ Peut devenir un "fourre-tout" si mal utilisé  
- ❌ Statistiques et index moins précis que sur colonnes dédiées

---

## Quand Utiliser JSONB : Critères de Décision

### ✅ Utilisez JSONB quand...

#### 1. **Données à schéma variable ou flexible**

Lorsque différents enregistrements ont des structures différentes.

**Exemple : Catalogue produits e-commerce**

Un site vend des livres, des vêtements et de l'électronique. Chaque catégorie a des attributs spécifiques :

```sql
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255),
    prix NUMERIC(10,2),
    categorie VARCHAR(50),
    specifications JSONB  -- Attributs variables selon la catégorie
);

-- Livre
INSERT INTO produits (nom, prix, categorie, specifications) VALUES (
    'Le Petit Prince',
    12.50,
    'livre',
    '{
        "auteur": "Antoine de Saint-Exupéry",
        "isbn": "978-2-07-061275-8",
        "nb_pages": 96,
        "editeur": "Gallimard",
        "langue": "français"
    }'
);

-- Vêtement
INSERT INTO produits (nom, prix, categorie, specifications) VALUES (
    'T-shirt coton bio',
    25.00,
    'vetement',
    '{
        "taille": "M",
        "couleur": "bleu",
        "matiere": "coton bio",
        "marque": "EcoWear",
        "entretien": "lavage 30°C"
    }'
);

-- Électronique
INSERT INTO produits (nom, prix, categorie, specifications) VALUES (
    'Smartphone Galaxy S25',
    799.00,
    'electronique',
    '{
        "marque": "Samsung",
        "modele": "Galaxy S25",
        "stockage": "256GB",
        "ram": "12GB",
        "couleur": "noir",
        "batterie": "5000mAh",
        "ecran": "6.8 pouces AMOLED"
    }'
);
```

**Alternative relationnelle :**
Créer une table par type de produit (produits_livres, produits_vetements, produits_electronique) ou utiliser EAV (Entity-Attribute-Value), tous deux complexes et peu pratiques.

#### 2. **Données provenant d'APIs externes**

Lorsque vous consommez des APIs qui retournent du JSON et que vous voulez stocker la réponse complète.

```sql
CREATE TABLE logs_api (
    log_id SERIAL PRIMARY KEY,
    api_endpoint VARCHAR(255),
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    status_code INTEGER,
    reponse JSONB,  -- Réponse complète de l'API
    duree_ms INTEGER
);

-- Stocker une réponse d'API météo
INSERT INTO logs_api (api_endpoint, status_code, reponse, duree_ms) VALUES (
    '/api/meteo/paris',
    200,
    '{
        "ville": "Paris",
        "temperature": 18.5,
        "humidite": 65,
        "conditions": "nuageux",
        "previsions": [
            {"jour": "lundi", "temp_max": 20, "temp_min": 12},
            {"jour": "mardi", "temp_max": 22, "temp_min": 14}
        ]
    }',
    145
);
```

#### 3. **Configuration et préférences utilisateur**

Les préférences utilisateur sont souvent nombreuses, évolutives, et chaque utilisateur peut avoir des préférences différentes.

```sql
CREATE TABLE utilisateurs (
    utilisateur_id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    preferences JSONB DEFAULT '{}'
);

INSERT INTO utilisateurs (email, preferences) VALUES (
    'alice@example.com',
    '{
        "interface": {
            "theme": "sombre",
            "langue": "fr",
            "taille_police": "medium"
        },
        "notifications": {
            "email": true,
            "push": false,
            "sms": false,
            "frequence": "quotidienne"
        },
        "confidentialite": {
            "profil_public": false,
            "partage_donnees": false
        }
    }'
);
```

#### 4. **Métadonnées et tags**

Informations supplémentaires non critiques mais utiles.

```sql
CREATE TABLE documents (
    document_id SERIAL PRIMARY KEY,
    titre VARCHAR(255),
    contenu TEXT,
    metadata JSONB  -- Informations additionnelles flexibles
);

INSERT INTO documents (titre, contenu, metadata) VALUES (
    'Rapport Q4 2025',
    'Contenu du rapport...',
    '{
        "auteur": "Bob Martin",
        "departement": "Finance",
        "tags": ["rapport", "finance", "q4", "2025"],
        "confidentialite": "interne",
        "version": "1.2",
        "reviseurs": ["Claire Dubois", "David Lefebvre"],
        "date_modification": "2025-11-15"
    }'
);
```

#### 5. **Historisation et audit trails**

Capturer l'état complet d'un objet à un moment donné.

```sql
CREATE TABLE audit_modifications (
    audit_id SERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    record_id INTEGER,
    action VARCHAR(20),  -- 'INSERT', 'UPDATE', 'DELETE'
    utilisateur VARCHAR(100),
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    anciennes_valeurs JSONB,  -- État avant modification
    nouvelles_valeurs JSONB   -- État après modification
);
```

#### 6. **Données semi-structurées issues d'IoT ou logs**

Événements et logs qui ont des structures variables.

```sql
CREATE TABLE evenements_iot (
    evenement_id SERIAL PRIMARY KEY,
    device_id VARCHAR(100),
    timestamp TIMESTAMPTZ,
    type_evenement VARCHAR(50),
    payload JSONB  -- Données de l'événement
);

-- Capteur de température
INSERT INTO evenements_iot (device_id, timestamp, type_evenement, payload) VALUES (
    'SENSOR_001',
    NOW(),
    'temperature',
    '{"temperature": 22.5, "unite": "celsius", "localisation": "salon"}'
);

-- Capteur de mouvement
INSERT INTO evenements_iot (device_id, timestamp, type_evenement, payload) VALUES (
    'SENSOR_002',
    NOW(),
    'mouvement',
    '{"detecte": true, "confiance": 0.95, "zone": "entree"}'
);
```

### ❌ N'utilisez PAS JSONB quand...

#### 1. **Données relationnelles classiques bien structurées**

```sql
-- ❌ MAUVAIS : Utiliser JSONB pour des relations simples
CREATE TABLE commandes (
    commande_id SERIAL PRIMARY KEY,
    client JSONB  -- Contient {id, nom, email, adresse...}
);

-- ✅ BON : Utiliser des relations normales
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255)
);

CREATE TABLE commandes (
    commande_id SERIAL PRIMARY KEY,
    client_id INTEGER REFERENCES clients(client_id)
);
```

**Pourquoi ?** Les relations entre entités doivent utiliser les clés étrangères pour garantir l'intégrité référentielle.

#### 2. **Recherches et filtres fréquents sur des attributs spécifiques**

Si vous recherchez souvent par un attribut précis, mettez-le en colonne dédiée.

```sql
-- ❌ MOINS BON : Tout dans JSONB
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    details JSONB  -- Contient {nom, prix, stock, categorie...}
);

-- Recherche lente
SELECT * FROM produits WHERE details->>'categorie' = 'electronique';

-- ✅ MEILLEUR : Colonnes pour critères de recherche fréquents
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255),
    prix NUMERIC(10,2),
    categorie VARCHAR(100),  -- Colonne indexable
    stock INTEGER,
    specifications JSONB  -- Reste en JSONB
);

-- Recherche rapide avec index classique
CREATE INDEX idx_produits_categorie ON produits(categorie);  
SELECT * FROM produits WHERE categorie = 'electronique';  
```

#### 3. **Calculs et agrégations fréquentes**

Les agrégations sur JSONB sont possibles mais moins performantes.

```sql
-- ❌ Calculs sur JSONB (moins optimal)
SELECT AVG((details->>'prix')::NUMERIC) FROM produits;

-- ✅ Calculs sur colonnes (optimal)
SELECT AVG(prix) FROM produits;
```

#### 4. **Besoin strict de validation de schéma au niveau base de données**

Le JSONB n'offre pas de validation native de structure (bien que vous puissiez créer des contraintes CHECK).

```sql
-- Relationnel : validation native
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'),
    age INTEGER CHECK (age >= 18)
);

-- JSONB : validation plus complexe
CREATE TABLE clients_jsonb (
    client_id SERIAL PRIMARY KEY,
    data JSONB,
    CONSTRAINT valid_email CHECK (data->>'email' ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'),
    CONSTRAINT valid_age CHECK ((data->>'age')::INTEGER >= 18)
);
```

---

## Manipulation de JSONB : Opérateurs et Fonctions

### Opérateurs d'Accès

#### 1. Opérateur `->` : Accès avec retour JSONB

Retourne un élément JSONB (permet le chaînage).

```sql
-- Créer une table de test
CREATE TABLE utilisateurs (
    utilisateur_id SERIAL PRIMARY KEY,
    profil JSONB
);

INSERT INTO utilisateurs (profil) VALUES (
    '{
        "nom": "Alice",
        "adresse": {
            "ville": "Paris",
            "code_postal": "75002"
        },
        "hobbies": ["lecture", "voyage"]
    }'
);

-- Accéder à un objet
SELECT profil->'adresse' FROM utilisateurs;
-- Résultat: {"ville": "Paris", "code_postal": "75002"}

-- Chaîner les accès
SELECT profil->'adresse'->'ville' FROM utilisateurs;
-- Résultat: "Paris" (toujours en JSONB)
```

#### 2. Opérateur `->>` : Accès avec retour TEXT

Retourne une chaîne de caractères (valeur finale).

```sql
-- Extraire comme texte
SELECT profil->>'nom' FROM utilisateurs;
-- Résultat: Alice (TEXT)

-- Chaînage avec ->>, dernière étape
SELECT profil->'adresse'->>'ville' FROM utilisateurs;
-- Résultat: Paris (TEXT)
```

**Règle pratique :**
- Utilisez `->` pour naviguer dans la structure
- Utilisez `->>` pour la valeur finale

#### 3. Opérateur `#>` : Chemin d'accès (array)

Accéder à un élément via un chemin sous forme de tableau.

```sql
-- Équivalent à profil->'adresse'->'ville'
SELECT profil #> '{adresse, ville}' FROM utilisateurs;
-- Résultat: "Paris" (JSONB)

-- Version TEXT
SELECT profil #>> '{adresse, ville}' FROM utilisateurs;
-- Résultat: Paris (TEXT)
```

#### 4. Accès aux éléments d'array

```sql
INSERT INTO utilisateurs (profil) VALUES (
    '{
        "nom": "Bob",
        "hobbies": ["natation", "cyclisme", "lecture"]
    }'
);

-- Accéder au premier hobby (index commence à 0)
SELECT profil->'hobbies'->0 FROM utilisateurs WHERE profil->>'nom' = 'Bob';
-- Résultat: "natation"

-- Extraire comme texte
SELECT profil->'hobbies'->>2 FROM utilisateurs WHERE profil->>'nom' = 'Bob';
-- Résultat: lecture
```

### Opérateurs de Comparaison et Inclusion

#### 1. Opérateur `@>` : Contient

Vérifie si le JSONB de gauche contient le JSONB de droite.

```sql
-- Trouver les utilisateurs qui ont Paris dans leur adresse
SELECT * FROM utilisateurs  
WHERE profil @> '{"adresse": {"ville": "Paris"}}';  

-- Trouver ceux qui ont "lecture" dans leurs hobbies
SELECT * FROM utilisateurs  
WHERE profil @> '{"hobbies": ["lecture"]}';  
```

#### 2. Opérateur `<@` : Est contenu dans

L'inverse de `@>`.

```sql
SELECT '{"nom": "Alice"}'::jsonb <@ '{"nom": "Alice", "age": 30}'::jsonb;
-- Résultat: true
```

#### 3. Opérateur `?` : Clé existe

Vérifie l'existence d'une clé au premier niveau.

```sql
-- Utilisateurs qui ont une clé "adresse"
SELECT * FROM utilisateurs WHERE profil ? 'adresse';

-- Vérifier plusieurs clés (OR)
SELECT * FROM utilisateurs WHERE profil ?| array['email', 'telephone'];

-- Vérifier plusieurs clés (AND)
SELECT * FROM utilisateurs WHERE profil ?& array['nom', 'adresse'];
```

### SQL/JSON Path (PostgreSQL 12+, standard SQL:2016)

PostgreSQL supporte également la syntaxe standard **SQL/JSON Path** pour interroger les données JSONB. Cette approche est plus portable et plus expressive pour les requêtes complexes :

```sql
-- Vérifier si un chemin existe
SELECT * FROM utilisateurs  
WHERE profil @? '$.adresse.ville';  

-- Filtrer avec une condition sur le chemin
SELECT * FROM utilisateurs  
WHERE profil @@ '$.adresse.ville == "Paris"';  

-- Extraire des valeurs avec jsonb_path_query
SELECT jsonb_path_query(profil, '$.hobbies[*]') AS hobby  
FROM utilisateurs;  

-- Extraire une valeur unique
SELECT jsonb_path_query_first(profil, '$.adresse.ville') AS ville  
FROM utilisateurs;  
```

**Opérateurs SQL/JSON Path :**
- `@?` : Le chemin existe-t-il ? (retourne booléen)  
- `@@` : Le prédicat est-il vrai ? (retourne booléen)  
- `jsonb_path_query()` : Extraire les valeurs correspondantes  
- `jsonb_path_query_first()` : Extraire la première valeur

> 💡 Les opérateurs traditionnels (`->`, `->>`, `@>`) restent plus concis pour les cas simples. SQL/JSON Path est préférable pour les requêtes complexes avec filtrage et pour la portabilité SQL standard.

### Fonctions de Manipulation

#### 1. `jsonb_set` : Modifier ou ajouter une valeur

```sql
-- Modifier la ville
UPDATE utilisateurs  
SET profil = jsonb_set(  
    profil,
    '{adresse, ville}',  -- Chemin
    '"Lyon"',            -- Nouvelle valeur (format JSON)
    true                 -- create_if_missing
)
WHERE utilisateur_id = 1;

-- Ajouter un champ
UPDATE utilisateurs  
SET profil = jsonb_set(  
    profil,
    '{telephone}',
    '"+33601020304"',
    true
)
WHERE utilisateur_id = 1;
```

#### 2. `jsonb_insert` : Insérer dans un array

```sql
-- Ajouter un hobby au début
UPDATE utilisateurs  
SET profil = jsonb_insert(  
    profil,
    '{hobbies, 0}',  -- Position 0
    '"photographie"',
    true  -- insert_before
)
WHERE utilisateur_id = 1;
```

#### 3. `jsonb_strip_nulls` : Supprimer les valeurs NULL

```sql
SELECT jsonb_strip_nulls('{"a": 1, "b": null, "c": 3}'::jsonb);
-- Résultat: {"a": 1, "c": 3}
```

#### 4. Opérateur `-` : Supprimer une clé

```sql
-- Supprimer la clé "telephone"
UPDATE utilisateurs  
SET profil = profil - 'telephone'  
WHERE utilisateur_id = 1;  

-- Supprimer plusieurs clés
UPDATE utilisateurs  
SET profil = profil - array['telephone', 'fax'];  
```

#### 5. Opérateur `||` : Fusion de JSONB

```sql
-- Fusionner deux objets (écrase les clés communes)
SELECT '{"a": 1, "b": 2}'::jsonb || '{"b": 3, "c": 4}'::jsonb;
-- Résultat: {"a": 1, "b": 3, "c": 4}

-- Ajouter des informations
UPDATE utilisateurs  
SET profil = profil || '{"newsletter": true, "langue": "fr"}'::jsonb  
WHERE utilisateur_id = 1;  
```

#### 6. `jsonb_each` : Décomposer en lignes

```sql
-- Transformer un objet en lignes clé-valeur
SELECT * FROM jsonb_each('{"a": 1, "b": 2, "c": 3}'::jsonb);
-- Résultat:
--  key | value
-- -----+-------
--  a   | 1
--  b   | 2
--  c   | 3
```

#### 7. `jsonb_array_elements` : Décomposer un array

```sql
-- Extraire chaque élément d'un array
SELECT jsonb_array_elements(profil->'hobbies') as hobby  
FROM utilisateurs  
WHERE profil->>'nom' = 'Bob';  
-- Résultat:
-- "natation"
-- "cyclisme"
-- "lecture"

-- Version TEXT
SELECT jsonb_array_elements_text(profil->'hobbies') as hobby  
FROM utilisateurs  
WHERE profil->>'nom' = 'Bob';  
-- Résultat:
-- natation
-- cyclisme
-- lecture
```

### Fonctions d'Agrégation

#### 1. `jsonb_agg` : Agréger en array JSON

```sql
-- Créer un array de tous les noms
SELECT jsonb_agg(profil->>'nom') FROM utilisateurs;
-- Résultat: ["Alice", "Bob", "Claire"]
```

#### 2. `jsonb_object_agg` : Agréger en objet JSON

```sql
-- Créer un objet clé-valeur
SELECT jsonb_object_agg(
    profil->>'nom',
    profil->'adresse'->>'ville'
) FROM utilisateurs;
-- Résultat: {"Alice": "Paris", "Bob": "Lyon"}
```

---

## Indexation JSONB

### Pourquoi indexer ?

Sans index, PostgreSQL doit scanner toute la table et parser chaque document JSONB. Avec un index, les recherches sont exponentiellement plus rapides.

### Index GIN (Generalized Inverted Index)

L'index **GIN** est l'index recommandé pour JSONB.

#### 1. Index GIN standard

```sql
-- Index sur toute la colonne JSONB
CREATE INDEX idx_utilisateurs_profil ON utilisateurs USING GIN (profil);

-- Optimise les requêtes avec @>, ?, ?|, ?&
SELECT * FROM utilisateurs WHERE profil @> '{"ville": "Paris"}';
```

#### 2. Index GIN avec `jsonb_path_ops`

Version optimisée qui consomme moins d'espace mais ne supporte que `@>`.

```sql
CREATE INDEX idx_utilisateurs_profil_ops ON utilisateurs  
USING GIN (profil jsonb_path_ops);  

-- Supporte uniquement @>
SELECT * FROM utilisateurs WHERE profil @> '{"adresse": {"ville": "Paris"}}';

-- Ne supporte PAS ? (existence de clé)
-- SELECT * FROM utilisateurs WHERE profil ? 'email';  -- N'utilisera pas l'index
```

**Quand utiliser `jsonb_path_ops` ?**
- ✅ Si vous utilisez principalement des requêtes `@>` (contient)  
- ✅ Index plus compact (30-50% plus petit)  
- ✅ Légèrement plus rapide pour `@>`  
- ❌ Ne supporte pas les opérateurs `?`, `?|`, `?&`

#### 3. Index sur une expression JSONB

Indexer un champ spécifique du JSONB.

```sql
-- Index sur l'email extrait
CREATE INDEX idx_utilisateurs_email ON utilisateurs ((profil->>'email'));

-- Index sur la ville
CREATE INDEX idx_utilisateurs_ville ON utilisateurs ((profil->'adresse'->>'ville'));

-- Requête optimisée
SELECT * FROM utilisateurs WHERE profil->>'email' = 'alice@example.com';  
SELECT * FROM utilisateurs WHERE profil->'adresse'->>'ville' = 'Paris';  
```

**Avantages :**
- ✅ Index classique B-Tree (plus rapide pour égalité/comparaison)  
- ✅ Moins d'espace que GIN  
- ✅ Supporte ORDER BY sur le champ

**Inconvénients :**
- ❌ Un index par champ nécessaire  
- ❌ Ne supporte que les requêtes exactes sur ce champ

### Stratégie d'Indexation

```sql
-- Produits avec JSONB
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255),
    prix NUMERIC(10,2),
    specifications JSONB
);

-- Index GIN général pour recherches flexibles
CREATE INDEX idx_produits_specs_gin ON produits USING GIN (specifications);

-- Index sur champs fréquemment recherchés
CREATE INDEX idx_produits_marque ON produits ((specifications->>'marque'));  
CREATE INDEX idx_produits_categorie ON produits ((specifications->>'categorie'));  

-- Index composé si nécessaire
CREATE INDEX idx_produits_prix_marque ON produits (prix, (specifications->>'marque'));
```

---

## Cas d'Usage Pratiques

### Cas 1 : Système de Configuration Multi-tenant

**Contexte :** Application SaaS où chaque client (tenant) a des configurations spécifiques.

```sql
CREATE TABLE tenants (
    tenant_id SERIAL PRIMARY KEY,
    nom_entreprise VARCHAR(255),
    configuration JSONB DEFAULT '{}'
);

-- Tenant 1 : E-commerce
INSERT INTO tenants (nom_entreprise, configuration) VALUES (
    'Boutique Mode',
    '{
        "modules_actifs": ["inventory", "shipping", "analytics"],
        "paiement": {
            "stripe_enabled": true,
            "paypal_enabled": false,
            "devise": "EUR"
        },
        "limites": {
            "nb_produits": 10000,
            "nb_utilisateurs": 50,
            "stockage_gb": 100
        },
        "design": {
            "theme": "moderne",
            "couleur_principale": "#3498db",
            "logo_url": "https://cdn.example.com/logo1.png"
        }
    }'
);

-- Tenant 2 : Blog
INSERT INTO tenants (nom_entreprise, configuration) VALUES (
    'Blog Voyage',
    '{
        "modules_actifs": ["blog", "comments", "newsletter"],
        "limites": {
            "nb_articles": 500,
            "nb_utilisateurs": 10,
            "stockage_gb": 20
        },
        "design": {
            "theme": "minimaliste",
            "couleur_principale": "#2ecc71"
        }
    }'
);

-- Récupérer les tenants avec module "inventory"
SELECT nom_entreprise, configuration  
FROM tenants  
WHERE configuration @> '{"modules_actifs": ["inventory"]}';  

-- Augmenter la limite de stockage pour un tenant
UPDATE tenants  
SET configuration = jsonb_set(  
    configuration,
    '{limites, stockage_gb}',
    '200'
)
WHERE tenant_id = 1;
```

### Cas 2 : Catalogue Produits E-commerce

**Contexte :** Différents types de produits avec attributs variables.

```sql
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255) NOT NULL,
    categorie VARCHAR(100) NOT NULL,
    prix NUMERIC(10,2) NOT NULL,
    stock INTEGER DEFAULT 0,
    attributs JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index pour recherches fréquentes
CREATE INDEX idx_produits_categorie ON produits(categorie);  
CREATE INDEX idx_produits_attributs_gin ON produits USING GIN (attributs);  
CREATE INDEX idx_produits_marque ON produits ((attributs->>'marque'));  

-- Livre
INSERT INTO produits (nom, categorie, prix, stock, attributs) VALUES (
    'Clean Code',
    'livre',
    39.99,
    50,
    '{
        "auteur": "Robert C. Martin",
        "isbn": "978-0132350884",
        "editeur": "Prentice Hall",
        "nb_pages": 464,
        "langue": "anglais",
        "date_publication": "2008-08-01",
        "format": "broché"
    }'
);

-- Smartphone
INSERT INTO produits (nom, categorie, prix, stock, attributs) VALUES (
    'iPhone 15 Pro',
    'electronique',
    1199.00,
    25,
    '{
        "marque": "Apple",
        "modele": "iPhone 15 Pro",
        "couleur": "titane naturel",
        "stockage": "256GB",
        "ram": "8GB",
        "ecran": "6.1 pouces",
        "batterie": "3274mAh",
        "5g": true,
        "garantie_mois": 24
    }'
);

-- Vêtement
INSERT INTO produits (nom, categorie, prix, stock, attributs) VALUES (
    'Jean Slim Fit',
    'vetement',
    79.90,
    100,
    '{
        "marque": "Levi''s",
        "modele": "511",
        "taille": "32/32",
        "couleur": "bleu foncé",
        "matiere": "98% coton, 2% élasthanne",
        "coupe": "slim",
        "origine": "Turquie",
        "entretien": "lavage 30°C"
    }'
);

-- Recherches complexes

-- 1. Tous les smartphones Apple avec 256GB
SELECT nom, prix, attributs  
FROM produits  
WHERE categorie = 'electronique'  
  AND attributs @> '{"marque": "Apple", "stockage": "256GB"}';

-- 2. Livres en français
SELECT nom, attributs->>'auteur' as auteur, prix  
FROM produits  
WHERE categorie = 'livre'  
  AND attributs->>'langue' = 'français';

-- 3. Produits avec garantie de plus de 12 mois
SELECT nom, categorie, attributs->>'garantie_mois' as garantie  
FROM produits  
WHERE (attributs->>'garantie_mois')::INTEGER > 12;  

-- 4. Recherche par marque (index optimisé)
SELECT nom, categorie, prix  
FROM produits  
WHERE attributs->>'marque' = 'Apple';  
```

### Cas 3 : Logs d'Événements et Analytics

**Contexte :** Tracking d'événements utilisateur avec données variables.

```sql
CREATE TABLE evenements (
    evenement_id BIGSERIAL PRIMARY KEY,
    utilisateur_id INTEGER,
    session_id VARCHAR(100),
    type_evenement VARCHAR(50),
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    proprietes JSONB
);

-- Index pour analyses
CREATE INDEX idx_evenements_type ON evenements(type_evenement);  
CREATE INDEX idx_evenements_timestamp ON evenements(timestamp);  
CREATE INDEX idx_evenements_utilisateur ON evenements(utilisateur_id);  
CREATE INDEX idx_evenements_props_gin ON evenements USING GIN (proprietes);  

-- Partition par mois pour performance
CREATE TABLE evenements_2025_11 PARTITION OF evenements  
FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');  

-- Page vue
INSERT INTO evenements (utilisateur_id, session_id, type_evenement, proprietes) VALUES (
    1001,
    'sess_abc123',
    'page_vue',
    '{
        "url": "/produits/smartphone-xyz",
        "referrer": "https://google.com",
        "user_agent": "Mozilla/5.0...",
        "duree_secondes": 45,
        "scroll_profondeur": 75
    }'
);

-- Ajout au panier
INSERT INTO evenements (utilisateur_id, session_id, type_evenement, proprietes) VALUES (
    1001,
    'sess_abc123',
    'ajout_panier',
    '{
        "produit_id": 5432,
        "nom_produit": "Smartphone XYZ",
        "prix": 799.00,
        "quantite": 1,
        "depuis_page": "/produits/smartphone-xyz"
    }'
);

-- Achat
INSERT INTO evenements (utilisateur_id, session_id, type_evenement, proprietes) VALUES (
    1001,
    'sess_abc123',
    'achat',
    '{
        "commande_id": 98765,
        "montant_total": 799.00,
        "articles": [
            {"produit_id": 5432, "prix": 799.00, "quantite": 1}
        ],
        "mode_paiement": "carte",
        "code_promo": "WELCOME10",
        "reduction": 79.90
    }'
);

-- Analytics : Taux de conversion
SELECT
    COUNT(DISTINCT CASE WHEN type_evenement = 'page_vue' THEN utilisateur_id END) as visiteurs,
    COUNT(DISTINCT CASE WHEN type_evenement = 'ajout_panier' THEN utilisateur_id END) as ajouts_panier,
    COUNT(DISTINCT CASE WHEN type_evenement = 'achat' THEN utilisateur_id END) as achats,
    ROUND(
        COUNT(DISTINCT CASE WHEN type_evenement = 'achat' THEN utilisateur_id END)::NUMERIC /
        COUNT(DISTINCT CASE WHEN type_evenement = 'page_vue' THEN utilisateur_id END) * 100,
        2
    ) as taux_conversion_pct
FROM evenements  
WHERE timestamp >= NOW() - INTERVAL '7 days';  

-- Produits les plus ajoutés au panier
SELECT
    proprietes->>'produit_id' as produit_id,
    proprietes->>'nom_produit' as nom_produit,
    COUNT(*) as nb_ajouts
FROM evenements  
WHERE type_evenement = 'ajout_panier'  
  AND timestamp >= NOW() - INTERVAL '30 days'
GROUP BY proprietes->>'produit_id', proprietes->>'nom_produit'  
ORDER BY nb_ajouts DESC  
LIMIT 10;  
```

### Cas 4 : Formulaires Dynamiques et Sondages

**Contexte :** Application de sondages où chaque sondage a des questions différentes.

```sql
CREATE TABLE sondages (
    sondage_id SERIAL PRIMARY KEY,
    titre VARCHAR(255),
    description TEXT,
    structure JSONB,  -- Questions et configuration
    actif BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE reponses_sondage (
    reponse_id SERIAL PRIMARY KEY,
    sondage_id INTEGER REFERENCES sondages(sondage_id),
    utilisateur_id INTEGER,
    reponses JSONB,  -- Réponses de l'utilisateur
    submitted_at TIMESTAMPTZ DEFAULT NOW()
);

-- Créer un sondage
INSERT INTO sondages (titre, description, structure) VALUES (
    'Satisfaction Client 2025',
    'Aidez-nous à améliorer nos services',
    '{
        "questions": [
            {
                "id": "q1",
                "type": "choix_unique",
                "question": "Comment évaluez-vous notre service ?",
                "options": ["Excellent", "Bon", "Moyen", "Mauvais"],
                "obligatoire": true
            },
            {
                "id": "q2",
                "type": "echelle",
                "question": "Recommanderiez-vous nos services ? (0-10)",
                "min": 0,
                "max": 10,
                "obligatoire": true
            },
            {
                "id": "q3",
                "type": "texte_libre",
                "question": "Suggestions d''amélioration",
                "max_caracteres": 500,
                "obligatoire": false
            },
            {
                "id": "q4",
                "type": "choix_multiples",
                "question": "Quels services utilisez-vous ?",
                "options": ["Support", "Documentation", "Formation", "API"],
                "obligatoire": false
            }
        ],
        "parametres": {
            "anonyme": false,
            "une_reponse_par_utilisateur": true
        }
    }'
);

-- Réponse d'un utilisateur
INSERT INTO reponses_sondage (sondage_id, utilisateur_id, reponses) VALUES (
    1,
    1001,
    '{
        "q1": "Excellent",
        "q2": 9,
        "q3": "Améliorer les temps de réponse du support",
        "q4": ["Support", "Documentation", "API"]
    }'
);

-- Analyse : Distribution des notes NPS (q2)
SELECT
    reponses->>'q2' as note,
    COUNT(*) as nb_reponses
FROM reponses_sondage  
WHERE sondage_id = 1  
GROUP BY reponses->>'q2'  
ORDER BY note;  

-- Analyse : Services les plus utilisés
SELECT
    service,
    COUNT(*) as nb_mentions
FROM reponses_sondage,
     jsonb_array_elements_text(reponses->'q4') as service
WHERE sondage_id = 1  
GROUP BY service  
ORDER BY nb_mentions DESC;  
```

---

## Modèle Hybride : Le Meilleur des Deux Mondes

La vraie puissance de PostgreSQL réside dans sa capacité à **combiner** relationnel et JSONB.

### Principe du Modèle Hybride

```
┌──────────────────────────────────────────────────┐
│  Colonnes relationnelles : Données structurées   │
│  - Clés primaires / étrangères                   │
│  - Champs fréquemment recherchés                 │
│  - Données critiques pour l'intégrité            │
└──────────────────────────────────────────────────┘
                      +
┌──────────────────────────────────────────────────┐
│  Colonnes JSONB : Données flexibles              │
│  - Attributs variables                           │
│  - Métadonnées non critiques                     │
│  - Extensions futures                            │
└──────────────────────────────────────────────────┘
```

### Exemple : Table Articles de Blog

```sql
CREATE TABLE articles (
    -- Colonnes relationnelles (structure fixe)
    article_id SERIAL PRIMARY KEY,
    titre VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    auteur_id INTEGER REFERENCES utilisateurs(utilisateur_id),
    statut VARCHAR(20) CHECK (statut IN ('brouillon', 'publie', 'archive')),
    date_publication TIMESTAMPTZ,
    nb_vues INTEGER DEFAULT 0,
    nb_commentaires INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    -- Colonne JSONB (contenu et métadonnées)
    contenu JSONB,
    metadata JSONB
);

-- Index mixtes
CREATE INDEX idx_articles_statut ON articles(statut);  
CREATE INDEX idx_articles_auteur ON articles(auteur_id);  
CREATE INDEX idx_articles_publication ON articles(date_publication);  
CREATE INDEX idx_articles_metadata_gin ON articles USING GIN (metadata);  
CREATE INDEX idx_articles_tags ON articles USING GIN ((metadata->'tags'));  

-- Insérer un article
INSERT INTO articles (
    titre,
    slug,
    auteur_id,
    statut,
    date_publication,
    contenu,
    metadata
) VALUES (
    'Introduction à PostgreSQL JSONB',
    'introduction-postgresql-jsonb',
    1,
    'publie',
    NOW(),
    '{
        "introduction": "PostgreSQL offre un support exceptionnel pour JSON...",
        "sections": [
            {
                "titre": "Qu''est-ce que JSONB ?",
                "contenu": "JSONB est un type de données binaire...",
                "ordre": 1
            },
            {
                "titre": "Avantages de JSONB",
                "contenu": "Les principaux avantages sont...",
                "ordre": 2
            }
        ],
        "conclusion": "JSONB est un outil puissant..."
    }',
    '{
        "tags": ["postgresql", "jsonb", "database", "tutorial"],
        "categorie": "Bases de données",
        "difficulte": "intermediaire",
        "duree_lecture_min": 15,
        "langue": "fr",
        "image_principale": "https://cdn.example.com/jsonb-intro.jpg",
        "seo": {
            "meta_description": "Découvrez comment utiliser JSONB...",
            "keywords": ["postgresql", "jsonb", "nosql"]
        }
    }'
);

-- Requêtes efficaces

-- 1. Articles publiés avec tag "postgresql" (utilise index GIN)
SELECT article_id, titre, slug, date_publication  
FROM articles  
WHERE statut = 'publie'  
  AND metadata @> '{"tags": ["postgresql"]}';

-- 2. Articles d'un auteur spécifique (utilise index sur auteur_id)
SELECT titre, metadata->>'categorie' as categorie, nb_vues  
FROM articles  
WHERE auteur_id = 1  
  AND statut = 'publie'
ORDER BY date_publication DESC;

-- 3. Articles de difficulté intermédiaire
SELECT titre, metadata->>'duree_lecture_min' as duree  
FROM articles  
WHERE metadata->>'difficulte' = 'intermediaire'  
  AND statut = 'publie';

-- 4. Top 10 des tags les plus utilisés
SELECT
    tag,
    COUNT(*) as nb_articles
FROM articles,
     jsonb_array_elements_text(metadata->'tags') as tag
WHERE statut = 'publie'  
GROUP BY tag  
ORDER BY nb_articles DESC  
LIMIT 10;  
```

**Avantages de cette approche :**
- ✅ **Performance** : Les requêtes fréquentes utilisent des index relationnels rapides  
- ✅ **Flexibilité** : Les métadonnées peuvent évoluer sans migration  
- ✅ **Intégrité** : Les relations (auteur, statut) restent sous contrôle de la base  
- ✅ **Simplicité** : Pas besoin de tables supplémentaires pour chaque métadonnée

---

## Validation et Contraintes avec JSONB

Bien que JSONB soit flexible, vous pouvez ajouter des contraintes pour garantir une certaine cohérence.

### Contraintes CHECK

```sql
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255),
    specifications JSONB,

    -- Contrainte : doit avoir une clé "marque"
    CONSTRAINT has_marque CHECK (specifications ? 'marque'),

    -- Contrainte : prix doit être positif
    CONSTRAINT prix_positif CHECK ((specifications->>'prix')::NUMERIC > 0),

    -- Contrainte : stock doit être un entier
    CONSTRAINT stock_entier CHECK (jsonb_typeof(specifications->'stock') = 'number')
);
```

### Fonction de Validation Personnalisée

```sql
-- Fonction pour valider la structure d'une adresse
CREATE OR REPLACE FUNCTION valider_adresse(adresse JSONB)  
RETURNS BOOLEAN AS $$  
BEGIN  
    -- Vérifier les champs obligatoires
    IF NOT (adresse ? 'rue' AND adresse ? 'ville' AND adresse ? 'code_postal') THEN
        RETURN FALSE;
    END IF;

    -- Vérifier le type des champs
    IF jsonb_typeof(adresse->'rue') != 'string' OR
       jsonb_typeof(adresse->'ville') != 'string' THEN
        RETURN FALSE;
    END IF;

    -- Valider le code postal (5 chiffres)
    IF NOT (adresse->>'code_postal' ~ '^\d{5}$') THEN
        RETURN FALSE;
    END IF;

    RETURN TRUE;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Utiliser dans une contrainte
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    adresse JSONB,
    CONSTRAINT adresse_valide CHECK (valider_adresse(adresse))
);

-- Test
INSERT INTO clients (nom, adresse) VALUES (
    'Alice',
    '{"rue": "15 rue de la Paix", "ville": "Paris", "code_postal": "75002"}'
);  -- OK

INSERT INTO clients (nom, adresse) VALUES (
    'Bob',
    '{"rue": "8 avenue", "ville": "Lyon"}'
);  -- ERREUR : code_postal manquant
```

### PostgreSQL 18 : Colonnes Virtuelles avec JSONB

```sql
-- Extraire des champs JSONB comme colonnes virtuelles
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    specifications JSONB,

    -- Colonnes virtuelles calculées à partir de JSONB
    marque TEXT GENERATED ALWAYS AS (specifications->>'marque') STORED,
    prix NUMERIC GENERATED ALWAYS AS ((specifications->>'prix')::NUMERIC) STORED,
    stock INTEGER GENERATED ALWAYS AS ((specifications->>'stock')::INTEGER) STORED
);

-- Index sur colonnes virtuelles (très performant)
CREATE INDEX idx_produits_marque ON produits(marque);  
CREATE INDEX idx_produits_prix ON produits(prix);  

-- Requête optimale
SELECT * FROM produits WHERE marque = 'Apple' AND prix < 1000;
-- Utilise les index sur les colonnes virtuelles, pas de parsing JSONB nécessaire
```

---

## Performance et Bonnes Pratiques

### 1. Privilégier les index appropriés

```sql
-- ❌ Mauvais : Pas d'index
CREATE TABLE logs (
    log_id SERIAL PRIMARY KEY,
    data JSONB
);

-- Recherche lente sur 1M de lignes
SELECT * FROM logs WHERE data @> '{"level": "error"}';
-- Seq Scan: 2500ms

-- ✅ Bon : Index GIN
CREATE INDEX idx_logs_data ON logs USING GIN (data);

-- Recherche rapide
SELECT * FROM logs WHERE data @> '{"level": "error"}';
-- Bitmap Index Scan: 15ms
```

### 2. Extraire les champs fréquemment recherchés

```sql
-- ❌ Moins performant
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    data JSONB
);

-- ✅ Plus performant (modèle hybride)
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE,  -- Colonne dédiée
    created_at TIMESTAMPTZ DEFAULT NOW(),
    data JSONB  -- Reste en JSONB
);

CREATE INDEX idx_users_email ON users(email);  -- Index B-Tree rapide
```

### 3. Utiliser des requêtes préparées

```python
# Python avec psycopg3
import psycopg

# ❌ Moins optimal : Parsing à chaque fois
cursor.execute(
    "SELECT * FROM produits WHERE specifications->>'marque' = %s",
    ('Apple',)
)

# ✅ Meilleur : Requête préparée
stmt = conn.prepare(
    "SELECT * FROM produits WHERE specifications->>'marque' = $1"
)
for marque in ['Apple', 'Samsung', 'Sony']:
    results = stmt.execute((marque,))
```

### 4. Éviter de stocker des données trop volumineuses

```sql
-- ❌ Mauvais : JSONB énorme (> 100KB)
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    contenu_complet JSONB  -- Texte complet + métadonnées
);

-- ✅ Mieux : Séparer contenu et métadonnées
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    contenu TEXT,  -- Texte séparé
    metadata JSONB  -- Seulement les métadonnées
);
```

### 5. Nettoyer régulièrement

```sql
-- Supprimer les clés inutiles périodiquement
UPDATE produits  
SET specifications = specifications - 'temp_field'  
WHERE specifications ? 'temp_field';  

-- Supprimer les NULL
UPDATE produits  
SET specifications = jsonb_strip_nulls(specifications);  

-- VACUUM après nettoyage important
VACUUM ANALYZE produits;
```

### 6. Monitorer la taille des JSONB

```sql
-- Vérifier la taille moyenne des JSONB
SELECT
    pg_size_pretty(AVG(pg_column_size(specifications))) as taille_moyenne,
    pg_size_pretty(MAX(pg_column_size(specifications))) as taille_max
FROM produits;

-- Identifier les JSONB trop gros (> 50KB)
SELECT
    produit_id,
    nom,
    pg_size_pretty(pg_column_size(specifications)) as taille
FROM produits  
WHERE pg_column_size(specifications) > 50000  
ORDER BY pg_column_size(specifications) DESC;  
```

---

## Migrations et Évolution du Schéma

### Ajouter un champ JSONB à une table existante

```sql
-- Table relationnelle existante
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255),
    prix NUMERIC(10,2)
);

-- Ajouter JSONB
ALTER TABLE produits ADD COLUMN specifications JSONB DEFAULT '{}';

-- Migrer les données existantes si nécessaire
UPDATE produits SET specifications = jsonb_build_object(
    'nom', nom,
    'prix', prix
);
```

### Transformer une colonne relationnelle en JSONB

```sql
-- Avant : Multiple colonnes
CREATE TABLE produits_old (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255),
    couleur VARCHAR(50),
    taille VARCHAR(10),
    poids_kg NUMERIC(5,2)
);

-- Après : JSONB consolidé
CREATE TABLE produits_new (
    produit_id SERIAL PRIMARY KEY,
    nom VARCHAR(255),
    attributs JSONB
);

-- Migration
INSERT INTO produits_new (produit_id, nom, attributs)  
SELECT  
    produit_id,
    nom,
    jsonb_build_object(
        'couleur', couleur,
        'taille', taille,
        'poids_kg', poids_kg
    )
FROM produits_old;
```

### Ajouter un champ à tous les JSONB existants

```sql
-- Ajouter "en_stock" par défaut
UPDATE produits  
SET specifications = specifications || '{"en_stock": true}'::jsonb  
WHERE NOT specifications ? 'en_stock';  
```

---

## JSONB vs Autres Solutions NoSQL

### JSONB vs MongoDB

| Critère | PostgreSQL JSONB | MongoDB |
|---------|------------------|---------|
| **Modèle de données** | Hybride (SQL + JSON) | Document pur |
| **Transactions ACID** | ✅ Complet | ✅ Depuis v4.0 |
| **Jointures** | ✅ Natif SQL | ⚠️ $lookup (moins performant) |
| **Requêtes complexes** | ✅ SQL puissant | ⚠️ Agrégation pipeline |
| **Typage fort** | ✅ Colonnes relationnelles | ❌ Schéma-less |
| **Indexation** | ✅ GIN, B-Tree, etc. | ✅ Diverses options |
| **Maturité** | ✅ 35+ ans | ✅ 15+ ans |
| **Écosystème** | ✅ Énorme | ✅ Très bon |

**Quand choisir PostgreSQL JSONB :**
- ✅ Besoin de transactions ACID rigoureuses  
- ✅ Requêtes complexes avec jointures  
- ✅ Modèle hybride (relationnel + flexible)  
- ✅ Réduction du nombre de technologies (une seule base)

**Quand choisir MongoDB :**
- ✅ 100% document-oriented sans relationnel  
- ✅ Sharding horizontal massif intégré  
- ✅ Équipe déjà experte MongoDB

### JSONB vs Colonnes Relationnelles Pures

| Besoin | Solution Recommandée |
|--------|---------------------|
| Données structurées stables | Colonnes relationnelles |
| Données variables entre enregistrements | JSONB |
| Recherches fréquentes par champ spécifique | Colonnes (ou hybride) |
| Intégrité référentielle critique | Relations FK |
| API REST JSON natif | JSONB |
| Agrégations et analytics | Colonnes (ou colonnes virtuelles) |
| Évolution rapide du schéma | JSONB |

---

## Checklist de Décision : Utiliser JSONB ?

Avant d'utiliser JSONB, répondez à ces questions :

### Questions à se poser

- [ ] **Les données ont-elles une structure variable ?**
  - Si oui → JSONB probablement utile
  - Si non → Colonnes relationnelles suffisent

- [ ] **Le champ est-il fréquemment recherché/filtré ?**
  - Si oui → Envisager une colonne dédiée (ou hybride)
  - Si non → JSONB approprié

- [ ] **L'intégrité référentielle est-elle critique ?**
  - Si oui → Utiliser FK relationnelles
  - Si non → JSONB acceptable

- [ ] **Les données proviennent-elles d'APIs JSON ?**
  - Si oui → JSONB très approprié
  - Si non → Évaluer la nécessité

- [ ] **Y aura-t-il des agrégations fréquentes sur ces données ?**
  - Si oui → Colonnes ou colonnes virtuelles (PG 18)
  - Si non → JSONB acceptable

- [ ] **L'équipe est-elle à l'aise avec JSONB ?**
  - Si oui → Go
  - Si non → Formation nécessaire

### Arbre de Décision

```
Est-ce une relation entre entités ?
├─ Oui → Utiliser FK (table relationnelle)
└─ Non → Continuer

Les données ont-elles une structure stable et prévisible ?
├─ Oui → Colonnes relationnelles
└─ Non → Continuer

Le champ est-il critique et fréquemment recherché ?
├─ Oui → Modèle HYBRIDE (colonne + JSONB)
└─ Non → JSONB pur

Ai-je besoin d'intégrité stricte au niveau base ?
├─ Oui → Ajouter contraintes CHECK
└─ Non → JSONB flexible

Conclusion : JSONB approprié ✓
```

---

## Conclusion

### Principes Clés à Retenir

1. **JSONB n'est pas un remplacement du relationnel**, c'est un **complément**  
2. **Privilégiez le modèle hybride** : relationnel pour la structure, JSONB pour la flexibilité  
3. **Indexez intelligemment** : GIN pour flexibilité, colonnes dédiées pour performance  
4. **Validez côté application** : JSONB est flexible, mais la cohérence reste importante  
5. **Utilisez JSONB quand la flexibilité apporte une vraie valeur**

### Forces de JSONB dans PostgreSQL

- ✅ **Flexibilité** sans sacrifier les transactions ACID  
- ✅ **Performance** avec indexation GIN optimisée  
- ✅ **Simplicité** pour stocker des données API  
- ✅ **Évolution** du schéma sans migration complexe  
- ✅ **Puissance** des opérateurs et fonctions natives  
- ✅ **Intégration** parfaite avec SQL relationnel

### Recommandations Finales

**Pour les débutants :**
- Commencez par un modèle relationnel classique
- Ajoutez JSONB progressivement pour les cas d'usage appropriés
- Expérimentez avec des données non critiques d'abord

**Pour les projets en production :**
- Utilisez le modèle hybride (colonnes + JSONB)
- Indexez systématiquement les JSONB fréquemment interrogés
- Documentez la structure attendue des JSONB
- Mettez en place une validation côté application

**Pour l'avenir :**
- PostgreSQL 18 améliore encore JSONB (colonnes virtuelles)
- L'écosystème continue d'évoluer (extensions, outils)
- JSONB devient un standard pour l'hybridation SQL/NoSQL

---

## Ressources pour Aller Plus Loin

### Documentation Officielle

- [PostgreSQL JSON Types](https://www.postgresql.org/docs/current/datatype-json.html)  
- [JSON Functions and Operators](https://www.postgresql.org/docs/current/functions-json.html)  
- [GIN Indexes](https://www.postgresql.org/docs/current/gin.html)

### Lectures Recommandées

- "The Art of PostgreSQL" - Dimitri Fontaine (chapitre sur JSONB)  
- "Mastering PostgreSQL" - Hans-Jürgen Schönig
- Blog PostgreSQL officiel : articles sur JSONB

### Outils Utiles

- **pg_trgm** : Extension pour recherche floue dans JSONB  
- **jsquery** : Extension pour requêtes JSON avancées  
- **pgAdmin** : Visualisation de structures JSONB

---

**Fin du Chapitre 11.2**

Ce chapitre vous a présenté JSONB, le type hybride qui fait de PostgreSQL une base unique combinant le meilleur du SQL et du NoSQL. Dans les chapitres suivants, nous explorerons d'autres techniques avancées de modélisation comme l'héritage de tables et le partitionnement.

⏭️ [Héritage de tables (Table Inheritance) : Concept et limites](/11-modelisation-avancee/03-heritage-de-tables.md)
