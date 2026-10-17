# Hermes deployment audit — sanitized external handoff

**Snapshot:** 2026-10-17 (MSK). **Scope:** one self-hosted Hermes Agent deployment and its local source checkout. This is an operational review, not a penetration test or a complete attestation of all connected services. No credentials, raw logs, private network topology, user records, or runtime configuration files are included. The deployment-specific inventory is held in a separate private repository.

## Evidence and version

- `hermes --version` reported **v0.22.2+1087.g6223e53** (`6223e535a26a`), installed from Git. This includes local changes in the fork; it does not establish that the already-running gateway loaded those changes.
- The gateway user service was **active** with zero systemd restarts reported at inspection; it had started at 10:11 MSK. Internal health checks returned HTTP 200 in the sampled check, while an unauthenticated external API request returned HTTP 401. Neither check exercises all agent tools or provider routes.
- `hermes doctor` completed in an earlier audit pass. An active service and passing doctor do **not** imply error-free behavior.

## Runtime architecture and trust boundaries

`messaging clients → Hermes gateway / agent → local tools & scheduled jobs → external APIs / private repositories`.

- The configured primary model was **openai-codex/gpt-6-sol**. The configured fallback order was OpenRouter `openai/gpt-6-sol`, `openai/gpt-6-luna`, `openrouter/free`. Configuration establishes intended routes, **not** a successful live request on each route. The user owns the routing decision; this audit did not alter it.
- Web search and extraction were configured for Exa. Two enabled MCP entries were listed: `exa` and a business-monitor connector. The latter exposes query/status tools; its presence is not evidence of production writes.
- The agent can act through filesystem/shell tools, Git, messaging, scheduled jobs, provider APIs and installed integration tools. Authorization and approval rules must be evaluated at each boundary; a connected tool alone is not proof of a successful or authorized action.
- A private Git-backed knowledge Vault is both context source and a write target under its repository procedure. Other business systems (monitoring, CRM export, dashboards, email/calendar, messaging and router management) are documented in the **private** inventory with separate labels for *verified access*, *historical documentation*, and *not verified*. Do not infer direct administrative control over all of them.

## Findings

1. **Running-code uncertainty (high operational priority).** The source checkout contains fixes for OAuth credential selection and the smart-approval guardian retry, committed as `6223e535a2`. The gateway was started before these changes were confirmed in its process. No gateway restart was performed during this audit; live execution of those fixes remains unverified.
2. **Regression suite incomplete.** A broad local run reported **215 passed, 32 failed**. A targeted run for the changed paths passed. Some broad-run failures interacted with the test/home bootstrap boundary; the 32 failures have **not** all been root-caused, and must not be relabeled harmless without reproducing in an isolated test home.
3. **Operational warnings observed.** An earlier sampled gateway-journal review found empty guardian responses, tracebacks, clean EOF events and warnings relating to Codex OAuth availability. These are historical counts in a sampled interval, not current rates or proof of an outage. The gateway remained active. Raw journals are intentionally omitted because they may contain sensitive information.
4. **Backup/restore proof absent.** Prior private inventory did not verify a recent successful restoration of the Hermes session database. The existence of a live database or backup-shaped files does not prove recoverability.
5. **Scope gaps.** Production container/image metadata of the separate business monitor, actual fallback-provider failover, each external write permission, full message delivery, and end-to-end provider health were not attested here.

## Recommended follow-up (not performed)

- From an independent operator shell and maintenance window, restart the gateway if deployment of the reviewed changes is desired; then recheck service health and the guardian/OAuth code path. Avoid restarting it from a tool call hosted by that same gateway.
- Re-run the broad test suite under an isolated `HERMES_HOME`, classify every failing test, and record exact command/environment and output **after redaction**.
- Validate each provider/fallback with a minimal authorized request, and test one backup restore in an isolated environment.
- Maintain a least-privilege integration register with owner, read/write scope, audit log, expiry/review date and verified evidence. Never attach `.env`, `auth.json`, session transcripts, full config, raw logs or personal data to a public issue/PR.

