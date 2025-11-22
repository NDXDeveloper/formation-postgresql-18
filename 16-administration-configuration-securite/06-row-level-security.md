🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.6. Row-Level Security (RLS) : Politiques par ligne

## Introduction

Jusqu'à présent, nous avons vu comment contrôler l'accès aux **tables entières** : un utilisateur peut ou ne peut pas lire une table, modifier une table, etc. Mais que faire si vous voulez un contrôle **plus fin**, au niveau des **lignes individuelles** ?

**Exemple concret :**
- Dans une application SaaS multi-tenant, chaque client ne doit voir que **ses propres données**
- Dans un système RH, chaque employé ne doit voir que **sa propre fiche de paie**
- Dans un réseau social, les utilisateurs ne doivent voir que les posts **publics ou de leurs amis**

C'est exactement ce que permet **Row-Level Security (RLS)** : filtrer automatiquement les lignes visibles selon l'utilisateur connecté.

---

## 🎯 Qu'est-ce que Row-Level Security (RLS) ?

### Définition

**Row-Level Security (RLS)** est un mécanisme de sécurité dans PostgreSQL qui permet de contrôler **quelles lignes** d'une table un utilisateur peut voir ou modifier, en fonction de **politiques** (policies) définies.

**En d'autres termes :**
> "Même si un utilisateur a SELECT sur une table, il ne verra que certaines lignes selon les règles que vous définissez."

### Analogie du Monde Réel

Imaginez une bibliothèque d'entreprise :

**Sans RLS :**
- Vous donnez la clé de la bibliothèque à quelqu'un
- Cette personne peut lire **tous les livres** de la bibliothèque

**Avec RLS :**
- Vous donnez la clé de la bibliothèque à quelqu'un
- Cette personne peut entrer, mais **ne voit que les livres autorisés pour son département**
- Les autres livres sont invisibles (comme s'ils n'existaient pas)

C'est exactement ainsi que fonctionne RLS : **filtrage invisible et automatique**.

---

## 🚫 Le Problème Sans RLS

### Approche Traditionnelle (Manuelle)

Sans RLS, vous devez filtrer manuellement dans **chaque requête** :

**Exemple : Application SaaS multi-tenant**

```sql
-- Table avec des données de plusieurs clients
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  tenant_id INT NOT NULL,  -- ID du client
  title TEXT,
  content TEXT
);

-- Données
INSERT INTO documents (tenant_id, title, content) VALUES
  (1, 'Doc Client A', 'Contenu confidentiel A'),
  (2, 'Doc Client B', 'Contenu confidentiel B'),
  (3, 'Doc Client C', 'Contenu confidentiel C');
```

**❌ Approche manuelle (problématique) :**

```sql
-- L'application doit TOUJOURS filtrer manuellement
-- Si tenant_id = 1 (Client A)
SELECT * FROM documents WHERE tenant_id = 1;

-- Si tenant_id = 2 (Client B)
SELECT * FROM documents WHERE tenant_id = 2;
```

### Les Dangers de l'Approche Manuelle

**1. Oubli de filtrer :**
```sql
-- ❌ Bug critique : Le développeur oublie le WHERE
SELECT * FROM documents;
-- Résultat : Le client A voit les données du client B et C !
```

**2. Requêtes complexes :**
```sql
-- ❌ Facile d'oublier le filtre dans une sous-requête
SELECT d.*,
  (SELECT COUNT(*) FROM documents WHERE tenant_id = d.tenant_id) AS total
FROM documents d
WHERE tenant_id = 1;  -- Filtre dans la requête principale
-- Mais la sous-requête compte TOUS les documents (bug !) ⚠️
```

**3. Maintenance cauchemardesque :**
- Chaque requête doit être vérifiée
- Risque d'erreur élevé
- Code répétitif partout : `WHERE tenant_id = current_tenant_id`

**4. Sécurité dépendante du code applicatif :**
- Si l'application est compromise, aucune protection côté base
- Aucune défense en profondeur

---

## ✅ La Solution : Row-Level Security

### Principe de Fonctionnement

Avec RLS, PostgreSQL **ajoute automatiquement** un filtre invisible à **toutes** les requêtes :

```sql
-- Ce que vous écrivez :
SELECT * FROM documents;

-- Ce que PostgreSQL exécute réellement (avec RLS) :
SELECT * FROM documents
WHERE tenant_id = current_setting('app.current_tenant')::INT;
```

