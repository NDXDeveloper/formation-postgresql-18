🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.1. L'ordre d'exécution logique d'une requête SQL

## Introduction

Lorsque vous écrivez une requête SQL, vous suivez une syntaxe spécifique : vous commencez par `SELECT`, puis `FROM`, ensuite `WHERE`, etc. Cependant, **PostgreSQL n'exécute pas votre requête dans l'ordre où vous l'écrivez**. Il existe un ordre logique d'exécution interne que le moteur de base de données suit, et comprendre cet ordre est crucial pour :

- Écrire des requêtes correctes et efficaces
- Comprendre pourquoi certaines requêtes génèrent des erreurs
- Optimiser les performances de vos requêtes
- Anticiper les résultats de vos requêtes complexes

Dans ce chapitre, nous allons démystifier cet ordre d'exécution logique.

---

## Ordre syntaxique vs Ordre logique

### L'ordre syntaxique (comment vous écrivez)

Voici l'ordre dans lequel vous **écrivez** une requête SQL :

```sql
SELECT      -- 1. Vous commencez par SELECT  
FROM        -- 2. Puis FROM  
WHERE       -- 3. Ensuite WHERE  
GROUP BY    -- 4. Suivi de GROUP BY  
HAVING      -- 5. Puis HAVING  
ORDER BY    -- 6. Et enfin ORDER BY  
LIMIT       -- 7. Terminé par LIMIT/OFFSET  
```

### L'ordre logique (comment PostgreSQL exécute)

Mais voici l'ordre dans lequel PostgreSQL **exécute réellement** votre requête :

```
1. FROM        -- PostgreSQL commence par identifier les tables
2. WHERE       -- Filtre les lignes AVANT toute agrégation
3. GROUP BY    -- Regroupe les lignes filtrées
4. HAVING      -- Filtre les groupes APRÈS agrégation
5. SELECT      -- Sélectionne et calcule les colonnes
6. DISTINCT    -- Élimine les doublons (si présent)
7. ORDER BY    -- Trie les résultats
8. LIMIT       -- Limite le nombre de lignes retournées
```

> **💡 Point clé** : Le `SELECT` n'est PAS la première étape ! PostgreSQL doit d'abord savoir quelles tables utiliser (`FROM`) et quelles lignes garder (`WHERE`) avant de pouvoir sélectionner des colonnes.

---

## Explication détaillée de chaque étape

### Étape 1 : FROM - Identifier la source des données

PostgreSQL commence par déterminer **d'où viennent les données**.

**Ce qui se passe :**
- Lecture de la ou des tables mentionnées après `FROM`
- Exécution des jointures (`JOIN`) si plusieurs tables sont impliquées
- Création d'un ensemble de données temporaire (appelé "relation de travail")

**Exemple :**
```sql
FROM employes
```

À cette étape, PostgreSQL charge toutes les lignes de la table `employes` en mémoire (conceptuellement).

**Avec une jointure :**
```sql
FROM employes  
JOIN departements ON employes.dept_id = departements.id  
```

PostgreSQL combine les deux tables selon la condition de jointure, créant un jeu de résultats temporaire contenant les colonnes des deux tables.

---

### Étape 2 : WHERE - Filtrage des lignes

Une fois les données sources identifiées, PostgreSQL applique les **conditions de filtrage**.

**Ce qui se passe :**
- Chaque ligne de l'ensemble temporaire est évaluée individuellement
- Seules les lignes satisfaisant la condition `WHERE` sont conservées
- Les lignes rejetées sont éliminées définitivement de la suite du traitement

**Exemple :**
```sql
SELECT nom, salaire  
FROM employes  
WHERE salaire > 50000;  
```

**Ordre d'exécution :**
1. PostgreSQL charge toutes les lignes de `employes` (FROM)  
2. PostgreSQL examine chaque ligne et ne garde que celles où `salaire > 50000` (WHERE)  
3. Seulement maintenant, PostgreSQL sélectionne les colonnes `nom` et `salaire` (SELECT)

**Important :** À ce stade, les fonctions d'agrégation comme `COUNT()`, `SUM()`, `AVG()` ne sont **pas encore** disponibles.

