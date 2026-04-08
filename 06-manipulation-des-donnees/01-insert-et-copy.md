🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1. INSERT : Insertion simple, multiple, et import via COPY

## Introduction

L'insertion de données est l'une des opérations fondamentales dans toute base de données. Dans PostgreSQL, la commande `INSERT` permet d'ajouter de nouvelles lignes dans une table, tandis que la commande `COPY` offre une solution optimisée pour l'importation massive de données. Cette section vous guidera à travers les différentes façons d'insérer des données, de la plus simple à la plus performante.

---

## Concepts de base

Avant de commencer, rappelons quelques notions essentielles :

- **Une table** contient des lignes (enregistrements) et des colonnes (attributs)  
- **Une insertion** ajoute une ou plusieurs nouvelles lignes dans une table  
- **Les types de données** de chaque colonne doivent être respectés  
- **Les contraintes** (NOT NULL, UNIQUE, CHECK, etc.) sont vérifiées à chaque insertion

---

## 6.1.1. Insertion simple : La commande INSERT basique

### Syntaxe générale

La forme la plus simple de `INSERT` permet d'ajouter une seule ligne à la fois :

```sql
INSERT INTO nom_table (colonne1, colonne2, colonne3)  
VALUES (valeur1, valeur2, valeur3);  
```

### Exemple concret

Imaginons une table `employes` avec la structure suivante :

```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    salaire NUMERIC(10, 2),
    date_embauche DATE DEFAULT CURRENT_DATE
);
```

Pour insérer un nouvel employé :

```sql
INSERT INTO employes (nom, prenom, email, salaire, date_embauche)  
VALUES ('Dupont', 'Marie', 'marie.dupont@example.com', 45000.00, '2025-01-15');  
```

### Insertion sans spécifier toutes les colonnes

Vous n'êtes pas obligé de spécifier toutes les colonnes :

```sql
INSERT INTO employes (nom, prenom, email)  
VALUES ('Martin', 'Pierre', 'pierre.martin@example.com');  
```

Dans cet exemple :
- La colonne `id` sera générée automatiquement (SERIAL)
- La colonne `salaire` recevra NULL (si autorisé)
- La colonne `date_embauche` prendra la valeur par défaut (CURRENT_DATE)

### Insertion sans spécifier les colonnes

Si vous fournissez des valeurs pour **toutes** les colonnes dans le bon ordre, vous pouvez omettre la liste des colonnes :

```sql
INSERT INTO employes  
VALUES (DEFAULT, 'Durand', 'Sophie', 'sophie.durand@example.com', 48000.00, '2025-02-01');  
```

> **Note** : Cette syntaxe est déconseillée car elle rend le code fragile. Si la structure de la table change (ajout/suppression de colonne), votre requête pourrait échouer.

### Gestion des valeurs NULL et DEFAULT

Plusieurs façons d'insérer des valeurs NULL ou par défaut :

```sql
-- Utiliser NULL explicitement
INSERT INTO employes (nom, prenom, email, salaire)  
VALUES ('Bernard', 'Luc', 'luc.bernard@example.com', NULL);  

-- Utiliser DEFAULT pour invoquer la valeur par défaut
INSERT INTO employes (nom, prenom, email, date_embauche)  
VALUES ('Petit', 'Anne', 'anne.petit@example.com', DEFAULT);  

-- Omettre les colonnes (elles prendront NULL ou DEFAULT)
INSERT INTO employes (nom, prenom, email)  
VALUES ('Roux', 'Thomas', 'thomas.roux@example.com');  
```

---

## 6.1.2. Insertion multiple : Plusieurs lignes en une seule commande

### Pourquoi insérer plusieurs lignes à la fois ?

L'insertion multiple présente plusieurs avantages :

1. **Performance** : Beaucoup plus rapide que plusieurs INSERT séparés  
2. **Transactionnel** : Toutes les insertions réussissent ou échouent ensemble  
3. **Lisibilité** : Code plus compact et plus clair  
4. **Économie réseau** : Un seul aller-retour avec le serveur

### Syntaxe de l'insertion multiple

```sql
INSERT INTO nom_table (colonne1, colonne2, colonne3)  
VALUES  
    (valeur1a, valeur2a, valeur3a),
    (valeur1b, valeur2b, valeur3b),
    (valeur1c, valeur2c, valeur3c);
```

### Exemple pratique

Insertion de plusieurs employés en une seule commande :

