# [Base.Cartesian](@id cartesian)

El módulo `Cartesian` (no exportado) proporciona macros que facilitan escribir algoritmos multidimensionales. Es deseable que, a largo plazo, este módulo `Cartesian` no sea necesario; sin embargo, en la actualidad es una de las pocas formas de escribir código multidimensional compacto y con rendimiento.

## Principios de uso

Un emeplo de uso simple podría ser:

```julia
@nloops 3 i A begin
    s += @nref 3 A i
end
```

que genera el siguiente código:

```julia
for i_3 = 1:size(A,3)
    for i_2 = 1:size(A,2)
        for i_1 = 1:size(A,1)
            s += A[i_1,i_2,i_3]
        end
    end
end
```

En general, `Cartesian` permitirá escribir código que contiene elementos repetitivos, como los bucles anidados de este ejemplo. Otras aplicaciones incluyen expresiones repetidas (por ejemplo, desenrollado de bucles) o crear llamadas a función con números variables de argumentos sin usar la construcción "*splat*" (`i...`).

## Sintaxis Básica

La sintaxis básica de `@nloops` es la siguiente:

  * El primer argumento debe ser un entero (*no* una variable) que especifica el número de bucles.
  * El segundo argumento es el prefijo simbólico que se utilizará para la variable iteradora. De este modo, en el ejemplo anterior usamos `i`, y se generaron las variables  `i_1, i_2, i_3`.
  * El tercer argumento especifica el rango para cada variable iteradora. Si se usa una variable (símbolo) aquí, es considerado como `1:size(A,dim)`. De forma más flexible, se puede usar la sintaxis de expresiones basadas en funciones anónimas que se decribe más adelante.
  * El último argumento es el cuerpo del bucle. En el ejemplo anterior, lo que aparece entre `begin...end`.

There are some additional features of `@nloops` described in the [reference section](@ref dev-cartesian-reference).

`@nref` follows a similar pattern, generating `A[i_1,i_2,i_3]` from `@nref 3 A i`. The general
practice is to read from left to right, which is why `@nloops` is `@nloops 3 i A expr` (as in
`for i_2 = 1:size(A,2)`, where `i_2` is to the left and the range is to the right) whereas `@nref`
is `@nref 3 A i` (as in `A[i_1,i_2,i_3]`, where the array comes first).

If you're developing code with Cartesian, you may find that debugging is easier when you examine
the generated code, using `macroexpand`:

```@meta
DocTestSetup = quote
    import Base.Cartesian: @nref
end
```

```jldoctest
julia> macroexpand(:(@nref 2 A i))
:(A[i_1, i_2])
```

```@meta
DocTestSetup = nothing
```

### Supplying the number of expressions

The first argument to both of these macros is the number of expressions, which must be an integer.
When you're writing a function that you intend to work in multiple dimensions, this may not be
something you want to hard-code. If you're writing code that you need to work with older Julia
versions, currently you should use the `@ngenerate` macro described in [an older version of this documentation](https://docs.julialang.org/en/release-0.3/devdocs/cartesian/#supplying-the-number-of-expressions).

Starting in Julia 0.4-pre, the recommended approach is to use a `@generated function`.  Here's
an example:

```julia
@generated function mysum(A::Array{T,N}) where {T,N}
    quote
        s = zero(T)
        @nloops $N i A begin
            s += @nref $N A i
        end
        s
    end
end
```

Naturally, you can also prepare expressions or perform calculations before the `quote` block.

### Anonymous-function expressions as macro arguments

Perhaps the single most powerful feature in `Cartesian` is the ability to supply anonymous-function
expressions that get evaluated at parsing time.  Let's consider a simple example:

```julia
@nexprs 2 j->(i_j = 1)
```

`@nexprs` generates `n` expressions that follow a pattern. This code would generate the following
statements:

```julia
i_1 = 1
i_2 = 1
```

In each generated statement, an "isolated" `j` (the variable of the anonymous function) gets replaced
by values in the range `1:2`. Generally speaking, Cartesian employs a LaTeX-like syntax.  This
allows you to do math on the index `j`.  Here's an example computing the strides of an array:

```julia
s_1 = 1
@nexprs 3 j->(s_{j+1} = s_j * size(A, j))
```

would generate expressions

```julia
s_1 = 1
s_2 = s_1 * size(A, 1)
s_3 = s_2 * size(A, 2)
s_4 = s_3 * size(A, 3)
```

Las expresiones función anónima tienen muchos usos en la práctica.

#### [Referencia de las Macros](@id dev-cartesian-reference)

```@docs
Base.Cartesian.@nloops
Base.Cartesian.@nref
Base.Cartesian.@nextract
Base.Cartesian.@nexprs
Base.Cartesian.@ncall
Base.Cartesian.@ntuple
Base.Cartesian.@nall
Base.Cartesian.@nany
Base.Cartesian.@nif
```
