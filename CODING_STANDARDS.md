# Rust Coding Standards

Conventions for Rust workspaces. Apply them from the first draft of new
code, not as a fix-up pass.

This document is built on Microsoft's
[Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/)
([single-file text edition](https://microsoft.github.io/rust-guidelines/agents/all.txt),
last compared 2026-10-23). The upstream text is not copied into this
repository: guidelines taken from it carry their upstream ID, such as
`M-PANIC-ON-BUG`, so look the ID up there for the full rationale and
examples. Rules without an ID are local. Where we deliberately differ from
upstream, the text is marked **Local override** and says what we do instead.

Sections marked *(conditional)* apply only when the project uses the
technology they name (applications, FFI, macros, Bazel, a port of existing
code, performance-sensitive code).

## Hard rules

No exceptions. These are not preferences.

**1. No `.unwrap()` / `.expect()` in production code.** This holds even
for a precondition that seems impossible to violate.

- Enforce it with `clippy::unwrap_used` / `clippy::expect_used` = `deny` in
  `[workspace.lints.clippy]`. The lint is the mechanism, not the rule. Don't
  route around it with a local `#[allow(clippy::expect_used)]` because a
  case is "unreachable".
- If a precondition cannot be violated, handle it explicitly: propagate a
  `Result` with a real (if currently unreachable) `thiserror` error
  variant.
- Test code may use `.unwrap()` / `.expect()` freely, via `clippy.toml`
  (`allow-unwrap-in-tests`, `allow-expect-in-tests`).
- **Local override of `M-PANIC-ON-BUG`.** An explicit `panic!`, `assert!` or
  `unreachable!` is allowed where a caller broke a documented contract
  (`M-PANIC-IS-STOP`). It must carry a message that states what went wrong
  and the relevant values (`M-PANIC-MESSAGE`). Never use `unwrap()` /
  `expect()` for this, and prefer a `Result` wherever the failure could
  plausibly be handled.

**2. No redundant `#[allow(clippy::unwrap_used)]` /
`#[allow(clippy::expect_used)]` in test code.** `clippy.toml` already
exempts recognized test code globally. A redundant `#[allow]` on a
`#[cfg(test)] mod tests` is dead weight, and would mask `clippy.toml`
silently not working.

**3. Prefer `#[expect(lint, reason = "...")]` over `#[allow(lint)]`** for
any suppression that survives review (`M-LINT-OVERRIDE-EXPECT`).
`#[expect]` (stable since Rust 1.81) emits `unfulfilled_lint_expectations`
when the lint stops firing, so stale suppressions get caught. `#[allow]` is
the exception, and every override states a `reason`.

- Known exception: `dead_code` on an item exercised only by its own
  `#[cfg(test)]` tests. The item is dead under a plain `cargo build` but
  live under `--all-targets` / `cargo test`, so `#[expect(dead_code)]`
  would warn in whichever configuration it doesn't fire in. Use
  `#[allow(dead_code, reason = "...")]` there, with a reason that gives this
  explanation so it isn't mistaken for an oversight.
- `#[allow]` is also fine on generated code and inside macros.

**4. Never use a `&mut` output parameter to return a result.** Return an
owned value, always, even for a single result.

- Example: a DMA-read helper returns `Result<[u8; N]>` (const-generic size
  fixed at the call site) instead of filling a caller-supplied
  `&mut [u8]`.
- For a size known only at runtime, write a distinct function that takes an
  explicit length and returns `Vec<u8>`.
- Multiple results: return a tuple. A function that partitions input into
  two groups returns `(GroupA, GroupB)`, not one group via `&mut` plus the
  other as the return value.
- No clippy lint catches this. It is a review-time rule.
- **Local override of `M-MEM-REUSE`.** A buffer-reuse variant
  (`get_in(id, &mut value)`, reset with `.clear()`) is allowed only as an
  extra function beside the owned-return one, only on a hot path that has
  been profiled or benchmarked, and with the evidence noted in its doc
  comment. The owned-return function stays the default API.

**5. All code is sound** (`M-UNSOUND`). A safe function must not be able to
cause undefined behavior from any calling mode, however remote. If you
cannot encapsulate something safely, expose an `unsafe` function and
document its contract. Soundness boundaries are module boundaries: a safe
function may rely on invariants guaranteed elsewhere in the same module.

## Correctness

- **Panics mean "stop the program"** (`M-PANIC-IS-STOP`). Don't use them to
  report errors upstream, to handle self-inflicted error conditions, or
  assume they will be caught (`panic = "abort"` makes them fatal).
- **`catch_unwind` is a last resort** (`M-PANIC-CONTINUATION`) and must be
  followed by a controlled restart. Libraries must not catch a panic and
  carry on. A server may catch one per request so the others can finish, but
  should still restart afterwards.
- **Detected bugs panic, they are not errors** (`M-PANIC-ON-BUG`), subject to
  the local override in hard rule 1. Where possible, use the type system so
  the panicking path cannot be reached ("correct by construction").
- **`unsafe` implies undefined behavior** (`M-UNSAFE-IMPLIES-UB`): mark a
  function or trait `unsafe` only when misuse risks UB, never merely because
  it is dangerous (`delete_database()` is not `unsafe`).
- **`unsafe` needs a valid reason** (`M-UNSAFE`): a novel abstraction,
  measured performance, or FFI/platform calls. Never use ad-hoc `unsafe` to
  shorten safe code, bypass `Send` bounds or lifetimes (`transmute`).
  - Every `unsafe` use carries a plain-text `SAFETY:` justification.
  - Novel abstractions and performance code must pass Miri, including
    adversarial test cases, and follow the unsafe code guidelines.
  - Benchmark before using `unsafe` for performance.

## Project structure

- **Use a workspace `Cargo.toml`** (`M-CARGO-WORKSPACE`). Members inherit
  metadata, dependency versions (`[workspace.dependencies]`) and lints
  (`[workspace.lints]`) from the root. Workspace dependency entries use
  `default-features = false` and enable only basic features such as
  `["std"]`.
- **Every third-party dependency is declared once, in the workspace
  `Cargo.toml`, and its reason is written down once.** A table in the
  project's conventions file (the crate, what it is for, why this one and what
  not to hand-roll, where it is used) gets a row in the same change that adds
  the dependency, and its line in `[workspace.dependencies]` goes in
  alphabetical order, not after whatever block was edited last.
  - The reasons for choosing a crate live in the table alone, so the two
    places don't drift.
  - A `Cargo.toml` comment stays only where it guards its line from a wrong
    edit: a version pin, a deliberately small feature set.
  - Neither cites a task or design document that a reader of the directory may
    not have.
