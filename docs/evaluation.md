# Evaluation Notes

A practitioner's assessment after building the Triple Gate end-to-end, written
through the lens that matters for my market: **French and European regulated
institutions** (banks, insurers, public sector, health), and the stricter subset
that runs **air-gapped or sovereignty-constrained** estates. The benchmark is not
"does it work" (it does) but "would I put this in front of a regulated buyer, and
how does it sit next to the alternatives".

!!! abstract "Scope & method"
    The transferable asset here is a **reproducible benchmark methodology** for
    API/AI/MCP gateways, built hands-on against Traefik Hub, repeatable against any
    gateway. The competitor notes below are a **factual landscape**, not a scoreboard:
    each platform is credited for what it genuinely does well. Vendor claims move
    fast in this space; everything here is **as of late 2025** and linked to a
    source; verify before quoting.

!!! success "Reviewed with the vendor"
    An earlier revision of this page drew three conclusions from **trial defaults and
    documentation examples** rather than from the product: that the AI and MCP
    gateways were Kubernetes-first, that offline mode was an open question, and that
    guard observability needed assembly. Traefik reviewed the page and corrected all
    three; the text below reflects the corrections, with sources. Keeping the method
    honest matters more than keeping the first draft.

## What impressed me

- **CRD-native, GitOps-first by construction.** Every gate in this PoC is a YAML
  object reconciled by ArgoCD: auth, rate limits, AI guards, and **tool-level
  authorization** are all the same declarative substrate. The OSS→Hub upgrade was
  literally a git commit. For a regulated shop, that means **change is auditable by
  default**: every policy change is a reviewed, signed, revertible commit, not a
  click in a console.
- **One control plane for three problem classes.** API auth, LLM governance
  (PII/safety/cost), and MCP **agent** authorization sit behind one gateway with one
  policy model. That consolidation is rare and strategically interesting as agentic
  systems arrive.
- **Defense in depth that actually composes.** A prompt-injection that subverts the
  agent's reasoning is still stopped by TBAC at the tool boundary. "A compromised
  agent is not a compromised system" is a message that lands with a CISO.
- **Standards-aligned telemetry.** AI metrics follow the OpenTelemetry **GenAI
  semantic conventions** (token usage, model, cost), so observability is portable,
  not a proprietary lock-in. Guard decisions are part of that: both guards emit a
  blocking **`reason`** you name yourself, as a counter label and a span attribute.
- **Provider-agnostic, not Kubernetes-bound.** Hub is built on OSS Traefik Proxy, so
  the same dynamic configuration is expressed as **Kubernetes CRDs, Docker labels, or
  plain file-provider YAML**, and the AI and MCP gateways are configured the same way
  as anything else. The published examples lean Kubernetes-heavy, which is a
  documentation shape rather than a product limit. For a VM-only or Docker-only
  estate the capability travels with you.

## What I'd flag

