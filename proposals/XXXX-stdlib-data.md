# `Data` type for standard library

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Authors: [Austin Zheng](https://github.com/austinzheng), [Dmitri Gribenko](https://github.com/gribozavr), TBD
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

This proposal describes the design of a native set of `Data` types for inclusion within Swift's standard library.


## Motivation

The Swift standard library currently makes no special effort to assist developers who wish to work with binary data. Similar functionality has traditionally been provided by the Foundation `NSData` and `NSMutableData` classes for the Objective-C platform. Passing around, examining, and manipulating binary data is a fundamental part of programs in many problem domains, and Swift should provide first-class support for working with binary data.


## Proposed solution

The Swift standard library should provide three abstractions to facilitate working with data:

* A `Data` protocol providing an abstract interface modeling a data object as a collection of bytes. Types that conform to `Data` can choose to model themselves explicitly as `Collection`s; otherwise, several view types provide a collection interface to various representations of the underlying data.
* An extension to `Array<UInt8>` allowing an array to model a contiguous byte buffer, conforming to `Data`. This can be referred to by the generic typealias `DataArray`.
* A `DataContainer` value type which provides contiguous or non-contiguous storage, conforming to `Data`, and also conforming to `Collection` and related protocols in its own right.

In the future, `NSData` and/or `dispatch_data_t` can be conformed to `Data`, but doing so is beyond the scope of this proposal.


## Detailed design

### `Data` protocol

The `Data` protocol allows a conforming type to model itself as a data object containing a collection of bytes.

Default implementations should be provided for all the method requirements defined by the `Data` protocol, unless noted. The two concrete implementations may provide specialized implementations for improved performance.

```
protocol Data : Hashable, CustomStringConvertible {

	// -- Initializers -- (no default implementations)

	// Instantiate data from a sequence of integer values.
	init<T: SequenceType where T.Generator.Element == IntegerType>(_: T, additionalCapacity: Int = 0)

	// Instantiate data from a base-64 encoded string.
	init?(base64String: String, ignoreUnknownCharacters: Bool = true, additionalCapacity: Int = 0)

	// Instantiate data from a string of hex tuples.
	init?(hexTupleString: String, ignoreUnknownCharacters: Bool = true, additionalCapacity: Int = 0)

	// Instantiate data comprised of bytes copied out of an implicit buffer at
	// a memory address.
	init(copiedBytes: UnsafePointer<Void>, length: Int, additionalCapacity: Int = 0)

	// Instantiate data comprised of bytes copied out of an buffer pointer.
	init(copiedBytes: UnsafeBufferPointer<Void>, additionalCapacity: Int = 0)


	// -- Pointer operations --

	// Copy a number of bytes out of the data object, starting from the
	// beginning. Throws if out of range.
	func copyBytesTo(buffer: UnsafeMutablePointer<Void>, count: Int) throws

	// Copy a range of bytes out of the data object based on a range. Throws if
	// the range is invalid.
	func copyBytesTo(buffer: UnsafeMutablePointer<Void>, range: Range<Int>) throws

	// Copy out a number of bytes from the pointer and append them to the data
	// instance.
	func appendBytesFrom(buffer: UnsafePointer<Void>, count: Int)

	// Replace existing bytes within a given range with bytes copied out of a
	// buffer. Optionally, specify a replacement length which differs from the
	// length of the replacement range. Throws if out of range.
	func replaceBytes(range: Range<Int>, replacementBytes: UnsafePointer<Void>, replacementCount: Int? = nil) throws

	// Return a collection of pointers and ranges corresponding to each buffer
	// within the data object. No default implementation.
	func byteRanges() -> [(UnsafePointer<Void>, Range<Int>)]


	// -- Data operations --

	// Append another data object to the end of this one.
	mutating func append(_: Data)

	// Reset part or all of the data object to a certain value.
	mutating func reset(valueToWrite: UInt8 = 0, range: Range<Int>? = nil)

	// Subscript getter/setters for data. Setter allows data within a given
	// range to be replaced by different data. Unlike the pointer operations
	// above, the subscript getters/setters trap if an out-of-range exception
	// occurs.
	subscript(range: Range<Int>) -> Data { get, set }
	subscript(range: Range<Int>, replacementCount replacement: Int) -> Data { get, set }

	// Concatenation of two Data objects
	func +(lhs: Data, rhs: Data) -> Data
	
	// In-place concatenation of two Data objects
	func +=(inout lhs: Data, rhs: Data)


	// -- Pseudo-collection behavior --
	// See note on conformance to `Collection` below
	// No default implementations; designed so that Array[Int8] will mostly
	// fulfill them automatically.
	// Functionality for backing the views' access to the underlying data.

	// Get or set a single byte.
	subscript(i: Int) -> UInt8 { get, set }

	// The number of bytes contained within this instance.
	var length : Int { get }

	var startIndex : Int { get }
	var endIndex : Int { get }


	// -- Views --

	var signedByteView : DataInt8View { get }
	var unsignedByteView : DataUInt8View { get }
	var hexTupleView : DataHexTupleView { get }
	var base64View : DataBase64View { get }	
}
```

A number of views on a data object are also defined. Data can be directly manipulated in the form of signed and unsigned bytes, and can be read as hex tuples or base-64 encoded characters. (Implementation and API may change based on what enhanced generics features make it into Swift 3.0.)

**NOTE**: Ideally, `Data` should inherit from `RandomAccessCollection` and force the associated type `Generator.Element` to be `UInt8`. This turns a `Data` instance into a collection of `UInt8`s by default, greatly simplifying parts of the API. If this does become possible, then `DataUInt8View` becomes unnecessary. However, doing so may make it difficult to conform `[Int8]` to `Data`, if this should be desirable.

**NOTE**: Ideally, Swift should allow nested types inside a protocol definition. If that becomes possible, the views below should lose their `Data` prefixes and become nested types within the `Data` protocol.

**TODO**: What is the best way to handle slicing? `DataArray` can use the default `ArraySlice`, or perhaps a `DataSlice` protocol that `ArraySlice<UInt8>` can conditionally conform to. In the latter case, a private `DataContainerSlice` that conforms to `DataSlice` can be provided.


```
// A view of the data as a collection of unsigned bytes
struct DataUInt8View : RandomAccessCollection, MutableCollection, RangeReplaceableCollection {
	init(_ data: Data)
	subscript(i: Int) -> UInt8

	mutating func append(_: UInt8)
	mutating func appendContentsOf<S : SequenceType where S.Generator.Element == UInt8>(_: S)
	// ...
}

// A view of the data as a collection of signed bytes
struct DataInt8View : RandomAccessCollection, MutableCollection, RangeReplaceableCollection {
	init(_ data: Data)
	subscript(i: Int) -> Int8

	mutating func append(_: Int8)
	mutating func appendContentsOf<S : SequenceType where S.Generator.Element == Int8>(_: S)
	// ...
}

// A read-only view of the data as a two-character hex tuple (e.g. "5F")
struct DataHexTupleView : RandomAccessCollection {
	init(_ data: Data)
	subscript(i: Int) -> String   // two-character string

	// ...
}

// A read-only view of the data in base-64 encoding
struct DataBase64View : RandomAccessCollection {
	init(_ data: Data)
	subscript(i: Int) -> Character

	// ...
}
```

Finally, `Data` might possibly also define a number of static utility methods that may be of interest to users working with various representations of binary/byte data:

```
extension Data {
	static func unsignedBytesFromSignedBytes<T: Collection where T.Generator.Element == Int8>(_: T) -> [UInt8]
	static func signedBytesFromUnsignedBytes<T: Collection where T.Generator.Element == UInt8>(_: T) -> [Int8]
	static func hexStringFromUnsignedBytes<T: Collection where T.Generator.Element == UInt8>(_: T) -> String
	static func unsignedBytesFromHexString(_: String) -> [UInt8]?
	// ... (other methods for other permutations)
}
```


### Extension on `Array<UInt8>`

`DataArray` provides the functionality of a contiguous byte buffer in other languages, and also provides all the existing capabilities of Swift's `Array` type.

**NOTE**: It's possible we may want separate conformances for `[UInt8]` and `[Int8]`. Generalizing the provided API for signed bytes is straightforward.

**NOTE**: One possible feature that `DataArray` may provide is the option to create an unsafe `Array` which is backed directly by a buffer pointer. See notes in the dummy comments below.

```
typealias DataArray = [UInt8]

extension Array<UInt8> : Data {
	
	// Instantiate an empty DataArray
	init(initialCapacity: Int? = nil)

	// Instantiate a DataArray from a DataContainer. If the DataContainer is
	// noncontiguous, its contents are copied into a contiguous buffer.
	init(_ dataContainer: DataContainer, additionalCapacity: Int = 0)

	// Instantiate an unsafe DataArray that is backed directly by an unsafe
	// buffer that has been previously initialized. Note that this requires
	// user care in two ways:
	// * The user must call the second init if they intend on writing to the
	//   punned array.
	// * The user must take care to not call invalid methods, including those
	//   that resize the array.
	// According to Dmitri Gribenko, this is a commonly-requested feature. It
	// seems dangerous enough, though, that its inclusion should be carefully
	// considered.
	init(underlyingBuffer: UnsafeBufferPointer<Void>)
	init(underlyingMutableBuffer: UnsafeMutableBufferPointer<Void>)
}
```


### `DataContainer` struct

The `DataContainer` struct provides contiguous or non-contiguous storage of data for applications where performing append operations without copying (or other similar tasks) is important for performance. Its design is based off `libdispatch`'s `dispatch_data_t` abstraction.

Conceptually, `DataContainer` is backed by one or more `_ArrayBuffer<UInt8>`s. In a similar manner to `dispatch_data_t`, `DataContainer` can be implemented as a list of one or more references to array buffers, and associated metadata. This has two advantages:

* The copy-on-write machinery used by `Array` can be trivially reused for implenting efficient copy-on-write behavior for `DataContainer`. As long as the invariant that the underlying storage class is only written to if owned by exactly one `Array` or `DataContainer`, correctness is preserved.

* Shared representation with `DataArray` allows common operations involving the two types to be efficiently implemented. For example, appending a `DataArray`'s contents to the end of a `DataContainer` should require little more than creating an additional reference to an existing underlying buffer.

```
struct DataContainer : Data {
	
	// Instantiate an empty DataContainer
	init(initialCapacity: Int? = nil)

	// Instantiate a DataContainer from a DataArray.
	init(_ dataArray: DataArray, additionalCapacity: Int = 0)

	// Instantiate a DataContainer from another DataContainer, possibly
	// flattening multiple buffers into a single contiguous buffer.
	init(_ dataContainer: DataContainer, flatten: Bool = false, additionalCapacity: Int = 0)

	// Whether or not the DataContainer is represented by a single underlying
	// buffer or not.
	var isContiguous : Bool { get }

	// The number of buffers the DataContainer owns.
	var bufferCount : Int { get }

	// Flatten the DataContainer if using non-contiguous storage.
	mutating func flatten()
}
```

In addition to having the same views that all `Data` types have, `DataContainer` will itself conform to `RandomAccessCollection`, with an element type of `UInt8` and an index type of `Int`. This is a minor optimization for convenience that ensures that both concrete implementations of `Data` in the standard library can be used directly as collections of bytes, even if this constraint is not possible to directly express in the type system.


## Impact on existing code

No direct impact, this is an additive feature. Code that is using Foundation's `NSData` may wish to implement support for such a `Data` type.


## Alternatives considered

The straightforward alternative is allowing Foundation to provide `NSData` as Swift's sole means of higher-level binary data representation and manipulation.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.
