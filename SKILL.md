---
name: google-rust-style
description: Apply Google's Rust Style Guide when writing, reviewing, refactoring, or debugging Rust code. Use for Rust implementation tasks, API design, code review, and style cleanup.
---

# Google Rust Style

Apply Google's Rust style guidance as an engineering standard for readability, maintainability, API quality, and refactorability.

Primary source: https://google.github.io/styleguide/rustguide.html

This skill is an operational summary, not a verbatim copy. The repository's explicit conventions and platform requirements take precedence where they intentionally differ. Do not apply rules marked Chrome, Chromium, or Fuchsia outside those environments.

## 1. Operating procedure

When working on Rust code:

1. Inspect the repository before changing code:
   - Read `Cargo.toml`, workspace manifests, `rust-toolchain.toml`, `rustfmt.toml` / `.rustfmt.toml`, and relevant module files.
   - Identify whether the target is library code or application code.
   - Identify the supported Rust toolchain, target platform, feature flags, and existing test/lint commands.
   - Follow existing project conventions when they are deliberate and consistent.
2. Make the smallest change that solves the requested problem. Prefer direct, understandable code over speculative abstractions.
3. Apply the design and implementation rules below.
4. Format and validate using the repository's actual toolchain and commands.
5. Review the diff for correctness, unnecessary churn, API compatibility, and accidental platform-specific assumptions.
6. Report commands run and any validation that could not be completed. Never claim a check passed unless it was run successfully.

Do not reformat unrelated files or make broad style-only changes unless requested.

## 2. Priorities and scope

Use this priority order:

1. User request and explicit repository requirements.
2. Correctness, safety, and compatibility with the project's supported toolchain/platform.
3. This Google Rust Style skill.
4. General Rust ecosystem conventions.

Distinguish API guidance by code type:

- **Library code:** Apply API-design recommendations strongly. Prefer flexible, composable, stable public interfaces.
- **Application code:** Treat those recommendations as strong guidance, not an excuse for abstraction. Prefer simplicity and directness when usage is known.
- **Platform-specific code:** Apply Chrome/Chromium/Fuchsia rules only when the repository is actually targeting that environment or explicitly adopts those rules.

The guide's goals are readability and maintainability over writeability, automation-friendly consistency, ecosystem consistency, and rules with tangible benefits.

## 3. Formatting and imports

- Use the repository's configured formatter. Normally run `cargo fmt --all -- --check` to verify formatting and `cargo fmt --all` only when formatting changes are intended.
- Do not use `#[rustfmt::skip]` for personal preferences. Reserve it for justified manual alignment or generated code.
- Respect project rustfmt settings. Google documents a 100-column preference, but do not silently impose it if the project/toolchain does not configure it.
- Avoid glob imports. Exceptions may include `use super::*` in inline test modules, narrowly scoped enum imports, and deliberate re-exports in non-leaf modules.
- Keep each `use` statement grouped by a single module; avoid nested paths inside a brace import.
- Import traits inside a function when their methods are only used locally. Keep them at module scope when used broadly.
- Qualify calls to top-level functions from other modules with the module name, e.g. `mem::replace(...)`.
- Import type names and use them unqualified by default. Use qualified paths or aliases to resolve ambiguity or avoid an excessive import block.
- Avoid defining prelude modules unless the codebase has a well-established reason.

## 4. Crates, modules, and file organization

- Prefer small crates with a clear responsibility and focused abstraction.
- Split crates when independent APIs or dependency/build boundaries justify it; do not split mechanically.
- Name a leaf module and its source file after its primary user-defined type or abstraction where practical (`widget.rs` for `Widget`).
- Keep non-leaf modules primarily structural: module declarations and re-exports. Avoid placing substantial API implementations there.
- Re-export public APIs from the crate root when that makes internal submodules an implementation detail.
- Do not apply Chromium's flat `rust_library` directory-layout rule to ordinary Cargo projects. Follow the repository's Cargo layout unless it explicitly adopts Chromium's build conventions.

## 5. Ownership, borrowing, and function signatures

