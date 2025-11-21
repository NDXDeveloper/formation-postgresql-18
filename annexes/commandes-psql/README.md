🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Commandes psql Essentielles

## Introduction

Bienvenue dans le guide complet de **psql**, l'interface en ligne de commande (CLI) officielle de PostgreSQL. Maîtriser psql est une compétence essentielle pour tout développeur ou administrateur travaillant avec PostgreSQL, car c'est l'outil le plus direct, puissant et universel pour interagir avec vos bases de données.

### Qu'est-ce que psql ?

**psql** (PostgreSQL interactive terminal) est le client officiel en ligne de commande de PostgreSQL. C'est un programme qui vous permet de :

- 🔍 **Explorer** la structure de vos bases de données
- 📝 **Écrire et exécuter** des requêtes SQL
- 📊 **Analyser** les résultats et les performances
- 💾 **Importer et exporter** des données
- ⚙️ **Administrer** vos serveurs PostgreSQL
- 🔧 **Automatiser** des tâches via des scripts

**Analogie** : Si PostgreSQL était une voiture, psql serait votre tableau de bord principal. Certes, il existe des interfaces graphiques (GUI) plus jolies, mais psql est :
- Toujours disponible (installé avec PostgreSQL)
- Le plus rapide et le plus léger
- Scriptable et automatisable
- Accessible à distance via SSH
- Universel (même interface sur tous les OS)

---

## Pourquoi maîtriser psql ?

### Pour les Développeurs

- ✅ **Productivité** : Les méta-commandes de psql sont plus rapides que n'importe quelle GUI
- ✅ **Debugging** : Exécution précise et contrôle total sur vos requêtes
- ✅ **Portabilité** : Fonctionne partout, même sur des serveurs distants sans interface graphique
- ✅ **Scripts** : Automatisation de tâches répétitives (migrations, tests, backups)
- ✅ **Compréhension** : Voir exactement ce qui se passe, sans abstraction

**Exemple concret** : Besoin de voir rapidement toutes les tables d'une base ? En GUI, clic-clic-clic dans les menus. Avec psql : `\dt` ✅

---

### Pour les DevOps et SRE

- ✅ **SSH** : Administrez vos serveurs PostgreSQL via SSH, aucune GUI nécessaire
- ✅ **Monitoring** : Surveillez en temps réel avec `\watch`
- ✅ **Automatisation** : Intégrez psql dans vos scripts shell, CI/CD, Ansible, etc.
- ✅ **Troubleshooting** : Diagnostiquez rapidement les problèmes de production
- ✅ **Backups** : Export/import de données avec `\copy`

**Exemple concret** : Serveur de production lent à 3h du matin ? Connectez-vous en SSH, lancez psql, et diagnostiquez avec quelques méta-commandes. Pas besoin d'installer quoi que ce soit.

---

### Pour les DBAs

- ✅ **Inspection** : Exploration complète du schéma, index, statistiques
- ✅ **Maintenance** : VACUUM, ANALYZE, REINDEX directement
- ✅ **Sécurité** : Gestion fine des permissions et rôles
- ✅ **Performance** : EXPLAIN détaillé, pg_stat_statements, métriques
- ✅ **Documentation** : Génération de rapports et exports

---

## Les Méta-commandes : La Puissance de psql

### Qu'est-ce qu'une Méta-commande ?

Les **méta-commandes** (ou commandes internes) sont des commandes spécifiques à psql, préfixées par un **backslash** `\`. Elles ne font pas partie du langage SQL mais sont des raccourcis puissants pour interagir avec PostgreSQL.

**Différence fondamentale** :

```sql
-- Ceci est du SQL (langage standard, se termine par ;)
SELECT * FROM clients;

-- Ceci est une méta-commande psql (commence par \, pas de ;)
\dt
```

### Pourquoi des Méta-commandes ?

Les méta-commandes offrent des raccourcis pour des tâches courantes qui nécessiteraient autrement des requêtes SQL complexes.

**Exemple** : Lister toutes les tables

```sql
-- Sans méta-commande (SQL complexe)
SELECT
    schemaname,
    tablename,
    tableowner
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;

