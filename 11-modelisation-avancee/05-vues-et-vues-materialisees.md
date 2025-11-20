🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.5. Vues et Vues Matérialisées : Refresh et indexation

## Introduction

Les **vues** sont l'un des outils les plus puissants de PostgreSQL pour simplifier l'accès aux données et améliorer la maintenabilité de votre base de données. Elles permettent de créer des "fenêtres" personnalisées sur vos données, cachant la complexité des requêtes SQL sous-jacentes.

PostgreSQL propose **deux types de vues** :

1. **Vues classiques (VIEWS)** : Requêtes nommées qui s'exécutent à chaque utilisation
2. **Vues matérialisées (MATERIALIZED VIEWS)** : Résultats pré-calculés et stockés physiquement

### Analogie Simple

Imaginez un rapport de ventes mensuel dans une entreprise :

**Vue classique (VIEW) :**
```
┌─────────────────────────────────────┐
│  Demande du rapport                 │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Calcul en temps réel               │
│  - Interroger les ventes            │
│  - Calculer les totaux              │
│  - Grouper par mois                 │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Résultat (toujours à jour)         │
└─────────────────────────────────────┘
```

**Vue matérialisée (MATERIALIZED VIEW) :**
```
┌─────────────────────────────────────┐
│  Rapport pré-calculé (snapshot)     │
│  Calculé hier à 23h00               │
│  Stocké et prêt à l'emploi          │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Demande du rapport                 │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Résultat instantané                │
│  (mais potentiellement pas à jour)  │
└─────────────────────────────────────┘
```

---

## Partie 1 : Vues Classiques (VIEWS)

### Qu'est-ce qu'une Vue ?

Une **vue** est une **requête SQL nommée et stockée** dans la base de données. Elle se comporte comme une table virtuelle : vous pouvez la sélectionner, mais elle n'existe pas physiquement. À chaque interrogation, PostgreSQL exécute la requête sous-jacente.

### Syntaxe de Base

```sql
CREATE VIEW nom_vue AS
SELECT ...
FROM ...
WHERE ...;
```

### Exemple Simple

```sql
-- Tables de base
CREATE TABLE employes (
    employe_id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    departement_id INTEGER,
    salaire NUMERIC(10,2),
    date_embauche DATE
);

CREATE TABLE departements (
    departement_id SERIAL PRIMARY KEY,
    nom_departement VARCHAR(100),
    localisation VARCHAR(100)
);

-- Insérer des données
INSERT INTO departements (nom_departement, localisation) VALUES
    ('Informatique', 'Paris'),
    ('Comptabilité', 'Lyon'),
    ('Ressources Humaines', 'Marseille');

INSERT INTO employes (nom, prenom, departement_id, salaire, date_embauche) VALUES
    ('Dupont', 'Alice', 1, 45000, '2020-01-15'),
    ('Martin', 'Bob', 1, 52000, '2019-06-20'),
    ('Dubois', 'Claire', 2, 48000, '2021-03-10'),
    ('Lefebvre', 'David', 2, 51000, '2018-11-05'),
    ('Rousseau', 'Emma', 3, 43000, '2022-05-18');

-- Créer une vue
CREATE VIEW v_employes_complet AS
SELECT
    e.employe_id,
    e.prenom || ' ' || e.nom AS nom_complet,
    e.salaire,
    e.date_embauche,
    d.nom_departement,
    d.localisation
FROM employes e
JOIN departements d ON e.departement_id = d.departement_id;
```

### Utilisation de la Vue

```sql
-- Interroger la vue comme une table
SELECT * FROM v_employes_complet;
```

**Résultat :**
```
employe_id | nom_complet    | salaire  | date_embauche | nom_departement        | localisation
-----------+----------------+----------+---------------+------------------------+-------------
1          | Alice Dupont   | 45000.00 | 2020-01-15    | Informatique           | Paris
2          | Bob Martin     | 52000.00 | 2019-06-20    | Informatique           | Paris
3          | Claire Dubois  | 48000.00 | 2021-03-10    | Comptabilité           | Lyon
4          | David Lefebvre | 51000.00 | 2018-11-05    | Comptabilité           | Lyon
5          | Emma Rousseau  | 43000.00 | 2022-05-18    | Ressources Humaines    | Marseille
```

**Ce qui se passe :**
Chaque fois que vous faites `SELECT * FROM v_employes_complet`, PostgreSQL exécute la requête de jointure sous-jacente. Les données sont **toujours à jour**.

### Avantages des Vues Classiques

#### 1. Simplification des Requêtes

**Sans vue :**
```sql
-- Requête complexe à répéter partout dans l'application
SELECT
    e.employe_id,
    e.prenom || ' ' || e.nom AS nom_complet,
    e.salaire,
    d.nom_departement,
    d.localisation
FROM employes e
JOIN departements d ON e.departement_id = d.departement_id
WHERE e.salaire > 45000
ORDER BY e.salaire DESC;
```

**Avec vue :**
```sql
-- Requête simplifiée
SELECT * FROM v_employes_complet
WHERE salaire > 45000
ORDER BY salaire DESC;
```

#### 2. Abstraction et Encapsulation

Cacher la complexité de la structure sous-jacente :

```sql
-- Vue qui masque la complexité d'une jointure multiple
CREATE VIEW v_commandes_detaillees AS
SELECT
    c.commande_id,
    c.date_commande,
    cl.nom AS client_nom,
    p.nom_produit,
    cd.quantite,
    cd.prix_unitaire,
    cd.quantite * cd.prix_unitaire AS montant_ligne
FROM commandes c
JOIN clients cl ON c.client_id = cl.client_id
JOIN commandes_details cd ON c.commande_id = cd.commande_id
JOIN produits p ON cd.produit_id = p.produit_id;
```

