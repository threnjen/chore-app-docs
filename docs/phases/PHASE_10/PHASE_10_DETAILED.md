# Phase 10: Native Mobile Apps
**Duration: 3-4 weeks**

> **Prerequisite**: Requires Phase 9 completion. PWA must be production-stable before wrapping in native shell.

---

## Overview

Phase 10 delivers native iOS and Android apps via Capacitor, enabling native push notifications, app store presence, and enhanced device capabilities. The existing React PWA is wrapped in a native shell.

---

## Goals

- Native iOS and Android apps via Capacitor
- Native push notifications (FCM/APNs)
- App Store / Google Play presence
- Enhanced native device capabilities

---

## Deliverables

### 1. Capacitor Integration

- Capacitor project setup wrapping existing React app
- iOS Xcode project configuration
- Android Studio project configuration
- Native splash screens and app icons
- Deep linking configuration

### 2. Backend - Push Notification Service

- Device registration endpoints (`POST /api/v1/devices/register`)
- Firebase Cloud Messaging (FCM) integration for Android
- Apple Push Notification service (APNs) integration for iOS
- Unified notification service abstraction
- Platform-specific payload formatting
- Device token management (registration, refresh, deregistration)

### 3. Frontend - Native Plugins

- `@capacitor/push-notifications` - Native push notifications
- `@capacitor/camera` - Photo evidence for chores
- `@capacitor/haptics` - Haptic feedback for interactions
- `@capacitor/app` - App lifecycle management
- `@capacitor/splash-screen` - Native splash screen
- `@capacitor/status-bar` - Status bar customization

### 4. App Store Deployment

- Apple Developer account setup
- Google Play Developer account setup
- App Store Connect configuration
- Google Play Console configuration
- Privacy policy and terms of service (app store versions)
- App review compliance (both platforms)
- Beta testing via TestFlight / Internal Testing Track

### 5. Features

- Native push notifications with rich content
- Badge counts on app icon
- Notification actions (approve/reject from notification)
- Camera access for chore photo evidence
- Haptic feedback for completions/celebrations
- Biometric authentication (Face ID / Touch ID / Fingerprint)
- Background app refresh
- Offline-first with native storage

---

## Capacitor Setup

### Project Initialization

```bash
# Install Capacitor
npm install @capacitor/core @capacitor/cli

# Initialize Capacitor
npx cap init PicklesApp com.picklesapp.app

# Add platforms
npm install @capacitor/ios @capacitor/android
npx cap add ios
npx cap add android

# Install plugins
npm install @capacitor/push-notifications
npm install @capacitor/camera
npm install @capacitor/haptics
npm install @capacitor/app
npm install @capacitor/splash-screen
npm install @capacitor/status-bar
```

### Capacitor Configuration

```typescript
// capacitor.config.ts

import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.picklesapp.app',
  appName: 'PicklesApp',
  webDir: 'dist',
  bundledWebRuntime: false,
  
  plugins: {
    SplashScreen: {
      launchShowDuration: 2000,
      backgroundColor: '#10B981',
      showSpinner: false,
      androidSpinnerStyle: 'small',
      spinnerColor: '#ffffff',
    },
    PushNotifications: {
      presentationOptions: ['badge', 'sound', 'alert'],
    },
    Camera: {
      allowEditing: false,
    },
  },
  
  server: {
    // For development only
    url: process.env.VITE_API_URL,
    cleartext: true,
  },
  
  ios: {
    contentInset: 'automatic',
    scheme: 'PicklesApp',
  },
  
  android: {
    allowMixedContent: true,
  },
};

export default config;
```

---

## Database Schema

### Device Tokens Table

```sql
CREATE TABLE device_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_token TEXT NOT NULL,
    platform VARCHAR(10) NOT NULL CHECK (platform IN ('ios', 'android', 'web')),
    device_name VARCHAR(255),
    app_version VARCHAR(20),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_used_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(user_id, device_token)
);

CREATE INDEX idx_device_tokens_user ON device_tokens(user_id);
CREATE INDEX idx_device_tokens_platform ON device_tokens(platform);
CREATE INDEX idx_device_tokens_active ON device_tokens(is_active) WHERE is_active = true;
```