-- Avec méta-commande (simple et rapide)
\dt
```

**Résultat** : Exactement les mêmes informations, mais en une commande de 3 caractères !

---

### Caractéristiques des Méta-commandes

1. **Préfixe backslash** : Toutes commencent par `\`
2. **Pas de point-virgule** : Exécutées immédiatement, pas besoin de `;`
3. **Insensibles à la casse** : `\DT`, `\dt`, et `\Dt` font la même chose
4. **Variante détaillée** : Ajoutez `+` pour plus d'informations (ex: `\dt+`)
5. **Patterns** : Acceptent des wildcards `*` et `?` pour filtrer

**Exemples** :

```bash
\dt              # Toutes les tables
\dt+             # Toutes les tables avec détails (taille, description)
\dt client*      # Tables commençant par "client"
\dt *log*        # Tables contenant "log"
```

---

## Organisation de ce Guide

Ce guide est organisé en **5 sections progressives**, de la navigation de base aux techniques avancées. Chaque section peut être lue indépendamment, mais nous recommandons de les parcourir dans l'ordre si vous débutez avec psql.

### 📘 Section 1 : Navigation

**Objectif** : Apprendre à explorer et comprendre la structure de vos bases de données.

**Commandes clés** :
- `\l` : Lister les bases de données
- `\c` : Se connecter à une base
- `\dt` : Lister les tables
- `\d` : Décrire un objet (table, vue, index...)
- `\di` : Lister les index
- `\dv` : Lister les vues
- `\du` : Lister les utilisateurs
- `\df` : Lister les fonctions
- Et bien d'autres...

**Pour qui** : Débutants et intermédiaires
**Durée de lecture** : 30-45 minutes
**Niveau de difficulté** : 🟢 Facile

**Vous apprendrez à** :
- Explorer une base de données inconnue
- Comprendre la hiérarchie des objets (database → schema → table)
- Inspecter la structure des tables et leurs relations
- Voir les permissions et les utilisateurs
- Naviguer efficacement entre les schémas

---

### 📘 Section 2 : Configuration

**Objectif** : Personnaliser psql pour améliorer votre confort et votre productivité.

**Commandes clés** :
- `\x` : Affichage étendu (vertical)
- `\timing` : Mesurer le temps d'exécution
- `\pset` : Formatter l'affichage (bordures, null, séparateurs...)
- `\set` : Définir des variables
- `\echo` : Afficher des messages
- Configuration du prompt
- Fichier `.psqlrc`

**Pour qui** : Tous niveaux
**Durée de lecture** : 45-60 minutes
**Niveau de difficulté** : 🟡 Intermédiaire

**Vous apprendrez à** :
- Rendre les résultats plus lisibles
- Mesurer les performances de vos requêtes
- Créer un environnement personnalisé et confortable
- Automatiser votre configuration avec `.psqlrc`
- Utiliser des variables pour des scripts dynamiques

---

### 📘 Section 3 : Export/Import

**Objectif** : Maîtriser l'import et l'export de données entre fichiers et PostgreSQL.

**Commandes clés** :
- `\copy` : Importer/exporter des données (CSV, TSV...)
- `\o` : Rediriger la sortie vers un fichier
- `\i` : Exécuter un script SQL
- `\ir` : Exécuter avec chemin relatif

**Pour qui** : Tous niveaux
**Durée de lecture** : 60-75 minutes
**Niveau de difficulté** : 🟡 Intermédiaire

**Vous apprendrez à** :
- Exporter des données vers Excel ou d'autres outils
- Importer des données depuis des fichiers CSV
- Sauvegarder des résultats de requêtes
- Créer des scripts SQL réutilisables
- Automatiser des migrations de données
- Gérer les erreurs à l'import

---

### 📘 Section 4 : Méta-commandes Avancées

**Objectif** : Découvrir les techniques de pro pour la productivité maximale.

**Commandes clés** :
- `\watch` : Exécution répétée (monitoring temps réel)
- `\gx`, `\gset`, `\gexec` : Variantes d'exécution avancées
- `\sf`, `\sv` : Visualiser le code source
- `\ef`, `\ev` : Éditer fonctions et vues
- `\e` : Éditer dans un éditeur externe
- `\s` : Gestion de l'historique
- Variables et scripts conditionnels

**Pour qui** : Intermédiaires et avancés
**Durée de lecture** : 75-90 minutes
**Niveau de difficulté** : 🔴 Avancé

**Vous apprendrez à** :
- Surveiller votre base en temps réel
- Générer et exécuter du SQL dynamiquement
- Créer des scripts puissants et réutilisables
- Éditer du code directement dans votre éditeur favori
- Automatiser des tâches complexes
- Déboguer efficacement

---

## Comment Utiliser ce Guide

### 🎯 Approche Progressive (Recommandé pour Débutants)

**Semaine 1 : Les Bases**
1. Lisez la **Section 1 : Navigation**
2. Pratiquez sur une base de test : créez des tables, explorez-les avec `\dt`, `\d`, etc.
3. Testez toutes les commandes de navigation

**Semaine 2 : Le Confort**
1. Lisez la **Section 2 : Configuration**
2. Créez votre fichier `.psqlrc` avec vos préférences
3. Activez `\timing` et `\x auto` par défaut

**Semaine 3 : Les Données**
1. Lisez la **Section 3 : Export/Import**
2. Exportez une table vers CSV
3. Créez votre premier script SQL avec `\i`

**Semaine 4 : La Maîtrise**
1. Lisez la **Section 4 : Méta-commandes Avancées**
2. Testez `\watch` pour surveiller une requête
3. Créez un script avec variables et conditions

---

### 🎓 Approche par Besoin (Pour Utilisateurs Expérimentés)

**"Je veux juste explorer une base rapidement"**
→ Section 1 : Navigation (`\l`, `\dt`, `\d`)

**"Je veux un psql confortable et productif"**
→ Section 2 : Configuration (`.psqlrc`, `\timing`, `\x`)

**"Je dois importer/exporter des données"**
→ Section 3 : Export/Import (`\copy`, `\o`)

**"Je veux automatiser et scripter"**
→ Section 4 : Méta-commandes Avancées (`\gset`, `\gexec`, `\watch`)

---

### 📚 Approche Référence (Consultation Ponctuelle)

Chaque section est autonome et peut servir de référence :
- **Table des matières détaillée** au début de chaque section
- **Exemples concrets** pour chaque commande
- **Tableaux récapitulatifs** pour référence rapide
- **Index des commandes** à la fin

**Usage** : Gardez ce guide ouvert à côté de votre terminal et consultez-le au besoin.

---

## Prérequis

### Ce Que Vous Devez Savoir

**Minimum requis** :
- ✅ PostgreSQL installé (version 12+, idéalement 18)
- ✅ Connaissance de base de SQL (SELECT, INSERT, UPDATE, DELETE)
- ✅ Accès à un terminal (Linux, macOS, ou Windows avec PowerShell/WSL)
- ✅ Une base de données de test (même vide)

**Recommandé** :
- 📖 Avoir lu les parties 1-3 du tutoriel principal (concepts fondamentaux)
- 📖 Comprendre les notions de table, schéma, index
- 🖥️ Être à l'aise avec la ligne de commande

**Pas nécessaire** :
- ❌ Expertise en SQL avancé
- ❌ Connaissance d'administration système
- ❌ Expérience avec d'autres SGBD

---

### Installation et Vérification

#### Vérifier que psql est installé

```bash
# Vérifier la version de psql
psql --version

