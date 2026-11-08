# Gerrit → GitLab migration plan

Terms follow [GLOSSARY.md](./GLOSSARY.md). Decisions and their reasons are in [docs/adr/](./docs/adr/).

## 0. Summary

| Area | Before (Gerrit) | After (GitLab) |
|---|---|---|
| alice tree | 6 nested repos (alice, user, release, item, att, mpe) with `.gitignore` tricks | **1 repo `alice`**, same directory layout |
| core, driver | separate repos | separate Restricted repos, same paths |
| tool, toolchains | Gerrit | GitLab, binaries in LFS |
| kernels | Gerrit | **stay on Gerrit (temporary)**, synced by `repo` via a second remote |
| Access | per-repo Gerrit ACLs | two GitLab subgroups: `shared/`, `internal/` |
| Workspace | `repo` + manifest per Audience | `repo` + manifest per Audience (unchanged for people) |
| Review | `git push origin HEAD:refs/for/master` | branch → `git push` → merge request |
| Branch | `master` | `main` |
| Prebuilts | built by hand, pushed to `release` repo | built by hand, MR to `alice` touching `release/` (CI later) |

The on-disk Workspace layout is **identical** to today, so `make AP=<ap>` (or however the AP parameter is passed) works unchanged for both Audiences.

---

## 1. Pre-flight: facts to collect

Check these at `https://<gitlab>/help/instance_configuration` or ask the GitLab admins. Rehearsal can't start until they're known.

- [ ] **Git LFS enabled** on the instance, and the LFS storage quota
- [ ] **Max push size** (the import pushes the merged `alice` repo in one go)
- [ ] **Repository size limit**
- [ ] **Tier** (Free / Premium). This decides whether approvals can be *enforced* later; nothing in this plan needs it.
- [ ] How existing CI (the system that sets `Verified` today) can report status to GitLab MRs
- [ ] Size of each Gerrit repo except the kernels after LFS conversion. Run `git lfs migrate info --everything --top=30` on a clone to see which file patterns take the space.

---

## 2. GitLab layout and access

Step-by-step UI walkthrough: [GITLAB-SETUP.md](./GITLAB-SETUP.md).

```
<project>/                          top group
├── shared/                         core-developers: Developer   item-developers: Developer
│   ├── alice                       merged repo
│   ├── tool
│   ├── manifest-item               Item developers' Manifest
│   └── bsp/
│       ├── sm8250-toolchain
│       ├── sm8550-toolchain
│       └── sm8650-toolchain
└── internal/                       core-developers: Developer   item-developers: (none)
    ├── core
    ├── driver
    └── manifest-core               Core developers' Manifest
```

- Create two **GitLab Groups** for people: `core-developers` and `item-developers`. Invite them to the subgroups ("Members → Invite a group"). Onboarding = adding a person to one group.
- Give a few Core developers **Maintainer** on the top group to manage settings.
- Putting a project in `shared/` or `internal/` *is* the access decision. Never grant per-project access to Item developers.

### Per-project settings (every project)

| Setting | Value |
|---|---|
| Default branch | `main` |
| Protected branch `main` | Allowed to push and merge: **No one**. Allowed to merge: **Developers + Maintainers**. Force push: off |
| Merge method | **Fast-forward merge** |
| Squash commits when merging | **Require** (one commit per change, as in Gerrit) |
| All threads must be resolved | on |
| Pipelines must succeed | on **once CI reports status to GitLab**. Turn it on too early and merges are blocked when no pipeline runs |
| Delete source branch by default | on |
| Approvals | 1 expected by team convention (enforce later if Premium) |

Set these once at the group level where the GitLab version allows it, or with a short API script, so new projects can't drift.

---

## 3. Manifests

The GitLab remote is deliberately **named `origin`**, so standard instructions (`git push -u origin <branch>`) work inside a `repo` Workspace. By default `repo` names the remote after the manifest's `<remote name>`.

AP projects carry the AP's group. Everything else carries group `base`. That gives:

- `repo init ...` → all APs (same as today)
- `repo init ... -g base,sm8650` → one AP only

