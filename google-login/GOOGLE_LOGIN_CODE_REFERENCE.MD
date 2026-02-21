Here is every piece of code needed. This is exactly how the code should work for Google authorization to work once you paste in all your information.

1. Backend — backend/src/auth.ts
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
    skipStateCookieCheck: true,  // CRITICAL for mobile
  },

  plugins: [expo()],  // CRITICAL - from @better-auth/expo

  trustedOrigins: [
    "yourscheme://",       // Replace with your app's scheme
    "yourscheme://*",
    "yourscheme:///*",
    "exp://*",
    "http://localhost:*",
    "http://127.0.0.1:*",
    // Add your backend domains here
  ],

  advanced: {
    useSecureCookies: true,
    defaultCookieAttributes: {
      sameSite: "none",   // CRITICAL for mobile OAuth
      secure: true,
      partitioned: true,
    },
    crossSubDomainCookies: { enabled: true },
  },

  trustedProxyHeaders: ["x-forwarded-for", "x-forwarded-proto", "x-forwarded-host"],
  disableCSRFCheck: true,  // CRITICAL for mobile
});
2. Mobile — src/lib/auth/auth-client.ts
import { createAuthClient } from "better-auth/react";
import { expoClient } from "@better-auth/expo/client";
import * as SecureStore from "expo-secure-store";
import "expo-web-browser";

export const authClient = createAuthClient({
  baseURL: process.env.EXPO_PUBLIC_BACKEND_URL!,
  plugins: [
    expoClient({
      scheme: "yourscheme",       // Must match app.json "scheme"
      storagePrefix: "yourscheme",
      storage: SecureStore,
    }),
  ],
});
3. Mobile — Sign-in screen button handler
import * as WebBrowser from "expo-web-browser";
import * as SecureStore from "expo-secure-store";
import { Platform } from "react-native";

// This line MUST be at the top of the file (outside the component)
WebBrowser.maybeCompleteAuthSession();

const BACKEND_URL = process.env.EXPO_PUBLIC_BACKEND_URL;

const handleGoogleSignIn = async () => {
  setIsLoading(true);
  try {
    const callbackURL = Platform.OS === "web"
      ? window.location.origin
      : "yourscheme://";  // Must match app.json "scheme"

    // Get OAuth URL from backend
    const response = await fetch(`${BACKEND_URL}/api/auth/sign-in/social`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Origin": BACKEND_URL!,
      },
      body: JSON.stringify({ provider: "google", callbackURL }),
      credentials: "include",
    });

    const data = await response.json();
    if (!data.url) return;

    if (Platform.OS === "web") {
      window.location.href = data.url;
    } else {
      // This opens the native Google popup
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
            // Replace "yourscheme" with your storagePrefix
            await SecureStore.setItemAsync("yourscheme_cookie", cookieJson);
          }
        }

        await invalidateSession(); // refresh auth state
      }
    }
  } finally {
    setIsLoading(false);
  }
};
4. Checklist — things that must match
What	Where	Must match
scheme	app.json → "scheme": "yourscheme"	everywhere you use yourscheme://
storagePrefix	expoClient({ storagePrefix })	SecureStore.setItemAsync("yourscheme_cookie", ...)
callbackURL	button handler	in trustedOrigins on backend
@better-auth/expo	backend dependencies	plugins: [expo()] used in auth.ts
The most common reason it spins forever is missing skipStateCookieCheck: true or missing plugins: [expo()] on the backend.