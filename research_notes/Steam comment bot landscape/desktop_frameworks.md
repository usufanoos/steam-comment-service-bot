# Desktop Application Frameworks for Wrapping a Node.js Steam Account Manager

## Electron: bundle size, memory per window, embedding node_modules, auto-update, code-signing

### Takeaway
Electron is still the lowest-friction fit for an existing Node.js codebase (steam-user / steam-session run unchanged in the main process), at the cost of 80–300 MB bundles and ~100–170 MB idle memory for the first window plus a Chromium renderer per additional window. Auto-update is a solved problem via electron-updater on NSIS (Windows) and Squirrel.Mac (macOS); macOS auto-update requires a signed+notarized build.

### Cited Findings
- Hello-World Electron 34.x bundle measured ~85 MB; a six-window test app ~244 MB; macOS empty-app ~268 MB — [betterstack Tauri vs Electron](https://betterstack.com/community/guides/scaling-nodejs/tauri-vs-electron-vs-deno-vs-electrobun.md)
- Single-window idle memory: 168 MB (Electron) vs 42 MB (Tauri); six-window active 409 MB — [betterstack](https://betterstack.com/community/guides/scaling-nodejs/tauri-vs-electron-vs-deno-vs-electrobun.md)
- Each Electron window = a separate Chromium renderer process — [betterstack Electron architecture](https://betterstack.com/community/guides/scaling-nodejs/electron-desktop-architecture.md)
- One runtime test found the memory gap smaller: 128 MB (Electron) vs 109 MB (Tauri) — [betterstack](https://betterstack.com/community/guides/scaling-nodejs/tauri-vs-electron-vs-deno-vs-electrobun.md)
- `process.dlopen` cannot load `.node` files inside an ASAR archive; they must be unpacked — [Electron asar docs](https://electronjs.org/docs/latest/tutorial/asar-archives)
- `@electron-forge/plugin-auto-unpack-natives` automatically adds all native Node modules in `node_modules` to `asar.unpack` — [electronforge docs](https://www.electronforge.io/config/plugins/auto-unpack-natives)
- electron-builder unpacks `.node` files by default; `asarUnpack` globs catch any that it misses — [electron-builder docs](https://www.electron.build/docs/features/auto-update)
- Native modules must be compiled against Electron's Node runtime via `electron-rebuild` — [Electron using-native-node-modules](https://github.com/electron/electron/blob/v4.2.10/docs/tutorial/using-native-node-modules.md)
- electron-updater supports Linux/macOS/Windows and verifies code signatures on macOS and Windows; NSIS is required on Windows (not Squirrel.Windows) for simplified auto-update — [electron-builder auto-update](https://www.electron.build/docs/features/auto-update)
- macOS auto-update via Squirrel.Mac requires the app to be code-signed; zip target is required so `latest-mac.yml` can be created — [Electron code-signing](https://electronjs.org/docs/tutorial/code-signing)
- As of mid-2026, Electron Forge is the official Electron-maintained distribution tool; electron-builder remains the most-downloaded alternative — [Doyensec: secure Electron auto-updater](https://blog.doyensec.com/2026/02/16/electron-safe-updater.html)
- xterm.js works seamlessly in Electron; the common pattern is node-pty in main + xterm + fit addon in renderer, with IPC channels like `terminal-data` — [xterm.js README](https://unpkg.com/@xterm/xterm@6.0.0/README.md)
- IPC is noticeably faster than WebSocket for in-process Electron log streaming — [scriptol IPC vs WebSocket](https://www.scriptol.com/javascript/ipc-vs-websocket.php)
- Electron Tray API is fully supported on Windows/macOS/Linux, with platform quirks (macOS: left-click opens menu; Linux: StatusNotifierItem falls back to GtkStatusIcon) — [Electron Tray docs](https://electronjs.org/docs/latest/api/tray)

### Inferences
- A multi-account dashboard that opens one BrowserWindow per account will push Electron into 300–500 MB territory fast; a single-window dashboard rendering account cards is cheaper.
- Because steam-user/steam-session are pure JS (no native `.node`), the ASAR unpacking risk for this codebase is limited to its transitive deps (e.g. `sqlite3`, `bcrypt`, `node-pty` if added for a terminal view) — those must be unpacked.

### Gaps
- No measured bundle size of the 3urobeat bot specifically; its ~150 MB `node_modules` is user-reported, not a cited figure.

---

## Tauri 2.x: hosting a long-lived Node sidecar and Steam-specific prior art

### Takeaway
Tauri 2 has an official, documented "Node.js as a sidecar" pattern that ships a compiled-to-binary Node app via `bundle.externalBin`, with Rust spawning it through the shell plugin. It works for a long-lived process but requires manual lifecycle management (Tauri does not kill sidecars on exit). No high-profile Steam tool uses Tauri yet; the only Steam projects found are *Samira* (Linux achievement manager) and *Depo Tool* (Steam library/manifest manager), both small.

### Cited Findings
- Official v2 pattern: build Node app with `pkg` or Bun compile, place at `src-tauri/binaries/<name>-<target-triple>`, list basename in `bundle.externalBin`, spawn via shell plugin — [Tauri v2: Node.js as a sidecar](https://v2.tauri.app/learn/sidecar-nodejs/)
- `pkg` cannot bundle native `.node` modules loaded via dynamic paths; use `bun build --compile` or vendor prebuilds — [Tauri v2 sidecar docs](https://v2.tauri.app/learn/sidecar-nodejs/)
- Alternative: ship a stock Node.js binary as the sidecar and the JS source + node_modules as Resources — [Tauri docs-wip sidecar](https://jonaskruckenberg.github.io/tauri-docs-wip/examples/sidecar.html)
- Tauri strips the target-triple suffix in production bundles (dev path: `node-aarch64-apple-darwin`; prod: `node`) — [DEV: Bundling a CLI binary as a Tauri v2 sidecar](https://dev.to/chenxxpro/bundling-a-cli-binary-as-a-tauri-v2-sidecar-architecture-and-lessons-from-building-moonproxy-21n3)
- Tauri does not kill sidecars automatically; keep the child handle in `State` and kill on exit — [DEV: Bundling a CLI binary](https://dev.to/chenxxpro/bundling-a-cli-binary-as-a-tauri-v2-sidecar-architecture-and-lessons-from-building-moonproxy-21n3)
- Capabilities file needs `shell` execute entry with `sidecar: true` — [DEV: Bundling a CLI binary](https://dev.to/chenxxpro/bundling-a-cli-binary-as-a-tauri-v2-sidecar-architecture-and-lessons-from-building-moonproxy-21n3)
- Tauri auto-updater: Ed25519 signatures generated with `tauri signer generate`; `TAURI_SIGNING_PRIVATE_KEY` env var at build time; cannot be disabled — [tauri-cli signer source](https://docs.rs/crate/tauri-cli/2.11.3/source/src/signer/generate.rs)
- Tauri v2 updater requires `createUpdaterArtifacts: true` in bundle config and Rust ≥ 1.77.2 — [stuffbucket: tauri-updater](https://stuffbucket.github.io/skills/catalog/tauri-updater/)
- Empty Tauri app on Windows x64 ≈ 1 MB (installer/exe); Neutralino ≈ 8 MB; both use OS WebView (WebView2 on Windows, WebKit on macOS, webkit2gtk on Linux) — [web-to-desktop benchmark](https://git.tdem.in/Elanis/web-to-desktop-framework-comparison) ; [LogRocket](https://blog.logrocket.com/why-use-electron-alternative)
- Tauri Steam projects: *Samira* (Linux SAM clone, Tauri+Rust, requires running Steam client) — [github.com/jsnli/Samira](https://github.imc.re/jsnli/Samira)
- *Depo Tool* — Tauri+Rust+React Steam library manifest manager — [depo-tool project page](https://lifeofhasan.pages.dev/project/depo-tool)

### Inferences
- A Tauri wrapper for the 3urobeat bot would still ship ~150 MB of `node_modules` as Resources plus a ~50 MB bundled Node binary, so the "Tauri is tiny" advantage largely evaporates — the shell binary is small, but the payload is not.
- Because steam-user depends on dynamically loaded native modules in some transitive deps, Bun `--compile` is safer than `pkg`; alternatively ship a stock Node runtime + source tree.
- Tauri gives real win on per-window memory (42 MB idle) only if the dashboard is Tauri-native; the Node sidecar still has its own ~30–60 MB footprint per persistent login session.

### Gaps
- No real-world measurements of a Tauri-wrapped Node sidecar app of this size (150 MB node_modules).
- No Steam-ecosystem project uses Tauri's Node sidecar pattern at scale that I could cite.

---

## Keep-Node approach: child_process + thin GUI shell (Wails / Neutralino / pywebview / WebView2)

### Takeaway
A "keep the CLI, add a thin GUI shell" approach is viable and arguably the lowest-risk path: Wails/Neutralino/pywebview act purely as windowing layers while the existing Node bot runs as a child process and streams logs over stdio or a local WebSocket. Documentation specifically for Wails spawning Node is thin; the general `go2node` pattern is established.

### Cited Findings
- Neutralino release build measured ~2 MB uncompressed on Linux; Tauri ~8 MB — [LogRocket: Electron alternatives](https://blog.logrocket.com/why-use-electron-alternative)
- Neutralino uses the OS web view and ships as a small loader + HTML/JS assets — [LogRocket](https://blog.logrocket.com/why-use-electron-alternative)
- `go2node` Go package provides IPC between Go and Node; requires Go ≥ 1.12.x and Node ≥ 8.x — [go2node pkg.go.dev](https://pkg.go.dev/github.com/csystem-it/go2node)
- Node `child_process.spawn()` is the standard entry point for a Go/Rust/C# host to drive a Node CLI — [Node child_process docs](https://github.com/nodejs/node/blob/6e1612fa15ff303bb5d71586bc0c34724f58adf1/doc/api/child_process.md)
- `WebviewJS` / `win_webview2` bind WebView2 to Node (reverse direction: Node drives the window) — [webview.js.org](https://webview.js.org/installation.md)
- NativeWebHost.Windows (.NET + WebView2) exposes a JS bridge for host↔page messaging — [NativeWebHost.Windows nuget](https://www.nuget.org/packages/NativeWebHost.Windows/)

### Inferences
- The cheapest architecture is: existing bot CLI (unchanged) + Wails or Neutralino shell + logs over a local Socket.IO/WebSocket that the UI consumes. This mirrors ASF-ui's split-process design (see next section).
- Neutralino's loader approach keeps the Node bundle as a separate payload that the user can update or inspect — good for a tool that already auto-updates its JS via npm/git.

### Gaps
- No Wails-specific guide to packaging a Node sidecar surfaced; needs direct inspection of Wails' `wails.json` + `assets` bundling.
- pywebview + Node child process — only indirect references; no end-to-end example found.

---

## ASF-ui as third option: local web server + browser

### Takeaway
ArchiSteamFarm's own IPC interface is a Kestrel web server inside the ASF process; ASF-ui is a Vue+webpack app served by it and is explicitly swappable. This "headless daemon + local web UI" is a legitimate third option — no desktop packaging, auto-updates follow the CLI release, and multi-account is just a REST API consumer problem.

### Cited Findings
- ASF IPC is "ASF web interface" based on Kestrel HTTP server — [ASF IPC wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/IPC)
- ASF API is RESTful + JSON, officially maintained by core ASF team — [ASF IPC wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/IPC)
- ASF-ui is a community project (not core-maintained), Vue + webpack + SCSS — [ASF-ui ecosyste.ms](https://awesome.ecosyste.ms/projects/github.com%2Fjustarchinet%2Fasf-ui)
- Users can swap ASF-ui for a custom web UI via `--path` + a custom `www` directory — [ASF IPC wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/IPC)
- Recent ASF releases ship built-in Scalar Swagger UI at `/swagger` — [ASF 6.1.4.3 release notes](https://newreleases.io/project/github/JustArchiNET/ArchiSteamFarm/release/6.1.4.3)

### Inferences
- A directly-analogous approach for 3urobeat's bot: expose Fastify/Express inside the existing Node process, serve a Vue/React dashboard, let users open `http://localhost:PORT` in their browser. Zero desktop-framework cost, no code signing, no SmartScreen, no notarization.
- Downsides vs packaged desktop app: no tray icon, no native autostart, no native OS notifications, firewall/permission prompt on Windows for listening sockets, harder "double-click to install" UX for non-technical users.

### Gaps
- No internal component map of ASF-ui (routes/stores/API shape) in search results.

---

## Native (WPF / WinUI / Qt / Avalonia) with Node sidecar

### Takeaway
Native toolkits (Avalonia for cross-platform .NET, WPF for Windows-only) give the best per-window memory and the smoothest OS integration (tray, autostart, notifications), but require a full UI rewrite in C#/C++/XAML and still need the Node sidecar. The clearest Steam-ecosystem exemplar is Watt Toolkit (Steam++) on Avalonia.

### Cited Findings
- Watt Toolkit / Steam++ is C# + Avalonia, GPL-3.0, cross-platform Steam toolbox — [SteamTools README](https://redirect.github.com/BeyondDimension/SteamTools/blob/develop/README.en.md)
- Watt Toolkit features local reverse proxy via YARP.ReverseProxy, account switching, local authenticator — [SteamTools README](https://redirect.github.com/BeyondDimension/SteamTools/blob/develop/README.en.md)
- Avalonia 11.x/12.x is on NuGet and used widely for cross-platform .NET desktop — [Avalonia nuget](https://www.nuget.org/packages/avalonia)
- Longoon12000/SteamAccountManager: WPF/.NET switcher for Steam accounts — [github.com/Longoon12000/SteamAccountManager](https://github.com/Longoon12000/SteamAccountManager)
- rex706/SAM: C# switcher, 1-click Steam login + 2FA — [SAM ecosyste.ms](https://awesome.ecosyste.ms/projects/github.com%2Frex706%2Fsam)
- luminary-cloud/steam-account-manager: native C++ Windows app with encrypted vault and Steam Guard — [ecosyste.ms](https://awesome.ecosyste.ms/projects/github.com%2Fluminary-cloud%2Fsteam-account-manager)
- dr1n/steamdesktopauthenticatorwpf: WPF port of SDA — [ecosyste.ms](https://awesome.ecosyste.ms/projects/github.com%2Fdr1n%2Fsteamdesktopauthenticatorwpf)
- C0Newb/SteamAuthenticator: WPF (vs WinForms) fork with add-account-by-login or maFile import — [github.com/C0Newb/SteamAuthenticator](https://github.com/C0Newb/SteamAuthenticator)

### Inferences
- Most prior-art in the Steam account-tool space is .NET, not Node-based — a Node project is somewhat the odd one out, which reinforces that *keeping* Node and adding a thin GUI is lower-risk than porting to Avalonia.
- Watt Toolkit is the closest apples-to-apples large-feature Steam GUI; its Avalonia + YARP architecture is a good reference but reusing it would mean rewriting the bot in C# (via SteamKit2 instead of steam-user).

### Gaps
- No bundle-size or memory numbers for Watt Toolkit specifically.

---

## Code-signing (Windows SmartScreen + macOS notarization) in 2026

### Takeaway
EV certificates no longer grant instant SmartScreen reputation in 2026 — Microsoft explicitly lists EV as "no longer recommended for SmartScreen" and new EV-signed apps must build reputation organically, same as OV. macOS still requires signing + notarization for distribution, gated by the $99/yr Apple Developer Program. Azure Trusted Signing / Artifact Signing (~$10/mo) is now the cheapest legitimate Windows signing path.

### Cited Findings
- Microsoft docs now say "EV certificates — no longer recommended for SmartScreen" — [Microsoft code-signing-options](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options)
- EV-signed apps must now build SmartScreen reputation organically — [ToDesktop blog](https://www.todesktop.com/blog/posts/windows-apps-psa-ev-certs-do-not-grant-immediate-reputation-anymore)
- OV certificates ~$150–300/yr; EV ~$400+/yr; Sectigo EV $290–$500 — [sslinsights](https://sslinsights.com/best-code-signing-certificate-for-windows-application/)
- Azure Artifact Signing / Trusted Signing priced ~$9.99/mo — [Microsoft code-signing-options](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options)
- EV still required for Windows kernel-mode drivers / WHQL — [sslinsights](https://sslinsights.com/best-code-signing-certificate-for-windows-application/)
- macOS Developer Program is $99/yr (covers notarization at no extra charge) — [flaviocopes: should you pay for Apple Developer Program](https://flaviocopes.com/apple-developer-free-vs-paid/)
- Homebrew cask (since Sep 1, 2026) disables casks for apps that fail Gatekeeper → effectively requires Developer ID signing + notarization — [codenote 2026 code-signing](https://codenote.net/en/posts/windows-desktop-app-code-signing-distribution-japan-2026/)
- Doyensec 2026 warning: macOS auto-update via Squirrel.Mac *requires* notarization (notarization is a prerequisite for a working macOS auto-update) — [Doyensec](https://blog.doyensec.com/2026/02/16/electron-safe-updater.html)

### Inferences
- Total 2026 signing budget for cross-platform: ~$120/yr Azure Trusted Signing + $99/yr Apple Developer = ~$220/yr, down from ~$500/yr EV + $99 Apple a few years ago. No reason to buy EV unless you're shipping drivers.
- Both Electron and Tauri auto-updaters depend on signing being correctly set up; the framework choice doesn't change this cost.

### Gaps
- No vendor-confirmed Azure Trusted Signing SLA/throughput numbers surfaced.

---

## Multi-window / multi-account visualization patterns observed in the Steam-tool ecosystem

### Takeaway
Real-world Steam tools converge on one of three UI patterns: (1) ASF-ui-style *single-page dashboard* with account cards + log panel served from a REST+WebSocket backend; (2) *tray-driven switcher* with modal account list (SAM, Steam Account Manager); (3) *tabbed native shell* with per-account panes (Watt Toolkit). None use per-account BrowserWindow / native window, which matches the memory intuition above.

### Cited Findings
- ASF-ui is a Vue SPA consuming ASF's REST IPC — [ASF IPC wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/IPC)
- Watt Toolkit groups features (account switch, authenticator, network accel) in a single Avalonia window — [SteamTools README](https://redirect.github.com/BeyondDimension/SteamTools/blob/develop/README.en.md)
- SAM / Longoon12000 SteamAccountManager: list-of-accounts + "two-click switch" UX — [Longoon12000/SteamAccountManager](https://github.com/Longoon12000/SteamAccountManager)
- 3urobeat/steam-comment-service-bot itself supports hundreds of accounts, uses a `relogQueue` and session handler module — [release notes 2.12.1](https://newreleases.io/project/github/3urobeat/steam-comment-service-bot/release/2.12.1)
- xterm.js is the standard live-log component in Electron apps; attach-addon streams over WebSocket, or node-pty + IPC inside the same process — [xterm.js README](https://unpkg.com/@xterm/xterm@6.0.0/README.md)

### Inferences
- For a ~100-account bot, one dashboard window with virtualized account rows + a log panel (xterm.js or a plain terminal view) is the right shape; separate windows per account would waste both memory and screen real estate.
- A live log of all accounts probably wants tagged/colored lines in one terminal plus per-account filter, rather than one xterm per account.

### Gaps
- No screenshots or UX writeups of ASF-ui's exact log component were retrieved — "uses the ASF REST API" is as deep as the sources went.

---

## Tray icon, autostart, run-at-boot, silent-background on Windows/macOS/Linux

### Takeaway
Electron's Tray API covers all three OSes with documented platform quirks; autostart is not provided by Electron's core API and must be implemented via platform-specific mechanisms (Windows registry/AppData Startup, macOS LoginItems / `app.setLoginItemSettings`, Linux `.desktop` file in `~/.config/autostart`). Tauri exposes similar primitives through its tray and autostart plugins.

### Cited Findings
- Electron `Tray` class takes a NativeImage or icon path; menu-bar extras on macOS, notification area on Windows — [Electron Tray docs](https://electronjs.org/docs/latest/api/tray)
- macOS tray icons should be template images — [Electron Tray docs](https://electronjs.org/docs/latest/api/tray)
- Linux tray uses StatusNotifierItem, falls back to GtkStatusIcon — [Electron Tray docs](https://electronjs.org/docs/latest/api/tray)
- macOS left-click opens context menu by default; Windows/Linux right-click — [Electron Tray docs](https://electronjs.org/docs/latest/api/tray)
- Example autostart (Trilium Notes, Linux): writes `trilium.desktop` to `$XDG_CONFIG_HOME/autostart` — [Trilium setup docs](https://docs.triliumnotes.org/user-guide/setup/desktop/tray-icon-and-startup)

### Inferences
- A silent-background bot is a natural fit for tray: minimize-to-tray on close, start-hidden on boot, right-click → "Pause all accounts" / "Open dashboard."
- Linux cross-desktop reliability remains the weakest link: StatusNotifierItem support varies by desktop environment (GNOME needs extension, KDE/XFCE OK).

### Gaps
- No tested, cited recipe for Windows login-item registration from Electron — `app.setLoginItemSettings` is the Electron API but didn't surface in results.

---

## Summary matrix (inferred, cross-referenced)

| Framework | Idle bundle | Idle RAM (1 window) | Keeps Node? | Steam prior art | Auto-update |
|---|---|---|---|---|---|
| Electron | ~85–270 MB | ~128–168 MB | Yes, natively | ASF-ui (via REST), plenty | electron-updater (NSIS + Squirrel.Mac) |
| Tauri 2 | ~1–15 MB shell + ~150 MB Node payload | ~42–109 MB | Only via sidecar | Samira, Depo Tool (small) | Built-in, Ed25519-signed |
| Neutralino | ~2–8 MB shell + payload | low (OS webview) | Only via child process | None found | Custom |
| WebView2-host | small shell + payload | low | Only via child process | None named | Custom |
| Wails (Go) | small shell + payload | low | Only via child process | None found | Custom |
| Avalonia / native | 20–100 MB | ~30–80 MB | No (would rewrite in C#/SteamKit2) | Watt Toolkit, SAM, SDA-WPF | Custom / Velopack / Squirrel |
| ASF-ui style (local web server) | 0 extra | 0 extra | Yes | ASF itself | Follows CLI release |
