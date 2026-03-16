# Course 12: Backoffice Customization & Security

## 🎯 Overview
Customize the Backoffice administration UI, configure security, user roles, access rights, and GDPR/data protection compliance.

**Duration**: 6-8 hours | **Level**: Advanced | **Prerequisites**: Courses 01-06

---

## Topics

### 1. Backoffice Framework
Backoffice is built on ZK framework (Java-based RIA):
- **Widgets** — UI components (editor areas, list views, charts)
- **Cockpit NG** — Configuration-driven UI framework
- **Application Orchestrator** — Drag & drop UI customization

### 2. Backoffice Configuration (backoffice-config.xml)
```xml
<!-- Custom editor area for Product -->
<context type="Product" component="editor-area" parent="Product">
    <editorArea:editorArea xmlns:editorArea="http://www.hybris.com/cockpitng/component/editorArea">
        <editorArea:tab name="hmc.properties" position="0">
            <editorArea:section name="hmc.basic">
                <editorArea:attribute qualifier="code" readonly="true"/>
                <editorArea:attribute qualifier="name"/>
                <editorArea:attribute qualifier="description" editor="com.hybris.cockpitng.editor.localized"/>
                <editorArea:attribute qualifier="myCustomField"/>
            </editorArea:section>
        </editorArea:tab>
    </editorArea:editorArea>
</context>

<!-- Custom list view -->
<context type="Product" component="listview">
    <list-view:list-view xmlns:list-view="http://www.hybris.com/cockpitng/component/listView">
        <list-view:column qualifier="code" width="150"/>
        <list-view:column qualifier="name" width="auto"/>
        <list-view:column qualifier="approvalStatus" width="120"/>
    </list-view:list-view>
</context>
```

### 3. Custom Backoffice Widgets
```java
@ViewEvent(componentID = "myWidget", eventName = "socket.input")
public void handleInput(WidgetInstanceManager widgetInstanceManager, Object input) {
    // Widget logic
    ProductModel product = (ProductModel) input;
    widgetInstanceManager.getModel().put("product", product);
}
```

Widget definition (`definition.xml`):
```xml
<widget-definition id="com.mycompany.mywidget" 
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <name>My Custom Widget</name>
    <controller class="com.mycompany.backoffice.widgets.MyWidgetController"/>
    <view src="mywidget.zul"/>
    <sockets>
        <input id="inputProduct" type="de.hybris.platform.core.model.product.ProductModel"/>
    </sockets>
</widget-definition>
```

### 4. User Roles & Access Rights
```impex
# User Groups / Roles
INSERT_UPDATE UserGroup;uid[unique=true];locname[lang=en];groups(uid)
;productmanagergroup;Product Managers;employeegroup

# Access Rights (Type-level)
INSERT_UPDATE SearchRestriction;code[unique=true];query;principal(uid);restrictedType(code);active;generate
;productmanager_restriction;{item:catalogVersion} IN ({{SELECT {cv.pk} FROM {CatalogVersion AS cv} WHERE {cv.catalog} = '8796093054977'}});productmanagergroup;Product;true;true

# Backoffice Access (UserGroup → Permissions)
$START_USERRIGHTS
Type;UID;MemberOfGroups;Password;Target;read;change;create;remove
UserGroup;productmanagergroup;;;;
;;;;Product;+;+;+;-
;;;;Category;+;+;-;-
;;;;Media;+;+;+;+
$END_USERRIGHTS
```

### 5. Search Restrictions
Limit what data users can see:
```impex
INSERT_UPDATE SearchRestriction;code[unique=true];query;principal(uid);restrictedType(code);active;generate
;online_products_only;{item:catalogVersion} = ({{SELECT {pk} FROM {CatalogVersion} WHERE {version}='Online'}});customergroup;Product;true;true
```

### 6. Spring Security Configuration
```xml
<!-- Custom authentication -->
<http pattern="/my-api/**" security="none"/>

<http pattern="/**">
    <csrf disabled="true"/>
    <intercept-url pattern="/admin/**" access="ROLE_ADMINGROUP"/>
    <intercept-url pattern="/**" access="ROLE_CUSTOMERGROUP"/>
</http>
```

### 7. Password & Authentication
```properties
# Password encoding
user.password.encoding=pbkdf2
password.encoder.pbkdf2.iterations=100000

# Account lockout
login.max.failed.attempts=5
login.lockout.duration=300

# Session management
session.timeout=1800
```

### 8. GDPR & Data Protection
```java
// Customer data erasure
customerAccountService.closeAccount(customerModel);
// Anonymizes personal data, retains order history

// Consent management
ConsentTemplateModel template = consentService.getLatestConsentTemplate("MARKETING");
ConsentModel consent = consentService.giveConsent(customer, template);
```

```impex
INSERT_UPDATE ConsentTemplate;id[unique=true];name;description;version;baseSite(uid)
;MARKETING;Marketing Consent;Allow marketing emails;1;mysite
;ANALYTICS;Analytics Consent;Allow analytics tracking;1;mysite
```

