# Course 04: SAP Commerce Installation & Setup

## 🎯 Course Overview

This course walks through installing SAP Commerce Cloud locally for development, configuring the platform, setting up a database, and preparing your first project. It also covers CCv2 cloud setup.

**Duration**: 4-6 hours  
**Level**: Beginner-Intermediate  
**Prerequisites**: Java 11+, basic command-line skills

---

## 📋 Table of Contents

1. [System Requirements](#1-system-requirements)
2. [Downloading SAP Commerce](#2-downloading-sap-commerce)
3. [Local Installation](#3-local-installation)
4. [Configuration (local.properties)](#4-configuration)
5. [Database Setup](#5-database-setup)
6. [Building & Starting](#6-building--starting)
7. [System Initialization](#7-system-initialization)
8. [IDE Setup (IntelliJ / Eclipse)](#8-ide-setup)
9. [CCv2 Cloud Setup](#9-ccv2-cloud-setup)
10. [Troubleshooting](#10-troubleshooting)
11. [Exercises](#11-exercises)
12. [Self-Check Questions](#12-self-check-questions)

---

## 1. System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| **OS** | Windows 10, macOS 12+, Linux | Linux (Ubuntu 22.04) |
| **Java** | SAP Machine JDK 17 | SAP Machine JDK 17 |
| **RAM** | 8 GB | 16 GB+ |
| **Disk** | 20 GB free | 50 GB+ SSD |
| **Database** | HSQLDB (bundled) | MySQL 8.0 / SAP HANA |
| **Build tool** | Apache Ant (bundled) | Bundled Ant |

### Java Setup

```bash
# Verify Java version
java -version
# Expected: openjdk version "17.x.x" (SAP Machine)

# Set JAVA_HOME
export JAVA_HOME=/path/to/sapmachine-jdk-17
export PATH=$JAVA_HOME/bin:$PATH
```

---

## 2. Downloading SAP Commerce

### From SAP Software Downloads

1. Go to [SAP Software Downloads](https://launchpad.support.sap.com/#/softwarecenter)
2. Search for "SAP Commerce Cloud"
3. Download the ZIP for your version (e.g., `CXCOMM2211.zip`)
4. Also download any required patches/hotfixes

### Package Contents

```
CXCOMM2211.zip
├── hybris/
│   ├── bin/
│   │   ├── platform/          # Core platform
│   │   ├── modules/           # All SAP modules/extensions
│   │   └── custom/            # Your custom extensions go here
│   ├── config/                # Configuration directory
│   │   ├── local.properties   # (created by you)
│   │   └── localextensions.xml # (created by you)
│   └── temp/
└── installer/                  # Installation recipes
```

---

## 3. Local Installation

### Step-by-Step Setup

```bash
# 1. Extract the archive
unzip CXCOMM2211.zip -d /opt/commerce

# 2. Navigate to platform
cd /opt/commerce/hybris/bin/platform

# 3. Set up environment
source ./setantenv.sh    # Linux/Mac
# OR
setantenv.bat            # Windows

# 4. Verify Ant
ant -version

# 5. Generate initial config from a recipe
cd /opt/commerce/installer
./install.sh -r b2c_acc     # B2C Accelerator recipe
# OR
./install.sh -r b2c_acc_plus  # B2C with additional modules
```

### Available Recipes

| Recipe | Description |
|--------|-------------|
| `b2c_acc` | B2C Accelerator (basic storefront) |
| `b2c_acc_plus` | B2C with additional features |
| `b2b_acc` | B2B Accelerator |
| `b2b_acc_plus` | B2B with additional features |
| `cx` | Full CX suite (Commerce + Marketing + Sales) |

### Manual Setup (Without Recipes)

```bash
# 1. Create config directory
mkdir -p /opt/commerce/hybris/config

# 2. Copy template configurations
cp /opt/commerce/hybris/bin/platform/resources/advanced.properties \
   /opt/commerce/hybris/config/local.properties

# 3. Create localextensions.xml
cat > /opt/commerce/hybris/config/localextensions.xml << 'EOF'
<hybrisconfig>
  <extensions>
    <path dir="${HYBRIS_BIN_DIR}"/>
    <extension name="backoffice"/>
    <extension name="commerceservices"/>
    <extension name="commercefacades"/>
    <extension name="acceleratorstorefrontcommons"/>
    <extension name="yacceleratorstorefront"/>
    <extension name="yacceleratorinitialdata"/>
    <extension name="solrserver"/>
  </extensions>
</hybrisconfig>
EOF
```

---

## 4. Configuration

### 4.1 local.properties

Key configuration properties:

```properties
# ─── Database ───
db.url=jdbc:mysql://localhost:3306/commerce?useSSL=false&allowPublicKeyRetrieval=true&useUnicode=true&characterEncoding=utf8
db.driver=com.mysql.cj.jdbc.Driver
db.username=commerce
db.password=commerce123
db.tableprefix=

# For HSQLDB (development — no setup needed)
# db.url=jdbc:hsqldb:file:${HYBRIS_DATA_DIR}/hsqldb/mydb;shutdown=true;hsqldb.log_size=500
# db.driver=org.hsqldb.jdbcDriver
# db.username=sa
# db.password=

# ─── Server ───
tomcat.http.port=9001
tomcat.ssl.port=9002
tomcat.ajp.port=8009

# ─── Cluster ───
cluster.id=0
cluster.broadcast.methods=jgroups
cluster.broadcast.method.jgroups=jgroups-udp.xml

# ─── Solr ───
solrserver.instances.default.autostart=true
solrserver.instances.default.port=8983

# ─── Media ───
media.read.dir=/opt/commerce/hybris/data/media
media.globalSettings.maxFileSize=10485760

# ─── Development ───
build.development.mode=true
tomcat.development.mode=true
backoffice.fill.typefacade.cache.on.startup=false

# ─── Performance (Development) ───
cronjob.timertask.loadonstartup=false
task.engine.loadonstartup=false
```

### 4.2 Key Property Categories

| Category | Properties |
|----------|-----------|
| Database | `db.url`, `db.driver`, `db.username`, `db.password` |
| Server | `tomcat.http.port`, `tomcat.ssl.port` |
| Cluster | `cluster.id`, `cluster.broadcast.methods` |
| Solr | `solrserver.instances.default.*` |
| Media | `media.read.dir`, `media.default.url.strategy` |
| Email | `mail.smtp.server`, `mail.from` |
| Build | `build.development.mode` |

---

## 5. Database Setup

### 5.1 MySQL Setup

```sql
-- Create database
CREATE DATABASE commerce CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user
CREATE USER 'commerce'@'localhost' IDENTIFIED BY 'commerce123';

-- Grant privileges
GRANT ALL PRIVILEGES ON commerce.* TO 'commerce'@'localhost';
FLUSH PRIVILEGES;
```

Place MySQL JDBC driver in `hybris/bin/platform/lib/dbdriver/`:
```bash
cp mysql-connector-java-8.0.33.jar hybris/bin/platform/lib/dbdriver/
```

### 5.2 HSQLDB (Development Only)

No setup required — HSQLDB is embedded. Use this config:
```properties
db.url=jdbc:hsqldb:file:${HYBRIS_DATA_DIR}/hsqldb/mydb;shutdown=true
db.driver=org.hsqldb.jdbcDriver
db.username=sa
db.password=
```

### 5.3 Docker-Based MySQL

```yaml
# docker-compose.yml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: commerce
      MYSQL_USER: commerce
      MYSQL_PASSWORD: commerce123
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

volumes:
  mysql_data:
```

---

## 6. Building & Starting

### 6.1 Build Commands

```bash
cd hybris/bin/platform
source ./setantenv.sh

# Full clean build
ant clean all

# Build only (no clean)
ant all

# Build specific extension
ant build -Dextname=myprojectcore
```

### 6.2 Starting the Server

```bash
# Start in foreground (see output in terminal)
./hybrisserver.sh

# Start in background (daemon mode)
./hybrisserver.sh start

# Stop
./hybrisserver.sh stop

# Debug mode (remote debugging on port 8000)
./hybrisserver.sh debug
```

### 6.3 Access Points

| URL | Purpose |
|-----|---------|
| `https://localhost:9002/hac` | Hybris Administration Console |
| `https://localhost:9002/backoffice` | Backoffice |
| `https://localhost:9002/yacceleratorstorefront` | Storefront |
| `https://localhost:9002/occ/v2` | OCC REST API |
| `http://localhost:8983/solr` | Solr Admin |

Default credentials: `admin` / `nimda`

---

## 7. System Initialization

### 7.1 Via HAC

1. Navigate to `https://localhost:9002/hac`
2. Log in with `admin` / `nimda`
3. Go to **Platform → Initialization**
4. Select desired options:
   - **Initialize** (creates schema + imports data)
   - **Update** (updates schema only)
5. Click **Initialize/Update**

### 7.2 Via Command Line

```bash
# Initialize (DESTROYS all data)
ant initialize

# System update (preserves data)
ant updatesystem

# Initialize with specific tenant
ant initialize -Dtenant=master
```

### 7.3 Via API

```bash
# Trigger update via REST
curl -X POST https://localhost:9002/hac/platform/update \
  -H "Content-Type: application/json" \
  -u admin:nimda
```

---

## 8. IDE Setup

### 8.1 IntelliJ IDEA

```bash
# Generate IntelliJ project files
cd hybris/bin/platform
ant idea
```

Then in IntelliJ:
1. **File → Open** → Select `hybris/bin/platform/hybris.ipr`
2. Configure SDK: **File → Project Structure → SDKs** → Add SAP Machine JDK 17
3. Set module SDK for all modules

### 8.2 Eclipse

```bash
# Generate Eclipse project files
ant eclipseprojectfiles
```

Then in Eclipse:
1. **File → Import → Existing Projects into Workspace**
2. Select `hybris/bin/platform`
3. Import all projects

### 8.3 Remote Debugging

Start server in debug mode:
```bash
./hybrisserver.sh debug
```

In IntelliJ:
1. **Run → Edit Configurations → + → Remote JVM Debug**
2. Port: `8000`
3. Click **Debug**

---

## 9. CCv2 Cloud Setup

### 9.1 Repository Structure for CCv2

```
my-commerce-project/
├── core-customize/
│   ├── hybris/
│   │   ├── bin/custom/           # Custom extensions
│   │   └── config/
│   │       └── cloud/            # Cloud-specific config
│   │           ├── common.properties
│   │           ├── persona/
│   │           │   ├── development.properties
│   │           │   ├── staging.properties
│   │           │   └── production.properties
│   │           └── solr/
│   └── manifest.json             # Build manifest
├── js-storefront/                 # Spartacus app
│   └── spartacusstore/
└── datahub/                       # (if needed)
```

### 9.2 Cloud Portal

1. Access [SAP Commerce Cloud Portal](https://portal.commerce.ondemand.com/)
2. Create a new subscription/environment
3. Connect your Git repository
4. Create a build from your branch
5. Deploy the build to an environment

### 9.3 CCV2 CLI

```bash
# Install CCV2 CLI
npm install -g @sap/ccv2-cli

# Login
ccv2 login

# Create build
ccv2 build create --branch main --name "Release 1.0"

# Deploy
ccv2 deployment create --build-code BUILD123 --environment-code dev
```

---

## 10. Troubleshooting

| Issue | Solution |
|-------|----------|
| `JAVA_HOME not set` | Export `JAVA_HOME` to SAP Machine JDK path |
| `Port 9001/9002 in use` | Change ports in `local.properties` or kill existing process |
| `Out of memory` | Add to `local.properties`: `tomcat.generaloptions=-Xms2G -Xmx4G` |
| `Database connection failed` | Verify DB is running, check credentials and JDBC driver |
| `Type system outdated` | Run system update via HAC or `ant updatesystem` |
| `Build fails with compilation errors` | Run `ant clean all` to rebuild from scratch |
| `Extension not found` | Check `localextensions.xml` and extension path |

---

## 11. Exercises

### Exercise 1: Fresh Installation
1. Download SAP Commerce 2211
2. Install using the `b2c_acc` recipe
3. Configure MySQL as the database
4. Build and initialize the system
5. Access HAC, Backoffice, and Storefront

### Exercise 2: Custom Project Setup
1. Create a custom project structure with extensions: `trainingcore`, `trainingfacades`, `trainingstorefront`, `traininginitialdata`
2. Configure `localextensions.xml` with all dependencies
3. Build and verify all extensions load correctly

### Exercise 3: IDE Configuration
1. Generate IntelliJ project files
2. Set up remote debugging
3. Set a breakpoint in `DefaultProductService.getProductForCode()`
4. Debug a product lookup via HAC FlexibleSearch

---

## 12. Self-Check Questions

1. **What is the default admin password?**
   <details><summary>Answer</summary>`nimda` (admin reversed)</details>

2. **What command initializes the system from command line?**
   <details><summary>Answer</summary>`ant initialize` — but be careful, it destroys all existing data.</details>

3. **Where do custom extensions go in the directory structure?**
   <details><summary>Answer</summary>In `hybris/bin/custom/` directory, and they must be registered in `config/localextensions.xml`.</details>

4. **What's the difference between `ant all` and `ant clean all`?**
   <details><summary>Answer</summary>`ant all` does an incremental build. `ant clean all` deletes all generated/compiled files first and does a full rebuild. Use `clean all` when you change type system definitions.</details>

---

**Previous Course**: [← 03 - Extensions](03-extensions.md)  
**Next Course**: [05 - Service Layer & Business Logic →](05-service-layer.md)