- **AI and MCP metrics are OTLP-only, so plan the pipeline.** Request metrics sit on
  Traefik's Prometheus endpoint, but the GenAI and MCP metrics are emitted over
  **OTLP**. That is a deliberate OTel-native design rather than a gap: if you already
  run an OTel pipeline you plug straight in, and since Prometheus 3.0 you can
  [enable its native OTLP receiver](https://prometheus.io/docs/guides/opentelemetry/)
  (`--web.enable-otlp-receiver`) and skip the collector entirely. I ran a collector in
  this PoC because I was building the pipeline from nothing. Greenfield setup cost,
  not a product shortcoming.
- **The AI metrics are opt-in.** Every GenAI metric is enabled through the
  `observability.metrics` block on each AI middleware (`level: detailed`, or an
  `excludeList`). Sensible for cardinality control, but it means an unconfigured
  middleware looks quiet. Worth knowing before you conclude a signal is missing, which
  is exactly the mistake I made on my first pass.
- **One possible bug, reported.** On a **denied** MCP `tools/call`, I could not get the
  tool name populated on the metric, so deny-by-tool charts lean on `error_type` plus
  the route's `403`. Traefik's own read is that this may be an unexpected bug rather
  than intended behaviour, and it is being investigated. Flagged here as an open item,
  not a verdict.
- **Entitlement coupling at trial time.** AI, MCP and offline mode are separate
  license entitlements. My trial token carried none of the offline claim, which is
  why this PoC ran connected. Nothing unusual for commercial software, but scope the
  entitlements with sales up front so a PoC actually exercises the mode you intend to
  buy.

## The landscape: Traefik Hub · IBM API Connect · WSO2 · Gravitee

A fair, point-in-time read of four credible platforms, two of them **French-rooted**
(Traefik, ex-Containous; and Gravitee, founded in Lille), which is itself relevant to
a sovereignty conversation. I know API Connect and WSO2 hands-on; the Traefik column
is this PoC; the Gravitee column is from public docs pending my own PoC.

| Dimension | **Traefik Hub** | **IBM API Connect** | **WSO2 APIM** | **Gravitee** |
| --- | --- | --- | --- | --- |
| Model / origin | Open-core; French-founded | Commercial (DataPower) | Open-core + subscription + Choreo SaaS | Open-core; French-founded (Lille, 2014) |
| Config & GitOps | **CRD-native, GitOps-first** | Mgmt UI/CLI; GitOps add-on | UI + APICTL; config-as-code possible | UI + APIs; K8s Gateway API + GitOps |
| Footprint | **Light** (proxy + agent) | Heavy (Mgmt/Portal/Analytics/DataPower) | Medium-heavy | Medium |
| AI / LLM gateway | First-class: Content + LLM guards, token cost, semantic cache | **GA 2025**: LLM governance, rate/cost, caching, analytics | Emerging | Agent platform: identity/access, guardrails maturing |
| MCP / agent governance | **TBAC** in front of MCP servers (per-identity, per-tool) | MCP via **API→MCP** + ContextForge proxy/guardrails | Emerging | **MCP** (APIs→MCP), **A2A**, agent identity/access |
| Event-native (Kafka/MQTT) | No (HTTP/gRPC) | Limited | Partial | **Yes, core differentiator** |
| Beyond Kubernetes | VMs/Docker/Swarm/Nomad/file, **same features** | VM/appliance/container | VM/container/hybrid | VM/container/hybrid/K8s |
| Air-gap / sovereignty | **Offline mode GA** across API+AI+MCP; self-host models | **Battle-tested** in regulated banks | OSS core **fully offline-able** | On-prem/hybrid; French-rooted trust |

Sources: IBM [AI Gateway announcement](https://www.ibm.com/new/announcements/how-an-ai-gateway-provides-greater-control-and-visibility-into-ai-services) & [API Connect MCP docs](https://www.ibm.com/docs/en/api-connect/software/12.1.0?topic=tools-ai-gateway-mcp), [ContextForge](https://github.com/IBM/mcp-context-forge); WSO2 [Choreo/subscription model](https://wso2.com/library/blogs/choreo-for-api-management/); Gravitee [AI agent platform](https://www.gravitee.io/platform/ai-agent-management) & [origin](https://siliconcanals.com/gravitee-io-raises-29-7m/); Traefik [multi-provider install](https://doc.traefik.io/traefik-hub/api-gateway/setup/installation/docker), [provider overview](https://doc.traefik.io/traefik-hub/api-gateway/reference/install/providers/ref-provider-overview), [offline mode](https://doc.traefik.io/traefik-hub/api-gateway/setup/installation/offline-mode) & [platform-wide offline launch](https://traefik.io/press/traefik-labs-launches-mcp-gateway-nvidia-safety-nims-integration-and-platform-wide-offline-deployment).

**Read:** all four are credible; the differences are about *emphasis*, and the field
is moving monthly.

- **Traefik Hub** has the **most GitOps/CRD-native operating model** and the most
  opinionated, composable guard chain (deterministic PII → safety-model → per-tool
  TBAC). Lightest footprint.
- **IBM API Connect** ships a GA AI Gateway (2025) with LLM governance, cost
  controls and caching, plus an MCP story (turn APIs into MCP tools; ContextForge as
  an MCP/A2A proxy with guardrails). Its enduring edge is **incumbency and air-gap
  pedigree** in French banks/insurers.
- **WSO2** is the **open-core, offline-friendly** choice sovereignty teams trust, with
  API + event coverage; AI gateway features are earlier.
- **Gravitee** is, alongside Traefik, **closest to where the market is heading**:
  **event-native** *and* leaning hard into **agent/MCP/A2A governance**, with a
  **French origin** that plays well for sovereignty.

The honest caveat: AI/agent feature sets across all four change fast, so a real
head-to-head needs a **hands-on PoC per vendor**, which is exactly the method this
project demonstrates.

## Positioning for the French / regulated market

### Sovereignty & air-gap: the decisive axis

French regulated buyers increasingly require **on-prem or SecNumCloud-qualified
(ANSSI)** deployments, and the strictest (defense, some banking, sensitive public
sector) require **true air-gap**. Two things in *this* PoC leave the perimeter, and
the important point is that **neither is an architectural blocker**:

1. **Hub control-plane registration (`api.traefik.io`).** My trial token carried no
   offline entitlement, so this gateway ran in **connected** mode and the agent
   logged `Unable to ping platform api.traefik.io` whenever the laptop was offline.
   That log line is easy to misread, so it is worth being precise about what it does
   and does not mean:
     - **Client traffic never touches the platform.** The data plane is entirely
       self-hosted; requests, prompts and tool calls stay inside the perimeter.
     - **In connected mode the data plane keeps serving traffic** when the platform
       is unreachable. What it loses is the ability to *push configuration changes*
       from the online dashboard, which is a control-plane inconvenience, not a
       traffic outage.
     - **Offline mode is GA and covers the whole offering**, API, AI and MCP
       gateways alike. It is enabled by an **`offline` claim baked into the gateway
       token**, which the dashboard writes when you tick "Offline gateway" while
       **creating a new gateway**. You cannot flip an existing connected gateway
       offline by changing a Helm value: the `hub.offline` / `--hub.offline=true`
       setting only accompanies a token that already carries the claim. Licensing in
       that mode is validated locally against the token's expiry, so a renewal means
       replacing the token in the gateway.

    So for a sovereign estate this is an **entitlement and provisioning decision made
    at gateway-creation time**, not an open technical risk to litigate.

2. **Hosted NVIDIA NIM endpoint.** This PoC routes to `integrate.api.nvidia.com`
   for convenience. An air-gapped variant **self-hosts the models**: NVIDIA NIM
   containers (`nvcr.io/nim/...`) on on-prem GPUs (the same `nemoguard` guard model
   Traefik's own guide self-hosts), with the AI Gateway pointed at in-cluster
   Services. No architecture change; only the endpoints move inside the perimeter.

!!! note "The air-gapped reference architecture"
    Same three gates, but: the gateway **created as an offline gateway** from the
    start, **self-hosted NIMs** on on-prem/OpenShift GPUs, models and images mirrored
    into an internal registry, and the LLM Guard / Content Guard pointed at
    in-cluster endpoints. The GitOps and TBAC story is **identical**, which is the
    point.

### Regulatory fit (why a French CISO should care)

- **DORA** (in force 2025): the gateway is a natural **ICT control point**: third-
  party LLM risk, rate/quota for resilience, and an audit trail (every policy is a
  git commit) map directly onto operational-resilience obligations.
- **EU AI Act:** Content Guard + LLM Guard + per-tool authorization + token/guard
  telemetry give a concrete, demonstrable **governance posture** for AI systems.
- **GDPR / data residency:** deterministic **PII blocking before the model**, and
  EU-region or on-prem inference, address data-minimisation and residency.
- **ANSSI / SecNumCloud, HDS (health):** reachable with the offline +
  self-hosted-model architecture above. The gating item is **procurement** (securing
  the offline entitlement and provisioning the gateway as offline from day one),
  not a capability gap.

### How I would pitch it

> *"For your AI and agent traffic, Traefik Hub gives you a single, GitOps-native
> control point (auth, PII and safety guards, cost governance, and per-tool agent
> authorization) with an audit trail that is your git history. For a sovereign or
> air-gapped estate we run it offline with self-hosted NIMs on your OpenShift GPUs;
> the policy model doesn't change. Where you already run API Connect for classic
> APIs, this is the **AI/agent-native layer** in front of your LLMs and MCP servers,
> not a rip-and-replace."*

## Bottom line

Traefik Hub has one of the **cleanest cloud-native operating models** I've worked
with: GitOps/CRD-native, light, with a composable guard chain and per-tool TBAC that
is genuinely well-executed. It is **not uniquely ahead**, though: IBM API Connect now
ships a GA AI gateway with an MCP story, and **Gravitee** matches the modern direction
(event-native, agent/MCP/A2A) with a French origin of its own. The differentiator is
fit, not a single winner.

For the **mainstream French cloud-native** segment (banks modernising on
Kubernetes/OpenShift with GitOps in place) Traefik Hub is an easy thing to propose
today. For **air-gapped / SecNumCloud** institutions it is also proposable, with the
work sitting in procurement and provisioning (an offline entitlement, the gateway
created as offline, self-hosted NIMs on your GPUs) rather than in architecture, and
with the incumbent (API Connect) still worth keeping in the conversation for the
classic-API, already-certified estate. The most likely winning play is rarely
rip-and-replace; it's **the right AI/agent-native gateway alongside what's already
approved**, and which gateway that is deserves a hands-on PoC per vendor, not a
datasheet comparison.
