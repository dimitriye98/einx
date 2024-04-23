Tutorial: Tensor manipulation
#############################

Overview
--------

einx supports a wide variety of tensor operations. In each function, Einstein expressions are used to specify how the operation should be performed.
einx internally parses the expressions to determine the required steps and forwards computation to the respective backend, e.g. by
calling `np.reshape <https://numpy.org/doc/stable/reference/generated/numpy.reshape.html>`_,
`np.transpose <https://numpy.org/doc/stable/reference/generated/numpy.transpose.html>`_ or 
`np.sum <https://numpy.org/doc/stable/reference/generated/numpy.sum.html>`_ with the appropriate arguments.

The most basic operation in einx is :func:`einx.rearrange` which transforms tensors between Einstein expressions by reshaping, permuting axes, inserting new
broadcasted axes, concatenating and splitting as required. All other functions support the same rearranging of expressions, but additionally perform some
operation on the tensors' values. For ease-of-use, most of these follow a Numpy-like naming convention:

* ``einx.{sum|prod|mean|any|all|max|min|count_nonzero|...}`` (see :func:`einx.reduce`).
* ``einx.{add|multiply|logical_and|where|equal|...}`` (see :func:`einx.elementwise`).
* ``einx.{flip|roll|softmax|...}`` (see :func:`einx.vmap_with_axis`).
* ``einx.{get_at|set_at|add_at|...}`` (see :func:`einx.index`).

Most functions in einx are specializations of the two main abstractions :func:`einx.vmap` and :func:`einx.vmap_with_axis` which allow applying arbitrary operations
using Einstein notation. Each class of functions additionally introduces shorthand notations that allow for a concise and expressive formulation of the respective
operations.

This tutorial gives an overview of most functions and their usage. For a complete list of available functions, see the :doc:`API reference </api>`.

Rearranging
-----------

The function :func:`einx.rearrange` transforms tensors between Einstein expressions by determining and applying the required backend operations. For example:

>>> x = np.ones((4, 256, 17))
>>> y, z = einx.rearrange("b (s p) (c + 1) -> (b s) p c, (b p) s 1", x, p=8)
>>> y.shape, z.shape
((128, 8, 16), (32, 32, 1))

Using :func:`einx.rearrange` often produces more readable and concise code than specifying backend operations in index-based notation directly. The index-based calls can be
inspected using the just-in-time compiled function that einx creates for this expression (see :doc:`Just-in-time compilation </gettingstarted/jit>`):

>>> print(einx.rearrange("b (s p) (c + 1) -> (b s) p c, (b p) s 1", x, p=8, graph=True))
import numpy as np
def op0(i0):
    x0 = np.reshape(i0, (4, 32, 8, 17))
    x1 = np.reshape(x0[:, :, :, 0:16], (128, 8, 16))
    x2 = np.reshape(x0[:, :, :, 16:17], (4, 32, 8))
    x3 = np.transpose(x2, (0, 2, 1))
    x4 = np.reshape(x3, (32, 32, 1))
    return [x1, x4]

Reduction ops
-------------

einx provides a family of functions that reduce tensors along one or more axes. For example:

.. code::

   einx.sum("a [b]", x)
   # same as
   np.sum(x, axis=1)

   einx.mean("a [...]", x)
   # same as
   np.mean(x, axis=tuple(range(1, x.ndim)))

These functions are specializations of :func:`einx.reduce` and use backend operations like `np.sum <https://numpy.org/doc/stable/reference/generated/numpy.sum.html>`_,
`np.prod <https://numpy.org/doc/stable/reference/generated/numpy.prod.html>`_ or `np.any <https://numpy.org/doc/stable/reference/generated/numpy.any.html>`_ as the ``op`` argument:

.. code::

   einx.reduce("a [b]", x, op=np.sum)
   # same as
   einx.sum("a [b]", x)

In ``einx.sum``, the respective backend is determined implicitly from the input tensor (see :doc:`How does einx support different tensor frameworks? </faq/backend>`).

In the most general case, the operation string represents both input and output expressions, and marks reduced axes using brackets:

>>> x = np.ones((16, 8, 4))
>>> einx.sum("a [b] c -> a c", x).shape
(16,)

:func:`einx.reduce` supports shorthand notation as follows. When no brackets are found, brackets are placed implicitly around all axes that do not appear in the output:

.. code::

   einx.sum("a b c -> a c", x) # Expands to: "a [b] c -> a c"

