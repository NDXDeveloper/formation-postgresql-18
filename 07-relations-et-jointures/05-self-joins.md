🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.5. Self-Joins : Joindre une Table à Elle-Même

## Introduction

Un **self-join** (auto-jointure ou jointure réflexive) est une opération où **une table est jointe à elle-même**. Bien que cela puisse sembler étrange au premier abord, c'est une technique extrêmement puissante et courante pour résoudre de nombreux problèmes réels.

### Pourquoi Joindre une Table à Elle-Même ?

Les self-joins sont utilisés quand les données d'une table ont des **relations entre elles** :
- Relations **hiérarchiques** : employés et leurs managers
- **Comparaisons** : trouver des paires ou des duplications
- Relations **symétriques** : amis, collègues, partenaires
- **Séquences** : ligne précédente/suivante, évolution temporelle

### Le Principe Clé

Imaginez que vous ayez deux **copies virtuelles** de la même table. Vous pouvez alors les joindre comme s'il s'agissait de deux tables différentes. La seule différence : **vous devez utiliser des alias différents** pour distinguer les deux "copies".

---

## 1. Concept de Base : Les Alias sont Obligatoires

### Pourquoi les Alias ?

Sans alias, PostgreSQL ne peut pas distinguer de quelle "instance" de la table vous parlez.

```sql
-- ❌ ERREUR : Impossible sans alias
SELECT *  
FROM employes  
JOIN employes ON employes.manager_id = employes.id;  
-- ERROR: table name "employes" specified more than once

-- ✅ CORRECT : Avec alias
SELECT *  
FROM employes AS e1  
JOIN employes AS e2 ON e1.manager_id = e2.id;  
```

### Analogie

Pensez aux alias comme à des **surnoms temporaires** :
- `employes AS e1` → "Appelons cette version 'e1'"  
- `employes AS e2` → "Appelons cette version 'e2'"

Maintenant, vous pouvez parler de `e1.nom` et `e2.nom` distinctement.

---

## 2. Premier Exemple : Hiérarchie Employés/Managers

### Le Problème

Vous avez une table d'employés où chaque employé peut avoir un manager (qui est aussi un employé).

### Structure de la Table

```sql
CREATE TABLE employes (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    poste VARCHAR(100),
    manager_id INTEGER REFERENCES employes(id),
    salaire NUMERIC(10, 2)
);
```

**Point clé** : `manager_id` est une **clé étrangère** qui référence `id` dans **la même table**.

### Données d'Exemple

```sql
INSERT INTO employes (nom, poste, manager_id, salaire) VALUES
    ('Alice Martin', 'PDG', NULL, 150000),
    ('Bob Dupont', 'Directeur IT', 1, 100000),
    ('Charlie Durand', 'Directeur RH', 1, 95000),
    ('Diana Bernard', 'Développeur Senior', 2, 75000),
    ('Eve Laurent', 'Développeur', 2, 60000),
    ('Frank Petit', 'Recruteur', 3, 55000);
```

### Représentation Hiérarchique

```
Alice Martin (PDG)
├── Bob Dupont (Directeur IT)
│   ├── Diana Bernard (Développeur Senior)
│   └── Eve Laurent (Développeur)
└── Charlie Durand (Directeur RH)
    └── Frank Petit (Recruteur)
```

### Visualisation des Données

| id | nom             | poste              | manager_id | salaire  |
|----|-----------------|-----------------------|------------|----------|
| 1  | Alice Martin    | PDG                   | NULL       | 150000   |
| 2  | Bob Dupont      | Directeur IT          | 1          | 100000   |
| 3  | Charlie Durand  | Directeur RH          | 1          | 95000    |
| 4  | Diana Bernard   | Développeur Senior    | 2          | 75000    |
| 5  | Eve Laurent     | Développeur           | 2          | 60000    |
| 6  | Frank Petit     | Recruteur             | 3          | 55000    |

### Objectif : Afficher Chaque Employé avec Son Manager

```sql
SELECT
    e.nom AS employe,
    e.poste AS poste_employe,
    m.nom AS manager,
    m.poste AS poste_manager
FROM employes AS e  
LEFT JOIN employes AS m ON e.manager_id = m.id  
ORDER BY e.id;  
```

**Explication** :
- `employes AS e` → La table des employés (première "copie")  
- `employes AS m` → La table des managers (deuxième "copie")  
- `e.manager_id = m.id` → Lier l'employé à son manager

