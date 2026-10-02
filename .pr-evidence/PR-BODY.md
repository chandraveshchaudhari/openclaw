## What Problem This Solves

Fixes: the "Connect your AI → API Keys → OpenRouter API key" onboarding flow rejects a valid OpenRouter API key when the user clicks Connect, showing
"That key didn't work" and "The candidate route does not match the selected provider, model, and credential." The same failure hits OpenRouter OAuth onboarding.

## User Impact

User impact: entering a valid OpenRouter API key (or signing in with OpenRouter OAuth) during onboarding now connects and activates the default OpenRouter model instead of failing. No configuration change or new key is required, and no user action is needed to pick up the fix. No change for any other provider.

## Why This Change Was Made

The onboarding screen only asks for a key — it has no model picker — so it uses the OpenRouter default model `openrouter/auto`. The configured-route
resolver built its `modelLabel` by raw `${provider}/${modelId}` concatenation. OpenRouter's catalog model id already carries its own provider prefix
(`openrouter/auto`), so the raw form produced the double-prefixed `openrouter/openrouter/auto`, while the staged candidate ref and the written config
primary stay `openrouter/auto` (both produced via the canonical `modelKey` / `upsertCanonicalModelConfigEntry`). `verifyAndActivateCandidate` compares
`route.modelLabel` to the staged ref, so the two representations of the same route diverged and a valid credential was rejected before it was ever tested.

Building `modelLabel` with the canonical `modelKey(selection.provider, selection.modelId)` collapses the duplicated self-prefix, so the label matches the staged ref and the config primary. `modelKey` only collapses a prefix a model id already carries, so providers whose ids are not self-prefixed are unaffected.

<details>
<summary>Code-level detail</summary>

- `src/system-agent/inference-route.ts`: `modelLabel` now uses `modelKey(selection.provider, selection.modelId)` instead of ``
  `${selection.provider}/${selection.modelId}` ``.
- The default model ref (`OPENROUTER_DEFAULT_MODEL_REF = "openrouter/auto"`) is unchanged — the canonical catalog/display id is intentionally the
slash-containing `openrouter/auto`, consistent with the OpenRouter manifest's `prefixWhenBare` and the rest of the system.
</details>

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

*(screenshot also available as `06-real-onboarding-proof.png` in the PR author's local `.pr-evidence/` folder)*

### Regression proof (deterministic, no network / no real key)

Temporarily reverting `modelLabel` to raw concatenation:

```
AssertionError: expected 'openrouter/openrouter/auto' to be 'openrouter/auto'
Tests  1 failed | 3 passed (4)
```

With the fix:

```
Tests  4 passed (4)
```

### Full local verification (terminal output, no network / no real key)

```
$ npx vitest run src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts src/system-agent/setup-inference-activate.test.ts

 Test Files  2 passed (2)
      Tests  55 passed (55)
```

- New regression tests (`src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts`) resolve the real configured route and assert:
  - the OpenRouter default route label equals the staged candidate ref (`openrouter/auto`) — previously `openrouter/openrouter/auto`;
  - a non-default OpenRouter model (`openrouter/moonshotai/kimi-k2.6`) still produces a distinct label, so the activation guard is not loosened;
  - a non-prefixing provider (`anthropic/claude-sonnet-4-6`) is unchanged.
- End-to-end activation tests (`src/system-agent/setup-inference-activate.test.ts`): OpenRouter API-key and OAuth onboarding both save the credential, run the tool-free model test, and activate `openrouter/auto@openrouter:default`.
- `tsgo` typecheck (core) and `oxlint` on the changed files: clean. Coercion-helper declaration guard: passes.

<details>
<summary>Suites run (no network / no real key), all green</summary>

inference-route, inference-route-runtime, assistant.configured, update-repair-inference(.route), OpenRouter onboard/oauth/index, model-selection (136
tests), setup-inference-credentials.lifecycle, setup-inference.groq-external.integration (full `activateSetupInference` flow), models/auth-activate,
gateway system-agent-setup-resolution and setup-auth-retry.
</details>
