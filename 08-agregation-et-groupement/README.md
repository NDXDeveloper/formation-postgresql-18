🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8. Agrégation et Groupement

## Introduction : De la Donnée Brute à l'Information Exploitable

Jusqu'à présent, nous avons appris à interroger, filtrer, trier et joindre des données. Ces compétences nous permettent de récupérer des **lignes individuelles** de nos tables. Mais dans le monde réel, nous avons rarement besoin de voir chaque ligne de données. Ce que nous recherchons, ce sont des **résumés**, des **statistiques**, des **tendances**.

Imaginez une table contenant un million de transactions de vente. Personne ne veut consulter les un million de lignes ! Ce dont on a besoin, c'est de réponses à des questions comme :

- **Combien** de ventes avons-nous réalisées ce mois-ci ?
- **Quel est le montant total** des ventes ?
- **Quelle est la vente moyenne** par client ?
- **Quel produit** a généré le plus de revenus ?
- **Quelle région** performe le mieux ?

C'est exactement le rôle de l'**agrégation** : **transformer de nombreuses lignes en quelques résultats synthétiques et exploitables**.

---

## Qu'est-ce que l'Agrégation ?

### Définition

**L'agrégation** est le processus qui consiste à **combiner plusieurs valeurs en une seule valeur résumée**.

### Analogie Visuelle

Imaginez que vous avez une pile de factures papier sur votre bureau :

**Sans agrégation** : Vous regardez chaque facture une par une
- Facture 1 : 150€
- Facture 2 : 200€
- Facture 3 : 75€
- Facture 4 : 300€
- ...

**Avec agrégation** : Vous utilisez une calculatrice pour obtenir :
- **Total** : 725€
- **Nombre de factures** : 4
- **Montant moyen** : 181.25€
- **Facture la plus élevée** : 300€
- **Facture la plus basse** : 75€

L'agrégation SQL fait exactement cela, mais automatiquement et sur des millions de lignes !

---

## Pourquoi l'Agrégation est Essentielle ?

### 1. Prise de Décision Éclairée

Les décisions métier ne se prennent pas sur des lignes individuelles, mais sur des **tendances et des statistiques**.

**Exemples :**
- "Nos ventes ont augmenté de 15% ce trimestre" (agrégation temporelle)
- "Le panier moyen est de 87€" (moyenne)
- "Le produit le plus vendu est X" (comptage)

### 2. Rapports et Tableaux de Bord

Les rapports de gestion, tableaux de bord et KPI (Key Performance Indicators) sont construits sur des agrégations.

**Exemples de KPI :**
- Chiffre d'affaires mensuel : `SUM(montant_vente)`
- Taux de conversion : `COUNT(achats) / COUNT(visites)`
- Note moyenne produit : `AVG(note_client)`
- Nombre de clients actifs : `COUNT(DISTINCT client_id)`

### 3. Détection de Tendances et Anomalies

L'agrégation révèle des patterns invisibles dans les données brutes.

**Exemples :**
- Un pic de ventes inhabituel un jour donné
- Une baisse progressive des commandes dans une région
- Une augmentation du panier moyen après une campagne marketing

### 4. Performance et Scalabilité

Transférer 1 million de lignes à une application est lent et coûteux. Agréger en base de données et transférer 10 résultats est **instantané** !

**Comparaison :**
- ❌ **Mauvais** : Récupérer 1M lignes → calculer en Python/Java
- ✅ **Bon** : Calculer en SQL → récupérer le résultat (1 ligne)

---

## Les Deux Piliers : Agrégation et Groupement

L'agrégation seule permet de calculer des statistiques **globales** (sur toute la table). Mais la vraie puissance vient de la combinaison **agrégation + groupement**.

### Agrégation Globale (Sans GROUP BY)

Calcule une statistique sur **toutes les lignes**.

