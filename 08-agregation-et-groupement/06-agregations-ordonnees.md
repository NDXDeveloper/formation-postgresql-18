🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.6. Agrégations Ordonnées (WITHIN GROUP, string_agg)

## Introduction : Quand l'Ordre Compte dans les Agrégations

Jusqu'à présent, nous avons utilisé des fonctions d'agrégation où **l'ordre des lignes n'a pas d'importance** :
- `SUM(montant)` : 10 + 20 + 30 = 20 + 30 + 10 (ordre indifférent)
- `AVG(note)` : La moyenne est la même quel que soit l'ordre
- `COUNT(*)` : Le comptage ne dépend pas de l'ordre

Mais certaines opérations **nécessitent un ordre** pour avoir du sens :
- **Calculer la médiane** : Il faut d'abord trier les valeurs !
- **Concaténer des noms** : "Alice, Bob, Charlie" ≠ "Charlie, Alice, Bob"
- **Trouver le mode** (valeur la plus fréquente) : Nécessite un tri par fréquence
- **Créer des listes ordonnées** : Top 3, classement, etc.

PostgreSQL offre des **agrégations ordonnées** pour ces besoins spécifiques.

---

## Deux Types d'Agrégations Ordonnées

PostgreSQL propose deux syntaxes pour les agrégations ordonnées :

### 1. WITHIN GROUP (ORDER BY ...)

Pour les fonctions qui **nécessitent impérativement** un ordre (comme les percentiles).

```sql
fonction_agregation(paramètres) WITHIN GROUP (ORDER BY colonne)
```

### 2. Fonctions avec ORDER BY Direct

Pour les fonctions qui **acceptent optionnellement** un ordre (comme string_agg).

```sql
fonction_agregation(colonne ORDER BY autre_colonne)
```

Nous allons explorer les deux approches en détail.

---

## 1. WITHIN GROUP : Agrégations Hypothétiques

### Qu'est-ce qu'une Agrégation Hypothétique ?

Les **agrégations hypothétiques** (hypothetical-set aggregate functions) calculent une statistique en supposant qu'une valeur hypothétique est ajoutée à l'ensemble de données.

PostgreSQL utilise WITHIN GROUP pour les fonctions qui **requièrent un ordre** pour fonctionner.

### Les Fonctions WITHIN GROUP

| Fonction | Description | Usage |
|----------|-------------|-------|
| **PERCENTILE_CONT** | Percentile continu (interpolé) | Médiane, quartiles |
| **PERCENTILE_DISC** | Percentile discret (valeur réelle) | Médiane, quartiles |
| **MODE** | Valeur la plus fréquente | Trouver la valeur modale |
| **RANK** | Rang d'une valeur hypothétique | Position dans un classement |
| **DENSE_RANK** | Rang dense d'une valeur | Position sans "trous" |
| **PERCENT_RANK** | Rang en pourcentage | Position relative (0-1) |

Nous avons déjà vu PERCENTILE_CONT et PERCENTILE_DISC dans la section 8.2. Voyons les autres.

---

## 2. MODE() : Trouver la Valeur la Plus Fréquente

### Qu'est-ce que le Mode ?

Le **mode** est la valeur qui apparaît le plus souvent dans un ensemble de données.

**Exemple :**
- Dataset : [1, 2, 2, 3, 2, 4, 5]
- Mode : 2 (apparaît 3 fois)

### Syntaxe

```sql
MODE() WITHIN GROUP (ORDER BY colonne)
```

**Note importante** : MODE() prend **zéro paramètre** dans les parenthèses, et la colonne est spécifiée dans ORDER BY.

### Exemple Théorique

Table `commandes` :

| id_commande | client_id | produit      | quantite |
|-------------|-----------|--------------|----------|
| 1           | 101       | Laptop       | 1        |
| 2           | 102       | Souris       | 2        |
| 3           | 103       | Laptop       | 1        |
| 4           | 104       | Clavier      | 1        |
| 5           | 105       | Laptop       | 1        |
| 6           | 106       | Souris       | 2        |

**Question : Quel est le produit le plus commandé ?**