### Notification Preferences Table

```sql
CREATE TABLE notification_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    chore_reminders BOOLEAN DEFAULT true,
    chore_overdue BOOLEAN DEFAULT true,
    chore_completed BOOLEAN DEFAULT true,
    chore_approved BOOLEAN DEFAULT true,
    allowance_received BOOLEAN DEFAULT true,
    goal_reached BOOLEAN DEFAULT true,
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(user_id)
);
```

---

## Push Notification Backend

### Device Registration

```python
# app/routers/devices.py

from fastapi import APIRouter, Depends
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/devices", tags=["devices"])

class DeviceRegistration(BaseModel):
    device_token: str
    platform: str  # 'ios', 'android', 'web'
    device_name: Optional[str] = None
    app_version: Optional[str] = None

@router.post("/register")
async def register_device(
    registration: DeviceRegistration,
    user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Register a device for push notifications."""
    # Check if token already exists
    existing = db.query(DeviceToken).filter(
        DeviceToken.device_token == registration.device_token
    ).first()
    
    if existing:
        # Update existing registration
        existing.user_id = user.id
        existing.platform = registration.platform
        existing.device_name = registration.device_name
        existing.app_version = registration.app_version
        existing.is_active = True
        existing.last_used_at = datetime.utcnow()
    else:
        # Create new registration
        device = DeviceToken(
            user_id=user.id,
            device_token=registration.device_token,
            platform=registration.platform,
            device_name=registration.device_name,
            app_version=registration.app_version
        )
        db.add(device)
    
    db.commit()
    return {"status": "registered"}

@router.get("")
async def list_devices(
    user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """List user's registered devices."""
    devices = db.query(DeviceToken).filter(
        DeviceToken.user_id == user.id,
        DeviceToken.is_active == True
    ).all()
    
    return devices

@router.delete("/{device_id}")
async def unregister_device(
    device_id: UUID,
    user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Unregister a device."""
    device = db.query(DeviceToken).filter(
        DeviceToken.id == device_id,
        DeviceToken.user_id == user.id
    ).first()
    
    if device:
        device.is_active = False
        db.commit()
    
    return {"status": "unregistered"}
```

### Unified Notification Service

