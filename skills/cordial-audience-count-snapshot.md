---
name: audience-count-snapshot
description: Give a marketer/CSM a quick, LIVE "how many contacts are in this audience right now" count — for a named saved audience or an ad-hoc filter — so routine operational asks get answered without opening Audience Builder. Use when someone asks "how many contacts are in [audience name]," "what's the count for [audience/segment] right now," "audience count for today," "how big is the [X] audience," "how many people match [filter/criteria]," "size of this segment," or "current count for [saved audience]."
---

# Audience Count Snapshot

Answer "how many contacts are in this audience right now?" with a single live number for a named audience or an ad-hoc filter — fast, read-only, reported as an estimate.

> Shared mechanics live in the project-root `_reference/` directory (resolves via `reference/` — it sits at the **project root**, NOT under `recipes/`): query grammar + traps (**silent-zero**, the **universal base `{"and":[]}`** control, subscription-state + date-band grammar, the dotted-vs-comma key-path conversion) in `audience-query-mechanics.md`; tool-traversal patterns — **reference-before-build** and the **suspicious-result law** — in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the count-snapshot chain.

## The linked path

Connective keys: **`list_audiences[].id` → `get_audience_health(id)`** (resolve name → id) · **`get_audience_health.criteria` → `estimate_audience.criteria`** (THE core link — a ready-to-run clause object; do **NOT** trust `health.count`, it is a cached/stale snapshot) · **`estimate_audience.count` → the final reported number** · for ad-hoc filters: **`search_audience_examples[].data.query` / `get_account_audience_samples` criteria → `estimate_audience.criteria`** (lift live grammar; convert comma-keys to dots first).

