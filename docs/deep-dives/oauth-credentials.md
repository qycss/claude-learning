# OAuth & Credentials

## Overview

Claude Code implements a complete OAuth 2.0 Authorization Code + PKCE flow with dual-mode authorization code acquisition (automatic localhost callback + manual copy-paste). Credential storage uses a layered architecture: macOS Keychain (via `security` CLI) with fallback to `~/.claude/.credentials.json` plaintext. Token refresh uses file lock + triple-check pattern to prevent cross-process races.

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/services/oauth/index.ts` | 199 | OAuthService — PKCE flow orchestration |
| `src/services/oauth/client.ts` | 567 | OAuth client — URL building, token exchange/refresh |
| `src/services/oauth/crypto.ts` | 24 | PKCE crypto — code_verifier, code_challenge, state |
| `src/services/oauth/auth-code-listener.ts` | 212 | Localhost HTTP server for OAuth callback |
| `src/utils/auth.ts` | 1900+ | Auth core — token read/refresh/save/cache |
| `src/utils/secureStorage/macOsKeychainStorage.ts` | 232 | macOS Keychain implementation |
| `src/utils/secureStorage/macOsKeychainHelpers.ts` | 112 | Keychain helpers — cache, TTL, generation counter |
| `src/utils/secureStorage/fallbackStorage.ts` | 71 | Fallback composition (primary → secondary) |

## PKCE Flow

```
1. Generate code_verifier = base64url(randomBytes(32))
2. Generate code_challenge = base64url(SHA-256(code_verifier))
3. Generate state = base64url(randomBytes(32))  // CSRF prevention
4. Build auth URL with code_challenge_method=S256
5. Dual flow parallel:
   ├── Automatic: openBrowser(url) → localhost:PORT/callback
   └── Manual: display URL for copy-paste
6. Promise.race waits for either authorization code
7. Exchange code → tokens via exchangeCodeForTokens()
```

## Triple-Check Token Refresh (`auth.ts:1427-1562`)

```
Check 1: Is cached token expired?
   ↓ yes
Check 2: Clear ALL caches, async re-read. Still expired?
   ↓ yes
Check 3: Acquire file lock, re-read inside lock. Still expired?
   ↓ yes
Execute refreshOAuthToken()
```

Lock contention uses randomized backoff (`1000 + Math.random() * 1000` ms), up to 5 retries.

## 401 In-Flight Dedup (`auth.ts:1338-1343`)

```typescript
const pending401Handlers = new Map<string, Promise<boolean>>()
```

When N proxy connectors 401 simultaneously (common at startup — #20930), dedup by `staleAccessToken`: only the first clears cache + reads keychain, others join the same Promise.

## Keychain Storage (macOS)

### Read
```
security find-generic-password -a "${username}" -w -s "${storageServiceName}"
```

### Write
1. JSON → hex encoding (`Buffer.from(json).toString('hex')`)
2. Prefer `security -i` stdin (avoids CrowdStrike/process monitoring seeing payload — INC-3028)
3. If command exceeds 4032 bytes (`4096 - 64`), fall back to argv

### 4096-Byte Stdin Limit

macOS `security -i` uses `fgets()` with `BUFSIZ=4096`. Exceeding this:
- First 4096 bytes → parsed as one command (unclosed quotes → failure)
- Overflow → parsed as next unknown command
- Result: non-zero exit but old keychain entry preserved
- 64-byte headroom for line terminator differences

### Stale-While-Error Cache

30-second TTL (`KEYCHAIN_CACHE_TTL_MS = 30_000`). On read failure with existing cache, returns stale value instead of `null` — prevents "Not logged in" state propagation (#23192).

### Generation Counter

`keychainCacheState.generation` increments on every `clearKeychainCache()`. `readAsync()` captures generation before subprocess spawn, discards result if generation changed during execution (update() wrote new data).

## Fallback Storage (`fallbackStorage.ts`)

```
createFallbackStorage(macOsKeychainStorage, plaintextStorage)
```

- Read: primary first, null → try secondary
- Write: primary fails → write to secondary; if secondary succeeds and primary has stale data, delete primary entry
- First successful primary write → delete secondary (migration cleanup)

## Unique Findings

1. **Scope expansion on refresh** (`client.ts:152-163`): Explicitly passes `CLAUDE_AI_OAUTH_SCOPES` during refresh, leveraging server-side `ALLOWED_SCOPE_EXPANSIONS` to expand scopes (e.g., add `user:file_upload`) without re-login.
2. **7M req/day optimization** (`client.ts:189-199`): Skip `/api/oauth/profile` call if complete profile data already exists in global config + secure storage. Comment states this saves ~7M daily requests fleet-wide.
3. **Keychain prefetch import isolation** (`macOsKeychainHelpers.ts:1-15`): Module must NOT import execa — Bun's `__esm` wrapper evaluates entire module on any symbol access, and execa's `human-signals/cross-spawn` chain costs ~58ms sync initialization.
4. **isMacOsKeychainLocked lifetime cache** (`macOsKeychainStorage.ts:210-231`): `security show-keychain-info` exit code 36 = locked (common in SSH). Cached for process lifetime to prevent 27ms/message rendering delay from repeated spawns.
5. **Cross-process disk change detection** (`auth.ts:1320-1336`): Checks `.credentials.json` mtime before each token operation — fixes stale memoize when another CC instance does `/login` (CC-1096, GH#24317).
