# EMA Code

EMA Code is an independent personal fork of [OpenCode](https://github.com/anomalyco/opencode), not affiliated with the OpenCode team. Upstream's MIT license and copyright notices remain applicable.

## Baseline (20 October 2026)

- Official source: `https://github.com/anomalyco/opencode.git`, default branch `dev`.
- Pinned baseline: `d8148796dc74a342c96c7b4febf03dc664e1058c`.
- Root and desktop package version: `2.1.0` (source package version, not an assertion about the latest stable release).
- Desktop: Electron `44.4.5`, electron-vite `6.0.0-beta.1`, SolidJS `1.9.15`.
- Runtime: the repository pins Bun `1.4.2`; Node `24.13.1` is available on the Windows machine.
- Official commands: root `bun install --frozen-lockfile`, then `bun run dev:desktop`.
- Packaging is a separate step: production builds require a CLI distribution from the same source/release. No installer has been built or installed in this setup.

Read `CONTRIBUTING.md`, `packages/desktop/README.md`, and the applicable `AGENTS.md` before changes. Rust is not a prerequisite for this Electron development workflow.

## Git layout

- `upstream`: official OpenCode, fetch updates here.
- `origin`: `https://github.com/erdemakkas1994/ema-code.git`.
- `ema-base`: untouched baseline commit, retained for comparison.
- `ema-chat-groups`: initial design documents only; no application behavior changes.

After this baseline is verified, update through a short-lived branch, never directly in the feature branch:

```powershell
git fetch upstream
git switch -c ema-update ema-chat-groups
git rebase upstream/dev
```

Resolve conflicts and review the diff; do not force-push a shared branch. Run the checks below and smoke-test an isolated profile before promoting the update. Preserve the previous verified commit/tag for rollback. The feature branch must remain a small set of EMA commits on upstream; do not copy upstream source into a parallel implementation.

## Safe Windows development

This checkout uses a workspace-local Bun installation and an external `outputs/start-ema-dev.mjs` launcher, one directory above this repository. From the parent workspace:

```powershell
node .\outputs\start-ema-dev.mjs
```

The launcher starts the official command with a separate `work/ema-profile` home, database, config, Electron profile, temporary directory and service registration. It disables `opencode://` protocol registration. It does not copy original API keys or chat history. Development retains OpenCode's upstream name and branding intentionally for the baseline.

The built-in isolated development server uses port 3084 and binds `0.0.0.0`; the desktop connects through loopback using upstream's authentication. Debugging uses port 9333. These are upstream development behaviors; remote access is not an EMA feature yet. Do not launch another instance on those ports.

Windows' known Documents folder is an OS setting and is not redirected by USERPROFILE. The upstream first-launch flow can create a default project there. The observed baseline did so; pre-existing files were not removed or replaced. The external launcher now seeds only a new isolated profile's `firstLaunchOnboardingComplete` setting, preventing this first-launch side effect in future fresh profiles. Select a project inside the workspace.

## Verification

```powershell
# Repository root; canonical lint and type checks
bun run check

# Desktop package; existing contracts
cd packages/desktop
bun typecheck
bun test --timeout 30000
```

Run the tests outside the restricted agent sandbox, with temporary files in the workspace. The first sandbox run encountered local networking/file-operation restrictions; the native Windows repeat passed all 131 desktop tests.

The repository tracks 43 symbolic links, including `packages/app/src/custom-elements.d.ts`. A Windows clone with `core.symlinks=false` writes the link target path as plain text, causing TS1128. This was resolved after the user enabled Windows Developer Mode: the local clone now has `core.symlinks=true`, and all tracked links were checked out as real symbolic links. No source workaround or duplicate files were committed. Desktop typecheck passes. See the external setup report for the final full-repository check status.

## Next change

See [EMA-CHAT-GROUPS.md](EMA-CHAT-GROUPS.md). Keep all branding, provider, remote-control and installer work separate from the first chat-groups feature.