### `shared/manifest-item` → `default.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="origin" fetch="ssh://git@<gitlab-host>/<project>/" />
  <default remote="origin" revision="main" sync-j="8" />

  <project name="shared/alice" path="app/alice" groups="base" />
  <project name="shared/tool"  path="tool"      groups="base" />

  <project name="shared/bsp/sm8250-toolchain" path="bsp/sm8250/toolchain" groups="sm8250" />
  <project name="shared/bsp/sm8550-toolchain" path="bsp/sm8550/toolchain" groups="sm8550" />
  <project name="shared/bsp/sm8650-toolchain" path="bsp/sm8650/toolchain" groups="sm8650" />
</manifest>
```

### `internal/manifest-core` → `default.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="origin" fetch="ssh://git@<gitlab-host>/<project>/" />
  <remote name="gerrit" fetch="ssh://<gerrit-host>:29418/" review="<gerrit-host>" />
  <default remote="origin" revision="main" sync-j="8" />

  <!-- Shared: keep identical to manifest-item -->
  <project name="shared/alice" path="app/alice" groups="base" />
  <project name="shared/tool"  path="tool"      groups="base" />
  <project name="shared/bsp/sm8250-toolchain" path="bsp/sm8250/toolchain" groups="sm8250" />
  <project name="shared/bsp/sm8550-toolchain" path="bsp/sm8550/toolchain" groups="sm8550" />
  <project name="shared/bsp/sm8650-toolchain" path="bsp/sm8650/toolchain" groups="sm8650" />

  <!-- Restricted, on GitLab -->
  <project name="internal/core"   path="app/alice/core"   groups="base" />
  <project name="internal/driver" path="app/alice/driver" groups="base" />

  <!-- Restricted, still on Gerrit (ADR 0003). Use the existing Gerrit project names. -->
  <project name="<gerrit-path>/sm8250/kernel" path="bsp/sm8250/kernel" remote="gerrit"
           revision="master" groups="sm8250" clone-depth="1" />
  <project name="<gerrit-path>/sm8550/kernel" path="bsp/sm8550/kernel" remote="gerrit"
           revision="master" groups="sm8550" clone-depth="1" />
  <project name="<gerrit-path>/sm8650/kernel" path="bsp/sm8650/kernel" remote="gerrit"
           revision="master" groups="sm8650" clone-depth="1" />
</manifest>
```

Nested checkout paths (`app/alice/core` inside `app/alice`) are fine: `repo` checks out parents first. `core/` and `driver/` are still separate repos sitting inside alice's directory, so `alice/.gitignore` **must keep ignoring `core/` and `driver/`**, as it does today. Otherwise `git add -A` in alice would pull Restricted code into a Shared repo.

The Shared entries are duplicated in both manifests (Item developers must not see `manifest-core`). Rule: any change to Shared entries is made in both repos in the same sitting. Later, a small CI check that diffs them can enforce this.

`clone-depth="1"` on the kernels makes the first sync fast. Anyone who needs kernel history runs `git fetch --unshallow` in that directory.

### Init commands people will use

```sh
# Core developers
repo init -u ssh://git@<gitlab-host>/<project>/internal/manifest-core.git -b main --git-lfs [-g base,sm8650]
# Item developers
repo init -u ssh://git@<gitlab-host>/<project>/shared/manifest-item.git  -b main --git-lfs [-g base,sm8650]
repo sync -c -j8
```

---

## 4. Import procedure

Run on a machine with fast access to both servers, plenty of disk, and `git`, `git-lfs`, and `git-filter-repo` installed. Run it once from a Gerrit snapshot for the rehearsal, then again for real on freeze day. All history rewrites happen here, before anything is pushed.

Keep every `.git/filter-repo/commit-map` file produced. They map old Gerrit SHAs to new ones and are the only way to translate SHAs quoted in old tickets.

### 4.1 Merged `alice`

```sh
#!/usr/bin/env bash
set -euo pipefail
G=ssh://<gerrit-host>:29418/<gerrit-path>      # adjust
W=$PWD/import && mkdir -p "$W" && cd "$W"

