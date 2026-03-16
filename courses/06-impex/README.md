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
- What's the difference between INSERT and INSERT_UPDATE?
- How do you reference a composite key in ImpEx?
- What does `[unique=true]` do?
- How do you import media/images via ImpEx?
- What is a translator and when do you need one?

---
**Previous**: [← 05 - Service Layer](../05-service-layer/README.md) | **Next**: [07 - Storefront & OCC →](../07-storefront-occ/README.md)