# SAP Commerce + S/4HANA Direct API Integration

## Overview

In some scenarios, **direct API integration** between SAP Commerce and S/4HANA (bypassing CPI middleware) is viable. This approach is simpler but less flexible — suitable for lightweight integrations or when CPI is not available.

---

## When to Use Direct Integration

| Use Direct API When | Use CPI Instead When |
|---------------------|---------------------|
| Simple point-to-point, 1-2 scenarios | Multiple integration scenarios |
| S/4HANA Cloud with public OData APIs | Complex message transformation |
| No complex mapping/routing needed | Error handling & retry logic required |
| Prototype or PoC | Production with monitoring needs |
| Low volume, non-critical flows | High volume, business-critical flows |

---

## Architecture

```
┌──────────────┐              ┌──────────────┐
│  SAP Commerce │◄────────────►│  S/4HANA     │
│  Cloud (CCv2) │   Direct     │  Cloud       │
│               │   OData/REST │              │
└──────────────┘              └──────────────┘
```

---

## Implementation Patterns

### Pattern 1: OData Client in Commerce

Commerce calls S/4HANA OData APIs directly using Spring's `RestTemplate` or `WebClient`.

#### Configuration
```properties
# local.properties
s4hana.base.url=https://my-s4hana.s4hana.ondemand.com
s4hana.api.user=COMM_TECH_USER
s4hana.api.password=encrypted_password
s4hana.auth.type=basic  # or oauth2
```

#### Spring Configuration
```xml
<bean id="s4hanaRestTemplate" class="org.springframework.web.client.RestTemplate">
    <property name="interceptors">
        <list>
            <bean class="com.myproject.integration.S4HANAAuthInterceptor">
                <property name="username" value="${s4hana.api.user}"/>
                <property name="password" value="${s4hana.api.password}"/>
            </bean>
        </list>
    </property>
    <property name="messageConverters">
        <list>
            <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter"/>
        </list>
    </property>
</bean>
```

#### Service Implementation: Real-Time Price Check
```java
@Service("s4hanaPriceService")
public class S4HANAPriceService {
    
    @Resource
    private RestTemplate s4hanaRestTemplate;
    
    @Value("${s4hana.base.url}")
    private String baseUrl;
    
    public BigDecimal getPrice(String materialNumber, String customerNumber, String salesOrg) {
        String url = String.format(
            "%s/sap/opu/odata/sap/API_SLSPRCGCONDITIONRECORD_SRV/A_SlsPrcgCndnRecdValidity" +
            "?$filter=Material eq '%s' and Customer eq '%s' and SalesOrganization eq '%s'" +
            "&$format=json",
            baseUrl, materialNumber, customerNumber, salesOrg);
        
        try {
            ResponseEntity<ODataResponse> response = 
                s4hanaRestTemplate.getForEntity(url, ODataResponse.class);
            
            if (response.getBody() != null && !response.getBody().getResults().isEmpty()) {
                return response.getBody().getResults().get(0).getConditionRateValue();
            }
        } catch (HttpClientErrorException e) {
            LOG.error("Failed to fetch price from S/4HANA: {}", e.getMessage());
        }
        
        return null; // Fallback to Commerce pricing
    }
}
```

#### Service Implementation: Order Posting
```java
@Service("s4hanaOrderService")
public class S4HANAOrderService {
    
    @Resource
    private RestTemplate s4hanaRestTemplate;
    
    public String createSalesOrder(OrderModel order) {
        String url = baseUrl + "/sap/opu/odata/sap/API_SALES_ORDER_SRV/A_SalesOrder";
        
        // Build S/4HANA sales order payload
        Map<String, Object> salesOrder = new LinkedHashMap<>();
        salesOrder.put("SalesOrderType", "OR");
        salesOrder.put("SalesOrganization", "1000");
        salesOrder.put("DistributionChannel", "10");
        salesOrder.put("OrganizationDivision", "00");
        salesOrder.put("SoldToParty", getERPCustomerNumber(order.getUser()));
        
        // Order items
        List<Map<String, Object>> items = new ArrayList<>();
        for (AbstractOrderEntryModel entry : order.getEntries()) {
            Map<String, Object> item = new LinkedHashMap<>();
            item.put("Material", entry.getProduct().getCode());
            item.put("RequestedQuantity", String.valueOf(entry.getQuantity()));
            item.put("RequestedQuantityUnit", "EA");
            items.add(item);
        }
        salesOrder.put("to_Item", items);
        
        // POST with CSRF token
        String csrfToken = fetchCSRFToken(url);
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-CSRF-Token", csrfToken);
        
        HttpEntity<Map<String, Object>> request = new HttpEntity<>(salesOrder, headers);
        ResponseEntity<Map> response = s4hanaRestTemplate.postForEntity(url, request, Map.class);
        
        return (String) response.getBody().get("SalesOrder");
    }
    
    private String fetchCSRFToken(String url) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-CSRF-Token", "Fetch");
        HttpEntity<?> entity = new HttpEntity<>(headers);
        
        ResponseEntity<String> response = s4hanaRestTemplate.exchange(
            url, HttpMethod.HEAD, entity, String.class);
        return response.getHeaders().getFirst("X-CSRF-Token");
    }
}
```

