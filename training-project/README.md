# SAP Commerce Training Project

## 🎯 Hands-On Training: Build an Electronics Store

This training project guides you through building a fully functional SAP Commerce B2C electronics store from scratch. Each phase maps to the course materials and builds on previous work.

**Total Duration**: 40-60 hours  
**Prerequisites**: Complete courses 01-06 minimum

---

## 📋 Project Overview

You will build **"TechMart"** — a B2C electronics webshop with:
- Custom product types (electronics with specifications)
- Product catalog with categories, images, prices
- Custom search with Solr facets
- Checkout with delivery and payment
- Customer reviews & ratings
- Backoffice management UI
- OCC REST API endpoints
- Composable Storefront (Spartacus)

---

## Phase 1: Environment Setup (4-6 hours)
*Maps to: [Course 04 - Installation](../courses/04-installation/README.md)*

### Steps

1. **Install SAP Commerce 2211**
   ```bash
   unzip CXCOMM2211.zip -d /opt/techmart
   cd /opt/techmart/hybris/bin/platform
   source ./setantenv.sh
   ```

2. **Run the B2C Accelerator recipe**
   ```bash
   cd /opt/techmart/installer
   ./install.sh -r b2c_acc
   ```

3. **Create project extensions**
   ```bash
   cd /opt/techmart/hybris/bin/platform
   
   # Core extension
   ant extgen -Dinput.template=yempty -Dinput.name=techmartcore -Dinput.package=com.techmart.core
   
   # Facades extension
   ant extgen -Dinput.template=yempty -Dinput.name=techmartfacades -Dinput.package=com.techmart.facades
   
   # Initial data extension
   ant extgen -Dinput.template=yempty -Dinput.name=techmartinitialdata -Dinput.package=com.techmart.initialdata
   
   # Storefront (or use Spartacus)
   ant extgen -Dinput.template=yempty -Dinput.name=techmartstorefront -Dinput.package=com.techmart.storefront
   
   # Backoffice
   ant extgen -Dinput.template=ybackoffice -Dinput.name=techmartbackoffice -Dinput.package=com.techmart.backoffice
   ```

4. **Configure database (MySQL)**
   ```sql
   CREATE DATABASE techmart CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER 'techmart'@'localhost' IDENTIFIED BY 'techmart123';
   GRANT ALL ON techmart.* TO 'techmart'@'localhost';
   ```

5. **Edit `config/local.properties`**
   ```properties
   db.url=jdbc:mysql://localhost:3306/techmart?useSSL=false&allowPublicKeyRetrieval=true
   db.driver=com.mysql.cj.jdbc.Driver
   db.username=techmart
   db.password=techmart123
   
   installed.tenants=
   build.development.mode=true
   tomcat.development.mode=true
   ```

6. **Edit `config/localextensions.xml`** — add all your extensions

7. **Build and initialize**
   ```bash
   ant clean all
   ant initialize
   ```

8. **Verify**: Access HAC (`https://localhost:9002/hac`), Backoffice, Storefront

### ✅ Checkpoint
- [ ] Server starts without errors
- [ ] HAC accessible with admin/nimda
- [ ] All custom extensions visible in Platform → Extensions
- [ ] IDE configured with remote debugging working

---

## Phase 2: Data Model (6-8 hours)
*Maps to: [Course 02 - Type System](../courses/02-type-system/README.md)*

### Steps

1. **Define custom types** in `techmartcore/resources/techmartcore-items.xml`:

