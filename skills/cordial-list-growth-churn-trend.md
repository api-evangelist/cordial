---
name: list-growth-churn-trend
description: Show subscriber/list growth and net churn (gross subscribes minus unsubscribes) by month across a multi-year window that exceeds the platform's native 12-month dashboard limit — overall or for a named audience. Use when a marketer/CSM asks "show subscriber growth by month for the last 2 years," "net subscriber gain/loss by month beyond 12 months," "list growth and churn trend," "subscribes vs unsubscribes per month," "year-over-year sign-up growth," "net subscribers added/lost for this audience," "the dashboard only goes back a year — pull more history," or "monthly emailable list size over time."
---

# List Growth & Churn Trend

Help a marketer or CSM see month-by-month subscriber growth and **net churn** (gross subscribes − unsubscribes) across a multi-year window — the history the native dashboard can't show past ~12 months — for the whole list or one named audience.

> Shared mechanics live in the project-root `_reference/` directory (resolves via `reference/` — note it is at the **project root**, NOT under `recipes/`; a cold agent that looks under `recipes/_reference/` will find nothing): query grammar + traps (silent-zero, progressive-exclusion, subscription-state grammar, `icfd` date bands, the channel-scoped **subscribe/unsubscribe date-band** grammar) in `audience-query-mechanics.md`; tool-traversal patterns — especially the **windowable-vs-lifetime-source** trap and the **suspicious-result law** — in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`; the brand spec for any rendered output in `cordial-brand.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds what's specific to the growth/churn time series.

## The linked path

Connective keys: **event/date-attr name + channel → per-month `estimate_audience` band → count → net = subscribes − unsubscribes**; and (per-audience) **`list_audiences` id → `get_audience_health(id).criteria` → AND-merged onto each band**.

1. **Orient.** `whoami` to confirm the account. Confirm the **window** (e.g. 24 months), the **channel** (email vs SMS/push — they use different paths, see Limits), and whether the ask is **account-wide or for a named audience**.
2. **Reference the schema, then learn the band grammar from a live example.** `describe_account_events` + `describe_contact_schema` orient you. **But the PRIMARY band source — the channel-scoped `subscribedDate`/`unsubscribedDate` clause — is a system path, NOT a contact attribute**, so it will **not** appear in `describe_contact_schema`. Learn its exact wrapper from `search_audience_examples`/`get_account_audience_samples` before banding (grammar pinned in `audience-query-mechanics.md`). Handoff = the **channel-scoped date-clause key + channel**. (A contact signup-date attribute like a `*_subscribe_date` field also exists and is schema-visible, but it measures a *different* thing — see Limits.)
3. **Discover the audience (only if named).** `list_audiences(query)` → pick the id → `get_audience_health(id)` to echo the full **criteria JSON**. Handoff = that criteria, to be `AND`-merged onto every monthly band. (Treat `get_audience_health` here as a criteria source — see Limits re: its trend `dataPoints`.)
   - **3a. Pre-flight base check (required gate — do this before firing ~48 band calls).** Estimate the raw audience criteria first. If the base is `0` / near-empty / `tracked:false` with `null` dataPoints, **stop and re-scope** rather than running months of bands that all return 0.
   - **3b. Classify every clause before merging.** A clause is safe to `AND` onto a *historical* band only if it is **time-invariant** (static attributes/segments). Two clause kinds **fight** a historical flow and must be **stripped** (and the user told which were dropped): (i) **point-in-time subscription-state snapshots** (e.g. `channels.email.ss matches ["s"]`) and (ii) **moving relative behavioral windows** (e.g. "clicked within the past 30 days"). Both conflate a present-tense state with a past-month flow. Named guardrail: **never `AND` a currently-subscribed (`ss=["s"]`) clause onto an unsubscribe-date band** (or an opted-out clause onto a subscribe band) — it is a guaranteed ~0 self-contradiction, not real churn. If stripping these leaves a meaningful static cohort, band on that and say so; if not, re-scope to a stable static segment instead.
4. **Terminal — per-month net derivation (PRIMARY).** For each month in the window, run two `estimate_audience` calls on a channel-scoped subscribe/unsubscribe **date band** and difference them:
   - subscribes(month) = count of the **subscribe-date band** `[start, nextStart)`
   - unsubscribes(month) = count of the **unsubscribe-date band** `[start, nextStart)`
   - **net(month) = subscribes − unsubscribes**
   The date-band clause is the **date-windowable** source (the grammar is in `audience-query-mechanics.md`) — it honors arbitrary start/end and has **no 12-month ceiling**, so it's how you exceed the native dashboard limit. Per-audience: `AND` the step-3 criteria onto each band.
5. **Point-in-time list size (optional add-on, and the running-total anchor).** For "emailable list size now," size the subscription-state path matching the **subscribed** value (one estimate). This is a snapshot, not a flow — keep it separate from the monthly flows. Note this snapshot can be finicky: the subscription-state path that works for *filtering* an audience does not always return a count via `estimate_audience` (probes saw `0` for both dot- and comma-path forms even when the value is large) — confirm it returns a non-zero count before relying on it.
6. **Assemble + (optional) render.** Tabulate month / subscribes / unsubscribes / net. The **running-total** column is contingent on a working point-in-time anchor (step 5): bands give *flows* only, so a running emailable total needs a current subscribed snapshot to anchor against. If that snapshot can't be retrieved, **omit the running-total column** (and say so) — never fabricate a baseline. If the marketer wants a shareable artifact (chart or exec one-pager), render it Cordial-branded per `reference/cordial-brand.md` (Cheery on the asset; Aqua-led bars/lines; Horizon text-only section titles; green/yellow/red variance chips for net + / flat / −). A plain in-chat table needs no branding.

