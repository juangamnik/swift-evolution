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

**Parameter vectors**, similar to C++'s *parameter packs*, will be used to define 'vectors' of generic type parameters. They can be used to define the types of **value vectors**, which are corresponding 'vectors' of arguments, expressions, or variables. Both parameter and value vectors are declared with a *leading* `...`, and used with a *trailing* `...`.

Sometimes a vector of types or values is desired (for example, when passing arguments into a function with an arity of 2 or more). Sometimes a tuple is desired. A parameter or value vector can be 'converted' into a corresponding scalar tuple type or value by wrapping it in `#tuple()`. A tuple can be turned back into a parameter or value vector using `#vector()`.

It is often useful to reduce a vector of values to a scalar. To this end `#first()`, which produces a scalar type or value, and `#rest()`, which produces a vector of one fewer arity than the input, are provided. This can be used to perform a calculation by spreading it across the arities of a variadic function. The example variadic implementation of `==` for tuples makes use of this functionality.

An alternative to recursion is the `#fold()` construct, which folds over all the values of a value vector using a reducer function that can accept all of the individual values.

Finally, `#invert()` is provided to convert a `T?...` into a `(T...)?`.

### Parameter Vectors

`...T` is a **parameter vector**. It represents an arbitrary number of *n* individual, independent type parameters (henceforth denoted as T1, T2, T3, and so on). The prefix `...` is used to declare that a type variable `T` represents a parameter vector, not an individual generic type.

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

Any given declaration (such as a function arguments signature or a generic type parameters signature) can have only one "directly exposed" parameter vector. However, it should be possible to wrap additional vectors in the `#tuple()` construct and expose them in that manner:

```swift
// There are always some number of variadic type parameters followed by a tuple
// containing an arbitrary number of elements.
struct Bag<...T, #tuple(...U)> {
	// ...
}
```

#### Typealiasing and concrete types

New parameter vectors inside the scope of a type can be declared using `typealias`. A typealias defined in terms of another parameter vector is necessarily bound to the length of that parameter vector:

```swift
struct Foo<...C : Collection> {
	// Indices and C are bound to be the same length.
	typealias ...Indices = C.Index...

	// A value vector of indices
	var (...startIndices) : (Indices...)

	init(...x: C...) {
		startIndices... = x.startIndex...
	}
}
```

A concrete type 'pseudo-vector' may also be defined using postfix `...`. Such a type can only be used to denote a property or variable vector that was initialized, at some point, based on a value whose type was a parameter vector defined in the variadic generic type's generic type header:

```swift
struct Foo<...T : CustomStringConvertible> {

	var (...descriptions) : (String...)

	init(...x: T) {
		// .description goes from Tn -> String
		// Because of this, descriptions' length is identical to that of T.
		descriptions... = x.description...
	}
}
```

For example, this is not allowed:

```swift
struct Foo<...T : CustomStringConvertible> {

	var (...descriptions) : (String...)

	func doSomething() {
		descriptions... = ("foo", "bar", "baz")
	}
}
```

#### Unpacking

The postfix `...` construct *unpacks* a parameter vector. It shows up everywhere a parameter vector is used besides its declaration:

* `T...` expands to:

	`T1, T2, ..., Tn`

Postfix `...` can be applied to a generic type's associated types to form a new parameter vector:

* `T.Iterator.Element...` expands to:

	`T1.Iterator.Element, T2.Iterator.Element, ..., Tn.Iterator.Element`

Postfix `...` can also be applied to a series of generic types parameterized by the vectored type parameters:

* `[T]...` expands to:

	`[T1], [T2], ..., [Tn-1]`

#### Constraints

If a parameter vector is followed by a protocol or subclass conformance requirement, that requirement is applied to each individual type parameter or associated type thereof:

* `...T : Fooable where T... : Barrable` expands to:

	`T1 : Fooable, T2 : Fooable, ..., Tn : Fooable where T1 : Barrable, T2 : Barrable, ..., Tn : Barrable`