```sql
SELECT
    MODE() WITHIN GROUP (ORDER BY produit) AS produit_le_plus_commande
FROM commandes;
```

**Résultat :**

| produit_le_plus_commande |
|--------------------------|
| Laptop                   |

**Explication :**
- Laptop apparaît 3 fois
- Souris apparaît 2 fois
- Clavier apparaît 1 fois
- Le mode est "Laptop"

### Utilisation avec GROUP BY

```sql
-- Produit le plus commandé par région
SELECT
    region,
    MODE() WITHIN GROUP (ORDER BY produit) AS produit_favori,
    COUNT(*) AS total_commandes
FROM commandes
GROUP BY region;
```

### Cas d'Usage du MODE

**1. E-commerce : Taille la Plus Populaire**

```sql
-- Trouver la taille de vêtement la plus commandée
SELECT
    categorie,
    MODE() WITHIN GROUP (ORDER BY taille) AS taille_populaire,
    COUNT(*) AS nb_ventes
FROM ventes_vetements
GROUP BY categorie;
```

**2. Support Client : Problème le Plus Fréquent**

```sql
-- Type de ticket le plus courant par produit
SELECT
    produit,
    MODE() WITHIN GROUP (ORDER BY type_probleme) AS probleme_principal
FROM tickets_support
GROUP BY produit;
```

**3. Analyse Comportementale : Jour de Visite Préféré**

```sql
-- Jour de la semaine le plus fréquent pour les visites
SELECT
    TO_CHAR(
        MODE() WITHIN GROUP (ORDER BY EXTRACT(DOW FROM date_visite)),
        'Day'
    ) AS jour_plus_frequent
FROM visites_site;
```

---

## 3. STRING_AGG() : Concaténer des Chaînes

### Qu'est-ce que STRING_AGG ?

**STRING_AGG** concatène des valeurs textuelles en une seule chaîne, avec un séparateur personnalisé. C'est l'une des fonctions les plus utiles de PostgreSQL !

### Syntaxe

```sql
STRING_AGG(expression, separateur)

-- Avec ordre :
STRING_AGG(expression, separateur ORDER BY colonne)
```

### Exemple de Base

Table `equipe` :

| id_membre | nom     | poste           |
|-----------|---------|-----------------|
| 1         | Alice   | Développeur     |
| 2         | Bob     | Designer        |
| 3         | Charlie | Chef de projet  |

```sql
-- Concaténer tous les noms
SELECT STRING_AGG(nom, ', ') AS membres
FROM equipe;
```

**Résultat :**

| membres                 |
|-------------------------|
| Alice, Bob, Charlie     |

**Sans ordre garanti** : L'ordre peut varier d'une exécution à l'autre !

### STRING_AGG avec ORDER BY

```sql
-- Concaténer les noms par ordre alphabétique
SELECT STRING_AGG(nom, ', ' ORDER BY nom) AS membres_ordonnes
FROM equipe;
```

**Résultat :**

| membres_ordonnes        |
|-------------------------|
| Alice, Bob, Charlie     |

**Avec ordre garanti** : Les noms sont toujours dans l'ordre alphabétique.

### Exemples Progressifs

#### Exemple 1 : Liste de Produits par Commande

```sql
-- Pour chaque commande, lister tous les produits achetés
SELECT
    id_commande,
    STRING_AGG(produit, ', ' ORDER BY produit) AS produits
FROM lignes_commande
GROUP BY id_commande;
```

**Résultat théorique :**

| id_commande | produits                     |
|-------------|------------------------------|
| 1           | Clavier, Laptop, Souris      |
| 2           | Écran, Laptop                |
| 3           | Souris                       |

#### Exemple 2 : Tags d'Articles de Blog

```sql
-- Concaténer tous les tags pour chaque article
SELECT
    article_id,
    titre,
    STRING_AGG(tag, ' #' ORDER BY tag) AS tags
FROM articles_tags
GROUP BY article_id, titre;
```

**Résultat théorique :**

| article_id | titre                  | tags                           |
|------------|------------------------|--------------------------------|
| 1          | Introduction à SQL     | #database #postgresql #sql     |
| 2          | Les agrégations        | #aggregation #groupby #sql     |

