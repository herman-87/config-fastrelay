# FastRelays - Architecture Microservices

> Plateforme e-commerce/marketplace basée sur une architecture microservices Spring Boot.

## Quick Start

```bash
# Démarrage rapide avec Docker Compose
docker-compose up -d

# Vérification des services
curl http://localhost:8060/actuator/health
```

---

## Table des Matières

1. [Présentation globale](#1-présentation-globale)
2. [Architecture](#2-architecture)
3. [Liste des Microservices](#3-liste-des-microservices)
4. [Stack technique](#4-stack-technique)
5. [Prérequis](#5-prérequis)
6. [Installation & Démarrage](#6-installation--démarrage)
7. [Configuration](#7-configuration)
8. [Base de données](#8-base-de-données)
9. [API & Endpoints](#9-api--endpoints)
10. [Communication inter-services](#10-communication-inter-services)
11. [Tests](#11-tests)
12. [Build & Déploiement](#12-build--déploiement)
13. [Monitoring & Logs](#13-monitoring--logs)
14. [Bonnes pratiques](#14-bonnes-pratiques--recommandations)
15. [Limitations connues](#15-limitations-connues)
16. [Contribution](#16-contribution)
17. [Licence](#17-licence)

---

## 1. Présentation globale

### Description du système

**FastRelays** est une plateforme e-commerce/marketplace dibangun sur une architecture microservices permettant la gestion complète d'un marché numérique incluant :

- Gestion des utilisateurs et authentification
- Gestion des produits et articles
- Gestion des commandes
- Paiements intégrés (Monetbil, Pawapay)
- Abonnements et subscriptions
- Messagerie entre utilisateurs
- Système de likes et feedback
- Gestion des images (MinIO)
- Statistiques en temps réel

### Objectifs métier

- Fournir une plateforme d'e-commerce scalable
- Gérer les paiements mobiles via Monetbil/Pawapay
- Permettre la création de boutiques (business)
- Gérer les abonnements premium
- Analyser les données avec des statistiques

---

## 2. Architecture

### Vue d'ensemble

```
                                    ┌─────────────────┐
                                    │   API Gateway   │
                                    │  (Port : 8060)  │
                                    └────────┬────────┘
                                             │
           ┌──────────────┬─────────────────┼─────────────────┬──────────────┐
           │              │                 │                 │              │
    ┌──────▼──────┐ ┌─────▼─────┐ ┌───────▼──────┐ ┌──────▼──────┐ ┌────▼─────┐
    │   User      │ │  Product  │ │   Business   │ │   Order     │ │  Payment  │
    │  (8083)     │ │           │ │              │ │             │ │          │
    └──────┬──────┘ └─────┬─────┘ └───────┬─────┘ └──────┬──────┘ └────┬─────┘
           │              │               │              │             │
           └──────────────┴──────────────┴──────────────┴─────────────┘
                                             │
                              ┌──────────────┴──────────────┐
                              │                             │
                       ┌──────▼──────┐              ┌───────▼───────┐
                       │   Kafka     │              │ PostgreSQL    │
                       │ (29092)     │              │               │
                       └─────────────┘              └───────────────┘
```

### Composants principaux

| Composant | Rôle | Port |
|-----------|------|------|
| **Discovery** | Service Registry (Eureka) | 8761 |
| **Gateway** | Point d'entrée API | 8060 |
| **Config Server** | Gestion centralisée de la configuration | 8088 |
| **Kafka** | Messaging asynchrone | 29092 |
| **PostgreSQL** | Base de données relationnelle | 5432 |
| **MinIO** | Stockage d'images | 9000 |

### Flux de communication

1. **Requête cliente** → Gateway (8060) → Service cible
2. **Communication synchrone** : REST via Gateway
3. **Communication asynchrone** : Kafka pour les événements
4. **Discovery** : Enregistrement automatique des services

---

## 3. Liste des Microservices

| # | Service | Description | Port | Stack |
|---|---------|-------------|------|-------|
| 1 | **discovery** | Service Discovery Eureka | 8761 | Spring Boot + Eureka Server |
| 2 | **config-server** | Configuration centralisée | 8088 | Spring Cloud Config |
| 3 | **gateway** | API Gateway / Reverse Proxy | 8060 | Spring Cloud Gateway |
| 4 | **user** | Gestion des utilisateurs | 8083 | Spring Boot + PostgreSQL |
| 5 | **business** | Gestion des entreprises | 8084 | Spring Boot + Kafka |
| 6 | **image** | Upload/Stockage images | 8085 | Spring Boot + MinIO |
| 7 | **product** | Gestion des produits | 8086 | Spring Boot + Kafka |
| 8 | **payment** | Paiements (Monetbil/Pawapay) | 8087 | Spring Boot + Kafka |
| 9 | **likes** | Système de likes | 8090 | Spring Boot |
| 10 | **feedback** | Retours utilisateurs | 8091 | Spring Boot |
| 11 | **messages** | Messagerie interne | 8092 | Spring Boot + Kafka |
| 12 | **subscription** | Gestion des abonnements | 8093 | Spring Boot + Kafka |
| 13 | **order** | Gestion des commandes | 8089 | Spring Boot + Kafka |
| 14 | **stats** | Statistiques | 8094 | Spring Boot + Kafka |

### Détails par service

#### discovery (Service Discovery)
- **Rôle** : Enregistrement et découverte des services
- **Technologie** : Netflix Eureka Server
- **Port** : 8761
- **Particularité** : Ne s'enregistre pas lui-même

#### config-server (Configuration Server)
- **Rôle** : Configuration centralisée
- **Technologie** : Spring Cloud Config Server
- **Port** : 8088

#### gateway (API Gateway)
- **Rôle** : Point d'entrée unique, routage, rewrite d'URLs
- **Technologie** : Spring Cloud Gateway
- **Port** : 8060
- **Routes** : `/user`, `/product`, `/business`, `/payment`, `/order`, `/subscription`, `/messages`, `/likes`, `/feedback`, `/stats`

#### user
- **Rôle** : Gestion des utilisateurs, authentification
- **Technologie** : Spring Boot + PostgreSQL + Keycloak
- **Port** : 8083
- **Événements Kafka** : `user-created`

#### business
- **Rôle** : Gestion des businesses/commerces
- **Technologie** : Spring Boot + Kafka
- **Port** : 8084
- **Topics Kafka** : `user-created`, `subscription-created`, `business-created`

#### image
- **Rôle** : Upload et gestion des images
- **Technologie** : Spring Boot + MinIO
- **Port** : 8085
- **Taille max** : 10MB

#### product
- **Rôle** : Gestion des articles/produits
- **Technologie** : Spring Boot + Kafka
- **Port** : 8086
- **Topics Kafka** : `article-created`, `promotion-created`

#### payment
- **Rôle** : Paiements integrados (Monetbil, Pawapay)
- **Technologie** : Spring Boot + Kafka + Spring Batch
- **Port** : 8087
- **Fonctionnalités** : Webhooks, compte plateforme, tokens API externes

#### likes
- **Rôle** : Gestion des likes/préférences
- **Technologie** : Spring Boot
- **Port** : 8090

#### feedback
- **Rôle** : Retours et évaluations utilisateurs
- **Technologie** : Spring Boot
- **Port** : 8091

#### messages
- **Rôle** : Système de messagerie
- **Technologie** : Spring Boot + Kafka
- **Port** : 8092
- **Topics Kafka** : `business-created`, `conversation-created`, `order-request-created`

#### subscription
- **Rôle** : Gestion des abonnements
- **Technologie** : Spring Boot + Kafka
- **Port** : 8093
- **Cron** : Vérification expiration quotidienne

#### order
- **Rôle** : Gestion des commandes
- **Technologie** : Spring Boot + Kafka
- **Port** : 8089
- **Topics Kafka** : `user-created`, `order-request-created`, `transfer-request-created`

#### stats
- **Rôle** : Agrégation des statistiques
- **Technologie** : Spring Boot + Kafka
- **Port** : 8094
- **Topics Kafka** : `article-created`, `promotion-created`, `business-created`

---

## 4. Stack technique

### Versions

| Technologie | Version |
|-------------|---------|
| Java | 17+ |
| Spring Boot | 3.x |
| Spring Cloud | 2023.x |
| PostgreSQL | 15+ |
| Kafka | 3.x |
| MinIO | RELEASE.2023 |
| Docker | 24.x |
| Keycloak | 24.x |

### Dépendances principales

- `spring-boot-starter-web`
- `spring-boot-starter-data-jpa`
- `spring-cloud-starter-gateway`
- `spring-cloud-starter-netflix-eureka-client`
- `spring-kafka`
- `spring-boot-starter-oauth2-resource-server`
- `spring-boot-starter-actuator`
- `liquibase-core`
- `postgresql`
- `hibernate-core`

### Build

- **Maven** ou **Gradle** (selon le projet)
- **Docker** pour la containerisation

---

## 5. Prérequis

### Logiciels requis

```bash
# Java 17+
java -version

# Docker & Docker Compose
docker --version
docker-compose --version

# Maven ou Gradle
mvn --version
# ou
gradle --version
```

### Variables d'environnement nécessaires

```bash
# Base de données
DB_URL=jdbc:postgresql://postgres:5432/fastrelays
DB_USERNAME=postgres
DB_PASSWORD=****

# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:29092

# Eureka
EUREKA_URL=http://eureka:8761/eureka/

# Keycloak (OAuth2)
JWT_ISSUER_URI=https://keycloak.auth.io/realms/fastrelays
JWT_JWK_SET_URI=https://keycloak.auth.io/realms/fastrelays/protocol/openid-connect/certs
KEYCLOAK_REALM_NAME=fastrelays
KEYCLOAK_CLIENT_ID=fastrelays-app
KEYCLOAK_CLIENT_SECRET=****

# MinIO
MINIO_ENDPOINT=http://minio:9000
MINIO_ACCESS_KEY=****
MINIO_SECRET_KEY=****

# Logging
ZIPKIN_ENDPOINT=http://zipkin:9411/api/v2/spans
```

---

## 6. Installation & Démarrage

### Option 1 : Docker Compose (Recommandé)

```bash
# Cloner le projet
git clone https://github.com/fastrelays/fastrelays.git
cd fastrelays

# Lancer tous les services
docker-compose up -d

# Vérifier l'état
docker-compose ps

# Logs
docker-compose logs -f
docker-compose logs -f gateway
```

### Option 2 : Sans Docker

```bash
# Compiler les services
mvn clean package -DskipTests

# Lancer Eureka Discovery
java -jar discovery/target/discovery-*.jar

# Lancer chaque service (dans des terminaux séparés)
java -jar usermanager/target/usermanager-*.jar
java -jar gateway/target/gateway-*.jar
java -jar product/target/product-*.jar
# ... etc
```

### Ordre de démarrage recommandé

```bash
1. PostgreSQL    # Base de données
2. Kafka         # Messaging
3. Eureka        # Discovery (port 8761)
4. MinIO         # Stockage images
5. Config Server # Configuration (optionnel)
6. Gateway       # API Gateway (port 8060)
7. Services      # Microservices
```

---

## 7. Configuration

### Variables d'environnement

| Variable | Description | Défaut |
|----------|-------------|-------|
| `SERVER_PORT` | Port du service | Variable |
| `DB_URL` | URL PostgreSQL | - |
| `EUREKA_URL` | URL Eureka | - |
| `KAFKA_BOOTSTRAP_SERVERS` | Adresse Kafka | kafka:29092 |
| `JWT_ISSUER_URI` | Issuer JWT | - |

### Profils Spring

| Profil | Usage |
|--------|-------|
| `dev` | Développement local |
| `test` | Tests unitaires |
| `prod` | Production |

```bash
# Exemple d'utilisation
SPRING_PROFILES_ACTIVE=dev java -jar app.jar
```

### Fichiers de configuration

Chaque service possède un fichier YAML dans `config-fastrelay/config/` :

- `application.yaml` - Configuration commune
- `discovery.yaml` - Eureka
- `gateway.yaml` - API Gateway
- `usermanager.yaml` - Utilisateurs
- `product.yaml` - Produits
- `order.yaml` - Commandes
- `payment.yaml` - Paiements
- etc.

---

## 8. Base de données

### Schéma par service

| Service | Base de données | Type |
|---------|-----------------|------|
| usermanager | PostgreSQL | Utilisateurs, roles |
| application | PostgreSQL | Config commune |
| business | PostgreSQL | Businesses |
| product | PostgreSQL | Articles, promotions |
| order | PostgreSQL | Commandes |
| payment | PostgreSQL | Transactions |
| subscription | PostgreSQL | Abonnements |
| messages | PostgreSQL | Conversations |
| likes | PostgreSQL | Likes |
| feedback | PostgreSQL | Feedbacks |
| image | PostgreSQL | Métadonnées images |
| stats | PostgreSQL | Statistiques agrégées |

### Migrations

- **Outil** : Liquibase
- **ChangeLog** : `classpath:db/changelog/master.xml`

```bash
# Activer les migrations
SPRING_LIQUIBASE_ENABLED=true
```

---

## 9. API & Endpoints

### Points d'accès

Tous les services sont accessibles via le Gateway (port 8060) :

| Route | Service | Description |
|-------|---------|-------------|
| `/user/**` | usermanager | Gestion utilisateurs |
| `/product/**` | product | Gestion produits |
| `/business/**` | business | Gestion businesses |
| `/order/**` | order | Gestion commandes |
| `/payment/**` | payment | Paiements |
| `/subscription/**` | subscription | Abonnements |
| `/messages/**` | messages | Messagerie |
| `/likes/**` | likes | Likes |
| `/feedback/**` | feedback | Feedbacks |
| `/stats/**` | stats | Statistiques |
| `/image/**` | image | Upload images |

### Documentation API

- **Swagger/OpenAPI** : `/swagger-ui.html` (si activé)
- **Actuator** : `/actuator/**`

```bash
# Endpoints actuator
curl http://localhost:8060/actuator/health
curl http://localhost:8060/actuator/info
curl http://localhost:8060/actuator/gateway/routes
```

---

## 10. Communication inter-services

### REST (Synchronous)

```bash
# Via Gateway
curl -H "Authorization: Bearer $TOKEN" \
     http://localhost:8060/user/api/v1/profile
```

### Kafka (Asynchronous)

#### Topics Kafka

| Topic | Producteur | Consommateur |
|-------|-----------|--------------|
| `user-created` | usermanager | business, order |
| `business-created` | business | messages, stats |
| `order-request-created` | order | messages, payment |
| `subscription-created` | subscription | business |
| `subscription-expired` | subscription | business |
| `article-created` | product | stats |
| `promotion-created` | product | stats |

#### Configuration Kafka

```yaml
spring:
  kafka:
    bootstrap-servers: kafka:29092
    consumer:
      group-id: "<service>-service"
      auto-offset-reset: earliest
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

---

## 11. Tests

### Exécuter les tests

```bash
# Tous les tests
mvn test

# Tests unitaires uniquement
mvn test -DskipIntegrationTests

# Tests d'intégration
mvn verify -Dspring.profiles.active=test
```

### Couverture

```bash
# Rapport de couverture
mvn jacoco:report
# Voir target/site/jacoco/index.html
```

---

## 12. Build & Déploiement

### Build Maven

```bash
# Compiler le projet
mvn clean package

# Sans tests
mvn clean package -DskipTests
```

### Docker

```bash
# Build image
docker build -t fastrelays/gateway:latest .

# Push vers registry
docker push fastrelays/gateway:latest
```

### Déploiement Kubernetes

```bash
# Appliquer les manifeste
kubectl apply -f k8s/
```

---

## 13. Monitoring & Logs

### Spring Boot Actuator

| Endpoint | Description |
|----------|-------------|
| `/actuator/health` | Santé du service |
| `/actuator/info` | Informations |
| `/actuator/metrics` | Métriques |
| `/actuator/loggers` | Configuration logs |
| `/actuator/prometheus` | Pour Prometheus |

```bash
# Vérifier la santé
curl http://localhost:8060/actuator/health
```

### Tracing distribué

- **Zipkin** : `ZIPKIN_ENDPOINT`
- **Tracing** : Spring Cloud Sleuth

```yaml
management:
  zipkin:
    tracing:
      endpoint: ${ZIPKIN_ENDPOINT}
  tracing:
    sampling:
      probability: 1.0
```

### Logs centralisés

- Configuration ELK (Elasticsearch, Logstash, Kibana)
- Ou Loki + Grafana

---

## 14. Bonnes pratiques / Recommandations

### Scaling

- **Horizontal** : Ajouter des instances Kubernetes
- **Stateless** : Tous les services doivent être stateless
- **Sessions** : Utiliser Redis pour les sessions

### Gestion des erreurs

- **Circuit Breaker** : Resilience4j
- **Retry** : Spring Retry
- **Fallback** : Méthodes de repli

```yaml
resilience4j:
  circuitbreaker:
    registerHealthIndicator: true
    slidingWindowSize: 10
    failureRateThreshold: 50
```

### Sécurité

- **HTTPS** : Toujours en production
- **JWT** : Validation des tokens
- **CORS** : Configurer les origines autorisées
- **Secrets** : Ne pas commiter les mots de passe

```yaml
security:
  cors:
    allowed-origins: "https://fastrelays.com"
    allowed-methods: "GET,POST,PUT,DELETE"
    allowed-headers: "Authorization,Content-Type"
```

### Monitoring

- **Métriques** : Micrometer
- **Alerting** : Prometheus + Alertmanager
- **Dashboards** : Grafana

---

## 15. Limitations connues

- ⚠️ **Stateless** : Les services doivent être stateless pour le scaling horizontal
- ⚠️ **Transactions distribuées** : Non supportées nativement (utiliser SAGA pattern)
- ⚠️ **Configuration** : Lesconfigs sont externies via Spring Cloud Config Server
- ⚠️ **Base de données** : Une seule DB par service (的最佳 pratiques)

---

## 16. Contribution

### Procédure

1. **Forker** le projet
2. Créer une **branche** feature (`git checkout -b feature/ma-feature`)
3. **Committer** vos changements (`git commit -m 'Ajout feature'`)
4. **Pusher** (`git push origin feature/ma-feature`)
5. Créer une **Pull Request**

### Standards de code

- Java 17+
- Conventions Spring
- Tests unitaires obligatoires
- Documentation Javadoc

---

## 17. Licence

Ce projet est sous licence **MIT**.

---

## Annexe : Ports par défaut

### Microservices

| Service | Port |
|---------|------|
| Gateway | 8060 |
| User | 8083 |
| Business | 8084 |
| Image | 8085 |
| Product | 8086 |
| Payment | 8087 |
| Order | 8089 |
| Likes | 8090 |
| Feedback | 8091 |
| Messages | 8092 |
| Subscription | 8093 |
| Stats | 8094 |

### Infrastructure

| Service | Port |
|---------|------|
| Discovery (Eureka) | 8761 |
| Config Server | 8088 |
| Kafka | 29092 (interne) / 9092 (externe) |
| Kafka UI | 8081 |
| PostgreSQL | 5432 (interne, voir ci-dessous) |
| PostgreSQL (user) | 5433 |
| PostgreSQL (business) | 5434 |
| PostgreSQL (product) | 5435 |
| PostgreSQL (image) | 5436 |
| PostgreSQL (keycloak) | 5437 |
| PostgreSQL (order) | 5438 |
| PostgreSQL (subscription) | 5439 |
| PostgreSQL (payment) | 5440 |
| PostgreSQL (likes) | 5441 |
| PostgreSQL (feedback) | 5442 |
| PostgreSQL (messages) | 5443 |
| PostgreSQL (stats) | 5444 |
| MinIO | 9000 (API) / 9001 (Console) |
| Zipkin | 9411 |
| Keycloak | 8080 |
| pgAdmin | 5050 |
| Zookeeper | 2181 |

---

*Document généré automatiquement à partir des configurations FastRelays*