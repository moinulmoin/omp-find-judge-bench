### Title
judge: anthropic-messages route sends `thinking: disabled` to models that cannot disable thinking (GLM-5.3 via z.ai) → server runs default max effort

### Summary
Follow-up to #14320, where the maintainer note said a provider route that does not honor `disableReasoning` should be filed as a separate bug.

The chat judge (`packages/ai/src/judgment/chat.ts`) always passes `disableReasoning: true`. On the **openai-completions** route this becomes the lowest supported effort (`reasoning_effort: "low"`) for GLM-5.3-Flash, which is correct. On the **anthropic-messages** route the same model gets `thinking: {type: "disabled"}`. z.ai documents that GLM-5.3 / GLM-5.3-Flash cannot disable thinking. The server returns 200, ignores the field, and reasons at its default effort (max).

A `models.yml` user cannot fix this: `disabledThinking` / `requiresThinkingEnabled` are catalog-only for `anthropic-messages` (docs/models.md, "Anthropic compatibility"), so a custom anthropic-messages provider has no way to say "this model can't turn thinking off; use the lowest effort instead".

### Environment
- omp 18.8.7, macOS arm64
- `models.yml` custom provider:
  ```yaml
  providers:
    zai:
      baseUrl: https://api.z.ai/api/anthropic
      api: anthropic-messages
      apiKey: ZAI_API_KEY
      models:
        - id: glm-5.3-flash
          reasoning: true
          thinking: { mode: anthropic-budget-effort, efforts: [low, high, max] }
  ```

### Repro
```sh
git clone -q https://github.com/router-for-me/CLIProxyAPI /tmp/CLIProxyAPI && git -C /tmp/CLIProxyAPI checkout -q 4a2c8186
printf 'modelRoles:\n  judge: zai/glm-5.3-flash\nretry:\n  fallbackChains:\n    judge: []\n' > /tmp/judge.yml
mkdir -p /tmp/rr && cd /tmp/rr
PI_REQ_DEBUG=1 PI_CONFIG_FILES=/tmp/judge.yml omp find "how a failed upstream request puts a credential into cooldown before it can be retried" /tmp/CLIProxyAPI --json -q > out.json
jq '.body | {thinking, reasoning_effort, output_config}' rr-session-1.json
grep -c thinking_delta rr-session-1.res.log
```

### Observed
- Request: `POST https://api.z.ai/api/anthropic/v1/messages` with `"thinking": {"type": "disabled"}`, `max_tokens: 4096`, `temperature: 0`.
- Response: `HTTP 200`, streamed `thinking_delta` blocks anyway. In one filename-ranking request there were 2,317 chars of reasoning before a 601-char answer.
- `omp find` on a 1,661-file Go repo: 33.0 / 47.8 / 38.3 s across 3 runs (judgment cache cleared before each).

The same model on the openai-completions route (`baseUrl: https://api.z.ai/api/coding/paas/v4`, `api: openai-completions`) receives `"thinking": {"type": "enabled"}, "reasoning_effort": "low"` and z.ai honors it.

Direct API check, same one-line classification prompt, `max_tokens: 4096`, `temperature: 0`:

| endpoint | body extra | time | output tokens | reasoning chars |
|---|---|---|---|---|
| anthropic `/v1/messages` | (none) | 6.3 s | 234 | 770 |
| anthropic `/v1/messages` | `reasoning_effort: "low"` | 5.7 s | 199 | 689 |
| anthropic `/v1/messages` | `output_config: {effort: "low"}` | 2.8 s | 41 | 1 |
| anthropic `/v1/messages` | `thinking: enabled` + `output_config: {effort: "low"}` | 1.6 s | 3 | 0 |
| openai `/chat/completions` | `reasoning_effort: "low"` | 1.8 s | 10 | 27 |

So the anthropic-compatible endpoint does honor `output_config.effort`; only the disabled-thinking form it receives today is ignored.

### Expected
When `disableReasoning` targets an anthropic-messages model that cannot disable thinking, send the lowest supported effort (`output_config.effort: <efforts[0]>`), as the openai-completions path already does with `reasoningDisableMode: "lowest-effort"`. Alternatively, expose `disabledThinking` / a lowest-effort disable mode in `models.yml` compat for custom anthropic-messages providers.

### Workaround
Define the model on the OpenAI-compatible endpoint (`api: openai-completions`, `baseUrl: https://api.z.ai/api/coding/paas/v4`). The bundled `zai` catalog entry already uses that route, so this only affects users who repoint `zai` (or a custom provider) at the anthropic endpoint.

### Related
- #14320 / #14325: judge forces `disableReasoning` (by design).
- #14970: Retry-After capping. Separately, z.ai returned 4–7 burst `429`s (`code 1302`, "Rate limit reached for requests") per find run when 16 requests were in flight.

Filed upstream: https://github.com/can1357/oh-my-pi/issues/15114
