# E-commerce + Smart-Home Automation Platform — Interview Prep (EN)

> Purpose: preparation for an **Automation Engineer** interview (Dubai). Contains: what I built, how it works,
> what it does, how it is built, key engineering challenges, and an interview Q&A set. Polish version:
> `system-do-rozmowy-PL.md`.

---

## 1. Elevator pitch (30 seconds)
I designed and shipped a **production automation platform** connecting e-commerce (BaseLinker, Allegro,
PrestaShop), accounting (wFirma), a wholesaler feed (Arte) and a smart home (Home Assistant), with an AI layer
(voice assistant + customer chatbot on Claude). The orchestration core is **Home Assistant + n8n**; integrations
run over **REST APIs, webhooks and OAuth**; business logic (margin, price/stock sync, warehouse documents) runs
hands-free with monitoring and alerting. Result: dozens of hours saved monthly and fewer manual errors.

## 2. Architecture (high level)
Layers:
- **Orchestration / hub:** Home Assistant (HA) — automations, entities, dashboards, exposed services.
- **Workflow engine:** n8n (HA add-on) — flows with Code nodes (JS), webhooks, cron, cache in `staticData`.
- **Public endpoints:** PHP on Kylos hosting (`bot.stylmebli.pl/*.php`) — where public HTTPS reachable from
  outside is required (browser, BaseLinker, shop).
- **Data/analytics:** InfluxDB (time series) + Grafana (charts embedded in HA dashboards).
- **AI:** Anthropic Claude (Nova voice assistant + shop chatbot), native tools (web_search), prompt caching;
  OpenAI Codex as an independent reviewer.
- **Frontend/client:** custom Lovelace cards (JS), Chrome extension (MV3), chat widget (JS) on the shop.

Example data flow (order margin):
`BaseLinker (new order) → n8n (resolves cost: price group/wFirma + commission from Allegro Billing) → InfluxDB →
HA dashboard`. On-demand view: `browser (BaseLinker panel) → extension → PHP endpoint (Kylos, CORS+token) → HA
(Nabu Casa remote, LLAT) → n8n/InfluxDB`.

Integration authentication (deliberate choices):
- **wFirma:** API-key method (accessKey/secretKey/appKey) instead of OAuth2 — OAuth2 required an IP restriction
  while the provider's public IP is dynamic. Keys = no IP, no expiry, no token race.
- **Allegro:** OAuth2 with a rotating refresh token, per selling account; refreshed centrally.
- **MF "VAT White List":** public API, no key.
- **HA ↔ Kylos:** Long-Lived Access Token via Nabu Casa remote.

## 3. What I built (modules)

### 3.1 Nova — voice assistant (Home Assistant + Claude)
- Pipeline: STT (Nabu Casa cloud) → Claude agent → TTS. Activation: tap-to-talk or "Hej Nova" wake word.
- Custom "eye" Lovelace card (SVG/Canvas, animated states), versioned.
- Features: home control (lights/AC/Tesla/gate/cameras/media/vacuum/blinds), **persistent memory** of facts
  (todo list; deletion behind a password+PIN verified on the HA side), multi-step scenes ("I'm going upstairs"),
  "empty house" geofencing, critical alerts (eye badge), entity history, "where is X" (radar), **voice sales
  queries** (BaseLinker), and voice control of a 3D globe ("God's Eye").

### 3.2 BaseLinker ↔ wFirma (sales, stock, margin)
- **Purchase-price sync** (source cascade: price group 781 → average_cost → supplier → wholesaler feed → wFirma
  net) and **margin calculation** per order (net revenue − cost − Allegro commission − Smart fee), written to
  InfluxDB, charted on a dashboard.
- **Chrome extension (MV3)** injecting into the BaseLinker panel: a wFirma stock badge (available / goods-receipt
  posted) and a **real order-margin chip** — computed on demand via the PHP endpoint.
