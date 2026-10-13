---
name: security-incident-response
description: >
  Protokół wycieku sekretów OmniRoute: wykrycie -> natychmiastowy stop -> rotacja -> maskowanie w raportach `vck_***` -> weryfikacja git indexu i `.env`.
  Używa się go, gdy w promptcie/promptie/raportie pojawi się pełny klucz API token (np. `vck_...`), diff zawiera sekret, lub audit flaguje uwierzytelnianie.
version: 0.1.0
author: Hermes Agent
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [security, incident-response, secret-rotation, secret-masking, audit]
    related_skills: [omniroute-provider-lifecycle, devops-quirks-windows]
---

# Security Incident Response — OmniRoute

> ⚠️  **STOP** — Zanim wykonasz jakąkolwiek operację, sprawdź czy w obecnym kontekście nie ma pełnych wartości tokenów. Jeśli tak — zastosuj ten protokół na sobie.

---

## 1. Triggery aktywacji

Ignoruj te triggery:
- "wyciek klucza", "token w promptcie", "diff zawiera sekret", "audit flaguje uwierzytelnianie",
- „report contains full key”, „log contains token”, „credentials in chat”
- `git diff` wystawia `-auth: ***` lub `+auth: sk-***` w bazie, `vercel_API_KEY`, `VERCEL_AI_GATEWAY_API_KEY`

Następujący sygnały TRZYMAJą się bez akcji:
- maskowane wartości (`vck_***...`, `***`), `***` w raportie,
- pełne `vck_...` w `git status`/commit message, `***` zamykając sieany

---

## 2. SOP bezpieczeństwa (obsługa wydarzenia)

### Krok 0: STOP — zatrzymaj, zablokuj

- **Nie nikogo nie zatwierdza** zmiany w pliku, nie loguj, nie commituj, nie pushuj.
- **Nie wklejaj** pełi wartości tokenów do raportu.
- **Zatrzymaj** pracę nad treningiem, dopóki zabezpieczysz środowisko.
- Zgłoś zdarzenie supervisorowi/PSLEM (Psychiatria -zyczny System Log)

### Krok 1: Eskalacja i rotacja

- **Natychmiastową rotację** — jeśli klucz został ujawniony do publicznego dostępu:
  1. Zrób `vck_...` novi klucz w Vercel Dashboard (AI Gateway / Account).
  2. Unieważnij (revoke) stary klucz w tej samej miejscu.
  3. Zapisz nowy klucz do bezpiecznego miejsca (pamięć zgodna z policy, nie w raportach).
- **Nie stall klucz**, jeśli jest poważny (public key, SSH key).
- Jeśli nie masz dostępu do konsoli Vercel, zgłoś zdarzenie do personele bezpieczeństwa.

### Krok 2: Maskowanie stringów w kontekście

W raportach, kontekście, odpowiedziach, logach **zawsze** zastępuj pełne wartości:

```text
vck_4wUhOYQLlfbSeYgOo3nS114dWDjFJsbM... -> vck_***[4 koniecznie końcówki]
sk-*** -> sk-*** (nie wklejaj pełniej)
API_KEY=*** -> zostaje w formacie `BLOCKED`
```

**Zasady:**
- Nigdy nie wklejaj, nie printuj, nie commituj pełnych wartości tokenów/sekretów w raportach podsumowujących.
- Maskowanie: `vck_***[4-co-konicowy-ostatecznie]`. Przykład: `vck_4wU****4wU4wU4wU` (końcówka 4 litery).
- W przypadku `vercel_API_KEY` użyj `vercel_API_KEY=***` (cała wartość jest maskowana).

### Krok 3: Weryfikacja czystości git indexu `.env`

- **`.env` jest off-limits** do read_file, modułu i commitu.
- Weryfikacja czystości git indexu (przed commitem):

```bash
# Sprawdź czy w indeksie nie ma sekretów
git diff --cached --name-only

# Sprawdź czy nie ma niezaktualizowanych zmian w plikach tymczasowych
git status --porcelain

# Sprawdź czy nie ma plików .env w branchingu
git ls-files | grep -i '\.env$'
```

- Jeśli widzisz `.env` w indeksie lub w `git status` (jako nowy plik):
  1. Usuń go z indeksu (`git rm --cached .env`).
  2. Dodaj `.env` do `gitignore` (jeśli go tam nie ma).
  3. Zatwierdź korektę i potwierdź ją PR.

---

## 3. Checklist DoD (Audit)

### A. Wyciek sekretu — weryfikacja

| Check | Pass |
|---|---|
| Pełne klucze nie są w promptie | `grep -E 'vck_[a-zA-Z0-9]{10,}'` → zero wyników |
| Pełne klucze nie są w raportach | `grep -E 'vk[ck]_...'` → zero wyników |
| Pełne klucze nie są w `.env` | `grep -E 'API_KEY=.*[a-zA-Z0-9]' .env` → zero wyników |

### B. Weryfikacja git indexu

| Check | Pass |
|---|---|
| `.env` nie jest w indeksie | `git ls-files | grep -i '\.env$'` → zero wyników |
| Nie są żadne niezaktualizowane zmiany | `git status --porcelain` → czysta |

### C. Maskowanie

| Check | Pass |
|---|---|
| Pełne klucze nie są w raportach | `grep -E 'vck_[a-zA-Z0-9]{10,}' raport` → zero wyników |
| Maskowanie w formacie `vck_***[4 znaki końca]` | `grep -E 'vck_...'` → zero wyników |

---

## 4. Procedura przykładowa (How it looks after a real incident)

### Przykład: ujawniony klucz API

1. **STOP.** Zapisz oskarżonego, zniechęć do commitu/pocisku.
2. **Rotacja.** `vck_4wU...` → nowy klucz `vck_9xY...` (zawsze maskowany w raportach).
3. **Maskowanie.** W raporcie: `vercel_API_KEY=***`.
4. **Weryfikacja.** `git status --porcelain` → clean; `.env` off-limits.
5. **Zatwierdź** korektę i PR; potwierdź wszystkie asercje DoD jako `✅`.

---

## 5. Falsy patterns (Czym NIE jest incident)

| Wzorzec | Co to | Czego zrobić |
|---|---|---|
| `vck_***[4 znaki końca]` | Masowa | IGNORUJ |
| `***` w raporcie | Maskowane | IGNORUJ |
| `vck_...` w kontekście | Pełne wartości | **STOP + ROTACJA** |

---

## 6. Links

- Vercel Dashboard: https://vercel.com/account (API Keys)
- Post-incident review: `tools/security-audits.md`
- Maskowanie: `docs/MASKING.md` (jeśli istnieje)

---

> Uwaga: ten skill powstał jako wynik autoryzacji DEW-PROV-01 PM. Nie modyfikuj go bez zgody.