```python
# app/services/notifications.py

import firebase_admin
from firebase_admin import credentials, messaging
import httpx
import jwt
from datetime import datetime, timedelta

class NotificationService:
    """Unified service for sending push notifications across platforms."""
    
    def __init__(self):
        # Initialize Firebase Admin SDK
        cred = credentials.Certificate(settings.FIREBASE_CREDENTIALS_PATH)
        firebase_admin.initialize_app(cred)
        
        # APNs configuration
        self.apns_key = settings.APNS_KEY
        self.apns_key_id = settings.APNS_KEY_ID
        self.apns_team_id = settings.APNS_TEAM_ID
    
    async def send_notification(
        self,
        user_id: str,
        title: str,
        body: str,
        data: Optional[dict] = None,
        notification_type: str = "general"
    ):
        """Send notification to all user's devices."""
        # Check notification preferences
        prefs = await get_notification_preferences(user_id)
        if not self._should_send(prefs, notification_type):
            return
        
        # Get user's devices
        devices = await get_user_devices(user_id)
        
        for device in devices:
            try:
                if device.platform == 'ios':
                    await self._send_apns(device, title, body, data)
                elif device.platform == 'android':
                    await self._send_fcm(device, title, body, data)
                elif device.platform == 'web':
                    await self._send_web_push(device, title, body, data)
            except Exception as e:
                logger.error(f"Failed to send notification to {device.id}: {e}")
                # Deactivate invalid tokens
                if "Invalid" in str(e) or "Unregistered" in str(e):
                    device.is_active = False
    
    async def _send_fcm(self, device, title: str, body: str, data: dict):
        """Send notification via Firebase Cloud Messaging (Android)."""
        message = messaging.Message(
            notification=messaging.Notification(
                title=title,
                body=body,
            ),
            data=data or {},
            token=device.device_token,
            android=messaging.AndroidConfig(
                priority='high',
                notification=messaging.AndroidNotification(
                    icon='notification_icon',
                    color='#10B981',
                    channel_id='picklesapp_default',
                ),
            ),
        )
        
        response = messaging.send(message)
        logger.info(f"FCM message sent: {response}")
    
    async def _send_apns(self, device, title: str, body: str, data: dict):
        """Send notification via Apple Push Notification service (iOS)."""
        # Generate JWT for APNs
        token = self._generate_apns_token()
        
        payload = {
            'aps': {
                'alert': {
                    'title': title,
                    'body': body,
                },
                'sound': 'default',
                'badge': await self._get_badge_count(device.user_id),
            },
            'data': data or {},
        }
        
        url = f"https://api.push.apple.com/3/device/{device.device_token}"
        
        async with httpx.AsyncClient(http2=True) as client:
            response = await client.post(
                url,
                json=payload,
                headers={
                    'authorization': f'bearer {token}',
                    'apns-topic': 'com.picklesapp.app',
                    'apns-push-type': 'alert',
                    'apns-priority': '10',
                }
            )
            
            if response.status_code != 200:
                raise Exception(f"APNs error: {response.text}")
    
    def _generate_apns_token(self) -> str:
        """Generate JWT for APNs authentication."""
        now = datetime.utcnow()
        
        headers = {
            'alg': 'ES256',
            'kid': self.apns_key_id,
        }
        
        payload = {
            'iss': self.apns_team_id,
            'iat': now,
        }
        
        return jwt.encode(payload, self.apns_key, algorithm='ES256', headers=headers)
    
    def _should_send(self, prefs, notification_type: str) -> bool:
        """Check if notification should be sent based on preferences."""
        if not prefs:
            return True
        
        # Check quiet hours
        now = datetime.now().time()
        if prefs.quiet_hours_start and prefs.quiet_hours_end:
            if prefs.quiet_hours_start <= now <= prefs.quiet_hours_end:
                return False
        
        # Check notification type preference
        pref_map = {
            'chore_reminder': prefs.chore_reminders,
            'chore_overdue': prefs.chore_overdue,
            'chore_completed': prefs.chore_completed,
            'chore_approved': prefs.chore_approved,
            'allowance_received': prefs.allowance_received,
            'goal_reached': prefs.goal_reached,
        }
        
        return pref_map.get(notification_type, True)
```

---

## Frontend Push Notifications

### Push Notification Registration

```typescript
// src/services/pushNotifications.ts

import { PushNotifications } from '@capacitor/push-notifications';
import { Capacitor } from '@capacitor/core';
import { api } from './api';

export async function registerPushNotifications() {
  if (!Capacitor.isNativePlatform()) {
    return; // Web uses different push mechanism
  }
  
  // Request permission
  const permission = await PushNotifications.requestPermissions();
  
  if (permission.receive !== 'granted') {
    console.log('Push notification permission denied');
    return;
  }
  
  // Register with APNs/FCM
  await PushNotifications.register();
  
  // Handle registration success
  PushNotifications.addListener('registration', async (token) => {
    console.log('Push registration token:', token.value);
    
    // Send token to backend
    await api.post('/devices/register', {
      device_token: token.value,
      platform: Capacitor.getPlatform(),
      app_version: import.meta.env.VITE_APP_VERSION,
    });
  });
  
  // Handle registration error
  PushNotifications.addListener('registrationError', (error) => {
    console.error('Push registration error:', error);
  });
  
  // Handle received notifications
  PushNotifications.addListener('pushNotificationReceived', (notification) => {
    console.log('Push notification received:', notification);
    
    // Show in-app notification if app is in foreground
    showInAppNotification({
      title: notification.title,
      body: notification.body,
      data: notification.data,
    });
  });
  
  // Handle notification tap
  PushNotifications.addListener('pushNotificationActionPerformed', (action) => {
    console.log('Push notification action:', action);
    
    const data = action.notification.data;
    
    // Navigate based on notification type
    switch (data.type) {
      case 'chore_completed':
        navigate(`/chores/assignments/${data.assignment_id}`);
        break;
      case 'chore_approved':
        navigate('/dashboard');
        break;
      case 'allowance_received':
        navigate('/accounts');
        break;
      case 'goal_reached':
        navigate(`/goals/${data.goal_id}`);
        break;
    }
  });
}
```

