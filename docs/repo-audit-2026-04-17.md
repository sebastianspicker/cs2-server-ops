# Repository Audit - 2026-04-17

## Scope

This audit covered the full repo:

- root/shared scripts and CI
- `apps/provision/bootstrap`
- `apps/maintain/updater`
- `apps/operate/panel`
- `configs/examples`
- shared docs/contracts under `docs/`

Searches for prior audit artifacts using filename patterns and markdown heading matches for `audit`, `report`, `review`, and `analysis` found no existing audit files to move into `deprecated/`.

## Category Map

1. Shared repo infrastructure
   - Entrypoints and boundaries: `README.md`, `docs/architecture.md`
   - Shared verification: `scripts/verify.sh`
   - Shared validation: `scripts/validate.sh`
   - CI/security workflows: `.github/workflows/ci.yml`, `.github/workflows/secret-scan.yml`

2. Provision/bootstrap
   - Runtime env contract: `apps/provision/bootstrap/env/server.env.example`
   - Seed generation: `apps/provision/bootstrap/scripts/bootstrap-admins.sh`, `apps/provision/bootstrap/scripts/bootstrap-plugins.sh`
   - Operator guidance: `apps/provision/bootstrap/README.md`, `docs/workflows/provision-server.md`

3. Maintain/updater
   - Main updater flow: `apps/maintain/updater/update_cs2.sh`
   - Config surface: `apps/maintain/updater/cs2-auto-update.conf.example`
   - Test harness and CI: `apps/maintain/updater/tests/run.sh`, `apps/maintain/updater/Makefile`

4. Operate/panel
   - App bootstrap and security middleware: `apps/operate/panel/app.ts`
   - DB/auth bootstrap: `apps/operate/panel/db.ts`, `apps/operate/panel/routes/auth.ts`
   - Inventory/status/game routes: `apps/operate/panel/routes/server.ts`, `apps/operate/panel/routes/status.ts`, `apps/operate/panel/routes/game/*.ts`
   - RCON lifecycle: `apps/operate/panel/modules/rcon.ts`
   - UI/static assets: `apps/operate/panel/views/*`, `apps/operate/panel/public/*`
   - Tests/docs: `apps/operate/panel/test/*`, `apps/operate/panel/docs/*`

5. Shared deployment examples
   - Compose examples: `configs/examples/compose/*.yaml`
   - Startup wrapper: `configs/examples/startup/server-start.sh`
   - Systemd examples: `configs/examples/systemd/*`

## Findings

### Shared repo infrastructure

1. High: repo-wide verification currently fails in the `operate` test suite, so the umbrella CI path is red right now.
   - `scripts/verify.sh:60-82`
   - `apps/operate/panel/package.json:22-23`
   - `apps/operate/panel/test/entrypoint.test.ts:98-103`
   - Observed failure: `npm test` returns `404` instead of `200` for a spawned entrypoint asset request, which aborts `./scripts/verify.sh` before the remaining repo checks finish.

2. Medium: CI-facing YAML validation uses `YAML.load_file`, which is Ruby's unsafe API family.
   - `scripts/verify.sh:54-55`
   - `apps/operate/panel/scripts/validate.sh:56-58`
   - `safe_load_file` is the safer equivalent for repo-controlled YAML validation.

3. Low: GitHub Actions are pinned to floating tags instead of immutable SHAs.
   - `.github/workflows/ci.yml:19-21`
   - `.github/workflows/secret-scan.yml:19-20`
   - This is a supply-chain hardening gap, not a functional bug.

### Provision/bootstrap and shared examples

4. High: the shared panel compose example mounts the database volume at `/app/data`, but the panel defaults to `/home/container/data/cspanel.db`.
   - `configs/examples/compose/panel.compose.yaml:9-10`
   - `apps/operate/panel/db.ts:9-12`
   - Result: persistent DB state can land in the container filesystem instead of the named volume.