- **List every crate in the workspace and depend on siblings through it**
  (`M-CRATES-IN-WORKSPACE`): `sibling.workspace = true`, with
  `sibling = { path = "...", version = "..." }` under
  `[workspace.dependencies]`, never `sibling.path = "../sibling"`.
- **Crates are siblings, never nested** (`M-CRATES-FLAT-FOLDER`). One
  workspace, no crate inside another crate's directory or `src/`. A flat
  `crates/` folder suits up to 1-2 dozen crates; beyond that, group by
  folder. Express relationships through name prefixes (`foo`, `foo_util`).
  A project that is known from the start to grow into many crates may group
  them by layer (next bullet) instead of one flat folder.
- **Layer rule: dependencies point only downward.** Define the layers up
  front, give each its own directory, and allow path dependencies only from
  a higher layer to a lower one. For example:
  `tools → services → {pure-logic, platform} → sys`.
  - A pure-logic layer never depends on platform or service layers. It does
    no I/O and is testable on any host.
  - This keeps libraries usable by non-CLI consumers (daemons, operators,
    services).
  - Name the layers that may use an async runtime, and keep every other
    layer synchronous.
  - A base layer of fundamental extensions to std types may be depended on by
    every layer (build scripts included) and depends on no other workspace
    crate.
  - A layer that consumes a service through its API depends on the API's
    crates only (the types and the client), never on the code that implements
    the service. Its tests may use that code as dev-dependencies.
  - Cargo and Bazel don't enforce the rule. An upward or sideways
    dependency is a design problem, not a convenience.
- **If in doubt, split the crate** (`M-SMALLER-CRATES`). If a submodule can
  be used independently, make it a crate. Crates are for things usable on
  their own; features are for extras that can't stand alone.
- **New crates use the latest stable edition** (`M-LATEST-EDITION`; at least
  2024). An older edition grants no ecosystem compatibility.
- **Set an MSRV and update it conservatively** (`M-MSRV`): a few releases
  behind the newest compiler. Bumping it is a minor-version change.
- **Read the release version from one crate; no per-crate `build.rs` for
  it.** The repo-wide release string reaches Rust through a single base-layer
  crate that owns the only build script that reads it. It exposes the string
  as a constant and a hand-written `Version` parser when the format is not
  semver (a Python-style `0.2.0.dev0+unknown`, say). A crate that needs the
  version, such as a binary's `--version` or an agent's build info, depends
  on that crate instead of reading `env!` itself.
  - A nightly or release-candidate string carries the commit hash, so it
    changes with every commit and everything depending on the crate
    recompiles. Only binaries and the top layers depend on it, never the
    low-level libraries.
  - Compare builds by the release part of `Version`, never by the whole
    string.
- **`Cargo.toml` `description` is terse**: a short phrase, not a paragraph.
  A longer description of what the crate is, its dependency relationships
  and links to its design documents goes in the crate's `//!` doc comment
  (see `M-NO-META-DESIGN-DOCUMENTATION`).
- **TOML spacing**: two blank lines between `[section]` blocks; one blank
  line between logical groups within a section.
- **No redundant `[[bin]]`** when the default `src/main.rs` inference
  already yields the right binary.

## Modules and imports

- **File order**: `use` statements and re-exports first (`pub use`, and
  `pub type` aliases such as the crate's `Result`, which are re-exports of
  a kind), then `mod` / `pub mod` declarations.
- **`use super::*` in submodules** instead of `use crate::xxx` paths. Hoist
  shared imports once at the parent (e.g. `lib.rs`).
- **Private `mod` for hoisted submodules.** If a submodule's items are
  re-exported at the parent (`use foo::{...};` in `lib.rs`), keep the
  submodule a private `mod`. The re-exports are the public surface. This is
  also what keeps each public item reachable through one path
  (`M-SINGLE-ITEM-PATH`): don't make a module `pub` and also re-export its
  items.
- **`module.rs`, not `module/mod.rs`**, for submodules with their own
  children.
- **No glob re-exports and no preludes** (`M-NO-GLOB-REEXPORTS`,
  `M-NO-PRELUDE`). Write `pub use foo::{A, B, C};`, not `pub use foo::*;`,
  and don't define a `prelude` module. Platform-selected re-exports from a
  set of HAL modules are the one accepted glob. If a prelude seems
  necessary, the module layout needs restructuring.
- **Don't re-export other crates' items** (`M-FOREIGN-REEXPORTS`). Users
  depend on the third-party crate directly. Accepted exceptions: umbrella
  crates, crates split for technical reasons (such as a `-sys` crate and its
  safe wrapper), and hidden macro-helper paths.
- **Modules are balanced** (`M-BALANCED-MODULES`): essential items in the
  crate root, the rest grouped by use case (`account`, `network`), not by
  kind (`traits`, `errors`).
- **Module-qualified calls over bare per-item imports** when a short,
  recognizable prefix exists.
  - Standard library: `use std::fs;` then `fs::File`; `use std::io;` then
    `io::Error`.
  - External crates, aliased short if needed: `use foo_sys as sys;` then
    `sys::some_function(...)`.
  - This mainly applies to **values** (constants, functions). **Third-party
    and std types** used repeatedly in type position (`Utf8Path`, `HashMap`,
    `c_void`) stay bare-imported. Types from your own workspace crates are
    the exception: see the next rule.
  - A trait imported only for its extension methods (`itertools::Itertools`)
    must stay a bare `use`.
  - Judgment call, not a hard boundary.
- **Import a workspace crate as a short alias and use it qualified, types
  included**: `use my_parser as parser;`, then `parser::Config`,
  `parser::MAX_ITEMS`, `parser::Error`.
  - The qualifier says which crate a type comes from, and it replaces a
    block of per-item `use my_parser::X;` lines. It also matches
    `M-SHORT-NAMES`: no crate prefix baked into type names.
  - Put the alias in the crate's `lib.rs`, in its own import group after the
    external crates, and let submodules reach it through `use super::*`.
  - When the crate's own name is already short, no alias is needed: drop the
    per-item imports and write `parser::Config`.
  - Doc comments may still name items bare, and runtime message strings are
    not touched.
