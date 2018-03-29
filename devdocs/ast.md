# [Julia ASTs](@id ast)

Julia tiene dos representaciones del código. Primero hay una sintaxis de superficie AST devuelta por el analizador sintáctico (e.g. la función [`parse()`](@ref)), y manipulada por macros. Es una representación estructurada del código tal como está escrito, construída por `julia-parser.scm` a partir de un flujo de caracteres. Después hay una forma disminuida, o IR ($intermediate representation$), que es usada por la inferencia de tipos y la generación de código. En la forma disminuida hay menos tipos de nodos, todas las macros son expandidas, y todo el control de flujo es convertido a bifurcaciones específicas y secuencias de instrucciones. La forma disminuida es construída por `julia-syntax.scm`.

Primero nos centraremo en la forma disminuida, ya que es ms importante para el compilador. Es también menos obvia para el humano, ya que ella resulta de un reordenamiento significativo de la sintaxis de entrada.

## Forma disminuida

En la forma disminuida existen los siguientes tipos de datos:

  * `Expr`

    Tiene un nodo indicado por el campo `head`, y un campo `args` que es un `Vector{Any}` de 
    subexpresiones.

  * `Slot`

    Identifica argumentos y variables locales mediante numerado consecutivo. `Slot` es un tipo abstracto con subtipos
    `SlotNumber` y `TypedSlot`. Ambos tipos tienen un campo `id` de valor entero que da el índice del slot. La mayoría 
    de los slots tienen el mismo tipo en todos sus usos, y por tanto están representados con `SlotNumber`. Los tipos
    de estos slots se encuentran en el campo `slottypes` de su objeto `MethodInstance`. Los slots que necesitan 
    anotaciones de tipo por uso son representados con `TypedSlot`, que tiene un campo `typ` field.

  * `CodeInfo`

    Envuelve el IR de un método.

  * `LineNumberNode`

    Contiene un solo número, especificando el número de línea de la siguiente instrucción.

  * `LabelNode`

    Blanco de la bifurcación, un número entero numerado consecutivamente que comienza en 0.

  * `GotoNode`

    Bifurcacin incondicional.

  * `QuoteNode`

    Envuelve un valor arbitrario para referenciarlo como datos. Por ejemplo, la función `f() =: a` contiene un 
    `QuoteNode` cuyo campo `value` es el símbolo `a`, para devolver el símbolo en sí mismo en lugar de evaluarlo.

  * `GlobalRef`

    Se refiere a una variable global `name` en el módulo `mod`.

  * `SSAValue`

    Se refiere a una variable de asignación única estática ($static single assignment$ - SSA) numerada 
    consecutivamente (comenzando en 0) insertada por el compilador.

  * `NewvarNode`

    Marca un punto donde se crea una variable. Esto tiene el efecto de restablecer una variable a indefinida.

### Expr types

