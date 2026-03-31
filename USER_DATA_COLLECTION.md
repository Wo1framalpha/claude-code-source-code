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
# 用户数据采集概览与 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 影响

下表梳理 CLI 会采集/发送的用户信息，以及受 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 控制的网络与遥测路径。

## 采集内容

| 类别 | 采集内容 | 主要来源文件/逻辑 |
| --- | --- | --- |
| 身份与账号 | 设备 ID（`user.id`）、会话 ID、版本号、OAuth 组织/账号 UUID、账号标签 ID、用户邮箱、终端类型 | `src/utils/telemetryAttributes.ts`、`src/utils/user.ts` |
| 订阅/计费上下文 | 订阅层级、速率限制层级、首次出 token 时间（可选）、GitHub Actions 元数据（actor/repo IDs） | `src/utils/user.ts` |
| 环境与运行时 | 平台/架构/Node 版本、终端、包管理器/运行时列表、CI/远程/容器标记、远程会话/容器 ID、标签、WSL/发行版/内核、VCS、构建与部署环境、GitHub Actions runner 详情 | `src/services/analytics/metadata.ts` (`buildEnvContext`) |
| 会话/事件上下文 | 当前模型与 betas、交互/客户端类型、入口/SDK 版本、代理/团队标识（agentId/parentSessionId/agentType/teamName）、SWE Bench IDs、仓库远程哈希 (`rh`) | `src/services/analytics/metadata.ts` (`getEventMetadata`) |
| 进程指标 | 运行时长、内存各项、CPU 使用及增量百分比 | `src/services/analytics/metadata.ts` (`buildProcessMetrics`) |
| 遥测载荷 | OTLP / 1P 事件会包含上面身份、环境、进程等字段；每个 `logEvent*` 调用可附加事件特定数值字段 | `src/services/analytics/firstPartyEventLogger.ts`、`src/utils/telemetryAttributes.ts` |
| 用户提交内容 | 反馈/bug 文本，错误上报（堆栈） | `src/commands/feedback/index.ts`、`src/utils/log.ts` |
| 账号配置拉取 | Bootstrap & 模型选项、Metrics opt-out 状态、Grove 设置、Referral 资格/兑换，附带 OAuth/org/account 标识并缓存 | `src/services/api/bootstrap.ts`、`src/services/api/metricsOptOut.ts`、`src/services/api/grove.ts`、`src/services/api/referral.ts` |

## `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 的作用

| 影响范围 | 具体效果 | 参考代码 |
| --- | --- | --- |
| 隐私级别 | 设为 `essential-traffic`，`isTelemetryDisabled()` / `isAnalyticsDisabled()` 返回真 | `src/utils/privacyLevel.ts` |
| 遥测/分析 | Statsig/Datadog/1P/OTLP 遥测初始化与发送被阻断，以上身份/环境/进程数据不再外送 | `src/services/analytics/config.ts`、`src/utils/telemetry/instrumentation.ts` |
| 错误与反馈 | 错误上报和反馈命令被短路 | `src/utils/log.ts`、`src/commands/feedback/index.ts` |
| 非必要网络调用 | 跳过官方 MCP registry 预取、bootstrap 配置、metrics opt-out、Grove/Referral 预取、以及其他 `isEssentialTrafficOnly()` 守卫的便利请求（如模型能力、发布说明、自动更新等） | `src/services/mcp/officialRegistry.ts`、`src/services/api/bootstrap.ts`、`src/services/api/metricsOptOut.ts`、`src/services/api/grove.ts`、`src/services/api/referral.ts` 等 |
| 仅保留必要流量 | 模型推理等核心调用仍可进行；其他非必要流量被抑制 | 行为由各调用处的 `isEssentialTrafficOnly()` 分支决定 |
