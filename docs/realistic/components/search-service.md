# Search Service

## Descrição

O Search Service é o microsserviço de busca da TechCorp, responsável por fornecer capacidades de pesquisa full-text, filtros avançados e relevância personalizada para produtos, pedidos e conteúdo da plataforma.

O serviço encapsula a complexidade do Elasticsearch, oferecendo uma API simplificada para os consumidores. Ele implementa funcionalidades como autocomplete, correção ortográfica, busca por sinônimos, boosting por popularidade e filtros facetados.

A indexação é realizada em near real-time através de eventos consumidos do message broker. O search-service mantém índices otimizados para diferentes casos de uso (produtos, pedidos, usuários) com mapeamentos e analyzers específicos para português brasileiro.

## Responsáveis

- **Time:** Platform Engineering
- **Tech Lead:** Lucas Ferreira
- **Slack:** #platform-search

## Stack Tecnológica

- Linguagem: Python 3.11
- Framework: FastAPI 0.104
- Search Engine: Elasticsearch 8.11
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `SEARCH_SERVICE_PORT` | Porta HTTP do serviço | `3007` |
| `ELASTICSEARCH_URL` | URL do cluster Elasticsearch | - |
| `REDIS_URL` | URL do Redis | - |
| `RABBITMQ_URL` | URL do RabbitMQ | - |
| `INDEX_PREFIX` | Prefixo dos índices | `techcorp` |
| `REINDEX_BATCH_SIZE` | Tamanho do batch de reindexação | `1000` |
| `SEARCH_TIMEOUT_MS` | Timeout das queries | `5000` |

### Configuração de Índices

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

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/search-service.git
cd search-service

# Criar ambiente virtual
python -m venv venv
source venv/bin/activate

# Instalar dependências
pip install -r requirements.txt

# Subir dependências
docker-compose up -d elasticsearch redis rabbitmq

# Criar índices
python scripts/create_indices.py

# Iniciar o serviço
uvicorn main:app --reload --port 3007

# Verificar saúde
curl http://localhost:3007/health
```

### Executar Busca de Teste

```bash
# Buscar produtos
curl "http://localhost:3007/api/search/products?q=celular&category=eletronicos&min_price=500&max_price=2000"

# Autocomplete
curl "http://localhost:3007/api/search/products/autocomplete?q=celu"
```

## Funcionalidades de Busca

### 1. Full-Text Search

Busca textual com suporte a:
- Stemming em português (correndo -> correr)
- Stop words (de, a, o, para...)
- Sinônimos configuráveis

### 2. Autocomplete

Sugestões em tempo real com:
- Completion suggester do Elasticsearch
- Busca por prefixo
- Correção ortográfica (did you mean?)

### 3. Filtros Facetados

```json
{
  "query": "smartphone",
  "filters": {
    "category": ["eletronicos"],
    "brand": ["samsung", "apple"],
    "price_range": {"min": 1000, "max": 5000}
  },
  "facets": ["category", "brand", "price_range"]
}
```

### 4. Boosting de Relevância

Fatores que influenciam o score:
- Match exato no título: 2x
- Popularidade do produto: 1.5x
- Disponibilidade em estoque: 1.2x
- Recência: decay exponencial

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/search-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `search_queries_total` | Total de buscas | - |
| `search_latency_seconds` | Latência das buscas | > 500ms |
| `search_zero_results_ratio` | Taxa de buscas sem resultado | > 20% |
| `search_cache_hit_ratio` | Taxa de acerto do cache | < 60% |
| `index_lag_seconds` | Atraso na indexação | > 60s |

### Alertas Configurados

- **SearchHighLatency:** Latência P99 acima de 1 segundo por 5 minutos
- **SearchZeroResultsHigh:** Taxa de buscas sem resultado acima de 20%
- **SearchIndexLagHigh:** Atraso de indexação acima de 5 minutos
- **ElasticsearchClusterYellow:** Cluster em estado yellow

## Troubleshooting

### Problema: Buscas retornando poucos ou nenhum resultado

**Causa:** Analyzer incorreto, sinônimos faltando ou índice desatualizado.

**Solução:**
1. Testar query diretamente no Elasticsearch: `GET /techcorp-products/_search`
2. Verificar analyzer: `GET /techcorp-products/_analyze`
3. Adicionar sinônimos se necessário: `PUT /techcorp-products/_settings`

### Problema: Latência de busca alta

**Causa:** Query muito complexa, índice não otimizado ou cluster sobrecarregado.

**Solução:**
1. Analisar query com `_profile`: `GET /techcorp-products/_search?profile=true`
2. Verificar uso de recursos do cluster: `GET /_cat/nodes?v`
3. Otimizar mapeamento ou adicionar cache

### Problema: Indexação atrasada

**Causa:** Fila de eventos acumulada ou worker lento.

**Solução:**
1. Verificar backlog da fila: `rabbitmqctl list_queues | grep search`
2. Escalar workers: `kubectl scale deployment/search-indexer --replicas=5`
3. Forçar reindexação se necessário: `POST /admin/search/reindex`

### Problema: Autocomplete lento ou incorreto

**Causa:** Completion suggester mal configurado ou índice de sugestões desatualizado.

**Solução:**
1. Verificar índice de sugestões: `GET /techcorp-suggestions/_stats`
2. Recriar completion field: `POST /admin/search/rebuild-suggestions`
3. Aumentar cache de sugestões

## Eventos Consumidos

| Evento | Origem | Ação |
|--------|--------|------|
| `product.created` | catalog-service | Indexar novo produto |
| `product.updated` | catalog-service | Atualizar documento |
| `product.deleted` | catalog-service | Remover do índice |
| `order.created` | order-service | Indexar pedido |
| `inventory.updated` | inventory-service | Atualizar disponibilidade |

## Links Relacionados

- [API de Produtos](../apis/products-api.md) - Caminhos da API que usam busca
- [Cache Service](cache-service.md) - Cache de resultados frequentes
- [Problemas de Performance](../troubleshooting/performance-issues.md) - Otimizações
- [Visão Geral da Arquitetura](../architecture/system-overview.md) - Posição no sistema