5. High: the repo says not to publish placeholder secrets in templates, but both shared compose examples load example env files that contain placeholder credentials.
   - `docs/reference/env.md:10-16`
   - `configs/examples/compose/panel.compose.yaml:5-6`
   - `apps/operate/panel/.env.example:1-3`
   - `configs/examples/compose/server-runtime.compose.yaml:5-6`
   - `apps/provision/bootstrap/env/server.env.example:5-6`
   - This is direct contract drift in the published examples.

6. High: the provision env example advertises a broader runtime contract than the shipped startup wrapper or compose example actually consumes.
   - `apps/provision/bootstrap/env/server.env.example:1-10`
   - `configs/examples/startup/server-start.sh:4-10`
   - `configs/examples/compose/server-runtime.compose.yaml:1-10`
   - `docs/workflows/provision-server.md:5-7`
   - Variables such as `CS2_HOSTNAME`, `CS2_GSLT`, `CS2_CFG_FILE`, `CSS_ADMINS_FILE`, and `CSS_GROUPS_FILE` are documented as part of setup, but the example runtime path does not wire them in.

7. Medium: the runtime compose example does not actually connect the documented bootstrap assets together.
   - `configs/examples/compose/server-runtime.compose.yaml:1-10`
   - `configs/examples/startup/server-start.sh:1-10`
   - `apps/provision/bootstrap/scripts/bootstrap-admins.sh:1-25`
   - `apps/provision/bootstrap/scripts/bootstrap-plugins.sh:1-15`
   - It loads the env example, but it does not invoke the startup wrapper or mount/generated admin/plugin outputs.

8. Medium: the updater systemd example and updater quick-start instructions point to different installation layouts.
   - `configs/examples/systemd/cs2-auto-update.service:6-8`
   - `apps/maintain/updater/README.md:22-31`
   - `apps/maintain/updater/update_cs2.sh:183-202`
   - Following both documents together can leave the service unit referencing a script path/config path that the README never installs.

9. Low: the shared panel compose example is materially less hardened than the module-local compose file.
   - `configs/examples/compose/panel.compose.yaml:1-14`
   - `apps/operate/panel/docker-compose.yaml:18-29`
   - Missing protections include `read_only`, `tmpfs`, `no-new-privileges`, and resource limits.

10. Low: the startup wrapper is cwd-sensitive and does not validate numeric envs.
    - `configs/examples/startup/server-start.sh:4-10`
    - Launching it outside the CS2 install root breaks `./game/cs2.sh`, and malformed `CS2_PORT` or `CS2_MAXPLAYERS` values are passed through unchecked.

### Maintain/updater

11. High: a successful `steamcmd` run is treated as a successful update even when the build ID is unchanged afterward.
    - `apps/maintain/updater/update_cs2.sh:794-803`
    - The script logs `BUILDID_AFTER` but never verifies that the update actually changed anything before restarting the service and optionally sending a success webhook.

12. High: remote build lookup failure falls back to a full stop/update/start cycle, which can create avoidable downtime during transient SteamCMD or network issues.
    - `apps/maintain/updater/update_cs2.sh:645-652`
    - `apps/maintain/updater/update_cs2.sh:782-786`
    - This is safe from a consistency perspective, but too aggressive operationally.

13. Medium: stale-lock detection trusts only a PID file, so PID reuse can produce a false "already running" result.
    - `apps/maintain/updater/update_cs2.sh:481-487`
    - Recording command metadata or process start time would make stale-lock recovery safer.

14. Medium: `systemctl` is required before `--status` and `--dry-run` branches, reducing portability and breaking read-only status checks on non-systemd hosts.
    - `apps/maintain/updater/update_cs2.sh:742-745`
    - `apps/maintain/updater/update_cs2.sh:765-791`

15. Medium: config parsing strips everything after `#` without supporting quoting.
    - `apps/maintain/updater/update_cs2.sh:160-179`
    - Values containing `#` are mangled, which is a realistic hazard for URLs or future token-like values.

16. Medium: status mode conflates "unknown" with "update available".
    - `apps/maintain/updater/update_cs2.sh:765-770`
    - When either build ID is unavailable, the script reports an available update instead of an indeterminate state.

