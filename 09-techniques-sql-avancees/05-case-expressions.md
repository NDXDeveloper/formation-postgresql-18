🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5. CASE expressions et logique conditionnelle

## Introduction

L'expression `CASE` est l'équivalent SQL du **if/else** des langages de programmation. Elle permet d'introduire de la **logique conditionnelle** directement dans vos requêtes SQL, vous offrant la possibilité de transformer, catégoriser ou filtrer des données selon des conditions spécifiques.

`CASE` est une expression, pas une instruction : elle **retourne une valeur** et peut être utilisée partout où une valeur est attendue (SELECT, WHERE, ORDER BY, UPDATE, etc.).

---

## Pourquoi utiliser CASE ?

### Sans CASE (limité)

```sql
-- Impossible de catégoriser directement
SELECT nom, salaire FROM employes;
```

**Résultat brut :**
```
nom      | salaire
---------+---------
Alice    | 35000  
Bob      | 55000  
Charlie  | 80000  
```

### Avec CASE (puissant)

```sql
-- Catégorisation dynamique
SELECT
    nom,
    salaire,
    CASE
        WHEN salaire < 40000 THEN 'Junior'
        WHEN salaire < 70000 THEN 'Intermédiaire'
        ELSE 'Senior'
    END AS niveau
FROM employes;
```

**Résultat enrichi :**
```
nom      | salaire | niveau
---------+---------+---------------
Alice    | 35000   | Junior  
Bob      | 55000   | Intermédiaire  
Charlie  | 80000   | Senior  
```

---

## Les deux syntaxes de CASE

