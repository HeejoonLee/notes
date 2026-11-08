# Setting up the demo GitLab (UI walkthrough)

How to build the layout from [PLAN.md](./PLAN.md) §2–§3 on a self-hosted GitLab where you are the administrator. It's written for GitLab 17/18. Menu labels may differ slightly in other versions.

## How the access model works (read first)

There are two kinds of groups. Keep them apart:

| Kind | Groups | Holds | Purpose |
|---|---|---|---|
| **People groups** | `core-developers`, `item-developers` | people only, no projects | "who is who" |
| **Project groups** | `proj` → `proj/shared`, `proj/internal` (+ `bsp` subgroups) | projects only, no individual people | "where code lives" |

You connect them by **inviting a people group into a project group** with a role:

```
proj/                       ◄── core-developers invited here as Developer
├── shared/                 ◄── item-developers invited here as Developer
│   ├── alice, tool, manifest-item
│   └── bsp/  sm8250-toolchain, sm8550-toolchain, sm8650-toolchain
└── internal/               (nobody else invited)
    ├── core, driver, manifest-core
    └── bsp/  sm8250-kernel, sm8550-kernel, sm8650-kernel   (demo stand-ins; real kernels stay on Gerrit)
```

- Access is **inherited downward**. Inviting `core-developers` at `proj` gives them `shared/`, `internal/`, and everything under both.
- `item-developers` is invited **only** at `proj/shared`, so they never see anything under `internal/`.
- Everything is **Private**. A private group or project is invisible (404) to anyone without access.
- Someone's effective role is the **lower** of their role inside the people group and the role the group was invited with. So add people to the people groups as **Developer**. If you add them as Guest, they'll only get Guest access everywhere.

Onboarding a new person = adding them to one people group. Nothing else.

---

## Where things are in the UI

- **Left sidebar.** It changes with where you are (your homepage, a group, or a project).
- **`+` button** (top of the left sidebar): create a new group, subgroup, or project. Inside a group, it creates things *in that group*.
- **Search or go to…** (top of the sidebar): jump to any group or project by name.
- **Admin area**: the **Admin** button at the bottom of the left sidebar. In older versions: *Search or go to… → Admin Area*.
- **Your avatar** (top of the sidebar) → **Edit profile**: your SSH keys and access tokens.

---

## Step 1 — Instance settings (Admin)

Go to **Admin → Settings → General**.

1. **Visibility and access controls** → *Expand*
   - *Default project visibility*: **Private**
   - *Default group visibility*: **Private**
   - *Restricted visibility levels*: tick **Public** so nothing can accidentally be made public.
   - Save changes.
2. **Sign-up restrictions** → *Expand* → untick **Sign-up enabled**. You create the accounts yourself. Save changes.
3. **Account and limit** → *Expand*. Note **Maximum push size (MiB)**, and *Size limit per repository* if your edition shows it. Raise the push size if a real import will be bigger. These are the facts PLAN.md §1 asks for.

Go to **Admin → Settings → Repository** → **Default branch** → *Expand* → *Initial default branch name*: `main`. Save.

**Git LFS** is on by default on self-managed GitLab (`gitlab_rails['lfs_enabled']` in `gitlab.rb`). LFS files transfer over HTTP(S) to the instance's `external_url`, even when you clone over SSH. Make sure that URL is reachable from the client machines.

---

## Step 2 — Demo user accounts (Admin)

Go to **Admin → Overview → Users → New user**.

| Name | Username | Will be in |
|---|---|---|
| Core One | `core1` | core-developers (also Maintainer on `proj`) |
| Core Two | `core2` | core-developers |
| Item One | `item1` | item-developers |
| Item Two | `item2` | item-developers |

For each user: fill in *Name*, *Username*, *Email* (any address works for a demo) → **Create user**. On the user's page, click **Edit**, set a **Password**, and save. They'll have to change it at first login.

Adding SSH keys for them: on the user's page in Admin, click **Impersonate** (top right). You are now browsing as that user. Go to **avatar → Edit profile → SSH Keys → Add new key**, paste their public key, and click **Add key**. Then click **Stop impersonating** (avatar menu or top bar).

For testing two identities from one machine, use **separate Linux users** (or containers), each with its own `~/.ssh` key. That's simpler than juggling SSH config inside `repo`.

---

## Step 3 — People groups

**`+` → New group → Create group**

- *Group name*: `core-developers`. The URL fills in automatically.
- *Visibility level*: **Private**
- The "who will use this group" questions are optional; skip them.
- **Create group**

Repeat for `item-developers`.

Add the people: open `core-developers` → left sidebar **Manage → Members** → **Invite members** (top right) → type `core1`, `core2` → *Select a role*: **Developer** → **Invite**. Do the same in `item-developers` with `item1` and `item2`.

You stay **Owner** of both groups, so only you can change who is in them.

---

## Step 4 — Project groups and subgroups

