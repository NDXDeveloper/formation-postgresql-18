🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.3. Full-Text Search Avancé : Introduction

## Introduction

Imaginez que vous gérez un blog avec 100 000 articles, un site e-commerce avec 1 million de produits, ou une application de gestion documentaire avec des millions de fichiers. Vos utilisateurs veulent pouvoir **rechercher efficacement** dans tout ce contenu. Comment faire ?

La solution classique avec `LIKE` ou `ILIKE` ne suffit pas :

```sql
-- Recherche simple avec LIKE
SELECT titre, contenu FROM articles  
WHERE contenu LIKE '%base de données%';  

-- Problèmes :
-- ❌ Très lent (scan de toute la table)
-- ❌ Pas de gestion du pluriel ("base" vs "bases")
-- ❌ Pas de pertinence (tous les résultats sont égaux)
-- ❌ Pas de recherche intelligente (stemming, synonymes)
-- ❌ Impossible d'indexer efficacement
```

C'est là qu'intervient le **Full-Text Search** (FTS) de PostgreSQL, un système de recherche textuelle puissant et performant, intégré nativement dans la base de données.

**Avec le Full-Text Search** :
```sql
-- Recherche intelligente et rapide
SELECT titre, ts_rank(search_vector, query) AS score  
FROM articles,  
     plainto_tsquery('french', 'base données') query
WHERE search_vector @@ query  
ORDER BY score DESC  
LIMIT 20;  

-- Avantages :
-- ✅ Ultra rapide (index GIN, millisecondes)
-- ✅ Gestion du pluriel automatique ("bases" → "base")
-- ✅ Tri par pertinence (score)
-- ✅ Stemming linguistique (20+ langues)
-- ✅ Stop words (ignore "le", "la", "de"...)
-- ✅ Opérateurs avancés (AND, OR, NOT, phrases)
```

---

## 1. Qu'est-ce que le Full-Text Search ?

### Définition

Le **Full-Text Search** (recherche plein texte) est une technique permettant de rechercher efficacement des mots ou des phrases dans de grandes quantités de texte.

À la différence d'une recherche de chaîne simple (`LIKE`), le Full-Text Search :

1. **Analyse** le texte (tokenization, normalisation)  
2. **Indexe** intelligemment le contenu (index inversé)  
3. **Recherche** rapidement avec des opérateurs avancés  
4. **Classe** les résultats par pertinence (ranking)

### Le Problème que ça Résout

#### Scénario : Site de Documentation Technique

Vous avez 50 000 pages de documentation. Un utilisateur cherche "connection timeout error".

**Sans Full-Text Search** :
```sql
-- Recherche naïve avec LIKE
SELECT titre, contenu FROM documentation  
WHERE contenu LIKE '%connection%'  
  AND contenu LIKE '%timeout%'
  AND contenu LIKE '%error%';

-- Problèmes :
-- ⏱️ Temps : 45 secondes (scan complet)
-- 📊 Résultats : 5000 pages (trop de bruit)
-- 🎯 Pertinence : Aucun tri (ordre aléatoire)
-- 🔤 Variantes : Rate "connections", "errors", "timed out"
```

**Avec Full-Text Search** :
```sql
-- Recherche intelligente
SELECT
    titre,
    ts_rank(search_vector, query) AS score,
    ts_headline('english', contenu, query) AS extrait
FROM documentation,
     websearch_to_tsquery('english', 'connection timeout error') query
WHERE search_vector @@ query  
ORDER BY score DESC  
LIMIT 20;  

-- Avantages :
-- ⏱️ Temps : 12 millisecondes (index GIN)
-- 📊 Résultats : 150 pages pertinentes
-- 🎯 Pertinence : Triées par score (les plus pertinentes en premier)
-- 🔤 Variantes : Trouve "connections", "errors", "timed out" (stemming)
-- 💡 Highlight : Extraits avec mots recherchés en évidence
```

**Gain : 3750× plus rapide + résultats pertinents !**

---

## 2. Full-Text Search vs Alternatives

### Comparaison avec les Solutions Classiques