If a parameter vector's associated type vector is followed by a concrete type equality constraint, that requirement is applied to each individual type parameter:

* `...T where T.Iterator.Element... == Int` expands to:

	`T1, T2, ... Tn where T1.Iterator.Element == Int, ..., Tn.Iterator.Element = Int`

If a parameter vector's associated type vector is followed by a non-variadic generic type equality constraint, then all the associated types must be equal, and that type is bound to `U`.

* `...T, U where T.Iterator.Element... == U` expands to:

	`T1, T2, ... Tn where T1.Iterator.Element == U, ..., Tn.Iterator.Element = U`

If postfix `...` is used to construct a same-type constraint between two parameter vectors, those vectors can only be used in such a way that they have the same number of elements:

* `...U where T.Iterator.Element... == U...` forces `T` and `U` to have the same number of items, and expands to:

	`U1, U2, ... Un where T1.Iterator.Element == U1, ..., Tn.Iterator.Element = Un`

A `where` clause may contain the following form, `#allequal(X)`, where `X` is a parameter vector's associated type vector. This forces all members of the associated type vector to be equal to each other, and obviates the need for an additional dummy type parameter:

* `where #allequal(T.Iterator.Element), ...` expands to:

	`where T1.Iterator.Element == T2.Iterator.Element, T1.Iterator.Element == T3.Iterator.Element, ...`

#### Usage

Parameter vectors are used to specify the types of value vectors (see *Value vectors*, below).

They can also be used to parameterize the use of variadic generic types or functions within a variadic generic type or function (from *Completing Generics*):

```swift
public struct ZipIterator<...Iterators : IteratorProtocol> : Iterator { ... }

public struct ZipSequence<...Sequences : Sequence> : Sequence {
	// ZipSequence is a variadic generic type that has a function returning a
	// different, but related variadic generic type.
	func makeIterator() -> ZipIterator<Sequences.Iterator...> { ... }
}
```

### Value vectors

Parameter vectors, which are vectors of types, are used in conjunction with value vectors. Parameter vectors define the types of value vectors, which represent zero or more values.

The prefix `...` is used at the declaration of a value vector. The postfix `...` is used whenever a vector is used (for example, in order to build an expression, or to declare the type of a value vector). It provides an immediate visual indicator that the identifier in question is not a normal identifier, but a vector.

There are a number of different types of value vectors, detailed below.

#### Function arguments

If a parameter vector is used as the type of an argument to an initializer, function, or method, that argument must be declared as an **argument vector** with the leading `...`. In the following example, `widgets` is an argument vector.

Argument vectors never have argument labels.

```swift
func foo<...U>(...widgets: U...) {
	// 'widgets' is not a single argument, it's a vector of arguments
}

// Usage:
foo(1, 2, "hello", true)
```

A function may only have one argument vector, but can have zero or more 'normal' arguments as well:

```swift
func foo<...U : Fooable>(x: Int, ...fooables : Fooable, y: String) { ... }

// Usage (assume Foo, Bar, and Baz all conform to Fooable)
foo(x: 15, Foo(), Bar(), Bar(), Baz(), y: "hello")
```

#### Properties and local variables

A variadic number of properties or local variables can be defined on a type by creating a **property vector** or **variable vector** with the leading `...`. The type of such a vector must be a tuple populated only by a parameter vector:

```swift
struct Foo<...U> {
	// 'greebles' is not a single property, it's a vector of properties
	// prefix '...' is only needed at the declaration, not site of use
	var (...greebles) : (U...)
}
```

A property vector can only be populated by setting it equal to a value vector of the same type:

```swift
class Foo<...U> {
	var (...greebles) : (U...)

	init(...initialGreebles: U...) {
		// Conceptually: "greebles1 = initialGreebles1; greebles2 = initialGreebles2, ..."
		greebles... = initialGreebles...
	}
}
```

Note the distinction between a property vector, and a scalar tuple built out of a value vector:

```swift
struct MyStruct<...T>() {
	
	// 'foo' is a value vector, not a tuple. This is exactly consistent with
	// Swift's current syntax for declaring multiple properties. 
	var (...foo) : (T...)

	// 'foo2' is a tuple, not a value vector.
	var foo2 : #tuple(T...)

	func doSomething() {
		// 'bar' is a tuple, not a value vector. It is a tuple of type (T...),
		// built by spreading the property vector 'foo' out into a tuple.
		var bar : #tuple(T...) = (foo...)

		// okay
		foo2 = bar
		// ...
	}
}
```

#### Expressions

An **expression vector** of type `...T` is formed by taking a value vector vector of type `...U`, and forming an expression for each element in the input vector. It can be seen as the transform `U... -> T...`. If such a relationship is defined, the number of elements in `...T` and `...U` must be equal.

An expression can be a method, properties, or other members called on each element of the value vector (subject to the constraints placed on the value vector's parameter vector), or it can be a chain of said member invocations.

It can also be the result of applying that method to a function that takes one parameter, written as the following:

```swift
func foo(x: Any) -> String { ... }

struct Bag<...T> {
	let (...x) : (String...)

	init(...things: T...) {
		x... = things.#apply(foo)...
	}
}
```
*TBD*: is this `#apply()` even necessary?

#### Usage

A value vector of type `T...` can be used as an input to a different value vector. For example, the following code populates a property vector based on the first elements of an argument vector of collections:

```swift
class CollectionBag<...U where U... : Collection> {
	var (...collections) : (U...)
	var (...firstElements) : (U.Iterator.Element...)

	init(...colls: U...) {
		self.collections... = colls...

		// An example of a property vector being set by an expression vector.
		// Conceptually: "firstElements1 = colls.first!, firstElements2 = colls.first!, ..."
		firstElements... = colls.first!...
	}
}
```

They can also be used to create a scalar tuple value of type `(T...)`. This can be seen as a many-to-one transform:

```swift
func firstElements<C... : CollectionType>(...c : C...) -> (C.Iterator.Element...) {
	// Conceptually: let a = (c1.first!, c2.first!, ..., cn.first!)
	// Note that 'a' is not declared as '...a', because it's not a vector. It's a single tuple.
	let a : C.Iterator.Element... = (c.first!...)
	return a
}
```

Finally, they can be passed to a function, method, or initializer defined with an argument vector.

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

An argument vector cannot be used to satisfy a non-vector generic parameter.

### Working with variadic generics

A very limited set of compile-time programming facilities are provided, for expressing how variadic generics should generate code.

#### `#tuple()`

`#tuple()` is used to turn a type or parameter vector into a corresponding scalar tuple type or tuple instance.

The expression `#tuple(U...)`, where `U...` contains *n* member parameters or values, expands into the following tuple type or value:

```swift
(U1, U2, ..., Un)
```

For a value vector, `#tuple(v...)` can be thought of as the following pseudo-expansion:

```swift
let tuple : #tuple(T...) = (v1, v2, v3, ..., vn)
```

It is a tuple of *n* arity, because there are *n* member parameters or values.

#### `#vector()`

`#vector()` is used to turn an instance of a tuple into a value vector.

If `t` is a tuple of type `#tuple(T...)`, then `#vector(t)` is a value vector of type `T...`. 

`#vector()` cannot be used on tuples whose types were not constructed using `#tuple`.

#### `#first(T...)` and `#rest(T...)`

Unfortunately, the most interesting use cases for variadic generics are impossible without some form of reduction from a value vector or tuple, or parameter vector, to a scalar value or type. These include all the examples outlined in *Completing Generics*.

Three special language constructs are therefore proposed to implement this reduction:

* `#First(T...)` is a special generic type (i.e. `T1`), which is the first generic type in the parameter vector, or `()` if the vector is empty.

* `#first(v...)`, for the value vector `...v`, is the first expression within the value vector, or `()` if the vector is empty.

* `#Rest(T...)` is a special generic parameter vector (i.e. `...Trest`, or `T2, T3, ..., Tn`), which consists of all the generic types in the parameter vector `...T` except the first, or `()` if the vector is empty.

* `#rest(v...)`, for the value vector `...v`, is a value vector consisting of `...v`'s members except the first, or `()` if the vector is empty.

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
		let restLeft : (#Rest(T...)) = (#rest(#vector(lhs)))
		let restRight : (#Rest(T...)) = (#rest(#vector(rhs)))
		return restLeft == restRight
	}
}
```

For any tuple of arity 2 or higher, this operator function should run recursively until the arity reaches 1, at which point the non-tuple `==` would be invoked directly on the participating types. Even though this operator function should never be chosen when `T...` is `()` or just `(T)`, all the parameter and value vectors in the function have well-defined behavior for all possible arities, so the compiler does not need to rely on the previous fact to prove that the types are sound.

#### `#fold()`

Recursive variadic types or functions can be cumbersome to implement and result in significant code generation. An alternative to doing so is 'folding' each element in a value vector in turn based on the previous value and some sort of reduction function. C++ implements this sort of functionality through [fold expressions](http://en.cppreference.com/w/cpp/language/fold).

The following construct is proposed: `#fold()`. It takes in a starting value of some type, a reducing function, and a value vector. The reducer is used, in conjunction with the starting value, to reduce the entire vector in a scalar value. If the pack is empty, the starting value is simply returned.

`#fold()` is defined in the following manner:

```swift
#fold<...T, U>(start: U, reducer: (#requirements, U) -> U, values: T...) -> U
```

`start` is a scalar expression of type `U`. This same type is returned by the `#fold()` pseudo-expression.

`values` is a value pack of type `T...`.

`reducer` is a function that takes in two arguments. The second argument's type must be `U`, and the function must return `U`.

`#requirements` is a placeholder, only for the sake of this document, used to describe an appropriate type for the first argument to `reducer`. That type must be any type, whether existential, concrete, or generic, which can satisfy every single type member of `T...` at compile time. This can always be `Any`. Here are other values it can take on:

* If `T...` has a concrete type which is forced upon it by its definition, then the type of `#requirements` can also be that concrete type or a supertype.

	One way this can happen is if `T...` is another parameter vector's associated type vector, and has been constrained to a concrete type:

	```swift
	struct Bag<...T : Collection where T.Element... == Int> {
		typealias ...U = T.Element...

		// All the elements are integers; there are as many ints as there are types in T...
		let (...firstElements) : (U...)

		let reducer : (Int, Int) -> Int = +

		func sumOfFirstElements() -> Int {
			// Reducer's "#requirements" type can be Int
			let result = #fold(start: 0, reducer: reducer, values: firstElements...)
			return result
		}
	}
	```

	Another way this can happen is if `T...` is the type of an expression vector resulting from an expression that returns a concrete type:

	```swift
	struct Bag<...T : Collection: Fooable> {
		let (...collections) : (T...)

		func totalCount() -> Int {
			let ...counts = collections.count...
			let result = #fold(start: 0, reducer: +, values: counts...)
			return result
		}
	}
	```

* If `T...` was a variadic generic type parameter, and was given constraints, the type of `#requirements` can be exactly equal to a generic type variable constrained by the same set of requirements, or a less strict set of requirements.

	An example:

	```swift
	protocol Fooable { 
		associatedtype FooAssoc
	}
	protocol Barrable { }
	protocol Bazzable { }

	struct Bag<U : Barrable, ...T : Fooable where T... : Barrable, T.Assoc... == U, U : Bazzable> {
		// populated somehow...
		var (...stuff) = (T...)

		func doSomething() -> Bool {
			return #fold(start: false, reducer: doerFunc, values: stuff...)
		}
	}

	// Type signature is "less strict" because it does not impose additional
	// requirements, and actually removes the transitive "U : Barrable" constraint.
	func doerFunc<T : Fooable where T.Assoc : Bazzable>(x: U, y: Bool) -> Bool {
		// ...
	}
	```

* If `T...` was a variadic generic type parameter, and was given constraints, the type of `#requirements` can be an existential constrained by the same set of requirements, or a supertype of that existential.

	The previous example, but reworked to use an existential rather than a generic:

	```swift
	protocol Fooable { 
		associatedtype FooAssoc
	}
	protocol Barrable { }
	protocol Bazzable { }

	struct Bag<U : Barrable, ...T : Fooable where T... : Barrable, T.Assoc... == U, U : Bazzable> {
		// populated somehow...
		var (...stuff) = (T...)

		func doSomething() -> Bool {
			return #fold(start: false, reducer: doerFunc, values: stuff...)
		}
	}

	// Type signature is "just as strict"
	// No generic parameter, but an existential parameter instead.
	func doerFunc(x: Any<Fooable where .Assoc : Barrable, .Assoc : Bazzable>, y: Bool) -> Bool {
		// ...
	}
	```

This provides a powerful, general, type-safe way to reduce a value vector to a scalar.

#### `#invert()`

One final construct is the `#invert()` compile-time feature. `#invert(v...)`, where `v...` is a value vector of parameter vector type `T?...`, produces a scalar tuple value of type `(T...)?`. This is useful for working with variadic expressions that generate optionals.

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
	var c = #invert(#vector(b)) // type: (G.Element...)?
}
```

*This is a terrible design hack. Any suggestions for a more elegant replacement, or even better naming, would be greatly appreciated.*

## Detailed design

### Zero-count parameter vectors

*TODO*: what to do about a generic type parameterized only on a generic vector, when that vector has zero types?

```swift
struct Foo<...Elements> { ... }

let a : Foo<> // Swift doesn't currently support generic types with 0 params
```

### Implementation

The most straightforward way to implement varidaic generics is through specialization at compile time. Generic parameter vectors can be typechecked using rules generalized from scalar generic type parameters:

```swift
struct Good : Fooable { ... }
struct Acceptable : Fooable { ... }
struct Fine : Fooable { ... }
struct Bad { ... }

struct Bag<...Element : Fooable> {
	// ...
}

// Okay
let a : Bag<Good, Fine>

// Okay
let b : Bag<Acceptable, Good, Good, Fine>

// NOT ALLOWED, since 'Bad' deoes not meet the constraint
// let c : Bag<Acceptable, Bad>
```

At this point, however many instantiations of the generic construct are necessary can be created. The name of a variadic type or function will be mangled in order to allow it to be instantiated multiple times. For example:

```swift
// Used for 'let a : Bag<Good, Fine>'
struct Bag_Element2<Element1 : Fooable, Element2 : Fooable> { ...}

// Used for 'let b : Bag<Acceptable, Good, Good, Fine>'
struct Bag_Element4<Element1 : Fooable, Element2 : Fooable, Element3 : Fooable, Element4 : Fooable> { ...}
```

All variadic-specific language constructs are expanded into tuples and/or collections of expressions or variables through the expansions described in the previous sections.

After instantiation is complete, references to a variadic construct can then be replaced with a reference to the non-variadic *n*-arity version of that construct. At this point the code can be compiled as usual.

#### Better solutions?

*If there is a superior alternative to the specialization-based solution described above, please propose it.*

## Case Studies

A few examples of how variadic generics could be used follow:

### Zip Sequences

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
	return function(#vector(arguments))
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
			buffer.append(f(#vector(result)))
		} else {
			// We're done, return the buffer
			return buffer
		}
	}
}

let results = multiMap(+, [1, 2, 3], [4, 5, 6])
// results = [5, 7, 9]
```

## Future directions

???


## Impact on existing code

No impact on existing code; this is a greenfield feature.

## Alternatives considered

Don't add this feature to the language.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.
