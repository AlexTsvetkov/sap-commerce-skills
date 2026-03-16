# SAP Commerce + S/4HANA Integration via CPI

## Overview

The **primary recommended** integration pattern between SAP Commerce Cloud and SAP S/4HANA is through **SAP Cloud Platform Integration (CPI)** — now called **SAP Integration Suite**. CPI acts as the middleware layer, handling message transformation, routing, error handling, and monitoring.

---

## Architecture

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  SAP Commerce │◄───────►│   SAP CPI    │◄───────►│  S/4HANA     │
│  Cloud (CCv2) │         │ (Integration │         │  (On-Prem or │
│               │         │   Suite)     │         │   Cloud)     │
└──────────────┘         └──────────────┘         └──────────────┘
     REST/OData              iFlow                  OData/IDoc/
     ImpEx Hotfolder         Mappings               BAPI/RFC
     Webhooks                Error Handling
```

---

## Integration Scenarios

### 1. Product Master Data (S/4HANA → Commerce)

**Direction**: S/4HANA → CPI → Commerce  
**Frequency**: Near-real-time or batch (daily)  
**Protocol**: S/4HANA OData API → CPI iFlow → Commerce ImpEx Hotfolder

#### Flow
```
S/4HANA                    CPI                         Commerce
─────────                  ─────                       ────────
Product changed ──►  OData request to           Hotfolder picks up
                     API_PRODUCT_SRV       ──►  ImpEx CSV file
                     Transform to ImpEx         Import into Staged
                     Generate CSV               catalog
                     SFTP to Commerce           Sync to Online
                     hotfolder
```

#### CPI iFlow Configuration
```xml
<!-- S/4HANA OData Source -->
<from uri="odata://API_PRODUCT_SRV/A_Product?$expand=to_Description,to_Plant"/>

<!-- Transform to Commerce ImpEx format -->
<to uri="processor:productMappingProcessor"/>

<!-- Output ImpEx CSV -->
<!-- Header: INSERT_UPDATE Product;code[unique=true];name[lang=en];... -->

<!-- SFTP to Commerce Hotfolder -->
<to uri="sftp://commerce-host/hybris/import/master/incoming?fileName=products_${date}.csv"/>
```

#### Commerce Hotfolder Configuration
```properties
# local.properties
impex.import.dir=/opt/hybris/import/master/incoming
hotfolder.root.dir=/opt/hybris/import
```

```xml
<!-- Spring config for hotfolder mapping -->
<bean id="productImportMapping" class="de.hybris.platform.acceleratorservices.dataimport.batch.converter.mapping.impl.DefaultConverterMapping">
    <property name="mapping" value="product"/>
    <property name="converter" ref="productConverter"/>
</bean>
```

#### Sample ImpEx Output from CPI
```impex
$catalog=myProductCatalog
$catalogVersion=catalogversion(catalog(id[default=$catalog]),version[default='Staged'])[unique=true,default=$catalog:Staged]

INSERT_UPDATE Product;code[unique=true];name[lang=en];description[lang=en];ean;unit(code);$catalogVersion
;MAT-001;Industrial Pump X100;High-performance industrial pump;4012345678901;pieces;
;MAT-002;Valve Assembly V200;Precision valve assembly;4012345678902;pieces;
```

---

### 2. Pricing & Conditions (S/4HANA → Commerce)

**Direction**: S/4HANA → CPI → Commerce  
**Frequency**: Batch (nightly) or triggered on change  
**Protocol**: S/4HANA Pricing Condition OData → CPI → Commerce ImpEx

#### Flow
```
S/4HANA                    CPI                         Commerce
─────────                  ─────                       ────────
Condition records ──►  OData:                     Import PriceRow
(PR00, ZPR1...)       A_SlsPrcgCndnRecdValidity   items via ImpEx
                      Map condition type    ──►   Map to Europe1
                      to Commerce PriceRow        pricing system
                      Generate ImpEx CSV
```

#### Sample Pricing ImpEx
```impex
INSERT_UPDATE PriceRow;product(code,$catalogVersion)[unique=true];price;currency(isocode);unit(code);net;startTime[dateformat=yyyy-MM-dd];endTime[dateformat=yyyy-MM-dd]
;MAT-001:$catalogVersion;1250.00;USD;pieces;true;2024-01-01;2024-12-31
;MAT-002:$catalogVersion;340.00;USD;pieces;true;2024-01-01;2024-12-31
```

#### Customer-Specific Pricing (B2B)
```impex
INSERT_UPDATE PriceRow;product(code,$catalogVersion)[unique=true];price;currency(isocode);user(uid)[unique=true];net
;MAT-001:$catalogVersion;1100.00;USD;buyer@acme.com;true
```

---

### 3. Order Replication (Commerce → S/4HANA)

**Direction**: Commerce → CPI → S/4HANA  
**Frequency**: Real-time (on order placement)  
**Protocol**: Commerce webhook/REST → CPI iFlow → S/4HANA Sales Order API

#### Flow
```
Commerce                   CPI                         S/4HANA
────────                   ─────                       ─────────
Order placed       ──►  Receive order JSON       Create Sales Order
 (business process       Transform to             via API_SALES_
  action sends           S/4HANA format           ORDER_SRV
  REST call)             Map products,      ──►   Returns SO number
                         customer, prices
                         Call OData API           Commerce stores
                         Return SO number  ◄──   SO number on order
```

#### Commerce: Custom Process Action
```java
public class SendOrderToERPAction extends AbstractProceduralAction<OrderProcessModel> {
    
    @Resource
    private RestTemplate restTemplate;
    