Les développeurs utilisent simplement `v_commandes_detaillees` sans connaître la structure interne.

#### 3. Sécurité et Contrôle d'Accès

Limiter l'accès à certaines colonnes ou lignes :

```sql
-- Vue qui masque les informations sensibles
CREATE VIEW v_employes_public AS
SELECT
    employe_id,
    prenom,
    nom,
    nom_departement,
    localisation
FROM v_employes_complet;
-- La colonne 'salaire' n'est pas exposée

-- Donner accès uniquement à la vue
GRANT SELECT ON v_employes_public TO role_lecture;
REVOKE SELECT ON employes FROM role_lecture;
```

#### 4. Compatibilité lors de Migrations

Maintenir une interface stable même si la structure change :

```sql
-- Ancien schéma : colonne 'nom_complet'
-- Nouveau schéma : colonnes séparées 'prenom' et 'nom'

-- Vue pour compatibilité ascendante
CREATE VIEW ancienne_interface AS
SELECT
    employe_id,
    prenom || ' ' || nom AS nom_complet,
    salaire
FROM employes;
```

Les anciennes applications continuent de fonctionner via la vue.

### Inconvénients des Vues Classiques

#### 1. Performance : Recalcul à Chaque Requête

```sql
-- Vue avec agrégation coûteuse
CREATE VIEW v_stats_ventes AS
SELECT
    DATE_TRUNC('month', date_vente) AS mois,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total,
    AVG(montant) AS montant_moyen
FROM ventes
GROUP BY DATE_TRUNC('month', date_vente);

-- Chaque SELECT sur cette vue recalcule tout
SELECT * FROM v_stats_ventes;  -- Peut être lent si 'ventes' est grosse
```

Si la table `ventes` contient 10 millions de lignes, chaque requête sur `v_stats_ventes` recalcule l'agrégation.

#### 2. Pas d'Indexation Directe

Impossible de créer un index directement sur une vue classique.

```sql
-- ❌ Erreur
CREATE INDEX idx_vue ON v_employes_complet(salaire);
-- ERROR: cannot create index on view "v_employes_complet"
```

**Workaround :** Créer des index sur les tables sous-jacentes.

### Vues Modifiables (Updatable Views)

Certaines vues simples peuvent être **modifiables** : vous pouvez faire INSERT/UPDATE/DELETE dessus.

**Conditions pour qu'une vue soit modifiable :**
- La vue interroge **une seule table** (pas de JOIN)
- Pas de DISTINCT, GROUP BY, HAVING, WINDOW functions
- Pas de WITH, UNION, INTERSECT, EXCEPT
- Pas de fonctions d'agrégation

```sql
-- Vue simple (modifiable)
CREATE VIEW v_employes_it AS
SELECT employe_id, nom, prenom, salaire
FROM employes
WHERE departement_id = 1;

-- INSERT sur la vue fonctionne
INSERT INTO v_employes_it (nom, prenom, salaire)
VALUES ('Nouveau', 'Jean', 50000);

-- UPDATE sur la vue fonctionne
UPDATE v_employes_it SET salaire = salaire * 1.05;

-- DELETE sur la vue fonctionne
DELETE FROM v_employes_it WHERE employe_id = 10;
```

**Note :** Les modifications sont appliquées à la table sous-jacente (`employes`).

#### Vues Non-Modifiables avec INSTEAD OF Triggers

Pour les vues complexes, utilisez des triggers `INSTEAD OF` :

```sql
-- Vue complexe (non-modifiable naturellement)
CREATE VIEW v_commandes_details_vue AS
SELECT
    c.commande_id,
    c.date_commande,
    cl.nom AS client_nom,
    p.nom_produit,
    cd.quantite
FROM commandes c
JOIN clients cl ON c.client_id = cl.client_id
JOIN commandes_details cd ON c.commande_id = cd.commande_id
JOIN produits p ON cd.produit_id = p.produit_id;

-- Trigger INSTEAD OF pour gérer les INSERT
CREATE OR REPLACE FUNCTION insert_commande_detail()
RETURNS TRIGGER AS $$
BEGIN
    -- Logique personnalisée d'insertion
    -- ...
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_insert_commande
INSTEAD OF INSERT ON v_commandes_details_vue
FOR EACH ROW EXECUTE FUNCTION insert_commande_detail();
```

---

## Partie 2 : Vues Matérialisées (MATERIALIZED VIEWS)

### Qu'est-ce qu'une Vue Matérialisée ?

Une **vue matérialisée** est une vue dont les **résultats sont stockés physiquement** sur le disque. Contrairement à une vue classique, elle ne recalcule pas à chaque requête : elle retourne les données pré-calculées.

### Différences Clés : Vue vs Vue Matérialisée

| Critère | Vue Classique | Vue Matérialisée |
|---------|--------------|------------------|
| **Stockage** | ❌ Aucun (requête virtuelle) | ✅ Physique sur disque |
| **Performance** | ⚠️ Recalcul à chaque requête | ✅ Lecture directe (rapide) |
| **Fraîcheur** | ✅ Toujours à jour | ⚠️ Snapshot (peut être obsolète) |
| **Refresh** | ❌ Automatique | ⚠️ Manuel (REFRESH) |
| **Indexation** | ❌ Impossible | ✅ Possible |
| **Espace disque** | ✅ Aucun | ❌ Occupe de l'espace |
| **Mise à jour** | ❌ Impossibles (sauf triggers) | ❌ Impossibles |

### Syntaxe de Base

```sql
CREATE MATERIALIZED VIEW nom_vue_materialisee AS
SELECT ...
FROM ...
WHERE ...;
```

### Exemple Simple

