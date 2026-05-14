+++
date = '2026-05-14T11:15:56+08:00'
draft = false
title = 'Your C++ struct is the schema: a proto3 serializer in C++26 reflection'
tags = ['C++', 'C++26 Reflection']
toc = true
+++


I built a header-only proto3 wire-format library with one constraint: **no `.proto` files, no codegen, no descriptor runtime**. The user writes a plain C++ struct, and that struct *is* the schema:

<!--more-->

```cpp
#include "proto3.hpp"

struct SearchRequest {
    std::string  query;             // field 1
    std::int32_t page_number;       // field 2
    std::int32_t results_per_page;  // field 3
};

SearchRequest req{"hello", 1, 10};
std::string   bytes = proto3::serialize(req);          // -> proto3 wire bytes
SearchRequest back  = proto3::deserialize<SearchRequest>(bytes);
```

That's the whole API surface for the common case. The library walks the struct's data members at compile time using C++26 static reflection, picks a wire format for each based on its type, and emits/parses bytes that conform to the proto3 wire-format spec — any conformant decoder should accept the output. Field numbers default to declaration order; override with `[[= proto3::field(N)]]`.

This post is about the C++26 reflection primitives that make it work, the type-driven dispatch they enable (where adding a new type category becomes a mechanical edit), and the handful of genuinely tricky bits — particularly how `oneof` annotations, compile-time field-number validation, and unknown-field preservation came together.


The full source is ~1400 lines of header (one core header plus a companion for well-known types), ~2000 lines of tests across 10 targets (including a property-based fuzzer), and 5 runnable examples. The patterns are small enough to absorb in a sitting.

## C++26 reflection in five primitives

