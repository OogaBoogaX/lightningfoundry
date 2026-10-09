# Architecture

The intended shape, written before the runtime exists so the boundaries are decided rather
than discovered. Expect this document to change as real components land; expect the trust
boundaries not to.

## Layers

```text
┌──────────────────────────────────────────────────────────────┐
│  Local UI          Assistant (explains, changes nothing)     │
├──────────────────────────────────────────────────────────────┤
│  Module (Lightning Jet) — decides and acts; Policy approves  │
├──────────────────────────────────────────────────────────────┤
│  Policy — deterministic limits. The only path to an action   │
├──────────────────────────────────────────────────────────────┤
│  Foundry Core — observation, accounting, events, supervision │
├──────────────────────────────────────────────────────────────┤
│  LND                                                         │
│  Bitcoin Core                                                │
└──────────────────────────────────────────────────────────────┘
```

Read it as a permission gradient. Everything above Policy can *ask*. Only Policy can *allow*.
The module acts itself, and LND carries out each change it asks for only once Policy has
allowed it.

The module is the node's intelligence: Lightning Jet, or any engine that keeps Foundry's
interface. It is not part of Foundry; Foundry runs it, contains it and decides what it may do
([decision 0011](decisions/0011-jet-optimizes-foundry-operates.md)).

## The one rule

**Nothing reaches LND except through Policy, and Policy is deterministic code.**

A module may decide anything. Before it acts, it sends Policy an intent, and Policy approves it
against limits the operator set, at once or after the operator agrees. The module then makes
the call itself, and LND holds the call until Policy has checked it against an approved
intent. That check contains no model, no heuristic that changes with training, and no path
that can be widened at runtime.

This is [invariant 6](invariants.md) as a structural property rather than a promise. If the
module is compromised, wrong, or replaced by something malicious, the worst it can do is ask
for actions that Policy refuses.

The check does not rest on the module's good manners. The macaroon it acts with carries a
custom caveat, so LND hands every request made with it to Policy first, through LND's RPC
middleware, and refuses the macaroon while Policy is absent. See
[`integrations/lightning-jet.md`](integrations/lightning-jet.md).

## Components

| Component | Holds | Responsibility |
|---|---|---|
| **Core / observer** | LND read macaroon | Subscribes to LND, normalizes to internal events, persists them |
| **Core / accounting** | nothing | Turns events into the measures in [`economics.md`](economics.md): the record every action is judged against, reported back to the module |
| **Core / supervisor** | control of the node's processes; no LND credential | Installs, starts, contains, backs up and upgrades Bitcoin Core, LND and the module |
| **Policy** | LND macaroon that can only register middleware and revoke macaroons | Approves intents against the operator's limits, and checks every call the module makes against them |
| **Module** | LND macaroon carrying Policy's caveat; a read-only macaroon | Outside Foundry: Lightning Jet, or another engine that keeps the interface. Decides and makes its own calls; every change waits in LND for Policy |
| **Assistant** | nothing | Explains state in language; read-only by construction |
| **Export** | export key, per-node credential | Translates internal events to the public schema and pushes them to one configured endpoint, opt-in; holds no LND credential |
| **UI** | nothing | Local interface for policy, approvals and the node's state; talks to Core and Policy, not to LND |

Two things follow from the table. **LND credentials live with three components**, and they are
different credentials: the observer's can only read, Policy's can only gate, and the module's
can act only call by call, as Policy allows. And **Foundry holds no credential that moves
money**: the one that can act belongs to the module, and LND will not honor it without Policy.

## Trust boundaries

Four, in order of how much it costs to get them wrong:

1. **Seed and wallet.** Outside Foundry entirely. Foundry never holds, reads or needs one. When
   the supervisor restarts LND, unlocking the wallet is LND's own configuration, set by the
   operator.
2. **The acting credential and the gate.** The module holds the one credential that can act;
   LND asks Policy before honoring any call made with it, and refuses it while Policy is
   absent. Policy holds the gate, a macaroon that can register that middleware and revoke
   macaroons, and nothing else. Each alone moves nothing. Together they are the authority to
   act, so no component ever holds both. LND's own admin macaroon could act alone, so it stays
   out of reach: the packaged install initializes LND statelessly, so it is never written to
   disk, and on an adopted node no Foundry component can read it.