```sql
-- Vue matérialisée des statistiques de ventes
CREATE MATERIALIZED VIEW vm_stats_ventes_mensuelles AS
SELECT
    DATE_TRUNC('month', date_vente) AS mois,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total,
    AVG(montant) AS montant_moyen,
    MIN(montant) AS montant_min,
    MAX(montant) AS montant_max
FROM ventes
GROUP BY DATE_TRUNC('month', date_vente)
ORDER BY mois;
```

**Ce qui se passe :**
1. PostgreSQL exécute la requête **une seule fois**
2. Les résultats sont **stockés physiquement** dans `vm_stats_ventes_mensuelles`
3. Les requêtes sur la vue matérialisée lisent directement ces résultats

### Interrogation

```sql
-- Lecture ultra-rapide (données pré-calculées)
SELECT * FROM vm_stats_ventes_mensuelles
WHERE mois >= '2024-01-01';
```

**Performance :**
- Vue classique : 5 secondes (agrégation sur 10M de lignes)
- Vue matérialisée : 10 millisecondes (lecture directe)

**Gain : 500× plus rapide !**

### Quand Utiliser des Vues Matérialisées ?

#### ✅ Cas d'Usage Idéaux

**1. Agrégations Coûteuses sur Grandes Tables**

```sql
-- Calculs complexes qui prennent des minutes
CREATE MATERIALIZED VIEW vm_dashboard_executif AS
SELECT
    DATE_TRUNC('day', created_at) AS jour,
    COUNT(DISTINCT user_id) AS utilisateurs_actifs,
    COUNT(DISTINCT session_id) AS sessions,
    SUM(revenue) AS revenue_journalier,
    AVG(session_duration) AS duree_moyenne_session,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY page_load_time) AS p95_load_time
FROM evenements
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY DATE_TRUNC('day', created_at);

-- Rafraîchir une fois par jour (par exemple à minuit)
```

**2. Rapports et Tableaux de Bord**

```sql
-- Rapport mensuel complexe
CREATE MATERIALIZED VIEW vm_rapport_ventes AS
SELECT
    v.mois,
    v.region,
    v.categorie_produit,
    COUNT(DISTINCT v.vendeur_id) AS nb_vendeurs,
    SUM(v.quantite) AS quantite_totale,
    SUM(v.montant) AS ca_total,
    SUM(v.montant) / COUNT(DISTINCT v.commande_id) AS panier_moyen,
    RANK() OVER (PARTITION BY v.region ORDER BY SUM(v.montant) DESC) AS rang_categorie
FROM ventes_enrichies v
GROUP BY v.mois, v.region, v.categorie_produit;
```

Rafraîchir une fois par jour ou semaine selon les besoins.

**3. Jointures Complexes et Coûteuses**

```sql
-- Données dénormalisées pour analytics
CREATE MATERIALIZED VIEW vm_commandes_enrichies AS
SELECT
    c.commande_id,
    c.date_commande,
    cl.client_id,
    cl.nom AS client_nom,
    cl.segment AS client_segment,
    cl.pays AS client_pays,
    p.produit_id,
    p.nom_produit,
    p.categorie,
    cd.quantite,
    cd.prix_unitaire,
    cd.quantite * cd.prix_unitaire AS montant_ligne,
    f.montant_fidelite,
    f.points_accumules
FROM commandes c
JOIN clients cl ON c.client_id = cl.client_id
JOIN commandes_details cd ON c.commande_id = cd.commande_id
JOIN produits p ON cd.produit_id = p.produit_id
LEFT JOIN programme_fidelite f ON cl.client_id = f.client_id;
```

**4. Données de Référence Peu Volatiles**

```sql
-- Catalogue produits enrichi (change rarement)
CREATE MATERIALIZED VIEW vm_catalogue_complet AS
SELECT
    p.produit_id,
    p.nom_produit,
    p.description,
    c.nom_categorie,
    c.departement,
    f.nom_fournisseur,
    f.pays_origine,
    i.stock_disponible,
    i.entrepot_principal,
    AVG(r.note) AS note_moyenne,
    COUNT(r.review_id) AS nb_avis
FROM produits p
JOIN categories c ON p.categorie_id = c.categorie_id
JOIN fournisseurs f ON p.fournisseur_id = f.fournisseur_id
LEFT JOIN inventaire i ON p.produit_id = i.produit_id
LEFT JOIN reviews r ON p.produit_id = r.produit_id
GROUP BY p.produit_id, p.nom_produit, p.description,
         c.nom_categorie, c.departement, f.nom_fournisseur,
         f.pays_origine, i.stock_disponible, i.entrepot_principal;
```

Rafraîchir chaque nuit.

#### ❌ Quand NE PAS Utiliser

**1. Données en Temps Réel**

Si vous avez besoin des données **immédiatement à jour** (ex: solde bancaire, stock en temps réel), les vues matérialisées ne conviennent pas.

**2. Tables Petites ou Requêtes Rapides**

Si la requête prend < 100ms, une vue classique suffit. L'overhead du refresh n'est pas justifié.

**3. Données Très Volatiles**

Si les données changent constamment et nécessitent un refresh toutes les minutes, l'intérêt est limité.

---

## Partie 3 : Refresh des Vues Matérialisées

### Le Problème : Données Obsolètes

Les vues matérialisées contiennent un **snapshot** (instantané) des données au moment du dernier refresh.

```sql
-- État initial
CREATE MATERIALIZED VIEW vm_compteur_ventes AS
SELECT COUNT(*) AS total_ventes FROM ventes;

SELECT * FROM vm_compteur_ventes;
-- Résultat: total_ventes = 10000

-- Nouvelles ventes insérées
INSERT INTO ventes (...) VALUES (...);  -- 100 nouvelles ventes

-- La vue matérialisée n'est PAS mise à jour automatiquement
SELECT * FROM vm_compteur_ventes;
-- Résultat: total_ventes = 10000 (obsolète !)
```

