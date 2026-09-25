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

APM is not required for this group model. With one harness and a Nix-based build, an analogous source directory such as `agents/<stable-id>/` can hold each role's native configuration; the release pipeline can derive a deployment index of those IDs and **published** image digests after building and pushing the images. This is a **proposed repository convention**, not a built-in Nix agent format or a second list to maintain ([language choice](agent-language-options.md), [Nix image builds](https://nixos.org/manual/nixpkgs/stable/#ex-ociTools-buildContainer-bash)).

If another system needs an `AgentGroup` object, derive it from the repository revision and its agent files rather than authoring an additional source of truth. Introduce an explicit membership manifest only if the desired members differ from that file set. Assign each agent a committed immutable ID, independent of its display name or path (for example, use an ID-based native directory or adjacent metadata for an APM file); renaming either leaves the ID, external accounts and sessions intact. Provider accounts, enforced grants and wake/sleep remain **per-agent operational concerns**, not shared group attributes; APM does not provision them ([APM boundary](microsoft-apm.md), [identity reconciliation](identity-reconciliation.md), [sleeping execution](sleeping-agents.md)). Do not mistake `tools` in agent frontmatter for a provider ACL.
