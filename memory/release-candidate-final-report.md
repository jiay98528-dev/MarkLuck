# MarkLuck 0.3.0-rc.1 Final RC Report

Date: 2026-06-24
Owner: Codex
Scope: M-R7 final release-candidate freeze, desktop RC risk closure, and gate verification.

## Executive Conclusion

M-R7 is at the final stop point, and the later desktop RC risk-validation pass has also completed. The Windows release candidate is acceptable for RC/beta release review with two documented environment blockers:

- WebKit E2E cannot run because the local Playwright WebKit executable is not installed.
- `cargo audit` cannot run because `cargo-audit` is not installed in this environment.

No product release blocker remains from the automated Chromium matrix, the repeated Firefox risk journey, the in-app browser GUI journey, or the installed Windows desktop GUI risk pass. After the initial RC report, a desktop external-file regression was fixed and revalidated: external `.md/.markdown/.mdx` files now open as a single-file read-only session, do not scan the parent folder, and save only the current file after explicit edit confirmation. The Tauri release bundle builds successfully and the NSIS installer has been copied to `打包/`.

## Final Code Changes In M-R7

- Split heavy Vue modal and export paths in `packages/app/src/pages/NotebookHome.vue` with async components.
- Replaced dynamic Tauri event import with a static `listen` import.
- Added Rollup manual chunks in `packages/app/vite.config.ts` for CodeMirror, export libraries, markdown libraries, Vue/Pinia, and Tauri API code.
- Fixed Windows Playwright web-server startup by using `pnpm.cmd` and a longer timeout in `packages/app/playwright.config.ts`.
- Added `e2e/report/` to `.gitignore` and removed tracked generated `e2e/report/index.html`.

## Automated Gate Matrix

| Gate                                                                                                                                                                                                                                   | Result  | Evidence                                                                                      |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `pnpm.cmd --filter @markluck/app typecheck`                                                                                                                                                                                            | PASS    | Completed before final GUI pass                                                               |
| `pnpm.cmd exec eslint packages/app/src packages/renderer/src`                                                                                                                                                                          | PASS    | Completed before final GUI pass                                                               |
| `pnpm.cmd exec prettier --check packages/app/src packages/renderer/src e2e spec memory README.md CHANGELOG.md RELEASE_NOTES.md KNOWN_LIMITATIONS.md package.json packages/app/package.json packages/app/vite.config.ts pnpm-lock.yaml` | PASS    | Completed after formatting historical docs/tests and removing tracked report artifact         |
| `pnpm.cmd --filter @markluck/app lint:style`                                                                                                                                                                                           | PASS    | Completed before final GUI pass                                                               |
| `pnpm.cmd --filter @markluck/app exec vitest run`                                                                                                                                                                                      | PASS    | 156/156                                                                                       |
| `pnpm.cmd --filter @markluck/app build`                                                                                                                                                                                                | PASS    | Build warning cleared; largest chunks split into vendor groups                                |
| `pnpm.cmd --filter @markluck/app exec playwright test --project=chromium --workers=1`                                                                                                                                                  | PASS    | 167/167 before final table padding tweak; table journey rechecked after final tweak           |
| `pnpm.cmd --filter @markluck/app exec playwright test e2e/tests/16-user-journeys.spec.ts --project=firefox --workers=1`                                                                                                                | PASS    | 10/10                                                                                         |
| `pnpm.cmd --filter @markluck/app exec playwright test --project=webkit --workers=1`                                                                                                                                                    | BLOCKED | Missing `C:\Users\m1771\AppData\Local\ms-playwright\webkit-2287\Playwright.exe`               |
| `pnpm.cmd audit --audit-level high`                                                                                                                                                                                                    | PASS    | High severity count is 0; remaining 2 low / 1 moderate                                        |
| `cargo fmt --check`                                                                                                                                                                                                                    | PASS    | Completed before final GUI pass                                                               |
| `cargo check`                                                                                                                                                                                                                          | PASS    | Completed before final GUI pass                                                               |
| `cargo audit`                                                                                                                                                                                                                          | BLOCKED | `error: no such command: audit`                                                               |
| `pnpm.cmd --filter @markluck/app tauri:build`                                                                                                                                                                                          | PASS    | Generated and copied `打包/MarkLuck-0.3.0-rc.1-windows-x64/MarkLuck_0.3.0-rc.1_x64-setup.exe` |

## Current Release Package

- Installer: `打包/MarkLuck-0.3.0-rc.1-windows-x64/MarkLuck_0.3.0-rc.1_x64-setup.exe`
- SHA256: `9b3d1f5fcec77996c1f8d5d046fe6724edda9baf425647bd848b68a7abcb8d8b`
- Windows registered note extensions: `.md`, `.markdown`, `.mdx`
- App-internal editable note extensions: `.md`, `.markdown`, `.mdx`, `.txt`

## Installed Desktop GUI Risk Validation

The installed Windows desktop build was risk-tested after the desktop RC fixes.

Passed journeys:

- Launching with a real external Markdown file opens a single-file read-only session and displays the target file content without WelcomePage, left wing, right wing, file drawer, search, tags, recent notebooks, or parent-folder scanning.
- Clicking "enable edit" shows a confirmation that only the current file will be edited; after confirmation, GUI typing saves back to that exact file and disk readback confirms the new content.
- File drawer filtering shows editable note files and hides ordinary non-note assets.
- `.txt` files can be opened inside MarkLuck without taking over the system `.txt` default association.
- Default onboarding/sample documents are present on first launch.
- Chinese IME input and immediate-mode table rendering in Live Preview do not corrupt block mapping, line decisions, or table column layout.
- Tab preserves native focus navigation in controls and only accepts ghost text when the editor owns focus.
- Welcome-page default-app guidance no longer claims Windows can silently set MarkLuck as the default app.

