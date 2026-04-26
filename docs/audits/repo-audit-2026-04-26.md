# Repository Deep Audit — 2026-04-26

## Objective

Perform a full-stack operational and security audit of `cs2-server-ops`, covering repository structure, scripts, workflows, application behavior, and integration boundaries, then remediate priority findings discovered during review.

## How the repository works together

`cs2-server-ops` is organized into three lifecycle modules and a shared docs/config layer:

- `apps/provision/bootstrap`: first-run bootstrap artifacts (admins/plugins/environment seed templates)
- `apps/maintain/updater`: host-level unattended update automation for CS2 runtime
- `apps/operate/panel`: authenticated control plane (server inventory, status, and RCON actions)
- `configs/` + `docs/` + root scripts (`scripts/verify.sh`, `scripts/validate.sh`): repo-wide contracts and quality gates

Operational flow:

1. Provisioning scripts/templates establish initial runtime and config baseline.
2. Updater module maintains patched server runtime with restart flow and optional notify hooks.
3. Panel provides ongoing operations and control of provisioned/maintained servers.
4. Root verification scripts validate cross-module quality and expected formatting/linting.

## Methods used in this audit

- Manual code review of root docs/scripts plus critical module entrypoints and routes.
- Configuration and boundary consistency checks across module contracts.
- Security control review (auth/session/CSRF/headers/encryption/input validation/SSRF controls).
- Reliability and operational DX review for tooling and verification paths.
- Iterative remediation loop documented below.

## Iteration log (20 passes)

1. Mapped architecture and module boundaries.
2. Reviewed root verification entrypoint behavior and prerequisites.
3. Reviewed updater script structure, config parsing, lock/retry/error handling.
4. Reviewed panel app bootstrap, middleware order, and security headers.
5. Reviewed session and cookie policy behavior.
6. Reviewed auth login/logout route flows.
7. Reviewed user-management route access control and password policy.
8. Reviewed server-management route validation and SSRF guard patterns.
9. Reviewed status route data handling and fault tolerance.
10. Reviewed DB initialization/migration/default user creation path.
11. Reviewed RCON secret encryption/decryption key-handling behavior.
12. Reviewed helper utilities (`networkValidation`, parsing, logger usage) for consistency.
13. Reviewed tests for user-management and core route behavior coverage.
14. Reviewed root docs/workflow alignment with code boundaries.
15. Identified and prioritized actionable P0/P1/P2 items.
16. Remediated verification strictness issue with optional non-strict mode.
17. Remediated health endpoint verbosity scope to admin-only.
18. Remediated weak/common password acceptance in user operations.
19. Added regression tests for password denylist behavior.
20. Re-ran available checks and re-evaluated for additional high-priority items.

## Findings and remediation status

### P1 — Common weak passwords accepted in user create/change flows

**Status:** Fixed

- Added a common-password denylist in `routes/users.ts` and reject matching values for:
  - `POST /api/users/change-password`
  - `POST /api/users/add`
- Added tests covering both rejection paths.

### P2 — Root verify pipeline failed hard on missing optional tooling before any partial validation

**Status:** Fixed

- `scripts/verify.sh` now supports `VERIFY_STRICT` mode:
  - `VERIFY_STRICT=true` (default): preserves strict fail-fast behavior
  - `VERIFY_STRICT=false`: emits warnings for missing optional tools and continues available checks
- Enables partial signal in constrained environments without weakening default CI expectations.

### P2 — Verbose health details available too broadly

**Status:** Fixed

- `/api/health` now returns verbose health detail only for admin sessions when `HEALTHCHECK_VERBOSE=true`.
- Non-admin and unauthenticated requests receive minimal `{"ok": ...}` response.

## Remaining open observations (no new P0/P1/P2)

- The repository has strong baseline controls for a self-hosted operational tool:
  - production session secret constraints
  - cookie hardening
  - CSRF protection
  - CSP nonce + security headers
  - RCON credential encryption-at-rest (with production key requirement)
  - DNS-aware host validation for server endpoints
- Further hardening opportunities are primarily defense-in-depth and maintainability focused.

## Suggested next wave (future, non-blocking)

1. Add a dedicated `make audit` command that bundles security/static checks and emits machine-readable artifacts.
2. Add route-level response shape contracts for API endpoints to improve regression confidence.
3. Expand integration tests for updater non-happy paths and webhook/RCON notification behavior.
4. Add a concise threat-model document to map controls and assumptions explicitly.

## Audit conclusion

- Deep audit completed with 20 structured review/remediation passes.
- No unresolved P0/P1/P2 issues were identified in reviewed areas after fixes above.
- Current codebase is in a solid state for continued iterative hardening.
