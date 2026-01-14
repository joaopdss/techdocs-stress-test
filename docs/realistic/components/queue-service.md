# Queue Service

## Descrição

O Queue Service é a infraestrutura de mensageria da TechCorp, fornecendo comunicação assíncrona confiável entre os microsserviços através de filas de mensagens e tópicos de eventos.

Este componente encapsula a complexidade do RabbitMQ, oferecendo abstrações para os padrões mais comuns: filas de trabalho, pub/sub com fanout, roteamento por tópicos e dead letter queues. Ele também gerencia a infraestrutura de mensageria, incluindo criação de exchanges, bindings e políticas de retry.

A arquitetura de mensageria é fundamental para o desacoplamento dos serviços da TechCorp, permitindo que operações não-críticas sejam processadas de forma assíncrona sem bloquear a experiência do usuário.

## Responsáveis

- **Time:** Platform Engineering
- **Tech Lead:** Rafael Lima
- **Slack:** #platform-messaging

## Stack Tecnológica

- Message Broker: RabbitMQ 3.12
- Management: RabbitMQ Management Plugin
- Linguagem do SDK: Múltiplas (Go, Java, Python, Node.js)
- Monitoramento: Prometheus Exporter

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `RABBITMQ_HOST` | Host do RabbitMQ | `rabbitmq.internal` |
| `RABBITMQ_PORT` | Porta AMQP | `5672` |
| `RABBITMQ_MANAGEMENT_PORT` | Porta do Management UI | `15672` |
| `RABBITMQ_USER` | Usuário de conexão | - |
| `RABBITMQ_PASSWORD` | Senha de conexão | - |
| `RABBITMQ_VHOST` | Virtual host | `/` |
| `MESSAGE_TTL_MS` | TTL padrão de mensagens | `86400000` |
| `MAX_RETRIES` | Máximo de tentativas | `3` |

### Configuração de Clustering

```yaml
# rabbitmq.conf
cluster_formation.peer_discovery_backend = k8s
cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
cluster_formation.k8s.address_type = hostname
cluster_partition_handling = pause_minority
```

## Arquitetura de Exchanges

### Exchanges Principais

| Exchange | Tipo | Propósito |
|----------|------|-----------|
| `events.direct` | direct | Eventos diretos para serviço específico |
| `events.fanout` | fanout | Broadcast para todos os consumidores |
| `events.topic` | topic | Roteamento por padrão de tópico |
| `dlx.events` | direct | Dead letter exchange |

### Padrões de Roteamento

```
# Eventos de pedido
order.created -> exchange: events.topic, routing_key: order.created
order.confirmed -> exchange: events.topic, routing_key: order.confirmed

# Consumidores se inscrevem com padrões
notification-service: order.*
inventory-service: order.created, order.cancelled
```

## Como Executar Localmente

```bash
# Subir RabbitMQ com Docker
docker run -d --name rabbitmq-local \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin \
  rabbitmq:3.12-management

# Acessar Management UI
open http://localhost:15672
# Login: admin / admin

# Criar estrutura básica via CLI
docker exec rabbitmq-local rabbitmqadmin declare exchange name=events.topic type=topic
```

### Publicar Mensagem de Teste

```bash
# Via rabbitmqadmin
docker exec rabbitmq-local rabbitmqadmin publish \
  exchange=events.topic \
  routing_key=test.event \
  payload='{"message": "teste"}'

# Via SDK (exemplo Python)
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.basic_publish(exchange='events.topic', routing_key='test.event', body='{"message": "teste"}')
```

## Filas Principais

| Fila | Exchange | Routing Key | Consumidor |
|------|----------|-------------|------------|
| `notification.email` | events.topic | notification.email.* | notification-service |
| `notification.sms` | events.topic | notification.sms.* | notification-service |
| `inventory.updates` | events.topic | inventory.* | inventory-service |
| `order.processing` | events.direct | - | order-service |
| `payment.webhooks` | events.direct | - | payment-service |

