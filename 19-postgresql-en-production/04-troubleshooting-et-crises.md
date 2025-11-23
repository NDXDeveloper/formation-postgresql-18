🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.4. Troubleshooting et Crises : Introduction

## Bienvenue dans la section la plus importante de cette formation

Si vous deviez ne retenir qu'une seule section de cette formation complète sur PostgreSQL, ce devrait être celle-ci.

Pourquoi ? Parce que **tous les systèmes finissent par rencontrer des problèmes**. Peu importe la qualité de votre architecture, la robustesse de votre code ou la performance de votre matériel, un jour ou l'autre, vous serez confronté à une situation critique :

- 🔥 Votre application est soudainement très lente
- 💥 Les utilisateurs ne peuvent plus se connecter
- 📉 Les requêtes prennent 100 fois plus de temps que d'habitude
- 🚨 Votre base de données refuse toute nouvelle transaction
- ⚠️ Des données semblent avoir disparu ou sont corrompues

**Dans ces moments-là**, la différence entre un simple incident mineur et une catastrophe majeure dépend de :
1. Votre capacité à **diagnostiquer rapidement** le problème
2. Votre connaissance des **outils** à votre disposition
3. Votre maîtrise des **techniques de résolution**
4. Votre **calme** et votre méthodologie

---

## Qu'est-ce que le Troubleshooting ?

### Définition simple

Le **troubleshooting** (ou dépannage) est l'art de :
1. **Identifier** qu'un problème existe
2. **Diagnostiquer** la cause racine
3. **Résoudre** le problème
4. **Prévenir** sa réapparition

**Analogie médicale** : C'est comme être médecin pour votre base de données :
- Les **symptômes** : L'application est lente, erreurs de connexion
- Le **diagnostic** : Interroger le patient (la base de données) avec les bons outils
- Le **traitement** : Appliquer la solution appropriée
- La **prévention** : Mettre en place monitoring et bonnes pratiques

### Troubleshooting vs Crises

Il existe deux contextes différents :

#### Troubleshooting (mode normal)

- Problème identifié **avant** qu'il impacte les utilisateurs
- Temps disponible pour analyser en profondeur
- Possibilité de tester plusieurs solutions
- Pression faible

**Exemple** : Vous remarquez dans votre monitoring que certaines requêtes deviennent lentes. Vous avez le temps d'analyser, d'optimiser, de tester.

#### Gestion de crise (mode urgence)

- Problème **en cours**, utilisateurs impactés
- Chaque seconde compte
- Besoin de solution rapide (pas forcément parfaite)
- Pression élevée, stress

**Exemple** : Il est 14h un mardi, votre site e-commerce est inaccessible, vous perdez 10,000€ par minute. Vous devez agir **maintenant**.

---

## Les 6 familles de problèmes PostgreSQL

Cette section couvre les 6 types de problèmes les plus courants que vous rencontrerez en production :

### 1. 🔒 Verrous et blocages (Locks)
**Symptôme** : Requêtes qui "pendent", timeouts, application figée
**Cause** : Transactions qui se bloquent mutuellement
**Impact** : Lenteur voire blocage complet
**Gravité** : ⚠️⚠️ Moyen à Élevé

### 2. 💻 Saturation des ressources (CPU, RAM, I/O)
**Symptôme** : Lenteur généralisée, serveur qui rame
**Cause** : Ressources matérielles saturées
**Impact** : Performance dégradée pour tous
**Gravité** : ⚠️⚠️⚠️ Élevé

### 3. ♻️ Transaction Wraparound (XID exhaustion)
**Symptôme** : Warnings dans les logs, puis arrêt complet
**Cause** : Épuisement des identifiants de transaction
**Impact** : Base de données en lecture seule forcée
**Gravité** : ⚠️⚠️⚠️⚠️ Critique

### 4. 💔 Corruption de données
**Symptôme** : Erreurs de checksum, données invalides
**Cause** : Défaillance matérielle ou bug
**Impact** : Perte de données possible
**Gravité** : ⚠️⚠️⚠️⚠️⚠️ Catastrophique

### 5. 🐌 Requêtes lentes (Slow Queries)
**Symptôme** : Certaines opérations prennent beaucoup de temps
**Cause** : Requêtes mal optimisées, index manquants
**Impact** : Expérience utilisateur dégradée
**Gravité** : ⚠️ Faible à Moyen

