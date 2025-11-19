🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.7. Gestion des Modifications de Schéma (ALTER, DROP) et Verrouillage

## Introduction

Dans PostgreSQL, la structure de vos bases de données n'est pas figée. Vous serez régulièrement amené à modifier des tables existantes, ajouter ou supprimer des colonnes, renommer des objets, ou encore supprimer des éléments devenus obsolètes. Ces opérations de modification de schéma utilisent principalement deux commandes SQL : **ALTER** et **DROP**.

Cependant, ces opérations ne sont pas anodines. Elles peuvent impacter les performances et la disponibilité de votre base de données, notamment à cause du **verrouillage** (locking) qu'elles génèrent. Comprendre ces mécanismes est essentiel pour intervenir en toute sécurité sur des bases de données en production.

---

## 1. La Commande ALTER : Modifier des Objets Existants

### 1.1. Qu'est-ce que ALTER ?

La commande **ALTER** permet de modifier la structure d'un objet de base de données existant sans le recréer entièrement. Elle s'applique à de nombreux types d'objets :

- Tables (`ALTER TABLE`)
- Schémas (`ALTER SCHEMA`)
- Séquences (`ALTER SEQUENCE`)
- Types (`ALTER TYPE`)
- Domaines (`ALTER DOMAIN`)
- Fonctions (`ALTER FUNCTION`)
- Et bien d'autres...

### 1.2. ALTER TABLE : Les Modifications de Tables

C'est la commande ALTER la plus courante. Elle permet de multiples opérations sur une table existante.

#### 1.2.1. Ajouter une Colonne

**Syntaxe de base :**

```sql
ALTER TABLE nom_table
ADD COLUMN nom_colonne type_donnees [contraintes];
```

**Exemple :**

```sql
-- Ajouter une colonne email à la table utilisateurs
ALTER TABLE utilisateurs
ADD COLUMN email VARCHAR(255);
```

**Points importants :**
- La nouvelle colonne sera NULL pour toutes les lignes existantes (sauf si vous spécifiez une valeur par défaut)
- Vous pouvez ajouter une contrainte NOT NULL, mais attention : cela échouera si la table contient déjà des données (car les valeurs existantes seraient NULL)

**Avec une valeur par défaut :**

```sql
ALTER TABLE utilisateurs
ADD COLUMN actif BOOLEAN DEFAULT TRUE;
```

Dans PostgreSQL (à partir de la version 11), ajouter une colonne avec une valeur par défaut est une opération **très rapide** car la base ne réécrit pas physiquement la table.

#### 1.2.2. Supprimer une Colonne

**Syntaxe :**

```sql
ALTER TABLE nom_table
DROP COLUMN nom_colonne [CASCADE | RESTRICT];
```

**Exemple :**

```sql
-- Supprimer la colonne temporaire
ALTER TABLE utilisateurs
DROP COLUMN colonne_temporaire;
```