**Exemple :**
```sql
-- Combien de ventes au total ?
SELECT COUNT(*) AS total_ventes FROM ventes;
-- Résultat : 1 ligne (ex: 50000 ventes)
```

### Agrégation par Groupe (Avec GROUP BY)

Calcule des statistiques **par catégorie**.

**Exemple :**
```sql
-- Combien de ventes par produit ?
SELECT produit, COUNT(*) AS nb_ventes
FROM ventes
GROUP BY produit;
-- Résultat : 1 ligne par produit
```

**Métaphore :**
- **Sans GROUP BY** : Une seule calculatrice pour tout le monde
- **Avec GROUP BY** : Une calculatrice par groupe (produit, région, mois, etc.)

---

## Les Concepts Fondamentaux

### 1. Fonctions d'Agrégation

Ce sont les "opérations" que vous pouvez effectuer sur un ensemble de valeurs.

**Les 5 fonctions essentielles :**

| Fonction | Opération | Exemple |
|----------|-----------|---------|
| **COUNT** | Compter | Nombre de clients |
| **SUM** | Additionner | Chiffre d'affaires total |
| **AVG** | Moyenne | Panier moyen |
| **MIN** | Minimum | Prix le plus bas |
| **MAX** | Maximum | Prix le plus élevé |

**Fonctions statistiques avancées :**
- **STDDEV** : Écart-type (dispersion)
- **VARIANCE** : Variance
- **PERCENTILE** : Médiane, quartiles
- **CORR** : Corrélation entre variables

### 2. GROUP BY : Créer des Groupes

**GROUP BY** divise vos données en groupes selon les valeurs d'une ou plusieurs colonnes.

**Exemple conceptuel :**

Vous avez cette table `ventes` :

| id | produit  | montant | region |
|----|----------|---------|--------|
| 1  | Laptop   | 899     | Nord   |
| 2  | Souris   | 25      | Sud    |
| 3  | Laptop   | 899     | Est    |
| 4  | Clavier  | 75      | Nord   |
| 5  | Souris   | 25      | Sud    |

**Grouper par produit** crée 3 groupes :

```
Groupe "Laptop":   [899, 899]
Groupe "Souris":   [25, 25]
Groupe "Clavier":  [75]
```

Puis vous appliquez une agrégation sur chaque groupe :

```
COUNT(*) sur "Laptop":  2
COUNT(*) sur "Souris":  2
COUNT(*) sur "Clavier": 1
```

### 3. HAVING : Filtrer les Groupes

**WHERE** filtre les **lignes** avant l'agrégation.
**HAVING** filtre les **groupes** après l'agrégation.

**Exemple :**
```sql
-- Produits ayant généré plus de 1000€ de CA
SELECT produit, SUM(montant) AS ca
FROM ventes
GROUP BY produit
HAVING SUM(montant) > 1000;
```

### 4. Extensions de Groupement (ROLLUP, CUBE, GROUPING SETS)

Pour créer des **sous-totaux automatiques** à plusieurs niveaux.

**Exemple :** Ventes par région ET par produit + sous-totaux par région + total général

Ces extensions génèrent plusieurs niveaux d'agrégation en une seule requête !

### 5. Filtres d'Agrégation (FILTER)

Permet d'appliquer des **conditions différentes** à chaque agrégation.

**Exemple :**
```sql
-- Dans une seule requête : ventes validées ET ventes annulées
SELECT
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS nb_annulees
FROM ventes;
```

### 6. Agrégations Ordonnées (STRING_AGG, ARRAY_AGG)

Pour **concaténer** ou **regrouper** des valeurs dans un ordre spécifique.

**Exemple :**
```sql
-- Liste des produits achetés par un client
SELECT
    client_id,
    STRING_AGG(produit, ', ' ORDER BY date_achat) AS historique
FROM achats
GROUP BY client_id;
```

---

## L'Ordre d'Exécution SQL (Avec Agrégation)

Comprendre l'ordre d'exécution est **crucial** pour écrire des requêtes correctes.