1. **`+` → New group → Create group** → name `proj` → **Private** → **Create group**.
2. Inside `proj`, click **New subgroup** (top right of the group page, or **`+` → New subgroup**) → name `shared` → **Private** → **Create subgroup**.
3. Back in `proj` → **New subgroup** → `internal` → **Private** → create.
4. Inside `proj/shared` → **New subgroup** → `bsp`. Inside `proj/internal` → **New subgroup** → `bsp`.

Lock down sharing on `proj` → **Settings → General → Permissions and group features** → *Expand*:

- Tick **Projects in proj cannot be shared with other groups**. Access can then only come from the group invites below, never from someone sharing a single project.
- Do **not** tick *Prevent members from sending invitations to groups outside of proj*. That setting would block the people-group invites in the next step.
- Save changes.

Default branch protection for new projects: `proj` → **Settings → Repository → Default branch** → *Expand* → *Initial default branch protection*: **Protected against pushes**. Save. Step 7 tightens it further for each project.

---

## Step 5 — Connect people to code (the important step)

1. Open `proj` → **Manage → Members** → **Invite a group** (top right).
   - *Select a group to invite*: `core-developers`
   - *Select a role*: **Developer**
   - **Invite**
2. Open `proj/shared` → **Manage → Members** → **Invite a group**.
   - Group: `item-developers`, role **Developer** → **Invite**
3. Make the lead a Maintainer: `proj` → **Manage → Members** → **Invite members** → `core1` → role **Maintainer** → **Invite**. When someone has two roles, the higher one wins.
4. **Do not invite anyone at `proj/internal`.** It gets Core developers by inheritance from `proj`, and nobody else.

Check: open `proj/internal` → **Manage → Members**. You should see `core1` and `core2` (inherited) and no `item1` or `item2`.

Note: Item developers can see the members list of `shared/`, and that list includes the inherited Core developers. They see names of people, not code.

---

## Step 6 — Create the projects

Create one project by hand to learn the screen:

Open `proj/shared` → **New project** → **Create blank project**

- *Project name*: `alice`
- *Project URL*: should already show `proj/shared/`
- *Visibility Level*: **Private**
- **Untick "Initialize repository with a README"**. You'll push existing content, and an initial commit would conflict.
- **Create project**

The other projects can be created the same way, or simply by **pushing to them**. GitLab creates a private project automatically when you push to a path in a group you own. The seed script below relies on that.

| Subgroup | Projects |
|---|---|
| `proj/shared` | `alice`, `tool`, `manifest-item` |
| `proj/shared/bsp` | `sm8250-toolchain`, `sm8550-toolchain`, `sm8650-toolchain` |
| `proj/internal` | `core`, `driver`, `manifest-core` |
| `proj/internal/bsp` | `sm8250-kernel`, `sm8550-kernel`, `sm8650-kernel` (demo only) |

### Seed demo content

Add your own (admin) SSH key first (**avatar → Edit profile → SSH Keys**). Then run the script below as yourself, **before** Step 7 locks `main`. It creates tiny placeholder repos in the real layout:
- the alice `.gitignore` ignores `core/` and `driver/`
- `release/` and the toolchain `bin/` hold fake binaries stored in LFS
- there are two manifests
- the `Makefile` reports whether it built core from source or linked the prebuilt binary

Set `GL` to your instance first. Docker installs often map SSH to port 2222.

