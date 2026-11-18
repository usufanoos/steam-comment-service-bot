# Steam Authentication and Confirmation Layer for Comment/Like/Fav/Follow Bots

## How does the modern Steam login flow work (post-2023 steam-session / EAuthTokenPlatformType / refresh tokens / access tokens)?

### Takeaway
Steam's 2023 overhaul replaced the legacy `ISteamUserAuth/RSA` + password + captcha login with the gRPC-style `IAuthenticationService` interface: clients begin an auth session, submit a Steam Guard factor, poll for a refresh token, and then exchange that long-lived refresh token for short-lived access tokens / web cookies. Every modern bot (node-steam-session, SteamKit2, python-steam, steamguard-cli) is a wrapper around this flow.

### Cited Findings
- `IAuthenticationService/BeginAuthSessionViaCredentials/v1` is the entry endpoint; the password field is RSA-encrypted client-side, with the key obtained from `GetPasswordRSAPublicKey` — [steamapi.xpaw.me via WebSearch summary](https://steamapi.xpaw.me/IAuthenticationService)
- `node-steam-session` exposes three `EAuthTokenPlatformType` values: `SteamClient` (audiences `['web','client']`, uses a WebSocket CM transport), `WebBrowser` (audience `['web']`), and `MobileApp` (audiences `['web','mobile']`) — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)
- Login entry points in the library are `startWithCredentials({accountName,password,persistence,steamGuardMachineToken,steamGuardCode})` and `startWithQR()` which returns a `qrChallengeUrl` and polls (treated as a device confirmation) — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)
- After auth, `refreshAccessToken()` returns a new access token and `renewRefreshToken()` may also rotate the refresh token (the old one is invalidated on success) — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session); maintainer reply states a CM WebSocket connection is required for a renew — [dev.doctormckay.com topic 5687](https://dev.doctormckay.com/topic/5687-refreshaccesstoken-not-working-on-eauthtokenplatformtypesteamclient/)
- `getWebCookies()` on the login session hands back `steamLoginSecure` cookies for `steamcommunity.com` / `store.steampowered.com`; access tokens can be passed as `?access_token=` to some `api.steampowered.com` endpoints — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)
- `renewRefreshToken` only works for `MobileApp` and `SteamClient` platform types; `WebBrowser` returns `AccessDenied` — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)
- Refresh-token lifetime is community-reported as ~200 days, not Valve-documented — [dev.doctormckay.com topic 4228 (summary)](https://dev.doctormckay.com/topic/4228-login-via-refresh-token)
- SteamKit2's Authentication sample uses `client.Authentication.BeginAuthSessionViaCredentialsAsync(AuthSessionDetails { Username, Password, IsPersistentSession, GuardData })`; a parallel `QrCodeAuthentication` sample exists — [SteamKit sample via rust-digger mirror](https://rust-digger.code-maven.com/src/steam-language-gen-0.1.4/assets/SteamKit/Samples/1a.Authentication/Program.cs)

### Inferences
- The full observed sequence is: `GetPasswordRSAPublicKey` → `BeginAuthSessionViaCredentials` (encrypted password) → user supplies factor via `UpdateAuthSessionWithSteamGuardCode` (TOTP) or `UpdateAuthSessionWithMobileConfirmation` (approve on phone) → `PollAuthSessionStatus` returns `refresh_token` and `access_token` → optional `GenerateAccessTokenForApp` to re-mint an access token from the refresh token. Names are consistent across SteamKit, node-steam-session, and xpaw's enumeration.
- The legacy `ISteamUserAuth/RSA`/`DoLogin` captcha flow is effectively retired for bots: current libraries exclusively use `IAuthenticationService`, and third-party clients that hit the new endpoint report that "a captcha aborts the function" rather than embedding a solvable image (session falls through to Steam Guard) — [search summary of SteamKit2 third-party client](https://pkg.go.dev/github.com/escrow-tf/steam/api/auth)

### Gaps
- Exact proto field numbers and the full `EAuthSessionGuardType` enum (only four values confirmed below) — attempts to fetch `cdn.jsdelivr.net` and `steamapi.xpaw.me` were blocked by this session's network allowlist.
- Official refresh-token TTL from Valve.

---

## How does Steam Guard Mobile Authenticator work under the hood, and what is the comment/trade/market confirmation flow?

### Takeaway
Steam Guard Mobile is two separate HMAC-SHA1 secrets with completely different roles: `shared_secret` drives the 30-second TOTP login code, and `identity_secret` signs the per-request confirmation key submitted to `steamcommunity.com/mobileconf`. A third value, `device_id` (formatted as `android:<UUID v4>`), identifies the virtual mobile device on every confirmation request. Trades, market listings, comments-from-new-devices, API-key creation, and family-view all funnel through the same `mobileconf` endpoints.

### Cited Findings
- maFiles must contain base64-encoded `shared_secret` and `identity_secret`; ASF ignores any other fields; typical files also include `device_id`, `revocation_code`, `serial_number`, `server_time`, `token_gid`, `secret_1`, `status`, and an `otpauth://` URI — [ArchiSteamFarm Two-factor-authentication wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/Two-factor-authentication-fr-FR), [libr2fa mafile.rs on docs.rs](https://docs.rs/crate/libr2fa/latest/source/src/steam/mafile.rs)
- `device_id` is a UUID v4 prefixed with `android:` — [libr2fa steam_key.rs on docs.rs](https://docs.rs/crate/libr2fa/latest/source/src/steam/steam_key.rs)
- `shared_secret` generates TOTP login codes; `identity_secret` is "the secret key used for generating confirmation codes, not the one used for generating login codes" — [DoctorMcKay forum topic 1000](https://dev.doctormckay.com/topic/1000-shared-secret-and-identity-secret)
- Confirmation key `k` is HMAC-SHA1(identity_secret, 8-byte big-endian time || tag), base64-encoded; `tag` is "conf" (list), "details", "allow", or "cancel"; steam-totp crate truncates `tag` to 32 bytes — [docs.rs steam-totp-rs](https://docs.rs/steam-totp-rs/0.1.2/src/steam_totp/lib.rs.html), [DoctorMcKay CConfirmation wiki (via fetch)](https://github.com/DoctorMcKay/node-steamcommunity/wiki/CConfirmation)
- Request parameters to `GET https://steamcommunity.com/mobileconf/getlist` and `GET https://steamcommunity.com/mobileconf/ajaxop`: `p` (device_id), `a` (steamID 64), `k` (confirmation hash), `t` (unix time), `m` ("android"), `tag` — [docs.rs steamguard/confirmation.rs](https://docs.rs/steamguard/latest/src/steamguard/confirmation.rs.html), [steampy generate_confirmation_key](https://snyk.io/advisor/python/steampy/functions/steampy.guard.generate_confirmation_key)
- Libraries must use Steam's server time, not the local clock; wrong time surfaces as an "identity secret is wrong" error — [DoctorMcKay topic 4379](https://dev.doctormckay.com/topic/4379-time-error-when-using-respondtoconfirmation-method), [topic 4829](https://dev.doctormckay.com/topic/4829-error-identity-secret-upon-confirmation-of-trade/)
- Each `CConfirmation` object exposes `id`, `type`, `creator` (trade offer id or market listing id), `key`, `title`, `receiving`, `sending`, `time`, `timestamp`, `icon`; `respond(time,key,accept,callback)` with tag "allow"/"cancel" — [node-steamcommunity CConfirmation wiki](https://github.com/DoctorMcKay/node-steamcommunity/wiki/CConfirmation)

### Inferences
- The two-secret design is why possession of a dumped `shared_secret` alone is enough to automate login but not enough to confirm trades/market listings — many leaked "bot pack" maFiles are shared_secret-only and that's why the owning service can't be stolen outright.
- Comment posting itself does not go through `mobileconf` in the normal case — only the login does; `mobileconf` becomes relevant when Steam demands a mobile confirmation for a sensitive action or when the bot is acting as a trade/market bot. Supported by the fact that node-steamcommunity's comment methods don't take a confirmation key.

### Gaps
- Full list of confirmation `type` enum values (trade=2, market listing=3 are commonly cited; API key and family-view types are mentioned anecdotally but not cited here).

---

## How do bots store and reuse maFiles, and which tools generate them?

### Takeaway
The de-facto on-disk format is the Jessecar96 SDA JSON ("maFile"). Every serious bot can ingest it: ASF imports `*.maFile` from `config/`, steamguard-cli reads SDA's `maFiles/manifest.json` layout verbatim, and node-steam-totp / steampy take the raw `shared_secret` and `identity_secret` strings pulled out of one. Generation is done by SDA (Windows, unmaintained), steamguard-cli (cross-platform, actively maintained), WinAuth (legacy, "Allow copy" exports the JSON), or ASF's built-in `2fainit` / `2fafinalize` commands.

### Cited Findings
- SDA documents `maFiles/` as the on-disk directory and warns to back it up; encryption is optional but recommended; project is "no longer supported" — [github.com/Jessecar96/SteamDesktopAuthenticator](https://github.com/Jessecar96/SteamDesktopAuthenticator)
- ASF import paths: (a) copy SDA `steamID.maFile` from `maFiles/` unencrypted (must start `{` end `}`) and rename to `BotName.maFile` in `config/`; (b) paste WinAuth "Allow copy" JSON into an empty `BotName.maFile`; (c) write manual JSON; ASF deletes the file after import for security — [ArchiSteamFarm Two-factor-authentication wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/Two-factor-authentication-fr-FR)
- ASF's own authenticator creation flow: `2fainit` → `.maFile.PENDING` + SMS/email code → `2fafinalize <code>` → `.maFile.NEW`; separate joint/standalone paths; `2fafinalizedforce` skips the code check — [ArchiSteamFarm Two-factor-authentication wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/Two-factor-authentication-fr-FR)
- steamguard-cli reads SDA's `maFiles/manifest.json` at `~/.config/steamguard-cli/maFiles/` (Linux) or `%APPDATA%\steamguard-cli\maFiles\` (Windows), with encrypted storage and optional system-keyring passkey; it can QR-export to Bitwarden or KeepassXC format — [github.com/dyc3/steamguard-cli](https://github.com/dyc3/steamguard-cli)
- Aegis explicitly does not support encrypted .maFile import — [search summary via Aegis mirror](https://git.morozoff.pro/aiden/Aegis/wiki/commit/d89aa7b28bfa1ca82985eb0951cc48838f8e7546)

### Inferences
- A bot pack's "one maFile per account" convention is universal because every major library already accepts the SDA JSON without transformation — there is no competing format in the ecosystem.
- Losing the `revocation_code` (`R12345` style) locks the account out of ever rotating authenticators without Steam Support involvement, which is why panels that take raw maFiles also demand the revocation code or refuse to work if it's missing.

### Gaps
- Precise byte-level encryption scheme SDA uses for passphrase-locked maFiles — the README does not expose it.

---

## What is the current state of captcha on Steam login and comment post?

### Takeaway
Steam's post-2023 `IAuthenticationService` login does not expose a solvable image captcha to clients; when Steam decides the attempt is suspicious the server returns a Steam Guard challenge or an outright abort, not a captcha. A separate rate-limit / CAPTCHA layer exists in the browser login form and on some Store/community actions, but none of the current open-source Steam libraries (node-steam-session, SteamKit2, python-steam, steamguard-cli) ship captcha-solver integration. 2captcha/anti-captcha publish no Steam-specific task type.

### Cited Findings
- "A captcha aborts the function" when hitting `BeginAuthSessionViaCredentials` in a third-party Go client — [pkg.go.dev escrow-tf/steam](https://pkg.go.dev/github.com/escrow-tf/steam/api/auth)
- Troubleshooting guide reports Steam applies short-term rate limits after repeated CAPTCHA failures or rapid login attempts, and that changing networks clears it immediately, which indicates the block is per-IP — [online-tech-tips Steam CAPTCHA fix](https://www.online-tech-tips.com/steam-captcha-not-working-how-to-fix-it/)
- 2captcha and anti-captcha advertise reCAPTCHA / hCaptcha / image "normal captcha" task types; neither lists a Steam-specific endpoint — [scrapfly captcha roundup](https://scrapfly.io/blog/posts/best-captcha-solving-api), [github 2captcha-java](https://github.com/2captcha/2captcha-java)
- Steam Community thread cites unverified daily comment caps around 200–250/account — [steamcommunity.com discussion 3113648183775065525](https://steamcommunity.com/discussions/forum/1/3113648183775065525)

### Inferences
- Modern bots route around captcha by never using the browser login form: they go straight to `IAuthenticationService`, so the only captcha surface is the login RSA page, which is skipped. If Steam flags the attempt, the symptom is an `EResult` error, not a captcha image — so the practical "captcha strategy" today is proxy rotation + slower pacing, not solver services.
- 3urobeat/steam-comment-service-bot has no documented captcha integration in the public wiki pages reviewed; its only mitigations are `requestDelay`, `botaccountcooldown`, and HTTP proxies — [3urobeat config_doc.md](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/config_doc.md)

### Gaps
- Whether `steamcommunity.com/comment/.../post` ever returns an inline reCAPTCHA token requirement under heavy use — no source found; the 3urobeat wiki does not mention one.

---

## How does session invalidation / IP banning / rate limiting work practically?

### Takeaway
Steam enforces three distinct limits that bots must respect: a per-account limited/unlimited gate (<$5 spend), per-IP cooldowns that trigger HTTP 429 and generic errors on community endpoints, and per-action cooldowns (particularly commenting and voting). Observed thresholds are mostly folklore; the 3urobeat bot codifies the operational pattern via `requestDelay` (15 s default between actions) and a 10-minute per-account cooldown.

### Cited Findings
- Limited accounts (less than $5 USD spent) can only comment on friends and sharedfiles, favorite sharedfiles, and follow users; unlimited accounts can comment on non-friend profiles, in groups, vote, and add friends — [3urobeat steam_limitations wiki](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/steam_limitations.md)
- Most community actions, "especially commenting," require Steam Guard enabled — [3urobeat steam_limitations wiki](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/steam_limitations.md)
- `requestDelay` default 15000 ms between each comment/vote/fav/follow "to avoid a Steam cooldown"; `botaccountcooldown` default 10 minutes per account after a request; `requestCooldown` default 5 minutes per requesting user — [3urobeat config_doc.md](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/config_doc.md)
- 2.15.3 release notes: fix for "IP cooldown error penalty in comment requests stacking when using proxies"; 2.15.4: "aborting request on IP cooldown on all proxies to non comment request types" — [3urobeat 2.15.3 release notes](https://newreleases.io/project/github/3urobeat/steam-comment-service-bot/release/2.15.3), [2.15.4 release notes](https://newreleases.io/project/github/3urobeat/steam-comment-service-bot/release/2.15.4)
- Inventory endpoint `getUserInventoryContent` returns 429 after "maybe 20-30 times" per IP — [dev.doctormckay.com topic 4045](https://dev.doctormckay.com/topic/4045-getting-rate-limited-on-pc-while-bots-are-running-with-proxies/)
- Unverified community numbers: ~200–250 comments/day/account — [steamcommunity.com discussion 3113648183775065525](https://steamcommunity.com/discussions/forum/1/3113648183775065525)

### Inferences
- The 3urobeat bot's defaults (15 s between actions, 10 min cooldown per bot) are a direct operational mapping of Steam's per-account comment cooldown — it's the safest observed pacing to avoid a `TooManyRequests`-style soft-ban.
- An IP cooldown is a shared-by-all-accounts-on-that-IP penalty: proxies cannot speed up a single-account limit but do multiply total throughput across accounts.

### Gaps
- Precise 429 thresholds for `postComment`/`voteUp`/`followUser` from Valve — none found; community values are informal.

---

## How do refresh tokens get rotated and how do bots survive "logged in elsewhere"?

### Takeaway
node-steam-user treats refresh tokens as persistent credentials: on first login it emits a `refreshToken` event the caller is expected to persist, and after that logs in via the stored token. The v5 migration added auto-renewal before expiry. When Steam kicks the session with `EResult.LoggedInElsewhere`, the pattern is to catch the `loggedOff` event and reconnect; the old session is dead but the refresh token typically still works.

### Cited Findings
- `steam-user will internally fetch a refresh token, emit the refreshToken event, and then use that token for subsequent logins` — [github.com/DoctorMcKay/node-steam-user v4→v5 migration guide](https://github.com/DoctorMcKay/node-steam-user/wiki/v4-%E2%86%92-v5-migration-guide)
- v5 `renewRefreshTokens` option: "steam-user will automatically attempt to renew your refresh token before it expires" — [v4→v5 migration guide](https://github.com/DoctorMcKay/node-steam-user/wiki/v4-%E2%86%92-v5-migration-guide)
- `loggedOff` event fires with eresult typically `LoggedInElsewhere` when another session takes over — [github.com/DoctorMcKay/node-steam-user docs](https://socket.dev/npm/package/steam-user)
- Community pattern: catch `LoggedInElsewhere`, log "Attempting to re-login", and re-run the login call — [dev.doctormckay.com topic 4715](https://dev.doctormckay.com/topic/4715-error-alreadyloggedinelsewhere-after-update-to-v5-from-v4)
- Renewing requires an active CM WebSocket connection; `WebBrowser` platform type cannot renew — [dev.doctormckay.com topic 5687](https://dev.doctormckay.com/topic/5687-refreshaccesstoken-not-working-on-eauthtokenplatformtypesteamclient/), [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)
- `renewRefreshToken()` may rotate — successful renewal invalidates the previous refresh token even if not expired — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)

### Inferences
- A bot operator should not call `renewRefreshToken()` on every start — only when the stored token is near expiry — because a failed rotation leaves the account unloggable until a full credential flow is re-run.
- The practical "survive a kick" flow is: on `loggedOff(LoggedInElsewhere)` wait a backoff (observed implementations use 30 s – 5 min), then call `logOn({refreshToken})` again; Steam allows the bot to reclaim the session but may kick the human user back.

### Gaps
- Whether rotation on kick vs. rotation on schedule produces different lifetimes — not documented.

---

## What is the role of proxies in evading per-IP limits?

### Takeaway
Proxies are the single most important rate-limit tool for mass-account bots, because many Steam limits are per-IP, not per-account. The 3urobeat bot and similar panels distribute accounts across HTTP proxies from a file; SOCKS5 is not documented as a first-class input in that bot, though underlying SteamKit2 and node-steam-user both accept SOCKS5. Residential proxies are preferred by operators because datacenter ranges are flagged for login/community actions; this is folklore not a Valve-published policy.

### Cited Findings
- 3urobeat `proxies.txt` format: `http://user:pass@ip:port` (user/pass optional); configurable `proxyFormat` for other layouts like `${ip}:${port}:${user}:${pass}` — [3urobeat adding_proxies wiki](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/adding_proxies.md)
- Accounts are auto-distributed across proxies; `useLocalIP` controls whether host IP is also used — [3urobeat adding_proxies wiki](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/adding_proxies.md)
- SOCKS5 is not mentioned in that wiki page — [3urobeat adding_proxies wiki](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/adding_proxies.md)
- "The Steam client has no proxy fields, so SOCKS5 proxies have to be configured in the bot itself" — [cyberyozh Steam proxy guide](https://app.cyberyozh.com/guides/tasks/games/proxy-for-steam/)
- Inventory 429 after 20–30 same-IP requests confirms per-IP enforcement — [dev.doctormckay.com topic 4045](https://dev.doctormckay.com/topic/4045-getting-rate-limited-on-pc-while-bots-are-running-with-proxies/)

### Inferences
- Because community endpoints share the IP bucket across accounts, a panel running 50 accounts behind one proxy sees roughly the same per-IP cooldown as a single account — proxy count, not account count, is the operational throughput knob.
- 3urobeat's HTTP-only documentation reflects node-steamcommunity's HTTPS-proxy posture; SOCKS5 would need to be done via a global agent or `socks-proxy-agent`, which the public wiki does not cover.

### Gaps
- Direct evidence that residential beats datacenter for Steam Community — only vendor marketing pages make the claim.

---

## What is handled by SDKs vs. hand-rolled?

### Takeaway
The three "big library" ecosystems — node-steam-session + node-steam-user + node-steamcommunity (JS), SteamKit2 (.NET), python-steam/ValvePython — cover login (RSA, Guard code submission, QR, refresh-token persistence) and the CM connection out of the box. What remains hand-rolled in every bot: proxy pool management, confirmation polling scheduling, maFile ingestion, CAPTCHA surfacing, and all comment/vote/follow business logic (posting, cooldown handling, error classification).

### Cited Findings
- node-steam-session provides `LoginSession` with `startWithCredentials`, `startWithQR`, `submitSteamGuardCode`, `refreshAccessToken`, `renewRefreshToken`, `getWebCookies`, plus `LoginApprover.approveAuthSession()` for approving someone else's QR — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)
- EAuthSessionGuardType has four values exposed by the library: `EmailCode`, `DeviceCode`, `DeviceConfirmation`, `EmailConfirmation`; "only one required guard must be satisfied" — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)
- node-steamcommunity's `CConfirmation` object + `respond(time,key,accept)` handles confirmations; keys are obtained from `steam-totp`'s `getConfirmationKey(identitySecret,time,tag)` — [node-steamcommunity CConfirmation wiki](https://github.com/DoctorMcKay/node-steamcommunity/wiki/CConfirmation)
- SteamKit2 sample uses `client.Authentication.BeginAuthSessionViaCredentialsAsync(AuthSessionDetails{...})` — [SteamKit sample via rust-digger](https://rust-digger.code-maven.com/src/steam-language-gen-0.1.4/assets/SteamKit/Samples/1a.Authentication/Program.cs)
- python-steam / `steam.webauth`: documents `WebAuth` wrapping `BeginAuthSessionViaCredentials` and following the Guard flow — [steam-next.readthedocs.io webauth](https://steam-next.readthedocs.io/latest/api/steam.webauth.html)
- ASF explicitly auto-accepts only confirmations it generated itself (not any ambient confirmation on the account) — [ArchiSteamFarm Two-factor-authentication wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/Two-factor-authentication-fr-FR)

### Inferences
- Every public bot repo (3urobeat, ASF, SDA sidecars) is roughly the same layered stack: `steam-session`-equivalent for auth → `steam-user`-equivalent for CM/chat → `steamcommunity`-equivalent for web/community actions → `steam-totp`-equivalent for OTP/HMAC — with the bot author writing only the top "task scheduler" layer.
- Captcha and image-recognition are nowhere in these SDKs, matching the finding that the modern login endpoint doesn't hand clients a captcha to solve.

### Gaps
- Exact feature parity table for refresh-token renewal between SteamKit2 and python-steam not confirmed from fetched sources.

---

## How do commercial panels handle user-supplied accounts?

### Takeaway
Evidence for how closed commercial panels operate is sparse and largely anecdotal; what the public open-source ecosystem shows is that accepting a maFile (or the two secrets + username/password) is the lowest-friction path, because it lets the panel both log in and sign mobile confirmations automatically. QR-login is used for "linking" your own account into a panel without surrendering credentials, but is less common for mass-account bots because the panel still needs the maFile to confirm trades/market actions.

### Cited Findings
- ASF, SDA, steamguard-cli, WinAuth all converge on "unencrypted maFile JSON" as the interchange format, with `shared_secret` + `identity_secret` as the two required fields — [ArchiSteamFarm wiki](https://github.com/JustArchiNET/ArchiSteamFarm/wiki/Two-factor-authentication-fr-FR), [github.com/Jessecar96/SteamDesktopAuthenticator](https://github.com/Jessecar96/SteamDesktopAuthenticator)
- 3urobeat `accounts.txt` expects `username:password[:shared_secret]` per line (inferred from the config wiki's references to `logininfo`/shared_secret fields) — [3urobeat config_doc.md](https://github.com/3urobeat/steam-comment-service-bot/blob/master/docs/wiki/config_doc.md)
- QR login is an available "approve-from-mobile" alternative in node-steam-session via `startWithQR()` — [github.com/DoctorMcKay/node-steam-session](https://github.com/DoctorMcKay/node-steam-session)

### Inferences
- A comment/follow/fav panel that only needs login (no trade) can get by with `username:password:shared_secret`; a panel that wants to confirm market or trade actions must also take `identity_secret` and `device_id`, i.e. a full maFile.
- "QR-login + user's own maFile remains on the user's phone" is architecturally possible via `LoginApprover.approveAuthSession()` but is not reflected in the open-source bot configs reviewed — those take credentials directly.

### Gaps
- No primary-source documentation of a specific commercial panel's intake form found in the sources consulted.

---

## Notes on fetch failures (for the report writer)

- `steamapi.xpaw.me`, `dev.doctormckay.com`, and `cdn.jsdelivr.net` all returned DNS/allowlist errors in this session; information attributed to those hosts in this file comes from web-search summaries, not direct fetches. Re-run fetches against those hosts if the session network is widened, especially for: the full `EAuthSessionGuardType` enum and the raw `steammessages_auth.steamclient.proto` RPC list.
