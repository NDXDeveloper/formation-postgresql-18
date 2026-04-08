🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.1. Fonctions SQL et PL/pgSQL : Différences et cas d'usage

## Introduction aux Fonctions dans PostgreSQL

Dans PostgreSQL, une **fonction** est un bloc de code réutilisable qui peut être appelé depuis n'importe quelle requête SQL. Les fonctions permettent d'encapsuler de la logique métier, de simplifier des requêtes complexes et de centraliser du code qui serait autrement répété.

PostgreSQL propose plusieurs langages pour écrire des fonctions, mais les deux principaux sont :
- **SQL** : Un langage déclaratif, simple et direct  
- **PL/pgSQL** : Un langage procédural, plus puissant et flexible

Cette section explore leurs différences fondamentales et leurs cas d'usage respectifs.

---

## 1. Les Fonctions SQL

### 1.1. Qu'est-ce qu'une fonction SQL ?

Une **fonction SQL** est une fonction écrite directement en langage SQL standard. Elle se compose essentiellement d'une ou plusieurs requêtes SQL qui sont exécutées séquentiellement.

### 1.2. Caractéristiques principales

- **Déclarative** : On décrit *ce qu'on veut* obtenir, pas *comment* l'obtenir  
- **Simple** : Pas de structures de contrôle (IF, LOOP, etc.)  
- **Performante** : Le planificateur peut optimiser directement le code SQL  
- **Inline** : PostgreSQL peut parfois "dérouler" la fonction dans la requête appelante (fonction inline)

### 1.3. Syntaxe de base

```sql
CREATE FUNCTION nom_fonction(parametre1 TYPE, parametre2 TYPE, ...)  
RETURNS TYPE_RETOUR  
LANGUAGE SQL  
AS $$  
    -- Requête(s) SQL
    SELECT ...
$$;
```

### 1.4. Exemple simple : Calculer le prix TTC

```sql
CREATE FUNCTION calculer_prix_ttc(prix_ht NUMERIC, taux_tva NUMERIC)  
RETURNS NUMERIC  
LANGUAGE SQL  
AS $$  
    SELECT prix_ht * (1 + taux_tva / 100);
$$;

-- Utilisation
SELECT calculer_prix_ttc(100, 20);  -- Retourne 120.00
```

**Explication** :
- La fonction prend deux paramètres : `prix_ht` et `taux_tva`
- Elle retourne un résultat de type `NUMERIC`
- Le corps de la fonction contient une simple expression SQL
- Aucune logique conditionnelle ou itérative

### 1.5. Exemple avec accès à une table

```sql
CREATE FUNCTION obtenir_nom_client(id_client INTEGER)  
RETURNS TEXT  
LANGUAGE SQL  
AS $$  
    SELECT nom FROM clients WHERE id = id_client;
$$;

-- Utilisation
SELECT obtenir_nom_client(42);
```

### 1.6. Fonctions SQL retournant plusieurs lignes

Une fonction SQL peut retourner un ensemble de lignes (table) plutôt qu'une seule valeur :

```sql
CREATE FUNCTION clients_par_ville(ville_recherchee TEXT)  
RETURNS TABLE(id INTEGER, nom TEXT, email TEXT)  
LANGUAGE SQL  
AS $$  
    SELECT id, nom, email
    FROM clients
    WHERE ville = ville_recherchee;
$$;

-- Utilisation : traiter la fonction comme une table
SELECT * FROM clients_par_ville('Paris');
```

### 1.7. Avantages des fonctions SQL

- ✅ **Simplicité** : Syntaxe minimale, facile à lire et maintenir  
- ✅ **Performance** : Le planificateur peut optimiser directement le SQL  
- ✅ **Inline expansion** : PostgreSQL peut "intégrer" la fonction dans la requête parente  
- ✅ **Sécurité** : Moins de risques d'erreurs logiques complexes

### 1.8. Limites des fonctions SQL

- ❌ **Pas de logique conditionnelle** : Impossible d'utiliser IF, CASE complexes, LOOP, etc.  
- ❌ **Pas de gestion d'erreurs** : Pas de bloc EXCEPTION pour capturer les erreurs  
- ❌ **Pas de variables temporaires** : On ne peut pas stocker d'états intermédiaires  
- ❌ **Une seule instruction de retour** : Limite la complexité des traitements

---

