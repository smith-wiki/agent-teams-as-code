# APM `install` and `compile` materialize files; they do not run agents

A repository maintainer, contributor, or CI job invokes APM from the active checkout; APM is a local CLI, not a service that deploys on its own. For an illustrative repository containing `.apm/agents/reviewer.agent.md` and `.apm/agents/triager.agent.md` (two independent primitives, not an APM “team” feature), the minimal project-scope flow is:

```bash
apm install --target copilot,claude
```

`install` resolves `apm.yml`, downloads remote and transitive dependencies, scans them, then deploys dependencies **and the repository’s own `.apm/` files**; local files are applied last and win collisions. Thus both agents land in each supported harness directory—for these targets, `.github/agents/*.agent.md` and `.claude/agents/*.md`. Codex is another supported agent target (`.codex/agents/*.toml`); Gemini does not support the agent primitive ([install CLI](https://microsoft.github.io/apm/reference/cli/install/), [target matrix](https://microsoft.github.io/apm/reference/targets-matrix/)).

A producer authors the package; a consumer can reference it remotely as `owner/repo#ref`, by HTTPS/SSH, or locally as `./packages/shared`. A positional `apm install PACKAGE_REF` adds that reference to `apm.yml`; bare `apm install` consumes the manifest. Resolved packages are materialized under `apm_modules/`. By default, `apm.lock.yaml` sits beside `apm.yml` and pins the graph, hashes and deployed-file ownership; commit it, then use `apm install --frozen` for lockfile-only replay ([dependency flow](https://microsoft.github.io/apm/consumer/manage-dependencies/), [lockfile specification](https://microsoft.github.io/apm/reference/lockfile-spec/)).

By default, writes go under `$PWD`. `apm install --root DIR` redirects `apm_modules/`, the lockfile, `.gitignore` and harness files, while sources and local-path dependencies still resolve from the current directory. `-g/--global` instead uses user scope under `~/.apm/` plus target-specific home directories, but skips deployment of the project’s own `.apm/` files; to publish these agents to global scope, consume their package as a dependency. Global root-context generation is a separate `apm compile -g`, which cannot be combined with `--target` or `--root` ([install flags and local-file behavior](https://microsoft.github.io/apm/reference/cli/install/#behavior), [compile flags](https://microsoft.github.io/apm/reference/cli/compile/#global-compilation)).

`apm compile` reads instruction primitives from `.apm/` and `apm_modules/` and writes root context files/rules; `--local-only` excludes dependency instructions. It neither fetches dependencies nor deploys agents. Compile only when the chosen harness needs generated instruction context; it is optional for Copilot and recommended for other context-producing targets ([compile guide](https://microsoft.github.io/apm/reference/cli/compile/#description)).

## Takeaway

Here “deploy” means deterministic file/configuration projection into a worktree or user home, not deployment of a remote service. For interactive use, the developer installs into their checkout. For unattended use, a deployment job must make the generated workspace or artifact available to the chosen harness; the [APM GitHub Action](https://github.com/microsoft/apm-action) can install, audit or pack in CI, but an ephemeral CI checkout is not a persistent worker or a remote publishing mechanism by itself. `install` and `compile` do not launch model processes; `apm run` can invoke a named shell command but is not an agent supervisor ([install CLI](https://microsoft.github.io/apm/reference/cli/install/), [run CLI](https://microsoft.github.io/apm/reference/cli/run/)). Provider-account onboarding and live sessions remain separate.