### 6. 🌊 Tempêtes de connexions (Connection Storms)
**Symptôme** : "Too many clients", connexions refusées
**Cause** : Trop de connexions simultanées
**Impact** : Application inaccessible
**Gravité** : ⚠️⚠️⚠️ Élevé

---

## La méthodologie universelle de troubleshooting

Quelle que soit la nature du problème, suivez toujours cette approche méthodique :

### Étape 1 : OBSERVER (2-5 minutes)

**Ne paniquez pas. Observez d'abord.**

```
Questions à se poser :
├─ Quand le problème a-t-il commencé ?
├─ Le problème est-il constant ou intermittent ?
├─ Quel est l'impact exact ? (Tout est lent ? Certaines requêtes seulement ?)
├─ Y a-t-il eu un changement récent ? (Déploiement, migration, pic de trafic)
└─ Le problème affecte-t-il une base, une table, ou tout le système ?
```

**Outils d'observation rapide** :

```sql
-- Vue d'ensemble des connexions actives
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state;

-- Requêtes actives longues
SELECT pid, now() - query_start as duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC
LIMIT 5;

-- Charge système
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
```

```bash
# CPU, RAM, I/O
top
htop
iostat -x 1 5
```

### Étape 2 : DIAGNOSTIQUER (5-15 minutes)

**Identifiez la cause racine, pas seulement les symptômes.**

**Analogie** : Si vous avez mal à la tête, ce n'est pas le problème en soi. C'est un symptôme. La cause peut être : déshydratation, stress, manque de sommeil, etc. Il faut traiter la cause, pas seulement le symptôme.

**Techniques de diagnostic** :

1. **Consultation des logs** :
```bash
sudo tail -f /var/log/postgresql/postgresql-*.log
sudo grep -i "error\|fatal\|panic" /var/log/postgresql/postgresql-*.log | tail -20
```

2. **Analyse des statistiques** :
```sql
-- Avec pg_stat_statements
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

3. **Vérification des verrous** :
```sql
-- Processus bloqués
SELECT pid, pg_blocking_pids(pid), query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

4. **Surveillance des ressources** :
```sql
-- Cache hit ratio
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100
    AS cache_hit_ratio
FROM pg_statio_user_tables;
```

### Étape 3 : CONTENIR (immédiat si crise)

**En situation de crise, stabilisez avant de réparer.**

**Actions de stabilisation** :

| Problème détecté | Action immédiate |
|------------------|------------------|
| Requête bloquante | `pg_cancel_backend()` ou `pg_terminate_backend()` |
| Saturation connexions | Tuer connexions idle anciennes |
| CPU saturé | Identifier et tuer requête lourde |
| Disque plein | Archiver/supprimer logs, vacuumer |
| RAM saturée | Redémarrer services non-essentiels |

**Exemple** :
```sql
-- Tuer une requête problématique
SELECT pg_terminate_backend(12345);  -- PID de la requête

-- Tuer toutes les connexions idle > 10 minutes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '10 minutes';
```

### Étape 4 : RÉSOUDRE (temps variable)

**Appliquez la solution appropriée selon la cause identifiée.**

**Matrice de décision** :

```
Cause identifiée → Solution
├─ Verrou : Terminer transaction bloquante
├─ Requête lente : Optimiser ou créer index
├─ Saturation CPU : Optimiser requêtes ou augmenter ressources
├─ RAM pleine : Ajuster work_mem, augmenter RAM
├─ I/O saturé : Optimiser checkpoints, meilleur disque
├─ Wraparound : VACUUM FREEZE immédiat
├─ Corruption : Restaurer depuis backup
└─ Trop de connexions : Connection pooling (PgBouncer)
```

### Étape 5 : VÉRIFIER (5 minutes)

**Ne partez jamais sans vérifier que le problème est vraiment résolu.**

```sql
-- Vérifier que les performances sont revenues à la normale
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- Vérifier les métriques clés
SELECT count(*) FROM pg_stat_activity;

-- Tester une requête représentative
\timing
SELECT count(*) FROM orders WHERE created_at > now() - interval '1 day';
```

### Étape 6 : DOCUMENTER (15-30 minutes)

**Créez un post-mortem pour apprendre et prévenir.**

**Template de post-mortem** :

