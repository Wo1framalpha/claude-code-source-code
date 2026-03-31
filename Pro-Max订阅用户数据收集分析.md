# Pro/Max 订阅用户数据收集分析报告

## 概述

本报告详细分析了之前发现的客户端信息收集中，哪些部分会应用于 Pro/Max 订阅用户，以及订阅状态如何影响数据收集和功能访问。

## 1. 订阅类型定义与识别

### 1.1 订阅类型枚举
**文件**: `src/utils/auth.ts` (第1564-1730行)

系统支持的订阅类型：
- `'max'` - Claude Max 订阅
- `'pro'` - Claude Pro 订阅
- `'team'` - Claude Team 订阅
- `'enterprise'` - Claude Enterprise 订阅
- `null` - API 用户或非订阅用户

### 1.2 订阅状态检测函数

```typescript
// 第1662-1677行：获取订阅类型
export function getSubscriptionType(): SubscriptionType | null {
  // 首先检查模拟订阅类型（ANT-only 测试）
  if (shouldUseMockSubscription()) {
    return getMockSubscriptionType()
  }
  if (!isAnthropicAuthEnabled()) {
    return null
  }
  const oauthTokens = getClaudeAIOAuthTokens()
  if (!oauthTokens) {
    return null
  }
  return oauthTokens.subscriptionType ?? null
}

// 第1680行：检查是否为 Max 订阅用户
export function isMaxSubscriber(): boolean {
  return getSubscriptionType() === 'max'
}

// 第1699行：检查是否为 Pro 订阅用户
export function isProSubscriber(): boolean {
  return getSubscriptionType() === 'pro'
}

// 第1564-1570行：检查是否为 Claude AI 订阅用户（包括 Pro/Max/Team/Enterprise）
export function isClaudeAISubscriber(): boolean {
  if (!isAnthropicAuthEnabled()) {
    return false
  }
  return shouldUseClaudeAIAuth(getClaudeAIOAuthTokens()?.scopes)
}
```

## 2. Pro/Max 用户的数据收集

### 2.1 核心用户数据结构
**文件**: `src/utils/user.ts` (第34-128行)

对于 Pro/Max 订阅用户，系统会收集以下额外数据：

```typescript
export type CoreUserData = {
  deviceId: string                  // 所有用户：持久化设备ID
  sessionId: string                 // 所有用户：会话ID
  email?: string                    // Pro/Max用户：从OAuth获取
  appVersion: string                // 所有用户：应用版本
  platform: typeof env.platform     // 所有用户：操作系统平台
  organizationUuid?: string         // Pro/Max用户（如果在组织中）
  accountUuid?: string              // Pro/Max用户：OAuth账户UUID
  userType?: string                 // 所有用户：环境类型
  subscriptionType?: string         // Pro/Max用户：订阅类型 ✓
  rateLimitTier?: string            // Pro/Max用户：速率限制等级 ✓
  firstTokenTime?: number           // Pro/Max用户：首次使用时间 ✓
  githubActionsMetadata?: object    // 所有用户：GitHub Actions上下文
}
```

**Pro/Max 用户特有收集**（第78-128行）：

```typescript
export const getCoreUserData = memoize(
  (includeAnalyticsMetadata?: boolean): CoreUserData => {
    // ...
    let subscriptionType: string | undefined
    let rateLimitTier: string | undefined
    let firstTokenTime: number | undefined

    if (includeAnalyticsMetadata) {
      subscriptionType = getSubscriptionType() ?? undefined  // 获取订阅类型
      rateLimitTier = getRateLimitTier() ?? undefined        // 获取速率限制等级

      if (subscriptionType && config.claudeCodeFirstTokenDate) {
        const configFirstTokenTime = new Date(
          config.claudeCodeFirstTokenDate,
        ).getTime()
        if (!isNaN(configFirstTokenTime)) {
          firstTokenTime = configFirstTokenTime  // 首次令牌时间
        }
      }
    }

    // 从OAuth获取账户信息（仅Pro/Max用户）
    const oauthAccountInfo = getOauthAccountInfo()
    return {
      // ...
      organizationUuid: oauthAccountInfo?.organizationUuid,
      accountUuid: oauthAccountInfo?.accountUuid,
      subscriptionType,
      rateLimitTier,
      firstTokenTime,
    }
  }
)
```

