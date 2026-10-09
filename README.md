# Lightning Foundry

An open-source, AI-native, local-first operating platform for autonomous Bitcoin Lightning
routing nodes. Foundry runs a minimal, verifiable Bitcoin Core and LND stack, keeps it
healthy, and decides, deterministically, whether each action the node's intelligence wants to
take is allowed. That intelligence is
[Lightning Jet](https://github.com/drneski/lightning-jet), or any module that keeps Foundry's
interface: **Jet optimizes the node; Foundry operates it.**

**Running a profitable routing node is a part-time job. Together, Jet and Foundry aim to make
it a decision you review, not a shift you work** — on modest hardware, with verifiable
software, intelligence that runs on your own machine, and an interface simple enough that
operating a node does not require becoming a Lightning expert first.

**Profitability is the optimization objective, not a guaranteed outcome.** Foundry measures
real economic performance and accounts for the full cost of running a routing node —
including the costs most dashboards leave out — and reports every outcome back to the module
that chose the action.

## Status

**Pre-alpha. Nothing here runs a node yet.**

This repository holds the design, written before the code so the boundaries are decided
rather than discovered: the vision, the event contract and its tests, the economic
definitions, the architecture and the invariants it keeps, the threat model, and the roadmap.
The runtime and the installer arrive in later milestones, and the intelligence comes from
Lightning Jet. Do not point
anything in this repository at a node holding funds you would mind losing, because there is
nothing here to point at yet.

## What Foundry is not

- **Not a Lightning implementation.** Bitcoin Core and LND provide the Bitcoin and
  Lightning infrastructure. Foundry operates that infrastructure; it does not reimplement it.
- **Not the intelligence.** Decisions about rebalancing, fees, channels and capital come from
  Lightning Jet, or another module that keeps Foundry's interface. Foundry decides whether
  each one is allowed.
- **Not a wallet.** Foundry never holds your seed and never needs it.
- **Not financial advice.** Routing is a business with real downside. A node can lose money
  through fees, capital lockup and force closes while behaving exactly as designed.
- **Not a service.** There is no cloud component, no account, and nothing phones home. The
  one thing that can leave the machine, the public event feed, is off until the operator
  turns it on.

## Documentation

Start with [`docs/vision.md`](docs/vision.md): who Foundry is for, and what it refuses to do.
Then, roughly in this order:

| Document | What it answers |
|---|---|
| [`docs/economics.md`](docs/economics.md) | What "profitable" means here, and how outcomes get attributed to decisions |
| [`docs/event-model.md`](docs/event-model.md) | What a node says about itself, what it never publishes, and how public events travel |
| [`docs/architecture.md`](docs/architecture.md) | The components, what each may hold, and why nothing reaches LND except through Policy |
| [`docs/invariants.md`](docs/invariants.md) | The eight principles every change is judged against |
| [`docs/threat-model.md`](docs/threat-model.md) | Who we defend against, and what actually stops them |
| [`docs/ai-strategy.md`](docs/ai-strategy.md) | The module's models and Foundry's assistant, and why neither enforces anything |
| [`docs/roadmap.md`](docs/roadmap.md) | The milestones in order, and what would make us stop |
| [`docs/integrations/oogabooga.md`](docs/integrations/oogabooga.md) | What Ooga Booga Land's Lightning Factory may show, and what publishing rebalances costs |
| [`docs/integrations/obl-payments-poc.md`](docs/integrations/obl-payments-poc.md) | How OBL's payments and Factory proof of concept lines up with the milestones |
| [`docs/integrations/lightning-jet.md`](docs/integrations/lightning-jet.md) | How Lightning Jet optimizes a node Foundry operates, and how Foundry approves what it does |
| [`docs/lightning-factory.md`](docs/lightning-factory.md) | What a consumer of the public stream must do with it, and must never show |
| [`docs/decisions/`](docs/decisions/) | Choices that are expensive to revisit, and why they were made |
| [`schemas/`](schemas/), [`examples/`](examples/) | The event contract as JSON Schema, and one channel's life in both streams |

Two of these are worth reading even if you skip the rest. `docs/economics.md` defines the
reward signal every automated decision is judged against, and `docs/event-model.md` explains
why there are two event schemas rather than one — publishing a node's liquidity state in real
time tells an adversary where to attack it.

## Tests

No dependencies and no build. Node 22 or newer:

```bash
node tests/contract.test.mjs
```

## Milestones

| | Milestone | Done when |
|---|---|---|
| **M1** | Contract and simulator | a consumer can build against the event stream with no node running |
| **M2** | Foundry Node, read-only | a real LND runs under Foundry and nothing has acted on it through Foundry |
| **M3** | Measurement and baselines | an operator can answer "was this channel worth having?" from their own data |
| **M4** | Supervised action | the module proposes, the operator approves, and no deterministic limit is ever breached |
| **M5** | Judging routing intelligence | Foundry can tell whether a module's model beats the M3 baselines on realized sats, and publishes the answer |
| **M6** | Autonomy within a mandate | a node runs unattended inside limits its module cannot widen |

Read-only comes before acting, and measurement comes before intelligence, deliberately.
M4 exists as its own milestone because the first time money moves under Foundry's authority,
even with the operator approving each action, is the riskiest step in the project.

Three tracks run alongside rather than gating the sequence: the Ooga Booga Land visualization
and community loop, Lightning Jet's own roadmap, and research into the smallest safe node.
Ordering, not dates — see [`docs/roadmap.md`](docs/roadmap.md).

## Contributing

Start with [`CONTRIBUTING.md`](CONTRIBUTING.md) for setup and what a reviewable change looks
like, and [`AGENTS.md`](AGENTS.md) for the rules code is held to: the dependency rules, the
security boundaries, the testing expectations, and the requirement that every commit is
written with AI assistance and says which model did the work.

**Contributions are open** under The Ooga Booga License — see
[`CONTRIBUTING.md`](CONTRIBUTING.md#licensing-your-contribution). There is no code yet, so the
most useful contributions today are to the design itself.

Found a vulnerability? Report it privately — see [`SECURITY.md`](SECURITY.md).

## License

[The Ooga Booga License](LICENSE), a public-domain dedication. The earlier Apache-2.0 and
Unlicense contribution policy is recorded in
[decision 0006](docs/decisions/0006-open-contributions.md), and the switch in
[decision 0007](docs/decisions/0007-ooga-booga-license.md).
