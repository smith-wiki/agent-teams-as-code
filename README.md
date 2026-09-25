# Agent teams as code

How to describe a team of agents—roles, tools, enforced access, communication identities—and operate onboarding plus event-driven, suspendable execution.

## Questions and answers

1. **How can I define agents declaratively and handle onboarding and sleeping?** Start with a small versioned [agent definition](agent-manifest.md) as desired state. Compare [existing descriptor formats](descriptor-landscape.md) before adopting one; reconcile communication accounts and grants separately ([identity reconciliation](identity-reconciliation.md)); let a coordinator own events and sessions and use an execution backend only when needed ([sleeping agents](sleeping-agents.md)). **Open:** which service “Buzz” means in this environment, whether agents need full mailboxes, and which provider-specific grants the first real workflow requires.

Start with [the proposed agent manifest](agent-manifest.md). These notes describe a design direction, not a deployed controller.
