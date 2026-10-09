# Lightning Jet

Lightning Jet optimizes the node. Lightning Foundry operates it. Ooga Booga Land makes it
visible and understandable.

[Lightning Jet](https://github.com/drneski/lightning-jet) is an independent, open-source
intelligence engine for LND routing nodes, and Foundry is the platform that runs it. Jet
decides what would improve the node. Foundry decides whether that action is allowed, keeps the
system healthy, and reports the real outcome back to Jet.
[Decision 0011](../decisions/0011-jet-optimizes-foundry-operates.md) records the split.

## Who does what

**Jet decides and acts:**

- circular rebalancing and liquidity forecasting;
- fees;
- topology analysis, and peer and channel scoring;
- channel sizing, retirement and replacement;
- capital allocation, including swaps, bought inbound liquidity and splicing;
- profitability and performance analysis, and in time, learning from real outcomes.

**Foundry operates and authorizes:**

- a minimal Bitcoin Core and LND stack, installed and configured reproducibly;
- process supervision and health monitoring;
- credentials kept apart;
- deterministic policy, and the approval of every action a managed module takes;
- backups, recovery, upgrades and software rollback;
- accounting: the record every action is judged against;
- the interface a module keeps;
- the sanitized feed for Ooga Booga Land, and the local assistant.

## Two modes

### Standalone

Jet runs on an existing LND node and enforces the operator's own policy. It acts within the
limits the operator set, and prompts them when the policy says to. Its macaroon can do
whatever the operator granted, and nothing outside Jet checks how it is used, so Jet's limits
should be deterministic code, not a model's judgment.

If Foundry runs on the same node, it observes what Jet does, accounts for it, and does nothing
else. M2 and M3 run this way.

### Managed

Foundry adds its policy on top of Jet's, and its check is final. Jet still does the work; the
difference from standalone is approval:

1. **Intent.** Before acting, Jet sends Foundry an intent: what it means to do and within what
   bounds, for example "rebalance 500,000 sats from A to B, fee cap 300 sats" or "close channel
   C", with the Jet and model versions that chose it.
2. **Approval.** Foundry's policy approves the intent at once, or sends it to the operator and
   waits. A standing budget, such as rebalances under a daily ceiling, is an approval given in
   advance.
3. **The call.** Jet makes the LND call itself, exactly as it would standalone.
4. **The check.** The macaroon Jet acts with carries a custom caveat, so LND hands the call to
   Foundry's Policy, registered as LND's RPC middleware for that caveat, before running it.
   Policy lets it through only if it matches an approved intent: same channel, amount within
   bounds, fee within the cap.
5. **The outcome.** Foundry's accounting records what happened and reports it back to Jet.

The check is the gate, and it holds without Jet's cooperation:

- **It fails closed.** If Policy is not registered, LND refuses the macaroon outright. If
  Policy does not answer in time, the call fails.
- **It is the kill switch.** Unregistering Policy stops Jet at its next call. Jet's acting
  macaroon is baked with its own root key, so revoking that key stops it for good and touches
  nothing else.
- **It is never mandatory.** Policy is not registered with `rpcmiddleware.addmandatory`. That
  would block every RPC call while Foundry is down, the operator's own included, and break
  [invariant 7](../invariants.md#7-failure-isolation).
- **Reads go around it.** Jet reads with a second macaroon that is read-only and carries no
  caveat.
- **Money moves only through it.** Anything that moves money goes through LND under the gated
  macaroon. A move another daemon makes with its own credentials, such as a Loop swap, goes
  through the gate or is not offered in managed mode.

Managed mode needs `rpcmiddleware.enable=true` in LND, which is off by default.

### What Policy allows

A call passes only when it matches an approved intent. Anything else is denied, including any
call Policy does not recognize, so a new LND call stays closed until Policy learns it.

- **Limits bind every approval:** the daily rebalance fee ceiling, the channel close ceiling,
  the reserve floor, and how often a channel's fee may change. A fee that changes often floods
  gossip, gets throttled by other nodes, and fails some payments made against the old fee.
- **Never, even with an intent:** sending on-chain funds anywhere but the node's own wallet and
  channels, and any macaroon or wallet operation.

The daily ceiling is also what bounds fee farming, where a peer who can predict rebalances
provokes them to collect the fees ([`threat-model.md`](../threat-model.md)). Standalone, Jet's
own limit is the only bound.

Under supervision, from M4, the operator approves each intent and Policy checks every call
against it. Nobody approves calls one by one; the middleware's timeout is measured in seconds.

## Two layers of economics

In managed mode both projects judge profitability. Jet runs the profitability rules Foundry
sets for it, Foundry's policy runs its own, and Foundry's check is final. Both are measured
against the same record: Foundry's accounting, under the definitions in
[`economics.md`](../economics.md) and the attribution rules of
[invariant 8](../invariants.md#8-economic-accountability). Jet's own analysis informs Jet; it
never grades it.

## The interface

Foundry owns the interface and versions it like a public contract, with conformance tests any
module can run. A module that passes them can be managed, and Jet is the first. Foundry needs
one: without a module, a Foundry node runs and reports, but nothing optimizes it. The schema is
not written yet; this is what it carries.

**From the module to Foundry:**

- **Intents,** each with the module's version and its model's.
- **Forecasts and analysis,** optionally, labeled derived. They inform Foundry's policy; they
  never change a limit.

**From Foundry to the module:**

- **Verdicts** on each intent.
- **Rules and budgets:** the profitability rules and spending budgets the module works within.
- **Withdrawals.** When an approval no longer stands, for instance because the operator
  changed a limit, the module drops any work pending on it.
- **Outcomes and history:** what each approved action did and cost, and the node's history to
  learn from, on the node.

**From the module to LND, through the gate:** the calls themselves.

A module reads Foundry's history through the interface, never through `foundry.event.v1`,
whose consumers stay Foundry's own components
([`event-model.md`](../event-model.md#versioning)).

## Running Jet

In managed mode Foundry runs Jet, and Jet runs its models. Foundry never looks inside them:

- **Containment.** Foundry starts and stops Jet, caps its CPU and memory so inference never
  starves LND or Bitcoin Core, and gives it no network path
  ([invariant 3](../invariants.md#3-network-isolation)).
- **Verification.** Foundry runs a Jet release only if its version, hash and signature match
  ([invariant 4](../invariants.md#4-verifiable-software)). Weights that ship inside the
  release are covered by that check, so Jet downloads no models at runtime, and weights it
  trains on the node are recorded by version.
- **Attribution.** Every intent carries Jet's version and its model's, so every approved action
  traces back to what chose it.
- **Earning trust.** Foundry's policy can hold a new Jet or model version to advisory: its
  intents are recorded and judged but not approved, until its results on Foundry's accounting
  beat the version before it.
- **Training data stays on the node.** Jet learns from this node's history, on this node, and
  its sandbox has no network to send it anywhere
  ([`ai-strategy.md`](../ai-strategy.md#training-data-and-privacy)).

Jet's models decide what Jet asks for, never what is allowed. In managed mode Foundry's Policy
is that check; standalone, Jet's own deterministic limits are
([invariant 6](../invariants.md#6-deterministic-security)).

## Jet's versions

- **Jet 1.x** runs standalone only, as it does today, and 1.6.1 is an incremental release. Its
  behavior is one of the baselines M3 measures.
- **Jet 2.0** is the rework built to be managed, and the intelligence engine described here.
  To run as part of a Foundry node it must meet invariants
  [1](../invariants.md#1-minimal-trusted-computing-base),
  [3](../invariants.md#3-network-isolation) and [4](../invariants.md#4-verifiable-software):
  pinned and hashed dependencies with no install scripts, no network path beyond the node it
  optimizes, and verified artifacts. Jet 1.6.0 does not meet them yet: its dependencies float,
  it runs a post-install script, two of its modules run install scripts for native code, and it
  ships a Telegram client. Alerts can stay, as a separate notifier that holds no credential.

## Adoption

Foundry runs on a compatible node that already exists, or installs with Bitcoin Core and LND
([decision 0010](../decisions/0010-adopt-or-install.md)). The path this integration opens:

```text
an LND node  →  Jet, standalone  →  Foundry adopts the node  →  Jet, managed
```

The compatibility list for an adopted node includes `rpcmiddleware.enable`.

## Where the work lives

- **Lightning Jet** owns the intelligence: its strategies, its models, standalone mode, Jet
  2.0, and its side of the interface.
- **Foundry** owns the platform: the interface, Policy and its gate, the accounting, the
  conformance tests and a scripted stand-in module.

Jet decides what would improve the node. Foundry decides whether it may. Neither repository
decides the other's half.
