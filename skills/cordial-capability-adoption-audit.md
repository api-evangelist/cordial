---
name: capability-adoption-audit
description: Account-level audit of which Cordial AI models and platform capabilities are ENABLED, which are actually being USED (contacts scored + dims targeted in live audiences/content), and which are still AVAILABLE to turn on — to prep a QBR or feature review and prioritize activations. Use when a CSM/marketer asks "what AI models / features do we have enabled?", "which capabilities are we using vs not using?", "feature/capability audit for this account", "what's live and what's next to activate / turn on?", "which AI models are we paying for but not using?", "go over all the features — which are enabled, which haven't we used yet?", or "prep a capability + adoption review for the QBR".
---

# Capability & Adoption Audit

For a CSM prepping a QBR/feature review: a single account-level picture of **what's enabled**, **what's actually being used**, and **what's left to turn on** — so activations can be prioritized. Discovered live; nothing about entitlements is invented.

> Shared mechanics live in `reference/`: query grammar + traps (esp. the **array `is_empty` silent trap**, silent-zero, and string-completeness reconcile) in `audience-query-mechanics.md`; tool-traversal patterns + the **suspicious-result law** in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`; the brand spec for the rendered output in `cordial-brand.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the three-state (enabled / used / available) capability framing.

## The three states — keep them distinct, this is the crux

- **ENABLED (observed)** — there's an artifact proving the capability is provisioned: a `crdl_ai_*` attribute exists, a channel type is configured, products/recs config is present. Phrase as **"observed enabled"** — this is inferred from artifacts, *not* a billing/entitlement endpoint.
- **USED** — split into two senses, never collapsed:
  - **SCORED** — contacts actually carry a value (`estimate_audience` populated count > 0). A model can be enabled but barely scoring (partial coverage).
  - **TARGETED** — the dim is actually referenced in a live audience/orchestration/content (`search_audience_examples` / where-used scan). A model can be enabled + scored but never targeted.
- **AVAILABLE to turn on** — the eligible-but-not-enabled slice. **The MCP cannot enumerate this** (see Limits). Report it as an explicit deferred slice, never fabricated.

## Steps (the linked path)