```markdown
# Incident du [DATE]

## Résumé
- Date/Heure : 2024-11-23 14:35:00
- Durée : 23 minutes
- Impact : Site web inaccessible
- Utilisateurs affectés : ~5,000

## Chronologie
14:35 - Alertes : CPU > 95%, connexions refusées
14:37 - Investigation démarrée
14:40 - Cause identifiée : Requête lente non optimisée
14:45 - Requête tuée, index créé
14:58 - Service complètement rétabli

## Cause racine
Nouvelle feature déployée avec requête SELECT * FROM orders
sans index sur colonne de filtrage.

## Solution appliquée
- Court terme : Tué la requête problématique
- Long terme : Créé index sur orders(status, created_at)

## Actions préventives
- [ ] Revoir processus code review (EXPLAIN obligatoire)
- [ ] Ajouter alerting sur slow queries (> 1s)
- [ ] Former équipe sur optimisation requêtes

## Leçons apprises
1. Toujours tester avec données de volume production
2. EXPLAIN ANALYZE avant déploiement requêtes
3. Monitoring proactif aurait permis détection précoce
```

---

## Les outils essentiels du troubleshooter

### Outils SQL intégrés

```sql
-- 1. pg_stat_activity : Votre tableau de bord
SELECT * FROM pg_stat_activity;

-- 2. pg_stat_statements : Historique des requêtes
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- 3. pg_locks : Voir les verrous
SELECT * FROM pg_locks WHERE NOT granted;

-- 4. EXPLAIN ANALYZE : Comprendre les requêtes
EXPLAIN ANALYZE SELECT ...;

-- 5. pg_stat_database : Métriques par base
SELECT * FROM pg_stat_database;
```

### Outils système

```bash
# Monitoring système
top               # Vue d'ensemble CPU/RAM
htop              # Version améliorée de top
iostat -x 1 5     # I/O disque
vmstat 1 5        # Mémoire virtuelle
dstat             # Statistiques complètes

# Logs
tail -f /var/log/postgresql/postgresql-*.log
journalctl -u postgresql -f

# Processus PostgreSQL
ps aux | grep postgres
```

### Extensions PostgreSQL essentielles

```sql
-- pg_stat_statements (OBLIGATOIRE)
CREATE EXTENSION pg_stat_statements;

-- pg_stat_kcache (métriques système)
CREATE EXTENSION pg_stat_kcache;

-- amcheck (vérification intégrité index)
CREATE EXTENSION amcheck;

-- pg_buffercache (analyse du cache)
CREATE EXTENSION pg_buffercache;
```

### Outils externes

| Outil | Usage | Priorité |
|-------|-------|----------|
| **pgBadger** | Analyse de logs | ⭐⭐⭐ |
| **explain.depesz.com** | Visualiser EXPLAIN | ⭐⭐⭐ |
| **Grafana + Prometheus** | Monitoring temps réel | ⭐⭐⭐ |
| **PgBouncer** | Connection pooling | ⭐⭐⭐ |
| **pg_top** | Monitoring interactif | ⭐⭐ |
| **HypoPG** | Test index hypothétiques | ⭐⭐ |

---

## État d'esprit et attitude face aux crises

### Les 10 commandements du troubleshooter

1. **Tu ne paniqueras point**
   - Le stress empêche de réfléchir clairement
   - Respire profondément, lis méthodiquement

2. **Tu mesureras avant d'agir**
   - "Je pense que..." n'est pas une mesure
   - Collecte des données avant toute action

3. **Tu ne devineras point**
   - Ne supposez pas, vérifiez
   - Les hypothèses non vérifiées causent des erreurs

4. **Tu documenteras tout**
   - Chaque commande exécutée
   - Chaque résultat observé
   - Utile pour post-mortem et futurs incidents

5. **Tu ne changeras qu'une chose à la fois**
   - Sinon, impossible de savoir ce qui a fonctionné
   - Test, mesure, puis prochaine action

6. **Tu sauvegarderas avant de modifier**
   - Toujours avoir un plan B
   - Pouvoir revenir en arrière

7. **Tu demanderas de l'aide si nécessaire**
   - Pas de honte à demander assistance
   - Deux cerveaux valent mieux qu'un

8. **Tu communiqueras clairement**
   - Tenir les parties prenantes informées
   - Expliquer ce qui se passe, ce que vous faites

9. **Tu apprendras de chaque incident**
   - Chaque problème est une opportunité d'apprentissage
   - Post-mortem systématique