PostgreSQL supporte deux formes d'expression `CASE` :
1. **CASE simple** (comparaison d'égalité)  
2. **CASE recherché** (conditions complexes)

---

## 1. CASE Simple (Égalité)

### Syntaxe

```sql
CASE expression
    WHEN valeur1 THEN resultat1
    WHEN valeur2 THEN resultat2
    [WHEN valeurN THEN resultatN ...]
    [ELSE resultat_defaut]
END
```

**Principe :** Compare `expression` à chaque `valeur` avec un test d'égalité (`=`).

### Exemple de base

```sql
SELECT
    nom,
    departement,
    CASE departement
        WHEN 'IT' THEN 'Informatique'
        WHEN 'HR' THEN 'Ressources Humaines'
        WHEN 'Sales' THEN 'Ventes'
        ELSE 'Autre'
    END AS departement_fr
FROM employes;
```

**Résultat :**
```
nom     | departement | departement_fr
--------+-------------+---------------------
Alice   | IT          | Informatique  
Bob     | HR          | Ressources Humaines  
Charlie | Sales       | Ventes  
Diana   | Legal       | Autre  
```

### Équivalent en logique

```sql
CASE departement
    WHEN 'IT' THEN ...
END

-- Est équivalent à :
IF departement = 'IT' THEN ...  
ELSIF departement = 'HR' THEN ...  
ELSE ...  
```

### Cas d'usage : Traduction / Mapping

```sql
-- Traduire des codes en libellés
SELECT
    code_statut,
    CASE code_statut
        WHEN 'P' THEN 'En attente'
        WHEN 'A' THEN 'Approuvé'
        WHEN 'R' THEN 'Rejeté'
        WHEN 'C' THEN 'Annulé'
        ELSE 'Inconnu'
    END AS statut_libelle
FROM commandes;
```

---

## 2. CASE Recherché (Conditions complexes)

### Syntaxe

```sql
CASE
    WHEN condition1 THEN resultat1
    WHEN condition2 THEN resultat2
    [WHEN conditionN THEN resultatN ...]
    [ELSE resultat_defaut]
END
```

**Principe :** Évalue chaque `condition` booléenne dans l'ordre et retourne le résultat de la première condition vraie.

### Exemple de base

```sql
SELECT
    nom,
    age,
    CASE
        WHEN age < 18 THEN 'Mineur'
        WHEN age < 65 THEN 'Adulte'
        ELSE 'Senior'
    END AS categorie_age
FROM personnes;
```

**Résultat :**
```
nom     | age | categorie_age
--------+-----+--------------
Alice   | 16  | Mineur  
Bob     | 35  | Adulte  
Charlie | 70  | Senior  
```

### Conditions multiples

Vous pouvez combiner plusieurs conditions avec `AND`, `OR`, etc.

```sql
SELECT
    nom,
    salaire,
    anciennete,
    CASE
        WHEN salaire > 70000 AND anciennete > 5 THEN 'Expert Senior'
        WHEN salaire > 70000 THEN 'Expert'
        WHEN salaire > 40000 AND anciennete > 3 THEN 'Confirmé'
        WHEN salaire > 40000 THEN 'Intermédiaire'
        ELSE 'Junior'
    END AS profil
FROM employes;
```

### Ordre d'évaluation : Important !

⚠️ **Les conditions sont évaluées de haut en bas** et s'arrêtent à la première condition vraie.

```sql
-- ❌ MAUVAIS ORDRE
CASE
    WHEN age > 0 THEN 'Valide'      -- Toujours vrai en premier !
    WHEN age < 18 THEN 'Mineur'     -- Jamais atteint
    WHEN age < 65 THEN 'Adulte'     -- Jamais atteint
    ELSE 'Senior'
END

-- ✅ BON ORDRE (du plus spécifique au plus général)
CASE
    WHEN age < 18 THEN 'Mineur'
    WHEN age < 65 THEN 'Adulte'
    ELSE 'Senior'
END
```

---

## Utilisation dans différentes clauses

### 1. Dans SELECT (le plus courant)

**Créer des colonnes calculées :**

```sql
SELECT
    nom,
    salaire,
    CASE
        WHEN salaire >= 60000 THEN salaire * 1.05  -- +5%
        WHEN salaire >= 40000 THEN salaire * 1.10  -- +10%
        ELSE salaire * 1.15  -- +15%
    END AS salaire_augmente
FROM employes;
```

**Formatage conditionnel :**

```sql
SELECT
    nom,
    solde,
    CASE
        WHEN solde < 0 THEN CONCAT(solde::TEXT, ' € (débiteur)')
        WHEN solde = 0 THEN '0 € (neutre)'
        ELSE CONCAT('+', solde::TEXT, ' € (créditeur)')
    END AS solde_formate
FROM comptes;
```

### 2. Dans WHERE (filtrage conditionnel)

```sql
-- Filtrer selon une logique complexe
SELECT nom, salaire, departement  
FROM employes  
WHERE  
    CASE departement
        WHEN 'IT' THEN salaire > 50000
        WHEN 'Sales' THEN salaire > 40000
        ELSE salaire > 35000
    END;
```

**Astuce :** Souvent, on peut aussi utiliser des conditions OR/AND, mais CASE peut être plus lisible.

### 3. Dans ORDER BY (tri conditionnel)

```sql
-- Trier différemment selon une condition
SELECT nom, statut, date_creation  
FROM utilisateurs  
ORDER BY  
    CASE statut
        WHEN 'VIP' THEN 1
        WHEN 'Premium' THEN 2
        WHEN 'Standard' THEN 3
        ELSE 4
    END,
    date_creation DESC;
```

**Tri personnalisé :**

```sql
-- Mettre certaines valeurs en premier
SELECT nom, ville  
FROM clients  
ORDER BY  
    CASE ville
        WHEN 'Paris' THEN 1
        WHEN 'Lyon' THEN 2
        WHEN 'Marseille' THEN 3
        ELSE 4
    END,
    nom;
```

### 4. Dans GROUP BY

```sql
-- Grouper par catégories dynamiques
SELECT
    CASE
        WHEN age < 25 THEN '18-24'
        WHEN age < 35 THEN '25-34'
        WHEN age < 50 THEN '35-49'
        ELSE '50+'
    END AS tranche_age,
    COUNT(*) AS nombre
FROM utilisateurs  
GROUP BY  
    CASE
        WHEN age < 25 THEN '18-24'
        WHEN age < 35 THEN '25-34'
        WHEN age < 50 THEN '35-49'
        ELSE '50+'
    END
ORDER BY tranche_age;
```

**Note :** Vous devez répéter l'expression CASE (ou utiliser une CTE/sous-requête pour éviter la répétition).

### 5. Dans UPDATE

```sql
-- Mise à jour conditionnelle
UPDATE employes  
SET salaire = CASE  
    WHEN performance = 'Excellent' THEN salaire * 1.15
    WHEN performance = 'Bon' THEN salaire * 1.10
    WHEN performance = 'Satisfaisant' THEN salaire * 1.05
    ELSE salaire
END  
WHERE evaluation_date = CURRENT_DATE;  
```

### 6. Dans des agrégations

```sql
-- Compter conditionnellement
SELECT
    departement,
    COUNT(*) AS total_employes,
    COUNT(CASE WHEN salaire > 50000 THEN 1 END) AS nb_salaire_eleve,
    COUNT(CASE WHEN salaire <= 50000 THEN 1 END) AS nb_salaire_bas
FROM employes  
GROUP BY departement;  
```

**Astuce :** `COUNT(CASE WHEN condition THEN 1 END)` compte uniquement les lignes où la condition est vraie.

---

## CASE imbriqués

Vous pouvez imbriquer des expressions `CASE` pour des logiques complexes.

### Exemple : Double catégorisation

```sql
SELECT
    nom,
    salaire,
    anciennete,
    CASE
        WHEN salaire > 70000 THEN
            CASE
                WHEN anciennete > 10 THEN 'Expert Senior Confirmé'
                WHEN anciennete > 5 THEN 'Expert Senior'
                ELSE 'Expert'
            END
        WHEN salaire > 50000 THEN
            CASE
                WHEN anciennete > 5 THEN 'Senior Confirmé'
                ELSE 'Senior'
            END
        ELSE 'Junior/Intermédiaire'
    END AS classification
FROM employes;
```

### Recommandation

⚠️ **Évitez trop d'imbrications** (plus de 2 niveaux) : le code devient difficile à lire.

**Alternative avec CTE :**

```sql
WITH employes_categorises AS (
    SELECT
        *,
        CASE
            WHEN salaire > 70000 THEN 'Haut'
            WHEN salaire > 50000 THEN 'Moyen'
            ELSE 'Bas'
        END AS categorie_salaire
)
SELECT
    nom,
    salaire,
    anciennete,
    CASE categorie_salaire
        WHEN 'Haut' THEN
            CASE
                WHEN anciennete > 10 THEN 'Expert Senior Confirmé'
                WHEN anciennete > 5 THEN 'Expert Senior'
                ELSE 'Expert'
            END
        WHEN 'Moyen' THEN
            CASE WHEN anciennete > 5 THEN 'Senior Confirmé' ELSE 'Senior' END
        ELSE 'Junior/Intermédiaire'
    END AS classification
FROM employes_categorises;
```

---

## Clause ELSE : Obligatoire ou optionnelle ?

### Sans ELSE

Si aucune condition n'est vraie et qu'il n'y a pas de clause `ELSE`, `CASE` retourne `NULL`.

```sql
SELECT
    nom,
    CASE
        WHEN salaire > 100000 THEN 'Très élevé'
        WHEN salaire > 70000 THEN 'Élevé'
    END AS niveau_salaire
FROM employes;
```

**Si salaire = 40000 :**
```
nom   | niveau_salaire
------+----------------
Alice | NULL           ← Aucune condition vraie, pas de ELSE
```

### Avec ELSE (recommandé)

Pour éviter les `NULL` inattendus, ajoutez toujours un `ELSE`.

```sql
SELECT
    nom,
    CASE
        WHEN salaire > 100000 THEN 'Très élevé'
        WHEN salaire > 70000 THEN 'Élevé'
        ELSE 'Standard'  -- ✅ Valeur par défaut
    END AS niveau_salaire
FROM employes;
```

---

## Fonctions conditionnelles associées

PostgreSQL offre des fonctions qui simplifient certains usages de `CASE`.

### 1. COALESCE : Première valeur non-NULL

**Syntaxe :**
```sql
COALESCE(valeur1, valeur2, ..., valeurN)
```

Retourne la première valeur non-NULL.

**Exemple :**

```sql
-- Sans COALESCE (avec CASE)
SELECT
    nom,
    CASE
        WHEN email IS NOT NULL THEN email
        WHEN telephone IS NOT NULL THEN telephone
        ELSE 'Non renseigné'
    END AS contact
FROM clients;

-- Avec COALESCE (plus concis)
SELECT
    nom,
    COALESCE(email, telephone, 'Non renseigné') AS contact
FROM clients;
```

**Cas d'usage courants :**

```sql
-- Valeur par défaut
SELECT COALESCE(prenom, 'Anonyme') AS prenom FROM utilisateurs;

-- Remplacement de NULL dans des calculs
SELECT nom, salaire + COALESCE(bonus, 0) AS remuneration_totale  
FROM employes;  

-- Chaînage de plusieurs colonnes
SELECT COALESCE(email_pro, email_perso, telephone, 'Aucun contact') AS contact  
FROM personnes;  
```

### 2. NULLIF : Convertir une valeur en NULL

**Syntaxe :**
```sql
NULLIF(valeur1, valeur2)
```

Retourne `NULL` si `valeur1 = valeur2`, sinon retourne `valeur1`.

**Exemple :**

```sql
-- Convertir les chaînes vides en NULL
SELECT
    nom,
    NULLIF(email, '') AS email  -- Si email = '', retourne NULL
FROM clients;

-- Éviter la division par zéro
SELECT
    montant,
    quantite,
    montant / NULLIF(quantite, 0) AS prix_unitaire  -- Si quantite = 0, retourne NULL
FROM lignes_commande;
```

**Équivalent CASE :**

```sql
-- NULLIF(x, y) est équivalent à :
CASE WHEN x = y THEN NULL ELSE x END
```

### 3. GREATEST et LEAST : Maximum et minimum

**GREATEST :** Retourne la plus grande valeur.
```sql
SELECT GREATEST(10, 25, 5, 30);  -- Résultat : 30
```

**LEAST :** Retourne la plus petite valeur.
```sql
SELECT LEAST(10, 25, 5, 30);  -- Résultat : 5
```

**Exemple pratique :**

```sql
-- Plafonner un salaire à 100 000
SELECT
    nom,
    salaire,
    LEAST(salaire, 100000) AS salaire_plafonne
FROM employes;

-- Garantir un salaire minimum de 30 000
SELECT
    nom,
    salaire,
    GREATEST(salaire, 30000) AS salaire_minimum
FROM employes;
```

**Avec NULL :**

⚠️ Si **une seule valeur** est `NULL`, le résultat est `NULL`.

```sql
SELECT GREATEST(10, NULL, 30);  -- Résultat : NULL  
SELECT LEAST(10, NULL, 30);     -- Résultat : NULL  
```

---

## Cas d'usage pratiques

### 1. Pivot de données (transformation ligne → colonne)

**Problème :** Transformer des lignes en colonnes.

```sql
-- Table source
CREATE TABLE ventes (
    mois INTEGER,
    categorie VARCHAR(50),
    montant NUMERIC
);

INSERT INTO ventes VALUES
    (1, 'Électronique', 1000),
    (1, 'Mode', 500),
    (2, 'Électronique', 1200),
    (2, 'Mode', 600);
```

**Requête avec CASE (pivot) :**

```sql
SELECT
    mois,
    SUM(CASE WHEN categorie = 'Électronique' THEN montant ELSE 0 END) AS electronique,
    SUM(CASE WHEN categorie = 'Mode' THEN montant ELSE 0 END) AS mode
FROM ventes  
GROUP BY mois  
ORDER BY mois;  
```

**Résultat :**

```
mois | electronique | mode
-----+--------------+------
   1 |         1000 |  500
   2 |         1200 |  600
```

### 2. Agrégation conditionnelle

**Compter selon des critères multiples :**

```sql
SELECT
    departement,
    COUNT(*) AS total,
    COUNT(CASE WHEN salaire > 50000 THEN 1 END) AS nb_salaire_eleve,
    COUNT(CASE WHEN age < 30 THEN 1 END) AS nb_jeunes,
    AVG(CASE WHEN sexe = 'F' THEN salaire END) AS salaire_moyen_femmes,
    AVG(CASE WHEN sexe = 'M' THEN salaire END) AS salaire_moyen_hommes
FROM employes  
GROUP BY departement;  
```

### 3. Scoring et notation

```sql
SELECT
    client_id,
    nom,
    CASE
        WHEN achats_total > 10000 THEN 100
        WHEN achats_total > 5000 THEN 75
        WHEN achats_total > 1000 THEN 50
        ELSE 25
    END +
    CASE
        WHEN nb_commandes > 50 THEN 50
        WHEN nb_commandes > 20 THEN 30
        WHEN nb_commandes > 5 THEN 10
        ELSE 0
    END +
    CASE
        WHEN anciennete_mois > 36 THEN 25
        WHEN anciennete_mois > 12 THEN 15
        ELSE 5
    END AS score_fidelite
FROM clients;
```

### 4. Catégorisation dynamique pour rapports

```sql
-- Rapport avec catégories personnalisées
SELECT
    CASE
        WHEN EXTRACT(HOUR FROM date_commande) BETWEEN 0 AND 5 THEN 'Nuit (0h-5h)'
        WHEN EXTRACT(HOUR FROM date_commande) BETWEEN 6 AND 11 THEN 'Matin (6h-11h)'
        WHEN EXTRACT(HOUR FROM date_commande) BETWEEN 12 AND 17 THEN 'Après-midi (12h-17h)'
        ELSE 'Soirée (18h-23h)'
    END AS tranche_horaire,
    COUNT(*) AS nb_commandes,
    SUM(montant) AS total_ventes
FROM commandes  
WHERE date_commande >= CURRENT_DATE - INTERVAL '30 days'  
GROUP BY  
    CASE
        WHEN EXTRACT(HOUR FROM date_commande) BETWEEN 0 AND 5 THEN 'Nuit (0h-5h)'
        WHEN EXTRACT(HOUR FROM date_commande) BETWEEN 6 AND 11 THEN 'Matin (6h-11h)'
        WHEN EXTRACT(HOUR FROM date_commande) BETWEEN 12 AND 17 THEN 'Après-midi (12h-17h)'
        ELSE 'Soirée (18h-23h)'
    END
ORDER BY
    CASE tranche_horaire
        WHEN 'Nuit (0h-5h)' THEN 1
        WHEN 'Matin (6h-11h)' THEN 2
        WHEN 'Après-midi (12h-17h)' THEN 3
        ELSE 4
    END;
```

### 5. Gestion des règles métier complexes

```sql
-- Calcul de remise selon des règles métier
SELECT
    commande_id,
    client_id,
    montant_brut,
    CASE
        -- VIP : 20% sur tout
        WHEN client_type = 'VIP' THEN montant_brut * 0.80

        -- Nouveaux clients : 15% si > 100€
        WHEN client_nouveau = TRUE AND montant_brut > 100 THEN montant_brut * 0.85

        -- Clients fidèles : paliers
        WHEN client_fidelite_points > 1000 AND montant_brut > 200 THEN montant_brut * 0.85
        WHEN client_fidelite_points > 500 AND montant_brut > 100 THEN montant_brut * 0.90

        -- Black Friday : 10% sur tout
        WHEN EXTRACT(MONTH FROM date_commande) = 11
             AND EXTRACT(DAY FROM date_commande) BETWEEN 25 AND 30
             THEN montant_brut * 0.90

        -- Pas de remise
        ELSE montant_brut
    END AS montant_final
FROM commandes;
```

### 6. Nettoyage et standardisation de données

```sql
-- Normaliser des données incohérentes
UPDATE clients  
SET pays = CASE  
    WHEN LOWER(pays) IN ('france', 'fr', 'fra', 'french') THEN 'FR'
    WHEN LOWER(pays) IN ('germany', 'de', 'deu', 'allemagne') THEN 'DE'
    WHEN LOWER(pays) IN ('uk', 'gb', 'united kingdom', 'royaume-uni') THEN 'GB'
    WHEN LOWER(pays) IN ('usa', 'us', 'united states', 'états-unis') THEN 'US'
    ELSE UPPER(LEFT(pays, 2))  -- Essayer de garder les 2 premières lettres
END  
WHERE pays IS NOT NULL;  
```

### 7. Indicateurs de qualité des données

```sql
-- Score de complétude d'un profil
SELECT
    user_id,
    nom,
    (
        CASE WHEN nom IS NOT NULL AND nom != '' THEN 10 ELSE 0 END +
        CASE WHEN prenom IS NOT NULL AND prenom != '' THEN 10 ELSE 0 END +
        CASE WHEN email IS NOT NULL AND email LIKE '%@%' THEN 20 ELSE 0 END +
        CASE WHEN telephone IS NOT NULL THEN 15 ELSE 0 END +
        CASE WHEN adresse IS NOT NULL THEN 15 ELSE 0 END +
        CASE WHEN ville IS NOT NULL THEN 10 ELSE 0 END +
        CASE WHEN code_postal IS NOT NULL THEN 10 ELSE 0 END +
        CASE WHEN date_naissance IS NOT NULL THEN 10 ELSE 0 END
    ) AS score_completude,
    CASE
        WHEN (score calculé ci-dessus) >= 90 THEN 'Excellent'
        WHEN (score calculé ci-dessus) >= 70 THEN 'Bon'
        WHEN (score calculé ci-dessus) >= 50 THEN 'Moyen'
        ELSE 'Faible'
    END AS qualite_profil
FROM utilisateurs;
```

---

## Performance et optimisation

### 1. CASE vs fonctions

Pour des logiques simples, les fonctions dédiées sont souvent plus performantes.

```sql
-- ❌ MOINS EFFICACE
SELECT
    CASE
        WHEN colonne IS NULL THEN 'Défaut'
        ELSE colonne
    END
FROM table;

-- ✅ PLUS EFFICACE
SELECT COALESCE(colonne, 'Défaut') FROM table;
```

### 2. CASE dans WHERE : Impact sur les index

Les expressions `CASE` dans `WHERE` peuvent empêcher l'utilisation d'index.

```sql
-- ❌ Pas d'utilisation d'index possible
SELECT * FROM employes  
WHERE CASE  
    WHEN departement = 'IT' THEN salaire > 50000
    ELSE salaire > 40000
END;

-- ✅ Réécriture pour utiliser les index
SELECT * FROM employes  
WHERE (departement = 'IT' AND salaire > 50000)  
   OR (departement != 'IT' AND salaire > 40000);
```

### 3. Éviter les CASE répétitifs

```sql
-- ❌ Répétition inutile
SELECT
    CASE
        WHEN age < 18 THEN 'Mineur'
        WHEN age < 65 THEN 'Adulte'
        ELSE 'Senior'
    END AS categorie,
    COUNT(*)
FROM personnes  
GROUP BY  
    CASE
        WHEN age < 18 THEN 'Mineur'
        WHEN age < 65 THEN 'Adulte'
        ELSE 'Senior'
    END;

-- ✅ Avec CTE (plus clair et potentiellement plus rapide)
WITH personnes_categorisees AS (
    SELECT
        *,
        CASE
            WHEN age < 18 THEN 'Mineur'
            WHEN age < 65 THEN 'Adulte'
            ELSE 'Senior'
        END AS categorie
    FROM personnes
)
SELECT categorie, COUNT(*)  
FROM personnes_categorisees  
GROUP BY categorie;  
```

### 4. Court-circuit d'évaluation

PostgreSQL évalue les conditions `CASE` dans l'ordre et s'arrête à la première vraie.

**Optimisation :** Placez les conditions les plus fréquentes en premier.

```sql
-- Si 80% des employés sont "Standard"
CASE
    WHEN salaire < 50000 THEN 'Standard'  -- ✅ En premier (80% des cas)
    WHEN salaire > 100000 THEN 'Exceptionnel'
    ELSE 'Élevé'
END
```

### 5. CASE dans des index fonctionnels

Vous pouvez créer des index sur des expressions `CASE`.

```sql
-- Créer un index sur une catégorisation
CREATE INDEX idx_employes_categorie ON employes (
    (CASE
        WHEN salaire < 40000 THEN 'Junior'
        WHEN salaire < 70000 THEN 'Intermédiaire'
        ELSE 'Senior'
    END)
);

-- Requête qui utilisera l'index
SELECT * FROM employes  
WHERE (CASE  
    WHEN salaire < 40000 THEN 'Junior'
    WHEN salaire < 70000 THEN 'Intermédiaire'
    ELSE 'Senior'
END) = 'Senior';
```

---

## Pièges courants et erreurs

### 1. Oublier ELSE et obtenir des NULL

```sql
-- ❌ Peut retourner NULL inattendu
SELECT
    nom,
    CASE
        WHEN salaire > 70000 THEN 'Élevé'
    END AS niveau
FROM employes;
-- Si salaire = 50000 → niveau = NULL

-- ✅ Toujours ajouter ELSE
SELECT
    nom,
    CASE
        WHEN salaire > 70000 THEN 'Élevé'
        ELSE 'Standard'
    END AS niveau
FROM employes;
```

### 2. Mauvais ordre des conditions

```sql
-- ❌ MAUVAIS : les conditions larges en premier
CASE
    WHEN age > 0 THEN 'Positif'    -- Toujours vrai !
    WHEN age < 18 THEN 'Mineur'    -- Jamais atteint
    WHEN age < 65 THEN 'Adulte'    -- Jamais atteint
END

-- ✅ BON : du plus spécifique au plus général
CASE
    WHEN age < 0 THEN 'Invalide'
    WHEN age < 18 THEN 'Mineur'
    WHEN age < 65 THEN 'Adulte'
    ELSE 'Senior'
END
```

### 3. Types de retour incompatibles

Tous les `THEN` et `ELSE` doivent retourner des types compatibles.

```sql
-- ❌ ERREUR : types incompatibles
SELECT
    CASE
        WHEN condition1 THEN 42           -- INTEGER
        WHEN condition2 THEN 'Texte'      -- TEXT
        ELSE NULL
    END;

-- ✅ CORRECT : types homogènes
SELECT
    CASE
        WHEN condition1 THEN '42'         -- TEXT
        WHEN condition2 THEN 'Texte'      -- TEXT
        ELSE NULL::TEXT                   -- TEXT
    END;
```

### 4. Confusion entre CASE simple et recherché

```sql
-- ❌ ERREUR : mélange des syntaxes
CASE age
    WHEN < 18 THEN 'Mineur'  -- Syntaxe invalide !
END

-- ✅ CORRECT : CASE simple (égalité seulement)
CASE age
    WHEN 17 THEN 'Mineur'
    WHEN 25 THEN 'Adulte'
    ELSE 'Autre'
END

-- ✅ CORRECT : CASE recherché (pour les comparaisons)
CASE
    WHEN age < 18 THEN 'Mineur'
    WHEN age < 65 THEN 'Adulte'
    ELSE 'Senior'
END
```

### 5. NULL dans les comparaisons

`NULL` se comporte différemment avec `CASE`.

```sql
-- Comportement de NULL
SELECT
    CASE valeur
        WHEN NULL THEN 'Est NULL'  -- ❌ Ne marche pas !
        ELSE 'Pas NULL'
    END
FROM table;

-- ✅ CORRECT : utiliser IS NULL
SELECT
    CASE
        WHEN valeur IS NULL THEN 'Est NULL'
        ELSE 'Pas NULL'
    END
FROM table;
```

---

## CASE vs autres approches

### CASE vs IF (dans les fonctions)

En PL/pgSQL (fonctions stockées), vous avez aussi `IF` :

```sql
-- Avec CASE (dans une requête SQL)
SELECT CASE WHEN x > 10 THEN 'Grand' ELSE 'Petit' END;

-- Avec IF (dans une fonction PL/pgSQL)
CREATE FUNCTION test(x INTEGER) RETURNS TEXT AS $$  
BEGIN  
    IF x > 10 THEN
        RETURN 'Grand';
    ELSE
        RETURN 'Petit';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

**Utiliser :**
- `CASE` dans les requêtes SQL  
- `IF` dans les fonctions/procédures stockées

### CASE vs FILTER (dans les agrégats)

Pour les agrégations conditionnelles, PostgreSQL offre aussi `FILTER` :

```sql
-- Avec CASE
SELECT
    COUNT(CASE WHEN salaire > 50000 THEN 1 END) AS nb_salaire_eleve
FROM employes;

-- Avec FILTER (plus lisible)
SELECT
    COUNT(*) FILTER (WHERE salaire > 50000) AS nb_salaire_eleve
FROM employes;
```

**Recommandation :** Préférez `FILTER` pour les agrégations conditionnelles.

---

## Exemples avancés

### 1. Calcul de percentiles personnalisés

```sql
WITH stats AS (
    SELECT
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salaire) AS q1,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY salaire) AS mediane,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salaire) AS q3
    FROM employes
)
SELECT
    e.nom,
    e.salaire,
    CASE
        WHEN e.salaire < s.q1 THEN 'Q1 (25% les plus bas)'
        WHEN e.salaire < s.mediane THEN 'Q2 (en-dessous médiane)'
        WHEN e.salaire < s.q3 THEN 'Q3 (au-dessus médiane)'
        ELSE 'Q4 (25% les plus hauts)'
    END AS quartile
