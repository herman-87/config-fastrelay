# Audit de performance FastRelays

## Résumé exécutif

L'analyse des fichiers `openapi/main.yaml` des microservices montre une plateforme orientée lecture, avec une part importante de parcours métier synchrones sur `business`, `products`, `order`, `user`, `payment`, `images`, `messages`, `feedback`, `likes`, `subscription` et `stats`. Les endpoints les plus sensibles pour la performance sont les lectures paginées de catalogue, les créations et transitions de commande, les appels de paiement externes, ainsi que les uploads d'images.

Avec l'infrastructure initiale fournie, en supposant une seule instance par microservice, PostgreSQL comme base principale, Kafka déjà présent pour certains flux, et sans cache distribué, la plateforme peut supporter de façon réaliste environ **120 à 220 utilisateurs simultanés actifs**, soit **35 à 70 requêtes HTTP/s côté gateway** selon le mix fonctionnel. Le point de saturation ne sera probablement pas la RAM globale de 48 Go, mais plutôt les chaînes synchrones `gateway -> service -> base -> service tiers`, la taille réduite des pools DB observés, l'absence de cache partagé, et la variabilité du réseau mobile au Cameroun.

À court terme, le système est viable pour un lancement limité ou une montée progressive si les pages publiques sont paginées strictement, si les images sont servies via CDN/cache objet, et si les opérations lentes de paiement/notification sont découplées. À moyen terme, il faut viser un front door robuste (`gateway + cache + observabilité`), un découplage plus poussé par messaging, et un scaling horizontal ciblé sur `products`, `business`, `order`, `payment` et `images`.

## Périmètre analysé

### Fichiers étudiés

- `user/openapi/main.yaml`
- `products/openapi/main.yaml`
- `order/openapi/main.yaml`
- `payment/openapi/main.yaml`
- `businesszone/openapi/main.yaml`
- `subscription/openapi/main.yaml`
- `images/openapi/main.yaml`
- `messages/openapi/main.yaml`
- `feedback/openapi/main.yaml`
- `likes/openapi/main.yaml`
- `stats/openapi/main.yaml`
- configurations `application.yaml` et `build.gradle` de plusieurs services

### Synthèse API

Le périmètre inspecté représente **environ 98 opérations HTTP** hors `template_v1`, réparties comme suit :

| Domaine | Exemples d'endpoints | Sensibilité perf |
|---|---|---|
| Auth / utilisateur | `/public/user/register`, `/user/me`, `/user/{id}` | moyenne |
| Catalogue / business | `/public/business`, `/public/article`, `/public/promotion`, `/public/business/{id}` | forte |
| Commande | `/my/cart`, `/businesses/{businessId}/order`, `/my/order`, transitions de statut | très forte |
| Paiement | `/transfer-request/monetbil`, `/transfer-request/internal` | critique |
| Images | `/my/images`, `/public/images/{imageId}` | forte sur réseau et I/O |
| Messagerie | `/conversations/{conversationId}/messages` | moyenne à forte |
| Engagement | likes, feedback, ratings | moyenne |
| Abonnement / stats | plans, souscriptions, statistiques agrégées | moyenne |

## Hypothèses techniques

Les estimations ci-dessous sont volontairement réalistes et prudentes, car les implémentations internes exactes, les indexes SQL, la topologie réseau finale et les volumes de données ne sont pas entièrement visibles dans les fichiers inspectés.

### Hypothèses d'architecture

- Base principale supposée: **PostgreSQL** pour les services métier.
- Stockage image supposé: **MinIO / stockage objet compatible S3**.
- Messaging déjà présent: **Kafka** sur `user`, `order`, `payment`, `business`, `stats`, `messages`.
- Authentification supposée: **Keycloak / JWT**.
- Appels interservices: principalement **HTTP synchrone** via gateway / OpenFeign, avec quelques événements Kafka.
- Déploiement initial: **1 instance par microservice**, pas d'autoscaling, pas de réplication DB, pas de read replicas.
- JVM observée dans plusieurs images: **heap ~512 Mo par service**.

### Hypothèses de charge utilisateur

- Utilisateur actif SPA: **1,2 à 2,0 requêtes/s** pendant une session interactive.
- Utilisateur moyen connecté: **1 requête toutes les 10 à 20 secondes**.
- Pic réaliste mobile: rafraîchissements, retries réseau et navigation fragmentée augmentent la charge de **20 à 35 %**.
- Mix de charge au lancement:
  - **60 % lecture**
  - **20 % navigation authentifiée / consultation privée**
  - **12 % écritures métier**
  - **5 % uploads images**
  - **3 % paiement**

