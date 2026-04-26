# 04 — Architecture Microservices

## 1. Principes Fondamentaux

### Définition
Un microservice est un service **autonome**, **déployable indépendamment**, responsable d'une **capacité métier** unique.

### Les 12 Facteurs (12-Factor App)
| # | Facteur | Exemple |
|---|---------|---------|
| 1 | Codebase | Un repo par service |
| 2 | Dépendances | Déclarées explicitement (pom.xml) |
| 3 | Configuration | Variables d'environnement (jamais dans le code) |
| 4 | Services externes | DB, cache = ressources attachées (configurables) |
| 5 | Build/Release/Run | Étapes séparées et immuables |
| 6 | Processus | Stateless, pas d'état en mémoire partagée |
| 7 | Port binding | Service expose un port (ex: 8080) |
| 8 | Concurrence | Scale par processus (pas threads) |
| 9 | Jetabilité | Démarrage rapide, arrêt propre (graceful shutdown) |
| 10 | Parité dev/prod | Environnements les plus proches possible |
| 11 | Logs | Traiter comme flux d'événements (stdout) |
| 12 | Admin | Tâches admin = processus ponctuels |

---

## 2. Communication entre Services

### Synchrone (REST / gRPC)
```
Client → [API Gateway] → [Order Service]
                       → [Customer Service]
                       → [Inventory Service]
```

```java
// RestTemplate (legacy)
RestTemplate restTemplate = new RestTemplate();
CustomerDto customer = restTemplate.getForObject(
    "http://customer-service/api/customers/{id}", CustomerDto.class, customerId
);

// WebClient (réactif, non-bloquant — Spring WebFlux)
WebClient client = WebClient.builder()
    .baseUrl("http://customer-service")
    .build();

Mono<CustomerDto> customerMono = client.get()
    .uri("/api/customers/{id}", customerId)
    .retrieve()
    .onStatus(HttpStatus::is4xxClientError, resp -> Mono.error(new CustomerNotFoundException()))
    .bodyToMono(CustomerDto.class);

// OpenFeign (Spring Cloud) : interface déclarative
@FeignClient(name = "customer-service", fallback = CustomerFallback.class)
public interface CustomerClient {
    @GetMapping("/api/customers/{id}")
    CustomerDto getById(@PathVariable Long id);
}
```

### Asynchrone (Kafka / AMQP)
```
[Order Service] → Kafka → [Email Service]
                        → [Inventory Service]
                        → [Analytics Service]
```

---

## 3. Patterns d'Architecture

### API Gateway
```
Internet → [API Gateway]
              ├── Routage (path-based, header-based)
              ├── Auth (JWT validation)
              ├── Rate Limiting
              ├── Load Balancing
              └── Circuit Breaker
           → [Service A] [Service B] [Service C]
```

```yaml
# Spring Cloud Gateway
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service    # lb = load balanced
          predicates:
            - Path=/api/orders/**
          filters:
            - AuthFilter
            - name: CircuitBreaker
              args:
                name: orderCircuitBreaker
                fallbackUri: forward:/fallback/orders
```

### Circuit Breaker (Resilience4j)
```java
@Service
public class OrderService {

    @CircuitBreaker(name = "inventory", fallbackMethod = "fallbackInventory")
    @Retry(name = "inventory", fallbackMethod = "fallbackInventory")
    @TimeLimiter(name = "inventory")
    public CompletableFuture<Boolean> checkInventory(Long productId, int quantity) {
        return CompletableFuture.supplyAsync(() ->
            inventoryClient.isAvailable(productId, quantity)
        );
    }

    private CompletableFuture<Boolean> fallbackInventory(Long productId, int quantity, Exception ex) {
        log.warn("Inventory service unavailable, using fallback", ex);
        return CompletableFuture.completedFuture(true); // optimistic fallback
    }
}
```

**États du Circuit Breaker :**
```
CLOSED → (seuil d'échecs atteint) → OPEN → (timeout) → HALF_OPEN → (succès) → CLOSED
                                                                    → (échec) → OPEN
```

### Saga Pattern (Transactions distribuées)
```
// Orchestration Saga
[Order Service] → crée commande (PENDING)
               → appelle [Payment Service]
               → si succès → appelle [Inventory Service]
               → si succès → confirme commande (CONFIRMED)
               → si échec Inventory → annule paiement (compensation)
               → passe commande à FAILED

// Choreography Saga (event-driven)
OrderCreated → [Payment Service] → PaymentProcessed
PaymentProcessed → [Inventory Service] → InventoryReserved
InventoryReserved → [Order Service] → OrderConfirmed
PaymentFailed → [Order Service] → OrderCancelled
```

### CQRS (Command Query Responsibility Segregation)
```java
// Séparation des modèles de lecture et d'écriture
// Write side : commandes
@CommandHandler
public class CreateOrderCommandHandler {
    public Order handle(CreateOrderCommand cmd) {
        Order order = Order.create(cmd);
        orderRepository.save(order);
        eventBus.publish(new OrderCreatedEvent(order));
        return order;
    }
}

// Read side : projections optimisées pour la lecture
@EventHandler
public class OrderReadModelProjection {
    @HandleEvent
    public void on(OrderCreatedEvent event) {
        // Mise à jour d'une vue dénormalisée (ex: MongoDB, Elasticsearch)
        orderSummaryRepo.save(new OrderSummary(event));
    }
}
```

