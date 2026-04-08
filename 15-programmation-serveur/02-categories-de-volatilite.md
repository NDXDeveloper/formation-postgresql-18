🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.2. Catégories de Volatilité : VOLATILE, STABLE, IMMUTABLE

## Introduction : Pourquoi la Volatilité est-elle Importante ?

Lorsque vous créez une fonction dans PostgreSQL, le système a besoin de savoir **comment cette fonction se comporte** pour optimiser son exécution. Plus précisément, PostgreSQL doit répondre à cette question :

> **"Est-ce que cette fonction retourne toujours le même résultat quand on l'appelle avec les mêmes paramètres ?"**

La réponse à cette question détermine la **catégorie de volatilité** de votre fonction, qui influence directement :
- **Les performances** : PostgreSQL peut-il mettre en cache le résultat ?  
- **L'optimisation** : PostgreSQL peut-il simplifier la requête ?  
- **La sécurité** : La fonction peut-elle être utilisée dans certains contextes ?

PostgreSQL propose trois catégories de volatilité :
1. **IMMUTABLE** - "Immuable" : Toujours le même résultat  
2. **STABLE** - "Stable" : Même résultat pendant une transaction  
3. **VOLATILE** - "Volatile" : Résultat peut changer à chaque appel

---

## 1. IMMUTABLE : Les Fonctions Immuables

### 1.1. Définition

Une fonction **IMMUTABLE** (immuable) est une fonction qui :
- ✅ Retourne **toujours exactement le même résultat** pour les mêmes arguments  
- ✅ Ne dépend **que** de ses paramètres d'entrée  
- ✅ N'accède **jamais** à la base de données  
- ✅ Ne produit **aucun effet de bord** (pas de modification de données)

### 1.2. Analogie simple

Pensez à une **fonction mathématique pure** :
```
f(x) = x * 2
```
- Si vous appelez `f(5)`, vous obtenez toujours `10`
- Peu importe quand, où, ou combien de fois vous l'appelez
- Le résultat ne dépend que de l'entrée

C'est exactement le comportement d'une fonction IMMUTABLE.

### 1.3. Exemples de fonctions IMMUTABLE

#### Exemple 1 : Calcul mathématique simple
```sql
CREATE FUNCTION carre(x INTEGER)  
RETURNS INTEGER  
LANGUAGE SQL  
IMMUTABLE  -- On déclare explicitement la volatilité  
AS $$  
    SELECT x * x;
$$;

-- Appels :
SELECT carre(5);     -- Retourne toujours 25  
SELECT carre(5);     -- Encore 25  
SELECT carre(5);     -- Toujours 25  
```

#### Exemple 2 : Conversion de température
```sql
CREATE FUNCTION celsius_vers_kelvin(celsius NUMERIC)  
RETURNS NUMERIC  
LANGUAGE SQL  
IMMUTABLE  
AS $$  
    SELECT celsius + 273.15;
$$;

-- La conversion est toujours la même
SELECT celsius_vers_kelvin(0);    -- Toujours 273.15  
SELECT celsius_vers_kelvin(100);  -- Toujours 373.15  
```

#### Exemple 3 : Calcul de prix avec formule fixe
```sql
CREATE FUNCTION calculer_prix_ttc(prix_ht NUMERIC)  
RETURNS NUMERIC  
LANGUAGE SQL  
IMMUTABLE  
AS $$  
    SELECT prix_ht * 1.20;  -- TVA fixe de 20%
$$;
```

### 1.4. Avantages de IMMUTABLE

✅ **Performance maximale** : PostgreSQL met en cache le résultat
```sql
-- PostgreSQL peut optimiser ceci :
SELECT carre(5), carre(5), carre(5)  
FROM une_grande_table;  

-- En calculant carre(5) une seule fois !
```

✅ **Utilisable dans les index**
```sql
-- On peut créer un index sur une fonction IMMUTABLE
CREATE INDEX idx_temperature_kelvin  
ON mesures(celsius_vers_kelvin(temperature));  

-- PostgreSQL peut utiliser cet index !
```

✅ **Optimisation de requête**
```sql
-- PostgreSQL peut simplifier la requête avant l'exécution
SELECT * FROM produits WHERE prix < calculer_prix_ttc(100);
-- → Transformé en : WHERE prix < 120.00
```

### 1.5. Règles strictes pour IMMUTABLE

