# System automatyzacji e-commerce + smart-home — materiał do rozmowy (PL)

> Cel pliku: przygotowanie do rozmowy o pracę jako **inżynier automatyzacji** (Dubaj). Zawiera: co zbudowałem,
> jak to działa, co potrafi, jak jest zbudowane, kluczowe wyzwania inżynierskie oraz zestaw pytań rekrutacyjnych
> z odpowiedziami. Wersja angielska: `system-do-rozmowy-EN.md`.

---

## 1. Elevator pitch (30 sekund)
Zaprojektowałem i wdrożyłem **produkcyjną platformę automatyzacji** łączącą sprzedaż e-commerce (BaseLinker,
Allegro, PrestaShop), księgowość (wFirma), hurtownie (Arte) i inteligentny dom (Home Assistant), z warstwą AI
(asystent głosowy + chatbot na Claude). Rdzeń orkiestracji to **Home Assistant + n8n**, integracje przez **REST
API, webhooki i OAuth**, a logika biznesowa (marża, synchronizacja stanów/cen, dokumenty magazynowe) działa
bezobsługowo z monitoringiem i alertami. Efekt: dziesiątki godzin miesięcznie oszczędności i mniej błędów
ręcznych.

## 2. Architektura (wysokopoziomowo)
Warstwy:
- **Orkiestracja / hub:** Home Assistant (HA) — automatyzacje, encje, dashboardy, wystawianie usług.
- **Silnik workflow:** n8n (dodatek do HA) — przepływy z węzłami Code (JS), webhooki, cron, cache w `staticData`.
- **Endpointy publiczne:** PHP na hostingu Kylos (`bot.stylmebli.pl/*.php`) — tam, gdzie potrzebny publiczny
  HTTPS dostępny z zewnątrz (przeglądarka, BaseLinker, sklep).
- **Dane/analityka:** InfluxDB (szereg czasowy) + Grafana (wykresy w dashboardach HA).
- **AI:** Anthropic Claude (asystent głosowy Nova + chatbot sklepu), natywne narzędzia (web_search), prompt
  caching; OpenAI Codex jako niezależny recenzent.
- **Frontend/klient:** custom karty Lovelace (JS), wtyczka Chrome (MV3), widget czatu (JS) w sklepie.

Przepływ danych (przykład — marża zamówienia):
`BaseLinker (nowe zamówienie) → n8n (pobiera koszt: grupa cenowa/wFirma + prowizję z Allegro Billing) → InfluxDB
→ dashboard HA`. Dla podglądu na żądanie: `przeglądarka (panel BaseLinker) → wtyczka → endpoint PHP (Kylos, CORS+token)
→ HA (Nabu Casa remote, LLAT) → n8n/InfluxDB`.

Uwierzytelnianie integracji (świadome decyzje):
- **wFirma:** metoda kluczy API (accessKey/secretKey/appKey) zamiast OAuth2 — bo OAuth2 wymagał restrykcji IP,
  a publiczny IP dostawcy jest zmienny. Klucze = zero IP, zero wygasania, zero wyścigu tokenów.
- **Allegro:** OAuth2 z rotującym refresh_tokenem, per konto sprzedażowe; token odświeżany centralnie.
- **MF „Biała lista VAT":** publiczne API, bez klucza.
- **HA ↔ Kylos:** Long-Lived Access Token przez Nabu Casa remote.

## 3. Co zbudowałem (moduły)

