🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.6. Gestion des Exceptions (BEGIN...EXCEPTION...END)

## Introduction : Gérer les Erreurs Élégamment

Lorsque vous écrivez du code PL/pgSQL, des erreurs peuvent survenir :
- Division par zéro
- Violation de contrainte (clé primaire, clé étrangère)
- Donnée introuvable
- Problème de conversion de type
- Et bien d'autres...

Sans gestion d'erreurs, votre fonction ou procédure s'arrête brutalement et toute la transaction est annulée. Avec la **gestion des exceptions**, vous pouvez :
- ✅ Capturer les erreurs  
- ✅ Réagir de manière appropriée  
- ✅ Continuer l'exécution  
- ✅ Fournir des messages d'erreur clairs

### Analogie simple

Pensez à la gestion des exceptions comme un **filet de sécurité** :
- Sans filet : Une erreur fait tout crasher
- Avec filet : L'erreur est attrapée, vous décidez quoi faire ensuite

---

## 1. Structure de Base : BEGIN...EXCEPTION...END

### 1.1. Syntaxe fondamentale

```sql
BEGIN
    -- Code qui pourrait générer une erreur
    -- Instructions SQL normales
EXCEPTION
    WHEN nom_exception THEN
        -- Code à exécuter si cette erreur survient
    WHEN autre_exception THEN
        -- Code pour une autre erreur
    WHEN OTHERS THEN
        -- Code pour toutes les autres erreurs
END;
```

### 1.2. Premier exemple : Division par zéro

Sans gestion d'erreur :
```sql
CREATE OR REPLACE FUNCTION diviser_sans_protection(a NUMERIC, b NUMERIC)  
RETURNS NUMERIC  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RETURN a / b;
END;
$$;

-- Test
SELECT diviser_sans_protection(10, 0);
-- ❌ ERREUR : division by zero
-- La fonction s'arrête brutalement !
```

Avec gestion d'erreur :
```sql
CREATE OR REPLACE FUNCTION diviser_avec_protection(a NUMERIC, b NUMERIC)  
RETURNS NUMERIC  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RETURN a / b;
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Division par zéro détectée ! Retour de NULL.';
        RETURN NULL;
END;
$$;

-- Test
SELECT diviser_avec_protection(10, 0);
-- NOTICE:  Division par zéro détectée ! Retour de NULL.
-- Résultat : NULL (pas d'erreur !)
```

**Ce qui se passe** :
1. Le code dans `BEGIN` s'exécute  
2. L'erreur `division_by_zero` se produit  
3. PostgreSQL saute au bloc `EXCEPTION`  
4. Le code sous `WHEN division_by_zero` s'exécute  
5. La fonction retourne `NULL` au lieu de crasher

---

## 2. Les Types d'Exceptions PostgreSQL

PostgreSQL définit de nombreux types d'exceptions. Voici les plus courantes :

### 2.1. Exceptions de données

| Nom de l'exception | Signification | Exemple |
|-------------------|---------------|---------|
| `division_by_zero` | Division par zéro | `10 / 0` |
| `numeric_value_out_of_range` | Nombre trop grand/petit | `999999999999999999::INTEGER` |
| `invalid_text_representation` | Conversion de texte invalide | `'abc'::INTEGER` |
| `datetime_field_overflow` | Date/heure hors limites | Calcul de date invalide |

### 2.2. Exceptions de contraintes

| Nom de l'exception | Signification | Exemple |
|-------------------|---------------|---------|
| `unique_violation` | Violation de contrainte UNIQUE | Insérer un doublon |
| `foreign_key_violation` | Violation de clé étrangère | Référence inexistante |
| `not_null_violation` | Tentative d'insérer NULL | Colonne NOT NULL |
| `check_violation` | Violation de contrainte CHECK | Valeur hors limites |

### 2.3. Exceptions de requêtes

| Nom de l'exception | Signification | Exemple |
|-------------------|---------------|---------|
| `no_data_found` | Aucune ligne retournée | SELECT INTO sans résultat |
| `too_many_rows` | Trop de lignes retournées | SELECT INTO avec plusieurs résultats |
| `undefined_table` | Table n'existe pas | SELECT sur table inexistante |
| `undefined_column` | Colonne n'existe pas | SELECT colonne_inexistante |

### 2.4. Exception générique

| Nom de l'exception | Signification |
|-------------------|---------------|
| `OTHERS` | Toutes les autres exceptions non spécifiées |

