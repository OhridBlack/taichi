Internal designs (WIP)
======================

Intermediate representation
---------------------------
Use ``ti.init(print_ir=True)`` to print IR on the console.


Data Structure Node Organization
--------------------------------

The internal organization of data structure nodes can be confusing.
It is important to distinguish the concept of **containers**, **cells** and **components**.

- A **container** can have multiple **cells**. The numbers of **cells** are recommended to be powers of two.
- A **cell** can have multiple **components**.
- Each **component** is a **container** of a lower-level SNode.

Consider the following example:

.. code-block:: python

    # misc/listgen_demo.py

    x = ti.var(ti.i32)
    y = ti.var(ti.i32)
    z = ti.var(ti.i32)

    S0 = ti.root
    S1 = S0.pointer(ti.i, 4)

    S2 = S1.dense(ti.i, 8)
    S2.place(x, y) # S3: y; S4: z

    S5 = S1.dense(ti.i, 8)
    S5.place(z) # S6: z


- The whole data structure is a ``S0root`` **container**, containing

  - ``1x`` ``S0root`` **cell**, each has only one **component**, which is

    - A ``S1pointer`` **container**, containing

      - 4x ``S1pointer`` **cells**, each with two **components**, which are

        - A ``S2dense`` **container**, containing

          - 8x ``S2dense`` **cells**, each with two **components**, which are

            - ``x: ti.i32``
            - ``y: ti.i32``

        - A ``S5dense`` **container**, containing

          - 8x ``S5dense`` **cells**, each with one **component**, which is

            - ``z: ti.i32``


Each SNode has two types (as implemented in struct compilers): ``container`` type and ``cell`` type.
Note that **cells** are never exposed to end-users.
**Components** of a higher level SNode **cells** are **containers** of a lower level SNode.

We are on our way to remove usages of **children**, **instances** and **elements** in Taichi.
These are very ambiguous terms.

**List generation** generates lists of SNode **containers** (instead of SNode **cells**).


List generation (WIP)
---------------------

Struct-fors in Taichi loop over all active elements of a (sparse) data structure **in parallel**.
Evenly distributing work onto processor cores is challenging on sparse data structures:
naively splitting an irregular tree into pieces can easily lead to
partitions with drastically different numbers of leaf elements.

Our strategy is to generate lists of active SNode elements layer by layer.
The list generation computation happens on the same device as normal computation kernels,
depending on the ``arch`` argument when the user calls ``ti.init``.

List generations flatten the data structure leaf elements into a 1D dense array, circumventing the irregularity of
incomplete trees. Then we can simply invoke a regular **parallel for** over the list.

For example,

.. code-block:: python

    # misc/listgen_demo.py

    import taichi as ti

    ti.init(print_ir=True)

    x = ti.var(ti.i32)

    S0 = ti.root
    S1 = S0.dense(ti.i, 4)
    S2 = S1.bitmasked(ti.i, 4)
    S2.place(x)

    @ti.kernel
    def func():
        for i in x:
            print(i)

    func()


gives you the following IR:

.. code-block:: none

  $0 = offloaded clear_list S1dense
  $1 = offloaded listgen S0root->S1dense
  $2 = offloaded clear_list S2bitmasked
  $3 = offloaded listgen S1dense->S2bitmasked
  $4 = offloaded struct_for(S2bitmasked) block_dim=0 {
    <i32 x1> $5 = loop index 0
    print i, $5
  }


Note that ``func`` leads to two list generations:

 - (Tasks ``$0`` and ``$1``) based on the list of ``root`` node (``S0``), generate the list of the ``dense`` nodes (``S1``);
 - (Tasks ``$2`` and ``$3``) based on the list of ``dense`` nodes (``S1``), generate the list of ``bitmasked`` nodes (``S2``).

The list of ``root`` node always has exactly one element (instance), so we never clear or re-generate this list.

.. note::

    The list of ``place`` (leaf) nodes (e.g., ``S3`` in this example) is never generated.
    Instead, we simply loop over the list of their parent nodes, and for each parent node we
    enumerate the ``place`` nodes on-the-fly (without actually generating a list).

    The motivation for this design is to amortize list generation overhead.
    Generating one list element per leaf node (``place`` SNode) element is too expensive, likely much
    more expensive than the essential computation happening on the leaf element.
    Therefore we only generate their parent element list, so that the list generation cost is
    amortized over multiple child elements of a second-to-last-level SNode element.

    In the example above, although we have ``16`` instances of ``x``,
    we only generate a list of ``4`` ``bitmasked`` nodes (and ``1`` ``dense`` node).


Code generation
---------------


Statistics
----------

In some cases, it is helpful to gather certain quantitative information about internal events during
Taichi program execution. The ``Statistics`` class is designed for this purpose.

Usage:

.. code-block:: C++

    #include "taichi/util/statistics.h"

    // add 1.0 to counter "codegen_offloaded_tasks"
    taichi::stat.add("codegen_offloaded_tasks");

    // add the number of statements in "ir" to counter "codegen_statements"
    taichi::stat.add("codegen_statements", irpass::analysis::count_statements(this->ir));


Note the keys are ``std::string`` and values are ``double``.

To print out all statistics in Python:

.. code-block:: Python

    ti.core.print_stat()


Why Python frontend
-------------------

Embedding Taichi in ``python`` has the following advantages:

* Easy to learn. Taichi has a very similar syntax to Python.
* Easy to run. No ahead-of-time compilation is needed.
* This design allows people to reuse existing python infrastructure:

  * IDEs. A python IDE mostly works for Taichi with syntax highlighting, syntax checking, and autocomplete.
  * Package manager (pip). A developed Taichi application and be easily submitted to ``PyPI`` and others can easily set it up with ``pip``.
  * Existing packages. Interacting with other python components (e.g. ``matplotlib`` and ``numpy``) is just trivial.

* The built-in AST manipulation tools in ``python`` allow us to do magical things, as long as the kernel body can be parsed by the Python parser.

However, this design has drawbacks as well:

* Taichi kernels must parse-able by Python parsers. This means Taichi syntax cannot go beyond Python syntax.

  * For example, indexing is always needed when accessing elements in Taichi tensors, even if the tensor is 0D. Use ``x[None] = 123`` to set the value in ``x`` if ``x`` is 0D. This is because ``x = 123`` will set ``x`` itself (instead of its containing value) to be the constant ``123`` in python syntax, and, unfortunately, we cannot modify this behavior.

* Python has relatively low performance. This can cause a performance issue when initializing large Taichi tensors with pure python scripts. A Taichi kernel should be used to initialize a huge tensor.