# Résultat attendu :
# psql (PostgreSQL) 18.0
```

Si psql n'est pas installé, installez PostgreSQL :

```bash
# Ubuntu/Debian
sudo apt-get install postgresql-client

# macOS (avec Homebrew)
brew install postgresql

# Windows
# Téléchargez depuis postgresql.org
```

---

#### Se Connecter pour la Première Fois

```bash
# Connexion locale (par défaut)
psql -U postgres

# Connexion à une base spécifique
psql -U postgres -d ma_base

# Connexion à un serveur distant
psql -h localhost -p 5432 -U postgres -d ma_base
```

**Paramètres de connexion** :
- `-U` : Utilisateur (username)
- `-d` : Base de données (database)
- `-h` : Hôte (host)
- `-p` : Port (par défaut 5432)
- `-W` : Demander le mot de passe (force password prompt)

---

#### Créer une Base de Test

Si vous n'avez pas de base de données, créez-en une pour pratiquer :

```bash
# Depuis le shell
createdb test_psql

# Ou depuis psql
psql -U postgres
CREATE DATABASE test_psql;
\c test_psql

# Créer quelques tables de test
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    ville VARCHAR(50)
);

CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    client_id INTEGER REFERENCES clients(id),
    montant_total NUMERIC(10, 2),
    date_commande TIMESTAMP DEFAULT NOW()
);

