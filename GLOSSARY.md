# Source Management

How the project's code is split into repos, who may see which parts, and how changes get reviewed and merged.

## Visibility

**Audience**:
A class of people defined by which code they may read. There are exactly two: Core developers and Item developers.
_Avoid_: group (clashes with GitLab Group), user (clashes with the `user/` directory)

**Core developers**:
People allowed to read and change all code, including Restricted repos.
_Avoid_: dev group, developers (too generic)

**Item developers**:
People allowed to read and contribute only to Shared repos. They build from Shared code plus Prebuilts.
_Avoid_: user group, users, partners

**Shared repo**:
A repo readable by both Audiences.

**Restricted repo**:
A repo readable only by Core developers (core, driver, kernel).
_Avoid_: dev-only repo, private repo, internal repo

**Prebuilt**:
A compiled binary of Restricted code, committed to `release/` so Item developers can build without seeing the source.
_Avoid_: release binary, lib drop

## Layout

**Repo**:
One git repository, the smallest unit of access control.

**Checkout path**:
The directory where a repo appears in a workspace. A repo's checkout path can be nested inside another repo's checkout path.

**Manifest**:
The file that lists which repos make up a workspace and their checkout paths. There is one per Audience.

**Workspace**:
A full checkout produced from a Manifest, where `make` produces a working binary.

**AP**:
A target application processor platform (e.g. sm8650). Each AP has its own BSP.
_Avoid_: chip, target, platform

**BSP**:
The per-AP toolchain and kernel.

## Review

**Merge request (MR)**:
The only way a change reaches `main`: a pushed branch plus a review.
_Avoid_: patch, change, CL, "upload"
