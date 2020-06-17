# VEP-2 Vitess Release Cycle

```
VEP: 1
Title: Vitess Patch Releases
Author: Deepthi Sigireddi
Reviewers: Morgan Tocker, Rafael Chacon Vivas, Guido Iaquinti
Status: Proposed
Created: 2020-06-17
```

## Abstract

VEP-1(#1) established a release cycle for Vitess. Now that there have been 3 major releases, this VEP proposes some changes to the release cycle based on that experience.
All other sections of VEP-1 will continue to be in force without any change.

## Versioning and Release Cadence

The proposed release cycle is best described as _Monotonic Versioning_ with a release cadence similar to Kubernetes. The terminology `COMPATIBILITY.RELEASE` used in this document is directly sourced from the [Monotonic Versioning Manifesto](http://blog.appliedcompscilab.com/monotonic_versioning_manifesto/). Monotonic versioning itself is a modified version of [Semantic Versioning](https://semver.org/).

1. The `COMPATIBILITY` number will be incremented every 13 weeks.
2. Each `COMPATIBILITY` number will be supported by the Open Source Vitess community for 9 months after the initial release. _Support Lifecycle_ is described in VEP-1.
3. Patch releases will be created as needed. Patch releases will follow SemVer instead of monotonic versioning. The numbering scheme will be `COMPATIBILITY.PATCH_NUMBER`.
4. Events that trigger patch releases are major regressions in functionality or performance, and CVEs.