❌ **Interdit** : Accès à la base de données
```sql
-- ❌ FAUX : Cette fonction ne devrait PAS être IMMUTABLE
CREATE FUNCTION obtenir_nom_client(id INTEGER)  
RETURNS TEXT  
LANGUAGE SQL  
IMMUTABLE  -- ERREUR ! Accède à une table  
AS $$  
    SELECT nom FROM clients WHERE id = id;
$$;
-- Si les données de la table changent, le résultat change !
```

❌ **Interdit** : Dépendance à l'état externe
```sql
-- ❌ FAUX : NOW() retourne un temps différent à chaque appel
CREATE FUNCTION ajouter_heure(h INTEGER)  
RETURNS TIMESTAMP  
LANGUAGE SQL  
IMMUTABLE  -- ERREUR ! Dépend de NOW()  
AS $$  
    SELECT NOW() + (h || ' hours')::INTERVAL;
$$;
```

❌ **Interdit** : Génération de valeurs aléatoires
```sql
-- ❌ FAUX : random() produit des valeurs différentes
CREATE FUNCTION nombre_aleatoire_multiple(x INTEGER)  
RETURNS NUMERIC  
LANGUAGE SQL  
IMMUTABLE  -- ERREUR ! Utilise random()  
AS $$  
    SELECT x * random();
$$;
```

### 1.6. Quand utiliser IMMUTABLE

- ✅ Fonctions mathématiques pures (addition, multiplication, racine carrée...)  
- ✅ Conversions d'unités (km → miles, °C → °F)  
- ✅ Calculs algorithmiques déterministes (hash, checksum)  
- ✅ Transformations de chaînes pures (upper, lower sur constantes)

---

## 2. STABLE : Les Fonctions Stables

### 2.1. Définition

Une fonction **STABLE** (stable) est une fonction qui :
- ✅ Retourne le **même résultat** pour les mêmes arguments **dans une même transaction**  
- ✅ Peut accéder à la base de données (lecture seule)  
- ✅ Ne modifie **jamais** les données  
- ⚠️ Le résultat peut changer entre différentes transactions

### 2.2. Analogie simple

Pensez à **regarder l'heure sur une horloge** :
- Pendant que vous regardez, l'heure ne change pas
- Mais si vous revenez demain, l'heure sera différente

Une fonction STABLE est comme cette horloge : stable pendant une "photo" (transaction), mais peut changer entre les photos.

### 2.3. Exemples de fonctions STABLE

#### Exemple 1 : Lecture de configuration
```sql
CREATE FUNCTION obtenir_taux_tva()  
RETURNS NUMERIC  
LANGUAGE SQL  
STABLE  
AS $$  
    SELECT valeur::NUMERIC
    FROM configuration
    WHERE cle = 'taux_tva';
$$;

-- Dans une même transaction, retourne toujours la même valeur
BEGIN;
    SELECT obtenir_taux_tva();  -- 20.0
    SELECT obtenir_taux_tva();  -- 20.0 (même résultat)
COMMIT;

-- Mais peut changer dans une autre transaction
-- (si quelqu'un met à jour la configuration)
```

#### Exemple 2 : Récupération de données
```sql
CREATE FUNCTION nom_complet_client(client_id INTEGER)  
RETURNS TEXT  
LANGUAGE SQL  
STABLE  
AS $$  
    SELECT prenom || ' ' || nom
    FROM clients
    WHERE id = client_id;
$$;

-- Pendant une transaction, le nom ne change pas
-- Même si appelé plusieurs fois
```

#### Exemple 3 : Fonction utilisant NOW()
```sql
CREATE FUNCTION est_recent(date_creation TIMESTAMP)  
RETURNS BOOLEAN  
LANGUAGE SQL  
STABLE  -- STABLE car utilise NOW()  
AS $$  
    SELECT date_creation > NOW() - INTERVAL '24 hours';
$$;

-- NOW() est STABLE : même valeur dans toute la transaction
BEGIN;
    SELECT NOW();                    -- 2025-01-15 14:30:00
    SELECT est_recent('2025-01-15'); -- Utilise le même NOW()
    SELECT NOW();                    -- Encore 2025-01-15 14:30:00
COMMIT;
```

### 2.4. Avantages de STABLE

