# Magento 2 Extensions for Tideways Analysis

Load this file when the project is a Magento 2 application (`app/etc/config.php` or `bin/magento` exists). Extends the generic PHP patterns in `bottleneck-detection.md`.

## Transaction Type Interpretation

### Frontend Transactions (Service: app)

| Controller | What it is | Red flags |
|-----------|-----------|-----------|
| `Catalog\Controller\Category\View` | Category/listing page | High Impact%, SQL prefix, high memory |
| `Catalog\Controller\Product\View` | Product detail page | SQL prefix, memory spikes |
| `Cms\Controller\Index\Index` | Homepage | Should be FPC cached — if it appears frequently, FPC is misconfigured |
| `Cms\Controller\Page\View` | CMS pages | Should be FPC cached |
| `CatalogSearch\Controller\Result\Index` | Search results | SQL prefix = search engine not handling queries |
| `Checkout\Controller\Index\Index` | Checkout page | SESSION prefix = session locking |
| `Customer\Controller\Section\Load` | Customer sections AJAX | COMPILE prefix = DI issue, should be lightweight |
| `Framework\App\Action\Redirect` | Redirects | COMPILE prefix = full Magento bootstrap for a redirect |

### AJAX/API Transactions

| Controller | What it is | Red flags |
|-----------|-----------|-----------|
| `Magento\Checkout\Controller\Cart\Add` | Add to cart AJAX | Should be fast; slowness = product load or stock check issue |
| `Magento\Checkout\Controller\Sidebar\RemoveItem` | Mini-cart remove | Should be lightweight |
| `Magento\Customer\Controller\Section\Load` | Customer section data | COMPILE prefix = DI issue |
| `Magento\Catalog\Controller\Product\Compare\Add` | Product compare | Should be lightweight |
| `Magento\Review\Controller\Product\ListAjax` | Reviews AJAX | SQL prefix = missing index on review tables |
| `Magento\Wishlist\Controller\Index\Add` | Wishlist add | Should be lightweight |

**How to judge "slow":** Compare against the project's own baseline. Use Tideways History to establish what is normal for each transaction, then flag deviations.

### CLI Transactions (Service: app:cli)

| Pattern | What it is | Watch for |
|---------|-----------|-----------|
| `bin/magento cron:run` | Cron execution | Individual cron job times, memory |
| `bin/magento indexer:reindex` | Indexing | Duration, memory growth |
| `bin/magento queue:consumers:start` | Queue processing | Backlog growth, processing rate |
| `bin/magento setup:static-content:deploy` | Static deploy | Memory, total time |

### Backend Transactions (Service: backend)

Admin panel requests. Generally slower but lower priority unless admin is production-critical.

---

## Bottleneck Prefix Deep-Dive

### SQL Prefix

**Meaning:** >40% of request time spent in database queries.

**Common Magento causes:**

1. **EAV attribute loading** — Multiple self-joins on `catalog_product_entity_varchar`, `_text`, `_decimal`, `_int`, `_datetime`
   - Fix: Use flat catalog, or `addAttributeToSelect(['specific', 'attributes'])`
   - Fix: Check if `catalog_product_flat` indexer is enabled and up-to-date

2. **Collection loading in loops** (N+1) — `Collection::load()` or `Repository::getById()` inside iteration
   ```php
   // BAD: N+1 - loads full product for each item
   foreach ($items as $item) {
       $product = $productRepository->getById($item->getProductId());
   }

   // GOOD: Bulk load via collection
   $productIds = array_map(fn($item) => $item->getProductId(), $items);
   $products = $collectionFactory->create()
       ->addIdFilter($productIds)
       ->addAttributeToSelect(['name', 'sku', 'price'])
       ->getItems();
   ```

3. **Missing indexes** — Check `EXPLAIN` on slow queries for full table scans
   - Signal: Query time grows with data volume
   - Fix: Add index in `db_schema.xml`

4. **Layered navigation** — Complex aggregation/filter queries
   - Signal: Category pages with SQL prefix, `CatalogSearch\Model\Search` in stacktrace
   - Fix: Ensure ElasticSearch/OpenSearch handles facets instead of MySQL

### COMPILE Prefix

**Meaning:** >40% of time in PHP compilation, autoloading, or DI resolution.

**Common Magento causes:**

1. **OPcache misconfiguration**
   - Signal: Every request shows COMPILE, not just first
   - Fix: Verify `opcache.enable=1`, `opcache.validate_timestamps=0` in production
   - Fix: Check `opcache.max_accelerated_files` > actual file count

2. **Too many plugins on hot paths**
   - Signal: `Interceptor::___callPlugins()` deep in trace
   - Fix: Audit plugins on `Collection::load()`, `Product::getPrice()`, `Block::toHtml()`
   - Fix: Replace `around` plugins with `before`/`after` where possible

3. **Excessive DI graph** — Constructor injection chains >5 levels deep
   - Fix: Use proxy classes for heavy dependencies
   ```xml
   <!-- di.xml: Use proxy for lazy loading -->
   <type name="MyClass">
       <arguments>
           <argument name="heavyService" xsi:type="object">
               Heavy\Service\Proxy
           </argument>
       </arguments>
   </type>
   ```

4. **Composer autoload not optimized**
   - Fix: `composer dump-autoload --optimize --classmap-authoritative`

### SESSION Prefix

**Meaning:** Session read/write is the bottleneck.

**Common Magento causes:**