17. Low: disk-space parsing has OS-layout assumptions that may not hold across all `df` variants.
    - `apps/maintain/updater/update_cs2.sh:524-530`

18. Medium: the current test suite misses several of the highest-risk updater branches.
    - `apps/maintain/updater/tests/run.sh:286-290`
    - `apps/maintain/updater/tests/run.sh:353-383`
    - `apps/maintain/updater/tests/run.sh:458-478`
    - Missing cases include false-success updates, remote lookup failure in `--status`, PID reuse, and stop/start retry failures after partial progress.

### Operate/panel

19. High: startup RCON initialization skips the DNS/private-range revalidation used for reconnects.
    - `apps/operate/panel/modules/rcon.ts:89-99`
    - `apps/operate/panel/modules/rcon.ts:107-127`
    - `apps/operate/panel/modules/rcon.ts:248-332`
    - Persisted servers loaded at process boot are connected via `connect()` directly, while reconnects go through the hostname revalidation guard.

20. High: same-server RCON commands are not serialized, but several routes deliberately fire concurrent commands against one shared connection.
    - `apps/operate/panel/modules/rcon.ts:141-184`
    - `apps/operate/panel/routes/game/controls.ts:61-64`
    - `apps/operate/panel/routes/game/controls.ts:90-93`
    - `apps/operate/panel/routes/game/controls.ts:115-118`
    - `apps/operate/panel/routes/game/controls.ts:213-216`
    - A timeout path destroys the socket and removes listeners for the shared connection, so one in-flight command can break another.

21. Medium: `/api/setup-game` still has a half-applied state if `changelevel` succeeds and `execCfg` fails.
    - `apps/operate/panel/routes/game/match.ts:132-160`
    - The cfg name is validated before any RCON work, but runtime execution failure after map change still leaves the server changed without matching metadata.

22. Medium: adding an existing server leaks inventory existence across users.
    - `apps/operate/panel/routes/server.ts:191-201`
    - Returning `403 Incorrect RCON password for this server` reveals that another user already registered the same `IP:port`.

23. Medium: the add-server limiter is instance-local while the app-wide limiters are Redis-capable.
    - `apps/operate/panel/routes/server.ts:90-96`
    - `apps/operate/panel/app.ts:256-305`
    - In multi-instance deployments, `/api/add-server` can be bypassed by routing traffic across instances.

24. Low: the login form enforces `minlength="12"`, but the backend login route accepts any non-empty password.
    - `apps/operate/panel/views/login.ejs:35-44`
    - `apps/operate/panel/routes/auth.ts:18-23`
    - Existing accounts with shorter passwords remain valid server-side but can become impossible to log into through the UI.

25. Low: production behavior for unhandled promise rejections is "log and continue".
    - `apps/operate/panel/app.ts:428-430`
    - For a stateful control plane using DB, Redis, and persistent sockets, fail-fast behavior under a supervisor is usually safer.

26. Low: the panel README understates the RCON encryption requirement.
    - `apps/operate/panel/README.md:37-42`
    - `apps/operate/panel/db.ts:43-48`
    - Docs call `RCON_SECRET_KEY` "recommended", but production startup makes it mandatory.

27. Medium: test/build contracts around static assets are fragile.
    - `apps/operate/panel/package.json:8-14`
    - `apps/operate/panel/package.json:22-23`
    - `apps/operate/panel/test/entrypoint.test.ts:98-103`
    - `npm test` does not build the client bundle, but entrypoint coverage expects static assets from the spawned app process. The current verify run is already red here.

## Confirmed strengths

1. The panel has a solid hardening baseline: strong session checks, CSRF, CSP nonce, HSTS, and route validation.
   - `apps/operate/panel/app.ts:44-252`
   - `apps/operate/panel/routes/auth.ts:18-67`
   - `apps/operate/panel/routes/server.ts:76-182`

2. The updater module has a strong shell-quality baseline and a meaningful regression harness.
   - `apps/maintain/updater/Makefile`
   - `apps/maintain/updater/tests/run.sh`

