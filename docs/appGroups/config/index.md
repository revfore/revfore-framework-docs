# Config

The **Config** application group is used for **non-daily administrative and system-level tasks** in Revfore Framework.

Where the **Admin** application group is focused on routine daily administration, the **Config** application group is intended for setup, upgrades, licensing, maintenance, and other less frequent actions that help install, maintain, or troubleshoot the solution.

## Config Sections

The Config application group includes the following sections:

- [Manage License Key](manage-license-key.md)
- [Setup & Upgrade](setup-upgrade.md)
- [Create New Instance](create-new-instance.md)
- [Uninstall](uninstall.md)
- [Run Maintenance Scripts](run-maintenance-scripts.md)
- [Other](other.md)

## Who can use Config

Each page checks who you are when it opens, and refuses anyone without the right:

| Page | Who |
|---|---|
| [Manage License Key](manage-license-key.md), [Other](other.md) | A **Revfore Administrator** (the `RevforeAdmin` [security role](../admin/security/Roles/index.md)) |
| [Setup & Upgrade](setup-upgrade.md), [Create New Instance](create-new-instance.md), [Uninstall](uninstall.md), [Run Maintenance Scripts](run-maintenance-scripts.md) | A member of OneStream's **Administrators** group, not viewing the application as another user |

OneStream administrators hold every Revfore role, so they can use every page - including on a new install, before Setup
has created any of Revfore's tables.

## When to Use Config

Use the Config application group when you need to:

- set up Revfore Framework for the first time
- upgrade to a newer version
- manage Starter or Full license keys
- uninstall some or all of the solution
- run maintenance or support-provided scripts
- perform occasional system-wide synchronization or relinking tasks

## Related Application Groups

For routine ongoing administration, go to the **Admin** application group, where users can review and manage existing Relational Tables, Models, and Views.