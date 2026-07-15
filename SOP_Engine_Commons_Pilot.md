# The Commons — Pilot via the SOP Engine

**Purpose:** use the SOP-engine build as The Commons's first real payload, proving WS-B (handoff packets) and seeding WS-A (knowledge substrate). This note maps the build onto the Commons's locked decisions so we stand the Commons up by *using* it, not by designing it in the abstract.

## Why this is the right pilot
- **D-006** ships **handoff packets (WS-B) first**, and its success criterion is "the first real delegated handoff uses a *manufactured* packet, not a hand-assembled bundle." The SOP build guide *is* that packet (`task: sop-generation-engine`).
- The reusable, proper-noun-light artifacts it produces — the **Neo Brain interop contract**, the **faceted-schema pattern**, the **e123 config-source hierarchy** — are exactly the kind of curated **knowledge slices** WS-A exists to publish.

## The boundary that keeps it clean (important)
The Commons **does not duplicate Neo Brain** (charter). So:
- **Into the Commons `/knowledge`:** the *build patterns* (how-to), which are generalizable and low-proper-noun.
- **Into Neo Brain (not the Commons):** the *generated SOP content* (carrier/product SOPs), which is product knowledge and proper-noun-heavy — it would (correctly) trip the Commons PII/proper-noun publish gate.
Same rule as the vault↔Neo Brain hard boundary; state it once, honor it everywhere.

## Repo shape (D-003: home = team GitHub org)
`commons` repo in `Neo-Insurance-Solutions`:
```
/templates   ← handoff_packet_template_v1.md (seed from the vault draft)
/tasks/sop-generation-engine/
     packet.md          ← the human Build Guide, as the packet
     ai-spec.md         ← the AI Build Spec
     status.md          ← not-started → checked-out → in-progress → in-review → done
/knowledge
     INDEX.md           ← resource catalog (D-007)
     neo-brain-interop-contract.md   ← pointer-slice → neo-brain repo + our contract notes
     faceted-field-schema.md         ← reusable rules-engine pattern
     config-source-hierarchy.md      ← pointer → products _cowork_instructions.md
```

## Check-in / monitor / check-out — mapped to the decisions
- **Check-in / publish (WS-A, D-001 one-way + pointer-not-copy):** knowledge slices are published via a `commons_manifest.md` (default-deny), each stamped with source-hash + `last_updated`, through the secret/PII gate. The SOP knowledge slices are the first entries. *First pass inlines-and-flags* (see caveat below), then converts to pointers.
- **Review (D-004 the system reviews, not a person):** automated gates (secret/PII/format/pointer-validity/contradiction) auto-handle the mechanical majority; CODEOWNERS routes each `/knowledge` area to its domain owner; reversibility over perfect gating. The SOP packet's PR is the first exercise of this.
- **Monitor / visualize (WS-C, D-005 completeness from work signals):** a team-facing view in `neo-tools-portal` derives task/project/workstream completeness from repo events (commits/PRs/checkouts) + Commons contributions — not activity-tracking. The SOP build's phase/contract/ingestion status is the **first instance of this view** (not a separate dashboard).
- **Check-out (the packet is the unit):** a delegate moves `status.md` → `checked-out`, branches `task/sop-generation-engine`, commits small (commits = progress), opens a PR to return. Their **AI** consumes `ai-spec.md`; the **human** reads `packet.md`.

## Three caveats to name up front (so they don't read as bugs)
1. **Minimal WS-D comes first for the pilot.** B/A/C all need the repo to *exist*. Stand up a bare `commons` repo (`/templates` + `/tasks`) now; mature the review governance later — consistent with "repository is foundational, governance matures last."
2. **The first packet is inline-heavy, not link-heavy.** "Link, don't inline" needs WS-A slices that don't exist yet, so this packet inlines + flags-to-publish. That's the intended bootstrap: this packet *creates* the first WS-A slices.
3. **The completeness view is WS-C piloted, not a new surface.** Build it as the first WS-C instance in the portal, fed by repo signals.

## Repo topology & containment (how it stays self-contained)

Two repos in `Neo-Insurance-Solutions`, distinct jobs:
- **`commons`** — the Commons concern *only*: `/templates`, `/tasks` (packets), `/knowledge` (pointer-slices + `INDEX.md`), governance policy (CODEOWNERS, gate workflows, `commons_manifest.md`). Point the team here to **pick up work**.
- **`sop-engine`** — the tool codebase; where Daniel & Aimee **build**. Governed by the Commons, not merged into it. (No repo exists yet — created at WS-D-min.)

