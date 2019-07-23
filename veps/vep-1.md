# VEP-1 Vitess Release Cycle

```
VEP: 1
Title: Vitess Release Cycle
Author: Morgan Tocker
Status: Proposed
Created: 2019-07-22
```

## Abstract

Vitess releases have historically been infrequent and not on a fixed release schedule. This has led to a status quo of advanced users maintaining their own branches and pulling in changes from Vitess master.

While keeping _master_ in a permanent stable state is desirable, there will be future scenarios where breaking backward compatibility is desired. This VEP aims to codify a release cycle with rules around backward compatibility, supported upgrade paths, and deprecation schedule.

Definitions of *MUST*, *SHOULD*, *MAY* from [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## Motivation

The motivation for stable releases is two-fold:

1. Stable releases are easier to consume by novice users. Bug reports, documentation, and support requests will be easier to follow when the user's deployed version is clearer.

2. Advanced users benefit from a clear cycle from which to merge upstream changes into their branch. This also allows for incompatible changes to be introduced at known intervals, which can help improve the health of the project.

## Versioning System

The proposed release cycle is best described as _Monotonic Versioning_ with a release cadence similar to Kubernetes. The terminology `COMPATIBILITY.RELEASE` used in this document is directly sourced from the [Monotonic Versioning Manifesto](http://blog.appliedcompscilab.com/monotonic_versioning_manifesto/):

1. The `COMPATIBILITY` number will be incremented every 12 weeks.
2. Docker images will be tagged weekly as `COMPATIBILITY.yy+YYYYMMDD`.
3. Each supported version of Vitess will be tagged weekly.
4. Each `COMPATIBILITY` number will be supported by the Open Source Vitess community for 9 months after the initial release. _Supported Fixes to Stable Releases_ is further described below.

Monotonic versioning itself is a modified version of [Semantic Versioning](https://semver.org/).

## Backwards and Forwards Compatibility Promise

For each `COMPATIBILITY` number, both backwards and forwards compatibility **MUST** be ensured. To use hypothetical examples:

### Example 1: Deprecating an unsafe of architecturally complex feature

Version `3.xx` of Vitess supports a query that is determined to be dangerous, and/or requires complex architectural support which Vitess needs to be deprecated to improve the code quality of the project:

```
# A hypothetical dangerous query!
SELECT 1;
```

Vitess `4.xx` **MUST** still support this query, because blocking it would be backwards incompatible. However, Vitess `4.xx` **MAY** add an option to _prevent unsafe queries_. It is expected that the default behavior of _prevent unsafe queries_ would be `FALSE`. Executing the query **SHOULD** result in a deprecation warning.

When Vitess `5.xx` is released, the default for _prevent unsafe_queries_ **MAY** be changed to `TRUE`. Waiting until Vitess `5.xx` to change the default makes the behavior _forwards compatible_ because users can test out the change prior to upgrading. Attempting to execute the query **SHOULD** produce an error stating that the `4.xx` behavior can be restored by changing a setting, but the setting is deprecated and will be removed soon.

When Vitess `6.xx` is released, the option to _prevent unsafe queries_ **MAY** be removed with the implied behavior that the option is permanently `TRUE`.

_Based on the impact of functionality being removed, the Vitess team **MAY** at their discretion extend the deprecation cycle. This example serves to demonstrate the minimum time required._

### Example 2: Deprecating and removing an obsolete utility

Version `3.xx` of Vitess supports a utility that has been made obsolete by recently added functionality:

```
vtdescribe: an obsolete utility
```

Vitess `4.xx` of Vitess **SHOULD** invoke a warning when attempting to use this utility, with the utility completely removed in Vitess `5.xx`.

_This change is backward compatible in `Vitess 4.xx` because the utility can still be used. It is forwards compatible because attempting to use it will suggest the alternative functionality which is available._

## Supported Fixes to Stable Releases

Because the release includes a new release every 12 weeks, bug fixes will typically only be pushed to the master branch. An exception to this process will be made for high severity bugs, in which case a fix will be available in the weekly release for all currently supported versions.

High severity bugs in this context include CVEs, data corruption, wrong results or outage inducing issues. The Vitess team **MAY** only backport bugs with higher visibility or user impact.

## Supported Upgrade Paths

Because the _Backwards and Forwards Compatibility Promise_ is only between single major versions, it is not recommended to skip versions when upgrading. i.e.

To use _Example 1: Deprecating an unsafe of architecturally complex feature_, a user upgrading from Vitess 3.x to 6.x would encounter a situation where a query works without warning or error, to the query is completely removed.

The Vitess team **MUST** document incompatibilities so that users who require skip-version upgrades can achieve this provided they read all release notes between versions.

## References

* [RFC2119](https://www.ietf.org/rfc/rfc2119.txt)
* [Kubernetes Release Versioning](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md)
* [Monitonic Versioning](http://blog.appliedcompscilab.com/monotonic_versioning_manifesto/)
* [Semantic Versioning](https://semver.org/)
