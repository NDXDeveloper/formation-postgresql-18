🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.4. Triggers (BEFORE, AFTER, INSTEAD OF)

## Introduction

Un **trigger** (ou "déclencheur" en français) est un objet de base de données qui **réagit automatiquement** à certains événements survenant sur une table ou une vue. C'est comme un garde automatique qui surveille votre base de données et exécute des actions prédéfinies lorsque certaines conditions sont remplies.

### Analogie pour comprendre

Imaginez une maison intelligente avec des capteurs :

- **Événement déclencheur** : Quelqu'un ouvre la porte d'entrée
- **Trigger** : Le système de sécurité qui surveille la porte
- **Action automatique** : Allumer les lumières, désactiver l'alarme, enregistrer l'heure dans un journal

En PostgreSQL, c'est la même logique :
- **Événement déclencheur** : INSERT, UPDATE ou DELETE sur une table
- **Trigger** : Fonction qui surveille ces opérations
- **Action automatique** : Validation, calcul, journalisation, notification, etc.

### Pourquoi utiliser des triggers ?

Les triggers permettent de :

1. **Automatiser des tâches répétitives**
   - Calculer automatiquement des valeurs dérivées
   - Maintenir des horodatages à jour
   - Générer des identifiants ou codes automatiques

2. **Garantir l'intégrité des données**
   - Valider des règles métier complexes
   - Empêcher des modifications non autorisées
   - Maintenir la cohérence entre tables liées

3. **Auditer et tracer les modifications**
   - Enregistrer l'historique des changements
   - Logger qui a fait quoi et quand
   - Conserver les anciennes valeurs

4. **Synchroniser des données**
   - Mettre à jour des tables dépendantes
   - Maintenir des données dénormalisées à jour
   - Propager des modifications

5. **Déclencher des notifications**
   - Alerter d'événements critiques
   - Envoyer des messages à des systèmes externes
   - Publier des événements

### Exemple conceptuel simple

```sql
-- Sans trigger : l'utilisateur doit penser à tout
INSERT INTO commandes (client_id, produit_id, quantite, prix_unitaire)
VALUES (123, 456, 5, 99.99);

-- L'utilisateur doit aussi penser à :
UPDATE commandes SET montant_total = quantite * prix_unitaire WHERE id = ...;
UPDATE commandes SET date_creation = now() WHERE id = ...;
UPDATE clients SET nb_commandes = nb_commandes + 1 WHERE id = 123;
UPDATE produits SET stock = stock - 5 WHERE id = 456;

-- Avec trigger : tout est automatique !
INSERT INTO commandes (client_id, produit_id, quantite, prix_unitaire)
VALUES (123, 456, 5, 99.99);
-- Le trigger calcule montant_total
-- Le trigger ajoute date_creation
-- Le trigger met à jour nb_commandes du client
-- Le trigger décrémente le stock du produit
```

---

## Anatomie d'un trigger

Un trigger PostgreSQL est composé de **deux éléments distincts** :

### 1. La fonction trigger (la logique)

C'est une fonction PL/pgSQL spéciale qui contient le code à exécuter. Elle doit :
- Retourner le type `TRIGGER`
- Ne prendre aucun paramètre explicite
- Retourner NEW, OLD ou NULL

```sql
CREATE OR REPLACE FUNCTION ma_fonction_trigger()
RETURNS TRIGGER AS $$
BEGIN
  -- Logique métier ici
  RETURN NEW;  -- ou OLD, ou NULL
END;
$$ LANGUAGE plpgsql;
```

### 2. Le trigger lui-même (la configuration)

C'est la déclaration qui lie la fonction à un événement sur une table :

```sql
CREATE TRIGGER nom_du_trigger
  BEFORE INSERT ON ma_table
  FOR EACH ROW
  EXECUTE FUNCTION ma_fonction_trigger();
```

### Schéma de fonctionnement

```
┌─────────────────────────────────────────────────────────────┐
│                     Instruction SQL                         │
│              INSERT / UPDATE / DELETE                       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Événement    │
                  │  détecté      │
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Trigger      │
                  │  déclenché    │
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Fonction     │
                  │  exécutée     │
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Action       │
                  │  effectuée    │
                  └───────────────┘
```

---

## Les trois moments d'exécution : BEFORE, AFTER, INSTEAD OF

