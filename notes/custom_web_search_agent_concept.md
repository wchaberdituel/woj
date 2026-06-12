# Custom Web Search Agent - Koncepcja

Data: 2026-06-12

---

## Problem wyjściowy

Wbudowane narzędzia `WebSearch` i `WebFetch` w Claude Code wykonują żądania HTTP z serwerów Anthropica (USA). Polskie sklepy internetowe blokują te żądania (HTTP 403). Potrzebne jest własne narzędzie search/scrape, gdzie żądania wychodzą z własnego IP.

---

## Jak działa wbudowany WebSearch

- `WebSearch` i `WebFetch` to hardkodowane narzędzia wewnętrzne Claude Code, nie MCP
- Agent generuje JSON z wywołaniem narzędzia
- Runtime Claude Code wykonuje faktyczne żądanie HTTP z serwerów Anthropica
- Wynik wraca jako tekst do kontekstu agenta
- Brak kontroli użytkownika nad tym mechanizmem

---

## Ścieżka rozważań - ewolucja konceptu

### Etap 1: Opcje podstawowe

Rozważane podejścia:

**A) MCP Server** - własny serwer MCP który Claude Code wywołuje jako narzędzie. Serwer robi HTTP requests z lokalnej maszyny.

**B) Agent przez Anthropic API z tool_use** - własny agent w pętli, samodzielna obsługa wywołań narzędzi. Pełna kontrola, ale wymaga budowy całego frameworku agenta.

**C) Proxy do zewnętrznego Search API** - cienka warstwa przekazująca do Brave Search API / SerpAPI / Google CSE. Omija blokady przez własne IP.

### Etap 2: Mechanizm scrapingu

Kluczowe pytania:
- Czy rozdzielić `search` od `scrape`? → **Tak, to dwa różne działania**
- Search: szybkie, API lub DDG, bez przeglądarki
- Scrape: wolne, wymaga JS, przeglądarka

**Dlaczego rozdzielić:**
- Search zwraca listę URL + snippety bez uruchamiania przeglądarki
- Agent używa search żeby znaleźć *gdzie*, scrape żeby *co* przeczytać
- Otwieranie przeglądarki dla każdego zapytania search = wolno i niepotrzebnie

**Silnik scrapingu:**
- `requests` + BeautifulSoup → tylko statyczny HTML, nie obsługuje JS
- Playwright headless → pełne JS, najlepsza opcja dla dynamicznych stron
- Playwright + `connect_over_cdp` → podłączenie do istniejącej instancji Chrome

**Co zwracać agentowi:** surowy HTML to zły pomysł. Agent powinien dostać czysty tekst lub strukturę JSON - nie parsować HTML samemu.

### Etap 3: Prawdziwa przeglądarka przez CDP

Kluczowa obserwacja: **prawdziwa przeglądarka nie jest blokowana** bo świat widzi prawdziwy fingerprint, prawdziwe cookies, prawdziwy profil użytkownika.

Chrome z flagą `--remote-debugging-port=9222` wystawia CDP (Chrome DevTools Protocol). Playwright `connect_over_cdp()` podłącza się do już działającej instancji.

```
Agent (Claude) → MCP Server → Playwright connect_over_cdp() → Twój Chrome
```

**Przewagi:**
- Realny fingerprint (Canvas, WebGL, fonts, plugins)
- Istniejące cookies i sesje
- Użytkownik może przejąć kontrolę gdy potrzeba (login, captcha)

### Etap 4: Hierarchia trybów z eskalacją

```
1. HEADLESS (default)
   szybko, bezszelestnie, nowe środowisko Playwright
        ↓ jeśli wykryto blokadę / captcha / login wall
2. REAL BROWSER (CDP)
   istniejący Chrome, cookies użytkownika, realny fingerprint
        ↓ jeśli agent nie może sam obsłużyć (2FA, trudna captcha)
3. HUMAN ACTION
   agent pauzuje → powiadamia użytkownika → czeka na sygnał "done"
```

**Wykrywanie blokady (automatyczne przejście 1→2):**
- HTTP status 403/429/503
- Selektory: `.captcha`, `#cf-challenge`, `[data-recaptcha]`, formularze CAPTCHA
- Redirect na `/login`, `/signin`
- Treść strony zawiera "robot", "bot detected", "verify you are human"
- Pusty DOM przy załadowanej stronie (SPA nie wyrenderował)

### Etap 5: Izolowane środowisko agenta

**Finalna architektura:**

```
Agent (Claude)
    ↓ MCP tools
MCP Server
    ↓ CDP / Playwright
Przeglądarka agenta (Chrome na maszynie agenta)

Użytkownik ← VNC/RDP ── pulpit maszyny agenta ── przeglądarka agenta
```

