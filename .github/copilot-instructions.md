# Copilot instructions

## Repository purpose and structure

This repository is an operational runbook for finding and deleting never-published Copilot Studio agents from Dataverse with the Power Platform CLI (`pac`).

- `README.md` is the source of truth for prerequisites, permissions, installation, authentication, discovery, deletion, and verification.
- `draft-agents.xml` is the FetchXML query consumed directly by `pac env fetch`. Keep the XML example embedded in `README.md` synchronized with this file.

There is no application build, automated test suite, or lint configuration in this repository. Validate changes by running the documented `pac` commands against an authorized non-production environment.

## Operational commands

```powershell
pac auth who
pac env fetch --environment <environmentId> --xmlFile draft-agents.xml
pac copilot list --environment <environmentId>
```

The `pac env fetch` command is the focused validation for `draft-agents.xml`. Use `pac copilot list` after a deletion to verify the resulting inventory, but do not treat its `Component State` column as publish state.

## Domain conventions and safety constraints

- In this repository, a **draft** agent specifically means a Dataverse `bot` record whose `publishedon` attribute is null. Preserve the `<condition attribute="publishedon" operator="null" />` filter unless intentionally changing that definition.
- Query the `bot` table through the checked-in XML file using `--xmlFile`. Inline `--xml` input is documented to fail, and a FetchXML `top` attribute conflicts with paging added by `pac`.
- `pac copilot delete` accepts the Copilot ID or schema name through `--bot` (alias `-id`). Deletion is permanent and requires appropriate organization-scope Dataverse privileges.
- Keep bulk-deletion examples review-first: print `WOULD DELETE` entries and leave the actual delete command commented until an operator explicitly enables it.
- Tenant-wide enumeration and deletion are environment-scoped. `pac env list` supplies environment IDs, while `pac env fetch` and `pac copilot delete` must run separately for each environment.
- The bulk example extracts GUIDs from formatted CLI output and explicitly removes the environment ID before deduplication. Preserve that exclusion when changing output parsing.
- Use PowerShell syntax in command examples and retain placeholders such as `<environmentId>` and `<copilotId>` for tenant-specific values. Never add tenant IDs, credentials, or real agent IDs to the repository.