**Solution :** Exécuter un **REFRESH**.

### REFRESH MATERIALIZED VIEW (Standard)

```sql
REFRESH MATERIALIZED VIEW vm_stats_ventes_mensuelles;
```

**Ce qui se passe :**
1. PostgreSQL ré-exécute la requête de la vue
2. Supprime les anciennes données
3. Insère les nouvelles données
4. Pendant ce temps, la vue est **verrouillée** (ACCESS EXCLUSIVE)

**Problème :** Les lectures sont **bloquées** pendant le refresh (peut prendre des secondes/minutes).

```
Début du refresh
    ↓
Vue verrouillée (pas de lecture possible)
    ↓
Recalcul complet
    ↓
Remplacement des données
    ↓
Fin du refresh (vue déverrouillée)
```

### REFRESH MATERIALIZED VIEW CONCURRENTLY (PostgreSQL 9.4+)

Le refresh **concurrent** permet de rafraîchir sans bloquer les lectures.

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats_ventes_mensuelles;
```

**Avantages :**
- ✅ Les lectures continuent pendant le refresh
- ✅ Pas d'interruption de service
- ✅ Transparent pour les utilisateurs

**Processus :**
```
Début du refresh concurrent
    ↓
Vue reste accessible en lecture (anciennes données)
    ↓
Recalcul en arrière-plan
    ↓
Swap atomique des données (très rapide)
    ↓
Fin du refresh (nouvelles données disponibles)
```

#### Prérequis pour CONCURRENTLY

**Obligatoire :** La vue matérialisée doit avoir **au moins un index UNIQUE**.

```sql
-- Créer la vue matérialisée
CREATE MATERIALIZED VIEW vm_stats_ventes AS
SELECT
    DATE_TRUNC('month', date_vente) AS mois,
    region,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca_total
FROM ventes
GROUP BY DATE_TRUNC('month', date_vente), region;

-- Créer un index UNIQUE (requis pour CONCURRENTLY)
CREATE UNIQUE INDEX idx_vm_stats_mois_region
ON vm_stats_ventes (mois, region);

-- Maintenant le refresh concurrent fonctionne
REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats_ventes;
```

**Sans index UNIQUE :**
```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats_ventes;
-- ERROR: cannot refresh materialized view "vm_stats_ventes" concurrently
-- HINT: Create a unique index with no WHERE clause on one or more columns
```

### Stratégies de Refresh

#### 1. Refresh Manuel

Exécution sur demande (par un administrateur).

```sql
-- Manuellement via psql
REFRESH MATERIALIZED VIEW CONCURRENTLY vm_dashboard;
```

**Usage :** Développement, tests, opérations ponctuelles.

#### 2. Refresh Planifié (Cron)

Automatiser avec un job cron.

```bash
# Crontab : refresh quotidien à 2h du matin
0 2 * * * psql -U user -d database -c "REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats_ventes;"
```

**Usage :** Rapports quotidiens, hebdomadaires, mensuels.

#### 3. Refresh via pg_cron (Extension)

Planification intégrée dans PostgreSQL.

```sql
-- Installer l'extension
CREATE EXTENSION pg_cron;

-- Planifier un refresh quotidien à 2h
SELECT cron.schedule(
    'refresh-vm-stats',           -- nom du job
    '0 2 * * *',                  -- cron schedule
    'REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats_ventes'
);

-- Lister les jobs
SELECT * FROM cron.job;

-- Supprimer un job
SELECT cron.unschedule('refresh-vm-stats');
```

#### 4. Refresh Déclenché par Événements

Utiliser des triggers pour rafraîchir automatiquement.

```sql
-- Fonction de refresh
CREATE OR REPLACE FUNCTION refresh_vm_stats()
RETURNS TRIGGER AS $$
BEGIN
    -- Refresh asynchrone via NOTIFY/LISTEN ou job
    PERFORM pg_notify('refresh_vm_stats', '');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger sur INSERT/UPDATE/DELETE
CREATE TRIGGER trig_refresh_vm_stats
AFTER INSERT OR UPDATE OR DELETE ON ventes
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_vm_stats();
```

**Note :** Le refresh reste coûteux, cette approche n'est viable que si les modifications sont peu fréquentes.

#### 5. Refresh Incrémental (Approximation)

PostgreSQL ne supporte pas nativement le refresh incrémental, mais on peut le simuler :

```sql
-- Vue matérialisée avec timestamp de dernière mise à jour
CREATE MATERIALIZED VIEW vm_ventes_recentes AS
SELECT
    *,
    NOW() AS last_refresh
FROM ventes
WHERE created_at >= NOW() - INTERVAL '7 days';

-- Refresh uniquement les nouvelles données (simulation)
-- Nécessite logique applicative ou procédure stockée complexe
```

**Alternative :** Utiliser des **incremental views** via l'extension IVM (Incremental View Maintenance).

### Monitoring du Refresh

#### Vérifier la Fraîcheur des Données

```sql
-- Ajouter une colonne de tracking
CREATE MATERIALIZED VIEW vm_stats AS
SELECT
    DATE_TRUNC('day', date_vente) AS jour,
    COUNT(*) AS nb_ventes,
    NOW() AS refreshed_at  -- Timestamp du refresh
FROM ventes
GROUP BY DATE_TRUNC('day', date_vente);

-- Vérifier quand la vue a été rafraîchie
SELECT MAX(refreshed_at) AS derniere_maj FROM vm_stats;
```

#### Historique des Refresh

```sql
-- Créer une table de log
CREATE TABLE refresh_log (
    log_id SERIAL PRIMARY KEY,
    view_name TEXT,
    refresh_start TIMESTAMPTZ,
    refresh_end TIMESTAMPTZ,
    refresh_duration INTERVAL,
    rows_affected BIGINT
);

