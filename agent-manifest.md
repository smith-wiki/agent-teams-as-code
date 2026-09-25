# Declare agents as desired state, then bind each field to an owner

**Proposal, not an existing standard or deployable AX manifest:** use [Microsoft APM](microsoft-apm.md) for versioned agent profiles, instructions, skills and MCP dependencies. Add only a small versioned `AgentTeam` YAML for what APM does not represent: stable operated identities, external accounts/grants, and event-driven execution. A role is guidance; a tool is a capability; an account is an external identity; *access* must be enforced by the provider or sandbox. A prompt saying “read-only” does not revoke a GitHub write token. AX's current public resource kinds are `Task`, `Workspace`, and `Model`, not `AgentTeam` ([AX API](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/pkg/apis/v1alpha1/ax.proto), [AX concepts](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md)).

```yaml
apiVersion: company.example/v1alpha1
kind: AgentTeam
metadata: {name: code-review}
agents:
  - id: reviewer                        # stable key across renames and restarts
    profileRef: "company/agents/roles/reviewer.agent.md#v1.0.0"  # APM file dependency in this proposed schema
    access:
      github-pr: {repositories: [org/app], operations: [read, comment]}
      zulip: {streams: [reviews], operations: [read, post]}
    accounts:
      zulip: {type: bot, owner: platform-team}
    runtime:
      image: "registry.example/reviewer@sha256:<digest>"
      activation: on-event
      sessionStore: reviewer-sessions
```

The example intentionally names **logical services and desired access**; production adapters must translate them into provider-specific grants, credentials, network policy and audit rules, or reject the manifest. No real provider promises these exact operations. `profileRef` points to an [APM package](https://microsoft.github.io/apm/reference/package-types/) with the agent's role instructions, tool allowlist and tool dependencies; the operator must pin and install it for the chosen harness. APM's [target matrix](https://microsoft.github.io/apm/reference/targets-matrix/) is not uniform, and its [installation policy](https://microsoft.github.io/apm/enterprise/policy-reference/) does not enforce per-agent runtime grants. Keep credential *references* in a separate secret store, never credential values in Git; map provider-assigned IDs back to stable agent IDs. If mail is needed, decide whether the agent needs a mailbox, sending identity, alias, or nothing ([identity reconciliation](identity-reconciliation.md)).

A controller should validate a definition, calculate a diff, provision identities, bind least-privilege access, publish a ready condition, then permit event-driven runs. Changes to desired state must also reduce old grants; destructive account deletion should require an explicit retention/offboarding policy. Keep observed state (remote IDs, failures, versions) outside the desired-state file. The coordinator owns runs, events, sessions and outcomes; it can translate one agent into an AX `Task` if that backend is chosen ([sleeping agents](sleeping-agents.md)). Compare existing formats before implementing the schema: [descriptor landscape](descriptor-landscape.md). This sketch deliberately leaves team routing and delegation rules undecided until a concrete workflow requires them.
