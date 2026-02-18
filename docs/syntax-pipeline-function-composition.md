# Pipeline operator for composing function calls

## Summary

Add a new pipeline operator that allows a value to be passed through multiple composed functions without increasing bracket level.

A function call composition like this:

```luau
h(
  g(
    f(5)
  )
)
```

Could be expressed flatly as a linear pipeline:

```luau
5
|> f()
|> g()
|> h()
```

This mirrors pipeline operators in languages like F#, OCaml, Hack, Julia and Elixir.

## Motivation

Composition is a key part of modern clean code; it encourages modular design of functions with single responsibilities.
For example, the `string` and `tableext` libraries contain multiple operations which are small and well-scoped.
These operations can be combined to build more complex operations.

```luau
local tableext = require("@std/tableext")

local function parse_csv(raw_text: string): {{string}}
    local lines = string.split(string.gsub(raw_text, "\n?\r", "\n"), "\n")
    local cells = tableext.map(lines, function(line) return string.split(line, ",") end)
    return cells
end
```

The problem with function composition is that it can obsfucate the location of the primary operand, and visually reverses the order of the functions being composed.
In the above snippet, the programmer would read `string.split` before `string.gsub`, and only discovers that it is operating on `raw_text` after reading past the middle of the line.

In addition to this, nested parentheses that span multiple lines do not cleanly diff in version control software, as it pollutes the patch with incidental changes to indentation level or trailing punctuation.

In an attempt to more ergonomically express the pipeline of operations, users may attempt to chain methods.
Some Luau primitives like strings expose such methods, which we don't prefer users to rely on because it complicates static analysis and inhibits some function call optimisations.

```luau
local lines = raw_text:gsub("\n\r", "\n"):split("\n")
```

It also relies on the operand exposing the required methods as indexable members.
That means it is dangerous to extend to newer libraries like `tableext`, which arguably stand to benefit _more_ from ergonomic compositions due to new map/filter/reduce APIs being introduced for list manipulation.

To fix this ideally without a performance hit and with forwards compatibility, we would want to look up functions in the current scope, not inside of the operand, such that Luau can trivially guarantee which function will be called at each step.

Most programming languages solve this in one of two ways;
- Universal function call syntax (aka UFCS, or "functions = methods").
- Functional-style pipeline operators.

We discard UFCS as Luau already has methods - see Alternatives for more on this.
The rest of this document focuses on the design of a pipeline operator which is distinct from method syntax.

## Design

We select `|>` as the pipeline operator token as it is not valid Luau syntax, and mirrors the popular token choice found in F#, OCaml, etc.

- The operand is placed on the left. After the token, some kind of function call expression is required.
- Pipeline operators can be chained; functions are composed in reverse order of appearance such that data flows from start to end.
- Similar to a method call, the first argument to the function is the operand, followed by the arguments listed in the parentheses.
- The name can be replaced with any valid left-hand-side expression that evaluates to a compatible function (e.g. member index).

```luau
{1, 2, 3}
|> tableext.map(function(x) return x * 2 end)
|> tableext.reverse()

"hello" |> print("world") -- hello world
```

The motivating code snippet is simplified without loss of performance or change of behaviour.

```luau
local tableext = require("@std/tableext")

local function parse_csv(raw_text: string): {{string}}
    return raw_text
    |> string.gsub("\n?\r", "\n")
    |> string.split("\n")
    |> tableext.map(function(line) return string.split(line, "\n") end)
end
```

## Drawbacks

- Pipeline operators are "yet another symbol" for developers to memorise. 
- Some users may feel that the choice of `|>` as the operator token is incongruent with the keyword-based syntax used for statements elsewhere in Luau.
- A similar issue to method calls may arise, where it is not obvious that the operand is prepended to the argument list, due to it not appearing between the parentheses.
    - [A TC39 proposal](https://github.com/tc39/proposal-pipeline-operator/issues/91) explored explicitly binding the operand to an identifier rather than silently passing it to the function, but no consensus on a suitable syntax was reached.



## Alternatives

### Other considered tokens

Other tokens were considered as part of this RFC:

- Tokens that "reflect" the method call syntax: `:>`, `>:`, `|:`, etc.
    - Since this is a function call, not a method call, it didn't feel right to use the colon in the token. This might imply relationship to `:method()`, which looks up the function on the operand.
- The bare pipe `|`.
    - This is not valid in expressions, but it _is_ valid in types, where it is used to create unions.
    - This also looks like bitwise OR in other programming languages adjacent to Luau, such as JavaScript.
- The right arrow `->`.
    - This is used in type definitions to indicate the return type of a function, and was previously used in function definitions.
    - While it may technically be usable, it felt overloaded as a token for this use case.
- Double right angle brackets `>>`.
    - This could easily come into conflict with generic syntax e.g. `foo<<T>>()`.

### Universal Function Call Syntax (UFCS)

Languages like [Vale](https://vale.dev/guide/functions#functions--methods) treat method calls as syntax sugar for function calls where the first argument is the operand.
Importantly, these functions are taken from the current namespace, not from the operand.

In Luau, this might have looked like this:

```luau
local function process(input: string, delim: string)
    -- do processing...
end

local parts = ("Hello, world"):process(";")
```

However this is clearly not backwards compatible, as method call syntax is defined in Luau to index into its operand, not into the environment.
Methods are already their own concept in Luau and this RFC makes no attempt to modify what is considered as a "method'.