#### Exemple 3 : Historique de Statuts

```sql
-- Tracer l'évolution des statuts d'une commande
SELECT
    id_commande,
    STRING_AGG(
        statut || ' (' || TO_CHAR(date_changement, 'DD/MM HH24:MI') || ')',
        ' → '
        ORDER BY date_changement
    ) AS historique
FROM historique_commandes
GROUP BY id_commande;
```

**Résultat théorique :**

| id_commande | historique                                                    |
|-------------|---------------------------------------------------------------|
| 1           | Créée (15/01 10:30) → Payée (15/01 10:35) → Expédiée (16/01 09:00) → Livrée (18/01 14:20) |

### Séparateurs Personnalisés

STRING_AGG accepte **n'importe quelle chaîne** comme séparateur :

```sql
-- Virgule + espace
STRING_AGG(nom, ', ')

-- Saut de ligne
STRING_AGG(ligne, E'\n')  -- E pour échapper \n

-- HTML (liste à puces)
STRING_AGG('<li>' || nom || '</li>', E'\n')

-- JSON-like
STRING_AGG('"' || nom || '"', ', ')

-- Pipe
STRING_AGG(valeur, ' | ')

-- Aucun séparateur
STRING_AGG(lettre, '')  -- Concatène sans espace
```

### STRING_AGG avec DISTINCT

```sql
-- Concaténer uniquement les valeurs uniques
SELECT
    client_id,
    STRING_AGG(DISTINCT categorie, ', ' ORDER BY categorie) AS categories_achetees
FROM achats
GROUP BY client_id;
```

**Exemple :**
- Achats : Électronique, Vêtements, Électronique, Livres, Vêtements
- Résultat : "Électronique, Livres, Vêtements"

### Gestion des NULL

Par défaut, STRING_AGG **ignore les valeurs NULL**.

```sql
-- Table avec NULL
SELECT STRING_AGG(nom, ', ')
FROM (VALUES ('Alice'), (NULL), ('Bob'), (NULL), ('Charlie')) AS t(nom);

-- Résultat : "Alice, Bob, Charlie"
```

Si vous voulez inclure les NULL comme chaîne :

```sql
SELECT STRING_AGG(COALESCE(nom, '[Inconnu]'), ', ')
FROM equipe;
```

---

## 4. ARRAY_AGG() : Créer des Tableaux

### Qu'est-ce qu'ARRAY_AGG ?

**ARRAY_AGG** agrège des valeurs dans un **tableau PostgreSQL** (type ARRAY) au lieu d'une chaîne.

### Syntaxe

```sql
ARRAY_AGG(expression)

-- Avec ordre :
ARRAY_AGG(expression ORDER BY colonne)
```

### Différence avec STRING_AGG

| Aspect | STRING_AGG | ARRAY_AGG |
|--------|------------|-----------|
| **Type retourné** | TEXT | ARRAY |
| **Séparateur** | Personnalisable | Pas de séparateur (structure native) |
| **Manipulation** | Fonctions texte | Fonctions array |
| **Usage** | Affichage, export | Traitement ultérieur |

### Exemple de Base

```sql
-- Créer un tableau de noms
SELECT ARRAY_AGG(nom ORDER BY nom) AS noms_array
FROM employes;
```

**Résultat :**

| noms_array                     |
|--------------------------------|
| {Alice,Bob,Charlie,Diana}      |

**Format PostgreSQL** : Les arrays sont affichés avec des accolades `{}`.

### Exemples d'Utilisation

#### Exemple 1 : Tags en Tableau

```sql
-- Tags sous forme de tableau
SELECT
    article_id,
    ARRAY_AGG(tag ORDER BY tag) AS tags
FROM articles_tags
GROUP BY article_id;
```

**Résultat :**

| article_id | tags                              |
|------------|-----------------------------------|
| 1          | {database,postgresql,sql}         |
| 2          | {aggregation,groupby,sql}         |

#### Exemple 2 : Historique de Prix

