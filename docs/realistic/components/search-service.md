# Search Service

## Description

The Search Service is TechCorp's search microservice, responsible for providing full-text search capabilities, advanced filters, and personalized relevance for products, orders, and platform content.

The service encapsulates Elasticsearch complexity, offering a simplified API for consumers. It implements features such as autocomplete, spell correction, synonym search, popularity boosting, and faceted filters.

Indexing is performed in near real-time through events consumed from the message broker. The search-service maintains optimized indexes for different use cases (products, orders, users) with specific mappings and analyzers for Brazilian Portuguese.

## Owners

- **Team:** Platform Engineering
- **Tech Lead:** Lucas Ferreira
- **Slack:** #platform-search

## Technology Stack

- Language: Python 3.11
- Framework: FastAPI 0.104
- Search Engine: Elasticsearch 8.11
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `SEARCH_SERVICE_PORT` | Service HTTP port | `3007` |
| `ELASTICSEARCH_URL` | Elasticsearch cluster URL | - |
| `REDIS_URL` | Redis URL | - |
| `RABBITMQ_URL` | RabbitMQ URL | - |
| `INDEX_PREFIX` | Index prefix | `techcorp` |
| `REINDEX_BATCH_SIZE` | Reindex batch size | `1000` |
| `SEARCH_TIMEOUT_MS` | Query timeout | `5000` |

### Index Configuration

```yaml
# config/indices.yaml
indices:
  products:
    shards: 3
    replicas: 1
    analyzer: brazilian
    fields:
      - name: title
        type: text
        boost: 2.0
      - name: description
        type: text
        boost: 1.0
      - name: category
        type: keyword
      - name: price
        type: float
      - name: popularity_score
        type: float
  orders:
    shards: 5
    replicas: 1
    analyzer: standard
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/search-service.git
cd search-service

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Start dependencies
docker-compose up -d elasticsearch redis rabbitmq

# Create indexes
python scripts/create_indices.py

# Start the service
uvicorn main:app --reload --port 3007

# Verify health
curl http://localhost:3007/health
```

### Run Test Search

```bash
# Search products
curl "http://localhost:3007/api/search/products?q=phone&category=electronics&min_price=500&max_price=2000"

# Autocomplete
curl "http://localhost:3007/api/search/products/autocomplete?q=pho"
```

## Search Features

### 1. Full-Text Search

Text search with support for:
- Portuguese stemming (running -> run)
- Stop words (the, a, an, to...)
- Configurable synonyms

### 2. Autocomplete

Real-time suggestions with:
- Elasticsearch completion suggester
- Prefix search
- Spell correction (did you mean?)

### 3. Faceted Filters

```json
{
  "query": "smartphone",
  "filters": {
    "category": ["electronics"],
    "brand": ["samsung", "apple"],
    "price_range": {"min": 1000, "max": 5000}
  },
  "facets": ["category", "brand", "price_range"]
}
```

### 4. Relevance Boosting

Factors that influence the score:
- Exact match in title: 2x
- Product popularity: 1.5x
- Stock availability: 1.2x
- Recency: exponential decay

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/search-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `search_queries_total` | Total searches | - |
| `search_latency_seconds` | Search latency | > 500ms |
| `search_zero_results_ratio` | Zero results rate | > 20% |
| `search_cache_hit_ratio` | Cache hit rate | < 60% |
| `index_lag_seconds` | Indexing delay | > 60s |

### Configured Alerts

- **SearchHighLatency:** P99 latency above 1 second for 5 minutes
- **SearchZeroResultsHigh:** Zero results rate above 20%
- **SearchIndexLagHigh:** Indexing delay above 5 minutes
- **ElasticsearchClusterYellow:** Cluster in yellow state

## Troubleshooting

### Issue: Searches returning few or no results

**Cause:** Incorrect analyzer, missing synonyms, or outdated index.

**Solution:**
1. Test query directly in Elasticsearch: `GET /techcorp-products/_search`
2. Check analyzer: `GET /techcorp-products/_analyze`
3. Add synonyms if necessary: `PUT /techcorp-products/_settings`

### Issue: High search latency

**Cause:** Too complex query, non-optimized index, or overloaded cluster.

**Solution:**
1. Analyze query with `_profile`: `GET /techcorp-products/_search?profile=true`
2. Check cluster resource usage: `GET /_cat/nodes?v`
3. Optimize mapping or add cache

### Issue: Delayed indexing

**Cause:** Accumulated event queue or slow worker.

**Solution:**
1. Check queue backlog: `rabbitmqctl list_queues | grep search`
2. Scale workers: `kubectl scale deployment/search-indexer --replicas=5`
3. Force reindex if necessary: `POST /admin/search/reindex`

### Issue: Slow or incorrect autocomplete

**Cause:** Poorly configured completion suggester or outdated suggestions index.

**Solution:**
1. Check suggestions index: `GET /techcorp-suggestions/_stats`
2. Rebuild completion field: `POST /admin/search/rebuild-suggestions`
3. Increase suggestions cache

## Consumed Events

| Event | Source | Action |
|-------|--------|--------|
| `product.created` | catalog-service | Index new product |
| `product.updated` | catalog-service | Update document |
| `product.deleted` | catalog-service | Remove from index |
| `order.created` | order-service | Index order |
| `inventory.updated` | inventory-service | Update availability |

## Related Links

- [Products API](../apis/products-api.md) - API paths that use search
- [Cache Service](cache-service.md) - Frequent results cache
- [Performance Issues](../troubleshooting/performance-issues.md) - Optimizations
- [System Architecture Overview](../architecture/system-overview.md) - Position in the system