## 2. Les Fonctions PL/pgSQL

### 2.1. Qu'est-ce que PL/pgSQL ?

**PL/pgSQL** (Procedural Language/PostgreSQL) est un langage procédural inspiré du PL/SQL d'Oracle. Il ajoute des structures de programmation impérative au SQL standard.

### 2.2. Caractéristiques principales

- **Procédurale** : On décrit *comment* faire les choses, étape par étape  
- **Structures de contrôle** : IF/ELSIF/ELSE, CASE, LOOP, WHILE, FOR  
- **Variables locales** : Stockage d'états intermédiaires  
- **Gestion d'exceptions** : Capture et traitement des erreurs  
- **Plus puissante** : Pour des logiques métier complexes

### 2.3. Syntaxe de base

```sql
CREATE FUNCTION nom_fonction(parametre1 TYPE, parametre2 TYPE, ...)  
RETURNS TYPE_RETOUR  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    -- Déclaration de variables locales
    variable1 TYPE;
    variable2 TYPE;
BEGIN
    -- Corps de la fonction avec logique procédurale
    -- Instructions SQL et structures de contrôle
    RETURN resultat;
EXCEPTION
    -- Gestion d'erreurs (optionnel)
    WHEN condition THEN
        -- Traitement d'erreur
END;
$$;
```

### 2.4. Exemple simple : Prix TTC avec remise conditionnelle

```sql
CREATE FUNCTION calculer_prix_avec_remise(
    prix_ht NUMERIC,
    taux_tva NUMERIC,
    montant_commande NUMERIC
)
RETURNS NUMERIC  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    prix_ttc NUMERIC;
    remise NUMERIC := 0;
BEGIN
    -- Calcul du prix TTC
    prix_ttc := prix_ht * (1 + taux_tva / 100);

    -- Logique conditionnelle : remise selon le montant
    IF montant_commande > 1000 THEN
        remise := 0.10;  -- 10% de remise
    ELSIF montant_commande > 500 THEN
        remise := 0.05;  -- 5% de remise
    END IF;

    -- Application de la remise
    prix_ttc := prix_ttc * (1 - remise);

    RETURN prix_ttc;
END;
$$;

-- Utilisation
SELECT calculer_prix_avec_remise(100, 20, 600);  -- Retourne 114.00 (avec 5% de remise)
```

**Explication** :
- Déclaration de variables locales (`prix_ttc`, `remise`)
- Utilisation de `IF/ELSIF` pour une logique conditionnelle
- Assignation avec l'opérateur `:=`
- Structure plus complexe qu'une simple fonction SQL

### 2.5. Exemple avec boucle : Calcul itératif

```sql
CREATE FUNCTION calculer_factorielle(n INTEGER)  
RETURNS INTEGER  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    resultat INTEGER := 1;
    i INTEGER;
BEGIN
    -- Validation de l'entrée
    IF n < 0 THEN
        RAISE EXCEPTION 'Le nombre doit être positif';
    END IF;

    -- Boucle de calcul
    FOR i IN 1..n LOOP
        resultat := resultat * i;
    END LOOP;

    RETURN resultat;
END;
$$;

-- Utilisation
SELECT calculer_factorielle(5);  -- Retourne 120
```

### 2.6. Exemple avec gestion d'erreurs

```sql
CREATE FUNCTION diviser_securise(numerateur NUMERIC, denominateur NUMERIC)  
RETURNS NUMERIC  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Tentative de division
    RETURN numerateur / denominateur;
EXCEPTION
    WHEN division_by_zero THEN
        -- Capture l'erreur de division par zéro
        RAISE NOTICE 'Division par zéro détectée, retour de NULL';
        RETURN NULL;
    WHEN OTHERS THEN
        -- Capture toute autre erreur
        RAISE NOTICE 'Erreur inattendue: %', SQLERRM;
        RETURN NULL;
END;
$$;

-- Utilisation
SELECT diviser_securise(10, 0);  -- Retourne NULL avec un message
```

### 2.7. Avantages des fonctions PL/pgSQL

- ✅ **Logique complexe** : Conditions, boucles, gestion d'états  
- ✅ **Gestion d'erreurs robuste** : Blocs EXCEPTION pour capturer et traiter les erreurs  
- ✅ **Variables et états** : Stockage et manipulation d'informations temporaires  
- ✅ **Réutilisabilité** : Encapsulation de logique métier complexe  
- ✅ **Débogage** : Messages avec RAISE NOTICE/DEBUG

