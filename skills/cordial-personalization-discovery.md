---
name: personalization-discovery
description: Explore what's possible to personalize or segment on in a Cordial account. Use when a marketer asks what contact attributes exist, what values an attribute actually holds, what data coverage looks like, whether an attribute is wired up/used anywhere, or "what can I personalize on / segment by". Self-learns the account's real attributes and values before estimating coverage, then checks where each attribute is already used in content and data jobs.
---

# Personalization Discovery

Help a marketer understand what data exists in their account and what's realistically viable to personalize or segment on — grounded in the account's **real attributes and real values**, never generic guesses — plus where each attribute is *already* wired up so they don't rebuild it.

> Shared mechanics live in the project-root `_reference/` directory (resolves via `reference/`): the self-learning loop, query grammar, **silent-zero** + completeness-check traps, and **coverage idioms** in `audience-query-mechanics.md`; the **reference-before-build** rule and **suspicious-result law** in `tool-ergonomics.md`; the content-layer map (messages → sculpt templates/blocks → includes → blockData) in `entity-model.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the discovery + usage-scan chain.

## The linked path

Connective keys: **`describe_contact_schema` → attribute `key` + `type`** (type drives the `icfs`/`icfn`/`icfd`/`icfa` wrapper) · **`search_audience_examples`/`get_account_audience_samples` → live operator + value grammar** · **lifted criteria → `estimate_audience` → coverage counts** · **attribute `key` → the Smarty literal `{$contact.<key>}` → content/data-job scan**.

1. **Orient** — `whoami` to confirm the account. Fire the universal-base control `estimate_audience({"and":[]})` once (file total = denominator + liveness proof) before trusting any later count.
2. **Per attribute the marketer cares about**, run the self-learning loop from the mechanics file: find it in the schema (note its `key`, `type`, `validators`), learn its real values from `search_audience_examples` / `get_account_audience_samples` (or probe empirically with the completeness check if examples don't cover it — common for unused attributes).
3. **Size coverage** with `estimate_audience` per the coverage idioms: populated (prefer the `has_value` operator; cross-check `file_total − is_empty`), missing (`is_empty` — when it works), and distribution across real values (progressive exclusion to a ~0 residual for strings). Watch the array trap: arrays take `icfa has_value`, never `is_empty`.
4. **Check existing usage — is it already wired up?** Scan these sources (primary first):
   - **Reusable content (primary):** `list_sculpt_blocks` / `get_sculpt_block` (id-only), `list_sculpt_templates` / `get_sculpt_template`, `list_html_includes` / `get_html_include` (takes the **key**). Search content for `{$contact.<key>}` and `{include 'content:<key>'}`.
   - **Data automations (primary):** `list_data_automations` (query by attribute prefix AND the friendly label). `get_data_automation` exposes the trigger filter + schedule but **not the inner transform script** — report job-level usage only. **For any content-critical attribute, also read the writer's `status` + `lastRunAt`** — a disabled/stale writer feeding live personalization content flips the viability verdict (confirmed live: a disabled points-balance job behind an active loyalty banner); report freshness alongside coverage.
   - Scan whichever content enumeration shows domain-named keys first — on some accounts the personalization lives in HTML includes rather than sculpt blocks; don't burn budget on the empty surface.
   - **Messages (secondary):** `list_messages` / `get_message` for one-off usage in specific sends — sample high-signal messages only; the corpus is too large to scan exhaustively.
   - **Supplements (secondary):** content sometimes sources from a supplement instead of a contact attribute (`$utils->getSupplementRecords`); `get_account_supplements` / `list_supplements` to note when the real source is a table.
5. **Report viability** — per attribute: type, real values observed, coverage counts + rough %, **existing usage** (block/template/include/data-job names) vs "data exists but nothing uses it yet," and a plain-language verdict on whether it's worth personalizing on (and whether there's already a block to reuse).

## Output

A per-attribute viability read in chat: type, observed values, coverage (populated/missing/distribution, reconciled to the file total), where it's already used, and a verdict. State that counts are estimates. No branding needed; render a branded one-pager per `reference/cordial-brand.md` only if the marketer asks for a shareable artifact.

## Guardrails

- **Read-only** — never imply an audience, block, or attribute was created, saved, or changed. Counts are estimates.
- **Never invent a value** (e.g. `loyalty_status = "Member"`) — it silently returns 0 and misleads. Learn values from the account (reference-before-build).
- **Silent zero:** fire the `{"and":[]}` control first; a 0 on a populated-looking attribute means re-check wrapper/operator/value, not "no data."
- **Usage scan has two blind spots** (per `entity-model.md`): data-job inner transform scripts are not exposed (job-level signal only), and per-message `blockData` dynamic include selections are invisible to a body grep — report both as explicit unidentified slices, never as "not used."
- `crdl_ai_*` attributes are Cordial-AI-generated and present in ~every account — great candidates; for profiling an existing audience by them, hand off to **audience-ai-breakdown**. AI values are the model's verdict under its own definition/lookback, not literal fact.
- **Don't over-test** — sample the attributes the marketer asked about, not the whole schema. If they asked "what CAN I personalize on" open-endedly, profile a meaningful sample (high-coverage + obviously-interesting attrs) and say it's a sample.
- Apply the **suspicious-result law** before reporting anything that looks off.

## Honest tool limits (what the probe found)

- **`search_audience_examples` is semantic, not exhaustive** — it surfaces values only where someone already built an audience on them. Absence of examples ≠ attribute unused; fall back to empirical probing.
- **`is_empty` can be non-functional on strings** on some accounts (returns 0 for everything) — derive coverage from `has_value` or enumerated `in [...]` matches + arithmetic residual (full pattern in `audience-query-mechanics.md`).
- **No distinct-values endpoint** — string distributions come from progressive exclusion (N calls); cap effort on high-cardinality free-form fields and report a deterministic "other" remainder.
- **Data-job transform scripts and per-message blockData are API-unexposed** — the two confirmed-unidentifiable slices of any usage scan.

## Worked example (illustrative — fictional values)

A marketer asks: *"Do we have anything on loyalty tier I could personalize with, and is anyone using it?"*

`whoami` → control `{"and":[]}` ≈ 10,000,000 (live) → `describe_contact_schema` finds `loyalty_current_tier` (string) → `search_audience_examples` lifts real values (`gold`/`silver`/`member`) and the live operator → coverage: populated ≈ 6,200,000 (62%), `is_empty` ≈ 3,800,000, per-value slices reconcile to the populated count via a ~0 exclusion residual → usage scan: one sculpt block renders `{$contact.loyalty_current_tier}` in a header greeting; a nightly data automation named for the loyalty sync writes it (inner script not exposed — job-level signal); no include or recent batch message references it. **Verdict: viable — 62% coverage with three clean values; a reusable block already exists, so personalization is a content decision, not a data build.** All values fictional, for shape only.


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