-- Insérer des données de test
INSERT INTO clients (nom, email, ville) VALUES
    ('Dupont', 'alice.dupont@mail.fr', 'Paris'),
    ('Martin', 'bob.martin@mail.fr', 'Lyon'),
    ('Bernard', 'claire.bernard@mail.fr', 'Marseille');

INSERT INTO commandes (client_id, montant_total) VALUES
    (1, 150.50),
    (1, 200.00),
    (2, 75.25);
```

---

## Conventions du Guide

### Notation des Commandes

```bash
# Commandes psql (dans psql)
\dt

# Commandes SQL (dans psql, se terminent par ;)
SELECT * FROM clients;

# Commandes shell (dans le terminal, avant de lancer psql)
$ psql -U postgres

# Commentaires
-- Ceci est un commentaire SQL
# Ceci est un commentaire shell/psql
```

---

### Symboles Utilisés

- 🟢 **Niveau débutant** : Concepts de base, essentiels
- 🟡 **Niveau intermédiaire** : Utilisation quotidienne, productivité
- 🔴 **Niveau avancé** : Techniques expertes, automatisation

- ✅ **À faire** : Bonne pratique recommandée
- ❌ **À éviter** : Erreur courante ou mauvaise pratique
- ⚠️ **Attention** : Point important, piège potentiel
- 💡 **Astuce** : Conseil pour gagner du temps
- 🔒 **Sécurité** : Considération de sécurité importante

---

### Format des Exemples

Tous les exemples suivent cette structure :

```sql
-- Description de ce que fait l'exemple
\commande argument

# Résultat attendu (précédé de #)
# Les sorties sont commentées pour clarté
```

**Exemple concret** :

```sql
-- Lister toutes les tables du schéma public
\dt

# Résultat :
#            List of relations
#  Schema |   Name    | Type  |  Owner
# --------+-----------+-------+----------
#  public | clients   | table | postgres
#  public | commandes | table | postgres
```

---

## Structure des Sections

Chaque section suit une structure cohérente pour faciliter l'apprentissage :

1. **Introduction** : Contexte et objectifs
2. **Concepts fondamentaux** : Théorie nécessaire
3. **Commandes de base** : Syntaxe et exemples simples
4. **Cas d'usage pratiques** : Exemples réels et concrets
5. **Techniques avancées** : Astuces de pro
6. **Bonnes pratiques** : Recommandations et pièges à éviter
7. **Dépannage** : Erreurs courantes et solutions
8. **Résumé** : Tableau récapitulatif et points clés

---

## Ressources Complémentaires

### Documentation Officielle

- **psql Documentation** : https://www.postgresql.org/docs/current/app-psql.html
  La référence ultime, très complète (en anglais)

- **Tutorial PostgreSQL** : https://www.postgresql.org/docs/current/tutorial.html
  Introduction officielle au SQL PostgreSQL

---

### Aide Intégrée dans psql

psql intègre une aide complète accessible directement :

```bash
# Aide sur les méta-commandes
\?

# Aide sur une commande SQL
\h SELECT
\h CREATE TABLE

