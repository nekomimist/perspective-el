# Perspective Architecture Notes

This document summarizes how `perspective.el` is structured internally and how
its main features interact. It is intended for contributors and maintainers.

## Scope

- Repository version analyzed: `2.21` (`CHANGELOG.md`, dated 2026-02-10)
- Main implementation file: `perspective.el`
- Main test file: `test/test-perspective.el`

## Repository Layout

- `perspective.el`: core implementation (mode, state model, commands,
  integrations, persistence).
- `test/test-perspective.el`: ERT regression and behavior tests.
- `README.md`: user-facing docs and configuration guidance.
- `CONTRIBUTING.org`: contributor workflow (tests and MELPA sandbox checks).
- `.github/workflows/test-perspective.yml`: CI test matrix (Emacs 24.4-30.2).
- `Makefile`: local test/byte-compile entry points.

## Core Data Model

Perspective data is represented by the `perspective` struct:

- `name`
- `buffers`
- `killed`
- `local-variables`
- `last-switch-time`
- `created-time`
- `window-configuration`
- `point-marker`

See `cl-defstruct` in `perspective.el`.

## Runtime State Model

Perspective state is frame-local and stored in frame parameters:

- `persp--hash`: hash table of perspective name -> struct
- `persp--curr`: current perspective
- `persp--last`: previously active perspective
- `persp--recursive`: recursive-edit support
- `persp--modestring`: rendered mode/header line text
- `persp-merge-list`: merge metadata for perspective merging

`persp-mode` initializes and tears down this state.

## Main Control Flow

### Mode Activation

`persp-mode` enables advice/hooks/integrations, initializes each frame via
`persp-init-frame`, then creates/activates the initial perspective.

### Perspective Switch

The switch pipeline is:

1. `persp-switch`
2. `persp-save` (persist outgoing perspective state in memory)
3. `persp-activate` (set current perspective, restore locals/windows/point)
4. mode line refresh and switch hooks

### Buffer Association

Buffer membership is managed by:

- `persp-add-buffer`
- `persp-remove-buffer`
- `persp-forget-buffer`
- `persp-set-buffer`

There is explicit handling to avoid killing the last buffer in a perspective
when that feature is enabled (`persp-avoid-killing-last-buffer-in-perspective`).

## Persistence and Session Durability

Disk persistence uses:

- `persp-state-save`
- `persp-state-load` (alias: `persp-state-restore`)

Saved state includes:

- file list (interesting buffers only)
- per-frame perspective tables
- perspective order
- merge metadata
- per-perspective window state

When lazy restore is enabled, saving reuses any still-deferred per-perspective
state instead of activating those perspectives and forcing their buffers/windows
to load. Loading follows a two-step flow in that mode: state file import first
registers empty perspective stubs plus pending payload, and the first explicit
activation of each perspective materializes its buffers and window state on
demand. Killing a still-deferred perspective similarly operates on its pending
state directly instead of activating it first.

Transient/unpersistable window buffers are rewritten to the perspective scratch
buffer during save-time window-state massage. The corresponding scratch buffer
is created lazily as part of perspective materialization, not at state-load
registration time.

## Perspective Merging

Merging lets one perspective temporarily include buffers from another:

- `persp-merge`
- `persp-unmerge`

Properties:

- one-directional
- non-transitive
- merge membership tracked in `persp-merge-list`

## Integrations

The package includes integrations for:

- Ido (`ido-make-buffer-list-hook`)
- Helm (advice + source actions)
- Ivy / Counsel (buffer switch wrappers)
- Consult (`persp-consult-source`)
- IBuffer (filter/group support)
- xref (per-perspective marker history/ring)
- winner-mode (perspective-local winner state)

## Test Strategy

`test/test-perspective.el` contains ERT tests for:

- perspective lifecycle (create/switch/kill/rename)
- buffer ownership semantics
- scratch-buffer behavior
- sorting semantics (`name`, `access`, `created`, `oldest`)
- save/load roundtrip
- merge/unmerge behavior
- targeted regressions for previously reported issues

Tests use helper macros to isolate mode/frame/buffer side effects between cases.

## Operational Commands

- `make` (or `make test`): run ERT tests
- `make compile`: byte-compile the package

## Contributor Notes

- The implementation is intentionally centralized in one file; when changing
  behavior, prefer adding focused tests in `test/test-perspective.el`.
- Several sections include compatibility/workaround code paths for older Emacs
  versions; verify behavior against CI matrix expectations.
- Advice-based integrations are still present in core flow; changes in Emacs
  internals or third-party buffer frameworks should be regression-tested.
