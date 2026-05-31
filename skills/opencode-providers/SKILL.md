---
name: opencode-providers
description: |
  Use this skill when configuring LLM providers in OpenCode, setting up API keys, connecting providers via /connect, configuring custom OpenAI-compatible providers, setting up local models (Ollama, llama.cpp, LM Studio), or troubleshooting provider authentication issues. Covers all 75+ supported providers, OpenCode Zen, OpenCode Go, provider-specific options, and environment variable configuration.
---

# OpenCode Providers

> **📚 Official Docs:** For the latest information, always refer to the official documentation:
> [https://opencode.ai/docs/providers/](https://opencode.ai/docs/providers/)

## Overview

OpenCode uses the [AI SDK](https://ai-sdk.dev/) and [Models.dev](https://models.dev) to support **75+ LLM providers** and local models. Providers are configured through `opencode.json` and authenticated via the `/connect` command.

**Quick start:**
1. Run `/connect` in the TUI and select your provider
2. Enter your API key when prompted
3. Run `/models` to select a model

### Credentials

API keys added via `/connect` are stored in `~/.local/share/opencode/auth.json`.

### Provider Config

Customize providers through the `provider` section in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "anthropic": {
      "options": {
        "baseURL": "https://api.anthropic.com/v1"
      }
    }
  }
}
```

**Key config fields:**
- `npm` — AI SDK package (e.g. `@ai-sdk/openai-compatible` for OpenAI-compatible endpoints, `@ai-sdk/openai` for `/v1/responses` endpoints)
- `name` — Display name in UI
- `options.baseURL` — API endpoint URL
- `options.apiKey` — API key (or use `/connect`)
- `options.headers` — Custom headers sent with each request
- `models` — Map of model IDs to configurations
- `limit.context` — Maximum input tokens the model accepts
- `limit.output` — Maximum tokens the model can generate

---

## OpenCode Zen

OpenCode Zen is a curated list of tested and verified models provided by the OpenCode team. It works like any other provider — pay-as-you-go, completely optional.

**Setup:**
1. Sign in at [opencode.ai/auth](https://opencode.ai/auth), add billing details, copy API key
2. Run `/connect`, select **OpenCode Zen**, paste API key
3. Run `/models` to see recommended models

**Free models available:**
- DeepSeek V4 Flash Free
- MiMo-V2.5 Free
- Nemotron 3 Super Free
- Big Pickle (stealth model)

**Pricing (per 1M tokens, selected models):**

| Model | Input | Output | Cached Read |
|-------|-------|--------|-------------|
| GPT 5.5 (≤272K) | $5.00 | $30.00 | $0.50 |
| GPT 5.4 | $2.50 | $15.00 | $0.25 |
| Claude Opus 4.8 | $5.00 | $25.00 | $0.50 |
| Claude Sonnet 4.6 | $3.00 | $15.00 | $0.30 |
| Gemini 3.5 Flash | $1.50 | $9.00 | $0.15 |
| Kimi K2.5 | $0.60 | $3.00 | $0.10 |
| MiniMax M2.7 | $0.30 | $1.20 | $0.06 |
| Qwen3.6 Plus | $0.20 | $1.20 | $0.02 |
| GPT 5 Nano | $0.05 | $0.40 | $0.005 |

**Features:**
- **Auto-reload**: Balance below $5 triggers $20 reload (configurable, can disable)
- **Monthly limits**: Set per-workspace and per-member spending caps
- **Teams**: Invite teammates, assign roles (Admin/Member), curate model access, bring your own keys
- **Privacy**: Zero-retention by default; free models may use data for improvement; OpenAI/Anthropic retain requests 30 days

**Model ID format in config:** `opencode/<model-id>` (e.g. `opencode/gpt-5.5`)

**Endpoints:**
- OpenAI models: `https://opencode.ai/zen/v1/responses`
- Anthropic models: `https://opencode.ai/zen/v1/messages`
- Google models: `https://opencode.ai/zen/v1/models/<model>`
- Other models: `https://opencode.ai/zen/v1/chat/completions`

