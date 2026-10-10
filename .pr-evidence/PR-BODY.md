## What Problem This Solves

Fixes: the "Connect your AI → API Keys → OpenRouter API key" onboarding flow rejects a valid OpenRouter API key when the user clicks Connect, showing
"That key didn't work" and "The candidate route does not match the selected provider, model, and credential." The same failure hits OpenRouter OAuth onboarding and the Control UI first-run model setup.

## User Impact

User impact: entering a valid OpenRouter API key (or signing in with OpenRouter OAuth) during onboarding now connects and activates the default OpenRouter model instead of failing. No configuration change or new key is required, and no user action is needed to pick up the fix. No change for any other provider.

## Why This Change Was Made

The onboarding screen only asks for a key — it has no model picker — so it uses the OpenRouter default model `openrouter/auto`. The configured-route
resolver built its `modelLabel` by raw `${provider}/${modelId}` concatenation. OpenRouter's catalog model id already carries its own provider prefix
(`openrouter/auto`), so the raw form produced the double-prefixed `openrouter/openrouter/auto`, while the staged candidate ref and the written config
primary stay `openrouter/auto` (both produced via the canonical `modelKey` / `upsertCanonicalModelConfigEntry`). `verifyAndActivateCandidate` compares
`route.modelLabel` to the staged ref, so the two representations of the same route diverged and a valid credential was rejected before it was ever tested.

The route label must equal the config-written primary. The completion resolver strips a self-provider prefix from the model id (`openrouter/auto` → `auto`), so rebuilding the label from `selection.provider`/`selection.modelId` cannot recover the literal catalog namespace. This PR therefore builds `modelLabel` from the **configured ref** (`configuredSelection.modelRef`), which preserves the literal namespace, and falls back to the canonical `modelKey(selection.provider, selection.modelId)` only when the configured ref is unavailable (implicit primary / utility derivation). The auth-profile suffix the selection already separated is dropped so the label stays profile-free.

This fixes the default (`openrouter/auto`) **and** preserves literal namespaces such as the documented `openrouter/openrouter/fusion` selection, whose upstream model id is `openrouter/fusion`. A naive `modelKey`-only fix would have collapsed `openrouter/openrouter/fusion` to `openrouter/fusion` and rejected that selection — the regression this revision adds coverage for.

<details>
<summary>Code-level detail</summary>

- `src/system-agent/inference-route.ts`: `modelLabel` is now derived from `configuredSelection.modelRef` (auth-profile suffix stripped), falling back to `modelKey(selection.provider, selection.modelId)` when no configured ref is available.
- The default model ref (`OPENROUTER_DEFAULT_MODEL_REF = "openrouter/auto"`) is unchanged — the canonical catalog/display id is intentionally the
  slash-containing `openrouter/auto`, consistent with the OpenRouter manifest's `prefixWhenBare` and the rest of the system.
- `openrouter/openrouter/fusion` is a documented OpenRouter selection (see `extensions/openrouter/index.fusion.test.ts`); its literal namespace is preserved.

</details>

## Follow-up: identity compatibility fixes ([a5b7c19](https://github.com/openclaw/openclaw/commit/a5b7c195d3cf61143f5ebea7aa1793f806123a47))

This revision addresses the two remaining identity mismatches identified in
review:

- Route construction now resolves provider-qualified aliases before preserving a
  literal configured reference. For example, `openai/Fast` (aliasing
  `openai/gpt-5.4-mini`) now labels the concrete model instead of the mutable
  alias spelling. Literal catalog namespaces, including
  `openrouter/openrouter/fusion`, remain preserved.
- Existing-model discovery now uses `modelKey(resolved.provider, resolved.model)`,
  matching activation for provider-prefixed catalog IDs such as OpenRouter
  `openrouter/auto`.

- Added a discovery-to-activation regression for a saved
  `openrouter/auto@openrouter:existing` installation. It discovers
  `openrouter/auto`, re-verifies it through `existing-model`, and uses the saved
  profile without rewriting the configuration.

## Follow-up: Fusion discovery + saved-sign-in rotation

This revision resolves the two remaining review blockers:

- **Preserve literal Fusion references in current-model discovery (P1).** Setup
  discovery advertised `openrouter/fusion` for the documented
  `openrouter/openrouter/fusion` primary while configured-route activation
  retained `openrouter/openrouter/fusion`, so selecting Current model failed with
  "The configured default model changed from openrouter/fusion to
  openrouter/openrouter/fusion" although configuration had not changed. A shared
  `resolveConfiguredRouteModelLabel` helper now backs both discovery and
  activation: it preserves the authored ref only when it is exactly the resolved
  model plus one self-provider prefix (a literal catalog namespace), and otherwise
  defers to the resolved selection (bare alias, qualified alias, or a resolved
  model carrying an extra literal suffix such as `local-utility/tiny@experimental`).
  A discovery-to-activation regression covers the Fusion primary.
