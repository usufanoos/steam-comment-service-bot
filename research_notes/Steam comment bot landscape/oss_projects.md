# Steam comment / view / like / follow bot OSS landscape

## Major OSS Steam engagement-bot repos beyond 3urobeat/steam-comment-service-bot

### Takeaway
3urobeat/steam-comment-service-bot is the clear leader for the comment/like/fav/follow use case. Most other active Node.js and Python repos in the surrounding "mass-account Steam bot" space target card farming, hour boosting, or trading rather than profile engagement; only a small handful target comments or reputation specifically. ArchiSteamFarm dominates the farming side but does not do profile engagement.

### Cited Findings

Projects discovered on github.com/topics/steam-bot (20 of 57 repos visible on the first page) with stars and last-push dates as reported:

- **3urobeat/steam-comment-service-bot** — Node.js, GPL-3.0, 358 stars; "the most advanced Steam Account Manager Bot" with mass coordinated comments/likes/favs/follows and plugin support; terminal + Steam chat only — [Source](https://github.com/3urobeat/steam-comment-service-bot)
- **zevnda/steam-game-idler** — TypeScript, Elastic-2.0, 801 stars, last update Nov 17 2026; card farming, achievement unlocker, playtime booster; desktop app, repo contains `src-tauri` folder (Tauri) and `next.config.ts` (Next.js frontend) — [Source](https://github.com/zevnda/steam-game-idler); listing — [Source](https://github.com/topics/steam-bot)
- **3urobeat/steam-idler** — JavaScript, 214 stars, last push Oct 18 2026; headless cross-platform idler with multi-account support; "slimmed-down version of steam-comment-service-bot" — [Source](https://github.com/topics/steam-bot)
- **Poltergeistlunagrid/Steam-Idler** — Python, 119 stars, last push Nov 7 2026; idles games for card farming and playtime boosting across multiple accounts — [Source](https://github.com/topics/steam-bot)
- **3urobeat/steam-mass-comment-bot** — JavaScript, 101 stars, last push Oct 18 2026; posts comments on many profiles/groups/discussions from ONE account (inverse of steam-comment-service-bot) — [Source](https://github.com/topics/steam-bot)
- **tacheometry/steam-hour-farmer** — JavaScript, 75 stars, last push Feb 13 2026; headless single-account hour farmer, also an npm package — [Source](https://github.com/topics/steam-bot)
- **prasunroy/steam-bot** — JavaScript, 38 stars, last push Apr 5 2026; automated Steam trading bot — [Source](https://github.com/topics/steam-bot)
- **Grimemperorcove/tf2-resp-express** — HTML, 28 stars, last push Nov 9 2026; automated TF2 trading bot with pricing and a GUI — [Source](https://github.com/topics/steam-bot)
- **typekcz/beepboop-steam** — JavaScript, 19 stars, last push Oct 17 2026; plays sounds in Steam chat — [Source](https://github.com/topics/steam-bot)
- **Lemon4ksan/g-man** — Go, 14 stars, last push Nov 1 2026; Go SDK and automation framework for Steam — [Source](https://github.com/topics/steam-bot)
- **ill-yes/steam-bee** — TypeScript, AGPL-3.0, 12 stars, last push Nov 18 2026; self-hosted hour booster with a Docker-friendly single-user React/Fastify web UI; QR login, scheduling, game library management — [Source](https://github.com/ill-yes/steam-bee)
- **Refloow/Steam-Vanity-URL-Swap** — JavaScript, 10 stars, last push Jul 22 2026; swaps vanity URLs between two accounts — [Source](https://github.com/topics/steam-bot)
- **TcNobo/TcNo-Mass-Steam-Idler** — C#, last push June 2024; cycles through app IDs and idles each for a set time; "auto-manager mode activates 50 new keys, idles each game for 5 seconds, and restarts every hour" — [Source](https://awesome.ecosyste.ms/projects/github.com%2FTCNOco%2FTcNo-Mass-Steam-Idler)
- **CSTraders/bot-idle** — takes accounts from a directory and idles app IDs listed in configuration — [Source](https://github.com/CSTraders/bot-idle)
- **Refloow/steam-games-idle** — Node.js hour booster, last push January 2025 — [Source](https://awesome.ecosyste.ms/projects/github.com%2Frefloow%2Fsteam-games-idle)
- **Marc3842h/Titan** — C# (.NET/Mono), MIT, 202 stars; CS:GO report & commendation bot (not profile comments); has GUI + CLI; "archived Apr 9 2026, unmaintained and provided as-is"; Linux deps hint at GTK — [Source](https://github.com/Marc3842h/Titan)
- **czQery/steam-comment-bot** — Python, 5 stars, last update May 28 2021 (STALE); automatically posts a comment and reposts it if deleted — [Source](https://github.com/topics/steam-comment-bot)
- **rep4rep/rep4rep-bot** — companion bot for the rep4rep.com reputation-exchange service; URL at github.com/rep4rep/rep4rep-bot returns 404, so account/repo likely deleted or moved (confirmed via WebFetch) — [Source](https://github.com/rep4rep/rep4rep-bot)
- **hschickdevs/steam-comment-bot** — separate project, only surfaced via search snippets, not verified — [Source](https://www.github.com/topics/steam-comment-bot)
- **ares0027/SteamBot, Top-Cat/SteamBot, tadev/SteamBot, hyt47/SteamBot** — older C# SteamBot family (trading-focused), surfaced via search but not verified currentness — [Source](https://github.com/tadev/SteamBot)

ArchiSteamFarm (JustArchiNET/ArchiSteamFarm): C#, Apache-2.0, 13.8k stars, 14,639 commits; CLI with separate ASF-ui web frontend — [Source](https://github.com/JustArchiNET/ArchiSteamFarm)

### Inferences
- Any repo last touched before 2023 is essentially dead for this ecosystem because Steam's auth flow moved to refresh-token-based login in late 2023; node-steam-user v5 and steam-session were required to keep working. czQery/steam-comment-bot (2021), user28111999/steam-booster (Jan 2021), SloRunner/node-steamidle (Dec 2022), tskxz/steam-farm (Apr 2021), annurdien/steam-idler (Mar 2023) are all likely broken against current Steam.

### Gaps
- GitHub topic page `steam-bot` lists 57 repos but only 20 were visible in the fetch; no "Load more" was expanded.
- License column was blank for every topic-page row; licenses only confirmed where the repo page was fetched directly.
- Last-commit dates for topic-page entries come from the page's "updated" column, which reflects any repo activity (including issue updates), not strictly last commit.

---

## Which projects have desktop GUI frontends vs terminal-only

### Takeaway
Nearly all mass-account Steam engagement bots are CLI/terminal-only. The only confirmed desktop-GUI application in the broader space is zevnda/steam-game-idler (Tauri + Next.js) which does card farming/achievements, not comments. The leading comment bot (3urobeat) is strictly terminal + Steam chat; a Vue/Vite web frontend is being built separately by DerDeathraven but is not a desktop app.

### Cited Findings

- **3urobeat/steam-comment-service-bot**: "accounts can be controlled from one terminal and Steam Chat"; no GUI mentioned — [Source](https://github.com/3urobeat/steam-comment-service-bot)
- **DerDeathraven/steam-comment-bot-rest-api**: TypeScript plugin, 2 stars; RPC-style API at `localhost:4000/rpc/${Class}.${MethodName}`; "An official frontend is currently under development" — [Source](https://github.com/DerDeathraven/steam-comment-bot-rest-api)
- **DerDeathraven/steam-comment-service-bot-frontend**: Vue + Tailwind + Pinia + Phosphor Icons + Vite; "served from the official rest-api plugin" runs at `localhost:5123`; no Electron or Tauri files — i.e. web GUI, not desktop — [Source](https://github.com/DerDeathraven/steam-comment-service-bot-frontend)
- **zevnda/steam-game-idler**: Windows installer/portable + Linux; `src-tauri` folder (Tauri) + `next.config.ts` (Next.js) — only confirmed desktop-native GUI in this space; does NOT do comments — [Source](https://github.com/zevnda/steam-game-idler)
- **ill-yes/steam-bee**: React web UI served by Fastify in production, Vite in dev; "Docker-friendly, single-user interface"; not desktop — [Source](https://github.com/ill-yes/steam-bee)
- **Marc3842h/Titan**: C# CS:GO report bot, has both GUI and CLI mode; Linux deps `Gtk 3` and `libAppindicator 3` imply GTK — but repo is archived/unmaintained — [Source](https://github.com/Marc3842h/Titan)
- **ArchiSteamFarm**: CLI-tagged (`cli`, `cli-app`); has a separate `ASF-ui` web frontend in the repo; no native desktop GUI — [Source](https://github.com/JustArchiNET/ArchiSteamFarm)
- **node-steam-bot-manager**: "GUI-based option" reported in search snippets, framework unverified — [Source](https://github.com/sherpic/node-steam-bot-manager)
- **Grimemperorcove/tf2-resp-express** (HTML, 28 stars): "automated TF2 trading bot with pricing and a GUI" — likely web GUI (HTML as primary language) — [Source](https://github.com/topics/steam-bot)

### Inferences
- There is no Electron, Tauri, WPF, Qt, WinForms, or Flutter-desktop fork of 3urobeat/steam-comment-service-bot — a desktop rewrite would be an uncontested slot in the comment-bot niche.
- The current GUI trajectory for this bot is web-only (DerDeathraven's Vue frontend served by REST API plugin), so wrapping that in Electron/Tauri is the shortest path to a desktop app.

### Gaps
- Could not confirm framework of `node-steam-bot-manager`'s GUI option.
- GitHub's /forks tab for 3urobeat/steam-comment-service-bot was not enumerated; a GUI fork could exist but is not surfaced by search.

---

## Dominant underlying libraries

### Takeaway
The Node.js engagement-bot ecosystem is almost entirely built on DoctorMcKay's `steam-user` + `steamcommunity` + `steam-session` trio. The .NET world runs on SteamRE/SteamKit2. Python has the ValvePython/steam package and steam.py/steamio, mainly used for trading. Go-steam is a SteamKit2 port with modest adoption.

### Cited Findings

- **DoctorMcKay/node-steam-user**: MIT, 1.1k stars, 191 forks; Node ≥14 required; supports anonymous logon, refresh tokens (recommended), name+password with Steam Guard; `SteamChatRoomClient` for chat — [Source](https://github.com/DoctorMcKay/node-steam-user)
- **3urobeat/steam-comment-service-bot package.json** confirms the "McKay stack": `steam-user ^5.3.0`, `steamcommunity ^3.50.0`, `steam-session ^1.9.4`, `steamid ^2.1.0`, `steamid-resolver ^1.4.0`, `@seald-io/nedb ^4.1.2` for storage, `cheerio`/`htmlparser2` for HTML scraping, `request`, `qrcode` (for QR login), `output-logger` — [Source](https://github.com/3urobeat/steam-comment-service-bot/blob/master/package.json)
- Note: steam-comment-service-bot's package.json has no `engines` field (no declared Node version floor despite node-steam-user requiring Node ≥14) — [Source](https://github.com/3urobeat/steam-comment-service-bot/blob/master/package.json)
- **SteamRE/SteamKit2**: .NET library for the Steam network; LGPL-2.1; needs .NET 8.0+ at runtime — [Source](https://rust-digger.code-maven.com/src/steam-language-gen-0.1.4/assets/SteamKit/readme.md)
- **ArchiSteamFarm** is built on SteamKit2 — [Source](https://github.com/JustArchiNET/ArchiSteamFarm)
- **Philipp15b/go-steam**: Go port of SteamKit2, New BSD license, ~363 stars, last sync April 2024 (per aggregator); APIs for friends, chatting, trading, trade offers, TF2 crafting — [Source](https://awesome.ecosyste.ms/projects/74355)
- **ValvePython/steam** ("python-steam"): SteamClient with login and game-status; general-purpose — [Source](https://awesome.ecosyste.ms/lists/scholtzm%2Fawesome-steam?topic=code)
- **steam.py / steamio**: async Python wrapper, includes a commands.Bot extension — [Source](https://pypi.org/project/steamio/0.9.9)
- **steampy**: trade-offer-focused Python library with SteamGuard — [Source](https://awesome.ecosyste.ms/lists/scholtzm%2Fawesome-steam?topic=code)
- 3urobeat's bot migrated to token-based login in v2.12.1: "Added support for Steam's new token based login flow with a new sessionHandler module powered by node-steam-session" — [Source](https://newreleases.io/project/github/3urobeat/steam-comment-service-bot/release/2.12.1)

### Inferences
- A desktop rewrite that stays in the Node ecosystem keeps access to the McKay libraries, which are the only libraries with first-class profile-comment/like/follow HTTP endpoints wrapped (via steamcommunity). A Rust/Tauri rewrite would need to either embed Node (via sidecar) or re-implement the community-page HTTP calls, since SteamKit2 and go-steam target the binary protocol and don't cover the HTML community endpoints.
- Tauri+sidecar-Node is the lowest-risk desktop path for preserving steam-comment-service-bot's functionality.

### Gaps
- No direct feature-by-feature comparison of Python libraries was obtained from primary sources.

---

## Mass-account management, chat command interfaces, plugin systems

### Takeaway
The dominant pattern (3urobeat) is: config file lists many accounts, a single "main" account acts as the control surface over Steam Chat, with a plugin system that discovers npm packages by name prefix. ArchiSteamFarm uses a similar plugin architecture plus a web UI and IPC. Other projects mostly skip plugin systems and just run a batch of idlers.

### Cited Findings

- **steam-comment-service-bot**: "text a bot network" and request comments "directly from the Steam Chat"; plugin system via `plugins` folder; plugin loader "dynamically loads installed npm packages with a specific prefix" (`steam-comment-bot-`); official plugins include steam-comment-bot-discord-plugin, steam-comment-bot-rest, steam-comment-bot-webserver (all listed in package.json dependencies) — [Source](https://github.com/3urobeat/steam-comment-service-bot); plugin docs — [Source](https://github.com/3urobeat/steam-comment-service-bot/blob/beta-testing/docs/wiki/creating_plugins.md)
- Storage: `@seald-io/nedb` (embedded JSON database) for persistence — [Source](https://github.com/3urobeat/steam-comment-service-bot/blob/master/package.json)
- Features also include: group advertising with automatic invites, proxy support, cooldowns, multi-language support — [Source](https://github.com/3urobeat/steam-comment-service-bot)
- **ArchiSteamFarm** plugin system: "Special plugin system, which allows you to extend ASF in any way you wish through your own code"; also includes mobile authenticator, chat logger, key/gift redemption, trade acceptance/sending, messaging — [Source](https://github.com/JustArchiNET/ArchiSteamFarm)
- **TcNo-Mass-Steam-Idler** auto-manager: activates 50 new keys, idles each game 5 seconds, restarts hourly — [Source](https://awesome.ecosyste.ms/projects/github.com%2FTCNOco%2FTcNo-Mass-Steam-Idler)
- **steam-bee**: QR login, session controls, scheduling, game library management as a single-user operations console — [Source](https://github.com/ill-yes/steam-bee)
- **DerDeathraven/steam-comment-bot-rest-api**: RPC-style endpoints `localhost:4000/rpc/${Class}.${MethodName}`; manages bot accounts, settings, comment queues; headless mode available — [Source](https://github.com/DerDeathraven/steam-comment-bot-rest-api)

### Inferences
- 3urobeat's "npm-prefix plugin discovery" is unusual for a desktop app — a rewritten Electron/Tauri version would want either (a) in-app plugin store that still installs npm packages, or (b) shift to a bundled, signed plugin manifest format.

### Gaps
- No detail recovered on exactly which chat commands 3urobeat's bot exposes (!comment, !like etc.) from primary sources during this pass — the README structure was summarised not quoted.

---

## Projects doing views/likes/favs/follows beyond comments

### Takeaway
Only 3urobeat/steam-comment-service-bot explicitly advertises "mass coordinated comments/likes/favs/follows" as a single integrated feature. Other active repos split into: idling (cards/playtime), trading, CSGO reports (Titan), and pure single-account spammers. No active repo surfaced in this research targets likes/favs/follows independently.

### Cited Findings
- 3urobeat/steam-comment-service-bot README description: "mass coordinated comments/likes/favs/follows" and plugin support — [Source](https://github.com/3urobeat/steam-comment-service-bot)
- Topic page listing: "Like, follow, favorite: No listed repo clearly targets these actions" (across the 20 visible `steam-bot` topic repos) — [Source](https://github.com/topics/steam-bot)
- zevnda/steam-game-idler: "No in-app comment feature is listed" — [Source](https://github.com/zevnda/steam-game-idler)

### Gaps
- Likes on Steam artwork/screenshots/guides and follows on users/curators are reachable through the `steamcommunity` HTML endpoints the McKay library wraps; whether any smaller tool exposes them without also doing comments was not confirmed.

---

## Notable forks adding a GUI to 3urobeat specifically

### Takeaway
No GUI fork of 3urobeat/steam-comment-service-bot was found. The only GUI effort is non-fork: DerDeathraven's two-repo pattern (REST API plugin + Vue frontend), which is a companion web GUI, not a desktop fork, and the frontend has only 1 star with 14 commits.

### Cited Findings
- DerDeathraven frontend repo: Vue/Vite/Pinia/Tailwind, 1 star, 14 commits, no Electron or Tauri files — [Source](https://github.com/DerDeathraven/steam-comment-service-bot-frontend)
- DerDeathraven REST API plugin: 2 stars, "frontend currently under development"; 66 commits — [Source](https://github.com/DerDeathraven/steam-comment-bot-rest-api)
- steam-comment-bot-webserver is already listed as a dependency in the main bot's package.json — [Source](https://github.com/3urobeat/steam-comment-service-bot/blob/master/package.json)

### Gaps
- 3urobeat's forks tab was not enumerated via the GitHub API; a stealth GUI fork could exist that doesn't show up in topic/search listings.

---

## ArchiSteamFarm position relative to engagement bots

### Takeaway
ArchiSteamFarm is the largest Steam-bot project (13.8k stars) but is scoped to card farming, trading, and authenticator functionality. It does NOT advertise commenting, liking, following, or favoriting as features, so it is a complementary tool rather than a competitor to steam-comment-service-bot.

### Cited Findings
- ASF feature list: automatic card farming across many accounts, offline farming, no Steam client required, key/gift redemption, trade acceptance/sending, messaging, SteamGuard/2FA, mobile authenticator, trade integration to complete sets, chat logging; cross-platform (Windows/Linux/macOS) — [Source](https://github.com/JustArchiNET/ArchiSteamFarm)
- ASF page "doesn't mention commenting, liking, following, or favoriting workshop items" — [Source](https://github.com/JustArchiNET/ArchiSteamFarm) (via WebFetch extraction)
- ASF farming performance factor: "the only thing which heavily affects speed of farming is selected cards farming algorithm" — [Source](https://sourceforge.net/projects/archisteamfarm.mirror/)
- ASF has a plugin system — [Source](https://github.com/JustArchiNET/ArchiSteamFarm)
- Separate active ASF, per Docker Hub listing "updated about 16 hours ago" and third-party pointer to version 6.3.4.2 — [Source](https://hub.docker.com/r/justarchi/archisteamfarm)
- Contradicting signal: one dev.to post calls ASF "abandoned (last update: 2016)" — this is contradicted by the Docker Hub update time and should be treated as promotional for the author's own alternative — [Source](https://dev.to/zevnda/how-i-built-an-open-source-alternative-to-archisteamfarm-and-why-you-should-care-37od)

### Inferences
- ASF's plugin system is the most obvious channel if someone ever wanted to add comment/like/follow features to it, but no such plugin surfaced in the search.

### Gaps
- ASF plugin registry was not inspected for community comment/like/follow plugins.

---

## Reputation / trust signals (malware, stealers, legit tools)

### Takeaway
No specific Steam-comment-bot repo surfaced as malware in this research pass. The relevant risk pattern is well-documented: fake GitHub repositories and comment spam deliver infostealers that target Steam session tokens among other credentials. This is background risk for any user downloading random "steam bot" binaries from unknown repos, but doesn't flag any of the specific projects above.

### Cited Findings
- Arctic Wolf found 292 fake GitHub repositories impersonating legitimate software; many redirect to fake download pages delivering infostealers; some stealer payloads reportedly include Steam session tokens — [Source](https://www.bleepingcomputer.com/news/security/nearly-300-github-repos-pose-as-legit-software-to-push-malware/amp/)
- Lumma Stealer was spread via fake "fixes" in GitHub comments; researcher counted >29,000 such comments over three days — [Source](https://www.techradar.com/pro/security/lumma-stealer-malware-linked-as-project-fixes-in-github-comments)
- Socket report covers automated spam-comment bots on GitHub — [Source](https://socket.dev/blog/github-users-targeted-by-new-wave-of-spambots-promoting-malicious-downloads)
- Steam Workshop was also abused to spread malware via Wallpaper Engine (separate vector) — [Source](https://www.hackread.com/github-comment-malware-fake-microsoft-repositories/)

### Inferences
- The specific projects in the inventory above (3urobeat, zevnda, ill-yes, DerDeathraven, ArchiSteamFarm, SteamKit2, node-steam-user) are maintained under identifiable author handles with sustained commit history, which is itself the primary trust signal distinguishing them from the "fake repo" pattern. Users should still verify authorship, release signing, and build provenance.

### Gaps
- No Reddit r/SteamBots or UnknownCheats/MPGH threads were retrieved in this pass; subjective reputation signals among users are not covered.
- No scan of npm advisories or GitHub Security Advisories for the `steam-comment-bot-*` npm prefix.
