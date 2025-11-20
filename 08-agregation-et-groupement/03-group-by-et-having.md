🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.3. GROUP BY et HAVING : Filtrage Post-Agrégation

## Introduction au Groupement de Données

Jusqu'à présent, nous avons utilisé les fonctions d'agrégation (COUNT, SUM, AVG, etc.) pour calculer des statistiques sur **l'ensemble d'une table**. Ces agrégations produisaient **une seule ligne de résultat**.

Mais que se passe-t-il si vous voulez calculer des statistiques **par catégorie** ? Par exemple :
- Le chiffre d'affaires **par client**
- Le salaire moyen **par département**
- Le nombre de ventes **par produit**
- Les ventes totales **par mois**

C'est exactement le rôle de la clause **GROUP BY** : **diviser les données en groupes et calculer des agrégations pour chaque groupe**.

---

## Qu'est-ce que GROUP BY ?

### Définition

**GROUP BY** regroupe les lignes qui partagent les mêmes valeurs dans une ou plusieurs colonnes, puis applique des fonctions d'agrégation à chaque groupe.

### Métaphore Visuelle

Imaginez que vous avez une pile de factures d'entreprise. Sans GROUP BY, vous calculez le total de toutes les factures ensemble. Avec GROUP BY, vous :

1. **Triez les factures par client** (création des groupes)
2. **Calculez le total pour chaque pile** (agrégation par groupe)
3. **Obtenez un résultat par client** (une ligne par groupe)

---

## Syntaxe de Base

```sql
SELECT
    colonne_de_groupement,
    fonction_agregation(colonne)
FROM table
GROUP BY colonne_de_groupement;
```

### Règle Fondamentale

⚠️ **RÈGLE CRITIQUE** : Dans une requête avec GROUP BY, vous ne pouvez sélectionner que :
1. Les colonnes mentionnées dans GROUP BY
2. Des fonctions d'agrégation

Tout le reste produira une **erreur** !

---

## Premier Exemple : Groupement Simple

Considérons une table `ventes` :

| id_vente | client_id | produit      | montant | date_vente |
|----------|-----------|--------------|---------|------------|
| 1        | 101       | Laptop       | 899     | 2024-01-15 |
| 2        | 102       | Souris       | 25      | 2024-01-16 |
| 3        | 101       | Clavier      | 75      | 2024-01-17 |
| 4        | 103       | Écran        | 250     | 2024-01-18 |
| 5        | 102       | Laptop       | 899     | 2024-01-19 |
| 6        | 101       | Souris       | 25      | 2024-01-20 |

### Question : Quel est le montant total dépensé par chaque client ?

**Sans GROUP BY (erreur conceptuelle) :**
```sql
-- Ceci donne le total GLOBAL, pas par client
SELECT SUM(montant) AS total
FROM ventes;
-- Résultat : 2173 (une seule ligne)
```

**Avec GROUP BY (correct) :**
```sql
SELECT
    client_id,
    SUM(montant) AS total_depense
FROM ventes
GROUP BY client_id;
```

**Résultat :**

| client_id | total_depense |
|-----------|---------------|
| 101       | 999           |
| 102       | 924           |
| 103       | 250           |

**Ce qui s'est passé :**
1. PostgreSQL a regroupé toutes les lignes ayant le même `client_id`
2. Pour chaque groupe, il a calculé `SUM(montant)`
3. Il a retourné **une ligne par groupe** (donc 3 lignes, une par client)

---

## Comprendre le Fonctionnement Interne

### Visualisation du Processus

**Étape 1 : Division en Groupes**

```
Groupe client_id = 101:
  - Vente 1: Laptop, 899
  - Vente 3: Clavier, 75
  - Vente 6: Souris, 25

Groupe client_id = 102:
  - Vente 2: Souris, 25
  - Vente 5: Laptop, 899

Groupe client_id = 103:
  - Vente 4: Écran, 250
```

**Étape 2 : Agrégation par Groupe**

```
Groupe 101: SUM(899, 75, 25) = 999
Groupe 102: SUM(25, 899) = 924
Groupe 103: SUM(250) = 250
```

**Étape 3 : Résultat Final**

| client_id | total_depense |
|-----------|---------------|
| 101       | 999           |
| 102       | 924           |
| 103       | 250           |

---

## Grouper avec Plusieurs Fonctions d'Agrégation

