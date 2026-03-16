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
- What is a business process in SAP Commerce?
- How does fraud checking work?
- What are consignments and how do they relate to orders?
- How do you customize the order process flow?
- What's the difference between payment authorization and capture?

---
**Previous**: [← 09 - Performance](../09-performance/README.md) | **Next**: [11 - B2B Commerce →](../11-b2b-commerce/README.md)