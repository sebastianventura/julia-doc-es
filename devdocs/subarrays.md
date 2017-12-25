# [SubArrays](@id subarrays)

El tipo `SubArray` de Julia es un contenedor que codifica una "vista" de un [`AbstractArray`](@ref) padre.  Esta pagina documenta algunos de los principios de diseño e implementación de `SubArray`.

## Indexación: indexación cartesiana vs. lineal

Ampliamente hablando, hay dos formas principales de acceder a los datos en un array. La primera, frecuentemente llamada indexación cartesiana, usa `N` índices para un `AbstractArray` `N` -dimensional. Por ejemlo, una matriz `A` (bidimensional) puede ser indexada en estilo cartesiano como `A[i,j]`. El segundo método de indexación, denominado indexación lineal, usa un solo índice incluso para objetos de mayor dimensión. Por ejemplo si `A = reshape(1:12, 3, 4)`, la expresión `A[5]` devuelve el valor 5. Julia nos permite combinar estos estilos de indexación: por ejemplo, un array 3d `A3` puede ser indexado como `A3[i,j]`, en cuyo caso `i` es interpretado como un índice cartesiano para la primera dimensión, y `j` es un índice lineal sobre las dimensiones 2 y 3.

Para los `Array`s, la indexación lineal apela al formato subyacente de almacenamiento: un array se presenta como un bloque contiguo de memoria, y por tanto el índice lineal es justo el desplazamiento (+1) de la correspondiente entrada relativa al principio del array. Sin embargo, esto no es cierto para muchos otros tipos `AbstractArray`: ejemplos de ello incluyen [`SparseMatrixCSC`](@ref), unos arrays que requieren alguna clase de cálculo (tal como interpolación), y el tipo bajo discusión aquí, `SubArray`. Para estos tipos, la información subyacente es descrita más naturalmente en términos de índices cartesianos. 

Uno puede convertir manualmente un índice cartesiano a uno lineal con `sub2ind`, y viceversa usando ìnd2sub`. Las funciones `getindex` and `setindex!` de los tipos `AbstractArray` puden incluir operaciones similares.

Aunque convertir de un índice cartesiano a uno lineal es rápido (es justo una multiplicación y una suma), convertir de un índice lineal a uno cartesiano es muy lento: se basa en la operación `div`, que es una de las operacions de bajo nivel más lentas que uno puede realizar con una CPU. Por esta razón, cualquier código que trate con tipos `AbstractArray` está mejor diseñado en términos de indexación cartesiana en lugar de lineal.

## Reemplazo de Índices

Considere hacer rebanadas bidimensionales de un array tridimensional:

```julia
S1 = view(A, :, 5, 2:6)
S2 = view(A, 5, :, 2:6)
```

`view` elimina las dimensiones "singleton" (las que están especificadas por un `Int`), por lo que tanto `S1` como `S2` son `SubArray`s bidimensionales. En consecuencia, el camino natural para indexar esto es con `S1[i,j]`. Para extraer el valor del array padre `A`, el enfoque natural es reemplazar `S1[i,j]` con `A[i,5,(2:6)[j]]` y `S2[i,j]` con `A[5,i,(2:6)[j]]`.

La característica clave del diseño de SubArrays es que este reemplazo de índices puede realizarse sin ninguna sobrecarga en tiempo de ejecución.

## Diseño de SubArray's

### Parámetros de Tipo y Campos

La estrategia adoptada está expresada en la definición del tipo:

```julia
struct SubArray{T,N,P,I,L} <: AbstractArray{T,N}
    parent::P
    indexes::I
    offset1::Int       # for linear indexing and pointer, only valid when L==true
    stride1::Int       # used only for linear indexing
    ...
