# Course 03: SAP Commerce Extensions & AddOns

## 🎯 Course Overview

Extensions are the fundamental modularization unit in SAP Commerce. This course covers how to create, configure, and manage extensions, the AddOn mechanism for storefront plugins, extension dependencies, and project structure best practices.

**Duration**: 6-8 hours  
**Level**: Intermediate  
**Prerequisites**: [Course 01: Architecture](../01-architecture/README.md), [Course 02: Type System](../02-type-system/README.md)

---

## 📋 Table of Contents

1. [Extension Fundamentals](#1-extension-fundamentals)
2. [Creating Extensions](#2-creating-extensions)
3. [Extension Configuration](#3-extension-configuration)
4. [Extension Lifecycle](#4-extension-lifecycle)
5. [Standard Extension Patterns](#5-standard-extension-patterns)
6. [AddOns](#6-addons)
7. [Extension Dependencies & Loading Order](#7-extension-dependencies--loading-order)
8. [Project Structure Best Practices](#8-project-structure-best-practices)
9. [Build Callbacks](#9-build-callbacks)
10. [External Dependencies (Maven)](#10-external-dependencies-maven)
11. [Exercises](#11-exercises)
12. [Self-Check Questions](#12-self-check-questions)

---

## 1. Extension Fundamentals

### What is an Extension?

An extension is a **self-contained module** that packages:
- Data model definitions (type system)
- Business logic (services, strategies)
- Web layer (controllers, APIs)
- Configuration (Spring beans, properties)
- Data (ImpEx scripts)
- Tests

### Extension Categories

| Category | Purpose | Examples |
|----------|---------|---------|
| **Platform** | Core framework | `core`, `processing`, `scripting` |
| **Commerce** | Business modules | `commerceservices`, `commercefacades`, `basecommerce` |
| **Accelerator** | Storefront templates | `yacceleratorstorefront`, `yacceleratorinitialdata` |
| **Backoffice** | Admin UI framework | `backoffice`, `pcmbackoffice`, `ordermanagementbackoffice` |
| **Integration** | External systems | `sapintegrationservices`, `kymaintegrationservices` |
| **Custom** | Your project code | `myprojectcore`, `myprojectfacades`, `myprojectstorefront` |

### Anatomy of an Extension

```
myextension/
├── src/                              # Java source code
│   └── com/mycompany/
│       ├── services/                 # Service interfaces
│       │   └── MyService.java
│       ├── services/impl/            # Service implementations
│       │   └── DefaultMyService.java
│       ├── daos/                     # Data Access Objects
│       │   └── MyDao.java
│       └── interceptors/             # Model interceptors
│           └── MyValidateInterceptor.java
├── testsrc/                          # Test code
│   └── com/mycompany/
│       ├── services/impl/
│       │   └── DefaultMyServiceTest.java
│       └── daos/
│           └── MyDaoTest.java
├── web/                              # Web application (optional)
│   ├── src/
│   │   └── com/mycompany/controllers/
│   │       └── MyController.java
│   ├── webroot/
│   │   ├── WEB-INF/
│   │   │   ├── web.xml
│   │   │   └── views/
│   │   └── static/
│   └── spring/
│       └── myextension-web-spring.xml
├── resources/
│   ├── myextension-spring.xml        # Spring bean definitions
│   ├── myextension-items.xml         # Type system definitions
│   ├── localization/
│   │   ├── myextension-locales_en.properties
│   │   └── myextension-locales_de.properties
│   └── impex/
│       ├── essentialdata-myextension.impex
│       └── projectdata-myextension.impex
├── buildcallbacks.xml                # Build hooks
├── extensioninfo.xml                 # Extension metadata
├── external-dependencies.xml         # Maven dependencies
└── project.properties                # Extension properties
```

---

## 2. Creating Extensions

### 2.1 Using the Extension Generator (ant extgen)

```bash
cd ${HYBRIS_DIR}/bin/platform
ant extgen
```

The generator prompts for:
1. **Template**: Which template to use (e.g., `yempty`, `yaddon`)
2. **Extension name**: Your extension name (e.g., `myprojectcore`)
3. **Package name**: Java package (e.g., `com.mycompany.core`)

### 2.2 Available Templates

| Template | Description | Use Case |
|----------|-------------|----------|
| `yempty` | Minimal extension | Services, utilities |
| `yaddon` | Storefront add-on | Pluggable storefront features |
| `ybackoffice` | Backoffice extension | Custom admin UI |
| `ywebservices` | REST API extension | Custom REST endpoints |
| `ycommercewebservices` | Commerce REST API | OCC API extensions |

### 2.3 Manual Extension Creation

You can also create an extension manually:

1. Create the directory structure
2. Create `extensioninfo.xml`
3. Create `project.properties`
4. Create Spring XML files
5. Register in `localextensions.xml`

### 2.4 extensioninfo.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<extensioninfo xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="extensioninfo.xsd">
    
    <extension abstractclassprefix="Generated"
               classprefix="MyProject"
               name="myprojectcore"
               version="1.0.0"
               managersuperclass="de.hybris.platform.jalo.extension.Extension"
               jaloclass="com.mycompany.core.jalo.MyProjectCoreManager"
               isOldStyleExtension="false">
        
        <!-- Dependencies -->
        <requires-extension name="commerceservices"/>
        <requires-extension name="catalog"/>
        <requires-extension name="europe1"/>
        
        <!-- Core module (always loaded) -->
        <coremodule generated="true" manager="com.mycompany.core.jalo.MyProjectCoreManager"
                    packageroot="com.mycompany.core"/>
        
        <!-- Web module (optional, for web applications) -->
        <webmodule jspcompile="false" webroot="/myprojectweb"/>
        
    </extension>
</extensioninfo>
```

---

## 3. Extension Configuration

### 3.1 project.properties

Extension-level properties (can be overridden in `config/local.properties`):

```properties
# Extension properties
myextension.feature.enabled=true
myextension.batch.size=100
myextension.cache.ttl=3600

# Build settings
myextension.extension.requires-javadoc=false
```

### 3.2 local.properties

Global properties in `config/local.properties`:

```properties
# Database configuration
db.url=jdbc:mysql://localhost:3306/commerce
db.driver=com.mysql.cj.jdbc.Driver
db.username=hybris
db.password=hybris

# Server settings
tomcat.http.port=9001
tomcat.ssl.port=9002

# Extension-specific overrides
myextension.feature.enabled=false
```

### 3.3 localextensions.xml

Register your extension:

```xml
<hybrisconfig>
    <extensions>
        <path dir="${HYBRIS_BIN_DIR}"/>
        
        <!-- SAP extensions -->
        <extension name="commerceservices"/>
        <extension name="commercefacades"/>
        <extension name="acceleratorstorefrontcommons"/>
        <extension name="backoffice"/>
        
        <!-- Custom extensions -->
        <extension name="myprojectcore"/>
        <extension name="myprojectfacades"/>
        <extension name="myprojectstorefront"/>
        <extension name="myprojectinitialdata"/>
        <extension name="myprojectbackoffice"/>
    </extensions>
</hybrisconfig>
```

### 3.4 Spring Configuration

**Core Spring context** (`resources/myextension-spring.xml`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- Service definitions -->
    <alias name="defaultMyService" alias="myService"/>
    <bean id="defaultMyService" class="com.mycompany.core.services.impl.DefaultMyService">
        <property name="modelService" ref="modelService"/>
        <property name="flexibleSearchService" ref="flexibleSearchService"/>
        <property name="myDao" ref="myDao"/>
    </bean>
    
    <!-- DAO definitions -->
    <alias name="defaultMyDao" alias="myDao"/>
    <bean id="defaultMyDao" class="com.mycompany.core.daos.impl.DefaultMyDao">
        <property name="flexibleSearchService" ref="flexibleSearchService"/>
    </bean>
    
    <!-- Interceptor registration -->
    <bean id="myValidateInterceptor"
          class="com.mycompany.core.interceptors.MyValidateInterceptor"/>
    
    <bean id="myValidateInterceptorMapping"
          class="de.hybris.platform.servicelayer.interceptor.impl.InterceptorMapping">
        <property name="interceptor" ref="myValidateInterceptor"/>
        <property name="typeCode" value="MyCustomType"/>
    </bean>
</beans>
```

**Web Spring context** (`web/spring/myextension-web-spring.xml`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="...">

    <context:component-scan base-package="com.mycompany.controllers"/>
    
    <!-- Web-specific beans -->
</beans>
```

---

## 4. Extension Lifecycle

### 4.1 Loading Order

Extensions load based on their dependency graph:

```
platform (always first)
  → core
    → catalog
      → commerceservices
        → myprojectcore (depends on commerceservices)
          → myprojectfacades (depends on myprojectcore)
            → myprojectstorefront (depends on myprojectfacades)
```

### 4.2 System Setup (Data Loading)

Extensions can hook into system initialization and update:

```java
public class MyProjectCoreSystemSetup extends AbstractSystemSetup {
    
    @SystemSetup(type = Type.ESSENTIAL, process = Process.ALL)
    public void createEssentialData() {
        // Loaded on EVERY init and update
        importImpexFile("/myprojectcore/import/essentialdata.impex", false);
    }
    
    @SystemSetup(type = Type.PROJECT, process = Process.ALL)
    public void createProjectData() {
        // Loaded on EVERY init and update
        importImpexFile("/myprojectcore/import/projectdata.impex", false);
    }
    
    @SystemSetup(type = Type.PROJECT, process = Process.INIT)
    public void createSampleData() {
        // Loaded ONLY on initialization
        importImpexFile("/myprojectcore/import/sampledata.impex", false);
    }
}
```

Register in Spring:
```xml
<bean id="myProjectCoreSystemSetup"
      class="com.mycompany.core.setup.MyProjectCoreSystemSetup"
      parent="abstractSystemSetup"/>
```

### 4.3 ImpEx Data Loading Convention

```
resources/
├── impex/
│   ├── essentialdata-myextension.impex    # Core data (types, enums)
│   └── projectdata-myextension.impex      # Business data (catalogs, products)
└── myprojectcore/
    └── import/
        ├── coredata/                       # Core system data
        │   ├── common/
        │   │   ├── essential-data.impex
        │   │   └── delivery-modes.impex
        │   └── stores/mystore/
        │       ├── store.impex
        │       └── site.impex
        └── sampledata/                     # Sample/test data
            ├── products/
            │   └── products.impex
            └── customers/
                └── customers.impex
```

---

## 5. Standard Extension Patterns

### 5.1 The Layered Extension Pattern

Most Commerce projects follow this pattern:

```
┌─────────────────────────────┐
│  myprojectstorefront        │  Presentation layer
│  (web controllers, views)   │
├─────────────────────────────┤
│  myprojectfacades           │  Facade layer
│  (DTOs, converters,         │  (data transformation)
│   populators)               │
├─────────────────────────────┤
│  myprojectcore              │  Service & DAO layer
│  (services, DAOs,           │  (business logic)
│   type system, interceptors)│
├─────────────────────────────┤
│  myprojectinitialdata       │  Data setup
│  (ImpEx, system setup)      │
├─────────────────────────────┤
│  myprojectbackoffice        │  Backoffice customization
│  (widgets, custom UI)       │
└─────────────────────────────┘
```

### 5.2 Core Extension

Contains the data model and business logic:

```
myprojectcore/
├── resources/
│   ├── myprojectcore-items.xml      ← Type system
│   ├── myprojectcore-spring.xml     ← Services, DAOs
│   └── impex/
│       ├── essentialdata.impex
│       └── projectdata.impex
├── src/
│   └── com/mycompany/core/
│       ├── services/
│       ├── daos/
│       ├── interceptors/
│       ├── strategies/
│       └── events/
└── testsrc/
```

### 5.3 Facades Extension

Contains data transformation logic:

```
myprojectfacades/
├── resources/
│   └── myprojectfacades-spring.xml  ← Facades, converters
├── src/
│   └── com/mycompany/facades/
│       ├── MyProductFacade.java
│       ├── impl/
│       │   └── DefaultMyProductFacade.java
│       ├── converters/
│       │   └── MyProductConverter.java
│       └── populators/
│           ├── MyProductBasicPopulator.java
│           └── MyProductPricePopulator.java
└── testsrc/
```

### 5.4 Storefront Extension

Contains web controllers and views:

```
myprojectstorefront/
├── web/
│   ├── src/
│   │   └── com/mycompany/storefront/
│   │       ├── controllers/
│   │       │   ├── pages/
│   │       │   │   └── ProductPageController.java
│   │       │   └── misc/
│   │       ├── forms/
│   │       └── security/
│   ├── webroot/
│   │   ├── WEB-INF/
│   │   │   ├── web.xml
│   │   │   ├── tags/
│   │   │   └── views/
│   │   │       └── pages/
│   │   │           └── product/
│   │   │               └── productLayout.jsp
│   │   └── static/
│   │       ├── css/
│   │       └── js/
│   └── spring/
│       └── myprojectstorefront-web-spring.xml
└── resources/
    └── myprojectstorefront-spring.xml
```

### 5.5 Initial Data Extension

Contains setup and data import:

```
myprojectinitialdata/
├── resources/
│   ├── myprojectinitialdata-spring.xml
│   └── myprojectinitialdata/
│       └── import/
│           ├── coredata/
│           │   ├── common/
│           │   │   ├── essential-data.impex
│           │   │   ├── countries.impex
│           │   │   ├── currencies.impex
│           │   │   └── delivery-modes.impex
│           │   └── stores/mystore/
│           │       ├── store.impex
│           │       ├── site.impex
│           │       └── solr.impex
│           └── sampledata/
│               └── stores/mystore/
│                   ├── products.impex
│                   ├── categories.impex
│                   ├── prices.impex
│                   ├── media.impex
│                   └── customers.impex
└── src/
    └── com/mycompany/initialdata/
        └── setup/
            └── MyInitialDataSystemSetup.java
```

---

## 6. AddOns

### 6.1 What is an AddOn?

An AddOn is a special type of extension that **plugs into a storefront** extension. It can:
- Add/override JSP pages and tags
- Add/override CSS and JavaScript
- Add controllers and Spring beans
- Modify storefront behavior without modifying the storefront extension directly

### 6.2 AddOn Structure

```
myaddon/
├── acceleratoraddon/
│   ├── web/
│   │   ├── src/                      # Controllers
│   │   ├── webroot/
│   │   │   ├── WEB-INF/
│   │   │   │   ├── views/           # JSP pages (override storefront views)
│   │   │   │   └── tags/            # Custom tags
│   │   │   └── _ui/                 # Static resources
│   │   │       └── responsive/
│   │   │           ├── common/
│   │   │           │   ├── css/
│   │   │           │   └── js/
│   │   └── spring/
│   │       └── myaddon-web-spring.xml
│   └── messages/
│       └── myaddon.properties
├── resources/
│   ├── myaddon-items.xml
│   └── myaddon-spring.xml
├── src/                              # Backend code
└── extensioninfo.xml
```

### 6.3 Installing an AddOn

```bash
# Install addon to a storefront
ant addoninstall -Daddonnames="myaddon" -DaddonStorefront.yacceleratorstorefront="myprojectstorefront"

# Uninstall addon
ant addonuninstall -Daddonnames="myaddon" -DaddonStorefront.yacceleratorstorefront="myprojectstorefront"
```

### 6.4 How AddOns Work

When installed, AddOn resources are **copied** into the target storefront:

```
myprojectstorefront/
├── web/
│   └── addonsrc/myaddon/           ← AddOn Java sources
├── webroot/
│   └── WEB-INF/
│       └── views/
│           └── addons/myaddon/     ← AddOn JSP views
└── _ui/
    └── addons/myaddon/             ← AddOn static resources
```

### 6.5 Common SAP AddOns

| AddOn | Purpose |
|-------|---------|
| `captchaaddon` | CAPTCHA integration |
| `b2bpunchoutaddon` | B2B PunchOut support |
| `commerceorgaddon` | B2B organization management |
| `selectivecartsplitlistaddon` | Save for later functionality |
| `ysapproductconfigaddon` | Product configuration |
| `assistedservicestorefront` | Assisted Service Module |
| `smarteditaddon` | SmartEdit integration |
| `cmsoccaddon` | CMS OCC API extensions |
| `profiletagaddon` | Profile tagging |

### 6.6 AddOn vs. Extension Decision

| Criteria | Use Extension | Use AddOn |
|----------|--------------|-----------|
| Backend logic only | ✅ | ❌ |
| Storefront UI changes | ❌ | ✅ |
| Reusable across storefronts | ❌ | ✅ |
| Needs to be togglable | ❌ | ✅ |
| Data model changes | ✅ | ✅ (via parent extension) |

> **Note:** With the shift to headless (Spartacus/Composable Storefront), AddOns for storefront are becoming less relevant. New storefront customizations are done in the Angular/React frontend layer instead.

---

## 7. Extension Dependencies & Loading Order

### 7.1 Dependency Declaration

In `extensioninfo.xml`:
```xml
<requires-extension name="commerceservices"/>
<requires-extension name="catalog"/>
```

### 7.2 Loading Order

The platform resolves dependencies into a topological order:

```
1. platform
2. core (depends on: platform)
3. deliveryzone (depends on: core)
4. europe1 (depends on: core)
5. catalog (depends on: core, europe1)
6. commerceservices (depends on: catalog, europe1, ...)
7. myprojectcore (depends on: commerceservices)
8. myprojectfacades (depends on: myprojectcore)
9. myprojectstorefront (depends on: myprojectfacades)
```

### 7.3 Circular Dependencies

Circular dependencies are **NOT allowed** and will prevent system startup:

```
❌ extensionA requires extensionB
   extensionB requires extensionA
```

### 7.4 Extension Loading Verification

Check loading order in HAC: **Platform → Extensions**

Or via Groovy console:
```groovy
import de.hybris.platform.core.Registry

def extensions = Registry.getCurrentTenant()
    .getTenantSpecificExtensionNames()
extensions.each { println it }
```

---

## 8. Project Structure Best Practices

### 8.1 Recommended Project Layout

```
myproject/
├── core-customize/
│   ├── hybris/
│   │   ├── bin/
│   │   │   └── custom/
│   │   │       ├── myprojectcore/
│   │   │       ├── myprojectfacades/
│   │   │       ├── myprojectstorefront/
│   │   │       ├── myprojectinitialdata/
│   │   │       ├── myprojectbackoffice/
│   │   │       └── myprojectfulfilment/
│   │   └── config/
│   │       ├── local.properties
│   │       ├── localextensions.xml
│   │       └── cloud/
│   │           ├── common.properties
│   │           ├── persona/
│   │           │   ├── dev.properties
│   │           │   ├── stag.properties
│   │           │   └── prod.properties
│   │           └── solr/
│   └── manifest.json
├── js-storefront/                    # Spartacus / Composable Storefront
│   └── spartacusstore/
│       ├── src/
│       ├── package.json
│       └── angular.json
└── .gitignore
```

### 8.2 Extension Naming Conventions

| Extension | Purpose | Naming Pattern |
|-----------|---------|----------------|
| Core | Data model, services | `<project>core` |
| Facades | DTOs, converters | `<project>facades` |
| Storefront | Web controllers | `<project>storefront` |
| Initial data | System setup, ImpEx | `<project>initialdata` |
| Backoffice | Admin customization | `<project>backoffice` |
| OCC | REST API extensions | `<project>occ` |
| Fulfilment | Order processing | `<project>fulfilment` |
| Test | Test utilities | `<project>test` |

### 8.3 What NOT to Modify

**Never modify** SAP-delivered extensions directly:
- Don't edit files in `bin/modules/`, `bin/platform/`
- Use the alias/override pattern for services
- Use AddOns for storefront customization
- Use Backoffice configuration XML for Backoffice customization

---

## 9. Build Callbacks

### 9.1 buildcallbacks.xml

Custom build hooks for your extension:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project name="myprojectcore_buildcallbacks">

    <!-- Called before build -->
    <macrodef name="myprojectcore_before_build">
        <sequential>
            <echo message="Before building myprojectcore"/>
        </sequential>
    </macrodef>

    <!-- Called after build -->
    <macrodef name="myprojectcore_after_build">
        <sequential>
            <echo message="After building myprojectcore"/>
        </sequential>
    </macrodef>

    <!-- Called before clean -->
    <macrodef name="myprojectcore_before_clean">
        <sequential/>
    </macrodef>

    <!-- Called after clean -->
    <macrodef name="myprojectcore_after_clean">
        <sequential/>
    </macrodef>

</project>
```

---

## 10. External Dependencies (Maven)

### 10.1 external-dependencies.xml

Add third-party Maven dependencies:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.mycompany</groupId>
    <artifactId>myprojectcore</artifactId>
    <version>1.0</version>
    
    <dependencies>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>32.1.3-jre</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-csv</artifactId>
            <version>1.10.0</version>
        </dependency>
    </dependencies>
</project>
```

### 10.2 Resolving Dependencies

```bash
# Download dependencies to lib/ folder
ant clean all

# Or explicitly
ant -f myprojectcore/external-dependencies.xml resolve
```

Dependencies are downloaded to `myprojectcore/lib/`.

---

## 11. Exercises

### Exercise 1: Create a Core Extension

1. Using `ant extgen`, create a new extension called `trainingcore` using the `yempty` template
2. Add a dependency on `commerceservices`
3. Define a simple type in `trainingcore-items.xml`:
   ```xml
   <itemtype code="TrainingCourse" ...>
       <attributes>
           <attribute qualifier="code" .../>
           <attribute qualifier="title" .../>
           <attribute qualifier="durationHours" .../>
       </attributes>
   </itemtype>
   ```
4. Register the extension in `localextensions.xml`
5. Build and verify with `ant clean all`

### Exercise 2: Create a Facades Extension

1. Create `trainingfacades` extension
2. Add dependency on `trainingcore`
3. Create a `TrainingCourseData` DTO class
4. Create a `TrainingCourseFacade` interface and implementation
5. Create a `TrainingCoursePopulator` that converts `TrainingCourseModel` → `TrainingCourseData`
6. Wire everything in Spring XML

### Exercise 3: Understand Extension Loading

1. In HAC, navigate to **Platform → Extensions**
2. Find `commerceservices` and list its dependencies
3. Calculate the total number of extensions loaded in your system
4. Identify which extensions define web modules

### Exercise 4: Analyze a Standard Extension

1. Find the `commerceservices` extension in your installation
2. Open its `extensioninfo.xml` and document:
   - All declared dependencies
   - Whether it has a web module
   - The core module package root
3. Open its Spring XML and find:
   - How `cartService` is defined
   - What class implements it
   - What dependencies it has

### Exercise 5: External Dependencies

1. Add Apache Commons CSV as a dependency to your `trainingcore` extension
2. Build and verify the JAR appears in `lib/`
3. Write a simple service that reads CSV data and creates `TrainingCourse` items

---

## 12. Self-Check Questions

1. **What is the difference between `essentialdata` and `projectdata` ImpEx?**
   <details>
   <summary>Answer</summary>
   Both are loaded during initialization and update. `essentialdata` should contain fundamental data needed for the system to work (types, enums, necessary configuration). `projectdata` contains business data (catalogs, products, etc.). The distinction is organizational — both run during init/update.
   </details>

2. **How do you add a third-party JAR to an extension?**
   <details>
   <summary>Answer</summary>
   Either add it to `external-dependencies.xml` as a Maven dependency (preferred), or manually place the JAR in the extension's `lib/` directory.
   </details>

3. **What happens when you install an AddOn?**
   <details>
   <summary>Answer</summary>
   The AddOn's web resources (JSP, CSS, JS) are copied into the target storefront extension under `web/addonsrc/`, `webroot/WEB-INF/views/addons/`, and `_ui/addons/` directories. The AddOn's backend code is compiled and loaded normally.
   </details>

4. **Why should you never modify SAP-delivered extensions?**
   <details>
   <summary>Answer</summary>
   Because modifications would be overwritten during upgrades. Instead, use the alias/override pattern for services, AddOns for storefront changes, and custom extensions for new functionality.
   </details>

5. **What is the purpose of `extensioninfo.xml`?**
   <details>
   <summary>Answer</summary>
   It declares extension metadata: name, version, dependencies (requires-extension), modules (core module, web module), and configuration (jalo class, package root). The platform uses this to resolve loading order and manage the extension lifecycle.
   </details>

6. **Can an extension have both a core module and a web module?**
   <details>
   <summary>Answer</summary>
   Yes. The core module provides backend functionality (services, DAOs, type system) while the web module provides a web application (controllers, views). They have separate Spring contexts.
   </details>

---

## 📚 Further Reading

- [Creating a New Extension](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8ac59e8286691014afe5f1ef24321445.html)
- [AddOn Concept](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8acbae3f86691014895f8b06f1a5423c.html)
- [Extension Dependencies](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8b94a3c286691014afacb6e6bce25832.html)

---

**Previous Course**: [← 02 - Type System](../02-type-system/README.md)  
**Next Course**: [04 - Installation & Setup →](../04-installation/README.md)