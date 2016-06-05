# Enums as generalized ADTs

* Proposal: [SE-NNNN](NNNN-filename.md)
* Author: [Austin Zheng](https://github.com/austinzheng)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

Swift's enums with associated values are the language's implementation of [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type), otherwise known as ADTs:

```swift
enum Either<T, U> {
	case left(T)
	case right(U)
}

let a : Either<Int, Bool> = .left(5)
let b : Either<Int, Bool> = .right(true)
```

Notice how `a` and `b` have the exact same type to an outside observer, `Either<Int, Bool>`. The concept of [*generalized algebraic data types*](https://en.wikipedia.org/wiki/Generalized_algebraic_data_type) generalizes the idea of ADTs in a way such that different cases on the same ADT can choose to differentiate themselves from each other via their type. This allows for more powerful abstractions which can be verified for correctness at compile-time thanks to the typechecker.

Swift-evolution thread: TBD

## Motivation

A normal ADT's different cases are indistinguishable from each other by their type. For example, the following `Expression` enum encodes expressions in a very simple programming language that can only represent integers, booleans, and a few different types of operations:

```swift
enum Either<T, U> { case .left(T); case .right(U) }

// An enum representing an expression in a very simple programming language.
// This language only supports integers, boolean values, adding, and checking
// two integers for equality.
enum Expression {
	case integer(Int)
	case boolean(Bool)
	
	// A sum expression is comprised of two integer-typed subexpressions. It
	// represents the sum of those two integer expressions' values.
	case sum(Expression, Expression)

	// An equals expression is comprised of two integer-typed subexpressions.
	// It represents a boolean value indicating whether the integers are equal.
	case eql(Expression, Expression)

	// An and expression is comprised of two boolean-typed subexpressions. It
	// represents a boolean value indicating whether both evaluate to true. 
	case and(Expression, Expression)

	// Return the evaluated result of an expression, either an integer or a
	// boolean value
	func evaluate() throws -> Either<Int, Bool> { /* ... */ }
}
```

The code listing above has a couple of problems. For example: a sum expression can only operate on two integer expressions, but there's no way to guarantee that we don't pass in expressions that evaluate to boolean type. To the typechecker, all expressions are simply `Expression`.

We can work around this issue through runtime type-checking:

```swift
struct ExprError : ErrorType {
	let message: String
	// ...
}

extension Expression {
	func evaluate() throws -> Either<Int, Bool> {
		switch self {
		case let .integer(v): return .left(v)
		case let .boolean(v): return .right(v)

		// Here is where things get interesting...
		case let .sum(lhs, rhs):
			guard case let .left(lhsInt) = lhs.evaluate() else {
				throw ExprError("expected an integer on the left, but got a boolean!")
			}
			guard case let .left(rhsInt) = rhs.evaluate() else {
				throw ExprError("expected an integer on the right, but got a boolean!")
			}
			return .left(lhsInt + rhsInt)

		// Likewise:
		case let .eql(lhs, rhs):
			/* ... */
		}
	}
}
```

However, this code is suboptimal for a variety of reasons:

* The sort of type errors the `guard case let` checks for are errors that we should be able to catch at compile-time. It would be better to find them there than to wait until our program throws an error at runtime.

* The missing type information is present, in a way: it's kludged into the `Either<T, U>` box that `evaluate()` returns. This becomes harder to deal with if we have more than two types in play. An alternative solution might be to declare an `ExpressionType` protocol and have `Int`, `Bool`, and any other valid expression type conform to that, but then we need runtime downcasting to do anything useful with our evaluation results.

An alternative solution might be to somehow encode type information into each enum constructor (enum case declaration). Having this additional per-case type information would allow us to encode restrictions on how a case can be used into the type system, allowing us more control over how our algebraic data types are being used:

```swift
enum Expression[_] {
	case integer(Int) -> Int
	case boolean(Bool) -> Bool
	case sum(Expression[Int], Expression[Int]) -> Int
	case mul(Expression[Int], Expression[Int]) -> Int
	case eql(Expression[Int], Expression[Int]) -> Bool
	case and(Expression[Bool], Expression[Bool]) -> Bool

	func evaluate() -> [_] { /* ... */ }
}

// Encode the expression:
// (10 == 20) && true
let a : Expression[Int] = .integer(10)
let b : Expression[Int] = .integer(20)
let c : Expression[Bool] = .boolean(true)
let areEqual : Expression[Bool] = .eql(a, b)
let andExpr : Expression[Bool] = .and(areEqual, c)
let finalResult : Bool = andExpr.evaluate()
```

This concept is known as *generalized algebraic data types* (GADTs), and is described in more detail below.

## Proposed solution

### Defining a GADT

An enum type which wishes to opt-in to GADT behavior can be declared as having one **phantom type parameter**. This takes the form of the production `[_]`, which follows the type name and the generic declaration (if one exists). (This means that GADT enums can be generic types in the standard Swift sense.)

Every case must be declared with an associated **phantom type** using the `-> SomeType` syntax following the case definition. Phantom types can be declared as any of the following:

* A concrete type (such as `Int`)
* A previously defined generic type parameter, or type parameterized on a generic type parameter

Example:

```swift
// Phantom type parameters are filled, in this case, by Int and Bool.
enum Expression[_] {
	case integer(Int) -> Int
	case boolean(Bool) -> Bool
	case add(Expression[Int], Expression[Int]) -> Int
	case mul(Expression[Int], Expression[Int]) -> Int
	case eql(Expression[Int], Expression[Int]) -> Bool
	case and(Expression[Bool], Expression[Bool]) -> Bool
}
```

In the above example, `.integer(...)` declares its phantom type to be `Int`, and therefore an instance of `.integer(...)` has the type `Expression[Int]`. See below for examples.

A case cannot build a GADT phantom type out of its own enclosing enum type, or produce a mutually recursive phantom type definition (for example, building its type out of another enum, whose cases in turn build their phantom types out of the first enum).

A case with associated values can take an associated value of either a specific phantom type (shown above as the `Expression[Int]` and `Expression[Bool]` arguments), or an associated value of any phantom type. In the second case the subscript brackets and type are omitted: `case .dynamic(Expression, Expression) -> Either<Int, Bool>`.

When instances of a GADT enum are declared, they can be parameterized on the phantom type parameter using square brackets. This parameterization is independent of any normal generic type parameters the enum may have. The parameterized phantom type must be equal to a type returned by at least one case. (If the type is omitted it is inferred.)

A variable of GADT enum type can only contain a GADT enum case whose phantom type matches the declared phantom type:

```swift
// Okay, because .integer(5) is parameterized on Int
let a : Expression[Int] = .integer(5)

// Okay, because .boolean(true) is parameterized on Bool
let b : Expression[Bool] = .boolean(true)

// Okay, because .eql is parameterized on Bool, and the two expression
// arguments it takes are correctly parameterized on Int
let c : Expression[Int] = .eql(a, a)
```

### Using a GADT

A function or type member which operates on a GADT enum may reference the phantom type parameter using the `[_]` syntax as either an argument type or a return value type. For any given case, the `[_]` can be thought of as a wildcard which is replaced with the case's phantom type.

```swift
extension Expression {
	func evaluate() -> [_] {
		// This method needs to return [_], which is whatever phantom type Self
		// was parameterized upon.
		switch self {
		// Typechecks because we get the raw integer out of .integer, and
		// .integer is parameterized on Int, thus the types match.
		case let .integer(v): return v

		// Typechecks because we get the raw boolean out of .boolean, and
		// .boolean is parameterized on Bool, thus the types match.
		case let .boolean(v): return v

		// Typechecks because .add is parameterized on Int, and u and v are
		// both parameterized on Int. This means u.evaluate() and v.evaluate()
		// are Ints, and we can add them.
		case let .add(u, v): return u.evaluate() + v.evaluate()

		// Likewise.
		case let .mul(u, v): return u.evaluate() * v.evaluate()

		// Typechecks because .eql is parameterized on Bool, and u and v are
		// both parameterized on Int. This means u.evaluate() and v.evaluate()
		// are Ints. We can then call == on the two integers and return a
		// boolean value.
		case let .eql(u, v): return u.evaluate() == v.evaluate()

		// Likewise
		case let .and(u, v): return u.evaluate() && v.evaluate()
		}
	}
}

// Usage:
let leftExpr, rightExpr : Expression[Int] = // ...
let a : Expression[Int] = .add(leftExpr, rightExpr)
let b : Expression[Bool] = .eql(leftExpr, rightExpr)

let result1 : Int = a.evaluate()
let result2 : Bool = b.evaluate()
```

Static enum members (e.g. static methods) are not allowed to use `[_]` anywhere in their declaration.

### Constrained GADT methods

Parameterized extensions should allow for methods on GADT enums constrained by the type of the phantom type. For example, here is a listing for a method which can only be called on cases whose phantom type is `Bool`:

```swift
extension Expression[Bool] {
	func inverseBooleanValue() -> [_] {
		// Okay, because [_] constrained to equal Bool
		let bv : Bool = self.evaluate()
		return !bv
	}
}

let x : Expression[Bool] = // ...
let y : Bool = x.inverseBooleanValue()

let a = .integer(5)
// Won't even compile
a.inverseBooleanValue()
```

In such a case switching on `self` should only allow patterns involving cases whose phantom types meet the extension's phantom type constraint. For example:

```swift
extension Expression[Bool] {
	func inverseBooleanValue() -> [_] {
		switch self {
		// We only need to switch on cases whose phantom types are Bool
		case let .boolean(v): return !v
		case let .eql(lhs, rhs): return lhs.evaluate() != rhs.evaluate()
		case let .and(lhs, rhs): return !(lhs.evaluate() && rhs.evaluate()) 
		}
	}
}
```

### GADTs and generics

The phantom type parameter may be referenced as part of a generic declaration. In this case, a generic type parameter that is previously declared is 'bound' to the value of the phantom type parameter by placing it in the square brackets, as shown below:

```swift
protocol DoubleConvertible {
	var doubleValue { get }
}

extension Int : DoubleConvertible { /* ... */ }

func getSums<T : DoubleConvertible>(exprs: [Expression[T]]) -> Double {
	var buffer : Double = 0
	for expression in exprs {
		buffer += exprs.evaluate().doubleValue
	}
	return buffer
}

let someExpressions : [Expression[Int]] = // ...

// Works, because Int conforms to DoubleConvertible
let doubleSum = getSums(someExpressions)
```

## Detailed design

### Invariance

Phantom types do not have to be mutually exclusive. However, GADTs are considered invariant relative to the phantom type. This is because phantom types may be used as both arguments to methods, as well as return values from methods. This can result in unsound behavior:

```swift
class Animal { /* ... */ }
class Dog : Animal { /* ... */ }
class Cat : Animal { /* ... */ }

let fido : Dog = // ...
let whiskers : Cat = // ...

enum MyEnum[_] {
	case animal(Animal) -> Animal
	case dog(Dog) -> Dog
	case cat(Cat) -> Cat
}

// Not allowed
let a : MyEnum[Animal] = .cat(whiskers)
```

Why is this a problem? Consider:

```swift
extension MyEnum {
	
	mutating func replaceInhabitant(inout with newCreature: [_]) {
		switch self {
		case .animal: self = .animal(newCreature)
		case .dog: self = .dog(newCreature)
		case .cat: self = .cat(newCreature)
		}
	}
}

let someAnimal : MyEnum[Animal] = .dog(fido)

// Typechecks, since whiskers is a Cat, Cat is a subclass of Animal.
// replaceInhabitant's argument is 'Animal', since that's what the phantom type
// was parameterized as when someAnimal was declared.
// However, doing so will cause 'self = .dog(whiskers)' to be invoked, which is
// a type error.
someAnimal.replaceInhabitant(with: whiskers)
```

### Usage of parameterized GADT enum cases

A parameterized GADT type (e.g. `Expression[Bool]`) is itself a proper type. It can be used as the value of a generic type parameter, the type of a property or local variable, as a function argument or return type, etc.

The metatype of the value of a parameterized GADT type is simply the enum's metatype:

```swift
let a : Expression[Bool] = // ...
let b : Expression.Type = a.self
```

This is okay, because the metatype can only be used to invoke static members (which cannot be parameterized on the phantom type), or used to construct an instance of the type with a specific case.

### Unparameterized cases

It should be possible to define a type-erased GADT type which can accept any enum case:

```swift
var expression : Expression = .mul(x, y)
expression = .integer(5)
expression = .boolean(true)
```

Such a value of nonparameterized type functions exactly like a normal enum. Any API which involves a phantom type is unavailable.

It is trivially possible to recover the parameterized type by switching upon the value of nonparameterized type:

```swift
var a : Expression = // ...

let b : Expression[Int]?

switch a {
	case .integer: b = a
	case .add: b = a
	case .mul: b = a
	default: b = nil
}

// do something with b...
```

## More example(s)

### 'Safe list'

Here is a code listing for a silly 'safe list' type, which models a [functional-style list](https://en.wikipedia.org/wiki/List_(abstract_data_type)#Abstract_definition) which encodes its empty/full state within its type.

```swift
// Two uninhabited types, used only to distinguish between empty and non-empty
// lists.
enum Empty { }
enum NonEmpty { }

enum SafeList<T>[_] {
	case none -> Empty
	// Second associated value can be SafeList with either Empty or NonEmpty
	case cons(T, SafeList<T>) -> NonEmpty
}

extension SafeList[NonEmpty] {
	func head() -> T {
		// Switch is exhaustive on just the .cons case
		switch self {
		case let .cons(head, _): return head
		}
	}
}

// Type: SafeList<Int>[Empty]
let emptyList = SafeList<Int>.none

// Type: SafeList<Int>[NonEmpty]
let nonEmptyList = SafeList<Int>.cons(5, .cons(4, .cons(3, emptyList)))

// returns 5, since it's a SafeList[NonEmpty]
// No optionals required!
let a = nonEmptyList.head()

// won't even compile, since it's a SafeList[Empty]
let b = emptyList.head()
```

## Impact on existing code

No impact. This is a greenfield feature.

## Alternatives considered

Don't add this feature.

Design variations follow:

### Implicit generic type parameter

It's possible that, instead of the `[_]` syntax, the GADT phantom type parameter could simply be expressed as an additional implicit generic parameter following any generic parameters already declared:

```swift
enum TypedBox<T> {
	case integer(Int) -> Int
	case boolean(Bool) -> Bool
	case generic(T) -> T
}

// Usage:
let a : TypedBox<Foo, Int> = .integer(1)
let b : TypedBox<Foo, Foo> = .generic(Foo())
```

The primary advantage of this syntax is the fact that it's no longer necessary to introduce any new productions to the generic syntax grammar (e.g. `[_]`).

The primary disadvantage of this syntax is that it is horrifyingly confusing at the site of use. There is no way for the user to immediately distingush whether `Foo` is a two-generic-argument normal ADT, or a one-generic-argument GADT.

As well, although phantom types are similar to generic type parameters in many ways, their behavior is very distinct (for example, it's the cases that 'specialize' the phantom type, not the caller), and it might be better not to conflate the two.

### Magic `Self.~` parameter

Rather than using `[_]`, perhaps GADTs can have an implicit type parameter defined on them. This parameter can be named something like `Self._`, `Self.~`, `Self.*`, `Self.Phantom`, or something similar:

```swift
enum TypedBox<T> {
	case integer(Int) -> Int
	case boolean(Bool) -> Bool
	case generic(T) -> T

	func someFunc() -> Self.~ { ... }
}

extension TypedBox where Self.~ == Int {
	func returnAnInt() -> Self.~ { ... }
}
```

The primary disadvantages of this syntax are:

* It's more 'magical' than `[_]`. With this technique, every GADT has an implicitly defined type associated with it (much like how `willSet` blocks have a `newValue` parameter by default). It's not immediately obvious what the magical type parameter is named, even if you do add phantom types to cases.
* It still leaves unanswered the question of how to parameterize the phantom type at the site of use (and for objections to just using the regular generic specialization syntax, see previous point).

### Less ugly syntax

Suggestions for more aesthetically pleasing syntax are welcome.