- **Product sync** into wFirma records (new scanned products → wFirma) + coverage indicator.
- **Receipt returns → PZ:** auto-creation of a wFirma goods-receipt (PZ) document + a PDF return protocol.
- **"Sync stalled" alerts** (binary_sensor + reason) based on real signals, not a misleading percentage.

### 3.3 Arte wholesaler import
- Product import from the feed into BaseLinker, **AI-generated descriptions** (multi-provider:
  Claude/OpenAI/Gemini), and automatic **cleaning of invalid EANs** (rejecting the GS1 internal range 20-29,
  "None", wrong length) before they reach Allegro.

### 3.4 stylmebli.pl (PrestaShop 1.6 shop)
- **AI chatbot** (Claude) on all pages: order status (2FA email+phone verification), product questions (cascade:
  shop DB → web_search → manufacturer page), accessories/compatibility, FAQ, lead capture (phone push via HA),
  multilingual (PL+5), prompt caching, hard limits and anti-jailbreak, confidentiality (never reveals the
  supplier or wholesale prices — deterministic filter).
- **Checkout: data from NIP (tax ID)** — after entering the NIP, fetch company data from the MF "VAT White List"
  (NIP checksum validation), autofill company/address + VAT status.
- **GEO/AEO:** schema.org JSON-LD (Organization/WebSite/Product/Offer/BreadcrumbList/FAQPage) for AI search and
  Google rich results; aggregateRating fix; robots.txt + sitemap.
- **Review automation** (post-delivery email + `opinia.php` deep-link mapping SKU→product + moderation).
- **Filter cleanup** (merging duplicate feature values, sorting, mobile) + **prevention cron** (n8n at 3:00).
- **Full shop UI modernization** (top bar, product card, listings) injected safely via `footer.tpl`.

### 3.5 Infrastructure, observability, documentation
- Energy/PV, home and sales dashboards (InfluxDB+Grafana), alerts, schedulers.
- **Per-tool documentation** (`/config/docs/`, markdown mirrored in HA as clickable subviews) + a RUNBOOK
  (rebuild-from-zero steps, ID tables, where secrets live) + Artifacts (dashboards, a "Tools" marketing page).
- **Backup:** HA (daily, encrypted, off-site via Nabu Casa) + a **nightly export of n8n workflows** into
  `/config` (because add-ons are NOT in the HA backup) + a git-mirrored copy of long-term memory.

## 4. How it is built (stack + patterns)
- **Home Assistant:** config as YAML packages (`packages/*.yaml`), custom Lovelace cards in vanilla JS (no deps),
  dashboards edited via websocket (`lovelace/config/save`), `rest_command`/`script` services with
  `response_variable`, entity exposure to Assist in `core.entity_registry`.
- **n8n:** Code nodes (JS, `this.helpers.httpRequest`), webhooks (responseMode lastNode), cron, cache/state in
  `getWorkflowStaticData('global')`; deploy via Public API (deactivate → PUT → activate to reload code without a
  race on `staticData`).
- **PHP (Kylos):** standalone endpoints (no framework), CORS scoped to a specific origin, token auth
  (`hash_equals`), secrets in a file outside the web root; PHP 7.0 compatibility in the `bot/` directory.
- **Chrome MV3 extension:** content-script on `panel.baselinker.com`, MutationObserver (SPA), batched fetch +
  cache, DOM badge injection.
- **AI (Claude):** tool use mapped to HA scripts, deterministic `web_search` forcing inside the tool loop
  (prompt alone is unreliable), prompt caching (static trunk + `cache_control`), output filter (`humanize`).
- **Verification:** headless Chromium with an injected token to screenshot live UI (HA and external web pages);
  `node --check`, `php -l`, and empirical error-path tests on the live instance.