✅ **Optimisation intra-transaction**
```sql
-- PostgreSQL peut optimiser ceci :
SELECT nom_complet_client(1), nom_complet_client(1), nom_complet_client(1)  
FROM commandes;  

-- En calculant nom_complet_client(1) une seule fois par ligne !
```

✅ **Lecture de données sécurisée**
```sql
-- Peut lire des tables sans risque d'incohérence
CREATE FUNCTION calculer_total_commande(commande_id INTEGER)  
RETURNS NUMERIC  
LANGUAGE SQL  
STABLE  
AS $$  
    SELECT SUM(quantite * prix_unitaire)
    FROM lignes_commande
    WHERE id_commande = commande_id;
$$;
```

✅ **Utilisable dans certaines optimisations**
```sql
-- PostgreSQL peut simplifier partiellement
WHERE prix > obtenir_taux_tva() * 100
-- Le taux sera récupéré une fois au début
```

### 2.5. Différence clé avec IMMUTABLE

| Aspect | IMMUTABLE | STABLE |
|--------|-----------|--------|
| Accès base de données | ❌ Non | ✅ Oui (lecture) |
| Utilise NOW() | ❌ Non | ✅ Oui |
| Résultat entre transactions | Toujours identique | Peut changer |
| Utilisable dans index | ✅ Oui | ❌ Non |
| Cache entre transactions | ✅ Oui | ❌ Non |

### 2.6. Quand utiliser STABLE

- ✅ Fonctions qui lisent des tables (SELECT uniquement)  
- ✅ Fonctions utilisant NOW(), CURRENT_DATE, CURRENT_USER  
- ✅ Fonctions de recherche/lookup dans des tables de référence  
- ✅ Calculs basés sur des données en base (tarifs, configuration)

---

## 3. VOLATILE : Les Fonctions Volatiles

### 3.1. Définition

Une fonction **VOLATILE** (volatile) est une fonction qui :
- ⚠️ Peut retourner un **résultat différent** à chaque appel, même avec les mêmes arguments  
- ⚠️ Peut **modifier** les données (INSERT, UPDATE, DELETE)  
- ⚠️ Peut avoir des **effets de bord** visibles  
- ⚠️ Aucune garantie de stabilité

### 3.2. Analogie simple

Pensez à **lancer un dé** :
- Même si vous le lancez toujours de la même façon
- Le résultat change à chaque fois
- Imprévisible et non-déterministe

Une fonction VOLATILE est comme ce dé : on ne sait jamais à l'avance ce qu'elle va retourner.

### 3.3. Exemples de fonctions VOLATILE

#### Exemple 1 : Génération de valeurs aléatoires
```sql
CREATE FUNCTION generer_code_promo()  
RETURNS TEXT  
LANGUAGE SQL  
VOLATILE  -- Par défaut si non spécifié  
AS $$  
    SELECT 'PROMO' || floor(random() * 10000)::TEXT;
$$;

-- Chaque appel produit un résultat différent
SELECT generer_code_promo();  -- 'PROMO7423'  
SELECT generer_code_promo();  -- 'PROMO1892'  
SELECT generer_code_promo();  -- 'PROMO5631'  
```

#### Exemple 2 : Modification de données
```sql
CREATE FUNCTION incrementer_compteur(compteur_nom TEXT)  
RETURNS INTEGER  
LANGUAGE plpgsql  
VOLATILE  -- Modifie des données !  
AS $$  
DECLARE  
    nouvelle_valeur INTEGER;
BEGIN
    UPDATE compteurs
    SET valeur = valeur + 1
    WHERE nom = compteur_nom
    RETURNING valeur INTO nouvelle_valeur;

    RETURN nouvelle_valeur;
END;
$$;

-- Chaque appel modifie la base et retourne une valeur différente
SELECT incrementer_compteur('visiteurs');  -- 1001  
SELECT incrementer_compteur('visiteurs');  -- 1002  
SELECT incrementer_compteur('visiteurs');  -- 1003  
```

#### Exemple 3 : Génération de séquence
```sql
CREATE FUNCTION prochain_numero_facture()  
RETURNS INTEGER  
LANGUAGE SQL  
VOLATILE  
AS $$  
    SELECT nextval('seq_factures');
$$;

-- nextval() change l'état de la séquence
SELECT prochain_numero_facture();  -- 1  
SELECT prochain_numero_facture();  -- 2  
SELECT prochain_numero_facture();  -- 3  
```

