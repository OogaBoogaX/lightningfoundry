# 0011. Lightning Jet optimizes the node; Foundry operates it

**Status:** accepted, 2026-10-08. Supersedes [0009](0009-rebalancing-in-lightning-jet.md).

## Decision

Lightning Jet is the node's intelligence, and Foundry is the platform that runs it.

- **Jet decides and acts.** Circular rebalancing, liquidity forecasting, fees, topology
  analysis, peer and channel scoring, channel sizing, capital allocation, channel retirement,
  profitability analysis and, in time, learning from real outcomes are Jet's.
  [Lightning Jet](https://github.com/drneski/lightning-jet) is an independent project. It runs
  standalone on an LND node, under the operator's own policy, or managed by Foundry.
- **Foundry operates and authorizes.** It runs a minimal Bitcoin Core and LND stack, installs
  and configures it reproducibly, supervises it, keeps credentials apart, backs it up and
  upgrades it, keeps the accounts, publishes the sanitized feed, and researches the smallest
  system that can do all of this safely. It decides whether each action Jet wants to take is
  allowed, and its check is final.
- **Approval comes before the call.** In managed mode Jet sends Foundry an intent. Foundry's
  policy approves it at once, or asks the operator and waits; a standing budget is an approval
  given in advance. Jet then makes the LND call itself. The macaroon it acts with carries a
  custom caveat, so LND holds the call until Foundry's Policy, registered as LND's RPC
  middleware for that caveat, has checked it against an approved intent.
- **Any module can take Jet's place.** Foundry needs a module to optimize the node, and a
  module needs only to keep the interface Foundry defines.

## Alternatives

- **Decision 0009's split.** Foundry keeps the capital decisions and Jet only moves liquidity.
  That puts a second intelligence in Foundry, beside the one Jet is becoming, and splits one
  set of judgments across two projects.
- **Foundry executes Jet's recommendations.** Jet would hold no credential in managed mode,
  but it would run differently managed and standalone, and the execution it is built around,
  routes, probes and retries, would move into Foundry.
- **One project.** Folding Jet into Foundry, or Foundry into Jet, would tie two different
  crafts, routing economics and secure systems, to one schedule and one community.

## Why

Optimizing a node and operating one safely are different crafts. Jet already does the first,
and its judgment grows by learning from outcomes. Foundry's value is the second: a small,
verifiable platform where software can be trusted near money because nothing it does escapes a
deterministic check. Approving intents and checking calls keeps that check structural rather
than polite, and lets Jet run the same code standalone and managed. Two projects can grow two
communities, and an interface any module can keep means neither is held hostage by the other.

## Consequences

- Routing intelligence leaves Foundry. Foundry keeps the local assistant, and the rules a
  module's models are run and judged by. M5 becomes judging the intelligence rather than
  building it.
- Foundry holds no credential that moves money. Policy holds a macaroon that can only register
  the gate and revoke macaroons, and the module's acting macaroon is refused while Policy is
  absent. Unregistering Policy is the kill switch.
- In managed mode, money moves only through LND under the module's gated macaroon. A move
  another daemon makes with its own credentials, such as a Loop swap, goes through the gate or
  is not offered.
- Both projects judge profitability: Jet by the rules Foundry sets for it, Foundry by its own,
  and Foundry's check is final. Both are measured against Foundry's accounting, never against
  the module's own analysis.
- A module reads Foundry's history and outcomes through the interface, which is versioned like
  a public contract, never through `foundry.event.v1`, whose consumers stay Foundry's own
  components.
- Foundry's milestones are tested against a scripted stand-in module, so none waits on a Jet
  release.
- Standalone Jet enforces the operator's policy itself, and its limits should be deterministic
  code, as Foundry's are.
- Rollback means software rollback, until a new version touches state that cannot go back: a
  database migration, channel state, anything on-chain. Before an upgrade, Foundry says
  whether it migrates a database.
- The internal schema's `trigger` on `rebalance.started` (`operator`, `policy`, `jet`,
  `model`) predates this split. Narrowing it removes values, which the versioning rules treat
  as a `foundry.event.v2` change, so it waits for the interface's schema.
- [`integrations/lightning-jet.md`](../integrations/lightning-jet.md) sets out the two modes,
  the gate and the interface. The architecture, AI strategy, vision, roadmap, README and
  AGENTS.md change to match, with smaller edits elsewhere.