```xml
<items xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="items.xsd">
    
    <enumtypes>
        <enumtype code="TechCategory" dynamic="true">
            <value code="LAPTOP"/>
            <value code="SMARTPHONE"/>
            <value code="TABLET"/>
            <value code="ACCESSORY"/>
            <value code="CAMERA"/>
        </enumtype>
        
        <enumtype code="WarrantyType">
            <value code="STANDARD"/>
            <value code="EXTENDED"/>
            <value code="PREMIUM"/>
        </enumtype>
    </enumtypes>
    
    <itemtypes>
        <!-- Electronics Product subtype -->
        <itemtype code="ElectronicsProduct" extends="Product"
                  jaloclass="com.techmart.core.jalo.ElectronicsProduct"
                  autocreate="true" generate="true">
            <attributes>
                <attribute qualifier="techCategory" type="TechCategory">
                    <persistence type="property"/>
                    <modifiers optional="false"/>
                </attribute>
                <attribute qualifier="screenSize" type="java.lang.Double">
                    <persistence type="property"/>
                    <description>Screen size in inches</description>
                </attribute>
                <attribute qualifier="batteryLife" type="java.lang.Integer">
                    <persistence type="property"/>
                    <description>Battery life in hours</description>
                </attribute>
                <attribute qualifier="processor" type="localized:java.lang.String">
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="ramGB" type="java.lang.Integer">
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="storageGB" type="java.lang.Integer">
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="weight" type="java.lang.Double">
                    <persistence type="property"/>
                    <description>Weight in grams</description>
                </attribute>
            </attributes>
        </itemtype>
        
        <!-- Warranty -->
        <itemtype code="ProductWarranty"
                  jaloclass="com.techmart.core.jalo.ProductWarranty"
                  autocreate="true" generate="true">
            <deployment table="productwarranties" typecode="25001"/>
            <attributes>
                <attribute qualifier="code" type="java.lang.String">
                    <persistence type="property"/>
                    <modifiers unique="true" optional="false"/>
                </attribute>
                <attribute qualifier="name" type="localized:java.lang.String">
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="warrantyType" type="WarrantyType">
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="durationMonths" type="java.lang.Integer">
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="price" type="java.lang.Double">
                    <persistence type="property"/>
                </attribute>
            </attributes>
        </itemtype>
        
        <!-- Customer Review extension -->
        <itemtype code="TechReview"
                  jaloclass="com.techmart.core.jalo.TechReview"
                  autocreate="true" generate="true">
            <deployment table="techreviews" typecode="25002"/>
            <attributes>
                <attribute qualifier="product" type="ElectronicsProduct">
                    <persistence type="property"/>
                    <modifiers optional="false"/>
                </attribute>
                <attribute qualifier="customer" type="Customer">
                    <persistence type="property"/>
                    <modifiers optional="false"/>
                </attribute>
                <attribute qualifier="rating" type="java.lang.Integer">
                    <persistence type="property"/>
                    <description>Rating 1-5</description>
                </attribute>
                <attribute qualifier="headline" type="java.lang.String">
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="comment" type="java.lang.String">
                    <persistence type="property">
                        <columntype database="mysql">
                            <value>TEXT</value>
                        </columntype>
                    </persistence>
                </attribute>
                <attribute qualifier="approved" type="java.lang.Boolean">
                    <persistence type="property"/>
                    <defaultvalue>Boolean.FALSE</defaultvalue>
                </attribute>
            </attributes>
        </itemtype>
    </itemtypes>
    
    <!-- Relations -->
    <relations>
        <relation code="ElectronicsProduct2Warranty" localized="false">
            <sourceElement type="ElectronicsProduct" qualifier="product" cardinality="one"/>
            <targetElement type="ProductWarranty" qualifier="warranties" cardinality="many" collectiontype="list"/>
        </relation>
    </relations>
    
</items>
```

2. **Build and update system**:
   ```bash
   ant clean all
   # Then in HAC → Platform → Update → check Update running system → Execute
   ```

3. **Verify in HAC**: FlexibleSearch → `SELECT {pk}, {code} FROM {ElectronicsProduct}`

### ✅ Checkpoint
- [ ] `ant clean all` compiles without errors
- [ ] System update succeeds
- [ ] `ElectronicsProductModel`, `ProductWarrantyModel`, `TechReviewModel` classes generated
- [ ] Types visible in Backoffice → System → Types

---

## Phase 3: Services & Business Logic (8-10 hours)
*Maps to: [Course 05 - Service Layer](../courses/05-service-layer/README.md)*

### Steps

1. **Create Service interfaces** in `techmartcore/src/`:
   - `ElectronicsProductService` — find products by tech category, screen size range
   - `ProductWarrantyService` — CRUD for warranties
   - `TechReviewService` — add/approve reviews, get average rating

2. **Create DAO implementations** using FlexibleSearch

3. **Create ValidateInterceptor** — rating must be 1-5, warranty duration > 0

4. **Create PrepareInterceptor** — auto-generate warranty codes if blank

5. **Register all beans** in `techmartcore-spring.xml`

6. **Create Facades** in `techmartfacades/src/`:
   - DTOs: `ElectronicsProductData`, `ProductWarrantyData`, `TechReviewData`
   - Populators for each conversion
   - Facade implementations

7. **Write unit tests** in `testsrc/`