#### Exemple 4 : Appel à une API externe (simulation)
```sql
CREATE FUNCTION obtenir_taux_change_actuel(devise TEXT)  
RETURNS NUMERIC  
LANGUAGE plpgsql  
VOLATILE  -- Résultat dépend d'un système externe  
AS $$  
BEGIN  
    -- Simule un appel API qui change constamment
    -- (dans la réalité, utiliserait une extension comme http)
    RETURN random() * 0.1 + 1.15;  -- Taux fluctuant
END;
$$;
```

### 3.4. Caractéristiques de VOLATILE

❌ **Aucune optimisation**
```sql
-- PostgreSQL NE peut PAS optimiser ceci :
SELECT generer_code_promo(), generer_code_promo()  
FROM commandes;  

-- Chaque appel sera réellement exécuté (aucun cache)
```

❌ **Pas d'utilisation dans les index**
```sql
-- ❌ IMPOSSIBLE de créer un index
CREATE INDEX idx_code ON commandes(generer_code_promo());
-- Erreur : functions in index expression must be marked IMMUTABLE
```

⚠️ **Comportement par défaut**
```sql
-- Si vous ne spécifiez rien, la fonction est VOLATILE par défaut
CREATE FUNCTION ma_fonction()  
RETURNS INTEGER  
LANGUAGE SQL  
AS $$  
    SELECT 1;
$$;
-- Cette fonction est VOLATILE (même si elle pourrait être IMMUTABLE)
```

### 3.5. Cas d'usage de VOLATILE

- ✅ Génération de nombres aléatoires (random(), uuid_generate_v4())  
- ✅ Modification de données (INSERT, UPDATE, DELETE)  
- ✅ Lecture avec effets de bord (nextval(), lastval())  
- ✅ Interaction avec des systèmes externes  
- ✅ Fonctions non-déterministes par nature

---

## 4. Tableau Comparatif Complet

| Caractéristique | IMMUTABLE | STABLE | VOLATILE |
|----------------|-----------|--------|----------|
| **Même résultat avec mêmes arguments** | ✅ Toujours | ✅ Dans une transaction | ❌ Non garanti |
| **Accès base de données (SELECT)** | ❌ Non | ✅ Oui | ✅ Oui |
| **Modification de données** | ❌ Non | ❌ Non | ✅ Oui |
| **Utilise NOW()** | ❌ Non | ✅ Oui | ✅ Oui |
| **Utilise random()** | ❌ Non | ❌ Non | ✅ Oui |
| **Utilisable dans index** | ✅ Oui | ❌ Non | ❌ Non |
| **Mise en cache du résultat** | ✅ Maximum | ⚠️ Par transaction | ❌ Aucune |
| **Optimisation par le planificateur** | ✅ Maximum | ⚠️ Partielle | ❌ Minimale |
| **Comportement par défaut** | Non | Non | ✅ Oui (si non spécifié) |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |

---

## 5. Impact sur les Performances

### 5.1. Exemple de Performance : IMMUTABLE vs VOLATILE

Comparons l'exécution de la même logique avec différentes volatilités :

```sql
-- Version IMMUTABLE
CREATE FUNCTION calcul_immutable(x INTEGER)  
RETURNS INTEGER  
LANGUAGE SQL  
IMMUTABLE  
AS $$  
    SELECT x * 2;
$$;

-- Version VOLATILE (par défaut)
CREATE FUNCTION calcul_volatile(x INTEGER)  
RETURNS INTEGER  
LANGUAGE SQL  
VOLATILE  
AS $$  
    SELECT x * 2;
$$;

-- Test sur une grande table
EXPLAIN ANALYZE  
SELECT calcul_immutable(5) FROM generate_series(1, 1000000);  
-- Résultat : ~50ms (fonction appelée 1 seule fois, résultat mis en cache)

EXPLAIN ANALYZE  
SELECT calcul_volatile(5) FROM generate_series(1, 1000000);  
-- Résultat : ~250ms (fonction appelée 1 million de fois !)
```

**Différence de performance : 5× plus rapide avec IMMUTABLE !**

### 5.2. Visualisation de l'optimisation

