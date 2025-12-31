+++
date = '2025-12-31T10:14:45+08:00'
draft = false
title = 'Bazel cc_shared_library Explained'
tags = ['Bazel']
+++


`cc_shared_library` is a Bazel rule for building C++ shared libraries (.so/.dll). Unlike the simple `cc_binary(linkshared=1)`, it provides fine-grained dependency management, intelligently deciding which libraries should be statically linked into the shared library and which should be obtained through dynamic dependencies.

This blog provides an in-depth analysis of the `cc_shared_library` implementation to help you understand its internal workings.

<!--more-->
## Running Example

We'll use this example throughout the blog:

BUILD file (simplified):

```python
cc_library(
    name = "A",
)

cc_library(
    name = "B",
    deps = [":A"],
)

cc_library(
    name = "D",
)

cc_library(
    name = "C",
    deps = [":B", ":D"],
)

cc_library(
    name = "E",
)

cc_library(
    name = "X",
)

cc_shared_library(
    name = "libB.so",
    deps = [":B"],
)

cc_shared_library(
    name = "libX.so",
    deps = [":X"],
)

cc_shared_library(
    name = "libC.so",
    deps = [":C", ":E"],
    dynamic_deps = [":libB.so", ":libX.so"],
)
```

Dependency Graph:

```
         ┌───────┐       ┌───────┐       ┌───────┐       ┌───────┐
         │   A   │       │   D   │       │   E   │       │   X   │
         └───────┘       └───────┘       └───────┘       └───────┘
             ▲               ▲               ▲               ▲
             │               │               │               │
         ┌───┴───┐       ┌───┴───┐           │               │
         │   B   │◄──────│   C   │           │               │
         └───────┘       └───────┘           │               │
             ▲               ▲               │               │
             │               │               │               │
             │               │               │               │
             │               │               │               │
             │  ┌────────────┴───────────────┴───────┐       │
             │  │            libC.so                 │       │
             │  │  ────────────────────────────────  │       │
             │  │  deps: [C, E]     static: C, D, E  │       │
             │  │  exports: [C, E]  dynamic: B       │       │
             │  │  unused: libX.so!                  │       │
             │  └───────┬───────────────────┬────────┘       │
             │          │                   │                │
             │          │ dynamic_deps      │                │
             │          │                   │                │
             │          ▼                   ▼                │
┌─────────────────────────────┐          ┌─────────────────────────────┐
│          libB.so            │          │          libX.so            │
│  ─────────────────────────  │          │  ─────────────────────────  │
│  deps: [B]   static: B, A   │          │  deps: [X]   static: X      │
│  exports: [B]               │          │  exports: [X]               │
└─────────────────────────────┘          └─────────────────────────────┘
```

Result:
- `libB.so` statically links B and A, exports [B]
- `libX.so` statically links X, exports [X]
- `libC.so` statically links C, D, and E; gets B (and A) from libB.so; exports [C, E]

Why this example is interesting:
- Multiple deps: C and E (both top-level)
- Multiple dynamic_deps: libB.so and libX.so
- B is exported by libB.so → pruned (dynamic link)
- D and E are not exported → must be statically linked
- libX.so exports X, but no one uses X → demonstrates unused dynamic_deps handling
- Shows pruning, non-pruning, and two-phase processing in one example


## Architecture & Implementation Flow

### Rule Attributes

Key attributes of `cc_shared_library`:

- deps: Top-level libraries to statically link
- dynamic_deps: Other `cc_shared_library` dependencies
- exports_filter: Declare exported transitive dependencies

### Processing Pipeline

1. graph_structure_aspect — Build dependency graph (GraphNodeInfo)
2. _get_deps — Get dependencies from `deps` attribute
3. find_cc_toolchain — Configure toolchain and features
4. _merge_cc_shared_library_infos — Collect info from all `dynamic_deps`
5. Build export → linker_input mapping
6. _build_link_once_static_libs_map — Track libraries that can only link once
7. _filter_inputs — Collect linker inputs, handle top-level
8. cc_common.link — Generate the .so file
9. Return providers — DefaultInfo, CcSharedLibraryInfo, OutputGroupInfo

## Core Data Structures
 
### GraphNodeInfo Provider

```python
GraphNodeInfo = provider(
    fields = {
        "children": "Child nodes in the dependency graph",
        "owners": "Owner labels of linker_inputs for this node",
        "linkable_more_than_once": "Whether it can be linked by multiple shared libraries",
    },
)
```

This is collected by `graph_structure_aspect` and used for linking decisions.