# Liste de toutes les commandes SQL disponibles
\h
```

**Astuce** 💡 : N'hésitez pas à utiliser `\?` régulièrement, c'est une référence rapide toujours disponible !

---

### Communautés et Support

- **Reddit r/PostgreSQL** : https://reddit.com/r/PostgreSQL
  Communauté active et bienveillante

- **PostgreSQL Discord** : Serveur Discord avec experts disponibles

- **Stack Overflow** : Tag [postgresql] pour vos questions techniques

- **Mailing Lists** : pgsql-general@postgresql.org (pour questions avancées)

---

## Conseils Avant de Commencer

### ✅ À Faire

1. **Créez une base de test** pour expérimenter sans risque
2. **Pratiquez chaque commande** en la tapant vous-même
3. **Créez votre .psqlrc** dès que possible
4. **Gardez ce guide ouvert** à côté de votre terminal
5. **Testez les exemples** - ne vous contentez pas de lire
6. **Expérimentez** - les variantes et combinaisons

---

### ❌ À Éviter

1. **Ne pas tester sur la production** sans backup !
2. **Ne pas sauter les bases** - commencez par la navigation
3. **Ne pas mémoriser tout** - comprenez les concepts, le reste viendra
4. **Ne pas avoir peur** de faire des erreurs - c'est comme ça qu'on apprend
5. **Ne pas négliger .psqlrc** - il change vraiment la vie

---

### 💡 Astuces Générales

**Raccourcis Clavier Essentiels** :

| Raccourci | Action |
|-----------|--------|
| `Ctrl + C` | Annuler la commande en cours |
| `Ctrl + D` | Quitter psql (comme `\q`) |
| `Ctrl + L` | Effacer l'écran |
| `Tab` | Auto-complétion |
| `↑` / `↓` | Naviguer dans l'historique |
| `Ctrl + R` | Recherche dans l'historique |

**Auto-complétion** :

```bash
# Taper \d puis Tab : liste toutes les commandes \d*
\d<Tab>

# Taper \dt cli puis Tab : complète avec les tables commençant par "cli"
\dt cli<Tab>

# Double Tab pour voir toutes les options
\dt <Tab><Tab>
```

---

## Votre Premier Workflow avec psql

Voici un workflow typique pour vous familiariser avec psql :

```bash
# 1. Se connecter
psql -U postgres -d test_psql

# 2. Voir où vous êtes
\conninfo

# 3. Activer le timing (toujours utile)
\timing on

# 4. Lister les tables disponibles
\dt

# 5. Examiner la structure d'une table
\d clients

# 6. Faire une requête simple
SELECT * FROM clients;

# 7. En mode étendu pour mieux voir
\x
SELECT * FROM clients WHERE id = 1;

# 8. Revenir au mode normal
\x

# 9. Sauvegarder la session dans un fichier
\o /tmp/ma_session.txt
\dt
SELECT COUNT(*) FROM clients;
\o

