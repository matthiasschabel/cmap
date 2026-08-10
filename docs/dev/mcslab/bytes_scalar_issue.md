* cmap version: 0.7.2 (also on main, 0.7.3.dev11+g02fa073)
* Python version: 3.12.12
* Operating System: macOS 26.5.1 (arm64), numpy 2.4.6

### Description

`Colormap.__call__` with a scalar and `bytes=True` always raises. Non-scalar array input in
byte mode works and scalar input in float mode works, so it is only the combination that
fails.

Byte mode converts the LUT to `uint8`, and scalar input wraps the result in `Color`, which
accepts a 3-element integer array but not a 4-element one (`parse_rgba` in `_color.py`).

Two parts of the contract disagree about what should come back, which is why this is an issue
rather than a PR:

- the scalar overload and the docstring promise a `Color` whatever `bytes` is set to,
- the `bytes` documentation promises `uint8` values in 0-255.

So either scalar byte mode returns a `(4,)` `uint8` array and the overloads split on `bytes`,
or it quantizes to 8 bits and returns a `Color`, which keeps the scalar contract and makes
`bytes=True` visible only in the rounding (`Color(rgba8 / 255)` gives `#20908C` against
`#21918C` in float mode). I do not have a stake in which one wins. Happy to send the PR for
whichever you prefer.

### What I Did

```python
import numpy as np
from cmap import Colormap

cmap = Colormap("viridis")
print(cmap(np.array([0.5]), bytes=True))   # [[ 32 144 140 255]]
print(cmap(0.5))                           # #21918C
print(cmap(0.5, bytes=True))               # raises
```

```
Traceback (most recent call last):
  File "<string>", line 7, in <module>
  File "/tmp/cmap-072/src/cmap/_colormap.py", line 420, in __call__
    return rgba if np.iterable(x) else Color(rgba)
                                       ^^^^^^^^^^^
  File "/tmp/cmap-072/src/cmap/_color.py", line 529, in __new__
    rgba = parse_rgba(value)
           ^^^^^^^^^^^^^^^^^
  File "/tmp/cmap-072/src/cmap/_color.py", line 467, in parse_rgba
    raise ValueError(f"Invalid color array: {value!r}")  # pragma: no cover
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ValueError: Invalid color array: array([ 32, 144, 140, 255], dtype=uint8)
```

A Python `float` or `int`, a numpy scalar, and a 0-d array all take this path, since the
return branches on `np.iterable(x)`.
