# Resposta a Incidentes

## Visão Geral

Este guia descreve o processo de resposta a incidentes da TechCorp. O objetivo é restaurar os serviços rapidamente, minimizar o impacto aos usuários e aprender com cada incidente para prevenir recorrências.

## Severidades de Incidente

| Severidade | Descrição | Tempo de Resposta | Exemplos |
|------------|-----------|-------------------|----------|
| **SEV1** | Sistema completamente indisponível | 15 minutos | Site fora do ar, checkout quebrado |
| **SEV2** | Funcionalidade crítica degradada | 30 minutos | Pagamentos falhando, login lento |
| **SEV3** | Funcionalidade não-crítica afetada | 2 horas | Busca lenta, notificações atrasadas |
| **SEV4** | Problema menor, sem impacto ao usuário | 24 horas | Log excessivo, alerta falso |

## Processo de Resposta

### Fase 1: Detecção e Triagem (0-15 min)

#### 1.1 Identificar o Incidente

Incidentes podem ser detectados por:
- Alertas automáticos (PagerDuty, Alertmanager)
- Relatos de usuários (Suporte, Redes Sociais)
- Monitoramento manual (Grafana, Logs)

#### 1.2 Criar Canal de Incidente

```
/incident create "Descrição breve do problema" --severity SEV2
```

Isso cria automaticamente:
- Canal Slack #incident-YYYY-MM-DD-NNNN
- Ticket no sistema de tracking
- Página de status (se SEV1/SEV2)

#### 1.3 Escalar se Necessário

| Severidade | Escalar para |
|------------|--------------|
| SEV1 | Tech Lead + Engineering Manager + CTO |
| SEV2 | Tech Lead + Engineering Manager |
| SEV3 | Tech Lead |
| SEV4 | Time de plantão |

### Fase 2: Diagnóstico (15-45 min)

#### 2.1 Coletar Informações

```bash
# Verificar status dos serviços
kubectl get pods -n production

# Verificar métricas recentes
# Acessar Grafana: https://grafana.techcorp.internal

# Verificar logs
kubectl logs -n production -l app=<suspeito> --since=30m

# Verificar deploys recentes
argocd app history <app-name>
```

#### 2.2 Identificar Causa Raiz

Perguntas chave:
- Quando o problema começou?
- Houve deploy recente?
- Houve mudança de configuração?
- O problema é em um serviço específico ou geral?
- O problema afeta todos os usuários ou um subconjunto?

#### 2.3 Comunicar Status

Atualizar o canal de incidente a cada 15 minutos:

```
**Atualização [10:30]**
- Status: Investigando
- Hipótese: Possível problema no payment-service após deploy
- Próximos passos: Verificando logs e métricas
- ETA para resolução: Ainda indeterminado
```

### Fase 3: Mitigação (Variável)

#### Opção A: Reverter Deploy

Se problema começou após deploy:

```bash
argocd app rollback <app-name> <revision-id>
```

Ver: [Guia de Reverter](rollback-guide.md)

#### Opção B: Escalar Recursos

Se problema é de capacidade:

```bash
kubectl scale deployment <app-name> -n production --replicas=10
```

Ver: [Guia de Escalar](scaling-guide.md)

#### Opção C: Desabilitar Feature

Se feature específica está causando problema:

```bash
curl -X POST https://feature-flags.techcorp.internal/api/flags/<feature>/disable
```

#### Opção D: Failover

Se problema é de infraestrutura:

```bash
# Executar failover de banco
aws rds failover-db-cluster --db-cluster-identifier techcorp-production

# Redirecionar tráfego
kubectl patch ingress <app-name> -n production --patch '...'
```

### Fase 4: Resolução

#### 4.1 Confirmar Resolução

- [ ] Métricas voltaram ao normal
- [ ] Logs não mostram erros
- [ ] Testes manuais passando
- [ ] Nenhum alerta ativo

#### 4.2 Comunicar Resolução

```
**Incidente Resolvido [11:15]**
- Causa: Deploy com bug no payment-service
- Resolução: Rollback para versão anterior
- Duração: 45 minutos
- Impacto: ~5% das transações de pagamento falharam
```

#### 4.3 Atualizar Status Page

Para SEV1/SEV2, atualizar página de status:

```
/statuspage update "Serviços restaurados. Investigando causa raiz."
```

### Fase 5: Post-Mortem

Para incidentes SEV1, SEV2 e SEV3 recorrentes:

#### 5.1 Agendar Post-Mortem

- Dentro de 48 horas para SEV1/SEV2
- Dentro de 1 semana para SEV3

#### 5.2 Template de Post-Mortem

```markdown
# Post-Mortem: [Título do Incidente]

## Resumo
[Descrição breve do incidente]

## Timeline
- [HH:MM] Evento
- [HH:MM] Evento
- [HH:MM] Incidente resolvido

## Impacto
- Duração: X minutos/horas
- Usuários afetados: X%
- Transações perdidas: X

## Causa Raiz
[Análise técnica detalhada]

## Resolução
[O que foi feito para resolver]

## Lições Aprendidas
- O que funcionou bem
- O que não funcionou bem
- Onde tivemos sorte

## Action Items
- [ ] [Ação] - [Responsável] - [Prazo]
- [ ] [Ação] - [Responsável] - [Prazo]
```

## Runbooks por Tipo de Incidente

### Serviço Indisponível (503)

```bash
# 1. Verificar pods
kubectl get pods -n production -l app=<service>

# 2. Se pods em CrashLoopBackOff
kubectl logs <pod> -n production --previous

# 3. Se pods pending
kubectl describe pod <pod> -n production

# 4. Se pods healthy mas 503
# Verificar service e ingress
kubectl get svc,ingress -n production | grep <service>
```

### Alta Latência

```bash
# 1. Verificar métricas de latência
# Grafana > Dashboard > Service Health

# 2. Verificar dependências
# Grafana > Dashboard > Dependencies

# 3. Se banco lento
# Ver métricas PostgreSQL

# 4. Se cache lento
# Ver métricas Redis
```

### Alta Taxa de Erros

```bash
# 1. Identificar erros nos logs
kubectl logs -n production -l app=<service> --since=10m | grep -i error

# 2. Verificar se específico de endpoint
# Grafana > Dashboard > API Errors by Endpoint

# 3. Verificar dependências externas
curl -I https://api.provedor-externo.com/health
```

### Banco de Dados Indisponível

```bash
# 1. Verificar status no RDS
aws rds describe-db-instances --db-instance-identifier techcorp-primary

# 2. Verificar conexões
kubectl exec -it <pod> -n production -- psql -c "SELECT count(*) FROM pg_stat_activity"

# 3. Failover se necessário
aws rds failover-db-cluster --db-cluster-identifier techcorp-production
```

## Contatos de Emergência

| Papel | Nome | Contato |
|-------|------|---------|
| Engineering Manager | [Nome] | [Telefone] |
| Tech Lead Platform | [Nome] | [Telefone] |
| DBA de Plantão | [Nome] | [Telefone] |
| Security | [Nome] | [Telefone] |

## Links Relacionados

- [Guia de Deploy](deploy-guide.md) - Deploy de correções
- [Guia de Reverter](rollback-guide.md) - Reverter versões
- [Guia de Escalar](scaling-guide.md) - Escalar recursos
- [Monitoring Stack](../components/monitoring-stack.md) - Ferramentas de monitoramento