## 5. Key engineering challenges (and how I solved them)
- **Rotating OAuth token race (wFirma/Allegro).** n8n persists the ENTIRE `staticData` at the end of every run;
  with a 5-min cron plus webhooks, a run that loaded the old token overwrote the fresh one → invalid_grant.
  Fixes: (a) a dedicated "token manager" as the single owner of the rotating token (consumers read-only),
  (b) ultimately migrating wFirma to API keys (no rotation). Lesson: when writing a shared rotating secret,
  deactivate + drain runs first, then PUT, then activate.
- **Silent Allegro commission bug.** A bulk header migration (sed) accidentally replaced `Authorization: Bearer`
  in the Allegro Billing call with wFirma keys → 401 swallowed by `catch` → commission = 0 for ALL orders for a
  month (inflated dashboard). Caught during verification (1 of 338 orders had commission). Fix: correct Bearer;
  lesson: Allegro=Bearer, wFirma=keys — don't mix; a silent `catch` can mask a systemic failure.
- **Margin correctness vs VAT.** Pinned the definition: net revenue/cost, commission as-returned; presented three
  methods and the real VAT impact so it was a conscious business decision, not an imposed one.
- **Backing up non-git state.** n8n logic lives only inside the add-on's DB, and add-ons are NOT in the HA backup
  (`include_addons: []`). Fix: nightly export of workflows into `/config` (which is in the encrypted HA backup),
  gitignored (nodes contain keys).
- **Environment constraints.** Container-in-container: Codex's `bwrap` sandbox fails; headless Chromium needs
  `--no-sandbox`; no LAN → everything via the HA API. Conscious workarounds, not blind patching.

## 6. Security, reliability, DR
- **Secrets** never in git or in prompt context: `secrets.yaml` (gitignored), Kylos config outside the web root,
  encrypted integration config-entries, keys in the n8n DB. Endpoints: token + `hash_equals` + CORS.
- **Chatbot:** per-IP/daily limits, anti-jailbreak, no secret ever enters model context, parameterized SQL, a
  hard cost cap on the AI account.
- **Reliability:** idempotent crons, alerts on real signals, health checks, retry-with-cooldown on API throttling
  (wFirma rate limit), fallbacks in the cost/source cascades.
- **DR:** daily encrypted off-site HA backup + n8n workflow export + memory mirror + a RUNBOOK to rebuild from
  zero (steps + ID tables).

## 7. Metrics / scale (examples)
- ~400 orders tracked in InfluxDB; 338 from Allegro; ~4,700-product Arte catalog; ~9,500 wFirma records after
  sync; ~2,200 product URLs in the shop; extension and chatbot in production.

## 8. Interview Q&A

**Q1. Briefly describe the system you built.**
A production e-commerce + smart-home automation platform: HA as the hub, n8n as the workflow engine, integrations
over REST/OAuth/webhooks, an AI layer (Claude) for a voice assistant and a chatbot, data in InfluxDB/Grafana, and
PHP endpoints where external access is needed. It automates sales (margin, price/stock sync, warehouse documents,
returns), customer support, and home control.

