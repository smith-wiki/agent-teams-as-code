# A repository is the agent group; each agent owns its definition

**Correction:** there is no separate `AgentTeam` resource or team-level configuration in the current requirement. A **group** is the set of independently defined agents in one repository at a given revision. It has no collective role, goal, permission set, topology, or runtime state. Those would be new requirements, not fields to invent now.

[Microsoft APM's package layout](https://microsoft.github.io/apm/reference/package-types/#apm-package-apm-directory) already represents such a collection:

```text
review-agents/
  apm.yml
  .apm/
    agents/
      reviewer.agent.md
      triager.agent.md
```

```yaml
# apm.yml: package metadata and optional dependencies, not a team object
name: review-agents
version: 1.0.0
targets: [copilot, claude]
```

The `.apm/agents/` files are the members; `apm.yml` names and versions the **package** but does not enumerate local agents or define team semantics. APM deploys each agent primitive independently to supported harnesses ([package layout](https://microsoft.github.io/apm/reference/package-types/#apm-package-apm-directory), [target matrix](https://microsoft.github.io/apm/reference/targets-matrix/)). Put role instructions and supported tool declarations in each `.agent.md` ([agent authoring](https://microsoft.github.io/apm/producer/author-primitives/instructions-and-agents/#agents)). Adding or removing a file changes membership; a consumer pins the repository revision or package version. No second hand-maintained `agents:` list is needed when membership equals the files in the repo.

APM is not required for this group model. If choosing a Kubernetes controller, the repository's `Agent` or `Sandbox` resource files are its deployment members; do not maintain a second hand-written roster. Build each BYO image with Nix and keep its harness-specific role configuration inside that image. `.github/agents/*.agent.md` may serve Copilot/VS Code clients but is not an input the reviewed controllers require. With mandatory Microsandbox, no shipped Kubernetes guest controller currently supplies this membership model, so a repository convention and an explicit lifecycle owner would have to be designed ([controller choices](agent-control-plane-options.md#kubernetes-operator-first-decision), [Microsandbox boundary](microsandbox-kubernetes.md), [language choice](agent-language-options.md)).

If another system needs an `AgentGroup` object, derive it from the repository revision and the selected deployer's resource files rather than authoring an additional source of truth. An explicit manifest is warranted only if intended membership differs from that file set. An opaque stable ID may be needed later for external identity matching, but renaming `metadata.name` creates a different Kubernetes object; an annotation cannot by itself preserve a workload or its sessions. Provider accounts and wake/sleep are separate per-agent integration questions, not selection criteria for the initial deployer ([identity reconciliation](identity-reconciliation.md), [sleeping execution](sleeping-agents.md)).
