# SAP Commerce — Alternative Integration Scenarios

## Beyond CPI: Other Ways to Integrate Commerce with S/4HANA and Third-Party Systems

---

## Integration Options Summary

| # | Pattern | Middleware | Best For |
|---|---------|-----------|----------|
| 1 | **CPI (SAP Integration Suite)** | SAP CPI | Primary recommendation for SAP-to-SAP |
| 2 | **Direct OData/REST** | None | Simple, low-volume point-to-point |
| 3 | **IDoc via CPI or PI/PO** | CPI / PI/PO | Legacy S/4HANA on-prem with IDoc interfaces |
| 4 | **SAP Event Mesh** | Event Mesh | Event-driven, loosely coupled architecture |
| 5 | **Commerce Hotfolder (File-based)** | SFTP / Blob storage | Batch imports from any source |
| 6 | **Kyma / SAP BTP Extension** | Kyma Runtime | Serverless extensions & event handlers |
| 7 | **Third-party iPaaS** | MuleSoft, Dell Boomi, etc. | Non-SAP middleware already in use |
| 8 | **SAP CDC (Customer Data Cloud)** | CDC | Customer identity & authentication |

---

## 1. IDoc-Based Integration (S/4HANA On-Premise)

For on-premise S/4HANA systems that primarily expose IDoc interfaces.

### Architecture
```
S/4HANA (On-Prem)          CPI / PI/PO           Commerce
─────────────────          ───────────           ────────
Material Master    ──►    Receive IDoc     ──►  ImpEx Hotfolder
 IDoc (MATMAS)            Transform to CSV       Import products
                          SFTP upload

Sales Order        ◄──    Build IDoc       ◄──  REST call from
 IDoc (ORDERS)            from JSON              Commerce process
                          Send to S/4HANA        action
```

### IDoc Types Used
| IDoc Type | Direction | Purpose |
|-----------|----------|---------|
| `MATMAS` | S/4HANA → Commerce | Material master data |
| `DEBMAS` | S/4HANA → Commerce | Customer master data |
| `COND_A` | S/4HANA → Commerce | Pricing conditions |
| `ORDERS` | Commerce → S/4HANA | Sales order creation |
| `DESADV` | S/4HANA → Commerce | Delivery notification |
| `INVOIC` | S/4HANA → Commerce | Invoice data |
| `WMMBID` | S/4HANA → Commerce | Inventory/stock levels |

### Setup Steps
1. **S/4HANA**: Configure partner profile, port, and IDoc type for distribution
2. **CPI/PI**: Create integration flow to receive IDoc XML, transform to ImpEx CSV
3. **Commerce**: Configure hotfolder to watch for incoming CSV files
4. **Monitoring**: Set up alerts in CPI for failed IDoc processing

---

## 2. SAP Event Mesh (Event-Driven Integration)

### Architecture
```
S/4HANA                 Event Mesh              Commerce
────────                ──────────              ────────
Business event    ──►  Topic:                   BTP Extension
(product changed)      sap/s4/material/changed  subscribes to topic
                                           ──►  Triggers sync
                       Topic:
Order placed     ◄──  sap/commerce/order/created  ◄── Commerce
in Commerce            publishes event                 publishes
```

### When to Use Event Mesh
- **Loose coupling** — systems don't need to know about each other
- **Multiple consumers** — same event can trigger actions in multiple systems
- **Eventual consistency** — real-time not required, near-real-time is fine
- **Scalability** — high event throughput

### Commerce: Publishing Events
```java
@Service
public class CommerceEventPublisher {
    
    @Resource
    private EventMeshClient eventMeshClient;
    
    public void publishOrderCreated(OrderModel order) {
        Map<String, Object> event = new HashMap<>();
        event.put("orderCode", order.getCode());
        event.put("customerID", order.getUser().getUid());
        event.put("totalPrice", order.getTotalPrice());
        event.put("currency", order.getCurrency().getIsocode());
        event.put("timestamp", Instant.now().toString());
        
        eventMeshClient.publish("sap/commerce/order/created", event);
    }
}
```

### Commerce: Subscribing to Events
```java
@EventListener
public class S4HANAEventSubscriber {
    
    @Resource
    private ProductSyncService productSyncService;
    
    @EventMeshSubscription(topic = "sap/s4/material/changed")
    public void onMaterialChanged(Map<String, Object> event) {
        String materialNumber = (String) event.get("Material");
        productSyncService.syncProductFromERP(materialNumber);
    }
}
```

---

## 3. Commerce Hotfolder (File-Based Batch Import)

The built-in batch import mechanism, suitable for any system that can produce CSV files.

### Architecture
```
Any Source System        SFTP / Azure Blob        Commerce
────────────────        ─────────────────        ────────
Export products    ──►  Upload CSV files   ──►  Hotfolder monitors
Export prices           to watched folder       Picks up files
Export stock            (scheduled or                ↓
                        event-driven)           Transforms CSV
                                                to ImpEx
                                                     ↓
                                                Imports data
                                                Logs results
```

### Configuration
```properties
# local.properties
cluster.node.groups=yHotfolderCandidate
hotfolder.root.dir=${HYBRIS_DATA_DIR}/acceleratorservices/import

# Watched directories
import.dirs=/import/master/product,/import/master/price,/import/master/stock
```