-- Fonction de refresh avec logging
CREATE OR REPLACE FUNCTION refresh_with_log(view_name TEXT)
RETURNS VOID AS $$
DECLARE
    start_time TIMESTAMPTZ;
    end_time TIMESTAMPTZ;
    row_count BIGINT;
BEGIN
    start_time := clock_timestamp();

    EXECUTE format('REFRESH MATERIALIZED VIEW CONCURRENTLY %I', view_name);

    end_time := clock_timestamp();

    EXECUTE format('SELECT COUNT(*) FROM %I', view_name) INTO row_count;

    INSERT INTO refresh_log (view_name, refresh_start, refresh_end, refresh_duration, rows_affected)
    VALUES (view_name, start_time, end_time, end_time - start_time, row_count);
END;
$$ LANGUAGE plpgsql;

-- Utilisation
SELECT refresh_with_log('vm_stats_ventes');

-- Consulter les logs
SELECT * FROM refresh_log ORDER BY refresh_start DESC LIMIT 10;
```

---

## Partie 4 : Indexation des Vues Matérialisées

### Pourquoi Indexer ?

Les vues matérialisées stockent des données physiquement, comme des tables. Sans index, les requêtes font des **scans séquentiels** (lents sur grandes vues).

```sql
-- Vue matérialisée sans index
CREATE MATERIALIZED VIEW vm_logs AS
SELECT * FROM logs WHERE log_level = 'ERROR';

-- Requête lente (scan séquentiel de toute la vue)
EXPLAIN ANALYZE
SELECT * FROM vm_logs WHERE timestamp > '2024-01-01';
-- Seq Scan on vm_logs (cost=0.00..1000.00 rows=500 width=100) (actual time=50.123..150.456)
```

**Solution :** Créer des index.

### Créer des Index sur Vues Matérialisées

```sql
-- Index sur colonne fréquemment filtrée
CREATE INDEX idx_vm_logs_timestamp ON vm_logs(timestamp);

-- Même requête avec index
EXPLAIN ANALYZE
SELECT * FROM vm_logs WHERE timestamp > '2024-01-01';
-- Index Scan using idx_vm_logs_timestamp (actual time=0.015..5.234)
```

**Gain :** De 150ms à 5ms = 30× plus rapide.

### Types d'Index Couramment Utilisés

#### 1. Index B-Tree (Par Défaut)

Pour égalité et plages.

```sql
CREATE INDEX idx_vm_stats_mois ON vm_stats_ventes(mois);
CREATE INDEX idx_vm_stats_region ON vm_stats_ventes(region);

-- Requête optimisée
SELECT * FROM vm_stats_ventes
WHERE mois = '2024-06-01' AND region = 'EMEA';
```

#### 2. Index Composite

Pour requêtes avec plusieurs colonnes.

```sql
CREATE INDEX idx_vm_stats_mois_region ON vm_stats_ventes(mois, region);

-- Très efficace pour cette requête
SELECT * FROM vm_stats_ventes
WHERE mois = '2024-06-01' AND region = 'EMEA';
```

#### 3. Index UNIQUE (Obligatoire pour CONCURRENTLY)

```sql
-- Vue matérialisée avec clé naturelle
CREATE MATERIALIZED VIEW vm_produits_stocks AS
SELECT
    produit_id,
    nom_produit,
    SUM(quantite) AS stock_total
FROM inventaire
GROUP BY produit_id, nom_produit;

-- Index UNIQUE pour refresh concurrent
CREATE UNIQUE INDEX idx_vm_produits_id
ON vm_produits_stocks(produit_id);
```

#### 4. Index Partiel

Pour requêtes fréquentes sur sous-ensemble.

```sql
-- Vue matérialisée de commandes
CREATE MATERIALIZED VIEW vm_commandes AS
SELECT * FROM commandes;

-- Index uniquement sur commandes récentes
CREATE INDEX idx_vm_commandes_recentes
ON vm_commandes(date_commande)
WHERE date_commande >= NOW() - INTERVAL '30 days';
```

#### 5. Index GIN (Pour JSONB, Arrays)

```sql
-- Vue matérialisée avec données JSONB
CREATE MATERIALIZED VIEW vm_evenements AS
SELECT
    evenement_id,
    user_id,
    proprietes::JSONB AS props
FROM evenements;

-- Index GIN pour recherche dans JSONB
CREATE INDEX idx_vm_evenements_props
ON vm_evenements USING GIN(props);

-- Requête optimisée
SELECT * FROM vm_evenements
WHERE props @> '{"action": "achat"}';
```

### Stratégie d'Indexation

#### 1. Identifier les Requêtes Fréquentes

```sql
-- Analyser les requêtes via pg_stat_statements
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
WHERE query LIKE '%vm_stats_ventes%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

#### 2. Créer des Index Ciblés

Basés sur les colonnes dans les clauses WHERE, ORDER BY, JOIN.

```sql
-- Si 80% des requêtes filtrent sur 'region'
CREATE INDEX idx_vm_stats_region ON vm_stats_ventes(region);

-- Si beaucoup de requêtes trient par 'ca_total'
CREATE INDEX idx_vm_stats_ca ON vm_stats_ventes(ca_total DESC);
```

#### 3. Index Covering (INCLUDE)

Pour éviter les lookups dans la table.

```sql
CREATE INDEX idx_vm_stats_covering
ON vm_stats_ventes(mois, region)
INCLUDE (ca_total, nb_ventes);

-- Requête utilisant uniquement l'index (index-only scan)
SELECT mois, region, ca_total
FROM vm_stats_ventes
WHERE mois = '2024-06-01';
```

