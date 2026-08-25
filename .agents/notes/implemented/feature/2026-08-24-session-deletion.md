# Agent Note: Permanent session deletion and the archived-sessions view

Status: implemented

English | [中文](2026-08-24-session-deletion.zh.md)

## Problem

The GUI could hide a session (Archive) but never remove one: archiving keeps the session log and the workspace accounting slot, and no surface could list archived sessions or restore them. Deleting a session — the durable log included — did not exist anywhere in the stack: `SessionPersistence` was strictly append-only, the workspace registry had no destructive session operation, the Host exposed no delete RPC, and the browser sidebar had no archived-sessions mode. Users accumulating archived sessions had no way to reclaim that history or free the disk space it occupied.

## Decision

### Host: durable removal is an explicit administrative operation

`SessionPersistence.delete(id)` is the one sanctioned exception to the append-only contract, and it is invoked only by the workspace registry's `deleteSession`, never by the write path. It resolves `true` when a durable artifact was removed and `false` for an unknown id; a lazily-created, never-appended id leaves the coordinator state so the id can be created again. The coordinator refuses an identity still bound to the live SessionStore, waits for any in-flight retirement of the identity to settle (the `session/disposed` final drain), drops the per-session state and retained preparations, and then calls the backend's new `deleteStored` hook. Backends implement physical removal: JSONL removes the log file and its per-session directory (best-effort sweep, deletion is retryable), SQLite deletes the session row with its cascade of event rows.

`ctx.workspaceRegistry.deleteSession(id)` removes the session's accounting slot from every workspace record, drops it from the archive set, and calls `SessionPersistence.delete`. A session still bound to the live SessionStore rejects with `WorkspaceLiveSessionError` — the caller stops a live agent first. The deleted identity also leaves the registry's header index so a later archive of the same id cannot succeed against a ghost. `ctx.workspaceRegistry.unarchiveSession(id)` is the archive-set inverse: it restores the session to every grouping surface, idempotent for ids outside the set.

The Host gateway (`workspace.deleteSession` RPC) stops the live agent through the new `AgentRegistry.stop(sessionId)` before the registry runs. `stop` is the administrative counterpart to the owner-held `AgentHandle`: the registry retains the teardown capability of every handle returned by `create`/`resume` (keyed by session id, dropped on detach) so an explicit identity lifecycle can stop an agent without knowing which consumer created it. Stopping fires `session/disposed`, which drains the persistence write-behind and emits the existing `host/session-removed` frame; the domain writes emit `host/workspace-changed` and `host/archived-sessions-changed` through the existing `domain/changed` projection, so every connected tab converges with zero new frame types.

### Client and GUI

`ctx.workspaces` gains `unarchiveSession` and `deleteSession`; the workspace list state already carried `archivedSessionIds`, and the sessions manager already drops a session on `host/session-removed`, so no new client state exists. The browser sidebar's view menu gains a third mode, **Archived** (`groupBy: 'workspace' | 'flat' | 'archived'`, persisted with the same viewing store): archived sessions render grouped by their owning workspace (or Ungrouped), always expanded, with the order controls hidden (archive order is Host append order). Each archived row's menu carries **Restore** (the archive-set flip, dialog-free like Archive) and **Delete** (danger). Delete opens a browser-owned confirmation dialog — the same pattern as workspace deletion — that stays open until the sessions projection has committed the removal (`host/session-removed`), so a stale frame cannot leak into the next gesture.

## Alternatives considered

**Soft delete via a durable tombstone in the archive set.** Rejected: a tombstone that persists in the workspace domain would still leave the session log on disk, and the request was to erase the history and reclaim space. The archive set remains a display-only layer; deletion is the workspace registry's destructive identity operation and the persistence service's explicit removal.

**A new `session.delete` RPC owned by the sessions domain.** Rejected: deletion touches workspace accounting, archive membership, and the durable log together, which is the workspace domain's identity surface — the same reason archiving lives there. Adding a second gateway method would split one user gesture across two domains.

**Stopping the live agent through a new loop-owned service.** Rejected in favor of `AgentRegistry.stop`: the gateway already depends on `ctx.agents`, the registry receives every `create`/`resume` handle anyway, and a loop-owned service would add a new dependency edge to the gateway for no additional coverage (config-declared agents are re-created by the loop on restart and are not GUI-deletable rows).

**Refusing to delete live sessions.** Rejected: the session a user is looking at is precisely the one most likely to be deleted; the gateway stops the agent first and the registry/persistence guards remain as defense in depth.

## Consequences

**The GUI can now permanently destroy history.** Delete erases the durable log irreversibly; the confirmation dialog states this explicitly and stays open until the committed projection lands. There is no undo, and a deleted session cannot be restored or re-archived.

**SessionPersistence is no longer strictly append-only.** The service Definition and both first-party backends document `delete`/`deleteStored` as the explicit administrative exception; third-party backends must implement the new required hook to keep compiling against the `PersistenceBackend` interface.

**Deleting a live session converges every tab.** One RPC stops the agent (draining its write-behind), removes accounting and archive membership, erases the log, and reuses the existing `session-removed`/`workspace-changed`/`archived-sessions-changed` frames — no new wire frame, no new client state machine.

**Archived sessions are now a first-class surface.** The archived view shows what Archive hid, Restore reverses Archive, and Delete is the terminal step — completing the archive lifecycle the GUI previously truncated at hide-only.