---

## 3. Exemples d'Utilisation par Type d'Erreur

### 3.1. Gérer les conversions de type

```sql
CREATE OR REPLACE FUNCTION convertir_en_entier(texte TEXT)  
RETURNS INTEGER  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    resultat INTEGER;
BEGIN
    -- Tentative de conversion
    resultat := texte::INTEGER;
    RETURN resultat;
EXCEPTION
    WHEN invalid_text_representation THEN
        RAISE NOTICE 'Impossible de convertir "%" en entier', texte;
        RETURN NULL;
END;
$$;

-- Tests
SELECT convertir_en_entier('123');    -- ✅ Retourne 123  
SELECT convertir_en_entier('abc');    -- ✅ Retourne NULL avec message  
SELECT convertir_en_entier('12.5');   -- ✅ Retourne NULL avec message  
```

### 3.2. Gérer les violations de contraintes

```sql
-- Table avec contrainte UNIQUE
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL
);

CREATE OR REPLACE FUNCTION ajouter_utilisateur(email_utilisateur TEXT)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    INSERT INTO utilisateurs (email) VALUES (email_utilisateur);
    RETURN 'Utilisateur ajouté avec succès';
EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Email % déjà utilisé', email_utilisateur;
        RETURN 'Email déjà existant';
    WHEN not_null_violation THEN
        RAISE NOTICE 'Email ne peut pas être NULL';
        RETURN 'Email requis';
END;
$$;

-- Tests
SELECT ajouter_utilisateur('alice@example.com');
-- ✅ "Utilisateur ajouté avec succès"

SELECT ajouter_utilisateur('alice@example.com');
-- NOTICE:  Email alice@example.com déjà utilisé
-- ✅ "Email déjà existant" (pas de crash !)

SELECT ajouter_utilisateur(NULL);
-- NOTICE:  Email ne peut pas être NULL
-- ✅ "Email requis"
```

### 3.3. Gérer les données manquantes

```sql
CREATE OR REPLACE FUNCTION obtenir_nom_client(client_id INTEGER)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    nom_client TEXT;
BEGIN
    -- SELECT INTO génère no_data_found si aucune ligne
    SELECT nom INTO STRICT nom_client
    FROM clients
    WHERE id = client_id;

    RETURN nom_client;
EXCEPTION
    WHEN no_data_found THEN
        RAISE NOTICE 'Aucun client avec ID %', client_id;
        RETURN 'Client introuvable';
    WHEN too_many_rows THEN
        RAISE NOTICE 'Plusieurs clients avec ID % !', client_id;
        RETURN 'Erreur : données incohérentes';
END;
$$;
```

**Note** : Le mot-clé `STRICT` force PostgreSQL à lever une exception si 0 ou plusieurs lignes sont retournées.

---

## 4. Variables Spéciales dans les Blocs EXCEPTION

PostgreSQL fournit des variables spéciales pour obtenir des informations sur l'erreur :

### 4.1. SQLERRM : Message d'erreur

`SQLERRM` contient le message d'erreur complet.

```sql
CREATE OR REPLACE FUNCTION demo_sqlerrm()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Code qui va générer une erreur
    PERFORM 1 / 0;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Message d''erreur : %', SQLERRM;
        -- Affiche : "Message d'erreur : division by zero"
END;
$$;
```

### 4.2. SQLSTATE : Code d'erreur

`SQLSTATE` contient le code d'erreur PostgreSQL (5 caractères).

```sql
CREATE OR REPLACE FUNCTION demo_sqlstate()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    PERFORM 1 / 0;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Code d''erreur : %', SQLSTATE;
        -- Affiche : "Code d'erreur : 22012"
        -- (22012 = code pour division_by_zero)
END;
$$;
```

### 4.3. GET STACKED DIAGNOSTICS : Informations détaillées

Pour obtenir encore plus d'informations :

```sql
CREATE OR REPLACE FUNCTION demo_diagnostics()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    v_sqlstate TEXT;
    v_message TEXT;
    v_context TEXT;
BEGIN
    -- Code qui génère une erreur
    PERFORM 1 / 0;
EXCEPTION
    WHEN OTHERS THEN
        -- Récupérer les informations détaillées
        GET STACKED DIAGNOSTICS
            v_sqlstate = RETURNED_SQLSTATE,
            v_message = MESSAGE_TEXT,
            v_context = PG_EXCEPTION_CONTEXT;

        RAISE NOTICE 'État SQL : %', v_sqlstate;
        RAISE NOTICE 'Message : %', v_message;
        RAISE NOTICE 'Contexte : %', v_context;
END;
$$;

SELECT demo_diagnostics();
-- NOTICE:  État SQL : 22012
-- NOTICE:  Message : division by zero
-- NOTICE:  Contexte : SQL function "demo_diagnostics" statement 1
```

