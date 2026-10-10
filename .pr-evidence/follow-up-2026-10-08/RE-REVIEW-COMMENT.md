@clawsweeper re-review

Rebased onto current `main` (0 behind / 9 ahead, clean rebase) and resolved every remaining review blocker. Head: `bfe08ad15ca`.

## Fixes since the last review

- **Fusion discovery (P1):** shared `resolveConfiguredRouteModelLabel` helper now backs both setup discovery and configured-route activation, preserving the literal `openrouter/openrouter/fusion` catalog namespace while still resolving bare/qualified aliases. Added a discovery-to-activation e2e regression (`discovers the same Fusion identity that activation resolves`).
- **Saved-sign-in rotation (issue #167381):** activating a saved sign-in without an explicit `modelRef` now prefers the agent's current configured model when it belongs to the same provider, instead of the method's starter model. Regression verified to fail on the previous behavior and pass after the fix.
- **Merge risk (P1):** fresh clean rebase onto current `main`; behavior re-verified post-rebase.
- Removed the low-value self-comparison test flagged in review.

## Verification (post-rebase, this head)

| Suite                                                                     | Result                          | Single-worker wall time       |
| ------------------------------------------------------------------------- | ------------------------------- | ----------------------------- |
| `src/commands/onboard-inference.test.ts`                                  | 13 passed                       | 2.26 s                        |
| `src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts` | 7 passed                        | 5.90 s                        |
| `src/system-agent/setup-inference-activate.test.ts`                       | 43 passed                       | 126.32 s                      |
| `src/system-agent/inference-route.test.ts` (combined run above)           | —                               | combined 4-file run: 143.15 s |
| Full `src/system-agent` suite                                             | **62 files / 749 tests passed** | 585.64 s                      |

`tsgo` typecheck (core + core test-types), `oxlint`, `oxfmt --check` on the changed files: clean. CI on this head: all checks pass (`openclaw/ci-gate` ✅, "PR context and evidence" ✅).

## Fresh proof — live on the current head (build `2026.9.9 (f2e35ff)`)

Captured today against this exact head in an isolated `$OPENCLAW_HOME` with a real OpenRouter key (no key visible in the UI):

**1. Real onboarding + real inference through the chat UI** — model selector shows **Auto Router · OpenRouter (Default ✓)**, and two real inference turns against the live OpenRouter API replied `OPENROUTER_ONBOARDING_OK`:

![Chat with Auto Router OpenRouter selected and real inference replies](https://raw.githubusercontent.com/chandraveshchaudhari/openclaw/fix/openrouter-onboarding-route-label/docs/assets/pr-evidence/openrouter-onboarding-2026-10-10/08-chat-auto-router-proof.png)

**2. Control UI first-run Model Setup** — Selected model: **OpenRouter · auto** (build stamp bottom-left attributes the capture to this PR head `fix/openrouter…@f2e35ff*`):

![Model Setup first-run with OpenRouter auto selected](https://raw.githubusercontent.com/chandraveshchaudhari/openclaw/fix/openrouter-onboarding-route-label/docs/assets/pr-evidence/openrouter-onboarding-2026-10-10/09-model-setup-selected.png)

**3. Check model passes** — **Ready · 1976 ms** (previously failed with "The candidate route does not match the selected provider, model, and credential"), with **Continue setup** enabled:

![Check model Ready 1976 ms](https://raw.githubusercontent.com/chandraveshchaudhari/openclaw/fix/openrouter-onboarding-route-label/docs/assets/pr-evidence/openrouter-onboarding-2026-10-10/10-model-check-ready.png)

CLI-side confirmation from the same isolated home:

```
$ openclaw models status
Default       : openrouter/auto
Aliases (1)   : OpenRouter -> openrouter/auto

$ openclaw agent --local --message "Reply with exactly: OPENROUTER_ONBOARDING_OK"
[provider-transport-fetch] [model-fetch] response provider=openrouter api=openai-completions model=openrouter/auto status=200 elapsedMs=2033
OPENROUTER_ONBOARDING_OK
```

The PR body has been updated with the same captures and the per-file wall-time table. Ready for merge.
