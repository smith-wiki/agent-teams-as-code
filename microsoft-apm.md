# Microsoft APM: the recalled tool, but a packaging layer

**Fit: yes, this is almost certainly the Microsoft “apm” you recalled; no, it is not an agent-team control plane.** Microsoft’s [`microsoft/apm`](https://github.com/microsoft/apm) is Agent Package Manager: it resolves, installs and projects reusable agent configuration into multiple harnesses.

## Input and output

APM is a **local package manager and adapter for agent-harness configuration**, not an agent runtime. Its input is a package directory: `apm.yml` identifies the package and optionally declares targets, package/MCP dependencies and scripts; `.apm/` contains authored primitives such as `.apm/agents/reviewer.agent.md` (a named role with instructions and optional model/tool hints). `apm install` also reads referenced packages and, when replaying a pinned install, `apm.lock.yaml`. `--target copilot,claude` chooses **output formats**, not machines on which agents will run ([package anatomy](https://microsoft.github.io/apm/concepts/package-anatomy/), [install reference](https://microsoft.github.io/apm/reference/cli/install/)).

For example, `apm install --target copilot,claude` in that directory produces harness-readable role files such as `.github/agents/reviewer.agent.md` and `.claude/agents/reviewer.md`, plus a resolved `apm.lock.yaml` and `apm_modules/` when dependencies are installed. Declared MCP dependencies produce harness configuration, not provider accounts or running servers. Separately, `apm compile` can generate root instruction context such as `AGENTS.md` where needed ([agent target mappings](https://microsoft.github.io/apm/producer/author-primitives/instructions-and-agents/#what-compiles-where-1), [install reference](https://microsoft.github.io/apm/reference/cli/install/)). **The output is files and configuration, not an agent process, an answer to a task, or an event-handling service.** The harness consumes these files later; a caller or operator supplies the actual task, execution, credentials and lifecycle ([installation boundary](apm-install-deployment.md)).

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

`dependencies.apm` accepts repositories, packages, subdirectories or individual `.agent.md` files; `dependencies.mcp` accepts registry references or explicit `transport` plus `command`/`url`, with optional `args`, `env`, headers and tool filters ([manifest schema §4](https://microsoft.github.io/apm/reference/manifest-schema/#4-dependencies)). `apm install` resolves transitive dependencies and emits `apm.lock.yaml`, pinning commits, hashes, deployed files and per-target MCP ownership; the project should commit it ([lockfile specification](https://microsoft.github.io/apm/reference/lockfile-spec/)). `targets` selects output adapters, not execution destinations; primitive support varies by target.

A role is a Markdown body in `.apm/agents/reviewer.agent.md`; frontmatter carries `name`, required `description`, and optional `model`, `tools` whitelist and `handoffs`. For example, `tools: {Read: true, Grep: true}`. APM translates or copies that file only for harnesses with agent-primitive support; tool semantics can degrade by target, notably Codex ([agent authoring contract](https://microsoft.github.io/apm/producer/author-primitives/instructions-and-agents/#agents)). `handoffs` can reference other agents, but this is harness metadata, not a reconciled team topology.

## Supported targets

The canonical [targets matrix](https://microsoft.github.io/apm/reference/targets-matrix/) currently lists these stable target slugs: `copilot`, `claude`, `grok-build`, `cursor`, `codex`, `gemini`, `antigravity`, `opencode`, `windsurf`, `kiro`, `intellij`, `agent-skills`, and `hermes`. This means APM has an output adapter; it does **not** mean that the harness is installed, licensed, authenticated or running.

For named roles authored as `.apm/agents/<name>.agent.md`, agent-primitive support is narrower: `copilot` writes `.github/agents/<name>.agent.md`; `claude` writes `.claude/agents/<name>.md`; `grok-build` writes `.grok/agents/<name>.md`; `cursor` writes `.cursor/agents/<name>.md`; `codex` writes `.codex/agents/<name>.toml`; `opencode` writes `.opencode/agents/<name>.md`; and `kiro` writes `.kiro/agents/<relative-stem>.md`. Kiro retains only `description`, `model` and `tools` frontmatter and rejects unsupported tool values. `intellij` is a special case: its own adapter configures MCP only, while explicitly selected file primitives route through the Copilot profile (agents therefore land under `.github/agents/`).

The stable `gemini`, `antigravity`, `windsurf`, `agent-skills`, and `hermes` targets do **not** deploy named agent files. They support differing subsets of skills, instructions/compiled context, commands, hooks and MCP; notably, Windsurf personas should be packaged as skills, and `agent-skills` is a skills-only meta-target rather than a harness. Experimental target slugs are separate from the stable list: `copilot-cowork`, `copilot-app`, `grok-cloud`, and `openclaw` require their experimental flags, are explicit `--target` selections, and cannot be declared in `apm.yml` ([experimental targets](https://microsoft.github.io/apm/reference/targets-matrix/#detection-and-resolution)).

Run [`apm targets`](https://microsoft.github.io/apm/reference/cli/targets/) in the project to see the canonical table, which targets are active, the detection signal and deploy directory; `apm targets --json` is the machine-readable form, and `apm targets --json --all` also includes the otherwise omitted `agent-skills` meta-target. The command itself has no target-selection flag: installation and compilation resolve targets in the order explicit `--target`/`--all`, `targets:` in `apm.yml`, then filesystem auto-detection. `agent-skills`, `antigravity`, and `hermes` are not auto-detected or included in target-selection `all`; `intellij` is also excluded from plain `all`.

`apm-policy.yml` can block package/MCP sources and transports at **install time**; some rules, including the manifest-scripts policy, run only under `apm audit --ci`. It does not grant runtime permissions, constrain OAuth/token scopes, or sandbox agent actions; `apm compile` and `apm run` do not re-enforce policy ([governance boundary and check matrix](https://microsoft.github.io/apm/enterprise/governance-guide/#3-what-you-can-govern), [policy fields](https://microsoft.github.io/apm/enterprise/policy-reference/)). `apm run` can execute a manifest shell script and compile referenced prompts, but it is launch convenience rather than lifecycle reconciliation ([CLI reference](https://microsoft.github.io/apm/reference/cli/run/)).

For the **group of agents in one repository**, the [APM package layout](agent-manifest.md) is sufficient: `.apm/agents/` supplies membership and `apm.yml` supplies package metadata and dependencies. No separate `AgentTeam` manifest is needed. APM still does not provision per-agent service accounts or provider grants, or manage durable sessions and wake/sleep; those are concerns of individual agent operators and execution coordinators, not package-level team properties.

For the operational split—**who runs the installer, which workspace receives files, and what remains outside APM**—see [installation and deployment](apm-install-deployment.md) and [onboarding and runtime](apm-onboarding-runtime.md).