### Ordre Logique

```sql
SELECT      -- 5. Sélection finale
    colonne,
    SUM(valeur) AS total
FROM table  -- 1. Chargement des données
WHERE       -- 2. Filtrage des LIGNES
    condition_ligne
GROUP BY    -- 3. Création des GROUPES
    colonne
HAVING      -- 4. Filtrage des GROUPES
    SUM(valeur) > 100
ORDER BY    -- 6. Tri
    total DESC
LIMIT 10;   -- 7. Limitation
```

**Séquence détaillée :**

1. **FROM** : PostgreSQL charge la table source
2. **WHERE** : Filtre les lignes individuelles (AVANT agrégation)
3. **GROUP BY** : Crée les groupes selon les colonnes spécifiées
4. **HAVING** : Filtre les groupes (APRÈS agrégation)
5. **SELECT** : Calcule les agrégations et sélectionne les colonnes
6. **ORDER BY** : Trie les résultats
7. **LIMIT** : Limite le nombre de lignes retournées

**Point clé :**
- WHERE s'exécute **AVANT** le groupement → filtre des lignes
- HAVING s'exécute **APRÈS** le groupement → filtre des groupes

---

## Visualisation : Du Détail à l'Agrégat

### Exemple Complet

**Table source `ventes` :**

| id_vente | produit     | montant | region | date_vente |
|----------|-------------|---------|--------|------------|
| 1        | Laptop      | 899     | Nord   | 2024-01-15 |
| 2        | Souris      | 25      | Sud    | 2024-01-16 |
| 3        | Laptop      | 899     | Est    | 2024-01-17 |
| 4        | Clavier     | 75      | Nord   | 2024-01-18 |
| 5        | Souris      | 25      | Sud    | 2024-01-19 |
| 6        | Laptop      | 899     | Nord   | 2024-01-20 |
| 7        | Écran       | 250     | Est    | 2024-01-21 |
| 8        | Souris      | 25      | Nord   | 2024-01-22 |

**Question : Quel est le chiffre d'affaires par produit ?**

```sql
SELECT
    produit,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total,
    AVG(montant) AS prix_moyen
FROM ventes
GROUP BY produit
ORDER BY ca_total DESC;
```

**Processus :**

1. **FROM ventes** : Charge les 8 lignes
2. **GROUP BY produit** : Crée 4 groupes
   ```
   Groupe "Laptop":  [899, 899, 899]
   Groupe "Souris":  [25, 25, 25]
   Groupe "Clavier": [75]
   Groupe "Écran":   [250]
   ```
3. **SELECT** : Calcule les agrégations pour chaque groupe
4. **ORDER BY** : Trie par CA décroissant

**Résultat :**

| produit  | nb_ventes | ca_total | prix_moyen |
|----------|-----------|----------|------------|
| Laptop   | 3         | 2697     | 899.00     |
| Écran    | 1         | 250      | 250.00     |
| Souris   | 3         | 75       | 25.00      |
| Clavier  | 1         | 75       | 75.00      |

**Transformation :** 8 lignes → 4 lignes (une par produit)

---

## Types de Questions Résolues par l'Agrégation

### 1. Questions de Volume

**"Combien ?"**

- Combien de clients avons-nous ? → `COUNT(DISTINCT client_id)`
- Combien de commandes ce mois ? → `COUNT(*) WHERE date >= ...`
- Combien de produits en stock ? → `SUM(quantite_stock)`

### 2. Questions Financières

**"Quel montant ?"**

- Quel est notre chiffre d'affaires ? → `SUM(montant)`
- Quel est le panier moyen ? → `AVG(montant_commande)`
- Quelle est la marge totale ? → `SUM(prix_vente - prix_achat)`

### 3. Questions de Performance

**"Quelle est la meilleure/pire performance ?"**

