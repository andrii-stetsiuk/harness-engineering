# abox

> One command. Full AI infrastructure.

`make run` gives you a local Kubernetes cluster with everything an AI project needs: an AI-aware API gateway, an agent runtime, observability, distributed tracing, and an eval harness — ready to use.

## What's included

| Component | Role |
|---|---|
| **agentgateway v2.2.1** | AI-aware API gateway (Gateway API–native, MCP-aware) |
| **kagent 0.10.1** | Kubernetes-native AI agent framework |
| **Qdrant 1.19.1** | Vector database for retrieval |
| **Qdrant MCP servers** | Custom embedding-aware tools and the official upstream memory server |
| **Neo4j 2026.7.1** | Graph database (`bolt://neo4j.neo4j:7687`) |
| **llm-d v0.3.17** | Distributed inference serving — llama.cpp + InferencePool/EPP, serving `nomic-embed-text-v1.5` |
| **llama.cpp** | Second, lightweight embeddings backend — same model as f16 GGUF |
| **Arize Phoenix 12.0.10** | LLM observability — tracing, evals, prompt playground |
| **Flux CD 2.x** | GitOps/GitLessOps operator — keeps the cluster in sync with OCI artifacts |
| **KinD** | Local Kubernetes (1 control-plane + 2 workers) - can be any k8s |
| **cloud-provider-kind** | LoadBalancer support so gateway gets a real IP for local development |

## Quickstart

```bash
make run
```

That's it. Installs OpenTofu and k9s, provisions the cluster, bootstraps Flux, and reconciles all components. When it finishes:

```bash
kubectl get gateway,httproute -A        # gateway is up
kubectl get agents -n kagent            # agent runtime is up
kubectl get svc -n agentgateway-system  # grab the LoadBalancer IP
```

Point your AI app at the gateway IP on port 80.

## How it works

```
make run  →  scripts/setup.sh
  → tofu apply (bootstrap/)
      → KinD cluster
      → Flux Operator + FluxInstance   via the upstream flux-operator-bootstrap module
      → ResourceSetInputProvider   polls the branch-specific GHCR OCI artifact
      → ResourceSet                creates OCIRepository + 2 Kustomizations
          → releases/crds/    gateway-api-crds, agentgateway-crds, kagent-crds
          → releases/         agentgateway (Gateway + GatewayClass)
                              kagent (agent runtime + HTTPRoute)
```

Everything after the cluster is **gitless GitOps via OCI**: no Git polling, no deploy keys. The root GitHub Actions workflow publishes `abox-llmd-embeddings/releases/` as an OCI artifact on every version tag. The cluster reconciles from that branch's artifact automatically.

`make run` bootstraps Flux to follow the branch-specific OCI artifact configured
by `bootstrap/variables.tf`; it does not apply the local `releases/` directory
directly. Publish a new release with `make push` before expecting local manifest
changes to appear in the cluster. Feature-branch artifact names must match the
suffix derived by the root `.github/workflows/flux-push.yaml` workflow. The
artifact package must be public for Flux to pull it without registry credentials.

## Releasing

```bash
make push   # bumps patch version, tags, pushes → CI publishes OCI artifact → cluster reconciles
```

> **Note:** RSIP tag sorting is lexicographic. If the patch version would exceed 9, bump the minor instead: `git tag vX.Y+1.0`.

## Directory layout

| Path | Purpose |
|---|---|
| `bootstrap/` | OpenTofu: KinD + Flux bootstrap (operator, instance, RSIP, ResourceSet) |
| `bootstrap/flux-instance.yaml` | `FluxInstance` applied by the bootstrap Job |
| `releases/crds/` | CRD HelmReleases: gateway-api, agentgateway, kagent, inference-extension |
| `releases/` | App HelmReleases + Gateway + HTTPRoutes |
| `images/nomic-embed/` | Dockerfile baking nomic-embed-text-v1.5 into llama.cpp server |
| `scripts/setup.sh` | Full setup script (`make run`) |
| `../.github/workflows/flux-push.yaml` | CI: publish `releases/` as OCI artifact on `v*` tags |

## Embeddings: two backends

Two OpenAI-compatible `/v1/embeddings` endpoints serving the same model
(`nomic-ai/nomic-embed-text-v1.5`), so llm-d can be evaluated against a plain
single-process server.

| | backend | in-cluster | via gateway |
|---|---|---|---|
| #1 | llm-d — llama.cpp behind an InferencePool + endpoint picker | `llm-d-embedding.llm-d:8000` | `http://<gw-ip>/llmd/v1/embeddings` |
| #2 | llama.cpp — one `llama-server --embeddings` pod | `llama-cpp-embeddings.llama-cpp:8090` | `http://<gw-ip>/llamacpp/v1/embeddings` |

Both backends run the **prebuilt image with the model baked in**
(`images/nomic-embed/`), so nothing is pulled from HuggingFace at pod start.
Backend #2 runs it directly; backend #1 mounts it as a Kubernetes **image
volume** (KEP-4639) and runs the stock `llama.cpp` server against it.
```
ghcr.io/den-vasyliev/abox/nomic-embed:v1.18.1-4ccc0ff
```

It is already published and public, so nothing has to be built to run this.
Two properties of it are worth knowing:

- **amd64 only** — a single manifest, not an index. Fine on Codespaces and any
  x86 node; it will not run natively on an arm64 Mac.
- Its baked `CMD` is only `--host/--port/--embeddings/--model`. The
  `--ctx-size`, `--ubatch-size` and `--parallel` values live on the Deployment
  and are **not** defaults — without them llama.cpp serves a single slot at a
  much smaller context.

To rebuild it, or to get an arm64 build, `images/nomic-embed/Dockerfile` builds
a multi-arch f16 equivalent from public sources:

