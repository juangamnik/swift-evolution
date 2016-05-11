# `Data` type for standard library

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): [Austin Zheng](https://github.com/austinzheng)
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

This proposal describes the design of a native `Data` type for inclusion within Swift's standard library.


## Motivation

The Swift standard library currently makes no special effort to assist developers who wish to work with binary data. Similar functionality has traditionally been provided by the Foundation `NSData` and `NSMutableData` classes for the Objective-C platform. Passing around, examining, and manipulating binary data is a fundamental part of programs in many problem domains, and Swift should provide first-class support for working with binary data.


## Proposed solution

The Swift standard library provides an implementation of a `Data` type, used to represent a mutable or immutable data buffer.

`Data` is:

* A value type (struct) made efficient through the use of the copy-on-write machinery used by Swift's `Array`, `Set`, and `Dictionary` data structures.
* Natively conformant to Swift protocols for which it makes sense.
* Equipped with a basic API, with extended capabilities (such as file I/O) provided by Foundation's `NSData` API.
* Bridged to `NSData` in the same way Swift standard library and Foundation collection types are bridged.

The most basic implementation of `Data` provides a contiguous byte buffer and a length field. It is possible to use the same CoW machinery to implement a more sophisticated variant of `Data` which can support both contiguous buffers as well as non-contiguous storage via a tree containing references to multiple buffers, although doing so presents significant complications in terms of API and implementation. The detailed description of such a variant is beyond the scope of this document.


## Detailed design

Implementation should be quite straightforward. The `Data` struct is linked to a hidden `_DataBuffer` class which manages the memory used to store the data. `isUniquelyReferenced` is used to implement copy-on-write behavior.

`Data` defines a number of initializers which allow for the creation of new instances from a variety of sources. `var` instances can also be concatenated and extended using methods or operators similar in semantics to those belonging to other Swift data types.

```
struct Data {
	
	// Returns an empty Data object
	init()

	// Returns a Data object built out of base-64 encoded data
	init?(base64EncodedData base64Data: Data, ignoreUnknownCharacters: Bool = true)

	// Returns a Data object built out of a base-64 encoded string
	init?(base64EncodedString base64String: String, ignoreUnknownCharacters: Bool = true)

	// Returns a Data object comprised of bytes copied out of an implicit
	// buffer at a memory address
	init(bytes: UnsafePointer<Void>, length: Int)

	// Returns a Data object comprised of bytes copied out of an buffer pointer
	init(bytes: UnsafeBufferPointer<UInt8>)

	// Returns a Data object comprised of bytes copied out of a UInt8 array
	init(data: [UInt8])

	// Build a new data object out of a slice
	init(slice: DataSlice)

	// Build a new data object out of a base 64 view
	init(view: Data.Base64View)

	// Build a new data object out of an base 64 view slice
	init(slice: Data.Base64Slice)

	// Append a single byte to the buffer
	mutating func appendByte(_: UInt8)

	// Append the contents of a sequence containing bytes to the buffer
	mutating func appendContentsOf<S : SequenceType where S.Generator.Element == UInt8>(_: S)

	// Append the contents of another Data instance
	mutating func appendContentsOf(_: Data)

	// Append data directly from a buffer
	mutating func appendBytes(bytes: UnsafePointer<Void>, length: Int)

	// Whether or not this data object was initialized as base 64 data.
	var initializedAsBase64Data : Bool { get }
}

// Concatenate data objects
func +<S : SequenceType where S.Generator.Element == UInt8>(lhs: Data, rhs: S) -> Bool

// Append data to another data
func +=<S : SequenceType where S.Generator.Element == UInt8>(inout lhs: Data, rhs: S) -> Bool
```

`Data` can export its contents as a base-64 encoded string, or as a base-64 encoded data blob. It also exposes buffer pointers allowing consumers direct low-level manipulation of its contents.

```
extension Data {
	enum LineLength { case NoLimit, UpTo64, UpTo76 }

	struct EndLineOptions : RawOptionSetType {
		static var CarriageReturn : EndLineOptions
		static var LineFeed : EndLineOptions
	}

	func base64EncodedData(lineLength: LineLength = .NoLimit, endLine: EndLineOptions = []) -> Data
	func base64EncodedString(lineLength: LineLength = .NoLimit, endLine: EndLineOptions = []) -> String

	// NOTE: These won't work if Data is allowed to be non-contiguous in the future...
	func bufferPointer() -> UnsafeBufferPointer<UInt8>
	mutating func mutableBufferPointer() -> UnsafeMutableBufferPointer<UInt8>
}
```

`Data` is modeled as a random access collection of `UInt8` values representing individual bytes. Since each element is constant size, O(1) random access should be possible. (If non-contiguous buffers are implemented this increases to O(log n), depending on implementation.) This makes `Data`'s conformance implementation considerably more straightforward than that of (e.g.) `String`'s views.

`Data` also provides a view of the contents as characters representing the base-64 encoded contents.

```
extension Data : RandomAccessCollection {
	var length : Int { get }
	subscript(position: Int) -> UInt8
	// ...

	struct DataSlice {
		// A slice into a Data instance 
		// ...
	}
}

extension Data {
	struct Base64View : RandomAccessCollection {
		var length : Int { get } // Note: not the same as Data.length
		subscript(position: Int) -> Character
		// ...
	}
	struct Base64Slice {
		// A slice into the Base64View of a Data instance
		// ...
	}
	// nil if initializedAsBase64Data == true (?)
	var base64View : Base64View? { get }
}
```

`Data` should be hashable, equatable, and provide its own pretty-printed representation.

```
extension Data : CustomStringConvertible {
	// A human-readable description of the data contents, with each byte
	// represented by a hexadecimal value. This description may be truncated to
	// the first X bytes.
	var description : String 
}

extension Data : Hashable {
	var hashValue : Int
}

// Check equality of two data objects
func ==(lhs: Data, rhs: Data) -> Bool

```

## Impact on existing code

No direct impact, this is an additive feature. Code that is using Foundation's `NSData` may wish to implement support for such a `Data` type.


## Alternatives considered

The straightforward alternative is allowing Foundation to provide `NSData` as Swift's sole means of higher-level binary data representation and manipulation.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.
