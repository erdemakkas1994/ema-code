# First EMA feature: chat groups

Status: design only, on `ema-chat-groups`. No runtime implementation or schema changes have been made.

## User behavior

The user can create groups such as Company, Personal and Experiments; rename and reorder them; and move a chat into one group. Unassigned chats remain in a visible Ungrouped section. Deleting a group returns its chats to Ungrouped and never deletes, archives or moves an OpenCode session or project on disk. The upstream session archive continues to mean archive, not a group named Archive.

For the first release, one chat belongs to at most one group. Groups can span projects, with the project still visible on each row. Search and existing session navigation remain available. No provider or server API changes are part of this feature.

## What the current source actually supports

- `packages/gui-extensions/src/sdk/` is the existing extension SDK; do not build another framework.
- `packages/gui-extensions/src/renderer.ts` composes window extensions; `src/main.ts` composes main-process extensions.
- `Store.global`/`Store.main` and extension namespaces already handle persistence. Desktop persistence uses the existing storage infrastructure; do not add an independent JSON file or database.
- `ctx.sessions.list()` exposes sessions in OPEN shell tabs. It is not a full historical chat catalog and must not be used as if it were one.
- `SlotMap` currently exposes `window.bottom`, `session.header`, `session.panel.end`, and `session.panel.sidebar`. The latter is a session side-panel slot, not the main historical chat list.
- The history UI lives in `packages/app/src/home/sessions/{controller,region,view}.tsx`; projects have a separate `home/projects` area. There is no verified general hook for replacing/grouping the full home list.

Consequently, a promise of zero upstream UI edits would be inaccurate. Start by adding one narrow generic grouping contribution to the host, with a documented SDK contract and default behavior identical to upstream when absent. The host must not know the EMA extension id, storage keys or business rules.

## Proposed ownership

Add `packages/gui-extensions/src/ema-chat-groups/`:

```text
index.ts          Extension.define, stores, i18n
contract.ts       shared grouping contract and schemas
renderer.tsx      commands, settings, list contribution
groups.ts         pure grouping operations
groups.test.ts    behavior contracts
i18n/en.ts        source copy; localization follows repository policy
```

Use the existing host persistence before considering a main entry. Add `main.ts` and a `Store.main` only if inspection proves cross-window writes cannot be coordinated by the existing global storage path. Do not duplicate state in main and renderer or introduce a second persistence owner.

Minimal upstream touch points: renderer composition, the generic SDK contract, and the home session-list adapter. Main composition changes only if a main entry is required. An SDK addition also requires its TSDoc, author-guide update and contract validation under the existing lint rules.

## Stored data, version 1

Namespace: extension `ema-chat-groups`, store `groups` (host-managed `extension.ema-chat-groups.*` keys). Shape:

```typescript
{
  version: 1,
  groups: [{ id: "stable-generated-id", name: "Company", order: 0 }],
  assignments: [{ server: "stable-server-identity", session: "session-id", group: "group-id" }]
}
```

Use the stable host server identity plus session id; never use a title, project path, temporary service port or an array index as identity. Confirm server identity stability across reconnects before implementing. Do not store authentication credentials or message contents. Validate stored unknown data at the store boundary. Missing/deleted groups map to Ungrouped. Preserve assignments when a server is temporarily offline; a paginated list omitting a session does not prove deletion. Persist an explicit migration version and retain earlier values until migration succeeds.

## Small delivery steps

1. Record a production history/timeline benchmark baseline as required by `packages/app/AGENTS.md`; do this before session UI changes.
2. Implement the store/schema and pure operations; cover create, rename, move, group deletion without session deletion, invalid group references and migration.
3. Add the generic list grouping contract. Verify the extension-disabled path preserves the existing list, pagination and search. Avoid duplicating the existing history query logic.
4. Compose the EMA extension and add group controls to the existing list. Use the host i18n and async storage loading contracts; keep list rendering free of accidental Suspense.
5. Exercise persistence after close/reopen, two-window updates, duplicate session IDs on different servers, offline servers and large histories. Extend the owning existing tests instead of duplicating coverage.
6. Compare production benchmark results, run package checks and `bun run check`, then make the feature commit. Keep the current design-only baseline available for rollback.

## Acceptance gates

- Group CRUD and assignment persist after reopening the isolated desktop.
- Deleting or disabling groups loses no chats and does not alter the upstream session database.
- All sessions remain reachable through grouping, search and pagination.
- Original OpenCode profile fingerprints remain unchanged during development checks.
- Upstream merge/rebase is tested on a disposable update branch; conflicts are limited to the named adapters/composition points.
- Packaging/branding and remote control remain separate changes with separate validation.
