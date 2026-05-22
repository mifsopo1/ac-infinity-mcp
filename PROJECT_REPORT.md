# AC Infinity MCP v1.0 — Project Report

**Repository:** [ober37/ac-infinity-mcp](https://github.com/ober37/ac-infinity-mcp)
**Completed:** 2026-05-22
**Built with:** Claude Sonnet 4.6 (personal subscription)

---

## Section 1 — Overview

This project extracted and expanded the AC Infinity MCP server from the private
`ober37/peekaboopoint` monorepo into a standalone, fully-featured, publicly-installable
package. The starting point was five working tools embedded in a private grow-ops
automation repo — useful but undocumented, untested at the module level, and inaccessible
to anyone else.

The ending point is a production-quality MCP server with 18 tools and 3 prompts covering
the complete AC Infinity cloud API surface: device discovery, live and historical sensor
readings, port status and automation settings, write-control for fans and environmental
automation, one-click grow stage templates, environment health scoring and trend analysis,
and three guided prompt flows for growers who are new to MCP. The package is installable
via PyPI (`pip install ac-infinity-mcp`), runnable as a Docker container from GitHub
Container Registry, and documented end-to-end in `docs/API.md` including all 15 confirmed
API quirks discovered through community reverse engineering.

**Disclaimer:** This project was built entirely on a personal Claude subscription.
No Anthropic resources were allocated to it; the time investment figures reflect what a
solo developer would achieve with standard subscription-tier access.

---

## Section 2 — Coverage Summary

### Tools

| Metric | Monorepo Baseline | v1.0 Standalone | Delta |
|---|---|---|---|
| MCP tools | 5 | 18 | +13 (+260%) |
| MCP prompts | 0 | 3 | +3 |
| Source modules | 2 (`client.py`, `server.py`) | 5 (`client`, `server`, `schema`, `controller`, `analytics`) | +3 modules |
| Write-control tools | 0 | 8 | +8 |
| Analytics tools | 0 | 3 | +3 |
| Intelligence tools | 0 | 2 | +2 |
| Production PRs | 0 | 17 feature/fix PRs + 3 Dependabot | 20 total merged |

### Lines of Code

| Metric | Monorepo Baseline | v1.0 Standalone |
|---|---|---|
| Source LOC | ~885 (approx. from git history) | 2,965 |
| Test LOC | 1,552 (recorded in plan file) | 5,389 |
| Test files | 3 | 12 |
| Test directories | 1 | 4 (`common/`, `devices/`, `fixtures/`, `integration/`) |

### Test Suite

| Metric | Value |
|---|---|
| Unit tests collected | 394 |
| Test failures | 0 |
| Coverage (unit suite) | 100% |
| Coverage threshold (CI) | 85% |
| Integration test files | 2 (`test_mcp_protocol.py`, `test_live.py`) |
| Async test framework | `pytest-asyncio` (auto mode) |

### Controller Compatibility

| Controller | Model | devType | Write Support | Notes |
|---|---|---|---|---|
| AC Infinity 69 Pro | WiFi Cloud | 11 | Full | Legacy read-before-write pattern |
| AC Infinity 69 Pro+ | WiFi Cloud | 18 | Full | Legacy read-before-write pattern |
| AC Infinity 89 AI+ | WiFi Cloud | 20+ | Read-only (v1.0) | New framework (`newFrameworkDevice=true`) — write deferred to v2.0 |

---

## Section 3 — Comprehensive Code Review

### Review Structure

Two independent passes ran across the full project lifecycle:

**Pass 1 — Sonnet (per-PR):** Every PR went through a Gate 1 (Senior Python Engineer)
and Gate 2 (Security Engineer) review before merge. Holistic audits also ran at Phase 11
and Phase 15 as standalone review sessions.

**Pass 2 — Independent second-model review:** Phase 11 used Opus 4.7 for a cold
independent pass with no prior context from the Sonnet reviews. Phase 15 used a fresh
cold Sonnet instance (Opus was unavailable due to daily session limit). Both second-model
passes were instructed to treat the code as if seeing it for the first time and to report
all findings independently.

The second-model passes were the most productive finding mechanism across the project —
both HIGH defects caught after Phase 11 (race condition, D005) and both HIGH defects
caught in Phase 15 (VPD rounding, D001; grade boundary, D002) came from the independent
model, not from the per-PR Gate reviews.

### Defect Summary

| Severity | Count | All Resolved? |
|---|---|---|
| HIGH | 7 | Yes |
| MEDIUM | 10 | Yes |
| LOW | 7 | Yes |
| **Total** | **24** | **Yes** |

### Defect Detail

#### Phase 1 — Scaffold & Core Migration

| ID | Discovered | Severity | Issue | Fix |
|---|---|---|---|---|
| Ph1-D001 | Gate 4 mypy | LOW | `Session.timeout` not a valid `requests.Session` attribute — timeout was silently ignored in monorepo code | Removed; each HTTP call already passes `timeout=` explicitly |
| Ph1-D002 | Gate 1 code review | HIGH | Async gap in `get_device_reading()` — blocking `get_devices()` called without `asyncio.to_thread()`. Plan listed 2 async gaps; code had 3. | Wrapped in `asyncio.to_thread()` |
| Ph1-D003 | Gate 3 pip-audit | MEDIUM | `requests` 2.32.5 — CVE-2026-25645 | Pinned to `>=2.33.0` |
| Ph1-D004 | Gate 3 pip-audit | LOW | `pytest` 9.0.2 — CVE-2025-71176 | Pinned to `>=9.0.3` |
| Ph1-D005 | Gate 4 pip install | MEDIUM | `setuptools.backends.legacy:build` not importable in system Python's setuptools 65.x | Changed to `setuptools.build_meta` |

#### Phase 2 — Test Suite Migration

| ID | Discovered | Severity | Issue | Fix |
|---|---|---|---|---|
| Ph2-D006 | Gate 4 pytest | LOW | `"0m"` incorrectly included in invalid-interval parametrize list — regex accepts it and returns 0, so it's syntactically valid | Removed from negative test case |

#### Phase 11 — README Refactor + Deep Quality Cycle

| ID | Discovered | Severity | Issue | Fix |
|---|---|---|---|---|
| Ph11-D001 | Opus Pass 1 | LOW | `VPDTargets` dataclass duplicated `STAGE_TARGETS` values — two sources of truth for the same data | Removed `VPDTargets`; all callers use `STAGE_TARGETS` |
| Ph11-D002 | Sonnet Pass 2 | LOW | `datetime.utcnow()` deprecated — 4 call sites (2 in `server.py`, 2 in `client.py`) | Replaced with `datetime.now(UTC)` |
| Ph11-D003 | Sonnet Pass 3 | MEDIUM | Dockerfile running as root | Added non-root `appuser` in multi-stage build |
| Ph11-D004 | Opus Pass 1 | MEDIUM | Invalid stage in `check_vpd_drift` silently defaulted to `veg` — grower receives VEG targets for a seedling query with no warning | Changed to explicit structured error response |
| Ph11-D005 | Opus Pass 2 | HIGH | Token refresh race condition — N concurrent 401 responses triggered N `authenticate()` calls, each overwriting the token | Added `threading.Lock` around token refresh; only first caller authenticates, rest wait and reuse |
| Ph11-D006 | Opus Pass 2 | MEDIUM | `get_historical_data()` did not detect HTTP 401 — raised `APIError` instead of `AuthenticationError`, preventing retry | Added 401 detection matching `authenticate()` pattern |
| Ph11-D007 | Opus Pass 2 | MEDIUM | `_enforce_write_rate_limit()` not thread-safe under concurrent write calls | Moved `_last_write_time` tracking inside the same lock as the token refresh |
| Ph11-D008 | Gate 5 smoke test | LOW | Wire protocol test missing for Phase 11 invalid-stage behavior change | Added test for structured error response on invalid stage |

#### Phase 13 — Automation Write Tools

| ID | Discovered | Severity | Issue | Fix |
|---|---|---|---|---|
| Ph13-D001 | Planning session | HIGH | `targetVpd` encoding in spec was `int(target_vpd*100)` but Phase 12 reads `/10` — correct encoding is `int(target_vpd*10)` | Corrected encoding; planning session caught this before any code was written |
| Ph13-D002 | Planning session | HIGH | `devLt`/`devHt`/`devLh`/`devHh` encoding in spec was `int(value*100)` but API fixture shows raw integers | Corrected; planning session cross-reference with Phase 12 fixtures caught this |
| Ph13-D003 | Planning session | HIGH | `atType` for VPD mode in spec was `3` (AUTO) but correct value is `8` (VPD) per mode encoding table | Corrected; planning session caught this before any code was written |
| Ph13-D004 | Gate 4 test failure | MEDIUM | `_parse_schedule_time` didn't validate hour/minute range — `"25:00"` parsed as 1500 minutes silently | Added range validation; raises `ValueError` |
| Ph13-D005 | Gate 1 code review | MEDIUM | `_parse_schedule_time` swallowed original exception chain | Fixed with `raise ValueError(...) from None` |

#### Phase 15 — Quality Cycle (Second Pass)

| ID | Discovered | Severity | Issue | Fix |
|---|---|---|---|---|
| Ph15-D001 | Cold Sonnet (independent) | HIGH | VPD banker's rounding: `round(1.25*10)=12` in Python (should be 13) — affected veg stage template and any user-supplied VPD with `.X5` pattern. **Latent since Phase 13.** | Replaced `round()` with `int(value * 10 + 0.5)` for always-round-half-up behavior |
| Ph15-D002 | Cold Sonnet (independent) | HIGH | Grade boundary mismatch: `_grade()` used `score >= 80` for B but `environment_alert_interpretation` prompt documented B=75–89 | Aligned grade boundaries to code (A≥90, B≥80, C≥70, D≥60, F<60) |
| Ph15-D003 | Cold Sonnet (independent) | MEDIUM | `deviation` field documented in `environment_alert_interpretation` prompt but absent from `check_vpd_drift` response | Added `deviation` field to `check_vpd_drift` output |
| Ph15-D004 | Cold Sonnet (independent) | MEDIUM | `check_vpd_drift` alert text gave backwards advice: HIGH VPD said "reduce humidity" (wrong — should raise humidity); LOW VPD said "lower fan speed or increase humidity" (wrong — should lower humidity) | Corrected alert text in both HIGH and LOW branches |
| Ph15-D005 | First code review pass | LOW | `get_port_activity_report` missing from `docs/API.md` MCP Tool Reference section | Added full tool reference section to docs/API.md |

---

## Section 4 — Time Investment Summary

Phases 0–2 and 11–15 have formal Time Investment blocks recorded in the plan file.
Phases 3–10 and 12 are estimated from plan scope and PR scope — marked with *.

| Phase | Deliverable | Claude Time | Projected Manual | Multiplier |
|---|---|---|---|---|
| Phase 0 — Ideation | Plan file + full API research | 0:52 | 26–46h (mid ~36h) | ~41x |
| Phase 1 — Scaffold | Repo, core migration, 5 tools | 0:40 | 12–18h (mid ~15h) | ~22x |
| Phase 2 — Tests | Test suite migration, 129 new tests | ~1:00 | 10–16h (mid ~13h) | ~13x |
| Phase 3 — Analytics Tools | Health score, trend, activity report tools | ~0:45\* | 6–10h (mid ~8h)\* | ~11x\* |
| Phase 4 — Docs & Errors | Typed exceptions, full docs/API.md | ~0:45\* | 6–10h (mid ~8h)\* | ~11x\* |
| Phase 5 — CI/CD | GitHub Actions, CodeQL, Dependabot | ~0:30\* | 4–8h (mid ~6h)\* | ~12x\* |
| Phase 6 — Write Foundation | 77-field payload builder, legacy vs AI+ paths | ~1:15\* | 14–22h (mid ~18h)\* | ~14x\* |
| Phase 7 — Write MCP Layer | set\_port\_speed, set\_port\_on, set\_port\_off | ~0:45\* | 6–10h (mid ~8h)\* | ~11x\* |
| Phase 8 — Write Hardening | Guard rails, AI+ limitation, 403 retry | ~0:45\* | 6–10h (mid ~8h)\* | ~11x\* |
| Phase 9 — Docker | Multi-stage Dockerfile, docker-compose, Claude Desktop | ~0:45\* | 4–8h (mid ~6h)\* | ~8x\* |
| Phase 10 — Integration Tests | MCP wire protocol, main() startup, live tests | ~1:00\* | 8–14h (mid ~11h)\* | ~11x\* |
| Phase 11 — Quality Cycle #1 | README + Opus independent review + concurrency fix | ~6:00 | ~45–60h (mid ~52h)\* | ~9x\* |
| Phase 12 — New Read Tools | get\_port\_status, get\_port\_settings | ~1:00\* | 8–14h (mid ~11h)\* | ~11x\* |
| Phase 13 — Automation Writes | set\_vpd/temp/humidity\_automation, set\_port\_mode | ~1:00 | 12–20h (mid ~16h) | ~16x |
| Phase 14 — Intelligence | apply\_grow\_stage\_template + 3 MCP prompts | ~1:00 | 8–14h (mid ~11h) | ~11x |
| Phase 15 — Quality Cycle #2 | Docs refresh, issue triage, second-model review | ~3:00 | 20–35h (mid ~27h) | ~9x |
| **Total** | | **~20:52** | **~255–370h (mid ~310h)** | **~15x** |

\* Estimated from plan file scope; no formal block recorded for these phases.

> **Calendar context:** At 2 hours per night of solo development, the projected manual midpoint
> of ~310 hours equals 155 evenings — approximately 22 weeks (5 months) of sustained part-time
> effort. The full range (255–370h) spans 17–31 weeks (4–7.5 months).

---

## Section 5 — Lessons Learned

### 1. The Phase Planning Session pays for itself before a line of code is written

Every phase began with a structured planning session: present scope, confirm field encodings,
walk through example inputs and outputs, get explicit user approval. The payoff was most
visible in Phase 13, where the planning session caught three HIGH-severity encoding errors
in the original spec before any code existed — `targetVpd` ×10 vs ×100, raw integer
encoding for temperature and humidity fields, and the wrong `atType` value for VPD mode.
These would have produced silently incorrect writes to live devices. Catching them in
planning cost five minutes; fixing them post-merge would have cost a rollback, corrected
writes, and a re-review cycle.

The planning session also surfaced design decisions that benefited from user input before
being locked in: the VPD midpoint strategy for grow stage templates (Phase 14), the
partial-failure response format for sequential multi-write operations, and the decision
to expand `set_port_mode` from four simple modes to all eight with mode-specific optional
parameters.

### 2. Two-model independent review finds what the first pass missed

The most productive defect-finding mechanism across the project was giving a second model
cold access to the code with no context from the first-pass reviews. In Phase 11, Opus 4.7
independently found the token refresh race condition (HIGH — N concurrent 401s triggering N
authenticate() calls) and the thread-safety gap in write rate limiting, both of which had
passed the per-PR Sonnet Gate 1 and Gate 2 reviews on every prior phase. In Phase 15, a
cold Sonnet instance independently found the VPD banker's rounding bug — present since Phase
13, passed through Phase 14 gate reviews untouched — and the check_vpd_drift alert text
that gave backwards grow advice for both HIGH and LOW VPD conditions.

The pattern is consistent: a model reviewing its own prior work or reviewing code with full
context of how it was written has a systematic blind spot. Independent cold review resolves
it. The cost is one additional session per quality cycle; the return is finding bugs that
would otherwise reach production.

### 3. The living plan file as shared state across 16 sessions

No session in this project relied on prior conversation context. Every session opened by
reading `.claude/ac-infinity-mcp-v1-implementation.md` fresh, found the complete API
surface documentation, all 15 API quirks, the test routing decision rules, field encoding
references, lessons learned from prior phases, and the exact closing format expected before
ending. Every session closed by writing back its lessons learned, time investment block, and
defect log.

The practical effect: 16 consecutive sessions with zero context loss and zero "wait, what
was the plan?" interruptions. When a later phase needed to know whether `targetVpd` was
scaled by 10 or 100, the answer was already in the plan file from Phase 12 research. When
Phase 14 needed to call the Phase 13 write tools correctly, the parameter signatures were
documented. The plan file was the team's shared memory.

### 4. `dry_run=True` as the default for every write tool

Every write-control tool in this project ships with `dry_run=True` as the default. A tool
call with no explicit `dry_run=False` returns the planned change without executing it. This
is not a flag for testing — it is the design contract for all write tools: show before act,
require explicit opt-in.

This pattern resolved the primary safety concern for a grow automation tool: a misconfigured
or hallucinated tool call that sets fan speed to 0 in a sealed room is a plant-loss event.
The `dry_run=True` default means a grower can ask Claude to "set my exhaust fan to speed 8"
and review the exact payload that would be sent before approving it. The cost is one
additional round-trip; the return is confidence that the tool does what you think it does
before it does it.

### 5. Per-controller-type test split catches bugs unit tests miss

The `tests/devices/` directory contains two separate test files for two separate behavioral
contracts: `test_legacy_controller.py` (devType 11/18 — read-before-write, all 77 fields,
no `modeSetid`) and `test_ai_plus_controller.py` (devType 20+ — static full payload,
`newFrameworkDevice=true`). Generic unit tests in `tests/common/` use a mock device that
does not exercise either controller-type-specific path.

This split found controller-type bugs that would have been invisible to a single-fixture
test suite. The `modeSetid` field exclusion for legacy controllers (which returns a 403 if
included) was tested explicitly per controller type. The AI+ static payload completeness
check — all required fields present even when unchanged — was validated against actual
fixture responses. Per-type fixtures also made it trivial to verify that a change to the
shared `build_write_payload()` function behaved correctly on both code paths.

### 6. API quirk documentation is as valuable as the code

The AC Infinity API has no official documentation. All 15 confirmed quirks were discovered
through community reverse engineering and documented in `docs/API.md`. Several of these
quirks are silent failure modes: `modeType=2` must be set when `onSpead > 0` or the change
appears to succeed but doesn't persist; `modeSetid` must be excluded for legacy controllers
or the write returns 403; `pageNum` in the history API is silently ignored. None of these
are documented anywhere by the manufacturer.

Maintaining `docs/API.md` as a living artifact — updated on every phase that touched the
API — meant that each new tool implementation had a reliable reference instead of
re-discovering quirks through trial and error on live hardware. The Phase 13 planning
session cross-referenced API.md field encodings against Phase 12 fixture data to catch
the three HIGH encoding errors before implementation. The documentation was worth more
than the code that produced it.

### 7. Test coverage as a diagnostic, not a goal

Phase 11 opened with a reported coverage of 19.12% — which looked alarming until the
invocation was checked. The coverage run had been scoped incorrectly by an explore agent;
actual coverage when run correctly was 87.25%. This episode established the discipline of
always verifying the tool invocation, not just the output number.

The path from 87% to 100% was not forced. Reaching 100% required only two `# pragma: no cover`
annotations — both on genuinely unreachable defensive branches. Every other gap represented
a real missing test: the invalid-stage error path in `check_vpd_drift`, the `get_devices`
exception path in `apply_grow_stage_template`, the connection error branch in the
retry-decorated client methods. Finding these through coverage gaps was cheaper than finding
them through incident reports.

---

## Section 6 — What Remains

### v2.0 Roadmap

v2.0 targets external UIS sensors (CO2, pH, EC/TDS, soil moisture, water temperature, water
level, light level) and Bluetooth-local control. These require hardware not accessible via
the WiFi cloud API and are intentionally out of scope for v1.0.

All v2.0 features are tracked as GitHub Issues against the
[v2.0 milestone](https://github.com/ober37/ac-infinity-mcp/milestone/4). Each issue is a
self-contained spec with user story, API field references, and acceptance criteria — ready
to implement by pointing Claude at the issue URL.

**Open issue in v1.0 scope deferred to v2.0:**
- [#22 — User-defined and overridable grow stage templates](https://github.com/ober37/ac-infinity-mcp/issues/22)

### Peekaboopoint Monorepo Cleanup

The AC Infinity tools still exist in the `ober37/peekaboopoint` monorepo at
`scripts/acinfinity/`. After confirming the standalone package works end-to-end in the
grower's environment, the monorepo should be updated to pull `ghcr.io/ober37/ac-infinity-mcp:latest`
instead of the local build, and the local `scripts/acinfinity/`, `tests/acinfinity/`, and
`docker/ac-infinity/` directories removed. This cleanup is tracked in the plan file under
"Changes Required in Peekaboopoint Monorepo."

---

## Section 7 — PR Appendix

All merged PRs in chronological order.

| PR | Merged | Title |
|---|---|---|
| [#1](https://github.com/ober37/ac-infinity-mcp/pull/1) | 2026-05-20 | feat: initial scaffold, core module migration, CLAUDE.md |
| [#2](https://github.com/ober37/ac-infinity-mcp/pull/2) | 2026-05-20 | test: migrate existing tests, add tests for new modules |
| [#3](https://github.com/ober37/ac-infinity-mcp/pull/3) | 2026-05-20 | feat(analytics): Phase 3 — health score, trend detection, port activity MCP tools |
| [#4](https://github.com/ober37/ac-infinity-mcp/pull/4) | 2026-05-20 | refactor(client): typed exceptions + full API docs (Phase 4) |
| [#5](https://github.com/ober37/ac-infinity-mcp/pull/5) | 2026-05-21 | ci: add GitHub Actions CI, CodeQL, Dependabot; add pip-audit to dev deps |
| [#6](https://github.com/ober37/ac-infinity-mcp/pull/6) | 2026-05-21 | chore(deps): Bump actions/checkout from 4 to 6 |
| [#7](https://github.com/ober37/ac-infinity-mcp/pull/7) | 2026-05-21 | chore(deps): Bump github/codeql-action from 3 to 4 |
| [#8](https://github.com/ober37/ac-infinity-mcp/pull/8) | 2026-05-21 | chore(deps): Bump actions/setup-python from 5 to 6 |
| [#9](https://github.com/ober37/ac-infinity-mcp/pull/9) | 2026-05-21 | feat(write-foundation): implement get\_mode\_settings, set\_port\_mode, build\_write\_payload |
| [#10](https://github.com/ober37/ac-infinity-mcp/pull/10) | 2026-05-21 | feat(server): add set\_port\_speed, set\_port\_on, set\_port\_off MCP tools |
| [#11](https://github.com/ober37/ac-infinity-mcp/pull/11) | 2026-05-21 | feat(write): Phase 8 guard rails, AI+ documented limitation, 403 retry |
| [#12](https://github.com/ober37/ac-infinity-mcp/pull/12) | 2026-05-21 | feat(docker): Docker + packaging + Claude Desktop integration |
| [#13](https://github.com/ober37/ac-infinity-mcp/pull/13) | 2026-05-21 | feat(phase-10): MCP wire protocol + main() startup + expanded live tests |
| [#14](https://github.com/ober37/ac-infinity-mcp/pull/14) | 2026-05-21 | feat(phase-11): README refactor + deep quality cycle |
| [#15](https://github.com/ober37/ac-infinity-mcp/pull/15) | 2026-05-21 | chore(plan): sync implementation plan — phases 11-16 |
| [#16](https://github.com/ober37/ac-infinity-mcp/pull/16) | 2026-05-21 | chore: remove hardcoded local paths, move closing requirements to plan file |
| [#18](https://github.com/ober37/ac-infinity-mcp/pull/18) | 2026-05-21 | feat(server): get\_port\_status + get\_port\_settings (Phase 12) |
| [#19](https://github.com/ober37/ac-infinity-mcp/pull/19) | 2026-05-21 | feat(server): Phase 13 — set\_vpd\_automation, set\_temperature\_automation, set\_humidity\_automation, set\_port\_mode |
| [#20](https://github.com/ober37/ac-infinity-mcp/pull/20) | 2026-05-21 | feat(server): Phase 14 — apply\_grow\_stage\_template + 3 MCP prompts |
| [#29](https://github.com/ober37/ac-infinity-mcp/pull/29) | 2026-05-22 | feat(quality): Phase 15 — quality cycle, docs, community files, GitHub hardening |
