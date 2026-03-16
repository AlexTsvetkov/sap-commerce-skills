# Course 08: Build, Deployment & CCv2

## 🎯 Overview
Master the SAP Commerce build system, CCv2 cloud deployment, manifest configuration, environment management, and CI/CD pipelines.

**Duration**: 6-8 hours | **Level**: Intermediate-Advanced | **Prerequisites**: Courses 01-04

---

## Topics

### 1. Build System (Ant)
```bash
ant clean all          # Full rebuild
ant all                # Incremental build
ant initialize         # Init DB (destroys data!)
ant updatesystem       # Update DB schema
ant unittests          # Run unit tests
ant integrationtests   # Run integration tests
ant production         # Create production package
```

### 2. CCv2 Manifest (manifest.json)
```json
{
  "commerceSuiteVersion": "2211.28",
  "useCloudExtensionPack": true,
  "extensions": [
    "modeltacceleratorservices",
    "commercewebservices",
    "backoffice",
    "myprojectcore",
    "myprojectfacades",
    "myprojectstorefront",
    "myprojectinitialdata"
  ],
  "storefrontAddons": [
    { "addon": "smarteditaddon", "storefront": "myprojectstorefront", "template": "yacceleratorstorefront" }
  ],
  "aspects": [
    {
      "name": "backoffice",
      "properties": [
        { "key": "cluster.node.groups", "value": "integration,yHotfolderCandidate,backgroundProcessing" }
      ],
      "webapps": [
        { "name": "backoffice", "contextPath": "/backoffice" },
        { "name": "hac", "contextPath": "/hac" }
      ]
    },
    {
      "name": "accstorefront",
      "webapps": [
        { "name": "myprojectstorefront", "contextPath": "" },
        { "name": "commercewebservices", "contextPath": "/occ" }
      ]
    }
  ],
  "tests": {
    "extensions": ["myprojectcore", "myprojectfacades"]
  }
}
```

### 3. Cloud Properties Hierarchy
```
config/cloud/
├── common.properties          # All environments
├── persona/
│   ├── development.properties # Dev environment
│   ├── staging.properties     # Staging
│   └── production.properties  # Production
└── local-dev.properties       # Local development only
```

Priority: `local.properties` > `persona/<env>.properties` > `common.properties` > `project.properties`

### 4. CCv2 Deployment Flow
```
Code Push → Cloud Build → Deploy to Environment
   │            │              │
   ├─ Git push  ├─ ant clean   ├─ Rolling update
   │            │    all       ├─ DB migration
   │            ├─ Run tests   ├─ System update
   │            └─ Package     └─ Health check
```

### 5. CCv2 CLI / Portal Operations
```bash
# Builds
ccv2 build create --branch main --name "Release 1.0"
ccv2 build list
ccv2 build get --code BUILD_CODE

# Deployments
ccv2 deployment create --build-code BUILD123 --environment-code d1
ccv2 deployment list --environment-code d1

# Endpoints
ccv2 endpoint list --environment-code d1
```

### 6. Environment Aspects
| Aspect | Purpose | Typical Webapps |
|--------|---------|-----------------|
| `backoffice` | Admin, background jobs | Backoffice, HAC |
| `accstorefront` | Customer-facing | Storefront, OCC |
| `backgroundProcessing` | CronJobs, imports | HAC only |
| `api` | API-only | OCC, webhook endpoints |

### 7. Production Packaging
```bash
ant production -Dproduction.legacy.mode=false
# Creates: hybris/temp/hybris/hybrisServer/*.zip
```

### 8. CI/CD Pipeline Example (Jenkins)
```groovy
pipeline {
    stages {
        stage('Build') {
            steps { sh 'cd hybris/bin/platform && ant clean all' }
        }
        stage('Test') {
            steps { sh 'cd hybris/bin/platform && ant unittests' }
        }
        stage('Cloud Build') {
            steps { sh 'ccv2 build create --branch ${env.BRANCH_NAME}' }
        }
        stage('Deploy to Dev') {
            steps { sh 'ccv2 deployment create --build-code ${BUILD_CODE} --environment-code dev' }
        }
    }
}
```

---

## Exercises
1. Create a `manifest.json` for a B2C project with Backoffice and Storefront aspects
2. Configure environment-specific properties (dev: debug logging, prod: caching)
3. Build a production package locally and examine its structure
4. Create a CI/CD pipeline that builds, tests, and deploys to a dev environment

## Self-Check

1. **What's the difference between `ant all` and `ant production`?**
   <details>
   <summary>Answer</summary>
   `ant all` is used for local development — it compiles all source code, generates model classes, builds web applications, and prepares the system for running locally. It includes debug information and all development tooling. `ant production` creates an optimized production deployment package — it pre-compiles everything, creates a distributable archive (ZIP), strips debug info, and packages only what's needed for deployment. On CCv2, `ant production` is automatically executed during the cloud build process.
   </details>

2. **How does CCv2 handle database migrations?**
   <details>
   <summary>Answer</summary>
   CCv2 runs a **system update** automatically during deployment. When a new build is deployed to an environment, the platform compares the current type system definition (from `items.xml` files) with the existing database schema. It then generates and executes DDL statements to add new tables, columns, and indexes. It does NOT remove columns/tables (to prevent data loss). For data migrations, you use ImpEx scripts in `essentialdata` or `projectdata` files, or custom update hooks via `SystemSetup` annotations.
   </details>

3. **What are aspects and why are they important?**
   <details>
   <summary>Answer</summary>
   Aspects define different pod types (deployment units) in CCv2. Each aspect specifies which web applications it runs and can be independently scaled. Common aspects:
   - **accstorefront** — serves the storefront and OCC API (scales for traffic)
   - **backoffice** — runs Backoffice admin UI (typically 1 pod)
   - **backgroundProcessing** — runs CronJobs, task engine, HAC (typically 1 pod)
   
   Aspects are important because they enable separation of concerns — storefront pods can scale horizontally for traffic without scaling the backoffice, and background jobs don't compete for resources with customer-facing requests.
   </details>

4. **How do cloud properties override each other?**
   <details>
   <summary>Answer</summary>
   Properties in CCv2 follow a precedence order (later overrides earlier):
   1. Platform defaults (built into extensions)
   2. Extension `project.properties` files
   3. `manifest.json` → `useConfig.properties` files (in order listed)
   4. Aspect-specific properties (applied only to that aspect)
   5. Environment-specific properties set via Cloud Portal
   6. Service properties (managed by SAP — e.g., DB connection)
   
   This allows you to define common properties at the base level and override per aspect or environment (e.g., different cache settings for storefront vs. backgroundProcessing).
   </details>

5. **What happens during a rolling deployment?**
   <details>
   <summary>Answer</summary>
   CCv2 uses rolling deployments to minimize downtime: (1) New pods with the updated build are started alongside existing pods. (2) Once new pods pass health checks, traffic is gradually routed to them. (3) A system update runs on one pod (typically backgroundProcessing) to update the database schema. (4) Old pods are terminated. During this process, the storefront remains available. There can be a brief period where old and new code run simultaneously, so changes must be backward-compatible. For breaking changes, a "maintenance mode" deployment with downtime may be required.
   </details>

---
**Previous**: [← 07 - Storefront & OCC](07-storefront-occ.md) | **Next**: [09 - Performance & Caching →](09-performance.md)