**Variables disponibles** :
- `RETURNED_SQLSTATE` : Code d'erreur  
- `MESSAGE_TEXT` : Message d'erreur  
- `PG_EXCEPTION_CONTEXT` : Contexte d'exécution  
- `PG_EXCEPTION_DETAIL` : Détails supplémentaires  
- `PG_EXCEPTION_HINT` : Conseil pour résoudre l'erreur  
- `COLUMN_NAME` : Nom de la colonne concernée  
- `CONSTRAINT_NAME` : Nom de la contrainte violée  
- `TABLE_NAME` : Nom de la table concernée

---

## 5. Blocs EXCEPTION Multiples

Vous pouvez gérer différentes exceptions de manière spécifique :

```sql
CREATE OR REPLACE FUNCTION operation_complexe(valeur TEXT)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    resultat INTEGER;
BEGIN
    -- Tenter la conversion
    resultat := valeur::INTEGER;

    -- Tenter une division
    resultat := resultat / (resultat - 10);

    RETURN 'Succès : ' || resultat;

EXCEPTION
    WHEN invalid_text_representation THEN
        RETURN 'Erreur : La valeur n''est pas un nombre';

    WHEN division_by_zero THEN
        RETURN 'Erreur : Division par zéro (valeur = 10)';

    WHEN numeric_value_out_of_range THEN
        RETURN 'Erreur : Nombre trop grand';

    WHEN OTHERS THEN
        RETURN 'Erreur inattendue : ' || SQLERRM;
END;
$$;

-- Tests
SELECT operation_complexe('15');    -- ✅ Succès : 3  
SELECT operation_complexe('abc');   -- ✅ "Erreur : La valeur n'est pas un nombre"  
SELECT operation_complexe('10');    -- ✅ "Erreur : Division par zéro (valeur = 10)"  
```

---

## 6. Blocs EXCEPTION Imbriqués

Vous pouvez avoir des blocs `BEGIN...EXCEPTION...END` imbriqués :

```sql
CREATE OR REPLACE FUNCTION traitement_multi_etapes()  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    etape1_ok BOOLEAN := FALSE;
    etape2_ok BOOLEAN := FALSE;
BEGIN
    -- Étape 1 dans son propre bloc
    BEGIN
        -- Code de l'étape 1
        INSERT INTO logs VALUES ('Étape 1');
        etape1_ok := TRUE;
    EXCEPTION
        WHEN OTHERS THEN
            RAISE NOTICE 'Erreur étape 1 : %', SQLERRM;
    END;

    -- Étape 2 dans son propre bloc (s'exécute même si étape 1 échoue)
    BEGIN
        -- Code de l'étape 2
        INSERT INTO logs VALUES ('Étape 2');
        etape2_ok := TRUE;
    EXCEPTION
        WHEN OTHERS THEN
            RAISE NOTICE 'Erreur étape 2 : %', SQLERRM;
    END;

    -- Résumé
    RETURN format('Étape 1: %s, Étape 2: %s',
                  etape1_ok, etape2_ok);
END;
$$;
```

**Avantage** : Chaque étape est isolée. Si une échoue, les autres continuent.

---

## 7. RAISE : Lever des Exceptions

### 7.1. Niveaux de sévérité

```sql
RAISE niveau 'message';
```

**Niveaux disponibles** (du moins au plus grave) :
- `DEBUG` : Information de debug (invisible par défaut)  
- `LOG` : Log dans le fichier de logs  
- `INFO` : Information  
- `NOTICE` : Notification (visible par défaut)  
- `WARNING` : Avertissement  
- `EXCEPTION` : Erreur qui arrête l'exécution

### 7.2. Exemples de RAISE

```sql
CREATE OR REPLACE FUNCTION demo_raise()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RAISE DEBUG 'Message de debug';
    RAISE LOG 'Message de log';
    RAISE INFO 'Message d''information';
    RAISE NOTICE 'Message de notice';
    RAISE WARNING 'Message d''avertissement';

    -- RAISE EXCEPTION arrête l'exécution
    -- RAISE EXCEPTION 'Erreur critique !';
END;
$$;

SELECT demo_raise();
-- INFO:  Message d'information
-- NOTICE:  Message de notice
-- WARNING:  Message d'avertissement
```

