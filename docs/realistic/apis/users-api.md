# API de Usuários

## Visão Geral

A API de Usuários fornece rotas para gerenciamento de dados cadastrais, preferências e endereços dos usuários na plataforma TechCorp. Esta API é utilizada tanto pelo portal web quanto pelos aplicativos mobile.

## Base URL

```
https://api.techcorp.com/v1/users
```

## Autenticação

Todas as rotas requerem autenticação via Bearer Token:

```
Authorization: Bearer <access_token>
```

## Rotas Disponíveis

### GET /profile

Esta rota retorna o perfil completo do usuário autenticado.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Sucesso):**

```json
{
  "id": "uuid",
  "email": "usuario@exemplo.com",
  "name": "João Silva",
  "phone": "+5511999999999",
  "document_number": "12345678900",
  "birth_date": "1990-05-15",
  "avatar_url": "https://cdn.techcorp.com/avatars/uuid.jpg",
  "status": "ACTIVE",
  "organization": {
    "id": "uuid",
    "name": "Empresa ABC"
  },
  "created_at": "2024-01-01T10:00:00Z",
  "updated_at": "2024-01-15T14:30:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Não autenticado |

---

### PUT /profile

Esta rota atualiza os dados cadastrais do usuário.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "name": "João Silva Santos",
  "phone": "+5511999998888",
  "birth_date": "1990-05-15"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| name | string | Não | Nome completo |
| phone | string | Não | Telefone com DDI |
| birth_date | string | Não | Data no formato YYYY-MM-DD |

**Response 200 (Sucesso):**

```json
{
  "message": "Perfil atualizado com sucesso",
  "user": {
    "id": "uuid",
    "name": "João Silva Santos",
    "phone": "+5511999998888"
  }
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Dados inválidos |
| 401 | Não autenticado |

---

### PUT /email

Esta rota inicia o processo de alteração de e-mail.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "new_email": "novoemail@exemplo.com",
  "password": "senhaAtual123"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| new_email | string | Sim | Novo e-mail |
| password | string | Sim | Senha atual para confirmação |

**Response 200 (Sucesso):**

```json
{
  "message": "E-mail de confirmação enviado para o novo endereço"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | E-mail inválido |
| 401 | Senha incorreta |
| 409 | E-mail já em uso |

---

### PUT /password

Esta rota altera a senha do usuário.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "current_password": "senhaAtual123",
  "new_password": "NovaSenha@456",
  "new_password_confirmation": "NovaSenha@456"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| current_password | string | Sim | Senha atual |
| new_password | string | Sim | Nova senha (mínimo 8 caracteres) |
| new_password_confirmation | string | Sim | Confirmação da nova senha |

**Response 200 (Sucesso):**

```json
{
  "message": "Senha alterada com sucesso"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Senha fraca ou não confere |
| 401 | Senha atual incorreta |

---

### GET /preferences

Esta rota retorna as preferências do usuário.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Sucesso):**

```json
{
  "language": "pt-BR",
  "timezone": "America/Sao_Paulo",
  "currency": "BRL",
  "theme": "light",
  "notifications": {
    "email": true,
    "sms": false,
    "push": true,
    "marketing": false
  }
}
```

---

### PUT /preferences

Esta rota atualiza as preferências do usuário.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "language": "en-US",
  "theme": "dark",
  "notifications": {
    "email": true,
    "sms": true,
    "push": true,
    "marketing": false
  }
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| language | string | Não | Idioma: pt-BR, en-US, es-ES |
| timezone | string | Não | Timezone IANA |
| currency | string | Não | Moeda: BRL, USD |
| theme | string | Não | Tema: light, dark, system |
| notifications | object | Não | Preferências de notificação |

**Response 200 (Sucesso):**

```json
{
  "message": "Preferências atualizadas com sucesso"
}
```

---

### GET /addresses

Esta rota lista os endereços cadastrados do usuário.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Query Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| type | string | Filtrar por tipo: shipping, billing |

**Response 200 (Sucesso):**

```json
{
  "data": [
    {
      "id": "uuid",
      "label": "Casa",
      "type": "shipping",
      "is_default": true,
      "recipient_name": "João Silva",
      "street": "Rua das Flores",
      "number": "123",
      "complement": "Apto 45",
      "neighborhood": "Jardim América",
      "city": "São Paulo",
      "state": "SP",
      "postal_code": "01310-100",
      "country": "BR",
      "phone": "+5511999999999"
    }
  ],
  "total": 1
}
```

---

### POST /addresses

Esta rota cadastra um novo endereço.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "label": "Trabalho",
  "type": "shipping",
  "is_default": false,
  "recipient_name": "João Silva",
  "street": "Av. Paulista",
  "number": "1000",
  "complement": "10º andar",
  "neighborhood": "Bela Vista",
  "city": "São Paulo",
  "state": "SP",
  "postal_code": "01310-100",
  "country": "BR",
  "phone": "+5511988887777"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| label | string | Sim | Identificador do endereço |
| type | string | Sim | Tipo: shipping ou billing |
| is_default | boolean | Não | Se é o endereço padrão |
| recipient_name | string | Sim | Nome do destinatário |
| street | string | Sim | Logradouro |
| number | string | Sim | Número |
| complement | string | Não | Complemento |
| neighborhood | string | Sim | Bairro |
| city | string | Sim | Cidade |
| state | string | Sim | Estado (UF) |
| postal_code | string | Sim | CEP |
| country | string | Não | País (default: BR) |
| phone | string | Não | Telefone de contato |

**Response 201 (Sucesso):**

```json
{
  "message": "Endereço cadastrado com sucesso",
  "address": {
    "id": "uuid",
    "label": "Trabalho"
  }
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Dados inválidos |
| 401 | Não autenticado |

---

### PUT /addresses/{id}

Esta rota atualiza um endereço existente.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID do endereço |

**Request:**

```json
{
  "label": "Trabalho Novo",
  "complement": "12º andar"
}
```

**Response 200 (Sucesso):**

```json
{
  "message": "Endereço atualizado com sucesso"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Dados inválidos |
| 404 | Endereço não encontrado |

---

### DELETE /addresses/{id}

Esta rota remove um endereço.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Nome | Tipo | Descrição |
|------|------|-----------|
| id | string | ID do endereço |

**Response 200 (Sucesso):**

```json
{
  "message": "Endereço removido com sucesso"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Não é possível remover endereço padrão |
| 404 | Endereço não encontrado |

---

### POST /avatar

Esta rota faz upload do avatar do usuário.

**Headers:**

```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

**Request:**

```
file: <arquivo de imagem>
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| file | file | Sim | Imagem JPG ou PNG, máximo 5MB |

**Response 200 (Sucesso):**

```json
{
  "avatar_url": "https://cdn.techcorp.com/avatars/uuid.jpg"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Arquivo inválido ou muito grande |

---

### DELETE /account

Esta rota solicita a exclusão da conta (LGPD).

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "password": "senhaAtual123",
  "reason": "Não uso mais o serviço"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| password | string | Sim | Senha para confirmação |
| reason | string | Não | Motivo da exclusão |

**Response 200 (Sucesso):**

```json
{
  "message": "Solicitação de exclusão registrada. Sua conta será removida em 30 dias."
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Há pedidos em andamento |
| 401 | Senha incorreta |

## Paginação

As rotas de listagem suportam paginação:

| Parâmetro | Tipo | Descrição | Padrão |
|-----------|------|-----------|--------|
| page | integer | Número da página | 1 |
| per_page | integer | Itens por página | 20 |

## Links Relacionados

- [User Service](../components/user-service.md) - Documentação do serviço
- [Auth API](auth-api.md) - Endpoints de autenticação
- [User Management](../components/user-management.md) - Tela administrativa
