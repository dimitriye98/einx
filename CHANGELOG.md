# Changelog

## [Unreleased]

### Added

- Add partial support for Apple's [mlx](https://github.com/ml-explore/mlx).
  - Supported:
    - `einx.rearrange`
    - `einx.{elementwise|add|multiply|where|...}`
    - `einx.{reduce|sum|mean|...}`
    - `einx.{vmap_with_axis|flip|softmax|...}`
  - Not supported yet:
    - `einx.dot` (`mx.einsum` is not implemented yet)
    - `einx.vmap` (`mx.vmap` does not fully support all primitives yet)
    - `einx.{index|get_at|set_at|...}` (due to relying on `einx.vmap`)
- Add partial support for [dask.array](https://docs.dask.org/en/stable/array.html).
  - Supported:
    - `einx.rearrange`
    - `einx.{elementwise|add|multiply|where|...}`
    - `einx.{reduce|sum|mean|...}`
    - `einx.{vmap_with_axis|flip|softmax|...}`
    - `einx.dot`
  - Not supported:
    - `einx.vmap` (`vmap` not implemented in dask)
    - `einx.{index|get_at|set_at|...}` (due to relying on `einx.vmap`)
- Add environment variable `EINX_WARN_ON_RETRACE` to warn when excessive retracing is detected.

### Changed

- Allow `->` and `,` to be composed with other operators. (This deprecates the existing `[|]` notation which should instead be implemented with
  composable `->`. The feature is still maintained for backwards compatibility). For example:
    - `einx.dot("b [c1->c2]", ...)` expands to `einx.dot("b [c1] -> b [c2]", ...)`
    - `einx.get_at("b p [i,->]", ...)` expands to `einx.get_at("b p [i], b p -> b p", ...)`
- Allow `einx.{set_at|add_at|...}` to be called with zero-sized updates or coordinates (in which case the input tensor is returned as-is).
- Remove `backend.dot` which was not used anywhere but in the unit tests.
- Improve error reporting:
  - Drop internal stack frames when raising exceptions.
  - Better error when passing invalid shape constraints to einx functions.
- Reduce overhead of einx when using the PyTorch backend.

### Fixed

- Fix compatibility of `einx.nn.torch.Norm` with PyTorch 2.2.
- Fix parameters in `einn.param` being ignored.
- Fix bug when using concatenations in `einx.rearrange`. See: https://github.com/fferflo/einx/issues/6
- Fix broadcasting new axes in `einx.vmap_with_axis`.


## [0.1.3]

### Added

- Add option to install einx via `pip install einx[torch]` or `pip install einx[keras]` to enforce version requirements on PyTorch or Keras.

### Changed

- Fail gracefully and report error when run with incompatible version of PyTorch and Keras.

### Fixed

- Fix compatibility with 2.0 <= PyTorch < 2.1.



## [0.1.2]

### Added

- Add type annotations to public API.
- Allow passing multiple coordinate tensors in `einx.{get_at|set_at|...}`.
- Allow implicit output shape in `einx.{set_at|add_at|...}`.
- Allow passing backend with string argument to `einx.nn.norm`.
- Make backends accessible as `einx.backend.{NAME}` once they are loaded.

### Changed

- Refactor tracing:
    - Trace vmapped functions (previously kept a pointer to an untraced function).
    - Add shape assertion when calling unsafe functions.
    - Add comments for better inspection.
    - Remove `pass_backend` argument from `einx.vmap`.
    - Cache different functions for different backends.
    - Don't call `backend.to_tensor` if input already has correct type.

  For example, tracing `einx.get_at` now gives the following jit-compiled code:
    ```python
    >>> print(einx.get_at("b [h w] c, b p [2] -> b p c", x, y, graph=True))
    # backend: einx.backend.numpy
    def op1(i0, i1):
        x1 = i1[:, 0]
        x2 = i1[:, 1]
        x0 = backend.get_at(i0, (x1, x2))
        return (x0,)
    def op0(i0, i1, op1=op1):
        op2 = backend.vmap(op1, in_axes=(0, 0), out_axes=(0,))
        op3 = backend.vmap(op2, in_axes=(3, None), out_axes=(2,))
        x0 = op3(i0, i1)
        return x0[0]
    ```

### Fixed

- Fix bug when using "1" as coordinate axis in einx.index.
- Add workaround for scalar indexing operations with torch.vmap (see https://github.com/pytorch/functorch/issues/747).
- Fix support for list/ tuple arguments as tensors with non-trivial shape.
- Change einx.reduce to accept only single tensors as arguments (API allowed multiple arguments, but was not implemented).
- Don't trace and jit functions if EINX_CACHE_SIZE=0.
- Fix bug where some static code analysis tools fail to recognize function specializations.