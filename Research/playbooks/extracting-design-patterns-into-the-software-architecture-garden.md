---
title: "Playbook: Extracting Design Patterns into the Software Architecture Garden"
slug: extracting-design-patterns-into-the-software-architecture-garden
short: "Turn a working system into evidence-backed Garden pattern studies: study the seams, catalog candidate patterns, write template entries for review, then batch the rest — with line-referenced evidence, newcomer-level exposition, and maturity labels."
topics:
  - architecture-garden
  - software-design
  - design-patterns
  - documentation
  - agent-workflows
  - provenance
  - playbook
section_type: Playbook
created: 2026-09-14
updated: 2026-09-14
published_vault: https://parc.yolo.scapegoat.dev/
---

# Playbook: Extracting Design Patterns into the Software Architecture Garden

A repository that works is not the same as a repository whose design is reusable. The Garden workflow turns the second into writing: named patterns, mathematical foundations, evidence pinned to files and lines, and a reader who has never seen the code able to judge the pattern's value. This playbook distills the iteration that extracted sixteen patterns from playback-ui (ticket PUI-019 and the dashboard-ui garden) — what to study, how to catalog, how to write the entries, where each artifact belongs, and which review loop keeps the quality honest.

> [!summary]
> - Study before naming: read the seams (recording machinery, math, pipeline, dispatch points), and collect real failures from tickets and diaries — patterns are proven by invariants and failure modes, not by intent.
> - Catalog first, write second: a table of pattern → statement → mathematical foundation → CS mapping → files → transfer sketch, reviewed before any full entry exists.
> - Write the reader into the contract: new to the pattern, new to the codebase, and roughly new to CS and math — teach the principles as numbered exposition before using them, and open every entry with condensed context before the overview.
> - Two template entries go to review before the batch: the feedback becomes the template for the rest.
> - Placement is a decision: pattern knowledge lives in the Garden, operational how-to lives in the repository, project status lives in reports, and workflow knowledge (this document) lives in the garden playbooks.

## 1. What this workflow is for

Use this workflow when the discovery is architectural knowledge rather than an implementation task: a recording DSL, a trust boundary, a deterministic generation contract, a validation discipline — a structure whose value extends beyond one function. Do not use it because a file is large, a TODO exists, or a refactor would be convenient. A Garden entry must explain a reusable design and the law it protects; if you cannot state the invariant in one sentence, it is not a pattern yet.

The reader contract is fixed before writing starts, and it is demanding: **assume the reader is new to the pattern, new to the codebase, and roughly new to computer science and mathematics.** This is not dumbing down. It is the discipline that keeps entries honest — an explanation that teaches probability streams and partial functions from scratch cannot hide hand-waving behind jargon.

## 2. Phase one — study before naming

The study pass reads the system's seams, not its average file:

- **The recording and dispatch machinery.** How authoring input becomes data (proxies, builders, interpreters) and how behavior is dispatched (the switch tables, the matrices, the validation maps). These files encode the system's real type system.
- **The math.** The random generators and distributions, the geometry, the statistics, the interval arithmetic. Patterns named without reading the math end up describing symptoms.
- **The pipeline.** Store, replay, retention, boundary validation — where data crosses lines and what may cross.
- **The tests and the diaries.** Real failures (from ticket diaries and implementation history) are the only acceptable source for "what goes wrong" sections. A theoretical concern must be labeled as a risk, never presented as a historical failure.

The playback-ui study pass read, among others: the proxy `chain` and its pending-method loop, the mulberry32/Box–Muller sampling, the store's watermark merge, the resolver dispatch and `expectedKind` map, the timeline geometry, and the tokens file — roughly a dozen files before the first pattern was named.

## 3. Phase two — catalog first, write second

Before writing any entry, produce the catalog table: one row per candidate pattern with the statement, the mathematical foundation, the computer-science mapping, the files and APIs, and a transfer sketch. Review the table with the system's owner before prose begins — rows get merged, split, and demoted at this stage at one hundredth the cost of rewriting entries.

The playback-ui catalog organized sixteen patterns into four domains (DSL and compilation; realtime playback; widget design; cross-cutting), each domain carrying the invariants its patterns protect. The catalog itself became the garden's Index of Design Patterns, with planned-but-unwritten rows marked until their entries landed.

## 4. Phase three — the entry template

Each entry follows the Garden anatomy (problem, concrete shape, woven-in, why it works, what goes wrong, reuse, ecosystem guidance) with two additions from this iteration:

1. **A Motivation section**, positioned after the intro, that expands the context and teaches the principles the pattern rests on. Principles are introduced as numbered, named exposition — a term, then its definition, then the smallest concrete example that makes it stick. No analogies: precise systems explained in their own terms.
2. **An intro that leads with condensed context.** The opening paragraphs are a compressed version of the Motivation — what the system is, what problem is in play — *before* presenting the pattern overview and its load-bearing rule. An intro that assumes the reader already knows what this is about has already lost them.

The rest of the template, proven stable across the playback-ui entries:

- Frontmatter mirroring the existing garden entries: aliases, `status`, `type: architecture-garden-design`, `created`/`analyzed`, `repository`/`repository_remote`/`source_commit` pinned to a real commit, `repository_note_url` for the published vault, tags, and `related_notes` wikilinks.
- File naming `NN - Title - Restated Rule.md`, numbered per topic folder, with the title stating the rule (the go-go-parc authn entry is the model).
- A `[!summary]` callout with the 3–5 load-bearing bullets.
- Concrete shape with real code quoted from the repository (line-referenced), simplified pseudocode where the real code is too dense, and one mermaid diagram where the flow carries weight.
- Failure modes from real history — diary entries, actual bug hunts, pinned fixture lessons — each with the file or command that documents it.
- A transfer sketch (when to reuse, when not to) that names applicability and non-applicability.
- A maturity label from the shared vocabulary, with the evidence that justifies it; one strong implementation makes a *candidate ecosystem pattern*, no more.

## 5. Phase four — placement decisions

The iteration produced four kinds of knowledge, and each had a different right home:

| Knowledge | Home | This iteration's example |
|---|---|---|
| Reusable pattern studies | The Garden (`Research/Software Architecture Garden/<scope>/<topic>/`) | `dashboard-ui/dsl/01–06` in playback-vault |
| Operational how-to for one repository | The repository itself (`docs/playbooks/`) | The Tailscale playbook in playback-ui — initially misfiled in the vault, moved on review |
| Project status and history | Project reports and ticket diaries | The PUI-018 report and diaries |
| Workflow knowledge (how to extract patterns) | The garden playbooks | This document |

The test for placement: if the knowledge would survive the repository being rewritten in another language, it is a pattern; if it would survive the *garden* disappearing but not a repository change, it is repo documentation; if it describes how *you* work rather than what the system is, it is a playbook.

## 6. Phase five — the review loop

Write **two template entries** first, then stop for review. The two should be the catalog's most central patterns (in playback-ui: the recording proxy and the trusted fold — the two everything else stands on). Feedback on those two becomes the template for the batch: the intro restructure in this iteration (context before overview) came from exactly this review, and reordering two entries was cheap; reordering sixteen would not have been.

Then batch the remainder, update the index, and commit per iteration — pushing each reviewed batch rather than accumulating unreviewed prose.

## 7. Checklist before publishing

- [ ] The invariant is stated in one sentence in the intro, as the load-bearing rule.
- [ ] The intro gives condensed context before the overview — a reader who knows neither the system nor the pattern can follow it.
- [ ] Every principle used anywhere in the entry is defined in the Motivation before it is used, with a concrete example and no analogies.
- [ ] Major claims carry file and line references; code quoted is real, from a pinned `source_commit`.
- [ ] Failure modes come from observed history (diaries, fixtures, bug hunts) or are explicitly labeled risks.
- [ ] The transfer sketch names when *not* to reuse the pattern.
- [ ] Frontmatter matches the sibling entries; `source_commit` and `repository_note_url` are real.
- [ ] The index row links the entry; planned rows are still marked.
- [ ] The maturity label matches the evidence vocabulary, not the author's enthusiasm.

## 8. The worked instance

The complete worked instance of this playbook is the dashboard-ui garden in the playback-vault (`https://github.com/manuel-tulip/playback-vault`, `Research/Software Architecture Garden/dashboard-ui/`): the README scoping the study, the Index of Design Patterns carrying the catalog, and six DSL entries (recording proxy, trusted fold, seeded measure, total-by-refusal lowering, executable gap list, distribution-typed authoring) written from the PUI-019 extraction. The intermediate artifact — the intern-facing guide mapping all sixteen patterns to files, APIs, pseudocode, and transfer sketches — lives in the playback-ui ticket workspace (`ttmp/2026/09/14/PUI-019--playback-ui-design-pattern-extraction-and-intern-guide/design-doc/`) and is the recommended companion whenever the catalog is larger than the garden can absorb in one pass.