### 2.8. Limites des fonctions PL/pgSQL

- ❌ **Performances** : Moins optimisable par le planificateur que le SQL pur  
- ❌ **Complexité** : Plus difficile à lire et maintenir pour des cas simples  
- ❌ **Pas d'inline** : Ne peut pas être "déroulée" dans la requête parente  
- ❌ **Overhead** : Coût d'exécution légèrement plus élevé

---

## 3. Tableau Comparatif : SQL vs PL/pgSQL

| Critère | Fonction SQL | Fonction PL/pgSQL |
|---------|--------------|-------------------|
| **Simplicité** | ⭐⭐⭐⭐⭐ Très simple | ⭐⭐⭐ Plus complexe |
| **Performance** | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐ Bonne |
| **Optimisation** | ✅ Inline possible | ❌ Pas d'inline |
| **Logique conditionnelle** | ❌ Limitée (CASE simple) | ✅ IF/ELSIF/ELSE complet |
| **Boucles** | ❌ Non supportées | ✅ LOOP, WHILE, FOR |
| **Variables** | ❌ Non (sauf WITH) | ✅ DECLARE variables |
| **Gestion d'erreurs** | ❌ Non | ✅ Bloc EXCEPTION |
| **Débogage** | ⭐⭐ Limité | ⭐⭐⭐⭐ RAISE NOTICE |
| **Cas d'usage** | Requêtes simples | Logique métier complexe |

---

## 4. Cas d'Usage : Quand Utiliser SQL ou PL/pgSQL ?

### 4.1. Utilisez une fonction SQL quand :

✅ **La logique est purement déclarative** : Une simple requête SELECT/JOIN/agrégation suffit

**Exemple** : Calculer le total d'une commande
```sql
CREATE FUNCTION total_commande(commande_id INTEGER)  
RETURNS NUMERIC  
LANGUAGE SQL  
AS $$  
    SELECT SUM(quantite * prix_unitaire)
    FROM lignes_commande
    WHERE id_commande = commande_id;
$$;
```

✅ **Pas de logique conditionnelle complexe** : Aucun IF/ELSE ou boucle nécessaire

**Exemple** : Récupérer les clients actifs
```sql
CREATE FUNCTION clients_actifs()  
RETURNS TABLE(id INTEGER, nom TEXT, email TEXT)  
LANGUAGE SQL  
AS $$  
    SELECT id, nom, email
    FROM clients
    WHERE actif = true;
$$;
```

✅ **Performance critique** : Vous voulez que le planificateur optimise au maximum

✅ **Simplicité avant tout** : Fonction appelée fréquemment et qui doit rester simple

---

### 4.2. Utilisez une fonction PL/pgSQL quand :

✅ **Logique métier complexe** : Conditions multiples, états, validations

**Exemple** : Valider et créer une commande
```sql
CREATE FUNCTION creer_commande_validee(
    client_id INTEGER,
    produits INTEGER[],
    quantites INTEGER[]
)
RETURNS INTEGER  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    commande_id INTEGER;
    credit_client NUMERIC;
    montant_total NUMERIC := 0;
    i INTEGER;
BEGIN
    -- Vérification du client
    SELECT credit_disponible INTO credit_client
    FROM clients WHERE id = client_id;

    IF credit_client IS NULL THEN
        RAISE EXCEPTION 'Client inexistant';
    END IF;

    -- Calcul du montant total
    FOR i IN 1..array_length(produits, 1) LOOP
        SELECT montant_total + (prix * quantites[i]) INTO montant_total
        FROM produits WHERE id = produits[i];
    END LOOP;

    -- Vérification du crédit
    IF montant_total > credit_client THEN
        RAISE EXCEPTION 'Crédit insuffisant';
    END IF;

    -- Création de la commande
    INSERT INTO commandes(client_id, montant, date_creation)
    VALUES (client_id, montant_total, NOW())
    RETURNING id INTO commande_id;

    RETURN commande_id;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Erreur lors de la création : %', SQLERRM;
        RETURN NULL;
END;
$$;
```

✅ **Gestion d'erreurs nécessaire** : Besoin de capturer et traiter les exceptions

✅ **Traitements itératifs** : Boucles sur des données, calculs séquentiels

