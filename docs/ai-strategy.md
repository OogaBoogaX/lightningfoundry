# AI strategy

There are **two** AI systems on a Foundry node, and only one of them is Foundry's. The routing
intelligence, the models that decide rebalances, fees, channels and capital, lives in Lightning
Jet, or another module that keeps Foundry's interface
([decision 0011](decisions/0011-jet-optimizes-foundry-operates.md)). The local assistant is
Foundry's own. Foundry's job toward the module's models is to run them safely and judge them
honestly. Conflating the two systems is how the hardware story becomes incoherent and the
safety story becomes vague.

| | Routing intelligence (the module's) | Local assistant (Foundry's) |
|---|---|---|
| Shape | small tabular and time-series models | a language model |
| Input | this node's own history | the operator's questions, plus state |
| Output | intents, for Policy to approve | explanations, in words |
| Hardware | runs on a Raspberry Pi | an order of magnitude more |
| Trained on | your node | someone else's corpus |
| Failure mode | bad capital decisions | bad explanations |
| Authority | asks; Policy decides | none, ever |
| Required? | no — deterministic strategies work without it | no — Foundry runs headless |

Neither is required for Foundry to be useful. That is deliberate: a node management system
that only works with a GPU attached is not local-first in any meaningful sense.

## Routing intelligence, in the module

### What it is for

Peer classification, topology analysis, demand forecasting, channel recommendation, capital
allocation. Concretely: which peers are worth a channel, how much liquidity a channel needs,
when a rebalance pays for itself, which fee moves flow, when a channel should be closed and its
capital redeployed. Lightning Jet builds this. Foundry decides whether it may act on it.

None of this needs a large model. It is tabular prediction over a few thousand rows with
strong structure, which is the home ground of gradient boosting and small time-series models.
A model that fits in a few megabytes and infers in milliseconds is not a compromise here — it
is the right tool, and it is what makes the hardware invariant survivable.

### The data problem, stated plainly

This is the central risk to the entire thesis, and it deserves to be written down before
anyone starts training.

- **The sample is small.** A mid-size node might see a few thousand forwards a month. That is
  not much to learn from, and it does not grow quickly.
- **You see only your own traffic.** Forwards that routed around you are invisible. Demand you
  could not serve mostly looks like nothing at all.
- **The environment is non-stationary.** Topology, fee markets and competitor behavior shift
  underneath you, so yesterday's optimum decays.
- **Rewards are delayed and confounded.** A channel opened today pays back over months, during
  which everything else also changed.
- **There are no counterfactuals.** You cannot observe what the channel you did not open would
  have earned.

Those five conditions describe a setting that favors simple, robust heuristics over learned
policies. We may do all of this well and still find that a threshold rule wins. That outcome
is a real result, and [`roadmap.md`](roadmap.md) commits to publishing it.

### Earning authority

A model gets operational influence only by beating the deterministic baselines built in M3, on
**realized sats net of full costs**, over a **pre-registered window**, with **holdouts**, as
measured by Foundry's accounting rather than the module's own analysis. Not on prediction
accuracy — a model that forecasts routing demand beautifully and allocates capital badly is
worse than useless, because it is convincing.

Evaluation looks at realized economic return, liquidity cost, channel reliability, capital
efficiency, and policy violations. A model that improves returns while occasionally breaching
a limit has not improved anything; it has moved the risk somewhere the metric does not look.

Foundry's policy can hold a new module or model version to advisory: its intents are recorded
and judged but not approved, until its results beat the version before it. Even once it earns
influence, the module asks. Policy disposes. See [`architecture.md`](architecture.md).

### Running the module's models

Foundry runs the module, and the module runs its models. Foundry never looks inside them:

- **Containment.** CPU and memory caps, so inference never starves LND or Bitcoin Core, and
  no network path ([invariant 3](invariants.md#3-network-isolation)).
- **Verification.** The weights ship inside the module's release, and Foundry runs a release
  only if its version, hash and signature match
  ([invariant 4](invariants.md#4-verifiable-software)). The module downloads no models at
  runtime, and weights it trains on the node are recorded by version.
- **Attribution.** Every intent carries the module's version and its model's, so every approved
  action traces back to what chose it
  ([invariant 8](invariants.md#8-economic-accountability)).

See [`integrations/lightning-jet.md`](integrations/lightning-jet.md#running-jet).

## The local assistant

A language model that explains what the node is doing, why a rebalance happened, what a
number in the interface means. It is the education half of the project and the reason someone
who is not a Lightning specialist can operate a node at all.

**It is read-only by construction, not by configuration.** It holds no credentials and has no
path to Policy. It cannot take an action, cannot approve one, and cannot widen a limit. If it
hallucinates, it produces a wrong sentence, not a wrong transaction — and that is an
architectural property rather than a behavioral hope.

It is also optional and off by default on constrained hardware. Foundry without it is a node
manager with a terser interface.

## Training data and privacy

- **Training data stays local by default.** Your node's history is yours. A module learns from
  it on the node, and its sandbox has no network to send it anywhere.
- **Contributing data is opt-in, separately reviewed, and never a condition of running
  Foundry.** A node that refuses to share must work exactly as well as one that shares.
- Any shared corpus is sanitized under the same reasoning as the public event schema:
  aggregate, bucketed, no channel identity. Balance history is the most sensitive data a node
  holds, and a research dataset is a public feed with extra steps.

## Model artifacts

Models are dependencies, and [invariant 4](invariants.md) applies: a release identifies and
verifies the exact model artifacts it uses. Versioned, hashed, and reproducible from a
recorded training pipeline. For a module's models, the module's release is the artifact.

**Licensing of weights is an open question.** Open-source licenses were written for source
code and map awkwardly onto model weights. Foundry's code uses The Ooga Booga License and
Jet's uses MIT, but weights need a separate, explicit licensing decision rather than an
assumption that a code license covers them. For Foundry the question is which weight licenses
it will run, for the assistant and for a module. Record that decision in
[`decisions/`](decisions/) when settled.

## Where the AI is not

Stated because "AI-native" invites the opposite assumption:

- **Not in enforcement.** No limit, policy or dependency rule is evaluated by a model.
- **Not in the Dependency Guardian's verdict.** A model proposes dependencies; a deterministic
  validator decides. See [`invariants.md`](invariants.md).
- **Not between the operator and their funds.** Every path that moves money passes a check a
  person can read.

Foundry is AI-native in how it is *built* and how it *explains*. The intelligence that decides
lives in a module, and nothing it decides is *allowed* on its judgment alone.
