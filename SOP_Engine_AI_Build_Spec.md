# SOP Generation Engine — AI Build Spec

**Audience:** the coding agent building this (Claude Code / Cursor). **Companion to:** `SOP_Engine_Build_Guide.md` (human rationale). This file is the machine-actionable contract: schemas, formats, acceptance criteria, constraints. Where the two differ, the human guide holds the *why*; this holds the *shape*.

**Hard constraints (read first):**
- The document-**rendering** path is deterministic. No LLM in the outline→Markdown/Word/JSON render. (An optional pre-fill "assist" may draft Variable values for human confirmation, but it never renders the final doc.)
- Changes to the `neo-brain` repo are **additive** and go via branch → PR → review → approve. Never remove core Neo Brain functionality.
- Never write member PII or secret values into code, output, chat, or the repo. Carrier contacts/phone lines are allowed; secrets are pointed-to, never pasted.
- Single source of truth per fact: `derived` fields reference e123/QCI; never persist a restated copy that can drift.

---

## 1. Field schema — faceted model (replaces the single `type`)

```typescript
interface SopField {
  key: string;                 // stable slug; also the machine anchor id. Immutable once shipped.
  section: string;             // human section (e.g. "Cancellations")
  wiki_section: WikiSection;   // Neo Brain target section (one of the 13; see §5)
  label: string;               // display label
  prompt?: string;             // author-facing question

  source: "constant" | "standard_default" | "variable" | "derived";
  datatype: "text" | "select" | "multiselect" | "boolean" | "number" | "date" | "table" | "reference";

  options?: string[];          // for select | multiselect
  columns?: { key: string; label: string; datatype: SopField["datatype"] }[]; // for table

  show_if?: string;            // predicate over other fields' keys, e.g. `plan_type == "ERISA discount program"`
  applicability: { required: boolean; allow_na?: boolean; repeatable?: boolean };

  default?: unknown;           // for constant | standard_default
  governance?: {               // REQUIRED when source === "standard_default" | "constant"
    owner: string;             // who owns this policy value
    last_verified: string;     // YYYY-MM-DD
  };
  provenance?: {               // REQUIRED when source === "derived"
    system: "e123" | "qci";
    field: string;             // source field name, e.g. "Rule: Member Minimum Age"
    via: "e123_api" | "qci" | "e123_csv"; // resolution tier actually used
    fetched_at: string;        // YYYY-MM-DD
  };
  audience?: ("internal" | "shareable")[];
  guardrail?: string[];        // maps to Neo Brain per-app section exclusions
}
```

Migration from v2: `Constant→source:constant`; `Standard Default→source:standard_default`; `Variable→source:variable`; `Matrix→datatype:table`; `Conditional→` drop the type, set `show_if`. Parse the legacy `SELECT:`/`SHOW_IF:`/`MATRIX_COLS:`/`ALLOW_NA` Notes flags into the structured facets, then retire the Notes-flag convention.

## 2. Rules-engine source format

Keep the xlsx as the human-editable schema, but the **build artifact** is a validated JSON (`sop_schema.json`) conforming to §1, produced by `convert_rules_to_json`. CI fails the build if any field violates the schema (e.g. `standard_default` without `governance`, `derived` without `provenance`, non-unique `key`, `show_if` referencing an unknown key).

## 3. Output contract

For each SOP the tool emits three artifacts from one internal outline object:

**(a) Human Markdown / Word** — as today, section grouping optimized for reading.

**(b) Machine Markdown** — same content, with:
- Every section a `## <Stable Section Name>` header. Section names come from `wiki_section` (align to §5). Chunk boundary = `##`, so no content may live above the first `##` except the product header block.
- Inline attribution on each generated fact: `[Source: <SOP title>, <section>]`. Match Neo Brain's marker format so `validateAttributions()` accepts it.
- Product header block matching Neo Brain's `renderWikiHeader` shape: `# <displayLabel> [<productId>]` + suite/category/status lines.

**(c) Structured JSON sidecar** (`sop-<product_id>.json`):
```json
{
  "product_id": 39475,
  "sop_title": "Allstate Enhanced STM PPO: CS & Support Operating Standards",
  "audience": "internal",
  "version": "2026-07-11T...",
  "supersedes": "<prior version id | null>",
  "header": { "...": "typed header fields" },
  "sections": [
    { "key": "...", "wiki_section": "CSR Best Practices", "field": "...",
      "value": "... | [] table rows", "source": "standard_default|variable|derived",
      "provenance": { "...": "for derived" } }
  ],
  "matrices": { "cancellation_decision": [ { "Scenario": "...", "...": "..." } ] }
}
```

**Per-product split:** if `header.Product` names N products, emit N machine records (Markdown + sidecar), one per `product_id`. The human doc may remain combined; the machine records must not.

