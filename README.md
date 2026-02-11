# Ship Studio Plugin Registry

Official plugin registry for [Ship Studio](https://shipstudio.dev). This registry powers the Plugin Library in the Ship Studio Plugin Manager.

## For Users

Plugins are installed per-project from the Plugin Manager inside Ship Studio. Open a project, click the puzzle icon in the header, and browse the Library tab.

## For Plugin Developers

See the **[Plugin Development Guide](PLUGIN_DEVELOPMENT.md)** for everything you need to build a Ship Studio plugin.

For a real-world example, check out the [Vercel plugin](https://github.com/ship-studio/plugin-vercel).

## Adding Your Plugin to the Registry

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
