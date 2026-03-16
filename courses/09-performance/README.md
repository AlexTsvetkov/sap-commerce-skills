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
- What caching layers exist in SAP Commerce?
- How do you invalidate cached data?
- What's the difference between a full and incremental Solr index?
- How do you add a facet to search results?
- When should you add a database index?

---
**Previous**: [← 08 - Build & Deployment](../08-build-deployment/README.md) | **Next**: [10 - Order Management →](../10-order-management/README.md)