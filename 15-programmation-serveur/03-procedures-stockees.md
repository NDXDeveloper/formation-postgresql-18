🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.3. Procédures Stockées et Gestion Transactionnelle (CALL, COMMIT/ROLLBACK)

## Introduction : Fonctions vs Procédures

Jusqu'à présent, nous avons étudié les **fonctions** dans PostgreSQL. À partir de PostgreSQL 11, un nouveau type d'objet a été introduit : les **procédures stockées** (ou simplement "procédures").

### Quelle est la différence fondamentale ?

| Aspect | Fonction | Procédure |
|--------|----------|-----------|
| **Retour de valeur** | ✅ Obligatoire (RETURNS) | ❌ Aucune valeur retournée |
| **Appel** | Dans une requête SELECT | Avec la commande CALL |
| **Gestion transactionnelle** | ❌ Non (hérite de la transaction parente) | ✅ Oui (COMMIT/ROLLBACK internes) |
| **Mot-clé** | `CREATE FUNCTION` | `CREATE PROCEDURE` |
| **Cas d'usage** | Calculs, transformations, requêtes | Traitements métier complexes, batch |

### Analogie simple

Pensez à la différence comme celle entre :
- **Fonction** = Une **calculatrice** : vous donnez des nombres, elle retourne un résultat  
- **Procédure** = Un **script d'automatisation** : elle fait un ensemble de tâches et termine

---

## 1. Créer et Appeler une Procédure Simple

### 1.1. Syntaxe de base

```sql
CREATE PROCEDURE nom_procedure(parametre1 TYPE, parametre2 TYPE, ...)  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Corps de la procédure
    -- Instructions SQL, logique métier
END;
$$;
```

**Différences clés avec une fonction** :
- `CREATE PROCEDURE` au lieu de `CREATE FUNCTION`
- Pas de clause `RETURNS`
- Pas de `RETURN` à la fin (sauf pour sortir prématurément)

### 1.2. Exemple simple : Afficher un message

```sql
CREATE PROCEDURE afficher_message(message TEXT)  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RAISE NOTICE 'Message : %', message;
END;
$$;

-- Appel de la procédure avec CALL
CALL afficher_message('Bonjour depuis une procédure !');
-- Affiche : NOTICE:  Message : Bonjour depuis une procédure !
```

**Points importants** :
- ✅ On utilise `CALL` pour exécuter une procédure  
- ✅ Pas de `SELECT`, contrairement aux fonctions  
- ✅ Pas de valeur de retour

### 1.3. Exemple avec modification de données

```sql
CREATE PROCEDURE ajouter_client(
    nom_client TEXT,
    email_client TEXT
)
LANGUAGE plpgsql  
AS $$  
BEGIN  
    INSERT INTO clients (nom, email, date_creation)
    VALUES (nom_client, email_client, NOW());

    RAISE NOTICE 'Client % ajouté avec succès', nom_client;
END;
$$;

-- Appel
CALL ajouter_client('Alice Martin', 'alice@example.com');
```

---

## 2. La Grande Innovation : Gestion Transactionnelle

### 2.1. Le problème avec les fonctions

Avec une **fonction**, vous ne pouvez pas contrôler les transactions :

```sql
CREATE FUNCTION traiter_commandes_fonction()  
RETURNS VOID  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Traitement 1
    INSERT INTO logs VALUES ('Début traitement');

    -- ❌ IMPOSSIBLE dans une fonction !
    -- COMMIT;  -- ERREUR : cannot commit/rollback in a function

    -- Traitement 2
    UPDATE stock SET quantite = quantite - 1 WHERE id = 1;
END;
$$;

-- Tout se passe dans UNE SEULE transaction
BEGIN;  
SELECT traiter_commandes_fonction();  
COMMIT;  -- Un seul COMMIT pour tout  
```

**Problème** : Si le traitement est long et touche beaucoup de données, vous bloquez des ressources pendant toute la durée.

### 2.2. La solution avec les procédures

Avec une **procédure**, vous pouvez faire des `COMMIT` et `ROLLBACK` **à l'intérieur** :

```sql
CREATE PROCEDURE traiter_commandes_procedure()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Traitement 1
    INSERT INTO logs VALUES ('Début traitement');
    COMMIT;  -- ✅ Valide ce premier traitement

    -- Traitement 2 (dans une nouvelle transaction)
    UPDATE stock SET quantite = quantite - 1 WHERE id = 1;
    COMMIT;  -- ✅ Valide ce deuxième traitement

    -- Si une erreur survient ici, les COMMIT précédents sont sauvegardés !
END;
$$;

-- Appel simple
CALL traiter_commandes_procedure();
```