### 2.2 账户信息存储
**文件**: `src/utils/config.ts` (第161-174行)

Pro/Max 用户的账户信息会持久化存储在 `~/.claude/.claude.json`：

```typescript
export type AccountInfo = {
  accountUuid: string                   // ✓ Pro/Max: OAuth账户UUID
  emailAddress: string                  // ✓ Pro/Max: 用户邮箱
  organizationUuid?: string             // ✓ Pro/Max: 组织UUID（如果有）
  organizationName?: string | null      // ✓ Pro/Max: 组织名称
  organizationRole?: string | null      // ✓ Pro/Max: 组织角色
  workspaceRole?: string | null         // ✓ Pro/Max: 工作区角色
  displayName?: string                  // ✓ Pro/Max: 显示名称
  hasExtraUsageEnabled?: boolean        // ✓ Pro/Max: 是否启用额外使用量
  billingType?: BillingType | null      // ✓ Pro/Max: 计费类型
  accountCreatedAt?: string             // ✓ Pro/Max: 账户创建时间
  subscriptionCreatedAt?: string        // ✓ Pro/Max: 订阅创建时间
}
```

**计费类型枚举** (第176-181行)：
```typescript
export type BillingType =
  | 'stripe_subscription'              // Stripe订阅
  | 'apple_subscription'               // Apple订阅
  | 'google_play_subscription'         // Google Play订阅
  | 'stripe_subscription_contracted'   // Stripe合约订阅
```

### 2.3 OAuth令牌刷新时的数据同步
**文件**: `src/services/oauth/client.ts` (第200-258行)

每次刷新OAuth令牌时，系统会获取并更新 Pro/Max 用户的以下信息：

```typescript
// 检查是否已有完整的 profile 信息
const haveProfileAlready =
  config.oauthAccount?.billingType !== undefined &&
  config.oauthAccount?.accountCreatedAt !== undefined &&
  config.oauthAccount?.subscriptionCreatedAt !== undefined &&
  existing?.subscriptionType != null &&
  existing?.rateLimitTier != null

// 如果缺少信息，则获取 profile
const profileInfo = haveProfileAlready
  ? null
  : await fetchProfileInfo(accessToken)

// 更新存储的属性
if (profileInfo && config.oauthAccount) {
  const updates: Partial<AccountInfo> = {}
  if (profileInfo.displayName !== undefined) {
    updates.displayName = profileInfo.displayName
  }
  if (typeof profileInfo.hasExtraUsageEnabled === 'boolean') {
    updates.hasExtraUsageEnabled = profileInfo.hasExtraUsageEnabled
  }
  if (profileInfo.billingType !== null) {
    updates.billingType = profileInfo.billingType  // ✓ 计费类型
  }
  if (profileInfo.accountCreatedAt !== undefined) {
    updates.accountCreatedAt = profileInfo.accountCreatedAt  // ✓ 账户创建时间
  }
  if (profileInfo.subscriptionCreatedAt !== undefined) {
    updates.subscriptionCreatedAt = profileInfo.subscriptionCreatedAt  // ✓ 订阅创建时间
  }
}

return {
  accessToken,
  refreshToken: newRefreshToken,
  expiresAt,
  scopes,
  subscriptionType:
    profileInfo?.subscriptionType ?? existing?.subscriptionType ?? null,  // ✓ 订阅类型
  rateLimitTier:
    profileInfo?.rateLimitTier ?? existing?.rateLimitTier ?? null,  // ✓ 速率限制等级
}
```

## 3. Pro/Max 用户的功能门控