- Quel produit se vend le plus ? → `GROUP BY produit ORDER BY COUNT(*) DESC`
- Quelle région génère le plus de CA ? → `GROUP BY region ORDER BY SUM(montant) DESC`
- Quel vendeur a la meilleure conversion ? → `AVG(taux_conversion) GROUP BY vendeur`

### 4. Questions de Tendance

**"Comment évolue... ?"**

- Les ventes augmentent-elles ? → `GROUP BY EXTRACT(MONTH FROM date)`
- Le panier moyen baisse-t-il ? → `AVG(montant) GROUP BY semaine`
- Quelle est la saisonnalité ? → Groupement par mois, analyse annuelle

### 5. Questions de Distribution

**"Comment se répartissent... ?"**

- Distribution des âges : `COUNT(*) GROUP BY tranche_age`
- Répartition géographique : `COUNT(*) GROUP BY region`
- Segmentation clients : `GROUP BY segment ORDER BY COUNT(*) DESC`

### 6. Questions de Qualité

**"Quelle est la variabilité ?"**

- Les délais de livraison sont-ils constants ? → `STDDEV(delai)`
- Les prix sont-ils homogènes ? → `VARIANCE(prix)`
- Quelle est la note médiane ? → `PERCENTILE_CONT(0.5)`

---

## Pièges Courants des Débutants

### 1. Mélanger Colonnes Agrégées et Non-Agrégées

```sql
-- ❌ ERREUR : montant n'est ni agrégé ni dans GROUP BY
SELECT produit, montant, COUNT(*)
FROM ventes
GROUP BY produit;

-- ✅ CORRECT : Soit agréger montant...
SELECT produit, SUM(montant), COUNT(*)
FROM ventes
GROUP BY produit;

-- ✅ ...soit l'inclure dans GROUP BY (si pertinent)
SELECT produit, montant, COUNT(*)
FROM ventes
GROUP BY produit, montant;
```

### 2. Utiliser WHERE au lieu de HAVING

```sql
-- ❌ ERREUR : WHERE ne peut pas filtrer sur une agrégation
SELECT produit, SUM(montant) AS total
FROM ventes
WHERE SUM(montant) > 1000  -- ❌
GROUP BY produit;

-- ✅ CORRECT : HAVING pour filtrer les groupes
SELECT produit, SUM(montant) AS total
FROM ventes
GROUP BY produit
HAVING SUM(montant) > 1000;
```

### 3. Oublier GROUP BY avec les Agrégations

```sql
-- ❌ Que voulez-vous vraiment ?
SELECT produit, COUNT(*)
FROM ventes;
-- Erreur : produit doit être dans GROUP BY

-- ✅ Option 1 : Grouper
SELECT produit, COUNT(*)
FROM ventes
GROUP BY produit;

-- ✅ Option 2 : Tout agréger
SELECT COUNT(DISTINCT produit) AS nb_produits_differents
FROM ventes;
```

### 4. Confondre COUNT(*) et COUNT(colonne)

```sql
-- COUNT(*) : compte TOUTES les lignes
SELECT COUNT(*) FROM ventes;  -- 1000

-- COUNT(colonne) : compte les valeurs NON-NULL
SELECT COUNT(email) FROM clients;  -- 850 (si 150 NULL)
```

### 5. Ignorer les NULL dans les Agrégations

Les fonctions d'agrégation (sauf COUNT(*)) **ignorent les NULL** :

```sql
-- Si montant contient des NULL :
SELECT AVG(montant) FROM ventes;
-- Calcule la moyenne uniquement sur les valeurs non-NULL !

-- Pour inclure les NULL comme zéros :
SELECT AVG(COALESCE(montant, 0)) FROM ventes;
```

---

## Performance : Pourquoi l'Agrégation en Base de Données ?

### Avantages de l'Agrégation Côté Serveur

**1. Réduction du Volume de Données Transférées**

```
Sans agrégation :
Base → Réseau → Application
1M lignes × 100 octets = 100 MB

Avec agrégation :
Base → Réseau → Application
10 lignes × 100 octets = 1 KB

Gain : 100 000× moins de données !
```

