🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5. Requêtes de Sélection (DQL)

## Introduction au chapitre

Bienvenue dans le chapitre le plus fondamental de votre apprentissage de PostgreSQL : les **requêtes de sélection**. C'est ici que commence véritablement votre voyage dans le monde des bases de données relationnelles.

Si vous deviez maîtriser un seul aspect de SQL, ce serait celui-ci. Pourquoi ? Parce que **90% de votre travail quotidien avec PostgreSQL consistera à interroger des données** : les lire, les filtrer, les trier, les analyser, les transformer.

Dans ce chapitre, nous allons explorer en profondeur le **DQL (Data Query Language)**, c'est-à-dire le langage d'interrogation de données, dont la pierre angulaire est la clause `SELECT`.

---

## Qu'est-ce que le DQL ?

### Les différentes familles de SQL

Le langage SQL se divise en plusieurs catégories selon le type d'opérations effectuées :

| Catégorie | Nom complet | Rôle | Commandes principales |
|-----------|-------------|------|----------------------|
| **DQL** | Data Query Language | **Interroger** les données | `SELECT` |
| **DML** | Data Manipulation Language | **Modifier** les données | `INSERT`, `UPDATE`, `DELETE` |
| **DDL** | Data Definition Language | **Définir** la structure | `CREATE`, `ALTER`, `DROP` |
| **DCL** | Data Control Language | **Contrôler** les accès | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | **Gérer** les transactions | `BEGIN`, `COMMIT`, `ROLLBACK` |

Le **DQL** se concentre exclusivement sur la **lecture** des données. Il ne modifie rien, ne crée rien, ne supprime rien. Il permet simplement de **poser des questions** à votre base de données et d'obtenir des réponses.

### Pourquoi le DQL est-il si important ?

Dans la vie d'une application :
- **Lecture (DQL)** : 80-95% des requêtes  
- **Écriture (DML)** : 5-20% des requêtes

Les applications **lisent** beaucoup plus qu'elles n'écrivent. Voici pourquoi maîtriser le DQL est crucial :

1. **Performance** : Des requêtes mal écrites peuvent ralentir toute une application  
2. **Précision** : Obtenir exactement les données dont vous avez besoin, ni plus ni moins  
3. **Efficacité** : Faire en SQL ce qui prendrait des dizaines de lignes de code applicatif  
4. **Analyse** : Extraire de l'intelligence métier de vos données

---

## La clause SELECT : Votre outil principal

### L'instruction la plus importante de SQL

```sql
SELECT colonnes  
FROM table  
WHERE conditions  
ORDER BY colonne;  
```

Cette simple instruction est **la plus puissante** et **la plus utilisée** de tout SQL. Elle permet de :

- Lire des données d'une ou plusieurs tables
- Filtrer les lignes qui nous intéressent
- Calculer de nouvelles valeurs
- Trier et organiser les résultats
- Agréger et résumer l'information
- Joindre des données de sources multiples

### Anatomie d'une requête SELECT

Une requête SELECT peut contenir de nombreuses clauses, chacune ayant un rôle spécifique :

```sql
SELECT          -- Quelles colonnes afficher ?
    colonnes
FROM            -- De quelle(s) table(s) ?
    table
WHERE           -- Quelles lignes garder ?
    conditions
GROUP BY        -- Comment regrouper ?
    colonnes
HAVING          -- Quels groupes garder ?
    conditions_agrégées
ORDER BY        -- Dans quel ordre afficher ?
    colonnes
LIMIT           -- Combien de lignes maximum ?
    nombre;
```

**Ne vous inquiétez pas** si cela semble complexe ! Nous allons explorer chacune de ces clauses en détail, une par une, dans les sections suivantes.

---

## Votre première requête SELECT

Commençons par le plus simple : lire toutes les données d'une table.

### Sélectionner toutes les colonnes

```sql
SELECT * FROM clients;
```

