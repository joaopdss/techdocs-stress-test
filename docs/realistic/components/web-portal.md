# Web Portal

## Descrição

O Web Portal é a aplicação frontend principal da TechCorp, oferecendo uma experiência de e-commerce completa para os clientes. Esta aplicação single-page (SPA) permite navegação de produtos, gerenciamento de carrinho, checkout e acompanhamento de pedidos.

A interface foi desenvolvida seguindo os princípios de design responsivo, garantindo uma experiência consistente em dispositivos desktop, tablet e mobile. O portal implementa Server-Side Rendering (SSR) para otimização de SEO e performance no carregamento inicial.

A arquitetura frontend utiliza uma abordagem baseada em micro-frontends, permitindo que diferentes times desenvolvam e deployem funcionalidades de forma independente. Cada módulo pode ser atualizado sem impactar o restante da aplicação.

## Responsáveis

- **Time:** Customer Experience Frontend
- **Tech Lead:** Patricia Gomes
- **Slack:** #cx-web-portal

## Stack Tecnológica

- Framework: Next.js 14
- Linguagem: TypeScript 5.3
- UI Library: React 18
- State Management: Zustand
- Styling: Tailwind CSS + shadcn/ui
- Testing: Jest + React Testing Library + Playwright

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `NEXT_PUBLIC_API_URL` | URL base da API | - |
| `NEXT_PUBLIC_CDN_URL` | URL do CDN de assets | - |
| `NEXT_PUBLIC_GA_ID` | ID do Google Analytics | - |
| `NEXT_PUBLIC_SENTRY_DSN` | DSN do Sentry | - |
| `API_SECRET_KEY` | Chave para chamadas server-side | - |
| `REVALIDATE_INTERVAL` | Intervalo de revalidação ISR (segundos) | `60` |

### Configuração de Build

```javascript
// next.config.js
module.exports = {
  images: {
    domains: ['cdn.techcorp.com'],
    formats: ['image/avif', 'image/webp'],
  },
  i18n: {
    locales: ['pt-BR', 'en-US', 'es-ES'],
    defaultLocale: 'pt-BR',
  },
  experimental: {
    serverActions: true,
  },
}
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/web-portal.git
cd web-portal

# Instalar dependências
npm install

# Configurar variáveis de ambiente
cp .env.example .env.local
# Editar .env.local com valores de desenvolvimento

# Iniciar em modo desenvolvimento
npm run dev

# Acessar aplicação
open http://localhost:3000
```

### Executar Testes

```bash
# Testes unitários
npm run test

# Testes de integração
npm run test:integration

# Testes E2E
npm run test:e2e

# Coverage
npm run test:coverage
```

## Estrutura de Diretórios

```
src/
├── app/                    # App Router (Next.js 14)
│   ├── (auth)/            # Grupo de rotas autenticadas
│   ├── (public)/          # Grupo de rotas públicas
│   ├── api/               # API Routes
│   └── layout.tsx         # Layout raiz
├── components/
│   ├── ui/                # Componentes base (shadcn)
│   ├── features/          # Componentes de funcionalidade
│   └── layouts/           # Componentes de layout
├── hooks/                 # Custom hooks
├── lib/                   # Utilitários e configurações
├── services/              # Chamadas de API
├── stores/                # Zustand stores
└── types/                 # TypeScript types
```

## Funcionalidades Principais

### 1. Catálogo de Produtos

- Listagem com filtros e ordenação
- Busca full-text com autocomplete
- Páginas de produto com galeria de imagens
- Avaliações e reviews

### 2. Carrinho de Compras

- Persistência local (localStorage)
- Sincronização com backend ao logar
- Cálculo de frete em tempo real
- Aplicação de cupons

### 3. Checkout

- Checkout em etapas (endereço, pagamento, confirmação)
- Múltiplas formas de pagamento
- Integração com gateways via iframe seguro
- Validação de formulários em tempo real

### 4. Área do Cliente

- Histórico de pedidos
- Gerenciamento de endereços
- Preferências de conta
- Acompanhamento de entregas

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/web-portal
- **Sentry:** https://sentry.techcorp.internal/web-portal
- **Web Vitals:** https://analytics.techcorp.internal/web-vitals

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `LCP` | Largest Contentful Paint | > 2.5s |
| `FID` | First Input Delay | > 100ms |
| `CLS` | Cumulative Layout Shift | > 0.1 |
| `TTFB` | Time to First Byte | > 800ms |
| `error_rate` | Taxa de erros JavaScript | > 1% |

### Alertas Configurados

- **WebVitalsLCPHigh:** LCP acima de 2.5s por 10 minutos
- **WebPortalErrorRateHigh:** Taxa de erros JS acima de 1%
- **WebPortalAPILatencyHigh:** Latência de API acima de 2s

## Troubleshooting

### Problema: Página carregando lenta

**Causa:** Bundle muito grande, imagens não otimizadas ou API lenta.

**Solução:**
1. Analisar bundle: `npm run analyze`
2. Verificar lazy loading de componentes
3. Confirmar que imagens usam next/image
4. Verificar latência da API no Network tab

### Problema: Erros de hidratação

**Causa:** Diferença entre render do servidor e cliente.

**Solução:**
1. Verificar uso de `window` ou `document` sem verificação
2. Usar `useEffect` para código client-only
3. Verificar estados iniciais consistentes

### Problema: Estado não persistindo entre páginas

**Causa:** Store não configurada corretamente ou SSR interferindo.

**Solução:**
1. Verificar configuração do Zustand persist
2. Usar `skipHydration` se necessário
3. Verificar se está usando o provider corretamente

### Problema: Checkout travando

**Causa:** Erro de validação ou falha na comunicação com payment-service.

**Solução:**
1. Verificar console para erros de validação
2. Checar rede para falhas de requisição
3. Verificar se token de pagamento não expirou

## Links Relacionados

- [API Gateway](api-gateway.md) - Backend das requisições
- [Auth Service](auth-service.md) - Autenticação de usuários
- [Mobile App](mobile-app.md) - Versão mobile
- [Admin Dashboard](admin-dashboard.md) - Painel administrativo
