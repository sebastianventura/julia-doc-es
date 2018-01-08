# [Puntuación](@id punctuation)

Puede encontrar documentación extendida sobre símbolos y funciones matemáticas [aquí](@ref math-ops).

| Símbolo     | Significado                                                                                 |
|:----------- |:------------------------------------------------------------------------------------------- |
| `@m`        | Invoca la macro `m`; seguido de expresiones separadas por espacios                          |
| `!`         | Operador "not" prefijo                                                                      |
| `a!( )`     | Al final de un nombre de función, `!` indica que la función modifica su(s) argumento(s)     |
| `#`         | Inicio de un comentario de una sola línea                                                   |
| `#=`        | Inicio de un comentario multilínea (pueden ser anidados)                                    |
| `=#`        | Final de un comentario multilínea                                                           |
| `$`         | Interpolación de cadena y expresión                                                         |
| `%`         | Operador resto                                                                              |
| `^`         | Operador exponenente                                                                        |
| `&`         | Operador and bit-a-bit                                                                      |
| `&&`        | Operador and booleano (en corto-circuito)                                                   |
| `\|`        | Operador or bit-a-bit                                                                       |
| `\|\|`      | Operador or booleano (en corto-circuito)                                                    |
| `⊻`         | Operador xor bit-a-bit                                                                      |
| `*`         | Multiplicación o producto matricial                                                         |
| `()`        | Tupla vacía                                                                                 |
| `~`         | Operador not bit-a-bit                                                                      |
| `\`         | Operador backslash                                                                          |
| `'`         | Operador transpuesto complejo Aᴴ                                                            |
| `a[]`       | Indexación de array                                                                         |
| `[,]`       | Concatenación vertical                                                                      |
| `[;]`       | Concatenación vertical (también)                                                            |
| `[    ]`    | Con expresiones separadas por espacios, concatenación horizontal                            |
| `T{ }`      | Instanciación de tipo paramétrico                                                           |
| `;`         | Separador de instrucciones                                                                  |
| `,`         | Separador de argumentos de función o de componentes de una tupla                            |
| `?`         | Operador condicional ternario (conditional ? if_true : if_false)                            |
| `""`        | Delimitador de literales cadena                                                             |
| `''`        | Delimitador de literales carácter                                                           |
| ``` ` ` ``` | Delimitador de especificaciones de proceso externo (mandato)                                    |
| `...`       | Une argumentos en una llamada a función o declara una función o tipo varargs                    |
| `.`         | Acceso nombrado a campos en objectos/módulos, también llamadas a operadores/funciones vectorizadas |
| `a:b`       | Rango a, a+1, a+2, ..., b                                                                       |
| `a:s:b`     | Rango a, a+s, a+2s, ..., b                                                                      |
| `:`         | Indexa una dimensión entera (1:final)                                                       |
| `::`        | Anotación de tipo, dependiendo del contexto                                                 |
| `:( )`      | Expresión citada                                                                            |
| `:a`        | Símbolo a                                                                                   |
| `<:`        | [`subtype operator`](@ref <:)                                                               |
| `>:`        | [`supertype operator`](@ref >:) (contrario al anterior)                                     |
| `===`       | [`egal comparison operator`](@ref ===)                                                      |
