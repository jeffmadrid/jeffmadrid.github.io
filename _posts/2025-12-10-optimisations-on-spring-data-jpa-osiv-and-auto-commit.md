---
title: "Optimisations on Spring Data JPA - OSIV and auto-commit"
date: 2025-12-10
draft: false
---

# Optimisations on Spring Data JPA - OSIV and auto-commit

## TL;DR - The Two Optimisations

Add these to your `application.properties`:

```properties
# Optimisation 1: Disable Open-In-View
spring.jpa.open-in-view=false

# Optimisation 2: Disable HikariCP Auto-Commit
spring.datasource.hikari.auto-commit=false
```
<br>

That's it! More on details below ;)

---

## ToC

1. [The Problem with Spring Boot defaults](#the-problem-with-spring-boot-defaults)
2. [Optimisation 1: Disabling Open-In-View](#optimisation-1-disabling-open-in-view-osiv)
3. [Optimisation 2: Disabling Auto-Commit](#optimisation-2-disabling-hikaricp-auto-commit)
4. [Best Practices](#best-practices)

<br>

---

## The Problem with Spring Boot defaults

Spring Boot prioritizes developer convenience with sensible defaults. However, two particular defaults can cause performance issues in production:

| Setting | Default | Problem |
|---------|---------|---------|
| `spring.jpa.open-in-view` | `true` | Database connections held for entire HTTP request |
| `spring.datasource.hikari.auto-commit` | `true` | Each SQL statement treated as separate transaction |

---

## Optimisation 1: Disabling Open-In-View (OSIV)

### What is Open-In-View?

Open Session In View (OSIV) is a pattern that keeps the Hibernate Session (and thus the database connection) open for the entire HTTP request lifecycle.

### The Timeline Comparison

<br>

**With `open-in-view=true` (Default):**

```
HTTP Request Start
    ↓
    │ Connection acquired from pool
    ↓
Controller receives request
    ↓
Service layer (@Transactional)
    ↓
Repository query
    ↓
Service returns
    ↓
Controller processes
    ↓
JSON Serialization ← Connection STILL held here!
    ↓
    │ Connection returned to pool
    ↓
HTTP Response Sent
```

<br>
**With `open-in-view=false` (Optimized):**

```
HTTP Request Start
    ↓
Controller receives request
    ↓
    │ Connection acquired from pool
    ↓
Service layer (@Transactional)
    ↓
Repository query
    ↓
    │ Connection returned to pool ← Much earlier!
    ↓
Service returns
    ↓
Controller processes
    ↓
JSON Serialization
    ↓
HTTP Response Sent
```
<br>

### Why This Matters

1. **Connection Pool Exhaustion**: With 10 connections and requests taking 500ms to serialize, you can only handle 20 requests/second before connections run out.

2. **Hidden N+1 Problems**: OSIV allows lazy loading in controllers/views, which can silently cause N+1 queries during JSON serialization.

3. **Unpredictable Performance**: Lazy loading happening outside your transactional service layer makes performance unpredictable.

### The Trade-off

With `open-in-view=false`, you'll get `LazyInitializationException` if you try to access lazy-loaded data outside a transaction. This is **actually good** because:

- Problems become visible immediately during development
- Forces proper data loading patterns (JOIN FETCH, DTOs)
- No hidden N+1 queries in production

### How to Fix LazyInitializationException
<br>

**Before (works with OSIV, but hides N+1):**
<br>

```java
// Repository
Optional<Product> findById(Long id);

// Controller - silently triggers extra query!
Product product = productService.findById(id);
String categoryName = product.getCategory().getName();
```
<br>

**After (proper JOIN FETCH):**
```java
// Repository
@Query("SELECT p FROM Product p JOIN FETCH p.category WHERE p.id = :id")
Product findByIdWithCategory(Long id);

// Controller - no extra query!
Product product = productService.findByIdWithCategory(id);
String categoryName = product.getCategory().getName();
```

---

## Optimisation 2: Disabling HikariCP Auto-Commit

### What is Auto-Commit?

When `auto-commit=true`, every SQL statement is immediately committed to the database as its own transaction.

### The Impact
<br>

**With `auto-commit=true` (Default):**

```
saveAll(100 products)
    ↓
INSERT product1 → COMMIT
INSERT product2 → COMMIT
INSERT product3 → COMMIT
...100 times...
```
Result: 100 round-trips to the database!

<br>

**With `auto-commit=false` (Optimized):**
```
saveAll(100 products)
    ↓
BEGIN TRANSACTION
INSERT product1
INSERT product2
INSERT product3
...100 inserts batched...
COMMIT
```
Result: Much fewer round-trips, Hibernate batching enabled!

### Why This Matters

1. **Transaction Overhead**: Each commit has overhead (network round-trip, transaction log flush)

2. **Batching Disabled**: Auto-commit prevents Hibernate's JDBC batching from working effectively

3. **Read-Only Optimisation Lost**: With auto-commit=true, `@Transactional(readOnly=true)` can't optimize reads properly

### Enabling Full Batching

With `auto-commit=false`, you can also enable Hibernate batching:

```properties
spring.datasource.hikari.auto-commit=false
spring.jpa.properties.hibernate.jdbc.batch_size=25
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

---

## Best Practices

### 1. Always Use @Transactional Properly

```java
@Service
public class ProductService {

    @Transactional(readOnly = true)  // For reads
    public List<Product> findAll() {
        return productRepository.findAll();
    }

    @Transactional  // For writes
    public Product save(Product product) {
        return productRepository.save(product);
    }
}
```

### 2. Use JOIN FETCH for Related Data

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p JOIN FETCH p.category")
    List<Product> findAllWithCategory();
}
```

### 3. Consider DTOs for Complex Responses

```java
@Query("""
    SELECT new com.example.dto.ProductDTO(p.id, p.name, c.name)
    FROM Product p JOIN p.category c
    """)
List<ProductDTO> findAllAsDTO();
```

### 4. Enable Hibernate Statistics in Development

```properties
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```

---

## References

- [Spring Boot Documentation - JPA Properties](https://docs.spring.io/spring-boot/reference/data/sql.html#data.sql.jpa-and-spring-data.open-entity-manager-in-view)
- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)
- [Hibernate Batching](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#batch)
- [Vlad Mihalcea - The Open Session in View Anti-Pattern](https://vladmihalcea.com/the-open-session-in-view-anti-pattern/)