| Critère | LIKE/ILIKE | SIMILAR TO | Regex | Full-Text Search |
|---------|------------|------------|-------|------------------|
| **Performance** | ❌ Lent | ❌ Très lent | ❌ Très lent | ✅ Rapide (index) |
| **Indexable** | ⚠️ Partiel | ❌ Non | ❌ Non | ✅ Oui (GIN) |
| **Stemming** | ❌ Non | ❌ Non | ❌ Non | ✅ Oui (20+ langues) |
| **Stop words** | ❌ Non | ❌ Non | ❌ Non | ✅ Oui |
| **Ranking** | ❌ Non | ❌ Non | ❌ Non | ✅ Oui (score) |
| **Langues** | ❌ Non | ❌ Non | ❌ Non | ✅ 20+ configs |
| **Opérateurs** | ⚠️ % _ | ⚠️ Basique | ✅ Avancé | ✅ Avancé |
| **Phrases** | ⚠️ Basique | ⚠️ Basique | ✅ Oui | ✅ Oui |
| **Highlights** | ❌ Non | ❌ Non | ❌ Non | ✅ Oui |
| **Complexité** | Simple | Moyenne | Complexe | Moyenne |

**Verdict** : Full-Text Search est supérieur pour la recherche textuelle sérieuse.

### Comparaison avec Solutions Externes

| Critère | PostgreSQL FTS | Elasticsearch | Solr | Algolia |
|---------|----------------|---------------|------|---------|
| **Installation** | ✅ Intégré | ⚠️ Séparé | ⚠️ Séparé | ☁️ SaaS |
| **Coût** | 💰 Gratuit | 💰 Gratuit | 💰 Gratuit | 💰💰💰 Payant |
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scalabilité** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Simplicité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **Cohérence** | ✅ ACID | ⚠️ Eventual | ⚠️ Eventual | ⚠️ Eventual |
| **Transactions** | ✅ Oui | ❌ Non | ❌ Non | ❌ Non |
| **Backup** | ✅ Intégré | ⚠️ Séparé | ⚠️ Séparé | ☁️ Géré |
| **Maintenance** | ✅ Une seule DB | ⚠️ Deux systèmes | ⚠️ Deux systèmes | ☁️ Géré |
| **Synchronisation** | ✅ Aucune | ⚠️ Manuelle | ⚠️ Manuelle | ⚠️ Manuelle |

**Recommandations** :

