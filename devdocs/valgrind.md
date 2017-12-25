# [Usando Valgrind con Julia](@id valgrind)

[Valgrind](http://valgrind.org/) es una herramienta para deputación de memoria, deteccin de fallos de página y creación de perfiles. Esta sección describe cosas a tner en cuenta cuando se usa Valgrind para depurar cuestiones de memoria con Julia.

## Consideraciones Generales

Por defecto, Valgrind asume que no hay código automodificador en el programa que está ejecutando. Esta suposición trabaja bien en la mayoría de las instancias, pero falla miserablemente para un compilador JIT como `julia`. Por esta razón es crucial pasar `--smc-check=all-non-file` a `valgrind`, si no el código puede bloquearse o comportarse de forma inesperada (frecuentemente de una forma sutil).

En algunos casos, para detectar mejor errores de memoria usando Valgrind puede ayudar compilar `julia` con los *pools* de memoria dehabilitados. El flag de tiempo de compilación `MEMDEBUG` desabilita los *pools* de memoria en Julia y el flag `MEMDEBUG2` deshabilita los pools de memoria en FemtoLisp. Para construir `julia` con ambos flags, añada la siguiente línea a `Make.user`:

```julia
CFLAGS = -DMEMDEBUG -DMEMDEBUG2
```

Otra cosa a notar: si nuestros programa usa múltiples procesos workers, es probable que queramos que todos esos procesos worker se ejecuten bajo Valgrind, no sólo bajo el proceso padre. Para hacer esto, pasaremos `--trace-children=yes` a `valgrind`.

## Supresiones

Valgrind tipicamente mostrará avisos falsos mientras se ejecuta. Para reducir el número de tales avisos, ayuda proporcionar un [fichero supresiones](http://valgrind.org/docs/manual/manual-core.html#manual-core.suppress) a Valgrind. Un fichero supresiones de ejemplo se incluye en la distribución fuente de Julia en `contrib/valgrind-julia.supp`.

El fichero de supresiones puede ser usado desde el directorio fuente `julia/` de la siguiente manera:

```
$ valgrind --smc-check=all-non-file --suppressions=contrib/valgrind-julia.supp ./julia progname.jl
```

Any memory errors that are displayed should either be reported as bugs or contributed as additional
suppressions.  Note that some versions of Valgrind are [shipped with insufficient default suppressions](https://github.com/JuliaLang/julia/issues/8314#issuecomment-55766210),
so that may be one thing to consider before submitting any bugs.

## Running the Julia test suite under Valgrind

It is possible to run the entire Julia test suite under Valgrind, but it does take quite some
time (typically several hours).  To do so, run the following command from the `julia/test/` directory:

```
valgrind --smc-check=all-non-file --trace-children=yes --suppressions=$PWD/../contrib/valgrind-julia.supp ../julia runtests.jl all
```

If you would like to see a report of "definite" memory leaks, pass the flags `--leak-check=full --show-leak-kinds=definite`
to `valgrind` as well.

## Caveats

Valgrind currently [does not support multiple rounding modes](https://bugs.kde.org/show_bug.cgi?id=136779),
so code that adjusts the rounding mode will behave differently when run under Valgrind.

In general, if after setting `--smc-check=all-non-file` you find that your program behaves differently
when run under Valgrind, it may help to pass `--tool=none` to `valgrind` as you investigate further.
 This will enable the minimal Valgrind machinery but will also run much faster than when the full
memory checker is enabled.
