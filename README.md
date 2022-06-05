# rules_nodejs - [Issue #3479](https://github.com/bazelbuild/rules_nodejs/issues/3479)

This is a replication of a bug in `rules_nodejs` where the `package_path`
directive seems to only work on references within the same repository, and
does not work when the same library is referenced externally.

In this repo, there are two workspaces, `ext_ws` and `main_ws`. In `ext_ws`
a simple `js_library` is defined and can be imported by `nodejs_binary`
targets within the workspace.

However, this fails to work when the same library is references from a
`nodejs_library` in another workspace, i.e. from `main_ws`. Note that this
target contains identical source code and BUILD definition to the target in
`ext_ws` that works.

We can see this with the following commands:

**`$ cd ext_ws && bazel run //:bin`**
```bash
INFO: Analyzed target //:bin (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //:bin up-to-date:
  bazel-bin/bin.sh
  bazel-bin/bin_loader.cjs
  bazel-bin/bin_require_patch.cjs
INFO: Elapsed time: 0.105s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Build completed successfully, 1 total action
Hello world? Hello world.
```

**`$ cd main_ws && bazel run //:bin`**
```bash
INFO: Analyzed target //:bin (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //:bin up-to-date:
  bazel-bin/bin.sh
  bazel-bin/bin_loader.cjs
  bazel-bin/bin_require_patch.cjs
INFO: Elapsed time: 0.245s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Build completed successfully, 1 total action
Error: Cannot find module '@ext/ws/lib'
Require stack:
- /home/pras/.cache/bazel/_bazel_pras/df2aff05c573141092662ae5db828664/execroot/main_ws/bazel-out/k8-fastbuild/bin/bin.sh.runfiles/main_ws/bin.js
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader:933:15)
    at Function.Module._load (node:internal/modules/cjs/loader:778:27)
    at Module.require (node:internal/modules/cjs/loader:1005:19)
    at require (node:internal/modules/cjs/helpers:102:18)
    at Object.<anonymous> (bin.js:1:13)
    at Module._compile (node:internal/modules/cjs/loader:1105:14)
    at Object.Module._extensions..js (node:internal/modules/cjs/loader:1159:10)
    at Module.load (node:internal/modules/cjs/loader:981:32)
    at Function.Module._load (node:internal/modules/cjs/loader:822:12)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:77:12)
```

If we compare the execroots, we see that a `node_modules` entry is populated
in the former case, but only an empty root directory is created in the latter:

```bash
# Execroot from a run of `@ext_ws//:bin
$ find /home/pras/.cache/bazel/_bazel_pras/92b5a6b34dd09a1331f2af4192a3d949/execroot/ | grep @ext
/home/pras/.cache/bazel/_bazel_pras/92b5a6b34dd09a1331f2af4192a3d949/execroot/ext_ws/node_modules/@ext
/home/pras/.cache/bazel/_bazel_pras/92b5a6b34dd09a1331f2af4192a3d949/execroot/ext_ws/node_modules/@ext/ws
```

```bash
# Execroot from a run of `@main_ws//:bin
$ find /home/pras/.cache/bazel/_bazel_pras/df2aff05c573141092662ae5db828664/execroot/ | grep @ext
/home/pras/.cache/bazel/_bazel_pras/df2aff05c573141092662ae5db828664/execroot/main_ws/node_modules/@ext
```
