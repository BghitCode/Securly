# Mobile App Security (Expo / React Native, Flutter)

Mobile apps have a different threat model than web apps: the client runs on a device the user (or an attacker with physical access) fully controls. Anything bundled into the app, API keys, business logic, certificate pins, can be extracted by someone willing to decompile the binary. The rules below assume that starting point rather than assuming the client is trustworthy.

This file applies whenever the project is a React Native / Expo app, a Flutter app, or a hybrid app calling into an AI backend, chatbot, or agent from a mobile client. Pair it with `owasp-web-security.md` for the backend API the app talks to, and `llm-security.md` if the app embeds any AI chat/agent features.

## Secrets and API keys

**Never ship secret API keys inside the app bundle.** Anything in JS/Dart source, `.env` files bundled at build time, or `app.json`/`Info.plist`/`AndroidManifest.xml` is extractable from the compiled binary with basic reverse-engineering tools — it is not meaningfully hidden by minification or obfuscation alone.

- Route third-party API calls (LLM providers, payment processors, maps, etc.) through your own backend, which holds the real key. The app calls your backend; your backend calls the third party.
- If a key must be on-device (e.g. a public/publishable key meant for client use), verify with the provider that it's actually designed to be public and scoped to least privilege — most aren't.
- For Expo: `expo-constants` / `app.config.js` values are not secrets. `EXPO_PUBLIC_*` env vars are explicitly bundled into the client and readable by anyone.

```javascript
// Never:
const OPENAI_API_KEY = "sk-..."; // in RN/Expo source, ends up in the bundle

// Instead: app calls your backend, backend holds the key
fetch('https://api.yourapp.com/chat', { method: 'POST', body: JSON.stringify({ message }) });
```

## Secure storage on-device

**Never store tokens, session credentials, or PII in `AsyncStorage`, `localStorage`-style web storage, or unencrypted `SharedPreferences`/`UserDefaults`.** These are stored in plaintext on disk and readable by anyone with device/backup access (or trivially on a rooted/jailbroken device).

- React Native / Expo: use `expo-secure-store` (iOS Keychain / Android Keystore-backed) for tokens and credentials. `AsyncStorage` is fine only for non-sensitive app state (UI preferences, cached non-sensitive data).
- Flutter: use `flutter_secure_storage`, which similarly backs onto Keychain/Keystore, not `shared_preferences`.
- Never persist a refresh token or long-lived session credential to storage that survives outside the OS-provided secure enclave.

## Transport security

**Enforce HTTPS/TLS for every network call, no exceptions for "just during development" configs that make it into a release build.** Watch specifically for:

- Android: `usesCleartextTraffic` left `true` in a release manifest.
- iOS: overly broad App Transport Security exceptions (`NSAllowsArbitraryLoads`) left enabled.

**Consider certificate pinning for high-sensitivity apps** (banking, health, anything handling payment or highly private data) to defend against MITM via a malicious or compromised CA. Libraries: `react-native-ssl-pinning` / platform-native pinning for RN, `http_certificate_pinning` or Dio interceptors for Flutter. Pinning adds an operational cost (cert rotation breaks the app if not planned for), so scope it to apps that actually need it rather than applying it by default.

## Deep links and app links

**Validate and sanitize every parameter received through a deep link or universal/app link before acting on it.** A deep link is untrusted input from outside your app, the same way a URL query parameter is untrusted input to a web backend.

- Never use a deep link parameter to directly construct a navigation target, API call, or piece of UI (e.g. rendering raw HTML) without validation.
- Verify the link's origin where the platform supports it (Android App Links / iOS Universal Links with domain verification) over the weaker custom URL scheme (`myapp://`), which any other app can register and trigger.
- Watch specifically for auth-bypass patterns: a deep link that jumps straight to an authenticated screen without re-checking session state.

## Authentication on-device

**Biometric authentication (Face ID / Touch ID / Android biometric prompt) should gate access to an already-issued credential, not replace server-side authentication.** Biometrics unlock something local (a stored token, a local encryption key); they don't themselves authenticate the user to your backend. Don't design a flow where "biometric succeeded" alone is sent to the server as proof of identity.

**Don't roll your own biometric integration on either platform** — use `expo-local-authentication` (RN/Expo) or `local_auth` (Flutter), which correctly delegate to the OS-level secure APIs rather than reimplementing the check in app code, where it could be bypassed.

## Runtime protections

**Detect (and decide deliberately how to respond to) rooted/jailbroken devices for high-sensitivity apps.** Root/jailbreak doesn't mean automatically blocking the app for every use case, but for apps handling payments, health data, or enterprise data, it's worth detecting and either warning the user or restricting sensitive functionality, since root/jailbreak removes OS-level protections your other controls (secure storage, pinning) depend on.

**Apply code obfuscation for release builds** (Hermes bytecode + tooling like `react-native-obfuscating-transformer` for RN; Dart's default AOT compilation already provides some obfuscation for Flutter, and `flutter build --obfuscate --split-debug-info` strengthens it further) to raise the cost of reverse-engineering business logic, not as a substitute for not shipping real secrets client-side in the first place.

**Disable or guard debug-only features in release builds** — remote JS debugging (RN), verbose logging, developer menus, and mock/test API endpoints should not be reachable in a production build.

## Platform-specific app config

**Review exported components and intent filters (Android) / URL scheme and extension configs (iOS) for anything broader than the app actually needs.** An `exported="true"` Activity with no permission check can be launched by any other app on the device.

**Restrict clipboard, screenshot, and screen-recording access for screens showing sensitive data** (payment forms, one-time codes, private messages) where the platform allows it — this is a common gap in AI chat apps that display sensitive user data or LLM-generated content containing PII.

## AI/chat-specific mobile concerns

If the mobile app embeds an LLM chatbot or agent (common in AI-first mobile products), the concerns in `llm-security.md` and `agent-security.md` still fully apply, with two mobile-specific additions:

- **Don't run the actual LLM inference or tool-calling logic on-device with embedded credentials.** Route it through your backend for the same reason API keys don't belong in the bundle, on-device tool-calling logic with embedded service credentials is extractable the same way any other secret is.
- **Sanitize any content a mobile agent tool can access on-device** (contacts, photos, files, clipboard) with the same "treat it as untrusted input" posture as RAG documents on the backend, especially if that content can reach the model's context.

## Quick pattern reference

| Risk | Control |
|---|---|
| Secrets in the app bundle | Route through backend; never embed real API keys client-side |
| Plaintext token storage | `expo-secure-store` (RN) / `flutter_secure_storage` (Flutter) |
| Cleartext traffic | Enforce TLS; disable cleartext exceptions in release builds |
| MITM on sensitive apps | Certificate pinning, scoped to apps that need it |
| Deep link abuse | Validate deep link params like any other untrusted input; verify link origin |
| Biometric as sole auth | Biometrics unlock a local credential; server-side auth still required |
| Reverse engineering | Release-build obfuscation, debug features stripped from release builds |
| Rooted/jailbroken devices | Detect and respond deliberately for high-sensitivity apps |
| On-device LLM/agent logic | Route inference and tool-calling through the backend, not on-device |
