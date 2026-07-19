# Backend design review — 2026-07-18

Findings about **ophyd-service** surfaced while modelling the three backend services into a
reflow2 design graph (Requirements → Capabilities → Components → Interfaces → Verifications).

Scope caveat: the graph's Components were **inferred from source**, not stated independently and
then compared. Design-vs-implementation drift is therefore invisible in this pass by construction
— if the code does something, the graph agrees with it. These findings come from contradictions
*within* the code and docs, from CI, and from contracts that turned out to have no counterpart —
not from a design/build diff.

Repo state: `main` fast-forwarded to `upstream/main` (24 commits) before review.

---

## 1. Circular runtime dependency: configuration ↔ direct_control — **critical**

The two services depend on each other over HTTP at runtime. `backend/README.md` describes three
co-equal services; no architecture doc names this cycle.

- **`direct_control → configuration`** — `RegistryClient` resolves devices/PVs.
  Unconditional. (`direct_control/registry_client.py`)
- **`configuration → direct_control`** — `DirectControlClient.enrich()` POSTs
  `/api/v1/devices/enrich`. Conditional: gated on `CONFIG_DIRECT_CONTROL_URL`, fires only on
  `needs_enrichment` outcomes from first-pass class introspection.
  (`configuration_service/direct_control_client.py:87`, wired `main.py:489`, documented
  `main.py:2571-2575`)

**Secondary concern — a boundary bent, not broken.** The configuration service is designed never
to open Channel Access; it imports ophyd for class-level introspection only. That holds literally.
But `main.py:2571-2575` states the enrichment pass has direct-control *instantiate the device
against the live IOC* and read back the leaf PV. So a registry resolution request synchronously
causes CA traffic on the registry's behalf. The letter of the constraint is intact; the spirit is
arguably not.

**Worth deciding:** is the enrichment fallback a permanent part of the architecture, or a
stopgap until class-level introspection covers the cases that currently return
`needs_enrichment`? If permanent, the co-equal-services framing in `backend/README.md` should say
so, because a new reader will not expect the registry to call back into a controller.

**Credit where due:** the failure path raises `DirectControlUnavailable`
(`direct_control_client.py:95`) rather than silently degrading — consistent with the
project-wide stance in `n3xtware-mapping.md` that misconfiguration should surface as an error
rather than be masked by defaults.

---

## 2. `queueserver-tests` has been red on `main` since 2026-07-14

Run `29336168124`. Groups 2/6 and 4/6 failed; the other 4 groups, the ipykernel-worker job and
the side-C api-client compat job all passed.

| Test | Error | Assessment |
|---|---|---|
| `tests/http/test_core_api_main.py::test_http_server_manager_kill` | expected `'…ZMQ communication error: timeout occurred'`, got `'…Resource temporarily unavailable'` | **Brittle assertion.** Line 1295 asserts an exact ZMQ error string; ZMQ surfaces EAGAIN vs timeout depending on timing. One-line fix: assert the stable prefix. |
| `tests/manager/test_start_re_manager_cli.py::test_cli_parameters_zmq_server_address_1[both_fail]` | `TimeoutError: RE Manager failed to start` | **Likely environmental** — startup race under 6-way parallel CI. |
| `tests/manager/test_scenarios.py::test_large_datasets_03[plan0-10000-60000]` | `TypeError: argument of type 'NoneType' is not iterable` | **Merits investigation.** A TypeError is a crash, not a timeout — something returned `None` where a container was expected, at 10k/60k scale. Not diagnosed; may be a genuine load-only bug or a timeout cascade. |

Recommended order: fix the brittle assertion first (unblocks `main` cheaply), then chase the
`TypeError`.

---

## 3. Stale service naming: 8 references to a service that no longer exists under that name

`configuration_service` refers throughout to **SVC-001 "Experiment Execution Service"** as "the
single source of truth for plans" and the typical writer of devices via the CRUD API:

`protocols.py:15,36,57` · `loader.py:15,749` · `main.py:8` · `config.py:26` ·
`docs/reference/configuration.md:26`

SVC-001 is now `queueserver_service` (bluesky-queueserver + bluesky-httpserver merged). The
comments predate the merge. As written they read as pointing at an **external** system, which
directly caused this review to model a phantom external component before it was corrected.

