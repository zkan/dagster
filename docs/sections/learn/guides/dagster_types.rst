Dagster Types
-------------

Every dependency between solids in dagster is attached to an input and output.
Each of these inputs and outputs are annotated with a type within the dagster
type system. The type system is *gradual* and *optional*. Inputs and outputs
default to the ``Any`` type.

The dagster type system is independent from the PEP 484 Python type system,
although we overload the type annotation syntax on functions to make it
easier to specify the input and output types of your solids.

A solid's metadata are meant to be self-describing and reusable. A solid, ideally, communicates
through metadata the conditions required in order for successful execution.
Put another way, through metadata it describes the state of the external world
that must exist for the computation to succeed.

These conditions are expressed through three different abstractions:

1. Environment: Solids communicate their environment requirements through `required_resource_keys`. Examples of environment requirements include a database connections or a spark context. Resources specified in this way are accessible via the `context` object passed to the body of solids.
2. Configuration: Solids declare a schema for their configuration. Configuration expresses parameters that are control or behavioral concerns which would never be produced by previous computation.
3. Data Requirements: Solids ingest data and produce data. Inputs represent either existing data whose value are managed by dagster's intermediate store -- such as a dataframe -- or the representation of data not managed by dagster's intermediate store -- such as a database table.

The type system and the input/out system is designed to capture and model the
third category: data requirements.

Dagster types pass or fail typechecks based on arbitrary, user-defined computation.
This allows types to express data requirements with maximum flexibility, going
far beyond the capabilities of Python's builtin types. This flexibility has
costs, as it does not guarantee the ability of types to verify compatibility
prior to computation.

Features of Dagster Types:

- **Gradual and Optional**: Typechecks are use-defined computation that is arbitrarily strict.
- **Self-Describing**: Types include a description and metadata, accessible via an API and, therefore, tooling.
- **Configuration**: Inputs modeled Dagster types can be hydrated from existing via configuration, and similarly outputs can be materialized to user-controlled locations. Types declare their config schema requires and define a function that translates the config value into an in-memory value from existing data made available as an input.
- **API**: All metadata is accessible over an API and available for tool building.
- **Compute-time Metadata**: Type checks can return structured metadata -- e.g. summary statistics -- at runtime, also accessible over an API. 
- **Serialization**: Types determine serialization behavior, used by dagster's intermediate store infrastructure to pass data between solids that reside in different processes or nodes at runtime.

DagsterType
^^^^^^^^^^^

The core API for creating dagster types is DagsterType.

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 28-31
   :caption: create_type_api.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

Once created, this type can be attached to solids via either an a)
``InputDefinition`` and ``OutputDefinition`` or b) type annotations. 

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 51-56
   :caption: types_on_definitions.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

Using python 3 and type annotations, the exact same declaration can be expressed:

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 33-35
   :caption: type_annotations.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

When raw DagsterTypes are passed, this is not compliant with mypy or other python
static type-checking system (we will refer to just mypy throughout the remainder of this
document for convenience). We detail how to do that later in this document.

The type_check_fn must succeed by returning  either ``True`` or a ``TypeCheck`` object
with ``success`` set to True for the type check to pass. If either the input value or
the output value fails the associated typecheck, then the solid will be marked as a
failure and any downstream solids will not be executed.

Scalars
~~~~~~~

Dagster exports top level scalars -- ``Int``, ``Bool``, ``String``, ``Float`` -- that can be used
as DagsterType instances. For convenience, one can use the corresponding python
built-in scalars -- ``int``, ``bool``, ``str``, ``float`` --  to specify the corresponding
dagster scalar types. This also makes these arguments mypy-compliant.
There is a 1:1 relationship between the python type -- e.g. ``int`` -- and its corresponding
dagster type -- e.g. ``dagster.Int``
and a mapping between them is maintained by the system. This means that those python types
can be used in InputDefinitions or OutputDefinitions. If passed as type annotation to solids,
the corresponding DagsterType will also be picked up by the system.

As we will see later, the user can also define their own 1:1 relationships between python 
and dagster types.

PythonObjectDagsterType
^^^^^^^^^^^^^^^^^^^^^^^

Although type checks are very flexible, it is also common to define types that
just do `isinstance` checks as the dagster type check. We provided an API to do
just that.

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 67-72
   :caption: python_object_dagster_type.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python


One can now declare the dagster type as the interface to the solid, and use the
corresponding python type in business logic.

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 74-80
   :caption: dagster_type_annotations.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

The examples above are contrived. The type system truly shines once the type check
expresses richer behavior, such as column-level schema on a dataframe. The
dagster-pandas library is an example of that capability (link).

Configurability
^^^^^^^^^^^^^^^

Solids are parameterized by their inputs. When executing a pipeline or solid whose data are
not produced by a dependency -- very common for a solid at the beginning of a pipeline
or execution subset -- the data need to come from somewhere. This is specified
by config.

The type must declare its interaction with the config system. This means declaring
the input schema and a function which the validated config value and produces
a valid in-memory value to be passed to the schema. This process is referred to
as hydration.

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 86-99
   :caption: input_hydration_config.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

