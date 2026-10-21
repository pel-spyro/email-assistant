# Email Reply Assistant — Instrukcja instalacji

Czas: ok. 30 minut. Bez wiedzy technicznej.

---

## CZĘŚĆ 1 — GitHub Pages (hosting plików)

### Krok 1: Stwórz repozytorium

1. Zaloguj się na **github.com**
2. Kliknij **+** (prawy górny róg) → **New repository**
3. Wypełnij:
   - Repository name: `email-assistant`
   - Visibility: **Private** (bezpieczniejsze)
   - ✅ Add a README file
4. Kliknij **Create repository**

### Krok 2: Wgraj pliki

1. W repozytorium kliknij **Add file → Upload files**
2. Wgraj wszystkie 4 pliki:
   - `taskpane.html`
   - `manifest.xml`
   - `commands.html`
   - `icon-80.png` *(dowolna ikona PNG 80×80px — możesz pobrać dowolną z internetu)*
3. Kliknij **Commit changes**

### Krok 3: Włącz GitHub Pages

1. W repozytorium → **Settings** (górna belka)
2. Lewe menu → **Pages**
3. Source: **Deploy from a branch**
4. Branch: **main** → folder: **/ (root)**
5. Kliknij **Save**
6. Po chwili (1–2 min) pojawi się URL:
   `https://TWOJ_USERNAME.github.io/email-assistant/`

### Krok 4: Zaktualizuj manifest.xml

1. W repozytorium otwórz `manifest.xml`
2. Kliknij ✏️ (edytuj)
3. Zamień **wszystkie** wystąpienia:
   - `YOUR_GITHUB_USERNAME` → Twój username GitHub
   - `YOUR_REPO_NAME` → `email-assistant`
4. Kliknij **Commit changes**

> Sprawdź czy działa: wejdź na
> `https://TWOJ_USERNAME.github.io/email-assistant/taskpane.html`
> Powinna się otworzyć strona asystenta.

---

## CZĘŚĆ 2 — Power Automate (AI backend)

### Krok 1: Utwórz nowy flow

1. Wejdź na **make.powerautomate.com**
2. Kliknij **+ Utwórz** → **Natychmiastowy przepływ w chmurze**
3. Nazwa: `Email Reply AI`
4. Wyzwalacz: **Po odebraniu żądania HTTP** → Utwórz

### Krok 2: Skonfiguruj wyzwalacz HTTP

1. Kliknij krok **Po odebraniu żądania HTTP**
2. W polu **Schemat JSON treści żądania** wklej:

```json
{
  "type": "object",
  "properties": {
    "mode":    { "type": "string" },
    "subject": { "type": "string" },
    "from":    { "type": "string" },
    "body":    { "type": "string" },
    "context": { "type": "array" },
    "userName":{ "type": "string" }
  }
}
```

### Krok 3: Dodaj akcję HTTP (wywołanie Claude API)