```sql
-- Évolution des prix sous forme de tableau
SELECT
    produit_id,
    ARRAY_AGG(prix ORDER BY date_modification) AS historique_prix
FROM historique_prix
GROUP BY produit_id;
```

**Résultat :**

| produit_id | historique_prix          |
|------------|--------------------------|
| 1          | {99.99,89.99,79.99}      |
| 2          | {149.99,149.99,139.99}   |

### Manipulation des Tableaux

Une fois créé, vous pouvez manipuler l'array :

```sql
-- Accéder au premier élément (index 1 en PostgreSQL !)
SELECT
    produit_id,
    (ARRAY_AGG(prix ORDER BY date_modification))[1] AS prix_initial
FROM historique_prix
GROUP BY produit_id;

-- Compter les éléments
SELECT
    produit_id,
    ARRAY_LENGTH(ARRAY_AGG(prix), 1) AS nb_changements_prix
FROM historique_prix
GROUP BY produit_id;

-- Vérifier si une valeur existe
SELECT
    article_id,
    'sql' = ANY(ARRAY_AGG(tag)) AS contient_sql
FROM articles_tags
GROUP BY article_id;
```

### ARRAY_AGG avec FILTER

Combinaison puissante !

```sql
-- Créer un array uniquement avec les produits chers
SELECT
    categorie,
    ARRAY_AGG(nom ORDER BY prix DESC) FILTER (WHERE prix > 100) AS produits_premium
FROM produits
GROUP BY categorie;
```

---

## 5. JSON_AGG() et JSONB_AGG() : Agrégations JSON

### Créer des Structures JSON

PostgreSQL peut agréger des lignes directement en JSON !

```sql
-- Agréger en JSON
SELECT JSON_AGG(
    JSON_BUILD_OBJECT(
        'nom', nom,
        'email', email,
        'role', role
    )
) AS employes_json
FROM employes;
```

**Résultat :**

```json
[
  {"nom": "Alice", "email": "alice@example.com", "role": "Dev"},
  {"nom": "Bob", "email": "bob@example.com", "role": "Designer"}
]
```

### JSON_AGG avec ORDER BY

```sql
-- JSON ordonné par nom
SELECT JSON_AGG(
    JSON_BUILD_OBJECT('nom', nom, 'note', note)
    ORDER BY note DESC
) AS top_etudiants
FROM etudiants
LIMIT 10;
```

### JSONB_AGG : JSON Binaire

```sql
-- Version binaire (plus performante pour traitement ultérieur)
SELECT JSONB_AGG(
    JSONB_BUILD_OBJECT('produit', produit, 'prix', prix)
    ORDER BY prix
) AS catalogue_jsonb
FROM produits;
```

### Cas d'Usage : API Response

```sql
-- Générer directement une réponse API
SELECT JSON_BUILD_OBJECT(
    'commande_id', c.id_commande,
    'client', c.nom_client,
    'total', c.montant_total,
    'produits', (
        SELECT JSON_AGG(
            JSON_BUILD_OBJECT(
                'nom', l.produit,
                'quantite', l.quantite,
                'prix', l.prix_unitaire
            )
            ORDER BY l.ligne_numero
        )
        FROM lignes_commande l
        WHERE l.id_commande = c.id_commande
    )
) AS commande_json
FROM commandes c
WHERE c.id_commande = 12345;
```

**Résultat :**

```json
{
  "commande_id": 12345,
  "client": "Jean Dupont",
  "total": 299.97,
  "produits": [
    {"nom": "Laptop", "quantite": 1, "prix": 899.99},
    {"nom": "Souris", "quantite": 2, "prix": 25.00}
  ]
}
```

---

## 6. Autres Fonctions d'Agrégation Ordonnées

### XMLAGG() : Agrégation XML

```sql
-- Créer du XML
SELECT XMLAGG(
    XMLELEMENT(NAME "produit",
        XMLELEMENT(NAME "nom", nom),
        XMLELEMENT(NAME "prix", prix)
    ) ORDER BY prix
) AS catalogue_xml
FROM produits;
```

**Résultat :**

```xml
<produit><nom>Souris</nom><prix>25</prix></produit>
<produit><nom>Clavier</nom><prix>75</prix></produit>
```