1. **Session locking** — Concurrent AJAX requests (especially `Customer\Controller\Section\Load`) block each other
   - Fix: Use Redis for sessions with `disable_locking=1` for read-only endpoints:
     ```php
     // env.php
     'session' => [
         'save' => 'redis',
         'redis' => [
             'host' => '127.0.0.1',
             'disable_locking' => '1',
         ]
     ]
     ```
   - Fix: Mark read-only AJAX controllers as `CsrfAwareActionInterface` without session

2. **Large session data** — Shopping cart with many items, customer data
   - Fix: Reduce session data, use database for large datasets

3. **File-based sessions on NFS**
   - Fix: Switch to Redis sessions

### HTTP Prefix

**Meaning:** External HTTP calls dominate request time.

**Common causes:**
1. Payment gateway calls (PayPal, Stripe, Adyen, etc.)
2. Shipping rate calculations (carrier APIs)
3. Search service calls (ElasticSearch, OpenSearch)
4. Email/marketing service calls (transactional email APIs)

**Analysis:** Is the call necessary? Can it be deferred (queue)? Is there a caching layer?

---

## Magento Architecture Patterns

### EAV Query Patterns

**Signature in Slow SQLs:**
- Tables: `catalog_product_entity_varchar`, `_text`, `_decimal`, `_int`, `_datetime`
- Multiple INNER JOIN / LEFT JOIN on same entity tables
- Calling function: `AbstractCollection::load()`, `Eav\Model\Entity\Collection`

**Severity:** 1 EAV query per page = normal. >3 = needs optimization. In loops = critical.

**Optimization paths:**
1. **Flat catalog** (if product count <100k): Enable in Stores > Config > Catalog > Storefront
2. **Specific attribute selection**: `$collection->addAttributeToSelect(['name', 'price', 'image'])`
3. **Pre-computed attributes**: Move calculated values to flat custom attributes
4. **ElasticSearch/OpenSearch**: Offload search/filter to search engine instead of MySQL

### Full Page Cache (FPC)

**Expected:** FPC cached pages should NOT appear in Tideways (served by Varnish/built-in cache before PHP).

**If cached pages appear:** cache hole punches, customer-specific content, missing cache tags, TTL too short.

**Diagnose:**
1. Check Page Cache Hit Rate in Performance Overview
2. In trace: look for `Magento\PageCache` in call stack
3. Search for `cacheable="false"` — one block disables FPC for the entire page:
   ```bash
   grep -r 'cacheable="false"' app/design/ app/code/ vendor/ --include="*.xml"
   ```

### Plugin/Interceptor Chain

**Detection:** `Interceptor::___callPlugins()` in stacktrace, `{closure}()` between calls, 3+ `around` plugins on same method.

**Example trace:**
```
Catalog\Product\Interceptor::load()
  → CatalogInventory\Plugin::beforeLoad()       # stock precheck
  → CatalogRule\Plugin::aroundLoad()            # wraps everything
    → Tax\Plugin::beforeLoad()                  # tax class resolution
    → Catalog\Product::load()                   # actual work
    → Review\Plugin::afterLoad()                # loads review summary
  → Wishlist\Plugin::afterLoad()                # checks wishlist state
```

**Fix:** Replace `around` with `before`/`after`, merge plugins, add early-return, check `etc/di.xml` across modules.

### Layout/Block Rendering

**In traces, look for:** `TemplateEngine\Php::render()` count, `Block::toHtml()`, `ProductListItem::getItemHtmlWithRenderer()` per product.

**Issues:**
1. **Block rendered multiple times** — Fix: `cacheable` attribute, review layout XML
2. **Product list overhead** — Fix: Optimize item template, reduce nested blocks
3. **Nested container explosion** — Fix: Flatten layout structure

### Cron & Queue

**Common issues:**
1. **Indexer during peak hours** — Use `Update by Schedule` instead of `Update on Save`
2. **Queue consumer memory leaks** — Set `--max-messages`, restart via supervisord
3. **Catalog price rule indexing** — Can be very slow on large catalogs, consider partial indexing

### Cold DI Container

Magento's DI container compiles class maps and builds the full object graph on first request.

**Fix:** Run `bin/magento setup:di:compile` during deployment, verify `generated/` is populated, warm-up script after deploy.

### Memory Growth

- Large collection loads without `setPageSize()` / `setCurPage()`
- Use `$collection->clear()` after processing batches
- Event observers storing data in class properties across iterations
- Full EAV loads (`addAttributeToSelect('*')`) instead of specific attributes

### Config Cache Misses

```sql
-- If these appear in Slow SQLs, config cache is broken:
SELECT * FROM core_config_data WHERE path = 'web/secure/base_url';
```
**Fix:** Verify `config` cache is enabled: `bin/magento cache:status`

---

## Infrastructure Patterns

### PHP-FPM & OPcache

| Pattern | Likely Cause | Fix |
|---------|-------------|-----|
| First request very slow, subsequent fast | Cold OPcache | Warmup script, `opcache.preload` |
| All requests have COMPILE prefix | OPcache disabled/full | Check `opcache.max_accelerated_files` |
| Memory spikes per request >300MB | FPM worker memory limit | Set `pm.max_requests`, check leaks |
| Response time degrades over hours | Worker pool exhaustion | Check `pm.max_children` |
| Intermittent extreme slowness | FPM pool restart | Check `pm.max_requests`, OOM kills |

### Redis/Cache

- Low Page Cache Hit Rate = FPC misconfiguration
- High session operation time = Redis overloaded or network latency
- COMPILE prefix on cached pages = cache cleared/expired

**Diagnostic questions:**
1. Is Redis memory sufficient? (eviction happening?)
2. Are cache tags generating too many entries?
3. Is session locking causing request serialization?
4. Is the Redis connection shared with other applications?