## Políticas de Retry

### Configuração de Dead Letter

```json
{
  "queue": "notification.email",
  "arguments": {
    "x-dead-letter-exchange": "dlx.events",
    "x-dead-letter-routing-key": "notification.email.dlq",
    "x-message-ttl": 86400000
  }
}
```

### Estratégia de Retry Exponencial

```
Tentativa 1: Imediata
Tentativa 2: Após 1 minuto
Tentativa 3: Após 5 minutos
Falha final: Move para DLQ
```

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/queue-service
- **RabbitMQ Management:** https://rabbitmq.techcorp.internal
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `rabbitmq_queue_messages` | Mensagens na fila | > 10000 |
| `rabbitmq_queue_messages_unacked` | Mensagens não confirmadas | > 1000 |
| `rabbitmq_connections` | Conexões ativas | > 500 |
| `rabbitmq_channels` | Canais ativos | > 1000 |
| `rabbitmq_queue_messages_dlq` | Mensagens na DLQ | > 0 |

### Alertas Configurados

- **QueueBacklogHigh:** Fila com mais de 10.000 mensagens por 5 minutos
- **QueueDLQNotEmpty:** Mensagens na dead letter queue
- **QueueConsumerDown:** Fila sem consumidores ativos
- **RabbitMQClusterPartition:** Partição de rede no cluster

## Troubleshooting

### Problema: Fila acumulando mensagens

**Causa:** Consumidor lento, parado ou insuficiente.

**Solução:**
1. Verificar consumidores: `rabbitmqctl list_consumers`
2. Checar logs do consumidor
3. Escalar consumidores: `kubectl scale deployment/<consumer> --replicas=5`
4. Se necessário, purgar mensagens antigas: `rabbitmqadmin purge queue name=<queue>`

### Problema: Mensagens na DLQ

**Causa:** Erro no processamento após todas as tentativas.

**Solução:**
1. Inspecionar mensagens: `rabbitmqadmin get queue=<queue>.dlq count=10`
2. Identificar padrão de erro nos headers `x-death`
3. Corrigir bug no consumidor
4. Reprocessar mensagens: mover de DLQ para fila original

### Problema: Conexões esgotadas

**Causa:** Leak de conexões ou muitos consumidores.

**Solução:**
1. Identificar origem: `rabbitmqctl list_connections | sort | uniq -c`
2. Verificar se conexões estão sendo fechadas corretamente
3. Aumentar limite: `rabbitmqctl set_vm_memory_high_watermark 0.6`

### Problema: Cluster particionado

**Causa:** Problema de rede entre nós do cluster.

**Solução:**
1. Verificar status: `rabbitmqctl cluster_status`
2. Identificar nó problemático
3. Reiniciar nó particionado: `rabbitmqctl stop_app && rabbitmqctl start_app`
4. Se necessário, remover e readicionar nó ao cluster

## Boas Práticas

### Para Produtores

- Sempre usar `publisher confirms` para garantir entrega
- Implementar retry com backoff exponencial
- Usar `correlation_id` para rastreamento

### Para Consumidores

- Sempre usar `manual ack` (não auto-ack)
- Implementar idempotência (mensagens podem ser redelivered)
- Configurar `prefetch_count` adequado (não muito alto)

### Nomenclatura

```
Exchanges: {domain}.{type}  -> events.topic, commands.direct
Filas: {service}.{action}   -> notification.email, order.processing
Routing keys: {entity}.{event} -> order.created, user.updated
```

## Links Relacionados

- [Notification Service](notification-service.md) - Principal consumidor
- [Order Service](order-service.md) - Produtor de eventos de pedido
- [Padrões de Integração](../architecture/integration-patterns.md) - Padrões de mensageria
- [Visão Geral da Arquitetura](../architecture/system-overview.md) - Papel no sistema
