🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.1. Normalisation (1NF à 3NF/BCNF) vs Dénormalisation Stratégique

## Introduction

La **normalisation** est un processus méthodique de conception de bases de données relationnelles qui vise à structurer les données de manière optimale. Ce processus, formalisé par Edgar F. Codd dans les années 1970, permet d'éliminer les redondances et les anomalies qui peuvent survenir lors de la manipulation des données.

### Pourquoi la normalisation est-elle importante ?

Imaginez une entreprise qui stocke toutes ses informations dans une seule grande table. Cette approche peut sembler simple au départ, mais elle pose rapidement plusieurs problèmes :

1. **Redondance des données** : Les mêmes informations sont répétées inutilement
2. **Anomalies d'insertion** : Impossibilité d'ajouter certaines données sans en ajouter d'autres
3. **Anomalies de mise à jour** : Risque d'incohérences lors de modifications
4. **Anomalies de suppression** : Perte d'informations lors de suppressions
5. **Gaspillage d'espace** : Stockage inefficace des données

La normalisation résout ces problèmes en décomposant les tables de manière logique et systématique.

---

## Les Formes Normales : Un Processus Progressif

La normalisation s'effectue par étapes successives, chaque "forme normale" (NF) apportant un niveau supplémentaire de rigueur. Chaque forme normale inclut les contraintes des formes précédentes.

```
Données brutes
    ↓
1ère Forme Normale (1NF)
    ↓
2ème Forme Normale (2NF)
    ↓
3ème Forme Normale (3NF)
    ↓
Forme Normale de Boyce-Codd (BCNF)
```

---

## 1. Première Forme Normale (1NF)

### Définition

Une table est en **1NF** si et seulement si :
- Chaque colonne contient des **valeurs atomiques** (indivisibles)
- Chaque colonne contient des valeurs d'un **seul type**
- Chaque ligne est **unique** (identifiable par une clé primaire)
- L'ordre des lignes et des colonnes n'a **pas d'importance**

### Principe : L'atomicité des données

Une valeur est atomique quand elle ne peut pas être décomposée davantage dans le contexte de votre modèle. Par exemple, un numéro de téléphone peut être considéré comme atomique, même s'il contient plusieurs chiffres.

### Exemple : Table NON conforme à 1NF

```
Table: Commandes_Client
+-------------+------------------+---------------------------+
| client_id   | nom_client       | telephones                |
+-------------+------------------+---------------------------+
| 1           | Alice Dupont     | 0601020304, 0143567890    |
| 2           | Bob Martin       | 0678901234                |
| 3           | Claire Dubois    | 0612345678, 0698765432    |
+-------------+------------------+---------------------------+
```

**Problèmes identifiés :**
- La colonne `telephones` contient **plusieurs valeurs** séparées par des virgules
- Impossible de rechercher efficacement un numéro spécifique
- Impossible de savoir combien de téléphones possède chaque client sans parser la chaîne
- Difficulté pour ajouter, modifier ou supprimer un téléphone individuel

### Solution : Transformation en 1NF

**Option 1 : Créer une table séparée (recommandé)**

```sql
-- Table principale
Table: Clients
+-------------+------------------+
| client_id   | nom_client       |
+-------------+------------------+
| 1           | Alice Dupont     |
| 2           | Bob Martin       |
| 3           | Claire Dubois    |
+-------------+------------------+

-- Table de relation
Table: Telephones_Client
+-------------+-------------+----------------+
| telephone_id| client_id   | numero         |
+-------------+-------------+----------------+
| 1           | 1           | 0601020304     |
| 2           | 1           | 0143567890     |
| 3           | 2           | 0678901234     |
| 4           | 3           | 0612345678     |
| 5           | 3           | 0698765432     |
+-------------+-------------+----------------+
```

**Option 2 : Colonnes multiples (si nombre fixe et limité)**

```
Table: Clients
+-------------+------------------+-------------+-------------+
| client_id   | nom_client       | telephone_1 | telephone_2 |
+-------------+------------------+-------------+-------------+
| 1           | Alice Dupont     | 0601020304  | 0143567890  |
| 2           | Bob Martin       | 0678901234  | NULL        |
| 3           | Claire Dubois    | 0612345678  | 0698765432  |
+-------------+------------------+-------------+-------------+
```

