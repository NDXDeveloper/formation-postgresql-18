🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3. UPDATE et DELETE : Ciblage et précautions

## Introduction

Les commandes `UPDATE` et `DELETE` sont parmi les plus puissantes et les plus dangereuses du langage SQL. Elles permettent de modifier ou supprimer des données existantes dans une base de données. Utilisées correctement, elles sont indispensables. Utilisées sans précaution, elles peuvent causer des pertes de données catastrophiques.

Cette section vous guidera à travers l'utilisation sûre et efficace de ces commandes, en insistant particulièrement sur les bonnes pratiques et les pièges à éviter.

> **⚠️ AVERTISSEMENT CRITIQUE** : Une commande `UPDATE` ou `DELETE` sans clause `WHERE` affectera **TOUTES** les lignes de la table. Lisez attentivement cette section avant de manipuler des données en production.

---

## 6.3.1. La commande UPDATE : Modifier des données existantes

### Concept de base

La commande `UPDATE` permet de **modifier** les valeurs d'une ou plusieurs colonnes dans des lignes existantes d'une table. Contrairement à `INSERT` qui crée de nouvelles lignes, `UPDATE` transforme des données déjà présentes.

### Syntaxe générale

```sql
UPDATE nom_table  
SET colonne1 = nouvelle_valeur1,  
    colonne2 = nouvelle_valeur2,
    colonne3 = nouvelle_valeur3
WHERE condition;
```

**Décomposition** :
- `UPDATE nom_table` : Spécifie la table à modifier  
- `SET colonne = valeur` : Définit les nouvelles valeurs pour les colonnes  
- `WHERE condition` : **CRITIQUE** - Détermine quelles lignes seront modifiées

### Exemple simple

Prenons une table `employes` :

```sql
CREATE TABLE employes (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(255),
    salaire NUMERIC(10, 2),
    poste VARCHAR(100),
    date_embauche DATE
);

-- Données initiales
INSERT INTO employes (nom, prenom, email, salaire, poste, date_embauche) VALUES
    ('Dupont', 'Marie', 'marie.dupont@example.com', 45000, 'Développeur', '2023-01-15'),
    ('Martin', 'Pierre', 'pierre.martin@example.com', 52000, 'Chef de projet', '2022-06-01'),
    ('Durand', 'Sophie', 'sophie.durand@example.com', 48000, 'Développeur', '2023-03-20');
```

#### Modification d'une seule ligne

Augmenter le salaire de Marie Dupont :

```sql
UPDATE employes  
SET salaire = 47000  
WHERE nom = 'Dupont' AND prenom = 'Marie';  
```

**Résultat** : Seule Marie Dupont voit son salaire modifié (45000 → 47000).

#### Vérification

```sql
SELECT nom, prenom, salaire FROM employes WHERE nom = 'Dupont';
```

```text
  nom   | prenom | salaire
--------+--------+----------
 Dupont | Marie  | 47000.00
```

### UPDATE de plusieurs colonnes

Vous pouvez modifier plusieurs colonnes en une seule commande :

```sql
UPDATE employes  
SET  
    salaire = 55000,
    poste = 'Directeur technique',
    email = 'pierre.martin.pro@example.com'
WHERE id = 2;
```

**Résultat** : Pierre Martin reçoit trois modifications simultanées.

### UPDATE avec expressions

Les valeurs peuvent être calculées dynamiquement :

#### Augmentation de 10 %

```sql
UPDATE employes  
SET salaire = salaire * 1.10  
WHERE poste = 'Développeur';  
```

Cette requête augmente le salaire de tous les développeurs de 10 %.

#### Calculs plus complexes

```sql
-- Ajouter 5 ans d'ancienneté sous forme de bonus (100€ par année)
UPDATE employes  
SET salaire = salaire + (EXTRACT(YEAR FROM age(CURRENT_DATE, date_embauche)) * 100)  
WHERE date_embauche < '2023-01-01';  
```

### UPDATE avec fonctions

PostgreSQL permet d'utiliser des fonctions dans les UPDATE :

```sql
-- Mettre les emails en minuscules
UPDATE employes  
SET email = LOWER(email);  

-- Concaténer le prénom et le nom pour créer un username
UPDATE employes  
SET username = LOWER(prenom || '.' || nom);  

-- Extraire le domaine de l'email
UPDATE employes  
SET domaine_email = SPLIT_PART(email, '@', 2);  
```

### UPDATE conditionnel avec CASE

La clause `CASE` permet des modifications conditionnelles complexes :

```sql
-- Ajustement de salaire selon l'ancienneté
UPDATE employes  
SET salaire = CASE  
    WHEN EXTRACT(YEAR FROM age(CURRENT_DATE, date_embauche)) >= 5 THEN salaire * 1.15
    WHEN EXTRACT(YEAR FROM age(CURRENT_DATE, date_embauche)) >= 2 THEN salaire * 1.10
    WHEN EXTRACT(YEAR FROM age(CURRENT_DATE, date_embauche)) >= 1 THEN salaire * 1.05
    ELSE salaire
END;
```

**Explication** :
- 5+ ans d'ancienneté : +15 %
- 2-4 ans : +10 %
- 1-2 ans : +5 %
- Moins d'1 an : pas d'augmentation

---

## 6.3.2. La clause WHERE : Le garde-fou essentiel

### L'importance vitale du WHERE

La clause `WHERE` détermine **quelles lignes** seront affectées par l'UPDATE. C'est le mécanisme de sécurité le plus important.