# 10. Quitter
\q
```

**Bravo !** 🎉 Vous venez de faire votre première session psql complète.

---

## Roadmap d'Apprentissage

### 🎯 Objectif : Maîtrise de psql en 4 Semaines

#### Semaine 1 : Les Fondamentaux
- ✅ Installer et configurer psql
- ✅ Maîtriser les commandes de navigation (`\l`, `\c`, `\dt`, `\d`)
- ✅ Comprendre la différence SQL vs méta-commandes
- ✅ Pratiquer sur une base de test
- 📊 **Objectif mesurable** : Explorer une base inconnue en < 5 minutes

#### Semaine 2 : La Productivité
- ✅ Créer votre `.psqlrc` personnalisé
- ✅ Activer `\timing`, configurer `\x auto`, `\pset null`
- ✅ Personnaliser votre prompt
- ✅ Maîtriser les raccourcis clavier
- 📊 **Objectif mesurable** : Environnement confortable et rapide

#### Semaine 3 : Les Données
- ✅ Exporter une table vers CSV avec `\copy`
- ✅ Importer des données depuis un fichier
- ✅ Créer votre premier script SQL avec `\i`
- ✅ Rediriger des résultats avec `\o`
- 📊 **Objectif mesurable** : Migration de données entre environnements

#### Semaine 4 : L'Excellence
- ✅ Utiliser `\watch` pour du monitoring
- ✅ Générer du SQL avec `\gexec`
- ✅ Créer des scripts avec variables et conditions
- ✅ Automatiser une tâche récurrente
- 📊 **Objectif mesurable** : Script de backup automatisé fonctionnel

---

## Certification de Compétence

Après avoir parcouru ce guide, vous devriez être capable de :

**Niveau Débutant** :
- [ ] Vous connecter à une base PostgreSQL
- [ ] Lister et explorer les tables (`\dt`, `\d`)
- [ ] Exécuter des requêtes SQL de base
- [ ] Naviguer entre les bases (`\l`, `\c`)
- [ ] Comprendre la sortie des commandes

**Niveau Intermédiaire** :
- [ ] Configurer un environnement psql personnalisé (`.psqlrc`)
- [ ] Exporter et importer des données (`\copy`)
- [ ] Mesurer les performances (`\timing`, `EXPLAIN`)
- [ ] Créer et exécuter des scripts SQL (`\i`)
- [ ] Utiliser des variables et des conditions

**Niveau Avancé** :
- [ ] Surveiller la base en temps réel (`\watch`)
- [ ] Générer et exécuter du SQL dynamiquement (`\gexec`)
- [ ] Éditer du code dans un éditeur externe (`\ef`, `\ev`)
- [ ] Automatiser des tâches complexes avec scripts
- [ ] Déboguer efficacement avec les outils psql

---

## Pour Aller Plus Loin

Après avoir maîtrisé psql, vous serez prêt pour :

- **PostgreSQL Avancé** : Optimisation, indexation, réplication
- **Administration** : Maintenance, monitoring, haute disponibilité
- **Scripts Shell** : Intégration de psql dans vos workflows DevOps
- **CI/CD** : Automatisation de migrations et tests de base de données

---

## Mot de Conclusion

psql est un outil extraordinairement puissant qui peut sembler intimidant au début. Mais avec de la pratique, il deviendra votre meilleur allié pour travailler avec PostgreSQL.

**Quelques principes à garder en mémoire** :

✨ **Pratique** : Testez chaque commande. La théorie sans pratique s'oublie vite.

✨ **Progression** : Commencez simple. Vous n'avez pas besoin de tout maîtriser d'un coup.

✨ **Personnalisation** : Créez votre environnement idéal avec `.psqlrc`. C'est votre outil, adaptez-le à vous.

✨ **Référence** : Ce guide est fait pour être consulté régulièrement. Gardez-le sous la main.

✨ **Communauté** : Partagez vos découvertes, aidez les autres, apprenez ensemble.

---

**Prêt à commencer ?** Passez à la **Section 1 : Navigation** et lancez-vous ! 🚀

---

## Table des Matières Générale

### 📘 Section 1 : Navigation
1. Se Connecter à une Base de Données
2. Explorer les Bases de Données
3. Explorer les Schémas
4. Explorer les Tables
5. Explorer les Index
6. Explorer les Vues
7. Explorer les Séquences
8. Explorer les Fonctions et Procédures
9. Navigation Entre Schémas
10. Astuces de Navigation

### 📘 Section 2 : Configuration
1. Affichage Étendu (\x)
2. Mesurer le Temps (\timing)
3. Formatage avec \pset
4. Autres Commandes de Configuration
5. Configuration du Prompt
6. Le Fichier .psqlrc
7. Variables d'Environnement Système
8. Commandes Avancées de Configuration

### 📘 Section 3 : Export/Import
1. Redirection de Sortie (\o)
2. Import/Export de Données (\copy)
3. Exécution de Scripts (\i)
4. Combinaison \o et \copy
5. Cas d'Usage Réels
6. Bonnes Pratiques

### 📘 Section 4 : Méta-commandes Avancées
1. Exécution Répétée (\watch)
2. Variantes d'Exécution (\gx, \gset, \gexec)
3. Édition et Visualisation de Code
4. Édition du Buffer de Requête
5. Gestion de l'Historique
6. Variables psql Avancées
7. Large Objects
8. Techniques Avancées de Scripting

---

**Bon apprentissage avec psql ! 🐘**

---


⏭️ [Navigation (\l, \c, \dt, \d+, \df, etc.)](/annexes/commandes-psql/01-navigation.md)
