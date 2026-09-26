---
title: "CV and Algorithm Inventory — Base Material for a Human Activity Recognition System"
aliases: [HAR base material, CV inventory, activity recognition starting point]
tags: [article, factory-shots, computer-vision, perception, activity-recognition, sites, state-machine]
status: active
type: project-report
created: 2026-09-17
updated: 2026-09-17
audience: machine-learning engineer, new to the system
---

# CV and Algorithm Inventory — Base Material for a Human Activity Recognition System

## 0. What this document is

This is a hand-off inventory for an ML engineer starting a **human activity
recognition (HAR)** effort on top of our video and operations stack. It
covers three things: (1) every CV/algorithm component that already exists in
the factory-videos corpus pipeline (`gt`), (2) the Sites **state machines**
in the playback ops-calling worktree, which are the downstream consumers
vision output eventually feeds and a source of operational labels, and (3)
the vault articles that document each piece in depth.

Read the `[[wikilinks]]` inside Obsidian at `~/code/tulip/playback-vault`;
every one is a full article. File paths below are exact.

| System | Repo / location |
|---|---|
| Factory-videos corpus pipeline (`gt` CLI) | `~/code/tulip/experiments/2026-09-14--factory-videos` |
| Sites / ops-calling app (state machines) | `~/code/tulip/playback/.worktrees/ops-calling-2026-watabe` |
| Knowledge base (deep-dive articles) | `~/code/tulip/playback-vault` |

## 1. The data substrate

The corpus is factory floor video (warehouse aisles, forklifts, people,
pallets, racking, loading docks). Every video has a **shot index**
(schema-v2 JSON): TransNet V2 shot segmentation, shots with start/end times
and clean spans, plus a `predictions.npz` cache of per-frame TransNet
single/many-head scores.

Derived data lives in one SQLite store per corpus
(`cache/index.db`, single writer, WAL): `model_runs` (every computation
with full provenance), `perception_attributes` (per-shot candidate
evidence: detections, tracks, prompt scores, VLM candidates), embeddings
and clusters, phash, and the human annotation layer (verified labels kept
**separate** from model candidates).

There is a shared **evidence frame cache**: deterministic w512 JPEGs keyed
by source content hash, subject (`video:shot`), and timestamp
(`t%.2f.jpg`-style). All models read the same bytes — this is what makes
model comparisons reproducible. Dense (5 fps) and sparse (8 frames/shot)
sampling share the cache.

## 2. Existing CV pipeline components

| Stage | Algorithm / model | Output artifact | Status |
|---|---|---|---|
| Shot segmentation | TransNet V2 (single + many heads) | `shots-v2` index, `predictions.npz` | production; postprocessing repaired + DGX-migrated |
| Frame extraction | ffmpeg batched decode (exact seeks, w512 cap) | shared evidence cache | production |
| Perceptual hash | pHash on representative frames | similarity search (`gt similar`) | production |
| Image embeddings | DINOv3 (ViT-S/B/L, 384/768/1024-d, incl. ungated timm mirrors) | per-shot pooled vectors, run-family identity | production |
| Text-aligned embeddings | SigLIP2 (base-p16-224, so400m-p16-384, so400m-naflex) | zero-shot prompt scoring per shot (`gt prompts score`) | production |
| Joint video-text embeddings | NVIDIA Cosmos-Embed1 (224p/336p/448p, 1152-d, bfloat16 compute) | clip-native shot vectors | experimental branch (FACTORY-COSMOS-001) |
| Clustering | over embedding runs (`gt cluster build`) | cluster assignments per shot | production |
| Semantic search | exact cosine (`gt embed search`) | ranked shots | production |
| Detection (fixed vocab) | YOLO26 (Ultralytics) | `detections` candidate attributes | implemented |
| Detection (open vocab) | YOLOE (`yoloe-11s-seg.pt`, **qualified default**), YOLO-World | prompted detections; class vocabulary is identity-bearing | production; YOLO-World has documented vocabulary sensitivity |
| Dense-window tracking | greedy IoU tracker, per-window stable ids, coasting (gap 2), fps 5 default | `tracks` attributes: motion summaries + full observations | implemented + DGX-qualified (bounded) |
| Motion/change summary | ffmpeg lowres frame-diff + TransNet scores | per-window `frame_diff`, `change_score` | production |
| Window features | bounded semantic window features: fixed crops, detector presence, SigLIP2 prompts per window | feature vectors for classifiers | production (recent main) |
| VLM candidate annotation | Qwen3-VL (e.g. `Qwen/Qwen3-VL-4B-Instruct`), mock-vlm for tests; nvidia/Cosmos-Reason experimental | structured candidate annotations + validation | implemented |
| Review / verification | gt-web UI, profile-bound runs, MCP adapter | human-verified labels (separate tables) | production |

