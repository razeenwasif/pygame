## Issue Overview
- **GitHub reference**: https://github.com/pygame/pygame/issues/4163  
- **Observed behaviour**: Assigning a Python `list` as the color when targeting
a single PixelArray element (`px[x][y] = [r, g, b]` or `px[x, y] = [...]`) 
crashed the interpreter with a segmentation fault.  
- **Expected behaviour**: The target pixel should be mapped to the supplied 
color, consistent with the PixelArray assignment rules documented in 
`docs/reST/ref/pixelarray.rst`.

## Reproduction & Failure Analysis
1. Initial reproduction used the snippet from the issue report on a freshly 
built pygame (`python3 setup.py build_ext -i`).  
2. Attaching gdb showed the crash occurred after `_pxarray_ass_item` delegated 
to `_array_assign_sequence`, which in turn attempted to iterate over a 
zero-width temporary slice created for the single pixel. The resulting pointer 
arithmetic stepped outside the locked surface buffer, producing the segfault.  
3. Tracing the control flow revealed `_get_color_from_object` rejects non-tuple 
sequences, so list colors fell through to the sequence-assignment path. 
That path expects per-column sequences (see lines 24–47 of 
`docs/reST/ref/pixelarray.rst`), so the empty temporary slice led to the crash.

## Implementation Details
- Updated `_pxarray_ass_item` (`src_c/pixelarray.c:1320`) to detect 1×1 views 
(`shape[1] == 0` for chained indexing and `(shape[0], shape[1]) == (1, 1)` for 
tuple indexing). For those cases, non-tuple sequences of length 3 or 4 are 
converted to RGBA via `pg_RGBAFromObj`, then mapped through `SDL_MapRGBA`.  
- The guard keeps legacy behaviour for wider assignments (`px[x] = [values...]`)
intact; those still route through `_array_assign_sequence`, preserving the 
documented requirement that sequence lengths match the PixelArray width.  
- Added defensive handling so any `PySequence_Size` error bubbles up 
immediately instead of continuing with an invalid state.

## Regression Coverage
- Added `PixelArrayTypeTest.test_single_pixel_sequence_color` in 
`test/pixelarray_test.py:1235`. The test exercises both chained (`px[x][y]`) 
and tuple (`px[x, y]`) indexing with list colors to confirm the fix and prevent 
regressions.
- Followed the test structure guidance from `test/README.rst` 
(importing `pygame.tests.test_utils` at the top and nesting assertions inside 
`unittest.TestCase` methods).

## Validation & Commands
- Built extensions in-place to ensure C changes were active:
  ```
  python3 setup.py build_ext -i
  ```
- Confirmed interactive repro succeeds without crashing:
  ```python
  import sys, importlib.util
  spec = importlib.util.spec_from_file_location(
      "pygame",
      "src_py/__init__.py",
      submodule_search_locations=["src_py", "build/lib.linux-x86_64-cpython-313/pygame"],
  )
  pygame = importlib.util.module_from_spec(spec)
  sys.modules["pygame"] = pygame
  spec.loader.exec_module(pygame)
  surf = pygame.Surface((10, 10))
  px = pygame.PixelArray(surf)
  px[3, 4] = [255, 128, 0]   # no segfault, pixel updated
  ```
- Verified the entire `pixelarray_test` suite using the reusable helper script 
`scripts/run_pixelarray_tests.py`, which auto-detects the built package 
directory and encapsulates the same manual loading logic described above 
in line with `docs/reST/ref/tests.rst`.
- Introduced a dedicated CI job (`pixelarray-regression` in 
`.github/workflows/format-lint.yml`) that performs an in-tree build and 
executes the helper script on every PR/push that touches source files.
  ```

## Repository Guidelines & References
- **PixelArray behaviour**: `docs/reST/ref/pixelarray.rst` documents that 
single-pixel assignments accept tuples and colours; the fix extends this to 
lists while honouring the documented width-matching rule for sequences.  
- **Testing conventions**: `test/README.rst` outlines the naming and import 
patterns used when adding new tests; the regression test follows those 
instructions.  
- **Test execution guidance**: `docs/reST/ref/tests.rst` explains the expected 
entry points (`python -m pygame.tests` and `run_tests.py`); the custom harness 
adapts those instructions for an in-tree build and is now automated via 
`scripts/run_pixelarray_tests.py` and the `pixelarray-regression` workflow job.  
- **Contribution workflow**: `README.rst` (sections “Building From Source” 
and linked “Documentation Contributions”) provided the baseline for 
building extensions locally before running tests.

## Contribution Summary
- Investigated and reproduced the PixelArray segfault, traced it to 
`_pxarray_ass_item`, and implemented a guarded colour conversion path for 
single-pixel writes in `src_c/pixelarray.c`.  
- Added `PixelArrayTypeTest.test_single_pixel_sequence_color` under 
`test/pixelarray_test.py` to capture the regression.  
- Documented the full analysis, decision points, guidelines consulted, and 
verification steps in this report to streamline reviewer onboarding for pull request.
