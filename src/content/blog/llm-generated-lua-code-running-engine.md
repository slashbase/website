
---
title: "An LLM generated Lua code running engine in Go"
description: "I built Layerengine, an open source Go library that turns a YAML flow spec into generated code."
pubDate: 2026-11-20
author: Paras Waykole
---

TLDR; I built Layerengine, an open source Go library that turns a YAML flow spec into generated code. An LLM writes one small Lua function per step, the library compiles it with gopher-lua, and runs it in a locked-down Lua state with a 60-second timeout, threading each step's named outputs into the next step's inputs.

What I wanted was narrow on purpose: not a model that decides what to do next, but one that writes a single function against a contract somebody else already wrote down. A function has a signature, a signature can be checked before the code runs, and checked code can run in a box you can describe in a paragraph.

Agentic coding is a development tool. You point a model at a repository and it explores, edits, runs things, reads its own errors and tries again, while a person decides when it is done. The model is the author and the source tree is the artifact, so what you review is what it wrote. Its authority is ambient — a shell, a filesystem, whatever it decides to install — which is exactly what makes it useful and exactly what leaves nothing to bound except the machine.

Layerengine puts the model somewhere else: inside a program that is already running, with no developer present. The artifact is a spec, not a source tree; the function is what the spec lowers to, and it can be lowered again when the contract or the model changes. The model never sees the flow's order or its branches — that sequence is a list in a YAML file a person wrote and can read — and it is handed one step of it at a time. That step runs against the module set the engine was given, each module bound as a global under its own name, and nothing else — so the reachable surface is exactly the modules somebody chose to load. The boundary is around a function rather than a machine: a signature to check before the code runs, a box to run it in, and a set of capabilities somebody named on purpose. Agentic coding ends with a tool you use. This ends with a program that writes the code from the spec, checks it, and then runs it.

## The spec is the program

A flow is a YAML file: a name, typed inputs, and an ordered list of layers. A layer says what it takes in, what it does in prose, and what it gives out. There is no code in the spec, no expressions, no conditionals. This is the repo's arithmetic example, trimmed of its optional multiplier input and its multiply layer to keep it short:

```yaml
name: arithmetic
input:
  - number_a: { type: number, description: first number }
  - number_b: { type: number, description: second number }
layers:
  - name: add_numbers
    description: adds number_a and number_b together
    input: [number_a, number_b]
    output:
      - sum_result: { type: number, description: the sum of number_a and number_b }
  - name: format_amount
    description: formats the result into the string "Your final amount is ₹{number}"
    input: [sum_result]
    output:
      - amount_string: { type: string, description: the formatted rupee amount string }
```

The spec gets validated by hand, walking the YAML node tree rather than unmarshalling into structs, because the useful errors need line numbers. A bad spec reports every problem it has in one pass, not the first one.

Most of what it checks is housekeeping: a name on every layer, no duplicates, no unknown keys, no type outside the six that exist. The check that earns its place is the last one. It walks the layers from the top, carrying a pool of names that are in scope at each point — the flow's inputs to begin with, then each layer's outputs as they are declared — and it refuses a layer that reaches for something not yet in the pool. The error tells you what was in scope when the layer was read:

```
layer[2] "format_amount" requires input "multiplied_result" but it is not available at this point
        available pool: [number_a, number_b, sum_result]
```

That refusal covers forward references, misspellings, and the whole family of spec that reads fine until layer four needs something nothing produced. And because the pool is built by walking the list in order, the spec's order is the execution order — there is no separate graph to keep in step with it.

## What the model is actually asked for

Each layer is generated on its own. The request has three parts: the module surface as a system message, a short task instruction as a second, and the layer's contract as the user message. The contract is plain text — function name, description, the inputs with types, the outputs with types — followed by two rules that carry more weight than they look like they do:

> Lua function parameter names must exactly match the input names above.
>
> The function must return all the values directly.

