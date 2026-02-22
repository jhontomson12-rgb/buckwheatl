# AI-Powered Accounting Extension Plan for Buckwheat

## System Architecture Overview

### Current repository architecture (as-is)
- **Client-only Android app** (Kotlin + Jetpack Compose + Hilt + Room + DataStore).
- **Single app module** (`app`) with layered organization:
  - UI feature packages (`editor`, `history`, `analytics`, `wallet`, `home`).
  - ViewModels (`SpendsViewModel`, `EditorViewModel`, `AppViewModel`).
  - Persistence via Room (`Transaction`, `TransactionDao`) and DataStore for budget metadata.
- **No backend service today**: all accounting state is local and exposed through repository/ViewModel flows.

### Target production architecture (to add AI safely)
Adopt a **hybrid architecture**:
1. **Mobile app (existing Buckwheat codebase)**
   - Continues as the source of user-entered data.
   - Adds AI Chat tab + category suggestion surfaces.
2. **AI Gateway Backend (new service)**
   - Owns Gemini API access.
   - Handles prompt assembly, redaction, caching, policy checks, and rate limits.
   - Returns compact structured outputs (JSON) to mobile.
3. **Gemini API (Google GenAI)**
   - Used only from backend server key / service account context.

Rationale: avoids shipping Gemini secrets in APK, enables compliance controls, and supports model/prompt evolution without forced app updates.

---

## AI Integration Strategy

### Capability mapping
1. **Budget Q&A**
   - Intent: direct numerical answers (“How much spent on food this week?”).
   - Flow: backend computes deterministic aggregates from transaction payload + optional LLM explanation.
2. **Trend analysis**
   - Intent: detect acceleration/deceleration by category, weekday patterns, anomalies.
   - Flow: backend pre-computes time-series features; Gemini explains insights.
3. **Daily/weekly/monthly summaries**
   - Intent: readable narrative and KPI cards.
   - Flow: deterministic metric block + Gemini natural-language summary.
4. **Auto-categorization for new expenses**
   - Intent: suggest category from merchant/comment/amount/time.
   - Flow: retrieval over user-specific correction history + Gemini classification constrained to user category list.

### Online/offline operating mode
- **Offline-first fallback**: if backend unreachable, app still saves expense with manual category selection.
- **Graceful AI degradation**:
  - `suggestion_status = unavailable` with reason.
  - Chat tab shows retry and local quick stats.

### “Learns from corrections” simulation design
Use a **feedback memory table** and re-ranking:
- Store each override pair (`suggested_category -> final_category`) with features (merchant token, text embedding hash, amount bucket).
- At suggestion time:
  1. Retrieve top-k similar corrections by merchant/comment and amount range.
  2. Boost corrected category score before sending to model (or after model output as business rule).
  3. Persist accepted/overridden outcomes for continual personalization.

---

## Database Modifications

> Existing app has only `transactions` table with `type`, `value`, `date`, `comment`, `uid`.

Add the following entities (Room + backend mirror schema if cloud sync exists):

### 1) `categories`
- `id` (UUID / Int PK)
- `name` (TEXT, user editable)
- `color` (TEXT)
- `icon` (TEXT)
- `is_archived` (BOOLEAN)
- `created_at`, `updated_at`

### 2) Extend `transactions`
- `category_id` (FK -> categories.id, nullable for legacy rows)
- `merchant` (TEXT, nullable)
- `currency_code` (TEXT)
- `notes` (TEXT) // keep `comment` or migrate to `notes`
- `ai_suggested_category_id` (nullable)
- `ai_suggestion_confidence` (REAL)
- `ai_suggestion_model` (TEXT)

### 3) `ai_chat_sessions`
- `id`
- `created_at`
- `last_active_at`
- `title`

### 4) `ai_chat_messages`
- `id`
- `session_id` FK
- `role` (`user`, `assistant`, `system`)
- `content` (TEXT)
- `structured_payload` (JSON/TEXT)
- `created_at`

### 5) `ai_category_feedback`
- `id`
- `transaction_id`
- `suggested_category_id`
- `final_category_id`
- `merchant_norm`
- `comment_norm`
- `amount_bucket`
- `timestamp`

### 6) Optional `ai_usage_events`
- for observability, token usage and throttling.

Migration notes:
- Bump Room DB version and provide manual migration from v5.
- Backfill `category_id` as `NULL` and create a default “Uncategorized” category where needed.

---

## Backend Implementation Plan

### New service modules
1. **API layer**
   - `/v1/ai/chat`
   - `/v1/ai/summarize`
   - `/v1/ai/categorize/suggest`
   - `/v1/ai/categorize/feedback`
2. **Domain services**
   - `BudgetQueryService` (deterministic aggregates)
   - `SpendingInsightsService`
   - `CategorySuggestionService`
3. **Gemini adapter**
   - prompt templates, safety settings, JSON schema response parsing.
4. **Policy & security**
   - authN/Z, quota checks, request signing, PII scrubber.

### API contract sketches

#### POST `/v1/ai/categorize/suggest`
Request:
```json
{
  "userId": "u_123",
  "transactionDraft": {
    "amount": 12.45,
    "currency": "USD",
    "merchant": "Starbucks",
    "description": "latte",
    "timestamp": "2026-01-10T08:10:00Z"
  },
  "categories": ["Food", "Transport", "Subscriptions"],
  "locale": "en-US"
}
```
Response:
```json
{
  "suggestedCategory": "Food",
  "confidence": 0.91,
  "reason": "Coffee purchase pattern and prior corrections.",
  "alternatives": ["Subscriptions", "Transport"],
  "modelVersion": "gemini-2.5-flash"
}
```

