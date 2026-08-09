# Upstream relationship

## Baseline

This fork starts from [cifertech/ESP32-DIV](https://github.com/cifertech/ESP32-DIV),
commit `9d4d82fe7a12febf554b12e1eca6d434ebe79d39` on `main`.
The exact unmodified source is tagged `upstream-baseline-2026-08-07` locally.
See `UPSTREAM_BASE.md` for the machine-readable baseline facts.

## Version detail

The source and web manifests report firmware `v1.7.2`. Upstream continued to add
repository commits after its `v1.7.2` Git tag, so the firmware version string does
not uniquely identify the audited source. Always record both version and commit.

## Preservation policy

- Keep the upstream remote so later changes can be inspected and merged selectively.
- Preserve upstream Git history and the MIT license notice.
- Avoid rewriting working protocol/radio implementations solely for style.
- Document intentional behavioral changes and their regression evidence.
- Make clean internal cutovers when an old path is replaced; do not retain permanent
  parallel implementations without a demonstrated compatibility requirement.

No firmware behavior was changed during Phase 0 or Phase 1.
