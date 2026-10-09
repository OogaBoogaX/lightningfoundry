# Roadmap

Ordering, not dates. This is volunteer work on software that moves money; a schedule would
be a guess dressed as a commitment, and the temptation to hit a date is exactly how autonomous
financial software ships early.

Two rules shape the list:

1. **Every milestone ends with something someone can use.** No milestone exists only to enable
   the next one.
2. **Every acceptance criterion is checkable inside this repository.** A milestone that can
   only be completed by another team's merge is a dependency, not a milestone.

## The spine

### M1 — Contract and simulator

Establish what a Foundry node says about itself, before building anything that says it.

- Event schemas, internal and public ([`event-model.md`](event-model.md))
- Economic definitions ([`economics.md`](economics.md))
- Invariants and threat model
- A deterministic simulator producing reproducible event streams for the scenarios that
  matter: channel lifecycle, healthy routing, liquidity imbalance, rebalancing, and an
  unprofitable channel
- Signed, push-only delivery of the public stream, exercised end to end by the simulator
  against a reference receiver
- The consumer's side of the contract, written for Ooga Booga Land's Lightning Factory
  ([`lightning-factory.md`](lightning-factory.md))
- Contract tests

**Done when:** a consumer can build against the event stream without a Lightning node
existing, and the same scenario and seed produce the same stream every run.

*A visualization built on this contract lives in Ooga Booga Land and ships on its own
schedule. It is not a gate on M1.*

### M2 — Foundry Node, read-only

A minimal, verifiable, installable node that **observes and touches nothing**.

- Bitcoin Core and LND, with versions and hashes published and verified at install
- Or adoption onto a compatible node that already runs, its Bitcoin Core and LND versions
  checked against a published list and recorded
  ([decision 0010](decisions/0010-adopt-or-install.md))
- Foundry Core emitting real events against the M1 contract, and publishing the public ones
  when the operator opts in
- Process supervision, health monitoring and backups for Bitcoin Core and LND
- Upgrades that say, before the operator approves them, whether they migrate a database and
  so cannot be rolled back
- Lightning Jet running standalone under the operator's own policy, with Foundry observing
  what it does ([decision 0011](decisions/0011-jet-optimizes-foundry-operates.md))
- Deterministic dependency and policy validation
- A small local interface
- A local assistant that can explain what the node is doing and change nothing

**Done when:** a contributor installs Foundry on supported hardware, runs a real LND, sees
real events, and nothing has acted on the node through Foundry.

Read-only first is deliberate. It puts the whole stack — install, verification, supervision,
event emission, visualization — under test while the blast radius is zero.

### M3 — Measurement and baselines

Make [`economics.md`](economics.md) real, and build the thing every later claim is measured
against.

- Per-channel cost accounting: revenue, rebalance cost, lifecycle cost, capital committed
- The three measures, reported and reconciled against the node's actual balance
- Holdout and staggered-rollout infrastructure, so attribution is possible later
- **Deterministic baseline strategies** — simple fee rules, threshold rebalancing, Jet's
  existing behavior — run by the operator and Jet as they are today, with Foundry recording
  their performance over real operation

**Done when:** an operator can answer "was this channel worth having?" with numbers derived
from their own node, and the baselines have published results on real data.

This milestone is missing from most projects like this, and it is the one that makes the rest
honest. Without baselines there is nothing for a model to beat, and "the AI improved things"
becomes unfalsifiable.

### M4 — Supervised action

The first time money moves under Foundry's authority. It gets its own milestone because it is
the single riskiest transition in the project, and burying it inside a larger one is how it
goes wrong.

- A managed module, Lightning Jet or another, asks for each action as an intent; the operator
  approves it; the module executes, and Policy checks each call against the approved intent
- Scoped LND macaroons: the component that reads is not the component that acts, and the one
  that acts cannot act without Policy
- Deterministic limits enforced in code, not policy: daily rebalance fee ceiling, channel
  close ceiling, reserve floor, how often a channel's fee may change
- A kill switch that returns the node to operator control immediately: unregistering Policy
  stops the module at its next call
- Every intent, verdict and outcome recorded against M3's accounting, with the module and
  model version that asked

