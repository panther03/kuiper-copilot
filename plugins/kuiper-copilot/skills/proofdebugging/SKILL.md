---
name: proofdebugging
description: Systematic workflows for debugging F*/Pulse verification failures
tools: Bash, Read
---

## Invocation

This skill is used when:
- A proof fails and you need to find out why
- Proofs are slow or flaky
- You need to systematically isolate a verification failure

## Debugging Workflow

### Step 0: Read the Error Properly

1. Analyze the F* error message to understand which property actually failed
2. Trace through the code to find the mismatch between implementation and specification
3. Adjust either the implementation or the assertions to resolve the discrepancy

If the error points at a nonsensical location (like `Prims.fst`), re-run with
`--error_contexts true` to find the source expression that really caused it:

```bash
./fstar.sh --error_contexts true path/to/Module.fst
```

If the message mentions names like `__y17`, those are escaped unification variables: the
solver lost context, usually because of a type inference failure upstream. Add explicit
type annotations at the point where the value is introduced.

### Step 1: Identify the Failing Query

```bash
./fstar.sh --query_stats --split_queries always Module.fst 2>&1 | grep -E 'cancelled|failed|succeeded'
```

- `--query_stats` shows time and result per query
- `--split_queries always` separates each assertion into its own query
- Look for `cancelled` (timeout) or `failed` queries

### Step 2: Isolate the Failure Point

Use `admit()` as a binary search tool:

```fstar
let my_proof () : Lemma (ensures conclusion) =
  step1;
  assert (fact1);      // Does this pass?
  admit();             // Cut here — if it passes, failure is below
  step2;
  assert (fact2);      // Move admit() down to find exact failure
  step3
```

Move the `admit()` down until the proof fails again. The assertion just before
the `admit()` position is where Z3 gets stuck.

### Step 3: Factor Out the Failing Part

Extract the failing assertion into a standalone lemma:

```fstar
// Separate lemma — easier to debug in isolation
let helper_lemma (x: t)
  : Lemma (requires precondition x) (ensures failing_fact x)
  = // prove it here, with full focus

let my_proof () : Lemma (ensures conclusion) =
  step1;
  helper_lemma arg;    // Call the helper
  step2;
  step3
```

Small lemmas are easier for Z3, easier to understand, and more reusable.

### Step 4: Harden the Proof

Once it passes, reduce rlimits:

```fstar
#push-options "--z3rlimit 10 --fuel 0 --ifuel 0"
let my_proof () : Lemma (...) = ...
#pop-options
```

If it fails at low rlimit, add more intermediate assertions rather than
increasing the limit.

## Common Root Causes and Fixes

### Z3 Doesn't Know a Library Fact

**Symptom:** Assertion about sequences, sets, or arithmetic fails.

**Fix:** Call the appropriate lemma:
```fstar
// FiniteSet facts
FS.all_finite_set_facts_lemma();

// Sequence properties
Seq.lemma_eq_intro s1 s2;

// Arithmetic
FStar.Math.Lemmas.pow2_plus a b;
```

### Extensional vs Propositional Equality

**Symptom:** `s1 == s2` fails but the sequences are clearly equal.

**Fix:** Use extensional equality:
```fstar
assert (Seq.equal s1 s2);      // Compares element-by-element
// NOT: assert (s1 == s2);     // Requires decidable equality proof
```

Same for sets: use `Set.equal`, not `==`.

### Quantifier Instantiation Failure

**Symptom:** A `forall` property is known but Z3 can't use it.

**Fixes:**
1. Add a pattern (trigger):
```fstar
forall (x:t). {:pattern (f x)} P x
```

2. Instantiate manually with a lemma call:
```fstar
my_forall_lemma specific_value;
assert (P specific_value);
```

3. Make the definition opaque and reveal selectively:
```fstar
[@@"opaque_to_smt"]
let complex_def = ...

let use_it (x:t) : Lemma (P x) =
  reveal_opaque (`%complex_def) complex_def
```

### Type Refinement Not Available

**Symptom:** Z3 knows `x < 100` but can't prove `x < 200`.

**Fix:** Add explicit assertion chains:
```fstar
assert (x < 100);     // Known
assert (100 <= 200);   // Trivial
assert (x < 200);      // Now provable
```

### Large File / Slow Verification

**Symptom:** Proofs that worked in small files break in large ones.

**Fixes:**
1. Split the module — move helper lemmas to a separate `.fst` file
2. Reduce fuel/ifuel to 0 where possible
3. Use `#push-options` / `#pop-options` to scope rlimit changes
4. Make definitions `[@@"opaque_to_smt"]` when not needed by nearby proofs

### Using assert_spinoff for Query Isolation

`assert_spinoff (P)` creates a separate Z3 query for P, preventing it from
bloating the main proof context. Use when:
- A function has many assertions and Z3 is slow on all of them
- You want to prove a property without polluting the main query context
- Individual assertions are fast but the combined query times out

```fstar
let complex_proof () : Lemma (...) =
  step1;
  assert_spinoff (intermediate_fact1);  // Proven in isolation
  assert_spinoff (intermediate_fact2);  // Proven in isolation
  // Main proof continues with both facts available but Z3 didn't have
  // to carry the burden of proving them while also proving the rest
  final_step
```

