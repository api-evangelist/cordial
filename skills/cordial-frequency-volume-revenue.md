---
name: frequency-volume-revenue
description: Prove whether send volume and attributed revenue move together or inversely — send volume, frequency (sends-per-contact), and revenue over a window, sliced by engagement tier and compared period-over-period / YoY (e.g. H1 vs H2) — as a Cordial-branded QBR artifact. Use when a CSM asks "are we over-mailing / can we cut frequency without losing revenue," "volume down but revenue up — show me the trend," "send volume vs revenue," "sends per contact this period vs last," "frequency by engagement tier," or wants a mail-frequency-vs-revenue case for a QBR / strategy review.
---

# Frequency, Volume & Revenue

Help a CSM make the "cut frequency, hold or grow revenue" case: show send volume, sends-per-contact, and attributed revenue over a window, sliced by engagement tier and compared period-over-period or YoY — as a clean Cordial-branded artifact for a QBR / strategy review.

> Shared mechanics live in `reference/`: query grammar + traps (silent-zero, type→wrapper, universal base `{"and":[]}`) in `audience-query-mechanics.md`; tool-traversal patterns — **windowable-vs-lifetime sources**, **progressive-exclusion enumeration**, **cross-channel engagement isn't one metric**, and the **suspicious-result law** — in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`; the **object graph** (message → child sends → per-run revenue path; `:rm` vs `:d` row shapes) in `entity-model.md`; the brand spec for the rendered output in `cordial-brand.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds what's specific to the volume↔revenue thesis.

## The linked path

Spans **Account context → engagement-tier denominators → messages & child sends (windowed volume) → per-run performance (revenue) → derived frequency → render**. Load-bearing handoff keys: **tier clause → `estimate_audience` size (the frequency denominator)**; **automated `list_messages.id` → `list_child_sends.templateId`**; **child-send `:rm{YYYYMMDD}{HHMM}` id → `get_message_performance.messageId` (the ONLY per-run revenue path)**; every figure scoped to a **date window**.

1. **Orient (auth gate).** `whoami` first. **If it returns "requires re-authorization" (expired token), STOP and report "cannot answer — re-authorize the Cordial MCP Production connector, then re-run." Do not proceed, do not fabricate or guess any tier key, message id, count, or revenue figure.** On success, confirm the account. Then **resolve the window and comparison basis** (period-over-period, YoY, or H1-vs-H2) and the **channel** (frequency is per-channel; email is the safe default — see cross-channel caveat) — the thesis is a comparison, valid only when both periods are scoped identically. **Incomplete-period trap:** "this year" or "H1 vs H2" is contradictory mid-year — the current year has no complete H2 yet, so comparing it understates the later period. **Default to the last *complete* year** (e.g. on a mid-2026 date, H1 = Jan–Jun 2025, H2 = Jul–Dec 2025) and **state that choice on the artifact**, or ask the user to confirm. Never compare a partial period against a full one.
2. **Find the engagement-tier dimension.** `describe_contact_schema` — there is usually **no single engagement-tier attribute**; expect frequency-adjacent attrs (e.g. an `email_frequency` string, an `crdl_ai_optimal_weekly_messages_email` number, an `crdl_ai_engagement_*` momentum string). Tiers are best **derived from behavioral recency** (open/click within the past N days) rather than a stored field. `search_audience_examples` for the live clause grammar and lift the exact `.data.query` (the `:type` of each attribute drives the wrapper).
3. **Size each tier (the frequency denominator).** Fire the **universal base `{"and":[]}`** first — it's the file-total denominator **and** a liveness control. Then `estimate_audience` per tier using a `bhl` open/click recency clause (lifted in step 2). **Do NOT tier on subscribe-status via `estimate_audience`** — it silent-zeros on the probe account (zeroing any AND it joins) even though the saved audience stores that grammar; derive tiers from `bhl` recency and treat subscribe-state as unavailable-via-estimate. Enumerate by **progressive exclusion to a ~0 residual** (named pattern in `tool-ergonomics.md`); report an explicit **"no engagement / unidentified"** slice. `estimate_audience` **requires** criteria — never call it empty-bodied.
4. **Window the VOLUME.** `list_messages(type=batch)` **and** `list_messages(type=automated)` → `messages[].id`, inline `stats.sent`, `audienceTotal`, and (batch only) `sentAt`. **Sort-key trap:** the list is ordered by `updatedAt`, not `sentAt` — future/edited sends leak onto page 1; filter **every** page on `sentAt`, never early-stop on page position (see Limits). For automated, `list_child_sends(templateId)` → per-run `:rm{YYYYMMDD}{HHMM}` rows are the **only date-windowable** volume source (their `sentAt` / the date in the id is the window key). Sum `stats.sent` per period.
5. **Add per-run REVENUE.** Child-send and inline stats carry **no revenue** — `list_child_sends.stats` is sent/opens/clicks/bounced/failures only. Revenue requires `get_message_performance(messageId, channel)` → `dashboard.stats.revenue.{direct,indirect}` + `orders.direct`: for batch, call it on the in-window batch id; for automated, **one call per in-window child `:rm…` id** (it resolves here and returns `parentTemplateId`). Retry-once on a transient connection drop. **Before looping per-send,** check for a cheaper bulk windowed-revenue source: `list_saved_reports` / `list_insight_reports` may already pre-aggregate revenue by period in one read (`run_saved_report` / `get_insight_report`); `export_message_stats` may carry revenue across many messages at once. Prefer one of these over an N-call loop when it covers the window — fall back to the per-send loop only for the gaps. **Call budget:** a full-year per-tier rollup is potentially thousands of reads (every batch page + every automated child run + one `get_message_performance` per in-window send). Set an explicit budget and say so: cap the per-send revenue calls (e.g. **top-N in-window sends by `audienceTotal`**, the volume drivers) and label the result a **bounded estimate** rather than stalling or silently truncating.
6. **Derive frequency.** There is **no native sends-per-contact metric.** Derive it: `sends-per-contact (tier, period) = period sends to tier / tier audience size`. Label it a **derived estimate.** Compare period-over-period / YoY per tier. **Denominator caveat:** `estimate_audience` recency tiers are **as-of-now** — there's no as-of-date param, so a prior period's sends are divided by the *current* tier size, not that period's historical size. State this; it's a point-in-time denominator, not a period-historical one.
7. **Render.** Hand the reconciled rollup to a Cordial-branded HTML/PPTX QBR artifact using `reference/cordial-brand.md` (Cheery on every asset; Aqua-led charts; Horizon as text-only section titles; green/yellow/red variance chips for the YoY deltas).

