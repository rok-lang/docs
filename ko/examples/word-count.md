# 단어 세기

텍스트의 줄, 단어, 문자 수를 세고 `Dict`와 정렬로 가장 자주 나오는 단어 다섯 개를 고릅니다.

## 코드

```rok
// Count lines, words and chars, then list the most frequent words.

const TEXT: str = "The quick brown fox jumps over the lazy dog.
The dog sleeps; the fox does not.
A fox, a dog, and the long summer day.

Quick thinking wins the day!";

struct Stats {
    lines: usize;
    words: usize;
    chars: usize;
}

fn stats(text: str) -> Stats {
    let s = Stats { lines: 0, words: 0, chars: text.char_count() };
    for line in text.lines() {
        s.lines += 1;
        s.words += line.split_whitespace().count();
    }
    s
}

// Lowercase a token and drop surrounding punctuation ("Dog," -> "dog").
fn normalize(token: str) -> str {
    let sb = StrBuf::new();
    for c in token.chars() {
        if c.is_alphanumeric() || c == '\'' {
            sb.push(c.to_lowercase());
        }
    }
    sb.to_str()
}

fn frequencies(text: str) -> Dict<str, i32> {
    let counts: Dict<str, i32> = Dict::new();
    for token in text.split_whitespace() {
        let w = normalize(token);
        if w.is_empty() { continue; }
        counts[w] = (counts.get(w) ?? 0) + 1;
    }
    counts
}

fn top(counts: Dict<str, i32>, n: usize) -> List<(str, i32)> {
    let pairs: List<(str, i32)> = [];
    for (w, c) in counts.entries() {
        pairs.push((w, c));
    }
    // Most frequent first; ties broken alphabetically so output is stable.
    pairs.sort_by(|a: (str, i32), b: (str, i32)| {
        if a.1 != b.1 { b.1.cmp(a.1) } else { a.0.cmp(b.0) }
    });
    let out: List<(str, i32)> = [];
    for p in pairs.iter().take(n) { out.push(p); }
    out
}

fn main() {
    let s = stats(TEXT);
    println(f"lines {s.lines:>4}");
    println(f"words {s.words:>4}");
    println(f"chars {s.chars:>4}");

    let counts = frequencies(TEXT);
    println(f"\n{counts.len()} distinct words, top 5:");
    let rank = 1;
    for (w, c) in top(counts, 5) {
        let bar = "#".repeat(c as usize);
        println(f"{rank}. {w:<6} {c:>2} {bar}");
        rank += 1;
    }
}
```

```output
lines    5
words   30
chars  147

18 distinct words, top 5:
1. the     6 ######
2. dog     3 ###
3. fox     3 ###
4. a       2 ##
5. day     2 ##
```

## 설명

- 세는 일은 `str`의 반복자가 맡습니다. `lines()`는 줄을, `split_whitespace()`는 단어를 내고, `count()`가 그 수를 셉니다. 문자 수는 바이트 길이 `len()`이 아니라 `char_count()`입니다([문자열](../learn/09-strings.html), [`str` 메서드](../../std/core/str.html#impl.str)).
- `normalize`는 구두점을 버리고 소문자로 바꾸면서 `StrBuf`에 한 글자씩 단어를 만듭니다. 그래서 `"Dog,"`와 `"dog."`는 같은 단어로 셉니다([`StrBuf`](../../std/core/str.html#struct.StrBuf)).
- 빈도 표는 `Dict<str, i32>`입니다. `counts.get(w)`는 없는 키면 `none`을 돌려주므로 `counts[w] = ...`가 새 개수를 쓰기 전에 `?? 0`이 기본값을 줍니다([컬렉션과 반복자](../learn/08-collections-iterators.html), [`Dict`](../../std/core/collections/dict.html#struct.Dict)).
- 개수로 정렬하려면 리스트가 필요하므로 쌍을 `(word, count)` 튜플의 리스트로 옮겨 `sort_by`로 정렬합니다. 개수가 같으면 알파벳순으로 정하므로 출력은 언제나 같습니다([`List::sort_by`](../../std/core/collections/list.html#List.sort_by)).
- 표는 f-string 포맷 지정자로 정렬합니다. `{w:<6}`은 왼쪽 정렬, `{c:>2}`는 오른쪽 정렬입니다([2.10.8 정렬과 폭](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)).
