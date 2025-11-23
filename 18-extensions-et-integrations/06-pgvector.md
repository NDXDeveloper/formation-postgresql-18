🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.6. pgvector : IA, Embeddings et Recherche Vectorielle

## Introduction

Nous vivons une révolution de l'intelligence artificielle. ChatGPT, Midjourney, GitHub Copilot... Ces technologies ont un point commun : elles utilisent des **vecteurs** (listes de nombres) pour représenter et comprendre le monde.

Et si votre base de données PostgreSQL pouvait, elle aussi, comprendre le *sens* des choses ? Rechercher non pas par mots-clés exacts, mais par **similarité sémantique** ? Permettre à ChatGPT d'accéder à vos données privées ? Recommander des produits en comprenant les préférences ?

C'est exactement ce que permet **pgvector**, l'extension PostgreSQL qui transforme votre base de données en moteur de recherche vectorielle intelligent.

### Analogie : De la bibliothèque classique à la bibliothèque magique

**Bibliothèque classique (recherche traditionnelle)** :
```
Vous : "Je cherche un livre sur les 'dauphins'"
Bibliothécaire : *Cherche le mot "dauphin" dans les titres*
                 "Voici 'Les Dauphins de l'Atlantique'"

❌ Manque : "Cétacés marins", "Intelligence animale",
           "Mammifères aquatiques" (pas le mot "dauphin")
```

**Bibliothèque magique (recherche vectorielle)** :
```
Vous : "Je cherche un livre sur les dauphins"
Bibliothécaire magique : *Comprend le CONCEPT*
                         "Voici des livres sur :"
                         - Les dauphins (correspondance exacte)
                         - Les cétacés (même famille)
                         - L'intelligence des mammifères marins
                         - La communication animale sous-marine
                         - Les orques (cousins proches)

✅ Comprend le sens, pas seulement les mots !
```

**pgvector** est ce bibliothécaire magique pour PostgreSQL.

---

## Le Contexte : L'Ère de l'IA et des Embeddings

### L'explosion de l'IA (2020-2025)

Ces dernières années ont vu une explosion des applications d'intelligence artificielle :

| Année | Innovation | Impact |
|-------|-----------|--------|
| **2020** | GPT-3 | Génération de texte humain-like |
| **2021** | DALL-E | Génération d'images depuis texte |
| **2022** | ChatGPT | Chatbots conversationnels grand public |
| **2022** | Stable Diffusion | IA générative open-source |
| **2023** | GPT-4 | Modèles multimodaux (texte + images) |
| **2023** | Explosion RAG | ChatGPT + données privées |
| **2024** | LLMs locaux | Llama, Mistral (open-source) |
| **2025** | IA embarquée | IA dans les bases de données |

**Constat** : L'IA n'est plus réservée aux géants de la tech. N'importe quelle application peut maintenant intégrer des capacités d'IA.

### Le Défi : Où Stocker les Données de l'IA ?

Toutes ces IA fonctionnent avec des **vecteurs** (représentations numériques). Le problème :

```
Application moderne :
- Données traditionnelles : PostgreSQL ✅
- Fichiers : S3 / Storage ✅
- Cache : Redis ✅
- Vecteurs pour IA : ??? 🤔
```

**Solutions historiques** :

1. **Bases vectorielles dédiées** (Pinecone, Weaviate, Milvus)
   - ✅ Optimisées pour vecteurs
   - ❌ Nouvelle infrastructure à gérer
   - ❌ Synchronisation données PostgreSQL ↔ Base vectorielle

2. **Fichiers locaux / S3**
   - ✅ Simple pour prototypage
   - ❌ Pas de requêtes SQL
   - ❌ Pas de transactions
   - ❌ Performances médiocres

**La révolution pgvector** : Et si vos vecteurs vivaient directement dans PostgreSQL ?

```
Application moderne avec pgvector :
- Données traditionnelles : PostgreSQL ✅
- Vecteurs IA : PostgreSQL (pgvector) ✅
- Tout au même endroit ! 🎉
```

---

## Qu'est-ce que pgvector ?

### Définition

