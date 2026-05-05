# ohmu-runner

CLI tool that orchestrates multi-target build, test, and screenshot workflows. Lives in `tools/ohmu-runner/`. Not a library — a standalone binary.

## What it does

Reads target configuration files and runs the appropriate build/test/deploy sequence for each target. Abstracts away the per-platform mechanics (cross-compilation flags, device deployment, result collection) behind a uniform interface.

## Commands

```bash
# Run tests by tier
ohmu-runner test --tier 1                          # pure logic, all targets
ohmu-runner test --tier 2                          # GPU headless, current machine
ohmu-runner test --tier 3 --target android         # full integration, specific target
ohmu-runner test --all                             # everything, including remote

# Build for specific targets
ohmu-runner build --target web --release
ohmu-runner build --target android --target ios

# Screenshot collection (visual regression baseline)
ohmu-runner screenshot --target android --profile phone-portrait
ohmu-runner screenshot --all-profiles              # all devices × all profiles

# Check which targets are available
ohmu-runner targets                                # lists configured targets + status
```

## Target configuration

Target configs live in `targets/` at the workspace root. Each describes how to build, deploy, and test for that target.

```toml
# targets/android.toml
[build]
toolchain = "ndk"
targets = ["arm64-v8a", "armeabi-v7a"]
features = ["mobile"]

[deploy]
method = "adb"
install_path = "/data/local/tmp/ohmu-test"

[test]
tier1 = { crates = ["ohmu-core", "ohmu-dsl"] }
tier2 = { crates = ["ohmu-gpu"], features = ["gpu-tests"] }
tier3 = { command = "adb shell am instrument ..." }

[screenshot]
capture = "adb shell screencap -p"
pull = "adb pull"
```

```toml
# targets/web.toml
[build]
toolchain = "wasm-pack"
target = "web"

[test]
tier1 = { runner = "wasm-pack test --headless --chrome" }
tier2 = { runner = "wasm-pack test --headless --chrome", features = ["gpu-tests"] }
tier3 = { runner = "playwright test" }
```

```toml
# targets/remote/macos-arm.toml
[connection]
host = "mac-mini.local"
user = "builder"
key = "~/.ssh/id_ed25519"

[build]
local_cross = false  # build on remote machine
remote_command = "cd /tmp/ohmu && cargo test"

[deploy]
method = "rsync"
dest = "/tmp/ohmu/"

[test]
sync_sources = true  # rsync source tree before building
```

## How it works

### Tier 1 (local, fast)

Just runs `cargo test` for the pure-logic crates. Optionally cross-compiles to verify WASM and ARM targets compile:

1. `cargo test -p ohmu-core -p ohmu-dsl` — native.
2. `cargo test -p ohmu-core --target wasm32-unknown-unknown` — WASM compilation check.
3. `cross test -p ohmu-core --target aarch64-linux-android` — ARM compilation check.

### Tier 2 (local, GPU)

Runs `cargo test -p ohmu-gpu --features gpu-tests` with `WGPU_BACKEND=vulkan` (or `gl` for software rendering). On CI, Mesa llvmpipe provides the software GPU.

### Tier 3 (per-target, may be remote)

For each target:

1. **Build** — cross-compile or build on remote, per target config.
2. **Deploy** — push binary/app to device or remote machine.
3. **Execute** — run test harness, capture stdout/stderr/exit code.
4. **Collect** — pull screenshots and test results back to local.
5. **Report** — aggregate pass/fail across all targets, show visual diffs.

### Result aggregation

```
ohmu-runner test --all

Target          Tier 1    Tier 2    Tier 3
─────────────────────────────────────────
native          142/142   38/38     12/12
web/wasm        142/142   38/38     —
android         142/142   —         8/8
ios-sim         142/142   —         6/6
remote/macos    142/142   38/38     12/12

Screenshots: 3 diffs detected (see target/screenshots/diff/)
```

## When to build this

Don't build it upfront. Development order:

1. Start with `cargo test` and manual commands.
2. Add shell scripts when you have two targets and the manual commands get tedious.
3. Build `ohmu-runner` when you have three+ targets and the shell scripts aren't scaling.

The target config format can be designed early (it's just TOML), but the runner binary should emerge from actual pain points, not speculation.

## Dependencies

- `clap` — CLI argument parsing.
- `toml` — target config parsing.
- `tokio` — async execution for parallel target testing.
- SSH/ADB/simctl invoked as subprocesses, not linked libraries.