- **`use serde_json as json;`**, then `json::Value`, `json::json!(...)`,
  `json::from_str`. This is the previous rule applied to `serde_json`. When
  you touch a file that still imports `serde_json::Value` bare, convert it
  rather than doing a separate sweep.
- **`size_of` / `align_of` / `size_of_val` / `align_of_val`** are in the
  prelude: use them bare. **`offset_of!` is not**: write
  `use std::mem::offset_of;` and call `offset_of!(...)` bare, not
  `mem::offset_of!(...)`.

## Formatting

- **Run `cargo fmt`** (`cargo fmt --check` to verify) as part of the
  standard verification pass, alongside build, test and clippy. Don't
  hand-format.
- **Commit a `rustfmt.toml`** with `imports_granularity = "Item"`: one
  import per `use` line. (This is not rustfmt's default, which is
  `"Preserve"`.)
- **Do not set `group_imports`.** `"StdExternalCrate"` merges `use` and
  `pub use` into one sorted group, which interleaves plain imports with
  re-exports.
- **Group imports manually**: std first, then blank-line-separated groups
  (external crates / crate-local `use` / `pub use` re-exports). Each group
  is sorted by rustfmt; which group an import belongs to is a manual
  choice.

## Naming and API design

- **Names are short** (`M-SHORT-NAMES`): at most two compounded words, no
  module or crate name baked into a prefix (`foo::Id`, not `foo::FooId`),
  common abbreviations (`CallbackFn`). Callers disambiguate with qualifiers.
- **Names are free of weasel words** (`M-WEASEL-WORDS`): avoid `Service`,
  `Manager`, `Factory`. Name the specific behavior (`Bookings`,
  `BookingDispatcher`). A factory is a `Builder`; to accept repeatable
  instantiation, take an `impl Fn() -> Foo`.
- **Follow the upstream Rust API guidelines** (`M-UPSTREAM-GUIDELINES`),
  especially the ones that get forgotten: `as_` / `to_` / `into_`
  conversion naming (C-CONV), getter names without `get_` (C-GETTER),
  eagerly implemented common traits (C-COMMON-TRAITS), `Foo::new()` even if
  `Foo::default()` exists (C-CTOR), and feature names free of placeholder
  words (C-FEATURE).
- **A function whose subject is one value of a type your crate defines is a
  method on that type, not a free function.** `fn status(response: &RawEvent)
  -> i32` is `fn status(&self) -> i32` in `impl RawEvent`; a `Copy` type's
  by-value `fn index(addr: Address)` is `Address::index(self)`. This is the
  receiver case that `M-REGULAR-FN` and `M-ESSENTIAL-FN-INHERENT` allow:
  associated functions are for constructors, and functionality with no clear
  receiver stays a regular function.
  - This holds for private functions too, and when the subject is the first
    of several parameters: `fn dma(f: &Fields<'_>, src: bool)` is
    `Fields::dma(&self, src: bool)`.
  - Drop the type name from the method name (`reset_response_status`
    becomes `reset_status`). Methods also let callers chain.
  - Constructors (`Type::new`, `Type::from_x`) are associated functions,
    which is a separate rule.
  - A free function stays when:
    - **The type is not yours.** An inherent `impl` needs a type from this
      crate, so `&str`, `&[u8]`, `&Utf8Path`, `&fs::File`, `json::Value`,
      primitives, generated bindings and other workspace crates' types
      can't take one. See the extension-trait rule below.
    - **The function is one of a set of strategies**, not a property of the
      receiver. Per-opcode encoders in a dispatch table take `&Fields`, but
      "encode opcode X" is not something a `Fields` does, and forty of them
      would swamp the type. The same goes for a check or step that belongs
      to one test or command and only borrows a shared context type.
    - **The first parameter is only context.** `wait_for(table: &Mutex<..>,
      ..)` locks and unlocks across a poll loop, so it can't be a method of
      what's inside the lock. Two peer arguments with no clear subject
      (`record(a: &Record, b: &Record)`) have no receiver either. When
      `cli: &Cli` is context for a command, a command's helper is a method
      of the command, not of `Cli`. What belongs on `Cli` is what's about
      the flags themselves.
  - Where the method goes:
    - **Parallel pieces of one owner make a method of the owner.** A
      function taking two entries of `Device`'s parallel `files` and `infos`
      vectors is about the `Device`, so it is a `Device` method that does
      the zip.
    - **An `impl` block may sit in any module of the crate**, and a private
      method is visible only in the module that defines it. Put it beside
      what it serves, and widen visibility only when a second module needs
      it.
    - **Method names are per type, not per module.** Two inherent methods of
      the same name on one type collide (E0592) even when written in
      different modules, which two free functions of the same name in
      different modules never did. Rename one (`xbar` becomes `put_xbar`).
  - When you touch a public free function, moving it is an API change:
    update its re-export (`pub use`) and its callers in the same commit.
- **Extension traits for foreign types are a judgment call, judged on
  use.** They are how a function over a type you don't own becomes a method.
  - **Worth it** when it makes call chains read better
    (`inst.control_info().is_some_and(..)` over
    `queues::control_info(inst).is_some_and(..)`), or when the function is
    part of the crate's public API that callers will use as a method.
  - **Not worth it** for an internal helper called once or twice: the trait,
    its import and its impl cost more than the free function.
  - Example that earned it: ten accessors over an instruction's
    `json::Value`, with about 60 call sites, in one `InstExt` trait defined
    next to the code that uses it and brought into scope at the crate root.
    The readers whose subject is not one instruction (value-level helpers,
    helpers over the whole entry array) stay free.
  - A large family of free functions over a foreign type with no clear
    subject (for example `&fs::File`, where a wrapper type already is the
    method layer) calls for a local newtype, or nothing. That is its own
    change.
- **Essential functionality is inherent** (`M-ESSENTIAL-FN-INHERENT`):
  trait impls forward to inherent methods, so users don't have to find the
  right trait to `use`.
- **Parameter order is consistent** (`M-PARAMETER-CONSISTENCY`): call-specific
  parameters first, ubiquitous ones (`&logger`) last, closures always last
  and at most one.
- **Don't expose smart pointers or nested generics in APIs**
  (`M-AVOID-WRAPPERS`, `M-SIMPLE-ABSTRACTIONS`). Hide `Arc<Mutex<..>>`,
  `Rc`, `Box`, `RefCell` behind simple `&T` / `&mut T` / `T` signatures.
  Service-like types should nest generics at most one level.
