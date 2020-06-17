# VEP-1 Vitess Release Cycle

```
VEP: 1
Title: Vitess Release Cycle
Author: Morgan Tocker
Reviewers: Deepthi Sigireddi, Manik Taneja, Derek Perkins, David Weitzman, Rafael Chacon, Brian Ramos
Status: Approved
Created: 2019-09-13
```

## Abstract

Vitess releases have historically been infrequent and not on a fixed release schedule. This has led to a status quo of advanced users maintaining their own branches and pulling in changes from Vitess master.

While keeping _master_ in a permanent stable state is desirable, there will be future scenarios where breaking backward compatibility is desired. This VEP aims to codify a release cycle with rules around backward compatibility, supported upgrade paths, and deprecation schedule.

Definitions of *MUST*, *SHOULD*, *MAY* from [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## Motivation

The motivation for stable releases is two-fold:

1. Stable releases are easier to consume by novice users. Bug reports, documentation, and support requests will be easier to follow when the user's deployed version is clearer.

2. Advanced users benefit from a clear cycle from which to merge upstream changes into their branch. This also allows for incompatible changes to be introduced at known intervals, which can help improve the health of the project.

## Versioning and Release Cadence

The proposed release cycle is best described as _Monotonic Versioning_ with a release cadence similar to Kubernetes. The terminology `COMPATIBILITY.RELEASE` used in this document is directly sourced from the [Monotonic Versioning Manifesto](http://blog.appliedcompscilab.com/monotonic_versioning_manifesto/):

1. The `COMPATIBILITY` number will be incremented every 12 weeks.
2. Docker images will be tagged weekly as `COMPATIBILITY.yy+YYYYMMDD`.
3. Each supported version of Vitess will be tagged weekly. If no changes have been made to the branch from the prior week, a tag will not be created.
4. Each `COMPATIBILITY` number will be supported by the Open Source Vitess community for 9 months after the initial release. _Support Lifecycle_ is further described below.

Monotonic versioning itself is a modified version of [Semantic Versioning](https://semver.org/).

## Release Criteria

The release cadence is based on time-based rather than feature-based milestones. However; all releases must pass the suite of continuous integration and regression tests prior to release.

Large or compatibility breaking features will be encouraged to merge within the first 6 weeks of the development cycle. Performance testing will be completed on a recurring basis throughout the development cycle.

## Backwards and Forwards Compatibility Promise

For each `COMPATIBILITY` number, both backwards and forwards compatibility **MUST** be ensured. To use hypothetical examples:

### Example 1: Deprecating an Unsafe or Architecturally Complex Feature

Version `3.xx` of Vitess supports a query that is determined to be dangerous, and/or requires complex architectural support which needs to be deprecated to improve the code quality of the project:

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

Vitess `4.xx` **SHOULD** invoke a warning when attempting to use this utility, with the utility completely removed in Vitess `5.xx`.

_This change is backward compatible in Vitess `4.xx` because the utility can still be used. It is forwards compatible because attempting to use it will suggest the alternative functionality which is available._

## Exclusions

The following is not subject to the Backwards and Forwards Compatibility Promise:

* Functionality which has been marked as experimental. As a new feature is developed, the Vitess authors may release it in a stable release but as experimental. As feedback is received, changes may be made to the feature without respecting the regular deprecation cycle associated with backwards and forwards compatibility.

* Functionality limited to the source code and/or build environment. If the Vitess authors desired to make a change, such as upgrading the required Go version, or switching from `govendor` to Go modules, there is no requirement to ensure compatibility with older versions of Go. The promise is intended for user-facing functionality.

## Support Lifecycle

Because the release cycle includes a new release every 12 weeks, most fixes will only be pushed to the master branch. An exception to this process will be made for high severity bugs, in which case a fix will be available in the weekly release for all currently supported versions.

High severity bugs in this context include [CVEs](https://cve.mitre.org/), data corruption, wrong results or outage inducing issues. The Vitess team **MAY** decide to only backport bugs with higher user impact.

## Supported Upgrade Paths

Because the _Backwards and Forwards Compatibility Promise_ is only between single major versions, it is not recommended to skip versions when upgrading. i.e.

	To use _Example 1_ (from above): Before upgrading, a user will be able to execute the statement `SELECT 1` on their Vitess `3.xx` system without error. If the user were to upgrade directly to `5.xx` this query would now result in an error by default, with a configuration option available to temporarily permit the query again.

	Had the user upgraded to Vitess `4.xx` first, they would have received deprecation warnings when executing the soon-to-be unsupported query.

	Further, had the user upgraded directly from `3.xx` to `6.xx` there would be no longer be support for a compatibility configuration option. The query would just be unsupported.

The Vitess team **MUST** document incompatibilities so that users who require skip-version upgrades can achieve this provided they read all release notes between versions.

## References

* [RFC2119](https://www.ietf.org/rfc/rfc2119.txt)
* [Kubernetes Release Versioning](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md)
* [Monotonic Versioning](http://blog.appliedcompscilab.com/monotonic_versioning_manifesto/)
* [Semantic Versioning](https://semver.org/)
