# kuiper specific tips
Note: most of these have been integrated into the skills and agent description, just keeping them for reference

1. **Use the project-local Makefile for everything.** Do not call `nvcc`, `fstar.exe`, or `krml` directly. The Makefile is the source of truth for how things are built, and we want it to work for other people too. Exact Makefile targets will depend on the project, and you should defer to project-local instructions for how to use it. A Makefile in a Kuiper project will provide a way to verify a single file and extract a single file. You should always use these when you want to verify/extract a module because it will verify and cache the dependency tree properly. The two exceptions are a) while iterating, you should use the F* MCP server instead, and b) if you need to verify a file with a specific flag (like "--print_z3_statistics") while iterating, you can use `fstar.sh` at the root of the repo. 

5. **No `assume pure`, `magic()`, or `admit()`.** These are holes in the proof — `assume pure` can hide memory safety bugs, `magic()` makes code non-executable (it's `undefined` at runtime), and `admit()` skips the proof entirely. If Z3 can't prove nonlinear arithmetic, restructure the code or add lemmas — do NOT paper over it with assumes. A kernel with assumes is NOT verified.

6. **Use `--error_contexts true` for confusing error locations.** If F\* reports an error pointing at a weird location (like `Prims.fst`), re-run with `./fstar.sh --error_contexts true path/to/File.fst` to see which source expression actually caused the problem.

7. **Mark type abbreviations as `inline_for_extraction noextract`** (not `unfold`). When defining a type alias for function signatures (e.g., `type smul_ty (t:Type0) = ...`), use `inline_for_extraction noextract` so Karamel inlines the type during extraction. Otherwise Karamel may generate function-pointer return types in the C code.

### Be faithful to CUDA source

When asked to translate CUDA code to Kuiper, be sure to cross-check the extracted CUDA source from the Kuiper implementation against the reference CUDA.
Spawn a sub-agent to scrutinize your implementation against the reference and ensure no shortcuts have been taken, especially those that simplify proof
obligations in Kuiper at the cost of performance. Even the small details matter a lot in CUDA.

## When Writing Kuiper Code

- The main `Kuiper` module re-exports core types and combinators — most files just `open Kuiper`
1. Use inline_for_extraction and noextract attributes appropriately
   - Any record type that is not marked inline_for_extraction will attempt to be extracted as a C struct. Oftentimes,
   it is desirable to put a group of arguments that is commonly passed around in functions into a record; make sure that
   this record is `inline_for_extraction noextract` otherwise it will be extracted as a struct in the CUDA which will potentially add overhead.
   - Concrete functions must be `inline_for_extraction noextract`; only top-level kernels are non-inlined (with `__global__`)
   - Typeclass instances used concretely (e.g., `clayout`) must also be `inline_for_extraction noextract`   
2. **Do not put an implicit argument (`#x` or `{| |}`) in the LAST position of a signature** — F* often fails to instantiate trailing implicits at a call site. Make the final parameter explicit (e.g. a `squash`/unit witness passed as `()`). Trailing implicits are only safe when a later explicit arg forces their inference.
3. Do not hardcode tensor layouts when writing kernels unless it is an absolute requirement (such as when using tensor cores). Layouts in kernel parameters should stay generic. 
4. Use `pts_to_len array` to establish array length facts needed for indexing
9. Use `gpu_pts_to_slice_ref` to extract `Seq.length` facts from GPU array predicates instead of refining erased sequences; export the `pure` fact out of `map_loc`
10. Use `Pulse.Lib.Vec.lvec f32 n` (length-indexed vec) and `pts_to_len` to derive sequence length facts from host vec predicates
5. Keep the imports of a module clean; don't open modules you don't need. This will only increase verification time and compromise parallelism in the project build by creating unnecessary serial dependencies.
6. **IMPORTANT: AVOID RE-PROVING EXISTING LEMMAS, OR TRIVIAL GOALS SMT CAN HANDLE**. Don't blindly add lemmas and intermediate asserts to your code. Many lemmas can be automatically discharged by SMT (ask yourself if you really need to write the lemma if the proof for it is just `()`! that means it is probably already solvable from the context you're using it in). Others may exist already in the Kuiper library and you should avoid re-proving these. If a lemma is genuinely needed, and is reusable for other kernels and would belong better in the Kuiper library, then if you are working in the main Kuiper tree, you should just modify the library directly. Otherwise, write a comment with a TODO flagging that it should be upstreamed. Extra `assert`s in the code could actually harm Z3's performance by polluting the context, so be careful and pragmatic with your asserts.
7. Avoid `erased (natlt z)` — instead write `n:(erased nat){n < z}` or use `enatlt` (erased is invariant w.r.t. types, causing brittle typechecking)
8. Coalesce multiple `map_loc` calls into one when lifting/lowering several predicates at the same location
11. Do NOT call `on_star_intro`/`on_star_elim` manually — the Pulse checker applies them automatically