### Fonctions Hypothétiques de Rang

Ces fonctions calculent le rang d'une **valeur hypothétique** si elle était insérée dans l'ensemble.

#### RANK() WITHIN GROUP

```sql
-- À quel rang serait un étudiant avec une note de 15 ?
SELECT
    RANK(15) WITHIN GROUP (ORDER BY note DESC) AS rang_si_note_15
FROM etudiants;
```

#### PERCENT_RANK() WITHIN GROUP

```sql
-- Quel serait le percentile d'une vente de 500€ ?
SELECT
    PERCENT_RANK(500) WITHIN GROUP (ORDER BY montant) AS percentile_500
FROM ventes;
-- Retourne 0.75 = 75e percentile
```

---

## Cas d'Usage Réels par Domaine

### 📊 E-commerce : Wishlist et Panier

```sql
-- Construire une wishlist avec produits ordonnés par priorité
SELECT
    user_id,
    STRING_AGG(
        produit_nom || ' (' || prix || '€)',
        ', '
        ORDER BY priorite DESC, date_ajout
    ) AS wishlist,
    ARRAY_AGG(produit_id ORDER BY priorite DESC) AS wishlist_ids,
    COUNT(*) AS nb_produits
FROM wishlist
GROUP BY user_id;
```

**Résultat théorique :**

| user_id | wishlist                                      | wishlist_ids | nb_produits |
|---------|-----------------------------------------------|--------------|-------------|
| 101     | Laptop (899€), Écran (250€), Souris (25€)    | {12,45,78}   | 3           |

### 🏥 Santé : Historique Médical

```sql
-- Résumé du parcours de soin d'un patient
SELECT
    patient_id,
    STRING_AGG(
        date_consultation::TEXT || ': ' || diagnostic || ' (Dr. ' || medecin || ')',
        E'\n'
        ORDER BY date_consultation
    ) AS historique_medical,
    ARRAY_AGG(DISTINCT medicament ORDER BY medicament) AS medicaments_prescrits
FROM consultations
WHERE patient_id = 12345
GROUP BY patient_id;
```

**Résultat :**

```
Historique :
2024-01-15: Grippe (Dr. Martin)
2024-03-22: Contrôle (Dr. Dubois)
2024-06-10: Allergie (Dr. Martin)

Médicaments : {Aspirine, Cétirizine, Paracétamol}
```

### 💰 Finance : Transactions Groupées

```sql
-- Résumé mensuel des transactions par compte
SELECT
    compte_id,
    DATE_TRUNC('month', date_transaction) AS mois,
    COUNT(*) AS nb_transactions,
    SUM(montant) AS solde_variation,
    STRING_AGG(
        TO_CHAR(date_transaction, 'DD/MM') || ': ' ||
        CASE WHEN montant > 0 THEN '+' ELSE '' END ||
        montant::TEXT || '€ (' || libelle || ')',
        E'\n'
        ORDER BY date_transaction
    ) AS detail_transactions
FROM transactions
WHERE date_transaction >= '2024-01-01'
GROUP BY compte_id, DATE_TRUNC('month', date_transaction)
ORDER BY compte_id, mois;
```

### 🌐 SaaS : Logs et Événements

```sql
-- Tracer le parcours utilisateur (session tracking)
SELECT
    session_id,
    user_id,
    STRING_AGG(
        page_url,
        ' → '
        ORDER BY timestamp
    ) AS parcours,
    EXTRACT(EPOCH FROM (MAX(timestamp) - MIN(timestamp)))::INT AS duree_session_sec,
    COUNT(*) AS nb_pages_vues
FROM page_views
WHERE date_visite = CURRENT_DATE
GROUP BY session_id, user_id
HAVING COUNT(*) > 5  -- Sessions actives uniquement
ORDER BY duree_session_sec DESC;
```

**Résultat théorique :**

| session_id | user_id | parcours                                      | duree_session_sec | nb_pages_vues |
|------------|---------|-----------------------------------------------|-------------------|---------------|
| abc123     | 42      | /home → /produits → /produit/12 → /panier → /checkout | 482        | 5             |