10. **Tu préviendras plutôt que guériras**
    - Monitoring proactif
    - Alertes avant que ça casse
    - Tests de charge réguliers

### La checklist "Panique Zéro"

Quand vous êtes face à une crise, affichez cette checklist à votre écran :

```
☐ J'ai pris une grande respiration
☐ J'ai noté l'heure de début de l'incident
☐ J'ai ouvert un document pour noter mes observations
☐ J'ai vérifié que j'ai les accès nécessaires
☐ J'ai prévenu les parties prenantes que j'investigue
☐ Je sais comment revenir en arrière si mes actions empirent les choses
☐ J'ai un collègue en backup si besoin
☐ Je ne vais changer qu'UNE SEULE chose à la fois
```

### Les erreurs classiques à éviter

❌ **"Je suis sûr que c'est ça, je vais directement appliquer la solution"**
→ ✅ Diagnostiquer d'abord, même si vous pensez savoir

❌ **"Je vais redémarrer PostgreSQL, ça règle souvent les problèmes"**
→ ✅ Le redémarrage efface les symptômes mais pas la cause

❌ **"Je n'ai pas le temps de noter ce que je fais"**
→ ✅ 30 secondes de documentation peuvent sauver 30 minutes plus tard

❌ **"Je vais changer plusieurs paramètres en même temps pour aller vite"**
→ ✅ Une modification à la fois, sinon vous ne saurez pas ce qui a marché

❌ **"C'est bon, ça marche maintenant, pas besoin de comprendre pourquoi"**
→ ✅ Sans comprendre la cause, le problème reviendra

---

## Organisation de cette section

Cette section est organisée par type de problème, du plus fréquent au plus rare :

### 19.4.1. Diagnostic des verrous
- Comment identifier qui bloque qui
- Utilisation de `pg_locks` et `pg_blocking_pids()`
- Résoudre deadlocks et blocages

### 19.4.2. Saturation des ressources
- Diagnostiquer saturation CPU, RAM, I/O
- Identifier les requêtes gourmandes
- Solutions d'optimisation et scaling

### 19.4.3. Transaction Wraparound
- Comprendre le problème du XID exhaustion
- Détecter l'âge des transactions
- VACUUM préventif et curatif

### 19.4.4. Corruption de données
- Détection avec checksums (nouveauté PG 18)
- Utilisation de `amcheck`
- Stratégies de récupération

### 19.4.5. Slow queries et Query tuning
- Identifier les requêtes lentes avec `pg_stat_statements`
- Lire et interpréter `EXPLAIN ANALYZE`
- Techniques d'optimisation

### 19.4.6. Connection storms et pooling
- Comprendre le problème des connexions
- Configuration de PgBouncer
- Bonnes pratiques de connection management

---

## Votre kit de survie : Commandes rapides

### Diagnostic rapide (2 minutes)

```sql
-- 1. Vue d'ensemble
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- 2. Requêtes longues
SELECT pid, now()-query_start as duration, state, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC LIMIT 5;

-- 3. Blocages
SELECT pid, pg_blocking_pids(pid) as blocked_by
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- 4. Connexions par base
SELECT datname, count(*)
FROM pg_stat_activity
GROUP BY datname;
```

### Commandes d'urgence

```sql
-- Tuer une requête
SELECT pg_cancel_backend(PID);     -- Annuler gentiment
SELECT pg_terminate_backend(PID);  -- Forcer l'arrêt

-- Tuer toutes les connexions idle
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '10 minutes';

-- Voir l'utilisation de la mémoire
SELECT pg_size_pretty(pg_database_size(current_database()));
```

### Monitoring continu

```bash
# Script de monitoring simple (à lancer en continu)
watch -n 2 'psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"'
```

---

## Préparez-vous maintenant, pas pendant la crise

### Checklist de préparation

Avant qu'un incident ne survienne, assurez-vous d'avoir :

#### Infrastructure

- [ ] Monitoring configuré (Grafana/Prometheus ou équivalent)
- [ ] Alertes configurées (CPU, RAM, connexions, slow queries)
- [ ] Sauvegardes testées régulièrement
- [ ] Plan de disaster recovery documenté
- [ ] PgBouncer ou connection pooling en place
- [ ] pg_stat_statements activé

#### Documentation