### Le danger de l'oubli du WHERE

#### ❌ Erreur catastrophique (sans WHERE)

```sql
UPDATE employes  
SET salaire = 30000;  
```

**Résultat catastrophique** : **TOUS** les employés auront désormais un salaire de 30000, écrasant les valeurs précédentes !

```sql
SELECT nom, prenom, salaire FROM employes;
```

```text
  nom   | prenom | salaire
--------+--------+----------
 Dupont | Marie  | 30000.00  ← Avant : 47000
 Martin | Pierre | 30000.00  ← Avant : 55000
 Durand | Sophie | 30000.00  ← Avant : 48000
```

#### ✅ Correct (avec WHERE)

```sql
UPDATE employes  
SET salaire = 30000  
WHERE id = 1;  
```

**Résultat** : Seul l'employé avec `id = 1` est modifié.

### Stratégies de sécurité

#### 1. Toujours tester avec SELECT d'abord

**Règle d'or** : Avant chaque UPDATE, exécutez d'abord un SELECT avec la même clause WHERE :

```sql
-- ÉTAPE 1 : Tester avec SELECT
SELECT id, nom, prenom, salaire  
FROM employes  
WHERE poste = 'Développeur';  

-- Vérifier que les lignes affichées sont bien celles à modifier

-- ÉTAPE 2 : Si le résultat est correct, exécuter UPDATE
UPDATE employes  
SET salaire = salaire * 1.10  
WHERE poste = 'Développeur';  
```

#### 2. Utiliser des transactions

Encapsuler l'UPDATE dans une transaction permet de revenir en arrière si nécessaire :

```sql
-- Démarrer une transaction
BEGIN;

-- Effectuer l'UPDATE
UPDATE employes  
SET salaire = salaire * 1.10  
WHERE poste = 'Développeur';  

-- Vérifier le résultat
SELECT nom, prenom, salaire FROM employes WHERE poste = 'Développeur';

-- Si tout est OK :
COMMIT;

-- Si erreur, annuler :
-- ROLLBACK;
```

#### 3. Compter les lignes affectées

PostgreSQL vous indique combien de lignes ont été modifiées :

```sql
UPDATE employes  
SET salaire = salaire * 1.10  
WHERE poste = 'Développeur';  

-- PostgreSQL affiche :
-- UPDATE 2
```

**Si le nombre ne correspond pas à vos attentes, c'est qu'il y a un problème !**

#### 4. Limiter le nombre de lignes modifiées

> ⚠️ **Attention** : contrairement à MySQL/MariaDB, **PostgreSQL n'accepte PAS `LIMIT` directement dans un `UPDATE` ou un `DELETE`**. La syntaxe `UPDATE … WHERE … LIMIT n` produira une erreur de syntaxe.

Pour limiter le nombre de lignes affectées, on passe par une **sous-requête** sur la clé primaire :

```sql
-- ✅ Forme correcte en PostgreSQL : limiter à 10 lignes via une sous-requête
UPDATE employes  
SET salaire = salaire * 1.05  
WHERE id IN (  
    SELECT id  
    FROM employes  
    WHERE poste = 'Développeur'  
    ORDER BY id  
    LIMIT 10  
);  
```

L'`ORDER BY` dans la sous-requête est important : sans lui, le « top 10 » serait indéterministe (la 11e ligne pourrait être traitée à la place de la 1re d'une exécution à l'autre).

Cette technique est particulièrement utile pour les **suppressions par lots** (voir plus loin dans cette section, *Limiter le nombre de lignes (pour les suppressions massives)*) afin d'éviter les verrous longue durée sur une table volumineuse.

### Conditions WHERE complexes

#### Opérateurs de comparaison

```sql
-- Égalité
UPDATE employes SET poste = 'Senior Developer' WHERE poste = 'Développeur';

-- Inégalité
UPDATE employes SET statut = 'Actif' WHERE salaire <> 0;

-- Comparaisons numériques
UPDATE employes SET categorie = 'Junior' WHERE salaire < 40000;  
UPDATE employes SET categorie = 'Senior' WHERE salaire >= 60000;  

-- Plages
UPDATE employes SET niveau = 'Intermédiaire' WHERE salaire BETWEEN 40000 AND 60000;
```

#### Conditions multiples avec AND/OR

```sql
-- AND : Les deux conditions doivent être vraies
UPDATE employes  
SET prime_anciennete = 2000  
WHERE poste = 'Développeur'  
  AND date_embauche < '2022-01-01';

-- OR : Au moins une condition doit être vraie
UPDATE employes  
SET eligible_formation = true  
WHERE salaire < 40000  
   OR date_embauche > '2024-01-01';

-- Combinaison avec parenthèses
UPDATE employes  
SET bonus = 5000  
WHERE (poste = 'Chef de projet' OR poste = 'Directeur technique')  
  AND salaire > 50000;
```

#### Pattern matching avec LIKE et ILIKE

```sql
-- LIKE (sensible à la casse)
UPDATE employes  
SET domaine = 'Gmail'  
WHERE email LIKE '%@gmail.com';  

-- ILIKE (insensible à la casse - spécifique à PostgreSQL)
UPDATE employes  
SET domaine = 'Gmail'  
WHERE email ILIKE '%@GMAIL.COM';  

-- Wildcard % (n'importe quelle séquence)
UPDATE employes  
SET type_compte = 'Pro'  
WHERE email LIKE '%@company.%';  

-- Wildcard _ (un seul caractère)
UPDATE employes  
SET categorie_code = 'DEV'  
WHERE poste LIKE 'D_veloppeur';  
```