**Avantage majeur** : Vous divisez un long traitement en petites transactions indépendantes.

### 2.3. Pourquoi est-ce important ?

#### Cas d'usage : Traitement par lots (Batch Processing)

Imaginez que vous devez traiter 1 million de lignes :

**Sans procédure (fonction)** :
```
┌─────────────────────────────────────────┐
│  BEGIN TRANSACTION                      │
│  ├─ Traiter 1 million de lignes         │
│  │  (peut prendre 30 minutes)           │
│  │  Bloque les ressources               │
│  │  Si erreur à la fin → tout rollback  │
│  COMMIT                                 │
└─────────────────────────────────────────┘
```

**Avec procédure** :
```
┌──────────────────────────────────────────┐
│  LOOP sur lots de 1000 lignes            │
│  │                                       │
│  ├─ BEGIN (automatique)                  │
│  │  ├─ Traiter 1000 lignes               │
│  │  COMMIT  ← Libère les verrous !       │
│  │                                       │
│  ├─ BEGIN (automatique)                  │
│  │  ├─ Traiter 1000 lignes               │
│  │  COMMIT  ← Libère les verrous !       │
│  │                                       │
│  └─ ... (répéter 1000 fois)              │
└──────────────────────────────────────────┘
```

**Avantages** :
- ✅ Libération régulière des verrous  
- ✅ Visibilité progressive des changements  
- ✅ Moins de risque de bloat (gonflement des tables)  
- ✅ Si erreur, seul le lot en cours est perdu

---

## 3. COMMIT et ROLLBACK dans les Procédures

### 3.1. COMMIT : Valider une partie du traitement

```sql
CREATE PROCEDURE traiter_par_lots()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    compteur INTEGER := 0;
BEGIN
    -- Traitement lot 1
    INSERT INTO resultats VALUES ('Lot 1 traité');
    compteur := compteur + 1;
    COMMIT;  -- ✅ Lot 1 validé et persisté

    RAISE NOTICE 'Lot % validé', compteur;

    -- Traitement lot 2
    INSERT INTO resultats VALUES ('Lot 2 traité');
    compteur := compteur + 1;
    COMMIT;  -- ✅ Lot 2 validé et persisté

    RAISE NOTICE 'Lot % validé', compteur;

    -- Si une erreur survient ici, les lots 1 et 2 sont déjà sauvegardés !
END;
$$;

CALL traiter_par_lots();
-- NOTICE:  Lot 1 validé
-- NOTICE:  Lot 2 validé
```

**Ce qui se passe** :
1. Insertion lot 1 → COMMIT → Données écrites sur disque  
2. Insertion lot 2 → COMMIT → Données écrites sur disque  
3. Même si une erreur survient après, les lots 1 et 2 restent en base

### 3.2. ROLLBACK : Annuler une partie du traitement

```sql
CREATE PROCEDURE traiter_avec_validation()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    montant_total NUMERIC;
BEGIN
    -- Tentative de traitement
    INSERT INTO commandes (client_id, montant) VALUES (1, 500);

    -- Vérification
    SELECT SUM(montant) INTO montant_total FROM commandes WHERE client_id = 1;

    IF montant_total > 10000 THEN
        -- Dépassement du plafond autorisé
        RAISE NOTICE 'Plafond dépassé, annulation';
        ROLLBACK;  -- ✅ Annule l'INSERT ci-dessus
    ELSE
        RAISE NOTICE 'Commande validée';
        COMMIT;  -- ✅ Valide l'INSERT
    END IF;
END;
$$;

CALL traiter_avec_validation();
```

**Comportement** :
- Si `montant_total > 10000` : l'INSERT est annulé (ROLLBACK)
- Sinon : l'INSERT est validé (COMMIT)

### 3.3. COMMIT automatique en fin de procédure

Si vous ne mettez pas de `COMMIT` explicite, PostgreSQL fait un COMMIT automatique à la fin :

```sql
CREATE PROCEDURE insertion_simple()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    INSERT INTO logs VALUES ('Test');
    -- Pas de COMMIT explicite
END;
$$;

CALL insertion_simple();
-- COMMIT automatique à la fin → Données sauvegardées
```

---

## 4. Exemple Complet : Traitement par Lots

### 4.1. Cas d'usage : Archiver des anciennes commandes

Imaginons que nous voulons archiver 100 000 commandes de plus de 2 ans, mais par lots de 1000 pour ne pas bloquer la base.

