# Course 06: ImpEx & Data Management

## 🎯 Overview
ImpEx is SAP Commerce's built-in import/export language. Master ImpEx syntax, macros, scripting, data import strategies, and cronjobs.

**Duration**: 6-8 hours | **Level**: Intermediate | **Prerequisites**: Courses 01-04

---

## Topics

### 1. ImpEx Syntax
```impex
# INSERT — create new items (fails if exists)
INSERT;Product;code[unique=true];name[lang=en];catalogVersion(catalog(id),version)
;;PROD-001;My Product;Default:Online

# INSERT_UPDATE — create or update
INSERT_UPDATE;Product;code[unique=true];name[lang=en];catalogVersion(catalog(id),version)
;;PROD-001;Updated Product;Default:Online

# UPDATE — update only (fails if not exists)
UPDATE;Product;code[unique=true];name[lang=en]
;;PROD-001;New Name

# REMOVE — delete items
REMOVE;Product;code[unique=true];catalogVersion(catalog(id),version)
;;PROD-001;Default:Online
```

### 2. Macros & Variables
```impex
$catalog=Default
$version=Online
$catalogVersion=catalogVersion(catalog(id),version)[unique=true,default=$catalog:$version]
$lang=en

INSERT_UPDATE Product;code[unique=true];name[lang=$lang];$catalogVersion
;PROD-001;My Product;
;PROD-002;Another Product;
```

### 3. Translators & Special Syntax
```impex
# Media import
INSERT_UPDATE Media;code[unique=true];@media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator];mime
;logo;jar:com.mycompany.setup.MySetup&/import/images/logo.png;image/png

# Collection/List
UPDATE;Product;code[unique=true];supercategories(code,$catalogVersion)
;;PROD-001;category1,category2,category3

# Boolean
;;myFlag;true

# Date
;;startDate;15.03.2024 08:00:00
```

### 4. Scripting in ImpEx
```impex
#% import de.hybris.platform.core.Registry;
INSERT_UPDATE Customer;uid[unique=true];name;password
#% beforeEach:
#% import org.apache.commons.lang3.StringUtils;
#% if (StringUtils.isBlank(line.get(1))) { line.put(1, "default@email.com"); }
;customer1;John Doe;1234
;customer2;;1234
```

### 5. Essential Data vs Project Data
| Type | When Loaded | Purpose |
|------|-------------|---------|
| Essential | Every init & update | Core types, enums, critical config |
| Project | Every init & update | Business data — catalogs, stores |
| Sample | Init only | Test/demo data |

### 6. CronJobs for Import/Export
```impex
INSERT_UPDATE ImpExImportCronJob;code[unique=true];job(code);fileURL
;myImportJob;impExImportJob;jar:com.mycompany&/import/data.impex

INSERT_UPDATE Trigger;cronJob(code);second;minute;hour;day;month;year;active
;myImportJob;0;0;3;-1;-1;-1;true
```

### 7. HAC ImpEx Console
- **Import**: Paste ImpEx → Execute
- **Export**: Write export script → Download results
- **Validation mode**: Test without committing

```impex
# Export example
"#% impex.setTargetFile(""products.csv"");"
INSERT_UPDATE Product;code;name[lang=en];price
"#% impex.exportItems(""SELECT {pk} FROM {Product}"", true);"
```

---

## Exercises
1. Write ImpEx to create a product catalog with 5 categories and 10 products
2. Create ImpEx with macros for multi-language product names (EN, DE, FR)
3. Write an export script that exports all customers with their addresses
4. Create a CronJob that imports product prices from a CSV file daily
5. Use scripting to generate sequential product codes during import

## Self-Check

1. **What's the difference between INSERT and INSERT_UPDATE?**
   <details>
   <summary>Answer</summary>
   `INSERT` creates new items only — if an item with the same unique keys already exists, it throws an error. `INSERT_UPDATE` creates the item if it doesn't exist, or updates it if it does (upsert). `UPDATE` only modifies existing items and fails if the item is not found. In practice, `INSERT_UPDATE` is the most commonly used mode because it's idempotent — safe to run multiple times.
   </details>

2. **How do you reference a composite key in ImpEx?**
   <details>
   <summary>Answer</summary>
   Use dot notation to traverse references. For example, `CatalogVersion` has a composite unique key of `catalog` + `version`:
   ```impex
   INSERT_UPDATE Product; code[unique=true]; catalogVersion(catalog(id), version)[unique=true]
   ; PROD-001 ; myStoreCatalog:Staged
   ```
   The parentheses define which attributes uniquely identify the referenced type. The colon `:` separates the values for the composite key.
   </details>

3. **What does `[unique=true]` do?**
   <details>
   <summary>Answer</summary>
   `[unique=true]` marks a column as part of the item's lookup key for ImpEx processing. During `INSERT_UPDATE`, the ImpEx engine uses all `[unique=true]` columns to find an existing item. If found, it updates; if not, it inserts. It does NOT create a database unique constraint — it's only used during ImpEx import resolution. Multiple columns can be marked `[unique=true]` to form a composite lookup key.
   </details>

4. **How do you import media/images via ImpEx?**
   <details>
   <summary>Answer</summary>
   Use the `@media` or `jar:` prefix to reference files. Two common approaches:
   ```impex
   # From a zip file uploaded with the ImpEx
   INSERT_UPDATE Media; code[unique=true]; @media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator]
   ; logo ; logo.png
   
   # From the classpath (extension resources)
   INSERT_UPDATE Media; code[unique=true]; @media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator]; mime
   ; logo ; jar:com.mycompany.constants.MyExtensionConstants&/myextension/import/images/logo.png ; image/png
   ```
   The media file must be accessible in the import context (zip, classpath, or hotfolder).
   </details>

5. **What is a translator and when do you need one?**
   <details>
   <summary>Answer</summary>
   A translator is a class that converts between ImpEx CSV text values and the actual Java/model attribute types. You need one when the default type conversion doesn't handle your case. Common scenarios: importing media binaries (`MediaDataTranslator`), importing special date formats, importing encrypted passwords, or handling custom attribute types. Custom translators implement `AbstractValueTranslator` and override `importValue()` / `exportValue()` methods.
   </details>

---
**Previous**: [← 05 - Service Layer](../05-service-layer/README.md) | **Next**: [07 - Storefront & OCC →](../07-storefront-occ/README.md)