FROM employes e  
CROSS JOIN stats s;  
```

### 2. État de machine à états

```sql
-- Workflow de commande
SELECT
    commande_id,
    statut_actuel,
    CASE statut_actuel
        WHEN 'brouillon' THEN
            CASE
                WHEN montant > 0 AND client_valide THEN 'validable'
                ELSE 'incomplet'
            END
        WHEN 'validee' THEN
            CASE
                WHEN paiement_recu THEN 'en_preparation'
                WHEN DATE_PART('day', CURRENT_TIMESTAMP - date_validation) > 7 THEN 'expiree'
                ELSE 'en_attente_paiement'
            END
        WHEN 'en_preparation' THEN
            CASE
                WHEN expedition_date IS NOT NULL THEN 'expediee'
                ELSE 'en_preparation'
            END
        WHEN 'expediee' THEN
            CASE
                WHEN livraison_confirmee THEN 'livree'
                WHEN DATE_PART('day', CURRENT_TIMESTAMP - expedition_date) > 14 THEN 'perdue'
                ELSE 'en_transit'
            END
        ELSE 'etat_inconnu'
    END AS prochain_etat_possible
FROM commandes;
```

### 3. Fenêtre glissante / Rolling average

```sql
-- Classer les performances par rapport à une moyenne mobile
WITH moyennes_mobiles AS (
    SELECT
        date,
        ventes,
        AVG(ventes) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS moyenne_7j
    FROM ventes_quotidiennes
)
SELECT
    date,
    ventes,
    moyenne_7j,
    CASE
        WHEN ventes > moyenne_7j * 1.5 THEN '🚀 Exceptionnel'
        WHEN ventes > moyenne_7j * 1.2 THEN '📈 Très bon'
        WHEN ventes > moyenne_7j THEN '✅ Au-dessus de la moyenne'
        WHEN ventes > moyenne_7j * 0.8 THEN '📊 Proche de la moyenne'
        WHEN ventes > moyenne_7j * 0.5 THEN '📉 En-dessous de la moyenne'
        ELSE '⚠️ Préoccupant'
    END AS performance