---

## OpenCode Go

OpenCode Go is a low-cost subscription plan for popular open coding models.

- **$5 first month, $10/month** after that
- Models tested and verified for coding tasks

**Setup:**
1. Run `/connect`, select **OpenCode Go**, go to [opencode.ai/auth](https://opencode.ai/zen)
2. Sign in, add billing, copy API key
3. Paste API key in terminal
4. Run `/models` to see available models

**Usage limits:**
- 5 hour limit: $12
- Weekly limit: $30
- Monthly limit: $60

**Available models:**
- GLM-5
- GLM-5.1
- Kimi K2.5
- Kimi K2.6
- MiMo-V2.5
- MiMo-V2.5-Pro
- MiniMax M2.5
- MiniMax M2.7
- Qwen3.6 Plus
- Qwen3.7 Max
- DeepSeek V4 Pro
- DeepSeek V4 Flash

---

## Major Providers

### Anthropic (Claude)

Supports Claude Pro/Max OAuth and API keys.

**Setup:**
1. Run `/connect`, select **Anthropic**
2. Select **Claude Pro/Pro Max** for OAuth (opens browser) or **Manually enter API Key**
3. Run `/models` to select model

**Available models:** Claude Opus 4, Claude Sonnet 4, Claude Haiku 3.5, and others.

**Config example:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "anthropic": {
      "options": {
        "baseURL": "https://api.anthropic.com/v1"
      }
    }
  }
}
```

**Note:** Claude Pro/Max OAuth is the only supported way to use Anthropic subscriptions. Third-party plugins that use Claude subscriptions are explicitly prohibited by Anthropic and are no longer bundled with OpenCode as of v1.3.0.

### OpenAI

Supports ChatGPT Plus/Pro OAuth and API keys.

**Setup:**
1. Run `/connect`, select **OpenAI**
2. Select **ChatGPT Plus/Pro** for OAuth or **Manually enter API Key**
3. Run `/models` to select model

**Using API keys:** Select **Manually enter API Key** and paste your key.

### xAI

Supports SuperGrok OAuth, device-code flow, and API keys.

**Auth Method 1 — SuperGrok OAuth (browser):**
1. Run `/connect`, select **xAI**
2. Select **xAI Grok OAuth (SuperGrok Subscription)**
3. Browser opens for consent

**Auth Method 2 — SuperGrok device-code (headless/VPS):**
1. Run `/connect`, select **xAI**
2. Select **xAI Grok OAuth (Headless/Remote/VPS)**
3. Visit the verification URL on another device and enter the displayed code

**Auth Method 3 — API key:**
1. Create API key at [console.x.ai](https://console.x.ai)
2. Run `/connect`, select **xAI**
3. Select **Manually enter API Key** and paste your key

**Available models:** Grok Beta and others.

### GitHub Copilot

Uses device code flow. Requires GitHub Copilot subscription (some models need Pro+).

**Setup:**
1. Run `/connect`, search for **GitHub Copilot**
2. Navigate to `github.com/login/device` and enter the displayed code
3. Wait for authorization
4. Run `/models` to select model

```
┌ Login with GitHub Copilot
│
│ https://github.com/login/device
│
│ Enter code: 8F43-6FCF
│
└ Waiting for authorization...
```

### Google Vertex AI

Requires Google Cloud project with Vertex AI API enabled.

**Environment variables:**
```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
export VERTEX_LOCATION=global  # optional, defaults to global
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
```

**Or authenticate with gcloud:**
```bash
gcloud auth application-default login
```

**Config example:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "google-vertex": {
      "options": {
        "baseURL": "https://global-aiplatform.googleapis.com"
      }
    }
  }
}
```

**Tip:** The `global` region improves availability at no extra cost. Use regional endpoints (e.g. `us-central1`) for data residency requirements.

### Amazon Bedrock

Supports multiple authentication methods with configurable precedence.

