# Microsoft APM: the recalled tool, but a packaging layer

**Fit: yes, this is almost certainly the Microsoft “apm” you recalled; no, it is not an agent-team control plane.** Microsoft’s [`microsoft/apm`](https://github.com/microsoft/apm) is Agent Package Manager: it resolves, installs and projects reusable agent configuration into multiple harnesses.

A minimal current working-draft manifest can package an agent primitive and an MCP dependency:

```yaml
name: review-team-context
version: 1.0.0
targets: [copilot, claude]
dependencies:
  apm:
    - acme/agent-library/agents/reviewer.agent.md#v1.0.0
  mcp:
    - name: io.github.github/github-mcp-server
      tools: [repos, issues]
```

`dependencies.apm` accepts repositories, packages, subdirectories or individual `.agent.md` files; `dependencies.mcp` accepts registry references or explicit `transport` plus `command`/`url`, with optional `args`, `env`, headers and tool filters ([manifest schema §4](https://microsoft.github.io/apm/reference/manifest-schema/#4-dependencies)). `apm install` resolves transitive dependencies and emits `apm.lock.yaml`, pinning commits, hashes, deployed files and per-target MCP ownership; the project should commit it ([lockfile specification](https://microsoft.github.io/apm/reference/lockfile-spec/)). `targets` selects output adapters, not execution destinations. Support is uneven: Copilot and Claude accept agents; Gemini does not ([targets matrix](https://microsoft.github.io/apm/reference/targets-matrix/)).

A role is a Markdown body in `.apm/agents/reviewer.agent.md`; frontmatter carries `name`, required `description`, and optional `model`, `tools` whitelist and `handoffs`. For example, `tools: {Read: true, Grep: true}`. APM translates or copies that file into each supported harness; tool semantics can degrade by target, notably Codex, while Gemini and Windsurf receive no agent primitive ([agent authoring contract](https://microsoft.github.io/apm/producer/author-primitives/instructions-and-agents/#agents)). `handoffs` can reference other agents, but this is harness metadata, not a reconciled team topology.

`apm-policy.yml` can block package/MCP sources and transports at **install time**; some rules, including the manifest-scripts policy, run only under `apm audit --ci`. It does not grant runtime permissions, constrain OAuth/token scopes, or sandbox agent actions; `apm compile` and `apm run` do not re-enforce policy ([governance boundary and check matrix](https://microsoft.github.io/apm/enterprise/governance-guide/#3-what-you-can-govern), [policy fields](https://microsoft.github.io/apm/enterprise/policy-reference/)). `apm run` can execute a manifest shell script and compile referenced prompts, but it is launch convenience rather than lifecycle reconciliation ([CLI reference](https://microsoft.github.io/apm/reference/cli/run/)).

Therefore APM has no schema/controller for per-agent service accounts, provider authorization, onboarding/offboarding, durable sessions, wake/sleep, health or event-driven reconciliation. Use it conditionally beneath our proposed [AgentTeam manifest](agent-manifest.md): APM distributes pinned role/skill/MCP artifacts; the higher control plane owns identity, least-privilege grants, topology and runtime state.