1. **Orient** — `whoami` to confirm the active account (`account`, `accountKey`).
2. **Capability inventory** — `get_account_overview_document` (lean) for the config surface; `list_channel_types` for enabled channels (`required` vs optional); `get_account_products_config` for the recs/catalog strategy (returns **prose, not JSON** — read narratively; supplement-based product storage gives weaker native-rec signal than catalog-native accounts). Optionally `list_orchestrations` / `get_account_supplements` for adjacent-capability signals.
3. **Discover observed-enabled AI models** — `describe_contact_schema`; every `crdl_ai_*` key present = one observed-enabled model. Record each `key`, friendly `name`, and `type`. **Enumerate from the live schema, never from a pre-supplied candidate list** — candidate lists (e.g. a probe or prior audit) can be stale, incomplete, or carry cross-account values; the schema is authoritative. (`crdl_ai_*_lm` date siblings and `cordial_ai_recom_*` support arrays are recs-pipeline artifacts, not standalone models — note, don't count.) **Handoff key:** schema `key` + `type` → drives the `estimate_audience` wrapper (string=`icfs`, number=`icfn`, date=`icfd`, array=`icfa`).
4. **Establish the denominator** — `estimate_audience {"and": []}` returns the whole contact file; this count is the coverage denominator. (Note if the file is unusually large / includes prospects/anonymous app installs — it inflates the denominator and deflates coverage %.) Optionally offer a **marketable-base denominator** as a secondary lens (e.g. an `email`-subscribed or channel-`has_value` base) so coverage isn't understated against a prospect-padded file.
5. **Measure SCORED coverage per model** — per type (reuse **ai-model-profiler** mechanics; profile each variant independently — e.g. send-time email vs SMS, a `_string` vs numeric sibling — they score differently):
   - **string / number — derive populated by arithmetic, not by complement:** `populated = file_total − direct is_empty(missing)`. Fire `{"and": []}` (file total) and `{"and": [{…is_empty}]}` (missing), then subtract. **Do NOT trust complement/`not`-wrapper/`is_not_empty` "populated" queries as primary** — on some accounts they silently return `0` or *echo the `is_empty` count*, so a literal reading reports 0% scored for fully-scored models. Red-flag signature: any "populated" result that **equals the `is_empty` count** (populated == missing) is wrong — use the subtraction. If `is_empty` *itself* returns 0 on a known-populated dim (it can be non-functional on strings), derive coverage from enumerated-value `in [v1,v2,…]` matches and take `file_total − populated` as the residual.
   - **array:** use **direct `{"and": [{"icfa": {key: {"operator": "has_value"}}}]}`** for the populated count. **`is_empty` is a silent trap on arrays** — direct `is_empty` returns 0 and the `is_empty`-exclude pattern returns the FULL file. Never use `is_empty` for array coverage.
   - **Handoff key:** schema `key` + `type` → criteria wrapper → `estimate_audience.count` (the only return field, a raw int).
6. **Measure TARGETED usage** — `search_audience_examples` across the `crdl_ai_*` names/keys: is any model actually referenced in a live audience? `search_audience_examples` is **semantic, not an exhaustive where-used index**, so a zero result needs a **contrast control**: run a query for a dim you *know* is heavily targeted (a real non-AI segment dim) and confirm it surfaces with explicit criteria — that proves the tool reads live criteria and the AI-zero is real, not a tool miss. (To fully harden a "zero" claim, a deterministic `list_audiences` + `get_audience` criteria grep is the exhaustive path.) If none surface, that's a strong **scored-but-not-used** signal (live targeting is often event/segment/channel-based instead). For content usage, note a where-used scan (`get_message`/sculpt/include grep for `{{ ...crdl_ai_* }}`) would be needed to fully confirm — recs/send-time models are often consumed in content/orchestration send-timing rather than audience filters, so flag content usage as not-yet-run rather than asserting unused.
7. **Defer the AVAILABLE slice** — name what's enabled, then state the eligible-but-not-enabled catalog is **not observable via MCP**; defer to Cordial docs / the `cordial-support` skill / AI model cards. Report as an explicit unidentified slice.
8. **Render** — hand the reconciled enabled/scored/targeted table to a branded HTML/PPTX feature-review one-pager using the brand tokens in `reference/cordial-brand.md` (Cheery must appear; Aqua-led charts; Horizon as text-only section titles; coverage % as KPI cards with green/yellow/red variance chips).

## Output

A QBR-ready capability audit, three columns: **Enabled (observed)** · **Used (scored coverage % + targeted yes/no)** · **Available (deferred to docs)**. Per enabled AI model: friendly name, scored count + share of the file, and whether it's targeted in any live audience. A prioritized "next to activate" read (high coverage but never targeted = activation opportunity; low coverage = provisioned-but-not-scored). Rendered as a Cordial-branded one-pager per `cordial-brand.md`.

## Guardrails

- **Read-only** — never imply anything was enabled, activated, built, saved, or sent. All counts are **estimates**.
- **"Enabled" is observed, not entitled** — inferred from `crdl_ai_*` presence / channel types / products config, not a billing endpoint. Always phrase "observed enabled." A model can be provisioned but not yet scoring — catch via the populated count.
- **Scored ≠ used.** Keep the two senses distinct: SCORED (populated count) vs TARGETED (referenced in a live audience/content). Report both; never let one imply the other.
- **Array coverage uses `has_value`, never `is_empty`** — `is_empty` on arrays is a silent trap (see `audience-query-mechanics.md`). Any prior audit using `is_empty`-exclude on an array AI dim would have wrongly reported the full file as covered.
- **String/number coverage = `file_total − is_empty`, never a complement query** — complement/`not`-wrapper/`is_not_empty` "populated" queries silently return 0 or echo the `is_empty` count on some accounts. Any "populated" value equal to the `is_empty` count is a red flag; use the subtraction (see `audience-query-mechanics.md`).
- **The available-to-turn-on catalog is not observable** — there is no MCP endpoint enumerating Cordial's full feature/SKU list. Defer that slice to docs / `cordial-support` / model cards and report it as explicit; never fabricate eligibility.
- **AI values are the model's verdict** under its own definition/lookback, not literal fact — phrase accordingly; point to the model card for canonical definitions.
- If a result looks suspicious (0 / too clean / doesn't reconcile), don't report it — find the correct traversal (see `tool-ergonomics.md`).

## Worked example (illustrative — fictional round numbers, not from any account)

Account file (denominator via `{"and": []}`) = 30,000,000 contacts. Observed-enabled AI models from `describe_contact_schema`, scored coverage measured per type:

| Model (observed enabled) | Type | Scored | Coverage | Targeted in a live audience? |
|---|---|---|---|---|
| AI Purchase Propensity | string | 27,000,000 | 90% | no |
| AI Send Time (Email) | number | 4,800,000 | 16% | no |
| AI Channel Affinity | string | 1,800,000 | 6% | no |
| AI Category Affinity | array (`has_value`) | 1,200,000 | 4% | no |
| AI Product Recommendations | array (`has_value`) | 600,000 | 2% | no |

Reconcile (string): scored = file_total − missing(`is_empty`) = 30,000,000 − 3,000,000 = 27,000,000 ✅ (subtraction is primary; never trust a complement "populated" query that equals the `is_empty` count). Array note: `is_empty` direct on Product Recommendations → 0 (trap), so `has_value` is required. Usage scan: `search_audience_examples` surfaced **0** audiences referencing any `crdl_ai_*` key → every model above is **scored but not targeted** — the headline activation opportunity for the QBR. **Available-to-turn-on** (models not present as attributes): deferred to Cordial docs / `cordial-support` / model cards — not enumerable via MCP.


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
