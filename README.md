# Eclipse ThreadX USBX – Middleware (Azure DevOps staging)

Internal Azure DevOps mirror of **Eclipse ThreadX USBX** (upstream: [eclipse-threadx/usbx](https://github.com/eclipse-threadx/usbx)), imported here for testing before being exported as a public middleware module to GitHub.

This is the **import** side only — see `ThreadX_AzureImport.sh --component usbx` in `Middlewares_Handler`. The export step (Azure DevOps → GitHub, with its own testing/approval logic) is separate and not yet built.

This file itself is regenerated (rendered from `README_ThreadXDefault.md`) by every import run — don't hand-edit it here, edit `README_ThreadXDefault.md` next to `ThreadX_AzureImport.sh` instead.

---

## Branch & repository layout

Everything lives on a single anchor branch (`master`). One real commit per Eclipse ThreadX USBX version, tree replaced wholesale each time, commit subject naming its **normalized** version (e.g. `USBX 6.4.2` — see Versioning below), commit body naming the raw upstream tag. No git tags are created here — this is the Azure DevOps staging side only; tagging happens later, during the separate export step that promotes a tested/approved version to the public GitHub middleware repo.

```text
<repo root>
├── CMakeLists.txt   auto-generated every version (see below) — do not hand-edit
├── README.md        this file
└── USBX/      the release as upstream tags it, resolvable submodules vendored
                     in place, no .git / .gitmodules / .gitignore anywhere
    ├── CMakeLists.txt     upstream's own — defines the `usbx` library target
    ├── common/            the portable component source
    ├── ports/             per-architecture / per-toolchain ports (where the component has any)
    └── ...
```

## Source

`ThreadX_AzureImport.sh` discovers versions via the GitHub **Releases** API (drafts and prereleases ignored). Each version is fetched with a shallow `git clone --branch <upstream tag>`, the checked-out tag is verified to be exactly that tag, and the resulting tree is copied into `USBX/` as-is.

## Versioning

Upstream `eclipse-threadx/usbx` tags are **not** a clean `X.Y.Z` — they carry an extra build tag and/or an `_rel`/letter suffix, and the format has changed over time. `ThreadX_AzureImport.sh` normalizes every upstream tag before importing:

| Upstream tag example | Normalized version |
|---|---|
| `v6.4.1_rel` | `6.4.1` |
| `v6.4.2.202503_rel` | `6.4.2` (build tag `202503` dropped) |
| `v6.4.4.202503a` | `6.4.4` |
| `v.6.4.4.202503_rel` | `6.4.4` (a "v." typo variant, seen on some netxduo tags) |
| `v6.0_rel` | `6.0.0` (X.Y only — patch synthesized as `0`) |

Where two upstream tags normalize to the **same** version (e.g. a build-tag revision that only bumped internal metadata), the release with the **later** publish date wins — the older one is not imported at all.

## Submodules

`eclipse-threadx/netxduo` references at least one test-only submodule in its own `.gitmodules` that has been observed broken in some releases (see upstream issue [netxduo#310](https://github.com/eclipse-threadx/netxduo/issues/310)). Submodule resolution here is therefore **best-effort**: a failure is logged as a warning, not treated as a reason to skip the version — what matters for a middleware consumer is the library source, not upstream's own test scaffolding. Every submodule that *did* resolve is fully flattened: **every** `.git` entry, `.gitmodules` and `.gitignore` file is stripped before committing — no submodules, no gitlinks, no dependency on any other repository.

## CMake support requirement — versions without a usable target are not imported at all

The generated root `CMakeLists.txt` only wraps upstream's own `USBX/CMakeLists.txt`, so a version is imported **only** if that file exists and defines the `usbx` library target (`add_library(usbx ...)`, or `project(usbx ...)` + `add_library(${PROJECT_NAME} ...)` as current upstream does). Anything else is **skipped entirely — no commit, nothing published for it in this repo.**

---

## Building against it — the generated `CMakeLists.txt`

The root `CMakeLists.txt` in this repo does **not** invent any library — it just `add_subdirectory()`s `USBX/`, whose upstream `CMakeLists.txt` defines the `usbx` target (plus its `azrtos::usbx` alias).

Upstream requires two variables to be set **before** the module is added — they select the port under `USBX/ports/<arch>/<toolchain>/`:

- `THREADX_ARCH` — e.g. `cortex_m0`, `cortex_m4`, `cortex_m7`, `cortex_m33`, ...
- `THREADX_TOOLCHAIN` — e.g. `gnu`, `ac6`, `iar`.

Vendor this repo (e.g. as a submodule at `Middlewares/USBX`), then from **your own** top-level `CMakeLists.txt`:

```cmake
set(THREADX_ARCH "cortex_m4")
set(THREADX_TOOLCHAIN "gnu")

add_subdirectory(Middlewares/USBX)
target_link_libraries(${PROJECT_NAME}
    PRIVATE
        usbx
)
```

### Notes on the upstream target

- **Dependencies between components.** Every component other than ThreadX itself links against `azrtos::threadx` (and some, e.g. NetX Duo and LevelX, optionally against `azrtos::filex`) — add the ThreadX module first, then the components that depend on it.
- **User configuration header.** Each component looks for an optional `<component>_user.h` (e.g. `tx_user.h`, `nx_user.h`) — enable it through upstream's own CMake options / defines (see the component's upstream documentation).
- Upstream's file may also declare `install()` rules — harmless unless you run `cmake --install` on your own project.

---

## Notes

- A brand-new repo (or an existing repo that's just missing the `master` branch, e.g. one still carrying the old `main`/`Content` layout) gets a real `Initial commit` containing just this `README.md`, pushed immediately when the branch is created — so it's never left with zero commits even if the first version afterwards fails or every available version is skipped. The first version actually imported then wholesale-replaces that with the full tree, same as every later version. Other branches are left untouched.
- No git tags are created by this importer — tagging happens later, during the separate export step.
- Re-running the import is idempotent — a version already present as a commit on `master` (matched by its commit subject, `USBX <version>`) is skipped.

---

## Authors

- **Mr.Nobody** — [embedbits.com](https://embedbits.com)