end
```

`SubArray` tiene cinco parámetros de tipo. Los dos primeros son el tipo de elemento estándar y la dimensionalidad. La siguiente es el tipo del `AbstractArray` padre. El usado más intensamente es el cuarto parámetro, una `Tuple` de los tipos de los índices para cada dimensión. El final, `L`,  es sólo proporcionado como una conveniencia para el despacho; es un valor booleano que representa si los tipos del índice soportan indexacion lineal rápida. Más sobre este tema después.  

Si en nuestro ejemplo de arriba `A` es un `Array{Float64, 3}`, nuestro caso `S1` sería un  `SubArray{Int64,2,Array{Int64,3},Tuple{Colon,Int64,UnitRange{Int64}},false}`. Note en particular el parámetro tupla, que almacena los tipos de los índices usados para crear `S1`.  Igualmente,

```julia-repl
julia> S1.indexes
(Colon(),5,2:6)
```

Almacenar estos valores permite el reemplzao de índices, y tener los tipos codificados como parámetros permite a uno despachar a eficientes algoritmos.

### Traducción de Índices

Realizar la traducción de índices requiere que uno haga diferentes cosas para diferentes tipos concretos de `SubArray`. Por ejemplo, para `S1` uno necesita aplicar los índices `i,j` a las dimensiones primera y tercera del array padre, mientras que para `S2` uno necesita aplicarlas a la segunda y la tercera. El enfoque más sencillo a indexar sería hacer el análisis de tipos en tiempo de ejecución:

```julia
parentindexes = Array{Any}(0)
for thisindex in S.indexes
    ...
    if isa(thisindex, Int)
        # Don't consume one of the input indexes
        push!(parentindexes, thisindex)
    elseif isa(thisindex, AbstractVector)
        # Consume an input index
        push!(parentindexes, thisindex[inputindex[j]])
        j += 1
    elseif isa(thisindex, AbstractMatrix)
        # Consume two input indices
        push!(parentindexes, thisindex[inputindex[j], inputindex[j+1]])
        j += 2
    elseif ...
end
S.parent[parentindexes...]
```

Desgraciadamente, esto sería desastroso en términos de rendimiento: cada acceso a elemento asignaría memoria, e implicaría la ejecución de un montón de código pobremente tipado.

El mejor enfoque es despachar a métodos específicos para manejar cada tipo de índice almacenado. Esto es lo que hace `reindex`: él despacha sobre el tipo del primer índice almacenado y consume el número apropiado de índices de entrada, y entonces recurre sobre los índices restantes. En el caso de `S1`, esto expande a

```julia
Base.reindex(S1, S1.indexes, (i, j)) == (i, S1.indexes[2], S1.indexes[3][j])
```

para cualquier par de índices `(i,j)` (excepto `CartesianIndex`s and arrays de este tipo, ver abajo).

Este es el núcleo de un `SubArray`; los métodos de indexación se basan en `reindex` para hacer esta traducción de índices. Sin embargo, algunas veces, podemos evitar la indirección y hacerlo incluso más rápido.

### Indexación Lineal

La indexación lineal puede implementarse de forma eficiente cuando el array completo tiene un solo paso que separe elementos sucesivos, empezando desde cierto desplazamiento. Esto significa que nosotros pre-computamos estos valores y representamos la indexación lineal simplemente como una adición y multiplicación, evitando la indirección de `reindex` y (lo que es más importante) la computación lenta de las coordenadas cartesianas por completo.

Para los tipos `SubArray`, la disponibilidad de una indexación lineal eficiente está basada puramente en los tipos de los índices, y no depende de valores como el tamaño de array padre. Uno puede preguntar si un cnjunto de índices dado soporta indexación lineal rápida con la función interna `Base.viewindexing`:

```julia-repl
julia> Base.viewindexing(S1.indexes)
IndexCartesian()

julia> Base.viewindexing(S2.indexes)
IndexLinear()
```

Esto se calcula durante la construcción del `SubArray` y se almacena en el parámetro de tipo `L` como un boolean que codifica soporte de indexación lineal rápido. Aunque n oes estrictamente necesario, esto significa qeu podemos definir despacho directamente sobre `SubArray{T,N,A,I,true}` sin intermediarios.

Como esta computación no depende de valores en tiempo de ejecución, puede perder algunos casos en los que el paso sea uniforme:

```jldoctest
julia> A = reshape(1:4*2, 4, 2)
4×2 Base.ReshapedArray{Int64,2,UnitRange{Int64},Tuple{}}:
 1  5
 2  6
 3  7
 4  8

julia> diff(A[2:2:4,:][:])
3-element Array{Int64,1}:
 2
 2
 2
```

Una vista construída como `view(A, 2:2:4, :)` tiene un paso uniforme y, por tanto la indexación lineal podría llevarse a cabo eficientemente. Sin embargo, el éxito en este caso depende del tamaño del array: Si, a diferencia del caso anterior, la primera dimensión fuera impar,

```jldoctest
julia> A = reshape(1:5*2, 5, 2)
5×2 Base.ReshapedArray{Int64,2,UnitRange{Int64},Tuple{}}:
 1   6
 2   7
 3   8
 4   9
 5  10

julia> diff(A[2:2:4,:][:])
3-element Array{Int64,1}:
 2
 3
 2
