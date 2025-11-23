🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.3. Patterns Anti-Corruption : Introduction

## Qu'est-ce qu'un Pattern Anti-Corruption ?

### Définition

Un **pattern anti-corruption** est une pratique de développement qui vise à **protéger votre code** contre les dégradations de performance, les bugs et les inefficacités causées par une mauvaise interaction avec la base de données.

**Analogie :** Imaginez votre application comme une maison bien construite. Les patterns anti-corruption sont comme des systèmes de protection (isolation, étanchéité, ventilation) qui empêchent l'humidité, la moisissure et les infiltrations de détériorer la structure.

### Origine du Terme

Le terme "anti-corruption" vient du **Domain-Driven Design (DDD)** d'Eric Evans. Dans ce contexte, une "corruption" n'est pas malveillante, mais représente plutôt une **dégradation progressive** de la qualité du code due à :
- Des pratiques inadéquates
- Des abstractions qui fuient
- Des inefficacités non détectées
- Des raccourcis qui deviennent la norme

---

## Pourquoi est-ce Important ?

### Le Problème Fondamental

L'interaction entre votre application et PostgreSQL est l'un des **points critiques** de performance et de qualité. Un code qui fonctionne parfaitement avec 10 lignes peut s'effondrer avec 10,000 lignes.

**Symptômes typiques :**
- ✅ **En développement** : "Ça marche super bien !"
- ⚠️ **En staging** : "C'est un peu lent..."
- ❌ **En production** : "Le site est down !" 😱

### Les Coûts de la Corruption

**Performance :**
- Requêtes qui passent de 10ms à 10 secondes
- Saturation de la base de données
- Timeouts et erreurs 503

**Évolutivité :**
- Impossible de scaler au-delà d'un certain volume
- Coûts d'infrastructure qui explosent
- Solutions de contournement (caches) qui masquent le problème

**Maintenance :**
- Code difficile à comprendre et modifier
- Bugs difficiles à reproduire et corriger
- Dette technique qui s'accumule

**Expérience utilisateur :**
- Pages lentes ou qui ne chargent pas
- Frustration et abandon
- Perte de revenus

---

## Les Quatre Corruptions Principales

Dans ce chapitre, nous allons explorer les quatre anti-patterns les plus courants et destructeurs dans les applications utilisant PostgreSQL.

### 1. Le Problème N+1 Queries 🔴

**Qu'est-ce que c'est ?**
Exécuter 1 requête pour récupérer une liste, puis N requêtes supplémentaires pour chaque élément.

**Exemple typique :**
```python
# 1 requête pour les articles
articles = Article.objects.all()

# N requêtes pour les auteurs (1 par article)
for article in articles:
    print(article.author.name)  # Nouvelle requête à chaque itération !
```

**Impact :**
- 100 articles = 101 requêtes au lieu d'1
- Temps de réponse multiplié par 10 à 100
- Saturation rapide en production

**Solution :**
JOIN, LATERAL, eager loading

➡️ **Détaillé dans la section 20.3.1**

---

### 2. ORM vs SQL Brut : Le Dilemme 🤔

**Qu'est-ce que c'est ?**
Utiliser un ORM (Object-Relational Mapping) partout, même quand ce n'est pas adapté, ou à l'inverse, tout faire en SQL brut par peur de l'ORM.

**Le problème des extrêmes :**

**Trop d'ORM :**
```python
# ORM forcé pour une requête complexe → SQL inefficace
users = User.objects.annotate(
    article_count=Count('articles')
).filter(article_count__gte=10).annotate(
    rank=Window(expression=RowNumber(), order_by=F('article_count').desc())
)
# Code illisible et SQL généré peut-être sous-optimal
```

**Trop de SQL brut :**
```python
# SQL brut pour un simple CRUD
cursor.execute("SELECT * FROM users WHERE id = %s", [user_id])
row = cursor.fetchone()
user = {'id': row[0], 'name': row[1], 'email': row[2]}
# Code verbeux et peu maintenable
```

**Impact :**
- Code difficile à maintenir
- Performance incohérente
- Équipe divisée sur les pratiques

**Solution :**
Utiliser le bon outil au bon moment