### Maintenance des Index

#### Impact du Refresh

Les index sont **automatiquement mis à jour** lors du refresh :

```sql
REFRESH MATERIALIZED VIEW vm_stats_ventes;
-- Les index sont reconstruits automatiquement
```

**Coût :** Le refresh prend plus de temps avec des index.

```
Sans index : REFRESH en 30 secondes
Avec 3 index : REFRESH en 45 secondes
```

**Trade-off :** Refresh plus long vs Requêtes plus rapides.

#### REINDEX si Nécessaire

Après beaucoup de refresh, les index peuvent se fragmenter.

```sql
-- Reconstruire les index
REINDEX TABLE vm_stats_ventes;
```

---

## Cas d'Usage Pratiques

### Cas 1 : Dashboard Analytics

**Besoin :** Dashboard affichant des KPIs calculés sur des millions de lignes.

```sql
-- Vue matérialisée des KPIs quotidiens
CREATE MATERIALIZED VIEW vm_dashboard_kpis AS
SELECT
    DATE_TRUNC('day', created_at) AS jour,
    COUNT(DISTINCT user_id) AS utilisateurs_actifs,
    COUNT(DISTINCT session_id) AS sessions_totales,
    SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS achats,
    SUM(CASE WHEN event_type = 'purchase' THEN amount ELSE 0 END) AS revenue_total,
    AVG(session_duration) AS duree_session_moyenne,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY session_duration) AS mediane_session,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY page_load_time) AS p95_load_time
FROM events
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY DATE_TRUNC('day', created_at);

-- Index pour requêtes rapides
CREATE UNIQUE INDEX idx_vm_dashboard_jour ON vm_dashboard_kpis(jour);

-- Refresh quotidien à 1h du matin
-- 0 1 * * * psql ... "REFRESH MATERIALIZED VIEW CONCURRENTLY vm_dashboard_kpis"

-- Requête dashboard ultra-rapide
SELECT * FROM vm_dashboard_kpis
WHERE jour >= CURRENT_DATE - 30
ORDER BY jour DESC;
```

### Cas 2 : Rapports Mensuels

**Besoin :** Rapport de ventes mensuel utilisé par plusieurs équipes.

```sql
-- Vue matérialisée du rapport mensuel
CREATE MATERIALIZED VIEW vm_rapport_ventes_mensuel AS
SELECT
    DATE_TRUNC('month', v.date_vente) AS mois,
    r.nom_region,
    c.nom_categorie,
    COUNT(DISTINCT v.commande_id) AS nb_commandes,
    SUM(v.quantite) AS quantite_totale,
    SUM(v.montant) AS ca_total,
    SUM(v.montant) / COUNT(DISTINCT v.commande_id) AS panier_moyen,
    COUNT(DISTINCT v.client_id) AS nb_clients_uniques,
    COUNT(DISTINCT v.vendeur_id) AS nb_vendeurs_actifs
FROM ventes v
JOIN regions r ON v.region_id = r.region_id
JOIN categories c ON v.categorie_id = c.categorie_id
WHERE v.date_vente >= DATE_TRUNC('month', NOW() - INTERVAL '24 months')
GROUP BY DATE_TRUNC('month', v.date_vente), r.nom_region, c.nom_categorie;

-- Index composite pour analyses
CREATE UNIQUE INDEX idx_rapport_pk
ON vm_rapport_ventes_mensuel(mois, nom_region, nom_categorie);

CREATE INDEX idx_rapport_mois ON vm_rapport_ventes_mensuel(mois);

-- Refresh mensuel (le 1er du mois à 3h)
-- 0 3 1 * * psql ... "REFRESH MATERIALIZED VIEW CONCURRENTLY vm_rapport_ventes_mensuel"
```

### Cas 3 : Recherche Full-Text Pré-Indexée

**Besoin :** Recherche dans des descriptions de produits (millions de produits).

```sql
-- Vue matérialisée avec tsvector pré-calculé
CREATE MATERIALIZED VIEW vm_produits_recherche AS
SELECT
    produit_id,
    nom_produit,
    description,
    to_tsvector('french', nom_produit || ' ' || COALESCE(description, '')) AS search_vector
FROM produits;

-- Index GIN sur le tsvector
CREATE INDEX idx_produits_search ON vm_produits_recherche USING GIN(search_vector);

-- Recherche ultra-rapide
SELECT produit_id, nom_produit
FROM vm_produits_recherche
WHERE search_vector @@ to_tsquery('french', 'ordinateur & portable');
```

### Cas 4 : Dénormalisation pour APIs

**Besoin :** API retournant des données de plusieurs tables jointes (utilisé des milliers de fois par seconde).

```sql
-- Vue matérialisée dénormalisée
CREATE MATERIALIZED VIEW vm_api_produits AS
SELECT
    p.produit_id,
    p.nom_produit,
    p.prix,
    p.description,
    c.nom_categorie,
    c.slug_categorie,
    f.nom_fournisseur,
    i.stock_disponible,
    i.stock_reserve,
    COALESCE(AVG(r.note), 0) AS note_moyenne,
    COUNT(r.review_id) AS nb_avis,
    json_agg(
        json_build_object('taille', v.taille, 'couleur', v.couleur, 'sku', v.sku)
    ) AS variantes
FROM produits p
JOIN categories c ON p.categorie_id = c.categorie_id
JOIN fournisseurs f ON p.fournisseur_id = f.fournisseur_id
LEFT JOIN inventaire i ON p.produit_id = i.produit_id
LEFT JOIN reviews r ON p.produit_id = r.produit_id
LEFT JOIN variantes_produit v ON p.produit_id = v.produit_id
GROUP BY p.produit_id, p.nom_produit, p.prix, p.description,
         c.nom_categorie, c.slug_categorie, f.nom_fournisseur,
         i.stock_disponible, i.stock_reserve;

-- Index pour accès par ID (API GET /products/:id)
CREATE UNIQUE INDEX idx_api_produits_id ON vm_api_produits(produit_id);

-- Index pour recherche par catégorie
CREATE INDEX idx_api_produits_cat ON vm_api_produits(slug_categorie);

-- Refresh nocturne (les prix/stocks changent peu)
```