## Output

A Cordial-branded QBR artifact (HTML or PPTX): a volume-vs-revenue trend (do they move together or inversely?), a frequency-by-tier table, and the period-over-period / YoY comparison. State the window and comparison basis on the asset. Phrase revenue as **attributed**, frequency as a **derived estimate**, and tier membership as **the model's verdict**.

## Guardrails

- Read-only — counts/sizes are estimates; never imply any send, audience, or report was created, saved, or refreshed.
- **Windowable ≠ lifetime.** Only batch `sentAt` + child-send `:rm{YYYYMMDD}{HHMM}` rows are date-windowable for volume; per-run revenue comes from `get_message_performance` on the in-window id. `get_message_performance` cumulative totals **cannot** be re-windowed — only fold in the per-run revenue for in-window sends; never present a lifetime total as a period figure.
- **Frequency is derived,** not measured (period sends / tier size) — say so.
- **Revenue is attributed / dashboard-reported, not incrementality** — phrase as "attributed revenue," never "incremental lift."
- **Engagement isn't one metric across channels** — email opens are MPP-inflated (use adjusted), SMS opens are always `0`, push opens are delivery-receipt artifacts. Keep channels separate; don't render one shared "open rate."
- **`crdl_ai_*` tiers are model-derived** under the model's own definition/lookback — the model's verdict, not literal fact. Enumerate by progressive exclusion to a 0 residual; report a "no engagement / unidentified" slice.
- Apply the **suspicious-result law** (`tool-ergonomics.md`): if volume↔revenue doesn't reconcile, find the right traversal before reporting.

## Honest tool limits (what the probe found)

- **A prior build was hard-blocked by an expired token; this chain was then re-walked live**, but treat all counts as illustrative — the proposed path is validated for shape, not for any specific period's figures.
- **No native sends-per-contact metric** — frequency must be derived (period send volume / tier audience size); always state it as a derived estimate.
- **Per-run revenue is N calls — try a bulk source first.** `list_child_sends.stats` has **no revenue/orders** (sent/opens/clicks/bounced/failures only) — every automated child run needs its own `get_message_performance` to get revenue, and the batch list alone can run to thousands of in-window sends. Before the loop, check whether `run_saved_report` / `get_insight_report` (via `list_saved_reports` / `list_insight_reports`) or `export_message_stats` already windows revenue across many messages in one read; prefer that and loop only for gaps. If no bulk source covers the window, set a call budget (cap to top-N in-window sends by `audienceTotal`) and label the rollup a **bounded estimate**.
- **Frequency denominator is point-in-time.** `estimate_audience` recency tiers are as-of-now (no as-of-date param), so prior-period sends-per-contact divides by the *current* tier size, not that period's historical size — state this caveat with any PoP/YoY frequency figure.
- **Subscribe-status clauses can silent-zero.** On the probe account the saved-audience subscribe-state grammar returned `0` via `estimate_audience` (both nested and flat forms), zeroing any AND it joined, while the universal base stayed live — derive tiers from `bhl` open/click recency instead. `estimate_audience` also **requires** criteria; calling it empty errors. See the silent-zero trap in `audience-query-mechanics.md`.
- **Aggregation lag — can span weeks, not just days.** Recent child runs show `status='sent'` with `stats=0` / `audienceTotal=0` until settled — observed lagging **~24 days** on a live automated program (older runs populate fully). The lag zeros land on exactly the rows a "this period" query grabs first. **Positive check:** before trusting any current-period automated rollup, spot-check that recent runs populate; **if recent runs are all 0 while older ones populate, declare current-period automated volume/revenue unavailable — never report the 0.** For a closed prior year (a settled QBR window) this is mostly safe, but flag any sends near a half/year boundary; consider excluding the last few settling days at the boundary.
- **Windowable-vs-lifetime trap** — automated lifetime totals from `get_message_performance` cannot be windowed; only batch `sentAt` and child-send `:rm{YYYYMMDD}{HHMM}` rows are. Parse the date from the child id or use `sends[].sentAt`.
- **Revenue is attributed, not incremental** — per-send / dashboard-reported (`revenue.direct/indirect`; `perMessage` is a derived rate), not a true incrementality measure.
- **Cross-channel engagement is not one metric** (MPP email opens → adjusted; SMS opens = 0; push opens = delivery artifacts) — keep channels separate; tier per-channel.
- **`list_messages` sort-key trap** — sorts by `updatedAt` not `sentAt`; filter every page on `sentAt` (batch totals can run to thousands of pages — set a page budget), never early-stop on page position, or report the window-coverage gap.
- Report at the level the tools support; never infer hidden internals (the exact MPP/bot model, the AI model's lookback, or attribution logic — defer to Cordial's own definition).