- **Rotate a saved sign-in onto the agent's current model (issue #167381).**
  Re-running a provider sign-in saves a replacement credential whose
  `setup.modelRef` is the method's starter model. Activating it without an explicit
  `modelRef` used that starter model, so the resolved route did not match the
  agent's configured model and activation failed with "The candidate route does
  not match the selected provider, model, and credential" (and would have silently
  switched the agent's model had it matched). Activation now prefers the agent's
  current configured model when it belongs to the same provider as the saved
  sign-in, dropping the configured ref's own auth-profile suffix so the
  replacement profile is bound. A regression fails on the previous behavior and
  passes after the fix.
- **Resolve merge risk (P1).** The branch is rebased onto current `main`
  (re-rebased 2026-10-10: 0 behind / 9 ahead, clean); the repaired behavior is
  re-verified against current `main`.
- Removed the low-value self-comparison test flagged in review
  (`mirrors the slash-ambiguous OpenRouter compatibility default`).

### Current verification (post-rebase, head `bfe08ad15ca`)

```
pnpm vitest run src/commands/onboard-inference.test.ts \
  src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts \
  src/system-agent/inference-route.test.ts \
  src/system-agent/setup-inference-activate.test.ts
Test Files  4 passed (4)
Tests       77 passed (77)
Duration    143.15s

pnpm vitest run src/system-agent
Test Files  62 passed (62)
Tests       749 passed (749)
Duration    585.64s
```

Single-worker wall time per new or materially changed test file:

| Suite                                                                     | Result    | Wall time |
| ------------------------------------------------------------------------- | --------- | --------- |
| `src/commands/onboard-inference.test.ts`                                  | 13 passed | 2.26 s    |
| `src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts` | 7 passed  | 5.90 s    |
| `src/system-agent/setup-inference-activate.test.ts`                       | 43 passed | 126.32 s  |

CI on this head: all checks pass (`openclaw/ci-gate` ✅, "PR context and evidence" ✅).
`tsgo` typecheck (core + core test-types), `oxlint`, and `oxfmt --check` on the
changed files: clean.

### Fresh head-attributed proof (2026-10-10, build `2026.9.9 (f2e35ff)`)

Captured live against the current PR head in an isolated `$OPENCLAW_HOME` with a
real OpenRouter key (no key visible in the UI):

- Chat with **Auto Router · OpenRouter (Default ✓)** selected and two real
  inference turns replying `OPENROUTER_ONBOARDING`:
  https://raw.githubusercontent.com/chandraveshchaudhari/openclaw/fix/openrouter-onboarding-route-label/docs/assets/pr-evidence/openrouter-onboarding-2026-10-10/08-chat-auto-router-proof.png
- Model Setup first-run, **Selected model: OpenRouter · auto** (build stamp
  attributes the capture to this PR head):
  https://raw.githubusercontent.com/chandraveshchaudhari/openclaw/fix/openrouter-onboarding-route-label/docs/assets/pr-evidence/openrouter-onboarding-2026-10-10/09-model-setup-selected.png
- **Check model → Ready · 1976 ms** with Continue setup enabled:
  https://raw.githubusercontent.com/chandraveshchaudhari/openclaw/fix/openrouter-onboarding-route-label/docs/assets/pr-evidence/openrouter-onboarding-2026-10-10/10-model-check-ready.png

## Evidence

### Real-behavior proof (live onboarding + real inference, redacted)

Ran the actual onboarding flow and a real inference turn from this branch
(build `2026.9.7 (8584e82)`) with a real OpenRouter API key against the live
OpenRouter API (key redacted; isolated temp `$OPENCLAW_HOME`):

```
$ openclaw onboard --non-interactive --accept-risk \
    --auth-choice openrouter-api-key --openrouter-api-key sk-or-v1-****REDACTED**** \
    --skip-channels --skip-skills --skip-ui --skip-daemon --skip-health --skip-hooks --skip-search --skip-bootstrap

Workspace OK: $OPENCLAW_HOME/.openclaw/workspace
Sessions OK: $OPENCLAW_HOME/.openclaw/agents/main/sessions
Updated config: $OPENCLAW_HOME/.openclaw/openclaw.json
  Backup: $OPENCLAW_HOME/.openclaw/openclaw.json.bak
```

