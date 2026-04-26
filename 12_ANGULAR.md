# 12 — Angular (Perspective Back-End)

> Ce module couvre les notions clés qu'un Référent Technique Java doit connaître pour collaborer efficacement avec des équipes front-end Angular.

## 1. Architecture Angular

```
src/app/
├── core/              # Services singleton (auth, http interceptors)
│   ├── auth.service.ts
│   └── http-error.interceptor.ts
├── shared/            # Composants, pipes, directives réutilisables
│   └── components/
├── features/          # Modules fonctionnels
│   ├── orders/
│   │   ├── order-list.component.ts
│   │   ├── order.service.ts
│   │   ├── order.model.ts
│   │   └── orders.module.ts
│   └── customers/
└── app.module.ts
```

---

## 2. Concepts Clés

### Composants & Cycle de Vie
```typescript
@Component({
  selector: 'app-order-list',
  templateUrl: './order-list.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush  // optimisation
})
export class OrderListComponent implements OnInit, OnDestroy {
  orders$: Observable<Order[]>;
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.orders$ = this.orderService.getAll().pipe(
      takeUntil(this.destroy$)  // évite les memory leaks
    );
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Services & Injection
```typescript
@Injectable({ providedIn: 'root' })  // singleton dans toute l'app
export class OrderService {
  private apiUrl = environment.apiUrl + '/orders';

  constructor(private http: HttpClient) {}

  getAll(): Observable<Order[]> {
    return this.http.get<Order[]>(this.apiUrl).pipe(
      catchError(this.handleError)
    );
  }

  create(request: CreateOrderRequest): Observable<Order> {
    return this.http.post<Order>(this.apiUrl, request);
  }
}
```

### RxJS — Opérateurs Essentiels
```typescript
// switchMap : annule la requête précédente (ex: recherche en temps réel)
searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.productService.search(term))
);

// combineLatest : combiner plusieurs observables
combineLatest([user$, permissions$]).pipe(
  map(([user, perms]) => ({ ...user, permissions: perms }))
);

// forkJoin : attendre que tous les observables se complètent (équiv. Promise.all)
forkJoin({
  orders: this.orderService.getAll(),
  customers: this.customerService.getAll()
}).subscribe(({ orders, customers }) => { ... });
```

### HTTP Interceptors (Auth, Error Handling)
```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    if (token) {
      req = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      });
    }
    return next.handle(req).pipe(
      catchError(err => {
        if (err.status === 401) {
          this.authService.logout();
          this.router.navigate(['/login']);
        }
        return throwError(() => err);
      })
    );
  }
}
```

---

## 3. Interface Back-End / Front-End

### CORS (à configurer côté Spring Boot)
```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:4200", "https://app.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

### Contrat API (OpenAPI / Swagger)
```java
// Spring Boot génère la spec OpenAPI 3.0
// Angular peut générer le client HTTP depuis la spec
// openapi-generator-cli generate -i api-spec.json -g typescript-angular

@Operation(summary = "Create order", description = "Creates a new order for the authenticated customer")
@ApiResponse(responseCode = "201", description = "Order created")
@ApiResponse(responseCode = "400", description = "Invalid request body")
@PostMapping
public ResponseEntity<OrderDto> create(@Valid @RequestBody CreateOrderRequest request) { ... }
```

---

## Questions d'Interview — Angular

---

### Q1 : Comment Angular communique-t-il avec une API REST Spring Boot ?

**Réponse :**

Via `HttpClient` qui retourne des `Observable<T>`. Le back-end doit configurer CORS pour autoriser l'origine Angular. En production, l'API Gateway ou un reverse proxy gère souvent le CORS.

Points d'attention côté back-end :
- **CORS** : origines autorisées, méthodes, headers
- **JWT** : Angular envoie le token dans `Authorization: Bearer <token>`
- **Format des erreurs** : structurer les réponses d'erreur de manière cohérente
- **Pagination** : retourner `Page<T>` avec métadonnées (total, pages)

---

### Q2 : Qu'est-ce que le lazy loading en Angular ?

**Réponse :**

Chargement différé des modules Angular — le code d'un module n'est téléchargé que lorsque l'utilisateur navigue vers la route correspondante.

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: 'orders', loadChildren: () =>
      import('./features/orders/orders.module').then(m => m.OrdersModule) },
  { path: 'admin', loadChildren: () =>
      import('./features/admin/admin.module').then(m => m.AdminModule),
    canLoad: [AdminGuard] }  // vérification des droits avant chargement
];
```

**Impact back-end :** réduire la taille des bundles réduit le TTFB et améliore les Core Web Vitals.

---

## Pièges Classiques

- Retourner des objets Java avec des noms en `camelCase` → Angular attend la même chose (cohérence avec Jackson `@JsonProperty`)
- API qui retourne `200 OK` pour les erreurs métier → Angular ne peut pas distinguer succès/erreur → utiliser les bons codes HTTP
- Champs de date en `LocalDateTime` sans timezone → sérialiser en ISO 8601 avec `ZonedDateTime` ou configurer Jackson
- Ne pas paginer les endpoints retournant des listes → Angular charge tout en mémoire
