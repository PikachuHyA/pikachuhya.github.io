+++
date = '2025-01-02T17:55:58+08:00'
draft = false
title = 'Summary of 2024'
tags = ['Summary']
+++

## Overview

In 2024, I continued to explore various areas within the compiler field. My focus this year included the Bazel Build System, LLVM JIT integration with our internal database, Sanitizers, and ClangIR. Below is a detailed summary of my work and achievements in each of these areas.

<!--more-->

## Bazel Build System

Since April 2023, I have been working on [supporting C++20 Modules](https://github.com/bazelbuild/bazel/issues/4005) in Bazel. On October 25, 2023, I [submitted a patch](https://github.com/bazelbuild/bazel/pull/19940) to the Bazel repository. In 2024, five additional small patches [were merged](https://github.com/bazelbuild/bazel/pulls?q=is%3Apr+author%3APikachuHyA+is%3Aclosed+created%3A%3E2024-01-01+created%3A%3C2025-01-01) into the Bazel repository. If [Bazel PR#22553](https://github.com/bazelbuild/bazel/pull/22553) is merged, we will be able to use C++20 Modules with Bazel in a fundamental way. However, there are still some issues related to C++20 Modules support in Bazel. I plan to continue improving the implementation, provided Google accepts my patch.

## LLVM JIT

I attempted to integrate LLVM JIT into our internal database product but was unsuccessful due to limited effort and knowledge. Despite dedicating several months to this project, my understanding of LLVM JIT remains constrained. From my perspective, applying LLVM JIT to our internal database was a challenging experience, and I did not achieve the desired results.

## Sanitizers

Although I initially had no clear idea why I needed to address issues related to Sanitizers, in practice, I spent considerable time troubleshooting problems such as missing stack traces, absent symbols, and how to suppress erroneous errors or warnings—particularly those misreported by UndefinedBehaviorSanitizer. Handling these issues was time-consuming but necessary for maintaining code quality.

## ClangIR

Working on ClangIR has been the most enjoyable and surprising aspect of my 2024. I submitted [my first patch for ClangIR](https://github.com/llvm/clangir/pull/1011) on October 28, and it was merged on October 31. I am grateful to [@Lancern](https://github.com/Lancern) and [@bcardosolopes](https://github.com/bcardosolopes) for reviewing my code and providing valuable suggestions to improve its quality. Throughout 2024, [14 of my patches](https://github.com/llvm/clangir/pulls?q=is%3Apr+author%3APikachuHyA+is%3Aclosed+created%3A%3E2024-01-01+created%3A%3C2025-01-01+) were merged into the ClangIR repository. Although patches related to [TBAA Support](https://github.com/llvm/clangir/pull/1076) were reverted, I am committed to refactoring my patches to meet ClangIR's requirements. My work plan for 2025 includes focusing on TBAA Support, Debugging Info Support, and lowering through MLIR. Notably, I contributed [my first patch to the LLVM repository](https://github.com/llvm/llvm-project/pull/115711), which I celebrated by [writing a blog post](https://pikachuhya.github.io/posts/commemorating-the-first-llvm-patch/).

## Conclusion

I chose to write this blog in English despite not being a native speaker, relying on ChatGPT to help polish my draft. As a result, some sentences may still seem unusual or unclear, reflecting my ongoing learning process. Overall, I am excited to continue contributing more code to the ClangIR repository and plan to dedicate some time to working on Triton in the coming year.
