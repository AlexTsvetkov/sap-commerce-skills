# Course 11: B2B Commerce

## 🎯 Overview
B2B Commerce features: organizations, units, budgets, approval workflows, cost centers, punchout, and quote management.

**Duration**: 6-8 hours | **Level**: Advanced | **Prerequisites**: Courses 01-07

---

## Topics

### 1. B2B vs B2C Key Differences
| Feature | B2C | B2B |
|---------|-----|-----|
| Buyer | Individual consumer | Organization/company |
| Pricing | Fixed/promo-based | Contract/negotiated |
| Payment | Credit card, wallet | Purchase orders, credit |
| Approval | None | Multi-level workflows |
| Catalog | Same for all | Customer-specific |
| Ordering | Cart → Order | Cart → Quote → Approval → Order |
| Account | Single user | Organization with units/roles |

### 2. B2B Organization Model
```
B2BCustomer → B2BUnit → B2BUnit (parent)
     │            │
     ├─ Roles     ├─ CostCenters
     ├─ Budgets   ├─ Addresses
     └─ Permissions└─ Budgets
```

```impex
INSERT_UPDATE B2BUnit;uid[unique=true];name;locName[lang=en];&B2BUnitID
;AcmeCorp;Acme Corporation;Acme Corporation;AcmeCorp
;AcmeCorp_East;Acme East Division;Acme East Division;AcmeCorp_East

INSERT_UPDATE B2BCustomer;uid[unique=true];name;email;defaultB2BUnit(&B2BUnitID);groups(uid)
;john@acme.com;John Doe;john@acme.com;AcmeCorp_East;b2bcustomergroup

INSERT_UPDATE B2BCostCenter;code[unique=true];name;unit(&B2BUnitID);currency(isocode);budgets(code)
;CC_EAST;East Cost Center;AcmeCorp_East;USD;BUDGET_EAST

INSERT_UPDATE B2BBudget;code[unique=true];budget;currency(isocode);dateRange;B2BUnit(&B2BUnitID)
;BUDGET_EAST;50000;USD;01/01/2024,12/31/2024;AcmeCorp_East
```

### 3. Approval Workflow
```
Order Placed → Permission Check → Approval Required?
                    │                    │
                    ├─ Under limit ──→ Auto-approve → Fulfillment
                    └─ Over limit ──→ Manager approval → Approve/Reject
```

Permission types:
| Permission | Description |
|-----------|-------------|
| `B2BOrderThresholdPermission` | Max per order |
| `B2BOrderThresholdTimespanPermission` | Max per timespan |
| `B2BBudgetExceededPermission` | Budget limit check |

```impex
INSERT_UPDATE B2BOrderThresholdPermission;code[unique=true];threshold;currency(isocode);unit(&B2BUnitID)
;APPROVE_5000;5000;USD;AcmeCorp

INSERT_UPDATE B2BUserGroup;uid[unique=true];permissions(code);unit(&B2BUnitID)
;approverGroup;APPROVE_5000;AcmeCorp
```

### 4. B2B Quotes
Buyers can request quotes, sellers respond with custom pricing:
```java
QuoteModel quote = commerceQuoteService.createQuoteFromCart(cart, user);
quote.setState(QuoteState.BUYER_SUBMITTED);
modelService.save(quote);

// Seller edits and submits back
quote.setState(QuoteState.SELLER_SUBMITTED);
```

### 5. Punchout (cXML / OCI)
Procurement system integration:
```
Buyer ERP (Ariba/Coupa) ──PunchOut Setup Request──→ Commerce
                         ←──PunchOut Setup Response──
                         ──Browse & Select──→
                         ←──PunchOut Order Message──
                         ──Purchase Order──→
```

Key extensions: `b2bpunchout`, `b2bpunchoutaddon`, `b2bpunchoutocctests`

### 6. B2B-Specific Extensions
| Extension | Purpose |
|-----------|---------|
| `b2bcommerce` | Core B2B services |
| `b2bapprovalprocess` | Approval workflow |
| `b2bacceleratorservices` | B2B facades/services |
| `b2bacceleratoraddon` | B2B storefront features |
| `commerceorgaddon` | Org management UI |
| `b2bpunchout` | Punchout integration |
| `powertoolsstore` | Sample B2B data |

---

## Exercises
1. Set up a B2B organization with 2 units, cost centers, and budgets
2. Create an approval process: orders > $5000 require manager approval
3. Configure B2B-specific pricing for a customer unit
4. Test the quote workflow: create quote → submit → seller response → accept
5. Explore the B2B Backoffice views for organization management

## Self-Check
- How does B2B organization hierarchy work?
- What are cost centers and how do they relate to budgets?
- How does the approval process determine who approves?
- What is punchout and when is it used?
- How do B2B catalogs differ from B2C?

---
**Previous**: [← 10 - Order Management](../10-order-management/README.md) | **Next**: [12 - Backoffice & Security →](../12-backoffice-security/README.md)