**Décortiquons cette requête :**
- `SELECT` : Je veux lire des données  
- `*` : Toutes les colonnes (l'astérisque signifie "tout")  
- `FROM clients` : De la table nommée "clients"

**Résultat hypothétique :**
```
id | nom      | prenom  | email                | ville      | age
---|----------|---------|----------------------|------------|----
1  | Dupont   | Jean    | jean.dupont@mail.com | Paris      | 34
2  | Martin   | Sophie  | sophie.m@mail.com    | Lyon       | 28
3  | Bernard  | Pierre  | p.bernard@mail.com   | Marseille  | 45
4  | Durand   | Marie   | m.durand@mail.com    | Paris      | 31
5  | Petit    | Luc     | luc.petit@mail.com   | Toulouse   | 52
```

**Important :** Le `*` est pratique pour explorer rapidement une table, mais en production, il est recommandé de **spécifier explicitement** les colonnes dont vous avez besoin.

### Sélectionner des colonnes spécifiques

```sql
SELECT nom, prenom, ville FROM clients;
```

**Résultat :**
```
nom      | prenom  | ville
---------|---------|----------
Dupont   | Jean    | Paris  
Martin   | Sophie  | Lyon  
Bernard  | Pierre  | Marseille  
Durand   | Marie   | Paris  
Petit    | Luc     | Toulouse  
```

Maintenant nous n'affichons que les colonnes qui nous intéressent. C'est plus lisible et plus performant.

### Sélectionner avec des alias

Vous pouvez renommer les colonnes dans le résultat avec `AS` :

```sql
SELECT
    nom AS nom_famille,
    prenom AS prenom_client,
    ville AS ville_residence
FROM clients;
```

**Résultat :**
```
nom_famille | prenom_client | ville_residence
------------|---------------|----------------
Dupont      | Jean          | Paris  
Martin      | Sophie        | Lyon  
Bernard     | Pierre        | Marseille  
```

**Le mot-clé `AS` est optionnel** (mais recommandé pour la lisibilité) :

```sql
-- Ces deux syntaxes sont équivalentes
SELECT nom AS nom_famille FROM clients;  
SELECT nom nom_famille FROM clients;  
```

---

## Expressions et calculs dans SELECT

SELECT ne se limite pas à afficher des colonnes existantes. Vous pouvez **calculer** de nouvelles valeurs.

### Calculs arithmétiques

```sql
-- Table produits
-- id | nom_produit      | prix | quantite_stock
-- 1  | Laptop          | 999  | 10
-- 2  | Souris          | 25   | 150
-- 3  | Clavier         | 75   | 80

-- Calculer la valeur totale du stock pour chaque produit
SELECT
    nom_produit,
    prix,
    quantite_stock,
    prix * quantite_stock AS valeur_stock
FROM produits;
```

**Résultat :**
```
nom_produit | prix | quantite_stock | valeur_stock
------------|------|----------------|-------------
Laptop      | 999  | 10             | 9990  
Souris      | 25   | 150            | 3750  
Clavier     | 75   | 80             | 6000  
```

**Opérateurs arithmétiques disponibles :**
- `+` : Addition  
- `-` : Soustraction  
- `*` : Multiplication  
- `/` : Division  
- `%` : Modulo (reste de la division)  
- `^` : Puissance

### Concaténation de texte

```sql
-- Créer un nom complet à partir du prénom et du nom
SELECT
    prenom || ' ' || nom AS nom_complet,
    email
FROM clients;
```

**Résultat :**
```
nom_complet    | email
---------------|----------------------
Jean Dupont    | jean.dupont@mail.com  
Sophie Martin  | sophie.m@mail.com  
Pierre Bernard | p.bernard@mail.com  
```

**L'opérateur `||`** est utilisé pour concaténer (coller ensemble) des chaînes de caractères.

### Fonctions dans SELECT

PostgreSQL offre des centaines de fonctions intégrées :

```sql
-- Fonctions de texte
SELECT
    nom,
    UPPER(nom) AS nom_majuscule,
    LOWER(nom) AS nom_minuscule,
    LENGTH(nom) AS longueur_nom
FROM clients;
```

**Résultat :**
```
nom      | nom_majuscule | nom_minuscule | longueur_nom
---------|---------------|---------------|-------------
Dupont   | DUPONT        | dupont        | 6  
Martin   | MARTIN        | martin        | 6  
Bernard  | BERNARD       | bernard       | 7  
```

```sql
-- Fonctions mathématiques
SELECT
    prix,
    ROUND(prix * 1.20, 2) AS prix_ttc,
    CEIL(prix) AS prix_arrondi_superieur,
    FLOOR(prix) AS prix_arrondi_inferieur
FROM produits;
```

```sql
-- Fonctions de date
SELECT
    date_embauche,
    EXTRACT(YEAR FROM date_embauche) AS annee,
    EXTRACT(MONTH FROM date_embauche) AS mois,
    AGE(CURRENT_DATE, date_embauche) AS anciennete
FROM employes;
```

---

## Valeurs littérales et constantes

Vous pouvez afficher des valeurs fixes dans votre SELECT :

```sql
-- Afficher une constante pour toutes les lignes
SELECT
    nom,
    prenom,
    'Client actif' AS statut,
    2024 AS annee
FROM clients;
```

**Résultat :**
```
nom      | prenom  | statut        | annee
---------|---------|---------------|------
Dupont   | Jean    | Client actif  | 2024  
Martin   | Sophie  | Client actif  | 2024  
Bernard  | Pierre  | Client actif  | 2024  
```

**Types de valeurs littérales :**
- **Texte** : Entre apostrophes `'Bonjour'`  
- **Nombres** : Sans délimiteurs `42`, `3.14`  
- **Booléens** : `TRUE`, `FALSE`  
- **Dates** : `'2024-11-19'`, `'2024-11-19 14:30:00'`  
- **NULL** : Valeur inconnue/absente

---

## Expressions conditionnelles : CASE

L'expression `CASE` permet d'appliquer une logique conditionnelle dans SELECT :

```sql
SELECT
    nom,
    age,
    CASE
        WHEN age < 18 THEN 'Mineur'
        WHEN age >= 18 AND age < 65 THEN 'Adulte'
        ELSE 'Senior'
    END AS categorie_age
FROM clients;
```

**Résultat :**
```
nom      | age | categorie_age
---------|-----|---------------
Dupont   | 34  | Adulte  
Martin   | 28  | Adulte  
Bernard  | 45  | Adulte  
Durand   | 31  | Adulte  
Petit    | 52  | Adulte  
Legrand  | 17  | Mineur  
Moreau   | 68  | Senior  
```

**Syntaxe de CASE :**
```sql
CASE
    WHEN condition1 THEN resultat1
    WHEN condition2 THEN resultat2
    WHEN condition3 THEN resultat3
    ELSE resultat_par_defaut
END
```

L'expression `CASE` évalue les conditions dans l'ordre et retourne le résultat de la **première condition vraie**. Si aucune condition n'est vraie, elle retourne la valeur du `ELSE` (ou `NULL` si pas de `ELSE`).

---

## La clause FROM : D'où viennent les données ?

### Table unique

Le cas le plus simple : lire depuis une seule table.

```sql
SELECT * FROM employes;
```

### Plusieurs tables (aperçu)

Vous pouvez lire depuis plusieurs tables simultanément (nous verrons les jointures en détail plus tard) :

```sql
SELECT
    employes.nom,
    departements.nom_departement
FROM employes, departements  
WHERE employes.departement_id = departements.id;  
```

### Sous-requêtes dans FROM

Vous pouvez utiliser le résultat d'une autre requête comme source de données :

```sql
SELECT nom, salaire_annuel  
FROM (  
    SELECT nom, salaire * 12 AS salaire_annuel
    FROM employes
) AS employes_annuel
WHERE salaire_annuel > 50000;
```

Nous approfondirons ces concepts avancés dans les sections suivantes.

---

## Requête la plus simple : Sans table !

PostgreSQL permet de faire des SELECT **sans table** pour effectuer des calculs simples :

```sql
-- Calculer une expression
SELECT 2 + 2;

-- Résultat : 4

-- Date et heure actuelles
SELECT CURRENT_DATE, CURRENT_TIME, NOW();

-- Résultat : 2024-11-19 | 14:30:45.123 | 2024-11-19 14:30:45.123

-- Tester une fonction
SELECT UPPER('bonjour');

-- Résultat : BONJOUR

-- Calculs complexes
SELECT
    10 * 5 AS multiplication,
    SQRT(144) AS racine_carree,
    PI() AS pi_value,
    RANDOM() AS nombre_aleatoire;
```

C'est pratique pour :
- Tester des fonctions
- Faire des calculs rapides
- Vérifier la connexion à la base de données
- Obtenir des informations système

---

## Structure de ce chapitre

Maintenant que vous avez une vue d'ensemble, voici ce que nous allons couvrir dans les sections suivantes :

### 5.1. L'ordre d'exécution logique d'une requête SQL
Comprendre **dans quel ordre** PostgreSQL traite réellement votre requête. C'est fondamental pour éviter les erreurs et écrire des requêtes efficaces.

### 5.2. Filtrage (WHERE) et logique booléenne
Apprendre à **sélectionner uniquement** les lignes qui vous intéressent avec des conditions précises.

### 5.3. Le piège du NULL : Logique ternaire et fonctions de gestion
Maîtriser les valeurs `NULL` (absentes/inconnues) qui se comportent de manière particulière en SQL.

### 5.4. Tri (ORDER BY) et gestion des NULLs
**Organiser vos résultats** dans l'ordre qui vous convient (alphabétique, numérique, chronologique).

### 5.5. Pagination et limitation (LIMIT, OFFSET)
Afficher les résultats **page par page** plutôt que tout d'un coup.

### 5.6. DISTINCT et élimination des doublons
Obtenir des **valeurs uniques** sans doublons.

---

## Bonnes pratiques dès le départ

Avant de plonger dans les détails, voici quelques bonnes pratiques à adopter dès maintenant :

### 1. Spécifiez les colonnes explicitement

```sql
-- ❌ À éviter en production
SELECT * FROM clients;

-- ✅ Préférez ceci
SELECT id, nom, prenom, email FROM clients;
```

**Pourquoi ?**
- Plus clair (on sait exactement ce qu'on récupère)
- Plus performant (moins de données transférées)
- Plus robuste (si la table change, votre requête continue de fonctionner)

### 2. Utilisez des alias pour la lisibilité

```sql
-- ✅ Bon : alias explicites
SELECT
    c.nom AS nom_client,
    c.email AS email_contact,
    v.ville AS ville_residence
FROM clients c  
JOIN villes v ON c.ville_id = v.id;  
```

### 3. Indentez et formatez vos requêtes

```sql
-- ❌ Difficile à lire
SELECT nom,prenom,email FROM clients WHERE ville='Paris' ORDER BY nom;

-- ✅ Facile à lire
SELECT
    nom,
    prenom,
    email
FROM clients  
WHERE ville = 'Paris'  
ORDER BY nom;  
```

### 4. Commentez les requêtes complexes

```sql
-- Rapport mensuel des ventes par région
-- Inclut uniquement les régions avec plus de 10 ventes
SELECT
    region,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS total_ventes
FROM ventes  
WHERE date_vente >= '2024-01-01'  
GROUP BY region  
HAVING COUNT(*) > 10  
ORDER BY total_ventes DESC;  
```

**Types de commentaires :**
- `--` : Commentaire sur une ligne  
- `/* ... */` : Commentaire sur plusieurs lignes

### 5. Testez sur un échantillon d'abord

Avant d'exécuter une requête complexe sur une grande table :

```sql
-- Tester d'abord avec LIMIT
SELECT
    nom,
    calcul_complexe(colonne) AS resultat
FROM grande_table  
LIMIT 10;  -- Vérifier que ça marche sur 10 lignes  

-- Une fois validé, retirer le LIMIT si nécessaire
```

---

## Outils pour écrire et tester vos requêtes

### psql : Le client en ligne de commande

```bash
# Se connecter à une base de données
psql -U username -d nom_base

# Dans psql
\dt              -- Lister les tables
\d nom_table     -- Décrire une table
\x               -- Activer l'affichage étendu
\timing          -- Afficher le temps d'exécution
```

### pgAdmin : Interface graphique

- Navigation visuelle des bases et tables
- Éditeur SQL avec coloration syntaxique
- Visualisation graphique des plans d'exécution
- Historique des requêtes

### DBeaver : Alternative multi-plateformes

- Supporte PostgreSQL et d'autres SGBD
- Auto-complétion SQL
- Export de résultats (CSV, JSON, Excel)
- Diagrammes ER

### Extensions pour éditeurs de code

- **VS Code** : PostgreSQL, SQL Tools  
- **IntelliJ/DataGrip** : Support PostgreSQL complet  
- **Sublime Text** : PostgreSQL Syntax

---

## Votre premier exercice mental

Avant de continuer, essayez de **visualiser mentalement** ce que feraient ces requêtes :

```sql
-- Requête 1
SELECT nom, age FROM clients WHERE age > 30;

-- Requête 2
SELECT
    nom,
    prenom,
    age,
    CASE
        WHEN age >= 18 THEN 'Majeur'
        ELSE 'Mineur'
    END AS statut_legal
FROM clients;

-- Requête 3
SELECT COUNT(*) FROM employes;

-- Requête 4
SELECT DISTINCT ville FROM clients ORDER BY ville;
```

**Réponses :**
1. Liste des noms et âges des clients ayant plus de 30 ans  
2. Liste complète avec une colonne calculée indiquant si majeur ou mineur  
3. Compte le nombre total d'employés  
4. Liste des villes uniques (sans doublons), triée alphabétiquement

Si vous avez compris ces exemples, vous êtes prêt à approfondir !

---

## Vocabulaire essentiel du DQL

Avant de continuer, assurons-nous de parler le même langage :

| Terme | Définition | Exemple |
|-------|------------|---------|
| **Requête** | Instruction complète envoyée à PostgreSQL | `SELECT * FROM clients;` |
| **Clause** | Partie d'une requête (SELECT, FROM, WHERE, etc.) | `WHERE age > 30` |
| **Colonne** | Champ d'une table | `nom`, `prenom`, `age` |
| **Ligne** | Enregistrement dans une table | Une personne dans la table clients |
| **Table** | Collection de lignes et colonnes | `clients`, `employes`, `produits` |
| **Alias** | Nom alternatif pour colonne ou table | `AS nom_complet` |
| **Expression** | Calcul ou transformation | `prix * quantite` |
| **Fonction** | Opération prédéfinie | `UPPER()`, `COUNT()`, `SUM()` |
| **Prédicat** | Condition dans WHERE/HAVING | `age > 18` |
| **Résultat** | Ensemble de lignes retournées | Table temporaire en mémoire |

---

## Les erreurs fréquentes des débutants

### 1. Oublier le point-virgule

```sql
-- ❌ Erreur : pas de point-virgule
SELECT * FROM clients

-- ✅ Correct
SELECT * FROM clients;
```

Le point-virgule `;` termine une instruction SQL. Dans certains outils, il est optionnel, mais c'est une bonne pratique de toujours l'inclure.

### 2. Confondre apostrophes simples et doubles

```sql
-- ❌ Erreur : guillemets doubles pour du texte
SELECT * FROM clients WHERE nom = "Dupont";

-- ✅ Correct : apostrophes simples
SELECT * FROM clients WHERE nom = 'Dupont';

-- Note : les guillemets doubles sont pour les identifiants (noms de tables/colonnes)
SELECT * FROM "Clients" WHERE "Nom" = 'Dupont';
```

**Règle :**
- **Apostrophes simples `'...'`** : pour les valeurs textuelles  
- **Guillemets doubles `"..."`** : pour les identifiants (noms de tables/colonnes)

### 3. Casse des mots-clés

```sql
-- Ces trois requêtes sont équivalentes
SELECT nom FROM clients;  
select nom from clients;  
SeLeCt NoM fRoM cLiEnTs;  

-- Mais par convention, on écrit les mots-clés en MAJUSCULES
SELECT nom FROM clients;
```

SQL est **insensible à la casse** pour les mots-clés, mais :
- **Convention** : Mots-clés en MAJUSCULES (`SELECT`, `FROM`, `WHERE`)  
- **Noms de colonnes/tables** : en minuscules ou avec underscores (`nom_client`)

### 4. Espaces dans les noms

```sql
-- ❌ Erreur : espace dans le nom de colonne sans guillemets
SELECT nom client FROM clients;

-- ✅ Solution 1 : utiliser un underscore
SELECT nom_client FROM clients;

-- ✅ Solution 2 : utiliser des guillemets doubles
SELECT "nom client" FROM clients;

-- Recommandation : éviter les espaces, utiliser underscores
```

---

## Exemples pour débuter : Base de données simple

Imaginons une petite base de données d'une bibliothèque :

### Structure des tables

**Table `livres` :**
```
id | titre                      | auteur           | annee_publication | disponible
---|----------------------------|------------------|-------------------|------------
1  | 1984                       | George Orwell    | 1949              | true
2  | Le Petit Prince            | Saint-Exupéry    | 1943              | true
3  | Harry Potter (tome 1)      | J.K. Rowling     | 1997              | false
4  | Le Seigneur des Anneaux    | J.R.R. Tolkien   | 1954              | true
5  | Les Misérables             | Victor Hugo      | 1862              | true
```

### Requêtes d'exemple

```sql
-- 1. Tous les livres
SELECT * FROM livres;

-- 2. Titres et auteurs uniquement
SELECT titre, auteur FROM livres;

-- 3. Livres disponibles
SELECT titre, auteur FROM livres WHERE disponible = true;

-- 4. Livres publiés après 1950
SELECT titre, annee_publication FROM livres WHERE annee_publication > 1950;

-- 5. Livres triés par année
SELECT titre, annee_publication FROM livres ORDER BY annee_publication DESC;

-- 6. Âge des livres
SELECT
    titre,
    annee_publication,
    2024 - annee_publication AS age_livre
FROM livres;

-- 7. Catégorie par âge
SELECT
    titre,
    CASE
        WHEN annee_publication < 1900 THEN 'Classique ancien'
        WHEN annee_publication < 1950 THEN 'Classique moderne'
        WHEN annee_publication < 2000 THEN 'Contemporain'
        ELSE 'Récent'
    END AS categorie
FROM livres;
```

Essayez de comprendre chacune de ces requêtes avant de passer à la suite !

---

## Conclusion de l'introduction

Vous avez maintenant une **vue d'ensemble** solide de ce qu'est le DQL et de la clause SELECT. Vous comprenez :

- ✅ Que SELECT permet d'interroger des données sans les modifier  
- ✅ La structure de base d'une requête SELECT  
- ✅ Comment sélectionner des colonnes spécifiques  
- ✅ Comment calculer de nouvelles valeurs  
- ✅ Comment utiliser des alias et des expressions conditionnelles  
- ✅ Les bonnes pratiques à adopter dès le début

Dans les sections suivantes, nous allons approfondir chaque aspect de SELECT :
- Comment PostgreSQL exécute réellement vos requêtes (ordre logique)
- Comment filtrer précisément les données (WHERE)
- Comment gérer les valeurs NULL délicates
- Comment trier et paginer les résultats
- Comment éliminer les doublons

**Prêt à devenir un expert des requêtes SELECT ?** Passons à la section 5.1 pour comprendre l'ordre d'exécution logique d'une requête SQL !

---


⏭️ [L'ordre d'exécution logique d'une requête SQL](/05-requetes-de-selection/01-ordre-execution-logique.md)
