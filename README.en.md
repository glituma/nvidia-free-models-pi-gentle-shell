# NVIDIA Free Models Profile for Gentle Shell

🇪🇸 [Leer en español](README.md)

A dedicated `gentle-shell` profile that uses only four free models from NVIDIA Build, intended for MVP and test projects.

> **Platform:** macOS with the fish shell. The Keychain step (3) and the alias (7) are macOS/fish specific.

| Model | ID | Context | Max output | Images |
|---|---|---|---|---|
| Kimi K3 (default) | `moonshotai/kimi-k3` | 1M | 131K | yes |
| GLM 5.3 | `z-ai/glm-5.3` | 1M | 131K | no |
| GLM 5.3 Flash | `z-ai/glm-5.3-flash` | 1M | 131K | yes |
| DeepSeek V4.1 Flash | `deepseek-ai/deepseek-v4.1-flash` | 131K* | 16K* | no |

\* Conservative values set manually. Pi ships no official metadata for this model.

---

## 0. Prerequisites

- Node.js and npm
- [pi](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) and [Gentle Shell](https://github.com/Gentleman-Programming/gentle-shell):

```fish
npm install -g @earendil-works/pi-coding-agent gentle-pi
```

Tested with pi `0.87.1` and gentle-pi `3.7.0`.

## 1. Create an NVIDIA account and API key

1. Go to <https://build.nvidia.com> and sign in or create an NVIDIA account (no credit card required).
2. Verify your email if prompted.
3. Open any model page (for example, Kimi K3) and click **Get API Key**, or go to your account's API keys section.
4. Generate a key. It starts with `nvapi-`. Copy it now, because it may not be shown again.

**Endpoint details**

- Base URL: `https://integrate.api.nvidia.com/v1`
- API format: OpenAI-compatible (`/chat/completions`)
- One key works for every model in the catalog.

**Limits and caveats**

- The free tier is for prototyping. Requests per minute are rate-limited. Production use requires a paid plan or self-hosted NIM.
- **Expect queues.** At peak times responses can take minutes or never arrive. In our tests on 2026-09-26, GLM 5.3 Flash took 78 s to reply "OK" and DeepSeek V4.1 Flash did not respond within 4 minutes. Kimi K3 responded normally.
- Prompts go through NVIDIA's servers. Never send sensitive data such as member records, credentials, or financial data.

## 2. Verify the models are available

The model catalog is public, so this check needs no key:

```fish
curl -s https://integrate.api.nvidia.com/v1/models \
  | python3 -c "import json,sys; [print(m['id']) for m in json.load(sys.stdin)['data'] if any(k in m['id'] for k in ['kimi','glm','deepseek'])]"
```

Expected to include:

```text
deepseek-ai/deepseek-v4.1-flash
moonshotai/kimi-k3
z-ai/glm-5.3
z-ai/glm-5.3-flash
```

## 3. Store the API key in the macOS Keychain

The key is never written to a config file. Pi reads it from the Keychain on every request.

```fish
security add-generic-password -a $USER -s nvidia-api-key -w
```

The command prompts for the key with hidden input.

Useful Keychain commands:

```fish
# Check it was stored (prints the key; avoid doing this while screen sharing)
security find-generic-password -a $USER -s nvidia-api-key -w

# Replace the key (rotation)
security add-generic-password -U -a $USER -s nvidia-api-key -w

# Remove the key
security delete-generic-password -a $USER -s nvidia-api-key
```

## 4. Test the API directly (optional)

```fish
set -x NVIDIA_API_KEY (security find-generic-password -a $USER -s nvidia-api-key -w)

curl -s https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Authorization: Bearer $NVIDIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"moonshotai/kimi-k3","messages":[{"role":"user","content":"Hello, who are you?"}],"max_tokens":200}'

set -e NVIDIA_API_KEY
```

## 5. How profiles work in Gentle Shell

`gentle-shell` wraps `pi`. Each profile is an **agent home**, a directory holding its own settings, models, credentials, and sessions.

| Home | Path | Selected with |
|---|---|---|
| Isolated (default) | `~/.gentle-shell/agent` | `gentle-shell` or `--isolated` |
| Vanilla pi | `~/.pi/agent` | `--link` |
| Custom | any directory | `--home <path>` |

Our profile is a custom home at `~/.gentle-shell/nvidia`. It is fully isolated: no credentials, sessions, or provider settings from the other profiles leak into it.

## 6. Create the profile

Quick install from this repository:

```fish
git clone https://github.com/glituma/nvidia-free-models-pi-gentle-shell.git
mkdir -p ~/.gentle-shell/nvidia
chmod 700 ~/.gentle-shell/nvidia
cp nvidia-free-models-pi-gentle-shell/profile/*.json ~/.gentle-shell/nvidia/
```

The two files are explained below.

### `~/.gentle-shell/nvidia/models.json`

Pi already ships a built-in `nvidia` provider with official metadata for Kimi K3, GLM 5.3, and GLM 5.3 Flash (context size, reasoning, image support). This file only:

1. Provides the API key through a Keychain command (the leading `!` tells pi to run a command).
2. Adds DeepSeek V4.1 Flash, which is not built in. Its `compat` flags mirror the built-in NVIDIA models.

```json
{
  "providers": {
    "nvidia": {
      "apiKey": "!security find-generic-password -a \"$USER\" -s nvidia-api-key -w",
      "models": [
        {
          "id": "deepseek-ai/deepseek-v4.1-flash",
          "name": "DeepSeek V4.1 Flash",
          "headers": { "NVCF-POLL-SECONDS": "3600" },
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 131072,
          "maxTokens": 16384,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "compat": {
            "supportsStore": false,
            "supportsDeveloperRole": false,
            "supportsReasoningEffort": false,
            "maxTokensField": "max_tokens",
            "supportsStrictMode": false
          }
        }
      ]
    }
  }
}
```

> Do not redefine the three built-in models here. A `models` entry replaces the built-in definition, and the official metadata is better than hand-written values.

### `~/.gentle-shell/nvidia/settings.json`

`enabledModels` restricts startup selection and `Ctrl+P` cycling to the four models, even though the built-in `nvidia` provider exposes about 80.

```json
{
  "defaultProvider": "nvidia",
  "defaultModel": "moonshotai/kimi-k3",
  "enabledModels": [
    "nvidia/moonshotai/kimi-k3",
    "nvidia/z-ai/glm-5.3",
    "nvidia/z-ai/glm-5.3-flash",
    "nvidia/deepseek-ai/deepseek-v4.1-flash"
  ],
  "theme": "Gentleman-Cute"
}
```

### Validate the profile

```fish
gentle-shell --home ~/.gentle-shell/nvidia --list-models | grep -E "provider|kimi-k3|glm-5.3|deepseek-v4.1"
```

Expected output:

```text
provider  model                            context  max-out  thinking  images
nvidia    deepseek-ai/deepseek-v4.1-flash  131.1K   16.4K    yes       no
nvidia    moonshotai/kimi-k3               1.0M     131.1K   yes       yes
nvidia    z-ai/glm-5.3                     1M       131.1K   yes       no
nvidia    z-ai/glm-5.3-flash               1M       131.1K   yes       yes
```

## 7. Create the alias

```fish
alias pinv 'gentle-shell --home ~/.gentle-shell/nvidia'
funcsave pinv
```

`funcsave` persists it to `~/.config/fish/functions/pinv.fish`, so it is available in every new shell.

## 8. Daily usage

```fish
cd ~/dev/my-mvp
pinv
```

Inside the session:

| Action | How |
|---|---|
| Cycle between the 4 models | `Ctrl+P` |
| Pick a model from a list | `/model` |
| Change the thinking level | `/thinking` |
| Save the current model as default | `Ctrl+S` in `/model` |

Suggested use:

- **Kimi K3**: general coding and agentic work (default).
- **GLM 5.3**: second opinion on complex tasks.
- **GLM 5.3 Flash / DeepSeek V4.1 Flash**: fast, cheap iterations and simple edits.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` | Key missing or wrong in Keychain | Re-run step 3 with `-U` |
| Keychain password prompt on each request | Keychain access not allowed for `security` | Choose **Always Allow** in the prompt |
| `429 Too Many Requests` | Free-tier rate limit | Wait a minute, or switch to another model |
| The model keeps thinking with no error | Free-tier queue: NVIDIA models send `NVCF-POLL-SECONDS: 3600`, so the request waits in queue up to 1 hour instead of failing | Cancel with `Esc`, switch models with `Ctrl+P`, and retry later |
| A model does not appear in `/model` | `enabledModels` typo or the model was removed from the catalog | Re-run the catalog check in step 2 |
| DeepSeek responses cut off | Conservative `maxTokens` | Raise `maxTokens` in `models.json` once real limits are known |

## Removal

```fish
rm -rf ~/.gentle-shell/nvidia
functions -e pinv; rm -f ~/.config/fish/functions/pinv.fish
security delete-generic-password -a $USER -s nvidia-api-key
```