## Follow-up check — 2026-10-17 11:29 MSK

- The operator restarted the gateway at 11:08:44 MSK; `systemctl show` subsequently reported `active/running` and zero restarts. A sampled journal query since restart found no matches for the selected traceback/error/OAuth/EOF strings. This is a liveness check, **not** an end-to-end exercise of the two changed paths.
- The isolated auxiliary-client and smart-approval test modules now pass: **247 passed**. Their test fixture was corrected to stub the unrelated HTTP transport helper instead of importing the production bootstrap and probing the live manifest. The change is committed in the same PR; production application code was not changed by this follow-up.
- A single-process whole-suite invocation still cannot be taken as a passing full suite: collection encountered eight missing-optional-package errors despite the package being importable before pytest, consistent with cross-module bootstrap/path effects. The project's canonical per-file runner avoided that collection problem for the selected files. Anthropic is outside the configured production routing (the primary is OpenAI Codex and all three fallbacks use OpenRouter); its optional dependency and end-to-end tests are deliberately excluded from further validation. A green **full** suite of enabled-provider tests and live provider failover remain unverified.

## Enabled-provider follow-up — 2026-10-17 11:44 MSK

- Canonical per-file runner: **317 tests passed across 15 selected files, 0 failed** in 41.7s (four workers), with `HERMES_HOME` and `HERMES_TEST_ISOLATION` set to a scratch sandbox and lazy installs disabled. Coverage includes Codex auth/recovery, OpenRouter caching, provider/fallback routing and credentials, gateway fallback reload/auth, auxiliary client, CLI fallback configuration, and smart approval. No Anthropic optional extra or Anthropic-provider end-to-end test was run in this follow-up. Test-only bootstrap stubs prevent unit-test imports from probing the live install manifest; the gateway reload fixture now uses a non-Anthropic OpenRouter model. This is **not the whole suite**.
- `hermes-gateway.service` was active/running at 11:44 MSK with the same PID since the operator's 11:08:44 MSK restart (`NRestarts=0`). This conversation exercises the real primary route, but production credentials were not copied into the sandbox and no deliberate failure was induced. Live OpenRouter failover is **not independently verified**.

## Expanded isolated run and backup-route probe — 2026-10-17 11:55 MSK

- The standard non-E2E discovery contains 6,302 test files after excluding Anthropic-named files and external-service suites (`e2e`, `integration`, `docker`). A full per-file run was started in a sandbox, then deliberately interrupted at the start after the first files exposed repeated real-home manifest access under the test guard. It has **no whole-suite result**; its partial log is local only.
- A bounded per-file run of 179 provider/auth/fallback-related files completed in 186.7s: **1,587 passed, 152 failed, 3 skipped**. Fifty files exited nonzero (48 with failed tests, one with errors after passing tests, one with a collection error). Forty-six of those 50 failure blocks contain the guard's refusal of an attempted read of the real `~/.hermes/manifest.json`; four do not. These are not safely dismissible or proof of production breakage. Other observed failure signatures include a missing optional `aiohttp` import, a timeout-ownership assertion, and a client-wiring TypeError; their root causes and overlaps remain to be investigated. Do not disable the guard or label this suite green.
- Two minimal CLI probes requesting the OpenRouter provider and `openai/gpt-6-sol` returned `DIAG_OK` with exit code 0; the CLI output did not independently attest which provider served the request, and no primary-provider outage was induced. Therefore neither OpenRouter identity nor automatic Codex→OpenRouter **live failover** is proven by these probes. The configured routing was not changed. Gateway remained active/running (`NRestarts=0`) at the follow-up check.
- GitHub reported **no PR checks** for this branch (the `gh pr checks` nonzero exit is not a failing check). Next engineering step: reproduce the real-home manifest probe on a safe independent checkout, isolate import-time payload detection without weakening the I/O guard, then re-run the failed files and the wider non-E2E suite. Check the remaining non-guard failures separately; do not merge or deploy on the basis of the 15-file green subset.