⚠️ Cette seconde option est **déconseillée** car elle limite le nombre de téléphones et crée des colonnes souvent vides (NULL).

### Avantages de la 1NF

- ✅ Recherches et filtres efficaces sur chaque valeur
- ✅ Facilité d'ajout, modification et suppression
- ✅ Possibilité d'indexation sur des valeurs individuelles
- ✅ Intégrité des données garantie

---

## 2. Deuxième Forme Normale (2NF)

### Définition

Une table est en **2NF** si et seulement si :
- Elle est déjà en **1NF**
- Tous les attributs non-clés dépendent de **la totalité de la clé primaire**, et non d'une partie seulement

### Concept : Éliminer les dépendances partielles

La 2NF concerne spécifiquement les tables avec des **clés primaires composites** (constituées de plusieurs colonnes). Si un attribut ne dépend que d'une partie de la clé primaire, il doit être extrait dans une table séparée.

### Exemple : Table NON conforme à 2NF

```
Table: Commandes_Details
+-------------+--------------+------------------+-------------+-----------------+
| commande_id | produit_id   | nom_produit      | quantite    | prix_unitaire   |
+-------------+--------------+------------------+-------------+-----------------+
| 1001        | P001         | Ordinateur       | 2           | 899.00          |
| 1001        | P002         | Souris           | 3           | 15.00           |
| 1002        | P001         | Ordinateur       | 1           | 899.00          |
| 1003        | P003         | Clavier          | 2           | 45.00           |
+-------------+--------------+------------------+-------------+-----------------+

Clé primaire composite: (commande_id, produit_id)
```

**Problèmes identifiés :**
- `nom_produit` et `prix_unitaire` dépendent **uniquement de `produit_id`**, pas de `commande_id`
- Le nom et le prix du produit "Ordinateur" sont répétés à chaque commande
- Si le prix d'un produit change, il faut mettre à jour toutes les lignes (risque d'incohérence)
- Impossible de créer un produit sans qu'il soit dans une commande

### Solution : Transformation en 2NF

```sql
-- Table des produits (informations dépendant uniquement du produit)
Table: Produits
+--------------+------------------+-----------------+
| produit_id   | nom_produit      | prix_unitaire   |
+--------------+------------------+-----------------+
| P001         | Ordinateur       | 899.00          |
| P002         | Souris           | 15.00           |
| P003         | Clavier          | 45.00           |
+--------------+------------------+-----------------+

-- Table des détails de commande (uniquement les données de la relation)
Table: Commandes_Details
+-------------+--------------+-------------+
| commande_id | produit_id   | quantite    |
+-------------+--------------+-------------+
| 1001        | P001         | 2           |
| 1001        | P002         | 3           |
| 1002        | P001         | 1           |
| 1003        | P003         | 2           |
+-------------+--------------+-------------+
```

### Avantages de la 2NF

- ✅ Élimination de la redondance des données produits
- ✅ Modification du prix en un seul endroit
- ✅ Possibilité de gérer les produits indépendamment des commandes
- ✅ Cohérence garantie des informations produits

---

## 3. Troisième Forme Normale (3NF)

### Définition

Une table est en **3NF** si et seulement si :
- Elle est déjà en **2NF**
- Aucun attribut non-clé ne dépend **transitivement** de la clé primaire

### Concept : Éliminer les dépendances transitives

Une dépendance transitive existe quand un attribut dépend de la clé primaire **à travers** un autre attribut non-clé.

**Schéma de dépendance transitive :**
```
Clé Primaire → Attribut Non-Clé A → Attribut Non-Clé B
```

L'attribut B dépend de la clé primaire de manière **indirecte** (transitive).

### Exemple : Table NON conforme à 3NF

```
Table: Employes
+-------------+-----------------+------------------+--------------------+
| employe_id  | nom_employe     | departement_id   | nom_departement    |
+-------------+-----------------+------------------+--------------------+
| E001        | Alice Dupont    | D10              | Informatique       |
| E002        | Bob Martin      | D20              | Comptabilité       |
| E003        | Claire Dubois   | D10              | Informatique       |
| E004        | David Lefebvre  | D30              | Ressources Humaines|
| E005        | Emma Rousseau   | D20              | Comptabilité       |
+-------------+-----------------+------------------+--------------------+

Clé primaire: employe_id
```

