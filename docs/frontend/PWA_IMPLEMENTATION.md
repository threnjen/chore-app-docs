# PWA Implementation Guide

## Overview

PicklesApp is built as a Progressive Web App (PWA) to provide a native app-like experience on mobile devices while maintaining a single codebase. The PWA implementation enables offline functionality, home screen installation, and background sync capabilities.

> **Mobile Strategy Note**: PWA is **Phase 7** of development and serves as the foundation for mobile experience. In **Phase 10**, we will wrap this same React codebase with **Capacitor** to create native iOS and Android apps for App Store/Google Play distribution with full push notification support. See [MOBILE_APP_STRATEGY.md](./MOBILE_APP_STRATEGY.md) for the complete mobile roadmap.

### PWA vs Native App Comparison

| Feature | PWA (Phase 7) | Native App (Phase 10) |
|---------|---------------|----------------------|
| Distribution | Browser / Add to Home | App Store / Google Play |
| Push Notifications | Web Push (limited iOS) | Full APNs/FCM |
| Offline Support | ✅ Service Worker | ✅ Native + SW |
| App Icon Badge | ❌ | ✅ |
| Camera Access | ✅ | ✅ (better UX) |
| Biometric Auth | ⚠️ WebAuthn | ✅ Face ID/Touch ID |
| Code Reuse | 100% | 95%+ (Capacitor) |

