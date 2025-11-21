🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Commandes psql Essentielles - Configuration

## Introduction

Après avoir appris à naviguer dans PostgreSQL avec psql, il est temps de personnaliser votre environnement pour le rendre plus confortable et efficace. Ce guide vous présente toutes les commandes de configuration qui transformeront votre expérience avec psql.

### Pourquoi configurer psql ?

Une bonne configuration vous permet de :

- ✅ **Améliorer la lisibilité** des résultats de requêtes
- ✅ **Mesurer les performances** de vos requêtes
- ✅ **Adapter l'affichage** selon le type de données
- ✅ **Automatiser** vos préférences au démarrage
- ✅ **Gagner du temps** avec des raccourcis

### Types de configuration

- **Temporaire** : Active pour la session en cours uniquement
- **Permanente** : Sauvegardée dans `.psqlrc`, active à chaque démarrage

---

## Affichage Étendu : \x

### Qu'est-ce que l'affichage étendu ?

Par défaut, psql affiche les résultats en **format tabulaire** (lignes et colonnes côte à côte). L'affichage étendu affiche chaque colonne sur **une ligne séparée**, ce qui est idéal pour :
- Les tables avec beaucoup de colonnes
- Les résultats avec des valeurs longues
- Une meilleure lisibilité sur des petits écrans

### \x - Basculer l'affichage étendu

**Syntaxe** :
```
\x [on|off|auto]
```

**Exemples** :

#### Sans \x (affichage normal)

```sql
SELECT id, nom, prenom, email, telephone, adresse, ville FROM clients WHERE id = 1;

# Résultat (difficile à lire si les valeurs sont longues) :
 id |   nom   | prenom |        email         |   telephone   |        adresse         |  ville
----+---------+--------+----------------------+---------------+------------------------+---------
  1 | Dupont  | Alice  | alice.dupont@mail.fr | 06 12 34 56 78| 15 rue de la Paix     | Paris
```

#### Avec \x (affichage étendu)

```bash
\x
SELECT id, nom, prenom, email, telephone, adresse, ville FROM clients WHERE id = 1;

# Résultat (beaucoup plus lisible) :
-[ RECORD 1 ]------------------
id        | 1
nom       | Dupont
prenom    | Alice
email     | alice.dupont@mail.fr
telephone | 06 12 34 56 78
adresse   | 15 rue de la Paix
ville     | Paris
```

### Options de \x

```bash
# Activer l'affichage étendu
\x on
# Expanded display is on.

# Désactiver l'affichage étendu
\x off
# Expanded display is off.

# Basculer (toggle) : si c'est on, passe à off et vice-versa
\x
# Expanded display is on.
\x
# Expanded display is off.

# Mode automatique : utilise l'étendu si ça ne rentre pas à l'écran
\x auto
# Expanded display is used automatically.
```

### Quand utiliser \x ?

**✅ Utilisez \x pour** :
- Examiner un enregistrement spécifique avec beaucoup de colonnes
- Lire des valeurs JSONB ou des textes longs
- Voir clairement les détails d'une ligne
- Travailler sur un petit terminal

**❌ N'utilisez PAS \x pour** :
- Comparer plusieurs lignes côte à côte
- Avoir une vue d'ensemble de nombreuses lignes
- Exporter des données (utilisez plutôt \copy)

### Exemple pratique

```sql
-- Sans \x : difficile à lire
SELECT * FROM commandes WHERE id = 100;

-- Activer \x
\x on

-- Relancer la même requête : beaucoup mieux !
SELECT * FROM commandes WHERE id = 100;

-[ RECORD 1 ]--------+----------------------
id                   | 100
client_id            | 42
date_commande        | 2024-11-15 14:23:45
montant_total        | 1599.99
statut               | livree
adresse_livraison    | 12 avenue des Champs Elysées
commentaires         | Livraison rapide demandée
created_at           | 2024-11-15 14:23:45.123456
updated_at           | 2024-11-18 10:15:32.789012
```

**Astuce** 💡 : Je recommande `\x auto` dans votre `.psqlrc` pour avoir le meilleur des deux mondes !

---

## Mesurer le Temps : \timing

### \timing - Activer la mesure du temps d'exécution

**Syntaxe** :
```
\timing [on|off]
```

**Description** : Affiche le temps d'exécution de chaque commande SQL.

**Exemples** :

