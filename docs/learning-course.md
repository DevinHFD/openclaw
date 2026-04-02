# OpenClaw Codebase Learning Course

A hands-on guide to understanding and contributing to the OpenClaw codebase.
Each module builds on the previous one with real code examples drawn from the repo.

---

## Module 1 — Project Orientation

### 1.1 What is OpenClaw?

OpenClaw is a self-hosted AI gateway that connects large-language models to
messaging channels (Telegram, Discord, Slack, IRC, WhatsApp, iMessage, and
many more). It is built on a **plugin-first** architecture: almost every
capability — providers, channels, tools, speech — is implemented as a plugin.

### 1.2 Top-level directory map

```
openclaw/
├── src/                  # Core runtime (not imported by plugins)
│   ├── cli/              # CLI command wiring (Commander.js)
│   ├── commands/         # Command implementations
│   ├── channels/         # Channel plugin infrastructure
│   ├── gateway/          # WebSocket/HTTP gateway server
│   ├── plugins/          # Plugin discovery, loading, registry
│   ├── plugin-sdk/       # PUBLIC contract for plugin authors
│   ├── agents/           # Model catalog, auth profiles, tools
│   ├── config/           # Config I/O and validation
│   ├── infra/            # Secrets, outbound delivery, dedup
│   ├── routing/          # Session/message routing
│   ├── auto-reply/       # Reply generation, chunking, templates
│   ├── media/            # Image/audio pipeline
│   └── tts/              # Text-to-speech providers
├── extensions/           # Bundled plugins (90+ packages)
│   ├── anthropic/        # Anthropic provider plugin
│   ├── telegram/         # Telegram channel plugin
│   ├── discord/          # Discord channel plugin
│   ├── irc/              # IRC channel plugin
│   └── ...
├── apps/                 # iOS / Android / macOS apps
├── docs/                 # Mintlify documentation
├── packages/             # Shared utility packages
├── scripts/              # Build and dev scripts
└── ui/                   # Web UI
```

### 1.3 Key rule: the import boundary

```
Plugin code  →  openclaw/plugin-sdk/*   (allowed)
Plugin code  →  src/**                  (FORBIDDEN)
Plugin code  →  another plugin's src/** (FORBIDDEN)
Core code    →  plugin's src/**         (FORBIDDEN)
```

Plugins cross into core only through `openclaw/plugin-sdk/*` subpaths,
manifest metadata, and documented runtime helpers.

### 1.4 Dev setup

```bash
# Install dependencies
pnpm install

# Type-check
pnpm tsgo

# Lint + format check
pnpm check

# Run all tests
pnpm test

# Run a scoped test
pnpm test -- extensions/irc/src/channel.test.ts

# Run the CLI in dev mode
pnpm openclaw channels status
```

---

## Module 2 — Plugin Fundamentals

### 2.1 Plugin manifest (`openclaw.plugin.json`)

Every plugin ships a manifest that declares its identity before any code runs.
The loader reads it to register capabilities, auth env-vars, and config schema.

**Minimal manifest (tool plugin):**

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

**Provider plugin manifest** (`extensions/anthropic/openclaw.plugin.json`):

```json
{
  "id": "anthropic",
  "enabledByDefault": true,
  "providers": ["anthropic"],
  "cliBackends": ["claude-cli"],
  "providerAuthEnvVars": {
    "anthropic": ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "anthropic",
      "method": "api-key",
      "choiceId": "apiKey",
      "choiceLabel": "Anthropic API key",
      "groupId": "anthropic",
      "groupLabel": "Anthropic",
      "optionKey": "anthropicApiKey",
      "cliFlag": "--anthropic-api-key"
    }
  ],
  "contracts": {
    "mediaUnderstandingProviders": ["anthropic"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

**Channel plugin manifest** (`extensions/irc/openclaw.plugin.json`):

```json
{
  "id": "irc",
  "channels": ["irc"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

**Channel plugin `package.json` `openclaw` block** (`extensions/telegram/package.json`):

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "telegram",
      "label": "Telegram",
      "selectionLabel": "Telegram (Bot API)",
      "docsPath": "/channels/telegram",
      "markdownCapable": true
    },
    "bundle": {
      "stageRuntimeDependencies": true
    }
  }
}
```

### 2.2 Plugin entry point: `definePluginEntry`

Non-channel plugins (providers, tools, hooks) export a default value created
by `definePluginEntry` from `openclaw/plugin-sdk/plugin-entry`.

```typescript
// index.ts — minimal tool plugin
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { Type } from "@sinclair/typebox";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Adds a custom tool to OpenClaw",
  register(api) {
    // api is OpenClawPluginApi — the only surface plugins touch
    api.registerTool({
      name: "echo",
      description: "Echo a message back",
      parameters: Type.Object({ message: Type.String() }),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: `Echo: ${params.message}` }],
        };
      },
    });
  },
});
```

The `register(api)` callback receives `OpenClawPluginApi`. Everything the
plugin needs to register must go through this object.

### 2.3 `OpenClawPluginApi` registration surface

From `docs/plugins/building-plugins.md`:

| Capability | Registration method |
|---|---|
| Text inference (LLM) | `api.registerProvider(...)` |
| CLI inference backend | `api.registerCliBackend(...)` |
| Channel / messaging | `api.registerChannel(...)` |
| Speech (TTS/STT) | `api.registerSpeechProvider(...)` |
| Media understanding | `api.registerMediaUnderstandingProvider(...)` |
| Image generation | `api.registerImageGenerationProvider(...)` |
| Web search | `api.registerWebSearchProvider(...)` |
| Agent tools | `api.registerTool(...)` |
| Custom commands | `api.registerCommand(...)` |
| Event hooks | `api.registerHook(...)` |
| HTTP routes | `api.registerHttpRoute(...)` |
| CLI subcommands | `api.registerCli(...)` |

### 2.4 Plugin config schema

Plugins declare a config schema that OpenClaw validates at startup.
The schema can use Zod-style `safeParse`, a lightweight `validate` function,
or raw JSON schema — or all three:

```typescript
// src/plugins/types.ts (simplified)
type OpenClawPluginConfigSchema = {
  safeParse?: (value: unknown) => {
    success: boolean;
    data?: unknown;
    error?: { issues?: Array<{ path: Array<string | number>; message: string }> };
  };
  parse?: (value: unknown) => unknown;
  validate?: (value: unknown) => { ok: true } | { ok: false; errors: string[] };
  uiHints?: Record<string, { label?: string; help?: string; sensitive?: boolean }>;
  jsonSchema?: Record<string, unknown>;
};
```

### 2.5 Tool context

When the LLM invokes a tool, OpenClaw passes a trusted context to the factory
function — never to the tool itself. This allows tools to behave differently
based on which channel or agent is active:

```typescript
// src/plugins/types.ts (simplified)
type OpenClawPluginToolContext = {
  config?: OpenClawConfig;
  runtimeConfig?: OpenClawConfig;  // resolved runtime snapshot
  workspaceDir?: string;
  agentDir?: string;
  agentId?: string;
  sessionKey?: string;
  sessionId?: string;   // ephemeral per-conversation UUID
  browser?: { sandboxBridgeUrl?: string };
  messageChannel?: string;
  agentAccountId?: string;
  deliveryContext?: DeliveryContext;
  requesterSenderId?: string;
  senderIsOwner?: boolean;
  sandboxed?: boolean;
};