**Avantages majeurs :**
- ✅ **Automatique** : Plus besoin de filtrer manuellement
- ✅ **Invisible** : Le développeur ne peut pas oublier
- ✅ **Sécurité en profondeur** : Protection au niveau de la base de données
- ✅ **Centralisé** : Les règles sont définies une seule fois

---

## 🔧 Activation de RLS

### Étape 1 : Activer RLS sur une Table

**Syntaxe :**
```sql
ALTER TABLE nom_table ENABLE ROW LEVEL SECURITY;
```

**Exemple :**
```sql
-- Activer RLS sur la table documents
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
```

⚠️ **Important** : Par défaut, une fois RLS activé, **PERSONNE** (sauf le propriétaire et les superusers) ne peut voir aucune ligne ! Il faut créer des politiques.

### Étape 2 : Créer des Politiques (Policies)

Une **politique** (policy) définit **qui** peut voir/modifier **quelles lignes**.

**Syntaxe de base :**
```sql
CREATE POLICY nom_politique
  ON nom_table
  FOR {ALL | SELECT | INSERT | UPDATE | DELETE}
  TO {role | PUBLIC}
  USING (condition);  -- Quelles lignes sont visibles
```

**Exemple complet :**
```sql
-- Politique : Chaque tenant ne voit que ses propres documents
CREATE POLICY tenant_isolation_policy
  ON documents
  FOR SELECT
  TO PUBLIC
  USING (tenant_id = current_setting('app.current_tenant')::INT);
```

**Explication :**
- `tenant_isolation_policy` : Nom de la politique
- `ON documents` : S'applique à la table "documents"
- `FOR SELECT` : Pour les requêtes SELECT (lecture)
- `TO PUBLIC` : Pour tous les utilisateurs
- `USING (...)` : Condition qui doit être vraie pour voir une ligne

---

## 🎭 Anatomie d'une Politique RLS

### Les Composants d'une Politique

```sql
CREATE POLICY nom_politique
  ON table_name
  [AS {PERMISSIVE | RESTRICTIVE}]  -- Optionnel
  FOR {ALL | SELECT | INSERT | UPDATE | DELETE}
  TO {role_name | PUBLIC}
  USING (condition_visibilite)     -- Quelles lignes voir
  [WITH CHECK (condition_modification)];  -- Quelles lignes modifier
```

### FOR : Type d'Opération

**Options disponibles :**

| Valeur | Description | Quand l'utiliser |
|--------|-------------|------------------|
| `SELECT` | Lecture uniquement | Filtrer les données visibles |
| `INSERT` | Insertion uniquement | Contrôler les nouvelles lignes |
| `UPDATE` | Modification uniquement | Contrôler les mises à jour |
| `DELETE` | Suppression uniquement | Contrôler les suppressions |
| `ALL` | Toutes les opérations | Politique générale |

**Exemples :**
```sql
-- Politique pour SELECT seulement
CREATE POLICY read_own_data
  ON documents
  FOR SELECT
  USING (owner_id = current_user_id());

-- Politique pour INSERT seulement
CREATE POLICY insert_own_tenant
  ON documents
  FOR INSERT
  WITH CHECK (tenant_id = current_tenant_id());

-- Politique pour toutes les opérations
CREATE POLICY all_operations
  ON documents
  FOR ALL
  USING (tenant_id = current_tenant_id());
```

### USING vs WITH CHECK

**USING (condition)** : Détermine **quelles lignes existantes** sont visibles/modifiables

**WITH CHECK (condition)** : Détermine **quelles nouvelles lignes** peuvent être insérées ou comment les lignes existantes peuvent être modifiées

**Différence illustrée :**

```sql
-- Exemple : Employés ne peuvent voir que leurs propres données
CREATE POLICY employee_access
  ON salaries
  FOR ALL
  USING (employee_id = current_user_id())          -- Lecture : seulement mes lignes
  WITH CHECK (employee_id = current_user_id());    -- Écriture : ne peut créer que pour moi
```

**Scénarios :**

**SELECT :**
```sql
-- Utilise USING
SELECT * FROM salaries;
-- PostgreSQL filtre avec : WHERE employee_id = current_user_id()
```