➡️ **Détaillé dans la section 20.3.2**

---

### 3. Lazy Loading vs Eager Loading : Le Piège Invisible 😴

**Qu'est-ce que c'est ?**
Le lazy loading charge les données liées seulement quand on y accède. C'est pratique mais dangereux dans les boucles.

**Exemple du piège :**
```python
# Chargement paresseux (lazy)
articles = Article.objects.all()  # Pas de chargement de l'auteur

# Dans une boucle : CATASTROPHE
for article in articles:
    print(article.author.name)  # Requête SQL à chaque itération !
```

**Impact :**
- N+1 queries garanti
- Mémoire gaspillée avec chargement anticipé excessif
- Code qui fonctionne bien en dev, plante en production

**Solution :**
Stratégies de chargement adaptées (select_related, prefetch_related)

➡️ **Détaillé dans la section 20.3.3**

---

### 4. Opérations Unitaires vs Batching : L'Inefficacité en Série 🐌

**Qu'est-ce que c'est ?**
Insérer, modifier ou supprimer des milliers de lignes une par une au lieu de les traiter en masse.

**Exemple du problème :**
```python
# Importer 100,000 utilisateurs
for row in csv_reader:
    User.objects.create(name=row['name'], email=row['email'])
    # 100,000 INSERT ! Temps : 30-60 minutes !
```

**La bonne approche :**
```python
# Bulk insert
users = [User(name=row['name'], email=row['email']) for row in csv_reader]
User.objects.bulk_create(users, batch_size=10000)
# Temps : 10-20 secondes !
```

**Impact :**
- Import de 1 heure → 1 minute
- Opérations de masse impossibles
- Timeouts et saturation

**Solution :**
Bulk operations et batching

➡️ **Détaillé dans la section 20.3.4**

---

## Comment Détecter la Corruption ?

### Symptômes d'Alerte

**Performance :**
- [ ] Temps de réponse > 1 seconde pour des pages simples
- [ ] Croissance linéaire du temps avec le nombre de lignes
- [ ] CPU ou I/O élevé sur le serveur PostgreSQL

**Logs et Monitoring :**
- [ ] Nombreuses requêtes SQL identiques dans les logs
- [ ] Queries exécutées des milliers de fois par minute
- [ ] Patterns répétitifs dans pg_stat_statements

**Expérience Développeur :**
- [ ] "Ça marchait en dev mais pas en prod"
- [ ] Besoin constant d'augmenter les ressources
- [ ] Workarounds et caches pour masquer la lenteur

### Outils de Détection

**En Développement :**
1. **Django Debug Toolbar** (Python/Django)
   - Affiche toutes les requêtes SQL
   - Détecte les N+1
   - Temps d'exécution par requête

2. **Hibernate Statistics** (Java)
   - Compteur de requêtes
   - Détection de lazy loading

3. **Logging SQL**
   - Activer les logs SQL en développement
   - Compter manuellement les requêtes

**En Production :**
1. **pg_stat_statements** (Extension PostgreSQL)
   - Statistiques de toutes les requêtes
   - Fréquence et temps d'exécution

2. **APM (Application Performance Monitoring)**
   - New Relic, DataDog, Sentry
   - Détection automatique de N+1
   - Alertes sur slow queries

3. **Logs PostgreSQL**
   - log_min_duration_statement
   - auto_explain

---

## Principes de Protection

### 1. Mesurer Avant d'Optimiser

**Ne jamais optimiser à l'aveugle.**

```python
# Toujours profiler d'abord
from django.test.utils import override_settings
from django.db import connection, reset_queries

@override_settings(DEBUG=True)
def test_query_count():
    reset_queries()

    # Votre code ici
    articles = Article.objects.all()
    for article in articles:
        print(article.author.name)

    # Compter les requêtes
    print(f"Nombre de requêtes : {len(connection.queries)}")
```

### 2. Tester avec des Données Réalistes

**Un dataset de 10 lignes ne révèle rien.**

- En développement : minimum 1,000 lignes
- En staging : volume proche de la production
- Tests de charge : simuler le trafic réel

### 3. Automatiser les Tests de Performance

**Les tests unitaires doivent vérifier le nombre de requêtes.**

