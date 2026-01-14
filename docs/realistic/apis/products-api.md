# API de Produtos

## Visão Geral

A API de Produtos fornece caminhos da API para consulta do catálogo de produtos, categorias, busca e informações de disponibilidade. Esta API é pública para listagem e busca, mas requer autenticação para funcionalidades como favoritos e avaliações.

## Base URL

```
https://api.techcorp.com/v1/products
```

## Autenticação

Caminhos públicos não requerem autenticação. Caminhos que modificam dados requerem:

```
Authorization: Bearer <access_token>
```

## Caminhos da API Disponíveis

### GET /

Este caminho da API lista produtos do catálogo com filtros e paginação.

**Query Parameters:**

| Nome | Tipo | Descrição | Padrão |
|------|------|-----------|--------|
| page | integer | Número da página | 1 |
| per_page | integer | Itens por página (max 100) | 20 |
| category | string | Slug da categoria | - |
| brand | string | Slug da marca | - |
| min_price | number | Preço mínimo | - |
| max_price | number | Preço máximo | - |
| in_stock | boolean | Apenas com estoque | false |
| sort | string | Ordenação: price, name, popularity, newest | popularity |
| order | string | Direção: asc, desc | desc |
| attributes | string | Filtros de atributos (ver abaixo) | - |

**Filtros por Atributos:**

```
attributes=cor:azul,vermelho;tamanho:m,g
```

**Response 200 (Sucesso):**

```json
{
  "data": [
    {
      "id": "uuid",
      "slug": "camiseta-basica-algodao",
      "name": "Camiseta Básica Algodão",
      "short_description": "Camiseta 100% algodão, confortável para o dia a dia",
      "price": 89.90,
      "original_price": 119.90,
      "discount_percentage": 25,
      "image": "https://cdn.techcorp.com/products/123/main.jpg",
      "brand": {
        "slug": "techcorp-basics",
        "name": "TechCorp Basics"
      },
      "category": {
        "slug": "camisetas",
        "name": "Camisetas"
      },
      "rating": 4.5,
      "reviews_count": 127,
      "in_stock": true,
      "variants_count": 12
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 1523,
    "total_pages": 77
  },
  "facets": {
    "categories": [
      {"slug": "camisetas", "name": "Camisetas", "count": 450},
      {"slug": "calcas", "name": "Calças", "count": 320}
    ],
    "brands": [
      {"slug": "techcorp-basics", "name": "TechCorp Basics", "count": 200}
    ],
    "price_ranges": [
      {"min": 0, "max": 50, "count": 150},
      {"min": 50, "max": 100, "count": 400},
      {"min": 100, "max": 200, "count": 300}
    ],
    "attributes": {
      "cor": [
        {"value": "azul", "count": 120},
        {"value": "vermelho", "count": 80}
      ],
      "tamanho": [
        {"value": "p", "count": 200},
        {"value": "m", "count": 250}
      ]
    }
  }
}
```

---

### GET /{slug}

Este caminho da API retorna os detalhes completos de um produto.

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| slug | string | Slug do produto |

**Response 200 (Sucesso):**

