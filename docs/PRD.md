# limitgauge

What limitgauge is for and how it behaves. Kept current: the [ADRs](adr/) record the
decisions behind it.

## Overview

limitgauge is a macOS menu bar app that shows the usage limits of a Claude subscription:
the same numbers as the Usage page in claude.ai settings, one click away. It shows the plan,
the current session limit, the weekly limit, the model-scoped weekly limit, when each resets,
cloud session credits, usage credits, and how the week's usage splits across surfaces.

It reads the data from the account usage endpoint that Claude Code uses, authenticated
with the token Claude Code keeps in the login Keychain. It never writes credentials, never
scans local logs, and never touches browser cookies.

The menu bar shows an icon only. The numbers live in the dropdown and are fetched when the
menu opens. A slow background poll exists for one purpose: a notification before a limit is
hit.

## Goals

- Every field of the claude.ai Usage page in the dropdown, with reset times in local time.
- Data that is current at the moment the user looks at it.
- A notification when a limit crosses a user-set threshold, once per limit per window.
- Zero setup beyond installing. No Keychain dialog, no commands to run, no token to paste.
  Signing in to Claude Code is the sign-in.
- Harmless to Claude Code: the token is read, never refreshed or rewritten.
- Negligible load: one request per menu open at most every two minutes, one background poll
  per 10 to 60 minutes depending on headroom.

## Non-goals

- A live number in the menu bar. Icon only; the dropdown has the numbers.
- Cost estimates from local Claude Code logs. The endpoint has no token counts and no
  API-price figures; estimating them means scanning session logs, which this app does not do.
- Browser cookie import. If the Keychain token is not available, the app says so.
- Other providers in the first version.
- A prebuilt download. limitgauge is built from source: no notarization, no auto-update.
- Identifying as Claude Code to the endpoint. The limit resets it serves only to Claude Code
  stay hidden.

## Design

### Data source

Two GET requests against `api.anthropic.com`, each with `Authorization: Bearer <token>` and
`anthropic-beta: oauth-2025-04-20`:

- `/api/oauth/usage?cedar_ember=1`: the limits and credits. Fetched on menu open and on
  background polls.
- `/api/oauth/profile`: the plan tier. Fetched after a successful usage fetch, at most once
  a day; it almost never changes.

Field mapping, as observed on 2026-10-18:

| Row | Source |
|---|---|
| Plan | `profile.organization.rate_limit_tier` (`default_claude_max_5x`) |
| Current session | `usage.limits[kind=session]`: `percent`, `resets_at` |
| This week | `usage.limits[kind=weekly_all]` |
| Model week (Fable) | `usage.limits[kind=weekly_scoped]`, `scope.model.display_name` |
| Cloud session credits | `usage.iguana_necktie`: `limit_dollars`, `remaining_dollars`, `resets_at` |
| Usage credits | `usage.spend` (`balance`, `used`) |
| Limit resets | `usage.cedar_ember`: `grants[]`, `ineligible_reason` |
| Weekly breakdown | `usage.seven_day_breakdown` |

The `limits[]` array is the stable-looking part. Several other keys are internal codenames
and will be renamed. Every row outside `limits[]` decodes as optional: a renamed key hides
its row and the rest of the menu keeps working. A `limits[]` element that cannot be read is
dropped, with a log line naming its kind and the failing fields. The endpoint is
undocumented; only a missing or wholly unreadable `limits[]`, or invalid JSON, is an error
(ADR-0001).

### Authentication

Claude Code stores its OAuth credentials as a generic password item named
`Claude Code-credentials` in the user's login Keychain, a JSON blob with an access token, a
refresh token and an expiry. limitgauge reads the item by running
`/usr/bin/security find-generic-password`, the tool Claude Code itself uses for the item,
and uses the access token. The item already trusts that tool, so no Keychain consent dialog
appears (ADR-0004).

Rules:

- Read only. The app never calls the token refresh endpoint and never writes the item. A
  refresh would rotate the refresh token and could log Claude Code out.
- On a 401, read the item again once: Claude Code may have rotated the token since the last
  read. If the second attempt also fails, the dropdown shows "Token expired. Run claude in a
  terminal to sign in again."
- The read is bounded by a five-second deadline, after which the tool is terminated.
- Each way the read can fail has its own message: no item, a locked keychain, a refused
  read, a timeout, a failing tool, an item in an unknown format. Only a missing item tells
  the user to sign in.
- The item's expiry is not read. The endpoint decides whether the token is still good.

### Menu bar

An `NSStatusItem` with a template image, so it follows light and dark menu bars. No text.
A small red dot at the bottom right of the icon appears while any limit is past its warning
threshold, and can be turned off in Settings.

The icon is a user choice: `hourglass.circle` (default), `hourglass`, `drop.circle`,
`gauge.with.needle`, `chart.bar.fill` or `timer`.

### Dropdown

An `NSMenu` whose rows are custom views, in the order of the claude.ai Usage page:

1. Plan name.
2. Current session: progress bar, percent, "Resets in 2h 14m (7:00 PM)".
3. This week: same.
4. Model week: same, titled with the model's display name from the response.
5. Cloud session credits: remaining of limit, expiry.
6. Usage credits: used and balance.
7. Limit resets: the grants, or "No resets right now".
8. Weekly breakdown: percent per surface.
9. Footer: "Updated 12 s ago", Refresh, Settings, Quit.

A row whose field is missing from the response is hidden, not shown empty, with these
exceptions:

- The endpoint drops the session limit while no five-hour window is open. **Current
  session** then shows 0% and no reset time.
