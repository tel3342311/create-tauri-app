# Tauri Quick-Start Tutorial

This tutorial walks you through the two repositories you're working with:

| Repo | What it is | You use it to… |
| --- | --- | --- |
| **create-tauri-app** (this repo) | A Rust CLI that scaffolds new Tauri projects from the folders in [`templates/`](./templates) | Create a new desktop/mobile app |
| **tauri-action** | A GitHub Action (TypeScript, entry point `src/index.ts`, built into `dist/index.js`) | Build and release that app on GitHub for macOS, Windows and Linux |

By the end you will have:

1. Scaffolded a Tauri v2 app
2. Learned how the frontend ↔ Rust bridge works and added your own features
3. Set up GitHub Actions to test every PR and publish releases

---

## Part 0 — Prerequisites

| Tool | Why | Install |
| --- | --- | --- |
| Rust (stable, ≥ 1.85) | Tauri backend | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| Node.js LTS + a package manager (npm / pnpm / yarn / bun) | Frontend + Tauri CLI | https://nodejs.org |
| OS system libs | WebView + bundling | see below |

**Linux (Debian/Ubuntu):**

```bash
sudo apt-get update
sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf xdg-utils build-essential curl wget file libssl-dev
```

**macOS:** `xcode-select --install`
**Windows:** Microsoft C++ Build Tools ("Desktop development with C++") and WebView2 (pre-installed on Windows 10/11).

---

## Part 1 — Scaffold an app

Run any one of these in the folder where the project should live:

```bash
npm create tauri-app@latest
```

```bash
pnpm create tauri-app
```

```bash
cargo install create-tauri-app --locked
cargo create-tauri-app
```

Answer the prompts (project name, identifier such as `com.yourname.myapp`, frontend language, package manager, UI template). You can also skip the prompts:

```bash
npm create tauri-app@latest my-app -- --template react-ts --manager npm --yes
```

Available templates (one folder per template under [`templates/`](./templates)):
`vanilla`, `vanilla-ts`, `react`, `react-ts`, `vue`, `vue-ts`, `svelte`, `svelte-ts`, `solid`, `solid-ts`, `preact`, `preact-ts`, `angular` (JS/TS) and `leptos`, `yew`, `sycamore`, `dioxus` (Rust frontends), `blazor` (.NET).

Then run it:

```bash
cd my-app
npm install
npm run tauri dev
```

A native window opens with a "Greet" form. Edit frontend files and the window hot-reloads; edit Rust files and the app recompiles and restarts.

---

## Part 2 — How a Tauri app is organised

Every template is the shared [`templates/_base_/src-tauri`](./templates/_base_/src-tauri) folder plus a frontend folder. A generated project looks like:

```
my-app/
├── package.json            # frontend deps + "tauri" script (@tauri-apps/cli)
├── index.html / src/       # your web UI (React, Vue, plain JS, …)
└── src-tauri/              # the Rust side
    ├── Cargo.toml          # Rust deps (tauri, tauri-plugin-opener, serde…)
    ├── build.rs            # calls tauri_build::build()
    ├── tauri.conf.json     # app name, version, window, build & bundle config
    ├── capabilities/
    │   └── default.json    # which APIs the frontend may call (security)
    ├── icons/              # app icons for every platform
    └── src/
        ├── main.rs         # desktop entry → calls my_app_lib::run()
        └── lib.rs          # your app: commands, plugins, state
```

### The mental model

```
┌──────────────── WebView (HTML/CSS/JS) ───────────────┐
│  invoke("greet", { name })   listen("progress", cb)  │
└───────────────│───────────────────────▲──────────────┘
                │ IPC (JSON)            │ events
┌───────────────▼───────────────────────│──────────────┐
│  #[tauri::command] fn greet(...)   app.emit(...)     │
│                 Rust (src-tauri)                     │
└──────────────────────────────────────────────────────┘
```

- The **frontend** is a normal web app rendered in the OS WebView.
- The **backend** is Rust. The frontend calls Rust functions with `invoke`, and Rust pushes data back with **events**.
- **Capabilities** say which windows can use which permissions. Nothing is allowed unless it is listed.

### The generated code, explained

`src-tauri/src/lib.rs` (from [`%(v2)%lib.rs.lte`](./templates/_base_/src-tauri/src/%25(v2)%25lib.rs.lte)):

```rust
#[tauri::command]                       // 1. expose a Rust fn to JS
fn greet(name: &str) -> String {
    format!("Hello, {}! You've been greeted from Rust!", name)
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]  // also the iOS/Android entry point
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_opener::init())             // 2. register plugins
        .invoke_handler(tauri::generate_handler![greet]) // 3. register commands
        .run(tauri::generate_context!())                 // 4. reads tauri.conf.json
        .expect("error while running tauri application");
}
```

