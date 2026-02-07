# Frontend Auth Integration

Complete patterns for integrating kiwi-user authentication in a Next.js/React frontend.

## Core Auth Module

The auth module (`lib/auth.ts`) handles all authentication logic. It communicates with kiwi-user service and manages token lifecycle.

### Configuration

```typescript
const KIWI_USER_API_BASE_URL = process.env.NEXT_PUBLIC_KIWI_USER_API_BASE_URL!;
const APPLICATION_NAME = process.env.NEXT_PUBLIC_APPLICATION_NAME!;
```

### Types

```typescript
interface TokenInfo {
  accessToken: string;
  refreshToken: string;
  accessTokenExpiresAt: number;  // unix seconds
  refreshTokenExpiresAt: number; // unix seconds
  userId: string;
  deviceType: string;
  deviceId: string;
}

interface LoginResponse {
  refresh_token: string;
  refresh_token_expires_at: number;
  access_token: string;
  access_token_expires_at: number;
  type: string;
  device_type: string;
  device_id: string;
  user_id: string;
}

interface UserInfo {
  id: string;
  application: string;
  name: string;
  avatar: string;
  phone?: string;
  personal_role?: string;
  personal_scopes?: string[];
  current_org_id?: string;
  orgs?: OrgInfo[];
  department?: string;
}

interface Device {
  device_type: string;
  device_id: string;
}
```

### Storage Keys

All tokens and auth metadata are stored in `localStorage`:

```typescript
const STORAGE_KEYS = {
  ACCESS_TOKEN: "kiwi_access_token",
  REFRESH_TOKEN: "kiwi_refresh_token",
  ACCESS_TOKEN_EXPIRES_AT: "kiwi_access_token_expires_at",
  REFRESH_TOKEN_EXPIRES_AT: "kiwi_refresh_token_expires_at",
  USER_ID: "kiwi_user_id",
  DEVICE_TYPE: "kiwi_device_type",
  DEVICE_ID: "kiwi_device_id",
} as const;
```

### Saving Tokens from Login Response

```typescript
function saveTokenInfo(response: LoginResponse): void {
  localStorage.setItem("kiwi_access_token", response.access_token);
  localStorage.setItem("kiwi_refresh_token", response.refresh_token);
  localStorage.setItem("kiwi_access_token_expires_at", String(response.access_token_expires_at));
  localStorage.setItem("kiwi_refresh_token_expires_at", String(response.refresh_token_expires_at));
  localStorage.setItem("kiwi_user_id", response.user_id);
  localStorage.setItem("kiwi_device_type", response.device_type);
  localStorage.setItem("kiwi_device_id", response.device_id);
}
```

### Clearing Auth Data

```typescript
function clearAuthData(): void {
  localStorage.removeItem("kiwi_access_token");
  localStorage.removeItem("kiwi_refresh_token");
  localStorage.removeItem("kiwi_access_token_expires_at");
  localStorage.removeItem("kiwi_refresh_token_expires_at");
  localStorage.removeItem("kiwi_user_id");
  localStorage.removeItem("kiwi_device_type");
  localStorage.removeItem("kiwi_device_id");
}
```

## Login Flows

### Google OAuth (Web) - Full Flow

This is the most common login method. It uses Google's OAuth 2.0 authorization code flow.

**Step 1: Redirect to Google**

```typescript
function getGoogleLoginUrl(): string {
  const state = crypto.randomUUID(); // CSRF protection
  sessionStorage.setItem("google_oauth_state", state);

  const params = new URLSearchParams({
    client_id: process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID!,
    redirect_uri: `${window.location.origin}/auth/google/callback`,
    response_type: "code",
    scope: "openid email profile",
    state: state,
    access_type: "offline",
    prompt: "consent",
  });

  return `https://accounts.google.com/o/oauth2/v2/auth?${params.toString()}`;
}

