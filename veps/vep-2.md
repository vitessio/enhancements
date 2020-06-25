# VEP-2 Vitess Release Cycle

```
VEP: 1
Title: Vitess Patch Releases
Author: Deepthi Sigireddi
Reviewers: Morgan Tocker, Rafael Chacon Vivas, Guido Iaquinti, Derek Perkins, David Weitzman
Status: Approved
Created: 2020-06-17
```

## Abstract

VEP-1(#1) established a release cycle for Vitess. Now that there have been 3 major releases, this VEP proposes some changes to the release cycle based on that experience.
All other sections of VEP-1 will continue to be in force without any change.

## Versioning and Release Cadence

1. The `COMPATIBILITY` number will be incremented every 13 weeks. For example, the next major release will have a `COMPATIBILITY` number of `7.0`.
2. These major releases will occur in the last week of January, April, July and October.
3. Each `COMPATIBILITY` number will be supported by the Open Source Vitess community for 9 months after the initial release. _Support Lifecycle_ is described in VEP-1.
4. Patch releases will be created as needed. Patch releases will follow SemVer. The numbering scheme will be `COMPATIBILITY.PATCH_NUMBER`. For example, the first patch release on `7.0`, if there is one, will be `7.0.1`. `PATCH_NUMBER` will be monotonically increasing.
5. Events that trigger patch releases are major regressions in functionality or performance, and CVEs.

## References

* [Semantic Versioning](https://semver.org/)






