# Product Service (Port: 8080)

## Overview
This is a Spring Boot project that provides product-related functionalities. It is configured as a Eureka client for service discovery and uses a centralized config server for configuration management.

## Dependencies
- Spring Web
- Spring Data JPA
- MySQL Driver
- Lombok
- Cloud Bootstrap
- Spring Boot DevTools
- Eureka Client
- Config Client

## Folder Structure
```
- controller
- services
- repository
- entity
- model
- exceptions
```

## Configuration
Update `application.properties` to `application.yaml`.

## Flow
1. **Model**
   - `ProductRequest`: Handles incoming product details.
   - `ProductResponse`: Returns product details from the controller.
   - `ErrorResponse`: Standard error response structure (errorMessage, errorCode).
2. **Repository**
   - `ProductRepository`: Extends `JpaRepository` for database operations.
3. **Entity**
   - `Product`: Represents the database entity for products.
4. **Services**
   - `ProductService`: Interface defining product operations.
   - `ProductServiceImpl`: Implements `ProductService` logic.
5. **Exceptions**
   - `ProductServiceCustomException`: Extends `RuntimeException`.
   - `RestResponseEntityExceptionHandler`: Handles global exceptions using `@ControllerAdvice`.

## Controller Example
```java
@PostMapping
public ResponseEntity<Long> addProduct(@RequestBody ProductRequest productRequest) {
    long productId = productService.addProduct(productRequest);
    return new ResponseEntity<>(productId, HttpStatus.CREATED);
}
```

## Exception Handling
```java
public class ProductServiceCustomException extends RuntimeException {
    private String errorCode;
    public ProductServiceCustomException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
}
```

## Controller Advice Example
```java
@ControllerAdvice
public class RestResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {
    @ExceptionHandler(ProductServiceCustomException.class)
    public ResponseEntity<ErrorResponse> handleProductServiceException(ProductServiceCustomException exception) {
        return new ResponseEntity<>(new ErrorResponse(exception.getMessage(),exception.getErrorCode()), HttpStatus.NOT_FOUND);
    }
}
```

# Service Registry (Eureka Server) (Port: 8761)
This is a separate Spring Boot project configured as a Eureka Server.

## Dependencies
- Cloud Bootstrap
- Eureka Server

## Configuration (`application.yaml`)
```yaml
spring:
  application:
    name: ServiceRegistry
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
```

## Application Class
```java
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceRegistryApplication.class, args);
    }
}
```

# Order Service (Port: 8082)

## Overview
This service handles order-related functionalities and interacts with Product and Payment services.

## Dependencies
- Spring Web
- Spring Data JPA
- MySQL Driver
- Lombok
- Cloud Bootstrap
- Spring Boot DevTools
- Eureka Client
- Config Client

## Folder Structure
```
- controller
- services
- repository
- entity
- model
- exceptions
```

## Controller Example
```java
@PostMapping("/placeOrder")
public ResponseEntity<Long> placeOrder(@RequestBody OrderRequest orderRequest) {
    long orderId = orderService.placeOrder(orderRequest);
    return new ResponseEntity<>(orderId, HttpStatus.OK);
}
```

## Service Implementation Example
```java
@Override
public long placeOrder(OrderRequest orderRequest) {
    Order order = new Order(orderRequest.getProductId(), orderRequest.getQuantity(), Instant.now(), "CREATED", orderRequest.getTotalAmount());
    order = orderRepository.save(order);
    return order.getId();
}
```

# Config Server (Port: 9296)

## Overview
The Config Server fetches configuration from a GitHub repository and provides it to microservices dynamically.

## Dependencies
- Config Server
- Eureka Client

## Application Class
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

## Configuration (`application.yaml`)
```yaml
server:
  port: 9296
spring:
  application:
    name: CONFIG-SERVER
  cloud:
    config:
      server:
        git:
          uri: https://github.com/ParagShah97/InventraXConfigData
          clone-on-start: true
eureka:
  instance:
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: ${EUREKA_SERVER_ADDRESS:http://localhost:8761/eureka}
```

# Notes
- The Eureka Client configuration is common for both Product and Order services.
- The Config Server eliminates redundant configuration copies by centrally managing properties.
- All services should have the following configuration to integrate with the Config Server:
```yaml
spring:
  config:
    import: configserver:http://localhost:9296
```
- Ensure that the Git repository mentioned in the `application.yaml` file contains valid configuration files for different services.