type OpenClawPluginToolFactory = (
  ctx: OpenClawPluginToolContext,
) => AnyAgentTool | AnyAgentTool[] | null | undefined;
```

Using a factory instead of a static list lets plugins conditionally expose
tools. Return `null` or an empty array to hide all tools for that context.

### 2.6 Event hooks

Hooks let plugins intercept lifecycle events. Hook semantics:

- `before_tool_call` — `{ block: true }` stops execution; `{ requireApproval: true }` pauses for user approval.
- `message_sending` — `{ cancel: true }` stops delivery.
- `before_install` — `{ block: true }` aborts installation.

```typescript
export default definePluginEntry({
  id: "audit-plugin",
  name: "Audit",
  description: "Log every tool call",
  register(api) {
    api.registerHook({
      event: "before_tool_call",
      priority: 50,
      async handle({ toolName, params }) {
        console.log(`Tool called: ${toolName}`, params);
        return { block: false };
      },
    });
  },
});
```

### 2.7 Plugin logger

The `PluginLogger` type is passed into registration, services, and CLI
surfaces. Always prefer it over `console.*` in plugin code:

```typescript
// src/plugins/types.ts
type PluginLogger = {
  debug?: (message: string) => void;
  info: (message: string) => void;
  warn: (message: string) => void;
  error: (message: string) => void;
};
```

---

## Module 3 — Channel Plugins

### 3.1 The `ChannelPlugin` contract

A channel plugin is a plain TypeScript object that satisfies `ChannelPlugin<ResolvedAccount, Probe, Audit>`.
Every field is optional except `id`, `meta`, `capabilities`, and `config`.

```typescript
// src/channels/plugins/types.plugin.ts (simplified)
type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  id: ChannelId;
  meta: ChannelMeta;                      // display label, docs link, icons
  capabilities: ChannelCapabilities;      // feature matrix
  config: ChannelConfigAdapter<ResolvedAccount>;
  configSchema?: ChannelConfigSchema;     // JSON schema + runtime parse
  setupWizard?: ChannelSetupWizard;
  setup?: ChannelSetupAdapter;            // initial token/key storage
  pairing?: ChannelPairingAdapter;        // approve new senders
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;           // group-chat policies
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;      // sending messages
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>; // long-running listener
  auth?: ChannelAuthAdapter;
  elevated?: ChannelElevatedAdapter;
  commands?: ChannelCommandAdapter;
  lifecycle?: ChannelLifecycleAdapter;
  approvals?: ChannelApprovalAdapter;
  allowlist?: ChannelAllowlistAdapter;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;
  directory?: ChannelDirectoryAdapter;
  resolver?: ChannelResolverAdapter;
  actions?: ChannelMessageActionAdapter;
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

### 3.2 `ChannelCapabilities`

Capabilities tell the core what the channel can do. The `chatTypes` field is
required; everything else defaults to `false`/`undefined`.

```typescript
type ChannelCapabilities = {
  chatTypes: Array<"direct" | "group">;
  media?: boolean;          // can send/receive images, audio, etc.
  blockStreaming?: boolean; // must buffer full reply before sending
  polls?: boolean;
  reactions?: boolean;
  threads?: boolean;
  // ...
};
```

### 3.3 `createChatChannelPlugin` helper

Rather than assembling a raw `ChannelPlugin` object, channel authors use the
`createChatChannelPlugin` helper from `openclaw/plugin-sdk/core`. It fills in
sensible defaults for pairing, security, and outbound so you only implement
what is unique to your channel.

From `extensions/irc/src/channel.ts`:

```typescript
import { createChatChannelPlugin } from "openclaw/plugin-sdk/core";

export const ircPlugin: ChannelPlugin<ResolvedIrcAccount, IrcProbe> =
  createChatChannelPlugin({
    base: {
      id: "irc",
      meta: {
        ...getChatChannelMeta("irc"),
        quickstartAllowFrom: true,
      },
      setup: ircSetupAdapter,
      setupWizard: ircSetupWizard,
      capabilities: {
        chatTypes: ["direct", "group"],
        media: true,
        blockStreaming: true,
      },
      reload: { configPrefixes: ["channels.irc"] },
      configSchema: IrcChannelConfigSchema,
      config: {
        ...ircConfigAdapter,
        isConfigured: (account) => account.configured,
        describeAccount: (account) =>
          describeAccountSnapshot({
            account,
            configured: account.configured,
            extra: {
              host: account.host,
              port: account.port,
              tls: account.tls,
              nick: account.nick,
            },
          }),
      },
      groups: {
        resolveRequireMention: ({ cfg, accountId, groupId }) => {
          const account = resolveIrcAccount({ cfg, accountId });
          if (!groupId) return true;
          const match = resolveIrcGroupMatch({ groups: account.config.groups, target: groupId });
          return resolveIrcRequireMention({
            groupConfig: match.groupConfig,
            wildcardConfig: match.wildcardConfig,
          });
        },
      },
      messaging: {
        normalizeTarget: normalizeIrcMessagingTarget,
        targetResolver: {
          looksLikeId: looksLikeIrcTargetId,
          hint: "<#channel|nick>",
        },
      },
      status: createComputedAccountStatusAdapter({
        defaultRuntime: createDefaultChannelRuntimeState(DEFAULT_ACCOUNT_ID),
        buildChannelSummary: ({ account, snapshot }) => ({
          ...buildBaseChannelStatusSummary(snapshot),
          host: account.host,
          port: snapshot.port,
          tls: account.tls,
        }),
        probeAccount: async ({ cfg, account, timeoutMs }) =>
          probeIrc(cfg, { accountId: account.accountId, timeoutMs }),
        resolveAccountSnapshot: ({ account }) => ({
          accountId: account.accountId,
          name: account.name,
          enabled: account.enabled,
          configured: account.configured,
          extra: { host: account.host, tls: account.tls, nick: account.nick },
        }),
      }),
      gateway: {
        startAccount: async (ctx) => {
          // Long-running listener — runs for the account lifetime
          await runStoppablePassiveMonitor({
            abortSignal: ctx.abortSignal,
            start: async () =>
              await monitorIrcProvider({
                accountId: ctx.accountId,
                config: ctx.cfg,
                runtime: ctx.runtime,
                abortSignal: ctx.abortSignal,
                statusSink: createAccountStatusSink({
                  accountId: ctx.accountId,
                  setStatus: ctx.setStatus,
                }),
              }),
          });
        },
      },
    },
    pairing: {
      // Inline-text pairing: ask the user to paste their IRC nick
      text: {
        idLabel: "ircUser",
        message: PAIRING_APPROVED_MESSAGE,
      },
    },
  });
```

### 3.4 Config adapter: `createScopedChannelConfigAdapter`

OpenClaw supports multiple accounts per channel (e.g., two IRC bots).
`createScopedChannelConfigAdapter` wires config reads/writes to the correct
config section and handles multi-account logic automatically.

```typescript
import { createScopedChannelConfigAdapter } from "openclaw/plugin-sdk/channel-config-helpers";

const ircConfigAdapter = createScopedChannelConfigAdapter<
  ResolvedIrcAccount,     // what resolveAccount returns
  ResolvedIrcAccount,     // what the config adapter exposes
  CoreConfig              // channel-specific config shape
>({
  sectionKey: "irc",             // config key: channels.irc
  listAccountIds: listIrcAccountIds,
  resolveAccount: adaptScopedAccountAccessor(resolveIrcAccount),
  defaultAccountId: resolveDefaultIrcAccountId,
  // fields to clear on "disconnect" / reset
  clearBaseFields: ["name", "host", "port", "tls", "nick", "username", "channels"],
  resolveAllowFrom: (account) => account.config.allowFrom,
  resolveDefaultTo: (account) => account.config.defaultTo,
});
```

### 3.5 Security: DM policy and allowlist

Every chat channel should enforce who is allowed to message the bot. The
helpers `createScopedDmSecurityResolver` and `createAllowlistProviderOpenWarningCollector`
handle this pattern:

```typescript
import {
  createScopedDmSecurityResolver,
} from "openclaw/plugin-sdk/channel-config-helpers";
import {
  createAllowlistProviderOpenWarningCollector,
  composeAccountWarningCollectors,
} from "openclaw/plugin-sdk/channel-policy";

// DM policy: who can DM the bot?
const resolveIrcDmPolicy = createScopedDmSecurityResolver<ResolvedIrcAccount>({
  channelKey: "irc",
  resolvePolicy: (account) => account.config.dmPolicy,
  resolveAllowFrom: (account) => account.config.allowFrom,
  policyPathSuffix: "dmPolicy",
  normalizeEntry: (raw) => normalizeIrcAllowEntry(raw),
});

// Collect warnings when group policy is too open
const collectGroupPolicyWarnings =
  createAllowlistProviderOpenWarningCollector<ResolvedIrcAccount>({
    providerConfigPresent: (cfg) => cfg.channels?.irc !== undefined,
    resolveGroupPolicy: (account) => account.config.groupPolicy,
    buildOpenWarning: {
      surface: "IRC channels",
      openBehavior: "allows all channels and senders (mention-gated)",
      remediation: 'Prefer channels.irc.groupPolicy="allowlist" with channels.irc.groups',
    },
  });

// Compose multiple warning sources
const collectIrcSecurityWarnings = composeAccountWarningCollectors<ResolvedIrcAccount, ...>(
  collectGroupPolicyWarnings,
  (account) =>
    !account.config.tls &&
    "- IRC TLS is disabled; traffic and credentials are plaintext.",
);
```