Static reflection in C++26 is centered on [P2996](https://wg21.link/p2996) (the `<meta>` header plus two new operators, `^^` and `[: :]`), with siblings like the `template for` loop coming in alongside it. Compiler support outside gcc is patchy as of writing — I built this against **gcc 16.1** from Ubuntu, with `-std=c++26 -freflection`. No compiler patches needed; check your distro and toolchain before you commit, since this is still "real but bring-your-own-compiler" territory.

The five things this project uses:

```cpp
^^T                                       // a meta::info value naming type T
[: m :]                                   // splice — turn a meta::info back into a usable token
meta::nonstatic_data_members_of(^^T, ...) // a vector of meta::info, one per data member
meta::annotations_of(m)                   // annotations attached to entity m
template for (constexpr auto x : range)   // iteration where the loop body is instantiated per element
```

`^^T` is a `consteval` operator that produces an opaque handle (`meta::info`) for any reflected entity — a type, a member, an annotation. `[: m :]` is the inverse splice: given a `meta::info` for, say, a class member, `obj.[:m:]` is the actual member access. So `meta::nonstatic_data_members_of(^^T)` plus `[:fields[I]:]` lets you write code that walks a struct's members generically, without touching member names:

```cpp
template <std::size_t I, class T>
void encode_one(std::string& out, const T& msg) {
    constexpr static auto fields = fields_of_type(^^T);
    constexpr auto m = fields[I];
    using FieldType = std::remove_cvref_t<decltype(msg.[:m:])>;
    if constexpr (is_optional_v<FieldType>) { ... }
    else if constexpr (is_vector_v<FieldType>) { ... }
    else if constexpr (is_message_v<FieldType>) { ... }
    // ...
}
```

The whole encoder is one `if constexpr` ladder per field, dispatched by the field's classification trait. No virtual calls, no descriptor lookup — every type-classification decision happens at compile time, and the function body collapses to straight-line scalar / string / length-prefixed writes (with normal runtime length checks at I/O boundaries).

## Annotations as constexpr values, not strings

The most novel bit of C++26 (to me) is that attribute syntax can carry constexpr *values*:

```cpp
struct field_number_t { int number; };
consteval field_number_t field(int n) { return {n}; }

struct MyMsg {
    [[= proto3::field(5)]] std::int32_t x;
};
```

`[[= ann]]` attaches the *value* `ann` (which has type `field_number_t`) to `x`. You can then ask reflection what annotations a member has:

```cpp
template <std::size_t I, class T>
consteval int field_number_at() {
    constexpr static auto fields = fields_of_type(^^T);
    constexpr auto a = get_annotation(fields[I], ^^field_number_t);
    if constexpr (a) {
        return meta::extract<field_number_t>(*a).number;
    } else {
        return static_cast<int>(I) + 1;  // declaration-order default
    }
}
```

This isn't a string the compiler sticks somewhere. It's a typed value. `meta::extract<T>` pulls it back out as the original C++ object, so an annotation is just a constexpr struct with whatever fields you want. Far cleaner than the C++ attribute story has ever been.

## The full annotation surface

Nine annotations in total, all in the `proto3::` namespace:

| Annotation | Purpose |
|---|---|
| `field(N)` | Override the wire field number (default: declaration order, starting at 1). |
| `skip` | Omit a member from serialization entirely — local-only state. |
| `zigzag` | Encode a signed-int (or `vector` / `optional` of same) as proto3 `sint32` / `sint64` — zigzag varint. |
| `fixed` | Encode a 4- or 8-byte integer (or `vector` / `optional` of same) on the I32 / I64 fixed-width wire — proto3 `fixed32` / `sfixed32` / `fixed64` / `sfixed64`. |
| `bytes` | Documentation marker on a `std::string` field that carries `bytes` rather than `string`. No wire effect; rejected at compile time on any other field type. |
| `oneof<N1, N2, ...>` | Mark a `std::variant<std::monostate, ...>` as a proto3 oneof, one wire field number per non-monostate alternative. |
| `as_timestamp` | Route a `std::chrono::system_clock::time_point` field through `proto3::Timestamp` on the wire. |
| `as_duration` | Route a `std::chrono::nanoseconds` field through `proto3::Duration` on the wire. |
| `unknown_fields` | Mark a `std::string` member as the sink for unknown-field preservation. |

The two with the most non-obvious wire impact are `zigzag` and `fixed`:

```cpp
struct M {
    std::int32_t                       n;       // 10 bytes for n = -1 (sign-extended varint)
    [[= proto3::zigzag]] std::int32_t  delta;   //  1 byte  for delta = -1 (zigzag(-1) = 1)
    [[= proto3::fixed]]  std::uint32_t hash;    //  4 bytes always, regardless of value
};
```

Use `zigzag` for signed fields that frequently go negative (deltas, offsets); the default `int32` encoding sign-extends to 10 bytes. Use `fixed` when values are large or random and a compact varint won't help — for `uint32` values above 2²¹ or so, the 4-byte fixed-width encoding wins. The whole set is deliberately small: this is what `protoc` would call "field options," minus everything that requires a `.proto` file context (custom options, JSON name overrides, etc.).

## The oneof puzzle: when each annotation is a *different type*

proto3 `oneof` (a tagged union of alternatives, each with its own wire field number) is the trickiest annotation in the library. The natural C++ mapping is `std::variant<std::monostate, T1, T2, ...>` (the leading `monostate` represents "unset"), and the natural annotation carries the per-alternative field numbers:

```cpp
struct Resp {
    std::int32_t id;
    [[= proto3::oneof<5, 6, 7>]]
    std::variant<std::monostate, std::int32_t, std::string, ErrorDetail> result;
};
```

The annotation is a value of type `oneof_t<5, 6, 7>` — and here's the wrinkle: every different field-number pack produces a *different type*. `oneof_t<5, 6, 7>` and `oneof_t<1, 2>` are unrelated. So I can't filter annotations by their exact `meta::info` type — there's no single `^^oneof_t<...>` that catches them all.

The fix uses `template for`, the C++26 construct that iterates over a constexpr range with each iteration getting its own constexpr context:

```cpp
template <std::size_t I, class T>
consteval std::optional<meta::info> find_oneof_ann_at() {
    constexpr static auto fields = fields_of_type(^^T);
    constexpr auto m = fields[I];
    constexpr static auto anns =
        std::define_static_array(meta::annotations_of(m));
    std::optional<meta::info> result;
    template for (constexpr auto ann : anns) {
        using AnnRaw = [: meta::type_of(ann) :];
        using AnnT = std::remove_cv_t<AnnRaw>;
        if constexpr (is_oneof_spec_v<AnnT>) {
            result = ann;
        }
    }
    return result;
}
```

For each annotation on the member, `template for` gives me a constexpr `ann` value. I splice its type with `[: meta::type_of(ann) :]` and probe with the trait:

```cpp
template <class T> struct is_oneof_spec : std::false_type {};
template <int... Ns> struct is_oneof_spec<oneof_t<Ns...>> : std::true_type {};
```

In C++20 this kind of "find the annotation whose type matches a variable-arity template" walk would have meant a macro DSL or a CRTP layer that pre-stamped each member with a typed tag. In C++26 it's a few-line library function over generic data. The `std::define_static_array` call is what materializes a consteval-computed `std::vector` into an addressable static array so `template for` can iterate it — another small but load-bearing C++26 helper.

## Type-driven dispatch via type traits

Once reflection hands you the field type, encoding is mostly a classification problem. Plain `is_class_v` isn't enough because `std::string`, `std::vector`, `std::map`, `std::variant`, and `std::optional` are all class types but each has its own wire representation. So `is_message_v` is defined by *exclusion*:

```cpp
template <class T>
inline constexpr bool is_message_v =
    std::is_class_v<T> &&
    !std::is_same_v<T, std::string> &&
    !is_vector_v<T> &&
    !is_map_v<T> &&
    !is_variant_v<T> &&
    !is_optional_v<T>;
```

If you forget to add `!is_optional_v<T>` here, `std::optional<int>` gets misclassified as a nested message and the encoder tries to walk its private members. Adding `optional` support was a five-piece change in the same shape every other type category took: a new `is_optional_v` trait, the exclusion line in `is_message_v` above, an encode branch, a decode branch, and tests. The whole feature was ~80 lines, but every piece dropped into a place the architecture already had. Composability falls out of the type system.

## Compile-time field-number validation

proto3 has three rules about field numbers: they must be in `[1, 2^29 − 1]`, can't fall in the reserved range `[19000, 19999]`, and must be unique within a message. All three are checkable at compile time:

```cpp
template <class T>
consteval auto collect_field_numbers() {
    constexpr std::size_t N = count_active_field_numbers<T>();
    constexpr static auto fields = fields_of_type(^^T);
    std::array<int, N> nums{};
    std::size_t pos = 0;
    [&]<std::size_t... Is>(std::index_sequence<Is...>) {
        (emit_field_numbers_at<Is, T>(nums, pos), ...);
    }(std::make_index_sequence<fields.size()>{});
    return nums;
}

template <class T>
consteval bool field_numbers_unique() {
    constexpr auto nums = collect_field_numbers<T>();
    auto sorted = nums;
    std::sort(sorted.begin(), sorted.end());
    return std::adjacent_find(sorted.begin(), sorted.end()) == sorted.end();
}
```

`std::sort` and `std::adjacent_find` are constexpr since C++20, so the whole pipeline runs at compile time. Three `static_assert`s at the top of `serialize_into` and `deserialize_from` make a malformed schema break the build with a specific error message instead of silently producing non-conformant wire bytes.

The handy bit: `field_numbers_unique<T>()` is a regular `consteval bool` predicate. So you can write `static_assert(!proto3::field_numbers_unique<BadStruct>())` in tests to verify it correctly *rejects* bad schemas without breaking the build. Compile-time tests for compile-time validators.

## Unknown-field preservation: a one-byte snapshot trick

proto3 (since ~2017) requires that unknown fields survive a `deserialize → serialize` round-trip. The whole feature ended up being ~120 lines (annotation tag + reflection helpers + dispatch tweaks + static_asserts), but the *trick* at the heart of it is a single snapshot. An opt-in annotation marks one `std::string` member as the sink:

```cpp
struct MyMsg {
    std::int32_t known;
    [[= proto3::unknown_fields]] std::string unknown_;
};
```

The decoder loop already had:

```cpp
while (!in.empty()) {
    auto tag = read_varint(in);
    int fn = tag >> 3;
    wire_type wt = wire_type(tag & 0x7);
    bool handled = ...;
    if (!handled) skip_field(in, wt);
}
```

The change: snapshot the input pointer *before* reading the tag, then on the no-match branch, after `skip_field` advances past the value, splice the byte range straight into the sink:

```cpp
const char* field_start = in.data();
auto tag = read_varint(in);
// ... handling chain ...
if (!handled) {
    skip_field(in, wt);
    constexpr auto uf_idx = unknown_fields_index<T>();
    if constexpr (uf_idx.has_value()) {
        constexpr auto m = fields[*uf_idx];
        msg.[:m:].append(field_start, in.data() - field_start);
    }
}
```

`skip_field` was already written for forward-skipping. By taking a snapshot before the tag varint, the slice `[field_start, in.data())` is the entire (tag + value) bytes for the unrecognized field. On encode, those bytes get appended at the tail of the output after the known-field fold. The result is conformant proto3 (decoders accept any field order) but not byte-stable — which I think is the right trade for an opt-in feature.

The `if constexpr` matters: structs *without* the annotation keep the old behavior (`skip_field` and discard) at zero runtime cost. The annotation is opt-in by design — you only pay for the feature when you ask for it.

## Chrono types as Timestamp/Duration via converting types

Proto3 has well-known types like `google.protobuf.Timestamp` (seconds + nanos). The natural C++ ergonomic is `std::chrono::system_clock::time_point`. But the wire format demands a struct with two scalar fields, so a type alias would silently produce wrong bytes (chrono's internals are private and don't match).

Two annotations bridge it:

```cpp
struct Event {
    [[= proto3::as_timestamp]] std::chrono::system_clock::time_point t;
    [[= proto3::as_duration]]  std::chrono::nanoseconds              ttl;
};
```

The `proto3::Timestamp` and `Duration` structs (defined in a companion header) carry implicit conversions to/from the chrono types. The annotation routes the field through a generic `encode_field_as<Wire>` helper that converts to the wire-side struct before serializing:

```cpp
template <class Wire, class T>
void encode_field_as(std::string& out, int fn, const T& v) {
    using U = std::remove_cvref_t<T>;
    if constexpr (is_optional_v<U>) {
        if (!v.has_value()) return;
        Wire w(*v);                    // converting ctor
        encode_field_present<Wire>(out, fn, w);
    } else if constexpr (is_vector_v<U>) {
        for (const auto& e : v) {
            Wire w(e);
            encode_field_present<Wire>(out, fn, w);
        }
    } else {
        Wire w(v);
        encode_field_present<Wire>(out, fn, w);
    }
}
```

Composes with `std::vector` and `std::optional` automatically. The chrono types stay first-class C++ values in user code; the wire-side `Timestamp` is an implementation detail. Wire bytes are byte-identical to a hand-written `proto3::Timestamp` field. And because the helper is parameterized on `Wire`, the same machinery would work for any user-defined type with a wire-side struct twin (UUIDs as `bytes`, BigDecimal as a struct, etc.).

The one architectural footnote: the dispatch in `proto3.hpp` needs to *name* `Timestamp`/`Duration`, but their full definitions live in `well_known.hpp` to keep the chrono dependency optional. Forward-declare in `proto3.hpp`, define in `well_known.hpp`. If the user hits the annotation without including `well_known.hpp`, they get a normal "incomplete type" error at template instantiation, which is the right time to discover they need the second header.

## What surprised me

A few things landed differently than I expected:

- **`if constexpr` does the work of a visitor pattern** without any of the boilerplate. The encoder is one big switch — but the switch is at compile time, so each instantiation collapses to the relevant branch.

- **Compile-time validators are testable like runtime ones.** `static_assert(!proto3::field_numbers_unique<BadStruct>())` reads exactly like a unit test, runs at compile time, and the build fails if you've broken the validator. No separate "test that the static_assert fires" framework needed.

- **Type-classification traits are the architecture.** Adding a new category (optional, then chrono types, then unknown-fields-sink) is mechanical: define `is_X_v`, exclude from neighboring traits as needed, add an `if constexpr` branch in the encoder and decoder. Each addition was 40-80 lines.

- **Proto3 has only 4 wire types and ~10 type categories total** (`VARINT`, `I64`, `LEN`, `I32` — proto2's `GROUP_*` were dropped). Once the dispatch chain exists, supporting another type category follows the same five-piece template. The library's growth pattern — "new annotation + new dispatch branch + new test file" — became almost rote.

## Where it doesn't go (and what it isn't competing with)

The library covers every wire-format type in proto3, plus the well-known types as plain structs. It explicitly *doesn't* implement:

- **Canonical JSON encoding.** Proto3 has a defined JSON mapping (RFC 3339 for `Timestamp`, etc.); the same reflection machinery would drive it but the scope is significant.
- **Runtime descriptor reflection.** `protoc`-generated code provides a `Descriptor`/`Reflection` API for inspecting types at runtime. Compile-time reflection doesn't try to be that.
- **Stack-overflow protection on deeply-nested input.** Easy to add (thread-local depth counter + RAII guard), but I haven't yet.

It's also not trying to replace **nanopb** or **protobuf-lite** for production use. nanopb (a battle-tested C library used in embedded for over a decade) and protobuf-lite (the trimmed-down Google C++ runtime) both have years of conformance testing, plug into existing `.proto`-based toolchains, and give you JSON / text format / descriptor APIs. This project is the answer to a different question: *what can you build when the C++ struct itself is the schema and the compiler does the introspection?* The point is the architectural shape, not displacing existing libraries.

The next features for anyone forking it are obvious from the gaps above. The codebase is small enough — one header, plus a companion for well-known types — that adding them shouldn't require rearchitecting anything.

## Closing

The thing that stuck with me is how cleanly C++26 reflection lets the *type system* carry the schema. There's no parallel `.proto` file to keep in sync, no `protoc` build step, no generated code with funny names to integrate around. The struct is the schema. The annotations are values. The encoder is `if constexpr`.

I'm not sure how many people will write proto3 in C++26 specifically (toolchain availability is still uneven outside gcc 16.1), but the reflection patterns generalize: anything that needs to walk a struct's fields generically — JSON, Avro, command-line parsers, ORM mappers — gets the same payoff. If you've been waiting for "what is C++ reflection actually good for," this is one concrete answer.

---

*Source code: https://github.com/PikachuHyA/struct_proto26 . Header-only. Builds against gcc 16.1 (Ubuntu) with `-std=c++26 -freflection` and any Bazel 9+. PRs welcome.*
