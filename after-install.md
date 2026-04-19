# TableStore plugin installed

Next steps:

1. Enable it as the active external memory provider:

```bash
hermes config set memory.provider tablestore-mem
```

2. Add credentials to `~/.hermes/.env`:

```bash
TABLESTORE_MEMORY_AK=your_access_key_id
TABLESTORE_MEMORY_SK=your_access_key_secret
TABLESTORE_MEMORY_ENDPOINT=https://your-instance.region.ots.aliyuncs.com
TABLESTORE_MEMORY_INSTANCE=your-instance-name
TABLESTORE_MEMORY_STORE=hermes_mem
```

3. Optionally create `~/.hermes/tablestore_memory.json` for non-secret config:

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

If `TABLESTORE_MEMORY_STORE` is omitted, the plugin uses `hermes_mem` and
creates it automatically when missing.

4. Verify:

```bash
hermes memory status
```

If the plugin is available, Hermes will use it for:

- memory prefetch before turns
- turn sync after completed turns
- explicit tools:
  - `tablestore_profile`
  - `tablestore_search`
  - `tablestore_remember`
  - `tablestore_forget`

CLI commands are also available when `tablestore-mem` is the active provider:

```bash
hermes tablestore-mem add "Remember this fact"
hermes tablestore-mem search "this fact"
```