### Predicate Abstraction

When you have large, repeated predicate expressions copied across multiple
functions or assertions:

1. Extract into a named predicate: `let my_pred x y = ...`
2. Write pure lemmas relating predicates to each other
3. Use fold/unfold (in Pulse) to control when Z3 sees internals
4. This dramatically reduces proof complexity and Z3 work

```fstar
// BAD: large inline predicate repeated in 5 places
ensures (forall i. 0 <= i /\ i < length arr ==> index arr i >= 0 /\ index arr i < bound /\ ...)

// GOOD: named predicate with a lemma
let all_in_bounds (arr: seq int) (bound: int) = forall i. 0 <= i /\ i < length arr ==> ...
val all_in_bounds_preserved : arr:_ -> bound:_ -> i:_ -> Lemma (...)
```

### Wrong Symbol (Copy-Paste Bug)

**Symptom:** Proof fails inexplicably; the code "looks right."

**Fix:** Use `--print_full_names --print_implicits`:
```bash
./fstar.sh --print_full_names --print_implicits Module.fst
```

Check that each symbol resolves to the intended module. A function copied from
another module may reference the wrong qualified name.

## Pulse-Specific Debugging

### Loops

If a loop-related proof fails, **check the loop condition first**: verify the invariant is
compatible with both the entry and the exit condition. Then simplify incrementally —
remove non-essential invariants and add them back one at a time until the failure
reappears.

### Resource Mismatch

**Symptom:** "Could not prove post-condition" with separation logic.

**Debugging approach:**
1. Add `assert` for each slprop component to find which one is missing
2. Check fold/unfold balance
3. Verify `rewrite` targets are correct
4. Ensure no resource was accidentally dropped

### Probing With `assert pure`

Use `assert pure (P)` to probe what the solver actually knows at a program point — a
failing assertion often reveals the real issue more precisely than the original error.

Never "fix" it by turning the `assert` into an `assume`. `assume pure` is a hole in the
proof that can hide memory safety bugs, and a kernel with assumes is not verified. The
same goes for `magic()` and `admit()`: `admit()` is acceptable only as a temporary
scaffold while validating structure, and must be gone before the change is done.

### Ghost Context Confusion

**Symptom:** "Application of stateful computation cannot have ghost effect."

**Debugging approach:**
1. Check if you're inside a `with ... . _` scope
2. Check if the `if` condition depends on ghost values
3. Restructure: do stateful reads before ghost reasoning

### Lemma Calls Failing in Pulse

**Symptom:** A lemma works in pure F* but fails when called from Pulse.

**Debugging approach:**
1. Verify the exact precondition with `assert (pure (precondition))` before the call
2. Use `--print_full_names` to verify the lemma resolves to the right definition
3. Check that value-level equalities match what the lemma expects
   (e.g., `U64.v x = U64.v y` vs `x == y`)
4. Try calling the lemma outside the current conditional/loop scope — Pulse scoping
   can affect what facts Z3 has available
5. Factor the lemma call into a separate pure F* helper function and call that
   from Pulse — this isolates whether the issue is in Pulse's VC generation or
   in the lemma itself

## Anti-Patterns

### Increasing rlimit Instead of Fixing

❌ `#push-options "--z3rlimit 300"` — masks the problem, creates flaky proof

✅ Factor into smaller lemmas with `--z3rlimit 10`

### Assuming Instead of Proving

❌ `assume pure (...)`, `magic()`, or a leftover `admit()`

✅ Restructure the code or add the lemma. A kernel with assumes is not verified.

### Reflexively Adding Lemmas and Asserts

❌ A new lemma whose proof body is just `()` — SMT could already discharge it in context

❌ A wall of intermediate `assert`s "to help Z3" — they pollute the context and can make
   Z3 *slower*

✅ Search the Kuiper library and the F* standard library first; add an assertion only when
   you have evidence it is load-bearing

### Blaming the Tool

❌ "Pulse can't handle this pattern" (without evidence)

✅ Produce a minimal reproducer; check for mundane bugs first

### Accepting Flaky Proofs

❌ "It passes sometimes, so it's fine"

✅ Run with `--z3refresh` and reduce rlimit until it's robust

### Admit and Move On

❌ `admit()` left in "for now"

✅ Extract the exact property into a named lemma and prove it

## Session Structure for Proof Work

1. **Exploration**: Understand the codebase and existing proofs; search the Kuiper library
   for lemmas and combinators that already do what you need
2. **Structure**: Write code with `admit()` placeholders to validate the approach
3. **Prove**: Remove admits one at a time, starting with the easiest
4. **Harden**: Reduce rlimits, run with `--z3refresh`, clean up
5. **Integrate**: Verify through the module's Makefile target, regenerate the extracted
   CUDA if extraction attributes changed, commit

## Know When to Stop

If a proof seems disproportionately hard, or you are cycling through the same failures
without progress, stop and ask the user for guidance rather than forcing it. A structural
change to the implementation is often far cheaper than the proof you are fighting.