## Table of Contents
1. [PWA Architecture](#pwa-architecture)
2. [Service Worker Setup](#service-worker-setup)
3. [Offline Caching Strategies](#offline-caching-strategies)
4. [Background Sync](#background-sync)
5. [Push Notifications](#push-notifications)
6. [Install Prompts](#install-prompts)
7. [Update Notifications](#update-notifications)
8. [Offline Indicators](#offline-indicators)
9. [Testing PWA Features](#testing-pwa-features)
10. [Native App Transition (Phase 10)](#native-app-transition-phase-10)

---

## PWA Architecture

### PWA Layers

```
┌─────────────────────────────────────────────────┐
│              User Interface (React)              │
│  - Components, Pages, Contexts                  │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│         Service Worker (Workbox)                │
│  - Caching strategies                           │
│  - Background sync                              │
│  - Push notification handling                   │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│         IndexedDB / Cache Storage               │
│  - Offline data storage                         │
│  - Cached assets                                │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│              Network (API)                       │
│  - When online, fetch fresh data               │
└─────────────────────────────────────────────────┘
```

---

## Service Worker Setup

### Installation with vite-plugin-pwa

```bash
# Install dependencies
npm install -D vite-plugin-pwa workbox-window
```

### Vite Configuration

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'prompt',
      includeAssets: ['favicon.ico', 'robots.txt', 'apple-touch-icon.png'],
      
      manifest: {
        name: 'PicklesApp - Family Chore & Allowance Manager',
        short_name: 'PicklesApp',
        description: 'Manage family chores, allowances, and financial education',
        theme_color: '#2563eb',
        background_color: '#ffffff',
        display: 'standalone',
        orientation: 'portrait',
        scope: '/',
        start_url: '/',
        icons: [
          {
            src: '/icons/icon-72x72.png',
            sizes: '72x72',
            type: 'image/png',
          },
          {
            src: '/icons/icon-96x96.png',
            sizes: '96x96',
            type: 'image/png',
          },
          {
            src: '/icons/icon-128x128.png',
            sizes: '128x128',
            type: 'image/png',
          },
          {
            src: '/icons/icon-144x144.png',
            sizes: '144x144',
            type: 'image/png',
          },
          {
            src: '/icons/icon-152x152.png',
            sizes: '152x152',
            type: 'image/png',
          },
          {
            src: '/icons/icon-192x192.png',
            sizes: '192x192',
            type: 'image/png',
          },
          {
            src: '/icons/icon-384x384.png',
            sizes: '384x384',
            type: 'image/png',
          },
          {
            src: '/icons/icon-512x512.png',
            sizes: '512x512',
            type: 'image/png',
            purpose: 'any maskable',
          },
        ],
        screenshots: [
          {
            src: '/screenshots/dashboard-mobile.png',
            sizes: '390x844',
            type: 'image/png',
            form_factor: 'narrow',
          },
          {
            src: '/screenshots/dashboard-desktop.png',
            sizes: '1920x1080',
            type: 'image/png',
            form_factor: 'wide',
          },
        ],
      },
      
      workbox: {
        // Service worker caching strategies
        runtimeCaching: [
          // API calls - Network first, fallback to cache
          {
            urlPattern: /^https:\/\/api\.picklesapp\.com\/api\/v1\/.*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: {
                maxEntries: 100,
                maxAgeSeconds: 60 * 60 * 24, // 24 hours
              },
              networkTimeoutSeconds: 10,
              cacheableResponse: {
                statuses: [0, 200],
              },
            },
          },
          
          // Images - Cache first
          {
            urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp)$/i,
            handler: 'CacheFirst',
            options: {
              cacheName: 'image-cache',
              expiration: {
                maxEntries: 60,
                maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
              },
            },
          },
          
          // Static assets - Stale while revalidate
          {
            urlPattern: /\.(?:js|css|woff2?)$/i,
            handler: 'StaleWhileRevalidate',
            options: {
              cacheName: 'static-assets',
              expiration: {
                maxEntries: 60,
                maxAgeSeconds: 60 * 60 * 24 * 7, // 7 days
              },
            },
          },
        ],
      },
      
      devOptions: {
        enabled: true,
        type: 'module',
      },
    }),
  ],
});
```

### Service Worker Registration

```javascript
// src/main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import { registerSW } from 'virtual:pwa-register';

// Register service worker
const updateSW = registerSW({
  onNeedRefresh() {
    // Show update notification
    if (confirm('New version available! Reload to update?')) {
      updateSW(true);
    }
  },
  onOfflineReady() {
    console.log('App ready to work offline');
  },
});

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

---

## Offline Caching Strategies

### Strategy Selection Guide

| Strategy | Use Case | Example |
|----------|----------|---------|
| **Network First** | Dynamic data that changes often | API calls, user data |
| **Cache First** | Static assets that rarely change | Images, fonts |
| **Stale While Revalidate** | Assets that can be slightly stale | CSS, JS bundles |
| **Network Only** | Real-time data, auth endpoints | Login, token refresh |
| **Cache Only** | Pre-cached app shell | index.html, manifest |

### Implementing Offline Data Storage

```javascript
// utils/offlineStorage.js
import { openDB } from 'idb';

const DB_NAME = 'picklesapp-offline';
const DB_VERSION = 1;

// Initialize IndexedDB
async function getDB() {
  return openDB(DB_NAME, DB_VERSION, {
    upgrade(db) {
      // Chores store
      if (!db.objectStoreNames.contains('chores')) {
        const choreStore = db.createObjectStore('chores', { keyPath: 'id' });
        choreStore.createIndex('status', 'status');
        choreStore.createIndex('family_id', 'family_id');
      }
      
      // Accounts store
      if (!db.objectStoreNames.contains('accounts')) {
        db.createObjectStore('accounts', { keyPath: 'id' });
      }
      
      // Pending actions store (for background sync)
      if (!db.objectStoreNames.contains('pending_actions')) {
        db.createObjectStore('pending_actions', { 
          keyPath: 'id',
          autoIncrement: true,
        });
      }
    },
  });
}

// Cache chores for offline access
export async function cacheChores(chores) {
  const db = await getDB();
  const tx = db.transaction('chores', 'readwrite');
  
  for (const chore of chores) {
    await tx.store.put(chore);
  }
  
  await tx.done;
}

// Retrieve cached chores
export async function getCachedChores(familyId) {
  const db = await getDB();
  const chores = await db.getAllFromIndex('chores', 'family_id', familyId);
  return chores;
}

// Cache accounts
export async function cacheAccounts(accounts) {
  const db = await getDB();
  const tx = db.transaction('accounts', 'readwrite');
  
  for (const account of accounts) {
    await tx.store.put(account);
  }
  
  await tx.done;
}

// Get cached accounts
export async function getCachedAccounts() {
  const db = await getDB();
  return db.getAll('accounts');
}

// Store pending action for background sync
export async function storePendingAction(action) {
  const db = await getDB();
  await db.add('pending_actions', {
    ...action,
    timestamp: Date.now(),
  });
}

// Get all pending actions
export async function getPendingActions() {
  const db = await getDB();
  return db.getAll('pending_actions');
}

// Remove completed action
export async function removePendingAction(id) {
  const db = await getDB();
  await db.delete('pending_actions', id);
}
```

### Offline-First Hook

```javascript
// hooks/useOfflineData.js
import { useState, useEffect } from 'react';
import { useOnlineStatus } from './useOnlineStatus';
import { 
  cacheChores, 
  getCachedChores,
  storePendingAction,
} from '@/utils/offlineStorage';

export function useOfflineChores(familyId) {
  const [chores, setChores] = useState([]);
  const [loading, setLoading] = useState(true);
  const [fromCache, setFromCache] = useState(false);
  const isOnline = useOnlineStatus();

  useEffect(() => {
    async function loadChores() {
      if (isOnline) {
        // Online: Fetch from API
        try {
          const response = await fetch(`/api/v1/chores?family_id=${familyId}`);
          const data = await response.json();
          setChores(data);
          
          // Cache for offline use
          await cacheChores(data);
          setFromCache(false);
        } catch (error) {
          // Network error, load from cache
          const cached = await getCachedChores(familyId);
          setChores(cached);
          setFromCache(true);
        }
      } else {
        // Offline: Load from cache
        const cached = await getCachedChores(familyId);
        setChores(cached);
        setFromCache(true);
      }
      
      setLoading(false);
    }

    loadChores();
  }, [familyId, isOnline]);

  const completeChore = async (choreId) => {
    // Optimistic update
    setChores(prev =>
      prev.map(chore =>
        chore.id === choreId
          ? { ...chore, status: 'PENDING_APPROVAL' }
          : chore
      )
    );

    if (isOnline) {
      // Online: Send to API
      try {
        await fetch(`/api/v1/chores/${choreId}/complete`, {
          method: 'POST',
        });
      } catch (error) {
        // Failed, store for background sync
        await storePendingAction({
          type: 'COMPLETE_CHORE',
          choreId,
        });
      }
    } else {
      // Offline: Store for background sync
      await storePendingAction({
        type: 'COMPLETE_CHORE',
        choreId,
      });
    }
  };

  return {
    chores,
    loading,
    fromCache,
    completeChore,
  };
}
```

---

## Background Sync

### Register Background Sync

```javascript
// utils/backgroundSync.js

/**
 * Register background sync for pending actions
 */
export async function registerBackgroundSync() {
  if ('serviceWorker' in navigator && 'sync' in ServiceWorkerRegistration.prototype) {
    const registration = await navigator.serviceWorker.ready;
    
    try {
      await registration.sync.register('sync-pending-actions');
      console.log('Background sync registered');
    } catch (error) {
      console.error('Background sync registration failed:', error);
    }
  }
}

/**
 * Sync pending actions when online
 */
export async function syncPendingActions() {
  const { getPendingActions, removePendingAction } = await import('./offlineStorage');
  const actions = await getPendingActions();

  for (const action of actions) {
    try {
      // Execute the pending action
      switch (action.type) {
        case 'COMPLETE_CHORE':
          await fetch(`/api/v1/chores/${action.choreId}/complete`, {
            method: 'POST',
            headers: {
              'Authorization': `Bearer ${localStorage.getItem('accessToken')}`,
            },
          });
          break;
          
        case 'CREATE_TRANSACTION':
          await fetch('/api/v1/transactions', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              'Authorization': `Bearer ${localStorage.getItem('accessToken')}`,
            },
            body: JSON.stringify(action.data),
          });
          break;
          
        default:
          console.warn('Unknown action type:', action.type);
      }

      // Remove from pending actions
      await removePendingAction(action.id);
    } catch (error) {
      console.error('Failed to sync action:', action, error);
      // Keep in pending actions to retry later
    }
  }
}
```

### Service Worker Sync Event

```javascript
// public/sw.js (custom service worker)
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-pending-actions') {
    event.waitUntil(syncPendingActions());
  }
});

async function syncPendingActions() {
  // Import IndexedDB utilities
  const { getPendingActions, removePendingAction } = await import('./offlineStorage.js');
  const actions = await getPendingActions();

  const syncPromises = actions.map(async (action) => {
    try {
      let response;
      
      switch (action.type) {
        case 'COMPLETE_CHORE':
          response = await fetch(`/api/v1/chores/${action.choreId}/complete`, {
            method: 'POST',
            headers: {
              'Authorization': `Bearer ${action.token}`,
            },
          });
          break;
          
        case 'CREATE_TRANSACTION':
          response = await fetch('/api/v1/transactions', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              'Authorization': `Bearer ${action.token}`,
            },
            body: JSON.stringify(action.data),
          });
          break;
      }

      if (response.ok) {
        await removePendingAction(action.id);
      }
    } catch (error) {
      console.error('Sync failed for action:', action, error);
    }
  });

  await Promise.allSettled(syncPromises);
}
```

### Trigger Sync on Connection

```javascript
// hooks/useBackgroundSync.js
import { useEffect } from 'react';
import { useOnlineStatus } from './useOnlineStatus';
import { syncPendingActions, registerBackgroundSync } from '@/utils/backgroundSync';

export function useBackgroundSync() {
  const isOnline = useOnlineStatus();

  useEffect(() => {
    // Register background sync on mount
    registerBackgroundSync();
  }, []);

  useEffect(() => {
    if (isOnline) {
      // Sync when coming back online
      syncPendingActions();
    }
  }, [isOnline]);
}
```

---

## Push Notifications

**Note:** Push notifications are implemented in Phase 6+ of the development plan.

### Request Notification Permission

```javascript
// utils/notifications.js

/**
 * Request notification permission from user
 */
export async function requestNotificationPermission() {
  if (!('Notification' in window)) {
    console.warn('Notifications not supported');
    return false;
  }

  const permission = await Notification.requestPermission();
  return permission === 'granted';
}

/**
 * Subscribe to push notifications
 */
export async function subscribeToPushNotifications() {
  const registration = await navigator.serviceWorker.ready;
  
  try {
    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(
        import.meta.env.VITE_VAPID_PUBLIC_KEY
      ),
    });

    // Send subscription to backend
    await fetch('/api/v1/notifications/subscribe', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('accessToken')}`,
      },
      body: JSON.stringify(subscription),
    });

    return true;
  } catch (error) {
    console.error('Push subscription failed:', error);
    return false;
  }
}

