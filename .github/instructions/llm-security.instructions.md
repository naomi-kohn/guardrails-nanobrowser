---
description: "Use when editing prompt construction, message history, action result handling, DOM extraction, URL navigation, or anything that flows external content into an LLM. Covers sanitizer, wrapUntrustedContent, firewall checks, and Manifest V3 CSP constraints. Apply whenever the words 'prompt injection', 'sanitize', 'untrusted', 'firewall', or 'CSP' come up."
applyTo: "chrome-extension/src/background/agent/**, chrome-extension/src/background/browser/**, chrome-extension/src/background/services/guardrails/**"
---

# LLM & Browser Security Rules

Nanobrowser drives a real browser with an LLM that sees raw page content. Treat every byte of page-, tab-, or user-supplied content as **untrusted** until it has been through the guardrails layer.

## Required helpers

| Helper | Source | Use for |
| --- | --- | --- |
| `sanitizeContent(content, strict?)` | [services/guardrails/sanitizer.ts](chrome-extension/src/background/services/guardrails/sanitizer.ts) | Strip zero-width chars and prompt-injection patterns from any string before it touches the LLM. |
| `wrapUntrustedContent(content)` | [agent/messages/utils.ts](chrome-extension/src/background/agent/messages/utils.ts) | Wrap external content (DOM, action results, tab titles) with the standard untrusted-content banner before adding it to the message history. |
| `filterExternalContent(...)` | [agent/messages/utils.ts](chrome-extension/src/background/agent/messages/utils.ts) | Filter/replace content blocks coming back from external sources before persisting them to `MessageManager`. |
| URL validation in [browser/util.ts](chrome-extension/src/background/browser/util.ts) | — | Run any URL through the firewall check before navigating. |

## Rules

1. **Never feed raw DOM or tab content directly to a `ChatModel.invoke()`** — always go through `MessageManager`, which applies wrapping/sanitization. If you must build a one-off prompt, call `wrapUntrustedContent()` yourself.
2. **Sanitize before storing.** Any string written into `ActionResult.extractedContent`, message history, or storage must be sanitized first if it originated from the page.
3. **Firewall every navigation.** Before calling `browserContext.navigateTo(url)` (or analogous APIs), validate against the firewall settings ([packages/storage/lib/settings/firewall.ts](packages/storage/lib/settings/firewall.ts)) and reject non-`http(s):` schemes.
4. **No `eval`, no `new Function`, no dynamic `import()` of remote code.** Manifest V3 CSP forbids it and we don't relax the CSP.
5. **No credentials in logs.** Don't log API keys, tokens, full request headers, or message bodies that may contain user content. Use `createLogger('Scope')` and log identifiers, not payloads.
6. **Use structured action schemas, not free-form strings.** The LLM must call a registered action; never `eval` a model-supplied script or pass model output directly to `chrome.scripting.executeScript`.
7. **Validate at the boundary, trust internally.** Validate user input and LLM output where they enter the system (action `safeParse`, URL checks, sanitizer). Once validated, don't re-validate on every internal hop.

## Adding a new external-content pathway

If you introduce a new source of external content (e.g. a new "extract" action, a new tab listener, a new clipboard read), you must:
- Pipe its output through `sanitizeContent` (use `strict: true` for content that will appear inside system prompts).
- Wrap it via `wrapUntrustedContent` before adding to message history.
- Add a Vitest case in [chrome-extension/src/background/services/guardrails/__tests__/](chrome-extension/src/background/services/guardrails/__tests__/) covering at least one known prompt-injection pattern.

## Secrets & config

- API keys live in `@extension/storage` only. Never read them from process env at runtime.
- Local development secrets go in a git-ignored `.env.local` at the workspace root with `VITE_*` prefix (loaded by [chrome-extension/vite.config.mts](chrome-extension/vite.config.mts) via `loadEnv(mode, '../', 'VITE_')`).
- Never echo a stored API key into a log, error message, or i18n placeholder.
