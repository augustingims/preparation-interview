# 02 — Spring Boot (Data, Web, Security)

## 1. IoC & Injection de Dépendances

### Concepts Clés

Le conteneur Spring (ApplicationContext) gère le cycle de vie des **Beans** et injecte les dépendances.

```java
// Types d'injection (préférer le constructeur)
@Service
public class OrderService {

    // 1. Injection par constructeur (RECOMMANDÉE)
    private final OrderRepository repo;
    private final EmailService emailService;

    public OrderService(OrderRepository repo, EmailService emailService) {
        this.repo = repo;
        this.emailService = emailService;
    }

    // 2. Injection par setter (optionnelle)
    @Autowired
    public void setEmailService(EmailService service) { ... }

    // 3. Injection par champ (déconseillée — difficile à tester)
    @Autowired
    private PaymentService paymentService;
}
```

**Pourquoi le constructeur ?**
- Dépendances **obligatoires** visibles
- Immutabilité (`final`)
- Testable sans Spring (nouveau avec `new OrderService(mockRepo, mockEmail)`)

### Scopes des Beans

| Scope | Description | Usage |
|-------|-------------|-------|
| `singleton` | Une instance par contexte (défaut) | Services, Repositories |
| `prototype` | Nouvelle instance à chaque demande | Objets avec état |
| `request` | Une instance par requête HTTP | Web uniquement |
| `session` | Une instance par session HTTP | Web uniquement |

```java
@Bean
@Scope("prototype")
public ReportBuilder reportBuilder() {
    return new ReportBuilder();
}
```

---

## 2. Spring Web MVC

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService service;

    public OrderController(OrderService service) {
        this.service = service;
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getById(@PathVariable Long id) {
        return service.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderDto create(@Valid @RequestBody CreateOrderRequest request) {
        return service.create(request);
    }

    @GetMapping
    public Page<OrderDto> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy
    ) {
        return service.findAll(PageRequest.of(page, size, Sort.by(sortBy)));
    }
}

// Gestion centralisée des erreurs
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(EntityNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return new ErrorResponse("VALIDATION_ERROR", errors.toString());
    }
}
```

---

## 3. Spring Data JPA

```java
// Entity
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String reference;

    @ManyToOne(fetch = FetchType.LAZY)  // LAZY par défaut = bonne pratique
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();
}

// Repository
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Méthode dérivée
    List<Order> findByCustomerIdAndStatusOrderByCreatedAtDesc(Long customerId, Status status);

    // JPQL
    @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
    Optional<Order> findByIdWithLines(@Param("id") Long id);

    // SQL natif
    @Query(value = "SELECT * FROM orders WHERE created_at > :date", nativeQuery = true)
    List<Order> findRecentOrders(@Param("date") LocalDateTime date);

    // Projection
    @Query("SELECT o.id as id, o.reference as reference FROM Order o")
    List<OrderSummary> findAllSummaries();
}

// Interface de projection
public interface OrderSummary {
    Long getId();
    String getReference();
}
```

### N+1 Problem

```java
// PROBLÈME N+1
List<Order> orders = orderRepo.findAll();
orders.forEach(o -> o.getCustomer().getName()); // 1 query + N queries customer

// SOLUTION 1 : JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();

// SOLUTION 2 : EntityGraph
@EntityGraph(attributePaths = {"customer", "lines"})
List<Order> findAll();

// SOLUTION 3 : Batch Size (pour collections)
@BatchSize(size = 30)
@OneToMany(mappedBy = "order")
private List<OrderLine> lines;
```

---

## 4. Spring Security

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // API REST stateless
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/orders/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // coût par défaut = 10
    }
}

// Filtre JWT
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }
        String token = header.substring(7);
        if (jwtService.isValid(token)) {
            UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
                jwtService.extractUsername(token), null, jwtService.extractAuthorities(token)
            );
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(request, response);
    }
}
```

---

## 5. Configuration & Profiles

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-server/mydb
    username: ${DB_USER}     # variable d'environnement
    password: ${DB_PASS}
```

```java
@Component
@Profile("prod")
public class ProdEmailService implements EmailService { ... }

@Component
@Profile("dev")
public class MockEmailService implements EmailService { ... }
```

---

## Questions d'Interview — Spring Boot

---

### Q1 : Qu'est-ce que `@SpringBootApplication` ?

**Réponse :**

C'est une annotation composite qui regroupe :
- `@Configuration` : classe de configuration Spring
- `@EnableAutoConfiguration` : active la configuration automatique basée sur le classpath
- `@ComponentScan` : scan les composants dans le package courant et sous-packages

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
// Équivaut à :
@Configuration
@EnableAutoConfiguration
@ComponentScan(basePackages = "com.example")
public class MyApplication { ... }
```

