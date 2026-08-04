---
name: experiment-report
description: Report the A/B / subject-line experiment results for a Cordial message (or rolled up across its sends) — the per-variant breakdown, the real winner on the metric the marketer cares about (open / click / conversion / revenue-per-email, bot-adjusted and channel-correct), stated with the right confidence and caveats. Use when a marketer asks "who won the A/B test on [message]?", "show me the subject-line test results", "which variant performed best", "did version A or B win", "what's the winning subject line and the adjusted open rate", or "aggregate the split-test results across all sends of this automation."
---

# Experiment Report

Tell a marketer which variant of their A/B / subject-line test won — on the metric they care about — with each arm's real numbers and the honest level of confidence.

> Shared mechanics live in `reference/`: query grammar + traps in `audience-query-mechanics.md`; tool-traversal patterns (incl. **windowable-vs-lifetime sources**, **cross-channel engagement isn't one metric**, the **suspicious-result law**) in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds what's specific to experiments.

## The linked path

This recipe spans **Messages → Performance/Experiments → Adjusted engagement → (optional) per-send rollup → Variant content**. The connective keys are **messageID (+ channel) → experiments[] → variant labels → real subject/creative**.

1. **Orient.** `whoami` to confirm the account — this is also the auth gate. **If `whoami` errors with "requires re-authorization" (token expired), STOP:** there is no programmatic re-auth on Production (only Staging/SAM expose an `authenticate` tool); ask the user to complete the interactive OAuth login, then resume. Do **not** fall back to any cached/probe id — ids drift and reusing them is fabrication. Then **make the marketer pick the winning metric** (open / click / conversion / revenue-per-email) — they can disagree, so never silently default. **Agent/one-pass mode (no user to ask): default to the test's own configured `conversion` metric** (the `conversion` field on the experiment round, e.g. `uniqueopens`) and state that choice in the output. Clarify the date window if they want one (only possible for automated; see step 5).
2. **Find the experiment-bearing message — TWO discovery paths, and the journey path is usually the right one for lifecycle emails.** (a) `list_messages(type=batch)` **and** `list_messages(type=automated)` → `messages[].id`, `channelType` (the `channel` arg), `name`, `tags`; names/tags hinting a split-test (e.g. an `ab_test` / `sl` prefix) point to the experiment message. (b) **Journey-node messages (welcome, cart, post-purchase…) do NOT appear in `list_messages` at all** (confirmed live) — for any lifecycle email, route `list_orchestrations(query/tags)` → `get_orchestration` → `composition.actions[]` message nodes → `settings.action.id` (see `entity-model.md`); also watch for orchestration-level `type:"experiment"` split nodes whose arms are *separate message ids* — each arm needs its own performance read. **Confirm the message the marketer named actually exists in this account before reporting on it** — if neither path matches, report **not-found**; never map their ask onto an unrelated message or assume a template exists.
   **Negative-result branch (message found, `experiments[]` empty):** widen once — check sibling/legacy versions of the same lifecycle email (older journeys, standalone `*_welcome`-style templates; triage order: newest enabled journey's first email → legacy day-1 nodes → literal-name templates, capped at ~15 performance reads) — then report **no test found**, with the honesty caveat: `experiments[]` is the record on the *current* message object only; a test run on a since-deleted/replaced template (journeys carry many archived composition versions the API doesn't expose) is invisible — "no test on any current message" is provable, "never ran one" is not; point to the platform UI's experiment history for the definitive never.
3. **Pull the per-variant arms.** `get_message_performance(messageId, channel)` → **lifetime** dashboard totals + revenue + `experiments[]` (per-variant arm stats: variant label, sends, opens, clicks, conversions, revenue). Handoff key = the variant **labels** inside `experiments[]`. **`experiments[]` is an array of ROUNDS whose names repeat** (six rounds all named `SLtest` observed) — identify rounds by `created`/`contentKey`/`isCurrent`, and **classify all-zero / near-zero-send rounds as aborted/republished configs** — list them as such, never report them as tests with a "0% open rate."
4. **Make the winner channel-correct and bot-adjusted — this is the crux.** Raw opens are MPP-inflated; call the winner on `explain_message_performance(messageId)` per-variant **human** opens (`ho`/`uho`) and `opensAdjusted` / `clicksAdjusted`, not raw. **Engagement is not one metric across channels** (`tool-ergonomics.md`): email = adjusted open/click; SMS opens are always `0` (judge on clicks/failures); push opens are delivery-receipt artifacts. Pick the metric-appropriate, channel-appropriate column before declaring an arm the winner.
5. **(Automated only) roll up across sends + date-window.** `list_child_sends(templateId)` → per-send rows with inline stats and `createdAt` — the **only** date-windowable source. Aggregate variant results across deployments / to a window here; never present the step-3 lifetime totals as a window.
6. **Translate label → content.** `get_message(messageId)` (and the sculpt/include content) maps "Version A/B" / `sl2`-style labels into the **real** subject line / creative the marketer recognizes. **This can fail honestly:** `get_message.content` can return `{}` on automated templates (confirmed live), and the round's `contentKey` hash is not fetchable by any exposed tool — when so, report variant **labels** with the numbers and point the marketer to the platform UI for the literal strings; never guess a subject line. Report any holdout/control arm as its own arm — never folded into a variant.

## Output

A per-variant table (one row per arm: sends + the chosen metric's value, with the real subject/creative) and a one-line winner call on the marketer's chosen metric. State the metric, the channel-correct column used, and whether the figures are **lifetime** (steps 3–4) or **windowed** (step 5 child sends). If significance isn't exposed by the API, state the winner **descriptively** with sample sizes — do not invent a p-value. Numbers only / in chat; no rendered artifact, so no branding needed.

## Guardrails

- Read-only. Never imply a winner was promoted, a test was stopped, or a send changed. `export_message_stats` is outside the read-only rail — use `get_message_performance`.
- **Winner is metric-dependent** — open / click / conversion / revenue-per-email can disagree. Report the winner on the metric the marketer chose, and flag if a different metric would name a different arm.
- **Use adjusted, channel-correct engagement** before declaring a winner (step 4). Never compare a raw open rate across channels or call a winner on MPP-inflated opens.
- **Lifetime ≠ window.** `get_message_performance` / `explain_message_performance` are lifetime and drift between calls; only `list_child_sends` rows are windowable. Label which one each figure is.
- Apply the **suspicious-result law** (`tool-ergonomics.md`): if an arm reads 0 / too clean / doesn't reconcile, find the right traversal before reporting.
- Report at the level the tools support; never infer hidden internals (the exact MPP/bot model, attribution logic, or whether the platform auto-picked a champion — defer to Cordial's definition).

## Honest tool limits (what the probe found)

- **`experiments[]` arm shape — FIELD-CONFIRMED LIVE** (live-account probe, 2026-06, MAB subject-line test). The experiment object carries: `name`, `strategy` (e.g. `"mab"`), `conversion` (the winning metric key, e.g. `"uniqueclicks"`), `created`, `contentKey`, `isCurrent`, `isDraft`, `publishDate`, and `variants[]`. Each variant is `{name (e.g. SLT_1/SLT_2), stats{…}}` where `stats` keys are: `ts` (sends), `to`/`uo` (total/unique opens), `tc`/`uc` (total/unique clicks), `tb` (bounced), `tr`/`tri` (revenue direct/indirect), `or`/`ori` (orders direct/indirect), `ho`/`uho` (human opens), `hc`/`uhc` (human clicks), `oo`, `cmp`, plus the conversion metric (e.g. `uniqueclicks`) as a rate. **`ho`/`uho`/`hc`/`uhc` (human/adjusted engagement) populate only on recent experiment rounds — older rounds omit them**; fall back to `to`/`uo` ratios when absent. Still call the winner on the marketer's chosen metric.
- **Statistical significance is NOT surfaced — confirmed.** No p-value / confidence field exists in the arm object (verified live). Always state the winner descriptively with sample sizes and flag **"significance not exposed"** — never fabricate one.
- **MAB arms are NOT a 50/50 split — read the skew as signal, caveat the rates.** `strategy:"mab"` reallocates traffic toward the leader, so arms are non-contemporaneous, unequal allocations: (a) the extreme send skew toward one arm IS the bandit's implicit winner call — say so; (b) rate comparisons between a 200K-send arm and a 6K-send arm carry allocation/timing bias — note it rather than presenting the rates as a controlled split.
- **`get_message`, `get_message_performance`, and `explain_message_performance` return the SAME dashboard payload** (verified live — identical `stats` + `experiments[]`). `explain_message_performance` adds **no** extra per-variant breakdown over `get_message_performance`; use whichever the arg shape favors. Tool grammar: `get_message_performance` needs `messageId` **and** `channel`; `explain_message_performance` and `get_message` take `messageId` **only**. The `channel` arg is a `list_channel_types` **key**, not the logical channel name (SMS can span `sms` + `sms-txn`; push lives under a brand-named key, not literally `push`) — enumerate keys live, never assume.
- **Lifetime totals drift between calls** (the same message read different raw-open counts across calls) — don't treat a single read as exact; the variant *ratio* is the signal, not the absolute.
- **Revenue/attribution may be a real `0`** on some accounts (verified `0`, not an artifact, on at least one prior account). If revenue/orders come back `0`, a revenue-per-email winner degrades to an **order-rate proxy** — probe attribution early (one `get_message_performance` on the flagship arm) before promising a revenue winner.
- **Aggregating across all sends of an automated template is heavy pagination** (one template showed ~1,500 sends across ~150 pages). Set a page budget and report a **coverage gap** rather than a silent under-count; automated messages have no `sentAt`, so they cannot be windowed from `list_messages` — only from `list_child_sends.createdAt`.
- **Re-fetch ids live.** Pull message/template ids from a fresh `list_messages`; ids drift and a cached/probe id won't match.
- **`list_messages` has no name/query filter** — name-hunting a test message means paging the corpus; for lifecycle emails skip straight to the orchestration path. **`search_orchestration_examples` can be broken on Production** (404 `Unknown artifact type` observed) — fall back to the full `list_orchestrations` scan without burning retries.
- **`clicksAdjusted` can be uniformly 0 on an account** — probe it before promising an adjusted-click winner; fall back to raw unique clicks with the caveat.

## Worked example (illustrative — fictional values)

A marketer asks: *"Who won the subject-line A/B test on our latest welcome email, and what was the adjusted open rate for the winning version?"*

`whoami` confirms the account; marketer's metric = **adjusted open rate** (email). `list_messages(type=automated)` surfaces a message whose tag prefix hints a subject-line test; `get_message_performance(messageId, channel=email)` returns `experiments[]` with two arms, and `explain_message_performance(messageId)` supplies per-variant human/adjusted opens. `get_message(messageId)` maps the labels to the real subject lines. Result (all values fictional, for shape only):

| Variant | Subject line | Sends | Adj. open rate |
|---|---|---|---|
| A | "Welcome — here's 10% off" | 12,400 | 18.2% |
| B (winner) | "Your discount is waiting inside" | 12,350 | 21.6% |

**Winner: Variant B** on adjusted open rate (21.6% vs 18.2%), figures **lifetime**. Significance is **not exposed by the API** — called descriptively on the sample sizes above; a click- or revenue-based read could name a different arm. All numbers fictional.


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