Key commands to explore: `gt videos`, `gt shots`, `gt embed index/search`,
`gt attributes index/track`, `gt motion`, `gt prompts score`, `gt vlm
annotate`. The `dgxp` orchestrator runs stage DAGs on the DGX under a
single-writer lock.

## 3. What exists specifically for activity recognition

Two layers of temporal material are already built:

1. **Per-window/shot features**: SigLIP2 prompt scores, detector
   presence/counts, DINOv3 embeddings, motion summaries, fixed crops.
   [[ARTICLE - Chat-Driven Video Classification - Semantic Features and Held-Out Evaluation]]
   documents the window-feature classifier path with a held-out protocol.
2. **Instance-level temporal evidence (new)**: dense-window tracks with
   per-track motion summaries — path length vs. net displacement, speed,
   direction, gaps — plus the full (timestamp, box) observation list per
   track. See [[ARTICLE - Dense-Window Tracking - Stable Track Ids, Motion Summaries, and a Detector-Flicker Diagnosis]].
   Limits: ids are per-window (no cross-shot re-identification), 46%
   single-observation tracks at default confidence (detector flicker),
   and no ground-truth trajectories yet.

A prior baseline effort — [[ARTICLE - Video Activity Classification - A Layered Baseline for Activity, Procedure, and Process Analysis]] —
is the most important read for HAR. It separates **activity / procedure /
process** levels, builds a linear baseline on frozen descriptors and a
custom TCN (not MS-TCN++) with careful padding/objective treatment,
defines procedure checking ("observed activity is not verified
completion"), duration modeling, streaming availability semantics, and
proposes concrete production monitors. Reuse its concepts and its honest
evaluation discipline rather than reinventing them.

Related: [[PROJ - Vision Classifier Builder - Building the Prototype - Synthetic Archive, Active Learning, GLM Agent, and Timeline Deployment]]
(active-learning classifier workflow) and the NVR segment-browsing/export
article for raw-camera time ranges.

## 4. The Sites state machines (ops-calling worktree)

The playback platform models a physical site as a graph of **stations**.
Ground truth is `shared/stations/machine/definition.ts` in the worktree
(schema is Zod, `.strict()`); full articles:
[[ARTICLE - Playback Sites - A Complete Walkthrough of the State Machine Model]] and
[[ARTICLE - Playback Sites - State Machine Semantics and Foundations for Classification and Statistics]].

The model, in one pass:

- A **station** is a typed attribute environment: a bounded set of typed
  attributes with per-attribute writer ownership inferred from the project.
- A station optionally declares a **state machine** layer:
  - **States** are *predicates*, not enum values: each state has a `when`
    expression (a strict, validated subset of TypeScript) evaluated over
    the station's attributes. State resolution is exactly predicate
    classification — "exactly one true" does not prove the rest false, and
    "unavailable" and "stateless" are distinct outcomes.
  - **Transitions** are declared moves between states (`from` may be `'*'`).
  - **Actions** are named proposals: a `guard` expression, a list of
    **assignments** (controlled attribute writes, literal or expression),
    `allowedTransitions`, and `allowedProducerKinds` — the producer kinds
    include **`vision`** (also app, api-user, api-key, mqtt, automation,
    system-timer). A vision pipeline can therefore be a first-class
    producer of observations and actions.
  - **Projections** are read-side allowlists (the resolved state plus
    listed attributes).
- Evaluation is **event-driven** — there is no scheduler; the only
  scheduled loop is camera input ingestion. Conditions may be temporal,
  evaluated without a background clock.
- Durability and history: definitions are compiled and stored; station
  observation lifecycle/archiving and attribute transitions/alerts live in
  Postgres migrations (`db/migrations/*station*`). The historical timeline
  (frozen reads, deterministic clocks) is documented in
  [[ARTICLE - Playback Sites - The Historical Timeline - Frozen Reads, Deterministic Clocks, and Honest Time]].

**Why this matters for HAR**: these machines define the operational
vocabulary that a HAR system should predict into (state/transition/action
labels over stations), and they are also a **label source** — resolved
states and fired transitions are recorded ground truth you can train and
evaluate against. The semantic-foundations article works through exactly
what is and is not safe to use as classification input (e.g., state
regions are not proved to partition the input domain).

## 5. Synthetic data generators

For pretraining, augmentation, or controlled evaluation before touching
real footage:

- **VirtualHome TEC**: a verifiable factory video simulator — layout
  manifests → loadable Unity environments, an embedded JavaScript scenario
  language, live RTSP cameras from the Unity player, arm64 cross-build.
  See [[PROJ - VirtualHome TEC - Building a Verifiable Factory Video Simulator]],
  [[PROJ - VirtualHome TEC - From Layout Manifest to a Loadable Factory Environment]],
  [[PROJ - VirtualHome TEC - An Embedded JavaScript Scenario Language for the Player]], and
  [[PROJ - VirtualHome TEC - Live RTSP Cameras from the Unity Player]] (2026/09/10–11).
- **Kimodo text-to-motion** on the DGX Spark:
  [[PROJ - Kimodo Text-to-Motion on the DGX Spark - Motion Diffusion, the aarch64 Port, and Demo Execution]].

## 6. Constraints to respect when building on this

1. **Candidate vs. verified evidence.** Model outputs (detections, tracks,
   VLM annotations) are candidates with provenance; only the human review
   path creates verified labels. An HAR model trained on candidates
   inherits their noise — decide explicitly which label source you train on.
2. **Provenance and run identity.** Every knob (model, vocabulary,
   confidence, fps, tracker config) is hashed into run ids. New model
   outputs must record their own run identity; never overwrite an
   existing run's evidence. Single SQLite writer; use the driver `flock`.
3. **Licensing.** Ultralytics YOLO weights and package are **AGPL-3.0**
   (internal research use recorded in run metadata); productization needs
   a licensing decision. HF-gated DINOv3 checkpoints have ungated timm
   mirrors.
4. **Honest gaps**: no pose estimation; no cross-window track ids; no
   ground-truth trajectories; ID-switch rates on real footage unmeasured
   (overlay review pending); YOLO-World vocabulary sensitivity documented
   — use YOLOE as the default prompted detector.

## 7. Reading list (in order)

1. [[ARTICLE - Video Activity Classification - A Layered Baseline for Activity, Procedure, and Process Analysis]]
2. [[ARTICLE - Playback Sites - State Machine Semantics and Foundations for Classification and Statistics]]
3. [[ARTICLE - Playback Sites - A Complete Walkthrough of the State Machine Model]]
4. [[ARTICLE - Dense-Window Tracking - Stable Track Ids, Motion Summaries, and a Detector-Flicker Diagnosis]]
5. [[ARTICLE - Open-Vocabulary Video Detection - Provenance, Vocabulary Sensitivity, and Controlled YOLO-World Experiments]]
6. [[ARTICLE - Chat-Driven Video Classification - Semantic Features and Held-Out Evaluation]]
7. [[PROJECT REPORT - Factory Video Shot Segmentation - Postprocessing Repair, DGX Spark Migration, and Low-FPS Analysis]]
8. [[ARTICLE - Orchestrating the Perception Plane - Stage Registries, Immutable Job Records, and a Shared-Host Supervisor]]
9. [[PROJ - Factory Videos - Profile-Bound Annotation and Multi-Incident Evidence]]

## 8. Suggested starting points

- **Feature substrate is ready**: window features + tracks + motion
  summaries give you per-shot and per-object temporal descriptors today.
- **First HAR target**: station state/transition prediction over the Sites
  vocabulary, using the recorded machine history as labels — it is the one
  place real ground truth already exists.
- **Evaluate like the baseline article**: held-out splits, honest
  availability semantics, procedure-vs-activity separation.
- **Synthetic-first iteration**: VirtualHome TEC for controlled scenarios
  where real labels are scarce.