- Prefer ownership and borrowing designs that avoid unnecessary shared ownership.
- Avoid `Rc` and `Arc` unless shared ownership materially simplifies the design or is required by the architecture. Do not replace a sound shared-ownership design merely to satisfy a slogan.
- Prefer borrowing over deep cloning. Use `.clone()` when an owned copy is genuinely required or when borrowing complexity would make the code substantially harder to understand.
- For immutable inputs, accept flexible references to the data rather than references to a particular wrapper:
  - `&T` rather than `&Box<T>`
  - `&[T]` rather than `&Vec<T>`
  - `&str` rather than `&String`
  - `Option<&T>` rather than `&Option<T>`
- For reusable library APIs:
  - Prefer returning an iterator (`impl Iterator` or a concrete iterator) over allocating and returning `Vec<T>` when callers benefit from choosing how to consume results.
  - Prefer `impl IntoIterator` when the function only needs to iterate and broad input flexibility is useful.
  - Consider `impl Into<T>` when call-site ergonomics justify the generic/API complexity.
- For application code, slices and owned `Vec<T>` returns are often clearer when caller usage is known. Do not add generic bounds or iterator abstractions without a concrete benefit.
- Prefer an enum over an unexplained boolean parameter when the boolean's meaning is not obvious at the call site.
- Keep function signatures focused. Avoid adding convenience generics that cause code bloat, inference problems, or needless complexity.

## 6. Errors, panics, and invariants

Choose the error mechanism according to the contract:

- Use `Option<T>` when absence/failure is expected and the reason is obvious or not useful to the caller.
- Use `Result<T, E>` for recoverable failures where callers need diagnostic context or need to choose a recovery path (I/O, parsing with meaningful diagnostics, network, external services).
- Use `panic!` for programming bugs and violated internal invariants, not routine external failures. Respect project policies that prohibit panics.
- Avoid check-then-`unwrap` patterns. Prefer `if let`, `match`, or `let ... else` so control flow proves the value is present.
- If a panic is genuinely necessary because an invariant cannot be expressed in the type system, prefer `expect("specific invariant")` over an unexplained `unwrap()`.
- Add useful context to assertions when it explains why the condition should hold, identifies misuse, or reports relevant values. Do not merely repeat the condition.
- Never access the contents of a poisoned mutex by recovering its inner value.
- Do not use `std::panic::catch_unwind` as ordinary error handling. Google's first-party guidance prohibits it, with only a narrow defense-in-depth exception in unwind-enabled environments. Follow the target project's panic strategy.

## 7. Types and API design

- Use a newtype instead of a type alias when a value has distinct semantics, needs its own API, may need a different representation later, or could be confused with another value.
- Prefer named struct fields over tuples or multi-field tuple structs when the values have semantic meaning. Single-field newtypes are appropriate.
- Prefer enums over booleans for public API choices with non-obvious meaning.
- Use `#[non_exhaustive]` for public enums and public-field structs when downstream users need forward compatibility and new variants/fields may be added. Consider whether this prevents desired struct-literal / functional-record-update use.
- Avoid exposing implementation details as public API without a reason.
- Treat `#[doc(hidden)]` items owned by other crates as private; do not depend on them.
- Mark public functions `const` only when long-term const-ness is part of the intended contract; changing a public `const fn` later can be breaking.
- Do not overload operators for unrelated syntactic convenience. Use operator traits for their conventional mathematical or logical meanings.
- Implement `len()` together with `is_empty()`.

## 8. Traits and generics

- Implement standard library traits when they match the intended behavior (`FromStr`, `Display`, `Error`, comparison traits, etc.) instead of inventing similarly named methods. An inherent method may coexist when it improves discoverability.
- Prefer `#[derive(...)]` over manual implementations when the derive expresses the intended semantics.
- In libraries, prefer generic parameters / `impl Trait` over `dyn Trait` when static dispatch and monomorphization are appropriate. Use `dyn Trait` when runtime polymorphism, object storage, ABI boundaries, or code-size considerations justify it.
- Define a trait when it enables generic programming, abstraction through `impl Trait` / `dyn Trait`, or a genuine extension-trait use case. Do not create traits merely to group methods.
- Name extension traits with an `Ext` suffix (for example, `ReadExt`).
- Add trait implementations proactively in foundational library code when users can reasonably expect them; in application code, add them as needed.
- Consider conventional traits deliberately:
  - `Copy` only when cheap and semantically stable.
  - `Clone` when duplication is meaningful and safe; avoid implying that external resources can be duplicated when they cannot.
  - `Default` when a meaningful default state exists, especially for options/configuration structs.
  - `PartialEq` / `Eq` and `PartialOrd` / `Ord` only when their mathematical contracts hold. `Eq` requires reflexivity; `Ord` requires a total order.
  - `Hash` and `Eq` for types likely to be map keys or cached; add `Ord` when a clear total order exists.