### Hypothèses de payload

| Type | Taille moyenne requête | Taille moyenne réponse | Hypothèse |
|---|---:|---:|---|
| Auth / user | 0,8 à 1,5 Ko | 1 à 3 Ko | DTO simples |
| Lectures détail | 0,3 à 0,8 Ko | 2 à 8 Ko | entité unique |
| Lectures paginées catalogue | 0,5 à 1 Ko | 15 à 80 Ko | page 10 à 20 items |
| Écriture commande | 1,5 à 4 Ko | 1 à 3 Ko | panier / order lines |
| Paiement | 1 à 2 Ko | 1 à 4 Ko | redirection ou statut |
| Messages | 0,5 à 2 Ko | 1 à 15 Ko | conversation courte |
| Upload image | 200 Ko à 2,5 Mo moyen | 0,5 à 1 Ko | mobile compressé, max 10 Mo |
| Ratings / likes / stats | < 0,5 Ko | 0,3 à 2 Ko | opérations légères |

### Hypothèses réseau Cameroun

Les latences ci-dessous sont des ordres de grandeur pour un frontend SPA utilisé principalement sur réseaux mobiles 4G/3G et Wi-Fi urbains:

- RTT frontend -> backend au Cameroun:
  - **4G / bon Wi-Fi**: 60 à 120 ms
  - **réseau mobile moyen**: 120 à 220 ms
  - **cas dégradé**: 250 à 500 ms
- Jitter important sur mobile: **20 à 80 ms**
- Retransmissions / perte de paquets perceptibles sur upload et longues listes

## Analyse des performances

### Classification des endpoints

| Famille | Services dominants | Nature |
|---|---|---|
| Lecture simple | `user`, `stats`, `likes`, `feedback`, `images` | 1 à 3 lectures DB |
| Lecture paginée / recherche | `products`, `businesszone`, `order`, `feedback`, `likes` | DB + tri + pagination + parfois agrégats |
| Écriture simple | likes, feedback, update user, accept/reject invitation | 1 transaction DB |
| Écriture métier complexe | create/update article, create order, transitions commande | plusieurs vérifications et appels interservices |
| Paiement externe | `payment` | DB + appel tiers + webhook |
| Upload / média | `images` | réseau + stockage objet + métadonnées |

### Estimation de temps de réponse par type d'endpoint

Les chiffres ci-dessous distinguent:

- **latence backend**: traitement serveur + DB + appels internes
- **latence perçue Cameroun**: backend + réseau SPA

| Type d'endpoint | Exemples | Temps backend moyen | P95 backend | Latence perçue Cameroun moyenne | Throughput réaliste par instance |
|---|---|---:|---:|---:|---:|
| Lecture simple | `/user/me`, `/public/images/{id}`, `/public/business/stats` | 25 à 60 ms | 80 à 140 ms | 120 à 260 ms | 80 à 180 req/s |
| Lecture paginée légère | `/my/likes`, `/feedback/entity/{id}` | 40 à 90 ms | 120 à 220 ms | 150 à 320 ms | 50 à 110 req/s |
| Lecture catalogue / recherche | `/public/business`, `/public/article`, `/public/promotion` | 70 à 160 ms | 180 à 350 ms | 190 à 420 ms | 20 à 60 req/s |
| Écriture simple | like/unlike, feedback, update user | 45 à 100 ms | 120 à 220 ms | 160 à 330 ms | 30 à 80 req/s |
| Écriture métier complexe | create article, create order, change status order | 90 à 220 ms | 250 à 500 ms | 220 à 560 ms | 12 à 35 req/s |
| Paiement interne | `/transfer-request/internal` | 120 à 260 ms | 300 à 650 ms | 250 à 700 ms | 8 à 20 req/s |
| Paiement Monetbil | `/transfer-request/monetbil` | 300 à 900 ms | 1,2 à 2,5 s | 450 ms à 2,7 s | 2 à 8 req/s |
| Messages | lecture/écriture conversation | 35 à 90 ms | 100 à 220 ms | 140 à 320 ms | 35 à 90 req/s |
| Upload image | `/my/images` | 250 ms à 1,5 s | 2 à 5 s | 700 ms à 6 s | 1 à 6 req/s |

### Estimation par service critique

