# Points de saturation et techniques de correction

## Objectif

Ce document liste les principaux points de saturation probables de la plateforme FastRelays, ainsi que les techniques concrètes pour les corriger dans un contexte microservices, frontend Vue SPA, déploiement initial mono-instance par service, et usage principal au Cameroun.

## Vue d'ensemble

| Zone | Niveau de risque | Impact principal |
|---|---|---|
| `gateway` | élevé | congestion globale et propagation des latences |
| `products` / `business` | élevé | lenteur catalogue et découverte |
| `order` | très élevé | baisse de débit sur parcours panier/commande |
| `payment` | critique | latence externe et indisponibilité partielle |
| `images` | élevé | lenteur mobile, bande passante, I/O |
| PostgreSQL | élevé | files d'attente, hausse du P95, contention |
| communication interservices | élevé | timeouts en cascade |
| frontend mobile | élevé | perception de lenteur même si backend correct |

## 1. Saturation du gateway

### Symptômes

- augmentation du temps de réponse sur tous les endpoints
- hausse des erreurs `502`, `504`, `timeout`
- backlog de requêtes aux heures de pointe
- effet domino quand un microservice lent bloque le trafic entrant

### Causes probables

- une seule instance de gateway
- trop d'appels SPA pour construire un écran
- absence de limitation de débit
- timeouts trop longs vers les services aval

### Techniques de correction

- déployer **au moins 2 instances** du gateway dès la première montée de charge
- placer un **load balancer / ingress** devant le gateway
- configurer des **timeouts courts et explicites**
- activer des **circuit breakers** pour éviter les cascades d'échec
- mettre du **rate limiting** par IP, session ou token
- ajouter des endpoints backend agrégés pour réduire le nombre d'appels frontend

## 2. Saturation des services catalogue `products` et `business`

### Symptômes

- pages publiques lentes
- temps de réponse très variable sur `/public/article`, `/public/business`, `/public/promotion`
- montée rapide du P95 quand plusieurs utilisateurs parcourent le catalogue

### Causes probables

- requêtes paginées coûteuses
- filtres et tris non indexés
- lectures répétées des mêmes données publiques
- agrégats calculés à la volée

### Techniques de correction

- ajouter un **cache Redis** pour:
  - détails business
  - promotions actives
  - catégories
  - compteurs et stats légères
- imposer une **pagination stricte**
- limiter la taille des pages à **10 ou 20 éléments** par défaut
- créer des **indexes SQL** sur:
  - `businessId`
  - `createdAt`
  - `status`
  - colonnes de tri fréquentes
- éviter les payloads trop riches sur les listes
- créer des projections DTO spécifiques liste/détail
- ajouter des endpoints optimisés pour mobile

## 3. Saturation du service `order`

### Symptômes

- lenteur sur panier, création de commande, changements de statut
- forte dégradation sous pics d'écritures
- blocage de transactions

### Causes probables

- logique métier transactionnelle lourde
- vérifications synchrones interservices
- écritures multiples en base
- dépendance au paiement ou à d'autres services

### Techniques de correction

- découpler le workflow commande en étapes:
  - création commande
  - réservation/validation
  - paiement
  - confirmation
- publier des **événements Kafka** pour les traitements secondaires
- utiliser des **idempotency keys** sur les opérations critiques
- réduire les transactions longues
- mettre des indexes sur:
  - `orderId`
  - `businessId`
  - `shopId`
  - `status`
  - `createdAt`
- séparer les endpoints très sollicités de lecture et d'écriture
- ajouter des vues ou tables matérialisées si les listes deviennent lourdes

## 4. Saturation du service `payment`

### Symptômes

- paiement lent ou instable
- temps de réponse fortement dépendant du prestataire externe
- commandes bloquées par latence Monetbil

### Causes probables

- appel HTTP externe dans le flux synchrone utilisateur
- indisponibilité ou lenteur du prestataire
- retries mal contrôlés

### Techniques de correction

- rendre le paiement **asynchrone autant que possible**
- transformer le flux en:
  - création requête de paiement
  - retour immédiat d'un statut `pending`
  - mise à jour ultérieure par webhook / événement
- mettre des **timeouts stricts**
- ajouter **circuit breaker**, **retry borné**, **bulkhead**
- journaliser les paiements avec corrélation métier
- prévoir un mode dégradé: commande créée mais paiement en attente
- séparer les workers de traitement paiement si le trafic augmente

## 5. Saturation du service `images`

### Symptômes

- upload lent
- erreurs réseau fréquentes sur mobile
- forte consommation bande passante
- expérience très dégradée sur pages riches en images

### Causes probables

- upload qui traverse entièrement le backend
- fichiers trop lourds
- absence de variantes mobile
- absence de cache edge

### Techniques de correction

- compresser les images côté client avant upload
- imposer des tailles maximales pratiques par usage
- générer plusieurs formats:
  - miniature
  - mobile
  - desktop
- servir les images via **CDN** ou cache HTTP agressif
- utiliser `webp` / `avif` quand possible
- utiliser des URLs signées ou versionnées si nécessaire
- envisager l'**upload direct vers stockage objet** via URL pré-signée

