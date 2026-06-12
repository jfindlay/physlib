# Local working notes (untracked — do not commit to upstream PRs)

## EM-gauge-followups candidates

### Algebraic instances for `ElectromagneticPotential` (captured 2026-06-12)

Discovered while polishing `jfindlay/EM-gauge-properties` (commit `23e23eb1`):
`ElectromagneticPotential d` carries `Add` (used by `gaugeTransform`) but **no `Zero`
instance**, so the bundled statement `ofGradient 0 = 0` is inexpressible — the pointwise
form of `ofGradient_zero` in `GaugeTransformation.lean` §B.3 is forced by the typeclass
landscape, not a style choice (now documented inline at the lemma).

Follow-up work, in dependency order:

1. `Zero` (and plausibly the full `AddCommGroup`) instance for `ElectromagneticPotential d`,
   lifted from the function type via `val`. Lives in
   `Physlib/Electromagnetism/Kinematics/EMPotential.lean`.
2. Upgrade `ofGradient_zero` to the bundled `@[simp] ofGradient 0 = 0`; simplify
   `gaugeTransform_zero` to `simp [gaugeTransform]`.
3. Graduate the §B.3 group-structure prose to a formal
   `AddAction (SpaceTime d → ℝ) (ElectromagneticPotential d)` via `gaugeTransform` —
   the module currently proves the action laws (`gaugeTransform_zero`,
   `gaugeTransform_gaugeTransform`) but does not bundle them. Caveat: the composition
   law needs differentiability side conditions, so a global `AddAction` on *all*
   functions is false; the action must be on a subtype/submonoid of (at least)
   differentiable gauge functions — this is the real design knot for the followup.

Session provenance: branch review session, gauge-transformation module
(`Physlib/Electromagnetism/Kinematics/GaugeTransformation.lean`).