```python
def test_article_list_no_n_plus_one():
    # Créer 100 articles
    create_test_articles(100)

    # Vérifier qu'il n'y a pas de N+1
    with assert_num_queries(2):  # Max 2 requêtes attendues
        response = client.get('/articles/')
        # Si plus de 2 requêtes → test échoue
```

### 4. Code Review : Checkpoint Critique

**Questions à poser en code review :**

- [ ] Y a-t-il une boucle qui accède à des relations ?
- [ ] Le nombre de requêtes augmente-t-il avec le volume ?
- [ ] Les relations sont-elles chargées avec eager loading ?
- [ ] Les opérations en masse utilisent-elles bulk operations ?

### 5. Documentation des Choix

**Documenter pourquoi vous utilisez une approche.**

```python
def get_user_statistics(user_id):
    """
    Récupère les statistiques utilisateur.

    Utilise SQL brut car :
    - Requête complexe avec 4 CTE
    - L'ORM génère un plan inefficace (450ms vs 80ms en SQL brut)
    - Testé et vérifié avec EXPLAIN ANALYZE

    Dernière révision : 2025-11-23
    """
    with connection.cursor() as cursor:
        cursor.execute(COMPLEX_STATS_QUERY, [user_id])
        return cursor.fetchone()
```

---

## Méthodologie de Correction

### Étape 1 : Identifier

**Utiliser les outils de profiling pour identifier les problèmes.**

1. Activer le logging SQL
2. Exécuter la fonctionnalité lente
3. Analyser les patterns dans les logs
4. Identifier le type de corruption

### Étape 2 : Mesurer

**Quantifier l'impact du problème.**

- Nombre de requêtes actuel
- Temps de réponse actuel
- Charge sur PostgreSQL

### Étape 3 : Corriger

**Appliquer le pattern approprié.**

| Problème Détecté | Solution |
|------------------|----------|
| N+1 queries | JOIN, eager loading |
| Requête ORM inefficace | SQL brut |
| Lazy loading dans boucles | select_related, prefetch_related |
| Import lent | bulk_create, COPY |

### Étape 4 : Vérifier

**Mesurer l'amélioration.**

- Nouveau nombre de requêtes
- Nouveau temps de réponse
- Utilisation des ressources

### Étape 5 : Prévenir

**Ajouter des tests pour éviter la régression.**

```python
@pytest.mark.django_db
def test_article_list_performance():
    """Assure qu'il n'y a pas de régression N+1."""
    create_articles(100)

    with assert_num_queries(2):
        response = client.get('/api/articles/')
        assert response.status_code == 200
```

---

## Architecture Défensive

### Couche d'Abstraction

**Isoler la logique d'accès aux données.**

```
┌─────────────────┐
│   Controllers   │  ← Logique métier
└────────┬────────┘
         │
┌────────▼────────┐
│   Repositories  │  ← Patterns anti-corruption ici !
└────────┬────────┘
         │
┌────────▼────────┐
│   ORM / SQL     │  ← Interaction avec PostgreSQL
└─────────────────┘
```

**Avantages :**
- Centralise les optimisations
- Facilite les tests
- Simplifie la maintenance

### Exemple de Repository

```python
class ArticleRepository:
    """
    Repository pour les articles.
    Encapsule tous les accès à la base de données.
    """

    @staticmethod
    def get_all_with_authors():
        """
        Récupère tous les articles avec leurs auteurs.
        Utilise eager loading pour éviter le N+1.
        """
        return Article.objects.select_related('author').all()

    @staticmethod
    def get_by_category_with_stats(category_id):
        """
        Articles d'une catégorie avec statistiques.
        Requête SQL brute car trop complexe pour l'ORM.
        """
        with connection.cursor() as cursor:
            cursor.execute(ARTICLE_STATS_QUERY, [category_id])
            return cursor.fetchall()

    @staticmethod
    def bulk_import(articles_data):
        """
        Import massif d'articles.
        Utilise bulk_create avec batching.
        """
        articles = [Article(**data) for data in articles_data]
        Article.objects.bulk_create(articles, batch_size=1000)
```

---

## Checklist Générale

Avant de déployer en production :