```
┌─────────────────────────────────────────────────────────┐
│ Requête : SELECT ma_fonction(5) FROM table (1M lignes)  │
└─────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │                               │
    IMMUTABLE                       VOLATILE
        │                               │
        ▼                               ▼
  ┌──────────┐                    ┌─────────────┐
  │ Calcul   │                    │ Calcul 1    │
  │ 1 fois   │                    │ Calcul 2    │
  │          │                    │ Calcul 3    │
  │ Cache →  │                    │ ...         │
  │ Réutilise│                    │ Calcul 1M   │
  └──────────┘                    └─────────────┘
   Résultat                        Résultat
   instantané                      1M calculs
```

---

## 6. Règles de Décision : Quelle Volatilité Choisir ?

### 6.1. Diagramme de décision

```
Ma fonction modifie-t-elle des données (INSERT/UPDATE/DELETE) ?
│
├─ Oui ──► 🔴 VOLATILE
│
└─ Non ──► Ma fonction lit-elle des données de tables ?
           │
           ├─ Non ──► Ma fonction dépend-elle de l'heure (NOW()) ?
           │          │
           │          ├─ Non ──► 🟢 IMMUTABLE
           │          └─ Oui ──► 🟡 STABLE
           │
           └─ Oui ──► Ma fonction utilise-t-elle random() ou nextval() ?
                      │
                      ├─ Oui ──► 🔴 VOLATILE
                      └─ Non ──► 🟡 STABLE
```

### 6.2. Questions à se poser

**Question 1** : Avec les mêmes arguments, ma fonction retourne-t-elle TOUJOURS exactement le même résultat, peu importe quand ?
- ✅ Oui → Candidat pour IMMUTABLE  
- ❌ Non → Continuer

**Question 2** : Ma fonction accède-t-elle à des données en base ?
- ✅ Oui → Au minimum STABLE  
- ❌ Non → Continuer

**Question 3** : Ma fonction modifie-t-elle des données ou a-t-elle des effets de bord ?
- ✅ Oui → VOLATILE obligatoire  
- ❌ Non → Continuer

**Question 4** : Ma fonction utilise-t-elle NOW(), CURRENT_DATE, CURRENT_USER ?
- ✅ Oui → STABLE (ces fonctions sont STABLE)  
- ❌ Non → IMMUTABLE probable

**Question 5** : Ma fonction utilise-t-elle random(), nextval(), currval() ?
- ✅ Oui → VOLATILE obligatoire  
- ❌ Non → Vérifier si IMMUTABLE ou STABLE

---

## 7. Exemples Pratiques Commentés

### Exemple 1 : Calcul de remise (IMMUTABLE)

```sql
CREATE FUNCTION calculer_remise_fixe(montant NUMERIC)  
RETURNS NUMERIC  
LANGUAGE SQL  
IMMUTABLE  -- ✅ Formule mathématique pure  
AS $$  
    SELECT CASE
        WHEN montant > 1000 THEN montant * 0.10
        WHEN montant > 500 THEN montant * 0.05
        ELSE 0
    END;
$$;

-- Justification :
-- - Ne lit pas de tables
-- - Formule déterministe
-- - Résultat dépend uniquement des arguments
```

### Exemple 2 : Obtenir le taux de TVA (STABLE)

```sql
CREATE FUNCTION obtenir_taux_tva_actuel()  
RETURNS NUMERIC  
LANGUAGE SQL  
STABLE  -- ✅ Lit une table mais stable dans la transaction  
AS $$  
    SELECT taux FROM parametres_fiscaux WHERE actif = true LIMIT 1;
$$;

-- Justification :
-- - Lit une table (pas IMMUTABLE)
-- - Résultat stable pendant la transaction
-- - Ne modifie rien (pas VOLATILE)
```

### Exemple 3 : Générer un UUID (VOLATILE)

```sql
CREATE FUNCTION generer_id_unique()  
RETURNS UUID  
LANGUAGE SQL  
VOLATILE  -- ✅ Génération aléatoire, jamais pareil  
AS $$  
    SELECT gen_random_uuid();
$$;

-- Justification :
-- - Utilise une fonction aléatoire
-- - Résultat différent à chaque appel
-- - Non-déterministe par nature
```

### Exemple 4 : Vérifier si un utilisateur est actif (STABLE)

```sql
CREATE FUNCTION utilisateur_est_actif(user_id INTEGER)  
RETURNS BOOLEAN  
LANGUAGE SQL  
STABLE  -- ✅ Lecture de table  
AS $$  
    SELECT actif FROM utilisateurs WHERE id = user_id;
$$;

-- Justification :
-- - Lit une table (pas IMMUTABLE)
-- - Résultat constant pendant la transaction
-- - Aucune modification
```