### Event Sourcing
```java
// L'état de l'entité = séquence d'événements (audit trail)
public class Order {
    private List<DomainEvent> events = new ArrayList<>();
    private OrderStatus status;
    private List<OrderLine> lines;

    public static Order reconstitute(List<DomainEvent> history) {
        Order order = new Order();
        history.forEach(order::apply);
        return order;
    }

    private void apply(DomainEvent event) {
        if (event instanceof OrderCreatedEvent e) {
            this.status = OrderStatus.PENDING;
            this.lines = e.getLines();
        } else if (event instanceof OrderConfirmedEvent e) {
            this.status = OrderStatus.CONFIRMED;
        }
    }
}
```

---

## 4. Service Discovery & Load Balancing

```yaml
# Eureka Server (registre de services)
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistry { ... }

# Eureka Client (chaque microservice)
eureka:
  client:
    serviceUrl:
      defaultZone: http://registry:8761/eureka/
  instance:
    preferIpAddress: true
```

---

## 5. Observabilité (Logs, Metrics, Traces)

```java
// Distributed Tracing avec Spring Sleuth / Micrometer Tracing
// Chaque requête = TraceId + SpanId propagés dans les headers
// X-B3-TraceId: abc123 | X-B3-SpanId: def456

// Logs structurés
log.info("Order created",
    kv("orderId", order.getId()),
    kv("customerId", order.getCustomerId()),
    kv("traceId", tracer.currentSpan().context().traceId())
);

// Health Checks (Spring Actuator)
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

---

## Questions d'Interview — Microservices

---

### Q1 : Microservices vs Monolithe — Quand choisir quoi ?

**Réponse :**

| Critère | Monolithe | Microservices |
|---------|-----------|---------------|
| **Équipe** | Petite (< 10) | Grande, multi-équipes |
| **Domaine** | Simple, peu de sous-domaines | Complexe, domaines métier distincts |
| **Déploiement** | Simple | Complexe (orchestration) |
| **Scalabilité** | Tout ou rien | Granulaire par service |
| **Latence** | Appels in-process | Réseau (10-100ms par appel) |
| **Consistance** | ACID (transactions locales) | Eventual consistency |

**Recommandation :** Commencer en **monolithe modulaire**, puis extraire en microservices quand la douleur est identifiable (équipe, scalabilité, déploiement indépendant).

---

### Q2 : Comment gérer la consistance des données entre microservices ?

**Réponse :**

Il n'y a pas de transaction distribuée ACID dans les microservices. Les solutions :

1. **Saga Pattern** : séquence de transactions locales avec compensations
2. **Outbox Pattern** : écrire l'événement dans la même BDD que la donnée (atomique), puis le publier
3. **Eventual Consistency** : accepter que les données soient temporairement inconsistantes

```java
// Outbox Pattern : évite la perte de messages
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepo.save(new Order(request));
    // Dans la même transaction, écrire l'event dans une table outbox
    outboxRepo.save(new OutboxEvent("ORDER_CREATED", order.toJson()));
    return order;
    // Un processus séparé lit l'outbox et publie sur Kafka
}
```

---

### Q3 : Qu'est-ce que le pattern Strangler Fig ?

**Réponse :**

Stratégie de migration d'un monolithe vers des microservices de manière incrémentale :

```
Phase 1 : [API Gateway] → [Monolithe]

Phase 2 : [API Gateway] → /orders → [Order Microservice] (nouveau)
                       → /*      → [Monolithe]

Phase 3 : [API Gateway] → /orders    → [Order Service]
                       → /customers → [Customer Service]
                       → /*         → [Monolithe restant]

Phase N : [Monolithe] retiré complètement
```

**Avantage :** Pas de "big bang rewrite", risque minimal, livraison continue de valeur.

---

### Q4 : Comment implémenter le pattern Outbox pour garantir la livraison des messages ?

**Réponse :**

```
1. Transaction DB : save(Order) + save(OutboxEvent)  ← atomique
2. Outbox Poller (Debezium CDC ou scheduler) lit les events non publiés
3. Publie sur Kafka
4. Marque l'event comme publié
```

```java
@Entity
public class OutboxEvent {
    @Id
    private UUID id = UUID.randomUUID();
    private String aggregateType;  // "Order"
    private String aggregateId;
    private String eventType;      // "ORDER_CREATED"
    private String payload;        // JSON
    private boolean published = false;
    private LocalDateTime createdAt = LocalDateTime.now();
}

// Scheduler (alternative à CDC)
@Scheduled(fixedDelay = 1000)
@Transactional
public void processOutbox() {
    List<OutboxEvent> pending = outboxRepo.findByPublishedFalse();
    pending.forEach(event -> {
        kafkaTemplate.send(event.getAggregateType(), event.getPayload());
        event.setPublished(true);
    });
    outboxRepo.saveAll(pending);
}
```

---

## Pièges Classiques

- **Distributed Monolith** : microservices couplés qui ne peuvent pas être déployés indépendamment
- **Chatty services** : trop d'appels synchrones entre services → latence cumulée
- **Partager une base de données** entre microservices → couplage fort, anti-pattern majeur
- Oublier l'**idempotence** dans les consumers Kafka → double traitement lors de rejeux
- Ne pas prévoir le **graceful shutdown** → messages perdus en cours de traitement