```sql
CREATE PROCEDURE archiver_anciennes_commandes()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    nb_traitees INTEGER := 0;
    nb_total INTEGER := 0;
    lot_size INTEGER := 1000;
BEGIN
    -- Compter le nombre total à traiter
    SELECT COUNT(*) INTO nb_total
    FROM commandes
    WHERE date_creation < NOW() - INTERVAL '2 years';

    RAISE NOTICE 'Début archivage de % commandes', nb_total;

    -- Boucle de traitement par lots
    LOOP
        -- Archiver un lot
        WITH lot AS (
            SELECT id FROM commandes
            WHERE date_creation < NOW() - INTERVAL '2 years'
            LIMIT lot_size
        )
        INSERT INTO commandes_archive
        SELECT * FROM commandes WHERE id IN (SELECT id FROM lot);

        -- Compter le nombre de lignes archivées
        GET DIAGNOSTICS nb_traitees = ROW_COUNT;

        -- Si aucune ligne traitée, on a fini
        EXIT WHEN nb_traitees = 0;

        -- Supprimer les commandes archivées
        DELETE FROM commandes
        WHERE id IN (
            SELECT id FROM commandes
            WHERE date_creation < NOW() - INTERVAL '2 years'
            LIMIT lot_size
        );

        -- ✅ COMMIT après chaque lot
        COMMIT;

        RAISE NOTICE 'Lot de % commandes archivé', nb_traitees;

        -- Petite pause pour ne pas saturer le système
        PERFORM pg_sleep(0.1);  -- 100ms de pause

    END LOOP;

    RAISE NOTICE 'Archivage terminé';
END;
$$;

-- Exécution
CALL archiver_anciennes_commandes();
-- NOTICE:  Début archivage de 100000 commandes
-- NOTICE:  Lot de 1000 commandes archivé
-- NOTICE:  Lot de 1000 commandes archivé
-- ... (répété 100 fois)
-- NOTICE:  Archivage terminé
```

**Avantages de cette approche** :
- ✅ Traitement progressif avec COMMIT réguliers  
- ✅ Libération des verrous entre chaque lot  
- ✅ Possibilité de suivre la progression  
- ✅ Possibilité d'interrompre sans tout perdre  
- ✅ Moins de stress sur le WAL et les buffers

---

## 5. Gestion d'Erreurs avec COMMIT/ROLLBACK

### 5.1. Bloc EXCEPTION dans les procédures

Les procédures peuvent gérer les erreurs avec des blocs `EXCEPTION`, mais attention à l'interaction avec COMMIT/ROLLBACK :

```sql
CREATE PROCEDURE traiter_avec_gestion_erreurs()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Bloc 1
    BEGIN
        INSERT INTO logs VALUES ('Traitement 1');
        COMMIT;
        RAISE NOTICE 'Traitement 1 : OK';
    EXCEPTION
        WHEN OTHERS THEN
            RAISE NOTICE 'Erreur dans traitement 1 : %', SQLERRM;
            ROLLBACK;  -- Annule ce qui n'était pas encore commité
    END;

    -- Bloc 2 (s'exécute même si le bloc 1 a échoué)
    BEGIN
        INSERT INTO logs VALUES ('Traitement 2');
        -- Simulation d'une erreur
        PERFORM 1/0;  -- Division par zéro !
        COMMIT;
    EXCEPTION
        WHEN division_by_zero THEN
            RAISE NOTICE 'Erreur division par zéro dans traitement 2';
            ROLLBACK;  -- Annule l'INSERT du traitement 2
    END;

    -- Bloc 3 (s'exécute quand même)
    INSERT INTO logs VALUES ('Traitement 3');
    COMMIT;
    RAISE NOTICE 'Traitement 3 : OK';

END;
$$;

CALL traiter_avec_gestion_erreurs();
-- NOTICE:  Traitement 1 : OK
-- NOTICE:  Erreur division par zéro dans traitement 2
-- NOTICE:  Traitement 3 : OK

-- Résultat : Les logs 1 et 3 sont dans la table, mais pas le log 2
```

**Important** :
- ✅ Chaque bloc EXCEPTION peut faire un ROLLBACK indépendant  
- ✅ L'exécution continue après un ROLLBACK  
- ✅ Seul le bloc en erreur est annulé, pas toute la procédure

### 5.2. Gestion d'erreurs globale