```bash
# Activer le timing
\timing on
# Timing is on.

# Exécuter une requête
SELECT COUNT(*) FROM clients;

# Résultat :
 count
-------
  5000
(1 row)

Time: 45.234 ms

# Requête plus lente
SELECT * FROM commandes
WHERE date_commande >= NOW() - INTERVAL '1 year'
ORDER BY montant_total DESC;

# Résultat : ... données ...
Time: 1234.567 ms (00:01.235)
```

### Désactiver le timing

```bash
\timing off
# Timing is off.

# Ou simplement basculer
\timing
```

### Interpréter les résultats

Le temps affiché inclut :
- ⏱️ Le temps d'exécution côté serveur
- 🌐 Le temps de transfert réseau
- 📊 Le temps de formatage côté client

**Unités** :
- `ms` : millisecondes (< 1 seconde)
- `s` : secondes (format `00:01.234` pour lisibilité)

### Quand utiliser \timing ?

**✅ Toujours !**

Sérieusement, activez-le par défaut dans votre `.psqlrc`. Cela vous aide à :

1. **Identifier les requêtes lentes**
   ```sql
   SELECT * FROM huge_table;  -- Time: 15234.567 ms
   -- Oups, besoin d'un index !
   ```

2. **Comparer différentes approches**
   ```sql
   -- Approche 1
   SELECT ... FROM ... WHERE ...;
   Time: 523.123 ms

   -- Approche 2 (avec un index)
   SELECT ... FROM ... WHERE ...;
   Time: 12.345 ms
   -- Beaucoup mieux !
   ```

3. **Valider l'impact d'un index**
   ```sql
   -- Avant création d'index
   SELECT * FROM produits WHERE categorie = 'electronique';
   Time: 1234.56 ms

   CREATE INDEX idx_produits_categorie ON produits(categorie);

   -- Après création d'index
   SELECT * FROM produits WHERE categorie = 'electronique';
   Time: 23.45 ms
   -- 50x plus rapide !
   ```

4. **Benchmarker vos optimisations**

**Attention** ⚠️ : Le timing inclut le réseau. Sur un serveur distant, les résultats seront plus lents qu'en local.

---

## Formatage avec \pset

### Qu'est-ce que \pset ?

`\pset` (pour "print set") contrôle **tous les aspects de l'affichage** des résultats : format, bordures, séparateurs, alignement, etc.

**Syntaxe générale** :
```
\pset [OPTION [VALUE]]
```

**Sans arguments** : Affiche toutes les options actuelles
```bash
\pset

# Résultat :
border                   1
columns                  0
expanded                 off
fieldsep                 '|'
fieldsep_zero            off
footer                   on
format                   aligned
linestyle                ascii
null                     ''
...
```

---

### \pset border - Bordures des tableaux

**Syntaxe** :
```
\pset border [0|1|2|3]
```

**Valeurs** :
- `0` : Aucune bordure
- `1` : Bordures internes (défaut)
- `2` : Bordures complètes
- `3` : Bordures doubles (style Unicode)

**Exemples** :

#### border 0 (aucune bordure)
```bash
\pset border 0
SELECT id, nom, prix FROM produits LIMIT 3;

# Résultat :
id nom          prix
1  Ordinateur   999.99
2  Souris       29.99
3  Clavier      79.99
```

#### border 1 (défaut - bordures internes)
```bash
\pset border 1
SELECT id, nom, prix FROM produits LIMIT 3;

# Résultat :
 id |    nom     |  prix
----+------------+--------
  1 | Ordinateur | 999.99
  2 | Souris     |  29.99
  3 | Clavier    |  79.99
```

#### border 2 (bordures complètes)
```bash
\pset border 2
SELECT id, nom, prix FROM produits LIMIT 3;

# Résultat :
+----+------------+--------+
| id |    nom     |  prix  |
+----+------------+--------+
|  1 | Ordinateur | 999.99 |
|  2 | Souris     |  29.99 |
|  3 | Clavier    |  79.99 |
+----+------------+--------+
```

#### border 3 (double - Unicode)
```bash
\pset border 3
\pset linestyle unicode
SELECT id, nom, prix FROM produits LIMIT 3;

# Résultat :
┌────┬────────────┬────────┐
│ id │    nom     │  prix  │
├────┼────────────┼────────┤
│  1 │ Ordinateur │ 999.99 │
│  2 │ Souris     │  29.99 │
│  3 │ Clavier    │  79.99 │
└────┴────────────┴────────┘
```

**Recommandation** : `border 1` pour l'usage quotidien, `border 2` pour les présentations.

---

