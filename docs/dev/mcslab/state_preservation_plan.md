# Plan: preserve full Colormap state through copy and serialization

**Status:** Active
**Last updated:** 2026-08-13
**Scope:** `src/cmap/_colormap.py`, `tests/test_colormap.py`, `tests/test_model_fields.py`
**Baseline:** `feat/exceptional-colors` (PR #151), rebased onto `upstream/main` at `f0a4aec`

## Context

The maintainer asked on #151 whether the extreme colors survive `reversed()`, `with_extremes()`,
pickle, and `as_dict()`/pydantic, and on 2026-08-13 said: "I suppose we should go ahead and split
that fix out into a new PR. And it needn't hold this one up either." Singular PR, and no
dependency in the other direction.

None of the four channels preserves even the pre-#151 `under`/`over`/`bad`. Verified against
`upstream/main`:

| Channel | What is lost today |
|---|---|
| `reversed()` | every extreme color, `interpolation`; passes only stops, name, category |
| `with_extremes()` | any extreme not repeated in the call; also re-derives `identifier` from `name` |
| `__reduce__` | everything but the stops: name, identifier, category, interpolation, all extremes |
| `as_dict()` | interpolation and all extremes; no keys exist for them |
| pydantic `_serialize` | a catalog colormap emits its bare qualified name, dropping added extremes, and a reversed catalog colormap emits the *unreversed* name |

Measured reference behavior, matplotlib 3.11.1:

- `reversed()` swaps `under` and `over` and preserves `bad`.
- `with_extremes()` preserves anything not passed.
- pickle preserves all three plus the name.

The maintainer agreed to mirror matplotlib where it has prior art, so the policy below is
settled rather than proposed.

## Current Decision

### One field list

Add a module-level tuple and one small accessor, so the seven extreme fields are enumerated once
outside `__init__` and `__eq__`:

```python
_EXTREME_FIELDS = ("under", "over", "bad", "neg_inf", "pos_inf", "nan", "masked")

@property
def _extremes(self) -> dict[str, Color | None]:
    """The extreme colors, keyed by their constructor argument name."""
    return {f: getattr(self, f"{f}_color") for f in _EXTREME_FIELDS}
```

`reversed()` and `shifted()` keep passing keywords explicitly, matching the block `shifted()`
already has. In `reversed()` the swap is the substance of the method, and a generic helper would
hide it.

### `reversed()`

Carry everything, and swap the directional pairs, because they name the ends they extend:

```python
under=self.over_color,       over=self.under_color,
neg_inf=self.pos_inf_color,  pos_inf=self.neg_inf_color,
bad=self.bad_color,          nan=self.nan_color,     masked=self.masked_color,
interpolation=self.interpolation,
```

`identifier` is deliberately not passed: the name changes to `name_r`, and identifier is derived
from name. That is the rule throughout this change, **identifier travels with the name**, which
also settles the question the roadmap left open. The consequence worth stating in the PR body:
`cm.reversed().reversed()` restores the name but not an explicitly supplied `identifier`, since
the second call re-derives it from the restored name. The maintainer can overrule the rule there.

`Colormap("x_r")` must agree with `Colormap("x").reversed()`, so `__init__`'s `rev` branch swaps
the catalog record's `under`/`over` too. Today `Colormap("napari:HiLo_r")` reports
`under=blue, over=red`, identical to the unreversed entry. `napari:HiLo` is the only catalog
entry carrying both.

The swap applies to the **record's** values, before the explicit-override resolution, not after:

```python
info_under, info_over = (info.over, info.under) if rev else (info.under, info.over)
under = info_under if under is None else under
over = info_over if over is None else over
```

Swapping the resolved variables would swap a caller's own `under=`/`over=` arguments as well,
which no other constructor argument does.

### `with_extremes()`

Preserve anything not passed. Clearing one color then means constructing a fresh `Colormap`,
which is matplotlib's limitation too. Also pass `identifier=self.identifier`, since the name does
not change here. For a catalog colormap the re-derived identifier happens to match, so the loss
shows only with an explicit one: `Colormap(stops, name="Foo Map", identifier="my_id")` becomes
`foo_map` after any `with_extremes()` call.

This is the one backward-incompatible item in the set: code calling `with_extremes()` to *clear*
an extreme keeps it after this change.

### `__reduce__`

```python
def __reduce__(self) -> str | tuple[Any, ...]:
    return _rebuild_colormap, (self.__class__, self.color_stops, self._constructor_kwargs())
```

with a module-level `_rebuild_colormap(cls, value, kwargs)` returning `cls(value, **kwargs)`. The
two-tuple pickle form takes positional args only, so the helper is what allows keywords;
`self.__class__` keeps a subclass reconstructing as itself.

`color_stops` is passed as the object, not as `as_dict()`'s sampled stops. A `ColorStops` built
from a callable keeps `_lut_func` and re-evaluates it at whatever `N` and `gamma` are later
requested; `as_dict()` materializes 256 fixed stops instead. Measured on
`Colormap("cubehelix", cmap_kwargs={"start": 1.0, "rotation": -1.0})`: the current pickle round
trip reproduces `lut(17, gamma=2)` exactly, while a round trip through `as_dict()` loses the
callable and shifts the LUT by 2.1e-5. Pickle must not regress that.

`_constructor_kwargs()` is the shared private accessor: name, identifier, category,
interpolation, and every set extreme as `Color` objects. `as_dict()` is that dict with the colors
and stops turned into JSON-compatible lists.

### `as_dict()`

Add optional keys, emitted only when they carry non-default state, so an existing payload is
byte-for-byte unchanged. `tests/test_model_fields.py:47` asserts an exact JSON string under CI and
must keep passing untouched.

- `interpolation` when it is not `"linear"`. `value` is always the expanded stops, so
  reconstruction goes through `_parse_colorstops`, whose default is linear.
- each set extreme, as `list(color)`, matching how stops already encode colors.

`ColormapDict` splits into a required base and a `total=False` subclass for the optional keys.

### pydantic serialization

```python
def _serialize(obj: Colormap) -> Any:
    state = obj.as_dict()
    if obj.info is not None and (qualified := obj.info.qualified_name):
        if state == Colormap(qualified).as_dict():
            return qualified
    return state
```

The name alone is a complete serialization exactly when it rebuilds the same `as_dict()`. This
keeps `Colormap("viridis")` serializing as `"bids:viridis"` and fixes three current losses in one
condition: a catalog colormap with added extremes; a catalog colormap renamed or recategorized;
and `Colormap("viridis_r")`, which today serializes as `"bids:viridis"` and comes back unreversed,
because `info` is looked up under the stripped name.

`__eq__` is not usable as this guard. It ignores `name`, `identifier`, and `category`, and
`ColorStops.__eq__` compares with `np.allclose`, so `Colormap("cubehelix", cmap_kwargs={"start":
0.500001})` compares equal to the default cubehelix and would serialize to a name that discards
the parametrization. Both verified. Comparing `as_dict()` is exact, and it stays correct when a
constructor argument is added later, which a hand-listed eligibility check would not.

Cost: `as_dict()` is 0.98 ms for a 256-stop catalog colormap and the reference construction is
0.04 ms, so an eligible colormap serializes in about 2 ms against 1 ms today. This is a
serialization path, not a mapping path.

### Known limitation, stated in the PR body

A round trip through the dict form does not restore `info`: it is set only when the constructor is
given a string or another `Colormap`. So an unpickled catalog colormap has `info is None` and
subsequently serializes as a dict rather than as its name. Equality holds, since `__eq__` does not
compare `info`. Fixing it would mean putting the qualified name in `value`, which changes existing
payloads for every catalog colormap, so it is out of scope. The PR therefore claims constructor
state, not "full state": `info` and the serialized *shape* are outside what it preserves.

## Alternatives Considered

- **Four PRs, one per channel.** Rejected. It is one defect with four symptoms, and each PR would
  re-enumerate the same field list, get the same "did you miss a field?" review, and conflict with
  the other three in the same functions. The maintainer asked for one PR.
- **A fifth PR extending the fix to #151's four colors.** Rejected. Its content is decided by merge
  order and disappears under either order. Stacking on #151 covers all seven at once.
- **Routing `__reduce__` through a private state dict instead of `as_dict()`.** Rejected here
  because both want the same payload and `_validate` already commits `as_dict()` to being
  constructor-shaped; a second near-identical dict builder is the duplication this change exists to
  remove.
- **Emitting the qualified name as `value` for catalog colormaps.** Smaller payloads and it would
  restore `info`, but it changes existing `as_dict()` output for every catalog colormap.

## Verification

Red first, per the practice recorded in `integration_workflow.md`.

1. `reversed()` round trip: a colormap with all seven set plus `interpolation="nearest"` reverses
   with the two directional pairs swapped and the rest intact.
2. `Colormap("napari:HiLo_r")` equals `Colormap("napari:HiLo").reversed()`, and
   `Colormap("napari:HiLo_r", under="green")` keeps green on `under`, so the swap does not reach
   an explicit override.
3. `with_extremes()` preserves what is not passed and keeps an explicit `identifier`.
4. `pickle.loads(pickle.dumps(cm)) == cm` for a colormap with all seven set, and the name,
   category, and interpolation survive. Separately, a callable-backed colormap
   (`cmap_kwargs`-parametrized cubehelix) survives pickle, `copy`, and `deepcopy` with
   `lut(17, gamma=2)` bit-identical, which is the B1 regression guard.
5. `as_dict()` of a plain colormap has exactly the four existing keys; of a configured one, round
   trips through `Colormap(**d)` to an equal object.
6. pydantic: `Colormap("viridis")` still serializes to `"bids:viridis"`;
   `Colormap("viridis", under="red")`, `Colormap("viridis", name="renamed")`, and
   `Colormap("viridis_r")` all round trip through the model with their distinguishing field
   intact.

Also run the existing suite: 240 passed, 2 skipped on the rebased base.

## Deferred Work

- `info` is not restored through the dict form. See above.
- `ColorStops.as_dict`/`_json_encode` are untouched; only `Colormap` is in scope.
- `to_mpl` and the other converters still cannot express the finer classes. Unchanged by this PR.

## Refinement record, codex plan review 2026-08-13

| ID | Finding | Disposition | Change |
|---|---|---|---|
| B1 | Routing pickle through `as_dict()` drops `ColorStops._lut_func` and changes later sampling | accepted | `__reduce__` passes `color_stops` plus `_constructor_kwargs()`; `as_dict()` is no longer the pickle payload. Verified: as_dict round trip loses the callable and shifts `lut(17, gamma=2)` by 2.1e-5; pickle round trip is exact. Regression test added |
| B2 | `__eq__` is an unsound compact-name guard: it ignores name/identifier/category, and `ColorStops.__eq__` uses `np.allclose` | accepted | Guard compares `as_dict()` exactly. Verified: a renamed viridis and a `start=0.500001` cubehelix both compare equal under `__eq__` and unequal under `as_dict()` |
| N1 | The plan claims "full state" but knowingly drops `info` | accepted | Claim narrowed to constructor state; `info` and serialized shape named as out of scope in the plan and the PR body |
| N2 | Swapping resolved `under`/`over` in the `rev` branch would also swap explicit overrides | accepted | Swap the record's values before override resolution; test with an explicit `under=` on an `_r` name |
| N3 | Identifier rule is settled without maintainer agreement; double reversal loses an explicit identifier | revised | Not blocking on a question. The rule and the double-reversal consequence go in the PR body for the maintainer to overrule |

## Refinement record, codex implementation review 2026-08-13

B1, B2, N1, N2, and N3 all assessed Resolved. One new finding:

| ID | Finding | Disposition | Change |
|---|---|---|---|
| N4 | The changed serialization paths lack coverage for the parametrized-callable case that motivates the exact `as_dict()` guard, and the psygnal path never exercises the new optional keys | accepted | Two assertions added: a `start=0.500001` cubehelix round trips through pydantic with its `as_dict()` intact, and a configured colormap round trips through `psygnal.EventedModel`. Both verified red against the base source |

No open disagreements. Reviewer: Codex, gpt-5.6-sol at xhigh effort, both passes.

## Next Steps

1. Implementation on `fix/state-preservation`, then codex implementation review.
2. Draft PR declaring "Depends on #151". #151 itself needs a force-push after the rebase, which
   needs the owner's authorization.
