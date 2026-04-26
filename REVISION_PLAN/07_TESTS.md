# 07 — Tests (JUnit 5, Mockito, BDD)

## 1. JUnit 5 — Les Bases

### Structure d'un Test

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        // Initialisation avant chaque test
    }

    @Test
    @DisplayName("should create order and notify customer")
    void createOrder_WhenValidRequest_ShouldCreateAndNotify() {
        // GIVEN (Arrange)
        CreateOrderRequest request = new CreateOrderRequest(1L, List.of(new OrderLineDto(1L, 2)));
        Customer customer = new Customer(1L, "Alice", "alice@example.com");
        when(customerRepository.findById(1L)).thenReturn(Optional.of(customer));
        when(orderRepository.save(any(Order.class))).thenAnswer(inv -> {
            Order o = inv.getArgument(0);
            o.setId(100L);
            return o;
        });

        // WHEN (Act)
        Order result = orderService.create(request);

        // THEN (Assert)
        assertThat(result.getId()).isEqualTo(100L);
        assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);
        verify(emailService).sendConfirmation(customer, result);
        verifyNoMoreInteractions(emailService);
    }

    @Test
    void createOrder_WhenCustomerNotFound_ShouldThrowException() {
        // GIVEN
        when(customerRepository.findById(anyLong())).thenReturn(Optional.empty());

        // WHEN / THEN
        assertThatThrownBy(() -> orderService.create(new CreateOrderRequest(99L, List.of())))
            .isInstanceOf(CustomerNotFoundException.class)
            .hasMessageContaining("99");
    }
}
```

### Assertions (AssertJ — Préféré à JUnit natif)

```java
// Scalaires
assertThat(result).isNotNull();
assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);
assertThat(result.getAmount()).isGreaterThan(BigDecimal.ZERO);
assertThat(result.getAmount()).isBetween(new BigDecimal("10"), new BigDecimal("1000"));

// Collections
assertThat(result.getLines()).hasSize(2);
assertThat(result.getLines()).extracting(OrderLine::getProductId).containsExactly(1L, 2L);
assertThat(result.getLines()).allSatisfy(line -> assertThat(line.getQuantity()).isPositive());

// Exceptions
assertThatThrownBy(() -> service.process(null))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("Request cannot be null");

assertThatCode(() -> service.process(validRequest))
    .doesNotThrowAnyException();

// Conditions composées
assertThat(order)
    .satisfies(o -> {
        assertThat(o.getId()).isNotNull();
        assertThat(o.getReference()).startsWith("ORD-");
    });
```

### Tests Paramétrés

```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "\t", "\n"})
void validate_WhenBlankEmail_ShouldThrow(String email) {
    assertThatThrownBy(() -> validator.validate(email))
        .isInstanceOf(InvalidEmailException.class);
}

@ParameterizedTest
@CsvSource({
    "100.00, SILVER, 5.0",
    "100.00, GOLD,   10.0",
    "100.00, PLATINUM, 20.0"
})
void calculateDiscount(BigDecimal amount, CustomerTier tier, double expectedDiscountPct) {
    double discount = discountService.calculate(amount, tier);
    assertThat(discount).isEqualTo(expectedDiscountPct);
}

@ParameterizedTest
@MethodSource("invalidRequests")
void createOrder_WhenInvalidRequest_ShouldThrow(CreateOrderRequest request, String expectedMessage) {
    assertThatThrownBy(() -> orderService.create(request))
        .hasMessageContaining(expectedMessage);
}

private static Stream<Arguments> invalidRequests() {
    return Stream.of(
        Arguments.of(null, "Request cannot be null"),
        Arguments.of(new CreateOrderRequest(null, List.of()), "Customer ID required"),
        Arguments.of(new CreateOrderRequest(1L, List.of()), "At least one line required")
    );
}
```

---

## 2. Mockito

```java
// Stubbing basique
when(repo.findById(1L)).thenReturn(Optional.of(order));
when(repo.save(any())).thenReturn(savedOrder);
when(service.calculate(any())).thenThrow(new ServiceException("timeout"));

