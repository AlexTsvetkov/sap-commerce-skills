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

1. **How does B2B organization hierarchy work?**
   <details>
   <summary>Answer</summary>
   B2B organizations use a hierarchical structure:
   - **B2BUnit** (root) — the company or division. Units can be nested (parent company → subsidiaries → departments).
   - **B2BCustomer** — individual users belonging to a unit. They inherit permissions and catalogs from their unit.
   - **B2BUserGroup** — groups within a unit for role-based access (e.g., buyers, approvers, admins).
   
   Each unit has its own addresses, cost centers, budgets, and catalog visibility. A customer's purchasing power is determined by the unit they belong to + their user groups + assigned permissions. Administrators manage their own unit's users via the Organization Management UI.
   </details>

2. **What are cost centers and how do they relate to budgets?**
   <details>
   <summary>Answer</summary>
   - **Cost Center** — represents a financial allocation unit within a B2B organization (e.g., "Marketing Department", "IT Department"). Each cost center is assigned to a B2BUnit and has one or more budgets.
   - **Budget** — defines a spending limit for a cost center over a time period (e.g., "$50,000 per quarter"). Budgets have a currency, start/end dates, and a total amount.
   
   During checkout, the buyer selects a cost center. The system checks if the order amount is within the remaining budget for that cost center. If the budget is exceeded, the order requires approval or is rejected. This provides financial control over B2B purchasing.
   </details>

3. **How does the approval process determine who approves?**
   <details>
   <summary>Answer</summary>
   The approval process uses **B2BPermissions** and **approval thresholds**:
   1. When an order is placed, the system evaluates it against the buyer's permissions (order threshold, budget exceeded, etc.)
   2. If the order exceeds a permission limit, it enters an approval workflow
   3. The system looks for approvers by traversing up the B2BUnit hierarchy — it finds users in the `b2bapprovergroup` who belong to the same unit or a parent unit
   4. Approvers are notified and can approve or reject via Backoffice or the storefront
   5. Multiple levels of approval can be configured (e.g., manager approves up to $5K, director approves up to $50K)
   
   Permission types include: `B2BOrderThresholdPermission` (per-order limit), `B2BBudgetExceededPermission` (budget check), `B2BOrderThresholdTimespanPermission` (spending over time).
   </details>

4. **What is punchout and when is it used?**
   <details>
   <summary>Answer</summary>
   Punchout is a procurement integration protocol (typically cXML or OCI) that allows B2B buyers to access a supplier's commerce storefront directly from their procurement system (e.g., SAP Ariba, Coupa, SAP SRM). The flow:
   1. Buyer clicks "shop" in their procurement system
   2. A punchout request (cXML SetupRequest) is sent to the Commerce storefront
   3. The buyer browses and fills their cart on the Commerce storefront
   4. At checkout, the cart contents are sent back to the procurement system as a PunchOutOrderMessage
   5. The actual purchase order is processed through the buyer's procurement workflow
   
   It's used when enterprise buyers must route all purchases through their procurement system for compliance, budgeting, and approval, while still leveraging a rich supplier storefront experience.
   </details>

5. **How do B2B catalogs differ from B2C?**
   <details>
   <summary>Answer</summary>
   Key differences:
   - **Customer-specific pricing** — B2B uses price lists per customer/unit with negotiated contract prices, volume discounts, and tiered pricing, rather than a single public price
   - **Restricted catalogs** — Different B2B units can see different product assortments. Catalog visibility is controlled per B2BUnit via `catalogVersion` access restrictions
   - **Custom catalogs** — Some B2B scenarios use customer-specific catalogs with custom SKUs, descriptions, or bundled products unique to that customer's agreement
   - **Product references** — B2B often uses customer material numbers (their internal part numbers mapped to supplier product codes)
   - **Minimum order quantities** — B2B catalogs often enforce MOQs, pack sizes, and unit-of-measure conversions not typical in B2C
   </details>

---
**Previous**: [← 10 - Order Management](../10-order-management/README.md) | **Next**: [12 - Backoffice & Security →](../12-backoffice-security/README.md)