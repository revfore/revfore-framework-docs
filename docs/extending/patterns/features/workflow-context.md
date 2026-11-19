# Feature Pattern: Workflow Context

[← Back to Design Patterns](../index.md)

## Purpose

Make the user's workflow selection the solution's coordinate system. What is picked once in Change Workflow decides what
every grid shows, what an edit changes, what the summary pages compare and what posts. Nothing is picked per grid. Use
it whenever a solution has more than one cycle or unit, and users must always know which one they are looking at.

The pattern has a **core** that any workflow solution can use (cycle, unit, compare-to, view flags), and two **optional
extensions** that only apply with other patterns:
- **Scenario layers:** several workflow scenarios per cycle, told apart in the cube. These come with
  [Scenario Flags](scenario-flags.md) and [Layered Cube Posting](layered-cube-posting.md) (Baseline, the Plan scenario,
  Upside).
- **Branch picking:** Branch and Compare Branch fields in Change Workflow. These come with
  [Plan Branches](plan-branches.md) (versions and proposals).

Most solutions will use the core alone.

## Recommended approach (Verified in Business Planning)

| Selection | Carried by (Framework) | Business Planning adds | Part |
|---|---|---|---|
| **Cycle** | Planning cycles (`WfCycle`: year, `ActualEndPeriod`, compare-tos) | Range periods per cycle (Periodic / YTD years); each unit's boundary in the cycle | Core |
| **Unit** | Workflow units and the cycle type's hierarchy | Parent units hold drivers and review roll-ups, never post | Core |
| **Compare To 1 / 2** | Core compare-to: a workflow scenario and a year offset | - | Core |
| **Workflow scenario** | Workflow scenarios with member sets (base / scope) | Three per cycle year on one cube Scenario, told apart by UD5 | Scenario layers |
| **Branch, Compare Branch 1 / 2** | Workflow context handler: extra fields in Change Workflow, saved on the user's context | Master, the cycle's versions, the proposals the user may see | Branch picking |

### Core: any workflow solution

- **Items can live across cycles.** Grids of things that outlast a cycle (items, revisions) are workflow-enabled **by
  unit only**. The cycle is applied by what the solution reads for it (here, the cycle revision rows), not by the view.
- **Choose each view's workflow flag deliberately:**
  - **By unit only:** grids of cross-cycle records.
  - **By cycle and unit:** summary and results views.
  - **Not workflow-enabled:** drills, and admin and lookup views.
- **Parent and base units play different roles.** Parents review roll-ups (and here hold the shared drivers), and base
  units hold the data and post. Roll-ups follow the cycle type's hierarchy in force on the cycle's start date.
- **Unit column for Parents only.** On workflow-enabled grids the Unit column shows only when a Parent unit is selected
  (`WfUnitProfileTypeFlags = Parent`). On a base unit it would repeat one value down every row. The structure generator
  sets this on every such grid.
- **Compare-tos for side-by-side columns.** Core's compare-to (a workflow scenario and a year offset) gives comparison
  columns with no solution code. Use it for this year against last year, or plan against another scenario.
- **Page defaults from the page.** BeforeViewRender sets the item type of each item page (`Set_DefaultValue` on the view
  column), so new rows need no picking.

### Extension: scenario layers (with Scenario Flags and Layered Cube Posting)

- **One workflow scenario per planning scenario, all on the cycle's one cube Scenario.** UD5 tells them apart:
  - **base** is the layer posted to (Baseline, PlanDelta, UpsideDelta);
  - **scope** is the total read back (Baseline, Plan_Total, Upside_Total).
- **Seeded compare-tos.** Each cycle is seeded with Compare To 1 = the Plan scenario and Compare To 2 = Upside, both at
  year offset 0. A compare-to column then reads that scenario's UD5 total.
- **Without layering,** use the Framework's usual one-Scenario-member-per-workflow-scenario setup, where base and scope
  are the same member.

### Extension: branch picking (with Plan Branches)