- [ ] Accès aux serveurs documentés (qui a les clés SSH ?)
- [ ] Commandes de base documentées (cheat sheet)
- [ ] Liste des contacts d'urgence (qui appeler à 3h du matin ?)
- [ ] Runbooks pour incidents courants
- [ ] Architecture système documentée

#### Compétences

- [ ] Équipe formée aux bases de PostgreSQL
- [ ] Au moins 2 personnes maîtrisent le troubleshooting
- [ ] Exercices de simulation d'incidents (game days)
- [ ] Post-mortems documentés des incidents passés

#### Tests

- [ ] Tests de charge réguliers (pgbench)
- [ ] Tests de failover (si réplication)
- [ ] Tests de restauration de backup
- [ ] Tests d'évacuation (procédure dégradée)

---

## Exercice de préparation mentale

Avant de continuer vers les chapitres techniques, prenez 5 minutes pour :

### Scénario 1 : Il est 15h un vendredi

Votre site e-commerce tourne normalement. Soudain :
- Les pages prennent 30 secondes à charger au lieu de 200ms
- Les utilisateurs se plaignent massivement
- Votre chef vous appelle : "C'est quoi ce bug ?"

**Question** : Quelle est votre toute première action ?

<details>
<summary>Réponse recommandée</summary>

1. **Respirer** (3 secondes)
2. **Observer** avant d'agir :
   - Se connecter à la base de données
   - Lancer `SELECT count(*), state FROM pg_stat_activity GROUP BY state;`
   - Voir les requêtes actives : `SELECT pid, query FROM pg_stat_activity WHERE state = 'active';`
3. **Identifier** rapidement si c'est un problème :
   - De verrous (beaucoup de requêtes en attente)
   - De ressources (CPU à 100% ?)
   - De requête lente (une requête domine le temps d'exécution)
4. **Communiquer** : Prévenir l'équipe que vous êtes sur le problème
5. **Agir** selon ce que vous avez trouvé

**Ne JAMAIS faire** : Redémarrer immédiatement PostgreSQL (vous perdez les symptômes).
</details>

### Scénario 2 : Il est 3h du matin

Votre téléphone vibre. Alerte Slack : "PostgreSQL CRITICAL - Connections > 95%"

**Question** : Comment réagissez-vous ?

<details>
<summary>Réponse recommandée</summary>

1. **Se réveiller complètement** (30 secondes de vraie conscience)
2. **Vérifier** que c'est une vraie alerte, pas un faux positif
3. **Voir** l'état actuel : connexions réelles vs max_connections
4. **Identifier** si système est encore responsive
5. **Si responsive** : Investiguer calmement
6. **Si non responsive** : Action de stabilisation (tuer connexions idle)
7. **Documenter** ce que vous faites
8. **Résoudre** la cause racine une fois stabilisé

**Important** : Même à 3h du matin, suivez la méthodologie. La fatigue amplifie les erreurs.
</details>

---

## Message final avant de commencer

Le troubleshooting et la gestion de crises sont des **compétences qui se développent avec la pratique**.

Ne vous attendez pas à être expert après avoir lu cette section. L'expertise vient de :
- 20% de théorie (cette formation)
- 80% de pratique (vos futurs incidents)

**Chaque incident est une opportunité d'apprentissage.**

Les meilleurs DBA que je connais ont tous une chose en commun : ils ont fait BEAUCOUP d'erreurs, mais ils ont **appris de chacune d'entre elles**.

Alors :
- Lisez attentivement les chapitres suivants
- Pratiquez dans un environnement de test
- Documentez vos apprentissages
- N'ayez pas peur de vous tromper
- Et surtout, **restez curieux**

---

## Prêt ?

Vous avez maintenant :
- ✅ Compris l'importance du troubleshooting
- ✅ Découvert la méthodologie universelle
- ✅ Connu les 6 familles de problèmes
- ✅ Appris l'état d'esprit du troubleshooter
- ✅ Reçu votre kit de survie de commandes

Il est temps de plonger dans le vif du sujet.

Le premier type de problème que nous allons aborder est l'un des plus fréquents : **les verrous et blocages**.

---

**Prochain chapitre** : 19.4.1 - Diagnostic des verrous (pg_locks, pg_blocking_pids)

---


⏭️ [Diagnostic des verrous (pg_locks, pg_blocking_pids)](/19-postgresql-en-production/04.1-diagnostic-verrous.md)