- **Prefer types over generics, generics over `dyn Trait`**
  (`M-DI-HIERARCHY`). Use a concrete type (an enum, if the only other
  implementation is a mock), then narrow traits with generic parameters,
  and only then `dyn Trait` hidden behind a custom wrapper. Don't port a C#
  or Java `IFoo` interface 1:1.
- **Complex construction uses builders** (`M-INIT-BUILDER`): with four or
  more initialization permutations, provide `Foo::builder()` returning a
  `FooBuilder` whose setters are named `x()` (not `set_x()`) and end in
  `.build()`. Required parameters go to the builder's constructor, not to
  setters. Setters don't fail: `.build()` validates and returns the
  `Result` (`M-BUILD-RESULT`).
- **Long parameter lists cascade into helper types** (`M-INIT-CASCADED`):
  four or more parameters become grouped types, such as `Account` and
  `Currency`, rather than four loose strings.
- **Use strong types** (`M-STRONG-TYPES`, `M-STRONG-TYPES-GUARD`). Use the
  most specific type available, as early as possible (paths are path types,
  not `String`). A newtype that encodes an invariant enforces it: at least
  one fallible constructor, conversions from weaker types through
  `TryFrom` / `FromStr`, and no infallible `From`. A panicking `const fn
  new` may be added. Plain numbers stay plain numbers at API boundaries
  (`window_size() -> usize`).
- **Accept the most general input type** (`M-IMPL-ASREF`, `M-IMPL-IO`,
  `M-IMPL-RANGEBOUNDS`). Function parameters take `impl AsRef<str>` /
  `impl AsRef<Utf8Path>` / `impl AsRef<[u8]>` where ownership isn't needed
  (but not in struct fields). One-shot I/O takes `impl std::io::Read` /
  `Write` rather than a `File`. Ranges take `impl RangeBounds<T>`, never two
  loose bounds.
- **Public types are `Send`** (`M-TYPES-SEND`), and so are futures they
  produce. A type that is only used instantaneously may be `!Send`.
- **Services are cheap to `Clone`** (`M-SERVICES-CLONE`): heavyweight shared
  types wrap an `Arc<Inner>` so a clone is a new handle.
- **Custom collections implement the iterator traits**
  (`M-COLLECTION-TRAITS`): `Iter`, `IterMut`, `IntoIter`, `IntoIterator` for
  all three receivers, `FromIterator`, `Extend`, and a truthful
  `size_hint()`.
- **Functions are `async fn`** (`M-ASYNC-FN`) unless the signature must be
  `impl Future` (inside traits, or a hot future per `M-ASYNC-STACK-SIZE`).
- **Public types are `Debug`** (`M-PUBLIC-DEBUG`), with a custom
  implementation and a unit test for types holding sensitive data. **Types
  meant to be read are `Display`** (`M-PUBLIC-DISPLAY`), including every
  error type.
- **Magic values are named constants with a comment** (`M-DOCUMENTED-MAGIC`)
  saying why the value was chosen, what changing it affects and which
  external system relies on it.
- **Don't port code 1:1 from C#, Java or C++** (`M-RUST-SHAPED`). Domain
  logic (an algorithm) carries over; technical constructs (null checks,
  interfaces, reflection, statics) are rewritten as Rust idiom. A striking
  technical similarity to the original is a smell. See "Ports and parity"
  for the exact-behavior requirement that still applies.
- **Don't leak external types** (`M-DONT-LEAK-TYPES`). Public APIs prefer
  `std` types over another crate's. Leaking is acceptable behind a feature
  flag, between sibling crates of an umbrella crate, or where it gives a
  substantial interoperability benefit.
  - **Local override.** This applies to crates published or consumed outside
    this workspace. Crates internal to the workspace may expose `camino`
    paths, `nix` results and sibling-crate types, as the rules in this
    document require.

## Types, errors and idiom

- **`thiserror` for error types**, not hand-written `Display` /
  `std::error::Error`. Use `#[derive(Debug, thiserror::Error)]`,
  `#[error("...")]` per variant, and `#[source]` on wrapped errors.
  - **Local override of `M-ERRORS-CANONICAL-STRUCTS`.** We use `thiserror`
    enums, not a struct carrying a `Backtrace` and hidden `ErrorKind`,
    because callers match on variants (including wrapped `Errno` values).
  - Still follow the upstream intent: give separate public error types to
    unrelated failure domains (`DownloadError`, `VmError`) and a reusable
    one to similar operations (`ParseError`), not one global enum.
  - Implement `From<Other> for Error` (`#[from]`) rather than repeating
    `.map_err(..)` at call sites (`M-FROM-ERROR`). Use `map_err` only for
    foreign errors or to add context.
- **Every crate names its main error type `Error`**, and callers write
  `<alias>::Error` to say whose it is (see the workspace-crate alias rule).
  - Never use a crate-prefixed name (`ParserError`, `IsaError`), and never
    import `Error` bare next to another crate's.
  - A narrower second error type in the same crate keeps a descriptive name
    (`parser::ConfigError`, `parser::ManifestError`).
- **Crate-local `Result` alias defaults its error type**:
  `pub type Result<T, E = Error> = std::result::Result<T, E>;`
- **Avoid explicit type annotations** unless needed (ambiguous numeric
  literals, non-obvious `.collect()` targets, or public fn / struct-field
  signatures).
  - `clippy::redundant_type_annotations` catches the easy cases such as
    `let foo: String = String::new();`.
  - It misses generics, refs from non-method calls, tuples/arrays and
    non-primitive paths. A clean clippy run does not prove zero redundant
    annotations; review still matters.
- **Widen, don't narrow.** Use `usize::from(x)` / `u64::from(x)` instead of
  narrowing `as` casts, especially when comparing mixed widths:
  `packages.len() < usize::from(minimum)`, not
  `(packages.len() as u8) < minimum`. When a narrowed value must be
  produced, use `u8::try_from(x)` with a real error (see hard rule 1).
- **Take owned values** when the function will consume the data into new
  owned structures anyway. Prefer
  `group_items(items: impl IntoIterator<Item = T>)` over `&[T]` plus a
  `.clone()` at each use site.