✅ **Variables d'état** : Besoin de stocker des informations temporaires

✅ **Débogage et traçabilité** : Utilisation de RAISE NOTICE pour loguer

---

## 5. Règles de Décision Pratiques

### Diagramme de décision

```
Ai-je besoin de IF/ELSE complexes ?
│
├─ Non ──► Ai-je besoin de boucles (LOOP/FOR) ?
│          │
│          ├─ Non ──► Ai-je besoin de variables temporaires ?
│          │          │
│          │          ├─ Non ──► Ai-je besoin de gérer des erreurs ?
│          │          │          │
│          │          │          ├─ Non ──► ✅ Utilisez SQL
│          │          │          └─ Oui ──► ⚠️ Utilisez PL/pgSQL
│          │          └─ Oui ──► ⚠️ Utilisez PL/pgSQL
│          └─ Oui ──► ⚠️ Utilisez PL/pgSQL
└─ Oui ──► ⚠️ Utilisez PL/pgSQL
```

### Règle d'or

> **"Restez avec SQL aussi longtemps que possible. Passez à PL/pgSQL uniquement quand SQL ne suffit plus."**

---

## 6. Bonnes Pratiques

### 6.1. Privilégier la simplicité

- Commencez toujours par SQL si possible
- N'ajoutez PL/pgSQL que si nécessaire
- Une fonction simple est plus facile à maintenir

### 6.2. Nommage cohérent

```sql
-- ✅ Bon : Nom explicite
CREATE FUNCTION calculer_prix_total_commande(...)

-- ❌ Mauvais : Nom vague
CREATE FUNCTION calc(...)
```

### 6.3. Documentation

```sql
CREATE FUNCTION calculer_remise(montant NUMERIC)  
RETURNS NUMERIC  
LANGUAGE plpgsql  
AS $$  
-- Cette fonction calcule la remise applicable selon le montant
-- Règles :
--   - 10% si montant > 1000
--   - 5% si montant > 500
--   - 0% sinon
BEGIN
    ...
END;
$$;

-- Ajouter un commentaire PostgreSQL
COMMENT ON FUNCTION calculer_remise(NUMERIC) IS
'Calcule la remise applicable en fonction du montant de la commande';
```

### 6.4. Gestion des erreurs explicite

```sql
-- ✅ Bon : Messages d'erreur clairs
RAISE EXCEPTION 'Client % introuvable', client_id;

-- ❌ Mauvais : Message générique
RAISE EXCEPTION 'Erreur';
```

### 6.5. Tester les cas limites

Pensez toujours à tester :
- Valeurs NULL
- Valeurs négatives (si inappropriées)
- Divisions par zéro
- Données manquantes dans les tables

---

## 7. Exemples Comparatifs Concrets

### Exemple 1 : Fonction simple de conversion

**Version SQL** (recommandée pour ce cas) :
```sql
CREATE FUNCTION celsius_vers_fahrenheit(celsius NUMERIC)  
RETURNS NUMERIC  
LANGUAGE SQL  
AS $$  
    SELECT celsius * 9.0 / 5.0 + 32;
$$;
```

**Version PL/pgSQL** (surcharge inutile) :
```sql
CREATE FUNCTION celsius_vers_fahrenheit(celsius NUMERIC)  
RETURNS NUMERIC  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    fahrenheit NUMERIC;
BEGIN
    fahrenheit := celsius * 9.0 / 5.0 + 32;
    RETURN fahrenheit;
END;
$$;
```

**Verdict** : SQL est largement préférable ici. Pas de logique complexe.

---

### Exemple 2 : Fonction avec validation et logique métier

**Version SQL** (impossible ou très complexe) :
```sql
-- Ne peut pas facilement gérer les validations et conditions multiples
```

