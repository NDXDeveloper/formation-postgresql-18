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
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    salaire NUMERIC(10, 2),
    date_embauche DATE DEFAULT CURRENT_DATE
);
```

> 📌 **`GENERATED AS IDENTITY` vs `SERIAL`** : `SERIAL` (et `BIGSERIAL`) reste supporté mais c'est un *raccourci historique* qui crée en sous-main une séquence et un `DEFAULT nextval(...)`. La syntaxe `GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY`, **conforme au standard SQL** et disponible depuis PostgreSQL 10, est désormais recommandée : meilleurs droits par défaut, séquence détruite avec la colonne, et `ALWAYS` interdit toute insertion explicite (sécurité). Vous croiserez encore `SERIAL` dans beaucoup de bases existantes — c'est normal, et il fonctionne toujours parfaitement.

> ⚠️ **Piège classique : les séquences ne sont pas transactionnelles**  
>  
> Quand un `INSERT` consomme une valeur de séquence (`IDENTITY` ou `SERIAL`), cette consommation est **immédiate** et **non annulée** par un `ROLLBACK`. Conséquence : vos `id` peuvent comporter des **trous**.  
>
> ```sql
> BEGIN;
> INSERT INTO employes (nom, prenom) VALUES ('Test', 'A');   -- id = 1
> INSERT INTO employes (nom, prenom) VALUES ('Test', 'B');   -- id = 2
> ROLLBACK;
>
> INSERT INTO employes (nom, prenom) VALUES ('Réel', 'C');   -- id = 3, pas 1 !
> ```
>  
> C'est **voulu** : si les séquences étaient transactionnelles, deux transactions concurrentes devraient se sérialiser autour de la séquence, anéantissant le débit d'insertion. Ne tentez pas de combler manuellement ces trous avec `ALTER SEQUENCE … RESTART` en production : vous risquez des collisions avec des `id` déjà attribués mais non encore *committés* par d'autres sessions.

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
- La colonne `id` sera générée automatiquement (la séquence associée à l'`IDENTITY` est avancée)
- La colonne `salaire` recevra `NULL` (puisqu'elle est *nullable*)
- La colonne `date_embauche` prendra la valeur par défaut (`CURRENT_DATE`)

### Insertion sans spécifier les colonnes

Si vous fournissez des valeurs pour **toutes** les colonnes dans le bon ordre, vous pouvez omettre la liste des colonnes :

```sql
INSERT INTO employes  
VALUES (DEFAULT, 'Durand', 'Sophie', 'sophie.durand@example.com', 48000.00, '2025-02-01');  
```

Le mot-clé `DEFAULT` pour `id` force l'utilisation de la valeur par défaut de la colonne — ici la prochaine valeur de la séquence d'`IDENTITY`. Si la colonne était déclarée `GENERATED ALWAYS AS IDENTITY`, il faut **soit** utiliser `DEFAULT`, **soit** ajouter `OVERRIDING SYSTEM VALUE` pour fournir une valeur explicite (ce dernier cas étant réservé à des imports/migrations).

> ⚠️ **Cette syntaxe sans liste de colonnes est déconseillée en production** : elle rend le code fragile. Si la structure de la table change (ajout/suppression/réordonnancement de colonne), votre requête échouera silencieusement ou insérera les valeurs dans les mauvaises colonnes. Préférez toujours nommer les colonnes explicitement.

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

`COPY` est une commande PostgreSQL spécialement conçue pour l'importation et l'exportation de données en masse. Elle est **beaucoup plus rapide** qu'une succession d'`INSERT` pour les raisons suivantes :

1. **Pas de re-parsing ligne par ligne** : `COPY` lit son propre format compact et écrit directement les tuples (le moteur SQL n'est invoqué qu'une fois pour la commande globale).  
2. **Allocation de pages amortie** : `COPY` peut remplir une page disque entière avant d'écrire, là où un `INSERT` peut générer un appel WAL par ligne.  
3. **Moins d'allers-retours protocolaires** : un seul `Bind`/`Execute` pour des millions de lignes, contre N pour autant d'`INSERT`.  
4. **Format binaire optionnel** : le format `binary` évite les conversions texte↔interne (encore plus rapide, mais le fichier devient dépendant de l'architecture/version PostgreSQL).

#### Le chemin d'une ligne dans un `INSERT` vs un `COPY`

Pour comprendre l'écart de performance, voici ce que fait PostgreSQL pour **une seule ligne** :

```
INSERT classique :
  1. Parser SQL          (analyser le texte VALUES …)
  2. Analyseur sémantique (résoudre noms de colonnes, types)
  3. Planificateur       (choisir l'index ; cas trivial mais facturé)
  4. Exécuteur           (évaluer expressions DEFAULT, IDENTITY)
  5. Validation contraintes (NOT NULL, CHECK, FK, UNIQUE)
  6. Écriture du tuple dans le buffer
  7. Enregistrement WAL  (avant écriture page → "Write-Ahead")
  8. Réveil des éventuels triggers BEFORE/AFTER

COPY :
  Étapes 1 à 4 : faites UNE SEULE FOIS pour toute la commande
  Étapes 5 à 8 : faites par ligne, mais avec un chemin code dédié
                 et un buffering plus agressif
```

L'écart provient surtout des **étapes 1 à 4 amorties** sur tout le flux : sur des millions de lignes, on économise des millions de parsings et planifications.

### Quand utiliser COPY ?

- **Import initial** : Chargement de millions de lignes lors de la création d'une base  
- **Migrations** : Transfert de données entre systèmes  
- **Chargements ETL** : Extraction, transformation, chargement de données  
- **Restaurations** : Rechargement de sauvegardes

**Règle empirique** : À partir de quelques milliers de lignes, COPY devient significativement plus performant que INSERT.

### Syntaxe de base

Il existe deux variantes de COPY :

#### 1. `COPY` côté serveur (depuis un fichier sur le serveur)

```sql
COPY nom_table (colonne1, colonne2, colonne3)  
FROM '/chemin/vers/fichier.csv'  
WITH (FORMAT csv, HEADER true, DELIMITER ',');  
```

> ⚠️ **Important** : le fichier doit être accessible **par le processus PostgreSQL** (utilisateur système `postgres`), pas par le client SQL. Cette commande nécessite par défaut des privilèges de superutilisateur. Depuis PostgreSQL 11, on peut aussi accorder l'accès via les rôles prédéfinis `pg_read_server_files` (pour `COPY FROM`) et `pg_write_server_files` (pour `COPY TO`), sans devoir donner les droits superutilisateur complets :  
>
> ```sql
> GRANT pg_read_server_files TO mon_role_import;
> ```

#### 2. `\copy` côté client (depuis psql)

```text
\copy nom_table (colonne1, colonne2, colonne3) FROM '/chemin/local/fichier.csv' WITH (FORMAT csv, HEADER true, DELIMITER ',')
```

> 📌 **Note** : `\copy` est une **commande client psql** (préfixée par `\`), pas une commande SQL. Elle doit tenir **sur une seule ligne logique** (psql ne reconnaît pas le multi-ligne pour les méta-commandes, sauf à terminer chaque ligne par `\`). Elle transfère les données depuis le **poste client** vers le serveur via la connexion psql : aucun privilège spécial côté serveur n'est requis, juste les droits `INSERT` sur la table. C'est l'option à privilégier quand le fichier réside sur votre poste et non sur le serveur.

#### 3. `COPY … FROM PROGRAM` — exécuter une commande shell

PostgreSQL peut alimenter un `COPY` en exécutant **un programme externe côté serveur** dont la sortie standard sert d'entrée :

```sql
-- Importer la sortie d'un script de génération
COPY logs (ip, methode, url, code_retour)  
FROM PROGRAM '/opt/scripts/extraire_logs.sh hier'  
WITH (FORMAT csv, HEADER true);  

-- Décompresser à la volée sans fichier temporaire
COPY mesures FROM PROGRAM 'zcat /data/mesures.csv.gz'  
WITH (FORMAT csv, HEADER true);  
```

> 🔐 **Risque de sécurité majeur** : `FROM PROGRAM` exécute du shell **avec les droits de l'utilisateur `postgres`** sur le serveur. Réservé aux superutilisateurs ou aux membres du rôle prédéfini `pg_execute_server_program`. À éviter à tout prix dans une application web (injection trivialement catastrophique).

#### 4. Pipe inter-bases via psql

Pour copier d'une base à une autre **sans fichier intermédiaire**, on enchaîne deux psql via un pipe shell :

```bash
# Du serveur "source" vers le serveur "cible", format binaire pour aller vite
psql -h source.example.com -d mabase \
     -c "COPY (SELECT * FROM gros_table WHERE date_creation > '2025-01-01') TO STDOUT WITH (FORMAT binary)" \
  | psql -h cible.example.com -d mabase \
     -c "COPY gros_table FROM STDIN WITH (FORMAT binary)"
```

C'est typiquement le moyen le plus rapide pour transférer une partie d'une grosse table d'une base à une autre.

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

### Nouveautés PostgreSQL 18 : Améliorations de `COPY`

PostgreSQL 18 apporte plusieurs améliorations à `COPY`, détaillées dans la section [6.2](/06-manipulation-des-donnees/02-ameliorations-copy-pg18.md). En résumé :

1. **Marqueur `\.`** : `COPY FROM` lisant depuis un **fichier** ne reconnaît plus `\.` comme marqueur de fin de données (en CSV comme en text). Le marqueur n'est conservé que pour `STDIN` lu via psql, pour la compatibilité des scripts.
2. **`ON_ERROR 'ignore'`** (déjà PG 17) couplé à **`REJECT_LIMIT n`** (nouveau PG 18) : permet de poursuivre l'import en ignorant les lignes invalides jusqu'à `n` rejets maximum.
3. **`LOG_VERBOSITY 'silent'`** (nouveau PG 18) : supprime le journal des lignes ignorées par `ON_ERROR 'ignore'`.
4. **`COPY (SELECT … FROM mv)`** : `COPY TO` accepte désormais une sous-requête sur une **vue matérialisée** peuplée comme source de données.
5. **Restriction `FREEZE` sur tables étrangères** : `COPY FREEZE` est explicitement refusé sur les *foreign tables* (auparavant accepté mais silencieusement ignoré).

### Comparaison de performances : `INSERT` vs `COPY`

Les chiffres ci-dessous sont des **ordres de grandeur** indicatifs pour l'insertion d'**1 million de lignes** dans une table simple, sur un poste de développement courant. La performance réelle dépend du matériel (CPU, type de disque SSD/NVMe, RAM disponible), du nombre d'index, du type des colonnes, et de la latence réseau entre client et serveur :

| Méthode | Temps typique | Cas d'usage |
|---------|---------------|-------------|
| `INSERT` simple (1 M requêtes séparées) | plusieurs heures | À proscrire pour le volume |
| `INSERT` multi-valeurs (lots de ~1000 lignes) | quelques minutes | Migration, volumes modérés |
| `COPY` (format CSV) | quelques dizaines de secondes | Import massif (par défaut) |
| `COPY` (format binaire) | encore quelques secondes plus rapide | Quand source et cible sont toutes deux PostgreSQL |

> 🚀 **Conclusion** : pour de gros volumes, `COPY` est typiquement de **un à deux ordres de grandeur** plus rapide qu'un torrent d'`INSERT` un-par-un. L'écart se réduit pour des `INSERT` multi-valeurs bien groupés, mais `COPY` reste presque toujours la solution la plus rapide pour des imports massifs.

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

#### 5. L'option `FREEZE` : pré-marquer les tuples pour éviter un futur VACUUM

Quand vous chargez une **table vide** dans une transaction qui l'a aussi créée (ou tronquée), `COPY … FREEZE` marque immédiatement les tuples comme « gelés » (visibles par toutes les futures transactions) :

```sql
BEGIN;

-- La table doit être créée OU tronquée dans la MÊME transaction que le COPY FREEZE
TRUNCATE TABLE gros_import;

COPY gros_import FROM '/data/init.csv'  
WITH (FORMAT csv, HEADER true, FREEZE);  

COMMIT;
```

**Pourquoi c'est utile** : sans `FREEZE`, les millions de lignes que vous venez d'insérer seront re-balayées plus tard par `VACUUM` quand leur `xmin` atteindra le seuil de `vacuum_freeze_min_age` (par défaut 50 millions de transactions plus tard). Sur des chargements initiaux de gros volumes, c'est une économie majeure de coût de maintenance ultérieur.

**Restrictions** :
- La table doit être vide **et** la transaction doit être celle qui a créé ou tronqué la table (PostgreSQL doit garantir qu'aucune autre transaction ne peut voir l'état antérieur).
- Aucun autre client ne doit voir la table avant le `COMMIT` (PostgreSQL le vérifie : sinon la commande échoue).
- ❌ Ne fonctionne **pas** sur les *foreign tables* (PG 18 rend cette restriction explicite — cf. section 6.2).

#### 6. Paramètres système à connaître pour les imports massifs

| Paramètre | Effet | Recommandation pendant un import massif |
|-----------|-------|-----------------------------------------|
| `maintenance_work_mem` | Mémoire pour `CREATE INDEX`, `VACUUM`, etc. | Monter à plusieurs Go le temps de l'import |
| `synchronous_commit` | Si `off`, `COMMIT` ne bloque pas l'écriture WAL sur disque | `off` accélère, **au prix** d'une fenêtre de perte sur crash (acceptable pour un *staging* rejouable) |
| `max_wal_size` | Plafond avant *checkpoint* forcé | Monter (ex : 4 Go) pour éviter des checkpoints fréquents |
| `wal_compression` | Compression des **images de pages complètes** (FPI) écrites dans le WAL — pas de tout le WAL | `lz4` ou `zstd` (PG 15+) ; gain net sur le débit disque |
| `autovacuum` | Maintenance automatique en arrière-plan | Ne **pas** désactiver : laisser tourner, ou si vraiment nécessaire, désactiver pour la table cible précise via `ALTER TABLE … SET (autovacuum_enabled = false)` puis lancer `VACUUM ANALYZE` manuellement à la fin |

> ⚠️ Tout changement de paramètre serveur impacte **toute l'instance**. Préférez les paramètres `SET LOCAL …` à l'intérieur d'une transaction, ou faites les modifications pendant une fenêtre de maintenance.

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

### Gestion des erreurs avec `COPY`

Par défaut, `COPY` est **transactionnel** : si une seule ligne échoue (type invalide, contrainte violée, etc.), toute la commande est annulée et **aucune** ligne n'est insérée :

```sql
COPY employes FROM '/tmp/data.csv' WITH (FORMAT csv);
-- Si la ligne 5000 contient une erreur, les 4999 premières lignes sont rollback :
-- la table reste exactement comme avant la commande.
```

#### Option `ON_ERROR` (PostgreSQL 17+)

Depuis PostgreSQL 17, l'option `ON_ERROR 'ignore'` permet de passer outre les lignes invalides au lieu d'avorter :

```sql
COPY employes FROM '/tmp/data.csv'  
WITH (FORMAT csv, HEADER true, ON_ERROR 'ignore');  
-- Les lignes invalides (mauvais type, conversion impossible) sont ignorées
-- Les lignes valides sont importées
```

À l'issue de la commande, un message `NOTICE` indique le nombre de lignes ignorées. Cette option ne contourne **pas** les violations de contraintes (`NOT NULL`, `UNIQUE`, clés étrangères) : elle ne capte que les **erreurs de conversion de type** sur les colonnes.

#### `REJECT_LIMIT` et `LOG_VERBOSITY` (PostgreSQL 18)

PostgreSQL 18 ajoute deux options qui complètent `ON_ERROR` :

```sql
-- Limiter à 100 lignes invalides maximum ; au-delà, COPY échoue
COPY employes FROM '/tmp/data.csv'  
WITH (FORMAT csv, HEADER true,  
      ON_ERROR 'ignore',  
      REJECT_LIMIT 100);  

-- Importer sans journal verbeux pour les lignes ignorées
COPY employes FROM '/tmp/data.csv'  
WITH (FORMAT csv, HEADER true,  
      ON_ERROR 'ignore',  
      LOG_VERBOSITY 'silent');  
```

Ces options sont détaillées en section [6.2](/06-manipulation-des-donnees/02-ameliorations-copy-pg18.md).

#### Stratégie alternative : table de *staging*

Une approche **portable** (toutes versions) consiste à charger les données brutes dans une table intermédiaire sans contraintes, puis à valider/transformer avant d'insérer dans la table finale :

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
WHERE email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'  -- Validation  
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
- [ ] Une table de *staging* est-elle nécessaire pour valider les données ?  
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
        WHEN email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN 'Email invalide'
        WHEN salaire !~ '^[0-9]+\.?[0-9]*$' THEN 'Salaire invalide'
        ELSE 'OK'
    END AS statut_validation
FROM employes_staging  
WHERE nom IS NULL OR nom = ''  
   OR prenom IS NULL OR prenom = ''
   OR email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
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
  AND email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
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
