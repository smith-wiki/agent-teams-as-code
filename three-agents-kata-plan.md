# Three OMP-backed Buzz agents in three Kata Pods

> **Status:** proposed architecture only. No Buzz identity, room grant, Secret, image, Pod, or cluster resource was created, and the Buzz → ACP → OMP path has not been integration-tested.

## Small proposed repository

```text
three-buzz-agents/
├── roles/
│   ├── triage.md
│   ├── analyst.md
│   └── reviewer.md
├── omp/
│   ├── triage.yml
│   ├── analyst.yml
│   └── reviewer.yml                 # non-secret model/tool settings
├── image/Containerfile              # pinned buzz-acp + OMP
└── deploy/
    ├── deployments/                 # one route
    └── sandboxes/                   # alternative route
```

The three role files contain the complete operating guidance:

**`roles/triage.md`**
```markdown
# Triage
Classify an incoming human request by subject, urgency, impact, and missing facts.
Return a queue recommendation, rationale, and questions for the requester.
Suggest a human-led analyst handoff when evidence is needed. Do not edit or approve work.
```

**`roles/analyst.md`**
```markdown
# Analyst
Investigate the supplied question using available evidence.
Distinguish observation from inference and cite the sources for material claims.
Return options and trade-offs. Do not make operational changes.
```

**`roles/reviewer.md`**
```markdown
# Reviewer
Independently check supplied work for correctness, risks, and missing cases.
Report findings by severity with supporting evidence and a clear disposition.
Do not edit, merge, deploy, or claim final approval.
```

These are three independent roles, not an automatic pipeline. Prompt text is guidance, not enforcement: model selection, enabled tools, filesystem/network access, Kubernetes security context, and approval policy must be configured separately.