### 3.6 Directory: peers and groups

Channels can expose peer (user) and group lists for the agent `directory`
tool. Use `createChannelDirectoryAdapter` and `createResolvedDirectoryEntriesLister`:

```typescript
import {
  createChannelDirectoryAdapter,
  createResolvedDirectoryEntriesLister,
} from "openclaw/plugin-sdk/directory-runtime";

const listPeers = createResolvedDirectoryEntriesLister<ResolvedIrcAccount>({
  kind: "user",
  resolveAccount: adaptScopedAccountAccessor(resolveIrcAccount),
  resolveSources: (account) => [
    account.config.allowFrom ?? [],
    account.config.groupAllowFrom ?? [],
  ],
  normalizeId: (entry) => normalizePairingTarget(entry) || null,
});

const listGroups = createResolvedDirectoryEntriesLister<ResolvedIrcAccount>({
  kind: "group",
  resolveAccount: adaptScopedAccountAccessor(resolveIrcAccount),
  resolveSources: (account) => [account.config.channels ?? []],
  normalizeId: (entry) => {
    const normalized = normalizeIrcMessagingTarget(entry);
    return normalized && isChannelTarget(normalized) ? normalized : null;
  },
});

// Wire into the plugin:
directory: createChannelDirectoryAdapter({
  listPeers: async (params) => listPeers(params),
  listGroups: async (params) => {
    const entries = await listGroups(params);
    return entries.map((e) => ({ ...e, name: e.id }));
  },
}),
```

### 3.7 Channel status and probing

Status adapters let `openclaw channels status` report health.
`createComputedAccountStatusAdapter` handles the polling loop; you provide
`probeAccount` and `resolveAccountSnapshot`:

```typescript
import {
  createComputedAccountStatusAdapter,
  createDefaultChannelRuntimeState,
} from "openclaw/plugin-sdk/status-helpers";

status: createComputedAccountStatusAdapter<ResolvedIrcAccount, IrcProbe>({
  defaultRuntime: createDefaultChannelRuntimeState("default"),
  buildChannelSummary: ({ account, snapshot }) => ({
    host: account.host,
    port: snapshot.port,
    tls: account.tls,
    nick: account.nick,
    probe: snapshot.probe,
    lastProbeAt: snapshot.lastProbeAt ?? null,
  }),
  probeAccount: async ({ cfg, account, timeoutMs }) =>
    probeIrc(cfg, { accountId: account.accountId, timeoutMs }),
  resolveAccountSnapshot: ({ account }) => ({
    accountId: account.accountId,
    name: account.name,
    enabled: account.enabled,
    configured: account.configured,
    extra: { host: account.host, nick: account.nick },
  }),
}),
```

### 3.8 Channel plugin entry point

A channel plugin uses `defineChannelPluginEntry` (not `definePluginEntry`):

```typescript
// extensions/telegram/index.ts pattern
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default defineChannelPluginEntry({
  id: "telegram",
  name: "Telegram",
  description: "Telegram channel plugin",
  plugin: telegramPlugin,       // ChannelPlugin object
  setRuntime: setTelegramRuntime, // called once with trusted runtime
});
```

### 3.9 Agent tools from a channel

Channels can register tools only available when that channel is active
(e.g., a "send_telegram_photo" tool):

```typescript
agentTools: (params) => {
  if (!params.cfg?.channels?.irc) return [];
  return [
    {
      name: "irc_whois",
      description: "Look up an IRC user",
      ownerOnly: true,   // only bot owners can call this
      parameters: Type.Object({ nick: Type.String() }),
      async execute(_id, { nick }) {
        // ... call IRC WHOIS
        return { content: [{ type: "text", text: `WHOIS: ${nick}` }] };
      },
    },
  ];
},
```

---

## Module 4 — Provider Plugins

### 4.1 What is a provider plugin?

A provider plugin registers one or more **model providers** with the core
inference loop. The core handles streaming, tool dispatch, and context
management; the provider plugin owns:

- Auth methods (API key, OAuth, token, device code)
- Model catalog (which models are available and their capabilities)
- Runtime auth resolution (producing a live API key or token per call)
- Forward-compatibility helpers (aliasing new model IDs to template models)
- Usage tracking and cache TTL eligibility

### 4.2 `definePluginEntry` + `api.registerProvider`

The Anthropic plugin is the canonical reference (`extensions/anthropic/index.ts`):