**pgvector** est une extension open-source pour PostgreSQL qui ajoute :
- Un type de données `vector` pour stocker des tableaux de nombres (embeddings)
- Des opérateurs pour calculer la similarité entre vecteurs
- Des index spécialisés pour recherches ultra-rapides

**Développé par** : Ankane (Andrew Kane)
**Première version** : 2021
**Licence** : Open-source (PostgreSQL License)
**GitHub** : https://github.com/pgvector/pgvector
**Stars** : 10K+ (très populaire !)

### Architecture conceptuelle

```
┌──────────────────────────────────────────────────────────┐
│                PostgreSQL (votre base existante)         │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  Tables      │  │  Tables      │  │  Extension   │    │
│  │  classiques  │  │  classiques  │  │  pgvector    │    │
│  │              │  │              │  │              │    │
│  │ id | nom     │  │ user | text  │  │ ADD TYPE     │    │
│  │ 1  | Alice   │  │ 123  | ...   │  │ vector(N)    │    │
│  │ 2  | Bob     │  │ 456  | ...   │  │              │    │
│  └──────────────┘  └──────────────┘  │ ADD OPERATORS│    │
│                                      │ <->, <=>, <#>│    │
│  ┌─────────────────────────────────┐ │              │    │
│  │  Table avec vecteurs            │ │ ADD INDEXES  │    │
│  │                                 │ │ IVFFlat,HNSW │    │
│  │ id | product | embedding        │ └──────────────┘    │
│  │ 1  | Laptop  | [0.1, 0.2, ...]  │                     │
│  │ 2  | Mouse   | [0.5, 0.3, ...]  │                     │
│  └─────────────────────────────────┘                     │
│                                                          │
│  Requêtes SQL classiques + Recherche vectorielle !       │
└──────────────────────────────────────────────────────────┘
```

### Fonctionnalités clés

✅ **Type de données vector** : Stocker des vecteurs de 1 à 16 000 dimensions

✅ **Opérateurs de distance** :
- Distance L2 (euclidienne) : `<->`
- Distance cosine : `<=>`
- Produit scalaire (inner product) : `<#>`

✅ **Index spécialisés** :
- IVFFlat : Rapide, économe en espace
- HNSW : Ultra-rapide, haute précision

✅ **Intégration SQL native** :
```sql
SELECT * FROM products
ORDER BY embedding <=> '[0.1, 0.2, 0.3]'::vector
LIMIT 10;
```

✅ **Performance** : Recherche parmi millions de vecteurs en millisecondes

---

## Qu'est-ce qu'un Embedding ?

### Définition simple

Un **embedding** (ou plongement en français) est la transformation d'un objet (texte, image, son, vidéo) en une **liste de nombres** (vecteur) qui capture son *sens* ou ses *caractéristiques*.

```
Objet réel → Modèle d'IA → Vecteur de nombres

"J'aime les chats" → [modèle] → [0.023, -0.154, 0.387, ..., 0.892]
                                  ↑ 1536 nombres (dimensions)
```

### Analogie : Le code postal du sens

**Code postal géographique** :
- Paris 75001 : Centre, Louvre, chic
- Paris 75018 : Montmartre, bohème, artistes
- Paris 75020 : Belleville, populaire, multiculturel

Les codes proches → Quartiers proches géographiquement

**Embedding sémantique** :
- "chat" → [0.5, 0.3, 0.1, ...]
- "chaton" → [0.51, 0.29, 0.12, ...] ← Proche !
- "voiture" → [0.1, 0.8, 0.9, ...] ← Éloigné

Les vecteurs proches → Concepts proches sémantiquement !

### Exemples concrets

#### Embedding de texte

```python
# Avec OpenAI text-embedding-3-small
texte = "PostgreSQL est une excellente base de données"

embedding = get_embedding(texte)
# → [0.023, -0.154, 0.387, 0.891, ..., 0.234]
#    1536 dimensions (nombres)

# Textes similaires auront des embeddings proches !
texte1 = "PostgreSQL est une excellente base de données"
texte2 = "PostgreSQL est un SGBD performant"
texte3 = "J'aime les pizzas"

distance(texte1, texte2) = 0.05  # Très proches !
distance(texte1, texte3) = 0.89  # Très éloignés
```

