# Variadic Generics

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): Douglas Gregor, TBD, Austin Zheng
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

Many kinds of generic types can take multiple concrete forms, differing only in the number of generic type parameters. We propose, in the spirit of [Completing Generics](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#variadic-generics), to add support for *variadic generics* to Swift.

## Motivation

Tuples illustrate well the problem that variadic generics try to solve. There are many different types of tuples: 2-ples, 3-ples, 4-ples, 15-ples, and so on:

```swift
// 2-ple
let a : (Int, String)

// 6-ple
let z : (String, String, UIViewController, Bool, Double, Bool)
```

One might want to make tuples containing `Equatable` elements comparable using `==` and `!=`. In order to implement this sort of feature, it is necessary to choose a finite number of tuple arities and write a separate comparison function for each arity. In fact, [SE-0015](https://github.com/apple/swift-evolution/blob/master/proposals/0015-tuple-comparison-operators.md) implements comparisons for tuples up to arity 6:

```swift
// Arity 2 implementation
public func ==<A : Equatable, B : Equatable>(lhs: (A, B), rhs: (A, B)) -> Bool {
  return lhs.0 == rhs.0 && lhs.1 == rhs.1
}

// Arity 3 implementation
public func ==<A : Equatable, B : Equatable, C : Equatable>(lhs: (A, B, C), rhs: (A, B, C)) -> Bool {
  return lhs.0 == rhs.0 && lhs.1 == rhs.1 && lhs.2 == rhs.2
}

// ...
```

This sort of implementation, while the best we can do currently, requires significant boilerplate code and is necessarily limited to whatever arities we end up choosing. At the same time, we can sense that the individual operator functions all share some sort of underlying structure that we might be able to factor out: in fact, we could probably write the *n*th arity implementation in terms of the *n-1*th arity implementation.

## Proposed solution

This proposal proposes *variadic generics* as a way to abstract over this sort of shared structure. The proposed implementation, described below, 'borrows' the C++ variadic templates syntax proposed in *Completing Generics*. Suggestions for alternative syntax welcome.

As with normal generic declarations today, a variadic generic declaration is considered "less specific" by the typechecker, and an equivalent non-variadic generic declaration will be chosen preferentially over a variadic generic declaration if one exists. (The exact semantics depend on how the feature is implemented; see *Implementation*, below.)

### Overview

**Parameter packs** will be used to define 'vectors' of generic type parameters. They can be used to define the types of **value packs**, which are corresponding 'vectors' of arguments, expressions, or variables. Both parameter and value packs are declared with a *leading* `...`, and used with a *trailing* `...`.

Sometimes a vector of types or values is desired (for example, when passing arguments into a function with an arity of 2 or more). Sometimes a tuple is desired. A parameter or value pack can be 'converted' into a corresponding scalar tuple type or value by wrapping it in `( )`. A tuple can be turned back into a parameter or value pack using `#unpack()`.

It is often useful to reduce a vector of values to a scalar. To this end `#first()`, which produces a scalar type or value, and `#rest()`, which produces a pack of one fewer arity than the input, are provided. This can be used to perform a calculation by spreading it across the arities of a variadic function. The example variadic implementation of `==` for tuples makes use of this functionality.

Finally, `#invert()` is provided to convert a `T?...` into a `(T...)?`.

*TODO*: what about splatting packs together, or splatting packs with scalars? For example, `(T..., U..., V)`; if `T...` has 5 elements and `U...` has 9, the overall tuple has 15 elements. Would this be useful? Disadvantages of this sort of aggregative composition: write code that runs forever only because of generic arity.

*TODO*: based on [this](https://github.com/atrick/swift/blob/type-safe-mem-docs/docs/TypeSafeMemory.rst#layout-compatible-types), smaller aggregate types are usually layout-compatible with larger aggregate types containing the smaller type. This [does not seem to be true](https://github.com/rust-lang/rfcs/issues/376#issuecomment-213692855) for Rust.

### Tuples

The expression `(U...)`, where `U...` contains *n* packed parameters or values, expands into the following tuple type or value:

```swift
(U1, U2, ..., Un)
```

For a value pack, `(v...)` can be thought of as the following pseudo-expansion:

```swift
let tuple : (T...) = (v1, v2, v3, ..., vn)
```

It is a tuple of *n* arity, because there are *n* packed parameters or values.

### Parameter packs

`...T` is a **parameter pack**. It represents an arbitrary number of *n* individual, independent type parameters (henceforth denoted as T1, T2, T3, and so on). The prefix `...` is used to declare that a type variable `T` represents a parameter pack, not an individual generic type.

For example, if a generic type is declared like this...

```swift
struct Foo<...T> { ... }
```

...it can be instantiated like such:

```swift
// one-arity Foo
struct Foo<NSIndexSet> { ... }

// five-arity Foo
struct Foo<Int, String, Bool, Bool, UIView> { ... }
```

#### Unpacking

The postfix `...` construct *unpacks* a parameter pack. It shows up everywhere a parameter pack is used besides its declaration:

* `T...` expands to:

	`T1, T2, ..., Tn`

Postfix `...` can be applied to a generic type's associated types:

* `T.Iterator.Element...` expands to:

	`T1.Iterator.Element, T2.Iterator.Element, ..., Tn.Iterator.Element`

Postfix `...` can also be applied to a series of generic types parameterized by the packed type parameters:

* `[T]...` expands to:

	`[T1], [T2], ..., [Tn-1]`

If prefix or postfix `...` is followed by a protocol or subclass conformance requirement, that requirement is applied to each packed type parameter or associated type thereof:

* `...T : Fooable where T... : Barrable` expands to:

	`T1 : Fooable, T2 : Fooable, ..., Tn : Fooable where T1 : Barrable, T2 : Barrable, ..., Tn : Barrable`

If postfix `...` is used to construct a same-type constraint between two parameter packs, those packs can only be used in such a way that they have the same number of elements:

* `...U where T.Iterator.Element... == U...` forces `T` and `U` to have the same number of items, and expands to:

	`U1, U2, ... Un where T1.Iterator.Element == U1, ..., Tn.Iterator.Element = Un`

#### Usage

Parameter packs are used to specify the types of value packs (see *Value packs*, below).

They can also be used to parameterize the use of variadic generic types or functions within a variadic generic type or function (from *Completing Generics*):

```swift
public struct ZipIterator<...Iterators : IteratorProtocol> : Iterator { ... }

public struct ZipSequence<...Sequences : Sequence> : Sequence {
	// ZipSequence is a variadic generic type that has a function returning a
	// different, but related variadic generic type.
	func makeIterator() -> ZipIterator<Sequences.Iterator...> { ... }
}
```

### Value packs

Parameter packs, which are packs of types, are used in conjunction with value packs. Parameter packs define the types of value packs, which represent zero or more values.

The prefix `...` is used at the declaration of a value pack. The postfix `...` is used whenever a pack is used (for example, in order to build an expression, or to declare the type of a value pack). It provides an immediate visual indicator that the identifier in question is not a normal identifier, but a pack.

There are a number of different types of value packs, detailed below.

#### Function arguments

If a parameter pack is used as the type of an argument to an initializer, function, or method, that argument must be declared as an **argument pack** with the leading `...`. In the following example, `widgets` is an argument pack.

Argument packs never have argument labels.

```swift
func foo<...U>(...widgets: U...) {
	// 'widgets' is not a single argument, it's a pack of arguments
}

// Usage:
foo(1, 2, "hello", true)
```

A function may only have one argument pack, but can have zero or more 'normal' arguments as well:

```swift
func foo<...U : Fooable>(x: Int, ...fooables : Fooable, y: String) { ... }

// Usage (assume Foo, Bar, and Baz all conform to Fooable)
foo(x: 15, Foo(), Bar(), Bar(), Baz(), y: "hello")
```

#### Properties and local variables

A variadic number of properties or local variables can be defined on a type by creating a **property pack** or **variable pack** with the leading `...`. The type of such a pack must be a tuple populated only by a parameter pack:

```swift
struct Foo<...U> {
	// 'greebles' is not a single property, it's a pack of properties
	// prefix '...' is only needed at the declaration, not site of use
	var (...greebles) : (U...)
}
```

A property pack can only be populated by setting it equal to a value pack of the same type:

```swift
class Foo<...U> {
	var (...greebles) : (U...)

	init(...initialGreebles: U...) {
		// Conceptually: "greebles1 = initialGreebles1; greebles2 = initialGreebles2, ..."
		greebles... = initialGreebles...
	}
}
```

Note the distinction between a property pack, and a scalar tuple built out of a value pack:

```swift
struct MyStruct<...T>() {
	
	// 'foo' is a value pack, not a tuple. This is exactly consistent with
	// Swift's current syntax for declaring multiple properties. 
	var (...foo) : (T...)

	// 'foo2' is a tuple, not a value pack.
	var foo2 : (T...)

	func doSomething() {
		// 'bar' is a tuple, not a value pack. It is a tuple of type (T...),
		// built by spreading the property pack 'foo' out into a tuple.
		var bar : (T...) = (foo...)

		// okay
		foo2 = bar
		// ...
	}
}
```

#### Expressions

An **expression pack** of type `...T` is formed by taking a value pack pack of type `...U`, and forming an expression for each element in the input pack. It can be seen as the transform `U... -> T...`. If such a relationship is defined, the number of elements in `...T` and `...U` must be equal.

#### Usage

A value pack of type `T...` can be used as an input to a different value pack. For example, the following code populates a property pack based on the first elements of an argument pack of collections:

```swift
class CollectionBag<...U where U... : Collection> {
	var (...collections) : (U...)
	var (...firstElements) : (U.Iterator.Element...)

	init(...colls: U...) {
		self.collections... = colls...

		// An example of a property pack being set by an expression pack.
		// Conceptually: "firstElements1 = colls.first!, firstElements2 = colls.first!, ..."
		firstElements... = colls.first!...
	}
}
```

They can also be used to create a scalar tuple value of type `(T...)`. This can be seen as a many-to-one transform:

```swift
func firstElements<C... : CollectionType>(...c : C...) -> (C.Iterator.Element...) {
	// Conceptually: let a = (c1.first!, c2.first!, ..., cn.first!)
	// Note that 'a' is not declared as '...a', because it's not a pack. It's a single tuple.
	let a : C.Iterator.Element... = (c.first!...)
	return a
}
```

Finally, they can be passed to a function, method, or initializer defined with an argument pack.

```swift
public struct ZipIterator<...Iterators : IteratorProtocol> : Iterator {
	init(arguments: Iterators...) { ... }
}

public struct ZipSequence<...T : Sequence> : Sequence {
	var (...sequences) : (T...)

	init(...arguments: T...) {
		sequences... = arguments...
	}

	func makeIterator() -> ZipIterator<T.Iterator...> { 
		// Conceptually: "init(sequences1.makeIterator(), ..., sequences_n.makeIterator())"
		return ZipIterator.init(sequences.makeIterator()...)
	}
}
```

An argument pack cannot be used to satisfy a non-pack generic parameter.

### `#unpack()`, `#first(T...)`, and `#rest(T...)`

Unfortunately, the most interesting use cases for variadic generics are impossible without some form of reduction from a value pack or tuple, or parameter pack, to a scalar value or type. These include all the examples outlined in *Completing Generics*.

Three special language constructs are therefore proposed to implement this reduction:

* `#First(T...)` is a special generic type (i.e. `T1`), which is the first generic type in the parameter pack, or `()` if the pack is empty.

* `#first(v...)`, for the value pack `...v`, is the first expression within the value pack, or `()` if the pack is empty.

* `#Rest(T...)` is a special generic parameter pack (i.e. `...Trest`, or `T2, T3, ..., Tn`), which consists of all the generic types in the parameter pack `...T` except the first, or `()` if the pack is empty.

* `#rest(v...)`, for the value pack `...v`, is a value pack consisting of `...v`'s members except the first, or `()` if the pack is empty.

* If `t` is a tuple of type `(T...)`, then `#unpack(t)` is a value pack of type `T...`.

Here is an example of these constructs, used to construct a function that can compare tuples of arbitrary arity.

```swift
// Generalized tuple comparison

// This function is chosen preferentially over the variadic generic function
// when `T...` is empty.
func ==(lhs: (), rhs: ()) -> Bool {
	return true
}

func ==<...T : Equatable>(lhs: (T...), rhs: (T...)) {
	// Peel the first items off the tuples
	let firstLeft : #First(T...) = #first(lhs)
	let firstRight : #First(T...) = #first(rhs)

	// Compare them. If they're false, we can bail
	let firstAreEqual = (firstLeft == firstRight)
	if !firstAreEqual {
		return false
	} else {
		// Turn the rest of the items in 'lhs' and 'rhs' into a tuple
		let restLeft : (#Rest(T...)) = (#rest(#unpack(lhs)))
		let restRight : (#Rest(T...)) = (#rest(#unpack(rhs)))
		return restLeft == restRight
	}
}
```

For any tuple of arity 2 or higher, this operator function should run recursively until the arity reaches 1, at which point the non-tuple `==` would be invoked directly on the participating types. Even though this operator function should never be chosen when `T...` is `()` or just `(T)`, all the parameter and value packs in the function have well-defined behavior for all possible arities, so the compiler does not need to rely on the previous fact to prove that the types are sound.

### `#invert()`

One final construct is the `#invert()` compile-time feature. `#invert(v...)`, where `v...` is a value pack of parameter pack type `T?...`, produces a scalar tuple value of type `(T...)?`. This is useful for working with variadic expressions that generate optionals.

Here is a (simplified) example:

```swift
// Non-variadic
func foo<G1, G2, G3>(iterator1: G1, iterator2: G2, iterator3: G3) {
	var a = (iterator1, iterator2, iterator3)	// type: (G1, G2, G3)
	var b = (a0.next(), a1.next(), a2.next())	// type: (G1.Element?, G2.Element?, G3.Element?)

	// Turn (G1.Element?, G2.Element?, G3.Element?) into (G1.Element, G2.Element, G3.Element)?
	let c : (G1.Element, G2.Element, G3.Element)?
	if case let (x0?, x1?, x2?) = b {
		c = (x0, x1, x2)
	} else {
		c = nil
	}
}

// Variadic
func foo<...G : Iterator>(...iterators : G) {
	var b = (iterators.next()...)	// type: (G.Element?...)
	var c = #invert(#unpack(b)) // type: (G.Element...)?
}
```

*This is a terrible design hack. Any suggestions for a more elegant replacement, or even better naming, would be greatly appreciated.*

## Detailed design

### Zero-count parameter packs

*TODO*: what to do about a generic type parameterized only on a generic pack, when that pack has zero types?

```swift
struct Foo<...Elements> { ... }

let a : Foo<> // Swift doesn't currently support generic types with 0 params
```

### Implementation

It's not clear how variadic generics can be implemented. Input from a compiler engineer would be appreciated.

C++ utilizes *templates*, which are vaguely similar to C's macro preprocessor in that a copy of the template is created for each type the template is specialized on at compile-time. This makes implementation extremely straightforward - simply instantiate as many arities of a given generic type as are needed at compile-time. Any type that uses e.g. `#rest()` in its definition would be specialized with every arity up to the maximum arity it would be used with.

Unfortunately, C++'s generics come with significant limitations, including the fact that templates that static libraries wish to expose to their consumers must be present as source code in header files.

Swift's generics are not templates; they do not have to be specialized and they can be invoked from library code even without the source being available to the consumer. This is because the generics system is designed in such a way that protocol methods can be dynamically dispatched from a conforming type unknown to the module the generic type or function was originally defined in.

It is not obvious what sort of features would allow a generic type in a library to be correspondingly instantiated for previously-unknown arities.

## Case Studies

A few examples of how variadic generics could be used follow:

### Zippers

Swift currently has `Zip2Sequence` and `Zip2Iterator`, for zipping two sequences together. These could conceivably be removed and replaced by a variadic set of zip types, as described in *Completing Generics*:

```swift
struct ZipSequence<...T : Sequence> {
	typealias Iterator = ZipIterator<T.Iterator...>

	var (...sequences) : (T...)

	func makeIterator() -> ZipIterator<T.Iterator...> { 
		return ZipIterator.init(sequences.makeIterator()...)
	}

	init(...sequences: T...) {
		self.sequences... = sequences...
	}
}

struct ZipIterator<...U : IteratorProtocol> : Iterator {
	typealias Element = (U.Element...)

	var (...iterators) : (U...)

	init(...iterators: Iterators...) { 
		self.iterators... = iterators...
	}

	mutating func next() -> (U.Element...)? {
		if let nextValues = #invert(iterators.next()...) {
			return nextValues
		}
		return nil
	}
}

let myZip = ZipSequence([1, 2, 3, 4, 5], [2, 4, 6, 8, 10], ["a", "b", "c", "d", "e"])
for (first, second, third) in myZip {
	// ...
}
```

### Generalized function application

Generalized function application has at least two uses: allowing a function or method to be called in a type-safe manner with an arbitrary number of arguments, and allowing for type-safe reflection upon a types's methods.

An example of a possible 'tuple splat' operator:

```swift
func apply<...Args, ReturnType>(function: (Args...) -> ReturnType, arguments: (Args...)) -> ReturnType {
	return function(#unpack(arguments))
}
```

An example of a possible reflection interface for attempting to get a type-safe reference to a method from a string:

```swift
// Given an object and a method name in some canonical format (e.g. 
// "foo(_:hello:)"), return a function value of the expected type if one
// exists, or nil otherwise.
func methodOnObject<Object, ...Args, ReturnType>(object: Object, name: String) -> ((Args...) -> ReturnType)? {
	...
}
```

### Multiple arity HOFs

Dynamic languages like Clojure feature higher-order functions that have an arbitrary number of arities. For example, Clojure's [`map` function](https://www.conj.io/store/v1/org.clojure/clojure/1.8.0/clj/clojure.core/map) takes a 'transforming' function and one or more collections:

```clojure
(map inc [1 2 3 4 5])
;; => (2 3 4 5 6)

(map + [1 2 3] [4 5 6])
;; => (5 7 9)
```

Note that `inc` is a function that takes one parameter (it increments a number), and `+` is a function that takes in two parameters. Varidaic generics would allow a Swift programmer to write an almost-as-expressive yet type-safe function:

```swift
func multiMap<...T : Sequence, U>(_ f: (T.Iterator.Element...) -> U, ...sequences: T...) -> [U] {
	var buffer : [U] = []

	// Precondition: all sequences are the same length
	var (...iterators) = (sequences.makeIterator()...)

	while true {
		// Get all the values out of the sequences for this iteration (they'll be optionals)
		var (...next) = (iterators.next()...)
		// Perform an inversion
		if let result = #invert(next...) {
			// Feed the result into the function and add the multimapped value to the buffer
			buffer.append(f(#unpack(result)))
		} else {
			// We're done, return the buffer
			return buffer
		}
	}
}

let results = multiMap(+, [1, 2, 3], [4, 5, 6])
// results = [5, 7, 9]
```

## Impact on existing code

No impact on existing code; this is a greenfield feature.

## Alternatives considered

Don't add this feature to the language.

### Alternative design choices

***Fold expressions***

In addition to the tuple-based and recursive operations for reducing packs to scalars, the notion of an explicit construct for reduction of packs (like [C++'s fold expressions](http://en.cppreference.com/w/cpp/language/fold)) was considered. Fold expressions in C++ only work with a fixed set of unary and binary operators. This is significant increased complexity and should be deferred to a follow-up proposal, if considered at all:

```swift
// Ugly pseudocode
// #requirements is some way to abstract across all requirements defined on each parameter pack,
// probably taking the form of an enhanced existential or something.
#fold<...T : #requirements, U>(start: U, reducer: (#requirements, U) -> U, ...values: T...) -> U

#foldl (...)
#foldr (...)
```

TBD

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.