// Usage: redirect user to login
window.location.href = getGoogleLoginUrl();
```

**Step 2: Handle Callback**

The callback page (`/auth/google/callback`) extracts the authorization code and exchanges it for tokens:

```typescript
// app/auth/google/callback/GoogleOAuthCallbackClient.tsx
"use client";

import { useSearchParams, useRouter } from "next/navigation";
import { useEffect, useState } from "react";
import { handleGoogleOAuthCallback } from "@/lib/auth";

export default function GoogleOAuthCallbackClient() {
  const searchParams = useSearchParams();
  const router = useRouter();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const code = searchParams.get("code");
    const state = searchParams.get("state");

    if (!code || !state) {
      setError("Missing code or state parameter");
      return;
    }

    // Prevent duplicate processing (e.g., from duplicate tabs)
    const processingKey = `google_oauth_processing_${state}`;
    if (sessionStorage.getItem(processingKey)) return;
    sessionStorage.setItem(processingKey, "true");

    handleGoogleOAuthCallback(code, state)
      .then(() => {
        router.push("/");
      })
      .catch((err) => {
        setError(err.message);
        sessionStorage.removeItem(processingKey);
      });
  }, [searchParams, router]);

  if (error) return <div>Login failed: {error}</div>;
  return <div>Logging in...</div>;
}
```

**Step 3: Exchange Code for Tokens**

```typescript
async function handleGoogleOAuthCallback(code: string, state: string): Promise<void> {
  // Verify CSRF state
  const savedState = sessionStorage.getItem("google_oauth_state");
  if (state !== savedState) {
    throw new Error("Invalid OAuth state - possible CSRF attack");
  }
  sessionStorage.removeItem("google_oauth_state");

  const response = await fetch(`${KIWI_USER_API_BASE_URL}/v1/login/google/web`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      application_name: APPLICATION_NAME,
      code: code,
      redirect_uri: `${window.location.origin}/auth/google/callback`,
      device: getDevice(),
    }),
  });

  if (!response.ok) throw new Error("Google login failed");

  const result = await response.json();
  const loginResponse: LoginResponse = result.data;
  saveTokenInfo(loginResponse);
}
```

### Google Login with ID Token (Alternative)

If you already have a Google ID token (e.g., from Google One Tap):

```typescript
async function loginWithGoogle(idToken: string): Promise<void> {
  const response = await fetch(`${KIWI_USER_API_BASE_URL}/v1/login/google/web`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      application_name: APPLICATION_NAME,
      id_token: idToken,
      device: getDevice(),
    }),
  });

  if (!response.ok) throw new Error("Google login failed");

  const result = await response.json();
  saveTokenInfo(result.data);
}
```

### Device Info Helper

```typescript
function getDevice(): Device {
  let deviceId = localStorage.getItem("kiwi_device_id");
  if (!deviceId) {
    deviceId = crypto.randomUUID();
    localStorage.setItem("kiwi_device_id", deviceId);
  }
  return {
    device_type: "web",
    device_id: deviceId,
  };
}
```

## Token Refresh

### Checking Expiration

```typescript
function isAccessTokenExpired(): boolean {
  const expiresAt = localStorage.getItem("kiwi_access_token_expires_at");
  if (!expiresAt) return true;
  const bufferSeconds = 60; // Refresh 1 minute before actual expiry
  return Date.now() / 1000 >= parseInt(expiresAt) - bufferSeconds;
}

function isRefreshTokenExpired(): boolean {
  const expiresAt = localStorage.getItem("kiwi_refresh_token_expires_at");
  if (!expiresAt) return true;
  return Date.now() / 1000 >= parseInt(expiresAt);
}