```sql
INSERT INTO employes (nom, prenom, email, salaire, date_embauche)  
VALUES  
    ('Moreau', 'Julie', 'julie.moreau@example.com', 42000.00, '2025-03-01'),
    ('Simon', 'Marc', 'marc.simon@example.com', 51000.00, '2025-03-01'),
    ('Laurent', 'Emma', 'emma.laurent@example.com', 47000.00, '2025-03-15'),
    ('Bertrand', 'Paul', 'paul.bertrand@example.com', 39000.00, '2025-03-20');
```

Cette unique requête insère 4 employés en une seule transaction.

### Comparaison de performance

**Méthode inefficace** (4 requêtes séparées) :
```sql
INSERT INTO employes (nom, prenom) VALUES ('Moreau', 'Julie');  
INSERT INTO employes (nom, prenom) VALUES ('Simon', 'Marc');  
INSERT INTO employes (nom, prenom) VALUES ('Laurent', 'Emma');  
INSERT INTO employes (nom, prenom) VALUES ('Bertrand', 'Paul');  
```

**Méthode efficace** (1 seule requête) :
```sql
INSERT INTO employes (nom, prenom)  
VALUES  
    ('Moreau', 'Julie'),
    ('Simon', 'Marc'),
    ('Laurent', 'Emma'),
    ('Bertrand', 'Paul');
```

> **Gain de performance** : L'insertion multiple peut être 5 à 10 fois plus rapide pour de petits volumes, et le gain augmente avec le nombre de lignes.

### Limites pratiques

Bien que PostgreSQL n'impose pas de limite stricte au nombre de lignes dans un INSERT multiple, il existe des considérations pratiques :

- **Mémoire** : Chaque ligne consomme de la mémoire  
- **Lisibilité** : Au-delà de quelques dizaines de lignes, le code devient difficile à lire  
- **Timeout** : Une très longue requête peut dépasser les timeouts réseau ou applicatifs

Pour des volumes importants (milliers de lignes ou plus), préférez la commande `COPY` décrite plus loin.

---

## 6.1.3. Insertion à partir d'une autre table : INSERT ... SELECT

### Concept

PostgreSQL permet d'insérer des données en les sélectionnant depuis une autre table (ou la même table). C'est très utile pour :

- Copier des données d'une table à une autre
- Créer des tables de travail ou d'historique
- Transformer des données lors de l'insertion

### Syntaxe

```sql
INSERT INTO table_destination (colonne1, colonne2, colonne3)  
SELECT colonne_a, colonne_b, colonne_c  
FROM table_source  
WHERE condition;  
```

### Exemple : Création d'une table d'archive

Supposons que vous vouliez archiver les employés qui ont quitté l'entreprise :

```sql
-- Table d'archive
CREATE TABLE employes_archives (
    id INTEGER,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(255),
    date_depart DATE DEFAULT CURRENT_DATE
);

-- Archiver les employés dont le contrat se termine
INSERT INTO employes_archives (id, nom, prenom, email)  
SELECT id, nom, prenom, email  
FROM employes  
WHERE date_fin_contrat < CURRENT_DATE;  
```

### Exemple : Transformation de données

Vous pouvez transformer les données lors de l'insertion :

```sql
-- Créer des noms d'utilisateur en combinant prénom et nom
INSERT INTO utilisateurs (username, email)  
SELECT  
    LOWER(prenom || '.' || nom) AS username,
    email
FROM employes  
WHERE email IS NOT NULL;  
```

Cette requête crée des noms d'utilisateur en minuscules du type "marie.dupont".

### Exemple : Insertion avec jointure

Vous pouvez même utiliser des jointures :

```sql
-- Table des départements
CREATE TABLE departements (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100)
);

-- Table des affectations
CREATE TABLE affectations (
    employe_id INTEGER,
    departement_id INTEGER,
    date_affectation DATE
);

-- Affecter tous les employés au département "Ventes"
INSERT INTO affectations (employe_id, departement_id, date_affectation)  
SELECT  
    e.id,
    d.id,
    CURRENT_DATE
FROM employes e  
CROSS JOIN departements d  
WHERE d.nom = 'Ventes';  
```

---

## 6.1.4. Import massif de données avec COPY

### Qu'est-ce que COPY ?

`COPY` est une commande PostgreSQL spécialement conçue pour l'importation et l'exportation de données en masse. Elle est **beaucoup plus rapide** que des INSERT multiples pour les raisons suivantes :

