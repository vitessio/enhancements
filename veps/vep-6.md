# # VEP-6 Vitess Release cadence

```
VEP: 6
Title: Vitess Release Cadence
Author: Harshit Gangal
Reviewers: 
Status: 
Created: 2024-11-02
```

## Abstract

[VEP-1](https://github.com/vitessio/enhancements/blob/main/veps/vep-1.md) established a support lifecycle for Vitess. [VEP-2](https://github.com/vitessio/enhancements/blob/main/veps/vep-2.md) later amended parts of it.
[VEP-5](https://github.com/vitessio/enhancements/blob/main/veps/vep-5.md) modified the yearly major release to 3 and 1 year support for each major release, thus replaces the Support lifecycle section in VEP-1, and all of VEP-2.
This VEP suggests a change to how often releases are made and its support lifecycle thus replacing all of VEP-5.

All other sections of VEP-1 will continue to be in force without any change.

## Versioning and Release Cadence

The proposed release cycle is best described as _Monotonic Versioning_ with a release cadence similar to Kubernetes.
The terminology `COMPATIBILITY.RELEASE` used in this document is directly sourced from the [Monotonic Versioning Manifesto](http://blog.appliedcompscilab.com/monotonic_versioning_manifesto/):

1. The `COMPATIBILITY` number will be incremented every 6 months. For example, the next major release will have a `COMPATIBILITY` number of `22.0`.
2. Each `COMPATIBILITY` number will be supported by the Open Source Vitess community for 12 months after the initial release.
3. Patch releases will be created as needed. Patch releases will follow SemVer. The numbering scheme will be `COMPATIBILITY.PATCH_NUMBER`. For example, the first patch release on `22.0`, if there is one, will be `22.0.1`. `PATCH_NUMBER` will be monotonically increasing.
4. Events that trigger patch releases are major regressions in functionality or performance, and CVEs.

## Support Lifecycle
A new version is released every 6 months, with each version supported for 12 months. At any time, two versions are actively supported: the latest release and the one prior.

Bug fixes are applied to the main branch, except for high-severity issues, which will be patched in all supported branches where the issue occurs.

Patch releases for these supported versions are released as needed, depending on the fixes available in each branch.

High-severity issues in this context include CVEs, data corruption, incorrect results, or issues causing outages. The Vitess team may choose to prioritize bug fixes in branches with the highest user impact.
