---
name: contact-360-trace
description: Trace ONE contact end-to-end — confirm the record exists (plus any duplicate), read its consent/subscription state, when it opted out, what suppression/send-governance applies, and which sends/journeys its identity qualified for — to explain a single-contact anomaly. Use when a marketer/CSM asks "what happened to this contact," "why did this person get 4 emails back-to-back / is there a frequency cap," "did my suppression actually fire for this customer," "why can't this contact receive push even though tokens look valid," "when and how did this number opt out," "find this complaining customer and any duplicate record," "is this email still subscribed," or "trace this one contact's history."
---

# Contact 360 Trace

Answer "what happened to this *one* contact?" — confirm the record, read its consent/subscription state, bracket *when* it opted out, check the suppression/governance that applies, and list the sends/journeys its identity qualified for, then assemble a why-did-this-happen verdict for a single-contact anomaly.

> Shared mechanics live in the **project-root** `_reference/` directory (resolves via `reference/` — it sits beside `recipes/`, NOT inside it): query grammar + traps (**silent-zero**, the **universal-base `{"and":[]}`** control, **`channels.<ch>.ss` subscription grammar + dotted↔comma key shapes**, **channel-scoped `unsubscribedDate`/`subscribedDate` band grammar**, `supplements_<KEY> has_value` membership) in `audience-query-mechanics.md`; tool-traversal patterns — **reference-before-build**, the **suspicious-result law**, **cross-channel engagement isn't one metric**, and the existing **Single-contact trace** pattern — in `tool-ergonomics.md`; the bucket/role/handoff-key map in `tool-index.md`; the **object graph** (orchestration → trigger limit/audience → message nodes → child sends; status disagreement across layers) in `entity-model.md`. **Open and read the referenced `reference/` files before building any query — they are required, not optional (do not work from the SKILL body alone).** This recipe only adds the single-contact tracing chain. This deliverable is a **chat verdict (profile + small tables) — no branding needed.**

## The linked path

This recipe spans **Account context → Audiences (single-contact criteria probes) → Messages/Sends/Orchestrations**, and turns on one inversion: **the MCP has no per-contact endpoint (no `get_contact`, no event/send-receipt log), so a contact's existence and state are reached ONLY by `estimate_audience` criteria that resolve to count 1 (or 0) on a tight identity filter.** Connective keys: **`describe_contact_schema` → the identity/state attr key + type** (drives the `icfs`/`icfn`/`icfd`/`icfa` wrapper) · **`list_channel_types` → the live channel keys** (the only valid `<ch>` for `channels.<ch>.ss`, `isNotInvalid:<ch>`, and the `unsubscribedDate` channel) · **`get_account_supplements` → `.Key` → `supplements_<KEY>`** (only if the table is contact-linked) · **`search_audience_examples` → live `ss` + `unsubscribedDate` band grammar** · **`estimate_audience.count` (1 vs 0/>1) → exists / state-clause true / duplicate** · **`list_messages` → messageID/templateID → `list_child_sends`** + **`list_orchestrations` → `get_orchestration`** for the configured cap.