#### Embedding d'image

```python
# Avec un modèle CNN (ResNet, VGG, etc.)
image = load_image("chat.jpg")

embedding = get_image_embedding(image)
# → [0.123, 0.456, -0.234, ..., 0.789]
#    2048 dimensions

# Images similaires (autres chats) → embeddings proches
# Images différentes (voitures) → embeddings éloignés
```

#### Embedding de produit e-commerce

```python
produit = {
    'nom': 'Ordinateur portable Dell XPS 15',
    'description': 'PC haut de gamme pour développeurs...',
    'categorie': 'Informatique',
    'prix': 1899
}

# Combiner texte pour embedding
texte_produit = f"{produit['nom']} {produit['description']}"
embedding = get_embedding(texte_produit)

# Produits similaires → embeddings proches
# "Laptop HP", "MacBook Pro" → proches
# "Chaise de bureau" → éloigné
```

### Propriétés magiques des embeddings

#### 1. Synonymes automatiques

```
"voiture" ≈ "automobile" ≈ "véhicule" ≈ "bagnole"
Tous ont des embeddings proches, sans programmation explicite !
```

#### 2. Contexte capturé

```
"banque" (institution financière) vs "banque" (mobilier)
Les embeddings seront différents selon le contexte !
```

#### 3. Multilingue

```
"hello" (anglais) ≈ "bonjour" (français) ≈ "hola" (espagnol)
Les modèles multilingues alignent les langues !
```

#### 4. Relations sémantiques

```
Opérations vectorielles :
king - man + woman ≈ queen
Paris - France + Italy ≈ Rome

(Approximativement, pas exact)
```

### D'où viennent les embeddings ?

#### Modèles commerciaux (API)

| Fournisseur | Modèle | Dimensions | Coût |
|------------|--------|-----------|------|
| **OpenAI** | text-embedding-3-small | 1536 | $0.02 / 1M tokens |
| **OpenAI** | text-embedding-3-large | 3072 | $0.13 / 1M tokens |
| **Cohere** | embed-english-v3.0 | 1024 | $0.10 / 1M tokens |
| **Google** | textembedding-gecko | 768 | $0.025 / 1M tokens |
| **Voyage AI** | voyage-2 | 1024 | $0.12 / 1M tokens |

**Avantages** : Qualité élevée, simple à utiliser
**Inconvénients** : Coût, dépendance externe, données envoyées à des tiers

#### Modèles open-source (local)

| Modèle | Dimensions | Qualité | Usage |
|--------|-----------|---------|-------|
| **Sentence-BERT (all-MiniLM-L6-v2)** | 384 | ⭐⭐⭐ | Général, rapide |
| **Sentence-BERT (all-mpnet-base-v2)** | 768 | ⭐⭐⭐⭐ | Haute qualité |
| **BGE-large-en** | 1024 | ⭐⭐⭐⭐⭐ | SOTA (State of the Art) |
| **E5-large-v2** | 1024 | ⭐⭐⭐⭐ | Multilingue |
| **Instructor** | 768 | ⭐⭐⭐⭐ | Avec instructions |

**Avantages** : Gratuit, privé, pas de limite d'usage
**Inconvénients** : Nécessite infrastructure (GPU/CPU), maintenance

#### Modèles spécialisés

- **Images** : ResNet, VGG, CLIP, Inception
- **Audio** : Wav2Vec, Whisper
- **Code** : CodeBERT, GraphCodeBERT
- **Multimodal** : CLIP (texte + images), DALL-E

---

## Qu'est-ce que la Recherche Vectorielle ?

### Définition

La **recherche vectorielle** (vector search ou similarity search) consiste à trouver des vecteurs "proches" d'un vecteur de référence, selon une métrique de distance.

### Métaphore : Chercher des amis

**Recherche classique (base de données)** :
```sql
SELECT * FROM users WHERE age = 25 AND ville = 'Paris';
-- Critères exacts, booléens (vrai/faux)
```

