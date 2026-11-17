---
name: fabric-workspace
description: 'Connect to Microsoft Fabric using Fabric CLI. Use when asked to connect to, select, or switch a Fabric workspace by name or ID, list its items, or create notebooks and other Fabric items in a specified or previously selected workspace.'
argument-hint: '<workspace name or ID> [optional item action]'
---

# Fabric Workspace

Verify Fabric management API access and select a workspace for this conversation.
This does not establish a SQL connection, start compute, or attach a data source.
Connecting alone is read-only. Perform an item action only when the user requests it.

Examples: "Connect to Fabric workspace Sales Analytics"; then "Create a notebook
named Daily_Checks in that workspace." `/fabric-workspace` accepts the same inputs.

## 1. Prepare the CLI

Prefer an existing `fab` executable and check `fab --version`.
If unavailable, locate `uv` using `command -v uv`, then `$HOME/.local/bin/uv`.
If neither executable exists, explain the prerequisite and ask before installing.
Fabric CLI requires Python 3.10 or later; do not modify an older system Python.

On macOS/Linux, set `fabric_uv` to the verified executable path and use this
session-only fallback. It may download isolated Python and CLI packages:

```sh
export UV_CACHE_DIR="${TMPDIR:-/tmp}/fabric-cli-uv-cache"
export UV_PYTHON_INSTALL_DIR="${TMPDIR:-/tmp}/fabric-cli-python"
export UV_TOOL_DIR="${TMPDIR:-/tmp}/fabric-cli-tools"
fab() {
  "$fabric_uv" tool run --python 3.12 --from ms-fabric-cli fab "$@"
}
fab --version
```

Reuse the same runner and directories for subsequent commands. Do not change
Fabric's authentication/configuration directory or global output settings.
For another shell, adapt the invocation rather than executing POSIX syntax.

## 2. Reuse Authentication

Run `fab auth status`. Reuse a valid login; do not log out or switch accounts.
If unauthenticated, run `fab auth login` and let the user complete browser login.
Use `--tenant <tenant-id>` only when the user specifies the intended tenant.
If the current tenant differs, confirm before changing the authentication context.
Never request secrets in chat, display raw tokens, or read/copy token caches.
Respect terminal approval and sandbox boundaries; do not bypass authentication.

## 3. Resolve and Verify the Workspace

1. Use the explicitly requested workspace, otherwise the selected workspace from
   this conversation. If neither exists, ask for a workspace name or ID.
2. For a name, run `fab api workspaces` and match the full `displayName`, not a
   substring or the first result. `fab ls / -l` provides a readable overview,
   but its table may wrap names and IDs; do not parse it as structured data.
   Accept the CLI's `.Workspace` suffix without appending it twice.
   Retain the matching workspace ID. If missing, stop; never create a workspace.
   If ambiguous, ask for its ID rather than guessing.
3. For an ID, retrieve its actual display name with the first command below.
   Verify item access with the second command for either input form:

   ```sh
   fab api "workspaces/<workspace-id>"
   fab api "workspaces/<workspace-id>/items"
   ```

4. Follow any continuation token/URI when listing workspaces or items. Use
   ID-scoped API requests if CLI listings are incomplete or names are ambiguous.
   Prefer unfiltered responses; use `--output_format json` when structured output
   is needed and supported by the installed version. Parse JSON structurally,
   check its reported status and HTTP status where present, and check exit codes.
5. Remember the runner, workspace name, ID, and canonical
   `/<workspace-name>.Workspace` path in conversation context, not a state file.
   Report the verified name, ID, workspace link, and a short item summary:
   `https://app.fabric.microsoft.com/groups/<workspace-id>/list`.
   Stop here unless an action was requested. Do not claim write access from a read.

## 4. Handle Follow-up Item Requests

Reuse the selected workspace unless the user names another; resolve and verify
the new target first. Do not rely on `fab cd` persisting across separate commands.
Quote paths and treat user names as literal data, never shell code or expressions.
Reject path separators/traversal in an item name, or use an ID-scoped JSON API.

For a requested blank notebook:

1. Obtain its intended name and list items in the resolved workspace. If the same
   name/type exists, report it as existing; do not overwrite or claim creation.
2. For an unambiguous workspace path, run:

   ```sh
   fab mkdir "/<workspace-name>.Workspace/<notebook-name>.Notebook"
   ```

   If the workspace name is ambiguous or cannot safely be addressed as a path,
   use `fab api "workspaces/<workspace-id>/items" -X post` with a JSON body
   containing `displayName` and `type: "Notebook"`. Build/escape JSON with a
   structured serializer, not shell interpolation. Consult `fab api --help` for
   input syntax; verify supported parameters with `fab mkdir --help` as needed.
3. Read back metadata with
   `fab api "workspaces/<workspace-id>/items/<item-id>"` using the returned ID.
   If creation is asynchronous, follow the operation status until it completes
   before claiming success. Do not fetch notebook definitions just to verify
   existence or automatically accept sensitivity-label prompts.
4. Report the actual item name, type, ID, and notebook link:
   `https://app.fabric.microsoft.com/groups/<workspace-id>/synapsenotebooks/<item-id>`.

For other Fabric items, consult `fab mkdir --help` and the official
[creation reference](https://microsoft.github.io/fabric-cli/commands/fs/mkdir/).
Obtain required parameters/definitions and use the same check/create/read-back
pattern. Not every item supports empty creation. Creating a notebook does not
authorize executing it, adding cells, or attaching a lakehouse or warehouse.

## Failure Boundaries

- For authentication failures, refresh login as needed. For permission failures,
  explain the missing access; do not change roles, tenants, or capacity to fix it.
- For syntax/output failures, inspect installed command help and retry a read
  without query/format flags before diagnosing authentication or permissions.
- After a timeout or uncertain create result, check for the item before retrying;
  never blindly repeat a write or claim success without read-back verification.
- Do not delete, overwrite, use force flags, run notebooks, provision capacity,
  or change permissions as a side effect of connecting or creating an empty item.
- Selection is conversation-local. In a new conversation, resolve the workspace
  again; do not promise a persistent connection or cross-chat default.