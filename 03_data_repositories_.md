# Chapter 3: Data Repositories

Welcome back, future online store moguls! In our journey so far, we've laid down the groundwork:
*   In [Chapter 1: Data Models (Entities)](01_data_models__entities__.md), we learned about the blueprints for our store's information, like `ProductInfo` and `User`. These are the Java classes that automatically become tables in our database.
*   In [Chapter 2: Business Services](02_business_services_.md), we explored the "brains" of our application. These services, like `ProductService` and `CartService`, contain all the crucial business logic and rules, such as `decreaseStock()` for products or `checkout()` for carts.

But there's still a missing piece! Our [Business Services](02_business_services_.md) know *what* needs to be done with the data (e.g., "find this product," "save this new order"). However, they don't know *how* to actually talk to the database to perform these operations. They don't write complicated database queries (like SQL).

### What Problem Do Data Repositories Solve?

Imagine our business services are managers. They give instructions: "Go get me the details of product X," or "Update the stock for product Y." But they don't go to the warehouse (the database) themselves. They need a dedicated team to handle that.

This is where **Data Repositories** come in!

Think of **Data Repositories** as the **specialized librarians** or **record keepers** for our application. Each repository is responsible for interacting with the database to store, retrieve, update, or delete information for a *specific type* of data.

For example:
*   `UserRepository` is the "librarian" just for `User` data.
*   `ProductInfoRepository` is the "librarian" just for `ProductInfo` data.
*   `OrderRepository` handles `OrderMain` data.

Their job is to provide a standard, easy-to-use set of methods (like `save`, `findById`, `findAll`) to perform common database operations. This allows our [Business Services](02_business_services_.md) to interact with the database without needing to know any complex database queries. It's like asking the librarian for a book by title, instead of having to search the entire library yourself.

### What are Data Repositories in Our Project?

In our Spring Boot application, Data Repositories are typically **Java Interfaces**. What's truly amazing is that we don't even need to write the code for *most* of the common database operations! Spring Data JPA (which works with Hibernate) does the heavy lifting for us.

Let's look at a typical Data Repository interface:

```java
// backend\src\main\java\me\zhulin\shopapi\repository\ProductInfoRepository.java
package me.zhulin.shopapi.repository;

import me.zhulin.shopapi.entity.ProductInfo;
import org.springframework.data.jpa.repository.JpaRepository;
// ... other imports ...

public interface ProductInfoRepository extends JpaRepository<ProductInfo, String> {
    // We can define custom methods here!
    ProductInfo findByProductId(String id);
    // ... more methods ...
}
```

Let's break this down:

1.  **`public interface ProductInfoRepository`**: This declares that `ProductInfoRepository` is a Java `interface`. We don't write an implementation class for it (mostly!). Spring Data JPA creates the actual working code for us behind the scenes.
2.  **`extends JpaRepository<ProductInfo, String>`**: This is the magic part!
    *   `JpaRepository` is a powerful interface provided by Spring Data JPA. By "extending" it, our `ProductInfoRepository` automatically inherits a ton of pre-built methods for common database operations.
    *   `<ProductInfo, String>`:
        *   `ProductInfo`: This tells `JpaRepository` *which* [Data Model (Entity)](01_data_models__entities__.md) this repository will manage. So, `ProductInfoRepository` is specifically for `ProductInfo` objects.
        *   `String`: This tells `JpaRepository` the data type of the **primary key** for `ProductInfo`. In our `ProductInfo` entity, `productId` is a `String`.

### What Methods Do We Get for Free?

Because `ProductInfoRepository` extends `JpaRepository`, it automatically provides methods like:

*   `save(ProductInfo product)`: To create a new product or update an existing one in the database.
*   `findById(String id)`: To find a product by its unique `productId`. This returns an `Optional<ProductInfo>`, which is a way to safely handle cases where the product might not be found.
*   `findAll()`: To get a list of all products.
*   `delete(ProductInfo product)`: To remove a product from the database.
*   ... and many more!

### Defining Our Own Custom Methods