### Résultat

| employe          | poste_employe         | manager        | poste_manager     |
|------------------|-----------------------|----------------|-------------------|
| Alice Martin     | PDG                   | NULL           | NULL              |
| Bob Dupont       | Directeur IT          | Alice Martin   | PDG               |
| Charlie Durand   | Directeur RH          | Alice Martin   | PDG               |
| Diana Bernard    | Développeur Senior    | Bob Dupont     | Directeur IT      |
| Eve Laurent      | Développeur           | Bob Dupont     | Directeur IT      |
| Frank Petit      | Recruteur             | Charlie Durand | Directeur RH      |

**Observations** :
- Alice (PDG) n'a pas de manager → `NULL`
- Bob et Charlie ont Alice comme manager
- Diana et Eve ont Bob comme manager
- Frank a Charlie comme manager

### Pourquoi LEFT JOIN ?

Nous utilisons **LEFT JOIN** (et non INNER JOIN) pour inclure les employés sans manager (comme Alice, le PDG).

```sql
-- Avec INNER JOIN : Alice disparaît
SELECT e.nom AS employe, m.nom AS manager  
FROM employes AS e  
INNER JOIN employes AS m ON e.manager_id = m.id;  

-- Avec LEFT JOIN : Alice apparaît avec NULL
SELECT e.nom AS employe, m.nom AS manager  
FROM employes AS e  
LEFT JOIN employes AS m ON e.manager_id = m.id;  
```

---

## 3. Cas d'Usage : Hiérarchies et Organigrammes

### Exemple 1 : Lister les Employés d'un Manager Spécifique

**Objectif** : Qui travaille directement sous Bob Dupont ?

```sql
SELECT
    e.nom AS employe,
    e.poste
FROM employes AS e  
INNER JOIN employes AS m ON e.manager_id = m.id  
WHERE m.nom = 'Bob Dupont'  
ORDER BY e.nom;  
```

**Résultat** :

| employe        | poste              |
|----------------|--------------------|
| Diana Bernard  | Développeur Senior |
| Eve Laurent    | Développeur        |

### Exemple 2 : Compter les Subordonnés Directs

**Objectif** : Combien de personnes chaque manager supervise-t-il directement ?

```sql
SELECT
    m.nom AS manager,
    m.poste,
    COUNT(e.id) AS nombre_subordonnes
FROM employes AS m  
LEFT JOIN employes AS e ON m.id = e.manager_id  
GROUP BY m.id, m.nom, m.poste  
HAVING COUNT(e.id) > 0  
ORDER BY nombre_subordonnes DESC;  
```

**Résultat** :

| manager         | poste         | nombre_subordonnes |
|-----------------|---------------|--------------------|
| Bob Dupont      | Directeur IT  | 2                  |
| Alice Martin    | PDG           | 2                  |
| Charlie Durand  | Directeur RH  | 1                  |

### Exemple 3 : Employés Gagnant Plus que Leur Manager

**Objectif** : Détecter les anomalies salariales où un employé gagne plus que son manager.

```sql
SELECT
    e.nom AS employe,
    e.salaire AS salaire_employe,
    m.nom AS manager,
    m.salaire AS salaire_manager,
    (e.salaire - m.salaire) AS difference
FROM employes AS e  
INNER JOIN employes AS m ON e.manager_id = m.id  
WHERE e.salaire > m.salaire  
ORDER BY difference DESC;  
```

**Résultat** (avec nos données, aucune anomalie) :

| employe | salaire_employe | manager | salaire_manager | difference |
|---------|-----------------|---------|-----------------|------------|
| (vide)  |                 |         |                 |            |

Si nous avions une anomalie, elle apparaîtrait ici.

### Exemple 4 : Trouver les Managers Directs et Leur Manager (2 Niveaux)

**Objectif** : Afficher employé → manager → manager du manager

```sql
SELECT
    e.nom AS employe,
    m1.nom AS manager_direct,
    m2.nom AS manager_niveau_2
FROM employes AS e  
LEFT JOIN employes AS m1 ON e.manager_id = m1.id  
LEFT JOIN employes AS m2 ON m1.manager_id = m2.id  
WHERE e.manager_id IS NOT NULL  -- Exclure le PDG  
ORDER BY e.id;  
```

**Résultat** :