### 7.3. RAISE avec variables

```sql
CREATE OR REPLACE FUNCTION valider_age(age INTEGER)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    IF age < 0 THEN
        RAISE EXCEPTION 'Âge invalide : %. L''âge doit être positif.', age;
    END IF;

    IF age < 18 THEN
        RAISE WARNING 'Attention : l''utilisateur est mineur (âge: %)', age;
    END IF;

    RETURN 'Âge valide';
END;
$$;

-- Tests
SELECT valider_age(25);   -- ✅ "Âge valide"  
SELECT valider_age(15);   -- ⚠️ "Âge valide" avec WARNING  
SELECT valider_age(-5);   -- ❌ EXCEPTION levée  
```

### 7.4. RAISE EXCEPTION personnalisée

Vous pouvez spécifier le code SQLSTATE :

```sql
CREATE OR REPLACE FUNCTION verifier_stock(produit_id INTEGER, quantite INTEGER)  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    stock_actuel INTEGER;
BEGIN
    SELECT stock INTO stock_actuel FROM produits WHERE id = produit_id;

    IF stock_actuel < quantite THEN
        RAISE EXCEPTION 'Stock insuffisant pour produit %: demandé %, disponible %'
            USING ERRCODE = 'P0001',  -- Code personnalisé
                  HINT = 'Réduisez la quantité ou attendez un réapprovisionnement';
    END IF;
END;
$$;
```

---

## 8. Cas d'Usage Pratiques

### 8.1. Transaction avec rollback partiel

```sql
CREATE OR REPLACE FUNCTION transfert_argent(
    compte_source INTEGER,
    compte_dest INTEGER,
    montant NUMERIC
)
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    solde_source NUMERIC;
BEGIN
    -- Vérifier le solde
    SELECT solde INTO solde_source FROM comptes WHERE id = compte_source;

    IF solde_source < montant THEN
        RAISE EXCEPTION 'Solde insuffisant : % disponible, % demandé',
                       solde_source, montant;
    END IF;

    -- Débiter le compte source
    UPDATE comptes SET solde = solde - montant WHERE id = compte_source;

    -- Créditer le compte destination
    UPDATE comptes SET solde = solde + montant WHERE id = compte_dest;

    -- Logger la transaction
    INSERT INTO historique_transferts (source, destination, montant, date)
    VALUES (compte_source, compte_dest, montant, NOW());

    RETURN format('Transfert réussi : % transférés du compte % vers %',
                  montant, compte_source, compte_dest);

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Échec du transfert : %', SQLERRM;
        -- La transaction est automatiquement annulée
        RETURN 'Transfert échoué : ' || SQLERRM;
END;
$$;
```

### 8.2. Retry logic (réessayer en cas d'erreur)

```sql
CREATE OR REPLACE FUNCTION inserer_avec_retry(valeur TEXT)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    tentatives INTEGER := 0;
    max_tentatives INTEGER := 3;
    success BOOLEAN := FALSE;
BEGIN
    WHILE tentatives < max_tentatives AND NOT success LOOP
        BEGIN
            tentatives := tentatives + 1;

            -- Tenter l'insertion
            INSERT INTO ma_table (data) VALUES (valeur);
            success := TRUE;

            RAISE NOTICE 'Insertion réussie à la tentative %', tentatives;

        EXCEPTION
            WHEN unique_violation THEN
                RAISE NOTICE 'Tentative % échouée (doublon), réessai...', tentatives;
                -- Modifier la valeur pour éviter le doublon
                valeur := valeur || '_' || tentatives;

            WHEN OTHERS THEN
                RAISE NOTICE 'Erreur inattendue : %', SQLERRM;
                EXIT;  -- Sortir de la boucle
        END;
    END LOOP;

    IF success THEN
        RETURN 'Succès après ' || tentatives || ' tentative(s)';
    ELSE
        RETURN 'Échec après ' || max_tentatives || ' tentatives';
    END IF;
END;
$$;
```

### 8.3. Validation de données avec messages détaillés

