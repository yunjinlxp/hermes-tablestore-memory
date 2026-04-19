# Hermes TableStore Memory Provider

TableStore-based external memory provider for Hermes Agent.

This plugin uses the official `tablestore` Python SDK and Hermes' memory
provider interface to add:

- semantic long-term memory backed by TableStore memory APIs
- automatic turn sync after each completed conversation turn
- prefetch of relevant memories before the next turn
- explicit memory tools for inspect/search/store/delete operations

The built-in `MEMORY.md` and `USER.md` stores remain active. This plugin adds
one external memory backend alongside them.

## Features

- `tablestore_profile`
  Lists memories in the current scope.
- `tablestore_search`
  Runs semantic search over stored memories.
- `tablestore_remember`
  Persists a fact or short note to long-term memory.
- `tablestore_forget`
  Deletes a memory by id.
- Automatic `sync_turn()`
  Sends completed user/assistant turns to TableStore memory ingestion.
- Automatic `queue_prefetch()`
  Retrieves likely relevant memories for the next turn.
- Mirrors Hermes built-in memory writes into TableStore.

## Requirements

- Hermes Agent installed
- Python environment used by Hermes can install `tablestore==6.4.5`
- A reachable TableStore OTS endpoint with memory APIs enabled
- A TableStore access key pair with permission to access the target instance
- An existing memory store name, or permission to create one

## Install

### Recommended: install from GitHub

```bash
hermes plugins install yourname/hermes-tablestore-memory
```

Or:

```bash
hermes plugins install https://github.com/yourname/hermes-tablestore-memory.git
```

Hermes installs the plugin into:

```text
~/.hermes/plugins/tablestore-mem/
```

### Manual install

Copy this repository into:

```text
~/.hermes/plugins/tablestore-mem/
```

Then install the SDK dependency into the Python environment used by Hermes:

```bash
uv pip install --python "$(which python)" tablestore==6.4.5
```

If Hermes runs from a venv, use that interpreter explicitly instead.

## Activate

Interactive setup:

```bash
hermes memory setup
```

Select `tablestore-mem` from the provider list.

Or activate manually:

```bash
hermes config set memory.provider tablestore-mem
```

## Configuration

Secrets should go into:

```text
~/.hermes/.env
```

Non-secret provider settings can live in:

```text
$HERMES_HOME/tablestore_memory.json
```

### Required environment variables

```bash
TABLESTORE_MEMORY_AK=your_access_key_id
TABLESTORE_MEMORY_SK=your_access_key_secret
TABLESTORE_MEMORY_ENDPOINT=https://your-instance.region.ots.aliyuncs.com
TABLESTORE_MEMORY_INSTANCE=your-instance-name
TABLESTORE_MEMORY_STORE=hermes_mem
```

### Optional environment variables

```bash
TABLESTORE_MEMORY_APP_ID=hermes
TABLESTORE_MEMORY_TENANT_ID=
TABLESTORE_MEMORY_DESCRIPTION=
TABLESTORE_MEMORY_ENABLE_RERANK=true
TABLESTORE_MEMORY_AUTO_CREATE_STORE=true
TABLESTORE_MEMORY_TIMEOUT=30
```

### Example `tablestore_memory.json`

```json
{
  "endpoint": "https://your-instance.region.ots.aliyuncs.com",
  "instance_name": "your-instance-name",
  "memory_store_name": "hermes_mem",
  "app_id": "hermes",
  "enable_rerank": true,
  "auto_create_store": true,
  "timeout": 30.0
}
```

If the user does not specify a memory store name, the plugin defaults to
`hermes_mem` and automatically creates it when missing.

## How scope works

The provider stores and retrieves memories under a TableStore memory scope:

```text
appId / tenantId / agentId / runId
```

Current field resolution:

- `appId`
  Source: user config if `TABLESTORE_MEMORY_APP_ID` or `app_id` is set.
  Default: `hermes`.
- `tenantId`
  Source: Hermes session `user_id` first.
  Fallback: user config `TABLESTORE_MEMORY_TENANT_ID` or `tenant_id`.
  Default: `__default__`.