1. **Orient.** `whoami` to confirm the connected account (every downstream count belongs to it).
2. **Fire the liveness control FIRST.** `estimate_audience({"and":[]})` → the full file total. This proves the tool returns live numbers before you trust any single-contact `0`/`1` (silent-zero trap — see Limits).
3. **Learn the keys (reference-before-build).** `describe_contact_schema` → the identity attr key + type (e.g. a hashed-email or primary-id field; pick the one with `unique:true` for a tight locate) and the live suppression-state attr key. `list_channel_types` → the channels that actually exist on THIS account (do not assume email/SMS/push are all present). `get_account_supplements` → any contact-linked suppression `supplements_<KEY>`. `search_audience_examples` → the live `ss` and `unsubscribedDate` band grammar. Discover every key here — bake none into this file.
3a. **Channel gate (only if the prompt names a specific channel — SMS, push, direct mail).** If the question is *about* a channel, check `list_channel_types` for that channel key **before** spending any locate effort. **If the channel is not configured, the channel-native question is dead on arrival** — "can't receive push," "how did this number opt out of SMS," and a channel-scoped `unsubscribedDate`/`ss` for that channel are all unanswerable regardless of whether the contact exists. Short-circuit to: that channel isn't a channel on this account; any signal lives only as a contact opt-in/suppression *attribute* (if one exists), which is attribute-derived, not channel-native; the live channel/provider/system-of-record is elsewhere. Don't burn the rest of the chain trying.
4. **Locate the contact — and first confirm the identifier even has a home.** The identifier the marketer gave (email, phone, loyalty/customer id) must map to a `describe_contact_schema` attribute that can hold it. Three outcomes, not two:
   - **No matching attribute exists** (e.g. a phone or "loyalty ID" prompt on an account whose schema has no phone/loyalty key) → **unlocatable by construction.** Report "cannot locate — no identity attribute on this account holds a `<type>` identifier," list the candidate identity keys (`unique:true` first, then non-unique fallbacks like a customer-number), and ask for one of those. This is **distinct from "not found"** — do not run a count and report 0 as "absent." A best-guess match against a plausible non-unique field is allowed only if labeled as such.
   - **Attribute exists, count = 1** → the record exists.
   - **Attribute exists, count = 0** → absent **ONLY if the liveness control passed** (else re-check the wrapper/operator/grammar, don't report "not found"). Confirm the matched attr is itself live (`has_value` ≈ file scale) so a 0 is a real absence, not a dead attr.
5. **Probe state clause-by-clause (each as its own count-1 estimate, anchored to the same identity clause).**
   - **Subscription / consent:** the intuitive path is `channels.<ch>.ss matches ["s"|"u"]`, **but a positive `ss` match can be silent-zero (exclude-only) on some accounts** — it returns 0 for subscribed AND unsubscribed despite a large live population, and works only inside an `exclude` block. On a silent zero, route consent to the estimable proxy **`isNotInvalid:<ch>` / `isInvalid:<ch>`** (deliverability state) anchored to the identity, and flag it as *not identical* to true `ss` state; also try the alternate `ss` key shape (dotted ↔ comma). Validity proxies reconcile to the file total (valid + invalid ≈ universal base), which is how you confirm they're live.
   - **Opt-out timing:** add the channel-scoped top-level `unsubscribedDate` band (lift the live grammar from `search_audience_examples` — it is **not** a contact attribute and is **not** in `describe_contact_schema`; it carries a `channel`). Narrow the band to bracket *when* they opted out. The **method/source** of opt-out (HOW) is typically **not a queryable field** — state honestly if unavailable; `unsubscribedDate` gives WHEN, not HOW.
   - **Suppression membership:** check the **populated** suppression-state attr (`has_value`) anchored to the identity — confirm which suppression attr is actually live on this account (legacy/empty ones return 0 even when the contact is suppressed). If the account has a contact-linked suppression `supplements_<KEY>`, check `has_value` there too. On accounts where suppression supplements are not contact-linked (e.g. coupon-code tables), the attr is the only per-contact suppression read.
   - **Duplicate / related record:** no dedup endpoint — re-run the locate filter on a **shared identifier** → count > 1 = a second matching record. A `unique:true` identity attr makes true email duplicates unlikely; non-unique ids (customer number, integration id) can collide. Report a *residual possibility*, never a clean dedup verdict.
6. **Expand to the sends/journeys the identity qualified for** (a qualified-for list, NOT a receipt log). `list_messages(type=batch)` and `list_messages(type=automated)` for recent sends; reconcile their audience definition + governance against the suppression/consent clauses from step 5 (would this contact have been in-audience and not suppressed?). `list_child_sends(templateId)` for per-run rows of an automated journey. **Find the relevant journey by TAG, not by an assumed name** — `list_orchestrations` then filter on tags/keywords matching the complaint (e.g. cart/abandon/browse/welcome); there is rarely a journey literally named after the complaint. **Report each journey's `status` (enabled vs disabled): a disabled journey could not have been actively sending, which changes the verdict** — flag it rather than describing its config as if it were live. **If the question involves a frequency cap / send governance, answer the configured-vs-fired split explicitly:** whether a cap *fired for this contact* is unidentifiable (no receipt log — step 7), but whether a cap is *configured* IS retrievable — `get_orchestration` exposes the journey's re-entry trigger limit (`{times, per, value, type}`) and per-step send-suppression (an `exclude` with a message-send-history clause), and `get_message_transport` can carry message-level cap config. Message-level `stats.frequencyCapped` is an **aggregate** counter on the send, not a per-contact signal.
7. **Assemble the verdict.** Identity (count 1 vs >1) + consent state (or the proxy used + caveat) + opt-out timing (+ method if exposed, else stated not-queryable) + suppression status + qualifying-send list + cap-configured-vs-fired → the why-did-this-happen answer, **with an explicit unidentified slice** for anything only a per-contact event/receipt stream would show ("4 emails on these dates," push delivery receipts) that the MCP does not expose.

## Output

A short in-chat verdict (no rendered artifact):

- **Identity** — which record(s) matched, on which identifier, count 1 vs >1 (duplicate flag).
- **Consent / subscription** — per-channel state via `ss`, or the `isNotInvalid:<ch>` proxy + a not-identical caveat if `ss` is silent-zero on this account. Note which channels exist at all.
- **Opt-out** — when (the `unsubscribedDate` band that bracketed it); method/source only if exposed, else stated as not-queryable.
- **Suppression** — on/off, read from the live suppression attr (and contact-linked supplement if present).
- **Sends qualified for** — the messages/journeys whose audience the contact matched, reconciled against suppression/consent.
- **Governance** — whether a frequency cap/cadence is *configured* on the relevant journey/message; whether it *fired for this contact* = unidentified.
- **Verdict + unidentified slice** — the explanation, plus an explicit "what the tools can't show per-contact" line. Counts are estimates.

## Guardrails

- **Read-only.** Never imply you re-subscribed, suppressed, resent, merged, or fixed anything — this recipe *explains* the contact's state, it does not change it.
- **Single-contact existence/state is inferred from `estimate_audience` count 1/0/>1 on a tight filter** — it is not a record fetch. Always run the **universal-base `{"and":[]}`** control first so a `0` is a real "no match," not a silent grammar failure.
- **Suspicious-result law:** a 0 where you expected the contact, a "too clean" count, or a state clause that zeros against a populated file means *find the correct traversal/grammar* (dotted↔comma `ss`, `isNotInvalid` proxy, the right suppression attr), not *report the zero*.
- **No faked timeline.** The MCP exposes no per-contact event/send-receipt log. Do **not** narrate "they got emails on these exact dates" or a push delivery receipt as if read from a stream — report the qualifying-send list from audience-criteria reconciliation and name the per-contact slice as a confirmed-unidentifiable gap.
- **Channel-appropriate consent** (see *cross-channel engagement isn't one metric*): first confirm the channel even exists on this account via `list_channel_types`. If a channel (e.g. SMS or push) is not configured, "can't receive push" / "how did this number opt out of SMS" are **not** answerable via channel subscription state — any signal then lives only as a contact attribute (opt-in/suppression flags), which is attribute-derived, not channel-native; say so.
- **Account-portable / publish-ready.** Resolve the identity attr key, channel keys, `ss`/`unsubscribedDate` grammar, suppression attr, and any `supplements_<KEY>` at runtime. Bake **no** account ids, contact emails/phones/hashes, channel keys, attr names, supplement keys, orchestration/message ids, or live counts into this file.

## Honest tool limits (what the probe found)

The live probe ran clean against the connected test account (file total ~10.5M contacts); the limits below were confirmed against live data. Report only at the level the tools support; never infer hidden internals.

- **No per-contact endpoint.** No `get_contact`, no per-contact event/send timeline, no receipt log. A contact's existence/state is reachable **only** via `estimate_audience` criteria resolving to count 1/0 on a tight identity filter. "This contact got N emails on these dates" and push delivery receipts are a **confirmed-unidentifiable** slice — name it, never fabricate a timeline.
- **`channels.<ch>.ss` positive match can be silent-zero (exclude-only).** On the probed account, `ss = ["s"]` and `ss = ["u"]` BOTH returned 0 despite a multi-hundred-thousand unsubscribed population (via `unsubscribedDate`) and ~9.8M valid contacts — `ss` worked only inside `exclude` blocks. A 0 from a positive `ss` match is **NOT** "no consent." Route consent to `isNotInvalid:<ch>` / `isInvalid:<ch>` (these reconciled to the file total) + the `unsubscribedDate` band, and try the alternate `ss` key shape. (The reference doc lists `ss matches [s|u|n]` as a primary; this account contradicts that — verify per account.)
- **Channels are account-specific; do not assume SMS/push exist.** The probed account had only email + a rest channel configured — no SMS, no push. So "why can't this contact receive push" and "how did this number opt out of SMS" were **not** answerable via channel subscription state; SMS/direct-mail signal, if any, existed only as contact opt-in/suppression attributes. Confirm channels via `list_channel_types` before promising a channel answer.
- **Suppression: use the populated attr, not the assumed one.** On the probed account one suppression-state attr was populated (tens of thousands of contacts) while a similarly named legacy attr returned 0. Read the live/populated one; verify per query, never assume. Suppression **supplements** were all coupon-code tables (contact-linked: no) → not a per-contact suppression source on that account.
- **Frequency cap = configured-vs-fired split.** Whether a cap *fired for a specific contact* is unidentifiable (no receipt log; message-level `stats.frequencyCapped` is an aggregate count). Whether a cap is *configured* IS answerable: `get_orchestration` exposes the re-entry trigger limit (`{times, per, value, type}` — e.g. 1 per 365 days) and per-step send-suppression (`exclude` on a message-send-history clause). Answer the split explicitly.
- **Opt-out method/source is not a queryable field.** `unsubscribedDate` gives WHEN (a working channel-scoped top-level band — it is a SYSTEM path, not in `describe_contact_schema`); it does not give HOW. A signup-source attr may exist but is signup, not opt-out, source. State honestly.
- **The given identifier may have no home attribute at all.** A phone-number or "loyalty ID" prompt on an account whose schema has only a hashed-email unique key, a customer-number, and integration ids is **unlocatable by construction** — there is no field that holds that identifier type to match against. This is a *third* outcome distinct from count-0 "absent": say "cannot locate — no `<type>` identity attribute on this account" and offer the candidate keys (unique first), rather than matching a guess and reporting 0. A `count` against the wrong field is meaningless here.
- **No native dedup endpoint.** Duplicates surface only as count > 1 on a shared identifier; a `unique:true` identity attr makes true email dupes unlikely, but non-unique ids can collide. Report a residual, never a clean dedup. With no located record (unlocatable or count 0), the dedup check can't run at all — report it indeterminate, not "no duplicate."
- **Re-read everything live; trust no remembered/seeded count.** Suppression and consent counts drift (a previously-populated suppression attr can read 0 now); per-account behavior (silent-zero `ss`, which suppression attr is live, which channels exist) is not portable. Re-probe every value against the connected account; never carry a count or attr-liveness fact from another account or a prior run.
- **Silent-zero trap is live.** Fire `{"and":[]}` before trusting any 0. Counts are estimates. `has_value` is the canonical population probe — a wildcard `matches "*"` is not reliable for population. Single MCP (Cordial) — cross-MCP probing N/A.

## Worked example (illustrative — fictional values)

A marketer: *"A customer emailed support saying they got 4 retargeting emails in a row even though they unsubscribed. Their email is jane.doe@example.com. Did our suppression work, is there a frequency cap, and is this person still subscribed?"* (`jane.doe@example.com` is a placeholder — a real run needs a live identifier; if the locate returns count 0 against a passed liveness control, the per-contact state probes in steps 4–5 are unreadable for that address and you report the structural answers only, as below.)

1. `whoami` → confirms the account. 2. Control `{"and":[]}` → full file total (proves liveness; a later 0 is real). 3. **Locate:** `estimate_audience` on the unique identity attr matched to jane's identifier → **count 1 = exists** (0 would mean absent, since the control passed). 4. **Consent:** the `ss` positive match returns 0 here (exclude-only), so anchor identity AND `isInvalid:<ch>` vs `isNotInvalid:<ch>` for deliverability state, AND an `unsubscribedDate after <early date>` band → brackets *when* she unsubscribed (HOW = unidentifiable). 5. **Suppression:** identity AND the populated suppress-state attr `has_value` → is she actually suppressed. 6. **Cap:** `list_orchestrations` → `get_orchestration` on the retargeting journey → read the trigger limit + per-step send-suppression → a cap **is** configured; whether it fired for jane, and "exactly 4 emails on these dates," = confirmed-unidentifiable, named as such. 7. **Duplicate:** re-locate on a shared id → count > 1 = residual (unlikely on a unique email attr).

Verdict shape (all values fictional): *"Record found (1 match, no duplicate on email). Email shows as unsubscribed — opt-out bracketed to ~[month]; opt-out method not exposed by the tools. She is on the suppression list (suppress attr populated). The retargeting journey HAS a frequency cap configured (re-entry limited per the trigger limit) plus per-step send-suppression — but whether it fired for her, and the literal '4 emails on these dates,' is not retrievable (no per-contact receipt log). Recommend confirming exact sends with the platform send log."* — ids/counts omitted by design; for shape only.


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
