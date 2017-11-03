# PureScript Pattern Matching in JS

Tim Humphries recently [asked on Twitter](https://twitter.com/thumphriees/status/925526349393027072):

> Javascript users: what's the best way to encode a sum type / tagged union? Either for library ergonomics or for pattern-matching performance

> I have a language with sum types that I currently compile to Purescript, and would also like to compile directly to JS

The following describes something I've used in the past in order to make PureScript sum types easier to deal with from JavaScript.

All credit for this goes to Phil Freeman ([@paf31](https://github.com/paf31)). He and I used this approach when we worked together at [Awake Security](https://awakesecurity.com/).

## An Example

First off, we need a sum type. For the purposes of this example I'm going to use [`Maybe`](https://pursuit.purescript.org/packages/purescript-maybe/3.0.0/docs/Data.Maybe#t:Maybe).


### Pattern Matching

In PureScript, we can pattern match on its constructors using a `case` expression, like so:

```purescript
case _ of
  Just x -> ...
  Nothing -> ...
```

This compiles to the following JavaScript:

```javascript
if (v instanceof Data_Maybe.Just) {
    ...
};
if (v instanceof Data_Maybe.Nothing) {
    ...
};
```

While it's technically possible to do the same thing by hand, it's definitely cumbersome.


### An Alternative Approach

In addition to pattern matching using `case` expressions, in PureScript we could also use the [`maybe'`](https://pursuit.purescript.org/packages/purescript-maybe/3.0.0/docs/Data.Maybe#v:maybe') function.

`maybe'` takes two functions and a `Maybe` value. If the `Maybe` value is `Nothing`, the first function is called with `unit` and its return value is used. If the `Maybe` value is `Just x`, the second function is called with `x` and its return value is used.

```purescript
maybe' :: forall a b. (Unit -> b) -> (a -> b) -> Maybe a -> b
```

We can use this as inspiration to provide a more idiomatic API for pattern matching against constructors of PureScript sum types in JavaScript.


### A More Idiomatic API

In PureScript, we can write the following function and expose it to JavaScript:

```purescript
module Data.Maybe.Interop where

maybe
  :: forall a b
   . { "Nothing" :: Unit -> b, "Just" :: a -> b }
  -> Maybe a
  -> b
maybe cases = maybe' cases."Nothing" cases."Just"
```

For our custom sum types, we may not have defined a function equivalent to `maybe'`, so here it is again using a `case` expression:

```purescript
module Data.Maybe.Interop where

maybe
  :: forall a b
   . { "Nothing" :: Unit -> b, "Just" :: a -> b }
  -> Maybe a
  -> b
maybe cases = case _ of
  Nothing -> cases."Nothing" unit
  Just x -> cases."Just" x
```

Now, we can use the function in JavaScript:

```javascript
import { maybe } from 'Data/Maybe/Interop.purs';

// Takes a `Maybe` value and returns `null` if the `Maybe` value is `Nothing`, or
// returns the value inside of `Just`.
const getMaybeValue = maybe({
  Nothing: () => null,
  Just: (x) => x,
});
```

We can choose to ignore the values passed to our functions if we like. Here's how we might reimplement [`isJust`](https://pursuit.purescript.org/packages/purescript-maybe/3.0.0/docs/Data.Maybe#v:isJust).

```javascript
import { maybe } from 'Data/Maybe/Interop.purs';

// Takes a `Maybe` value and returns `true` when it was constructed with `Just`.
const isJust = maybe({
  Nothing: () => false,
  Just: (_) => true,
});
```

We can even perform effects:

```javascript
import { maybe } from 'Data/Maybe/Interop.purs';

// Takes a `Maybe` value and logs a message to the console.
const logMaybe = maybe({
  Nothing: () => console.log("Nothing to see here."),
  Just: (x) => console.log(`Just: ${x}`),
});
```
