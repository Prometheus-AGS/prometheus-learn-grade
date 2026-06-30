# Examples

Reference configuration files. Copy and edit, don't use as-is.

## `.env.example`

Environment variables that `learn-grade` reads from `~/.prometheus/.env`
(via `set -a; . ~/.prometheus/.env; set +a`). The wrapper falls back to
`CLOUD_LLM_URL` and `LOCAL_LLM_URL` if `LEARN_GRADER_URL` is unset.

Required for the grader to function:

```bash
LEARN_GRADER_URL=http://localhost:8181/v1     # the openai-proxy
LEARN_GRADER_MODEL=gpt-5.5                   # the strongest available model
LEARN_GRADER_API_KEY=__USE_PROXY__           # ignored by the proxy (uses ~/.codex/auth.json)
```

Production alternative (without the openai-proxy):

```bash
LEARN_GRADER_URL=https://api.openai.com/v1
LEARN_GRADER_MODEL=gpt-4o          # or gpt-4-turbo, or whatever you have access to
LEARN_GRADER_API_KEY=sk-proj-...
```