Frontend side (vanilla template, [`main.js.lte`](./templates/template-vanilla/src/main.js.lte)):

```js
const { invoke } = window.__TAURI__.core;   // available because withGlobalTauri: true
const msg = await invoke("greet", { name: "Ada" });
```

With a bundler template (React/Vue/Svelte…) you import it instead:

```ts
import { invoke } from "@tauri-apps/api/core";
const msg = await invoke<string>("greet", { name: "Ada" });
```

> Argument names: JS uses **camelCase** keys, Rust uses **snake_case** parameters. `invoke("save", { fileName })` maps to `fn save(file_name: String)`.

---

## Part 3 — Programming recipes

### 3.1 Add a new command

```rust
// src-tauri/src/lib.rs
#[tauri::command]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// register it next to greet:
.invoke_handler(tauri::generate_handler![greet, add])
```

```ts
const sum = await invoke<number>("add", { a: 2, b: 3 }); // 5
```

### 3.2 Async commands and errors

Return `Result<T, E>`. `Ok` resolves the JS promise, `Err` rejects it. `E` must be serializable (`String` is the easiest).

```rust
#[tauri::command]
async fn read_text(path: String) -> Result<String, String> {
    std::fs::read_to_string(&path).map_err(|e| e.to_string())
}
```

```ts
try {
  const text = await invoke<string>("read_text", { path: "/tmp/a.txt" });
} catch (err) {
  console.error("Rust said:", err);
}
```

### 3.3 Pass structs (serde)

`serde` is already in `Cargo.toml`.

```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct NewTodo { title: String }

#[derive(Serialize, Clone)]
struct Todo { id: u32, title: String, done: bool }

#[tauri::command]
fn create_todo(todo: NewTodo) -> Todo {
    Todo { id: 1, title: todo.title, done: false }
}
```

```ts
const todo = await invoke("create_todo", { todo: { title: "Learn Tauri" } });
```

### 3.4 Shared app state

```rust
use std::sync::Mutex;
use tauri::State;

#[derive(Default)]
struct Counter(Mutex<u32>);

#[tauri::command]
fn increment(counter: State<Counter>) -> u32 {
    let mut n = counter.0.lock().unwrap();
    *n += 1;
    *n
}

pub fn run() {
    tauri::Builder::default()
        .manage(Counter::default())                    // register the state
        .invoke_handler(tauri::generate_handler![increment])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 3.5 Events: Rust → frontend

```rust
use tauri::{AppHandle, Emitter};

#[tauri::command]
async fn long_task(app: AppHandle) {
    for pct in (0..=100).step_by(10) {
        app.emit("progress", pct).unwrap();
        std::thread::sleep(std::time::Duration::from_millis(200));
    }
}
```

```ts
import { listen } from "@tauri-apps/api/event";
const unlisten = await listen<number>("progress", (e) => console.log(e.payload + "%"));
await invoke("long_task");
unlisten();
```

### 3.6 Plugins (filesystem, dialogs, HTTP, store, updater…)

The template already ships `tauri-plugin-opener` (open URLs/files with the default app). Add more with the CLI. It updates `Cargo.toml`, `package.json`, `lib.rs` and the capabilities for you:

```bash
npm run tauri add dialog
```

```bash
npm run tauri add fs
```

```ts
import { open } from "@tauri-apps/plugin-dialog";
const file = await open({ multiple: false });
```

If a call fails with a *"not allowed"* error, add the permission to `src-tauri/capabilities/default.json`:

```json
{
  "identifier": "default",
  "windows": ["main"],
  "permissions": ["core:default", "opener:default", "dialog:default", "fs:default"]
}
```

### 3.7 Configure the window and app

Edit `src-tauri/tauri.conf.json`:

```jsonc
{
  "productName": "my-app",
  "version": "0.1.0",                 // used by tauri-action for release names
  "identifier": "com.yourname.myapp", // must be unique, reverse-DNS
  "app": {
    "windows": [{ "title": "My App", "width": 1024, "height": 768, "resizable": true }]
  },
  "bundle": { "active": true, "targets": "all" }
}
```

Replace the icons with your own:

```bash
npm run tauri icon ./my-logo.png
```

### 3.8 Build a release locally

```bash
npm run tauri build
```

Installers end up in `src-tauri/target/release/bundle/` (`.dmg`/`.app` on macOS, `.msi`/`-setup.exe` on Windows, `.deb`/`.rpm`/`.AppImage` on Linux). You can only build for the OS you're on, which is why Part 4 uses CI.

### 3.9 Mobile (optional)

```bash
npm run tauri android init
```

```bash
npm run tauri android dev
```

(iOS: `tauri ios init` / `tauri ios dev`, on macOS only.) `#[cfg_attr(mobile, tauri::mobile_entry_point)]` in `lib.rs` is what makes this work.