| employe         | manager_direct  | manager_niveau_2 |
|-----------------|-----------------|------------------|
| Bob Dupont      | Alice Martin    | NULL             |
| Charlie Durand  | Alice Martin    | NULL             |
| Diana Bernard   | Bob Dupont      | Alice Martin     |
| Eve Laurent     | Bob Dupont      | Alice Martin     |
| Frank Petit     | Charlie Durand  | Alice Martin     |

**Explication** :
- Diana a Bob comme manager, et Bob a Alice comme manager
- C'est une **double self-join** !

---

## 4. Cas d'Usage : Comparaisons entre Lignes

### Exemple 1 : Trouver les Doublons (Noms Identiques)

**Problème** : Identifier les employés ayant le même nom (potentiels doublons).

```sql
-- Ajouter un doublon pour l'exemple
INSERT INTO employes (nom, poste, manager_id, salaire)  
VALUES ('Bob Dupont', 'Consultant', 1, 85000);  

-- Trouver les doublons
SELECT
    e1.id AS id1,
    e1.nom,
    e1.poste AS poste1,
    e2.id AS id2,
    e2.poste AS poste2
FROM employes AS e1  
INNER JOIN employes AS e2 ON e1.nom = e2.nom  
WHERE e1.id < e2.id  -- Éviter les paires symétriques et l'auto-match  
ORDER BY e1.nom;  
```

**Résultat** :

| id1 | nom        | poste1       | id2 | poste2     |
|-----|------------|--------------|-----|------------|
| 2   | Bob Dupont | Directeur IT | 7   | Consultant |

**Important** : La condition `e1.id < e2.id` est cruciale :
- Évite de comparer une ligne avec elle-même (e1.id = e2.id)
- Évite les paires symétriques (2,7) et (7,2) → on garde seulement (2,7)

### Exemple 2 : Trouver les Employés avec un Salaire Similaire

**Objectif** : Identifier les employés ayant des salaires dans la même fourchette (±5000 €).

```sql
SELECT
    e1.nom AS employe1,
    e1.salaire AS salaire1,
    e2.nom AS employe2,
    e2.salaire AS salaire2,
    ABS(e1.salaire - e2.salaire) AS difference
FROM employes AS e1  
INNER JOIN employes AS e2 ON ABS(e1.salaire - e2.salaire) <= 5000  
WHERE e1.id < e2.id  -- Éviter doublons et auto-match  
ORDER BY difference;  
```

**Résultat** :

| employe1       | salaire1 | employe2        | salaire2 | difference |
|----------------|----------|-----------------|----------|------------|
| Charlie Durand | 95000    | Bob Dupont      | 100000   | 5000       |
| Diana Bernard  | 75000    | Eve Laurent     | 60000    | 15000 (❌) |

**Note** : Seulement les employés avec ≤5000 € de différence apparaissent.

### Exemple 3 : Comparer les Ventes Année par Année

**Structure de données** :

```sql
CREATE TABLE ventes_annuelles (
    annee INTEGER PRIMARY KEY,
    montant NUMERIC(12, 2)
);

INSERT INTO ventes_annuelles (annee, montant) VALUES
    (2020, 1000000),
    (2021, 1200000),
    (2022, 1350000),
    (2023, 1280000),
    (2024, 1450000);
```

**Objectif** : Calculer la variation année par année.

```sql
SELECT
    v1.annee AS annee,
    v1.montant AS montant,
    v2.annee AS annee_precedente,
    v2.montant AS montant_precedent,
    v1.montant - v2.montant AS variation,
    ROUND(((v1.montant - v2.montant) / v2.montant) * 100, 2) AS variation_pct
FROM ventes_annuelles AS v1  
INNER JOIN ventes_annuelles AS v2 ON v1.annee = v2.annee + 1  
ORDER BY v1.annee;  
```

**Résultat** :

| annee | montant  | annee_precedente | montant_precedent | variation | variation_pct |
|-------|----------|------------------|-------------------|-----------|---------------|
| 2021  | 1200000  | 2020             | 1000000           | 200000    | 20.00         |
| 2022  | 1350000  | 2021             | 1200000           | 150000    | 12.50         |
| 2023  | 1280000  | 2022             | 1350000           | -70000    | -5.19         |
| 2024  | 1450000  | 2023             | 1280000           | 170000    | 13.28         |

**Explication** : `v1.annee = v2.annee + 1` lie chaque année à l'année précédente.

---

## 5. Cas d'Usage : Relations Symétriques

### Le Problème des Relations Bidirectionnelles

