---
name: tag-coverage-audit
description: Audit whether messages are tagged consistently enough to slice program/regional reporting — what share of the message corpus carries the tags a dashboard slices by (region, promo-type, channel), where the untagged gaps are, and which tag names are near-duplicates that fragment a dimension — so a CSM/marketer can decide whether to trust an O1/O12 sliced report before building it. Use when someone asks "are my messages tagged consistently enough to slice reporting", "what's our tag coverage across sends", "which messages are missing region / promo-type tags", "can I trust a regional report or are tags too patchy", "audit our message tagging / naming convention", "how many sends are untagged", "before I build a sliced report, check tag coverage", or "are our tags clean enough for a segmented QBR".
---

# Tag Coverage Audit

Before a marketer trusts a sliced (region / promo-type / channel) program report, tell them what **share of the message corpus actually carries the tags the dashboard slices by**, where the **untagged gaps** are, and which **tag names are near-duplicate or inconsistent** — so they know whether an O1/O12 sliced report is reliable or built on patchy tagging. Tag names and the account's tagging convention are discovered live and never hardcoded.

> Shared mechanics live in the project-root `_reference/` folder (sibling to `recipes/`, i.e. `reference/` from this file): the **no-reverse-index / where-used** pattern + the message-list **`updatedAt`-not-`sentAt` windowing trap** + the **suspicious-result law** in `tool-ergonomics.md`; string/value-completeness traps in `audience-query-mechanics.md`; the bucket/role/handoff-key map in `tool-index.md`. If the output is a rendered artifact, the brand spec in `cordial-brand.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the tag-coverage framing.

## Steps (the linked path)

1. **Orient (fail-closed)** — `whoami` to confirm the active account (`account`, `accountKey`). Tag taxonomies are per-account; never carry tag names across accounts. **If `whoami` errors with re-auth / token-expired / unauthorized, STOP here:** report the connector blocker and that the audit can't run, ask the user to re-authorize the Cordial connector, and do **not** fall back to the worked-example values below — there is no live data to compute coverage from.
2. **Establish the denominators** — `list_messages(type=batch)` and `list_messages(type=automated)` are **separate enumerations**; read **`totalEntries` from each** (a cheap `perPage=1` reads it). These two totals are your coverage denominators (batch corpus + automated corpus). **Handoff key: each type's `totalEntries`.**
3. **Discover the taxonomy** — `list_tags` (page through all pages, or use `query=<dimension>` to narrow when the user named one like "region"). Read the **tag names actually in use** and infer which reporting dimensions exist. **Handoff key: each tag `name` + its `assetCounts` / `totalAssets`.** Group near-duplicate names here (`Region-West` vs `region_west` vs `West`; `BOGO` vs `promo_bogo`) — this grouping **is** the consistency finding; it is a judgment over returned names, not a tool field. **Grouping tags into a reporting dimension is itself a judgment step that precedes any %** — e.g. whether `Outlet` is a promo-type value or a separate product-line dimension changes what "promo-type coverage" even means; decide (and state) the grouping before computing a number. If `query=<dimension>` returns `totalEntries=0` on a span where later pages are non-empty, that is a **genuine absence** of that dimension, not a paging artifact — say the dimension does not exist as a tag and **early-exit for that dimension** (coverage = 0%; do not call `get_tag` looking for it — there is nothing to read).
4. **Read per-tag usage counts** — read counts with **`list_tags(query=NAME)` as the primary/default source** — it is server-side name-filtered and reliable. `get_tag(name)` returns the same shape (`assetCounts.batchMessage`, `assetCounts.automationMessage`, `totalAssets`) but is **fragile**: names that `list_tags` returns verbatim still error "Tag not found" in `get_tag` with no discernible case/exactness rule (confirmed live: `Digital` works, `welcome`/`Welcome` and `Promo` error). Only reach for `get_tag` when you need an exact id-keyed read; otherwise stay on `list_tags(query=NAME)`. **Handoff key: `assetCounts.batchMessage` (and `.automationMessage`) = the coverage NUMERATOR per tag.**
5. **Compute coverage + flag gaps (terminal)** — per reporting dimension the dashboard needs:
   - **Coverage % (asset-level proxy)** = `tag.assetCounts.batchMessage ÷ list_messages(batch).totalEntries` (and the same for automated). State plainly this is an **asset/message-count proxy, not a coverage-of-sends figure** (an asset count is not a send count).
   - **Multi-tag dimension = a BOUND, not a point number.** The dimension's covered share is the count carrying **any** of its tags — but there is no reverse where-used index, so the per-tag counts **cannot be deduped** for messages carrying more than one of the tags. Summing them therefore over-counts. Report a **non-overlap CEILING** (sum of the tags) and a **floor** (the single largest tag), and say the true coverage sits in that range — never present a summed union as a precise %. If the absent-dimension early-exit fired in step 3, coverage is exactly 0% with **no numerator call needed** — the math reconciles trivially (0 ÷ corpus = 0%).
   - **Untagged share** = `1 − coverage` for the dimension; report as a count and a percent.
   - **Near-duplicate / inconsistent names** — surface the groups from step 3; these split one real dimension across multiple tags and silently fragment a sliced report.
   - **Unclassifiable bucket** — tags that map to no reporting dimension (date-stamps, `test`, legacy). Report as its own explicit slice; never force-fit.
   - **Scan-coverage note + verdict** — which corpora (batch / automated) and how many tag pages were read; then state whether a sliced O1/O12 report is trustworthy at the observed coverage or whether tagging is too patchy and needs backfill/consolidation first.

## Output

A tag-coverage audit. Per reporting dimension (region / promo-type / channel): **coverage %** (tagged assets ÷ corpus `totalEntries`), the **untagged count + share**, and the **near-duplicate/inconsistent name groups** that fragment that dimension — plus an **unclassifiable** bucket and an explicit **scan-coverage note**. State plainly whether a sliced O1/O12 report is trustworthy at the observed coverage or whether tagging is too patchy first. Every figure is an **asset-level proxy** (see guardrails), labelled as such. For a **multi-tag** dimension, report coverage as a **bounded range** (floor = largest tag, ceiling = sum), not a single point — per-tag counts can't be deduped. A dimension with **no tag namespace** (e.g. region) is **0% tag coverage**; route the reader to the field/name-parse alternative (see guardrails) rather than leaving them with just a zero.

- **Default (chat):** a coverage table per dimension + the gap/inconsistency lists. No branding needed.
- **If the user wants a shareable artifact** (HTML one-pager, slides, doc for a QBR): render it **Cordial-branded** per `reference/cordial-brand.md` — this recipe is self-branding and does **not** depend on any external brand-guidelines skill. Apply the palette/type/layout: Cheery `#FABC1C` on every asset; Aqua-led coverage bars/charts (Aqua > Pacific > Horizon); Horizon `#FF833E` as **text-only** eyebrows/section titles; coverage % as KPI cards with green/yellow/red variance chips (high coverage green, patchy yellow, untagged-heavy red); Poppins headings, Open Sans body.

