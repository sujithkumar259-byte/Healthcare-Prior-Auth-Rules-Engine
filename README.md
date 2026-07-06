# Coral Prior-Authorization Rules Engine

A category-agnostic, versioned, refreshable prior-authorization rules engine. **Rules are
data, not code.** The engine evaluates an already-extracted patient case against the payer
policy in force on the service date and returns a verdict with the specific gap.

- **No runtime LLM.** The shipped engine makes no model/API calls and depends on no LLM SDK.
  Model reasoning happens only at *build time* (an agent reading policy text and authoring
  criteria packs), which is committed, diff-able data that a human reviews before activation.
- **Structured-first.** Coding/eligibility rules (HCPCS, ICD-10, modifiers) are ingested
  automatically from CMS structured endpoints. Only narrative clinical thresholds (e.g.
  AHI ≥ 15) are hand-authored from the policy text.
- **Traceable.** Every criterion carries a `source` (doc id + section + url). No criterion
  without one.
- **Versioned & effective-dated.** Updates are append-only; evaluation resolves the version
  in force on the case's service date.
- **Human-gated.** Nothing ingested or authored goes live without review-queue approval.

Golden example: **Medicare DME PAP (CPAP/BiPAP)** — NCD 240.4 + LCD L33718.

**Coverage today: 940 policy packs, 5,284 sourced criteria** across three payers:
- **Medicare (63):** all 58 DME-MAC LCDs + 6 outpatient procedure-PA LCDs (blepharoplasty,
  botulinum toxin, varicose veins, spinal cord stimulators, facet joint interventions, lumbar
  spinal fusion).
- **Cigna commercial (155):** the entire public medical-coverage-policy library, ingested from
  the policy PDFs on static.cigna.com (search → fetch → PDF text extract → author).
- **Aetna commercial (722):** the Clinical Policy Bulletin (CPB) library, fetched from the
  public CPB pages and authored the same way.

Every pack ships `pending_review` — nothing is resolvable until a clinician approves it through
the review queue. Run `npm run coverage` for the live breakdown. Commercial packs prove the
engine is payer-agnostic: a policy is just a `PolicyVersion` with `key.payer` set and a
`PAYER_POLICY` source. (Panniculectomy, rhinoplasty, and cervical fusion are PA-required but have
no standalone LCD to author from, so they're intentionally omitted rather than fabricated. Part D
drugs and Medicare Advantage set their own PA, which CMS does not centrally publish.)

## Quick start

Requires Node 20+ (developed on Node 24). Then:

```bash
npm install
npm run demo        # evaluate the PAP appendix case (submit_ready, needs_info, abstain, no_policy)
npm run coverage    # how many categories/criteria are loaded, by payer
npm test            # full acceptance suite (no network)
npm run typecheck   # tsc --noEmit
npm run explorer      # build explorer.html — a browsable UI over every loaded pack
npm run build-netlify # bundle a static deploy: the explorer + a PA-eval API function
```

## Live demo

The engine ships two deployable front-ends, produced by `npm run build-netlify` into `deploy/`:

- **Explorer** (`explorer.html` / `deploy/index.html`) — a self-contained page to browse and
  search every pack and criterion, each linked to its source document.
- **PA-eval API** (`deploy/netlify/functions/evaluate.js`) — `POST` a case, get a `Verdict`
  back. All packs are bundled into the function, so it runs with no database:

  ```bash
  curl -X POST <site>/.netlify/functions/evaluate \
    -H "Content-Type: application/json" \
    -d '{"category":"aetna_abdominal_aortic_aneurysm_screening","coverage":{"payer":"aetna"},
         "evidence":{"member.sex":{"value":"male"},"member.age_years":{"value":67},
         "condition.prior_aaa_screening_normal":{"value":false}}}'
  ```

  Drag `deploy/` to [app.netlify.com/drop](https://app.netlify.com/drop) to publish. (The demo
  function evaluates packs directly; the review gate is enforced in the engine, not the demo.)

Scripts that hit the live CMS API (no key required):

