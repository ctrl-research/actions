# pr-review

LLM-powered pull request review. Fetches the PR diff, sends it to a configurable LLM provider, and posts the review as a sticky PR comment (updated in place on subsequent pushes).

Supported providers:

| `provider` | Endpoint | Auth |
|---|---|---|
| `anthropic` | Anthropic Messages API (`/v1/messages`) | `api-key` required |
| `openai` | OpenAI Chat Completions (`/chat/completions`) | `api-key` required |
| `openai-compatible` | Any OpenAI-compatible server — Ollama, vLLM, LM Studio, OpenRouter, LiteLLM, etc. | `api-key` optional; `base-url` required |

No checkout step is needed — the action reads the diff via the GitHub API.

## Usage — reusable workflow

```yaml
# .github/workflows/pr-review.yaml in your repo
name: PR Review
on:
  pull_request:

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    uses: ctrl-research/actions/.github/workflows/pr-review.yaml@main
    with:
      provider: anthropic
      model: claude-opus-4-8
    secrets:
      llm-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Local/self-hosted LLM (e.g. Ollama)

Point at any OpenAI-compatible endpoint. Use a self-hosted runner label if the endpoint is on a private network:

```yaml
jobs:
  review:
    uses: ctrl-research/actions/.github/workflows/pr-review.yaml@main
    with:
      provider: openai-compatible
      base-url: http://ollama.internal:11434/v1
      model: llama3.1:70b
      runs-on: self-hosted
```

### OpenAI

```yaml
jobs:
  review:
    uses: ctrl-research/actions/.github/workflows/pr-review.yaml@main
    with:
      provider: openai
      model: gpt-4o
    secrets:
      llm-api-key: ${{ secrets.OPENAI_API_KEY }}
```

## Usage — composite action

For more control (custom prompt, consuming the review output in later steps):

```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - name: PR Review
        id: review
        uses: ctrl-research/actions/pr-review@main
        with:
          provider: anthropic
          model: claude-opus-4-8
          api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          post-comment: "false"

      - name: Use the review elsewhere
        run: echo "${{ steps.review.outputs.review }}"
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `provider` | `anthropic` | `anthropic`, `openai`, or `openai-compatible` |
| `base-url` | provider default | API base URL. Required for `openai-compatible` |
| `model` | `claude-opus-4-8` | Model ID |
| `api-key` | — | Provider API key (pass from a secret) |
| `github-token` | `github.token` | Token for reading the diff and posting the comment |
| `pr-number` | from event | PR number when not running on a `pull_request` event |
| `max-tokens` | `16000` | Max output tokens |
| `max-diff-bytes` | `300000` | Diff truncation limit |
| `review-prompt` | built-in | Override the review system prompt |
| `post-comment` | `true` | Post/update the sticky PR comment |

## Outputs

| Output | Description |
|---|---|
| `review` | The generated review body (markdown) |

## Notes

- The sticky comment is identified by an HTML marker (`<!-- pr-review-action -->`); re-runs update it instead of stacking new comments.
- Diffs larger than `max-diff-bytes` are truncated with a notice appended, so the model knows the diff is partial.
- The `openai`/`openai-compatible` path sends `max_tokens`; some newer OpenAI models require `max_completion_tokens` instead — prefer broadly-compatible models or a proxy (LiteLLM) if you hit that.
