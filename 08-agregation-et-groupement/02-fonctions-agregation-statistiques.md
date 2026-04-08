🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.2. Fonctions d'Agrégation Statistiques (STDDEV, VARIANCE, CORR, PERCENTILE)

## Introduction aux Statistiques en SQL

Dans la section précédente, nous avons exploré les fonctions d'agrégation de base (COUNT, SUM, AVG, MIN, MAX). Ces fonctions sont essentielles, mais elles ne racontent qu'une partie de l'histoire des données.

PostgreSQL propose des **fonctions d'agrégation statistiques avancées** qui permettent d'analyser plus finement vos données : mesurer la dispersion, identifier des tendances, détecter des corrélations, et comprendre la distribution des valeurs.

### Pourquoi les Statistiques en Base de Données ?

Traditionnellement, les analyses statistiques se font dans des outils comme Excel, Python ou R. Mais effectuer ces calculs directement en SQL présente des avantages majeurs :

- ✅ **Performance** : Calculs effectués au plus près des données, sans transfert réseau  
- ✅ **Simplicité** : Pas besoin d'exporter puis réimporter les résultats  
- ✅ **Cohérence** : Garantie transactionnelle sur les données analysées  
- ✅ **Scalabilité** : PostgreSQL optimise les calculs sur des millions de lignes

---

## Vue d'Ensemble des Fonctions Statistiques

PostgreSQL offre un ensemble complet de fonctions statistiques :

| Catégorie | Fonctions | Usage |
|-----------|-----------|-------|
| **Dispersion** | STDDEV, VARIANCE | Mesurer l'écart par rapport à la moyenne |
| **Corrélation** | CORR, COVAR_POP, COVAR_SAMP | Relations entre deux variables |
| **Régression** | REGR_SLOPE, REGR_INTERCEPT | Analyse de tendance linéaire |
| **Distributions** | PERCENTILE_CONT, PERCENTILE_DISC | Médiane, quartiles, centiles |

Nous allons explorer les plus importantes dans cette section.

---

## 1. VARIANCE - La Variance

### Qu'est-ce que la Variance ?

La variance mesure **à quel point les valeurs sont dispersées autour de la moyenne**. C'est un indicateur fondamental de variabilité des données.

- **Variance faible** : Les valeurs sont proches de la moyenne (données homogènes)  
- **Variance élevée** : Les valeurs sont très dispersées (données hétérogènes)

### Formule Mathématique

Pour une population :

```
Variance = Σ(xi - moyenne)² / N
```

Où :
- `xi` = chaque valeur  
- `moyenne` = moyenne des valeurs  
- `N` = nombre total de valeurs

### Syntaxe PostgreSQL

PostgreSQL propose deux variantes :

```sql
-- Variance d'échantillon (Sample Variance) - LA PLUS COURANTE
VAR_SAMP(colonne)  
VARIANCE(colonne)  -- Alias de VAR_SAMP  

-- Variance de population (Population Variance)
VAR_POP(colonne)
```

**Quelle version utiliser ?**

