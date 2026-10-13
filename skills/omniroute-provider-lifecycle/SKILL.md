---
name: omniroute-provider-lifecycle
description: >
  Pełny cykl życia providera OmniRoute: architektura (Bearer vs Session Cookie), domeny (`v0-*` vs modele upstream), rejestr (`buildModels`, `passthroughModels`, `unsupportedParams`), procedury testów regresyjnych.
  Używa się go przy dodawaniu nowego providera, aktualizacji istniejącego lub debugowaniu problemów w rejestrze.
version: 0.1.0
author: Hermes Agent
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [omniroute, provider-lifecycle, provider-registry, buildModels, passthroughModels, unsupportedParams, router]
    related_skills: [security-incident-response, devops-quirks-windows]
---

# OmniRoute Provider Lifecycle

> Cel: Uniknięcie błędów architektonicznych przy integracji nowych providerów. Zapewnia zgodność między rejestrem, statycznym katalogiem, executorami i dokumentacją Vercel.

---

## 1. Architektura OmniRoute — rodzaje dostawców

### Rodzaje dostawców i ich domeny

| Rodzaj | Endpoint | Auth | Domain modele | Executor | Uwagi |
|---|---|---|---|---|---|
| `v0-vercel` | `https://api.v0.dev/v1/chat/completions` | Bearer (`vercel_API_KEY`) | `v0-*` (only v0.dev codegen) | `default` | API-key provider, do weryfikacji lokalnej |
| `v0-vercel-web` | `https://v0.dev/api/chat` | Session Cookie (`__vercel_session`) | `v0-*` (only v0.dev codegen) | `V0VercelWebExecutor` | Web-cookie provider, nie może obsługiwać modeli zewnętrznych |
| `vercel-ai-gateway` | `https://ai-gateway.vercel.sh/v1/chat/completions` | Bearer (`VERCEL_AI_GATEWAY_API_KEY`) | Modele upstream (pozorzec) | `default` | Auto-fetch, `passthroughModels: true` |

### Reguły Ernestne

- **Nie mieszaj domen modeli:** `v0-*` to tylko v0.dev codegen. Modele zewnętrzne (Claude, GPT, Gemini) nie mogą być w `v0-vercel` ani `v0-vercel-web`.
- **Różne źródła autoryzacji:** `vercel_API_KEY` (Bearer) ≠ `VERCEL_AI_GATEWAY_API_KEY` (Bearer) ≠ `__vercel_session` (Session Cookie).
- **Executor determinuje domenę:** `V0VercelWebExecutor` również rozumie tylko `v0-*` modele przez `POST https://v0.dev/api/chat`.

---

## 2. Rejestr providera — struktura

### Kompletny przykład (registry entry)

```typescript
// open-sse/config/providers/registry/vercel-ai-gateway/index.ts
import type { RegistryEntry } from "../../shared.ts";
import { CHAT_OPENAI_COMPAT_MODELS } from "../../shared.ts";

export const vercel_ai_gatewayProvider: RegistryEntry = {
  id: "vercel-ai-gateway",
  alias: "vag",
  format: "openai",
  executor: "default",
  baseUrl: "https://ai-gateway.vercel.sh/v1/chat/completions",
  authType: "apikey",
  authHeader: "bearer",
  models: CHAT_OPENAI_COMPAT_MODELS["vercel-ai-gateway"],
  passthroughModels: true, // <-- kluczowe dla auto-fetch
};
```

### Polebka (API key env vars)

```bash
# v0-vercel (Bearer)
vercel_API_KEY=***

# vercel-ai-gateway (Bearer)
VERCEL_AI_GATEWAY_API_KEY=***
```

---

## 3. Katalog statyczny — `staticModels.ts`

### Reguła: tylko modele, które są obsługiwane przez provider.

| Provider | Scope | Przykłady |
|---|---|---|
| `v0-vercel-web` | `v0-*` only (v0.dev codegen) | `v0-1.0-md`, `v0-1.5-lg`, `v0-1.5-md` |
| `vercel-ai-gateway` | Pozor na modele upstream | `openai/gpt-4.1`, `anthropic/claude-4-sonnet`, `google/gemini-2.5-pro`, `moonshotai/kimi-k2` |

### Pattern

```typescript
// src/lib/providers/staticModels.ts
// v0-vercel-web: tylko v0.dev codegen
export const v0VercelWebModels = [
  "v0-1.0-md",
  "v0-1.5-lg",
  "v0-1.5-md",
];
```

---

## 4. Procedura testów regresyjnych

### Krok po kroku

1. **Czytelność (deduplikacja ID):** Sprawdź czy nie ma duplikatów:
   ```bash
   node -e "const ids=['v0-1.0-md','v0-1.5-lg','v0-1.5-md']; console.log(new Set(ids).size===ids.length)"
   ```
2. **Selekcja domeny:** Tylko modele z prefixem `$1`?
   ```bash
   node -e "const ids=['v0-1.0-md','v0-1.5-lg']; console.log(ids.every(id=>id.startsWith('v0-')))"
   ```
3. **Liczba modeli:** Dopuszczona liczba (status quo dla v0-vercel-web = 3).
4. **Run unit test:**
   ```bash
   node --import tsx/esm --test --test-concurrency=1 tests/unit/repro-10990-v0-vercel-web-static-models.test.ts
   ```

### Testy regresyjne — co sprawdzać

- Brak duplikatów (`Set(ids).size === ids.length`)
- Tylko modele z prefixem `$1`
- Dokładna liczba zadeklarowanych modeli
- Dla `v0-vercel-web`: tylko v0.dev modele (`v0-*` prefix)

---

## 5. Debugowanie rejestru `getUnsupportedParams`

### Cascady sprawdzania

```typescript
// Konwencja: getUnsupportedParams(provider, modelId)
// 1. Model-specific: entry.models.find(m => m.id === modelId)?.unsupportedParams
// 2. O(1) cache map
// 3. Prefixed ID: "openai/o3" → "o3"
// 4. Provider-level fallback: entry.unsupportedParams
```

### Przykład: `passthroughModels: true`

- `passthroughModels: true` oznacza, że provider obsługuje modele poza statycznym katalogiem.
- `unsupportedParams` na poziomie providera cross-cutting: zastępuje wszystkie modele, które nie mają własnego wpisu.

---

## 6. Audit checklist DoD

### A. Architektura

| Check | Pass |
|---|---|
| Tylko modele `v0-*` w `v0-vercel-web` | ✅/❌ |
| Tylko modele `v0-*` w `v0-vercel` | ✅/❌ |
| `vercel-ai-gateway` ma autoryzację Bearer | ✅/❌ |
| `v0-vercel-web` używa `__vercel_session` cookie | ✅/❌ |

### B. Rejestr providerów

| Check | Pass |
|---|---|
| Registry entry zawiera `id`, `alias`, `format`, `executor`, `baseUrl`, `authType`, `authHeader`, `models` | ✅/❌ |
| `passthroughModels` ustawione (gdy auto-fetch) | ✅/❌ |
| `authType` zgodne z endpointem | ✅/❌ |

### C. Testy regresyjne

| Check | Pass |
|---|---|
| Brak duplikatów ID | ✅/❌ |
| Tylko modele z prefixem `$1` | ✅/❌ |
| Dokładna liczba modeli | ✅/❌ |

---

## 7. Links

- Registry: `open-sse/config/providers/registry/`
- Static models: `src/lib/providers/staticModels.ts`
- Shared config: `open-sse/config/providers/shared.ts`
- Vercel AI Gateway docs: https://vercel.com/ai-gateway/models