Les triggers peuvent s'exécuter à **trois moments différents** par rapport à l'opération SQL :

### 1. BEFORE (Avant)

Le trigger s'exécute **avant** que l'opération soit réellement effectuée dans la table.

#### Caractéristiques BEFORE

- ✅ Peut **modifier** les données avant insertion/mise à jour
- ✅ Peut **annuler** l'opération en retournant NULL
- ✅ Idéal pour la **validation** et la **transformation**
- ❌ Les données ne sont pas encore dans la table

#### Cas d'usage typiques BEFORE

1. **Validation avancée** : Vérifier des règles métier complexes
2. **Normalisation** : Mettre en forme les données (trim, lowercase, etc.)
3. **Calculs automatiques** : Remplir des colonnes calculées
4. **Valeurs par défaut** : Générer des valeurs automatiques
5. **Prévention** : Bloquer certaines opérations

#### Exemple BEFORE

```sql
CREATE OR REPLACE FUNCTION valider_email()
RETURNS TRIGGER AS $$
BEGIN
  -- Normaliser l'email
  NEW.email := LOWER(TRIM(NEW.email));

  -- Valider le format
  IF NEW.email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
    RAISE EXCEPTION 'Email invalide : %', NEW.email;
  END IF;

  RETURN NEW;  -- Continuer avec NEW (potentiellement modifié)
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_email
  BEFORE INSERT OR UPDATE ON utilisateurs
  FOR EACH ROW
  EXECUTE FUNCTION valider_email();
```

### 2. AFTER (Après)

Le trigger s'exécute **après** que l'opération ait été effectuée dans la table.

#### Caractéristiques AFTER

- ❌ Ne peut **pas modifier** les données de l'opération en cours
- ✅ Peut effectuer des actions sur **d'autres tables**
- ✅ Idéal pour l'**audit**, la **synchronisation** et les **notifications**
- ✅ Les données sont déjà dans la table

#### Cas d'usage typiques AFTER

1. **Audit** : Enregistrer les modifications dans une table d'historique
2. **Synchronisation** : Mettre à jour des tables liées
3. **Statistiques** : Maintenir des compteurs à jour
4. **Notifications** : Déclencher des alertes ou événements
5. **Journalisation** : Logger les opérations importantes

#### Exemple AFTER

```sql
CREATE OR REPLACE FUNCTION auditer_modification()
RETURNS TRIGGER AS $$
BEGIN
  -- Enregistrer dans la table d'audit
  INSERT INTO audit_log (
    table_name,
    operation,
    ancien_record,
    nouveau_record,
    user_name,
    timestamp
  ) VALUES (
    TG_TABLE_NAME,
    TG_OP,
    row_to_json(OLD),
    row_to_json(NEW),
    current_user,
    now()
  );

  -- Retourner NEW ou OLD (la valeur est ignorée en AFTER)
  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  ELSE
    RETURN NEW;
  END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_audit
  AFTER INSERT OR UPDATE OR DELETE ON produits
  FOR EACH ROW
  EXECUTE FUNCTION auditer_modification();
```

### 3. INSTEAD OF (À la place de)

Le trigger **remplace complètement** l'opération normale. Disponible **uniquement sur les vues**.

#### Caractéristiques INSTEAD OF

- ⚠️ **Uniquement pour les vues** (pas les tables)
- ✅ Permet de rendre des vues **modifiables**
- ✅ Définit **exactement** ce qui doit se passer
- ✅ Utile pour les vues complexes (jointures, agrégations)

#### Cas d'usage typiques INSTEAD OF

1. **Vues modifiables** : Permettre INSERT/UPDATE/DELETE sur vues complexes
2. **Abstraction** : Cacher la complexité du schéma physique
3. **Distribution** : Répartir les modifications sur plusieurs tables
4. **Validation complexe** : Appliquer une logique métier élaborée

#### Exemple INSTEAD OF

