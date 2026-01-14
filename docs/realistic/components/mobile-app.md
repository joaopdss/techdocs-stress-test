# Mobile App

## Description

The Mobile App is TechCorp's mobile application, available for iOS and Android. This application offers a shopping experience optimized for mobile devices, with native features such as push notifications, biometrics, and camera for barcode scanning.

The application was developed with React Native, allowing code sharing between iOS and Android platforms. Performance-critical components were implemented natively to ensure a smooth experience.

The app implements an offline-first architecture, allowing users to browse the catalog even without internet connection. Data is automatically synchronized when the connection is restored.

## Owners

- **Team:** Mobile Engineering
- **Tech Lead:** Vanessa Campos
- **Slack:** #mobile-app

## Technology Stack

- Framework: React Native 0.73
- Language: TypeScript 5.3
- State Management: Zustand + React Query
- Navigation: React Navigation 6
- Push: Firebase Cloud Messaging (Android) + APNs (iOS)
- Analytics: Firebase Analytics

## System Requirements

| Platform | Minimum Version |
|----------|-----------------|
| iOS | 14.0 |
| Android | 8.0 (API 26) |

## Configuration

### Environment Variables

```bash
# .env
API_URL=https://api.techcorp.com
CDN_URL=https://cdn.techcorp.com
SENTRY_DSN=https://xxx@sentry.io/xxx
FIREBASE_PROJECT_ID=techcorp-mobile
CODEPUSH_KEY_IOS=xxx
CODEPUSH_KEY_ANDROID=xxx
```

### iOS Configuration

```ruby
# ios/Podfile
platform :ios, '14.0'

target 'TechCorpApp' do
  use_react_native!

  pod 'Firebase/Messaging'
  pod 'Firebase/Analytics'
end
```

### Android Configuration

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

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/mobile-app.git
cd mobile-app

# Install dependencies
npm install

# iOS: Install pods
cd ios && pod install && cd ..

# Start Metro bundler
npm start

# Run on iOS (requires macOS)
npm run ios

# Run on Android
npm run android
```

### Debug with Flipper

```bash
# Install Flipper
brew install --cask flipper

# Open Flipper and connect to the development app
```

## Directory Structure

```
src/
├── components/        # Reusable components
├── screens/          # App screens
├── navigation/       # Navigation configuration
├── services/         # API calls
├── stores/           # Zustand stores
├── hooks/            # Custom hooks
├── utils/            # Utilities
├── assets/           # Images, fonts
└── native/           # Custom native modules
```

## Native Features

### 1. Push Notifications

- Order status notifications
- Personalized promotions
- Abandoned cart alerts
- Deep linking to specific screens

### 2. Biometrics

- Login with Face ID / Touch ID (iOS)
- Login with Fingerprint / Face (Android)
- Sensitive data protection

### 3. Camera

- Barcode scanning for product search
- Photo upload for reviews
- QR Code scanning for promotions

### 4. Offline Mode

- Catalog cache for offline browsing
- Locally persisted cart
- Automatic sync when online

## Release Process

### CodePush (OTA Updates)

```bash
# Release for iOS (staging)
appcenter codepush release-react -a TechCorp/TechCorp-iOS -d Staging

# Release for Android (staging)
appcenter codepush release-react -a TechCorp/TechCorp-Android -d Staging

# Promote to production
appcenter codepush promote -a TechCorp/TechCorp-iOS -s Staging -d Production
```

### Native Release

```bash
# Build iOS for TestFlight
npm run build:ios:release

# Build Android for Play Console
npm run build:android:release
```

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/mobile-app
- **Sentry:** https://sentry.techcorp.internal/mobile-app
- **Firebase Console:** https://console.firebase.google.com/project/techcorp-mobile

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `app_crashes` | Crashes per session | > 0.1% |
| `app_anr_rate` | ANR rate (Android) | > 0.1% |
| `app_startup_time` | Startup time | > 3s |
| `api_error_rate` | API error rate | > 2% |

### Configured Alerts

- **MobileCrashRateHigh:** Crash rate above 0.5% for 1 hour
- **MobileANRRateHigh:** ANR rate above 0.5%
- **MobileAPIErrorsHigh:** API errors above 5%

## Troubleshooting

### Issue: App crashing on startup

**Cause:** Native version incompatibility or incorrect configuration.

**Solution:**
1. Check Sentry for stack trace
2. Test on physical device (simulators may mask issues)
3. Check if CodePush didn't break the bundle
4. Rollback via CodePush if necessary

### Issue: Push notifications not arriving

**Cause:** Token not registered or permission denied.

**Solution:**
1. Check notification permission in device settings
2. Check if token is registered: `GET /api/users/:id/push-tokens`
3. Test direct send via Firebase Console

### Issue: Biometrics not available

**Cause:** Device without support or biometrics not configured.

**Solution:**
1. Check support: `LocalAuthentication.hasHardwareAsync()`
2. Check if biometrics is registered on device
3. Offer fallback to PIN/password

### Issue: Offline sync not working

**Cause:** Sync queue issue or data conflict.

**Solution:**
1. Check sync queue logs
2. Clear local cache and force resync
3. Check network connectivity

## Related Links

- [API Gateway](api-gateway.md) - Request backend
- [Auth Service](auth-service.md) - Authentication
- [Notification Service](notification-service.md) - Push notifications
- [Web Portal](web-portal.md) - Web version