**INSERT :**
```sql
-- Utilise WITH CHECK
INSERT INTO salaries (employee_id, amount) VALUES (123, 50000);
-- PostgreSQL vérifie : 123 = current_user_id() ?
-- Si faux : ERROR: new row violates row-level security policy
```

**UPDATE :**
```sql
-- Utilise USING (pour trouver les lignes) ET WITH CHECK (pour valider le résultat)
UPDATE salaries SET amount = 55000 WHERE employee_id = 123;
-- 1. USING : Peut-on voir la ligne employee_id=123 ?
-- 2. WITH CHECK : La ligne modifiée respecte-t-elle toujours la condition ?
```

**DELETE :**
```sql
-- Utilise USING seulement
DELETE FROM salaries WHERE employee_id = 123;
-- PostgreSQL vérifie : Peut-on voir la ligne employee_id=123 ?
```

---

## 🔀 Types de Politiques : PERMISSIVE vs RESTRICTIVE

### PERMISSIVE (Par Défaut)

**Principe :** Les politiques PERMISSIVE utilisent l'opérateur logique **OR** entre elles.

**Comportement :**
```
Une ligne est visible SI :
  politique1 = TRUE  OR  politique2 = TRUE  OR  politique3 = TRUE
```

**Exemple :**
```sql
-- Politique 1 : Voir ses propres documents
CREATE POLICY see_own_docs
  ON documents
  FOR SELECT
  USING (owner_id = current_user_id());

-- Politique 2 : Voir les documents publics
CREATE POLICY see_public_docs
  ON documents
  FOR SELECT
  USING (is_public = TRUE);

-- Résultat : L'utilisateur voit :
-- - Ses propres documents (même privés)
-- - OU tous les documents publics
-- C'est une union (OR)
```

### RESTRICTIVE

**Principe :** Les politiques RESTRICTIVE utilisent l'opérateur logique **AND** entre elles.

**Comportement :**
```
Une ligne est visible SI :
  politique1 = TRUE  AND  politique2 = TRUE  AND  politique3 = TRUE
```

**Syntaxe :**
```sql
CREATE POLICY nom_politique
  ON table_name
  AS RESTRICTIVE  -- Mot clé important
  FOR SELECT
  USING (condition);
```

**Exemple :**
```sql
-- Politique PERMISSIVE : Voir documents de son département
CREATE POLICY see_department_docs
  ON documents
  FOR SELECT
  USING (department_id = current_department_id());

-- Politique RESTRICTIVE : Mais jamais les documents archivés
CREATE POLICY never_see_archived
  ON documents
  AS RESTRICTIVE
  FOR SELECT
  USING (archived = FALSE);

-- Résultat : L'utilisateur voit :
-- - Documents de son département (PERMISSIVE)
-- - ET qui ne sont pas archivés (RESTRICTIVE)
-- Les deux conditions doivent être vraies
```

### Combinaison PERMISSIVE + RESTRICTIVE

**Logique globale :**
```
Une ligne est accessible SI :
  (
    permissive1 = TRUE
    OR permissive2 = TRUE
    OR ...
    OR permissiveN = TRUE
  )
  AND
  (
    restrictive1 = TRUE
    AND restrictive2 = TRUE
    AND ...
    AND restrictiveN = TRUE
  )
```

**Exemple concret :**
```sql
-- PERMISSIVE : Voir ses docs OU les docs publics
CREATE POLICY see_own_or_public
  ON documents
  FOR SELECT
  USING (owner_id = current_user_id() OR is_public = TRUE);

-- RESTRICTIVE : Jamais les documents supprimés
CREATE POLICY never_deleted
  ON documents
  AS RESTRICTIVE
  FOR SELECT
  USING (deleted_at IS NULL);

-- RESTRICTIVE : Seulement si l'utilisateur est actif
CREATE POLICY only_active_users
  ON documents
  AS RESTRICTIVE
  FOR SELECT
  USING (current_user_is_active());

-- Résultat final : L'utilisateur voit les documents qui sont :
-- - (Siens OU publics)
--   ET
-- - (Non supprimés)
--   ET
-- - (Utilisateur actif)
```

---

## 🛠️ Configuration de l'Utilisateur Actuel

### Le Problème de l'Identification

Dans les politiques, nous avons utilisé des fonctions comme `current_user_id()` ou `current_tenant_id()`. Mais **comment PostgreSQL sait-il qui est l'utilisateur actuel** ?

