# Buildozer Demo

This project demonstrates the use of [buildozer](https://github.com/bazelbuild/buildtools/blob/master/buildozer/README.md), a command-line tool for programmatically editing Bazel `BUILD` files.

## What is Buildozer?

Buildozer is part of the [Bazel buildtools](https://github.com/bazelbuild/buildtools) suite. It rewrites `BUILD` files by operating directly on the Starlark Abstract Syntax Tree (AST), which means it can safely add, remove, or modify rules and attributes without disturbing formatting, comments, or unrelated content.

Typical use cases include:

- **Repository-wide refactoring** — rename a rule, swap one dependency for another, or add a tag across hundreds of packages in a single command.
- **Scripted maintenance** — pipe a list of targets into buildozer to keep `BUILD` files in sync with code changes or policy requirements.
- **Introspection** — query `BUILD` files for attribute values without loading the full Bazel graph.

### Command format

```
buildozer '<command> <args>' <target>
```

- `<command>` is an action like `add`, `remove`, `set`, `print`, `new`, or `delete`.
- `<target>` uses standard Bazel label syntax (`//package:target`). Use `//package:__pkg__` to address the package itself (e.g. when creating a new rule) and `//package:all` to match every rule in a package.

## Project structure

This is a minimal TypeScript project built with [Bazel 9](https://bazel.build/) and [aspect_rules_ts](https://github.com/aspect-build/rules_ts). The `src/` package contains a `ts_project` target that compiles two TypeScript source files and a `filegroup` that collects them.

```
src/
├── BUILD.bazel      # ts_project "lib" + filegroup "sources"
├── greeting.ts      # greeting/farewell functions
├── index.ts         # re-exports from greeting
└── tsconfig.json    # TypeScript compiler options
```

The starting `BUILD` file looks like this:

```starlark
load("@aspect_rules_ts//ts:defs.bzl", "ts_project")

package(default_visibility = ["//visibility:public"])

ts_project(
    name = "lib",
    srcs = glob(["*.ts"]),
    declaration = True,
    transpiler = "tsc",
    tsconfig = "tsconfig.json",
)

filegroup(
    name = "sources",
    srcs = glob(["*.ts"]),
)
```

## CI workflow

The GitHub Actions workflow (`.github/workflows/buildozer.yml`) installs buildozer and walks through a series of commands that modify the `BUILD` file above.

### 1. Install buildozer

The workflow downloads the pre-built binary from the [buildtools releases](https://github.com/bazelbuild/buildtools/releases) page:

```yaml
- name: Install buildozer
  run: |
    curl -fL -o /usr/local/bin/buildozer \
      https://github.com/bazelbuild/buildtools/releases/download/v8.5.1/buildozer-linux-amd64
    chmod +x /usr/local/bin/buildozer
```

### 2. Build the project

Before any modifications, the workflow verifies that the project compiles:

```bash
bazel build //...
```

### 3. Print all targets

```bash
buildozer 'print name kind' //src:all
```

Lists every rule in the `src` package together with its type. Output:

```
lib ts_project
sources filegroup
```

### 4. Add a tag

```bash
buildozer 'add tags no-remote-cache' //src:lib
```

Appends `no-remote-cache` to the `tags` list attribute of the `lib` target. If `tags` does not exist yet, buildozer creates it.

### 5. Set an attribute

```bash
buildozer 'set testonly True' //src:sources
```

Sets the boolean `testonly` attribute on the `sources` filegroup.

### 6. Add and remove a dependency

```bash
buildozer 'add deps //utils:helpers' //src:lib
buildozer 'remove deps //utils:helpers' //src:lib
```

Demonstrates round-tripping a dependency: `add` inserts a value into a list attribute; `remove` takes it back out.

### 7. Show the diff

```bash
git diff src/BUILD.bazel
```

Prints the cumulative changes buildozer has made so far, making it easy to review modifications before committing.

### 8. Create a new rule

```bash
buildozer 'new filegroup assets' //src:__pkg__
buildozer 'add srcs README.md' //src:assets
```

Creates a brand-new `filegroup` rule named `assets` in the `src` package, then adds `README.md` to its `srcs`. The special target `//src:__pkg__` tells buildozer to operate on the package rather than an existing rule.

### 9. Print the final BUILD file

```bash
cat src/BUILD.bazel
```

Displays the fully modified `BUILD` file so the result of every preceding step is visible in the workflow log.