### \pset format - Format d'affichage

**Syntaxe** :
```
\pset format [aligned|unaligned|wrapped|html|asciidoc|latex|latex-longtable|troff-ms]
```

**Formats principaux** :

#### aligned (défaut)
```bash
\pset format aligned
SELECT id, nom FROM produits LIMIT 2;

 id |    nom
----+------------
  1 | Ordinateur
  2 | Souris
```

#### unaligned (pour parsing)
```bash
\pset format unaligned
\pset fieldsep '|'
SELECT id, nom FROM produits LIMIT 2;

id|nom
1|Ordinateur
2|Souris
```

**Usage** : Excellent pour des scripts shell qui parsent la sortie.

#### html
```bash
\pset format html
SELECT id, nom FROM produits LIMIT 2;

<table border="1">
  <tr>
    <th align="center">id</th>
    <th align="left">nom</th>
  </tr>
  <tr valign="top">
    <td align="right">1</td>
    <td align="left">Ordinateur</td>
  </tr>
  <tr valign="top">
    <td align="right">2</td>
    <td align="left">Souris</td>
  </tr>
</table>
```

**Usage** : Génération de rapports HTML.

#### wrapped
```bash
\pset format wrapped
# Comme aligned, mais enveloppe les longues lignes au lieu de les tronquer
```

---

### \pset null - Affichage des valeurs NULL

**Syntaxe** :
```
\pset null 'STRING'
```

**Problème** : Par défaut, NULL s'affiche comme une chaîne vide '', ce qui le rend invisible et indiscernable d'une vraie chaîne vide.

**Exemples** :

#### Sans configuration
```sql
SELECT id, nom, email FROM clients WHERE id IN (1, 2);

 id |  nom   | email
----+--------+-------
  1 | Dupont | alice@example.com
  2 | Martin |           -- Impossible de savoir si c'est NULL ou ''
```

#### Avec \pset null
```bash
\pset null '(null)'
SELECT id, nom, email FROM clients WHERE id IN (1, 2);

 id |  nom   |       email
----+--------+-------------------
  1 | Dupont | alice@example.com
  2 | Martin | (null)            -- Maintenant c'est clair !
```

**Autres valeurs populaires** :
```bash
\pset null 'NULL'
\pset null '¤'
\pset null '␀'
\pset null '[NULL]'
```

**Astuce** 💡 : Mettez `\pset null '(null)'` dans votre `.psqlrc` pour toujours voir les NULL clairement !

---

### \pset fieldsep - Séparateur de champs

**Syntaxe** :
```
\pset fieldsep 'STRING'
```

**Usage** : Utilisé uniquement en mode `unaligned`, définit le caractère séparant les colonnes.

**Exemples** :

```bash
\pset format unaligned

# Séparateur pipe (défaut)
\pset fieldsep '|'
SELECT id, nom, prix FROM produits LIMIT 2;
id|nom|prix
1|Ordinateur|999.99
2|Souris|29.99

# Séparateur virgule (CSV)
\pset fieldsep ','
SELECT id, nom, prix FROM produits LIMIT 2;
id,nom,prix
1,Ordinateur,999.99
2,Souris,29.99

# Séparateur tabulation (TSV)
\pset fieldsep '\t'
# ou
\pset fieldsep E'\t'
```

**Usage pratique** : Export de données pour Excel, scripts, etc.

---

### \pset linestyle - Style des lignes

**Syntaxe** :
```
\pset linestyle [ascii|old-ascii|unicode]
```

**Exemples** :

#### ascii (défaut)
```bash
\pset linestyle ascii
\pset border 2
SELECT * FROM produits LIMIT 2;

+----+------+
| id | nom  |
+----+------+
|  1 | PC   |
|  2 | Souris |
+----+------+
```

#### unicode (moderne, joli)
```bash
\pset linestyle unicode
\pset border 2
SELECT * FROM produits LIMIT 2;

┌────┬────────┐
│ id │  nom   │
├────┼────────┤
│  1 │ PC     │
│  2 │ Souris │
└────┴────────┘
```

