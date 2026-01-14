# API de Autenticação

## Visão Geral

A API de Autenticação fornece endpoints para gerenciamento de identidade e sessões na plataforma TechCorp. Esta API implementa os padrões OAuth 2.0 e OpenID Connect para autenticação segura.

## Base URL

```
https://api.techcorp.com/v1/auth
```

## Autenticação

A maioria dos endpoints requer autenticação via Bearer Token no header:

```
Authorization: Bearer <access_token>
```

Alguns endpoints são públicos (login, registro) e não requerem token.

## Endpoints Disponíveis

### POST /login

Este endpoint realiza a autenticação do usuário com credenciais (e-mail e senha).

**Request:**

```json
{
  "email": "usuario@exemplo.com",
  "password": "senha123",
  "device_info": {
    "device_id": "uuid",
    "device_type": "web",
    "user_agent": "Mozilla/5.0..."
  }
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| email | string | Sim | E-mail do usuário |
| password | string | Sim | Senha do usuário |
| device_info | object | Não | Informações do dispositivo |

**Response 200 (Sucesso):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 1800,
  "user": {
    "id": "uuid",
    "email": "usuario@exemplo.com",
    "name": "João Silva"
  }
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Requisição inválida (campos faltando) |
| 401 | Credenciais inválidas |
| 403 | Conta bloqueada |
| 429 | Muitas tentativas, aguarde |

---

### POST /logout

Este endpoint encerra a sessão do usuário, invalidando o token de acesso.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "logout_all_devices": false
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| refresh_token | string | Sim | Refresh token para invalidar |
| logout_all_devices | boolean | Não | Se true, invalida todas as sessões |

**Response 200 (Sucesso):**

```json
{
  "message": "Logout realizado com sucesso"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Token inválido ou expirado |

---

### POST /refresh

Este endpoint renova o access token usando um refresh token válido.

**Request:**

```json
{
  "refresh_token": "eyJhbGciOiJSUzI1NiIs..."
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| refresh_token | string | Sim | Refresh token válido |

**Response 200 (Sucesso):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 1800
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Refresh token inválido ou expirado |

---

### POST /register

Este endpoint cria uma nova conta de usuário na plataforma.

**Request:**

```json
{
  "email": "novo@exemplo.com",
  "password": "SenhaForte@123",
  "password_confirmation": "SenhaForte@123",
  "name": "Maria Santos",
  "phone": "+5511999999999",
  "document_number": "12345678900",
  "birth_date": "1990-05-15",
  "accept_terms": true
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| email | string | Sim | E-mail único |
| password | string | Sim | Mínimo 8 caracteres, com letras e números |
| password_confirmation | string | Sim | Deve ser igual a password |
| name | string | Sim | Nome completo |
| phone | string | Sim | Telefone com DDI |
| document_number | string | Sim | CPF (apenas números) |
| birth_date | string | Sim | Data no formato YYYY-MM-DD |
| accept_terms | boolean | Sim | Deve ser true |

**Response 201 (Sucesso):**

```json
{
  "message": "Conta criada com sucesso. Verifique seu e-mail.",
  "user": {
    "id": "uuid",
    "email": "novo@exemplo.com",
    "status": "PENDING_VERIFICATION"
  }
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Dados inválidos |
| 409 | E-mail ou CPF já cadastrado |

---

### POST /verify-email

Este endpoint confirma o e-mail do usuário através de um código enviado por e-mail.

**Request:**

```json
{
  "email": "novo@exemplo.com",
  "code": "123456"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| email | string | Sim | E-mail a verificar |
| code | string | Sim | Código de 6 dígitos |

**Response 200 (Sucesso):**

```json
{
  "message": "E-mail verificado com sucesso",
  "user": {
    "id": "uuid",
    "email": "novo@exemplo.com",
    "status": "ACTIVE"
  }
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Código inválido ou expirado |
| 404 | E-mail não encontrado |

---

### POST /forgot-password

Este endpoint inicia o processo de recuperação de senha.

**Request:**

```json
{
  "email": "usuario@exemplo.com"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| email | string | Sim | E-mail da conta |

**Response 200 (Sucesso):**

```json
{
  "message": "Se o e-mail existir, você receberá instruções de recuperação"
}
```

**Nota:** Este endpoint sempre retorna 200, mesmo se o e-mail não existir, para evitar enumeração de contas.

---

### POST /reset-password

Este endpoint define uma nova senha usando o token de recuperação.

**Request:**

```json
{
  "token": "abc123...",
  "password": "NovaSenha@456",
  "password_confirmation": "NovaSenha@456"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| token | string | Sim | Token recebido por e-mail |
| password | string | Sim | Nova senha |
| password_confirmation | string | Sim | Confirmação da nova senha |

**Response 200 (Sucesso):**

```json
{
  "message": "Senha alterada com sucesso"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 400 | Token inválido, expirado ou senhas não conferem |

---

### POST /mfa/enable

Este endpoint ativa autenticação de dois fatores para a conta.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "type": "totp"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| type | string | Sim | Tipo de MFA: "totp" ou "sms" |

**Response 200 (Sucesso):**

```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "qr_code_url": "data:image/png;base64,...",
  "recovery_codes": [
    "abc123",
    "def456",
    "ghi789"
  ]
}
```

---

### POST /mfa/verify

Este endpoint valida o código MFA durante o login.

**Request:**

```json
{
  "mfa_token": "eyJhbGciOiJSUzI1NiIs...",
  "code": "123456"
}
```

**Parâmetros:**

| Nome | Tipo | Obrigatório | Descrição |
|------|------|-------------|-----------|
| mfa_token | string | Sim | Token temporário recebido no login |
| code | string | Sim | Código do app authenticator ou SMS |

**Response 200 (Sucesso):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 1800
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Código MFA inválido |
| 429 | Muitas tentativas |

---

### GET /me

Este endpoint retorna informações do usuário autenticado.

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
  "document_number": "***456***",
  "status": "ACTIVE",
  "mfa_enabled": true,
  "email_verified": true,
  "phone_verified": true,
  "created_at": "2024-01-01T10:00:00Z"
}
```

**Códigos de Erro:**

| Código | Descrição |
|--------|-----------|
| 401 | Token inválido ou expirado |

## Rate Limiting

Os endpoints de autenticação possuem limites específicos:

| Endpoint | Limite |
|----------|--------|
| /login | 5 tentativas por minuto por IP |
| /register | 3 tentativas por minuto por IP |
| /forgot-password | 3 tentativas por hora por e-mail |
| /mfa/verify | 3 tentativas por minuto |

## Links Relacionados

- [Auth Service](../components/auth-service.md) - Documentação do serviço
- [Modelo de Segurança](../architecture/security-model.md) - Arquitetura de segurança
- [Erros Comuns](../troubleshooting/common-errors.md) - Solução de problemas