function isAuthenticated(): boolean {
  return !isRefreshTokenExpired();
}
```

### Refreshing Tokens

```typescript
async function refreshAccessToken(): Promise<boolean> {
  const userId = localStorage.getItem("kiwi_user_id");
  const refreshToken = localStorage.getItem("kiwi_refresh_token");

  if (!userId || !refreshToken || isRefreshTokenExpired()) {
    clearAuthData();
    return false;
  }

  try {
    const response = await fetch(`${KIWI_USER_API_BASE_URL}/v1/token/refresh`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        user_id: userId,
        refresh_token: refreshToken,
        device: getDevice(),
      }),
    });

    if (!response.ok) {
      clearAuthData();
      return false;
    }

    const result = await response.json();
    saveTokenInfo(result.data); // Both tokens are replaced
    return true;
  } catch {
    return false;
  }
}
```

### Transparent Token Access

This is the primary function all API calls should use:

```typescript
async function getAccessToken(): Promise<string | null> {
  const token = localStorage.getItem("kiwi_access_token");
  if (!token) return null;

  if (!isAccessTokenExpired()) {
    return token;
  }

  // Token expired or about to expire - refresh
  const refreshed = await refreshAccessToken();
  if (!refreshed) return null;

  return localStorage.getItem("kiwi_access_token");
}
```

## API Client

### Standard Request Helper

```typescript
const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL!; // Your backend, NOT kiwi-user

async function apiRequest<T>(
  path: string,
  options: RequestInit = {}
): Promise<T> {
  const token = await getAccessToken();
  if (!token) {
    window.location.href = "/";
    throw new Error("Not authenticated");
  }

  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
      ...options.headers,
    },
  });

  if (response.status === 401) {
    // Token was rejected - try refresh once
    const refreshed = await refreshAccessToken();
    if (refreshed) {
      const newToken = localStorage.getItem("kiwi_access_token");
      const retryResponse = await fetch(`${API_BASE_URL}${path}`, {
        ...options,
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${newToken}`,
          ...options.headers,
        },
      });
      if (retryResponse.ok) {
        const data = await retryResponse.json();
        return data.data as T;
      }
    }
    // Refresh failed or retry failed
    clearAuthData();
    window.location.href = "/";
    throw new Error("Unauthorized");
  }

  if (!response.ok) {
    throw new Error(`API error: ${response.status}`);
  }

  const data = await response.json();
  return data.data as T;
}
```

### SSE Streaming Request

For streaming responses (e.g., chat), handle token manually since `EventSource` does not support custom headers:

```typescript
async function streamRequest(
  path: string,
  body: any,
  onChunk: (data: string) => void
): Promise<void> {
  const token = await getAccessToken();
  if (!token) throw new Error("Not authenticated");

  const response = await fetch(`${API_BASE_URL}${path}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 401) {
    // Retry with refreshed token
    const refreshed = await refreshAccessToken();
    if (!refreshed) {
      clearAuthData();
      throw new Error("Unauthorized");
    }
    // Recursive retry with new token
    return streamRequest(path, body, onChunk);
  }

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const text = decoder.decode(value, { stream: true });
    // Parse SSE format: "data: {...}\n\n"
    const lines = text.split("\n");
    for (const line of lines) {
      if (line.startsWith("data: ")) {
        onChunk(line.slice(6));
      }
    }
  }
}
```

## User Info

### Fetching with Deduplication and Cache

```typescript
let userInfoPromise: Promise<UserInfo> | null = null;
let cachedUserInfo: { data: UserInfo; timestamp: number } | null = null;
const USER_INFO_CACHE_DURATION = 5000; // 5 seconds

async function getUserInfo(): Promise<UserInfo> {
  // Return cached if fresh
  if (cachedUserInfo && Date.now() - cachedUserInfo.timestamp < USER_INFO_CACHE_DURATION) {
    return cachedUserInfo.data;
  }

  // Deduplicate concurrent requests
  if (userInfoPromise) return userInfoPromise;

  userInfoPromise = (async () => {
    const token = await getAccessToken();
    if (!token) throw new Error("Not authenticated");

    const response = await fetch(`${KIWI_USER_API_BASE_URL}/v1/user/info`, {
      headers: { Authorization: `Bearer ${token}` },
    });

    if (response.status === 401) {
      // Retry once after refresh
      const refreshed = await refreshAccessToken();
      if (!refreshed) throw new Error("Unauthorized");

      const newToken = localStorage.getItem("kiwi_access_token");
      const retryResponse = await fetch(`${KIWI_USER_API_BASE_URL}/v1/user/info`, {
        headers: { Authorization: `Bearer ${newToken}` },
      });
      if (!retryResponse.ok) throw new Error("Failed to get user info");
      const retryData = await retryResponse.json();
      return retryData.data as UserInfo;
    }

    if (!response.ok) throw new Error("Failed to get user info");
    const data = await response.json();
    return data.data as UserInfo;
  })();

  try {
    const info = await userInfoPromise;
    cachedUserInfo = { data: info, timestamp: Date.now() };
    return info;
  } finally {
    userInfoPromise = null;
  }
}
```

## React Auth Hook

### useAuth Hook

```typescript
// lib/useAuth.ts
import { useState, useEffect } from "react";
import {
  isAuthenticated as checkAuth,
  getUserInfo,
  logout as authLogout,
} from "./auth";

interface UseAuthReturn {
  isAuthenticated: boolean;
  isLoading: boolean;
  user: UserInfo | null;
  error: Error | null;
  logout: () => Promise<void>;
  refreshUserInfo: () => Promise<void>;
}

export function useAuth(): UseAuthReturn {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const [user, setUser] = useState<UserInfo | null>(null);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    async function init() {
      try {
        const authed = checkAuth();
        setIsAuthenticated(authed);
        if (authed) {
          const userInfo = await getUserInfo();
          setUser(userInfo);
        }
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    }
    init();
  }, []);

  const logout = async () => {
    await authLogout();
    setIsAuthenticated(false);
    setUser(null);
  };

  const refreshUserInfo = async () => {
    const userInfo = await getUserInfo();
    setUser(userInfo);
  };

  return { isAuthenticated, isLoading, user, error, logout, refreshUserInfo };
}
```

### AuthContext Provider

For sharing auth state across the component tree:

```typescript
// lib/AuthContext.tsx
"use client";