#### POST `/v1/ai/chat`
- Input includes scoped transactions/categories/date range constraints.
- Response includes:
  - `answerText`
  - `facts` (machine-readable totals)
  - `charts` metadata (optional)

### Gemini integration flow
1. Mobile sends authenticated request to backend.
2. Backend loads user transaction slice and categories.
3. Deterministic pre-aggregation computes trusted numbers.
4. Backend builds prompt with:
   - system rules (must use provided facts)
   - user question
   - constrained category vocabulary
5. Call Gemini with low temperature for finance accuracy.
6. Validate structured JSON output against schema.
7. Return response + trace id + token usage.

### Recommended model split
- `gemini-2.5-flash`: categorization and short Q&A (latency sensitive).
- `gemini-2.5-pro` (optional): deep monthly insight reports.

---

## Frontend Changes

### Navigation
- Add **new bottom tab**: `AI Chat`.
- In compact mode, keep current history/editor behavior; AI tab is a separate composable route.

### Editor expense flow updates
- On amount/comment entry completion:
  - Fire debounced `suggestCategory(draftTransaction)` call.
  - Show suggested chip: `Suggested: Food (91%)`.
  - User may accept/change category.
- On override:
  - Save transaction with chosen category.
  - Send feedback event asynchronously.

### New UI components
- `AiChatScreen`
- `AiMessageBubble`
- `AiInsightCard`
- `CategorySuggestionChip`
- `AiSummarySheet` (daily/weekly/monthly presets)

### ViewModel additions
- `AiChatViewModel`
- `CategorySuggestionViewModel`
- Extend existing `EditorViewModel` to hold selected category and suggestion state.

---

## Gemini API Prompt Engineering Strategy

### Prompt rules (system message)
- You are a budgeting assistant.
- Use only provided transactions/facts.
- Never invent totals.
- If data missing, explicitly say insufficient data.
- Category output must be one of `allowed_categories`.
- Return JSON only for machine-parseable endpoints.

### Grounding payload structure
Provide compact normalized facts:
- `transactions`: `[ {date, amount, merchant, category, note} ]`
- `aggregates`: totals by day/week/month and by category.
- `allowed_categories`: user-defined list.
- `user_preferences`: locale, currency, week-start.

### Determinism controls
- Temperature: `0.1–0.3` for numeric answers and categorization.
- Response schema enforcement + strict JSON parser.
- Post-generation validator compares cited totals against deterministic aggregates.

---

## Example Prompts Sent to Gemini

### 1) Categorization
```text
SYSTEM:
Classify expense into one of allowed_categories. Return JSON.

USER:
allowed_categories=["Food","Transport","Bills","Health"]
transaction={"merchant":"Uber","description":"trip to office","amount":14.20,"currency":"USD","time":"2026-02-01T09:00:00Z"}
recent_corrections=[{"merchant":"uber","final_category":"Transport"}]

Return:
{"category":"...","confidence":0-1,"reason":"...","alternatives":[...]}
```

### 2) Weekly spending Q&A
```text
SYSTEM:
Answer using only provided facts. If uncertain, say so.

USER:
question="How much did I spend on food this week?"
facts={"week_range":"2026-02-16..2026-02-22","totals_by_category":{"Food":86.10,"Transport":23.50}}
```

### 3) Monthly summary
```text
SYSTEM:
Generate concise summary with 3 actionable recommendations.
Use provided metrics and mention top 3 categories.

USER:
month="2026-01"
metrics={...}
```

---

## Security & Privacy Considerations

### Secret management
- **Never embed Gemini API key in APK**.
- Store key in server secret manager (GCP Secret Manager / Vault).
- Rotate keys; scope IAM minimally.

### Transport/auth
- Mobile -> backend over TLS 1.2+.
- JWT/OAuth2 or signed session token.
- Device attestation (Play Integrity) optional hardening.

### Data minimization
- Send only required fields for each AI request.
- Optionally hash merchant names for analytics endpoints if exact text not needed.
- Configurable retention for chat logs and feedback memory.

### Compliance and user trust
- Add explicit consent toggle: “Use AI features”.
- Explain what data is transmitted.
- Support delete/export for AI interaction data.

### Rate limiting strategy
- Per-user: e.g., 60 chat req/hour, 300 categorization req/day.
- Per-device burst limits + global circuit breaker.
- Cache idempotent summary responses by `(userId, range, updatedAt)`.
- Fallback messaging on 429 with retry-after handling.

---

## Step-by-Step Development Roadmap

### Phase 0 — Foundation
1. Add category domain in app DB and UI (CRUD categories).
2. Attach category to transaction create/edit flows.
3. Introduce backend skeleton + auth.

### Phase 1 — AI Categorization MVP
1. Implement `/categorize/suggest` endpoint.
2. Add suggestion chip in editor UI.
3. Persist user overrides in `ai_category_feedback`.
4. Add offline fallback/manual path.

### Phase 2 — AI Chat MVP
1. Add AI Chat tab and session/message local cache.
2. Implement `/ai/chat` endpoint with deterministic aggregate precompute.
3. Support top intents: category total, date-range summary, saving opportunities.

### Phase 3 — Summaries & Insights
1. Add `/ai/summarize` daily/weekly/monthly presets.
2. Render insight cards/charts.
3. Add scheduled summary generation (optional local notification).

### Phase 4 — Learning Simulation + Quality
1. Retrieval over correction history for personalized categorization.
2. Add evaluation harness with labeled transactions.
3. Tune prompts, thresholds, and model routing.

### Phase 5 — Production Hardening
1. Token/cost monitoring dashboards.
2. Red-team prompt injection tests.
3. Data retention jobs + privacy controls.
4. Rollout via feature flags and staged percentage.