## Worked example (illustrative — fictional values)

A CSM asks: *"We cut email volume this year but I think revenue held up — pull send volume vs attributed revenue by half (H1 vs H2) so I can show it in the QBR, and break out frequency by engagement tier."*

`whoami` confirms the account; "this year" mid-year is incomplete, so the comparison is set to H1 vs H2 of the **last complete year** (stated on the artifact), channel = email. `describe_contact_schema` shows no single tier field, so tiers are derived from email open recency via lifted `bhl` grammar — e.g. *Highly engaged* = opened in past 90d, *Lapsing* = 90–365d, *No engagement* = none in 365d — sized on `{"and":[]}` by progressive exclusion to a ~0 residual, with a "no engagement / unidentified" slice reported (subscribe-status tiering avoided — it silent-zeros). Batch `sentAt` + automated child-send `:rm{YYYYMMDD}{HHMM}` rows summed per half give windowed sends; `get_message_performance` per in-window send adds attributed revenue. Tier sizes from `estimate_audience` give the frequency denominator (illustrative):

| Engagement tier | Tier size | Sends H1 | Sends H2 | Sends/contact H1 → H2 | Attributed rev H1 → H2 |
|---|---|---|---|---|---|
| Highly engaged | 1,200,000 | 6,500,000 | 5,200,000 | 5.4 → 4.3 | $4.1M → $4.4M |
| Lapsing | 900,000 | 5,300,000 | 3,000,000 | 5.9 → 3.3 | $1.0M → $1.1M |
| No engagement / unidentified | 700,000 | 2,800,000 | 900,000 | 4.0 → 1.3 | $0.1M → $0.1M |

Story for the QBR: total volume fell H1→H2 while attributed revenue held or rose — the cut came mostly from the lapsing / no-engagement tiers, so frequency-down did **not** cost revenue. Rendered to a branded one-pager per `cordial-brand.md` (Aqua volume bars, Cheery revenue line, green YoY chips). All numbers fictional and for shape only; revenue is attributed (not incremental) and frequency is derived (sends ÷ tier size).


---

## Inlined shared mechanics — read and follow these; do not skip
_(These are the shared references this skill depends on, flattened in so they are never skipped. Deeper traversal patterns, if referenced, are bundled under `reference/`.)_

### Tool index (buckets, roles, handoff keys)

Reference map shared by all recipes. Three lenses: **buckets** (what), **role tags** (when), **handoff keys** (how they link). All tools are read-only.

## Role tags

Every tool plays a repeatable role in a chain. Recipes reference these roles so they survive tool changes.

| Role | Purpose | Needs an ID first? |
|---|---|---|
| **Orient** | Establish account context | No |
| **Discover/List** | Find the thing; returns IDs + names | No |
| **Inspect/Get** | Drill into one entity by ID | Yes |
| **Expand** | Fan out from an entity to its children | Yes |
| **Reference/Example** | Show a correct pattern before building | No |
| **Analyze/Terminal** | The payoff call that produces the answer | Usually |

Typical flow: **Orient → Discover → Inspect → Expand → Analyze**, with Reference as a side-trip before constructing any query.

## The 8 buckets

**1. Session & Account Context** — `whoami`(Orient) · `get_account_overview`/`_document` · `get_recent_account_activity` · `describe_contact_schema` · `describe_account_events` · `get_account_products_config` · `list_channel_types`

**2. Audiences & Segmentation** — `list_audiences`(Discover) · `get_audience`(Inspect) · `get_audience_count` · `describe_audience` · `estimate_audience`(Terminal) · `get_audience_health` · `get_audience_trends` · `get_account_audience_samples`(Reference) · `search_audience_examples`(Reference)

**3. Messages & Sends** — `list_messages`(Discover) · `get_message`(Inspect) · `get_message_transport` · `list_message_transports` · `list_child_sends`(Expand) · `get_message_report`

**4. Performance & Reporting** — `get_message_performance`(Terminal) · `explain_message_performance` · `export_message_stats` · `list_insight_reports` · `get_insight_report` · `compare_insight_reports` · `list_saved_reports` · `run_saved_report` · `describe_report_entities` · `describe_report_export_columns`