### Exemple 5 : Enregistrer une visite (VOLATILE)

```sql
CREATE FUNCTION enregistrer_visite(page TEXT)  
RETURNS VOID  
LANGUAGE plpgsql  
VOLATILE  -- ✅ Modifie des données  
AS $$  
BEGIN  
    INSERT INTO logs_visites(page, timestamp)
    VALUES (page, NOW());
END;
$$;

-- Justification :
-- - Effectue un INSERT (modification)
-- - A un effet de bord visible
-- - Change l'état de la base
```

---

## 8. Erreurs Courantes et Pièges

### ⚠️ Erreur #1 : Déclarer IMMUTABLE une fonction qui accède à des tables

```sql
-- ❌ MAUVAIS : DANGER !
CREATE FUNCTION prix_produit(produit_id INTEGER)  
RETURNS NUMERIC  
LANGUAGE SQL  
IMMUTABLE  -- ❌ FAUX ! Lit une table !  
AS $$  
    SELECT prix FROM produits WHERE id = produit_id;
$$;

-- Problème : Si le prix change dans la table, PostgreSQL
-- peut retourner l'ancienne valeur depuis le cache !

-- ✅ CORRECT :
CREATE FUNCTION prix_produit(produit_id INTEGER)  
RETURNS NUMERIC  
LANGUAGE SQL  
STABLE  -- ✅ Lit une table = STABLE  
AS $$  
    SELECT prix FROM produits WHERE id = produit_id;
$$;
```

### ⚠️ Erreur #2 : Oublier de spécifier la volatilité

```sql
-- ⚠️ PAS OPTIMAL : Par défaut = VOLATILE
CREATE FUNCTION doubler(x INTEGER)  
RETURNS INTEGER  
LANGUAGE SQL  
AS $$  
    SELECT x * 2;
$$;
-- Cette fonction est VOLATILE alors qu'elle pourrait être IMMUTABLE !
-- Impact : Performances dégradées

-- ✅ CORRECT : Spécifier explicitement
CREATE FUNCTION doubler(x INTEGER)  
RETURNS INTEGER  
LANGUAGE SQL  
IMMUTABLE  -- Déclaration explicite  
AS $$  
    SELECT x * 2;
$$;
```

### ⚠️ Erreur #3 : STABLE pour une fonction avec random()

```sql
-- ❌ MAUVAIS : Comportement imprévisible
CREATE FUNCTION nombre_aleatoire_entre(min INT, max INT)  
RETURNS INTEGER  
LANGUAGE SQL  
STABLE  -- ❌ FAUX ! random() n'est pas stable !  
AS $$  
    SELECT floor(random() * (max - min + 1) + min)::INTEGER;
$$;

-- ✅ CORRECT :
CREATE FUNCTION nombre_aleatoire_entre(min INT, max INT)  
RETURNS INTEGER  
LANGUAGE SQL  
VOLATILE  -- random() est VOLATILE  
AS $$  
    SELECT floor(random() * (max - min + 1) + min)::INTEGER;
$$;
```

### ⚠️ Erreur #4 : Mélanger IMMUTABLE et NOW()

```sql
-- ❌ MAUVAIS : Contradiction !
CREATE FUNCTION est_date_future(date_test DATE)  
RETURNS BOOLEAN  
LANGUAGE SQL  
IMMUTABLE  -- ❌ FAUX ! NOW() n'est pas immuable !  
AS $$  
    SELECT date_test > CURRENT_DATE;
$$;

-- ✅ CORRECT :
CREATE FUNCTION est_date_future(date_test DATE)  
RETURNS BOOLEAN  
LANGUAGE SQL  
STABLE  -- NOW() et CURRENT_DATE sont STABLE  
AS $$  
    SELECT date_test > CURRENT_DATE;
$$;
```

---

## 9. Vérifier la Volatilité d'une Fonction

### 9.1. Consulter la volatilité dans le catalogue système

```sql
-- Voir la volatilité de toutes vos fonctions
SELECT
    proname AS nom_fonction,
    CASE provolatile
        WHEN 'i' THEN 'IMMUTABLE'
        WHEN 's' THEN 'STABLE'
        WHEN 'v' THEN 'VOLATILE'
    END AS volatilite
FROM pg_proc  
WHERE pronamespace = 'public'::regnamespace  
ORDER BY proname;  
```

