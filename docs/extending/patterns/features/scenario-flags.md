# Feature Pattern: Scenario Flags

[← Back to Design Patterns](../index.md)


## Purpose

Model several scenarios (Baseline, Plan, Upside...) on one item without copying the item per scenario. Values that agree
across scenarios are entered and stored once, and only the differences are extra rows. Use it whenever a solution has
more than one scenario and most assumptions are shared.

## Recommended approach (Verified in Business Planning)

- **One solution-wide scenario list** (`BpScenario`). Each row has:
  - a `BitValue` (Baseline 1, Plan 2, Upside 4);
  - an `IsActive` flag;
  - **Layers On** (`BaseBpScenarioId`: Plan on Baseline, Upside on Plan);
  - its UD5 member, used by [Layered Cube Posting](layered-cube-posting.md).

  Domains never define their own scenarios.
- **`ScenarioFlags` on the revision header** says which scenarios the item exists in.
- **`ScenarioFlags` on every input range row** (`ScenarioFlags`, `StartPeriod`, `EndPeriod`, value):
  - a row flagged 7 applies to all three scenarios;
  - to make Upside differ, split it into a row flagged 3 (Baseline + Plan) and one flagged 4 (Upside).

  The rule is that ranges never overlap for one scenario and month, and the handler checks it on save.
- **Planners enter full values in every scenario**, never deltas. Deltas exist only at posting.
- **Calculation loops over the active bits** the revision has. For each bit and month it picks the one range row whose
  flags contain the bit. Driver rows are flagged the same way.
- **Results are compressed.** Adjacent equal months become one range, and scenarios with identical results share one
  `WfItem_Activity_Period` row whose `ScenarioFlags` holds all of their bits.
- **Comparison in the UI** uses the Framework's workflow scenarios and compare-tos:
  - three workflow scenarios per cycle year, all on the cycle's one cube Scenario, told apart by a UD5 member set;
  - each cycle plans Baseline and sets Compare To 1 = Plan and Compare To 2 = Upside.

## Framework capabilities

- **Typed columns:** `ScenarioFlags` is a typed integer column.
- **Workflow scenarios, member sets (base / scope) and compare-tos:** see the Extension Kit's `docs/workflow-context.md`,
  "Combinations and compare-tos" and "Compare-to".
- **Period columns:** see "Period columns" in the Claude skill's `references/design-patterns.md`.
  The source uses `YYYYMM` with `299999` as open-ended.

## Adaptation guidance

- **Bit width.** An `int` holds 31 scenarios. Assign bits centrally and never reuse a retired bit, because history keeps
  its meaning.
- **Few differences.** If scenarios rarely differ, this is the cheapest storage.
- **Different structure.** If a scenario has a different structure (other items, other accounts), flags on the revision
  header handle "item exists only in Upside". If whole domains differ, consider separate items instead.
- **Inheritance.** Scenarios here do not inherit from each other in the data: every scenario's value is explicit on a
  range row. "Layers On" is used only for posting deltas. If users want "the Plan scenario = Baseline unless overridden", add that
  rule to the save. The pattern itself does not have it.
- **Full values or deltas in the cube.** Keep full values in the relational data either way. Whether the cube gets full
  values or deltas is a separate choice (see [Layered Cube Posting](layered-cube-posting.md)).

## Important considerations

- Grids must make the flags readable (a typed lookup or checkboxes). Raw numbers confuse planners.
- Splitting a range to change one scenario is the main training point. A tool or action that does the split helps.
- Overlap validation has to run on every write path (grid, import, MCP): put it in the handler, not the grid.
- The original v0.2 pattern note left "scenario precedence" open. The built rule is simple (no overlaps, explicit values),
  and that simplicity is why it works.

## References

- Seed data: `BpScenario` rows in `solutions/BusinessPlanning/410-BusinessPlanning-data.json`.
- Calculation: `Recalculate` and `DriverSet.Row` in `BpHandlers/BpPlanningHandler.Calculation.cs`.
- Design: rounds 20-21 in `solutions/BusinessPlanning/010-BusinessPlanning-design.md`.
- Used by: [General Planning](../solutions/general-planning.md). Related: [Layered Cube Posting](layered-cube-posting.md).