**Attention** ⚠️ : `unicode` nécessite que votre terminal supporte UTF-8 (cas général aujourd'hui).

---

### \pset footer - Pied de page

**Syntaxe** :
```
\pset footer [on|off]
```

**Description** : Affiche ou masque la ligne de comptage en bas des résultats.

**Exemples** :

#### Avec footer (défaut)
```sql
SELECT * FROM produits LIMIT 5;

-- Résultat : ...
(5 rows)  ← C'est le footer
```

#### Sans footer
```bash
\pset footer off
SELECT * FROM produits LIMIT 5;

-- Résultat : ...
-- (pas de ligne de comptage)
```

**Usage** : Désactiver pour des exports propres ou des scripts.

---

### \pset columns - Largeur des colonnes

**Syntaxe** :
```
\pset columns [N]
```

**Description** : Définit la largeur maximale de l'affichage (en caractères).

**Exemples** :

```bash
# Largeur automatique (défaut)
\pset columns 0

# Largeur fixe à 80 caractères
\pset columns 80

# Largeur pour écran large
\pset columns 200
```

**Usage** : Utile si psql ne détecte pas correctement la largeur de votre terminal.

---

### \pset title - Titre du tableau

**Syntaxe** :
```
\pset title 'TEXTE'
```

**Description** : Ajoute un titre au-dessus du tableau de résultats.

**Exemples** :

```bash
\pset title 'Liste des Produits en Promotion'
SELECT nom, prix FROM produits WHERE promotion = true;

# Résultat :
       Liste des Produits en Promotion
    nom     |  prix
------------+--------
 Ordinateur | 799.99
 Clavier    |  59.99
```

**Astuce** 💡 : Utile pour les rapports et les exports.

---

### \pset tableattr - Attributs HTML

**Syntaxe** :
```
\pset tableattr 'HTML_ATTRIBUTES'
```

**Description** : Définit des attributs HTML supplémentaires pour les tableaux (utilisé avec `format html`).

**Exemples** :

```bash
\pset format html
\pset tableattr 'class="table table-striped" id="results"'

SELECT * FROM produits LIMIT 2;

# Résultat :
<table border="1" class="table table-striped" id="results">
  ...
</table>
```

**Usage** : Intégration avec Bootstrap, Tailwind, ou votre CSS personnalisé.

---

### \pset recordsep - Séparateur d'enregistrements

**Syntaxe** :
```
\pset recordsep 'STRING'
```

**Description** : En mode `unaligned`, définit le séparateur entre les lignes.

**Exemples** :

```bash
\pset format unaligned
\pset fieldsep ','
\pset recordsep '\n'  # Nouvelle ligne (défaut)

SELECT id, nom FROM produits LIMIT 2;

id,nom
1,Ordinateur
2,Souris
```

---

### \pset tuples_only - Masquer l'en-tête et le footer

**Syntaxe** :
```
\pset tuples_only [on|off]
\t  # Raccourci
```

**Description** : N'affiche que les données, sans en-tête ni footer.

**Exemples** :

#### Affichage normal
```sql
SELECT nom FROM produits LIMIT 3;

    nom
------------
 Ordinateur
 Souris
 Clavier
(3 rows)
```

#### Avec tuples_only
```bash
\pset tuples_only on
# Ou simplement :
\t

SELECT nom FROM produits LIMIT 3;

 Ordinateur
 Souris
 Clavier
```

**Usage** : Scripts shell qui récupèrent uniquement les valeurs.

**Exemple pratique** :
```bash
# Script bash utilisant psql
for db in $(psql -t -c "SELECT datname FROM pg_database WHERE datname NOT LIKE 'template%'"); do
    echo "Sauvegarde de $db..."
    pg_dump "$db" > "/backup/${db}.sql"
done
```

---

## Combinaisons Utiles

### Configuration pour l'export CSV

```bash
# Configuration
\pset format unaligned
\pset fieldsep ','
\pset footer off
\pset tuples_only on

# Export
SELECT * FROM clients;

# Ou directement dans un fichier
\o /tmp/clients.csv
SELECT * FROM clients;
\o
```

### Configuration pour un affichage optimal

```bash
# Ma configuration recommandée
\x auto                          # Étendu si nécessaire
\pset null '(null)'             # Voir les NULL
\pset border 1                  # Bordures internes
\pset linestyle unicode         # Joli !
\timing on                      # Toujours utile
```

### Configuration pour des rapports

```bash
\pset border 2
\pset linestyle unicode
\pset title 'Rapport Mensuel des Ventes'
\pset footer on

SELECT * FROM rapport_ventes WHERE mois = 'Novembre';
```

---

## Autres Commandes de Configuration

### \echo - Afficher un message

**Syntaxe** :
```
\echo TEXT
```

**Exemples** :

```bash
\echo '=== Début du rapport ==='
SELECT COUNT(*) FROM commandes;
\echo '=== Fin du rapport ==='

# Résultat :
=== Début du rapport ===
 count
-------
  1234
(1 row)
=== Fin du rapport ===
```

**Usage** : Scripts, rapports, debugging.

---

### \qecho - Afficher dans le fichier de sortie

**Syntaxe** :
```
\qecho TEXT
```

**Description** : Comme `\echo`, mais écrit dans le fichier de sortie (`\o`) et non sur la console.

**Exemples** :

```bash
\o /tmp/rapport.txt
\qecho '=== Rapport généré le 2024-11-21 ==='
SELECT * FROM ventes;
\qecho '=== Fin du rapport ==='
\o
```

---

### \prompt - Demander une saisie utilisateur

**Syntaxe** :
```
\prompt [TEXT] VARIABLE
```

**Exemples** :

```bash
# Demander un ID
\prompt 'Entrez un ID client : ' client_id

# Utiliser la variable
SELECT * FROM clients WHERE id = :client_id;
```

**Autre exemple** :
```bash
\prompt 'Ville à rechercher : ' ville_recherche
SELECT nom, prenom FROM clients WHERE ville = :'ville_recherche';
```

**Note** : Utilisez `:variable` pour une valeur numérique et `:'variable'` pour une chaîne.

---

### \set - Définir une variable

**Syntaxe** :
```
\set [VARIABLE [VALUE]]
```

**Exemples** :

```bash
# Définir une variable
\set limit_value 10

# Utiliser la variable
SELECT * FROM produits LIMIT :limit_value;

# Variable de chaîne
\set search_term 'Ordinateur'
SELECT * FROM produits WHERE nom LIKE '%' || :'search_term' || '%';

# Voir toutes les variables
\set

# Résultat :
AUTOCOMMIT = 'on'
COMP_KEYWORD_CASE = 'preserve-upper'
...
limit_value = '10'
search_term = 'Ordinateur'
```

**Variables spéciales** :
- `AUTOCOMMIT` : Mode auto-commit (on/off)
- `ECHO` : Afficher les commandes exécutées
- `ON_ERROR_STOP` : Arrêter sur erreur
- `QUIET` : Mode silencieux

---

### \unset - Supprimer une variable

**Syntaxe** :
```
\unset VARIABLE
```

**Exemples** :

```bash
\set ma_variable 'valeur'
\unset ma_variable
```

---

### \! - Exécuter une commande shell

**Syntaxe** :
```
\! COMMAND
```

**Exemples** :

```bash
# Voir les fichiers du répertoire courant
\! ls -la

# Voir l'utilisation disque
\! df -h

# Éditer un fichier avec votre éditeur
\! nano mon_script.sql

# Afficher un fichier
\! cat /etc/hostname

# Windows
\! dir
\! type fichier.txt
```

**Usage** : Pratique pour rester dans psql sans quitter.

---

## Configuration du Prompt

### \set PROMPT1, PROMPT2, PROMPT3

psql a trois niveaux de prompts configurables :

- **PROMPT1** : Prompt principal (quand psql attend une commande)
- **PROMPT2** : Prompt de continuation (quand une commande est incomplète)
- **PROMPT3** : Prompt lors d'un COPY FROM STDIN

**Variables de prompt disponibles** :

| Variable | Description | Exemple |
|----------|-------------|---------|
| `%M` | Nom complet de l'hôte | `localhost` |
| `%m` | Nom court de l'hôte | `localhost` |
| `%>` | Port | `5432` |
| `%n` | Nom d'utilisateur | `postgres` |
| `%/` | Base de données actuelle | `ma_boutique` |
| `%~` | Base de données (avec ~ si HOME) | `ma_boutique` |
| `%#` | `#` si superuser, sinon `>` | `>` |
| `%R` | État de transaction | `=` (aucune), `^` (transaction), `!` (erreur) |
| `%x` | État de transaction (verbose) | ` ` ou `*` |
| `%?` | Code d'erreur de la dernière commande | `0` |
| `%[...%]` | Séquences d'échappement terminal | Couleurs |

### Exemples de prompts personnalisés

#### Prompt par défaut
```bash
ma_boutique=#
```

#### Prompt avec informateur complet
```bash
\set PROMPT1 '%n@%M:%> %/ %R%# '
# Résultat :
postgres@localhost:5432 ma_boutique =#
```

#### Prompt avec couleurs
```bash
\set PROMPT1 '%[%033[1;32m%]%n@%/%[%033[0m%]%R%# '
# Résultat : postgres@ma_boutique en vert
```

#### Prompt avec état de transaction
```bash
\set PROMPT1 '%n@%/ %x%R%# '

# Sans transaction
postgres@ma_boutique =#

# Dans une transaction
BEGIN;
postgres@ma_boutique *=#

# Après erreur
SELECT * FROM table_inexistante;
postgres@ma_boutique !=#
```

#### Mon prompt recommandé
```bash
\set PROMPT1 '%[%033[1;34m%]%n@%/%[%033[0m%]%R%x%# '
\set PROMPT2 '%[%033[1;34m%]%R%[%033[0m%]%# '
```

---

## Le Fichier .psqlrc

### Qu'est-ce que .psqlrc ?

`.psqlrc` est un fichier de configuration qui s'exécute automatiquement à chaque démarrage de psql. C'est l'équivalent du `.bashrc` pour psql.

**Emplacement** :
- **Linux/Mac** : `~/.psqlrc` (home directory)
- **Windows** : `%APPDATA%\postgresql\psqlrc.conf`

### Créer votre .psqlrc

```bash
# Linux/Mac
nano ~/.psqlrc

# Windows
notepad %APPDATA%\postgresql\psqlrc.conf
```

### Exemple de .psqlrc complet

```sql
-- ~/.psqlrc
-- Configuration personnalisée de psql

-- ============================================
-- Affichage et Formatage
-- ============================================

-- Affichage étendu automatique
\x auto

-- Voir les NULL clairement
\pset null '(null)'

-- Bordures et style
\pset border 1
\pset linestyle unicode

-- Toujours afficher le temps d'exécution
\timing on

-- ============================================
-- Historique
-- ============================================

-- Sauvegarder l'historique de manière persistante
\set HISTFILE ~/.psql_history

-- Taille de l'historique
\set HISTSIZE 10000

-- ============================================
-- Erreurs et Comportement
-- ============================================

-- Arrêter l'exécution en cas d'erreur dans un script
-- (désactivé par défaut pour l'usage interactif)
-- \set ON_ERROR_STOP on

-- Afficher les erreurs en détail
\set VERBOSITY verbose

-- ============================================
-- Prompt Personnalisé
-- ============================================

-- Prompt coloré avec base de données et état de transaction
\set PROMPT1 '%[%033[1;34m%]%n@%/%[%033[0m%]%x%R%# '
\set PROMPT2 '%[%033[1;34m%]%R%[%033[0m%]%# '

-- ============================================
-- Auto-complétion
-- ============================================

-- Complétion en majuscules pour les mots-clés SQL
\set COMP_KEYWORD_CASE upper

-- ============================================
-- Messages
-- ============================================

-- Message de bienvenue personnalisé
\echo ''
\echo '╔════════════════════════════════════════╗'
\echo '║   Bienvenue dans PostgreSQL !          ║'
\echo '╚════════════════════════════════════════╝'
\echo ''
\echo 'Configuration personnalisée chargée depuis .psqlrc'
\echo 'Astuce: Utilisez \\? pour l''aide'
\echo ''

-- ============================================
-- Raccourcis Personnalisés (Optionnel)
-- ============================================

-- Raccourci pour voir les connexions actives
\set show_connections 'SELECT pid, usename, application_name, client_addr, state, query_start, state_change FROM pg_stat_activity WHERE state <> \'idle\' ORDER BY query_start;'

-- Raccourci pour voir les tables et leur taille
\set show_table_sizes 'SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||''.''||tablename)) AS size FROM pg_tables ORDER BY pg_total_relation_size(schemaname||''.''||tablename) DESC;'

-- Usage des raccourcis :
-- :show_connections
-- :show_table_sizes
```

### Utiliser les raccourcis définis

```bash
# Après avoir défini les raccourcis dans .psqlrc

# Voir les connexions actives
:show_connections

# Voir la taille des tables
:show_table_sizes
```

### Tester votre .psqlrc

```bash
# Relancer psql
psql -U postgres -d ma_base

# Ou depuis psql, recharger la configuration
\i ~/.psqlrc
```

---

## Variables d'Environnement Système

### PSQL_HISTORY

```bash
# Définir un emplacement personnalisé pour l'historique
export PSQL_HISTORY=~/.config/psql_history
```

### PAGER

```bash
# Utiliser less comme paginateur
export PAGER=less

# Ou more
export PAGER=more

# Désactiver le paginateur
\pset pager off
```

### PSQL_EDITOR ou EDITOR

```bash
# Définir l'éditeur pour \e
export PSQL_EDITOR=nano
export PSQL_EDITOR=vim
export PSQL_EDITOR=code  # VS Code
```

---

## Commandes Avancées de Configuration

### \pset pager - Contrôle du paginateur

**Syntaxe** :
```
\pset pager [on|off|always]
```

**Description** : Contrôle si les résultats longs utilisent un paginateur (less, more).

**Exemples** :

```bash
# Utiliser le paginateur seulement si nécessaire (défaut)
\pset pager on

# Ne jamais utiliser de paginateur
\pset pager off

# Toujours utiliser le paginateur
\pset pager always
```

---

### \encoding - Encodage client

**Syntaxe** :
```
\encoding [ENCODING]
```

**Exemples** :

```bash
# Voir l'encodage actuel
\encoding
# UTF8

# Changer l'encodage (rarement nécessaire)
\encoding LATIN1
```

---

### \setenv - Définir une variable d'environnement

**Syntaxe** :
```
\setenv NAME VALUE
```

**Exemples** :

```bash
\setenv PAGER less
\setenv EDITOR nano
```

---

## Cas d'Usage Pratiques

### Configuration pour développement

```sql
-- .psqlrc pour développement
\x auto
\pset null '(null)'
\pset border 2
\pset linestyle unicode
\timing on
\set VERBOSITY verbose
\set ON_ERROR_STOP on
\set COMP_KEYWORD_CASE upper
\echo 'Environnement de développement chargé'
```

### Configuration pour production (lecture seule)

```sql
-- .psqlrc pour consultation production
\x auto
\pset null '(null)'
\timing on
\set AUTOCOMMIT off
\echo ''
\echo '⚠️  MODE PRODUCTION - Transactions manuelles activées'
\echo '    N''oubliez pas de COMMIT ou ROLLBACK !'
\echo ''
```

### Configuration pour exports

```sql
-- .psqlrc pour exports CSV
\pset format unaligned
\pset fieldsep ','
\pset footer off
\pset tuples_only on
\timing off
\echo 'Mode export CSV activé'
```

### Configuration pour scripts

```sql
-- Fichier script.sql
\set ON_ERROR_STOP on
\set QUIET on

-- Votre script
BEGIN;
-- ... commandes SQL ...
COMMIT;
```

---

## Commandes de Débogage

### \set ECHO - Afficher les commandes

**Syntaxe** :
```
\set ECHO [all|errors|queries]
```

**Valeurs** :
- `all` : Affiche tout (commandes et méta-commandes)
- `errors` : Affiche uniquement les commandes qui échouent
- `queries` : Affiche uniquement les requêtes SQL

**Exemples** :

```bash
\set ECHO queries

SELECT * FROM clients WHERE id = 1;

# Affiche :
# SELECT * FROM clients WHERE id = 1;
# (puis le résultat)
```

**Usage** : Déboguer des scripts, comprendre ce qui est exécuté.

---

### \set ON_ERROR_ROLLBACK - Rollback automatique

**Syntaxe** :
```
\set ON_ERROR_ROLLBACK [on|off|interactive]
```

**Description** : En cas d'erreur dans une transaction, effectue automatiquement un ROLLBACK.

**Valeurs** :
- `on` : Toujours rollback sur erreur
- `off` : Jamais (défaut)
- `interactive` : Uniquement en mode interactif

**Exemples** :

```bash
\set ON_ERROR_ROLLBACK on

BEGIN;
    INSERT INTO clients (nom) VALUES ('Test');
    INSERT INTO clients (nom) VALUES (NULL);  -- Erreur si NOT NULL
    -- Rollback automatique ici
COMMIT;  -- Rien à committer
```

---

### \set VERBOSITY - Niveau de détail des erreurs

**Syntaxe** :
```
\set VERBOSITY [default|verbose|terse]
```

**Exemples** :

```bash
# Défaut
\set VERBOSITY default
SELECT * FROM table_inexistante;
# ERROR:  relation "table_inexistante" does not exist
# LINE 1: SELECT * FROM table_inexistante;
#                       ^

# Verbose (plus de détails)
\set VERBOSITY verbose
SELECT * FROM table_inexistante;
# ERROR:  42P01: relation "table_inexistante" does not exist
# LINE 1: SELECT * FROM table_inexistante;
#                       ^
# LOCATION:  parserOpenTable, parse_relation.c:1180

# Terse (minimal)
\set VERBOSITY terse
SELECT * FROM table_inexistante;
# ERROR:  relation "table_inexistante" does not exist
```

---

## Résumé des Commandes Essentielles

### Tableau Récapitulatif

| Commande | Description | Exemple |
|----------|-------------|---------|
| `\x [auto]` | Affichage étendu | `\x auto` |
| `\timing` | Temps d'exécution | `\timing on` |
| `\pset border N` | Bordures (0-3) | `\pset border 2` |
| `\pset null 'X'` | Affichage NULL | `\pset null '(null)'` |
| `\pset format X` | Format de sortie | `\pset format html` |
| `\t` | Tuples seuls | `\t` |
| `\set VAR VAL` | Définir variable | `\set limit 10` |
| `\echo TEXT` | Afficher message | `\echo 'Début'` |
| `\!` | Commande shell | `\! ls -l` |

---

## Ma Configuration Recommandée

Voici la configuration que je recommande pour la plupart des utilisateurs :

```sql
-- ~/.psqlrc optimal

-- Affichage
\x auto
\pset null '(null)'
\pset border 1
\pset linestyle unicode

-- Performance
\timing on

-- Historique
\set HISTFILE ~/.psql_history- :DBNAME
\set HISTCONTROL ignoredups
\set HISTSIZE 10000

-- Erreurs
\set VERBOSITY verbose

-- Prompt coloré
\set PROMPT1 '%[%033[1;34m%]%n@%/%[%033[0m%]%x%R%# '
\set PROMPT2 '%[%033[1;34m%]...%[%033[0m%]%R%# '

-- Auto-complétion
\set COMP_KEYWORD_CASE upper

-- Raccourcis utiles
\set show_slow_queries 'SELECT query, calls, total_time, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 20;'

-- Message de bienvenue
\echo ''
\echo '🐘 PostgreSQL - Configuration optimisée chargée'
\echo '   Astuce: Tapez \\? pour l''aide'
\echo ''
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Créez un .psqlrc**
   - Gagnez du temps à chaque session
   - Standardisez votre environnement

2. **Activez \timing**
   - Identifiez immédiatement les requêtes lentes
   - Validez vos optimisations

3. **Configurez \pset null**
   - Évitez la confusion entre NULL et chaîne vide
   - Déboguez plus facilement

4. **Utilisez \x auto**
   - Le meilleur des deux mondes
   - S'adapte automatiquement

5. **Définissez des raccourcis**
   - Requêtes fréquentes dans .psqlrc
   - Gagnez du temps

6. **Personnalisez votre prompt**
   - Voyez immédiatement où vous êtes
   - État de transaction visible

### ❌ À Éviter

1. **Ne mettez pas de mots de passe dans .psqlrc**
   - Utilisez `.pgpass` à la place
   - Sécurité avant tout

2. **N'activez pas ON_ERROR_STOP dans .psqlrc**
   - Trop strict pour l'usage interactif
   - Réservez-le aux scripts

3. **Ne forcez pas un format dans .psqlrc**
   - Gardez `aligned` par défaut
   - Changez selon le besoin

4. **N'oubliez pas de commenter votre .psqlrc**
   - Vous oublierez pourquoi vous avez fait ça
   - Facilitez la maintenance

---

## Commandes de Réinitialisation

Si vous avez fait des modifications et voulez revenir à la normale :

```bash
# Réinitialiser tous les pset
\pset border 1
\pset format aligned
\pset linestyle ascii
\pset null ''
\pset footer on

# Désactiver les options
\x off
\timing off
\t off

# Recharger .psqlrc
\i ~/.psqlrc
```

---

## Conclusion

La configuration de psql est essentielle pour une expérience optimale. Les points clés à retenir :

🎯 **Essentiels** :
- `\timing` pour mesurer les performances
- `\x auto` pour un affichage adaptatif
- `\pset null '(null)'` pour voir les NULL

🎨 **Confort** :
- Bordures et style Unicode
- Prompt personnalisé avec couleurs
- Variables et raccourcis

📁 **Automatisation** :
- Créez un `.psqlrc` robuste
- Définissez des raccourcis fréquents
- Standardisez votre environnement

💡 **Astuce finale** : Commencez simple avec les 3-4 configurations essentielles, puis ajoutez progressivement selon vos besoins !

---

**Prochaines sections** :
- Export/Import avec psql
- Meta-commandes avancées
- Scripts et automatisation

---


⏭️ [Export/Import (\copy, \i, \o)](/annexes/commandes-psql/03-export-import.md)