1. Kliknij **+ Nowy krok** → wyszukaj **HTTP** → wybierz akcję **HTTP**
2. Wypełnij:
   - **Metoda**: POST
   - **URI**: `https://api.anthropic.com/v1/messages`
   - **Nagłówki** (dodaj każdy osobno):
     - `x-api-key` : *(Twój klucz Claude API)*
     - `anthropic-version` : `2023-06-01`
     - `content-type` : `application/json`
   - **Treść** (wklej, kliknij "Przełącz na tryb zaawansowany"):

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": @{if(equals(triggerBody()?['mode'], 'analyze'), concat('Analyze this email and return ONLY valid JSON (no markdown, no explanation).\n\nFormat:\n{\"status\":\"REPLY|ACTION|WAITING|FYI\",\"summary\":\"one sentence\",\"questions\":[{\"question\":\"...\",\"why\":\"...\"}]}\n\nRules:\n- status REPLY = needs a reply\n- status ACTION = action required from sender side\n- status WAITING = waiting for something\n- status FYI = no action needed\n- questions: 0-3 missing context questions. Empty array if enough context.\n- Do not invent information.\n\nEmail:\nSubject: ', triggerBody()?['subject'], '\nFrom: ', triggerBody()?['from'], '\n\n', triggerBody()?['body']), concat('Write a professional email reply draft.\n\nRules:\n1. If any internal decision is pending — write acknowledgement only, no commitment.\n2. Do not promise deadlines or scope changes without explicit confirmation.\n3. Do not invent owners, decisions, or deadlines.\n4. Mark EVERY potential commitment sentence with [REVIEW THIS] at the end.\n5. No subject line — body only.\n6. No filler openings like "I hope this email finds you well".\n7. Sign as: ', triggerBody()?['userName'], '\n\nEmail:\nSubject: ', triggerBody()?['subject'], '\nFrom: ', triggerBody()?['from'], '\n\n', triggerBody()?['body'], '\n\nContext provided:\n', string(triggerBody()?['context'])))}
    }
  ]
}
```

### Krok 4: Dodaj krok Przeanalizuj JSON

1. **+ Nowy krok** → **Przeanalizuj JSON**
2. Zawartość: `Treść` z poprzedniego kroku HTTP
3. Schemat:
```json
{
  "type": "object",
  "properties": {
    "content": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "text": { "type": "string" }
        }
      }
    }
  }
}
```

### Krok 5: Dodaj odpowiedź HTTP

1. **+ Nowy krok** → **Odpowiedź** (HTTP Response)
2. Kod stanu: `200`
3. Nagłówki: `Content-Type` : `application/json`
4. Treść:

```json
@{if(equals(triggerBody()?['mode'], 'analyze'), body('Przeanalizuj_JSON')?['content']?[0]?['text'], json(concat('{"draft":"', replace(replace(body('Przeanalizuj_JSON')?['content']?[0]?['text'], '"', '\"'), decodeUriComponent('%0A'), '\n'), '"}')))}
```

> **Uproszczenie:** Jeśli powyższe wyrażenia Power Automate są zbyt skomplikowane,
> możesz zamiast tego użyć dwóch osobnych flow — jednego dla analizy, drugiego dla draftu.
> Powiedz mi, a zrobię dwa prostsze flow.

### Krok 6: Zapisz i skopiuj URL

1. Kliknij **Zapisz**
2. Wróć do kroku **Po odebraniu żądania HTTP**
3. Skopiuj **HTTP POST URL** — długi link zaczynający się od `https://prod-...`

---

## CZĘŚĆ 3 — Instalacja w Outlook

### Outlook Desktop (Windows)

1. Otwórz Outlook
2. Górna belka → **Pobierz dodatki** (lub **Get Add-ins**)
3. Lewe menu → **Moje dodatki**
4. Na dole → **Dodaj niestandardowy dodatek → Z pliku**
5. Wskaż plik `manifest.xml`
6. Kliknij **Zainstaluj** (zignoruj ostrzeżenie o źródle)

### Outlook Web (outlook.office.com)

1. Otwórz dowolny email
2. Kliknij **...** (trzy kropki) → **Pobierz dodatki**
3. Lewe menu → **Moje dodatki**
4. **Dodaj niestandardowy dodatek → Z adresu URL**
5. Wklej: `https://TWOJ_USERNAME.github.io/email-assistant/manifest.xml`

---

## CZĘŚĆ 4 — Pierwsze uruchomienie

1. Otwórz dowolny email w Outlook
2. W ribbonie (górna belka) powinien pojawić się przycisk **Email Reply**
3. Kliknij → otworzy się panel boczny
4. Przy pierwszym uruchomieniu: wklej **Power Automate URL** z Części 2 Kroku 6
5. Wpisz swoje imię → **Save and start**
6. Gotowe — panel odczyta email automatycznie

---

## Klucz Claude API — jak uzyskać

1. Wejdź na **console.anthropic.com**
2. Zarejestruj się (email + karta płatnicza)
3. **API Keys** → **Create Key**
4. Skopiuj klucz (zaczyna się od `sk-ant-...`)
5. Wklej go w Power Automate w kroku HTTP (nagłówek `x-api-key`)

> ⚠️ Klucz widoczny tylko raz — zapisz go bezpiecznie (np. w managerze haseł).
> Klucz jest przechowywany tylko w Power Automate — nigdy w przeglądarce ani w plikach add-inu.

---

## Podsumowanie kosztów

| Co | Koszt |
|---|---|
| GitHub Pages | **0 zł** |
| Power Automate | **0 zł** (w M365) |
| Claude API | **~$10–15/mies.** przy 50 emailach/dzień |

---

## Problemy?

- **Panel nie pojawia się** → sprawdź czy GitHub Pages jest aktywne i URL działa
- **Błąd analizy** → sprawdź czy Power Automate URL jest poprawny w ustawieniach panelu
- **Brak przycisku w ribbonie** → uruchom ponownie Outlook po instalacji add-inu
