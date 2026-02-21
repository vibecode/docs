---
name: google-login
description: Set up Google OAuth sign-in with Better Auth and Expo. Use when the user needs to add Google login, Google Sign-In, social login, or "set up Google login like this". Triggers include "add Google login", "Google Sign-In", "social login", "set up Google login like this" after GOOGLE_CLIENT_ID and GOOGLE_CLIENT_SECRET are in the backend environment variables.
---

# Google Login with Better Auth

Use this skill when the user wants Google Sign-In and has already added `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` to the backend ENV. Implement the following exactly. Use the scheme `vibecode` for testing inside the Vibecode app.

**Prerequisite:** The user must have the **database-auth** skill implemented before using this skill. If not, use the database-auth skill first, then return to set up Google login.

## Setup Steps

1. Backend: update `auth.ts` and `env.ts`
2. Mobile: update `auth-client.ts`, add sign-in handler and web redirect handling
3. Confirm `app.json` scheme
4. After implementing, tell the user the messages in **After Implementing — Tell the User** below

## Backend Setup

### Auth config — `backend/src/auth.ts`

Replace or update the file with this config. Keep any existing `emailOTP` plugin if present.

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
    "vibecode://",
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

### Env schema — `backend/src/env.ts`

Add to the Zod schema:

```typescript
GOOGLE_CLIENT_ID: z.string().min(1, "GOOGLE_CLIENT_ID is required"),
GOOGLE_CLIENT_SECRET: z.string().min(1, "GOOGLE_CLIENT_SECRET is required"),
```

## Mobile Setup

### Auth client — `src/lib/auth/auth-client.ts`

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

### Sign-in screen — Google button handler

Put the handler in the sign-in screen. **`WebBrowser.maybeCompleteAuthSession()` MUST be at the top of the file, outside the component.**

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

Add this `useEffect` in the component to handle the web redirect-back case:

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

### app.json — confirm scheme

Ensure this exists:

```json
{
  "expo": {
    "scheme": "vibecode"
  }
}
```

## Checklist — values that must match

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

## Most common reasons sign-in spins forever

- Missing `skipStateCookieCheck: true` in `account: {}` on the backend
- Missing `plugins: [expo()]` on the backend
- `disableCSRFCheck: true` not set at the **top level** of `betterAuth` config
- `import "expo-web-browser"` missing from `auth-client.ts`
- `storagePrefix` in `expoClient` doesn’t match the SecureStore key (`vibecode_cookie`)
- `callbackURL` not in `trustedOrigins` on the backend
- Google Cloud Console missing the **Authorized JavaScript origin** or **Redirect URI**

## After Implementing — Tell the User

After implementing, tell the user all of the following.

**1. Scheme when publishing**  
*"The Vibecode scheme will work inside the Vibecode app. When you publish your real app, you must update the scheme to match your app's bundle identifier in: app.json (expo.scheme), auth-client.ts (expoClient scheme and storagePrefix), the sign-in handler (SecureStore key = scheme_cookie, callbackURL = scheme://), and backend auth.ts (trustedOrigins). Google Console redirect URI does not need to change."*

**2. Preview URL vs BACKEND_URL**  
*"Here is the preview URL" (if you shared one). Then: Once you publish your app, you get a `BACKEND_URL` — that is your main URL for Google OAuth. The preview URL may change; don’t keep adding preview URLs to Google Cloud. Your `BACKEND_URL` never changes. To get it, you have to deploy your app. Add that same `BACKEND_URL` in both places in Google Cloud: **Authorized JavaScript origins** (as-is) and **Authorized redirect URIs** (`https://YOUR_BACKEND_URL/api/auth/callback/google`).*

**3. How to find BACKEND_URL**
- **On computer:** Deployments (top right) → Deployment details → Settings. `BACKEND_URL` is shown there. Copy it. Add that URL in **both** places in Google Cloud: **Authorized JavaScript origins** (add the `BACKEND_URL` as-is) and **Authorized redirect URIs** (add `https://YOUR_BACKEND_URL/api/auth/callback/google`).
- **On mobile:** ENV VAR tab → Backend. `BACKEND_URL` is shown there.
- **If the URL starts with `preview`** (e.g. `https://preview-abc.dev.vibecode.run`): the project is not deployed yet. Deploy/publish the app first to get the real `BACKEND_URL`. The preview URL is not stable for Google OAuth — deploy to get the permanent backend URL.
