# API de Pagamentos

## Visão Geral

A API de Pagamentos fornece operações para processamento de transações financeiras, consulta de pagamentos e gerenciamento de métodos de pagamento salvos. Esta API integra-se com múltiplos provedores de pagamento de forma transparente.

## Base URL

```
https://api.techcorp.com/v1/payments
```

## Autenticação

Todas as operações requerem autenticação via Bearer Token:

```
Authorization: Bearer <access_token>
```

## Operações Disponíveis

### POST /process

Esta operação processa um novo pagamento.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request (Cartão de Crédito):**

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "BRL",
  "method": "credit_card",
  "card": {
    "token": "tok_visa_4242",
    "installments": 3,
    "save_card": true
  },
  "billing_address": {
    "postal_code": "01310-100",
    "number": "123"
  }
}
```

**Request (PIX):**

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "BRL",
  "method": "pix",
  "pix": {
    "expiration_minutes": 30
  }
}
```

**Request (Boleto):**

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "BRL",
  "method": "boleto",
  "boleto": {
    "due_days": 3,
    "instructions": "Não receber após vencimento"
  }
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| order_id | string | Sim | ID do pedido |
| amount | integer | Sim | Valor em centavos |
| currency | string | Sim | Moeda (BRL) |
| method | string | Sim | credit_card, debit_card, pix, boleto |
| card.token | string | Condicional | Token do cartão (se cartão) |
| card.installments | integer | Não | Parcelas (1-12) |
| card.save_card | boolean | Não | Salvar cartão para compras futuras |
| pix.expiration_minutes | integer | Não | Expiração do PIX (default: 30) |
| boleto.due_days | integer | Não | Dias para vencimento (default: 3) |

**Response 200 (Cartão Aprovado):**

```json
{
  "transaction_id": "uuid",
  "status": "APPROVED",
  "amount": 29990,
  "installments": 3,
  "installment_amount": 9997,
  "authorization_code": "123456",
  "card": {
    "brand": "Visa",
    "last_four": "4242",
    "saved_card_id": "uuid"
  },
  "processed_at": "2024-01-15T10:30:00Z"
}
```

**Response 200 (PIX Gerado):**

