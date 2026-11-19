# Design Patterns

[← Back to Extending the Framework](../index.md)

Design patterns taken from working Revfore Framework solutions, for AI tools (and people) designing the next one. The
[Claude skill](../../integrations/aiModels/index.md) reads the patterns of the release it was built for.

## What is here

| Kind | What it is |
|---|---|
| **Solution Pattern** | How a whole kind of solution is put together: the business need, the main design decisions, and which Feature Patterns it combines. |
| **Feature Pattern** | One reusable capability (scenarios, revisions, layered posting...): what it is for, how a real solution built it, and what to watch for when reusing it. |

| Pattern | Kind | From |
|---|---|---|
| [General Planning](solutions/general-planning.md) | Solution | Business Planning (SaaS) |
| [Scenario Flags](features/scenario-flags.md) | Feature | Business Planning |
| [Item Revisions and Frozen Plans](features/item-revisions.md) | Feature | Business Planning |
| [Shared Drivers](features/shared-drivers.md) | Feature | Business Planning |
| [Plan Branches: Versions and Proposals](features/plan-branches.md) | Feature | Business Planning |
| [Layered Cube Posting](features/layered-cube-posting.md) | Feature | Business Planning |
| [Stored Plan Summary](features/stored-plan-summary.md) | Feature | Business Planning |

File names in a pattern refer to these places:
- **`solutions/<Solution>/...` and handler folders (`BpHandlers/...`):** the source solution's files, delivered with
  that solution.
- **`references/...`:** the Claude skill (`.claude/skills/revfore-framework/`).
- **`docs/...`:** the Extension Kit.

## Words used

- **Plan** is any set of forward-looking numbers a solution produces: a budget, a forecast, a proforma, a long-range plan.
  The patterns say "plan" for all of them. Use the customer's own word in a solution's names and screens.
- **The Plan scenario** is the source solution's middle scenario (Baseline, Plan, Upside). It is always written that way.
- **Forecast** appears only in the source's own names (`BpForecastSummary`, cycle `Forecast2026`), because that solution
  is a rolling forecast.

## How to use them

1. **Start from the business requirement**, not from a pattern. Write down what the users need first.
2. **Find the patterns that fit.** Read the Solution Pattern closest to the requirement, then only the Feature Patterns it
   points to that the requirement actually needs. Most solutions need one or two, not all of them.
3. **Read the Framework documentation and skill** each pattern refers to. They are the authority on the Framework's own
   behaviour; a pattern only describes how one solution used it.
4. **Adapt, don't copy.** Table names, item types, accounts, cube members and workflow names in a pattern are examples
   from one solution, never requirements, unless the pattern says the Framework requires them.
5. **Use the current APIs.** Check every Framework call a pattern mentions against the Extension Kit's stubs (`rfaRef/`)
   and docs before writing code. A pattern can fall behind the Framework.

Each pattern marks what it is:
- **Verified** means read in the source solution's code or data.
- **Recommended** is advice drawn from that solution's design rounds.
- **Open** has not been proven, for example not yet run on a live environment.

Treat Open items as questions to settle, not facts.

## Versions

The patterns are versioned with this documentation. Use the version that matches your Framework release: the version
selector on this site, or the repository branch of the same name (`PV921-SV200`).

A pattern is published after the solution it comes from is released, so a release's patterns can grow after it ships.
Each pattern says which solution revision it was verified against.

## Adding a pattern

- **Only extract what exists.** A Feature Pattern describes a capability a real solution implements and could reuse. It
  is not a wish list or a candidate. Don't write one for something the Framework already does on its own (CRUD, view
  security, grids, ordinary metadata). Reference the Framework docs instead.
- **Only from a released solution.** Its files must be available to the readers the pattern refers them to.
- **One page per pattern**, in `solutions/` or `features/` under this folder, with a lower-case, hyphenated name. Add it
  to the site navigation and to the table above. Use the sections of an existing pattern and leave out the ones that
  would be empty.
- **Keep it short.** Write about decisions and their reasons, not reproduced code. Name the implementing files for the
  detail.
- **Link both ways.** A Solution Pattern lists its Feature Patterns, and each Feature Pattern links back to the
  solutions that use it.
- **Say where it came from:** the solution, its revision, the files read, and the date. Keep anything client-specific,
  and any credentials or environment details, out.

## Keep them true

Patterns describe a solution at a point in time. The source solution and the Framework both move on. Before relying on
a pattern, check the claims you depend on against the current code and docs. When they disagree, the code wins: fix
the pattern.

Current patterns: extracted 2026-11-19 from Business Planning (SaaS), BP revision 52+ with the R057 handler changes.
