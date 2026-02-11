# Ship Studio Plugin Registry

Official plugin registry for [Ship Studio](https://shipstudio.dev).

## Adding a Plugin

Submit a pull request adding your plugin to `registry.json`.

Each entry requires:

| Field | Description |
|-------|-------------|
| `id` | Unique plugin identifier (matches `plugin.json`) |
| `name` | Display name |
| `description` | Short description shown in the plugin library |
| `repo` | GitHub repository URL |
| `author` | Plugin author |
| `category` | One of: `deployment`, `integration`, `tools`, `ui` |
| `icon` | Icon identifier (optional) |

## For Plugin Developers

See the [Plugin Development Guide](https://github.com/ship-studio/plugin-vercel) for an example plugin.
