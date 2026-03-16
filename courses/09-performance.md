# Course 09: Performance, Caching & Search (Solr)

## 🎯 Overview
Optimize SAP Commerce performance through caching strategies, Solr search configuration, database tuning, and monitoring.

**Duration**: 8-10 hours | **Level**: Advanced | **Prerequisites**: Courses 01-06

---

## Topics

### 1. Caching Architecture
SAP Commerce has multiple cache layers:

| Layer | Technology | Scope |
|-------|-----------|-------|
| Region Cache | Ehcache/Redis | Type system entities |
| HTTP Cache | Varnish / CDN | Static & OCC responses |
| Solr Cache | Solr built-in | Search results |
| Spring Cache | Spring `@Cacheable` | Custom services |
| Session Cache | In-memory | User session data |

### 2. Region Cache Configuration
```properties
# local.properties
regioncache.entityregion.size=100000
regioncache.entityregion.evictionpolicy=LRU
regioncache.typesystemregion.size=50000
regioncache.queriesregion.size=10000

# Monitor cache in HAC → Monitoring → Cache
```

### 3. FlexibleSearch Query Caching
```properties
# Enable query cache
flexiblesearch.cache.enabled=true
flexiblesearch.cache.size=50000
flexiblesearch.cache.ttl=3600
```

```java
// Force bypass cache for specific query
FlexibleSearchQuery query = new FlexibleSearchQuery("...");
query.setDisableCaching(true);
```

### 4. Solr Search Configuration
```impex
# Define Solr index
INSERT_UPDATE SolrFacetSearchConfig;name[unique=true];indexNamePrefix;languages(isocode);currencies(isocode);solrServerConfig(name);solrSearchConfig(description)
;myStoreIndex;mystore;en,de;USD,EUR;Default;Default

# Indexed properties
INSERT_UPDATE SolrIndexedProperty;solrIndexedType(identifier)[unique=true];name[unique=true];type(code);sortableType(code);fieldValueProvider;facet;facetType(code)
;myStoreProductType;code;string;;;false;
;myStoreProductType;name;text;;;false;
;myStoreProductType;price;double;;;true;MultiSelectOr
;myStoreProductType;category;string;;categoryCodeValueProvider;true;Refine
;myStoreProductType;inStockFlag;boolean;;productInStockFlagValueProvider;true;MultiSelectOr
```

Custom value provider:
```java
public class MyCustomValueProvider extends AbstractPropertyFieldValueProvider {
    @Override
    public Collection<FieldValue> getFieldValues(IndexConfig config, 
            IndexedProperty property, Object model) {
        ProductModel product = (ProductModel) model;
        List<FieldValue> values = new ArrayList<>();
        values.add(new FieldValue(property.getName(), product.getMyCustomField()));
        return values;
    }
}
```

### 5. Solr Operations
```bash
# Full index
ant solrIndex -DsolrFacetSearchConfig=myStoreIndex

# Via HAC Groovy console
import de.hybris.platform.solrfacetsearch.indexer.impl.*
def config = solrFacetSearchConfigDao.getConfiguration("myStoreIndex")
solrIndexerService.performFullIndex(config)
```

### 6. Database Performance
```properties
# Connection pool
db.pool.maxActive=100
db.pool.maxIdle=50
db.pool.minIdle=10

# Query logging (dev only)
db.log.sql=true
db.log.sql.parameters=true
```

Key FlexibleSearch tips:
- Always use parameterized queries (avoid string concatenation)
- Use `{pk}` in SELECT, not `{*}` when you don't need all columns
- Add indexes for frequently queried attributes in `items.xml`:
```xml
<indexes>
    <index name="codeIdx">
        <key attribute="code"/>
    </index>
</indexes>
```

### 7. Monitoring & Profiling
| Tool | Access | Purpose |
|------|--------|---------|
| HAC Monitoring | `/hac/monitoring/` | JVM, cache, threads |
| Performance tab | HAC → Monitoring → Performance | SQL timing |
| Dynatrace/AppDynamics | External APM | Full stack monitoring |
| JMX | Port 9003 | JVM metrics |