What if we need to find a product not just by its `productId`, but by its `productStatus` (e.g., all products that are 'on sale')? Spring Data JPA has another trick: you can define methods by following a specific naming convention, and it will *automatically generate the query for you*!

In `ProductInfoRepository`, you'll see this:

```java
// backend\src\main\java\me\zhulin\shopapi\repository\ProductInfoRepository.java
// ...
public interface ProductInfoRepository extends JpaRepository<ProductInfo, String> {
    // ... inherited methods ...

    // Custom method: Find a product by its product ID
    ProductInfo findByProductId(String id);

    // Custom method: Find all products with a specific status, ordered by product ID
    Page<ProductInfo> findAllByProductStatusOrderByProductIdAsc(Integer productStatus, Pageable pageable);

    // ... more methods ...
}
```

*   `findByProductId(String id)`: Spring Data JPA looks at `findBy` and `ProductId` and understands you want to find a `ProductInfo` where the `productId` field matches the `id` you provide.
*   `findAllByProductStatusOrderByProductIdAsc(Integer productStatus, Pageable pageable)`: This is more advanced! Spring understands:
    *   `findAllBy`: Get all records.
    *   `ProductStatus`: Filter by the `productStatus` field.
    *   `OrderByProductIdAsc`: Sort the results by `productId` in ascending order.
    *   `Pageable pageable`: This allows us to ask for results in "pages" (e.g., "give me the first 10 products," "give me the next 10 products"), which is great for large lists.

It's like telling the librarian, "Find all the product books that are 'on sale,' and please give them to me 10 at a time, sorted by their ID numbers."

### How Services Use Repositories (The "How-to")

Let's revisit our `ProductServiceImpl.decreaseStock()` method from [Chapter 2: Business Services](02_business_services_.md) to see how repositories are used:

```java
// backend\src\main\java\me\zhulin\shopapi\service\impl\ProductServiceImpl.java
package me.zhulin.shopapi.service.impl;

import me.zhulin.shopapi.entity.ProductInfo;
import me.zhulin.shopapi.repository.ProductInfoRepository; // Import the repository
// ... other imports ...
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProductServiceImpl implements ProductService {
    @Autowired // Spring automatically provides an instance of ProductInfoRepository
    ProductInfoRepository productInfoRepository; // The "librarian" for products

    @Override
    @Transactional
    public void decreaseStock(String productId, int amount) {
        // 1. Ask the ProductInfoRepository to find the product by its ID
        ProductInfo productInfo = productInfoRepository.findById(productId)
                                                       .orElse(null); // Get the ProductInfo object or null if not found

        // ... (business rules: check if product exists, check stock) ...

        // 2. Update the product's stock in the Java object
        productInfo.setProductStock(productInfo.getProductStock() - amount);

        // 3. Ask the ProductInfoRepository to save the updated product back to the database
        productInfoRepository.save(productInfo); // This sends the updated info to the database
    }
    // ... other methods ...
}
```

Here's what's happening:
*   `@Autowired ProductInfoRepository productInfoRepository;`: The `ProductService` doesn't create the repository itself. Spring automatically "injects" (provides) an instance of `ProductInfoRepository` when `ProductServiceImpl` is created. This is [Dependency Injection](02_business_services_.md#under-the-hood-service-implementations--the-how) in action!
*   `productInfoRepository.findById(productId).orElse(null);`: This line calls the `findById` method that `ProductInfoRepository` inherited from `JpaRepository`. It asks the repository to fetch the `ProductInfo` data from the database using the given `productId`.
*   `productInfoRepository.save(productInfo);`: After the business logic updates the `productStock` in the Java `productInfo` object, this line calls the `save` method (also inherited). This tells the repository to take the updated `productInfo` object and persist (save) its changes back into the database.

### Under the Hood: The Spring Data JPA Magic

When you call a method like `productInfoRepository.findById("someId")` or `productInfoRepository.save(productInfo)`, here's a simplified view of what happens behind the scenes:

```mermaid
sequenceDiagram
    participant Service as Business Service
    participant Repository as Data Repository (Interface)
    participant SD_JPA as Spring Data JPA
    participant Hibernate as Hibernate (JPA Impl)
    participant Database as Database

    Service->>Repository: findById("someId")
    Repository->>SD_JPA: Passes the call to Spring Data JPA
    SD_JPA->>Hibernate: Instructs Hibernate to generate SQL for "findById"
    Hibernate->>Database: Executes SQL Query (e.g., SELECT * FROM product_info WHERE product_id = 'someId')
    Database-->>Hibernate: Returns raw data (rows from table)
    Hibernate-->>SD_JPA: Converts raw data into Java ProductInfo object
    SD_JPA-->>Repository: Returns the ProductInfo object
    Repository-->>Service: The ProductInfo object is now available in your service

    Service->>Service: productInfo.setProductStock(newStock); (Java object update)
    Service->>Repository: save(productInfo)
    Repository->>SD_JPA: Passes the call to Spring Data JPA
    SD_JPA->>Hibernate: Instructs Hibernate to generate SQL for "save" (INSERT/UPDATE)
    Hibernate->>Database: Executes SQL Update (e.g., UPDATE product_info SET product_stock = ? WHERE product_id = ?)
    Database-->>Hibernate: SQL execution confirmed
    Hibernate-->>SD_JPA: Confirmation
    SD_JPA-->>Repository: Confirmation
    Repository-->>Service: Operation complete
```

The amazing part is that as a developer, you only write the `interface` and call simple Java methods. You don't write any `SELECT`, `INSERT`, `UPDATE`, or `DELETE` SQL commands directly. Spring Data JPA, along with Hibernate, translates your Java method calls into the correct SQL queries and interacts with your database.

### Other Repositories in Our Project

Our online store project uses many repositories, each specializing in a different [Data Model (Entity)](01_data_models__entities__.md):

*   **`UserRepository`**: Manages `User` entities.
    ```java
    // backend\src\main\java\me\zhulin\shopapi\repository\UserRepository.java
    // ...
    public interface UserRepository extends JpaRepository<User, String> {
        User findByEmail(String email); // Find a user by their email
        Collection<User> findAllByRole(String role); // Find all users with a specific role
    }
    ```
    This shows how we can find a user simply by their email address, which is crucial for login.

*   **`CartRepository`**: Manages `Cart` entities.
    ```java
    // backend\src\main\java\me\zhulin\shopapi\repository\CartRepository.java
    // ...
    public interface CartRepository extends JpaRepository<Cart, Integer> {
    }
    ```
    Even without custom methods, `CartRepository` gets all the standard `save`, `findById`, `findAll` operations from `JpaRepository`.

*   **`OrderRepository`**: Manages `OrderMain` entities.
    ```java
    // backend\src\main\java\me\zhulin\shopapi\repository\OrderRepository.java
    // ...
    public interface OrderRepository extends JpaRepository<OrderMain, Integer> {
        OrderMain findByOrderId(Long orderId);
        Page<OrderMain> findAllByBuyerEmailOrderByOrderStatusAscCreateTimeDesc(String buyerEmail, Pageable pageable);
        // ... and other ways to find orders ...
    }
    ```
    This repository allows us to easily find a specific order by its ID or retrieve all orders for a particular buyer, sorted by status and creation time.

### Conclusion

In this chapter, we've explored **Data Repositories**, the dedicated components that bridge our [Business Services](02_business_services_.md) to the database. We learned that these are powerful Java interfaces that, thanks to Spring Data JPA, automatically provide common database operations and allow us to define custom queries simply by naming methods correctly. They hide the complexity of SQL, enabling our services to focus purely on business logic.

Now that we understand our data structure ([Data Models (Entities)](01_data_models__entities__.md)), how business rules are applied ([Business Services](02_business_services_.md)), and how data is saved/retrieved (Data Repositories), the next step is to make our application accessible to the outside world. How do users and our Angular frontend actually send requests to our Spring Boot backend? That's what we'll cover in the next chapter: [API Controllers](04_api_controllers_.md)!

Ready to expose our store to the world? Let's dive into [API Controllers](04_api_controllers_.md)!

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)