3. **Read credentials.** Held by the observer and the module. Leak operational data if
   compromised; cannot move funds.
4. **Export.** Internal events become public events here, by translation into a different
   schema — never by filtering fields out of the internal one — and leave the machine only as
   signed batches the node pushes out. Nothing can connect in to fetch them. See
   [`event-model.md`](event-model.md).

## Data flow

```text
LND ──(read macaroon)──► observer ──► internal events ──► store
                                              │
                          ┌───────────────────┴───────────────────┐
                          ▼                                       ▼
                     accounting                                export
                          │                                       │
                          ▼                                       ▼
               POLICY, and the module                       public events
                through the interface                 (opt-in, signed, pushed)
```

The store is the seam. Everything downstream reads events rather than querying LND directly,
which means accounting can be replayed against history, and a module can be tested against
fixtures and the simulator with no node present. Accounting gives Policy the measures it
checks intents against, and gives the module the outcomes of what it did, through the
interface.

A module asks before it acts. Its intents go to Policy, which approves them at once or after
the operator agrees. The module then makes each call itself, and LND holds every call made with
the gated macaroon until Policy has checked it:

```text
module ──(intent)──► POLICY ──(verdict)──► module
module ──(call, gated macaroon)──► LND ◄──(allow or refuse)── POLICY
```

Public events leave by one path: the export pushes signed batches to a single configured
endpoint, and the node listens for nothing. The Delivery section of
[`event-model.md`](event-model.md) sets out the rules.

## Failure isolation

What happens when each piece dies:

| Fails | Consequence |
|---|---|
| Assistant | Nothing. It only ever talked. |
| Module | No decisions and no actions. The node keeps routing; Foundry keeps watching, and the operator or another module can take over. |
| Policy | No intent is approved, and LND refuses the module's gated macaroon. The node keeps routing. |
| Observer | No new events. The node keeps routing; Foundry goes blind. |
| Supervisor | Processes keep running as they are; nothing restarts, backs up or upgrades them. |
| Export | The public feed stops. Nothing else notices. |
| Foundry entirely | LND and Bitcoin Core carry on, and the module's gated macaroon stops working. The operator resumes manual control. |

There is no failure mode in that table where Foundry breaking takes the node down with it.
That is the requirement, not a happy accident: Bitcoin Core and LND are the trust anchors, and
Foundry is a management layer on top of them.

## Deliberately outside

- **The Lightning protocol.** LND implements it. Foundry never will.
- **The node's intelligence.** Rebalancing, fees, channels, capital and the models behind them
  belong to Lightning Jet, or any module that keeps the interface. Foundry decides only whether
  each action is allowed — see
  [decision 0011](decisions/0011-jet-optimizes-foundry-operates.md).
- **Payment processing.** Donations, invoicing and merchant flows are a separate concern with
  a separate stack. BTCPay Server lives on Ooga Booga Land's payments side and is not a Foundry
  dependency — see [`integrations/obl-payments-poc.md`](integrations/obl-payments-poc.md).
- **Any cloud component.** There is no server half of this product.
- **Visualization.** Foundry emits events; consumers draw pictures.

## Open decisions

**Implementation language is undecided**, and deliberately so — there is no manifest in this
repository yet, because adding one would decide it silently.

The considerations: LND is Go and its gRPC bindings are first-class there. A component holding
the gate and enforcing capital limits has a real argument for a compiled, memory-safe
language. Lightning Jet is Node, but it meets Foundry at an interface and at LND's API rather
than in shared code ([decision 0011](decisions/0011-jet-optimizes-foundry-operates.md)), so it
does not argue for Node. The answer may still be "more than one," with the event store as the
seam between them.

Whatever the answer, it should be recorded in [`decisions/`](decisions/) with its reasoning,
because it is the kind of choice that is expensive to revisit and easy to forget the reasons
for.
