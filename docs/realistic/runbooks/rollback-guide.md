# Guia de Reverter Versão

## Visão Geral

Este guia descreve os procedimentos para reverter uma aplicação para uma versão anterior quando um problema é identificado após o deploy. A reversão deve ser considerada quando issues críticos são detectados e não podem ser corrigidos rapidamente.

## Quando Reverter

Considere reverter quando:

- Taxa de erros aumentou significativamente após o deploy
- Latência P99 está acima do aceitável
- Funcionalidades críticas estão quebradas
- Incidente de severidade alta foi aberto

**Não reverta automaticamente se:**
- O problema pode ser resolvido com hotfix em menos de 30 minutos
- O problema afeta apenas uma pequena porcentagem de usuários
- Reverter causaria perda de dados

## Métodos de Reversão

### Método 1: Reverter via ArgoCD (Recomendado)

Este é o método mais seguro e rastreável.

#### Passo 1: Identificar a Versão Anterior

```bash
# Listar histórico de deploys
argocd app history <app-name>

# Exemplo de saída:
# ID  DATE                           REVISION
# 1   2024-01-15T10:00:00Z          abc123 (v1.1.0)
# 2   2024-01-16T14:00:00Z          def456 (v1.2.0)  <- atual com problema
```

#### Passo 2: Executar a Reversão

```bash
# Reverter para revisão específica
argocd app rollback <app-name> <ID>

# Exemplo: reverter para ID 1
argocd app rollback my-service 1

# Verificar status
argocd app get <app-name>
```

#### Passo 3: Confirmar a Reversão

```bash
# Verificar pods
kubectl get pods -n production -l app=<app-name>

# Verificar versão da imagem
kubectl get deployment <app-name> -n production -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Método 2: Reverter via Argo Rollouts

Se estiver usando Argo Rollouts para canary ou blue-green:

```bash
# Abortar rollout atual e reverter
kubectl argo rollouts abort <app-name> -n production

# Ou reverter para revisão específica
kubectl argo rollouts undo <app-name> -n production --to-revision=<revision>

# Monitorar
kubectl argo rollouts get rollout <app-name> -n production --watch
```

### Método 3: Reverter via Git (Último Recurso)

Use este método se os anteriores não funcionarem.

```bash
# Identificar commit da versão estável
git log --oneline -10

# Criar branch de reversão
git checkout -b revert/v1.2.0 main

# Reverter commit problemático
git revert <commit-hash>

# Push e criar PR emergencial
git push origin revert/v1.2.0
gh pr create --base main --title "URGENTE: Reverter v1.2.0"
```

## Procedimento Completo de Reversão

### Fase 1: Detecção e Decisão (5 minutos)

1. **Confirmar o problema**
   - Verificar métricas no Grafana
   - Verificar logs no Kibana
   - Confirmar que problema iniciou após deploy

2. **Comunicar a equipe**
   ```
   @here Problema detectado após deploy de <app-name>
   Iniciando procedimento de reversão
   ```

3. **Decidir reverter**
   - Se problema crítico → reverter imediatamente
   - Se problema moderado → avaliar hotfix vs reversão

### Fase 2: Execução da Reversão (5-10 minutos)

1. **Executar reversão**
   ```bash
   argocd app rollback <app-name> <ID>
   ```

2. **Monitorar progresso**
   ```bash
   kubectl rollout status deployment/<app-name> -n production
   ```

3. **Verificar saúde**
   ```bash
   kubectl get pods -n production -l app=<app-name>
   ```

### Fase 3: Validação (10-15 minutos)

1. **Verificar métricas**
   - Taxa de erros voltou ao normal?
   - Latência voltou ao normal?
   - Throughput voltou ao normal?

2. **Executar smoke tests**
   ```bash
   # Testar endpoints críticos
   curl -I https://api.techcorp.com/health
   curl -I https://api.techcorp.com/v1/products
   ```

3. **Verificar logs**
   ```bash
   kubectl logs -n production -l app=<app-name> --tail=100 | grep -i error
   ```

### Fase 4: Comunicação (5 minutos)

1. **Atualizar status**
   ```
   @here Reversão de <app-name> concluída
   Versão atual: v1.1.0
   Métricas normalizadas
   ```

2. **Criar incident report**
   - O que aconteceu
   - Impacto
   - Ações tomadas
   - Próximos passos

## Casos Especiais

### Reverter com Migrations de Banco

Se a versão problemática incluiu migrations:

1. **Avaliar impacto**
   - Migration é reversível?
   - Há perda de dados?
   - Tempo necessário para reverter migration?

2. **Se migration for reversível:**
   ```bash
   # Executar migration de rollback
   kubectl exec -it <pod> -n production -- ./migrate rollback

   # Depois reverter aplicação
   argocd app rollback <app-name> <ID>
   ```

3. **Se migration NÃO for reversível:**
   - Manter banco na versão atual
   - Reverter apenas aplicação (se compatível)
   - Ou fazer hotfix para corrigir problema

### Reverter Múltiplos Serviços

Se o deploy afetou múltiplos serviços:

```bash
# Reverter na ordem inversa do deploy
argocd app rollback service-c <ID>
argocd app rollback service-b <ID>
argocd app rollback service-a <ID>
```

### Reverter com Feature Flags

Se a feature problemática usa feature flag:

```bash
# Opção mais rápida: desabilitar feature
curl -X POST https://feature-flags.techcorp.internal/api/flags/new-checkout/disable

# Depois avaliar se reversão ainda é necessária
```

## Checklist de Reversão

### Antes de Reverter

- [ ] Confirmar que problema é causado pelo deploy
- [ ] Identificar versão estável anterior
- [ ] Comunicar time e stakeholders
- [ ] Verificar se há migrations envolvidas

### Durante a Reversão

- [ ] Executar comando de reversão
- [ ] Monitorar rollout status
- [ ] Verificar pods estão healthy

### Após Reverter

- [ ] Validar métricas normalizadas
- [ ] Executar smoke tests
- [ ] Comunicar conclusão da reversão
- [ ] Documentar o incidente
- [ ] Agendar post-mortem

## Prevenindo Necessidade de Reversão

Para reduzir a necessidade de reversões:

1. **Deploy canary** - Detectar problemas com tráfego parcial
2. **Feature flags** - Desabilitar features sem reverter código
3. **Testes automatizados** - Detectar problemas antes do deploy
4. **Smoke tests pós-deploy** - Detectar problemas imediatamente

## Links Relacionados

- [Guia de Deploy](deploy-guide.md) - Processo de deploy
- [Resposta a Incidentes](incident-response.md) - Gestão de incidentes
- [Kubernetes Cluster](../components/kubernetes-cluster.md) - Comandos K8s
- [Erros Comuns](../troubleshooting/common-errors.md) - Diagnóstico
