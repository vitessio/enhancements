# VEP-4 Vitess Flag Restructure

```
VEP: 4
Title: Vitess Flag Restructure
Author: Andrew Mason
Reviewers: Deepthi Sigireddi, Derek Perkins, Richard Bailey
Status: Early Drafting/Design Phase
Created: 2021-04-07
```

## Introduction

Vitess's flag usage has evolved over the life of the project into a sort of
organic sprawl.

Flags are defined on the global `flag.FlagSet` in the packages that use them,
meaning that a binary that imports that package (or imports a package which
imports a package which imports ....) implicitly gets that flag defined and
that flag will appear in the binary's help text, whether the flag makes sense
or not. This impacts the discoverability and ease-of-use of the various
binaries.

In addition, this imposes maintenance burdens. When a new flag is introduced,
care must be taken to avoid name collisions (even if you are adding a flag
which is only used by `vttablet`, it may end up importing a flag that's really
only used by vtgate but happens to share the same name), and it is difficult to
extract small pockets of functionality, because you end up importing all the
flags anyway.

These two issues are somewhat entangled with each other, and are some of the
primary motivations, in addition to overall binary size, raised in a more
general binary structure RFC in [vitessio/vitess#7471].

This VEP aims to cover just the flag restructuring, in a way that can be folded
into the broader refactor with few if any (ideally, none at all) changes.
However, I believe this work is important enough on its own, such that we
should do it regardless of that work stream, but also we should design this
work in a way that is compatible with that work stream, if that distinction
makes sense.

### Goals

The design should aim to achieve the following:
1. Importing vitess packages (e.g. `package topo`) should not mutate the global
   flagset implicitly. Binaries should need to be explicit about exactly which
   packages' flags should be usable by that binary, including at the cost of
   increased verbosity in the code.
2. The design should enable a modular approach to vitess commands and
   subcommands, as laid out in [vitessio/vitess#7471].
3. The design should not prohibit a move to config-file based approaches,
   such as [viper].

### Non-Goals

1. The design should not prescribe a particular approach to vitess commands and
   subcommands, as that is a slightly higher-level, although related, concern.
2. The design should not _require_ a move to a library such as [viper].

## Design

Our primary goal is to get flags out of implicitly appearing in the global
flagset. To do this, we will make a minor refactor to every package that defines
flags, one at a time. The gist of each package's refactor is as follows:

<details>
<summary>Before</summary>

```go
package topo

var (
	// topoImplementation is the flag for which implementation to use.
	topoImplementation = flag.String("topo_implementation", "", "the topology implementation to use")

	// topoGlobalServerAddress is the address of the global topology
	// server.
	topoGlobalServerAddress = flag.String("topo_global_server_address", "", "the address of the global topology server")

	// topoGlobalRoot is the root path to use for the global topology
	// server.
	topoGlobalRoot = flag.String("topo_global_root", "", "the path of the global topology data in the global topology server")

	// RemoteOperationTimeout is used for operations where we have to
	// call out to another process.
	// Used for RPC calls (including topo server calls)
	RemoteOperationTimeout = flag.Duration("remote_operation_timeout", 30*time.Second, "time to wait for a remote operation")
)
```
</details>

<details>
<summary>After</summary>

```go
package topo

import (
    // ... other imports elided

    "github.com/spf13/pflag"
)

var (
	// topoImplementation is the flag for which implementation to use.
	topoImplementation string
	// topoGlobalServerAddress is the address of the global topology
	// server.
	topoGlobalServerAddress string
	// topoGlobalRoot is the root path to use for the global topology
	// server.
	topoGlobalRoot string
	// remoteOperationTimeout is used for operations where we have to
	// call out to another process.
	// Used for RPC calls (including topo server calls)
	remoteOperationTimeout = 30*time.Second
)

func AddFlags(fs *pflag.FlagSet) {
    fs.StringVar(&topoImplementation, "topo_implementation", "", "the topology implementation to use")
    fs.StringVar(&topoGlobalServerAddress, "topo_global_server_address", "", "the address of the global topology server")
    fs.StringVar(&topoGlobalRoot, "topo_global_root", "", "the path of the global topology data in the global topology server")
    fs.DurationVar(&remoteOperationTimeout, "remote_operation_timeout", 30*time.Second, "time to wait for a remote operation")
}

func RemoteOperationTimeout() time.Duration {
    return remoteOperationTimeout
}
```
</details>
<br>

And then every reference to `*topo.RemoteOperationTimeout` throughout the
codebase needs to update to `topo.RemoteOperationTimeout()`. Because these
refactors do not affect RPC-boundaries, we can make these breaking changes at
any point (i.e. a `vtgate` running a version of the code after the refactor and
talk to a `vttablet` running a version of the code before the refactor without
issue). In addition, the various `./go/cmd/<component>/main.go` files would
need to audit whether they depend on these flags, and be sure to add a call to
`topo.AddFlags(flag.CommandLine)`, which would add the newly-scoped
`topo`-related flags back into the global flagset, but now it's explicit rather
than implicit.

I'll also point out that `./go/vt/servenv` uses this pattern specifically for
the `-socket_file` flag, but none of the other flags in that package.

### Tradeoffs

* ➕ Pro: This refactoring form allows us to work through each packages' flags incrementally.
  We can refactor a single package, and audit any binaries that may rely on it as a single, contained PR; it's not an all-at-once/all-or-nothing refactor.
* ➕ Pro: Making all flags package-private gives the package that "owns" the flag complete control
  over how that flag is accessed (e.g. could guard a flag behind a mutex if we wanted to).
  * ➕ Pro: Having complete control over the access gives us the ability to swap out access "implementations"
  transparently to callers, paving the way for us to drop in libraries like [viper] in cases where it's appropriate, but without making that a hard requirement.
  * ➖ Con: There are currently certain places where we mutate another package's flag value (mostly in tests, but we would need to do an audit of this). "Package-private variable + getter function" won't fit this model, so we would need to refactor that code, or provide setter functions in some cases.
* ➖ Con: We need to specify the default values for each flag twice. Once, where the
  variable is declared, to cover the case where the binary never made a call to
  `AddFlags()` and we should implicitly rely on the default value. Then, a
  duplication of the default value must also appear within `AddFlags()` to pass
  to the relevant `flag.<Type>Var(...)` call.
* ⚠️ Moving to `pflag` implicitly changes all our flags from `-this-style` to
  `--this-style`. **We will need to be very clear about this in the release
  notes.**

### Subpackages

(NB: I'm not convinced this is a good idea at this point, but I wanted to put
it out there for criticism).

To go _even further_ down the path of modularizing/scoping flags, we could
make subpackages to separate client- and server-related flags. For example, you
could imagine `topo` flags that only make sense on a server-side (e.g.
[these][topo server flags]), whereas the `RemoteOperationTimeout` flag makes
sense for both server and client binaries.

To scope these, we could create:

```
❯ tree ./go/vt/topo/*flags
./go/vt/topo/commonflags
./go/vt/topo/serverflags
```

and then client binaries would import only `topo/commonflags`, whereas server
binaries would import `topo/serverflags` as well.


### More complex cases

* Packages that set up flags in `init` functions vs just `var` (e.g. `package healthcheck`).

[topo server flags]: https://github.com/vitessio/vitess/blob/d52d854826b35da13af281f1e2879e8fe01c6802/go/vt/topo/server.go#L154-L162
[viper]: https://github.com/spf13/viper
[vitessio/vitess#7471]: https://github.com/vitessio/vitess/issues/7471