function urlBase64ToUint8Array(base64String) {
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
```

### Handle Push Events in Service Worker

```javascript
// Service worker push event handler
self.addEventListener('push', (event) => {
  const data = event.data.json();

  const options = {
    body: data.body,
    icon: '/icons/icon-192x192.png',
    badge: '/icons/badge-72x72.png',
    vibrate: [200, 100, 200],
    data: {
      url: data.url,
    },
    actions: data.actions || [],
  };

  event.waitUntil(
    self.registration.showNotification(data.title, options)
  );
});

// Handle notification clicks
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  event.waitUntil(
    clients.openWindow(event.notification.data.url)
  );
});
```

---

## Install Prompts

### Custom Install Prompt

```jsx
// components/pwa/InstallPrompt.jsx
import { useState, useEffect } from 'react';
import { Button } from '@/components/shared/Button';
import { Modal } from '@/components/shared/Modal';

export function InstallPrompt() {
  const [deferredPrompt, setDeferredPrompt] = useState(null);
  const [showPrompt, setShowPrompt] = useState(false);

  useEffect(() => {
    const handler = (e) => {
      // Prevent the default install prompt
      e.preventDefault();
      
      // Store the event for later use
      setDeferredPrompt(e);
      
      // Show custom install prompt
      // Only show if user hasn't dismissed before
      const dismissed = localStorage.getItem('installPromptDismissed');
      if (!dismissed) {
        setShowPrompt(true);
      }
    };

    window.addEventListener('beforeinstallprompt', handler);

    return () => {
      window.removeEventListener('beforeinstallprompt', handler);
    };
  }, []);

  const handleInstall = async () => {
    if (!deferredPrompt) return;

    // Show the install prompt
    deferredPrompt.prompt();

    // Wait for the user's response
    const { outcome } = await deferredPrompt.userChoice;

    if (outcome === 'accepted') {
      console.log('User accepted install prompt');
    } else {
      console.log('User dismissed install prompt');
    }

    // Clear the prompt
    setDeferredPrompt(null);
    setShowPrompt(false);
  };

  const handleDismiss = () => {
    setShowPrompt(false);
    localStorage.setItem('installPromptDismissed', 'true');
  };

  if (!showPrompt) return null;

  return (
    <Modal isOpen={showPrompt} onClose={handleDismiss}>
      <div className="text-center p-6">
        <div className="text-6xl mb-4">📱</div>
        
        <h2 className="text-2xl font-bold mb-3">
          Install PicklesApp
        </h2>
        
        <p className="text-gray-600 mb-6">
          Add PicklesApp to your home screen for quick access and offline use!
        </p>
        
        <div className="space-y-3">
          <Button onClick={handleInstall} className="w-full">
            Install App
          </Button>
          
          <button
            onClick={handleDismiss}
            className="text-sm text-gray-500 hover:text-gray-700"
          >
            Maybe later
          </button>
        </div>
      </div>
    </Modal>
  );
}
```

### iOS Install Instructions

```jsx
// components/pwa/IOSInstallPrompt.jsx
import { useState, useEffect } from 'react';
import { Modal } from '@/components/shared/Modal';
import { ShareIcon } from '@heroicons/react/24/outline';