PostgreSQL ne le sait pas automatiquement ! Vous devez le **configurer explicitement**.

### Méthode 1 : Utilisation de `current_user`

**Variable built-in :** `current_user` retourne le rôle PostgreSQL connecté.

**Exemple :**
```sql
-- Créer des utilisateurs PostgreSQL par personne
CREATE USER alice WITH PASSWORD 'pwd';
CREATE USER bob WITH PASSWORD 'pwd';

-- Politique basée sur current_user
CREATE POLICY user_sees_own_data
  ON documents
  FOR SELECT
  USING (owner = current_user);

-- Quand alice se connecte :
SELECT * FROM documents;
-- PostgreSQL exécute : WHERE owner = 'alice'
```

**Avantages :**
- ✅ Simple
- ✅ Pas de configuration supplémentaire

**Inconvénients :**
- ❌ Nécessite un utilisateur PostgreSQL par utilisateur (non scalable)
- ❌ Gestion des connexions complexe

### Méthode 2 : Variables de Session (current_setting)

**Principe :** Utiliser `current_setting()` pour lire des variables de session personnalisées.

**Configuration côté application :**
```sql
-- Au début de chaque session (connexion)
SET app.current_user_id = '123';
SET app.current_tenant_id = '5';
```

**Politique utilisant les variables :**
```sql
CREATE POLICY tenant_isolation
  ON documents
  FOR SELECT
  USING (tenant_id = current_setting('app.current_tenant_id')::INT);
```

**Exemple complet avec driver Python (psycopg3) :**
```python
import psycopg

# Connexion à PostgreSQL
conn = psycopg.connect("dbname=myapp user=app_user password=secret")

# Définir le tenant actuel pour cette session
conn.execute("SET app.current_tenant_id = %s", [current_tenant_id])

# Maintenant toutes les requêtes sont automatiquement filtrées
cursor = conn.execute("SELECT * FROM documents")
# Ne voit que les documents du tenant actuel
```

**Avantages :**
- ✅ Scalable (un seul utilisateur PostgreSQL pour l'application)
- ✅ Flexible
- ✅ Facile à intégrer dans les applications modernes

**Inconvénients :**
- ❌ Nécessite de définir les variables à chaque connexion
- ❌ Risque d'oubli (mitigation : utiliser un middleware)

### Méthode 3 : Variables de Transaction Locales (Recommandé)

**Principe :** Utiliser `set_config()` avec `is_local = true` pour que la variable ne dure que le temps de la transaction.

**Configuration :**
```sql
-- Début de transaction
BEGIN;

-- Définir la variable pour cette transaction uniquement
SELECT set_config('app.current_tenant_id', '5', true);  -- true = local

-- Requêtes filtrées automatiquement
SELECT * FROM documents;  -- Ne voit que tenant_id = 5

-- Fin de transaction
COMMIT;

-- La variable est automatiquement réinitialisée
```

**Avantages :**
- ✅ Sécurisé : La variable disparaît automatiquement
- ✅ Isolation : Pas de risque de "pollution" entre transactions
- ✅ Scalable

---

## 📋 Cas d'Usage Pratiques

### Cas 1 : Application SaaS Multi-Tenant

**Contexte :** Application SaaS avec plusieurs clients (tenants). Chaque client doit être complètement isolé.

**Structure de table :**
```sql
CREATE TABLE customers (
  id SERIAL PRIMARY KEY,
  tenant_id INT NOT NULL,
  name TEXT,
  email TEXT
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  tenant_id INT NOT NULL,
  customer_id INT REFERENCES customers(id),
  total DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Configuration RLS :**
```sql
-- Activer RLS sur toutes les tables
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Politique d'isolation par tenant
CREATE POLICY tenant_isolation_customers
  ON customers
  FOR ALL
  USING (tenant_id = current_setting('app.current_tenant_id')::INT)
  WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::INT);

CREATE POLICY tenant_isolation_orders
  ON orders
  FOR ALL
  USING (tenant_id = current_setting('app.current_tenant_id')::INT)
  WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::INT);
```

**Usage côté application :**
```sql
-- Connexion du tenant 5
SET app.current_tenant_id = '5';

-- Requêtes automatiquement filtrées
SELECT * FROM customers;  -- Seulement tenant_id = 5
SELECT * FROM orders;     -- Seulement tenant_id = 5

