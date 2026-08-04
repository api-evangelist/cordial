---
name: orchestration-performance-report
description: Build an accurate performance report for a Cordial orchestration (automated journey) — per-node sends/engagement over a real date window, node-to-node drop-off explained by each step's filters/delays, per-run revenue, and any A/B experiment running inside the journey. Use when a marketer asks "how is this journey/automation performing," "results by node/step/send for [journey]," "why does step 2 send to fewer people than step 1," "which message in the welcome series wins," "what's the test running inside this journey," or "journey stats for the last N days." Chains orchestration → message nodes → child sends → performance so the numbers tie out.
---

# Orchestration Performance Report

Help a marketer get an accurate performance picture for an automated journey — broken down by the actual sends inside it, over a defined time window, with node-to-node drop-off explained and any in-journey experiment reported correctly.

> Shared mechanics live in the project-root `_reference/` directory (resolves via `reference/`): the **object graph** — orchestration → nodes → message → experiments/child sends — in `entity-model.md`; tool-traversal patterns (esp. **windowable-vs-lifetime sources**, **cross-channel engagement isn't one metric**, the **suspicious-result law**) in `tool-ergonomics.md`; query grammar + traps in `audience-query-mechanics.md`; the bucket/role/handoff-key map in `tool-index.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the journey-rollup chain.

## The linked path

Spans **Orchestrations → Messages & Sends → Performance**. Connective keys: **orchestration `id` → `composition.actions[]` message nodes → `settings.action.id` (= the message/template id; `publishedId` is a content hash, NOT the message id) → `list_child_sends(templateId)` rows (windowed volume) → child `:rm…` id → `get_message_performance` (per-run revenue) → `experiments[]`** — all scoped by a **date range**.

1. **Orient.** `whoami` to confirm the account. Clarify the time frame with the marketer if not given — reporting is only accurate when scoped. If they want "all-time," say the figures will be lifetime and skip windowing.
2. **Discover the journey — and check whether the "series" spans MORE than one.** `list_orchestrations` — find it by **name or tags** (there is rarely a journey literally named after the ask; e.g. cart-abandon journeys surface under `abandon_cart`-style tags). `search_orchestration_examples` if the marketer describes it by intent rather than name (it can be broken on Production — 404 `Unknown artifact type` observed; the full `list_orchestrations` scan is the dependable primary). **Forks ≠ filters:** if the journey contains `audienceFork`/Skip/merge nodes, the fork paths' `audienceCriteria` — not the step filters — are usually the real volume gate; paths re-merge, so a later node can legitimately send to MORE people than an earlier one (see `entity-model.md`; size the paths live by reusing the fork's criteria, incl. `saved_audience` clauses, in `estimate_audience`). **Read `status`:** a `disabled` journey has not been actively sending — report that up front; its numbers are historical, not current. **If the matched journey contains only one or two message nodes but the marketer described a multi-email series, the series likely spans multiple orchestrations** (confirmed live): an add-to-list Transformation node feeds a group that *triggers* a second journey (signup → drip pattern). Follow the group/list to the consuming journey (`list_orchestrations` by tag/name; the downstream trigger's `settings.audience` carries the full entry criteria — subscribed/valid/not-suppressed/msgh clauses — which is often where the real gating lives). Roll all member journeys into one report. Handoff key: orchestration **id(s)**.
3. **Map the structure.** `get_orchestration(id)` returns `composition.actions[]` (and a Mermaid diagram; `composition` arg selects published vs draft — state which you read). Message nodes are `type:"block"` with `settings.action.type === "Message"` — their `settings.action.id` is the **message/template id** and `channelKey`/`channelType` the channel. Note each node's **`delay`** and **`filter.audienceCriteria`** — per-step filters use the same criteria grammar as `estimate_audience` and are the usual explanation for node-to-node audience drop-off (the "why does step 2 send less" answer is in the filter, not a bug). The trigger node's `limit {times, operator}` is the re-entry cap. Ignore Transformation/Delay-only nodes for stats; keep them for the drop-off story.
4. **Choose the right source for the question — this is the crux:**
   - **Date-scoped report (e.g. "last 30 days"):** `list_child_sends(templateId)` per message node → per-run rows (`:rm{YYYYMMDD}{HHMM}` / `:dYYMMDD` ids encode the run date) with inline stats — `sent / opens / clicks / bounced / failures` — and `createdAt`/`sentAt`. **Filter rows to the window and aggregate.** This is the **only** date-windowable source. Paginated (can be hundreds of pages) — page only as far as the window needs, and report a coverage gap if a page budget truncates it.
   - **Lifetime / ROI context:** `get_message_performance(messageId=node action.id, channel=channelKey)` returns **all-time** dashboard totals plus revenue, orders, opt-outs, and `experiments[]`. It is **NOT date-scoped** — never present its lifetime totals as the requested window.
   - **Windowed revenue:** child-send rows carry **no revenue**. Feed an in-window child `:rm…` id back into `get_message_performance` — the one per-run revenue path (one call per run; budget it, cap to top runs if needed and label a bounded estimate). **Probe lifetime revenue first:** if `dashboard.stats.revenue`/`orders` are 0 on every node lifetime, skip the per-run loop entirely and flag "attribution may not be wired to this journey — confirm config before reading $0 as zero ROI." (Whether `:d` daily-aggregate ids resolve in `get_message_performance` for per-run revenue is unverified — test one before looping.)
5. **Report any in-journey experiment (if asked, or if material).** The same `get_message_performance` call returns `experiments[]` on the node's message. Per `entity-model.md`: it is an array of **rounds** whose names repeat — identify the round by `created`/`contentKey`/`isCurrent`, skip all-zero rounds, call the winner on the marketer's metric using `ho`/`uho` adjusted fields when present (recent rounds only), and state that **significance is not exposed**. MAB rounds have intentionally unequal sends. For a full experiment deep-dive, hand off to the **experiment-report** recipe.
6. **Assemble — and reconcile any claim the marketer made before explaining it.** Roll up per-node and total for the metrics asked, with the date window stated explicitly. **If the ask asserts a delta ("step 2 sends way less"), reproduce it against the windowed volumes FIRST** — sum every feeder journey's volume into each step; the claimed gap may not exist on the named journey alone, or may run the other direction (suspicious-result law applied to the premise). Then explain the real delta via the step filters / delays / downstream trigger audience / re-entry cap read in steps 2–3. Use channel-appropriate engagement — **note that windowed child-send rows expose RAW opens only** (MPP-inflated; adjusted/human opens exist only on lifetime `get_message_performance`): report windowed engagement as raw with that caveat, or pair it with the lifetime adjusted rate, labeled. SMS opens are always 0 — judge on clicks/failures; push opens are delivery-receipt artifacts. Label every lifetime figure as lifetime.

## Output

A per-node table (node label → message name, channel, windowed sends/engagement, per-run revenue if requested) + journey totals, the window stated, drop-off between nodes explained from the composition (filters/delays/cap), journey `status` flagged, and any active experiment summarized with its round identified. Lifetime figures labeled lifetime. Chat-first; only render a branded artifact (per `reference/cordial-brand.md`) if the marketer asks for a shareable deliverable.

## Guardrails

- **Read-only.** Never imply the orchestration, a message, or a test was modified, enabled, paused, or promoted.
- **Windowable ≠ lifetime.** Child-send rows are the only windowable source; dashboard/`get_message_performance` totals are lifetime. Never mix them in one column without labels.
- **`publishedId` is not the message id.** Chain on `settings.action.id`; a `publishedId` fed to message tools dead-ends.
- **Journey `status` changes the verdict.** A disabled journey isn't sending — say so before reporting its stats as if live. Node-level message status can disagree with journey status (observed live) — read status at the layer the question is about.
- **Engagement isn't one metric across channels** (`tool-ergonomics.md`) — adjusted email opens, click-based SMS, sends-only push.
- **Don't dump every send.** If child sends are numerous, summarize by node; state pages scanned and any coverage gap.
- Apply the **suspicious-result law**: an all-zero node on an enabled journey, or totals that don't tie out across nodes, means find the right traversal (settling lag, wrong id, draft-vs-published composition) before reporting.

## Honest tool limits (what the probe found)

- **`get_orchestration` exposes config, not per-node windowed stats** — node-level volume comes only from each node message's child sends; `composition.stats {audience, goalReached}` is a thin journey-level counter, not a reporting surface. ◐
- **Per-step filter criteria are readable** (`filter.audienceCriteria`, same grammar as `estimate_audience`) — you can *explain* drop-off and even re-estimate a step filter's audience live, but there is **no per-node "entered/exited" funnel count** in the API; phrase drop-off as configured-filter explanation, not measured funnel. ◐
- **Aggregation lag can span weeks** on recent child runs (`status:'sent'` with stats=0 until settled — ~24 days observed once): spot-check that recent runs populate; if recent rows are all 0 while older populate, declare current-period volume unavailable rather than reporting the 0. ◐
- **Experiments live on the node message, not the journey** — `experiments[]` rounds repeat names; identify by `created`/`contentKey`; no significance field; `ho`/`uho` only on recent rounds (all field-confirmed live ✅; full shape in `entity-model.md`).
- **Heavy pagination is real** (one template ≈ 1,500 child sends across ~150 pages) — set a page budget and report a coverage gap rather than silently under-counting.
- **Child-send row shapes differ** ◐: `:rm…` rows carry `sentAt` + `audienceTotal`; `:d…` rows (status `interval`, daily aggregates) carry neither — window those on `createdAt`. Same-day duplicate rows can appear after a republish (different content hash) — dedupe before summing. A date hole present across sibling variants is a real feed gap — corroborate downstream rather than averaging over it.
- **Re-fetch ids live** — never reuse a cached/probe orchestration or message id; ids drift across accounts and time.

## Worked example (illustrative — fictional values)

A marketer asks: *"How has the welcome series performed over the last 30 days, and why does email 2 go to so many fewer people than email 1?"*

`whoami` confirms the account → `list_orchestrations` finds the journey via its `welcome_series` tag, `status: enabled` → `get_orchestration` maps 3 message nodes (T1 immediate; T2 delayed 3 days with a step filter excluding contacts already in a promotions group; T3 delayed 7 days) → per node, `list_child_sends` rows filtered to the 30-day window and aggregated → for the top runs, child `:rm…` ids fed to `get_message_performance` for per-run revenue:

| Node | Windowed sends | Adj. open rate | Clicks | Revenue (per-run) |
|---|---|---|---|---|
| T1 — Welcome | 84,000 | 41% | 6,900 | $61,000 |
| T2 — Drip (3d) | 52,000 | 33% | 3,100 | $24,000 |
| T3 — Drip (7d) | 47,000 | 29% | 2,400 | $18,000 |

The T1→T2 drop is **configured, not broken**: T2's step filter excludes contacts in the promotions group, and the 3-day delay sheds contacts who unsubscribe or convert in between (explanation from the composition; no per-node funnel count exists in the API). An `experiments[]` read on T1 shows a MAB subject-line test — current round identified by `created`, winner called descriptively on adjusted unique opens, significance not exposed. All numbers fictional, for shape only.


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
