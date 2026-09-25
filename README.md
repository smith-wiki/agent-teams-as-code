# Agent teams as code

How to describe a team of agents—roles, tools, enforced access, communication identities—and operate onboarding plus event-driven, suspendable execution.

## Questions and answers

1. **How can I define agents declaratively and handle onboarding and sleeping?** **Revised:** use [APM](microsoft-apm.md) for reusable agent profiles and dependencies, and a small [desired-state manifest](agent-manifest.md) only for identities, enforced grants and lifecycle. Reconcile communication accounts separately ([identity reconciliation](identity-reconciliation.md)); a coordinator owns events and sessions ([sleeping agents](sleeping-agents.md)). **Open:** which service “Buzz” means here, whether agents need full mailboxes, and which provider-specific grants the first workflow requires.

2. **Was the missing Microsoft tool called APM, and does it solve this?** Yes: [Microsoft Agent Package Manager](microsoft-apm.md) installs/version-pins agent files, instructions, skills and MCP servers across supported harnesses. It can launch shell scripts, but does not provision communication accounts or manage sleeping sessions. [Microsoft Agent Framework and Microsoft 365 declarative agents](microsoft-declarative-agents.md) are separate runtime/product-specific formats. This [corrects the earlier comparison](descriptor-landscape.md).

Start with [Microsoft APM](microsoft-apm.md), then [the proposed operational manifest](agent-manifest.md). These notes describe a design direction, not a deployed controller.