**2. Utilisation des Index**

PostgreSQL peut utiliser des index pour accélérer les agrégations :
- Index sur colonnes GROUP BY
- Index couvrants pour MIN/MAX
- Index partiels pour WHERE + agrégation

**3. Calculs Optimisés**

Les bases de données sont **optimisées** pour les agrégations :
- Algorithmes de hachage rapides
- Tri efficace
- Parallélisation automatique

**4. Moins de Code Applicatif**

```python
# ❌ Mauvais : Calculer en Python
lignes = db.query("SELECT * FROM ventes")
total = sum(ligne['montant'] for ligne in lignes)

# ✅ Bon : Calculer en SQL
total = db.query("SELECT SUM(montant) FROM ventes")[0]['sum']
```

---

## Vue d'Ensemble des Sections à Venir

Cette section 8 est structurée de manière progressive, du plus simple au plus avancé.

### 8.1. Fonctions d'Agrégation Standards

**Objectif :** Maîtriser les 5 fonctions essentielles
- COUNT : Compter
- SUM : Additionner
- AVG : Moyenne
- MIN/MAX : Extrema

**Niveau :** Débutant
**Durée estimée :** 1-2 heures

### 8.2. Fonctions d'Agrégation Statistiques

**Objectif :** Analyses statistiques avancées
- STDDEV : Écart-type
- VARIANCE : Variance
- CORR : Corrélation
- PERCENTILE : Médiane, quartiles

**Niveau :** Intermédiaire
**Durée estimée :** 2-3 heures

### 8.3. GROUP BY et HAVING

**Objectif :** Agréger par catégorie et filtrer les groupes
- Créer des groupes multi-colonnes
- Filtrage avec HAVING
- Différence WHERE vs HAVING

**Niveau :** Intermédiaire
**Durée estimée :** 2-3 heures

### 8.4. Extensions de Groupement

**Objectif :** Sous-totaux automatiques multi-niveaux
- ROLLUP : Sous-totaux hiérarchiques
- CUBE : Toutes les combinaisons
- GROUPING SETS : Contrôle total

**Niveau :** Avancé
**Durée estimée :** 3-4 heures

### 8.5. Filtres d'Agrégation (FILTER)

**Objectif :** Agrégations conditionnelles modernes
- Clause FILTER
- Remplacement de CASE WHEN
- Agrégations multi-critères

**Niveau :** Intermédiaire
**Durée estimée :** 1-2 heures

### 8.6. Agrégations Ordonnées

**Objectif :** Agréger avec ordre
- STRING_AGG : Concaténation
- ARRAY_AGG : Tableaux
- JSON_AGG : Structures JSON
- WITHIN GROUP

**Niveau :** Intermédiaire
**Durée estimée :** 2-3 heures

**Durée totale estimée :** 12-18 heures

---

## Prérequis

Avant d'aborder cette section, vous devriez être à l'aise avec :

✅ **SQL de base**
- SELECT, FROM, WHERE
- Filtrage et conditions (AND, OR, IN, BETWEEN)
- Tri (ORDER BY)
- Limitation (LIMIT)

✅ **Types de données**
- Numériques (INTEGER, NUMERIC, FLOAT)
- Texte (VARCHAR, TEXT)
- Dates (DATE, TIMESTAMP)

✅ **Concepts de base**
- NULL et sa gestion (IS NULL, COALESCE)
- Opérateurs de comparaison
- Expressions de base

**Optionnel mais utile :**
- Jointures (pour agréger sur plusieurs tables)
- Sous-requêtes (pour agrégations avancées)

---

## Cas d'Usage dans le Monde Réel

### E-commerce
- Tableau de bord : CA du jour, panier moyen, taux de conversion
- Analyse produits : Top ventes, stock moyen, rotation
- Segmentation clients : RFM (Recency, Frequency, Monetary)

