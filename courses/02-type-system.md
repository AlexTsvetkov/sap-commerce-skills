# Course 02: SAP Commerce Type System & Data Modeling

## 🎯 Course Overview

The Type System is the heart of SAP Commerce. It defines the data model, generates Java classes, and manages database schema — all without writing SQL DDL. This course covers every aspect of the type system including item types, relation types, enum types, collection types, map types, and dynamic attributes.

**Duration**: 10-12 hours  
**Level**: Intermediate  
**Prerequisites**: [Course 01: Architecture](01-architecture.md), Java basics, XML

---

## 📋 Table of Contents

1. [Type System Overview](#1-type-system-overview)
2. [items.xml Structure](#2-itemsxml-structure)
3. [Item Types](#3-item-types)
4. [Attributes & Properties](#4-attributes--properties)
5. [Enum Types](#5-enum-types)
6. [Relation Types](#6-relation-types)
7. [Collection & Map Types](#7-collection--map-types)
8. [Atomic Types](#8-atomic-types)
9. [Dynamic Attributes](#9-dynamic-attributes)
10. [Model Generation & The Build Process](#10-model-generation--the-build-process)
11. [Deployment & Table Mapping](#11-deployment--table-mapping)
12. [Type System Best Practices](#12-type-system-best-practices)
13. [Exercises](#13-exercises)
14. [Self-Check Questions](#14-self-check-questions)

---

## 1. Type System Overview

### What is the Type System?

The Type System is SAP Commerce's **metadata-driven data model**. Instead of writing JPA entities or SQL DDL, you define your data model in XML files (`*-items.xml`), and the platform:

1. **Generates Java model classes** (POJOs with getters/setters)
2. **Creates/updates database tables** automatically
3. **Manages relationships** between types
4. **Handles localization** transparently
5. **Enforces constraints** and validation

### Type Hierarchy

```
                    ┌──────────┐
                    │   Type   │
                    └────┬─────┘
           ┌─────────────┼─────────────┐
           │             │             │
    ┌──────┴──────┐ ┌────┴────┐ ┌─────┴─────┐
    │ AtomicType  │ │EnumType │ │ComposedType│
    │             │ │         │ │            │
    │ java.lang.  │ │ Custom  │ │ ItemType   │
    │ String,     │ │ enums   │ │ RelationType│
    │ Integer...  │ │         │ │ MapType    │
    └─────────────┘ └─────────┘ │CollectionType│
                                └────────────┘
```

### Core Built-in Types

| Type | Description | Example |
|------|-------------|---------|
| `GenericItem` | Root of all item types | Base for everything |
| `Item` | Base item with PK | Foundation type |
| `Product` | Product data | Electronics, clothing |
| `Category` | Product categorization | Brands, collections |
| `Catalog` | Container for catalog versions | Product catalogs |
| `CatalogVersion` | Versioned catalog | Online, Staged |
| `User` | Base user type | Customers, employees |
| `Customer` | B2C customer (extends User) | End consumers |
| `Order` | Customer order | Purchase records |
| `Cart` | Shopping cart (extends Order) | Active carts |
| `Address` | Physical/digital address | Shipping, billing |
| `Media` | Binary content reference | Images, PDFs |

---

## 2. items.xml Structure

### File Location

Each extension can define its type system additions in `resources/<extension>-items.xml`.

### Basic Structure

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<items xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="items.xsd">

    <!-- Enum type definitions -->
    <enumtypes>
        <enumtype code="MyEnum" dynamic="true">
            <value code="VALUE1"/>
            <value code="VALUE2"/>
        </enumtype>
    </enumtypes>

    <!-- Item type definitions -->
    <itemtypes>
        <itemtype code="MyCustomType"
                  extends="GenericItem"
                  autocreate="true"
                  generate="true"
                  jaloclass="com.mycompany.jalo.MyCustomType">
            <deployment table="mycustomtypes" typecode="10001"/>
            <attributes>
                <attribute qualifier="code" type="java.lang.String">
                    <persistence type="property"/>
                    <modifiers read="true" write="true" optional="false" unique="true"/>
                </attribute>
            </attributes>
        </itemtype>
    </itemtypes>

    <!-- Relation definitions -->
    <relations>
        <relation code="MyRelation" localized="false" 
                  generate="true" autocreate="true">
            <sourceElement type="TypeA" qualifier="typeAs" 
                          cardinality="many" ordered="false"/>
            <targetElement type="TypeB" qualifier="typeBs" 
                          cardinality="many" ordered="false"/>
        </relation>
    </relations>

</items>
```

### Key XML Elements

| Element | Purpose |
|---------|---------|
| `<enumtypes>` | Define enumeration types |
| `<itemtypes>` | Define item types (entities) |
| `<relations>` | Define relationships between types |
| `<collectiontypes>` | Define typed collections |
| `<maptypes>` | Define typed maps |
| `<atomictypes>` | Define primitive type mappings |

---

## 3. Item Types

### 3.1 Defining a New Item Type

```xml
<itemtype code="Warranty"
          extends="GenericItem"
          autocreate="true"
          generate="true"
          jaloclass="com.mycompany.jalo.Warranty">
    
    <deployment table="warranties" typecode="10100"/>
    
    <attributes>
        <attribute qualifier="code" type="java.lang.String">
            <description>Unique warranty code</description>
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="false" unique="true"/>
        </attribute>
        
        <attribute qualifier="name" type="localized:java.lang.String">
            <description>Localized warranty name</description>
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
        
        <attribute qualifier="durationMonths" type="java.lang.Integer">
            <description>Warranty duration in months</description>
            <persistence type="property"/>
            <defaultvalue>Integer.valueOf(12)</defaultvalue>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
        
        <attribute qualifier="active" type="java.lang.Boolean">
            <persistence type="property"/>
            <defaultvalue>Boolean.TRUE</defaultvalue>
            <modifiers read="true" write="true"/>
        </attribute>
    </attributes>
</itemtype>
```

### 3.2 Extending an Existing Type

Adding attributes to existing types is common and non-intrusive:

```xml
<!-- Add a custom attribute to Product -->
<itemtype code="Product" autocreate="false" generate="false">
    <attributes>
        <attribute qualifier="warranty" type="Warranty">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
        
        <attribute qualifier="customField" type="java.lang.String">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
    </attributes>
</itemtype>
```

**Important flags when extending:**
- `autocreate="false"` — Do NOT create a new type (it already exists)
- `generate="false"` — Do NOT generate a new model class (it already exists)

### 3.3 Subtyping

Creating subtypes enables specialization:

```xml
<!-- Electronics product extends Product -->
<itemtype code="ElectronicsProduct"
          extends="Product"
          autocreate="true"
          generate="true"
          jaloclass="com.mycompany.jalo.ElectronicsProduct">
    
    <!-- No deployment needed — shares parent's table (single table inheritance) -->
    
    <attributes>
        <attribute qualifier="voltage" type="java.lang.String">
            <persistence type="property"/>
        </attribute>
        <attribute qualifier="wattage" type="java.lang.Double">
            <persistence type="property"/>
        </attribute>
    </attributes>
</itemtype>
```

### 3.4 Abstract Types

```xml
<itemtype code="AbstractDocument"
          extends="GenericItem"
          abstract="true"
          autocreate="true"
          generate="true"
          jaloclass="com.mycompany.jalo.AbstractDocument">
    <deployment table="documents" typecode="10101"/>
    <attributes>
        <attribute qualifier="documentId" type="java.lang.String">
            <persistence type="property"/>
            <modifiers optional="false" unique="true"/>
        </attribute>
    </attributes>
</itemtype>
```

### 3.5 The `typecode` and Deployment

Every item type needs a unique `typecode` (integer) for database mapping:

```xml
<deployment table="warranties" typecode="10100"/>
```

**Rules:**
- typecodes 0-9999 are reserved for SAP
- Custom types should use 10000+ 
- Once assigned, a typecode **cannot be changed** (it's embedded in PKs)
- The `table` attribute defines the database table name

---

## 4. Attributes & Properties

### 4.1 Attribute Types

| XML Type | Java Type | Description |
|----------|-----------|-------------|
| `java.lang.String` | `String` | Text field |
| `java.lang.Integer` | `Integer` | Integer number |
| `java.lang.Long` | `Long` | Long number |
| `java.lang.Double` | `Double` | Decimal number |
| `java.lang.Boolean` | `Boolean` | True/false |
| `java.util.Date` | `Date` | Date/time |
| `java.math.BigDecimal` | `BigDecimal` | Precise decimal (prices) |
| `localized:java.lang.String` | `String` (per locale) | Localized text |
| `Product` | `ProductModel` | Reference to another type |
| `MyEnumType` | `MyEnumType` (generated) | Enum value |

### 4.2 Attribute Modifiers

```xml
<modifiers read="true"        <!-- Generates getter -->
           write="true"       <!-- Generates setter -->
           optional="false"   <!-- NOT NULL constraint -->
           unique="true"      <!-- UNIQUE constraint -->
           initial="true"     <!-- Only writable at creation time -->
           search="true"      <!-- Indexed for search -->
           private="true"     <!-- Not accessible from outside -->
           encrypted="true"   <!-- Stored encrypted -->
           removable="true"   <!-- Can be set to null -->
/>
```

### 4.3 Persistence Types

```xml
<!-- Stored as a column in the type's table -->
<persistence type="property"/>

<!-- Stored in a separate attribute table (for large text/blobs) -->
<persistence type="property">
    <columntype database="oracle">
        <value>CLOB</value>
    </columntype>
    <columntype database="mysql">
        <value>TEXT</value>
    </columntype>
    <columntype>
        <value>HYBRIS.LONG_STRING</value>
    </columntype>
</persistence>

<!-- Calculated dynamically — not stored in DB -->
<persistence type="dynamic" attributeHandler="myDynamicHandler"/>

<!-- Stored in jalo layer (legacy — avoid in new code) -->
<persistence type="jalo"/>
```

### 4.4 Localized Attributes

Localized attributes store different values per language:

```xml
<attribute qualifier="description" type="localized:java.lang.String">
    <persistence type="property"/>
</attribute>
```

In Java:
```java
// Set localized values
product.setDescription("English description", Locale.ENGLISH);
product.setDescription("Deutsche Beschreibung", Locale.GERMAN);

// Get localized value (uses session language)
String desc = product.getDescription();

// Get specific locale
String descEn = product.getDescription(Locale.ENGLISH);
```

**How it works internally:**
- Localized values are stored in a separate `*lp` table (e.g., `productslp`)
- The platform transparently joins the tables

### 4.5 Default Values

```xml
<attribute qualifier="status" type="OrderStatus">
    <defaultvalue>em().getEnumerationValue("OrderStatus","CREATED")</defaultvalue>
</attribute>

<attribute qualifier="quantity" type="java.lang.Integer">
    <defaultvalue>Integer.valueOf(1)</defaultvalue>
</attribute>

<attribute qualifier="active" type="java.lang.Boolean">
    <defaultvalue>Boolean.TRUE</defaultvalue>
</attribute>

<attribute qualifier="creationDate" type="java.util.Date">
    <defaultvalue>new java.util.Date()</defaultvalue>
</attribute>
```

### 4.6 Custom Indexes

```xml
<itemtype code="MyType" ...>
    <deployment table="mytypes" typecode="10102"/>
    <attributes>
        <attribute qualifier="code" type="java.lang.String">
            <persistence type="property"/>
        </attribute>
        <attribute qualifier="catalogVersion" type="CatalogVersion">
            <persistence type="property"/>
        </attribute>
    </attributes>
    <indexes>
        <index name="codeVersionIdx" unique="true">
            <key attribute="code"/>
            <key attribute="catalogVersion"/>
        </index>
        <index name="codeIdx">
            <key attribute="code"/>
        </index>
    </indexes>
</itemtype>
```

---

## 5. Enum Types

### 5.1 Static Enums

Static enums generate Java enum classes and cannot be modified at runtime:

```xml
<enumtype code="ProductCondition" generate="true" autocreate="true"
          dynamic="false">
    <value code="NEW"/>
    <value code="USED"/>
    <value code="REFURBISHED"/>
</enumtype>
```

Generated Java:
```java
public enum ProductCondition implements HybrisEnumValue {
    NEW, USED, REFURBISHED;
}
```

### 5.2 Dynamic Enums

Dynamic enums can be extended at runtime (via ImpEx or Backoffice):

```xml
<enumtype code="ApprovalStatus" generate="true" autocreate="true"
          dynamic="true">
    <value code="PENDING"/>
    <value code="APPROVED"/>
    <value code="REJECTED"/>
</enumtype>
```

**Key difference:** Dynamic enum values are stored in the database and can be added without redeployment.

### 5.3 Using Enums

In `items.xml`:
```xml
<attribute qualifier="condition" type="ProductCondition">
    <persistence type="property"/>
</attribute>
```

In Java:
```java
product.setCondition(ProductCondition.NEW);

if (ProductCondition.REFURBISHED.equals(product.getCondition())) {
    // Handle refurbished logic
}
```

---

## 6. Relation Types

Relations define associations between item types.

### 6.1 One-to-Many Relation

```xml
<relation code="Category2ProductRelation" 
          localized="false" 
          generate="true" 
          autocreate="true">
    <sourceElement type="Category" 
                   qualifier="category" 
                   cardinality="one">
        <modifiers read="true" write="true" optional="true"/>
    </sourceElement>
    <targetElement type="Product" 
                   qualifier="products" 
                   cardinality="many"
                   collectiontype="list"
                   ordered="true">
        <modifiers read="true" write="true" optional="true"/>
    </targetElement>
</relation>
```

This generates:
```java
// On CategoryModel
List<ProductModel> getProducts();
void setProducts(List<ProductModel> products);

// On ProductModel
CategoryModel getCategory();
void setCategory(CategoryModel category);
```

**Database implementation:** A foreign key column is added to the `products` table pointing to the `categories` table.

### 6.2 Many-to-Many Relation

```xml
<relation code="Product2WarrantyRelation"
          localized="false"
          generate="true"
          autocreate="true">
    <deployment table="prod2warranty" typecode="10103"/>
    <sourceElement type="Product"
                   qualifier="products"
                   cardinality="many"
                   ordered="false">
    </sourceElement>
    <targetElement type="Warranty"
                   qualifier="warranties"
                   cardinality="many"
                   ordered="false">
    </targetElement>
</relation>
```

**Key:** Many-to-many relations require their own `<deployment>` (link table).

This generates:
```java
// On ProductModel
Collection<WarrantyModel> getWarranties();
void setWarranties(Collection<WarrantyModel> warranties);

// On WarrantyModel
Collection<ProductModel> getProducts();
void setProducts(Collection<ProductModel> products);
```

### 6.3 One-to-One Relation

```xml
<relation code="User2DefaultAddressRelation"
          localized="false"
          generate="true"
          autocreate="true">
    <sourceElement type="User"
                   qualifier="owner"
                   cardinality="one"/>
    <targetElement type="Address"
                   qualifier="defaultAddress"
                   cardinality="one"/>
</relation>
```

### 6.4 Ordered Relations

When `ordered="true"`, the platform maintains an ordering column:

```xml
<targetElement type="Product" 
               qualifier="products" 
               cardinality="many"
               collectiontype="list"
               ordered="true"/>
```

- Use `collectiontype="list"` with `ordered="true"` for ordered results
- Use `collectiontype="set"` or `collectiontype="collection"` for unordered

### 6.5 Relation vs. Attribute Reference

| Approach | When to Use |
|----------|-------------|
| **Relation** | Bidirectional navigation needed, proper FK constraints, many-to-many |
| **Attribute reference** | Simple unidirectional reference, no need for reverse navigation |

```xml
<!-- Attribute reference (unidirectional) -->
<attribute qualifier="defaultWarranty" type="Warranty">
    <persistence type="property"/>
</attribute>

<!-- Relation (bidirectional) — prefer for important domain relationships -->
<relation code="Product2WarrantyRelation" ...>
```

---

## 7. Collection & Map Types

### 7.1 Collection Types

Collection types define typed collections for use in attributes:

```xml
<collectiontypes>
    <collectiontype code="StringCollection" 
                    elementtype="java.lang.String"
                    autocreate="true"
                    generate="false"
                    type="collection"/>
    
    <collectiontype code="MediaList" 
                    elementtype="Media"
                    autocreate="true"
                    generate="false"
                    type="list"/>
</collectiontypes>
```

Usage:
```xml
<attribute qualifier="tags" type="StringCollection">
    <persistence type="property"/>
</attribute>
```

**⚠️ Warning:** Collection type attributes serialize data into a single database column (BLOB). This is:
- **Not searchable** via FlexibleSearch
- **Not performant** for large collections
- **Prefer Relations** for type-safe, searchable associations

### 7.2 Map Types

```xml
<maptypes>
    <maptype code="StringToStringMap"
             argumenttype="java.lang.String"
             returntype="java.lang.String"
             autocreate="true"
             generate="false"/>
</maptypes>
```

Usage:
```xml
<attribute qualifier="metadata" type="StringToStringMap">
    <persistence type="property"/>
</attribute>
```

In Java:
```java
Map<String, String> metadata = product.getMetadata();
metadata.put("origin", "Germany");
product.setMetadata(metadata);
modelService.save(product);
```

---

## 8. Atomic Types

Atomic types map Java primitive types to database column types:

```xml
<atomictypes>
    <atomictype code="java.lang.String" class="java.lang.String"/>
    <atomictype code="java.lang.Integer" class="java.lang.Integer"/>
    <atomictype code="java.lang.Boolean" class="java.lang.Boolean"/>
    <atomictype code="java.util.Date" class="java.util.Date"/>
    <atomictype code="java.math.BigDecimal" class="java.math.BigDecimal"/>
</atomictypes>
```

Most atomic types are pre-defined. You rarely need to create custom atomic types.

---

## 9. Dynamic Attributes

Dynamic attributes are **computed at runtime** and NOT stored in the database.

### 9.1 Defining a Dynamic Attribute

In `items.xml`:
```xml
<attribute qualifier="fullName" type="java.lang.String">
    <persistence type="dynamic" attributeHandler="customerFullNameHandler"/>
    <modifiers read="true" write="false"/>
</attribute>
```

### 9.2 Implementing the Handler

```java
public class CustomerFullNameHandler implements DynamicAttributeHandler<String, CustomerModel> {

    @Override
    public String get(CustomerModel model) {
        return String.format("%s %s", 
            model.getFirstName() != null ? model.getFirstName() : "",
            model.getLastName() != null ? model.getLastName() : "").trim();
    }

    @Override
    public void set(CustomerModel model, String value) {
        throw new UnsupportedOperationException("fullName is read-only");
    }
}
```

### 9.3 Registering the Handler

In `*-spring.xml`:
```xml
<bean id="customerFullNameHandler"
      class="com.mycompany.core.attributes.CustomerFullNameHandler"/>
```

### 9.4 Use Cases for Dynamic Attributes

- **Computed values**: Full name from first/last, formatted price, age from birth date
- **Aggregations**: Total order count, average rating
- **Cross-system lookups**: Fetch data from external systems (use with caution — performance impact)
- **Business rules**: Eligibility flags, calculated status

---

## 10. Model Generation & The Build Process

### 10.1 How Models are Generated

```
items.xml → ant all → Generated Model Classes → Compiled to JAR
```

When you run `ant all` (or `ant clean all`):

1. Platform reads all `*-items.xml` files from all extensions
2. Merges them into a complete type system definition
3. Generates Java model classes in `bootstrap/gensrc/`
4. Compiles generated code along with your custom code

### 10.2 Generated Model Example

For the `Warranty` type defined earlier, the platform generates:

```java
// Generated — DO NOT EDIT
public class WarrantyModel extends ItemModel {
    
    public static final String CODE = "code";
    public static final String NAME = "name";
    public static final String DURATIONMONTHS = "durationMonths";
    public static final String ACTIVE = "active";
    
    public String getCode() {
        return getPersistenceContext().getPropertyValue(CODE);
    }
    
    public void setCode(String value) {
        getPersistenceContext().setPropertyValue(CODE, value);
    }
    
    public String getName() {
        return getName(null);
    }
    
    public String getName(Locale loc) {
        return getPersistenceContext().getLocalizedValue(NAME, loc);
    }
    
    public void setName(String value) {
        setName(value, null);
    }
    
    public void setName(String value, Locale loc) {
        getPersistenceContext().setLocalizedValue(NAME, loc, value);
    }
    
    public Integer getDurationMonths() {
        return getPersistenceContext().getPropertyValue(DURATIONMONTHS);
    }
    
    // ... more getters/setters
}
```

### 10.3 Important Build Commands

```bash
# Full build — generates models, compiles everything
ant clean all

# Generate models only (without full compilation)
ant generateModels

# Update the running system (apply type system changes)
ant updatesystem

# Initialize the system (DESTROYS all data)
ant initialize
```

---

## 11. Deployment & Table Mapping

### 11.1 Single Table Inheritance

By default, subtypes share the parent's table:

```xml
<!-- Product table stores Product, ElectronicsProduct, ApparelProduct -->
<itemtype code="ElectronicsProduct" extends="Product" ...>
    <!-- No deployment tag = inherits parent's table -->
</itemtype>
```

A discriminator column (`TypePkString`) identifies the actual type.

### 11.2 Own Table Deployment

For new root types, you must specify a deployment:

```xml
<itemtype code="Warranty" extends="GenericItem" ...>
    <deployment table="warranties" typecode="10100"/>
</itemtype>
```

### 11.3 TypeCode Rules

| Range | Usage |
|-------|-------|
| 0 - 9,999 | Reserved for SAP platform and commerce modules |
| 10,000 - 32,767 | Available for custom types |

**Critical:** Once a typecode is assigned and data exists, it **CANNOT be changed**. Plan your typecodes carefully.

### 11.4 Database Schema Example

For the `Warranty` type:

```sql
CREATE TABLE warranties (
    hjmpTS        BIGINT,
    PK            BIGINT NOT NULL,
    TypePkString  BIGINT,
    p_code        VARCHAR(255),
    p_durationmonths INTEGER,
    p_active      TINYINT,
    -- ... system columns
    PRIMARY KEY (PK)
);

-- Localized properties table
CREATE TABLE warrantieslp (
    ITEMPK    BIGINT NOT NULL,
    LANGPK    BIGINT NOT NULL,
    p_name    VARCHAR(255),
    PRIMARY KEY (ITEMPK, LANGPK)
);
```

---

## 12. Type System Best Practices

### ✅ Do

1. **Use meaningful typecodes** — Organize in ranges (10000-10099 for products, 10100-10199 for orders, etc.)
2. **Use `optional="false"`** for truly required fields — Enforces NOT NULL at DB level
3. **Use Relations** for entity associations — Not collection type attributes
4. **Use localized attributes** for user-facing text — `localized:java.lang.String`
5. **Use dynamic attributes** for computed values — Don't store what you can calculate
6. **Add indexes** for frequently queried attributes
7. **Use `unique="true"`** for natural keys (e.g., `code` + `catalogVersion`)
8. **Extend existing types** rather than creating parallel hierarchies
9. **Use `initial="true"`** for attributes that should only be set once (like `code`)

### ❌ Don't

1. **Don't change typecodes** after deployment to production
2. **Don't use collection type attributes** for searchable data (serialized to BLOB)
3. **Don't create deep inheritance hierarchies** — Increases table size due to single-table inheritance
4. **Don't use `jalo` persistence** — It's legacy; use `property` or `dynamic`
5. **Don't forget `autocreate="false"` and `generate="false"`** when extending existing types
6. **Don't store large text in regular properties** — Use `HYBRIS.LONG_STRING` column type
7. **Don't remove attributes from items.xml** — This can break existing data. Instead, deprecate them

---

## 13. Exercises

### Exercise 1: Define a Custom Type

Create a `LoyaltyProgram` item type with:
- `code` (String, required, unique)
- `name` (localized String)
- `pointsMultiplier` (Double, default 1.0)
- `active` (Boolean, default true)
- `startDate` (Date)
- `endDate` (Date)

Write the complete `items.xml` definition.

<details>
<summary>Solution</summary>

```xml
<itemtype code="LoyaltyProgram"
          extends="GenericItem"
          autocreate="true"
          generate="true"
          jaloclass="com.mycompany.jalo.LoyaltyProgram">
    <deployment table="loyaltyprograms" typecode="10200"/>
    <attributes>
        <attribute qualifier="code" type="java.lang.String">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="false" unique="true" initial="true"/>
        </attribute>
        <attribute qualifier="name" type="localized:java.lang.String">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
        <attribute qualifier="pointsMultiplier" type="java.lang.Double">
            <persistence type="property"/>
            <defaultvalue>Double.valueOf(1.0)</defaultvalue>
            <modifiers read="true" write="true"/>
        </attribute>
        <attribute qualifier="active" type="java.lang.Boolean">
            <persistence type="property"/>
            <defaultvalue>Boolean.TRUE</defaultvalue>
            <modifiers read="true" write="true"/>
        </attribute>
        <attribute qualifier="startDate" type="java.util.Date">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
        <attribute qualifier="endDate" type="java.util.Date">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
    </attributes>
</itemtype>
```
</details>

### Exercise 2: Create a Relation

Define a many-to-many relation between `Customer` and `LoyaltyProgram`.

<details>
<summary>Solution</summary>

```xml
<relation code="Customer2LoyaltyProgramRelation"
          localized="false"
          generate="true"
          autocreate="true">
    <deployment table="cust2loyalty" typecode="10201"/>
    <sourceElement type="Customer"
                   qualifier="customers"
                   cardinality="many"
                   ordered="false"/>
    <targetElement type="LoyaltyProgram"
                   qualifier="loyaltyPrograms"
                   cardinality="many"
                   ordered="false"/>
</relation>
```
</details>

### Exercise 3: Extend Product with Custom Attributes

Add the following to the `Product` type:
- `warrantyDuration` (Integer)
- `technicalSpecs` (localized long String)
- `condition` enum (NEW, USED, REFURBISHED)

<details>
<summary>Solution</summary>

```xml
<enumtype code="ProductCondition" generate="true" autocreate="true" dynamic="false">
    <value code="NEW"/>
    <value code="USED"/>
    <value code="REFURBISHED"/>
</enumtype>

<itemtype code="Product" autocreate="false" generate="false">
    <attributes>
        <attribute qualifier="warrantyDuration" type="java.lang.Integer">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
        <attribute qualifier="technicalSpecs" type="localized:java.lang.String">
            <persistence type="property">
                <columntype>
                    <value>HYBRIS.LONG_STRING</value>
                </columntype>
            </persistence>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
        <attribute qualifier="condition" type="ProductCondition">
            <persistence type="property"/>
            <modifiers read="true" write="true" optional="true"/>
        </attribute>
    </attributes>
</itemtype>
```
</details>

### Exercise 4: Implement a Dynamic Attribute

Create a dynamic attribute `isExpired` on `LoyaltyProgram` that returns `true` if `endDate` is in the past.

<details>
<summary>Solution</summary>

items.xml:
```xml
<itemtype code="LoyaltyProgram" autocreate="false" generate="false">
    <attributes>
        <attribute qualifier="isExpired" type="java.lang.Boolean">
            <persistence type="dynamic" attributeHandler="loyaltyProgramExpiredHandler"/>
            <modifiers read="true" write="false"/>
        </attribute>
    </attributes>
</itemtype>
```

Java handler:
```java
public class LoyaltyProgramExpiredHandler 
    implements DynamicAttributeHandler<Boolean, LoyaltyProgramModel> {

    @Override
    public Boolean get(LoyaltyProgramModel model) {
        if (model.getEndDate() == null) {
            return Boolean.FALSE;
        }
        return model.getEndDate().before(new Date());
    }

    @Override
    public void set(LoyaltyProgramModel model, Boolean value) {
        throw new UnsupportedOperationException("isExpired is read-only");
    }
}
```

Spring XML:
```xml
<bean id="loyaltyProgramExpiredHandler"
      class="com.mycompany.core.attributes.LoyaltyProgramExpiredHandler"/>
```
</details>

### Exercise 5: Analyze Existing Type System

Using HAC:
1. Navigate to **Console → FlexibleSearch**
2. Run: `SELECT {pk}, {code}, {InternalCode} FROM {ComposedType} WHERE {code} LIKE '%Product%'`
3. List all product-related types in the system
4. For `Product` type, list all its attributes using: `SELECT {pk}, {qualifier}, {attributeType} FROM {AttributeDescriptor} WHERE {enclosingType} = (SELECT {pk} FROM {ComposedType} WHERE {code} = 'Product')`

---

## 14. Self-Check Questions

1. **What is the difference between `autocreate="true"` and `autocreate="false"`?**
   <details>
   <summary>Answer</summary>
   `autocreate="true"` creates a new type during system initialization/update. `autocreate="false"` means the type already exists (defined in another extension) and you're only modifying it (e.g., adding attributes).
   </details>

2. **When do you need a `<deployment>` tag?**
   <details>
   <summary>Answer</summary>
   When creating a new root item type (extends GenericItem or Item) that needs its own database table, or for many-to-many relations that need a link table.
   </details>

3. **What's the difference between a static and dynamic enum?**
   <details>
   <summary>Answer</summary>
   Static enums (`dynamic="false"`) generate Java enum classes and their values are fixed at compile time. Dynamic enums (`dynamic="true"`) store values in the database and can be extended at runtime via ImpEx or Backoffice.
   </details>

4. **Why should you avoid collection type attributes?**
   <details>
   <summary>Answer</summary>
   Collection type attributes are serialized into a single BLOB column in the database, making them not searchable via FlexibleSearch, not indexable, and potentially causing performance issues with large collections. Use relations instead.
   </details>

5. **How are localized attributes stored in the database?**
   <details>
   <summary>Answer</summary>
   In a separate `*lp` (localized properties) table with a composite key of (item PK, language PK). For example, Product localized attributes go to the `productslp` table.
   </details>

6. **What happens if you remove an attribute from items.xml?**
   <details>
   <summary>Answer</summary>
   The column remains in the database but the getter/setter is removed from the model class. Existing data is preserved. This is why you should deprecate attributes rather than remove them — removing can break FlexibleSearch queries and ImpEx scripts that reference the attribute.
   </details>

7. **What is single-table inheritance in Commerce?**
   <details>
   <summary>Answer</summary>
   When a subtype doesn't have its own `<deployment>` tag, it shares the parent type's database table. A discriminator column (TypePkString) identifies which subtype each row belongs to. All attributes from all subtypes are columns in the same table.
   </details>

---

## 📚 Further Reading

- [Type System Documentation](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8bcf8e0286691014b0fcc17fbb1b0855.html)
- [items.xml Reference](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8bc396b186691014a70cdfca7553c9fb.html)
- [Data Modeling Best Practices](https://help.sap.com/docs/SAP_COMMERCE/d0224eca81e249cb821f2cdf45a82ace/8bbf79d886691014a30ab9c8ee4da3d4.html)

---

**Previous Course**: [← 01 - Architecture](01-architecture.md)  
**Next Course**: [03 - Extensions & AddOns →](03-extensions.md)