### 📱 Réseaux Sociaux : Agrégation de Commentaires

```sql
-- Top commentaires d'un post avec leurs auteurs
SELECT
    post_id,
    STRING_AGG(
        '@' || username || ': ' ||
        CASE
            WHEN LENGTH(commentaire) > 50
            THEN LEFT(commentaire, 50) || '...'
            ELSE commentaire
        END,
        E'\n---\n'
        ORDER BY nb_likes DESC, date_commentaire
    ) AS top_commentaires
FROM commentaires
WHERE post_id = 789
GROUP BY post_id
LIMIT 1;
```

### 🎓 Éducation : Bulletin Scolaire

```sql
-- Générer un bulletin par étudiant
SELECT
    e.etudiant_id,
    e.nom,
    e.prenom,
    ROUND(AVG(n.note), 2) AS moyenne_generale,
    STRING_AGG(
        m.nom_matiere || ': ' || n.note || '/20 (' ||
        CASE
            WHEN n.note >= 16 THEN 'Excellent'
            WHEN n.note >= 14 THEN 'Très bien'
            WHEN n.note >= 12 THEN 'Bien'
            WHEN n.note >= 10 THEN 'Assez bien'
            ELSE 'Insuffisant'
        END || ')',
        E'\n'
        ORDER BY m.nom_matiere
    ) AS notes_detaillees,
    JSON_AGG(
        JSON_BUILD_OBJECT(
            'matiere', m.nom_matiere,
            'note', n.note,
            'coefficient', m.coefficient
        )
        ORDER BY m.nom_matiere
    ) AS notes_json
FROM etudiants e
JOIN notes n ON e.etudiant_id = n.etudiant_id
JOIN matieres m ON n.matiere_id = m.matiere_id
WHERE e.etudiant_id = 12345
  AND n.semestre = 1
  AND n.annee_scolaire = '2024-2025'
GROUP BY e.etudiant_id, e.nom, e.prenom;
```

---

## STRING_AGG vs ARRAY_AGG : Quand Utiliser Quoi ?

### Tableau Comparatif

| Critère | STRING_AGG | ARRAY_AGG |
|---------|------------|-----------|
| **Type de sortie** | TEXT (chaîne) | ARRAY (tableau natif) |
| **Affichage** | ✅ Direct (lisible) | ⚠️ Nécessite conversion |
| **Export (CSV, JSON)** | ✅ Simple | ⚠️ Nécessite formatage |
| **Traitement ultérieur** | ❌ Parse requis | ✅ Manipulation native |
| **Performance** | ✅ Léger | ⚠️ Plus lourd en mémoire |
| **Séparateur** | ✅ Personnalisable | ❌ Structure fixe |
| **Indexation** | ❌ Impossible | ✅ Possible (GIN) |
| **Longueur maximale** | Limitée (1GB par champ) | Limitée (nombre éléments) |

### Recommandations

**Utilisez STRING_AGG si :**
- ✅ Résultat destiné à l'affichage ou export
- ✅ Rapport pour humains
- ✅ Export CSV, logs, emails
- ✅ Concaténation simple

**Utilisez ARRAY_AGG si :**
- ✅ Traitement ultérieur en SQL
- ✅ Vérifications (contient X ?)
- ✅ Indexation nécessaire (recherche)
- ✅ Application manipule les arrays

### Exemple de Décision

```sql
-- Pour un rapport PDF/Excel : STRING_AGG
SELECT
    client_id,
    nom,
    STRING_AGG(produit, ', ' ORDER BY date_achat DESC) AS historique_achats
FROM achats
GROUP BY client_id, nom;

-- Pour traitement applicatif : ARRAY_AGG
SELECT
    client_id,
    nom,
    ARRAY_AGG(produit_id ORDER BY date_achat DESC) AS historique_produit_ids
FROM achats
GROUP BY client_id, nom;
```

---

## Performance et Optimisation

### Coût des Agrégations Ordonnées

Les agrégations avec ORDER BY nécessitent un **tri**, ce qui a un coût :