---

### Étape 3 : GROUP BY - Regroupement des lignes

Si votre requête contient un `GROUP BY`, PostgreSQL **regroupe les lignes** ayant les mêmes valeurs pour les colonnes spécifiées.

**Ce qui se passe :**
- Les lignes restantes (après filtrage WHERE) sont organisées en groupes
- Chaque groupe partage les mêmes valeurs pour les colonnes de regroupement
- Les agrégations (`COUNT`, `SUM`, `AVG`, etc.) sont calculées pour chaque groupe

**Exemple :**
```sql
SELECT departement, COUNT(*) as nb_employes  
FROM employes  
WHERE salaire > 50000  
GROUP BY departement;  
```

**Ordre d'exécution :**
1. Charger toutes les lignes de `employes` (FROM)  
2. Garder seulement les employés avec `salaire > 50000` (WHERE)  
3. Regrouper ces employés par `departement` (GROUP BY)  
4. Calculer `COUNT(*)` pour chaque groupe (SELECT)

**Résultat conceptuel après GROUP BY :**
```
Groupe 1 : departement = 'IT'     → 15 employés  
Groupe 2 : departement = 'RH'     → 8 employés  
Groupe 3 : departement = 'Ventes' → 12 employés  
```

---

### Étape 4 : HAVING - Filtrage des groupes

La clause `HAVING` permet de **filtrer les groupes** (pas les lignes individuelles).

**Ce qui se passe :**
- Après le regroupement, PostgreSQL évalue la condition `HAVING` sur chaque groupe
- Seuls les groupes satisfaisant la condition sont conservés
- C'est le moment d'utiliser les fonctions d'agrégation dans les conditions

**Exemple :**
```sql
SELECT departement, COUNT(*) as nb_employes  
FROM employes  
WHERE salaire > 50000  
GROUP BY departement  
HAVING COUNT(*) > 10;  
```

**Ordre d'exécution :**
1. Charger `employes` (FROM)  
2. Filtrer `salaire > 50000` (WHERE)  
3. Regrouper par `departement` (GROUP BY)  
4. **Éliminer les groupes ayant moins de 10 employés** (HAVING)  
5. Afficher le résultat (SELECT)

**Différence WHERE vs HAVING :**
- `WHERE` filtre les **lignes individuelles** AVANT le regroupement  
- `HAVING` filtre les **groupes** APRÈS le regroupement

**Exemple illustratif :**
```sql
-- WHERE : filtre les lignes avant regroupement
SELECT departement, AVG(salaire)  
FROM employes  
WHERE salaire > 30000  -- ← Filtre les employés individuels  
GROUP BY departement;  

-- HAVING : filtre les groupes après regroupement
SELECT departement, AVG(salaire)  
FROM employes  
GROUP BY departement  
HAVING AVG(salaire) > 50000;  -- ← Filtre les départements entiers  
```

---

### Étape 5 : SELECT - Sélection et calcul des colonnes

Ce n'est qu'**à cette étape** que PostgreSQL évalue réellement la clause `SELECT`.

**Ce qui se passe :**
- Les colonnes et expressions mentionnées après `SELECT` sont calculées
- Les alias de colonnes sont créés (avec `AS`)
- Les fonctions d'agrégation sont finalisées
- Les expressions calculées sont évaluées

**Exemple :**
```sql
SELECT
    departement,
    COUNT(*) as nb_employes,
    AVG(salaire) as salaire_moyen,
    AVG(salaire) * 1.1 as salaire_moyen_augmente
FROM employes  
GROUP BY departement;  
```

**Point important :** Les alias créés dans `SELECT` ne sont **pas disponibles** dans les clauses précédentes (FROM, WHERE, GROUP BY, HAVING), car elles sont exécutées AVANT le SELECT.

**❌ Ceci génère une erreur :**
```sql
SELECT salaire * 12 as salaire_annuel  
FROM employes  
WHERE salaire_annuel > 600000;  -- Erreur : salaire_annuel n'existe pas encore  
```

