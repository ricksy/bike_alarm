# Firmware Submodule — Setup and Daily Workflow

How `meshtastic/firmware` is vendored into this project, and how to work with it
day to day.

---

## Why a submodule

This repository is the project home: design docs, hardware notes, measurements. The
firmware is a fork of a large, fast-moving upstream. Keeping them as one repo would
either bury the project's own artefacts inside a firmware fork, or leave a coding
agent with documentation and no source to act on.

A submodule gives one checkout containing both, with the firmware pinned to an exact
commit that travels with the parent repo's history.

## Facts about the upstream that shape everything below

| | |
|---|---|
| Repository | `meshtastic/firmware` |
| **Default branch** | **`develop`** — *not* `main` |
| Size | ~75 MB |
| Licence | GPL-3.0 |
| Own submodules | `protobufs`, `meshtestic` — so this becomes a **nested** submodule setup |

The default branch catches people out constantly. The nested submodules mean every
initialisation has to be `--recursive`.

## Resulting layout

```
bike_alarm/
  CLAUDE.md
  README.md
  .gitmodules
  docs/
    bike-alarm-project.md
    measurements.md
    firmware-submodule.md      ← this file
  hardware/                     (schematics, enclosure — later)
  firmware/                     ← submodule → ricksy/meshtastic-firmware
      protobufs/                ← nested
      meshtestic/               ← nested
```

---

## One-time setup

### 1. Fork the firmware

A fork is required because the project modifies driver-level code
(`src/motion/LIS3DHSensor.cpp` and related).

```bash
gh repo fork meshtastic/firmware --fork-name meshtastic-firmware --clone=false
```

The explicit `--fork-name` avoids a bare `ricksy/firmware`, which is ambiguous later.

### 2. Add the submodule

```bash
cd bike_alarm
git submodule add -b develop https://github.com/ricksy/meshtastic-firmware.git firmware
```

Use the **HTTPS** URL. It is written into `.gitmodules`, which is committed — HTTPS
keeps the repository cloneable by anyone, and the `gh` credential helper handles
pushes without needing keys present.

### 3. Initialise the nested submodules

`git submodule add` does **not** do this automatically:

```bash
git submodule update --init --recursive
```

### 4. Add the upstream remote inside the submodule

Needed later to pull Meshtastic's changes.

```bash
git -C firmware remote add upstream https://github.com/meshtastic/firmware.git
git -C firmware fetch upstream
```

### 5. Commit the pointer

```bash
git add .gitmodules firmware
git commit -m "Add meshtastic firmware fork as submodule"
git push
```

What the parent repository stores is a **single commit SHA** — not the code, not a
branch name. This is the part that surprises people.

### 6. Quality-of-life config

```bash
git config submodule.recurse true
git config push.recurseSubmodules check
```

The first makes `git pull` / `git checkout` recurse automatically instead of silently
leaving the submodule stale. The second refuses a parent push if the submodule commits
it references have not been pushed yet — which prevents the most common way to break
this setup.

---

## Daily workflow

This is the real cost of the approach: **two repositories, and commits do not
cascade.**

```bash
# 1. Branch inside the submodule BEFORE editing
cd firmware
git switch -c lis3dh-interrupt-wake

# 2. Work
$EDITOR src/motion/LIS3DHSensor.cpp
pio run -e rak4631
./bin/run-tests.sh
trunk fmt

# 3. Commit and push in the submodule
git commit -am "LIS3DH: interrupt-driven wake via INT1"
git push -u origin lis3dh-interrupt-wake

# 4. Record the new pointer in the parent
cd ..
git commit -am "firmware: LIS3DH interrupt wake"
git push
```

Step 4 is not optional. Without it the parent still points at the old commit and the
change effectively does not exist as far as this repository is concerned.

### Useful status commands

```bash
git submodule status                  # what commit is the submodule pinned to
git diff --submodule                  # show submodule pointer movement in a diff
git -C firmware status                # status inside the firmware repo
git -C firmware log --oneline -5      # recent firmware commits
```

---

## Syncing with upstream

```bash
cd firmware
git fetch upstream
git rebase upstream/develop        # on your working branch
cd ..
git commit -am "firmware: sync with upstream develop"
git push
```

Meshtastic moves quickly, and `src/motion/` is precisely where this project makes
changes, so conflicts there are expected over time.

**Mitigation:** keep the diff small and surgical, and structure the interrupt work as
an opt-in low-power mode rather than a rewrite. A change that is plausibly upstreamable
is also a change that is cheap to rebase.

---

## Cloning fresh (new machine, or after a reinstall)

```bash
git clone --recurse-submodules https://github.com/ricksy/bike_alarm.git
```

Without the flag you get an empty `firmware/` directory. If that happens:

```bash
git submodule update --init --recursive
```

---

## Gotchas

**Default branch is `develop`.** Branching off `main` in a Meshtastic fork gets you
something stale or nonexistent.

**Submodules check out in detached HEAD.** If you edit and commit without
`git switch -c` first, the commit is real but unattached — the next
`git submodule update` will orphan it. Recovery is `git reflog` inside the submodule,
but it is far easier to always branch first.

**Pushing the parent before the submodule.** The parent then references a SHA nobody
can fetch and fresh clones break. `push.recurseSubmodules check` (set in step 6)
guards against this.

**`--recursive` everywhere.** Because of `protobufs` and `meshtestic`, a plain
`git submodule update --init` leaves the build broken in a way whose error message
does not obviously point at submodules.

**Shallow clones.** Tempting given the 75 MB, but they make rebasing onto upstream
painful. Take the full clone.

**Licence.** The firmware fork is GPL-3.0. That constrains anything derived from it.
This parent repository's own documentation and hardware files are not automatically
covered, but do not blur the line by vendoring firmware code into it.

---

## Telling the coding agent

The following belongs in the root `CLAUDE.md`, because an agent will otherwise treat
`firmware/` as ordinary subdirectories of one repository:

```markdown
## Repository structure

`firmware/` is a git submodule pointing at a fork of meshtastic/firmware
(branch: develop, licence: GPL-3.0). It is a **separate git repository**.

- Always create a branch inside `firmware/` before editing — submodules
  check out detached HEAD and commits will be orphaned otherwise.
- Commits inside `firmware/` do not appear in the parent repo. After
  committing there, commit the updated pointer in the parent too.
- Push the submodule before pushing the parent.
- Read `firmware/.github/copilot-instructions.md` before any non-trivial
  firmware change.
```

---

## Backing out

This is a reversible decision, not a foundational one. If the ceremony costs more than
it is worth:

```bash
git submodule deinit -f firmware
git rm -f firmware
rm -rf .git/modules/firmware
git commit -m "Remove firmware submodule"
```

Then keep a sibling checkout and start sessions with:

```bash
claude --add-dir ../meshtastic-firmware
```

What is lost: the pinned-version guarantee, and you must remember the flag each
session. Nothing else.

---

## Command quick reference

| Task | Command |
|---|---|
| Clone this project | `git clone --recurse-submodules <url>` |
| Repair an empty `firmware/` | `git submodule update --init --recursive` |
| See pinned commit | `git submodule status` |
| Start firmware work | `git -C firmware switch -c <branch>` |
| Build | `cd firmware && pio run -e rak4631` |
| Test | `cd firmware && ./bin/run-tests.sh` |
| Format | `cd firmware && trunk fmt` |
| Sync with upstream | `git -C firmware fetch upstream && git -C firmware rebase upstream/develop` |
| Record pointer after firmware commit | `git commit -am "firmware: <what>"` |