#### Expressions régulières

```sql
-- Syntaxe ~ (sensible à la casse)
UPDATE employes  
SET domaine_valide = true  
WHERE email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$';  

-- Syntaxe ~* (insensible à la casse)
UPDATE employes  
SET nom_valide = true  
WHERE nom ~* '^[a-z]+$';  
```

#### Conditions NULL

```sql
-- IS NULL
UPDATE employes  
SET salaire = 35000  
WHERE salaire IS NULL;  

-- IS NOT NULL
UPDATE employes  
SET statut = 'Complet'  
WHERE email IS NOT NULL AND telephone IS NOT NULL;  
```

#### Sous-requêtes dans WHERE

```sql
-- Mettre à jour les employés dont le salaire est inférieur à la moyenne
UPDATE employes  
SET salaire = salaire * 1.15  
WHERE salaire < (SELECT AVG(salaire) FROM employes);  

-- Mettre à jour selon une autre table
UPDATE employes e  
SET departement_id = 5  
WHERE e.id IN (  
    SELECT employe_id
    FROM affectations
    WHERE date_fin IS NULL
);
```

---

## 6.3.3. UPDATE avec JOIN : Modifier selon d'autres tables

### Syntaxe PostgreSQL (FROM)

PostgreSQL permet de joindre d'autres tables dans un UPDATE via la clause `FROM` :

```sql
UPDATE table_a  
SET colonne = nouvelle_valeur  
FROM table_b  
WHERE table_a.id = table_b.id  
  AND autre_condition;
```

### Exemples pratiques

#### Tables de démonstration

```sql
CREATE TABLE departements (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    budget NUMERIC(12, 2)
);

CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    departement_id INTEGER REFERENCES departements(id),
    salaire NUMERIC(10, 2),
    bonus NUMERIC(10, 2) DEFAULT 0
);

INSERT INTO departements (nom, budget) VALUES
    ('IT', 500000),
    ('RH', 200000),
    ('Ventes', 800000);

INSERT INTO employes (nom, departement_id, salaire) VALUES
    ('Alice', 1, 50000),
    ('Bob', 1, 55000),
    ('Charlie', 2, 45000),
    ('Diana', 3, 60000);
```

#### UPDATE basé sur une autre table

Attribuer un bonus proportionnel au budget du département :

```sql
UPDATE employes e  
SET bonus = d.budget * 0.02  -- 2% du budget du département  
FROM departements d  
WHERE e.departement_id = d.id;  
```

**Résultat** :
- Alice et Bob (IT) : bonus = 500000 * 0.02 = 10000
- Charlie (RH) : bonus = 200000 * 0.02 = 4000
- Diana (Ventes) : bonus = 800000 * 0.02 = 16000

#### UPDATE avec conditions complexes

```sql
-- Augmenter le salaire des employés dans les départements avec gros budget
UPDATE employes e  
SET salaire = salaire * 1.20  
FROM departements d  
WHERE e.departement_id = d.id  
  AND d.budget > 300000
  AND e.salaire < 60000;
```

### UPDATE avec multiples tables

```sql
CREATE TABLE evaluations (
    id SERIAL PRIMARY KEY,
    employe_id INTEGER,
    note NUMERIC(3, 1),
    date_eval DATE
);

-- Augmenter le salaire selon la dernière évaluation et le budget du département
UPDATE employes e  
SET salaire = salaire * (1 + (ev.note / 100))  
FROM departements d, evaluations ev  
WHERE e.departement_id = d.id  
  AND e.id = ev.employe_id
  AND ev.date_eval = (
      SELECT MAX(date_eval)
      FROM evaluations
      WHERE employe_id = e.id
  )
  AND d.budget > 400000;
```

---

## 6.3.4. La commande DELETE : Supprimer des données

### Concept de base

`DELETE` supprime définitivement des lignes d'une table. C'est une opération **irréversible** (sauf si utilisée dans une transaction non committée).

### Syntaxe générale

```sql
DELETE FROM nom_table  
WHERE condition;  
```

### Exemple simple

```sql
-- Supprimer un employé spécifique
DELETE FROM employes  
WHERE id = 5;  

-- Vérifier
SELECT COUNT(*) FROM employes WHERE id = 5;
-- Résultat : 0 (l'employé n'existe plus)
```

### DELETE avec conditions

#### Supprimer selon une valeur

```sql
-- Supprimer tous les employés d'un département
DELETE FROM employes  
WHERE departement_id = 3;  
```

#### Supprimer selon une plage

```sql
-- Supprimer les anciens enregistrements
DELETE FROM logs  
WHERE date_creation < '2024-01-01';  

-- Supprimer les bas salaires (attention !)
DELETE FROM employes  
WHERE salaire < 30000;  
```

#### Supprimer avec conditions multiples

```sql
-- Supprimer les employés sans email et sans téléphone
DELETE FROM employes  
WHERE email IS NULL  
  AND telephone IS NULL;

-- Supprimer les doublons (garder le plus récent)
DELETE FROM employes e1  
WHERE EXISTS (  
    SELECT 1
    FROM employes e2
    WHERE e1.email = e2.email
      AND e1.date_embauche < e2.date_embauche
);
```

### ⚠️ DELETE sans WHERE : DANGER ABSOLU

```sql
-- ❌❌❌ CATASTROPHIQUE - NE JAMAIS FAIRE ❌❌❌
DELETE FROM employes;
```