---

### Pattern 2: S/4HANA Webhooks to Commerce OCC

S/4HANA Cloud supports event-driven integration. When master data changes, S/4HANA pushes events to Commerce.

#### Setup
1. In S/4HANA Cloud, configure **Enterprise Event Enablement**
2. Create event subscription pointing to Commerce OCC endpoint
3. Commerce receives the event and triggers data refresh

#### Commerce: Webhook Receiver Controller
```java
@Controller
@RequestMapping("/api/v1/s4hana-events")
public class S4HANAEventController {
    
    @Resource
    private ProductSyncService productSyncService;
    
    @PostMapping("/product-changed")
    @ResponseBody
    public ResponseEntity<String> handleProductChanged(@RequestBody Map<String, Object> event) {
        String materialNumber = (String) event.get("Material");
        
        // Trigger async product sync
        productSyncService.syncProduct(materialNumber);
        
        return ResponseEntity.ok("accepted");
    }
    
    @PostMapping("/price-changed")
    @ResponseBody
    public ResponseEntity<String> handlePriceChanged(@RequestBody Map<String, Object> event) {
        String conditionRecord = (String) event.get("ConditionRecord");
        priceSyncService.syncPrice(conditionRecord);
        return ResponseEntity.ok("accepted");
    }
}
```

---

### Pattern 3: Commerce Data Hub (Legacy)

> **Note**: Data Hub is deprecated since Commerce 2005. Mentioned here for legacy reference only.

Data Hub was Commerce's built-in ETL tool for S/4HANA integration:
```
S/4HANA → IDoc → Data Hub → Canonical → Commerce ImpEx
```

For new projects, use **CPI** or **Direct API** instead.

---

## Authentication Options

### Basic Authentication
```java
public class S4HANAAuthInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {
        String auth = username + ":" + password;
        String encoded = Base64.getEncoder().encodeToString(auth.getBytes());
        request.getHeaders().set("Authorization", "Basic " + encoded);
        return execution.execute(request, body);
    }
}
```

### OAuth2 (S/4HANA Cloud)
```java
public class S4HANAOAuth2Interceptor implements ClientHttpRequestInterceptor {
    
    private String tokenUrl;
    private String clientId;
    private String clientSecret;
    private volatile String cachedToken;
    private volatile long tokenExpiry;
    
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {
        if (cachedToken == null || System.currentTimeMillis() > tokenExpiry) {
            refreshToken();
        }
        request.getHeaders().setBearerAuth(cachedToken);
        return execution.execute(request, body);
    }
    
    private synchronized void refreshToken() {
        // POST to token URL with client credentials
        RestTemplate tokenTemplate = new RestTemplate();
        Map<String, String> params = new HashMap<>();
        params.put("grant_type", "client_credentials");
        params.put("client_id", clientId);
        params.put("client_secret", clientSecret);
        
        Map<String, Object> response = tokenTemplate.postForObject(tokenUrl, params, Map.class);
        cachedToken = (String) response.get("access_token");
        tokenExpiry = System.currentTimeMillis() + ((Number) response.get("expires_in")).longValue() * 1000 - 60000;
    }
}
```

### SAP Destination Service (BTP)
When both Commerce (CCv2) and S/4HANA are on BTP, use the Destination Service:
```properties
# Reference a BTP destination
s4hana.destination.name=S4HANA_PROD
```

---

## Error Handling for Direct Integration

```java
@Service
public class ResilientS4HANAService {
    
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 2000;
    
    public <T> T callWithRetry(Supplier<T> apiCall, T fallback) {
        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                return apiCall.get();
            } catch (HttpServerErrorException e) {
                LOG.warn("S/4HANA call failed, attempt {}/{}: {}", attempt, MAX_RETRIES, e.getMessage());
                if (attempt < MAX_RETRIES) {
                    sleep(RETRY_DELAY_MS * attempt);
                }
            } catch (HttpClientErrorException e) {
                LOG.error("S/4HANA client error (no retry): {}", e.getMessage());
                break;
            }
        }
        LOG.error("All retries exhausted, using fallback");
        return fallback;
    }
}
```

---

## Comparison: Direct vs CPI

| Aspect | Direct API | Via CPI |
|--------|-----------|---------|
| **Setup complexity** | Low | Medium |
| **Transformation** | In Commerce code | In CPI graphical mapper |
| **Error handling** | Custom code | Built-in retry, dead letter |
| **Monitoring** | Custom logging | CPI Operations dashboard |
| **Scalability** | Limited by Commerce threads | CPI handles async/batch |
| **Maintainability** | Developers needed | Business/integration team can manage |
| **Cost** | No additional license | CPI license required |
| **Best for** | PoC, simple integrations | Production, complex flows |

---

*Direct API integration is suitable for simple scenarios. For production-grade multi-scenario integrations, CPI is recommended.*