- A session limit that is there but unreadable shows a dash and "Couldn't read this limit".
- A limit whose window has reset since the last fetch is unknown: a dash, an empty bar and
  "Reset since the last update" (ADR-0005).
- The endpoint withholds the limit resets from limitgauge and says so in
  `ineligible_reason`, so an empty list does not mean the account has none. The row is
  hidden whenever the endpoint gives a reason.

When a fetch fails, the menu keeps the last numbers and adds the failure's message below
them.

### Refresh model

Three triggers fetch usage; one snapshot serves them all.

- **Menu open, and launch.** The menu shows the cached snapshot immediately. If the last
  fetch attempt, successful or not, is older than two minutes, a fetch starts in the
  background and the rows update in place when it lands, about 0.2 s later. Nothing the user
  would act on changes faster than that, even in the five-hour window. Timing from the last
  attempt means a failing endpoint is not asked again on every menu open.
- **Refresh item.** Forces a fetch.
- **Background poll.** Exists only to drive warnings. After every successful fetch, the app
  computes each limit's headroom, the warning threshold minus the current percent, in
  percentage points. The limit with the least headroom sets the next poll:

  | Least headroom | Next poll |
  |---|---|
  | more than 40 points | 60 min |
  | 15 to 40 points | 30 min |
  | under 15 points | 10 min |

  A limit at or past its threshold, warned or not, is excluded until its window resets, so a
  warning that could not be delivered does not hold polling at 10 minutes; the next poll
  comes one minute after that reset if that is sooner. Any fetch attempt, including one from
  a click, restarts the clock.

After a failed fetch, polls back off, never sooner than the last snapshot's own pace: 10
minutes doubling up to 60 for no network, a 5xx or an unreadable response; 30 minutes, then
60, for a 429; 60 minutes for a failure only the user can fix. A poll does not resend a
token the endpoint rejected, and does not read the Keychain again after a read that was
locked out, refused or timed out; opening the menu or Refresh does both, and such a poll
does not count toward the two-minute menu cache (ADR-0002).

Timers do not fire while the Mac sleeps; on wake, an overdue poll runs at once. A fetch
without a network waits for one, up to 30 seconds, before it fails.

The cached snapshot and its timestamp persist across launches, so the menu is never empty at
startup.

### Warnings

One threshold per limit, in percent. Defaults: session 80, week 85, model week 80. When a
fetch shows a limit at or past its threshold, the app posts a macOS notification through
`UNUserNotificationCenter` naming the limit, the percent and the reset time.

A warning fires once per limit per window. The dedupe key is the limit kind and its
`resets_at`, rounded to the nearest minute because the endpoint shifts `resets_at` by under
a second between fetches; a new `resets_at` re-arms it. A warning counts as sent once
Notification Center accepts it, not when it is due. One that cannot be delivered, because
notifications are off or not allowed, goes out once they are on, if its window is still
open: at once when limitgauge's switch is turned on, and at the next refresh when
notifications are allowed in System Settings. A limit whose window has reset since the last
fetch does not warn (ADR-0005).

Notifications are on by default: the app requests the system permission at first launch, and
Settings has a switch to turn them off. If the permission was denied, Settings links to the
Notifications pane of System Settings.

### Settings

A small window: thresholds, notifications on or off, icon, warning dot on or off, launch at
login through `SMAppService`. Stored in `UserDefaults` alongside the last snapshot.

### Verification

- Unit tests decode synthetic fixtures in the real response's shape into the model,
  including a fixture with the codename keys removed. No account's usage is in the
  repository.
- The poll scheduler is a pure function of the snapshot, the run of failures and the
  thresholds, tested directly.
- The warning dedupe is a pure function of the snapshot and the warnings already sent,
  tested directly.
- Manual: open the menu next to the claude.ai Usage page and compare every row. Lower a
  threshold below the current percent and confirm one notification, then none on the next
  poll.

## Alternatives considered

- **An existing menu bar usage app.** Several exist. They poll on a timer and show a number
  in the menu bar, and none shows the full Usage page. Building our own gives fetch on open,
  the full set of fields, and the smallest exposure: one token, two endpoints.
- **Claude Code's status line as a data feed.** Claude Code hands its status line script the
  five-hour and weekly percentages after each turn. That covers two of the three limits, only
  while Claude Code is in use, and the script also runs on events other than turn end, so
  stale writes would race fresh fetches. The model-scoped limit and the credits are only on
  the endpoint. Not used.
- **Timer polling with a number in the menu bar.** Always slightly stale. Fetching on open
  is current by construction and simpler. The trade-off, accepted, is no information at a
  glance: the icon says nothing until it is clicked.
- **Refreshing the token ourselves.** Removes the "run claude" failure mode but risks
  invalidating Claude Code's refresh token. Rejected.

## Risks

- **Undocumented endpoint.** Keys are renamed or the shape changes. Mitigation: optional
  decoding per row, and a log line for each `limits[]` element that stops decoding.
- **Claude Code changes how it stores the Keychain item.** If it stops using the `security`
  tool, the consent dialog returns. The fallback then is a long-lived token created by
  `claude setup-token` and stored in limitgauge's own Keychain item. It is not built
  (ADR-0004).
- **Token expiry when Claude Code is idle.** The access token is short-lived and only Claude
  Code renews it. The menu says what to do. Accepted.
- **Usage endpoint rate limits.** Unknown. The two-minute cache and the headroom schedule
  keep requests to a handful per hour.

## Stack

Swift 6, AppKit, UserNotifications, ServiceManagement. SwiftPM executable, no
dependencies. macOS 26. Build and packaging as in ADR-0003.
