# Error codes

Every rokc diagnostic starts with a code: `error[E0040]: mismatched types: ...`. There is one code per kind of error, so one code covers many messages; the message itself says what is wrong and, usually, what to write instead. `rokc --explain E0040` (or `rok --explain E0040`) prints a short explanation of a code; its page here says the same and shows an example. With `--message-format=json` the code is the `code` field and its area the `kind` field ([9.9.4 Diagnostics](../spec/refs-lang/009-tools/909-debug-tools.html#994-diagnostics--error-messages)).

Codes are numbered in the order they were added, from E0001 up, and a number is never reused. This index describes rokc 0.2.0. Most pages show an example, checked against the real compiler output when the docs are built, and its fix.

## Lexical

| Code | Kind | Title |
|------|------|-------|
| [E0001](E0001.html) | `lexical` | unexpected character in source |
| [E0002](E0002.html) | `lexical` | bidirectional control character in source |
| [E0003](E0003.html) | `lexical` | malformed numeric literal |
| [E0004](E0004.html) | `lexical` | literal out of range for its type |
| [E0005](E0005.html) | `lexical` | unterminated literal or comment |
| [E0006](E0006.html) | `lexical` | invalid escape in a literal |
| [E0007](E0007.html) | `lexical` | a character literal holds one character |
| [E0008](E0008.html) | `lexical` | malformed f-string |
| [E0009](E0009.html) | `lexical` | name cannot be a raw identifier |

## Syntax

| Code | Kind | Title |
|------|------|-------|
| [E0010](E0010.html) | `parse` | unexpected token |
| [E0011](E0011.html) | `parse` | nesting too deep |
| [E0012](E0012.html) | `parse` | syntax from another language |
| [E0013](E0013.html) | `parse` | struct literal in a block head |
| [E0014](E0014.html) | `parse` | modifiers out of order |
| [E0015](E0015.html) | `parse` | doc comment documents nothing |
| [E0016](E0016.html) | `parse` | visibility where it has no meaning |
| [E0017](E0017.html) | `parse` | item in the wrong place |
| [E0018](E0018.html) | `parse` | `return` in top-level statements |
| [E0019](E0019.html) | `parse` | operators that need parentheses |
| [E0020](E0020.html) | `parse` | `return` in a trailing closure |
| [E0021](E0021.html) | `parse` | reserved word used as a name |
| [E0022](E0022.html) | `parse` | `let` without a value |
| [E0023](E0023.html) | `parse` | invalid function signature |
| [E0024](E0024.html) | `parse` | invalid generic parameters |
| [E0025](E0025.html) | `parse` | unsupported ABI |

## Names and modules

| Code | Kind | Title |
|------|------|-------|
| [E0026](E0026.html) | `resolve` | name defined more than once |
| [E0027](E0027.html) | `resolve` | cannot find value or function |
| [E0028](E0028.html) | `resolve` | cannot find type or trait |
| [E0029](E0029.html) | `resolve` | cannot find enum variant |
| [E0030](E0030.html) | `module` | unresolved import |
| [E0031](E0031.html) | `resolve` | ambiguous name |
| [E0032](E0032.html) | `module` | conflicting imports |
| [E0033](E0033.html) | `module` | invalid re-export |
| [E0034](E0034.html) | `visibility` | item is private |
| [E0035](E0035.html) | `module` | module file problem |
| [E0036](E0036.html) | `module` | invalid program entry point |
| [E0037](E0037.html) | `module` | invalid dependency |
| [E0038](E0038.html) | `resolve` | variable named like a nested function |
| [E0039](E0039.html) | `resolve` | nested item uses an outer local |
| [E0129](E0129.html) | `module` | invalid WIT |
| [E0136](E0136.html) | `resolve` | trait of the method not in scope |
| [E0142](E0142.html) | `resolve` | method generic parameter named like its impl's |

## Types

| Code | Kind | Title |
|------|------|-------|
| [E0040](E0040.html) | `type` | mismatched types |
| [E0041](E0041.html) | `type` | branches of different types |
| [E0042](E0042.html) | `type` | type annotations needed |
| [E0043](E0043.html) | `type` | no such field |
| [E0044](E0044.html) | `type` | missing field in a literal |
| [E0045](E0045.html) | `type` | field given more than once |
| [E0046](E0046.html) | `type` | no such method or associated item |
| [E0047](E0047.html) | `type` | item used in the wrong form |
| [E0048](E0048.html) | `type` | wrong number of arguments |
| [E0049](E0049.html) | `type` | invalid named argument |
| [E0050](E0050.html) | `type` | `ref` parameter and argument mismatch |
| [E0051](E0051.html) | `type` | invalid spread argument |
| [E0052](E0052.html) | `type` | value is not callable |
| [E0053](E0053.html) | `type` | invalid cast |
| [E0054](E0054.html) | `type` | operator cannot be applied |
| [E0055](E0055.html) | `type` | literal fits several types |
| [E0056](E0056.html) | `type` | invalid indexing |
| [E0057](E0057.html) | `type` | invalid assignment |
| [E0058](E0058.html) | `type` | invalid closure capture |
| [E0059](E0059.html) | `type` | keyword outside its context |
| [E0060](E0060.html) | `type` | `let .. else` that does not diverge |
| [E0061](E0061.html) | `type` | operation needs `unsafe` |
| [E0062](E0062.html) | `type` | error not handled |
| [E0063](E0063.html) | `type` | thrown value is not an `Error` |
| [E0064](E0064.html) | `type` | optional operator on a value that is not optional |
| [E0065](E0065.html) | `type` | lock guard not released by `using` |
| [E0066](E0066.html) | `type` | `using` on a value without `Drop` |
| [E0067](E0067.html) | `type` | block statement leaves a value |
| [E0068](E0068.html) | `type` | value cannot be iterated |
| [E0069](E0069.html) | `type` | trait bound not satisfied |
| [E0070](E0070.html) | `type` | not a trait |
| [E0071](E0071.html) | `type` | trait used as a type |
| [E0072](E0072.html) | `type` | impl is missing a trait item |
| [E0073](E0073.html) | `type` | impl item is not in the trait |
| [E0074](E0074.html) | `type` | impl item does not match the trait |
| [E0075](E0075.html) | `type` | conflicting implementations |
| [E0076](E0076.html) | `type` | impl of a foreign trait for a foreign type |
| [E0077](E0077.html) | `type` | supertrait not implemented |
| [E0078](E0078.html) | `type` | impl not allowed |
| [E0079](E0079.html) | `type` | cannot evaluate at compile time |
| [E0080](E0080.html) | `type` | definition depends on itself |
| [E0081](E0081.html) | `type` | invalid const generic argument |
| [E0082](E0082.html) | `type` | wrong generic arguments |
| [E0083](E0083.html) | `type` | invalid enum value |
| [E0084](E0084.html) | `type` | invalid foreign-function use |
| [E0085](E0085.html) | `type` | value cannot cross threads |
| [E0086](E0086.html) | `type` | invalid union |
| [E0087](E0087.html) | `type` | ambiguous associated item |
| [E0088](E0088.html) | `type` | invalid format |
| [E0089](E0089.html) | `type` | invalid compiler built-in call |
| [E0090](E0090.html) | `type` | invalid coroutine use |
| [E0091](E0091.html) | `type` | invalid broadcast |
| [E0092](E0092.html) | `type` | `Drop::drop` called directly |
| [E0093](E0093.html) | `type` | generic instantiation too deep |
| [E0094](E0094.html) | `type` | invalid representation |
| [E0095](E0095.html) | `type` | type too large |
| [E0096](E0096.html) | `type` | `try` or `catch` that does nothing |
| [E0097](E0097.html) | `type` | ROKON value of the wrong shape |
| [E0139](E0139.html) | `type` | `pub` function signature not written out |
| [E0141](E0141.html) | `type` | assignment to a value that is not a place |

## Patterns

| Code | Kind | Title |
|------|------|-------|
| [E0098](E0098.html) | `pattern` | non-exhaustive match |
| [E0099](E0099.html) | `pattern` | unreachable match arm |
| [E0100](E0100.html) | `pattern` | pattern does not fit the value's type |
| [E0101](E0101.html) | `pattern` | wrong number of fields in a pattern |
| [E0102](E0102.html) | `pattern` | or-pattern alternatives bind differently |
| [E0103](E0103.html) | `pattern` | binding looks like a variant |
| [E0104](E0104.html) | `pattern` | invalid rest pattern |

## Hand-off and resources

| Code | Kind | Title |
|------|------|-------|
| [E0105](E0105.html) | `handoff` | value used after it was handed over |
| [E0106](E0106.html) | `handoff` | value used after its lock was released |
| [E0107](E0107.html) | `handoff` | shared value handed over |
| [E0134](E0134.html) | `handoff` | resource used after it was moved or closed |
| [E0135](E0135.html) | `handoff` | resource not closed |
| [E0137](E0137.html) | `handoff` | thread-local value kept past `with` |
| [E0138](E0138.html) | `handoff` | list view used after a resize |
| [E0140](E0140.html) | `handoff` | stream view used past its stream |

## Attributes and meta

| Code | Kind | Title |
|------|------|-------|
| [E0108](E0108.html) | `meta` | cannot derive |
| [E0109](E0109.html) | `meta` | malformed attribute |
| [E0110](E0110.html) | `meta` | attribute on the wrong item |
| [E0111](E0111.html) | `meta` | invalid `#[lang]` item |
| [E0112](E0112.html) | `meta` | invalid operator declaration |
| [E0113](E0113.html) | `meta` | invalid `#[keyword]` declaration |
| [E0114](E0114.html) | `meta` | invalid literal handler |
| [E0115](E0115.html) | `meta` | invalid `#[intrinsic]` |
| [E0116](E0116.html) | `meta` | malformed macro definition |
| [E0117](E0117.html) | `meta` | no macro rule matches |
| [E0118](E0118.html) | `meta` | invalid meta template |
| [E0119](E0119.html) | `meta` | invalid meta block |
| [E0120](E0120.html) | `meta` | recursive expansion |
| [E0121](E0121.html) | `meta` | invalid `cfg` predicate |
| [E0122](E0122.html) | `meta` | file include failed |
| [E0123](E0123.html) | `meta` | registered twice |
| [E0124](E0124.html) | `meta` | meta name cannot be resolved |

## Build and back end

| Code | Kind | Title |
|------|------|-------|
| [E0125](E0125.html) | `other` | cannot read or write a file |
| [E0126](E0126.html) | `linkage` | link failed |
| [E0127](E0127.html) | `codegen` | code generation failed |
| [E0128](E0128.html) | `linkage` | external tool failed |
| [E0130](E0130.html) | `unsupported` | not supported by this backend |
| [E0131](E0131.html) | `internal` | internal compiler error |
| [E0133](E0133.html) | `other` | invalid compiler option |

## Lints

| Code | Kind | Title |
|------|------|-------|
| [E0132](E0132.html) | `lint` | lint |
