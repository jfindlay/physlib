# PLAN — Issue #353: Multigoal linter

Handoff plan for a `@build` (Sonnet) agent. Branch: `jfindlay/353/multigoal-lint`.
Toolchain: `leanprover/lean4:v4.30.0`, Mathlib `v4.30.0`. Mathlib is cached as a
separate Lake package, so enabling a root `leanOption` rebuilds only first-party
modules (~2050 incremental jobs), not Mathlib.

## Goal

Enable Mathlib's `linter.style.multiGoal` linter project-wide and fix every
first-party warning it produces, by focusing each goal with `·` (cdot bullet).

## Key finding (do not skip)

The issue's literal recipe — `linter.style.multiGoal = true` in `[leanOptions]` —
**breaks the build** on this toolchain. 20 placeholder/minimal modules that do not
transitively import Mathlib's linter framework error out with:

```
error: <file>.lean:1:0: invalid -D parameter, unknown configuration option 'linter.style.multiGoal'
```

The correct form, matching Mathlib's own lakefile, is the **`weak.` prefix**:

```toml
[leanOptions]
weak.linter.style.multiGoal = true
```

A `weak`-prefixed option is silently ignored by modules that don't recognize it, so
the 20 placeholder files build clean with zero edits, while proof files still emit
the warnings. Both halves were verified during planning. **Mention this deviation
from the issue text in the PR description.**

## Commit structure (2 commits)

- **Commit 1** — `physlib-353 Enable multiGoal style linter`
  - Edit `lakefile.toml`: insert a `[leanOptions]` block with
    `weak.linter.style.multiGoal = true` immediately after the
    `defaultTargets = [...]` line (before the first `[[require]]`).
  - This commit enables the linter only; no fixes yet.

- **Commit 2** — `physlib-353 Fix multiGoal linter warnings`
  - All 358 `·`-focus fixes across the 43 files below.
  - `@build` may work file-by-file but **squashes to this single fix commit**.

## Step-by-step

### Step 0 — baseline
Confirm clean tree on branch `jfindlay/353/multigoal-lint`. Confirm
`lake build Physlib.Optics.Basic` succeeds (warm cache check).

### Step 1 — enable (Commit 1)
Apply the `[leanOptions]` edit above. Build a previously-failing placeholder file
to confirm no `-D` error:
```
lake build Physlib.Cosmology.Basic Physlib.Optics.Basic Physlib.Units.WithDim.Mass
```
All three must succeed. Commit.

### Step 2 — harvest authoritative warning list
Run the full build, capturing output:
```
env LEAN_ABORT_ON_PANIC=1 lake build 2>&1 | tee /tmp/opencode/mg-build-2.log
```
The warning inventory below was captured during planning; re-harvest to get exact
current line:col coordinates before editing each file (line numbers shift as you
edit). **Always re-read a file's current warnings right before fixing it** — do not
trust stale line numbers.

### Step 3 — fix warnings (Commit 2)

For each warning of the form:
```
<file>:<line>:<col>: The following tactic starts with N goals and ends with M goals, K of which is not operated on.
  <the offending tactic line>
```

The offending tactic runs while N>1 goals are open but only operates on the first.
Fix: focus each goal with a `·` bullet and indent that goal's tactics under it.

Minimal example (from the issue):
```lean
  constructor
  exact h1
  exact h2
```
becomes
```lean
  constructor
  · exact h1
  · exact h2
```

**Care points — these are NOT blind mechanical edits:**

1. **Re-indentation.** Every tactic belonging to a focused goal must be indented one
   level under its `·`. A `·` bullet's body is the indented block that follows it.

2. **Goal-spawning tactics.** Warnings where the tactic *ends with more goals than it
   started* (e.g. `starts with 2 ends with 3`) mean the tactic split the focused goal
   into several. Those new subgoals belong inside that bullet's block — keep them under
   the same `·` (possibly with nested `·` bullets), not as siblings of the original
   goals.

3. **Nested multi-goal blocks.** A focused goal may itself contain another unfocused
   multi-goal sequence (the linter reports each independently). Resolve from the
   outermost goal inward; re-build to surface any still-nested warnings.

4. **Do not change proof semantics.** The proof must still compile. `·` focusing is
   purely structural — if a proof breaks after adding bullets, the indentation/grouping
   is wrong, not the tactic content. Never delete or reorder tactics to silence a
   warning.

5. **Preserve the project's formatting** (100-char wrap, existing indent width — these
   files use 2-space indent).

**Per-file loop:**
- Re-read the file's current warnings (build just that module:
  `lake build <Module.Name>`).
- Apply `·` fixes.
- Rebuild that module; confirm **zero** multiGoal warnings remain AND the module still
  builds successfully.
- Move to next file.

### Step 4 — final verification
Full clean build must produce **zero** `linter.style.multiGoal` warnings from
first-party files and **no errors**:
```
env LEAN_ABORT_ON_PANIC=1 lake build 2>&1 | tee /tmp/opencode/mg-build-final.log
```
Grep the log for `multiGoal` / `starts with .* goals` — expect none from
`Physlib/`, `QuantumInfo/`, `PhyslibAlpha/`. (Mathlib-origin warnings, if any, are
out of scope and acceptable — but with the linter enabled only at the root project
level, Mathlib should emit none.)