**5. Orchestrations / Journeys** — `list_orchestrations`(Discover) · `get_orchestration`(Inspect) · `search_orchestration_examples`(Reference)

**6. Data Tables & Supplements** — `get_account_supplements` · `list_supplements` · `get_supplement` · `list_supplement_records`(Expand) · `list_data_batches` · `get_data_batch` · `list_data_automations` · `get_data_automation`

**7. Content & Creative** — `list_sculpt_templates`/`get_sculpt_template` · `list_sculpt_blocks`/`get_sculpt_block` · `list_html_includes`/`get_html_include` · `search_images`/`get_image` · `list_tags`/`get_tag`

**8. Product Catalog** — `list_products` · `get_product`

## Handoff keys (the connective tissue)

| Key | Produced by | Consumed by |
|---|---|---|
| **attribute/field names** | `describe_contact_schema` | audience criteria in `estimate_audience`, `get_audience_count` |
| **audienceID** | `list_audiences` | `get_audience`, `get_audience_health`, `estimate_audience` |
| **orchestrationID** | `list_orchestrations` | `get_orchestration` → `list_child_sends` |
| **messageID** | `list_child_sends`, `list_messages` | `get_message_performance`, `export_message_stats` |
| **date range** | (user intent) | cross-cutting filter on all reporting tools |

## Always start with `whoami`

The user may have switched accounts since last login. Confirm the account before any chain.

### Query grammar & traps

Account-agnostic *how-to* for any recipe that touches `estimate_audience` / audience criteria. This file is the **mechanism**; the real **values** are always learned live from the account (never hardcode them here).

## The self-learning loop

For any attribute, learn the account before querying it:

1. **Find it** — `describe_contact_schema` (contact attributes: `key` + `type`) or `describe_account_events` (behavioral). The **type drives the wrapper**.
2. **Learn real values + operators** — `search_audience_examples` (natural-language) and/or `get_account_audience_samples`. Read the returned live query JSON and lift the actual values, operator, and wrapper. These examples are the source of truth.
3. **Fallback when examples don't cover it** (common for `crdl_ai_*` and other unused attributes) — probe empirically with `estimate_audience` (see completeness check below).
4. **Size** — `estimate_audience` with the learned attribute + operator + value(s).

## Grammar notes (verify against live examples — these can vary by account)

- **Wrapper by type:** string → `icfs`, number → `icfn`, date → `icfd`, **array → `icfa`**, geo → `geoRange`. Don't put a numeric field under `icfs`. Arrays expose a `has_value` operator (`{"icfa":{"<key>":{"operator":"has_value"}}}`) — use it for "has any value" coverage rather than excluding `is_empty`.
- **Conditions** always nest under `and` / `or`. Negation goes under an `exclude` block — **but `exclude` needs a base to subtract from.** A top-level `exclude` with no `and`/`or` base returns `count: 0` (a silent-zero variant), not the complement. Pair it with a base: `{"and": [ … ], "exclude": { "or": [ … ] }}`.
- **Universal base / file total:** an **empty `and`** — `{"and": []}` — returns the entire contact file count. Use it as the denominator for account-wide coverage and as the base for a deterministic exclusion residual (`{"and": [], "exclude": {"or": […]}}`).
- **"Is set"** = `is_empty` *negated* (there is no `exists` / `set` / `not_empty` operator).
- **Operators seen in the wild:** `matches`, `in`, `gt` / `lt`, `after`, `is_empty`, `has_domain`, `begins_with`. Subscription state via `channels.email.ss matches ["s"|"u"|"n"]`; validity via `isInvalid` / `isNotInvalid`.
- **The canonical subscribed-state clause, written out in full** (the single most-asked clause; both key notations shown because examples display the comma form while `estimate_audience` usually wants dots — if one silent-zeros, flip to the other):
  - As `search_audience_examples` displays it (comma form): `{"channels":{"channels,email,ss":{"operator":"matches","value":["s"]}}}` — this form can silently return 0 through `estimate_audience`.
  - Converted (the form that worked live): `{"and":[{"channels":{"channels.email.ss":{"operator":"matches","value":["s"]}}}]}` — keep the outer `channels` wrapper object, dot only the inner key.
  - "Emailable" is stricter than "subscribed": saved audiences routinely AND `isNotInvalid: email` onto the `ss` clause — state which definition you used; offer the other as a follow-up.
  - Searching for ss grammar: query status wording ("subscription status subscribed opted in"), not "subscribed to email" — the latter semantically matches subscribe-DATE band audiences instead.