When no output is given, it is determined implicitly by removing marked subexpressions from the input:

..  code::

   einx.sum("a [b] c", x) # Expands to: "a [b] c -> a c"

:func:`einx.reduce` also allows custom reduction operations that accept the ``axis`` argument similar to `np.sum <https://numpy.org/doc/stable/reference/generated/numpy.sum.html>`_:

.. code::

   def custom_mean(x, axis):
       return np.sum(x, axis=axis) / x.shape[axis]
   einx.reduce("a [b] c", x, op=custom_mean)

:func:`einx.reduce` fully supports Einstein expression rearranging:

>>> x = np.ones((16, 8))
>>> einx.prod("a (b [c]) -> b a", x, c=2).shape
(4, 16)

Element-by-element ops
----------------------

einx provides a family of functions that apply element-by-element operations to tensors. For example:

.. code::

   einx.add("a b, b -> a b", x, y)
   # same as
   x + y[np.newaxis, :]

   einx.multiply("a, a b -> a b", x, y)
   # same as
   x[:, np.newaxis] * y

   einx.subtract("a, (a b) -> b a", x, y)
   # requires reshape and transpose in index-based notation

Internally, the inputs are rearranged such that the operation can be applied using `Numpy broadcasting rules <https://numpy.org/doc/stable/user/basics.broadcasting.html>`_.
These functions are specializations of :func:`einx.elementwise` and use backend operations like `np.add <https://numpy.org/doc/stable/reference/generated/numpy.add.html>`_,
`np.logical_and <https://numpy.org/doc/stable/reference/generated/numpy.logical_and.html>`_ and `np.where <https://numpy.org/doc/stable/reference/generated/numpy.where.html>`_
as the ``op`` argument:

.. code::

   einx.elementwise("a b, b -> a b", x, y, op=np.add)
   # same as
   einx.add("a b, b -> a b", x, y)

In the most general case, the operation string of :func:`einx.elementwise` represents all input and output expressions explicitly:

>>> x = np.ones((16, 8))
>>> y = np.ones((16,))
>>> einx.add("a b, a -> a b", x, y).shape
(16, 8)

The output is determined implicitly if one of the input expressions contains the named axes of all other inputs and if this choice is unique:

.. code::

   einx.add("a b, a", x, y)         # Expands to: "a b, a -> a b"

   einx.where("b a, b, a", x, y, z) # Expands to "b a, b, a -> b a"

   einx.subtract("a b, b a", x, y)  # Raises an exception

   einx.add("a b, a b", x, y)       # Expands to: "a b, a b -> a b"

Bracket notation can be used to indicate that the second input is a subexpression of the first:

.. code::

   einx.add("a [b]", x, y) # Expands to: "a b, b"

:func:`einx.elementwise` fully supports Einstein expression rearranging:

>>> x = np.ones((16, 16, 32))
>>> bias = np.ones((4,))
>>> einx.add("b... (g [c])", x, bias).shape
(16, 16, 32)

Indexing ops
------------

einx provides a family of functions that perform multi-dimensional indexing and update/retrieve values from tensors at specific coordinates:

.. code::

   image = np.ones((256, 256, 3))
   coordinates = np.ones((100, 2), dtype=np.int32)
   updates = np.ones((100, 3))

   # Retrieve values at specific locations in an image
   y = einx.get_at("[h w] c, i [2] -> i c", image, coordinates)
   # same as
   y = image[coordinates[:, 0], coordinates[:, 1]]

   # Update values at specific locations in an image
   y = einx.set_at("[h w] c, i [2], i c -> [h w] c", image, coordinates, updates)
   # same as
   image[coordinates[:, 0], coordinates[:, 1]] = updates
   y = image

Brackets in the first input indicate axes that are indexed, and a single bracket in the second input indicates the coordinate axis. The length of the coordinate axis should equal
the number of indexed axes in the first input. Coordinates can also be passed in separate tensors:

.. code::

   coordinates_x = np.ones((100,), dtype=np.int32)
   coordinates_y = np.ones((100,), dtype=np.int32)

   y = einx.get_at("[h w] c, i, i -> i c", image, coordinates_x, coordinates_y)

Indexing functions are specializations of :func:`einx.index` and fully support Einstein expression rearranging:

.. code::

   einx.add_at("b ([h w]) c, ([2] b) i, c i -> c [h w] b", image, coordinates, updates)

Vectorization
-------------