The runner asks Lua for exactly as many return values as the layer declares outputs, positionally: `CallByParam` with `NRet` set to the output count. Lua pads a shortfall with nil instead of complaining, so a function that returns fewer values than the layer declares does not fail at all. The missing outputs arrive as nil, and that nil travels into the next layer, which then fails for a reason that names neither the layer nor the cause. Wrong order is quieter still: every value arrives and every value is wrong. Nothing type-checks an output on the way out, so the declaration is a contract with the model rather than with the runtime — which is why the prompt spells the rule out instead of leaving it to convention.

Above that sits the module block, generated from whatever is registered on the engine rather than hardcoded. Each function renders as the line a module author would write by hand:

```
celsius:number = weather.temperature(city:string) - Temperature in degrees Celsius for a city.
```

and above *that*, a flat statement of what the environment is: base, `string`, `table`, `math` and the module tables exist; `os`, `io`, `debug`, `package`, `require`, `dofile`, `loadstring` and the rest do not. Modules and functions are sorted before rendering, so the same module set always produces the same prompt bytes — which is not the same as the same output, but it means a difference in output is the model's, not the prompt's.

Both providers sit behind one `CodeGen` type, on current SDKs, and both are asked for JSON with a single `code` field. Retries are the SDKs' own, twice by default, honouring `Retry-After`. The library carries no retry code, which is the right amount.

## Load-time is where you find out

`LoadSpec` is all or nothing, and it stages the work so a failure leaves nothing behind. Each chunk is parsed and compiled, then executed in a throwaway sandbox to prove it defines a global function named after its layer. Compiling proves the Lua parses; it does not prove the chunk defines `format_amount` instead of `formatAmount`, and the runner looks the function up by name at call time. Catching it at load turns a nil-call inside a run into a message that names the layer and the `function <name>(...)` line it wanted.

Then one write lock installs everything. Layer names are engine-global: two flows cannot both own a `format` layer with different code, and the error names the flow that already holds the name. Identical code is shared instead, which makes re-loading the same spec idempotent and keeps pointers already held by a registered flow valid.

## A run

One `map[string]any` holds every name. A layer reads its declared inputs out of it by name, and its returned values are written back under the names it declared. The last layer's returns are the result. A name nobody produced reads as nil, which the Lua sees as `nil`.

Two things make that bearable in practice. Caller arguments are coerced to the declared types first — a numeric string to a number, a number to a string, `"true"` to a boolean — with a strict mode and a callback for dropped arguments when you want them. And `RunFlowResult` returns the last layer's outputs *keyed by name*, with the layer that produced them, for the case the positional slice handles badly: a caller running a spec it did not write.

Every run method also takes a context. The per-layer 60-second timeout is derived from it, so cancelling stops the layer that is running instead of waiting the limit out, and the error wraps `context.Canceled` rather than reporting a timeout. Without a context, a ten-layer flow is bounded only by ten per-layer limits.

## The box

Each layer gets a fresh Lua state, built with the standard libraries skipped so the set is explicit. Four are opened: base, table, string, math. Then the dynamic loaders are nil'd — `load`, `loadstring`, `dofile`, `getfenv`, `setfenv`, `collectgarbage` — so generated code cannot compile new chunks or reach the environment table. The call stack is capped at 256 frames and the registry at 5120 slots at construction, with the max equal to the initial size so it cannot grow. When the layer returns, the state is closed.

So a layer cannot read a file, spawn a process, read the environment, exit the process, or introspect the runtime. It cannot loop forever — 60 seconds, then the context dies. It cannot leak a global into the next layer, because there is no next layer in that state.

Memory was the piece I spent the most time on. The guard samples the live heap every 50 ms and cancels the layer if it has grown past roughly 256 MiB over where it started. Two details are not obvious. The sample comes from `runtime/metrics` rather than `runtime.ReadMemStats`, because `ReadMemStats` stops the world on every call, and a 50 ms poll per layer means twenty pauses a second per concurrent flow. And the comparison is against a baseline taken when the layer starts, not the absolute heap, so one busy process does not fail layers that allocated nothing. The cancel carries a cause, so the error says "layer exceeded memory limit" rather than "context canceled".

It is still a poll, not a wall. One huge allocation can overshoot before the ticker looks, and the number is the process heap, not the layer's. Capability isolation and resource isolation are different mechanisms, and only the first one is finished here.