```sh
#!/usr/bin/env bash
set -euo pipefail
GL="ssh://git@gitlab.example.com:2222/proj"     # <-- your host/port
W="$(mktemp -d)"
APS=(sm8250 sm8550 sm8650)
git lfs install --skip-repo >/dev/null

start()  { mkdir -p "$W/$1" && cd "$W/$1" && git init -q -b main; }
finish() { git add -A && git commit -qm "Initial demo content" && git push -q "$GL/$1.git" main && echo "pushed $1"; }
blob()   { mkdir -p "$(dirname "$1")" && head -c 1048576 /dev/urandom > "$1"; }

# --- shared/alice: merged repo; core/ and driver/ are separate repos, so ignored
start shared/alice
printf 'core/\ndriver/\n' > .gitignore
git lfs track 'release/**' >/dev/null
cat > Makefile <<'EOF'
.RECIPEPREFIX = >
AP ?= sm8650
all:
> @test -d ../../bsp/$(AP)/toolchain || { echo "missing bsp/$(AP)/toolchain"; exit 1; }
> @if [ -d core ]; then echo "core: building from source"; \
>  else test -f release/$(AP)/libcore.a && echo "core: linking release/$(AP)/libcore.a"; fi
> @echo "alice built for $(AP)"
EOF
mkdir -p user item/att item/mpe
echo user > user/README; echo item > item/README
echo att > item/att/README; echo mpe > item/mpe/README
for ap in "${APS[@]}"; do blob "release/$ap/libcore.a"; done
finish shared/alice

start shared/tool;      echo tools  > README; finish shared/tool
start internal/core;    echo core   > README; finish internal/core
start internal/driver;  echo driver > README; finish internal/driver

for ap in "${APS[@]}"; do
  start "shared/bsp/$ap-toolchain"
  git lfs track 'bin/**' >/dev/null; blob bin/cc
  finish "shared/bsp/$ap-toolchain"
  start "internal/bsp/$ap-kernel"; echo "kernel $ap (demo stand-in)" > README
  finish "internal/bsp/$ap-kernel"
done

# --- manifests
header() { printf '%s\n' '<?xml version="1.0" encoding="UTF-8"?>' '<manifest>' \
  "  <remote name=\"origin\" fetch=\"$GL/\" />" \
  '  <default remote="origin" revision="main" sync-j="8" />'; }
shared_projects() {
  echo '  <project name="shared/alice" path="app/alice" groups="base" />'
  echo '  <project name="shared/tool"  path="tool"      groups="base" />'
  for ap in "${APS[@]}"; do
    echo "  <project name=\"shared/bsp/$ap-toolchain\" path=\"bsp/$ap/toolchain\" groups=\"$ap\" />"
  done; }

start shared/manifest-item
{ header; shared_projects; echo '</manifest>'; } > default.xml
finish shared/manifest-item

start internal/manifest-core
{ header; shared_projects
  echo '  <project name="internal/core"   path="app/alice/core"   groups="base" />'
  echo '  <project name="internal/driver" path="app/alice/driver" groups="base" />'
  for ap in "${APS[@]}"; do
    echo "  <project name=\"internal/bsp/$ap-kernel\" path=\"bsp/$ap/kernel\" groups=\"$ap\" />"
  done
  echo '</manifest>'; } > default.xml
finish internal/manifest-core
```

In the real setup, the kernel entries point at the `gerrit` remote instead (PLAN.md §3). The demo keeps them on GitLab so it doesn't need Gerrit.

Check in the UI: open `proj/shared/alice` → `release/sm8650/libcore.a`. It should carry a **"Stored with Git LFS"** label.

---

## Step 7 — Branch protection and merge settings (every project)

Do this for all 12 projects. Free/CE has no group-wide version of these settings.

**Settings → Repository → Protected branches** → *Expand*. Find the `main` row, or add it with **Add protected branch**:

| Field | Value |
|---|---|
| Allowed to merge | **Developers + Maintainers** |
| Allowed to push and merge | **No one** |
| Allowed to force push | **off** |

**Settings → Merge requests**:

| Section | Value |
|---|---|
| Merge method | **Fast-forward merge** |
| Merge options | tick **Enable "Delete source branch" option by default** |
| Squash commits when merging | **Require** |
| Merge checks | tick **All threads must be resolved**. Leave **Pipelines must succeed** off until CI reports to GitLab; with no pipeline, it blocks every merge |

**Save changes** at the bottom of each section.

Once "No one" can push to `main`, the seed script can't push to `main` again. From here on, every change goes through an MR, including manifest changes.

---

## Step 8 — Verify (as each identity)

Log in as `item1` (in a private browser window):

- [ ] Group `proj` shows only the `shared` subgroup. Opening `/proj/internal` directly returns **404**.
- [ ] Searching for `core` or `manifest-core` finds nothing.
- [ ] `repo init` with the **Core** manifest fails with an access error.

As `item1`'s Linux user:

```sh
mkdir ws && cd ws
repo init -u ssh://git@gitlab.example.com:2222/proj/shared/manifest-item.git -b main --git-lfs -g base,sm8650
repo sync -c
ls bsp/                         # only sm8650
cd app/alice && make AP=sm8650  # "core: linking release/sm8650/libcore.a"
```

As `core1`'s Linux user (same steps, `internal/manifest-core.git`):

```sh
cd app/alice && make AP=sm8650  # "core: building from source"
git status                      # core/ and driver/ do NOT appear
repo status                     # shows changes across all repos
```

Test the review flow as `item1`:

```sh
cd app/alice
repo start test-mr .
echo change >> user/README && git commit -am "Test MR"
git push origin HEAD:main       # must be REJECTED (protected branch)
git push -u origin test-mr      # prints a "create merge request" link
```

Open the link → **Create merge request** → (as `item2`) **Approve** → **Merge**. On `main` it should land as a single fast-forward commit, and the `test-mr` branch should be deleted.

---

## Day-to-day admin

| Task | Where |
|---|---|
| New Core developer | `core-developers` → Manage → Members → Invite members (Developer) |
| New Item developer | `item-developers` → Manage → Members → Invite members (Developer) |
| Someone leaves | Remove them from their people group, then block them in **Admin → Overview → Users** |
| New Shared repo | create it under `proj/shared` (or a subgroup) → Step 7 settings → add it to **both** manifests |
| New Restricted repo | create it under `proj/internal` → Step 7 settings → add it to `manifest-core` only |