### CcSharedLibraryInfo Provider

The provider returned by the rule, containing:
- `exports`: List of exported libraries
- `link_once_static_libs`: Static libraries that can only be linked once
- `linker_input`: The linker input for this shared library
- `dynamic_deps`: Transitive dynamic dependencies

## graph_structure_aspect: Building the Dependency Graph

`graph_structure_aspect` is the foundation. It traverses dependency edges and creates a `GraphNodeInfo` for each target.


What aspect produces (using Running Example):

- A → children: `[]`
- B → children: `[A]`
- C → children: `[B, D]`
- D → children: `[]`
- E → children: `[]`
- X → children: `[]`

Key point: Aspect only traverses `deps` of cc_library targets. It does NOT traverse `dynamic_deps` of cc_shared_library — those are handled separately via `CcSharedLibraryInfo`.

Implementation (simplified):

```python
graph_structure_aspect = aspect(
    attr_aspects = ["*"],    # Propagate along ALL attributes (not just deps)
    implementation = _graph_structure_aspect_impl,
)

def _graph_structure_aspect_impl(target, ctx):
    children = []
    for attr_name in dir(ctx.rule.attr):           # Get all attribute names
        deps = getattr(ctx.rule.attr, attr_name)
        # Collect GraphNodeInfo from any attribute containing targets
        if has_graph_node_info(deps):
            children.append(deps[GraphNodeInfo])
    
    return [GraphNodeInfo(
        owners = [ctx.label],
        children = children,
        linkable_more_than_once = "LINKABLE_MORE_THAN_ONCE" in ctx.rule.attr.tags,
    )]
```

Key points:
- `attr_aspects = ["*"]` ensures aspect propagates along **all** attributes, not just `deps`
- `dir(ctx.rule.attr)` returns all attribute names, allowing traversal of custom attributes

## Export Mechanism: exports and exports_filter

Before diving into the linking algorithm, we need to understand what "exports" means — this is key to understanding link decisions.

### What Are Exports?

Export declarations tell other `cc_shared_library` targets: "I've already included these symbols, you don't need to link them again."

Using our running example:

```
libB.so (exports: [B])
    │
    └── B

libC.so (dynamic_deps: [libB.so])
    │
    └── C ──► B   ← B is exported by libB.so, so libC.so won't static link B
```

### Two Ways to Export

- `deps` (direct dependencies) — Automatically considered exported
- `exports_filter` (transitive dependencies) — Manually declare which to export

### exports_filter Syntax

```python
exports_filter = [
    "//pkg:target",           # Exact match
    "//pkg:__pkg__",          # Match all targets in package
    "//pkg:__subpackages__",  # Match all targets in package and subpackages
]
```

### Important Note

`exports_filter` only declares intent—it doesn't add dependencies. Actual symbol visibility is controlled by the linker (e.g., via version scripts):

```python
cc_shared_library(
    name = "libB.so",
    deps = [":B"],
    exports_filter = ["//pkg:A"],  # Also export A (transitive dep)
    additional_linker_inputs = [":libB.lds"],
    user_link_flags = ["-Wl,--version-script=$(location :libB.lds)"],
)
```

## Core Algorithm: Static/Dynamic Link Separation

### The Concept

This is the heart of `cc_shared_library`. Think of it as packing for a trip:

> Your suitcase is the current `cc_shared_library`. Your friend's suitcase is a shared library in `dynamic_deps`.
> 
> The rule is simple:
> - If your friend already packed something (in dynamic_deps exports) → don't pack it yourself (dynamic link)
> - If your friend didn't pack it → you must pack it yourself (static link)

### Implementation

Algorithm: Depth-first traversal with pruning

```
For each node in dependency graph:
  1. Check: Is this library exported by a dynamic_dep?
     - YES → Mark as dynamic, DON'T traverse children (prune)
     - NO  → Mark as static, traverse children
  2. Build depset for topological ordering
```

Key data structures:

- can_be_linked_dynamically (Set) — All exports from all dynamic_deps
- targets_to_be_linked_statically_map (Dict) — Libraries to package into this .so
- targets_to_be_linked_dynamically_set (Dict) — Libraries to get from other .so

The core loop (simplified):

```python
for node in graph_traversal:
    if node.owner in can_be_linked_dynamically:
        # Dynamic: use from another .so
        mark_dynamic(node)
        # DON'T traverse children — the other .so handles them
    else:
        # Static: package into this .so
        mark_static(node)
        traverse_children(node)  # Keep going deeper
```

