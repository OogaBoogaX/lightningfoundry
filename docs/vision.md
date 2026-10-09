# Vision

**Running a profitable routing node is a part-time job. Lightning Jet and Foundry together aim
to make it a decision you review, not a shift you work.** Jet optimizes the node. Foundry
operates it.

## Who this is for

**Operators who already run nodes.** You know what a rebalance costs. You have opinions about
fee policy. You have channels you should have closed months ago and know it. You are not short
of data — you are short of hours.

**Operators who gave up.** You ran a node, it worked, and it slowly became a chore: watching
liquidity drift, chasing peers who never route, rebalancing into channels that empty again by
morning, adjusting fees on instinct and never quite knowing whether it helped. The node did
not fail. It just stopped being worth the attention, so you stopped paying it.

**Operators who never started,** because the honest answer to "what do I need to know first"
is a year of accumulated judgment nobody has written down.

## What already exists, and where it stops

The tooling is genuinely good now. Rebalancing automation, policy-based fee setting, dashboards,
HTLC firewalls, analytics, scoring. Lightning Jet already rebalances on routing volumes and
missed forwarding opportunities. These are real tools that solve real problems.

**What they automate is execution. What they do not automate is judgment.**

A tool will rebalance a channel on a schedule. It will not tell you that this channel has
consumed more in rebalancing fees than it has ever earned, and should be closed. A tool will
set fees by a rule you wrote. It will not notice that the rule stopped working six weeks ago
because a competitor repriced. A tool will open a channel where you point it. Deciding *where
to point it* is still the operator's problem, and LND's own autopilot is the standing proof
that channel selection is unsolved rather than merely unimplemented.

So the operator remains the intelligence in the loop. Every tool makes the mechanics cheaper
and leaves the decisions where they were. That is why running a good node still takes years of
pattern recognition, and why people with the skill to do it well mostly have something better
to do with their evenings.

## Two projects, one node

**Lightning Jet optimizes the node.** It is becoming the intelligence the tools above lack:
rebalancing that asks whether it pays, liquidity forecasting, fees, topology analysis, peer and
channel scoring, channel sizing and retirement, capital allocation, and in time learning from
real outcomes. Jet stays independent. It runs on an existing LND node under the operator's own
policy, or managed by Foundry. Its goal is the open-source intelligence and optimization engine
for routing nodes.

**Foundry operates the node.** It runs the system Jet optimizes, and keeps it safe: a minimal,
verified Bitcoin Core and LND stack, installed reproducibly, supervised, backed up and
upgraded; credentials kept apart; and deterministic policy that decides whether each action
Jet wants to take is allowed. Its goal is the open-source operating platform for autonomous
Lightning nodes.

Jet decides what would improve the node. Foundry decides whether that action is allowed, keeps
the system healthy, and reports the real outcome back to Jet. Jet needs nothing from Foundry;
Foundry needs a module to optimize the node, Jet or any engine that keeps its interface. The
two can grow as separate open-source projects, with their own contributors, and together they
are the autonomous node. [Decision 0011](decisions/0011-jet-optimizes-foundry-operates.md)
records the split.

## What next level means

Three capabilities, in the order they get built, and all of them Jet's:

**Intelligent rebalancing.** Not "move liquidity when a threshold trips" but "is this rebalance
worth its cost, given what this channel actually earns, how quickly it drains, and what else
the capital could do." A rebalance that costs more than the forwards it enables is a loss
executed efficiently.

**Intelligent channel selection.** Which peers to open to, how much to commit, and — the part
nobody automates — when to stop. Closing a mediocre channel and redeploying its capital is one
of the highest-value decisions an operator makes, and it is almost entirely manual because
nothing measures the opportunity cost well enough to argue for it.

**Capital allocation.** Treating the node as a portfolio rather than a collection of channels.
Where the sats are working, where they are parked, and what moving them would cost.

Underneath all three: **accounting that tells the truth,** and that part is Foundry's. Most
tooling reports revenue. Revenue minus rebalancing costs minus amortized channel lifecycle
costs, measured against the capital committed, is a different number — often a much less
flattering one. See [`economics.md`](economics.md). That measurement is not a feature, it is
the precondition for any of this being real, which is why it lands before the intelligence
does. Jet's choices are judged by Foundry's numbers, not by its own.

## The smallest safe node

Foundry's research question: **what is the minimum software and hardware that can run a
secure, reliable, autonomous Lightning routing node?** Answering it means going deep into Linux
internals, storage, memory, CPU and networking; surviving power loss; working out what local
models need; and seeing how small the attack surface can get. The answers become reference
hardware configurations anyone can build. See [`roadmap.md`](roadmap.md).

## The two promises

**To the experienced operator:** the decisions you currently make on instinct get made on
evidence, and you stop being the bottleneck. Jet proposes, Foundry checks every proposal
against your limits, and you approve what your policy sends you. Over time you approve less,
because the proposals stop surprising you. Your node stops being a shift.

**To the newcomer:** you do not need a year of accumulated judgment to run a node that earns
its keep. The system explains what it is doing and why, in words, while it does it. The
learning curve becomes something you climb while operating rather than before starting.

Both promises have the same shape: **move the expertise into software that can explain
itself,** and keep the operator in authority rather than in the loop.

## What we refuse

- **No custody.** Foundry never holds a seed and never needs one.
- **No cloud control plane.** Nothing required to run a node lives on someone else's computer.
- **No telemetry.** There is no opt-out because there is nothing to opt out of.
- **No opaque automation.** If an action cannot be explained, Foundry should not allow it.
- **No model with authority it did not earn.** Deterministic limits constrain the system, and
  a model cannot widen its own mandate.

These are written as [invariants](invariants.md) because principles that apply only when
convenient are marketing.

## What we are honest about

**Routing may not be profitable at small scale.** Once capital cost is counted properly, many
nodes that look profitable are not. If Foundry's own measurements show that reliably, we
publish it as clearly as we would publish a success. Operators deserve to know whether they are
running a business or subsidizing a network.

**The learning problem is hard.** A node sees only its own forwards. The sample is small, the
environment shifts, rewards arrive late and confounded, and there are no counterfactuals. Those
conditions favor simple heuristics. We may build excellent measurement and find that a
threshold rule beats everything Jet trains — and [`roadmap.md`](roadmap.md) commits to saying
so.

**Autonomy is a risk, not a feature.** Software that moves money unattended will eventually do
something its authors did not intend. The design answer is bounded blast radius, not confidence.

## Why local, and why open

You are being asked to let software manage your money. Nobody should accept that from a binary
they cannot inspect, talking to a server they do not control, run by people who can change the
terms later.

Local-first is not a preference here. A routing node's operational data — balances, peers,
timing — is precisely what an adversary needs to attack it. Software that ships that somewhere
for analysis has manufactured the vulnerability it claims to manage.
