# Frontend State Management

## Overview

PicklesApp uses a lightweight state management approach built on React Context API and custom hooks. This architecture avoids the complexity of Redux while providing sufficient power for the application's needs.

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Context Providers](#context-providers)
3. [Custom Hooks](#custom-hooks)
4. [API Integration Patterns](#api-integration-patterns)
5. [Token Management](#token-management)
6. [Family Context Switching](#family-context-switching)
7. [Caching Strategies](#caching-strategies)
8. [Optimistic Updates](#optimistic-updates)

---

## Architecture Overview

### State Management Layers

```
┌────────────────────────────────────────────────┐
│          Global State (Context API)             │
│  - AuthContext (user, tokens, login/logout)    │
│  - FamilyContext (families, currentFamily)     │
│  - UserContext (user profile, ageBracket)      │
└────────────────────────────────────────────────┘
                      ↓
┌────────────────────────────────────────────────┐
│         Custom Hooks (Business Logic)          │
│  - useAuth, useFamily, useAgeBracket          │
│  - useChores, useAccounts, useTransactions    │
└────────────────────────────────────────────────┘
                      ↓
┌────────────────────────────────────────────────┐
│         API Services (Data Fetching)           │
│  - authService, choreService, etc.            │
│  - Axios instance with interceptors           │
└────────────────────────────────────────────────┘
                      ↓
┌────────────────────────────────────────────────┐
│         Local Component State                  │
│  - Form inputs, UI toggles, loading states    │
└────────────────────────────────────────────────┘
```

### When to Use Each Layer

**Global Context:**
- User authentication state
- Current family context
- User profile information
- App-wide settings

**Custom Hooks:**
- Data fetching with caching
- Business logic reuse
- Complex state operations
- Side effect management

**Local State:**
- Form inputs
- UI toggles (modals, dropdowns)
- Temporary data
- Component-specific state

---

## Context Providers

### AuthContext

Manages authentication state, tokens, and auth operations.

```jsx
// contexts/AuthContext.jsx
import { createContext, useState, useEffect, useCallback } from 'react';
import { authService } from '@/services/authService';
import { setAuthToken, clearAuthToken } from '@/services/api';

export const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [accessToken, setAccessToken] = useState(null);
  const [refreshToken, setRefreshToken] = useState(null);
  const [loading, setLoading] = useState(true);

  // Initialize auth state from localStorage
  useEffect(() => {
    const initAuth = async () => {
      const storedAccessToken = localStorage.getItem('accessToken');
      const storedRefreshToken = localStorage.getItem('refreshToken');

      if (storedAccessToken && storedRefreshToken) {
        try {
          setAuthToken(storedAccessToken);
          const userData = await authService.getMe();
          setUser(userData);
          setAccessToken(storedAccessToken);
          setRefreshToken(storedRefreshToken);
        } catch (error) {
          // Token invalid, try refresh
          if (storedRefreshToken) {
            try {
              const tokens = await authService.refreshToken(storedRefreshToken);
              setAuthToken(tokens.access_token);
              localStorage.setItem('accessToken', tokens.access_token);
              localStorage.setItem('refreshToken', tokens.refresh_token);
              
              const userData = await authService.getMe();
              setUser(userData);
              setAccessToken(tokens.access_token);
              setRefreshToken(tokens.refresh_token);
            } catch (refreshError) {
              // Refresh failed, logout
              logout();
            }
          }
        }
      }
      setLoading(false);
    };

    initAuth();
  }, []);

  const login = useCallback(async (email, password) => {
    const { user, access_token, refresh_token } = await authService.login(
      email,
      password
    );

    setUser(user);
    setAccessToken(access_token);
    setRefreshToken(refresh_token);
    
    // Persist tokens
    localStorage.setItem('accessToken', access_token);
    localStorage.setItem('refreshToken', refresh_token);
    setAuthToken(access_token);

    return user;
  }, []);

  const loginWithGoogle = useCallback(async (credential) => {
    const { user, access_token, refresh_token } = await authService.loginWithGoogle(
      credential
    );

    setUser(user);
    setAccessToken(access_token);
    setRefreshToken(refresh_token);
    
    localStorage.setItem('accessToken', access_token);
    localStorage.setItem('refreshToken', refresh_token);
    setAuthToken(access_token);

    return user;
  }, []);

  const register = useCallback(async (userData) => {
    const { user, access_token, refresh_token } = await authService.register(
      userData
    );

    setUser(user);
    setAccessToken(access_token);
    setRefreshToken(refresh_token);
    
    localStorage.setItem('accessToken', access_token);
    localStorage.setItem('refreshToken', refresh_token);
    setAuthToken(access_token);

    return user;
  }, []);

  const logout = useCallback(() => {
    setUser(null);
    setAccessToken(null);
    setRefreshToken(null);
    localStorage.removeItem('accessToken');
    localStorage.removeItem('refreshToken');
    clearAuthToken();
  }, []);

  const refreshAccessToken = useCallback(async () => {
    if (!refreshToken) {
      throw new Error('No refresh token available');
    }

    const tokens = await authService.refreshToken(refreshToken);
    
    setAccessToken(tokens.access_token);
    setRefreshToken(tokens.refresh_token);
    localStorage.setItem('accessToken', tokens.access_token);
    localStorage.setItem('refreshToken', tokens.refresh_token);
    setAuthToken(tokens.access_token);

    return tokens.access_token;
  }, [refreshToken]);

  const updateUser = useCallback((userData) => {
    setUser(prev => ({ ...prev, ...userData }));
  }, []);

  const value = {
    user,
    accessToken,
    refreshToken,
    loading,
    login,
    loginWithGoogle,
    register,
    logout,
    refreshAccessToken,
    updateUser,
    isAuthenticated: !!user,
  };

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}
```

### FamilyContext

Manages family data and context switching for multi-household support.

```jsx
// contexts/FamilyContext.jsx
import { createContext, useState, useEffect, useCallback } from 'react';
import { useAuth } from '@/hooks/useAuth';
import { familyService } from '@/services/familyService';

export const FamilyContext = createContext(null);

export function FamilyProvider({ children }) {
  const { user, isAuthenticated } = useAuth();
  const [families, setFamilies] = useState([]);
  const [currentFamily, setCurrentFamily] = useState(null);
  const [loading, setLoading] = useState(true);

  // Fetch user's families
  useEffect(() => {
    if (!isAuthenticated || !user) {
      setLoading(false);
      return;
    }

    const fetchFamilies = async () => {
      try {
        const data = await familyService.getFamilies();
        setFamilies(data);

        // Set current family from localStorage or default to first
        const storedFamilyId = localStorage.getItem('currentFamilyId');
        const family = storedFamilyId
          ? data.find(f => f.id === storedFamilyId)
          : data[0];

        if (family) {
          setCurrentFamily(family);
        }
      } catch (error) {
        console.error('Failed to fetch families:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchFamilies();
  }, [isAuthenticated, user]);

  const switchFamily = useCallback((familyId) => {
    const family = families.find(f => f.id === familyId);
    if (family) {
      setCurrentFamily(family);
      localStorage.setItem('currentFamilyId', familyId);
    }
  }, [families]);

  const refreshFamilies = useCallback(async () => {
    const data = await familyService.getFamilies();
    setFamilies(data);

    // Update current family if it changed
    if (currentFamily) {
      const updated = data.find(f => f.id === currentFamily.id);
      if (updated) {
        setCurrentFamily(updated);
      }
    }
  }, [currentFamily]);

  const createFamily = useCallback(async (familyData) => {
    const newFamily = await familyService.createFamily(familyData);
    setFamilies(prev => [...prev, newFamily]);
    return newFamily;
  }, []);

  const updateFamily = useCallback(async (familyId, updates) => {
    const updated = await familyService.updateFamily(familyId, updates);
    setFamilies(prev =>
      prev.map(f => (f.id === familyId ? updated : f))
    );
    if (currentFamily?.id === familyId) {
      setCurrentFamily(updated);
    }
    return updated;
  }, [currentFamily]);

  const value = {
    families,
    currentFamily,
    loading,
    switchFamily,
    refreshFamilies,
    createFamily,
    updateFamily,
    hasMultipleFamilies: families.length > 1,
  };

  return (
    <FamilyContext.Provider value={value}>
      {children}
    </FamilyContext.Provider>
  );
}
```

### UserContext

Manages user profile and age bracket information.

```jsx
// contexts/UserContext.jsx
import { createContext, useMemo } from 'react';
import { useAuth } from '@/hooks/useAuth';
import { useFamily } from '@/hooks/useFamily';
import { calculateAgeBracket } from '@/utils/ageBracket';

export const UserContext = createContext(null);

export function UserProvider({ children }) {
  const { user, updateUser } = useAuth();
  const { currentFamily } = useFamily();

  // Calculate age bracket
  const ageBracket = useMemo(() => {
    if (!user || user.role === 'PARENT') {
      return 'adult';
    }

    // Check for parent override in current family
    if (currentFamily) {
      const membership = currentFamily.members.find(m => m.user_id === user.id);
      if (membership?.age_bracket_override) {
        return membership.age_bracket_override;
      }
    }

    // Calculate from birthdate
    return calculateAgeBracket(user.birthdate);
  }, [user, currentFamily]);

  const value = {
    user,
    ageBracket,
    updateUser,
    isParent: user?.role === 'PARENT',
    isChild: user?.role === 'CHILD',
  };

  return (
    <UserContext.Provider value={value}>
      {children}
    </UserContext.Provider>
  );
}
```

### Context Composition

Wrap app with all providers:

```jsx
// App.jsx
import { AuthProvider } from '@/contexts/AuthContext';
import { FamilyProvider } from '@/contexts/FamilyContext';
import { UserProvider } from '@/contexts/UserContext';

export function App() {
  return (
    <AuthProvider>
      <FamilyProvider>
        <UserProvider>
          <Router>
            {/* Routes */}
          </Router>
        </UserProvider>
      </FamilyProvider>
    </AuthProvider>
  );
}
```

---

## Custom Hooks

### useAuth

Access authentication context with convenience methods.

```jsx
// hooks/useAuth.js
import { useContext } from 'react';
import { AuthContext } from '@/contexts/AuthContext';

export function useAuth() {
  const context = useContext(AuthContext);
  
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  
  return context;
}
```

### useFamily

Access family context and operations.

```jsx
// hooks/useFamily.js
import { useContext } from 'react';
import { FamilyContext } from '@/contexts/FamilyContext';

export function useFamily() {
  const context = useContext(FamilyContext);
  
  if (!context) {
    throw new Error('useFamily must be used within FamilyProvider');
  }
  
  return context;
}
```

### useUser

Access user context and age bracket.

```jsx
// hooks/useUser.js
import { useContext } from 'react';
import { UserContext } from '@/contexts/UserContext';

export function useUser() {
  const context = useContext(UserContext);
  
  if (!context) {
    throw new Error('useUser must be used within UserProvider');
  }
  
  return context;
}
```

### useAgeBracket

Dedicated hook for age bracket logic.

```jsx
// hooks/useAgeBracket.js
import { useMemo } from 'react';
import { useUser } from './useUser';
import { FEATURE_ACCESS } from '@/utils/featureAccess';

export function useAgeBracket() {
  const { ageBracket, isParent } = useUser();

  const hasFeatureAccess = useMemo(() => {
    return (feature) => {
      if (isParent) return true;
      return FEATURE_ACCESS[ageBracket]?.[feature] ?? false;
    };
  }, [ageBracket, isParent]);

  const getTypographyScale = useMemo(() => {
    const scales = {
      young: {
        base: 'text-lg',
        large: 'text-3xl',
        small: 'text-base',
      },
      tween: {
        base: 'text-base',
        large: 'text-2xl',
        small: 'text-sm',
      },
      teen: {
        base: 'text-sm',
        large: 'text-xl',
        small: 'text-xs',
      },
      adult: {
        base: 'text-sm',
        large: 'text-xl',
        small: 'text-xs',
      },
    };
    return scales[ageBracket] || scales.tween;
  }, [ageBracket]);

  return {
    ageBracket,
    hasFeatureAccess,
    getTypographyScale,
    isYoung: ageBracket === 'young',
    isTween: ageBracket === 'tween',
    isTeen: ageBracket === 'teen',
  };
}
```

### useChores

Fetch and manage chores with caching.

```jsx
// hooks/useChores.js
import { useState, useEffect, useCallback } from 'react';
import { useFamily } from './useFamily';
import { useUser } from './useUser';
import { choreService } from '@/services/choreService';

export function useChores(filters = {}) {
  const { currentFamily } = useFamily();
  const { user } = useUser();
  const [chores, setChores] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchChores = useCallback(async () => {
    if (!currentFamily) return;

    try {
      setLoading(true);
      setError(null);

      const params = {
        ...filters,
        family_id: currentFamily.id,
      };

      // Auto-filter for children
      if (user.role === 'CHILD') {
        params.assigned_to_id = user.id;
      }

      const data = await choreService.getChores(params);
      setChores(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [currentFamily, user, filters]);

  useEffect(() => {
    fetchChores();
  }, [fetchChores]);

  const completeChore = useCallback(async (choreId, evidence = null) => {
    await choreService.completeChore(choreId, evidence);
    
    // Optimistic update
    setChores(prev =>
      prev.map(chore =>
        chore.id === choreId
          ? { ...chore, status: 'PENDING_APPROVAL' }
          : chore
      )
    );

    // Refetch for server state
    await fetchChores();
  }, [fetchChores]);

  const approveChore = useCallback(async (choreId) => {
    await choreService.approveChore(choreId);
    
    // Optimistic update
    setChores(prev =>
      prev.map(chore =>
        chore.id === choreId
          ? { ...chore, status: 'COMPLETED' }
          : chore
      )
    );

    await fetchChores();
  }, [fetchChores]);

  const rejectChore = useCallback(async (choreId, reason) => {
    await choreService.rejectChore(choreId, reason);
    
    setChores(prev =>
      prev.map(chore =>
        chore.id === choreId
          ? { ...chore, status: 'PENDING' }
          : chore
      )
    );

    await fetchChores();
  }, [fetchChores]);

  return {
    chores,
    loading,
    error,
    refresh: fetchChores,
    completeChore,
    approveChore,
    rejectChore,
  };
}
```

### useAccounts

Fetch and manage accounts with caching.

```jsx
// hooks/useAccounts.js
import { useState, useEffect, useCallback } from 'react';
import { useFamily } from './useFamily';
import { useUser } from './useUser';
import { accountService } from '@/services/accountService';

export function useAccounts() {
  const { currentFamily } = useFamily();
  const { user } = useUser();
  const [accounts, setAccounts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchAccounts = useCallback(async () => {
    if (!currentFamily) return;

    try {
      setLoading(true);
      setError(null);

      const params = { family_id: currentFamily.id };
      
      // Children only see their own accounts
      if (user.role === 'CHILD') {
        params.user_id = user.id;
      }

      const data = await accountService.getAccounts(params);
      setAccounts(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [currentFamily, user]);

  useEffect(() => {
    fetchAccounts();
  }, [fetchAccounts]);

  const totalBalance = accounts.reduce(
    (sum, account) => sum + account.balance,
    0
  );

  const getAccountsByType = useCallback((type) => {
    return accounts.filter(account => account.account_type === type);
  }, [accounts]);

  return {
    accounts,
    loading,
    error,
    refresh: fetchAccounts,
    totalBalance,
    getAccountsByType,
  };
}
```

### useTransactions

Fetch transactions with pagination and filters.

```jsx
// hooks/useTransactions.js
import { useState, useEffect, useCallback } from 'react';
import { useFamily } from './useFamily';
import { transactionService } from '@/services/transactionService';

export function useTransactions(accountId = null, filters = {}) {
  const { currentFamily } = useFamily();
  const [transactions, setTransactions] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [hasMore, setHasMore] = useState(false);
  const [page, setPage] = useState(1);

  const fetchTransactions = useCallback(async (pageNum = 1) => {
    if (!currentFamily) return;

    try {
      setLoading(true);
      setError(null);

      const params = {
        family_id: currentFamily.id,
        account_id: accountId,
        page: pageNum,
        limit: 50,
        ...filters,
      };

      const data = await transactionService.getTransactions(params);
      
      if (pageNum === 1) {
        setTransactions(data.transactions);
      } else {
        setTransactions(prev => [...prev, ...data.transactions]);
      }
      
      setHasMore(data.has_more);
      setPage(pageNum);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [currentFamily, accountId, filters]);

  useEffect(() => {
    fetchTransactions(1);
  }, [fetchTransactions]);

  const loadMore = useCallback(() => {
    if (!loading && hasMore) {
      fetchTransactions(page + 1);
    }
  }, [loading, hasMore, page, fetchTransactions]);

  const createTransaction = useCallback(async (transactionData) => {
    const newTransaction = await transactionService.createTransaction(
      transactionData
    );
    
    // Optimistic update
    setTransactions(prev => [newTransaction, ...prev]);
    
    return newTransaction;
  }, []);

  return {
    transactions,
    loading,
    error,
    hasMore,
    loadMore,
    refresh: () => fetchTransactions(1),
    createTransaction,
  };
}
```

---

## API Integration Patterns

### Axios Instance with Interceptors

```jsx
// services/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor - Add auth token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Response interceptor - Handle token refresh
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Token expired, try refresh
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const { data } = await axios.post(
          `${import.meta.env.VITE_API_BASE_URL}/auth/refresh`,
          { refresh_token: refreshToken }
        );

        localStorage.setItem('accessToken', data.access_token);
        localStorage.setItem('refreshToken', data.refresh_token);
        
        // Retry original request with new token
        originalRequest.headers.Authorization = `Bearer ${data.access_token}`;
        return api(originalRequest);
      } catch (refreshError) {
        // Refresh failed, logout user
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);

export default api;

// Helper functions
export function setAuthToken(token) {
  api.defaults.headers.common['Authorization'] = `Bearer ${token}`;
}

export function clearAuthToken() {
  delete api.defaults.headers.common['Authorization'];
}
```

### Service Layer Pattern

```jsx
// services/choreService.js
import api from './api';

export const choreService = {
  // GET /chores
  getChores: async (params) => {
    const { data } = await api.get('/chores', { params });
    return data;
  },

  // GET /chores/:id
  getChore: async (id) => {
    const { data } = await api.get(`/chores/${id}`);
    return data;
  },

  // POST /chores
  createChore: async (choreData) => {
    const { data } = await api.post('/chores', choreData);
    return data;
  },

  // PATCH /chores/:id
  updateChore: async (id, updates) => {
    const { data } = await api.patch(`/chores/${id}`, updates);
    return data;
  },

  // DELETE /chores/:id
  deleteChore: async (id) => {
    await api.delete(`/chores/${id}`);
  },

  // POST /chores/:id/complete
  completeChore: async (id, evidence = null) => {
    const { data } = await api.post(`/chores/${id}/complete`, { evidence });
    return data;
  },

  // POST /chores/:id/approve
  approveChore: async (id) => {
    const { data } = await api.post(`/chores/${id}/approve`);
    return data;
  },

  // POST /chores/:id/reject
  rejectChore: async (id, reason) => {
    const { data } = await api.post(`/chores/${id}/reject`, { reason });
    return data;
  },
};
```

---

## Token Management

### Token Storage Strategy

```jsx
// utils/tokenStorage.js

// Store tokens securely
export function storeTokens(accessToken, refreshToken) {
  localStorage.setItem('accessToken', accessToken);
  localStorage.setItem('refreshToken', refreshToken);
}

// Retrieve tokens
export function getTokens() {
  return {
    accessToken: localStorage.getItem('accessToken'),
    refreshToken: localStorage.getItem('refreshToken'),
  };
}

// Clear tokens on logout
export function clearTokens() {
  localStorage.removeItem('accessToken');
  localStorage.removeItem('refreshToken');
}

// Check if token is expired
export function isTokenExpired(token) {
  if (!token) return true;
  
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.exp * 1000 < Date.now();
  } catch (error) {
    return true;
  }
}
```

### Automatic Token Refresh

Handled by axios interceptor (see above), but can also be preemptive:

```jsx
// utils/tokenRefresh.js
import { authService } from '@/services/authService';
import { isTokenExpired } from './tokenStorage';

let refreshPromise = null;

export async function ensureValidToken() {
  const { accessToken, refreshToken } = getTokens();

  // No tokens, user needs to login
  if (!accessToken || !refreshToken) {
    throw new Error('No tokens available');
  }

  // Access token still valid
  if (!isTokenExpired(accessToken)) {
    return accessToken;
  }

  // Already refreshing, return existing promise
  if (refreshPromise) {
    return refreshPromise;
  }

  // Refresh token
  refreshPromise = authService
    .refreshToken(refreshToken)
    .then(tokens => {
      storeTokens(tokens.access_token, tokens.refresh_token);
      refreshPromise = null;
      return tokens.access_token;
    })
    .catch(error => {
      refreshPromise = null;
      clearTokens();
      throw error;
    });

  return refreshPromise;
}
```

---

## Family Context Switching

### Multi-Household Support

For children with divorced/separated parents:

```jsx
// components/layout/FamilySwitcher.jsx
import { useFamily } from '@/hooks/useFamily';
import { Select } from '@/components/shared/Select';

export function FamilySwitcher() {
  const { families, currentFamily, switchFamily } = useFamily();

  if (families.length <= 1) {
    return null;
  }

  return (
    <div className="bg-blue-50 border-b border-blue-200 p-3">
      <div className="container mx-auto flex items-center gap-3">
        <span className="text-sm font-medium text-blue-900">
          Viewing:
        </span>
        <Select
          value={currentFamily?.id}
          onChange={(e) => switchFamily(e.target.value)}
          className="w-64"
        >
          {families.map(family => (
            <option key={family.id} value={family.id}>
              {family.name}
            </option>
          ))}
        </Select>
      </div>
    </div>
  );
}
```

### Context Isolation

All API requests automatically include current family context:

```jsx
// services/api.js interceptor addition
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    const familyId = localStorage.getItem('currentFamilyId');
    
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    
    // Add family context to all requests
    if (familyId && !config.params?.family_id) {
      config.params = {
        ...config.params,
        family_id: familyId,
      };
    }
    
    return config;
  }
);
```

---

## Caching Strategies

### In-Memory Cache

Simple caching for frequently accessed data:

```jsx
// utils/cache.js
class SimpleCache {
  constructor() {
    this.cache = new Map();
    this.timestamps = new Map();
  }

  set(key, value, ttl = 60000) {
    this.cache.set(key, value);
    this.timestamps.set(key, Date.now() + ttl);
  }

  get(key) {
    const timestamp = this.timestamps.get(key);
    
    if (!timestamp || Date.now() > timestamp) {
      this.cache.delete(key);
      this.timestamps.delete(key);
      return null;
    }
    
    return this.cache.get(key);
  }

  clear() {
    this.cache.clear();
    this.timestamps.clear();
  }

  delete(key) {
    this.cache.delete(key);
    this.timestamps.delete(key);
  }
}

export const cache = new SimpleCache();
```

### Hook with Cache

```jsx
// hooks/useAccountsWithCache.js
import { useState, useEffect } from 'react';
import { cache } from '@/utils/cache';
import { accountService } from '@/services/accountService';

export function useAccountsWithCache(familyId) {
  const [accounts, setAccounts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const cacheKey = `accounts_${familyId}`;
    
    // Check cache first
    const cached = cache.get(cacheKey);
    if (cached) {
      setAccounts(cached);
      setLoading(false);
      return;
    }

    // Fetch and cache
    accountService.getAccounts({ family_id: familyId })
      .then(data => {
        setAccounts(data);
        cache.set(cacheKey, data, 60000); // 1 minute TTL
      })
      .finally(() => setLoading(false));
  }, [familyId]);

  return { accounts, loading };
}
```

### Cache Invalidation

```jsx
// On data mutations, invalidate cache
const updateAccount = async (accountId, updates) => {
  await accountService.updateAccount(accountId, updates);
  
  // Invalidate cache
  cache.delete(`accounts_${familyId}`);
  
  // Refetch
  fetchAccounts();
};
```

---

## Optimistic Updates

### Transaction Creation with Rollback

```jsx
// hooks/useOptimisticTransactions.js
import { useState, useCallback } from 'react';
import { transactionService } from '@/services/transactionService';
import toast from 'react-hot-toast';

export function useOptimisticTransactions() {
  const [transactions, setTransactions] = useState([]);

  const createTransaction = useCallback(async (transactionData) => {
    // Generate temporary ID
    const tempId = `temp_${Date.now()}`;
    const optimisticTransaction = {
      ...transactionData,
      id: tempId,
      created_at: new Date().toISOString(),
      status: 'pending',
    };

    // Optimistically add to list
    setTransactions(prev => [optimisticTransaction, ...prev]);

    try {
      // Make API call
      const savedTransaction = await transactionService.createTransaction(
        transactionData
      );

      // Replace temp with real transaction
      setTransactions(prev =>
        prev.map(tx => (tx.id === tempId ? savedTransaction : tx))
      );

      toast.success('Transaction created!');
      return savedTransaction;
    } catch (error) {
      // Rollback on error
      setTransactions(prev => prev.filter(tx => tx.id !== tempId));
      toast.error('Failed to create transaction');
      throw error;
    }
  }, []);

  return {
    transactions,
    createTransaction,
  };
}
```

### Chore Completion with Optimistic UI

```jsx
// components/chores/ChoreCard.jsx
export function ChoreCard({ chore }) {
  const [localStatus, setLocalStatus] = useState(chore.status);
  const [completing, setCompleting] = useState(false);

  const handleComplete = async () => {
    // Optimistic update
    setLocalStatus('PENDING_APPROVAL');
    setCompleting(true);

    try {
      await choreService.completeChore(chore.id);
      toast.success('Chore marked complete!');
    } catch (error) {
      // Rollback on error
      setLocalStatus(chore.status);
      toast.error('Failed to complete chore');
    } finally {
      setCompleting(false);
    }
  };

  return (
    <Card>
      <h3>{chore.title}</h3>
      <StatusBadge status={localStatus} />
      
      {localStatus === 'PENDING' && (
        <Button onClick={handleComplete} disabled={completing}>
          {completing ? 'Completing...' : 'Mark Complete'}
        </Button>
      )}
    </Card>
  );
}
```

---

## Best Practices

1. **Keep global state minimal** - Only store truly global data
2. **Use context for cross-cutting concerns** - Auth, theming, family context
3. **Prefer hooks for data fetching** - Encapsulate logic, make reusable
4. **Implement proper error handling** - Don't let errors crash the app
5. **Cache wisely** - Balance freshness vs performance
6. **Optimistic updates for better UX** - With proper rollback
7. **Token refresh transparently** - Users shouldn't see expired sessions
8. **Invalidate cache on mutations** - Keep data fresh
9. **Use loading states properly** - Show users what's happening
10. **Test state management** - Mock providers in tests

---

*Last Updated: January 29, 2026*