export function IOSInstallPrompt() {
  const [show, setShow] = useState(false);

  useEffect(() => {
    // Check if iOS and not in standalone mode
    const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
    const isStandalone = window.navigator.standalone;
    const dismissed = localStorage.getItem('iosInstallPromptDismissed');

    if (isIOS && !isStandalone && !dismissed) {
      // Show prompt after 10 seconds
      const timer = setTimeout(() => {
        setShow(true);
      }, 10000);

      return () => clearTimeout(timer);
    }
  }, []);

  const handleDismiss = () => {
    setShow(false);
    localStorage.setItem('iosInstallPromptDismissed', 'true');
  };

  if (!show) return null;

  return (
    <Modal isOpen={show} onClose={handleDismiss}>
      <div className="p-6">
        <h2 className="text-xl font-bold mb-4">
          Install PicklesApp
        </h2>
        
        <div className="space-y-4 text-sm">
          <p>
            To install PicklesApp on your iPhone:
          </p>
          
          <ol className="list-decimal list-inside space-y-2 text-gray-700">
            <li>
              Tap the Share button{' '}
              <ShareIcon className="inline w-4 h-4" /> in Safari
            </li>
            <li>
              Scroll down and tap "Add to Home Screen"
            </li>
            <li>
              Tap "Add" in the top right corner
            </li>
          </ol>
          
          <p className="text-gray-600 text-xs">
            You'll be able to access PicklesApp like a native app!
          </p>
        </div>
        
        <button
          onClick={handleDismiss}
          className="mt-6 w-full py-2 text-blue-600 hover:text-blue-700"
        >
          Got it
        </button>
      </div>
    </Modal>
  );
}
```

---

## Update Notifications

### Detect App Updates

```jsx
// components/pwa/UpdateNotification.jsx
import { useState, useEffect } from 'react';
import { useRegisterSW } from 'virtual:pwa-register/react';
import toast from 'react-hot-toast';

