🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.5. Event Triggers : Surveillance DDL

## Introduction : Surveiller les Changements de Structure

Dans les sections précédentes, nous avons vu les **triggers classiques** qui réagissent aux modifications de **données** (INSERT, UPDATE, DELETE). Mais que se passe-t-il quand vous voulez surveiller les modifications de la **structure** de votre base de données ?

C'est là qu'interviennent les **Event Triggers** (déclencheurs d'événements).

### Rappel : Triggers classiques vs Event Triggers

| Aspect | Trigger Classique | Event Trigger |
|--------|-------------------|---------------|
| **Surveille** | Données (DML) | Structure (DDL) |
| **Événements** | INSERT, UPDATE, DELETE | CREATE, ALTER, DROP, etc. |
| **Associé à** | Une table spécifique | Toute la base de données |
| **Exemples d'utilisation** | Audit de modifications, validation métier | Audit de changements de schéma, protection |

### Analogie simple

Imaginez un bâtiment :
- **Trigger classique** = Caméra de surveillance dans une pièce (surveille ce qui se passe dans cette pièce)  
- **Event Trigger** = Alarme qui se déclenche quand on modifie la structure du bâtiment (abattre un mur, ajouter une porte, etc.)

---

## 1. Qu'est-ce qu'un Event Trigger ?

### 1.1. Définition

Un **Event Trigger** est une fonction qui s'exécute automatiquement lorsqu'un **événement DDL** (Data Definition Language) se produit dans la base de données.

**DDL** = Commandes qui modifient la structure :
- `CREATE TABLE`, `CREATE INDEX`  
- `ALTER TABLE`, `ALTER COLUMN`  
- `DROP TABLE`, `DROP FUNCTION`  
- `GRANT`, `REVOKE`
- Et bien d'autres...

### 1.2. Pourquoi utiliser des Event Triggers ?

- ✅ **Audit** : Enregistrer qui fait quoi et quand  
- ✅ **Protection** : Empêcher certaines modifications dangereuses  
- ✅ **Conformité** : Appliquer des standards de nommage  
- ✅ **Notification** : Alerter les administrateurs de changements  
- ✅ **Documentation automatique** : Maintenir un historique des changements

---

## 2. Les Événements DDL Disponibles

PostgreSQL propose quatre moments clés pour intercepter les commandes DDL :

### 2.1. ddl_command_start

Se déclenche **avant** l'exécution d'une commande DDL.

```
┌─────────────────────────────────────┐
│ Utilisateur tape : DROP TABLE test  │
├─────────────────────────────────────┤
│ 1. Event Trigger (ddl_command_start)│  ← On peut ANNULER ici
│ 2. Exécution de DROP TABLE          │
│ 3. Event Trigger (ddl_command_end)  │
└─────────────────────────────────────┘
```

**Utilisation typique** : Empêcher certaines opérations

### 2.2. ddl_command_end

Se déclenche **après** l'exécution réussie d'une commande DDL.

**Utilisation typique** : Enregistrer l'événement dans un journal

### 2.3. table_rewrite

Se déclenche quand une table doit être complètement réécrite (ALTER TABLE avec changement de type par exemple).

**Utilisation typique** : Alerter sur les opérations potentiellement longues

### 2.4. sql_drop

Se déclenche **juste avant** qu'un objet soit supprimé.

**Utilisation typique** : Empêcher la suppression accidentelle d'objets critiques

---

## 3. Créer un Event Trigger Simple

### 3.1. Syntaxe de base

```sql
-- Étape 1 : Créer la fonction qui sera appelée
CREATE OR REPLACE FUNCTION ma_fonction_event()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Code à exécuter
    RAISE NOTICE 'Un événement DDL s'est produit !';
END;
$$;

-- Étape 2 : Créer l'Event Trigger
CREATE EVENT TRIGGER nom_event_trigger  
ON ddl_command_start  -- ou ddl_command_end, sql_drop, table_rewrite  
EXECUTE FUNCTION ma_fonction_event();  
```

**Points importants** :
- La fonction doit retourner le type `event_trigger`
- Pas de paramètres dans la fonction
- On utilise `CREATE EVENT TRIGGER` (pas juste `CREATE TRIGGER`)

### 3.2. Premier exemple : Logger toutes les commandes DDL

```sql
-- Table pour stocker l'historique
CREATE TABLE ddl_history (
    id SERIAL PRIMARY KEY,
    event_time TIMESTAMP DEFAULT NOW(),
    event_type TEXT,
    command_tag TEXT,
    object_type TEXT,
    object_identity TEXT,
    username TEXT
);

-- Fonction de logging
CREATE OR REPLACE FUNCTION log_ddl_command()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
BEGIN
    -- Insérer dans l'historique
    INSERT INTO ddl_history (
        event_type,
        command_tag,
        username
    ) VALUES (
        TG_EVENT,           -- Type d'événement (ddl_command_end, etc.)
        TG_TAG,             -- Commande SQL (CREATE TABLE, DROP INDEX, etc.)
        current_user        -- Utilisateur qui a lancé la commande
    );

    RAISE NOTICE 'Commande DDL loggée : % par %', TG_TAG, current_user;
END;
$$;

-- Event Trigger
CREATE EVENT TRIGGER log_all_ddl  
ON ddl_command_end  
EXECUTE FUNCTION log_ddl_command();  
```

**Test** :
```sql
-- Créer une table
CREATE TABLE test_event (id INTEGER);
-- NOTICE:  Commande DDL loggée : CREATE TABLE par postgres

-- Vérifier l'historique
SELECT * FROM ddl_history;
-- id | event_time          | event_type       | command_tag  | username
-- 1  | 2025-01-15 10:30:00 | ddl_command_end  | CREATE TABLE | postgres

-- Supprimer la table
DROP TABLE test_event;
-- NOTICE:  Commande DDL loggée : DROP TABLE par postgres
```

---

## 4. Variables Spéciales dans les Event Triggers

PostgreSQL fournit des variables spéciales pour obtenir des informations sur l'événement :

### 4.1. TG_EVENT

Le type d'événement qui a déclenché l'Event Trigger.

**Valeurs possibles** :
- `ddl_command_start`  
- `ddl_command_end`  
- `sql_drop`  
- `table_rewrite`

```sql
CREATE OR REPLACE FUNCTION afficher_event_type()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RAISE NOTICE 'Type d''événement : %', TG_EVENT;
END;
$$;
```

### 4.2. TG_TAG

La commande SQL qui a été exécutée.

**Exemples de valeurs** :
- `CREATE TABLE`  
- `ALTER TABLE`  
- `DROP INDEX`  
- `CREATE FUNCTION`  
- `GRANT`

```sql
CREATE OR REPLACE FUNCTION afficher_command_tag()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RAISE NOTICE 'Commande : %', TG_TAG;
END;
$$;
```

### 4.3. pg_event_trigger_ddl_commands()

Fonction spéciale qui retourne des détails sur les objets créés (disponible uniquement dans `ddl_command_end`).

```sql
CREATE OR REPLACE FUNCTION details_objets_crees()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
BEGIN
    -- Récupérer les détails des objets créés/modifiés
    FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        RAISE NOTICE 'Objet créé : type=%, identité=%',
                     obj.object_type, obj.object_identity;
    END LOOP;
END;
$$;

CREATE EVENT TRIGGER show_created_objects  
ON ddl_command_end  
EXECUTE FUNCTION details_objets_crees();  
```

**Test** :
```sql
CREATE TABLE ma_table (id INTEGER);
-- NOTICE:  Objet créé : type=table, identité=public.ma_table
```

### 4.4. pg_event_trigger_dropped_objects()

Fonction qui retourne les objets qui vont être supprimés (disponible uniquement dans `sql_drop`).

```sql
CREATE OR REPLACE FUNCTION details_objets_supprimes()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_dropped_objects()
    LOOP
        RAISE NOTICE 'Objet supprimé : type=%, identité=%',
                     obj.object_type, obj.object_identity;
    END LOOP;
END;
$$;

CREATE EVENT TRIGGER show_dropped_objects  
ON sql_drop  
EXECUTE FUNCTION details_objets_supprimes();  
```

**Test** :
```sql
DROP TABLE ma_table;
-- NOTICE:  Objet supprimé : type=table, identité=public.ma_table
```

---

## 5. Empêcher Certaines Opérations

### 5.1. Protéger contre la suppression de tables

```sql
CREATE OR REPLACE FUNCTION prevent_table_drop()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
BEGIN
    -- Vérifier si on essaie de supprimer une table
    IF TG_TAG = 'DROP TABLE' THEN
        FOR obj IN SELECT * FROM pg_event_trigger_dropped_objects()
        LOOP
            -- Bloquer la suppression de certaines tables critiques
            IF obj.object_identity LIKE '%clients%' OR
               obj.object_identity LIKE '%commandes%' THEN
                RAISE EXCEPTION 'Suppression de la table % interdite ! Table critique.',
                               obj.object_identity;
            END IF;
        END LOOP;
    END IF;
END;
$$;

CREATE EVENT TRIGGER no_critical_table_drop  
ON sql_drop  
EXECUTE FUNCTION prevent_table_drop();  
```

**Test** :
```sql
-- Table normale : OK
CREATE TABLE test_temp (id INTEGER);  
DROP TABLE test_temp;  -- ✅ Fonctionne  

-- Table critique : BLOQUÉ
CREATE TABLE clients (id INTEGER);  
DROP TABLE clients;  
-- ❌ ERREUR : Suppression de la table public.clients interdite ! Table critique.
```

### 5.2. Empêcher les modifications le week-end

```sql
CREATE OR REPLACE FUNCTION no_ddl_on_weekend()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Vérifier si on est en week-end
    IF EXTRACT(DOW FROM NOW()) IN (0, 6) THEN  -- 0 = Dimanche, 6 = Samedi
        RAISE EXCEPTION 'Les modifications de structure sont interdites le week-end ! Commande : %', TG_TAG;
    END IF;
END;
$$;

CREATE EVENT TRIGGER block_weekend_ddl  
ON ddl_command_start  
EXECUTE FUNCTION no_ddl_on_weekend();  
```

### 5.3. Appliquer un standard de nommage

```sql
CREATE OR REPLACE FUNCTION enforce_naming_convention()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
BEGIN
    -- Vérifier uniquement pour la création de tables
    IF TG_TAG = 'CREATE TABLE' THEN
        FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
        LOOP
            -- Les tables doivent commencer par 'tbl_'
            IF obj.object_identity NOT LIKE 'public.tbl_%' THEN
                RAISE EXCEPTION 'Nom de table invalide : %. Les tables doivent commencer par "tbl_"',
                               obj.object_identity;
            END IF;
        END LOOP;
    END IF;
END;
$$;

CREATE EVENT TRIGGER enforce_table_naming  
ON ddl_command_end  
EXECUTE FUNCTION enforce_naming_convention();  
```

**Test** :
```sql
-- ❌ Nom invalide
CREATE TABLE users (id INTEGER);
-- ERREUR : Nom de table invalide : public.users. Les tables doivent commencer par "tbl_"

-- ✅ Nom valide
CREATE TABLE tbl_users (id INTEGER);
-- Succès !
```

---

## 6. Cas d'Usage Pratiques

### 6.1. Audit complet des changements de schéma

```sql
-- Table d'audit enrichie
CREATE TABLE schema_audit (
    audit_id SERIAL PRIMARY KEY,
    event_time TIMESTAMP DEFAULT NOW(),
    event_type TEXT,
    command_tag TEXT,
    object_type TEXT,
    object_identity TEXT,
    schema_name TEXT,
    username TEXT,
    client_address INET,
    application_name TEXT,
    query_text TEXT
);

-- Fonction d'audit complète
CREATE OR REPLACE FUNCTION audit_schema_changes()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
    query TEXT;
BEGIN
    -- Récupérer la requête originale
    query := current_query();

    -- Pour les objets créés/modifiés
    IF TG_EVENT = 'ddl_command_end' THEN
        FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
        LOOP
            INSERT INTO schema_audit (
                event_type,
                command_tag,
                object_type,
                object_identity,
                schema_name,
                username,
                client_address,
                application_name,
                query_text
            ) VALUES (
                TG_EVENT,
                TG_TAG,
                obj.object_type,
                obj.object_identity,
                obj.schema_name,
                current_user,
                inet_client_addr(),
                current_setting('application_name'),
                query
            );
        END LOOP;
    END IF;

    -- Pour les objets supprimés
    IF TG_EVENT = 'sql_drop' THEN
        FOR obj IN SELECT * FROM pg_event_trigger_dropped_objects()
        LOOP
            INSERT INTO schema_audit (
                event_type,
                command_tag,
                object_type,
                object_identity,
                schema_name,
                username,
                client_address,
                application_name,
                query_text
            ) VALUES (
                TG_EVENT,
                TG_TAG,
                obj.object_type,
                obj.object_identity,
                obj.schema_name,
                current_user,
                inet_client_addr(),
                current_setting('application_name'),
                query
            );
        END LOOP;
    END IF;
END;
$$;

CREATE EVENT TRIGGER full_schema_audit  
ON ddl_command_end  
EXECUTE FUNCTION audit_schema_changes();  

CREATE EVENT TRIGGER full_schema_audit_drop  
ON sql_drop  
EXECUTE FUNCTION audit_schema_changes();  
```

**Consultation de l'audit** :
```sql
-- Voir tous les changements récents
SELECT
    event_time,
    command_tag,
    object_identity,
    username
FROM schema_audit  
ORDER BY event_time DESC  
LIMIT 10;  

-- Voir qui a supprimé quoi
SELECT
    event_time,
    object_identity,
    username,
    query_text
FROM schema_audit  
WHERE command_tag LIKE 'DROP%'  
ORDER BY event_time DESC;  
```

### 6.2. Notification Slack/Email lors de changements

```sql
CREATE OR REPLACE FUNCTION notify_ddl_changes()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    message TEXT;
    obj RECORD;
BEGIN
    -- Construire le message
    message := format(
        '🔔 Alerte DDL sur %s
Utilisateur : %s  
Commande : %s  
Heure : %s',  
        current_database(),
        current_user,
        TG_TAG,
        NOW()
    );

    -- Pour les créations/modifications
    IF TG_EVENT = 'ddl_command_end' THEN
        FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
        LOOP
            message := message || format('
Objet : %s (%s)', obj.object_identity, obj.object_type);
        END LOOP;
    END IF;

    -- Envoyer la notification (via NOTIFY)
    PERFORM pg_notify('ddl_alerts', message);

    -- Dans un système réel, on pourrait :
    -- - Appeler une API externe (avec pl/python ou pl/curl)
    -- - Écrire dans une table de notifications
    -- - Envoyer un email (avec une extension)

    RAISE NOTICE '%', message;
END;
$$;

CREATE EVENT TRIGGER notify_on_ddl  
ON ddl_command_end  
EXECUTE FUNCTION notify_ddl_changes();  
```

### 6.3. Protection en production

```sql
CREATE OR REPLACE FUNCTION protect_production()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    allowed_users TEXT[] := ARRAY['dba_user', 'migration_user'];
BEGIN
    -- Vérifier l'environnement (via une variable de configuration)
    IF current_setting('app.environment', true) = 'production' THEN

        -- Seuls certains utilisateurs peuvent faire du DDL en production
        IF NOT (current_user = ANY(allowed_users)) THEN
            RAISE EXCEPTION 'DDL interdit en production pour l''utilisateur % ! Commande : %',
                           current_user, TG_TAG;
        END IF;

        -- Bloquer certaines commandes dangereuses même pour les utilisateurs autorisés
        IF TG_TAG IN ('DROP DATABASE', 'DROP SCHEMA') THEN
            RAISE EXCEPTION 'Commande % interdite même pour les administrateurs en production !',
                           TG_TAG;
        END IF;

        -- Logger toutes les opérations autorisées
        RAISE WARNING 'DDL en production par % : %', current_user, TG_TAG;
    END IF;
END;
$$;

CREATE EVENT TRIGGER production_protection  
ON ddl_command_start  
EXECUTE FUNCTION protect_production();  
```

**Configuration** :
```sql
-- En développement
ALTER DATABASE mydb SET app.environment = 'development';

-- En production
ALTER DATABASE mydb SET app.environment = 'production';
```

---

## 7. Filtrer les Event Triggers avec WHEN

Vous pouvez limiter quand un Event Trigger se déclenche avec une clause `WHEN` :

### 7.1. Filtrer par type de commande

```sql
-- Se déclenche uniquement pour CREATE TABLE et DROP TABLE
CREATE EVENT TRIGGER audit_table_changes  
ON ddl_command_end  
WHEN TAG IN ('CREATE TABLE', 'DROP TABLE')  
EXECUTE FUNCTION log_ddl_command();  
```

### 7.2. Filtrer par type d'objet

```sql
-- Se déclenche uniquement pour les modifications de fonctions
CREATE EVENT TRIGGER audit_function_changes  
ON ddl_command_end  
WHEN TAG IN ('CREATE FUNCTION', 'ALTER FUNCTION', 'DROP FUNCTION')  
EXECUTE FUNCTION log_ddl_command();  
```

### 7.3. Exemple complet avec filtres

```sql
-- Event Trigger spécifique pour les tables
CREATE EVENT TRIGGER audit_tables_only  
ON ddl_command_end  
WHEN TAG IN ('CREATE TABLE', 'ALTER TABLE', 'DROP TABLE')  
EXECUTE FUNCTION audit_table_operations();  

-- Event Trigger spécifique pour les index
CREATE EVENT TRIGGER audit_indexes_only  
ON ddl_command_end  
WHEN TAG IN ('CREATE INDEX', 'DROP INDEX', 'REINDEX')  
EXECUTE FUNCTION audit_index_operations();  

-- Event Trigger pour tout le reste
CREATE EVENT TRIGGER audit_other_ddl  
ON ddl_command_end  
EXECUTE FUNCTION audit_other_operations();  
```

---

## 8. Gérer les Event Triggers

### 8.1. Lister les Event Triggers

```sql
-- Voir tous les Event Triggers
SELECT
    evtname AS trigger_name,
    evtevent AS event,
    evtfoid::regproc AS function_name,
    evtenabled AS enabled
FROM pg_event_trigger  
ORDER BY evtname;  
```

**Avec psql** :
```sql
\dy  -- Liste tous les Event Triggers
```

### 8.2. Désactiver temporairement un Event Trigger

```sql
-- Désactiver
ALTER EVENT TRIGGER nom_event_trigger DISABLE;

-- Réactiver
ALTER EVENT TRIGGER nom_event_trigger ENABLE;
```

**Exemple** :
```sql
-- Avant une grosse migration
ALTER EVENT TRIGGER audit_ddl DISABLE;

-- Faire la migration
CREATE TABLE ...;  
ALTER TABLE ...;  
-- Pas de logging pendant cette période

-- Réactiver après
ALTER EVENT TRIGGER audit_ddl ENABLE;
```

### 8.3. Supprimer un Event Trigger

```sql
DROP EVENT TRIGGER nom_event_trigger;
```

### 8.4. Modifier un Event Trigger

Pour modifier un Event Trigger, vous devez :
1. Modifier la fonction associée  
2. Ou supprimer et recréer l'Event Trigger

```sql
-- Option 1 : Modifier la fonction
CREATE OR REPLACE FUNCTION ma_fonction_event()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Nouvelle logique
END;
$$;
-- L'Event Trigger utilise automatiquement la nouvelle version

-- Option 2 : Recréer l'Event Trigger
DROP EVENT TRIGGER mon_event_trigger;  
CREATE EVENT TRIGGER mon_event_trigger  
ON ddl_command_end  
WHEN TAG IN ('CREATE TABLE')  
EXECUTE FUNCTION ma_fonction_event();  
```

---

## 9. Limitations et Considérations

### 9.1. Permissions requises

Seul un **superutilisateur** peut créer des Event Triggers.

```sql
-- ❌ Utilisateur normal
CREATE EVENT TRIGGER test ON ddl_command_end EXECUTE FUNCTION f();
-- ERREUR : permission denied to create event trigger "test"
-- CONSEIL : Only superusers can create event triggers.

-- ✅ Superutilisateur
ALTER USER mon_user WITH SUPERUSER;
-- Maintenant OK
```

### 9.2. Event Triggers et transactions

Les Event Triggers s'exécutent **dans la même transaction** que la commande DDL.

```sql
BEGIN;
    CREATE TABLE test (id INTEGER);
    -- L'Event Trigger s'exécute ici
    -- Si l'Event Trigger fait RAISE EXCEPTION, tout est annulé
ROLLBACK;  -- La table ET le log de l'Event Trigger sont annulés
```

### 9.3. Pas d'Event Trigger sur Event Trigger

Un Event Trigger **ne peut pas** déclencher d'autres Event Triggers pour éviter les boucles infinies.

```sql
-- Event Trigger 1 crée une table de log
-- Cette création NE déclenche PAS Event Trigger 2
-- (Protection automatique contre les récursions)
```

### 9.4. Performance

Les Event Triggers ajoutent un overhead à chaque commande DDL.

**Recommandations** :
- ✅ Gardez les fonctions rapides  
- ✅ Évitez les opérations complexes  
- ✅ Désactivez les Event Triggers pendant les grosses migrations

### 9.5. Commandes non interceptables

Certaines commandes ne déclenchent **pas** d'Event Triggers :
- Commandes de transaction (`BEGIN`, `COMMIT`, `ROLLBACK`)
- Commandes sur les Event Triggers eux-mêmes
- Certaines commandes système

---

## 10. Bonnes Pratiques

### ✅ Pratique #1 : Nommer clairement les Event Triggers

```sql
-- ✅ BON : Nom descriptif
CREATE EVENT TRIGGER audit_schema_changes_for_compliance  
ON ddl_command_end  
EXECUTE FUNCTION audit_ddl();  

-- ❌ ÉVITER : Nom vague
CREATE EVENT TRIGGER et1  
ON ddl_command_end  
EXECUTE FUNCTION f();  
```

### ✅ Pratique #2 : Documenter la raison d'être

```sql
CREATE EVENT TRIGGER prevent_weekend_changes  
ON ddl_command_start  
EXECUTE FUNCTION no_ddl_on_weekend();  

COMMENT ON EVENT TRIGGER prevent_weekend_changes IS
'Empêche toute modification de schéma le week-end pour respecter
la politique de gel des changements pendant les périodes de faible activité.  
Créé le : 2025-01-15  
Responsable : Équipe DBA';  
```

### ✅ Pratique #3 : Utiliser des filtres WHEN

```sql
-- ✅ BON : Filtre pour éviter de traiter tous les événements
CREATE EVENT TRIGGER audit_critical_operations  
ON ddl_command_end  
WHEN TAG IN ('DROP TABLE', 'DROP DATABASE', 'ALTER TABLE')  
EXECUTE FUNCTION audit_critical_ddl();  

-- ❌ MOINS BON : Filtre dans la fonction (plus lent)
CREATE EVENT TRIGGER audit_all  
ON ddl_command_end  
EXECUTE FUNCTION audit_with_internal_filter();  -- Vérifie TG_TAG dans la fonction  
```

### ✅ Pratique #4 : Prévoir une désactivation facile

```sql
-- Créer une fonction de contrôle
CREATE OR REPLACE FUNCTION should_audit_ddl()  
RETURNS BOOLEAN  
LANGUAGE SQL  
AS $$  
    SELECT current_setting('app.audit_ddl', true)::BOOLEAN;
$$;

-- L'utiliser dans l'Event Trigger
CREATE OR REPLACE FUNCTION audit_ddl_with_control()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    -- Vérifier si l'audit est activé
    IF NOT should_audit_ddl() THEN
        RETURN;
    END IF;

    -- Logique d'audit
    INSERT INTO ddl_history ...;
END;
$$;

-- Désactiver l'audit temporairement sans toucher à l'Event Trigger
ALTER DATABASE mydb SET app.audit_ddl = false;
-- Réactiver
ALTER DATABASE mydb SET app.audit_ddl = true;
```

### ✅ Pratique #5 : Tester en développement d'abord

```sql
-- Toujours tester sur une base de développement
-- avant de déployer en production

-- En dev : Version permissive avec seulement du logging
CREATE EVENT TRIGGER dev_audit  
ON ddl_command_end  
EXECUTE FUNCTION log_ddl();  

-- En prod : Version stricte avec blocages
CREATE EVENT TRIGGER prod_protection  
ON ddl_command_start  
EXECUTE FUNCTION prevent_dangerous_ddl();  
```

---

## 11. Exemples Avancés

### 11.1. Générer automatiquement des index

```sql
CREATE OR REPLACE FUNCTION auto_create_indexes()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
    has_id_column BOOLEAN;
BEGIN
    -- Uniquement pour CREATE TABLE
    IF TG_TAG = 'CREATE TABLE' THEN
        FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
        LOOP
            -- Vérifier si la table a une colonne 'id'
            SELECT EXISTS (
                SELECT 1
                FROM information_schema.columns
                WHERE table_name = split_part(obj.object_identity, '.', 2)
                  AND column_name = 'id'
            ) INTO has_id_column;

            -- Créer automatiquement un index sur 'id' si elle existe
            IF has_id_column THEN
                EXECUTE format(
                    'CREATE INDEX IF NOT EXISTS idx_%s_id ON %s (id)',
                    split_part(obj.object_identity, '.', 2),
                    obj.object_identity
                );
                RAISE NOTICE 'Index créé automatiquement sur %.id', obj.object_identity;
            END IF;
        END LOOP;
    END IF;
END;
$$;

CREATE EVENT TRIGGER auto_index_creation  
ON ddl_command_end  
WHEN TAG IN ('CREATE TABLE')  
EXECUTE FUNCTION auto_create_indexes();  
```

**Test** :
```sql
CREATE TABLE commandes (
    id INTEGER PRIMARY KEY,
    montant NUMERIC
);
-- NOTICE:  Index créé automatiquement sur public.commandes.id
```

### 11.2. Versioning automatique des fonctions

```sql
-- Table de versions
CREATE TABLE function_versions (
    version_id SERIAL PRIMARY KEY,
    function_name TEXT,
    function_definition TEXT,
    created_by TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION version_functions()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
    func_def TEXT;
BEGIN
    IF TG_TAG IN ('CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION') THEN
        FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
        LOOP
            -- Récupérer la définition de la fonction
            SELECT pg_get_functiondef(obj.objid) INTO func_def;

            -- Enregistrer la version
            INSERT INTO function_versions (
                function_name,
                function_definition,
                created_by
            ) VALUES (
                obj.object_identity,
                func_def,
                current_user
            );

            RAISE NOTICE 'Version de fonction enregistrée : %', obj.object_identity;
        END LOOP;
    END IF;
END;
$$;

CREATE EVENT TRIGGER track_function_versions  
ON ddl_command_end  
WHEN TAG IN ('CREATE FUNCTION', 'CREATE OR REPLACE FUNCTION')  
EXECUTE FUNCTION version_functions();  
```

### 11.3. Alertes sur opérations longues

```sql
CREATE OR REPLACE FUNCTION alert_table_rewrite()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    obj RECORD;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_table_rewrite_oid()
    LOOP
        RAISE WARNING 'ATTENTION : La table % va être complètement réécrite.
Cette opération peut être très longue et verrouiller la table !',
                      obj.table_name;
    END LOOP;
END;
$$;

CREATE EVENT TRIGGER warn_on_table_rewrite  
ON table_rewrite  
EXECUTE FUNCTION alert_table_rewrite();  
```

**Test** :
```sql
-- Créer une table avec des données
CREATE TABLE test (id INTEGER, data TEXT);  
INSERT INTO test SELECT i, 'data' FROM generate_series(1, 1000) i;  

-- Changer le type de colonne (force une réécriture)
ALTER TABLE test ALTER COLUMN id TYPE BIGINT;
-- WARNING:  ATTENTION : La table test va être complètement réécrite...
```

---

## 12. Debugging et Troubleshooting

### 12.1. Voir ce qui déclenche les Event Triggers

```sql
-- Ajouter du logging détaillé
CREATE OR REPLACE FUNCTION debug_event_trigger()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    RAISE NOTICE '─────────────────────────────────────';
    RAISE NOTICE 'Event Trigger déclenché !';
    RAISE NOTICE 'TG_EVENT: %', TG_EVENT;
    RAISE NOTICE 'TG_TAG: %', TG_TAG;
    RAISE NOTICE 'Utilisateur: %', current_user;
    RAISE NOTICE 'Base: %', current_database();
    RAISE NOTICE 'Requête: %', current_query();
    RAISE NOTICE '─────────────────────────────────────';
END;
$$;
```

### 12.2. Gérer les erreurs dans les Event Triggers

```sql
CREATE OR REPLACE FUNCTION safe_event_trigger()  
RETURNS event_trigger  
LANGUAGE plpgsql  
AS $$  
BEGIN  
    BEGIN
        -- Logique principale
        INSERT INTO audit_log ...;
    EXCEPTION
        WHEN OTHERS THEN
            -- Logger l'erreur mais ne pas bloquer la commande DDL
            RAISE WARNING 'Erreur dans Event Trigger : %', SQLERRM;
            -- Insérer dans une table d'erreurs
            INSERT INTO event_trigger_errors (error_message, event_time)
            VALUES (SQLERRM, NOW());
    END;
END;
$$;
```

---

## 13. Résumé Visuel

```
┌────────────────────────────────────────────────────────────┐
│              EVENT TRIGGERS : VUE D'ENSEMBLE               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ÉVÉNEMENTS DISPONIBLES :                                  │
│  ┌──────────────────────────────────────────────────┐      │
│  │ ddl_command_start  → AVANT la commande DDL       │      │
│  │ ddl_command_end    → APRÈS la commande DDL       │      │
│  │ sql_drop           → AVANT suppression d'objet   │      │
│  │ table_rewrite      → Réécriture de table         │      │
│  └──────────────────────────────────────────────────┘      │
│                                                            │
│  VARIABLES SPÉCIALES :                                     │
│  • TG_EVENT          → Type d'événement                    │
│  • TG_TAG            → Commande SQL                        │
│  • pg_event_trigger_ddl_commands()     → Objets créés      │
│  • pg_event_trigger_dropped_objects()  → Objets supprimés  │
│                                                            │
│  CAS D'USAGE :                                             │
│  ✅ Audit de changements de schéma                         │
│  ✅ Protection contre suppressions accidentelles           │
│  ✅ Application de standards de nommage                    │
│  ✅ Notifications automatiques                             │
│  ✅ Versionning de schéma                                  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 14. Checklist de Mise en Production

Avant de déployer des Event Triggers en production :

- [ ] Testé sur environnement de développement  
- [ ] Performance évaluée (pas de ralentissement)  
- [ ] Gestion d'erreurs robuste (ne bloque pas les DDL critiques)  
- [ ] Documentation complète (COMMENT ON)  
- [ ] Noms clairs et descriptifs  
- [ ] Clause WHEN pour filtrer si possible  
- [ ] Mécanisme de désactivation facile  
- [ ] Permissions vérifiées (superutilisateur)  
- [ ] Plan de rollback défini  
- [ ] Équipe informée des changements

---

## 15. Conclusion

Les **Event Triggers** sont un outil puissant pour :

1. **Surveiller** tous les changements de structure de votre base  
2. **Protéger** contre les opérations dangereuses  
3. **Auditer** qui fait quoi et quand  
4. **Automatiser** certaines tâches (création d'index, versionning)

**Points clés à retenir** :
- ✅ Les Event Triggers surveillent les commandes DDL (structure), pas DML (données)  
- ✅ Quatre événements : `ddl_command_start`, `ddl_command_end`, `sql_drop`, `table_rewrite`  
- ✅ Variables spéciales : `TG_EVENT`, `TG_TAG`, `pg_event_trigger_*_objects()`  
- ✅ Seuls les superutilisateurs peuvent créer des Event Triggers  
- ✅ Utilisez la clause `WHEN` pour filtrer les événements  
- ✅ Testez toujours en développement avant la production

**Recommandation** :
> Commencez simple avec un Event Trigger d'audit en lecture seule, puis ajoutez progressivement des protections selon vos besoins.

---

**🎓 Prochaines étapes dans le tutoriel :**
- 15.6. Gestion des exceptions (BEGIN...EXCEPTION...END)
- 15.7. Autres langages procéduraux (PL/Python, PL/Perl, PL/v8)
- 16. Administration, Configuration et Sécurité

**💡 Pour aller plus loin :**
- Documentation officielle : [Event Triggers](https://www.postgresql.org/docs/current/event-triggers.html)
- Expérimentez avec différents types d'événements
- Créez un système d'audit complet pour votre base de données

⏭️ [Gestion des exceptions (BEGIN...EXCEPTION...END)](/15-programmation-serveur/06-gestion-des-exceptions.md)