```typescript
import {
  definePluginEntry,
  type ProviderAuthContext,
  type ProviderResolveDynamicModelContext,
  type ProviderRuntimeModel,
} from "openclaw/plugin-sdk/plugin-entry";
import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth-api-key";
import { cloneFirstTemplateModel } from "openclaw/plugin-sdk/provider-model-shared";
import { fetchClaudeUsage } from "openclaw/plugin-sdk/provider-usage";

const PROVIDER_ID = "anthropic";

export default definePluginEntry({
  id: PROVIDER_ID,
  name: "Anthropic Provider",
  description: "Bundled Anthropic provider plugin",
  register(api) {
    api.registerProvider({
      id: PROVIDER_ID,
      label: "Anthropic",
      docsPath: "/providers/models",
      envVars: ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"],

      // ── Auth methods ───────────────────────────────────────────────
      auth: [
        // Token-based auth (setup-token from `claude setup-token`)
        {
          id: "setup-token",
          label: "setup-token (claude)",
          hint: "Paste a setup-token from `claude setup-token`",
          kind: "token",
          run: async (ctx: ProviderAuthContext) => runAnthropicSetupToken(ctx),
        },
        // API key auth — use the shared factory helper
        createProviderApiKeyAuthMethod({
          providerId: PROVIDER_ID,
          methodId: "api-key",
          label: "Anthropic API key",
          hint: "Direct Anthropic API key",
          optionKey: "anthropicApiKey",
          flagName: "--anthropic-api-key",
          envVar: "ANTHROPIC_API_KEY",
          promptMessage: "Enter Anthropic API key",
          defaultModel: "anthropic/claude-sonnet-4-6",
          expectedProviders: ["anthropic"],
        }),
      ],

      // ── Model forward-compatibility ────────────────────────────────
      // Lets users specify "claude-sonnet-4-6" (a future model) and
      // resolve it to the best available template model at runtime.
      resolveDynamicModel: (ctx: ProviderResolveDynamicModelContext) =>
        resolveAnthropicForwardCompatModel(ctx),

      // ── Provider capabilities ──────────────────────────────────────
      capabilities: {
        providerFamily: "anthropic",
        dropThinkingBlockModelHints: ["claude"],
      },
      isModernModelRef: ({ modelId }) => matchesAnthropicModernModel(modelId),

      // ── Thinking level defaults ────────────────────────────────────
      resolveDefaultThinkingLevel: ({ modelId }) =>
        matchesAnthropicModernModel(modelId) ? "adaptive" : undefined,

      // ── Usage tracking ─────────────────────────────────────────────
      resolveUsageAuth: async (ctx) => await ctx.resolveOAuthToken(),
      fetchUsageSnapshot: async (ctx) =>
        await fetchClaudeUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn),

      // ── Cache TTL ──────────────────────────────────────────────────
      isCacheTtlEligible: () => true,
    });
  },
});
```

### 4.3 Auth methods in depth

Each auth method has an `id`, a `kind`, and a `run` function that returns
`ProviderAuthResult`:

```typescript
// src/plugins/types.ts (simplified)
type ProviderAuthResult = {
  profiles: Array<{
    profileId: string;
    credential: AuthProfileCredential;  // api_key | oauth | token | device_code
  }>;
  configPatch?: Partial<OpenClawConfig>; // e.g. set default model after login
  defaultModel?: string;
  notes?: string[];
};

type ProviderAuthContext = {
  config: OpenClawConfig;
  env?: NodeJS.ProcessEnv;
  prompter: WizardPrompter;   // interactive CLI prompting
  runtime: RuntimeEnv;
  opts?: ProviderAuthOptionBag;
  secretInputMode?: SecretInputMode;
  allowSecretRefPrompt?: boolean;
  isRemote: boolean;
  openUrl: (url: string) => Promise<void>;
  oauth: { createVpsAwareHandlers: typeof createVpsAwareOAuthHandlers };
};
```

**Implementing a token auth method** (the Anthropic setup-token example):

```typescript
async function runAnthropicSetupToken(
  ctx: ProviderAuthContext,
): Promise<ProviderAuthResult> {
  // Prompt for the token value (with optional SecretRef support)
  const tokenRaw = await ctx.prompter.text({
    message: "Paste Anthropic setup-token",
    validate: (value) => validateAnthropicSetupToken(String(value ?? "")),
  });
  const token = String(tokenRaw ?? "").trim();

  // Optionally prompt for a profile name
  const profileNameRaw = await ctx.prompter.text({
    message: "Token name (blank = default)",
    placeholder: "default",
  });

  return {
    profiles: [
      {
        profileId: buildTokenProfileId({
          provider: "anthropic",
          name: String(profileNameRaw ?? ""),
        }),
        credential: {
          type: "token",
          provider: "anthropic",
          token,
        },
      },
    ],
  };
}
```

### 4.4 Model catalog

The catalog hook tells OpenClaw which models this provider offers. It runs
at startup and on cache miss.

```typescript
// ProviderCatalogContext gives you auth resolution helpers
type ProviderCatalogContext = {
  config: OpenClawConfig;
  env: NodeJS.ProcessEnv;
  resolveProviderApiKey: (providerId?: string) => { apiKey?: string };
  resolveProviderAuth: (providerId?: string) => {
    apiKey?: string;
    mode: "api_key" | "oauth" | "token" | "none";
    source: "env" | "profile" | "none";
    profileId?: string;
  };
};

// Return value
type ProviderCatalogResult =
  | { provider: ModelProviderConfig }
  | { providers: Record<string, ModelProviderConfig> }
  | null
  | undefined;
```

### 4.5 Dynamic model resolution

When a user types `claude-sonnet-4-6` but only `claude-sonnet-4-5` is known
to the catalog, `resolveDynamicModel` bridges the gap.
The helper `cloneFirstTemplateModel` copies capabilities from a known model:

```typescript
import { cloneFirstTemplateModel } from "openclaw/plugin-sdk/provider-model-shared";

function resolveAnthropicForwardCompatModel(
  ctx: ProviderResolveDynamicModelContext,
): ProviderRuntimeModel | undefined {
  const lower = ctx.modelId.trim().toLowerCase();
  const is46 = lower.startsWith("claude-sonnet-4-6");
  if (!is46) return undefined;

  // Clone capabilities from the nearest known template
  return cloneFirstTemplateModel({
    providerId: "anthropic",
    modelId: ctx.modelId.trim(),
    templateIds: ["claude-sonnet-4-5", "claude-sonnet-4.5"],
    ctx,
  });
}
```

