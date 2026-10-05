`test_napari_name_parity` fails with `AttributeError` on every CI leg that resolves napari 0.9 (currently the 3.11 to 3.13 legs, e.g. on #159). napari 0.9 renamed the private `colormap_utils._VISPY_COLORMAPS_ORIGINAL` to `_VISPY_COLORMAPS`, with the same contents, and dropped the `_MATPLOTLIB_COLORMAP_NAMES` alias.

This looks up the vispy dict under the new name with a fallback for napari < 0.9, and reads the matplotlib names from the public `matplotlib_colormaps`, which exists from napari 0.5 through 0.9. Test-only; passes on napari 0.5.0, 0.8.0 and 0.9.2, and every napari name (now including `fire` and `ice`) is still in the catalog.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