### ✅ Checkpoint
- [ ] All services compile and are wired in Spring
- [ ] Interceptors fire correctly (test via HAC Groovy console)
- [ ] Facades convert models to DTOs correctly
- [ ] Unit tests pass: `ant unittests -Dtestclasses.packages=com.techmart.*`

---

## Phase 4: Data Import (4-6 hours)
*Maps to: [Course 06 - ImpEx](../courses/06-impex/README.md)*

### Steps

1. **Create essential data ImpEx** — enums, base types
2. **Create catalog structure** — product catalog, categories
3. **Create sample products** — 20+ electronics products with images, prices
4. **Create sample customers** — 5 test customers
5. **Create SystemSetup class** to load data on initialization

Sample product data:
```impex
$catalog=techmartProductCatalog
$catalogVersion=catalogversion(catalog(id[default=$catalog]),version[default='Staged'])[unique=true,default=$catalog:Staged]
$lang=en

INSERT_UPDATE ElectronicsProduct;code[unique=true];name[lang=$lang];techCategory(code);screenSize;ramGB;storageGB;$catalogVersion
;LAPTOP-001;ProBook Ultra 15;LAPTOP;15.6;16;512;
;LAPTOP-002;AirSlim 13;LAPTOP;13.3;8;256;
;PHONE-001;Galaxy Ultra X;SMARTPHONE;6.8;12;256;
;PHONE-002;iPhone ProMax;SMARTPHONE;6.7;8;512;
;TABLET-001;TabPro 12;TABLET;12.4;8;128;
;CAM-001;MirrorShot Alpha;CAMERA;;;
```

### ✅ Checkpoint
- [ ] `ant initialize` loads all data
- [ ] Products visible in Backoffice with correct attributes
- [ ] Categories, prices, and media assigned
- [ ] Catalog synchronization (Staged → Online) works

---

## Phase 5: Search & Solr (4-6 hours)
*Maps to: [Course 09 - Performance](../courses/09-performance/README.md)*

1. Configure Solr indexed type for `ElectronicsProduct`
2. Add facets: `techCategory`, `price`, `screenSize`, `ramGB`
3. Create custom value provider for `batteryLife` ranges
4. Run full index and verify in Solr Admin
5. Test search via OCC API

---

## Phase 6: OCC API & Storefront (8-10 hours)
*Maps to: [Course 07 - Storefront & OCC](../courses/07-storefront-occ/README.md)*

1. Create custom OCC endpoint for electronics product details (with specs)
2. Create OCC endpoint for product reviews (GET, POST)
3. Set up Spartacus storefront connected to your Commerce instance
4. Create custom Spartacus component for product specifications table
5. Test full flow: browse → search → PDP → add to cart → checkout

---

## Phase 7: Backoffice & Admin (4-6 hours)
*Maps to: [Course 12 - Backoffice & Security](../courses/12-backoffice-security/README.md)*

1. Add custom editor tab for `ElectronicsProduct` showing specifications
2. Add review management view with approve/reject actions
3. Create custom user group for product managers
4. Configure search restrictions

---

## Phase 8: Cloud Deployment (4-6 hours)
*Maps to: [Course 08 - Build & Deployment](../courses/08-build-deployment/README.md)*

1. Create `manifest.json` for CCv2
2. Set up cloud properties for dev/staging/prod
3. Create build and deploy to dev environment
4. Verify all functionality in cloud environment

---

## 🏆 Final Deliverables

After completing all phases, you should have:

| Deliverable | Description |
|-------------|-------------|
| Custom extensions | `techmartcore`, `techmartfacades`, `techmartinitialdata`, `techmartbackoffice` |
| Data model | `ElectronicsProduct`, `ProductWarranty`, `TechReview` with relations |
| Services | Full CRUD for all custom types with interceptors |
| Facades | DTOs, converters, populators |
| ImpEx | Complete data import for 20+ products |
| Solr | Full-text search with faceted filtering |
| OCC API | Custom REST endpoints |
| Storefront | Working Spartacus app |
| Backoffice | Custom editor views |
| Tests | Unit tests for all services |
| Cloud config | `manifest.json` ready for CCv2 |

---

## 💡 Tips

- **Save frequently** — use `ant all` for incremental builds (faster than `ant clean all`)
- **Debug in HAC** — use the Groovy console to test code snippets
- **Check logs** — `hybris/log/tomcat/console-YYYYMMDD.log`
- **ImpEx first** — always test ImpEx in HAC console before adding to initial data
- **Git commits** — commit after each successful phase checkpoint