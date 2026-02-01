# Mobile App Strategy

## Overview

PicklesApp targets two deployment platforms to maximize reach and user experience:

1. **Web App (Browser-based)** - Works on desktop and mobile browsers
2. **Native Mobile Apps** - iOS and Android apps distributed via App Store and Google Play

This document outlines the strategy for achieving both targets with **minimal code duplication** using a shared React codebase wrapped with Capacitor for native deployment.

---

## Table of Contents

1. [Platform Comparison](#platform-comparison)
2. [Technology Decision: Capacitor](#technology-decision-capacitor)
3. [Architecture Overview](#architecture-overview)
4. [Implementation Phases](#implementation-phases)
5. [Push Notification Strategy](#push-notification-strategy)
6. [Native Plugin Usage](#native-plugin-usage)
7. [App Store Deployment](#app-store-deployment)
8. [Development Workflow](#development-workflow)
9. [Testing Strategy](#testing-strategy)
10. [Cost Analysis](#cost-analysis)

---

## Platform Comparison

### Target Platforms

| Platform | Distribution | Notification Support | Offline | Primary Users |
|----------|--------------|---------------------|---------|---------------|
| **Desktop Web** | Browser | Web Push (limited) | Service Worker | Parents |
| **Mobile Web (PWA)** | Browser + Add to Home | Web Push (iOS limited*) | Service Worker | Parents, Teens |
| **iOS App** | App Store | Full APNs | Native Storage | Kids, Parents |
| **Android App** | Google Play | Full FCM | Native Storage | Kids, Parents |

\* iOS PWA push notifications require the app to be installed to home screen (iOS 16.4+), and user adoption of this is typically low.

### Why Native Apps Are Essential

| Requirement | PWA Capability | Native App Capability |
|-------------|----------------|----------------------|
| **Reliable Push Notifications** | ⚠️ Limited on iOS | ✅ Full support |
| **App Store Presence** | ❌ Not applicable | ✅ Discoverable, trusted |
| **Background Refresh** | ⚠️ Limited | ✅ Full support |
| **Biometric Auth** | ⚠️ WebAuthn (limited) | ✅ Face ID, Touch ID, Fingerprint |
| **Camera Access** | ✅ Supported | ✅ Better UX |
| **Badge Counts** | ❌ Not on iOS | ✅ Full support |
| **Haptic Feedback** | ⚠️ Limited | ✅ Full support |

### User Journey by Platform

```
Parents (Primary Management):
├── Desktop Web (80%) - Creating chores, reviewing finances
└── Mobile App (20%) - Quick approvals, notifications

Kids (Daily Interaction):
├── Mobile App (90%) - View chores, mark complete, check balance
└── Mobile Web (10%) - Occasional browser access
```

---

## Technology Decision: Capacitor

### Why Capacitor Over Alternatives

| Technology | Pros | Cons | Fit for PicklesApp |
|------------|------|------|-------------------|
| **Capacitor** | Wraps existing React app, minimal changes, web-first | Less "native feel" than React Native | ✅ **Best fit** |
| **React Native** | True native components, better performance | Requires rewrite (~60-80%), separate codebase | ❌ Too costly |
| **Flutter** | Great performance, growing ecosystem | Complete rewrite, new language (Dart) | ❌ Too costly |
| **Ionic** | Similar to Capacitor | Older, Angular-focused historically | ⚠️ Capacitor is better |

### Capacitor Advantages for PicklesApp

1. **Minimal Migration Effort**: Our React app works as-is
2. **Single Codebase**: Web, iOS, and Android from one source
3. **Progressive Enhancement**: Start with PWA, add native features
4. **Native API Access**: Full access to device capabilities via plugins
5. **Web-First Development**: Develop in browser, deploy everywhere
6. **Maintained by Ionic Team**: Active development, good documentation

### Code Reuse Estimate

| Component | Reusable | Changes Needed |
|-----------|----------|----------------|
| React Components | 95% | Minor responsive adjustments |
| API Services | 100% | None |
| Contexts/State | 100% | None |
| Routing | 95% | Deep link configuration |
| Styling (Tailwind) | 90% | Safe area adjustments |
| Push Notifications | 0% | New native implementation |

---

## Architecture Overview

### Multi-Platform Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Shared React Codebase                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │ Components  │ │   Pages     │ │  Contexts   │ │  Services   │   │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    ↓            ↓            ↓
         ┌──────────────┐ ┌───────────┐ ┌───────────┐
         │  Web Build   │ │ iOS Build │ │Android Bld│
         │   (Vite)     │ │(Capacitor)│ │(Capacitor)│
         └──────────────┘ └───────────┘ └───────────┘
                │              │              │
                ↓              ↓              ↓
         ┌──────────────┐ ┌───────────┐ ┌───────────┐
         │   Browser    │ │ App Store │ │Google Play│
         │   + PWA      │ │  (iOS)    │ │ (Android) │
         └──────────────┘ └───────────┘ └───────────┘
```

### Project Structure with Capacitor

```
chore-app-frontend/
├── src/                          # Shared React source
│   ├── components/
│   ├── contexts/
│   ├── pages/
│   ├── services/
│   │   ├── api.js               # HTTP client
│   │   └── pushNotifications.js # Platform-aware notifications
│   └── utils/
│       └── platform.js          # Platform detection utilities
├── public/
├── android/                      # Android project (Capacitor)
│   ├── app/
│   │   ├── src/main/
│   │   │   ├── AndroidManifest.xml
│   │   │   └── java/.../MainActivity.java
│   │   └── build.gradle
│   └── capacitor.config.json
├── ios/                          # iOS project (Capacitor)
│   ├── App/
│   │   ├── App/
│   │   │   ├── AppDelegate.swift
│   │   │   └── Info.plist
│   │   └── App.xcworkspace
│   └── capacitor.config.json
├── capacitor.config.ts           # Capacitor configuration
├── package.json
└── vite.config.js
```

### Platform Detection

```javascript
// src/utils/platform.js
import { Capacitor } from '@capacitor/core';

export const Platform = {
  isNative: Capacitor.isNativePlatform(),
  isWeb: !Capacitor.isNativePlatform(),
  isIOS: Capacitor.getPlatform() === 'ios',
  isAndroid: Capacitor.getPlatform() === 'android',
  
  /**
   * Check if running as installed PWA
   */
  isPWA: () => {
    return window.matchMedia('(display-mode: standalone)').matches ||
           window.navigator.standalone === true;
  },
};

// Usage
if (Platform.isNative) {
  // Use Capacitor plugin
  await PushNotifications.requestPermissions();
} else {
  // Use Web Push API
  await Notification.requestPermission();
}
```

---

## Implementation Phases

### Phase 7: PWA Foundation (Current Plan)
- Service worker with offline caching
- Web Push notifications (VAPID)
- Install prompts
- Responsive mobile UI

### Phase 10: Native Mobile Apps (New)

#### Week 1: Capacitor Setup
- [ ] Install Capacitor core and CLI
- [ ] Initialize iOS and Android projects
- [ ] Configure app identifiers and bundle IDs
- [ ] Set up native splash screens and icons
- [ ] Test basic app builds on simulators

#### Week 2: Push Notifications
- [ ] Set up Firebase project (Android + cross-platform)
- [ ] Configure APNs (iOS)
- [ ] Implement backend device registration
- [ ] Create unified notification service
- [ ] Test notifications on both platforms

#### Week 3: Native Features
- [ ] Camera integration for chore photos
- [ ] Biometric authentication
- [ ] Haptic feedback
- [ ] Deep linking
- [ ] Safe area handling

#### Week 4: App Store Deployment
- [ ] Apple Developer account setup
- [ ] Google Play Developer account setup
- [ ] Beta testing (TestFlight, Internal Track)
- [ ] App store listings and screenshots
- [ ] Privacy policy compliance
- [ ] Submit for review

---

## Push Notification Strategy

### Unified Notification Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Backend Notification Service                    │
├─────────────────────────────────────────────────────────────────────┤
│  NotificationService                                                 │
│  ├── send_notification(user_id, type, data)                        │
│  │   ├── Get user's registered devices                              │
│  │   ├── For each device:                                           │
│  │   │   ├── iOS → APNs                                            │
│  │   │   ├── Android → FCM                                         │
│  │   │   └── Web → Web Push (VAPID)                                │
│  │   └── Log delivery status                                        │
│  └── register_device(user_id, token, platform)                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Backend Implementation

```python
# app/services/notification_service.py
from enum import Enum
from typing import Optional
import firebase_admin
from firebase_admin import messaging
import httpx

class NotificationPlatform(str, Enum):
    IOS = "ios"
    ANDROID = "android"
    WEB = "web"

class NotificationType(str, Enum):
    CHORE_REMINDER = "chore_reminder"
    CHORE_COMPLETED = "chore_completed"
    CHORE_APPROVED = "chore_approved"
    CHORE_REJECTED = "chore_rejected"
    CHORE_OVERDUE = "chore_overdue"
    ALLOWANCE_RECEIVED = "allowance_received"
    GOAL_REACHED = "goal_reached"
    AGE_BRACKET_CHANGE = "age_bracket_change"

class NotificationService:
    """
    Unified push notification service for all platforms.
    """
    
    def __init__(self, db: Session):
        self.db = db
        self._init_firebase()
    
    def _init_firebase(self):
        """Initialize Firebase Admin SDK for FCM/APNs."""
        if not firebase_admin._apps:
            cred = firebase_admin.credentials.Certificate(
                settings.FIREBASE_SERVICE_ACCOUNT_PATH
            )
            firebase_admin.initialize_app(cred)
    
    async def send_notification(
        self,
        user_id: UUID,
        notification_type: NotificationType,
        title: str,
        body: str,
        data: Optional[dict] = None,
        action_url: Optional[str] = None,
    ) -> dict:
        """
        Send push notification to all user's registered devices.
        
        Returns dict with delivery results per device.
        """
        devices = self.db.query(DeviceToken).filter(
            DeviceToken.user_id == user_id,
            DeviceToken.is_active == True
        ).all()
        
        results = {}
        
        for device in devices:
            try:
                if device.platform == NotificationPlatform.WEB:
                    result = await self._send_web_push(device, title, body, data, action_url)
                else:
                    # iOS and Android both use Firebase
                    result = await self._send_firebase(device, title, body, data, action_url)
                
                results[device.id] = {"success": True, "result": result}
                device.last_used_at = datetime.utcnow()
                
            except Exception as e:
                results[device.id] = {"success": False, "error": str(e)}
                
                # Deactivate invalid tokens
                if self._is_invalid_token_error(e):
                    device.is_active = False
        
        self.db.commit()
        return results
    
    async def _send_firebase(
        self,
        device: DeviceToken,
        title: str,
        body: str,
        data: Optional[dict],
        action_url: Optional[str],
    ) -> str:
        """Send notification via Firebase Cloud Messaging (iOS + Android)."""
        
        # Build platform-specific configuration
        if device.platform == NotificationPlatform.IOS:
            apns = messaging.APNSConfig(
                payload=messaging.APNSPayload(
                    aps=messaging.Aps(
                        alert=messaging.ApsAlert(title=title, body=body),
                        badge=await self._get_badge_count(device.user_id),
                        sound="default",
                    )
                )
            )
            android = None
        else:
            android = messaging.AndroidConfig(
                priority="high",
                notification=messaging.AndroidNotification(
                    title=title,
                    body=body,
                    icon="ic_notification",
                    color="#4f46e5",
                )
            )
            apns = None
        
        message = messaging.Message(
            notification=messaging.Notification(title=title, body=body),
            data={
                **(data or {}),
                "type": notification_type.value,
                "url": action_url or "",
            },
            token=device.device_token,
            android=android,
            apns=apns,
        )
        
        return messaging.send(message)
    
    async def _send_web_push(
        self,
        device: DeviceToken,
        title: str,
        body: str,
        data: Optional[dict],
        action_url: Optional[str],
    ) -> dict:
        """Send notification via Web Push (VAPID)."""
        from pywebpush import webpush
        
        payload = {
            "title": title,
            "body": body,
            "icon": "/icons/icon-192x192.png",
            "badge": "/icons/badge-72x72.png",
            "url": action_url,
            "data": data,
        }
        
        return webpush(
            subscription_info=json.loads(device.device_token),
            data=json.dumps(payload),
            vapid_private_key=settings.VAPID_PRIVATE_KEY,
            vapid_claims={"sub": f"mailto:{settings.VAPID_EMAIL}"},
        )
    
    async def register_device(
        self,
        user_id: UUID,
        device_token: str,
        platform: NotificationPlatform,
        device_name: Optional[str] = None,
        app_version: Optional[str] = None,
    ) -> DeviceToken:
        """Register a device for push notifications."""
        
        # Upsert device token
        existing = self.db.query(DeviceToken).filter(
            DeviceToken.user_id == user_id,
            DeviceToken.device_token == device_token,
        ).first()
        
        if existing:
            existing.is_active = True
            existing.updated_at = datetime.utcnow()
            existing.device_name = device_name or existing.device_name
            existing.app_version = app_version or existing.app_version
            device = existing
        else:
            device = DeviceToken(
                user_id=user_id,
                device_token=device_token,
                platform=platform,
                device_name=device_name,
                app_version=app_version,
            )
            self.db.add(device)
        
        self.db.commit()
        return device
```

### Frontend Implementation

```javascript
// src/services/pushNotifications.js
import { Capacitor } from '@capacitor/core';
import { PushNotifications } from '@capacitor/push-notifications';
import api from './api';

class PushNotificationService {
  constructor() {
    this.isNative = Capacitor.isNativePlatform();
  }

  /**
   * Initialize push notifications based on platform
   */
  async initialize() {
    if (this.isNative) {
      return this.initializeNative();
    } else {
      return this.initializeWeb();
    }
  }

  /**
   * Native push notification setup (iOS/Android)
   */
  async initializeNative() {
    // Request permission
    const permStatus = await PushNotifications.requestPermissions();
    
    if (permStatus.receive !== 'granted') {
      console.warn('Push notification permission denied');
      return false;
    }

    // Register with APNs/FCM
    await PushNotifications.register();

    // Listen for registration success
    PushNotifications.addListener('registration', async (token) => {
      console.log('Push registration success:', token.value);
      
      // Send token to backend
      await this.registerDeviceToken(token.value);
    });

    // Listen for registration errors
    PushNotifications.addListener('registrationError', (error) => {
      console.error('Push registration error:', error);
    });

    // Handle notification received while app is in foreground
    PushNotifications.addListener('pushNotificationReceived', (notification) => {
      console.log('Push received:', notification);
      // Show in-app notification
      this.showInAppNotification(notification);
    });

    // Handle notification tap
    PushNotifications.addListener('pushNotificationActionPerformed', (action) => {
      console.log('Push action performed:', action);
      // Navigate to relevant screen
      this.handleNotificationAction(action.notification.data);
    });

    return true;
  }

  /**
   * Web Push notification setup
   */
  async initializeWeb() {
    if (!('Notification' in window) || !('serviceWorker' in navigator)) {
      console.warn('Web Push not supported');
      return false;
    }

    const permission = await Notification.requestPermission();
    
    if (permission !== 'granted') {
      console.warn('Web Push permission denied');
      return false;
    }

    const registration = await navigator.serviceWorker.ready;
    
    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: this.urlBase64ToUint8Array(
        import.meta.env.VITE_VAPID_PUBLIC_KEY
      ),
    });

    // Send subscription to backend
    await this.registerWebPushSubscription(subscription);
    
    return true;
  }

  /**
   * Register device token with backend
   */
  async registerDeviceToken(token) {
    const platform = Capacitor.getPlatform(); // 'ios' or 'android'
    
    await api.post('/api/v1/devices/register', {
      device_token: token,
      platform: platform,
      device_name: await this.getDeviceName(),
      app_version: import.meta.env.VITE_APP_VERSION,
    });
  }

  /**
   * Register web push subscription with backend
   */
  async registerWebPushSubscription(subscription) {
    await api.post('/api/v1/devices/register', {
      device_token: JSON.stringify(subscription),
      platform: 'web',
      device_name: navigator.userAgent,
    });
  }

  /**
   * Handle notification tap navigation
   */
  handleNotificationAction(data) {
    if (!data?.url) return;
    
    // Use router to navigate
    window.location.href = data.url;
  }

  /**
   * Convert VAPID key for web push
   */
  urlBase64ToUint8Array(base64String) {
    const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
    const base64 = (base64String + padding)
      .replace(/-/g, '+')
      .replace(/_/g, '/');
    
    const rawData = window.atob(base64);
    const outputArray = new Uint8Array(rawData.length);
    
    for (let i = 0; i < rawData.length; ++i) {
      outputArray[i] = rawData.charCodeAt(i);
    }
    
    return outputArray;
  }
}

export default new PushNotificationService();
```

---

## Native Plugin Usage

### Required Capacitor Plugins

```bash
# Core
npm install @capacitor/core @capacitor/cli

# Essential plugins
npm install @capacitor/push-notifications  # Push notifications
npm install @capacitor/camera              # Photo evidence
npm install @capacitor/haptics             # Haptic feedback
npm install @capacitor/app                 # App lifecycle
npm install @capacitor/splash-screen       # Splash screen
npm install @capacitor/status-bar          # Status bar control
npm install @capacitor/keyboard            # Keyboard events
npm install @capacitor/preferences         # Local storage
npm install @capacitor/device              # Device info
npm install @capacitor/browser             # In-app browser

# Optional - Phase 10+
npm install @capacitor/local-notifications # Local notifications
npm install @capacitor/share               # Native share sheet
npm install @capacitor/filesystem          # File system access
```

### Plugin Usage Examples

#### Camera (Chore Photo Evidence)

```javascript
import { Camera, CameraResultType, CameraSource } from '@capacitor/camera';

async function takeChorePhoto() {
  const image = await Camera.getPhoto({
    quality: 80,
    allowEditing: false,
    resultType: CameraResultType.Base64,
    source: CameraSource.Camera,
    width: 1024,
    height: 1024,
  });
  
  return image.base64String;
}
```

#### Haptics (Completion Celebration)

```javascript
import { Haptics, ImpactStyle, NotificationType } from '@capacitor/haptics';

async function celebrateCompletion() {
  // Success haptic
  await Haptics.notification({ type: NotificationType.Success });
}

async function buttonTap() {
  // Light tap
  await Haptics.impact({ style: ImpactStyle.Light });
}
```

#### Status Bar (Theme-Aware)

```javascript
import { StatusBar, Style } from '@capacitor/status-bar';

async function setLightMode() {
  await StatusBar.setStyle({ style: Style.Light });
  await StatusBar.setBackgroundColor({ color: '#ffffff' });
}

async function setDarkMode() {
  await StatusBar.setStyle({ style: Style.Dark });
  await StatusBar.setBackgroundColor({ color: '#1f2937' });
}
```

---

## App Store Deployment

### iOS (App Store Connect)

#### Requirements
- Apple Developer Program ($99/year)
- Mac with Xcode 15+
- Valid code signing certificates

#### Checklist
- [ ] Bundle ID: `com.picklesapp.app`
- [ ] App name and subtitle
- [ ] Description and keywords
- [ ] Screenshots (6.5", 5.5", iPad Pro)
- [ ] App icon (1024x1024)
- [ ] Privacy policy URL
- [ ] Age rating questionnaire
- [ ] App Review Information (test account)

#### Build Commands
```bash
# Build web assets
npm run build

# Sync to iOS project
npx cap sync ios

# Open in Xcode
npx cap open ios

# Build in Xcode: Product → Archive → Distribute
```

### Android (Google Play Console)

#### Requirements
- Google Play Developer account ($25 one-time)
- Signed release APK/AAB

#### Checklist
- [ ] Package name: `com.picklesapp.app`
- [ ] App name and short description
- [ ] Full description
- [ ] Screenshots (phone, 7" tablet, 10" tablet)
- [ ] Feature graphic (1024x500)
- [ ] App icon (512x512)
- [ ] Privacy policy URL
- [ ] Content rating questionnaire
- [ ] Target audience and content

#### Build Commands
```bash
# Build web assets
npm run build

# Sync to Android project
npx cap sync android

# Open in Android Studio
npx cap open android

# Build signed AAB: Build → Generate Signed Bundle/APK
```

### CI/CD with Fastlane

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  desc "Push a new beta build to TestFlight"
  lane :beta do
    build_app(
      workspace: "App.xcworkspace",
      scheme: "App"
    )
    upload_to_testflight
  end
  
  desc "Push a new release build to the App Store"
  lane :release do
    build_app(
      workspace: "App.xcworkspace",
      scheme: "App"
    )
    upload_to_app_store(
      skip_screenshots: true,
      skip_metadata: true
    )
  end
end
```

---

## Development Workflow

### Local Development

```bash
# Develop in browser (fastest iteration)
npm run dev

# Test on iOS Simulator
npm run build && npx cap sync ios && npx cap run ios

# Test on Android Emulator
npm run build && npx cap sync android && npx cap run android

# Test on physical device (iOS)
npm run build && npx cap sync ios && npx cap run ios --target=<device-id>
```

### Live Reload (Development)

```typescript
// capacitor.config.ts
import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.picklesapp.app',
  appName: 'PicklesApp',
  webDir: 'dist',
  server: {
    // Enable for development live reload
    url: 'http://192.168.1.100:5173',
    cleartext: true,
  },
};

export default config;
```

### Capacitor Configuration

```typescript
// capacitor.config.ts (production)
import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.picklesapp.app',
  appName: 'PicklesApp',
  webDir: 'dist',
  
  plugins: {
    PushNotifications: {
      presentationOptions: ['badge', 'sound', 'alert'],
    },
    SplashScreen: {
      launchShowDuration: 2000,
      backgroundColor: '#4f46e5',
      showSpinner: false,
    },
    Keyboard: {
      resize: 'body',
      resizeOnFullScreen: true,
    },
  },
  
  ios: {
    contentInset: 'automatic',
    preferredContentMode: 'mobile',
  },
  
  android: {
    allowMixedContent: false,
  },
};

export default config;
```

---

## Testing Strategy

### Platform-Specific Testing

| Test Type | Web | iOS Simulator | Android Emulator | Physical Device |
|-----------|-----|---------------|------------------|-----------------|
| Unit tests | ✅ | N/A | N/A | N/A |
| Component tests | ✅ | N/A | N/A | N/A |
| E2E (Playwright) | ✅ | ❌ | ❌ | ❌ |
| Push notifications | ⚠️ | ❌ (no APNs) | ✅ | ✅ Required |
| Camera | ⚠️ | ✅ | ✅ | ✅ |
| Haptics | ❌ | ✅ | ✅ | ✅ |
| Biometrics | ❌ | ✅ | ✅ | ✅ |

### Testing Checklist

#### Pre-Release
- [ ] All unit tests pass
- [ ] All E2E tests pass (web)
- [ ] App launches on iOS Simulator
- [ ] App launches on Android Emulator
- [ ] Navigation works correctly
- [ ] Offline mode works
- [ ] Push notifications work (physical devices)
- [ ] Camera permission and capture work
- [ ] Deep links work
- [ ] Safe areas are respected (notch, home indicator)

#### App Store Compliance
- [ ] No crashes on launch
- [ ] No placeholder content
- [ ] Privacy policy accessible
- [ ] Required permissions have usage descriptions
- [ ] No private API usage
- [ ] Content rating accurate

---

## Cost Analysis

### One-Time Costs

| Item | Cost | Notes |
|------|------|-------|
| Apple Developer Program | $99/year | Required for iOS |
| Google Play Developer | $25 | One-time fee |
| Firebase (Blaze plan) | $0 | Free tier sufficient |

### Ongoing Costs

| Item | Estimated Monthly | Notes |
|------|-------------------|-------|
| Firebase Cloud Messaging | $0 | Free for push notifications |
| APNs | $0 | Included with Apple Developer |
| CI/CD (GitHub Actions) | $0-20 | Free tier, pay for more minutes |

### Total First Year: ~$150

---

## Future Native-Only Features (Post Phase 10)

Features that would require native-only implementation:

| Feature | Complexity | Value |
|---------|------------|-------|
| Apple Watch companion app | High | Medium |
| Widgets (iOS/Android) | Medium | High |
| Siri/Google Assistant integration | Medium | Medium |
| NFC for physical chore cards | Low | Low |
| AR chore visualization | High | Low |

---

## Migration Path Summary

```
Phase 1-6: Web App Foundation
    ↓
Phase 7: PWA Enhancement
    ├── Service Worker
    ├── Offline Support
    └── Web Push Notifications
    ↓
Phase 9: Production Polish
    ↓
Phase 10: Native Mobile Apps
    ├── Capacitor Integration
    ├── iOS App (App Store)
    ├── Android App (Google Play)
    └── Native Push Notifications (FCM/APNs)
```

The key insight is that **no major refactoring is needed** before Phase 10. The current React architecture is already Capacitor-ready, and native features will be added progressively via plugins.
