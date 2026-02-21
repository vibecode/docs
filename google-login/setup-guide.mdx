# 🔐 Add Google Sign-In to Your Mobile App

**Using Better Auth + Prisma** — follow every step in order.

---

## Prerequisites

Do these **before** writing any code.

### Google Cloud Console

- Go to [Google Cloud Console](https://console.cloud.google.com)
- Create a project (or select an existing one)
- **APIs & Services → OAuth consent screen** — configure it
- **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
  - Set **Application type** to **Web application**
  - **Authorized JavaScript origins:** `https://YOUR_BACKEND_URL`
  - **Authorized redirect URIs:** `https://YOUR_BACKEND_URL/api/auth/callback/google`
- Copy the **Client ID** and **Client Secret**
- Add them in your **ENV** tab:
  - `GOOGLE_CLIENT_ID` = your client ID
  - `GOOGLE_CLIENT_SECRET` = your client secret

### Already set up (use database-auth skill if not)

- Prisma + SQLite database
- Better Auth installed on backend
- `BETTER_AUTH_SECRET` and `BACKEND_URL` env vars set
- `@better-auth/expo` installed on **both** backend and mobile
- `expo-web-browser` and `expo-secure-store` installed on mobile

---

## Backend

### `backend/src/auth.ts`

Replace the entire file with this. Keep your existing `emailOTP` plugin if you have it.

```typescript
import { betterAuth } from "better-auth";
import { expo } from "@better-auth/expo";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { prisma } from "./prisma";
import { env } from "./env";

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "sqlite" }),
  baseURL: env.BACKEND_URL,
  basePath: "/api/auth",
  secret: env.BETTER_AUTH_SECRET,

  socialProviders: {
    google: {
      clientId: env.GOOGLE_CLIENT_ID || "",
      clientSecret: env.GOOGLE_CLIENT_SECRET || "",
    },
  },

  account: {
    skipStateCookieCheck: true,  // CRITICAL for mobile — without this it spins forever
    accountLinking: { enabled: true },
  },

  plugins: [expo()],  // CRITICAL — must be @better-auth/expo

  disableCSRFCheck: true,  // CRITICAL for mobile

  trustedOrigins: [
    "vibecode://",        // Replace "vibecode" with your app's scheme from app.json
    "vibecode://*",
    "vibecode:///*",
    "exp://*",
    "http://localhost:*",
    "http://127.0.0.1:*",
    "https://*.dev.vibecode.run",
    "https://*.vibecode.run",
    "https://*.vibecodeapp.com",
    "https://*.vibecode.dev",
    "https://vibecode.dev",
  ],

  advanced: {
    useSecureCookies: true,
    defaultCookieAttributes: {
      sameSite: "none",
      secure: true,
      partitioned: true,
    },
    crossSubDomainCookies: { enabled: true },
    trustedProxyHeaders: true,
    disableCSRFCheck: true,
  },
});
```

### `backend/src/env.ts`

Add these two fields to your Zod schema:

```typescript
GOOGLE_CLIENT_ID: z.string().min(1, "GOOGLE_CLIENT_ID is required"),
GOOGLE_CLIENT_SECRET: z.string().min(1, "GOOGLE_CLIENT_SECRET is required"),
```

---

## Mobile

### `src/lib/auth/auth-client.ts`

```typescript
import { createAuthClient } from "better-auth/react";
import { expoClient } from "@better-auth/expo/client";
import * as SecureStore from "expo-secure-store";
import "expo-web-browser";  // REQUIRED — must import even if unused

export const authClient = createAuthClient({
  baseURL: process.env.EXPO_PUBLIC_BACKEND_URL! as string,
  plugins: [
    expoClient({
      scheme: "vibecode",        // Must match "scheme" in app.json
      storagePrefix: "vibecode", // Must match the SecureStore key prefix below
      storage: SecureStore,
    }),
  ],
});
```

### Google Sign-In button handler

Put this in your sign-in screen. The `maybeCompleteAuthSession()` call **MUST** be outside the component at the top of the file.

```typescript
import * as WebBrowser from "expo-web-browser";
import * as SecureStore from "expo-secure-store";
import { Platform } from "react-native";

// MUST be at the top of the file, outside the component
WebBrowser.maybeCompleteAuthSession();

const BACKEND_URL = process.env.EXPO_PUBLIC_BACKEND_URL!;

const handleGoogleSignIn = async () => {
  setIsLoading(true);
  try {
    const callbackURL =
      Platform.OS === "web"
        ? (typeof window !== "undefined" ? window.location.origin : "/")
        : "vibecode://";  // Must match scheme in app.json

    // Step 1: Get the Google OAuth URL from the backend
    const response = await fetch(`${BACKEND_URL}/api/auth/sign-in/social`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Origin": BACKEND_URL,
      },
      body: JSON.stringify({ provider: "google", callbackURL }),
      credentials: "include",
    });

    const data = await response.json() as { url?: string };
    if (!data.url) return;

    if (Platform.OS === "web") {
      // Web: redirect the browser directly to Google
      window.location.href = data.url;
      return; // page is navigating away
    }

    // Native: open the system browser with the Google OAuth URL
    const result = await WebBrowser.openAuthSessionAsync(data.url, callbackURL);

    if (result.type === "success" && result.url) {
      const url = new URL(result.url);
      const cookie = url.searchParams.get("cookie");

      if (cookie) {
        const decodedCookie = decodeURIComponent(cookie);
        const sessionTokenMatch = decodedCookie.match(
          /__Secure-better-auth\.session_token=([^;]+)/
        );
        const sessionToken = sessionTokenMatch?.[1];

        if (sessionToken) {
          const decodedToken = decodeURIComponent(sessionToken);
          const cookieJson = JSON.stringify({
            "__Secure-better-auth.session_token": {
              value: decodedToken,
              expires: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString(),
            },
          });
          // "vibecode" must match storagePrefix in auth-client.ts
          await SecureStore.setItemAsync("vibecode_cookie", cookieJson);
        }
      }

      await invalidateSession(); // refresh auth state — triggers navigation guards
    }
  } finally {
    setIsLoading(false);
  }
};
```

Also add this `useEffect` in the component to handle the web redirect-back case:

```typescript
useEffect(() => {
  if (Platform.OS === "web" && typeof window !== "undefined") {
    const params = new URLSearchParams(window.location.search);
    if (params.has("code") || params.has("session")) {
      invalidateSession();
    }
  }
}, []);
```

### `app.json` — confirm scheme

```json
{
  "expo": {
    "scheme": "vibecode"
  }
}
```

---

## ✅ Checklist — things that must all match

| What | Where | Value |
|------|-------|--------|
| scheme | `app.json` | `vibecode` |
| scheme | `expoClient({ scheme })` | `vibecode` |
| storagePrefix | `expoClient({ storagePrefix })` | `vibecode` |
| SecureStore key | `setItemAsync("vibecode_cookie", ...)` | `vibecode_cookie` |
| callbackURL on native | button handler | `vibecode://` |
| trustedOrigins | `backend/src/auth.ts` | includes `vibecode://` and `vibecode://*` |
| Redirect URI | Google Console OAuth credentials | `https://YOUR_BACKEND/api/auth/callback/google` |
| JavaScript origin | Google Console OAuth credentials | `https://YOUR_BACKEND` |

---

## ⚠️ Most common reasons it spins forever

- Missing `skipStateCookieCheck: true` in `account: {}` on the backend
- Missing `plugins: [expo()]` on the backend
- `disableCSRFCheck: true` not set at the **top level** of `betterAuth` config
- `import "expo-web-browser"` missing from `auth-client.ts`
- `storagePrefix` in `expoClient` doesn’t match the SecureStore key (`vibecode_cookie`)
- `callbackURL` not in `trustedOrigins` on the backend
- Google Cloud Console missing the **Authorized JavaScript origin** or **Redirect URI**
