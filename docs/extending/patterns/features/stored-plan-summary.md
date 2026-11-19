# Feature Pattern: Stored Plan Summary

[← Back to Design Patterns](../index.md)


## Purpose

Give every report, pivot, AI query and cube post one fast, consistent read of "the plan in force for this cycle and
unit", at any volume, including parents rolled up the unit hierarchy. Use it when results are calculated per item and
must be totalled by account, scenario and year quickly. Posting must read exactly what reports show.

## Recommended approach (Verified in Business Planning)

- **Compressed results at the bottom.** `WfItem_Activity_Period` stores `ScenarioFlags`, `StartPeriod`, `EndPeriod` and
  a **per-period** value. A range of 12 months at 10,000 means 10,000 each month.
- **Member combinations.** Each activity's 13 resolved members (cube, entity, account, flow, IC, UD1-UD8) are stored once
  in `BpMemberCombination`, which is unique on a key of the member ids and created on the fly under `updlock, holdlock`.
- **Cycle revision rows choose revisions.** For each cycle and unit, the cycle revision rows choose which revisions and months count (see
  [Item Revisions](item-revisions.md)).
- **The summary table** (`BpForecastSummary`) holds, per cycle and unit:
  - basis and branch;
  - item type, category, measure and combination;
  - scenario and posting flag;
  - `M01`-`M60` and `Y1`-`Y5`. `Y` follows the measure's aggregation: Sum, Average (÷12), or EndingBalance (December).

  A base unit's rows come from its cycle revision rows (`BpBuildForecastSummary`). A parent's come from its direct children's rows
  (`BpRollupForecastSummary`), following the cycle type's hierarchy in force on the cycle's start date.
- **Kept current on write, never on render:**
  - Every change stamps `ChangedAt` on the unit and its ancestors, in every cycle. `BuiltAt` is stamped with the time
    the build *started*, so a change during a build still leaves it out of date.
  - After every write, once per write, `KeepDerivedCurrent` rebuilds only what is out of date: cycle revision rows for the changed
    items, then the summary for the item type × category groups they touch, then the parents, bottom up.
  - An unchanged write costs one query.
  - **Rebuild Summary** (incremental or forced) is the safety net for changes made outside the handler.
- **Reads are inline table-valued functions keyed on cycle and unit**, behind Direct SQL views (`WITH VIEW_METADATA`,
  one derived unique key). The grid's workflow filter lands on the driving tables, so only the selection is computed.
- **Posting reads the summary**, so the cube and the reports agree by construction.

## Framework capabilities

- **Direct SQL views** registered by the structure file. See
  "Direct SQL view" in the Claude skill's `references/design-patterns.md`; workflow
  columns on them use `KeyFlags: None`.
- **"Derived data: keep it current on write"** and **"Summary and top-N views"** in
  the Claude skill's `references/design-patterns.md`.
- **Hand-written SQL run by the setup package** (`510`, `530`, `540`). See
  [Setup Packages](../../setup-packages.md) and the Extension Kit's `docs/sql.md`.
- **Summary views (`_mS`) and compare-tos** for the pivot: the Plan and Upside scenarios in columns beside Baseline.

## Adaptation guidance

- **Small volume** (thousands of rows, one horizon)? A plain summary view over the results may be fast enough. Measure
  first. Add the stored table when reads are measured as slow, or when posting must equal reporting.
- **Horizon width.** Size `M`/`Y` columns to the longest horizon (60 months here). The columns hold relative months of
  the cycle, not calendar months.
- **Grain.** Group by what the reports and the cube need. The source keeps item type, category and measure so drill-down
  works. Add item grain in a separate result (`540`'s item view) rather than widening the summary.
- **Hierarchy.** Roll up by the hierarchy in force for the cycle. Reconsolidate after a retroactive hierarchy change.

## Important considerations

- **Every write path must keep it current:**
  - saves, including BeforeSave's redirect paths, which can leave AfterSave with no rows;
  - bulk updates;
  - deletes;
  - every custom action.

  A data import that bypasses the handler leaves results missing until Calculate.
- **Never rebuild in a render hook.** The source moved catch-up out of render (R057) because a read must not cost a
  build, and AI agents and exports read too.
- **Set-based SQL bypasses the Model**, so no save handler on these tables will fire.
- **Check the execution plan** for seeks on the driving tables. A function the optimizer cannot inline scans everything.
- **Open:** the v0.2 note's read-path speed at representative volume is not measured. The source logs ~1.4 s to build
  5,000 revisions for one unit. The README's check script `591` verifies the diff build.

## References

- Summary and marking: `BpHandlers/BpPlanningHandler.Summary.cs`. Its header lists every write path.
- SQL: `solutions/BusinessPlanning/510-BusinessPlanning-view-WfItemActivityMonth.sql` (range functions),
  `solutions/BusinessPlanning/530-BusinessPlanning-proc-ForecastSummary.sql` (build, roll-up, unit tree),
  `solutions/BusinessPlanning/591-BusinessPlanning-check-ForecastSummary.sql` (consistency check).
- Used by: [General Planning](../solutions/general-planning.md). Related: [Layered Cube Posting](layered-cube-posting.md).
