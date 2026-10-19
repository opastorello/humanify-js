# humanify

> Un-minify JavaScript code using LLMs ("AI")

This tool uses large language models (like ChatGPT, Claude, Gemini, and
locally-hosted Ollama models) to unminify and rename minified JavaScript code.
The LLM only suggests new identifier names; the heavy lifting is done by
[oxc](https://github.com/oxc-project/oxc) at the AST level so the rewritten code
remains structurally identical to the input.

## Version 3 is out! 🎉

v3 highlights compared to v2:

* **Single static binary:** (Rust) — grab a binary from Releases. No Node, no
  npm, no Python.
* **Unix-style I/O**: read from stdin or a file, write to stdout or `-o <file>`.
  Combine easily with a deobfuscator like [webcrack][webcrack].
* **New providers:** Ollama (local), Anthropic and OpenRouter.

Check out the [v3 PR](https://github.com/jehna/humanify/pull/744) for more info.

[webcrack]:https://github.com/j4k0xb/webcrack

### ➡️ Check out the [introduction blog post][blogpost] for in-depth explanation!

[blogpost]: https://thejunkland.com/blog/using-llms-to-reverse-javascript-minification

## Example

Given the following minified code in `splitstring.min.js`:

```javascript
function a(e,t){var n=[];var r=e.length;var i=0;for(;i<r;i+=t){if(i+t<r){n.push(e.substring(i,i+t))}else{n.push(e.substring(i,r))}}return n}
```

Run:

```shell
humanify openai splitstring.min.js -o splitstring.js
```

Result (`splitstring.js`):

```javascript
function splitString(inputString, chunkSize) {
  var chunks = [];
  var stringLength = inputString.length;
  var startIndex = 0;
  for (; startIndex < stringLength; startIndex += chunkSize) {
    if (startIndex + chunkSize < stringLength) {
      chunks.push(inputString.substring(startIndex, startIndex + chunkSize));
    } else {
      chunks.push(inputString.substring(startIndex, stringLength));
    }
  }
  return chunks;
}
```

You can also pipe via stdin:

```shell
cat splitstring.min.js | humanify openai - > splitstring.js
```

To unbundle Webpack output first, pipe through `npx webcrack`:

```shell
npx webcrack < bundle.min.js | humanify openai - -o bundle.js
```

## Optional Jev / Laya decisions

Use a System One decision API to check whether an identifier already has a
meaningful name before asking your existing generative provider for a replacement.
This works with every provider (`openai`, `anthropic`, `gemini`, `ollama`,
`openrouter`, and `requesty`) and is **disabled by default**.

Use `humanify -h` for the command overview, or
`humanify openai -h` / `humanify openai --help` for the decision
options, defaults, environment variables and examples. The same options are
available in every provider subcommand.

For hosted [Jev](https://docs.typesafe.ai/api), configure a separate decision key:

```shell
export TYPESAFE_API_KEY=your-typesafe-key
humanify openai input.js --decision-provider jev -o output.js
```

For a local or self-hosted [Laya](https://github.com/NandhaKishorM/laya) server
running `laya-serve`:

```shell
humanify ollama input.js --decision-provider laya \
  --decision-base-url http://127.0.0.1:8000/v1 \
  --decision-model english -o output.js
```

The original provider still generates new names. Its model, URL and credentials
remain separate from the decision service's configuration.

| Option | Default / purpose |
| --- | --- |
| `--decision-provider jev\|laya` | Enable the optional preflight decision |
| `--decision-base-url URL` | Jev: `https://api.typesafe.ai/v1`; Laya: `http://127.0.0.1:8000/v1` |
| `--decision-model MODEL` | Jev: `jev-latest`; Laya: `english` (override for other checkpoints) |
| `--decision-api-key KEY` | Overrides `TYPESAFE_API_KEY` for Jev or `LAYA_API_KEY` for Laya; local Laya may omit authentication |
| `--decision-threshold NUMBER` | `0.95`; must be greater than `0.5` and at most `1` |
| `--decision-timeout-seconds N` | `5`; increase for slow servers or Laya cold starts |

The URL can also be the full `/v1/systemone` endpoint. Both services receive the
identifier, language and surrounding code through the same `POST /v1/systemone`
protocol. The gate asks a two-option `choice` question and skips generation only
when the selected **keep** option meets the threshold. It uses the option's
probability, not the providers' differently defined `confidence` fields. Tune
thresholds separately against representative code for each model.

An uncertain answer continues to the generative provider. A timeout, HTTP error,
or invalid decision disables preflight decisions for the rest of that run and
continues with the original provider, with one warning on stderr. `--verbose`
reports skipped generative calls without printing API keys. Decisions do not
change the parser's scope, reference or collision checks.

Speed improvements are workload-dependent: every evaluated identifier adds a
decision request, and only confident keeps save a generative request. This may
be slower on heavily minified code where most identifiers need new names.
Enabling a remote decision service sends it the same surrounding code used for
renaming. No speed or naming-quality improvement has been established by the
offline protocol tests.

### Apple Silicon with laya-mlx

If you have [laya-mlx](https://github.com/mizorewww/laya-mlx) installed, use the
included bridge to expose its Python API over the same HTTP protocol. Run it
with the Python interpreter that has `laya_mlx` installed:

```shell
python3 scripts/laya_mlx_server.py --offline \
  --model aac6fef/laya-multilingual-mlx
```

`--offline` uses cached weights only; omit it for the first download. The bridge
binds to `127.0.0.1:8000`, loads one checkpoint once, and serves predictions
serially on the GPU. Wait for `Ready` before running Humanify. In another terminal:

```shell
humanify openai input.js --decision-provider laya \
  --decision-base-url http://127.0.0.1:8000/v1 \
  --decision-model aac6fef/laya-multilingual-mlx -o output.js
```

The client model must exactly match the bridge's `--model`; it does not silently
substitute checkpoints. `--port`, `--device cpu` and `--dtype float32` are available
on the bridge. If you set `LAYA_API_KEY`, use the same value in the server and
Humanify environments. Stop the server with Ctrl+C. Check readiness with
`curl http://127.0.0.1:8000/health`.

The bridge needs no additional web framework. Test its protocol without MLX or
weights using `python3 -m unittest discover -s scripts -p 'test_*.py'`.

## Note on token usage

🚨 **NOTE:** 🚨

By default, humanify makes one LLM call per identifier in your code. For ChatGPT-class APIs
the cost roughly scales with the number of identifiers and the surrounding
context window (default 500 chars per call). A medium minified file (~500
identifiers) typically costs in the range of $0.10–$1.00 with OpenAI's small
models, free with the Gemini free tier, and free with Ollama or OpenRouter free
models.

For a rough character-count estimate of OpenAI mode:

```shell
echo "$((2 * $(wc -c < yourscript.min.js)))"
```

Using `humanify ollama` is free but slower; quality depends on your local model.
Free OpenRouter models (e.g. `qwen/qwen3-coder:free`) can help with your budget,
but expect them to be heaviy rate limited.

## Getting started

### Installation

The preferred way to install humanify is to download a pre-built binary from the
[latest release](https://github.com/jehna/humanify/releases/latest).

```shell
# macOS (Apple Silicon)
curl -L https://github.com/jehna/humanify/releases/latest/download/humanify-aarch64-apple-darwin.tar.gz | tar xz
sudo mv humanify /usr/local/bin/

# macOS (Intel)
curl -L https://github.com/jehna/humanify/releases/latest/download/humanify-x86_64-apple-darwin.tar.gz | tar xz
sudo mv humanify /usr/local/bin/

# Linux (x86_64)
curl -L https://github.com/jehna/humanify/releases/latest/download/humanify-x86_64-unknown-linux-gnu.tar.gz | tar xz
sudo mv humanify /usr/local/bin/

# Linux (aarch64)
curl -L https://github.com/jehna/humanify/releases/latest/download/humanify-aarch64-unknown-linux-gnu.tar.gz | tar xz
sudo mv humanify /usr/local/bin/

# Windows: download humanify-x86_64-pc-windows-msvc.zip from the releases page
```

Or build from source:

```shell
cargo install --git https://github.com/jehna/humanify
```

### Usage

```shell
humanify <openai|gemini|anthropic|ollama|openrouter|requesty> [FLAGS] <INPUT>
```

* `<INPUT>` is a file path or `-` for stdin.
* `-o <FILE>` writes to a file (default: stdout).
* `-m <MODEL>` overrides the preset's default model.
* `-k <KEY>` overrides the env-var-based API key.
* `--base-url <URL>` overrides the preset's base URL.
* `--context-size <N>` sets surrounding-code chars per identifier (default 500).
* `--json-mode <MODE>` pins a JSON-mode strategy. Options:
  `ladder` (default), `openai-json-schema`, `anthropic-native`,
  `forced-tool-call`, `tool-call-and-prompt`, `prompt`.
* `-v` prints resolved configuration and identifier-level rename steps to stderr.
* `--progress` shows an identifier progress bar on stderr.
* `--decision-provider jev|laya` enables optional name-quality decisions; see
  [Jev / Laya setup](#optional-jev--laya-decisions) for the separate API settings.

Run `humanify -h` for commands and `humanify <COMMAND> -h`
(or `--help`) for that provider's complete options and examples.

Note: humanify does one job — rename identifiers in one JavaScript file in,
one out. To unbundle webpack output first, pipe through e.g.
[webcrack](https://github.com/j4k0xb/webcrack):

```shell
npx webcrack < bundle.min.js | humanify openai - -o bundle.js
```

### OpenAI mode

You'll need an OpenAI API key. Sign up at https://openai.com/ and create a
key in the dashboard.

```shell
humanify openai obfuscated.js -o readable.js -k your-token
```

Or via environment variable:

```shell
export OPENAI_API_KEY=your-token
humanify openai obfuscated.js -o readable.js
```

Default model: `gpt-5-mini`. Override with `-m`.

### Gemini mode

You'll need a Google AI Studio key. Sign up at https://aistudio.google.com/.
Gemini's free tier is generous and is enough for most files.

```shell
export GEMINI_API_KEY=your-token
humanify gemini obfuscated.js -o readable.js
```

Default model: `gemini-3.1-flash-lite`. Override with `-m`.

### Anthropic mode

You'll need an Anthropic API key. Sign up at https://console.anthropic.com/.

```shell
export ANTHROPIC_API_KEY=your-token
humanify anthropic obfuscated.js -o readable.js
```

Default model: `claude-sonnet-4-6`. Override with `-m`.

The Anthropic preset uses Anthropic's native structured-outputs API
(`output_format: json_schema`) when available, falling back to forced
tool-calls if your account doesn't have the structured-outputs beta enabled.

### Local mode (Ollama)

Local mode runs against [Ollama](https://ollama.com/), which manages local LLM
weights and exposes an OpenAI-compatible API on `localhost:11434`. (pre-v3
migration note: There's no `humanify download` anymore — use a local inference
provider like Ollama to run your own models)

Prerequisites:
1. Install Ollama: <https://ollama.com/download>
2. Pull the recommended model: `ollama pull qwen3.5:4b`

Then run:

```shell
humanify ollama obfuscated.js -o readable.js
```

Default model: `qwen3.5:4b`. Override with `-m` to use any model you've
pulled. Local mode is free and private, but slower and less accurate than
the hosted providers; quality depends on the model you pick.

If you want to point humanify at a remote Ollama instance, override the
base URL:

```shell
humanify ollama obfuscated.js --base-url http://my-server:11434/v1
```

### OpenRouter mode

[OpenRouter](https://openrouter.ai/) routes requests across many backend
models. Useful for trying free-tier coding models without setting up
multiple accounts.

You'll need an OpenRouter API key. Sign up at https://openrouter.ai/.

```shell
export OPENROUTER_API_KEY=your-token
humanify openrouter obfuscated.js -o readable.js
```

Default model: `openai/gpt-oss-120b`. For free-tier usage:

```shell
humanify openrouter obfuscated.js -m qwen/qwen3-coder:free
```

### Requesty mode

[Requesty](https://requesty.ai/) provides an OpenAI-compatible router across
many backend models via a single API key.

You'll need a Requesty API key. Sign up at https://requesty.ai/.

```shell
export REQUESTY_API_KEY=your-token
humanify requesty obfuscated.js -o readable.js
```

Default model: `nvidia/nemotron-3-super-120b-a12b`. Override with `-m`:

```shell
humanify requesty obfuscated.js -m nvidia/nemotron-3-super-120b-a12b
```

## Features

* Optional [Jev / Laya decisions](#optional-jev--laya-decisions), with hosted,
  self-hosted and Apple Silicon MLX support; disabled by default.

* Uses LLMs to get smart suggestions to rename variable and function names, and
  make the rename using deterministic AST-level shenanigans via [oxc][oxc]
* Renames preserve all references and respect lexical scoping
* Reserved-word and collision-aware safe naming. The LLM's suggestion is
  normalised to a valid JS identifier and `_`-prefixed if it collides with
  an existing binding

[oxc]:https://github.com/oxc-project/oxc

## Contributing

If you'd like to contribute, please fork the repository and use a feature
branch. Pull requests are warmly welcome.

```shell
git clone https://github.com/jehna/humanify
cd humanify
cargo build
cargo test
```

CI runs `cargo fmt --check`, `cargo clippy -D warnings`, `cargo test` on
every PR. Local `ollama` and judge e2e suites also run on every PR.
The `gemini` e2e suite runs by default only for branches in this repository.
Other providers' e2e suites require both a branch in this repository and
their corresponding label (`test-openai`, `test-anthropic`,
`test-openrouter`, `test-requesty`) to avoid exposing secrets to forks or
burning API credits on every PR.

## Star History

<a href="https://www.star-history.com/?repos=jehna%2Fhumanify&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=jehna/humanify&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=jehna/humanify&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=jehna/humanify&type=date&legend=top-left" />
 </picture>
</a>

## Licensing

The code in this project is licensed under MIT license.
