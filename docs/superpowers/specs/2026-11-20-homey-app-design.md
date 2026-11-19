# Calvin app for Homey: design

Date: 2026-11-20. Status: **for owner approval before building.** Issue: calvin-gkdx.

Sources:

- Research, approved by the owner in full:
  [A Calvin app for Homey](https://github.com/osterbergsimon/calvin/blob/research/homey-app/docs/research/2026-11-19-homey-app.md)
  (Calvin branch `research/homey-app`). This page records what was decided and how the app is
  built; it does not repeat why each alternative lost.
- **UX and copy spec by the Calvin design lead**:
  [Calvin app for Homey: UX and copy](https://github.com/osterbergsimon/calvin/blob/design/homey-app/docs/design/2026-11-20-homey-app.md)
  (Calvin PR 377, branch `design/homey-app`; it moves to `develop` when merged). Called **the UI
  spec** below, cited as *UI §n*.
- Calvin's side: [Home automation: turn screens on and off](https://github.com/osterbergsimon/calvin/blob/develop/docs/integrations/HOME_AUTOMATION.md),
  the [screen power design](https://github.com/osterbergsimon/calvin/blob/develop/docs/superpowers/specs/2026-11-18-screen-power-design.md),
  and on `develop`: `backend/app/api/routes/kiosks.py`, `backend/app/security/route_policy.py`,
  `backend/app/security/host_origin.py`.
- Athom: the [Homey Apps SDK](https://apps.developer.homey.app/) docs and the
  [SDK v3 reference](https://apps-sdk-v3.developer.homey.app/).

**User-facing text.** Every word a person sees comes from the UI spec, verbatim (the text; its
§9 string ids are suggestions, and the plan uses them plus the few it adds): pairing and repair
views, device, driver and capability names, insights titles, flow card titles and hints, the app
settings page, warnings, unavailable messages and thrown errors. This design names each place and
points at the UI spec section; it does not restate or change the copy. Where the UI spec leaves a
mechanism to the planning lane, the decision is here.

## Intent

A household with Calvin and a Homey Pro sees each Calvin wall screen as a Homey device. They turn
it on or off from the Homey app, by voice or in a flow ("nobody home: screens off"), send it back
to its schedule, see whether its schedule is paused, and get a flow trigger when it goes offline.
They connect the Homey to Calvin once with a limited key, and never paste the key into a flow card
again.

## Goals

- One driver, **Screen**: each Calvin screen is one Homey device.
- `onoff` means **hold on / hold off** in Calvin and shows Calvin's `desiredOn` (planned on, not
  lit).
- A read-only **schedule paused** state (Calvin's `held`, counting holds set in Calvin Settings),
  an **offline** alarm, and a separate **back to schedule** action, with their flow cards.
- App-level actions for every screen added to Homey: on, off, back to schedule.
- One app-level **connection** (Calvin address plus a `screens.power` key), entered at the first
  pairing, changed through repair or the app settings page. Later pairings go straight to the list.
- State read from Calvin about every 30 s, and from each command's answer at once.
- Clear device states when Calvin cannot be reached, refuses the key, is too old, or a screen was
  removed from Calvin.
- Installed with the Homey CLI (`homey app install`); checked in CI without a Homey.

## Non-goals (v1)

- **Homey Cloud / Homey Bridge.** The Bridge cannot reach a LAN server; cloud apps need a Verified
  Developer account. Homey Pro only, `"platforms": ["local"]`.
- **The Homey App Store.** Deferred until Calvin announces itself over mDNS and offers a pairing
  code ([Deferred expansions](#deferred-expansions)).
- **An event stream from Calvin.** Deferred; v1 polls.
- **Show a page, show a message, a touch trigger.** Later, each with its own design.
- **More than one Calvin server in the UI.** The stored shape allows it; v1 connects one.
- **Who set a hold** (Homey or Calvin Settings): not shown (UI §4.2).
- **An admin-key warning.** Calvin cannot tell the app a key's kind yet; the warning comes with
  Calvin's "who am I" read (UI §3.1).
- **Syncing names.** The device takes Calvin's screen name at pairing; renames do not follow.
- **Translations.** English only; Dutch and others slot in as UI §9 describes.
- **Old firmware.** `"compatibility": ">=12.9.0"` (Node.js 22 on every Homey Pro).
- **Changes in Calvin.** v1 uses Calvin's API as it is ([Calvin API used](#calvin-api-used)).

## Anti-goals (never)

- Never ask for `homey:manager:api` or any other app permission. The app needs none.
- Never need Calvin to be reachable from the internet, and never open a `public` Web API endpoint
  on the Homey.
- Never compute a screen's schedule in the app: what a screen will do is Calvin's `desiredOn`.
- Never put the key anywhere but the app's connection record: not in a flow, a device setting or
  store, a capability, a token, a log line, an error, a pairing view after entry, or the settings
  page.
- Never send the key to a host other than the one entered (no redirects followed).
- Never turn a screen on or off as a side effect of pairing, repair, settings, polling or an app
  update.

## Constraints

- **Calvin's API as it is** on `develop`: the three calls below, the `screens.power` permission
  (`PERMISSION_ROUTES`), and their answers.
- **30 changes a minute per key**, then `429` with `Retry-After`. Requests that change nothing and
  every `GET` do not count.
- **Failed key attempts are rate-limited per IP address** (`api_key_limiter`): a Homey that keeps
  sending a revoked key locks itself out. The app stops polling on the first refusal.
- **Calvin's Host check** accepts an IP address, a single-label name, `localhost` and
  `*.localhost`, names ending in `.local`, `.lan`, `.home`, `.home.arpa`, `.internal` or
  `.localdomain`, and names in its allowed-hosts setting (`*` there turns the check off). Any other
  name gets a plain-text `400`. Bearer requests skip the Origin
  check.
- **Homey runtime**: Node.js 22, SDK v3, the `homey` module provided at runtime, timers through
  `this.homey.setInterval` / `setTimeout`. An unavailable device blocks its capabilities and flow
  actions. No manifest option that needs newer firmware than 12.9.0 (for example `uiState`, which
  needs 13.5.0).
- **Ids are forever.** The app id `xyz.wholab.calvin`, the driver id, capability ids and flow card
  ids break flows if changed; a device's `data` cannot change after pairing.
- **Public repo**, GPL-3.0. No household data anywhere; tests never contact a real Homey or Calvin.

## Approaches

From the research; the owner approved the first of each.

| Question | Picked | Rejected |
|---|---|---|
| Platform | Homey Pro only, SDK v3 | Homey Cloud (cannot reach a LAN server; needs a business account) |
| Language | TypeScript on Node.js 22 | Python (needs Homey 13); plain JavaScript |
| Devices | Each screen is a device with `onoff` | One hub device with a picker; flow cards only |
| On/off meaning | Hold on / hold off; schedule is a separate action | A three-way `power` picker (loses the native tile and voice) |
| Held | Mirrors Calvin's `held`, counts holds set in Settings | Only automation holds |
| Connection | One app-level connection, entered at pairing | Per-device settings |
| Key | Paste a `screens.power` key | Pairing code (later, Calvin work); OAuth-like redirect |
| State | Poll every 30 s | Event stream (later); webhook into Homey; Homey Cloud webhook |
| Distribution | `homey app install` | App Store (later); Test channel |
| Repo | New public `calvin-homey` | A folder in calvin-plugins |

One addition since the research: the UI spec adds a small **app settings page** for changing the
address or key, because that is where people look first (UI §6). It uses the same checks as
pairing. The research had left it out to keep the key in fewer places; the
[key storage](#connection-and-key-storage) rules below keep the page from ever reading the key.

## The design

### App structure

A Homey Compose app; `app.json` is generated and committed, never edited by hand.

```
.homeycompose/
  app.json                      id xyz.wholab.calvin, sdk 3, platforms ["local"],
                                compatibility ">=12.9.0", no permissions, brandColor,
                                api: getConnection GET /connection,
                                     putConnection PUT /connection (not public)
  capabilities/calvin_held.json
  capabilities/alarm_calvin_offline.json
  flow/actions/all_on.json, all_off.json, all_back_to_schedule.json
drivers/screen/
  driver.compose.json           name, class, capabilities, capabilitiesOptions, pair, repair
  driver.flow.compose.json      device triggers, conditions, actions
  driver.mts                    onPair, onRepair
  device.mts                    one Calvin screen
  pair/connect.html             custom view, pairing and repair mode
  assets/                       driver icon and images
settings/index.html             app settings page (UI §6)
api.mts                         app Web API for the settings page
app.mts                         App: connection store, pollers, app flow cards
lib/                            no Homey imports, unit-tested on their own
  calvin-client.mts             HTTP client (HIGH RISK)
  connection-store.mts          connections in ManagerSettings (HIGH RISK)
  connection-check.mts          the checks of UI §3.1, in order
  address.mts                   address parsing
  screen-state.mts              Calvin answer -> device values
  poller.mts                    one per connection
  errors.mts                    typed errors, each with a UI spec string id
locales/en.json                 custom-view and message strings (UI §9 keys)
assets/                         app icon and images
```

`lib/` holds everything that can be tested without the `homey` module. `app.mts`, `driver.mts`,
`device.mts` and `api.mts` stay thin: they wire `lib/` to Homey.

### Driver and device

- Driver id **`screen`**, name from UI §2.
- Device class **`tv`**, falling back to **`other`** if `tv` adds UI or voice behaviour that does
  not fit (UI §2; checked on the owner's Homey). A class can be changed later with `setClass`.
- Device name at pairing: Calvin's `overrides.displayName`, else the screen id (UI §2).
- Device `data` (fixed at pairing): `{ "kioskId": "<Calvin screen id>", "connectionId": "<id>" }`.
  No address, no key. `connectionId` is the app's own random id for the Calvin server, so a
  second server or an mDNS server id can come later without re-pairing.
- No device settings and nothing in the device store in v1.

### Capabilities

| Capability | Kind | Value | Set by the user |
|---|---|---|---|
| `onoff` | system | Calvin's `desiredOn` | Yes: on sends `PUT {"power":"on"}`, off sends `{"power":"off"}`; both hold |
| `calvin_held` | custom boolean, `getable`, `setable: false`, `uiComponent: "sensor"` | Calvin's `held` | No |
| `alarm_calvin_offline` | custom boolean, `getable`, `setable: false` | `!online` | No |
| `button.back_to_schedule` | system `button`, `maintenanceAction: true` | none | Yes: sends `{"power":"schedule"}` |

Titles, `titleTrue` / `titleFalse`, insights titles, the maintenance button's `title` and `desc`:
UI §4.1. `onoff` keeps Homey's own title; only its insights titles are set, in the driver's
`capabilitiesOptions`. Homey generates no condition cards for the custom booleans, hence
`schedule_paused` and `is_offline` below.
If Homey shows Yes/No instead of `titleTrue` / `titleFalse` for a custom boolean, `calvin_held`
uses the UI spec's fallback title (UI §4.1).

**How Calvin's answer maps** (one function in `screen-state.mts`, from a `ScreenPowerView`; the
resulting tile states are UI §4.2):

| Calvin answer | Device |
|---|---|
| `desiredOn` | `onoff`. The plan, not a measurement of the panel. |
| `held` | `calvin_held`. True for any hold, including one a person set in Calvin Settings. |
| `online: false` | `alarm_calvin_offline` true. The device stays **available**: a hold sent while a screen is offline is kept and obeyed when it is back. |
| `agentHolds: false` | `setWarning` with the UI §4.2 warning. If the owner's Homey does not show warning text, the warning is dropped (UI §4.2). |
| `agentHolds: null` or `true` | `unsetWarning`. `null` means no screen has checked in since Calvin was updated. |
| `power`, `setAt`, `by`, `id` | Not shown in v1. |

A value is written only when it differs from the current one, so Homey's triggers fire on real
changes only.

### Flow cards

Ids, titles, inverted forms and hints: UI §5, verbatim. Every device card takes only the screen
(`filter: driver_id=screen`); app cards take no arguments.

| Id | Kind | Mechanism |
|---|---|---|
| `onoff` cards (turn on, turn off, toggle, is on, turned on, turned off) | device | Built in |
| `calvin_held_true`, `calvin_held_false` | device triggers | Declared in `driver.flow.compose.json`; Homey runs them when the app sets `calvin_held` |
| `alarm_calvin_offline_true`, `alarm_calvin_offline_false` | device triggers | Same, for `alarm_calvin_offline` |
| `schedule_paused` | device condition | Run listener reads `calvin_held` |
| `is_offline` | device condition | Run listener reads `alarm_calvin_offline` |
| `back_to_schedule` | device action | `PUT {"power":"schedule"}` |
| `all_on`, `all_off`, `all_back_to_schedule` | app actions | Loop the paired screens |

The app actions send one `PUT` per paired screen that is available, in sequence, and continue
past a failure. *total* is every paired screen; an unavailable screen gets no request and counts as
failed, because it was not changed (my call). If any failed, the card throws UI §5.3's
partial-failure message with *n* and *total*; the screens that worked stay changed. No paired
screens: the card succeeds and sends nothing. With more than 30 screens Calvin's change limit can
stop the rest; they count as failed.

### Pairing, repair and the settings page

All three use the same [connection check](#connection-check) and the same messages (UI §3.1).

**Pair views** (`driver.compose.json` `pair`): `connect` (custom) → `list_devices` (system) →
`add_devices` (system). If a connection already exists and is not *key refused* or *too old*,
`onPair` calls `session.showView('list_devices')` at once, skipping `connect` (UI §3). A refused or
too-old connection shows `connect` instead, prefilled with the saved address; its check replaces
that connection under the same `connectionId`, so existing screens follow (my call). An
unreachable one goes to the list, which throws UI §3.2's unreachable line.

**When the connection is saved** (my call): on a successful check, so `list_devices` can use it.
When the pairing session disconnects, a connection that no device uses is deleted again, so an
abandoned pairing leaves no key on the Homey.

**Repair views**: the same `connect` view (`drivers/screen/pair/connect.html`, listed under
`repair` with the same id) in repair mode: title, note, prefilled address, empty key, **Save**
(UI §3.3). On load the view emits `init`; the driver answers `{ mode: "pair" | "repair",
address? }`, so one file serves both (my call). A repair on any screen, in this order, saves the
new address and key under the same `connectionId` (clearing *key refused*), sets the poller back to
ok, and runs a poll at once; that poll's answers then follow the normal state rules. Repair cannot
bring back a removed screen (UI §3.3).

**The `connect` view** (UI §3.1 elements, layout and messages) checks for an empty address and an
empty key in the view itself, then emits one session event, `connect`, with `{ address, key }`. In
repair mode an empty key is refused too: repair always takes a fresh key (UI §3.3). The driver
runs the connection check and answers `{ ok: true }` or `{ ok: false, error: "<string id>",
tokens }`. The view shows the message for that id under the form, or `Homey.alert` for
`error.unknown`. The key is never sent back to the view.

**`list_devices`**: Calvin's screens that are not in Homey yet, from the check's screen list, named
as above, not filtered by online state. Empty cases throw the UI §3.2 messages. The view title is
UI §3.2's if the template takes one.

**The app settings page** (`settings/index.html`, UI §6) never calls `Homey.get` or `Homey.set`:
the settings view API can read every app setting, and the key is one. It talks only to the app's
own Web API (`api.mts`, routes not `public`):

- `GET /connection` → `{ connected, problem, address, keyHint, screens }`: the status line
  (UI §6), the saved address, the key's first characters, and the number of screens in Homey. No
  connection: `{ connected: false }`.
- `PUT /connection` with `{ address, key? }`: runs the connection check with the new key, or the
  saved key when **Change key** was not used, then saves and polls at once. Answers like the pairing
  event.

`keyHint` is the key's first 8 characters (`clv_` plus 4, Calvin's own prefix display).

#### Connection check

`connection-check.mts` runs these in UI §3.1's order and maps each failure to its UI §3.1 message
id. Every request sends the key as `Authorization: Bearer`. Nothing in it changes a screen.

1. **Address** (`address.mts`): trimmed; no `http://` or `https://` → `http://` is added silently;
   then it must parse as a URL with a host, an optional port, and no path (a trailing `/` is
   allowed), query, fragment, user or password. Otherwise *address invalid*. No port is guessed:
   the UI spec's hint asks for it.
2. **`GET /api/kiosks`.** Network error or timeout → *unreachable*. TLS certificate error →
   *certificate*. `401` → *key refused*. `400` from Calvin's Host check → *unreachable* with the
   address (my call: the person fixes the address either way; the log line says "host refused").
   A `200` without a `kiosks` array, or anything that is not JSON → *not Calvin*. Keeps the list.
3. **Calvin too old?** `GET /api/kiosks/<probe>/screen-power`, where `<probe>` is a fresh random id
   that matches Calvin's id pattern (`[A-Za-z0-9._-]{1,64}`) and is not in the list, such as
   `homey-check-<12 hex digits>`. A `404` whose body is `{"detail": "Unknown kiosk"}` means this
   Calvin has screen power. Any other `404`, a `405`, or a `200` that is not a `ScreenPowerView`
   → *too old*. (Calvin's SPA fallback skips `api/` paths and answers `404` there, per
   `backend/app/services/frontend_static.py`, so an old Calvin should answer `404 Not found`.) As a second signal, a
   non-empty screen list whose entries lack `powerHold` and `agentHolds` → *too old* too (UI §7).
   This is the exact check the UI spec left to this design; the owner confirms it on a real old
   Calvin if one is at hand (plan, owner step).
4. **May the key switch screens?** `PUT /api/kiosks/<probe>/screen-power` with
   `{"power": "schedule"}`. Calvin's route policy runs before the handler, and the handler never
   creates a screen for an unknown id (screen-power design, decision 4), so `404 Unknown kiosk`
   means yes. `401` or `403` → *key refused* (a plain read key gets `403 insufficient_scope`).
5. Anything else at any step → *unknown* (`Homey.alert`), including a `429` (Calvin's lockout after
   failed keys from this address; UI §3.1's "try again in a moment" fits it).

The check cannot tell a `screens.power` key from an admin key; that warning waits for Calvin's
"who am I" read (UI §3.1).

### Connection and key storage

**HIGH RISK.** The connection is the only secret the app holds.

- Stored in `ManagerSettings` (`this.homey.settings`) under one key, `connections`:
  `{ "<connectionId>": { "address": "http://calvin.example:8000", "key": "clv_…",
  "keyRefused": false, "createdAt": "…" } }`. JSON, kept across restarts and updates, deleted when
  the app is uninstalled.
- v1 holds at most one entry. The map shape lets a second Calvin, or an mDNS `serverId`, be added
  without touching devices.
- Only `connection-store.mts` reads or writes it, and it hands the key only to
  `calvin-client.mts` and to the connection check (a settings save that keeps the saved key). Everything else sees `{ connectionId, address, keyHint, keyRefused }`.
- `keyRefused` is persisted, so a restart does not send a refused key again; it clears only
  through repair or the settings page (UI §7).
- When the last device of a connection is deleted, the connection is deleted with it, so the key
  does not linger on the Homey (my call). The settings page and README tell the person to revoke
  the key in Calvin (UI §6).
- Readable by whom: the app's code, and anyone who can open this app's settings page with
  developer tools or call Homey's own Web API with enough rights (the settings view API can read
  every app setting). Athom does not document who else can; assume any owner-level user of that
  Homey. Encrypting the key inside the same app would not change that, so v1 does not pretend to.

### Polling and offline handling

- **One poller per connection**, owned by the App, started when the first device of that
  connection initialises, stopped when the last is deleted or the app unloads. Timers via
  `this.homey.setInterval`.
- **Every 30 s**, the poller reads `GET /api/kiosks/{id}/screen-power` for each paired screen of
  the connection, one after another, and hands each answer to its device. A round still running
  when the next is due makes the next one skip. First round right after start, and right after a
  repair or settings save.
- **After a command**, the device applies the `PUT` answer at once (UI §4.3).
- **A stale poll never overwrites a newer command.** Each device keeps a command counter; a poll
  answer requested before the latest command was sent is dropped. **HIGH RISK**: the race the
  research flagged.
- Polls send the key, so a revoked key shows within one round.
- **A failed round** is one in which no screen got an answer from Calvin (network error, timeout,
  `5xx`, Host `400`). Any Calvin answer, a `404` for one screen included, makes the round good. A
  skipped round counts as neither (my call).
- **`429` on a poll** (the failed-key lockout, since `GET`s never hit the change limit): the round
  stops, the poller waits `Retry-After` seconds before the next round, and the round counts as
  neither good nor failed (my call).
- A command increments the device's counter when it is sent, whether it then succeeds or fails; a
  poll dropped because of a failed command is harmless, the next round reads the truth.
- `GET /api/kiosks` is not polled; only the connection check reads it.

**Connection states** (applied to every device of the connection; messages are UI §7):

| State | Cause | Devices | Polling |
|---|---|---|---|
| ok | Answers as expected | available | every 30 s |
| unreachable | Network error, timeout, `5xx`, or Host `400`, **3 rounds in a row** (about 90 s, UI §7) | unavailable, UI §7 *unreachable* | continues; available again on the first good round |
| key refused | `401` (any code) or `403 insufficient_scope`, once | unavailable, UI §7 *key refused*; `keyRefused` saved | **stops** until repair or settings save |
| too old | `404`/`405` not from the screen-power handler | unavailable, UI §7 *too old* | continues; screens come back by themselves after Calvin is updated |

**Screen states** inside an ok connection:

| Answer | Device |
|---|---|
| `200` | values mapped as above; available |
| `404 Unknown kiosk` | unavailable with UI §7 *screen removed*; still polled, and available again if it reappears |
| `200` with `online: false` | available, `alarm_calvin_offline` true |

`setUnavailable` is called with the message; if the owner's Homey does not show it, the device is
still unavailable and the thrown errors carry the text (plan, owner step).

### Commands and errors

- `onoff` listener → `PUT {"power": "on"|"off"}`, always, even when `desiredOn` already matches:
  setting it means "hold", and Calvin writes nothing when nothing changes; `back_to_schedule` card, maintenance button →
  `{"power": "schedule"}`. Success: the answer is applied. Failure: the listener throws the UI §7
  "thrown" text, and Homey reverts the tile.
- Errors are typed in `errors.mts`; each carries a UI spec string id and tokens, never a raw
  Calvin body, the key or the address.

| Calvin answer to a command | Thrown (UI §7) | Side effect |
|---|---|---|
| network error, timeout, `5xx` | *unreachable* | none (only polls count toward the 3-round rule) |
| `401`, `403 insufficient_scope` | *key refused* | connection moves to *key refused* |
| `404 Unknown kiosk` | *screen removed* | that device unavailable |
| `404`/`405` otherwise | *too old* | connection moves to *too old* |
| `429` | *too many changes* | none |
| `3xx`, `422`, unparseable, anything else | *anything else* | logged by status only |

### The HTTP client

**HIGH RISK.** `calvin-client.mts`, no Homey imports, no runtime dependencies.

- Node's built-in `fetch`. Base URL from the connection; paths from fixed templates, the screen id
  checked against Calvin's id pattern and `encodeURIComponent`-ed. An id made only of dots (`.`,
  `..`) matches Calvin's pattern but would change the path, so the client refuses it (my call).
- Headers: `Authorization: Bearer <key>`, `Accept: application/json`, `Content-Type:
  application/json` on `PUT`. No `Origin`, no cookies.
- `redirect: "manual"`: a `3xx` is an error, so the key never goes to another host and never drops
  from `https` to `http`.
- Timeout 5 s per request (`AbortSignal.timeout`).
- `https://` verifies the certificate (Node's default). No "skip verification" switch (UI §3.1's
  certificate message tells the person what to do instead).
- Every answer is parsed and checked against the expected shape; anything else is an error, never
  a crash.
- The client never logs.

### Logging

Homey app logs can end up in a diagnostics report a person sends to the developer. The app logs
the cause in plain words (UI §7), the status code and the screen id. Never the key, never the
address (UI §7), no screen names, no Calvin answer bodies.

## Calvin API used

On Calvin `develop`. **Calvin needs nothing new for v1.**

| Call | When | Auth | Success | Failures the app handles |
|---|---|---|---|---|
| `GET /api/kiosks` | connection check | public; key sent so a refused key shows | `200 {"kiosks": [{"id", "hostname", "lastSeen", "overrides": {"displayName"?}, "powerHold", "agentHolds", …}]}` | `401 invalid_api_key`; `429 rate_limited` after failed keys from this IP; `400` Host; network |
| `GET /api/kiosks/{id}/screen-power` | each poll; check step 3 | public (listed under `screens.power` to document it) | `200` `ScreenPowerView` | `404 Unknown kiosk`; `400 Invalid kiosk id`; `401`; `429`; `400` Host; network; `5xx` |
| `PUT /api/kiosks/{id}/screen-power` | commands; check step 4 | `screens.power` key (or admin) | `200` `ScreenPowerView` | `401 invalid_api_key`/`admin_required`; `403 insufficient_scope`; `404 Unknown kiosk`; `422`; `429 rate_limited` with `Retry-After`; `400`; network |

`ScreenPowerView`: `{ "id", "power": "on"|"off"|"schedule", "held", "setAt", "by":
"automation"|"settings"|null, "desiredOn", "online", "agentHolds": true|false|null }`.
Error bodies are `{"detail": {"code", "message"}}`, except `{"detail": "<text>"}` for `404`/`400`
from the screen-power handler and a plain-text `400` from the Host check. The app keys on the
status, then `code`, and on the exact `"Unknown kiosk"` detail.

The fake Calvin in unit tests is shaped by Calvin's contract snapshot
(`backend/tests/contract/openapi.json`) for these three paths.

## Security

- **Where the key lives**: the app's `ManagerSettings` on the Homey
  ([Connection and key storage](#connection-and-key-storage)). The gain over today: it is no longer
  in a flow card that anyone who can edit flows can read, and the settings page shows only its
  first characters.
- **A leaked key** can switch screens on, off or back to schedule, 30 changes a minute, and read
  what any anonymous caller on the LAN can. Not settings, backups, MCP or host actions.
- **Plain HTTP on the LAN**: a device that can watch traffic can copy the key from any request
  (polls included, every 30 s). Same exposure as the flow-card setup today. The app accepts
  `https://` with certificate checks; the README says so.
- **Revoke**: Calvin › Settings › Security › API keys › Revoke. The next request gets `401`; every
  screen becomes unavailable and polling stops until a repair or settings save with a new key.
- **Uninstall**: the app's settings, and the key with them, are deleted from the Homey. The key
  stays valid in Calvin until revoked; the settings page and README say so.
- **No inbound path**: the app opens every connection itself; its Web API routes are not `public`
  and serve only the settings page.
- **No permissions** in the manifest.
- **Presence**: unchanged; holds are already public reads in Calvin.
- **Supply chain**: no runtime dependencies beyond what the Homey provides; dev dependencies and
  GitHub Actions are pinned (plan).

## Left to the builder

- The TypeScript layout as Athom's TypeScript template produces it (`.mts`, `tsconfig.json` with
  `outDir: ".homeybuild/"`), as long as `lib/` stays free of Homey imports.
- The shape of the stub `homey` module for tests, as long as it covers what the app calls.
- `onPairListDevices` or a `list_devices` session handler, whichever works with the custom view.
- Placeholder icons and images until the store release (UI §2 describes the final ones).
- The settings page's markup, using Athom's settings styles and UI §6's elements.

## Deferred expansions

What comes later, and what v1 must not paint into a corner. Copy for each is in UI §8.

| Later | What it needs | v1 keeps the door open by |
|---|---|---|
| **App Store** | Calvin's mDNS advertisement (`_calvin._tcp`, TXT `id`, `v`, `name`), a pairing code exchanged for a limited key, store images, `README.txt` (UI §2), `validate --level publish` | The app id is final. Devices carry no address, so `onDiscoveryAddressChanged` updates one connection record. `connectionId` can map to the mDNS `id`. The check takes `{ address, key }` from any source, so a discovery view and a `pincode` view can sit in front of it (UI §8). Store releases continue the same semver. |
| **Event stream** | A Calvin SSE route for screen-power and online changes | Devices consume `ScreenPowerView` updates through one `applyState`; the poller is one source behind a small interface (`start`, `stop`, updates out). A stream source plugs in beside it, polling stays as the fallback. |
| **Bulk state, "who am I"** | Calvin's optional `GET /api/kiosks/screen-power` and `GET /api/auth/key` | The poller asks the client for "states of these screens", so a bulk call replaces N calls inside the client. The check is one module that can add the admin-key warning (UI §3.1). |
| **Page, message, touch** | Calvin routes and permissions; their own designs | The key is one opaque value; the app requires `screens.power` but does not assume the key has only that. New cards go beside the existing ones; no id changes. |
| **Several Calvin servers** | A server picker | `connections` is a map; devices carry `connectionId`; one poller per connection. |
| **Who set the hold** | Calvin already returns `by` | A second sensor capability (UI §8); `screen-state.mts` already receives `by`. |
| **Translations** | Strings per language | Every string is a translation object or a `locales/` key (UI §9). |

## Builder check

A fresh reader went through this design as the builder (24 points). How they were sorted:

- **Answered in the design** (the answer is now in the section named): `api` route methods and
  paths (App structure); skipping `connect` (Pair views); cost of one `GET` per screen (Polling);
  the stale-poll counter (Polling); custom booleans get no generated conditions (Capabilities);
  the SPA fallback on an old Calvin (Connection check, step 3); insights titles in
  `capabilitiesOptions` (Capabilities); which string ids to use (User-facing text); `setWarning`
  and `setUnavailable` text, checked on the owner's Homey (plan, Part 6).
- **Minor, settled as my call** (marked "my call" where it is written): `429` on a poll, a command
  and the check; one view file for pairing and repair; the order of a repair; when the connection
  is saved and how an abandoned pairing cleans up; pairing while the connection is refused or too
  old; what makes a round failed; how "all screens" counts unavailable screens; sending `on` when
  the screen is already planned on; screen ids made only of dots; the Host check list; no manifest
  option newer than 12.9.0.
- **For the owner**: below.

## Open questions for the owner

1. **Minimum Calvin version.** Screen power is on Calvin `develop`, not on `main` (last tag
   `0.1.0-rc.3`). Which Calvin release should the README name as the minimum, and should the app's
   first tag wait for it?
