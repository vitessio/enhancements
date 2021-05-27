# # VEP-5 Vitess Release cadence and Bugfixes

```
VEP: 5
Title: Vitess Bugfixes
Author: Andres Taylor
Reviewers: Derek Perkins, Rafael Chaco, Harshit Gangal, Deepthi Sigireddi
Status: Proposed
Created: 2021-05-19
```

## Abstract

[VEP-1](https://github.com/vitessio/enhancements/blob/master/veps/vep-1.md) established a support lifecycle for Vitess. [VEP-2](https://github.com/vitessio/enhancements/blob/master/veps/vep-2.md) later amended parts of the
This VEP suggests a change to how bugfixes are handled, how often releases are made, and how long they are supported for and thus replaces the Support lifecycle section in VEP-1, and all of VEP-2.

All other sections of VEP-1 will continue to be in force without any change.

## Support Lifecycle
Because the release cycle includes a new release every 4 months, most fixes will only be pushed to the master branch.
An exception to this process will be made for high severity bugs.
For these bug fixes, the fix will be applied to all supported release branches where the bug is manifested.

Patch releases for supported versions are released at irregular intervals, depending on what has been fixed on the branch.

High severity bugs in this context include CVEs, data corruption, wrong results or outage inducing issues. The Vitess team MAY decide to only fix bugs in release branches with higher user impact.

## Versioning and Release Cadence

1. The `COMPATIBILITY` number will be incremented every 4 months. For example, the next major release will have a `COMPATIBILITY` number of `11.0`.
2. Each `COMPATIBILITY` number will be supported by the Open Source Vitess community for 12 months after the initial release. _Support Lifecycle_ is described in VEP-1.
3. Patch releases will be created as needed. Patch releases will follow SemVer. The numbering scheme will be `COMPATIBILITY.PATCH_NUMBER`. For example, the first patch release on `7.0`, if there is one, will be `7.0.1`. `PATCH_NUMBER` will be monotonically increasing.
4. Events that trigger patch releases are major regressions in functionality or performance, and CVEs.