-- Impossible de voir les données d'autres tenants !
```

**Sécurité garantie :**
- ✅ Même si l'application a un bug, PostgreSQL filtre
- ✅ Impossible d'accéder aux données d'un autre tenant
- ✅ Pas besoin de filtrer dans chaque requête SQL

### Cas 2 : Système RH - Fiches de Paie

**Contexte :** Chaque employé peut voir uniquement sa propre fiche de paie. Les RH peuvent tout voir.

**Structure de table :**
```sql
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  username TEXT UNIQUE NOT NULL,
  is_hr BOOLEAN DEFAULT FALSE
);

CREATE TABLE payslips (
  id SERIAL PRIMARY KEY,
  employee_id INT REFERENCES employees(id),
  month DATE,
  amount DECIMAL(10,2),
  details JSONB
);
```

**Configuration RLS :**
```sql
-- Activer RLS
ALTER TABLE payslips ENABLE ROW LEVEL SECURITY;

-- Politique 1 : Employés voient leurs propres fiches
CREATE POLICY employee_own_payslips
  ON payslips
  FOR SELECT
  USING (
    employee_id = (
      SELECT id FROM employees
      WHERE username = current_user
    )
  );

-- Politique 2 : RH voient tout
CREATE POLICY hr_see_all_payslips
  ON payslips
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM employees
      WHERE username = current_user
      AND is_hr = TRUE
    )
  );
```

**Résultat :**
```sql
-- Alice (employé normal) se connecte
SET ROLE alice;
SELECT * FROM payslips;
-- Voit uniquement ses propres fiches

-- Bob (RH) se connecte
SET ROLE bob;
SELECT * FROM payslips;
-- Voit toutes les fiches de tous les employés
```

### Cas 3 : Réseau Social - Visibilité des Posts

**Contexte :** Les utilisateurs voient leurs propres posts, les posts publics, et les posts de leurs amis.

**Structure de table :**
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username TEXT UNIQUE
);

CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  author_id INT REFERENCES users(id),
  content TEXT,
  visibility TEXT CHECK (visibility IN ('public', 'friends', 'private'))
);

CREATE TABLE friendships (
  user_id INT REFERENCES users(id),
  friend_id INT REFERENCES users(id),
  PRIMARY KEY (user_id, friend_id)
);
```

**Configuration RLS :**
```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY posts_visibility
  ON posts
  FOR SELECT
  USING (
    -- Condition 1 : Posts publics
    visibility = 'public'
    OR
    -- Condition 2 : Mes propres posts
    author_id = current_setting('app.current_user_id')::INT
    OR
    -- Condition 3 : Posts d'amis avec visibilité "friends"
    (
      visibility = 'friends'
      AND EXISTS (
        SELECT 1 FROM friendships
        WHERE user_id = current_setting('app.current_user_id')::INT
        AND friend_id = author_id
      )
    )
  );
```

**Comportement :**
- Alice voit : posts publics + ses posts + posts "friends" de ses amis
- Impossible de voir les posts privés d'autres personnes
- Impossible de voir les posts "friends" de non-amis

### Cas 4 : Données Temporelles - Archivage

**Contexte :** Les utilisateurs normaux ne voient pas les données archivées. Les administrateurs voient tout.

**Structure de table :**
```sql
CREATE TABLE projects (
  id SERIAL PRIMARY KEY,
  name TEXT,
  archived BOOLEAN DEFAULT FALSE,
  archived_at TIMESTAMP
);
```

**Configuration RLS :**
```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Politique PERMISSIVE : Voir les projets actifs
CREATE POLICY see_active_projects
  ON projects
  FOR SELECT
  USING (archived = FALSE);

-- Politique PERMISSIVE : Les admins voient tout
CREATE POLICY admins_see_all
  ON projects
  FOR SELECT
  USING (current_user IN ('admin', 'superuser'));
```

---

## 🔍 Gestion et Maintenance des Politiques

### Lister les Politiques Existantes

```sql
-- Voir toutes les politiques d'une table
SELECT
  schemaname,
  tablename,
  policyname,
  permissive,  -- PERMISSIVE ou RESTRICTIVE
  roles,
  cmd,         -- ALL, SELECT, INSERT, UPDATE, DELETE
  qual         -- Expression USING
FROM pg_policies
WHERE tablename = 'documents';
```