```sql
CREATE PROCEDURE traiter_avec_securite()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    v_erreur TEXT;
BEGIN
    -- Tentative de traitement
    BEGIN
        INSERT INTO commandes VALUES (1, 100);
        INSERT INTO commandes VALUES (2, 200);
        INSERT INTO commandes VALUES (3, 300);
        COMMIT;

    EXCEPTION
        WHEN OTHERS THEN
            GET STACKED DIAGNOSTICS v_erreur = MESSAGE_TEXT;
            RAISE NOTICE 'ERREUR : %', v_erreur;
            ROLLBACK;

            -- Insérer dans une table d'erreurs
            INSERT INTO erreurs_traitement (message, timestamp)
            VALUES (v_erreur, NOW());
            COMMIT;  -- Valider l'enregistrement de l'erreur
    END;
END;
$$;
```

---

## 6. Paramètres IN, OUT et INOUT

Les procédures peuvent avoir des paramètres de sortie (contrairement aux fonctions qui utilisent RETURNS) :

### 6.1. Paramètres OUT

```sql
CREATE PROCEDURE calculer_statistiques(
    IN client_id INTEGER,
    OUT nb_commandes INTEGER,
    OUT montant_total NUMERIC
)
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Calcul des statistiques
    SELECT COUNT(*), COALESCE(SUM(montant), 0)
    INTO nb_commandes, montant_total
    FROM commandes
    WHERE id_client = client_id;

    RAISE NOTICE 'Client % : % commandes pour un total de %',
                 client_id, nb_commandes, montant_total;
END;
$$;

-- Appel avec récupération des valeurs OUT
DO $$  
DECLARE  
    v_nb INTEGER;
    v_total NUMERIC;
BEGIN
    CALL calculer_statistiques(42, v_nb, v_total);
    RAISE NOTICE 'Résultats : % commandes, % euros', v_nb, v_total;
END;
$$;
```

### 6.2. Paramètres INOUT

```sql
CREATE PROCEDURE incrementer_compteur(
    INOUT compteur INTEGER
)
LANGUAGE plpgsql  
AS $$  
BEGIN  
    compteur := compteur + 1;
    RAISE NOTICE 'Nouveau compteur : %', compteur;
END;
$$;

-- Utilisation
DO $$  
DECLARE  
    mon_compteur INTEGER := 10;
BEGIN
    RAISE NOTICE 'Avant : %', mon_compteur;
    CALL incrementer_compteur(mon_compteur);
    RAISE NOTICE 'Après : %', mon_compteur;
END;
$$;
-- NOTICE:  Avant : 10
-- NOTICE:  Nouveau compteur : 11
-- NOTICE:  Après : 11
```

---

## 7. Différences Comportementales Importantes

### 7.1. Transactions et points de sauvegarde

**Dans une fonction** :
```sql
BEGIN;  -- Transaction externe

SELECT ma_fonction();  -- S'exécute dans cette transaction
-- La fonction ne peut pas COMMIT/ROLLBACK

COMMIT;  -- Tout est validé ici
```

**Dans une procédure** :
```sql
-- Pas besoin de BEGIN externe

CALL ma_procedure();  -- Crée ses propres transactions internes
-- COMMIT et ROLLBACK internes possibles

-- Validation automatique à la fin
```

### 7.2. Visibilité des changements

```sql
CREATE PROCEDURE demo_visibilite()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Insertion 1
    INSERT INTO test VALUES (1);
    RAISE NOTICE 'Insertion 1 effectuée';

    -- Pas encore de COMMIT → Données non visibles par les autres sessions
    PERFORM pg_sleep(5);

    COMMIT;  -- ✅ Maintenant visible !
    RAISE NOTICE 'COMMIT 1 effectué - données visibles';

    -- Insertion 2
    INSERT INTO test VALUES (2);
    COMMIT;
    RAISE NOTICE 'COMMIT 2 effectué';
END;
$$;

-- Session 1
CALL demo_visibilite();

-- Session 2 (pendant l'exécution de Session 1)
SELECT * FROM test;
-- Voit les données progressivement après chaque COMMIT
```

---

## 8. Cas d'Usage Typiques des Procédures

### 8.1. Maintenance de base de données

```sql
CREATE PROCEDURE nettoyer_anciennes_donnees()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    nb_supprimees INTEGER;
BEGIN
    -- Nettoyer les logs de plus de 90 jours
    DELETE FROM logs WHERE timestamp < NOW() - INTERVAL '90 days';
    GET DIAGNOSTICS nb_supprimees = ROW_COUNT;
    COMMIT;
    RAISE NOTICE 'Logs nettoyés : %', nb_supprimees;

    -- Nettoyer les sessions expirées
    DELETE FROM sessions WHERE date_expiration < NOW();
    GET DIAGNOSTICS nb_supprimees = ROW_COUNT;
    COMMIT;
    RAISE NOTICE 'Sessions nettoyées : %', nb_supprimees;

    -- VACUUM des tables concernées
    VACUUM ANALYZE logs;
    VACUUM ANALYZE sessions;

    RAISE NOTICE 'Maintenance terminée';
END;
$$;

-- Planifier avec pg_cron
-- SELECT cron.schedule('maintenance-quotidienne', '0 2 * * *',
--                      'CALL nettoyer_anciennes_donnees()');
```