export function UpdateNotification() {
  const {
    needRefresh: [needRefresh, setNeedRefresh],
    updateServiceWorker,
  } = useRegisterSW({
    onRegistered(registration) {
      console.log('Service worker registered');
    },
    onRegisterError(error) {
      console.error('Service worker registration error:', error);
    },
  });

  useEffect(() => {
    if (needRefresh) {
      toast(
        (t) => (
          <div>
            <p className="font-medium mb-2">New version available!</p>
            <div className="flex gap-2">
              <button
                onClick={() => {
                  updateServiceWorker(true);
                  toast.dismiss(t.id);
                }}
                className="px-3 py-1 bg-blue-600 text-white rounded text-sm"
              >
                Update
              </button>
              <button
                onClick={() => {
                  setNeedRefresh(false);
                  toast.dismiss(t.id);
                }}
                className="px-3 py-1 bg-gray-200 text-gray-800 rounded text-sm"
              >
                Later
              </button>
            </div>
          </div>
        ),
        {
          duration: Infinity,
          position: 'bottom-center',
        }
      );
    }
  }, [needRefresh, updateServiceWorker, setNeedRefresh]);

  return null;
}
```

---

## Offline Indicators

### Online Status Hook

```javascript
// hooks/useOnlineStatus.js
import { useState, useEffect } from 'react';

