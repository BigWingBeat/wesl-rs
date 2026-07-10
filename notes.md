# Reviewer notes: iterative import resolution refactor

These notes accompany the refactor of `crates/wesl/src/import.rs`. The goal was to
behave identically to the previous recursive implementation while separating
responsibilities: usage analysis no longer loads modules.

## What changed

The old implementation was mutually recursive: `resolve_decl` → `resolve_ty` →
`load_module` (which invoked an `onload` callback) → `resolve_decl` again on the
external module. The `onload` callback was how `resolve_lazy` and `resolve_eager`
injected their per-module behavior into the recursion.

The new implementation has two functions with separate responsibilities:

- **`used_idents(module, entrypoints)`** — pure per-module analysis, no resolver
  involved. Starting from a list of `Entrypoint`s, it marks used local declarations
  (in `Module::used_idents`, as before) and returns the external items referenced,
  in encounter order. It is iterative: an explicit stack of "frames" (one iterator of
  type expressions per declaration being traversed) replaces the recursive descent
  into referenced local declarations.

- **`load_modules(source, path, keep, resolver)`** — the iterative loader. It runs a
  worklist of `ExtItem`s (target module path + declaration name): pop an item,
  get-or-load its module, analyze with `used_idents`, push the newly discovered
  external items.

`resolve_lazy` and `resolve_eager` are now thin wrappers that differ only in the
`keep: Option<Vec<Ident>>` argument. `keep: Some(...)` means lazy (analyze
`const_assert`s plus the keep list on the root); `None` means eager (analyze all
declarations of every loaded module).

## Design choices

**The `Entrypoint` enum.** Analysis starting points are either named declarations
(`Ident`) or unnamed ones (`Decl`, for module-scope `const_assert`s, which have no
identifier to name them by). The enum exists because eager mode must analyze
declarations in *source order*, interleaving named and unnamed ones — the old eager
`onload` iterated `global_declarations` in order, and a different order would change
the module load order (see below). A named/unnamed split into two parameters would
have lost that interleaving.

**`ExtItem` carries a `from` path.** `Diagnostic::with_module_path` only sets the
path if none is set yet, so in the recursive version the *innermost* wrap won. Tracing
the old error paths: missing/private declarations were attributed to the *target*
module, but resolver failures (file not found) were attributed to the *referencing*
module — the first enclosing `err_with_module` on the unwind path. Since the loader
processes items detached from the analysis that produced them, each item records which
module referenced it, so load failures can be attributed identically.

**`used_idents` returns `Result<_, ImportError>` instead of `Result<_, Error>`.** The
old `resolve_decl`/`resolve_ty` needed the broader `Error` because they called the
resolver. The analysis function can't fail with a resolve error anymore, and the
narrower type makes that visible in the signature.

**LIFO worklist with reverse push.** Module load order is observable: `assemble`
concatenates declarations in `Resolutions::order` (first-load order), so a plain FIFO
queue (BFS) would reorder the compiled output. Popping from the back while pushing
each batch reversed reproduces the old depth-first order: the first item encountered
is fully explored (transitively) before its siblings.

**Post-order type-expression flattening (`type_exprs`).** The old `resolve_ty`
recursed into nested type expressions *before* handling the enclosing one, so external
references were emitted innermost-first. `type_exprs` reproduces this: a pre-order
traversal with children visited right-to-left, reversed, yields the left-to-right
post-order of the old recursion. This matters for both module load order and which
error surfaces first when several are present.

**Per-entrypoint draining.** `used_idents` fully drains the frame stack after each
entrypoint rather than seeding all frames up front. The marking state
(`used_idents` set) must evolve in the same sequence as the old code: a declaration
reached from entrypoint 1 must already be marked when entrypoint 2 references it.

**Newly loaded modules analyze their own entrypoints first.** The old `load_module`
ran `onload` (const_asserts, or everything in eager mode) *before* the caller resolved
the specific item that triggered the load. The loader preserves this by prepending
`module_entrypoints` to the arriving item.

## Discoveries along the way

- **`Ident` equality is `Arc` pointer identity**, not name equality. Within a module,
  `retarget_idents()` (called on load) makes local references share the declaration's
  `Arc`, so `idents.contains_key(&ty.ident)` works by identity. Across modules,
  idents are distinct `Arc`s — that's why `find_decl`/`find_import` fall back to
  by-name search, and why cross-module work items effectively match by name. The
  refactor preserves all of these lookups verbatim.

- **The self-path special case doesn't mark usage.** When an import path resolves back
  to the module itself (`ext_path == module.path`), the old code only checked that the
  declaration *exists* — it did not mark it used or descend into it. That looks like a
  latent quirk (a declaration referenced only via a self-path could be stripped), but
  the refactor preserves it as-is, including the pointer-identity `contains_key` check
  on `ty.ident`. `retarget` has the same special case, kept untouched.

- **Cyclic `@publish` re-exports with no underlying declaration** (A re-exports from B,
  B re-exports from A) made the old code recurse infinitely (stack overflow); the new
  code spins in the worklist loop instead. Equally broken input, equally unhelpful
  outcome — deliberately not fixed here to keep the diff behavior-neutral. If worth
  fixing, the iterative shape makes cycle detection easy: track visited
  `(path, name)` pairs in the loader. Note `retarget`'s `find_ext_ident` has the same
  recursion on re-export chains.

## Known behavioral caveat

The old code explored an external reference *immediately*, suspending the current
declaration's traversal; the new code defers it until the current module batch
finishes. For acyclic module graphs the resulting first-load order is identical (the
deferred items are explored depth-first in encounter order). With *cyclic* imports —
specifically, when a later-analyzed module reaches back into a partially-analyzed
one — the load order can in principle differ, because the old interleaving attributed
a shared declaration's references to a different point in the exploration. Similarly,
in a program containing several errors, *which* error surfaces first could differ in
such cases. No test in the suite (including the wesl-testsuite conformance tests)
exercises this; in those cases the old order was essentially arbitrary anyway.

## Verification

- `cargo test --workspace --lib --bins --tests --benches`: 551 passed, 0 failed
  (including the 532-test spec/bevy/wgpu suite in `wesl-test`).
- `cargo fmt --all` and `cargo clippy --workspace --all-targets --all-features`: clean.
