---
name: ai-model-profiler
description: Explain and profile a single Cordial AI attribute (a crdl_ai_* field). Use when a marketer or CSM asks what an AI attribute means, what its distinct values are and how many contacts fall in each, why a large share of the file has no score, or what feeds the model. Triggers include "what does crdl_ai_… mean", "unique values + counts for this AI attribute", "why do half my contacts have no [propensity/affinity] score", "what data feeds this model". Profiles the attribute itself across the whole file (or a stated base), reconciling every slice.
---

# AI Model Profiler

Marketers target on `crdl_ai_*` attributes but can't see inside them. This recipe opens one up: its real values, the contact count and share for each, the no-score cohort explained, and a plain-language read of what the model means and what feeds it — all discovered live, nothing invented.

> Shared mechanics live in `reference/`: query grammar + traps in `audience-query-mechanics.md` (esp. the self-learning loop, silent-zero trap, progressive-exclusion enumeration); tool-traversal patterns in `tool-ergonomics.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe adds the per-attribute profiling method and the "what feeds it" trace.

## Pick the method by the attribute's type — this is the crux

`describe_contact_schema` gives each `crdl_ai_*` attribute a friendly `name` (the portable human label — there is no description field) and a `type`. The type decides how you profile it. The three variants of even one model can differ in coverage, so **profile each attribute on its own; never assume one stands in for another.**

| Type | Wrapper | Method | No-score cohort |
|---|---|---|---|
| **string / bucketed** (e.g. propensity_string, price_sensitivity, *_category, momentum, unsubscribe_risk) | `icfs` | Enumerate values by **progressive exclusion** to a 0 residual | `is_empty` *and/or* a real `no <activity>` bucket — probe both |
| **number** (e.g. propensity 0–100, send_time HHMM, optimal_weekly_messages) | `icfn` | **Band** with `gt`/`lt` into meaningful ranges | `is_empty` |
| **array** (e.g. category_affinity, brand_affinity, product_recommendations) | `icfa` for coverage, `icfs` for a named member | **Coverage** (`icfa has_value`) + **size a named member value** (`icfs matches "<value>"`). A *reconciling* distribution isn't possible — members overlap (a contact holds several), so value slices exceed coverage and there's no distinct-values endpoint. Trace the vocabulary instead. | not-`has_value` |

## Steps (the linked path)

1. **Orient** — `whoami` to confirm the account.
2. **Find the attribute** — `describe_contact_schema`; record its `key`, friendly `name`, and `type`. The type selects the method above.
3. **Establish the base / denominator** — the universal base is an empty `and`: `{"and": []}` returns the entire contact file. Use that count as the denominator (or `AND` the attribute onto a base the marketer named). Reference: lift the live operator/wrapper from `search_audience_examples` before building any clause.
4. **Profile by type:**
   - **String:** slice each value (`icfs matches "<v>"`), then run the **deterministic residual** — base `{"and": []}` with `exclude: { or: [ is_empty, v1, v2, … ] }`. Residual → 0 proves completeness, and this residual is REQUIRED: an additive sum of the slices that happens to match the file total is suggestive but NOT proof (a hidden value can coincidentally sum) — always fire the exclusion residual. A large residual is an unnamed value (often a `no <activity>` bucket like `no purchase` / `no engagement`) — probe it and re-measure. **Cap the effort**; these are free-form strings.
   - **Number:** count `is_empty` (no-score), then band with `icfn gt/lt` into ranges that mean something for the model (e.g. a top band, mid, low). Bands + is_empty should reconcile to the base.
   - **Array:** count `icfa has_value` vs the base for coverage. If the marketer names a member value, size it with `icfs matches "<value>"` (this works on array members). Do **not** present an enumerated distribution — members overlap so slices won't sum to coverage, and there's no distinct-values endpoint; **trace the vocabulary** instead (next step).
5. **Trace "what feeds it" (live, portable):**
   - **Affinity / recommendation models** map to the product catalog taxonomy — `get_account_products_config` exposes the catalog's category/brand fields the model draws on, and `describe_account_events` shows the browse/purchase events that drive it. State the trace (e.g. "category affinity is built from product-browse/purchase events against the catalog's `category` field").
   - **Engagement / propensity / momentum / RFM** are behavioral — `describe_account_events` (clicks, opens, carts, purchases) shows the inputs.
   - For the **canonical definition, inputs, and lookback window, defer to the Cordial AI model card** (or the `cordial-support` skill). Do not invent the definition — read what's traceable, name the rest as model-card territory.
6. **Report** — friendly name; per-value (or per-band) count and share of the base; the no-score cohort split into `is_empty` vs any `no <activity>` bucket; coverage for arrays; a one-line plain-language read of what the model expresses and what feeds it; explicit residual/"other (value not identified)" if anything didn't close.

## Output

A profile of one AI attribute: its values/bands with counts and shares that reconcile to the base, the no-score cohort explained, the feeds-it trace, and a plain-language meaning — with any unresolved slice named honestly.

## Guardrails

- **Read-only** — never imply an audience was built, saved, or sent. All counts are **estimates**.
- **AI values are the model's verdict, not literal fact.** Phrase as "the model classifies ~X% as 'no purchase'," not "X% never bought." A value reflects the model's definition and lookback window; if the exact meaning matters, point to the model card rather than asserting it.
- **No-score is two different things** — `is_empty` (truly unset) vs a real `no <activity>` value. Keep them separate; probe for both; never fold one into the other.
- **`exclude` needs a base.** A top-level `exclude` with no `and`/`or` base returns `0` (a silent-zero variant) — always pair it with a base (`{"and": [], "exclude": {…}}`). Likewise, a value-match `0` means "re-check the value," not "none."
- **Arrays:** report coverage (`icfa has_value`) and, if asked, a named member's size (`icfs matches`) — but never present an enumerated distribution as if it reconciles (members overlap; slices exceed coverage). Trace the vocabulary to the catalog/events instead.
- If a result looks suspicious (0 / too clean / doesn't reconcile), don't report it — find the correct traversal (see `tool-ergonomics.md`).

## Worked example (illustrative — fictional round numbers, not from any account)

Profiling a bucketed string AI attribute, "AI Purchase Propensity," against a fictional 10,000,000-contact file (denominator via `{"and": []}`):

| Value | Count | Share |
|---|---|---|
| low | 5,000,000 | 50% |
| **no score (`is_empty`)** | 3,500,000 | 35% |
| medium | 1,200,000 | 12% |
| high | 300,000 | 3% |
| residual (exclude all above) | 0 | — |

Read: the model scores most of this file "low" intent; the 35% with **no score** are genuinely `is_empty` here. Always probe whether the no-score cohort is `is_empty` *or* a real `no <activity>` bucket (e.g. `no purchase` / `no engagement`) — some attributes have one, some the other, some both. What feeds it: behavioral purchase/engagement events (see `describe_account_events`); exact inputs and lookback window live in the Cordial AI model card.


---

## Inlined shared mechanics — read and follow these; do not skip
_(These are the shared references this skill depends on, flattened in so they are never skipped. Deeper traversal patterns, if referenced, are bundled under `reference/`.)_

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
