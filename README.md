# Please Go Modules experiment

Experiments with [Please](https://please.build) and Go modules.


## Usage

Add the following snippet to your `BUILD` file in the root of your repository:

```starlark
http_archive(
    name = "pleasegomod",
    urls = [f"https://github.com/sagikazarmark/please-go-modules/releases/download/v0.0.5/gogetgen_{CONFIG.HOSTOS}_{CONFIG.HOSTARCH}.tar.gz"],
)
```

Add the following snippet to your `.plzconfig`:

```
[please]
version = 15.2.1-beta.1

[buildconfig]
moddown-tool = ///pleasegomod//:moddown

[alias "godeps"]
desc = Generate third-party dependency rules for a Go project
cmd = run ///pleasegomod//:gogetgen -- -dir third_party/go -clean -genpkg -subinclude "///pleasegomod//:build_defs"
```

Run the following:

```bash
plz godeps
```

The above command will generate build rules in `third_party/go` for your third party dependencies.

The last step is to use those dependencies.
[Wollemi](https://github.com/tcncloud/wollemi) is a great candidate for that, but it doesn't support the rules generated by this tool (yet).

You can try using this patch though: https://github.com/tcncloud/wollemi/pull/5


## Change log

This experiment went through a couple versions, hit a few dead ends.
Every major turn should be documented here so in the future we know how and why decisions have been made.


## Latest revision - 2020-09-03

After a number of attempts to make Please Go module compatible using the high level Go toolchain (`build`, `install`),
I gave up using the high level tools and turned to generating standard `go_library` rules for packages instead.

It turned out to be a great idea, especially since Please improved its Go rules to support library archives saved to non-standard GOPATH locations.

The only additional rule needed is `go_module_download` (and `go_downloaded_source`, but that's just a helper).
It downloads the module source using `go mod download`. The source is then used by a set of generated `go_library` rules.


## `go_get` with `go mod download` - 2020-06-29

Using the Go tooling for downloading turned out to be a dead end,
because in module mode object files can only be built and saved for direct dependencies, but not transitive ones.
(Chances are this is not true, but I couldn't find the right solution)

After looking at [Gazelle](https://github.com/bazelbuild/bazel-gazelle), it turns out that they don't use go modules for package management either.
This is kind of understandable, since go modules break the incremental build idea behind Bazel/Please (every change to `go.mod` would result it massive rebuilds).

Gazelle actually uses a small tool, called [fetch_repo](https://github.com/bazelbuild/bazel-gazelle/tree/5c00b77/cmd/fetch_repo) to download packages.
It supports regular `go get` mode (and more) and works with modules as well. It uses a little trick to do that: it creates a temporary module and downloads the package using `go mod download`.

I was able to integrate this tool into the existing `go_get` rule with a few changes. The working title is `go_module_get`.
If this tool gets integrated into the upstream `go_get` rule, it might worth checking if it can replace the rest of the current download logic.


## First few revisions - 2020 June

The idea is to reuse Go modules and `go.mod` to download and manage dependencies,
therefor I experimented with a couple rules.

The final rule turned out to be less automated than I'd hoped,
but worked well and used caching as much as possible.

Basically a `go_module` rule executed `go mod download` and then built the packages listed in the rule.

This worked quite well with direct dependencies, but unfortunately Go modules doesn't allow building and saving the output of transitive dependencies.
