# nanogpt-code: Claude Code with NanoGPT models (GLM, Kimi, DeepSeek, Qwen, MiniMax)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.8+](https://img.shields.io/badge/python-3.8%2B-blue.svg)](#install)
[![macOS | Linux](https://img.shields.io/badge/platform-macOS%20%7C%20Linux-lightgrey.svg)](#install)

Run [Claude Code](https://code.claude.com) on [NanoGPT](https://nano-gpt.com) models such as GLM 5.3, Kimi K3, DeepSeek V4, Qwen 3.8, MiniMax M3, uncensored and abliterated models, and hundreds more. Use the separate `nanogpt-code` command for NanoGPT, and your normal `claude` command and Claude subscription stay exactly as they are.

It's a single Python file with no dependencies. It handles the errors you hit when you point Claude Code at non-Claude models: unsupported `reasoning.effort` values, image inputs sent to text-only models, the "model isn't recognized" warning, and `/model` picks that leak into your normal Claude sessions.

```bash
nanogpt-code                                # Claude Code on GLM 5.3 via NanoGPT
nanogpt-code --model moonshotai/kimi-k3     # any NanoGPT model with tool calling
claude                                      # still your Claude subscription, untouched
```

## Contents

- [Why not just set ANTHROPIC_BASE_URL?](#why-not-just-set-anthropic_base_url)
- [Install](#install)
- [Quick start](#quick-start)
- [Choose your models](#choose-your-models)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [How it works](#how-it-works)
- [FAQ](#faq)
- [Uninstall](#uninstall)

## Why not just set ANTHROPIC_BASE_URL?

NanoGPT serves an Anthropic-compatible API, and its [official Claude Code setup](https://docs.nano-gpt.com/integrations/claude-code) writes your NanoGPT key and base URL into `~/.claude/settings.json`. That works if NanoGPT is all you use. If you also pay for Claude, every `claude` session now goes through NanoGPT instead of your subscription. Non-Claude models also run into problems that Claude Code wasn't built for:

| Problem | What you see | What nanogpt-code does |
|---|---|---|
| Your subscription gets bypassed | Plain `claude` bills NanoGPT | Sets NanoGPT's key and URL for its own process only |
| `/model` picks leak | Plain `claude` tries to start on a NanoGPT model | Puts your previous model back on exit, even if the terminal is closed |
| Unsupported effort level | `API Error: 400 Invalid value for reasoning.effort: "xhigh"` | Sends `high`, or the nearest level a model accepts |
| Images sent to text-only models | `API Error: 400 Model "…" does not support image inputs`, on every message after that | Blocks Read on image files while a text-only model is active, and tells the model why |
| Unknown context window | `Model "…" isn't recognized · context assumed to be 200k tokens` | Declares the window, so the warning goes away |
| Hand-maintained model lists | Model IDs and prices go out of date | Builds the `/model` picker from NanoGPT's live catalog |

## Install

You need [Claude Code](https://code.claude.com/docs/en/setup), Python 3.8 or newer (preinstalled on macOS), and a [NanoGPT API key](https://nano-gpt.com/api).

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/bicced/nanogpt-code/main/nanogpt-code -o ~/.local/bin/nanogpt-code
chmod +x ~/.local/bin/nanogpt-code
```

Make sure `~/.local/bin` is on your `PATH`. Or clone the repo and symlink `nanogpt-code` into any directory on your `PATH`.

## Quick start

```bash
nanogpt-code --set-key   # paste your NanoGPT key; it goes into the OS keychain
nanogpt-code --check     # confirms the key works and shows your balance
nanogpt-code             # start Claude Code on NanoGPT
```

Inside a session, `/model` lists your NanoGPT models with their context window, image support and price. Every `claude` flag works too: `nanogpt-code -p "..."`, `nanogpt-code --resume`, `nanogpt-code --model <id>`.

On macOS the key is stored in Keychain. On Linux it goes to the system keyring through `secret-tool` (libsecret). You can also set `NANOGPT_API_KEY` in your shell, which takes priority over the stored key.

## Choose your models

With no config file, the picker shows a starter set of open-weight coding models. The first one is the default:

```text
$ nanogpt-code --models
Picker list (built-in defaults); the first is the default:

 1. z-ai/glm-5.3
    GLM 5.3 · 1048K context · text only · $0.60/$1.10 per M tokens · effort high
 2. moonshotai/kimi-k3
    Kimi K3 · 1048K context · sees images · $1.20/$5.50 per M tokens · effort high
 3. deepseek/deepseek-v4-pro-0813
    DeepSeek V4 Pro 0813 · 1048K context · text only · $0.66/$1.98 per M tokens · effort high
 4. qwen/qwen3.8-max-0902
    Qwen3.8 Max 0902 · 991K context · sees images · $2.00/$6.00 per M tokens · effort high
 5. minimax/minimax-m3.1-flash
    MiniMax M3.1 Flash · 1000K context · sees images · $0.15/$0.60 per M tokens · effort high
 6. z-ai/glm-5.3-flash
    GLM 5.3 Flash · 1048K context · sees images · $0.10/$0.30 per M tokens · effort high
```

Prices and details are read live from NanoGPT's catalog (the output above is from October 2026). This is a starting list, not a benchmark ranking.

To use your own list, put one model ID per line in `~/.config/nanogpt-code/models`, best first. Lines starting with `#` are comments. IDs that aren't in NanoGPT's catalog, or that lack tool calling (Claude Code needs it), are left out with a warning.

```bash
mkdir -p ~/.config/nanogpt-code
curl -s https://nano-gpt.com/api/v1/models | python3 -m json.tool | grep '"id"'   # browse IDs
$EDITOR ~/.config/nanogpt-code/models
```

`examples/uncensored-models` is a ready-made list of uncensored and abliterated models with tool calling:

```bash
cp examples/uncensored-models ~/.config/nanogpt-code/models
```

How each model is used:

- **Default model**, also used for Claude Code's `opus` and `fable` aliases: the first in your list.
- **`sonnet` alias:** the second.
- **`haiku` alias** (background tasks): the cheapest.

Every alias points at NanoGPT, so subagents and background calls never reach Anthropic.

## Configuration

| Environment variable | Default | What it does |
|---|---|---|
| `NANOGPT_API_KEY` | keychain | API key; overrides the stored one |
| `NANOGPT_MODEL` | first in your list | Default model for the session |
| `NANOGPT_SMALL_MODEL` | cheapest in your list | Model for the `haiku` alias and background tasks |
| `NANOGPT_CONTEXT` | `200000` | Context window Claude Code assumes; it compacts the conversation at this size |

The context default is 200K even though many NanoGPT models accept 1M tokens, because a full context is billed again on every turn. At $3 per million input tokens, a full 200K context costs about $0.60 per turn and a full 1M context about $3. Raise it for a session with `NANOGPT_CONTEXT=1000000 nanogpt-code`. If any model in your list has a smaller window than 200K, that smaller window is used.

Other commands:

```bash
nanogpt-code --models    # picker list with context, image support, price and effort
nanogpt-code --refresh   # re-download NanoGPT's catalog (it's cached for 24 hours)
nanogpt-code --check     # test the API key and show your balance
nanogpt-code --help
```

## Troubleshooting

### `API Error: 400 Invalid value for reasoning.effort: "xhigh". Supported values are: low, high, max.`

Claude Code sends the effort level from your settings or `/effort`. NanoGPT rejects levels that some models don't support. nanogpt-code sends `high`, which nearly every NanoGPT model accepts, and for the few that don't it sends the nearest level they do accept. If you change `/effort` mid-session, pick a level shown under that model in `nanogpt-code --models`.

### `API Error: 400 Model "…" does not support image inputs. Remove image content or choose a multimodal model.`

An image got into the conversation. Often the model opened a screenshot with the Read tool, for example a debug PNG from a browser-automation script. Claude Code resends the whole conversation on every turn, so every message after that fails the same way. nanogpt-code blocks image reads while a text-only model is active, which stops this from happening. If a session is already stuck, press Esc twice (or run `/rewind`) to go back to before the image, or run `/clear`. Models marked "sees images" in `/model` can read images normally.

### `Model "…" isn't recognized · context assumed to be 200k tokens`

Claude Code doesn't know the context window of non-Claude model IDs. nanogpt-code sets `CLAUDE_CODE_MAX_CONTEXT_TOKENS`, so this warning doesn't appear. Change the value with `NANOGPT_CONTEXT`.

### `authentication_error: Invalid session` / `invalid_api_key` (HTTP 401)

NanoGPT doesn't accept your key, often because it was revoked or regenerated. Run `nanogpt-code --check`, which also tells you where the key was read from. A leftover `NANOGPT_API_KEY` in your shell overrides the keychain. Make a new key at [nano-gpt.com/api](https://nano-gpt.com/api) and run `nanogpt-code --set-key`.

### `claude.ai connectors are disabled because ANTHROPIC_API_KEY or another auth source is set`

This is expected. The session signs in with your NanoGPT key, not your claude.ai login, so connectors from your Claude account aren't loaded.

### `API Error: 400 Model … is temporarily unavailable`

NanoGPT can't currently reach that model's provider. Switch models with `/model` or try again later.

## How it works

`nanogpt-code` starts `claude` as a child process with:

- **Environment for that process only:** `ANTHROPIC_BASE_URL=https://nano-gpt.com/api`, your key as `ANTHROPIC_AUTH_TOKEN`, every model alias (`ANTHROPIC_DEFAULT_*_MODEL`) mapped to NanoGPT, and `CLAUDE_CODE_MAX_CONTEXT_TOKENS`. Your shell and settings files don't change.
- **Settings passed with `--settings`:** a `modelPicker` built from NanoGPT's catalog, `effortLevel` and per-model `modelSettings`, and a `PreToolUse` hook on Read (`nanogpt-code --hook-image-guard`). The hook finds the active model in the session transcript and blocks image files if NanoGPT lists that model as text-only.
- **On exit, including Ctrl-C, a closed terminal or `kill`:** if `/model` saved a NanoGPT model to `~/.claude/settings.json`, it puts back the model that was there before.

NanoGPT's catalog (`/api/v1/models?detailed=true`) is cached in `~/.cache/nanogpt-code/catalog.json` for 24 hours.

## FAQ

**Does this use my Claude Pro or Max subscription?**
No. A `nanogpt-code` session runs on your NanoGPT balance. Plain `claude` keeps using your subscription.

**Can I use Claude models through NanoGPT?**
Yes. Add IDs such as `anthropic/claude-sonnet-5.5` to your model list. They're billed to NanoGPT.

**Which NanoGPT model is best for Claude Code?**
Claude Code relies on tool calling, so only models with tool calling appear in the picker. Beyond that, try a few on your own tasks. Open-weight models are cheaper but follow Claude Code's long tool instructions less reliably than Claude.

**Does it work with uncensored or abliterated models?**
Yes, if the model supports tool calling. See `examples/uncensored-models`. Many of them are text-only, which is the case the image guard exists for.

**Windows?**
Not supported yet. It needs POSIX signals and a macOS or Linux keychain. On Windows, NanoGPT's official setup is the simplest option.

## Uninstall

```bash
rm ~/.local/bin/nanogpt-code
rm -rf ~/.config/nanogpt-code ~/.cache/nanogpt-code
security delete-generic-password -s nanogpt        # macOS
secret-tool clear service nanogpt                  # Linux
```

## Disclaimer

An independent project, not affiliated with or endorsed by Anthropic or NanoGPT. Claude and Claude Code are trademarks of Anthropic. Model availability, prices and behavior are set by NanoGPT and its providers.

## License

[MIT](LICENSE)