**✅ Solution correcte :**
```sql
SELECT salaire * 12 as salaire_annuel  
FROM employes  
WHERE salaire * 12 > 600000;  -- Répéter l'expression  
```

Ou avec une sous-requête :
```sql
SELECT *  
FROM (  
    SELECT salaire * 12 as salaire_annuel
    FROM employes
) sub
WHERE salaire_annuel > 600000;  -- Maintenant l'alias existe
```

---

### Étape 6 : DISTINCT - Élimination des doublons

Si `DISTINCT` est présent, PostgreSQL élimine les lignes dupliquées.

**Ce qui se passe :**
- Après le calcul des colonnes SELECT, PostgreSQL compare toutes les lignes
- Les lignes identiques (mêmes valeurs pour toutes les colonnes) sont éliminées
- Une seule ligne est conservée pour chaque ensemble de valeurs uniques

**Exemple :**
```sql
SELECT DISTINCT departement  
FROM employes;  
```

**Ordre d'exécution :**
1. Charger `employes` (FROM)  
2. Sélectionner la colonne `departement` (SELECT)  
3. Éliminer les doublons (DISTINCT)

Si vous avez 100 employés répartis dans 5 départements, le résultat final ne contiendra que 5 lignes (une par département unique).

---

### Étape 7 : ORDER BY - Tri des résultats

PostgreSQL trie maintenant les résultats selon les colonnes spécifiées.

**Ce qui se passe :**
- Les lignes du résultat final sont triées selon les critères définis
- Le tri peut être ascendant (`ASC`, par défaut) ou descendant (`DESC`)
- Plusieurs colonnes de tri peuvent être spécifiées

**Exemple :**
```sql
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC, nom ASC;  
```

**Particularité importante :** `ORDER BY` peut utiliser les alias définis dans `SELECT`, car il est exécuté APRÈS :

```sql
SELECT
    nom,
    prenom,
    salaire * 12 as salaire_annuel
FROM employes  
ORDER BY salaire_annuel DESC;  -- ✅ Fonctionne : l'alias existe déjà  
```

**Ordre de tri avec plusieurs colonnes :**
```sql
ORDER BY departement ASC, salaire DESC
```
Signifie : "Trier d'abord par département (A→Z), puis pour chaque département, trier par salaire (du plus élevé au plus bas)".

---

### Étape 8 : LIMIT / OFFSET - Limitation du nombre de lignes

En dernier lieu, PostgreSQL applique les restrictions `LIMIT` et `OFFSET`.

**Ce qui se passe :**
- `LIMIT` : Restreint le nombre de lignes retournées  
- `OFFSET` : Ignore un certain nombre de lignes avant de commencer à retourner des résultats
- Très utile pour la pagination

**Exemple :**
```sql
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC  
LIMIT 10;  -- Retourne les 10 employés les mieux payés  
```

**Avec OFFSET (pagination) :**
```sql
-- Page 1 (lignes 1-10)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC  
LIMIT 10 OFFSET 0;  

-- Page 2 (lignes 11-20)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC  
LIMIT 10 OFFSET 10;  

-- Page 3 (lignes 21-30)
SELECT nom, salaire  
FROM employes  
ORDER BY salaire DESC  
LIMIT 10 OFFSET 20;  
```

**⚠️ Important :** `LIMIT` et `OFFSET` sont appliqués **après le tri**. L'ordre des lignes est donc prévisible et stable (si vous utilisez `ORDER BY`).

---

## Visualisation complète avec un exemple

Prenons une requête complète et déroulons son exécution :

```sql
SELECT
    departement,
    COUNT(*) as nb_employes,
    AVG(salaire) as salaire_moyen
FROM employes  
WHERE annee_embauche >= 2020  
GROUP BY departement  
HAVING COUNT(*) > 5  
ORDER BY salaire_moyen DESC  
LIMIT 3;  
```

### Déroulement étape par étape :

**Étape 1 : FROM**
```
Action : Charger toutes les lignes de la table employes  
Résultat : 150 lignes disponibles  
```

**Étape 2 : WHERE**
```
Action : Garder seulement les employés embauchés depuis 2020  
Condition : annee_embauche >= 2020  
Résultat : 60 lignes restantes  
```