## 6. Saturation PostgreSQL

### Symptômes

- hausse du temps SQL moyen
- connexions en attente
- locks fréquents
- lenteur généralisée sur plusieurs services

### Causes probables

- pools de connexions faibles ou mal réglés
- absence d'indexes
- requêtes N+1
- tables à forte croissance non optimisées

### Techniques de correction

- augmenter progressivement les **pools Hikari** selon service
- analyser les **slow queries**
- créer les indexes manquants
- vérifier les plans d'exécution
- réduire les N+1 ORM
- utiliser des requêtes paginées stables
- ajouter un **read replica** plus tard pour les fortes lectures publiques
- séparer si besoin certaines bases par domaine critique à moyen terme

## 7. Saturation due à la communication inter-microservices

### Symptômes

- un écran SPA déclenche beaucoup de requêtes
- les latences s'additionnent
- un service dégradé ralentit plusieurs autres

### Causes probables

- trop de dépendances synchrones
- orchestration distribuée dans le flux HTTP utilisateur
- réponses fragmentées entre plusieurs domaines

### Techniques de correction

- réduire le nombre d'appels synchrones dans le parcours utilisateur
- introduire des **endpoints d'agrégation**
- déplacer les traitements non bloquants vers Kafka
- ajouter des **caches locaux** et **caches Redis**
- versionner et simplifier les DTO renvoyés au frontend
- utiliser des **timeouts différents par type d'appel**

## 8. Saturation réseau côté Cameroun

### Symptômes

- application ressentie comme lente alors que le backend reste acceptable
- temps de chargement élevés sur mobile
- abandons pendant upload ou paiement

### Causes probables

- latence mobile fluctuante
- payloads trop volumineux
- trop de requêtes séquentielles
- images non optimisées

### Techniques de correction

- réduire drastiquement le poids des réponses
- activer compression HTTP
- mettre en cache agressivement les contenus publics
- éviter les appels en cascade au chargement initial
- précharger uniquement les données essentielles
- utiliser lazy loading et infinite scroll maîtrisé
- mettre en place des retries frontend limités avec backoff

## 9. Saturation mémoire JVM

### Symptômes

- ralentissements progressifs
- Full GC
- redémarrages de conteneurs

### Causes probables

- heaps de 512 Mo sur services à trafic variable
- gros payloads en mémoire
- uploads ou listes volumineuses
- trop de logs ou de traces

### Techniques de correction

- surveiller `heap used`, `GC pause`, `allocation rate`
- augmenter la heap uniquement après mesure
- réduire la taille des payloads
- éviter les listes non bornées
- limiter la verbosité des logs
- séparer les flux lourds comme images ou batchs

## 10. Saturation liée au frontend Vue SPA

### Symptômes

- premier chargement lent
- nombreux appels API pour un seul écran
- mauvaise expérience sur mobile

### Causes probables

- stratégie de chargement trop bavarde
- manque de cache local
- requêtes déclenchées trop tôt ou trop souvent

### Techniques de correction

- regrouper les appels par vue
- utiliser cache local mémoire/session
- différer les données non critiques
- mettre en place pagination, debounce, batching
- limiter les refresh automatiques
- compresser et découper les bundles frontend

## Priorisation recommandée

### Priorité 1

- cache Redis sur données publiques fréquentes
- pagination stricte
- optimisation images
- timeouts + circuit breakers
- désactivation logs DEBUG en production
- instrumentation Prometheus/Grafana

### Priorité 2

- 2 instances pour `gateway`, `products`, `business`, `order`
- découplage asynchrone `order` / `payment`
- endpoints backend agrégés pour la SPA
- tuning pools DB et indexes

### Priorité 3

- autoscaling Kubernetes
- read replica PostgreSQL
- upload direct objet
- séparation plus nette des charges critiques

## Mapping saturation -> correction rapide

| Saturation | Correction rapide | Correction structurante |
|---|---|---|
| gateway saturé | timeout, rate limit | 2+ instances + LB |
| catalogue lent | cache Redis + pagination | scaling `products`/`business` |
| commandes lentes | indexes + raccourcir transactions | architecture événementielle |
| paiement lent | timeout + mode pending | flux asynchrone complet |
| images lentes | compression + cache HTTP | CDN + upload direct objet |
| DB saturée | indexes + slow query tuning | read replica / séparation charges |
| latence mobile | payloads plus petits | stratégie mobile-first complète |

## Conclusion

Les premières saturations les plus probables sont:

1. `products` et `business` sur les lectures catalogue
2. `order` sur les écritures métier
3. `payment` à cause des appels externes
4. `images` à cause du réseau mobile et de la bande passante
5. PostgreSQL si les indexes et pools ne sont pas ajustés

Les corrections les plus rentables à court terme sont:

1. cache Redis
2. pagination stricte
3. optimisation images
4. timeouts, retries bornés, circuit breakers
5. découplage asynchrone des flux `order` et `payment`