### Command cheat-sheet

| Command | Does |
| --- | --- |
| `npm run tauri dev` | Run with hot reload |
| `npm run tauri build` | Production build + installers |
| `npm run tauri build -- --debug` | Build with devtools enabled |
| `npm run tauri add <plugin>` | Add an official plugin |
| `npm run tauri icon <png>` | Generate all icons |
| `npm run tauri info` | Print environment info (useful for bug reports) |

Right-click → **Inspect** opens the WebView devtools in dev builds. Rust `println!` output goes to the terminal running `tauri dev`.

---

## Part 4 — GitHub Actions with tauri-action

`tauri-action` installs the Tauri CLI if needed, runs `tauri build`, and (optionally) creates a GitHub Release and uploads the installers to it. Its inputs are defined in `tauri-action/action.yml`, and ready-made workflows live in `tauri-action/examples/`.

### 4.1 One-time repo setup

1. Push your app to GitHub. The project path must **not** be git-ignored.
2. **Settings → Actions → General → Workflow permissions** → choose **Read and write permissions**. Without this the action can't create releases.
3. Pick a package manager and commit its lockfile (`package-lock.json`, `pnpm-lock.yaml`, …).

### 4.2 Workflow A: build on every pull request (no release)

Create `.github/workflows/test.yml`:

```yaml
name: test-on-pr

on: [pull_request]

jobs:
  test-tauri:
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: 'macos-latest'       # Apple Silicon
            args: '--target aarch64-apple-darwin'
          - platform: 'macos-latest'       # Intel Mac
            args: '--target x86_64-apple-darwin'
          - platform: 'ubuntu-22.04'
            args: ''
          - platform: 'windows-latest'
            args: ''

    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@v4

      - name: setup node
        uses: actions/setup-node@v4
        with:
          node-version: lts/*
          cache: 'npm'

      - name: install Rust stable
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.platform == 'macos-latest' && 'aarch64-apple-darwin,x86_64-apple-darwin' || '' }}

      - name: Rust cache
        uses: swatinem/rust-cache@v2
        with:
          workspaces: './src-tauri -> target'

      - name: install dependencies (ubuntu only)
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf xdg-utils

      - name: install frontend dependencies
        run: npm ci            # or: pnpm install / yarn install / bun install

      # No tagName / releaseId → build only, nothing is uploaded
      - uses: tauri-apps/tauri-action@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          args: ${{ matrix.args }}
```

### 4.3 Workflow B: publish a release

Create `.github/workflows/release.yml`. Pushing to the `release` branch (or pushing a `v*` tag) builds all platforms and uploads the installers to a **draft** release:

```yaml
name: publish

on:
  push:
    branches: [release]
    # or release on tags instead:
    # tags: ['v*']
  workflow_dispatch:      # allows a manual "Run workflow" button

jobs:
  publish-tauri:
    permissions:
      contents: write     # needed to create the release
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: 'macos-latest'
            args: '--target aarch64-apple-darwin'
          - platform: 'macos-latest'
            args: '--target x86_64-apple-darwin'
          - platform: 'ubuntu-22.04'
            args: ''
          - platform: 'windows-latest'
            args: ''

    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
          cache: 'npm'

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.platform == 'macos-latest' && 'aarch64-apple-darwin,x86_64-apple-darwin' || '' }}

      - uses: swatinem/rust-cache@v2
        with:
          workspaces: './src-tauri -> target'

      - name: install dependencies (ubuntu only)
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf xdg-utils

      - run: npm ci

      - uses: tauri-apps/tauri-action@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tagName: app-v__VERSION__        # __VERSION__ = "version" from tauri.conf.json
          releaseName: 'App v__VERSION__'
          releaseBody: 'See the assets to download and install this version.'
          releaseDraft: true               # review it, then click "Publish"
          prerelease: false
          args: ${{ matrix.args }}
```

**How to ship a version:**

1. Bump `"version"` in `src-tauri/tauri.conf.json` (e.g. `0.2.0`).
2. Commit, then push to the `release` branch:

   ```bash
   git push origin main:release
   ```

3. Wait for the four jobs to finish. All of them upload to the same draft release `app-v0.2.0`.
4. Open **Releases** on GitHub, check the assets and click **Publish**.