**Résultat** : **TOUTES** les lignes de la table sont supprimées définitivement !

```sql
SELECT COUNT(*) FROM employes;
-- Résultat : 0 (table vide)
```

### DELETE avec sous-requêtes

```sql
-- Supprimer les employés dont le salaire est inférieur à la moyenne de leur département
DELETE FROM employes e  
WHERE salaire < (  
    SELECT AVG(salaire)
    FROM employes
    WHERE departement_id = e.departement_id
);

-- Supprimer les employés dans les départements dissous
DELETE FROM employes  
WHERE departement_id IN (  
    SELECT id
    FROM departements
    WHERE date_dissolution IS NOT NULL
);
```

### DELETE avec JOIN (syntaxe PostgreSQL)

```sql
-- Supprimer les employés des départements avec budget insuffisant
DELETE FROM employes e  
USING departements d  
WHERE e.departement_id = d.id  
  AND d.budget < 100000;
```

---

## 6.3.5. Précautions et bonnes pratiques

### 1. Toujours utiliser des transactions en environnement critique

```sql
-- Transaction explicite
BEGIN;

-- Effectuer la suppression
DELETE FROM employes WHERE poste = 'Stagiaire';

-- Vérifier le nombre de lignes affectées
-- PostgreSQL affiche : DELETE 5

-- Si c'est correct :
COMMIT;

-- Si erreur :
-- ROLLBACK;
```

### 2. Sauvegarder avant modification massive

Avant de modifier ou supprimer de nombreuses lignes, créez une copie :

```sql
-- Créer une table de backup
CREATE TABLE employes_backup AS  
SELECT * FROM employes;  

-- Ou exporter
COPY employes TO '/tmp/employes_backup.csv' WITH (FORMAT csv, HEADER true);

-- Effectuer ensuite les modifications
DELETE FROM employes WHERE date_embauche < '2020-01-01';

-- En cas de problème, restaurer
-- INSERT INTO employes SELECT * FROM employes_backup;
```

### 3. Utiliser des constraints pour protéger l'intégrité

```sql
-- Une foreign key avec ON DELETE protège
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    departement_id INTEGER REFERENCES departements(id) ON DELETE RESTRICT
);

-- Tentative de suppression d'un département avec des employés
DELETE FROM departements WHERE id = 1;
-- ERROR: update or delete on table "departements" violates foreign key constraint
```

Options de `ON DELETE` :
- `NO ACTION` (**défaut**) : empêche la suppression si des dépendances existent ; la vérification a lieu en **fin d'instruction**, ce qui la rend *différable* avec `DEFERRABLE INITIALLY DEFERRED`  
- `RESTRICT` : empêche la suppression de la même façon, mais la vérification est **immédiate** (non différable)  
- `CASCADE` : Supprime aussi les lignes dépendantes  
- `SET NULL` : Met à NULL la foreign key dans les lignes dépendantes  
- `SET DEFAULT` : Met la valeur par défaut dans les lignes dépendantes

### 4. Tester avec SELECT avant DELETE

```sql
-- ÉTAPE 1 : Compter combien de lignes seront affectées
SELECT COUNT(*)  
FROM employes  
WHERE date_embauche < '2020-01-01';  
-- Résultat : 23 lignes

-- ÉTAPE 2 : Examiner les lignes concernées
SELECT id, nom, prenom, date_embauche  
FROM employes  
WHERE date_embauche < '2020-01-01'  
LIMIT 10;  

-- ÉTAPE 3 : Si tout est OK, exécuter DELETE
DELETE FROM employes  
WHERE date_embauche < '2020-01-01';  
```

### 5. Limiter le nombre de lignes (pour les suppressions massives)

Pour éviter de verrouiller la table trop longtemps et d'enfler le journal des transactions (WAL), il vaut mieux supprimer **par lots successifs** dans une boucle plutôt qu'en un seul gros `DELETE`. Comme `DELETE` n'accepte pas `LIMIT` directement, on passe par une sous-requête sur la clé primaire :

```sql
-- Supprimer par lots de 10 000 lignes au sein d'une procédure
-- (une procédure appelée par CALL peut COMMITer entre chaque lot ; une
--  FONCTION PL/pgSQL classique, elle, ne le peut pas)
CREATE OR REPLACE PROCEDURE purge_anciens_logs()  
LANGUAGE plpgsql  
AS $$  
DECLARE  
    deleted_count INTEGER;
BEGIN
    LOOP
        DELETE FROM logs
        WHERE id IN (
            SELECT id FROM logs
            WHERE date_creation < '2023-01-01'
            ORDER BY id
            LIMIT 10000
        );

        GET DIAGNOSTICS deleted_count = ROW_COUNT;
        EXIT WHEN deleted_count = 0;

        -- Commit après chaque lot : libère les verrous, permet à autovacuum
        -- de progresser, réduit la taille du WAL non rejoué.
        COMMIT;
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

-- Lancer la purge (s'exécute hors d'une transaction englobante)
CALL purge_anciens_logs();
```