```sql
-- Vue qui joint deux tables
CREATE VIEW vue_employes_complet AS
SELECT
  e.id,
  e.nom,
  e.prenom,
  d.nom_departement,
  d.budget
FROM employes e
JOIN departements d ON e.departement_id = d.id;

-- Trigger INSTEAD OF pour permettre l'insertion via la vue
CREATE OR REPLACE FUNCTION inserer_employe_via_vue()
RETURNS TRIGGER AS $$
DECLARE
  dept_id INTEGER;
BEGIN
  -- Trouver ou créer le département
  SELECT id INTO dept_id
  FROM departements
  WHERE nom_departement = NEW.nom_departement;

  IF dept_id IS NULL THEN
    INSERT INTO departements (nom_departement)
    VALUES (NEW.nom_departement)
    RETURNING id INTO dept_id;
  END IF;

  -- Insérer l'employé
  INSERT INTO employes (nom, prenom, departement_id)
  VALUES (NEW.nom, NEW.prenom, dept_id);

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_insert_vue
  INSTEAD OF INSERT ON vue_employes_complet
  FOR EACH ROW
  EXECUTE FUNCTION inserer_employe_via_vue();

-- Maintenant on peut insérer directement dans la vue !
INSERT INTO vue_employes_complet (nom, prenom, nom_departement)
VALUES ('Dupont', 'Jean', 'Informatique');
```

---

## Comparaison BEFORE vs AFTER vs INSTEAD OF

### Tableau récapitulatif

| Critère | BEFORE | AFTER | INSTEAD OF |
|---------|--------|-------|------------|
| **Moment** | Avant l'opération | Après l'opération | Remplace l'opération |
| **Modifier les données** | ✅ Oui (NEW) | ❌ Non | ✅ Oui (contrôle total) |
| **Annuler l'opération** | ✅ Oui (RETURN NULL) | ❌ Non | ✅ Oui (ne rien faire) |
| **Accès aux données** | NEW/OLD selon opération | NEW/OLD selon opération | NEW/OLD selon opération |
| **Tables/Vues** | Tables uniquement | Tables uniquement | Vues uniquement |
| **Cas d'usage** | Validation, transformation | Audit, synchronisation | Vues complexes modifiables |
| **Performance** | Impact direct sur l'opération | Exécution additionnelle | Remplace l'opération |

### Diagramme de flux des triggers

```
OPÉRATION SQL (INSERT/UPDATE/DELETE)
         │
         ▼
    ┌─────────┐
    │ BEFORE  │ ◄── Peut modifier NEW
    │ Trigger │ ◄── Peut annuler (RETURN NULL)
    └────┬────┘
         │
         ▼
  ┌──────────────┐
  │  OPÉRATION   │ ◄── Écriture réelle dans la table
  │  EFFECTUÉE   │
  └──────┬───────┘
         │
         ▼
    ┌─────────┐
    │  AFTER  │ ◄── Ne peut pas modifier l'opération
    │ Trigger │ ◄── Peut agir sur d'autres tables
    └─────────┘
```

```
VUE COMPLEXE
         │
         ▼
  ┌──────────────┐
  │ INSTEAD OF   │ ◄── Remplace complètement l'opération
  │   Trigger    │ ◄── Contrôle total de ce qui se passe
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Logique     │ ◄── Votre code décide de tout
  │ personnalisée│
  └──────────────┘
```

---

## Les événements déclencheurs

Un trigger peut réagir à **quatre types d'opérations** :

### 1. INSERT

Déclenché lors de l'insertion de nouvelles lignes.

```sql
CREATE TRIGGER trigger_insert
  BEFORE INSERT ON ma_table
  FOR EACH ROW
  EXECUTE FUNCTION ma_fonction();
```