Vous pouvez utiliser plusieurs fonctions d'agrégation dans la même requête :

```sql
SELECT
    client_id,
    COUNT(*) AS nombre_achats,
    SUM(montant) AS total_depense,
    AVG(montant) AS panier_moyen,
    MIN(montant) AS achat_min,
    MAX(montant) AS achat_max
FROM ventes
GROUP BY client_id
ORDER BY total_depense DESC;
```

**Résultat :**

| client_id | nombre_achats | total_depense | panier_moyen | achat_min | achat_max |
|-----------|---------------|---------------|--------------|-----------|-----------|
| 101       | 3             | 999           | 333.00       | 25        | 899       |
| 102       | 2             | 924           | 462.00       | 25        | 899       |
| 103       | 1             | 250           | 250.00       | 250       | 250       |

**Interprétation pour le client 101 :**
- A effectué 3 achats
- Total dépensé : 999 €
- Panier moyen : 333 €
- Achat le moins cher : 25 €
- Achat le plus cher : 899 €

---

## Groupement Multi-Colonnes

Vous pouvez grouper par **plusieurs colonnes** pour des analyses plus fines.

### Exemple : Ventes par Produit et par Mois

```sql
SELECT
    EXTRACT(MONTH FROM date_vente) AS mois,
    produit,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total
FROM ventes
GROUP BY EXTRACT(MONTH FROM date_vente), produit
ORDER BY mois, ca_total DESC;
```

**Résultat théorique :**

| mois | produit | nb_ventes | ca_total |
|------|---------|-----------|----------|
| 1    | Laptop  | 2         | 1798     |
| 1    | Écran   | 1         | 250      |
| 1    | Clavier | 1         | 75       |
| 1    | Souris  | 2         | 50       |

**Ce qui se passe :**
- Les lignes sont groupées par **combinaison unique** de (mois, produit)
- Il y a une ligne de résultat pour chaque combinaison

### Ordre des Colonnes dans GROUP BY

L'ordre des colonnes dans GROUP BY n'affecte **pas** le résultat, uniquement la logique conceptuelle :

```sql
-- Ces deux requêtes donnent le MÊME résultat
GROUP BY produit, mois
GROUP BY mois, produit
```

Mais l'ordre dans ORDER BY, lui, change l'affichage !

---

## L'Ordre d'Exécution SQL avec GROUP BY

### Ordre Logique d'Exécution

```sql
SELECT      -- 5. Sélection des colonnes finales
    client_id,
    SUM(montant) AS total
FROM ventes -- 1. Source des données
WHERE date_vente >= '2024-01-01'  -- 2. Filtrage des LIGNES
GROUP BY client_id                 -- 3. Regroupement
HAVING SUM(montant) > 500         -- 4. Filtrage des GROUPES
ORDER BY total DESC;              -- 6. Tri
```

**Séquence :**
1. **FROM** : Charger la table `ventes`
2. **WHERE** : Filtrer les lignes (uniquement celles de 2024)
3. **GROUP BY** : Créer les groupes par `client_id`
4. **HAVING** : Filtrer les groupes (garder ceux avec total > 500)
5. **SELECT** : Calculer les agrégations et sélectionner les colonnes
6. **ORDER BY** : Trier le résultat final

**Point clé** : WHERE s'exécute **AVANT** GROUP BY, HAVING s'exécute **APRÈS** !

---

## Introduction à HAVING

### Le Problème avec WHERE

Imaginez que vous voulez trouver les clients qui ont dépensé **plus de 500 €**. Vous pourriez essayer :

```sql
-- ❌ ERREUR : Impossible d'utiliser une agrégation dans WHERE
SELECT
    client_id,
    SUM(montant) AS total_depense
FROM ventes
WHERE SUM(montant) > 500  -- ❌ Erreur de syntaxe !
GROUP BY client_id;
```

**Pourquoi cette erreur ?**

WHERE s'exécute **AVANT** le groupement. À ce stade, `SUM(montant)` n'a pas encore été calculé !

### La Solution : HAVING

**HAVING** est une clause de filtrage qui s'applique **après** le groupement et les agrégations.

```sql
-- ✅ CORRECT : Filtrage des groupes avec HAVING
SELECT
    client_id,
    SUM(montant) AS total_depense
FROM ventes
GROUP BY client_id
HAVING SUM(montant) > 500
ORDER BY total_depense DESC;
```

**Résultat :**