```sql
CREATE OR REPLACE FUNCTION creer_commande(
    client_id INTEGER,
    produit_id INTEGER,
    quantite INTEGER
)
RETURNS INTEGER  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    commande_id INTEGER;
    client_actif BOOLEAN;
    stock_disponible INTEGER;
    v_erreur TEXT;
BEGIN
    -- Validation 1 : Client existe et actif
    BEGIN
        SELECT actif INTO STRICT client_actif FROM clients WHERE id = client_id;

        IF NOT client_actif THEN
            RAISE EXCEPTION 'Client % inactif', client_id;
        END IF;
    EXCEPTION
        WHEN no_data_found THEN
            RAISE EXCEPTION 'Client % introuvable', client_id
                USING HINT = 'Vérifiez l''ID du client';
    END;

    -- Validation 2 : Produit existe et stock suffisant
    BEGIN
        SELECT stock INTO STRICT stock_disponible FROM produits WHERE id = produit_id;

        IF stock_disponible < quantite THEN
            RAISE EXCEPTION 'Stock insuffisant : % disponible(s), % demandé(s)',
                           stock_disponible, quantite
                USING HINT = 'Réduisez la quantité ou choisissez un autre produit';
        END IF;
    EXCEPTION
        WHEN no_data_found THEN
            RAISE EXCEPTION 'Produit % introuvable', produit_id
                USING HINT = 'Vérifiez le catalogue des produits';
    END;

    -- Validation 3 : Quantité positive
    IF quantite <= 0 THEN
        RAISE EXCEPTION 'Quantité invalide : %', quantite
            USING HINT = 'La quantité doit être supérieure à 0';
    END IF;

    -- Créer la commande
    INSERT INTO commandes (client_id, produit_id, quantite, date_creation)
    VALUES (client_id, produit_id, quantite, NOW())
    RETURNING id INTO commande_id;

    -- Mettre à jour le stock
    UPDATE produits SET stock = stock - quantite WHERE id = produit_id;

    RETURN commande_id;

EXCEPTION
    WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS v_erreur = MESSAGE_TEXT;
        RAISE NOTICE 'Échec création commande : %', v_erreur;
        RETURN NULL;
END;
$$;
```

---

## 9. Impact sur les Transactions

### 9.1. Rollback automatique

Quand une exception est capturée, la **sous-transaction** est automatiquement annulée :

```sql
CREATE OR REPLACE FUNCTION demo_rollback()  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    INSERT INTO test VALUES (1, 'Avant erreur');

    -- Ce bloc est une sous-transaction
    BEGIN
        INSERT INTO test VALUES (2, 'Dans le bloc');
        -- Erreur !
        PERFORM 1 / 0;
        INSERT INTO test VALUES (3, 'Après erreur');  -- N'est jamais exécuté
    EXCEPTION
        WHEN division_by_zero THEN
            RAISE NOTICE 'Erreur capturée';
            -- Les INSERT du bloc sont annulés automatiquement
    END;

    INSERT INTO test VALUES (4, 'Après le bloc');

    RETURN 'Terminé';
END;
$$;

-- Résultat : Seules les lignes 1 et 4 sont insérées
-- La ligne 2 est annulée à cause de l'exception
```

### 9.2. Visualisation des sous-transactions

```
┌────────────────────────────────────────┐
│ Transaction principale                 │
│ ├─ INSERT 1 (OK)                       │
│ │                                      │
│ ├─ Sous-transaction (bloc EXCEPTION)   │
│ │  ├─ INSERT 2                         │
│ │  ├─ Division par zéro ! ⚠️           │
│ │  └─ ROLLBACK automatique             │ ← INSERT 2 annulé
│ │                                      │
│ ├─ INSERT 4 (OK)                       │
│ COMMIT                                 │
└────────────────────────────────────────┘

Résultat : INSERT 1 et 4 validés, INSERT 2 annulé
```

---

## 10. Bonnes Pratiques

### ✅ Pratique #1 : Capturer spécifiquement, générique en dernier

```sql
-- ✅ BON : Gérer les exceptions spécifiques d'abord
BEGIN
    -- Code
EXCEPTION
    WHEN division_by_zero THEN
        -- Gestion spécifique
    WHEN unique_violation THEN
        -- Gestion spécifique
    WHEN OTHERS THEN
        -- Gestion générique en dernier recours
END;

-- ❌ ÉVITER : OTHERS en premier masque tout
BEGIN
    -- Code
EXCEPTION
    WHEN OTHERS THEN
        -- Trop générique, on ne sait pas ce qui s'est passé
END;
```

### ✅ Pratique #2 : Toujours logger les erreurs

