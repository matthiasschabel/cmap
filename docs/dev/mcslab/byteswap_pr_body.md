Data in non-native byte order maps to the wrong colors.

- On a little-endian host, a big-endian float array has its bytes relabelled rather than
  reordered, so `-0.5`, `1.5` and `NaN` all decode as tiny denormals.
- They land on the first ramp color, which makes `under`, `over` and `bad` unreachable for
  such input.

```python
import numpy as np
from cmap import Colormap, Color

cmap = Colormap(["red", "blue"], under="green", over="yellow", bad="black")
native = np.array([-0.5, 0.5, 1.5, np.nan])
non_native = native.astype(native.dtype.newbyteorder())

print([Color(c).hex for c in cmap(native)])
print([Color(c).hex for c in cmap(non_native)])
```

```
['#008000', '#7F0080', '#FFFF00', '#000000']   # under, ramp, over, bad
['#FF0000', '#FF0000', '#FF0000', '#FF0000']   # all the first ramp color
```

The numpy 2 migration in #60 replaced `xa.byteswap().newbyteorder()`, which numpy 2 removed,
with a plain `.view()` of the swapped dtype. `byteswap().view(dtype.newbyteorder())` is the
numpy 2 spelling of the original, and what matplotlib uses today.

The existing coverage maps an array of zeros and asserts only its shape, both of which are
invariant under the bug, which is why it stayed hidden.