**Environment variables:**
```bash
# Option 1: AWS access keys
export AWS_ACCESS_KEY_ID=XXX
export AWS_SECRET_ACCESS_KEY=YYY

# Option 2: Named AWS profile
export AWS_PROFILE=my-profile

# Option 3: Bedrock bearer token
export AWS_BEARER_TOKEN_BEDROCK=XXX
```

**Config example:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "us-east-1",
        "profile": "my-aws-profile",
        "endpoint": "https://bedrock-runtime.us-east-1.vpce-xxxxx.amazonaws.com"
      }
    }
  }
}
```

**Available options:**
- `region` — AWS region (e.g. `us-east-1`, `eu-west-1`)
- `profile` — Named profile from `~/.aws/credentials`
- `endpoint` — Custom endpoint URL for VPC endpoints (alias for `baseURL`)

**Authentication methods:**
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — IAM user access keys
- `AWS_PROFILE` — Named profiles from `~/.aws/credentials`
- `AWS_BEARER_TOKEN_BEDROCK` — Long-term API keys from Bedrock console
- `AWS_WEB_IDENTITY_TOKEN_FILE` / `AWS_ROLE_ARN` — EKS IRSA / Kubernetes OIDC

**Authentication precedence:**
1. Bearer Token (`AWS_BEARER_TOKEN_BEDROCK` or `/connect` token)
2. AWS Credential Chain (profile, access keys, shared credentials, IAM roles, Web Identity, instance metadata)

**Custom inference profiles:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "amazon-bedrock": {
      "models": {
        "anthropic-claude-sonnet-4.5": {
          "id": "arn:aws:bedrock:us-east-1:xxx:application-inference-profile/yyy"
        }
      }
    }
  }
}
```

### Azure OpenAI

Requires Azure OpenAI resource and model deployment.

**Setup:**
1. Create Azure OpenAI resource in Azure portal
2. Deploy a model in Azure AI Foundry (deployment name must match model name)
3. Run `/connect`, search for **Azure**
4. Enter API key
5. Set resource name:
```bash
export AZURE_RESOURCE_NAME=XXX
```
6. Run `/models` to select deployed model

**Note:** If you see "I'm sorry, but I cannot assist with that request" errors, change the content filter from **DefaultV2** to **Default** in your Azure resource.

**Azure Cognitive Services** uses a separate provider with `AZURE_COGNITIVE_SERVICES_RESOURCE_NAME`.

### GitLab Duo

Requires GitLab Premium or Ultimate subscription. Supports OAuth and personal access token.

**Setup:**
1. Run `/connect`, select **GitLab**
2. Choose authentication method:
   - **OAuth (Recommended):** Browser opens for authorization
   - **Personal Access Token:** Generate at GitLab User Settings > Access Tokens (scopes: `api`, token starts with `glpat-`)
3. Run `/models` to select model

**Available models:**
- `duo-chat-haiku-4-5` (default) — Fast responses
- `duo-chat-sonnet-4-5` — Balanced performance
- `duo-chat-opus-4-5` — Most capable

**Self-hosted GitLab:**
```bash
export GITLAB_INSTANCE_URL=https://gitlab.company.com
export GITLAB_TOKEN=glpat-...
export GITLAB_AI_GATEWAY_URL=https://ai-gateway.company.com  # optional
```

**Compliance config (lock to self-hosted):**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "small_model": "gitlab/duo-chat-haiku-4-5",
  "share": "disabled"
}
```

**OAuth for self-hosted:** Create application with callback URL `http://127.0.0.1:8080/callback` and scopes `api`, `read_user`, `read_repository`. Set `GITLAB_OAUTH_CLIENT_ID` env var.

**GitLab API tools plugin:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["opencode-gitlab-plugin"]
}
```

---

## Cloud Providers

### Cloudflare AI Gateway

Unified endpoint for OpenAI, Anthropic, Workers AI, and more.

**Setup:**
1. Create gateway in Cloudflare dashboard > AI > AI Gateway
2. Run `/connect`, search for **Cloudflare AI Gateway**
3. Enter Account ID, Gateway ID, and API token

**Environment variables:**
```bash
export CLOUDFLARE_ACCOUNT_ID=your-32-character-account-id
export CLOUDFLARE_GATEWAY_ID=your-gateway-id
export CLOUDFLARE_API_TOKEN=your-api-token
```

**Config:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "cloudflare-ai-gateway": {
      "models": {
        "openai/gpt-4o": {},
        "anthropic/claude-sonnet-4": {}
      }
    }
  }
}
```

