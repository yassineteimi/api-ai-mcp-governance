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
    fast in this space; everything here is **as of September 2026** and linked to a
    source; verify before quoting.

!!! warning "The hands-on run predates some of what is described here"
    The PoC itself was built in **June 2026** against **Traefik Hub 3.19 / Proxy
    3.7.5**. The landscape below is refreshed to September 2026, so it includes
    capabilities I did **not** exercise myself. Anything I ran is reported from
    captured output; anything I did not is attributed to a vendor source and labelled
    as such. Three things since the first draft materially changed the field: **IBM
    completed its $11B acquisition of Confluent on 17 March 2026**, **Traefik Hub 3.20
    shipped on 6 May 2026** with several of the things this PoC had deferred to a v2
    backlog, and **MCP itself went stateless on 28 July 2026**. Each is flagged in
    place below.

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

!!! note "Shipped since this PoC ran (Hub 3.20, 6 May 2026)"
    Three things I deferred to a v2 backlog or worked around have since landed, so
    treat the flags above as bounded by the 3.19 version I tested: an **AI token rate
    limit and quota middleware** with pre-request estimation, a **parallel LLM Guard**
    that runs guardrails concurrently instead of chaining them serially, and a
    **Content Guard regex engine** plus configurable `onDenyResponse` formats. The
    same release added **FIPS 140-3 support**, **multi-cluster API federation**, and
    **OpenAPI request body schema validation**. I have not exercised any of them; they
    are listed because leaving them out would make this page read as more damning than
    the current product deserves.

