# Feature Pattern: Item Revisions and Frozen Plans

[← Back to Design Patterns](../index.md)


## Purpose

Let planners keep changing items while every closed plan stays exactly reproducible, without copying the item
population per cycle. "What did US's 2027 plan say when it closed?" is answered from the same rows as today's
plan. Use it when closed or submitted numbers must never move, or when the population is large and cycles are many.

This is the source's **snapshot** capability. A unit's plan is frozen by a boundary, not by a copy. (The scenario
named "Baseline" is unrelated.)

## Recommended approach (Verified in Business Planning)

- **Identity, then revisions.**
  - `WfItem` holds the permanent identity: `MasterNumber`, and `Number` with the item type's prefix.
  - `WfItem` also carries current fields, which are only a projection of the latest commit.
  - `WfItem_Revision` is a package from its `EffectivePeriod` forward: header, terms, ranges and its own calculated
    results.
- **Statuses.**
  - Items go Working → Committed.
  - Drivers go Draft (private, never priced) → Working → Committed.
  - There is at most one uncommitted revision per item per plan branch.
- **Committed never changes.** BeforeSave redirects an edit of a committed row into the item's Working revision, which
  it creates by copying the committed package.
  - The redirects of one save are gathered, checked together and made together in one transaction: all or nothing.
  - Uncommitted revisions are edited in place, so a busy plan does not pile up revisions.
- **Commit allocates a global `CommitSequence`.** That sequence is the visibility order; a revision id is not.
  - Commit All commits drivers before the items they price, all or nothing.
- **Effective period is derived.** It is the earliest month an edit touches; a terms change takes the cycle's first open
  month. The planner can override it.
- **Items carry no cycle.** For a cycle and unit, the *cycle revision rows* (`WfCycleUnitItemRevision`) are worked out by SQL. They
  hold each item's revision in force and the months it is in force, ordered by `EffectivePeriod`, then priority, then
  `CommitSequence`. Cycle revision rows come in sets:
  - **Latest:** an open unit's master Committed revisions, overlaid by Working.
  - **Cycle (frozen):** for a closed unit, master Committed at Close Unit, or a locked version's set.
  - **Proposal:** one per proposal.
- **Close Unit** commits the unit's Working revisions, freezes `CommitBoundary` on `WfCycleUnitBoundary`, writes the
  frozen set once and posts. Frozen sets are never worked out again, so later commits never reach a closed cycle.
  Reopen releases it.
- **Delete rules:** a committed revision can be deleted only if it is the item's latest, no closed boundary covers it,
  and nothing references it.

## Framework capabilities

- **Workflow units and cycles, views workflow-enabled by unit only.** Item grids cross cycles. See
  the Claude skill's `references/workflow-enabled-views.md`.
- **Save hooks.** BeforeSave and AfterSave, bulk update, delete. See the Extension Kit's `docs/handlers.md` and
  the Extension Kit's `docs/records.md`.
- **Custom actions:** Revise, Commit, Commit All, Close and Reopen Unit, Delete All. See the Extension Kit's `docs/actions.md`.
- **Numbering.** `WfItemType.NumberPrefix` and a database sequence for `MasterNumber`. See
  "Sequential auto-numbering" in the Claude skill's `references/design-patterns.md`.
- **Hand-written SQL in the setup package.** The cycle-revision procedures are in `510`.

## Adaptation guidance

- **Drop Draft** if there are no shared drivers.
- **No unit closing at all.** If nothing ever needs freezing, plain editable rows are enough. Don't adopt revisions for
  audit alone: the Framework's audit columns and change history may suffice.
- **Approval.** Close Unit here has no approval step. A workflow status (submit and approve) can drive the same freeze.
- **Monthly granularity.** Keep revisions whole-package and forward from the effective month. Partial-period revisions
  (weeks) need an explicit calendar first.
- **One shared Pending Changes grid** lists every uncommitted revision across types and branches, with Commit All and
  Delete All. It is the planner's control point; keep it.

## Important considerations

- **Results are per revision**, so a driver change that reprices thousands of items creates thousands of revisions. That
  growth is real. The v0.2 note proposes archiving old results (Design, not built).
- **Every write path must redirect.** Grid, import and MCP must all redirect: the handler is mandatory.
- **Use `CommitSequence` for visibility, not identity values.** Ids are allocated before commit and do not give a
  consistent visibility order under concurrency.
- **A unit closed before cycle revision rows existed** is frozen from its `CommitBoundary` on Initialize Plan. That migration case
  is handled in `BpPlanningHandler.Cycles.cs`.
- **Open:** the source README asks to confirm on a live run that the grid accepts the committed row set to Ignore when an
  edit is redirected.

## References

- Lifecycle: `BpHandlers/BpPlanningHandler.Revisions.cs` (redirects, commit, delete rules).
- Boundaries and cycle revision rows: `BpHandlers/BpPlanningHandler.Cycles.cs`, and
  `solutions/BusinessPlanning/510-BusinessPlanning-view-WfItemActivityMonth.sql`
  (`BpResolveCycleRevisions`, `BpFreezeCycleRevisions`; its header explains the sets).
- Design: section 6 and round 24 of `solutions/BusinessPlanning/010-BusinessPlanning-design.md`.
- Used by: [General Planning](../solutions/general-planning.md). Related: [Plan Branches](plan-branches.md), [Shared Drivers](shared-drivers.md).