```bash
npm run ingest -- L33718              # 8a: structured code tables -> pending_review criteria
npm run ingest -- L33797 home_oxygen  # explicit category; --json for the full version
npm run fetch-lcd -- L33797 --sections # 8b: print LCD narrative section names + sizes
npm run fetch-lcd -- L33797 --save     # save narrative text to .lcd-cache/ for authoring
npm run sync                          # pull catalog, ingest changed LCDs -> review queue
```

> The CMS Coverage API occasionally returns transient `fetch failed` errors under load;
> just re-run. Data endpoints use a license-agreement bearer token the client fetches and
> caches automatically (~1h TTL).

## Architecture

```
  CMS structured tables ──▶ ingest-structured (8a, no LLM) ─┐
  (HCPCS, ICD-10, modifiers)                                 ├─▶ Policy Registry ◀─ Review Queue
  CMS narrative text ──▶ agent authors src/packs/*.ts (8b) ─┘    (versioned,          (human
                          → committed as pending_review            effective-dated)    approval)
                                                                        │
  Case (already extracted) ─▶ store.resolve(service_date) ─▶ evaluate ─▶ Verdict
                                          engine core (safe DSL interpreter + abstention)
```

### Files

| Path | Role |
|---|---|
| `src/types.ts` | Domain model (`Check`, `Criterion`, `PolicyVersion`, `Case`, `Verdict`) |
| `src/dsl.ts` | Safe recursive `Check` interpreter — **no `eval`, no code from data** |
| `src/engine.ts` | `evaluateCriterion` / `evaluateCase` + abstention logic |
| `src/store.ts` | `PolicyStore` interface, `InMemoryPolicyStore`, `ReviewQueue` |
| `src/cms-client.ts` | License token, catalog, code tables, narrative text |
| `src/ingest-structured.ts` | 8a: code tables → deterministic criteria (no LLM) |
| `src/refresh.ts` | `sync()`: detect change by LCD version → pending_review → review |
| `src/seed-catalog.ts` / `seed-catalog.json` | Doc→category mapping (the one curated lookup) |
| `src/packs/data/<category>.json` | **Every** policy pack as data — all 940 packs ship `pending_review` |
| `src/packs/data/_manifest.json` | Per-policy metadata (payer, doc id/url, effective date, items, status) |
| `src/packs/load-data-packs.ts` | Loads + validates the JSON packs into PolicyVersions (content-hashed version ids) |
| `src/dsl-validate.ts` | Structural validator — malformed generated checks fail loudly |
| `src/packs/_pack-helpers.ts` | `buildPack` / source builders |
| `src/packs/registry.ts` | Gathers all packs (data-only); `loadActivePacks` / `loadPendingPacks` |
| `scripts/*.ts` | `demo`, `ingest`, `fetch-lcd`, `sync`, `coverage`, `explorer`, `build-netlify`, `demo-update` |
| `scripts/refresh_commercial.py` | re-fetch tracked commercial policies, checksum-diff, stage changed ones |

### Keeping policies current (updates)

Policies are versioned and effective-dated, so an update is a **data swap, not a code change**:
a new/changed policy enters as a new `PolicyVersion` (its `version_id` carries a content hash,
so it appends alongside the old one — history preserved), sits `pending_review`, and on
approval the prior version is auto-retired at the new effective date. `resolve()` always
returns the version in force on the case's service date. See `npm run demo-update` for a live
demonstration (simulates Cigna tightening a bariatric BMI threshold).

To detect updates at scale:
- **Medicare:** `npm run sync` (CMS API version-number diff -> re-ingest -> pending_review).
- **Commercial:** `npm run refresh-commercial` — re-fetches every tracked Cigna/Aetna policy,
  hashes its normalized text, and stages only the changed ones (the *check* is free/no-LLM;
  re-authoring the changed list is the only token cost). `--limit N` / `--payer X` to scope.
| `test/*.ts` | Zero-dependency acceptance suite |

## The Check DSL

A criterion's logic is a serializable object. The interpreter (`evalCheck`) only ever reads
values and compares them with fixed operators — it never executes data.

