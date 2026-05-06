# Firecracker microVMs — Project Instructions

This is an upstream-tracking fork of [`firecracker-microvm/firecracker`](https://github.com/firecracker-microvm/firecracker), AWS's open-source VMM purpose-built for secure, multi-tenant serverless workloads. It powers AWS Lambda and AWS Fargate.

**Local status (2026-05-06): upstream-clean.** No AIBC-specific customizations have been applied. The repo is reserved as the future network-deny execution sandbox backend for [`SCRIM`](../SCRIM/) — see SCRIM's `tasks/prd-execution-network-sandbox.md` (pending). Until that PRD lands, treat this repo as upstream and avoid forking-incompatible changes.

## Tech Stack

- **Rust** 1.79.0 (pinned in `rust-toolchain.toml`, edition 2021)
- License: Apache 2.0
- Targets: `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl`
- Host requirements: Linux 5.10+ with KVM, io_uring 5.10.51+
- Guest support: x86_64 and aarch64 (Ubuntu 22.04 tested)
- Key crates: `kvm-bindings`, `kvm-ioctls`, `vm-memory`, `vm-superio`, `aws-lc-rs`, `seccompiler`, `linux-loader`, `event-manager`

## Build / Test / Run

All commands wrap through the Docker-based `tools/devtool` (Ubuntu 24.04 dev container, Rust 1.79.0, QEMU 8.1.1, musl tools).

```bash
./tools/devtool build                # debug build → build/cargo_target/<toolchain>/debug/firecracker
./tools/devtool build --release      # release build
./tools/devtool test                 # full pytest suite (Python integration tests)
./tools/devtool checkstyle           # rustfmt + cargo audit + black + isort + mdformat
./tools/devtool shell                # interactive shell in dev container
./tools/devtool sandbox              # interactive IPython REPL for VM interaction

# Specific test
./tools/devtool -y test -- integration_tests/performance/test_boottime.py::test_boottime

# Native Rust tests (not via Docker)
cargo test --test integration_tests --all

# A/B performance comparison
./tools/devtool -y test --ab run main HEAD --test integration_tests/performance/test_boottime.py::test_boottime
```

Pre-commit hook (`pre-commit` + `rusty-hook.toml`): runs `cargo audit`, `rustfmt`, `black`, `isort`, `mdformat` on staged files. Install via `cargo install rusty-hook && rusty-hook init`.

## Workspace Layout

```
src/
├── firecracker/        VMM binary entrypoint
├── vmm/                VMM library (arch, devices, io_uring, snapshot, mmds, gdb, logger)
├── jailer/             cgroup/namespace process jailer (statically linked)
├── seccompiler/        BPF seccomp filter compiler
├── cpu-template-helper CPU fingerprinting & template management
├── snapshot-editor     snapshot manipulation
├── utils/              shared library
├── log-instrument/     compile-time tracing
└── acpi-tables/        ACPI table generation

tests/
├── integration_tests/  pytest categories: functional, performance, security, style, build
├── framework/          test fixtures
└── data/               test data & images

tools/
├── devtool             primary build/test wrapper (Docker-based)
├── devctr/Dockerfile   dev container definition
├── ab_test.py          A/B perf testing
└── sandbox.py          interactive REPL

.buildkite/             Buildkite CI pipeline generators (Python)
docs/                   architecture, API, snapshotting, CPU templates, prod-host-setup
resources/seccomp/      seccomp filter JSON definitions
```

## Conventions

### Rust
- `rustfmt` is enforced; no manual formatting
- `clippy` runs strict: `undocumented_unsafe_blocks`, `cast_possible_truncation`, `tests_outside_test_module`, `error_impl_error` all warn-level
- **`unsafe` blocks require both `JUSTIFICATION:` and `SAFETY:` comments** — performance benefit and invariants
- Prefer `?`, `.map_err`, `.expect("rationale")` over `.unwrap()`
- Codegen units: 1 (better optimization), LTO on release, panic=abort

### Python (test infrastructure)
- `black` (line length 88), `isort` (multi-line 3, black profile), `pylint`
- Defined in `tools/devctr/pyproject.toml` (poetry)

### Markdown
- `mdformat` with GFM, footnotes, frontmatter; 80-char wrap

### Commits
- **DCO required** — every commit must be `git commit -s`
- Title ≤72 chars; body lines ≤72 chars; one logical change per commit
- `.gitlint` enforces; dependabot exempt

### Test markers
- `@pytest.mark.nonci` — separate scheduled pipelines only
- `@pytest.mark.no_block_pr` — optional PR tests, won't block merge
- No marker → required to pass for PR merge

## CI/CD

- **Buildkite** (primary): `pipeline_pr.py`, `pipeline_pr_no_block.py`, `pipeline_perf.py`, `pipeline_cross.py`, `pipeline_cpu_template.py`
- **GitHub Actions** (secondary): dependency-modification checks, A/B test triggers, release notifications
- Test environment: EC2 `.metal` instances (single-tenant for perf), Amazon Linux 2/2023 with KVM

## Sibling Repos in the AIBC Platform

| Repo | How it relates |
|---|---|
| `SCRIM` | **Planned consumer.** SCRIM's execution policy reserves a `FailedClosed(NetworkSandboxUnavailable)` path for the case where no OS sandbox is attested. Once `prd-execution-network-sandbox.md` lands, microVMs from this repo will provide the per-`scrim_exec` sandbox (vsock-injected secrets, network deny). Until then, this fork stays upstream-clean. |
| `claude-code-config`, `opencode_CUSTOM`, `claude_code_router`, `cc_gateway_project` | Orthogonal — different layers (agent harness, MCP routing, network proxy). No direct integration. |

## Important Notes for AI Assistants

1. **Do not add AIBC-specific code or branding** to this repo without explicit authorization. Customizations must be reversible and small enough to live alongside upstream sync.
2. **Safety-critical**: every `unsafe` block is reviewed; provide thorough justification.
3. **API stability**: changes to `src/firecracker/swagger/firecracker.yaml` trigger runbook procedures.
4. **Performance-sensitive**: A/B perf tests gate every PR on the upstream-tracking branch.
5. **Platform diversity**: maintain x86_64 and aarch64 parity.
6. **Formal verification**: Kani is integrated for selected critical modules (`tests/integration_tests/test_kani.py`).
7. **Platform conventions do NOT apply here.** This is an upstream-clean fork; do not add papa-git trailers, FYI.md, HANDOFF.md, or other AIBC-specific files. Cross-repo coordination happens elsewhere.

## Workflow

1. Setup: install Docker, rustup, poetry; `cargo install rusty-hook && rusty-hook init`
2. Build: `./tools/devtool build` (or `--release`)
3. Test: `./tools/devtool test`
4. Lint: handled by pre-commit hook; manual: `./tools/devtool checkstyle`
5. Shell/sandbox: `./tools/devtool shell` / `./tools/devtool sandbox`
