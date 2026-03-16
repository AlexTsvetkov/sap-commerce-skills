# Course 10: Order Management & Commerce Processes

## 🎯 Overview
Understand the Commerce order lifecycle, business processes, payment/fulfillment integration, and the Order Management module.

**Duration**: 8-10 hours | **Level**: Advanced | **Prerequisites**: Courses 01-06

---

## Topics

### 1. Order Lifecycle
```
Cart → Checkout → Order → Fulfillment → Delivery → Return
  │        │         │         │            │         │
  ├─ Add   ├─ Address ├─ Payment ├─ Warehouse ├─ Ship  ├─ RMA
  ├─ Promo ├─ Delivery ├─ Fraud   ├─ Pick/Pack  ├─ Track ├─ Refund
  └─ Save  └─ Payment  └─ Confirm └─ Consign    └─ Done  └─ Credit
```

### 2. Business Process Engine
SAP Commerce uses an XML-based process engine for order workflows:

```xml
<!-- Order process definition -->
<process xmlns="http://www.hybris.de/xsd/processdefinition" 
         name="order-process" start="checkOrder">
    
    <action id="checkOrder" bean="checkOrderAction">
        <transition name="OK" to="fraudCheck"/>
        <transition name="NOK" to="error"/>
    </action>
    
    <action id="fraudCheck" bean="fraudCheckAction">
        <transition name="OK" to="authorizePayment"/>
        <transition name="POTENTIAL" to="manualFraudCheck"/>
        <transition name="FRAUD" to="cancelOrder"/>
    </action>
    
    <wait id="manualFraudCheck" then="authorizePayment" 
          prependProcessCode="true">
        <event>FraudApproved</event>
    </wait>
    
    <action id="authorizePayment" bean="authorizePaymentAction">
        <transition name="OK" to="sendOrderConfirmation"/>
        <transition name="NOK" to="cancelOrder"/>
    </action>
    
    <action id="sendOrderConfirmation" bean="sendOrderConfirmationAction">
        <transition name="OK" to="waitForWarehouse"/>
    </action>
    
    <end id="error" state="ERROR">Order check failed</end>
    <end id="cancelOrder" state="CANCELLED">Order cancelled</end>
</process>
```

### 3. Custom Process Actions
```java
public class MyCustomAction extends AbstractProceduralAction<OrderProcessModel> {
    
    @Override
    public void executeAction(OrderProcessModel process) throws Exception {
        OrderModel order = process.getOrder();
        
        // Custom business logic
        if (order.getTotalPrice() > 1000) {
            order.setStatus(OrderStatus.CHECKING);
            modelService.save(order);
        }
    }
    
    @Override
    public Set<String> getTransitions() {
        return AbstractSimpleDecisionAction.createTransitions("OK", "NOK");
    }
}
```

### 4. Order Statuses
| Status | Meaning |
|--------|---------|
| `CREATED` | Order just placed |
| `CHECKED_VALID` | Validation passed |
| `PAYMENT_AUTHORIZED` | Payment authorized |
| `FRAUD_CHECKED` | Fraud check complete |
| `ORDER_SPLIT` | Split into consignments |
| `PAYMENT_CAPTURED` | Payment captured |
| `COMPLETED` | Fully fulfilled |
| `CANCELLED` | Cancelled |

### 5. Consignments & Warehouses
```impex
INSERT_UPDATE Warehouse;code[unique=true];name;vendor(code);default
;warehouse_east;East Warehouse;default;true
;warehouse_west;West Warehouse;default;false

INSERT_UPDATE PointOfService;name[unique=true];type(code);address(&addrID);warehouses(code)
;store_nyc;STORE;addr1;warehouse_east
```

### 6. Payment Integration
Commerce provides PSP (Payment Service Provider) abstractions:
```java
public interface PaymentService {
    PaymentTransactionModel authorize(CartModel cart, PaymentInfoModel paymentInfo);
    PaymentTransactionModel capture(OrderModel order);
    PaymentTransactionModel refund(OrderModel order, BigDecimal amount);
}
```