```sql
-- Sans ordre : plus rapide
STRING_AGG(nom, ', ')

-- Avec ordre : nécessite tri
STRING_AGG(nom, ', ' ORDER BY nom)
```

### Optimisations

#### 1. Index sur les Colonnes de Tri

```sql
-- Si vous triez souvent par date_ajout :
CREATE INDEX idx_wishlist_user_date ON wishlist(user_id, date_ajout);

-- STRING_AGG pourra utiliser l'index
SELECT
    user_id,
    STRING_AGG(produit_nom, ', ' ORDER BY date_ajout)
FROM wishlist
GROUP BY user_id;
```

#### 2. Limiter le Nombre d'Éléments

```sql
-- Limiter avant d'agréger (plus efficace)
WITH top_produits AS (
    SELECT produit_nom
    FROM wishlist
    WHERE user_id = 12345
    ORDER BY priorite DESC
    LIMIT 10
)
SELECT STRING_AGG(produit_nom, ', ') AS top_10
FROM top_produits;

-- vs agréger tout puis tronquer (moins efficace)
```

#### 3. Matérialiser les Agrégations Fréquentes

```sql
-- Table de résumé mise à jour périodiquement
CREATE TABLE produits_tags_cache AS
SELECT
    produit_id,
    STRING_AGG(tag, ', ' ORDER BY tag) AS tags_str,
    ARRAY_AGG(tag ORDER BY tag) AS tags_array
FROM produits_tags
GROUP BY produit_id;

CREATE INDEX idx_produits_tags_cache ON produits_tags_cache(produit_id);
```

#### 4. Attention à la Longueur des Chaînes

```sql
-- Peut générer des chaînes très longues !
SELECT STRING_AGG(description_longue, '\n')
FROM articles;

-- Tronquer si nécessaire :
SELECT STRING_AGG(
    CASE
        WHEN LENGTH(description_longue) > 100
        THEN LEFT(description_longue, 100) || '...'
        ELSE description_longue
    END,
    '\n'
)
FROM articles;
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Toujours spécifier ORDER BY quand l'ordre importe**
   ```sql
   -- ✅ Ordre explicite
   STRING_AGG(nom, ', ' ORDER BY nom)

   -- ❌ Ordre non déterministe
   STRING_AGG(nom, ', ')
   ```

2. **Utiliser DISTINCT pour éliminer les doublons**
   ```sql
   STRING_AGG(DISTINCT categorie, ', ' ORDER BY categorie)
   ```

3. **Gérer les NULL explicitement**
   ```sql
   STRING_AGG(COALESCE(nom, '[Inconnu]'), ', ')
   ```

4. **Choisir le bon type de séparateur**
   ```sql
   -- Pour affichage lisible
   STRING_AGG(nom, ', ')

   -- Pour traitement ultérieur
   STRING_AGG(nom, '|')  -- Séparateur rare
   ```

5. **Limiter la longueur si nécessaire**
   ```sql
   STRING_AGG(
       CASE WHEN LENGTH(texte) > 50 THEN LEFT(texte, 50) || '...' ELSE texte END,
       '\n'
   )
   ```

6. **Documenter le séparateur choisi**
   ```sql
   -- Séparateur : virgule + espace (pour CSV)
   STRING_AGG(valeur, ', ' ORDER BY date)
   ```

### ❌ À Éviter

1. **Oublier ORDER BY quand l'ordre compte**
   ```sql
   -- ❌ Historique dans un ordre aléatoire
   STRING_AGG(statut, ' → ')

   -- ✅ Historique chronologique
   STRING_AGG(statut, ' → ' ORDER BY date_changement)
   ```

2. **Concaténer sans limite sur de gros volumes**
   ```sql
   -- ❌ Peut générer une chaîne de plusieurs MB !
   STRING_AGG(log_message, '\n')

   -- ✅ Limiter d'abord
   SELECT STRING_AGG(log_message, '\n')
   FROM (
       SELECT log_message
       FROM logs
       ORDER BY timestamp DESC
       LIMIT 100
   ) recent_logs;
   ```

3. **Utiliser STRING_AGG pour des calculs**
   ```sql
   -- ❌ Difficile à parser
   STRING_AGG(montant::TEXT, ',')

   -- ✅ Utiliser ARRAY_AGG
   ARRAY_AGG(montant ORDER BY date)
   ```

4. **Oublier de grouper correctement**
   ```sql
   -- ❌ Erreur si plusieurs groupes attendus
   SELECT STRING_AGG(nom, ', ')
   FROM employes;
   -- Retourne TOUS les noms en une ligne

   -- ✅ Grouper par département
   SELECT
       departement,
       STRING_AGG(nom, ', ' ORDER BY nom)
   FROM employes
   GROUP BY departement;
   ```

---

## Comparaison avec les Window Functions

### Confusion Fréquente

Les débutants confondent souvent les agrégations ordonnées (WITHIN GROUP) avec les window functions (OVER).

### Différences Fondamentales

| Aspect | Agrégations Ordonnées | Window Functions |
|--------|----------------------|------------------|
| **Syntaxe** | `WITHIN GROUP (ORDER BY ...)` | `OVER (ORDER BY ...)` |
| **Résultat** | Une ligne par groupe | Une ligne par ligne source |
| **GROUP BY** | Nécessaire (sauf total global) | Optionnel |
| **Usage** | Créer des listes, calculs sur ensemble trié | Calculs par fenêtre glissante |

### Exemple Comparatif

**Avec Agrégation Ordonnée (STRING_AGG) :**

```sql
SELECT
    departement,
    STRING_AGG(nom, ', ' ORDER BY salaire DESC) AS employes_par_salaire