### Cloudflare Workers AI

Run AI models on Cloudflare's global network via REST API.

**Environment variables:**
```bash
export CLOUDFLARE_ACCOUNT_ID=your-32-character-account-id
export CLOUDFLARE_API_KEY=your-api-token
```

### DigitalOcean

Supports OAuth (recommended) and Model Access Keys. Inference Routers route requests to optimal models.

**OAuth (recommended):**
1. Run `/connect`, select **DigitalOcean**, choose **Login with DigitalOcean**
2. Authorize in browser
3. Inference Routers appear as `router:<name>` in model picker

**Model Access Key:**
```bash
export DIGITALOCEAN_ACCESS_TOKEN=your-model-access-key
```

**Note:** Inference Routers are only auto-discovered with OAuth, not with Model Access Keys.

### Vercel AI Gateway

Unified endpoint for OpenAI, Anthropic, Google, xAI, and more. Models at list price.

**Setup:**
1. Create API key in Vercel dashboard > AI Gateway tab
2. Run `/connect`, search for **Vercel AI Gateway**
3. Enter API key

**Routing options:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "vercel": {
      "models": {
        "anthropic/claude-sonnet-4": {
          "options": {
            "order": ["anthropic", "vertex"],
            "only": ["anthropic"],
            "zeroDataRetention": false
          }
        }
      }
    }
  }
}
```

### SAP AI Core

Access 40+ models from OpenAI, Anthropic, Google, Amazon, Meta, Mistral, and AI21.

**Setup:**
1. Create service key in SAP BTP Cockpit
2. Run `/connect`, search for **SAP AI Core**
3. Enter service key JSON

**Environment variables:**
```bash
export AICORE_SERVICE_KEY='{"clientid":"...","clientsecret":"...","url":"...","serviceurls":{"AI_API_URL":"..."}}'
export AICORE_DEPLOYMENT_ID=your-deployment-id  # optional
export AICORE_RESOURCE_GROUP=your-resource-group  # optional
```

---

## AI Platform Providers

### OpenRouter

Multi-provider gateway with many preloaded models.

**Setup:**
1. Create API key at [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys)
2. Run `/connect`, search for **OpenRouter**
3. Enter API key

**Config — add models and provider routing:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openrouter": {
      "models": {
        "moonshotai/kimi-k2": {
          "options": {
            "provider": {
              "order": ["baseten"],
              "allow_fallbacks": false
            }
          }
        }
      }
    }
  }
}
```

### Together AI

