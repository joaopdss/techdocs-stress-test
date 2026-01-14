# Guia de Deploy

## Visão Geral

Este guia descreve o processo padrão para realizar deploy de aplicações na plataforma TechCorp. O processo utiliza GitOps com ArgoCD para gerenciamento declarativo das aplicações no Kubernetes.

## Pré-requisitos

Antes de iniciar um deploy, certifique-se de:

- [ ] Ter acesso ao repositório da aplicação
- [ ] Ter permissão no ArgoCD para o ambiente alvo
- [ ] Pipeline de CI passou com sucesso
- [ ] Code review aprovado
- [ ] Testes automatizados passando

## Ambientes

| Ambiente | Branch | Cluster | Auto-sync |
|----------|--------|---------|-----------|
| Development | develop | techcorp-dev | Sim |
| Staging | release/* | techcorp-staging | Sim |
| Production | main | techcorp-production | Não |

## Processo de Deploy

### 1. Deploy para Development

O deploy para development é automático ao fazer merge para a branch `develop`.

```bash
# Criar feature branch
git checkout -b feature/nova-funcionalidade

# Desenvolver e commitar
git add .
git commit -m "feat: adiciona nova funcionalidade"

# Push e criar PR para develop
git push origin feature/nova-funcionalidade
```

Após o merge:
1. Pipeline de CI executa build e testes
2. Imagem Docker é construída e enviada ao registry
3. ArgoCD detecta mudança e faz deploy automático

### 2. Deploy para Staging

O deploy para staging é automático ao criar uma branch `release/*`.

```bash
# Criar release branch a partir de develop
git checkout develop
git pull
git checkout -b release/1.2.0

# Push da release branch
git push origin release/1.2.0
```

O ArgoCD automaticamente faz deploy no ambiente de staging.

### 3. Deploy para Production

O deploy para production requer aprovação manual e segue um processo controlado.

#### Passo 1: Preparar o Deploy

```bash
# Garantir que release branch está atualizada
git checkout release/1.2.0
git pull

# Criar PR para main
# Via GitHub UI ou gh CLI:
gh pr create --base main --head release/1.2.0 --title "Release 1.2.0"
```

#### Passo 2: Aprovar o Deploy

1. Obter aprovação do PR (mínimo 2 reviewers)
2. Verificar que todos os checks passaram
3. Fazer merge do PR

#### Passo 3: Executar o Deploy

Após o merge, o deploy NÃO é automático em produção. Execute manualmente:

```bash
# Acessar ArgoCD
# https://argocd.techcorp.internal

# Ou via CLI:
argocd login argocd.techcorp.internal

# Sincronizar aplicação
argocd app sync <app-name> --prune

# Verificar status
argocd app get <app-name>
```

#### Passo 4: Verificar o Deploy

```bash
# Verificar pods
kubectl get pods -n production -l app=<app-name>

# Verificar logs
kubectl logs -n production -l app=<app-name> --tail=100

# Verificar métricas
# Acessar Grafana: https://grafana.techcorp.internal/d/<app-name>
```

## Deploy Canary

Para deploys de alto risco, utilize a estratégia canary:

### Configuração

```yaml
# values.yaml
canary:
  enabled: true
  steps:
    - setWeight: 10
    - pause: { duration: 5m }
    - setWeight: 30
    - pause: { duration: 5m }
    - setWeight: 50
    - pause: { duration: 10m }
    - setWeight: 100
  analysis:
    successCondition: result[0] >= 0.95
    failureLimit: 3
```

### Execução

```bash
# Iniciar canary
argocd app sync <app-name> --strategy canary

# Monitorar progresso
kubectl argo rollouts get rollout <app-name> -n production --watch

# Promover manualmente (se pausado)
kubectl argo rollouts promote <app-name> -n production

# Abortar se necessário
kubectl argo rollouts abort <app-name> -n production
```

## Deploy Blue-Green

Para deploys que requerem rollback instantâneo:

### Configuração

```yaml
# values.yaml
blueGreen:
  enabled: true
  autoPromotionEnabled: false
  previewService: <app-name>-preview
  activeService: <app-name>
```

### Execução

```bash
# Fazer deploy (cria versão preview)
argocd app sync <app-name>

# Testar preview
curl https://<app-name>-preview.techcorp.internal/health

# Promover para ativo
kubectl argo rollouts promote <app-name> -n production
```

## Checklist de Deploy

### Antes do Deploy

- [ ] Código revisado e aprovado
- [ ] Testes automatizados passando
- [ ] Documentação atualizada (se necessário)
- [ ] Migrations de banco preparadas
- [ ] Feature flags configuradas (se aplicável)
- [ ] Comunicação com stakeholders

### Durante o Deploy

- [ ] Monitorar dashboard Grafana
- [ ] Verificar logs de erro
- [ ] Testar endpoints críticos
- [ ] Verificar métricas de latência

### Após o Deploy

- [ ] Verificar que todos os pods estão healthy
- [ ] Confirmar que métricas estão normais
- [ ] Executar smoke tests
- [ ] Atualizar status no canal #releases

## Troubleshooting

### Deploy travado em "Progressing"

**Causa:** Pods não conseguem iniciar ou probes falhando.

**Solução:**
1. Verificar eventos: `kubectl describe pod <pod> -n production`
2. Verificar logs: `kubectl logs <pod> -n production`
3. Verificar recursos: `kubectl top pods -n production`

### Imagem não encontrada

**Causa:** Pipeline de CI não completou ou tag incorreta.

**Solução:**
1. Verificar pipeline no CI
2. Confirmar tag da imagem no registry
3. Verificar `image` no deployment.yaml

### Secrets não disponíveis

**Causa:** External Secrets Operator não sincronizou.

**Solução:**
1. Verificar ExternalSecret: `kubectl get externalsecrets -n production`
2. Verificar logs do operator
3. Forçar sync se necessário

### ArgoCD mostra OutOfSync

**Causa:** Diferença entre estado desejado e atual.

**Solução:**
1. Verificar diff: `argocd app diff <app-name>`
2. Se esperado, sincronizar: `argocd app sync <app-name>`
3. Se não esperado, investigar mudanças manuais

## Horários Recomendados

| Tipo de Deploy | Horário Permitido | Restrições |
|----------------|-------------------|------------|
| Hotfix crítico | Qualquer | Aprovação do Tech Lead |
| Deploy normal | 09:00 - 16:00 (dias úteis) | Não às sextas após 14:00 |
| Deploy com migration | 09:00 - 12:00 (dias úteis) | Não às sextas |

## Links Relacionados

- [Guia de Reverter](rollback-guide.md) - Como reverter um deploy
- [Resposta a Incidentes](incident-response.md) - Se algo der errado
- [Kubernetes Cluster](../components/kubernetes-cluster.md) - Infraestrutura
- [Visão Geral da Arquitetura](../architecture/system-overview.md) - Arquitetura