**Analyse des dépendances :**
```
employe_id → nom_employe       (OK, dépendance directe)
employe_id → departement_id    (OK, dépendance directe)
employe_id → nom_departement   (PROBLÈME : dépendance transitive)

Car en réalité :
departement_id → nom_departement
```

**Problèmes identifiés :**
- Le nom du département "Informatique" est répété pour chaque employé de ce département
- Modification du nom d'un département nécessite de mettre à jour plusieurs lignes
- Impossibilité de créer un département sans employé
- Suppression du dernier employé d'un département entraîne la perte du nom du département

### Solution : Transformation en 3NF

```sql
-- Table des départements
Table: Departements
+------------------+---------------------+
| departement_id   | nom_departement     |
+------------------+---------------------+
| D10              | Informatique        |
| D20              | Comptabilité        |
| D30              | Ressources Humaines |
+------------------+---------------------+

-- Table des employés (sans redondance)
Table: Employes
+-------------+-----------------+------------------+
| employe_id  | nom_employe     | departement_id   |
+-------------+-----------------+------------------+
| E001        | Alice Dupont    | D10              |
| E002        | Bob Martin      | D20              |
| E003        | Claire Dubois   | D10              |
| E004        | David Lefebvre  | D30              |
| E005        | Emma Rousseau   | D20              |
+-------------+-----------------+------------------+
```

### Avantages de la 3NF

- ✅ Chaque information n'existe qu'à un seul endroit
- ✅ Modifications simples et sans risque d'incohérence
- ✅ Gestion indépendante des départements
- ✅ Structure logique et maintenable

---

## 4. Forme Normale de Boyce-Codd (BCNF)

### Définition

Une table est en **BCNF** si et seulement si :
- Elle est déjà en **3NF**
- Pour chaque dépendance fonctionnelle X → Y, X est une **superclé**

### Concept : La version stricte de la 3NF

La BCNF est une version plus rigoureuse de la 3NF. Elle élimine certaines anomalies que la 3NF peut encore contenir dans des cas particuliers où plusieurs clés candidates se chevauchent.

**Note importante :** Dans la plupart des cas pratiques, une table en 3NF est également en BCNF. Les violations de BCNF (tout en respectant 3NF) sont rares et surviennent dans des situations spécifiques.

### Exemple : Table en 3NF mais NON en BCNF

Considérons une table de gestion de cours universitaires :

```
Table: Enseignements
+-------------+--------------+------------------+
| etudiant_id | cours        | professeur       |
+-------------+--------------+------------------+
| 1001        | Mathématiques| Dr. Durand       |
| 1001        | Physique     | Dr. Martin       |
| 1002        | Mathématiques| Dr. Durand       |
| 1003        | Physique     | Dr. Martin       |
| 1003        | Chimie       | Dr. Petit        |
+-------------+--------------+------------------+

Contraintes métier :
- Un étudiant peut suivre plusieurs cours
- Un cours peut avoir plusieurs étudiants
- Un professeur enseigne UN SEUL cours spécifique
- Un cours peut être enseigné par plusieurs professeurs

Clé primaire composite: (etudiant_id, cours)
```

**Analyse des dépendances :**
```
(etudiant_id, cours) → professeur     (OK pour 3NF)
professeur → cours                     (PROBLÈME pour BCNF)
```

Le problème : `professeur` détermine `cours`, mais `professeur` n'est pas une superclé.

**Problèmes identifiés :**
- Si Dr. Durand enseigne toujours "Mathématiques", cette information est répétée
- Redondance : le lien professeur-cours est dupliqué
- Anomalie de mise à jour : si Dr. Durand change de matière, il faut modifier plusieurs lignes

### Solution : Transformation en BCNF

```sql
-- Table de l'association professeur-cours
Table: Professeurs_Cours
+-----------------+------------------+
| professeur      | cours            |
+-----------------+------------------+
| Dr. Durand      | Mathématiques    |
| Dr. Martin      | Physique         |
| Dr. Petit       | Chimie           |
+-----------------+------------------+

-- Table des inscriptions étudiants
Table: Inscriptions_Etudiants
+-------------+-----------------+
| etudiant_id | professeur      |
+-------------+-----------------+
| 1001        | Dr. Durand      |
| 1001        | Dr. Martin      |
| 1002        | Dr. Durand      |
| 1003        | Dr. Martin      |
| 1003        | Dr. Petit       |
+-------------+-----------------+
```