- Pair important `From<T>` conversions with a discoverable inherent constructor such as `new`, `new_from_foo`, or `from_foo` when appropriate.
- Options/configuration structs with public fields should generally implement `Default`.

## 9. Naming and accessors

- Follow standard Rust naming conventions and the standard library's method conventions.
- Use `as_...`, `to_...`, and `into_...` according to borrowing, conversion, and ownership semantics.
- Prefer descriptive lifetime names for public or complex relationships. Short lifetimes are fine for simple local relationships; use `'_` when it improves clarity.
- Choose generic parameter names related to their meaning or trait bounds. `T` and `U` are fine for broadly generic APIs.
- Avoid obscure abbreviations and deleted-letter shorthand. Use abbreviations only when broadly recognized or unambiguous in the domain.
- Constructors: use `new()`; use `new_with_foo()` when there are multiple constructors; use `try_new()` when construction can fail.
- Value-to-value conversions: use `from_foo()` alongside an appropriate `From` implementation when useful.
- Accessor naming:
  - `foo()` for shared access returning `&T` (or a `Copy` value).
  - `foo_mut()` for mutable access returning `&mut T`.
  - `take_foo()` when removing/resetting a value and returning the old value.
  - `set_foo(value)` for assignment.
  - `replace_foo(value)` when returning the previous value.
  - `with_foo(...)` / `with_mut_foo(...)` for closure-scoped access, especially around interior mutability or validation.

## 10. Documentation

- Document public APIs with `///`; use `//!` for crate or module documentation.
- Use ordinary `//` comments for implementation notes, TODOs, and non-public commentary.
- Write documentation that adds information beyond the identifier and signature. Avoid comments that simply restate what the code obviously does.
- Use Markdown in doc comments and follow rustdoc conventions.
- Include examples where they help users understand how a crate or API works; examples need not be repeated for every function.
- Document panic conditions, malformed-input behavior, and other important failure modes.
- Document safety invariants for unsafe functions and traits.
- Use capitalization and punctuation in documentation.
- Keep comments accurate when code changes. Prefer directly readable documentation over macro-generated documentation when practical.

## 11. Unsafe code and language features

- Do not introduce `unsafe` unless safe Rust cannot reasonably implement the required behavior.
- When unsafe is necessary:
  - Isolate it in a small, auditable abstraction, away from business logic.
  - Prefer a safe public interface around the unsafe implementation.
  - Add a `// SAFETY:` comment immediately adjacent to each unsafe block, explaining the invariants that make it sound.
  - Explain why a safe alternative is insufficient. If performance is the reason, provide benchmark evidence where feasible.
  - Keep dependencies and the unsafe surface area small.
- Do not use unstable Rust features unless the project explicitly requires and approves them for its toolchain/target.
- Do not arrange for application code to execute before `main()` through constructor mechanisms such as `ctor`.

## 12. Concurrency and platform-specific rules

General guidance:
- Keep synchronization and ownership policies explicit.
- Never recover a poisoned mutex's inner value as a normal path.
- Choose synchronization primitives based on the project's runtime, poisoning semantics, performance, and platform requirements.

Platform-specific Google guidance (apply only in the matching codebase):
- **Chromium:** Do not use `async` / `.await`; prefer the Chromium-approved task/sequence model over spawning standard-library threads; use Chromium's approved import mechanism and test framework.
- **Fuchsia:** Use `fuchsia_sync::Mutex` and `#[fuchsia::test]` where required by the project.
- **Chrome:** Follow Chrome-specific build, import, and test conventions only in Chrome code.
- Do not transplant these restrictions into normal Tokio, async-std, or other Cargo applications.

## 13. Macros and generated code

Prefer, in order:

1. Ordinary Rust functions, types, traits, objects, and modules.
2. `macro_rules!` for small, justified syntax-level generation.
3. Build-time code generation when it is clearer and maintainable.
4. Procedural macros when simpler approaches do not fit.