### 3.1 Guest Pass 功能（仅 Max 用户）
**文件**: `src/services/api/referral.ts` (第71-77行)

Guest Pass（访客通行证）功能**仅对 Max 订阅用户开放**：

```typescript
function shouldCheckForPasses(): boolean {
  return !!(
    getOauthAccountInfo()?.organizationUuid &&
    isClaudeAISubscriber() &&
    getSubscriptionType() === 'max'      // 仅 Max 用户可用
  )
}
```

这意味着：
- ✓ **Max 用户**：可以检查和使用 Guest Pass
- ✗ **Pro 用户**：无法访问此功能
- ✗ **免费用户/API用户**：无法访问此功能

### 3.2 Effort Level 默认值（Pro/Max/Team）
**文件**: `src/components/EffortCallout.tsx` (第237-254行)

不同订阅等级的用户获得不同的默认 effort level：

```typescript
// Pro 用户已经在此 PR 之前有了 medium 默认值
if (isProSubscriber()) {
  if (config.effortCalloutDismissed) {
    markV2Dismissed();
    return false;
  }
  return getOpusDefaultEffortConfig().enabled;
}

// Max/Team 是 tengu_grey_step2 配置的目标
if (isMaxSubscriber() || isTeamSubscriber()) {
  return getOpusDefaultEffortConfig().enabled;
}

// 其他所有人（免费层、API密钥、非订阅者）：不在范围内
markV2Dismissed();
return false;
```

**Effort Level 配置详细**：
**文件**: `src/utils/effort.ts` (第310-315行)

```typescript
if (isProSubscriber()) {
  // Pro 用户获得 medium 默认值
}
// ...
if (isMaxSubscriber() || isTeamSubscriber()) {
  // Max/Team 用户获得更高的默认值
}
```

这意味着：
- ✓ **Max 用户**：获得高级默认 effort level
- ✓ **Pro 用户**：获得中级默认 effort level
- ✓ **Team 用户**：获得高级默认 effort level
- ✗ **免费用户**：不提供默认 effort level

### 3.3 计费访问控制（Pro/Max 自动访问）
**文件**: `src/utils/billing.ts` (第53-78行)

Pro/Max 用户自动获得计费访问权限，而 Team/Enterprise 用户需要特定角色：

```typescript
export function hasClaudeAiBillingAccess(): boolean {
  if (mockBillingAccessOverride !== null) {
    return mockBillingAccessOverride
  }

  if (!isClaudeAISubscriber()) {
    return false
  }

  const subscriptionType = getSubscriptionType()

  // 消费者计划（Max/Pro）- 个人用户总是有计费访问权限
  if (subscriptionType === 'max' || subscriptionType === 'pro') {
    return true  // ✓ Pro/Max 自动访问
  }

  // Team/Enterprise - 检查管理员或计费角色
  const config = getGlobalConfig()
  const orgRole = config.oauthAccount?.organizationRole

  return (
    !!orgRole &&
    ['admin', 'billing', 'owner', 'primary_owner'].includes(orgRole)
  )
}
```

这意味着：
- ✓ **Max 用户**：自动拥有完整计费访问权限
- ✓ **Pro 用户**：自动拥有完整计费访问权限
- ~ **Team/Enterprise 用户**：需要管理员、计费、所有者或主要所有者角色

### 3.4 速率限制等级管理
**文件**: `src/utils/auth.ts` (第1702-1712行)
**文件**: `src/commands/rate-limit-options/rate-limit-options.tsx` (第34-84行)

Pro/Max 用户有不同的速率限制等级：

```typescript
// auth.ts
export function getRateLimitTier(): string | null {
  if (!isAnthropicAuthEnabled()) {
    return null
  }
  const oauthTokens = getClaudeAIOAuthTokens()
  if (!oauthTokens) {
    return null
  }
  return oauthTokens.rateLimitTier ?? null
}

// rate-limit-options.tsx
const isMax = subscriptionType === "max";
const isMax20x = isMax && rateLimitTier === "default_claude_max_20x";

// 根据等级显示不同的选项
if (!isMax20x && !isTeamOrEnterprise && upgrade.isEnabled()) {
  // 显示升级选项
}
```