Certaines relations sont **symétriques** : si A est ami avec B, alors B est ami avec A.

### Structure de Données : Amis

```sql
CREATE TABLE amities (
    id SERIAL PRIMARY KEY,
    utilisateur_id INTEGER NOT NULL,
    ami_id INTEGER NOT NULL,
    date_amitie DATE NOT NULL,
    UNIQUE(utilisateur_id, ami_id)
);

INSERT INTO amities (utilisateur_id, ami_id, date_amitie) VALUES
    (1, 2, '2024-01-15'),
    (2, 1, '2024-01-15'),  -- Relation symétrique
    (1, 3, '2024-02-10'),
    (3, 1, '2024-02-10'),
    (2, 4, '2024-03-05'),
    (4, 2, '2024-03-05');
```

**Table utilisateurs** (pour l'exemple) :

```sql
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

INSERT INTO utilisateurs (nom) VALUES
    ('Alice'), ('Bob'), ('Charlie'), ('Diana');
```

### Exemple 1 : Lister les Amis d'un Utilisateur

**Objectif** : Qui sont les amis d'Alice (id=1) ?

```sql
SELECT
    u.nom AS utilisateur,
    a.nom AS ami
FROM amities  
INNER JOIN utilisateurs AS u ON amities.utilisateur_id = u.id  
INNER JOIN utilisateurs AS a ON amities.ami_id = a.id  
WHERE amities.utilisateur_id = 1;  
```

**Résultat** :

| utilisateur | ami     |
|-------------|---------|
| Alice       | Bob     |
| Alice       | Charlie |

### Exemple 2 : Trouver les Amis Mutuels

**Objectif** : Quels utilisateurs sont amis mutuellement (relation bidirectionnelle) ?

```sql
SELECT DISTINCT
    u1.nom AS utilisateur1,
    u2.nom AS utilisateur2
FROM amities AS a1  
INNER JOIN amities AS a2 ON a1.utilisateur_id = a2.ami_id  
                         AND a1.ami_id = a2.utilisateur_id
INNER JOIN utilisateurs AS u1 ON a1.utilisateur_id = u1.id  
INNER JOIN utilisateurs AS u2 ON a1.ami_id = u2.id  
WHERE a1.utilisateur_id < a1.ami_id  -- Éviter les doublons  
ORDER BY u1.nom;  
```

**Résultat** :

| utilisateur1 | utilisateur2 |
|--------------|--------------|
| Alice        | Bob          |
| Alice        | Charlie      |
| Bob          | Diana        |

### Exemple 3 : Amis en Commun (Amis de Mes Amis)

**Objectif** : Trouver les amis en commun entre deux utilisateurs.

```sql
-- Amis en commun entre Alice (id=1) et Bob (id=2)
SELECT DISTINCT
    u.nom AS ami_commun
FROM amities AS a1  
INNER JOIN amities AS a2 ON a1.ami_id = a2.ami_id  
INNER JOIN utilisateurs AS u ON a1.ami_id = u.id  
WHERE a1.utilisateur_id = 1  -- Alice  
  AND a2.utilisateur_id = 2  -- Bob
  AND a1.ami_id NOT IN (1, 2);  -- Ne pas inclure Alice ou Bob
```

---

## 6. Cas d'Usage : Séquences et Évolution

### Exemple 1 : Ligne Précédente et Suivante

**Structure** : Historique de température

```sql
CREATE TABLE mesures_temperature (
    id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    temperature NUMERIC(4, 1) NOT NULL
);

INSERT INTO mesures_temperature (date, temperature) VALUES
    ('2025-01-01', 5.2),
    ('2025-01-02', 6.8),
    ('2025-01-03', 4.5),
    ('2025-01-04', 7.1),
    ('2025-01-05', 8.3);
```

**Objectif** : Comparer chaque jour avec le jour précédent.

```sql
SELECT
    t1.date AS date,
    t1.temperature AS temperature_actuelle,
    t2.date AS date_precedente,
    t2.temperature AS temperature_precedente,
    t1.temperature - t2.temperature AS variation
FROM mesures_temperature AS t1  
LEFT JOIN mesures_temperature AS t2 ON t1.id = t2.id + 1  
ORDER BY t1.date;  
```

**Résultat** :

| date       | temperature_actuelle | date_precedente | temperature_precedente | variation |
|------------|----------------------|-----------------|------------------------|-----------|
| 2025-01-01 | 5.2                  | NULL            | NULL                   | NULL      |
| 2025-01-02 | 6.8                  | 2025-01-01      | 5.2                    | 1.6       |
| 2025-01-03 | 4.5                  | 2025-01-02      | 6.8                    | -2.3      |
| 2025-01-04 | 7.1                  | 2025-01-03      | 4.5                    | 2.6       |
| 2025-01-05 | 8.3                  | 2025-01-04      | 7.1                    | 1.2       |

**Alternative moderne** : Utiliser les **window functions** (LAG/LEAD) - Plus performant !

```sql
SELECT
    date,
    temperature,
    LAG(temperature) OVER (ORDER BY date) AS temperature_precedente,
    temperature - LAG(temperature) OVER (ORDER BY date) AS variation
FROM mesures_temperature  
ORDER BY date;  
```

**Note** : Les window functions sont généralement préférées pour ce type d'opération, mais les self-joins restent utiles pour des comparaisons plus complexes.

---

## 7. Self-Joins avec Différents Types de Jointures

### INNER JOIN : Seulement les Correspondances

```sql
-- Employés ayant un manager (exclut le PDG)
SELECT e.nom AS employe, m.nom AS manager  
FROM employes AS e  
INNER JOIN employes AS m ON e.manager_id = m.id;  
```

**Résultat** : Alice (PDG) n'apparaît pas car elle n'a pas de manager.

### LEFT JOIN : Inclure Tous les Employés

```sql
-- Tous les employés, avec manager si existe
SELECT e.nom AS employe, m.nom AS manager  
FROM employes AS e  
LEFT JOIN employes AS m ON e.manager_id = m.id;  
```

**Résultat** : Alice apparaît avec `NULL` comme manager.

### RIGHT JOIN : Tous les Managers

```sql
-- Tous les employés qui sont managers, avec leurs subordonnés
SELECT e.nom AS employe, m.nom AS manager  
FROM employes AS e  
RIGHT JOIN employes AS m ON e.manager_id = m.id;  
```

**Résultat** : Tous les managers apparaissent, même ceux sans subordonnés directs.

### Self-Join avec Agrégation

```sql
-- Nombre de subordonnés par manager
SELECT
    m.nom AS manager,
    COUNT(e.id) AS nombre_subordonnes
FROM employes AS m  
LEFT JOIN employes AS e ON m.id = e.manager_id  
GROUP BY m.id, m.nom  
ORDER BY nombre_subordonnes DESC;  
```

---

## 8. Self-Joins Multiples (Plus de 2 Niveaux)

### Hiérarchie à 3 Niveaux

**Objectif** : Afficher employé → manager → grand-manager

```sql
SELECT
    e.nom AS employe,
    m1.nom AS manager,
    m2.nom AS directeur
FROM employes AS e  
LEFT JOIN employes AS m1 ON e.manager_id = m1.id  
LEFT JOIN employes AS m2 ON m1.manager_id = m2.id  
ORDER BY e.id;  
```

**Résultat** :

| employe         | manager         | directeur    |
|-----------------|-----------------|--------------|
| Alice Martin    | NULL            | NULL         |
| Bob Dupont      | Alice Martin    | NULL         |
| Charlie Durand  | Alice Martin    | NULL         |
| Diana Bernard   | Bob Dupont      | Alice Martin |
| Eve Laurent     | Bob Dupont      | Alice Martin |
| Frank Petit     | Charlie Durand  | Alice Martin |

### Chaîne Hiérarchique Complète

Pour afficher **toute la hiérarchie** d'un employé jusqu'au sommet, on utiliserait une **CTE récursive** (voir section 9.3), mais voici un aperçu :

```sql
WITH RECURSIVE hierarchy AS (
    -- Point de départ : l'employé ciblé
    SELECT id, nom, manager_id, 1 AS niveau
    FROM employes
    WHERE id = 4  -- Diana Bernard

    UNION ALL

    -- Récursion : remonter la hiérarchie
    SELECT e.id, e.nom, e.manager_id, h.niveau + 1
    FROM employes e
    INNER JOIN hierarchy h ON e.id = h.manager_id
)
SELECT nom, niveau  
FROM hierarchy  
ORDER BY niveau;  
```

**Résultat** :

| nom           | niveau |
|---------------|--------|
| Diana Bernard | 1      |
| Bob Dupont    | 2      |
| Alice Martin  | 3      |

---

## 9. Pièges et Erreurs Courantes

### Erreur 1 : Oublier les Alias

```sql
-- ❌ ERREUR
SELECT *  
FROM employes  
JOIN employes ON employes.manager_id = employes.id;  
-- ERROR: table name "employes" specified more than once
```

**Solution** : Toujours utiliser des alias !

```sql
-- ✅ CORRECT
SELECT *  
FROM employes AS e  
JOIN employes AS m ON e.manager_id = m.id;  
```

### Erreur 2 : Créer des Boucles Infinies (Comparaisons)

```sql
-- ⚠️ DANGER : Produit cartésien !
SELECT *  
FROM employes AS e1  
CROSS JOIN employes AS e2;  
-- Résultat : n × n lignes (explosion combinatoire)
```

**Solution** : Toujours avoir une condition de jointure précise.

### Erreur 3 : Paires Symétriques et Auto-Match

Lors de comparaisons, sans précaution, vous obtenez des doublons :

```sql
-- ❌ Problème : Paires (A,B) et (B,A), plus (A,A)
SELECT e1.nom, e2.nom  
FROM employes AS e1  
JOIN employes AS e2 ON e1.salaire = e2.salaire;  
```

**Solution** : Utiliser `e1.id < e2.id`

```sql
-- ✅ CORRECT : Pas de doublons, pas d'auto-match
SELECT e1.nom, e2.nom  
FROM employes AS e1  
JOIN employes AS e2 ON e1.salaire = e2.salaire  
WHERE e1.id < e2.id;  
```

### Erreur 4 : Confusion entre INNER et LEFT JOIN

```sql
-- INNER JOIN : Exclut les employés sans manager
SELECT e.nom, m.nom  
FROM employes AS e  
INNER JOIN employes AS m ON e.manager_id = m.id;  
-- Alice (PDG) ne apparaît pas

-- LEFT JOIN : Inclut tous les employés
SELECT e.nom, m.nom  
FROM employes AS e  
LEFT JOIN employes AS m ON e.manager_id = m.id;  
-- Alice apparaît avec NULL
```

**Règle** : Utilisez LEFT JOIN si vous voulez inclure les lignes sans correspondance.

### Erreur 5 : Oublier ORDER BY avec Séquences

Sans `ORDER BY`, les comparaisons séquentielles peuvent être incorrectes :

```sql
-- ⚠️ Sans ORDER BY, l'ordre n'est pas garanti
SELECT t1.date, t2.date  
FROM mesures_temperature AS t1  
JOIN mesures_temperature AS t2 ON t1.id = t2.id + 1;  

-- ✅ Avec ORDER BY explicite
SELECT t1.date, t2.date  
FROM mesures_temperature AS t1  
JOIN mesures_temperature AS t2 ON t1.id = t2.id + 1  
ORDER BY t1.date;  
```

---

## 10. Performances et Optimisation

### Bonnes Pratiques

#### 1. Indexer les Colonnes de Jointure

```sql
-- Index sur manager_id pour accélérer les self-joins
CREATE INDEX idx_employes_manager_id ON employes(manager_id);

-- Index sur les colonnes utilisées dans les comparaisons
CREATE INDEX idx_employes_salaire ON employes(salaire);
```

#### 2. Utiliser EXPLAIN pour Vérifier

```sql
EXPLAIN ANALYZE  
SELECT e.nom, m.nom  
FROM employes AS e  
LEFT JOIN employes AS m ON e.manager_id = m.id;  
```

Vérifiez que les index sont utilisés (cherchez "Index Scan" plutôt que "Seq Scan").

#### 3. Limiter les Résultats

Pour les tables volumineuses, limitez les résultats lors des tests :

```sql
SELECT e1.nom, e2.nom  
FROM employes AS e1  
JOIN employes AS e2 ON e1.salaire = e2.salaire  
WHERE e1.id < e2.id  
LIMIT 100;  
```

#### 4. Préférer Window Functions pour les Séquences

Pour les comparaisons séquentielles (précédent/suivant), les **window functions** sont plus performantes :

```sql
-- ✅ Plus performant
SELECT
    date,
    temperature,
    LAG(temperature) OVER (ORDER BY date) AS temperature_precedente
FROM mesures_temperature;

-- vs

-- ⚠️ Moins performant
SELECT
    t1.date,
    t1.temperature,
    t2.temperature AS temperature_precedente
FROM mesures_temperature AS t1  
LEFT JOIN mesures_temperature AS t2 ON t1.id = t2.id + 1;  
```

#### 5. Attention aux Self-Joins sur Grandes Tables

Un self-join sur une table de 1 million de lignes peut générer des résultats énormes :

```
1 000 000 × 1 000 000 = 1 trillion de comparaisons potentielles !
```

**Solutions** :
- Filtrer au maximum (`WHERE`)
- Utiliser des index
- Partitionner les données
- Considérer des approches alternatives (window functions, CTEs)

---

## 11. Cas d'Usage Avancés

### Exemple 1 : Graphe Social (Amis d'Amis)

**Objectif** : Recommandations d'amis (amis de mes amis qui ne sont pas encore mes amis).

```sql
-- Trouver les amis potentiels pour Alice (id=1)
SELECT DISTINCT
    u.nom AS ami_potentiel
FROM amities AS a1  -- Mes amis  
INNER JOIN amities AS a2 ON a1.ami_id = a2.utilisateur_id  -- Amis de mes amis  
INNER JOIN utilisateurs AS u ON a2.ami_id = u.id  
WHERE a1.utilisateur_id = 1  -- Alice  
  AND a2.ami_id != 1  -- Pas moi-même
  AND a2.ami_id NOT IN (  -- Pas déjà mon ami
      SELECT ami_id FROM amities WHERE utilisateur_id = 1
  )
ORDER BY u.nom;
```

### Exemple 2 : Détection de Cycles

**Objectif** : Détecter les employés qui sont indirectement leurs propres managers (cycle dans la hiérarchie).

```sql
-- Cycle à 2 niveaux (A gère B, B gère A)
SELECT
    e1.nom AS employe1,
    e2.nom AS employe2
FROM employes AS e1  
INNER JOIN employes AS e2 ON e1.manager_id = e2.id  
WHERE e2.manager_id = e1.id;  
```

**Note** : Pour des cycles plus profonds, utilisez des CTEs récursives.

### Exemple 3 : Trouver les "Orphelins" (Références Invalides)

**Objectif** : Identifier les employés dont le manager_id ne correspond à aucun employé.

```sql
SELECT
    e.id,
    e.nom,
    e.manager_id
FROM employes AS e  
LEFT JOIN employes AS m ON e.manager_id = m.id  
WHERE e.manager_id IS NOT NULL  
  AND m.id IS NULL;
```

**Résultat** : Si un employé a un `manager_id` qui ne correspond à aucun ID existant, il apparaîtra ici.

### Exemple 4 : Distance dans un Graphe (Degrés de Séparation)

**Objectif** : Calculer le nombre de "sauts" entre deux personnes dans un réseau social.

```sql
-- Distance 1 : Amis directs
SELECT a.utilisateur_id, a.ami_id, 1 AS distance  
FROM amities AS a  
WHERE a.utilisateur_id = 1  

UNION

-- Distance 2 : Amis d'amis
SELECT a1.utilisateur_id, a2.ami_id, 2 AS distance  
FROM amities AS a1  
INNER JOIN amities AS a2 ON a1.ami_id = a2.utilisateur_id  
WHERE a1.utilisateur_id = 1  
  AND a2.ami_id != 1;
```

**Note** : Pour des distances arbitraires, utilisez des CTEs récursives.

---

## 12. Alternatives aux Self-Joins

### 1. Window Functions (LAG, LEAD)

Pour les comparaisons séquentielles :

```sql
-- Self-join
SELECT
    t1.date,
    t1.temperature - t2.temperature AS variation
FROM mesures_temperature AS t1  
LEFT JOIN mesures_temperature AS t2 ON t1.id = t2.id + 1;  

-- Window function (préférable)
SELECT
    date,
    temperature - LAG(temperature) OVER (ORDER BY date) AS variation
FROM mesures_temperature;
```

### 2. CTEs Récursives

Pour les hiérarchies profondes :

```sql
-- Self-join (limité à quelques niveaux)
SELECT e.nom, m1.nom, m2.nom  
FROM employes AS e  
LEFT JOIN employes AS m1 ON e.manager_id = m1.id  
LEFT JOIN employes AS m2 ON m1.manager_id = m2.id;  

-- CTE récursive (niveaux illimités)
WITH RECURSIVE hierarchy AS (
    SELECT id, nom, manager_id, 1 AS niveau
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.nom, e.manager_id, h.niveau + 1
    FROM employes e
    INNER JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy;
```

### 3. Tableaux et Types JSONB

Pour des structures complexes :

```sql
-- Stocker les amis dans un tableau
CREATE TABLE utilisateurs_v2 (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    amis INTEGER[]  -- Tableau d'IDs
);

-- Requête
SELECT nom, amis  
FROM utilisateurs_v2  
WHERE 1 = ANY(amis);  -- Utilisateurs dont Alice (id=1) est amie  
```

---

## 13. Comparaison : Self-Join vs Autres Techniques

| Cas d'Usage | Self-Join | Alternative | Recommandation |
|-------------|-----------|-------------|----------------|
| Employé → Manager | ✅ Idéal | CTE récursive (>2 niveaux) | Self-join |
| Ligne précédente/suivante | ⚠️ OK | Window functions (LAG/LEAD) | Window functions |
| Hiérarchie profonde | ⚠️ Limité | CTE récursive | CTE récursive |
| Comparaisons (doublons) | ✅ Idéal | DISTINCT, GROUP BY | Self-join |
| Graphe social | ✅ Flexible | CTE récursive, Graph DB | Dépend de la complexité |

---

## 14. Récapitulatif : Modèles de Self-Joins

### Modèle 1 : Hiérarchie Parent-Enfant

```sql
SELECT
    enfant.nom AS enfant,
    parent.nom AS parent
FROM table AS enfant  
LEFT JOIN table AS parent ON enfant.parent_id = parent.id;  
```

### Modèle 2 : Comparaison (Éviter Doublons)

```sql
SELECT
    t1.colonne,
    t2.colonne
FROM table AS t1  
INNER JOIN table AS t2 ON t1.colonne_cle = t2.colonne_cle  
WHERE t1.id < t2.id;  -- Clé pour éviter doublons  
```

### Modèle 3 : Séquence (Précédent/Suivant)

```sql
SELECT
    t1.id,
    t1.valeur AS valeur_actuelle,
    t2.valeur AS valeur_precedente
FROM table AS t1  
LEFT JOIN table AS t2 ON t1.id = t2.id + 1  
ORDER BY t1.id;  
```

### Modèle 4 : Relation Symétrique

```sql
SELECT DISTINCT
    u1.nom,
    u2.nom
FROM relations AS r1  
INNER JOIN relations AS r2 ON r1.utilisateur_id = r2.ami_id  
                          AND r1.ami_id = r2.utilisateur_id
INNER JOIN utilisateurs AS u1 ON r1.utilisateur_id = u1.id  
INNER JOIN utilisateurs AS u2 ON r1.ami_id = u2.id  
WHERE r1.utilisateur_id < r1.ami_id;  
```

---

## Conclusion

### Points Clés à Retenir

1. **Self-join** = Joindre une table à elle-même en utilisant des alias différents  
2. **Alias obligatoires** : `FROM table AS t1 JOIN table AS t2`  
3. **Cas d'usage principaux** :
   - Hiérarchies (employés/managers)
   - Comparaisons entre lignes
   - Relations symétriques (amis)
   - Séquences temporelles

4. **Éviter les pièges** :
   - Utiliser `e1.id < e2.id` pour éviter doublons
   - LEFT JOIN pour inclure les lignes sans correspondance
   - Indexer les colonnes de jointure

5. **Alternatives** :
   - Window functions pour les séquences
   - CTEs récursives pour les hiérarchies profondes

### Quand Utiliser un Self-Join ?

✅ **Utilisez un self-join** quand :
- Vous avez une relation parent-enfant simple (1-2 niveaux)
- Vous devez comparer des lignes entre elles
- Vous recherchez des doublons ou similarités
- Vous gérez des relations symétriques

⚠️ **Considérez des alternatives** quand :
- Hiérarchies > 2-3 niveaux (→ CTE récursive)
- Comparaisons séquentielles (→ Window functions)
- Graphes complexes (→ Extensions dédiées ou Graph DB)

### Prochaines Étapes

Dans les sections suivantes, nous explorerons :
- **7.6 LATERAL JOIN** : Jointures corrélées avancées  
- **7.7 Anti-jointures et Semi-jointures** : NOT EXISTS, NOT IN, EXCEPT

Le self-join est une technique puissante qui, une fois maîtrisée, ouvre de nombreuses possibilités pour résoudre des problèmes complexes avec élégance !

⏭️ [Jointures latérales (LATERAL JOIN) : Corrélation avancée](/07-relations-et-jointures/06-lateral-join.md)