```json
{
  "transaction_id": "uuid",
  "status": "PENDING",
  "amount": 29990,
  "pix": {
    "code": "00020126580014br.gov.bcb.pix0136...",
    "qr_code_base64": "data:image/png;base64,...",
    "expires_at": "2024-01-15T11:00:00Z"
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Response 200 (Boleto Gerado):**

```json
{
  "transaction_id": "uuid",
  "status": "PENDING",
  "amount": 29990,
  "boleto": {
    "barcode": "23793.38128 60000.000003 00000.000400 1 84340000029990",
    "digitable_line": "23793381286000000000300000004001843400000299.90",
    "pdf_url": "https://cdn.techcorp.com/boletos/uuid.pdf",
    "due_date": "2024-01-18"
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Dados inválidos |
| 401 | Não autenticado |
| 402 | Pagamento recusado |
| 409 | Pedido já pago |
| 422 | Cartão inválido ou expirado |

---

### GET /{id}

Esta operação retorna os detalhes de uma transação.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID da transação |

**Response 200 (Sucesso):**

```json
{
  "transaction_id": "uuid",
  "order_id": "uuid",
  "status": "APPROVED",
  "amount": 29990,
  "currency": "BRL",
  "method": "credit_card",
  "installments": 3,
  "installment_amount": 9997,
  "card": {
    "brand": "Visa",
    "last_four": "4242"
  },
  "events": [
    {
      "type": "AUTHORIZED",
      "timestamp": "2024-01-15T10:30:00Z"
    },
    {
      "type": "CAPTURED",
      "timestamp": "2024-01-15T10:30:05Z"
    }
  ],
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Não autenticado |
| 404 | Transação não encontrada |

---

### POST /{id}/refund

Esta operação processa estorno total ou parcial.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID da transação |

**Request:**

```json
{
  "amount": 10000,
  "reason": "customer_request",
  "description": "Cliente solicitou cancelamento parcial"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| amount | integer | Não | Valor em centavos (se parcial) |
| reason | string | Sim | customer_request, product_issue, fraud |
| description | string | Não | Descrição do motivo |

**Response 200 (Sucesso):**

```json
{
  "refund_id": "uuid",
  "transaction_id": "uuid",
  "status": "PROCESSED",
  "amount": 10000,
  "method": "original_payment",
  "estimated_arrival": "2024-01-25",
  "processed_at": "2024-01-15T14:00:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Valor maior que o disponível |
| 401 | Não autenticado |
| 404 | Transação não encontrada |
| 422 | Transação não permite estorno |

---

### GET /methods

Esta operação lista os métodos de pagamento disponíveis.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Query Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| amount | integer | Valor para calcular parcelas |

**Response 200 (Sucesso):**

```json
{
  "credit_card": {
    "enabled": true,
    "brands": ["visa", "mastercard", "amex", "elo", "hipercard"],
    "installments": [
      {"quantity": 1, "amount": 29990, "interest_free": true},
      {"quantity": 2, "amount": 14995, "interest_free": true},
      {"quantity": 3, "amount": 9997, "interest_free": true},
      {"quantity": 6, "amount": 5248, "interest_free": false, "total": 31488},
      {"quantity": 12, "amount": 2749, "interest_free": false, "total": 32988}
    ],
    "saved_cards": [
      {
        "id": "uuid",
        "brand": "Visa",
        "last_four": "4242",
        "holder_name": "JOAO SILVA",
        "expiry_month": 12,
        "expiry_year": 2025,
        "is_default": true
      }
    ]
  },
  "debit_card": {
    "enabled": true,
    "brands": ["visa", "mastercard", "elo"]
  },
  "pix": {
    "enabled": true,
    "discount_percentage": 5
  },
  "boleto": {
    "enabled": true,
    "due_days_options": [1, 3, 5]
  }
}
```

---

### POST /cards

Esta operação salva um novo cartão para compras futuras.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "token": "tok_visa_4242",
  "set_default": true
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| token | string | Sim | Token do cartão (gerado pelo frontend) |
| set_default | boolean | Não | Definir como padrão |

**Response 201 (Sucesso):**

```json
{
  "card_id": "uuid",
  "brand": "Visa",
  "last_four": "4242",
  "holder_name": "JOAO SILVA",
  "expiry_month": 12,
  "expiry_year": 2025,
  "is_default": true
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Token inválido |
| 401 | Não autenticado |
| 409 | Cartão já cadastrado |
| 422 | Cartão expirado ou inválido |

---

### DELETE /cards/{id}

Esta operação remove um cartão salvo.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID do cartão |

**Response 200 (Sucesso):**

```json
{
  "message": "Cartão removido com sucesso"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Não autenticado |
| 404 | Cartão não encontrado |

---

### PUT /cards/{id}/default

Esta operação define um cartão como padrão.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Sucesso):**

```json
{
  "message": "Cartão definido como padrão"
}
```

---

### POST /validate-card

Esta operação valida um cartão sem cobrar (verificação).

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "token": "tok_visa_4242"
}
```

**Response 200 (Sucesso):**

```json
{
  "valid": true,
  "brand": "Visa",
  "last_four": "4242",
  "country": "BR",
  "type": "credit"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 422 | Cartão inválido |

## Status de Transação

| Status | Descrição |
|--------|-----------|
| `PENDING` | Aguardando pagamento (PIX/Boleto) |
| `AUTHORIZED` | Autorizado, não capturado |
| `APPROVED` | Aprovado e capturado |
| `DECLINED` | Recusado pelo emissor |
| `CANCELLED` | Cancelado antes da captura |
| `REFUNDED` | Estornado totalmente |
| `PARTIALLY_REFUNDED` | Estornado parcialmente |
| `EXPIRED` | PIX/Boleto expirado |
| `CHARGEBACK` | Contestação do titular |

## Códigos de Recusa

| Código | Descrição | Ação Sugerida |
|--------|-----------|---------------|
| `insufficient_funds` | Saldo insuficiente | Tentar outro cartão |
| `card_declined` | Cartão recusado | Contatar banco |
| `expired_card` | Cartão expirado | Usar cartão válido |
| `invalid_cvv` | CVV incorreto | Verificar código |
| `fraud_suspected` | Suspeita de fraude | Contatar banco |
| `processing_error` | Erro de processamento | Tentar novamente |

## Segurança

- Todos os dados de cartão são tokenizados no frontend
- A API nunca recebe ou armazena números completos de cartão
- Transações são processadas em ambiente PCI-DSS
- Tokens de cartão expiram em 15 minutos

## Links Relacionados

- [Payment Service](../components/payment-service.md) - Documentação do serviço
- [Orders API](orders-api.md) - Recursos REST de pedidos
- [Modelo de Segurança](../architecture/security-model.md) - Segurança de pagamentos
- [Erros Comuns](../troubleshooting/common-errors.md) - Problemas de pagamento