- **Say "this value, unless it is empty" with the base layer's `NonEmpty`
  extension trait**: `opt.non_empty()`, not `opt.filter(|x| !x.is_empty())`,
  `Some(x).filter(..)` or `if x.is_empty() { None } else { Some(x) }`.
  `.filter(NonEmpty::is_not_empty)` is the iterator form.
  - It covers strings, paths, slices, the std collections and ranges, and an
    `Option` of one of those (flattened: `Some(None)` is `None`). For a
    `json::Value` only `Null` is empty.
  - A plain `!x.is_empty()` condition stays as it is: the trait is for a
    value that continues as an `Option`.
  - Import it in `lib.rs` with the other workspace imports
    (`use ext::NonEmpty as _;`).
- **Consider `itertools`** where it meaningfully simplifies iterator-heavy
  code (grouping, chunking, sorting, dedup). It's a judgment call. Verify
  semantics when swapping in a combinator: `chunk_by` is not equivalent to
  a hand-rolled greedy-pairwise algorithm.
- **Prefer idiomatic over C-shaped code.** Replace a manual loop that
  builds a `BTreeMap` with `itertools::into_group_map_by`, and a loop that
  feeds two mutable accumulators with `partition_map` / `Either`.
  Iterator-driven construction (`collect`) is also the preferred way to
  build a collection (`M-INITIAL-CAPACITY`).
- **`parking_lot::Mutex` and `RwLock`, not the `std::sync` ones.** `lock()`
  returns the guard directly with no poisoning, so there is no
  `.lock().unwrap()` and no poison-recovery boilerplate.
  - Enforce it: list both `std::sync` types under `disallowed-types` in
    `clippy.toml` and set `clippy::disallowed_types = "deny"`, so the build
    fails (under Cargo and the Bazel clippy aspect) instead of review having
    to catch it. It also catches a type named only in a signature or a
    `static`.
  - A crate-level `clippy.toml` replaces the workspace one instead of
    extending it, so a crate that ever adds one must repeat the entries.
- **Avoid `static` and thread-local state** where a consistent view matters
  (`M-AVOID-STATICS`). Duplicate crate versions each get their own copy.
  Statics used only as a performance optimization are fine.

## Paths

- **`camino::Utf8Path` / `Utf8PathBuf`, not `std::path::Path` / `PathBuf`,**
  where the paths the project handles are textual in practice (sysfs, device
  nodes, config directories). `Path`'s non-UTF-8 generality is then only
  friction.
  - Struct fields holding a path are `Utf8PathBuf`; parameters are
    `&Utf8Path` or `impl AsRef<Utf8Path>`.
  - Use `Utf8PathBuf::to_path_buf()`, not `.to_string()` or
    `.display().to_string()`, when producing an owned path for an error
    variant.
  - Walk directories with `Utf8Path::read_dir_utf8()` (yields
    `Utf8DirEntry` with `.path()` / `.file_name()` already UTF-8). Do
    **not** use `Utf8Path::read_dir()`, which just passes through std's
    un-validated `ReadDir`.
  - A non-UTF-8 path from the OS is a real (if very unlikely) error:
    `read_dir_utf8()` / `from_path_buf()` surface it as
    `io::ErrorKind::InvalidData` / `Result` / `Option`.
  - Prefer the `_utf8` variants over the std ones: `canonicalize_utf8()`,
    `read_link_utf8()`, `read_dir_utf8()`.
  - **Local override** of the `PathBuf` expectation in `M-STRONG-TYPES`: the
    principle (path-like data gets a path type, not `String`) stands, with
    `camino` as the path type.
- **`camino-tempfile`, not `tempfile` or `std::env::temp_dir()`,** for
  temporary files and directories. It mirrors `tempfile`'s API with UTF-8
  paths (`Utf8TempDir`, `NamedUtf8TempFile`, `camino_tempfile::tempdir()`),
  so a test fixture's `.path()` is already a `&Utf8Path`.
  - Don't convert `tempfile`'s `PathBuf` or `std::env::temp_dir()` with
    `Utf8PathBuf::from_path_buf(..).expect(..)` at each fixture. That
    reintroduces the conversion boundary the crate removes.
  - Other std APIs that return a `PathBuf` (`std::env::current_dir()`,
    `std::env::current_exe()`) have no `camino` replacement. Convert them
    once at the boundary with `Utf8PathBuf::from_path_buf(..)`, returning a
    real error (see hard rule 1).

## Time

- **`jiff` for dates and times, not `time` or `chrono`.**
  - `time`'s `OffsetDateTime::now_local()` fails once a process is
    multithreaded, and the usual `unwrap_or_else(now_utc)` fallback hides
    that by writing UTC as if it were local.
  - Resolve the zone once at startup with
    `jiff::tz::TimeZone::try_system()` and fail with a message if it can't
    be found. Do **not** use `TimeZone::system()`, which silently falls back
    to UTC.
- **Async code reads the runtime's clock**: `tokio::time::Instant`, not
  `std::time::Instant`. This holds in every crate that uses an async runtime,
  tests included, for a deadline, an elapsed time or a limit that the code
  computes or compares with a runtime timer.
  - Tests run on a paused clock (`#[tokio::test(start_paused = true)]`,
    `tokio::time::advance`). A `std` instant next to a runtime timer does not
    move with it, so a limit counted from `std` time never arrives in the
    test.
  - `tokio::time::Instant` is a newtype over std's. With the runtime's test
    utilities enabled, `now()` also reads the runtime's clock (one
    thread-local read and one atomic load) and falls back to std's outside a
    runtime.
  - The rule is not universal. Synchronous crates have no runtime and stay on
    `std`. Wall-clock timestamps stay `jiff` and must not decide an age or a
    timeout that a test has to advance.
  - A test that waits for a real child process or socket needs the real clock
    and says so with `// STD-INSTANT: <reason>` on the `use` line.
  - Enforce it with a check that CI runs. A text scan is enough; an aliased
    `use std::time as t;` goes unseen.

## Documentation

- **Public items use the canonical doc sections** (`M-CANONICAL-DOCS`): a
  summary sentence, then `# Examples`, `# Errors`, `# Panics`, `# Safety`
  and `# Abort` as applicable. Explain parameters in prose, never in a
  parameter table.
- **First sentence is one line, around 15 words** (`M-FIRST-DOC-SENTENCE`).
- **Public modules have `//!` docs** (`M-MODULE-DOCS`) covering what the
  module contains, when to use it, examples, side effects and relevant
  implementation details.