Maintenant, toutes les dépendances fonctionnelles respectent la règle de BCNF.

### Quand s'arrêter ?

Dans la **grande majorité des cas pratiques**, la **3NF est suffisante**. La BCNF est rarement nécessaire car :
- Les situations nécessitant BCNF sont peu fréquentes
- La 3NF offre déjà un excellent niveau de qualité
- Le passage en BCNF peut parfois compliquer les requêtes

**Règle pratique :** Visez la 3NF dans vos projets, et ne vous préoccupez de BCNF que si vous identifiez un problème concret.

---

## Récapitulatif Visuel des Formes Normales

```
┌─────────────────────────────────────────────────────────────────┐
│ 1NF : Valeurs atomiques, pas de répétition de groupes           │
│                                                                 │
│ ❌ telephones: "0601, 0602"                                     │
│ ✅ Une ligne par téléphone                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2NF : Pas de dépendance partielle (clés composites)             │
│                                                                 │
│ ❌ (commande_id, produit_id) → nom_produit                      │
│    nom_produit ne dépend que de produit_id                      │
│ ✅ Séparer les tables Produits et Commandes_Details             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3NF : Pas de dépendance transitive                              │
│                                                                 │
│ ❌ employe_id → departement_id → nom_departement                │
│ ✅ Séparer les tables Employes et Departements                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ BCNF : Toute dépendance provient d'une superclé                 │
│                                                                 │
│ Cas rare et spécifique                                          │
│ En pratique, 3NF suffit souvent                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## La Dénormalisation Stratégique

### Pourquoi dénormaliser ?

Après avoir appris à normaliser, il peut sembler contre-intuitif de **volontairement** introduire de la redondance. Pourtant, la dénormalisation est une technique essentielle en production pour :

1. **Améliorer les performances** : Réduire le nombre de jointures coûteuses
2. **Simplifier les requêtes** : Rendre le code plus lisible et maintenable
3. **Optimiser les cas d'usage spécifiques** : Adapter la structure aux besoins réels

### Principe fondamental

> La normalisation est la règle, la dénormalisation est l'exception justifiée.

La dénormalisation doit être :
- **Intentionnelle** : Résultat d'une décision consciente, pas de paresse
- **Documentée** : Les raisons doivent être expliquées
- **Mesurée** : Les gains doivent être quantifiés
- **Contrôlée** : Les risques d'incohérence doivent être gérés

### Quand dénormaliser ?

#### 1. Requêtes très fréquentes avec nombreuses jointures

**Exemple :** Affichage d'une liste de commandes

```sql
-- Schéma normalisé (3NF)
SELECT
    c.commande_id,
    cl.nom_client,
    cl.email,
    p.nom_produit,
    cd.quantite,
    p.prix_unitaire
FROM commandes c
JOIN clients cl ON c.client_id = cl.client_id
JOIN commandes_details cd ON c.commande_id = cd.commande_id
JOIN produits p ON cd.produit_id = p.produit_id
WHERE c.date_commande > '2025-01-01';
```

Cette requête nécessite **3 jointures** à chaque exécution.

**Solution dénormalisée :**

```sql
-- Ajouter des colonnes redondantes dans commandes_details
Table: Commandes_Details_Denorm
+-------------+--------------+------------------+------------+---------------+
| commande_id | produit_id   | nom_produit      | quantite   | prix_unitaire |
+-------------+--------------+------------------+------------+---------------+

-- Requête simplifiée (0 jointure avec produits)
SELECT
    cd.commande_id,
    cd.nom_produit,
    cd.quantite,
    cd.prix_unitaire
FROM commandes_details_denorm cd
JOIN commandes c ON cd.commande_id = c.commande_id
WHERE c.date_commande > '2025-01-01';
```

**Trade-off :**
- ✅ Gain : 1 jointure en moins, requête plus rapide
- ❌ Coût : Duplication du nom et du prix du produit

#### 2. Agrégations répétitives

**Exemple :** Nombre de commandes par client

```sql
-- Calcul à chaque fois (coûteux)
SELECT
    cl.client_id,
    cl.nom_client,
    COUNT(c.commande_id) as nb_commandes