**Recherche vectorielle** :
```sql
SELECT * FROM users
ORDER BY preferences_vector <=> mon_profil_vector
LIMIT 10;
-- Similarité (gradient de 0 à 1), les "plus proches"
```

C'est comme chercher des amis avec des goûts similaires aux vôtres, pas des clones exacts !

### Comment ça fonctionne ?

#### Étape 1 : Vectorisation

```
Documents → Embeddings

Doc 1 : "Python programming"     → [0.1, 0.8, 0.3, ...]
Doc 2 : "Python snake species"   → [0.7, 0.2, 0.1, ...]
Doc 3 : "Java programming"       → [0.12, 0.79, 0.31, ...]
Doc 4 : "Chocolate cake recipe"  → [0.5, 0.1, 0.9, ...]
```

#### Étape 2 : Stockage dans PostgreSQL

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    texte TEXT,
    embedding vector(1536)  -- pgvector !
);

INSERT INTO documents (texte, embedding) VALUES
    ('Python programming', '[0.1, 0.8, 0.3, ...]'),
    ('Python snake species', '[0.7, 0.2, 0.1, ...]'),
    -- ...
```

#### Étape 3 : Recherche de similarité

```
Requête : "Learn coding in Python"
    ↓
Embedding : [0.11, 0.81, 0.29, ...]
    ↓
Recherche :
SELECT * FROM documents
ORDER BY embedding <=> '[0.11, 0.81, 0.29, ...]'::vector
LIMIT 3;
    ↓
Résultats :
1. "Python programming"      (distance: 0.02) ← Très proche !
2. "Java programming"        (distance: 0.15) ← Proche
3. "Python snake species"    (distance: 0.67) ← Éloigné
```

### Visualisation (2D simplifiée)

```
       Similarité Sémantique (espace 2D simplifié)

     Programming ↑
          |
      1 ★ Python programming
          |
          |
      3 ★ Java programming
          |
          |
  ────────┼────────────────────→ Biology
          |
          |
      2 ★ Python snake
          |
          |    4 ★ Chocolate cake
          |

★ = Documents dans l'espace vectoriel
Requête = "Learn Python coding"
Distance = Proximité dans l'espace
```

**En réalité** : Espace à 384, 768, 1536 ou 3072 dimensions (impossible à visualiser) !

---

## Pourquoi pgvector dans PostgreSQL ?

### Avantages de l'Intégration

#### 1. Données au même endroit

```sql
-- UNE SEULE table pour tout !
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom TEXT,
    description TEXT,
    prix NUMERIC,
    stock INTEGER,
    categorie TEXT,
    embedding vector(1536),  -- Pour recherche sémantique
    created_at TIMESTAMP
);

-- Requête combinée : classique + vectorielle
SELECT
    nom, prix, stock,
    embedding <=> '[...]'::vector AS similarite
FROM produits
WHERE categorie = 'Électronique'  -- Filtre classique
  AND prix < 1000                  -- Filtre classique
  AND stock > 0                    -- Filtre classique
ORDER BY embedding <=> '[...]'::vector  -- Recherche vectorielle
LIMIT 10;
```

**Vs base vectorielle séparée** : Synchronisation cauchemardesque !

#### 2. Transactions ACID

```sql
BEGIN;

-- Insérer produit
INSERT INTO produits (nom, description, prix, embedding)
VALUES ('Nouveau produit', 'Description...', 99.99, '[...]');

-- Insérer dans table liée
INSERT INTO inventory (produit_id, quantite)
VALUES (CURRVAL('produits_id_seq'), 100);

COMMIT;  -- Tout ou rien, garanti !
```

**Vs base vectorielle séparée** : Cohérence difficile à garantir.

#### 3. Écosystème PostgreSQL

```sql
-- Utiliser TOUTES les fonctionnalités PostgreSQL :

-- Jointures
SELECT p.nom, c.nom_client, p.embedding <=> q.embedding AS sim
FROM produits p
JOIN commandes c ON p.id = c.produit_id
CROSS JOIN LATERAL (SELECT embedding FROM query) q;

-- Vues matérialisées
CREATE MATERIALIZED VIEW produits_populaires_vecteurs AS
SELECT * FROM produits WHERE ventes > 1000;