**Depuis psql :**
```
\d+ documents
```

### Modifier une Politique

```sql
-- PostgreSQL ne supporte pas ALTER POLICY directement
-- Il faut supprimer et recréer

-- 1. Supprimer l'ancienne politique
DROP POLICY old_policy ON documents;

-- 2. Créer la nouvelle
CREATE POLICY new_policy
  ON documents
  FOR SELECT
  USING (nouvelle_condition);
```

### Désactiver RLS Temporairement

```sql
-- Désactiver RLS sur une table
ALTER TABLE documents DISABLE ROW LEVEL SECURITY;

-- Réactiver RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
```

### Forcer RLS pour le Propriétaire de la Table

Par défaut, RLS **ne s'applique PAS** au propriétaire de la table ni aux superusers.

**Forcer l'application :**
```sql
ALTER TABLE documents FORCE ROW LEVEL SECURITY;
```

Maintenant, **même le propriétaire** doit respecter les politiques RLS.

### Supprimer une Politique

```sql
DROP POLICY nom_politique ON nom_table;
```

---

## ⚠️ Pièges et Erreurs Courantes

### Erreur #1 : Activer RLS sans Créer de Politiques

```sql
-- On active RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- ❌ ERREUR : Aucune politique définie
-- Résultat : PERSONNE ne peut voir aucune ligne (même pas SELECT) !

SELECT * FROM documents;
-- Résultat : 0 lignes (même si la table contient des données)
```

**Solution :** Toujours créer au moins une politique après activation de RLS.

### Erreur #2 : Oublier WITH CHECK

```sql
-- Politique sans WITH CHECK
CREATE POLICY tenant_read
  ON documents
  FOR ALL
  USING (tenant_id = current_setting('app.current_tenant_id')::INT);
-- ❌ Manque WITH CHECK !

-- Problème lors de INSERT
INSERT INTO documents (tenant_id, title)
VALUES (999, 'Document malveillant');
-- ✅ Insertion réussie ! (bug de sécurité)
-- L'utilisateur peut insérer pour un autre tenant !
```

**Solution :** Toujours utiliser WITH CHECK pour INSERT/UPDATE.

```sql
CREATE POLICY tenant_isolation
  ON documents
  FOR ALL
  USING (tenant_id = current_setting('app.current_tenant_id')::INT)
  WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::INT);
```

### Erreur #3 : Performance - Sous-requêtes Complexes

```sql
-- ❌ Politique avec sous-requête coûteuse
CREATE POLICY expensive_policy
  ON orders
  FOR SELECT
  USING (
    customer_id IN (
      SELECT id FROM customers
      WHERE region = (
        SELECT region FROM users WHERE id = current_user_id
      )
    )
  );

-- Cette politique est exécutée POUR CHAQUE LIGNE !
-- Si la table a 1 million de lignes = 1 million de sous-requêtes
```

**Solution :** Optimiser les politiques ou utiliser des vues matérialisées.

### Erreur #4 : Confusion PERMISSIVE vs RESTRICTIVE

```sql
-- Intention : Bloquer l'accès aux documents archivés
CREATE POLICY no_archived  -- ❌ Par défaut = PERMISSIVE
  ON documents
  FOR SELECT
  USING (archived = FALSE);

-- Autre politique existante
CREATE POLICY see_own_docs
  ON documents
  FOR SELECT
  USING (owner_id = current_user_id());

-- Résultat : OR entre les deux politiques
-- L'utilisateur voit : (documents non archivés) OU (ses propres docs)
-- Donc il voit quand même ses docs archivés ! (bug)
```

**Solution :** Utiliser RESTRICTIVE pour les contraintes globales.

```sql
CREATE POLICY no_archived
  ON documents
  AS RESTRICTIVE  -- ✅ Correct
  FOR SELECT
  USING (archived = FALSE);
```

### Erreur #5 : Ne Pas Tester en Tant qu'Utilisateur Normal

```sql
-- En tant que superuser ou propriétaire
SELECT * FROM documents;
-- ✅ Fonctionne (RLS ne s'applique pas aux superusers)

-- Vous pensez que RLS fonctionne...
-- Mais en production, les utilisateurs normaux ont un comportement différent !
```

**Solution :** Toujours tester avec un utilisateur normal.