> 📌 **Qui peut faire un `COMMIT` au milieu du code ?** Depuis PostgreSQL 11, le contrôle de transaction (`COMMIT`/`ROLLBACK`) est autorisé **aussi bien dans une procédure (`CALL`) que dans un bloc anonyme `DO`** — les deux conviennent pour cette purge par lots. Une **fonction** PL/pgSQL classique (évaluée à l'intérieur d'un `SELECT`), elle, ne le peut **pas** : un `COMMIT` y lève `ERROR: invalid transaction termination`. Seule condition pour que le `COMMIT` interne soit permis : la procédure (ou le `DO`) doit être lancée **hors d'une transaction explicite** — un `BEGIN; CALL …; COMMIT;` (ou `BEGIN; DO …; COMMIT;`) englobant ferait échouer le `COMMIT` interne avec cette même `invalid transaction termination`. On choisit ici une **procédure nommée** parce qu'elle est réutilisable et planifiable (par exemple via `pg_cron`), non pour une raison technique de transaction.

> 💡 **Variante avec `FOR UPDATE SKIP LOCKED`** : dans un système où plusieurs workers purgent en parallèle, ajouter `FOR UPDATE SKIP LOCKED` dans la sous-requête évite que deux workers attendent les mêmes lignes :  
>
> ```sql
> DELETE FROM logs
> WHERE id IN (
>     SELECT id FROM logs
>     WHERE date_creation < '2023-01-01'
>     ORDER BY id
>     LIMIT 10000
>     FOR UPDATE SKIP LOCKED
> );
> ```

### 6. Documenter les opérations critiques

```sql
-- Ajouter des commentaires explicites
-- MAINTENANCE : Suppression des logs de plus de 1 an
-- Date : 2025-11-19
-- Responsable : Jean Dupont
-- Nombre estimé : ~50 000 lignes

BEGIN;

DELETE FROM logs  
WHERE date_creation < CURRENT_DATE - INTERVAL '1 year';  

-- Vérifier : DELETE 48732

COMMIT;
```

---

## 6.3.6. TRUNCATE : L'alternative à DELETE pour vider une table

### Qu'est-ce que TRUNCATE ?

`TRUNCATE` est une commande spécialisée pour supprimer **toutes** les lignes d'une table rapidement.

### Syntaxe

```sql
TRUNCATE TABLE nom_table;
```

### Différences entre `DELETE` et `TRUNCATE`

