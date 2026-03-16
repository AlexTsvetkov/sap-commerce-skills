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
- What's the difference between `ant all` and `ant production`?
- How does CCv2 handle database migrations?
- What are aspects and why are they important?
- How do cloud properties override each other?
- What happens during a rolling deployment?

---
**Previous**: [← 07 - Storefront & OCC](../07-storefront-occ/README.md) | **Next**: [09 - Performance & Caching →](../09-performance/README.md)