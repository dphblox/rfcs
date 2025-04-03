# `os.loadmodule`

## Summary

Introduce a new `os.loadmodule()` function to allow Luau users to portably and safely configure the module loading process.

## Motivation

There is no good way for consumers of Luau to sandbox a module or bypass the default require cache in a manner that is portable across Luau environments.

For sandboxing, there is no solution other than to use the deprecated & deoptimising `setfenv()` or `getfenv()` functions, or the unsafe & deoptimising `loadstring()` function. Because they're needlessly powerful, they inhibit Luau's internal optimisations and therefore are rightly discouraged. However, sandboxing must use these functions, even though it doesn't really need all that power. There is perhaps space for a much less powerful feature to serve valid sandboxing use cases without going quite so overboard.

```Lua
local doModule = loadstring(myModule.Source)

-- Deny use of the `task` library and intercept prints.
local env = table.clone(getfenv())
env.task = nil
env.print = function(...)
    print("[Sandboxed]", ...)
end
setfenv(doModule, env)

doModule()
```

For cache control, internal workarounds have been instituted for many of Roblox's own libraries already (viewable in our public OSS libraries) but these are not exposed publicly as they have not been through a proper API design process. There is consensus that we would not want to ship this internal workaround entirely as-is.

```Lua
moduleFunction, errorMessage, cleanupFn = debug.loadmodule(modulePath)
```

Thus, the goal of this RFC is to reconcile these two things together, and propose a single API that provides proper control over how modules are loaded and configured.

## Design

A new `loadmodule()` function will be added to the `os` library (not the `debug` library). This function loads in a unique copy of a module and returns an opaque `module` object representing it.

It accepts a string path in all Luau environments according to the require-by-string RFC. It may also accept other environment-specific module paths, like Roblox instance references. 

*(Open question: should we get rid of asset ID requires?)*

```Lua
local module = os.loadmodule("../foo/bar")
print(typeof(module)) --> module

local module2 = os.loadmodule("../foo/bar")
print(module == module2) --> false
```

When passed into `require`, the loaded module will be run, and the result will conceptually be cached "inside of" the module object. That is to say, if two modules are `==`, they will share the same cache.

This does not change the current global caching behaviour of `require()` for non-`module` arguments.

```Lua
-- side-effect.luau
print("I'm being loaded")
return math.random(1, 5)
```

```Lua
-- main.luau
local first = os.loadmodule("./side-effect")
local second = os.loadmodule("./side-effect")

print(require(first))
--> I'm being loaded
--> 2
print(require(first))
--> 2
print(require(first))
--> 2

print(require(second))
--> I'm being loaded
--> 5
print(require(second))
--> 5
print(require(second))
--> 5

print(require(first))
--> 2
```

When no references are left to a `module` object, it will be garbage collected, and the Luau VM is allowed to free any loaded module information or caches related to that module. Garbage collection behaviour is the key motivator for `module` objects.

`loadmodule` accepts an optional table parameter for accepting further options. Any options left unspecified are assumed to have their default value. A table was chosen to allow for the possibility of including more options in the future.

Note that options do not transitively apply to nested modules; only the directly loaded module will use these options.

```Lua
local MY_LABEL = "[Sandboxed]"
local foo = os.loadmodule("../foo/bar", {
    -- Default: true
    -- When enabled, all normal environment members (including libraries like
    -- `utf8` or globals like `print`) are accessible.
    defaultenv = true,
    -- Default: {}
    -- Overwrites environment members statically. This table is cloned, so
    -- the environment can't be mutated after creation. Metamethods are ignored,
    -- but functions are passed by reference so they can use upvalues.
    env = {
        task = { value = nil } -- wrapped so that `nil` can be passed
        print = { value = function(...)
            print(MY_LABEL, ...) -- functions can use upvalues
        end }
    }
})
```

## Drawbacks

TODO

## Alternatives

TODO