```sql
CREATE OR REPLACE FUNCTION operation_critique()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    v_erreur TEXT;
    v_context TEXT;
BEGIN
    -- Code critique

EXCEPTION
    WHEN OTHERS THEN
        -- Récupérer les détails
        GET STACKED DIAGNOSTICS
            v_erreur = MESSAGE_TEXT,
            v_context = PG_EXCEPTION_CONTEXT;

        -- Logger dans une table
        INSERT INTO error_log (
            function_name,
            error_message,
            error_context,
            occurred_at
        ) VALUES (
            'operation_critique',
            v_erreur,
            v_context,
            NOW()
        );

        -- Re-lever l'exception
        RAISE;
END;
$$;
```

### ✅ Pratique #3 : Messages d'erreur informatifs

```sql
-- ✅ BON : Message clair et actionnable
RAISE EXCEPTION 'Impossible de créer la commande : client % introuvable. Vérifiez l''ID client.',
               client_id
    USING HINT = 'Consultez la liste des clients valides avec SELECT * FROM clients';

-- ❌ ÉVITER : Message vague
RAISE EXCEPTION 'Erreur';
```

### ✅ Pratique #4 : Ne pas masquer les erreurs importantes

```sql
-- ❌ MAUVAIS : Masquer toutes les erreurs
BEGIN
    -- Code
EXCEPTION
    WHEN OTHERS THEN
        RETURN NULL;  -- L'appelant ne sait pas qu'il y a eu un problème !
END;

-- ✅ BON : Logger et re-lever
BEGIN
    -- Code
EXCEPTION
    WHEN OTHERS THEN
        INSERT INTO error_log ...;
        RAISE;  -- Re-lever l'exception
END;
```

### ✅ Pratique #5 : Utiliser ASSERT pour les validations

```sql
CREATE OR REPLACE FUNCTION traiter_donnees(valeur INTEGER)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Assertion pour valider les préconditions
    ASSERT valeur > 0, 'La valeur doit être positive';
    ASSERT valeur < 1000, 'La valeur doit être inférieure à 1000';

    -- Traitement...
    RETURN 'OK';
END;
$$;

-- Test
SELECT traiter_donnees(-5);
-- ❌ ERREUR : La valeur doit être positive
```

**Note** : `ASSERT` est désactivable avec `SET client_min_messages TO WARNING;` pour la production.

---

## 11. Erreurs Courantes

### ⚠️ Erreur #1 : Oublier que EXCEPTION crée une sous-transaction

```sql
-- ❌ PROBLÈME : Performance
CREATE OR REPLACE FUNCTION mauvaise_pratique()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    i INTEGER;
BEGIN
    FOR i IN 1..10000 LOOP
        BEGIN
            -- Chaque itération crée une sous-transaction !
            INSERT INTO test VALUES (i);
        EXCEPTION
            WHEN OTHERS THEN
                NULL;  -- Ignorer l'erreur
        END;
    END LOOP;
END;
$$;

-- ✅ SOLUTION : Éviter les blocs EXCEPTION inutiles
CREATE OR REPLACE FUNCTION bonne_pratique()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Une seule transaction pour toutes les insertions
    INSERT INTO test SELECT generate_series(1, 10000);
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Erreur : %', SQLERRM;
END;
$$;
```

### ⚠️ Erreur #2 : RETURN dans le bloc EXCEPTION sans cleanup

```sql
-- ❌ PROBLÈME : Ressources non libérées
CREATE OR REPLACE FUNCTION mauvaise_gestion()  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    verrou_acquis BOOLEAN := FALSE;
BEGIN
    -- Acquérir un verrou
    PERFORM pg_advisory_lock(12345);
    verrou_acquis := TRUE;

    -- Code qui pourrait planter
    PERFORM risque_erreur();

    -- Libérer le verrou
    PERFORM pg_advisory_unlock(12345);
    RETURN 'OK';

EXCEPTION
    WHEN OTHERS THEN
        -- ❌ Le verrou n'est jamais libéré !
        RETURN 'Erreur';
END;
$$;

-- ✅ SOLUTION : Cleanup dans EXCEPTION
CREATE OR REPLACE FUNCTION bonne_gestion()  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    PERFORM pg_advisory_lock(12345);

    PERFORM risque_erreur();

    PERFORM pg_advisory_unlock(12345);
    RETURN 'OK';

EXCEPTION
    WHEN OTHERS THEN
        -- ✅ Libérer le verrou même en cas d'erreur
        PERFORM pg_advisory_unlock(12345);
        RETURN 'Erreur : ' || SQLERRM;
END;
$$;
```

