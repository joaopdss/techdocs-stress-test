# Admin Dashboard

## Descrição

O Admin Dashboard é a interface administrativa da TechCorp, utilizada pelos times internos para gerenciar operações do dia-a-dia. Esta aplicação web oferece funcionalidades de gestão de pedidos, produtos, usuários, relatórios e configurações do sistema.

O dashboard foi projetado para suportar diferentes perfis de acesso, desde atendentes de suporte até administradores de sistema. Cada perfil possui um conjunto específico de permissões que determinam quais funcionalidades estão disponíveis.

A interface prioriza eficiência operacional, com atalhos de teclado, busca rápida e ações em lote para as operações mais frequentes. O dashboard também inclui widgets configuráveis para monitoramento de KPIs em tempo real.

## Responsáveis

- **Time:** Internal Tools
- **Tech Lead:** Diego Martins
- **Slack:** #internal-admin

## Stack Tecnológica

- Framework: React 18 + Vite
- Linguagem: TypeScript 5.3
- UI Library: Ant Design 5
- State Management: React Query + Zustand
- Charts: Recharts
- Tables: TanStack Table

## Perfis de Acesso

| Perfil | Descrição | Permissões Principais |
|--------|-----------|----------------------|
| `support` | Atendimento ao cliente | Visualizar pedidos, usuários |
| `operations` | Operações de fulfillment | Gerenciar pedidos, estoque |
| `finance` | Time financeiro | Relatórios, pagamentos |
| `product_manager` | Gestão de catálogo | Produtos, categorias, preços |
| `admin` | Administrador do sistema | Acesso total |

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `VITE_API_URL` | URL base da API | - |
| `VITE_AUTH_URL` | URL do auth-service | - |
| `VITE_SENTRY_DSN` | DSN do Sentry | - |
| `VITE_FEATURE_FLAGS_URL` | URL do serviço de feature flags | - |
| `VITE_WEBSOCKET_URL` | URL do WebSocket para real-time | - |

### Configuração de Build

```javascript
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'antd'],
          charts: ['recharts'],
        },
      },
    },
  },
})
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/admin-dashboard.git
cd admin-dashboard

# Instalar dependências
npm install

# Configurar variáveis de ambiente
cp .env.example .env.local

# Iniciar em modo desenvolvimento
npm run dev

# Acessar aplicação
open http://localhost:5173
```

### Usuários de Teste

| E-mail | Senha | Perfil |
|--------|-------|--------|
| support@techcorp.com | Test@123 | support |
| admin@techcorp.com | Test@123 | admin |

## Módulos Disponíveis

### 1. Gestão de Pedidos

- Lista de pedidos com filtros avançados
- Detalhes do pedido com timeline de status
- Ações: cancelar, reembolsar, reenviar
- Exportação para CSV/Excel

### 2. Gestão de Usuários

- Busca por e-mail, CPF, telefone
- Histórico de pedidos do usuário
- Ações: bloquear, desbloquear, resetar senha
- Merge de contas duplicadas

### 3. Gestão de Produtos

- CRUD de produtos e variações
- Upload em lote via CSV
- Gestão de categorias e atributos
- Controle de preços e promoções

### 4. Relatórios

- Vendas por período
- Produtos mais vendidos
- Métricas de conversão
- Exportação de dados

### 5. Configurações

- Gestão de permissões
- Configuração de integrações
- Logs de auditoria
- Feature flags

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/admin-dashboard
- **Sentry:** https://sentry.techcorp.internal/admin-dashboard
- **Logs de Auditoria:** https://kibana.techcorp.internal/admin-audit

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `admin_active_sessions` | Sessões ativas | - |
| `admin_actions_total` | Total de ações realizadas | - |
| `admin_error_rate` | Taxa de erros | > 1% |
| `admin_page_load_time` | Tempo de carregamento | > 3s |

## Troubleshooting

### Problema: Tabela de pedidos carregando lentamente

**Causa:** Muitos filtros ativos ou período muito longo.

**Solução:**
1. Reduzir período de busca
2. Remover filtros desnecessários
3. Usar exportação para análises grandes

### Problema: Ação retornando erro 403

**Causa:** Usuário sem permissão para a ação.

**Solução:**
1. Verificar perfil do usuário
2. Solicitar elevação de permissão ao admin
3. Verificar se a ação está disponível no perfil

### Problema: WebSocket desconectando frequentemente

**Causa:** Timeout de conexão ou problema de rede.

**Solução:**
1. Verificar estabilidade da conexão
2. Recarregar a página
3. Verificar se há firewall bloqueando WebSocket

### Problema: Dados desatualizados na tela

**Causa:** Cache do React Query ou falta de invalidação.

**Solução:**
1. Usar botão de refresh na toolbar
2. Limpar cache do navegador
3. Verificar se WebSocket está conectado

## Atalhos de Teclado

| Atalho | Ação |
|--------|------|
| `Ctrl+K` | Busca rápida |
| `Ctrl+Shift+O` | Ir para pedidos |
| `Ctrl+Shift+U` | Ir para usuários |
| `Ctrl+Shift+P` | Ir para produtos |
| `Esc` | Fechar modal |

## Auditoria

Todas as ações administrativas são registradas para auditoria:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "user_id": "uuid",
  "user_email": "admin@techcorp.com",
  "action": "order.cancel",
  "resource_type": "order",
  "resource_id": "uuid",
  "details": {
    "reason": "Cliente solicitou cancelamento"
  },
  "ip_address": "10.0.0.1"
}
```

## Links Relacionados

- [Auth Service](auth-service.md) - Autenticação e permissões
- [User Service](user-service.md) - Dados de usuários
- [User Management](user-management.md) - Guia de gestão de usuários
- [Order Service](order-service.md) - Dados de pedidos