- **Document the end state, not the design journey**
  (`M-NO-META-DESIGN-DOCUMENTATION`), when design documents live in the same
  repository: crate and module docs then don't carry "why we picked X over Y"
  essays, design journals or self-reports of which guidelines were followed,
  and link to the design document instead. Without such documents, use
  common sense and keep the rationale that future readers need next to the
  code.
- **Inline re-exports in rustdoc** (`M-DOC-INLINE`): `#[doc(inline)]` on
  `pub use` of your own crate's items, but not on std or third-party items.
- **Design for agents** (`M-DESIGN-FOR-AI`): idiomatic APIs, thorough docs
  and examples, strong types, testable APIs and real coverage of observable
  behavior.

## Lints and static verification

- **Zero warnings.** `warn` means "must be fixed before done", the same
  spirit as `deny`. Don't defer a firing `warn` lint.
- **Enable the lint categories with an explicit priority**
  (`M-STATIC-VERIFICATION`). This also avoids `clippy::lint_groups_priority`
  when a group and its member lints are both configured. `clippy::pedantic`
  is deliberately not in the mandatory set: it is too aggressive to enforce.
  Run it now and then as guidance, and enable individual pedantic lints that
  prove their worth.

  ```toml
  [workspace.lints.rust]
  ambiguous_negative_literals = "warn"
  missing_debug_implementations = "warn"
  redundant_imports = "warn"
  redundant_lifetimes = "warn"
  trivial_numeric_casts = "warn"
  unsafe_op_in_unsafe_fn = "warn"
  unused_lifetimes = "warn"

  [workspace.lints.clippy]
  cargo = { level = "warn", priority = -1 }
  complexity = { level = "warn", priority = -1 }
  correctness = { level = "warn", priority = -1 }
  perf = { level = "warn", priority = -1 }
  style = { level = "warn", priority = -1 }
  suspicious = { level = "warn", priority = -1 }

  # Restriction-group lints worth enabling individually.
  allow_attributes_without_reason = "warn"
  assertions_on_result_states = "warn"
  clone_on_ref_ptr = "warn"
  redundant_type_annotations = "warn"
  semicolon_outside_block = "warn"
  undocumented_unsafe_blocks = "warn"
  unnecessary_safety_comment = "warn"
  unused_result_ok = "warn"
  too_long_first_doc_paragraph = "warn"
  # ... see M-STATIC-VERIFICATION for the full list.
  ```

  Opt out of an individual lint case by case (numeric-cast lints are the
  usual candidates), with a reason.
- **Workspace lint levels are deliberate.** Don't change a level, such as
  `unsafe_op_in_unsafe_fn` from `warn` to `deny`, without checking why it
  was set.
- **Also run** `cargo-audit` (dependency vulnerabilities), `cargo-hack`
  (all feature combinations build), `cargo-udeps` (unused dependencies) and
  Miri (unsafe code), and treat them as check-in gates.

## Testing

- **Embed test fixtures at compile time** (`include_bytes!` /
  `include_str!`); never read them through `CARGO_MANIFEST_DIR` at runtime.
  - Under rules_rust, `CARGO_MANIFEST_DIR` is the compile sandbox's path,
    which no longer exists when `rust_test` runs, and `compile_data` doesn't
    reach the test's runtime. A runtime read passes under Cargo and fails
    with `NotFound` under Bazel.
  - Parse the embedded bytes instead (`Parser::parse(BYTES)`).
  - A test that really needs a file on disk needs `rust_test` `data` plus a
    runfiles path.
- **Tests don't assert ground truth** (`M-TAUTOLOGICAL-TESTS`). A test that
  restates a constant or mirrors the implementation's branches passes by
  construction. Test a property (spacing, monotonicity, round trips).
- **Public-API tests are integration tests** (`M-INTEGRATION-TESTS`): put
  tests that touch only the public surface under `tests/`, and prefer an
  integration test where either kind would do. Unit tests of private logic
  stay in `#[cfg(test)] mod tests`.
- **Consider making I/O and system calls mockable**
  (`M-MOCKABLE-SYSCALLS`) where it is relevant. It is good practice but
  often hard to achieve, so it is not mandatory. For anything
  non-deterministic or environment-dependent (files, network, clocks,
  entropy, hardware), pass in a mockable core or provide a
  `new_mocked() -> (Self, MockCtrl)` constructor, and avoid ad-hoc I/O in
  library code where that is practical.
- **Test utilities are feature gated** (`M-TEST-UTIL`): mocks, sensitive
  inspection and safety-check overrides sit behind a single `test-util`
  feature.
  - **Local override for non-trivial utilities.** Test helpers that are
    substantial and public (a fake node, a mock server) go in a separate
    `<name>-test` crate instead, which only tests depend on. It avoids a
    feature that must be switched on for tests (and a second Bazel library
    target for it), and several crates' tests can share it. The feature stays
    right for small mocks and overrides on a type that production code
    already uses.

## Logging

- **Production code uses telemetry, not `println!` / `dbg!`**
  (`M-LOG-NOT-PRINT`). Console output is reserved for CLIs whose interface
  is stdout.
- **Structured events with message templates** (`M-LOG-STRUCTURED`): named
  properties (`file.path = ..`), a `{{file.path}}` template instead of
  `format!`, hierarchical event names (`file.open.success`), OpenTelemetry
  attribute names where they exist, and no sensitive data (redact it).
- **Telemetry stays cheap** (`M-LOG-OVERHEAD`): none in hot inner loops, and
  no allocating `format!` in the events you do emit.

## Architecture and unsafe boundaries

- **`-sys` crates vs. safe wrappers** *(conditional: FFI)*.
  - A `-sys` crate mirrors the raw C ABI; its public API is `unsafe fn`s by
    design. Higher-level crates must not call it directly. Name crates by
    role: `-sys` to call into an existing C library, `-ffi` to be called from
    C (`M-FFI-NAMING`).
  - Once `unsafe` call sites accumulate outside the `-sys` crate, add a
    same-named-minus-`-sys` safe wrapper crate (as with
    `libgit2-sys`/`git2`, `openssl-sys`/`openssl`) rather than spreading
    `unsafe` blocks and their `SAFETY:` comments across every caller.
  - If the wrapper's functions take a file, take `&std::fs::File` when
    callers already hold one, not `BorrowedFd` / `RawFd`.
  - Return a plain `nix::Result<T>` / `io` result, not a bespoke error
    enum, so callers can still match on specific `Errno` values (e.g.
    `EBUSY`).
  - The wrapper depends only on its `-sys` crate and re-exports, through a
    curated `pub use` list (never a wildcard), the plain data types and
    constants the crates above it need. The `-sys` crate is therefore a
    dependency of the wrapper alone.
  - This covers one kind of `unsafe` (an ioctl / fd boundary). Other kinds,
    such as `ptr::read_unaligned` on a packed struct, have their own
    reasoning and are not covered automatically.