| client_id | total_depense |
|-----------|---------------|
| 101       | 999           |
| 102       | 924           |

Le client 103 (total = 250) a été **exclu** car il ne satisfait pas la condition `SUM(montant) > 500`.

---

## WHERE vs HAVING : La Différence Fondamentale

| Critère | WHERE | HAVING |
|---------|-------|--------|
| **Moment d'exécution** | AVANT GROUP BY | APRÈS GROUP BY |
| **Filtre** | Lignes individuelles | Groupes entiers |
| **Peut utiliser agrégations** | ❌ Non | ✅ Oui |
| **Peut référencer colonnes non-agrégées** | ✅ Oui | ⚠️ Limité (doit être dans GROUP BY) |
| **Performance** | Meilleur (filtre tôt) | Moins bon (filtre tard) |

### Exemples Comparatifs

**Cas 1 : Filtrer des LIGNES avant groupement (WHERE)**

```sql
-- Trouver les dépenses totales par client, mais uniquement
-- pour les achats supérieurs à 50€
SELECT
    client_id,
    SUM(montant) AS total_depense
FROM ventes
WHERE montant > 50  -- Filtre les LIGNES
GROUP BY client_id;
```

**Processus :**
1. Exclure les achats de 25€ (souris)
2. Grouper par client sur les lignes restantes
3. Calculer les sommes

**Cas 2 : Filtrer des GROUPES après agrégation (HAVING)**

```sql
-- Trouver les clients qui ont dépensé au total plus de 500€
SELECT
    client_id,
    SUM(montant) AS total_depense
FROM ventes
GROUP BY client_id
HAVING SUM(montant) > 500;  -- Filtre les GROUPES
```

**Processus :**
1. Grouper toutes les lignes par client
2. Calculer les sommes
3. Exclure les groupes dont la somme ≤ 500€

---

## Combinaison de WHERE et HAVING

Vous pouvez (et devriez souvent) combiner WHERE et HAVING pour un filtrage en deux étapes :

```sql
SELECT
    client_id,
    COUNT(*) AS nb_achats,
    SUM(montant) AS total_depense
FROM ventes
WHERE date_vente >= '2024-01-01'     -- Filtre 1 : lignes (dates récentes)
GROUP BY client_id
HAVING COUNT(*) >= 2                 -- Filtre 2 : groupes (au moins 2 achats)
ORDER BY total_depense DESC;
```

**Résultat :**

| client_id | nb_achats | total_depense |
|-----------|-----------|---------------|
| 101       | 3         | 999           |
| 102       | 2         | 924           |

Le client 103 est exclu car il n'a effectué qu'un seul achat (HAVING COUNT(*) >= 2).

**Pourquoi cette approche ?**
- WHERE réduit le volume de données **avant** le groupement (plus efficace)
- HAVING affine les résultats sur les **statistiques calculées**

---

## Exemples Avancés de HAVING

### 1. Conditions Multiples dans HAVING

```sql
-- Trouver les clients VIP : au moins 3 achats ET dépense > 800€
SELECT
    client_id,
    COUNT(*) AS nb_achats,
    SUM(montant) AS total_depense
FROM ventes
GROUP BY client_id
HAVING COUNT(*) >= 3
   AND SUM(montant) > 800;
```

### 2. HAVING avec Plusieurs Agrégations

```sql
-- Clients avec un panier moyen élevé mais peu d'achats
SELECT
    client_id,
    COUNT(*) AS nb_achats,
    AVG(montant) AS panier_moyen,
    SUM(montant) AS total_depense
FROM ventes
GROUP BY client_id
HAVING COUNT(*) <= 2
   AND AVG(montant) > 400;
```

### 3. HAVING avec Expressions Calculées

```sql
-- Clients dont les achats sont très variables (écart-type élevé)
SELECT
    client_id,
    COUNT(*) AS nb_achats,
    STDDEV(montant) AS variabilite
FROM ventes
GROUP BY client_id
HAVING COUNT(*) >= 3                -- Minimum statistique
   AND STDDEV(montant) > 200;       -- Forte dispersion
```

---

## Utilisation d'Alias dans HAVING

### ⚠️ Attention : Portée des Alias

