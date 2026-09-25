# Three Buzz agents in three Kata-backed Pods

> **Assumption:** “Buzz” means [Block’s Buzz](https://github.com/block/buzz). This is a proposed design, not a created repository, account, Secret, or Pod. Buzz identifies each agent by a Nostr keypair, not a username/password bot account. [OpenClaw’s official Buzz plugin](https://docs.openclaw.ai/channels/buzz) can connect a runnable agent to a room; it does not support DMs or automatically approve room access.

## Proposed repository and agent descriptions

```text
three-buzz-agents/
├── agents/
│   ├── triage/AGENTS.md
│   ├── analyst/AGENTS.md
│   └── reviewer/AGENTS.md
├── openclaw/
│   ├── triage.json5
│   ├── analyst.json5
│   └── reviewer.json5                    # non-secret templates
├── image/                                  # pinned OpenClaw + Buzz build
└── deploy/{sandboxes,deployments}/         # choose one workload route
```

The three `AGENTS.md` files would contain these actual operating instructions:

**`agents/triage/AGENTS.md`**
```markdown
# Triage
Classify an incoming human request by subject, urgency, impact, and missing facts.
Return a queue recommendation, rationale, and questions for the requester.
Suggest a human-led analyst handoff when evidence is needed. Do not edit or approve work.
```

**`agents/analyst/AGENTS.md`**
```markdown
# Analyst
Investigate the supplied question using available evidence.
Distinguish observation from inference and cite the sources for material claims.
Return options and trade-offs. Do not make operational changes.
```

**`agents/reviewer/AGENTS.md`**
```markdown
# Reviewer
Independently check supplied work for correctness, risks, and missing cases.
Report findings by severity with supporting evidence and a clear disposition.
Do not edit, merge, deploy, or claim final approval.
```

These are independent roles, not an automatic triage → analysis → review pipeline. OpenClaw loads `AGENTS.md` from each agent’s workspace; the text is guidance, **not** a tool-access control. Configure actual tools and filesystem/network permissions separately ([workspace contract](https://docs.openclaw.ai/concepts/agent-workspace), [multi-agent routing](https://docs.openclaw.ai/concepts/multi-agent)). No Kubernetes controller executes these files by itself.

For the **three-Pod** variant, mount only the matching role’s workspace in each Gateway. Each non-secret config template is copied to that Gateway’s `~/.openclaw/openclaw.json`; the role’s state/session directory remains on its **own** persistent volume. A representative *incomplete* `triage.json5` fragment shows the native OpenClaw fields; replace the relay and room identifiers and add the chosen model and Gateway authentication before use:

```json5
{
  gateway: { mode: "local" },
  agents: { defaults: { workspace: "/home/node/.openclaw/workspace", skipBootstrap: true } },
  channels: { buzz: {
    name: "Triage",
    relayUrl: "wss://REPLACE_WITH_RELAY",
    privateKey: { source: "env", provider: "default", id: "BUZZ_PRIVATE_KEY" },
    groupPolicy: "open",
    groups: { "REPLACE_WITH_ROOM_UUID": {} },
  } },
}
```

This initial policy accepts current members of that **one approved room**. Tighten it with `groupAllowFrom` and/or `requireMention` as appropriate; mentions require a Buzz client that can address the bot ([Buzz access control and manual config](https://docs.openclaw.ai/channels/buzz#access-control)). A shared Gateway instead needs three `agents.entries` workspaces, three `channels.buzz.accounts` identities, and account-specific `bindings` ([multiple identities](https://docs.openclaw.ai/channels/buzz#multiple-bot-identities)).

## Give each agent a Buzz identity

Repeat for **triage, analyst, reviewer**; use three different keypairs and preferably three separate rooms. A Buzz owner/room admin and the relay URL are prerequisites.

1. **Build the connector.** Pin compatible OpenClaw and `@openclaw/buzz` versions (the plugin declares `>=2026.9.6`). Build a Linux image from OpenClaw source with `OPENCLAW_EXTENSIONS=buzz`, publish and deploy its immutable digest; the ordinary prebuilt image need not contain Buzz ([plugin metadata](https://github.com/openclaw/openclaw/blob/main/extensions/buzz/package.json), [image build](https://docs.openclaw.ai/install/docker#source-built-images-with-selected-plugins)).
2. **Generate an identity.** Against each isolated Gateway’s state, an operator runs `openclaw channels add --channel buzz`, selects its relay and lets guided setup generate/reuse the dedicated keypair; record the displayed **public** key. The flow waits for approval. The shared-Gateway option instead creates three **named** Buzz accounts ([guided setup](https://docs.openclaw.ai/channels/buzz#guided-setup)). Guided setup may save a **plaintext private key**: replace it with an `env`, `file`, or `exec` SecretRef before keeping the configuration, and keep the same identity across restarts ([key storage](https://docs.openclaw.ai/channels/buzz#bot-key-storage)).
3. **Obtain required Buzz grants.** If the hosted/closed relay requires community membership, its operator runs `buzz-admin add-member --pubkey <BOT_PUBLIC_KEY> --role member`. Separately, an existing human owner/admin of each room runs `buzz channels add-member --channel <ROOM_UUID> --pubkey <BOT_PUBLIC_KEY> --role bot`. Membership alone is **not** the room’s Bot role; never hand the human’s private key to OpenClaw ([Buzz room approval](https://docs.openclaw.ai/channels/buzz#bot-approval), [Buzz relay membership](https://github.com/block/buzz/blob/main/NOSTR.md#relay-membership-nip-43)).
4. **Select rooms and test.** Configure only the approved room UUID for each identity. Run `openclaw channels status --channel buzz --probe`, send a test to `buzz:<ROOM_UUID>`, and ask an allowed human to send an inbound message and see the reply ([verification](https://docs.openclaw.ai/channels/buzz#verify-the-connection)). If multiple bots share one room, first design mention/sender restrictions and test bot-loop behavior; the default guided setup accepts ordinary room-member messages ([bot conversations](https://docs.openclaw.ai/channels/buzz#bot-conversations)).

The Buzz private key, optional operator-issued owner-attestation `authTag`, model-provider credential, and Gateway access token are **different secrets**. Keep them out of Git, images, and ConfigMaps; project them from separately scoped Kubernetes Secrets or a secret manager. The Gateway does not need a Kubernetes API token just to talk to Buzz; disable ServiceAccount token automount unless a chosen tool needs it. Room Bot role is Buzz authorization, not Kubernetes RBAC.

## Launch three Kata-backed Pods: two real routes

**Common prerequisites:** On each eligible Linux node, verify KVM and `kata-runtime check`; install architecture-matched Kata, configure the k3s containerd Kata handler and a matching cluster-scoped `RuntimeClass`, then prove a **plain Pod** runs with that class. Use its *actual* name—Kata Deploy may create `kata-qemu-runtime-rs`, not `kata`. Heterogeneous clusters need constrained scheduling ([Kata/k3s steps and smoke resource](microsandbox-kubernetes.md#k3s-and-kata-a-real-pod-sandbox-not-an-agent-definition), [RuntimeClass rules](https://kubernetes.io/docs/concepts/containers/runtime-class/)). The image owner supplies the pinned OpenClaw/Buzz image and model configuration. Each Pod gets its own workspace, writable Gateway state/PVC, Buzz/model/Gateway secrets, resources and probes; the Gateway runs `openclaw gateway run` with `gateway.mode=local` and explicit `channels.buzz` configuration ([Gateway startup](https://docs.openclaw.ai/cli/gateway/running), [container probes](https://docs.openclaw.ai/install/docker#health-checks)).

| Route | What the repository would declare | Controller and trade-off |
|---|---|---|
| **Agent Sandbox v1.0.4** | Three `agents.x-k8s.io/v1beta1` `Sandbox` objects, one per role; each has `spec.podTemplate.spec.runtimeClassName: <installed-class>` plus its one Gateway container and state volumes. | [Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox/blob/v1.0.4/README.md) owns each singleton Pod/PVC lifecycle. It copies the Pod spec, but does **not** parse agent roles, provide a model, or create Buzz identities ([released API](https://github.com/kubernetes-sigs/agent-sandbox/blob/v1.0.4/api/v1beta1/sandbox_types.go#L219-L231)). |
| **Ordinary Deployments** | Three `apps/v1` Deployments, `replicas: 1`, each with `spec.template.spec.runtimeClassName: <installed-class>` and its own PVC and Secret projections. | Standard Kubernetes; no extra sandbox controller. Use a single-writer replacement strategy for Gateway SQLite/PVC state; own persistence and pause/resume operations yourself. |

Apply **one** chosen route only after the Buzz approvals and Kata smoke check. Inspect each backing Pod’s `spec.runtimeClassName`, readiness and Gateway logs, then perform the real Buzz round trip above. An outbound Buzz WebSocket needs no Kubernetes Service; expose the Gateway UI only with deliberate authentication. A running Pod alone does not prove model access or room membership.

One Gateway **can** host three agents and three Buzz identities in **one** Kata Pod via account bindings; that saves resources but shares a process, secret boundary, and failure domain ([OpenClaw multi-agent](https://docs.openclaw.ai/concepts/multi-agent)). It is **not three Pods**. [kagent 0.10.2](agent-control-plane-options.md#k3s-and-kata-neither-semantic-agent-cr-can-select-the-runtime) can interpret its *own* declarative agent format, but neither its `Agent` nor its [OpenClaw `AgentHarness`](https://github.com/kagent-dev/kagent/blob/v0.10.2/go/api/v1alpha2/agentharness_types.go) natively selects Kata; the latter lists Telegram/Slack rather than Buzz. A separately owned Kata admission/default policy **and** Buzz integration would be required, so it is not a drop-in third route.

**Decision before implementation:** confirm Block Buzz, choose **three isolated Gateways** if three Pods are mandatory, pick Sandbox versus Deployment, and supply the relay/room UUIDs, human approvers, model/provider/keys, verified RuntimeClass, storage class, image registry, and desired tool permissions. Nothing here has been deployed.
