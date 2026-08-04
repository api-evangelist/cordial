---
name: send-health-diagnostic
description: TRIAGE a send that looks wrong — stuck, slow, under-delivered, or showing an implausible delivered rate — by reconciling its INTENDED audience size against sent/delivered/failed counts, classifying the symptom (in-progress vs slow-but-moving vs short-fired vs truly stalled vs benign), and either resolving it from account data or producing an escalation-ready packet for Cordial support. Use when a marketer/CSM asks "why did this send under-deliver," "this message is stuck processing," "is this send stuck or just slow," "it only sent to 96 of my audience," "the batch only went to 3k of 7k," "why does SMS show 5% delivered," "my scheduled campaign never went out," or "is the delivery rate real or inflated by failures."
---

# Send Health Diagnostic

When a send looks wrong — stuck "processing," fired short of its audience, or showing an implausible delivered rate — **triage it**: reconcile the **intended audience size** against **queued/sent/delivered/failed**, classify what's actually happening, and either close it (most "stuck" reports are benign or explainable from account data) or hand the marketer a precise escalation packet. **Scope honesty up front:** this toolset sees the account-level symptom; the platform's job/queue execution layer (why a worker didn't process a scheduled job, why contacts sit queued) is not exposed by the API — a *true* stall is diagnosed and fixed by Cordial support/engineering, and this recipe's job is to prove it's a true stall and package the evidence.

> Shared mechanics live in `reference/`: query grammar + silent-zero traps in `audience-query-mechanics.md`; tool-traversal patterns (incl. the **suspicious-result law** and **cross-channel engagement isn't one metric**) in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the intended-vs-processed reconciliation that's specific to this outcome. This deliverable is a chat reconciliation (numbers/tables), so no brand spec is needed.

## The linked path

This recipe spans **Account context → Messages/Sends → Performance**, and turns on **one number the native stats ignore: the intended-audience denominator.** Connective keys: **message id → its `audienceTotal` (the intended denominator) + its `dashboard.stats`**; for automated, **template id → child-send rows**.

1. **Orient.** `whoami` to confirm the account.
2. **Find the send.** `list_messages(type=batch)` for a one-time blast, `list_messages(type=automated)` for a journey template; `list_orchestrations` if the marketer names a journey. Handoff key = message **id** (batch) or **template id** (automated). Note the **status** here — `tz-processing` (recipient-timezone) or sent-far-below-audience = in-progress / short-fired, **not final**.
3. **Read the intended denominator off the message itself.** `get_message(id)` → **`audienceTotal`** is the INTENDED audience size — the denominator the native delivered % ignores. (It also returns a human-readable set-algebra `audience` string, e.g. *"Saved audience X OR Y Except Z"* — that is **not** an audience id, so you usually cannot feed it to `get_audience_count`.) Same call returns `dashboard.stats` with `delivered {total, rate}`, `bounced`, `opens/clicks` (+ `opensAdjusted`/`clicksAdjusted` twins).
4. **Reconcile intended vs processed — the crux.** Native `delivered.rate = delivered / SENT` (processed), **never** `delivered / audienceTotal`. So a send stuck at, say, 244K of 30.7M can read **rate: 1.0 (100%) delivered** while it actually reached <1% of the intended audience. **Recompute** `sent / audienceTotal` and `delivered / audienceTotal` by hand, and report the gap (intended − sent = not-yet-reached). The machine-detectable inflation signature: **status `tz-processing` (or `sent << audienceTotal`) paired with `delivered.rate ≈ 1.0`** → flag short-fired + inflated, do not present the native rate. Reconciliation identity to check: **sent − delivered = bounced + failed + still-pending**; pull `bounced` from `dashboard.stats`, `failures` from the flat/child-send stats (it's absent from the dashboard `delivered` object — don't read failures=0 off the dashboard). Note: there is **no "queued" field** — if the marketer asks for queued, derive it as `audienceTotal − sent` (= not-yet-reached) for an in-progress send.
5. **Isolate a single stalled run (automated only).** `list_child_sends(templateId)` → per-run rows (id suffix `:dYYMMDD` encodes the run date) each with `status` + per-run `{sent, opens, clicks, bounced, failures}`. Use these to spot one throttled/stalled run; do **not** try to window an automated template via `list_messages` (it carries lifetime-cumulative stats, no single `sentAt`).
6. **Classify — the triage verdict (the crux of the deliverable).** Every send lands in exactly one bucket:
   - **In-progress, healthy** — `tz-processing` / recipient-TZ drip, or sent climbing between reads. Not stuck; state the expected behavior.
   - **Slow-but-moving** — sent advances between two reads a few minutes apart. Heavy personalization (many data-table reads per render) legitimately slows throughput; slow ≠ stuck. Re-read before declaring a stall.
   - **Short-fired / under-delivered, final** — sent < intended with a final status; reconcile the gap to bounces/failures/suppression.
   - **Benign / not an incident** — audience genuinely smaller than the marketer expected, scheduled-but-not-yet-due, settling-lag zeros on recent automated runs, or the inflated-rate illusion (native 100% on a partial send).
   - **TRULY STALLED** — sent flat across two spaced reads while status stays processing, or a scheduled send past its send time with `sent: 0` and no status movement. **This is platform-side. Stop diagnosing — escalate.**
7. **Terminal — report or escalate.** For the first four buckets: report the reconciliation table + verdict. For a true stall: produce the **escalation packet** for Cordial support — message id + name + channel, current status, scheduled vs actual send time, `audienceTotal` vs sent vs delivered (with timestamps of your two reads proving no movement), the recomputed rate, and the benign causes you ruled out. Do **not** speculate about queue/worker/job internals — name it "platform-side, not visible to account-level tools" and hand off.

## Output

A short reconciliation in chat (no rendered artifact):

| | Count | % of intended |
|---|---|---|
| Intended audience (`audienceTotal`) | … | 100% |
| Sent / processed | … | sent ÷ intended |
| Delivered | … | **delivered ÷ intended** (not the native rate) |
| Bounced (from `dashboard.stats`) | … | … |
| Failed (from flat/child-send stats) | … | … |
| Not yet reached / queued (gap) | intended − sent | … |

State the **native** delivered rate alongside the recomputed one and explain the difference. Flag status (final vs in-progress / short-fired). Use **channel-appropriate** delivery columns (see Guardrails).

## Guardrails

- **Native delivered % is computed on processed/sent, not intended.** Always recompute against `audienceTotal`. A short-fired or still-processing send reads near-100% delivered natively — that is the headline trap, not a footnote.
- **In-progress ≠ final.** `tz-processing` (recipient-timezone, drip-by-local-time) or sent far below `audienceTotal` = partial, in-flight counts. Flag as in-progress / short-fired; never present partial as final.
- **`audienceTotal` is a current estimate** and may differ from the audience size at actual send time. State this; counts are estimates, not ledger truth.
- **Channel-appropriate columns** (see *cross-channel engagement isn't one metric* in `tool-ergonomics.md`): diagnose delivery on sent/delivered/bounced/failed. SMS opens are always `0`; push `opens` are delivery receipts (a tiny opens count on a huge push is a receipt artifact, not engagement); email opens are MPP-inflated — use `opensAdjusted`. Don't read engagement fields as delivery signal.
- Apply the **suspicious-result law**: a 0-row report or a "perfect" 100% on a stalled send means *find the right traversal*, not *report the clean number*.
- **If the marketer's stated symptom matches no surfaced send, say so — don't substitute the nearest send.** When the named channel has no short-fired send, the stated size matches no under-delivered batch, or the stated rate (e.g. "5% delivered") matches nothing live: scan the other channels, surface the actually-short candidate(s), and state the correction plainly (e.g. "no stuck *email* — the stalled send is a *push*"). Ask for the exact message name/id rather than diagnosing a send the marketer didn't mean. Inventing a matching send to fit the premise is the failure mode.
- **Read-only.** Never imply a resend, retry, unstick, or throttle change — this recipe explains the stall, it does not fix it.
- **The job layer is invisible — don't fake it.** Why a scheduled job didn't execute, why contacts sit queued, or what a worker is doing is NOT exposed by this API. A true stall's terminal is the escalation packet, never a guessed infra cause. Two spaced reads (sent flat vs climbing) are the cheapest stall-vs-slow discriminator — use them before escalating.

## Honest tool limits (what the probe found)

- **The intended denominator lives on the message as `audienceTotal`** (`get_message` / `list_messages`), **not** via an `audienceID` lookup. **Read it directly from `list_messages` flat stats for SENT batches** (verified live — e.g. `audienceTotal: 199259` alongside `stats.sent: 197386`) — no need for the large `get_message` body just to get the denominator. **`audienceTotal` is ABSENT on scheduled / not-yet-fired sends** (only the `audience` set-algebra string is present; `sent: 0`) — the intended total locks at send time, so the intended-vs-sent reconciliation is only computable once a send has fired. `get_message`'s `audience` is a human-readable set-algebra string, so the "follow audienceID → `get_audience_count`" path generally does **not** apply. `get_audience_count(id)` is a **fallback only** when the send targets exactly one saved-audience id (most real batch sends target a multi-audience expression with no single id).
- **`get_message_performance` and `explain_message_performance` return the same dashboard payload as `get_message` on the probed account** — they add **no** extra queued/failed raw breakdown. Rely on `get_message` + manual reconciliation; don't expect a deeper stat breakdown from "explain."
- **`failures` is exposed in `list_messages` flat stats and child-send rows, but is absent as a keyed field in the `dashboard.stats.delivered` object.** Pull failure counts from the flat/child-send stats, not the dashboard delivered breakdown.
- **`get_message_report` needs an explicit `startDate`/`endDate` spanning the send time** or it silently returns 0 rows (suspicious-zero, not "no data"). On the probed account, push sends did **not** appear in `get_message_report` under a `push`/brand-key channel query at all — push delivery diagnosis is reliable only via `get_message` / `get_message_performance` dashboard, not the report endpoint.
- Report at the level the tools support; never infer hidden delivery internals (exact throttle/timezone-drip mechanics, the MPP/bot model, attribution) — defer to Cordial's definitions.

## Worked example (illustrative — fictional values)

A marketer: *"This push looks stuck — it only went to a fraction of my audience. How many was it supposed to reach vs how many actually got it, and where did it stall?"*

`whoami` confirms the account. `list_messages(type=batch)` surfaces the send with **status `tz-processing`**, `audienceTotal` **30,700,000**, `sent` **244,000**. `get_message(id)` → `dashboard.stats.delivered {total: 244,000, rate: 1.0}` — native **100% delivered**. Reconcile against the intended denominator:

| | Count | % of intended |
|---|---|---|
| Intended (`audienceTotal`) | 30,700,000 | 100% |
| Sent / processed | 244,000 | 0.8% |
| Delivered | 244,000 | **0.8%** (not 100%) |
| Not yet reached | 30,456,000 | 99.2% |

Verdict: the send is **in-progress (recipient-timezone drip) and short-fired** — it has reached <1% of the intended audience; the native 100% is computed on processed-only and is inflated. ~30.46M not yet reached. (The push's small opens figure is a delivery-receipt artifact, not engagement.) All numbers fictional, for shape only.


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