---

## Comparaison et Choix de la Bonne Approche

### Arbre de Décision

```
Besoin de requêter des données ?
    ↓
┌─────────────────────────────────────────────┐
│ Les données doivent être TOUJOURS à jour ?  │
└─────────────────────────────────────────────┘
    ↓ OUI                    ↓ NON
┌──────────────┐    ┌───────────────────────┐
│ Requête      │    │ Vue matérialisée      │
│ directe sur  │    │ + Refresh planifié    │
│ tables       │    └───────────────────────┘
└──────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ La requête est complexe/répétée souvent ?   │
└─────────────────────────────────────────────┘
    ↓ OUI                    ↓ NON
┌──────────────┐    ┌───────────────────────┐
│ Vue classique│    │ Requête directe       │
│ (VIEW)       │    │                       │
└──────────────┘    └───────────────────────┘
```

### Tableau Comparatif Détaillé

| Besoin | Solution Recommandée | Justification |
|--------|---------------------|---------------|
| **Dashboard temps réel** | Vue classique | Données toujours à jour |
| **Rapport quotidien** | Vue matérialisée | Refresh nocturne suffit |
| **API haute fréquence** | Vue matérialisée | Performance maximale |
| **Requête simple fréquente** | Vue classique | Simplicité |
| **Agrégation lourde** | Vue matérialisée | Pré-calcul avantageux |
| **Données cachées (sécurité)** | Vue classique | Contrôle d'accès |
| **Join complexe répété** | Vue classique ou matérialisée | Selon fréquence |
| **Analytics historique** | Vue matérialisée | Données stables |
| **Données volatiles** | Vue classique ou cache applicatif | MV inefficace |

### Critères de Choix

**Choisir une VUE CLASSIQUE si :**
- ✅ Fraîcheur critique (données en temps réel)
- ✅ Requête rapide (< 100ms)
- ✅ Données très volatiles
- ✅ Petite table
- ✅ Besoin de sécurité/abstraction uniquement

**Choisir une VUE MATÉRIALISÉE si :**
- ✅ Performance critique (requête > 1s)
- ✅ Requête très fréquente (centaines/milliers par seconde)
- ✅ Agrégations complexes
- ✅ Données relativement stables (refresh quotidien/hebdomadaire OK)
- ✅ Jointures massives

**Utiliser les DEUX si :**
- Vue classique sur la vue matérialisée pour filtrage supplémentaire
- Couches d'abstraction successives

---

## Bonnes Pratiques

### 1. Nommage Cohérent

```sql
-- Conventions de nommage
-- Vues classiques : v_nom_descriptif
CREATE VIEW v_employes_actifs AS ...

-- Vues matérialisées : vm_nom_descriptif
CREATE MATERIALIZED VIEW vm_stats_quotidiennes AS ...
```

### 2. Documenter les Vues

```sql
-- Ajouter des commentaires
COMMENT ON MATERIALIZED VIEW vm_dashboard IS
'Vue matérialisée des KPIs pour le dashboard executif.
Rafraîchie quotidiennement à 2h du matin via cron.
Dépend de : events, users, sessions.
Propriétaire : équipe Analytics';

COMMENT ON COLUMN vm_dashboard.utilisateurs_actifs IS
'Nombre d''utilisateurs distincts ayant au moins un événement dans la journée';
```

### 3. Toujours Créer un Index UNIQUE pour CONCURRENTLY

```sql
-- ✅ Bon
CREATE MATERIALIZED VIEW vm_stats AS
SELECT jour, region, SUM(montant) AS total
FROM ventes
GROUP BY jour, region;

CREATE UNIQUE INDEX idx_vm_stats_pk ON vm_stats(jour, region);
REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats;

-- ❌ Mauvais (pas d'index unique)
REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats;  -- Erreur !
```

### 4. Monitorer la Taille et la Fraîcheur

```sql
-- Vue de monitoring
CREATE VIEW v_monitoring_mvs AS
SELECT
    schemaname,
    matviewname,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||matviewname)) AS taille,
    (SELECT MAX(refreshed_at) FROM vm_stats) AS derniere_maj,
    AGE(NOW(), (SELECT MAX(refreshed_at) FROM vm_stats)) AS age_donnees
FROM pg_matviews
ORDER BY pg_total_relation_size(schemaname||'.'||matviewname) DESC;
```

### 5. Tester le Temps de Refresh

```sql
-- Mesurer le temps de refresh
\timing on
REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats;
-- Time: 12345.678 ms (12.3 seconds)
```

Si le refresh prend trop de temps, considérer :
- Optimiser la requête sous-jacente
- Réduire la fenêtre de données
- Partitionner la vue matérialisée

### 6. Éviter les Dépendances Circulaires

```sql
-- ❌ Interdit
CREATE MATERIALIZED VIEW vm_a AS SELECT * FROM vm_b;
CREATE MATERIALIZED VIEW vm_b AS SELECT * FROM vm_a;
-- Erreur : dépendance circulaire
```

### 7. Planifier les Refresh pendant Heures Creuses