### 4.6 Media understanding provider

Providers that support vision can also register a `MediaUnderstandingProvider`
alongside the LLM registration:

```typescript
api.registerMediaUnderstandingProvider(anthropicMediaUnderstandingProvider);
```

`anthropicMediaUnderstandingProvider` implements the `MediaUnderstandingProvider`
interface from `openclaw/plugin-sdk/provider-entry` and handles converting
image buffers into the provider's message format.

### 4.7 Auth profiles

Credentials are stored in named *auth profiles*. A profile stores the
credential type plus metadata (provider, email, expiry, etc.):

```typescript
// Credential types (src/agents/auth-profiles/types.ts, simplified)
type ApiKeyCredential = { type: "api_key"; provider: string; apiKey: string };
type OAuthCredential  = { type: "oauth";   provider: string; accessToken: string; refreshToken?: string };
type TokenCredential  = { type: "token";   provider: string; token: string };
```

SecretRef lets credentials live in env vars, files, or exec commands instead
of being stored in the config file directly:

```typescript
// SecretRef — stored in config, resolved at runtime
type SecretRef =
  | { source: "env";  provider: string; id: string }
  | { source: "file"; provider: string; id: string }
  | { source: "exec"; provider: string; id: string };
```

Use `isSecretRef` from `openclaw/plugin-sdk/core` to check at runtime:

```typescript
import { isSecretRef } from "openclaw/plugin-sdk/core";

if (isSecretRef(credential.tokenRef)) {
  // token is stored externally — resolve via runtime
}
```

### 4.8 Adding a minimal new provider

Checklist for a new provider plugin:

1. Create `extensions/<provider>/` with `package.json`, `openclaw.plugin.json`, `index.ts`.
2. Declare `"providers": ["<provider>"]` in `openclaw.plugin.json`.
3. Declare auth env vars in `"providerAuthEnvVars"`.
4. Call `definePluginEntry` → `api.registerProvider({ id, label, auth, ... })`.
5. Implement at least one auth method (use `createProviderApiKeyAuthMethod` for API key).
6. Keep provider-specific deps in the extension `dependencies` (not root `package.json`).
7. Run `pnpm check` and `pnpm test` before pushing.

---

## Module 5 — Gateway Protocol

### 5.1 What is the gateway?

The **gateway** is a long-running process that:
- Accepts WebSocket connections from the desktop/mobile apps and web UI.
- Routes messages to the correct agent session.
- Manages channel lifecycle (starting/stopping channel listeners).
- Exposes a typed HTTP API for config, channel pairing, and status.

### 5.2 Protocol schema

All messages on the wire are typed through `src/gateway/protocol/schema.ts`,
which re-exports from domain-specific files:

```
src/gateway/protocol/schema/
├── agent.ts           # agent session events
├── agents-models-skills.ts
├── channels.ts        # channel config, TTS, voice
├── config.ts          # config read/write
├── cron.ts            # scheduled jobs
├── error-codes.ts     # closed error code union
├── exec-approvals.ts  # tool approval flow
├── devices.ts         # device pairing
├── frames.ts          # WebSocket frame envelope
├── logs-chat.ts       # chat log streaming
├── nodes.ts           # multi-node gateway
├── protocol-schemas.ts
├── push.ts            # push notifications
├── secrets.ts         # secret management
├── sessions.ts        # session CRUD
├── snapshot.ts        # status snapshots
├── types.ts           # shared primitives
├── plugin-approvals.ts
└── wizard.ts          # onboarding wizard frames
```

### 5.3 TypeBox schemas

The protocol uses `@sinclair/typebox` for runtime-validated, TypeScript-typed
schemas. Never use raw `any` or `string` to branch on protocol values — use
the typed schema objects:

```typescript
import { Type } from "@sinclair/typebox";
import { NonEmptyString, SecretInputSchema } from "./primitives.js";

// Example from src/gateway/protocol/schema/channels.ts
export const TalkSpeakParamsSchema = Type.Object(
  {
    text: NonEmptyString,
    voiceId: Type.Optional(Type.String()),
    modelId: Type.Optional(Type.String()),
    outputFormat: Type.Optional(Type.String()),
    speed: Type.Optional(Type.Number()),
    stability: Type.Optional(Type.Number()),
  },
  { additionalProperties: false },
);
```

### 5.4 Protocol change rules

- Prefer **additive** changes: new optional fields, new message types.
- Breaking changes require: version bump, docs update, client/codegen follow-through.
- Add new schema fields as `Type.Optional(...)` to stay backward-compatible.
- Do not use `anyOf`/`oneOf`/`allOf` in tool input schemas — use `Type.Optional` and `stringEnum` helpers.

### 5.5 Plugin runtime

Plugins receive a **trusted runtime object** injected by the gateway. This is
the only way plugins can call back into core at runtime:

```typescript
// src/plugins/runtime/types.ts
type PluginRuntime = PluginRuntimeCore & {
  subagent: {
    // Dispatch a message to an agent session
    run: (params: {
      sessionKey: string;
      message: string;
      provider?: string;
      model?: string;
      deliver?: boolean;
    }) => Promise<{ runId: string }>;

    // Wait for a subagent run to finish
    waitForRun: (params: {
      runId: string;
      timeoutMs?: number;
    }) => Promise<{ status: "ok" | "error" | "timeout"; error?: string }>;

    // Read conversation history
    getSessionMessages: (params: {
      sessionKey: string;
      limit?: number;
    }) => Promise<{ messages: unknown[] }>;

    // Delete a session
    deleteSession: (params: {
      sessionKey: string;
      deleteTranscript?: boolean;
    }) => Promise<void>;
  };

  channel: PluginRuntimeChannel;
};
```

