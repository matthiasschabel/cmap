# MCSLAB cmap upstream roadmap

**Status:** Active
**Last updated:** 2026-08-09
**Scope:** small upstream cmap contributions that allow MCSLAB to reduce its colormap layer

## Context

MCSLAB currently compensates for several cmap defects and owns capabilities that would be useful in
cmap itself. The changes must go upstream as independent, low-attention pull requests. This note
holds the complete sequence so individual PR descriptions do not need to explain the downstream
migration program.

The verified baseline is cmap commit
`8040ef777c9c7aee7959e6029ffcd1a97582f41e`. Current dispatch sends NaN and masked values to
`bad`, negative infinity through `under`, and positive infinity through `over`, despite one
constructor-docstring line describing `bad` as handling infinity.

## Current Decision

### Queue ordered from clearest to most design-dependent

1. **Fix napari `high_color` conversion.** `to_napari()` checks for `high_color` but assigns the
   over color to `nan_color`. Change the key and add one converter regression test.
2. **Stop reversal from mutating source stops.** `ColorStops.reversed()` rewrites a reversed NumPy
   view. Copy before changing stop positions and assert the source remains unchanged.
3. **Preserve unmasked NaNs in masked-array input.** Combine mask metadata with `np.isnan` instead
   of replacing the NaN mask. Test one mixed masked array, including an all-false mask case.
4. **Clarify exceptional-value documentation.** Align constructor, attribute, call, and LUT docs
   with the verified NaN/mask and signed-infinity dispatch. This is documentation-only unless the
   maintainer chooses a new breaking definition of `bad`.
5. **Prevent constructor/update aliasing of `ColorStops`.** Construction from existing stops and
   `with_extremes()` can rewrite the source interpolation. Give the result owned stops and test a
   nearest-interpolation source.
6. **Preserve existing public state in copy/update paths.** Copy name, identifier, category,
   interpolation, and under/over/bad; distinguish omitted fields from explicit `None`. This changes
   accidental `with_extremes()` clearing and needs maintainer agreement.
7. **Make reversal semantics complete and consistent.** Require named `_r` construction and
   `.reversed()` to agree, including swapping under/over catalog defaults while preserving bad.
   Test a catalog entry with explicit extremes.
8. **Make persistence lossless.** Keep the default `as_dict()` shape stable, but use a tagged state
   mapping for customized objects in pickle/Pydantic/psygnal paths. Canonical catalog objects may
   continue serializing as qualified names.
9. **Add shared and signed infinity colors.** Add `inf`, `neg_inf`, and `pos_inf` with the compatible
   fallback `sign-specific -> inf -> legacy under/over -> ramp endpoint`. This is the direct MCS
   migration capability.
10. **Optionally split masked data from unmasked NaN.** Add `nan` and `masked` together, resolving
    each through `bad`. Do not add `nan` alone merely as a second spelling for existing behavior.
11. **Add exact palette-variant selection.** Expose exact available sizes and `variant(size)` without
    changing the sampling meaning of `lut(N)`.
12. **Annotate verified ColorBrewer and Tol families.** Add catalog relationships and validation as
    a data-only follow-up to the accepted variant API.

Each item starts from current `upstream/main`. Merge it into `integration` only after its own test
passes. Combine adjacent items only when the maintainer explicitly prefers the larger review unit.

### Exceptional-value contract

The compatible proposed resolution is:

| Input class | Resolution order |
|---|---|
| masked | `masked -> bad -> transparent` |
| unmasked NaN | `nan -> bad -> transparent` |
| negative infinity | `neg_inf -> inf -> under -> first ramp color` |
| positive infinity | `pos_inf -> inf -> over -> last ramp color` |

Constructor arguments use cmap's unsuffixed convention; properties use `_color`. Every field uses
the existing `ColorLike` parser, including RGBA tuples.

Making `bad` a fallback for infinity is not additive. A cmap configured only with `bad` currently
routes infinities through under/over or the ramp endpoints; changing those colors requires an
explicit compatibility decision. Do not introduce a hidden activation rule or a
`bad_includes_inf` mode flag.

### Scope retained by MCSLAB

Do not upstream the MCS `ColormapSpec`, `SpecialColors` aggregate, mutable registry, physical-limit
normalization, GUI schema, evaluated-value facade, or editing transforms as one package. Upstream
only the generally useful primitives. MCS consumes released capabilities through feature probes and
keeps application policy downstream.

## Alternatives Considered

- **One comprehensive colormap PR.** Rejected because state fixes, exceptional-value policy, and
  catalog variants have independent risk and review ownership.
- **A public upstream `SpecialColors` object.** Rejected because flat constructor fields already fit
  cmap's API and avoid a second construction model.
- **Make `bad` universally mean all non-finite values.** Coherent but breaking; retain as an explicit
  major-version option if the maintainer rejects legacy infinity fallbacks.
- **Expand the N+3 LUT for every exceptional class.** Rejected because it breaks Matplotlib-shaped
  consumers. Rich dispatch remains in `__call__`.
- **Make `lut(N)` choose a published size variant.** Rejected because sampling and selecting an
  independently designed palette are different operations.

## Deferred Work

- Final public spelling of signed fields remains a maintainer decision. `neg_inf`/`pos_inf` is
  preferred over `minf`/`pinf` because it is unambiguous.
- Converter behavior for targets that cannot represent the richer classes needs a per-target loss
  policy. Warnings should identify the user callsite and follow Python's normal warning filter.
- Separate mask coloring is useful but not required for current MCS, whose `colorize` rejects masked
  arrays.

## Next Steps

1. Implement the napari `high_color` fix on a branch from `upstream/main`.
2. Keep the PR body to symptom, cause, fix, and the one regression test.
3. Merge the feature branch into `integration` after local verification.
4. Advance through source-mutation and masked-array defects before proposing new public fields.