| Service | Profil | Capacité réaliste 1 instance | Contrainte principale |
|---|---|---:|---|
| `gateway` | point d'entrée | 120 à 250 req/s | backpressure et timeouts |
| `user` | lecture/auth simple | 60 à 120 req/s | pool DB `maximum-pool-size: 5` |
| `business` | listes, détails, collaborations | 25 à 60 req/s | lecture catalogue + joins |
| `products` | catalogue, promotions, filtres | 20 à 55 req/s | pagination et recherche |
| `order` | panier + création + workflow | 12 à 30 req/s | transactions, DB, appels interservices |
| `payment` | appels tiers | 2 à 15 req/s selon mix | latence Monetbil / résilience |
| `images` | upload + metadata | 5 à 25 req/s metadata, 1 à 6 upload/s | réseau et stockage objet |
| `messages` | conversation | 30 à 70 req/s | polling si pas websocket |
| `likes` / `feedback` | engagement | 40 à 100 req/s | contention DB sous pics |
| `subscription` | faible fréquence | 10 à 40 req/s | dépendances métier |
| `stats` | agrégats | 20 à 80 req/s | requêtes agrégées |

### Estimation globale de capacité sans scaling horizontal

Le nombre d'utilisateurs simultanés dépend surtout du mix fonctionnel.

| Scénario | Description | Req/s globales | Utilisateurs simultanés supportés |
|---|---|---:|---:|
| Conservateur | beaucoup de mobile moyen, peu de cache, catalogue consulté fréquemment | 35 à 45 | 120 à 160 |
| Réaliste lancement | mix lecture/écriture normal, pagination correcte | 45 à 60 | 160 à 220 |
| Optimisé sans scaling | cache HTTP, compression images, requêtes front réduites | 60 à 80 | 220 à 320 |

### Interprétation

- Le système **ne sera pas limité par les 16 CPU ou les 48 Go RAM au départ** si chaque service reste autour de 256 à 512 Mo de heap.
- La **capacité métier réelle** sera plafonnée par les microservices les plus lents: `payment`, `order`, `products`, `business`, `images`.
- Une seule instance par service implique qu'un service lent ou dégradé peut faire chuter la capacité globale de toute la SPA.

## Bottlenecks potentiels

### CPU

Risque moyen au lancement, fort sous croissance.

Points sensibles:

- sérialisation JSON sur grosses listes de catalogue
- traitement image et validation mime/metadata
- chiffrement/auth JWT/Keycloak sur fort trafic
- agrégats métier si non matérialisés

Signal d'alerte:

- CPU > 70 % soutenu sur `products`, `business`, `order`, `images`
- hausse du temps GC ou threads HTTP bloqués

### RAM

Risque faible à moyen sur l'hôte, mais moyen par conteneur.

Observations:

- plusieurs services sont configurés à **512 Mo max heap**
- cela suffit pour du trafic modéré, mais reste serré pour:
  - pages catalogue volumineuses
  - buffers upload
  - pics de concurrence + traces + logs DEBUG

Signal d'alerte:

- OOM kill
- Full GC répétés
- augmentation de `jvm.memory.used` > 80 % de la heap

### I/O disque

Risque surtout pour:

- PostgreSQL sous écritures `order`, `payment`, `feedback`, `likes`
- journaux applicatifs trop bavards
- batchs Spring Batch côté `payment`

Signal d'alerte:

- latence fsync DB
- saturation IOPS lors de campagnes ou pics d'upload

### Réseau

C'est un bottleneck majeur dans le contexte Cameroun.

Points sensibles:

- taille des réponses paginées
- téléchargement / upload d'images
- multiplicité des appels SPA pour construire un écran
- paiements tiers hors réseau local

Signal d'alerte:

- TTFB correct côté backend mais temps perçu mauvais côté mobile
- taux de retry front et abandon utilisateur en hausse

### Communication inter-microservices

Risque élevé si les écrans exigent plusieurs appels chaînés.

Exemples probables:

- commande: `order` + `products` + `business` + `payment`
- catalogue enrichi: `products` + `feedback` + `likes` + `images`
- collaboration/business: `business` + `user` + éventuellement `subscription`

Conséquence:

- amplification de la latence
- propagation des erreurs
- timeouts en cascade

## Points de saturation

| Point de saturation | Cause probable | Impact |
|---|---|---|
| Pool DB trop faible sur `user` | `maximum-pool-size: 5` observé | files d'attente, montée rapide du P95 |
| Lecture catalogue non cachée | beaucoup de GET publics | saturation `products` / `business` |
| Création de commande synchrone | validations interservices + DB + paiement | baisse forte de débit |
| Upload direct via backend | réseau mobile + corps multipart | threads occupés et latence élevée |
| Paiement externe | dépendance Monetbil / API tierce | indisponibilité partielle ou lenteur globale |
| Logs DEBUG actifs | vu dans `order` | surcoût CPU/I/O et bruit observabilité |

