# 에러 코드

rokc의 모든 진단은 `error[E200]: mismatched types: ...`처럼 코드로 시작합니다. 코드는 에러 종류마다 하나이므로 한 코드가 여러 메시지를 덮습니다. 무엇이 잘못되었는지, 대개는 대신 무엇을 써야 하는지도 메시지가 직접 알려 줍니다. 코드와 그 범위는 [9.9.4 진단](../spec/refs-lang/009-tools/909-debug-tools.html#994-진단--에러-메시지)이 정합니다. `--message-format=json`에서는 코드가 `code` 필드에 들어갑니다.

이 목록은 rokc 0.1.5 기준입니다. '내지 않음'으로 표시한 코드는 사양이 예약했지만 이 버전이 출력하지 않는 코드입니다.

## E0xx: 어휘와 구문

| 코드 | 종류 | 뜻 |
|------|------|---------|
| [E001](E001.html) | `lexical` | 텍스트를 토큰으로 나눌 수 없음(``unknown escape sequence `\%` ``) |
| [E002](E002.html) | `parse` | 토큰이 올바른 구문이 아님(``expected `)`, found `;` ``) |

## E1xx: 이름과 모듈

| 코드 | 종류 | 뜻 |
|------|------|---------|
| [E100](E100.html) | `resolve` | 이름이 스코프에 없음(``cannot find value `coun` in this scope``) |
| [E101](E101.html) | `visibility` | 내지 않음. 비공개 항목은 E102 |
| [E102](E102.html) | `module` | 비공개 항목, 잘못된 `use`, 없는 모듈 파일(``function `geo::area` is private to module `geo` ``) |

## E2xx: 타입

| 코드 | 종류 | 뜻 |
|------|------|---------|
| [E200](E200.html) | `type` | 프로그램이 타입 검사를 통과하지 못함(``mismatched types: argument `x` of `half` expects `i32`, found `f64` ``) |
| [E201](E201.html) | `pattern` | 내지 않음. 완전하지 않은 match는 E200 |
| [E202](E202.html) | `handoff` | 다른 소유자에게 넘긴 값을 그 뒤에 씀(`` `xs` was moved into the `Mutex` ``) |

## E3xx: 메타

| 코드 | 종류 | 뜻 |
|------|------|---------|
| [E300](E300.html) | `meta` | 속성, derive, 메타 블록을 잘못 씀(``unknown attribute `#[inlin]` ``) |

## E4xx: 빌드와 링크

| 코드 | 종류 | 뜻 |
|------|------|---------|
| [E400](E400.html) | `lowering` | 내지 않음 |
| [E401](E401.html) | `codegen` | 내지 않음. E500 참고 |
| [E402](E402.html) | `linkage` | LLVM 백엔드나 링커가 실패함(`the link failed:`) |

## E5xx: 기타

| 코드 | 종류 | 뜻 |
|------|------|---------|
| [E500](E500.html) | `unsupported` | 코드 생성이 아직 처리하지 않는 구문(`unsupported in codegen: ...`) |
| [E501](E501.html) | `internal` | 프로그램이 아니라 rokc의 버그 |
| [E502](E502.html) | `other` | 내지 않음 |
| [E503](E503.html) | `lint` | 린트가 걸림(``unused variable `unused` ``) |