```

entonces `A[2:2:4,:]` no tiene un paso uniforme, por lo que no podemos garantizar indexación lineal eficiente. Como tenemos que basar esta decisión puramente en los tipos codificados en los parámetros del `SubArray`, `S = view(A, 2:2:4, :)` no puede implementar una indexación lienal eficiente.

### Unos pocos detalles

* Note que la función `Base.reindex` es agnóstica a los tipos de los índices de entrada; ella simplemente
  determina como y donde deberían reindexarse los indices almacenados. Ella no solo soporta indices enteros,
  sino que también soporta indexación no escalar. Esto significa que las vistas de vistas no necesitan dos 
  niveles de indirección; ellas pueden simplemente recomputar los índices en el array padre original.  
* Es de esperar que a estas alturas esté bastante claro que soportar rebanadas en arrays significa que la
  dimensionalidad, dada por el parámetro `N`, no es necesariamente igual a la dimensionalidad del array padre
  o la longitud de la tupla `indexes`. Tampoco los índices proporcionados por el usuario se alinean necesariamente
  con las entradas en la tupla `indexes` (por ejemplo, el segundo índice proporcionado por el usuario puede
  corresponder a la tercera dimensión de la matriz padre, y el tercer elemento en la tupla `indexes`).
  
  Lo que podría ser menos obvio es que la dimensionalidad del array padre almacenado sea igual al número de 
  índices efectivos en la tupla `indexes`. Algunos ejemplos:

  ```julia
  A = reshape(1:35, 5, 7) # A 2d parent Array
  S = view(A, 2:7)         # A 1d view created by linear indexing
  S = view(A, :, :, 1:1)   # Appending extra indices is supported
  ```
  
  Ingenuamente, uno pensaría que podría simplemente establecer `S.parent = A` y` S.indexes = (:,:, 1: 1)`, 
  pero el hecho de soportar esto complica dramáticamente el proceso de reindexación, especialmente para 
  vistas de vistas. No solo se necesita despachar los tipos de los índices almacenados, sino que se debe 
  examinar si un índice dado es el último y "fusionar" los índices almacenados restantes. Esto no es una 
  tarea fácil, y aún peor: es lenta ya que depende implícitamente de la indexación lineal.
  
  Afortunadamente, este es precisamente el cálculo que 'ReshapedArray' realiza, y lo hace linealmente si
  es posible. En consecuencia, `view` asegura que el array padre es la dimensionalidad adecuada para los 
  índices dados mediante reformateo (*reshaping*) si es necesario. El constructor interno `SubArray` 
  asegura que este invariante esté satisfecha.
* `CartesianIndex` y sus matrices retuercen de una forma desagradable el esquema `reindex`. Recuerde 
  que `reindex` simplemente despacha sobre el tipo de índices almacenados para determinar cuántos 
  índices pasados deberían usarse y a dónde deberían ir. Pero con `CartesianIndex`, ya no hay una 
  correspondencia uno a uno entre la cantidad de argumentos pasados y la cantidad de dimensiones en 
  las que indexan. Si volvemos al ejemplo anterior de `Base.reindex(S1, S1.indexes, (i, j))`, puede 
  ver que la expansión es incorrecta para `i, j = CartesianIndex (), CartesianIndex (2,1 )`. Él 
  debería *salta* el `CartesianIndex()` por completo y devolver:

  ```julia
  (CartesianIndex(2,1)[1], S1.indexes[2], S1.indexes[3][CartesianIndex(2,1)[2]])
  ```

  Y si embargo, lo que devuelve es:

  ```julia
  (CartesianIndex(), S1.indexes[2], S1.indexes[3][CartesianIndex(2,1)])
  ```

  Hacer esto correctamente requeriría el envío *combinado* en los índices almacenados y pasados en 
  todas las combinaciones de dimensionalidades de una manera intratable. Como tal, `reindex` nunca 
  debe invocarse con índices `CartesianIndex`. Afortunadamente, el caso escalar se maneja fácilmente 
  aplanando primero los argumentos `CartesianIndex` a enteros simples. Sin embargo, las matrices de 
  `CartesianIndex` no se pueden dividir en piezas ortogonales tan fácilmente. Antes de intentar usar 
  `reindex`,`view` debe garantizar que no haya matrices de `CartesianIndex` en la lista de argumentos. 
  Si los hay, simplemente puede "puntualizar" evitando por completo el cálculo de 'reindex', construyendo 
  un `SubArray` anidado con dos niveles de indirección en su lugar.