FROM clients cl
LEFT JOIN commandes c ON cl.client_id = c.client_id
GROUP BY cl.client_id, cl.nom_client;
```

**Solution dénormalisée :**

```sql
-- Ajouter une colonne calculée dans clients
ALTER TABLE clients ADD COLUMN nb_commandes INTEGER DEFAULT 0;

-- Maintenir via trigger ou application
Table: Clients
+-------------+------------------+---------------+
| client_id   | nom_client       | nb_commandes  |
+-------------+------------------+---------------+
| 1           | Alice Dupont     | 15            |
| 2           | Bob Martin       | 3             |
+-------------+------------------+---------------+

-- Requête instantanée
SELECT client_id, nom_client, nb_commandes
FROM clients
WHERE nb_commandes > 10;
```

**Trade-off :**
- ✅ Gain : Requête ultra-rapide, pas d'agrégation
- ❌ Coût : Nécessite de maintenir la cohérence via trigger/code applicatif

#### 3. Données historiques immuables

**Exemple :** Prix des produits au moment de la commande

Les prix changent dans le temps, mais une commande doit toujours refléter le prix au moment de l'achat.

```sql
-- ❌ ERREUR : Ne pas faire ça
Table: Commandes_Details
+-------------+--------------+-------------+
| commande_id | produit_id   | quantite    |
+-------------+--------------+-------------+

-- Requête pour obtenir le prix (INCORRECT pour historique)
SELECT cd.*, p.prix_unitaire
FROM commandes_details cd
JOIN produits p ON cd.produit_id = p.produit_id;
-- Le prix actuel n'est pas forcément celui de la commande !
```

**Solution dénormalisée (nécessaire) :**

```sql
-- ✅ Capturer le prix au moment de la commande
Table: Commandes_Details
+-------------+--------------+-------------+------------------+
| commande_id | produit_id   | quantite    | prix_historique  |
+-------------+--------------+-------------+------------------+
| 1001        | P001         | 2           | 899.00           |
| 1002        | P001         | 1           | 899.00           |
| 1003        | P001         | 1           | 950.00           |  -- Prix a augmenté
+-------------+--------------+-------------+------------------+
```

Ce n'est pas vraiment une dénormalisation, mais une **donnée métier essentielle**.

### Techniques de dénormalisation courantes

#### 1. Colonnes calculées (Computed Columns)

PostgreSQL 18 supporte les **colonnes virtuelles générées** :

```sql
CREATE TABLE produits (
    produit_id SERIAL PRIMARY KEY,
    prix_ht NUMERIC(10,2),
    tva_taux NUMERIC(5,2),
    prix_ttc NUMERIC(10,2) GENERATED ALWAYS AS (prix_ht * (1 + tva_taux / 100)) VIRTUAL
);

-- Pas de stockage physique, calcul à la volée
-- Maintien automatique de la cohérence
```

#### 2. Vues matérialisées

Pour des agrégations complexes utilisées fréquemment :

```sql
CREATE MATERIALIZED VIEW stats_ventes AS
SELECT
    DATE_TRUNC('month', c.date_commande) as mois,
    p.categorie,
    SUM(cd.quantite * cd.prix_unitaire) as chiffre_affaires,
    COUNT(DISTINCT c.commande_id) as nb_commandes
FROM commandes c
JOIN commandes_details cd ON c.commande_id = cd.commande_id
JOIN produits p ON cd.produit_id = p.produit_id
GROUP BY 1, 2;

-- Rafraîchissement périodique
REFRESH MATERIALIZED VIEW stats_ventes;
```

#### 3. Tables de cache/snapshot

Copie dénormalisée mise à jour périodiquement :

```sql
-- Table optimisée pour l'affichage
CREATE TABLE commandes_display_cache AS
SELECT
    c.commande_id,
    c.date_commande,
    cl.nom_client,
    cl.email,
    SUM(cd.quantite * cd.prix_unitaire) as montant_total
FROM commandes c
JOIN clients cl ON c.client_id = cl.client_id
JOIN commandes_details cd ON c.commande_id = cd.commande_id
GROUP BY c.commande_id, c.date_commande, cl.nom_client, cl.email;