## 4. e123 / QCI pull (derived fields)

Resolve `derived` fields through the config-source hierarchy **e123 APIs → QCI tables → e123 CSV export** (pull live first; QCI for what e123 can't give directly; CSV only as last resort). Minimum derived set (mirror Neo Brain's `structured-seeder.ts` so the two reconcile):

| Field | e123 source field | Neo Brain table it must reconcile with |
|---|---|---|
| Member/spouse/child min–max age | `Rule: Member/Spouse/Child Minimum/Maximum Age` | `product_eligibility` |
| SSN / DOB / gender required | `Rule: * Required` | `product_eligibility` |
| No-sale states | `No Sale States` | `product_state_availability` |
| Add-on / bundle rules | `Product Rules - Must have ALL`, `Auto Assign - No Delete` | `product_addon_rules` |
| Effective-date display rules | `Rule: Effective Date Display`, `Active Date Rule` | (narrative) |

Author confirms pulled values; on mismatch vs. a prior SOP or vs. Neo Brain, surface a diff, don't silently overwrite.

## 5. Neo Brain ingestion contract

- **Section taxonomy** (`WikiSection`): `Coverage Overview`, `Eligibility`, `State Availability`, `Billing and Effective Date Rules`, `Plan Benefits and Features`, `Enrollment Information`, `Bundled Products and Add-Ons`, `Claims and Authorization`, `Key Terms and Definitions`, `Frequently Asked Questions`, `Must-Add-On Products`, `CSR Best Practices`, `Miscellaneous`. Map every SOP section to one of these; the servicing sections (Cancellations/Refunds/Reinstatements/etc.) map to `CSR Best Practices`.
- **Chunking:** Neo Brain splits by `##` at ~800 tokens with a `[Product/Section/Suite]` context header. Keep sections coherent and not enormous; a section far over 800 tokens will be sub-split at paragraph boundaries, so use paragraph breaks deliberately.
- **Retrieval:** `POST /api/query` is single-`product_id`, SSE, `x-api-key` auth, hybrid vector+fulltext + structured lookup, per-`use_case` prompt (`cs_rep`/`agent`/`dtc`/`custom`), guardrail section-exclusions. Structured facts (eligibility/state/addon) are answered from structured tables, not narrative — which is why the sidecar (§3c) matters.
- **Update path:** deliver SOP updates via the `wiki_pending_updates` flow (ingest → supersede → admin approval), as additive PRs. Preserve `[Source:]` provenance through ingestion.

## 6. Acceptance criteria (extend the existing 22-check suite)

- [ ] Schema validation: every field conforms to §1; `standard_default`⇒`governance`, `derived`⇒`provenance`, unique `key`, resolvable `show_if`.
- [ ] Machine Markdown: no content above first `##`; every section header ∈ §5 taxonomy; every generated fact has a well-formed `[Source:]` marker (`validateAttributions()` passes).
- [ ] Sidecar validates against the §3c JSON schema; matrices present as arrays.
- [ ] Per-product split: an SOP with `Product = "A, B"` emits two machine records with correct `product_id`s.
- [ ] Chunk round-trip: feeding the machine Markdown to Neo Brain's `wiki-chunker` yields ≥1 chunk per section with correct `section_heading`.
- [ ] Derived pull: a fixture product's eligibility/no-sale/addon values match `structured-seeder` output for the same e123 row.
- [ ] Determinism: same inputs ⇒ byte-identical Markdown + sidecar.
- [ ] localStorage draft cache is schema-version-stamped and migrates/discards on mismatch.

## 7. Repo layout & workflow

- Home: team org (`Neo-Insurance-Solutions`). Product tool repo governed by the Commons `commons` repo.
- Branch: `task/sop-generation-engine` (+ sub-branches). Small commits = the progress signal (Commons WS-C).
- CI on every PR: rebuild tool + full acceptance suite (§6). Block merge on any failure.
- Deploy target: `neo-tools-portal`; retain a self-contained offline HTML export.
- Return: PR to reviewer; passes secret/PII/format/test gates.

## 8. Resource index (for more, see…)
- Faceted schema rationale + phasing → `SOP_Engine_Build_Guide.md`.
- Neo Brain code of record → `Neo-Insurance-Solutions/neo-brain` (`templates.ts`, `wiki-chunker.ts`, `structured-seeder.ts`, `api/query/route.ts`, `docs/neo-brain-portal-agent-guide.md`).
- Config-source hierarchy → products `_cowork_instructions.md`; QCI schema → `_knowledge/qci/`.
- If under-resourced: the Commons resource index (`/knowledge/INDEX.md`), the domain owner (Daniel/David), or the source system (e123/QCI). Never dead-end.