---

## Exercises
1. Add a custom tab to the Product editor in Backoffice showing warranty information
2. Create a custom user group with read-only access to products and no access to customers
3. Implement a search restriction that limits CSR agents to see only their assigned customers
4. Configure password policies: min 12 chars, lockout after 3 failures
5. Set up consent templates for GDPR compliance

## Self-Check

1. **How does Backoffice differ from HAC?**
   <details>
   <summary>Answer</summary>
   - **Backoffice** is the business user administration tool for managing products, orders, catalogs, customers, CMS content, and other business data. It has a rich, widget-based UI built on the ZK framework, supports customization (custom editors, widgets, workflows), and is role-based so different users see different views. Business users (merchandisers, customer service, content managers) use Backoffice daily.
   - **HAC** (Hybris Administration Console) is a developer/admin tool for technical operations: running ImpEx, FlexibleSearch, Groovy scripts, monitoring JVM/cache/threads, triggering system updates, managing CronJobs, and viewing logs. HAC should only be accessible to developers and system administrators.
   
   In short: Backoffice = business operations, HAC = technical operations.
   </details>

2. **What is a search restriction and how does it enforce data visibility?**
   <details>
   <summary>Answer</summary>
   A search restriction is a filter automatically appended to FlexibleSearch queries at runtime. It restricts which items a user or user group can see. For example:
   ```impex
   INSERT_UPDATE SearchRestriction; code[unique=true]; principal(uid); restrictedType(code); query; active
   ; customerOrderRestriction ; customergroup ; Order ; {user} = ?session.user ; true
   ```
   This ensures customers only see their own orders — whenever a FlexibleSearch for `Order` runs in the context of a user in `customergroup`, the clause `{user} = ?session.user` is automatically appended. Search restrictions enforce data-level security transparently — services and DAOs don't need to include these filters manually. They can be set per principal (user/group) and per type.
   </details>

3. **How do you add custom fields to the Backoffice editor?**
   <details>
   <summary>Answer</summary>
   Modify the Backoffice configuration XML (`backoffice-config.xml`) in your extension's `backoffice/resources/` folder:
   ```xml
   <context type="Product" component="editor-area" module="myextension">
       <editorArea:editorArea xmlns:editorArea="http://www.hybris.com/cockpitng/component/editorArea">
           <editorArea:tab name="hmc.properties" position="0">
               <editorArea:section name="hmc.basic">
                   <editorArea:attribute qualifier="myCustomAttribute"/>
               </editorArea:section>
           </editorArea:tab>
       </editorArea:editorArea>
   </context>
   ```
   The `qualifier` must match an attribute defined in `items.xml`. You can control positioning with `position`, group fields into sections, and create entirely new tabs. After changes, rebuild and restart (or use Backoffice's "Reset Everything" in Cockpit NG config).
   </details>

4. **What is the `$START_USERRIGHTS` / `$END_USERRIGHTS` block?**
   <details>
   <summary>Answer</summary>
   It's a special ImpEx syntax for managing access rights (permissions) in bulk:
   ```impex
   $START_USERRIGHTS
   Type      ; uid[unique=true] ; memberOfGroups ; password ; allTypes ; Product ; Order
   UserGroup ; myCustomGroup    ;                ;          ; +       ; +r+w    ; +r
   $END_USERRIGHTS
   ```
   This grants the `myCustomGroup` user group read+write access to `Product` and read-only access to `Order`. Permission flags:
   - `+r` = grant read, `-r` = deny read
   - `+w` = grant write (change), `-w` = deny write
   - `+c` = grant create, `+d` = grant delete
   - `+` = grant all CRUD
   
   This is more readable than creating individual `AccessRight` entries manually and is the standard way to configure permissions in Commerce projects.
   </details>

5. **How does Commerce handle GDPR data erasure?**
   <details>
   <summary>Answer</summary>
   SAP Commerce provides a GDPR framework with the `gdpr` and `customercleanup` extensions:
   - **Right to Erasure** — `CustomerCleanupCronJob` anonymizes customer data by replacing personal information (name, email, addresses) with anonymized values while retaining the record for order history integrity
   - **Consent Management** — `ConsentTemplate` and `ConsentModel` track customer consent for data processing. Consent can be given/withdrawn via the storefront
   - **Data Export** — customers can request a full export of their personal data (portability)
   - **Retention Rules** — configurable rules define how long personal data is retained before automatic cleanup
   - **Audit Trail** — changes to sensitive data are logged for compliance
   
   The framework doesn't delete records outright (to maintain referential integrity) — it anonymizes PII fields so the data is no longer personally identifiable.
   </details>

---
**Previous**: [← 11 - B2B Commerce](11-b2b-commerce.md)

---

## 🎓 Course Catalog Complete
Congratulations! You've completed all 12 courses. See the [Training Project](../training-project/README.md) for hands-on practice.