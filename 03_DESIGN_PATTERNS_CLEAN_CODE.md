# 03 — Design Patterns & Clean Code

## Design Patterns — Les 3 Familles

### 1. Patterns Créationnels

#### Singleton
```java
// Thread-safe avec double-checked locking
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;

    private DatabaseConnection() {}

    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}

// Meilleure approche : Enum Singleton (thread-safe par nature)
public enum AppConfig {
    INSTANCE;
    private final Properties props = loadProperties();
}
```

#### Builder
```java
public class Order {
    private final String reference;
    private final Customer customer;
    private final List<OrderLine> lines;
    private final LocalDate deliveryDate;
    private final String notes; // optionnel

    private Order(Builder builder) {
        this.reference = builder.reference;
        this.customer = builder.customer;
        this.lines = List.copyOf(builder.lines);
        this.deliveryDate = builder.deliveryDate;
        this.notes = builder.notes;
    }

    public static class Builder {
        private final String reference;    // obligatoire
        private final Customer customer;   // obligatoire
        private List<OrderLine> lines = new ArrayList<>();
        private LocalDate deliveryDate;
        private String notes;

        public Builder(String reference, Customer customer) {
            this.reference = Objects.requireNonNull(reference);
            this.customer = Objects.requireNonNull(customer);
        }
        public Builder line(OrderLine line) { lines.add(line); return this; }
        public Builder deliveryDate(LocalDate date) { this.deliveryDate = date; return this; }
        public Builder notes(String notes) { this.notes = notes; return this; }
        public Order build() { return new Order(this); }
    }
}

// Usage
Order order = new Order.Builder("ORD-001", customer)
    .line(new OrderLine(product, 2))
    .deliveryDate(LocalDate.now().plusDays(3))
    .notes("Livraison urgente")
    .build();
```

#### Factory Method
```java
// Abstraction
public interface NotificationSender {
    void send(String message, String recipient);
}

// Implémentations
public class EmailSender implements NotificationSender { ... }
public class SmsSender implements NotificationSender { ... }
public class PushSender implements NotificationSender { ... }

// Factory
public class NotificationFactory {
    public static NotificationSender create(NotificationType type) {
        return switch (type) {
            case EMAIL -> new EmailSender();
            case SMS -> new SmsSender();
            case PUSH -> new PushSender();
        };
    }
}
```

---

### 2. Patterns Structurels

#### Adapter
```java
// Interface attendue par le système
public interface PaymentGateway {
    PaymentResult processPayment(PaymentRequest request);
}

// API externe incompatible
public class LegacyPaymentApi {
    public LegacyResponse chargeCard(String cardNumber, int amount, String currency) { ... }
}

// Adapter : adapte l'API legacy à l'interface attendue
public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentApi legacyApi;

    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        LegacyResponse response = legacyApi.chargeCard(
            request.getCardNumber(),
            request.getAmount().intValue(),
            request.getCurrency()
        );
        return new PaymentResult(response.getCode().equals("00"), response.getTransactionId());
    }
}
```

#### Decorator
```java
public interface OrderService {
    Order create(CreateOrderRequest request);
}

// Implémentation de base
public class OrderServiceImpl implements OrderService { ... }

// Decorator : ajoute du logging sans modifier OrderServiceImpl
public class LoggingOrderService implements OrderService {
    private final OrderService delegate;

    @Override
    public Order create(CreateOrderRequest request) {
        log.info("Creating order for customer {}", request.getCustomerId());
        try {
            Order order = delegate.create(request);
            log.info("Order {} created successfully", order.getId());
            return order;
        } catch (Exception e) {
            log.error("Failed to create order", e);
            throw e;
        }
    }
}
```

#### Proxy (et Spring AOP)
```java
// Spring AOP = Proxy pattern appliqué
@Aspect
@Component
public class PerformanceAspect {

    @Around("@annotation(Monitored)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            log.info("{} took {}ms", pjp.getSignature().getName(), elapsed);
        }
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Monitored {}
```

---

### 3. Patterns Comportementaux