FROM moyennes_mobiles  
ORDER BY date DESC;  
```

---

## Bonnes pratiques

### ✅ À faire

1. **Toujours inclure une clause ELSE**
   ```sql
   CASE ... ELSE 'Valeur par défaut' END
   ```

2. **Ordonner les conditions du plus spécifique au plus général**
   ```sql
   CASE
       WHEN condition_rare THEN ...
       WHEN condition_frequente THEN ...
       ELSE ...
   END
   ```

3. **Utiliser COALESCE pour les valeurs par défaut simples**
   ```sql
   COALESCE(colonne, 'Défaut')
   -- Au lieu de : CASE WHEN colonne IS NULL THEN 'Défaut' ELSE colonne END
   ```

4. **Documenter les logiques complexes**
   ```sql
   -- Calcul de score de fidélité selon les règles métier v2.3
   CASE
       WHEN ... THEN ...  -- Règle 1 : Clients VIP
       WHEN ... THEN ...  -- Règle 2 : Nouveaux clients
       ...
   END
   ```

5. **Utiliser des CTE pour éviter la répétition**
   ```sql
   WITH categories AS (
       SELECT *, CASE ... END AS categorie FROM table
   )
   SELECT * FROM categories WHERE categorie = 'X';
   ```

6. **Utiliser FILTER pour les agrégations conditionnelles**
   ```sql
   COUNT(*) FILTER (WHERE condition)
   -- Au lieu de : COUNT(CASE WHEN condition THEN 1 END)
   ```

### ❌ À éviter

1. **CASE sans ELSE (risque de NULL inattendu)**

2. **Conditions dans le mauvais ordre**

3. **Imbrications excessives (> 2 niveaux)**

4. **Types de retour incompatibles**

5. **Répéter des CASE identiques (utiliser CTE)**

6. **CASE dans WHERE si ça empêche l'utilisation d'index**

---

## Résumé

### Points clés à retenir

| Concept | Description |
|---------|-------------|
| **CASE expression** | Logique conditionnelle en SQL (équivalent if/else) |
| **Deux syntaxes** | Simple (égalité) et Recherchée (conditions complexes) |
| **Retourne une valeur** | Utilisable dans SELECT, WHERE, ORDER BY, UPDATE, etc. |
| **ELSE important** | Sans ELSE, retourne NULL si aucune condition vraie |
| **Ordre d'évaluation** | De haut en bas, s'arrête à la première condition vraie |

### Fonctions associées

| Fonction | Usage |
|----------|-------|
| **COALESCE** | Première valeur non-NULL |
| **NULLIF** | Convertir une valeur spécifique en NULL |
| **GREATEST** | Maximum entre plusieurs valeurs |
| **LEAST** | Minimum entre plusieurs valeurs |

### Quand utiliser CASE ?

- ✅ Catégorisation dynamique  
- ✅ Transformation de données  
- ✅ Pivot de données  
- ✅ Logique métier complexe  
- ✅ Agrégations conditionnelles  
- ✅ Tri personnalisé

### Checklist

- [ ] Ai-je inclus une clause ELSE ?  
- [ ] Mes conditions sont-elles dans le bon ordre ?  
- [ ] Les types de retour sont-ils compatibles ?  
- [ ] Ai-je évité les imbrications excessives ?  
- [ ] Pourrais-je utiliser COALESCE à la place ?  
- [ ] Ai-je documenté la logique complexe ?

---

## Pour aller plus loin

- **Window Functions** (Chapitre 10) : Combiner CASE avec des fonctions de fenêtrage  
- **CTE** (Chapitre 9.2) : Éviter la répétition de CASE avec des CTE  
- **PL/pgSQL** (Chapitre 15) : IF/ELSIF/ELSE dans les fonctions stockées  
- **Triggers** (Chapitre 15.4) : CASE dans les déclencheurs

---

**Prochain chapitre :** 10. Fonctions de Fenêtrage (Window Functions)

---


⏭️ [Nouveauté PG 18 : Optimisation des OR-clauses transformées en ANY](/09-techniques-sql-avancees/06-optimisation-or-clauses.md)
