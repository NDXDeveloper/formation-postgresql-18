🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2. Nouveauté PG 18 : Améliorations COPY et gestion du marqueur de fin `\.` en CSV

## Introduction

PostgreSQL 18, sorti en septembre 2025, apporte plusieurs améliorations à la commande `COPY`. Cette section explore en détail :

1. La **fin du double sens du marqueur `\.`** lors de la lecture depuis un fichier
2. Les nouvelles options **`REJECT_LIMIT`** et **`LOG_VERBOSITY 'silent'`** qui complètent `ON_ERROR`
3. La possibilité de **`COPY TO` depuis une vue matérialisée**
4. L'**interdiction de `FREEZE` sur les *foreign tables***

Ces changements rendent l'importation de données plus prévisible, plus robuste et plus tolérante aux erreurs, tout en restant rétrocompatibles pour l'écrasante majorité des scripts existants.

---

## Comprendre le contexte historique

### Le marqueur de fin `\.` : un héritage du format texte

Pour comprendre l'amélioration apportée par PostgreSQL 18, il faut d'abord comprendre l'origine du marqueur.

#### Le format texte de PostgreSQL

Historiquement, PostgreSQL utilise un format spécifique (non standardisé) appelé « format texte » pour ses opérations d'import/export via `COPY`. Dans ce format :