- `agentId`
  Source: Hermes session identity, currently `agent_identity`.
  Default: `hermes`.
- `runId`
  Source priority:
  1. `gateway_session_key`
  2. `session_title`
  3. current `session_id`
  Default: `__default__` only if all of the above are empty.

This means only `appId` and `tenantId` are user-facing configuration inputs.
`agentId` and `runId` are intentionally session-derived so users do not need
to manage them manually.

Write scope and search scope are intentionally different:

- writes use the current session scope exactly
- searches use the current `tenantId` with `agentId=*` and `runId=*`

This lets Hermes search across all agents and sessions for the same tenant
while still writing memories with precise session attribution.

## Verify the installation

Check that Hermes sees the provider:

```bash
hermes memory status
```

You should see:

```text
Provider:  tablestore-mem
Plugin:    installed
Status:    available
```

Then start a Hermes session and use the memory tools, or run a quick manual
flow from a prompt:

1. Ask Hermes to remember a fact.
2. Ask Hermes to search for that fact.
3. Optionally delete the stored memory by id.

If the plugin is installed but `tablestore` is not yet available in the
Python environment Hermes is using, install the declared dependency first:

```bash
uv pip install --python /path/to/hermes/venv/bin/python tablestore==6.4.5
```

## CLI commands

When `memory.provider` is set to `tablestore-mem`, Hermes exposes:

```bash
hermes tablestore-mem add "User prefers concise answers"
hermes tablestore-mem add "User likes Rust" --metadata source=manual --metadata topic=preferences
hermes tablestore-mem add "Write this synchronously" --sync
hermes tablestore-mem search "concise answers"
hermes tablestore-mem search "Rust" --top-k 10
```

Notes:

- `hermes tablestore-mem add` writes one memory payload through the provider.
- `hermes tablestore-mem add` is asynchronous by default; pass `--sync` to wait.
- `hermes tablestore-mem search` returns JSON search results.
- `--metadata KEY=VALUE` can be repeated.
- These CLI commands are only registered when `tablestore-mem` is the active
  external memory provider.

## Operational notes

- The plugin uses `tablestore.OTSClient` memory methods directly.
- Request signing and authentication are handled by the OTS SDK.
- `is_available()` only checks local config presence and does not do network
  calls, which matches Hermes plugin expectations.
- `sync_turn()` runs asynchronously in a daemon thread to avoid blocking the
  agent loop.
- explicit `tablestore_remember` writes are asynchronous by default unless
  `sync=true` is provided.
- search and prefetch use rerank by default unless disabled in config or at
  call time.
- `tablestore_remember` may create multiple memory units from a single input
  if the backend extracts structured memories.

## Common issues

### Provider shows "not available"

Usually one of these values is missing:

- `TABLESTORE_MEMORY_ENDPOINT`
- `TABLESTORE_MEMORY_INSTANCE`
- `TABLESTORE_MEMORY_AK`
- `TABLESTORE_MEMORY_SK`

Run:

```bash
hermes memory status
```

### Authentication fails

Check:

- the endpoint is an OTS endpoint, not a separate application gateway
- the instance name is correct
- the AK/SK pair has permission for the target instance
- the SDK version is `tablestore==6.4.5`

### Search does not return the original raw text

This is expected in some deployments. `AddMemories` can extract and normalize
the input into multiple memory units, so search results may contain structured
facts rather than the exact original sentence.

### First write is slow

If `hermes_mem` does not exist yet and automatic creation is enabled, the
first write may take longer than a normal request because the provider first
creates the memory store and then writes into it. The default timeout is
`30` seconds.

## Repository layout

This repository is intentionally structured so Hermes can install it directly:

```text
.
├── __init__.py
├── cli.py
├── CHANGELOG.md
├── plugin.yaml
├── README.md
├── README.zh-CN.md
├── after-install.md
└── LICENSE
```

The repository root is the plugin root.

## License

This project is released under the MIT License. See [LICENSE](./LICENSE).