- **A compare-to gains a second axis.** Core supplies the scenario (UD5) and the year; the solution adds the branch
  (UD6). "The Plan scenario in V2 against the Upside scenario in P1" is one selection: Branch V2, plus Compare To 1 =
  Upside with Compare Branch P1.
- **Extend core's dialog; don't build a selector.** The handler (`IWorkflowContextHandler`) does four things:
  - It applies to one cycle type.
  - It adds its fields when the dialog opens, defaulting to the user's last picks while the cycle still lists them,
    otherwise Master. It places each Compare Branch between core's scenario and year.
  - It refuses a pick that is no longer listed when the dialog is saved.
  - On every render, it sets scope UD6 on each member set and adds the branch to the title and column headers (e.g.
    "Forecast 2027 Plan (P1)").
- **Grids filter themselves.** A view's run-time where clause names the token, for example
  `... in (select ... from BpBranchRevisions({WfCycleId}, {WfExtInt_BP_Branch}))`, so no render hook sets filters.
  Picking another branch changes every grid on the next refresh.
- **Drills can't use the token.** A token reads 0 on a view that is not workflow-enabled, so drills filter on the
  drilled row's own keys instead.
- The same mechanism suits any value users pick once that every page should follow: a currency, a version set, a
  region. Keep the stored value an integer id, so `{WfExtInt_<key>}` works in a where clause.

## Framework capabilities

- **Workflow context handlers:** dialog fields, saving, `WorkflowContextPopulated`, dialog size, and filtering grids on an
  extension value. See the Extension Kit's `docs/workflow-context.md`.
- **Member sets and compare-tos**, and the compare-to's workflow scenario and year: same doc, "Combinations and
  compare-tos" and "Compare-to: workflow scenario and year".
- **Workflow-enabled views and unit profile flags:** see the Claude skill's `references/workflow-enabled-views.md` and
  `references/workflow-unit-profiles-and-view-flags.md`.
- **Cycle types and workflow scenarios:** see the Claude skill's `references/core-workflow-configuration-tables.md`. Show
  compare-tos on the cycle type (`ShowCompareTo`); the branch fields need the Medium Change Workflow dialog.

## Adaptation guidance

- **Start with the core.** Add scenario layers only with [Layered Cube Posting](layered-cube-posting.md), and branch
  picking only with [Plan Branches](plan-branches.md).
- **No cube?** Compare-tos still give side-by-side columns from a relational summary.
- **If Parents enter data too,** drop the "Parents never post" rule. Decide then how a parent's own items combine with
  its children's roll-up.

## Important considerations

- **Never rebuild derived data in a render hook.** Context changes only choose what to read. See
  [Stored Plan Summary](stored-plan-summary.md).
- **Extension values:**
  - **Tokens resolve for the same cycle only.** A view that is not workflow-enabled, or a page on another cycle, reads
    0. Plan for it in drills and pop-ups.
  - **Core rebuilds the member set context before every call**, so set the scope overrides on every render, not once.
  - **Picks can go stale.** A proposal can be made private, discarded or reverted after it was picked. Validate on
    save, and treat an unknown id as Master on read.
- **Open:** the source README asks to confirm on a live run that the branch token sees the page's cycle on the
  unit-only grids, and that picking another Branch changes the grids on the next refresh.

## References

- Branch picking: `BpHandlers/BpPlanningHandler.Workflow.cs` (`BpWorkflowContextHandler`).
- Page defaults: `BeforeViewRenderHandler` in `BpHandlers/BpPlanningHandler.cs`.
- Run-time filters and the Unit column rule: `solutions/BusinessPlanning/scripts/build_structure.py`.
- Workflow scenarios and cycle compare-tos: `solutions/BusinessPlanning/410-BusinessPlanning-data.json`.
- Design: rounds 21-23 of `solutions/BusinessPlanning/010-BusinessPlanning-design.md`.
- Used by: [General Planning](../solutions/general-planning.md). Related: [Plan Branches](plan-branches.md),
  [Scenario Flags](scenario-flags.md), [Layered Cube Posting](layered-cube-posting.md).
