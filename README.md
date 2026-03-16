# SAP Commerce Cloud — Skills, Training & Pre-Sale Materials

A comprehensive knowledge base for SAP Commerce Cloud (Hybris) covering training courses, hands-on projects, pre-sale materials, and integration documentation.

---

## 📚 Courses

12 structured courses for experienced Java developers learning SAP Commerce Cloud. Each course includes detailed explanations, code examples, exercises, and self-check steps.

| # | Course | Duration | Topics |
|---|--------|----------|--------|
| 01 | [Architecture](courses/01-architecture.md) | 4-6 hrs | Platform overview, layers, extension model, Spring integration |
| 02 | [Type System](courses/02-type-system.md) | 6-8 hrs | items.xml, ItemTypes, Relations, Enums, generated models |
| 03 | [Extensions](courses/03-extensions.md) | 4-6 hrs | Extension types, creation, dependencies, AddOns, templates |
| 04 | [Installation & Setup](courses/04-installation.md) | 4-6 hrs | Local install, recipes, DB config, IDE setup, CCv2 |
| 05 | [Service Layer](courses/05-service-layer.md) | 6-8 hrs | Services, DAOs, Interceptors, Events, FlexibleSearch |
| 06 | [ImpEx & Data Management](courses/06-impex.md) | 4-6 hrs | ImpEx syntax, macros, translators, hotfolder, data migration |
| 07 | [Storefront & OCC API](courses/07-storefront-occ.md) | 6-8 hrs | Spartacus, OCC REST API, CMS, SmartEdit, SSR |
| 08 | [Build & Deployment](courses/08-build-deployment.md) | 4-6 hrs | CCv2, manifest.json, CI/CD, environments, monitoring |
| 09 | [Performance & Caching](courses/09-performance.md) | 4-6 hrs | Caching, Solr search, DB optimization, load testing |
| 10 | [Order Management](courses/10-order-management.md) | 4-6 hrs | Order process, fulfillment, payment, returns, business process engine |
| 11 | [B2B Commerce](courses/11-b2b-commerce.md) | 6-8 hrs | Organizations, approvals, punchout, quotes, B2B checkout |
| 12 | [Backoffice & Security](courses/12-backoffice-security.md) | 4-6 hrs | Backoffice framework, custom widgets, user roles, security |

**Total estimated learning time: 60-80 hours**

---

## 🔧 Training Project

| Resource | Description |
|----------|-------------|
| [Training Project Guide](training-project/README.md) | Hands-on project: build "TechMart" — a B2C electronics store from scratch covering data model, services, ImpEx, Solr, OCC API, Spartacus, Backoffice, and CCv2 deployment (40-60 hours) |

---

## 💼 Pre-Sale Materials

Documents and presentation content for the sales team to showcase SAP Commerce expertise to potential clients.

| Document | Purpose |
|----------|---------|
| [Expertise Overview](pre-sale/01-commerce-expertise-overview.md) | Capabilities matrix, engagement models, differentiators |
| [Technical Capabilities Deck](pre-sale/02-technical-capabilities-deck.md) | Slide-ready content: architecture, B2C/B2B features, team composition |
| [Case Studies & References](pre-sale/03-case-studies-references.md) | 4 scenario-based case studies demonstrating delivery capability |

---

## 🔗 Integration Documentation

Technical guides for integrating SAP Commerce with S/4HANA and other systems.

| Document | Integration Pattern |
|----------|-------------------|
| [S/4HANA via CPI](integrations/01-commerce-s4hana-cpi-integration.md) | **Primary**: Product, pricing, order, inventory, customer sync via SAP Integration Suite |
| [Direct API Integration](integrations/02-direct-api-integration.md) | Direct OData/REST calls, webhooks, authentication patterns |
| [Alternative Scenarios](integrations/03-alternative-integration-scenarios.md) | IDoc, Event Mesh, Kyma, hotfolder, iPaaS, CDC, payment providers |

---

## 🗂️ Repository Structure

```
sap-commerce-skills/
├── README.md                          ← You are here
├── courses/
│   ├── 01-architecture.md
│   ├── 02-type-system.md
│   ├── 03-extensions.md
│   ├── 04-installation.md
│   ├── 05-service-layer.md
│   ├── 06-impex.md
│   ├── 07-storefront-occ.md
│   ├── 08-build-deployment.md
│   ├── 09-performance.md
│   ├── 10-order-management.md
│   ├── 11-b2b-commerce.md
│   └── 12-backoffice-security.md
├── training-project/
│   └── README.md
├── pre-sale/
│   ├── 01-commerce-expertise-overview.md
│   ├── 02-technical-capabilities-deck.md
│   └── 03-case-studies-references.md
└── integrations/
    ├── 01-commerce-s4hana-cpi-integration.md
    ├── 02-direct-api-integration.md
    └── 03-alternative-integration-scenarios.md
```

---

## 🚀 Getting Started

1. **New to Commerce?** Start with [Course 01 — Architecture](courses/01-architecture.md)
2. **Want hands-on?** Jump to the [Training Project](training-project/README.md)
3. **Preparing for a client pitch?** Check the [Pre-Sale Materials](pre-sale/)
4. **Planning an integration?** Read the [Integration Docs](integrations/)