Plugins receive runtime via `setRuntime` (channel plugins) or as a field on
`PluginRuntimeCore` (general plugins). Never import runtime helpers directly
from `src/**` — use the injected object.

---

## Module 6 — Testing

### 6.1 Test setup

```bash
# Run all tests
pnpm test

# Run tests for a single file/pattern
pnpm test -- extensions/irc/src/channel.test.ts

# Run a specific test by name
pnpm test -- extensions/irc/src/channel.test.ts -t "resolves DM policy"

# Run with coverage
pnpm test:coverage
```

Coverage thresholds: **70%** lines, branches, functions, statements.

### 6.2 Test file conventions

| Pattern | Purpose |
|---|---|
| `*.test.ts` | Unit / integration (colocated with source) |
| `*.e2e.test.ts` | End-to-end workflows |
| `*.live.test.ts` | Tests that hit real external services |
| `*.integration.test.ts` | Multi-component integration |
| `*.contract.test.ts` | Plugin boundary contract verification |

### 6.3 A typical unit test

```typescript
// extensions/irc/src/normalize.test.ts
import { describe, expect, it } from "vitest";
import { normalizeIrcMessagingTarget, isChannelTarget } from "./normalize.js";

describe("normalizeIrcMessagingTarget", () => {
  it("lowercases channel names", () => {
    expect(normalizeIrcMessagingTarget("#OpenClaw")).toBe("#openclaw");
  });

  it("returns null for empty input", () => {
    expect(normalizeIrcMessagingTarget("")).toBeNull();
  });
});

describe("isChannelTarget", () => {
  it("returns true for # prefixed strings", () => {
    expect(isChannelTarget("#general")).toBe(true);
  });

  it("returns false for user nicks", () => {
    expect(isChannelTarget("alice")).toBe(false);
  });
});
```

### 6.4 Testing channel plugins

For channel plugin tests, avoid prototype mutation. Use per-instance stubs
and test the adapter functions directly:

```typescript
import { describe, expect, it, vi } from "vitest";
import { createScopedChannelConfigAdapter } from "openclaw/plugin-sdk/channel-config-helpers";

describe("ircConfigAdapter", () => {
  it("resolveAllowFrom returns account allowFrom list", () => {
    const account = {
      accountId: "default",
      configured: true,
      host: "irc.libera.chat",
      config: { allowFrom: ["alice!*@*", "bob!*@*"] },
    } as ResolvedIrcAccount;

    // Test the adapter function in isolation
    const result = account.config.allowFrom;
    expect(result).toEqual(["alice!*@*", "bob!*@*"]);
  });
});
```

### 6.5 Testing provider auth

Provider auth methods receive a `ProviderAuthContext`. Stub out the `prompter`
to avoid real CLI interaction:

```typescript
import { describe, expect, it, vi } from "vitest";

function makePrompter(responses: Record<string, string>) {
  return {
    text: vi.fn(({ message }: { message: string }) =>
      Promise.resolve(responses[message] ?? ""),
    ),
    note: vi.fn(() => Promise.resolve()),
    select: vi.fn(),
    confirm: vi.fn(),
  };
}

describe("runAnthropicSetupToken", () => {
  it("builds a token profile from prompter input", async () => {
    const ctx = {
      config: {},
      prompter: makePrompter({
        "Paste Anthropic setup-token": "sk-ant-test-token",
        "Token name (blank = default)": "work",
      }),
      runtime: {},
      isRemote: false,
      openUrl: vi.fn(),
      oauth: { createVpsAwareHandlers: vi.fn() },
    } as unknown as ProviderAuthContext;

    const result = await runAnthropicSetupToken(ctx);
    expect(result.profiles).toHaveLength(1);
    expect(result.profiles[0].credential.type).toBe("token");
  });
});
```

### 6.6 Test performance guardrails

From `CLAUDE.md`:

- Do **not** put `vi.resetModules()` + `await import(...)` in `beforeEach` for heavy modules. Use `beforeAll` or static imports.
- Keep `forks` pool only — never `threads`, `vmThreads`, `vmForks`.
- Keep expensive runtime work (snapshots, installs, migrations) behind `*.runtime.ts` seams so tests mock the seam instead.
- Use `OPENCLAW_TEST_PROFILE=serial OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test` for low-memory gate runs.

---

## Module 7 — Config System

### 7.1 Config shape

The OpenClaw config is a deeply nested JSON5 object. Key top-level keys:

```typescript
type OpenClawConfig = {
  agents?: { defaults?: { model?: { primary?: string; fallbacks?: string[] } } };
  channels?: {
    telegram?: TelegramConfig;
    discord?: DiscordConfig;
    slack?: SlackConfig;
    irc?: IrcConfig;
    // ... one key per channel
  };
  models?: { providers?: Record<string, ModelProviderConfig> };
  auth?: { profiles?: Record<string, AuthProfile> };
  tools?: Record<string, unknown>;
  commands?: Record<string, unknown>;
  gateway?: { mode?: "local" | "remote"; host?: string; port?: number };
};
```

### 7.2 Config reads and writes

Never read config directly from disk in plugin code. Use the helpers provided
by `openclaw/plugin-sdk/channel-config-helpers` or the `ProviderAuthContext`.

For CLI commands, use `src/cli/config-cli.ts` patterns:

```bash
# Set a config value
openclaw config set channels.irc.host irc.libera.chat

# Get a config value
openclaw config get channels.irc.host

# Open the config file in your editor
openclaw config edit
```

### 7.3 Config schema drift check

When you add fields to a plugin's config schema, update the generated baseline:

```bash
pnpm config:docs:gen    # regenerate baseline
pnpm config:docs:check  # verify no drift
```

---

## Module 8 — Architecture Boundaries Reference

### 8.1 Boundary summary

| Boundary | Defined in | Rule |
|---|---|---|
| Plugin SDK | `src/plugin-sdk/*` | Plugins import only from `openclaw/plugin-sdk/*` |
| Channel internals | `src/channels/**` | Internal; expose new seams via plugin SDK |
| Provider internals | `src/plugins/types.ts` | Core owns inference loop; plugins own provider behavior |
| Gateway protocol | `src/gateway/protocol/schema.ts` | Additive changes; versioned for breaking changes |
| Bundled plugin internals | `extensions/<id>/src/**` | Not imported by core or other plugins |

### 8.2 Adding a new seam

When you need plugin code to access something that currently lives in core:

1. **Do not** import `src/**` from the extension.
2. Add the needed function/type to `src/plugin-sdk/<domain>.ts`.
3. Add the subpath export to `package.json` exports.
4. Import from `openclaw/plugin-sdk/<domain>` in the plugin.
5. Update `src/plugin-sdk/AGENTS.md` if the seam is new.

### 8.3 Dynamic import guardrail

Do not mix `await import("x")` and `import ... from "x"` for the same module
in production code. Create a `*.runtime.ts` boundary for lazy-loaded modules:

```typescript
// bad — mixes static and dynamic import of the same module
import { helper } from "./heavy-module.js";          // static
const mod = await import("./heavy-module.js");        // dynamic (same module)

// good — lazy callers import a dedicated boundary file
// heavy-module.runtime.ts
export { helper } from "./heavy-module.js";

// lazy-caller.ts
const mod = await import("./heavy-module.runtime.js");
```

After refactoring module boundaries, run `pnpm build` and check for
`[INEFFECTIVE_DYNAMIC_IMPORT]` warnings.

---

## Module 9 — Contributing Workflow

### 9.1 Commit conventions

```bash
# Use the repo committer script (scopes staging correctly)
scripts/committer "CLI: add verbose flag to channels status" src/cli/channels-cli.ts

# Or use FAST_COMMIT=1 to skip the repo-wide format+check hook
FAST_COMMIT=1 git commit -m "IRC: fix port normalization"
```

Commit message format: `<Area>: <action verb> <what>` (e.g., `IRC: fix`, `Anthropic: add`, `Docs: update`).

### 9.2 Pre-commit hook

The hook runs `pnpm format` then `pnpm check`. `FAST_COMMIT=1` skips both.
Run them manually when using fast-commit mode:

```bash
pnpm format:fix   # auto-fix formatting
pnpm check        # lint + type-check
pnpm test         # full test suite (required before pushing)
```

### 9.3 Landing gate

Before pushing to `main`:

```bash
pnpm check       # lint + type-check
pnpm test        # full test suite
pnpm build       # required if touching build output or module boundaries
```

### 9.4 Changelog

Add user-facing changes to `CHANGELOG.md` in the active version block,
appended to the end of the relevant section (`### Changes` or `### Fixes`):

```markdown
### Changes

- IRC: add NickServ SASL authentication support. Thanks @yourhandle
```

Do not add changelog entries for internal refactors or test-only changes
unless they affect user-visible behavior.

---

## Quick Reference Card

### Plugin types at a glance

```
Tool plugin       definePluginEntry  →  api.registerTool(...)
Hook plugin       definePluginEntry  →  api.registerHook(...)
Provider plugin   definePluginEntry  →  api.registerProvider(...)
Channel plugin    defineChannelPluginEntry  →  { plugin: ChannelPlugin, setRuntime }
```

### Key SDK imports

```typescript
// Entry points
import { definePluginEntry, defineChannelPluginEntry } from "openclaw/plugin-sdk/plugin-entry";

// Auth helpers
import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth-api-key";
import { upsertAuthProfile, buildTokenProfileId } from "openclaw/plugin-sdk/provider-auth";

// Channel helpers
import { createChatChannelPlugin } from "openclaw/plugin-sdk/core";
import { createScopedChannelConfigAdapter } from "openclaw/plugin-sdk/channel-config-helpers";
import { createComputedAccountStatusAdapter } from "openclaw/plugin-sdk/status-helpers";
import { createChannelDirectoryAdapter } from "openclaw/plugin-sdk/directory-runtime";

// Model helpers
import { cloneFirstTemplateModel } from "openclaw/plugin-sdk/provider-model-shared";

// Shared utilities
import { isSecretRef } from "openclaw/plugin-sdk/core";
import { formatCliCommand } from "openclaw/plugin-sdk/cli-runtime";
```

### Files to read first

| Goal | File |
|---|---|
| Understand the plugin boundary | `src/plugin-sdk/AGENTS.md` |
| Understand the channel boundary | `src/channels/AGENTS.md` |
| Write a provider plugin | `extensions/anthropic/index.ts` |
| Write a channel plugin | `extensions/irc/src/channel.ts` |
| Write a tool plugin | `docs/plugins/building-plugins.md` |
| Protocol changes | `src/gateway/protocol/AGENTS.md` |
| PR workflow | `.agents/skills/openclaw-pr-maintainer/SKILL.md` |
