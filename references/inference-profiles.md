# NemoClaw Inference Profiles

## Routed Provider Model

NemoClaw keeps provider credentials on the **host**. The sandbox never receives raw API keys.

### How inference.local Works

`inference.local` is a special endpoint exposed inside every OpenShell sandbox:

```
Agent (sandbox) → https://inference.local → OpenShell proxy → strips sandbox creds → injects provider creds → upstream provider
```

- The proxy automatically handles credential management — sandbox doesn't need its own API keys
- A single provider+model configuration serves **all sandboxes on a gateway**
- Changes propagate within **~5 seconds** without recreating sandboxes
- Supported API patterns:
  - OpenAI: `/v1/chat/completions`, `/v1/completions`, `/v1/responses`, `/v1/models`
  - Anthropic: `/v1/messages`
  - Requests to unsupported patterns are denied

### Two Traffic Pathways

| Pathway | How It Works | Use Case |
|---------|-------------|----------|
| `inference.local` | Managed routing with credential injection | Primary inference (recommended) |
| External endpoints | Standard network policy allow/deny | Direct API access (requires network policy + own credentials) |

At onboard time, NemoClaw configures:
1. An OpenShell **provider** (credential + upstream endpoint)
2. An OpenShell **inference route** (sandbox → gateway → provider)
3. The baked OpenClaw **model reference** inside the sandbox

## Supported Providers

### Cloud Providers

| Provider | Provider Name | Endpoint Type |
|----------|--------------|---------------|
| NVIDIA Endpoints | `nvidia-prod` | OpenAI-compatible (`integrate.api.nvidia.com`) |
| OpenAI | `openai-api` | Native OpenAI-compatible |
| Anthropic | `anthropic-prod` | Native Anthropic (`anthropic-messages`) |
| Google Gemini | `gemini-api` | OpenAI-compatible (Google's endpoint) |
| Custom OpenAI-compat | `compatible-endpoint` | Custom OpenAI-compatible proxy/gateway |
| Custom Anthropic-compat | `compatible-anthropic-endpoint` | Custom Anthropic-compatible proxy/gateway |

### Local Providers

| Provider | Notes |
|----------|-------|
| Local Ollama | Auto-detected if running. Onboarding offers starter models, pulls and warms selected model. |
| Local NVIDIA NIM | Requires `NEMOCLAW_EXPERIMENTAL=1` |
| Local vLLM | Requires `NEMOCLAW_EXPERIMENTAL=1`. Port configurable via `VLLM_PORT` env var (default 8000 — most deployments use a non-default port like 8001 to avoid conflicts). |

All local providers use the same routed `inference.local` pattern — the upstream just runs on the host instead of in the cloud.

## Validation During Onboarding

NemoClaw validates the provider and model before creating the sandbox:

| Provider Type | Validation Method |
|---------------|-------------------|
| OpenAI-compatible | Tries `/responses` first, then `/chat/completions` |
| Anthropic-compatible | Tries `/v1/messages` |
| NVIDIA Endpoints (manual) | Validates model name against `https://integrate.api.nvidia.com/v1/models` |
| Compatible endpoints | Sends a real inference request (many proxies don't expose `/models`) |

If validation fails, the wizard does not continue to sandbox creation.

## Runtime Model Switching

Switch the active model without restarting the sandbox:

```bash
# NVIDIA Endpoints
openshell inference set --provider nvidia-prod --model nvidia/nemotron-3-super-120b-a12b

# OpenAI
openshell inference set --provider openai-api --model gpt-5.4

# Anthropic
openshell inference set --provider anthropic-prod --model claude-sonnet-4-6

# Google Gemini
openshell inference set --provider gemini-api --model gemini-2.5-flash

# Custom compatible endpoint
openshell inference set --provider compatible-endpoint --model <model-name>

# Custom Anthropic-compatible endpoint
openshell inference set --provider compatible-anthropic-endpoint --model <model-name>
```

### Important Notes on Switching

- Runtime switching changes the OpenShell route only
- It does NOT rewrite stored credentials
- It does NOT restart the sandbox
- The sandbox continues using `inference.local`
- You CAN switch between models within the same provider (e.g., `gpt-5.4` → `gpt-5`) and between already-configured providers (e.g., `nvidia-prod` → `openai-api` if both were set up during onboard)
- If you need to **add a new provider type** (one not configured during onboard), re-run `nemoclaw onboard`
- **Chat template must match model** — especially for tool calling. See `references/tool-calling.md` "Chat Template and Think-Block Interaction" and SKILL.md rule #15. Qwen3.5 MoE has inherent tool calling limitations due to think-block requirements.
- Verify after switching: `nemoclaw <name> status`

## Model Fallbacks

OpenClaw supports a `fallbacks` array in the model configuration (`openclaw.json`):

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "inference/main-model",
        fallbacks: ["inference/fallback-model"]
      }
    }
  }
}
```

**How fallbacks work:**
- If the primary model returns an error (timeout, 5xx, rate limit), OpenClaw retries with the first fallback
- Fallbacks are tried in order until one succeeds or all fail
- Each fallback model must be served by a configured provider
- In NemoClaw with local vLLM, fallbacks are limited — you typically serve one model per vLLM instance. Fallbacks are more useful with cloud providers (e.g., primary: NVIDIA Endpoints, fallback: OpenAI)
- Fallback configuration is in `openclaw.json` (Landlock read-only) — changes require sandbox rebuild

**Practical note:** For local inference on DGX Spark, fallbacks add little value since you're running a single model. Use fallbacks when mixing cloud and local providers, or when your primary cloud provider has availability issues.

## Local vLLM Configuration

For local vLLM inference (experimental):

```bash
NEMOCLAW_EXPERIMENTAL=1 \
VLLM_PORT=8001 \
nemoclaw onboard
```

- vLLM base URL becomes `http://host.openshell.internal:<VLLM_PORT>/v1`
- Model ID must match what vLLM serves — check your startup: `vllm serve --model <name>` (the `<name>` is what goes in `openclaw.json`)
- Provider type: `vllm-local` or `compatible-endpoint`

## Related References

- **`dgx-spark.md`** — Local vLLM setup on DGX Spark, port configuration, OOM avoidance
- **`tool-calling.md`** — Model compatibility for tool calling (llama.cpp issues, `--tool-call-parser` flag)
- **`openclaw-config.md`** — Full `openclaw.json` model configuration schema

## Context Window & OOM

- Configure `contextWindow` and `maxTokens` in sandbox configuration (`openclaw.json` — read-only, requires rebuild)
- If model switch causes crash: context may exceed new model's limit
- Use `/compact` inside sandbox to reclaim 80-90% of context window

### OOM Prevention (especially DGX Spark)

- Set `--gpu-memory-utilization 0.50` or lower when launching vLLM — CUDA graph compilation needs headroom on top of model weights
- Reduce `max-model-len` if context window is too large for available memory
- Emergency cache flush: `sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'`
- Monitor with `nvidia-smi` — watch for memory pressure before it OOMs
- See `references/dgx-spark.md` for Spark-specific OOM guidance
