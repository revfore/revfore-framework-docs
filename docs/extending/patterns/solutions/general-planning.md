# Solution Pattern: General Planning

[← Back to Design Patterns](../index.md)

Source: Business Planning (SaaS), `solutions/BusinessPlanning/`. Design:
`solutions/BusinessPlanning/010-BusinessPlanning-design.md`. Handler: `BpHandlers/`.

## Purpose

A multi-year financial plan built bottom-up from business items (contracts, services, positions, G&A, assets),
priced by shared drivers, in several scenarios (Baseline, Plan, Upside), and posted to a OneStream Finance cube per
workflow unit. It answers what the plan is now, what it was when a unit closed, and how one scenario or plan branch
differs from another, without copying the item population for every cycle.

The same design serves a budget, a rolling forecast or a proforma. The source is a rolling five-year forecast, so its
own names say "forecast"; see [Words used](../index.md#words-used).

## Core capabilities (Verified)

- **Items with typed assumptions.** One revision grid per item type, with its terms (1:1) and its input ranges
  (`ScenarioFlags`, `StartPeriod`, `EndPeriod`, value).
- **Scenarios as bit flags on the input ranges**, all entered as full values. One stored result serves every scenario
  whose numbers agree. See [Scenario Flags](../features/scenario-flags.md).
- **Revisions and commit boundaries.** Working, Draft and Committed states; committed data never changes; Close Unit
  freezes a unit's plan for the cycle with no copy. See [Item Revisions](../features/item-revisions.md).
- **Shared drivers** (prices, rates, schedules, benefits), resolved up the unit hierarchy and per month. A driver change
  reprices the items it drives. See [Shared Drivers](../features/shared-drivers.md).
- **Plan branches.** Alternative copies of the plan data. Versions change a closed plan and stack on each other; Proposals are private or public what-ifs.
  Both are picked in Change Workflow. See [Plan Branches](../features/plan-branches.md).
- **Plan layering in the cube.** Scenarios post as stacked UD5 deltas (Baseline, Plan minus Baseline, Upside minus
  Plan), and plan branches as UD6 deltas over Master. The cube's parent members add the layers back up. See
  [Layered Cube Posting](../features/layered-cube-posting.md).
- **A stored plan summary** per cycle and unit, rolled up the unit hierarchy and kept current on write. Reports,
  pivots and posting all read it. See [Stored Plan Summary](../features/stored-plan-summary.md).
- **One workflow selection drives everything.** Cycle, unit and two compare-tos are picked once in Change Workflow,
  and every page follows. Business Planning extends the selection with a workflow scenario per planning scenario and a
  plan branch beside each compare-to. See [Workflow Context](../features/workflow-context.md).

The solution does **not** have plan-to-actual comparison. A cycle's `ActualEndPeriod` (a core column) only sets where
the plan starts. It has no financial-statement engine either: accounts come from category member sets.

## Recommended design

1. **Separate identity, assumptions and results.**
   - `WfItem` is the permanent identity, with a permanent `MasterNumber`.
   - `WfItem_Revision` is an immutable package of assumptions from an `EffectivePeriod` forward.
   - Domain tables hang off the revision.
   - Results are `WfItem_Activity`: one measure × one unit × one resolved member combination. Under each activity,
     `WfItem_Activity_Period` holds compressed ranges.

   Only the handler writes results.
2. **Items carry no cycle.** The workflow supplies the cycle at read time. The unit's boundary in that cycle decides
   which revisions count, and the cycle's years decide which months. One population serves every past and future cycle.
3. **Configuration over code for outputs.** Each item type has a configuration (`WfItemConfig`) listing its measures.
   Each measure carries its member set type, aggregation (Sum, Average, EndingBalance) and a posting flag. Accounts come
   from the core category member sets (`WfItemCategory_MemberSet`) for that member set type; entity and UDs come from
   the unit's member set.
4. **One handler, one save path.** Grids, imports and AI agents (MCP) all save through the same views and the same
   `IExtensionHandler`. The handler redirects edits of committed data, calculates, and keeps derived data current.
5. **Set-based SQL for volume.**
   - Revision copies, activities, cycle revision rows, the summary and posting rows are written by SQL in the handler and in
     shipped scripts (`510`, `530`, `540`), not row by row through the Model.
   - Reporting reads go through inline table-valued functions keyed on cycle and unit, so only the selection is
     computed.
6. **Post from the same data reports read.** Posting reads the stored summary, so what reports show and what posts
   cannot disagree.

## Framework capabilities used

| Framework capability | Used for | Reference |
|---|---|---|
| Revfore workflow: units, hierarchies, planning cycles, workflow scenarios, close by unit | Cycles, units, Parent / Base units | the Claude skill's `references/workflow-enabled-views.md`, the Claude skill's `references/core-workflow-configuration-tables.md` |
| Workflow-item tables (`WfItemType`, `WfItemCategory`, `WfArea`, `WfItemCategory_MemberSet`, `WfUnit_MemberSet`) | Item types, number prefixes, categories, accounts per measure | "Workflow-item solutions" in the Claude skill's `references/design-patterns.md` |
| Views workflow-enabled by unit only | Item grids across cycles | the Claude skill's `references/workflow-unit-profiles-and-view-flags.md` |
| Workflow context handlers and `{WfExtInt_<key>}` run-time filters | Plan branch picking in Change Workflow | the Extension Kit's `docs/workflow-context.md` |
| Compare-tos and member sets (scope / base) | Plan-scenario and Upside-scenario columns; UD5 / UD6 scope | the Extension Kit's `docs/workflow-context.md` |
| Cube sync (`UpdateCubeHelper`, Sync with Controlled Itemized) | Posting each layer | the Claude skill's `references/onestream-development-guide.md` |
| Direct SQL views (`_dV`), hand-written SQL in the setup package | Summary and cycle-revision reads | "Direct SQL view" in the Claude skill's `references/design-patterns.md`, [Setup Packages](../../setup-packages.md) |
| Custom view actions, render hooks (`Set_ColumnDefaultValue`) | Commit, Close Unit, Lock & Post; page defaults | the Extension Kit's `docs/actions.md`, the Claude skill's `references/render-hooks.md` |
| Solution guide and semantic declarations | AI agents saving and querying items | the Claude skill's `references/solution-guide.md` |

The planning layer itself (`WfItem*`, `WfCycleUnitBoundary`, `WfCycleRangePeriods`, `WfItemBranch*`, `BpForecastSummary`,
`BpPosting`) lives in the **extension** schema and is solution code, not core.

## Feature patterns

Most solutions need only some of these:

- [Scenario Flags](../features/scenario-flags.md): needed whenever there is more than one scenario.
- [Item Revisions and Frozen Plans](../features/item-revisions.md): needed when closed plans must reproduce
  exactly.
- [Shared Drivers](../features/shared-drivers.md): needed when rates are maintained centrally.
- [Plan Branches: Versions and Proposals](../features/plan-branches.md): add for what-ifs and post-close corrections.
- [Layered Cube Posting](../features/layered-cube-posting.md): add when scenarios or plan branches must be compared in the
  cube.
- [Stored Plan Summary](../features/stored-plan-summary.md): needed at volume and for posting.
- [Workflow Context](../features/workflow-context.md): the core is needed whenever there is more than one cycle or
  unit. Its scenario and branch parts come with the patterns above.

## Adaptation guidance

- **Item types and domains.** Replace contracts, positions and so on with the business's own. Each needs:
  - typed terms and range tables under the revision;
  - one calculation that turns them into measures;
  - a configuration row listing those measures.
- **Accounts and dimensions.** Keep the mapping in data (category member sets per measure type, the unit's member set),
  never in code. Add measures by configuration.
- **Scenarios.** Any number up to the bit width. Decide early whether they post as full values (simple, separate cube
  members) or as layers (see [Layered Cube Posting](../features/layered-cube-posting.md)).
- **Horizon and frequency.** Business Planning uses five years: 1-2 monthly, 3-5 one YTD value at P12, set per cycle in
  `WfCycleRangePeriods`. Calculations stay monthly, and only presentation changes.
- **Workflow.** The source has no approval: units just close. Add approval through the Framework's workflow statuses if
  needed. Parents can hold drivers only, as here, or items too.
- **A smaller requirement** (one scenario, no closing, no cube) needs none of revisions, plan branches or layering. Use plain
  header/child tables (see the Claude skill's `references/design-patterns.md`) and
  add these patterns only when the requirement appears.

## Important considerations

- **Complexity is real.** The source is around 6,000 lines of handler code and about 1,800 lines of shipped SQL across 52+
  design revisions. Adopt each feature only for a stated need.
- **Cube members are a dependency.** Finance must create the Scenario per cycle, the UD5 layers and the UD6 pool, with
  their `_Total` parents, before the first post. The source's README lists them.
- **Set-based SQL bypasses Model save hooks.** No handler registered on the results or posting tables will run for those
  writes. Every write path must also keep derived data current: see
  [Stored Plan Summary](../features/stored-plan-summary.md).
- **The source was refined over many rounds.** The design document's newer rounds (top of the file) replace older
  sections, so read rounds 20-24 before sections 1-10. An earlier design note
  (Revfore Workflow Planning Pattern, v0.2) also describes ideas that are **not built**: actuals at item grain, archiving, a proposal review lifecycle, and physical storage options.
- **Open:** the source README keeps a "To verify on the first live run" list:
  - the multi-member UD5 bracket in the sync's POV filter;
  - Clear Cube on delete;
  - planner security;
  - the speed of the inline-function reads at volume.

  Treat those behaviours as unproven until it is cleared.
