Closes #157.

Adds a read-only `ColorStops.lut_func` property so colormaps defined by a function (prism, flag, gnuplot, cubehelix, ...) can be evaluated exactly at arbitrary positions, rather than through the 256-entry LUT that `Colormap.__call__` samples. It returns the stored callable unwrapped, so output is not clipped and may be RGB; it is `None` for stop-defined colormaps. For a reversed `ColorStops` it returns the existing picklable `partial`, which evaluates `f(1 - x)`.

```python
cm = Colormap("prism")
cm.color_stops.lut_func(np.array([0.1234]))  # exact, not LUT-sampled
```

Also corrects the `lut_func` parameter docs: the callable receives an `(N,)` array (it is called with `np.linspace`), not `(N, 1)`, and may return RGB or RGBA.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
