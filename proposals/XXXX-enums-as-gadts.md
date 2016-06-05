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
indirect enum Expression {
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

An better solution might be to somehow encode type information into each enum constructor (enum case declaration). Having this additional per-case type information would allow us to encode restrictions on how a case can be used into the type system, allowing us more control over how our algebraic data types are being used:

```swift
indirect enum Expression {
	phantomtype E

	case integer(Int) -> Self<Int>
	case boolean(Bool) -> Self<Bool>
	case sum(Expression<Int>, Expression<Int>) -> Self<Int>
	case mul(Expression<Int>, Expression<Int>) -> Self<Int>
	case eql(Expression<Int>, Expression<Int>) -> Self<Bool>
	case and(Expression<Bool>, Expression<Bool>) -> Self<Bool>

	func evaluate() -> E { /* ... */ }
}

// Encode the expression:
// (10 == 20) && true
let a : Expression<Int> = .integer(10)
let b : Expression<Int> = .integer(20)
let c : Expression<Bool> = .boolean(true)
let areEqual : Expression<Bool> = .eql(a, b)
let andExpr : Expression<Bool> = .and(areEqual, c)
let finalResult : Bool = andExpr.evaluate()
```

This concept is known as *generalized algebraic data types* (GADTs), and is described in more detail below.

## Proposed solution

### Defining a GADT

An enum type which wishes to opt-in to GADT behavior must fufill two requirements:

* Each case must be given a **case-specific type signature**, which describes the type of the GADT case. (This means that, unlike non-GADT enums, all cases do not share the same type.) It is an error to give only some cases case-specific type signatures. Every case-specific type signature must have the same number of generic type parameters.

* There must be at least one type argument in the case-specific type signature. The last argument in the signature is what is known as the **phantom type parameter**, populated by a **phantom type**.

Here is an example:

Example:

```swift
// Phantom type parameters are filled, in this case, by Int and Bool.
indirect enum Expression {
	case integer(Int) -> Self<Int>
	case boolean(Bool) -> Self<Bool>
	case add(Expression<Int>, Expression<Int>) -> Self<Int>
	case mul(Expression<Int>, Expression<Int>) -> Self<Int>
	case eql(Expression<Int>, Expression<Int>) -> Self<Bool>
	case and(Expression<Bool>, Expression<Bool>) -> Self<Bool>
}
```

The **case-specific type signature** follows each case declaration, and is separated from the case declaration by the `->`. This type signature takes the form of `Self<...>`, where `...` are replaced by one or more type arguments. If the enum is a generic type, those generic types may be used as part of the type signature.

The **phantom type parameter** is the final type argument in the case-specific type signature. In the `Expression` example, every case has either a **phantom type** of `Int` or `Bool`. More generally, a phantom type can be declared as any of the following:

1. A concrete type (such as `Int`)
2. A previously defined generic type parameter, or type parameterized on a generic type parameter (e.g. `T`, for an `enum MyEnum<T> { ... }`)
3. An associated type associated with #2.

A case cannot build a phantom type out of its own enclosing enum type, or produce a mutually recursive phantom type definition (for example, building its type out of another enum, whose cases in turn build their phantom types out of the first enum). So the following is invalid:

```swift
enum Expression {
	// Can't build phantom type out of the enclosing type
	case invalid -> Self<Expression<Int>>
}
```

The case-specific type signature for a case determines how instances of that case are typed. Note the following examples: `Self<Int>` becomes `Expression<Int>`, and `Self<Bool>` becomes `Expression<Bool>`:

```swift
// Okay, because .integer(5) is parameterized on Int
// Note that .integer's type signature is declared as 'Self<Int>'
let a : Expression<Int> = .integer(5)

// Okay, because .boolean(true) is parameterized on Bool
// Note that .boolean's type signature is declared as 'Self<Bool>'
let b : Expression<Bool> = .boolean(true)

// Okay, because .eql is parameterized on Bool, and the two expression
// arguments it takes are correctly parameterized on Int
// Note that .eql's type signature is declared as 'Self<Int>'
let c : Expression<Int> = .eql(a, a)
```

### Generic and type-erased GADTs

In addition to generic parameters declared in the type's declaration, each GADT enum case is allowed to declare its own generic type signature containing additional type parameters. Enum cases may declare a generic type signature even if the enum type itself does not. If an enum case does not declare a generic type signature it still has access to any generic type parameters declared by the enclosing type (and can, for example, constrain against those parameters' associated types, etc).

```swift
enum MyEnum<T> {
	// Uses the type parameters declared by MyEnum
	case first(T) -> Self<T>

	// Uses T, but also declares another type parameter U
	case second<U>(T, U) -> Self<T>

	case third<U>(U, Int) -> Self<Int>
}
```

Case-specific type signatures are not allowed to utilize generic parameters declared within a case's generic type signature:

```swift
enum MyEnum<T> {
	// NOT ALLOWED; U was declared in the case's generic type signature, so it
	// cannot be used in Self<...>
	case first<U>(T, U) -> Self<(T, U)>
}
```

It is an error to shadow a generic type parameter that was already declared:

```swift
enum MyEnum<T> {
	// NOT ALLOWED; cannot redeclare T
	case first<T, U>(T, U) -> Self<T>
	// ...
}
```

Note that, in some of the examples above, additional type parameters were introduced (and are available when constructing an instance), but are 'lost'. Such concrete types can be recovered later through pattern matching (see *Extracting types from type-erased cases*).

### Using a GADT

If the phantom type parameter must be reified (for example, a function or type member that operates on a GADT enum needs to reference it), the `phantomtype` keyword can be used to name the phantom type parameter. Only one name can be given to a phantom type per class. No constraints can be declared on the type:

```swift
enum Expression {
	phantomtype ExprType

	// We can now refer to Expression.ExprType, or just ExprType in the scope
	// of the type definition.
}
```

Once this is done, it is possible to write a method that is parameterized on the phantom type. The following example demonstrates an `evaluate()` method, which returns an instance of whatever phantom type the 'self' case is associated with (so, either an `Int` or a `Bool`):

```swift
extension Expression {
	func evaluate() -> ExprType {
		// This method needs to return ExprType, which is whatever phantom type Self
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
let leftExpr, rightExpr : Expression<Int> = // ...
let a : Expression<Int> = .add(leftExpr, rightExpr)
let b : Expression<Bool> = .eql(leftExpr, rightExpr)

let result1 : Int = a.evaluate()
let result2 : Bool = b.evaluate()
```

Static enum members (e.g. static methods) are not allowed to use the phantom type anywhere in their declaration.

### Constrained GADT methods

Parameterized extensions should allow for methods on GADTs constrained by the type of the phantom type. For example, here is a listing for a method which can only be called on cases whose phantom type is `Bool`:

```swift
extension Expression where ExprType == Bool {
	func inverseBooleanValue() -> ExprType {
		// Okay, because phantom type (ExprType) constrained to equal Bool
		let bv : Bool = self.evaluate()
		return !bv
	}
}

let x : Expression<Bool> = // ...
let y : Bool = x.inverseBooleanValue()

let a = .integer(5)
// Won't even compile
a.inverseBooleanValue()
```

In such a case switching on `self` should only allow patterns involving cases whose phantom types meet the extension's phantom type constraint. For example:

```swift
extension Expression where ExprType == Bool {
	func inverseBooleanValue() -> ExprType {
		switch self {
		// We only need to switch on cases whose phantom types are Bool
		case let .boolean(v): return !v
		case let .eql(lhs, rhs): return lhs.evaluate() != rhs.evaluate()
		case let .and(lhs, rhs): return !(lhs.evaluate() && rhs.evaluate()) 
		}
	}
}
```

If a phantom type can be either a concrete type (like `Bool`) or a generic type variable (like `T`), such a constrained extension will require a switch statement to also handle any cases whose phantom types are generic:

```swift
enum MyEnum<T> {
	phantomtype PhantomType

	case first(Bool) -> Self<Bool>
	case second(Int) -> Self<Int>
	case third(T) -> Self<T>
}

extension MyEnum where PhantomType == Bool {
	func doSomething() {
		switch self {
		case .first: // do something
		case .third: // also need this case
		}
	}
}
```

## Detailed design

### GADTs and generics

The phantom type parameter may be referenced as part of a generic type or function declaration simply by treating it as the last generic parameter that the GADT type takes:

```swift
protocol DoubleConvertible {
	var doubleValue { get }
}

extension Int : DoubleConvertible { /* ... */ }

// 'T' serves the role as the phantom type parameter
func getSums<T : DoubleConvertible>(exprs: [Expression<T>]) -> Double {
	var buffer : Double = 0
	for expression in exprs {
		buffer += exprs.evaluate().doubleValue
	}
	return buffer
}

let someExpressions : [Expression<Int>] = // ...

// Works, because Int conforms to DoubleConvertible
let doubleSum = getSums(someExpressions)
```

### Extracting types from type-erased cases

Individual cases, as mentioned earlier, are allowed to define additional generic type parameters to parameterize any associated values they might contain. However, these types are generally 'hidden' or 'lost' as soon as an instance of the case is instantiated:

```swift
// A type that represents a pair of values.
// There are three variants of pairs:
//   - 'cell', for storing one value whose type is known and one value whose
//     type isn't known
//   - 'homogenous', for storing two values of the same, known type 'T'
//   - 'anyduplicate', for storing two values of some unknown type
// In all cases, the 'unknown' types aren't stored as part of the GADT's type,
// and must be recovered at runtime if they are desired.
enum SingleGenericParamPair<T> {
	// Type-erased heterogenous storage cell
	case cell<T, U>(T, U) -> Self<T>

	// Homogenous storage cell
	case homogenous(T, T) -> Self<T>

	// Type-erased homogenous storage cell
	case anyduplicate(U, U) -> Self<()>
}

let x : SingleGenericParamPair.cell(5, "hello world")

// The type of x is "SingleGenericParamPair<Int>"
// We've lost the type 'String' associated with U
```

Such types can be recovered through pattern matching and dynamic casting:

```swift
let x : SingleGenericParamPair<Int> = ...

// We only want a sum if both items in the pair are integers 
let sum : Int?

switch x {
case let .cell(x, y):
	// x is known to be Int, but y has to be opened first
	if let y = y as? Int {
		sum = x + y
	} else {
		sum = nil
	}

case let .homogenous(x, y):
	// Both x and y are known to be Int
	// No dynamic casting necessary!
	let sum = x + y

case let .anyduplicate(x, y):
	// Both x and y have to be opened first, since they are unknown
	if let x = x as? Int, y = y as? Int {
		sum = x + y
	} else {
		sum = nil
	}
}
```

### Invariance

Phantom types do not have to be mutually exclusive. However, GADTs are considered invariant relative to the phantom type. This is because phantom types may be used as both arguments to methods, as well as return values from methods. This can result in unsound behavior:

```swift
class Animal { /* ... */ }
class Dog : Animal { /* ... */ }
class Cat : Animal { /* ... */ }

let fido : Dog = // ...
let whiskers : Cat = // ...

enum MyEnum {
	phantomtype AnimalType
	case animal(Animal) -> Animal
	case dog(Dog) -> Dog
	case cat(Cat) -> Cat
}

// Not allowed
let a : MyEnum<Animal> = .cat(whiskers)
```

Why is this a problem? Consider:

```swift
extension MyEnum {
	
	mutating func replaceInhabitant(inout with newCreature: AnimalType) {
		switch self {
		case .animal: self = .animal(newCreature)
		case .dog: self = .dog(newCreature)
		case .cat: self = .cat(newCreature)
		}
	}
}

let someAnimal : MyEnum<Animal> = .dog(fido)

// Typechecks, since whiskers is a Cat, Cat is a subclass of Animal.
// replaceInhabitant's argument is 'Animal', since that's what the phantom type
// was parameterized as when someAnimal was declared.
// However, doing so will cause 'self = .dog(whiskers)' to be invoked, which is
// a type error.
someAnimal.replaceInhabitant(with: whiskers)
```

### Usage of parameterized GADT enum cases

A parameterized GADT type (e.g. `Expression<Bool>`) is itself a proper type. It can be used as the value of a generic type parameter, the type of a property or local variable, as a function argument or return type, etc.

The metatype of the value of a parameterized GADT type is simply the enum's metatype:

```swift
let a : Expression<Bool> = // ...
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

Such a value of nonparameterized type functions exactly like a normal enum. Any API which involves a phantom type is unavailable. The generic arity of such a type is one less than the arity specified in the case-specific type signatures.

It is trivially possible to recover the parameterized type by switching upon the value of nonparameterized type:

```swift
var a : Expression = // ...

let b : Expression<Int>?

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

indirect enum SafeList<T> {
	phantomtype EmptyState
	case none -> Empty
	// Second associated value can be SafeList with either Empty or NonEmpty
	case cons(T, SafeList<T>) -> NonEmpty
}

extension SafeList where EmptyState == NonEmpty {
	func head() -> T {
		// Switch is exhaustive on just the .cons case
		switch self {
		case let .cons(head, _): return head
		}
	}
}

// Type: SafeList<Int, Empty>
let emptyList = SafeList<Int>.none

// Type: SafeList<Int, NonEmpty>
let nonEmptyList = SafeList<Int>.cons(5, .cons(4, .cons(3, emptyList)))

// returns 5, since it's a SafeList<Int, NonEmpty>
// No optionals required!
let a = nonEmptyList.head()

// won't even compile, since it's a SafeList<Int, Empty>
let b = emptyList.head()
```

## Impact on existing code

No impact. This is a greenfield feature.

## Alternatives considered

### Use `associatedtype` instead of `phantomtype`

Instead of introducing a new keyword, the `associatedtype` keyword could be repurposed to serve as a way to name the phantom type.

Advantages:

* No new keywords
* Phantom types are vaguely analogous to associated types in that specific 'subtypes' constrain them to be a concrete type

Disadvantages:

* Syntactic differences: can only declare one `associatedtype`; can't constrain to protocols
* Semantic differences: phantom types and associated types are different (if similar) concepts; should not be conflated


### Less ugly syntax

Suggestions for more aesthetically pleasing syntax are welcome.