L'**auto-configuration** détecte les dépendances dans le classpath (ex: si `spring-boot-starter-web` est présent → configure Tomcat, DispatcherServlet, Jackson automatiquement).

---

### Q2 : Quelle est la différence entre `@Controller` et `@RestController` ?

**Réponse :**

- `@Controller` : retourne des **vues** (templates Thymeleaf, JSP). La valeur de retour est le nom d'une vue résolue par le `ViewResolver`.
- `@RestController` = `@Controller` + `@ResponseBody` : retourne des **données** (JSON/XML) sérialisées directement dans le corps de la réponse HTTP.

```java
@Controller
public class PageController {
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("user", currentUser);
        return "home"; // → templates/home.html
    }
}

@RestController
public class ApiController {
    @GetMapping("/api/users")
    public List<UserDto> getUsers() {
        return userService.findAll(); // → JSON automatiquement
    }
}
```

---

### Q3 : Expliquez la gestion des transactions avec `@Transactional`.

**Réponse :**

Spring crée un **proxy AOP** autour de la méthode annotée pour gérer la transaction automatiquement.

```java
@Service
public class TransferService {

    @Transactional  // commit si succès, rollback si RuntimeException
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepo.findById(fromId).orElseThrow();
        Account to = accountRepo.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        accountRepo.save(from);
        accountRepo.save(to);
        // Si RuntimeException ici → rollback des deux opérations
    }
}
```

**Propagation :**

| Propagation | Comportement |
|-------------|-------------|
| `REQUIRED` (défaut) | Réutilise la tx existante ou en crée une |
| `REQUIRES_NEW` | Suspend la tx courante, crée une nouvelle |
| `SUPPORTS` | Tx si existante, sinon sans |
| `NOT_SUPPORTED` | Toujours sans transaction |
| `NEVER` | Erreur si tx existante |
| `MANDATORY` | Erreur si pas de tx existante |

**Pièges :**
1. `@Transactional` sur méthode `private` → ignoré (proxy AOP ne peut pas intercepter)
2. Appel interne dans la même classe → proxy contourné → `@Transactional` ignoré
3. Rollback par défaut seulement sur `RuntimeException` et `Error` → préciser `rollbackFor = Exception.class` pour les checked exceptions

---

### Q4 : Comment implémenter la pagination en Spring Data ?

**Réponse :**

```java
// Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByCategory(String category, Pageable pageable);
}

// Controller
@GetMapping("/products")
public Page<ProductDto> getProducts(
    @RequestParam String category,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size
) {
    Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
    return productRepo.findByCategory(category, pageable)
        .map(productMapper::toDto);
}

// Réponse JSON automatique
{
    "content": [...],
    "totalElements": 150,
    "totalPages": 8,
    "number": 0,
    "size": 20,
    "first": true,
    "last": false
}
```

---

### Q5 : Expliquez le mécanisme de cache avec Spring Cache.

**Réponse :**

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return new CaffeineCacheManager("products", "orders");
    }
}

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return repo.findById(id).orElseThrow(); // exécuté seulement si pas en cache
    }

    @CacheEvict(value = "products", key = "#product.id")
    public Product update(Product product) {
        return repo.save(product); // invalide le cache après update
    }

    @CachePut(value = "products", key = "#result.id")
    public Product create(Product product) {
        return repo.save(product); // met à jour le cache avec le résultat
    }
}
```

---

### Q6 : Comment sécuriser une API REST avec JWT ?

**Réponse :**

Le flux complet :

```
Client → POST /auth/login {username, password}
       ← 200 OK {accessToken, refreshToken}

Client → GET /api/orders [Authorization: Bearer <accessToken>]
       → JwtAuthFilter valide le token
       → SecurityContext peuplé
       ← 200 OK [données]
```

**Points clés :**
- Le token JWT est **stateless** : pas de session côté serveur
- Signature avec `HS256` (secret partagé) ou `RS256` (clé privée/publique)
- **Access token** : courte durée (15min-1h)
- **Refresh token** : longue durée (7-30j), stocké en base pour révocation

```java
// Génération du token
public String generateToken(UserDetails user) {
    return Jwts.builder()
        .subject(user.getUsername())
        .claim("roles", user.getAuthorities())
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + 3600_000))
        .signWith(getSigningKey())
        .compact();
}
```

---

## Pièges Classiques

- `@Transactional` sur méthode privée ou appel self-invocation → pas de transaction
- Lazy loading hors transaction → `LazyInitializationException` → utiliser DTOs ou `JOIN FETCH`
- `@Autowired` sur champ statique → ne fonctionne pas
- Utiliser `FetchType.EAGER` sur les collections → toujours chargé même si inutile → problèmes de perf
- Oublier `@EnableWebSecurity` ou avoir deux `SecurityFilterChain` en conflit
