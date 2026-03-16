# Course 07: Storefront & OCC API

## 🎯 Overview
Covers SAP Commerce storefront options: Accelerator (JSP), Composable Storefront (Spartacus/Angular), and the OCC (Omni Commerce Connect) REST API that powers headless commerce.

**Duration**: 8-10 hours | **Level**: Intermediate-Advanced | **Prerequisites**: Courses 01-06

---

## Topics

### 1. Storefront Evolution
| Generation | Technology | Status |
|-----------|-----------|--------|
| Accelerator | JSP + Spring MVC | Legacy (still supported) |
| Spartacus / Composable Storefront | Angular | Current recommended |
| Headless via OCC | REST API + any frontend | Current recommended |

### 2. OCC REST API
OCC is the RESTful API layer for SAP Commerce, exposing all commerce functionality.

**Base URL**: `https://{host}/occ/v2/{baseSiteId}/`

Key endpoints:
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/products/{code}` | GET | Get product details |
| `/products/search` | GET | Search products |
| `/users/{userId}/carts` | POST | Create cart |
| `/users/{userId}/carts/{cartId}/entries` | POST | Add to cart |
| `/users/{userId}/orders` | POST | Place order |
| `/catalogs` | GET | Get catalogs |
| `/categories/{categoryId}` | GET | Get category |

**Authentication**: OAuth 2.0 with client credentials or password grant.

```bash
# Get OAuth token
curl -X POST https://localhost:9002/authorizationserver/oauth/token \
  -d "client_id=mobile_android&client_secret=secret&grant_type=password&username=user@example.com&password=1234"

# Call OCC API
curl -H "Authorization: Bearer {token}" \
  https://localhost:9002/occ/v2/electronics/products/search?query=camera
```

### 3. Extending OCC
```java
@Controller
@RequestMapping("/{baseSiteId}/myresource")
@Api(tags = "My Custom Resource")
public class MyCustomController extends BaseController {
    
    @Autowired
    private MyFacade myFacade;
    
    @RequestMapping(value = "/{code}", method = RequestMethod.GET)
    @ResponseBody
    public MyData getByCode(@PathVariable String code) {
        return myFacade.getByCode(code);
    }
}
```

Register in `web/spring/` of your OCC extension:
```xml
<context:component-scan base-package="com.mycompany.occ.controllers"/>
```

### 4. Composable Storefront (Spartacus)
- Angular-based SPA consuming OCC API
- Server-Side Rendering (SSR) with Angular Universal
- CMS-driven layout via SmartEdit
- Extensible via feature modules and custom components

```bash
# Create new Spartacus app
ng new mystore --style=scss
cd mystore
ng add @spartacus/schematics --baseUrl=https://localhost:9002 --baseSite=electronics
```

### 5. CMS Integration
Commerce uses a CMS system for page layouts:
- **Pages** — Templates for URL patterns
- **Slots** — Regions on a page
- **Components** — Content blocks in slots
- **SmartEdit** — WYSIWYG editor for content managers

```impex
INSERT_UPDATE ContentPage;uid[unique=true];label;masterTemplate(uid,$contentCV);$contentCV
;myCustomPage;/my-custom-page;ProductPage2Template;

INSERT_UPDATE ContentSlot;uid[unique=true];cmsComponents(uid,$contentCV);$contentCV
;MyCustomPageSlot;MyBannerComponent,MyProductListComponent;
```

### 6. Accelerator Storefront (Legacy)
JSP-based storefront with Spring MVC controllers:
```java
@Controller
@RequestMapping("/my-page")
public class MyPageController extends AbstractPageController {
    @RequestMapping(method = RequestMethod.GET)
    public String showPage(Model model) {
        storeCmsPageInModel(model, getCmsPage("myCustomPage"));
        return getViewForPage(model);
    }
}
```

---

## Exercises
1. Call the OCC API to search products, create a cart, add products, and place an order using curl/Postman
2. Create a custom OCC endpoint that returns product recommendations
3. Set up a Spartacus storefront connected to your local Commerce instance
4. Create a custom CMS component and render it in Spartacus
5. Configure OAuth2 client credentials for a mobile app

## Self-Check

1. **What is OCC and why is it important for headless commerce?**
   <details>
   <summary>Answer</summary>
   OCC (Omni Commerce Connect) is the RESTful API layer of SAP Commerce. It exposes all commerce functionality (products, carts, checkout, users, orders) as REST endpoints. It's essential for headless commerce because it decouples the frontend from the backend — any frontend (Spartacus, mobile app, kiosk, third-party SPA) can consume OCC APIs. This enables multi-channel experiences from a single Commerce backend.
   </details>

2. **How does Spartacus communicate with Commerce?**
   <details>
   <summary>Answer</summary>
   Spartacus is an Angular-based SPA that communicates with SAP Commerce exclusively through OCC REST APIs over HTTP/HTTPS. It uses OAuth2 for authentication (client credentials for anonymous, resource owner password for logged-in users). Spartacus never directly accesses the database or Java services — all data flows through OCC endpoints returning JSON. Server-Side Rendering (SSR) via Node.js is used for SEO and initial page load performance.
   </details>

3. **What's the role of SmartEdit?**
   <details>
   <summary>Answer</summary>
   SmartEdit is a WYSIWYG content management tool that allows business users to edit CMS content directly on the storefront in a visual, drag-and-drop manner. It works by loading the storefront in an iframe and overlaying editing controls. Content managers can: edit page content, rearrange CMS components in slots, create personalization variations, preview changes before publishing, and manage content across catalog versions (Staged → Online synchronization).
   </details>

4. **How do you add a custom REST endpoint to OCC?**
   <details>
   <summary>Answer</summary>
   Create a Spring MVC controller in your OCC extension (e.g., `myextensionocc`), annotate it with `@RestController` and `@RequestMapping`, and use Commerce Facades to fetch data:
   ```java
   @RestController
   @RequestMapping("/{baseSiteId}/myresource")
   public class MyResourceController extends BaseController {
       @Resource
       private MyFacade myFacade;
       
       @GetMapping("/{code}")
       public MyDataWsDTO getResource(@PathVariable String code) {
           MyData data = myFacade.getByCode(code);
           return dataMapper.map(data, MyDataWsDTO.class);
       }
   }
   ```
   Register it in your extension's web Spring context. Use WsDTO classes for the API response (not internal Data DTOs directly).
   </details>

5. **What are CMS components, slots, and pages?**
   <details>
   <summary>Answer</summary>
   - **Pages** (`ContentPage`, `ProductPage`, `CategoryPage`) — represent full web pages with a layout template
   - **Slots** (`ContentSlot`) — named placeholder areas within a page template (e.g., "HeaderSlot", "BodySlot", "FooterSlot")  
   - **Components** (`CMSComponent` subtypes) — individual UI elements placed in slots (e.g., `BannerComponent`, `ProductCarouselComponent`, `CMSParagraphComponent`)
   
   The hierarchy is: Page → Template → Slots → Components. Pages reference a PageTemplate, templates define which slots are available, and components are placed into slots. This structure is managed via ImpEx or SmartEdit and is catalog-version-aware (Staged/Online).
   </details>

---
**Previous**: [← 06 - ImpEx](../06-impex/README.md) | **Next**: [08 - Build & Deployment →](../08-build-deployment/README.md)