- Les colonnes sont séparées par des **tabulations** (`\t`)
- Les lignes sont séparées par des **retours à la ligne** (`\n`)
- Les valeurs `NULL` sont représentées par `\N`
- La **fin des données** était traditionnellement marquée par **une ligne contenant uniquement `\.`** (un antislash suivi d'un point)

#### Exemple de format texte

Flux de données au format texte PostgreSQL :

```text
1	Dupont	Marie	marie.dupont@example.com	45000.00
2	Martin	Pierre	pierre.martin@example.com	52000.00
3	Durand	Sophie	sophie.durand@example.com	48000.00
\.
```

La dernière ligne `\.` indique à PostgreSQL : « c'est la fin des données, arrête de lire ».

#### Pourquoi ce marqueur existe-t-il ?

Le marqueur `\.` a été conçu principalement pour les flux dans lesquels la fin de fichier (EOF) **n'arrive pas** naturellement, typiquement :

1. **`COPY FROM STDIN` dans psql** : le script SQL continue après le `COPY`, donc psql a besoin d'un délimiteur explicite pour savoir où s'arrête la donnée et où reprend le SQL.
2. **Saisie interactive** : un utilisateur tapant des lignes au clavier doit pouvoir signaler la fin.

#### Exemple d'utilisation avec STDIN

```bash
psql -d mabase << 'EOF'  
COPY employes (id, nom, prenom, email) FROM STDIN;  
1	Dupont	Marie	marie@example.com
2	Martin	Pierre	pierre@example.com
\.
SELECT count(*) FROM employes;  
EOF  
```

Le `\.` indique ici la fin du flux de données ; le `SELECT` qui suit est interprété comme une nouvelle commande SQL.

---

## Le problème historique

### Règle PG 17 et antérieures

Dans **toutes** les versions antérieures à PostgreSQL 18, `COPY FROM` reconnaissait `\.` comme marqueur de fin **dès lors qu'il apparaissait seul sur une ligne**, et ce **quel que soit le contexte** :

| Source de données | Format | `\.` seul sur une ligne | Résultat PG 17 |
|-------------------|--------|------------------------|----------------|
| STDIN (psql) | text | Présent | Fin reconnue (attendu) |
| STDIN (psql) | csv | Présent | Fin reconnue (attendu) |
| **Fichier** | text | Présent | Fin reconnue (souvent attendu) |
| **Fichier** | **csv** | **Présent** | **Fin reconnue — ⚠️ piège** |

Le **piège** se cache dans la dernière ligne du tableau : un fichier CSV légitime contenant `\.` seul sur une ligne provoquait l'arrêt prématuré de l'import — alors qu'en CSV, le marqueur n'a aucune raison d'être (le fichier a une fin de fichier naturelle).

### Cas concrets affectés en PG 17

**À retenir** : pour déclencher le bug, il fallait qu'une **ligne entière** du flux soit exactement `\.`. Un `\.` à l'intérieur d'une cellule (par exemple `C:\.config\system.ini`) n'a **jamais** été interprété comme marqueur, car il n'est pas seul sur sa ligne.

Les scénarios réellement piégés étaient :

#### 1. Cellule contenant un saut de ligne, puis `\.` sur la ligne suivante

```csv
id,description
1,"Texte sur
\.
plusieurs lignes"
2,"Autre donnée"
```

Ici, à l'intérieur de la cellule quotée de la ligne 1, on a un retour à la ligne suivi de `\.` seul, puis un autre retour à la ligne. Selon les versions et le parseur, ce `\.` interne pouvait être interprété comme une fin prématurée.

#### 2. CSV mal formé ou concaténé

Un fichier produit par concaténation de plusieurs exports `COPY ... TO` au format texte, où chaque bloc se terminait par `\.`, et que l'on tentait de relire en CSV :

```csv
nom,valeur  
a,1  
b,2  
\.
c,3  
d,4  
```

→ PG 17 s'arrêtait à `\.` et n'importait que `a,1` et `b,2`.

#### 3. Données générées par un outil ajoutant systématiquement `\.`

Certaines moulinettes d'export, par mimétisme avec le format texte historique, ajoutaient `\.` en fin de fichier CSV — provoquant en PG 17 un comportement « ça marche aujourd'hui, mais comment ? ».

### Contournements dans les versions antérieures

Avant PostgreSQL 18, les solutions étaient toutes imparfaites :

1. **Nettoyer les fichiers** en amont (script `sed`/`awk` pour échapper ou supprimer les `\.` isolés) — fragile et ajoute une étape.
2. **Utiliser le format text avec un délimiteur exotique** — perte du standard CSV.
3. **Passer par une table de *staging* en `TEXT`** puis valider et convertir — robuste mais coûteux en code.

---

## L'amélioration de PostgreSQL 18

### Changement de règle (release notes officielles)

> *« Disallow `COPY FROM` from treating `\.` as an end-of-data marker in CSV mode »* (Daniel Vérité, Tom Lane)

**Règle nouvelle, depuis PostgreSQL 18 :**

| Source | Format | `\.` seul sur une ligne | Comportement PG 18 |
|--------|--------|------------------------|---------------------|
| **Fichier** | **csv** | Présent | **Traité comme donnée** (changement) |
| Fichier | text | Présent | Toujours marqueur de fin (inchangé) |
| STDIN (psql) | text | Présent | Toujours marqueur de fin (compatibilité psql) |
| STDIN (psql) | csv | Présent | Toujours marqueur de fin (compatibilité psql) |

Autrement dit :
- **Lecture depuis un fichier** : le format CSV est désormais autonome — la fin de fichier (EOF) suffit. Le format `text` conserve le marqueur pour la rétrocompatibilité.
- **Lecture depuis STDIN via psql** : `\.` reste indispensable comme délimiteur logique entre les données et la suite du script SQL, en CSV comme en text.

Note de la documentation officielle :

> *Pour la compatibilité avec les anciennes versions, `COPY TO` continue de **quoter** `\.` quand il se trouve seul sur une ligne, même si ce n'est plus strictement nécessaire en PG 18.*

### Exemple : import d'un CSV contenant `\.` seul sur une ligne

Fichier `notations.csv` :

```csv
id,notation
1,"Méthode habituelle"
2,"\."
3,"\."
4,"Donnée finale"
```

(Les cellules `\.` ici sont volontairement entre guillemets et donc pas seules sur leur ligne — illustration : `\.` ne déclenche pas le marqueur dans ce cas, ni en PG 17 ni en PG 18.)

Fichier `notations_piege.csv` (cas qui était problématique) :

```csv
id,notation
1,Méthode habituelle
\.
2,Donnée après marqueur
```

#### En PostgreSQL 17

```sql
COPY notations FROM '/tmp/notations_piege.csv' WITH (FORMAT csv, HEADER true);
-- Résultat : 1 seule ligne importée (id = 1)
-- La ligne 2 est ignorée, sans erreur visible : import silencieusement tronqué.
```

#### En PostgreSQL 18

```sql
COPY notations FROM '/tmp/notations_piege.csv' WITH (FORMAT csv, HEADER true);
-- Résultat : selon le contenu de la ligne `\.`, soit elle est importée comme
-- une ligne de données (si elle a le bon nombre de colonnes après parsing CSV),
-- soit elle déclenche une erreur de format CSV.
-- L'import ne s'arrête plus silencieusement.
```

Cette différence est **précieuse** : un import tronqué sans erreur est l'un des bugs les plus difficiles à détecter en production.

### Comportement STDIN inchangé

Pour maintenir la compatibilité avec **les milliers de scripts** utilisant `COPY FROM STDIN` (le pattern le plus courant dans les dumps `pg_dump` au format SQL, par exemple), le marqueur `\.` reste reconnu en STDIN :

```bash
psql -d mabase << 'EOF'  
COPY employes (nom, prenom) FROM STDIN WITH (FORMAT csv);  
Dupont,Marie  
Martin,Pierre  
\.
SELECT count(*) FROM employes;  
EOF  
```

Sans le `\.`, psql ne pourrait pas distinguer la fin du flux CSV du début du `SELECT` qui suit.

> ⚠️ **Effet de bord sur les anciens clients psql** : les notes officielles signalent qu'un **client psql antérieur à 18** se connectant à un **serveur PostgreSQL 18** peut rencontrer des comportements inattendus avec `\copy` (la méta-commande côté client) sur les fichiers CSV. Mettez à jour `psql` côté client pour éviter ces écarts.

#### Pourquoi `pg_dump` n'est pas affecté

Un dump SQL produit par `pg_dump` (sans option `-Fc`/`-Fd`) contient typiquement, pour chaque table :

```sql
COPY public.employes (id, nom, prenom) FROM stdin;
1   Dupont  Marie
2   Martin  Pierre
\.
```

Quand on rejoue ce dump avec `psql -f dump.sql`, psql lit ces lignes depuis **STDIN** (par rapport au `COPY`) : le marqueur `\.` reste donc indispensable et est correctement interprété, en PG 17 comme en PG 18. **C'est exactement pour ne pas casser ces millions de dumps existants** que le comportement STDIN a été préservé.

---

## Nouvelles options de gestion d'erreurs

Pour les imports d'origine externe (CSV venant d'un partenaire, d'un export tiers, etc.), il est fréquent d'avoir quelques lignes mal formées dans un fichier par ailleurs valide. PostgreSQL 17 avait introduit `ON_ERROR 'ignore'` ; PostgreSQL 18 ajoute deux options pour mieux le contrôler.

### Rappel : `ON_ERROR` (PostgreSQL 17)

```sql
COPY employes FROM '/tmp/employes.csv'  
WITH (FORMAT csv, HEADER true, ON_ERROR 'ignore');  
```

- Les lignes pour lesquelles la **conversion de type** échoue (par exemple `"abc"` dans une colonne `INTEGER`) sont **ignorées** au lieu d'avorter la commande.
- Les autres violations (`NOT NULL`, `UNIQUE`, clés étrangères, `CHECK`) ne sont **pas** captées et provoquent toujours l'erreur de COPY classique.
- Valeurs possibles : `'stop'` (défaut, comportement historique) ou `'ignore'`.

### `REJECT_LIMIT` (PostgreSQL 18)

Plafonne le nombre de lignes que `ON_ERROR 'ignore'` est autorisé à ignorer :

```sql
COPY employes FROM '/tmp/employes.csv'  
WITH (FORMAT csv, HEADER true,  
      ON_ERROR 'ignore',  
      REJECT_LIMIT 100);  
```

- Si **plus de 100** lignes sont invalides, `COPY` échoue (avec rollback total).
- Si **100 ou moins**, l'import réussit, et un `NOTICE` indique combien ont été ignorées.

C'est un garde-fou précieux : on tolère « un peu de bruit » dans les données, mais on refuse un fichier massivement corrompu (qui pourrait masquer un vrai problème en amont).

`REJECT_LIMIT` n'est valide qu'en combinaison avec `ON_ERROR 'ignore'`.

### `LOG_VERBOSITY` (PostgreSQL 18)

Contrôle le détail des messages produits par `ON_ERROR 'ignore'` :

```sql
-- Mode silencieux : ne pas spammer le journal pour chaque ligne ignorée
COPY employes FROM '/tmp/employes.csv'  
WITH (FORMAT csv, HEADER true,  
      ON_ERROR 'ignore',  
      LOG_VERBOSITY 'silent');  
```

Valeurs :
- `'default'` (par défaut) : un seul `NOTICE` **récapitulatif** en fin de commande (« N rows were skipped due to data type incompatibility »).
- `'verbose'` : un `NOTICE` **par ligne ignorée** (numéro de ligne, colonne et valeur fautive), suivi du récapitulatif.
- `'silent'` (**nouveau PG 18**) : **aucun** message — ni par ligne, ni récapitulatif.

Utile pour un import volumineux où l'on ne veut **aucun bruit** dans le journal : le mode `'verbose'` peut sinon émettre des milliers de `NOTICE`, un par ligne ignorée.

### Exemple combiné

```sql
-- Charger un export quotidien d'un partenaire :
-- - on tolère jusqu'à 50 lignes en erreur de type
-- - au-delà, on rejette tout l'import
-- - on garde le journal lisible
COPY ventes (date_vente, client_id, produit_id, montant)  
FROM '/imports/ventes_J-1.csv'  
WITH (  
    FORMAT csv,  
    HEADER true,  
    ON_ERROR 'ignore',  
    REJECT_LIMIT 50,  
    LOG_VERBOSITY 'silent'  
);  
```

---

## `COPY TO` depuis une vue matérialisée

PostgreSQL 18 lève une restriction historique : `COPY TO` accepte désormais une **vue matérialisée peuplée** comme source.

### Avant (PG 17 et antérieures)

```sql
COPY ventes_par_mois TO '/tmp/ventes.csv' WITH (FORMAT csv, HEADER true);
-- ERROR: cannot copy from materialized view "ventes_par_mois"
```

Il fallait passer par une sous-requête :

```sql
COPY (SELECT * FROM ventes_par_mois) TO '/tmp/ventes.csv' WITH (FORMAT csv, HEADER true);
```

### Depuis PG 18

```sql
-- Fonctionne désormais directement
COPY ventes_par_mois TO '/tmp/ventes.csv' WITH (FORMAT csv, HEADER true);
```

La vue matérialisée doit être **peuplée** (avoir reçu au moins un `REFRESH MATERIALIZED VIEW`) ; une vue matérialisée vide produit toujours une erreur.

> 📌 La forme `COPY (SELECT … FROM mv) TO …` reste utilisable et permet de filtrer/transformer à la volée.

---

## Restriction `FREEZE` sur les *foreign tables*

`COPY FREEZE` est une option qui marque les lignes comme « gelées » dès l'insertion : utile pour pré-remplir une table sans déclencher de `VACUUM FREEZE` ultérieur. Cette option n'a jamais eu de sens sur une *foreign table* (table distante), puisque le moteur PostgreSQL local ne contrôle pas le stockage physique de l'autre côté.

| Version | Comportement |
|---------|--------------|
| PG 17 et antérieures | `COPY foreign_table FROM … FREEZE` accepté, **mais `FREEZE` ignoré silencieusement** |
| **PG 18** | **Erreur explicite** : `FREEZE option not supported for COPY to foreign tables` |

C'est un changement « cosmétique » mais utile : il évite de croire qu'on a un gain de performance qui en réalité n'existe pas.

---

## Compatibilité et migration

### Migration vers PostgreSQL 18 : check-list

| Pratique | Action |
|----------|--------|
| `COPY FROM '...' WITH (FORMAT csv)` sur un fichier sans `\.` isolé | ✅ Rien à faire |
| `COPY FROM '...' WITH (FORMAT csv)` sur un fichier **contenant** `\.` isolé | ✅ L'import qui était tronqué silencieusement va maintenant aller au bout |
| `COPY FROM STDIN` dans un script psql (dump, init data…) | ✅ Rien à faire, `\.` reste reconnu |
| Script avec contournement maison du `\.` | 🧹 Vous pouvez le retirer |
| `\copy` depuis un client **psql ≤ 17** vers un serveur PG 18 | ⚠️ Mettre à jour psql côté client |
| `COPY foreign_table FROM … FREEZE` | ❌ Retirer `FREEZE` ou cibler une table locale |
| Pipeline ignorant manuellement des lignes erronées | 💡 Envisager `ON_ERROR 'ignore'` + `REJECT_LIMIT` |

### Avant / après : un script qui contournait le bug

#### Avant (PG ≤ 17)

```bash
#!/bin/bash
# Le fichier CSV reçu d'un partenaire contient parfois "\." en cellule
# Échappement préventif pour éviter une troncature silencieuse en COPY
sed 's/^\\\.$/__BACKSLASH_DOT__/g' input.csv > clean.csv

psql -d mabase -c "COPY t FROM '/tmp/clean.csv' WITH (FORMAT csv, HEADER true)"

psql -d mabase -c "UPDATE t SET col = '\.' WHERE col = '__BACKSLASH_DOT__'"  
rm clean.csv  
```

#### Après (PG 18)

```bash
#!/bin/bash
psql -d mabase -c "COPY t FROM '/tmp/input.csv' WITH (FORMAT csv, HEADER true)"
```

---

## Tests de validation après migration

Voici un script reproductible pour vérifier le nouveau comportement.

```bash
# Préparer un CSV qui aurait tronqué l'import en PG 17
cat > /tmp/test_pg18_copy.csv <<'EOF'  
id,libelle  
1,Avant le piège
\.
2,Après le piège
EOF
```

```sql
-- 1. Table de test
CREATE TABLE test_pg18_copy (id INTEGER, libelle TEXT);

-- 2. Import
COPY test_pg18_copy FROM '/tmp/test_pg18_copy.csv'  
WITH (FORMAT csv, HEADER true, ON_ERROR 'ignore');  

-- 3. Comptage
SELECT count(*) FROM test_pg18_copy;
--   PG 17 : 1   (import tronqué silencieusement à la ligne "\.")
--   PG 18 : 1 ou 2 selon que la ligne "\." est rejetée par ON_ERROR ou non,
--           mais dans tous les cas la ligne "Après le piège" est TENTÉE.

-- 4. Détail
SELECT * FROM test_pg18_copy ORDER BY id;

-- 5. Nettoyage
DROP TABLE test_pg18_copy;
```

L'élément clé du test : **en PG 18, la ligne `2,Après le piège` n'est plus ignorée**.

---

## Questions fréquentes (FAQ)

### Q1 — Les anciens scripts vont-ils casser après upgrade vers PG 18 ?

**Non**, le changement est rétrocompatible pour les usages réels :
- `COPY FROM STDIN` (les dumps `pg_dump`, les scripts d'init) : comportement strictement inchangé.
- `COPY FROM '…' (FORMAT text)` : comportement strictement inchangé.
- `COPY FROM '…' (FORMAT csv)` : seuls les imports qui étaient **tronqués silencieusement** changent — et le nouveau comportement est désirable.

Le seul piège est un **client `psql` plus ancien que 18** parlant à un **serveur PG 18**, pour la méta-commande `\copy` sur CSV : alignez les versions.

### Q2 — Dois-je nettoyer mes fichiers CSV pour PG 18 ?

Non. Au contraire, des fichiers qui étaient « toxiques » en PG 17 (`\.` isolé) sont maintenant lisibles.

### Q3 — Le format `text` est-il affecté ?

**Non.** En format `text`, `\.` reste un marqueur de fin de données (qu'on lise depuis un fichier ou depuis STDIN). Le changement de PG 18 ne concerne que le **format CSV** et la **source fichier**.

### Q4 — Comment garder l'ancien comportement « \\. = EOF » si je le voulais en CSV ?

Vous ne pouvez plus en lecture depuis fichier. Si votre format de données utilise réellement `\.` comme marqueur, il faut soit l'importer en `FORMAT text`, soit le pipe-er via STDIN avec psql, soit retirer la ligne `\.` du fichier avant import.

### Q5 — Quelle est l'amélioration de performance de `COPY` en PG 18 ?

Les release notes ne documentent **pas** d'optimisation directe de `COPY` lui-même. En revanche, deux évolutions du moteur peuvent indirectement accélérer les imports massifs :
- Le **nouveau sous-système d'E/S asynchrones (AIO)**, configurable via `io_method`, qui peut améliorer les opérations d'E/S intensives.
- Les **améliorations des jointures de hachage et `GROUP BY`** qui bénéficient aux traitements post-import (analytique, agrégation).

Les gains réels dépendent fortement du matériel et de la charge — il est sage de **mesurer sur vos propres données** plutôt que de citer un pourcentage générique.

### Q6 — Puis-je combiner `ON_ERROR`, `REJECT_LIMIT` et `LOG_VERBOSITY` ?

Oui :

```sql
COPY t FROM '/tmp/data.csv'  
WITH (FORMAT csv, HEADER true,  
      ON_ERROR 'ignore',  
      REJECT_LIMIT 100,  
      LOG_VERBOSITY 'silent');  
```

`REJECT_LIMIT` et `LOG_VERBOSITY 'silent'` n'ont de sens qu'avec `ON_ERROR 'ignore'`.

### Q7 — `ON_ERROR` couvre-t-il les violations de contraintes ?

**Non.** `ON_ERROR 'ignore'` ne capte que les erreurs de conversion d'entrée (mauvais type, format de date invalide, etc.). Les violations de `NOT NULL`, `UNIQUE`, `CHECK`, clés étrangères provoquent toujours l'avortement de la commande, quel que soit le réglage de `ON_ERROR`.

---

## Comparatif récapitulatif : PG 17 vs PG 18

| Aspect | PostgreSQL 17 | PostgreSQL 18 |
|--------|---------------|---------------|
| `\.` seul sur une ligne, `COPY FROM` **fichier CSV** | Marqueur de fin (import tronqué silencieusement) | **Donnée normale** (import complet) |
| `\.` seul sur une ligne, `COPY FROM STDIN` | Marqueur de fin | Marqueur de fin (inchangé) |
| `\.` seul sur une ligne, `COPY FROM` fichier `text` | Marqueur de fin | Marqueur de fin (inchangé) |
| `COPY TO` depuis vue matérialisée peuplée | ❌ Erreur (sous-requête requise) | ✅ Direct |
| `COPY FREEZE` sur *foreign table* | Silencieusement ignoré | ❌ Erreur explicite |
| `ON_ERROR 'ignore'` | ✅ (déjà PG 17) | ✅ (inchangé) |
| `REJECT_LIMIT` | ❌ | ✅ **Nouveau** |
| `LOG_VERBOSITY 'silent'` | ❌ | ✅ **Nouveau** |

---

## Conclusion

Les évolutions de `COPY` dans PostgreSQL 18 sont **discrètes mais utiles** :

- **Robustesse** : un piège silencieux du parseur CSV (`\.` isolé interprété comme EOF) disparaît. Un import qui semblait fonctionner en PG 17 mais tronquait parfois ses données va désormais aller au bout — ou échouer explicitement si la ligne pose réellement problème.
- **Tolérance** : `REJECT_LIMIT` et `LOG_VERBOSITY 'silent'` complètent `ON_ERROR`, transformant `COPY` en outil d'ingestion fiable même sur des données « sales ».
- **Cohérence** : `COPY TO` accepte les vues matérialisées peuplées comme n'importe quelle table ; `FREEZE` est refusé sur ce qui n'a pas de sens.

Le changement le plus important à retenir pour un développeur ou un DBA migrant vers PG 18 : **revérifiez vos imports CSV historiques** — il est possible que certains d'entre eux ramenaient en silence moins de lignes qu'attendu.

---


⏭️ [UPDATE et DELETE : Ciblage et précautions](/06-manipulation-des-donnees/03-update-et-delete.md)