Both :func:`einx.reduce` and :func:`einx.elementwise` are adaptations of :func:`einx.vmap_with_axis`. The purpose of :func:`einx.vmap_with_axis`
is to augment backend functions providing a numpy-like interface (e.g. ``np.sum``) such that they can be called using Einstein notation.
For exmaple, :func:`einx.sum` wraps ``np.sum`` using :func:`einx.vmap_with_axis`:

.. code::

   y = einx.sum("a [b]", x)
   # internally calls
   y = np.sum(x, axis=1)

Functions such as ``np.sum`` can be used with :func:`einx.vmap_with_axis` if they accept the ``axis`` argument (or work on scalars)
and follow `Numpy broadcasting rules <https://numpy.org/doc/stable/user/basics.broadcasting.html>`_ for multiple inputs.

The ``axis`` argument specifies axes that the operation is applied to, and the operation is repeated implicitly over all other dimensions.
In the above example, the sum is computed over elements in a row, and this is repeated for all rows.

A naive implementation without ``np.sum`` could simply loop over the first dimension manually to perform the same operation:

.. code::

   for r in range(x.shape[0]):
       y[r] = sum(x[r, :])

However, since Python loops are notoriously slow, Numpy provides the highly optimized *vectorized* implementation ``np.sum`` that allows specifying which dimensions to apply the operation
to, and which dimensions to vectorize/ "loop" over.

The bracket notation in Einstein expressions serves a similar purpose as the ``axis`` parameter: Operations are applied to 
axes that are marked with ``[]``, and other axes are vectorized over. :func:`einx.vmap_with_axis` takes care of vectorization by 
rearranging the inputs and outputs as required and determining the correct ``axis`` argument to pass to the backend function. This allows
applying operations to tensors with arbitrary Einstein expressions:

.. code::

   y = einx.sum("a ([b] c)", x, c=2)
   # cannot be expressed in a single call to np.sum
   y = np.sum(x, axis="?")

:func:`einx.vmap` allows for more general vectorization than :func:`einx.vmap_with_axis` by applying arbitrary functions in vectorized form. Consider a function that accepts two tensors
and computes the mean and max:

.. code::

    def op(x, y): # c, d -> 2
        return np.stack([np.mean(x), np.max(y)])

This function can be vectorized over a batch dimension as follows:

>>> x = np.ones((4, 16))
>>> y = np.ones((4, 8))
>>> einx.vmap("b [c], b [d] -> b [2]", x, y, op=op).shape
(4, 2)

:func:`einx.vmap` takes care of vectorization automatically such that the arguments arriving at ``op`` always match the marked subexpressions in the inputs. Analogously, the return
value of ``op`` should match the marked subexpressions in the output. :func:`einx.vmap` is implemented using efficient automatic vectorization in the respective backend (e.g. 
`jax.vmap <https://jax.readthedocs.io/en/latest/jax-101/03-vectorization.html>`_, `torch.vmap <https://pytorch.org/docs/stable/generated/torch.vmap.html>`_).

.. note::

    einx implements a simple ``vmap`` function for the Numpy backend for testing/ debugging purposes using a Python loop.

Analogous to other einx functions, :func:`einx.vmap` fully supports Einstein expression rearranging:

>>> x = np.ones((4, 16))
>>> y = np.ones((5, 8 * 4))
>>> einx.vmap("b1 [c], b2 ([d] b1) -> [2] b1 b2", x, y, op=op).shape
(2, 4, 5)

Since most backend operations that accept an ``axis`` argument operate on the entire input tensor when ``axis`` is not given, :func:`einx.vmap_with_axis` can often
analogously be expressed using :func:`einx.vmap`:

>>> x = np.ones((4, 16))
>>> einx.vmap_with_axis("a [b] -> a", x, op=np.sum).shape
(4,)
>>> einx.vmap          ("a [b] -> a", x, op=np.sum).shape
(4,)

>>> x = np.ones((4, 16))
>>> y = np.ones((4,))
>>> einx.vmap_with_axis("a b, a -> a b", x, y, op=np.add).shape
(4, 16)
>>> einx.vmap          ("a b, a -> a b", x, y, op=np.add).shape
(4, 16)

:func:`einx.vmap` provides more general vectorization capabilities than :func:`einx.vmap_with_axis`, but might in some cases be slower if the latter relies on a
specialized implementation.

Composability of ``->`` and ``,``
---------------------------------