### Performance
- [ ] Profiling effectué avec données réalistes (> 1,000 lignes)
- [ ] Nombre de requêtes vérifié pour chaque endpoint
- [ ] Pas de N+1 détecté dans les listes
- [ ] Bulk operations pour opérations en masse

### Code Quality
- [ ] Stratégies de chargement explicites (lazy vs eager)
- [ ] SQL brut documenté et justifié
- [ ] Pas de requêtes dans les boucles
- [ ] Repository pattern pour encapsuler l'accès données

### Tests
- [ ] Tests de performance automatisés
- [ ] Tests vérifiant le nombre de requêtes
- [ ] Tests de charge effectués
- [ ] Benchmarks documentés

### Monitoring
- [ ] pg_stat_statements activé
- [ ] Logs SQL en développement
- [ ] APM configuré en production
- [ ] Alertes sur slow queries

---

## Le Coût de l'Ignorance

### Scénario Réel

**Startup SaaS avec 1,000 clients :**

**Avant optimisation (patterns corrompus) :**
- Temps de réponse : 2-5 secondes
- Serveur PostgreSQL : 16 vCPU, 64GB RAM
- Coût infrastructure : $1,500/mois
- Clients qui se plaignent de lenteur

**Après correction (patterns anti-corruption appliqués) :**
- Temps de réponse : 50-200ms (10× plus rapide)
- Serveur PostgreSQL : 4 vCPU, 16GB RAM
- Coût infrastructure : $400/mois
- Clients satisfaits

**Économie : $1,100/mois = $13,200/an**

### Le ROI de la Qualité

**Investissement :**
- 2 semaines de refactoring
- Formation de l'équipe
- Mise en place des outils

**Retour :**
- ✅ 10× amélioration des performances
- ✅ 70% de réduction des coûts serveur
- ✅ Meilleure expérience utilisateur
- ✅ Code plus maintenable
- ✅ Scaling possible sans augmenter les coûts

---

## Vue d'Ensemble des Chapitres Suivants

### 20.3.1. N+1 Queries : Détection et Correction

**Vous apprendrez :**
- Identifier les N+1 dans vos logs
- Utiliser JOIN et LATERAL pour corriger
- Eager loading avec les ORM
- Tests automatisés anti-N+1

**Temps de lecture : 30 minutes**

### 20.3.2. ORM vs SQL Brut : Quand Utiliser Quoi

**Vous apprendrez :**
- Avantages et limites des ORM
- Quand passer au SQL brut
- L'approche hybride recommandée
- Arbre de décision pratique

**Temps de lecture : 25 minutes**

### 20.3.3. Lazy Loading vs Eager Loading

**Vous apprendrez :**
- Différence entre lazy et eager loading
- select_related vs prefetch_related
- Stratégies par type de relation
- Éviter les sur-optimisations

**Temps de lecture : 25 minutes**

### 20.3.4. Batching et Bulk Operations

**Vous apprendrez :**
- bulk_create, bulk_update, bulk_delete
- La commande COPY (50,000 lignes/s)
- Batching pour gros volumes
- Gestion des erreurs et monitoring

**Temps de lecture : 30 minutes**

---

## Conclusion de l'Introduction

Les **patterns anti-corruption** ne sont pas des optimisations prématurées. Ce sont des **pratiques de base** pour écrire du code qui interagit sainement avec PostgreSQL.

**Les erreurs sont faciles à faire :**
- Un `for` loop avec lazy loading → N+1
- Un ORM forcé sur une requête complexe → Lenteur
- Un import ligne par ligne → 100× plus lent

**Mais les corrections sont simples :**
- Ajouter `select_related()` → Problème résolu
- Passer en SQL brut pour les cas complexes → Performance optimale
- Utiliser `bulk_create()` → 100× plus rapide

**La vraie compétence :** Savoir identifier et corriger ces patterns **avant** qu'ils ne deviennent des problèmes en production.

---

**Prêt à plonger dans les détails ?**

➡️ **Commencez par : 20.3.1. N+1 Queries : Détection et Correction**

---

⏭️ [N+1 queries : Détection et correction (JOIN, LATERAL)](/20-drivers-connexion-applicative/03.1-n-plus-1-queries.md)
