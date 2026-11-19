# Feature Pattern: Plan Branches - Versions and Proposals

[← Back to Design Patterns](../index.md)


## Purpose

Work on an alternative plan without touching the master plan.

A **plan branch** is an alternative set of plan data (item revisions) over the master plan. It is not a
source-control branch, a workflow route or a scenario: a plan branch holds its own Baseline, Plan and Upside, and
the solution's code calls it `WfItemBranch`. There are two kinds:
- **Version:** a numbered, stacking correction of a plan, typically after a unit has closed.
- **Proposal:** a what-if anyone can explore privately or share, then promote into master or discard.

Use it when people need to change closed numbers in a controlled way, or to model options side by side.

## Recommended approach (Verified in Business Planning)

- **`WfItemBranch`** has a type (Version or Proposal), status, number, cycle, private flag and UD6 member.
  **`WfItemBranch_Unit`** records each unit it covers, with the `BaseCommitBoundary` and base version it starts from.
- **Copy-on-write.** A branch copies nothing when it opens. Its revisions are ordinary `WfItem_Revision` rows carrying
  `WfItemBranchId`. An edit made while the branch is picked is redirected into the branch, as committed edits are (see
  [Item Revisions](item-revisions.md)).
- **What a branch shows** is one SQL definition, `BpBranchRevisions(@Cycle, @Branch)`:
  - master as of the branch's snapshot (each unit's base boundary), overlaid by the branch's own revisions;
  - for branch 0, master's current revisions.
- **Versions:**
  - A version covers a unit and its descendants, open or closed.
  - Each stacks on the last locked one: V2 on V1.
  - A version can be opened only when everything it stacks on is committed. The refusal names each branch, unit and
    count.
  - **Lock & Post** commits its Working revisions, freezes its set and posts. **Revert** undoes the latest. A locked
    version is never reopened.
- **Proposals:**
  - Proposals stack on master alone: master's commits at the snapshot, plus master's live Working revisions.
  - Each is private or public.
  - **Promote To Master** copies changes into master Working revisions (drivers: Drafts). It skips items master changed
    since the snapshot unless confirmed.
  - **Discard** clears the proposal's cube layer and frees it.
- **Picking a branch.** Branch and Compare Branch 1 / 2 are fields added to the Framework's Change Workflow dialog.
  Grids follow the pick through a run-time filter on `{WfExtInt_BP_Branch}` that calls `BpBranchRevisions`. See
  [Workflow Context](workflow-context.md).
- **A fixed UD6 member pool** (`BpBranchMember`: V1-V3 and P1-P3, each with a `_Total` parent) is handed out in order.
  Opening a branch is refused when the pool is used up. See [Layered Cube Posting](layered-cube-posting.md).

## Framework capabilities

- **Workflow context handlers:** extra fields in Change Workflow, kept per user. `{WfExt_<key>}` and `{WfExtInt_<key>}`
  tokens resolve in run-time where clauses. See the Extension Kit's `docs/workflow-context.md`, "Filtering
  grids on an extension value".
- **Member set scope per compare-to:** see the same doc, "Combinations and compare-tos".
- **View security:** Versions and Proposals are open to every planner.
- **Private proposals:** hidden by a run-time filter on the current user.

## Adaptation guidance

- **Need only one kind?** Proposals alone suit what-if modeling. Versions alone suit post-close corrections. Both rely on
  [Item Revisions](item-revisions.md).
- **Pool size** is a cube decision. Members are never reused for versions, so plan for how many corrections a cycle may
  need.
- **AI agents.** The v0.2 note's AI-assisted modeling (Design, not built) puts agents in proposals: agents write only
  inside their own proposals and never promote. That is a natural extension; add the gates on top of the same save path,
  never as a separate path.
- **No cube?** Branches still work relationally. Drop the UD6 pool and posting.

## Important considerations

- **Tokens and drills.** A token reads 0 on a view that is not workflow-enabled (the Framework resolves it for the same
  cycle only), so the drill views filter by the drilled row's branch instead.
- **Delete hooks get no parameter set.** Clearing a posted branch on delete loads the posting view at core's dialog set
  7003. If any clear fails, the branch is kept (`PostedAt` stays), so a layer is never left behind under a member that
  will be reused.
- **Stale layers.** Reposting master marks posted proposals, and versions stacked on it, **Post Out Of Date**.
- **Removing a master item inside a proposal** has no marker yet. It is an open item in the v0.2 note.
- **Open:** the README's live-run checks cover Clear Cube on delete and the branch filters seeing the page's cycle.

## References

- Branch actions: regions "Branches", "Versions" and "Proposals" in
  `BpHandlers/BpPlanningHandler.Cycles.cs`.
- Change Workflow fields: `BpHandlers/BpPlanningHandler.Workflow.cs`.
- `BpBranchRevisions`: `solutions/BusinessPlanning/510-BusinessPlanning-view-WfItemActivityMonth.sql`.
- Design: rounds 20, 22, 23 and 24 of `solutions/BusinessPlanning/010-BusinessPlanning-design.md`.
- Used by: [General Planning](../solutions/general-planning.md).