-- Rafraîchir chaque nuit via cron
```

### Maintenir la cohérence en dénormalisation

La dénormalisation introduit un risque d'**incohérence**. Voici les stratégies pour le gérer :

#### 1. Triggers automatiques

```sql
-- Maintenir automatiquement nb_commandes dans clients
CREATE OR REPLACE FUNCTION update_client_nb_commandes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE clients
        SET nb_commandes = nb_commandes + 1
        WHERE client_id = NEW.client_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE clients
        SET nb_commandes = nb_commandes - 1
        WHERE client_id = OLD.client_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_nb_commandes
AFTER INSERT OR DELETE ON commandes
FOR EACH ROW EXECUTE FUNCTION update_client_nb_commandes();
```

#### 2. Logique applicative

```python
# Exemple Python avec psycopg3
def create_commande(client_id, produits):
    with conn.transaction():
        # Insérer la commande
        cursor.execute(
            "INSERT INTO commandes (client_id, date_commande) VALUES (%s, NOW()) RETURNING commande_id",
            (client_id,)
        )
        commande_id = cursor.fetchone()[0]

        # Insérer les détails avec prix capturés
        for produit in produits:
            cursor.execute(
                """INSERT INTO commandes_details
                   (commande_id, produit_id, quantite, prix_historique)
                   SELECT %s, %s, %s, prix_unitaire
                   FROM produits WHERE produit_id = %s""",
                (commande_id, produit['id'], produit['quantite'], produit['id'])
            )

        # Mettre à jour le compteur dénormalisé
        cursor.execute(
            "UPDATE clients SET nb_commandes = nb_commandes + 1 WHERE client_id = %s",
            (client_id,)
        )
```

#### 3. Validation périodique

```sql
-- Script de vérification de cohérence
SELECT
    c.client_id,
    c.nom_client,
    c.nb_commandes as compteur_stocke,
    COUNT(co.commande_id) as compteur_reel,
    c.nb_commandes - COUNT(co.commande_id) as ecart
FROM clients c
LEFT JOIN commandes co ON c.client_id = co.client_id
GROUP BY c.client_id, c.nom_client, c.nb_commandes
HAVING c.nb_commandes != COUNT(co.commande_id);
```

### Règles de décision : Normaliser ou Dénormaliser ?

```
┌─────────────────────────────────────────────────────────────────┐
│                   Arbre de décision                             │
└─────────────────────────────────────────────────────────────────┘

Question 1 : Est-ce un nouveau projet ?
   ├─ Oui → Commencer par normaliser (3NF)
   └─ Non → Continuer

Question 2 : Y a-t-il un problème de performance mesuré ?
   ├─ Non → Ne PAS dénormaliser (optimisation prématurée)
   └─ Oui → Continuer

Question 3 : Les requêtes concernées sont-elles très fréquentes ?
   ├─ Non → Optimiser autrement (index, EXPLAIN, cache applicatif)
   └─ Oui → Continuer

Question 4 : Les jointures sont-elles le goulot d'étranglement ?
   ├─ Non → Chercher ailleurs (index, requête, hardware)
   └─ Oui → Continuer

Question 5 : Peut-on maintenir la cohérence facilement ?
   ├─ Non → Envisager d'autres solutions (vues matérialisées, cache)
   └─ Oui → Dénormaliser est justifié

```

### Exemples de dénormalisation en production

#### E-commerce : Table produits

```sql
-- Normalisé (3NF)
Table: Produits
+-------------+------------------+-------------+
| produit_id  | nom_produit      | categorie_id|
+-------------+------------------+-------------+

Table: Categories
+-------------+------------------+
| categorie_id| nom_categorie    |
+-------------+------------------+

-- Dénormalisé (ajout nom_categorie redondant)
Table: Produits_Denorm
+-------------+------------------+-------------+------------------+
| produit_id  | nom_produit      | categorie_id| nom_categorie    |
+-------------+------------------+-------------+------------------+

-- Justification :
-- La catégorie est affichée sur CHAQUE page produit
-- Éviter une jointure sur 100% des requêtes produits
-- Les noms de catégories changent rarement
```

#### Analytics : Tableaux de bord

```sql
-- Au lieu de recalculer à chaque affichage
CREATE MATERIALIZED VIEW dashboard_ventes_journalieres AS
SELECT
    DATE(c.date_commande) as jour,
    COUNT(DISTINCT c.commande_id) as nb_commandes,
    COUNT(DISTINCT c.client_id) as nb_clients,
    SUM(cd.quantite * cd.prix_unitaire) as ca_jour,
    AVG(cd.quantite * cd.prix_unitaire) as panier_moyen
