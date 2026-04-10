# Latchkey Fast Cache

High-performance streaming cache for [Latchkey](https://github.com/latchkey-dev) managed runners.

Uses a single streaming HTTP request instead of the serial chunk pattern used by `actions/cache@v4`, eliminating Node.js bootstrap, reserve/commit API overhead, and serial 32MB PATCH round trips.

On non-Latchkey runners, the cache proxy is not available and this action will not work. Use `actions/cache@v4` instead.

## Usage

```yaml
jobs:
  build:
    runs-on: [self-hosted, latchkey-medium]
    steps:
      - uses: actions/checkout@v4

      - uses: latchkey-dev/cache-action@v1
        id: cache
        with:
          action: restore
          key: deps-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          path: node_modules

      - run: npm ci
        if: steps.cache.outputs.cache-hit != 'true'

      # ... build steps ...

      - uses: latchkey-dev/cache-action@v1
        if: steps.cache.outputs.cache-hit != 'true'
        with:
          action: save
          key: deps-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          path: node_modules
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `action` | `"save"` or `"restore"` | **Yes** | |
| `path` | Path(s) to cache (newline or space separated) | **Yes** | |
| `key` | Cache key | **Yes** | |
| `cache-url` | Cache proxy base URL | No | `http://localhost:8080` |

## Outputs

| Output | Description |
|--------|-------------|
| `cache-hit` | Whether the cache was restored successfully (`true`/`false`) |

## How It Works

1. **Restore**: Checks cache existence (HEAD), then streams a `GET` through `zstd -d` decompression and `tar` extraction
2. **Save**: Checks if cache already exists (HEAD request), then streams `tar | zstd | curl PUT` in a single pipeline — no intermediate files
3. **Cache isolation**: Keys are versioned by `RUNNER_OS` automatically. For finer isolation (e.g., different Ubuntu versions), encode it in your cache key

The cache proxy runs on each Latchkey managed runner as a local service backed by S3. It provides a V2 streaming API that bypasses the reserve/chunk/commit pattern of GitHub's cache, achieving significantly higher throughput through:

- Single streaming HTTP request (no round trips)
- Multi-threaded zstd compression and decompression (`-T0`)
- Parallel S3 multipart uploads (16 concurrent 8MB parts)
- Parallel S3 range-request downloads (16 concurrent 64MB parts)
- Data locality — S3 in the same region as the runner

## License

MIT