## Output

A monthly time series across the requested window: **subscribes, unsubscribes, net change** per month (plus a **running list total** only if a working snapshot anchor exists — see step 5/6), scoped account-wide or to the named audience and channel. State the window, channel, and whether scoped. If rendered, a Cordial-branded growth/churn chart + table. Every figure is an **estimate** — phrase as such, never as an exact ledger.

## Guardrails

- **Read-only.** Never imply an audience, report, or export was created, saved, or sent — all figures are live estimates.
- **All counts are estimates.** Phrase the series as estimated subscribes/unsubscribes/net, never as a definitive ledger.
- **Unsubscribe ≠ never-subscribed.** Subscription state is multi-valued (subscribed / opted-out / none). **Net churn uses opted-out only** — never conflate the opt-out value with the never-subscribed/none value, or you'll overstate churn.
- **Channel matters — don't share one number across channels.** Email subscription state lives on a different path than SMS/push opt-in (SMS is often program-scoped; push can have loud vs quiet opt-in). Report per logical channel; never reuse an email figure as an SMS/push figure.
- **Windowable vs lifetime trap.** The deliverable is a *windowed* flow. Only use a source that honors arbitrary start/end (the per-month date band). Do **not** present a lifetime total or a snapshot as if it were one month's flow.
- **Apply the suspicious-result law** (`tool-ergonomics.md`): a `0` or a too-clean number means re-check the path/operator (fire the universal-base control first), not "no activity." For a *spike* (e.g. an unsubscribe month ~20× its neighbors), don't dismiss it as an artifact — re-fire, then **sub-band within the month** to confirm it reconciles to the full-month count. If it's a genuine one-time event (a bulk list-hygiene/suppression purge), **report the totals both ways** — with and without the one-time event — so the underlying organic trend isn't masked.

## Honest tool limits (what the probe found)

- **The account-wide time-series tool is not a subscribe/unsubscribe ledger.** `get_audience_trends` *does* honor a custom multi-year start/end (no 12-month ceiling), **but** with no audience filter it returns **daily audience-size** points for *all* audiences — a multi-megabyte payload that can exceed context, and it measures audience **size**, not gross subscribes/unsubscribes. Do **not** use it as the net-churn source; use the per-month `estimate_audience` date bands instead.
- **`get_audience_health` trend history exists only for *tracked* audiences.** For an untracked audience its trend `dataPoints` come back `null` (and certain period keys produced forward-looking dates) — treat `null` as "not tracked," **not** zero. Use `get_audience_health` here only to lift the audience's **criteria**, then derive the flow from date bands.
- **Event-based subscribe date ≠ signup-date attribute.** A channel-scoped subscribe/unsubscribe **event** date and a contact **signup-date attribute** measure different things and return different counts for the same month — **pick one and state which**; don't treat them as interchangeable.
- **A small AND'd base can yield a legitimate 0 with a date band** (no grammar error) — verify the base is non-empty before reading a 0 as "no subscribes that month."
- **Early-month gaps are gaps, not zero.** If a month's subscribe/unsubscribe count can't be retrieved (UI/data-retention limit), state the gap explicitly — never back-fill a guess.
- **Per-month banding is many calls.** A 24-month window account-wide is ~48 `estimate_audience` calls (×N for multiple audiences/channels). Budget for it; if you must truncate, say which months are covered.
- Single MCP (cordial) — cross-MCP probing N/A. Report at the level the tools support; never infer hidden internals (the exact subscription-state model, retention windows, or attribution logic — defer to Cordial's definition).

## Worked example (illustrative — fictional values)

A marketer asks: *"Show net email subscriber growth (subscribes − unsubscribes) by month for the last 24 months — our dashboard only goes back a year."*

`whoami` confirms the account; channel = email, scope = account-wide, window = 24 months. `describe_account_events` orients; the `subscribedDate`/`unsubscribedDate` channel-scoped band grammar is learned from `search_audience_examples` (it is not a schema attribute). For each month, two `estimate_audience` calls on a `[start, nextStart)` band, e.g. for one illustrative month (running total shown only because a working subscribed-snapshot anchor was available):

| Month | Subscribes | Unsubscribes | Net | Running total |
|---|---|---|---|---|
| Jan (yr 1) | 120,000 | 64,000 | +56,000 | 12,556,000 |
| Feb (yr 1) | 98,000 | 71,000 | +27,000 | 12,583,000 |
| … | … | … | … | … |

For a named audience, `list_audiences` → `get_audience_health(id)` → its criteria is `AND`-merged onto every band. Current emailable size is a separate snapshot from the subscribed subscription-state value. Rendered to a branded line/bar chart per `cordial-brand.md` (net bars colored green/red, Aqua subscribe line, Cheery accent). **All values above are fictional and for shape only.**


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