```
$ openclaw models status

Default       : openrouter/auto
Aliases (1)   : OpenRouter -> openrouter/auto

Auth overview
- openrouter effective=profiles:$OPENCLAW_HOME/.openclaw/state/openclaw.sqlite |
  profiles=1 (0 oauth, 0 token, 1 api-key) | openrouter:default=sk-or-v1...e1a682b4
```

```
$ openclaw agent --local --message "Reply with exactly: OPENROUTER_ONBOARDING_OK"

[provider-transport-fetch] [model-fetch] response provider=openrouter api=openai-completions model=openrouter/auto status=200 elapsedMs=5019 dispatcher=new contentType=text/event-stream
OPENROUTER_ONBOARDING_OK
[agents/agent-command] [agent] run e7d91a5e-ab03-4ea5-8f36-c2492637ca12 ended with stopReason=stop
```

Persisted credential (from the state DB, redacted):

```json
{
  "profiles": {
    "openrouter:default": {
      "type": "api_key",
      "provider": "openrouter",
      "key": "sk-or-v1-****REDACTED**** (ends: ...82b4)"
    }
  }
}
```

![Real onboarding + inference proof](https://github.com/user-attachments/assets/placeholder)

_(screenshot also available as `06-real-onboarding-proof.png` in the PR author's local `.pr-evidence/` folder)_

### Real-behavior proof (Control UI first-run model setup, redacted)

Ran the actual Control UI onboarding at `/model-setup?firstRun=1` against the installed build with the real OpenRouter key:

- Model Setup shows **Selected model: OpenRouter · openrouter/auto**.
- **Check model** → `Ready · 2296 ms` (previously failed with the route-mismatch error).
- **Continue setup** → navigates to chat; the model selector shows **Auto Router · OpenRouter**.
- Config written: `agents.defaults.model.primary = "openrouter/auto"`.

![Control UI onboarding working](https://github.com/user-attachments/assets/placeholder)

_(screenshot also available as `07-gui-onboarding-working.png` in the PR author's local `.pr-evidence/` folder)_

### Regression proof (deterministic, no network / no real key)

The new regression suite resolves the real configured route and asserts the label for each case:

```
$ npx vitest run src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts

 ✓ resolves the OpenRouter default route label to the staged candidate ref
 ✓ keeps a distinct label for a non-default OpenRouter model
 ✓ preserves a literal OpenRouter catalog namespace (Fusion)
 ✓ does not change labels for a non-prefixing provider
 Test Files  1 passed (1)
      Tests  5 passed (5)
```

- `openrouter/auto` → `openrouter/auto` (previously `openrouter/openrouter/auto`).
- `openrouter/openrouter/fusion` → `openrouter/openrouter/fusion` (literal namespace preserved; a `modelKey`-only fix produced `openrouter/fusion`).
- `openrouter/moonshotai/kimi-k2.6` → `openrouter/moonshotai/kimi-k2.6` (distinct label; guard not loosened).
- `anthropic/claude-sonnet-4-6` → `anthropic/claude-sonnet-4-6` (non-prefixing provider unchanged).

### Full local verification (terminal output, no network / no real key)

```
$ npx vitest run src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts src/system-agent/setup-inference-activate.test.ts

 Test Files  2 passed (2)
      Tests  46 passed (46)
```

- New regression tests (`src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts`) resolve the real configured route and assert:
  - the OpenRouter default route label equals the staged candidate ref (`openrouter/auto`) — previously `openrouter/openrouter/auto`;
  - the documented `openrouter/openrouter/fusion` selection keeps its literal namespace;
  - a non-default OpenRouter model (`openrouter/moonshotai/kimi-k2.6`) still produces a distinct label, so the activation guard is not loosened;
  - a non-prefixing provider (`anthropic/claude-sonnet-4-6`) is unchanged.
- End-to-end activation tests (`src/system-agent/setup-inference-activate.test.ts`): OpenRouter API-key and OAuth onboarding both save the credential, run the tool-free model test, and activate `openrouter/auto@openrouter:default`.
- `tsgo` typecheck (core) and `oxlint` on the changed files: clean. `oxfmt --check`: clean.

<details>
<summary>Suites run (no network / no real key), all green</summary>

inference-route, inference-route-runtime, assistant.configured, update-repair-inference(.route), OpenRouter onboard/oauth/index, model-selection (136
tests), setup-inference-credentials.lifecycle, setup-inference.groq-external.integration (full `activateSetupInference` flow), models/auth-activate,
gateway system-agent-setup-resolution and setup-auth-retry.
</details>
