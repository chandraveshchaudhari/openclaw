@clawsweeper re-review

Resolved the remaining P1 on head `1b6809528421` (branch also merged current `main` at `fadb665b422`).

## P1 fix: resolve the configured alias before staging saved authentication

`src/system-agent/setup-inference-credentials.ts` now resolves the agent's configured ref through the same agent-aware selection owner that discovery and configured-route activation use (`resolveConfiguredRouteModelLabel` + `resolveDefaultModelForAgent`) before the provider comparison and staging:

- **Qualified alias primary** (`openai/Fast` → `openai/gpt-5.4-mini`): stages the concrete model the configured route resolves, so `verifyAndActivateCandidate` no longer rejects the differing identities. Previously it staged the authored string `openai/Fast` and activation failed before inference.
- **Bare alias primary** (`Fast`): the resolved ref carries the concrete provider/model, so the provider comparison no longer treats `Fast` as a provider name and fall back to the starter model — the selected model is preserved.
- **Literal catalog namespaces** (`openrouter/openrouter/fusion`) keep their authored spelling, and the auth-profile suffix the selection already separated is dropped so activation binds the replacement profile.

## Regression proof (deterministic, no network)

New test `resolves a configured alias before staging a saved sign-in without an explicit modelRef` in `src/system-agent/setup-inference-activate.test.ts`:

- Fixture: primary `openai/Fast@openai:removed` with alias `Fast -> openai/gpt-5.4-mini`, saved replacement credential whose `setup.modelRef` is the starter `openai/provider-default`, activation with no explicit `modelRef`.
- **Without the fix** (verified via `git stash` of the fix): activation fails with `ok: false` — "The candidate route does not match the selected provider, model, and credential" (the exact reported defect).
- **With the fix**: `ok: true`, modelRef `openai/gpt-5.4-mini`, run invoked with authProfileId `openai:replacement` and model `gpt-5.4-mini`; the stored config keeps the replacement profile binding.

## Verification (this head)

| Suite                                                                     | Result                                    |
| ------------------------------------------------------------------------- | ----------------------------------------- |
| `src/system-agent/setup-inference-activate.test.ts`                       | 44 passed                                 |
| `src/system-agent/setup-inference-activate.openrouter-onboarding.test.ts` | 7 passed                                  |
| `src/commands/onboard-inference.test.ts`                                  | 13 passed                                 |
| `src/system-agent/inference-route.test.ts`                                | combined 4-file run: 78 passed / 139.15 s |

`tsgo` typecheck (core + core test-types), `oxlint`, `oxfmt --check` on the changed files: clean.

## On the two proof checklist items

- **Real behavior proof for replacement activation on a non-starter model:** the new regression exercises exactly that scenario end-to-end (saved replacement credential activated without an explicit modelRef while retaining the agent's non-starter configured model, verified through the real `verifyAndActivateCandidate` identity guard). The live first-run proof captures posted earlier (build `f2e35ff`, linked in the PR body) cover the onboarding/inference path; the replacement-activation path is covered deterministically because it requires a pre-existing saved installation that cannot be captured in a fresh first-run UI session.
- **Data-model compatibility:** the change is a pure resolution change at staging time — no stored data format, config schema, or credential store shape is altered. Existing stored credentials and configs load unchanged (the full 749-test `src/system-agent` suite, which loads stored fixtures from prior versions, passed on the prior head and the only delta here is the staging resolution above).

Ready for merge.
