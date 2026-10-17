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

**Change record:** Source fixes and the test-isolation correction are in the fork's audit PR; this document describes observed state, not approval to merge or deploy. Findings are time-bound to the dated checks above.