En SQL standard, les alias définis dans SELECT **ne sont pas accessibles** dans HAVING (car SELECT s'exécute après HAVING).

```sql
-- ❌ Peut ne pas fonctionner (dépend du SGBD)
SELECT
    client_id,
    SUM(montant) AS total
FROM ventes
GROUP BY client_id
HAVING total > 500;  -- ❌ "total" peut ne pas être reconnu
```

**Solution standard : Répéter l'expression**

```sql
-- ✅ CORRECT et portable
SELECT
    client_id,
    SUM(montant) AS total
FROM ventes
GROUP BY client_id
HAVING SUM(montant) > 500;
```

**Note PostgreSQL** : PostgreSQL **permet** d'utiliser les alias dans HAVING (extension non-standard), mais ce n'est pas portable vers d'autres SGBD.

---

## Patterns Courants avec GROUP BY et HAVING

### Pattern 1 : Top N par Catégorie

```sql
-- Top 3 des produits les plus vendus
SELECT
    produit,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total
FROM ventes
GROUP BY produit
ORDER BY nb_ventes DESC
LIMIT 3;
```

### Pattern 2 : Identification des Outliers

```sql
-- Clients avec un comportement d'achat inhabituel
SELECT
    client_id,
    COUNT(*) AS nb_achats,
    MAX(montant) AS achat_max
FROM ventes
GROUP BY client_id
HAVING MAX(montant) > (SELECT AVG(montant) * 3 FROM ventes);
-- Achat maximum > 3× la moyenne globale
```

### Pattern 3 : Analyse Temporelle

```sql
-- Jours avec un volume de ventes anormalement élevé
SELECT
    date_vente::DATE AS jour,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_jour
FROM ventes
GROUP BY date_vente::DATE
HAVING COUNT(*) > 10  -- Plus de 10 ventes
ORDER BY ca_jour DESC;
```

### Pattern 4 : Détection de Segments Inactifs

```sql
-- Clients qui n'ont pas acheté depuis 90 jours
SELECT
    client_id,
    MAX(date_vente) AS derniere_vente,
    CURRENT_DATE - MAX(date_vente) AS jours_inactivite
FROM ventes
GROUP BY client_id
HAVING MAX(date_vente) < CURRENT_DATE - INTERVAL '90 days'
ORDER BY derniere_vente;
```

---

## GROUP BY avec Expressions

Vous pouvez grouper par des **expressions calculées**, pas seulement des colonnes brutes.

### Exemple 1 : Groupement par Mois/Année

```sql
-- Ventes mensuelles
SELECT
    EXTRACT(YEAR FROM date_vente) AS annee,
    EXTRACT(MONTH FROM date_vente) AS mois,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_mensuel
FROM ventes
GROUP BY
    EXTRACT(YEAR FROM date_vente),
    EXTRACT(MONTH FROM date_vente)
ORDER BY annee, mois;
```

**Alternative avec DATE_TRUNC (PostgreSQL) :**

```sql
-- Plus élégant avec DATE_TRUNC
SELECT
    DATE_TRUNC('month', date_vente) AS mois,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_mensuel
FROM ventes
GROUP BY DATE_TRUNC('month', date_vente)
ORDER BY mois;
```

### Exemple 2 : Groupement par Tranches

```sql
-- Répartition des ventes par tranche de prix
SELECT
    CASE
        WHEN montant < 50 THEN 'Petit panier'
        WHEN montant BETWEEN 50 AND 200 THEN 'Panier moyen'
        WHEN montant BETWEEN 201 AND 500 THEN 'Gros panier'
        ELSE 'Très gros panier'
    END AS tranche_prix,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total
FROM ventes
GROUP BY
    CASE
        WHEN montant < 50 THEN 'Petit panier'
        WHEN montant BETWEEN 50 AND 200 THEN 'Panier moyen'
        WHEN montant BETWEEN 201 AND 500 THEN 'Gros panier'
        ELSE 'Très gros panier'
    END
ORDER BY
    MIN(montant);  -- Trier par le montant minimum de chaque tranche
```

---

## Cas d'Usage Réels par Domaine

### 📊 E-commerce : Analyse RFM (Recency, Frequency, Monetary)

```sql
-- Segmentation RFM des clients
SELECT
    client_id,
    MAX(date_vente) AS derniere_vente,                    -- Recency
    COUNT(*) AS nb_achats,                                 -- Frequency
    SUM(montant) AS total_depense,                         -- Monetary
    AVG(montant) AS panier_moyen
FROM ventes
WHERE date_vente >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY client_id
HAVING COUNT(*) >= 3  -- Clients récurrents uniquement
ORDER BY total_depense DESC;
```

### 🏥 Santé : Analyse de Consultations Médicales

```sql
-- Médecins avec un volume de consultations élevé
SELECT
    medecin_id,
    COUNT(*) AS nb_consultations,
    AVG(duree_consultation_min) AS duree_moyenne,
    COUNT(DISTINCT patient_id) AS nb_patients_uniques
FROM consultations
WHERE date_consultation >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY medecin_id
HAVING COUNT(*) > 100  -- Plus de 100 consultations/mois
ORDER BY nb_consultations DESC;
```

### 📈 Finance : Analyse de Transactions Bancaires

```sql
-- Comptes avec des transactions suspectes (montants inhabituels)
SELECT
    compte_id,
    COUNT(*) AS nb_transactions,
    SUM(montant) AS total_mouvement,
    MAX(montant) AS transaction_max,
    STDDEV(montant) AS variabilite
FROM transactions
WHERE date_transaction >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY compte_id
HAVING MAX(montant) > 10000  -- Transaction élevée
   OR STDDEV(montant) > 5000  -- Forte variabilité
ORDER BY transaction_max DESC;
```

### 🌐 SaaS : Analyse d'Utilisation d'API

```sql
-- Endpoints avec un taux d'erreur élevé
SELECT
    endpoint,
    COUNT(*) AS nb_requetes,
    SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) AS nb_erreurs,
    ROUND(
        100.0 * SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS taux_erreur_pct,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latence_ms) AS p95_latence
FROM logs_api
WHERE timestamp >= NOW() - INTERVAL '1 hour'
GROUP BY endpoint
HAVING COUNT(*) > 100  -- Minimum de requêtes
   AND SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) > 5  -- Au moins 5 erreurs
ORDER BY taux_erreur_pct DESC;
```

### 🎓 Éducation : Performance des Étudiants

```sql
-- Étudiants en difficulté (moyenne < 10 et au moins 3 évaluations)
SELECT
    etudiant_id,
    COUNT(*) AS nb_evaluations,
    AVG(note) AS moyenne,
    MIN(note) AS note_min,
    MAX(note) AS note_max
FROM notes
WHERE annee_scolaire = '2024-2025'
GROUP BY etudiant_id
HAVING COUNT(*) >= 3  -- Au moins 3 notes
   AND AVG(note) < 10  -- Moyenne insuffisante
ORDER BY moyenne;
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Filtrer avec WHERE avant de grouper (performance)**
   ```sql
   -- ✅ Bon : WHERE réduit les données AVANT groupement
   SELECT client_id, SUM(montant)
   FROM ventes
   WHERE date_vente >= '2024-01-01'  -- Filtre d'abord
   GROUP BY client_id;
   ```

2. **Utiliser des alias clairs pour les agrégations**
   ```sql
   -- ✅ Lisible
   SELECT
       client_id,
       SUM(montant) AS total_depense,
       COUNT(*) AS nb_achats
   FROM ventes
   GROUP BY client_id;
   ```

3. **Toujours inclure COUNT(*) pour comprendre la taille des groupes**
   ```sql
   SELECT
       categorie,
       COUNT(*) AS taille_echantillon,  -- Important !
       AVG(prix) AS prix_moyen
   FROM produits
   GROUP BY categorie;
   ```

4. **Ajouter HAVING COUNT(*) pour écarter les petits échantillons**
   ```sql
   SELECT
       categorie,
       COUNT(*) AS nb_produits,
       AVG(prix) AS prix_moyen
   FROM produits
   GROUP BY categorie
   HAVING COUNT(*) >= 10;  -- Statistiques fiables
   ```

5. **Ordonner les résultats pour une meilleure lisibilité**
   ```sql
   SELECT client_id, SUM(montant) AS total
   FROM ventes
   GROUP BY client_id
   ORDER BY total DESC;  -- Du plus gros au plus petit
   ```

### ❌ À Éviter

1. **Sélectionner des colonnes non-agrégées absentes de GROUP BY**
   ```sql
   -- ❌ ERREUR : produit n'est pas dans GROUP BY
   SELECT
       client_id,
       produit,  -- ❌ Quelle valeur retourner si le client a acheté plusieurs produits ?
       SUM(montant)
   FROM ventes
   GROUP BY client_id;

   -- ✅ Correct : soit ajouter produit au GROUP BY...
   GROUP BY client_id, produit;

   -- ✅ ...soit utiliser une agrégation
   SELECT
       client_id,
       STRING_AGG(produit, ', ') AS produits_achetes,
       SUM(montant)
   FROM ventes
   GROUP BY client_id;
   ```

2. **Utiliser des agrégations dans WHERE**
   ```sql
   -- ❌ ERREUR
   WHERE SUM(montant) > 500

   -- ✅ CORRECT
   HAVING SUM(montant) > 500
   ```

3. **Grouper sans agrégation (utiliser DISTINCT à la place)**
   ```sql
   -- ❌ Inutilement complexe
   SELECT client_id
   FROM ventes
   GROUP BY client_id;

   -- ✅ Plus clair
   SELECT DISTINCT client_id
   FROM ventes;
   ```

4. **Oublier de filtrer les NULL si pertinent**
   ```sql
   -- Peut produire un groupe NULL si la colonne contient des NULL
   SELECT categorie, COUNT(*)
   FROM produits
   GROUP BY categorie;

   -- Si vous voulez exclure les NULL :
   SELECT categorie, COUNT(*)
   FROM produits
   WHERE categorie IS NOT NULL
   GROUP BY categorie;
   ```

---

## Performance et Optimisation

### Impact de GROUP BY sur les Performances

GROUP BY nécessite :
1. **Tri ou hachage** des données pour créer les groupes
2. **Parcours complet** de la table (ou de l'index)
3. **Calculs d'agrégation** pour chaque groupe

### Optimisations

#### 1. Index sur les Colonnes de Groupement

```sql
-- Si vous groupez souvent par client_id :
CREATE INDEX idx_ventes_client_id ON ventes(client_id);

-- PostgreSQL peut utiliser cet index pour accélérer le groupement
SELECT client_id, SUM(montant)
FROM ventes
GROUP BY client_id;
```

#### 2. Filtrage Précoce avec WHERE

```sql
-- ✅ RAPIDE : WHERE filtre avant groupement
SELECT client_id, SUM(montant)
FROM ventes
WHERE date_vente >= '2024-01-01'  -- Réduit le volume
GROUP BY client_id;

-- ❌ LENT : Tout est groupé puis filtré
SELECT client_id, SUM(montant), MAX(date_vente)
FROM ventes
GROUP BY client_id
HAVING MAX(date_vente) >= '2024-01-01';
```

#### 3. Index Couvrants (Covering Index)

```sql
-- Index incluant colonnes de groupement ET d'agrégation
CREATE INDEX idx_ventes_covering
ON ventes(client_id)
INCLUDE (montant);

-- Cette requête peut être résolue uniquement avec l'index (index-only scan)
SELECT client_id, SUM(montant)
FROM ventes
GROUP BY client_id;
```

#### 4. Matérialiser les Agrégations Fréquentes

Pour des requêtes exécutées régulièrement :

```sql
-- Table de résumé mise à jour quotidiennement
CREATE TABLE ventes_quotidiennes AS
SELECT
    date_vente::DATE AS jour,
    client_id,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_jour
FROM ventes
GROUP BY date_vente::DATE, client_id;

-- Créer des index
CREATE INDEX idx_ventes_quotidiennes_jour ON ventes_quotidiennes(jour);
CREATE INDEX idx_ventes_quotidiennes_client ON ventes_quotidiennes(client_id);

-- Requêtes très rapides sur la table précalculée
SELECT client_id, SUM(ca_jour)
FROM ventes_quotidiennes
WHERE jour >= '2024-01-01'
GROUP BY client_id;
```

---

## Comprendre EXPLAIN avec GROUP BY

Analysons le plan d'exécution d'une requête avec GROUP BY :

```sql
EXPLAIN ANALYZE
SELECT
    client_id,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS total
FROM ventes
GROUP BY client_id;
```

**Résultat possible :**

```
HashAggregate  (cost=15.50..17.50 rows=100 width=20)
  Group Key: client_id
  ->  Seq Scan on ventes  (cost=0.00..10.50 rows=1000 width=12)
```

**Interprétation :**
- **HashAggregate** : PostgreSQL utilise une table de hachage pour grouper (rapide)
- Alternative : **GroupAggregate** (nécessite tri préalable, utilisé si index disponible)
- **Seq Scan** : Scan séquentiel de la table

**Si un index existe :**

```
GroupAggregate  (cost=0.15..25.30 rows=100 width=20)
  Group Key: client_id
  ->  Index Scan using idx_client_id on ventes  (cost=0.15..20.30 rows=1000 width=12)
```

L'index permet d'éviter un tri explicite (les données sont déjà ordonnées).

---

## Combinaisons Avancées

### GROUP BY + Sous-Requête

```sql
-- Clients dont les dépenses dépassent la moyenne globale
SELECT
    v.client_id,
    SUM(v.montant) AS total_depense
FROM ventes v
GROUP BY v.client_id
HAVING SUM(v.montant) > (
    SELECT AVG(total_client)
    FROM (
        SELECT SUM(montant) AS total_client
        FROM ventes
        GROUP BY client_id
    ) AS moyennes
);
```

### GROUP BY + JOIN

```sql
-- Ventes totales par client avec leurs informations personnelles
SELECT
    c.nom,
    c.prenom,
    c.email,
    COUNT(v.id_vente) AS nb_achats,
    COALESCE(SUM(v.montant), 0) AS total_depense
FROM clients c
LEFT JOIN ventes v ON c.client_id = v.client_id
GROUP BY c.client_id, c.nom, c.prenom, c.email
ORDER BY total_depense DESC;
```

**Note** : Toutes les colonnes de `clients` sélectionnées doivent être dans GROUP BY !

### GROUP BY + CTE (Common Table Expression)

```sql
-- Analyse en deux étapes : agrégation puis filtrage
WITH stats_clients AS (
    SELECT
        client_id,
        COUNT(*) AS nb_achats,
        SUM(montant) AS total_depense,
        AVG(montant) AS panier_moyen
    FROM ventes
    GROUP BY client_id
)
SELECT *
FROM stats_clients
WHERE nb_achats >= 5
  AND panier_moyen > 200
ORDER BY total_depense DESC;
```

---

## Tableau Récapitulatif : WHERE vs HAVING

| Aspect | WHERE | HAVING |
|--------|-------|--------|
| **Filtre** | Lignes individuelles | Groupes agrégés |
| **Moment** | Avant GROUP BY | Après GROUP BY |
| **Peut contenir agrégations** | ❌ Non (SUM, COUNT, etc.) | ✅ Oui |
| **Peut contenir colonnes simples** | ✅ Oui | ⚠️ Seulement si dans GROUP BY |
| **Performance** | ⚡ Meilleure (filtre tôt) | 🐢 Moins bonne (filtre tard) |
| **Usage typique** | Filtrer les données sources | Filtrer les résultats agrégés |
| **Exemple** | `WHERE date > '2024-01-01'` | `HAVING SUM(montant) > 1000` |

**Règle d'Or** : Si vous pouvez filtrer avec WHERE, faites-le ! N'utilisez HAVING que pour les conditions sur les agrégations.

---

## À Retenir

1. **GROUP BY divise les données en groupes** et calcule des agrégations par groupe
2. **Une ligne de résultat par groupe** (ou combinaison de groupes si multi-colonnes)
3. **Seules les colonnes de GROUP BY et les agrégations** peuvent apparaître dans SELECT
4. **WHERE filtre les lignes AVANT groupement**, HAVING filtre les groupes APRÈS
5. **Ordre d'exécution** : FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
6. **Performance** : Toujours filtrer avec WHERE quand possible, utiliser des index
7. **Bonnes pratiques** : Inclure COUNT(*), utiliser HAVING pour écarter petits échantillons
8. **Expressions calculées** : On peut grouper par DATE_TRUNC, EXTRACT, CASE, etc.

---

## Prochaines Étapes

Vous maîtrisez maintenant GROUP BY et HAVING ! Prochains sujets :

- **Section 8.4** : Extensions de groupement (ROLLUP, CUBE, GROUPING SETS) - Pour des agrégations multi-niveaux
- **Section 8.5** : Filtres d'agrégation (FILTER clause) - Agrégations conditionnelles avancées
- **Section 10** : Window Functions - Agréger SANS perdre les détails des lignes !

GROUP BY est un **pilier fondamental** de l'analyse de données en SQL. Avec HAVING, vous contrôlez précisément quels groupes apparaissent dans vos résultats. Ces outils sont essentiels pour tout travail analytique !

⏭️ [Extensions de groupement (ROLLUP, CUBE, GROUPING SETS)](/08-agregation-et-groupement/04-extensions-de-groupement.md)
