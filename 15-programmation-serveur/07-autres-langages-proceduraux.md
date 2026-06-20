🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.7. Autres langages procéduraux

## Introduction

Dans les sections précédentes, nous avons découvert **PL/pgSQL**, le langage procédural natif et principal de PostgreSQL. PL/pgSQL est puissant, performant et parfaitement intégré à PostgreSQL. Cependant, PostgreSQL adopte une philosophie d'ouverture et de flexibilité en permettant l'utilisation de **plusieurs langages procéduraux** pour écrire des fonctions et procédures stockées.

Cette section présente les **langages procéduraux alternatifs** disponibles dans l'écosystème PostgreSQL, leurs forces respectives, et comment choisir le bon langage pour votre cas d'usage.

---

## Pourquoi Plusieurs Langages Procéduraux ?

### 1. Philosophie de PostgreSQL : L'Ouverture

PostgreSQL a été conçu dès le départ avec une architecture extensible. L'un des principes fondateurs est de permettre aux développeurs d'utiliser les outils et langages avec lesquels ils sont à l'aise, plutôt que de les forcer à apprendre un langage propriétaire unique.

**Avantages de cette approche** :
- Réutilisation des compétences existantes
- Accès à des écosystèmes de bibliothèques riches
- Choix du meilleur outil pour chaque tâche
- Innovation et évolution continues

### 2. Complémentarité des Langages

Chaque langage apporte ses forces uniques :

- **PL/pgSQL** : Intégration native SQL, performance optimale  
- **PL/Python** : Écosystème riche (IA, data science, APIs modernes)  
- **PL/Perl** : Expressions régulières puissantes, manipulation de texte  
- **PL/v8** : JavaScript moderne, manipulation JSON native

Cette diversité permet de **choisir le meilleur outil** selon le contexte plutôt que d'être limité à un seul langage.

### 3. Cas d'Usage Spécifiques

Certaines tâches sont beaucoup plus faciles dans un langage que dans un autre :

| Tâche | Langage optimal | Raison |
|-------|----------------|--------|
| Requêtes SQL complexes | PL/pgSQL | Intégration native |
| Machine Learning | PL/Python | scikit-learn, TensorFlow |
| Parsing de logs | PL/Perl | Regex ultra-puissantes |
| Transformation JSON | PL/v8 | JavaScript ❤️ JSON |
| Appels d'API REST | PL/Python | Bibliothèque requests |
| Extraction de données textuelles | PL/Perl | Traitement texte |

---

## Vue d'Ensemble des Langages Disponibles

### PL/pgSQL (Natif)

**Le langage par défaut de PostgreSQL**

```sql
CREATE OR REPLACE FUNCTION exemple_plpgsql(montant NUMERIC)  
RETURNS NUMERIC AS $$  
BEGIN  
    RETURN montant * 1.20;  -- TVA 20%
END;
$$ LANGUAGE plpgsql;
```

