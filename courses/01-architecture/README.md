# Course 01: SAP Commerce Cloud Architecture

## 🎯 Course Overview

This course provides a deep dive into the architecture of SAP Commerce Cloud (formerly Hybris). You'll understand the platform's layered architecture, runtime model, Spring integration, multi-tenancy, clustering, and how all core components fit together.

**Duration**: 8-10 hours  
**Level**: Intermediate  
**Prerequisites**: Java 11+, Spring Framework basics, relational databases

---

## 📋 Table of Contents

1. [Platform Overview & History](#1-platform-overview--history)
2. [High-Level Architecture](#2-high-level-architecture)
3. [The Platform Layer](#3-the-platform-layer)
4. [Spring Framework Integration](#4-spring-framework-integration)
5. [Extension-Based Architecture](#5-extension-based-architecture)
6. [Persistence Layer & Database](#6-persistence-layer--database)
7. [Multi-Tenancy](#7-multi-tenancy)
8. [Clustering & Scalability](#8-clustering--scalability)
9. [Request Processing Pipeline](#9-request-processing-pipeline)
10. [SAP Commerce Cloud (CCv2)](#10-sap-commerce-cloud-ccv2)
11. [Exercises](#11-exercises)
12. [Self-Check Questions](#12-self-check-questions)

---

## 1. Platform Overview & History

### What is SAP Commerce Cloud?

SAP Commerce Cloud is an enterprise-grade, Java-based e-commerce platform that provides:

- **Product Content Management (PCM)** — rich product modeling and catalog management
- **Order Management** — complete order lifecycle from cart to fulfillment
- **Customer Experience** — personalization, promotions, and customer segmentation
- **Omnichannel Commerce** — B2C, B2B, B2B2C, marketplace scenarios
- **Cloud-Native Deployment** — managed Kubernetes-based cloud infrastructure (CCv2)

### Evolution Timeline

| Year | Milestone |
|------|-----------|
| 2001 | Hybris founded in Switzerland |
| 2013 | SAP acquires Hybris |
| 2017 | Rebranded to SAP Commerce Cloud |
| 2018 | CCv2 (Cloud v2) — Kubernetes-based deployment |
| 2020 | Spartacus (Angular-based headless storefront) |
| 2023 | Composable Storefront GA, Commerce 2211 LTS |
| 2024+ | Continued cloud-native evolution, AI integration |

### Key Differentiators

- **Type System** — flexible, runtime-extendable data model (no manual DDL)
- **Extension Mechanism** — modular architecture where functionality is packaged in extensions
- **ImpEx** — powerful data import/export framework
- **Built-in B2B/B2C** — out-of-the-box accelerators for both business models
- **SAP Ecosystem** — native integration with S/4HANA, CPI, CDC, Emarsys, Qualtrics

---

## 2. High-Level Architecture

SAP Commerce follows a **layered architecture** pattern:

```
┌─────────────────────────────────────────────────┐
│              Presentation Layer                  │
│  (Composable Storefront / Spartacus / OCC API)  │
├─────────────────────────────────────────────────┤
│                Façade Layer                      │
│      (Commerce Facades — DTO conversion)        │
├─────────────────────────────────────────────────┤
│               Service Layer                     │
│    (Business Logic — Services, Strategies)      │
├─────────────────────────────────────────────────┤
│                 DAO Layer                        │
│  (FlexibleSearch, GenericDAO, ModelService)      │
├─────────────────────────────────────────────────┤
│              Persistence Layer                   │
│        (Type System, Jalo → Model)              │
├─────────────────────────────────────────────────┤
│            Platform / Kernel                     │
│   (Initialization, Tenants, Clustering, Cache)  │
├─────────────────────────────────────────────────┤
│          Infrastructure Layer                    │
│    (Database, Solr, Media Storage, Messaging)    │
└─────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility | Key Classes |
|-------|---------------|-------------|
| **Presentation** | UI rendering, REST APIs | Controllers, OCC endpoints |
| **Façade** | Data transformation (Model → DTO) | `*Facade`, Converters, Populators |
| **Service** | Business logic orchestration | `*Service`, Strategies, Interceptors |
| **DAO** | Data access abstraction | `*Dao`, FlexibleSearch queries |
| **Persistence** | Object-relational mapping | `ModelService`, Type System |
| **Platform** | Core infrastructure | `Registry`, `Tenant`, `Cluster` |

### Important Architectural Principles

1. **Don't bypass layers** — Always go through the proper layer chain (Controller → Facade → Service → DAO)
2. **Models are POJOs** — Data objects generated from the type system, not JPA entities
3. **Spring-managed beans** — All services, facades, and DAOs are Spring beans
4. **Extension isolation** — Each extension has its own Spring context, web context, and resources

---

## 3. The Platform Layer

The platform is the core kernel of SAP Commerce. It handles:

### 3.1 Initialization & Update

```
┌────────────────────────────┐
│     System Initialization  │
│                            │
│  1. Create DB schema       │
│  2. Create type system     │
│  3. Import essential data  │
│  4. Import project data    │
│  5. Run system setup       │
└────────────────────────────┘

┌────────────────────────────┐
│       System Update        │
│                            │
│  1. Update DB schema       │
│  2. Update type system     │
│  3. Import essential data  │
│  4. Import project data    │
│  5. Run update hooks       │
└────────────────────────────┘
```

- **Initialization** destroys and recreates everything — used for fresh installations
- **Update** modifies existing schema without data loss — used for deployments

### 3.2 The Registry

The `Registry` class is the entry point to the platform:

```java
// Access the current tenant
Tenant tenant = Registry.getCurrentTenant();

// Access the master tenant
Tenant master = Registry.getMasterTenant();

// Access application context
ApplicationContext ctx = Registry.getApplicationContext();

// Check if the system is running
boolean running = Registry.isCurrentTenantRunning();
```

### 3.3 Startup Lifecycle

```
1. JVM starts → Tomcat initializes
2. Platform webapp loads → HybrisContextLoaderListener
3. Registry.startup() called
4. Master tenant activates
5. Extensions load in dependency order
6. Spring contexts initialize per extension
7. System is ready to serve requests
```

### 3.4 The HAC (Hybris Administration Console)

HAC (`/hac`) is the administrative web application providing:

- **Console** — Execute FlexibleSearch, Groovy, ImpEx
- **Monitoring** — JVM metrics, cache stats, cluster info
- **Maintenance** — System update, initialization, CronJobs
- **Configuration** — Runtime property management
- **Type System** — Browse and inspect types

---

## 4. Spring Framework Integration

SAP Commerce deeply integrates with Spring Framework. Understanding this integration is critical.

### 4.1 Spring Context Hierarchy

Each extension has its own Spring application context:

```
Global Application Context (platform)
├── core-spring.xml (core extension)
├── catalog-spring.xml (catalog extension)
├── commerceservices-spring.xml
├── myextension-spring.xml
└── ... (one per extension)
```

The contexts are merged into a single `ApplicationContext` at runtime.

### 4.2 Bean Definition

Beans are defined in `<extension>-spring.xml`:

```xml
<!-- Service bean definition -->
<alias name="defaultProductService" alias="productService"/>
<bean id="defaultProductService" 
      class="de.hybris.platform.product.impl.DefaultProductService">
    <property name="modelService" ref="modelService"/>
    <property name="productDao" ref="productDao"/>
</bean>
```

### 4.3 Bean Override Pattern

The **alias pattern** is the standard way to override services:

```xml
<!-- In your custom extension -->
<alias name="myCustomProductService" alias="productService"/>
<bean id="myCustomProductService" 
      class="com.mycompany.core.services.impl.MyProductService"
      parent="defaultProductService">
    <!-- Override specific dependencies or add new ones -->
</bean>
```

This pattern allows:
- Original bean remains available as `defaultProductService`
- All references to `productService` now resolve to your custom implementation
- You can extend the default implementation using `parent`

### 4.4 Spring Profiles

Commerce supports Spring profiles for environment-specific configuration:

```xml
<beans profile="development">
    <bean id="mockPaymentService" class="...MockPaymentService"/>
</beans>

<beans profile="production">
    <bean id="realPaymentService" class="...RealPaymentService"/>
</beans>
```

### 4.5 Event System

Commerce uses Spring events extensively:

```java
// Publishing an event
eventService.publishEvent(new AfterItemCreationEvent(myProduct));

// Listening to an event
public class MyEventListener extends AbstractEventListener<AfterItemCreationEvent> {
    @Override
    protected void onEvent(AfterItemCreationEvent event) {
        // Handle the event
    }
}
```

---

## 5. Extension-Based Architecture

Extensions are the fundamental building blocks of SAP Commerce.

### 5.1 What is an Extension?

An extension is a self-contained module that:
- Defines its own **type system** additions (via `*-items.xml`)
- Has its own **Spring context** (`*-spring.xml`)
- Can include **web applications** (controllers, views)
- Declares **dependencies** on other extensions
- Contains **ImpEx** scripts for data loading
- Has its own **build configuration**

### 5.2 Extension Structure

```
myextension/
├── src/                          # Java source code
├── testsrc/                      # Test source code
├── web/
│   ├── src/                      # Web layer source
│   ├── webroot/
│   │   ├── WEB-INF/
│   │   │   ├── web.xml
│   │   │   └── views/           # JSP/Thymeleaf templates
│   │   └── static/              # Static resources
│   └── spring/
│       └── myextension-web-spring.xml
├── resources/
│   ├── myextension-spring.xml    # Spring bean definitions
│   ├── myextension-items.xml     # Type system definitions
│   ├── localization/
│   │   └── myextension-locales_en.properties
│   └── impex/
│       ├── essentialdata.impex   # Loaded on init & update
│       └── projectdata.impex    # Loaded on init & update
├── buildcallbacks.xml           # Custom build hooks
├── extensioninfo.xml            # Extension metadata & dependencies
├── external-dependencies.xml    # Maven dependencies
└── project.properties           # Extension-level properties
```

### 5.3 Extension Types

| Type | Description | Example |
|------|-------------|---------|
| **Platform extension** | Core platform functionality | `core`, `catalog`, `europe1` |
| **Commerce extension** | Business functionality | `commerceservices`, `commercefacades` |
| **Accelerator** | Storefront templates | `yacceleratorstorefront` |
| **AddOn** | Pluggable storefront features | `captchaaddon`, `b2bpunchoutaddon` |
| **Template** | Starting point for custom code | `yempty`, `ycoreplus` |
| **Custom extension** | Your project code | `myprojectcore`, `myprojectstorefront` |

### 5.4 Extension Dependencies

Defined in `extensioninfo.xml`:

```xml
<extensioninfo>
    <extension name="myextension" 
               classprefix="MyExtension"
               version="1.0"
               jaloclass="com.mycompany.jalo.MyExtensionManager">
        <requires-extension name="core"/>
        <requires-extension name="commerceservices"/>
        <requires-extension name="catalog"/>
    </extension>
</extensioninfo>
```

### 5.5 localextensions.xml

This file in `config/` defines which extensions are loaded:

```xml
<hybrisconfig>
    <extensions>
        <path dir="${HYBRIS_BIN_DIR}"/>
        <extension name="backoffice"/>
        <extension name="commerceservices"/>
        <extension name="myprojectcore"/>
        <extension name="myprojectstorefront"/>
    </extensions>
</hybrisconfig>
```

---

## 6. Persistence Layer & Database

### 6.1 How Persistence Works

SAP Commerce uses its own **persistence framework** (NOT JPA/Hibernate):

```
Type System (items.xml)
       ↓
  Code Generation (ant all)
       ↓
  Model Classes (generated POJOs)
       ↓
  ModelService (CRUD operations)
       ↓
  Database Tables (auto-generated DDL)
```

### 6.2 ModelService

The `ModelService` is the primary API for CRUD operations:

```java
@Autowired
private ModelService modelService;

// CREATE
ProductModel product = modelService.create(ProductModel.class);
product.setCode("PROD-001");
product.setCatalogVersion(catalogVersion);
product.setName("My Product");
modelService.save(product);

// READ (via FlexibleSearch — see DAO layer)
// UPDATE
product.setName("Updated Product");
modelService.save(product);

// DELETE
modelService.remove(product);
```

### 6.3 FlexibleSearch

FlexibleSearch is Commerce's query language (translates to SQL):

```java
String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code";
FlexibleSearchQuery fsq = new FlexibleSearchQuery(query);
fsq.addQueryParameter("code", "PROD-001");
SearchResult<ProductModel> result = flexibleSearchService.search(fsq);
List<ProductModel> products = result.getResult();
```

### 6.4 Database Support

| Database | Support Level |
|----------|--------------|
| SAP HANA | Primary (CCv2) |
| MySQL | Development |
| HSQLDB | Unit testing |
| Oracle | Supported |
| MS SQL Server | Supported |
| PostgreSQL | Supported (recent versions) |

### 6.5 Table Naming Convention

Type system types map to database tables:

| Type | Table Name |
|------|-----------|
| `Product` | `products` |
| `Category` | `categories` |
| `Customer` (extends User) | `users` |
| Custom `MyType` | `mytypes` (configurable via deployment table) |

---

## 7. Multi-Tenancy

SAP Commerce supports multi-tenancy at the platform level.

### 7.1 Tenant Concept

```
┌─────────────────────────────────────┐
│          SAP Commerce Instance      │
│                                     │
│  ┌──────────┐    ┌──────────────┐  │
│  │  Master   │    │   JUnit      │  │
│  │  Tenant   │    │   Tenant     │  │
│  │           │    │              │  │
│  │ - Own DB  │    │ - Own DB     │  │
│  │ - Own     │    │ - Separate   │  │
│  │   Config  │    │   Schema     │  │
│  │ - Own     │    │ - Test data  │  │
│  │   Data    │    │              │  │
│  └──────────┘    └──────────────┘  │
└─────────────────────────────────────┘
```

### 7.2 Tenant Types

- **Master Tenant** — Primary tenant running the application
- **JUnit Tenant** — Isolated tenant for integration tests (separate DB schema)
- **Slave Tenants** — Additional tenants (rarely used in modern deployments)

### 7.3 JUnit Tenant

The JUnit tenant enables isolated integration testing:

```java
@IntegrationTest
@UnitTest // For unit tests (no tenant needed)
public class MyServiceIntegrationTest extends ServicelayerTransactionalTest {
    
    @Resource
    private ProductService productService;
    
    @Test
    public void testFindProduct() {
        // Runs against JUnit tenant with its own DB
        ProductModel product = productService.getProductForCode("testProduct");
        assertNotNull(product);
    }
}
```

---

## 8. Clustering & Scalability

### 8.1 Cluster Architecture

SAP Commerce supports clustering for high availability:

```
                    ┌──────────────┐
                    │ Load Balancer│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────┴─────┐ ┌───┴───┐ ┌─────┴─────┐
        │  Node 1   │ │Node 2 │ │  Node 3   │
        │ (Primary) │ │       │ │           │
        │           │ │       │ │           │
        └─────┬─────┘ └───┬───┘ └─────┬─────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────┴───────┐
                    │   Database   │
                    │   (HANA)     │
                    └──────────────┘
```

### 8.2 Cluster Communication

Nodes communicate via:

- **UDP Multicast** (legacy, on-premise)
- **TCP Unicast** (recommended)
- **Database-based** (CCv2 default)

### 8.3 Cache Invalidation

When data changes on one node, caches must be invalidated across the cluster:

```
Node 1: Save Product → Invalidate local cache → Broadcast invalidation
Node 2: Receive invalidation → Clear local cache for that entity
Node 3: Receive invalidation → Clear local cache for that entity
```

### 8.4 CronJob Distribution

In a cluster, CronJobs can be configured to run on specific nodes:

```properties
# Run only on node with ID 0
cronjob.myJob.nodeId=0

# Run on any available node
cronjob.myJob.nodeId=-1
```

---

## 9. Request Processing Pipeline

### 9.1 HTTP Request Flow

```
Client Request
      │
      ▼
┌─────────────┐
│   Tomcat    │  (Embedded Tomcat / Servlet Container)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Filters    │  (CORS, Security, Session, Tenant activation)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Dispatcher │  (Spring MVC DispatcherServlet)
│  Servlet    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Controller  │  (OCC REST Controller or Storefront Controller)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Facade    │  (Business orchestration + DTO conversion)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Service    │  (Business logic)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    DAO      │  (Data access)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Database   │
└─────────────┘
```

### 9.2 OCC (Omni Commerce Connect) API

OCC is the RESTful API layer:

```
GET /occ/v2/{baseSiteId}/products/{productCode}
GET /occ/v2/{baseSiteId}/users/{userId}/carts
POST /occ/v2/{baseSiteId}/users/{userId}/carts/{cartId}/entries
```

Key features:
- RESTful endpoints following REST conventions
- JSON response format
- OAuth2 authentication
- Swagger/OpenAPI documentation
- Extensible via custom controllers

### 9.3 Session & Context

Commerce maintains session context:

```java
// Access session context
JaloSession session = JaloSession.getCurrentSession();

// Modern way — SessionService
sessionService.setAttribute("myKey", "myValue");

// Base site, language, currency context
BaseSiteModel site = baseSiteService.getCurrentBaseSite();
LanguageModel lang = commonI18NService.getCurrentLanguage();
CurrencyModel curr = commonI18NService.getCurrentCurrency();
```

---

## 10. SAP Commerce Cloud (CCv2)

### 10.1 What is CCv2?

Commerce Cloud v2 is the managed cloud deployment platform:

```
┌─────────────────────────────────────────────┐
│           SAP Commerce Cloud (CCv2)         │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │         Kubernetes Cluster             │ │
│  │                                        │ │
│  │  ┌─────────┐  ┌─────────┐  ┌───────┐ │ │
│  │  │Commerce │  │Commerce │  │ Solr  │ │ │
│  │  │ Node 1  │  │ Node 2  │  │Cluster│ │ │
│  │  └─────────┘  └─────────┘  └───────┘ │ │
│  │                                        │ │
│  │  ┌─────────┐  ┌─────────┐  ┌───────┐ │ │
│  │  │ Backoff │  │  Media  │  │  CDN  │ │ │
│  │  │  ice    │  │ Storage │  │       │ │ │
│  │  └─────────┘  └─────────┘  └───────┘ │ │
│  └────────────────────────────────────────┘ │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │            SAP HANA DB                 │ │
│  └────────────────────────────────────────┘ │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │        Dynatrace Monitoring            │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 10.2 CCv2 Key Components

| Component | Purpose |
|-----------|---------|
| **Cloud Portal** | Web UI for managing environments, builds, deployments |
| **Manifest** | `manifest.json` — defines build & deployment config |
| **Build** | CI/CD builds triggered from Git repositories |
| **Environments** | dev, staging, production — isolated infrastructure |
| **Endpoints** | Public URLs for storefront, API, Backoffice |
| **Media Storage** | Cloud blob storage for media (images, documents) |
| **Monitoring** | Dynatrace APM integration |

### 10.3 manifest.json

```json
{
    "commerceSuiteVersion": "2211.28",
    "enableImageProcessingService": true,
    "extensions": [
        "modeltacceleratorservices",
        "commercewebservicescommons",
        "myprojectcore",
        "myprojectstorefront"
    ],
    "useConfig": {
        "extensions": {
            "location": "cloud/extensions.xml"
        },
        "properties": [
            {
                "location": "cloud/common.properties"
            },
            {
                "location": "cloud/dev.properties",
                "aspect": "accstorefront"
            }
        ],
        "solr": {
            "location": "cloud/solr"
        }
    },
    "aspects": [
        {
            "name": "accstorefront",
            "webapps": [
                {
                    "name": "myprojectstorefront",
                    "contextPath": ""
                }
            ]
        },
        {
            "name": "backoffice",
            "webapps": [
                {
                    "name": "backoffice",
                    "contextPath": "/backoffice"
                }
            ]
        },
        {
            "name": "backgroundProcessing",
            "webapps": [
                {
                    "name": "hac",
                    "contextPath": "/hac"
                }
            ]
        }
    ]
}
```

### 10.4 Aspects

Aspects define how the application is deployed across pods:

| Aspect | Purpose | Typical Webapps |
|--------|---------|-----------------|
| `accstorefront` | Storefront serving | Storefront, OCC API |
| `backoffice` | Admin/back-office | Backoffice |
| `backgroundProcessing` | CronJobs, processing | HAC |
| `api` | API-only serving | OCC webservices |

---

## 11. Exercises

### Exercise 1: Explore the Platform

1. Access HAC at `https://<your-server>/hac`
2. Navigate to **Platform → Configuration** and find the following properties:
   - `db.url`
   - `cluster.id`
   - `installed.tenants`
3. Navigate to **Console → FlexibleSearch** and run:
   ```sql
   SELECT {pk}, {code}, {name} FROM {Product} WHERE {code} LIKE 'HW%'
   ```
4. Document the results and explain what the query does.

### Exercise 2: Understand Extension Loading

1. Open `config/localextensions.xml` on your project
2. List all extensions that are loaded
3. For 3 extensions of your choice, find their `extensioninfo.xml` and document their dependencies
4. Draw a dependency graph for these extensions

### Exercise 3: Explore the Service Layer

1. Using HAC Groovy console, execute:
   ```groovy
   import de.hybris.platform.product.ProductService
   
   def productService = spring.getBean("productService")
   println "ProductService class: ${productService.class.name}"
   
   def ctx = de.hybris.platform.core.Registry.getApplicationContext()
   def beanNames = ctx.getBeanDefinitionNames().findAll { it.contains("product") }
   beanNames.each { println it }
   ```
2. Identify which class implements `productService`
3. Find the Spring XML definition for this service

### Exercise 4: Cluster Configuration

1. Review the following properties in `local.properties`:
   ```properties
   cluster.id=0
   cluster.broadcast.method.jgroups.tcp.bind_addr=
   cluster.broadcast.method.jgroups.tcp.bind_port=
   ```
2. Explain what each property controls
3. Describe what would happen if two nodes had the same `cluster.id`

### Exercise 5: CCv2 Manifest Analysis

1. Review the `manifest.json` example in section 10.3
2. Answer:
   - Which Commerce version is being deployed?
   - How many aspects are defined?
   - Which webapp serves the storefront?
   - Where are custom properties loaded from?
3. Create a modified manifest that adds an `api` aspect for OCC webservices

---

## 12. Self-Check Questions

### Knowledge Check

1. **What are the main layers in SAP Commerce architecture?**
   <details>
   <summary>Answer</summary>
   Presentation, Façade, Service, DAO, Persistence, Platform, Infrastructure
   </details>

2. **How does SAP Commerce manage its database schema?**
   <details>
   <summary>Answer</summary>
   Through the Type System. Developers define types in `items.xml` files, and the platform automatically generates/updates database tables during system initialization or update. No manual DDL is needed.
   </details>

3. **What is the difference between system initialization and system update?**
   <details>
   <summary>Answer</summary>
   Initialization destroys all existing data and recreates the schema from scratch. Update modifies the existing schema to match the current type system definition without destroying data.
   </details>

4. **How do you override an existing service in SAP Commerce?**
   <details>
   <summary>Answer</summary>
   Using the alias pattern in Spring XML. Create a new bean definition, then create an alias that points the original service name to your new bean. Example:
   ```xml
   <alias name="myCustomService" alias="originalService"/>
   <bean id="myCustomService" class="..." parent="defaultOriginalService"/>
   ```
   </details>

5. **What is the purpose of the JUnit tenant?**
   <details>
   <summary>Answer</summary>
   The JUnit tenant provides an isolated environment for integration testing with its own database schema. This allows tests to run without affecting the master tenant's data.
   </details>

6. **What is an Aspect in CCv2?**
   <details>
   <summary>Answer</summary>
   An Aspect defines a deployment unit (pod type) in CCv2. Each aspect specifies which web applications it serves and can be independently scaled. Common aspects are: accstorefront (storefront), backoffice, backgroundProcessing, and api.
   </details>

7. **What communication method do CCv2 cluster nodes use?**
   <details>
   <summary>Answer</summary>
   CCv2 uses database-based cluster communication by default, rather than UDP multicast or TCP unicast used in on-premise deployments.
   </details>

8. **Why is the alias pattern used instead of directly overriding bean IDs?**
   <details>
   <summary>Answer</summary>
   The alias pattern preserves the original bean definition (accessible via its original ID like `defaultProductService`), allows your custom implementation to extend it using `parent`, and ensures all existing references to the alias resolve to your custom implementation. Direct ID override would lose the original definition.
   </details>

---

## 📚 Further Reading

- [SAP Commerce Cloud Help Portal](https://help.sap.com/docs/SAP_COMMERCE)
- [SAP Commerce Cloud Architecture Overview](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8b93af3e86691014b586f08a4843b5cc.html)
- [Extension Concept](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8b94a3c286691014afacb6e6bce25832.html)
- [CCv2 Documentation](https://help.sap.com/docs/SAP_COMMERCE_CLOUD_PUBLIC_CLOUD)

---

**Next Course**: [02 - Type System & Data Modeling →](../02-type-system/README.md)