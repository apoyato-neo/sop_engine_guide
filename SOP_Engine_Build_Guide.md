---
title: How to Build the SOP Generation Engine — Build Guide & Delivery Plan
audience: Daniel Brassloff, Aimee Poyato (+ their AI build agent)
companion: SOP_Engine_AI_Build_Spec.md (machine-actionable spec) · SOP_Engine_Commons_Pilot.md (Commons wiring)
delivered_as: The Commons handoff packet — task `sop-generation-engine` (WS-B pilot)
author: Prepared for Lu Woods
date: 2026-07-11
status: Draft for team review
---

# How to Build the SOP Generation Engine

> This is **build guidance**, not a review of the v2 tool. It tells you how to build a robust, flexible SOP generation system that produces documents *born* both human-ready and machine-ready, feeding Neo Brain through a governed pipeline — and it folds the *how* and the *when* together, because they're inseparable. A companion **AI Build Spec** carries the machine-actionable contracts for your coding agent; a **Commons Pilot** note wires this build into The Commons as its first handoff packet.

---

## 0. Packet header (Commons WS-B format)

| Field | Value |
|---|---|
| **Task ID** | `sop-generation-engine` |
| **Title** | Production SOP generation engine — human- and machine-ready, Neo-Brain-integrated |
| **Delegated to** | Daniel Brassloff, Aimee Poyato + their AI (Claude Code / Cursor) |
| **Domain owner / reviewer** | Daniel (tool) · David Schneider (Neo Brain interop) · Lu (sponsor) |
| **Delegated by** | Lu Woods |
| **Tier** | 1 (read-only / internal tooling) — no money movement, no PHI |
| **Autonomy** | Build + PR; additive changes; review before merge (per David's repo rule) |
| **Comes back to** | Reviewer as a PR into the team org |
| **Status** | `not-started` → `checked-out` → `in-progress` → `in-review` → `done` |

---

## 1. Mission

Build the system that lets a non-technical operator author a Standard Operating Procedure once — answering a guided form — and get, from that single action: (a) a clean human document, and (b) a machine record that flows through a governed pipeline into Neo Brain, the knowledge base our CSRs and AI agents actually query. Facts we already hold in e123/QCI are pulled, not re-typed; every fact carries its provenance; and an SOP update propagates to the knowledge base as a controlled event. The result kills the two problems the current world has: inconsistent hand-written SOPs, and silent drift between what an SOP says and what the knowledge base serves.

The v2 tool Daniel and Aimee built is a strong base to build *from* — deterministic, spreadsheet-driven, self-contained, well-tested. This guide is about the next altitude, not a critique of that work.

## 2. The one idea to hold onto

**One authoring action → two co-designed readers → one governed pipeline.**

An SOP has two readers, and today only one is designed for:
- a **human** (ops staff, CSRs), and
- **Neo Brain's ingestion pipeline**, which chunks the SOP by section, embeds it, attributes its facts, and serves it per-product to CSRs and AI agents.

We confirmed the second reader is real: Neo Brain generates each product wiki from source documents, and the live Allstate wiki cites `Allstate Customer Service SOP.md` as one of its sources. So an SOP is *already* a machine-ingested document — it's just not built for that job yet. Everything below follows from designing for both readers at once.

## 3. The schema — build on a faceted field model (not the 5-type taxonomy)

This is the most important build decision, so it comes first.

The v2 rules engine types every field as one of five things — Constant, Standard Default, Variable, Conditional, Matrix. That's a good *vocabulary*, but it's structurally single-axis where the real world is multi-axis, and the tool has already hit the wall: orthogonal attributes (`SELECT:`, `SHOW_IF:`, `MATRIX_COLS:`, `ALLOW_NA`) are overflowing into the free-text Notes column and being regex-parsed back out. When a schema forces you to encode structure as prose, it's telling you it's missing dimensions.

The five "types" are secretly **three different questions** about a field:
- *Where does the value come from / who owns it?* → Constant, Standard Default, Variable (and a missing fourth: **Derived**).
- *What shape is the data?* → Matrix is really just "table"; everything else is scalar.
- *When do we ask for it?* → Conditional is a **display rule**, not a value type.

So model each field with **independent facets** instead of one `type`:

| Facet | Values | What it drives |
|---|---|---|
| **`source`** | `constant` · `standard_default` · `variable` · **`derived`** | Drift/governance, and whether to **pull from e123/QCI** instead of asking |
| **`datatype`** | `text` · `select` · `multiselect` · `boolean` · `number` · `date` · `table` · `reference` | The input control + the machine record shape |
| **`show_if`** | predicate (optional, on any field) | When to ask — orthogonal to everything else |
| **`applicability`** | `required` · `optional` · `allow_na` · `repeatable` | Completeness enforcement |
| **`governance`** | `owner`, `last_verified` (on defaults); `provenance` (on derived) | The anti-drift controls, as structured data not prose |
| **`machine`** | `key` (stable slug/anchor), `wiki_section` (Neo Brain target), `audience`/`guardrail` tags | Chunking, attribution, and guardrail alignment |

**Before → after, on two real fields:**

- *Duplicate enrollment handling* — v2: `type: Conditional`, with `SHOW_IF: planType=ERISA…` buried in Notes. Faceted: `source: standard_default`, `datatype: text`, `show_if: plan_type == "ERISA discount program"`, `owner: <ops>`, `last_verified: <date>`, `wiki_section: "CSR Best Practices"`. The conditionality is now a first-class rule, and the default carries an owner.
- *Cancellation Decision Matrix* — v2: `type: Matrix`, columns in Notes. Faceted: `source: variable`, `datatype: table` with typed `columns: [Scenario, Inactive Reason, Inactive Date Logic, Notes]`, `wiki_section: "CSR Best Practices"`. Now a table can *also* be conditional or derived if it needs to be.
- *Eligibility (age/SSN/DOB), No-sale states, Add-on rules* — v2: not modeled (or hand-typed). Faceted: `source: derived`, `provenance: {system: e123, field: "Rule: Member Minimum Age", via: "e123 API → QCI → CSV"}`. **The tool pulls these; the author confirms.**

Why this is the right foundation, not just tidier: it makes the three hard goals — machine record, e123 pull, provenance/section-mapping — *expressible at the schema level* instead of bolted on afterward. Everything downstream in this guide depends on it.

## 4. The output contract — make every SOP machine-ready

Design the output as two artifacts from one `buildOutline()` object (the tool already builds that object internally — make it an export target, don't throw it away):

- **Human outputs** — Markdown (Notion) and Word `.doc`, as today.
- **Machine record** — the same content with four properties Neo Brain needs:
  1. **Stable `##` section anchors**, named to align with Neo Brain's wiki-section taxonomy, because those headers are literally how Neo Brain's chunker splits the document (~800-token chunks, one per `##`). Bad headers = bad retrieval.
  2. **Inline `[Source: <SOP title>, <section>]` attribution** on generated facts — adopt Neo Brain's own convention so SOP-origin facts are as traceable as facts from an e-sig PDF once ingested.
  3. **A structured JSON sidecar** — header metadata, matrices as arrays, contacts/effective-dates as typed fields — so structured facts can seed Neo Brain's *structured tables*, not only its narrative chunks.
  4. **Per-product split** — Neo Brain's `/api/query` is strictly single-`product_id`. An SOP spanning multiple products (e.g., "GigCare, PSM, TDK") must split into per-product records at the ingestion boundary; the `Product` field drives the split.

## 5. Pull, don't ask — leverage what we already know

Eligibility rules, no-sale states, add-on/bundle rules, and effective-date rules already exist in e123/QCI, and Neo Brain already seeds them from the e123 products CSV. Anything the tool asks a human to hand-type that e123 already holds is both redundant and a drift source. Wire the `derived` fields to the standing **config-source hierarchy: e123 APIs → QCI tables → e123 CSV export** — pull, pre-fill, and have the author *confirm* rather than type. This is the single biggest "leverage our systems" win and it directly reduces drift.

## 6. Governance — the part that keeps it true over time

The v2 tool has no git repo, no central SOP store, and SOPs live "wherever the user saves the download." Neo Brain already has the governance model to borrow: a `wiki_pending_updates` approval flow, versioned prompts, CODEOWNERS-style ownership, and a branch→PR→review→approve workflow (David's stated rule for the `neo-brain` repo).

Build these in:
- **Version the rules engine + scripts + tests in the team org, with CI** that rebuilds the tool and runs the checks on every PR. Cheapest large risk-reduction available.
- **A governed SOP store** (not local downloads): generated SOPs land versioned, with a "supersedes" link, and updates route through Neo Brain's `wiki_pending_updates` approval flow — so an SOP edit is a controlled event the knowledge base re-ingests, not a silent re-save.
- **Owner + `last_verified` on every Standard Default.** The defaults encode live policy (72-hour invalid-SSN handling, 60-day reinstatement, demographic-change rules). The vault's "Children over 26" case — where an SOP's stated refund step was *not* current practice — is exactly the failure a deterministic generator scales if its defaults go stale. Make ownership and verification structural (§3), not a good intention.

## 7. Hosting — deploy it; keep an offline mode

Stop distributing a file. Deploy the tool into **`neo-tools-portal`** (our role-gated internal launcher — already hosts the KM assistant, ASG dashboard, ProductQA), which gives auth, a central SOP store, discoverability, and a clean handoff path to Neo Brain — no new infra. Keep the self-contained offline HTML as an export/fallback; it's genuinely useful.

## 8. The Neo Brain interop contract (summary)

The machine record must satisfy Neo Brain's ingestion expectations: `##`-anchored sections that chunk cleanly; inline `[Source:]` attribution; a structured sidecar for the structured tables; one record per `product_id`; and updates delivered through `wiki_pending_updates` as additive PRs. The **full, machine-actionable version of this contract is in the AI Build Spec** — hand that to your coding agent.

Boundary to respect: **Neo Brain owns product knowledge; the Commons does not duplicate it.** The *generated SOPs* are Neo Brain source material. The *build guidance and reusable patterns* (this guide, the faceted-schema pattern, the interop contract) are Commons knowledge. Don't publish SOP content into the Commons — it's proper-noun-heavy and belongs in Neo Brain.

## 9. How & when — the folded plan

Weeks are relative (W1 = kickoff). Each phase ships value on its own; sequencing puts the cheap high-leverage wins first.

**Phase 0 — Align & baseline (W1).** Team review of this packet; agree the P0/P1 cut. Put the rules engine + scripts + tests into the team org with CI (mirrors David's branch→PR→approve). *Exit: versioned, CI-gated, no functional change yet.*

**Phase 1 — Faceted schema + machine-ready output (W2–W3).** Migrate the schema to the faceted model (§3). Emit stable `##` anchors, inline `[Source:]`, the JSON sidecar, and per-product split (§4). Surface required-field enforcement; version-stamp the localStorage draft cache. *Exit: a generated SOP round-trips cleanly through Neo Brain's chunker; the sidecar validates.*

**Phase 2 — Pull from e123/QCI (W3–W5).** Wire `derived` fields to the config-source hierarchy (§5); author confirms pre-filled facts; flag mismatches against e123. Assign owner + `last_verified` to every Standard Default; first policy-truth review pass. *Exit: ≥90% of e123-held facts auto-fill and reconcile against Neo Brain's structured tables.*

**Phase 3 — Governed store + hosting (W5–W7).** Deploy into `neo-tools-portal`; central versioned SOP store with "supersedes"; offline HTML retained as export. *Exit: ops author in the portal; every SOP versioned + discoverable.*

**Phase 4 — Neo Brain bridge (W7–W10).** Build the pipeline from the governed store into Neo Brain: narrative → wiki chunks, structured facts → structured tables, provenance preserved, updates via `wiki_pending_updates` (additive PRs). *Exit: an SOP edit propagates to a pending Neo Brain wiki update automatically — drift loop closed.*

**Phase 5 — Completeness, assist & rollout (W10–W13).** Add remaining product-reference sections; align the SOP `Audience` field with Neo Brain guardrails/use-cases; add an optional LLM "draft-assist" that pre-fills Variables from source docs for human confirmation (the rendering path stays deterministic). Org-wide rollout; instrument adoption + time-to-first-draft. *Exit: every new launch runs through the tool; metrics live.*

**Milestones:** M0 versioned+CI (W1) · M1 machine-ready (W3) · M2 e123 pull (W5) · M3 hosted+governed (W7) · M4 Neo Brain bridge (W10) · M5 rollout (W13).

**Success metrics:** 100% of new launches via the tool; median SOP-update→re-ingestion < 1 business day; zero stale-default incidents; every SOP chunks cleanly + carries attribution; ≥90% e123 facts auto-pulled.

**Key risks:** stale Standard Defaults (highest — mitigated by §3/§6); e123 access latency (sequence so Phase 1 ships regardless); two-system ownership across Daniel/Aimee (tool) and David (Neo Brain) — engage David early on the Phase 4 contract; scope creep into generation (keep rendering deterministic; assist is opt-in pre-fill only).

## 10. How this becomes The Commons's first packet

This build *is* the Commons pilot (details in the companion Commons note). In short: this document is the first **handoff packet** (WS-B, which ships first per D-006), delivered into the `commons` repo `/tasks/sop-generation-engine/`. The reusable pieces — the Neo Brain interop contract, the faceted-schema pattern, the e123 config-source hierarchy — become the first **knowledge slices** published to `/knowledge` (WS-A). Work happens in the repo (branch `task/sop-generation-engine`, small commits), so progress is visible through the WS-C completeness view without status meetings. You check out by moving Status → `checked-out`; you return by opening a PR to the reviewer.

## 11. Parking lot — future build, not critical path

- Cross-product / suite-level retrieval (today Neo Brain is single-`product_id`; would need a fan-out layer).
- Formulary + provider data as structured sources (Neo Brain's own roadmap).
- Legacy-SOP backfill into the governed store + Neo Brain as products cycle through hypercare.
- Process/"Other" SOPs → a lighter machine contract (they don't map to product wikis; likely feed an internal-process knowledge area, not a product).
- Multi-language / accessibility passes on generated output.
- SOP ↔ ProductQA cross-check (validate a generated SOP's stated rules against live product config as a test).
- Retire the SheetJS CDN dependency once hosted (bundle the parser locally).

## 12. Rules to honor (guardrails)

- **Rendering path stays deterministic.** No LLM in the document-generation path; the optional assist is pre-fill-for-confirmation only.
- **Additive PRs into Neo Brain**, never removing core functionality (David's rule). Review before merge.
- **No PII, no secrets.** SOP content holds contacts/phone lines (fine) but never member PII; never paste secret values into code, chat, or the repo — point at where the credential lives.
- **Single source of truth per fact.** Derived facts point at e123/QCI; don't restate them as an independent copy that can drift.

## 13. Sources & resource index (named + "for more, see…")

- **Rules engine** — `SOP_Rules_Engine.xlsx` (4 sheets); `sop_builder.html` (`buildOutline`, renderers, conditional logic, matrix editor, SheetJS upload); `AI_System_Blueprint_SOP_Builder_Tool.html`.
- **Neo Brain** (`Neo-Insurance-Solutions/neo-brain`) — for more, see: `src/lib/wiki/templates.ts` (13-section taxonomy + generation prompt + inline `[Source:]` rule), `.../wiki/generator.ts` (source-docs→wiki), `.../ingestion/wiki-chunker.ts` (`##` chunking), `.../ingestion/structured-seeder.ts` (e123 CSV → structured tables), `.../app/api/query/route.ts` (single-`product_id` RAG), `.../prompts/exclusions.ts` + `system-prompts.ts` (guardrails + per-use-case prompts), `docs/neo-brain-portal-agent-guide.md` + `-human-guide.md` (the dual-audience model), `neo-knowledge-brain-implementation-plan.md`, `wikis/allstate/wiki-39475.md` (worked example citing an SOP as a source).
- **Commons** — `the_commons/_decision_log.md` (D-001 pointer-not-copy, D-003 team-org home, D-004 system-reviews, D-005 completeness-from-signals, D-006 B→A→C→D), `_reference/handoff_packet_template_v1.md`.
- **Vault** — products `_cowork_instructions.md` (config-source hierarchy e123→QCI→CSV); `_ops/briefing_corrections.md` (2026-05-28 "Children over 26" stale-default example).
- **AI companion** — `SOP_Engine_AI_Build_Spec.md`. **Commons wiring** — `SOP_Engine_Commons_Pilot.md`.
