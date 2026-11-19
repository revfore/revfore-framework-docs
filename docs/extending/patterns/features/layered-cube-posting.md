# Feature Pattern: Layered Cube Posting

[← Back to Design Patterns](../index.md)


## Purpose

Post several scenarios and plan branches ([versions and proposals](plan-branches.md)) into one cube Scenario as **stacked deltas**, so the cube's own parent members show
each full plan and each difference. For example:
- `Plan_Total` = Baseline + PlanDelta;
- `V2_Total` = V1_Total + V2.

Each layer can be posted, reposted or cleared alone. Use it when the cube must compare scenarios or versions and Finance
wants one Scenario member per cycle, not one per scenario or branch.

## Recommended approach (Verified in Business Planning)

- **Cube layout:**

  ```
  UD5  Upside_Total = Plan_Total + UpsideDelta     UD6  V2_Total = V1_Total + V2   (versions stack)
       Plan_Total   = Baseline   + PlanDelta            V1_Total = Master   + V1
       Baseline     (full values)                       P1_Total = Master   + P1   (proposals on master)
                                                        Master   (full values)
  ```

- **Relational data stays full values.** Deltas are worked out only when posting:
  - **UD5:** each active scenario's rows go to its UD5 member. The same rows also go to the member of the scenario
    that Layers On it, with the sign reversed. Baseline is posted full.
  - **UD6:** the master layer posts full. A branch posts its plan minus what it stacks on (a version: its base
    version, or master; a proposal: master).
- **One posting table** (`BpPosting`) is filled from the [stored summary](stored-plan-summary.md) per cycle, unit and
  UD6 layer. Each row carries its own UD5 and UD6 members, and the integration code is the full cell key. A zero amount
  writes no row.
- **Sync once per year, per UD6 layer, all UD5 layers at once.** The sync's POV filter, which is the region it clears and
  reads back, is rewritten to `[U5#Baseline, U5#PlanDelta, U5#UpsideDelta], [U6#<layer>]`. Posting one layer therefore
  never touches another, and an empty layer still syncs, so it clears what it posted before.
- **Horizon presentation.** `WfCycleRangePeriods` says, per relative year, Periodic (P1-P12) or YTD (one value at
  `LastYTDPeriod`). A YTD year syncs only that period.
- **When posting happens:**
  - Master posts on Post to Cube, Close Unit, and after Calculate and Commit All on open units.
  - A branch posts on Lock & Post, Repost Version and Post Proposal, and as it is edited while picked.
  - Parent units never post.
  - Reposting a layer marks the layers stacked on it **Post Out Of Date**.

## Framework capabilities

- **Cube sync** (`UpdateCubeHelper.UpdateCube_GetUpdateCubeContext`, `UpdateCube`, mode `SyncWithControlledItemized`)
  over a summary view (`BpPosting_mS`). The where clause you pass is the whole filter, and the year must be in it. See
  the Claude skill's `references/onestream-development-guide.md` and
  the Claude skill's `references/core-dimension-and-cube-tables.md`.
- **Workflow scenario member sets.** Base UD5 is the layer posted to, and scope UD5 is the total read back. The workflow
  area is required by the sync even when every row carries its own data source.
- **Debug scopes** (`RF.ErrorLog.DebugScope`) log each sync's resolved context and POV. The context is always logged on
  failure.

## Adaptation guidance

- **Few scenarios, no comparison in the cube?** Post full values to separate Scenario members and skip layering. It is
  simpler, and the cube's own consolidation is not needed.
- **Choose the UD dimensions with Finance.** The source uses UD5 for scenarios and UD6 for plan branches because they were
  free. Any spare dimension works if its members carry the `_Total` parents.
- **Layer order** is data (`BaseBpScenarioId`). A scenario could layer on Baseline directly instead of on the Plan scenario.
- **Branch layers are optional** even when scenario layers are used.

## Important considerations

- **Finance must create the members first:** the Scenario per cycle, each UD5 layer, and the UD6 pool with `_Total`
  parents. Missing members stop the post with a message; nothing is posted half.
- **Stock measures** (FTE, balances) must not be summed across periods. Aggregation (Sum, Average, EndingBalance) is set
  per measure, and the source never posts statistical or balance-sheet measures.
- **Every posted account and UD7 must be a base member** inside the data source's controlled scope
  (`ControlledItemized_RFA`), or the sync rejects it.
- **Open (README, "Layered posting (R044)"):** confirm on a live run that `FilterMembers` accepts the multi-member UD5
  bracket, that rows land on their own UD5 and UD6, that the `_Total` parents add up, and that reposting one layer clears
  only that layer.

## References

- Posting: region "Post to Cube" (`PostLayer`, `SyncLayer`, `PointPovAtLayer`) in
  `BpHandlers/BpPlanningHandler.Cycles.cs`.
  The file header summarizes the layering.
- Members Finance must create: "Install" in `solutions/BusinessPlanning/000-BusinessPlanning-README.md`.
- Design: rounds 20-21 of `solutions/BusinessPlanning/010-BusinessPlanning-design.md`.
- Used by: [General Planning](../solutions/general-planning.md). Related: [Scenario Flags](scenario-flags.md), [Plan Branches](plan-branches.md).
