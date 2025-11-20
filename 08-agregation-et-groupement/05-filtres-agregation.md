🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.5. Filtres d'Agrégation (FILTER clause)

## Introduction : Le Besoin d'Agrégations Conditionnelles

Jusqu'à présent, quand nous utilisions des fonctions d'agrégation (COUNT, SUM, AVG, etc.), elles s'appliquaient à **toutes les lignes** d'un groupe. Mais que faire si vous voulez calculer plusieurs agrégations avec **des conditions différentes** dans la même requête ?

### Exemple de Besoin Métier

Imaginez une table `ventes` avec des transactions :

| id_vente | montant | statut    | date_vente | region |
|----------|---------|-----------|------------|--------|
| 1        | 150     | Validée   | 2024-01-15 | Nord   |
| 2        | 200     | Annulée   | 2024-01-16 | Sud    |
| 3        | 75      | Validée   | 2024-01-17 | Nord   |
| 4        | 300     | Validée   | 2024-01-18 | Est    |
| 5        | 125     | Annulée   | 2024-01-19 | Sud    |

**Questions :**
- Combien de ventes validées ?
- Combien de ventes annulées ?
- Montant total des ventes validées ?
- Montant total des ventes annulées ?
- **Le tout dans une seule ligne de résultat !**

Sans la clause FILTER, le code devient rapidement verbeux et répétitif.

---

## L'Approche Classique : CASE WHEN

### Solution Sans FILTER

Avant PostgreSQL 9.4, il fallait utiliser CASE WHEN à l'intérieur des agrégations :

```sql
SELECT
    -- Compter les ventes validées
    COUNT(CASE WHEN statut = 'Validée' THEN 1 END) AS nb_validees,

    -- Compter les ventes annulées
    COUNT(CASE WHEN statut = 'Annulée' THEN 1 END) AS nb_annulees,

    -- Total des ventes validées
    SUM(CASE WHEN statut = 'Validée' THEN montant ELSE 0 END) AS ca_valide,

    -- Total des ventes annulées
    SUM(CASE WHEN statut = 'Annulée' THEN montant ELSE 0 END) AS ca_annule
FROM ventes;
```

**Résultat :**

| nb_validees | nb_annulees | ca_valide | ca_annule |
|-------------|-------------|-----------|-----------|
| 3           | 2           | 525       | 325       |

**Problèmes avec cette approche :**
- ❌ **Verbeux** : Beaucoup de répétition (CASE WHEN... THEN... END)
- ❌ **Moins lisible** : La logique métier est noyée dans la syntaxe
- ❌ **Erreurs fréquentes** : Facile d'oublier ELSE 0 dans SUM, ou de mettre ELSE NULL dans COUNT
- ❌ **Difficile à maintenir** : Modifier une condition nécessite de toucher plusieurs endroits

---

## La Solution Moderne : FILTER Clause

### Qu'est-ce que FILTER ?

**FILTER** est une clause qui s'applique directement à une fonction d'agrégation pour **filtrer les lignes prises en compte** avant d'effectuer le calcul.

### Syntaxe

```sql
fonction_agregation(colonne) FILTER (WHERE condition)
```

**Traduction littérale** : "Applique cette agrégation uniquement sur les lignes qui satisfont cette condition"

### Exemple : Réécriture avec FILTER

```sql
SELECT
    -- Compter les ventes validées
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,

    -- Compter les ventes annulées
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS nb_annulees,

    -- Total des ventes validées
    SUM(montant) FILTER (WHERE statut = 'Validée') AS ca_valide,

    -- Total des ventes annulées
    SUM(montant) FILTER (WHERE statut = 'Annulée') AS ca_annule
FROM ventes;
```

**Résultat :**

| nb_validees | nb_annulees | ca_valide | ca_annule |
|-------------|-------------|-----------|-----------|
| 3           | 2           | 525       | 325       |

**Avantages :**
- ✅ **Clair et lisible** : L'intention est immédiatement visible
- ✅ **Concis** : Moins de code répétitif
- ✅ **Maintenable** : Conditions faciles à modifier
- ✅ **SQL Standard** : Conforme à la norme SQL:2003

---

## Comment ça Fonctionne ?

### Processus d'Exécution

Pour `SUM(montant) FILTER (WHERE statut = 'Validée')` :

