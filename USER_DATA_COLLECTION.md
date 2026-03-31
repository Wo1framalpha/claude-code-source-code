# User Data Collection & `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`

This repo contains the decompiled source for the Claude Code CLI bundle. The notes below summarize what user information the CLI collects/sends, and which parts are suppressed when `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` is set.

## What gets collected
- **Identity & account markers** (`src/utils/telemetryAttributes.ts`, `src/utils/user.ts`): generated `user.id`/device ID, `session.id`, optional `app.version`, OAuth-derived `organization.id`, `user.email`, `user.account_uuid` plus a tagged account ID, and terminal type.
- **Account profile & billing context** (`src/utils/user.ts`): subscription tier, rate-limit tier, first-token timestamp, and optional GitHub Actions metadata (`actor`, repo IDs) when running in CI.
- **Environment & runtime context** (`src/services/analytics/metadata.ts`): platform + arch, Node version, terminal, installed package managers/runtimes, CI/remote/container flags, optional coworker/container/remote session IDs, tags, WSL version, Linux distro/kernel, VCS type, build/version info, deployment environment, and GitHub Actions runner details.
- **Session/event context** (`src/services/analytics/metadata.ts`): current model/betas, interactive/client type, entrypoint/SDK version, agent/team identifiers, SWE Bench IDs, and a hashed repo remote (`rh`) for join with server data.
- **Process/runtime metrics** (`src/services/analytics/metadata.ts` `buildProcessMetrics`): uptime, memory stats, CPU usage, and CPU percent deltas.
- **Telemetry payloads** (`src/services/analytics/firstPartyEventLogger.ts`, `src/utils/telemetryAttributes.ts`): OTLP/1P events include the identity, account, environment, and process metadata above; additional per-event fields come from individual `logEvent*` call sites.
- **User-submitted content**: feedback/bug reports (`src/commands/feedback/index.ts`) and error reports (`src/utils/log.ts`) send user-entered text and stack traces when enabled.
- **Account-setting fetches**: Grove settings/notice status (`src/services/api/grove.ts`), referral eligibility/redemptions (`src/services/api/referral.ts`), metrics opt-out status (`src/services/api/metricsOptOut.ts`), and bootstrap/model option data (`src/services/api/bootstrap.ts`) call Anthropic APIs with OAuth/org/account IDs and cache the responses locally.

## Impact of `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
- The env var forces the privacy level to `essential-traffic` (`src/utils/privacyLevel.ts`), which makes `isTelemetryDisabled()`/`isAnalyticsDisabled()` return true. This stops Statsig/Datadog/1P event sinks and OTLP initialization, so the identity/environment/process data above is not sent.
- Error reporting and feedback submission are short-circuited (`src/utils/log.ts`, `src/commands/feedback/index.ts`), preventing those user inputs from leaving the machine.
- Background/nonessential fetches that would otherwise include org/account identifiers are skipped entirely: official MCP registry prefetch (`src/services/mcp/officialRegistry.ts`), bootstrap/config fetch (`src/services/api/bootstrap.ts`), metrics opt-out check (`src/services/api/metricsOptOut.ts`), Grove/referral prefetch (`src/services/api/grove.ts`, `src/services/api/referral.ts`), overage credit/fast-mode/release-notes/model-capability fetches, and other `isEssentialTrafficOnly()` gates.
- With the flag set, only essential product calls (e.g., direct model API traffic) proceed; ancillary analytics/telemetry and convenience fetches are suppressed.
