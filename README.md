# Spring Contract Engine (SCE) 

Contract-first Spring Boot framework. Write business logic. Never write controllers, DTOs, or HTTP clients again.

SCE combines OpenAPI-driven code generation with annotation-based binding to eliminate boilerplate. The developer experience is deliberately Lombok-like: no generated code in `src/`, no visible wiring, fail-fast startup validation.

## Quick start
 
### 1. Add the dependency and plugin
 
```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.spring-contract-engine</groupId>
    <artifactId>sce-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
 
<plugin>
    <groupId>io.spring-contract-engine</groupId>
    <artifactId>sce-codegen-maven-plugin</artifactId>
    <version>1.0.0</version>
    <executions>
        <execution>
            <goals><goal>generate</goal></goals>
        </execution>
    </executions>
</plugin>
```

### 2. Write your OpenAPI specs
 
```
my-service/
└── contracts/
    ├── outbound/            # API this service exposes
    │   └── order-service.yaml
    └── inbound/             # APIs this service consumes
        └── payment-service.yaml
```

### 3. Configure SCE
 
```yaml
# application.yml
sce:
  base-package: com.example.generated
  client:
    timeout: 2s
    retry:
      attempts: 3
```

### 4. Write only business logic
 
```java
@ContractService("OrderApi")
@Service
public class OrderService {
 
    private final PaymentApiClient paymentClient; // auto-generated, just @Autowired
 
    @ContractOperation("createOrder")
    public ResponseEntity<OrderResponse> create(@RequestBody @Valid CreateOrderRequest req) {
        var payment = paymentClient.processPayment(toPaymentRequest(req)).block();
        return ResponseEntity.status(201).body(buildResponse(req, payment));
    }
 
    @ContractOperation("getOrderById")
    public ResponseEntity<OrderResponse> getById(@PathVariable String orderId) {
        return ResponseEntity.ok(findOrder(orderId));
    }
}
```
 
That's it. SCE generates the controller, DTOs, and payment client at build time.
## How it works
 
### Build time — `sce-codegen-maven-plugin`
 
Runs in `generate-sources`. For each YAML in `contracts/outbound/`, generates a Spring MVC `@RequestMapping` **interface** (not a class) plus DTOs. For each YAML in `contracts/inbound/`, generates a `WebClient`-based API client.
 
All output lands in `target/generated-sources/sce` — inspectable, type-checked, never committed.
 
```
target/generated-sources/sce/
└── com/example/generated/
    ├── api/
    │   ├── OrderApi.java          ← Spring MVC interface
    │   └── PaymentApiClient.java  ← WebClient client
    └── model/
        ├── CreateOrderRequest.java
        ├── OrderResponse.java
        └── PaymentRequest.java
```
 
### Runtime — `sce-runtime-binder`
 
`ContractBindingProcessor` implements `BeanDefinitionRegistryPostProcessor`. Spring calls it after all bean definitions are loaded but before any bean is instantiated — the correct hook for adding new bean definitions.
 
For each `@ContractService` bean:
 
1. Resolves the generated interface from the classpath via `{basePackage}.api.{ApiName}`.
2. Builds a `MethodMapping` — each `@ContractOperation` value matched to a method on the interface.
3. Validates at startup: missing operation or incompatible return type → `ContractBindingException` on boot.
4. Registers a `GenericBeanDefinition` backed by a `ContractControllerProxyFactoryBean`.
The factory bean creates a JDK dynamic proxy implementing the generated interface. Spring MVC discovers it, reads the `@PostMapping`/`@GetMapping` annotations, and registers HTTP endpoints normally.
 
### Request flow
 
```
HTTP POST /orders
  → Spring DispatcherServlet
  → ContractControllerProxy (JDK proxy)
  → InvocationHandler.invoke()
  → LoggingContractInterceptor.beforeInvocation()
  → OrderService.create(request)          ← your code
  → LoggingContractInterceptor.afterSuccess()
  → ResponseEntity<OrderResponse>
  → Spring HttpMessageConverter → JSON
```
 
---
 
## Configuration reference
 
| Property | Default | Description |
|---|---|---|
| `sce.spec-root` | `./contracts` | Root directory for YAML specs |
| `sce.inbound` | `inbound` | Subdirectory for consumer specs |
| `sce.outbound` | `outbound` | Subdirectory for provider specs |
| `sce.base-package` | `{groupId}.generated` | Package for generated classes |
| `sce.debug` | `false` | Verbose binding + interceptor logs |
| `sce.client.timeout` | `2s` | Per-request timeout for outbound calls |
| `sce.client.retry.attempts` | `3` | Retry count for outbound calls |
| `sce.client.retry.backoff` | `500ms` | Initial exponential back-off |
 
---
 
## Modules
 
| Module | Purpose |
|---|---|
| `sce-core` | `@ContractService`, `@ContractOperation`, `ContractInvocationInterceptor` SPI. Zero Spring dependencies. |
| `sce-codegen-maven-plugin` | Maven plugin wrapping openapi-generator. Runs in `generate-sources`. |
| `sce-runtime-binder` | `ContractBindingProcessor`, `ContractControllerProxy`, `MethodMapping`. Plain library — no auto-config. |
| `sce-spring-boot-starter` | Auto-configuration, `WebClient.Builder`, `SceProperties`, built-in logging interceptor. |
 
---
 
## Annotations
 
### `@ContractService`
 
Applied to a Spring `@Service` class. Binds the class to a generated API interface.
 
```java
// By interface name (resolved via base-package convention)
@ContractService("OrderApi")
 
// By direct class reference (compile-time safe)
@ContractService(api = OrderApi.class)
```
 
### `@ContractOperation`
 
Applied to a method. Maps it to an `operationId` in the OpenAPI spec.
 
```java
@ContractOperation("createOrder")
public ResponseEntity<OrderResponse> create(OrderRequest req) { ... }
```
 
The value must exactly match the `operationId` in the YAML. SCE validates every interface method has a corresponding `@ContractOperation` at startup — missing one throws `ContractBindingException`.
 
---
 
## Extensibility
 
### Custom invocation interceptor
 
```java
@Bean
public ContractInvocationInterceptor metricsInterceptor(MeterRegistry registry) {
    return new ContractInvocationInterceptor() {
        @Override
        public void beforeInvocation(Method iface, Method svc, Object[] args) {
            registry.counter("sce.invocations", "operation", iface.getName()).increment();
        }
    };
}
```
 
### Custom WebClient configuration
 
```java
@Bean("sceWebClientBuilder")  // overrides SCE's default
public WebClient.Builder customWebClientBuilder() {
    return WebClient.builder()
        .defaultHeader("Authorization", "Bearer " + token)
        .filter(myAuthFilter());
}
```
 
---
 
## Startup log
 
At `INFO` level, SCE prints a binding summary on every boot:
 
```
SCE  Binding summary (2 contracts bound):
SCE  [outbound] OrderApi → OrderService
SCE    createOrder          POST /orders              → create(CreateOrderRequest)
SCE    getOrderById         GET  /orders/{orderId}    → getById(String)
SCE    updateOrderStatus    PATCH /orders/{orderId}   → updateStatus(String, UpdateOrderStatusRequest)
SCE  [inbound]  PaymentApiClient  → http://payment-service (retry=3, timeout=2s)
```
 
Set `sce.debug=true` to see full parameter type maps and interceptor chain details.
 
---
 
## .gitignore
 
```gitignore
target/generated-sources/sce/
```
 
Generated sources are rebuilt on every `mvn generate-sources`. They are always inspectable in `target/` but never committed.
 


 