**Version PL/pgSQL** (adaptée) :
```sql
CREATE FUNCTION appliquer_promotion(
    produit_id INTEGER,
    code_promo TEXT
)
RETURNS NUMERIC  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    prix_actuel NUMERIC;
    reduction NUMERIC := 0;
    stock_disponible INTEGER;
BEGIN
    -- Récupération du prix et stock
    SELECT prix, stock INTO prix_actuel, stock_disponible
    FROM produits WHERE id = produit_id;

    IF prix_actuel IS NULL THEN
        RAISE EXCEPTION 'Produit % inexistant', produit_id;
    END IF;

    -- Vérification du stock
    IF stock_disponible < 1 THEN
        RAISE NOTICE 'Produit en rupture de stock';
        RETURN NULL;
    END IF;

    -- Application du code promo
    IF code_promo = 'SOLDES2025' THEN
        reduction := 0.20;
    ELSIF code_promo = 'BIENVENUE' THEN
        reduction := 0.10;
    ELSIF code_promo IS NOT NULL THEN
        RAISE NOTICE 'Code promo invalide : %', code_promo;
    END IF;

    RETURN prix_actuel * (1 - reduction);
END;
$$;
```

**Verdict** : PL/pgSQL est indispensable pour cette logique métier complexe.

---

## 8. Résumé des Concepts Clés

### Fonctions SQL
- ✅ Simples et performantes  
- ✅ Optimisées par le planificateur  
- ✅ Pas de variables ni logique procédurale  
- 🎯 **Idéal pour** : requêtes déclaratives, calculs simples, encapsulation de SELECT

### Fonctions PL/pgSQL
- ✅ Puissantes et flexibles  
- ✅ Structures de contrôle complètes  
- ✅ Gestion d'erreurs robuste  
- 🎯 **Idéal pour** : logique métier, validations, traitements complexes

### Principe directeur
> Choisissez la **simplicité** (SQL) quand c'est possible, la **puissance** (PL/pgSQL) quand c'est nécessaire.

---

## 9. Points de Vigilance pour les Débutants

### ⚠️ Erreur courante n°1 : Utiliser PL/pgSQL par défaut
```sql
-- ❌ Mauvais : PL/pgSQL pour une simple requête
CREATE FUNCTION get_user(uid INTEGER)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    username TEXT;
BEGIN
    SELECT name INTO username FROM users WHERE id = uid;
    RETURN username;
END;
$$;

-- ✅ Bon : SQL suffit largement
CREATE FUNCTION get_user(uid INTEGER)  
RETURNS TEXT  
LANGUAGE SQL  
AS $$  
    SELECT name FROM users WHERE id = uid;
$$;
```

### ⚠️ Erreur courante n°2 : Oublier le RETURN
```sql
-- ❌ Mauvais : Pas de RETURN en PL/pgSQL
CREATE FUNCTION test() RETURNS INTEGER LANGUAGE plpgsql AS $$  
BEGIN  
    -- Oubli du RETURN !
END;
$$;
-- Erreur : control reached end of function without RETURN
```

### ⚠️ Erreur courante n°3 : Confondre SQL et PL/pgSQL dans l'assignation
```sql
-- En SQL : pas d'assignation, uniquement SELECT
CREATE FUNCTION f() RETURNS INTEGER LANGUAGE SQL AS $$
    SELECT 42;  -- ✅ Correct
$$;

-- En PL/pgSQL : assignation avec :=
CREATE FUNCTION f() RETURNS INTEGER LANGUAGE plpgsql AS $$  
DECLARE  
    x INTEGER;
BEGIN
    x := 42;  -- ✅ Correct
    x = 42;   -- ❌ Erreur : = est pour la comparaison !
    RETURN x;
END;
$$;
```

---

## 10. Conclusion

Les fonctions SQL et PL/pgSQL sont deux outils complémentaires dans votre arsenal PostgreSQL :

- **SQL** : Votre choix par défaut pour les traitements simples et performants  
- **PL/pgSQL** : Votre solution pour les logiques métier complexes nécessitant des structures de contrôle

La maîtrise des deux vous permet de choisir l'outil adapté à chaque situation, en optimisant à la fois la lisibilité du code, la maintenabilité et les performances.

---

**🎓 Prochaines étapes dans le tutoriel :**
- 15.2. Catégories de volatilité (VOLATILE, STABLE, IMMUTABLE)
- 15.3. Procédures stockées et gestion transactionnelle
- 15.4. Triggers et automatisation

**💡 Astuce :** Avant de passer à la section suivante, prenez le temps de réfléchir aux fonctions que vous pourriez créer dans vos propres projets. Identifiez lesquelles seraient mieux en SQL et lesquelles nécessiteraient PL/pgSQL.

⏭️ [Catégories de volatilité : VOLATILE, STABLE, IMMUTABLE](/15-programmation-serveur/02-categories-de-volatilite.md)