For unattended turns, define an explicit OMP tool/approval policy: ACP client permission requests can be rejected when no interactive approval is available; `--auto-approve` is not a substitute for capability scoping ([OMP ACP approval behavior](https://github.com/can1357/oh-my-pi/blob/04f58a91d141bb0e7c5f7679c2235945ae813057/docs/approval-mode.md#acp-sessions)).

## The ACP seam keeps the harness replaceable

Each Pod has the same path:

```text
Buzz room → relay WebSocket → buzz-acp → ACP over stdio → omp acp
```

The stable **Interface** at the process **Seam** is ACP over stdio; OMP is the initial **Adapter**. `buzz-acp` owns Buzz identity, channel discovery, subscriptions, inbound-author policy, and reply tooling, keeping transport changes local to that Module. Buzz documents its relay-to-ACP architecture and the command/argument switch for [any ACP agent](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/README.md#L7-L15). Buzz Desktop also has an explicit [Oh My Pi preset](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/desktop/src-tauri/src/managed_agents/discovery/presets.rs#L139-L148), but that local PATH-based preset is evidence of compatibility, not a Kubernetes controller.

A representative environment for the **triage** Pod is:

```dotenv
BUZZ_ACP_AGENT_COMMAND=omp
BUZZ_ACP_AGENT_ARGS=acp,--config,/opt/omp/triage.yml
BUZZ_ACP_SYSTEM_PROMPT_FILE=/opt/roles/triage.md
BUZZ_RELAY_URL=wss://REPLACE_WITH_RELAY
BUZZ_PRIVATE_KEY=REPLACE_FROM_SECRET
BUZZ_ACP_SUBSCRIBE=mentions
BUZZ_ACP_RESPOND_TO=owner-only
BUZZ_ACP_AGENT_OWNER=REPLACE_WITH_HUMAN_HEX_PUBKEY
```

Repeat with the matching role file and identity. `BUZZ_API_TOKEN` is additionally required only where relay token auth is enforced. These names and the separate-keypair requirement are documented by [`buzz-acp`](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/README.md#L24-L49) and its [configuration source](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/src/config.rs#L253-L303). Prefer a verified `BUZZ_AUTH_TAG` owner attestation when available; otherwise explicitly supply the owner as above. The default `owner-only` policy drops **all** inbound events when no owner resolves ([owner resolution](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/src/lib.rs#L140-L170), [fail-closed gate](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/src/lib.rs#L2672-L2686)).

`BUZZ_ACP_SYSTEM_PROMPT_FILE` loads the role text in released `buzz-acp`; its delivery depends on the agent's ACP support: modern sessions may receive `systemPrompt` in `session/new`, while legacy sessions get standing instructions in the first user message ([Buzz ACP transport and fallback](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/src/pool.rs#L1495-L1545)). Verify the OMP pairing rather than assuming equal prompt priority. OMP can alternatively discover `.omp/AGENTS.md` or standalone `AGENTS.md` from the session working directory; avoid duplicate role injection ([OMP context discovery](https://github.com/can1357/oh-my-pi/blob/04f58a91d141bb0e7c5f7679c2235945ae813057/docs/context-files.md#L18-L72)). The server mode here is [`omp acp`](https://github.com/can1357/oh-my-pi/blob/04f58a91d141bb0e7c5f7679c2235945ae813057/docs/cli-reference.md#L166-L204), not `omp -p` or `omp --mode rpc`.

To replace OMP, keep the three role files, Buzz identities/grants, `buzz-acp`, and the three-Pod topology. Update the image, `BUZZ_ACP_AGENT_COMMAND`/`ARGS`, and harness-native model/provider credentials, configuration, tool permissions, and working-directory assumptions in the same workload resources. ACP does not standardize those concerns; OMP documents the [ACP config overlay](https://github.com/can1357/oh-my-pi/blob/04f58a91d141bb0e7c5f7679c2235945ae813057/docs/approval-mode.md#acp-sessions) and its own [provider resolution](https://github.com/can1357/oh-my-pi/blob/04f58a91d141bb0e7c5f7679c2235945ae813057/docs/providers.md#L9-L37).

## Make three identities addressable

For **triage, analyst, and reviewer**, run `buzz-admin generate-key` three times in a trusted operator environment; it prints each public/private Nostr keypair without retaining the private key ([Buzz ACP key instructions](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/README.md#L24-L43)). Store each private key outside Git and images. If the relay enforces community membership, its operator admits each public key with `buzz-admin add-member --pubkey <PUBLIC_KEY>`; this grant is conditional and distinct from channel membership ([Buzz relay membership](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/NOSTR.md#L240-L286)).

An authorized human owner/admin then adds that public key to its intended channel with `buzz channels add-member --channel <ROOM_UUID> --pubkey <PUBLIC_KEY> --role member` (using the **human** signing identity), or the agent creates its own channel. The [CLI accepts the `member` role](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-cli/src/lib.rs#L725-L735); `buzz-acp` discovers member channels, and private channels require explicit membership ([channel discovery](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/README.md#L45-L53)). A generic ACP identity does **not** require the OpenClaw plugin’s Bot-role approval.

With the default subscription, a permitted human must address the agent with an actual `@mention`/Nostr `p` tag inside that channel; the mention and inbound-author gate are separate checks ([subscription settings](https://github.com/block/buzz/blob/930b8bb800d8149ce29a881ba4c5d9f424580434/crates/buzz-acp/src/config.rs#L339-L356)). Prefer three separate channels, so each keypair, room, role, Secret, and Pod map one-to-one.

## Two k3s + Kata deployment routes

First prove KVM, architecture-matched Kata, the k3s containerd handler, and a plain Pod using the cluster's **actual** `RuntimeClass` name; Kata Deploy may create `kata-qemu-runtime-rs`, not `kata` ([prerequisites](microsandbox-kubernetes.md#k3s-and-kata-a-real-pod-sandbox-not-an-agent-definition), [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)).

| Route | Declare exactly three resources |
|---|---|
| **Ordinary Deployments** | Three `apps/v1` Deployments, `replicas: 1`, each Pod setting `runtimeClassName`, running one `buzz-acp` that spawns one OMP ACP subprocess, and receiving only its role/config and Buzz/model Secrets. This is the simpler default. |
| **Agent Sandbox v1.0.4** | Three `agents.x-k8s.io/v1beta1` `Sandbox` objects with the same container design under `spec.podTemplate.spec.runtimeClassName`. The controller owns singleton Pod/PVC lifecycle, but does not interpret roles, supply models, or create Buzz identities ([released API](https://github.com/kubernetes-sigs/agent-sandbox/blob/v1.0.4/api/v1beta1/sandbox_types.go#L210-L231)). |

Choose one route, not both. Outbound relay WebSocket traffic needs no Kubernetes Service. Before implementation, supply the relay/community policy, three channel IDs and authorized grant actor, three keypairs, owner attestations/pubkeys, model/provider and credentials, tool/security policy, pinned image registry, verified RuntimeClass, and storage choice. Then test each mention/reply round trip; a running Pod proves neither Buzz admission nor model access.
