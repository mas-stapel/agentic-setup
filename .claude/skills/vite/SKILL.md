---
name: vite
description: >
  Vite v5+ build tooling and Vitest expertise for frontend and desktop applications.
  Provides opinionated standards for vite.config.ts structure, Tauri integration,
  Vitest setup, path alias conventions, self-hosted asset patterns, and environment
  variable handling. Activate this skill when configuring Vite or Vitest, reviewing
  build configs, debugging HMR in a Tauri context, or optimising a desktop app bundle.
---

# Vite — Build Tooling

Opinionated, production-tested standards for Vite v5+ and Vitest — config structure, Tauri integration, Vitest setup, path aliases, self-hosted assets, and environment variables.

---

## `vite.config.ts` Structure

Always export config via `defineConfig` for full TypeScript inference. Keep plugins, resolve, server, and build in clearly separated sections:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],

  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
    },
  },

  server: {
    // Required for Tauri — see Tauri Integration section
    host: true,
    port: 1420,
    strictPort: true,
  },

  build: {
    // Produce a single entry for Tauri (no HTML entry rewriting needed)
    rollupOptions: {
      output: {
        manualChunks: undefined, // desktop apps: don't split by default
      },
    },
  },
});
```

**`tsconfig.node.json`**: Vite's own config file must be covered by a separate TypeScript config that includes `vite.config.ts` and `vitest.config.ts`. Without it, `tsc --noEmit` misses the config file entirely:

```json
{
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts", "vitest.config.ts"]
}
```

Reference it from the root `tsconfig.json`:

```json
{
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

---

## Tauri-Specific Vite Configuration

When Vite powers a Tauri application, three settings are **non-negotiable**. Getting any of them wrong silently breaks the Tauri dev runner without a clear error:

```typescript
server: {
  host: true,        // Expose to all interfaces — Tauri connects via 127.0.0.1
  port: 1420,        // Tauri's default devUrl port — must match tauri.conf.json
  strictPort: true,  // Fail fast if the port is taken; don't silently bind :1421
},

clearScreen: false,  // Preserve Tauri's own terminal output; do NOT set to true
```

`tauri.conf.json` must reference the exact port:

```json
{
  "build": {
    "devUrl": "http://localhost:1420",
    "frontendDist": "../dist"
  }
}
```

**HMR in Tauri**: Hot module replacement works out of the box when `host: true` and `port` match. If HMR stops updating the Tauri window, the first thing to check is whether another process claimed port 1420 — `strictPort: true` makes this a hard error rather than a silent rebind.

---

## Path Aliases

Always configure `@/` as the canonical alias for `src/`. Two places must stay in sync — they are independent systems and neither infers from the other:

**`vite.config.ts`**:
```typescript
import { resolve } from 'path';

resolve: {
  alias: {
    '@': resolve(__dirname, './src'),
  },
},
```

**`tsconfig.json`**:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

If you add the alias in only one place:
- Vite-only: TypeScript shows errors, `tsc --noEmit` fails.
- tsconfig-only: TypeScript is happy but the runtime bundler cannot resolve the path and throws at build time.

ESLint also needs awareness of the alias to correctly order imports. Add `eslint-import-resolver-typescript` and configure it in `.eslintrc.cjs`:

```js
settings: {
  'import/resolver': {
    typescript: {
      alwaysTryTypes: true,
    },
  },
},
```

---

## Vitest — The Natural Test Runner for Vite Projects

**Use Vitest, not Jest, in any Vite-based project.** Vitest shares Vite's transform pipeline, meaning the same plugins, aliases, and TypeScript config apply to tests without any separate Babel, ts-jest, or jest.config transform setup. This eliminates an entire class of "works in source but fails in tests" bugs (mismatched module resolution, import alias not resolving, CSS module errors).

### `vitest.config.ts` — Separate File Pattern

Split the test config from the build config. This makes the test environment explicit and prevents test-only settings from polluting the production build:

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],

  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
    },
  },

  test: {
    environment: 'jsdom',          // DOM environment for React component tests
    globals: true,                 // Removes the need to import describe/it/expect
    setupFiles: ['./src/test-setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      exclude: ['src/test-setup.ts', '**/*.d.ts'],
    },
  },
});
```

**`src/test-setup.ts`**: Configure `@testing-library/react` and any global mocks here:

```typescript
import '@testing-library/jest-dom';
// Global mock for Tauri IPC (override per-test as needed)
vi.mock('@tauri-apps/api/core', () => ({
  invoke: vi.fn(),
}));
```

### Mocking Tauri APIs in Tests

Tauri APIs (`@tauri-apps/api/core`, `@tauri-apps/api/event`, `@tauri-apps/api/dialog`) are not available in the jsdom test environment — they require the native Tauri runtime. Always mock them:

```typescript
// Per-test override
vi.mocked(invoke).mockResolvedValueOnce({ version: '6.8.5', total_tracks: 4821 });
```

Keep a central `src/test-utils/tauriMocks.ts` with typed mock factories for common IPC responses:

```typescript
import { vi } from 'vitest';
import { invoke } from '@tauri-apps/api/core';

export function mockImportCollection(dto: Partial<CollectionDto> = {}) {
  vi.mocked(invoke).mockResolvedValueOnce({
    version: '6.8.5',
    total_tracks: 100,
    playlists: [],
    tracks: [],
    ...dto,
  });
}
```

### Key Differences from Jest

| Jest | Vitest |
| ---- | ------ |
| `jest.fn()` | `vi.fn()` |
| `jest.mock()` | `vi.mock()` |
| `jest.spyOn()` | `vi.spyOn()` |
| `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| `jest.resetAllMocks()` | `vi.resetAllMocks()` |
| Runs via `npx jest` | Runs via `npx vitest run` |

Global `describe`, `it`, `expect`, `beforeEach` work identically. The API surface is intentionally compatible.

---

## Self-Hosted Assets in Desktop Apps

Desktop apps (Tauri, Electron) cannot fetch assets from CDNs at runtime — there is no guarantee of network access, and loading fonts/icons over the network introduces latency on first render. **All fonts, icons, and static assets must be bundled with the app.**

### Font Setup

Download font files as `.woff2` (best compression, modern support) into `src/assets/fonts/`. Declare `@font-face` in a CSS file that Vite will include in the bundle:

```css
/* src/styles/fonts.css */
@font-face {
  font-family: 'IBM Plex Sans';
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url('/assets/fonts/IBMPlexSans-Regular.woff2') format('woff2');
}
```

**Path resolution**: In development, Vite serves `src/assets/` at `/assets/`. In production, Vite copies `src/assets/` to `dist/assets/` — paths remain the same. Do not use relative paths (`../assets/`) in CSS because they break when the CSS is inlined or extracted.

### Icon Libraries

For component-level icon libraries (`lucide-react`, `@heroicons/react`), import individual icons — never import the entire library. Vite's tree-shaking eliminates unused icons at build time:

```typescript
// ✅ Tree-shaken — only Music and FileAudio end up in the bundle
import { Music, FileAudio } from 'lucide-react';

// ❌ Imports the entire icon library
import * as Icons from 'lucide-react';
```

---

## Environment Variables

Vite exposes environment variables to the browser bundle **only** when they are prefixed with `VITE_`. Variables without this prefix are stripped at build time.

```typescript
// Accessible in the browser:
const apiUrl = import.meta.env.VITE_API_URL;

// NOT accessible (undefined in the bundle):
const secret = import.meta.env.API_SECRET;
```

**In a Tauri app**: avoid `VITE_` secrets entirely — anything in the Vite bundle is readable by the user. Use Tauri commands to proxy sensitive operations through the Rust backend instead.

**TypeScript types for env vars**: Extend the `ImportMetaEnv` interface to get autocomplete and type checking:

```typescript
// src/vite-env.d.ts (auto-generated by Vite scaffold — extend it)
interface ImportMetaEnv {
  readonly VITE_APP_VERSION: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## Build Optimisation for Desktop Apps

Desktop apps have different optimisation priorities than web apps:

**Do not aggressively split chunks.** Code splitting exists to reduce initial network payload for web apps. In a Tauri app, the entire bundle is loaded from the local filesystem in milliseconds — splitting into many small chunks adds filesystem overhead with no benefit. Keep `rollupOptions.output.manualChunks: undefined` unless you have a specific reason to split.

**Do inline small assets.** Vite's `assetsInlineLimit` (default 4096 bytes) controls which assets are inlined as base64. For a desktop app, raising this to 8192 or even 16384 reduces the number of filesystem reads on startup:

```typescript
build: {
  assetsInlineLimit: 8192,
},
```

**Source maps in development only.** Never ship source maps in a production Tauri build — they bloat the bundle and expose source code to the user. Source maps are enabled by default in `vite dev` and disabled in `vite build` — do not override this.

---

## Common Pitfalls

**HMR not updating the Tauri window after a save**
- Check: Is `server.host: true` set? Without it, Vite binds to `127.0.0.1` only and Tauri cannot connect.
- Check: Did another process grab port 1420? Add `strictPort: true` to detect this immediately.
- Check: Is `clearScreen: false` set? Without it, Tauri's output is cleared and errors may be hidden.

**TypeScript path aliases resolve in editor but fail at build time**
- Cause: The alias is in `tsconfig.json` but not in `vite.config.ts` `resolve.alias`.
- Fix: Both must define `@/`. They are independent systems.

**`process.env` is undefined in the browser bundle**
- Cause: Vite does not polyfill `process`. Use `import.meta.env` instead.
- Note: Some Node.js packages reference `process.env.NODE_ENV`. Vite handles this specific variable automatically — it does not handle arbitrary `process.env` references.

**ESLint import ordering fails on `@/` aliases**
- Cause: `eslint-plugin-import` doesn't know `@/` maps to `src/` without a resolver.
- Fix: Install `eslint-import-resolver-typescript` and configure it under `settings.import/resolver` in your ESLint config.

**`vitest run` passes but `tsc --noEmit` fails on test files**
- Cause: Test files import `vi` from `vitest` but the root `tsconfig.json` doesn't include `src/**/*.test.ts`.
- Fix: Add `"include": ["src"]` to the root tsconfig (not just `"include": ["src/**/*.tsx"]`) so all `.ts` files in `src/` are covered.
