+++
date = '2025-02-20T20:57:54+08:00'
draft = false
title = "Bazel collect2: fatal error: cannot find 'ld' with build"
tags = ['Bazel']
+++

The issue at hand stems from the linker `ld.lld`, which is specified by the `-fuse-ld=lld` option, not being present in the PATH during the binary linking process. The error message "cannot find 'ld'" can be misleading; it does not refer to `/usr/bin/ld`, but rather indicates a failure to locate `ld.lld`. The resolution involves adding the directory containing `ld.lld` to your **PATH**, either by overriding the PATH variable directly or by using `--host_action_env=PATH=xxx`.

**Note:** The **PATH** variable will not work if the `--incompatible_strict_action_env` option is enabled.

Our CI system is reporting a build error for `protoc` that states: `collect2: fatal error: cannot find 'ld' with build`. This is perplexing because we have never encountered this error on our development machines. Moreover, we cannot reproduce the error by running the command `bazel build xxx` manually on CI machine or development machine, and `ld` is indeed present in `/usr/bin`. This raises the question: why does Bazel indicate that it cannot find `ld`? There is an existing [issue](https://github.com/bazelbuild/bazel/issues/21334) on the Bazel GitHub repository, but it does not provide a solution.

Initially, I suspected that the presence of `-fuse-ld=lld` in the parameter file was the source of a Bazel bug. However, upon examining Bazel's source code more closely, I realized my assumption was incorrect. The `-fuse-ld=lld` option is derived from `link_flags` in `local_config_cc/BUILD`. The template for `local_config_cc/BUILD` can be found at [@rules_cc//cc/private/toolchain:BUILD.tpl](https://github.com/bazelbuild/rules_cc/blob/main/cc/private/toolchain/BUILD.tpl), while its instantiation occurs in [unix_cc_configure.bzl](https://github.com/bazelbuild/rules_cc/blob/main/cc/private/toolchain/unix_cc_configure.bzl#L611). The setting of `link_flags` is determined by the following code snippet:

```python
gold_or_lld_linker_path = (
    _find_linker_path(repository_ctx, cc, "lld", is_clang) or
    _find_linker_path(repository_ctx, cc, "gold", is_clang)
)
cc_path = repository_ctx.path(cc)
if not str(cc_path).startswith(str(repository_ctx.path(".")) + "/"):
    # cc is outside the repository, set -B
    bin_search_flags = ["-B" + escape_string(str(cc_path.dirname))]
else:
    # cc is inside the repository, don't set -B.
    bin_search_flags = []
if not gold_or_lld_linker_path:
    ld_path = repository_ctx.path(tool_paths["ld"])
    if ld_path.dirname != cc_path.dirname:
        bin_search_flags.append("-B" + str(ld_path.dirname))
force_linker_flags = []
if gold_or_lld_linker_path:
    force_linker_flags.append("-fuse-ld=" + gold_or_lld_linker_path)
```

The detection logic within the `_find_linker_path` function is executed by running the command `gcc xxx.cc -Wl,--start-lib -Wl,--end-lib -fuse-ld=<linker> -v`.

```python
result = repository_ctx.execute([
    cc,
    str(repository_ctx.path("tools/cpp/empty.cc")),
    "-o",
    "/dev/null",
    "-Wl,--start-lib",
    "-Wl,--end-lib",
    "-fuse-ld=" + linker,
    "-v",
])
```

In my development environment, this command executes successfully. However, when I run it on the CI machine, it fails, returning the error **collect2: fatal error: cannot find 'ld' with build**. This leads me to question why `gcc` cannot find `ld`, despite it being located in `/usr/bin`.

I tested the command both without and with `-fuse-ld=lld`, and it failed only in the latter case. This caused me to speculate that the issue lies not with `ld`, but rather with `lld` being unfindable. To further investigate, I issued the command with `-Wl,-verbose` and confirmed that using `-fuse-ld=lld` indeed invokes `ld.lld` during the linking phase.

I even experimented with creating a symbolic link for `ld.lld` on the CI machine, which resolved the issue. Additionally, adding `ld.lld` to the PATH worked as well. These tests confirmed that specifying `-fuse-ld=lld` requires `ld.lld` to be present in the PATH to avoid the `cannot find 'ld'` error.

Another pressing question is why Bazel reported an error even though I set the PATH using `--action_env=PATH=xxx`. Moreover, changing the PATH before running Bazel also yielded the same error. The `protoc` configuration does not possess any special values; you can see it [here](https://github.com/protocolbuffers/protobuf/blob/main/BUILD.bazel#L274-L280).

```
cc_binary(
    name = "protoc",
    copts = COPTS,
    linkopts = LINK_OPTS,
    visibility = ["//visibility:public"],
    deps = ["//src/google/protobuf/compiler:protoc_lib"],
)
```

From the CI error message, I noted the `[for tool]` suffix:

```
Linking external/com_google_protobuf/protoc [for tool] failed: (Exit 1): gcc failed: error executing command (from target @com_google_protobuf//:protoc) /usr/bin/gcc @bazel-out/k8-opt-exec-2B5CBBC6/bin/external/com_google_protobuf/protoc-2.params
```

The only instance where we utilize `protoc` is as follows:

```python
cc_proto_library(
    name = "echo_cc_library",
    srcs = ["echo_message.proto"],
    cc_libs = ["@com_google_protobuf//:protobuf"],
    visibility = ["//visibility:public"],
)
proto_gen = rule(
    attrs = {
        "protoc": attr.label(
            cfg = "host",
            executable = True,
            allow_single_file = True,
            mandatory = True,
        ),
    },
)
```

The `protoc` is defined as `@com_google_protobuf//:protoc`.

Note: There is a warning message indicating that `cfg = "host"` in attribute definitions should be replaced by `cfg = "exec"` (buildifier(attr-cfg)).

After reviewing the documentation regarding [attr.label](https://bazel.build/rules/lib/toplevel/attr#label)

> Configuration of the attribute. It can be either "exec", which indicates that the dependency is built for the execution platform, or "target", which indicates that the dependency is build for the target platform. A typical example of the difference is when building mobile apps, where the target platform is Android or iOS while the execution platform is Linux, macOS, or Windows. This parameter is required if executable is True to guard against accidentally building host tools in the target configuration. "target" has no semantic effect, so don't set it when executable is False unless it really helps clarify your intentions.

[Configuration](https://bazel.build/extending/rules#configurations)

> In general, sources, dependent libraries, and executables that will be needed at runtime can use the same configuration.
Tools that are executed as part of the build (such as compilers or code generators) should be built for an exec configuration. In this case, specify cfg = "exec" in the attribute.
Otherwise, executables that are used at runtime (such as as part of a test) should be built for the target configuration. In this case, specify cfg = "target" in the attribute.


[Platform Documentation](https://bazel.build/extending/platforms)

> Bazel recognizes three roles that a platform may serve:
> - **Host** - the platform on which Bazel itself runs.
> - **Execution** - a platform on which build tools execute build actions to produce intermediate and final outputs.
> - **Target** - a platform on which a final output resides and executes.
> 
> Bazel supports the following build scenarios regarding platforms:
> - **Single-platform builds** (default) - host, execution, and target platforms are the same. For example, building a Linux executable on Ubuntu running on an Intel x64 CPU.
> - **Cross-compilation builds** - host and execution platforms are the same, but the target platform is different. For example, building an iOS app on macOS running on a MacBook Pro.
> - **Multi-platform builds** - host, execution, and target platforms are all different.

I can’t recall how I stumbled upon the `--host_action_env` option; it may have simply been through searching for the term `action_env` on the Bazel official website.

The `--host_action_env` option
> Specifies the set of environment variables available to actions with execution configurations. Variables can be either specified by name, in which case the value will be taken from the invocation environment, or by the name=value pair which sets the value independent of the invocation environment. This option can be used multiple times; for options given for the same variable, the latest wins, options for different variables accumulate. 


This option is particularly relevant as `protoc` is built with execution configurations. Hence, utilizing this option with `--host_action_env` will be beneficial in resolving the environment variable issue.

I have created a demo to reproduce this issue; you may find it at [this repository](https://github.com/PikachuHyA/bazel_21334_demo) and [CI logs](https://github.com/PikachuHyA/bazel_21334_demo/actions/runs/13432086115/job/37526003234).

```
# demo.bzl
def _impl(ctx):
    ctx.actions.run(
        inputs = [],
        outputs = [ctx.outputs.output_file],
        arguments = [ctx.outputs.output_file.path],
        executable = ctx.executable._tool,
    )
    return

demo_rule = rule(
    implementation = _impl,
    attrs = {
        "_tool": attr.label(
            executable = True,
            cfg = "exec",
            allow_files = True,
            default = Label("//:demo"),
        ),
        "output_file": attr.output(mandatory = True),
    },
)
```

```
# BUILD.bazel
load(":demo.bzl", "demo_rule")

cc_binary(
    name = "demo",
    srcs = ["main.cc"],
    # linkopts = ["-Wl,-verbose"],
)

demo_rule(
    name = "tool",
    output_file = "demo_output",
)
```

## Q&A

**Q: Why is `-fuse-ld=lld` used when compiling with GCC?**  
**A:** Bazel automatically detects the available linker during the initialization of `local_config_cc`. The relevant code can be found [here](https://github.com/bazelbuild/rules_cc/blob/main/cc/private/toolchain/unix_cc_configure.bzl#L178-L190).

Bazel first checks whether `lld` is available by executing `gcc xxx.cc -Wl,--start-lib -Wl,--end-lib -fuse-ld=lld -v`. If `lld` is not found, it checks for `ld.gold` with the command `gcc xxx.cc -Wl,--start-lib -Wl,--end-lib -fuse-ld=gold -v`. You can review the source for this process [here](https://github.com/bazelbuild/rules_cc/blob/main/cc/private/toolchain/unix_cc_configure.bzl#L476-L493).

Therefore, if `ld.lld` is present in your PATH, `local_config_cc` will register that `lld` is being used.

**Q: Why does a binary linking error occur even when `lld` is specified in the PATH using `--action_env=PATH=xxx`?**  
**A:** The tool we are using, such as `protoc`, is built with the following configuration:

```python
"_tool": attr.label(
    executable = True,
    cfg = "exec",
    allow_files = True,
    mandatory = True,
),
```

The `cfg = "exec"` setting indicates that the tool is built for the execution platform. For further details, please refer to the documentation on [attr.label](https://bazel.build/rules/lib/toplevel/attr#label).

In Bazel, platforms can be categorized into three roles: Host, Execution, and Target. Generally, we build binaries or libraries for the Target platform, for instance, when executing `bazel build :demo`. However, the PATH specified with `--action_env=PATH=xxx` applies to the Target platform. If we are building for the Host platform, this PATH will not be effective and will inherit from the shell environment, defaulting to `/bin:/usr/bin:/usr/local/bin` if the `--incompatible_strict_action_env` option is enabled.
