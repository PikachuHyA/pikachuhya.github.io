+++
date = '2026-02-26T09:48:11+08:00'
draft = false
title = 'Summary of 2025'
tags = ['Summary']
toc = true
+++

## Overview

This is a late summary of 2025—the calendar has turned to February 2026.

In the past year, almost all of my time was devoted to the Bazel build system: a unified build system based on Bazel for projects in all languages. I had originally planned to spend time on Triton, but ended up with none. The LLVM JIT effort was suspended due to business changes, and no effort went into ClangIR.

## Bazel Build System

The most significant progress was that [the C++20 Modules patch](https://github.com/bazelbuild/bazel/pull/19940) has been merged into the Bazel master branch. Thanks to [@fmeum](https://github.com/fmeum) for his work. There is still plenty to do before the feature is complete. Here are some [examples of using C++20 Modules with Bazel](https://github.com/PikachuHyA/bazel_cxx20_modules_demo).

## Using Bazel to Build Everything

When we decided to use Bazel to build everything, we did not use native Bazel rules (e.g. rules_java) due to the high cost of migrating all projects. Instead, we wrote new rules that wrap existing build tools—Maven, Gradle, Cargo, etc.—to build each project type.

Building projects with Bazel is only the first step. We have two categorical requirements.

The first is **offline building**: we must build all projects without access to external sites like GitHub. All resources must be backed up to our internal resource center, and Bazel fetches them from there before building. For example, for a Maven project we declare all dependency JARs in our custom rule in `BUILD.bazel`, then sync those JARs to a specified internal backup location. Offline building introduces many problems, though most can be solved with workarounds.

The second is **source-building second-party dependencies**: we must build 2nd-party deps from source rather than using prebuilts. There are three main difficulties:

1. **Dependency scale**: The number of 2nd-party dependencies can be 10× or more than that of 1st-party projects. To source-build one, we often must build more 2nd-party deps, since each may depend on another.
2. **Validation**: We lack a reliable way to validate that a source-built 2nd-party dependency is actually usable.
3. **Lost sources**: Some 2nd-party projects no longer have source code available—e.g., decade-old projects whose developers have all left the company.

Beyond these two requirements, we face environment isolation. When a project is sensitive to the environment (e.g., OS, glibc version, GCC version), we run its build inside a Docker container. For example, C++ projects built with different GCC. That means many containers and thus a lot of work to build Docker images from source.

Reproducibility is another core requirement for Bazel: it ensures correct build results and enables efficient cache usage. Maven can achieve it with 3.9.8 and the corresponding plugin versions. Docker images, however, cannot: when `yum install` runs in a Dockerfile, yum writes metadata to its database on disk, and that file is not reproducible.

## Conclusion

I am not sure whether building everything with Bazel is the best solution, but this may be the first serious attempt to use Bazel to connect all projects in this way. Bazel’s dependency analysis is slow when many projects need to be built, which is a direction for improvement. Overall, I spent much of my year on the build system somewhat by accident.

I sometimes wonder whether it makes sense for a compiler engineer to invest this much in build systems instead of the compiler itself. That said, C++20 Modules sit at the boundary between build systems and the compiler, so I will continue to work on that feature—it feels like the right place to be.