| Aspect | `DELETE` | `TRUNCATE` |
|--------|----------|------------|
| **Syntaxe** | `DELETE FROM table WHERE …` | `TRUNCATE TABLE table` |
| **Clause `WHERE`** | ✅ Oui (ciblage fin) | ❌ Non (vide la table entière) |
| **Vitesse** | O(n) — proportionnelle au nombre de lignes | ⚡ Quasi instantanée, indépendante du volume |
| **Triggers `ROW` (`BEFORE`/`AFTER DELETE`)** | ✅ Déclenchés pour chaque ligne | ❌ Non déclenchés |
| **Triggers `STATEMENT` (`BEFORE`/`AFTER TRUNCATE`)** | N/A | ✅ Déclenchés (depuis PG 8.4) |
| **`ROLLBACK`** | ✅ Transactionnel | ✅ Transactionnel (spécificité PostgreSQL) |
| **Espace disque** | Récupéré par `VACUUM` ultérieur (les pages restent allouées) | **Restitué immédiatement à l'OS** (la table est tronquée à 0 octet) |
| **Séquences `IDENTITY` / `SERIAL`** | Inchangées | Réinitialisées si `RESTART IDENTITY`, sinon inchangées |
| **Clés étrangères entrantes** | Vérifiées ligne par ligne (échec sur dépendance) | Refusée tant que d'autres tables référencent celle-ci, sauf `CASCADE` |
| **Verrouillage** | `RowExclusiveLock` (autres lecteurs et écrivains progressent) | `AccessExclusiveLock` (bloque tout autre accès le temps de l'opération) |
| **WAL généré** | Important (un enregistrement par tuple) | Minimal (juste la troncature du fichier) |

### Exemples d'utilisation

#### TRUNCATE simple

```sql
-- Vider complètement la table
TRUNCATE TABLE logs;
```

#### TRUNCATE avec réinitialisation de séquences

```sql
-- Vider et réinitialiser les ID auto-incrémentés
TRUNCATE TABLE employes RESTART IDENTITY;

-- Le prochain INSERT aura id = 1
```

#### TRUNCATE en cascade

```sql
-- Vider la table et les tables dépendantes
TRUNCATE TABLE departements CASCADE;
-- Vide aussi 'employes' si foreign key existe
```

### Quand utiliser TRUNCATE ?

✅ **Utilisez TRUNCATE quand** :
- Vous voulez vider complètement une table
- La performance est critique
- Vous n'avez pas besoin de triggers
- C'est une table de *staging* ou temporaire

❌ **N'utilisez PAS TRUNCATE quand** :
- Vous voulez supprimer seulement certaines lignes
- Vous devez garder une trace (audit)
- Les triggers sont nécessaires
- Des foreign keys pointent vers la table

### Exemple comparatif

```sql
-- Créer une table de test
CREATE TABLE test_delete (
    id SERIAL PRIMARY KEY,
    data TEXT
);

-- Insérer 1 million de lignes
INSERT INTO test_delete (data)  
SELECT 'data_' || generate_series(1, 1000000);  

-- Test DELETE (lent)
BEGIN;  
DELETE FROM test_delete;  
-- Durée : ~15 secondes
ROLLBACK;

-- Test TRUNCATE (rapide)
BEGIN;  
TRUNCATE TABLE test_delete;  
-- Durée : ~0.1 secondes
ROLLBACK;
```

---

## 6.3.7. Impact sur les performances

### UPDATE : Considérations de performance

#### 1. Index et UPDATE — l'optimisation HOT

Les `UPDATE` peuvent être ralentis par les index, puisque chaque colonne indexée modifiée doit aussi voir son index mis à jour :

```sql
-- Table avec plusieurs index
CREATE TABLE produits (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(200),
    prix NUMERIC(10, 2),
    stock INTEGER,
    date_maj TIMESTAMP
);

CREATE INDEX idx_produits_nom   ON produits(nom);  
CREATE INDEX idx_produits_prix  ON produits(prix);  
CREATE INDEX idx_produits_stock ON produits(stock);  
```

PostgreSQL applique cependant une **optimisation très efficace appelée HOT update** (*Heap-Only Tuple update*) : si l'`UPDATE`
- **ne modifie aucune colonne indexée** *et*
- **la nouvelle version de la ligne tient sur la même page** que l'ancienne,

alors **aucun index n'est mis à jour**. La nouvelle version est chaînée à l'ancienne sur la même page, et seule l'entrée d'index pointe (indirectement) vers la chaîne. C'est **considérablement plus rapide** et réduit le bloat.

```sql
-- 🚀 HOT update possible : aucune colonne indexée touchée
UPDATE produits SET date_maj = NOW() WHERE id = 42;

-- ❌ HOT update impossible : modifie 'prix' qui est indexé
UPDATE produits SET prix = prix * 1.1 WHERE id = 42;
```

> 💡 **Conséquence pratique** : si une table subit beaucoup d'`UPDATE` fréquents sur une colonne « technique » (compteur, dernier accès, `last_seen_at`), évitez de l'indexer si ce n'est pas nécessaire — vous préserverez la possibilité de HOT updates. Pour favoriser les HOT, on peut aussi baisser le `fillfactor` de la table (par défaut 100, on met souvent 80–90 sur les tables très volatiles) afin de laisser de la place libre sur chaque page pour la nouvelle version :  
>
> ```sql
> ALTER TABLE produits SET (fillfactor = 85);
> ```

#### 2. MVCC et *bloat*

PostgreSQL implémente l'isolation via **MVCC** (*Multi-Version Concurrency Control*). Lors d'un `UPDATE` :
- L'ancienne version (tuple) est **marquée comme obsolète** mais reste physiquement présente tant que d'anciennes transactions peuvent encore la voir.
- Une **nouvelle version** est écrite (sur la même page si possible — voir HOT, sinon ailleurs).
- L'espace des versions obsolètes est récupéré par **`VACUUM`** (manuel ou automatique via `autovacuum`).

```sql
-- UPDATE massif : crée une nouvelle version pour chaque ligne touchée
UPDATE employes SET salaire = salaire * 1.1;

-- Récupération de l'espace par autovacuum, ou manuellement :
VACUUM employes;

-- Pour réorganiser physiquement et récupérer tout l'espace
-- (⚠️ prend un AccessExclusiveLock — table indisponible le temps de l'opération) :
VACUUM FULL employes;
```

L'accumulation de tuples obsolètes non encore *vacuumés* s'appelle le **bloat** : un `UPDATE` qui met à jour 10 fois une table de 1 Go peut, sans `VACUUM`, faire enfler la table à plusieurs Go.

#### 3. UPDATE partiel vs UPDATE complet

```sql
-- ⚠️ Verbeux : réassigne des colonnes qui ne changent pas (expressions évaluées pour rien)
UPDATE employes  
SET nom = nom,  
    prenom = prenom,
    email = email,
    salaire = salaire * 1.1;

-- ✅ Efficace : ne liste que la colonne réellement modifiée
UPDATE employes  
SET salaire = salaire * 1.1;  
```

Pourquoi cette différence ?

- Une nouvelle version du tuple est **toujours créée** dès lors que l'`UPDATE` matche la ligne, **même si toutes les nouvelles valeurs sont identiques aux anciennes** (PostgreSQL ne compare pas « avant/après » pour *annuler* l'écriture).
- Ce qui décide du **HOT update**, ce n'est pas la *présence* d'une colonne dans `SET`, mais le fait qu'une colonne **indexée change réellement de valeur**. PostgreSQL compare l'ancienne et la nouvelle valeur de chaque colonne indexée : réécrire `nom = nom` (valeur identique) **préserve** le HOT update, alors que `nom = UPPER(nom)` (valeur réellement modifiée), si `nom` est indexé, le **bloque**. *(Vérifiable via `n_tup_hot_upd` dans `pg_stat_user_tables`.)*
- Lister inutilement des colonnes reste néanmoins une mauvaise habitude : on évalue des expressions pour rien et on risque de déclencher des triggers `BEFORE UPDATE OF colonne`.
- La seconde forme ne touche que `salaire` : si `salaire` n'est pas indexé, le HOT update est pleinement possible.

### DELETE : Considérations de performance

#### 1. DELETE et index

```sql
-- Les index doivent être mis à jour lors du DELETE
DELETE FROM logs WHERE date_creation < '2024-01-01';
-- Tous les index sur 'logs' sont modifiés
```

#### 2. DELETE en cascade

```sql
-- Si ON DELETE CASCADE est défini
DELETE FROM departements WHERE id = 1;
-- Peut déclencher la suppression de milliers d'employés
-- → Opération potentiellement très longue
```

#### 3. Fragmentation

Les DELETE répétés fragmentent les tables :

```sql
-- Supprimer 50% des lignes
DELETE FROM logs WHERE id % 2 = 0;

-- Les pages contiennent maintenant beaucoup d'espace vide
-- Solution : VACUUM FULL (réorganise physiquement)
VACUUM FULL logs;
```

---

