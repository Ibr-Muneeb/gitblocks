# `bl_git`

The version-control engine at the heart of **GitBlocks**. This package
knows nothing about Blender's UI (that's `ui/`) or how individual
datablock types are captured (that's `bl_types/`) — its job is everything
between them: tracking which datablocks exist, detecting when they change,
persisting them as content-addressed JSON, and driving Git to get
branching, committing, merging, and rebasing semantics on top of a
Blender scene.

## Why not just diff the `.blend` file?

`.blend` files are an opaque binary format — a single-byte change
anywhere produces a completely different file, so Git can't diff or merge
them meaningfully. GitBlocks sidesteps this by never versioning the
`.blend` file's *content* at all. Instead:

1. Every tracked datablock (`Object`, `Mesh`, `Material`, ...) gets a
   stable UUID (`tracking.py`).
2. On a timer, each UUID-tagged datablock is serialized to deterministic
   JSON and written to its own file, `.gitblocks/blocks/<uuid>.json`
   (`state.py` for capture, `json_io.py` for deterministic serialization,
   `blocks.py` for the file I/O).
3. A manifest (`.gitblocks/manifest.json`) indexes every block: its type,
   dependency UUIDs, content hash, and merge group (`manifest.py`,
   `constants.py`).
4. Git tracks the block files and the manifest — plain JSON, so it diffs,
   merges, and displays history exactly the way Git already knows how to.
5. Checking out a commit/branch re-hydrates the scene by reading the
   manifest at that ref and reconstructing each datablock in dependency
   order (`checkout.py`).

The one `.blend` file that does exist (the "bootstrap" file,
`bootstrap.py`) is just an empty shell used to open the project; all real
content lives in tracked blocks and is restored on load.

## Module map

| Module | Responsibility |
|---|---|
| `__init__.py` | Assembles `BpyGit`, the facade object from all mixins below, and drives initial load. |
| `tracking.py` | Assigns/maintains `gitblocks_uuid` on every tracked datablock. |
| `state.py` | Captures live scene state into hashable/serializable snapshots, resolves merge groups, and builds the UI-facing state tree consumed by `ui/`. |
| `blocks.py` | Low-level read/write/delete for individual block JSON files. |
| `json_io.py` | Deterministic JSON encode/decode (stable float rounding, sorted keys, binary payload support). |
| `manifest.py` | Manifest schema, migration, and integrity validation. |
| `paths.py` | Canonical on-disk layout (`.gitblocks/...`) as a single source of truth. |
| `constants.py` | Manifest schema keys/version. |
| `bootstrap.py` | Creates/locates the shell `.blend` file used to open a project. |
| `diffs.py` | Computes the flat working-tree/staged diff list from Git. |
| `ops.py` | `init`, `stage`, `unstage`, `discard`, `commit`, and the background change-detection poll. |
| `checkout.py` | Branch/commit switching and rebuilding the live scene from a manifest; the "carryover" stash mechanism that protects uncommitted changes across branch/merge/rebase operations. |
| `merge.py` | Per-block, tiered three-way merge for `merge`/`rebase`, plus conflict resolution. |

## The merge model

Ordinary Git merges work on text lines; GitBlocks instead merges per
*datablock*, three-way, keyed by UUID. Each block type falls into a tier
that controls how much auto-merging is trusted:

- **Tier A** — objects, meshes, collections: safe to recursively merge
  key-by-key.
- **Tier B** — materials, images, lights, cameras, scenes: any real
  overlap is always a conflict (auto-blending two shader graphs rarely
  makes sense).
- **Tier C** — everything else: conservative default, any overlap
  conflicts.

Conflicts are recorded on the manifest and resolved one at a time via
`resolve_conflict`, choosing "ours" or "theirs" per block.

## Requirements

- [`GitPython`](https://gitpython.readthedocs.io/) — Git plumbing.
- [`DeepDiff`](https://zepworks.com/deepdiff/) — content hashing
  (`DeepHash`) for change detection.
- Blender's bundled Python (`bpy`) — everything here runs inside Blender.

## Status

This module is being open-sourced folder by folder as part of a larger
cleanup pass. See the project root `README.md` (added once every module
is in) for the full picture, including `bl_types/`, `ui/`, and `utils/`.
