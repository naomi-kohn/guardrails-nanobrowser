---
description: "Use when adding, configuring, or debugging an LLM provider (OpenAI, Anthropic, Gemini, Ollama, Groq, xAI, DeepSeek, Cerebras, Azure OpenAI, …). Covers the ProviderTypeEnum, createChatModel factory, llmProviders storage, agentModels selection, and the options UI form."
---

# Adding / Editing an LLM Provider

Nanobrowser keeps providers behind LangChain's `BaseChatModel`. Adding a provider touches four files in a fixed order:

1. **Enum & types** — [packages/storage/lib/settings/types.ts](packages/storage/lib/settings/types.ts)
   - Add a variant to `ProviderTypeEnum`.
   - Extend `ProviderConfig` if the provider needs new fields (e.g. region, deployment id). Keep new fields optional to stay backward-compatible with stored configs.
2. **Storage** — [packages/storage/lib/settings/llmProviders.ts](packages/storage/lib/settings/llmProviders.ts)
   - Add a default entry if the provider should appear pre-listed.
3. **Model factory** — [chrome-extension/src/background/agent/helper.ts](chrome-extension/src/background/agent/helper.ts)
   - Install the matching `@langchain/<provider>` package in [chrome-extension/package.json](chrome-extension/package.json).
   - Add a `case ProviderTypeEnum.<New>` to `createChatModel` that returns a configured `BaseChatModel`.
   - Read credentials from the `ProviderConfig` argument; do not reach into env or storage directly here.
4. **Options UI** — [pages/options/src/components/ModelSettings.tsx](pages/options/src/components/ModelSettings.tsx)
   - Add the form fields for any new `ProviderConfig` keys.
   - Use existing storage hooks (`useStorage(llmProviderStore)`) — do not invent a parallel state path.
5. **i18n** — add `options_models_<provider>_*` keys per [i18n.instructions.md](.github/instructions/i18n.instructions.md) for the provider name, field labels, and error states.

## Rules

- Never hard-code an API key, base URL, or model name. Everything configurable goes through `ProviderConfig`.
- Don't change the `BaseChatModel` interface used by agents; pick a LangChain integration that already implements it. If none exists, wrap a raw client in a custom `BaseChatModel` subclass — don't sprinkle provider-specific branches in the agents themselves.
- Default the new provider OFF in `DEFAULT_LLM_PROVIDERS` (or however the storage seeds defaults). Users must opt in.
- Add at least one Vitest case in `chrome-extension/src/background/agent/__tests__/` (create the folder if missing) that exercises `createChatModel` with a mocked LangChain class.

## Don'ts

- Don't bypass the factory and instantiate a provider client inside an agent or action.
- Don't store any per-request data (last response, latency) inside the provider config; that belongs in chat history or analytics.
- Don't log the resolved API key or request bodies — see [llm-security.instructions.md](.github/instructions/llm-security.instructions.md).
