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
- How does Backoffice differ from HAC?
- What is a search restriction and how does it enforce data visibility?
- How do you add custom fields to the Backoffice editor?
- What is the `$START_USERRIGHTS` / `$END_USERRIGHTS` block?
- How does Commerce handle GDPR data erasure?

---
**Previous**: [← 11 - B2B Commerce](../11-b2b-commerce/README.md)

---

## 🎓 Course Catalog Complete
Congratulations! You've completed all 12 courses. See the [Training Project](../../training-project/README.md) for hands-on practice.