# Word count

Count the lines, words and characters of a text, then use a `Dict` and sorting to pick the five most frequent words.

## Code

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

## Walkthrough

- Iterators on `str` do the counting. `lines()` yields lines, `split_whitespace()` yields words, and `count()` counts them. The character count is `char_count()`, not the byte length `len()` ([Strings](../learn/09-strings.html), [`str` methods](../std/core/str.html#impl.str)).
- `normalize` builds the word in a `StrBuf` one character at a time, dropping punctuation and lowercasing, so `"Dog,"` and `"dog."` count as the same word ([`StrBuf`](../std/core/str.html#struct.StrBuf)).
- The frequency table is a `Dict<str, i32>`. `counts.get(w)` returns `none` for a missing key, so `?? 0` supplies the default before `counts[w] = ...` writes the new count ([Collections and iterators](../learn/08-collections-iterators.html), [`Dict`](../std/core/collections/dict.html#struct.Dict)).
- `Dict` iteration order is unspecified, so the pairs are copied into a list of `(word, count)` tuples and sorted with `sort_by`. Ties are broken alphabetically, so the output is always the same ([`List::sort_by`](../std/core/collections/list.html#List.sort_by)).
- The table is aligned with f-string format specs: `{w:<6}` left-aligns, `{c:>2}` right-aligns ([2.10.8 align and width](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)).
