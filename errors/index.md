# Error codes

Every rokc diagnostic starts with a code: `error[E200]: mismatched types: ...`. There is one code per kind of error, so one code covers many messages; the message itself says what is wrong and, usually, what to write instead. The codes and their ranges are defined in [9.9.4 Diagnostics](../spec/refs-lang/009-tools/909-debug-tools.html#994-diagnostics--error-messages). With `--message-format=json` the code is the `code` field.

This index describes rokc 0.1.5. Codes marked *not emitted* are reserved by the spec but never printed by this version.

## E0xx: lexical and syntax

| Code | Kind | Meaning |
|------|------|---------|
| [E001](E001.html) | `lexical` | The text cannot be split into tokens (``unknown escape sequence `\%` ``) |
| [E002](E002.html) | `parse` | The tokens are not valid syntax (``expected `)`, found `;` ``) |

## E1xx: names and modules

| Code | Kind | Meaning |
|------|------|---------|
| [E100](E100.html) | `resolve` | A name is not in scope (``cannot find value `coun` in this scope``) |
| [E101](E101.html) | `visibility` | Not emitted; private items are E102 |
| [E102](E102.html) | `module` | A private item, a bad `use`, or a missing module file (``function `geo::area` is private to module `geo` ``) |

## E2xx: types

| Code | Kind | Meaning |
|------|------|---------|
| [E200](E200.html) | `type` | The program does not type-check (``mismatched types: argument `x` of `half` expects `i32`, found `f64` ``) |
| [E201](E201.html) | `pattern` | Not emitted; non-exhaustive matches are E200 |
| [E202](E202.html) | `handoff` | A value is used after it was handed to another owner (`` `xs` was moved into the `Mutex` ``) |

## E3xx: meta

| Code | Kind | Meaning |
|------|------|---------|
| [E300](E300.html) | `meta` | An attribute, derive or meta block is used wrongly (``unknown attribute `#[inlin]` ``) |

## E4xx: build and link

| Code | Kind | Meaning |
|------|------|---------|
| [E400](E400.html) | `lowering` | Not emitted |
| [E401](E401.html) | `codegen` | Not emitted; see E500 |
| [E402](E402.html) | `linkage` | The LLVM back end or the linker failed (`the link failed:`) |

## E5xx: other

| Code | Kind | Meaning |
|------|------|---------|
| [E500](E500.html) | `unsupported` | Code generation does not handle a construct yet (`unsupported in codegen: ...`) |
| [E501](E501.html) | `internal` | A bug in rokc, never in the program |
| [E502](E502.html) | `other` | Not emitted |
| [E503](E503.html) | `lint` | A lint fired (``unused variable `unused` ``) |
