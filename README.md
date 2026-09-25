# Agent groups as code

How to describe a repository-scoped group of independent agents—roles, tools, enforced access, communication identities—and operate onboarding plus event-driven, suspendable execution.

## Questions and answers

1. **How can I define agents declaratively and handle onboarding and sleeping?** **Revised:** use [APM](microsoft-apm.md) for agent definitions and dependencies. The [repository's agent files form the group](agent-manifest.md); there is no separate team object. Reconcile each agent's communication accounts and grants ([identity reconciliation](identity-reconciliation.md)); a coordinator owns events and sessions ([sleeping agents](sleeping-agents.md)). **Open:** which service “Buzz” means here, whether agents need full mailboxes, and which provider-specific grants the first workflow requires.

2. **Was the missing Microsoft tool called APM, and does it solve this?** Yes: [Microsoft Agent Package Manager](microsoft-apm.md) installs/version-pins agent files, instructions, skills and MCP servers across supported harnesses. It can launch shell scripts, but does not provision communication accounts or manage sleeping sessions. [Microsoft Agent Framework and Microsoft 365 declarative agents](microsoft-declarative-agents.md) are separate runtime/product-specific formats. This [corrects the earlier comparison](descriptor-landscape.md).

3. **Do we need a distinct team object, or a group of agents in one repository?** Just the [repository-scoped group](agent-manifest.md). An APM package can contain multiple `.apm/agents/*.agent.md` files; `apm.yml` describes the package, not shared team behavior. Do not author an `AgentTeam` or duplicate the file list unless explicit membership beyond the repository becomes a requirement. Accounts, grants and runtime state belong to individual agents and their operators.

4. **Who deploys agents from the repository, where do they run, and does APM handle onboarding?** A developer or deployment job invokes [APM installation](apm-install-deployment.md) in a target workspace: it projects agent files and MCP settings to the selected harness, not to a remote running service. APM can [scaffold the project, install some runtime CLIs, run a shell script and offer trusted install hooks](apm-onboarding-runtime.md), but it neither creates provider accounts/grants nor hosts event-driven sessions. For interactive agents, provider onboarding can be manual; unattended agents need provider integration and a worker/coordinator. **Open:** which harness and execution host will actually run these agents.

Start with [the repository-scoped agent group](agent-manifest.md), then [the APM installation path](apm-install-deployment.md) and [onboarding boundary](apm-onboarding-runtime.md). These notes describe a design direction, not a deployed controller.
