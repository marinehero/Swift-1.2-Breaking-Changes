# Swift-1.2-Breaking-Changes

Swift 1.2 Xcode 6.3 breaking changes

## Changes to the Swift Standard Library in 1.2 beta 1 ##

Swift v1.2 is out, and, well, it’s pretty cool actually.

### Beefed-up syntax ###

The most exciting part for me is the beefed-up if let syntax. Not only does it allow you to unwrap multiple optionals at once, you can define a later let in terms of an earlier one. For instance, here are 3 functions, each one of which takes a non-optional and returns an optional, chained together:

    if  let url = NSURL(string: urlString), 
	      let data = NSData(contentsOfURL: url)
	    , let image = UIImage(data: data) where image.size > 1070 { 
	    // display image 
	 }

The key here is that if at any of the preceding lets fail, the whole thing fails and the later functions are not called. If you’re already familiar with optional chaining with ?. this should come naturally to you. For more on this, see [NSHipster’s article](http://nshipster.com/swift-1.2/).

The new ability to defer initialization of let constants makes much of of “avoiding var” arguments redundant, as you can now do this:

    let direction: Direction    
    switch (char) {
      case "a": direction = .Left
      case "s": direction = .Right
      default:  direction = .Forward
    }

It’d still be nice to have a form of switch that evaluates to a value – but with the new let, that’s really just a desire for some syntactic sugar (imagine using it inside a short closure expression, without needing any return).

Speaking of switch, the new if, with the ability to have a where clause on the bound variables, means if may now be suitable for several places where you were previously using switch. But there’s still a place for switch – especially when using it for full coverage of each case of an enum. And I feel a bit like pattern matching via ~= is one of switch’s powerful overlooked features that I keep meaning to write a post about. This release adds a ~= operator for ranges, not just intervals.

## Library Additions ##

The big new addition to the standard library is of course the Set collection type. The focus on mathematical set operations is very nice, and also that those operations take any SequenceType, not just another Set:

    let alphabet = Set("abcdefghijklmnopqrstuvwxyz")
    let consonants = alphabet.subtract("aeiou")
    // don't forget, it's "jumps" not "jumped"
    let foxy = "the quick brown fox jumped over the lazy dog"
    alphabet.isSubsetOf(foxy)  // false

Note that the naming conventions emphasize the pure versions of these operations, which create new sets, with each operation having a mutating equivalent with an InPlace suffix:

    // create a new set
    var newalpha = consonants.union("aeiou")
    // modify an existing set
    newalpha.subtractInPlace(consonants)

Note also for those fearful of the evil of custom operators, there’s no - for set subtraction etc. The one operator defined for sets is ==, though just like with Array, don’t forget that this doesn’t mean they’re Equatable.2

## Random ## 

But that’s not all. Here’s a rundown of some of the other library changes:

- There‘s no longer a separate countElements call for counting the size of collections vs count for ranges – it’s now just count for both.

- Dictionary‘s index, which used to be bi-directional, is now forward-only, as are the lazy collections you get back from keys and values.

- The C_ARGC and C_ARGV variables are gone, as is the _Process protocol. In their place there’s now a Process enum with 3 static methods: arguments, which gives you a [String] of the arguments, and argc and argv with give you the raw C-string versions.
Protocols that used to declare class funcs now declare static funcs since the language now requires that.

- Array, ClosedInterval and Slice now conform to a new protocol, _DestructorSafeContainer. This protocol indicates “a container is destructor safe if whether it may store to memory on destruction only depends on its type parameters.” i.e. an array of Int is known never to allocate memory when its destroyed since Int doesn’t, whereas a more complex type (with a deinit, or aggregating a type with one) may not be.

- They also now conform to a new __ArrayType protocol. Two underscores presumably means you definitely definitely shouldn’t try and conform to this directly yourself.

- The builtin-converting Array.convertFromHeapArray function is gone.

- The pointer types (AutoreleasingUnsafeMutablePointer, UnsafePointer and UnsafeMutablePointer now conform to a new _PointerType protocol.

- The CVaListPointer (the equivalent of C’s va_list) now has an init from an unsafe pointer.

- EmptyGenerator now has a public initializer, so you can use it directly.

- The various unicode types now have links in their docs to the relevant parts of unicode.org’s glossary
HeapBuffer and HeapBufferStorage are gone, replaced by the more comprehensive and documented ManagedBuffer and ManagedBufferPointer, of which more later.

- ObjectIdentifier is now Comparable.

- The enigmatic OnHeap struct is gone.

- All the views (UnicodeScalarView, UTF8View and UTF8View) inside String are now Printable and DebugPrintable

- String.lowercaseString and uppercaseString are back! I missed you guys.

- The various convertFrom static functions in StringInterpolationConvertible are now inits.

- UnsafePointer and UnsafeMutablePointer no longer have a static null() instance, presumably you should just use the nil literal.

- There’s a new zip function, which returns a Zip2 struct – consistent with the lazy function that returns LazyThings, and possibly a gateway to ZipMoreThan2 structs in the future…

- The Printable and DebugPrintable protocol docs now point out: if you want a string representation of something, use the toString protocol – in case you were trying to get fancy with if let x as? Printable { use(x.description) }. Which the language will now let you do, since as? and is now work with Swift protocols, which is cool. But still don’t do that – especially considering you may be sorry when you find out String doesn’t conform to Printable.

- The Character type is now a struct, not an enum, and no longer exposes a “small representation” and “large representation”. This appears to go along with a performance-improving refactoring that has made a big difference – for example, a simple bit of code that sucks in /usr/share/dict/words into a Swift String and counts the word lengths now runs in a quarter of the time it used to.

## Bovine Husbandry ##

Suppose you wanted to write your very own collection, similar to Array, but managing all the memory yourself. Up until Swift 1.2, this was tricky to do.

The memory allocation part isn’t too hard – UnsafeMutablePointer makes it pretty straightforward (and typesafe) to allocate memory, create and destroy objects in that memory, move those objects to fresh memory when you need to expand the storage etc.

But what about de-allocating the memory? Structs have no deinit, and that unsafely-pointed memory ain’t going to deallocate itself. But you still want your collection to be a struct, just like Array is, so instead, you implement a class to hold the buffer, and make that a member. That class can implement deinit to destroy its contained objects and deallocate the buffer when the struct is destroyed.

But you’re still not done. At this point, you’re in a similar position Swift arrays were in way back in the first ever Swift beta. They were structs, but they didn’t have full value semantics. If someone copies your collection, they’ll get a bitwise copy of the struct, but all this will do is copy the reference to the buffer. The two structs will both point to the same buffer, and updating the values in one would update both. Behaviour when the buffer is expanded would depend on your implementation (if the buffer expanded itself, they might both see the change, if the struct replaced its own buffer, they might not).

How to handle this? One way would be to be able to detect when a struct is copied, and make a copy of its buffer. But just as structs don’t have deinit, they also don’t have copy constructors – this ain’t C++!

So plan B – let the bitwise copy happen, but when the collection is mutated (say, via subscript set), detect whether the buffer is referenced more than once, and at that point, if it is, make a copy of the buffer before changing it. This solves the problem, and also comes with a big side-benefit – your collection is now copy-on-write (COW). Arrays in Swift are also COW, and this means that if you make a copy of your array, but then don’t make any changes, no actual copying of the buffer takes place – the buffer is only copied when it has to be, because you are mutating the values of some shared storage.

Since classes in Swift are reference-counted, it ought to be possible to detect if your buffer class is being referenced by more than one struct. But there wasn’t an easy Swift-native way to do this, until 1.2, which introduces isUniquelyReferenced and isUniquelyReferencedNonObjC function calls:

extension MyFancyCollection: MutableCollectionType {
  // snip...
 
  subscript(idx: Index) -> T {
    get {
      return _buffer[idx]
    }
    set(newValue) {
      if isUniquelyReferencedNonObjC(&_buffer) {
          // all fine, this buffer is solely owned
          // by this struct:
          _buffer[idx] = newValue
      }
      else {
          // uh-oh, someone else is referencing this buffer
          _buffer = _buffer.cloneAndModify(idx, newValue)
      }
   }
}

Great, COW collections all round. But before you run off and start implementing your own buffer class for use with your collection, you probably want to check out the ManagedBuffer class implementation, and a ManagedBufferPointer struct that can be used to manage it. These provide the ability to create and resize a buffer, check its size, check if its uniquely referenced, access the underlying memory via the usual withUnsafeMutablePointer pattern etc. The commenting docs are pretty thorough so check them [out.](http://swiftdoc.org/type/ManagedBufferPointer/)

Just in case you want to try out this new feature in a playground – this technique does not work with globals. See [this](https://devforums.apple.com/thread/261847?tstart=0) thread on the dev forums 

Though, unlike Array, they could be, since they mandate their contents are Equatable (via Hashable). And in fact you can make then equatable with extension Set: Equatable { }.

## Other changes ##

[unowned self] in init() closures

@autoclosure @noescape parameter versus type attribute changes

## References ##

https://functionwhatwhat.com/breaking-changes-to-swift-in-xcode-6-beta-3/

https://developer.apple.com/swift/blog/

http://iphonedev.tv/blog/2015/2/9/swift-12-fixes-and-breaks-a-few-things-you-should-be-excited