// Argument Matchers
when(repo.findByStatus(eq(OrderStatus.PENDING))).thenReturn(pendingOrders);
when(repo.save(argThat(o -> o.getAmount().compareTo(BigDecimal.ZERO) > 0))).thenReturn(order);

// Vérification d'appels
verify(emailService, times(1)).send(any());
verify(emailService, never()).sendError(any());
verify(repo, atLeastOnce()).save(any());

// Capturer les arguments
ArgumentCaptor<Order> orderCaptor = ArgumentCaptor.forClass(Order.class);
verify(repo).save(orderCaptor.capture());
Order captured = orderCaptor.getValue();
assertThat(captured.getStatus()).isEqualTo(OrderStatus.CONFIRMED);

// Spy : objet réel + comportement partiel mocké
@Spy
private OrderValidator orderValidator = new OrderValidator();

doReturn(true).when(orderValidator).isValid(any()); // spy : override une méthode

// Mock statique (Mockito 3.4+)
try (MockedStatic<UuidGenerator> mocked = mockStatic(UuidGenerator.class)) {
    mocked.when(UuidGenerator::generate).thenReturn("fixed-uuid-123");
    Order order = orderService.create(request);
    assertThat(order.getReference()).isEqualTo("fixed-uuid-123");
}
```

---

## 3. Tests d'Intégration (Spring Boot Test)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@Transactional
class OrderControllerIntegrationTest {

    // Testcontainer : vraie BDD en Docker pendant les tests
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void createOrder_ShouldPersistAndReturn201() {
        CreateOrderRequest request = buildValidRequest();

        ResponseEntity<OrderDto> response = restTemplate.postForEntity(
            "/api/v1/orders", request, OrderDto.class
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(orderRepository.findById(response.getBody().getId())).isPresent();
    }
}

// MockMvc pour tests Web Layer uniquement (plus rapide)
@WebMvcTest(OrderController.class)
class OrderControllerWebTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    void getOrder_WhenExists_ShouldReturn200WithJson() throws Exception {
        when(orderService.findById(1L)).thenReturn(Optional.of(buildOrderDto()));

        mockMvc.perform(get("/api/v1/orders/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.status").value("PENDING"))
            .andExpect(jsonPath("$.lines").isArray())
            .andDo(print());
    }

    @Test
    void createOrder_WhenInvalidBody_ShouldReturn400() throws Exception {
        mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))  // corps invalide
            .andExpect(status().isBadRequest());
    }
}
```

---

## 4. BDD (Behavior-Driven Development)

### Cucumber + Spring Boot

```gherkin
# src/test/resources/features/order.feature
Feature: Order Management
  As a customer
  I want to create an order
  So that I can receive my products

  Scenario: Successfully create an order
    Given a customer with id 1 exists
    And the product with id 10 is in stock with quantity 5
    When I submit an order for 2 units of product 10
    Then the order should be created with status "PENDING"
    And an email confirmation should be sent to the customer

  Scenario Outline: Order fails with invalid data
    When I submit an order with <customerId> and <productId>
    Then the order should fail with error "<error>"

    Examples:
      | customerId | productId | error                    |
      | 999        | 10        | Customer not found: 999  |
      | 1          | 888       | Product not found: 888   |
      | 1          | 10        | Product out of stock     |
```

```java
@CucumberContextConfiguration
@SpringBootTest
public class CucumberSpringConfig {}

@Given("a customer with id {long} exists")
public void aCustomerExists(Long id) {
    customer = Customer.builder().id(id).name("Alice").email("alice@test.com").build();
    customerRepository.save(customer);
}

@When("I submit an order for {int} units of product {long}")
public void iSubmitOrder(int quantity, Long productId) {
    request = new CreateOrderRequest(customer.getId(), List.of(new OrderLineDto(productId, quantity)));
    try {
        createdOrder = orderService.create(request);
    } catch (Exception e) {
        thrownException = e;
    }
}

@Then("the order should be created with status {string}")
public void orderShouldHaveStatus(String status) {
    assertThat(createdOrder).isNotNull();
    assertThat(createdOrder.getStatus().name()).isEqualTo(status);
}
```

---

## 5. Pyramide des Tests

