.. index:: ! runtime

Runtime Structure
-----------------


.. _syntax-val:
.. index:: ! value, constant
   pair: abstract syntax; value

Values
~~~~~~

WebAssembly computations manipulate *values* of the four basic :ref:`value types <syntax-valtype>`: :ref:`uninterpreted integers <syntax-int>` and :ref:`floating-point data <syntax-float>` of 32 or 64 bit width each, respectively.

In most places of the semantics, values of different types can occur.
In order to avoid ambiguities, values are therefor represented with an abstract syntax that makes their type explicit.
It is convenient to reuse the same notation as for the :ref:`instructions <syntax-const>` producing them:

.. math::
   \begin{array}{llll}
   \production{(value)} & \val &::=&
     \I32.\CONST~\i32 ~|~
     \I64.\CONST~\i64 ~|~
     \F32.\CONST~\f32 ~|~
     \F64.\CONST~\f64
   \end{array}


.. _store:
.. _syntax-store:
.. index:: ! store, table instance, memory instance, global instance, module
   pair: abstract syntax; store

Store
~~~~~

The *store* represents all *mutable* global state that can be manipulated by WebAssembly programs.
It consists of the runtime representation of all *instances* of :ref:`tables <syntax-tableinst>`, :ref:`memories <syntax-meminst>`, and :ref:`globals <syntax-globalinst>` that have been *allocated* during the life time of the execution engine. [#gc]_

Syntactically, the store is defined as a :ref:`record <syntax-record>` listing the existing instances of each category:

.. math::
   \begin{array}{llll}
   \production{(store)} & \store &::=& \{~
     \begin{array}[t]{l@{~}ll}
     \TABLES & \tableinst^\ast, \\
     \MEMS & \meminst^\ast, \\
     \GLOBALS & \globalinst^\ast ~\} \\
     \end{array}
   \end{array}

.. note::
   Since the store holds the state of arbitrary many, possibly interacting, :ref:`module instances <syntax-moduleinst>`, it can host multiple tables and memories.

   :ref:`Function instances <syntax-funcinst>` and :ref:`module instances <syntax-moduleinst>` are not part of the store because they are not mutable.
   In particular, there is no requirement that function instances have a unique identity, since it is not observable.
   :ref:`Embedders <embedder>` that make the identity of function instances observable need to provide a suitable definition.

.. [#gc]
   In practice, implementations may apply techniques like garbage collection to remove objects from the store that are no longer referenced.
   However, such techniques are not semantically observable,
   and hence outside the scope of this specification.


Convention
..........

* The meta variable :math:`S` ranges over stores where clear from context.


.. _syntax-addr:
.. _syntax-tableaddr:
.. _syntax-memaddr:
.. _syntax-globaladdr:
.. index:: ! address, store, table instance, memory instance, global instance
   pair: abstract syntax; table address
   pair: abstract syntax; memory address
   pair: abstract syntax; global address
   pair: table; address
   pair: memory; address
   pair: global; address

Addresses
~~~~~~~~~

:ref:`Table instances <syntax-tableinst>`, :ref:`memory instances <syntax-meminst>`, and :ref:`global instances <syntax-globalinst>` in the :ref:`store <syntax-store>` are referenced with abstract *addresses*.
These are simply indices into the respective store component.

.. math::
   \begin{array}{llll}
   \production{(address)} & \addr &::=&
     0 ~|~ 1 ~|~ 2 ~|~ \dots \\
   \production{(table address)} & \tableaddr &::=&
     \addr \\
   \production{(memory address)} & \memaddr &::=&
     \addr \\
   \production{(global address)} & \globaladdr &::=&
     \addr \\
   \end{array}

.. note::
   There is no specific limit on the number of allocations of store objects,
   hence logical addresses can be arbitrarily large natural numbers.

   A *memory address* |memaddr| denotes the abstract address *of* a memory *instance* in the store,
   not an offset *inside* a memory instance.


.. _syntax-moduleinst:
.. index:: ! instance, function type, function instance, table instance, memory instance, global instance, export instance, table address, memory address, global address, index
   pair: abstract syntax; module instance
   pair: module; instance

Module Instances
~~~~~~~~~~~~~~~~

A *module instance* is the runtime representation of a :ref:`module <syntax-module>`.
It is created by :ref:`instantiating <instantiation>` a module,
and collects runtime representations of all entities that are imported, defined, or exported by the module.

.. math::
   \begin{array}{llll}
   \production{(module instance)} & \moduleinst &::=& \{
     \begin{array}[t]{l@{~}ll}
     \TYPES & \functype^\ast, \\
     \FUNCS & \funcinst^\ast, \\
     \TABLES & \tableaddr^\ast, \\
     \MEMS & \memaddr^\ast, \\
     \GLOBALS & \globaladdr^\ast \\
     \EXPORTS & \exportinst^\ast ~\} \\
     \end{array}
   \end{array}

Each component contains runtime instances corresponding to respective entities from the original module -- whether imported or defined -- in the order of their static :ref:`indices <syntax-index>`.
:ref:`Table instances <syntax-tableinst>`, :ref:`memory instances <syntax-meminst>`, and :ref:`global instances <syntax-globalinst>` are referenced with an indirection through their respective :ref:`addresses <syntax-addr>` in the :ref:`store <syntax-store>`.

It is an invariant of the semantics that all :ref:`export instances <syntax-exportinst>` in a given module instance have different :ref:`names <syntax-name>`.


.. _syntax-funcinst:
.. index:: ! function instance, module instance, function, closure
   pair: abstract syntax; function instance
   pair: function; instance

Function Instances
~~~~~~~~~~~~~~~~~~

A *function instance* is the runtime representation of a :ref:`function <syntax-func>`.
It is effectively a *closure* of the original function over the runtime :ref:`module instance <syntax-moduleinst>` of its own :ref:`module <syntax-module>`.
The module instance is used to resolve references to other non-local definitions during execution of the function.

.. math::
   \begin{array}{llll}
   \production{(function instance)} & \funcinst &::=&
     \{ \MODULE~\moduleinst, \FUNC~\func \} \\
   \end{array}


.. _syntax-tableinst:
.. _syntax-funcelem:
.. index:: ! table instance, table, function instance
   pair: abstract syntax; table instance
   pair: table; instance

Table Instances
~~~~~~~~~~~~~~~

A *table instance* is the runtime representation of a :ref:`table <syntax-table>`.
It holds a vector of *function elements* and an optional maximum size, if one was specified at the definition site of the table.

Each function element is either empty, representing an uninitialized table entry, or a :ref:`function instance <syntax-funcinst>`.
Function elements can be mutated through the execution of an :ref:`element segment <syntax-elem>` or by other means provided by the :ref:`embedder <embedder>`.

.. math::
   \begin{array}{llll}
   \production{(table instance)} & \tableinst &::=&
     \{ \ELEM~\vec(\funcelem), \MAX~\u32^? \} \\
   \production{(function element)} & \funcelem &::=&
     \funcinst^? \\
   \end{array}

It is an invariant of the semantics that the length of the element vector never exceeds the maximum size, if present.

.. note::
   Other table elements may be added in future versions of WebAssembly.


.. _syntax-meminst:
.. index:: ! memory instance, memory, byte, ! page size, memory type
   pair: abstract syntax; memory instance
   pair: memory; instance

Memory Instances
~~~~~~~~~~~~~~~~

A *memory instance* is the runtime representation of a linear :ref:`memory <syntax-mem>`.
It holds a vector of bytes and an optional maximum size, if one was specified at the definition site of the memory.

The length of the vector always is a multiple of the *page size*, which is defined to be the constant :math:`65536` -- abbreviated :math:`64\,\F{Ki}`.
Like in a :ref:`memory type <syntax-memtype>`, the maximum size in a memory instance is given in units of this page size.

The bytes can be mutated through specific instructions, the execution of a :ref:`data segment <syntax-data>`, or by other means provided by the :ref:`embedder <embedder>`.

.. math::
   \begin{array}{llll}
   \production{(memory instance)} & \meminst &::=&
     \{ \DATA~\vec(\byte), \MAX~\u32^? \} \\
   \end{array}

It is an invariant of the semantics that the length of the byte vector, divided by page size, never exceeds the maximum size, if present.


.. _syntax-globalinst:
.. index:: ! global instance, value
   pair: abstract syntax; global instance
   pair: global; instance

Global Instances
~~~~~~~~~~~~~~~~

A *global instance* is the runtime representation of a :ref:`global variable <syntax-global>`.
It holds an individual :ref:`value <syntax-val>` and a flag indicating whether it is mutable.

The value of mutable globals can be mutated through specific instructions or by other means provided by the :ref:`embedder <embedder>`.

.. math::
   \begin{array}{llll}
   \production{(global instance)} & \globalinst &::=&
     \{ \VALUE~\val, \MUT~\mut \} \\
   \end{array}


.. _syntax-exportinst:
.. index:: ! export instance, name, external value
   pair: abstract syntax; export instance
   pair: export; instance

Export Instances
~~~~~~~~~~~~~~~~

An *export instance* is the runtime representation of an :ref:`export <syntax-export>`.
It defines the export's :ref:`name <syntax-name>` and the :ref:`external value <syntax-externval>` being exported.

.. math::
   \begin{array}{llll}
   \production{(export instance)} & \exportinst &::=&
     \{ \NAME~\name, \VALUE~\externval \} \\
   \end{array}


.. _syntax-externval:
.. index:: ! external value, function instance, table address, memory address, global address
   pair: abstract syntax; external value
   pair: external; value

External Values
~~~~~~~~~~~~~~~

An *external value* is the runtime representation of an entity that can be imported or exported.
It is either a :ref:`function instance <syntax-funcinst>`, or an :ref:`address <syntax-addr>` denoting a :ref:`table instance <syntax-tableinst>`, :ref:`memory instance <syntax-meminst>`, and :ref:`global instances <syntax-globalinst>` in the shared :ref:`store <syntax-store>`.

.. math::
   \begin{array}{llll}
   \production{(external value)} & \externval &::=&
     \FUNC~\funcinst ~|~
     \TABLE~\tableaddr ~|~
     \MEM~\memaddr ~|~
     \GLOBAL~\globaladdr \\
   \end{array}


Conventions
...........

The following auxiliary notation is defined for sequences of external values.
It filters out entries of a specific kind in an order-preserving fashion:

.. math::
   \begin{array}{lcl}
   \funcs(\externval^\ast) &=& [\funcinst ~|~ (\FUNC~\funcinst) \in \externval^\ast] \\
   \tables(\externval^\ast) &=& [\tableaddr ~|~ (\TABLE~\tableaddr) \in \externval^\ast] \\
   \mems(\externval^\ast) &=& [\memaddr ~|~ (\MEM~\memaddr) \in \externval^\ast] \\
   \globals(\externval^\ast) &=& [\globaladdr ~|~ (\GLOBAL~\globaladdr) \in \externval^\ast] \\
   \end{array}


.. _syntax-externtype:
.. index:: ! external type, function type, table type, memory type, global type
   pair: abstract syntax; external type
   pair: external; type

External Types
~~~~~~~~~~~~~~

*External types* classify :ref:`external values <syntax-externval>`, and thereby imports and exports, with their respective types.

.. math::
   \begin{array}{llll}
   \production{external types} & \externtype &::=&
     \FUNC~\functype ~|~
     \TABLE~\tabletype ~|~
     \MEM~\memtype ~|~
     \GLOBAL~\globaltype \\
   \end{array}

These types are used in the definition of :ref:`instantiation <instantiation>`.


Conventions
...........

The following auxiliary notation is defined for sequences of external types.
It filters out entries of a specific kind in an order-preserving fashion:

.. math::
   \begin{array}{lcl}
   \funcs(\externtype^\ast) &=& [\functype ~|~ (\FUNC~\functype) \in \externtype^\ast] \\
   \tables(\externtype^\ast) &=& [\tabletype ~|~ (\TABLE~\tabletype) \in \externtype^\ast] \\
   \mems(\externtype^\ast) &=& [\memtype ~|~ (\MEM~\memtype) \in \externtype^\ast] \\
   \globals(\externtype^\ast) &=& [\globaltype ~|~ (\GLOBAL~\globaltype) \in \externtype^\ast] \\
   \end{array}


.. _stack:
.. _frame:
.. _label:
.. _syntax-frame:
.. _syntax-label:
.. index:: ! stack, ! frame, ! label
   pair: abstract syntax; frame
   pair: abstract syntax; label

Stack
~~~~~

Besides the :ref:`store <store>`, most :ref:`instructions <syntax-instr>` interact with an implicit *stack*.
The stack contains three kinds of entries:

* *Values*: the *operands* (arguments and results) of instructions.

* *Labels*: an active (entered) :ref:`structured control instruction <syntax-instr-control>` that can be targeted by branches.

* *Locals*: the *call frame* of an active :ref:`function <syntax-func>` call.

These entries can occur on the stack in any order during the execution of a program.

.. note::
   It is possible to model the WebAssebmly semantics using two or three separate stacks for operands, control constructs, and calls.
   However, because the stacks are interdependent, additional book keeping about associated stack heights would be required.
   For the purpose of this specification, an interleaved representation is simpler.

Stack entries are described by abstract syntax as follows.

**Values** are represented by :ref:`themselves <syntax-val>`.

**Labels** carry an argument arity :math:`n` and the branch *target*, which is expressed syntactically as an :ref:`instruction <syntax-instr>` sequence:

.. math::
   \begin{array}{llll}
   \production{(label)} & \label &::=&
     \LABEL_n\{\instr^\ast\} \\
   \end{array}

Intuitively, :math:`\instr^\ast` is the *continuation* to execute when the branch is taken, "replacing" the original control construct.

.. note::
   For example, a loop label has the form

   .. math::
      \LABEL_n\{\LOOP~[t^?]~\instr~\dots~\END\}

   When performing a branch to this label, this restarts the loop from the  beginning.
   Conversely, a simple block label has the form

   .. math::
      \LABEL_n\{\epsilon\}

   When branching, the empty continuation ends the targeted block and proceeding with consecutive instructions.

**Frames** carry the return arity of the respective function,
and hold the values of its locals (including arguments) in the order corresponding to their static :ref:`local indices <syntax-localidx>`,
as well as a reference to the function's own :ref:`module instance <syntax-moduleinst>`:

.. math::
   \begin{array}{llll}
   \production{(activation)} & \X{activation} &::=&
     \FRAME_n\{\frame\} \\
   \production{(frame)} & \frame &::=&
     \{ \LOCALS~\val^\ast, \MODULE~\moduleinst\} \\
   \end{array}

The values of the locals are mutated by respective instructions.

.. note::
   In the current version of WebAssembly, the arities of labels and activations cannot be larger than :math:`1`.
   This may be generalized in future versions.


Conventions
...........

* The meta variable :math:`L` ranges over labels where clear from context.

* The meta variable :math:`F` ranges over frames where clear from context.


.. _syntax-instr-admin:
.. _syntax-ctxt-eval:
.. _syntax-ctxt-label:
.. index:: ! administrative instructions, ! label context, ! evaluation context
   pair:: abstract syntax; administrative instructions

Administrative Instructions
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::
   This section is only relevant for the :ref:`formal notation <exec-notation>`.

In order to express the reduction of :ref:`traps <trap>` and :ref:`control instructions <syntax-instr-control>`, the syntax of instructions is extended to include the following *administrative instructions*:

.. math::
   \begin{array}{llll}
   \production{(administrative instruction)} & \instr &::=&
     \dots ~|~ \\&&&
     \INVOKE~\funcinst \\&&&
     \TRAP \\&&&
     \LABEL_n\{\instr^\ast\}~\instr^\ast~\END ~|~ \\&&&
     \FRAME_n\{\frame\}~\instr^\ast~\END ~|~ \\
   \end{array}

The |TRAP| instruction represents the occurrence of a trap.
Traps are bubbled up through nested instruction sequences, ultimately reducing the entire program to a single |TRAP| instruction.

The |INVOKE| instruction represents the imminent invocation of a :ref:`function instance <syntax-funcinst>`, and unifies the handling of different forms of calls.

The |LABEL| and |FRAME| instructions model :ref:`labels <syntax-label>` and :ref:`frames <syntax-frame>` :ref:`"on the stack" <exec-notation>`.
However, the administrative syntax also maintains the structure of the nested :ref:`instruction sequence <syntax-instr-seq>`, including the |END| pseudo instruction marking the end of the original :ref:`structured control instruction <syntax-instr-control>` or :ref:`function body <syntax-func>`;
that way, the end of the inner instruction sequence is tracked even when the instruction is part of an outer sequence.

.. note::
   For example, the :ref:`reduction rule <exec-block>` for |BLOCK| is:

   .. math::
      \BLOCK~[t^n]~\instr^\ast~\END \quad\stepto\quad
      \LABEL_n\{\epsilon\}~\instr^\ast~\END

   This replaces the block with a label instruction,
   which can be interpreted as "pushing" the label on the stack.
   When |END| is reached, i.e., the inner instruction sequence has been reduced to the empty sequence -- or a sequence of |CONST| instructions, the representation of non-empty local operand stack -- then the |LABEL| instruction is eliminated courtesy of its own :ref:`reduction rule <exec-label>`:

   .. math::
      \LABEL_n\{\instr^\ast\}~\val^\ast~\END \quad\stepto\quad \val^\ast

   This can be interpreted as removing the label from the stack and only leaving the locally accumulated operands.

To express :ref:`branches <syntax-instr-control>`, the following syntax of *block contexts* is defined, indexed by the :math:`k` count of labels surrounding the hole:

.. math::
   \begin{array}{llll}
   \production{(label contexts)} & B^0 &::=&
     \val^\ast~[\_]~\instr^\ast \\
   \production{(label contexts)} & B^{k+1} &::=&
     \val^\ast~\LABEL_n\{\instr^\ast\}~L^k~\END~\instr^\ast \\
   \end{array}

.. note::
   Given this definition, the :ref:`reduction <exec-br>` of a simple branch can be expressed as follows:

   .. math::
      \LABEL_0\{\instr^\ast\}~\B^l[\BR~l]~\END \quad\stepto\quad \instr^\ast

   When a branch occurs in a block context,
   this rule replaces the targeted label and associated instruction sequence with the label's continuation.

Finally, the following definition of *evaluation context* and associated structural rule enables the reduction inside instruction sequences and administrative forms and expresses the propagation of traps:

.. math::
   \begin{array}{llll}
   \production{(evaluation contexts)} & E &::=&
     [\_] ~|~
     \val^\ast~E~\instr^\ast ~|~
     \LABEL_n\{\instr^\ast\}~E~\END ~|~
     \FRAME_n\{\frame\}~E~\END \\
   \end{array}

.. math::
   \frac{
     S; F; \instr^\ast \stepto S'; F'; {\instr'}^\ast
   }{
     S; F; E[\instr^\ast] \stepto S'; F'; E[{\instr'}^\ast]
   }
   \qquad
   \frac{
     E \neq [\_]
   }{
     S; F; E[\TRAP] \stepto S; F; \TRAP
   }
