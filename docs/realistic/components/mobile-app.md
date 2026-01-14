# Mobile App

## Descrição

O Mobile App é a aplicação móvel da TechCorp, disponível para iOS e Android. Esta aplicação oferece uma experiência de compra otimizada para dispositivos móveis, com funcionalidades nativas como push notifications, biometria e câmera para leitura de códigos de barras.

A aplicação foi desenvolvida com React Native, permitindo compartilhamento de código entre as plataformas iOS e Android. Componentes críticos de performance foram implementados nativamente para garantir uma experiência fluida.

O app implementa uma arquitetura offline-first, permitindo que usuários naveguem no catálogo mesmo sem conexão com internet. Os dados são sincronizados automaticamente quando a conexão é restabelecida.

## Responsáveis

- **Time:** Mobile Engineering
- **Tech Lead:** Vanessa Campos
- **Slack:** #mobile-app

## Stack Tecnológica

- Framework: React Native 0.73
- Linguagem: TypeScript 5.3
- State Management: Zustand + React Query
- Navigation: React Navigation 6
- Push: Firebase Cloud Messaging (Android) + APNs (iOS)
- Analytics: Firebase Analytics

## Requisitos de Sistema

| Plataforma | Versão Mínima |
|------------|---------------|
| iOS | 14.0 |
| Android | 8.0 (API 26) |

## Configuração

### Variáveis de Ambiente

```bash
# .env
API_URL=https://api.techcorp.com
CDN_URL=https://cdn.techcorp.com
SENTRY_DSN=https://xxx@sentry.io/xxx
FIREBASE_PROJECT_ID=techcorp-mobile
CODEPUSH_KEY_IOS=xxx
CODEPUSH_KEY_ANDROID=xxx
```

### Configuração iOS

```ruby
# ios/Podfile
platform :ios, '14.0'

target 'TechCorpApp' do
  use_react_native!

  pod 'Firebase/Messaging'
  pod 'Firebase/Analytics'
end
```

### Configuração Android

```groovy
// android/app/build.gradle
android {
    compileSdkVersion 34
    defaultConfig {
        minSdkVersion 26
        targetSdkVersion 34
    }
}
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/mobile-app.git
cd mobile-app

# Instalar dependências
npm install

# iOS: Instalar pods
cd ios && pod install && cd ..

# Iniciar Metro bundler
npm start

# Rodar no iOS (requer macOS)
npm run ios

# Rodar no Android
npm run android
```

### Debug com Flipper

```bash
# Instalar Flipper
brew install --cask flipper

# Abrir Flipper e conectar ao app em desenvolvimento
```

## Estrutura de Diretórios

```
src/
├── components/        # Componentes reutilizáveis
├── screens/          # Telas do app
├── navigation/       # Configuração de navegação
├── services/         # Chamadas de API
├── stores/           # Zustand stores
├── hooks/            # Custom hooks
├── utils/            # Utilitários
├── assets/           # Imagens, fonts
└── native/           # Módulos nativos customizados
```

## Funcionalidades Nativas

### 1. Push Notifications

- Notificações de status de pedido
- Promoções personalizadas
- Alertas de carrinho abandonado
- Deep linking para telas específicas

### 2. Biometria

- Login com Face ID / Touch ID (iOS)
- Login com Fingerprint / Face (Android)
- Proteção de dados sensíveis

### 3. Câmera

- Leitura de código de barras para busca de produtos
- Upload de fotos para reviews
- Scan de QR Code para promoções

### 4. Offline Mode

- Cache de catálogo para navegação offline
- Carrinho persistido localmente
- Sincronização automática quando online

## Processo de Release

### CodePush (Atualizações OTA)

```bash
# Release para iOS (staging)
appcenter codepush release-react -a TechCorp/TechCorp-iOS -d Staging

# Release para Android (staging)
appcenter codepush release-react -a TechCorp/TechCorp-Android -d Staging

# Promover para produção
appcenter codepush promote -a TechCorp/TechCorp-iOS -s Staging -d Production
```

### Release Nativa

```bash
# Build iOS para TestFlight
npm run build:ios:release

# Build Android para Play Console
npm run build:android:release
```

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/mobile-app
- **Sentry:** https://sentry.techcorp.internal/mobile-app
- **Firebase Console:** https://console.firebase.google.com/project/techcorp-mobile

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `app_crashes` | Crashes por sessão | > 0.1% |
| `app_anr_rate` | ANR rate (Android) | > 0.1% |
| `app_startup_time` | Tempo de inicialização | > 3s |
| `api_error_rate` | Taxa de erros de API | > 2% |

### Alertas Configurados

- **MobileCrashRateHigh:** Crash rate acima de 0.5% por 1 hora
- **MobileANRRateHigh:** ANR rate acima de 0.5%
- **MobileAPIErrorsHigh:** Erros de API acima de 5%

## Troubleshooting

### Problema: App crashando na inicialização

**Causa:** Incompatibilidade de versão nativa ou configuração incorreta.

**Solução:**
1. Verificar Sentry para stack trace
2. Testar em device físico (simuladores podem mascarar problemas)
3. Verificar se CodePush não quebrou o bundle
4. Rollback via CodePush se necessário

### Problema: Push notifications não chegando

**Causa:** Token não registrado ou permissão negada.

**Solução:**
1. Verificar permissão de notificações nas configurações do device
2. Verificar se token está registrado: `GET /api/users/:id/push-tokens`
3. Testar envio direto via Firebase Console

### Problema: Biometria não disponível

**Causa:** Device sem suporte ou biometria não configurada.

**Solução:**
1. Verificar suporte: `LocalAuthentication.hasHardwareAsync()`
2. Verificar se biometria está cadastrada no device
3. Oferecer fallback para PIN/senha

### Problema: Sincronização offline não funcionando

**Causa:** Problema no queue de sincronização ou conflito de dados.

**Solução:**
1. Verificar logs do sync queue
2. Limpar cache local e forçar resync
3. Verificar conectividade de rede

## Links Relacionados

- [API Gateway](api-gateway.md) - Backend das requisições
- [Auth Service](auth-service.md) - Autenticação
- [Notification Service](notification-service.md) - Push notifications
- [Web Portal](web-portal.md) - Versão web
