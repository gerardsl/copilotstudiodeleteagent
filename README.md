# Find and delete draft Copilot Studio agents with the Power Platform CLI

Practical runbook for listing every **draft** (never-published) Copilot Studio agent in a tenant and deleting them, using `pac` (Power Platform CLI) against Dataverse.

> "Draft" here means the agent record in the Dataverse `bot` table has **no `publishedon` value** — it was created but never published.

---

## 1. Permissions required

### Identity / tenant roles

| Purpose | Role needed | Where |
|---|---|---|
| Sign in with `pac auth create` | Any licensed user in the tenant | Microsoft Entra ID |
| List **all** environments in the tenant (`pac env list`) | **Power Platform Administrator** (or Dynamics 365 Administrator / Global Administrator) | Microsoft Entra ID |
| List only environments you personally have access to | No admin role — Dataverse access is sufficient | — |

### Dataverse (per-environment) privileges

`pac copilot list`, `pac env fetch` and `pac copilot delete` operate directly on the Dataverse `bot` table, so the signed-in user needs a security role in **each target environment**:

| Operation | Minimum privilege on the `bot` table | Typical role |
|---|---|---|
| `pac env fetch` / `pac copilot list` | **Read** (Organization scope, to see other users' agents) | System Customizer, Environment Maker + role with org-level read |
| `pac copilot delete` | **Delete** (Organization scope) | **System Administrator** |

Practical guidance:

- **System Administrator** in the target environment covers everything in this runbook.
- Organization-level scope matters: with only User-level read you will see *your own* agents, not the whole tenant's.
- Agents that belong to a **managed solution** cannot be deleted this way — remove/uninstall the solution instead.
- Deletion is **permanent**. There is no recycle bin for the `bot` record.

### Licenses

- No Copilot Studio license is required to *delete* an agent, but the environment must have Dataverse provisioned.

---

## 2. Binaries and CLIs to install

| Component | Version used | Required? | Notes |
|---|---|---|---|
| **Power Platform CLI (`pac`)** | 2.11.1 | **Yes** | The Windows build is a .NET Framework 4.8 app — no extra runtime needed on Windows |
| Windows PowerShell | 5.1 (built in) | Yes | Any PowerShell 5.1+ / 7.x works |
| .NET SDK 10 | 10.0.x | **Yes** | `pac` **2.x targets .NET 10**; installing it with SDK 8 fails with `DotnetToolSettings.xml was not found in the package` |

### Install as a dotnet global tool (needs .NET SDK 10)

```powershell
dotnet tool install --global Microsoft.PowerApps.CLI.Tool
# update later:
dotnet tool update --global Microsoft.PowerApps.CLI.Tool
```

---

## 3. Authenticate and pick the environment

```powershell
# interactive browser sign-in (creates a named auth profile)
pac auth create --name MyTenant

# manage / switch profiles
pac auth list
pac auth select --index 1
pac auth who

# list environments (needs Power Platform Administrator to see them all)
pac env list
```

Note `pac auth create --username` requires `--password` as well, so for interactive MFA sign-in omit both and use the browser dialog (or use `--deviceCode`).

---

## 4. Command to get all draft agents in the tenant

`pac copilot list` does **not** expose publish state (its "Component State" column is the Dataverse *solution* component state), so query the `bot` table with FetchXML instead.

`draft-agents.xml`:

```xml
<fetch>
  <entity name="bot">
    <attribute name="name" />
    <attribute name="botid" />
    <attribute name="schemaname" />
    <attribute name="publishedon" />
    <attribute name="createdon" />
    <attribute name="statecode" />
    <filter>
      <condition attribute="publishedon" operator="null" />
    </filter>
    <order attribute="createdon" />
  </entity>
</fetch>
```

Run it against one environment:

```powershell
pac env fetch --environment <environmentId> --xmlFile draft-agents.xml
```

Run it across **every environment in the tenant**:

```powershell
$envs = pac env list | Select-String -Pattern '([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})' |
        ForEach-Object { $_.Matches[0].Value }

foreach ($e in $envs) {
    "=== $e ==="
    pac env fetch --environment $e --xmlFile draft-agents.xml
}
```

Gotchas (both produce hard errors):

- Use `--xmlFile`. Passing FetchXML inline with `--xml` throws `System.Xml.XmlException` and terminates `pac`.
- Do **not** add a `top="..."` attribute: `The top attribute can't be specified with paging attribute page`.

---

## 5. Command to delete the draft agents

Single agent:

```powershell
pac copilot delete --environment <environmentId> --bot <copilotId> --confirm
```

- `--bot` (alias `-id`) takes the Copilot ID **or** the schema name.
- `--confirm` (alias `-y`) skips the interactive prompt.
- `--environment` can be omitted to use the active organization of the current auth profile.

Bulk delete every draft agent in an environment:

```powershell
$envId = '<environmentId>'

$out = pac env fetch --environment $envId --xmlFile draft-agents.xml
$ids = [regex]::Matches(($out -join "`n"),
        '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}') |
        ForEach-Object { $_.Value } |
        Where-Object { $_ -ne $envId } |
        Select-Object -Unique

# review first — remove the echo and uncomment the delete once the list looks right
foreach ($id in $ids) {
    "WOULD DELETE: $id"
    # pac copilot delete --environment $envId --bot $id --confirm
}
```

Verify afterwards:

```powershell
pac copilot list --environment <environmentId>
```

---

## 6. Quick reference

```powershell
pac help                                                        # command list
pac auth create --name MyTenant                                 # interactive sign-in
pac auth list / pac auth select --index N / pac auth who        # profile management
pac env list                                                    # environments (= pac org list)
pac copilot list --environment <envId>                          # all agents (no publish state)
pac env fetch --environment <envId> --xmlFile draft-agents.xml  # draft agents only
pac copilot delete --environment <envId> --bot <id> --confirm   # permanent delete
```