### 3.1 Nova — asystent głosowy (Home Assistant + Claude)
- Pipeline: STT (chmura Nabu Casa) → agent Claude → TTS. Aktywacja: tap-to-talk lub słowo „Hej Nova".
- Custom karta Lovelace „oko" (SVG/Canvas, stany + animacje), wersjonowana.
- Funkcje: sterowanie domem (światła/klimatyzacja/Tesla/brama/kamery/media/odkurzacz/żaluzje), **trwała pamięć**
  faktów (lista todo, kasowanie za hasłem+PIN weryfikowanym po stronie HA), scenariusze wieloetapowe („Idę na
  górę"), geofencing „dom pusty", alerty krytyczne (odznaka w oku), historia encji, „gdzie jest X" (radar),
  **pytania sprzedażowe głosem** (BaseLinker), sterowanie globusem 3D „Oko Boga".

### 3.2 BaseLinker ↔ wFirma (sprzedaż, magazyn, marża)
- **Synchronizacja cen zakupu** (kaskada źródeł: grupa cenowa 781 → average_cost → dostawca → cennik hurtowni →
  wFirma netto) i **liczenie marży** per zamówienie (przychód netto − koszt − prowizja Allegro − Smart), zapis do
  InfluxDB, wykresy na dashboardzie.
- **Wtyczka Chrome (MV3)** wstrzykująca w panelu BaseLinker: badge stanu magazynowego wFirma (dostępne/przyjęte PZ)
  oraz **chip realnej marży** zamówienia — liczone na żądanie przez endpoint PHP.
- **Synchronizacja produktów** do kartotek wFirma (nowe produkty ze skanera → wFirma) + wskaźnik pokrycia.
- **Zwroty paragonowe → PZ:** automatyczne utworzenie dokumentu PZ w wFirma + protokołu PDF przy zwrocie.
- **Alerty** „synchronizacja stanęła" (binary_sensor + powód) na realnych sygnałach, nie na mylącym procencie.

### 3.3 Import z hurtowni Arte
- Import produktów z feedu do BaseLinker, **opisy generowane AI** (multi-provider: Claude/OpenAI/Gemini),
  automatyczne **czyszczenie nieprawidłowych EAN** (odrzucanie zakresu wewnętrznego GS1 20-29, „None", złej długości)
  zanim trafią do Allegro.

### 3.4 stylmebli.pl (sklep PrestaShop 1.6)
- **Chatbot AI** (Claude) na wszystkich stronach: status zamówienia (weryfikacja 2FA e-mail+telefon), pytania o
  produkty (kaskada: baza sklepu → web_search → strona producenta), akcesoria/kompatybilność, FAQ, zbieranie
  leadów (push na telefon przez HA), wielojęzyczność (PL+5), prompt caching, twarde limity i anti-jailbreak,
  poufność (nie ujawnia dostawcy ani cen hurtowych — filtr deterministyczny).
- **Checkout: dane z NIP** — po wpisaniu NIP pobranie danych firmy z MF „Białej listy VAT" (walidacja sumy
  kontrolnej NIP), autofill firmy/adresu + status VAT.
- **GEO/AEO:** schema.org JSON-LD (Organization/WebSite/Product/Offer/BreadcrumbList/FAQPage) dla AI-search i
  rich results Google; naprawa aggregateRating; robots.txt + sitemapa.
- **Automat opinii** (mail po dostawie + deep-link `opinia.php` mapujący SKU→produkt + moderacja).
- **Porządkowanie filtrów** (scalanie duplikatów Cech, sortowanie, mobile) + **cron prewencji** (n8n 3:00).
- **Modernizacja UI** całego sklepu (belka, karta produktu, listingi) wstrzykiwana bezpiecznie przez `footer.tpl`.

### 3.5 Infrastruktura, obserwowalność, dokumentacja
- Dashboardy energii/PV, dom, sprzedaż (InfluxDB+Grafana), alerty, harmonogramy.
- **Dokumentacja** per-narzędzie (`/config/docs/`, markdown + lustro w HA jako klikalne podwidoki) + RUNBOOK
  (odtwarzanie od zera, tabele ID, sekrety) + Artifacty (dashboardy, strona „Narzędzia").
- **Backup:** HA (codzienny, szyfrowany, off-site Nabu Casa) + **nocny eksport workflowów n8n** do `/config`
  (bo dodatki NIE są w backupie HA) + lustro pamięci w repo git.

## 4. Jak jest zbudowane (stack + wzorce)
- **Home Assistant:** konfiguracja w pakietach YAML (`packages/*.yaml`), custom karty Lovelace w czystym JS
  (bez zależności), edycja dashboardów przez websocket (`lovelace/config/save`), usługi `rest_command`/`script`
  z `response_variable`, ekspozycja encji do Assist w `core.entity_registry`.
- **n8n:** węzły Code (JS, `this.helpers.httpRequest`), webhooki (responseMode lastNode), cron, cache i stan w
  `getWorkflowStaticData('global')`; deploy przez Public API (deactivate → PUT → activate, żeby przeładować kod
  bez wyścigu o `staticData`).
- **PHP (Kylos):** samodzielne endpointy (bez frameworka), CORS zawężony do konkretnego origin, autoryzacja
  tokenem (`hash_equals`), sekrety w pliku poza web-rootem; zgodność z PHP 7.0 w katalogu `bot/`.
- **Wtyczka Chrome MV3:** content-script na `panel.baselinker.com`, MutationObserver (SPA), batch fetch + cache,
  wstrzykiwanie badge'y do DOM.
- **AI (Claude):** narzędzia (tool use) mapowane na skrypty HA, wymuszanie `web_search` deterministycznie w
  pętli narzędzi, prompt caching (statyczny trzon + `cache_control`), filtr wyjścia (`humanize`).
- **Weryfikacja:** headless Chromium + wstrzyknięty token do zrzutów żywego UI (HA i strony www); `node --check`,
  `php -l`, empiryczne testy ścieżki błędu na żywej instancji.

## 5. Kluczowe wyzwania inżynierskie (i jak rozwiązane)
- **Wyścig o rotujący token OAuth (wFirma/Allegro).** n8n zapisuje CAŁY `staticData` na końcu każdego przebiegu;
  przy cronie co 5 min + webhookach przebieg, który wczytał stary token, nadpisywał świeży → invalid_grant.
  Rozwiązania: (a) dedykowany „token manager" jako jedyny właściciel rotującego tokenu (konsumenci tylko czytają
  read-only), (b) docelowo migracja wFirma na metodę kluczy API (zero rotacji). Nauka: przy zapisie
  współdzielonego rotującego sekretu — najpierw deactivate + drenaż przebiegów, potem PUT, potem activate.
- **Cichy błąd prowizji Allegro.** Migracja nagłówków (sed) przypadkiem podmieniła w wywołaniu Allegro Billing
  `Authorization: Bearer` na klucze wFirma → 401 łapany w `catch` → prowizja = 0 dla WSZYSTKICH zamówień przez
  miesiąc (dashboard zawyżony). Wykryte przy weryfikacji (1 na 338 zamówień miało prowizję). Naprawa: właściwy
  Bearer; nauka: Allegro=Bearer, wFirma=klucze — nie mieszać; cichy `catch` może maskować systemowy błąd.
- **Poprawność marży a VAT.** Ustalenie definicji: sprzedaż/koszt netto, prowizja jako-zwrócona; świadoma decyzja
  biznesowa (brutto vs netto) po pokazaniu trzech metod i realnego wpływu VAT na marżę.
- **Backup stanu spoza gita.** Logika n8n żyje tylko w bazie dodatku, a dodatki NIE są w backupie HA
  (`include_addons: []`). Rozwiązanie: nocny eksport workflowów do `/config` (trafia do szyfrowanego backupu HA),
  gitignored (bo nody mają klucze).
- **Ograniczenia środowiska.** Kontener-w-kontenerze: `bwrap` sandbox Codeksa pada; do zrzutów UI headless
  Chromium wymaga `--no-sandbox`; brak LAN → wszystko przez API HA. Świadome obejścia, nie „łatanie na ślepo".

## 6. Bezpieczeństwo, niezawodność, DR
- **Sekrety** nigdy w gicie/kodzie promptu: `secrets.yaml` (gitignored), config na Kylos poza web-rootem,
  zaszyfrowane config-entry integracji, klucze w bazie n8n. Endpointy: token + `hash_equals` + CORS.
- **Chatbot:** limity per-IP/dobowe, anti-jailbreak, żaden sekret nie trafia do kontekstu modelu, SQL
  parametryzowany, twardy limit kosztów na koncie AI.
- **Niezawodność:** idempotentne crony, alerty na realnych sygnałach, health-checki, retry z cooldownem przy
  limitach API (throttle wFirma), fallbacki w kaskadach (koszt/źródła danych).
- **DR:** codzienny szyfrowany backup HA off-site + eksport workflowów n8n + mirror pamięci + RUNBOOK do
  odtworzenia od zera (kroki + tabele ID).

## 7. Metryki / skala (przykłady)
- ~400 zamówień śledzonych w InfluxDB; 338 z Allegro; katalog ~4700 produktów Arte; ~9500 kartotek wFirma po
  synchronizacji; sklep ~2200 URL produktów; wtyczka i chatbot na produkcji.

## 8. Zestaw pytań rekrutacyjnych z odpowiedziami

**Q1. Opisz w skrócie system, który zbudowałeś.**
Produkcyjna platforma automatyzacji e-commerce + smart-home: HA jako hub, n8n jako silnik workflow, integracje
przez REST/OAuth/webhooki, warstwa AI (Claude) do asystenta głosowego i chatbota, dane w InfluxDB/Grafana,
publiczne endpointy PHP tam, gdzie potrzebny dostęp z zewnątrz. Automatyzuje sprzedaż (marża, sync cen/stanów,
dokumenty magazynowe, zwroty), obsługę klienta i sterowanie domem.

**Q2. Dlaczego Home Assistant + n8n, a nie np. jeden monolit?**
HA daje gotową warstwę encji, harmonogramów, UI i integracji domowych; n8n daje wizualne, łatwe w utrzymaniu
przepływy z kodem tam, gdzie trzeba. Rozdzielenie: HA orkiestruje i prezentuje, n8n wykonuje logikę integracji.
Endpointy PHP dokładam tylko tam, gdzie potrzebny publiczny HTTPS (przeglądarka/sklep nie dosięgną wewnętrznego n8n).

**Q3. Jak bezpiecznie zarządzasz uwierzytelnianiem do wielu API?**
Per system dobieram metodę: wFirma — klucze API (bez IP/rotacji), Allegro — OAuth2 z centralnym odświeżaniem
rotującego refresh_tokenu, MF — publiczne. Sekrety poza gitem i poza kontekstem modelu. Kluczowa nauka: przy
współdzielonym rotującym tokenie trzeba wyeliminować wyścig (jeden właściciel zapisu) — inaczej invalid_grant.

**Q4. Opowiedz o trudnym bugu, który zdebugowałeś.**
Prowizja Allegro = 0 dla wszystkich zamówień przez miesiąc. Objaw: dashboard marży zawyżony. Diagnoza: tylko 1
z 338 zamówień miało prowizję > 0 (twardy dowód z InfluxDB). Root cause: masowa podmiana nagłówków (sed) wstawiła
w wywołaniu Allegro Billing klucze wFirma zamiast `Bearer`, a błąd 401 był łykany w `catch`. Naprawa + reguła
„Allegro=Bearer, wFirma=klucze". Wniosek: ciche `try/catch` maskują błędy — trzeba weryfikować danymi.

**Q5. Jak liczysz marżę zamówienia?**
Przychód netto − koszt netto − prowizja Allegro − Smart. Koszt z kaskady źródeł z fallbackami. Prowizja z Allegro
Billing API (rzeczywista, nie szacunek). Zwróciłem uwagę na spójność VAT (netto vs brutto) i przedstawiłem
decyzję biznesową zamiast narzucać.

**Q6. Jak zapewniasz niezawodność i monitoring?**
Idempotentne crony, health-checki, alerty na realnych sygnałach (a nie na plateau procentu), retry z cooldownem
przy throttlingu API, fallbacki w kaskadach, dashboardy z wykresami trendów. Alert HA (binary_sensor) + powód do
popupu, push na telefon.

**Q7. Jak podchodzisz do wdrażania zmian w produkcji?**
Przed zmianą: commit + „blast-radius check" (grep wszystkich konsumentów encji/usługi), sprawdzenie fallbacków,
empiryczny test ścieżki błędu na żywej instancji, check_config, dopiero potem reload/restart. Zmiany istotne
konsultuję z góry.

**Q8. Jak testujesz i weryfikujesz?**
`node --check`/`php -l` dla składni, testy jednostkowe logiki na realnych danych, headless Chromium do zrzutów
żywego UI (weryfikacja wizualna bez proszenia o screenshoty), oraz — kluczowe — empiryczne odtworzenie ścieżki
błędu, nie tylko happy path. Dodatkowo niezależny przegląd przez drugi model (Codex/GPT).

**Q9. Jak wygląda backup / disaster recovery?**
Codzienny szyfrowany backup HA (lokalnie + off-site Nabu Casa) obejmujący cały `/config`. Odkryłem, że dodatki
(n8n/InfluxDB) NIE są w backupie, więc dorobiłem nocny eksport workflowów n8n do `/config`. Do tego RUNBOOK z
krokami odtworzenia i tabelami ID (webhooki, workflowy, endpointy) + gdzie leżą sekrety.

**Q10. Jak zapewniasz bezpieczeństwo publicznego endpointu i chatbota?**
Token + `hash_equals`, CORS zawężony do origin, sekrety poza web-rootem. Chatbot: limity per-IP/dobowe,
anti-jailbreak, żaden sekret w kontekście modelu, SQL parametryzowany, poufność (nie ujawnia dostawcy/cen
hurtowych — deterministyczny filtr), twardy limit kosztów.

**Q11. Jak zintegrowałeś AI w sposób produkcyjny (koszt/jakość)?**
Model dobrany do zadania (Haiku do głosu/latencji, Sonnet do jakości chatbota), prompt caching (~89% oszczędności
na wejściu przy powtarzalnym prompcie), wymuszanie narzędzi deterministycznie w kodzie (bo sam prompt bywa
loterią), filtr wyjścia gwarantujący reguły niezależnie od modelu, limity kosztów.

**Q12. Jak byś to skalował / zrobił wielo-najemcowe (multi-tenant)?**
Wtyczka/endpoint z kluczem licencyjnym per klient zamiast jednego zestawu kluczy; rozdzielenie danych per tenant;
kolejkowanie i rate-limiting; obserwowalność per klient; CI/CD i wersjonowanie; docelowo produktyzacja flagowca
(wtyczka wFirma) jako subskrypcja.

**Q13. Największe ograniczenie/kompromis w projekcie?**
Środowisko (kontener-w-kontenerze) ogranicza sandbox/LAN — obchodzę świadomie. Marża w InfluxDB liczona raz przy
zamówieniu (prowizja bywa księgowana później) — dlatego chip liczy na żądanie; docelowo retry/recompute.

**Q14. Czego się nauczyłeś?**
Że w automatyzacji integracyjnej wygrywają: idempotencja, jedno źródło prawdy dla stanu, eliminacja wyścigów,
weryfikacja danymi (nie założeniami), i dobra dokumentacja/DR — bo systemy żyją latami i muszą być odtwarzalne.

**Q15. Dlaczego chcesz pracować jako inżynier automatyzacji?**
Bo lubię zamieniać powtarzalną, błędogenną pracę ręczną w niezawodne, monitorowane procesy — i mam to
udokumentowane realnymi wdrożeniami end-to-end (od API i baz danych po UI, AI i DR).

## 9. Słowniczek pojęć
- **Orkiestracja** — koordynacja usług/przepływów (tu: HA).
- **Webhook / cloudhook** — URL wyzwalający przepływ; cloudhook = publiczny URL Nabu Casa → webhook HA.
- **OAuth2 / refresh token** — token dostępu odświeżany rotującym refresh_tokenem (ryzyko wyścigu).
- **Idempotencja** — wielokrotne wykonanie daje ten sam efekt (bezpieczne crony/retry).
- **Line protocol / InfluxDB** — format zapisu szeregów czasowych (tag=wymiar, field=wartość).
- **MV3 content-script** — skrypt wtyczki wstrzykiwany w stronę.
- **Prompt caching** — cache statycznej części promptu → tańsze wywołania LLM.
- **Blast-radius check** — sprawdzenie wszystkich miejsc dotkniętych zmianą przed wdrożeniem.