```sql
-- Créer un utilisateur de test
CREATE USER test_user WITH PASSWORD 'pwd';
GRANT SELECT ON documents TO test_user;

-- Se connecter en tant que test_user
SET ROLE test_user;

-- Maintenant RLS s'applique
SELECT * FROM documents;
```

---

## 🆚 RLS vs Autres Approches

### Comparaison : RLS vs Filtrage Applicatif

| Critère | RLS | Filtrage Applicatif |
|---------|-----|---------------------|
| **Où ?** | Base de données | Code application |
| **Oubli possible ?** | ❌ Non (automatique) | ✅ Oui (manuel) |
| **Défense en profondeur** | ✅ Oui | ❌ Non |
| **Performance** | ⚠️ Peut être plus lent | ✅ Contrôlable |
| **Complexité** | ⚠️ Apprentissage requis | ✅ Simple |
| **Audit** | ✅ Centralisé en base | ❌ Dispersé dans le code |
| **Maintenance** | ✅ Un seul endroit | ❌ Partout dans le code |

### Comparaison : RLS vs Vues Dédiées

**Approche par vues :**
```sql
-- Créer une vue par rôle
CREATE VIEW documents_for_tenant AS
  SELECT * FROM documents
  WHERE tenant_id = current_setting('app.current_tenant_id')::INT;

-- Donner accès à la vue uniquement
GRANT SELECT ON documents_for_tenant TO app_user;
REVOKE ALL ON documents FROM app_user;
```

| Critère | RLS | Vues Dédiées |
|---------|-----|--------------|
| **Nombre d'objets** | ✅ Une table | ❌ Table + Vue(s) |
| **INSERT/UPDATE/DELETE** | ✅ Direct | ⚠️ Nécessite INSTEAD OF triggers |
| **Flexibilité** | ✅ Haute | ⚠️ Limitée |
| **Complexité** | ⚠️ Moyenne | ✅ Simple |
| **Performance** | ✅ Bonne | ✅ Bonne |

### Quand Utiliser RLS ?

✅ **Utilisez RLS pour :**
- Applications multi-tenant (SaaS)
- Données sensibles nécessitant une isolation stricte
- Systèmes où différents utilisateurs ont différents niveaux d'accès
- Conformité réglementaire (RGPD, HIPAA)
- Défense en profondeur (sécurité au niveau BDD)