1. **Orient.** `whoami` to confirm the connected account.
2. **Fire the liveness control FIRST.** `estimate_audience({"and":[]})` → the full file total. This confirms the tool is returning live numbers before you report anything — required so you never report a silent `0` as a real answer (see Limits).
3. **Discover.**
   - **Named audience:** `list_audiences(query="<name>")` → the matching `.id` (use `.name`/`.translation` to pick the right one; it is paginated — use `query=` to resolve a name rather than paging). A generic name can also resolve to a saved audience whose *whole definition* equals the asked-for segment surfaced via `search_audience_examples` (not always via `list_audiences(query=)`) — accept either route to the `.id`/criteria.
   - **Ad-hoc filter:** `describe_contact_schema` for the attribute key + type (drives the wrapper), then `search_audience_examples` / `get_account_audience_samples` to lift the account's **real** operator + value grammar. Never invent operators/values (reference-before-build).
   - **Name-vs-scope ambiguity (resolve, don't silently pick).** A phrase like "Subscribed Email" or "VIP segment" can mean both a broad ad-hoc filter (all email-subscribed) **and** a saved audience whose name closely matches but is *scoped narrower* (e.g. `Subscribed_EmailOnly` excludes contacts also subbed to SMS/push; "VIP" as a tier label vs. a multi-value segment). When a saved-audience name closely matches the phrasing, **state the resolved definition you used** (the actual clause/field+values) and note the scope difference, rather than silently returning one. Default to the broad filter for "[channel]-subscribed" phrasing unless the user named a specific saved audience.
4. **Inspect (named audience only).** `get_audience_health(id)` → hand `.criteria` (the machine clause object) forward. Do **not** report `.count` — it is a cached/tracked snapshot that can be stale.
5. **Terminal — get the live count.** `estimate_audience(criteria)` → `{count}`. This is the answer, reported as a live estimate.

## Output

One in-chat number, phrased as a live estimate as of now — e.g. "~38,170 contacts (live estimate as of now)." Optionally name the audience/filter scope and the channel/program if subscription state was involved. This deliverable is **numbers in chat only — no branding needed**.

## Guardrails

- **Read-only.** Never imply the audience was built, saved, refreshed, or sent — it's a live count of an existing definition or ad-hoc filter.
- **The count is an estimate as of call time,** not a materialized or billed number. Phrase as "~N (live estimate)."
- **Never trust `get_audience_health.count`** — re-estimate live via `estimate_audience(criteria)` every time. Equality between cached and live is luck, not proof.
- **Silent-zero trap:** an invalid operator/value/wrapper returns `count: 0` with no error. Fire the universal-base control `{"and":[]}` first; if it returns the file total, the tool is live, so a `0` on your query means re-check the path/operator (suspicious-result law) — not "no one."
- **Suspicious-result law:** a too-clean number or a tiny count where you expected a population (e.g. a literal value match on the wrong field level) means re-traverse to the correct field, don't report the small number.
- **Account-portable:** resolve audience names, attribute keys, and value grammar at runtime; bake in nothing account-specific.

## Honest tool limits (what the probe found)

- **`get_audience_health.count` is a cached/tracked snapshot — confirmed stale.** Observed a tracked audience whose cached `count` and every trend point sat flat while the live re-estimate was the real number; on another the cached count matched live only by coincidence. Always re-estimate via `estimate_audience`.
- **Untracked audiences have no trend history** — `trends.dataPoints` are all `null` (though `.count` is still present). Only tracked audiences populate daily trend counts. Don't read null trends as "no contacts."
- **Silent-zero is real:** invalid operator/value/wrapper → `count: 0`, no error. The `{"and":[]}` control returning the full file is the proof-of-liveness gate before reporting any `0`.
- **Dual key-path notation:** `estimate_audience` / `get_audience_health.criteria` use **dotted** paths (`channels.email.ss`); `search_audience_examples` returns **comma-delimited** keys (`channels,email,ss`). **Convert commas to dots before feeding `estimate_audience`** or you risk a silent zero.
- **Subscription-state (`channels.*.ss`) can be materialize-only on some accounts** and silently return `0` while live membership is large — don't report that as "no one." On the probe account it returned correct large live numbers, but the failure mode is account-specific: verify per account, and if an `ss` clause zeros, treat it as the silent-zero family, not a true zero.
- **`list_audiences` is paginated** (25/page, hundreds of audiences across many pages) — use `query=` to resolve a name rather than paging.
- **Counts are estimates** reported at the level the tools support; never infer hidden internals (the exact subscription model, retention windows). Single MCP (cordial) — cross-MCP probing N/A.

## Worked example (illustrative — fictional values)

A CSM asks: *"How big is the 30_Band audience right now?"*

`whoami` confirms the account → liveness control `estimate_audience({"and":[]})` ≈ 37,800,000 (full file — tool is live) → `list_audiences(query="30_Band")` → `id` resolves → `get_audience_health(id)` → `.criteria` ≈ `{"and":[{"icfs":{"preferred_bra_size":{"operator":"in","value":["30A","30B","30C","30D"]}}}]}` (its cached `.count` is ignored, trends all `null` = untracked) → `estimate_audience(that criteria)` → `{count: 38170}` live → **Answer: "~38,170 contacts (live estimate as of now)."**

Ad-hoc variant — *"How many contacts have a value for `loyalty_current_tier`?"*: `describe_contact_schema` → key + string type → `search_audience_examples` to lift the real `has_value` / `is_empty` grammar → `estimate_audience` the populated count. When the account supports a direct `has_value` operator, it's the simplest primary; cross-check it with `file_total − is_empty` (they should match exactly — both are in the coverage idioms in `audience-query-mechanics.md`). Note: `search_audience_examples` *translation* strings are channel/field-blind for `ss` and presence clauses (they render as "Filter by — Matches: s" with a blank label) — you cannot tell which channel/field from the translation; **read `data.query` keys** to identify it. Watch for the **VIP-style trap:** a literal value match on the wrong field level can return a tiny count (e.g. 82) that looks like "almost no one" when the real segment lives on a different field/value and is far larger — re-traverse before reporting. **All values above are fictional and for shape only.**


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
