# bllm

A small command-line tool for prompting models on **Amazon Bedrock**, in the spirit of Simon Willison's [`llm`](https://llm.datasette.io/).

It is a single Bash script that wraps `aws bedrock-runtime converse`, so it works with any text model that Bedrock's Converse API supports — Claude, Nova, Llama, Mistral and others — using your existing AWS credentials.

```bash
$ bllm "Ten fun names for a pet pelican"
$ cat app.py | bllm "Explain this code"
$ bllm -c "Now rewrite it in Go"
```

## Requirements

- Bash 4+
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), configured with credentials (`aws configure`, SSO, environment variables, an instance role, etc.)
- [`jq`](https://jqlang.github.io/jq/)
- Model access enabled in the Bedrock console for the models you want to use

## Installation

```bash
curl -o ~/.local/bin/bllm <url-to-bllm>   # or copy the file into place
chmod +x ~/.local/bin/bllm
```

Any directory on your `PATH` works.

## Quick start

1. Find a model ID that works in your region:

   ```bash
   bllm models -q claude
   ```

   ```
   jp.anthropic.claude-sonnet-4-5-20250929-v1:0                  JP Anthropic Claude Sonnet 4.5
   ```

2. Make it your default (or give it a short alias):

   ```bash
   export BLLM_MODEL=jp.anthropic.claude-sonnet-4-5-20250929-v1:0
   # or
   bllm aliases set sonnet jp.anthropic.claude-sonnet-4-5-20250929-v1:0
   ```

3. Prompt it:

   ```bash
   bllm "What is the capital of Japan?"
   bllm -m sonnet "What is the capital of Japan?"
   ```

## Usage

```
bllm [OPTIONS] [PROMPT]
bllm models [-q QUERY]
bllm aliases [list|set NAME MODEL|remove NAME]
bllm logs [-n COUNT]
```

### Prompting

| Option | Description |
|---|---|
| `-m, --model MODEL` | Model to use: an inference profile ID, model ID, application profile ARN, or alias |
| `-s, --system TEXT` | System prompt |
| `-c, --continue` | Continue the previous conversation |
| `--cid FILE` | Read and save the conversation in a specific file (implies `-c`) |
| `-t, --temperature N` | Sampling temperature |
| `-n, --max-tokens N` | Maximum output tokens (default 4096) |
| `--top-p N` | Top-p sampling |
| `-u, --usage` | Print input/output token counts to stderr |
| `--json` | Print the raw Converse API response instead of just the text |
| `--no-log` | Don't record this prompt in the log |
| `--region REGION` | AWS region (overrides `AWS_REGION`) |
| `--profile PROFILE` | AWS CLI profile (overrides `AWS_PROFILE`) |
| `-h, --help` | Show help |

### Piping input

As with `llm`, standard input is placed before the prompt argument, separated by a newline. If there is no argument, stdin alone is the prompt.

```bash
git diff | bllm "Write a commit message for this diff"
bllm < question.txt
pbpaste | bllm -s "Translate to English"
```

### Conversations

Every prompt is saved as the "last conversation". `-c` sends the full history along with your new prompt:

```bash
bllm "Suggest a name for a CLI that talks to Bedrock"
bllm -c "Shorter, please"
```

Use `--cid` to keep separate named threads:

```bash
bllm --cid ~/notes/design.json "Let's design a cache layer"
bllm --cid ~/notes/design.json "What about invalidation?"
```

### Listing models

```bash
bllm models              # all active inference profiles in the current region
bllm models -q nova      # filter by ID or name
bllm models --region us-east-1
```

Each line starts with a value you can paste straight into `-m`. The list includes:

- **System-defined cross-region inference profiles** such as `jp.*`, `apac.*`, `us.*`, `eu.*` and `global.*` IDs. Most current models can only be called through one of these.
- **Your application inference profiles**, shown by ARN.

Which profiles appear depends on the region you query. The list is not limited to text models, and it doesn't reflect whether model access has been enabled for your account.

### Aliases

```bash
bllm aliases set haiku jp.anthropic.claude-haiku-4-5-20251001-v1:0
bllm aliases list
bllm -m haiku "Hello"
bllm aliases remove haiku
```

Aliases also work in `BLLM_MODEL`.

### Logs

Every prompt and response is appended to a JSON Lines log.

```bash
bllm logs          # last 3 entries
bllm logs -n 10
```

The raw file is easy to query with `jq`:

```bash
jq -r 'select(.model | test("haiku")) | .prompt' ~/.config/bllm/logs.jsonl
```

## Configuration

| Variable | Description |
|---|---|
| `BLLM_MODEL` | Default model (ID, ARN or alias). Built-in default: `us.anthropic.claude-sonnet-4-5-20250929-v1:0` |
| `BLLM_DIR` | State directory. Default: `~/.config/bllm` |
| `AWS_REGION`, `AWS_PROFILE` | Standard AWS CLI settings |

The built-in default is a US profile. Outside US regions, set `BLLM_MODEL` to an ID from `bllm models`.

Files in the state directory:

| File | Contents |
|---|---|
| `aliases.json` | Alias → model mapping |
| `last.json` | Message history of the most recent conversation (used by `-c`) |
| `logs.jsonl` | One JSON object per prompt: timestamp, model, system prompt, prompt, response |

## IAM permissions

The calling identity needs at least:

- `bedrock:InvokeModel` on the inference profile and on the underlying foundation models in the regions it routes to (used by `converse`)
- `bedrock:ListInferenceProfiles` (used by `bllm models`)

## Troubleshooting

**`ValidationException: Invocation of model ID ... with on-demand throughput isn't supported`**
The model must be called through an inference profile. Use an ID from `bllm models` (e.g. `jp.anthropic...`) instead of the bare model ID.

**`AccessDeniedException`**
Model access isn't enabled for that model in the Bedrock console, or your IAM policy doesn't allow `bedrock:InvokeModel` for it.

**`The provided model identifier is invalid`**
The ID doesn't exist in the current region. Check `bllm models` with the same `--region`.

## Limitations

- **No streaming.** The AWS CLI doesn't support `converse-stream`, so the response is printed once it is complete.
- **Text only.** No image or document attachments, and no tool use.
- **Prompt arguments are joined with spaces.** `bllm "a" "b"` sends `a b`. Quote the prompt as one argument for exact text.
- **Conversations grow without limit.** `-c` resends the full history each time; start a new conversation when it gets long.

## License

Apache 2.0