**已知的速率限制等级**：
- `default_claude_max_20x` - Max 用户的 20x 速率限制
- `default_claude_max_5x` - Team Premium 用户的 5x 速率限制
- 其他等级由系统动态分配

## 4. Pro/Max 用户分析事件中的元数据

### 4.1 事件元数据收集
**文件**: `src/services/analytics/metadata.ts` (第729-731行)

所有发送到分析系统的事件都会包含订阅类型（如果用户已订阅）：

```typescript
const metadata: EventMetadata = {
  // ... 其他元数据
  ...(getSubscriptionType() && {
    subscriptionType: getSubscriptionType()!,  // ✓ 订阅类型
  }),
}
```

### 4.2 发送到 Anthropic API 的完整事件结构
**文件**: `src/services/analytics/metadata.ts` (第796-973行)

Pro/Max 用户的事件包含以下订阅相关字段：

```typescript
// Proto Schema: ClaudeCodeInternalEvent
{
  session_id: string,              // 会话ID
  device_id: string,               // 设备ID
  email?: string,                  // ✓ Pro/Max: 用户邮箱
  account_uuid?: string,           // ✓ Pro/Max: 账户UUID
  organization_uuid?: string,      // ✓ Pro/Max: 组织UUID（如果有）
  subscription_type?: string,      // ✓ Pro/Max: 订阅类型（'max' 或 'pro'）
  rate_limit_tier?: string,        // ✓ Pro/Max: 速率限制等级
  user_type: string,               // 用户类型（ant/external）
  client_type: string,             // 客户端类型
  model: string,                   // 使用的模型
  is_interactive: boolean,         // 是否交互式
  platform: string,                // 操作系统平台
  arch: string,                    // CPU架构
  // ... 其他系统指标
}
```

### 4.3 Datadog 事件标签
**文件**: `src/services/analytics/datadog.ts` (第66-83行)

发送到 Datadog 的事件包含订阅相关标签：

```typescript
// 标签字段
const tagFields = [
  'arch',
  'clientType',
  'model',
  'platform',
  'version',
  'userType',
  'subscriptionType',  // ✓ Pro/Max: 订阅类型标签
  // ...
]
```

## 5. Pro/Max 用户数据收集汇总

### 5.1 Pro/Max 特有的收集字段

| 字段名 | Pro用户 | Max用户 | 说明 | 文件位置 |
|--------|---------|---------|------|----------|
| `subscriptionType` | ✓ | ✓ | 订阅类型（'pro' 或 'max'） | user.ts:34-47 |
| `rateLimitTier` | ✓ | ✓ | 速率限制等级 | user.ts:34-47 |
| `firstTokenTime` | ✓ | ✓ | 首次使用令牌时间戳 | user.ts:34-47 |
| `email` | ✓ | ✓ | 用户邮箱（从OAuth获取） | user.ts:34-47 |
| `accountUuid` | ✓ | ✓ | OAuth账户UUID | user.ts:34-47 |
| `organizationUuid` | ✓ (可选) | ✓ (可选) | 组织UUID（如果在组织中） | user.ts:34-47 |
| `billingType` | ✓ | ✓ | 计费类型（Stripe/Apple/Google Play） | config.ts:161-174 |
| `accountCreatedAt` | ✓ | ✓ | 账户创建时间 | config.ts:161-174 |
| `subscriptionCreatedAt` | ✓ | ✓ | 订阅创建时间 | config.ts:161-174 |
| `hasExtraUsageEnabled` | ✓ | ✓ | 是否启用额外使用量/超额 | config.ts:161-174 |
| `displayName` | ✓ | ✓ | 用户显示名称 | config.ts:161-174 |
| `organizationName` | ✓ (可选) | ✓ (可选) | 组织名称 | config.ts:161-174 |
| `organizationRole` | ✓ (可选) | ✓ (可选) | 组织角色 | config.ts:161-174 |
| `workspaceRole` | ✓ (可选) | ✓ (可选) | 工作区角色 | config.ts:161-174 |