#### Strategy
```java
// Interface stratégie
public interface SortStrategy<T> {
    List<T> sort(List<T> items, Comparator<T> comparator);
}

// Stratégies concrètes
public class QuickSort<T> implements SortStrategy<T> { ... }
public class MergeSort<T> implements SortStrategy<T> { ... }

// Contexte
public class DataProcessor<T> {
    private SortStrategy<T> sortStrategy;

    public void setStrategy(SortStrategy<T> strategy) {
        this.sortStrategy = strategy;
    }

    public List<T> process(List<T> data, Comparator<T> comparator) {
        return sortStrategy.sort(data, comparator);
    }
}
```

#### Observer (Event-Driven avec Spring)
```java
// Event
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}

// Publisher
@Service
public class OrderService {
    private final ApplicationEventPublisher publisher;

    public Order create(CreateOrderRequest request) {
        Order order = orderRepo.save(new Order(request));
        publisher.publishEvent(new OrderCreatedEvent(this, order));
        return order;
    }
}

// Listeners (décorrélés)
@Component
public class EmailNotificationListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendConfirmation(event.getOrder());
    }
}

@Component
public class InventoryListener {
    @EventListener
    @Async  // asynchrone
    public void onOrderCreated(OrderCreatedEvent event) {
        inventoryService.reserve(event.getOrder().getLines());
    }
}
```

#### Template Method
```java
// Squelette de l'algorithme dans la classe abstraite
public abstract class ReportGenerator {

    // Template method : définit la séquence
    public final Report generate(ReportRequest request) {
        List<RawData> data = fetchData(request);       // abstract
        List<ProcessedData> processed = processData(data); // hook (optionnel)
        return formatReport(processed);                 // abstract
    }

    protected abstract List<RawData> fetchData(ReportRequest request);
    protected abstract Report formatReport(List<ProcessedData> data);

    // Hook : peut être surchargé mais a une impl par défaut
    protected List<ProcessedData> processData(List<RawData> data) {
        return data.stream().map(this::transform).toList();
    }
}

// Implémentation concrète
public class PdfReportGenerator extends ReportGenerator {
    @Override
    protected List<RawData> fetchData(ReportRequest request) { ... }

    @Override
    protected Report formatReport(List<ProcessedData> data) {
        return pdfService.generate(data);
    }
}
```

---

## Clean Code

### Nommage

```java
// BAD
int d; // elapsed time in days
List<int[]> theList;
void processItems(List<int[]> theList) { ... }

// GOOD
int elapsedDays;
List<Cell> gameBoard;
void getFlaggedCells(List<Cell> gameBoard) { ... }

// Méthodes : verbe + nom
getUserById(Long id)
calculateTotalPrice(List<OrderLine> lines)
isEligibleForDiscount(Customer customer)
```

### Fonctions

```java
// Principe : une fonction = une chose
// BAD : fait trop de choses
public void processOrder(Order order) {
    validate(order);
    save(order);
    sendEmail(order.getCustomer().getEmail());
    updateInventory(order.getLines());
}

// GOOD : orchestration explicite
public void processOrder(Order order) {
    validateOrder(order);
    Order saved = orderRepository.save(order);
    notifyCustomer(saved);
    reserveInventory(saved);
}

// Arguments : max 3 (idéalement 0-2)
// BAD
createUser(String name, String email, String phone, int age, String city, String role)
// GOOD
createUser(CreateUserRequest request)
```

### DRY, KISS, YAGNI

```java
// DRY (Don't Repeat Yourself) : extraire la logique commune
// KISS (Keep It Simple, Stupid) : solution la plus simple possible
// YAGNI (You Ain't Gonna Need It) : ne pas coder ce qui n'est pas requis

// BAD : duplication de validation
public void createUser(User u) {
    if (u.getEmail() == null || !u.getEmail().contains("@")) throw new IllegalArgumentException();
}
public void updateUser(User u) {
    if (u.getEmail() == null || !u.getEmail().contains("@")) throw new IllegalArgumentException();
}

// GOOD : extraction
private void validateEmail(String email) {
    if (email == null || !email.contains("@"))
        throw new IllegalArgumentException("Email invalide: " + email);
}
```

