🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15. Programmation Serveur (PL/pgSQL)

## Introduction : Au-delà du SQL Standard

Jusqu'à présent dans ce tutoriel, nous avons exploré le **SQL standard** : les requêtes SELECT, les jointures, les agrégations, les transactions, etc. Mais PostgreSQL offre bien plus qu'un simple langage de requête. Il propose un véritable **environnement de programmation côté serveur**.

### Qu'est-ce que la programmation serveur ?

La **programmation serveur** consiste à écrire du code qui s'exécute **directement dans la base de données**, plutôt que dans votre application (Python, Java, Node.js, etc.).

**Analogie simple** :
- **SQL standard** = Poser une question à quelqu'un : "Combien font 2+2 ?"
- **Programmation serveur** = Donner une série d'instructions : "Si le résultat est positif, fais ceci, sinon fais cela, puis répète 10 fois..."

### Pourquoi programmer dans la base de données ?

Vous vous demandez peut-être : "Pourquoi écrire du code dans PostgreSQL alors que je peux tout faire dans mon application ?"

Voici les raisons principales :

#### 1. **Performance** : Réduire les allers-retours réseau

```
❌ Sans programmation serveur (application) :
┌─────────────┐                           ┌──────────────┐
│ Application │ ─── SELECT ligne 1 ────►  │              │
│             │ ◄─── Résultat 1 ───────   │              │
│             │                           │              │
│             │ ─── SELECT ligne 2 ────►  │  PostgreSQL  │
│             │ ◄─── Résultat 2 ───────   │              │
│             │                           │              │
│             │ ─── SELECT ligne 3 ────►  │              │
│             │ ◄─── Résultat 3 ───────   │              │
└─────────────┘                           └──────────────┘
1000 requêtes = 1000 allers-retours = LENT !

✅ Avec programmation serveur (fonction/procédure) :
┌─────────────┐                           ┌──────────────┐
│ Application │ ─── CALL ma_procedure()─► │              │
│             │                           │  Traite les  │
│             │                           │  1000 lignes │
│             │                           │  en interne  │
│             │ ◄─── Résultat final ───── │              │
└─────────────┘                           └──────────────┘
1 seul aller-retour = RAPIDE !
```

#### 2. **Encapsulation** : Logique métier centralisée

Au lieu d'avoir la même logique métier répétée dans chaque application (web, mobile, API), vous la centralisez dans la base de données.

```sql
-- Logique de calcul de prix complexe centralisée
CREATE FUNCTION calculer_prix_final(...)
RETURNS NUMERIC
AS $$
    -- Règles métier complexes
    -- Remises, promotions, TVA, etc.
$$;

-- Toutes vos applications utilisent la même fonction
-- → Cohérence garantie
```

#### 3. **Sécurité** : Contrôle d'accès fin

Vous pouvez donner accès à une fonction sans donner accès direct aux tables :

```sql
-- L'utilisateur ne peut PAS faire : SELECT * FROM salaires;
-- Mais il PEUT faire : SELECT moyenne_salaires_anonymisee();
```

#### 4. **Intégrité** : Garantir la cohérence des données

Certaines opérations complexes doivent être **atomiques** (tout ou rien) et sont mieux gérées côté base :

```sql
-- Transfert bancaire : débiter ET créditer en une seule opération atomique
CALL transferer_argent(compte_source, compte_dest, montant);
```

---

## PL/pgSQL : Le Langage Procédural de PostgreSQL

### Qu'est-ce que PL/pgSQL ?

**PL/pgSQL** signifie **Procedural Language/PostgreSQL**. C'est un langage de programmation spécialement conçu pour PostgreSQL qui étend le SQL avec :

- ✅ **Variables** : Stocker des valeurs temporaires
- ✅ **Structures de contrôle** : IF/ELSE, CASE, boucles (LOOP, WHILE, FOR)
- ✅ **Gestion d'erreurs** : Capturer et traiter les exceptions
- ✅ **Fonctions et procédures** : Encapsuler du code réutilisable
- ✅ **Triggers** : Réagir automatiquement aux changements de données

### SQL vs PL/pgSQL : Quelle différence ?