## 6.3.7 bis. Verrouillage et concurrence

Comprendre les verrous est essentiel dès qu'on a plus d'une session qui écrit.

### Verrous de ligne (row-level locks)

`UPDATE` et `DELETE` posent un **`ROW EXCLUSIVE` lock** sur la table et, **plus important**, un **verrou de tuple** (*row lock*) sur chaque ligne qu'ils modifient. Ce verrou est conservé jusqu'à la fin de la transaction (`COMMIT` ou `ROLLBACK`).

```sql
-- Session A
BEGIN;  
UPDATE comptes SET solde = solde - 100 WHERE id = 1;  
-- La ligne id=1 est verrouillée pour modification.

-- Session B (en parallèle)
UPDATE comptes SET solde = solde + 50 WHERE id = 1;
-- ⏳ ATTEND la fin de la transaction A.

UPDATE comptes SET solde = solde - 100 WHERE id = 2;
-- ✅ OK : ligne différente.

SELECT * FROM comptes WHERE id = 1;
-- ✅ OK : les lecteurs ne sont JAMAIS bloqués par les écrivains (MVCC).
-- Session B voit ENCORE l'ancienne valeur de id=1.
```

> 💡 **Règle d'or** : un `UPDATE` qui dure 10 secondes verrouille les lignes affectées pendant 10 secondes. Évitez de combiner du `UPDATE` avec du code applicatif lent à l'intérieur d'une même transaction.

### Verrous explicites : `SELECT … FOR UPDATE`

Pour réserver une ligne avant de la modifier (par exemple pour un pattern « lire-décider-écrire » atomique), on utilise `SELECT … FOR UPDATE` :

```sql
BEGIN;

-- Réserver la ligne pour modification : pose le même verrou qu'un UPDATE
SELECT solde FROM comptes WHERE id = 1 FOR UPDATE;
-- → 1000

-- Vérification métier côté applicatif…
-- Si tout est OK, modifier
UPDATE comptes SET solde = solde - 100 WHERE id = 1;

COMMIT;
```

Variantes utiles :

| Clause | Effet |
|--------|-------|
| `FOR UPDATE` | Verrou exclusif sur la ligne — autres `UPDATE`/`DELETE`/`SELECT FOR UPDATE` attendent |
| `FOR NO KEY UPDATE` | Plus permissif : autorise les `UPDATE` qui ne touchent pas une clé référencée par une FK |
| `FOR SHARE` | Verrou partagé : empêche modification mais autorise d'autres `FOR SHARE` |
| `FOR KEY SHARE` | Le plus faible : empêche uniquement suppression et modification des clés |
| `… NOWAIT` | Échec immédiat (`could not obtain lock`) si la ligne est déjà verrouillée |
| `… SKIP LOCKED` | Saute les lignes verrouillées (idéal pour des *workers* concurrents qui se partagent une file) |

### Pattern : file d'attente avec `SKIP LOCKED`

Plusieurs *workers* veulent traiter une table `tâches` sans se marcher dessus :

```sql
-- Chaque worker exécute, en boucle :
BEGIN;

SELECT id, payload  
FROM taches  
WHERE statut = 'à_traiter'  
ORDER BY priorite DESC, id  
LIMIT 1  
FOR UPDATE SKIP LOCKED;  -- ← clé du pattern  

-- (traitement côté application…)

UPDATE taches SET statut = 'fait' WHERE id = …;  
COMMIT;  
```

Chaque *worker* récupère **une** tâche, saute toutes celles déjà tenues par d'autres *workers*, et n'attend pas. C'est le pattern utilisé par les files de tâches **basées sur PostgreSQL** comme [Oban](https://github.com/oban-bg/oban) (Elixir), [graphile-worker](https://github.com/graphile/worker) (Node.js) ou [Que](https://github.com/que-rb/que) (Ruby). *(À l'inverse, des systèmes comme Sidekiq reposent sur Redis, pas sur PostgreSQL.)*

### Détecter les conflits de verrous (*lock waits*)

Quand une session bloque, on peut voir qui attend qui :

```sql
SELECT
    blocked.pid          AS pid_bloque,
    blocked.query        AS requete_bloquee,
    blocking.pid         AS pid_bloquant,
    blocking.query       AS requete_bloquante,
    age(now(), blocked.xact_start) AS attente_depuis
FROM pg_stat_activity blocked  
JOIN pg_stat_activity blocking  
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.state = 'active';
```

En cas d'urgence, on peut tuer une session :

```sql
SELECT pg_cancel_backend(12345);   -- Annule la requête (doux : envoie SIGINT)  
SELECT pg_terminate_backend(12345); -- Tue la connexion (brutal)  
```

### Deadlocks

Deux transactions qui se demandent mutuellement des verrous **en sens inverse** créent un *deadlock*. PostgreSQL le détecte et **avorte l'une des deux** avec `ERROR: deadlock detected` :

```
-- Session A         |  Session B
BEGIN;               |  BEGIN;  
UPDATE c WHERE id=1; |  UPDATE c WHERE id=2;  
UPDATE c WHERE id=2; |  UPDATE c WHERE id=1;  
-- attend B          |  -- attend A → deadlock !
```

**Prévention** : toujours acquérir les verrous **dans le même ordre** (par exemple `ORDER BY id` avant un `FOR UPDATE`, ou trier les `UPDATE` côté application par clé primaire croissante).

---

## 6.3.8. Patterns anti-corruption

### Pattern 1 : Soft Delete (suppression logique)

