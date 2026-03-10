# OPECode token context and free provider limits

**Session ID:** ses_38d235e85ffe1MkPax4FW9LkXc
**Created:** 2/18/2026, 4:26:27 PM
**Updated:** 2/18/2026, 4:37:29 PM

---

## User

how does opecode keep token context and does it relate to actual api usage from providor. how does it know user using tokens? what i want to know is there a limit to using the free providers like kimi and big pickle

---

## Assistant (Plan · big-pickle · 255.2s)

_Thinking:_

The user is asking about how OpenCode (the tool I'm running as) handles token context and API usage from providers. They want to understand:

1. How OpenCode keeps token context
2. How it relates to actual API usage from providers
3. How it knows if a user is using tokens
4. Whether there are limits to using free providers like "kimi" and "big pickle"

This is a research question about the codebase. I need to explore the OpenCode codebase to understand how it handles:
- Token tracking/context
- Provider API usage
- Free tier limitations

Let me search the codebase to find relevant code.

I'll explore the codebase to understand how OpenCode handles token context, API usage tracking, and provider limits.

**Tool: task**

**Input:**
```json
{
  "command": "Explore token context and provider API usage",
  "description": "Explore token context handling",
  "prompt": "I need to understand how OpenCode handles token context and API usage. Please search the codebase thoroughly for:\n\n1. How token usage/context is tracked - look for terms like \"token\", \"usage\", \"limit\", \"quota\", \"context\"\n2. How different providers are configured - look for provider configs, especially for \"kimi\" and \"big pickle\" \n3. How API calls are made to providers\n4. Any rate limiting or free tier handling\n\nSearch in these areas:\n- Configuration files\n- Provider-related code\n- API client code\n- Any documentation about providers\n\nReport back:\n- Where token tracking happens\n- How it determines which provider to use\n- Any limits on free providers\n- The file paths and line numbers of relevant code",
  "subagent_type": "explore"
}
```

**Output:**
```
task_id: ses_38d231052ffeQuAfcKqkbyQVmn (for resuming to continue this task if needed)

<task_result>
Based on my thorough search of the codebase, here's a comprehensive report on how OpenCode handles token context and API usage:

---

## Token Tracking and Context Handling

### Where Token Tracking Happens

1. **Token Collection from API Responses** (`/home/farmer/opencodetest/packages/opencode/src/session/index.ts`, lines 682-759)
   - The `getUsage` function extracts token usage from API responses
   - Handles input, output, reasoning, and cache tokens
   - Returns a `tokens` object: `{ total, input, output, reasoning, cache: { read, write } }`
   - Calculates cost based on model pricing from models.dev

2. **Token Overflow Detection** (`/home/farmer/opencodetest/packages/opencode/src/session/compaction.ts`, lines 32-48)
   - `SessionCompaction.isOverflow()` checks if tokens exceed usable context
   - Uses `model.limit.context` and `model.limit.input` from model definitions
   - Has a `COMPACTION_BUFFER` of 20,000 tokens
   - Triggers session compaction when approaching context limits

3. **Token Estimation** (`/home/farmer/opencodetest/packages/opencode/src/util/token.ts`)
   - Used for estimating token counts for pruning decisions

---

## Provider Configuration

### How "kimi" and "big-pickle" Models are Configured

1. **Model Data Source** (`/home/farmer/opencodetest/packages/opencode/src/provider/models.ts`, lines 83-98)
   - Models fetched from `https://models.dev` (configurable via `OPENCODE_MODELS_URL` flag)
   - Fallback to bundled snapshot at build time
   - Cached in `~/.cache/opencode/models.json`

2. **Opencode Provider Special Handling** (`/home/farmer/opencodetest/packages/opencode/src/provider/provider.ts`, lines 128-149)
   ```typescript
   async opencode(input) {
     const hasKey = await (async () => {
       // Check env vars, auth, or config for API key
       // ...
     })()
     
     if (!hasKey) {
       // Remove models with cost.input > 0 (paid models)
       for (const [key, value] of Object.entries(input.models)) {
         if (value.cost.input === 0) continue  // Keep free models
         delete input.models[key]
       }
     }
     
     return {
       autoload: Object.keys(input.models).length > 0,
       options: hasKey ? {} : { apiKey: "public" },
     }
   }
   ```
   - **Key behavior**: If no API key is configured, removes paid models, keeps only free models (those with `cost.input === 0`)

3. **Model Limits Definition** (`/home/farmer/opencodetest/packages/opencode/src/provider/models.ts`, lines 52-56)
   - Each model has `limit: { context: number, input?: number, output: number }`

4. **Priority Sorting** (`/home/farmer/opencodetest/packages/opencode/src/provider/provider.ts`, line 1279)
   ```typescript
   const priority = ["gpt-5", "claude-sonnet-4", "big-pickle", "gemini-3-pro"]
   ```

5. **Default Model Selection** (`/home/farmer/opencodetest/packages/opencode/src/acp/agent.ts`, lines 1522-1547)
   - If available, uses `big-pickle` as the default model
   - Falls back to other opencode models sorted by priority
   - Then falls back to any available provider

---

## API Calls to Providers

1. **Language Model Retrieval** (`/home/farmer/opencodetest/packages/opencode/src/provider/provider.ts`, lines 1171-1196)
   - `getLanguage()` retrieves the SDK and loads the language model

2. **Streaming API Calls** (`/home/farmer/opencodetest/packages/opencode/src/session/llm.ts`, lines 46-238)
   - Uses Vercel AI SDK's `streamText()` function
   - Gets provider via `Provider.getLanguage(input.model)`
   - Passes system prompts, messages, and tools

3. **Model Transformation** (`/home/farmer/opencodetest/packages/opencode/src/provider/transform.ts`)
   - Handles provider-specific settings (temperature, topP, topK)
   - Special handling for kimi models (lines 300-307, 313, 341-343, 712-733)
   - Configures thinking/reasoning for various models

---

## Rate Limiting and Free Tier Handling

### Free Models Provided by OpenCode

1. **Free Model Identification**
   - Models with `cost.input === 0` are free
   - Examples: `kimi-k2.5-free`, `gpt-5-nano` (on opencode provider)

2. **Model Filtering** (`/home/farmer/opencodetest/packages/opencode/src/provider/provider.ts`, lines 138-143)
   - Without API key, only free models (cost.input === 0) are available

### Zen/Console Rate Limiting

1. **IP-based Rate Limiting** (`/home/farmer/opencodetest/packages/console/app/src/routes/zen/util/rateLimiter.ts`)
   - Tracks requests by IP address
   - Limits: configurable per plan (day or hour rolling windows)
   - Throws `FreeUsageLimitError` when exceeded

2. **Black Tier (Zen) Limits** (`/home/farmer/opencodetest/packages/console/core/src/black.ts`)
   - Plans: 200, 100, 20 (dollar limits per week/month)
   - Both fixed weekly and rolling limits

3. **Provider-specific Rate Limits** (`/home/farmer/opencodetest/packages/opencode/src/session/retry.ts`, line 93)
   - Detects `rate_limit` errors from providers
   - Retries with exponential backoff

---

## Summary of Key Files

| File | Purpose |
|------|---------|
| `/home/farmer/opencodetest/packages/opencode/src/provider/models.ts` | Model definitions from models.dev |
| `/home/farmer/opencodetest/packages/opencode/src/provider/provider.ts` | Provider loading, model selection, SDK initialization |
| `/home/farmer/opencodetest/packages/opencode/src/provider/transform.ts` | Provider-specific settings and transformations |
| `/home/farmer/opencodetest/packages/opencode/src/session/index.ts` (lines 682-759) | Token usage extraction from API responses |
| `/home/farmer/opencodetest/packages/opencode/src/session/compaction.ts` | Context overflow detection and pruning |
| `/home/farmer/opencodetest/packages/opencode/src/session/llm.ts` | API call execution via Vercel AI SDK |
| `/home/farmer/opencodetest/packages/opencode/src/acp/agent.ts` (lines 1522-1547) | Default model selection logic |
| `/home/farmer/opencodetest/packages/console/app/src/routes/zen/util/rateLimiter.ts` | Free tier IP-based rate limiting |
</task_result>
```

---

## Assistant (Plan · big-pickle · 12.4s)

_Thinking:_

This is excellent information. Let me summarize the key points for the user in a concise way.

Based on my research, here's how OpenCode handles token context and provider usage:

## Token Context & API Usage

**Token tracking** happens in two main places:
1. `packages/opencode/src/session/index.ts:682-759` - extracts token usage from API responses (input/output/reasoning/cache)
2. `packages/opencode/src/session/compaction.ts:32-48` - monitors context overflow and triggers pruning when approaching limits

The token data comes directly from the **provider's API response** - OpenCode doesn't track usage itself, it just reads what the LLM API returns.

## How It Knows If You're Using Tokens

OpenCode checks for an **API key configuration**. In `packages/opencode/src/provider/provider.ts:128-149`:

```typescript
if (!hasKey) {
  // Remove models with cost.input > 0 (paid models)
  delete input.models[key]
}
```

If no API key is found, it **filters out paid models** and only shows free models (those with `cost.input === 0`).

## Free Provider Limits (kimi, big-pickle)

**OpenCode's free models** (like `kimi-k2.5-free`, `big-pickle`) come through the `opencode` provider. The limits are:

1. **Model-defined limits** - Each model has `limit.context`, `limit.input`, `limit.output` in `/packages/opencode/src/provider/models.ts`
2. **Rate limiting** - The Zen/Console service (`packages/console/app/src/routes/zen/util/rateLimiter.ts`) enforces IP-based rate limits on free tier
3. **Provider rate limits** - `/packages/opencode/src/session/retry.ts:93` handles rate limit errors with retries

The actual limits depend on:
- The specific free model (kimi-k2.5-free, big-pickle, etc.)
- OpenCode's service limits (if using the public API)
- Your API key setup (if you've configured your own keys)

**TL;DR**: You can use free providers, but they're limited by the model's context window and OpenCode's rate limiting on the public endpoint. If you need unlimited usage, you'd need to configure your own API keys.

---

## User

1. Model-defined limits - Each model has limit.context, limit.input, limit.output in /packages/opencode/src/provider/models.ts
2. Rate limiting - The Zen/Console service (packages/console/app/src/routes/zen/util/rateLimiter.ts) enforces IP-based rate limits on free tier
3. Provider rate limits - /packages/opencode/src/session/retry.ts:93 handles rate limit errors with retries is there a way to remove limits for free tier

---

## Assistant (Plan · big-pickle · 206.7s)

_Thinking:_

The user is asking about removing limits for the free tier. They want to know if there's a way to do this. Let me explore the codebase to understand:

1. Where the rate limits are defined
2. How they can be configured or removed
3. What the implications would be

Since I'm in plan mode (read-only), I should search the relevant files to understand the rate limiting mechanism and then present a plan.

Let me explore the rate limiting code to understand how to remove or adjust those limits.

**Tool: task**

**Input:**
```json
{
  "command": "Explore free tier rate limits",
  "description": "Explore rate limit removal",
  "prompt": "I need to understand how to remove or bypass free tier rate limits in OpenCode. Please search thoroughly:\n\n1. Find where rate limits are defined (look for FREE_TIER, rateLimit, FreeUsageLimitError, etc.)\n2. Find configuration options for rate limiting\n3. Look for any environment variables that control limits\n4. Find how the opencode provider determines if it should use free vs paid models\n\nFocus on:\n- packages/console/app/src/routes/zen/util/rateLimiter.ts\n- packages/opencode/src/provider/provider.ts\n- Any config files\n\nReport:\n- Exact file paths and line numbers where limits are set\n- How limits can be configured or disabled\n- Any environment variables that affect this",
  "subagent_type": "explore"
}
```

**Output:**
```
task_id: ses_38d1c6989ffeLDfzGL1hwo2HfN (for resuming to continue this task if needed)

<task_result>
Based on my thorough search, here's a comprehensive report on how rate limits work in OpenCode:

## Summary Report

### 1. Rate Limit Definitions and Locations

#### Primary Rate Limiting Files:
- **`/home/farmer/opencodetest/packages/console/app/src/routes/zen/util/rateLimiter.ts`** (lines 1-84)
  - Line 7-8: `createRateLimiter()` - creates rate limiter based on `ZenData.RateLimit` config
  - Lines 28-42: `check()` function - enforces daily/hourly limits
  - Line 38: Throws `FreeUsageLimitError` when limit is exceeded
  - Lines 46-68: Retry time calculation functions

#### Error Handling:
- **`/home/farmer/opencodetest/packages/console/app/src/routes/zen/util/error.ts`** (lines 7-16)
  - Line 14: `FreeUsageLimitError` class definition

#### Schema Definition:
- **`/home/farmer/opencodetest/packages/console/core/src/model.ts`** (lines 21-29)
  ```typescript
  const RateLimitSchema = z.object({
    period: z.enum(["day", "rolling"]),
    value: z.number().int(),
    checkHeader: z.string().optional(),
    fallbackValue: z.number().int().optional(),
  })
  ```

#### Database Schema:
- **`/home/farmer/opencodetest/packages/console/core/src/schema/ip.sql.ts`** (lines 14-22)
  - IP-based rate limiting stored in `ip_rate_limit` table

---

### 2. Configuration Options for Rate Limiting

The rate limits are configured **per-model** in the model definitions (stored in SST resources `ZEN_MODELS1` through `ZEN_MODEMS30`).

Each model can have an optional `rateLimit` property with:
- `period`: `"day"` or `"rolling"` (3-hour rolling window)
- `value`: Maximum requests allowed in the period
- `checkHeader`: Optional header to check for bypass
- `fallbackValue`: Fallback limit if header check fails

---

### 3. Environment Variables

**No environment variables directly control rate limits.** The limits are:
- Stored in SST secrets (`ZEN_BLACK_LIMITS` - subscription limits)
- Configured per-model in the ZEN_MODELS resources (managed via SST)

However, there are related configurations:
- **`FREE_WORKSPACES`** in handler.ts (line 59-62): Hardcoded workspaces that bypass billing
  ```typescript
  const FREE_WORKSPACES = [
    "wrk_01K46JDFR0E75SG2Q8K172KF3Y", // frank
    "wrk_01K6W1A3VE0KMNVSCQT43BG2SX", // opencode bench
  ]
  ```

---

### 4. How OpenCode Determines Free vs Paid Models

The determination happens in **`/home/farmer/opencodetest/packages/console/app/src/routes/zen/util/handler.ts`**:

#### `validateModel()` (lines 354-367):
- Returns model info including `allowAnonymous` flag

#### `validateBilling()` (lines 517-609):
- **Line 518**: No auth → `"anonymous"` (free tier)
- **Line 519**: User has provider credentials → `"byok"` (bring your own key)
- **Line 520**: User in `FREE_WORKSPACES` → `"free"`
- **Line 521**: Model has `allowAnonymous: true` → `"free"`
- **Lines 524-569**: Subscription billing checks
- **Lines 572-609**: Pay-as-you-go balance checks

---

### 5. How to Bypass or Remove Rate Limits

#### Option 1: Add Workspace to FREE_WORKSPACES
In `/home/farmer/opencodetest/packages/console/app/src/routes/zen/util/handler.ts` (lines 59-62), add your workspace ID to the `FREE_WORKSPACES` array. This bypasses all billing checks.

#### Option 2: Use a Model with `allowAnonymous: true`
In the model definition (ZEN_MODELS), models can have `allowAnonymous: true` which allows unauthenticated (free) access.

#### Option 3: Remove rateLimit from Model Config
Modify the model's rateLimit configuration in the ZEN_MODELS resources to remove or increase the limit.

#### Option 4: Bypass via Header
If a model has `checkHeader` configured, providing the specified header with a valid value can use `fallbackValue` instead.

#### Option 5: Client-side in OpenCode SDK
In `/home/farmer/opencodetest/packages/opencode/src/session/retry.ts` (line 66-67):
```typescript
if (error.data.responseBody?.includes("FreeUsageLimitError"))
  return `Free usage exceeded, add credits https://opencode.ai/zen`
```
This just shows a message - the actual enforcement is server-side.

---

### Additional Rate Limiting Mechanisms

1. **Trial Limiter** (`/home/farmer/opencodetest/packages/console/app/src/routes/zen/util/trialLimiter.ts`):
   - Tracks IP-based usage for trial models
   - Limits based on tokens used

2. **Subscription Limits** (`/home/farmer/opencodetest/packages/console/core/src/black.ts`):
   - Plan-based limits: `$20`, `$100`, `$200` plans
   - Fixed weekly limits and rolling hourly limits
</task_result>
```

---

## Assistant (Plan · big-pickle · 9.3s)

_Thinking:_

Now I have a comprehensive understanding of how to remove/bypass the free tier limits. Let me present this as a plan to the user.

Based on my research, here's how to remove or bypass free tier limits:

## Options to Remove Free Tier Limits

### Option 1: Add Workspace to FREE_WORKSPACES (Easiest)
**File**: `packages/console/app/src/routes/zen/util/handler.ts:59-62`

Add your workspace ID to the `FREE_WORKSPACES` array - this completely bypasses billing checks:
```typescript
const FREE_WORKSPACES = [
  "wrk_01K46JDFR0E75SG2Q8K172KF3Y", // frank
  "wrk_01K6W1A3VE0KMNVSCQT43BG2SX", // opencode bench
  // add your workspace ID here
]
```

### Option 2: Set `allowAnonymous: true` on Model
In the ZEN_MODELS SST resources, configure the model to allow anonymous (free) access without rate limits.

### Option 3: Modify Rate Limit Config
Change or remove the `rateLimit` property in the model configuration (stored in SST secrets).

### Option 4: Use Your Own API Key (BYOK)
Configure your own API key in OpenCode - this puts you in "byok" mode and bypasses all rate limits since you're paying the provider directly.

---

## Recommendation

**Option 1** is the simplest - just add a workspace ID to the hardcoded array. 

**Option 4** (bring your own key) is the most sustainable if you want unlimited usage without modifying server code.

Would you like me to implement Option 1 (adding a workspace to FREE_WORKSPACES), or would you prefer to configure your own API key (Option 4)? If you want Option 1, what's your workspace ID?

---

