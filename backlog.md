# Backlog

- **Don't claim there are no resets when the endpoint withholds them.** The usage
  endpoint serves `cedar_ember` grants only to clients that identify as Claude Code; to
  limitgauge it answers `eligible: false` with `ineligible_reason: "surface"` and no grants,
  even when the account has resets (seen 2026-11-10 with a Fable and a full reset). The menu
  then shows "No resets right now", which is false. Read `ineligible_reason` and stop
  showing that line when the endpoint withholds the grants.
- **Label resets by `label`, not `id`.** `RawResets.Grant` reads only `id` and shows it as
  the reset's name. Each grant also carries a `label` (plus `clears`, `type`,
  `resets_left`); use `label` for the row.
