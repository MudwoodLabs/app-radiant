# Reproducible builds — `app-radiant`

This is a fork of [`LedgerHQ/app-bitcoin`](https://github.com/LedgerHQ/app-bitcoin) that builds a Radiant (RXD) variant. v1 target: Ledger Nano S Plus.

See [`../radiant-ledger-app`](https://github.com/MudwoodLabs/radiant-ledger-app) for the plan and design docs.

---

## Quick build (local, reproducible)

Requires Docker. Apple Silicon: add `--platform linux/amd64`.

```bash
git clone --recurse-submodules git@github.com:MudwoodLabs/app-radiant.git
cd app-radiant

# Pinned builder image digest (multi-arch index, resolved 2026-04-15)
IMAGE='ghcr.io/ledgerhq/ledger-app-builder/ledger-app-builder-lite@sha256:b82bfff7862d890ea0c931f310ed1e9bce6efe2fac32986a2561aaa08bfc2834'

# Build the bootstrap variant (unmodified bitcoin_cash — proves the toolchain).
# Phase 1 will switch to COIN=radiant once the variant is implemented.
docker run --rm \
  -v "$(pwd):/app" \
  -u "$(id -u):$(id -g)" \
  "$IMAGE" \
  bash -c "cd /app && make COIN=bitcoin_cash"

# Artifacts land in ./bin/
sha256sum bin/app.hex bin/app.elf
```

## Pins

| Artifact | Pin | Notes |
|---|---|---|
| Builder image | `sha256:b82bfff7862d890ea0c931f310ed1e9bce6efe2fac32986a2561aaa08bfc2834` | multi-arch index digest of `ledger-app-builder-lite:latest` as of 2026-04-15 |
| `lib-app-bitcoin` submodule | `8b28f6687e94abc4bb88cd648ba1f9e1b6dd1b63` | tracks fork `MudwoodLabs/lib-app-bitcoin` (mirror of upstream as of fork date; Phase 1 will add the Radiant C diff here) |
| `ledger-app-workflows` | `2ddae7bf080353584b77bd1356c8909c5b8f8257` | pinned by commit SHA in `.github/workflows/*.yml`, not `@v1` tag |

## Verifying a release

Once a `vX.Y.Z` tag ships:

1. Clone at the tag: `git checkout vX.Y.Z --recurse-submodules`
2. Build locally using the exact command above.
3. `sha256sum bin/app.hex` → must match the SHA256 published in the GitHub release notes.

If it doesn't match:
- First check: did the release builder use a **different** image digest than the one in this file? Look at `.github/workflows/build_and_functional_tests.yml` at that tag.
- Second check: is your host Docker on a recent enough version to use the pinned digest correctly?
- Third check: did you reset the submodule to the pinned SHA? `git submodule status` should show `8b28f66...` (or whatever the tag pinned).

## Known limitation

The Ledger reusable workflow (`reusable_build.yml`) hardcodes `:latest` when pulling the builder image. That means **CI builds use whatever `:latest` points to at the moment they run**, while local builds (per this doc) use the pinned digest. When the builder image changes upstream, CI artifact hashes drift while local builds stay stable. To close this gap we'd need to fork `ledger-app-workflows` and pass the digest through — a v1.1 improvement, not blocking for v1.

## Non-determinism sources to watch

- `__DATE__` / `__TIME__` macros — grep the C source and reject any occurrence.
- `SOURCE_DATE_EPOCH` env — honored by modern GCC; set in workflow to avoid timestamp drift.
- `PYTHONHASHSEED=0`, `PYTHONDONTWRITEBYTECODE=1` — for any Python tools invoked during build.
- Submodule SHA drift — `git submodule update --init --recursive` before every release build.

See [the plan](https://github.com/MudwoodLabs/radiant-ledger-app/blob/main/docs/plans/2026-04-14-feat-radiant-ledger-app-v1-plan.md) for the full rationale.