```
         /\
        /  \
       / E2E\           Peu nombreux, lents, coûteux
      /------\
     /        \
    /Integration\       Moyennement nombreux, réalistes
   /------------\
  /              \
 /   Unit Tests   \     Nombreux, rapides, isolés
/------------------\
```

| Type | Outil | Vitesse | Couverture |
|------|-------|---------|------------|
| **Unit** | JUnit 5 + Mockito | < 1ms | Logique métier isolée |
| **Slice** | `@WebMvcTest`, `@DataJpaTest` | < 1s | Couche spécifique |
| **Integration** | `@SpringBootTest` + Testcontainers | 5-30s | Flux end-to-end |
| **BDD** | Cucumber | 5-60s | Scénarios métier |

---

## Questions d'Interview — Tests

---

### Q1 : Quelle est la différence entre Mock, Stub et Spy ?

**Réponse :**

| Type | Description | Usage |
|------|-------------|-------|
| **Stub** | Remplace une dépendance avec des réponses prédéfinies | Simuler des données retournées |
| **Mock** | Stub + vérifie les interactions (appels) | Vérifier qu'une méthode est bien appelée |
| **Spy** | Objet réel dont certaines méthodes sont remplacées | Tester la vraie implémentation mais contrôler certains comportements |

```java
// Stub
when(repo.findById(1L)).thenReturn(Optional.of(order));

// Mock (stub + verify)
when(emailService.send(any())).thenReturn(true);
// ... exécution ...
verify(emailService, times(1)).send(argThat(e -> e.getTo().equals("alice@test.com")));

// Spy
OrderService realService = new OrderService(realRepo, realEmail);
OrderService spy = spy(realService);
doReturn(mockOrder).when(spy).fetchOrder(anyLong()); // overrides only this method
```

---

### Q2 : Comment tester du code asynchrone avec JUnit ?

**Réponse :**

```java
// CompletableFuture
@Test
void asyncMethod_ShouldComplete() throws ExecutionException, InterruptedException {
    CompletableFuture<String> future = service.processAsync("input");
    String result = future.get(5, TimeUnit.SECONDS); // timeout
    assertThat(result).isEqualTo("expected");
}

// Awaitility : polling jusqu'à condition
@Test
void eventDriven_ShouldUpdateStatus() {
    orderService.processOrder(orderId);

    await()
        .atMost(5, SECONDS)
        .pollInterval(100, MILLISECONDS)
        .untilAsserted(() ->
            assertThat(orderRepo.findById(orderId))
                .hasValueSatisfying(o -> assertThat(o.getStatus()).isEqualTo(CONFIRMED))
        );
}
```

---

### Q3 : Qu'est-ce que le TDD et comment l'appliquer ?

**Réponse :**

TDD = **Red → Green → Refactor** :
1. **Red** : écrire un test qui échoue (pas encore d'implémentation)
2. **Green** : écrire le minimum de code pour faire passer le test
3. **Refactor** : améliorer le code sans casser les tests

```java
// Red : le test échoue (OrderService n'existe pas encore)
@Test
void calculateTotal_ShouldSumLinesWithTax() {
    Order order = Order.builder()
        .line(new OrderLine(product(10.0), 2))
        .line(new OrderLine(product(5.0), 1))
        .taxRate(0.20)
        .build();

    BigDecimal total = order.calculateTotal();

    assertThat(total).isEqualByComparingTo("30.00"); // (20 + 5) * 1.20 = 30
}

// Green : implémentation minimale
public BigDecimal calculateTotal() {
    BigDecimal subtotal = lines.stream()
        .map(l -> l.getProduct().getPrice().multiply(BigDecimal.valueOf(l.getQuantity())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    return subtotal.multiply(BigDecimal.ONE.add(taxRate));
}

// Refactor : extraire méthodes, améliorer lisibilité...
```

---

## Pièges Classiques

- Tester l'implémentation plutôt que le comportement → tests fragiles, cassants à chaque refactor
- `@SpringBootTest` pour tester la logique métier → trop lent, utiliser des tests unitaires
- `verify()` sans `when()` → test qui vérifie que rien ne se passe (peut toujours passer)
- Shared state entre tests (`@BeforeAll` sans `static`, champs non réinitialisés) → tests interdépendants
- Mock tout → aucune confiance dans l'intégration réelle des composants
