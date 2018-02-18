---
title: "Pair"
description: "Canonical Product Type"
layout: "guide"
functions: ["branch", "fst", "merge", "snd", "topairs", "writertopair"]
weight: 100
---

```haskell
Pair a b
```

`Pair` allows the ability to represent (2) distinct values of different types.
Much like how `Either` is known as canonical Sum Type and defines the basis for
all other Sum Types that ship with `crocks`, `Pair` is known as the canonical
Product Type and also at the heart of all Product Types in `crocks`.

As `Pair` is a `Bifunctor`, it can vary in each of the (2) types it represents.
When used as a normal `Functor`, `Pair` will always have a bias for the far
right or second value, matching the pattern of the other ADTs in `crocks`. When
mapped with a function, the function will only be applied to the second value,
and will leave the first value untouched.

`Pair` also provides the ability to use `ap` and `chain`, but in order to
combine the resulting instances in a predictable, repeatable fashion the first
values in the `Pair`s must be `Semigroup` instances of the same type. When
applied, `ap` and `chain` will concatenate the `Semigroup`s providing the result
of the concatenation in the first position of the resulting `Pair`.

A helpful benefit of the `Bifunctor` aspects `Pair` allows for defining parallel
computations. There are many functions that ship with `crocks` that allow for
parallelization such as [`branch`](#branch), [`merge`](#merge-pointfree) and
`fanout`. Using those helpers in conjunction with the ability to
[`bimap`](#bimap) functions over a given `Pair`s values.

```javascript
import Pair from 'crocks/Pair'
import Sum from 'crocks/Sum'
import bimap from 'crocks/pointfree/bimap'
import branch from 'crocks/Pair/branch'
import compose from 'crocks/helpers/compose'
import mreduce from 'crocks/helpers/mreduce'
import merge from 'crocks/Pair/merge'

// negate :: a -> Boolean
const negate =
  x => !x

// inc :: Number -> Number
const inc =
  x => x + 1

// length :: Array -> Number
const length =
  x => x.length

// divide :: Number -> Number
const divide =
  (x, y) => x / y

Pair(76, false)
  .bimap(inc, negate)
//=> Pair(77, true)

// average :: [ Number ] -> Number
const average = compose(
  merge(divide),
  bimap(mreduce(Sum), length),
  branch
)

average([ 9, 77, 34 ])
//=> 40
```

<article id="topic-implements">

## Implements
`Setoid`, `Semigroup`, `Functor`,  `Bifunctor`, `Apply`, `Chain`,
`Traversable`, `Extend`

</article>

<article id="topic-instance">

## Instance Methods

#### equals

```haskell
Pair a b ~> c -> Boolean
```

Used to compare the underlying values of (2) `Pair` instances for equality by
value. `equals` takes any given argument and returns `true` if the passed
arguments is a `Pair` with an underlying values both in the first and second are
equal to the underlying values in the first and second  of the `Maybe` the
method is being called on. If the passed argument is not a `Maybe` or the
underlying values are not equal, `equals` will return `false`.

```javascript
import Pair from 'crocks/Pair'
import equals from 'crocks/pointfree/equals'

Pair({ num: 33 }, 'string')
  .equals(Pair({ num: 33 }, 'string'))
//=> true

Pair({ num: 33 }, 'string')
  .equals(Pair({ num: 10 }, 'string'))
//=> false

Pair({ num: 33 }, 'string')
  .equals(Pair({ num: 33 }, 'different'))
//=> false

equals(Pair([ 1, 2 ], ''), [ 1, 2 ])
//=> false
```

#### concat

```haskell
Semigroup s, t => Pair s t ~> Pair s t -> Pair s t
```

When both underlying values of a given `Pair` are fixed to a `Semigroup`,
`concat` can be used to concatenate another `Pair` instance with underlying
`Semigroup`s of the same type and structure. Expecting a `Maybe` wrapping a
`Semigroup` of the same type, `concat` will give back a new `Pair` instance
wrapping the result of combining the underlying `Semigroup` instances.

```javascript
import Pair from 'crocks/Pair'
import Sum from 'crocks/Sum'
import Maybe from 'crocks/Maybe'

import compose from 'crocks/helpers/compose'
import concat from 'crocks/pointfree/concat'
import fanout from 'crocks/helpers/fanout'
import flip from 'crocks/combinators/flip'
import map from 'crocks/pointfree/map'
import mapReduce from 'crocks/helpers/mapReduce'
import option from 'crocks/pointfree/option'
import prop from 'crocks/Maybe/prop'

Pair(Sum(3), [ 3 ])
  .concat(Pair(Sum(10), [ 10 ]))
//=> Pair( Sum 13, [ 3, 10 ] )

// Person :: { name: String, age: Number }
// peeps :: [ Person ]
const peeps = [
  { name: 'Haskell', age: 82 },
  { name: 'Heinrich', age: 81 },
  { name: 'Maria', age: 93 }
]

// mapProp :: (String, (a -> b)) -> Object -> Maybe b
const mapProp = (key, fn) =>
  compose(map(fn), prop(key))

// Combined :: Pair (Maybe [ String ]) (Maybe Sum)
// splitPerson :: Person -> Combined
const splitPerson = fanout(
  mapProp('name', x => [ x ]),
  mapProp('age', Sum)
)

// empty :: Combined
const empty =
  Pair(Maybe([]), Maybe(Sum.empty()))

// combine :: [ Person ] -> Combined
const combine = mapReduce(
  splitPerson,
  flip(concat),
  empty
)

combine(peeps)
  .bimap(option([]), option(Sum(0)))
//=> Pair( [ "Haskell", "Heinrich", "Maria" ], Sum 256 )
```

#### map

```haskell
Pair c a ~> (a -> b) -> Pair c b
```

Used to apply transformations to values to the second portion of a given `Pair`
instance. `map` takes a function that it will lift into the context of the
`Pair` and apply to it second value in the `Pair`, returning a new `Pair`
instance. The new instance will contain the result of mapping in the second,
leaving the value in the first untouched. If you need to map the first value,
[`bimap`](#bimap) can be used instead.

```javascript
import Pair from 'crocks/Pair'

import compose from 'crocks/helpers/compose'
import map from 'crocks/pointfree/map'
import merge from 'crocks/Pair/merge'
import objOf from 'crocks/helpers/objOf'

// length :: String -> Number
const length =
  x => x.length

// add10 :: Number -> Number
const add10 =
  x => x + 10

// keyedLength :: Pair String String -> Object
const keyedLength =
  compose(merge(objOf), map(length))

Pair('number', 32)
  .map(add10)
//=> Pair("number", 42)

keyedLength(
  Pair('text', 'This is some text')
)
//=> { text: 17 }
```

#### bimap

```haskell
Pair a b ~> ((a -> c), (b -> d)) -> Pair c d
```

The types and values that make up a `Pair` can vary independently in both the
first and second portions of the `Pair`. While [`map`](#map) can be used to
apply these transformations, `bimap` allows for independent transformations
on both sides, in parallel.

`bimap` takes (2) mapping functions as its arguments. The first function is used
to map the first, while the second maps the second. `Pair` only provides a
means to map the second's value exclusively using [`map`](#map). If the need
arises to map the first portion exclusively, use `bimap` passing the mapping
function to the first argument and an `identity` to the second.

```javascript
import Pair from 'crocks/Pair'
import Sum from 'crocks/Sum'

import compose from 'crocks/helpers/compose'
import bimap from 'crocks/pointfree/bimap'
import branch from 'crocks/Pair/branch'
import identity from 'crocks/combinators/identity'
import merge from 'crocks/Pair/merge'
import mreduce from 'crocks/helpers/mreduce'

// add10 :: Number -> Number
const add10 =
  x => x + 10

// divide :: (Number, Number) -> Number
const divide =
  (x, y) => x / y

// length :: [ Number ] -> Number
const length =
  x => x.length

// average :: [ Number ] -> Number
const average = compose(
  merge(divide),
  bimap(mreduce(Sum), length),
  branch
)

Pair(35, 'number')
  .bimap(add10, identity)
//=> Pair( 45, 'number' )

average([ 2, 3, 4 ])
//=> 3
```

#### ap

```haskell
Semigroup s => Pair s (a -> b) ~> Pair s a -> Pair s b
```

Short for apply, `ap` is used to apply a `Pair` instance containing a value on
its second portion to another `Pair` instance that contains a function in its
second portion. The result of this application provides a new `Pair` instance
containing the result in the second portion. `ap` requires that it is called on
an instance that wraps a curried polyadic function in the second.

An additional constraint when using `ap` is that the `Pair` must contain a
`Semigroup` instance in its first. This is required for both the `Pair` with
the function and the `Pair` with the value to be applied. With both `Semigroups`
being of the same type.

```javascript
import Pair from 'crocks/Pair'

import liftA2 from 'crocks/helpers/liftA2'

// add :: Number -> Number -> Number
const add =
  x => y => x + y

// NumberRec :: Pair [ Number ] Number

// twentyThree :: NumberRec
const twentyThree =
  Pair([ 23 ], 23)

// seventySeven :: NumberRec
const seventySeven =
  Pair([ 77 ], 77)

// combine :: NumberRec -> NumberRec -> NumberRec
const combine =
  liftA2(add)

twentyThree
  .map(add)
  .ap(seventySeven)
// Pair( [ 23, 77 ], 100 )

combine(twentyThree, seventySeven)
// Pair( [ 23, 77 ], 100 )
```

#### chain

```haskell
Semigroup s => Pair s a ~> (a -> Pair s b) -> Pair s b
```

Combining a sequential series of transformations that allows for custom
accumulation in addition to transforming a value. `chain` requires a `Pair`
returning function that contains a `Semigroup` in its first position. As an
additional requirement, is that instances of the same `Semigroup` must occupy
the first position of the source `Pair` and the `Pair` returned by the function.

```javascript
```

#### sequence

```haskell
Applicative TypeRep t, Apply f => Pair a (f b) ~> (t | (b -> f b)) -> f (Pair a b)
```

When an instance of `Pair` wraps an `Apply` instance in its second position,
`sequence` can be used to swap the type sequence. `sequence` requires either an
`Applicative TypeRep` or an `Apply` returning function is provided for its
argument.

While it is not a requirement that the first position be occupied by a
`Semigroup`, in having an instance there sequencing back on a data structure
with multiple items can allow for accumulation then sequencing back.

`sequence` can be derived from [`traverse`](#traverse) by passing it an
`identity` function (`x => x`).

```javascript
import Pair from 'crocks/Pair'
import Sum from 'crocks/Sum'

import bimap from 'crocks/pointfree/bimap'
import compose from 'crocks/helpers/compose'
import concat from 'crocks/pointfree/concat'
import flip from 'crocks/combinators/flip'
import sequence from 'crocks/pointfree/sequence'
import mapReduce from 'crocks/helpers/mapReduce'

// pair :: Pair Number [ String ]
const pair =
  Pair(1, [ 'a', 'b', 'c' ])

// empty :: () -> Pair Sum String
const empty =
  () => Pair(Sum.empty(), '')

// seqArray :: Traversable t => t [ a ] -> [ t a ]
const seqArray =
  sequence(Array)

// toUpper :: String -> String
const toUpper =
  x => x.toUpperCase()

// combine :: [ Pair Number String ] -> Pair Sum String
const combine = mapReduce(
  bimap(Sum, toUpper),
  flip(concat),
  empty()
)

// flow :: Pair Number [ String ] -> Pair Sum String
const flow =
  compose(combine, seqArray)

seqArray(pair)
//=> [ Pair(1, "a"), Pair(1, "b"), Pair(1, "c") ]

flow(pair)
//=> Pair(Sum 3, "ABC")
```

#### traverse

```haskell
Applicative TypeRep t, Apply f => Pair a b ~> ((t | (b -> f b)), (b -> f c)) -> f (Pair a c)
```

Used to apply the "effect" of an `Apply` to a value in the second position of
a `Pair`, `traverse` combines both the "effects" of the `Apply` and the `Pair`
by returning a new instance of the `Apply`, wrapping the result of the
`Apply`s "effect" on the value in the second position of the `Pair`.

`traverse` requires either an `Applicative TypeRep` or an `Apply` returning
function as its first argument and a function that is used to apply the "effect"
of the target `Apply` to the value in the second position of the `Pair`. The
"effect" will only be applied to second value and leaves the first value
untouched.Both arguments must return an instance of the target `Apply`.

```javascript
import Maybe from 'crocks/Maybe'
import Pair from 'crocks/Pair'

import identity from 'crocks/combinators/identity'
import safe from 'crocks/Maybe/safe'
import traverse from 'crocks/pointfree/traverse'

const { Just, Nothing } = Maybe

// isOdd :: Integer -> Boolean
const isOdd =
  x => !!(x % 2)

// safeOdd :: Traversable t => t Integer -> Maybe (t Integer)
const safeOdd =
  traverse(Maybe, safe(isOdd))

// seqMaybe :: Traversable t => t (Maybe a) -> Maybe (t a)
const seqMaybe =
  traverse(Maybe.of, identity)

// odd :: Pair [ Number ] Integer
const odd =
  Pair([ 10 ], 23)

// even :: Pair String  Integer
const even =
  Pair('nope', 42)

safeOdd(odd)
//=> Just Pair( [ 10 ], 23 )

safeOdd(even)
//=> Nothing

seqMaybe(Pair(true, Just('good')))
//=> Just Pair( true, "good" )

seqMaybe(Pair(false, Nothing()))
//=> Nothing
```

#### extend

```haskell
Pair a b ~> (Pair a b -> c) -> Pair a c
```

Used map the second position of a given `Pair` instance by taking the entire
`Pair` into consideration. `extend` takes a function the receives a `Pair` as
its input and returns a new `Pair` with the result of that function in the
second position, while leaving the value in the first position untouched.

```javascript
```

#### swap

```haskell
Pair a b ~> ((a -> c), (b -> d)) -> Pair d c
```

Used to map the value of a `Pair`s first position into the second position and
the second position into the first, `swap` takes (2) functions as its arguments.
The first function is used to map the value in the first position to the second,
while the second 'maps the second into the first'. If no mapping is required on
either side, then `identity` functions can be used in one or both arguments.

```javascript
```

#### fst

```haskell
Pair a b ~> () -> a
```

`fst` is one of (2) projection methods used to extract the values contained in
a given `Pair` instance. `fst` takes nothing as its input and will unwrap and
provide the value in the first position, throwing away the value in the second.
[`snd`](#snd) is the other projection function provided and is used to extract
the value in the second position.

```javascript
```

#### snd

```haskell
Pair a b ~> () -> b
```

`snd` is one of (2) projection methods used to extract the values contained in
a given `Pair` instance. `snd` takes nothing as its input and will unwrap and
provide the value in the second position, throwing away the value in the first.
[`fst`](#fst) is the other projection function provided and is used to extract
the value in the first position.

```javascript
```

#### toArray

```haskell
Pair a b ~> () -> [ a, b ]
```

While both [`fst`](#fst) and [`snd`](#snd) can be used to extract specific
values out of the structure of `Pair`, `toArray` extracts values but
maintains the structure. Taking nothing as its input, `toArray` will return an
`Array` of two values. The first value in the `Pair` will occupy the [0] index,
while the [1] index will house the second.

```javascript
```

#### merge

```haskell
Pair a b ~> ((a, b) -> c) -> c
```

Acting as a means to fold a given `Pair` over a binary operation, `merge` takes
a binary function as its sole argument. Using the function, `merge` will unwrap
each of its values and apply them to the function in order from first to second.
The result of the provided function is then provided as the overall result
for `merge`.

This method comes in handy when using a `Pair` as a means to run parallel
computations and combine their results into a final answer. Typically this
method works hand in hand with the either the [`branch`](#branch) or
`fanout`helper functions.

```javascript
```

</article>

<article id="topic-helpers">

## Helper Functions

#### branch

`crocks/Pair/branch`

```haskell
branch :: a -> Pair a a
```

Typically the starting point for handling parallel computations on a single
value, `branch` takes a single value of any type as its only argument. `branch`
then returns a `Pair` with the reference or value in both the first and second
positions.

Using `branch` can simplify how computations that depend on the same value are
constructed and encoded by removing the need to keep the original value in some
state that needs to be passed from function to function.

```javascript
```

#### toPairs

`crocks/Pair/toPairs`

```haskell
toPairs :: Object -> List (Pair String a)
```

When dealing with `Object`s, sometimes it makes more sense to work in a
`Foldable` structure like a `List` of key-value `Pair`s. `toPairs` provides a
means to take an object and give you back a `List` of `Pairs` that have a
`String` that represents the key in the `fst` and the value for that key in the
`snd`. The primitive values are copied, while non-primitive values are
references. Like most of the `Object` functions in `crocks`, any keys with
`undefined` values will be omitted from the result. `crocks` provides an inverse
to this function named [`fromPairs`][frompairs].

```javascript
```

</article>

<article id="topic-pointfree">

## Pointfree Functions

#### fst (pointfree)

`crocks/Pair/fst`

```haskell
fst :: Pair a b -> a
```

The `fst` pointfree function is used extract the leftmost value of a `Pair` by
invoking the [`fst`](#fst) method on a given instance, returning the result.
`fst` takes a `Pair` as its only argument and returns the value wrapped in the
leftmost portion of the provided `Pair`.

```javascript
```

#### merge (pointfree)

`crocks/Pair/merge`

```haskell
merge :: ((a, b) -> c) -> Pair a b -> c
```

The pointfree `merge` function allows the ability to fold out a new value, using
a binary function to combine the two values into one. This function takes (2)
arguments, with the first being a binary function used to combine a `Pair`s
values and the second is the instance of `Pair` for the function to be applied.
Like the method on `Pair`, this function will return the result of the provided
binary function.

```javascript
```

#### snd (pointfree)

`crocks/Pair/snd`

```haskell
snd :: Pair a b -> b
```

The `snd` pointfree function is used extract the rightmost value of a `Pair` by
invoking the [`snd`](#snd) method on a given instance, returning the result.
`snd` takes a `Pair` as its only argument and returns the value wrapped in the
rightmost portion of the provided `Pair`.

```javascript
```

</article>

<article id="topic-transformation">

## Transformation Functions

#### writerToPair

`crocks/Pair/writerToPair`

```haskell
writerToPair :: Monoid m => Writer m a -> Pair m a
writerToPair :: Monoid m => (a -> Writer m b) -> a -> Pair m b
```

Used to transform a `Writer` instance to a `Pair` instance,
`writerToPair` will take a given `Writer` and provide a new `Pair` with
the `log` portion of the `Writer` in the first position and the `resultant`
in the second.

Like all `crocks` transformation functions, `writerToPair` has (2) possible
signatures and will behave differently when passed either a `Writer` instance
or a function that returns an instance of `Writer`. When passed the instance,
a `Pair` instance is returned. When passed a `Writer` returning function,
a function will be returned that takes a given value and returns an `Pair`.

```javascript
```

</article>