Before adding a macro, consider IDE navigation, code search, debugging, reviewability, and the difficulty of large-scale refactoring. Avoid creating a private mini-language without a compelling reason.

## 14. Implementation readability

- Add explicit type annotations when inference makes a non-trivial type difficult to understand, especially for `.into()`, `.parse()`, and complex iterator chains.
- Prefer pattern matching over checking a value and then unwrapping it.
- For `#[non_exhaustive]` enums, handle known variants deliberately and retain a wildcard arm for future variants when required.
- Prefer readable loops over long iterator chains that mix branching, mutation, and error handling.
- Avoid `Iterator::for_each` when a normal `for` loop is clearer.
- For a complex iterator pipeline used by a loop, bind it to a named local before iterating.
- Use iterator combinators when they express a short, clear operation better than a loop.
- Prefer `format!` / `write!` for concatenating more than two strings or substituting values into a literal format string.
- Do not use `panic::catch_unwind` as a substitute for correct error handling.

## 15. Testing and validation

Inspect the repository and use its configured commands. For a conventional Cargo project, a reasonable baseline is:

```bash
cargo fmt --all -- --check
cargo check --workspace --all-targets
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace
```

Adapt commands when:
- the workspace does not support all features on the current target;
- integration tests require services, credentials, devices, or environment variables;
- the project uses nextest, cargo-hack, custom test runners, or platform-specific harnesses;
- a command would trigger an expensive or destructive build/test.

Do not automatically add `#![deny(warnings)]` or new lint policies to an existing project without being asked. Use Clippy findings as actionable feedback, but do not silence lints blindly. If an allow is justified, scope it narrowly and explain why.

For unsafe code, concurrency, parsers, serialization, and public API changes, add or update tests for the relevant invariants and failure paths. Prefer deterministic tests and avoid timing-sensitive assertions where possible.

## 16. Rust code review checklist

Review the final diff and ask:

### Structure and style
- [ ] Does the change follow the repository's formatting and module conventions?
- [ ] Are imports explicit and free of unnecessary glob imports?
- [ ] Are crates/modules focused, with implementation in appropriate leaf modules?
- [ ] Is unrelated formatting churn avoided?

### Ownership and API
- [ ] Are borrows used instead of unnecessary deep clones?
- [ ] Are `Rc` / `Arc` justified by actual shared ownership?
- [ ] Are immutable parameters expressed with flexible references?
- [ ] Are iterator, slice, and owned-collection choices appropriate for library vs application code?
- [ ] Are public APIs explicit, semantically typed, and forward-compatible where needed?
- [ ] Are enums used instead of opaque boolean parameters where appropriate?

### Correctness and errors
- [ ] Do `Option`, `Result`, and panics match the API's failure contract?
- [ ] Are check-then-unwrap patterns avoided?
- [ ] Do assertions include useful context where it helps diagnosis?
- [ ] Are poisoned mutexes handled without extracting their inner values?
- [ ] Are non-exhaustive enums matched safely?

### Safety and maintainability
- [ ] Is unsafe necessary, isolated, and documented with `SAFETY:` reasoning?
- [ ] Are macros/code generation justified?
- [ ] Are public APIs documented, including panic and safety conditions?
- [ ] Are names descriptive and abbreviations understandable?
- [ ] Are iterator chains and type inference readable?
- [ ] Are platform-specific restrictions applied only to the correct target?

### Validation
- [ ] Were relevant formatting, check, lint, and test commands run?
- [ ] Are failures reported accurately rather than hidden or waived?
- [ ] Does the final diff contain only intended changes?

## 17. Response expectations

When completing a Rust task, summarize:
- what changed;
- important API, ownership, safety, or compatibility decisions;
- validation commands run and their results;
- known limitations or checks not run.

Do not claim full compliance with Google's guide when repository-specific rules or platform constraints prevented it. Explain material deviations briefly.

## Source and maintenance

This skill is based on Google's Rust Style Guide:
https://google.github.io/styleguide/rustguide.html

The upstream guide may change. For high-impact API or policy decisions, consult the current source rather than assuming this summary is exhaustive. In particular, re-check platform-specific sections before applying them.
