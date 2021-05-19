# VEP-5 Vitess Bugfixes

```
VEP: 5
Title: Vitess Bugfixes
Author: Andres Taylor
Reviewers: 
Status: Proposed
Created: 2021-05-19
```

## Abstract

[VEP-1](https://github.com/vitessio/enhancements/blob/master/veps/vep-1.md) established a support lifecycle for Vitess. This VEP suggests a change to how bugfixes are handled and thus replaces the Support lifecycle section in VEP-1. 

All other sections of VEP-1 will continue to be in force without any change.

## Support Lifecycle
Because the release cycle includes a new release every 12 weeks, most fixes will only be pushed to the master branch.
An exception to this process will be made for high severity bugs.
For these bug fixes, the fix will be merged into the oldest supported release branch where the bug is manifested.
The fix will be available in the weekly release for all currently supported versions.

High severity bugs in this context include CVEs, data corruption, wrong results or outage inducing issues. The Vitess team MAY decide to only fix bugs in release branches with higher user impact.