### 7. Promotions & Pricing
```impex
# Rule-based promotions
INSERT_UPDATE PromotionSourceRule;code[unique=true];name;priority;status(code)
;buy2get1free;Buy 2 Get 1 Free;100;PUBLISHED

# Europe1 price rows
INSERT_UPDATE PriceRow;product(code,$cv)[unique=true];price;currency(isocode);unit(code);net
;PROD-001;99.99;USD;pieces;true
```

### 8. Returns & Refunds
```java
// Create return request
ReturnRequestModel returnRequest = modelService.create(ReturnRequestModel.class);
returnRequest.setOrder(order);
returnRequest.setRMA("RMA-001");

// Create return entry
RefundEntryModel refundEntry = modelService.create(RefundEntryModel.class);
refundEntry.setOrderEntry(orderEntry);
refundEntry.setExpectedQuantity(1L);
refundEntry.setReturnRequest(returnRequest);
```

---

## Exercises
1. Trace the full order process by placing an order and checking business process status in Backoffice
2. Create a custom process action that sends a notification when order value exceeds a threshold
3. Configure a multi-warehouse fulfillment with sourcing strategies
4. Implement a simple return flow with refund
5. Create a promotion: 10% off for orders over $100

## Self-Check

1. **What is a business process in SAP Commerce?**
   <details>
   <summary>Answer</summary>
   A business process is a stateful, event-driven workflow defined in an XML process definition. It models multi-step operations like order fulfillment, returns, or approvals. Each process has: states (action nodes, wait nodes, split/join nodes), transitions between states, and action beans (Spring beans that execute logic at each step). The Business Process Engine persists process state to the database, making processes durable across server restarts. Processes are triggered by events and can wait asynchronously for external events (e.g., payment confirmation).
   </details>

2. **How does fraud checking work?**
   <details>
   <summary>Answer</summary>
   Fraud checking is a step in the order process that evaluates orders against configurable fraud rules before payment capture. The default implementation uses `FraudService` with a list of `FraudSymptom` providers that score risk factors (e.g., mismatched billing/shipping addresses, high-value orders, blacklisted IPs). Each symptom contributes a score, and if the total exceeds a threshold, the order is flagged for manual review or rejected. The fraud check runs after payment authorization but before capture. You can integrate external fraud providers (e.g., Accertify, Riskified) by implementing custom `FraudServiceProvider`.
   </details>

3. **What are consignments and how do they relate to orders?**
   <details>
   <summary>Answer</summary>
   A consignment represents a shipment — a group of order entries fulfilled from a single warehouse/location. One order can have multiple consignments (split shipment). Each consignment has its own status lifecycle (READY, SHIPPED, DELIVERED) and tracking information. The warehousing module creates consignments during the sourcing step by allocating order entries to fulfillment locations based on availability and sourcing rules. Consignment entries reference original order entries with the allocated quantity.
   </details>

4. **How do you customize the order process flow?**
   <details>
   <summary>Answer</summary>
   Customize the process definition XML (e.g., `order-process.xml`) in your extension's `resources/processes/` folder:
   1. Copy the OOTB process definition to your extension
   2. Modify the XML — add/remove/reorder nodes, add custom action nodes
   3. Implement custom action beans extending `AbstractProceduralAction` or `AbstractAction`
   4. Register your action beans in Spring XML
   5. Point to your custom process definition via properties:
   ```properties
   order.process.definition.name=myproject-order-process
   ```
   Each action returns a transition string (e.g., "OK", "NOK") that determines the next state in the process flow.
   </details>

5. **What's the difference between payment authorization and capture?**
   <details>
   <summary>Answer</summary>
   - **Authorization** — reserves funds on the customer's payment method (credit card, PayPal) without actually transferring money. Happens at checkout/order placement. The auth hold guarantees funds are available but no charge appears on the customer's statement. Authorizations expire (typically 7-30 days).
   - **Capture** — actually charges the customer and transfers funds. Happens after order validation, fraud check, and often after shipping. You can capture the full amount or partial amounts (e.g., per consignment).
   
   This two-step process protects both merchant and customer — you only charge for what you actually ship, and you can cancel the auth if the order can't be fulfilled.
   </details>

---
**Previous**: [← 09 - Performance](../09-performance/README.md) | **Next**: [11 - B2B Commerce →](../11-b2b-commerce/README.md)