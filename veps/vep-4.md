# # VEP-4 Vitess Flag Restructure, `pflag`, and `viper`

```
VEP: 4
Title: Vitess Flag Restructure, `pflag`, and `viper`
Author: Andrew Mason
Reviewers: Deepthi Sigireddi, Derek Perkins, Richard Bailey
Status: In Review
Created: 2021-04-07 (second major revision 2021-11-03)
```

## Abstract

This VEP suggests changes and outlines a migration path, in phases, to restructure flag use across the Vitess codebase, migrate to [`pflag`][pflag], and eventually to [`viper`][viper].

## Introduction

Vitess's flag usage has evolved over the life of the project into a sort of organic sprawl.

Flags are defined on the global `flag.FlagSet` in the packages that create them — a binary that imports that package (or imports a package which imports a package which imports ... and so on) implicitly gets that flag defined and that flag will appear in the binary's help text [1], whether the flag makes sense or not.
This impacts the discoverability and ease-of-use of the various binaries.

In addition, this imposes maintenance burdens.
When a new flag is introduced, care must be taken to avoid name collisions (even if you are adding a flag which is only used by `vttablet`, it may end up importing a flag that's really only used by vtgate but happens to share the same name), and it is difficult to extract small pockets of functionality, because you end up importing all the flags anyway.

These two issues are somewhat entangled with each other, and are some of the primary motivations, in addition to overall binary size, raised in a more general binary structure RFC in [vitessio/vitess#7471].

## tl;dr

1. Deprecate flag usages that are incompatible with `pflag`.
1. Switch to `pflag`.
1. Code refactor to encapsulate flag-backed variables.
1. Adopt `viper`.

## Open Questions

1. The changes in phases 1 and 2 _can_ break existing scripts.
Details of exactly how and why are described in each phase.

    Users can actually already update the way they pass flags and subcommands in a way that won't break, without any code changes needed in Vitess.

    The question: **Do we need to do a formal deprecation cycle?**
    Or can we announce in the monthly meeting and in the release notes and proceed with the migration, saving ourselves 3 months of waiting?

## Migration

### Phase 1: Preparation

Unfortunately, we cannot simply drop-in `pflag` as a replacement for `flag`, despite its advertisement to support exactly that.

Since the very beginning, Vitess documentation has encouraged the use of the single-dash form of flags (e.g. `-my_flag`).
`pflag` actually treats these differently from double-dash (e.g. `--my_flag`), while the standard library `package flag` treats these as the same.

Attempting to drop-in `pflag` results in errors such as the following, when trying to run the `vtctl` binary:

```
unknown shorthand flag: 't' in -topo_implementation
```

So, before we can switch to `pflag` (see [Open Questions](#Open_Questions)), we need to tell people to update any scripts to use the double-dash forms, which, while not advertised, are already supported.

_Technically_, users could have begun making this transition starting with Vitess version 1, so I am not sure how strictly we want to adhere to a dedicated two-release deprecation+removal cycle.
We could just tell people to update and make the breaking change in v13.
If we want to do the more formal deprecation,
we can do the following:

```go
// go/vt/servenv/servenv.go

func parse() {
    flag.Parse()

	argv := os.Args
	flag.Visit(func(f *flag.Flag) {
		for _, arg := range argv {
			if strings.HasPrefix(arg, "-"+f.Name) {
				log.Warningf("Use of single-dash long flags is deprecated and will be removed in 14.0. Please use --%s instead", f.Name)
			}
		}
	})
}

// then replace all calls to flag.Parse() with a call to parse()
```

Which, when running the same `vtctl` binary, produces output like:

```
W1031 13:17:56.497111   88580 servenv.go:270] Use of single-dash long flags is deprecated and will be removed in 14.0. Please use --backup_storage_implementation instead
W1031 13:17:56.497648   88580 servenv.go:270] Use of single-dash long flags is deprecated and will be removed in 14.0. Please use --cell instead
```

The second issue with dropping in `pflag` is that it supports mixing of positional and flag arguments.
**This change will break subcommands, such as those in `vtctl` and `mysqlctl`.**

To illustrate the problem, consider the command `./mycmd -a 1 -b 2 X -c 3 -d 4`.
With `flag`, parsing this will result in two flags, `a` and `b`, `flag.NArg()` will return 5 and `flag.Args()` will return the slice `["X", "-c", "3", "-d", "4"]` ([c.f.](https://play.golang.org/p/sO4UlQPGg4l)).
With `pflag`, all 4 flags will be parsed, `pflag.NArg()` will return 1, and `pflag.Args()` will return the slice `["X"]`.

In order to match this behavior, users should update their commands to insert a double-dash (`--`) before the first positional argument.
This causes `pflag` to stop looking for flags and treat everything after as a positional argument (matching what `flag` does even without the double-dash), and has no effect on the parsing behavior of `flag` ([c.f.](https://play.golang.org/p/76FrFrLFOMT)).

As with the single-dash long flag issue, we can make code changes similar to the `parse` snippet above to warn users and target v14 for switching, or we can announce and move to `pflag` immediately.

### Phase 2: `pflag` adoption and migration

After the preparatory work we did in phase 1, dropping in `pflag` is fairly straightforward.
For example, `servenv` has two functions, `ParseFlags` and `ParseFlagsWithArgs`.
Taking the first as an example, we would write:

```go
import (
    goflag "flag"
    flag "github.com/spf13/pflag" // minimize code changes
)
func ParseFlags(cmd string) {
    flag.CommandLine.AddGoFlagSet(goflag.CommandLine) // (TODO) Remove after all other packages add flags directly to pflag.
	flag.Parse()

	if *Version {
		AppVersion.Print()
		os.Exit(0)
	}

	args := flag.Args()
	if len(args) > 0 {
		flag.Usage()
		log.Exitf("%s doesn't take any positional arguments, got '%s'", cmd, strings.Join(args, " "))
	}
}
```

Now, other packages which declare flags can begin moving to `pflag`.
Once all packages are adding their flags directly to the `pflag.CommandLine` flagset, we can remove the "TODO" interop line.
Now, other packages can gradually move over to `pflag`, and once everything is adding their flags directly to the `pflag` flagset, we can remove the interop line.

Note that there are other places that call `flag.Parse` directly (most in `go/test`, some in `go/cmd`) that will need the same treatment.
In addition, places with subcommands will need to update callsites to take a `pflag.FlagSet`, but the changes there should be fairly minimal.
For example, modulo running `goimports`, [this][mysqlctl_pflag_diff] is the full diff required to compile `mysqlctl`.

We can make a GitHub project to divide and conquer this migration.

### Phase 3: Restructuring and encapsulation

This phase aims to address two problems:
1. Packages that get imported by various components all install their flags in the global flagset, regardless of whether they should show in that components usage/help text.
1. Flag variables that are not exported by a package are not accessible by other packages (see [vitessio/vitess#9026]), but exporting those variables allows anyone to change their values, and in a non-threadsafe manner (see [`NewVtctldServerWithTabletManagerClient`][NewVtctldServerWithTabletManagerClient], for example).

For currently-private variables, this is simple:
* If we have a need to export the variable, add a function named with the public version of the name (e.g. `var myVar T` gets `func MyVar() T { return myVar }` ).
* If we have no need to export the variable, do nothing.

For currently-exported variables, this is trickier. My proposal:

Currently:
```go
// go/vt/vttablet/tmclient/client.go
package tmclient

import "flag"

var TabletManagerProtocol = flag.String(...)
```

After:
```go
// go/vt/vttablet/tmclient/client.go
package tmclient

import flag "github.com/spf13/pflag"

var clientProtocol string

func AddFlags(fs *flag.FlagSet) {
    fs.StringVar(&clientProtocol, "tablet_manager_client_protocol", ...)
}

func TabletManagerProtocol() string {
    return clientProtocol
}

// mypkg/some_test.go
package mypkg

import (
    "testing"

    flag "github.com/spf13/pflag"
)

func TestMyFunc(t *testing.T) {
    ...
    flag.Set("tablet_manager_client_protocol", "mytestvalue")
}
```

Commentary:
* This makes flag variables read-only ... ish.
The only way to set `clientProtocol` outside of `package tmclient` is with a call to `pflag.Set`.
This is easy to lint, and we can enforce it by convention in code review, or we can add a CI hook if it becomes frequent enough to warrant the extra effort.
* It does **not** make flag-setting in tests threadsafe.
This is a bummer to me, as it reduces the number of tests we can mark as `t.Parallel()`, but that's not one of our goals, so figuring that out can wait for later and should not stop us from making this clear improvement to The Flag Situation.
* It allows us to import packages without polluting help text with flags if we just need one or two utility functions.
* As we move each non-`cmd` package to this form, we will need to audit each `cmd`, determine if they need those flags, and add the appropriate call to `AddFlags`.
* Packages that depend on an appropriate default value need to duplicate that default in the flag definition and the variable declaration ([c.f.](https://play.golang.org/p/aWPA-IgiGq_r)).
For example, it should be:
```go
package topo
import (
    "time"

    flag "github.com/spf13/pflag"
)

var remoteOperationTimeout = time.Second*30
func AddFlags(fs *flag.FlagSet) { fs.DurationVar(&remoteOperationTimeout, "remote_operation_timeout" time.Second*30, "...") }
```

and not:
```go
package topo

import (
    "time"

    flag "github.com/spf13/pflag"
)

var remoteOperationTimeout time.Duration
func AddFlags(fs *flag.FlagSet) { fs.DurationVar(&remoteOperationTimeout, "remote_operation_timeout", time.Second*30, "...") }
```

### Phase 4: `viper` and beyond

At this point, we're using `pflag` everywhere, and our flags are well-encapsulated within the packages that define them.
However, there's more we can improve!

For binaries that have many options (like `vttablet`), specifying all of those on the command-line is cumbersome.
In addition, we currently don't support live-reloading of configuration variables.
Live-reload is a huge usability win; consider for example being able to adjust an operation timeout without having to restart the component ... :chefs-kiss:

The good news is: we can get all of this with `viper`!

The migration is straightforward.
In any place we call `pflag.Parse()`, we add a call to `viper.BindPFlags(pflag.CommandLine)`.
Technically, we can add this line anywhere before we first attempt to read a value out of `viper`, since values are lazy-loaded, but colocating with the `Parse` call makes things easier to see we've set everything up correctly.

Then, we update all of our flag getter functions to read from `viper` instead of directly from the flag variable.

Before:
```go
package tmclient

import flag "github.com/spf13/pflag"

var tabletManagerProtocol string

func AddFlags(fs *flag.FlagSet) { fs.StringVar(&tabletManagerProtocol, "tablet_manager_protocol", ...) }

func TabletManagerProtocol() string { return tabletManagerProtocol }
```

After:
```go
package tmclient

import (
    flag "github.com/spf13/pflag"
    "github.com/spf13/viper"
)

var tabletManagerProtocol string

func AddFlags(fs *flag.FlagSet) { fs.StringVar(&tabletManagerProtocol, "tablet_manager_protocol", ...) }

func TabletManagerProtocol() string { return viper.GetString("tablet_manager_protocol") }
```

Then, to support config files, we need the following code:

```go
viper.SetConfigName("config") // looks for config.{yaml,yml,json,toml,...}
viper.AddConfigPath("/etc/vitess/vttablet") // if we wanted to namespace by component. you do need at least one path, though
viper.AddConfigPath(".")

if err := viper.ReadInConfig(); err != nil {
    if _, ok := err.(viper.ConfigFileNotFound); ok {
        // warn and continue
        log.Warningf("config file not found: %s", err)
    } else {
        log.Fatalf("error loading config: %s", err)
    }
}

// same as before:
viper.BindPFlags(pflag.CommandLine)
pflag.Parse()
```

I need to verify this, but I'm _pretty_ sure that, same as `BindPFlags`, we can read in configs at any point in the code before attempting to read a value from `viper`.
Again, colocating with the `flag.Parse` and `viper.BindPFlags` calls makes things simpler to reason about as the reader, in my opinion.

Finally, for live-reloading:

```go
viper.AddConfigPath(path1) // MUST have all paths added before WatchConfig
viper.OnConfigChange(func (e fsnotify.Event) {
    log.Infof(...)
})
viper.WatchConfig()

// same as before: ReadInConfig; BindPFlags; Parse
```

**NOTE**: vttablet and vtadmin have some support for yaml-based configs already; when we switch to `viper` we should update that as well, so we have a unified way config variables are loaded in from files.

[pflag]: github.com/spf13/pflag
[viper]: github.com/spf13/viper

[NewVtctldServerWithTabletManagerClient]: https://github.com/vitessio/vitess/blob/af67bf8eff7515c6dc1e14ee76fe35006216e1c3/go/vt/vtctl/grpcvtctldserver/testutil/test_tmclient.go#L51-L122
[vitessio/vitess#7471]: github.com/vitessio/vitess/issues/7471
[vitessio/vitess#9026]: github.com/vitessio/vitess/pull/9026
[mysqlctl_pflag_diff]: https://gist.github.com/ajm188/3f85a5a4922d91fb7480a8892027d0f1

## Appendix

[1]: Flag pollution

```
❯ ./bin/vtctl -h 2>&1 | grep -E '\w+-' | wc -l
     123
❯ ./bin/vtadmin -h 2>&1 | grep -E '\w+-' | wc -l
      18
```

Quite a few callsites to update, fortunately most of them are in tests:
```
❯ git grep --name-only -E "^(\w)*[^/].*flag\.Parse\(\)" | wc -l
     112
❯ git grep --name-only -E "^(\w)*[^/].*flag\.Parse\(\)" | grep -v test | wc -l
      32
```