**Étape 3 : GROUP BY**
```
Action : Regrouper les 60 employés par departement  
Résultat : 8 groupes créés  
  - IT : 15 employés
  - RH : 4 employés
  - Ventes : 18 employés
  - Marketing : 12 employés
  - Finance : 8 employés
  - Support : 2 employés
  - R&D : 7 employés
  - Production : 14 employés
```

**Étape 4 : HAVING**
```
Action : Éliminer les groupes avec moins de 6 employés  
Condition : COUNT(*) > 5  
Résultat : 6 groupes restants  
  - IT : 15 employés
  - Ventes : 18 employés
  - Marketing : 12 employés
  - Finance : 8 employés
  - R&D : 7 employés
  - Production : 14 employés
(RH et Support éliminés)
```

**Étape 5 : SELECT**
```
Action : Calculer les colonnes demandées pour chaque groupe  
Résultat :  
  departement | nb_employes | salaire_moyen
  ------------|-------------|---------------
  IT          | 15          | 65000
  Ventes      | 18          | 58000
  Marketing   | 12          | 62000
  Finance     | 8           | 70000
  R&D         | 7           | 72000
  Production  | 14          | 54000
```

**Étape 6 : DISTINCT**
```
Action : Pas de DISTINCT dans cette requête, étape ignorée
```

**Étape 7 : ORDER BY**
```
Action : Trier par salaire_moyen décroissant  
Résultat :  
  departement | nb_employes | salaire_moyen
  ------------|-------------|---------------
  R&D         | 7           | 72000
  Finance     | 8           | 70000
  IT          | 15          | 65000
  Marketing   | 12          | 62000
  Ventes      | 18          | 58000
  Production  | 14          | 54000
```

**Étape 8 : LIMIT**
```
Action : Garder seulement les 3 premières lignes  
Résultat final :  
  departement | nb_employes | salaire_moyen
  ------------|-------------|---------------
  R&D         | 7           | 72000
  Finance     | 8           | 70000
  IT          | 15          | 65000
```

---

## Conséquences pratiques et pièges courants

### 1. Impossible d'utiliser un alias dans WHERE

**❌ Ne fonctionne pas :**
```sql
SELECT nom, salaire * 12 as salaire_annuel  
FROM employes  
WHERE salaire_annuel > 600000;  
```

**Raison :** `WHERE` est exécuté AVANT `SELECT`, donc l'alias `salaire_annuel` n'existe pas encore.

**✅ Solution :**
```sql
SELECT nom, salaire * 12 as salaire_annuel  
FROM employes  
WHERE salaire * 12 > 600000;  
```

### 2. Impossible d'utiliser un alias dans GROUP BY

**❌ Ne fonctionne pas :**
```sql
SELECT EXTRACT(YEAR FROM date_embauche) as annee, COUNT(*)  
FROM employes  
GROUP BY annee;  -- Erreur dans PostgreSQL < 9.6  
```

**✅ Solution (compatible toutes versions) :**
```sql
SELECT EXTRACT(YEAR FROM date_embauche) as annee, COUNT(*)  
FROM employes  
GROUP BY EXTRACT(YEAR FROM date_embauche);  
```

> **Note PostgreSQL ≥ 9.6 :** Les versions récentes de PostgreSQL acceptent les alias dans `GROUP BY` par commodité, mais conceptuellement, `GROUP BY` est toujours exécuté avant `SELECT`.

### 3. ORDER BY peut utiliser des alias

**✅ Fonctionne :**
```sql
SELECT nom, salaire * 12 as salaire_annuel  
FROM employes  
ORDER BY salaire_annuel DESC;  -- OK : ORDER BY est après SELECT  
```

### 4. WHERE vs HAVING : choisir le bon filtre

**Règle d'or :**
- Utilisez `WHERE` pour filtrer des **lignes individuelles**
- Utilisez `HAVING` pour filtrer des **groupes agrégés**