### Notification Actions

```typescript
// iOS notification actions (configured in Xcode)
// Add to ios/App/App/AppDelegate.swift

let approveAction = UNNotificationAction(
  identifier: "APPROVE_CHORE",
  title: "Approve",
  options: [.foreground]
)

let rejectAction = UNNotificationAction(
  identifier: "REJECT_CHORE",
  title: "Reject",
  options: [.destructive]
)

let choreCategory = UNNotificationCategory(
  identifier: "CHORE_APPROVAL",
  actions: [approveAction, rejectAction],
  intentIdentifiers: []
)

UNUserNotificationCenter.current().setNotificationCategories([choreCategory])
```

---

## Camera Integration

```typescript
// src/services/camera.ts

import { Camera, CameraResultType, CameraSource } from '@capacitor/camera';

export async function takeChorePhoto(): Promise<string | null> {
  try {
    const photo = await Camera.getPhoto({
      quality: 80,
      allowEditing: false,
      resultType: CameraResultType.Base64,
      source: CameraSource.Camera,
      width: 1024,
      height: 1024,
    });
    
    if (photo.base64String) {
      // Upload to S3 and get URL
      const photoUrl = await uploadPhoto(photo.base64String);
      return photoUrl;
    }
    
    return null;
  } catch (error) {
    console.error('Camera error:', error);
    return null;
  }
}

// Usage in chore completion
async function completeChoreWithPhoto(assignmentId: string) {
  const photoUrl = await takeChorePhoto();
  
  await api.post(`/chores/assignments/${assignmentId}/complete`, {
    photo_url: photoUrl,
  });
}
```

---

## Haptic Feedback

```typescript
// src/services/haptics.ts

import { Haptics, ImpactStyle, NotificationType } from '@capacitor/haptics';
import { Capacitor } from '@capacitor/core';

export async function hapticSuccess() {
  if (Capacitor.isNativePlatform()) {
    await Haptics.notification({ type: NotificationType.Success });
  }
}

export async function hapticWarning() {
  if (Capacitor.isNativePlatform()) {
    await Haptics.notification({ type: NotificationType.Warning });
  }
}

export async function hapticLight() {
  if (Capacitor.isNativePlatform()) {
    await Haptics.impact({ style: ImpactStyle.Light });
  }
}

export async function hapticMedium() {
  if (Capacitor.isNativePlatform()) {
    await Haptics.impact({ style: ImpactStyle.Medium });
  }
}

// Usage: celebration on chore completion
async function onChoreComplete() {
  await hapticSuccess();
  showConfetti();
}
```

---

## Biometric Authentication

```typescript
// src/services/biometrics.ts

import { BiometricAuth } from '@capawesome-team/capacitor-biometric-auth';

export async function checkBiometricAvailability() {
  const result = await BiometricAuth.checkBiometry();
  return result.isAvailable;
}

export async function authenticateWithBiometrics(): Promise<boolean> {
  try {
    await BiometricAuth.authenticate({
      reason: 'Confirm your identity',
      iosFallbackTitle: 'Use Passcode',
    });
    return true;
  } catch (error) {
    console.error('Biometric auth failed:', error);
    return false;
  }
}

// Usage: require biometric for sensitive actions
async function transferMoney(amount: number) {
  if (amount > 50) {
    const authenticated = await authenticateWithBiometrics();
    if (!authenticated) {
      throw new Error('Biometric authentication required');
    }
  }
  
  await executeTransfer(amount);
}
```

---

## App Store Configuration

### iOS App Store

