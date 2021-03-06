---
authors: Andrew Lytvynov (andrew@goteleport.com)
state: implemented
---

# RFD 12 - Teleport versioning

## What

Versioning scheme for Teleport releases (post-5.0).

### Terminology

Quick note on terminology used below. I'll use naming from
[semver](https://semver.org):

```
vX.Y.Z
 ^ ^ ^
 | | *- "patch version Z"
 | *- "minor version Y"
 *- "major version X"
```

## Why

Teleport has ~4 big releases per year.

Up to v5.0.0, Teleport used a versioning scheme that looks like semver (but
actually isn't):
- a minor version is bumped for all regular releases (e.g. 4.3 -> 4.4)
- a major version is bumped when a release _feels_ particularly significant
  (e.g. 4.4 -> 5.0 with introduction of application access)
- a patch version is bumped for bug patches

Our compatibility promise is:

> When running multiple binaries of Teleport within a cluster (nodes, proxies, clients, etc), the following rules apply:
>
> * Patch versions are always compatible, for example any 4.0.1 component will work with any 4.0.3 component.
> * Other versions are always compatible with their previous release. This means you must not attempt to upgrade from 4.1 straight to 4.3. You must upgrade to 4.2 first.
> * Teleport clients tsh for users and tctl for admins may not be compatible with different versions of the teleport service.

The 2nd point is crucial: we **never** break compatibility with a previous
release (be it a major or a minor version bump).

The downsides of this versioning scheme are:
- the distinction between major and minor version bumps is largely driven by
  business/product and has no technical difference
- a user that doesn't read our upgrade docs carefully can assume semver
  semantics:
  - upgrading from `vX.A.B` to `vX.C.D` is safe - it's not if `C - A > 1`
  - upgrading from `vX.A.B` to `vY.C.D` is going to break things - it won't, as
    long as these are sequential releases

Therefore, I propose to switch to a more semver-like scheme, starting with 6.0.

## Details

- Major versions are for teleport releases.
- Minor versions are for regular, non-critical bugfix batches and important
  backported fixes for users.
- Patch versions are for quick followup regression and critical bug fixes.
- Suffixes:
  - `-dev` suffix is for development builds (e.g. off of the `master` branch)
  - `-alpha.X` suffix, good for demos and early deploys, but not staging
  - `-beta.X` suffix, good for staging deploys, but not production
  - `-rc.X` suffixes are for release candidates, potentially production-ready
    but still in testing

The benefits are:
- Major version bumps clearly communicate to users to exercise caution when
  upgrading, and read release notes
- Minor/patch version bumps are a no-brainer and can be automated
- Users can intuitively understand versioning semantics, due to popularity of
  semver

### Compatibility

`vN.*.*` binaries are compatible with any other `vN.*.*` or `vN-1.*.*` binaries
(both client and server).

Users are free to use any `vN.*.*` versions throughout their deployment, after
upgrading everything from `vN-1.*.*`.

### Support window

Teleport officially supports the latest major version and two previous major
versions. For example:
- `v8.*.*` (latest)
- `v7.*.*`
- `v6.*.*`

### Example: new release

The current release of teleport is `v5` and the next will be `v6`.

- initially, builds from `master` will be `v6.0.0-dev`
- when most big changes are merged, we make `branch/v6` and cut `v6.0.0-alpha.1`
- when only small bugfix changes are pending, we cut `v6.0.0-beta.1`
- all planned changes are merged, we cut `v6.0.0-rc.1`
- during release testing, we fix bugs and cut `v6.0.0-rc.2`, `v6.0.0-rc.3`, etc
- assuming `v6.0.0-rc.3` passes the tests, we tag it as `v6.0.0`
- we discover a serious bug in `v6.0.0`, fix it and cut `v6.0.1`
- over the next few days/weeks, we gather user feedback
- based on feedback, we fix a number of bugs and cut `v6.1.0`

### Example: backports

The latest 3 released versions are `v8.0.1`, `v7.1.2`, `v6.3.2`.

- we discover a bug affecting all versions
- we fix it in `master`
- if the bug is urgent, we backport to `branch/v8`, `branch/v7`, `branch/v6`
  and release `v8.0.2`, `v7.1.3`, `v6.3.3`
- if the bug is not urgent and no customer has requested backports to older
  versions, we backport to `branch/v8` and release `v8.1.0`
  - optionally, we wait for more bug fixes to accumulate in `branch/v8` before
    releasing `v8.1.0`

### Example: security vulnerability

The latest 3 released versions are `v8.0.1`, `v7.1.2`, `v6.3.2`.

- we discover a security bug affecting all versions
- we fix it in `master` and backport to `branch/v8`, `branch/v7`, `branch/v6`
- after backports merge, we release `v8.0.2`, `v7.1.3`, `v6.3.3`

### Why not use minor versions, like we do now?

An experienced reader might notice that this is not exactly semver. In semver,
major versions are for strictly breaking changes with no compatibility.

So why not keep using minor version bumps like we do now? And reserve major
version for when we really need to make a breaking change?

Because we want to keep our compatibility guarantee and avoid the Python 2/3
story. We never want a `vN+1` that has no migration path other than "rebuild
the cluster from scratch".

So we don't want to freeze the major version forever this way.