### 5.2 Pro/Max 独有的功能访问

| 功能 | Pro用户 | Max用户 | 说明 | 文件位置 |
|------|---------|---------|------|----------|
| Guest Pass | ✗ | ✓ | 仅Max用户可访问访客通行证功能 | referral.ts:71-77 |
| 高级 Effort Level | ✗ | ✓ | Max用户获得更高的默认effort level | EffortCallout.tsx:237-254 |
| 中级 Effort Level | ✓ | ✗ | Pro用户获得中级默认effort level | EffortCallout.tsx:237-254 |
| 计费访问 | ✓ | ✓ | 两者都自动获得计费访问权限 | billing.ts:53-78 |
| 速率限制提升 | ✓ | ✓ | 两者都有优于免费用户的速率限制 | auth.ts:1702-1712 |
| 20x 速率限制 | ✗ | ✓ | Max用户可能获得20x速率限制等级 | rate-limit-options.tsx:34-84 |

### 5.3 数据发送端点

Pro/Max 用户的数据会发送到以下端点：

1. **Anthropic API 事件日志**
   - 端点：`https://api.anthropic.com/api/event_logging/batch`
   - 频率：每10秒或200个事件批量发送
   - 包含完整的订阅和账户信息

2. **Datadog 日志**
   - 端点：`https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
   - API密钥：`pubbbf48e6d78dae54bceaa4acf463299bf`（公开客户端令牌）
   - 包含订阅类型标签

3. **OAuth 端点**
   - 用于令牌刷新和profile信息获取
   - 定期同步订阅状态和计费信息

## 6. 风控应用场景

### 6.1 订阅验证与功能门控
系统通过以下方式验证 Pro/Max 用户身份并控制功能访问：

1. **OAuth 令牌验证**：每次请求都会验证 OAuth 令牌的有效性
2. **订阅类型检查**：通过 `isProSubscriber()` 和 `isMaxSubscriber()` 函数检查订阅状态
3. **速率限制执行**：根据 `rateLimitTier` 执行不同的速率限制
4. **功能门控**：某些功能（如 Guest Pass）仅对特定订阅类型开放

### 6.2 计费与超额使用监控

**文件**: `src/utils/billing.ts`

系统监控以下计费相关指标：

1. **计费类型** (`billingType`)：
   - Stripe订阅
   - Apple订阅
   - Google Play订阅
   - Stripe合约订阅

2. **超额使用** (`hasExtraUsageEnabled`)：
   - 跟踪用户是否启用了额外使用量
   - 用于防止未授权的超额消费

3. **订阅生命周期**：
   - `accountCreatedAt`：用于新用户分析
   - `subscriptionCreatedAt`：用于订阅留存分析
   - `firstTokenTime`：用于激活/参与度分析

### 6.3 用户分桶与隐私保护

即使对于 Pro/Max 用户，系统仍然实施隐私保护措施：

**文件**: `src/services/analytics/datadog.ts` (第281-299行)

```typescript
// 使用 SHA256 哈希将用户ID映射到30个桶之一
// 允许在不泄露PII的情况下统计用户数量
```

这确保了即使在分析数据中，也不会直接暴露用户身份。

## 7. 数据收集对比：免费用户 vs Pro/Max 用户

| 数据类别 | 免费/API用户 | Pro用户 | Max用户 |
|----------|--------------|---------|---------|
| **设备ID** | ✓ | ✓ | ✓ |
| **会话ID** | ✓ | ✓ | ✓ |
| **平台/架构** | ✓ | ✓ | ✓ |
| **环境检测** | ✓ | ✓ | ✓ |
| **系统指标** | ✓ | ✓ | ✓ |
| **用户邮箱** | ✗ | ✓ | ✓ |
| **账户UUID** | ✗ | ✓ | ✓ |
| **订阅类型** | ✗ | ✓ ('pro') | ✓ ('max') |
| **速率限制等级** | ✗ | ✓ | ✓ (可能更高) |
| **计费类型** | ✗ | ✓ | ✓ |
| **账户创建时间** | ✗ | ✓ | ✓ |
| **订阅创建时间** | ✗ | ✓ | ✓ |
| **首次令牌时间** | ✗ | ✓ | ✓ |
| **超额使用标志** | ✗ | ✓ | ✓ |
| **组织信息** | ✗ | ✓ (可选) | ✓ (可选) |
| **Guest Pass数据** | ✗ | ✗ | ✓ |

## 8. 关键发现总结

### 8.1 Pro 用户特点
1. **认证方式**：通过 OAuth 认证，不使用 API 密钥
2. **数据收集**：收集完整的账户和订阅信息
3. **功能访问**：
   - ✓ 中级默认 effort level
   - ✓ 自动计费访问权限
   - ✓ 优于免费用户的速率限制
   - ✗ 无法访问 Guest Pass
   - ✗ 无法获得最高级别的速率限制（20x）

### 8.2 Max 用户特点
1. **认证方式**：通过 OAuth 认证，不使用 API 密钥
2. **数据收集**：收集完整的账户和订阅信息
3. **功能访问**：
   - ✓ 高级默认 effort level
   - ✓ 自动计费访问权限
   - ✓ Guest Pass 功能
   - ✓ 可能获得 20x 速率限制
   - ✓ 所有 Pro 用户功能

### 8.3 数据用途
Pro/Max 用户的数据主要用于：

1. **订阅管理**：跟踪订阅状态、计费类型、超额使用
2. **功能门控**：根据订阅类型启用/禁用特定功能
3. **速率限制**：根据订阅等级执行不同的速率限制
4. **用户分析**：
   - DAU（日活跃用户）按账户年龄分析
   - 订阅留存率分析
   - 功能使用率按订阅类型分析
   - 激活和参与度分析
5. **风控**：
   - 防止未授权访问付费功能
   - 监控超额使用和计费异常
   - 检测账户共享或滥用

### 8.4 隐私保护措施
即使对于 Pro/Max 付费用户，系统仍然实施以下隐私保护：

1. **用户分桶**：使用 SHA256 哈希避免直接暴露用户ID
2. **PII 标记**：明确标记个人识别信息（邮箱、UUID等）
3. **功能门控**：某些数据收集受功能开关控制
4. **事件采样**：可动态配置采样率以减少数据量
5. **本地存储**：敏感配置仅存储在本地 `~/.claude/.claude.json`

## 9. 代码位置索引

### 核心文件列表

| 功能 | 文件路径 | 关键行号 |
|------|---------|---------|
| 订阅类型检测 | `src/utils/auth.ts` | 1564-1730 |
| 用户数据收集 | `src/utils/user.ts` | 34-128 |
| 账户信息存储 | `src/utils/config.ts` | 161-174 |
| OAuth令牌刷新 | `src/services/oauth/client.ts` | 200-258 |
| Guest Pass（Max-only） | `src/services/api/referral.ts` | 71-77 |
| Effort Level门控 | `src/components/EffortCallout.tsx` | 237-254 |
| 计费访问控制 | `src/utils/billing.ts` | 53-78 |
| 速率限制管理 | `src/utils/auth.ts` | 1702-1712 |
| 分析元数据 | `src/services/analytics/metadata.ts` | 729-731, 796-973 |
| Datadog集成 | `src/services/analytics/datadog.ts` | 66-83, 281-299 |

---

**报告生成时间**: 2026-03-31
**分析版本**: Claude Code v2.1.88
**关注重点**: Pro/Max 订阅用户的数据收集与功能差异
