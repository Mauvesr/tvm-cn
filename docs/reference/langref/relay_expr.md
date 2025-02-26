---
title: Expressions in Relay
---

The Relay IR is a pure, expression-oriented language. The below sections
describe the different expressions in Relay and give details of their
semantics.

## Dataflow and Control Fragments

For the purposes of comparing Relay to traditional computational
graph-based IRs, it can be useful to consider Relay expressions in terms
of dataflow and control fragments. Each portion of a Relay program
containing expressions that only affect the dataflow can be viewed as a
traditional computation graph when writing and expressing
transformations.

The dataflow fragment covers the set of Relay expressions that do not
involve control flow. That is, any portion of a program containing only
the following constructs corresponds to a pure computation graph:

-   [Variables](#variables)
-   Tuple [Construction](#construction) and [Projection](#projection)
-   [Let Bindings](#let-bindings)
-   [Graph Bindings](#graph-bindings)
-   Calls to [Operators](#operators) and [ADT
    Constructors](#adt-constructors)

Control flow expressions allow the graph topology to change based on the
value of previously executed expressions. The control fragment in Relay
includes the following constructs:

-   [If-Then-Else](#if-then-else) Expressions
-   [ADT Matching](#adt-matching) Expressions
-   Recursive Calls in Functions

From the point of view of a computation graph, a function is a subgraph
and a function call inlines the subgraph, substituting its arguments for
the free variables in the subgraph with corresponding names. Thus, if a
function\'s body uses only dataflow constructs, a call to that function
is in the dataflow fragment; conversely, if the function\'s body
contains control flow, a call to that function is not part of the
dataflow fragment.

## Variables

Inspired by LLVM, Relay explicitly distinguishes between local and
global variables both in the AST and in the text format. In the text
format, global and local variables are distinguished by prefixes, or
*sigils*. Global variables are prefixed with `@` and local variables
with `%`.

This explicit distinction makes certain optimizations easier to
implement. For example, inlining a global definition requires no
analysis: simply substituting the definition suffices.

### Global Variable

Global identifiers are prefixed by the `@` sigil, such as \"`@global`\".
A global identifier always references a globally visible definition
contained in the globally visible environment, known as the
[module](#module-and-global-functions). Global identifiers must be
unique.

See :py`~tvm.relay.expr.GlobalVar`{.interpreted-text role="class"} for
its implementation and documentation.

### Local Variable

Local identifiers are prefixed by the `%` sigil, such as \"`%local`\". A
local identifier always references a function argument or a variable
bound in a `let` expression, and will be scoped to the function where it
appears or the `let` expression where it is bound, respectively.

In the below code segment, notice that `%a` is defined twice. This is
permitted, as in most functional languages; in the scope of the second
`let` expression, the name `%a` is \"shadowed,\" meaning all references
to `%a` in the inner scope refer to the later definition, while
references to `%a` in the outer scope continue to refer to the first
one.

``` 
let %a = 1;
let %b = 2 * %a;  // %b = 2
let %a = %a + %a; // %a = 2. %a is shadowed
%a + %b           // has value 2 + 2 = 4
```

(Note that in Relay\'s implementation, each definition of a local
variable creates a new :py`~tvm.relay.expr.Var`{.interpreted-text
role="class"}, so a shadowed local variable, despite having the same
name as one in an outer scope, will be a different object. This allows
for comparing local variables by pointer identity with the knowledge
that the same local variable object corresponds to a different binding
site.)

See :py`~tvm.relay.expr.Var`{.interpreted-text role="class"} for its
implementation and documentation.

## Functions

Functions in Relay act similarly to procedures or functions in other
programming languages and serve to generalize the concept of a named
subgraph.

Functions are first class in Relay, which means they are expressions
just like variables, constants, and tuples. Additionally, functions in
Relay are higher-order, which means that a function can be passed as an
argument to a function or returned by a function, as function
expressions evaluate to closures (see the [Closures](#closures)
subsection), which are values like tensors and tuples.

See :py`~tvm.relay.function.Function`{.interpreted-text role="class"}
for the definition and documentation of function nodes.

### Syntax

A definition minimally consists of the keyword `fn`, an empty set of
parameters, and a body expression
(:py`~tvm.relay.expr.Expr`{.interpreted-text role="class"}) contained by
curly braces.

``` 
fn() { body }
```

A definition may contain any number of parameters. For example, a simple
function that invokes the `add` operator:

``` 
fn(%x, %y) { add(%x, %y) }
```

Notice that within the function\'s body, the parameters are local
variables, just like those bound in a `let` expression.

One may also annotate explicit types on functions. For example, we can
restrict the above function to only work on certain types:

``` 
fn(%x : Tensor[(10, 10), float32], %y : Tensor[(10, 10), float32])
           -> Tensor[(10, 10), float32] {
    add(%x, %y)
}
```

The above function only takes arguments of type
`Tensor[(10, 10), float32]` and returns a value of type
`Tensor[(10, 10), float32]`. A function parameter is just a local
variable (:py`~tvm.relay.expr.LocalVar`{.interpreted-text role="class"})
optionally annotated with a type, written as `%x : T`.

When the type information is omitted, Relay attempts to infer the most
general type for the users. This property is known as generalization:
for a definition without explicit annotations, Relay attempts to assign
the most general type to the parameters and return type based on the
function body and call sites.

A recursive function expression can be defined using a `let` binding, as
here:

``` 
let %fact = fn(%x : Tensor[(10, 10), float32]) -> Tensor[(10, 10), float32] {
    if (%x == Constant(0, (10, 10), float32)) {
        Constant(1, (10, 10), float32)
    } else {
        %x * %fact(%x - Constant(1, (10, 10), float32))
    }
};
%fact(Constant(10, (10, 10), float32))
```

### Closures

A function expression evaluates to a closure. Closures are values that
are represented as a pair of a local environment (storing the values for
all variables defined outside the scope of the function\'s body) and the
function itself.

For example, in the below example, the final result will be a tensor of
zero values because the closure for `%f` stores the value of `%x` at the
pointer where `%f` was defined.

``` 
let %g = fn() {
  let %x = Constant(0, (10, 10), float32);
  // %x is a free variable in the below function
  fn(%y) { %y * %x }
};
// the %x in %g's body is not in scope anymore
// %f is a closure where %x maps to Constant(0, (10, 10), float32)
let %f = %g();
let %x = Constant(1, (10, 10), float32);
%f(%x) // evaluates to Constant(0, (10, 10), float32)
```

### Polymorphism and Type Relations

*Note: type parameter syntax is not yet supported in the text format.*

A function may also be given a set of type parameters, which can be
substituted for specific types at call sites. Functions with type
parameters are *type polymorphic*; their return type or the types of
arguments they will accept can vary based on the type arguments given at
call sites.

Type parameters are classified by *kind* and can only appear in parts of
the type signature where their kind is appropriate (e.g., type
parameters of kind `Shape` can only appear where a shape would be
expected in a tensor type); for a full discussion, see
`the documentation on type parameters <type-parameter>`{.interpreted-text
role="ref"}.

For example, one can define a polymorphic identity function for any
Relay type as follows:

``` 
fn<t : Type>(%x : t) -> t {
    %x
}
```

The below definition is also polymorphic, but restricts its arguments to
tensor types:

``` 
fn<s : Shape, bt : BaseType>(%x : Tensor[s, bt]) {
    %x
}
```

Notice that the return type is omitted and will be inferred.

*Note: \"where\" syntax is not yet supported in the text format.*

A function may also be subject to one or more type relations, such as in
the following:

``` 
fn(%x, %y) where Broadcast { add(%x, %y) }
```

In the above definition, the types of `%x` and `%y` and the return type
are subject to the `Broadcast` relation, meaning all three must be
tensors and their shapes follow the elementwise broadcast relation. As
with operators, the definitions of relations are not transparent to
Relay and they are instead implemented externally in either C++ or
Python.

As in the case of `Broadcast`, relations are used to express complicated
constraints on types (especially tensor shapes). All function relations
must hold at all call sites; type checking is thus treated as a
constraint-solving problem. For more detail on type relations and their
implementations, please see
`their section in the documentation on Relay's type system <type-relation>`{.interpreted-text
role="ref"}.

## Operators

An operator is a primitive operation, such as `add` or `conv2d`, not
defined in the Relay language. Operators are declared in the global
operator registry in C++. Many common operators are backed by TVM\'s
Tensor Operator Inventory.

To register an operator a user must provide an implementation of the
operator, its type, and any other desired metadata. The operator
registry is a column-based store where operators are keys, so any
metadata (which might be referenced by optimization passes) may be
registered as a new column.

From the perspective of Relay\'s type system, an operator is a function,
so operators may be called like any other function and have function
types. In particular, operator types are registered using a single type
relation (see
`the documentation on type relations <type-relation>`{.interpreted-text
role="ref"}), typically a relation specialized to that operator. For
example, the `add` operator is registered with the `Broadcast` relation,
indicating that the arguments of `add` must be tensors and that the
return type is a tensor whose shape depends on those of its arguments.

Operators are rendered without a sigil (e.g `conv2d`, `flatten`) when
pretty-printing Relay programs. Operators are explicitly contained in
the program and are uniquely identifiable by pointer.

Note that common arithmetic operators such as `add` and `multiply` may
be written using the corresponding arithmetic operators in the text
format (e.g., `+` or `*`) as syntactic sugar.

See :py`~tvm.relay.op.Op`{.interpreted-text role="class"} for the
definition and documentation of operator nodes, demonstrating the
infrastructure for registering operator metadata. The other files in
:py`~tvm.relay.op`{.interpreted-text role="class"} give handles for
generating a call to various pre-registered operators. The
`tutorial on adding operators to Relay <relay-add-op>`{.interpreted-text
role="ref"} shows how to add further operators into the language.

## ADT Constructors

Algebraic data types (ADTs) in Relay are described in detail in a
`separate overview<adt-overview>`{.interpreted-text role="ref"} and
their integration into the type system is described
`here<adt-typing>`{.interpreted-text role="ref"}.

In this section, we will simply note that ADT constructors are given a
function type and should be used inside call nodes like a function or
operator. An ADT constructor is defined by giving the name of the ADT it
constructs (a global type variable) and the types of the expected
arguments for the constructor.

If the ADT definition includes type variables, those type variables may
appear in the constructor. Constructors cannot include any other type
variables.

Let us suppose that `D` is an ADT that takes type parameters `a` and
`b`. If `C1` is a constructor for `D` and expects two arguments, one of
type `a` and one of type `b`, then `C1` has the following type
signature: `fun<a, b>(a, b) -> D[a, b]`. (See either the ADT overview or
the discussion of ADT typing for an explanation of the type call in the
return type.) If another constructor for `D`, `C2`, takes no arguments,
then it has the following type signature: `fun<a, b>() -> D[a, b]`; the
type parameters will always appear in the return type.

Once called, a constructor produces an ADT instance, which is a
container that stores the values of the arguments to the constructor as
well as the name (\"tag\") of the constructor. The tag will be used for
deconstructing the instances and retrieving the values when [ADT
Matching](#adt-matching).

See :py`~tvm.relay.adt.Constructor`{.interpreted-text role="class"} for
the definition and documentation.

## Call

Expressions with function types in Relay are \"callable,\" meaning that
they can be invoked via a function call. These consist of any expression
that evaluates to a closure (i.e., function expressions or global
functions) and Relay operators.

The syntax of calls follows that used in C-like languages, demonstrated
in the example below:

``` 
let %c = 1;
let %f = fn(%x : Tensor[(), float32], %y : Tensor[(), float32]) { %x + %y + %c };
%f(10, 11)
```

When a closure is called (see [Closures](#closures)), the closure\'s
body is evaluated in the stored environment (i.e., using the stored
values for free variables) with local variable bindings added for each
argument; the final value obtained by evaluating the body is the call\'s
return value. Thus, in the above example, the call evaluates to 22. In
the case of operators, the implementation is opaque to Relay, so the
result is left up to the registered TVM implementation.

*Note: type parameters are not yet supported in the text format.*

A type-polymorphic function can also include type arguments at a call
site. The type arguments are substituted for type parameters when type
checking. If a function is type-polymorphic and type arguments are not
given, type inference will attempt to infer type arguments if possible.
The following code gives examples of explicit and inferred type
arguments:

``` 
// %f : fn<a : Type, b : Type, c : Type>(a, b) -> c
let %x1 = %f<Tensor[(), bool], Tensor[(), bool], Tensor[(), bool)]>(True, False);
// %x1 is of type Tensor[(), bool]
let %x2 : () = %f(%x1, %x1)
// the type arguments in the second call are inferred to be <Tensor[(), bool], Tensor[(), bool], ()>
```

Note that all type relations in the function type must hold at each call
site. Specifically, this means that the relation will be checked against
the specific types of the arguments at a given call site. This is also a
form of polymorphism, since there may be multiple valid assignments of
argument types and a return type so long as the relation is satisfied.

For example, if we have a function `%f` that takes tensor arguments and
has the `Broadcast` relation, then there are many different shapes that
the arguments in the below call could have that would satisfy the type
annotation:

``` 
let %x : Tensor[(100, 100, 100), float32] = %f(%a, %b);
%x
```

See :py`~tvm.relay.expr.Call`{.interpreted-text role="class"} for its
definition and documentation.

## Module and Global Functions {#module-description}

Relay keeps a global data structure known as a \"module\" (often called
an \"environment\" in other functional programming languages) to keep
track of the definitions of global functions. In particular, the module
keeps a globally accessible mapping of global variables to the function
expressions they denote. The utility of the module is that it allows
global functions to recursively refer to themselves or any other global
function (e.g., as in mutual recursion).

Note Relay\'s module is analogous to data structures for keeping track
of subgraphs in computation graph-based IRs.

Global functions in Relay behave identically to the function expressions
defined in [Functions](#functions), but have syntactic sugar in the text
format to enter their definitions into the module. Namely, a global
function definition includes a global identifier and is allowed to
recursively refer to that identifier in the body, as in the following
example:

``` 
def @ackermann(%m : Tensor[(), int32], %n : Tensor[(), int32]) -> Tensor[(), int32] {
    if (%m == 0) {
        %n + 1
    } else if (%m > 0 && %n == 0) {
        @ackermann(%m - 1, 1)
    } else {
        @ackermann(%m - 1, @ackermann(%m, %n - 1))
    }
}
```

This definition would result in a module entry mapping the identifier
`@ackermann` to a function expression with the parameters, return type,
and body above. Any reference to the identifier `@ackermann` elsewhere
in the code could then look up the identifier in the module and replace
the function definition as needed.

See :py`~tvm.IRModule`{.interpreted-text role="class"} for the
definition and documentation of a module.

## Constant

This node represents a constant tensor value (see
:py`~tvm.relay.Value`{.interpreted-text role="mod"} for more details). A
constant is represented as a :py`~tvm.NDArray`{.interpreted-text
role="class"}, allowing Relay to utilize TVM operators for constant
evaluation.

This node can also represent scalar constants, since scalars are tensors
with a shape of `()`. In the text format, numerical and boolean literals
are thus syntactic sugar for constants encoding a tensor type with a
rank-zero shape.

See :py`~tvm.relay.expr.Constant`{.interpreted-text role="class"} for
its definition and documentation.

## Tuples

### Construction

The tuple node builds a finite (that is, of statically known size)
sequence of heterogeneous data. These tuples match Python\'s closely,
and their fixed length allows for efficient projection of their members.

``` 
fn(%a : Tensor[(10, 10), float32], %b : float32, %c : Tensor[(100, 100), float32]) {
    let %tup = (%a, %b);     // type: (Tensor[(10, 10), float32], float32)
    ((%tup.0 + %tup.1), %c)  // type: (Tensor[(10, 10), float32], Tensor[(100, 100), float32])
}
```

See :py`~tvm.relay.expr.Tuple`{.interpreted-text role="class"} for its
definition and documentation.

### Projection

A tuple must be indexed by an integer constant in order to extract a
particular member of the tuple. Projections are 0-indexed.

For example, the below projection evaluates to `%b`:

``` 
(%a, %b, %c).1
```

See :py`~tvm.relay.expr.TupleGetItem`{.interpreted-text role="class"}
for its definition and documentation.

## Let Bindings

A `let` binding is an immutable local variable binding, allowing the
user to bind an expression to a name.

A `let` binding contains a local variable, an optional type annotation,
a value, and a body expression that may reference the bound identifier.
If a type annotation on the bound variable is omitted, Relay attempts to
infer the most general type permitted for the variable.

The bound variable in a `let` expression is only in scope in its body,
except when the variable defines a function expression. When a `let`
expression creates a function, the variable is also in scope in its
value to allow for recursively defined functions (see the previous
subsection).

The value of a `let` binding is the value of the final expression after
evaluating the bindings it depends on. For example, in the following
example the entire expression evaluates to a tensor of shape `(10, 10)`
where all elements are 2:

``` 
let %x : Tensor[(10, 10), float32] = Constant(1, (10, 10), float32);
%x + %x
```

A sequence of `let` bindings can be considered as a dataflow graph,
where the bindings are a series of sub-graphs connected by bound
variables. Since these binding sequences are pure, a pair of bindings
where neither depends on the other can be safely reordered. For example,
the first and second `let` bindings below may be evaluated in either
order because neither has a dataflow dependency on the other:

``` 
let %x = %a + %b;
let %y = %c + %d;
%x * %y
```

See :py`~tvm.relay.expr.Let`{.interpreted-text role="class"} for its
definition and documentation.

## Graph Bindings

A `let` binding creates a named variable that is bound to the given
value and scoped to the subsequent expression. By contrast, a graph
binding allows for explicitly constructing dataflow graphs in a Relay
program by binding an expression (graph node) directly to a temporary
variable, which is not scoped. Each reference to the variable
corresponds to an edge in the dataflow graph. This has the semantics of
substituting the expression wherever the variable appears, even though
the graph node will only be evaluated once by the compiled program.

These bindings allow for a style of programming that corresponds to that
already employed by NNVM and other dataflow graph-based input formats.
The fact that the variables are not scoped offers some flexibility in
evaluation order compared to `let` bindings, though this can also
introduce some ambiguity in programs (the
`developer introduction to the Relay IR<relay-dev-intro>`{.interpreted-text
role="ref"} includes more detailed discussion of this nuance).

*Note: Graph bindings are not currently parsed by the text format.*

In Relay\'s text format, a graph binding can be written as below (note
the lack of a `let` keyword and a semicolon):

``` 
%1 = %a + %b
%2 = %1 + %1
%2 * %2
```

Unlike a let binding, a graph binding is not represented as an AST node
in Relay, but rather as a meta-variable referencing its AST node value.
For example, a program like the above could be constructed in Relay\'s
Python front-end by setting *Python variables* equal to the
corresponding Relay AST node and using the variables repeatedly, as
below (a C++ program using the corresponding API bindings could
accomplish the same thing):

``` 
sum1 = relay.add(a, b)
sum2 = relay.add(sum1, sum1)
relay.multiply(sum2, sum2)
```

For development purposes and to enable certain optimizations, Relay
includes passes to convert between dataflow graphs defined using graph
bindings and programs with `let` bindings in A-normal form, employed by
many compiler optimizations from the functional programming community
(see [\"A-Normalization: Why and How\" by Matt
Might](http://matt.might.net/articles/a-normalization/) for an
introduction to A-normal form).

## If-Then-Else

Relay has a simple if-then-else expression that allows programs to
branch on a single value of type `bool`, i.e., a zero-rank tensor of
booleans (`Tensor[(), bool]`).

``` 
if (%t == %u) {
    %t
} else {
    %u
}
```

Since if-then-else branches are expressions, they may appear inline
wherever any other expression may be expected, like invocations of the
ternary operator in C-like languages. The if-then-else expression
evaluates to the value of the \"then\" branch if the condition value
evaluates to `True` and evaluates to the value of the \"else\" branch if
the condition value evaluates to `False`.

See :py`~tvm.relay.expr.If`{.interpreted-text role="class"} for its
definition and documentation.

## ADT Matching

Instances of algebraic data types (ADTs), as discussed in the
`ADT overview<adt-overview>`{.interpreted-text role="ref"}, are
containers that store the arguments passed to the constructor used to
create them, tagged by the constructor name.

Match expressions in Relay allow for retrieving the values stored in an
ADT instance (\"deconstructing\" it) based on their constructor tag. A
match expression behaves similarly to a C-style `switch` statement,
branching on the different possible constructors for the type of the
value being deconstructed. As the ADT overview details, match
expressions are capable of more general pattern-matching than simply
splitting by constructors: any ADT instance nested inside an instance
(e.g., a list of lists) can be deconstructed at the same time as the
outer instance, while the different fields of the instance can be bound
to variables. (See `this section<adt-pattern>`{.interpreted-text
role="ref"} for a detailed description of ADT pattern-matching.)

A match expression is defined using the input value (an expression) and
a list of clauses, each of which consists of a pattern and an
expression. When executed, the *first* clause whose pattern matches the
structure of the queried value is executed; the clause expression is
evaluated and returned.

For example, suppose we have an ADT for natural numbers:

``` 
data Nat {
  Z : () -> Nat # zero
  S : (Nat) -> Nat # successor (+1) to a nat
}
```

Then the following function subtracts one from a passed nat:

``` 
fn(%v: Nat[]) -> Nat[] {
  match(%v) {
    case Z() { Z() }
    case S(%n) { %n } # the variable %n is bound in the scope of this clause
  }
}
```

The following function subtracts two from its argument if it is at least
two and returns the argument otherwise, using a nested constructor
pattern:

``` 
fn(%v : Nat[]) -> Nat[] {
  match(%v) {
     case S(S(%n)) { %n }
     # wildcard pattern: matches all cases not matched already
     case _ { %v }
  }
}
```

As aforementioned, the ordering of match clauses is relevant. In the
below example, the first clause will always match so those below it can
never run:

``` 
fn(%v : Nat[]) -> Nat[] {
  match(%v) {
    case _ { %v }
    case S(S(%n)) { S(%n) }
    case S(%n) { %n }
    case Z() { S(Z()) }
  }
}
```

See :py`~tvm.relay.adt.Match`{.interpreted-text role="class"} for its
definition and documentation.

## TempExprs

Program transformations (passes) in Relay may require inserting
temporary state into the program AST to guide further transformations.
The `TempExpr` node is provided as a utility to developers for this
purpose; nodes inheriting from `TempExpr` cannot appear directly in
user-provided code but may be inserted in a pass. Any `TempExpr` created
in a pass should ideally be eliminated before the pass is complete, as a
`TempExpr` only stores internal state and has no semantics of its own.

For an example of `TempExpr` being used in a pass, see
`src/relay/transforms/fold_scale_axis.cc`, which uses `TempExpr` nodes
to store information about scaling parameters as the pass tries to fold
these into the weights of a convolution.

See :py`~tvm.relay.expr.TempExpr`{.interpreted-text role="class"} for
its definition and documentation.
