# The ionium fork

This repository is a fork of Archipelago that the ionium lobby
([Archipelago-lobby](https://github.com/ionium-ap/Archipelago-lobby)) builds its generation workers from. It carries a
small set of patches on top of each upstream release. `main` mirrors upstream and carries nothing.

## Lines and refs

There is one line per upstream release the lobby supports. The lobby keeps rooms on the release they were created
with, so an old line stays alive for as long as rooms on it exist.

| Ref | Kind | Purpose |
|---|---|---|
| `ionium-<release>` | branch | The line for that upstream release, for example `ionium-0.6.8`. Backports land here. |
| `<release>-ionium.<n>` | tag | An immutable pin, for example `0.6.8-ionium.1`. A later patch on the same line gets the next `n`. |
| `ionium` | branch | The original 0.6.7 line. Frozen, deprecated. |

The lobby pins a commit by SHA and fetches it with `git fetch origin <sha> --depth 1`, so a pinned commit must stay
reachable from a branch or tag on GitHub. Never move or delete a tag, and never rebase a line that has one.

## What the lobby requires

These come from how the worker image is built (`taskcluster/docker/ap-worker/setup.sh` and `prepare_worlds.sh` in the
lobby repository) and how its harness uses Archipelago (`ap-worker/*.py`).

* `Utils.__version__` is exactly the upstream release, written as `__version__ = "0.6.8"` on one line. The image build
  extracts it with grep and sed, names every core world `<world>-<version>.apworld` after it, and the lobby routes jobs
  by it. Do not add a suffix.
* Every core world loads from a zip. The image build zips each `worlds/<name>` directory into an `.apworld` and deletes
  the directory. Only `generic` and packages starting with `_` stay unpacked.
* Archipelago runs from a read-only tree.
* `Options.as_dict` accepts a call that names every option. Several apworlds in the lobby's index do this.

## What the 0.6.8 line carries

Each commit was cherry-picked with `-x`, so its message names the 0.6.7 commit it came from.

| 0.6.7 commit | What it does |
|---|---|
| `ab9bfe91` | LADX: an item name containing a double quote no longer breaks the ROM patch |
| `39bf81bf` | Pokemon Emerald: unofficial `skip_e4` option |
| `ef694b02` | `Utils.user_path` accepts a read-only root |
| `bc7b32d5` | Fill progress logging from upstream PR 3575 |
| `2a763366` | Faster fill swaps, upstream PR 4865 |
| `379cf58b` | Faster playthrough sphere culling, upstream PR 3890 |
| `91617d40` | Item classification flags are cached on the item |
| `ab4aad5e` | `Options.as_dict` no longer asserts that fewer than all options were named |
| `c38bfc54` | Raft loads from a zip |
| `ca8c2dd6` | Lufia II Ancient Cave loads from a zip |
| `5d56cf15` | Super Metroid loads its presets from a zip |
| `9c33ad62` | The accessibility check passes when a game is beatable from the start |
| `18b46b2c` | YAML errors are raised as a `PlayerFilesError` group that keeps each cause |
| `f04b3a34` | The "Found N World Types" log line skips hidden worlds |

## What the 0.6.8 line leaves out, and why

* **Hosting and WebHost patches** (`MultiServer.py`, `NetUtils.py`, `WebHostLib/`): 34 commits on the 0.6.7 line,
  including the broadcast batching, the orjson encoder, DataStorage limits and the tracker cache. From 0.6.8 on, this
  fork is used for generation only.
* **Ocarina of Time patches** (`ea25fabe`, `91774bd8`): the world is unmaintained upstream (see `docs/CODEOWNERS`) and
  is not enabled in the lobby's index.
* **The YAML version checks stay in** (`a260a654` and `7a2a5ca6` are not carried). A YAML whose `requires: version`
  is newer than the generator, or whose `requires: game` asks for a newer world than the one loaded, is rejected. That
  tells an uploader early that they used a template from the wrong version. The 0.6.7 line accepts both.
* **Batched restrictive fill** (`3f84e384`): merged upstream in 0.6.8 as PR 3872.

## Known limitation

Ocarina of Time loads from a zip but cannot generate from one, because it opens its data files by path. This is
upstream behavior and is the same on the 0.6.7 line.

## Cutting a line for a new upstream release

1. Update `main` to the upstream release tag and branch `ionium-<release>` from it.
2. Cherry-pick the previous line's commits one at a time with `git cherry-pick -x`. Before each one, check whether
   upstream has merged an equivalent (the fill and playthrough speedups and the fill logger are open upstream PRs) and
   whether the code it patches still exists in that form. A pick that applies cleanly can still be wrong.
3. Verify in an exported copy of the branch, with the same installs as the lobby's `setup.sh` plus `setuptools<81`
   (the lobby worker's own dependencies provide it, and Pokemon Emerald imports `pkg_resources`):
   * `python -c "import Utils; print(Utils.__version__)"` prints the release.
   * The core tests pass: `test/general`, `test/programs`, `test/netutils`, `test/multiserver`, `test/multiworld`,
     `test/options`, `test/utils`, and the tests of each patched world.
   * With every world zipped as `prepare_worlds.sh` does and its directory removed, importing `worlds` leaves
     `worlds.failed_world_loads` empty.
   * A multiworld generates from that zipped tree. Skip the `assert_generate` stage and unset `DISPLAY`: several
     worlds ask for their base ROM in that stage and open a file dialog when it is missing.
4. Tag the tip `<release>-ionium.1`, push the branch and the tag, and give the lobby the SHA.