### 8.2. Import de données par lots

```sql
CREATE PROCEDURE importer_fichier_csv()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    lot_size INTEGER := 10000;
    nb_importees INTEGER := 0;
BEGIN
    -- Créer une table temporaire pour le staging
    CREATE TEMP TABLE staging_import (
        donnee1 TEXT,
        donnee2 TEXT,
        donnee3 INTEGER
    );

    -- Copier le fichier CSV
    COPY staging_import FROM '/tmp/import.csv' CSV HEADER;
    COMMIT;
    RAISE NOTICE 'Données chargées dans staging';

    -- Insérer par lots
    LOOP
        WITH lot AS (
            SELECT * FROM staging_import LIMIT lot_size
        )
        INSERT INTO table_finale
        SELECT * FROM lot;

        GET DIAGNOSTICS nb_importees = ROW_COUNT;
        EXIT WHEN nb_importees = 0;

        DELETE FROM staging_import
        WHERE ctid IN (SELECT ctid FROM staging_import LIMIT lot_size);

        COMMIT;
        RAISE NOTICE 'Lot de % lignes importé', nb_importees;
    END LOOP;

    DROP TABLE staging_import;
    RAISE NOTICE 'Import terminé';
END;
$$;
```

### 8.3. Traitements métier complexes

```sql
CREATE PROCEDURE cloturer_mois()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    mois_cloture DATE;
BEGIN
    -- Déterminer le mois à clôturer
    mois_cloture := date_trunc('month', NOW() - INTERVAL '1 month');

    RAISE NOTICE 'Début clôture du mois %', mois_cloture;

    -- Étape 1 : Calculer les totaux
    INSERT INTO totaux_mensuels
    SELECT
        mois_cloture,
        client_id,
        SUM(montant)
    FROM commandes
    WHERE date_trunc('month', date_creation) = mois_cloture
    GROUP BY client_id;
    COMMIT;
    RAISE NOTICE 'Totaux calculés';

    -- Étape 2 : Générer les factures
    INSERT INTO factures (client_id, mois, montant, statut)
    SELECT client_id, mois, total, 'A_ENVOYER'
    FROM totaux_mensuels
    WHERE mois = mois_cloture;
    COMMIT;
    RAISE NOTICE 'Factures générées';

    -- Étape 3 : Marquer les commandes comme facturées
    UPDATE commandes
    SET facturee = TRUE
    WHERE date_trunc('month', date_creation) = mois_cloture;
    COMMIT;
    RAISE NOTICE 'Commandes marquées';

    -- Étape 4 : Mettre à jour le statut de clôture
    INSERT INTO clotures (mois, date_cloture)
    VALUES (mois_cloture, NOW());
    COMMIT;

    RAISE NOTICE 'Clôture terminée pour %', mois_cloture;
END;
$$;
```

---

## 9. Limitations et Considérations

### 9.1. Pas de COMMIT/ROLLBACK dans les blocs EXCEPTION

⚠️ **Important** : Vous ne pouvez pas faire de COMMIT dans un bloc EXCEPTION à cause de la gestion automatique des sous-transactions.

```sql
-- ❌ CECI NE FONCTIONNE PAS
CREATE PROCEDURE exemple_incorrect()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    INSERT INTO test VALUES (1);

EXCEPTION
    WHEN OTHERS THEN
        COMMIT;  -- ❌ ERREUR : cannot commit in exception block
END;
$$;
```

**Solution** : Structurer avec des sous-blocs :

```sql
-- ✅ CECI FONCTIONNE
CREATE PROCEDURE exemple_correct()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    BEGIN
        INSERT INTO test VALUES (1);
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            RAISE NOTICE 'Erreur capturée';
            ROLLBACK;
    END;

    -- Le COMMIT/ROLLBACK est dans le sous-bloc, pas dans le EXCEPTION
END;
$$;
```

### 9.2. Performance : Le coût des COMMIT

Chaque COMMIT a un coût :
- Écriture sur disque (fsync)
- Mise à jour des fichiers WAL
- Libération des verrous

