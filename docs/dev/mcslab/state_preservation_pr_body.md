Follow-up to the four preservation questions on #151. Depends on #151: the first commit here is
that PR's, and the diff shrinks to the second commit alone once it lands.

None of the four channels preserved the extreme colors, and two of them dropped the interpolation
mode as well. Verified on `main` before this change:

- `reversed()` passes only stops, name, and category, so a `nearest` colormap comes back `linear`
  with no extremes.
- `with_extremes()` clears anything not repeated in the call.
- `__reduce__` carries only `color_stops`, so `pickle.loads(pickle.dumps(cm)) == cm` is already
  False for any colormap with `bad` set.
- `as_dict()` has no keys for interpolation or the extremes, and it is what both the pydantic
  serializer and `_json_encode` emit.

Following matplotlib 3.11 where it has a position:

- `reversed()` swaps `under`/`over` and `neg_inf`/`pos_inf`, which name the ends they extend, and
  preserves the rest.
- `with_extremes()` preserves anything not passed. Clearing a single color now means constructing
  a new `Colormap`, the same limitation matplotlib has. This is the one backward-incompatible
  change here.
- `__reduce__` carries the full constructor state.
- `as_dict()` gains optional keys, written only when they hold non-default state, so existing
  payloads are unchanged.

Two changes that are more than preservation:

- `Colormap("x_r")` now swaps the catalog record's `under`/`over`, so it agrees with
  `Colormap("x").reversed()`. `napari:HiLo` is the only affected entry. An explicit
  `under=`/`over=` argument still lands on the end the caller named.
- The pydantic serializer emitted a catalog colormap's qualified name unconditionally, so
  `Colormap("viridis", under="red")` came back as plain viridis, and `Colormap("viridis_r")` came
  back unreversed because `info` is looked up under the stripped name. It now emits the name only
  when the name alone rebuilds the same `as_dict()`. `__eq__` is not usable as that guard: it
  ignores name, identifier, and category, and `ColorStops.__eq__` compares with `np.allclose`, so
  a `cmap_kwargs={"start": 0.500001}` cubehelix compares equal to the default one.

What I left alone, and why:

- Pickle passes `color_stops` as the object rather than `as_dict()`'s samples, so a colormap
  backed by a lut function keeps the function. Routing pickle through `as_dict()` shifted
  `lut(17, gamma=2)` of a parametrized cubehelix by 2e-5.
- A round trip through the dict form does not restore `info`, which is set only from a string or
  another `Colormap`, so an unpickled catalog colormap serializes as a dict rather than as its
  name. Fixing that means putting the qualified name in `value`, which would change existing
  payloads for every catalog colormap.
- `identifier` follows the name: `reversed()` and `shifted()` rename, so it re-derives;
  `with_extremes()` does not, so it is preserved. One consequence: `cm.reversed().reversed()`
  restores the name but not an explicitly supplied identifier. Say if you would rather it were
  carried through.
- The converters still have no representation for the finer classes.