A reader with less repo context than a full review — a new contributor, or an agent — will make
the same inference. Worth a rename pass. Note `config.py:34` hardcodes
`service_id: str = "SVC-004"`, so the SVC numbering is load-bearing somewhere; check consumers
before changing identifiers rather than just prose.

---

## 4. WebSocket contracts have no schema and no drift gate

Seven WebSocket endpoints carry live telemetry to the frontend:

- `direct_control`: `/api/v1/pv-socket`, `/device-socket`, `/camera-socket`, `/tiff-socket`
  (`main.py:1554-1578`)
- `queueserver`: `/api/status/ws`, `/api/info/ws`, `/api/console_output/ws`

**None appear in the published OpenAPI artifacts** — `shared-schema/direct_control.openapi.json`
contains zero socket paths. Meanwhile all 125 REST paths across the three contracts are covered by
the OpenAPI drift CI.

They are *not* untested — `integration.yml` has an "Exercise direct_control WebSocket" step, so
the sockets are driven end-to-end. The gap is specifically **contractual**: the frontend builds
against `shared-schema/` artifacts, so a WS envelope change has no machine-readable contract to
break and no drift job to catch it. The asymmetry is worth closing deliberately (AsyncAPI, or
documented envelope schemas with their own drift check) or accepting explicitly.

---

## 5. Coverage gates are asymmetric, and the largest service has none

| Service | Tests | Gate |
|---|---|---|
| `configuration_service` | 398 | `--cov-fail-under=75` |
| `direct_control_service` | 327 | `--cov-fail-under=60` |
| `queueserver_service` | 875 | **none** |

queueserver_service is the largest surface (~75k LOC, 74 OpenAPI paths) and the only one whose
coverage can regress silently. Some of that is reasonable — it is vendored-origin code — but the
first-party additions (`config_service.py`, `config_service_coordinator.py`, the Side-B locking
path) are exactly where regressions would hurt, and are new enough to gate.

---

## 6. Requirements from `n3xtware-mapping.md` with no in-repo evidence

The matrix's `Coverage` column is hand-maintained; `FR-*`/`NFR-*`/`SEC-*` ids appear **only** in
that file and nowhere in any test or source. Mapping each row to actual CI evidence:

**Genuinely verified**
- **NFR-004** Standardization → OpenAPI drift guard
- **FR-014** Software test environment → both integration pods

**Partial — claim is broader than the evidence**
- **NFR-001** Maintainability claims "CI, schema drift checks, peer review"; only lint is
  mechanically attached. Marked `Strong`.

**Not verifiable inside this repo** (not hollow — out of scope for code to prove)
- **SEC-003** Review process → GitHub branch protection, not a repo artifact
- **SEC-004** Change traceability → git history, not a repo artifact

  These two are marked `Strong` on the strength of process, not code. Worth annotating the matrix
  so the distinction between *unproven* and *unprovable here* is explicit — otherwise a future
  reader will keep looking for a test that cannot exist.

**Unresolved conflicts and conditions carried forward**
- **FR-012** is marked `Conflict` — N3XTware targets AAP + GitHub for configuration/deployment UI;
  this repo has its own `frontend/`. The matrix itself says letting both grow in parallel is the
  worst outcome. Still undecided.
- **SEC-001** is `Aligned (conditional)` — satisfied *only* because `github.com/nsls2/ophyd-service`
  is private. Trigger to revisit: any move toward open-sourcing. At that point site-specific JSON
  under `integration/happi/sites/` must move to an access-controlled repo.

---

## Suggested order of work

1. Fix the brittle ZMQ assertion (`test_core_api_main.py:1295`) — unblocks `main`, one line.
2. Investigate `test_large_datasets_03` `TypeError` — the only failure that may be a real bug.
3. Decide the enrichment cycle (§1): permanent architecture, or stopgap to remove.
4. Rename stale SVC-001 references (§3) — cheap, prevents recurring misreadings.
5. Decide the WebSocket contract question (§4) — close the asymmetry or record accepting it.
6. Annotate SEC-003/004 in the matrix as process-verified (§6).