### 9.2. Consulter une fonction spécifique

```sql
\df+ ma_fonction  -- Dans psql, affiche tous les détails

-- Ou en SQL :
SELECT
    pg_get_functiondef('ma_fonction'::regproc);
```

---

## 10. Bonnes Pratiques

### ✅ Pratique #1 : Toujours déclarer explicitement la volatilité

```sql
-- ✅ BON : Clair et explicite
CREATE FUNCTION ma_fonction(x INTEGER)  
RETURNS INTEGER  
LANGUAGE SQL  
IMMUTABLE  -- Déclaration claire  
AS $$ ... $$;  

-- ❌ ÉVITER : Implicite = confusion
CREATE FUNCTION ma_fonction(x INTEGER)  
RETURNS INTEGER  
LANGUAGE SQL  
AS $$ ... $$;  -- Quelle est la volatilité ? On ne sait pas au premier coup d'œil  
```

### ✅ Pratique #2 : Privilégier IMMUTABLE quand c'est possible

```sql
-- Les fonctions IMMUTABLE offrent les meilleures performances
-- Essayez toujours de concevoir vos fonctions pour qu'elles soient IMMUTABLE
```

### ✅ Pratique #3 : Documenter les raisons de la volatilité

```sql
CREATE FUNCTION calculer_montant(...)  
RETURNS NUMERIC  
LANGUAGE plpgsql  
STABLE  -- STABLE car lit la table tarifs qui peut changer entre transactions  
AS $$  
BEGIN  
    -- Récupère le tarif actuel depuis la base
    ...
END;
$$;

COMMENT ON FUNCTION calculer_montant IS
'Calcule le montant en appliquant le tarif actuel.
STABLE car dépend de la table tarifs.';
```

### ✅ Pratique #4 : Tester l'impact sur les performances

```sql
-- Créez deux versions et comparez
EXPLAIN ANALYZE SELECT ma_fonction_immutable(x) FROM grande_table;  
EXPLAIN ANALYZE SELECT ma_fonction_volatile(x) FROM grande_table;  

-- Mesurez la différence et choisissez en conséquence
```

### ✅ Pratique #5 : Réévaluer lors des modifications

```sql
-- Si vous modifiez une fonction, revérifiez sa volatilité
-- Une fonction IMMUTABLE qui commence à lire une table
-- doit devenir STABLE
```

---

## 11. Impact sur les Index Fonctionnels

### 11.1. Seules les fonctions IMMUTABLE peuvent être indexées

```sql
-- ✅ POSSIBLE : Fonction IMMUTABLE
CREATE FUNCTION nom_complet_normalise(prenom TEXT, nom TEXT)  
RETURNS TEXT  
LANGUAGE SQL  
IMMUTABLE  
AS $$  
    SELECT upper(trim(prenom || ' ' || nom));
$$;

CREATE INDEX idx_nom_normalise  
ON clients(nom_complet_normalise(prenom, nom));  
-- ✅ Succès !

-- ❌ IMPOSSIBLE : Fonction STABLE
CREATE FUNCTION prix_avec_tva_actuelle(prix NUMERIC)  
RETURNS NUMERIC  
LANGUAGE SQL  
STABLE  -- Lit le taux de TVA depuis une table  
AS $$  
    SELECT prix * (1 + obtenir_taux_tva());
$$;

CREATE INDEX idx_prix_tva  
ON produits(prix_avec_tva_actuelle(prix));  
-- ❌ ERREUR : functions in index must be marked IMMUTABLE
```

### 11.2. Raison : Cohérence de l'index

Un index doit être **déterministe** :
- Si `f(x) = y` aujourd'hui, alors `f(x)` doit toujours égaler `y`
- Sinon, l'index devient incohérent avec les données réelles

---

## 12. Résumé Visuel

