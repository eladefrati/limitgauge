# Backlog

- **Label resets by `label`, not `id`.** `RawResets.Grant` reads only `id` and shows it as
  the reset's name. Each grant also carries a `label` (plus `clears`, `type`,
  `resets_left`); use `label` for the row.