FROM commandes c
JOIN commandes_details cd ON c.commande_id = cd.commande_id
GROUP BY DATE(c.date_commande);

-- Rafraîchir chaque nuit à 1h du matin
-- Dashboard = requête SELECT simple et ultra-rapide
```

---

## Bonnes Pratiques et Conseils

### Pour la normalisation

1. **Commencez toujours normalisé** : Plus facile de dénormaliser que l'inverse
2. **Documentez vos choix** : Expliquez pourquoi vous vous arrêtez à 2NF ou 3NF
3. **Utilisez des outils de modélisation** : ERD (Entity-Relationship Diagram)
4. **Testez avec des données réalistes** : Simulez le volume de production
5. **Révisez régulièrement** : Le modèle évolue avec l'application

### Pour la dénormalisation

1. **Mesurez avant d'agir** : `EXPLAIN ANALYZE` est votre ami
2. **Dénormalisez progressivement** : Une colonne à la fois
3. **Automatisez la cohérence** : Triggers ou logique applicative robuste
4. **Monitorer les écarts** : Scripts de validation réguliers
5. **Documentez clairement** : Indiquez les données redondantes et leur source

### Erreurs courantes à éviter

- ❌ **Dénormaliser par paresse** : "C'est plus simple d'avoir tout dans une table"
- ✅ Normaliser d'abord, dénormaliser si besoin justifié

- ❌ **Optimisation prématurée** : Dénormaliser avant d'avoir mesuré
- ✅ Profiler, identifier les goulots, puis optimiser

- ❌ **Ignorer les coûts de maintenance** : Oublier la synchronisation
- ✅ Évaluer le coût total (développement + maintenance)

- ❌ **Dénormaliser sans stratégie de cohérence** : Données incohérentes
- ✅ Mettre en place triggers/validation dès le départ

---

## Cas d'Étude : Système de Blog

### Modèle Initial (Normalisé - 3NF)

```sql
-- Articles
CREATE TABLE articles (
    article_id SERIAL PRIMARY KEY,
    titre VARCHAR(200) NOT NULL,
    contenu TEXT,
    auteur_id INTEGER REFERENCES auteurs(auteur_id),
    date_publication TIMESTAMPTZ
);

-- Auteurs
CREATE TABLE auteurs (
    auteur_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(100)
);

-- Commentaires
CREATE TABLE commentaires (
    commentaire_id SERIAL PRIMARY KEY,
    article_id INTEGER REFERENCES articles(article_id),
    auteur_commentaire VARCHAR(100),
    texte TEXT,
    date_commentaire TIMESTAMPTZ
);

-- Tags
CREATE TABLE tags (
    tag_id SERIAL PRIMARY KEY,
    nom_tag VARCHAR(50)
);

-- Association articles-tags
CREATE TABLE articles_tags (
    article_id INTEGER REFERENCES articles(article_id),
    tag_id INTEGER REFERENCES tags(tag_id),
    PRIMARY KEY (article_id, tag_id)
);
```

### Problème de Performance Identifié

Requête d'affichage de la page d'accueil (exécutée 10 000 fois par jour) :

```sql
SELECT
    a.article_id,
    a.titre,
    au.nom as nom_auteur,
    COUNT(DISTINCT c.commentaire_id) as nb_commentaires,
    STRING_AGG(DISTINCT t.nom_tag, ', ') as tags
FROM articles a
JOIN auteurs au ON a.auteur_id = au.auteur_id
LEFT JOIN commentaires c ON a.article_id = c.article_id
LEFT JOIN articles_tags at ON a.article_id = at.article_id
LEFT JOIN tags t ON at.tag_id = t.tag_id
WHERE a.date_publication <= NOW()
GROUP BY a.article_id, a.titre, au.nom
ORDER BY a.date_publication DESC
LIMIT 20;
```

**Analyse :** 4 jointures, 2 agrégations, exécuté des milliers de fois.

### Solution Dénormalisée

```sql
-- Ajouter des colonnes dénormalisées
ALTER TABLE articles ADD COLUMN nom_auteur VARCHAR(100);
ALTER TABLE articles ADD COLUMN nb_commentaires INTEGER DEFAULT 0;
ALTER TABLE articles ADD COLUMN tags_list TEXT;