**Recommandation** : Trouvez le bon équilibre entre :
- Trop de COMMIT → Overhead de performance
- Trop peu de COMMIT → Verrous prolongés, bloat

**Règle générale** : COMMIT tous les 100 à 10 000 lignes selon le contexte

---

## 10. Comparaison Finale : Fonction vs Procédure

### Exemple équivalent pour illustrer la différence

**Fonction** (pas de contrôle transactionnel) :
```sql
CREATE FUNCTION traiter_lot_fonction()  
RETURNS INTEGER  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    nb_traites INTEGER := 0;
BEGIN
    -- Tout dans une seule transaction
    INSERT INTO logs VALUES ('Début');
    UPDATE stock SET quantite = quantite - 1;
    INSERT INTO logs VALUES ('Fin');

    nb_traites := 3;
    RETURN nb_traites;

    -- ❌ Impossible de faire COMMIT ici
END;
$$;

-- Appel
BEGIN;  
SELECT traiter_lot_fonction();  -- Tout ou rien  
COMMIT;  -- Un seul COMMIT à la fin  
```

**Procédure** (contrôle transactionnel) :
```sql
CREATE PROCEDURE traiter_lot_procedure()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Transaction 1
    INSERT INTO logs VALUES ('Début');
    COMMIT;  -- ✅ Validé immédiatement

    -- Transaction 2
    UPDATE stock SET quantite = quantite - 1;
    COMMIT;  -- ✅ Validé immédiatement

    -- Transaction 3
    INSERT INTO logs VALUES ('Fin');
    COMMIT;  -- ✅ Validé immédiatement
END;
$$;

-- Appel
CALL traiter_lot_procedure();  -- Contrôle fin des transactions
```

---

## 11. Bonnes Pratiques

### ✅ Pratique #1 : Utiliser des procédures pour les traitements par lots

```sql
-- ✅ BON : Procédure avec COMMIT réguliers pour gros volume
CREATE PROCEDURE traiter_gros_volume()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    LOOP
        -- Traiter un lot
        COMMIT;
        EXIT WHEN condition;
    END LOOP;
END;
$$;
```

### ✅ Pratique #2 : Logger la progression

```sql
CREATE PROCEDURE traitement_avec_logs()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    etape TEXT;
BEGIN
    etape := 'Étape 1';
    -- Traitement
    COMMIT;
    INSERT INTO logs_procedures (procedure_name, etape, timestamp)
    VALUES ('traitement_avec_logs', etape, NOW());
    COMMIT;

    etape := 'Étape 2';
    -- Traitement
    COMMIT;
    INSERT INTO logs_procedures (procedure_name, etape, timestamp)
    VALUES ('traitement_avec_logs', etape, NOW());
    COMMIT;
END;
$$;
```

### ✅ Pratique #3 : Gérer les erreurs par bloc

```sql
CREATE PROCEDURE traitement_resilient()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Chaque étape dans son propre bloc avec gestion d'erreur
    BEGIN
        -- Étape 1
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK;
            RAISE NOTICE 'Erreur étape 1';
    END;

    BEGIN
        -- Étape 2
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK;
            RAISE NOTICE 'Erreur étape 2';
    END;
END;
$$;
```

### ✅ Pratique #4 : Documenter les transactions

```sql
CREATE PROCEDURE ma_procedure()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Transaction 1 : Préparation des données
    -- COMMIT car les données préparées doivent être visibles
    -- pour les étapes suivantes même en cas d'erreur ultérieure
    INSERT INTO staging ...;
    COMMIT;

    -- Transaction 2 : Traitement principal
    -- COMMIT par lots de 1000 pour libérer les verrous
    LOOP
        -- Traiter 1000 lignes
        COMMIT;
    END LOOP;

    -- Transaction 3 : Finalisation
    -- COMMIT car on veut garder les statistiques
    -- même si une notification échoue après
    UPDATE statistiques ...;
    COMMIT;
END;
$$;

COMMENT ON PROCEDURE ma_procedure IS
'Traite les données en 3 phases avec COMMIT entre chaque phase.
Chaque phase est indépendante et persistante.';
```

---

## 12. Erreurs Courantes

### ⚠️ Erreur #1 : Utiliser RETURN au lieu de COMMIT

```sql
-- ❌ MAUVAIS : RETURN ne valide pas la transaction !
CREATE PROCEDURE erreur_return()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    INSERT INTO test VALUES (1);
    RETURN;  -- Sort de la procédure mais ne valide pas !
END;
$$;
-- Les données sont validées automatiquement à la fin

-- ✅ CORRECT : Utiliser COMMIT explicite si nécessaire
CREATE PROCEDURE correct_commit()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    INSERT INTO test VALUES (1);
    COMMIT;  -- Validation explicite
END;
$$;
```