**Exemple comparatif :**
```sql
-- Trouver les départements où AU MOINS UN employé gagne > 100000
SELECT departement, COUNT(*)  
FROM employes  
WHERE salaire > 100000  -- Filtre les individus  
GROUP BY departement;  

-- Trouver les départements où le SALAIRE MOYEN > 100000
SELECT departement, AVG(salaire)  
FROM employes  
GROUP BY departement  
HAVING AVG(salaire) > 100000;  -- Filtre les groupes  
```

### 5. Performance : WHERE est plus rapide que HAVING

Si un filtre peut être appliqué avec `WHERE`, préférez-le à `HAVING` :

**❌ Moins performant :**
```sql
SELECT departement, COUNT(*)  
FROM employes  
GROUP BY departement  
HAVING departement = 'IT';  -- Filtre après regroupement  
```

**✅ Plus performant :**
```sql
SELECT departement, COUNT(*)  
FROM employes  
WHERE departement = 'IT'  -- Filtre avant regroupement (moins de données à traiter)  
GROUP BY departement;  
```

**Raison :** `WHERE` élimine les données inutiles dès le début, réduisant le volume de données à regrouper.

---

## Schéma récapitulatif

```
┌─────────────────────────────────────────────────────────────┐
│                   ORDRE D'EXÉCUTION SQL                     │
└─────────────────────────────────────────────────────────────┘

    Données brutes (tables)
            ↓
    [1. FROM + JOIN]  ← Charger les tables et les joindre
            ↓
       Ensemble de lignes complètes
            ↓
    [2. WHERE]        ← Filtrer les LIGNES individuelles
            ↓
       Lignes filtrées
            ↓
    [3. GROUP BY]     ← Regrouper les lignes
            ↓
       Groupes formés
            ↓
    [4. HAVING]       ← Filtrer les GROUPES
            ↓
       Groupes filtrés
            ↓
    [5. SELECT]       ← Calculer les colonnes et agrégations
            ↓
       Colonnes sélectionnées
            ↓
    [6. DISTINCT]     ← Éliminer les doublons
            ↓
       Lignes uniques
            ↓
    [7. ORDER BY]     ← Trier les résultats
            ↓
       Lignes triées
            ↓
    [8. LIMIT/OFFSET] ← Limiter le nombre de lignes
            ↓
    Résultat final renvoyé au client
```

---

## Points clés à retenir

1. **L'ordre syntaxique ≠ ordre d'exécution**
   - Vous écrivez `SELECT FROM WHERE`, mais PostgreSQL exécute `FROM WHERE SELECT`

2. **FROM est toujours la première étape**
   - PostgreSQL doit d'abord savoir quelles tables utiliser

3. **WHERE filtre les lignes, HAVING filtre les groupes**  
   - `WHERE` : avant agrégation → filtre individuel  
   - `HAVING` : après agrégation → filtre de groupes

4. **SELECT n'est PAS la première étape**
   - Les alias créés dans `SELECT` ne sont pas disponibles dans `WHERE`, `GROUP BY`, ou `HAVING`

5. **ORDER BY peut utiliser les alias**
   - Car il est exécuté après `SELECT`

6. **LIMIT est la dernière étape**
   - Toujours appliqué sur le résultat final trié

7. **Performance : filtrer tôt est essentiel**
   - Utilisez `WHERE` plutôt que `HAVING` quand c'est possible
   - Réduisez le volume de données dès les premières étapes

---

## Conclusion

Comprendre l'ordre d'exécution logique d'une requête SQL est fondamental pour :

- **Éviter les erreurs** : Savoir pourquoi un alias ne fonctionne pas dans une clause  
- **Écrire des requêtes correctes** : Placer les filtres au bon endroit  
- **Optimiser les performances** : Réduire les données à traiter dès le début  
- **Anticiper les résultats** : Comprendre comment PostgreSQL traite vos données

Dans les prochains chapitres, nous approfondirons chaque clause (WHERE, GROUP BY, HAVING, etc.) avec des exemples concrets et des cas d'usage avancés. Mais gardez toujours en tête cet ordre d'exécution : c'est la clé pour maîtriser SQL !

---


⏭️ [Filtrage (WHERE) et logique booléenne](/05-requetes-de-selection/02-filtrage-where.md)
