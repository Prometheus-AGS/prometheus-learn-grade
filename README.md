# prometheus-learn-grade

> **External grader for self-evaluation — gpt-5.5 via the openai-proxy.**
>
> A zero-dependency shell script that grades any freeform explanation against
> a corpus on four dimensions with anti-sycophancy built in. The critic is
> a different model from the producer. That is the entire product.

Self-evaluation only works if the critic is *separate* from the producer.
A model grading its own output produces the same blind spots it already
has. The fix is structural: a different model, a different process,
a different context.

`learn-grade` is a single shell script that does exactly that. It calls
`gpt-5.5` (the strongest model exposed by the [openai-proxy][proxy]) over
the OpenAI chat completions API, with a grading schema that explicitly
forbids flattery — *"lead with errors and gaps, do not soft-pedal. 'Minor
issue' rewrites as 'factual error in X.'"*

[proxy]: https://github.com/GQAdonis/librefang

## Why this is its own repo

`learn-grade` is logically independent of the wiki loop. It does not read
the wiki, write the wiki, or care that a wiki exists. It grades an
explanation against an optional corpus reference. It could be used by:

- The [prometheus-skill-pack][psp] Feynman learning loop (its canonical caller)
- Any evaluation pipeline that needs a corpus-grounded, structured grade
- A CI step that runs after a model writes documentation, to catch hallucinations
- A teaching tool that needs to score student answers against a textbook

Keeping it separate from [prometheus-wiki-loop][pwl] makes it
discoverable for those other use cases. One tool, one job, one repo.

[psp]: https://github.com/Prometheus-AGS/prometheus-skill-system
[pwl]: https://github.com/Prometheus-AGS/prometheus-wiki-loop

## Install

```bash
git clone https://github.com/Prometheus-AGS/prometheus-learn-grade ~/Projects/prometheus-learn-grade
cd ~/Projects/prometheus-learn-grade
cp examples/.env.example ~/.prometheus/.env       # or merge into your existing one
cp bin/learn-grade ~/.local/bin/
chmod +x ~/.local/bin/learn-grade
```

The `.env` file needs three things for the grader to work:

```bash
LEARN_GRADER_URL=http://localhost:8181/v1     # the openai-proxy
LEARN_GRADER_MODEL=gpt-5.5                   # the strongest available model
LEARN_GRADER_API_KEY=__USE_PROXY__           # the proxy uses ~/.codex/auth.json
```

If you don't have the openai-proxy running, point `LEARN_GRADER_URL` at
any OpenAI-compatible endpoint and set `LEARN_GRADER_API_KEY` to a real key:

```bash
LEARN_GRADER_URL=https://api.openai.com/v1
LEARN_GRADER_MODEL=gpt-4o          # or whatever you have access to
LEARN_GRADER_API_KEY=sk-...
```

## Usage

```bash
# direct invocation
learn-grade --concept-id "tokio-spawn" --explanation "tokio::spawn requires the future to be Send and 'static..."

# pipe via stdin
echo "tokio::spawn returns a JoinHandle..." | learn-grade --concept-id "tokio-spawn"

# with a corpus reference
learn-grade \
  --concept-id "tokio-spawn" \
  --explanation "..." \
  --corpus-path ~/.prometheus/knowledge/shared/wiki/tokio-spawn.md
```

## Output

A JSON object on stdout:

```json
{
  "concept_id": "tokio-spawn",
  "scores": {
    "completeness": 0.85,
    "accuracy": 0.95,
    "clarity": 0.90,
    "misconceptions_absent": 1.0
  },
  "overall_score": 0.927,
  "passed": true,
  "pass_threshold": 0.7,
  "gaps": [],
  "feedback": "...",
  "grader_model": "gpt-5.5",
  "grader_url": "http://localhost:8181/v1"
}
```

### The four dimensions

| Dimension | Range | What it measures | Pass criterion |
|---|---|---|---|
| `completeness` | 0..1 | Does it cover the corpus's key aspects? | ≥ 0.7 |
| `accuracy` | 0..1 | Is it factually correct vs the corpus? A single factual error caps at 0.7. | ≥ 0.7 |
| `clarity` | 0..1 | Understandable to a peer audience? | ≥ 0.7 |
| `misconceptions_absent` | 0 or 1 | Any known misconceptions from the corpus? 1.0 = none. | = 1.0 |

`passed` is `true` iff `overall_score ≥ 0.7 AND accuracy ≥ 0.7 AND misconceptions_absent ≥ 0.99`.

### Anti-sycophancy mandate

Built into the system prompt, not a documentation promise:

> Lead with errors and gaps. Do not soft-pedal. "Minor issue" rewrites as
> "factual error in X." Pedagogical sycophancy produces worse learning
> outcomes. The grade is for the learner's benefit, not the author's
> comfort.

This is enforced structurally by routing through the
[sycophancy-correction][sc] MCP server when available. When run standalone
via this shell wrapper, the anti-sycophancy rule is built into the prompt
itself — the wrapper emits a "no prose outside the JSON object" instruction
to the model.

[sc]: https://github.com/Prometheus-AGS/prometheus-skill-system/tree/main/skills/imported/sycophancy-correction

## The producer/grader separation principle

The producer that wrote the explanation and the grader that judges it
must be different models. Same model = same blind spots = false passes.

This repo enforces that by default:

- The producer in the prometheus-skill-pack runs on `gpt-5.4-mini` (cheap, fast, sufficient for compile).
- The grader in this repo runs on `gpt-5.5` (strongest available, longest context, best structured-output following).

If you change `LEARN_GRADER_MODEL` to `gpt-5.4-mini` to "save money", you
have defeated the structural property. The grader will agree with the
producer more often, including in cases where the producer is wrong.

The cost difference is small. A typical grade round-trip is 200–500
tokens (~$0.01–0.03 at standard rates, or free via the openai-proxy).
The cost of a false-pass on a learning loop is unmeasured: the learner
"masters" something they do not understand. Pick the better grader.

## Model routing

For multi-tier setups, the [`config/liter-llm.example.toml`](config/liter-llm.example.toml)
file shows the canonical alias map:

```toml
[aliases]
small    = "gpt-5.4-nano"
medium   = "gpt-5.4-mini"
frontier = "gpt-5.5"
```

Use `small` for mechanical work (file listing, regex matching). Use `medium`
for compile/lint/focus. Use `frontier` only for grading, reflecting, and
strategic dreaming — anywhere the critic must be sharper than the producer.

## Testing

```bash
./scripts/test.sh
```

The test runs two real grader calls: one with an accurate explanation
(must score ≥ 0.7), one with a deliberately wrong explanation (must
have `passed=false` and `misconceptions_absent=0`). Requires the
openai-proxy to be live on `:8181`.

## Related projects

- [`Prometheus-AGS/prometheus-wiki-loop`][pwl] — the universal session-close + session-prime + chat-MCP suite that calls `learn-grade` indirectly via the Feynman loop.
- [`Prometheus-AGS/prometheus-skill-system`][psp] — the 280-skill manifest pack where `learn-grade` is defined as a skill.
- [`GQAdonis/librefang`][bossfang] — the upstream BossFang / Libre Agent OS that this tool builds on.

## License

MIT — see [`LICENSE`](LICENSE).