### Building Topological Order

#### Why is topological order needed?

The linker requires dependencies to appear **before** their dependents. If B depends on A, the link command must be:
```
... -lA -lB ...    ✓ correct (A before B)
... -lB -lA ...    ✗ wrong (undefined symbols)
```

Bazel's `depset(order="topological")` automatically handles this — children appear before parents when the depset is flattened.

#### When is a node "finalized"?

A node is finalized (its depset created) when:

- Leaf node (no children) → Create depset with no transitive
- Pruned node (dynamically linked) → Create depset, don't traverse children
- Parent node (all children processed) → Create depset with children's depsets as transitive

The code:

```python
first_owner_to_depset[node.owners[0]] = depset(
    direct = node.owners,           # This node's labels
    transitive = children_depsets,  # Children's depsets (already in order)
    order = "topological"           # Ensure correct flattening order
)
```

Visualization (building libC.so):

```
Building libC.so:
  deps: [C, E]
  dynamic_deps: [libB.so, libX.so] → exports: [B, X]
  can_be_linked_dynamically: {B, X}

Stack: [C, E]  ← E is at top (rightmost), processed first

1. Pop E → not in exports → static link → no children (leaf)
   → Create depset[E] = depset([E])
   Stack: [C]

2. Pop C → not in exports → static link → push children [B, D]
   Stack: [C, D, B]  ← B at top, processed first

3. Pop B → IN exports (from libB.so)! → dynamic link
   → DON'T push B's children (pruning!)
   → Create depset[B] = depset([B])
   Stack: [C, D]

4. Pop D → not in exports → static link → no children (leaf)
   → Create depset[D] = depset([D])
   Stack: [C]

5. Pop C (again) → all children processed
   → Create depset[C] = depset([C], transitive=[depset[B], depset[D]])
   Stack: []

Final topological order: [E, B, D, C]
  - E: static (packaged into libC.so)
  - B: dynamic (linker_input from libB.so, not packaged)
  - D: static (packaged into libC.so)
  - C: static (packaged into libC.so)

Note: X is never encountered — no one depends on it.
      libX.so will be linked via _add_unused_dynamic_deps.
```

#### Why include B in C's transitive depset?

Even though B is dynamically linked, it still needs a position in the link order. The depset tracks **all** dependencies for correct ordering — the static vs dynamic distinction determines **how** each is linked, not **whether** it appears in the order.

### Key Point: Pruning

When a library is dynamically linked (exported by another .so), **its children are not traversed**:

Without pruning:
- libC.so would traverse: C → B → A → ...
- Risk: A packaged in both libB.so and libC.so

With pruning:
- libC.so traverses: C → B (stop!)
- A only in libB.so, libC.so gets it at runtime

In the visualization above, see step 3 where B is found in libB.so's exports — we create its depset but don't push B's children (A is never visited).

## _filter_inputs: Processing Linker Inputs

`_filter_inputs` iterates over all linker inputs and classifies them based on the decisions made in the previous phase.

### Collecting and Classifying Linker Inputs

```python
for linker_input in dependency_linker_inputs:
    owner = str(linker_input.owner)
    
    if owner in targets_to_be_linked_dynamically_set:
        # Dynamic: use linker_input from dynamic_dep
        _add_linker_input_to_dict(linker_input.owner, transitive_exports[owner])
        
    elif owner in targets_to_be_linked_statically_map:
        # Static: process static libraries and precompiled dynamic libraries
        # ...
        
        # Track libraries that can only be linked once
        if not targets_to_be_linked_statically_map[owner]:
            curr_link_once_static_libs_set[owner] = True
```

Understanding `curr_link_once_static_libs_set`:

Recall that targets_to_be_linked_statically_map stores:
```python
targets_to_be_linked_statically_map[owner] = node.linkable_more_than_once
```

The value is `True` if the library has `LINKABLE_MORE_THAN_ONCE` tag, `False` otherwise.

So this check:
```python
if not targets_to_be_linked_statically_map[owner]:  # If False (no tag)
    curr_link_once_static_libs_set[owner] = True     # Mark as "link once only"
```

Collects all statically linked libraries that **cannot** be linked multiple times. This set is:
1. Returned in CcSharedLibraryInfo.link_once_static_libs
2. Used by _build_link_once_static_libs_map to detect duplicate static linking conflicts

### Finding Top-Level Entry Points

Top-level libraries (first libraries with actual code in `deps`) are linked with `alwayslink` (whole-archive).

#### Why find "top-level"?

Example 1: Normal case (using Running Example)

