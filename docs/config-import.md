# Configurations can import other configurations

## Summary

Allow a `.config` module to resolve values from another `.config` module by running it in the `.config` sandbox.

## Motivation

### Automation versus user intent

We would like to make it possible for build systems, projects, and package managers to cleanly make use of require-by-string.

In doing so, we found a core conflict:

- The `.config` module is the only module the runtime reads to determine alias location
- The `.config` module is _also_ taken to be user intent, and should not be overwritten

This leaves automations that update aliases nowhere to record _their_ intent cleanly without possibly overriding user intent.

### Sharing configuration widely

We also recognise the existence of a third problem: some configurations are shared widely, and we want to lower the cost of updating these values across multiple configuration files and ensure they don't fall out of sync with each other.

The typical example is a build system that synchronises metadata between a workspace of packages.

At the root of the workspace may be specific metadata:

```lua
-- workspace/shared.config.luau

return {
    version = "1.2.3"
}
```

Which may then be explicity synchronised by path into a package's own config somehow:

```lua
-- workspace/my-module/pkg.config.luau
local workspace = getConfigSomehow("../shared.config.luau")

return {
    name = "MyModule",
    version = workspace.version,
    -- ... etc ...
}
```

In previous workspace systems (such as the one built into Rotriever) this was an implicit capability hardcoded into the workspace system.

This causes a lot of problems:

- Every workspace system defines its own rules for config inheritance - Rotriever's `{ workspace = true }` hardcodes you to a particular file structure and `config.toml` location, for instance.
- These rules often _conflict_ with other build tooling, as we experienced when migrating Jest to dual Wally/Rojo/OCALE and Rotriever/RBXP/roblox-cli - a structure that worked for one system almost always did not work for the other.

It is our view that implicit hardcoded capabilities in these systems are leading to a more fragmented and incompatible ecosystem over time. These incompatibilities arise to fill a gap in the configuration spec that people externally _and_ internally find they need, but don't have general tools to solve.

Towards our goal of users being able to freely choose different interoperable tooling, we believe this motivates a slightly stronger configuration spec.

## Design

We propose the introduction of a new function - distinct from `require` - which loads Luau files in a `.config` sandbox. It is available for use in all kinds of Luau module, configuration or not.

To allow the library to be permitted in isolation in the `.config` sandbox, we locate it in a special `@std/config` library, where we can guarantee all members operate in a config environment. All other requires still fail to resolve in order to keep the config sandbox sealed.

```lua
-- workspace/my-module/pkg.config.luau
local config = require("@std/config")
local workspace = config.load("../shared.config.luau")

return {
    name = "MyModule",
    version = workspace.version,
    -- ... etc ...
}
```

Some nuances of this function's behaviour:

- `config.load` may cache its returned result, as configuration files should not have side effects, and this permits greater efficiency when dealing with widely-used configuration files
- `config.load` errors when it detects a cyclic load between two modules.
- In a config environment, the path passed to `config.load` does not recognise any config-provided aliases. It only recognises built-in aliases such as `@self`.

## Drawbacks

Why should we *not* do this?

## Alternatives

What other designs have been considered? What is the impact of not doing this?

## Prior Art

* Do other programming languages have similar features? What do those look like? How do they compare and contrast to this design? Are there unique constraints for Luau that interact with said design?
* What supporting features do those languages have that might _not_ be included in this design? If we're adding something feature A, are there features B, C, and D that are often used with A?
* Are there other similar features or libraries in Luau already? How does this feature align with _those_ features in terms of naming, syntax, and/or semantics?
