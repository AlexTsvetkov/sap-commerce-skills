# Course 05: SAP Commerce Service Layer & Business Logic

## 🎯 Course Overview
Deep dive into the Service Layer — services, facades, converters, populators, interceptors, strategies, and events. Learn how to implement business logic the Commerce way.

**Duration**: 10-12 hours | **Level**: Intermediate | **Prerequisites**: Courses 01-04

---

## 📋 Topics Covered

### 1. Service Layer Architecture
- **Services** — Business logic (e.g., `ProductService`, `CartService`)
- **Facades** — DTO conversion layer (Model → Data)
- **DAOs** — Data access via FlexibleSearch
- **Interceptors** — Lifecycle hooks (validate, prepare, load, remove)
- **Strategies** — Pluggable algorithms (e.g., pricing, tax calculation)

### 2. Services
```java
// Interface
public interface MyProductService {
    ProductModel getProductByCode(String code);
    List<ProductModel> getActiveProducts();
}

// Implementation
public class DefaultMyProductService implements MyProductService {
    private FlexibleSearchService flexibleSearchService;
    private ModelService modelService;
    
    @Override
    public ProductModel getProductByCode(String code) {
        String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code";
        FlexibleSearchQuery fsq = new FlexibleSearchQuery(query);
        fsq.addQueryParameter("code", code);
        return flexibleSearchService.searchUnique(fsq);
    }
}
```

Spring registration:
```xml
<alias name="defaultMyProductService" alias="myProductService"/>
<bean id="defaultMyProductService" class="com.mycompany.core.services.impl.DefaultMyProductService">
    <property name="flexibleSearchService" ref="flexibleSearchService"/>
    <property name="modelService" ref="modelService"/>
</bean>
```

### 3. Facades, Converters & Populators
```java
// DTO
public class ProductData implements Serializable {
    private String code;
    private String name;
    private PriceData price;
    // getters/setters
}

// Populator
public class ProductBasicPopulator implements Populator<ProductModel, ProductData> {
    @Override
    public void populate(ProductModel source, ProductData target) {
        target.setCode(source.getCode());
        target.setName(source.getName());
    }
}

// Facade
public class DefaultMyProductFacade implements MyProductFacade {
    private MyProductService myProductService;
    private Converter<ProductModel, ProductData> productConverter;
    
    @Override
    public ProductData getProductByCode(String code) {
        ProductModel model = myProductService.getProductByCode(code);
        return productConverter.convert(model);
    }
}
```

Spring wiring:
```xml
<bean id="myProductBasicPopulator" class="...ProductBasicPopulator"/>
<bean id="myProductConverter" parent="abstractPopulatingConverter">
    <property name="targetClass" value="com.mycompany.facades.product.data.ProductData"/>
    <property name="populators">
        <list>
            <ref bean="myProductBasicPopulator"/>
            <ref bean="myProductPricePopulator"/>
        </list>
    </property>
</bean>
```

### 4. Model Interceptors
| Type | When | Use Case |
|------|------|----------|
| `InitDefaultsInterceptor` | Before create | Set default values |
| `PrepareInterceptor` | Before save | Compute derived fields |
| `ValidateInterceptor` | Before save | Validate business rules |
| `LoadInterceptor` | After load | Enrich loaded model |
| `RemoveInterceptor` | Before delete | Prevent deletion / cleanup |

```java
public class ProductValidateInterceptor implements ValidateInterceptor<ProductModel> {
    @Override
    public void onValidate(ProductModel model, InterceptorContext ctx) 
            throws InterceptorException {
        if (StringUtils.isBlank(model.getCode())) {
            throw new InterceptorException("Product code is required");
        }
    }
}
```

### 5. Event System
```java
// Custom event
public class ProductApprovedEvent extends AbstractEvent {
    private final String productCode;
    public ProductApprovedEvent(String code) { this.productCode = code; }
}

// Listener
public class ProductApprovedListener extends AbstractEventListener<ProductApprovedEvent> {
    @Override
    protected void onEvent(ProductApprovedEvent event) {
        LOG.info("Product approved: " + event.getProductCode());
    }
}

// Publishing
eventService.publishEvent(new ProductApprovedEvent("PROD-001"));
```

### 6. FlexibleSearch
```java
// Basic query
"SELECT {pk} FROM {Product} WHERE {code} = ?code"

// Join with category
"SELECT {p.pk} FROM {Product AS p JOIN CategoryProductRelation AS rel 
 ON {rel.target} = {p.pk} JOIN Category AS c ON {c.pk} = {rel.source}} 
 WHERE {c.code} = ?categoryCode"

// Localized attribute
"SELECT {pk} FROM {Product} WHERE {name[en]} LIKE ?name"

// Subquery
"SELECT {pk} FROM {Product} WHERE {pk} NOT IN (
    {{SELECT {target} FROM {StockLevel}}})"
```

### 7. Strategies Pattern
```java
public interface DeliveryTimeStrategy {
    int getEstimatedDeliveryDays(AbstractOrderModel order);
}
// Multiple implementations registered and selected based on conditions
```

---

## Exercises
1. Create a `WarrantyService` with CRUD operations for the `Warranty` type
2. Create a `WarrantyFacade` with converter + populator
3. Implement a `ValidateInterceptor` that ensures warranty duration > 0
4. Create a custom event `WarrantyExpiredEvent` and listener
5. Write FlexibleSearch queries: products without warranties, products by price range, products with specific categories

## Self-Check
- What's the difference between a Service and a Facade?
- Why use Populators instead of direct conversion?
- When does a ValidateInterceptor run vs PrepareInterceptor?
- How do you override an existing service?
- Can interceptors be ordered? How?

---
**Previous**: [← 04 - Installation](../04-installation/README.md) | **Next**: [06 - ImpEx →](../06-impex/README.md)