## Build Size Result

Final production build no longer emits the previous Vite chunk warning.

- `NotebookHome-MP6M65b4.js`: 206.82 kB, gzip 66.75 kB
- `vendor-codemirror-language`: 293.51 kB, gzip 107.07 kB
- `vendor-export`: 424.10 kB, gzip 120.93 kB

## In-App Browser GUI Verification

URL: `http://localhost:5173/`

Passed GUI-level journeys:

- App load and editor readiness.
- File drawer new note with unique filename `gui-rc-1782281017131.md`.
- Edit content with H1 `GUI-RC-1782281017131`, auto-save, and immediate left bookmark appearance.
- Reload, restore from left bookmark, and verify content persisted.
- File drawer subdirectory expansion, child file open, edit, and save.
- Search panel query, result jump, edit matched note, and save.
- Live Preview visible block click, edit, Escape restore, and save.
- Settings dialog, open "文字补全" section, toggle "启用幽灵文本补全" by Space and Enter.
- Theme toggle and refresh persistence.
- Export dialog, select TXT, trigger export, and observe success state.
- Delete temporary note, reload, verify file tree and left bookmark both have no residual entry.

GUI limitations and substitute evidence:

- In-app browser did not expose a real desktop drag/drop or binary clipboard path for image upload. Substitute evidence is automated `e2e/tests/16-user-journeys.spec.ts` J6, which verifies drag-drop event handling, Markdown `./assets/img_...png`, binary asset write, non-data-URL content, and hidden `assets` drawer entry.
- In-app browser download event capture timed out once, but GUI success state was visible and Playwright E2E reads exported TXT/XLSX files from disk.

## Environment Blockers

| Item                  | Status  | Required action outside this run                                                                        |
| --------------------- | ------- | ------------------------------------------------------------------------------------------------------- |
| WebKit E2E            | BLOCKED | Install WebKit with `pnpm.cmd exec playwright install webkit` or equivalent approved environment setup. |
| Rust dependency audit | BLOCKED | Install `cargo-audit`, then run `cargo audit`.                                                          |

## Residual Non-Blocking Risks

- NPM audit still reports 2 low and 1 moderate vulnerabilities, but the high-severity release gate is clean.
- Image upload final manual verification depends on host support for binary clipboard/desktop drag-drop; automated coverage has stronger file-layer assertions than this in-app browser session can provide.
- WebKit remains unexecuted on this machine until the browser runtime is installed.

## Git State At RC Stop

The worktree is intentionally dirty with accumulated release-hardening changes from M-R0 through M-R7. Notable generated-artifact cleanup:

- `package-lock.json` is deleted because `pnpm-lock.yaml` is the authoritative JS lockfile.
- `e2e/report/index.html` is deleted and `e2e/report/` is ignored.
- `test-results/` remains ignored.

## Final Recommendation

Proceed to RC/beta release review with the external-file hotfix included. Do not describe this as a fully stable signed release until WebKit and `cargo audit` are either executed or formally waived, and code signing/notarization is handled. If the user requires a fully green three-browser matrix, install WebKit and rerun the WebKit E2E gate first.

M-R7 已到达最终停止验收点，等待 Codex 最终发布审计。不得继续修改。

## Desktop External File Hotfix Addendum

Date: 2026-06-24/25 local validation window.

This section supersedes the earlier installed-desktop statement that external
file launch opened the parent folder as a notebook root.

Current authoritative behavior:

- System-level external launch accepts `.md`, `.markdown`, and `.mdx`.
- `.txt` remains app-internal only and is not accepted as a system launch
  format.
- External launch enters a single-file read-only session first.
- The session never calls notebook open, recursive file-tree load, index
  initialization, background training, search, tags, backlinks, or
  recent-notebook save for the parent directory.
- Enabling edit requires confirmation and saves only the currently opened file.
- Full read-only rendering uses the normal Markdown renderer with real
  `<table>` output.
- Immediate edit-mode table rendering uses grouped CSS-grid rows with shared
  column templates and class-based alignment.

Additional validation after the hotfix:

| Check                                                                 | Result                                                                                                              |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `pnpm.cmd --filter @markluck/app typecheck`                           | PASS                                                                                                                |
| `pnpm.cmd --filter @markluck/app exec eslint src --max-warnings=0`    | PASS                                                                                                                |
| `pnpm.cmd --filter @markluck/app exec stylelint "src/**/*.{vue,css}"` | PASS                                                                                                                |
| `pnpm.cmd --filter @markluck/app exec vitest run`                     | PASS, 156/156                                                                                                       |
| `cargo fmt --check` / `cargo check` / `cargo test`                    | PASS after Rust external-file command changes                                                                       |
| Chromium full E2E                                                     | PASS, 167/167 after main hotfix; table journey rechecked after final CSS padding tweak                              |
| Firefox `16-user-journeys`                                            | PASS, 10/10                                                                                                         |
| `pnpm.cmd --filter @markluck/app tauri:build`                         | PASS                                                                                                                |
| Installed desktop GUI                                                 | PASS: double-click/read-only open, edit confirmation, GUI save/disk readback, no parent scan, table visual verified |

Final package:

- Installer:
  `打包/MarkLuck-0.3.0-rc.1-windows-x64/MarkLuck_0.3.0-rc.1_x64-setup.exe`
- SHA256:
  `9b3d1f5fcec77996c1f8d5d046fe6724edda9baf425647bd848b68a7abcb8d8b`