- **`search_audience_examples` payloads can be huge** (>100KB on one line at limit 8) — use a small `limit` (≤5) and re-query rather than one broad pull.
- **`saved_audience` clause works in `estimate_audience`** ◐: criteria lifted from an orchestration fork/filter can reference `{"saved_audience":{"id":"…"}}` — that clause can be fed back to `estimate_audience` as-is to size the referenced audience (confirmed live once).
- **Sibling-attribute value donation:** when the target attribute has no audience example, lift candidate values/casing from a *same-domain sibling* that does (a loyalty-status example donates tier-value candidates for a tier-level field; `tracking_AI_*_High` audiences donate lowercase `high` for an unexampled `crdl_ai_*` dim) — then verify via the residual check before reporting.
- **Geo fields** use a dedicated `geoRange` wrapper for radius targeting: `{"geoRange":{"geo_location":{"distance":15,"operator":"within","units":"mi","location":{"postal_code":"78748"}}}}`. Common geo fields: `geo_location`, `auto_locate_geo`. (Store-level targeting is often an `or` of `home_store matches "<store#>"` plus a `geoRange` around the store's ZIP.)
- **Channel-scoped subscribe/unsubscribe date bands** (`subscribedDate` / `unsubscribedDate`): these are **top-level clause keys** carrying a `channel` (e.g. `"email"`), **NOT** `icfd` and **NOT** contact attributes — they will not appear in `describe_contact_schema`. They are the date-windowable source for subscribe/unsubscribe *flow* (no 12-month ceiling). **Two grammar shapes exist and which one works varies by account — learn the live one from `search_audience_examples` before banding:**
  - **Two-clause `after`/`before`, `AND`ed** (one observed working form): a month `[start, nextStart)` = `{"and":[{"subscribedDate":{"operator":"after","channel":"email","value":"2025-01-01"}},{"subscribedDate":{"operator":"before","channel":"email","value":"2025-02-01"}}]}`. (Some accounts band the upper bound via an `exclude` on the next-month clause instead — same `[start,nextStart)` result.)
  - **Single `between` with a `{gte,lt}` value object** (other observed working form): `{"subscribedDate":{"operator":"between","channel":"email","value":{"gte":"2025-01-01","lt":"2025-02-01"}}}`.
  - **Silent-zero traps (all returned `0`, no error):** nested-path shapes (`{"channels":{"email":{"subscribedDate":{...}}}}`), `between` with a date *array* instead of a `{gte,lt}` object, and (on the account where `after`/`before` was the working form) `between` was unsupported entirely. If a band returns 0, fire the universal-base control, then try the *other* grammar shape — don't read it as "no subscribes that month."

## Two traps to always respect

- **Silent zero.** Invalid operators *and* wrong values both return `count: 0` with no error. A zero means "re-check the operator/value against a real example," not "no data." When in doubt, fire one known-good control query first to confirm the tool is returning live numbers.
- **Enumerate values by progressive exclusion (deterministic).** With no distinct-values endpoint, don't sum guessed values additively. Instead peel known values off and measure the exact remainder: estimate against a real base while `exclude`-ing `is_empty` plus every value found so far — `{"and": [], "exclude": { "or": [ {…is_empty}, {…=v1}, {…=v2} ] }}` (the empty `and` base is required; a bare `exclude` returns 0). When the residual hits ~0 you've found them all; while it's large, add one more candidate and re-measure. **Cap the effort** — these are free-form strings and can be high-cardinality, so capture the meaningful buckets + a deterministic "other" remainder rather than enumerating exhaustively.
- **Distinguish "no value" from "unnamed value."** `is_empty` catches truly-unset fields (often a small slice). A large residual that survives excluding `is_empty` *and* all known values is a **real, non-empty value you haven't named yet** — not null. For Cordial AI `*_category` / `*_string` fields this is almost always the **`no <activity>` bucket** (`no engagement`, `no purchase`); probe that and the residual closes to 0. (Observed: `crdl_ai_engagement_momentum` → `no engagement`; `crdl_ai_average_order_value_category` → `no purchase`, residual hit exactly 0.) Report "no value" and any named bucket separately.

## Coverage idioms

- **Has any value (strings/numbers).** If the account supports a direct `has_value` operator on the field (`{"and":[{"icfs":{"<key>":{"operator":"has_value"}}}]}` — confirm it's real via `search_audience_examples`, don't assume), that's the **simplest primary**. Otherwise (and always as the cross-check) use the **arithmetic residual:** `populated = file_total − direct is_empty(missing)` — fire `{"and":[]}` for the file total and `{"and":[{…is_empty}]}` for the missing count, then subtract. The residual is the **most robust** fallback and self-reconciles (missing + populated must equal file total); a direct `has_value` result that does *not* equal `file_total − is_empty` is a silent-zero/grammar smell — re-check.
- **Has any value via complement — use only as a cross-check, and verify it reconciles.** The complement forms (`{"and":[],"exclude":{"or":[{…is_empty}]}}`, a `{"not":[{…is_empty}]}` wrapper, or an `is_not_empty` operator) are **unreliable on some accounts**: they have been observed silently returning `0`, or **echoing the same value as the direct `is_empty` count** (so populated == missing, summing to ~2× the file). Treat any populated query whose result *equals the `is_empty` count* as suspicious-by-definition and fall back to the `file_total − is_empty` arithmetic. Do **not** put `is_empty` as a positive `and`-member to mean "has value" (a non-empty base `{"and":[{…is_empty}]}` was observed returning `0`).
- **`is_empty` itself can be non-functional on strings.** On some accounts `is_empty:true`, `is_empty:false`, and `not(is_empty:true)` all return `0` for a string dim. When `is_empty` returns 0 but you have other evidence the dim is populated (e.g. positive `in [v1,v2,…]` matches), do not report 0 missing — derive coverage from enumerated-value `in` matches and treat `file_total − populated` as the residual. `not` is **not** a reliable set-complement here either: `not(field in [v1,v2,v3])` has been observed returning the *same* count as the positive `in` match, not the complement.
- **Missing:** the `is_empty` clause directly — **when it works** (see above). Cross-check it reconciles against `file_total − populated`.
- **Distribution:** estimate each value (or `in` a set).
- **Within an audience:** `AND` the attribute clause onto the base audience's criteria.

## Content references (for usage scans)

Attributes appear in message/block content as **Smarty tokens**: `{$contact.<attribute_key>}` (e.g. `{$contact.digital_giftcard_pin}`). Supplements and other data use similar `{$...}` vars. Reusable content is pulled in via `{include 'content:<key>'}`. To find where an attribute is already used, fetch content (`get_sculpt_block`, `get_html_include` — prefer the `key` arg, `get_sculpt_template`, `get_message`) and search for `{$contact.<key>}`.

**Where attributes/content get used, by source:**
- Sculpt blocks / templates, HTML includes → `{$contact.<key>}` and `{include 'content:<key>'}` in content (statically searchable).
- Data automations ("data jobs") → strong signal from name/key/tags; `get_data_automation` exposes trigger **filter criteria** (may reference attributes) + schedule, **but not the inner transformation script** — usage is reportable at job level only.
- Supplements → content sometimes sources from a supplement (data table) instead of a contact attribute (`$utils->getSupplementRecords` with a supplement key, rendering `{$row.<col>}`); note this so usage attribution is accurate. Coupons/gift codes especially are often supplement- or month-rotating-include-driven rather than a static literal.

**Dynamic include binding (blockData) — a static scan blind spot.** Reusable content can be pulled in *dynamically*: a sculpt block whose html is `{include "content:{$blockData.<...>.select_block|default:'<fallback>'}"}` (a "select menu of all HTML Content" widget). The **selected** include key is stored per-message in `blockData` and is **NOT** present in the rendered message HTML — `get_message` exposes only the `|default` fallback. So grepping message bodies for `content:<key>` systematically **misses** these dynamic bindings. Treat any message built on such a block as a *possible* consumer of the target, and flag the per-message selection as an API-unexposed (confirmed-unidentifiable) slice rather than counting it absent.

## Etiquette

- All tools read-only — never imply an audience was built or saved.
- Counts are estimates; say so.
- Transient `MCP server connection lost` happens; retry once.

### Cordial brand tokens (for rendered deliverables)

Self-contained Cordial brand spec for recipes that produce a **deliverable** (HTML report, slides, doc, dashboard). It ships inside this repo so a branded output needs **no external brand skill** — reference this file, apply the tokens, done. This is Cordial's own brand (vendor brand, not client data); applying it consistently is the goal.

> A recipe that only returns numbers/tables in chat does **not** need this. Pull it in only when the outcome is a rendered, shareable artifact.

## Palette (use the hex values verbatim)

| Token | Hex | Use |
|---|---|---|
| **Cheery** (yellow) | `#FABC1C` | **Must appear on every asset** — accent, highlight, key stat, a rule/CTA. Cordial's signature. |
| **Aqua** | `#0DBDF6` | Primary accent — use **most frequently** (charts, bars, highlights). |
| **Pacific** (blue) | `#148EF3` | Links, secondary highlights. |
| **Horizon** (orange) | `#FF833E` | Eyebrows + section titles — as **TEXT only, never a fill/background**. |
| White | `#FFFFFF` | Base background. |
| Sand | `#FCF4ED` | Warm background / section banding. |
| Haze | `#F5F5F5` | Neutral background / cards. |
| Medium Gray | `#EBE8E5` | Borders, dividers, gridlines. |
| Dark Gray | `#474747` | Subtitles, secondary text. |
| Black | `#151515` | Primary text, headers. |

**Frequency rule:** Aqua > Pacific > Horizon. Lead with Aqua; Pacific second; Horizon reserved for eyebrows/section titles. On dark backgrounds, text is white.

**Variance / KPI indicators:** light **green** = positive (`#E3F4E4` fill / `#1F7A33` text), light **yellow** = neutral/flat (`#FCF4ED` / `#8A6D1B`), light **red** = negative (`#FBE6E6` / `#B3261E`). Use for WoW/MoM deltas, pacing, over/under.

## Typography

- **Poppins** — headlines, titles, callouts, statistics.
  - Title: SemiBold (600), 28–34pt, −1% letter-spacing.
  - Subtitle: Medium (500), 18–24pt.
  - **Eyebrow: Bold (700), 12–14pt, ALL CAPS, +0.5% letter-spacing, Horizon `#FF833E`.**
- **Open Sans** — body copy, paragraphs, table cells. Regular (400) or Bold (700), 12–24pt, 120–150% line-height.
- Big numbers (the "so what") in Poppins SemiBold; label them in an Open Sans caption.

Web fonts (when rendering HTML): `Poppins` + `Open Sans` from Google Fonts; fall back to system sans (`-apple-system, Segoe UI, Roboto, sans-serif`).

## Layout idioms for reports

- **Eyebrow → Title → Subtitle** header stack: Horizon eyebrow (ALL CAPS) above a Poppins title, optional Dark Gray subtitle.
- **Section titles** = Horizon **text** (no colored bar behind them).
- **KPI cards**: Haze or white card, Medium-Gray border, big Poppins stat, Open Sans label, a variance chip (green/yellow/red).
- **Charts**: Aqua as the primary series, Pacific as the secondary, Cheery to spotlight one bar/point; gridlines Medium Gray; never use Horizon as a fill.
- **Tables**: header row Black text on Haze, Medium-Gray row separators, right-align numbers.
- Include **one Cheery element per asset** even on data-dense pages (a top rule, a highlighted KPI, the logo lockup).

## Output formats (recipe states which)

- **HTML report** — self-contained `<style>` block using the hexes above + the Google Fonts link. No external CSS dependency.
- **Slides (pptx)** — title slide with eyebrow/title, KPI grid, one chart per slide; brand colors as the theme.
- **Doc (docx)** — Poppins headings, Open Sans body, Horizon section titles, branded cover.

## Do / Don't

- ✅ Cheery on every asset · Aqua-led charts · Horizon for eyebrows/section titles (text) · white text on dark.
- ❌ Horizon as a background/fill · Cheery omitted · rainbow charts · client logos/names baked into a published recipe (the deliverable is branded **Cordial**; the client's *data* fills it at runtime).

### Entity model (the Cordial object graph)

The map of how Cordial's domain objects **relate** and which tool reads each edge. Audience/count questions rarely need this (estimate_audience is self-contained); orchestration / message / experiment / content questions live or die on it. Mechanics and traps stay in `audience-query-mechanics.md` + `tool-ergonomics.md` — this file is the *schema*, not the how-to.

Confidence tags: ✅ = confirmed in ≥2 accounts · ◐ = observed in one account (verify live before relying).

## The graph at a glance

```
Account
├── Contact file ──(NO per-contact endpoint — reach state only via estimate_audience count 1/0) ✅
│     ├── attributes ··········· describe_contact_schema (key + type + validators; name = only human label)
│     ├── events ················ describe_account_events
│     └── channels.<key>.ss ····· subscription state (system path, NOT in schema; can be materialize-only)
├── Audience
│     ├── list_audiences ········ name → id (NO criteria in the list) ✅
│     ├── get_audience_health ··· id → .criteria (machine clauses; .count = stale cached snapshot) ✅
│     └── estimate_audience ····· criteria → {count} (terminal; silent-zero on bad grammar) ✅
├── Supplement (data table) ····· get_account_supplements/.Key → supplements_<KEY> clause; may be
│     │                           contactLinked:false yet still a source (read by transforms) ✅
│     └── Data automation/batch · the writer; batch.transform.dataMapping[] = the ONLY explicit
│                                 feed-column → attribute join; recurring jobs hide the script ✅
├── Orchestration (journey) ──── list_orchestrations → id, status (enabled/DISABLED — check it), tags
│     └── composition.actions[] (get_orchestration; published vs draft arg)        ✅
│           ├── trigger node ··· settings: event, list/listName, podiumTriggerType,
│           │                    limit {times, operator, value} = re-entry cap ✅;
│           │                    recurring triggers carry a FULL settings.audience criteria
│           │                    (ss + validity + suppression + msgh sent/not_sent) — often
│           │                    where the real entry gating lives ◐
│           ├── experiment node · type:"experiment" (e.g. SPLIT_EVENLY) fanning to MULTIPLE
│           │                    message-node ids — a 2nd experiment surface beyond the
│           │                    message-level experiments[]; each arm is its own message ◐
│           ├── audienceFork ···· fork/path/Skip/merge segment routing ◐: each fork path
│           │                    carries its own audienceCriteria (can reference a
│           │                    {"saved_audience":{"id":…}} clause — reusable in
│           │                    estimate_audience to size the paths live); a Skip block is
│           │                    a no-op message slot; paths re-MERGE downstream — so a
│           │                    LATER node can legitimately send to MORE people than an
│           │                    earlier one. Forks, not step filters, are often the real
│           │                    "why did fewer people get email X" answer.
│           └── block node ····· settings.action.type = "Message" | "Transformation"
│                 ├── action.id = the MESSAGE/TEMPLATE id (the handoff key) ✅
│                 │   (action.publishedId = a content hash, NOT the message id) ✅
│                 ├── channelType/channelKey, classification
│                 ├── delay {type, unit, value, quietHoursConfig}
│                 ├── filter.audienceCriteria — SAME grammar as estimate_audience ◐
│                 │   (per-step filters explain node-to-node audience drop-off)
│                 └── nextIds[]/prevId = the DAG edges; stats {audience, goalReached} ◐
│     ⚠ A logical "series" can SPAN orchestrations ◐: an add-to-list Transformation node
│       feeds a group that TRIGGERS a second journey (welcome signup → drip pattern).
│       Follow the group/list to the consuming journey (list_orchestrations by tag/name)
│       before concluding a journey "has only one email."
├── Message (batch | automated — separate list_messages enumerations) ✅
│     ⚠ Journey-node messages do NOT appear in list_messages at all ✅ (confirmed in 2
│       accounts) — a journey's template is reachable ONLY via get_orchestration →
│       action.id. Searching list_messages for "the welcome email" misses it; route
│       through the orchestration. list_messages also has NO name/query filter —
│       name-hunting means paging the corpus; prefer the orchestration path for
│       lifecycle emails.
│     ├── audienceTotal ········· intended denominator (absent until fired) ✅
│     ├── dashboard.stats ······· LIFETIME totals + revenue + adjusted opens/clicks ✅
│     ├── experiments[] ········· see below ✅
│     ├── transport ············· get_message_transport / transportName on the dashboard
│     ├── content.html ·········· get_message (subject + body; blockData selections NOT exposed) ✅
│     └── child sends ··········· list_child_sends(templateId) → per-run rows
│           ├── :rm{YYYYMMDD}{HHMM} / :dYYMMDD id = the run date (only windowable source) ✅
│           ├── row shapes DIFFER ◐: :rm rows carry sentAt + audienceTotal; :d rows
│           │   (status "interval", daily aggregates) carry neither — window them by
│           │   createdAt. Same-day duplicate rows can appear after a republish
│           │   (different content hash) — dedupe before summing.
│           ├── stats: sent/opens/clicks/bounced/failures — RAW only (adjusted/human
│           │   opens exist ONLY on lifetime get_message_performance), NO revenue ✅
│           └── child :rm id fed BACK into get_message_performance = per-run revenue ✅
├── Content layer
│     ├── Sculpt template → sculpt blocks (get_sculpt_block is id-only) ✅
│     ├── HTML includes (get_html_include takes the KEY) ✅
│     └── per-message blockData ·· dynamic include selection, API-unexposed slice ✅
├── Tags ·· list_tags → assetCounts (counts only, no reverse index; list_messages[].tags always []) ✅
└── Products ·· list_products/get_product; get_account_products_config (prose, not JSON) ✅
```

## Experiments — the structure inside the message (field-confirmed live ✅)

Experiments live **on the message**, not on the orchestration. The chain for "how did the test in this journey do":
`list_orchestrations → get_orchestration → composition.actions[] (action.type==="Message") → action.id → get_message_performance(action.id, channelKey) → experiments[]`.

- `experiments[]` is an **array of test ROUNDS**, not one test. Round names **repeat** (an account ran six rounds all named `SLtest`) — distinguish rounds by `created` + `contentKey`, never by name. ✅
- Round object: `name`, `strategy` (e.g. `"mab"`), `conversion` (the winning-metric key, e.g. `uniqueopens`/`uniqueclicks`), `created`, `contentKey` (links the round to a content version — not directly retrievable), `isCurrent` (the active round), `isDraft`, `publishDate`, `variants[]`. ✅
- Variant: `{name (Control/Test1/SLT_1…), stats{…}}`. Stats keys: `ts` (sends), `to`/`uo` (total/unique opens), `tc`/`uc` (clicks), `tb` (bounced), `tr`/`tri` (revenue direct/indirect), `or`/`ori` (orders), `oo` (opt-outs), `cmp` (complaints), plus the conversion key as a **rate**. `ho`/`uho`/`hc`/`uhc` (human/adjusted) populate **only on recent rounds** — fall back to raw ratios when absent and say so. ✅
- **Rounds can be all-zero** (configured/published but never accrued sends) — skip them in reporting, don't average them in. ◐
- **No significance/p-value field exists.** Winner calls are descriptive, with sample sizes. ✅
- MAB (`strategy:"mab"`) reallocates traffic — arm send counts are intentionally unequal; don't read unequal `ts` as a targeting bug. ✅

## Cross-cutting facts that change verdicts

- **Status lives at every layer and they can disagree** ✅ (confirmed in 2 accounts): an *enabled* orchestration routinely points at node messages whose own dashboard `status` is `disabled` — for "is it live," the journey status + recent child-send rows win, not the message status. A *disabled* journey can't have been sending (flips a contact-trace verdict). Read status at the layer the question is about.
- **Opens accrue to the day of the OPEN, not the send** ◐ — small daily child rows can show opens > sends; normal, not corruption. But a daily row with opens **orders of magnitude** above sends (125 sent / 27k opens observed) is a **bot/scanner storm** — exclude it from windowed rates, corroborate via the lifetime total-vs-unique gap.
- **`search_orchestration_examples` can be broken on Production** ◐ (hard 404 `Unknown artifact type: orchestration`, two runs) — treat the full `list_orchestrations` scan as the primary discovery path; the semantic tool is a bonus when it works.
- **Windowable vs lifetime** ✅: only batch `sentAt` and child-send run rows can be date-windowed; message dashboards and `get_message_performance` are lifetime. (Full pattern in `tool-ergonomics.md`.)
- **The three message-id-like keys**: orchestration node `action.id` (= message/template id, chainable), `publishedId` (content hash, NOT chainable), child-send `:rm…`/`:d…` ids (chainable into `get_message_performance` for per-run revenue). Confusing these is the classic dead-end. ✅