**Caractéristiques** :
- ✅ **Natif** : Inclus par défaut dans PostgreSQL  
- ✅ **Performance** : Optimisé pour les opérations SQL  
- ✅ **Intégration** : Accès direct à toutes les fonctionnalités PostgreSQL  
- ✅ **Stabilité** : Éprouvé et mature  
- ✅ **Transactions** : Gestion native des transactions  
- ⚠️ **Syntaxe** : Spécifique à PostgreSQL (courbe d'apprentissage)  
- ⚠️ **Bibliothèques** : Limitées aux fonctionnalités PostgreSQL

**Utilisé pour** :
- Logique métier SQL pure
- Triggers et fonctions système
- Performance critique
- Opérations transactionnelles complexes

---

### PL/Python (plpython3u)

**Python 3 dans PostgreSQL**

```sql
CREATE OR REPLACE FUNCTION exemple_python(texte TEXT)  
RETURNS TEXT AS $$  
    import json
    data = {"message": texte.upper()}
    return json.dumps(data)
$$ LANGUAGE plpython3u;
```

**Caractéristiques** :
- ✅ **Écosystème** : Accès à pip/PyPI (NumPy, pandas, requests, etc.)  
- ✅ **Populaire** : Langage très utilisé (data science, IA, web)  
- ✅ **Lisible** : Syntaxe claire et accessible  
- ✅ **Moderne** : Écosystème actif et en évolution  
- ✅ **Polyvalent** : Data science, APIs, transformations  
- ⚠️ **Performance** : Plus lent que PL/pgSQL pour SQL pur  
- ⚠️ **Overhead** : Interpréteur Python par session  
- ⚠️ **Sécurité** : plpython3u = untrusted (accès système complet)

**Utilisé pour** :
- Machine Learning / Intelligence Artificielle
- Traitement de données complexe
- Appels d'API REST
- Intégration avec des services externes
- Calculs scientifiques et statistiques

---

### PL/Perl (plperlu)

**Perl dans PostgreSQL**

```sql
CREATE OR REPLACE FUNCTION exemple_perl(texte TEXT)  
RETURNS TEXT AS $$  
    my $texte = shift;
    $texte =~ s/(\d{4})-(\d{2})-(\d{2})/$3\/$2\/$1/g;
    return $texte;
$$ LANGUAGE plperlu;
```

**Caractéristiques** :
- ✅ **Regex** : Les meilleures expressions régulières  
- ✅ **Texte** : Manipulation de texte exceptionnelle  
- ✅ **CPAN** : Accès à des milliers de modules Perl  
- ✅ **Mature** : Langage éprouvé depuis 1987  
- ✅ **Parsing** : Excellent pour logs, formats propriétaires  
- ⚠️ **Popularité** : Moins utilisé aujourd'hui  
- ⚠️ **Syntaxe** : Considérée comme complexe  
- ⚠️ **Communauté** : Plus petite qu'avant

**Utilisé pour** :
- Parsing de logs et fichiers texte
- Expressions régulières complexes
- Transformation de formats propriétaires
- Extraction de données non structurées
- Migration de code Perl existant

---

### PL/v8 (JavaScript)

**JavaScript (moteur V8 de Google) dans PostgreSQL**

```sql
CREATE OR REPLACE FUNCTION exemple_v8(data JSONB)  
RETURNS JSONB AS $$  
    // data est passé comme objet JavaScript (déserialisé depuis JSONB)
    if (!Array.isArray(data)) {
        plv8.elog(ERROR, 'data doit être un tableau');
    }
    return data.map(item => ({
        ...item,
        prix_ttc: item.prix * 1.20
    }));
$$ LANGUAGE plv8 IMMUTABLE STRICT;
```

**Caractéristiques** :
- ✅ **Moderne** : JavaScript ES6+ (const, let, arrow functions, classes)  
- ✅ **JSON** : Manipulation JSON native et triviale  
- ✅ **Performance** : Moteur V8 ultra-optimisé avec JIT  
- ✅ **Populaire** : Langage très utilisé  
- ✅ **Stack unifiée** : Même langage frontend/backend/database  
- ⚠️ **Extension tierce** : Pas inclus par défaut (mais largement disponible chez les hébergeurs cloud — Supabase, Neon, etc.)  
- ⚠️ **npm** : Pas d'accès natif aux modules npm (il faut « bundler » manuellement le code)  
- ⚠️ **Disponibilité** : Vérifier auprès de l'hébergeur

**Utilisé pour** :
- Manipulation JSON/JSONB intensive
- APIs REST internes
- Logique métier moderne
- Validation de données
- Stack JavaScript complète

---

### Autres langages procéduraux disponibles

L'écosystème PostgreSQL ne se limite pas à ces quatre langages. Voici d'autres options qui peuvent être pertinentes selon les besoins :

| Langage | Module | Type | Cas d'usage typique |
|---------|--------|------|---------------------|
| **PL/Tcl** | `pltcl`, `pltclu` | Trusted/Untrusted | Scripts simples, intégration historique |
| **PL/R** | `plr` (tierce) | Untrusted | Analyse statistique, graphiques R, modèles statistiques |
| **PL/Java** | `pljava` (tierce) | Trusted | Réutilisation de code Java existant, écosystème JVM |
| **PL/Lua** | `pllua` (tierce) | Trusted/Untrusted | Scripts légers, performances embarquées |
| **PL/sh** | `plsh` (tierce) | Untrusted | Exécution de scripts shell (avec précaution) |
| **PL/Rust** | `pgrx` / `plrust` (tierce) | Trusted | Performance native, sécurité mémoire, ML |

⭐ **À noter sur PL/Rust** (en pleine ascension depuis 2023) :
- L'extension `plrust` (Trusted Language Extensions) propose une variante sandboxée de Rust.
- L'écosystème `pgrx` (anciennement `pgx`) permet d'écrire des **extensions PostgreSQL complètes en Rust**, pas seulement des fonctions stockées.
- Particulièrement utile pour les fonctions à forte intensité CPU, le ML embarqué, et les types personnalisés.

⚠️ **Disponibilité variable :** la plupart de ces langages sont des extensions tierces qui doivent être compilées et installées séparément. Vérifiez la disponibilité chez votre hébergeur avant de planifier leur utilisation.

---

## Tableau Comparatif Complet

### Performance

| Langage | SQL intensif | Calculs purs | Démarrage | Mémoire |
|---------|-------------|--------------|-----------|---------|
| **PL/pgSQL** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **PL/Python** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **PL/Perl** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **PL/v8** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### Facilité d'utilisation

| Langage | Courbe apprentissage | Lisibilité | Popularité | Communauté |
|---------|---------------------|-----------|-----------|------------|
| **PL/pgSQL** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **PL/Python** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **PL/Perl** | ⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **PL/v8** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### Capacités spécifiques

| Langage | Regex | JSON | Bibliothèques | Intégration SQL |
|---------|-------|------|---------------|-----------------|
| **PL/pgSQL** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐⭐ |
| **PL/Python** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ (pip) | ⭐⭐⭐ |
| **PL/Perl** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ (CPAN) | ⭐⭐⭐ |
| **PL/v8** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ (limité) | ⭐⭐⭐⭐ |

---

## Comment Choisir le Bon Langage ?

### Arbre de Décision

```
┌─────────────────────────────────┐
│  Nouvelle fonction à créer      │
└────────────┬────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ Opérations SQL     │ OUI
    │ intensives ?       ├────────► PL/pgSQL
    └─────────┬──────────┘
              │ NON
              ▼
    ┌────────────────────┐
    │ Manipulation JSON  │ OUI
    │ complexe ?         ├────────► PL/v8 (JavaScript)
    └─────────┬──────────┘
              │ NON
              ▼
    ┌────────────────────┐
    │ Regex complexes ou │ OUI
    │ parsing texte ?    ├────────► PL/Perl
    └─────────┬──────────┘
              │ NON
              ▼
    ┌────────────────────┐
    │ Machine Learning,  │ OUI
    │ Data Science, APIs?├────────► PL/Python
    └─────────┬──────────┘
              │ NON
              ▼
         PL/pgSQL
    (choix par défaut)
```

### Guide de Sélection par Cas d'Usage

#### 🎯 Logique Métier et Transactions

**Choisir : PL/pgSQL**

```sql
-- Gestion de commandes avec transactions
CREATE OR REPLACE FUNCTION passer_commande(...)  
RETURNS INTEGER AS $$  
DECLARE  
    commande_id INTEGER;
BEGIN
    -- Insert commande
    INSERT INTO commandes (...) VALUES (...) RETURNING id INTO commande_id;

    -- Insert lignes
    INSERT INTO lignes_commande (...) VALUES (...);

    -- Update stock
    UPDATE produits SET stock = stock - quantite WHERE ...;

    RETURN commande_id;
    -- Pas de WHEN OTHERS : on laisse remonter les exceptions PostgreSQL
    -- (contrainte FK violée, NOT NULL, etc.) avec leur SQLSTATE original.
    -- Un bloc EXCEPTION WHEN OTHERS perdrait cette information.
END;
$$ LANGUAGE plpgsql;
```

**Raisons** :
- Performance SQL optimale
- Gestion transactionnelle native
- Exceptions PostgreSQL intégrées
- Pas d'overhead externe

---

#### 🤖 Intelligence Artificielle et Machine Learning

**Choisir : PL/Python**

```sql
-- Prédiction avec modèle ML
CREATE OR REPLACE FUNCTION predire_prix(caracteristiques JSONB)  
RETURNS NUMERIC AS $$  
    import pickle
    import numpy as np

    # Charger le modèle pré-entraîné
    with open('/models/prix_model.pkl', 'rb') as f:
        model = pickle.load(f)

    # Préparer les features
    features = np.array([
        caracteristiques['surface'],
        caracteristiques['chambres'],
        caracteristiques['etage']
    ]).reshape(1, -1)

    # Prédiction
    prediction = model.predict(features)[0]

    return float(prediction)
$$ LANGUAGE plpython3u;
```

**Raisons** :
- Accès à scikit-learn, TensorFlow, PyTorch
- Écosystème data science complet
- NumPy pour calculs matriciels
- pandas pour manipulation de données

---

#### 📝 Parsing de Logs et Extraction de Données

**Choisir : PL/Perl**

```sql
-- Parser des logs Apache complexes
CREATE OR REPLACE FUNCTION parser_log_serveur(log_line TEXT)  
RETURNS TABLE(  
    ip TEXT,
    horodatage TIMESTAMP,   -- NB : `timestamp` est un mot-clé SQL, interdit comme nom de colonne
    methode TEXT,
    url TEXT,
    code INTEGER,
    user_agent TEXT
) AS $$
    my $line = shift;

    # Regex Perl ultra-puissante
    if ($line =~ /^(\S+) .* \[([^\]]+)\] "(\w+) ([^"]+) [^"]+" (\d+) .* "([^"]+)"$/) {
        my ($ip, $ts, $method, $url, $code, $ua) = ($1, $2, $3, $4, $5, $6);

        # Conversion timestamp
        # ... (logique Perl)

        return_next({
            ip => $ip,
            horodatage => $ts,
            methode => $method,
            url => $url,
            code => $code,
            user_agent => $ua
        });
    }

    return undef;
$$ LANGUAGE plperlu;
```

**Raisons** :
- Expressions régulières les plus puissantes
- Parsing de texte non structuré
- Manipulation de chaînes avancée
- Modules CPAN pour formats spécifiques

---

#### 🌐 APIs REST et Manipulation JSON

**Choisir : PL/v8 (JavaScript)**

```sql
-- Transformation d'API REST
CREATE OR REPLACE FUNCTION transformer_api_response(response JSONB)  
RETURNS JSONB AS $$  
    // Destructuring et transformation native
    const { users } = response;

    // Map moderne avec arrow functions
    const transformed = users
        .filter(user => user.active)
        .map(user => ({
            id: user.id,
            fullName: `${user.firstName} ${user.lastName}`,
            email: user.email.toLowerCase(),
            registeredAt: new Date(user.createdAt).toISOString(),
            premium: user.plan === 'premium'
        }))
        .sort((a, b) => a.fullName.localeCompare(b.fullName));

    return {
        total: transformed.length,
        users: transformed,
        processedAt: new Date().toISOString()
    };
$$ LANGUAGE plv8 IMMUTABLE STRICT;
```

**Raisons** :
- JSON manipulation native et triviale
- Syntaxe moderne et claire
- Performance V8 excellente
- Cohérence avec frontend/backend JavaScript

---

## Architecture Multi-Langages

### Principe de Complémentarité

Dans un projet réel, il est **courant et recommandé** d'utiliser plusieurs langages procéduraux selon les besoins :

```
┌──────────────────────────────────────────────────┐
│             Application PostgreSQL               │
├──────────────────────────────────────────────────┤
│                                                  │
│  ┌──────────────┐  ┌──────────────┐              │
│  │  PL/pgSQL    │  │  PL/Python   │              │
│  ├──────────────┤  ├──────────────┤              │
│  │ • Triggers   │  │ • ML Models  │              │
│  │ • CRUD       │  │ • Analytics  │              │
│  │ • Business   │  │ • API Calls  │              │
│  └──────────────┘  └──────────────┘              │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐              │
│  │  PL/Perl     │  │  PL/v8       │              │
│  ├──────────────┤  ├──────────────┤              │
│  │ • Log Parse  │  │ • JSON API   │              │
│  │ • Text Proc  │  │ • Validation │              │
│  │ • Regex      │  │ • Transform  │              │
│  └──────────────┘  └──────────────┘              │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Exemple d'Architecture Réelle

**Scénario** : Application e-commerce avec PostgreSQL

```sql
-- 1. PL/pgSQL : Logique transactionnelle
CREATE OR REPLACE FUNCTION valider_commande(commande_id INT)  
RETURNS BOOLEAN AS $$  
    -- Logique SQL pure avec transactions
$$ LANGUAGE plpgsql;

-- 2. PL/Python : Recommandations ML
CREATE OR REPLACE FUNCTION recommander_produits(user_id INT)  
RETURNS TABLE(produit_id INT, score FLOAT) AS $$  
    import sklearn  # Modèle ML de recommandation
    # ...
$$ LANGUAGE plpython3u;

-- 3. PL/Perl : Parsing logs serveur
CREATE OR REPLACE FUNCTION analyser_logs_acces(log_file TEXT)  
RETURNS TABLE(ip TEXT, url TEXT, horodatage TIMESTAMP) AS $$  
    # Regex puissantes pour parsing
$$ LANGUAGE plperlu;

-- 4. PL/v8 : API GraphQL interne
CREATE OR REPLACE FUNCTION graphql_query(query JSONB)  
RETURNS JSONB AS $$  
    // Résolution de requêtes GraphQL
    // Transformation JSON native
$$ LANGUAGE plv8;
```

---

## Considérations Pratiques

### 1. Installation et Disponibilité

| Langage | Module | Statut | Disponibilité typique |
|---------|--------|--------|----------------------|
| **PL/pgSQL** | `plpgsql` | Natif, trusted | ✅ Toujours installé |
| **PL/Python** | `plpython3u` | Officiel, untrusted seulement | ⚠️ Paquet séparé (`postgresql-plpython3-XX`) |
| **PL/Perl** | `plperl` / `plperlu` | Officiel, trusted + untrusted | ⚠️ Paquet séparé (`postgresql-plperl-XX`) |
| **PL/Tcl** | `pltcl` / `pltclu` | Officiel, trusted + untrusted | ⚠️ Paquet séparé (`postgresql-pltcl-XX`) |
| **PL/v8** | `plv8` | Extension tierce | ⚠️ Compilation ou paquet tiers requis |
| **PL/Rust** | `plrust`, `pgrx` | Extensions tierces | ⚠️ Compilation requise |

**Installation sur Debian/Ubuntu** (exemple pour PostgreSQL 18) :

```bash
# Langages officiels (depuis les dépôts PostgreSQL)
sudo apt install postgresql-plperl-18 postgresql-plpython3-18 postgresql-pltcl-18

# Puis dans la base :
CREATE EXTENSION plperl;       -- trusted  
CREATE EXTENSION plperlu;      -- untrusted  
CREATE EXTENSION plpython3u;   -- untrusted (unique variante)  
```

**Recommandation** : Vérifiez toujours la disponibilité des extensions dans votre environnement de production (hébergement cloud, conteneurs Docker, etc.). Certains services managés (AWS RDS, Azure Database for PostgreSQL) restreignent fortement les langages untrusted, voire les interdisent complètement.

### 2. Sécurité : Trusted vs Untrusted

Les langages procéduraux ont, pour certains, deux variantes :

**Trusted (sécurisé)** :
- Sandbox limitant les opérations dangereuses
- Pas d'accès au système de fichiers depuis la fonction
- Pas d'exécution de commandes système
- **Création de fonctions** : tout rôle disposant du privilège `USAGE` sur le langage (par défaut accordé à `PUBLIC`)

**Untrusted (non sécurisé)** :
- Accès complet au système d'exploitation, au système de fichiers et aux commandes externes
- Possibilité de lire/écrire des fichiers, d'exécuter des programmes
- **Création de fonctions** : **rôle superutilisateur uniquement**

```sql
-- Variantes disponibles aujourd'hui
CREATE EXTENSION plpgsql;     -- Trusted, installé par défaut  
CREATE EXTENSION plperl;      -- Trusted  
CREATE EXTENSION plperlu;     -- Untrusted (suffixe « u »)  
CREATE EXTENSION plpython3u;  -- Untrusted (suffixe « u »)  
CREATE EXTENSION plv8;        -- Trusted (extension tierce)  
```

⚠️ **Particularité de Python : pas de version trusted**

PostgreSQL ne fournit **que** la variante `plpython3u` (untrusted). Il n'existe **pas** de `plpython3` trusted. La raison : Python ne dispose pas d'un sandbox robuste, et tenter d'en construire un avait créé des failles de sécurité. Cette spécificité a deux conséquences :
- Toute fonction PL/Python nécessite que le créateur soit superutilisateur.
- Une fois créée par un superutilisateur, la fonction peut être **exécutée** par des rôles normaux (selon les droits `EXECUTE`), mais avec les permissions de l'utilisateur PostgreSQL sur le système hôte (souvent `postgres`).

**Rappel historique :** `plpython2u` (Python 2) ainsi que `plpython` et `plpythonu` (alias) ont été **retirés dans PostgreSQL 12** (octobre 2019), peu avant la fin de vie officielle de Python 2 (janvier 2020). PostgreSQL 12 et toutes les versions suivantes ne fournissent plus que la variante Python 3 (`plpython3u`). Tout code Python doit donc être Python 3 compatible.

**Règle pratique** : si vous avez besoin d'accéder aux bibliothèques système (pip, CPAN, npm), utilisez la version untrusted en sachant que cela élargit considérablement la surface d'attaque. Pour une simple manipulation de texte ou de JSON sans dépendance externe, préférez la version trusted lorsqu'elle existe.

### 3. Performance et Overhead

⚠️ **Les chiffres de performance dépendent fortement du cas d'usage.** Voici des **tendances observées** plutôt que des mesures absolues :

**Pour la logique SQL intensive (jointures, agrégats, parcours de tables) :**
- **PL/pgSQL** est généralement le plus performant : pas de conversion de type entre PostgreSQL et un langage externe, pas de marshalling.
- Les autres langages doivent **convertir** les valeurs PostgreSQL en types natifs (Python, Perl, JS) puis revenir, ce qui ajoute un coût par appel.

**Pour les calculs purs (CPU intensive sans SQL) :**
- **PL/v8** peut être très rapide grâce au JIT du moteur V8.
- **PL/pgSQL** est mauvais pour les calculs numériques bruts (boucles arithmétiques).
- **PL/Python** est moyen, mais peut devenir rapide avec NumPy (vectorisation).

**Coût de démarrage :**
- **PL/pgSQL** : démarrage négligeable (intégré au backend).
- **PL/Python** : chargement de l'interpréteur Python à la première fonction de la session (centaines de ms).
- **PL/Perl** : chargement de l'interpréteur Perl à la première fonction (similaire).
- **PL/v8** : chargement du moteur V8 à la première fonction (similaire).

Une fois chargé, l'interpréteur reste en mémoire pour toute la session : les appels suivants sont rapides.

**Recommandations** :
- Utilisez PL/pgSQL pour les chemins SQL critiques (triggers fréquents, fonctions appelées par milliers).
- Pour les calculs numériques lourds, comparez PL/v8 et PL/Python avec NumPy via un benchmark réel.
- **Mesurez systématiquement** dans votre environnement avant de choisir.

### 4. Maintenance et Évolutivité

| Aspect | PL/pgSQL | PL/Python | PL/Perl | PL/v8 |
|--------|----------|-----------|---------|--------|
| **Tests** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Debugging** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **Recrutement** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Documentation** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## Migration entre Langages

### Peut-on mélanger les langages ?

**Oui, absolument !** Vous pouvez :
- Appeler une fonction PL/Python depuis PL/pgSQL
- Appeler une fonction PL/pgSQL depuis PL/v8
- Combiner plusieurs langages dans une même base

```sql
-- Fonction PL/pgSQL appelant PL/Python
CREATE OR REPLACE FUNCTION analyser_commande(commande_id INT)  
RETURNS JSONB AS $$  
DECLARE  
    data JSONB;
    prediction NUMERIC;
BEGIN
    -- Récupération des données (PL/pgSQL)
    SELECT row_to_json(c.*) INTO data
    FROM commandes c
    WHERE c.id = commande_id;

    -- Appel fonction ML (PL/Python)
    SELECT predire_fraude(data) INTO prediction;

    -- Retour combiné
    RETURN jsonb_build_object(
        'commande', data,
        'score_fraude', prediction
    );
END;
$$ LANGUAGE plpgsql;
```

### Stratégies de Migration

Si vous voulez migrer du code d'un langage à un autre :

1. **Approche progressive**
   - Migrer fonction par fonction
   - Tester en parallèle
   - Comparer les résultats

2. **Wrapper pattern**
   - Créer des wrappers dans le nouveau langage
   - Appeler l'ancienne fonction en interne
   - Migration transparente

3. **Tests de régression**
   - Suite de tests complète
   - Validation des résultats
   - Benchmarking des performances

---

## Bonnes Pratiques Générales

### 1. Principe du Bon Outil

**Ne pas** forcer un langage quand un autre est plus adapté :

❌ **Mauvais** : Regex complexes en PL/pgSQL
```sql
-- Difficile et illisible
CREATE FUNCTION regex_complexe(texte TEXT)...
-- 50 lignes de code PL/pgSQL...
```

✅ **Bon** : Regex en PL/Perl
```sql
-- Simple et naturel
CREATE FUNCTION regex_complexe(texte TEXT)
$$ my $texte = shift; $texte =~ s/.../.../g; $$
LANGUAGE plperlu;
```

### 2. Documentation

Toujours documenter **pourquoi** vous avez choisi un langage particulier :

```sql
CREATE OR REPLACE FUNCTION analyser_sentiment(commentaire TEXT)  
RETURNS NUMERIC AS $$  
    """
    Analyse de sentiment avec modèle NLP pré-entraîné.

    Langage: PL/Python
    Raison: Utilisation de la bibliothèque transformers (Hugging Face)
    Alternative: Impossible en PL/pgSQL (pas de ML)

    Args:
        commentaire: Texte à analyser

    Returns:
        Score de sentiment (-1 à 1)
    """
    from transformers import pipeline
    # ...
$$ LANGUAGE plpython3u;

-- Commentaire PostgreSQL
COMMENT ON FUNCTION analyser_sentiment(TEXT) IS
'Analyse de sentiment avec modèle BERT (nécessite PL/Python pour transformers)';
```

### 3. Isolation des Dépendances

Documentez les dépendances externes requises :

```sql
-- Requirements: pip install requests beautifulsoup4
CREATE OR REPLACE FUNCTION scraper_web(url TEXT)...
$$ LANGUAGE plpython3u;

-- Requirements: cpan install LWP::UserAgent HTML::Parser
CREATE OR REPLACE FUNCTION parser_html(html TEXT)...
$$ LANGUAGE plperlu;
```

### 4. Tests et Validation

Testez particulièrement les conversions de types entre PostgreSQL et le langage :

```sql
-- Test des types
SELECT
    test_fonction_python(
        42::INTEGER,
        'texte',
        '2025-11-22'::DATE,
        ARRAY[1,2,3],
        '{"key": "value"}'::JSONB
    );
```

---

## Résumé : Quand Utiliser Quel Langage ?

### 🎯 Règle Générale

1. **Par défaut** : PL/pgSQL  
2. **Besoin spécifique** : Langage adapté  
3. **Performance critique** : PL/pgSQL ou PL/v8  
4. **Lisibilité/Maintenance** : Langage le plus connu de l'équipe

### 🏆 Champions par Catégorie

| Catégorie | Champion | Runner-up |
|-----------|----------|-----------|
| **Performance SQL** | PL/pgSQL | - |
| **Performance calculs** | PL/v8 | PL/pgSQL |
| **Manipulation JSON** | PL/v8 | PL/Python |
| **Expressions régulières** | PL/Perl | PL/v8 |
| **Machine Learning** | PL/Python | - |
| **APIs REST** | PL/Python | PL/v8 |
| **Parsing texte** | PL/Perl | PL/Python |
| **Facilité apprentissage** | PL/Python / PL/v8 | PL/pgSQL |
| **Écosystème** | PL/Python | PL/Perl |
| **Popularité** | PL/Python / PL/v8 | PL/pgSQL |

### 📊 Matrice de Décision Rapide

```
┌─────────────────────┬──────────┬──────────┬─────────┬────────┐
│ Critère             │ PL/pgSQL │ PL/Python│ PL/Perl │ PL/v8  │
├─────────────────────┼──────────┼──────────┼─────────┼────────┤
│ SQL pur             │    ✓✓✓   │    ✗     │    ✗    │   ✓    │
│ JSON                │    ✓     │    ✓✓    │    ✗    │   ✓✓✓  │
│ Regex               │    ✓     │    ✓     │   ✓✓✓   │   ✓✓   │
│ ML/AI               │    ✗     │    ✓✓✓   │    ✗    │   ✗    │
│ Performance         │    ✓✓✓   │    ✓     │    ✓    │   ✓✓   │
│ Disponibilité       │    ✓✓✓   │    ✓✓    │    ✓✓   │   ✓    │
│ Communauté          │    ✓✓✓   │    ✓✓✓   │    ✓    │   ✓✓   │
└─────────────────────┴──────────┴──────────┴─────────┴────────┘

Légende: ✓✓✓ Excellent | ✓✓ Très bon | ✓ Bon | ✗ Pas adapté
```

---

## Prochaines Sections

Dans les sections suivantes, nous explorerons en détail chacun de ces langages :

- **15.7.1. PL/Python (plpython3u)** : Installation, syntaxe, exemples complets, intégration avec pip/PyPI  
- **15.7.2. PL/Perl (plperlu)** : Expressions régulières avancées, CPAN, parsing de texte  
- **15.7.3. PL/v8 (JavaScript)** : Manipulation JSON, ES6+, moteur V8, stack JavaScript unifiée

Chaque section comprendra :
- Installation et configuration
- Syntaxe et types de données
- Exemples pratiques progressifs
- Cas d'usage avancés
- Bonnes pratiques
- Limitations et considérations

---

## Conclusion

PostgreSQL offre une **flexibilité exceptionnelle** avec son support multi-langages. Au lieu d'imposer un seul langage propriétaire, PostgreSQL permet aux développeurs de choisir l'outil le plus adapté à chaque situation.

**Recommandations finales** :

1. ✅ **Maîtrisez PL/pgSQL** : C'est le fondamental  
2. ✅ **Apprenez au moins un langage alternatif** : Selon votre domaine  
3. ✅ **N'ayez pas peur de mélanger** : Utilisez le meilleur outil pour chaque tâche  
4. ✅ **Documentez vos choix** : Expliquez pourquoi vous avez choisi tel langage  
5. ✅ **Testez et benchmarker** : Validez que votre choix est optimal

PostgreSQL ne vous limite pas : il vous donne le pouvoir de choisir ! 🚀

---

*Cette introduction fait partie de la formation complète "Maîtriser PostgreSQL 18 - De la Théorie à l'Expertise"*

**Sections suivantes** :
- 15.7.1. PL/Python (plpython3u) →
- 15.7.2. PL/Perl (plperlu) →
- 15.7.3. PL/v8 (JavaScript) - mention →

⏭️ [PL/Python (plpython3u)](/15-programmation-serveur/07.1-plpython.md)