```yaml
# App Store Connect Settings

App Name: PicklesApp - Family Chores
Subtitle: Chore & Allowance Management
Category: Finance (Primary), Lifestyle (Secondary)
Age Rating: 4+ (No objectionable content)

Privacy Policy URL: https://picklesapp.com/privacy
Support URL: https://picklesapp.com/support

Keywords: chores, allowance, kids money, family finance, parenting, savings

Description: |
  PicklesApp makes managing family chores and allowances fun and easy!
  
  Features:
  • Create and schedule chores with flexible recurrence
  • Track allowances and savings goals
  • Age-adaptive interface for kids 5-17
  • Support for multi-household families
  • Real-time notifications
  
  Your parenting, your way.

Screenshots Required:
  - 6.7" iPhone (1290 x 2796)
  - 6.5" iPhone (1284 x 2778)
  - 5.5" iPhone (1242 x 2208)
  - 12.9" iPad (2048 x 2732)

App Privacy:
  - Data Linked to You: Contact Info, User Content
  - Data Not Linked to You: Usage Data, Diagnostics
```

### Google Play Store

```yaml
# Play Console Settings

App Name: PicklesApp - Family Chores
Short Description: Chore & allowance management for families
Category: Finance → Family Finance

Content Rating: Everyone (IARC)

Privacy Policy: https://picklesapp.com/privacy
Target Audience: Parents, Families

Description: |
  PicklesApp makes managing family chores and allowances fun and easy!
  
  [Full description...]

Feature Graphic: 1024 x 500 px
Screenshots: Phone, 7" Tablet, 10" Tablet

Data Safety:
  - Data collected: Email, Name, Financial info
  - Data not shared with third parties
  - Data encrypted in transit
  - Users can request data deletion
```

---

## API Endpoints

### Devices

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/devices/register` | POST | Register device for push notifications |
| `/api/v1/devices` | GET | List user's registered devices |
| `/api/v1/devices/{device_id}` | DELETE | Unregister a device |

### Notification Preferences

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/notifications/preferences` | GET | Get notification preferences |
| `/api/v1/notifications/preferences` | PUT | Update notification preferences |

---

## Notification Types

| Type | Trigger | Platforms | Actionable |
|------|---------|-----------|------------|
| `chore_reminder` | X hours before due | iOS, Android, Web | View chore |
| `chore_overdue` | Chore past due date | iOS, Android, Web | View chore |
| `chore_completed` | Child completes chore (to parent) | iOS, Android, Web | Approve/Reject |
| `chore_approved` | Parent approves (to child) | iOS, Android, Web | View dashboard |
| `chore_rejected` | Parent rejects (to child) | iOS, Android, Web | View chore |
| `allowance_received` | Scheduled allowance paid | iOS, Android, Web | View account |
| `goal_reached` | Savings goal achieved | iOS, Android, Web | View goal |
| `age_bracket_change` | Birthday triggers new UI | iOS, Android, Web | View features |

---

## Testing Requirements

### Native App Tests

```javascript
// e2e/native-app.spec.js

describe('Native App', () => {
  test('app launches successfully', async () => {
    await device.launchApp();
    await expect(element(by.id('splash-screen'))).toBeVisible();
    await waitFor(element(by.id('login-screen'))).toBeVisible().withTimeout(5000);
  });
  
  test('push notification permission requested', async () => {
    await device.launchApp({ permissions: { notifications: 'unset' } });
    await expect(element(by.text('Allow Notifications'))).toBeVisible();
  });
  
  test('biometric auth works', async () => {
    await device.setBiometrics({ enrolled: true });
    await element(by.id('transfer-button')).tap();
    await device.matchFace();
    await expect(element(by.id('transfer-success'))).toBeVisible();
  });
  
  test('camera captures photo', async () => {
    await element(by.id('complete-with-photo')).tap();
    await device.takeScreenshot();
    await element(by.id('capture-button')).tap();
    await expect(element(by.id('photo-preview'))).toBeVisible();
  });
});
```

### Push Notification Tests