Au lieu de supprimer physiquement, marquer comme supprimé :

```sql
-- Ajouter une colonne
ALTER TABLE employes ADD COLUMN deleted_at TIMESTAMP;

-- "Supprimer" logiquement
UPDATE employes  
SET deleted_at = CURRENT_TIMESTAMP  
WHERE id = 5;  

-- Requêter les employés actifs
SELECT * FROM employes WHERE deleted_at IS NULL;

-- Avantages :
-- - Récupération possible
-- - Historique conservé
-- - Audit simplifié
```

### Pattern 2 : Archivage avant suppression

```sql
-- Créer une table d'archive
CREATE TABLE employes_archives (LIKE employes INCLUDING ALL);

-- Déplacer avant de supprimer
BEGIN;

-- 1. Copier dans l'archive
INSERT INTO employes_archives  
SELECT * FROM employes  
WHERE date_embauche < '2020-01-01';  

-- 2. Supprimer de la table principale
DELETE FROM employes  
WHERE date_embauche < '2020-01-01';  

COMMIT;
```

### Pattern 3 : Validation en deux étapes

```sql
-- Étape 1 : Marquer pour suppression
UPDATE employes  
SET to_delete = true  
WHERE condition;  

-- Vérification manuelle ou automatique

-- Étape 2 : Suppression effective (après validation)
DELETE FROM employes  
WHERE to_delete = true;  
```

---

## 6.3.9. Audit et traçabilité

### Enregistrer les modifications

```sql
-- Table d'audit
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    operation VARCHAR(10),
    old_data JSONB,
    new_data JSONB,
    user_name VARCHAR(100),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Trigger pour tracker les modifications
CREATE OR REPLACE FUNCTION audit_trigger()  
RETURNS TRIGGER AS $$  
BEGIN  
    IF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, new_data, user_name)
        VALUES (TG_TABLE_NAME, 'UPDATE', row_to_json(OLD), row_to_json(NEW), current_user);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, user_name)
        VALUES (TG_TABLE_NAME, 'DELETE', row_to_json(OLD), current_user);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Attacher le trigger
CREATE TRIGGER employes_audit  
AFTER UPDATE OR DELETE ON employes  
FOR EACH ROW EXECUTE FUNCTION audit_trigger();  
```

---

## 6.3.10. Checklist de sécurité

Avant chaque UPDATE ou DELETE en production :

- [ ] **Ai-je une clause WHERE appropriée ?**  
- [ ] **Ai-je testé avec SELECT d'abord ?**  
- [ ] **Le nombre de lignes affectées correspond-il à mes attentes ?**  
- [ ] **Suis-je dans une transaction BEGIN...COMMIT ?**  
- [ ] **Ai-je une sauvegarde récente ?**  
- [ ] **Les contraintes d'intégrité sont-elles respectées ?**  
- [ ] **Les utilisateurs/applications sont-ils informés (downtime) ?**  
- [ ] **Ai-je un plan de rollback si problème ?**  
- [ ] **La commande est-elle documentée (qui, quand, pourquoi) ?**  
- [ ] **Ai-je testé sur un environnement de développement / *staging* ?**

---

## Récapitulatif et meilleures pratiques

### UPDATE : Points clés

1. **Toujours avoir une clause WHERE** (sauf intention explicite de tout modifier)  
2. **Tester avec SELECT d'abord** pour vérifier le ciblage  
3. **Utiliser des transactions** pour permettre le rollback  
4. **Vérifier le nombre de lignes affectées** après exécution  
5. **Préférer les modifications partielles** aux modifications complètes  
6. **Considérer l'impact sur les index** pour les gros volumes

### DELETE : Points clés

1. **Extrêmement dangereux sans WHERE** - Toujours vérifier  
2. **Tester le ciblage avec SELECT COUNT(*) d'abord**  
3. **Considérer les contraintes de foreign key** (CASCADE, RESTRICT)  
4. **Envisager le soft delete** pour les données sensibles  
5. **Archiver avant de supprimer** pour l'historique  
6. **Utiliser TRUNCATE** pour vider complètement une table (plus rapide)

### Comparaison rapide

| Action | Commande | Réversible | WHERE | Performance |
|--------|----------|-----------|-------|-------------|
| Modifier des lignes | UPDATE | ✅ (transaction) | Requis | Moyen |
| Supprimer des lignes | DELETE | ✅ (transaction) | Requis | Moyen |
| Vider table | TRUNCATE | ✅ (transaction) | ❌ | ⚡ Rapide |
| Supprimer table | DROP TABLE | ✅ (transaction) | N/A | ⚡ Instantané |

---

## Conclusion

Les commandes `UPDATE` et `DELETE` sont des outils puissants mais dangereux. Leur maîtrise requiert :

1. **Prudence** : Toujours vérifier le ciblage avec WHERE  
2. **Précaution** : Utiliser des transactions et des sauvegardes  
3. **Préparation** : Tester sur des environnements non-critiques  
4. **Prévoyance** : Mettre en place des mécanismes d'audit et de récupération

En suivant les bonnes pratiques de cette section, vous minimiserez les risques d'erreurs catastrophiques tout en exploitant pleinement ces commandes essentielles.

> **Rappel final** : En production, une erreur UPDATE ou DELETE peut coûter des heures de travail, des données précieuses, voire la confiance des utilisateurs. Soyez **toujours** vigilant.

---


⏭️ [La clause RETURNING : Une spécificité puissante](/06-manipulation-des-donnees/04-clause-returning.md)