1. Create account at [api.together.ai](https://api.together.ai), click **Add Key**
2. Run `/connect`, search for **Together AI**, enter key
3. Run `/models` to select

### Fireworks AI

1. Create account at [app.fireworks.ai](https://app.fireworks.ai/), click **Create API Key**
2. Run `/connect`, search for **Fireworks AI**, enter key
3. Run `/models` to select

### Groq

1. Create account at [console.groq.com](https://console.groq.com/), click **Create API Key**
2. Run `/connect`, search for **Groq**, enter key
3. Run `/models` to select

### Cerebras

1. Create account at [inference.cerebras.ai](https://inference.cerebras.ai/), generate API key
2. Run `/connect`, search for **Cerebras**, enter key
3. Run `/models` to select (e.g. Qwen 3 Coder 480B)

### DeepSeek

1. Create account at [platform.deepseek.com](https://platform.deepseek.com/), create API key
2. Run `/connect`, search for **DeepSeek**, enter key
3. Run `/models` to select (e.g. DeepSeek V4 Pro)

### Hugging Face

Inference Providers access via 17+ providers.

1. Create token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained) with Inference Providers permission
2. Run `/connect`, search for **Hugging Face**, enter token
3. Run `/models` to select

### NVIDIA

Free access to Nemotron and open models via [build.nvidia.com](https://build.nvidia.com).

1. Create account at build.nvidia.com, generate API key
2. Run `/connect`, search for **NVIDIA**, enter key
3. Run `/models` to select

**Environment variable:**
```bash
export NVIDIA_API_KEY=nvapi-your-key-here
```

**On-prem / NIM:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "nvidia": {
      "options": {
        "baseURL": "http://localhost:8000/v1"
      }
    }
  }
}
```

### Nebius Token Factory

1. Create account at [tokenfactory.nebius.com](https://tokenfactory.nebius.com/), click **Add Key**
2. Run `/connect`, search for **Nebius Token Factory**, enter key
3. Run `/models` to select

### Baseten

1. Create account at [app.baseten.co](https://app.baseten.co/), generate API key
2. Run `/connect`, search for **Baseten**, enter key
3. Run `/models` to select

### Deep Infra

1. Create account at [deepinfra.com/dash](https://deepinfra.com/dash), generate API key
2. Run `/connect`, search for **Deep Infra**, enter key
3. Run `/models` to select

### Venice AI

1. Create account at [venice.ai](https://venice.ai), generate API key
2. Run `/connect`, search for **Venice AI**, enter key
3. Run `/models` to select (e.g. Llama 3.3 70B)

### Moonshot AI

1. Create account at [platform.moonshot.ai/console](https://platform.moonshot.ai/console), create API key
2. Run `/connect`, search for **Moonshot AI**, enter key
3. Run `/models` to select (e.g. Kimi K2)

### MiniMax

1. Create account at [platform.minimax.io/login](https://platform.minimax.io/login), generate API key
2. Run `/connect`, search for **MiniMax**, enter key
3. Run `/models` to select (e.g. M2.1)

### Cortecs

1. Create account at [cortecs.ai](https://cortecs.ai/), generate API key
2. Run `/connect`, search for **Cortecs**, enter key
3. Run `/models` to select (e.g. Kimi K2 Instruct)

### IO.NET

1. Create account at [ai.io.net](https://ai.io.net/), generate API key
2. Run `/connect`, search for **IO.NET**, enter key
3. Run `/models` to select

### FrogBot

1. Create account at [app.frogbot.ai/signup](https://app.frogbot.ai/signup), generate API key
2. Run `/connect`, search for **FrogBot**, enter key
3. Run `/models` to select

### Z.AI

1. Create account at [z.ai/manage-apikey/apikey-list](https://z.ai/manage-apikey/apikey-list), create API key
2. Run `/connect`, search for **Z.AI**
3. If subscribed to GLM Coding Plan, select **Z.AI Coding Plan**
4. Enter API key
5. Run `/models` to select (e.g. GLM-4.7)

### ZenMux

1. Create API key at [zenmux.ai/settings/keys](https://zenmux.ai/settings/keys)
2. Run `/connect`, search for **ZenMux**, enter key
3. Run `/models` to select

### LLM Gateway

1. Create API key at [llmgateway.io/dashboard](https://llmgateway.io/dashboard)
2. Run `/connect`, search for **LLM Gateway**, enter key
3. Run `/models` to select

**Config example:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "llmgateway": {
      "models": {
        "glm-4.7": { "name": "GLM 4.7" },
        "gpt-5.2": { "name": "GPT 5.2" },
        "gemini-2.5-pro": { "name": "Gemini 2.5 Pro" }
      }
    }
  }
}
```

### STACKIT

Sovereign European AI model serving (Llama, Mistral, Qwen).

1. Create auth token in [STACKIT Portal](https://portal.stackit.cloud) > AI Model Serving
2. Run `/connect`, search for **STACKIT**, enter token
3. Run `/models` to select

### OVHcloud AI Endpoints

1. Create API key in OVHcloud panel > Public Cloud > AI & Machine Learning > AI Endpoints
2. Run `/connect`, search for **OVHcloud AI Endpoints**, enter key
3. Run `/models` to select

### Scaleway

1. Generate API key in [Scaleway Console IAM settings](https://console.scaleway.com/iam/api-keys)
2. Run `/connect`, search for **Scaleway**, enter key
3. Run `/models` to select

### 302.AI

1. Create account at [302.ai](https://302.ai/), generate API key
2. Run `/connect`, search for **302.AI**, enter key
3. Run `/models` to select

### Helicone

LLM observability platform with logging, monitoring, and analytics.

1. Create account at [helicone.ai](https://helicone.ai), generate API key
2. Run `/connect`, search for **Helicone**, enter key
3. Run `/models` to select

**Custom config with headers:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "helicone": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Helicone",
      "options": {
        "baseURL": "https://ai-gateway.helicone.ai",
        "headers": {
          "Helicone-Cache-Enabled": "true",
          "Helicone-User-Id": "opencode"
        }
      },
      "models": {
        "gpt-4o": { "name": "GPT-4o" },
        "claude-sonnet-4-20250514": { "name": "Claude Sonnet 4" }
      }
    }
  }
}
```

**Session tracking plugin:**
```bash
npm install -g opencode-helicone-session
```
```json
{ "plugin": ["opencode-helicone-session"] }
```

**Common Helicone headers:**

| Header | Description |
|--------|-------------|
| `Helicone-Cache-Enabled` | Enable response caching (`true`/`false`) |
| `Helicone-User-Id` | Track metrics by user |
| `Helicone-Property-[Name]` | Add custom properties |
| `Helicone-Prompt-Id` | Associate requests with prompt versions |

---

## Local Models

### Ollama

Ollama can auto-configure itself for OpenCode. See the [Ollama integration docs](https://docs.ollama.com/integrations/opencode).

**Manual config:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://localhost:11434/v1"
      },
      "models": {
        "llama2": { "name": "Llama 2" }
      }
    }
  }
}
```

**Tip:** If tool calls aren't working, increase `num_ctx` in Ollama to 16k-32k.

### Ollama Cloud

1. Sign in at [ollama.com](https://ollama.com/), go to Settings > Keys, create API key
2. Run `/connect`, search for **Ollama Cloud**, enter key
3. **Pull model info locally first:**
```bash
ollama pull gpt-oss:20b-cloud
```
4. Run `/models` to select cloud model

### llama.cpp

Uses llama-server utility from [llama.cpp](https://github.com/ggml-org/llama.cpp).

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "llama.cpp": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "llama-server (local)",
      "options": {
        "baseURL": "http://127.0.0.1:8080/v1"
      },
      "models": {
        "qwen3-coder:a3b": {
          "name": "Qwen3-Coder: a3b-30b (local)",
          "limit": {
            "context": 128000,
            "output": 65536
          }
        }
      }
    }
  }
}
```

### LM Studio

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "lmstudio": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LM Studio (local)",
      "options": {
        "baseURL": "http://127.0.0.1:1234/v1"
      },
      "models": {
        "google/gemma-3n-e4b": { "name": "Gemma 3n-e4b (local)" }
      }
    }
  }
}
```

### Atomic Chat

Desktop app that runs local LLMs behind an OpenAI-compatible API (default: `http://127.0.0.1:1337/v1`).

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "atomic-chat": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Atomic Chat (local)",
      "options": {
        "baseURL": "http://127.0.0.1:1337/v1"
      },
      "models": {
        "<your-model-id>": { "name": "<your-model-name>" }
      }
    }
  }
}
```

**Tip:** If tool calls aren't working, pick a model with strong tool-calling support (e.g. Qwen-Coder or DeepSeek-Coder).

---

## Custom Provider

Add any **OpenAI-compatible** provider not listed in `/connect`:

1. Run `/connect`, scroll to **Other**
2. Enter a unique provider ID
3. Enter your API key
4. Configure in `opencode.json`

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My AI Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1",
        "apiKey": "{env:MY_PROVIDER_API_KEY}",
        "headers": {
          "Authorization": "Bearer custom-token"
        }
      },
      "models": {
        "my-model-name": {
          "name": "My Model Display Name",
          "limit": {
            "context": 200000,
            "output": 65536
          }
        }
      }
    }
  }
}
```