```python
def test_push_notification_delivery_ios():
    """Test APNs notification delivery."""
    device = create_device(platform='ios', token='valid-apns-token')
    
    service = NotificationService()
    await service.send_notification(
        user_id=device.user_id,
        title="Test Notification",
        body="This is a test",
        notification_type="chore_reminder"
    )
    
    # Verify notification was sent (mock APNs)
    assert mock_apns.send.called

def test_push_notification_respects_quiet_hours():
    """Notifications not sent during quiet hours."""
    prefs = create_preferences(
        quiet_hours_start=time(22, 0),
        quiet_hours_end=time(7, 0)
    )
    
    with freeze_time("2026-02-15 23:00:00"):
        service = NotificationService()
        await service.send_notification(
            user_id=prefs.user_id,
            title="Test",
            body="Test",
            notification_type="chore_reminder"
        )
        
        # Should not send
        assert not mock_apns.send.called
```

### Coverage Requirements

- Device registration: 100%
- Push notification delivery: 100%
- Notification preferences: 100%
- Camera integration: 80%
- Biometric authentication: 80%

---

## Build & Deployment

### iOS Build

```bash
# Build web assets
npm run build

# Sync with Capacitor
npx cap sync ios

# Open Xcode
npx cap open ios

# Build in Xcode:
# Product → Archive → Distribute App → App Store Connect
```

### Android Build

```bash
# Build web assets
npm run build

# Sync with Capacitor
npx cap sync android

# Open Android Studio
npx cap open android

# Build signed APK/AAB:
# Build → Generate Signed Bundle/APK
```

### CI/CD for Native Builds

```yaml
# .github/workflows/native-build.yml

name: Native Build

on:
  release:
    types: [created]

jobs:
  ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build web
        run: npm run build
      
      - name: Sync Capacitor
        run: npx cap sync ios
      
      - name: Build iOS
        uses: yukiarrr/ios-build-action@v1
        with:
          p12-base64: ${{ secrets.IOS_P12_BASE64 }}
          mobileprovision-base64: ${{ secrets.IOS_MOBILEPROVISION }}
          team-id: ${{ secrets.APPLE_TEAM_ID }}
      
      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1

  android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build web
        run: npm run build
      
      - name: Sync Capacitor
        run: npx cap sync android
      
      - name: Build Android
        run: |
          cd android
          ./gradlew bundleRelease
      
      - name: Sign AAB
        run: |
          jarsigner -keystore ${{ secrets.ANDROID_KEYSTORE }} \
            -storepass ${{ secrets.KEYSTORE_PASSWORD }} \
            android/app/build/outputs/bundle/release/app-release.aab \
            picklesapp
      
      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT }}
          packageName: com.picklesapp.app
          releaseFiles: android/app/build/outputs/bundle/release/app-release.aab
          track: internal
```

---

## Cost Considerations

| Item | Cost |
|------|------|
| Apple Developer Program | $99/year |
| Google Play Developer | $25 one-time |
| Firebase (free tier) | $0 |
| APNs | Included with Apple Developer |
| **Total Year 1** | **~$125** |

---

## Definition of Done

### Must Have
- [ ] iOS app builds and runs on device
- [ ] Android app builds and runs on device
- [ ] Push notifications working on both platforms
- [ ] Device registration/deregistration
- [ ] Notification preferences respected
- [ ] App submitted to App Store Connect
- [ ] App submitted to Google Play Console
- [ ] TestFlight beta available
- [ ] Internal Testing track available

### Nice to Have
- [ ] Camera photo evidence for chores
- [ ] Biometric authentication
- [ ] Haptic feedback
- [ ] Background app refresh
- [ ] Notification actions (approve/reject from notification)

---

## Dependencies

- **Requires**: Phase 9 (production stability)
- **Enables**: Public launch on app stores

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| App Store rejection | Follow guidelines closely, prepare for revision |
| Push token expiration | Handle token refresh, retry failed sends |
| Platform-specific bugs | Thorough testing on real devices |
| Native plugin compatibility | Use Capacitor official plugins when possible |

---

*Phase 10 Document - Last Updated: February 2026*
