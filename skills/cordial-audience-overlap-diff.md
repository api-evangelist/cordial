---
name: audience-overlap-diff
description: Run set operations across two audiences (or an audience and a loaded list) live and reconcile them — intersection, A-not-B difference, complement, rule-by-rule diff of why two audiences differ, and external-list overlap (e.g. how many on a suppression list are still subscribed). Use when a marketer/CSM asks "why does this audience pull more contacts than the client's file," "how many on this suppression list are still subscribed/emailable," "what's the overlap between audience A and audience B," "diff these two audiences rule by rule," "how many are in both segments / in A but not B," "reconcile the count difference between two segments," or "cross-tab my loyalty members by AI attribute." Cordial has no native audience-diff or overlap endpoint, so the answer is composed by hand and reconciled.
---

# Audience Overlap & Diff

Tell a marketer/CSM **why two audiences differ** (rule by rule, with the size of the gap) and **how much one list overlaps another** (intersection / difference / complement) — composed live as set operations and reconciled, because Cordial has **no native audience-diff or overlap endpoint**.

> Shared mechanics live in the project-root `_reference/` directory (resolves via `reference/` — it is at the **project root**, NOT under `recipes/`): query grammar + traps (silent-zero, the **`exclude` needs a base** rule, the **universal base `{"and":[]}`**, progressive-exclusion, subscription-state + date-band grammar) in `audience-query-mechanics.md`; tool-traversal patterns — the **reference-before-build** rule and the **suspicious-result law** — in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the set-op composition + reconciliation specific to overlap/diff.

## The linked path

