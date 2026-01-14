# API de Pedidos

## Visão Geral

A API de Pedidos fornece recursos REST para criação, consulta e gerenciamento de pedidos na plataforma TechCorp. Esta API é utilizada pelo portal web, aplicativo mobile e integrações com sistemas parceiros.

## Base URL

```
https://api.techcorp.com/v1/orders
```

## Autenticação

Todos os recursos REST requerem autenticação via Bearer Token:

```
Authorization: Bearer <access_token>
```

## Recursos REST Disponíveis

### GET /

Este recurso REST lista os pedidos do usuário autenticado.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Query Parameters:**

| Nome | Tipo | Descrição | Padrão |
|------|------|-----------|--------|
| page | integer | Número da página | 1 |
| per_page | integer | Itens por página | 20 |
| status | string | Filtrar por status | - |
| start_date | string | Data inicial (YYYY-MM-DD) | - |
| end_date | string | Data final (YYYY-MM-DD) | - |
| sort | string | Ordenação: created_at, total | created_at |
| order | string | Direção: asc, desc | desc |

**Response 200 (Sucesso):**

```json
{
  "data": [
    {
      "id": "uuid",
      "number": "TC-2024-001234",
      "status": "DELIVERED",
      "total": 299.90,
      "items_count": 3,
      "created_at": "2024-01-15T10:30:00Z",
      "delivered_at": "2024-01-20T14:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 45,
    "total_pages": 3
  }
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Não autenticado |

---

### GET /{id}

Este recurso REST retorna os detalhes completos de um pedido.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID ou número do pedido |

**Response 200 (Sucesso):**

```json
{
  "id": "uuid",
  "number": "TC-2024-001234",
  "status": "DELIVERED",
  "payment_status": "PAID",
  "subtotal": 279.90,
  "shipping_cost": 20.00,
  "discount": 0.00,
  "total": 299.90,
  "items": [
    {
      "id": "uuid",
      "product_id": "uuid",
      "product_name": "Camiseta Básica",
      "product_image": "https://cdn.techcorp.com/products/123.jpg",
      "variant": "Azul / M",
      "quantity": 2,
      "unit_price": 89.95,
      "total": 179.90
    },
    {
      "id": "uuid",
      "product_id": "uuid",
      "product_name": "Calça Jeans",
      "product_image": "https://cdn.techcorp.com/products/456.jpg",
      "variant": "40",
      "quantity": 1,
      "unit_price": 100.00,
      "total": 100.00
    }
  ],
  "shipping_address": {
    "recipient_name": "João Silva",
    "street": "Rua das Flores",
    "number": "123",
    "complement": "Apto 45",
    "neighborhood": "Jardim América",
    "city": "São Paulo",
    "state": "SP",
    "postal_code": "01310-100"
  },
  "payment": {
    "method": "credit_card",
    "brand": "Visa",
    "last_four_digits": "1234",
    "installments": 3,
    "installment_value": 99.97
  },
  "tracking": {
    "carrier": "Correios",
    "code": "BR123456789BR",
    "url": "https://rastreamento.correios.com.br/..."
  },
  "timeline": [
    {
      "status": "PENDING",
      "timestamp": "2024-01-15T10:30:00Z",
      "description": "Pedido recebido"
    },
    {
      "status": "PAYMENT_CONFIRMED",
      "timestamp": "2024-01-15T10:35:00Z",
      "description": "Pagamento confirmado"
    },
    {
      "status": "PROCESSING",
      "timestamp": "2024-01-16T08:00:00Z",
      "description": "Pedido em separação"
    },
    {
      "status": "SHIPPED",
      "timestamp": "2024-01-17T16:00:00Z",
      "description": "Pedido enviado"
    },
    {
      "status": "DELIVERED",
      "timestamp": "2024-01-20T14:00:00Z",
      "description": "Pedido entregue"
    }
  ],
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-20T14:00:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Não autenticado |
| 404 | Pedido não encontrado |

---

### POST /

Este recurso REST cria um novo pedido a partir do carrinho.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "cart_id": "uuid",
  "shipping_address_id": "uuid",
  "billing_address_id": "uuid",
  "payment": {
    "method": "credit_card",
    "card_token": "tok_visa_1234",
    "installments": 3
  },
  "coupon_code": "DESCONTO10",
  "notes": "Entregar na portaria"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| cart_id | string | Sim | ID do carrinho |
| shipping_address_id | string | Sim | ID do endereço de entrega |
| billing_address_id | string | Não | ID do endereço de cobrança |
| payment | object | Sim | Dados de pagamento |
| payment.method | string | Sim | credit_card, debit_card, pix, boleto |
| payment.card_token | string | Condicional | Token do cartão (se cartão) |
| payment.installments | integer | Não | Parcelas (1-12) |
| coupon_code | string | Não | Código de cupom |
| notes | string | Não | Observações |

**Response 201 (Sucesso):**

```json
{
  "id": "uuid",
  "number": "TC-2024-001235",
  "status": "PENDING",
  "total": 299.90,
  "payment_url": "https://payment.techcorp.com/checkout/uuid",
  "pix_code": null,
  "boleto_url": null
}
```

**Response 201 (PIX):**

```json
{
  "id": "uuid",
  "number": "TC-2024-001235",
  "status": "PAYMENT_PENDING",
  "total": 299.90,
  "pix_code": "00020126580014br.gov.bcb.pix...",
  "pix_qr_code": "data:image/png;base64,...",
  "pix_expiration": "2024-01-15T11:30:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Dados inválidos ou carrinho vazio |
| 401 | Não autenticado |
| 402 | Pagamento recusado |
| 409 | Produto sem estoque |
| 422 | Cupom inválido ou expirado |

---

### POST /{id}/cancel

Este recurso REST cancela um pedido (antes do envio).

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID do pedido |

**Request:**

```json
{
  "reason": "Desisti da compra"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| reason | string | Não | Motivo do cancelamento |

**Response 200 (Sucesso):**

```json
{
  "message": "Pedido cancelado com sucesso",
  "refund": {
    "amount": 299.90,
    "method": "credit_card",
    "estimated_date": "2024-01-25"
  }
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Pedido já foi enviado (não pode cancelar) |
| 401 | Não autenticado |
| 404 | Pedido não encontrado |

---

### POST /{id}/return

Este recurso REST solicita devolução de um pedido entregue.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID do pedido |

**Request:**

```json
{
  "items": [
    {
      "item_id": "uuid",
      "quantity": 1,
      "reason": "product_defect",
      "description": "Produto veio com defeito na costura"
    }
  ],
  "pickup_address_id": "uuid"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| items | array | Sim | Itens a devolver |
| items[].item_id | string | Sim | ID do item |
| items[].quantity | integer | Sim | Quantidade |
| items[].reason | string | Sim | Motivo: product_defect, wrong_product, changed_mind |
| items[].description | string | Não | Descrição detalhada |
| pickup_address_id | string | Sim | Endereço para coleta |

**Response 201 (Sucesso):**

```json
{
  "return_id": "uuid",
  "status": "PENDING_APPROVAL",
  "message": "Solicitação de devolução registrada. Você receberá a confirmação por e-mail."
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Prazo de devolução expirado (30 dias) |
| 401 | Não autenticado |
| 404 | Pedido ou item não encontrado |

---

### GET /{id}/invoice

Este recurso REST retorna a nota fiscal do pedido.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID do pedido |

**Query Parameters:**

| Nome | Tipo | Descrição | Padrão |
|------|------|-----------|--------|
| format | string | Formato: json, pdf, xml | json |

**Response 200 (JSON):**

```json
{
  "invoice_number": "000123456",
  "invoice_key": "35240112345678000100550010001234561123456789",
  "issue_date": "2024-01-17T10:00:00Z",
  "pdf_url": "https://cdn.techcorp.com/invoices/uuid.pdf",
  "xml_url": "https://cdn.techcorp.com/invoices/uuid.xml"
}
```

**Response 200 (PDF):**

Retorna o arquivo PDF da nota fiscal.

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Não autenticado |
| 404 | Pedido não encontrado ou nota não emitida |

---

### GET /tracking/{code}

Este recurso REST retorna o status de rastreamento de um pedido.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| code | string | Código de rastreamento |

**Response 200 (Sucesso):**

```json
{
  "carrier": "Correios",
  "code": "BR123456789BR",
  "status": "IN_TRANSIT",
  "estimated_delivery": "2024-01-20",
  "events": [
    {
      "timestamp": "2024-01-19T08:00:00Z",
      "status": "IN_TRANSIT",
      "location": "São Paulo, SP",
      "description": "Objeto saiu para entrega ao destinatário"
    },
    {
      "timestamp": "2024-01-18T16:00:00Z",
      "status": "IN_TRANSIT",
      "location": "Curitiba, PR",
      "description": "Objeto em trânsito - por favor aguarde"
    },
    {
      "timestamp": "2024-01-17T16:00:00Z",
      "status": "POSTED",
      "location": "São Paulo, SP",
      "description": "Objeto postado"
    }
  ]
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Não autenticado |
| 404 | Código de rastreamento não encontrado |

## Status do Pedido

| Status | Descrição |
|--------|-----------|
| `PENDING` | Pedido criado, aguardando pagamento |
| `PAYMENT_PENDING` | Aguardando confirmação do pagamento |
| `PAYMENT_CONFIRMED` | Pagamento confirmado |
| `PROCESSING` | Pedido em separação |
| `SHIPPED` | Pedido enviado |
| `IN_TRANSIT` | Em trânsito para entrega |
| `OUT_FOR_DELIVERY` | Saiu para entrega |
| `DELIVERED` | Entregue |
| `CANCELLED` | Cancelado |
| `RETURN_REQUESTED` | Devolução solicitada |
| `RETURNED` | Devolvido |
| `REFUNDED` | Reembolsado |

## Webhooks

Para integrações, é possível receber notificações de mudança de status via webhook:

```json
{
  "event": "order.status_changed",
  "order_id": "uuid",
  "order_number": "TC-2024-001234",
  "previous_status": "SHIPPED",
  "new_status": "DELIVERED",
  "timestamp": "2024-01-20T14:00:00Z"
}
```

## Links Relacionados

- [Order Service](../components/order-service.md) - Documentação do serviço
- [Payments API](payments-api.md) - Operações de pagamento
- [Products API](products-api.md) - Caminhos da API de produtos
- [Fluxo de Dados](../architecture/data-flow.md) - Fluxo do pedido