**Configuration options:**
- `npm` — Use `@ai-sdk/openai-compatible` for `/v1/chat/completions` endpoints; use `@ai-sdk/openai` for `/v1/responses` endpoints
- `apiKey` — Set using `{env:VAR_NAME}` syntax or literal value
- `headers` — Custom headers sent with each request
- `limit.context` — Maximum input tokens
- `limit.output` — Maximum output tokens

The `limit` fields allow OpenCode to understand remaining context. Standard providers pull these from models.dev automatically.

---

## Environment Variables

| Variable | Provider | Description |
|----------|----------|-------------|
| `AWS_ACCESS_KEY_ID` | Amazon Bedrock | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Amazon Bedrock | AWS secret key |
| `AWS_PROFILE` | Amazon Bedrock | Named AWS profile |
| `AWS_REGION` | Amazon Bedrock | AWS region |
| `AWS_BEARER_TOKEN_BEDROCK` | Amazon Bedrock | Long-term Bedrock API key |
| `AWS_WEB_IDENTITY_TOKEN_FILE` | Amazon Bedrock | EKS IRSA token file |
| `AWS_ROLE_ARN` | Amazon Bedrock | EKS IRSA role ARN |
| `GOOGLE_CLOUD_PROJECT` | Google Vertex AI | GCP project ID |
| `GOOGLE_APPLICATION_CREDENTIALS` | Google Vertex AI | Service account JSON path |
| `VERTEX_LOCATION` | Google Vertex AI | Region (default: `global`) |
| `AZURE_RESOURCE_NAME` | Azure OpenAI | Azure resource name |
| `AZURE_COGNITIVE_SERVICES_RESOURCE_NAME` | Azure Cognitive Services | Resource name |
| `NVIDIA_API_KEY` | NVIDIA | API key |
| `DIGITALOCEAN_ACCESS_TOKEN` | DigitalOcean | Model access key |
| `GITLAB_TOKEN` | GitLab Duo | Personal access token |
| `GITLAB_INSTANCE_URL` | GitLab Duo | Self-hosted instance URL |
| `GITLAB_AI_GATEWAY_URL` | GitLab Duo | Custom AI Gateway URL |
| `GITLAB_OAUTH_CLIENT_ID` | GitLab Duo | OAuth app client ID (self-hosted) |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare | Account ID |
| `CLOUDFLARE_API_KEY` | Cloudflare Workers AI | API token |
| `CLOUDFLARE_API_TOKEN` | Cloudflare AI Gateway | API token |
| `CLOUDFLARE_GATEWAY_ID` | Cloudflare AI Gateway | Gateway ID |
| `AICORE_SERVICE_KEY` | SAP AI Core | Service key JSON |
| `AICORE_DEPLOYMENT_ID` | SAP AI Core | Deployment ID (optional) |
| `AICORE_RESOURCE_GROUP` | SAP AI Core | Resource group (optional) |

---

## Troubleshooting

1. **Check auth setup:** Run `opencode auth list` to verify credentials are stored. Does not apply to environment-variable-based providers (Amazon Bedrock, Google Vertex AI, etc.).

2. **Custom provider issues:**
   - Ensure provider ID in `/connect` matches the ID in `opencode.json`
   - Verify correct npm package:
     - `@ai-sdk/openai-compatible` for `/v1/chat/completions` endpoints
     - `@ai-sdk/openai` for `/v1/responses` endpoints
     - Provider-specific packages (e.g. `@ai-sdk/cerebras`) when available
   - Verify `options.baseURL` is correct

3. **Clear provider cache:** If models aren't appearing, try clearing the provider package cache and restarting.

4. **API call errors:** Check API key validity, endpoint URL, and model availability with the provider directly.

5. **Local model issues:** Ensure the local server (Ollama, llama.cpp, LM Studio) is running and accessible at the configured `baseURL`.