```bash
# Cron : refresh à 2h du matin (heure creuse)
0 2 * * * psql -c "REFRESH MATERIALIZED VIEW CONCURRENTLY vm_dashboard"

# Éviter les heures de pointe (9h-18h)
```

### 8. Gérer les Échecs de Refresh

```sql
-- Script de refresh avec gestion d'erreurs
DO $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY vm_stats;
    RAISE NOTICE 'Refresh réussi : vm_stats';
EXCEPTION WHEN OTHERS THEN
    RAISE WARNING 'Échec refresh vm_stats : %', SQLERRM;
    -- Alerter l'équipe (email, Slack, etc.)
END $$;
```

---

## Limitations et Alternatives

### Limitations des Vues Matérialisées

**1. Pas de Refresh Incrémental Natif**

PostgreSQL ne supporte pas le refresh incrémental (mise à jour des seules lignes modifiées).

**Alternative :** Extension **pg_ivm** (Incremental View Maintenance)
```sql
CREATE EXTENSION pg_ivm;

-- Vue avec maintenance incrémentale
CREATE INCREMENTAL MATERIALIZED VIEW ivm_stats AS
SELECT jour, SUM(montant) AS total
FROM ventes
GROUP BY jour;

-- Mise à jour automatique lors d'INSERT/UPDATE/DELETE sur ventes
```

**2. Pas de Refresh Automatique sur Modification**

Contrairement à certains SGBD, PostgreSQL ne rafraîchit pas automatiquement.

**Alternative :** Triggers + Refresh asynchrone (mais coûteux)

**3. Pas de Vues Matérialisées sur Requêtes avec WITH RECURSIVE**

```sql
-- ❌ Non supporté
CREATE MATERIALIZED VIEW vm_hierarchie AS
WITH RECURSIVE ...  -- Erreur
```

### Alternatives aux Vues Matérialisées

#### 1. Cache Applicatif (Redis, Memcached)

Pour données ultra-volatiles ou refresh très fréquent.

```python
# Exemple Python
import redis
r = redis.Redis()

# Calculer et cacher
result = expensive_query()
r.setex('dashboard:stats', 300, json.dumps(result))  # Cache 5 minutes

# Lire du cache
cached = r.get('dashboard:stats')
```

#### 2. Tables Agrégées Manuelles

Contrôle total sur la mise à jour.

```sql
CREATE TABLE stats_quotidiennes (
    jour DATE PRIMARY KEY,
    nb_ventes INTEGER,
    ca_total NUMERIC(15,2),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Mise à jour via procédure stockée
CREATE OR REPLACE FUNCTION update_stats_quotidiennes()
RETURNS VOID AS $$
BEGIN
    DELETE FROM stats_quotidiennes WHERE jour = CURRENT_DATE;

    INSERT INTO stats_quotidiennes (jour, nb_ventes, ca_total)
    SELECT
        CURRENT_DATE,
        COUNT(*),
        SUM(montant)
    FROM ventes
    WHERE DATE(date_vente) = CURRENT_DATE;
END;
$$ LANGUAGE plpgsql;
```

#### 3. TimescaleDB (Continuous Aggregates)

Pour séries temporelles avec refresh incrémental.

```sql
CREATE EXTENSION timescaledb;

-- Agrégation continue (refresh automatique)
CREATE MATERIALIZED VIEW stats_horaires
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', timestamp) AS heure,
    COUNT(*) AS nb_events
FROM events
GROUP BY heure;
```

---

## Conclusion

### Points Clés à Retenir

1. **Vues Classiques (VIEW)**
   - Requête nommée virtuelle
   - Toujours à jour
   - Recalcul à chaque interrogation
   - Idéales pour abstraction et sécurité

2. **Vues Matérialisées (MATERIALIZED VIEW)**
   - Résultats pré-calculés et stockés
   - Ultra-rapides à interroger
   - Nécessitent refresh manuel
   - Idéales pour agrégations coûteuses

3. **Refresh**
   - Standard : REFRESH MATERIALIZED VIEW (bloquant)
   - Concurrent : + CONCURRENTLY (non-bloquant, requis index UNIQUE)
   - Planifier via cron ou pg_cron

4. **Indexation**
   - Créer des index comme sur des tables
   - Index UNIQUE obligatoire pour CONCURRENTLY
   - Trade-off : Refresh plus lent vs Requêtes plus rapides

### Checklist de Décision

**Avant de créer une vue matérialisée :**
- [ ] La requête prend-elle > 1 seconde ?
- [ ] Est-elle exécutée fréquemment (> 100×/jour) ?
- [ ] Les données peuvent-elles être obsolètes de quelques heures/jours ?
- [ ] Ai-je un moyen de rafraîchir régulièrement ?
- [ ] Les bénéfices justifient-ils l'espace disque utilisé ?

**Si 4-5 "oui" → Vue matérialisée recommandée**

### Ressources

- [Documentation PostgreSQL - Views](https://www.postgresql.org/docs/current/sql-createview.html)
- [Documentation PostgreSQL - Materialized Views](https://www.postgresql.org/docs/current/sql-creatematerializedview.html)
- [Extension pg_cron](https://github.com/citusdata/pg_cron)
- [Extension pg_ivm](https://github.com/sraoss/pg_ivm)

---

**Fin du Chapitre 11.5**

Vous maîtrisez maintenant les vues classiques et matérialisées de PostgreSQL. Ces outils sont essentiels pour simplifier vos requêtes, améliorer les performances et créer des abstractions solides dans votre architecture de données. N'oubliez pas : les vues matérialisées sont puissantes, mais nécessitent une stratégie de refresh bien pensée pour être efficaces en production.

⏭️ [Nouveauté PG 18 : Colonnes générées virtuelles (Virtual Generated Columns)](/11-modelisation-avancee/06-colonnes-generees-virtuelles.md)
