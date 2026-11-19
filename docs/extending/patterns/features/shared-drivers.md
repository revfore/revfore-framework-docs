# Feature Pattern: Shared Drivers

[← Back to Design Patterns](../index.md)


## Purpose

Maintain rates and assumptions once and have every item that uses them priced from them. Examples:
- unit prices and channel fees;
- hourly rates;
- work schedules;
- benefit and merit percentages.

Each number stays explainable: which driver revision priced it. Use it when many items share rates that change over
time or differ by part of the organization.

## Recommended approach (Verified in Business Planning)

- **A driver is an ordinary item of a driver item type** (CDR, SDR, WSD, WBD in the source), with its own revisions,
  terms and scenario-flagged range rows. There is no separate driver engine.
- **Applies To** on the driver's terms: an item type, and optionally a category.
- **Resolution, per item**, in this order:
  1. matching type;
  2. category equal or blank;
  3. nearest unit up the unit hierarchy;
  4. category-specific beats blank.

  A remaining tie is an **error**, never a guess. Drivers usually sit on a Parent unit (Corporate holds drivers only).
- **Resolution, per month.** A driver revision prices the months from its own `EffectivePeriod` on. Among the revisions
  of the resolved driver, the latest effective period wins, then priority, then `CommitSequence`.
- **Provenance.** Calculate stamps the driver revisions it used on the item's terms row
  (`ContractDriverWfItemRevisionId`...). They are shown and never typed. Store the **revision** id, not just the driver
  item.
- **Draft, then Make Working.**
  - A driver change starts as a private Draft; Calculate checks it without pricing anything.
  - **Make Working** puts it live and reprices the items it drives.
  - The cascade finds the items set-based, in batches of 1,000: same type and category, on the driver's unit or below,
    and resolving to this driver rather than a nearer one. It gives each a Working revision (or reuses one) and
    recalculates them.
- **Commit order.** Commit All commits drivers before the items they price.

## Framework capabilities

- **Unit hierarchy** (`WfUnitHierarchy_Relation`) for the nearest-ancestor rule. See
  the Claude skill's `references/hierarchy-tables.md`.
- **Item types and categories** (`WfItemType`, `WfItemCategory`). The driver's Applies To columns are **plain foreign
  keys**, not the core workflow item-type and category columns. As workflow columns, core's area filter hid drivers from
  their own grid (R051).
- **Custom actions:** `CalculateDraft` and `MakeWorking`. The standard Calculate needs a workflow-enabled view.

## Adaptation guidance

- Keep one driver type per family of rates whose dimensions match (applies to type and category, by unit). Don't build a
  generic key/value driver table: typed drivers keep validation and grids simple.
- If drivers vary by more than type, category and unit (by partner, by region), add that to Applies To *and* to the
  resolution order, and keep "a tie is an error".
- If drivers are few and never change mid-horizon, a plain settings table read by the calculation is enough. Skip
  revisions.
- A driver can have no posting measures at all (the source's never post).

## Important considerations

- **The cascade is the largest source of volume.** One driver change produces new revisions and results for every item
  it prices. Batch it, and keep it all or nothing, so reports never show a half-repriced plan.
- **Inside a [plan branch](plan-branches.md), drivers resolve against the branch's snapshot:** master at its boundary, overlaid by the branch's
  own driver revisions. A driver changed in a proposal reprices only that proposal's items.
- **Percent storage needs one convention.** The source stores percent columns as fractions (0.05) but an escalator
  "Percent" as a plain number (3.5). Pick one and write it down in the solution guide for agents.

## References

- Resolution and cascade: regions "Driver resolution" and "Driver cascade" in
  `BpHandlers/BpPlanningHandler.Calculation.cs`.
- Design: section 6 ("Driver resolution") of `solutions/BusinessPlanning/010-BusinessPlanning-design.md`.
- Used by: [General Planning](../solutions/general-planning.md). Related: [Item Revisions](item-revisions.md).
