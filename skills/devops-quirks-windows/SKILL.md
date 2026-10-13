---
name: devops-quirks-windows
description: >
  Specyfika Windows/Git Bash: ścieżki, CRLF, wywołania `tsx`/npm w środowisku lokalnym.
  Używa się go przy pracy z OmniRoute na Windowsie, gdy występują problemy ze zgodnością ścieżek, linii końcowych lub konwencji katalogów.
version: 0.1.0
author: Hermes Agent
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [windows, devops, path, crlf, git-bash, msys, npm, tsx, environment]
    related_skills: [security-incident-response, omniroute-provider-lifecycle]
---

# DevOps Quirks — Windows (Git Bash / MSYS)

> Zasada: Windows to nie jest środowisko uniwersalne. Git Bash (MSYS) i PowerShell się różnią. NPM, TSX i ścieżki plików mają swoje konwencje. Nie założono, że zachowanie w Linuxu będzie identyczne.

---

## 1. Błędy typu PATH vs MSYS

| Sytuacja | Co dzieje się | Co powinno być |
|---|---|---|
| `git -C /c/Users/x ...` | Failed: cannot change to directory | Użyj `cd /c/Users/x` (bash builtin) bez `git -C` |
| `node /tmp/a.js` | Failed: not found | Użyj `node C:/tmp/a.js` (native path, forward slashes) |
| `bash` vs PowerShell | PS: `Get-ChildItem`, `$env:FOO` | Bash: `ls`, `$FOO` |
| `mkdir -p` w git-bash | Tworzy folder lokalnie | Nie działa w PowerShell (użyj `mkdir` lub `New-Item`) |

### Rule of thumb

- **Native tools** (node, git, npm, tsx): pass native paths `C:/Users/x/...`, never MSYS paths like `/c/Users/x`.
- **Shell builtins** (`cd`, `ls`, `mkdir -p`): use POSIX syntax inside bash.
- **PowerShell builtins** (`Get-ChildItem`, `$env:FOO`, `Select-String`): never use in a bash terminal.

---

## 2. Ścieżki i katalogi

| Kontekst | Format | Przykład |
|---|---|---|
| Bash (git-bash/MSYS) | `/c/Users/x/...` lub `~` | `ls ~/Desktop/Hermes/...` |
| Native Windows tools | `C:\Users\x\...` w native sense, but forward slashes in command args | `git -C C:/Users/x/...` |
| PowerShell (not used) | `C:\Users\x\...` | `Get-ChildItem C:\Users\x` |

### Rules for scratch files

- Use `C:/Users/<user>/AppData/Local/heroes/cache/scratch` for temporary files.
- Never use bare `/tmp`. On Windows, `/tmp` is not accessible to native tools.
- Use `$LOCALAPPDATA/Temp` or `$TMPDIR` (set dynamically) for scratch files.

---

## 3. Linia końcowa (`CRLF` vs `LF`)

| Operacja | Znaleziona | Co zrobić |
|---|---|---|
| `git config --global core.autocrlf true` | Changes LF → CRLF on checkout | Suitable for Windows repos |
| `git config --global core.autocrlf input` | LF → CRLF on commit, CRLF → LF on checkout | Mixed filesystem (often used in cross-platform dev) |
| `git config --global core.autocrlf false` | No conversion | Unix-like systems only |

### Check configured values

```bash
git config --global core.autocrlf
```

### Recommended for OmniRoute

- Keep consistent line endings across the whole repo.
- Configure editor to save with `LF` (or `CRLF` depending on your preference) and ignore others.

---

## 4. Wywołania `tsx` / `npm` w środowisku lokalnym

### Testing

```bash
# Test a single test file (node:test + tsx esm transform)
node --import tsx/esm --test --test-concurrency=1 tests/unit/repro-10990-v0-vercel-web-static-models.test.ts
```

### NPX not found

- Some projects have no `npx` binary. Use `node --import tsx/esm` directly.
- The vite/vitest binary may be missing in `node_modules/.bin`. Use:
  ```bash
  npx vitest -> node --import tsx/esm --test
  ```

### Environment variables

- `.env` reads: `process.env` in Node. In bash, only environment variables at module load time are visible.
- `bash` reads `.env` via `set -a; source .env; set +a` (if the file is key=VALUE pairs).

---

## 5. Checklist DoD

| Wire | Check | Pass |
|---|---|---|
| `git config --global core.autocrlf` returns `true`, `input` or `false` | No mixed line endings | ✅/❌ |
| Node runs from bash (`node --version`) | Version matches your expected Node.js | ✅/❌ |
| `which tsx` returns a path | TSX available | ✅/❌ |
| `which npm` returns a path | NPM available | ✅/❌ |
| Scratches in `C:/Users/<user>/AppData/Local/hermes/cache/scratch` | All temp files in this directory | ✅/❌ |

---

## 6. Links

- Node.js docs: https://nodejs.org/
- TypeScript: https://www.typescriptlang.org/
- Testing with Node.js: https://nodejs.org/api/test.html
- Git Bash / MSYS: https://git-scm.com/