export function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  return isOnline;
}
```

### Offline Banner

```jsx
// components/pwa/OfflineBanner.jsx
import { useOnlineStatus } from '@/hooks/useOnlineStatus';
import { WifiIcon } from '@heroicons/react/24/outline';

export function OfflineBanner() {
  const isOnline = useOnlineStatus();

  if (isOnline) return null;

  return (
    <div className="fixed top-0 left-0 right-0 bg-amber-500 text-white py-2 px-4 z-50 flex items-center justify-center gap-2">
      <WifiIcon className="w-5 h-5" />
      <span className="text-sm font-medium">
        You're offline. Some features may be limited.
      </span>
    </div>
  );
}
```

### Connection Status in Components

```jsx
// components/chores/ChoreList.jsx
import { useOnlineStatus } from '@/hooks/useOnlineStatus';
import { CloudIcon } from '@heroicons/react/24/outline';

export function ChoreList({ chores, fromCache }) {
  const isOnline = useOnlineStatus();

  return (
    <div>
      {fromCache && !isOnline && (
        <div className="bg-blue-50 border border-blue-200 rounded-lg p-3 mb-4 flex items-center gap-2">
          <CloudIcon className="w-5 h-5 text-blue-600" />
          <span className="text-sm text-blue-800">
            Showing cached data. Changes will sync when you're back online.
          </span>
        </div>
      )}
      
      {/* Chore list */}
      {chores.map(chore => (
        <ChoreCard key={chore.id} chore={chore} />
      ))}
    </div>
  );
}
```

---

## Testing PWA Features

### Manual Testing Checklist

**Install & Launch:**
- [ ] App installs from browser
- [ ] App appears on home screen
- [ ] App launches in standalone mode
- [ ] Splash screen displays correctly
- [ ] Correct theme color in status bar

**Offline Functionality:**
- [ ] App loads when offline
- [ ] Cached data displays correctly
- [ ] Offline banner appears
- [ ] Actions queue for background sync
- [ ] Data syncs when back online

**Service Worker:**
- [ ] Service worker registers successfully
- [ ] Assets cache correctly
- [ ] Cache updates on new version
- [ ] Update notification appears

**Performance:**
- [ ] Initial load < 3 seconds
- [ ] Offline load < 1 second
- [ ] Smooth animations
- [ ] No jank on scroll

### Lighthouse PWA Audit

```bash
# Run Lighthouse PWA audit
npm run build
npx serve -s dist

# In Chrome DevTools
# Lighthouse -> Progressive Web App -> Generate Report

# Target scores:
# Performance: > 90
# Accessibility: > 90
# Best Practices: > 90
# SEO: > 90
# PWA: 100
```

### Automated Testing

```javascript
// tests/pwa/serviceWorker.test.js
import { describe, it, expect, beforeEach } from 'vitest';