!!! danger "MCP went stateless on 28 July 2026, which dates this PoC's Gate 3"
    The [2026-07-28 MCP specification](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
    made the protocol **stateless at the protocol layer**: the
    `initialize` / `notifications/initialized` handshake is **removed**, protocol-level
    sessions and the `Mcp-Session-Id` header are gone from the Streamable HTTP
    transport, every request carries its protocol version and client capabilities in
    `_meta`, and list results now carry `ttlMs` / `cacheScope` so intermediaries can
    cache them. This PoC was built against the previous spec, and its TBAC policy
    **explicitly allow-lists `initialize` and `notifications/initialized`** because the
    policy language has no `NotEquals` (see [Gate 3](gates/mcp-gateway.md)). Those two
    rules are now vestigial.

    The interesting part is what it means for **gateways in general**, not just this
    one. A stateless MCP server can sit behind a plain round-robin load balancer, be
    routed on an `Mcp-Method` header, and have its `tools/list` cached, so the
    sticky-session and deep-inspection work gateways used to do for MCP largely goes
    away. Per-tool authorization, which is the whole point of Gate 3, becomes *more*
    important rather than less: it is now the main thing a gateway is uniquely
    positioned to enforce on this traffic. Re-running this benchmark against the new
    spec is the first item I would put in a v2.

## The landscape: Traefik Hub · IBM API Connect · WSO2 · Gravitee

A fair, point-in-time read of four credible platforms, two of them **French-rooted**
(Traefik, ex-Containous; and Gravitee, founded in Lille), which is itself relevant to
a sovereignty conversation. I know API Connect and WSO2 hands-on; the Traefik column
is this PoC; the Gravitee column is from public docs pending my own PoC.

| Dimension | **Traefik Hub** | **IBM API Connect** | **WSO2 APIM** | **Gravitee** |
| --- | --- | --- | --- | --- |
| Model / origin | Open-core; French-founded | Commercial (DataPower); **owns Confluent since 3/2026** | Open-core + subscription + Choreo SaaS; **EQT-owned since 2024** | Open-core; French-founded (Lille, 2014) |
| Config & GitOps | **CRD-native, GitOps-first** | Mgmt UI/CLI; GitOps add-on | UI + APICTL; config-as-code possible | UI + APIs; K8s Gateway API + GitOps |
| Footprint | **Light** (proxy + agent) | Heavy (Mgmt/Portal/Analytics/DataPower) | Medium-heavy | Medium |
| AI / LLM gateway | First-class: Content + LLM guards, parallel guard pipeline, token quota, semantic cache | **GA 2025**: LLM governance, rate/cost, caching, analytics | **API Platform GA 3/2026**: AI Gateway over models, MCP servers and prompts | Agent platform: identity/access, **agent mesh**, guardrails |
| MCP / agent governance | **TBAC** in front of MCP servers (per-identity, per-tool) | MCP via **API→MCP** + ContextForge proxy/guardrails | **MCP tool management** in the AI Gateway | **MCP** (APIs→MCP), **A2A**, agent identity/access |
| Event-native (Kafka/MQTT) | No (HTTP/gRPC) | **Yes, now the leader** (Confluent/Kafka + MQ) | Partial | **Yes, long-standing strength** |
| Beyond Kubernetes | VMs/Docker/Swarm/Nomad/file, **same features** | VM/appliance/container | VM/container/hybrid | VM/container/hybrid/K8s |
| Air-gap / sovereignty | **Offline mode GA** across API+AI+MCP; **FIPS 140-3** (Hub 3.20); self-host models | **Battle-tested** in regulated banks | OSS core **fully offline-able** | On-prem/hybrid; French-rooted trust |

Sources: IBM [AI Gateway announcement](https://www.ibm.com/new/announcements/how-an-ai-gateway-provides-greater-control-and-visibility-into-ai-services) & [API Connect MCP docs](https://www.ibm.com/docs/en/api-connect/software/12.1.0?topic=tools-ai-gateway-mcp), [ContextForge](https://github.com/IBM/mcp-context-forge), [Confluent acquisition completed 17 March 2026](https://newsroom.ibm.com/2026-03-17-ibm-completes-acquisition-of-confluent,-making-real-time-data-the-engine-of-enterprise-ai-and-agents); WSO2 [Choreo/subscription model](https://wso2.com/library/blogs/choreo-for-api-management/), [API Platform GA March 2026](https://wso2.com/about/news/wso2-launches-api-platform/) & [EQT acquisition completed](https://wso2.com/about/news/eqt-completes-acquisition-of-wso2/); Gravitee [AI agent platform](https://www.gravitee.io/platform/ai-agent-management) & [origin](https://siliconcanals.com/gravitee-io-raises-29-7m/); Traefik [multi-provider install](https://doc.traefik.io/traefik-hub/api-gateway/setup/installation/docker), [provider overview](https://doc.traefik.io/traefik-hub/api-gateway/reference/install/providers/ref-provider-overview), [offline mode](https://doc.traefik.io/traefik-hub/api-gateway/setup/installation/offline-mode), [platform-wide offline launch](https://traefik.io/press/traefik-labs-launches-mcp-gateway-nvidia-safety-nims-integration-and-platform-wide-offline-deployment) & [Hub 3.20, 6 May 2026](https://www.businesswire.com/news/home/20260506649151/en/Traefik-Labs-Makes-Ingress-NGINX-Replacement-GA-Adds-Multi-Cluster-API-Federation-and-Agent-Aware-AI-Controls).

**Read:** all four are credible; the differences are about *emphasis*, and the field
is moving monthly.

- **Traefik Hub** has the **most GitOps/CRD-native operating model** and the most
  opinionated, composable guard chain (deterministic PII → safety-model → per-tool
  TBAC). Lightest footprint.
- **IBM API Connect** ships a GA AI Gateway (2025) with LLM governance, cost
  controls and caching, plus an MCP story (turn APIs into MCP tools; ContextForge as
  an MCP/A2A proxy with guardrails). Its enduring edge is **incumbency and air-gap
  pedigree** in French banks/insurers, and the Confluent acquisition (below) has
  turned its weakest column into its strongest.
- **WSO2** is the **open-core, offline-friendly** choice sovereignty teams trust. Its
  **API Platform went GA in March 2026**, putting APIs, AI models, MCP servers and
  prompts under one control plane with an AI Gateway that manages MCP tools, so the
  "AI features are earlier" read I held in 2025 no longer holds. EQT has owned the
  company since 2024, which matters to buyers who weigh vendor ownership.
- **Gravitee** leans hard into **agent/MCP/A2A governance** (including an agent mesh)
  on top of a long-standing **event-native** gateway, with a **French origin** that
  plays well for sovereignty. Its event-native story is no longer uncontested, which
  makes the agent layer the more interesting part of its pitch.

!!! info "The consolidation that reshaped this table"
    **IBM completed its $11B acquisition of Confluent on 17 March 2026** (announced
    8 December 2025). Confluent is the commercial Kafka vendor, used by over 6,500
    enterprises, and the deal puts IBM at roughly **half the event broker and
    messaging market** when combined with MQ. Two consequences for this comparison:
    the "event-native" row, where IBM was previously the weakest of the four, is now
    where it is **strongest**; and **event-native alone is no longer a differentiator
    a challenger can lean on**, because the incumbent most French banks already run
    now owns the category leader. For a buyer already running MQ and API Connect,
    "real-time data plus API and AI governance from one vendor" became a much easier
    story to tell internally. That raises the bar for anyone displacing them: the
    argument has to be the **operating model** (GitOps, footprint, agent-native
    control), not the feature checklist.

The honest caveat: AI/agent feature sets across all four change fast, and this page
has already been overtaken once by two acquisitions and a major release. A real
head-to-head needs a **hands-on PoC per vendor, re-run at the time of the decision**,
which is exactly the method this project demonstrates.

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
       replacing the token in the gateway. Traefik's own framing when they launched
       platform-wide offline is that it runs "from Oracle Cloud to private
       datacenters to completely air-gapped military installations with identical
       features and performance", with the NVIDIA safety NIMs chaining into a
       multi-layered pipeline that also runs entirely offline.

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
- **FIPS 140-3** (added in Hub 3.20, May 2026): not an EU requirement, but it is the
  kind of cryptographic-module assurance that public-sector and defence-adjacent
  procurement checklists ask for, and it signals the vendor is investing in the
  certification track that regulated buyers screen on. Worth raising early, because
  a missing box on a procurement grid kills deals that technical evaluations pass.

### How I would pitch it

> *"For your AI and agent traffic, Traefik Hub gives you a single, GitOps-native
> control point (auth, PII and safety guards, cost governance, and per-tool agent
> authorization) with an audit trail that is your git history. For a sovereign or
> air-gapped estate we run it offline with self-hosted NIMs on your OpenShift GPUs;
> the policy model doesn't change. Where you already run API Connect and MQ for
> classic APIs and messaging, this is the **AI/agent-native layer** in front of your
> LLMs and MCP servers, not a rip-and-replace."*

Since IBM closed Confluent, expect the counter-pitch to be consolidation: one vendor
for streaming, APIs and AI governance. The honest answer is not to deny the pull of
that, but to separate the two decisions. Event streaming and agent-traffic governance
have different lifecycles and different blast radii, and the agent layer is the one
changing monthly. Buying it from the incumbent because the incumbent also sells Kafka
is how estates end up with a control point that cannot be changed independently of
the data backbone.

## Bottom line

Traefik Hub has one of the **cleanest cloud-native operating models** I've worked
with: GitOps/CRD-native, light, with a composable guard chain and per-tool TBAC that
is genuinely well-executed. It is **not uniquely ahead**, though, and less so than
when I first wrote this page: IBM API Connect ships a GA AI gateway with an MCP story
and now owns Confluent, **WSO2** shipped a unified API + AI + MCP platform in March
2026, and **Gravitee** pairs agent/MCP/A2A governance with an event-native gateway and
a French origin of its own. The differentiator is fit, not a single winner.

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
