# Declare agents as desired state, then bind each field to an owner

**Proposal, not an existing standard or deployable AX manifest:** start with a small versioned `AgentTeam` YAML, not an all-purpose agent DSL. A role is guidance; a tool is a capability; an account is an external identity; *access* must be enforced by the provider or sandbox. A prompt saying “read-only” does not revoke a GitHub write token. AX's current public resource kinds are `Task`, `Workspace`, and `Model`, not `AgentTeam` ([AX API](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/pkg/apis/v1alpha1/ax.proto), [AX concepts](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md)).

```yaml
apiVersion: company.example/v1alpha1
kind: AgentTeam
metadata: {name: code-review}
agents:
  - id: reviewer                        # stable key across renames and restarts
    role: "Review pull requests and explain findings"
    instructionsRef: "git:prompts/reviewer.md@<commit>"
    tools: [github-pr, zulip]
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

The example intentionally names **logical** tools and access; production adapters must translate them into provider-specific grants, credentials, network policy and audit rules, or reject the manifest. No real provider promises these exact operations. Keep credential *references* in a separate secret store, never credential values in Git; map provider-assigned IDs back to stable agent IDs. If mail is needed, decide whether the agent needs a mailbox, sending identity, alias, or nothing—those are different capabilities ([identity reconciliation](identity-reconciliation.md)).

A controller should validate a definition, calculate a diff, provision identities, bind least-privilege access, publish a ready condition, then permit event-driven runs. Changes to desired state must also reduce old grants; destructive account deletion should require an explicit retention/offboarding policy. Keep observed state (remote IDs, failures, versions) outside the desired-state file. The coordinator owns runs, events, sessions and outcomes; it can translate one agent into an AX `Task` if that backend is chosen ([sleeping agents](sleeping-agents.md)). Compare existing formats before implementing the schema: [descriptor landscape](descriptor-landscape.md). This sketch deliberately leaves team routing and delegation rules undecided until a concrete workflow requires them.