### ⚠️ Erreur #2 : Oublier la gestion d'erreurs

```sql
-- ❌ MAUVAIS : Pas de gestion d'erreur
CREATE PROCEDURE sans_gestion_erreur()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Si une erreur survient, toute la procédure échoue
    INSERT INTO table1 VALUES (1);
    COMMIT;
    INSERT INTO table2 VALUES (2);  -- Si erreur ici, table1 est quand même validée !
    COMMIT;
END;
$$;

-- ✅ CORRECT : Gérer les erreurs
CREATE PROCEDURE avec_gestion_erreur()  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    BEGIN
        INSERT INTO table1 VALUES (1);
        COMMIT;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK;
        RAISE NOTICE 'Erreur table1';
    END;

    BEGIN
        INSERT INTO table2 VALUES (2);
        COMMIT;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK;
        RAISE NOTICE 'Erreur table2';
    END;
END;
$$;
```

### ⚠️ Erreur #3 : Trop de COMMIT (overhead)

```sql
-- ❌ MAUVAIS : COMMIT à chaque ligne (très lent)
CREATE PROCEDURE trop_de_commit()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    r RECORD;
BEGIN
    FOR r IN SELECT * FROM grande_table LOOP
        UPDATE autre_table SET col = r.val WHERE id = r.id;
        COMMIT;  -- 1 million de COMMIT si 1 million de lignes !
    END LOOP;
END;
$$;

-- ✅ CORRECT : COMMIT par lots
CREATE PROCEDURE commit_par_lots()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    r RECORD;
    compteur INTEGER := 0;
BEGIN
    FOR r IN SELECT * FROM grande_table LOOP
        UPDATE autre_table SET col = r.val WHERE id = r.id;
        compteur := compteur + 1;

        IF compteur % 1000 = 0 THEN  -- COMMIT tous les 1000
            COMMIT;
        END IF;
    END LOOP;

    COMMIT;  -- Dernier lot
END;
$$;
```

---

## 13. Quand Utiliser une Procédure au Lieu d'une Fonction ?

### Utilisez une PROCÉDURE quand :

- ✅ Vous avez besoin de **COMMIT/ROLLBACK** internes  
- ✅ Vous traitez de **gros volumes** de données (batch processing)  
- ✅ Vous voulez **libérer les verrous** progressivement  
- ✅ Vous orchestrez des **traitements métier complexes** multi-étapes  
- ✅ Vous faites de la **maintenance** de base de données  
- ✅ Vous n'avez **pas besoin de retourner une valeur**

### Utilisez une FONCTION quand :

- ✅ Vous devez **retourner une valeur** (calcul, transformation)  
- ✅ Vous voulez l'appeler dans un **SELECT**  
- ✅ Vous avez besoin d'**atomicité** (tout ou rien)  
- ✅ Vous créez des **index fonctionnels**  
- ✅ Le traitement est **rapide** et ne nécessite pas de COMMIT intermédiaires

---

## 14. Résumé Visuel

```
┌─────────────────────────────────────────────────────────────┐
│              FONCTIONS vs PROCÉDURES                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  FONCTION                      PROCÉDURE                    │
│  ────────────                  ──────────                   │
│                                                             │
│  CREATE FUNCTION               CREATE PROCEDURE             │
│  RETURNS type                  Pas de RETURNS               │
│  SELECT ma_fonction()          CALL ma_procedure()          │
│  ❌ Pas de COMMIT/ROLLBACK     ✅ COMMIT/ROLLBACK           │
│  Transaction parente           Transactions internes        │
│  Retourne une valeur           Pas de retour (ou OUT)       │
│  Peut être IMMUTABLE           Toujours VOLATILE            │
│                                                             │
│  📊 Cas d'usage :              📦 Cas d'usage :             │
│  • Calculs                     • Batch processing           │
│  • Transformations             • Maintenance                │
│  • Requêtes encapsulées        • Imports                    │
│  • Index fonctionnels          • Traitements métier         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 15. Exemples Récapitulatifs

### Exemple 1 : Migration de données historiques

```sql
CREATE PROCEDURE migrer_donnees_historiques(annee INTEGER)  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    nb_migrees INTEGER := 0;
    total_migrees INTEGER := 0;
    lot_size INTEGER := 5000;
