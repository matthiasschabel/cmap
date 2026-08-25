Closes #144.

Adds three optional colors so exceptional float values can be told apart: `neg_inf`,
`pos_inf`, and `nan`. Today `-inf` is indistinguishable from any other under-range value,
and NaN is indistinguishable from `bad`. Log-transformed signal data and saturated logistic
regression both produce infinities worth marking rather than blending into the ends of the
scale.

Each new color falls back to the one its class uses now, so nothing changes for an existing
colormap:

| Class | Color | Falls back to |
|---|---|---|
| negative infinity | `neg_inf` | `under`, then the first ramp color |
| positive infinity | `pos_inf` | `over`, then the last ramp color |
| NaN | `nan` | `bad`, then transparent |

`bad` stays the fallback for `nan` rather than being replaced by it, so code that sets it is
unaffected and the more specific color may be set alone.

Two things the diff does not show:

- The infinity masks are taken before `xa *= N`. That multiply overflows large finite values
  to infinity (`float16` 65504 does at N=256), and those are out of range, not infinite.
  There is a test for it, because classifying after the multiply looks right and is not.
- The new colors do not round trip through pickle or `as_dict()`. Neither do
  `under`/`over`/`bad`, so this is an existing limitation rather than a new one. PR #155
  addresses that separately.

`to_napari` now prefers `nan_color` over `bad_color` for napari's `nan_color`, since it is
the one converter target that represents the class. matplotlib's `bad` covers NaN and masked
together, so `bad` is still what goes there.