**Variables disponibles :**
- ✅ NEW (la ligne à insérer)
- ❌ OLD (n'existe pas encore)

### 2. UPDATE

Déclenché lors de la modification de lignes existantes.

```sql
CREATE TRIGGER trigger_update
  AFTER UPDATE ON ma_table
  FOR EACH ROW
  EXECUTE FUNCTION ma_fonction();
```

**Variables disponibles :**
- ✅ NEW (la ligne après modification)
- ✅ OLD (la ligne avant modification)

### 3. DELETE

Déclenché lors de la suppression de lignes.

```sql
CREATE TRIGGER trigger_delete
  BEFORE DELETE ON ma_table
  FOR EACH ROW
  EXECUTE FUNCTION ma_fonction();
```

**Variables disponibles :**
- ❌ NEW (n'existe plus)
- ✅ OLD (la ligne supprimée)

### 4. TRUNCATE

Déclenché lors d'un TRUNCATE de table (vidage complet).

```sql
CREATE TRIGGER trigger_truncate
  BEFORE TRUNCATE ON ma_table
  FOR EACH STATEMENT  -- Toujours STATEMENT pour TRUNCATE
  EXECUTE FUNCTION ma_fonction();
```

**Particularité :**
- ⚠️ Toujours de type STATEMENT (jamais ROW)
- ❌ Pas de NEW ni OLD (opération en masse)

### Combiner plusieurs événements

Un même trigger peut réagir à plusieurs événements :

```sql
-- Trigger qui réagit à INSERT, UPDATE et DELETE
CREATE TRIGGER trigger_multi
  AFTER INSERT OR UPDATE OR DELETE ON ma_table
  FOR EACH ROW
  EXECUTE FUNCTION ma_fonction();
```

Dans la fonction, utilisez `TG_OP` pour distinguer l'opération :

```sql
CREATE OR REPLACE FUNCTION ma_fonction()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    -- Logique spécifique à INSERT
    RAISE NOTICE 'Insertion détectée';
    RETURN NEW;

  ELSIF TG_OP = 'UPDATE' THEN
    -- Logique spécifique à UPDATE
    RAISE NOTICE 'Mise à jour détectée';
    RETURN NEW;

  ELSIF TG_OP = 'DELETE' THEN
    -- Logique spécifique à DELETE
    RAISE NOTICE 'Suppression détectée';
    RETURN OLD;
  END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## Syntaxe complète de création d'un trigger

```sql
CREATE [ OR REPLACE ] [ CONSTRAINT ] TRIGGER nom_trigger
    { BEFORE | AFTER | INSTEAD OF } { événement [ OR ... ] }
    ON nom_table
    [ FROM nom_table_référencée ]
    [ NOT DEFERRABLE | [ DEFERRABLE ] [ INITIALLY IMMEDIATE | INITIALLY DEFERRED ] ]
    [ REFERENCING { { OLD | NEW } TABLE [ AS ] nom_table_transition } [ ... ] ]
    [ FOR [ EACH ] { ROW | STATEMENT } ]
    [ WHEN ( condition ) ]
    EXECUTE { FUNCTION | PROCEDURE } nom_fonction ( arguments )
```

### Éléments de syntaxe expliqués

#### CREATE TRIGGER nom_trigger
Nom unique du trigger dans la table.

#### BEFORE | AFTER | INSTEAD OF
Le moment d'exécution (expliqué précédemment).

#### événement [ OR ... ]
- `INSERT` : Insertion
- `UPDATE [ OF colonne1, colonne2, ... ]` : Mise à jour (optionnellement sur certaines colonnes seulement)
- `DELETE` : Suppression
- `TRUNCATE` : Vidage de table

#### ON nom_table
La table (ou vue) sur laquelle le trigger est attaché.

#### FOR EACH ROW | FOR EACH STATEMENT
Le niveau de granularité :
- `FOR EACH ROW` : Une exécution par ligne affectée
- `FOR EACH STATEMENT` : Une seule exécution par instruction SQL

*(Détaillé dans les sections suivantes 15.4.1 et 15.4.2)*

#### WHEN ( condition )
Clause optionnelle pour exécuter conditionnellement le trigger.

```sql
CREATE TRIGGER trigger_prix_eleves
  BEFORE UPDATE ON produits
  FOR EACH ROW
  WHEN (NEW.prix > 1000)  -- Seulement si prix > 1000
  EXECUTE FUNCTION notifier_prix_eleve();
```

#### EXECUTE FUNCTION nom_fonction()
La fonction à exécuter (sans paramètres explicites).

---

## Exemples pratiques complets

### Exemple 1 : Horodatage automatique

**Besoin :** Maintenir automatiquement les colonnes `created_at` et `updated_at`.

```sql
-- Table avec colonnes d'horodatage
CREATE TABLE articles (
  id SERIAL PRIMARY KEY,
  titre TEXT NOT NULL,
  contenu TEXT,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

-- Fonction trigger
CREATE OR REPLACE FUNCTION gerer_timestamps()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    NEW.created_at := now();
    NEW.updated_at := now();
  ELSIF TG_OP = 'UPDATE' THEN
    NEW.created_at := OLD.created_at;  -- Ne pas changer created_at
    NEW.updated_at := now();            -- Mettre à jour updated_at
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Création du trigger
CREATE TRIGGER trigger_timestamps
  BEFORE INSERT OR UPDATE ON articles
  FOR EACH ROW
  EXECUTE FUNCTION gerer_timestamps();
```

**Utilisation :**
```sql
-- L'utilisateur n'a pas besoin de gérer les dates
INSERT INTO articles (titre, contenu)
VALUES ('Mon article', 'Contenu intéressant');

-- created_at et updated_at sont automatiquement remplis

UPDATE articles SET contenu = 'Contenu modifié' WHERE id = 1;
-- updated_at est automatiquement mis à jour
-- created_at reste inchangé
```

### Exemple 2 : Validation métier complexe

**Besoin :** Empêcher qu'un employé gagne plus que son manager.

```sql
CREATE TABLE employes (
  id SERIAL PRIMARY KEY,
  nom TEXT NOT NULL,
  salaire NUMERIC(10,2),
  manager_id INTEGER REFERENCES employes(id)
);

CREATE OR REPLACE FUNCTION valider_hierarchie_salaire()
RETURNS TRIGGER AS $$
DECLARE
  salaire_manager NUMERIC(10,2);
BEGIN
  -- Si l'employé a un manager
  IF NEW.manager_id IS NOT NULL THEN
    -- Récupérer le salaire du manager
    SELECT salaire INTO salaire_manager
    FROM employes
    WHERE id = NEW.manager_id;

    -- Validation
    IF NEW.salaire > salaire_manager THEN
      RAISE EXCEPTION
        'Le salaire de % (%) ne peut pas dépasser celui de son manager (%)',
        NEW.nom, NEW.salaire, salaire_manager;
    END IF;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_validation_salaire
  BEFORE INSERT OR UPDATE ON employes
  FOR EACH ROW
  EXECUTE FUNCTION valider_hierarchie_salaire();
```

### Exemple 3 : Synchronisation automatique

**Besoin :** Maintenir un compteur de commandes par client.

```sql
CREATE TABLE clients (
  id SERIAL PRIMARY KEY,
  nom TEXT,
  nb_commandes INTEGER DEFAULT 0
);

CREATE TABLE commandes (
  id SERIAL PRIMARY KEY,
  client_id INTEGER REFERENCES clients(id),
  montant NUMERIC(10,2)
);

CREATE OR REPLACE FUNCTION maj_compteur_commandes()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    -- Incrémenter le compteur
    UPDATE clients
    SET nb_commandes = nb_commandes + 1
    WHERE id = NEW.client_id;

  ELSIF TG_OP = 'DELETE' THEN
    -- Décrémenter le compteur
    UPDATE clients
    SET nb_commandes = nb_commandes - 1
    WHERE id = OLD.client_id;

  ELSIF TG_OP = 'UPDATE' THEN
    -- Si le client a changé
    IF OLD.client_id != NEW.client_id THEN
      -- Décrémenter l'ancien client
      UPDATE clients
      SET nb_commandes = nb_commandes - 1
      WHERE id = OLD.client_id;

      -- Incrémenter le nouveau client
      UPDATE clients
      SET nb_commandes = nb_commandes + 1
      WHERE id = NEW.client_id;
    END IF;
  END IF;

  -- Retourner la bonne valeur selon l'opération
  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  ELSE
    RETURN NEW;
  END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_sync_commandes
  AFTER INSERT OR UPDATE OR DELETE ON commandes
  FOR EACH ROW
  EXECUTE FUNCTION maj_compteur_commandes();
```

### Exemple 4 : Audit complet avec historique

**Besoin :** Enregistrer toutes les modifications dans une table d'audit.

```sql
-- Table d'audit générique
CREATE TABLE audit_trail (
  id SERIAL PRIMARY KEY,
  table_name TEXT,
  operation TEXT,
  old_values JSONB,
  new_values JSONB,
  changed_by TEXT DEFAULT current_user,
  changed_at TIMESTAMP DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_trail (
    table_name,
    operation,
    old_values,
    new_values
  ) VALUES (
    TG_TABLE_NAME,
    TG_OP,
    CASE WHEN TG_OP IN ('UPDATE', 'DELETE')
         THEN row_to_json(OLD)::jsonb
         ELSE NULL
    END,
    CASE WHEN TG_OP IN ('INSERT', 'UPDATE')
         THEN row_to_json(NEW)::jsonb
         ELSE NULL
    END
  );

  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  ELSE
    RETURN NEW;
  END IF;
END;
$$ LANGUAGE plpgsql;

-- Appliquer sur plusieurs tables
CREATE TRIGGER audit_produits
  AFTER INSERT OR UPDATE OR DELETE ON produits
  FOR EACH ROW
  EXECUTE FUNCTION audit_changes();

CREATE TRIGGER audit_employes
  AFTER INSERT OR UPDATE OR DELETE ON employes
  FOR EACH ROW
  EXECUTE FUNCTION audit_changes();
```

---

## Ordre d'exécution des triggers

Lorsque plusieurs triggers sont définis sur la même table, PostgreSQL les exécute dans un **ordre déterministe** :

### 1. Ordre selon le timing

```
1. BEFORE STATEMENT triggers
2. BEFORE ROW triggers (pour chaque ligne)
3. L'opération SQL elle-même
4. AFTER ROW triggers (pour chaque ligne)
5. AFTER STATEMENT triggers
```

### 2. Ordre alphabétique

Si plusieurs triggers du **même type** existent, ils s'exécutent par **ordre alphabétique** de leur nom.

```sql
-- Ces triggers s'exécutent dans cet ordre :
CREATE TRIGGER a_premier ...
CREATE TRIGGER b_deuxieme ...
CREATE TRIGGER c_troisieme ...
```

**Astuce :** Préfixez vos triggers avec des numéros pour contrôler l'ordre :

```sql
CREATE TRIGGER trigger_01_validation ...
CREATE TRIGGER trigger_02_transformation ...
CREATE TRIGGER trigger_03_audit ...
```

### 3. Exemple d'ordre complet

```sql
-- Table de test
CREATE TABLE test_ordre (
  id SERIAL PRIMARY KEY,
  valeur TEXT
);

-- Triggers pour observer l'ordre
CREATE OR REPLACE FUNCTION logger_ordre()
RETURNS TRIGGER AS $$
BEGIN
  RAISE NOTICE '% % % sur %',
    TG_NAME, TG_WHEN, TG_LEVEL, TG_TABLE_NAME;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER a_before_statement
  BEFORE INSERT ON test_ordre
  FOR EACH STATEMENT
  EXECUTE FUNCTION logger_ordre();

CREATE TRIGGER b_before_row
  BEFORE INSERT ON test_ordre
  FOR EACH ROW
  EXECUTE FUNCTION logger_ordre();

CREATE TRIGGER c_after_row
  AFTER INSERT ON test_ordre
  FOR EACH ROW
  EXECUTE FUNCTION logger_ordre();

CREATE TRIGGER d_after_statement
  AFTER INSERT ON test_ordre
  FOR EACH STATEMENT
  EXECUTE FUNCTION logger_ordre();
```

**Test :**
```sql
INSERT INTO test_ordre (valeur) VALUES ('test1'), ('test2');

-- Résultat dans les logs :
-- NOTICE: a_before_statement BEFORE STATEMENT sur test_ordre
-- NOTICE: b_before_row BEFORE ROW sur test_ordre
-- NOTICE: b_before_row BEFORE ROW sur test_ordre
-- NOTICE: c_after_row AFTER ROW sur test_ordre
-- NOTICE: c_after_row AFTER ROW sur test_ordre
-- NOTICE: d_after_statement AFTER STATEMENT sur test_ordre
```

---

## Gestion des triggers

### Lister les triggers d'une table

```sql
-- Méthode 1 : Vue système
SELECT
  tgname AS nom_trigger,
  tgtype & 1 = 1 AS for_each_row,
  CASE
    WHEN tgtype & 2 = 2 THEN 'BEFORE'
    WHEN tgtype & 64 = 64 THEN 'INSTEAD OF'
    ELSE 'AFTER'
  END AS timing,
  tgenabled AS statut
FROM pg_trigger
WHERE tgrelid = 'ma_table'::regclass
  AND tgisinternal = false;

-- Méthode 2 : Obtenir la définition complète
SELECT pg_get_triggerdef(oid) AS definition
FROM pg_trigger
WHERE tgrelid = 'ma_table'::regclass
  AND tgisinternal = false;
```

### Désactiver un trigger

```sql
-- Désactiver temporairement un trigger spécifique
ALTER TABLE ma_table DISABLE TRIGGER nom_trigger;

-- Désactiver tous les triggers de la table
ALTER TABLE ma_table DISABLE TRIGGER ALL;

-- Désactiver tous sauf les triggers système
ALTER TABLE ma_table DISABLE TRIGGER USER;
```

### Réactiver un trigger

```sql
-- Réactiver un trigger spécifique
ALTER TABLE ma_table ENABLE TRIGGER nom_trigger;

-- Réactiver tous les triggers
ALTER TABLE ma_table ENABLE TRIGGER ALL;
```

### Supprimer un trigger

```sql
DROP TRIGGER nom_trigger ON nom_table;

-- Avec IF EXISTS pour éviter les erreurs
DROP TRIGGER IF EXISTS nom_trigger ON nom_table;
```

### Modifier un trigger

PostgreSQL ne permet pas de modifier directement un trigger. Il faut le recréer :

```sql
-- Supprimer l'ancien
DROP TRIGGER IF EXISTS mon_trigger ON ma_table;

-- Créer le nouveau
CREATE TRIGGER mon_trigger
  BEFORE INSERT ON ma_table
  FOR EACH ROW
  EXECUTE FUNCTION ma_nouvelle_fonction();
```

Ou utiliser `CREATE OR REPLACE TRIGGER` (PostgreSQL 14+) :

```sql
CREATE OR REPLACE TRIGGER mon_trigger
  BEFORE INSERT ON ma_table
  FOR EACH ROW
  EXECUTE FUNCTION ma_nouvelle_fonction();
```

---

## Avantages et inconvénients des triggers

### ✅ Avantages

1. **Automatisation** : Les règles sont appliquées automatiquement, impossible de les oublier
2. **Centralisation** : La logique métier est dans la base, pas dispersée dans le code applicatif
3. **Cohérence** : Garantit que les règles sont appliquées quelle que soit l'application
4. **Performance** : Évite les aller-retours entre application et base
5. **Sécurité** : Les règles sont appliquées même si l'application est contournée

### ❌ Inconvénients

1. **Complexité cachée** : Les triggers sont "invisibles" depuis l'application
2. **Debugging difficile** : Erreurs parfois difficiles à tracer
3. **Performance** : Peut ralentir les opérations en masse
4. **Maintenance** : Logique métier fragmentée entre code et base
5. **Portabilité** : Dépendance forte à PostgreSQL

### Quand utiliser des triggers ?

| Scénario | Recommandation |
|----------|----------------|
| Règles d'intégrité critiques | ✅ Trigger recommandé |
| Calculs automatiques simples | ✅ Trigger recommandé |
| Audit et traçabilité | ✅ Trigger recommandé |
| Logique métier complexe | ⚠️ Évaluer (peut-être dans l'application) |
| Opérations lourdes | ❌ Éviter, utiliser des jobs asynchrones |
| Transformation de données volumineuses | ❌ Préférer des processus batch |

---

## Bonnes pratiques essentielles

### 1. Garder les triggers simples

```sql
-- ✅ Bon : Trigger simple et rapide
CREATE OR REPLACE FUNCTION maj_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ❌ Mauvais : Trigger complexe et lent
CREATE OR REPLACE FUNCTION trigger_complexe()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM heavy_computation();
  PERFORM call_external_api();
  PERFORM recursive_operation();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 2. Documenter abondamment

```sql
CREATE OR REPLACE FUNCTION ma_fonction()
RETURNS TRIGGER AS $$
/**
 * Trigger : Validation des prix
 * Table : produits
 * Timing : BEFORE INSERT OR UPDATE
 * Level : FOR EACH ROW
 *
 * Description :
 *   - Valide que le prix est positif
 *   - Applique une TVA de 20%
 *   - Arrondit à 2 décimales
 *
 * Auteur : Jean Dupont
 * Date : 2025-11-22
 */
BEGIN
  IF NEW.prix < 0 THEN
    RAISE EXCEPTION 'Prix négatif interdit';
  END IF;

  NEW.prix_ttc := ROUND(NEW.prix * 1.20, 2);

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 3. Utiliser des noms descriptifs

```sql
-- ✅ Bon : Nom descriptif
CREATE TRIGGER trigger_valider_email_avant_insertion
  BEFORE INSERT ON utilisateurs
  FOR EACH ROW
  EXECUTE FUNCTION valider_format_email();

-- ❌ Mauvais : Nom générique
CREATE TRIGGER t1
  BEFORE INSERT ON utilisateurs
  FOR EACH ROW
  EXECUTE FUNCTION f1();
```

### 4. Gérer les erreurs proprement

```sql
CREATE OR REPLACE FUNCTION fonction_securisee()
RETURNS TRIGGER AS $$
BEGIN
  -- Logique métier
  IF NEW.valeur < 0 THEN
    RAISE EXCEPTION 'Valeur négative interdite : %', NEW.valeur
      USING HINT = 'Utilisez une valeur positive',
            ERRCODE = '23514';  -- check_violation
  END IF;

  RETURN NEW;
EXCEPTION
  WHEN OTHERS THEN
    RAISE NOTICE 'Erreur dans trigger % : %', TG_NAME, SQLERRM;
    -- Décider si on propage l'erreur
    RAISE;
END;
$$ LANGUAGE plpgsql;
```

### 5. Éviter les boucles infinies

```sql
-- ⚠️ DANGER : Risque de boucle infinie
CREATE TRIGGER trigger_a
  AFTER UPDATE ON table_a
  FOR EACH ROW
  EXECUTE FUNCTION update_table_b();

CREATE TRIGGER trigger_b
  AFTER UPDATE ON table_b
  FOR EACH ROW
  EXECUTE FUNCTION update_table_a();

-- ✅ Solution : Utiliser des conditions
CREATE OR REPLACE FUNCTION update_avec_protection()
RETURNS TRIGGER AS $$
BEGIN
  -- Éviter la récursion avec une condition
  IF NOT (NEW.updated_by_trigger) THEN
    NEW.updated_by_trigger := TRUE;
    -- Faire la mise à jour
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## Résumé des concepts clés

### Les trois moments d'exécution

| Timing | Moment | Peut modifier NEW | Peut annuler | Tables/Vues |
|--------|--------|-------------------|--------------|-------------|
| **BEFORE** | Avant l'opération | ✅ Oui | ✅ Oui | Tables |
| **AFTER** | Après l'opération | ❌ Non | ❌ Non | Tables |
| **INSTEAD OF** | Remplace l'opération | ✅ Oui | ✅ Oui | Vues |

### Les quatre événements

- **INSERT** : Nouvelle ligne (NEW disponible)
- **UPDATE** : Modification (NEW et OLD disponibles)
- **DELETE** : Suppression (OLD disponible)
- **TRUNCATE** : Vidage complet (ni NEW ni OLD)

### Structure d'un trigger

1. **Fonction trigger** : Contient la logique (type TRIGGER)
2. **Trigger** : Lie la fonction à un événement sur une table

### Quand utiliser quoi ?

| Besoin | Utiliser |
|--------|----------|
| Valider avant insertion | BEFORE INSERT |
| Calculer une valeur | BEFORE INSERT/UPDATE |
| Empêcher une opération | BEFORE + RETURN NULL |
| Auditer les changements | AFTER INSERT/UPDATE/DELETE |
| Synchroniser des tables | AFTER INSERT/UPDATE/DELETE |
| Vue modifiable | INSTEAD OF sur vue |

---

## Conclusion

Les triggers sont des outils puissants qui permettent d'**automatiser** et de **garantir** l'intégrité de vos données. Bien utilisés, ils simplifient votre code applicatif et centralisent la logique métier critique.

**Principe directeur :** Un bon trigger est simple, bien documenté, et résout un problème d'intégrité ou d'automatisation qui ne peut pas être facilement géré au niveau applicatif.

Dans les sections suivantes, nous approfondirons :
- **15.4.1** : Triggers par ligne (FOR EACH ROW) - Exécution pour chaque enregistrement
- **15.4.2** : Triggers par instruction (FOR EACH STATEMENT) - Exécution une fois par opération
- **15.4.3** : Variables spéciales (NEW, OLD, TG_OP) - Accès aux données et métadonnées

---

**📚 Pour aller plus loin :**
- Section 15.1 : Fonctions SQL et PL/pgSQL
- Section 15.4.1 : Triggers par ligne (FOR EACH ROW)
- Section 15.4.2 : Triggers par instruction (FOR EACH STATEMENT)
- Section 15.4.3 : Variables spéciales (NEW, OLD, TG_OP)
- Section 15.5 : Event Triggers
- Documentation officielle PostgreSQL sur les triggers

⏭️ [Triggers par ligne (FOR EACH ROW)](/15-programmation-serveur/04.1-triggers-par-ligne.md)