### Setup and Teardown

The `setup` and `teardown` functions establish preconditions and postconditions for the kernel's ghost state:

**Setup pattern**:
1. Share matrices among threads with `gpu_matrix_share_threads`
2. Tile matrices with `gpu_matrix_tile`
3. Combine predicates with `forevery_zip_2`
4. Return combined precondition suitable for per-thread work

**Teardown pattern** (roughly symmetric to setup):
1. Unfold precondition to expose structure
2. Unfactor permissions with `forevery_unfactor'`
3. Unzip predicates with `forevery_unzip`
4. Gather matrices with `gpu_matrix_gather_n`
5. Untile with `gpu_matrix_untile`
6. Return original postcondition

## Debugging Verification Errors

1. Analyze the F* error message to understand what property failed
2. Trace through the code to find the mismatch between implementation and specification
3. Adjust either the implementation or the assertions to resolve the discrepancy
4. Common issues: missing synchronization, incorrect postconditions, type mismatches
5. Use proof tactics and lemmas from Kuiper.* modules as needed

**Additional debugging tips**:
- **Check the loop condition first**: If a loop-related proof fails, verify the loop invariant is compatible with both entry and exit conditions
- **Use assert-before-assume**: Try `assert pure` before falling back to `assume pure` - sometimes the assertion will fail in a way that reveals the real issue
- **Check type inference**: Look for `__y<number>` in error messages - these are escaped unification variables and indicate the SMT solver lost context
- **Simplify incrementally**: If a proof fails, remove non-essential invariants and re-add them one at a time
- **Ask for help**: If a proof seems too difficult or gets stuck, ask the user for guidance rather than spending excessive time trying to force it

## Extraction Quality (avoiding structs/tuples in generated CUDA)

Concrete index tuples (e.g. `conc (batch @| rows @| cols @| INil)` values like `(page, (grow, (gcol, ())))`) and other tuple-typed `let` bindings that survive to extraction get emitted as C `struct`s in the generated CUDA, which is slow and ugly. To keep the generated code flat:

- **Repeat the literal tuple at the call site instead of binding it.** A `let ci = (page, (grow, (gcol, ())))` passed to a function does NOT inline — Karamel keeps `ci` as a struct. Pass the literal `(page, (grow, (gcol, ())))` directly to each consumer. If you still need `ci` for proof/rewriting, keep the binding but add `assert rewrites_to ci (page, (grow, (gcol, ())))` so both the readable name and the inlined literal coexist.
- **Use `[@@inline_let]` on tuple-typed `let`s inside layout lambdas** (e.g. the `cimap` in `c_subtile_layout`), so the reindexed tuple inlines rather than extracting as a struct.
- **Make ghost index values `erased`.** Index bindings used only in specs/proofs (e.g. `let bidn : natlt (...) = SZ.v bid`) should be `let bidn : erased (natlt (...)) = SZ.v bid` — otherwise they extract as concrete tuples/values.
- **Erase spec-only `nat`/index *function arguments*.** A `nat`/`natlt` parameter that is used only to compute a ghost layout/spec (not needed at runtime) must be declared `erased`, or extraction fails. Pair it with a `{| concrete_sz page |}` typeclass instance to recover the runtime value where genuinely needed, rather than taking a concrete `nat` parameter. Example: `Kuiper.Array2.Strided.Slice.slice_of_3` takes `(page : erased (natlt batch)) {| concrete_sz page |}` — an earlier `(page : natlt batch)` broke extraction. (This is the one place `erased (natlt z)` is the right pattern despite the general "avoid `erased (natlt z)`" guidance, because the arg must be erased for extraction.)
- **Mark concrete helpers called from device code `inline_for_extraction noextract`.** Any top-level concrete function invoked from within a kernel (device) — e.g. a small `szp`/index bridge like `nthr_to_prod_sz` — must be `inline_for_extraction noextract`. Otherwise extraction forces you to annotate it as a device or host function; for trivial/identity bridges it is easier to just inline it.
- After changing extraction attributes, re-verify the module AND regenerate the `.cu` (`make obj/<Module_With_Dots_As_Underscores>.cu`) to confirm the structs are gone.

## File Organization

- **Do not modify unrelated files**: Changes to other files make the build slower and can affect build success
- Keep changes localized to the functions you're working on

## When Uncertain

- Ask clarifying questions about requirements (kernel dimensions, data layout, synchronization needs)
- Request examples of expected behavior
- Ask about acceptable performance trade-offs
- Request guidance on verification complexity tolerance
