# Ship Studio Plugin Development Guide

This guide covers everything you need to build, test, and distribute plugins for Ship Studio.

## Table of Contents

- [Overview](#overview)
- [Plugin Structure](#plugin-structure)
- [Manifest (plugin.json)](#manifest-pluginjson)
- [Plugin Entry Point](#plugin-entry-point)
- [Available APIs](#available-apis)
- [UI Slots](#ui-slots)
- [Invoking Tauri Commands](#invoking-tauri-commands)
- [Build Configuration](#build-configuration)
- [Testing Your Plugin](#testing-your-plugin)
- [Distributing Your Plugin](#distributing-your-plugin)
- [What Plugins Can Do](#what-plugins-can-do)
- [What Plugins Cannot Do](#what-plugins-cannot-do)
- [Example: Minimal Plugin](#example-minimal-plugin)
- [Example: Vercel Plugin](#example-vercel-plugin)

---

## Overview

Ship Studio plugins are **React components** bundled as ES modules that render into designated **slots** in the app UI. Plugins are **project-level** — each project has its own set of installed plugins at `<project>/.shipstudio/plugins/`.

Plugins run inside the host app's webview and share the same React instance. They access host app data and actions through a **plugin context** exposed on `window.__SHIPSTUDIO_PLUGIN_CONTEXT__`.

## Plugin Structure

```
my-plugin/
├── plugin.json          # Manifest (required)
├── dist/
│   └── index.js         # Built bundle (required)
├── src/
│   └── index.tsx        # Source entry point
├── package.json         # Build dependencies
├── tsconfig.json        # TypeScript config
└── vite.config.ts       # Build config
```

**Important:** The `dist/index.js` bundle must be committed to the repo. Ship Studio installs plugins by cloning the repo — it does not run any build step.

## Manifest (plugin.json)

Every plugin needs a `plugin.json` at the repo root:

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "A short description of what this plugin does",
  "author": "Your Name",
  "slots": ["toolbar"],
  "required_commands": []
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier. Must be lowercase, alphanumeric, hyphens only. Used as the directory name. |
| `name` | string | Yes | Display name shown in the Plugin Manager. |
| `version` | string | Yes | Semver version string (e.g., `"1.0.0"`). |
| `description` | string | Yes | Short description shown in the Plugin Manager. |
| `author` | string | No | Plugin author name. |
| `slots` | string[] | No | UI slots this plugin renders into. See [UI Slots](#ui-slots). |
| `required_commands` | string[] | No | Tauri commands this plugin needs to invoke. See [Invoking Tauri Commands](#invoking-tauri-commands). |
| `repository` | string | No | Source repository URL. |
| `min_app_version` | string | No | Minimum Ship Studio version required. |
| `icon` | string | No | Icon filename relative to plugin directory. |

## Plugin Entry Point

Your `dist/index.js` must export:

```typescript
// Display name (shown in the host app)
export const name: string;

// Map of slot name → React component
export const slots: Record<string, React.ComponentType>;
```

Example:

```tsx
export const name = 'My Plugin';

export const slots = {
  toolbar: MyToolbarComponent,
};
```

## Available APIs

Plugins access host app data through the plugin context. You can read it directly from `window.__SHIPSTUDIO_PLUGIN_CONTEXT__` or use the SDK hooks.

### Plugin Context

```typescript
interface PluginContext {
  pluginId: string;

  // Current project (null if no project open)
  project: {
    name: string;
    path: string;
    currentBranch: string;
    hasUncommittedChanges: boolean;
  } | null;

  // App actions
  actions: {
    showToast: (message: string, type?: 'success' | 'error') => void;
    openUrl: (url: string) => void;
    refreshGitStatus: () => void;
    refreshBranches: () => void;
    focusTerminal: () => void;
  };

  // Shell command execution
  shell: {
    exec: (command: string, args: string[]) => Promise<{
      stdout: string;
      stderr: string;
      exit_code: number;
    }>;
  };

  // Persistent storage (scoped to this plugin + project)
  storage: {
    read: () => Promise<Record<string, unknown>>;
    write: (scope: 'global' | 'project', data: Record<string, unknown>) => Promise<void>;
  };

  // Tauri command invocation (restricted to required_commands)
  invoke: {
    call: <T = unknown>(command: string, args?: Record<string, unknown>) => Promise<T>;
  };

  // Theme CSS variables
  theme: {
    bgPrimary: string;
    bgSecondary: string;
    bgTertiary: string;
    textPrimary: string;
    textSecondary: string;
    border: string;
    accent: string;
  };
}
```

### Accessing the Context

**Direct access (recommended for external plugins):**

```tsx
function getCtx() {
  const ctx = (window as any).__SHIPSTUDIO_PLUGIN_CONTEXT__;
  if (!ctx) throw new Error('Plugin context not available');
  return ctx;
}

function MyComponent() {
  const ctx = getCtx();
  const project = ctx.project;
  const toast = ctx.actions.showToast;
  // ...
}
```

**Using the SDK (`@shipstudio/plugin-sdk`):**

```tsx
import { useProject, useToast, useInvoke, useShell, useTheme } from '@shipstudio/plugin-sdk';

function MyComponent() {
  const project = useProject();
  const toast = useToast();
  const invoke = useInvoke();
  const shell = useShell();
  const theme = useTheme();
  // ...
}
```

### Actions

| Action | Description |
|--------|-------------|
| `showToast(message, type?)` | Show a toast notification. `type` is `'success'` or `'error'`. |
| `openUrl(url)` | Open a URL in the user's default browser. **Use this instead of `window.open()`** — `window.open` is blocked in Tauri's webview. |
| `refreshGitStatus()` | Trigger a refresh of the git status display. |
| `refreshBranches()` | Trigger a refresh of the branch list. |
| `focusTerminal()` | Focus the terminal panel. |

### Shell

Execute commands in the project directory with a 30-second timeout:

```typescript
const result = await ctx.shell.exec('node', ['--version']);
console.log(result.stdout); // "v20.11.0\n"
console.log(result.exit_code); // 0
```

### Storage

Persist data scoped to your plugin and the current project:

```typescript
// Read
const data = await ctx.storage.read();

// Write
await ctx.storage.write('project', { lastDeployedAt: Date.now() });
```

Storage is saved at `<project>/.shipstudio/plugins/<plugin-id>/storage.json`.

### Theme

Use theme values for consistent styling that matches the user's theme:

```tsx
const theme = ctx.theme;
// theme.bgPrimary, theme.bgSecondary, theme.textPrimary, etc.
```

Or use CSS variables directly in your styles: `var(--bg-primary)`, `var(--bg-secondary)`, `var(--text-primary)`, `var(--border)`, etc.

## UI Slots

Plugins render into designated slots in the Ship Studio UI:

| Slot | Location | Description |
|------|----------|-------------|
| `toolbar` | Workspace header (top bar) | Small buttons/indicators next to the publish button. Best for status indicators and quick actions. |
| `sidebar` | Dashboard sidebar | Content shown on the project dashboard sidebar. |
| `publish` | Next to the publish dropdown | Content related to the publish/deploy workflow. |
| `terminal` | Terminal toolbar area | Buttons/indicators in the terminal panel header. |

Most plugins will use the `toolbar` slot. Your component should be compact — a button or small indicator.

## Invoking Tauri Commands

Plugins can call Ship Studio's Rust backend commands through the **invoke proxy**. This is the most powerful plugin capability — it lets you interact with the full backend.

**Security:** Commands must be declared in `required_commands` in your manifest. Any command not listed will be rejected at runtime.

```json
{
  "required_commands": [
    "check_vercel_cli_status",
    "get_project_vercel_status"
  ]
}
```

```typescript
const invoke = ctx.invoke;

// Call a declared command
const status = await invoke.call<{ installed: boolean }>('check_vercel_cli_status');

// This will throw — not in required_commands
await invoke.call('some_other_command'); // Error!
```

### Available Commands

Ship Studio exposes commands for git, GitHub, Vercel, projects, and more. To see all available commands, browse `src-tauri/src/commands/` in the Ship Studio repo. Common ones:

**Git:** `get_git_status`, `get_branches`, `get_commit_log`, `get_diff`
**GitHub:** `get_github_auth_status`, `get_project_github_status`
**Vercel:** `check_vercel_cli_status`, `get_project_vercel_status`, `deploy_to_vercel`, `get_vercel_teams`, `list_vercel_projects`
**Projects:** `get_project_metadata`, `save_project_metadata`

**Important:** When a Rust command takes a struct argument (e.g., `options: DeployOptions`), you must wrap the args in the struct name:

```typescript
// Rust: fn deploy_to_vercel(options: DeployToVercelOptions)
await invoke.call('deploy_to_vercel', {
  options: {  // <-- wrapped in struct name
    projectPath: project.path,
    projectName: 'my-app',
  },
});
```

## Build Configuration

Plugins must be built as ES modules with React externalized (the host app provides React).

### vite.config.ts

```typescript
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    lib: {
      entry: 'src/index.tsx',
      formats: ['es'],
      fileName: () => 'index.js',
    },
    outDir: 'dist',
    emptyOutDir: true,
    rollupOptions: {
      external: ['react', 'react-dom', 'react/jsx-runtime'],
      output: {
        paths: {
          react: 'data:text/javascript,export default window.__SHIPSTUDIO_REACT__;export const useState=window.__SHIPSTUDIO_REACT__.useState;export const useEffect=window.__SHIPSTUDIO_REACT__.useEffect;export const useRef=window.__SHIPSTUDIO_REACT__.useRef;export const useCallback=window.__SHIPSTUDIO_REACT__.useCallback;export const useMemo=window.__SHIPSTUDIO_REACT__.useMemo;export const createElement=window.__SHIPSTUDIO_REACT__.createElement;',
          'react/jsx-runtime':
            'data:text/javascript,const R=window.__SHIPSTUDIO_REACT__;export const jsx=R.createElement;export const jsxs=R.createElement;export const Fragment=R.Fragment;',
          'react-dom': 'data:text/javascript,export default window.__SHIPSTUDIO_REACT_DOM__;',
        },
      },
    },
    minify: false,
  },
});
```

**Why `data:` URLs?** Plugins are loaded via Blob URLs in Tauri's webview. Normal `import 'react'` won't resolve. The `paths` config maps React imports to inline modules that read from window globals provided by the host app.

### package.json

```json
{
  "name": "@shipstudio/plugin-my-plugin",
  "private": true,
  "scripts": {
    "build": "vite build"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "react": "^19.0.0",
    "typescript": "^5.0.0",
    "vite": "^6.0.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false
  },
  "include": ["src"]
}
```

## Testing Your Plugin

1. Build your plugin: `npm run build`
2. In a Ship Studio project, open the Plugin Manager
3. Go to Library > "Install from URL"
4. Enter your local or remote repo URL
5. The plugin will be cloned and loaded immediately

For rapid iteration during development, you can manually copy your `dist/index.js` and `plugin.json` into `<project>/.shipstudio/plugins/<plugin-id>/` and reload the project.

## Distributing Your Plugin

1. Push your plugin to a **public GitHub repository**
2. Make sure `dist/index.js` and `plugin.json` are committed
3. Users can install via the Plugin Manager URL input

To get listed in the **Plugin Library** (the browsable list in the Plugin Manager), submit a PR to [ship-studio/plugin-registry](https://github.com/ship-studio/plugin-registry) adding your plugin to `registry.json`.

## What Plugins Can Do

- **Render UI** in designated slots (toolbar, sidebar, publish area, terminal area)
- **Show toast notifications** to the user
- **Open URLs** in the user's browser
- **Execute shell commands** in the project directory (30s timeout)
- **Read/write persistent storage** scoped to the plugin and project
- **Call Tauri backend commands** (only those declared in `required_commands`)
- **Access project data**: name, path, current branch, uncommitted changes status
- **Access theme data** for consistent styling
- **Trigger app actions**: refresh git status, refresh branches, focus terminal
- **Use React hooks and state** — full React capabilities
- **Include CSS styles** — either inline or via CSS-in-JS

## What Plugins Cannot Do

- **Access the filesystem directly** — use `shell.exec` or `invoke` for file operations
- **Call undeclared Tauri commands** — only commands in `required_commands` are allowed
- **Modify other plugins** — plugins are isolated from each other
- **Access `window.open()`** — blocked by Tauri's webview security. Use `actions.openUrl()` instead
- **Import from `@tauri-apps/*`** — Tauri APIs are not available to plugins. Use the invoke proxy instead
- **Run background processes** — shell commands have a 30-second timeout
- **Access other projects' data** — storage and context are scoped to the current project
- **Modify the host app's DOM** outside their designated slot
- **Import npm packages at runtime** — all dependencies must be bundled into `dist/index.js`
- **Use Node.js APIs** — plugins run in the browser, not Node.js

## Example: Minimal Plugin

A plugin that shows a button in the toolbar:

**plugin.json:**
```json
{
  "id": "hello-world",
  "name": "Hello World",
  "version": "1.0.0",
  "description": "A minimal example plugin",
  "slots": ["toolbar"]
}
```

**src/index.tsx:**
```tsx
import { useState } from 'react';

function getCtx() {
  const ctx = (window as any).__SHIPSTUDIO_PLUGIN_CONTEXT__;
  if (!ctx) throw new Error('Plugin context not available');
  return ctx;
}

function HelloToolbar() {
  const ctx = getCtx();
  const [count, setCount] = useState(0);

  return (
    <button
      style={{
        fontSize: 12,
        padding: '4px 8px',
        borderRadius: 6,
        border: `1px solid ${ctx.theme.border}`,
        background: ctx.theme.bgSecondary,
        color: ctx.theme.textPrimary,
        cursor: 'pointer',
      }}
      onClick={() => {
        setCount(count + 1);
        ctx.actions.showToast(`Clicked ${count + 1} times!`, 'success');
      }}
    >
      Hello ({count})
    </button>
  );
}

export const name = 'Hello World';

export const slots = {
  toolbar: HelloToolbar,
};
```

## Example: Vercel Plugin

For a full real-world example, see the [Vercel plugin source](https://github.com/ship-studio/plugin-vercel).

It demonstrates:
- Using `invoke` to call multiple Tauri commands
- Declaring `required_commands` for the command allowlist
- Complex state management (CLI status, project linking, team selection)
- Modal dialogs rendered from a toolbar slot
- Hover dropdowns for quick URL access
- Using `actions.openUrl()` for external links