```
libC.so deps: [C, E]
               │
               └── C(has code) ← top-level!
```
Result: top_level = {C, E}
Both have code, so they are both top-level.


Example 2: Wrapper pattern

```
deps: [wrapper(no code)]    ← Empty cc_library, only has deps
            │
            └── real_impl(has code)
```
Traversal:
1. Check wrapper → no code → must_add_children = True
2. Check real_impl → has code! → mark as top-level

Result: top_level = {real_impl}


```python
def _find_top_level_linker_input_labels(...):
    # Find the first library with actual code
    for linker_input in linker_inputs:
        if _contains_code_to_link(linker_input):
            top_level_linker_input_labels_set[owner] = True
            break  # Found it, stop going deeper
```

### Wrapping with alwayslink

#### Why use alwayslink (whole-archive)?

By default, the linker only pulls in object files that resolve undefined symbols. This is a problem for:
- Plugin entry points — functions called by the host, not by the library itself
- Static initializers — code that runs at load time
- Registration patterns — e.g., `REGISTER_MODULE(MyModule)`

Without `alwayslink`, these symbols would be silently dropped.

What it does:

Without alwayslink:
- Linker: "No one calls `init()`, skip it"
- Result: Missing symbols at runtime

With alwayslink:
- Linker: "Include everything from this .a"
- Result: All symbols available

The code:

```python
new_library_to_link = cc_common.create_library_to_link(
    alwayslink = True,  # Equivalent to -Wl,--whole-archive
)
```

This is applied only to **top-level** libraries (direct `deps`), not transitive dependencies.

### Handling Unused Dynamic Dependencies

The main loop only sees dependencies from `deps`. But what if a `dynamic_deps` exports something nobody uses?

Example: In our running example, `libX.so` exports `X`, but no one depends on `X`. The main loop never encounters `X`.

Solution: After the main loop, `_add_unused_dynamic_deps` ensures all explicitly declared `dynamic_deps` are linked:

```python
for dynamic_dep in unused_dynamic_linker_inputs:
    if dynamic_dep not in already_linked:
        link(dynamic_dep)  # Ensure it's linked even if unused
```

This is needed because users may declare a `dynamic_deps` for runtime purposes (e.g., a plugin needs `core.so` at load time, even without compile-time symbol dependencies).

## Error Detection

`cc_shared_library` includes several error checks:

### Duplicate Export Check

```python
def _build_exports_map_from_only_dynamic_deps(...):
    for export in exports:
        if export in exports_map:
            fail("Two shared libraries export the same symbols: " + ...)
```

### Duplicate Static Link Check

```python
def _build_link_once_static_libs_map(...):
    for static_lib in link_once_static_libs:
        if static_lib in link_once_static_libs_map:
            fail("Two shared libraries link the same library statically: " + ...)
```

### Linked But Not Exported Check

When library A is statically linked by `libB.so` but not exported, and `libC.so` (with `dynamic_deps: [libB.so]`) also needs A, you get a conflict.

Fix: Either export A via `exports_filter`, or create a separate `libA.so`.

### LINKABLE_MORE_THAN_ONCE Tag

By default, a library can only be statically linked by one `cc_shared_library`. To allow multiple linking:

```python
cc_library(
    name = "safe_lib",
    tags = ["LINKABLE_MORE_THAN_ONCE"],
)
```

Caution: Use sparingly — only for libraries without static initializers or global state.

## Summary

The core logic of `cc_shared_library` is simple:

> **If a library is exported by a `dynamic_deps` → dynamic link. Otherwise → static link.**

The key optimization is **pruning**: when a library is dynamically linked, its transitive dependencies are not traversed — they're already handled by the exporting shared library.

Everything else (topological sorting, alwayslink, error detection) supports this core decision.

## Reference

- [YouTube: Bazel’s Take on (Cc) Shared Libraries - Claudio Bley, Modus Create](https://www.youtube.com/watch?v=Y7qh-RGtkjg), [Slide](https://static.sched.com/hosted_files/bazelcon2024/b2/BazelCon%20cc_shared_library.pdf)
- [Blog: Linux shared libraries with CMake and Bazel](https://ltekieli.github.io/linux-shared-libraries-with-cmake-and-bazel/)

---

*This blog is based on analysis of [rules_cc cc_shared_library.bzl](https://github.com/bazelbuild/rules_cc/blob/e3a884c864540b8a93fe906add87b21aab1004f4/cc/private/rules_impl/cc_shared_library.bzl) in rules_cc. Corrections welcome.*
