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
- What is OCC and why is it important for headless commerce?
- How does Spartacus communicate with Commerce?
- What's the role of SmartEdit?
- How do you add a custom REST endpoint to OCC?
- What are CMS components, slots, and pages?

---
**Previous**: [← 06 - ImpEx](../06-impex/README.md) | **Next**: [08 - Build & Deployment →](../08-build-deployment/README.md)