### Finance
- Bilans : Sommes par compte, soldes agrégés
- Risque : Volatilité (STDDEV), corrélations
- Reporting : Agrégations mensuelles, trimestrielles, annuelles

### Santé
- Statistiques : Nombre de patients, durée moyenne de séjour
- Qualité : Taux de satisfaction, taux de réadmission
- Épidémiologie : Tendances, distributions par région

### SaaS / Web
- Analytics : Utilisateurs actifs, sessions moyennes
- Performance : P95/P99 latence, taux d'erreur
- Conversion : Funnel d'acquisition par étape

### RH
- Effectifs : Nombre d'employés par département
- Masse salariale : Salaire moyen, médian, écart-type
- Turnover : Taux de départ, ancienneté moyenne

---

## Conseils pour Bien Démarrer

### 1. Commencez Simple

Ne cherchez pas à tout maîtriser d'un coup. Progression recommandée :
1. COUNT et SUM (les plus simples)
2. AVG, MIN, MAX
3. GROUP BY simple (une colonne)
4. GROUP BY multi-colonnes
5. HAVING
6. Fonctions avancées (STDDEV, PERCENTILE, etc.)

### 2. Visualisez Mentalement

Avant d'écrire une requête, visualisez :
- Quelles lignes seront groupées ensemble ?
- Quel calcul sera fait sur chaque groupe ?
- À quoi ressemblera le résultat final ?

### 3. Testez sur de Petits Échantillons

Utilisez LIMIT pour tester sur peu de données :
```sql
-- Tester la logique sur 100 lignes
SELECT produit, COUNT(*)
FROM ventes
WHERE date_vente >= '2024-01-01'
GROUP BY produit
LIMIT 100;
```

### 4. Vérifiez Vos Résultats

Validez vos agrégations :
```sql
-- Total doit correspondre à la somme des groupes
SELECT SUM(montant) FROM ventes;  -- 10000

SELECT produit, SUM(montant)
FROM ventes
GROUP BY produit;
-- La somme de tous les SUM(montant) doit faire 10000 !
```

### 5. Utilisez des Alias Explicites

```sql
-- ✅ Clair
SELECT
    produit,
    COUNT(*) AS nombre_ventes,
    SUM(montant) AS chiffre_affaires_total,
    AVG(montant) AS panier_moyen
FROM ventes
GROUP BY produit;

-- ❌ Obscur
SELECT
    produit,
    COUNT(*),
    SUM(montant),
    AVG(montant)
FROM ventes
GROUP BY produit;
```

---

## À Retenir

1. **L'agrégation transforme beaucoup de lignes en quelques résultats synthétiques**
2. **GROUP BY crée des groupes**, les agrégations calculent sur chaque groupe
3. **WHERE filtre les lignes**, **HAVING filtre les groupes**
4. **Ordre d'exécution** : FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
5. Les agrégations **ignorent les NULL** (sauf COUNT(*))
6. **COUNT(*)** compte les lignes, **COUNT(colonne)** compte les non-NULL
7. Toujours **nommer vos colonnes agrégées** avec AS
8. L'agrégation en SQL est **bien plus performante** qu'en application
9. Les fonctions d'agrégation sont le **fondement de l'analyse de données**
10. Commencez simple et progressez graduellement !

---

## Prochaine Étape

Maintenant que vous comprenez les concepts fondamentaux de l'agrégation et du groupement, passons à la pratique avec la **Section 8.1 : Fonctions d'Agrégation Standards (COUNT, SUM, AVG, MIN, MAX)**.

Vous y apprendrez à utiliser les 5 fonctions essentielles qui constituent 90% des agrégations quotidiennes en SQL.

**Bonne découverte de l'agrégation PostgreSQL !** 🚀

⏭️ [Fonctions d'agrégation standards (COUNT, SUM, AVG, MIN, MAX)](/08-agregation-et-groupement/01-fonctions-agregation-standards.md)