**Co to daje:**
- Agent ma własną izolowaną przeglądarkę - nie miesza się z przeglądarką użytkownika
- Użytkownik obserwuje i interweniuje przez zdalny pulpit
- Interwencja naturalna: użytkownik widzi dokładnie co widzi agent, działa w tej samej przeglądarce
- Jeśli maszyna agenta ma polskie IP (VPS w PL) → nawet headless nie jest blokowany

**Kanał eskalacji i sygnał "done":** rozwiązany przez dodatkowe endpointy / kanał komunikacyjny (szczegóły implementacji poza zakresem tej notatki - zakładamy że działa).

---

## Finalna decyzja: zestaw narzędzi

```
SEARCH (bez przeglądarki, przez API):
  search(query, num=5)
    → [{url, title, snippet}]

NAWIGACJA I DOM:
  open_page(url)
    → {status, title, mode_used, blocked: bool}
    wewnętrznie: headless first, CDP jeśli wykryto blokadę

  get_text(selector=None)
    → tekst całej strony lub konkretnego elementu

  get_elements(selector)
    → [{text, href, attrs}]  ← do ustalenia: jakie atrybuty zawsze, jakie opcjonalnie

  click(selector)
    → potrzebne do cookie bannerów, paginacji, rozwijanych list

  wait_for(selector, timeout=5000)
    → czekaj aż element pojawi się w DOM (lazy loading, AJAX)

ESKALACJA:
  request_human(reason, hint=None)
    → wysyła powiadomienie do użytkownika
    → blokuje i czeka na sygnał "done"
    → agent kontynuuje od miejsca gdzie skończył

FALLBACK:
  screenshot()
    → base64 PNG gdy DOM nie daje sensu
    → agent "widzi" co jest na ekranie
```

---

## Stan sesji w MCP server

Serwer MCP **utrzymuje stan między wywołaniami** - nie tworzy nowej przeglądarki na każde wywołanie:

```
MCP server startup:
  → uruchamia headless Playwright (zawsze gotowy)
  → opcjonalnie: connect_over_cdp (jeśli Chrome z CDP działa)

session state:
  → current_page (aktywna zakładka)
  → current_mode (headless / cdp)
  → browser context (cookies, local storage)
```

Dzięki temu `open_page` → `get_elements` → `click` → `get_elements` to sekwencja na tej samej stronie.

---

## Otwarte pytania (nierozstrzygnięte)

1. **Co wraca z `get_elements`?**
   - Tylko `innerText`?
   - Tekst + wybrane atrybuty (`href`, `data-*`)?
   - Zagnieżdżony JSON struktury DOM?
   - Kto decyduje jakie atrybuty - zawsze te same, czy agent podaje listę?

2. **Kto wykrywa blokadę - serwer MCP czy agent?**
   - Automatyczna detekcja w MCP (przezroczysty fallback) - agent nie musi o tym wiedzieć
   - Agent sam ocenia na podstawie treści którą dostał i decyduje o eskalacji
   - Hybryda: MCP wykrywa oczywiste blokady, agent decyduje o nieoczywistych

3. **Lifecycle sesji przeglądarki**
   - Jedna przeglądarka na cały czas życia serwera MCP?
   - Reset kontekstu (cookies) między zadaniami?
   - Jak obsługiwać równoległe zadania?

---

## Opcje pominięte / odrzucone

| Opcja | Dlaczego pominięta |
|---|---|
| Zewnętrzne proxy do search API jako jedyne rozwiązanie | Nie rozwiązuje problemu scrapingu stron wymagających JS |
| Własny agent przez API (bez Claude Code) | Wymaga budowy całego frameworku - większy zakres |
| Screenshot jako główny interfejs (computer-use) | Wolniejsze, mniej precyzyjne; DOM-first jest lepszy dla znanych stron |
| Hardkodowane mappingi DOM per-sklep jako jedyne narzędzie | Nieelastyczne, wymaga utrzymania; warto jako opcjonalne uzupełnienie |
| `playwright-stealth` zamiast prawdziwej przeglądarki | Wystarczy dla wielu przypadków ale nie wszystkich; prawdziwa przeglądarka bardziej niezawodna |

---

## Podsumowanie architektury

```
[search API / DDG]
       ↓
  search(query) → lista URL

[Maszyna agenta - VPS lub lokalna]
       ↓
  MCP Server (Python/Node)
       ↓
  Playwright
  ├── headless (default, szybki)
  └── connect_over_cdp (fallback, realny fingerprint)
       ↓
  Browser (headed Chrome)
       ↓
  open_page / get_text / get_elements / click / wait_for
       ↓
  request_human → [kanał eskalacji] → Użytkownik przez VNC/RDP
```
