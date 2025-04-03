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

```Lua
local module = os.loadmodule("../foo/bar")
print(typeof(module)) --> module

local module2 = os.loadmodule("../foo/bar")
print(module == module2) --> false
```

`loadmodule()` accepts a string path in all Luau environments according to the require-by-string RFC. It may also accept other environment-specific module paths, like Roblox instance references. 

*(Open question: should we get rid of asset ID requires?)*

When passed into `require`, the loaded module will be run, and the result will conceptually be cached "inside of" the module object. That is to say, if two modules are `==`, they will share the same cache.

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

(Note that this does not change the current global caching behaviour of `require()` for non-`module` arguments.)

When no references are left to a `module` object, it will be garbage collected, and the Luau VM is allowed to free any loaded module information or caches related to that module. Garbage collection behaviour is the key motivator for `module` objects.

```Lua
do
    -- module is loaded here
    local myModule = os.loadmodule("../foo/bar")
    -- the returned value is internally cached here
    local data = require(myModule)
    -- the cache can be used here
    local data2 = require(myModule)
end
-- because `myModule` is no longer accessible, its cache and loaded info can be
-- safely disposed of without observable side effects
```

These objects are intentionally not compatible with `setfenv()` or `getfenv()`.

```Lua
getfenv(module) --> invalid argument #1 to 'getfenv' (number expected, got module)
```

`loadmodule` accepts an optional table to configure how the module is loaded.

For now, only a minimal set of static sandboxing options are provided, to allow
for very basic modifications without incurring significant overhead.

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

Allowing users to bypass the internal module cache means it's possible for people to instantiate the same module many times. Done naively, this could easily become a memory leak. This proposal addresses the issue by using a garbage-collected handle to automatically clean up modules that are no longer needed, but this doesn't protect against users holding strong references to modules indefinitely. In any case, this is better than the status quo in Roblox, which involves cloning module scripts and which suffers the same problem with no solution.

Require calls that accept a module may be harder to statically analyse, as modules can be loaded and stored dynamically. To some extent, this is already true with all requires that don't directly accept literals, but it should be a consideration nonetheless. The predicted common use cases for `loadmodule()` skew towards dynamic content, but it's possible that someone may want to simply require a unique copy of a module to ensure state separation, in which case this may harm intellisense.

By allowing users some control over the environment, we may have to disable some optimisations for those modules specifically. This is the biggest unknown; while great effort has been made to minimise the number of dynamic features and ensure they mesh well with the caching system, there is always the possibility that some optimisations depend on specific details of the environment. However, even if not optimal, such a solution may still perform tangibly better than the status quo, which can easily double the runtime of certain code snippets as shown by internal benchmarks.

## Alternatives

### Return a function chunk instead of a module object

In theory, `loadmodule()` could return a function chunk instead of a module object:

```
local module = os.loadmodule("../foo/bar")
print(typeof(module)) --> function

local module2 = os.loadmodule("../foo/bar")
print(module == module2) --> false

local returnedValue = module()
```

This would be more in line with the behaviour of `loadstring` and the internal `debug.loadmodule`.
However, these older APIs are also often used with functions like `getfenv()`,
which we would ideally not have to support for our module loading system.

Furthermore, it would become more awkward to track when it is safe to clean up
a module's cached information. The existing internal workaround returns a
cleanup callback for this purpose, but this could easily be overlooked,
especially since it's the third returned value and can be dropped during
assignment with no consequences.

```
-- the cleanup function is dropped here! this will leak memory
moduleFunction = debug.loadmodule(modulePath)
```

Additionally, if the user tries to print the function chunk, there's no
opportunity to provide a useful debug representation:

```lua
print(module) --> function: 0x123456789abcdef
```

For this reason, it was chosen to return a module object instead, which can
automatically free any caches on collection, which is explicitly incompatible
with function-based APIs, and which can provide convenient debugging behaviour.

### Don't do anything

There is a large cost to not doing anything. Roblox's own Jest library relies
inseparably on cache control and sandboxing for modules, which is used widely
on very large codebases. Internal benchmarks demonstrate a substantial
performance hit from using today's existing workarounds as most user code runs
without optimisations enabled.

Furthermore, there is a hidden memory cost for today's status quo. A common way
to work around Roblox's implementation of the require cache is to clone module
instances so that they are freshly cached each time. Since this cache remains
until the end of the session, the memory usage of the project will increase
monotonically with each require. Some projects have been observed doing this in
the wild, showing that there is user demand for this feature. As such, by not
implementing this, we continue to encourage excessive memory consumption.