### ⚠️ Erreur #3 : Capturer puis ignorer silencieusement

```sql
-- ❌ TRÈS MAUVAIS : Erreur silencieuse
BEGIN
    -- Code critique
EXCEPTION
    WHEN OTHERS THEN
        NULL;  -- Ne rien faire = masquer le problème !
END;

-- ✅ AU MINIMUM : Logger
BEGIN
    -- Code critique
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Erreur capturée : %', SQLERRM;
        -- Et/ou insérer dans une table de logs
END;
```

---

## 12. Debugging avec les Exceptions

### 12.1. Stack trace complet

```sql
CREATE OR REPLACE FUNCTION debug_exception()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    v_state TEXT;
    v_msg TEXT;
    v_detail TEXT;
    v_hint TEXT;
    v_context TEXT;
BEGIN
    -- Code qui génère une erreur
    PERFORM fonction_inexistante();

EXCEPTION
    WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_state = RETURNED_SQLSTATE,
            v_msg = MESSAGE_TEXT,
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT,
            v_context = PG_EXCEPTION_CONTEXT;

        RAISE NOTICE E'──────────────────────────────────────';
        RAISE NOTICE 'EXCEPTION CAPTURÉE';
        RAISE NOTICE E'──────────────────────────────────────';
        RAISE NOTICE 'SQLSTATE    : %', v_state;
        RAISE NOTICE 'Message     : %', v_msg;
        RAISE NOTICE 'Détail      : %', COALESCE(v_detail, '(aucun)');
        RAISE NOTICE 'Conseil     : %', COALESCE(v_hint, '(aucun)');
        RAISE NOTICE 'Contexte    : %', v_context;
        RAISE NOTICE E'──────────────────────────────────────';
END;
$$;
```

### 12.2. Table de debug

```sql
-- Table pour capturer les exceptions
CREATE TABLE exception_debug (
    id SERIAL PRIMARY KEY,
    occurred_at TIMESTAMP DEFAULT NOW(),
    function_name TEXT,
    sqlstate TEXT,
    message TEXT,
    detail TEXT,
    hint TEXT,
    context TEXT
);

CREATE OR REPLACE FUNCTION log_exception_debug(func_name TEXT)  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    v_state TEXT;
    v_msg TEXT;
    v_detail TEXT;
    v_hint TEXT;
    v_context TEXT;
BEGIN
    GET STACKED DIAGNOSTICS
        v_state = RETURNED_SQLSTATE,
        v_msg = MESSAGE_TEXT,
        v_detail = PG_EXCEPTION_DETAIL,
        v_hint = PG_EXCEPTION_HINT,
        v_context = PG_EXCEPTION_CONTEXT;

    INSERT INTO exception_debug (
        function_name, sqlstate, message, detail, hint, context
    ) VALUES (
        func_name, v_state, v_msg, v_detail, v_hint, v_context
    );
END;
$$;

-- Utilisation
CREATE OR REPLACE FUNCTION ma_fonction()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Code
EXCEPTION
    WHEN OTHERS THEN
        PERFORM log_exception_debug('ma_fonction');
        RAISE;
END;
$$;
```

---

## 13. Exceptions Personnalisées

### 13.1. Créer ses propres codes d'erreur

PostgreSQL réserve les codes commençant par `[0-9A-E]`. Vous pouvez utiliser `[F-Z]` :

```sql
CREATE OR REPLACE FUNCTION valider_commande(montant NUMERIC)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    IF montant < 0 THEN
        RAISE EXCEPTION 'Montant négatif interdit'
            USING ERRCODE = 'P0001';  -- Code personnalisé
    END IF;

    IF montant > 100000 THEN
        RAISE EXCEPTION 'Montant trop élevé : limite de 100000'
            USING ERRCODE = 'P0002',
                  HINT = 'Divisez en plusieurs commandes';
    END IF;

    RETURN 'Commande valide';
END;
$$;

-- Capturer l'exception personnalisée
CREATE OR REPLACE FUNCTION tester_commande(montant NUMERIC)  
RETURNS TEXT  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RETURN valider_commande(montant);
EXCEPTION
    WHEN SQLSTATE 'P0001' THEN
        RETURN 'Erreur : ' || SQLERRM;
    WHEN SQLSTATE 'P0002' THEN
        RETURN 'Montant trop élevé';
    WHEN OTHERS THEN
        RETURN 'Erreur inattendue';
END;
$$;
```

