# Compare Strings by Codepoint

A TC39 proposal for a new method to compare strings by Unicode code points.

## Status

[The TC39 Process](https://tc39.es/process-document/)

**Stage**: 0

**Champions**:
- Mathieu Hofman (@mhofman)
- Mark S. Miller (@erights)

## Motivation

### Background on strings in JavaScript

JavaScript exposes strings as a sequence of 16-bits / UCS-2 characters. For well-formed strings, these characters are UTF-16 code units. UTF-16 uses Surrogate Pairs for any Unicode codepoints outside the Basic Multilingual Plane.

That means that a Unicode character with a code point in the [0x010000 - 0x10FFFF] range will be represented as 2 code units, the first leading surrogate in the [0xD800 - 0xDBFF] range, and the second trailing surrogate in the [0xDC00 - 0xDFFF] range.

For further reading, see @mathiasbynens's post on [JavaScript’s internal character encoding](https://mathiasbynens.be/notes/javascript-encoding).

### Effect of string encoding on JavaScript programs

This encoding choice is observed by JavaScript programs in the following main cases:
- Indexed access to a strings. This extends to all String APIs involving offsets or length
- Matching string using RegExp without the `u` or `v` flag
- Comparing strings

Unlike indexed access, iterators on strings do operate on codepoints. Similarly the `u` and `v` RegExp flags enables matching whole Unicode codepoints instead of code units / surrogate halves. However no String API exists that allows comparing strings by code point.

Because JavaScript compares strings by their 16-bits code units, any codepoint in the range [0xE0000 - 0xFFFF] will sort after a leading surrogate used to encode the first half of a codepoint in the [0x010000 - 0x10FFFF] range.

### Interoperability with other systems

This comparison behavior puts JavaScript at odds with other languages or systems that end up comparing strings by their Unicode codepoint. That includes any language or system that uses UTF-8 as their string encoding and relies on bytes comparison (UTF-8 does preserve sort order).

Example of languages using UTF-8 encoding are Swift and Golang. SQLite by default encodes strings using UTF-8 as well. Because of this, the sort order of strings in these systems may not match the sort order of the same strings in JavaScript.

## Proposal

A `String.codePointCompare(a, b)` (actual name TBD) that can be used to compare 2 strings by their code point. The function could be used with `Array.prototype.sort`.

## Alternatives considered

### Manual iteration

It's possible to write a comparator using a manual iteration of the strings, and is the status-quo today. This can be implemented by either manually advancing String iterators in lock-step, or by using indexed access and retrieving the code units.

### A "locale-less" collation

There are already alternative comparators in the language, e.g. `String.prototype.localeCompare` or `Intl.Collator.prototype.compare`. While these function on codepoints, they take into consideration the locale and collapse characters in the same equivalence class.

We could imagine a collation that does not perform any locale specific logic, and simply enables comparing strings by their Unicode code point.

## Q&A

### What should the comparator return if it encounters malformed strings?

An unmatched surrogate in a string could fallback to comparing using its code unit. There may be alternative behaviors that could be considered.

### What about comparison operators like < or >

Because changing their behavior would be a breaking change, they would continue comparing strings lexicographically by code unit. However a program could instead compare the sign of the result of calling the proposed codepoint comparator function with the 2 strings.

## See also

https://github.com/endojs/endo/pull/2008

https://github.com/Agoric/agoric-sdk/issues/10335

https://github.com/Agoric/agoric-sdk/pull/10299
