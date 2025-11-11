# Cargo Configuration

This directory contains Cargo configuration that applies to all developers working on this project.

## sccache Setup

This project uses [sccache](https://github.com/mozilla/sccache) to cache compiled dependencies, dramatically reducing build times.

### Installation

```bash
cargo install sccache
```

### How it works

The [config.toml](config.toml) file automatically enables sccache for all cargo builds in this repository. No additional configuration is required!

### First build vs. subsequent builds

- **First build**: Normal speed (~20-30 seconds for simple crates, longer for full workspace)
- **Subsequent clean builds**: Much faster thanks to cached dependencies

### Verify it's working

Check sccache statistics:
```bash
sccache --show-stats
```

You should see:
- `Compile requests` increasing during builds
- `Cache hits` on subsequent builds
- `Cache misses` on first build or new dependencies

### Optional: Shared build directory (faster builds across projects)

Add to your `~/.zshrc` or `~/.bashrc`:
```bash
export CARGO_TARGET_DIR=$HOME/.cargo/shared-target
export SCCACHE_DIR=$HOME/.cache/sccache
```

This shares build artifacts and sccache across all Rust projects on your machine.

### Troubleshooting

If sccache isn't working:

1. **Check if sccache is installed**: `which sccache`
2. **View detailed stats**: `sccache --show-stats`
3. **Clear cache**: `sccache --stop-server` (restarts automatically on next build)
4. **Enable debug logging**: `export SCCACHE_LOG=debug`

### Non-cacheable compilations

Some crates can't be cached (typically 2-5 per workspace):
- Vendored dependencies (in `vendored/` directory)
- Procedural macros
- Crates with build scripts that reference external files

This is normal and expected. Most dependencies from crates.io will cache properly.