# name-on-gerrit  target-subdirectory  tag-prefix
SUBS=(
  "user     user        user/"
  "release  release     release/"
  "item     item        item/"
  "att      item/att    att/"
  "mpe      item/mpe    mpe/"
)

git clone "$G/alice" alice            # alice keeps its root and its own tags
for s in "${SUBS[@]}"; do
  read -r name dir prefix <<<"$s"
  git clone "$G/$name" "$name"
  ( cd "$name" && git filter-repo --to-subdirectory-filter "$dir" --tag-rename ":$prefix" )
done

cd alice
for s in "${SUBS[@]}"; do
  read -r name _ _ <<<"$s"
  git remote add "$name" "../$name"
  git fetch "$name" --tags
  git merge --allow-unrelated-histories --no-edit "$name/master" \
      -m "Import $name history into alice"
  git remote remove "$name"
done
```

Then by hand, in one commit:

1. Edit `alice/.gitignore`: **remove** `user/`, `release/` and `item/`, and **keep** `core/` and `driver/`.
2. Edit `item/.gitignore`: **remove** `att/` and `mpe/`.
3. Check that nothing else tracked was hidden: `git status --ignored`, and do a build.

```sh
git commit -am "Drop .gitignore entries for absorbed repos"
```

Convert binaries to LFS across all history. Choose patterns from the `migrate info` output in §1:

```sh
git lfs migrate info   --everything --top=30
git lfs migrate import --everything --include="release/**,<other binary patterns>"
git branch -m master main
```

The `att` and `mpe` merges are independent of `item`'s merge, so their order doesn't matter. Their files end up under `item/att/` and `item/mpe/` because of the subdirectory filter.

### 4.2 Single repos (core, driver, tool, each toolchain)

```sh
git clone ssh://<gerrit-host>:29418/<gerrit-path>/<repo> && cd <repo>
git lfs migrate import --everything --include="<binary patterns>"   # toolchains; others only if needed
git branch -m master main
```

### 4.3 Push

```sh
git remote add gitlab ssh://git@<gitlab-host>/<project>/<subgroup>/<repo>.git
git lfs push --all gitlab           # upload LFS objects first
git push gitlab main
git push gitlab --tags
```

Then set the project settings from §2, if group-level defaults didn't already apply them.

### 4.4 Manifests

Create `manifest-item` and `manifest-core` from §3 and push to `main`.

---

## 5. Review workflow

There's no wrapper and no magic ref. This is the standard GitLab/GitHub flow.

| Gerrit habit | GitLab equivalent |
|---|---|
| `repo start <topic> .` | same (or `git switch -c <topic>`) |
| commit | same |
| `git push origin HEAD:refs/for/master` | `git push -u origin <topic>` → open the MR link git prints |
| New patchset (`--amend` + push) | `--amend` + `git push --force-with-lease`, **or** add commits (squashed at merge) |
| Rebase on latest | `git fetch origin && git rebase origin/main` + force-push, or the MR's **Rebase** button |
| Code-Review +2 | **Approve** button (1 expected) |
| Verified +1 | Pipeline status |
| Submit | **Merge** button |
| Abandon | **Close** MR |
| Download someone's change | `git fetch origin refs/merge-requests/<id>/head && git checkout FETCH_HEAD` |
| Topic across repos | One MR per repo; reference the others in the description (`internal/core!12`). Much rarer now that alice is one repo |
| `repo sync` / `repo prune` | same |
| `repo upload` | **kernel only** (still Gerrit) |

Because merges are fast-forward, an MR must be up to date with `main` before merging. GitLab shows a Rebase button when it isn't.

### Prebuilts (manual until CI)

1. MR in `internal/core` or `internal/driver` → merged.
2. Core developer builds Prebuilts for each affected AP from the new `main`.
3. MR in `shared/alice` changing `release/`. The binaries go to LFS automatically through `.gitattributes`.

**Follow-up #1:** CI on `core`/`driver` `main` builds Prebuilts per AP and opens step 3's MR automatically.

### Order for a change spanning core and alice

Example: a new function in core, called from new code in `user/` and `item/`.

Rules that apply in every case:

- **Core merges first.** Prebuilts are built only from the merged core `main`, never from an open core MR. It can still change, and the binaries wouldn't match any commit on core `main`. Name the source in the commit message: `release: core@<sha>`.
- **alice CI builds the way Item developers do**, using only `release/` Prebuilts and no `core/` source. It tests exactly what Item developers will get.
- Open the `user/`/`item/` MR as a **Draft** at the same time as the core MR, so its review runs in parallel. Its pipeline fails until the new Prebuilts are on `main` or in the same MR.

**Additive core change** (new function only; existing functions unchanged):

1. Merge the core MR.
2. Build Prebuilts for each AP and open a **release-only** alice MR (`release/` and the new headers). The new symbols are unused, so `main` stays green. Merge it quickly; nothing to review but the provenance.
3. Rebase the `user/`/`item/` MR onto `main`. The pipeline turns green. Merge after review.

**Breaking core change** (existing function changed or removed):

1. Merge the core MR.
2. Build Prebuilts and add them as a commit to the **same** alice MR that updates the `user/`/`item/` callers. Merge it as one commit. Merging either half alone would break `main` for Item developers.
3. Keep the time between steps 1 and 2 short. Until step 2 lands, Core developers' source builds of alice's `main` are broken. Where practical, split it into additive steps instead: add the new function, move callers, then remove the old one.

---

## 6. Cutover timeline

| When | What |
|---|---|
| **T−4 weeks** | Pre-flight facts (§1). Create groups, subgroups and settings (§2). |
| **T−3 weeks** | **Rehearsal import** from a Gerrit snapshot (§4). Volunteers from each Audience run `repo init/sync`, with and without `-g base,<ap>`, and `make` for every AP. Write the cheat sheet (§5 table) to the GitLab wiki or `tool/`. |
| **T−3 weeks** | **Announce:** cutover date, and that *open Gerrit changes must be submitted or abandoned by T−1 day; anything left is gone*. |
| **T−2 weeks** | Rehearsal sign-off checklist (below). Fix the import scripts, then rerun the rehearsal if anything changed. |
| **T−1 day** | Deadline for open Gerrit changes. |
| **T (freeze day)** | Gerrit read-only for every repo except the kernels. Final import (§4). Spot-check builds. Publish the `repo init` commands. |
| **T+0** | Everyone makes a **fresh Workspace** in a new directory and deletes the old one after their first successful build. |
| **T+2 weeks** | Remove remaining write access on archived Gerrit projects. Gerrit = read-only archive + live kernels. |

### Rehearsal sign-off checklist

- [ ] Core developer Workspace: `repo sync` succeeds with GitLab + Gerrit remotes, and `make` succeeds for each AP
- [ ] Item developer Workspace: same, with only Shared repos present, and `make` succeeds using Prebuilts from `release/`
- [ ] `-g base,<ap>` Workspace builds that AP
- [ ] An Item developer account **cannot** open, clone, or see in search anything under `internal/`, including `manifest-core`
- [ ] `git log -- item/att` in merged alice shows the old att history. Tags `att/<x>` exist
- [ ] LFS files check out as real binaries, not pointer files (`git lfs ls-files`, and the build)
- [ ] A test MR goes through branch → push → MR → approve → rebase → merge, and lands as one fast-forward commit on `main`

---

## 7. Follow-ups after cutover

1. **CI-built Prebuilts** (§5).
2. **Connect CI to GitLab**, then turn on "Pipelines must succeed".
3. **Move kernels to GitLab** (ADR 0003): fresh-history import with binaries in LFS, then switch their manifest entries to `origin`, and Gerrit can be retired.
4. Optional: CI check that the Shared entries in both manifests are identical.
5. If the instance is Premium: enforce 1 approval and CODEOWNERS for `core/`, `driver/` and the `alice` subdirectories.