## Recommandations techniques

### 1. Quick wins avant mise en production

- Activer une **pagination stricte** sur toutes les listes publiques et privées.
- Limiter par défaut les pages à **10 ou 20 éléments** sur mobile.
- Ajouter des **indexes SQL** sur:
  - `businessId`
  - `articleId`
  - `userId` / `username`
  - `createdAt`
  - statuts de commande et de paiement
- Réduire ou désactiver les **logs DEBUG** en production, surtout sur `order`.
- Mettre des **timeouts** explicites sur tous les appels interservices et externes.
- Ajouter des **retries bornés** uniquement sur les GET idempotents et paiements asynchrones contrôlés.

### 2. Caching

#### Redis

À recommander en priorité pour:

- détails de business publics
- listes de promotions actives
- catégories d'articles
- compteurs `likes`, `ratings`, `collaborator/count`
- stats publiques

TTL suggérés:

| Donnée | TTL suggéré |
|---|---:|
| business public / article détail | 1 à 5 min |
| promotions actives | 30 à 60 s |
| stats publiques | 30 à 120 s |
| compteurs likes/ratings | 15 à 60 s |

#### CDN / cache objet

- Servir les images via **CDN** ou au minimum via cache HTTP agressif.
- Utiliser des URLs versionnées et `Cache-Control`.
- Générer plusieurs tailles d'image pour mobile.

### 3. Optimisation des endpoints

- Éviter les écrans SPA qui nécessitent plus de **4 à 6 appels** pour un rendu initial.
- Prévoir des endpoints agrégés dédiés aux vues front si nécessaire.
- Remplacer certains enchaînements `GET détail + GET likes + GET ratings + GET images` par des payloads consolidés.
- Pour `/public/article` et `/public/business`, supporter:
  - champs filtrés
  - tri borné
  - pagination curseur si croissance forte

### 4. Messaging et découplage

À pousser au-delà de l'existant Kafka:

- `order created` -> publication événement -> traitements secondaires asynchrones
- paiement -> confirmation par webhook + consommateur événementiel
- recalcul des stats / ratings / compteurs -> asynchrone
- notifications et invitations -> asynchrone

Recommandation:

- **Kafka** si vous attendez un volume croissant et besoin de replay
- **RabbitMQ** si vous privilégiez la simplicité pour des workflows transactionnels modestes

Dans ce contexte, le plus cohérent avec l'existant est de **standardiser Kafka**.

### 5. Load balancing et scaling

- Placer un **reverse proxy / ingress** devant la gateway.
- Éviter la single point of failure de la gateway dès que possible.
- Passer rapidement à **2 instances** pour `gateway`, `products`, `business`, `order`.
- Garder `payment` et `images` isolés avec autoscaling spécifique.

### 6. Base de données

- PostgreSQL reste un bon choix initial.
- Prévoir rapidement:
  - tuning pool de connexions
  - slow query log
  - indexes composites
  - read replicas plus tard pour lectures catalogue
- Pour `messages`, si le volume conversationnel croît fortement, envisager à moyen terme une base plus adaptée aux lectures append-only ou un stockage spécialisé, mais **pas nécessaire au lancement**.

## Capacité d'évolution

### Capacité après scaling raisonnable

Scénario recommandé à moyen terme:

- `gateway`: 2 instances
- `products`: 3 instances
- `business`: 3 instances
- `order`: 2 à 3 instances
- `payment`: 2 instances
- `images`: 2 instances
- Redis partagé
- PostgreSQL mieux dimensionné, éventuellement read replica

| État cible | Req/s globales réalistes | Utilisateurs simultanés réalistes |
|---|---:|---:|
| Initial 1 instance/service | 35 à 70 | 120 à 220 |
| Optimisé sans scale massif | 60 à 80 | 220 à 320 |
| Scale modéré + cache | 120 à 200 | 450 à 900 |
| Scale ciblé + découplage avancé | 200 à 350 | 900 à 1 800 |

### Architecture recommandée à moyen terme

- `gateway` hautement disponible
- `Redis` pour cache partagé et éventuellement rate limiting
- `Kafka` pour workflows métier critiques non bloquants
- `PostgreSQL` principal + read replica pour lectures publiques si besoin
- `MinIO/S3 + CDN` pour images
- observabilité centralisée `Prometheus + Grafana + Loki/ELK + tracing`

### Services à découpler en priorité

1. `payment` du flux synchrone de commande
2. `stats` et agrégats publics des lectures transactionnelles
3. compteurs `likes` / `ratings` des lectures de catalogue
4. notifications / invitations des transactions utilisateur

