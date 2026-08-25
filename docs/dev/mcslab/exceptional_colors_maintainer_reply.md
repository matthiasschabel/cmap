# Draft reply to tlambert03 on PR #151 (comment 5266155726)

**Status:** Superseded
**Last updated:** 2026-08-25
**Scope:** historical draft for PR #151 before the mask-specific field was removed

The owner posted a revised response during review, then removed the proposed `masked` field in
commit `1a998df`. The mask-specific argument below is retained only as review history.

Drafted 2026-08-12 for the owner to edit and post. Answers the maintainer's five questions;
policies settled by the owner the same day. Findings verified empirically against `main` and
against matplotlib 3.11.1; see `upstream_roadmap.md` items 6 to 8.

---

Thanks for the careful read.

**Behavior when the new values are unset:** unchanged, by construction. The routing appends
fallback-resolved rows to a call-local copy of the LUT, so a class with no color of its own
lands on exactly the row it lands on today.
`test_exceptional_colors_fall_back_to_the_legacy_extremes` pins the four legacy
destinations, and I also compared outputs against `main` for data containing both
infinities, NaN, and a masked entry, with and without `under`/`over`/`bad` set: identical.

**Are all three of `bad`/`nan`/`masked` needed?** I think so. `bad` is not naming regret; it
is the umbrella tier of a two-level hierarchy, parallel to `under`/`over` above
`neg_inf`/`pos_inf`. What remains for `bad` to claim is the common case: one color for
"invalid, I don't care why", which is also the matplotlib-compatible spelling, so the casual
user never needs the finer names. `nan` and `masked` are for when "measured but undefined"
and "deliberately excluded" have to read differently. `masked` also does one thing no other
class can: it can mark values that are in range. Masking by a predicate gives you a cheap
contour band or a polarity-change overlay on raster data without touching the values.

**The four preservation questions:** your suspicion is right, and it goes further than the
new fields. Today none of the four channels preserve the existing `under`/`over`/`bad`
either. Verified on current `main`:

- `reversed()` passes only stops, name, and category, so it drops every extreme color and
  the interpolation mode (a nearest colormap comes back linear).
- `with_extremes()` clears anything not repeated, where matplotlib's preserves what you do
  not pass.
- `__reduce__` carries only `color_stops`, so a pickle round trip loses name, category,
  interpolation, and all extreme colors. `pickle.loads(pickle.dumps(cm)) == cm` is already
  False for any colormap with `bad` set.
- `as_dict()` has no extreme keys, and it is what the pydantic serializer and
  `_json_encode` emit, so a model round trip silently strips them. A catalog colormap
  serializes as just its qualified name, so `Colormap("viridis", under="red")` comes back
  as plain viridis.

What I would propose, following matplotlib wherever it has a position:

- `reversed()`: swap `under` and `over`, preserve `bad`, as matplotlib does; `neg_inf` and
  `pos_inf` swap along with the ends they extend, and `nan`/`masked` are preserved with
  `bad`. Interpolation carries through.
- `with_extremes()`: preserve anything not passed, as matplotlib does. Clearing a single
  color then means constructing a fresh Colormap, the same limitation matplotlib has.
- pickle: carry the full constructor state through `__reduce__`.
- `as_dict()`: optional keys for interpolation and the extreme colors, emitted only when
  set, so existing payloads are byte-for-byte unchanged; the pydantic serializer falls back
  to the dict form when a catalog colormap carries extremes. Deserialization already works,
  since `_validate` calls the constructor and it accepts all of these as kwargs.

Happy to do that as commits on this PR or as a separate follow-up so this one stays the
feature alone, whichever you prefer.