1. PostgreSQL examine chaque ligne
2. Si `statut = 'Validée'` est vrai, la ligne est **incluse** dans le calcul
3. Si faux, la ligne est **ignorée** (comme si elle n'existait pas pour cette agrégation)
4. La fonction SUM s'applique uniquement aux lignes sélectionnées

**Visualisation :**

```
Table originale :
- Ligne 1 : montant=150, statut='Validée'   → ✅ Incluse dans SUM
- Ligne 2 : montant=200, statut='Annulée'   → ❌ Ignorée
- Ligne 3 : montant=75,  statut='Validée'   → ✅ Incluse dans SUM
- Ligne 4 : montant=300, statut='Validée'   → ✅ Incluse dans SUM
- Ligne 5 : montant=125, statut='Annulée'   → ❌ Ignorée

Résultat : SUM(150 + 75 + 300) = 525
```

---

## Exemples Progressifs

### Exemple 1 : Comptages Conditionnels Multiples

```sql
-- Statistiques de ventes par statut
SELECT
    COUNT(*) AS total_ventes,
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS nb_annulees,
    COUNT(*) FILTER (WHERE statut = 'En attente') AS nb_en_attente,

    -- Pourcentages
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE statut = 'Validée') / COUNT(*),
        2
    ) AS pct_validees
FROM ventes;
```

**Résultat théorique :**

| total_ventes | nb_validees | nb_annulees | nb_en_attente | pct_validees |
|--------------|-------------|-------------|---------------|--------------|
| 5            | 3           | 2           | 0             | 60.00        |

### Exemple 2 : Agrégations Numériques Conditionnelles

```sql
-- Analyse financière des ventes
SELECT
    -- Montants totaux
    SUM(montant) AS ca_total,
    SUM(montant) FILTER (WHERE statut = 'Validée') AS ca_valide,
    SUM(montant) FILTER (WHERE statut = 'Annulée') AS ca_perdu,

    -- Montants moyens
    AVG(montant) AS panier_moyen_global,
    AVG(montant) FILTER (WHERE statut = 'Validée') AS panier_moyen_valide,

    -- Extrema
    MAX(montant) FILTER (WHERE statut = 'Validée') AS vente_max_validee,
    MIN(montant) FILTER (WHERE statut = 'Validée') AS vente_min_validee
FROM ventes;
```

**Résultat théorique :**

| ca_total | ca_valide | ca_perdu | panier_moyen_global | panier_moyen_valide | vente_max_validee | vente_min_validee |
|----------|-----------|----------|---------------------|---------------------|-------------------|-------------------|
| 850      | 525       | 325      | 170.00              | 175.00              | 300               | 75                |

### Exemple 3 : Conditions Complexes

```sql
-- Analyse par tranches de montant
SELECT
    COUNT(*) AS total_ventes,

    -- Petits paniers (< 100€)
    COUNT(*) FILTER (WHERE montant < 100) AS nb_petits_paniers,
    SUM(montant) FILTER (WHERE montant < 100) AS ca_petits_paniers,

    -- Paniers moyens (100-200€)
    COUNT(*) FILTER (WHERE montant BETWEEN 100 AND 200) AS nb_paniers_moyens,
    SUM(montant) FILTER (WHERE montant BETWEEN 100 AND 200) AS ca_paniers_moyens,

    -- Gros paniers (> 200€)
    COUNT(*) FILTER (WHERE montant > 200) AS nb_gros_paniers,
    SUM(montant) FILTER (WHERE montant > 200) AS ca_gros_paniers
FROM ventes
WHERE statut = 'Validée';  -- Filtre global en amont
```

### Exemple 4 : Conditions Temporelles

```sql
-- Comparaison ventes récentes vs anciennes
SELECT
    -- Total global
    COUNT(*) AS total_ventes,
    SUM(montant) AS ca_total,

    -- Ventes du mois en cours
    COUNT(*) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE)
    ) AS ventes_mois_courant,
    SUM(montant) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE)
    ) AS ca_mois_courant,

    -- Ventes du mois précédent
    COUNT(*) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
          AND date_vente < DATE_TRUNC('month', CURRENT_DATE)
    ) AS ventes_mois_precedent,
    SUM(montant) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
          AND date_vente < DATE_TRUNC('month', CURRENT_DATE)
    ) AS ca_mois_precedent
FROM ventes;
```

---

## FILTER avec GROUP BY

La puissance de FILTER se révèle vraiment quand on la combine avec GROUP BY !

### Exemple : Statistiques par Région

```sql
-- Analyse des ventes par région
SELECT
    region,
    COUNT(*) AS total_ventes,

    -- Ventes par statut
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS nb_annulees,

    -- Chiffre d'affaires par statut
    SUM(montant) AS ca_total,
    SUM(montant) FILTER (WHERE statut = 'Validée') AS ca_valide,
    SUM(montant) FILTER (WHERE statut = 'Annulée') AS ca_perdu,

    -- Taux de conversion
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE statut = 'Validée') / COUNT(*),
        2
    ) AS taux_validation_pct
FROM ventes
GROUP BY region
ORDER BY ca_valide DESC;
```

**Résultat théorique :**

| region | total_ventes | nb_validees | nb_annulees | ca_total | ca_valide | ca_perdu | taux_validation_pct |
|--------|--------------|-------------|-------------|----------|-----------|----------|---------------------|
| Nord   | 2            | 2           | 0           | 225      | 225       | 0        | 100.00              |
| Est    | 1            | 1           | 0           | 300      | 300       | 0        | 100.00              |
| Sud    | 2            | 0           | 2           | 325      | 0         | 325      | 0.00                |

**Interprétation :**
- Une ligne par région
- Chaque ligne contient plusieurs agrégations avec des conditions différentes
- Le Sud a un problème : 0% de ventes validées !

### Exemple : Analyse Temporelle par Produit

```sql
-- Ventes par produit : ce mois vs mois précédent
SELECT
    produit,

    -- Mois en cours
    COUNT(*) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE)
    ) AS ventes_mois_actuel,
    SUM(montant) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE)
    ) AS ca_mois_actuel,

    -- Mois précédent
    COUNT(*) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
          AND date_vente < DATE_TRUNC('month', CURRENT_DATE)
    ) AS ventes_mois_precedent,
    SUM(montant) FILTER (
        WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
          AND date_vente < DATE_TRUNC('month', CURRENT_DATE)
    ) AS ca_mois_precedent,

    -- Évolution
    ROUND(
        100.0 * (
            COALESCE(SUM(montant) FILTER (WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE)), 0) -
            COALESCE(SUM(montant) FILTER (WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
              AND date_vente < DATE_TRUNC('month', CURRENT_DATE)), 0)
        ) / NULLIF(SUM(montant) FILTER (WHERE date_vente >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
              AND date_vente < DATE_TRUNC('month', CURRENT_DATE)), 0),
        2
    ) AS evolution_pct
FROM ventes
GROUP BY produit
ORDER BY ca_mois_actuel DESC;
```

---

## FILTER avec Fonctions Statistiques

FILTER fonctionne avec **toutes les fonctions d'agrégation**, y compris les fonctions statistiques !

### Exemple : Statistiques Conditionnelles

```sql
-- Analyse de la dispersion des ventes par statut
SELECT
    -- Ventes validées
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,
    AVG(montant) FILTER (WHERE statut = 'Validée') AS moyenne_validees,
    STDDEV(montant) FILTER (WHERE statut = 'Validée') AS ecart_type_validees,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY montant)
        FILTER (WHERE statut = 'Validée') AS mediane_validees,

    -- Ventes annulées
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS nb_annulees,
    AVG(montant) FILTER (WHERE statut = 'Annulée') AS moyenne_annulees,
    STDDEV(montant) FILTER (WHERE statut = 'Annulée') AS ecart_type_annulees,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY montant)
        FILTER (WHERE statut = 'Annulée') AS mediane_annulees
FROM ventes;
```

**Note importante** : Avec PERCENTILE_CONT/DISC, la syntaxe est légèrement différente :
```sql
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY montant) FILTER (WHERE condition)
```

---

## Cas d'Usage Réels par Domaine

### 📊 E-commerce : Analyse de Conversion

```sql
-- Funnel de conversion par étape
SELECT
    source_trafic,

    -- Étape 1 : Visites
    COUNT(DISTINCT session_id) AS nb_visites,

    -- Étape 2 : Consultations produit
    COUNT(DISTINCT session_id) FILTER (
        WHERE a_consulte_produit = TRUE
    ) AS nb_consultations_produit,

    -- Étape 3 : Ajout au panier
    COUNT(DISTINCT session_id) FILTER (
        WHERE a_ajoute_panier = TRUE
    ) AS nb_ajouts_panier,

    -- Étape 4 : Initiation paiement
    COUNT(DISTINCT session_id) FILTER (
        WHERE a_initie_paiement = TRUE
    ) AS nb_paiements_inities,

    -- Étape 5 : Achat finalisé
    COUNT(DISTINCT session_id) FILTER (
        WHERE a_achete = TRUE
    ) AS nb_achats,

    -- Taux de conversion global
    ROUND(
        100.0 * COUNT(DISTINCT session_id) FILTER (WHERE a_achete = TRUE)
        / COUNT(DISTINCT session_id),
        2
    ) AS taux_conversion_pct,

    -- Revenus
    SUM(montant_achat) FILTER (WHERE a_achete = TRUE) AS ca_total
FROM sessions_utilisateurs
WHERE date_session >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY source_trafic
ORDER BY nb_visites DESC;
```

### 🏥 Santé : Statistiques Hospitalières

```sql
-- Admissions urgences par service et gravité
SELECT
    service,

    -- Total admissions
    COUNT(*) AS total_admissions,

    -- Par niveau de gravité
    COUNT(*) FILTER (WHERE niveau_gravite = 'Faible') AS gravite_faible,
    COUNT(*) FILTER (WHERE niveau_gravite = 'Modérée') AS gravite_moderee,
    COUNT(*) FILTER (WHERE niveau_gravite = 'Élevée') AS gravite_elevee,
    COUNT(*) FILTER (WHERE niveau_gravite = 'Critique') AS gravite_critique,

    -- Durées de séjour moyennes par gravité
    ROUND(AVG(duree_sejour_heures) FILTER (WHERE niveau_gravite = 'Faible'), 1)
        AS duree_moy_faible,
    ROUND(AVG(duree_sejour_heures) FILTER (WHERE niveau_gravite = 'Critique'), 1)
        AS duree_moy_critique,

    -- Admissions hors heures (20h-8h)
    COUNT(*) FILTER (
        WHERE EXTRACT(HOUR FROM timestamp_admission) >= 20
           OR EXTRACT(HOUR FROM timestamp_admission) < 8
    ) AS admissions_nuit,

    -- Pourcentage d'hospitalisations
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE hospitalise = TRUE) / COUNT(*),
        2
    ) AS taux_hospitalisation_pct
FROM admissions_urgences
WHERE date_admission >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY service
ORDER BY total_admissions DESC;
```

### 💰 Finance : Analyse de Transactions

```sql
-- Analyse des transactions bancaires par compte
SELECT
    compte_id,

    -- Volume de transactions
    COUNT(*) AS total_transactions,
    COUNT(*) FILTER (WHERE type_transaction = 'Débit') AS nb_debits,
    COUNT(*) FILTER (WHERE type_transaction = 'Crédit') AS nb_credits,

    -- Montants
    SUM(montant) FILTER (WHERE type_transaction = 'Débit') AS total_debits,
    SUM(montant) FILTER (WHERE type_transaction = 'Crédit') AS total_credits,
    SUM(montant) FILTER (WHERE type_transaction = 'Crédit') -
    SUM(montant) FILTER (WHERE type_transaction = 'Débit') AS solde_variation,

    -- Transactions suspectes
    COUNT(*) FILTER (
        WHERE montant > 10000
           OR (type_transaction = 'Débit'
               AND heure_transaction BETWEEN 2 AND 5)
    ) AS transactions_suspectes,

    -- Transactions internationales
    COUNT(*) FILTER (WHERE pays_transaction != 'FR') AS nb_internationales,
    SUM(montant) FILTER (WHERE pays_transaction != 'FR') AS montant_international
FROM transactions
WHERE date_transaction >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY compte_id
HAVING COUNT(*) FILTER (
    WHERE montant > 10000
       OR (type_transaction = 'Débit' AND heure_transaction BETWEEN 2 AND 5)
) > 0  -- Seulement les comptes avec activité suspecte
ORDER BY transactions_suspectes DESC;
```

### 🌐 SaaS : Métriques d'API

```sql
-- Performance et fiabilité des endpoints API
SELECT
    endpoint,

    -- Volume
    COUNT(*) AS total_requetes,

    -- Par code HTTP
    COUNT(*) FILTER (WHERE status_code BETWEEN 200 AND 299) AS requetes_success,
    COUNT(*) FILTER (WHERE status_code BETWEEN 400 AND 499) AS erreurs_client,
    COUNT(*) FILTER (WHERE status_code BETWEEN 500 AND 599) AS erreurs_serveur,

    -- Taux d'erreur
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE status_code >= 400) / COUNT(*),
        2
    ) AS taux_erreur_pct,

    -- Latence (ms)
    ROUND(AVG(latence_ms), 2) AS latence_moyenne,
    ROUND(AVG(latence_ms) FILTER (WHERE status_code BETWEEN 200 AND 299), 2)
        AS latence_success,
    ROUND(AVG(latence_ms) FILTER (WHERE status_code >= 500), 2)
        AS latence_erreurs,

    -- Percentiles de latence
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latence_ms) AS p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latence_ms) AS p99,

    -- Requêtes lentes (> 1s)
    COUNT(*) FILTER (WHERE latence_ms > 1000) AS nb_requetes_lentes,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE latence_ms > 1000) / COUNT(*),
        2
    ) AS pct_requetes_lentes
FROM logs_api
WHERE timestamp >= NOW() - INTERVAL '1 hour'
GROUP BY endpoint
HAVING COUNT(*) > 100  -- Minimum de volume
ORDER BY taux_erreur_pct DESC, latence_moyenne DESC;
```

### 🎓 Éducation : Analyse Pédagogique

```sql
-- Performance des étudiants par matière
SELECT
    matiere,

    -- Total évaluations
    COUNT(*) AS total_evaluations,
    COUNT(DISTINCT etudiant_id) AS nb_etudiants,

    -- Répartition des notes
    COUNT(*) FILTER (WHERE note >= 16) AS excellent,
    COUNT(*) FILTER (WHERE note >= 14 AND note < 16) AS tres_bien,
    COUNT(*) FILTER (WHERE note >= 12 AND note < 14) AS bien,
    COUNT(*) FILTER (WHERE note >= 10 AND note < 12) AS assez_bien,
    COUNT(*) FILTER (WHERE note < 10) AS insuffisant,

    -- Statistiques
    ROUND(AVG(note), 2) AS moyenne_generale,
    ROUND(AVG(note) FILTER (WHERE note >= 10), 2) AS moyenne_reussite,
    ROUND(AVG(note) FILTER (WHERE note < 10), 2) AS moyenne_echec,

    -- Taux de réussite
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE note >= 10) / COUNT(*),
        2
    ) AS taux_reussite_pct,

    -- Étudiants en difficulté (< 8/20)
    COUNT(DISTINCT etudiant_id) FILTER (WHERE note < 8) AS etudiants_difficulte
FROM notes
WHERE annee_scolaire = '2024-2025'
  AND semestre = 1
GROUP BY matiere
ORDER BY moyenne_generale DESC;
```

---

## FILTER vs WHERE : Quelle Différence ?

### WHERE : Filtre Global

WHERE filtre les lignes **avant** toute agrégation. Toutes les agrégations travaillent sur le même sous-ensemble.

```sql
-- Seules les ventes validées sont considérées
SELECT
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total,
    AVG(montant) AS panier_moyen
FROM ventes
WHERE statut = 'Validée';  -- Filtre global
```

**Résultat :** Statistiques uniquement sur les ventes validées.

### FILTER : Filtre Par Agrégation

FILTER s'applique **individuellement** à chaque agrégation. Chaque agrégation peut avoir sa propre condition.

```sql
-- Chaque agrégation a son propre filtre
SELECT
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS nb_annulees,
    SUM(montant) FILTER (WHERE statut = 'Validée') AS ca_valide,
    SUM(montant) FILTER (WHERE statut = 'Annulée') AS ca_annule
FROM ventes;  -- Pas de WHERE : toutes les lignes
```

**Résultat :** Plusieurs statistiques avec des conditions différentes dans une seule ligne.

### Tableau Comparatif

| Aspect | WHERE | FILTER |
|--------|-------|--------|
| **Portée** | Globale (toute la requête) | Individuelle (par agrégation) |
| **Moment** | Avant agrégation | Pendant agrégation |
| **Flexibilité** | Une seule condition pour tout | Condition différente par agrégation |
| **Résultat** | Réduit le jeu de données | Permet plusieurs vues sur les mêmes données |
| **Usage** | Filtrer les données sources | Calculer plusieurs métriques conditionnelles |

### Combinaison WHERE + FILTER

Vous pouvez (et devriez souvent) combiner les deux :

```sql
SELECT
    region,

    -- WHERE : uniquement les ventes de 2024
    -- FILTER : ensuite, compter par statut
    COUNT(*) AS total_ventes_2024,
    COUNT(*) FILTER (WHERE statut = 'Validée') AS validees_2024,
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS annulees_2024,

    SUM(montant) FILTER (WHERE statut = 'Validée') AS ca_valide_2024
FROM ventes
WHERE EXTRACT(YEAR FROM date_vente) = 2024  -- Filtre global
GROUP BY region;
```

**Ordre d'exécution :**
1. WHERE filtre les ventes de 2024
2. GROUP BY crée les groupes par région
3. FILTER applique les conditions spécifiques à chaque agrégation

---

## FILTER vs CASE WHEN : Comparaison Détaillée

### Exemple Comparatif

**Avec CASE WHEN (ancien style) :**

```sql
SELECT
    region,
    COUNT(CASE WHEN statut = 'Validée' THEN 1 END) AS nb_validees,
    COUNT(CASE WHEN statut = 'Annulée' THEN 1 END) AS nb_annulees,
    SUM(CASE WHEN statut = 'Validée' THEN montant ELSE 0 END) AS ca_valide,
    SUM(CASE WHEN statut = 'Annulée' THEN montant ELSE 0 END) AS ca_annule,
    AVG(CASE WHEN statut = 'Validée' THEN montant END) AS panier_moyen_valide
FROM ventes
GROUP BY region;
```

**Avec FILTER (moderne) :**

```sql
SELECT
    region,
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,
    COUNT(*) FILTER (WHERE statut = 'Annulée') AS nb_annulees,
    SUM(montant) FILTER (WHERE statut = 'Validée') AS ca_valide,
    SUM(montant) FILTER (WHERE statut = 'Annulée') AS ca_annule,
    AVG(montant) FILTER (WHERE statut = 'Validée') AS panier_moyen_valide
FROM ventes
GROUP BY region;
```

### Avantages de FILTER

1. **Lisibilité** : L'intention est immédiatement claire
2. **Moins d'erreurs** : Pas besoin de gérer ELSE (NULL vs 0)
3. **Maintenabilité** : Modification d'une condition = un seul endroit
4. **Performance** : PostgreSQL peut optimiser FILTER plus facilement
5. **Standard SQL** : Conforme à SQL:2003

### Quand Utiliser CASE WHEN ?

CASE WHEN reste utile pour :
- **Transformations de valeurs** (non-agrégations)
- **Compatibilité** avec d'autres SGBD qui ne supportent pas FILTER
- **Logique complexe** avec plusieurs branches

```sql
-- CASE est approprié ici (pas d'agrégation)
SELECT
    nom,
    CASE
        WHEN note >= 16 THEN 'Excellent'
        WHEN note >= 14 THEN 'Très bien'
        WHEN note >= 12 THEN 'Bien'
        WHEN note >= 10 THEN 'Assez bien'
        ELSE 'Insuffisant'
    END AS appreciation
FROM notes;
```

---

## Performance et Optimisation

### FILTER est Généralement Plus Performant

PostgreSQL peut optimiser FILTER plus efficacement que CASE WHEN car :
1. L'intention est explicite
2. Le planificateur peut réorganiser les calculs
3. Moins d'évaluations conditionnelles en interne

### Exemple de Plan d'Exécution

```sql
EXPLAIN ANALYZE
SELECT
    region,
    COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees,
    SUM(montant) FILTER (WHERE statut = 'Validée') AS ca_valide
FROM ventes
GROUP BY region;
```

PostgreSQL génère un plan efficace qui évalue la condition une seule fois par ligne.

### Bonnes Pratiques de Performance

1. **Combiner WHERE et FILTER pour réduire le volume**
   ```sql
   -- ✅ Bon : WHERE réduit les données en amont
   SELECT
       COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_validees
   FROM ventes
   WHERE date_vente >= '2024-01-01';  -- Filtre tôt
   ```

2. **Éviter les FILTER complexes dans les sous-requêtes répétées**
   ```sql
   -- ❌ Moins bon : condition complexe répétée
   SELECT
       SUM(montant) FILTER (WHERE condition_complexe(...)) AS total1,
       AVG(montant) FILTER (WHERE condition_complexe(...)) AS total2
   FROM ventes;

   -- ✅ Meilleur : précalculer dans un CTE
   WITH ventes_filtrees AS (
       SELECT *, condition_complexe(...) AS satisfait_condition
       FROM ventes
   )
   SELECT
       SUM(montant) FILTER (WHERE satisfait_condition) AS total1,
       AVG(montant) FILTER (WHERE satisfait_condition) AS total2
   FROM ventes_filtrees;
   ```

3. **Index sur les colonnes utilisées dans FILTER**
   ```sql
   -- Si vous filtrez souvent par statut
   CREATE INDEX idx_ventes_statut ON ventes(statut);
   ```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Préférer FILTER à CASE WHEN pour les agrégations conditionnelles**
   ```sql
   -- ✅ Clair et moderne
   COUNT(*) FILTER (WHERE condition)

   -- ❌ Verbeux
   COUNT(CASE WHEN condition THEN 1 END)
   ```

2. **Nommer clairement les colonnes agrégées**
   ```sql
   SELECT
       COUNT(*) FILTER (WHERE statut = 'Validée') AS nb_ventes_validees,
       -- Pas juste "validees" ou "count"
   FROM ventes;
   ```

3. **Utiliser COALESCE pour gérer les NULL dans les résultats**
   ```sql
   SELECT
       region,
       COALESCE(SUM(montant) FILTER (WHERE statut = 'Validée'), 0) AS ca_valide
   FROM ventes
   GROUP BY region;
   -- Retourne 0 au lieu de NULL si aucune vente validée
   ```

4. **Documenter les conditions complexes**
   ```sql
   SELECT
       -- Ventes en heures creuses (20h-8h)
       COUNT(*) FILTER (
           WHERE EXTRACT(HOUR FROM date_vente) >= 20
              OR EXTRACT(HOUR FROM date_vente) < 8
       ) AS ventes_heures_creuses
   FROM ventes;
   ```

5. **Combiner WHERE (global) et FILTER (spécifique) intelligemment**
   ```sql
   SELECT
       COUNT(*) FILTER (WHERE montant > 100) AS gros_paniers
   FROM ventes
   WHERE date_vente >= '2024-01-01'  -- Filtre global d'abord
     AND statut != 'Brouillon';       -- Exclure les brouillons
   ```

### ❌ À Éviter

1. **Utiliser FILTER sur des colonnes non-agrégées**
   ```sql
   -- ❌ ERREUR : region n'est pas une agrégation
   SELECT
       region FILTER (WHERE statut = 'Validée')
   FROM ventes;

   -- ✅ Correct : utiliser WHERE
   SELECT region
   FROM ventes
   WHERE statut = 'Validée';
   ```

2. **Dupliquer la même condition FILTER**
   ```sql
   -- ❌ Répétitif
   SELECT
       COUNT(*) FILTER (WHERE date_vente >= '2024-01-01') AS nb,
       SUM(montant) FILTER (WHERE date_vente >= '2024-01-01') AS ca
   FROM ventes;

   -- ✅ Meilleur : utiliser WHERE
   SELECT
       COUNT(*) AS nb,
       SUM(montant) AS ca
   FROM ventes
   WHERE date_vente >= '2024-01-01';
   ```

3. **Oublier COALESCE avec SUM/AVG et FILTER**
   ```sql
   -- Si aucune ligne ne satisfait le filtre, retourne NULL
   SUM(montant) FILTER (WHERE condition)

   -- Préférer :
   COALESCE(SUM(montant) FILTER (WHERE condition), 0)
   ```

4. **Conditions trop complexes dans FILTER (lisibilité)**
   ```sql
   -- ❌ Difficile à lire
   COUNT(*) FILTER (WHERE (a > 10 AND b < 20 OR c = 'X') AND (d != 'Y' OR e IS NULL))

   -- ✅ Meilleur : CTE ou fonction
   WITH lignes_complexes AS (
       SELECT *,
              ((a > 10 AND b < 20 OR c = 'X') AND (d != 'Y' OR e IS NULL)) AS condition_ok
       FROM table
   )
   SELECT COUNT(*) FILTER (WHERE condition_ok)
   FROM lignes_complexes;
   ```

---

## Exemples Avancés

### Analyse de Cohortes

```sql
-- Rétention utilisateurs par cohorte d'inscription
SELECT
    DATE_TRUNC('month', date_inscription) AS cohorte,
    COUNT(DISTINCT user_id) AS utilisateurs_inscrits,

    -- Actifs mois 1 (même mois que l'inscription)
    COUNT(DISTINCT user_id) FILTER (
        WHERE derniere_connexion >= date_inscription
          AND derniere_connexion < date_inscription + INTERVAL '1 month'
    ) AS actifs_mois_1,

    -- Actifs mois 2
    COUNT(DISTINCT user_id) FILTER (
        WHERE derniere_connexion >= date_inscription + INTERVAL '1 month'
          AND derniere_connexion < date_inscription + INTERVAL '2 months'
    ) AS actifs_mois_2,

    -- Actifs mois 3
    COUNT(DISTINCT user_id) FILTER (
        WHERE derniere_connexion >= date_inscription + INTERVAL '2 months'
          AND derniere_connexion < date_inscription + INTERVAL '3 months'
    ) AS actifs_mois_3,

    -- Taux de rétention mois 3
    ROUND(
        100.0 * COUNT(DISTINCT user_id) FILTER (
            WHERE derniere_connexion >= date_inscription + INTERVAL '2 months'
        ) / COUNT(DISTINCT user_id),
        2
    ) AS retention_mois_3_pct
FROM utilisateurs
WHERE date_inscription >= '2024-01-01'
GROUP BY DATE_TRUNC('month', date_inscription)
ORDER BY cohorte;
```

### Analyse de Distribution Multi-Critères

```sql
-- Répartition des commandes par montant ET délai de livraison
SELECT
    -- Total
    COUNT(*) AS total_commandes,

    -- Matrice montant × délai
    COUNT(*) FILTER (WHERE montant < 50 AND delai_livraison_jours <= 2)
        AS petit_panier_rapide,
    COUNT(*) FILTER (WHERE montant < 50 AND delai_livraison_jours > 2)
        AS petit_panier_lent,
    COUNT(*) FILTER (WHERE montant >= 50 AND montant < 200 AND delai_livraison_jours <= 2)
        AS panier_moyen_rapide,
    COUNT(*) FILTER (WHERE montant >= 50 AND montant < 200 AND delai_livraison_jours > 2)
        AS panier_moyen_lent,
    COUNT(*) FILTER (WHERE montant >= 200 AND delai_livraison_jours <= 2)
        AS gros_panier_rapide,
    COUNT(*) FILTER (WHERE montant >= 200 AND delai_livraison_jours > 2)
        AS gros_panier_lent,

    -- Statistiques ciblées
    AVG(montant) FILTER (WHERE delai_livraison_jours <= 2) AS panier_moyen_livraison_rapide,
    AVG(note_satisfaction) FILTER (WHERE delai_livraison_jours <= 2) AS satisfaction_rapide,
    AVG(note_satisfaction) FILTER (WHERE delai_livraison_jours > 2) AS satisfaction_lente
FROM commandes
WHERE date_commande >= CURRENT_DATE - INTERVAL '30 days';
```

---

## À Retenir

1. **FILTER** applique une condition **spécifique à une agrégation**
2. Syntaxe : `fonction_agregation(colonne) FILTER (WHERE condition)`
3. **Plus lisible et maintenable** que CASE WHEN dans les agrégations
4. Fonctionne avec **toutes les fonctions d'agrégation** (COUNT, SUM, AVG, STDDEV, PERCENTILE, etc.)
5. **WHERE** = filtre global, **FILTER** = filtre par agrégation
6. Combinez WHERE (réduire volume) et FILTER (conditions spécifiques)
7. Utilisez **COALESCE** pour gérer les NULL dans les résultats
8. FILTER est un **standard SQL** (depuis SQL:2003)
9. **Performance** : Généralement meilleur que CASE WHEN
10. Idéal pour les **tableaux de bord** et **rapports multi-critères**

---

## Prochaines Étapes

Vous maîtrisez maintenant les filtres d'agrégation ! Prochains sujets :

- **Section 8.6** : Agrégations ordonnées (WITHIN GROUP, string_agg) - Agréger avec tri
- **Section 9** : Techniques SQL Avancées - Sous-requêtes, CTEs, récursion
- **Section 10** : Window Functions - Le summum de l'analyse SQL !

La clause **FILTER** est un outil moderne et puissant qui rend vos requêtes analytiques plus **claires, concises et performantes**. Elle est indispensable pour tout analyste de données travaillant avec PostgreSQL !

⏭️ [Agrégations ordonnées (WITHIN GROUP, string_agg)](/08-agregation-et-groupement/06-agregations-ordonnees.md)
