# Notes from [phrack magazine](http://www.phrack.org/papers/attacking_javascript_engines.html)

JavaScript engine contains at least: 
- a compiler infrastructure. at least one just-in-time compiler
- a virtual machine that operated on JS values 
- a runtime that provides a set of builtin objects and functions

# VM, Values & NaN-boxing
The virtual machine contains an interpreter which can directly execute the emitted bytecode. The VM is often implemented as stack-based machines and
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

Objectes
- arrays
- functions

All major JS engines represent a value a value with no more than 8 bytes forperfomance reasons (fast copying, fits into a register on 64-bit archs).
JSC(JavaScript Core) & SpiderMonkey use a concept called NaN-Boxing. NaN-Boxing makes use of the fact that there exist multiple bit pattern which all
represent NaN, so other values can be encoded in these. Specifically, ever IEE 754 floating point value with all exponent bits set, but a fraction not
equal to zero represents NaN. For double precision values this leaves us with 2^51 different bits of patterns. Thats enough to encode both 32-bit ints
and pointers, since even on 64-bit platforms only 48 bits are currently used for addressing 

# Objects and Arrays
*Objects* in JS are esentially collections of properties which are stores as key, value pairs. Properties can be accessed either with the dot operator
or through square brackers. At least in theory, values used as keys are converted to strings before performing the lookup.

*Arrays* are described by the specs as special objects whose props are also called elements if the property name can be represented by a 32-bit
integer. Most engines today extent this notion to all objects. An array then becomes an object with special 'length' property whose value is always
equal to the index of the highest element plus one. The net result of all this is that every object has both properties, accesed through a string or
symbol key, and elements accessed through int indices.

Internally, JSC stores both properties and elements in the same memory region and stores a pointer to that region in the object itself. This pointer
points to the middle of the region, propertiesare stored to the left of it(lower addressed) and elements to the right. There is also a small header
located just before the pointed to address that contains the length of the element vector. This concept is called a "Butterfly" since the values
expand to the left and right, similar to the wings of a butterfly. Presumably. 

# Functions
When executing a function's body, two special variables become available. One of them *'arguments'* provides access to the arguments and caller of the
function, thus enabling the creation of function with a varible number of arguments. The other *'this'* refers to different objects depending on the
invocation of the function.

- If the function was called a constructor (using 'new func()'), then *'this'* points to the newly created object. Its proto has already been set to
  the `.prototype` prop of the func object, which is set to the a new object during function definition. 
- If the function was called as a method of the same object (using 'obj.func()'), then *'this'* will point to the reference object. 
- Else *'this'* simply points to the current global object, as it does outside of a function as well. 

Since function are first class objects in JS they too can have props. Two interesting props of functions are `.call` and `.apply` which allow calling
the function with a given *'this'* object and args. This can for example be used to implement decorator functionality: 
```js
function decorate(func) {
	return function() {
		for(var i=0;i < arguments.length; i++) {
		}
		return func.apply(this.arguments);
	};
}
```
This also has some implications on the implementations of JS functions inside the engine as they cannot make any assumptions about the value of the
reference object which they are called with, as it can be set to arbitrary values from script. Thus, all internal JS functions will need to check the
type of not only their args but also of the *'this'* object.

Internally, the built-in function and methods are usually implemented in one of two ways; as native functions in C++ of in JS itself. 

# The bug
The bug in question lies in the implementation of Array.prototype.slice. The native function arrayProtoFuncSlice, located in ArrayPrototype.cpp, is
invoked whenever the slice method is called in JS
```js
var a = [1,2,3, 4]
var s = a.slice(1,3)
// s now contains [2,3]
```

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


