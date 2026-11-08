# Repo boundaries follow Audience boundaries; Shared repos under alice are merged

Git has no per-path read permissions, so the repo is the smallest unit of access control and Restricted code (core, driver, kernel) must stay in its own repos. Apart from that, nested repos only add friction. A cross-directory change took N separate patches. So the Shared repos under `app/alice` (alice, user, release, item, att, mpe) are merged into a single `alice` repo, keeping the same directory layout and their history. core and driver stay separate Restricted repos at their existing checkout paths, so nothing changes on disk for anyone.

## Considered Options

- **Single monorepo**: rejected because it can't hide Restricted code from Item developers.
- **Merge core + driver into one Restricted repo, symlinked into place**: rejected because symlinks in a build tree cause more trouble than one extra repo.