**Options importantes :**
- **RESTRICT** (par défaut) : La suppression échoue si d'autres objets dépendent de cette colonne (vues, index, contraintes)
- **CASCADE** : Supprime également tous les objets dépendants (attention : c'est destructif !)

**Exemple avec CASCADE :**

```sql
ALTER TABLE utilisateurs
DROP COLUMN email CASCADE;
-- Supprime aussi les index, contraintes et vues utilisant cette colonne
```

⚠️ **Attention :** La suppression d'une colonne est une opération lourde qui nécessite un verrouillage exclusif et peut prendre du temps sur de grandes tables.

#### 1.2.3. Modifier le Type d'une Colonne

**Syntaxe :**

```sql
ALTER TABLE nom_table
ALTER COLUMN nom_colonne TYPE nouveau_type [USING expression];
```

**Exemple simple :**

```sql
-- Changer VARCHAR(50) en VARCHAR(100)
ALTER TABLE utilisateurs
ALTER COLUMN nom TYPE VARCHAR(100);
```

**Exemple avec conversion :**

```sql
-- Convertir une colonne texte en entier
ALTER TABLE commandes
ALTER COLUMN quantite TYPE INTEGER USING quantite::INTEGER;
```

**Points critiques :**
- Cette opération nécessite souvent une **réécriture complète de la table** (longue et coûteuse)
- PostgreSQL doit vérifier que toutes les valeurs existantes sont compatibles avec le nouveau type
- Sur de grandes tables en production, cette opération peut causer des interruptions

**Optimisation PostgreSQL :** Certains changements de type ne nécessitent pas de réécriture, par exemple :
- Augmenter la longueur d'un VARCHAR
- Passer de VARCHAR(n) à TEXT

#### 1.2.4. Renommer une Colonne

**Syntaxe :**

```sql
ALTER TABLE nom_table
RENAME COLUMN ancien_nom TO nouveau_nom;
```

**Exemple :**

```sql
ALTER TABLE utilisateurs
RENAME COLUMN prenom TO first_name;
```

Cette opération est **rapide** car elle ne modifie que les métadonnées, pas les données elles-mêmes.

#### 1.2.5. Ajouter une Contrainte

**Syntaxe générale :**

```sql
ALTER TABLE nom_table
ADD CONSTRAINT nom_contrainte type_contrainte (colonnes);
```

**Exemples :**

```sql
-- Ajouter une clé primaire
ALTER TABLE utilisateurs
ADD CONSTRAINT pk_utilisateurs PRIMARY KEY (id);

-- Ajouter une contrainte UNIQUE
ALTER TABLE utilisateurs
ADD CONSTRAINT uk_email UNIQUE (email);

-- Ajouter une clé étrangère
ALTER TABLE commandes
ADD CONSTRAINT fk_utilisateur
FOREIGN KEY (utilisateur_id)
REFERENCES utilisateurs(id);

-- Ajouter une contrainte CHECK
ALTER TABLE produits
ADD CONSTRAINT check_prix_positif
CHECK (prix > 0);

-- Ajouter NOT NULL
ALTER TABLE utilisateurs
ALTER COLUMN email SET NOT NULL;
```

**Attention avec les contraintes :**
- L'ajout d'une contrainte vérifie **toutes les lignes existantes**
- Sur de grandes tables, cela peut prendre beaucoup de temps
- PostgreSQL pose un verrou qui peut bloquer l'accès à la table

**Option NOT VALID (pour les clés étrangères et CHECK) :**

```sql
ALTER TABLE commandes
ADD CONSTRAINT fk_utilisateur
FOREIGN KEY (utilisateur_id)
REFERENCES utilisateurs(id)
NOT VALID;
```

Avec `NOT VALID`, la contrainte est créée sans vérifier les données existantes (rapide), mais elle s'applique aux nouvelles données. Vous pouvez ensuite la valider en arrière-plan :

```sql
ALTER TABLE commandes
VALIDATE CONSTRAINT fk_utilisateur;
```

#### 1.2.6. Supprimer une Contrainte

**Syntaxe :**

```sql
ALTER TABLE nom_table
DROP CONSTRAINT nom_contrainte [CASCADE | RESTRICT];
```

**Exemple :**

```sql
ALTER TABLE utilisateurs
DROP CONSTRAINT uk_email;
```

Pour retirer une contrainte NOT NULL :

```sql
ALTER TABLE utilisateurs
ALTER COLUMN email DROP NOT NULL;
```

#### 1.2.7. Modifier une Valeur par Défaut

**Définir une valeur par défaut :**

```sql
ALTER TABLE utilisateurs
ALTER COLUMN pays SET DEFAULT 'France';
```

**Supprimer une valeur par défaut :**

```sql
ALTER TABLE utilisateurs
ALTER COLUMN pays DROP DEFAULT;
```

**Important :** Modifier la valeur par défaut n'affecte **que les nouvelles lignes**. Les lignes existantes conservent leurs valeurs actuelles.

### 1.3. ALTER SCHEMA, SEQUENCE, TYPE...

**Renommer un schéma :**

```sql
ALTER SCHEMA ancien_nom RENAME TO nouveau_nom;
```

**Modifier une séquence :**

```sql
-- Changer la valeur de départ
ALTER SEQUENCE ma_sequence RESTART WITH 1000;

-- Changer l'incrément
ALTER SEQUENCE ma_sequence INCREMENT BY 10;
```

**Renommer un type personnalisé :**

```sql
ALTER TYPE mon_type RENAME TO nouveau_type;
```

---

## 2. La Commande DROP : Supprimer des Objets

### 2.1. Qu'est-ce que DROP ?

La commande **DROP** permet de supprimer définitivement un objet de la base de données. C'est une opération **irréversible** (sauf restauration depuis une sauvegarde).

### 2.2. DROP TABLE

**Syntaxe :**

```sql
DROP TABLE [IF EXISTS] nom_table [CASCADE | RESTRICT];
```

**Exemple simple :**

```sql
DROP TABLE utilisateurs_temp;
```

**Options importantes :**

- **IF EXISTS** : Évite une erreur si la table n'existe pas

```sql
DROP TABLE IF EXISTS utilisateurs_temp;
-- Ne génère pas d'erreur si la table n'existe pas
```

- **CASCADE** : Supprime automatiquement tous les objets dépendants (vues, clés étrangères, etc.)

```sql
DROP TABLE utilisateurs CASCADE;
-- Supprime aussi les vues qui référencent cette table
```

- **RESTRICT** (par défaut) : Échoue si des objets dépendent de la table

```sql
DROP TABLE utilisateurs RESTRICT;
-- Échoue si des clés étrangères pointent vers cette table
```

**⚠️ Avertissement :** `DROP TABLE` est définitif. Soyez très prudent, surtout avec l'option CASCADE !

### 2.3. Supprimer Plusieurs Tables

Vous pouvez supprimer plusieurs tables en une seule commande :

```sql
DROP TABLE table1, table2, table3 CASCADE;
```

### 2.4. DROP COLUMN vs DROP TABLE

**Différence importante :**

```sql
-- Supprimer une colonne (la table reste)
ALTER TABLE utilisateurs DROP COLUMN email;

-- Supprimer toute la table (données + structure)
DROP TABLE utilisateurs;
```

### 2.5. Autres Objets DROP

PostgreSQL permet de supprimer de nombreux types d'objets :

```sql
-- Supprimer un schéma (doit être vide ou utiliser CASCADE)
DROP SCHEMA mon_schema CASCADE;

-- Supprimer une séquence
DROP SEQUENCE ma_sequence;

-- Supprimer un index
DROP INDEX mon_index;

-- Supprimer une vue
DROP VIEW ma_vue;

-- Supprimer une fonction
DROP FUNCTION ma_fonction(paramètres);

-- Supprimer un type personnalisé
DROP TYPE mon_type;

-- Supprimer un domaine
DROP DOMAIN mon_domaine;
```

### 2.6. Bonnes Pratiques pour DROP

1. **Toujours utiliser IF EXISTS** pour éviter les erreurs dans les scripts :
   ```sql
   DROP TABLE IF EXISTS table_temporaire;
   ```

2. **Faire une sauvegarde** avant de supprimer des objets critiques

3. **Tester d'abord avec RESTRICT** pour voir les dépendances :
   ```sql
   DROP TABLE utilisateurs RESTRICT;
   -- Vous informe des objets dépendants
   ```

4. **Préférer ALTER TABLE ... DROP COLUMN** si vous ne voulez supprimer qu'une colonne

5. **En production**, privilégier une suppression en deux temps :
   - D'abord, renommer l'objet (pour le "désactiver")
   - Puis, après vérification, le supprimer définitivement

---

## 3. Le Verrouillage (Locking) : Comprendre l'Impact

### 3.1. Pourquoi le Verrouillage ?

PostgreSQL est un système **multi-utilisateurs** : plusieurs connexions peuvent travailler simultanément sur la même base. Pour garantir la **cohérence des données** lors des modifications, PostgreSQL utilise un système de **verrous** (locks).

**Principe fondamental :** Quand vous modifiez la structure d'une table (ALTER ou DROP), PostgreSQL pose un verrou pour éviter que d'autres opérations n'interfèrent.

### 3.2. Types de Verrous pour ALTER et DROP

Il existe plusieurs niveaux de verrous en PostgreSQL. Voici les plus importants dans le contexte d'ALTER et DROP :

#### 3.2.1. ACCESS SHARE

- **Le plus permissif**
- Utilisé par les requêtes SELECT simples
- N'empêche aucune autre opération

#### 3.2.2. ACCESS EXCLUSIVE

- **Le plus restrictif**
- Utilisé par DROP TABLE, ALTER TABLE (la plupart des cas)
- **Bloque toutes les autres opérations** sur la table, même les SELECT

**Impact critique :** Pendant qu'un ALTER TABLE est en cours, **personne ne peut accéder à la table**, même en lecture. Sur une application en production, cela signifie une **indisponibilité**.

### 3.3. Quel Type de Verrou pour Quelle Opération ?

| Opération | Type de Verrou | Impact |
|-----------|----------------|--------|
| `SELECT` | ACCESS SHARE | Aucun blocage |
| `INSERT`, `UPDATE`, `DELETE` | ROW EXCLUSIVE | Bloque ALTER/DROP |
| `ALTER TABLE ADD COLUMN` (sans DEFAULT) | ACCESS EXCLUSIVE | Bloque tout |
| `ALTER TABLE ADD COLUMN DEFAULT` (PG 11+) | ACCESS EXCLUSIVE (court) | Bloque brièvement |
| `ALTER TABLE DROP COLUMN` | ACCESS EXCLUSIVE | Bloque tout |
| `ALTER TABLE ALTER COLUMN TYPE` | ACCESS EXCLUSIVE | Bloque tout (long) |
| `ALTER TABLE RENAME COLUMN` | ACCESS EXCLUSIVE | Bloque tout (court) |
| `DROP TABLE` | ACCESS EXCLUSIVE | Bloque tout |
| `CREATE INDEX` (normal) | SHARE | Bloque INSERT/UPDATE/DELETE |
| `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | Moins bloquant |

### 3.4. Durée du Verrouillage : Le Vrai Problème

Le problème n'est pas le verrou lui-même, mais **sa durée**.

**Opérations rapides (< 1 seconde) :**
- Renommer une colonne
- Ajouter une colonne avec DEFAULT (PG 11+)
- Supprimer une contrainte

**Opérations lentes (minutes/heures) :**
- Modifier le type d'une colonne (réécriture de table)
- Ajouter une colonne NOT NULL sur une grande table
- Supprimer une colonne (réécriture de table)

### 3.5. Exemple Concret : Le Drame de l'ALTER TABLE en Production

**Scénario :**

Vous avez une table `commandes` avec 10 millions de lignes. Vous voulez ajouter une colonne NOT NULL.

```sql
ALTER TABLE commandes
ADD COLUMN statut VARCHAR(20) NOT NULL DEFAULT 'en_attente';
```

**Ce qui se passe :**

1. PostgreSQL pose un verrou ACCESS EXCLUSIVE sur la table
2. **Tous les SELECT, INSERT, UPDATE, DELETE sont bloqués**
3. PostgreSQL vérifie toutes les 10 millions de lignes
4. Cette opération peut prendre **plusieurs minutes**
5. Pendant ce temps, votre application est **figée** pour cette table

**Conséquence :** Indisponibilité de service, utilisateurs mécontents, perte de revenus potentielle.

### 3.6. Techniques pour Minimiser l'Impact

#### 3.6.1. Utiliser ADD COLUMN avec DEFAULT (PostgreSQL 11+)

**Ancien comportement (PG 10 et avant) :**

```sql
ALTER TABLE commandes
ADD COLUMN statut VARCHAR(20) DEFAULT 'en_attente';
```

→ Réécrivait toute la table (lent)

**Nouveau comportement (PG 11+) :**

Cette même commande est **instantanée** ! PostgreSQL enregistre juste le défaut dans les métadonnées. La réécriture n'a lieu qu'en arrière-plan si nécessaire.

#### 3.6.2. Ajouter d'abord la Colonne NULL, puis Remplir, puis NOT NULL

**Au lieu de :**

```sql
ALTER TABLE commandes
ADD COLUMN statut VARCHAR(20) NOT NULL DEFAULT 'en_attente';
```

**Faire :**

```sql
-- Étape 1 : Ajouter la colonne (rapide)
ALTER TABLE commandes
ADD COLUMN statut VARCHAR(20) DEFAULT 'en_attente';

-- Étape 2 : Mettre à jour les valeurs NULL en plusieurs batchs (sans verrouillage long)
UPDATE commandes SET statut = 'en_attente' WHERE statut IS NULL;

-- Étape 3 : Ajouter la contrainte NOT NULL (rapide si plus de NULL)
ALTER TABLE commandes
ALTER COLUMN statut SET NOT NULL;
```

#### 3.6.3. Utiliser NOT VALID pour les Contraintes

**Au lieu de :**

```sql
ALTER TABLE commandes
ADD CONSTRAINT check_prix
CHECK (prix > 0);
```

**Faire :**

```sql
-- Étape 1 : Ajouter sans validation (rapide)
ALTER TABLE commandes
ADD CONSTRAINT check_prix
CHECK (prix > 0) NOT VALID;

-- Étape 2 : Valider en arrière-plan (verrou moins bloquant)
ALTER TABLE commandes
VALIDATE CONSTRAINT check_prix;
```

#### 3.6.4. CREATE INDEX CONCURRENTLY

Pour créer un index sans bloquer les écritures :

```sql
CREATE INDEX CONCURRENTLY idx_email ON utilisateurs(email);
```

Cette opération est plus longue, mais n'empêche pas l'application de fonctionner.

#### 3.6.5. Planifier les Opérations Lourdes

- **Fenêtre de maintenance** : Planifiez les ALTER lourds pendant les heures creuses
- **Blue-Green Deployment** : Créez une nouvelle version de la table, basculez, puis supprimez l'ancienne
- **Réplication Logique** : Effectuez la modification sur une réplique, puis basculez

### 3.7. Surveiller les Verrous

**Voir les verrous actifs :**

```sql
SELECT
    locktype,
    relation::regclass AS table_name,
    mode,
    granted,
    pid
FROM pg_locks
WHERE relation IS NOT NULL;
```

**Voir les requêtes bloquées :**

```sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

**Tuer une requête bloquante (avec précaution) :**

```sql
SELECT pg_terminate_backend(pid);
```

---

## 4. Stratégies et Bonnes Pratiques

### 4.1. Avant de Modifier une Table en Production

**Checklist de sécurité :**

1. ✅ **Sauvegarde** : Toujours faire un backup avant une modification lourde
2. ✅ **Tester sur un environnement de staging** identique à la production
3. ✅ **Estimer la durée** : Testez sur une copie de la table de production
4. ✅ **Vérifier les dépendances** : Utilisez `\d+ nom_table` dans psql
5. ✅ **Planifier une fenêtre de maintenance** si nécessaire
6. ✅ **Prévoir un rollback** : Comment annuler si ça tourne mal ?
7. ✅ **Informer l'équipe** : Coordonner avec les développeurs et les ops

### 4.2. Règles d'Or pour ALTER/DROP

1. **Évitez ALTER TABLE sur de grandes tables en production sans préparation**
2. **Utilisez des transactions** pour regrouper plusieurs ALTER :
   ```sql
   BEGIN;
   ALTER TABLE users ADD COLUMN age INTEGER;
   ALTER TABLE users ADD COLUMN city VARCHAR(100);
   COMMIT;
   ```
3. **Préférez les opérations incrémentales** aux grosses modifications atomiques
4. **Utilisez les options modernes** (DEFAULT instantané, NOT VALID, CONCURRENTLY)
5. **Testez d'abord avec RESTRICT** pour voir les dépendances

### 4.3. Cas Particulier : Renommer vs Supprimer

**Ne supprimez pas immédiatement** une colonne ou table en production. Préférez un processus en deux étapes :

**Étape 1 : Renommer (réversible)**
```sql
ALTER TABLE utilisateurs RENAME TO utilisateurs_old;
```

**Étape 2 : Après vérification (plusieurs jours/semaines), supprimer**
```sql
DROP TABLE utilisateurs_old;
```

### 4.4. Outils et Extensions Utiles

- **pg_repack** : Réorganise les tables sans verrouillage exclusif
- **pg_squeeze** : Alternative à VACUUM FULL sans bloquer
- **pgAdmin / DBeaver** : Interfaces graphiques qui montrent les dépendances avant suppression

---

## 5. Exemples Pratiques Commentés

### Exemple 1 : Migration de Schéma Sécurisée

**Contexte :** Vous voulez ajouter une colonne `last_login` à une grande table `users`.

```sql
-- ✅ Bonne approche
BEGIN;

-- 1. Ajouter la colonne (rapide avec DEFAULT)
ALTER TABLE users
ADD COLUMN last_login TIMESTAMP DEFAULT CURRENT_TIMESTAMP;

-- 2. Créer un index en différé (si besoin)
-- On le fera avec CONCURRENTLY en dehors de la transaction

COMMIT;

-- 3. Index concurrent (ne bloque pas les écritures)
CREATE INDEX CONCURRENTLY idx_last_login ON users(last_login);
```

### Exemple 2 : Supprimer une Colonne en Sécurité

```sql
-- ✅ Processus en deux temps
-- Phase 1 : Renommer (peut être annulé facilement)
ALTER TABLE products RENAME COLUMN old_price TO deprecated_old_price;

-- Attendre quelques jours, surveiller les erreurs applicatives

-- Phase 2 : Supprimer définitivement
ALTER TABLE products DROP COLUMN deprecated_old_price;
```

### Exemple 3 : Changer le Type d'une Colonne

```sql
-- ❌ Mauvaise approche (bloque longtemps)
ALTER TABLE orders
ALTER COLUMN total TYPE NUMERIC(10,2);

-- ✅ Bonne approche : Créer nouvelle colonne, migrer, renommer
BEGIN;

-- 1. Nouvelle colonne
ALTER TABLE orders ADD COLUMN total_new NUMERIC(10,2);

-- 2. Migrer les données (peut être fait par batchs)
UPDATE orders SET total_new = total::NUMERIC(10,2);

-- 3. Renommer
ALTER TABLE orders RENAME COLUMN total TO total_old;
ALTER TABLE orders RENAME COLUMN total_new TO total;

-- 4. Supprimer l'ancienne (plus tard)
-- ALTER TABLE orders DROP COLUMN total_old;

COMMIT;
```

---

## 6. Résumé des Points Clés

### Ce qu'il faut retenir

1. **ALTER** modifie des objets existants, **DROP** les supprime définitivement
2. Ces opérations peuvent **poser des verrous** qui bloquent l'accès à la base
3. Certaines opérations sont **rapides** (renommer), d'autres **très lentes** (changer un type de colonne)
4. PostgreSQL 11+ a apporté des **optimisations majeures** (ADD COLUMN DEFAULT instantané)
5. En production, **toujours tester** et **planifier** les modifications de schéma
6. Utilisez les techniques modernes : **NOT VALID**, **CONCURRENTLY**, opérations **incrémentales**
7. **Sauvegardez** avant toute opération destructive
8. **Surveillez les verrous** et l'impact sur l'application

### Tableau Récapitulatif

| Opération | Vitesse | Verrou | Impact Production | Recommandation |
|-----------|---------|--------|-------------------|----------------|
| ADD COLUMN (NULL) | ⚡ Rapide | Court | ✅ Faible | OK en production |
| ADD COLUMN DEFAULT (PG 11+) | ⚡ Rapide | Court | ✅ Faible | OK en production |
| DROP COLUMN | 🐌 Lent | Long | ❌ Élevé | Fenêtre de maintenance |
| ALTER TYPE | 🐌 Très lent | Très long | ❌ Très élevé | Approche alternative |
| RENAME COLUMN | ⚡ Instantané | Court | ✅ Négligeable | OK en production |
| DROP TABLE | ⚡ Rapide | Court | ⚠️ Moyen | Vérifier dépendances |
| ADD CONSTRAINT | 🐌 Lent | Long | ❌ Élevé | Utiliser NOT VALID |

---

## 7. Pour Aller Plus Loin

### Lectures Recommandées

- Documentation officielle PostgreSQL : [ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html)
- Documentation officielle PostgreSQL : [DROP TABLE](https://www.postgresql.org/docs/current/sql-droptable.html)
- [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) : Comprendre les verrous en détail

### Concepts Liés

- **MVCC (Multiversion Concurrency Control)** : Le système de gestion de la concurrence de PostgreSQL (Chapitre 12)
- **Transactions** : Garantir l'atomicité des opérations (Chapitre 12)
- **VACUUM** : Maintenance et récupération d'espace (Chapitre 16)
- **Réplication** : Effectuer des migrations sans downtime (Chapitre 17)

---

## Conclusion

La gestion des modifications de schéma est un aspect **critique** de l'administration PostgreSQL. Les commandes ALTER et DROP sont puissantes mais potentiellement dangereuses si utilisées sans précaution.

**La règle d'or** : En développement, vous pouvez expérimenter librement. En production, chaque modification de schéma doit être **planifiée, testée et exécutée avec soin**.

Avec les techniques modernes (PostgreSQL 11+) et une bonne compréhension du verrouillage, vous pouvez effectuer la plupart des modifications de schéma avec un **impact minimal** sur vos applications en production.

Dans le prochain chapitre, nous aborderons le langage SQL de manipulation et d'interrogation des données (DQL et DML), qui vous permettra d'exploiter pleinement les structures que vous venez d'apprendre à créer et modifier.

---


⏭️ [Requêtes de Sélection (DQL)](/05-requetes-de-selection/README.md)