BEGIN
    RAISE NOTICE 'Début migration année %', annee;

    LOOP
        -- Copier un lot
        WITH lot AS (
            SELECT * FROM transactions
            WHERE EXTRACT(YEAR FROM date_transaction) = annee
            LIMIT lot_size
        )
        INSERT INTO transactions_archive
        SELECT * FROM lot;

        GET DIAGNOSTICS nb_migrees = ROW_COUNT;
        EXIT WHEN nb_migrees = 0;

        -- Supprimer les lignes migrées
        DELETE FROM transactions
        WHERE id IN (
            SELECT id FROM transactions
            WHERE EXTRACT(YEAR FROM date_transaction) = annee
            LIMIT lot_size
        );

        total_migrees := total_migrees + nb_migrees;

        COMMIT;  -- ✅ Valider chaque lot

        RAISE NOTICE 'Migration : % lignes (total : %)',
                     nb_migrees, total_migrees;

        -- Pause pour ne pas saturer
        PERFORM pg_sleep(0.05);
    END LOOP;

    RAISE NOTICE 'Migration terminée : % lignes au total', total_migrees;
END;
$$;

-- Exécution
CALL migrer_donnees_historiques(2020);
```

### Exemple 2 : Réconciliation bancaire automatique

```sql
CREATE PROCEDURE reconcilier_transactions()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    nb_reconciliees INTEGER;
BEGIN
    -- Étape 1 : Marquer les correspondances exactes
    UPDATE transactions t
    SET statut = 'RECONCILIE'
    FROM relevés_bancaires rb
    WHERE t.montant = rb.montant
      AND t.date = rb.date
      AND t.statut = 'EN_ATTENTE'
      AND rb.statut = 'NON_RECONCILIE';

    GET DIAGNOSTICS nb_reconciliees = ROW_COUNT;
    COMMIT;
    RAISE NOTICE 'Étape 1 : % correspondances exactes', nb_reconciliees;

    -- Étape 2 : Marquer les correspondances avec tolérance de date (±2 jours)
    UPDATE transactions t
    SET statut = 'RECONCILIE_PARTIEL'
    FROM relevés_bancaires rb
    WHERE t.montant = rb.montant
      AND ABS(EXTRACT(EPOCH FROM (t.date - rb.date))/86400) <= 2
      AND t.statut = 'EN_ATTENTE'
      AND rb.statut = 'NON_RECONCILIE';

    GET DIAGNOSTICS nb_reconciliees = ROW_COUNT;
    COMMIT;
    RAISE NOTICE 'Étape 2 : % correspondances partielles', nb_reconciliees;

    -- Étape 3 : Enregistrer le rapport
    INSERT INTO rapports_reconciliation (date_rapport, nb_exact, nb_partiel)
    VALUES (NOW(), nb_reconciliees, nb_reconciliees);
    COMMIT;

    RAISE NOTICE 'Réconciliation terminée';
END;
$$;
```

---

## 16. Conclusion

Les **procédures stockées** sont un outil puissant de PostgreSQL pour :

1. **Gérer des transactions complexes** avec COMMIT/ROLLBACK internes  
2. **Traiter de gros volumes** de données par lots sans bloquer la base  
3. **Orchestrer des processus métier** multi-étapes avec validation progressive  
4. **Maintenir la base de données** avec des opérations de nettoyage et archivage

**Points clés à retenir** :
- ✅ Les procédures permettent COMMIT/ROLLBACK internes (contrairement aux fonctions)  
- ✅ Utilisez CALL pour exécuter une procédure  
- ✅ Les procédures ne retournent pas de valeur (sauf paramètres OUT/INOUT)  
- ✅ Validez régulièrement par lots pour libérer les verrous  
- ✅ Gérez les erreurs avec des blocs EXCEPTION imbriqués

**Règle de décision** :
> **Besoin de COMMIT/ROLLBACK internes ou traitement par lots → PROCÉDURE**  
> **Besoin de retourner une valeur ou utilisation dans SELECT → FONCTION**

---

**🎓 Prochaines étapes dans le tutoriel :**
- 15.4. Triggers (BEFORE, AFTER, INSTEAD OF)
- 15.5. Event Triggers : Surveillance DDL
- 15.6. Gestion des exceptions (BEGIN...EXCEPTION...END)

**💡 Pour aller plus loin :**
- Documentation officielle : [Procedures](https://www.postgresql.org/docs/current/sql-createprocedure.html)
- Testez des traitements par lots avec différentes tailles de COMMIT
- Mesurez l'impact des COMMIT réguliers sur les performances

⏭️ [Triggers (BEFORE, AFTER, INSTEAD OF)](/15-programmation-serveur/04-triggers.md)