## Recommandations adaptées au contexte Cameroun

### Gestion de la latence

- privilégier des réponses compactes
- compresser JSON et images
- éviter les cascades d'appels front
- précharger uniquement l'essentiel au premier écran
- utiliser `stale-while-revalidate` sur données publiques peu sensibles

### Optimisation mobile

- images en `webp`/`avif` quand possible
- miniatures dédiées mobile
- pagination courte
- lazy loading systématique
- reprise d'upload si les fichiers deviennent fréquents et volumineux

### Résilience réseau

- timeouts front explicites
- retries limités avec backoff
- file d'attente locale côté SPA pour certaines actions non critiques
- idempotency keys sur paiements et créations sensibles
- mode dégradé si `payment` indisponible: commande créée en attente de paiement plutôt qu'échec global

## Plan d'évolution

### Phase 1: avant lancement

- corriger indexes et slow queries
- désactiver logs DEBUG
- mettre cache HTTP sur images et contenus publics
- ajouter métriques, traces et dashboards
- valider les timeouts et circuit breakers
- tester la charge jusqu'à **60 req/s** globales

### Phase 2: après premiers utilisateurs

- ajouter Redis
- passer `gateway`, `products`, `business`, `order` à 2+ replicas
- rendre les paiements et notifications davantage asynchrones
- optimiser les endpoints agrégés pour le frontend Vue SPA

### Phase 3: croissance

- autoscaling Kubernetes sur services les plus exposés
- read replica PostgreSQL pour lectures publiques
- CDN edge pour images
- isolation éventuelle des workloads `messages` et `analytics`

## Bonus: scénario de test de charge

### Objectifs

- valider le comportement jusqu'au seuil de 120, puis 220 utilisateurs simultanés
- mesurer la dégradation des P95/P99
- identifier le premier service saturé

### Scénario k6 recommandé

Parcours à simuler:

1. visite catalogue public
2. consultation business et article
3. authentification / récupération profil
4. ajout panier
5. création commande
6. paiement simulé ou stub externe
7. consultation historique
8. upload image sur un sous-ensemble d'utilisateurs

Répartition suggérée:

| Flux | Part du trafic |
|---|---:|
| consultation catalogue / business | 45 % |
| consultation promotions / stats | 15 % |
| likes / feedback / interactions légères | 10 % |
| panier / commande | 15 % |
| paiement | 5 % |
| messagerie | 5 % |
| upload image | 5 % |

Exécution suggérée:

- palier 1: 50 VUs pendant 10 min
- palier 2: 100 VUs pendant 15 min
- palier 3: 200 VUs pendant 15 min
- spike: 300 VUs pendant 3 min
- soak test: 80 VUs pendant 1 h

Critères d'acceptation initiaux:

- P95 GET publics < **450 ms** perçu backend+réseau simulé hors images
- P95 écritures métier < **700 ms**
- taux d'erreur < **1 %**
- aucun épuisement pool DB
- aucune dérive mémoire significative

## Bonus: métriques à monitorer

### Infrastructure

- CPU par pod / conteneur
- mémoire RSS et heap JVM
- I/O disque
- bande passante réseau
- saturation sockets / connexions

### JVM / Spring

- `http.server.requests`
- P50/P95/P99 par endpoint
- nombre de threads occupés
- GC pause time
- taux d'erreur 4xx/5xx
- timeouts et retries

### Base de données

- temps moyen et P95 des requêtes SQL
- connexions actives / pool wait time
- locks
- cache hit ratio
- slow queries

### Kafka / messaging

- consumer lag
- temps de traitement des événements
- retries / DLQ
- débit par topic

### Métier

- commandes créées / minute
- paiements réussis / échoués / timeout
- uploads images réussis / abandonnés
- temps de rendu des pages catalogue côté frontend

## Conclusion

Le système est cohérent pour un démarrage, mais **pas encore dimensionné pour une montée rapide sans cache, sans décorrélation supplémentaire et sans optimisation front/back conjointe**. Avec l'état actuel des contrats API et l'infrastructure cible, une enveloppe réaliste est de **120 à 220 utilisateurs simultanés actifs**. En appliquant les optimisations proposées puis un scaling horizontal ciblé, une capacité de **450 à 900 utilisateurs simultanés** devient réaliste à moyen terme, avec une trajectoire possible vers **900 à 1 800** sous réserve d'une meilleure distribution de charge, d'un cache partagé et d'une architecture plus asynchrone sur `order`, `payment` et `catalogue`.