```ts
{ op: "present" | "absent" | "nonEmpty", path }
{ op: "cmp", path, cmp: ">="|"<="|">"|"<"|"=="|"!=", value }   // strict, no type coercion
{ op: "in", path, values }                                      // "path[*]" => ANY element in set
{ op: "ratio", num, den, cmp, value, minDen? }                  // false if den 0 / non-numeric
{ op: "all" | "any", of: [...] }   |   { op: "not", of: {...} }
```

**Path resolution:** a path resolves first against `case.evidence[path].value`, otherwise as
a dotted path into the case object (`order.items`, `order.prescriber_npi`). A trailing `[*]`
marks an array for membership (the "any dx in covered set" helper). Prototype-chain segments
(`__proto__`, `constructor`, `prototype`) are blocked.

## Engine behavior

`evaluateCriterion(case, criterion, tau=0.85)`:
1. `present` = every `requires` path resolves to a non-null value.
2. `conf` = min confidence over `requires` paths (1 where unspecified).
3. not present → `missing`; present but `conf < tau` → `uncertain` (abstain); else
   `evalCheck` → `satisfied` / `unmet`.

`evaluateCase`: no version → `no_policy`; evaluate criteria whose phase is unset or equals
`case.phase`; verdict from **hard** criteria — any `missing|unmet` → `needs_info`; else any
`uncertain` → `abstain`; else `submit_ready`. Soft criteria surface gaps but never change the
verdict.

## Authoring narrative packs (8b)

The boundary against fabrication: an agent reads the LCD's *Coverage Indications* section
(`npm run fetch-lcd -- <LCD> --save`), writes each numeric/boolean threshold as a `Criterion`
in `src/packs/<category>.ts`, and **commits the pack as `pending_review`**. A human (Sujith)
diffs it against the source and flips it to `active` via the review queue.

- Every criterion carries `source.section` + `url`.
- Requirements that **cannot** be expressed deterministically (clinical judgement, narrative
  thresholds, field-vs-field comparisons the DSL can't express) are recorded as `[NEEDS_HUMAN]`
  notes in the pack (`<category>NeedsHuman`) and **left out of the active checks** — never guessed.

**Every** pack is a JSON data file in `src/packs/data/` (`<category>.json`) + a manifest row,
loaded and DSL-validated by `load-data-packs.ts` — including the verified golden PAP pack and
the first six hand-authored Medicare packs, which were migrated from TypeScript so the format
is fully uniform. A pack may carry a per-criterion `source` override (e.g. PAP cites both NCD
240.4 and LCD L33718). Adding a category = drop a JSON file + a manifest row; no code change.
Every pack ships `pending_review` — nothing is active until a clinician approves it.

To extend the set: `npm run fetch-lcd -- <LCD> --save` to read the narrative, then author the
JSON pack following `.lcd-cache/AUTHORING_GUIDE.md` (the rules the authoring agents follow).

## Acceptance criteria → tests

| # | Criterion | Where |
|---|---|---|
| 1 | PAP appendix → submit_ready; flip improved → needs_info; conf 0.6 → abstain | `npm run demo`, `test/engine.test.ts` |
| 2 | `resolve` honors effective/retirement dates; unmatched → no_policy | `test/store.test.ts` |
| 3 | DSL handles every op; hostile string treated as data | `test/dsl.test.ts` |
| 4 | `ingest -- L33718` → ICD-10 `in` + HCPCS criteria, sourced, no LLM | `npm run ingest -- L33718`, `test/ingest.test.ts` |
| 5 | pending_review never resolved; only `approve` activates + retires prior | `test/store.test.ts` |
| 6 | every criterion has `source.doc_id` + `url` | `test/ingest.test.ts`, `test/packs.test.ts` |
| 7 | authored packs are pending_review, sourced; no LLM import / model call | `test/packs.test.ts` |

## Extending

- **New DME category:** add a row to `seed-catalog.json`, `npm run ingest -- <LCD>` for the
  code tables, then author `src/packs/data/<category>.json` from the narrative and add its
  manifest row (the registry is data-only — no code to touch). No engine-code change.
- **New payer:** add a connector (fetch + parse) following `cms-client.ts`; author packs the
  same way. `PolicyStore` has a Postgres seam (it's an interface; only `InMemoryPolicyStore`
  ships in v1).
