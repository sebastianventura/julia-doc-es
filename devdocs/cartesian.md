# [Base.Cartesian](@id cartesian)

El módulo `Cartesian` (no exportado) proporciona macros que facilitan escribir algoritmos multidimensionales. Es deseable que, a largo plazo, este módulo `Cartesian` no sea necesario; sin embargo, en la actualidad es una de las pocas formas de escribir código multidimensional compacto y con rendimiento.

## Principios de uso

Un ejemplo de uso simple podría ser:

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

Hay otras características adicionales de `@nloops` descritas en la [sección de referencia](@ref dev-cartesian-reference).

`@nref` sigue un patrón similar, generando `A[i_1,i_2,i_3]` a partir de `@nref 3 A i`. La práctica general es leer de izquierda a derecha, por lo que `@nloops` es `@nloops 3 i A expr` (como en el bucle `for i_2 = 1:size(A,2)`, donde `i_2` está a la izquierda y el rango a la derecha) mientras que `@nref` es `@nref 3 A i` (como en `A[i_1,i_2,i_3]`, donde el array va primero).

Si se está desarrollando código con Cartesian, puede utilizarse `macroexpand`, que muestra el código generado, para depurar de un modo más sencillo:

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

### Proporcionando el número de expresiones

El primer argumento de estas dos macros es el número de expresiones, que debe ser un entero. Cuando se está escribiendo una función para que trabaje en múltiples dimensiones, esto puede no ser algo que uno desee codificar. Si se está escribiendo código que tiene que funcionar con versiones antiguas de Julia, debería usarse la macro `@ngenerate` descrita en [una versión más antigua de esta documentación](https://docs.julialang.org/en/release-0.3/devdocs/cartesian/#supplying-the-number-of-expressions).

Empezando en Julia 0.4-pre, el enfoque recomendado es usar una `@generated function`.  He aquí un ejemplo:

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

Naturalmente, también se pueden preparar expresiones o realizar cálculos antes del bloque `quote`.

### Expresiones función anónima como argumentos de macros

Quizás la característica más potente de `Cartesian` es la capacidad de proporcionar expresiones función-anónima que son evaluadas en tiempo de análisis sintáctico. Consideremos un ejemplo sencillo:

```julia
@nexprs 2 j->(i_j = 1)
```

`@nexprs` genera `n` expresiones que siguen un patrón. Este código generaría las siguientes instrucciones:

```julia
i_1 = 1
i_2 = 1
```

En cada instrucción generada un `j` aislado (la variable de la función anónima) es reemplazada por valores en el rango `1:2`. Hablando de forma general, Cartesian emplea una sintaxis parecida a LaTeX. Esto te permite hacer operaciones sobre el índice `j`.  He aquí un ejemplo que calcula los pasos de un array:

```julia
s_1 = 1
@nexprs 3 j->(s_{j+1} = s_j * size(A, j))
```

generará las expresiones

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
