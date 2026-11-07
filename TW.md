# `tw` interop (fork notes)

This fork (`gbedford/templated-external`) lets Zena programs bind to JS
through a [ts-gen](https://github.com/wasm-bindgen/ts-gen) `tw` bridge. ts-gen
lowers TypeScript declarations to its IR, and generates both a Zena binding
and a Wasm bridge module from it. The bridge, not JS, provides the binding's
imports and calls its exports, keyed by IR spellings. The user-facing rules
are in `docs/language-reference.md` (Generic Imports, `tw` Imports,
Exports). This file says why they're shaped the way they are, for anyone
upstreaming them.

## The `tw` option

`@external(module, name, "tw")` marks an import a bridge provides. Its
modules are `tw:js/global`, `tw:js/<module>`, `tw:future`, `tw:loop` and
`tw:tag`. Since the provider is a Wasm module, its types cross as
themselves, with no JS-host shims, except:

- `String` crosses as its JS string handle (`externref`), the IR's `str`.
  This needs `-t js`.
- `Option<T>` crosses as the IR's `T?`, laid out per `T`:
  - a reference as itself or null;
  - a narrow scalar as its `i32` carrier with a niche;
  - any other scalar as a presence flag, then the value.
- Strings inside arrays and tuples are copied to and from their handles.

The declared function becomes a wrapper over the raw import when anything
needs converting.

`@export(name, "tw")` is the same contract in the other direction. The
bridge calls these exports, its callback trampolines (`tw:call:<fn type>`)
and entry points (`default.fetch(Request,Env,ExecutionContext)`), by IR key.

The option string takes `throws:<key>` as well, for example
`"tw, throws:err"`. An IR `throws E` error crosses as the Wasm tag
`("tw:tag", key)`. A class marked `@external("tw:tag", key, "tw")` is that
error in Zena.

## Templated names

The bridge is generated from the IR, so it finds each generic
instantiation by its import or export name alone.

- `{T}` in an `@external` or `@export` name of a generic function is the
  IR spelling of `T`'s type argument. Each instantiation that spells a
  distinct name is a distinct import or export.
- `{{` and `}}` are literal braces, which the IR uses for inline object
  types.
- A placeholder naming no type parameter is an error, as is an argument
  with no spelling. So is a Zena class argument, other than an extension
  class, whose values cross, since the bridge has no Zena structs.

`@spelling("<IR name>")` on a class or enum gives its IR name where that
isn't a Zena name (`WebAssembly.Module`, `"node:fs".Stats`).

## IR spellings

The spellings are ts-gen IR's canonical type spellings, with no spaces
(see ts-gen's `IR.md`, §3):

| Zena                                  | IR                                                   |
| ------------------------------------- | ---------------------------------------------------- |
| `i32` … `f64`, `i8`/`u8`/`i16`/`u16`  | the same name                                        |
| `boolean`                             | `bool`                                               |
| `void`                                | `unit`                                               |
| `String`                              | `str`                                                |
| `externref`                           | `Any`                                                |
| `T \| null`, `Option<T>`              | `T?`                                                 |
| `array<T>`                            | `arr<T>`, a JS Array. `T[]` is the IR's typed array. |
| `(A, B)`, `(A,)`                      | `(A,B)`, `(A,)`                                      |
| `(a: A) => R`, `(a: A) => void`       | `fn(A)->R`, `fn(A)`                                  |
| a class, distinct type or enum `N<A>` | `N<A>`, or its `@spelling`                           |

`unit` and `arr<T>` follow two IR renames, from `void` and from `T[]`. Keep
the table in step with ts-gen's IR when it changes.