```
┌─────────────────────────────────────────────────────────────────┐
│                   CHOISIR LA VOLATILITÉ                         │
├─────────────────────────────────────────────────────────────────┤
│
│  🟢 IMMUTABLE
│  ├─ Résultat : Toujours identique
│  ├─ Accès DB : NON
│  ├─ Modif DB : NON
│  ├─ Cache : Maximum
│  ├─ Index : OUI
│  └─ Perf : ⭐⭐⭐⭐⭐
│
│  🟡 STABLE
│  ├─ Résultat : Identique dans la transaction
│  ├─ Accès DB : OUI (lecture)
│  ├─ Modif DB : NON
│  ├─ Cache : Par transaction
│  ├─ Index : NON
│  └─ Perf : ⭐⭐⭐⭐
│
│  🔴 VOLATILE
│  ├─ Résultat : Peut changer à chaque appel
│  ├─ Accès DB : OUI (tout)
│  ├─ Modif DB : OUI
│  ├─ Cache : Aucun
│  ├─ Index : NON
│  └─ Perf : ⭐⭐⭐
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13. Cas Pratiques et Exercices de Réflexion

### Exercice Mental 1 : Identifier la bonne volatilité

Pour chacune des fonctions suivantes, réfléchissez à la volatilité appropriée :

1. Une fonction qui calcule l'âge à partir d'une date de naissance  
2. Une fonction qui retourne le nombre de commandes d'un client  
3. Une fonction qui génère un mot de passe aléatoire  
4. Une fonction qui convertit des euros en dollars avec un taux fixe de 1.10  
5. Une fonction qui incrémente un compteur de visites

**Réponses** :
1. STABLE (utilise CURRENT_DATE)  
2. STABLE (lit une table)  
3. VOLATILE (aléatoire)  
4. IMMUTABLE (calcul pur avec constante)  
5. VOLATILE (modifie des données)

---

## 14. Points Clés à Retenir

### 🎯 Règle d'Or

> **"Quand vous hésitez entre deux catégories, choisissez toujours la plus restrictive (VOLATILE), puis optimisez vers STABLE ou IMMUTABLE une fois que vous êtes certain du comportement."**

### 📋 Checklist de décision

- [ ] Ma fonction ne dépend que de ses arguments → IMMUTABLE ?  
- [ ] Ma fonction lit des données mais ne les modifie pas → STABLE ?  
- [ ] Ma fonction modifie des données ou a des effets de bord → VOLATILE  
- [ ] Ma fonction utilise random(), nextval() → VOLATILE  
- [ ] Ma fonction utilise NOW(), CURRENT_DATE → STABLE  
- [ ] J'ai déclaré explicitement la volatilité → ✅

### ⚠️ Erreurs à éviter

1. ❌ Déclarer IMMUTABLE une fonction qui lit des tables  
2. ❌ Ne pas spécifier de volatilité (défaut = VOLATILE)  
3. ❌ Ignorer l'impact sur les performances  
4. ❌ Oublier que NOW() est STABLE, pas IMMUTABLE

---

## 15. Conclusion

La **volatilité** est un concept fondamental dans PostgreSQL qui influence directement :
- Les **performances** de vos requêtes
- Les **possibilités d'optimisation** offertes au planificateur
- La **création d'index** sur des expressions

En comprenant et en appliquant correctement les catégories de volatilité, vous permettez à PostgreSQL de :
- ✅ Mettre en cache les résultats quand c'est sûr  
- ✅ Créer des index fonctionnels efficaces  
- ✅ Optimiser vos requêtes de manière agressive

**Principes directeurs** :
1. **IMMUTABLE** pour les fonctions pures (mathématiques, conversions)  
2. **STABLE** pour les fonctions de lecture (lookup, configuration)  
3. **VOLATILE** pour les fonctions avec effets de bord (modification, aléatoire)

**Toujours déclarer explicitement la volatilité** pour éviter les surprises et optimiser les performances !

---

**🎓 Prochaines étapes dans le tutoriel :**
- 15.3. Procédures stockées et gestion transactionnelle (CALL, COMMIT/ROLLBACK)
- 15.4. Triggers (BEFORE, AFTER, INSTEAD OF)
- 15.5. Event Triggers : Surveillance DDL

**💡 Pour aller plus loin :**
- Documentation officielle PostgreSQL : [Function Volatility Categories](https://www.postgresql.org/docs/current/xfunc-volatility.html)
- Testez différentes volatilités sur vos propres fonctions et mesurez l'impact avec EXPLAIN ANALYZE

⏭️ [Procédures stockées et gestion transactionnelle (CALL, COMMIT/ROLLBACK)](/15-programmation-serveur/03-procedures-stockees.md)
