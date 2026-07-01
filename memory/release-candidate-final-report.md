# MarkLuck v0.15.0-rc.1 Final RC Report

Date: 2026-06-30
Owner: Codex
Scope: M-R7 final release-candidate freeze, desktop RC risk closure, WebKit
rapid-switching fix, release gate verification, and final external audit.

## Executive Conclusion

The WebKit rapid note-switching blocker has been closed. The final independent
audit reports Chromium, Firefox, and WebKit E2E as passing, along with L1/L2,
coverage, build, Rust fmt/check/test, Tauri packaging, and GUI smoke.

The current candidate may proceed to real-installer L4 review. It must not be
described as final public release until the installed-app L4 report is complete
and Rust audit evidence is attached from CI or a local `cargo-audit` run.

## Final Code Changes In M-R7

- Split heavy Vue modal and export paths in `packages/app/src/pages/NotebookHome.vue`
  with async components.
- Replaced dynamic Tauri event import with a static `listen` import.
- Added Rollup manual chunks in `packages/app/vite.config.ts` for CodeMirror,
  export libraries, markdown libraries, Vue/Pinia, and Tauri API code.
- Removed generated Playwright report tracking and ignored `e2e/report/`.
- Closed WebKit rapid note-selection races by guarding asynchronous note
  selection with a monotonic version.

## Automated Gate Matrix

| Gate                                             | Result  | Evidence                                                 |
| ------------------------------------------------ | ------- | -------------------------------------------------------- |
| `pnpm typecheck`                                 | PASS    | Final external audit                                     |
| `pnpm lint`                                      | PASS    | Final external audit                                     |
| `pnpm --filter @markluck/app lint:style`         | PASS    | Final external audit                                     |
| `pnpm format`                                    | PASS    | Final external audit                                     |
| `pnpm test`                                      | PASS    | 187/187                                                  |
| `pnpm test:coverage`                             | PASS    | 80% threshold active; core whitelist over threshold      |
| `pnpm --filter @markluck/app build`              | PASS    | Final external audit                                     |
| Chromium E2E                                     | PASS    | 140 passed                                               |
| Firefox E2E                                      | PASS    | 137 passed / 3 skipped                                   |
| WebKit E2E                                       | PASS    | 137 passed / 3 skipped                                   |
| WebKit rapid note switching                      | PASS    | repeat=5 passed                                          |
| `pnpm audit --audit-level high`                  | PASS    | No high-or-higher advisories                             |
| `cargo fmt --check && cargo check && cargo test` | PASS    | Rust tests 11/11                                         |
| `pnpm --filter @markluck/app tauri:build`        | PASS    | Generated `MarkLuck_0.15.0_x64-setup.exe`                |
| `pnpm audit:rust`                                | PENDING | Requires CI green evidence or local `cargo-audit` output |

## Current RC Package Identity

- App version: `0.15.0`
- Release channel: `v0.15.0-rc.1`
- Installer:
  `packages/app/src-tauri/target/release/bundle/nsis/MarkLuck_0.15.0_x64-setup.exe`
- SHA256: must be recorded in the installed-app L4 report for the exact
  published build.
- Windows registered note extensions: `.md`, `.markdown`, `.mdx`
- App-internal editable note extensions: `.md`, `.markdown`, `.mdx`, `.txt`

## GUI Verification

The final external audit completed a GUI smoke journey in the browser:

- Created a unique temporary note.
- Typed heading and body.
- Confirmed bookmark creation.
- Rapidly switched between existing notes and the temporary note.
- Returned to the temporary note and appended content.
- Confirmed auto-save.
- Deleted the temporary note through the file drawer.
- Confirmed no bookmark or body residual entry remained.

Installed-app L4 is still required before public release. It must cover the
paths defined in `doc/release-rc-gate.md` and use
`doc/release-installed-l4-template.md` as the report template.

## Residual Non-Blocking Risks

- Local Rust audit remains unverified if `cargo-audit` is not installed. CI may
  satisfy this gate if the job URL, commit, pass time, and status are recorded.
- Coverage threshold is active for the current core services/tools whitelist,
  not the entire UI surface. Three-browser E2E remains the primary UI gate for
  this RC.
- The current Windows installer is unsigned. Code signing remains a release
  operations item before describing the package as a fully stable signed
  release.

## Final Recommendation

Proceed to real-installer L4 review for `v0.15.0-rc.1`. Do not describe the
candidate as final public release until `pnpm release:rc-gate` passes with a
completed installed-app L4 report and Rust audit evidence.
