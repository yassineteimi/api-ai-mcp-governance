# Architecture

The PoC is one gateway enforcing **three gates** in front of three workload classes (a REST API, an LLM, and an
MCP/agent server), all reconciled from Git by ArgoCD on a single-node homelab. This page is the map: what each gate enforces, how a request flows
through them as **defense in depth**, the GitOps control loop, and the homelab topology
(including what leaves the perimeter, which is the crux of the [Evaluation](evaluation.md)).

## The three gates

| Gate | Host route | Enforces | Built from |
| --- | --- | --- | --- |
| **1. API** | `api.localhost` | Identity (JWT) + abuse control (rate limit) | [Gate 1](gates/api-gateway.md) |
| **2. AI** | `ai.localhost` | PII block, LLM safety, model/cost governance | [Gate 2](gates/ai-gateway.md) |
| **3. MCP** | `mcp.localhost` | Per-identity, per-tool authorization (TBAC) | [Gate 3](gates/mcp-gateway.md) |

Each gate is an `IngressRoute` plus an ordered chain of middlewares; the route's **Host**
selects the gate. The same Traefik Hub instance serves all three.

!!! info "Kubernetes here, not Kubernetes-only"
    Every gate below is expressed as a **Kubernetes CRD** because this PoC is
    GitOps-on-Kubernetes by design. Hub is built on OSS Traefik Proxy, so the same
    routers and middlewares (AI and MCP ones included) can be declared as
    [Docker labels or file-provider YAML](https://doc.traefik.io/traefik-hub/api-gateway/reference/install/providers/ref-provider-overview)
    instead. Read the CRDs as one dialect of a provider-agnostic configuration model,
    not as a Kubernetes prerequisite.

## Request flow: defense in depth

A request is screened by an *ordered chain*, and each gate refuses a **different class**
of abuse. A step that slips past one is still caught by the next, and crucially, the
last gate (MCP/TBAC) holds even if an attacker fully controls the agent's reasoning.

```mermaid
flowchart TB
  C([client / agent]) --> EP["Traefik entrypoint :80"]
  EP -->|"Host = api.localhost"| G1
  EP -->|"Host = ai.localhost"| G2
  EP -->|"Host = mcp.localhost"| G3

  subgraph G1["Gate 1 · API"]
    direction LR
    J1[api-jwt] --> RL[api-ratelimit] --> WH[whoami API]
  end
  subgraph G2["Gate 2 · AI"]
    direction LR
    CG[content-guard<br/>regex PII] --> LG[llm-guard<br/>nemoguard] --> CC[chat-completion<br/>inject key + pin model]
  end
  subgraph G3["Gate 3 · MCP"]
    direction LR
    J3[mcp-jwt] --> TB[mcp gateway<br/>TBAC policies] --> SP[strip-prefix] --> MS[ecommerce MCP]
  end

  CC -->|"OpenAI-compatible"| NIM{{NVIDIA NIM<br/>chat + guard model}}
```

| Attack | Stopped at | Why the others can't see it |
| --- | --- | --- |
| Anonymous / forged token | **Gate 1** (or 3) JWT | AI/MCP gates never receive the request |
| Exfiltrating PII via a prompt | **Gate 2** Content Guard | A valid token passes Gate 1; regex catches the pattern |
| Jailbreak to harmful output | **Gate 2** LLM Guard | Deterministic regex can't judge intent; the safety model can |
| Agent coerced into a privileged action | **Gate 3** TBAC | The prompt may be "clean"; only per-tool authz stops the *action* |

The end-to-end run lives in the [Unified Demo](unified-demo.md).

## GitOps control loop

Nothing is applied by hand. A **root app-of-apps** watches `poc/argocd/apps/` in the Git
repository; every gate, the gateway itself, and the observability stack are ArgoCD
`Application`s reconciled from Git. A change to any gate is a reviewed, revertible commit: **the audit trail is the git history**.

```mermaid
flowchart LR
  Git[("GitHub repo")] --> Root["ArgoCD<br/>triple-gate-root"]
  Root --> TF[traefik]
  Root --> A1[gate1-api]
  Root --> A2[gate2-ai]
  Root --> A3[gate3-mcp]
  Root --> OBS["observability-*<br/>(prometheus · grafana · otel · dashboards)"]
```

## Homelab topology

A single Mac runs everything via a Colima VM and a **kind** single-node cluster. Host
ports 80/443/8000 map to Traefik's NodePorts, so the gates are reachable at
`*.localhost`.

```mermaid
flowchart TB
  subgraph Host["macOS host · Colima VM"]
    subgraph KIND["kind cluster: triple-gate (k8s v1.36.1)"]
      subgraph NS_TRAEFIK["ns: traefik"]
        T[Traefik Hub gateway<br/>web/websecure + metrics:9100]
      end
      subgraph NS_APPS["ns: apps"]
        W[whoami] 
        M[ecommerce-mcp]
        X[nvidia-external<br/>ExternalName]
        SEC[(secrets: jwt · nvidia-nim<br/>traefik-hub-license)]
      end
      subgraph NS_ARGOCD["ns: argocd"]
        AG[ArgoCD]
      end
      subgraph NS_OBS["ns: observability"]
        P[Prometheus] 
        GR[Grafana]
        OT[OTel Collector]
      end
    end
  end
  GH[(GitHub repo)] --> AG
  AG -.->|reconciles| T
  AG -.->|reconciles| M
  AG -.->|reconciles| P
  T -->|Prometheus :9100| P
  T -.->|OTLP| OT --> P
  P --> GR
  T <-.->|"config pull (connected mode only)"| HUB{{Traefik Hub platform<br/>api.traefik.io}}
  X -->|HTTPS| NIM{{NVIDIA NIM<br/>integrate.api.nvidia.com}}
```

### Trust boundary: what leaves the perimeter

Two things cross the homelab boundary, and both are covered in the
[air-gapped analysis](evaluation.md#sovereignty-air-gap-the-decisive-axis):

1. **Traefik Hub control plane** (`api.traefik.io`): the agent registers and pulls
   configuration changes from the platform because this gateway was created as a
   **connected** gateway (my trial token carried no offline entitlement). **Client
   traffic never leaves the data plane**, and if the platform is unreachable the
   gateway keeps serving; it only stops receiving config pushes. A gateway created as
   an **offline gateway** in the dashboard drops this link entirely.
2. **NVIDIA hosted NIM** (`integrate.api.nvidia.com`) serves *both* the chat model
   and the safety guard. An air-gapped variant self-hosts these as in-cluster NIMs; the gate
   topology is unchanged; only the endpoints move inside.

!!! info "Why a single node is enough"
    The point of the PoC is the **policy and GitOps model**, not scale. Every gate, guard,
    and TBAC decision behaves identically on one kind node or a multi-node OpenShift
    cluster, which is exactly what makes the [air-gapped reference architecture](evaluation.md#sovereignty-air-gap-the-decisive-axis)
    a redeploy, not a redesign.

Build it for real in the [Bootstrap](bootstrap.md) chapter.
