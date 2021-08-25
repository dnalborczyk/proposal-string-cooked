# String.cooked proposal

**Champions:**

- _none_

**Authors:**

- Darien Maillet Valentine

**Stage:** -1 (initial idea, no champion)

## Overview

This proposes a new static String method like `String.raw` but which
concatenates the “cooked” (escaped string value) strings rather than the raw
strings — the same behavior as that of an untagged template literal.

```js
String.raw`consuming \u0072aw or undercooked strings may increase your risk of stringborne illness`;
// → "consuming \u0072aw or undercooked strings may increase your risk of stringborne illness"

String.cooked`mmm ... \u0064elicious cooked string`;
// → "mmm ... delicious cooked string"
```

## Motivation

Many template tags are interested in some kind of preprocessing of either the
“cooked” string values, the substitution values, or both — but ultimately they
may still want to perform the “default” concatenation behavior after this
processing.

This can be achieved today in at least two ways:

1. Implementing the “zip-like” concatenation behavior for the string values and
   substitution values “manually”.
2. Delegating to `String.raw`.

The latter is very attractive, but in a way that makes it a potential pit of
failure. It’s the only exposed way to get something that looks like the
“default” behavior but it’s not super obvious that it’s not since, for most
input strings people are likely to test, it would appear as though it is!

In order to access the “right” behavior for most use cases, delegating to `raw`
is possible, but you have to pass the cooked strings _as if_ they were raw
strings, i.e. `String.raw({ raw: strs }, ...subs)`, not
`String.raw(strs, ...subs)`. Though this works, the indirection is confusing;
there aren’t any real raw string values in play here.

> It may also be tempting for folks to use `String.raw` as if it were a true
> identity function for other reasons, as is shown in
> [this Twitter post](https://twitter.com/wcbytes/status/1430271001632415745).
> Again, it’s understandable why folks might see the _one_ built-in tag and
> think this is what they’re looking for, but with `cooked` present as well,
> the distinction being made becomes more apparent.

## Use cases

The primary use case is to serve as a final step in custom template tags which
perform some kind of mapping over input. For example, consider a tag which is
meant to escape URL path segments in such a way that they round trip (i.e., the
interpolated `/` characters get escaped as `%2F`):

```js
function safePath(strings, ...subs) {
  return String.cooked(strings, ...subs.map(sub => encodeURIComponent(sub)));
}
```

In other words, although it has the signature of a template tag function, it is
mainly expected to facilitate building other template tags without needing to
reimplement the usual string/substitution concatenation logic.

As a tag in its own right, it acts like the template tag equivalent of the
identity function, which may also help with usage patterns like the example
linked to earlier where the user wished to use template tags to provide a signal
to their editor that the content should be interpreted as HTML. Some compilation
or preprocessing tools may also benefit from that, e.g. Prettier singles out the
tags with the binding “html” for different string transformation.

## Q&A

**Q:** Should the name be “cooked”?

**A:** Not sure! This is an initial proposal. Feedback about whether this name
is intuitive and clear would be helpful. The term does have a history of usage
in discussion contexts (ES Discuss, etc) as the counterpart for “raw,” but it
has not appeared in any spec text or API surface to date as far as I know.

**Q:** What is the behavior if `undefined` is encountered when reading
properties from the first argument object?

**A:** The tentatively proposed behavior is that if `undefined` is returned when
reading one of the index-keyed properties, a `TypeError` is thrown. For any
other value type, ordinary `ToString` conversion is attempted.

This is because (assuming the first argument is a “real” template array object)
`undefined` appearing indicates that the raw segment source contained
[NotEscapeSequence](https://tc39.es/ecma262/#prod-NotEscapeSequence), i.e.
it is a template which has a raw value but _does not have_ a cooked value. If
such a template literal is _untagged,_ a `SyntaxError` would be thrown (though
that would likely not be appropriate for an evaluation-time API, hence use of
`TypeError` instead).