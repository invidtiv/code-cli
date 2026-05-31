# ACP OpenRouter Model Discovery Handoff

> For agentic workers: REQUIRED: follow the repository workflow in `AGENTS.md`: inspect implementation, inspect tests, write failing tests first, implement the minimal fix, then run tests, lint, and `bun run proof`.

## Goal

Make Autohand ACP report and validate actual OpenRouter models from the OpenRouter models API instead of the current hardcoded ACP model list.

This should fix Paperclip and other ACP clients seeing invalid model choices like `deepseek/deepseek-v4`, while valid OpenRouter model IDs are hidden or rejected.

## User-Visible Problem

Paperclip asks Autohand ACP for supported models and then calls `session/set_model`.

Observed with the installed `autohand`:

- ACP advertises `deepseek/deepseek-v4`.
- `session/set_model` accepts `deepseek/deepseek-v4`.
- The next prompt fails because OpenRouter rejects it: `deepseek/deepseek-v4 is not a valid model ID`.
- `session/set_model` rejects `deepseek/deepseek-v4-flash:free` and `deepseek/deepseek-v4-pro` as unsupported because they are absent from ACP's hardcoded list.
- `autohand --acp --model ...` can start successfully while not actually changing the ACP session's `currentModelId`.

## Root Cause

ACP model discovery is static and provider-agnostic.

Key files:

| File | Current behavior |
|------|------------------|
| `src/modes/acp/types.ts` | `parseAvailableModels(config)` adds the configured model, then appends a hardcoded "popular models" list. This list includes invalid/stale OpenRouter IDs such as `deepseek/deepseek-v4`. |
| `src/modes/acp/adapter.ts` | `buildSessionModels()` and `validateModel()` call the static `parseAvailableModels(config)`. |
| `src/core/agent/AgentCommandRuntime.ts` | `applyAgentAcpModel()` applies the requested model to the active provider and calls `host.llm.setModel(modelId)` without checking whether that provider accepts it. |
| `src/providers/OpenRouterProvider.ts` | Has `listModels()` and already calls `fetchOpenRouterModelCapabilities()`, but ACP does not use it. |
| `src/providers/modelCapabilities.ts` | Already contains `fetchOpenRouterModelCapabilities()` with caching and timeout behavior. Reuse this. |
| `src/modes/acp/adapter.ts` | `createManagedSession()` derives `modelId` from config and does not apply `cliOptions.model`, unlike the normal CLI path. |

## Recommended Design

Add provider-aware ACP model discovery.

For the first change, prioritize OpenRouter because that is the default provider and the failing Paperclip path.

1. Add an async ACP model discovery helper.
   - Suggested location: `src/modes/acp/types.ts` or a new small module under `src/modes/acp/`.
   - Keep `resolveDefaultModel(config)` synchronous if possible.
   - Replace ACP usage of `parseAvailableModels(config)` with an async resolver where the adapter can await it.

2. For `config.provider === "openrouter"`:
   - Use existing `fetchOpenRouterModelCapabilities()` from `src/providers/modelCapabilities.ts`.
   - Map `capability.id` into ACP `modelId`.
   - Put the currently configured OpenRouter model first.
   - Deduplicate while preserving order.
   - Include `openrouter/auto` if the API response does not include it, because it is a useful OpenRouter route selector and works in ACP prompt testing.
   - On API failure, fall back to a small safe list:
     - current configured model, if present
     - `openrouter/auto`
     - `nvidia/nemotron-3-super-120b-a12b:free` if configured or known from local defaults
   - Do not include `deepseek/deepseek-v4` unless the OpenRouter API actually returns that exact ID.

3. For non-OpenRouter providers:
   - Do not mix OpenRouter IDs into provider-specific ACP lists.
   - Either preserve the current configured model plus provider-specific `listModels()` where available, or keep a conservative fallback containing only the configured model.
   - DeepSeek provider model IDs are unprefixed in `src/providers/DeepSeekProvider.ts`: `deepseek-v4-flash`, `deepseek-v4-pro`, `deepseek-chat`, `deepseek-reasoner`.

4. Make validation use the same discovered list.
   - `unstable_setSessionModel()` should reject models that are not valid for the active provider.
   - If model discovery fails, validation should still allow the current configured model and `openrouter/auto` for OpenRouter.

5. Apply ACP `--model` override during session setup.
   - Mirror the normal CLI behavior from `src/index.ts`.
   - In ACP `ensureConfig()` or `createManagedSession()`, if `this.cliOptions.model` is present, set the active provider config's `model` before resolving default model and creating the provider.

## Test Plan

Write failing tests before changing production code.

Update or add tests around:

- `tests/modes/acp/types.test.ts`
- `tests/modes/acp/adapter.test.ts`
- Add a new focused ACP model discovery test file if that keeps async behavior cleaner.

Required test cases:

- OpenRouter ACP models are populated from mocked `fetchOpenRouterModelCapabilities()`.
- The configured model is first and is not duplicated.
- `deepseek/deepseek-v4` is not advertised when the mocked OpenRouter API does not return it.
- `unstable_setSessionModel()` accepts a mocked OpenRouter API model.
- `unstable_setSessionModel()` rejects a model not returned by OpenRouter discovery.
- API failure falls back to the configured model plus safe OpenRouter defaults.
- ACP `--model` option changes the initial `currentModelId` for new sessions.

Existing tests that will need updates:

- `tests/modes/acp/types.test.ts` currently asserts `parseAvailableModels()` contains `deepseek/deepseek-v4`.
- `tests/modes/acp/adapter.test.ts` currently assumes a hardcoded model list and validates against `openai/gpt-5`.

## Manual Verification

After implementation, test against a real OpenRouter network response:

```bash
autohand --acp
```

Then connect with a small JSON-RPC/ACP client and verify:

- `initialize` succeeds.
- `session/new` returns `availableModels` from OpenRouter.
- Invalid stale IDs such as `deepseek/deepseek-v4` are absent unless OpenRouter returns them.
- `session/set_model` accepts a real returned OpenRouter model ID.
- Prompting with that model no longer fails with `model_not_found`.

Also test CLI override:

```bash
autohand --acp --model openrouter/auto
```

Expected: the new ACP session reports `currentModelId: "openrouter/auto"`.

## Validation Commands

Use the smallest relevant checks first:

```bash
bun test tests/modes/acp/types.test.ts tests/modes/acp/adapter.test.ts
```

Then complete the repository-required validation:

```bash
bun test
bun lint
bun run proof
```

## Acceptance Criteria

- ACP model list for OpenRouter reflects the OpenRouter API, not a stale static list.
- `deepseek/deepseek-v4` is not advertised or accepted unless OpenRouter actually returns that exact model ID.
- Paperclip can discover models through ACP and choose a cheap valid OpenRouter model without hardcoded adapter workarounds.
- `autohand --acp --model <model>` affects the first ACP session model.
- Tests cover model discovery, validation, fallback, and CLI override behavior.