- **The compiler enforces where `unsafe` may live** *(conditional: FFI)*.
  - Set `unsafe_code = "deny"` in the workspace lints (and in any build-system
    lint config that mirrors them), at a priority that no lint group can
    lower, so a new `unsafe` block, `unsafe fn` or `unsafe impl` anywhere is a
    compile error.
  - The crates that may use it (the `-sys` crates, their safe wrappers, an
    FFI bridge whose macro expansion is `unsafe`) opt in with
    `#[expect(unsafe_code)]`, on the narrowest item that needs it where that
    is practical.
  - It is `expect`, not `allow`, so a crate that stops needing `unsafe` gets
    an unfulfilled-expectation warning, and `deny`, not `forbid`, so the
    opt-in is possible.
  - A check that CI runs fails on any other crate that allows, expects or
    warns on `unsafe_code`, at any level, and on a crate whose `Cargo.toml`
    doesn't inherit the workspace lints. Adding `unsafe` to a new crate means
    changing that allowlist, which is a review conversation.
- **No C-ABI types above the safe wrapper's API surface** *(conditional:
  FFI)*.
  - `std::os::raw::c_char` / `c_void` and `libc` / `nix::libc` types stay
    inside the wrapper and `-sys` crates.
  - When generated fields are `c_char`, the wrapper exposes a converting
    helper (`c_chars_as_bytes(&[c_char]) -> Vec<u8>`) that callers use
    without naming `c_char` (inference supplies the type). This avoids call
    sites mixing `std::os::raw::c_char` with `libc::c_char`.
  - Exception: test code building synthetic raw-struct fixtures may refer
    to `std::os::raw::c_char` fully qualified inline, with no `use`, so it
    stays visibly scoped to the test module.
- **Business logic lives in core crates; FFI only translates**
  *(conditional: FFI, `M-FFI-TRANSLATES`)*. Keep interop concerns
  (`#[repr(C)]` layouts, pointers) out of the core crate, even if signatures
  are repeated in the `-ffi` crate.
- **Isolate DLL state** *(conditional: several Rust DLLs in one process,
  `M-ISOLATE-DLL-STATE`)*. Share only portable `#[repr(C)]` data with no
  statics, thread-locals or `TypeId`; never `String`, `Vec`, `Box` or
  runtime state across DLLs.
- **Native handle wrappers offer `unsafe` escape hatches**
  *(conditional: interop, `M-ESCAPE-HATCHES`)*: `unsafe from_native`,
  `into_native`, `to_native`, with the safety requirements documented.
- **Native `-sys` crates should build with no external tools**
  *(conditional: published `-sys` crates wrapping a native library,
  `M-SYS-CRATES`)*: build from `build.rs` through `cc`, embed verifiable
  upstream source, pre-generate bindings. A `-sys` crate generated from
  in-tree headers is exempt.

## Bazel *(conditional: rules_rust)*

- **Set `lint_config` on every hand-written `rust_library` / `rust_binary`
  / `rust_test` target**, pointing at a shared `rust_lint_config` target.
  - This is the Bazel equivalent of `[lints] workspace = true`. Without it,
    `bazel build` and the clippy aspect silently skip the workspace lint
    levels.
  - Keep the `rust_lint_config` target in sync **by hand** with the root
    `Cargo.toml` `[workspace.lints]` table. Bazel can't read it directly.
  - Don't add `lint_config` to generated crates (e.g.
    `rust_bindgen_library`). They have no Cargo package or `[lints]` table.
- **The clippy aspect reads `clippy.toml` from
  `--@rules_rust//rust/settings:clippy.toml`**, which is set repo-wide in
  `.bazelrc`. It does not find the file automatically. If `clippy.toml`
  moves or another is added, update that flag, or `allow-unwrap-in-tests` /
  `allow-expect-in-tests` silently stop applying under Bazel while still
  working under `cargo clippy`.
- **List `BUILD.bazel` `srcs` explicitly: add a new `.rs` file to its
  crate's `srcs` in the same change.**
  - Cargo finds modules through `mod` declarations, so Cargo builds pass
    without it. Only Bazel fails, with E0583 "file not found for module".
  - When adding a file, grep the crate's `BUILD.bazel` for a sibling file's
    name and add the new one next to it.
- **Cargo passing is not Bazel passing.** Run both before declaring a
  change done.

## Ports and parity *(conditional: replacing an existing implementation)*

- **Ported algorithms stay ports, not redesigns.** When a module's doc
  comment says it is a port of an original unchanged, a cleanup pass may
  change *expression* (index loops to `Peekable` / iterator adapters,
  splitting a function into named phases) but must preserve *exact
  behavior*. Verify every existing test still passes with identical
  results, and confirm any combinator you reach for is semantically
  equivalent. This is compatible with `M-RUST-SHAPED`: the domain logic
  carries over, the language constructs do not.
- **The original is the parity oracle.** A port is right when it agrees
  with the implementation it replaces, byte for byte or text for text.
  - Test against the original, not against what the new code happens to
    produce. Use a differential test that runs both over a corpus and
    compares the output, and check in golden files captured from the
    original.
  - Link the original into the differential test only, never into shipped
    code.
  - Know which runs actually exercise the oracle. If the differential test
    runs only under one build system, a pass under the other says nothing
    about the corpus.
  - A disagreement is a bug in the port unless the difference is intended.
    An intended difference is written down where the comparison is (a
    normalization in the golden-file comparison, a note in the design doc),
    never absorbed by loosening the test.
  - The oracle moves. After syncing with upstream, re-run the differential
    test against the new version.