1. **Contournement du parseur SQL** : Les données sont traitées directement  
2. **Optimisation du WAL** : Écriture groupée dans les journaux de transactions  
3. **Moins de overhead** : Pas de préparation de requête pour chaque ligne  
4. **Format binaire optionnel** : Encore plus rapide que le format texte

### Quand utiliser COPY ?

- **Import initial** : Chargement de millions de lignes lors de la création d'une base  
- **Migrations** : Transfert de données entre systèmes  
- **Chargements ETL** : Extraction, transformation, chargement de données  
- **Restaurations** : Rechargement de sauvegardes

**Règle empirique** : À partir de quelques milliers de lignes, COPY devient significativement plus performant que INSERT.

### Syntaxe de base

Il existe deux variantes de COPY :

#### 1. COPY côté serveur (depuis un fichier sur le serveur)

```sql
COPY nom_table (colonne1, colonne2, colonne3)  
FROM '/chemin/vers/fichier.csv'  
WITH (FORMAT csv, HEADER true, DELIMITER ',');  
```

> **Important** : Le fichier doit être accessible par le processus PostgreSQL (droits de lecture). Cette commande nécessite généralement des privilèges superutilisateur.

#### 2. \copy côté client (depuis psql)

```sql
\copy nom_table (colonne1, colonne2, colonne3)
FROM '/chemin/local/fichier.csv'  
WITH (FORMAT csv, HEADER true, DELIMITER ',');  
```

