# [Funciones del Administrador de Paquetes](@id pkg)

Todas las funciones del administrador de paquetes están definidas en el módulo `Pkg`. Ninguna de las funciones del módulo `Pkg` están exportadas. Por tanto, para usarlas, necesitamos prefijar cada llamada a función con un `Pkg.` explícito, por ejemplo [`Pkg.status()`](@ref) o [`Pkg.dir()`](@ref).

Las funciones para el desarrollo de paquetes (por ejemplo, `tag`, `publish`, etc.) se han movido al paquete [PkgDev](https://github.com/JuliaLang/PkgDev.jl). Ver [PkgDev README](https://github.com/JuliaLang/PkgDev.jl/blob/master/README.md) para la documentación de estas funciones.

```@docs
Base.Pkg.dir
Base.Pkg.init
Base.Pkg.resolve
Base.Pkg.edit
Base.Pkg.add
Base.Pkg.rm
Base.Pkg.clone
Base.Pkg.setprotocol!
Base.Pkg.available
Base.Pkg.installed
Base.Pkg.status
Base.Pkg.update
Base.Pkg.checkout
Base.Pkg.pin
Base.Pkg.free
Base.Pkg.build
Base.Pkg.test
Base.Pkg.dependents
```