-- Partitionnement
CREATE TABLE produits_partitioned (
    -- ...
    embedding vector(1536)
) PARTITION BY RANGE (created_at);

-- Sécurité (RLS)
ALTER TABLE produits ENABLE ROW LEVEL SECURITY;

-- Réplication
-- Tout fonctionne, y compris les vecteurs !
```

#### 4. Compétences existantes

```
Votre équipe connaît déjà :
✅ PostgreSQL
✅ SQL
✅ psql, pgAdmin
✅ Backups, réplication, monitoring

Avec pgvector :
✅ Même outils
✅ Même processus
✅ Courbe d'apprentissage minimale
```

**Vs base vectorielle séparée** : Nouvelle stack à apprendre.

#### 5. Coût et Simplicité

```
Sans pgvector :
- PostgreSQL : $X/mois
- Base vectorielle (Pinecone, etc.) : $Y/mois
- Infrastructure supplémentaire
- Synchronisation à gérer

Avec pgvector :
- PostgreSQL : $X/mois
- C'est tout ! 🎉
```

### Cas d'Usage de pgvector

#### 1. Recherche Sémantique

**Application** : Moteur de recherche intelligent

```
Utilisateur tape : "comment réparer fuite eau"

Recherche traditionnelle (Full-Text) :
→ Trouve : documents avec "réparer" ET "fuite" ET "eau"

Recherche sémantique (pgvector) :
→ Trouve : "colmater canalisation", "urgence plomberie",
          "tuyau percé", "dégât des eaux"

Comprend le SENS, pas juste les mots !
```

#### 2. RAG (Retrieval-Augmented Generation)

**Application** : ChatGPT avec vos données privées

```
┌─────────────────────────────────────────┐
│ Question : "Quelle est notre politique  │
│            de congés ?"                 │
└──────────┬──────────────────────────────┘
           │
           ↓ Embedding
┌──────────▼──────────────────────────────┐
│ pgvector : Recherche docs similaires    │
│ SELECT * FROM knowledge_base            │
│ ORDER BY embedding <=> question_emb     │
└──────────┬──────────────────────────────┘
           │
           ↓ Top 3 documents pertinents
┌──────────▼──────────────────────────────┐
│ ChatGPT : Génère réponse basée sur      │
│           les documents fournis         │
└──────────┬──────────────────────────────┘
           │
           ↓
    Réponse précise et sourcée !
```

#### 3. Recommandation

**Application** : E-commerce, Netflix-like

```sql
-- "Les clients qui ont aimé X ont aussi aimé..."
SELECT
    p2.nom,
    p2.prix,
    p1.embedding <=> p2.embedding AS similarite
FROM produits p1
CROSS JOIN produits p2
WHERE p1.id = 123  -- Produit actuel
  AND p2.id != 123
ORDER BY similarite ASC  -- Les plus similaires
LIMIT 10;
```

#### 4. Déduplication / Détection de Plagiat

```sql
-- Trouver documents très similaires (potentiels doublons)
SELECT
    d1.titre AS doc1,
    d2.titre AS doc2,
    1 - (d1.embedding <=> d2.embedding) AS similarite
FROM documents d1
CROSS JOIN documents d2
WHERE d1.id < d2.id
  AND d1.embedding <=> d2.embedding < 0.1  -- 90%+ similaires
ORDER BY similarite DESC;
```

#### 5. Classification Automatique

```sql
-- Classifier un nouveau texte dans la catégorie la plus proche
WITH nouveau_texte AS (
    SELECT '[0.123, 0.456, ...]'::vector(1536) AS embedding
)
SELECT
    c.categorie,
    c.description,
    1 - (c.embedding_reference <=> nt.embedding) AS confiance
FROM categories c, nouveau_texte nt
ORDER BY confiance DESC
LIMIT 1;

-- Résultat : "Support Technique" (confiance: 94%)
```

#### 6. Recherche d'Images Similaires

```sql
-- Computer Vision + pgvector
CREATE TABLE images (
    id SERIAL PRIMARY KEY,
    url TEXT,
    description TEXT,
    embedding vector(2048)  -- ResNet-50, VGG, etc.
);