These symbols appear in the `head` field of `Expr`s in lowered form.

  * `call`

    Function call (dynamic dispatch). `args[1]` is the function to call, `args[2:end]` are the arguments.

  * `invoke`

    Function call (static dispatch). `args[1]` is the MethodInstance to call, `args[2:end]` are the
    arguments (including the function that is being called, at `args[2]`).

  * `static_parameter`

    Reference a static parameter by index.

  * `line`

    Line number and file name metadata. Unlike a `LineNumberNode`, can also contain a file name.

  * `gotoifnot`

    Conditional branch. If `args[1]` is false, goes to label identified in `args[2]`.

  * `=`

    Assignment.

  * `method`

    Adds a method to a generic function and assigns the result if necessary.

    Has a 1-argument form and a 4-argument form. The 1-argument form arises from the syntax `function foo end`.
    In the 1-argument form, the argument is a symbol. If this symbol already names a function in the
    current scope, nothing happens. If the symbol is undefined, a new function is created and assigned
    to the identifier specified by the symbol. If the symbol is defined but names a non-function,
    an error is raised. The definition of "names a function" is that the binding is constant, and
    refers to an object of singleton type. The rationale for this is that an instance of a singleton
    type uniquely identifies the type to add the method to. When the type has fields, it wouldn't
    be clear whether the method was being added to the instance or its type.

    The 4-argument form has the following arguments:

      * `args[1]`

        A function name, or `false` if unknown. If a symbol, then the expression first
        behaves like the 1-argument form above. This argument is ignored from then on. When
        this is `false`, it means a method is being added strictly by type, `(::T)(x) = x`.

      * `args[2]`

        A `SimpleVector` of argument type data. `args[2][1]` is a `SimpleVector` of the
        argument types, and `args[2][2]` is a `SimpleVector` of type variables corresponding
        to the method's static parameters.

      * `args[3]`

        A `CodeInfo` of the method itself. For "out of scope" method definitions (adding a
        method to a function that also has methods defined in different scopes) this is an
        expression that evaluates to a `:lambda` expression.

      * `args[4]`

        `true` or `false`, identifying whether the method is staged (`@generated function`).

  * `const`

    Declares a (global) variable as constant.
  * `null`

    Has no arguments; simply yields the value `nothing`.

  * `new`

    Allocates a new struct-like object. First argument is the type. The `new` pseudo-function is lowered
    to this, and the type is always inserted by the compiler.  This is very much an internal-only
    feature, and does no checking. Evaluating arbitrary `new` expressions can easily segfault.

  * `return`

    Returns its argument as the value of the enclosing function.

  * `the_exception`

    Yields the caught exception inside a `catch` block. This is the value of the run time system variable
    `jl_exception_in_transit`.

  * `enter`

    Enters an exception handler (`setjmp`). `args[1]` is the label of the catch block to jump to on
    error.

  * `leave`

    Pop exception handlers. `args[1]` is the number of handlers to pop.

  * `inbounds`

    Controls turning bounds checks on or off. A stack is maintained; if the first argument of this
    expression is true or false (`true` means bounds checks are disabled), it is pushed onto the stack.
    If the first argument is `:pop`, the stack is popped.

  * `boundscheck`

    Indicates the beginning or end of a section of code that performs a bounds check. Like `inbounds`,
    a stack is maintained, and the second argument can be one of: `true`, `false`, or `:pop`.

  * `copyast`

    Part of the implementation of quasi-quote. The argument is a surface syntax AST that is simply
    copied recursively and returned at run time.

  * `meta`

    Metadata. `args[1]` is typically a symbol specifying the kind of metadata, and the rest of the
    arguments are free-form. The following kinds of metadata are commonly used:

      * `:inline` and `:noinline`: Inlining hints.

      * `:push_loc`: enters a sequence of statements from a specified source location.

          * `args[2]` specifies a filename, as a symbol.
          * `args[3]` optionally specifies the name of an (inlined) function that originally contained the
            code.

      * `:pop_loc`: returns to the source location before the matching `:push_loc`.


### Method

A unique'd container describing the shared metadata for a single method.

  * `name`, `module`, `file`, `line`, `sig`

    Metadata to uniquely identify the method for the computer and the human.

  * `ambig`

    Cache of other methods that may be ambiguous with this one.

  * `specializations`

    Cache of all MethodInstance ever created for this Method, used to ensure uniqueness.
    Uniqueness is required for efficiency, especially for incremental precompile and
    tracking of method invalidation.

  * `source`

    The original source code (usually compressed).

  * `roots`

    Pointers to non-AST things that have been interpolated into the AST, required by
    compression of the AST, type-inference, or the generation of native code.

  * `nargs`, `isva`, `called`, `isstaged`, `pure`

    Descriptive bit-fields for the source code of this Method.

  * `min_world` / `max_world`

    The range of world ages for which this method is visible to dispatch.


### MethodInstance

