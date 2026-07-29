# LiteLLM Multi-Model Gateway

A production deployment of [LiteLLM](https://github.com/BerriAI/litellm) as an intelligent model gateway on AWS EKS, routing requests to self-hosted LLMs based on query complexity.

## Architecture

```
┌──────────────┐         ┌─────────────────────┐         ┌────────────────────────┐
│   Clients    │────────▶│   LiteLLM Proxy     │────────▶│  GLM-4-9B (vLLM)       │
│  (any app)   │         │  (Complexity Router) │         │  g6.2xlarge SPOT GPU   │
└──────────────┘         └──────────┬──────────┘         └────────────────────────┘
                                    │
                                    └────────────────────▶┌────────────────────────┐
                                                          │  Qwen2.5-14B-AWQ (vLLM)│
                                                          │  g5.xlarge ON-DEMAND   │
                                                          └────────────────────────┘
```

### How It Works

1. Client sends a request to LiteLLM using the `smart-router` model alias
2. LiteLLM's **complexity router** scores the request across 7 dimensions
3. Based on the score, the request is routed to the appropriate model tier
4. The response is streamed back to the client via OpenAI-compatible API

### Routing Tiers

| Tier | Model | Use Case | GPU Node |
|------|-------|----------|----------|
| SIMPLE | GLM-4-9B-Chat | Greetings, simple Q&A, translations | g6.2xlarge SPOT (~$0.50/hr) |
| MEDIUM | GLM-4-9B-Chat | General knowledge, short answers | g6.2xlarge SPOT |
| COMPLEX | Qwen2.5-14B-Instruct-AWQ | Coding, analysis, technical tasks | g5.xlarge ON-DEMAND (~$1.00/hr) |
| REASONING | Qwen2.5-14B-Instruct-AWQ | Multi-step reasoning, math, proofs | g5.xlarge ON-DEMAND |

### Complexity Scoring Dimensions

LiteLLM scores each request on:

| Dimension | Weight | What It Detects |
|-----------|--------|-----------------|
| tokenCount | 0.10 | Short prompts (<15 tokens) vs long (>400) |
| codePresence | 0.30 | "function", "class", "api", "database", language names |
| reasoningMarkers | 0.25 | "step by step", "think through", "analyze" |
| technicalTerms | 0.25 | "architecture", "distributed", "encryption" |
| simpleIndicators | 0.05 | "what is", "define", greetings |
| multiStepPatterns | 0.03 | "first...then", numbered steps |
| questionComplexity | 0.02 | Multiple question marks, nested questions |

### Keyword Rules (Deterministic Override)

Keyword rules run before the heuristic scorer:

```yaml
keyword_tier_rules:
  - keywords: ["hi", "hello", "thanks", "thank you", "bye"]
    tier: SIMPLE
  - keywords: ["translate", "翻译", "中文"]
    tier: SIMPLE
  - keywords: ["code", "function", "class", "algorithm", "debug", "refactor"]
    tier: COMPLEX
  - keywords: ["python", "javascript", "typescript", "java", "rust", "sql"]
    tier: COMPLEX
  - keywords: ["step by step", "analyze", "compare", "explain in detail"]
    tier: REASONING
  - keywords: ["architecture", "design pattern", "system design"]
    tier: REASONING
  - keywords: ["prove", "derive", "calculate", "equation"]
    tier: REASONING
```

When multiple keyword rules match, routing escalates to the **highest** tier.

---

## Deployment Guide

### Step 1: Create GPU Node Groups on EKS

```bash
# GPU node group for GLM-4 (SPOT - cheapest)
aws eks create-nodegroup \
  --cluster-name glm-chat \
  --nodegroup-name gpu-spot \
  --node-role <NODE_ROLE_ARN> \
  --subnets <SUBNET_IDS> \
  --instance-types g5.xlarge g5.2xlarge g6.xlarge g6.2xlarge \
  --capacity-type SPOT \
  --scaling-config minSize=1,maxSize=3,desiredSize=1 \
  --ami-type AL2_x86_64_GPU \
  --disk-size 100 \
  --labels workload=gpu-inference \
  --region us-east-1

# GPU node group for Qwen2.5-14B (ON-DEMAND)
aws eks create-nodegroup \
  --cluster-name glm-chat \
  --nodegroup-name gpu-ondemand-qwen \
  --node-role <NODE_ROLE_ARN> \
  --subnets <SUBNET_IDS> \
  --instance-types g5.xlarge g5.2xlarge g6.xlarge g6.2xlarge \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=1,maxSize=1,desiredSize=1 \
  --ami-type AL2_x86_64_GPU \
  --disk-size 100 \
  --labels workload=gpu-qwen3 \
  --region us-east-1
```

### Step 2: Install NVIDIA Device Plugin

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.0/deployments/static/nvidia-device-plugin.yml
```

### Step 3: Deploy GLM-4-9B

```bash
kubectl apply -f k8s/vllm-glm4.yaml
# Wait ~5-10 minutes for model download and loading
kubectl logs -f -n glm-chat -l app=inference-service
```

### Step 4: Deploy Qwen2.5-14B-AWQ

```bash
kubectl apply -f k8s/vllm-qwen3.yaml
# Wait ~5-7 minutes for model download and loading
kubectl logs -f -n glm-chat -l app=qwen3-service
```

### Step 5: Deploy LiteLLM Proxy

```bash
kubectl apply -f k8s/litellm-deployment.yaml
kubectl get pods -n glm-chat -l app=litellm-proxy
```

### Step 6: Test

```bash
# Auto-routed (smart-router decides)
curl -X POST http://<LITELLM_SERVICE>:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-glmchat-litellm-internal" \
  -H "Content-Type: application/json" \
  -d '{"model": "smart-router", "messages": [{"role": "user", "content": "Hello!"}]}'

# Explicit GLM-4
curl -X POST http://<LITELLM_SERVICE>:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-glmchat-litellm-internal" \
  -H "Content-Type: application/json" \
  -d '{"model": "glm-4", "messages": [{"role": "user", "content": "Hello!"}]}'

# Explicit Qwen2.5-14B
curl -X POST http://<LITELLM_SERVICE>:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-glmchat-litellm-internal" \
  -H "Content-Type: application/json" \
  -d '{"model": "qwen2.5-14b", "messages": [{"role": "user", "content": "Write quicksort in Python"}]}'
```

---

## Models Deployed

### GLM-4-9B-Chat

| Property | Value |
|----------|-------|
| HuggingFace | `THUDM/glm-4-9b-chat` |
| Parameters | 9B |
| Precision | bfloat16 |
| VRAM | ~17.6 GB |
| Context | 4,096 tokens |
| vLLM | v0.6.4 |
| Strengths | Fast, Chinese language, general chat |

### Qwen2.5-14B-Instruct-AWQ

| Property | Value |
|----------|-------|
| HuggingFace | `Qwen/Qwen2.5-14B-Instruct-AWQ` |
| Parameters | 14B (4-bit quantized) |
| Precision | INT4 AWQ |
| VRAM | ~9.4 GB |
| Context | 8,192 tokens |
| vLLM | v0.8.5 |
| Strengths | Coding, math, reasoning, analysis |

---

## Adding More Models

1. Deploy the model with vLLM (see existing manifests as templates)
2. Add to LiteLLM ConfigMap under `model_list`
3. Optionally update tier routing in `complexity_router_config`
4. `kubectl apply` the changes

---

## Cost Breakdown

| Component | Instance | Cost |
|-----------|----------|------|
| LiteLLM Proxy (2 pods) | Shared CPU nodes | ~$0 |
| GLM-4-9B | g6.2xlarge SPOT | ~$0.50/hr |
| Qwen2.5-14B | g5.xlarge ON-DEMAND | ~$1.00/hr |
| **Total** | | **~$1.50/hr (~$36/day)** |

---

## Session Affinity

LiteLLM pins multi-turn conversations to the same model, avoiding issues where:
- Response styles differ between turns
- Provider-side prompt caches are invalidated
- Model-specific tokens (thinking blocks) are replayed to a different model

Configure via:
```yaml
session_affinity: true
session_affinity_ttl_seconds: 3600
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| LiteLLM OOMKilled | Increase memory limit to 2Gi+ |
| vLLM model timeout | Increase liveness probe `initialDelaySeconds` to 300+ |
| CUDA OOM | Check model VRAM vs instance GPU memory |
| Spot interruption | Use ON-DEMAND for critical models |
| Empty LiteLLM logs | Check ConfigMap mounting and YAML syntax |

---

## Files

```
k8s/
├── litellm-config.yaml       # Standalone LiteLLM config (reference)
├── litellm-deployment.yaml   # ConfigMap + Deployment + Service
├── vllm-glm4.yaml           # GLM-4-9B vLLM deployment
└── vllm-qwen3.yaml          # Qwen2.5-14B-AWQ vLLM deployment
```

## License

MIT