## Guardrails

- **Read-only** — never imply a tag was created, applied, renamed, or that any message was changed.
- **Coverage is an asset-count proxy, not a send count, and not a per-message scan.** `list_messages` returns an **always-empty `tags:[]`** in the list summary (verified across pages and both message types) — per-message tags appear **only** in `get_message`, which is large and would require one call per message across the whole corpus (infeasible). So coverage is reconciled from `list_tags`/`get_tag` `assetCounts` over `list_messages.totalEntries`, **never** by scanning a per-message tag field. Label every figure as an asset-level proxy.
- **No reverse where-used index for tags.** `get_tag` returns asset COUNTS, never the list of which messages carry the tag — you cannot drill from a tag to its messages (see the where-used / no-reverse-index pattern in `tool-ergonomics.md`).
- **Both message types are separate denominators.** Read `totalEntries` for `type:batch` AND `type:automated`, or coverage under-counts. Automated messages have **no `sentAt`** and are lifetime-cumulative, so a **date-bounded** coverage figure is not derivable from `list_messages` alone — say so if the user asks for one. The list sorts by `updatedAt` not `sentAt`; never early-stop on page position (windowing pattern in `tool-ergonomics.md`).
- **The dimension may live in the message NAME or a message FIELD, not in tags.** Some accounts carry no structured region/promo-type tag namespace (e.g. `list_tags(query=region)` returns 0) — the taxonomy is a flat grab-bag and region/promo/channel data is encoded in a pipe-delimited message-name convention instead. If so, **report the absent tag dimension honestly** (tag-based coverage = 0%) and point to the realistic alternative — do not invent a tag-based coverage figure that does not exist:
  - **Channel** is a first-class message field, not a tag — slice it via the `channel` / `channelType` field on `list_messages` (and `list_channel_types` for the enumeration), **not** via tags. Tags named `email`/`sms` usually tag transforms/includes, not messages — don't mistake them for a channel dimension.
  - **Region / promo-type** with no tag namespace require a **name-token parse** of the pipe/underscore message-name convention (e.g. the `_US_` token in `BAT_060226_US_AM_...`); name which token carries the dimension. This is the parse step, not a tag read.
