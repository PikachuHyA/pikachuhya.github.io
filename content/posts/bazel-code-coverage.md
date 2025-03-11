+++
date = '2025-03-11T17:55:26+08:00'
draft = false
title = 'C++ Code Coverage with Bazel'
tags = ['Bazel', 'C++20 Modules']
+++

Recently, it was reported that C++20 module interface files generate no code coverage data. Upon investigating, I discovered that this is due to an instrumentation file that only accounts for explicitly instrumented files. If a file is not listed, its coverage data is skipped. For more details, refer to [CoverageOutputGenerator--Main.java#L151-L155](https://github.com/bazelbuild/bazel/blob/299e9037108ad539104fdfed39c5e41c9b3288df/tools/test/CoverageOutputGenerator/java/com/google/devtools/coverageoutputgenerator/Main.java#L151-L155).

To address this, I've created a patch to include module interfaces in the instrumentation file. Below is the fix:

<!--more-->

```diff
diff --git a/src/main/java/com/google/devtools/build/lib/analysis/test/InstrumentedFilesCollector.java b/src/main/java/com/google/devtools/build/lib/analysis/test/InstrumentedFilesCollector.java
index 8a7dac95af..208423668c 100644
--- a/src/main/java/com/google/devtools/build/lib/analysis/test/InstrumentedFilesCollector.java
+++ b/src/main/java/com/google/devtools/build/lib/analysis/test/InstrumentedFilesCollector.java
@@ -219,6 +219,16 @@ public final class InstrumentedFilesCollector {
           }
         }
       }
+      // add module_interfaces to xxx.instrumented_files
+      for (TransitiveInfoCollection dep :
+          getPrerequisitesForAttributes(ruleContext, ImmutableList.of("module_interfaces"))) {
+        for (Artifact artifact : dep.getProvider(FileProvider.class).getFilesToBuild().toList()) {
+          // skip check file extension in module_interfaces
+          if (shouldIncludeArtifact(ruleContext.getConfiguration(), artifact)) {
+            localSourcesBuilder.add(artifact);
+          }
+        }
+      }
       localSources = localSourcesBuilder.build();
     }
     instrumentedFilesInfoBuilder.addLocalSources(localSources);
diff --git a/tools/cpp/unix_cc_toolchain_config.bzl b/tools/cpp/unix_cc_toolchain_config.bzl
index 5c17196a90..8db824ad2c 100644
--- a/tools/cpp/unix_cc_toolchain_config.bzl
+++ b/tools/cpp/unix_cc_toolchain_config.bzl
@@ -1187,6 +1187,8 @@ def _impl(ctx):
                     ACTION_NAMES.cpp_compile,
                     ACTION_NAMES.cpp_header_parsing,
                     ACTION_NAMES.cpp_module_compile,
+                    ACTION_NAMES.cpp20_module_compile,
+                    ACTION_NAMES.cpp20_module_codegen,
                 ],
                 flag_groups = ([
                     flag_group(flags = ctx.attr.coverage_compile_flags),

```

## Background
To illustrate how to generate code coverage, let's consider a simple example using a `main.cc` file:

```cpp
// main.cc
#include <iostream>
int main() {
    std::cout << "Hello World\n" << std::endl;
    return 0;
}
```

### Code Coverage with GCC
GCC can generate code coverage data by adding the `--coverage` flag during both the compilation and linking phases. For example, running the commands below produces the relevant coverage files:

```bash
$ g++ main.cc --coverage -c -o main.o
$ g++ main.o --coverage -o main
$ ./main
```

The `main.gcno` file is generated during compilation, while the execution of `main` creates a `main.gcda` file.
The `main.gcno` file contains information to reconstruct the basic block graphs and assign source line numbers to blocks. The `main.gcda` file contains arc transition counts, value profile counts, and some summary information.

You can then use `lcov` to collect coverage data:

```bash
$lcov --directory . --capture --output-file app.info
Capturing coverage data from .
Found gcov version: 10.2.1
Using intermediate gcov format
Scanning . for .gcda files ...
Found 1 data files in .
Processing main.gcda
Finished .info-file creation
```

After running this, you can use `genhtml` to generate HTML coverage reports:

```bash
$genhtml app.info -o html
Reading data file app.info
Found 1 entries.
Found common filename prefix "/cpp_cov_examples"
Writing .css and .png files.
Generating output.
Processing file cli_example/main.cc
Writing directory view page.
Overall coverage rate:
  lines......: 100.0% (3 of 3 lines)
  functions..: 100.0% (1 of 1 function)
```

### Code Coverage with Clang

#### Using `-fprofile-instr-generate`
Clang can generate code coverage data by using the flags `-fprofile-instr-generate` and `-fcoverage-mapping` during the compilation phase, as well as `-fprofile-instr-generate` during the linking phase.

```bash
$ clang++ main.cc -c -o main.o -fprofile-instr-generate -fcoverage-mapping
$ clang++ main.o -o main -fprofile-instr-generate
$ ./main 
Hello World
```

After executing `./main`, a file named `default.profraw` is created. This file contains the raw profiling data, which must be processed by `llvm-profdata` before coverage reports can be generated with `llvm-cov`.

```bash
$ llvm-profdata merge -sparse default.profraw -o default.profdata
$ llvm-cov show ./main -instr-profile=default.profdata
    1|       |// main.cc
    2|       |#include <iostream>
    3|      1|int main() {
    4|      1|    std::cout << "Hello World\n" << std::endl;
    5|      1|    return 0;
    6|      1|}
```

Using `llvm-cov`, you can also export the coverage data in LCOV format by running:

```bash
$ llvm-cov export -instr-profile=default.profdata -format=lcov ./main
```

The output will look like this:

```
SF:/cpp_cov_examples/cli_example/main.cc
FN:3,main
FNDA:1,main
FNF:1
FNH:1
DA:3,1
DA:4,1
DA:5,1
DA:6,1
BRF:0
BRH:0
LF:4
LH:4
end_of_record
```

Redirect the output to `app.info`, and then use `genhtml` to generate HTML reports.

#### Using `--coverage`
Clang also supports the `--coverage` flag, which follows a workflow similar to GCC. However, you may encounter an issue related to versioning: 

```
main.gcno: version '408*', prefer 'B02A'.
```

For instance, if you are using GCC version `10.2.1`, you can reproduce this issue with the following commands:

```bash
$ clang++ main.cc -c -o main.o --coverage
$ clang++ main.o --coverage -o main
$ ./main 
Hello World
$ lcov --directory . --capture --output-file app.info
Capturing coverage data from .
Found gcov version: 10.2.1
Using intermediate gcov format
Scanning . for .gcda files ...
Found 1 data file in .
Processing main.gcda
/cpp_cov_examples/cli_example/main.gcno: version '408*', prefer 'B02A'
geninfo: ERROR: GCOV failed for /cpp_cov_examples/cli_example/main.gcda!
```

To resolve this issue, add the following flag:

```bash
-Xclang -coverage-version=B02A
```

(Note: The value of `-coverage-version` depends on the GCC version. Refer to the logs for guidance on choosing the correct version.)

Here’s how to adjust your commands:

```bash
$ clang++ main.cc -c -o main.o --coverage -Xclang -coverage-version=B02A
$ clang++ main.o --coverage -o main
$ ./main 
Hello World
$ lcov --directory . --capture --output-file app.info
Capturing coverage data from .
Found gcov version: 10.2.1
Using intermediate gcov format
Scanning . for .gcda files ...
Found 1 data file in .
Processing main.gcda
geninfo: WARNING: could not open /main.cc
geninfo: WARNING: some exclusion markers may be ignored
Finished .info-file creation
```

The warning `geninfo: WARNING: could not open /main.cc` arises because Clang does not record the current working directory in the `gcno` file. For more details, see [LLVM Issue #121368](https://github.com/llvm/llvm-project/issues/121368). I attempted to resolve this by adding the current working directory, but the patch was not accepted as it would compromise local determinism.

As a workaround, change the build path and use an absolute path when compiling:

```bash
$ mkdir build && cd build
$ clang++ /absolute/path/to/main.cc -c -o main.o --coverage -Xclang -coverage-version=B02A
$ clang++ main.o --coverage -o main
$ ./main 
Hello World
$ lcov --directory . --capture --output-file app.info
Capturing coverage data from .
Found gcov version: 10.2.1
Using intermediate gcov format
Scanning . for .gcda files ...
Found 1 data file in .
Processing main.gcda
Finished .info-file creation
```

The final step to generate HTML files is identical to the process used with GCC. Simply execute:

```bash
$ genhtml app.info -o html
```


### Bazel Code Coverage
#### A Simple C++ Case
The Bazel documentation on [Code Coverage](https://bazel.build/configure/coverage?hl=en) provides a comprehensive overview. Here, we'll focus on C++ code coverage. 

Consider a minimal project structure:

```bash
$ tree
.
├── BUILD.bazel
├── foo.cc
├── main.cc
├── MODULE.bazel
├── MODULE.bazel.lock
└── README.md
```

The `BUILD.bazel` file might look like this:

```python
# BUILD.bazel
cc_library(
    name = "foo",
    srcs = ["foo.cc"],
)

cc_test(
    name = "test",
    srcs = ["main.cc"],
    deps = [":foo"],
)
```

And the source code could be:

```cpp
// main.cc
void foo();
int main() {
  foo();
  return 0;
}
// foo.cc
#include <iostream>
void foo() {
  std::cout << "hello" << std::endl;
  std::cout << "world" << std::endl;
}
```

#### GCC with Bazel

To generate a coverage report using Bazel, run the following command:

```bash
$bazel coverage ... --combined_report=lcov --action_env=CC=/usr/bin/gcc
```

When you execute this command, you will see log messages similar to the following:

```
INFO: Using default value for --instrumentation_filter: "^//".
INFO: Override the above default with --instrumentation_filter
INFO: Analyzed 2 targets (96 packages loaded, 1452 targets configured).
INFO: LCOV coverage report is located at /disk8/xxx/bazel_cache/_bazel_xxx/2986c21b4a1e0acd2b79986480a4fbb3/execroot/_main/bazel-out/_coverage/_coverage_report.dat
 and execpath is bazel-out/_coverage/_coverage_report.dat
INFO: From Coverage report generation:
Mar 11, 2025 3:53:53 PM com.google.devtools.coverageoutputgenerator.Main getTracefiles
INFO: Found 1 tracefiles.
Mar 11, 2025 3:53:53 PM com.google.devtools.coverageoutputgenerator.Main parseFilesSequentially
INFO: Parsing file bazel-out/k8-fastbuild/testlogs/test/coverage.dat
Mar 11, 2025 3:53:53 PM com.google.devtools.coverageoutputgenerator.Main getGcovInfoFiles
INFO: No gcov info file found.
Mar 11, 2025 3:53:53 PM com.google.devtools.coverageoutputgenerator.Main getGcovJsonInfoFiles
INFO: No gcov json file found.
Mar 11, 2025 3:53:53 PM com.google.devtools.coverageoutputgenerator.Main getProfdataFileOrNull
INFO: No .profdata file found.
INFO: Found 1 target and 1 test target...
INFO: Elapsed time: 7.414s, Critical Path: 3.80s
INFO: 26 processes: 15 internal, 10 linux-sandbox, 1 worker.
INFO: Build completed successfully, 26 total actions
//:test                                                                  PASSED in 0.5s
  /disk8/xxx/bazel_cache/_bazel_xxx/2986c21b4a1e0acd2b79986480a4fbb3/execroot/_main/bazel-out/k8-fastbuild/testlogs/test/coverage.dat

Executed 1 out of 1 test: 1 test passes.
There were tests whose specified size is too big. Use the --test_verbose_timeout_warnings command line option to see which ones these are.
```

highlight log

- **Instrumentation Filter**: 
  ```
  INFO: Using default value for --instrumentation_filter: "^//".
  ```

- **Coverage Report Location**: 
  ```
  INFO: LCOV coverage report is located at /disk8/xxx/bazel_cache/_bazel_xxx/2986c21b4a1e0acd2b79986480a4fbb3/execroot/_main/bazel-out/_coverage/_coverage_report.dat
  and execpath is bazel-out/_coverage/_coverage_report.dat
  ```

The rule names that match the regex `^//` will be instrumented, and the resulting coverage data will be saved in `bazel-out/_coverage/_coverage_report.dat`.

Next, you can generate HTML reports using the following command:

```bash
$ genhtml -o html bazel-out/_coverage/_coverage_report.dat
Reading data file bazel-out/_coverage/_coverage_report.dat
Resolved relative source file path "foo.cc" with CWD to "/cpp_cov_examples/bazel_example/foo.cc".
Found 1 entries.
Found common filename prefix "/cpp_cov_examples"
Writing .css and .png files.
Generating output.
Processing file bazel_example/foo.cc
Writing directory view page.
Overall coverage rate:
  lines......: 100.0% (4 of 4 lines)
  functions..: 100.0% (1 of 1 function)
```
The test log can be found in `bazel-testlogs/[path/to/target]/[target_name]/test.log`:

```bash
exec ${PAGER:-/usr/bin/less} "$0" || exit 1
Executing tests from //:test
-----------------------------------------------------------------------------
hello
world
File 'foo.cc'
Lines executed: 100.00% of 4
No branches
Calls executed: 100.00% of 4
File '/usr/include/c++/10/iostream'
No executable lines
No branches
No calls
File 'main.cc'
Lines executed: 100.00% of 3
No branches
Calls executed: 100.00% of 1
Mar 11, 2025 8:11:50 AM com.google.devtools.coverageoutputgenerator.Main getTracefiles
INFO: No lcov file found.
Mar 11, 2025 8:11:50 AM com.google.devtools.coverageoutputgenerator.Main getGcovInfoFiles
INFO: No gcov info file found.
Mar 11, 2025 8:11:50 AM com.google.devtools.coverageoutputgenerator.Main getGcovJsonInfoFiles
INFO: Found 2 gcov json files.
Mar 11, 2025 8:11:50 AM com.google.devtools.coverageoutputgenerator.Main parseFilesSequentially
INFO: Parsing file /disk8/xxx/bazel_cache/_bazel_xxx/2986c21b4a1e0acd2b79986480a4fbb3/sandbox/linux-sandbox/72/execroot/_main/bazel-out/k8-fastbuild/testlogs/_coverage/test/test/bazel-out/k8-fastbuild/bin/_objs/foo/foo.pic.gcda.gcov.json.gz
Mar 11, 2025 8:11:50 AM com.google.devtools.coverageoutputgenerator.Main parseFilesSequentially
INFO: Parsing file /disk8/xxx/bazel_cache/_bazel_xxx/2986c21b4a1e0acd2b79986480a4fbb3/sandbox/linux-sandbox/72/execroot/_main/bazel-out/k8-fastbuild/testlogs/_coverage/test/test/bazel-out/k8-fastbuild/bin/_objs/test/main.pic.gcda.gcov.json.gz
Mar 11, 2025 8:11:51 AM com.google.devtools.coverageoutputgenerator.Main getProfdataFileOrNull
INFO: No .profdata file found.
```

highlight log
```bash
Parsing file ... bazel-out/k8-fastbuild/bin/_objs/foo/foo.pic.gcda.gcov.json.gz
Parsing file ... bazel-out/k8-fastbuild/bin/_objs/test/main.pic.gcda.gcov.json.gz
```

If you encounter any issues, check the log file and run the command with the following option to enable verbose logging:

```bash
--action_env=VERBOSE_COVERAGE=1
```


#### Clang with Bazel

To use Bazel with Clang for code coverage, execute the following command:

```bash
$ bazel coverage ... --combined_report=lcov --action_env=CC=/usr/bin/clang --repo_env=BAZEL_USE_LLVM_NATIVE_COVERAGE=1 --repo_env=GCOV=llvm-profdata --experimental_use_llvm_covmap --experimental_generate_llvm_lcov
```

explanation of flags

- **--repo_env=BAZEL_USE_LLVM_NATIVE_COVERAGE=1**:  
  This flag adds `-fprofile-instr-generate` and `-fcoverage-mapping` during the compilation phase, and `-fprofile-instr-generate` during the linking phase.

- **--experimental_use_llvm_covmap**:  
  This option indicates that `llvm-cov` should be used. However, it currently has no effect, as the [collect_cc_coverage.sh](https://github.com/bazelbuild/bazel/blob/c422744caa072c66311a937049504901bc674b7d/tools/test/collect_cc_coverage.sh#L93) script already hardcodes `${LLVM_COV}`.

- **--experimental_generate_llvm_lcov**:  
  This flag directs Bazel to generate coverage data in LCOV format.

- **--repo_env=GCOV=llvm-profdata**:  
  This flag seems a bit peculiar. After reviewing the code, I submitted a [PR](https://github.com/bazelbuild/bazel/pull/25516) to propose its removal.

For differences in processes between Clang and GCC, refer to the `test.log` file:

```bash
exec ${PAGER:-/usr/bin/less} "$0" || exit 1
Executing tests from //:test
-----------------------------------------------------------------------------
hello
world
Mar 11, 2025 8:16:29 AM com.google.devtools.coverageoutputgenerator.Main getTracefiles
INFO: Found 1 tracefile.
Mar 11, 2025 8:16:29 AM com.google.devtools.coverageoutputgenerator.Main parseFilesSequentially
INFO: Parsing file /disk8/xxx/bazel_cache/_bazel_xxx/2986c21b4a1e0acd2b79986480a4fbb3/sandbox/linux-sandbox/82/execroot/_main/bazel-out/k8-fastbuild/testlogs/_coverage/test/test/_cc_coverage.dat
Mar 11, 2025 8:16:29 AM com.google.devtools.coverageoutputgenerator.Main getGcovInfoFiles
INFO: No gcov info file found.
Mar 11, 2025 8:16:29 AM com.google.devtools.coverageoutputgenerator.Main getGcovJsonInfoFiles
INFO: No gcov json file found.
Mar 11, 2025 8:16:29 AM com.google.devtools.coverageoutputgenerator.Main getProfdataFileOrNull
INFO: No .profdata file found.
```

highlight log

```bash
Parsing file bazel-out/k8-fastbuild/testlogs/_coverage/test/test/_cc_coverage.dat
```
In GCC, the processed files take the form of xxx.gcda.gcov.json.gz.

### Under the Hood

When you invoke `bazel coverage xxx`, it enables the `--collect_code_coverage` flag, which allows Bazel to gather coverage data after running tests. Ultimately, this coverage data is merged into `bazel-out/_coverage/_coverage_report.dat`.

The coverage command is managed by [CoverageCommand.java](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/runtime/commands/CoverageCommand.java). 

Typically, tests are executed via the script `bazel_tools/tools/test/test-setup.sh path/to/test`. However, when using `bazel coverage`, the command runs the test through `bazel_tools/tools/test/test-setup.sh bazel_tools/tools/test/collect_coverage.sh path/to/test`.

Here’s the relevant portion of the `test-setup.sh` script:

```bash
set -m
if [[ "${EXPERIMENTAL_SPLIT_XML_GENERATION}" == "1" ]]; then
  if [ -z "$COVERAGE_DIR" ]; then
    ("${TEST_PATH}" "$@" 2>&1) <&0 &
  else
    ("$1" "$TEST_PATH" "${@:3}" 2>&1) <&0 &
  fi
else
  set -o pipefail
  if [ -z "$COVERAGE_DIR" ]; then
    ("${TEST_PATH}" "$@" 2>&1 | tee -a "${XML_OUTPUT_FILE}.log") <&0 &
  else
    ("$1" "$TEST_PATH" "${@:3}" 2>&1 | tee -a "${XML_OUTPUT_FILE}.log") <&0 &
  fi
  set +o pipefail
fi
childPid=$!
```

The `collect_coverage.sh` script runs the actual tests and collects C++ code coverage data using `collect_cc_coverage.sh`:

```bash
...
  # Execute the test.
  "$@"
  TEST_STATUS=$?
...

# Call the C++ code coverage collection script.
if [[ "$CC_CODE_COVERAGE_SCRIPT" ]]; then
    eval "${CC_CODE_COVERAGE_SCRIPT}"
fi
```

The `collect_cc_coverage.sh` script dispatches the appropriate function to gather data based on the value of `$BAZEL_CC_COVERAGE_TOOL`:

```bash
  case "$BAZEL_CC_COVERAGE_TOOL" in
        ("GCOV") gcov_coverage "$COVERAGE_DIR/_cc_coverage.gcov" ;;
        ("PROFDATA") llvm_coverage_profdata "$COVERAGE_DIR/_cc_coverage.profdata" ;;
        ("LLVM_LCOV") llvm_coverage_lcov "$COVERAGE_DIR/_cc_coverage.dat" ;;
        (*) echo "Coverage tool $BAZEL_CC_COVERAGE_TOOL not supported" \
            && exit 1
  esac
```

The `gcov_coverage` function runs `gcov -i -o <dir> <gcda-file>`, generating a `xxx.gcda.gcov` file. Alternatively, it can run `gcov -i -b -o <dir> <gcda-file>`, which produces a `xxx.gcda.gcov.json.gz` file.

Here is a simplified version of the `gcov_coverage` function:

```bash
while read -r line; do
  if [[ $gcov_major_version -le 7 ]]; then
    "${GCOV}" -i $COVERAGE_GCOV_OPTIONS -o "$(dirname ${gcda})" "${gcda}"
  else
    "${GCOV}" -i -b $COVERAGE_GCOV_OPTIONS -o "$(dirname ${gcda})" "${gcda}"
  fi
done < "${COVERAGE_MANIFEST}"
```

The variable `$COVERAGE_GCOV_OPTIONS` remains empty unless specified. The `${COVERAGE_MANIFEST}` refers to the `<target-name>.instrumented_files`.

example of a `xxx.gcda.gcov` file

```bash
version: 8.3.0
file: main.cc
function: 3,6,1,main
lcount: 3,1,0
lcount: 4,1,0
lcount: 5,1,0
version: 8.3.0
file: /usr/include/c++/8.3.0/iostream
```

example of a `xxx.gcda.gcov.json` file (inside `xxx.gcda.gcov.json.gz`)

```json
{
  "gcc_version": "10.2.1",
  "files": [
    {
      "lines": [
        {
          "branches": [],
          "count": 1,
          "line_number": 3,
          "unexecuted_block": false,
          "function_name": "main"
        },
        {
          "branches": [],
          "count": 1,
          "line_number": 4,
          "unexecuted_block": false,
          "function_name": "main"
        },
        {
          "branches": [],
          "count": 1,
          "line_number": 5,
          "unexecuted_block": false,
          "function_name": "main"
        }
      ],
      "functions": [
        {
          "blocks": 4,
          "end_column": 1,
          "start_line": 3,
          "name": "main",
          "blocks_executed": 4,
          "execution_count": 1,
          "demangled_name": "main",
          "start_column": 5,
          "end_line": 6
        }
      ],
      "file": "main.cc"
    },
    {
      "lines": [],
      "functions": [],
      "file": "/usr/include/c++/10/iostream"
    }
  ],
  "format_version": "1",
  "current_working_directory": "/cpp_cov_examples/cli_example",
  "data_file": "main.gcda"
}
```

The `llvm_coverage_lcov` function calls `llvm-profdata` to merge the raw profiles and then uses `llvm-cov` to generate coverage data in LCOV format. When running `llvm-cov`, the binary file is required. This information is stored in the `xxxruntime_objects_list.txt` file.

The final step involves merging all coverage data. The merger, implemented in Java, can be found in `tools/test/CoverageOutputGenerator`.

The merger is invoked at the end of the `collect_coverage.sh` script:

```bash
...

LCOV_MERGER_CMD="${LCOV_MERGER} --coverage_dir=${COVERAGE_DIR} \
  --output_file=${COVERAGE_OUTPUT_FILE} \
  --filter_sources=/usr/bin/.+ \
  --filter_sources=/usr/lib/.+ \
  --filter_sources=/usr/include.+ \
  --filter_sources=/Applications/.+ \
  --source_file_manifest=${COVERAGE_MANIFEST}"
JAVA_RUNFILES= exec $LCOV_MERGER_CMD
```

The merger scans all the coverage data files (`.dat`, `.gcov`, `.gcov.json.gz`, `.profdata`) and filters the data according to the specified `--filter_sources` and `--source_file_manifest`. Only files included in the `--source_file_manifest` will be retained, while those listed in `--filter_sources` will be excluded.

### References
- [gcov—a Test Coverage Program](https://gcc.gnu.org/onlinedocs/gcc/Gcov.html)
- [Clang Source-based Code Coverage](https://clang.llvm.org/docs/SourceBasedCodeCoverage.html)
- [Code Coverage with Bazel](https://bazel.build/configure/coverage?hl=en)
- [YouTube: Coverage with Bazel (Ulf Adams @ EngFlow) - Oct 2023](https://www.youtube.com/watch?v=f0rq27W7Ppc)
- [Bazel Issue: Support coverage](https://github.com/bazelbuild/bazel/issues/1118)
- [Bazel commit: Coverage support.](https://github.com/bazelbuild/bazel/commit/8829abaeec8fa0be7ea6d87cbfde656e9c780cf3)
- [Bazel Coverage design document (WIP)](https://docs.google.com/document/d/1-ZWHF-Q-qCKf19ik-t33ie58BkNurrYYzKR4OLtcilY/edit)