A unique'd container describing a single callable signature for a Method. See especially [Proper maintenance and care of multi-threading locks](@ref locks)
for important details on how to modify these fields safely.

  * `specTypes`

    The primary key for this MethodInstance. Uniqueness is guaranteed through a
    `def.specializations` lookup.

  * `def`

    The `Method` that this function describes a specialization of. Or `#undef`, if this is
    a top-level Lambda that is not part of a Method.

  * `sparam_vals`

    The values of the static parameters in `specTypes` indexed by `def.sparam_syms`. For the
    `MethodInstance` at `Method.unspecialized`, this is the empty `SimpleVector`. But for a
    runtime `MethodInstance` from the `MethodTable` cache, this will always be defined and
    indexable.

  * `rettype`

    The inferred return type for the `specFunctionObject` field, which (in most cases) is
    also the computed return type for the function in general.

  * `inferred`

    May contain a cache of the inferred source for this function, or other information about
    the inference result such as a constant return value may be put here (if `jlcall_api ==
    2`), or it could be set to `nothing` to just indicate `rettype` is inferred.

  * `ftpr`

    The generic jlcall entry point.

  * `jlcall_api`

    The ABI to use when calling `fptr`. Some significant ones include:

      * 0 - Not compiled yet
      * 1 - JL_CALLABLE `jl_value_t *(*)(jl_function_t *f, jl_value_t *args[nargs], uint32_t nargs)`
      * 2 - Constant (value stored in `inferred`)
      * 3 - With Static-parameters forwarded `jl_value_t *(*)(jl_svec_t *sparams, jl_function_t *f, jl_value_t *args[nargs], uint32_t nargs)`
      * 4 - Run in interpreter `jl_value_t *(*)(jl_method_instance_t *meth, jl_function_t *f, jl_value_t *args[nargs], uint32_t nargs)`

  * `min_world` / `max_world`

    The range of world ages for which this method instance is valid to be called.


### CodeInfo

A temporary container for holding lowered source code.

  * `code`

    An `Any` array of statements

  * `slotnames`

    An array of symbols giving the name of each slot (argument or local variable).

  * `slottypes`

    An array of types for the slots.

  * `slotflags`

    A `UInt8` array of slot properties, represented as bit flags:

      * 2  - assigned (only false if there are *no* assignment statements with this var on the left)
      * 8  - const (currently unused for local variables)
      * 16 - statically assigned once
      * 32 - might be used before assigned. This flag is only valid after type inference.

  * `ssavaluetypes`

    Either an array or an `Int`.

    If an `Int`, it gives the number of compiler-inserted temporary locations in the
    function. If an array, specifies a type for each location.

Boolean properties:

  * `inferred`

    Whether this has been produced by type inference.

  * `inlineable`

    Whether this should be inlined.

  * `propagate_inbounds`

    Whether this should should propagate `@inbounds` when inlined for the purpose of eliding
    `@boundscheck` blocks.

  * `pure`

    Whether this is known to be a pure function of its arguments, without respect to the
    state of the method caches or other mutable global state.


## Surface syntax AST

Front end ASTs consist entirely of `Expr`s and atoms (e.g. symbols, numbers). There is generally
a different expression head for each visually distinct syntactic form. Examples will be given
in s-expression syntax. Each parenthesized list corresponds to an Expr, where the first element
is the head. For example `(call f x)` corresponds to `Expr(:call, :f, :x)` in Julia.

### Calls

| Input            | AST                                |
|:---------------- |:---------------------------------- |
| `f(x)`           | `(call f x)`                       |
| `f(x, y=1, z=2)` | `(call f x (kw y 1) (kw z 2))`     |
| `f(x; y=1)`      | `(call f (parameters (kw y 1)) x)` |
| `f(x...)`        | `(call f (... x))`                 |

`do` syntax:

```julia
f(x) do a,b
    body
end
```

parses as `(call f (-> (tuple a b) (block body)) x)`.

### Operators

Most uses of operators are just function calls, so they are parsed with the head `call`. However
some operators are special forms (not necessarily function calls), and in those cases the operator
itself is the expression head. In julia-parser.scm these are referred to as "syntactic operators".
Some operators (`+` and `*`) use N-ary parsing; chained calls are parsed as a single N-argument
call. Finally, chains of comparisons have their own special expression structure.

| Input       | AST                       |
|:----------- |:------------------------- |
| `x+y`       | `(call + x y)`            |
| `a+b+c+d`   | `(call + a b c d)`        |
| `2x`        | `(call * 2 x)`            |
| `a&&b`      | `(&& a b)`                |
| `x += 1`    | `(+= x 1)`                |
| `a ? 1 : 2` | `(if a 1 2)`              |
| `a:b`       | `(: a b)`                 |
| `a:b:c`     | `(: a b c)`               |
| `a,b`       | `(tuple a b)`             |
| `a==b`      | `(call == a b)`           |
| `1<i<=n`    | `(comparison 1 < i <= n)` |
| `a.b`       | `(. a (quote b))`         |
| `a.(b)`     | `(. a b)`                 |

### Bracketed forms