With this the input can be specified via config as below:

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 101-108
   :caption: execute_with_yaml.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

Output values can also be materialized to a user-controlled location using config in
a similar fashion.


Dagster Type Use Cases
^^^^^^^^^^^^^^^^^^^^^^

Dagster types are checked via arbitrary, user-defined computation, and can be used
to cover a wide variety of use cases common within data applications.

Inputs and Outputs can represent, via the type system:

1) **In-Memory-Data**: In-memory data that is either produced a previous computation or specified via configuration. Dagster infrastructure -- the intermediates store -- handles the serialization the transfer between of these data in cases whereas multiple processes or nodes are being used to run the orchestration cluster.

2) **Metadata**: Metadata about data that is managed by the user but not the intermediates store. The type system would ensure that the in-memory metadata format is correct and that any preconditions about the in situ data are met. An example of this would be a data lake with a user-defined URL scheme.

3) **Preconditions**: Preconditions about external state necessary for computation to succeed. An example of this is a hard-coded database table name. In some pipelines, parameterizing this is not necessary, as testability might be achieved by swapping out an environmental concern, such as a database instance. 

4) **Execution Ordering**: In some pipelines there is no data to pass in between solids and there are no preconditions. This is a common in cases where one is migrating workflows from other systems -- that do not have this concept -- to dagster, or where the upstream solids are purely operational whose effects are difficult or awkward to model. For these cases we offer the ``Nothing`` type. 

Execution Order Dependencies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes one wishes to declare a dependency that specifies execution order
and does not encode a data dependency. This is often useful when porting
workflows into dagster from other systems with no notion of data dependencies,
or in cases where it is awkward to express a data dependency.

Use of the ``Nothing`` type is used, typically point to a situation in which
although there does exist some semantic dependency between two solids,
usually to do with external state.  An example would be a database
table created in an upstream solid with hard-coded name.

In general, a ``Nothing`` dependency is in actuality an unexpressed dependency
on external state, making a pipeline harder to understand, and solids within
that pipeline harder to test and reuse.


We also believe that any purely operational and environment concern
means that there are missing APIs or abstractions in Dagster itself. Our goal
is for as many solids and pipelines and possible to be expressed in
terms of pure business logic. 

Here is an example usage of the ``Nothing`` type:

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 180-194
   :caption: serial_nothing_pipeline.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

Note that the input with the ``Nothing`` type is not passed to the
solid definition function. It is also not possible to use type
annotations to specify inputs and outputs with the ``Nothing`` type.
:py:class:`InputDefinition` and ``OutputDefinition`` must be used.

``Nothing`` is also useful when fanning in multiple dependencies. An
input with a ``Nothing`` type can depend on multiple outputs upstream.
(Note: this is also true for dagster.List)

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 198-233
   :caption: fanin_nothing_pipeline.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

Python Types and Dagster Types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As alluded to earlier, the user can create define their own 1:1 relationships
between dagster types and Python types, making the corresponding python type usable
where the dagster type is expected.

This is convenient when one has business objects that require nothing more
than an ``isinstance`` check in the dagster type check *and* one wishes
to use them both in type annotations directly and as business objects. These type
definitions reduce boilerplate as well as deliver out-of-the-box mypy
compliance.

There are two APIs: :py:func:`usable_as_dagster_type` -- for direct annotations of class
declarations -- and :py:func:`make_python_type_usable_as_dagster_type` -- for mapping *existing*
classes. This is designed for importing python types libraries that cannot
be altered and mapping them to dagster types.


.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 267-277
   :caption: usable_as_dagster_type.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

And make_python_type_usable_as_dagster_type

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 281-294
   :caption: make_python_type_usable_as_dagster_type.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

This approach does have downsides. Importing these types will causes a global side
effect as there is an internal registry which manages these types. This can
introduce challenges in terms of testability and also can causes in behavior
based on import order.

Additionally some of the most useful patterns in the ecosystem are
to use type factories to programmatically create dagster types, such
as in our dagster-pandas module. In these cases a 1:1 mapping between
dagster type and python type *no longer exists*. E.g. in
dagster-pandas the python representation for all the dataframe
variants is simply `pandas.DataFrame`.

For clearly scoped business objects, the reduction in boilerplate is significant
and eases mypy compliance, hence its inclusion in the public dagster API.

We do *not* recommend that libraries use this pattern and instead rely on other
techniques to achieve mypy compliance.

MyPy Compliance
~~~~~~~~~~~~~~~

Users who do not use "usable-as-dagster" python types types by choice or necessity 
and who desire mypy compliance need additional support. 

This is a challenge to do elegantly in light of the fact that as of mypy 0.761 there
is no way to directly annotate a object to treat it as a type during a type checking
pass. However there is a way to do this.

.. literalinclude:: ../../../../examples/dagster_examples_tests/intro_tutorial_tests/test_type_guide.py
   :lines: 162-174
   :caption: type_checking_mypy_compliance.py
   :linenos:
   :lineno-start: 1
   :dedent: 4
   :language: python

While not particularly elegant, this does work. We recommend centralizing
type definitions so that this style of definition is compartmentalized to
a single file.