Four mechanisms keep it from bleeding across the org's git:
1. **One home for Commons artifacts** — knowledge/tasks/templates live in `commons` and nowhere else.
2. **Pointer-not-copy (D-001) is the anti-bleed control** — `commons` never mirrors product code/knowledge; it publishes pointers (path + source-hash + `last_updated`). A pointer is three lines; a copy is the bleed. This keeps `commons` a small, legible index.
3. **Product code in its own repos, governed *by reference*** — shared reusable CI workflows (the secret/PII/format/test gate defined once, *called* by each repo) + org-level rulesets (branch protection, required checks) + CODEOWNERS. Policy defined once, adopted by a one-line reference — not copy-pasted.
4. **Work never relocates; only signals aggregate** — WS-C reads org events (commits/PRs/checkouts) read-only into the portal. A product repo's Commons footprint is ~nil (a branch name + at most one shared-workflow call).

## What gets handed off into git, and how

**Into `commons` (PR):**
- `/templates/handoff_packet_template_v1.md` — seeded at repo init (from the vault draft).
- `/tasks/sop-generation-engine/` — `packet.md` (= the human Build Guide), `ai-spec.md` (= the AI Build Spec), `status.md` (the lifecycle state).
- `/knowledge/` — `INDEX.md` + three slices: `neo-brain-interop-contract.md`, `faceted-field-schema.md`, `config-source-hierarchy.md`. Published via WS-A (manifest → gate → hash-stamp → push); first pass inlines-and-flags, then converts to pointers.

**Into `sop-engine` (Daniel/Aimee build, PR):**
- The rules engine (serialized) + `convert_rules_to_json` + build scripts + `sop_builder` source + `sop_schema.json` + the acceptance suite + CI workflow. Branch `task/sop-generation-engine`; small commits = the progress signal.

**The clean split:** the *guidance docs* → `commons` (the how-to). The *tool code* → `sop-engine` (the build). The *generated SOP content* → Neo Brain (never the Commons).

**How (order):** (1) org admin creates both repos (WS-D-min); (2) seed `commons/templates`; (3) PR the packet + AI spec into `commons/tasks/`; (4) publish the three knowledge slices through the gate (its first live test); (5) team builds the tool in `sop-engine`, PRs back for review.

## Commons audit process (yes — mostly automated, scheduled, loud-on-RED)

This is the Commons instance of the **Layer-2 audit + sync-truth/loud-failure** discipline your framework already runs — not something to invent from scratch. It runs as CI on the `commons` repo plus a scheduled check; RED → team DM. It verifies:
1. **Gate integrity** — every published slice passed the secret/PII/proper-noun gate; a seeded canary leak is still blocked (WS-A's own success criterion).
2. **Drift** — every pointer-slice's source-hash matches its live source; loud on mismatch (the drift-check, confirmed firing).
3. **Containment / no-bleed** — no Commons artifacts living outside `commons`; no product code copied in; slices are pointers, not restated copies.
4. **Index completeness (D-007)** — every resource is named, in `INDEX.md`, and carries a "for more, see…" pointer — no dead-ends.
5. **Ownership routing** — CODEOWNERS covers every `/knowledge` area; no orphaned areas.
6. **Signal fidelity** — the completeness view reconciles with actual repo state (spot-check), so the dashboard can't quietly lie.
7. **Reversibility** — every published change is versioned and revertable.

The SOP pilot is the audit's first subject: does the packet publish clean, does the knowledge slice pass the gate, does the drift-check fire when the guidance source changes? Ownership sits in **WS-D** (governance); the automated checks are policy-as-code (D-004), the cadence is a scheduled task.

## Suggested first steps
1. WS-D-min: create the `commons` **and** `sop-engine` repos; seed `commons/templates` with the packet template.
2. Drop this build's `packet.md` + `ai-spec.md` into `commons/tasks/sop-generation-engine/`.
3. Publish the three knowledge slices to `commons/knowledge/` with an `INDEX.md`; run them through the secret/PII gate as the gate's first live test.
4. Stand up the WS-C completeness view in `neo-tools-portal` reading these repos' events.
5. Wire the Commons audit (CI + scheduled drift/gate/containment check); run it first against the SOP pilot.