**Done when:** a node runs under supervision for a sustained period with zero limit
violations, and every action taken can be traced to the intent that asked for it and the
approval that allowed it.

### M5 — Judging routing intelligence

Jet builds learned models for peer classification, demand forecasting, channel recommendation
and capital allocation. Foundry decides whether they earn authority: a new version runs
advisory-only, recorded and judged but not approved, against M3's baselines.

**Done when:** Foundry can tell whether a module's model beats the deterministic baseline on
**realized sats net of full costs**, over a pre-registered evaluation window, with holdouts,
and publishes the answer either way.

A negative result here is a real contribution. A routing node sees only its own forwards, in
a non-stationary environment, with delayed and confounded rewards and no observable
counterfactuals. Those conditions favor simple heuristics, and "we tried, the heuristic won,
here is the data" is more useful to the ecosystem than a model that quietly underperforms one.

### M6 — Autonomy within a mandate

The full loop, decided and carried out by the module: peer discovery, channel allocation, fee
policy, rebalancing, channel retirement, capital redeployment — inside a deterministic
mandate Foundry enforces and the module cannot widen.

**Done when:** a node operates unattended within its mandate for a sustained period, its
economic outcome is attributable rather than merely recorded, and the operator can explain
every decision it made.

## Parallel tracks

These do not gate the spine and do not wait for it.

### Ooga Booga Land and the community loop

OBL is Foundry's reference deployment and its front door. The OBL node becomes a real
Foundry-operated node with real economic activity, which gives Foundry a live system to
operate rather than a synthetic demo. The Lightning Factory cave turns the public event
stream into something a person can watch and understand. OBL's proof of concept, mapped onto
these milestones, is in [`integrations/obl-payments-poc.md`](integrations/obl-payments-poc.md).

The loop that matters: someone meets Lightning through a game, watches a gorilla build a
channel, learns why a rebalance happened, finds Foundry or Jet, contributes, runs a node — and
may eventually connect that node back to the ecosystem.

Later, additional operators' nodes can appear in the Factory as separate rooms, each showing
only what its operator chose to publish.

**The boundary is permanent:** the cave consumes exported events. It never holds credentials,
never controls LND, and receives nothing the public schema cannot express. Foundry stays
useful with no cave at all, and OBL stays a separate project with its own schedule.

### Lightning Jet

Jet is an independent project with its own roadmap: 1.6.1 is an incremental release, and Jet
2.0 is the rework built to be managed, and to be the intelligence M5 and M6 judge. Foundry's
milestones never wait on it. Each is checked against a scripted stand-in module that keeps the
interface, as rule 2 requires. See
[`integrations/lightning-jet.md`](integrations/lightning-jet.md).

### The smallest safe node — research

What is the minimum software and hardware that can run a secure, reliable, autonomous routing
node? Open questions anyone can pick up:

- the smallest Linux system that can safely run Bitcoin Core, LND and a module;
- surviving power loss on single-board computers without corrupting state;
- storage wear on SD cards and SSDs under a node's write load;
- what to strip from the operating system to shrink the attack surface;
- what local models need from the hardware.

Benchmark affordable commodity hardware across the real workload: Bitcoin Core, LND, Foundry
and the module's inference together.

**The gate:** build a dedicated appliance only if the measurements show commodity hardware is
genuinely inadequate. The default outcome is published benchmarks and reference
configurations, which is a useful result and much cheaper than hardware.

This sits last for a reason. Designing hardware for a workload that does not exist yet
produces hardware for an imagined workload.

## What would make us stop, or change course

Stated now, while it is cheap to be honest:

- **The baselines win.** If no module's model can beat M3's deterministic strategies on real
  economics, Foundry approves only the deterministic strategies, says so publicly, and the
  models stay research rather than product.
- **The economics don't work.** If complete profitability is reliably negative once capital is
  accounted for, that is a finding about Lightning routing, not a failure of the software, and
  it should be published as clearly as a success would be.
- **The security model can't hold.** If deterministic limits cannot actually constrain the
  autonomous loop, M6 does not ship. Autonomy is not worth a weakened mandate.