```bash
gh workflow run "Build nomic-embed image"     # linux/amd64 + linux/arm64
```

A newly built package is **private on first push** and the kind nodes carry no
pull secret, so make it public or the pod sits in `ImagePullBackOff`.

Its args and resources are a production llama.cpp embedder sidecar's, verbatim.

```bash
GW=$(kubectl get svc -n agentgateway-system -o jsonpath='{.items[?(@.spec.type=="LoadBalancer")].status.loadBalancer.ingress[0].ip}')

curl -s "$GW/llmd/v1/embeddings" -H 'Content-Type: application/json' \
  -d '{"model":"nomic-ai/nomic-embed-text-v1.5","input":"search_document: hello"}' | jq '.data[0].embedding | length'

curl -s "$GW/llamacpp/v1/embeddings" -H 'Content-Type: application/json' \
  -d '{"model":"nomic-embed-text","input":"search_document: hello"}' | jq '.data[0].embedding | length'
```

Both return **768 dims** — neither truncates. Clients that want nomic's 256-dim
Matryoshka vectors do that reduction themselves. nomic also expects a
`search_document:` / `search_query:` task prefix on the input, as above.

Same model does not mean identical vectors: different kernels and quantization
move them slightly, so don't mix output from both into one collection.

### What to watch when evaluating

Both backends run the same engine on the same weights, so llm-d's cost is the
extra machinery; its benefit only shows up under concurrency and as models are
added.

```bash
kubectl get pods -n llm-d -n llama-cpp                       # readiness gap on first boot
kubectl get inferencepool,inferenceobjective -n llm-d        # pool + EPP wiring
kubectl logs -n llm-d deploy/llm-d-pool-epp                  # endpoint picker decisions
kubectl top pod -n llm-d; kubectl top pod -n llama-cpp       # steady-state cost
```

- **Both backends are llama.cpp**, same weights, same vectors. The comparison is
  *llm-d-managed* vs *plain Deployment*: what #1 adds is the InferencePool, the
  endpoint picker and the modelservice chart's prefill/decode shape.
- **Startup**: neither backend contacts HuggingFace. #2 carries the weights in
  its own image layer; #1 mounts them from an image volume, pulled once per node
  and cached by the kubelet like any other image.
- **Concurrency**: both are fixed at 8 slots (`--parallel 8`).
- **Scaling out**: raising `decode.replicas` puts real work in front of the EPP.
  At one replica the endpoint picker has nothing to choose between.

Note the `/llmd/v1/embeddings` route goes **straight to the decode Service**,
bypassing the EPP, and only the catch-all `/llmd` rule routes through the pool —
the same split the reference deployment uses. The EPP's scorers are built for decode/prefill
generation traffic, not a single-shot embed.

### Codespaces

Neither backend pulls from HuggingFace, but the nodes still pull images, so fix
egress **before** `make run` or every pull times out:

```bash
make fix-egress
```

A default 2-core/8GB Codespace is tight — the three kind nodes are containers
sharing that one host. Both backends are sized to fit it; raise the
requests/limits if you run somewhere with real headroom.

## Adding components

1. Put CRD charts in `releases/crds/` as HelmReleases.
2. Put app charts in `releases/` as HelmReleases.
3. Run `make push` — the cluster reconciles automatically.

The CRD kustomization runs first (`wait: true`), apps run after (`dependsOn: releases-crds`). This ordering is enforced by Flux and must be preserved.

### Qdrant MCP servers

`releases/mcp-servers.yaml` deploys two MCP servers in the `kagent` namespace:

| MCPServer | Implementation | Tools / collection |
|---|---|---|
| `qdrant-mcp` | Project's Go server; embeds through the cluster's llama.cpp endpoint | `vector_store`, `vector_find` / `abox-nomic` |
| `qdrant-official-mcp` | Official [Qdrant MCP server](https://github.com/qdrant/mcp-server-qdrant), launched with `uvx` | `qdrant-store-memory`, `qdrant-find-memories` / `abox-official-memory` |

The official server uses `sentence-transformers/all-MiniLM-L6-v2` through its
FastEmbed provider. It writes to the same Qdrant service, but its collection is
separate from `abox-nomic`; do not mix the two embedding spaces. It runs as an
MCP service, not a web app with its own UI. Check its status and logs with:

```bash
kubectl get mcpserver qdrant-official-mcp -n kagent
kubectl get pods,svc -n kagent | grep qdrant-official-mcp
kubectl logs -n kagent deploy/qdrant-official-mcp --tail=100
```

The Qdrant database dashboard is separate from the MCP server:

```bash
kubectl port-forward -n qdrant svc/qdrant 6333:6333
```

Then open http://localhost:6333/dashboard.

### Agent model

The kagent chart generates `default-model-config` from the provider settings
in `releases/kagent.yaml`. Both `retrieval-agent` and the chart-provided
`k8s-agent` use this shared config, which pins the OpenAI model to
`gpt-4.1-mini`. The API key is read from the `kagent-openai` Kubernetes Secret;
do not commit the key in the release manifests. Check the deployed model and
agent references with:

```bash
kubectl get modelconfig default-model-config -n kagent \
  -o jsonpath='{.spec.provider}{" / "}{.spec.model}{"\n"}'
kubectl get agents retrieval-agent k8s-agent -n kagent \
  -o custom-columns=NAME:.metadata.name,MODEL:.spec.declarative.modelConfig
```

The retrieval agent currently uses the custom `qdrant-mcp` tools
(`vector_store` and `vector_find`), not `qdrant-official-mcp`. The official
server is deployed independently and can be exposed to an agent by adding its
`MCPServer` reference and the desired tool names to that agent's manifest.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License

Apache 2.0 — see [LICENSE](./LICENSE).