**Q2. Why Home Assistant + n8n rather than a single monolith?**
HA provides a ready entity/scheduling/UI/home-integration layer; n8n gives visual, maintainable flows with code
where needed. Separation of concerns: HA orchestrates and presents, n8n executes integration logic. PHP endpoints
are added only where public HTTPS is required (a browser/shop can't reach the internal n8n).

**Q3. How do you securely manage authentication to many APIs?**
Per system I pick the method: wFirma — API keys (no IP/rotation), Allegro — OAuth2 with centralized refresh of a
rotating token, MF — public. Secrets stay out of git and out of model context. Key lesson: with a shared rotating
token you must eliminate the write race (single owner) or you hit invalid_grant.

**Q4. Tell me about a hard bug you debugged.**
Allegro commission = 0 for all orders for a month; symptom was an inflated margin dashboard. Diagnosis: only 1 of
338 orders had commission > 0 (hard evidence from InfluxDB). Root cause: a bulk header change (sed) put wFirma
keys instead of `Bearer` in the Allegro Billing call, and the 401 was swallowed by a `catch`. Fixed the header
and added the rule "Allegro=Bearer, wFirma=keys". Takeaway: silent try/catch masks failures — verify with data.

**Q5. How do you compute order margin?**
Net revenue − net cost − Allegro commission − Smart fee. Cost from a source cascade with fallbacks. Commission
from the Allegro Billing API (actual, not estimated). I flagged VAT consistency (net vs gross) and presented the
business decision rather than imposing one.

**Q6. How do you ensure reliability and monitoring?**
Idempotent crons, health checks, alerts on real signals (not a plateauing percentage), retry-with-cooldown on API
throttling, cascade fallbacks, trend-chart dashboards. An HA alert (binary_sensor) + a reason surfaced in a popup
and a phone push.

**Q7. How do you deploy changes to production?**
Before a change: commit + a "blast-radius check" (grep every consumer of the entity/service), verify fallbacks,
empirically test the error path on the live instance, check_config, then reload/restart. Significant changes are
agreed up front.

**Q8. How do you test and verify?**
`node --check`/`php -l` for syntax, unit tests of logic on real data, headless Chromium to screenshot live UI
(visual verification without asking for screenshots), and — crucially — reproducing the error path, not only the
happy path. Plus an independent review by a second model (Codex/GPT).

**Q9. What does backup / disaster recovery look like?**
A daily encrypted HA backup (local + off-site Nabu Casa) covering all of `/config`. I discovered add-ons
(n8n/InfluxDB) are NOT in the backup, so I added a nightly export of n8n workflows into `/config`. Plus a RUNBOOK
with rebuild steps and ID tables (webhooks, workflows, endpoints) and where secrets live.

**Q10. How do you secure the public endpoint and the chatbot?**
Token + `hash_equals`, CORS scoped to the origin, secrets outside the web root. Chatbot: per-IP/daily limits,
anti-jailbreak, no secret in model context, parameterized SQL, confidentiality (never reveals the supplier or
wholesale prices via a deterministic filter), a hard cost cap.

**Q11. How did you integrate AI in a production-grade way (cost/quality)?**
Model chosen per task (Haiku for voice/latency, Sonnet for chatbot quality), prompt caching (~89% input savings
on a repeated prompt), deterministic tool forcing in code (the prompt alone is a lottery), an output filter that
guarantees rules regardless of the model, and cost caps.

**Q12. How would you scale this / make it multi-tenant?**
A per-client license key in the extension/endpoint instead of one shared key set; per-tenant data isolation;
queuing and rate-limiting; per-client observability; CI/CD and versioning; ultimately productizing the flagship
(the wFirma extension) as a subscription.

**Q13. Biggest limitation/trade-off in the project?**
The environment (container-in-container) limits the sandbox/LAN — I work around it consciously. Margin in InfluxDB
is computed once at order time (commission can post later) — hence the on-demand chip; a retry/recompute is the
next step.

**Q14. What did you learn?**
In integration automation the winners are: idempotency, a single source of truth for state, eliminating races,
verifying with data (not assumptions), and solid documentation/DR — because these systems live for years and must
be reproducible.

**Q15. Why do you want to work as an automation engineer?**
Because I enjoy turning repetitive, error-prone manual work into reliable, monitored processes — and I can back it
with real end-to-end deliverables (from APIs and databases to UI, AI and DR).

## 9. Glossary
- **Orchestration** — coordinating services/flows (here: HA).
- **Webhook / cloudhook** — a URL that triggers a flow; a cloudhook is a public Nabu Casa URL → HA webhook.
- **OAuth2 / refresh token** — an access token refreshed via a rotating refresh token (race risk).
- **Idempotency** — running multiple times yields the same result (safe crons/retries).
- **Line protocol / InfluxDB** — time-series write format (tag=dimension, field=value).
- **MV3 content-script** — an extension script injected into a page.
- **Prompt caching** — caching the static prompt portion → cheaper LLM calls.
- **Blast-radius check** — checking everything a change touches before deploying.
