# Notification Service

## Descrição

O Notification Service é o microsserviço centralizado para envio de notificações da TechCorp. Este componente gerencia o disparo de comunicações através de múltiplos canais: e-mail, SMS, push notifications e mensagens in-app.

O serviço implementa um sistema de templates que permite personalização das mensagens por canal e idioma. Ele também oferece funcionalidades avançadas como agendamento de envios, controle de frequência (throttling) e preferências de opt-out por usuário.

Para garantir a entrega, o notification-service utiliza filas com retry exponencial e fallback entre provedores. Todas as notificações são registradas para auditoria e métricas de engajamento.

## Responsáveis

- **Time:** Customer Communications
- **Tech Lead:** Amanda Souza
- **Slack:** #comms-notifications

## Stack Tecnológica

- Linguagem: Node.js 20
- Framework: NestJS 10
- Banco de dados: PostgreSQL 15
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12
- Template Engine: Handlebars

## Canais Suportados

| Canal | Provedor Principal | Provedor Fallback |
|-------|-------------------|-------------------|
| E-mail | SendGrid | Amazon SES |
| SMS | Twilio | Zenvia |
| Push (iOS) | APNs | - |
| Push (Android) | FCM | - |
| In-App | WebSocket interno | - |

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `NOTIFICATION_PORT` | Porta HTTP do serviço | `3005` |
| `DATABASE_URL` | Connection string PostgreSQL | - |
| `REDIS_URL` | URL do Redis | - |
| `RABBITMQ_URL` | URL do RabbitMQ | - |
| `SENDGRID_API_KEY` | Chave da API SendGrid | - |
| `TWILIO_ACCOUNT_SID` | Account SID Twilio | - |
| `TWILIO_AUTH_TOKEN` | Auth Token Twilio | - |
| `FCM_SERVER_KEY` | Chave do Firebase Cloud Messaging | - |
| `APNS_KEY_ID` | Key ID do Apple Push | - |
| `APNS_TEAM_ID` | Team ID do Apple Push | - |

### Configuração de Rate Limiting

```yaml
# config/throttling.yaml
throttling:
  email:
    per_user_per_hour: 10
    per_user_per_day: 50
    global_per_minute: 1000
  sms:
    per_user_per_hour: 5
    per_user_per_day: 20
    global_per_minute: 500
  push:
    per_user_per_hour: 20
    per_user_per_day: 100
    global_per_minute: 5000
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/notification-service.git
cd notification-service

# Instalar dependências
npm install

# Subir dependências
docker-compose up -d postgres redis rabbitmq

# Executar migrações
npm run migration:run

# Iniciar em modo desenvolvimento
npm run start:dev

# Verificar saúde
curl http://localhost:3005/health
```

### Enviar Notificação de Teste

```bash
# Enviar e-mail de teste
curl -X POST http://localhost:3005/api/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "user_id": "uuid",
    "channel": "email",
    "template": "welcome",
    "data": {
      "user_name": "João"
    }
  }'
```

## Sistema de Templates

### Estrutura de Template

```
templates/
├── email/
│   ├── pt-BR/
│   │   ├── welcome.hbs
│   │   ├── order-confirmed.hbs
│   │   └── password-reset.hbs
│   └── en-US/
│       └── ...
├── sms/
│   ├── pt-BR/
│   │   ├── order-shipped.hbs
│   │   └── verification-code.hbs
│   └── en-US/
│       └── ...
└── push/
    └── ...
```

### Exemplo de Template

```handlebars
{{! templates/email/pt-BR/order-confirmed.hbs }}
<h1>Pedido Confirmado!</h1>
<p>Olá {{user_name}},</p>
<p>Seu pedido <strong>#{{order_number}}</strong> foi confirmado.</p>
<p>Valor total: R$ {{format_currency total}}</p>
<p>Previsão de entrega: {{format_date delivery_date}}</p>
```

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/notification-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `notifications_sent_total` | Total de notificações enviadas | - |
| `notifications_delivered_total` | Notificações entregues | - |
| `notifications_failed_total` | Falhas de envio | > 5% |
| `notifications_latency_seconds` | Latência de envio | > 30s |
| `notifications_by_channel` | Envios por canal | - |

### Alertas Configurados

- **NotificationDeliveryRateLow:** Taxa de entrega abaixo de 95% por 10 minutos
- **NotificationProviderDown:** Provedor principal indisponível
- **NotificationQueueBacklog:** Fila com mais de 10.000 mensagens pendentes
- **NotificationBounceRateHigh:** Taxa de bounce de e-mail acima de 5%

## Troubleshooting

### Problema: Notificações não estão sendo enviadas

**Causa:** Fila parada ou worker crashando.

**Solução:**
1. Verificar status da fila: `rabbitmqctl list_queues`
2. Checar logs dos workers: `kubectl logs -l app=notification-worker`
3. Reiniciar workers se necessário: `kubectl rollout restart deployment/notification-worker`

### Problema: E-mails caindo em spam

**Causa:** Configuração de SPF/DKIM incorreta ou reputação do IP baixa.

**Solução:**
1. Verificar configuração DNS (SPF, DKIM, DMARC)
2. Analisar reputação do IP no SendGrid dashboard
3. Reduzir volume temporariamente se IP estiver em blacklist

### Problema: Push notifications não chegando no iOS

**Causa:** Token do device expirado ou certificado APNs vencido.

**Solução:**
1. Verificar validade do certificado APNs
2. Forçar refresh do token no app
3. Verificar feedback service da Apple para tokens inválidos

### Problema: Limite de rate atingido para usuário

**Causa:** Configuração de throttling muito restritiva ou loop de envios.

**Solução:**
1. Verificar histórico de envios do usuário: `GET /api/notifications?user_id=<id>`
2. Identificar origem dos envios excessivos
3. Ajustar throttling se necessário ou corrigir bug no serviço origem

## Eventos Consumidos

| Evento | Origem | Template |
|--------|--------|----------|
| `user.created` | user-service | welcome |
| `order.confirmed` | order-service | order-confirmed |
| `order.shipped` | order-service | order-shipped |
| `order.delivered` | order-service | order-delivered |
| `payment.failed` | payment-service | payment-failed |
| `auth.password.reset` | auth-service | password-reset |

## Eventos Publicados

| Evento | Exchange | Descrição |
|--------|----------|-----------|
| `notification.sent` | `notification.events` | Notificação enviada |
| `notification.delivered` | `notification.events` | Entrega confirmada |
| `notification.failed` | `notification.events` | Falha no envio |
| `notification.opened` | `notification.events` | E-mail aberto |
| `notification.clicked` | `notification.events` | Link clicado |

## Links Relacionados

- [User Service](user-service.md) - Preferências de notificação do usuário
- [Order Service](order-service.md) - Eventos de pedido
- [Queue Service](queue-service.md) - Infraestrutura de filas
- [Erros Comuns](../troubleshooting/common-errors.md) - Problemas frequentes