    @Override
    public void executeAction(OrderProcessModel process) throws Exception {
        OrderModel order = process.getOrder();
        
        // Build order payload
        Map<String, Object> payload = new HashMap<>();
        payload.put("orderCode", order.getCode());
        payload.put("customerID", order.getUser().getUid());
        payload.put("totalPrice", order.getTotalPrice());
        payload.put("currency", order.getCurrency().getIsocode());
        payload.put("entries", buildEntries(order));
        payload.put("deliveryAddress", buildAddress(order.getDeliveryAddress()));
        
        // Send to CPI
        String cpiUrl = configurationService.getConfiguration()
            .getString("cpi.order.replication.url");
        
        ResponseEntity<Map> response = restTemplate.postForEntity(
            cpiUrl, payload, Map.class);
        
        // Store ERP order number
        if (response.getStatusCode().is2xxSuccessful()) {
            String salesOrderNumber = (String) response.getBody().get("SalesOrder");
            order.setErpOrderNumber(salesOrderNumber);
            modelService.save(order);
        }
    }
}
```

#### CPI: Order Mapping to S/4HANA
```json
// Input from Commerce
{
    "orderCode": "ORD-0001",
    "customerID": "CUST-100",
    "entries": [
        { "productCode": "MAT-001", "quantity": 2, "price": 1250.00 }
    ]
}

// Output to S/4HANA API_SALES_ORDER_SRV
{
    "SalesOrderType": "OR",
    "SoldToParty": "100",
    "to_Item": [
        { "Material": "MAT-001", "RequestedQuantity": "2", "NetAmount": "2500.00" }
    ]
}
```

---

### 4. Inventory / ATP Check (Commerce ↔ S/4HANA)

**Direction**: Commerce → CPI → S/4HANA (real-time request/response)  
**Frequency**: Real-time (on PDP, cart, checkout)  
**Protocol**: Commerce REST → CPI → S/4HANA ATP OData

#### Flow
```
Storefront        Commerce           CPI              S/4HANA
──────────        ────────           ─────            ─────────
User views  ──►  StockFacade  ──►  ATP iFlow  ──►  API_ATP_SRV
product          calls CPI          transforms       checks stock
                 REST endpoint      request           
                                                     Returns qty
◄── Display  ◄──  Return     ◄──  Transform  ◄──   available
    stock         stock data       response
```

#### Commerce: Custom Stock Service
```java
@Service
public class ERPStockService implements StockService {
    
    @Resource
    private RestTemplate cpiRestTemplate;
    
    @Override
    public StockLevelModel getStockLevel(ProductModel product, WarehouseModel warehouse) {
        String cpiUrl = String.format("%s/atp?material=%s&plant=%s",
            cpiBaseUrl, product.getCode(), warehouse.getCode());
        
        Map<String, Object> response = cpiRestTemplate.getForObject(cpiUrl, Map.class);
        
        StockLevelModel stock = modelService.create(StockLevelModel.class);
        stock.setAvailable(((Number) response.get("AvailableQuantity")).intValue());
        stock.setProductCode(product.getCode());
        return stock;
    }
}
```

---

### 5. Customer Master Sync (Bidirectional)

**Direction**: Bidirectional  
**Frequency**: Near-real-time

#### Commerce → S/4HANA (new registration)
```
Customer registers ──► CPI ──► S/4HANA Business Partner API
                               (API_BUSINESS_PARTNER)
                               Returns BP number
                       ◄──     Store BP number on Commerce customer
```

#### S/4HANA → Commerce (updates from ERP)
```
BP updated in ERP ──► CPI ──► Commerce OCC API or ImpEx
                              Update customer address, credit limit, etc.
```

---

## CPI Best Practices

### Error Handling
```
iFlow design:
1. Try-catch around each external call
2. Store failed messages in JMS queue for retry
3. Alert via email on 3rd consecutive failure
4. Dead letter queue after max retries
5. Dashboard monitoring in CPI cockpit
```

### Security
| Aspect | Configuration |
|--------|--------------|
| Commerce → CPI | OAuth2 client credentials |
| CPI → S/4HANA | Principal propagation or technical user |
| CPI → Commerce | Basic auth or OAuth2 |
| Data in transit | TLS 1.2+ mandatory |

### Performance
- Use **batch processing** for bulk data (products, prices)
- Use **real-time** only for critical paths (ATP, order placement)
- Implement **delta detection** — only sync changed records
- Use **parallel processing** in CPI for large payloads

---

## Setup Instructions

### Step 1: Configure CPI Tenant
1. Access SAP Integration Suite via BTP cockpit
2. Create a service instance for Process Integration Runtime
3. Generate service key (client ID + secret)

### Step 2: Configure S/4HANA Connectivity
1. In CPI, create HTTP destination pointing to S/4HANA OData endpoints
2. Configure authentication (basic or OAuth2)
3. Test connectivity with a simple GET request

### Step 3: Configure Commerce Connectivity
1. In Commerce, add CPI endpoint URLs to `local.properties`:
   ```properties
   cpi.base.url=https://your-tenant.it-cpi018.cfapps.eu10.hana.ondemand.com
   cpi.client.id=your-client-id
   cpi.client.secret=your-client-secret
   cpi.token.url=https://your-tenant.authentication.eu10.hana.ondemand.com/oauth/token
   ```

2. Create REST template bean with OAuth2:
   ```xml
   <bean id="cpiRestTemplate" class="org.springframework.web.client.RestTemplate">
       <property name="interceptors">
           <list><ref bean="cpiOAuth2Interceptor"/></list>
       </property>
   </bean>
   ```

### Step 4: Deploy iFlows
1. Import iFlow packages for each scenario
2. Configure externalized parameters (endpoints, credentials)
3. Deploy and test each flow individually
4. Monitor in CPI Operations cockpit

---

*This is the primary recommended integration pattern for SAP Commerce + S/4HANA.*