-- Recherche "images similaires"
SELECT url, description
FROM images
ORDER BY embedding <-> '[...]'::vector(2048)
LIMIT 10;
```

---

## L'Écosystème pgvector

### Langages et Frameworks

#### Python

```python
# psycopg2 / psycopg3
import psycopg2
from pgvector.psycopg2 import register_vector

conn = psycopg2.connect("postgresql://...")
register_vector(conn)

# LangChain
from langchain.vectorstores import PGVector

vectorstore = PGVector(
    connection_string="postgresql://...",
    embedding_function=embeddings
)

# LlamaIndex
from llama_index import VectorStoreIndex
from llama_index.vector_stores import PGVectorStore

vector_store = PGVectorStore.from_params(...)
```

#### Node.js / TypeScript

```typescript
// pg + pgvector
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: 'postgresql://...'
});

// Prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [vector]
}
```

#### Ruby

```ruby
# pgvector-ruby
require 'pgvector'

# Active Record
class Item < ApplicationRecord
  has_neighbors :embedding
end

Item.nearest_neighbors(:embedding, [1, 2, 3], distance: 'cosine').limit(5)
```

#### Go

```go
// pgx + pgvector
import (
    "github.com/jackc/pgx/v5"
    "github.com/pgvector/pgvector-go"
)

conn, _ := pgx.Connect(context.Background(), "postgresql://...")

vec := pgvector.NewVector([]float32{1, 2, 3})
```

### Outils et Services

#### Supabase

**Supabase** inclut pgvector par défaut !

```javascript
// Supabase JS
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(SUPABASE_URL, SUPABASE_KEY)

// Recherche vectorielle
const { data } = await supabase
  .rpc('match_documents', {
    query_embedding: embedding,
    match_threshold: 0.7,
    match_count: 10
  })
```

#### Neon

**Neon** (PostgreSQL serverless) supporte pgvector.

#### Timescale

**Timescale** (séries temporelles) + pgvector = Vecteurs temporels !

### Comparaison avec Bases Vectorielles Dédiées

| Critère | pgvector | Pinecone | Weaviate | Milvus |
|---------|----------|----------|----------|--------|
| **Type** | Extension PG | SaaS dédié | Open-source | Open-source |
| **Performance** | ⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐⭐ Optimale | ⭐⭐⭐⭐⭐ Optimale | ⭐⭐⭐⭐⭐ Optimale |
| **Échelle** | Millions | Milliards | Milliards | Milliards |
| **Setup** | 🟢 Trivial (1 ligne) | 🟢 API key | 🟡 Docker/K8s | 🟡 Docker/K8s |
| **Coût** | 🟢 Inclus PG | 🔴 $70+/mois | 🟢 Gratuit | 🟢 Gratuit |
| **Données mixtes** | ✅ SQL + vecteurs | ❌ Vecteurs seuls | ⚠️ Limité | ❌ Vecteurs seuls |
| **Transactions** | ✅ ACID | ❌ | ⚠️ Limité | ❌ |
| **Écosystème** | ✅ PostgreSQL | ⚠️ Propriétaire | ✅ Riche | ✅ Riche |

**Conclusion** :
- **pgvector** : 95% des cas d'usage, simplicité maximale
- **Bases dédiées** : Échelle extrême (>100M vecteurs), optimisation maximale

---

## Installation de pgvector

### Vérifier la Disponibilité

```sql
-- Vérifier si pgvector est disponible
SELECT * FROM pg_available_extensions WHERE name = 'vector';
```

### Méthode 1 : Package Manager

#### Ubuntu / Debian

```bash
sudo apt install postgresql-16-pgvector
```

#### macOS (Homebrew)

```bash
brew install pgvector
```

#### Windows

Télécharger depuis https://github.com/pgvector/pgvector/releases

### Méthode 2 : Compilation depuis les Sources

```bash
# Cloner le repository
git clone https://github.com/pgvector/pgvector.git
cd pgvector