FROM employes
GROUP BY departement;
```

**Résultat :** Une ligne par département

| departement | employes_par_salaire         |
|-------------|------------------------------|
| IT          | Alice, Bob, Charlie          |
| Marketing   | Diana, Eve                   |

**Avec Window Function (ROW_NUMBER) :**

```sql
SELECT
    departement,
    nom,
    salaire,
    ROW_NUMBER() OVER (PARTITION BY departement ORDER BY salaire DESC) AS rang
FROM employes;
```

**Résultat :** Une ligne par employé

| departement | nom     | salaire | rang |
|-------------|---------|---------|------|
| IT          | Alice   | 70000   | 1    |
| IT          | Bob     | 65000   | 2    |
| IT          | Charlie | 60000   | 3    |
| Marketing   | Diana   | 55000   | 1    |
| Marketing   | Eve     | 50000   | 2    |

**Les window functions seront détaillées dans la section 10 !**

---

## À Retenir

1. **WITHIN GROUP (ORDER BY)** est utilisé pour les fonctions nécessitant un ordre (MODE, PERCENTILE, etc.)
2. **STRING_AGG** concatène des valeurs textuelles avec un séparateur personnalisé
3. **ARRAY_AGG** crée un tableau natif PostgreSQL pour traitement ultérieur
4. **JSON_AGG/JSONB_AGG** génèrent des structures JSON directement
5. **ORDER BY dans STRING_AGG** garantit un ordre déterministe
6. **DISTINCT** élimine les doublons avant agrégation
7. Les agrégations ordonnées sont **idéales pour les rapports et exports**
8. **STRING_AGG** pour affichage, **ARRAY_AGG** pour traitement
9. Toujours spécifier **ORDER BY** quand l'ordre importe
10. Ne pas confondre avec les **window functions** (section 10)

---

## Prochaines Étapes

Vous avez terminé la section 8 sur les agrégations ! Prochains sujets majeurs :

- **Section 9** : Techniques SQL Avancées (Sous-requêtes, CTEs, récursion)
- **Section 10** : Window Functions (Fonctions de fenêtrage) - Le summum de l'analyse SQL !
- **Section 11** : Modélisation Avancée (Normalisation, partitionnement, vues)

Les **agrégations ordonnées** sont des outils puissants pour générer des rapports lisibles, construire des listes, et structurer les données pour l'export. STRING_AGG en particulier est l'une des fonctions les plus utilisées dans les requêtes de production PostgreSQL !

⏭️ [Techniques SQL Avancées](/09-techniques-sql-avancees/README.md)