### Custom Hotfolder Setup
```xml
<!-- Spring config: custom file-based import -->
<bean id="stockImportTransformer" 
      class="de.hybris.platform.acceleratorservices.dataimport.batch.converter.impl.DefaultImpexConverter">
    <property name="header">
        <value>
            INSERT_UPDATE StockLevel;productCode[unique=true];available;warehouse(code)[unique=true]
        </value>
    </property>
    <property name="impexRow">
        <value>;{0};{1};{2}</value>
    </property>
</bean>

<bean id="stockImportMapping" 
      class="de.hybris.platform.acceleratorservices.dataimport.batch.converter.mapping.impl.DefaultConverterMapping">
    <property name="mapping" value="stock"/>
    <property name="converter" ref="stockImportTransformer"/>
</bean>
```

### Sample CSV Input
```csv
productCode,available,warehouse
MAT-001,150,warehouse_east
MAT-002,75,warehouse_east
MAT-001,200,warehouse_west
```

---

## 4. Kyma / SAP BTP Extensions

Serverless functions on SAP BTP Kyma Runtime for lightweight integration logic.

### Architecture
```
S/4HANA ──► Event Mesh ──► Kyma Function ──► Commerce OCC API
                           (Node.js/Python)
                           Transform data
                           Call Commerce API
```

### Example Kyma Function (Node.js)
```javascript
// Triggered by S/4HANA material change event
module.exports = {
    main: async function (event, context) {
        const materialNumber = event.data.Material;
        
        // Fetch product details from S/4HANA
        const s4Product = await fetch(
            `${S4_URL}/API_PRODUCT_SRV/A_Product('${materialNumber}')`,
            { headers: { Authorization: `Bearer ${S4_TOKEN}` } }
        ).then(r => r.json());
        
        // Transform and push to Commerce via OCC
        const impexData = transformToImpex(s4Product);
        await fetch(`${COMMERCE_URL}/api/import`, {
            method: 'POST',
            headers: { 
                Authorization: `Bearer ${COMMERCE_TOKEN}`,
                'Content-Type': 'text/plain'
            },
            body: impexData
        });
        
        return { status: 'synced', material: materialNumber };
    }
};
```

---

## 5. Third-Party iPaaS (MuleSoft, Dell Boomi, Workato)

When the customer already uses a non-SAP integration platform.

### Architecture
```
S/4HANA ◄──► MuleSoft / Boomi ◄──► Commerce
              │
              ├─ Connectors for both systems
              ├─ Visual mapping tools
              ├─ Built-in monitoring
              └─ Existing enterprise license
```

### Commerce Integration Points for iPaaS
| Commerce Endpoint | Protocol | Use |
|-------------------|----------|-----|
| OCC REST API | REST/JSON | Read/write products, orders, customers |
| ImpEx Hotfolder | SFTP/CSV | Bulk data import |
| HAC Import | HTTP POST | On-demand ImpEx execution |
| Custom REST endpoints | REST/JSON | Specialized integration APIs |
| Webhooks (custom) | HTTP POST | Event notifications from Commerce |

---

## 6. SAP CDC (Customer Data Cloud) Integration

For centralized customer identity, authentication, and consent management.

### Architecture
```
Customer                   SAP CDC                  Commerce
────────                   ───────                  ────────
Register/Login  ──►  CDC Handles Auth     ──►  Commerce receives
                     SSO, Social Login          verified identity
                     Consent management
                     CIAM features              Stores customer
                                                profile data
Profile update  ──►  CDC stores profile   ──►  Syncs to Commerce
                     Triggers webhook           customer model
```

### Commerce Configuration
```properties
# CDC (Gigya) integration
gigya.api.key=your-api-key
gigya.datacenter=eu1
gigya.application.key=your-app-key
gigya.application.secret=your-app-secret
```

### Required Extensions
```
gigyaloginaddon
gigyafacades
gigyaservices
gigyabackoffice
```

---

## 7. Payment Service Provider Integration

### Architecture
```
Spartacus ──► Commerce ──► PSP Adapter ──► Payment Gateway
                                           (Stripe/Adyen/PayPal)
              ◄───────── Payment result ◄──
```

### Commerce Payment Integration Points
```java
// Implement PaymentProvider interface
public class StripePaymentProvider implements PaymentProvider {
    
    @Override
    public PaymentTransactionModel authorize(CartModel cart, PaymentInfoModel info) {
        // Call Stripe API
        Charge charge = Charge.create(Map.of(
            "amount", cart.getTotalPrice().longValue() * 100,
            "currency", cart.getCurrency().getIsocode().toLowerCase(),
            "source", info.getToken(),
            "capture", false
        ));
        
        // Create transaction record
        PaymentTransactionModel tx = modelService.create(PaymentTransactionModel.class);
        tx.setRequestId(charge.getId());
        tx.setPaymentProvider("STRIPE");
        return tx;
    }
    
    @Override
    public PaymentTransactionModel capture(OrderModel order) { /* ... */ }
    
    @Override
    public PaymentTransactionModel refund(OrderModel order, BigDecimal amount) { /* ... */ }
}
```

---

## Integration Decision Matrix

| Scenario | Recommended Pattern | Why |
|----------|-------------------|-----|
| S/4HANA Cloud + Commerce Cloud | **CPI** | Full SAP stack, best tooling |
| S/4HANA On-Prem + Commerce Cloud | **CPI + IDoc** | IDoc is native to on-prem |
| Simple ATP check only | **Direct API** | Low complexity, real-time |
| Event-driven micro-integrations | **Event Mesh + Kyma** | Scalable, loosely coupled |
| Customer already uses MuleSoft | **MuleSoft** | Leverage existing investment |
| Bulk product import from PIM | **Hotfolder** | Built-in, no middleware needed |
| Customer identity / SSO | **SAP CDC** | Purpose-built for CIAM |
| Payment processing | **PSP direct** | Commerce has PSP framework |

---

*Choose the integration pattern that fits your project's complexity, existing infrastructure, and budget.*