# Compiler
make
sudo make install
```

### Méthode 3 : Services Managés

**Supabase** : ✅ Inclus par défaut
**Neon** : ✅ Disponible
**AWS RDS** : ⚠️ Certaines instances
**Azure Database** : ⚠️ Certaines instances
**Google Cloud SQL** : ⚠️ Vérifier disponibilité

### Activer l'Extension

```sql
-- Activer pgvector dans votre base
CREATE EXTENSION vector;

-- Vérifier l'installation
SELECT extversion FROM pg_extension WHERE extname = 'vector';
```

**Résultat** :
```
 extversion
------------
 0.7.0
```

### Utilisation Basique

```sql
-- Créer une table avec vecteurs
CREATE TABLE items (
    id SERIAL PRIMARY KEY,
    nom TEXT,
    embedding vector(3)  -- Vecteur de 3 dimensions
);

-- Insérer des vecteurs
INSERT INTO items (nom, embedding) VALUES
    ('item1', '[1, 2, 3]'),
    ('item2', '[4, 5, 6]'),
    ('item3', '[7, 8, 9]');

-- Recherche de similarité
SELECT
    nom,
    embedding <-> '[3, 1, 2]' AS distance
FROM items
ORDER BY distance
LIMIT 2;
```

**Résultat** :
```
  nom   | distance
--------+----------
 item1  |   2.45
 item2  |   5.10
```

Ça marche ! 🎉

---

## Architecture d'une Application avec pgvector

### Stack Typique

```
┌─────────────────────────────────────────────────────────────┐
│                  APPLICATION FRONT-END                      │
│              (React, Vue, Next.js, etc.)                    │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTP/REST
┌──────────────────────▼─────────────────────────────────────┐
│                  BACKEND API                               │
│            (Python FastAPI, Node.js Express,               │
│             Ruby on Rails, etc.)                           │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Business Logic                                      │  │
│  │  - Authentification                                  │  │
│  │  - Validation                                        │  │
│  │  - Orchestration                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌──────────────────┐           ┌──────────────────────┐   │
│  │  Embedding       │           │  Search & RAG        │   │
│  │  Generation      │           │  Logic               │   │
│  │  (OpenAI API,    │           │                      │   │
│  │   Local model)   │           │                      │   │
│  └────────┬─────────┘           └──────────┬───────────┘   │
└───────────┼────────────────────────────────┼───────────────┘
            │                                │
            │ embeddings                     │ SQL queries
            │                                │
┌───────────▼────────────────────────────────▼────────────────┐
│              PostgreSQL + pgvector                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Users      │  │  Documents   │  │   Products   │       │
│  │   (classic)  │  │  + vectors   │  │  + vectors   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                             │
│  Index HNSW pour recherche rapide                           │
└─────────────────────────────────────────────────────────────┘
```

### Workflow Typique

#### 1. Indexation (une fois / mise à jour)

```python
# Backend
def indexer_document(titre, contenu):
    # 1. Générer embedding
    embedding = openai.embeddings.create(
        input=contenu,
        model="text-embedding-3-small"
    ).data[0].embedding

    # 2. Stocker dans PostgreSQL
    conn.execute("""
        INSERT INTO documents (titre, contenu, embedding)
        VALUES (%s, %s, %s)
    """, (titre, contenu, embedding))
```

#### 2. Recherche (à chaque requête)

```python
# Backend
@app.get("/search")
def search(query: str):
    # 1. Générer embedding de la requête
    query_emb = openai.embeddings.create(
        input=query,
        model="text-embedding-3-small"
    ).data[0].embedding

    # 2. Rechercher dans PostgreSQL
    results = conn.execute("""
        SELECT titre, contenu,
               1 - (embedding <=> %s::vector) AS similarite
        FROM documents
        ORDER BY embedding <=> %s::vector
        LIMIT 10
    """, (query_emb, query_emb))

    return [dict(r) for r in results]