❌ **N'utilisez PAS RLS pour :**
- Politiques très complexes avec impact performance majeur
- Filtres qui changent constamment (RLS n'est pas un moteur de règles métier)
- Tables avec milliards de lignes et politiques très coûteuses

---

## 🎯 Bonnes Pratiques

### 1. Toujours Définir WITH CHECK

```sql
-- ✅ BON
CREATE POLICY tenant_isolation
  ON documents
  FOR ALL
  USING (tenant_id = current_tenant())
  WITH CHECK (tenant_id = current_tenant());
```

### 2. Utiliser des Fonctions pour Réutilisation

```sql
-- Créer une fonction réutilisable
CREATE FUNCTION current_tenant_id()
RETURNS INT AS $$
  SELECT current_setting('app.current_tenant_id')::INT;
$$ LANGUAGE SQL STABLE;

-- Utiliser dans les politiques
CREATE POLICY tenant_isolation
  ON documents
  FOR ALL
  USING (tenant_id = current_tenant_id())
  WITH CHECK (tenant_id = current_tenant_id());
```

### 3. Nommer Clairement les Politiques

```sql
-- ❌ Nom vague
CREATE POLICY policy1 ON documents ...

-- ✅ Nom descriptif
CREATE POLICY tenant_isolation_select ON documents ...
CREATE POLICY block_archived_documents ON documents ...
```

### 4. Documenter les Politiques

```sql
-- Ajouter des commentaires
COMMENT ON POLICY tenant_isolation ON documents IS
  'Isole les données par tenant_id. Chaque tenant ne voit que ses données.
   Dépend de la variable app.current_tenant_id définie par l application.';
```

### 5. Tester en Tant qu'Utilisateur Normal

```sql
-- Toujours tester avec un rôle non-privilégié
CREATE ROLE test_user;
GRANT SELECT, INSERT ON documents TO test_user;

SET ROLE test_user;
-- Tester les politiques
RESET ROLE;
```

### 6. Monitorer les Performances

```sql
-- Activer le timing pour mesurer l'impact
\timing on

-- Comparer avec et sans RLS
ALTER TABLE documents DISABLE ROW LEVEL SECURITY;
EXPLAIN ANALYZE SELECT * FROM documents WHERE ...;

ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
EXPLAIN ANALYZE SELECT * FROM documents WHERE ...;
```

### 7. Utiliser FORCE pour les Tests de Sécurité

```sql
-- Forcer RLS même pour le propriétaire
ALTER TABLE documents FORCE ROW LEVEL SECURITY;

-- Tester en tant que propriétaire
SELECT * FROM documents;  -- RLS s'applique maintenant
```

---

## 🧠 Points Clés à Retenir

1. **RLS = Filtrage automatique** au niveau des lignes, invisible pour l'application
2. **Activation en deux étapes** : `ENABLE ROW LEVEL SECURITY` + `CREATE POLICY`
3. **USING vs WITH CHECK** :
   - USING : Quelles lignes voir/modifier
   - WITH CHECK : Quelles lignes créer/résultat après modification
4. **PERMISSIVE (OR) vs RESTRICTIVE (AND)** : Choisir selon le besoin
5. **Configuration utilisateur** : Via `current_user` ou `current_setting()`
6. **RLS ne s'applique pas** aux superusers/propriétaires (sauf FORCE)
7. **Toujours tester** avec un utilisateur normal
8. **Performance** : Attention aux politiques complexes sur grandes tables

---

## 📊 Checklist de Mise en Place RLS

### Phase 1 : Planification

```
☐ Identifier les tables nécessitant RLS
☐ Définir les règles métier de visibilité
☐ Choisir la méthode d'identification utilisateur
☐ Planifier les politiques (PERMISSIVE vs RESTRICTIVE)
☐ Évaluer l'impact performance
```

### Phase 2 : Implémentation

```
☐ Activer RLS sur les tables : ALTER TABLE ... ENABLE ROW LEVEL SECURITY
☐ Créer les politiques SELECT avec USING
☐ Créer les politiques INSERT/UPDATE/DELETE avec WITH CHECK
☐ Créer des fonctions utilitaires si nécessaire
☐ Documenter les politiques (COMMENT)
```

### Phase 3 : Tests

```
☐ Créer un utilisateur de test non-privilégié
☐ Tester SELECT (voir les bonnes lignes)
☐ Tester INSERT (bloquer les insertions illégales)
☐ Tester UPDATE (bloquer les modifications illégales)
☐ Tester DELETE (bloquer les suppressions illégales)
☐ Mesurer l'impact performance (EXPLAIN ANALYZE)
☐ Tester avec FORCE ROW LEVEL SECURITY
```

### Phase 4 : Production

```
☐ Déployer progressivement (par table ou par environnement)
☐ Monitorer les performances
☐ Auditer les accès (logs PostgreSQL)
☐ Former les développeurs
☐ Documenter pour l'équipe
```

---

## 🚀 Pour Aller Plus Loin

### Sujets Connexes

Dans les sections suivantes du tutoriel :

- **16.4** : Gestion des autorisations (`GRANT`/`REVOKE`)
- **16.5** : Rôles, Groupes et principe du moindre privilège
- **19.4** : Troubleshooting et performance des politiques RLS

### Concepts Avancés

**Politiques Dynamiques :**
- Modifier les politiques selon le contexte (heure, jour, géolocalisation)
- Utiliser des fonctions PL/pgSQL complexes

**RLS + Partitionnement :**
- Combiner RLS avec le partitionnement de tables
- Optimiser les performances sur grandes tables

**RLS + Audit :**
- Extension pgaudit pour tracer les accès
- Logs détaillés des violations de politiques

---

## 📚 Ressources Complémentaires

- **Documentation PostgreSQL** : [Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- **Documentation PostgreSQL** : [CREATE POLICY](https://www.postgresql.org/docs/current/sql-createpolicy.html)
- **Blog** : [Row-Level Security in Practice](https://www.postgresql.org/about/news/row-level-security-1622/)
- **Tutorial** : [Multi-Tenant Applications with RLS](https://www.citusdata.com/blog/2016/10/03/designing-your-saas-database-for-high-scalability/)

---


⏭️ [SSL/TLS et chiffrement des connexions](/16-administration-configuration-securite/07-ssl-tls-chiffrement.md)