- **VAR_SAMP / VARIANCE** : Utiliser **dans 99% des cas** (données = échantillon d'une population plus large)  
- **VAR_POP** : Seulement si vos données représentent la **totalité** d'une population (rare en pratique)

### Exemple Théorique

Imaginons une table `salaires_employes` :

| employe_id | nom      | salaire |
|------------|----------|---------|
| 1          | Alice    | 35000   |
| 2          | Bob      | 38000   |
| 3          | Charlie  | 36000   |
| 4          | Diana    | 37000   |
| 5          | Eve      | 34000   |

**Calcul de la variance :**

```sql
SELECT
    AVG(salaire) AS salaire_moyen,
    VARIANCE(salaire) AS variance_salaire
FROM salaires_employes;
```

**Résultat théorique :**

| salaire_moyen | variance_salaire |
|---------------|------------------|
| 36000         | 2500000          |

**Interprétation :**
- La moyenne est de 36 000 €
- La variance est de 2 500 000 (en €²)
- Cette valeur seule est difficile à interpréter → c'est pourquoi on utilise l'écart-type !

### Cas d'Usage

- **Finance** : Volatilité des prix, risque de portefeuille  
- **Qualité** : Cohérence de la production industrielle  
- **Performance** : Variabilité des temps de réponse d'une API  
- **Marketing** : Dispersion des comportements d'achat

---

## 2. STDDEV - L'Écart-Type (Standard Deviation)

### Qu'est-ce que l'Écart-Type ?

L'écart-type est la **racine carrée de la variance**. C'est la mesure de dispersion la plus utilisée car elle est exprimée **dans la même unité que les données originales**.

### Pourquoi l'Écart-Type est Plus Intuitif que la Variance ?

Reprenons l'exemple des salaires :
- Variance : 2 500 000 **€²** (unité difficile à interpréter)
- Écart-type : √2 500 000 ≈ 1 581 **€** (unité claire : euros)

**Interprétation** : "En moyenne, les salaires s'écartent de 1 581 € par rapport à la moyenne de 36 000 €"

### Syntaxe PostgreSQL

```sql
-- Écart-type d'échantillon (Sample Standard Deviation) - LA PLUS COURANTE
STDDEV_SAMP(colonne)  
STDDEV(colonne)  -- Alias de STDDEV_SAMP  

-- Écart-type de population (Population Standard Deviation)
STDDEV_POP(colonne)
```

### Exemple Théorique

```sql
SELECT
    AVG(salaire) AS salaire_moyen,
    STDDEV(salaire) AS ecart_type_salaire,
    VARIANCE(salaire) AS variance_salaire
FROM salaires_employes;
```

**Résultat :**

| salaire_moyen | ecart_type_salaire | variance_salaire |
|---------------|-------------------|------------------|
| 36000         | 1581.14           | 2500000          |

### Interprétation de l'Écart-Type : La Règle des 68-95-99.7

Pour une distribution normale (en cloche) :

- **68%** des valeurs se situent à ±1 écart-type de la moyenne  
- **95%** des valeurs se situent à ±2 écarts-types de la moyenne  
- **99.7%** des valeurs se situent à ±3 écarts-types de la moyenne

**Exemple avec nos salaires** (moyenne = 36 000, écart-type = 1 581) :

- **68%** des salaires devraient être entre 34 419 € et 37 581 €  
- **95%** des salaires devraient être entre 32 838 € et 39 162 €

### Comparaison de Dispersion

Imaginons deux départements :

**Département A :**
```sql
-- Salaires : 35k, 36k, 37k (très homogènes)
-- Moyenne : 36k, Écart-type : ~817€
```

**Département B :**
```sql
-- Salaires : 25k, 36k, 47k (très dispersés)
-- Moyenne : 36k, Écart-type : ~9000€
```

**Observation** : Même moyenne, mais dispersion radicalement différente !

### Détection d'Anomalies avec l'Écart-Type

Un usage classique : identifier les valeurs aberrantes (outliers)

```sql
-- Trouver les salaires "anormaux" (> 3 écarts-types de la moyenne)
WITH stats AS (
    SELECT
        AVG(salaire) AS moyenne,
        STDDEV(salaire) AS ecart_type
    FROM salaires_employes
)
SELECT e.*  
FROM salaires_employes e, stats s  
WHERE e.salaire > s.moyenne + (3 * s.ecart_type)  
   OR e.salaire < s.moyenne - (3 * s.ecart_type);
```

Cette requête identifie les salaires qui s'écartent de plus de 3 σ (sigma) de la moyenne, ce qui est statistiquement très rare (<0.3% si distribution normale).

---

## 3. Corrélation et Covariance

### 3.1. Covariance (COVAR)

#### Qu'est-ce que la Covariance ?

La covariance mesure **dans quelle mesure deux variables varient ensemble** :

- **Covariance positive** : Quand X augmente, Y augmente aussi  
- **Covariance négative** : Quand X augmente, Y diminue  
- **Covariance proche de zéro** : Aucune relation linéaire évidente

#### Syntaxe PostgreSQL

```sql
-- Covariance d'échantillon
COVAR_SAMP(Y, X)

-- Covariance de population
COVAR_POP(Y, X)
```

⚠️ **Ordre des arguments** : `COVAR(Y, X)` → Y est la variable dépendante, X la variable indépendante

#### Exemple Théorique

Table `ventes_marketing` :

| mois | budget_pub | ventes |
|------|------------|--------|
| 1    | 1000       | 15000  |
| 2    | 1500       | 18000  |
| 3    | 2000       | 21000  |
| 4    | 1200       | 16000  |
| 5    | 1800       | 20000  |

```sql
SELECT COVAR_SAMP(ventes, budget_pub) AS covariance  
FROM ventes_marketing;  
-- Résultat positif → relation positive entre budget pub et ventes
```

#### Limite de la Covariance

**Problème** : La valeur de la covariance dépend de l'échelle des variables. Une covariance de 1000 est-elle forte ou faible ? Impossible à dire sans contexte !

**Solution** : Utiliser le coefficient de corrélation (standardisé).

---

### 3.2. Corrélation (CORR)

#### Qu'est-ce que le Coefficient de Corrélation ?

Le coefficient de corrélation (de Pearson) est une **version standardisée de la covariance**. Il est toujours compris entre **-1 et +1** :

- **+1** : Corrélation positive parfaite (relation linéaire croissante parfaite)  
- **0** : Aucune corrélation linéaire  
- **-1** : Corrélation négative parfaite (relation linéaire décroissante parfaite)

#### Formule

```
CORR = COVAR(X,Y) / (STDDEV(X) * STDDEV(Y))
```

#### Syntaxe PostgreSQL

```sql
CORR(Y, X)
```

#### Interprétation des Valeurs

| Valeur de CORR | Interprétation |
|----------------|----------------|
| 0.9 à 1.0      | Corrélation très forte positive |
| 0.7 à 0.9      | Corrélation forte positive |
| 0.4 à 0.7      | Corrélation modérée positive |
| 0.0 à 0.4      | Corrélation faible ou nulle |
| -0.4 à 0.0     | Corrélation faible négative |
| -0.7 à -0.4    | Corrélation modérée négative |
| -0.9 à -0.7    | Corrélation forte négative |
| -1.0 à -0.9    | Corrélation très forte négative |

#### Exemple Théorique

```sql
SELECT
    CORR(ventes, budget_pub) AS correlation_ventes_pub,
    COVAR_SAMP(ventes, budget_pub) AS covariance
FROM ventes_marketing;
```

**Résultat hypothétique :**

| correlation_ventes_pub | covariance |
|------------------------|------------|
| 0.95                   | 2500000    |

**Interprétation** :
- Corrélation de 0.95 → **relation très forte** entre budget pub et ventes
- Quand le budget augmente, les ventes augmentent de façon quasi-linéaire

#### Cas d'Usage Réels

**E-commerce :**
```sql
-- Y a-t-il une corrélation entre le nombre de produits consultés
-- et le montant du panier final ?
SELECT CORR(montant_panier, nb_produits_vus) AS correlation  
FROM sessions_utilisateurs;  
```

**Performance Web :**
```sql
-- Le temps de chargement impacte-t-il le taux de rebond ?
SELECT CORR(taux_rebond, temps_chargement_ms) AS correlation  
FROM analytics_pages;  
-- Résultat attendu : corrélation positive (+ de temps = + de rebond)
```

**RH :**
```sql
-- Ancienneté vs salaire : quelle corrélation ?
SELECT CORR(salaire, annees_anciennete) AS correlation  
FROM employes;  
```

#### ⚠️ Attention : Corrélation ≠ Causalité !

**TRÈS IMPORTANT** : Une forte corrélation ne signifie PAS qu'il y a un lien de cause à effet !

Exemple classique :
- Corrélation entre ventes de glace et noyades en été : **forte**
- Cause réelle : La température (variable cachée)

Toujours analyser le contexte métier avant de conclure !

---

## 4. Percentiles et Distribution

### Qu'est-ce qu'un Percentile ?

Un **percentile** est une valeur qui sépare les données en pourcentages :

- **Médiane** = 50ᵉ percentile (50% des valeurs en dessous, 50% au-dessus)  
- **Premier quartile (Q1)** = 25ᵉ percentile  
- **Troisième quartile (Q3)** = 75ᵉ percentile  
- **95ᵉ percentile** = 95% des valeurs sont inférieures

### Pourquoi les Percentiles ?

La **moyenne est sensible aux valeurs extrêmes**. Les percentiles sont plus robustes :

**Exemple :**
- 9 personnes gagnent 30 000 €
- 1 personne (PDG) gagne 1 000 000 €

Statistiques :
- **Moyenne** : 127 000 € (ne représente PERSONNE !)  
- **Médiane (50ᵉ percentile)** : 30 000 € (représentative)

### Syntaxe PostgreSQL

PostgreSQL propose deux fonctions de percentile :

```sql
-- PERCENTILE_CONT : Interpolation continue (peut retourner une valeur qui n'existe pas)
PERCENTILE_CONT(percentile) WITHIN GROUP (ORDER BY colonne)

-- PERCENTILE_DISC : Valeur discrète (retourne une valeur réelle du dataset)
PERCENTILE_DISC(percentile) WITHIN GROUP (ORDER BY colonne)
```

**Paramètre `percentile`** : Valeur entre 0 et 1
- 0.5 = médiane
- 0.25 = premier quartile
- 0.75 = troisième quartile
- 0.95 = 95ᵉ percentile

### Différence entre CONT et DISC

**Table exemple `notes_examen` :**

| etudiant_id | note |
|-------------|------|
| 1           | 10   |
| 2           | 12   |
| 3           | 14   |
| 4           | 16   |
| 5           | 18   |

```sql
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY note) AS mediane_cont,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY note) AS mediane_disc
FROM notes_examen;
```

**Résultat :**

| mediane_cont | mediane_disc |
|--------------|--------------|
| 14.0         | 14           |

**Cas avec nombre pair de valeurs :**

Si on ajoute une 6ᵉ note de 20 :

| mediane_cont | mediane_disc |
|--------------|--------------|
| 15.0         | 14 ou 16     |

- **CONT** : Interpole entre 14 et 16 → 15 (valeur qui n'existe pas dans le dataset)  
- **DISC** : Choisit une valeur réelle (généralement la plus basse des deux)

**Quand utiliser quoi ?**

- **PERCENTILE_CONT** : Pour des données continues (salaires, températures, temps)  
- **PERCENTILE_DISC** : Pour des données discrètes ou quand vous voulez une valeur réelle

### Exemple : Analyse Salariale Complète

```sql
SELECT
    MIN(salaire) AS salaire_min,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salaire) AS q1_25,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salaire) AS mediane,
    AVG(salaire) AS moyenne,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salaire) AS q3_75,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY salaire) AS p95,
    MAX(salaire) AS salaire_max,
    STDDEV(salaire) AS ecart_type
FROM salaires_employes;
```

**Résultat théorique :**

| salaire_min | q1_25  | mediane | moyenne | q3_75  | p95    | salaire_max | ecart_type |
|-------------|--------|---------|---------|--------|--------|-------------|------------|
| 25000       | 35000  | 42000   | 48000   | 55000  | 85000  | 150000      | 18000      |

**Interprétation :**
- La **médiane** (42k) est bien inférieure à la **moyenne** (48k) → distribution asymétrique avec quelques très hauts salaires
- 25% des employés gagnent moins de 35k
- 95% des employés gagnent moins de 85k
- L'écart-type de 18k montre une forte dispersion

### Percentiles Multiples : Analyse de Performance Web

```sql
SELECT
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY temps_reponse_ms) AS p50_mediane,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY temps_reponse_ms) AS p90,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY temps_reponse_ms) AS p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY temps_reponse_ms) AS p99
FROM logs_requetes_api;
```

**Résultat :**

| p50_mediane | p90  | p95  | p99  |
|-------------|------|------|------|
| 120         | 350  | 520  | 1850 |

**Interprétation (SLA Web typique) :**
- **50% des requêtes** répondent en moins de 120ms (expérience majoritaire)  
- **90% des requêtes** répondent en moins de 350ms (bon)  
- **5% des requêtes** prennent plus de 520ms (acceptable)  
- **1% des requêtes** prennent plus de 1.8s (cas extrêmes à investiguer)

**Pourquoi P95/P99 plutôt que la moyenne ?**

Une moyenne de 200ms peut cacher :
- Cas A : Toutes les requêtes entre 150-250ms (excellent)
- Cas B : 90% à 100ms, 10% à 1000ms+ (problématique pour les utilisateurs)

Les percentiles révèlent cette disparité !

---

## 5. Autres Fonctions Statistiques Utiles

### 5.1. Analyse de Régression Linéaire

PostgreSQL peut calculer automatiquement les paramètres d'une droite de régression (y = ax + b) :

```sql
-- Pente de la droite (coefficient a)
REGR_SLOPE(Y, X)

-- Ordonnée à l'origine (coefficient b)
REGR_INTERCEPT(Y, X)

-- Coefficient de détermination R² (qualité de l'ajustement)
REGR_R2(Y, X)
```

**Exemple : Prédire les ventes en fonction du budget pub**

```sql
SELECT
    REGR_SLOPE(ventes, budget_pub) AS pente,
    REGR_INTERCEPT(ventes, budget_pub) AS ordonnee,
    REGR_R2(ventes, budget_pub) AS r_carre,
    CORR(ventes, budget_pub) AS correlation
FROM ventes_marketing;
```

**Résultat théorique :**

| pente | ordonnee | r_carre | correlation |
|-------|----------|---------|-------------|
| 8.5   | 6500     | 0.90    | 0.95        |

**Interprétation :**
- **Équation de régression** : `Ventes = 8.5 × Budget_pub + 6500`
- Pour chaque euro investi en pub, on génère 8.5€ de ventes supplémentaires
- R² de 0.90 signifie que 90% de la variance des ventes est expliquée par le budget pub
- **Prédiction** : Si budget = 2000€ → Ventes prédites = 8.5×2000 + 6500 = 23 500€

### 5.2. Fonctions d'Agrégation Statistiques par Groupe

Toutes ces fonctions peuvent être combinées avec GROUP BY :

```sql
SELECT
    departement,
    COUNT(*) AS nb_employes,
    AVG(salaire) AS salaire_moyen,
    STDDEV(salaire) AS ecart_type,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salaire) AS mediane
FROM employes  
GROUP BY departement  
ORDER BY salaire_moyen DESC;  
```

**Résultat théorique :**

| departement | nb_employes | salaire_moyen | ecart_type | mediane |
|-------------|-------------|---------------|------------|---------|
| IT          | 25          | 55000         | 12000      | 52000   |
| Finance     | 18          | 52000         | 8000       | 50000   |
| Marketing   | 30          | 42000         | 6000       | 41000   |
| RH          | 12          | 38000         | 4000       | 37500   |

**Observations :**
- IT a le salaire moyen le plus élevé mais aussi la plus forte dispersion (12k)
- RH a une faible dispersion (4k) → salaires très homogènes
- Dans IT, moyenne (55k) > médiane (52k) → quelques très hauts salaires tirent la moyenne vers le haut

---

## Tableau Récapitulatif des Fonctions Statistiques

| Fonction | Objectif | Résultat Typique | Interprétation |
|----------|----------|------------------|----------------|
| **VARIANCE(col)** | Dispersion des valeurs | Nombre positif (unité²) | Plus élevé = plus dispersé |
| **STDDEV(col)** | Écart-type (√variance) | Nombre positif (même unité que col) | "Distance moyenne" à la moyenne |
| **COVAR_SAMP(Y,X)** | Covariation entre X et Y | Nombre (peut être négatif) | Relation (mais dépend de l'échelle) |
| **CORR(Y,X)** | Corrélation de Pearson | -1 à +1 | Force et sens de la relation linéaire |
| **PERCENTILE_CONT(p)** | Percentile continu | Valeur (peut être interpolée) | Seuil : p% en dessous |
| **PERCENTILE_DISC(p)** | Percentile discret | Valeur réelle du dataset | Seuil : p% en dessous |
| **REGR_SLOPE(Y,X)** | Pente de régression | Nombre | Coefficient directeur de la droite |
| **REGR_INTERCEPT(Y,X)** | Ordonnée origine | Nombre | Valeur de Y quand X=0 |
| **REGR_R2(Y,X)** | Qualité ajustement | 0 à 1 | % de variance expliquée |

---

## Cas d'Usage Pratiques par Domaine

### 📊 Finance & Trading

```sql
-- Analyse de volatilité d'un actif financier
SELECT
    ticker,
    AVG(prix_cloture) AS prix_moyen,
    STDDEV(prix_cloture) AS volatilite,
    (STDDEV(prix_cloture) / AVG(prix_cloture)) * 100 AS coeff_variation_pct
FROM cours_bourse  
WHERE date >= CURRENT_DATE - INTERVAL '30 days'  
GROUP BY ticker  
ORDER BY volatilite DESC;  
```

### 🏥 Santé & Biologie

```sql
-- Distribution des temps d'attente aux urgences
SELECT
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY duree_attente_min) AS q1,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY duree_attente_min) AS mediane,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY duree_attente_min) AS q3,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duree_attente_min) AS p95
FROM admissions_urgences  
WHERE date_admission >= CURRENT_DATE - INTERVAL '7 days';  
```

### 🌐 Performance Web & SRE

```sql
-- SLA monitoring : percentiles de latence
SELECT
    endpoint,
    COUNT(*) AS nb_requetes,
    ROUND(AVG(latence_ms), 2) AS latence_moyenne,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY latence_ms) AS p50,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latence_ms) AS p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latence_ms) AS p99
FROM logs_http  
WHERE timestamp >= NOW() - INTERVAL '1 hour'  
GROUP BY endpoint  
HAVING COUNT(*) > 100  -- Minimum de requêtes pour statistiques fiables  
ORDER BY p95 DESC;  
```

### 🎓 Éducation

```sql
-- Analyse de la distribution des notes par matière
SELECT
    matiere,
    AVG(note) AS note_moyenne,
    STDDEV(note) AS ecart_type,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY note) AS mediane,
    -- Écart entre moyenne et médiane (asymétrie)
    AVG(note) - PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY note) AS skew_indicator
FROM resultats_examens  
WHERE annee_scolaire = '2024-2025'  
GROUP BY matiere;  
```

### 🛒 E-commerce & Marketing

```sql
-- Corrélation entre comportement de navigation et conversion
SELECT
    CORR(montant_achat, nb_pages_vues) AS corr_pages_achat,
    CORR(montant_achat, duree_session_sec) AS corr_duree_achat,
    CORR(montant_achat, nb_produits_favoris) AS corr_favoris_achat
FROM sessions_utilisateurs  
WHERE a_achete = TRUE  
  AND date_session >= CURRENT_DATE - INTERVAL '90 days';
```

---

## Bonnes Pratiques et Pièges à Éviter

### ✅ Bonnes Pratiques

1. **Toujours combiner moyenne et médiane pour détecter l'asymétrie**
   ```sql
   SELECT
       AVG(prix) AS moyenne,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY prix) AS mediane
   FROM produits;
   -- Si moyenne >> médiane → quelques valeurs très élevées
   ```

2. **Utiliser STDDEV_SAMP (ou STDDEV) par défaut**
   - Sauf si vous avez vraiment la population totale (très rare)

3. **Préférer les percentiles aux moyennes pour les SLA**
   - P95, P99 révèlent l'expérience des utilisateurs "malchanceux"

4. **Vérifier la taille de l'échantillon avant d'interpréter les statistiques**
   ```sql
   SELECT
       COUNT(*) AS taille_echantillon,
       STDDEV(mesure) AS ecart_type
   FROM donnees
   HAVING COUNT(*) > 30;  -- Minimum statistique
   ```

5. **Arrondir les résultats pour la lisibilité**
   ```sql
   SELECT ROUND(STDDEV(prix), 2) AS ecart_type
   FROM produits;
   ```

### ❌ Pièges à Éviter

1. **Confondre corrélation et causalité**
   - CORR(X,Y) = 0.95 ne signifie PAS que X cause Y !

2. **Utiliser VAR_POP/STDDEV_POP par erreur**
   - Dans le doute, utilisez VAR_SAMP/STDDEV_SAMP

3. **Interpréter la variance sans contexte**
   - La variance seule (en unité²) est difficile à comprendre → utilisez l'écart-type

4. **Ignorer les valeurs NULL**
   - Ces fonctions ignorent automatiquement les NULL
   - Vérifiez si c'est le comportement souhaité !

5. **Statistiques sur des petits échantillons**
   - Avec moins de 30 valeurs, les résultats sont peu fiables
   - Toujours afficher COUNT(*) à côté des statistiques

6. **Oublier WITHIN GROUP pour PERCENTILE_CONT/DISC**
   ```sql
   -- ❌ ERREUR
   SELECT PERCENTILE_CONT(0.5, prix) FROM produits;

   -- ✅ CORRECT
   SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY prix) FROM produits;
   ```

---

## Performance et Optimisation

### Coût de Calcul

Les fonctions statistiques nécessitent de **parcourir toutes les lignes** (full table scan) :

- **COUNT, SUM, AVG** : Scan simple (rapide)  
- **STDDEV, VARIANCE** : Deux passes (moyenne d'abord, puis variance)  
- **PERCENTILE** : Tri complet des données (le plus coûteux)  
- **CORR, COVAR** : Calculs croisés sur deux colonnes

### Optimisations

1. **Filtrer avec WHERE avant d'agréger**
   ```sql
   -- ✅ Bon : filtrer d'abord
   SELECT STDDEV(prix)
   FROM produits
   WHERE categorie = 'Électronique'
     AND date_ajout >= '2024-01-01';

   -- ❌ Moins bon : agréger puis filtrer
   SELECT STDDEV(prix)
   FROM produits
   HAVING STDDEV(prix) > 100;  -- Calcul sur toute la table !
   ```

2. **Utiliser des index pour les préfiltres**
   - Index sur colonnes utilisées dans WHERE avant agrégation

3. **Matérialiser les statistiques dans des tables de résumé**
   ```sql
   -- Pour des dashboards, précalculer quotidiennement :
   CREATE TABLE stats_ventes_quotidiennes AS
   SELECT
       date_vente::DATE AS jour,
       COUNT(*) AS nb_ventes,
       AVG(montant) AS panier_moyen,
       STDDEV(montant) AS ecart_type_panier,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY montant) AS mediane
   FROM ventes
   GROUP BY date_vente::DATE;
   ```

4. **Approximations pour les très grandes tables**
   - PostgreSQL 18+ : Certaines extensions proposent des statistiques approximatives (HyperLogLog)
   - Pour des milliards de lignes, considérer l'échantillonnage

---

## Exemples Avancés : Combinaison de Fonctions

### Analyse de Distribution Complète

```sql
-- "Boîte à moustaches" (Box plot) en SQL
WITH stats AS (
    SELECT
        MIN(salaire) AS minimum,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salaire) AS q1,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salaire) AS mediane,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salaire) AS q3,
        MAX(salaire) AS maximum,
        AVG(salaire) AS moyenne,
        STDDEV(salaire) AS ecart_type
    FROM employes
)
SELECT
    minimum,
    q1,
    mediane,
    moyenne,
    q3,
    maximum,
    ecart_type,
    -- Écart interquartile (robuste aux outliers)
    (q3 - q1) AS iqr,
    -- Coefficient de variation (écart-type relatif)
    (ecart_type / moyenne) * 100 AS coeff_variation_pct
FROM stats;
```

### Détection d'Outliers avec Z-Score

```sql
-- Identifier les valeurs aberrantes (> 3 écarts-types)
WITH stats AS (
    SELECT
        AVG(prix) AS moyenne,
        STDDEV(prix) AS ecart_type
    FROM produits
)
SELECT
    p.id_produit,
    p.nom,
    p.prix,
    -- Z-score : nombre d'écarts-types par rapport à la moyenne
    (p.prix - s.moyenne) / s.ecart_type AS z_score
FROM produits p, stats s  
WHERE ABS((p.prix - s.moyenne) / s.ecart_type) > 3  
ORDER BY ABS((p.prix - s.moyenne) / s.ecart_type) DESC;  
```

### Analyse de Tendance Temporelle

```sql
-- Régression linéaire : les ventes augmentent-elles avec le temps ?
WITH series AS (
    SELECT
        date_vente,
        EXTRACT(EPOCH FROM date_vente) AS temps_unix,  -- Convertir date en nombre
        montant_vente
    FROM ventes
    WHERE date_vente >= '2024-01-01'
)
SELECT
    REGR_SLOPE(montant_vente, temps_unix) AS tendance_journaliere,
    REGR_INTERCEPT(montant_vente, temps_unix) AS base,
    REGR_R2(montant_vente, temps_unix) AS qualite_fit,
    CORR(montant_vente, temps_unix) AS correlation
FROM series;

-- Si tendance_journaliere > 0 → ventes en croissance
-- Si R² proche de 1 → tendance linéaire claire
```

---

## À Retenir

1. **VARIANCE et STDDEV** mesurent la dispersion des données (STDDEV est plus intuitif)  
2. **CORR** (entre -1 et +1) révèle les relations entre variables  
3. **Corrélation ≠ Causalité** : toujours analyser le contexte métier  
4. **PERCENTILE** est plus robuste que la moyenne pour les distributions asymétriques  
5. **PERCENTILE_CONT** = interpolation, **PERCENTILE_DISC** = valeur réelle  
6. Les fonctions de **régression** (REGR_*) permettent de prédire et détecter des tendances  
7. Toujours vérifier la **taille de l'échantillon** (>30 pour fiabilité)  
8. Préférer **STDDEV_SAMP** et **VAR_SAMP** (échantillon) sauf cas très spécifiques

---

## Prochaines Étapes

Maintenant que vous maîtrisez les fonctions d'agrégation statistiques, vous êtes prêt pour :

- **Section 8.3** : GROUP BY et HAVING - Agréger par groupes et filtrer les résultats  
- **Section 8.4** : Extensions de groupement (ROLLUP, CUBE, GROUPING SETS)  
- **Section 10** : Window Functions - Agrégations sans perdre les détails de chaque ligne !

Les statistiques en SQL sont un **superpouvoir** pour l'analyse de données. PostgreSQL vous donne les outils d'un data scientist directement dans votre base de données !

⏭️ [GROUP BY et HAVING : Filtrage post-agrégation](/08-agregation-et-groupement/03-group-by-et-having.md)
