# Compare Strings by Codepoint

A TC39 proposal for a new method to compare strings by Unicode codepoints.

## Status

[The TC39 Process](https://tc39.es/process-document/)

**Stage**: 1

**Champions**:
- Mathieu Hofman (@mhofman)
- Mark S. Miller (@erights)
- Christopher Hiller (@boneskull)

## Motivation

### Background on strings in JavaScript

JavaScript exposes strings as a sequence of 16-bits / UCS-2 characters. For well-formed strings, these characters are UTF-16 code units. UTF-16 uses Surrogate Pairs for any Unicode codepoints outside the Basic Multilingual Plane.

That means that a Unicode character with a codepoint in the [0x010000 - 0x10FFFF] range will be represented as 2 code units, the first leading surrogate in the [0xD800 - 0xDBFF] range, and the second trailing surrogate in the [0xDC00 - 0xDFFF] range.

For further reading, see @mathiasbynens's post on [JavaScript’s internal character encoding](https://mathiasbynens.be/notes/javascript-encoding).

### Effect of string encoding on JavaScript programs

This encoding choice is observed by JavaScript programs in the following main cases:
- Indexed access to a strings. This extends to all String APIs involving offsets or length
- Matching string using RegExp without the `u` or `v` flag
- Comparing strings

Unlike indexed access, iterators on strings do operate on codepoints. Similarly the `u` and `v` RegExp flags enables matching whole Unicode codepoints instead of code units / surrogate halves. However no String API exists that allows comparing strings by codepoints.

Because JavaScript compares strings by their 16-bits code units, any codepoint in the range [0xE000 - 0xFFFF] will sort after a leading surrogate used to encode the first half of a codepoint in the [0x010000 - 0x10FFFF] range.

### Interoperability with other systems

This comparison behavior puts JavaScript at odds with other languages or systems that end up comparing strings by their Unicode codepoints. That includes any language or system that uses UTF-8 as their string encoding and relies on bytes comparison (UTF-8 does preserve sort order).

Example of languages using UTF-8 encoding are Swift and Golang. SQLite by default encodes strings using UTF-8 as well. Because of this, the sort order of strings in these systems may not match the sort order of the same strings in JavaScript.

### Locale independent compare

Data is not solely presented to human users. Some data processing involves sorting, and needs to be deterministic and portable. A comparison by Unicode codepoints satisfies those requirements where a locale based comparison does not.

## Proposal

A `String.codePointCompare(a, b)` (actual name TBD) that can be used to compare 2 strings by their codepoints. The function can be used as an argument for `Array.prototype.sort`.

### Example

```js
const arr = [
  '\u{ff42}', // Fullwidth Latin Small Letter B
  '\u{1d5ba}', // Mathematical Sans-Serif Small A
  '\u{63}', // Latin Small Letter C
];

console.log('native compare', [...arr].sort()); // [ 'c', '𝖺', 'ｂ' ]
console.log('locale compare', [...arr].sort((a, b) => a.localeCompare(b))); // [ '𝖺', 'ｂ', 'c' ]
console.log('null locale compare', [...arr].sort(new Intl.Collator('zxx').compare)); // [ '𝖺', 'ｂ', 'c' ]
console.log('codepoint compare', [...arr].sort(String.codePointCompare)); // [ 'c', 'ｂ', '𝖺' ]
```

## Alternatives considered

### Manual iteration

It's possible to write a comparator using a manual iteration of the strings, and is the status-quo today. This can be implemented by either manually advancing String iterators in lock-step, or by using indexed access and retrieving the code units.

<details>
  <summary>codePointCompare shim</summary>

```js
function codePointCompare(left, right) {
  const leftIter = left[Symbol.iterator]();
  const rightIter = right[Symbol.iterator]();
  for (;;) {
    const { value: leftChar } = leftIter.next();
    const { value: rightChar } = rightIter.next();
    if (leftChar === undefined && rightChar === undefined) {
      return 0;
    } else if (leftChar === undefined) {
      // left is a prefix of right.
      return -1;
    } else if (rightChar === undefined) {
      // right is a prefix of left.
      return 1;
    }
    const leftCodepoint = leftChar.codePointAt(0);
    const rightCodepoint = rightChar.codePointAt(0);
    if (leftCodepoint < rightCodepoint) return -1;
    if (leftCodepoint > rightCodepoint) return 1;
  }
};
```
</details>

### Null locale collation

There are already alternative comparators in the language, e.g. `String.prototype.localeCompare` or `Intl.Collator.prototype.compare`. While these operate on codepoints, they take into consideration the locale and collapse characters in the same equivalence class. This is also the case for the `zxx` "locale" in the [Stable Formatting proposal](https://github.com/tc39/proposal-stable-formatting).

While we could imagine a collation option for a null locale that does not perform any equivalence class or grapheme logic, and simply enables comparing strings by their Unicode codepoints, this seems to [go counter](https://github.com/tc39/proposal-stable-formatting/issues/13) to the core of Intl.

Furthermore `Intl` is a normative optional part of the spec and this would require JS engines that opt-out to implement enough of `Intl` just to offer a comparison that is agnostic of Internationalization concerns.

## Q&A

### What should the comparator return if it encounters malformed strings?

An unmatched surrogate in a string could fallback to comparing using its code unit. There may be alternative behaviors that could be considered.

### What about comparison operators like `<` or `>`

Because changing their behavior would be a breaking change, they would continue comparing strings lexicographically by code unit. However a program could instead compare the sign of the result of calling the proposed codepoint comparator function with the 2 strings.

### Should we change the default `Array.prototype.sort` comparison

Like for operators, this would be a breaking change. Anyone interested in portability can fairly easily provide the comparator function.

### Can you provide an example of system where a portable comparison is needed?

The Agoric platform implements collections that use a well defined sort order for keys instead of using insertion order. These collections can be either "heap-only", or backed by a SQLite DB. To provide a consistent iteration order independent of the backing store, the heap implementation needs to use a comparator compatible with the sort order implemented by SQLite.

## See also

https://github.com/endojs/endo/pull/2008

https://github.com/Agoric/agoric-sdk/issues/10335

https://github.com/Agoric/agoric-sdk/pull/10299

https://es.discourse.group/t/builtin-ord-compare-method-for-primitives/724/25

