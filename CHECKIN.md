# Lab 5 check-in route

Use this as preparation. The TA check-in still requires explaining the work in
your own words and answering follow-ups without an agent.

1. Open `SMELLS.md`, Milestone 1. For each smell, point to the named method in
   the source, explain the unnecessary cost, and name the principle.
2. Open commit `a848941` and its `src/reservationManager.ts` diff. This is the
   complete production change: nine lines removed, no test changes.
3. Show `evidence/tests-after.txt` (39 passing tests) and
   `evidence/typecheck-after.txt` (successful typecheck). Explain the stronger
   local argument: the manager's private cache was empty and never populated,
   so a supported query could not take its cache-hit branch.
4. Return to Milestone 2 for the scope line. Removing an inert cache path is a
   refactoring. Implementing a working cache would introduce invalidation and
   stale-data behavior, so that is outside this change.
5. Show Milestone 3's two proposals and false positive. Pricing would become
   one shared pure rule; notification delivery would become an explicit
   dependency. The costs are shared-module coupling/rounding preservation and
   construction-site/API migration, respectively. Neither proposal is coded.

## Useful distinctions

- A code smell is a reason to investigate a maintenance cost, not proof of a
  functional bug. This starter already passed its tests.
- DRY means keeping one authoritative business rule, not eliminating every
  repeated character. Pricing policy is duplicated knowledge.
- YAGNI means not adding machinery before it has a concrete purpose. An
  interface can still be useful for testing even when a mutable global factory
  registry is unnecessary.
- A green suite supports preservation but does not prove every possible
  behavior. Combine the tests with reasoning about the changed path.
- Many validation guards can be cohesive: they check actual constraints in a
  deliberate order. Independent policy variation or duplicated checks would
  be a reason to reconsider that design.

Tool/model attribution is in `README.md`; no agent transcript is included in
this public lab repository.
