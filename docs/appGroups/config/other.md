# Other

[← Back to Config Summary](index.md)

The **Other** section includes additional system-level actions that administrators can run as needed.

The page is for a **Revfore Administrator** (the `RevforeAdmin` [security role](../admin/security/Roles/index.md)); anyone
else is refused when they open it.

These actions are generally safe to run and are helpful for synchronizing metadata or relinking records after installs or upgrades.

## Available Actions

### Sync All Managed Relational Views

This action loops through all managed relational views and syncs their definitions with the actual SQL views in the database.

#### When to Use

This is helpful when:

- you have made changes to relational view definitions
- you want to ensure those changes have been applied to the database
- you want to confirm that all managed SQL views are in sync

#### Notes

- This action is safe to run anytime.
- The system automatically runs this during setup and after any upgrade.

---

### Relink Forms and Users

Forms and Users are linked using both:

- their hidden unique IDs
- their names

This process takes the names and updates the unique IDs.

Revfore Framework normally uses the unique IDs, so these records need to be relinked after an install or upgrade.

#### When to Use

Use this action when:

- forms or users need to be relinked after an install
- forms or users need to be relinked after an upgrade
- the automatic relinking process did not complete as expected

#### Notes

- This process should automatically run, but it can also be run manually here if needed.
- It is safe to run anytime.
- Revfore Framework automatically creates user records when it encounters a new user login.

---

### Decrypt Error Log

Shows the parameter values in Debug log entries. When a user's **Debug** is set to a **values** option (see
[Parameter values](../admin/security/Users/index.md#parameter-values)), each statement's values are written to the
error log encrypted, as `values=enc:...`. This form turns them back into readable values.

#### How to Use

1. Go to **System | Logging | Error Log**.
2. Select an error log row and copy it.
3. In **Config | Other**, click **Decrypt Error Log**.
4. Paste the entry into the input box. You can paste several entries at once.
5. Click **Decrypt**.

The output shows the same text with each encrypted section replaced in place by that statement's values, for example
`values=@Lk0='Acme Corp%' @Lk1=42`, so they read in the context of their line. Paste the next entry and click
**Decrypt** again to work through the log one entry at a time.

#### Notes

- Only a **Revfore Administrator** can open the form or decrypt. An administrator who is viewing the application as
  another user cannot, until they stop.
- Every decrypt is recorded in the error log: who decrypted, and how many sections. The values themselves are never
  recorded.
- Values longer than 200 characters were cut when they were logged; the output shows their full length.
- `values=redacted` stays redacted: statements on settings and secrets are never logged with values, so there is
  nothing to decrypt.
- A section shown as *cannot decrypt* was changed after it was written, was cut short when copied, or was written by
  another application. The rest of the text still decrypts.
- Decrypted values are business data. Share only what is needed, and don't paste them into tickets or chats that the
  original log entry was safe to share in.
- There is no other way to decrypt the values - no API, report or command line. This keeps them inside OneStream.
