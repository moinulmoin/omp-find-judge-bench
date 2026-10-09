# Reproducing

## Setup

- omp 18.8.7 (`omp --version`)
- Test repo, pinned:
  ```sh
  git clone https://github.com/router-for-me/CLIProxyAPI && git -C CLIProxyAPI checkout 4a2c8186
  ```
- Query: `how a failed upstream request puts a credential into cooldown before it can be retried`
- Ground truth, fixed before any run: `sdk/cliproxy/auth/conductor_cooldown.go` (holds `MarkResult` and the cooldown scheduling).

## One run

The judge model is swapped per run with a config overlay. The empty fallback chain stops omp from silently switching to another judge model when the one under test fails.

```sh
printf 'modelRoles:\n  judge: anthropic/claude-haiku-5-5:medium\nretry:\n  fallbackChains:\n    judge: []\n' > /tmp/judge.yml
PI_CONFIG_FILES=/tmp/judge.yml omp find "how a failed upstream request puts a credential into cooldown before it can be retried" ./CLIProxyAPI --json -q
```

To test fast mode, add `tier:\n  openai: priority\n` to the overlay.

**Clear the judgment cache between runs.** omp caches judge answers in `~/.omp/cache/judgment-cache.db*`, and a repeat query is served from the cache in about 40 ms with 0 tokens. Move those files aside before benchmarking and restore them afterwards. `bench/omp-find-bench` does this for you.

## See what omp sends

`PI_REQ_DEBUG=1` writes every request body (`rr-session-N.json`) and raw response (`rr-session-N.res.log`) to the current directory:

```sh
mkdir -p /tmp/rr && cd /tmp/rr
PI_REQ_DEBUG=1 PI_CONFIG_FILES=/tmp/judge.yml omp find "<query>" <repo> --json -q > out.json
jq '.url, (.body | {thinking, reasoning_effort, output_config, service_tier})' rr-session-1.json
```

## Interactive benchmark

`bench/omp-find-bench` needs `fzf` and Python 3. It lists every model omp knows about and lets you pick several, then asks for a query, a directory and a repeat count. It runs the models in rotation (one run each per round), clears the cache before every run, and appends results to `~/.omp/find-bench.jsonl`.

## Data in this repo

| file | contents |
|---|---|
| `data/runs.json` | all 42 runs (14 judge configs × 3), with timing, requests, errors, output tokens, and the correct file's rank |
| `data/wire-capture.json` | first judge request body per config, as captured with `PI_REQ_DEBUG=1` |
| `data/glm-oai-low-request-timeline.json` | every HTTP request in one GLM `low` run, including the 429 bodies |
| `data/zai-direct-api.json` | direct z.ai API calls bypassing omp, comparing reasoning fields on both endpoints |
