# 09 — Docker

## 1. Concepts Fondamentaux

| Concept | Description |
|---------|-------------|
| **Image** | Template immuable (layers) |
| **Container** | Instance en cours d'exécution d'une image |
| **Registry** | Stockage d'images (Docker Hub, ECR, GCR) |
| **Dockerfile** | Script de construction d'une image |
| **Volume** | Persistance de données hors du container |
| **Network** | Communication entre containers |

---

## 2. Dockerfile Spring Boot Optimisé

```dockerfile
# Multi-stage build : sépare le build de l'image finale
# Stage 1 : Build avec Maven
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Copier d'abord le pom.xml pour bénéficier du cache Docker
# Si pom.xml ne change pas → layer Maven cache réutilisé
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .
RUN ./mvnw dependency:go-offline -B

# Copier les sources et builder
COPY src/ src/
RUN ./mvnw package -DskipTests -B

# Stage 2 : Image finale légère (JRE seulement)
FROM eclipse-temurin:21-jre-alpine AS runtime

# Sécurité : utilisateur non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

WORKDIR /app

# Layers Spring Boot : classes modifiées fréquemment séparées des dépendances
COPY --from=build /app/target/app.jar app.jar

# Optimisation JVM pour containers
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Layers Docker Spring Boot

```dockerfile
# Optimisation avancée : layered JAR Spring Boot
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=build /app/target/app.jar app.jar

# Extraire les layers (dépendances changent rarement, code souvent)
RUN java -Djarmode=layertools -jar app.jar extract
# Crée : dependencies/ | spring-boot-loader/ | snapshot-dependencies/ | application/

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=runtime /app/dependencies/ ./
COPY --from=runtime /app/spring-boot-loader/ ./
COPY --from=runtime /app/snapshot-dependencies/ ./
COPY --from=runtime /app/application/ ./  # seule cette layer change à chaque build
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

---

## 3. Docker Compose (Environnement de Dev)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: runtime
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/mydb
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      mysql:
        condition: service_healthy
      kafka:
        condition: service_healthy
    networks:
      - app-network

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: mydb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    healthcheck:
      test: kafka-topics --bootstrap-server localhost:9092 --list
      interval: 30s
    networks:
      - app-network

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - app-network

volumes:
  mysql-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

---

## 4. Commandes Essentielles

```bash
# Build et run
docker build -t myapp:1.0 .
docker run -d -p 8080:8080 --name myapp myapp:1.0
docker run -d -p 8080:8080 -e "SPRING_PROFILES_ACTIVE=prod" --name myapp myapp:1.0

# Inspection
docker logs -f myapp          # logs en temps réel
docker exec -it myapp sh      # shell dans le container
docker stats                  # CPU/mémoire en temps réel
docker inspect myapp          # configuration complète

# Nettoyage
docker system prune -af       # supprimer tout ce qui est inutilisé
docker image prune            # images non utilisées
docker volume prune           # volumes orphelins

# Docker Compose
docker-compose up -d          # démarrer tous les services
docker-compose down -v        # arrêter et supprimer volumes
docker-compose logs -f app    # logs d'un service
docker-compose exec app sh    # shell dans un service
```

---

## Questions d'Interview — Docker

---

### Q1 : Qu'est-ce qu'un layer Docker et pourquoi est-ce important ?

**Réponse :**

Chaque instruction `RUN`, `COPY`, `ADD` dans un Dockerfile crée un **layer immuable**. Docker cache les layers et ne recrée que ceux qui ont changé.

```dockerfile
# MAUVAIS : une seule couche, tout recalculé à chaque changement de code
COPY . .
RUN mvn package

# BON : les dépendances Maven sont dans un layer séparé du code
COPY pom.xml .
RUN mvn dependency:go-offline  # layer caché si pom.xml ne change pas
COPY src/ src/
RUN mvn package                # seul ce layer est recréé si le code change
```

---

### Q2 : Comment gérer les secrets dans Docker (ne pas mettre dans l'image) ?

**Réponse :**

```bash
# JAMAIS dans le Dockerfile ou l'image
ENV DB_PASSWORD=secret123  # mauvaise pratique → visible dans docker inspect

# BONNE PRATIQUE 1 : variables d'environnement à l'exécution
docker run -e DB_PASSWORD=$DB_PASSWORD myapp

# BONNE PRATIQUE 2 : fichier .env (non commité dans git)
# .env
DB_PASSWORD=secret123
docker-compose --env-file .env up

# BONNE PRATIQUE 3 : Docker Secrets (Swarm) ou Kubernetes Secrets
docker secret create db_password ./password.txt
# Dans le service : monté en fichier /run/secrets/db_password

# BONNE PRATIQUE 4 : Vault / AWS Secrets Manager
# L'application récupère les secrets au démarrage via API
```

---

### Q3 : Quelle est la différence entre CMD et ENTRYPOINT ?

**Réponse :**

| | `ENTRYPOINT` | `CMD` |
|-|-------------|-------|
| **Rôle** | Commande principale (fixe) | Arguments par défaut (remplaçables) |
| **Override** | `docker run --entrypoint` | `docker run image <cmd>` |
| **Combinaison** | `ENTRYPOINT + CMD` = commande + args par défaut |

```dockerfile
# ENTRYPOINT fixe l'exécutable, CMD fournit des arguments par défaut
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=prod"]

# docker run myapp  → java -jar app.jar --spring.profiles.active=prod
# docker run myapp --spring.profiles.active=dev  → surcharge CMD
# docker run --entrypoint sh myapp  → ouvre un shell
```

---

## Pièges Classiques

- Lancer le process en tant que `root` dans le container → risque de sécurité
- Image de prod avec JDK complet (> 500MB) → utiliser JRE ou distroless (< 100MB)
- Copier le `.git` ou `target/` dans l'image → `.dockerignore` obligatoire
- `docker-compose` en prod sans orchestrateur → utiliser Kubernetes pour la haute disponibilité
- Stocker des données dans le container sans volume → données perdues au redémarrage