**Utilisez PostgreSQL FTS pour** :
- ✅ Petites à moyennes applications (< 10M documents)  
- ✅ Besoin de cohérence transactionnelle  
- ✅ Simplicité architecturale (pas d'infrastructure supplémentaire)  
- ✅ Budget limité  
- ✅ Équipe petite/moyenne

**Envisagez Elasticsearch/Solr pour** :
- 📈 Très grandes applications (> 100M documents)  
- 🌐 Recherche distribuée multi-serveurs  
- 📊 Analytics et agrégations complexes  
- 🔥 Recherche temps réel ultra-performante  
- 💼 Équipe dédiée à la recherche

**Vérité** : Pour 80% des applications, PostgreSQL FTS suffit largement et évite la complexité d'un système externe.

---

## 3. Concepts Fondamentaux

### 3.1. Le Pipeline de Traitement

Le Full-Text Search suit un pipeline en plusieurs étapes :

```
1. TEXTE ORIGINAL
   "Les bases de données PostgreSQL sont puissantes"

   ↓

2. PARSER (Découpage)
   ["Les", "bases", "de", "données", "PostgreSQL", "sont", "puissantes"]

   ↓

3. DICTIONNAIRES (Normalisation)
   - Stop words : Supprime "Les", "de", "sont" (mots vides)
   - Stemming : "bases" → "base", "données" → "donn", "puissantes" → "puiss"

   ↓

4. TSVECTOR (Représentation indexable)
   'base':2 'donn':4 'postgresql':5 'puiss':7
   (lexèmes + positions)

   ↓

5. INDEX GIN (Structure d'index inversé)
   "base" → [doc1, doc2, doc5, doc8]
   "donn" → [doc1, doc2, doc9]
   "postgresql" → [doc1, doc3, doc7, doc10]

   ↓

6. RECHERCHE RAPIDE
   Requête : "base données"
   → Lookup dans l'index : [doc1, doc2] (intersection)
   → Résultat en millisecondes
```

### 3.2. Les Composants Clés

#### tsvector : Le Document Indexé

**tsvector** est une représentation optimisée d'un document pour la recherche.

```sql
SELECT to_tsvector('french', 'Les bases de données sont essentielles');
-- Résultat : 'base':2 'donn':4 'essenti':6
```

**Caractéristiques** :
- Lexèmes normalisés (stemming appliqué)
- Positions conservées (pour recherche de phrases)
- Stop words supprimés
- Triés alphabétiquement
- Compressés pour l'efficacité

#### tsquery : La Requête de Recherche

**tsquery** représente une requête avec opérateurs logiques.

```sql
SELECT plainto_tsquery('french', 'base de données');
-- Résultat : 'base' & 'donn'
```

**Opérateurs** :
- `&` : AND (tous les termes)  
- `|` : OR (au moins un terme)  
- `!` : NOT (exclure)  
- `<->` : Suivi de (mots adjacents)  
- `<N>` : Proximité (N mots d'écart max)

#### L'Opérateur @@

L'opérateur `@@` teste si un tsvector correspond à une tsquery.

```sql
SELECT to_tsvector('french', 'PostgreSQL est puissant') @@
       plainto_tsquery('french', 'postgresql');
-- Résultat : true
```

### 3.3. Stemming et Normalisation

Le **stemming** réduit les mots à leur racine linguistique :

```
Français :
"chanter"    → "chant"
"chantons"   → "chant"
"chantaient" → "chant"
"chantera"   → "chant"

"base"       → "base"
"bases"      → "base"

"données"    → "donn"
"donnée"     → "donn"

Anglais :
"running"    → "run"
"runs"       → "run"
"ran"        → "run"

"databases"  → "databas"
"database"   → "databas"
```

**Avantage** : Recherche de "chaussure" trouve aussi "chaussures", "chausser", etc.

### 3.4. Stop Words (Mots Vides)

Les **stop words** sont des mots très fréquents et peu significatifs qui sont ignorés.

**Exemples français** : le, la, les, un, une, de, du, des, à, au, en, et, ou, mais  
**Exemples anglais** : the, a, an, in, on, at, to, for, of, with  

```sql
SELECT to_tsvector('french', 'Le chat et le chien');
-- Résultat : 'chat':2 'chien':5
-- "Le" et "et" ignorés (stop words)
```

---

## 4. Architecture du Full-Text Search dans PostgreSQL

### Intégration Native

Le Full-Text Search est **totalement intégré** dans PostgreSQL :

```
┌────────────────────────────────────────────┐
│         PostgreSQL Core                    │
│                                            │
│  ┌────────────────────────────────────┐    │
│  │   Full-Text Search Subsystem       │    │
│  │                                    │    │
│  │  • Types: tsvector, tsquery        │    │
│  │  • Fonctions: to_tsvector, @@      │    │
│  │  • Opérateurs: &&, ||, <->         │    │
│  │  • Index: GIN (inversé)            │    │
│  │  • Configurations: 20+ langues     │    │
│  │  • Dictionnaires: stemming, stop   │    │
│  │                                    │    │
│  └────────────────────────────────────┘    │
│                                            │
│  Transaction ACID ✓                        │
│  Backup/Restore ✓                          │
│  Réplication ✓                             │
│                                            │
└────────────────────────────────────────────┘
```

### Avantages de l'Intégration

1. **Cohérence transactionnelle**
   - Les données textuelles et l'index sont mis à jour **ensemble**
   - ACID garanti (pas de désynchronisation)

2. **Simplicité opérationnelle**
   - Une seule base de données à gérer
   - Un seul backup
   - Une seule réplication

3. **Performances**
   - Pas de latence réseau (tout local)
   - Pas de sérialisation/désérialisation
   - Optimisations du planificateur

4. **Sécurité**
   - Mêmes contrôles d'accès (GRANT/REVOKE)
   - Mêmes utilisateurs et rôles
   - Chiffrement unifié

---

## 5. Capacités du Full-Text Search PostgreSQL

### 5.1. Support Multilingue

PostgreSQL supporte **20+ langues** avec stemming et stop words :

```sql
-- Français
SELECT to_tsvector('french', 'Les coureurs courent rapidement');
-- Résultat : 'cour':2,3 'rapid':4

-- Anglais
SELECT to_tsvector('english', 'The runners are running quickly');
-- Résultat : 'quick':5 'run':2,4

-- Espagnol
SELECT to_tsvector('spanish', 'Los corredores corren rápidamente');
-- Résultat : 'corr':2,3 'rapid':4
```

**Langues supportées** :
- Européennes : français, anglais, espagnol, allemand, italien, portugais, russe, etc.
- Nordiques : danois, finnois, norvégien, suédois
- Autres : arabe, hongrois, roumain, turc

### 5.2. Opérateurs Riches

```sql
-- AND : Tous les termes
'postgresql & base & données'

-- OR : Au moins un terme
'postgresql | mysql | mariadb'

-- NOT : Exclusion
'base & données & !oracle'

-- Phrase exacte (mots adjacents)
'base <-> données'

-- Proximité (max 2 mots d'écart)
'base <2> données'

-- Préfixe (autocomplétion)
'post:*'  -- Trouve postgresql, postgis, poster, etc.
```

### 5.3. Ranking (Pertinence)

Tri des résultats par **score de pertinence** :

```sql
SELECT
    titre,
    ts_rank(search_vector, query) AS score
FROM articles,
     plainto_tsquery('french', 'postgresql') query
WHERE search_vector @@ query  
ORDER BY score DESC;  
```

**Facteurs de pertinence** :
- Fréquence du terme dans le document
- Position du terme (titre > contenu)
- Rareté du terme (termes rares = plus significatifs)
- Longueur du document (normalisation)

### 5.4. Highlights (Extraits)

Génération d'extraits avec mots recherchés mis en évidence :

```sql
SELECT ts_headline('french',
    'PostgreSQL est un système de gestion de base de données relationnel.',
    plainto_tsquery('french', 'base données')
);

-- Résultat :
-- "PostgreSQL est un système de gestion de <b>base</b> de <b>données</b> relationnel."
```

### 5.5. Pondération

Attribuer des **poids** différents aux parties d'un document :

```sql
-- Titre (A) > Description (B) > Contenu (C)
SELECT
    setweight(to_tsvector('french', titre), 'A') ||
    setweight(to_tsvector('french', description), 'B') ||
    setweight(to_tsvector('french', contenu), 'C')
AS search_vector;
```

### 5.6. Configurations Personnalisées

Créer des dictionnaires et configurations sur mesure :

```sql
-- Dictionnaire de synonymes
CREATE TEXT SEARCH DICTIONARY tech_synonyms (
    TEMPLATE = synonym,
    SYNONYMS = tech_synonyms  -- postgres → postgresql, pgsql → postgresql
);

-- Configuration personnalisée
CREATE TEXT SEARCH CONFIGURATION french_tech (COPY = french);  
ALTER TEXT SEARCH CONFIGURATION french_tech  
    ALTER MAPPING FOR asciiword WITH tech_synonyms, french_stem;
```

---

## 6. Cas d'Usage du Full-Text Search

### 6.1. Blog et Sites de Contenu

**Besoin** : Recherche dans des millions d'articles

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(200),
    contenu TEXT,
    auteur VARCHAR(100),
    search_vector tsvector
);

-- Recherche intelligente
SELECT titre, auteur, ts_rank(search_vector, query) AS score  
FROM articles, plainto_tsquery('french', 'postgresql performance') query  
WHERE search_vector @@ query  
ORDER BY score DESC  
LIMIT 20;  
```

**Fonctionnalités** :
- Recherche rapide (index GIN)
- Tri par pertinence
- Highlights dans les extraits
- Recherche dans titre + contenu avec poids différents

### 6.2. E-commerce

**Besoin** : Recherche de produits par nom, description, caractéristiques

```sql
CREATE TABLE produits (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(200),
    description TEXT,
    marque VARCHAR(100),
    categorie VARCHAR(50),
    search_vector tsvector
);

-- Recherche produits
SELECT nom, marque, prix  
FROM produits  
WHERE search_vector @@ websearch_to_tsquery('french', 'chaussure running nike')  
ORDER BY ts_rank(search_vector, query) DESC;  
```

**Fonctionnalités** :
- Autocomplétion (préfixes)
- Synonymes ("portable" → "téléphone")
- Recherche multi-champs (nom + marque + description)
- Boost du nom produit (poids A)

### 6.3. Documentation Technique

**Besoin** : Recherche dans bases de connaissances

```sql
CREATE TABLE documentation (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(200),
    contenu TEXT,
    categorie VARCHAR(50),
    tags TEXT[],
    search_vector tsvector
);

-- Recherche avec phrases exactes
SELECT titre,
       ts_headline('english', contenu, query) AS extrait
FROM documentation,
     websearch_to_tsquery('english', '"connection timeout" error postgresql') query
WHERE search_vector @@ query;
```

**Fonctionnalités** :
- Phrases exactes (entre guillemets)
- Recherche multilingue
- Highlights dans extraits
- Tags et métadonnées

### 6.4. Systèmes de Tickets/Support

**Besoin** : Recherche dans historique de tickets

```sql
CREATE TABLE tickets (
    id SERIAL PRIMARY KEY,
    sujet VARCHAR(200),
    description TEXT,
    resolution TEXT,
    statut VARCHAR(20),
    search_vector tsvector
);

-- Trouver tickets similaires
SELECT id, sujet, statut  
FROM tickets  
WHERE search_vector @@ plainto_tsquery('french', 'erreur connexion base')  
  AND statut = 'résolu'
ORDER BY ts_rank(search_vector, query) DESC;
```

**Fonctionnalités** :
- Recherche dans historique
- Suggestions de tickets similaires
- Filtres combinés (texte + statut + date)

### 6.5. Réseaux Sociaux

**Besoin** : Recherche dans posts, commentaires

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    contenu TEXT,
    auteur_id INTEGER,
    created_at TIMESTAMP,
    search_vector tsvector
);

-- Recherche temporelle + textuelle
SELECT contenu, auteur_id, created_at  
FROM posts  
WHERE search_vector @@ plainto_tsquery('french', 'postgresql tips')  
  AND created_at > NOW() - INTERVAL '7 days'
ORDER BY ts_rank(search_vector, query) DESC;
```

### 6.6. Recherche Juridique

**Besoin** : Recherche dans corpus de lois, jurisprudence

```sql
CREATE TABLE jurisprudence (
    id SERIAL PRIMARY KEY,
    titre VARCHAR(200),
    contenu TEXT,
    juridiction VARCHAR(100),
    date_decision DATE,
    search_vector tsvector
);

-- Recherche avec opérateurs complexes
SELECT titre, juridiction, date_decision  
FROM jurisprudence  
WHERE search_vector @@ to_tsquery('french', 'responsabilité & civil & !pénal')  
ORDER BY date_decision DESC;  
```

---

## 7. Workflow Typique d'Implémentation

### Étape 1 : Préparation de la Table

```sql
-- Ajouter colonne tsvector
ALTER TABLE ma_table ADD COLUMN search_vector tsvector;
```

### Étape 2 : Remplir la Colonne

```sql
-- Remplir avec poids différents
UPDATE ma_table  
SET search_vector =  
    setweight(to_tsvector('french', COALESCE(titre, '')), 'A') ||
    setweight(to_tsvector('french', COALESCE(contenu, '')), 'B');
```

### Étape 3 : Créer un Trigger de Maintenance

```sql
-- Trigger pour mise à jour automatique
CREATE FUNCTION ma_table_search_trigger() RETURNS trigger AS $$  
BEGIN  
    NEW.search_vector :=
        setweight(to_tsvector('french', COALESCE(NEW.titre, '')), 'A') ||
        setweight(to_tsvector('french', COALESCE(NEW.contenu, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tsvector_update  
BEFORE INSERT OR UPDATE ON ma_table  
FOR EACH ROW EXECUTE FUNCTION ma_table_search_trigger();  
```

### Étape 4 : Créer l'Index GIN

```sql
-- Index pour performance
CREATE INDEX idx_ma_table_search ON ma_table USING GIN(search_vector);  
ANALYZE ma_table;  
```

### Étape 5 : Rechercher

```sql
-- Recherche simple
SELECT * FROM ma_table  
WHERE search_vector @@ plainto_tsquery('french', 'ma recherche');  

-- Recherche avec ranking
SELECT *, ts_rank(search_vector, query) AS score  
FROM ma_table, plainto_tsquery('french', 'ma recherche') query  
WHERE search_vector @@ query  
ORDER BY score DESC  
LIMIT 20;  
```

---

## 8. Performance du Full-Text Search

### Ordres de Grandeur

| Taille de la Table | Sans Index | Avec Index GIN | Gain |
|-------------------|------------|----------------|------|
| 1 000 lignes | 10 ms | 2 ms | 5× |
| 10 000 lignes | 100 ms | 3 ms | 33× |
| 100 000 lignes | 1 000 ms | 5 ms | 200× |
| 1 000 000 lignes | 10 000 ms | 10 ms | 1000× |
| 10 000 000 lignes | 100 000 ms | 20 ms | 5000× |

**Conclusion** : Plus la table est grande, plus l'index GIN est indispensable.

### Facteurs de Performance

**Ce qui accélère** :
- ✅ Index GIN sur colonne tsvector  
- ✅ Colonne tsvector pré-calculée (vs expression)  
- ✅ Index partiel (sous-ensemble de données)  
- ✅ Statistiques à jour (ANALYZE)  
- ✅ Configuration linguistique appropriée

**Ce qui ralentit** :
- ❌ Pas d'index GIN (scan séquentiel)  
- ❌ tsvector calculé à la volée  
- ❌ Statistiques obsolètes  
- ❌ Requêtes trop génériques (retournent 50% de la table)  
- ❌ Trop de dictionnaires dans la configuration

---

## 9. Limitations et Considérations

### Limitations du Full-Text Search PostgreSQL

1. **Scalabilité horizontale limitée**
   - PostgreSQL FTS ne se distribue pas nativement sur plusieurs serveurs
   - Pour > 100M documents, envisager Elasticsearch

2. **Pas de "fuzzy search" natif**
   - Pas de tolérance aux fautes de frappe intégrée
   - Extension pg_trgm peut aider (similitude)

3. **Pas d'agrégations complexes**
   - Pas de facettes/filtres dynamiques avancés comme Elasticsearch
   - Possible mais nécessite des requêtes manuelles

4. **Langue par document**
   - Difficile de gérer plusieurs langues dans un même champ
   - Nécessite des colonnes séparées ou des configurations custom

### Quand NE PAS Utiliser PostgreSQL FTS

- ❌ **Très gros volumes** (> 100M documents) avec besoins de sharding  
- ❌ **Analytics avancés** (facettes, agrégations multidimensionnelles)  
- ❌ **Machine Learning** intégré (recommandations, clustering)  
- ❌ **Recherche floue/typo** critique (correction automatique essentielle)  
- ❌ **Recherche géographique** complexe (combiner avec PostGIS)

---

## 10. Écosystème et Extensions

### Extensions Complémentaires

#### pg_trgm : Similitude et Fuzzy Search

```sql
CREATE EXTENSION pg_trgm;

-- Recherche avec fautes de frappe
SELECT * FROM articles  
WHERE titre % 'postgrasql';  -- Trouve "postgresql"  
```

#### unaccent : Ignorer les Accents

```sql
CREATE EXTENSION unaccent;

-- Créer configuration sans accents
CREATE TEXT SEARCH DICTIONARY french_unaccent (TEMPLATE = unaccent);
```

#### dict_xsyn : Synonymes Étendus

Extension pour gestion avancée des synonymes.

### Outils d'Interface Utilisateur

**Bibliothèques Frontend** :
- **JavaScript** : Typeahead.js, Select2, Choices.js  
- **React** : Downshift, React Select  
- **Vue.js** : Vue-select, Vue-multiselect

**Pattern API** :
```javascript
// Endpoint de recherche
GET /api/search?q=postgresql&limit=20

// Backend PostgreSQL
SELECT titre, description, ts_rank(...) as score  
FROM articles  
WHERE search_vector @@ websearch_to_tsquery('french', :query)  
ORDER BY score DESC  
LIMIT :limit;  
```

---

## 11. Roadmap d'Apprentissage

Pour maîtriser le Full-Text Search PostgreSQL, suivez ce parcours :

### Niveau 1 : Débutant (1-2 semaines)

**Objectifs** :
- Comprendre tsvector et tsquery
- Créer des recherches simples
- Utiliser l'opérateur @@

**À étudier** :
- Section 18.3.1 : tsvector et tsquery
- Création d'index GIN de base

### Niveau 2 : Intermédiaire (2-4 semaines)

**Objectifs** :
- Maîtriser le ranking
- Utiliser les poids (A, B, C, D)
- Optimiser les performances

**À étudier** :
- Section 18.3.2 : Ranking et pondération
- Section 18.3.4 : Index GIN avancés

### Niveau 3 : Avancé (1-2 mois)

**Objectifs** :
- Gérer le multilinguisme
- Créer des configurations personnalisées
- Implémenter en production

**À étudier** :
- Section 18.3.3 : Dictionnaires et langues
- Intégration avec applications réelles
- Monitoring et maintenance

---

## 12. Ressources Complémentaires

### Documentation Officielle

- **PostgreSQL Text Search** : https://www.postgresql.org/docs/current/textsearch.html  
- **Tutorial** : https://www.postgresql.org/docs/current/textsearch-intro.html  
- **Functions** : https://www.postgresql.org/docs/current/functions-textsearch.html

### Articles et Tutoriels

- **Mastering PostgreSQL Full-Text Search** : Blog Compose.io  
- **The Basics of PostgreSQL Text Search** : Thoughtbot  
- **PostgreSQL Full-Text Search Cookbook** : Plusieurs sources

### Livres

- **PostgreSQL: Up and Running** - Regina Obe, Leo Hsu (Chapitre FTS)  
- **The Art of PostgreSQL** - Dimitri Fontaine (Section recherche)

### Communautés

- **PostgreSQL Mailing List** : pgsql-general@postgresql.org  
- **Stack Overflow** : Tag [postgresql] + [full-text-search]  
- **Reddit** : r/PostgreSQL

---

## Conclusion

Le **Full-Text Search** de PostgreSQL est une fonctionnalité puissante et mature qui permet d'implémenter des systèmes de recherche performants sans infrastructure externe.

**Points clés à retenir** :

1. **Intégration native** dans PostgreSQL
   - Cohérence transactionnelle (ACID)
   - Simplicité opérationnelle
   - Pas de synchronisation

2. **Performance excellente**
   - Index GIN (inversé)
   - Gain 100× à 10 000× selon la taille
   - Scalable jusqu'à 10M+ documents

3. **Fonctionnalités riches**
   - 20+ langues avec stemming
   - Opérateurs avancés (AND, OR, NOT, phrases)
   - Ranking (pertinence)
   - Highlights (extraits)
   - Pondération (poids)

4. **Cas d'usage multiples**
   - Blogs, e-commerce, documentation
   - Support, réseaux sociaux, juridique
   - Toute application nécessitant de la recherche textuelle

5. **Alternative sérieuse** à Elasticsearch/Solr
   - Pour 80% des applications
   - Économie d'infrastructure
   - Moins de complexité

Le Full-Text Search transforme PostgreSQL en un moteur de recherche complet, directement au cœur de votre base de données. C'est une compétence essentielle pour tout développeur travaillant avec des données textuelles.

**Prochaine étape** : Plongez dans les détails techniques avec [18.3.1. tsvector et tsquery](./18.3.1-tsvector-tsquery.md) pour comprendre le fonctionnement interne et commencer à construire vos premières recherches !

---

**Prochaines sections** :
- 18.3.1. tsvector et tsquery
- 18.3.2. Ranking et pondération
- 18.3.3. Dictionnaires et langues multiples
- 18.3.4. Index GIN pour la performance

**Ressources** :
- Documentation officielle : https://www.postgresql.org/docs/current/textsearch.html
- Tutorial interactif : https://www.postgresql.org/docs/current/textsearch-intro.html

⏭️ [tsvector et tsquery](/18-extensions-et-integrations/03.1-tsvector-tsquery.md)
