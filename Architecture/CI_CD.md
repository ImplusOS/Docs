# CI/CD — ImplusOS

*Last reviewed: 2026-08-29 (phase P8 of `Docs/Others/TODO_OS_Refactor.md`, lowest
priority — planned last, once P1–P7 were done, per the project owner's explicit
decision that CI/CD isn't very important for this project).*

> **Current status (2026-08-29): the `.github/` directory is not committed to
> this tree.** The workflow files described below (`build.yml`,
> `boot-smoke-test.yml`, `docs-lint.yml`, `static-analysis.yml`) and the
> `.markdownlint.jsonc` do not exist here yet — this document is the design
> spec for them. What *is* already wired in is the build-side hook:
> `Kernel/config/arch.mk` accepts `make CI=1 ...` to promote warnings to
> `-Werror` (§3), and its comments reference the intended `build.yml`. Adding
> the actual workflow files is the remaining P8 work.

## 1. Workflows

| File | Runs on | What it checks |
|---|---|---|
| `.github/workflows/build.yml` | push to `main`, every PR | `make ARCH={x86_64,arm64} all image_livecd` compiles and links cleanly |
| `.github/workflows/boot-smoke-test.yml` | push to `main`, every PR | The x86_64 image actually boots to the end of its documented sequence (`Docs/Architecture/Boot_Sequence.md`) in QEMU, without panicking or hanging |
| `.github/workflows/docs-lint.yml` | push/PR touching `Docs/**/*.md` | `markdownlint-cli2` against `Docs/**/*.md`, using the lenient `.markdownlint.jsonc` at the repo root |
| `.github/workflows/static-analysis.yml` | push to `main`, every PR | `cppcheck` over `Kernel/`, `Userland/`, `libc/I_libc/`; findings uploaded as an artifact, non-blocking for now (see below) |

## 2. Toolchain: Homebrew, not a GHCR image

`Docs/Others/TODO_OS_Refactor.md`'s own P8 research recommended a pre-built
Docker image on GHCR (deterministic, fast repeat CI) over building
`x86_64-elf-gcc`/`aarch64-elf-gcc` from source on every run. That wasn't
implemented here — publishing and maintaining a GHCR image needs registry
credentials this repository's CI setup pass didn't have — but building from
source was also avoided, because it means multi-minute GCC builds even with
caching, and the plan's own analysis flagged Ubuntu's `apt install
gcc-x86-64-elf` as fragile (it depends on Debian/Ubuntu-specific packaging
that can break across runner image updates).

The middle ground taken: install the same **Homebrew formulae** this project
already uses for local development (`x86_64-elf-gcc`, `x86_64-elf-binutils`,
`aarch64-elf-gcc`, `aarch64-elf-binutils` — verify what's in use locally with
`brew list | grep elf`). Homebrew formulae are versioned, reproducible
independent of the Ubuntu runner's own package repositories, and
`actions/cache` keyed on `/home/linuxbrew/.linuxbrew` keeps repeat runs from
re-installing every time. If GHCR access becomes available later, replacing
this with a pre-built image remains a clean, self-contained follow-up (only
the "Install Homebrew" / "Install cross-compiler toolchain" steps in
`build.yml` and `boot-smoke-test.yml` would change).

## 3. `-Werror` exists but is off by default

`Kernel/config/arch.mk` supports `make CI=1 ...` to promote every warning in
`KERNEL_CFLAGS` (`-Wall -Wextra -Wtype-limits -Wconversion -Wsign-conversion
-Wshadow`) to a hard error. `build.yml` does **not** pass `CI=1`: as of this
CI setup pass, the kernel tree has several pre-existing warnings (mostly
`-Wunused-function`/`-Wunused-variable` and a few `-Wconversion`/
`-Wsign-conversion` spots) predating every phase of `Docs/Others/
TODO_OS_Refactor.md` and out of scope to fix as part of it. Enabling `CI=1`
before those are cleaned up would make the very first CI run fail for
reasons unrelated to whatever change triggered it — a good, contained
follow-up task once someone does that cleanup pass (`make ARCH=x86_64 CI=1
kernel 2>&1 | grep warning:` finds all of them at once).

## 4. `cppcheck`, not `clang-tidy`

`clang-tidy` needs a `compile_commands.json` built against the real
cross-compiler flags to avoid a flood of freestanding/no-libc false
positives (missing `<stdio.h>`, no C runtime, etc.) — generating and
maintaining that was judged more setup than P8's "implement if there's time
left over" budget covers. `cppcheck` runs standalone against the source tree
with no compile step and no cross-compiler dependency at all, which is
enough to catch the classes of bug `Docs/Others/TODO_OS_Refactor.md` phase
P3 (kernel robustness) already cares about — null derefs, uninitialized
reads, integer overflow — without that setup cost. `static-analysis.yml`
does not fail the build on findings yet (`--error-exitstatus` is
deliberately not passed): this is a first pass at wiring it in at all, and
this codebase's actual finding count against it is unknown. Once a baseline
run has been reviewed and either fixed or suppressed (a
`.cppcheck-suppressions` file), flipping that on is a small follow-up.

## 5. Not yet done: branch protection

GitHub branch protection rules (`Settings → Branches → Branch protection
rules`, requiring `build.yml` and `boot-smoke-test.yml` to pass before a PR
can merge to `main`) **cannot be configured from a commit or a workflow
file** — it's a repository-settings change, not a code change, and this
refactor's plan (`Docs/Others/TODO_OS_Refactor.md` 2.2) explicitly calls out
that it needs a repository admin to apply it manually. Recommended settings,
once the workflows above have had a few successful real runs to confirm
they're stable:

1. **Settings → Branches → Add branch protection rule**, pattern `main`.
2. Enable **"Require status checks to pass before merging"**, and select the
   `build (x86_64)` and `boot-smoke-test` check names (they only appear in
   the list after each workflow has run at least once).
3. Leave `build (arm64)` **out** of the required list for now — see
   `build.yml`'s own comment on the pre-existing, unrelated arm64 link gap;
   requiring a check that's known to currently fail would block every PR.
4. `docs-lint`/`static-analysis` are informational for now (§3, §4) — leave
   them out of the required list until `-Werror`/`--error-exitstatus` are
   actually turned on for them.