- **`get_tag` name-keying is fragile — use `list_tags(query=NAME)` as the primary count source.** A name returned verbatim by `list_tags` can still error "Tag not found" in `get_tag` with no discernible rule (`Digital` works; `welcome`/`Promo` error). `list_tags(query=NAME)` is the reliable read for every count; reserve `get_tag` for an exact id-keyed read.
- **Naming consistency is a judgment call, not a tool field.** Near-duplicate/inconsistent names are **surfaced as observations**, never asserted as a canonical correct name unless asked.
- **Required-tag policy is not MCP-enumerable.** Report **observed** coverage, never what the account is "supposed" to tag — there is no entitlement/policy endpoint.
- **Account-portable.** Discover this account's tags + naming at runtime via `list_tags`; never hardcode tag names or values.
- **Suspicious-result law** (`tool-ergonomics.md`): 100% coverage / 0 untagged / a `get_tag` count that doesn't reconcile against the corpus `totalEntries` must be re-checked (tag-field shape, the empty-`tags` list trap, the asset-vs-send distinction) before reporting. If the corpus is too large to deterministically cover within budget, report the scan-coverage gap rather than asserting a complete figure.

## Worked example (illustrative — fictional values, not from any account)

A marketer wants a regional performance report and asks whether tags are consistent enough first.

1. `whoami` → confirm account (connector live).
2. `list_messages(type=batch, perPage=1).totalEntries` = **4,000** and `list_messages(type=automated).totalEntries` = **40** — the two coverage denominators.
3. `list_tags` (all pages) → a flat taxonomy including promo tags `promo_bogo` and `BOGO` (same promo, two names), channel tags, and many single-use date-stamps. `list_tags(query=region)` → **0 hits** → there is **no region tag dimension**; region lives in the message-name convention instead.
4. `get_tag('promo_bogo').assetCounts.batchMessage` = 900; `list_tags(query=BOGO)` → another 600 — the BOGO promo is split across two names.
5. Compute + verdict:
   - **Promo-type coverage (proxy, bounded):** BOGO is fragmented — a naive `BOGO`-only slice catches 600/4,000 (**15%**) and silently drops the 900 tagged `promo_bogo`. Consolidated, coverage is **between ~23% (floor, the 900-count tag) and ~38% (ceiling, 1,500/4,000)** — the two tag sets can't be deduped, so report the range, not a point. Asset-level, not sends.
   - **Channel coverage (proxy):** the channel tag = 3,700/4,000 ≈ **92%** — reliable to slice.
   - **Region:** **no region tag exists** — a tag-based regional slice is not possible; region would have to be parsed from the message-name convention.
   - **Unclassifiable bucket:** hundreds of single-use date-stamp tags map to no reporting dimension.
   - **Verdict:** a **regional** report is **not trustworthy from tags** (no region dimension); a **promo** slice is patchy and fragmented (`BOGO`/`promo_bogo` must be consolidated first); a **channel** slice (~92%) is reliable. All figures are asset-level proxies, read-only — nothing was changed.


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