## Additional isolated triage — 2026-10-17

- A local test-only checkout-manifest fixture avoids the installed-parent lookup for the repository root while leaving the real-home I/O guard active for all other roots. Codex auxiliary test doubles were updated to accept the `model` keyword argument. With sandboxed `HERMES_HOME`/`HERMES_TEST_ISOLATION` and lazy installs disabled, a targeted three-file run passed **31/31** (cloudflare headers, usage attribution, local cwd permission fallback). Earlier local single-file and four-file smoke runs passed **27/27** and **57/57**, respectively. These overlapping counts must not be summed as unique coverage; the broader 179-file provider run remains non-green.
- Two gateway tests that launch a child daemon did **not** pass in local retests: their child subprocess did not inherit the sandbox's optional-dependency path. A direct pytest retry also encountered an isolated fixture failure. Its child process attempted an installation bootstrap in the scratch home; the child log reported an `apt` permission failure. A staging directory was subsequently observed under the live Hermes tools directory, but its creation time and provenance were **not established**. No cleanup, deployment or gateway restart was performed. Before any further child-daemon test, verify that subprocesses inherit an isolated home, installation root and dependency environment; do not assume the Python in-process I/O guard covers children. The gateway still reported `active/running` with `NRestarts=0` after these retries.
- This follow-up does not establish end-to-end backup-provider identity, automatic failover, or a green full provider suite. Do not merge or deploy on this evidence alone.

## Child-daemon and automatic-fallback verification — 2026-10-17

- Added test-only inheritance of `HERMES_TEST_ISOLATION`, `HERMES_DISABLE_LAZY_INSTALLS`, `HERMES_SKIP_NODE_BOOTSTRAP` and `UV_OFFLINE` in child-daemon and CLI E2E harnesses. Installed `aiohttp==3.14.3` in the repository's local `.venv` (not in the live gateway). With a scratch `HERMES_HOME` and no production config changes, the two gateway test modules plus a managed-worker fallback case passed **5/5** in 28.62s. This includes the model-lock case, which must *refuse* fallback rather than silently switch models.
- The real `hermes -z` subprocess E2E fallback file passed **2 tests, 1 known XFAIL** in 48.99s against two loopback OpenAI-compatible servers and an isolated home. A primary returning 503 yielded a successful fallback response on a resumed conversation; the test verifies the fallback server received the entire conversation and usage attributes the fallback model. A 5xx credential-refresh error also fell back successfully. The connection-refused credential-resolution variant remains **XFAIL** (#120608): transport errors during credential resolution still prevent fallback. A separate rerun confirmed that exact XFAIL.
- The E2E exercises the actual CLI process and fallback implementation with simulated providers, **not** live Codex→OpenRouter service failover; no production route or secrets were changed. The earlier 179-file suite remains non-green, and this result alone is not approval to merge/deploy.

## Runtime-provider unit follow-up — 2026-10-17

- Seven additional provider-focused test files passed **143/143** when run one file per process with the repository `.venv` interpreter. The launcher passed an explicit scratch `HERMES_HOME` and `HERMES_TEST_ISOLATION`, disabled lazy installs and Node bootstrap, set `UV_OFFLINE=1`, and supplied a minimal environment without credentials. The tested modules cover runtime resolution and OpenAI-compatible runtime wiring, late binding, precedence, configured-provider model switching, keyed credential-pool behavior, and provider auth seams. Per-file output is retained in local scratch logs, not in this repository.
- The gateway still reported `active/running` and `NRestarts=0` at this check; GitHub reported no PR checks. This unit run is additional regression evidence, **not** a whole-suite green result, a live provider-identity assertion, or a production failover exercise. No live route, gateway service, or production credentials were changed.

**Change record:** Source fixes and the test-isolation correction are in the fork's audit PR; this document describes observed state, not approval to merge or deploy. Findings are time-bound to the dated checks above.