Then squash file-batches into the single Commit 2.

## Warning inventory (captured during planning — 358 warnings, 43 files)

Line numbers are from the planning build and WILL shift as you edit; use them only to
locate files and gauge density. Re-harvest per Step 2.

Physlib (sorted by count):
```
 59  Physlib/Mathematics/List/InsertionSort.lean
 36  Physlib/Mathematics/InnerProductSpace/Basic.lean
 25  Physlib/StatisticalMechanics/CanonicalEnsemble/Basic.lean
 21  Physlib/QuantumMechanics/OneDimension/GeneralPotential/Basic.lean
 17  Physlib/Mathematics/List.lean
 13  Physlib/StringTheory/FTheory/SU5/Quanta/FiveQuanta.lean
 13  Physlib/StringTheory/FTheory/SU5/Quanta/TenQuanta.lean
 12  Physlib/Mathematics/Trigonometry/Tanh.lean
 12  Physlib/Mathematics/Calculus/AdjFDeriv.lean
 12  Physlib/StatisticalMechanics/CanonicalEnsemble/Finite.lean
 11  Physlib/QuantumMechanics/OneDimension/HarmonicOscillator/TISE.lean
 10  Physlib/Mathematics/FDerivCurry.lean
  9  Physlib/QuantumMechanics/OneDimension/HarmonicOscillator/Completeness.lean
  8  Physlib/QFT/PerturbationTheory/FieldStatistics/Basic.lean
  8  Physlib/QFT/PerturbationTheory/FieldStatistics/OfFinset.lean
  7  Physlib/Particles/SuperSymmetry/SU5/ChargeSpectrum/Completions.lean
  6  Physlib/QuantumMechanics/OneDimension/Operators/Momentum.lean
  5  Physlib/SpaceAndTime/Time/TimeTransMan.lean
  4  Physlib/Mathematics/Calculus/ParametricIntegration.lean
  3  Physlib/Mathematics/Distribution/PowMul.lean
  3  Physlib/QuantumMechanics/OneDimension/Operators/Commutation.lean
  2  Physlib/Particles/FlavorPhysics/CKMMatrix/StandardParameterization/StandardParameters.lean
  2  Physlib/QuantumMechanics/OneDimension/Operators/Parity.lean
  2  Physlib/StringTheory/FTheory/SU5/Fluxes/NoExotics/ChiralIndices.lean
  2  Physlib/QuantumMechanics/DDimensions/Operators/Unbounded.lean
  2  Physlib/QuantumMechanics/OneDimension/ReflectionlessPotential/Basic.lean
  2  Physlib/Particles/SuperSymmetry/SU5/ChargeSpectrum/MinimalSuperSet.lean
  1  Physlib/Mathematics/Fin/Involutions.lean
```

QuantumInfo:
```
 10  QuantumInfo/ForMathlib/Majorization.lean
  9  QuantumInfo/States/Mixed/MState.lean
  7  QuantumInfo/ForMathlib/HermitianMat/CFC.lean
  5  QuantumInfo/Entropy/SSA.lean
  3  QuantumInfo/ForMathlib/LimSupInf.lean
  3  QuantumInfo/ForMathlib/Isometry.lean
  3  QuantumInfo/ForMathlib/HermitianMat/LogExp.lean
  2  QuantumInfo/ForMathlib/HermitianMat/Proj.lean
  2  QuantumInfo/ForMathlib/HermitianMat/Schatten.lean
  2  QuantumInfo/ForMathlib/HermitianMat/Peierls.lean
  1  QuantumInfo/ForMathlib/Misc.lean
  1  QuantumInfo/ForMathlib/Matrix.lean
  1  QuantumInfo/ForMathlib/HermitianMat/Order.lean
  1  QuantumInfo/States/Pure/Braket.lean
  1  QuantumInfo/Entropy/VonNeumann.lean
```

PhyslibAlpha: none.

## Reference: the 20 files the `weak.` prefix protects (no edits needed)

Listed so `@build` can sanity-check that none of these accidentally get touched.
```
Physlib/Meta/TODO/Basic.lean
Physlib/ClassicalMechanics/Basic.lean
Physlib/Meta/Basic.lean
Physlib/ClassicalMechanics/Pendulum/MiscellaneousPendulumPivotMotions.lean
Physlib/Meta/Informal/Basic.lean
Physlib/CondensedMatter/Basic.lean
Physlib/Cosmology/Basic.lean
Physlib/Meta/Informal/SemiFormal.lean
Physlib/Meta/Notes/Basic.lean
Physlib/Meta/Notes/NoteFile.lean
Physlib/Meta/Remark/Basic.lean
Physlib/Optics/Basic.lean
Physlib/Optics/Polarization/Basic.lean
Physlib/QFT/PerturbationTheory/FeynmanDiagrams/Basic.lean
Physlib/StringTheory/Basic.lean
Physlib/StringTheory/FTheory/SU5/Basic.lean
Physlib/Thermodynamics/Basic.lean
Physlib/Units/WithDim/Mass.lean
Physlib/Units/WithDim/Velocity.lean
QuantumInfo/Capacity/Capacity_doc.lean
```

## Out of scope
- Mathlib-origin warnings (the issue says ignore these; with the linter set only at
  the root project level they should not appear).
- Any proof refactor beyond inserting `·` and re-indenting.