| Input                    | AST                                  |
|:------------------------ |:------------------------------------ |
| `a[i]`                   | `(ref a i)`                          |
| `t[i;j]`                 | `(typed_vcat t i j)`                 |
| `t[i j]`                 | `(typed_hcat t i j)`                 |
| `t[a b; c d]`            | `(typed_vcat t (row a b) (row c d))` |
| `a{b}`                   | `(curly a b)`                        |
| `a{b;c}`                 | `(curly a (parameters c) b)`         |
| `[x]`                    | `(vect x)`                           |
| `[x,y]`                  | `(vect x y)`                         |
| `[x;y]`                  | `(vcat x y)`                         |
| `[x y]`                  | `(hcat x y)`                         |
| `[x y; z t]`             | `(vcat (row x y) (row z t))`         |
| `[x for y in z, a in b]` | `(comprehension x (= y z) (= a b))`  |
| `T[x for y in z]`        | `(typed_comprehension T x (= y z))`  |
| `(a, b, c)`              | `(tuple a b c)`                      |
| `(a; b; c)`              | `(block a (block b c))`              |

### Macros

| Input         | AST                                   |
|:------------- |:------------------------------------- |
| `@m x y`      | `(macrocall @m x y)`                  |
| `Base.@m x y` | `(macrocall (. Base (quote @m)) x y)` |
| `@Base.m x y` | `(macrocall (. Base (quote @m)) x y)` |

### Strings

| Input           | AST                          |
|:--------------- |:---------------------------- |
| `"a"`           | `"a"`                        |
| `x"y"`          | `(macrocall @x_str "y")`     |
| `x"y"z`         | `(macrocall @x_str "y" "z")` |
| `"x = $x"`      | `(string "x = " x)`          |
| ``` `a b c` ``` | `(macrocall @cmd "a b c")`   |

Doc string syntax:

```julia
"some docs"
f(x) = x
```

parses as `(macrocall (|.| Core '@doc) "some docs" (= (call f x) (block x)))`.

### Imports and such

| Input               | AST                                          |
|:------------------- |:-------------------------------------------- |
| `import a`          | `(import a)`                                 |
| `import a.b.c`      | `(import a b c)`                             |
| `import ...a`       | `(import . . . a)`                           |
| `import a.b, c.d`   | `(toplevel (import a b) (import c d))`       |
| `import Base: x`    | `(import Base x)`                            |
| `import Base: x, y` | `(toplevel (import Base x) (import Base y))` |
| `export a, b`       | `(export a b)`                               |

### Numbers

Julia supports more number types than many scheme implementations, so not all numbers are represented
directly as scheme numbers in the AST.

| Input                   | AST                                              |
|:----------------------- |:------------------------------------------------ |
| `11111111111111111111`  | `(macrocall @int128_str "11111111111111111111")` |
| `0xfffffffffffffffff`   | `(macrocall @uint128_str "0xfffffffffffffffff")` |
| `1111...many digits...` | `(macrocall @big_str "1111....")`                |

### Block forms

A block of statements is parsed as `(block stmt1 stmt2 ...)`.

If statement:

```julia
if a
    b
elseif c
    d
else e
    f
end
```

parses as:

```
(if a (block (line 2) b)
    (block (line 3) (if c (block (line 4) d)
                        (block (line 5) e (line 6) f))))
```

A `while` loop parses as `(while condition body)`.

A `for` loop parses as `(for (= var iter) body)`. If there is more than one iteration specification,
they are parsed as a block: `(for (block (= v1 iter1) (= v2 iter2)) body)`.

`break` and `continue` are parsed as 0-argument expressions `(break)` and `(continue)`.

`let` is parsed as `(let body (= var1 val1) (= var2 val2) ...)`.

A basic function definition is parsed as `(function (call f x) body)`. A more complex example:

```julia
function f{T}(x::T; k = 1)
    return x+1
end
```

parses as:

```
(function (call (curly f T) (parameters (kw k 1))
                (:: x T))
          (block (line 2 file.jl) (return (call + x 1))))
```

Type definition:

```julia
mutable struct Foo{T<:S}
    x::T
end
```

parses as:

```
(type #t (curly Foo (<: T S))
      (block (line 2 none) (:: x T)))
```

The first argument is a boolean telling whether the type is mutable.

`try` blocks parse as `(try try_block var catch_block finally_block)`. If no variable is present
after `catch`, `var` is `#f`. If there is no `finally` clause, then the last argument is not present.

