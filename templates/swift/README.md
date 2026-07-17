# Swift local check adapter

Swift repos do not need npm script naming.

Use one stable local entrypoint:

- `make check`, or
- `./scripts/check.sh`

Recommended responsibilities for `check`:

1. Run SwiftLint.
2. Run fast unit tests or compile checks.
3. Exit non-zero on any failure.

Keep CI workflow command names aligned with this local adapter where possible.