Connective keys: **`list_audiences[].id` → `get_audience_health(id).criteria`** (lift each side's clause set — `list_audiences` does **not** return criteria, you must round-trip through health) · **`get_account_supplements`.Key → `supplements_<key>` `has_value` clause** (external/suppression list) · **`channels.<channel>.programs.<program>.ss matches ["s"]`** (subscribed-state clause, ANDed onto membership) · **composed criteria → `estimate_audience` → `{count}`** (terminal; reconcile across calls by hand).

1. **Orient.** `whoami` to confirm the account. Pin the question type: overlap (A∩B), difference (A-not-B), complement, **rule-by-rule diff**, or **external-list overlap**. Pin the channel/program if subscription state is involved (see Limits — marketing vs transactional are different paths).
2. **Discover.** `list_audiences(query=…)` → the `id` for each named audience (A, B). For an external/suppression list: `get_account_supplements` / `list_supplements` → the supplement **Key**.
3. **Inspect — lift each side's clause set.** `get_audience_health(id)` for each audience → `.criteria` (the machine clause object) + `.count`. **`list_audiences` omits criteria — this hop is mandatory** before any set-op. `get_account_audience_samples` / `search_audience_examples` are the alternate criteria sources. **`.count` is a cached/tracked snapshot, not live — it can be stale (observed: cached `0`/`1` while live was thousands).** Use `.criteria` for the clause object but **re-estimate each side live via `estimate_audience` for the actual diff** — don't trust `health.count`.
4. **Reference before composing.** `search_audience_examples` / `get_account_audience_samples` → lift the **live operator + wrapper + values** for any clause you compose (subscription state, supplement membership, date bands). Never invent grammar — per the reference-before-build rule. Fire the **universal-base control `{"and":[]}`** once first to confirm the tool is returning live numbers (it returns the full file total).
5. **Terminal — compose the set-op, then reconcile.** `estimate_audience` returns only `{count}`; build each op as criteria and reconcile by hand:
   - **Intersection (A∩B):** `AND` both criteria — `{"and":[ …A… , …B… ]}`.
   - **Difference (A-not-B):** base on A, exclude B — `{"and":[ …A… ], "exclude":{"or":[ …B… ]}}`. **Never a bare top-level `exclude`** (silent-zero, returns 0); difference/complement must base on a real `{"and":[…]}` or the universal base `{"and":[]}`.
   - **Complement / file-total:** universal base `{"and":[]}`, optionally `exclude` the cohort. (`{"allContacts":true,"exclude":{…}}` is an equivalent base form seen in the wild.)
   - **Rule-by-rule diff:** estimate each side, then peel A's clauses off B one at a time (and vice-versa) and watch the residual; attribute each chunk of the gap to the clause that closes it. **Reconcile to the known gap** (e.g. A−B should equal the reported "18k more"); any unattributable count is an **explicit residual**, never force-fit to a rule. **No clean single-rule A/B pair?** When the two named audiences don't differ by one isolable clause (e.g. they cross the marketing-vs-transactional path boundary, or differ on several clauses at once), don't diff two unrelated saved audiences — compose the faithful diff from **one** audience's lifted clauses, peeling one clause at a time, and tell the user that's what you did.
   - **External-list overlap:** `AND` the supplement-membership clause (`{"supplements_<key>":{"operator":"has_value"}}`, **top-level**, not under `icfs`/`icfa`) onto the subscribed-state clause — e.g. "on the suppression list AND still SMS-subscribed."
   - **Cross-tab (loyalty × AI):** `AND` the loyalty-member criteria onto each AI-attribute value (see the `audience-ai-breakdown` recipe for the progressive-exclusion enumeration of AI values).
6. **Sanity-check the reconciliation.** Difference + intersection should add back to the base (A = (A-not-B) + (A∩B)). If it doesn't reconcile, re-check the path/operator (suspicious-result law), don't report it. **Sanity-check each definition's magnitude before trusting it:** a membership/loyalty clause that returns a tiny count vs the file total (e.g. 82 of 37M) is likely a legacy micro-cohort or the wrong field, not the real population — verify against the live field (e.g. the actual loyalty-tier attribute via `describe_contact_schema`) before composing the set-op. A cold agent that takes such a seed literally reports a false `0` overlap.

## Output

In-chat numbers + a small reconciliation table — e.g. A, B, A∩B, A-not-B, B-not-A, and (for a diff) a rule-by-rule attribution of the gap with an explicit "unattributed residual" row. State the channel/program scope, that **all counts are estimates**, and any reconciliation residual as its own line (never absorbed). This deliverable is chat-first and needs **no branding**. *Only if the user asks for a rendered, shareable artifact* (e.g. a branded cross-tab one-pager or chart of the loyalty × AI split), render it Cordial-branded per `reference/cordial-brand.md` (Cheery on the asset; Aqua-led bars; Horizon text-only section titles; green/yellow/red variance chips) — the recipe is self-branding and depends on no external brand skill.

## Guardrails

- **Read-only.** Never imply an audience, list, or report was built, saved, or sent — every figure is a live estimate.
- **All counts are estimates;** phrase as such, never as an exact ledger.
- **`exclude` needs a base.** A bare top-level `exclude` returns `0` (silent-zero). Always base difference/complement on `{"and":[…]}` or the universal base `{"and":[]}`, and fire the `{"and":[]}` control first.
- **Classify clauses before merging.** A lifted criteria can mix **time-invariant** rules (static attributes/segments — safe to AND/exclude) with **point-in-time subscription snapshots** (`channels.*.ss`) and **moving relative behavioral windows** (e.g. "clicked within the past N days"). The latter two fight a clean set-op merge — classify before merging or the op chases a moving target.
- **Don't force-fit the gap.** Rule-by-rule diff is bounded by the granularity of each audience's lifted criteria. An unattributable count gap is an **explicit residual**, not a rule.
- **Cited figure not reproducible is a first-class branch.** If the user asserts a gap ("18k more") that the live base can't contain (the whole population is smaller, or the account is in a sparse/snapshot-stale state), report the **real reconciled gap** and **state the asserted figure can't be reproduced in this account's live data** — do not force-fit the answer to the cited number or invent the gap.
- **Apply the suspicious-result law** (`tool-ergonomics.md`): a `0`, a too-clean number, or a non-reconciling result means re-check the path/operator, not "no overlap."

## Honest tool limits (what the probe found)

- **No native diff/overlap/set-op endpoint.** `estimate_audience` returns only `{count}` — all intersection/difference/complement are composed by hand and reconciled across multiple calls (verified: a base of 82 minus a clause → 75, intersection → 7; 82−75=7 reconciles exactly). Budget for several calls per question.
- **`list_audiences` omits criteria.** You must round-trip through `get_audience_health(id)` (or `get_account_audience_samples` / `search_audience_examples`) to lift the machine clause object — one extra hop per audience.
- **Supplement key ≠ display name, and sometimes ≠ the `get_account_supplements` Key.** The membership clause is `supplements_<internalKey>` at top level; the suffix is the supplement's **internal** key. Verify the `supplements_<key>` `has_value` count against the supplement's Records count — a `0` means **wrong key, not zero overlap**; lift the exact `supplements_*` key from a live `search_audience_examples` result when in doubt.
- **A loaded supplement won't perfectly reconcile to its membership count.** Record count vs `has_value` count differ by a small margin (unlinked / duplicate contact IDs). Report the gap as an explicit residual — don't force-fit.
- **A truly external file not loaded into the account cannot be cross-referenced via MCP.** The list must already exist in-account as a supplement / data table; cross-reference = supplement-membership `AND` subscribed-state.
- **Channel/program path matters.** SMS marketing (`channels.sms.programs.<program>.ss`) vs transactional (`channels.sms-txn…`) are different subscription states — the wrong one silently changes the overlap answer. Email is `channels.email.ss`. Pick the path for the actual question. **Also watch the grammar form:** examples may display the key comma-delimited (`channels,email,ss` / `channels,sms,programs,<program>,ss`); the dotted form (`channels.email.ss`) is what worked through `estimate_audience` on one account, the comma-key form was the live saved-audience form on another — lift the exact key from a live example and, if one form silent-zeros, try the other.
- **Subscription-state (`ss`) can be un-estimable (materialize-only) on some accounts.** Observed: every `ss` clause (any channel/program, dotted *or* comma-key, standalone *or* ANDed) silently returned `0` through `estimate_audience`, while every non-`ss` clause (supplement `has_value`, `isNotInvalid`, universal base) returned correct live numbers — isolating the failure to the `ss` family, **not** a true zero. When `ss` won't size: (a) **route to an estimable proxy** — `isNotInvalid:<channel>` for validity/emailability, or an opt-in attribute (`newsletter_option`, `*_marketing_option`, `membership_segment`) — and state the proxy is **not identical** to true subscribed-state; (b) offer the **materialize-then-count** follow-up (`get_audience_health` on an existing `Subscribed_*` audience ANDed with the list) as a platform step `estimate_audience` can't do. Per the suspicious-result law, do **not** report the `0` as "no overlap."
- **List name ≠ asked channel.** An SMS-named suppression list with an EMAIL question (or vice-versa) is a likely mismatch — answer the channel literally asked, but flag the name/channel mismatch and confirm intent before sizing.
- **Counts are estimates.** Report at the level the tools support; never infer hidden internals (the exact subscription-state model, retention windows, attribution logic) — defer to Cordial's definition. Single MCP (cordial) — cross-MCP probing N/A.

## Worked example (illustrative — fictional values)

A CSM asks: *"We loaded a third-party SMS suppression list as a data table — how many of those contacts are still subscribed to SMS marketing?"*

`whoami` confirms the account → `get_account_supplements` surfaces the suppression list (Key `sms_suppression_q3`, ~3.30M records) → universal-base control `estimate_audience({"and":[]})` returns the full file (~37.8M — tool is live) → membership `estimate_audience({"and":[{"supplements_sms_suppression_q3":{"operator":"has_value"}}]})` ≈ 3,300,000 (record count differs by ~20 unlinked IDs — reported as residual) → overlap `estimate_audience({"and":[{"supplements_sms_suppression_q3":{"operator":"has_value"}},{"channels.sms.programs.brand.ss":{"operator":"matches","value":["s"]}}]})` ≈ **1,000** → **Answer: ~1,000 contacts on the suppression list are still subscribed to SMS marketing.**

Difference cross-check (same account): base cohort ≈ 82; cohort minus a sub-clause via `{"and":[…cohort…],"exclude":{"or":[…clause…]}}` ≈ 75; intersection `{"and":[…cohort…,…clause…]}` ≈ 7; 82 − 75 = 7 reconciles exactly. **All values above are fictional and for shape only.**


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