---

## Exercises
1. Configure Solr indexing for a custom product attribute, run full index, verify in Solr Admin
2. Enable query caching, run the same FlexibleSearch twice, compare execution times in HAC
3. Add a database index on a custom attribute and measure query improvement
4. Configure a custom value provider for Solr that computes stock availability

## Self-Check

1. **What caching layers exist in SAP Commerce?**
   <details>
   <summary>Answer</summary>
   SAP Commerce has multiple caching layers:
   - **Type System Cache** — caches type metadata (always on)
   - **Entity Cache (Region Cache)** — caches individual items by PK, configured via cache regions in `*-cache.xml`
   - **Query Cache** — caches FlexibleSearch query results
   - **CDN Cache** — external CDN for static assets and media (Akamai, CloudFront)
   - **HTTP/Browser Cache** — client-side caching via Cache-Control headers
   - **Solr Cache** — Solr's internal filter/query caches
   
   Region Cache is the most important — it's configured per type and dramatically reduces database queries.
   </details>

2. **How do you invalidate cached data?**
   <details>
   <summary>Answer</summary>
   Cache invalidation happens at multiple levels:
   - **Automatic** — when `modelService.save()` or `modelService.remove()` is called, the entity cache for that item is invalidated automatically
   - **Cluster broadcast** — invalidation events are propagated across cluster nodes via the cluster communication channel
   - **Manual** — via `CacheController` or HAC cache management for specific regions
   - **Time-based (TTL)** — cache regions can be configured with `eviction` time after which entries expire
   - **Full flush** — clear all caches via HAC (should be avoided in production)
   
   In a cluster, invalidation must propagate to all nodes to prevent stale data.
   </details>

3. **What's the difference between a full and incremental Solr index?**
   <details>
   <summary>Answer</summary>
   - **Full index** — drops the existing index and rebuilds it completely from the database. Used for initial indexing or when the index is corrupted. Takes longer (minutes to hours depending on catalog size) and temporarily affects search quality during rebuild.
   - **Incremental (update) index** — only indexes items that have changed since the last index run. Uses a modification timestamp to detect changes. Much faster (seconds to minutes) and non-disruptive. Typically run as a CronJob every few minutes.
   
   Best practice: full index during deployment/initialization, incremental index via CronJob (e.g., every 5 minutes).
   </details>

4. **How do you add a facet to search results?**
   <details>
   <summary>Answer</summary>
   Add a `SolrIndexedProperty` with `facet=true` in your Solr configuration ImpEx:
   ```impex
   INSERT_UPDATE SolrIndexedProperty; solrIndexedType(identifier); name; type(code); facet; facetType(code); visible
   ; $indexedType ; brand ; string ; true ; Refine ; true
   ```
   Then implement a `ValueProvider` if the attribute isn't a simple type:
   ```java
   public class BrandValueProvider extends AbstractPropertyFieldValueProvider {
       public Collection<FieldValue> getFieldValues(...) {
           // Extract brand value from the product model
       }
   }
   ```
   After configuration, run a full Solr index to populate the new facet. The facet will then appear in search results via OCC/Spartacus.
   </details>

5. **When should you add a database index?**
   <details>
   <summary>Answer</summary>
   Add a database index when:
   - FlexibleSearch queries filter/sort on a column that's frequently queried but not already indexed
   - Query execution plans (via HAC SQL console or DB tools) show full table scans on large tables
   - Performance monitoring (Dynatrace) shows slow DB queries as a bottleneck
   
   Define indexes in `items.xml`:
   ```xml
   <index name="productCodeIdx">
       <key attribute="code"/>
   </index>
   ```
   **Don't** over-index — each index slows down INSERT/UPDATE operations. Focus on columns used in WHERE clauses and JOIN conditions of your most frequent queries.
   </details>

---
**Previous**: [← 08 - Build & Deployment](08-build-deployment.md) | **Next**: [10 - Order Management →](10-order-management.md)