The operators ``->`` and ``,`` that delimit input and output expressions can optionally be composed with other Einstein operations. If
they appear within a nested expression, the expression is expanded
`according to distributive law <https://en.wikipedia.org/wiki/Distributive_property>`_ such that ``->`` and ``,`` appear only at the root
of the expression tree. For example:

.. code::

   einx.vmap("a [b -> c]", x, op=..., c=16)
   # expands to
   einx.vmap("a [b] -> a [c]", x, op=..., c=16)

   einx.get_at("b p [i,->]", x, y)
   # expands to
   einx.get_at("b p [i], b p -> b p", x, y)

General dot-product
-------------------

The function :func:`einx.dot` computes general dot-products similar to `np.einsum <https://numpy.org/doc/stable/reference/generated/numpy.einsum.html>`_. It represents a special case
of vectorization since matrix multiplications using ``einsum`` are highly optimized in the respective backends.

In the most general case, the operation string is similar to that of ``einsum``. The inputs and outputs expressions are specified explicitly, and axes that appear in the input, but
not the output are reduced via a dot-product:

>>> # Matrix multiplication between x and y
>>> x = np.ones((4, 16))
>>> y = np.ones((16, 8))
>>> einx.dot("a b, b c -> a c", x, y).shape
(4, 8)

.. note::

    ``einx.dot`` is not called ``einx.einsum`` despite providing einsum-like functionality to avoid confusion with ``einx.sum``. The name is 
    motivated by the fact that the function computes a generalized dot-product, and is in line with expressing the same operation using :func:`einx.vmap`:

    .. code::

       einx.dot("a b, b c -> a c", x, y)
       einx.vmap("a [b], [b] c -> a c", x, y, op=np.dot)

:func:`einx.dot` fully supports Einstein expression rearranging:

>>> # Simple grouped linear layer
>>> x = np.ones((20, 16))
>>> w = np.ones((8, 4))
>>> einx.dot("b (g c1), c1 c2 -> b (g c2)", x, w, g=2).shape
(20, 8)

The graph representation shows that the inputs and output are rearranged as required and the dot-product is forwarded to the ``einsum`` function of the backend:

>>> print(einx.dot("b (g c1), c1 c2 -> b (g c2)", x, w, g=2, graph=True))
import numpy as np
def op0(i0, i1):
    x0 = np.reshape(i0, (20, 2, 8))
    x1 = np.einsum("abc,cd->abd", x0, i1)
    x2 = np.reshape(x1, (20, 8))
    return x2

Shorthand notation in :func:`einx.dot` is supported as follows. When given two input tensors, the expression of the second input is determined implicitly by marking
its components in the input and output expression:

.. code::

   einx.dot("a [b] -> a [c]", x, y) # Expands to: "a b, b c -> a c"

This dot-product can be interpreted as a linear map that maps from ``b`` to ``c`` channels and is repeated over dimension ``a``, which motivates the
usage of bracket notation in this manner.

Axes marked multiple times appear only once in the implicit second input expression:

.. code::

   einx.dot("[a b] -> [a c]", x, y) # Expands to: "a b, a b c -> a c"

The graph representation shows that the expression forwarded to the ``einsum`` call is as expected:

>>> x = np.ones((4, 8))
>>> y = np.ones((8, 5))
>>> print(einx.dot("a [b->c]", x, y, graph=True))
import numpy as np
def op0(i0, i1):
    x0 = np.einsum("ab,bc->ac", i0, i1)
    return x0

.. _lazytensorconstruction:

Tensor factories
----------------

All einx operations also accept tensor factories instead of tensors as arguments. A tensor factory is a function that accepts a ``shape``
argument and returns a tensor with that shape. This allows deferring the construction of a tensor to the point inside
an einx operation where its shape has been resolved, and avoids having to manually determine the shape in advance:

..  code::

    einx.dot("b... c1, c1 c2 -> b... c2", x, lambda shape: np.random.uniform(shape), c2=32)

In this example, the shape of ``x`` is used by the expression solver to determine the values of ``b...`` and ``c1``. Since the tensor factory provides no shape
constraints to the solver, the remaining axis values have to be specified explicitly, i.e. ``c2=32``.

Tensor factories are particularly useful in the context of deep learning modules: The shapes of a layer's weights are typically chosen to align with the shapes
of the layer input and outputs (e.g. the number of input channels in a linear layer must match the corresponding axis in the layer's weight matrix).
This can be achieved implicitly by constructing layer weights using tensor factories.

The following tutorial describes in more detail how this is used in einx to implement deep learning models.