---

## 14. Résumé Visuel

```
┌─────────────────────────────────────────────────────────┐
│            GESTION DES EXCEPTIONS PL/pgSQL              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  STRUCTURE DE BASE :                                    │
│  ┌────────────────────────────────────────────────┐     │
│  │ BEGIN                                          │     │
│  │     -- Code normal                             │     │
│  │     INSERT, UPDATE, SELECT ...                 │     │
│  │                                                │     │
│  │ EXCEPTION                                      │     │
│  │     WHEN specific_error THEN                   │     │
│  │         -- Gestion spécifique                  │     │
│  │                                                │     │
│  │     WHEN OTHERS THEN                           │     │
│  │         -- Gestion générique                   │     │
│  │ END;                                           │     │
│  └────────────────────────────────────────────────┘     │
│                                                         │
│  VARIABLES SPÉCIALES :                                  │
│  • SQLERRM          → Message d'erreur                  │
│  • SQLSTATE         → Code d'erreur                     │
│  • GET STACKED DIAGNOSTICS → Détails complets           │
│                                                         │
│  BONNES PRATIQUES :                                     │
│  ✅ Capturer spécifiquement                             │
│  ✅ Logger les erreurs                                  │
│  ✅ Messages informatifs                                │
│  ✅ Éviter EXCEPTION dans les boucles                   │
│  ✅ Cleanup des ressources                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 15. Checklist de Gestion des Exceptions

Avant de finaliser votre fonction/procédure avec gestion d'exceptions :

- [ ] J'ai identifié les erreurs possibles  
- [ ] Je capture les exceptions spécifiques (pas seulement OTHERS)  
- [ ] Je log les erreurs dans une table ou avec RAISE NOTICE  
- [ ] Mes messages d'erreur sont clairs et actionnables  
- [ ] J'ai ajouté des HINT pour aider l'utilisateur  
- [ ] Je libère les ressources (verrous, fichiers) même en cas d'erreur  
- [ ] Je ne masque pas silencieusement les erreurs importantes  
- [ ] J'évite les blocs EXCEPTION dans les boucles (performance)  
- [ ] J'ai testé tous les chemins d'erreur possibles  
- [ ] J'ai documenté les exceptions que ma fonction peut lever

---

## 16. Conclusion

La **gestion des exceptions** en PL/pgSQL vous permet de :

1. **Capturer** les erreurs au lieu de crasher  
2. **Réagir** de manière appropriée (log, retry, message)  
3. **Continuer** l'exécution après une erreur  
4. **Fournir** des messages d'erreur clairs aux utilisateurs

**Points clés à retenir** :
- ✅ Structure `BEGIN...EXCEPTION...END` pour capturer les erreurs  
- ✅ Exceptions spécifiques (`division_by_zero`, `unique_violation`, etc.)  
- ✅ `WHEN OTHERS` pour capturer tout le reste  
- ✅ Variables `SQLERRM` et `SQLSTATE` pour les détails  
- ✅ `RAISE` pour lever des exceptions personnalisées  
- ✅ Blocs imbriqués pour isoler les erreurs  
- ✅ Attention à la performance : éviter EXCEPTION dans les boucles

**Règle d'or** :
> **"Capturez spécifiquement, loggez systématiquement, ne masquez jamais silencieusement."**

Une bonne gestion des exceptions rend votre code :
- Plus **robuste** (ne crashe pas au premier problème)
- Plus **maintenable** (messages d'erreur clairs)
- Plus **debuggable** (logs détaillés)
- Plus **user-friendly** (messages utiles)

---

**🎓 Prochaines étapes dans le tutoriel :**
- 15.7. Autres langages procéduraux (PL/Python, PL/Perl, PL/v8)
- 16. Administration, Configuration et Sécurité
- 16.1. Authentification vs Autorisation : Concepts fondamentaux

**💡 Pour aller plus loin :**
- Documentation officielle : [Errors and Messages](https://www.postgresql.org/docs/current/plpgsql-errors-and-messages.html)
- Liste complète des codes SQLSTATE : [Appendix A](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- Pratiquez la gestion d'erreurs dans vos propres fonctions
- Créez une table centralisée de logs d'erreurs

⏭️ [Autres langages procéduraux](/15-programmation-serveur/07-autres-langages-proceduraux.md)