3. Shared shell/bootstrap scripts are syntax-clean under `shellcheck` and `shfmt`.
   - `scripts/verify.sh:41-55`

## Verification run

**Original run (2026-04-17)**

1. `./scripts/verify.sh`: failed in `apps/operate/panel` tests.
2. `apps/maintain/updater -> make ci`: passed.
3. Provision bootstrap smoke tests: passed.
4. `docker compose -f configs/examples/compose/panel.compose.yaml config`: parsed successfully.
5. `docker compose -f configs/examples/compose/server-runtime.compose.yaml config`: parsed successfully.

**Follow-up run (post-fix)**

All 181 panel tests pass (181/181). `./scripts/verify.sh` exits 0. `make ci` in updater still
green. All 27 findings resolved.

## Resolution summary

| # | Severity | Finding | Resolution |
|---|----------|---------|------------|
| 1 | High | Panel test suite red (`verify.sh` aborts) | `npm test` now runs `build:client` first; test fixture stubs prerequisite binaries |
| 2 | Medium | `YAML.load_file` unsafe | Replaced with `safe_load_file` |
| 3 | Low | Actions pinned to floating tags | Pinned to immutable commit SHAs |
| 4 | High | Volume mount mismatch (`/app/data` vs `/home/container/data`) | Corrected in shared compose example |
| 5 | High | Placeholder credentials in compose examples | Examples use variable substitution; no literal secrets |
| 6 | High | Env contract drift (CS2_HOSTNAME etc. not wired) | `server-start.sh` wires all documented env vars |
| 7 | Medium | Runtime compose not connected to bootstrap assets | Compose mounts bootstrap outputs and invokes startup wrapper |
| 8 | Medium | Systemd unit vs README path mismatch | Both reference `/opt/cs2-server-ops/…` layout |
| 9 | Low | Panel compose missing hardening | Added `read_only`, `tmpfs`, `no-new-privileges`, resource limits |
| 10 | Low | Startup wrapper cwd-sensitive; no numeric validation | Uses `SCRIPT_DIR`; `require_integer_in_range` guards ports/player counts |
| 11 | High | False-success on unchanged build ID | `determine_post_update_state` compares build ID before/after |
| 12 | High | Lookup failure forces stop/update/start | Non-zero exit, service left running |
| 13 | Medium | Stale lock trusts PID alone | Lock records `process_start_time`; both validated on stale-lock check |
| 14 | Medium | `systemctl` required before `--status`/`--dry-run` | Status/dry-run exit before any `systemctl` call |
| 15 | Medium | Config `#` stripping ignores quoting | `strip_unquoted_comment` tracks quote state |
| 16 | Medium | `--status` conflates unknown with update-available | Unknown state returns distinct label + `exit 1` |
| 17 | Low | `df` parsing OS assumptions | `df -Pk` (POSIX portable) |
| 18 | Medium | Missing updater test cases | false-success, lookup fail, PID reuse, start failure all covered |
| 19 | High | RCON `init()` skips DNS/private-range revalidation | `connect()` → `createAuthenticatedConnection()` → `isResolvedHostAllowed()` |
| 20 | High | Concurrent RCON commands share connection | `enqueueServerTask` serialises per-server queue |
| 21 | Medium | `setup-game` half-applied state | cfg applied before `changelevel`; validation before any RCON |
| 22 | Medium | Server existence leak via RCON error | Generic "Unable to authenticate" returned |
| 23 | Medium | Add-server limiter instance-local | `RateLimitRedisStore` used when Redis configured |
| 24 | Low | Login form `minlength` vs backend mismatch | `minlength` attribute removed |
| 25 | Low | Unhandled rejection "log and continue" | `process.exit(1)` in production |
| 26 | Low | README understates RCON_SECRET_KEY requirement | Docs updated to "required in production" |
| 27 | Medium | Test/build contract around static assets fragile | `npm test` builds client bundle; path resolution fixed |

VERDICT: PASS