-- Trigger pour maintenir nom_auteur
CREATE OR REPLACE FUNCTION sync_nom_auteur()
RETURNS TRIGGER AS $$
BEGIN
    NEW.nom_auteur := (SELECT nom FROM auteurs WHERE auteur_id = NEW.auteur_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_auteur
BEFORE INSERT OR UPDATE OF auteur_id ON articles
FOR EACH ROW EXECUTE FUNCTION sync_nom_auteur();

-- Trigger pour nb_commentaires
CREATE OR REPLACE FUNCTION update_nb_commentaires()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE articles SET nb_commentaires = nb_commentaires + 1
        WHERE article_id = NEW.article_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE articles SET nb_commentaires = nb_commentaires - 1
        WHERE article_id = OLD.article_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_nb_commentaires
AFTER INSERT OR DELETE ON commentaires
FOR EACH ROW EXECUTE FUNCTION update_nb_commentaires();

-- Nouvelle requête ultra-rapide
SELECT
    article_id,
    titre,
    nom_auteur,
    nb_commentaires,
    tags_list
FROM articles
WHERE date_publication <= NOW()
ORDER BY date_publication DESC
LIMIT 20;
```

**Résultat :**
- ✅ Requête 10× plus rapide (0 jointure, 0 agrégation)
- ✅ Code applicatif simplifié
- ⚠️ Complexité déplacée vers les triggers de synchronisation
- ⚠️ Nécessite tests rigoureux de la cohérence

---

## Conclusion

La normalisation et la dénormalisation sont deux faces d'une même médaille : **la conception optimale de bases de données**.

### Principes à retenir

1. **Normaliser est la base** : Commencez toujours par une structure normalisée (3NF)
2. **Dénormaliser avec intention** : Seulement pour résoudre des problèmes mesurés
3. **Équilibrer** : Performance vs Cohérence vs Maintenabilité
4. **Mesurer** : Utilisez `EXPLAIN ANALYZE`, `pg_stat_statements`
5. **Documenter** : Expliquez vos choix de conception

### Philosophie de conception

```
┌──────────────────────────────────────────────────────────────┐
│  "Normalize until it hurts, denormalize until it works"      │
│                                                              │
│  "Make it work, make it right, make it fast"                 │
└──────────────────────────────────────────────────────────────┘
```

### Checklist de décision

Avant de dénormaliser, posez-vous ces questions :

- [ ] Ai-je mesuré le problème de performance avec `EXPLAIN ANALYZE` ?
- [ ] Est-ce réellement la jointure qui est le goulot ?
- [ ] Ai-je exploré toutes les alternatives (index, cache, query optimization) ?
- [ ] Puis-je maintenir la cohérence facilement et automatiquement ?
- [ ] Le gain de performance justifie-t-il la complexité ajoutée ?
- [ ] Ai-je documenté ma décision et les raisons ?
- [ ] Ai-je mis en place un mécanisme de validation de la cohérence ?

Si vous répondez "oui" à toutes ces questions, la dénormalisation est probablement justifiée.

---

## Pour Aller Plus Loin

### Ressources recommandées

- **Documentation PostgreSQL** : [Normalisation et conception de schémas](https://www.postgresql.org/docs/)
- **Livre** : "Database Design and Relational Theory" par C.J. Date
- **Article** : "Normalization and Denormalization" sur PostgreSQL Wiki

### Concepts connexes à explorer

- **Partitionnement de tables** : Diviser une grande table en sous-tables
- **Vues matérialisées** : Cache de requêtes complexes
- **Index covering** : Inclure des colonnes dans l'index pour éviter les lookups
- **JSONB** : Alternative NoSQL dans PostgreSQL pour certains cas d'usage

---

**Fin du Chapitre 11.1**

Ce chapitre vous a présenté les fondements de la normalisation (1NF à BCNF) et les stratégies de dénormalisation. Dans les chapitres suivants, nous explorerons d'autres techniques avancées de modélisation et d'optimisation pour PostgreSQL.

⏭️ [Modélisation JSONB : Quand utiliser le NoSQL dans SQL](/11-modelisation-avancee/02-modelisation-jsonb.md)