describe('Service Worker', () => {
  beforeEach(() => {
    // Mock service worker
    global.navigator.serviceWorker = {
      register: vi.fn(),
      ready: Promise.resolve({
        sync: { register: vi.fn() },
      }),
    };
  });

  it('registers service worker', async () => {
    const registration = await navigator.serviceWorker.register('/sw.js');
    expect(registration).toBeDefined();
  });

  it('registers background sync', async () => {
    const { registerBackgroundSync } = await import('@/utils/backgroundSync');
    await registerBackgroundSync();
    
    const registration = await navigator.serviceWorker.ready;
    expect(registration.sync.register).toHaveBeenCalledWith('sync-pending-actions');
  });
});
```

---

## Best Practices

1. **Cache strategically** - Not everything needs to be cached offline
2. **Show offline indicators** - Be transparent about connection status
3. **Queue actions for sync** - Don't lose user actions when offline
4. **Test on real devices** - Emulators don't fully replicate PWA behavior
5. **Handle update gracefully** - Don't force reload during user activity
6. **Respect storage limits** - Clean up old cache entries
7. **Use HTTPS everywhere** - PWAs require secure contexts
8. **Optimize for mobile** - Touch targets, gestures, viewport
9. **Monitor service worker** - Log errors, track update success
10. **Progressive enhancement** - App should work without SW

---

## Native App Transition (Phase 10)

The PWA implementation in this document is designed to be **forward-compatible** with our Phase 10 native app strategy using Capacitor.

### What Changes in Phase 10

| Component | PWA (Current) | Native App (Phase 10) |
|-----------|---------------|----------------------|
| Push Notifications | Web Push API + VAPID | Capacitor Push Notifications (FCM/APNs) |
| Storage | IndexedDB + Cache API | Same + Capacitor Preferences |
| Camera | MediaDevices API | Capacitor Camera Plugin |
| Offline Detection | `navigator.onLine` | Same (works in Capacitor) |
| Background Sync | Service Worker | Same + Native Background Fetch |

### Code Patterns That Enable Easy Transition

The following patterns used in our PWA implementation are specifically chosen to make the Capacitor transition seamless:

#### 1. Platform-Agnostic Service Layer

```javascript
// services/api.js - Works unchanged in Capacitor
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

export default api;
```

#### 2. Abstracted Push Notification Service

```javascript
// services/pushNotifications.js
// This abstraction makes it easy to swap implementations

export async function requestNotificationPermission() {
  // Phase 7: Web Push
  if ('Notification' in window) {
    return Notification.requestPermission();
  }
  
  // Phase 10: Will add Capacitor check here
  // if (Capacitor.isNativePlatform()) {
  //   return PushNotifications.requestPermissions();
  // }
}
```

#### 3. Offline-First Data Hooks

```javascript
// hooks/useOfflineData.js
// IndexedDB storage works in both PWA and Capacitor
import { openDB } from 'idb';

// This code runs unchanged in Capacitor
export function useOfflineChores() {
  // ... implementation
}
```

### Preparing for Phase 10

When implementing PWA features, keep these guidelines in mind:

1. **Avoid Web-Only APIs Without Fallbacks**
   - Always check for API availability before use
   - Provide graceful degradation

2. **Use Environment Variables for Configuration**
   - Don't hardcode URLs or keys
   - Capacitor can use the same `.env` files

3. **Keep Push Notification Logic Isolated**
   - All notification code in `services/pushNotifications.js`
   - Easy to swap for Capacitor plugin in Phase 10

4. **Test Responsive Design on Multiple Screen Sizes**
   - Native apps will use the same responsive components
   - Test on 320px (small phones) to 428px (large phones)

### Migration Checklist for Phase 10

When we begin Phase 10 native app development:

- [ ] Install Capacitor core and plugins
- [ ] Run `npx cap init` to initialize projects
- [ ] Update push notification service for native
- [ ] Add native splash screens and icons
- [ ] Configure deep linking
- [ ] Test on iOS Simulator and Android Emulator
- [ ] Set up Firebase for FCM (Android + iOS)
- [ ] Configure APNs certificates (iOS)

For complete Phase 10 implementation details, see [MOBILE_APP_STRATEGY.md](./MOBILE_APP_STRATEGY.md).

---

## Troubleshooting

### Service Worker Not Updating

```javascript
// Force service worker update
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.getRegistrations().then((registrations) => {
    registrations.forEach((registration) => {
      registration.update();
    });
  });
}
```

### Clear All Caches

```javascript
// Clear all caches (useful for development)
if ('caches' in window) {
  caches.keys().then((names) => {
    names.forEach((name) => {
      caches.delete(name);
    });
  });
}
```

### Debug Service Worker

```javascript
// Log service worker state
navigator.serviceWorker.getRegistration().then((registration) => {
  if (registration) {
    console.log('Service Worker State:', {
      installing: !!registration.installing,
      waiting: !!registration.waiting,
      active: !!registration.active,
      scope: registration.scope,
    });
  }
});
```

---

*Last Updated: February 1, 2026*