- **Mark copied upstream quirks `// WATCH-UPSTREAM: <file>, <what>`.**
  - Use it where a port must reproduce behavior that looks accidental (a
    missing enum name, a header field the writer never reads, an offset
    computed in the wrong order) because the oracle has it and the output
    must match.
  - Name the upstream file and what to do when upstream changes it
    (`add FLOAT32 here when upstream does`).
  - Use the exact marker so `grep -rn WATCH-UPSTREAM` finds every one
    during an upstream sync.
  - Leave it off behavior that is a design decision rather than a quirk.

## Performance *(conditional: performance- or cost-sensitive code)*

- **Identify, profile and optimize the hot path early** (`M-HOTPATH`):
  benchmarks (`criterion` or `divan`), regular CPU and allocation profiling
  with `debug = 1` under `[profile.bench]`, and a note of the sensitive
  areas.
- **Optimize for throughput** (`M-THROUGHPUT`): batch work, partition ahead
  of time, sleep when idle, no hot spinning, no per-item work stealing.
- **Long-running tasks yield** (`M-YIELD-POINTS`): CPU-bound async loops call
  `yield_now().await` every 10-100 µs of work.
- **Hot futures track their size** (`M-ASYNC-STACK-SIZE`): assert
  `size_of_val` in a test, and shrink them by returning `impl Future`,
  reducing what is held across `.await`, and extracting setup.
- **Avoid nested heap indirection** in hot types (`M-AVOID-INDIRECTION`):
  don't reflexively `Arc` nested structures; lift hot fields next to the
  access.
- **Use `Box<[T]>` / `Box<str>` / `Arc<str>`** for frequently created,
  immutable sequences (`M-BOX-DST`), and **`shrink_to_fit`** on large
  long-lived collections built without an exact reservation
  (`M-SHRINK-TO-FIT`).
- **Create collections with capacity** when the size is known
  (`M-INITIAL-CAPACITY`), or build them with `collect`.
- **Use a fast hasher** (`foldhash`, `FxHash`) for trusted, internal keys
  (`M-FAST-HASHER`).
- **Reuse allocations** where it matters (`M-MEM-REUSE`), subject to the
  local override in hard rule 4.

## Applications *(conditional: binaries)*

- **Applications may use an application-level error crate** (`anyhow`,
  `eyre`, `ohno::AppError`) instead of defining error types
  (`M-APP-ERROR`). Pick one and don't mix them. Libraries used by more than
  one crate follow the error rules above instead.
- **Binaries are thin, and `main` returns the application error type.** A
  binary crate holds process setup and little else, and what is reusable
  lives in library crates that are tested through their public API. Process
  setup is the binary's own business: the runtime (`#[tokio::main]`), the
  log subscriber, command-line and config handling, signal handling and
  wiring the libraries together. It does not get a library crate of its own
  just to make `main` shorter.
  - `main` returns `anyhow::Result<()>`. A versatile CLI with rich
    diagnostics returns `miette::Result<()>` instead. Never both in one
    binary.
  - Libraries keep `thiserror` errors; the application error type appears
    only at the `main` boundary.
  - Code with a reuse or test boundary of its own (a server front-end, an
    inventory scan) goes in library crates, one per concern.
- **A command's JSON output is the typed data; the response as received is
  a separate, explicit mode.** `--output json` serializes the value the
  program decoded: the same facts as the table, in JSON notation, with the
  field order of the type, and nothing the program did not decode. A person
  who wants to see exactly what the service sent (debugging, the extra
  fields of a newer server) gets `--output raw`, which writes the body byte
  for byte, without reformatting and without a newline of its own. Do not make
  `json` a pretty-printed parsed `serde_json::Value`: it is not the typed
  data, and it is not the bytes either, since the sender's key order,
  spacing and number spelling are already gone.
- **Try `mimalloc` as the global allocator as an experiment during final
  tests and benchmarks** (`M-MIMALLOC-APPS`). In many applications the
  allocator is not a bottleneck, so it is a nice-to-have, adopted only when
  the measurements justify it.
- **Compile against the highest `target-cpu` the deployment is guaranteed to
  support** (`M-TARGET-CPU`), e.g. `x86-64-v3` for server applications. Don't
  raise it for binaries that run on hosts you don't control.

## Libraries *(conditional: published or shared crates)*

- **Features are additive** (`M-FEATURES-ADDITIVE`): any combination works,
  enabling a feature never removes or changes a public item (enums are
  `#[non_exhaustive]`), and use `std`, not `no-std`.
- **Crates work out of the box** (`M-OOBE`): they build on all Tier 1
  platforms with only `cargo` and `rust`, with no required external tools or
  environment variables. Generated sources are checked in or shipped in the
  package.

- **A client library offers typed calls and one way to get the body as
  received, not a raw twin of every call.** The one way is a single method
  that takes an endpoint selector (an enum) and returns the body as bytes
  with the same status and error handling as the typed calls. A set of
  `xxx_raw()` methods returning `serde_json::Value` doubles the API for a
  result that is neither the data nor the bytes, and a caller that wants
  JSON from the typed result calls `serde_json::to_value` on it (public wire
  types are `Serialize`). This holds for an API both sides of which we write
  too: the versions on the two sides still differ, so the typed calls ignore
  what they do not know and the bytes show it.

## Macros *(conditional: the project defines macros)*

- **Macros are a last resort** (`M-MACRO-LAST-RESORT`), and **macros by
  example beat proc macros** where they suffice (`M-EXAMPLE-OVER-PROC`).
- **Macros don't lie about signatures** (`M-MACROS-DONT-LIE`): no changing
  the nature of items, no altering signatures, no converting `async`-ness.
- **Proc macros don't define implied or hidden items**
  (`M-PROC-IMPLIED-ITEMS`), apart from the same-name function/type
  namespace trick.
- **Proc macros are a thin shim over an `_impl` crate with the tests**
  (`M-PROC-IMPL`), using snapshot tests and `trybuild` UI tests.
- **Macros assume the main crate** (`M-MACRO-MAIN-CRATE`) and emit its
  paths; they don't support renamed imports.
- **Third-party items used in expansions come from a hidden `_private`
  module** (`M-MACRO-HELPERS`).
- **Pin companion proc-macro crates to exact versions** (`=x.y.z`) and publish
  them together (`M-MACRO-VERSION-PIN`).

## Maintaining this document

Add new conventions here rather than leaving them in chat or commit
messages. Where a rule came from an incident, keep the reason next to it
so the rule isn't removed as arbitrary. Re-check the upstream guidelines
from time to time; when a rule here is a **Local override**, say what it
overrides so the divergence stays visible.