---

## Questions d'Interview — Design Patterns

---

### Q1 : Quand utiliser le pattern Strategy vs Template Method ?

**Réponse :**

| Critère | Strategy | Template Method |
|---------|----------|-----------------|
| **Mécanisme** | Composition (interface injectée) | Héritage (méthodes abstraites) |
| **Changement** | À l'exécution (runtime) | À la compilation |
| **Couplage** | Faible | Plus fort (héritage) |
| **Favoriser** | Oui (composition > héritage) | Si séquence fixe avec variantes |

**Exemple de justification :** Pour un système de calcul de prix avec différentes stratégies de remise → **Strategy** car les règles changent selon le client. Pour un pipeline de traitement ETL avec des étapes fixes (extract → transform → load) → **Template Method**.

---

### Q2 : Expliquez le pattern Repository et pourquoi l'utiliser.

**Réponse :**

Le Repository isole la couche métier de la couche de persistance. Le domaine ne connaît que des interfaces.

```java
// Interface dans le domaine
public interface OrderRepository {
    Optional<Order> findById(Long id);
    Order save(Order order);
    List<Order> findByCustomer(Customer customer);
}

// Implémentation JPA dans l'infrastructure
@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager em;

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(em.find(Order.class, id));
    }
}

// Avantages :
// 1. Testabilité : mocker l'interface sans base de données
// 2. Isolation : changer de BDD (MySQL → PostgreSQL) sans toucher le domaine
// 3. Expressivité : noms métier plutôt que SQL
```

---

### Q3 : Qu'est-ce que le principe d'inversion de dépendance (DIP) ?

**Réponse :**

Les modules de haut niveau ne doivent pas dépendre des modules de bas niveau. Les deux doivent dépendre d'**abstractions**.

```java
// BAD : OrderService dépend de MySQLOrderRepo (implémentation concrète)
public class OrderService {
    private MySQLOrderRepo repo = new MySQLOrderRepo();
    // Si on change de BDD → modifier OrderService
}

// GOOD : dépend de l'interface
public class OrderService {
    private final OrderRepository repo;  // abstraction

    public OrderService(OrderRepository repo) {  // injection
        this.repo = repo;
    }
}
// Spring injecte JpaOrderRepository automatiquement
// Pour les tests : inject MockOrderRepository
```

---

### Q4 : Comment éviter les anti-patterns courants en Java ?

**Réponse :**

```java
// Anti-pattern 1 : God Class (classe qui fait tout)
// Solution : découper par responsabilité (SRP)

// Anti-pattern 2 : Primitive Obsession
// BAD
public void createUser(String email, String phone, int zipCode) {}
// GOOD
public void createUser(Email email, PhoneNumber phone, ZipCode zipCode) {}
// Value Objects : validation dans le constructeur, immutables

// Anti-pattern 3 : Magic Numbers
// BAD
if (status == 3) { ... }
// GOOD
if (status == OrderStatus.CANCELLED) { ... }

// Anti-pattern 4 : Long Method
// Règle : une méthode doit tenir sur un écran (< 20 lignes)
// Solution : extraire des méthodes privées nommées

// Anti-pattern 5 : Feature Envy (méthode qui utilise plus d'autres classes que la sienne)
// BAD dans OrderService
double total = order.getLines().stream()
    .mapToDouble(l -> l.getProduct().getPrice() * l.getQuantity() * (1 - l.getDiscount()))
    .sum();
// GOOD : méthode dans Order
double total = order.calculateTotal();
```

---

## Pièges Classiques

- Utiliser Singleton pour des objets avec état mutable partagé → problèmes de concurrence
- Créer des hiérarchies d'héritage profondes → préférer la composition
- Observer synchrone pour des opérations lentes → bloquer le thread principal
- Builder sans validation dans `build()` → objet incohérent créé silencieusement
- Confondre Proxy (contrôle d'accès) et Decorator (ajout de comportement)