```json
{
  "id": "uuid",
  "slug": "camiseta-basica-algodao",
  "sku": "CAM-BAS-001",
  "name": "Camiseta Básica Algodão",
  "description": "Camiseta confeccionada em 100% algodão premium...",
  "short_description": "Camiseta 100% algodão, confortável para o dia a dia",
  "price": 89.90,
  "original_price": 119.90,
  "discount_percentage": 25,
  "brand": {
    "slug": "techcorp-basics",
    "name": "TechCorp Basics",
    "logo": "https://cdn.techcorp.com/brands/techcorp-basics.png"
  },
  "category": {
    "slug": "camisetas",
    "name": "Camisetas",
    "breadcrumb": [
      {"slug": "vestuario", "name": "Vestuário"},
      {"slug": "masculino", "name": "Masculino"},
      {"slug": "camisetas", "name": "Camisetas"}
    ]
  },
  "images": [
    {
      "url": "https://cdn.techcorp.com/products/123/main.jpg",
      "alt": "Camiseta Básica Algodão - Frente",
      "is_main": true
    },
    {
      "url": "https://cdn.techcorp.com/products/123/back.jpg",
      "alt": "Camiseta Básica Algodão - Costas",
      "is_main": false
    }
  ],
  "variants": [
    {
      "id": "uuid",
      "sku": "CAM-BAS-001-AZU-M",
      "attributes": {
        "cor": "Azul",
        "tamanho": "M"
      },
      "price": 89.90,
      "in_stock": true,
      "stock_quantity": 15
    },
    {
      "id": "uuid",
      "sku": "CAM-BAS-001-AZU-G",
      "attributes": {
        "cor": "Azul",
        "tamanho": "G"
      },
      "price": 89.90,
      "in_stock": false,
      "stock_quantity": 0
    }
  ],
  "attributes": {
    "material": "100% Algodão",
    "origem": "Brasil",
    "lavagem": "Máquina água fria"
  },
  "rating": 4.5,
  "reviews_count": 127,
  "questions_count": 23,
  "related_products": ["uuid1", "uuid2", "uuid3"],
  "created_at": "2024-01-01T10:00:00Z",
  "updated_at": "2024-01-15T14:30:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 404 | Produto não encontrado |

---

### GET /search

Este caminho da API realiza busca full-text no catálogo.

**Query Parameters:**

| Nome | Tipo | Descrição | Padrão |
|------|------|-----------|--------|
| q | string | Termo de busca (obrigatório) | - |
| page | integer | Número da página | 1 |
| per_page | integer | Itens por página | 20 |
| filters | string | Filtros adicionais | - |

**Response 200 (Sucesso):**

```json
{
  "query": "camiseta azul",
  "data": [...],
  "pagination": {...},
  "facets": {...},
  "suggestions": {
    "did_you_mean": null,
    "related_searches": ["camiseta azul marinho", "camiseta azul claro"]
  }
}
```

---

### GET /search/autocomplete

Este caminho da API retorna sugestões de busca em tempo real.

**Query Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| q | string | Termo parcial (mínimo 2 caracteres) |

**Response 200 (Sucesso):**

```json
{
  "suggestions": [
    {
      "type": "product",
      "text": "Camiseta Básica Algodão",
      "slug": "camiseta-basica-algodao",
      "image": "https://cdn.techcorp.com/products/123/thumb.jpg"
    },
    {
      "type": "category",
      "text": "Camisetas",
      "slug": "camisetas"
    },
    {
      "type": "brand",
      "text": "Camisa & Cia",
      "slug": "camisa-cia"
    }
  ]
}
```

---

### GET /categories

Este caminho da API lista todas as categorias.

**Query Parameters:**

| Nome | Tipo | Descrição | Padrão |
|------|------|-----------|--------|
| parent | string | Slug da categoria pai | - |
| depth | integer | Profundidade da árvore | 3 |

**Response 200 (Sucesso):**

```json
{
  "data": [
    {
      "slug": "vestuario",
      "name": "Vestuário",
      "image": "https://cdn.techcorp.com/categories/vestuario.jpg",
      "products_count": 2500,
      "children": [
        {
          "slug": "masculino",
          "name": "Masculino",
          "products_count": 1200,
          "children": [
            {"slug": "camisetas", "name": "Camisetas", "products_count": 450},
            {"slug": "calcas", "name": "Calças", "products_count": 320}
          ]
        }
      ]
    }
  ]
}
```

---

### GET /{slug}/reviews

Este caminho da API lista avaliações de um produto.

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| slug | string | Slug do produto |

**Query Parameters:**

| Nome | Tipo | Descrição | Padrão |
|------|------|-----------|--------|
| page | integer | Número da página | 1 |
| per_page | integer | Itens por página | 10 |
| rating | integer | Filtrar por nota (1-5) | - |
| sort | string | Ordenação: newest, helpful | newest |

**Response 200 (Sucesso):**

```json
{
  "summary": {
    "average_rating": 4.5,
    "total_reviews": 127,
    "distribution": {
      "5": 80,
      "4": 30,
      "3": 10,
      "2": 5,
      "1": 2
    }
  },
  "data": [
    {
      "id": "uuid",
      "rating": 5,
      "title": "Excelente qualidade!",
      "content": "Comprei para usar no dia a dia e superou expectativas...",
      "author": {
        "name": "João S.",
        "verified_purchase": true
      },
      "variant": "Azul / M",
      "helpful_count": 15,
      "images": [
        "https://cdn.techcorp.com/reviews/uuid/1.jpg"
      ],
      "created_at": "2024-01-10T10:00:00Z"
    }
  ],
  "pagination": {...}
}
```

---

### POST /{slug}/reviews

Este caminho da API cria uma avaliação para um produto comprado.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| slug | string | Slug do produto |

**Request:**

```json
{
  "order_item_id": "uuid",
  "rating": 5,
  "title": "Excelente qualidade!",
  "content": "Comprei para usar no dia a dia e superou expectativas...",
  "images": ["base64..."]
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| order_item_id | string | Sim | ID do item do pedido |
| rating | integer | Sim | Nota de 1 a 5 |
| title | string | Sim | Título da avaliação |
| content | string | Sim | Texto da avaliação |
| images | array | Não | Imagens em base64 (máx 5) |

**Response 201 (Sucesso):**

```json
{
  "message": "Avaliação enviada para moderação",
  "review_id": "uuid"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Dados inválidos |
| 401 | Não autenticado |
| 403 | Não comprou este produto |
| 409 | Já avaliou este produto |

---

### GET /{slug}/availability

Este caminho da API verifica disponibilidade e prazo de entrega.

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| slug | string | Slug do produto |

**Query Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| variant_id | string | ID da variante |
| postal_code | string | CEP para cálculo de frete |
| quantity | integer | Quantidade desejada |

**Response 200 (Sucesso):**

```json
{
  "available": true,
  "stock_quantity": 15,
  "shipping_options": [
    {
      "carrier": "Correios PAC",
      "price": 15.90,
      "estimated_days": 7,
      "delivery_date": "2024-01-25"
    },
    {
      "carrier": "Correios SEDEX",
      "price": 29.90,
      "estimated_days": 3,
      "delivery_date": "2024-01-21"
    }
  ]
}
```

---

### POST /favorites/{slug}

Este caminho da API adiciona um produto aos favoritos.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 201 (Sucesso):**

```json
{
  "message": "Produto adicionado aos favoritos"
}
```

---

### DELETE /favorites/{slug}

Este caminho da API remove um produto dos favoritos.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Sucesso):**

```json
{
  "message": "Produto removido dos favoritos"
}
```

---

### GET /favorites

Este caminho da API lista os produtos favoritos do usuário.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Sucesso):**

```json
{
  "data": [
    {
      "product": {...},
      "added_at": "2024-01-10T10:00:00Z"
    }
  ],
  "pagination": {...}
}
```

## Links Relacionados

- [Search Service](../components/search-service.md) - Motor de busca
- [Inventory Service](../components/inventory-service.md) - Controle de estoque
- [Orders API](orders-api.md) - Recursos REST de pedidos