```

---

## Limites et Considérations

### Limites Techniques

❌ **Dimensions maximales** : 16 000 (largement suffisant pour la plupart des modèles)

❌ **Performance sur TRÈS grands volumes** : Au-delà de 100 millions de vecteurs, les bases dédiées (Milvus, etc.) peuvent être plus performantes

❌ **Pas de sharding automatique** : Pour distribuer sur plusieurs serveurs, nécessite configuration manuelle

❌ **Mémoire** : Les index HNSW sont gourmands en RAM (prévoir 2-3× la taille des données)

### Considérations Coût

**Embeddings API** (exemple OpenAI) :
- text-embedding-3-small : $0.02 / 1M tokens
- Pour 100K documents de 500 tokens → ~$1

**Stockage** (exemple) :
- 1M vecteurs de 1536 dimensions → ~6 GB
- Index HNSW → ~12 GB supplémentaires
- Total : ~18 GB

**Compute** :
- Recherche : ~5-50ms par requête (avec index)
- Nécessite CPU/RAM conséquent pour gros volumes

### Alternatives à Considérer

**Quand utiliser pgvector** :
- ✅ < 10M vecteurs
- ✅ Besoins SQL + vecteurs
- ✅ Simplicité prioritaire
- ✅ Infrastructure PostgreSQL existante

**Quand utiliser base vectorielle dédiée** :
- ✅ > 100M vecteurs
- ✅ Vecteurs uniquement (pas de données relationnelles)
- ✅ Optimisation maximale requise
- ✅ Équipe dédiée DevOps

---

## Roadmap et Futur

### Évolutions Récentes (2024-2025)

✅ **pgvector 0.7.0** (2024) :
- Amélioration performances HNSW
- Support de nouvelles distances
- Optimisations mémoire

✅ **Intégration écosystème** :
- LangChain natif support
- LlamaIndex integration
- Supabase by default

✅ **Adoption massive** :
- 10K+ stars GitHub
- Milliers d'applications en production

### Futures Directions

🔮 **pgvector 1.0** (roadmap) :
- Quantization (réduction taille)
- Nouveaux types d'index
- Performances accrues

🔮 **PostgreSQL 19+ (2026)** :
- Intégration native possible ?
- Optimisations au niveau du core

🔮 **IA embarquée** :
- Génération d'embeddings directement dans PostgreSQL ?
- Modèles légers en extension ?

---

## Conclusion

### Points Clés à Retenir

✅ **pgvector** = Extension PostgreSQL pour recherche vectorielle

✅ **Embeddings** = Représentation numérique capturant le *sens*

✅ **Recherche vectorielle** = Trouver éléments similaires par proximité

✅ **Avantages** :
- Tout dans PostgreSQL (pas d'infrastructure supplémentaire)
- Transactions ACID
- Écosystème PostgreSQL complet
- Simplicité d'utilisation

✅ **Cas d'usage** :
- Recherche sémantique
- RAG (ChatGPT + données privées)
- Recommandation
- Classification
- Détection de doublons

⚠️ **Considérations** :
- Coût des embeddings (API)
- Mémoire pour index
- Performance à très grande échelle

🎯 **pgvector transforme PostgreSQL en moteur de recherche intelligent !**

### Structure des Chapitres Suivants

Les chapitres suivants approfondissent les aspects de pgvector :

**18.6.1. Recherche de similarité (cosine, L2)**
- Les 3 métriques de distance
- Comment choisir
- Exemples pratiques

**18.6.2. Index HNSW et IVFFlat**
- Optimisation des performances
- Configuration des index
- Tuning et maintenance

**18.6.3. Cas d'usage : RAG, Semantic Search**
- Implémentation complète d'un RAG
- Recherche sémantique en production
- Optimisations avancées

### Ressources pour Démarrer

📚 **Documentation** :
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Documentation officielle PostgreSQL](https://www.postgresql.org/docs/)

🎓 **Tutoriels** :
- [Supabase Vector Docs](https://supabase.com/docs/guides/ai)
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)

💬 **Communauté** :
- [Reddit r/PostgreSQL](https://reddit.com/r/PostgreSQL)
- [Discord PostgreSQL](https://discord.gg/postgresql)

🛠️ **Outils** :
- [LangChain](https://python.langchain.com/)
- [LlamaIndex](https://www.llamaindex.ai/)
- [Sentence Transformers](https://www.sbert.net/)

---


⏭️ [Recherche de similarité (cosine, L2)](/18-extensions-et-integrations/06.1-recherche-similarite.md)