### 4.4 Useful tauri-action inputs

| Input | Default | Use it for |
| --- | --- | --- |
| `tagName` / `releaseName` / `releaseBody` | – | Create or update a release (`__VERSION__` is replaced) |
| `releaseId` | – | Upload to a release you created in an earlier job (see `examples/publish-to-manual-release.yml`) |
| `releaseDraft` / `prerelease` | `false` | Release visibility |
| `generateReleaseNotes` | `false` | Use GitHub's auto-generated notes |
| `projectPath` | `.` | App lives in a sub-folder (monorepo) |
| `tauriScript` | auto | Custom CLI command, e.g. `pnpm tauri` |
| `args` | – | Extra `tauri build` args, e.g. `--target universal-apple-darwin` |
| `uploadWorkflowArtifacts` | `false` | Also attach builds to the workflow run (handy for PR testing) |
| `uploadUpdaterJson` | `true` | Upload `latest.json` for the auto-updater |
| `retryAttempts` | `0` | Retry flaky builds/uploads |
| `mobile` | – | *Experimental*: `android` or `ios` |

Outputs: `releaseId`, `releaseHtmlUrl`, `releaseUploadUrl`, `artifactPaths`, `appVersion`. Read them in later steps by giving the step an `id`.

Tip: to download PR builds without making a release, add `uploadWorkflowArtifacts: true` to Workflow A.

### 4.5 Optional: code signing and auto-updates

Add secrets under **Settings → Secrets and variables → Actions** and pass them as `env:` on the tauri-action step.

**Auto-updater signing** (needed if you use `tauri-plugin-updater`):

```bash
npm run tauri signer generate -- -w ~/.tauri/myapp.key
```

```yaml
env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
  TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
```

Put the public key in `tauri.conf.json` → `plugins.updater.pubkey`, and set the endpoint to
`https://github.com/<owner>/<repo>/releases/latest/download/latest.json`. tauri-action uploads `latest.json` for you.

**macOS signing and notarization:** see `tauri-action/examples/publish-to-auto-release-universal-macos-app-with-signing-certificate.yml`. It needs `APPLE_CERTIFICATE` (base64 `.p12`), `APPLE_CERTIFICATE_PASSWORD`, `KEYCHAIN_PASSWORD`, `APPLE_ID`, `APPLE_PASSWORD` (app-specific password) and `APPLE_TEAM_ID`.

### 4.6 Troubleshooting

| Symptom | Fix |
| --- | --- |
| `Resource not accessible by integration` | Enable read/write workflow permissions, or add `permissions: contents: write` |
| Linux job: `webkit2gtk-4.1 not found` | The apt step's `if:` must match the matrix `platform` string exactly |
| `couldn't find tauri.conf.json` | Set `projectPath`, and make sure `src-tauri` isn't in `.gitignore` |
| Each job creates its own release | Use the same `tagName` in every job, or create the release once and pass `releaseId` |
| Slow builds | Add `swatinem/rust-cache@v2` (shown above) |

---

## Part 5 — Hacking on these two repos themselves (optional)

**create-tauri-app** (Rust CLI):

- `src/main.rs` / `src/lib.rs`: CLI entry and prompt flow. `src/args.rs` has the flags, `src/template.rs` the template list, `src/package_manager.rs` the npm/pnpm/yarn/bun/deno/cargo/dotnet handling.
- `templates/**`: files ending in `.lte` are rendered with `{% var %}` / `{% if v2 %}` placeholders. A filename prefix such as `%(v2)%` or `%(pnpm-yarn-npm-deno-bun)%` means "include only for that Tauri version or package manager". `_gitignore` is renamed to `.gitignore`.
- Try it locally:

  ```bash
  cargo run -- my-test-app --template vanilla --manager npm --yes
  ```

**tauri-action** (TypeScript):

- `src/index.ts` orchestrates: `inputs.ts` → `build.ts` → `create-release.ts` → `upload-release-assets.ts` / `upload-version-json.ts` / `upload-workflow-artifacts.ts`.
- The action runs the compiled `dist/index.js` (see `runs:` in `action.yml`). Rebuild it after code changes with `pnpm install && pnpm build`.
- To test your fork in a workflow, point the step at it: `uses: <your-user>/tauri-action@<branch>`.

---

### Further reading

- Tauri docs: https://tauri.app/start/
- Calling Rust from the frontend: https://tauri.app/develop/calling-rust/
- Permissions and capabilities: https://tauri.app/security/capabilities/
- Distribution and signing: https://tauri.app/distribute/
