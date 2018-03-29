# [Carga de módulos](@id require)

`Base.require`[@ref] es responsable de cargar los módulos y también maneja la caché de precompilación.
Es la implementación de la instrucción `import`.

## Características experimentales
Las características siguientes son experimentales y no forman parte de la API estable de Julia. 
Antes de desarrollarlos, infórmese sobre su estado actual y si podrían cambiar pronto.

### Retrollamadas de carga del módulo

Es posible escuchar los módulos cargados por `Base.require`, registrando una devolución de llamada.

```julia
loaded_packages = Channel{Symbol}()
callback = (mod::Symbol) -> put!(loaded_packages, mod)
push!(Base.package_callbacks, callback)
```

Tenga en cuenta que el símbolo dado a la devolución de llamada es un identificador no único y es responsabilidad del proveedor de devolución de llamada recorrer la cadena de módulos para determinar el nombre completo del enlace cargado.

La devolución de llamada a continuación es un ejemplo de cómo hacerlo:

```julia
# Get the fully-qualified name of a module.
function module_fqn(name::Symbol)
    fqn = Symbol[name]
    mod = getfield(Main, name)
    parent = Base.module_parent(mod)
    while parent !== Main
        push!(fqn, Base.module_name(parent))
        parent = Base.module_parent(parent)
    end
    fqn = reverse!(fqn)
    return join(fqn, '.')
end
```