> **Note** : `\copy` est une commande **psql** (l'outil en ligne de commande), pas une commande SQL. Elle transfère les données depuis le client vers le serveur.

### Format CSV (le plus courant)

#### Structure d'un fichier CSV

Exemple de fichier `employes.csv` :

```csv
nom,prenom,email,salaire,date_embauche  
Dupont,Marie,marie.dupont@example.com,45000.00,2025-01-15  
Martin,Pierre,pierre.martin@example.com,52000.00,2025-01-20  
Durand,Sophie,sophie.durand@example.com,48000.00,2025-02-01  
Bernard,Luc,luc.bernard@example.com,41000.00,2025-02-10  
```

#### Importation basique

```sql
COPY employes (nom, prenom, email, salaire, date_embauche)  
FROM '/tmp/employes.csv'  
WITH (FORMAT csv, HEADER true);  
```

Explications :
- `FORMAT csv` : Indique que le fichier est au format CSV  
- `HEADER true` : La première ligne contient les noms de colonnes (elle sera ignorée)

#### Options courantes de COPY

PostgreSQL offre de nombreuses options pour personnaliser l'importation :

```sql
COPY employes (nom, prenom, email, salaire)  
FROM '/tmp/employes.csv'  
WITH (  
    FORMAT csv,              -- Format du fichier
    HEADER true,             -- Première ligne = en-têtes
    DELIMITER ',',           -- Séparateur (virgule par défaut)
    QUOTE '"',               -- Caractère de citation
    ESCAPE '"',              -- Caractère d'échappement
    NULL 'NULL',             -- Représentation de NULL dans le fichier
    ENCODING 'UTF8'          -- Encodage du fichier
);
```

### Gestion des cas particuliers

#### Fichier sans en-tête

Si votre fichier ne contient pas de ligne d'en-tête :

```sql
COPY employes (nom, prenom, email, salaire)  
FROM '/tmp/employes_sans_entete.csv'  
WITH (FORMAT csv, HEADER false);  
```

#### Séparateur personnalisé (TSV, pipe, etc.)

Pour un fichier délimité par des tabulations (TSV) :

```sql
COPY employes (nom, prenom, email)  
FROM '/tmp/employes.tsv'  
WITH (FORMAT csv, DELIMITER E'\t');  
```

Pour un fichier délimité par des pipes (|) :

```sql
COPY employes (nom, prenom, email)  
FROM '/tmp/employes.txt'  
WITH (FORMAT csv, DELIMITER '|');  
```

#### Gestion des valeurs NULL

Par défaut, une chaîne vide ('') dans un fichier CSV est importée comme chaîne vide, pas comme NULL. Pour traiter certaines valeurs comme NULL :

```sql
COPY employes (nom, prenom, salaire)  
FROM '/tmp/employes.csv'  
WITH (FORMAT csv, NULL 'NULL');  
```

Dans le fichier, les cellules contenant exactement `NULL` seront importées comme NULL SQL.

#### Gestion des guillemets et caractères spéciaux

Si vos données contiennent des virgules ou des retours à la ligne, utilisez des guillemets :

Fichier `employes_complexe.csv` :
```csv
nom,prenom,commentaire
"Dupont",Marie,"Excellente employée, très motivée"
"Martin",Pierre,"Expert en SQL
et PostgreSQL"
```

Import :
```sql
COPY employes (nom, prenom, commentaire)  
FROM '/tmp/employes_complexe.csv'  
WITH (FORMAT csv, HEADER true, QUOTE '"');  
```

### Format texte (format par défaut de PostgreSQL)

Le format texte est le format historique de PostgreSQL, moins courant que CSV mais toujours supporté :

```sql
COPY employes (nom, prenom, email)  
FROM '/tmp/employes.txt'  
WITH (FORMAT text, DELIMITER E'\t', NULL '\N');  
```

Caractéristiques :
- Délimiteur : tabulation par défaut
- NULL représenté par `\N`
- Moins flexible que CSV pour les cas complexes

### Format binaire (le plus performant)

Pour des performances maximales, notamment avec des données volumineuses :

```sql
COPY employes (nom, prenom, salaire)  
FROM '/tmp/employes.bin'  
WITH (FORMAT binary);  
```

**Avantages** :
- Vitesse d'importation maximale
- Pas de conversion de types
- Données stockées directement au format interne PostgreSQL

**Inconvénients** :
- Fichiers non lisibles par un humain
- Dépendance à l'architecture (peu portable entre systèmes différents)
- Difficile à générer manuellement

### Exportation avec COPY (bonus)

`COPY` permet aussi d'exporter des données :

```sql
-- Exporter toute la table
COPY employes TO '/tmp/export_employes.csv'  
WITH (FORMAT csv, HEADER true);  

-- Exporter avec une requête
COPY (
    SELECT nom, prenom, salaire
    FROM employes
    WHERE salaire > 45000
    ORDER BY salaire DESC
) TO '/tmp/employes_hauts_salaires.csv'
WITH (FORMAT csv, HEADER true);
```

### Nouveautés PostgreSQL 18 : Amélioration de COPY

PostgreSQL 18 apporte des améliorations importantes à la commande COPY :

#### 1. Gestion améliorée du marqueur de fin `\.` en CSV

Dans les versions précédentes, le marqueur de fin `\.` (backslash point) utilisé dans le format texte pouvait causer des problèmes s'il apparaissait dans les données CSV. PostgreSQL 18 améliore la gestion de ce cas :

```sql
-- Désormais plus robuste face au \. dans les données CSV
COPY employes FROM '/tmp/data.csv'  
WITH (FORMAT csv);  
```

Cette amélioration rend COPY plus fiable lors de l'importation de fichiers CSV contenant des caractères spéciaux.

#### 2. Meilleures performances

PostgreSQL 18 optimise le traitement interne de COPY, rendant les importations encore plus rapides, particulièrement pour :
- Les fichiers volumineux (plusieurs Go)
- Les types de données complexes (JSONB, ARRAY)
- Les tables avec de nombreux index

### Comparaison de performances : INSERT vs COPY

Voici des ordres de grandeur typiques pour l'insertion d'1 million de lignes :

| Méthode | Temps approximatif | Cas d'usage |
|---------|-------------------|-------------|
| INSERT simple (1M requêtes) | ~2h-4h | À éviter pour du volume |
| INSERT multiple (1000 lignes/requête) | ~5-10 min | Migration, petits volumes |
| COPY (format CSV) | ~10-30 sec | Import massif (recommandé) |
| COPY (format binaire) | ~5-15 sec | Import massif haute performance |

> **Conclusion** : COPY peut être **100 à 500 fois plus rapide** que des INSERT pour de gros volumes.

### Bonnes pratiques pour les imports massifs

#### 1. Désactiver temporairement les index

Pour des imports très volumineux, recréer les index après l'import est souvent plus rapide :

```sql
-- Avant l'import
DROP INDEX IF EXISTS idx_employes_email;  
DROP INDEX IF EXISTS idx_employes_nom;  

-- Import massif
COPY employes FROM '/tmp/gros_fichier.csv' WITH (FORMAT csv);

-- Recréer les index
CREATE INDEX idx_employes_email ON employes(email);  
CREATE INDEX idx_employes_nom ON employes(nom);  
```

#### 2. Désactiver temporairement les contraintes (avec précaution)

Pour des imports de données déjà validées :

```sql
-- Désactiver les triggers
ALTER TABLE employes DISABLE TRIGGER ALL;

-- Import
COPY employes FROM '/tmp/data.csv' WITH (FORMAT csv);

-- Réactiver les triggers
ALTER TABLE employes ENABLE TRIGGER ALL;
```

> **Attention** : À utiliser uniquement si vous êtes certain de la qualité des données.

#### 3. Augmenter temporairement maintenance_work_mem

```sql
-- Augmenter la mémoire pour les opérations de maintenance
SET maintenance_work_mem = '1GB';

-- Effectuer l'import
COPY employes FROM '/tmp/data.csv' WITH (FORMAT csv);

-- Réinitialiser (ou laisser la session se fermer)
RESET maintenance_work_mem;
```

#### 4. Utiliser des transactions

Encapsuler l'import dans une transaction pour garantir l'atomicité :

```sql
BEGIN;

COPY employes FROM '/tmp/data1.csv' WITH (FORMAT csv);  
COPY departements FROM '/tmp/data2.csv' WITH (FORMAT csv);  
COPY affectations FROM '/tmp/data3.csv' WITH (FORMAT csv);  

COMMIT;
```

Si une erreur survient, toutes les insertions sont annulées.

---

## 6.1.5. Gestion des erreurs et validation

### Contraintes et INSERT

Lors d'une insertion, PostgreSQL vérifie toutes les contraintes :

```sql
-- Cette insertion échouera si l'email existe déjà (contrainte UNIQUE)
INSERT INTO employes (nom, prenom, email)  
VALUES ('Test', 'Test', 'marie.dupont@example.com');  
-- ERROR: duplicate key value violates unique constraint "employes_email_key"

-- Cette insertion échouera si nom est NULL (contrainte NOT NULL)
INSERT INTO employes (prenom, email)  
VALUES ('Jean', 'jean@example.com');  
-- ERROR: null value in column "nom" violates not-null constraint
```

### Comportement transactionnel

Par défaut, chaque commande SQL est une transaction :

```sql
-- Insertion réussie (automatiquement committée)
INSERT INTO employes (nom, prenom) VALUES ('Test1', 'Test1');

-- Insertion échouée (automatiquement rollback)
INSERT INTO employes (nom, prenom, email) VALUES ('Test2', 'Test2', 'marie.dupont@example.com');
-- La première insertion reste en base, la seconde est annulée
```

Pour grouper plusieurs insertions dans une transaction :

```sql
BEGIN;

INSERT INTO employes (nom, prenom) VALUES ('Test1', 'Test1');  
INSERT INTO employes (nom, prenom) VALUES ('Test2', 'Test2');  
INSERT INTO employes (nom, prenom) VALUES ('Test3', 'Test3');  

-- Si tout s'est bien passé
COMMIT;

-- Ou en cas de problème
-- ROLLBACK;
```

### Gestion des erreurs avec COPY

Contrairement à INSERT, COPY s'arrête à la première erreur par défaut :

```sql
COPY employes FROM '/tmp/data.csv' WITH (FORMAT csv);
-- Si la ligne 5000 contient une erreur, les 4999 premières lignes ne seront PAS insérées
```

**Solution** : Valider vos données avant l'import, ou utiliser une table de staging :

```sql
-- 1. Créer une table temporaire sans contraintes
CREATE TEMP TABLE employes_staging (
    nom TEXT,
    prenom TEXT,
    email TEXT,
    salaire TEXT  -- Notez : TEXT pour accepter n'importe quoi
);

-- 2. Importer dans la table temporaire
COPY employes_staging FROM '/tmp/data.csv' WITH (FORMAT csv);

-- 3. Valider et insérer dans la table finale
INSERT INTO employes (nom, prenom, email, salaire)  
SELECT  
    nom,
    prenom,
    email,
    salaire::NUMERIC  -- Conversion avec gestion d'erreur
FROM employes_staging  
WHERE email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'  -- Validation  
    AND salaire ~ '^[0-9]+\.?[0-9]*$';  -- Validation numérique
```

---

## Récapitulatif et recommandations

### Quand utiliser quelle méthode ?

| Situation | Méthode recommandée | Raison |
|-----------|---------------------|--------|
| 1 à 10 lignes | INSERT simple | Simplicité, clarté |
| 10 à 1000 lignes | INSERT multiple | Bon compromis performance/lisibilité |
| 1000 à 10000 lignes | INSERT multiple ou COPY | COPY commence à être intéressant |
| 10000+ lignes | COPY (CSV) | Performance maximale |
| Millions de lignes | COPY (binaire) + optimisations | Performance critique |
| Copie entre tables | INSERT ... SELECT | Transformation à la volée possible |

### Points clés à retenir

1. **INSERT simple** : Pour des insertions ponctuelles, lisible et flexible  
2. **INSERT multiple** : Toujours préférer à plusieurs INSERT simples  
3. **INSERT ... SELECT** : Idéal pour dupliquer ou transformer des données existantes  
4. **COPY** : L'outil de référence pour les imports massifs  
5. **PostgreSQL 18** : Améliorations de COPY pour plus de robustesse

### Checklist avant un import massif

- [ ] Les types de données correspondent-ils à la structure de la table ?  
- [ ] Les contraintes (PK, FK, UNIQUE) sont-elles compatibles avec les données ?  
- [ ] L'encodage du fichier est-il correct (UTF-8 recommandé) ?  
- [ ] Les index peuvent-ils être temporairement supprimés ?  
- [ ] Une table de staging est-elle nécessaire pour valider les données ?  
- [ ] Une transaction globale est-elle nécessaire ?  
- [ ] Les performances du système permettent-elles l'import (I/O, mémoire) ?

---

## Exemple complet : Pipeline d'importation

Voici un exemple complet d'importation de données avec validation :

```sql
-- 1. Créer la table finale
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    salaire NUMERIC(10, 2) CHECK (salaire > 0),
    date_embauche DATE DEFAULT CURRENT_DATE
);

-- 2. Créer une table de staging (sans contraintes)
CREATE TEMP TABLE employes_staging (
    nom TEXT,
    prenom TEXT,
    email TEXT,
    salaire TEXT,
    date_embauche TEXT
);

-- 3. Importer les données brutes
COPY employes_staging  
FROM '/tmp/employes.csv'  
WITH (FORMAT csv, HEADER true);  

-- 4. Analyser les données invalides
SELECT
    nom, prenom, email, salaire,
    CASE
        WHEN nom IS NULL OR nom = '' THEN 'Nom manquant'
        WHEN prenom IS NULL OR prenom = '' THEN 'Prénom manquant'
        WHEN email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$' THEN 'Email invalide'
        WHEN salaire !~ '^[0-9]+\.?[0-9]*$' THEN 'Salaire invalide'
        ELSE 'OK'
    END AS statut_validation
FROM employes_staging  
WHERE nom IS NULL OR nom = ''  
   OR prenom IS NULL OR prenom = ''
   OR email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'
   OR salaire !~ '^[0-9]+\.?[0-9]*$';

-- 5. Insérer uniquement les données valides
INSERT INTO employes (nom, prenom, email, salaire, date_embauche)  
SELECT  
    TRIM(nom),
    TRIM(prenom),
    LOWER(TRIM(email)),
    salaire::NUMERIC,
    COALESCE(date_embauche::DATE, CURRENT_DATE)
FROM employes_staging  
WHERE nom IS NOT NULL AND nom != ''  
  AND prenom IS NOT NULL AND prenom != ''
  AND email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'
  AND salaire ~ '^[0-9]+\.?[0-9]*$';

-- 6. Vérifier le résultat
SELECT
    (SELECT COUNT(*) FROM employes_staging) AS lignes_importees,
    (SELECT COUNT(*) FROM employes) AS lignes_valides,
    (SELECT COUNT(*) FROM employes_staging) - (SELECT COUNT(*) FROM employes) AS lignes_rejetees;
```

---

## Conclusion

L'insertion de données dans PostgreSQL offre une grande flexibilité :

- **INSERT** pour les opérations courantes et les petits volumes  
- **INSERT multiple** pour améliorer les performances avec des volumes moyens  
- **INSERT ... SELECT** pour transformer et dupliquer des données  
- **COPY** pour les imports massifs avec des performances optimales

Avec PostgreSQL 18, les améliorations apportées à COPY renforcent encore sa robustesse pour traiter des fichiers CSV complexes.

Le choix de la méthode dépend avant tout du volume de données, des contraintes de performance, et de la complexité des validations nécessaires.

---


⏭️ [Nouveauté PG 18 : Améliorations COPY et gestion du marqueur de fin \. en CSV](/06-manipulation-des-donnees/02-ameliorations-copy-pg18.md)