## Modules are the only way out

A module is how you extend the engine. Left alone, a layer is pure computation: it can add numbers, walk a table, format a string, and that is the whole of its world. A module is a capability you add to that world — an external API, a database, a payments client, anything the flow needs to reach beyond arithmetic. You write it in Go, register it on the engine, and from then on every layer can call it, and the model is told it exists.

Concretely it is a Go type with four methods: a name, an `Init` that returns a table of functions, prose, and a manifest. The manifest declares the surface as data — functions, params with types and defaults, named returns, docs — and that declaration is what renders into the prompt, so the model gets exact arity instead of parsing English. Registration then holds the module to it: the name has to be free in a real sandbox (a module called `string` would replace the string library), and every path the manifest claims has to resolve to a function on the table `Init` actually built.

Which gives the security model in one line: a module should own its endpoint and take only the arguments the layer supplies. `inventory.reserve(sku, quantity)` is a capability — one endpoint, two arguments, both of them named in the spec.

## The examples that convinced me

The early examples were proof of concept, written to check that the idea held together at all: one adds two numbers, one prints fizzbuzz. They show a spec becoming a running function. The recent ones are why I think this is more than an experiment. They are real jobs — a budget report, a sudoku solver — not demos built to show the engine off.

`personal-expenses` is five layers over a month of spend: parse a ledger and count malformed lines, normalise categories and round every amount to two decimals, check totals against rupee caps, work out savings and a verdict, format the report. It runs the same ledger three times — ₹95,000 income, then ₹70,000, then ₹95,000 with a 40% savings target — and only the savings branch moves: the verdict walks from "on track" to "overspending" and then moves again on the target alone, while every rupee of spend stays put. The rounding formula is written into the layer description on purpose, because float drift from adding twenty amounts can otherwise push a total past a whole-rupee cap.

`sudoku-solve` is where I stopped guessing about the ceiling. Five layers: parse the grid, check the givens for rule violations before anything searches, solve by backtracking, verify the answer independently, render the board. The solver's description is a paragraph of prose asking for a specific shape of code:

> Define a local function `legal(i, v)` that scans the row, the column and the 3x3 box containing position `i` … Define a local recursive function `search()` declared as "local function search()" so it can call itself … Do not use `loadstring`, `load`, `require` or any other dynamic loader; a local function calling itself is all the recursion needed.

A recursive search, written from a description, in a sandbox with no loader to cheat with. It works. But the layer after it is the point: `verify_solution` re-checks the board without assuming the solver was right — every cell a digit, every row, column and box nine distinct values, every original given unchanged. The example runs twice, once on a legal puzzle and once on the same grid with a repeated 5 in the first row, so you can watch the check layer fire and the verify layer refuse the answer instead of passing it along.

## The point of the shape

Every part of this is arranged so the model's output is data, never authority. It arrives as a string in a JSON field. It is parsed as Lua and compiled. It is executed in a throwaway state to prove it defines the function it was asked for, and rejected at load if it does not. When it finally runs, it runs with four libraries, no loaders, no exit, a capped stack, a clock and a watchdog. No step trusts the previous one; each step narrows what the code is allowed to be.

The other half is that the model writes only the inside of one function. Ordering, data threading, types, naming, coercion, error messages — all of that is YAML and Go, written by a person, read once, and identical on every run. What varies is a handful of lines of Lua per layer, against a contract checked before the model saw it and checked again before the code was loaded. That is a small enough surface to read by hand, which is the only property that makes generated code reviewable at all.

A flow is the spec, and the spec says what happens and in what order. That is the whole shape of the program: a list of steps, the names that move between them, and nothing the engine has to infer. The model sits in the middle, writing the code part, and the code part is the only part it is allowed to write.

The repo is at [github.com/slashbase/layerengine](https://github.com/slashbase/layerengine), MIT licensed.

**PS.** If this looks interesting, try it: clone the repo, set a provider key, run the arithmetic example, then point a flow at something you actually care about. The codebase is small on purpose, so a first patch is an evening's work — a model ID in the registry, an example that shows what the current ones don't, or a seam that ought to be public and isn't. Issues and pull requests are welcome.
