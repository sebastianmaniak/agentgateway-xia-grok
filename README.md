# agentgateway-xia-grok

## What is agentgateway?

[agentgateway](https://agentgateway.dev) is an open-source, Rust-based data plane purpose-built for **agentic AI traffic** — LLM APIs, MCP servers, and agent-to-agent (A2A) communication. It speaks the OpenAI, Anthropic, Gemini, Bedrock, and Azure protocols natively, so a single endpoint can front any combination of providers while applying:

- **Auth** — passthrough, API key, OIDC/JWT, external authz, AWS/Azure/GCP IAM
- **Rate limiting & token-aware quotas** — per-user, per-model, per-route
- **Prompt guard & transformations** — request/response rewriting, prompt caching, model aliases
- **Observability** — OpenTelemetry traces, Prometheus metrics, structured access logs (gen_ai semantic conventions)
- **Routing** — model fallback, weighted splits, header/path matching via Gateway API

It runs two ways:

- **Standalone** — single static binary or container reading a YAML config file (this repo's Path A).
- **Kubernetes** — Gateway API conformant control plane + data plane, configured with `Gateway`, `HTTPRoute`, and `AgentgatewayBackend` CRDs (this repo's Path B).

The same proxy binary powers both modes, and the same policies apply regardless of how it's deployed.

---

Run xAI Grok behind agentgateway, two ways:

1. **Standalone** — agentgateway as a local container, OpenAI-compatible passthrough to `api.x.ai`.
2. **Kubernetes (kind)** — agentgateway installed in a local kind cluster with Gateway API resources.

Both paths consume your `XAI_API_KEY` env var and expose an OpenAI-compatible endpoint you can hit with `curl`.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/) v3
- An xAI API key

```bash
export XAI_API_KEY="xai-..."
```

---

## Path A — Standalone (local)

Reference: [openai-compatible providers](https://agentgateway.dev/docs/standalone/latest/llm/providers/openai-compatible/).

### 1. Write the config

```bash
cat > config.yaml <<'EOF'
# yaml-language-server: $schema=https://agentgateway.dev/schema/config
binds:
- port: 3000
  listeners:
  - routes:
    - backends:
      - ai:
          name: grok
          provider:
            openAI:
              model: grok-4-1-fast-non-reasoning
          hostOverride: api.x.ai:443
          policies:
            backendTLS: {}
EOF
```

### 2. Provide the API key

The OpenAI provider auto-detects `OPENAI_API_KEY` from the env — xAI uses a non-standard name, so re-export it:

```bash
export OPENAI_API_KEY="$XAI_API_KEY"
```

Alternatively, add an explicit reference in `config.yaml`:

```yaml
provider:
  openAI:
    model: grok-4-1-fast-non-reasoning
    apiKey:
      file: .secrets/xai
```

```bash
mkdir -p .secrets && printf '%s' "$XAI_API_KEY" > .secrets/xai
```

### 3. Run agentgateway

**Binary** ([install](https://github.com/agentgateway/agentgateway/releases)):
```bash
agentgateway -f config.yaml
```

**Docker:**
```bash
docker run --rm -it \
  -e OPENAI_API_KEY \
  -p 3000:3000 \
  -v "$PWD/config.yaml:/config.yaml" \
  ghcr.io/agentgateway/agentgateway:latest \
  -f /config.yaml
```

### 4. Test

```bash
curl -s http://localhost:3000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -d '{
    "model": "grok-4-1-fast-non-reasoning",
    "messages": [{"role":"user","content":"Say hi from Grok"}]
  }' | jq
```

---

## Path B — Kubernetes on kind

Reference: [agentgateway kubernetes quickstart](https://agentgateway.dev/docs/kubernetes/latest/quickstart/install/).

### 1. Create the kind cluster

```bash
kind create cluster --name agentgateway
kubectl cluster-info --context kind-agentgateway
```

### 2. Install Gateway API CRDs

```bash
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```

### 3. Install agentgateway

```bash
helm upgrade -i agentgateway-crds \
  oci://cr.agentgateway.dev/charts/agentgateway-crds \
  --create-namespace --namespace agentgateway-system \
  --version v1.2.0 \
  --set controller.image.pullPolicy=Always

helm upgrade -i agentgateway \
  oci://cr.agentgateway.dev/charts/agentgateway \
  --namespace agentgateway-system \
  --version v1.2.0 \
  --set controller.image.pullPolicy=Always \
  --set controller.extraEnv.KGW_ENABLE_GATEWAY_API_EXPERIMENTAL_FEATURES=true \
  --wait

kubectl get pods -n agentgateway-system
```

Sanity check the installed CRDs:
```bash
kubectl api-resources --api-group=agentgateway.dev
# Expect: AgentgatewayBackend, AgentgatewayPolicy, AgentgatewayParameters
```

### 4. Create the Gateway

```bash
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
EOF

kubectl get gateway agentgateway-proxy -n agentgateway-system -w
```

Wait for `PROGRAMMED=True`.

### 5. Store the API key in a Secret

The OSS `AgentgatewayBackend` `secretRef` expects the key to live under the `Authorization` field, with the `Bearer ` prefix included.

```bash
kubectl create secret generic grok-secret \
  -n agentgateway-system \
  --from-literal=Authorization="Bearer ${XAI_API_KEY}"
```

### 6. Create the Grok backend

```bash
kubectl apply -f - <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: grok
  namespace: agentgateway-system
spec:
  ai:
    groups:
    - providers:
      - name: grok
        host: api.x.ai
        port: 443
        openai:
          model: grok-4-1-fast-non-reasoning
        policies:
          auth:
            secretRef:
              name: grok-secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: BackendTLSPolicy
metadata:
  name: grok-tls
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: agentgateway.dev
    kind: AgentgatewayBackend
    name: grok
  validation:
    wellKnownCACertificates: System
    hostname: api.x.ai
EOF
```

### 7. Route `/grok/*` at the backend

```bash
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grok
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /grok
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: grok
      group: agentgateway.dev
      kind: AgentgatewayBackend
EOF
```

### 8. Port-forward and test

```bash
kubectl port-forward -n agentgateway-system deployment/agentgateway-proxy 8080:80 &

curl -s http://localhost:8080/grok/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "grok-4-1-fast-non-reasoning",
    "messages": [{"role":"user","content":"Say hi from Grok via agentgateway on kind"}]
  }' | jq
```

---

## Cleanup

**Standalone:** stop the container.

**Kind:**
```bash
kind delete cluster --name agentgateway
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `no matches for kind "Backend"` | Use `AgentgatewayBackend` — that's the OSS kind. Check with `kubectl api-resources --api-group=agentgateway.dev`. |
| `no matches for kind "BackendTLSPolicy" ... v1alpha3` | Use `gateway.networking.k8s.io/v1` (GA in Gateway API v1.5+). |
| `401 Unauthorized` from xAI | Secret missing or wrong key. The data field must be `Authorization`, value `Bearer xai-...`. |
| `no healthy upstream` | `kubectl describe agentgatewaybackend grok -n agentgateway-system` and check status conditions. |
| TLS handshake error to `api.x.ai` | Confirm `BackendTLSPolicy` is applied and `hostname: api.x.ai` matches. |
| `404 The requested resource was not found` from xAI | The `/grok` prefix is being forwarded to xAI. Add the `URLRewrite` filter with `ReplacePrefixMatch: /` (or set `path: /v1/chat/completions` on the backend provider). |
| Gateway stuck `PROGRAMMED=False` | CRDs not installed in the right order — re-run step 2 then step 3. |