| Aspect | SQL Standard | PL/pgSQL |
|--------|-------------|----------|
| **Nature** | Déclaratif ("Quoi faire") | Procédural ("Comment le faire") |
| **Variables** | ❌ Non | ✅ Oui |
| **Conditions** | ⚠️ CASE limité | ✅ IF/ELSIF/ELSE complet |
| **Boucles** | ❌ Non | ✅ LOOP, WHILE, FOR |
| **Gestion d'erreurs** | ❌ Non | ✅ BEGIN...EXCEPTION...END |
| **Exemple** | `SELECT SUM(prix) FROM produits` | `FOR r IN SELECT * FROM produits LOOP ... END LOOP;` |

### Exemple simple pour comprendre

**En SQL standard** (déclaratif) :
```sql
-- Je veux la somme des prix
SELECT SUM(prix) FROM produits WHERE categorie = 'Livres';

-- C'est direct, mais limité
```

**En PL/pgSQL** (procédural) :
```sql
DECLARE
    total NUMERIC := 0;
    prix_courant NUMERIC;
BEGIN
    -- Je parcours les produits un par un
    FOR prix_courant IN SELECT prix FROM produits WHERE categorie = 'Livres' LOOP
        -- J'applique une logique personnalisée
        IF prix_courant > 100 THEN
            total := total + prix_courant * 0.9;  -- 10% de remise
        ELSE
            total := total + prix_courant;
        END IF;
    END LOOP;

    RETURN total;
END;

-- Plus complexe, mais beaucoup plus flexible !
```

---

## Les Objets de Programmation Serveur

PostgreSQL propose plusieurs types d'objets programmables :

### 1. **Fonctions** (Functions)

Une fonction **retourne toujours une valeur**.

```sql
CREATE FUNCTION calculer_tva(prix_ht NUMERIC, taux NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN prix_ht * (1 + taux / 100);
END;
$$;

-- Utilisation
SELECT calculer_tva(100, 20);  -- Retourne 120
```

**Caractéristiques** :
- ✅ Retourne obligatoirement une valeur (ou un ensemble de valeurs)
- ✅ Peut être utilisée dans un SELECT
- ✅ S'exécute dans la transaction de l'appelant
- ❌ Ne peut pas faire de COMMIT/ROLLBACK interne

### 2. **Procédures** (Procedures)

Une procédure **ne retourne pas de valeur** mais peut contrôler les transactions.

```sql
CREATE PROCEDURE archiver_anciennes_commandes()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO archive SELECT * FROM commandes WHERE date < NOW() - INTERVAL '1 year';
    COMMIT;  -- ✅ Possible dans une procédure !

    DELETE FROM commandes WHERE date < NOW() - INTERVAL '1 year';
    COMMIT;
END;
$$;

-- Utilisation
CALL archiver_anciennes_commandes();
```

**Caractéristiques** :
- ❌ Ne retourne pas de valeur (sauf paramètres OUT/INOUT)
- ❌ Ne peut pas être utilisée dans un SELECT
- ✅ Peut faire des COMMIT/ROLLBACK internes
- ✅ Idéale pour les traitements par lots

### 3. **Triggers** (Déclencheurs)

Un trigger est une fonction qui s'exécute **automatiquement** lors d'événements sur une table.

```sql
-- Fonction trigger
CREATE FUNCTION log_modifications()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO audit_log (table_name, action, timestamp)
    VALUES (TG_TABLE_NAME, TG_OP, NOW());
    RETURN NEW;
END;
$$;

-- Associer le trigger à une table
CREATE TRIGGER audit_clients
AFTER INSERT OR UPDATE OR DELETE ON clients
FOR EACH ROW EXECUTE FUNCTION log_modifications();
```

**Caractéristiques** :
- ✅ S'exécute automatiquement (INSERT, UPDATE, DELETE)
- ✅ Accès aux anciennes (OLD) et nouvelles (NEW) valeurs
- ✅ Peut modifier les données avant insertion (BEFORE trigger)
- ✅ Utile pour l'audit, validation, synchronisation

### 4. **Event Triggers** (Déclencheurs d'événements)