import { createContext, useContext, ReactNode } from "react";
import { useAuth } from "./useAuth";

interface AuthContextType {
  isAuthenticated: boolean;
  isLoading: boolean;
  user: UserInfo | null;
  error: Error | null;
  logout: () => Promise<void>;
  refreshUserInfo: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const auth = useAuth();

  return <AuthContext.Provider value={auth}>{children}</AuthContext.Provider>;
}

export function useAuthContext(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuthContext must be used within an AuthProvider");
  }
  return context;
}
```

Usage in layout:

```typescript
// app/layout.tsx
import { AuthProvider } from "@/lib/AuthContext";

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html>
      <body>
        <AuthProvider>{children}</AuthProvider>
      </body>
    </html>
  );
}
```

### Tab Focus Refresh

Refresh auth state when user returns to the tab (handles token expiry while away):

```typescript
// Inside AuthProvider
useEffect(() => {
  let lastCheck = 0;
  const THROTTLE_MS = 30000; // 30 seconds

  function handleVisibilityChange() {
    if (document.visibilityState !== "visible") return;
    if (Date.now() - lastCheck < THROTTLE_MS) return;
    lastCheck = Date.now();
    refreshUserInfo();
  }

  document.addEventListener("visibilitychange", handleVisibilityChange);
  return () => document.removeEventListener("visibilitychange", handleVisibilityChange);
}, []);
```

## Logout

```typescript
async function logout(): Promise<void> {
  const userId = localStorage.getItem("kiwi_user_id");
  const refreshToken = localStorage.getItem("kiwi_refresh_token");
  const device = getDevice();

  if (userId && refreshToken) {
    try {
      await fetch(`${KIWI_USER_API_BASE_URL}/v1/user/logout`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          user_id: userId,
          refresh_token: refreshToken,
          device: device,
        }),
      });
    } catch {
      // Best-effort - clear local state regardless
    }
  }

  clearAuthData();
}
```