Un event trigger réagit aux modifications de **structure** (DDL) :

```sql
CREATE FUNCTION audit_ddl()
RETURNS EVENT_TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO ddl_history (command, user, timestamp)
    VALUES (TG_TAG, current_user, NOW());
END;
$$;

CREATE EVENT TRIGGER log_ddl_commands
ON ddl_command_end
EXECUTE FUNCTION audit_ddl();
```

**Caractéristiques** :
- ✅ Surveille CREATE, ALTER, DROP, etc.
- ✅ Audit de changements de schéma
- ✅ Protection contre suppressions accidentelles

---

## Quand Utiliser la Programmation Serveur ?

### ✅ Utilisez-la quand :

1. **Logique métier complexe** nécessitant conditions et boucles
2. **Traitement de gros volumes** (éviter des milliers d'allers-retours)
3. **Opérations atomiques** critiques (tout ou rien)
4. **Audit et traçabilité** automatiques (triggers)
5. **Calculs intensifs** sur des données déjà en base
6. **Réutilisation** de logique entre plusieurs applications
7. **Sécurité** : limiter l'accès direct aux tables

### ❌ N'en abusez pas quand :

1. **Logique d'interface utilisateur** (mieux dans l'application)
2. **Logique métier changeante fréquemment** (plus facile à déployer côté app)
3. **Calculs nécessitant des bibliothèques externes** (mieux dans l'app)
4. **Tests unitaires complexes** (plus facile côté application)
5. **Scalabilité horizontale** nécessaire (base de données = point unique)

---

## Avantages et Inconvénients de la Programmation Serveur

### Avantages ✅

| Avantage | Explication |
|----------|-------------|
| **Performance** | Pas d'allers-retours réseau pour chaque opération |
| **Centralisation** | Une seule source de vérité pour la logique métier |
| **Sécurité** | Contrôle d'accès granulaire via fonctions |
| **Cohérence** | Garantit l'intégrité des données complexes |
| **Réutilisabilité** | Toutes les applications utilisent le même code |
| **Atomicité** | Transactions gérées au plus près des données |

### Inconvénients ❌

| Inconvénient | Explication |
|--------------|-------------|
| **Couplage** | Applications dépendantes de la structure de la base |
| **Débogage** | Moins d'outils que pour le code applicatif |
| **Versioning** | Gestion des versions plus complexe |
| **Tests** | Tests unitaires moins faciles |
| **Portabilité** | Code spécifique à PostgreSQL (PL/pgSQL) |
| **Scalabilité** | Base de données = point de contention potentiel |

---

## Les Langages Disponibles dans PostgreSQL

PostgreSQL ne se limite pas à PL/pgSQL. Plusieurs langages sont disponibles :

### 1. **PL/pgSQL** (par défaut)
- ✅ Installé par défaut
- ✅ Syntaxe proche du SQL
- ✅ Performance optimale
- ✅ Intégration native

### 2. **SQL pur**
- ✅ Encore plus simple que PL/pgSQL
- ✅ Fonctions très performantes
- ⚠️ Limité (pas de structures de contrôle)

### 3. **PL/Python** (extension)
- ✅ Puissance de Python
- ✅ Accès à des bibliothèques Python
- ⚠️ Moins performant que PL/pgSQL
- ⚠️ Risques de sécurité

### 4. **PL/Perl** (extension)
- ✅ Expressions régulières puissantes
- ✅ Manipulation de texte avancée
- ⚠️ Moins populaire aujourd'hui

### 5. **PL/v8** (extension)
- ✅ JavaScript côté serveur
- ✅ Familier pour les développeurs web
- ⚠️ Maintenance moins active

### 6. **C** (pour experts)
- ✅ Performance maximale
- ✅ Accès bas niveau
- ❌ Complexe et dangereux

**Recommandation pour débutants** : Commencez avec **PL/pgSQL** (couvert dans ce chapitre) ou **SQL pur** pour les cas simples.

---

## Structure de ce Chapitre

Ce chapitre couvre tous les aspects de la programmation serveur en PostgreSQL :

### **15.1. Fonctions SQL et PL/pgSQL : Différences et cas d'usage**
Comparaison des deux approches et guide de décision.

### **15.2. Catégories de volatilité : VOLATILE, STABLE, IMMUTABLE**
Optimiser les performances en déclarant correctement vos fonctions.

### **15.3. Procédures stockées et gestion transactionnelle (CALL, COMMIT/ROLLBACK)**
Maîtriser les procédures et le contrôle des transactions.

### **15.4. Triggers (BEFORE, AFTER, INSTEAD OF)**
Automatiser des actions en réponse aux modifications de données.

### **15.5. Event Triggers : Surveillance DDL**
Surveiller et contrôler les modifications de structure.

### **15.6. Gestion des exceptions (BEGIN...EXCEPTION...END)**
Capturer et gérer les erreurs élégamment.

### **15.7. Autres langages procéduraux**
Aperçu de PL/Python, PL/Perl, et autres alternatives.

---

## Conventions et Bonnes Pratiques

Avant de commencer, voici quelques conventions que nous suivrons :

### 1. **Nommage**

```sql
-- ✅ BON : Noms explicites
CREATE FUNCTION calculer_prix_ttc(...)
CREATE PROCEDURE archiver_commandes_anciennes()

-- ❌ ÉVITER : Noms vagues
CREATE FUNCTION calc(...)
CREATE PROCEDURE proc1()
```

### 2. **Documentation**

```sql
CREATE FUNCTION ma_fonction(param INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
-- Cette fonction fait ceci et cela
-- Paramètres :
--   param : Description du paramètre
-- Retour :
--   Description de ce qui est retourné
BEGIN
    ...
END;
$$;

-- Ajouter un commentaire PostgreSQL
COMMENT ON FUNCTION ma_fonction(INTEGER) IS
'Calcule le résultat en fonction du paramètre fourni.';
```

### 3. **Gestion d'erreurs**

```sql
-- ✅ Toujours gérer les erreurs potentielles
BEGIN
    -- Code
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Division par zéro !';
        RETURN NULL;
END;
```

### 4. **Indentation et lisibilité**

```sql
-- ✅ BON : Code indenté et lisible
CREATE FUNCTION exemple()
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    compteur INTEGER := 0;
BEGIN
    FOR i IN 1..10 LOOP
        IF i % 2 = 0 THEN
            compteur := compteur + 1;
        END IF;
    END LOOP;

    RETURN compteur;
END;
$$;
```

---

## Premier Exemple Complet : Découverte de PL/pgSQL

Voici un exemple simple mais complet pour illustrer la puissance de PL/pgSQL :

```sql
-- Fonction qui calcule des statistiques sur les commandes d'un client
CREATE FUNCTION stats_client(client_id INTEGER)
RETURNS TABLE(
    nb_commandes INTEGER,
    montant_total NUMERIC,
    montant_moyen NUMERIC,
    derniere_commande DATE
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_nb INTEGER;
    v_total NUMERIC;
BEGIN
    -- Compter le nombre de commandes
    SELECT COUNT(*), COALESCE(SUM(montant), 0)
    INTO v_nb, v_total
    FROM commandes
    WHERE id_client = client_id;

    -- Si aucune commande, retourner des valeurs nulles
    IF v_nb = 0 THEN
        RETURN QUERY SELECT 0, 0::NUMERIC, 0::NUMERIC, NULL::DATE;
        RETURN;
    END IF;

    -- Retourner les statistiques
    RETURN QUERY
    SELECT
        v_nb,
        v_total,
        v_total / v_nb,
        MAX(date_commande)::DATE
    FROM commandes
    WHERE id_client = client_id;
END;
$$;

-- Utilisation
SELECT * FROM stats_client(42);
```

**Ce que cet exemple démontre** :
- ✅ Déclaration de variables (`DECLARE`)
- ✅ Conditions (`IF...THEN...END IF`)
- ✅ Récupération de valeurs (`INTO`)
- ✅ Retour de résultats tabulaires (`RETURN QUERY`)
- ✅ Logique métier (gestion du cas "aucune commande")

---

## Outils pour Travailler avec PL/pgSQL

### 1. **psql** (ligne de commande)

```sql
-- Créer une fonction
\e  -- Ouvre un éditeur pour écrire du code

-- Voir les fonctions
\df  -- Liste toutes les fonctions
\df+ ma_fonction  -- Détails d'une fonction

-- Voir le code source
\sf ma_fonction
```

### 2. **pgAdmin** (interface graphique)

- Éditeur de code avec coloration syntaxique
- Débogueur intégré pour fonctions PL/pgSQL
- Navigation dans les objets

### 3. **DBeaver** (interface graphique)

- Support multi-bases
- Auto-complétion
- Visualisation des dépendances

### 4. **Extensions VS Code**

- `PostgreSQL` pour la coloration syntaxique
- Intégration avec PostgreSQL

---

## Ressources Complémentaires

### Documentation officielle PostgreSQL
- [PL/pgSQL Guide](https://www.postgresql.org/docs/current/plpgsql.html)
- [Function Creation](https://www.postgresql.org/docs/current/sql-createfunction.html)
- [Procedure Creation](https://www.postgresql.org/docs/current/sql-createprocedure.html)

### Livres recommandés
- **"PostgreSQL: Up and Running"** - Regina Obe & Leo Hsu
- **"The Art of PostgreSQL"** - Dimitri Fontaine
- **"Mastering PostgreSQL"** - Hans-Jürgen Schönig

### Communautés
- PostgreSQL mailing lists
- Stack Overflow (tag: postgresql)
- Reddit : r/PostgreSQL
- Discord PostgreSQL

---

## Récapitulatif : Prêt pour la Programmation Serveur

Avant de plonger dans les détails techniques des sections suivantes, assurez-vous de bien comprendre :

- [ ] La différence entre SQL standard et PL/pgSQL
- [ ] Pourquoi programmer côté serveur (performance, centralisation, sécurité)
- [ ] Les différents objets : fonctions, procédures, triggers, event triggers
- [ ] Quand utiliser la programmation serveur (et quand l'éviter)
- [ ] Les avantages et inconvénients de cette approche
- [ ] Les bonnes pratiques de nommage et documentation

**Vous êtes maintenant prêt à explorer la programmation serveur PostgreSQL !**

Les sections suivantes vous guideront pas à pas, du plus simple (fonctions SQL) au plus avancé (triggers et gestion d'exceptions), avec de nombreux exemples pratiques et des explications détaillées pour débutants.

---

## Schéma Récapitulatif

```
┌────────────────────────────────────────────────────────────────┐
│           PROGRAMMATION SERVEUR POSTGRESQL                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  POURQUOI ?                                                    │
│  ├─ Performance (moins d'allers-retours)                       │
│  ├─ Centralisation (une seule source de vérité)                │
│  ├─ Sécurité (contrôle d'accès fin)                            │
│  └─ Intégrité (garantir la cohérence)                          │
│                                                                │
│  QUOI ?                                                        │
│  ├─ Fonctions       → Retournent des valeurs                   │
│  ├─ Procédures      → Contrôlent les transactions              │
│  ├─ Triggers        → Réagissent aux changements (données)     │
│  └─ Event Triggers  → Réagissent aux changements (structure)   │
│                                                                │
│  COMMENT ?                                                     │
│  ├─ PL/pgSQL (langage procédural PostgreSQL)                   │
│  ├─ SQL pur (fonctions simples)                                │
│  └─ Autres langages (Python, Perl, JavaScript...)              │
│                                                                │
│  QUAND ?                                                       │
│  ✅ Logique métier complexe                                    │
│  ✅ Gros volumes de données                                    │
│  ✅ Opérations critiques atomiques                             │
│  ❌ Logique d'interface utilisateur                            │
│  ❌ Déploiements fréquents                                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

**🎓 Prochaine étape :**
- **15.1. Fonctions SQL et PL/pgSQL : Différences et cas d'usage**

Nous allons maintenant entrer dans le vif du sujet en comparant les deux approches principales de programmation serveur dans PostgreSQL !

---


⏭️ [Fonctions SQL et PL/pgSQL : Différences et cas d'usage](/15-programmation-serveur/01-fonctions-sql-et-plpgsql.md)
