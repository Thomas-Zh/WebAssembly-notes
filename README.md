# Embind: C++ in a JS way
### A few days ago, I was handed to a task to make an existing C++ library run in browsers. After hours of grinding, I found the way to assemble all parts together, and to help more newbies like me, it is necessary to summarize the useful information and formulate a general solution.

This note is not intended to dive into the WebAssembly technology with lots of critical thinking but rather give A general solution to the problem in which case developers have to convert a C++ library to JS.
___
## **WebAssembly**

Simple video introduction:

https://www.youtube.com/watch?v=1J6Z5wBfSnQ&t=478s

The general idea behind WebAssembly is to call machine code (.WASM) directly from JS APIs. It is fast, efficient, and compact.
___

## **Getting started**
https://emscripten.org/docs/getting_started/downloads.html

***For C++ library that depends on Windows-specific libraries, it cannot be compiled to run on web platforms :(***

Source:https://github.com/emscripten-core/emsdk/issues/153#issuecomment-401106516

Just follow the steps and get your development tool kit set up.
Notes for Windows users:

1. run PowerShell as administrator and type in:
```
Set-ExecutionPolicy Unrestricted
```

Source: https://stackoverflow.com/questions/16460163/ps1-cannot-be-loaded-because-the-execution-of-scripts-is-disabled-on-this-syste

2. whenever open a new PowerShell, run the following command to start `emsdk`.

```
./emsdk activate latest
```

> There are different ways to convert a C/C++ function/class to JS, but we are going to use `Embind` as it is really easy to implement and requires less interference with the original C/C++ library IMHO.

>Source: https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html


I don't have a lot to add to this introductory tutorial as it covers most of the basics and use cases.
>https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html#embind

### ***Just a few reminders, when dealing with Class:***
1.
```
// for different use cases
class_<Base>("Base")
// simple function
.function("MyFunction", &Base::MyFunction)
// function that takes in pointers as arguments
.function("YourFunction", &Base::YourFunction, allow_raw_pointers())
// function that needs a wrapper
.function("OurFunction", optional_override([](Base& self, [type] [myArgument]){return self.OurFunction([myArgument])}); 

```
2.   Member variable that is raw pointer cannot be treated as `property`, which means you cannot access it directly as it is in JS. It sort of make sense since there is no `pointer` type in JS, and the closest type is `Number`. Thus, the way around it is to either cast the pointer to an unsigned int and pass that to JS OR use the `val` method to pass the content of the memory block it's pointing to JS.

This brings us to the next topic: Memory Views.


## **Memory Views & Pointers :)**
Here things are getting interesting.

Imagine your C++ function is asking an array of ints or chars as inputs, how can we deal with it?
Well, thankfully, Emscripten provides us with a way to access the memory allocated just like using the pointer in C/C++.

Here, it gives a brief introduction to how JS allocates memory and how Emscripten simulates the malloc() from C: 
>https://emscripten.org/docs/porting/emscripten-runtime-environment.html#emscripten-memory-model

>https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#interacting-with-code-access-memory

In practice, we can use
```
var buf = Module._malloc(myTypedArray.length*myTypedArray.BYTES_PER_ELEMENT);
Module.HEAPU8.set(myTypedArray, buf);
```
to set up a "pointer" to our allocated memory and copy the whole typedArray into the memory. In fact, `buf` is just a number represents the byteOffset as described here:

>Source: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/byteOffset


Then, we need to pass the "pointer" to the C++ function using the `reinterpret_cast<>` type casting operator. 
```
.function("OurFunction", optional_override([](Base& self, [int] [myArgument]){return self.OurFunction(reinterpret_cast<int*>([myArgument])}); 
```
This wrapper function will tell C++ to cast the incoming `int` type argument as `int*` type, in which the actual value it's holding up can now be interpreted as the byteOffset to the memory block. Same thing goes with the `char*` type.


When getting the data from C++ function to JS, we can apply the same principle, but this time some boilerplate functions are there ready for us to use.

Here is the copypasta from
 >https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html#memory-views
```
#include <emscripten/bind.h>
#include <emscripten/val.h>

using namespace emscripten;

unsigned char *byteBuffer = /* ... */;
size_t bufferLength = /* ... */;

val getBytes() {
    return val(typed_memory_view(bufferLength, byteBuffer));
}

EMSCRIPTEN_BINDINGS(memory_view_example) {
    function("getBytes", &getBytes);
}
```
If dealing with `Class`, we can simply include another .function() in our Class bindings.
```
.function("getBytes", optional_override([](BaseClass& self) {return self.getBytes(); }))
```

Awesome, now it should be clear how to pass data between C++ and JS.
One more thing:
Since the Garbage Collection is still under development, we still need to call
```
Module._free(buff);
```
to release the memory.

>Source: https://stackoverflow.com/a/41372484/8560580


## **Testing**
Note that we can always require extra methods (e.g. UTF8ToString()) to be exported for future use while compiling. Just add the following command:
```
-s 'EXTRA_EXPORTED_RUNTIME_METHODS=["YOUR_EXTRA_METHOD"]'
```
So, what if you want to test your converted JS APIs?
First, create a HTML file 
```
<!DOCTYPE html>
<html>
<body>
    <script src="generated_js_file.js"></script>
    <script>
        Module.onRuntimeInitialized = _ => {
            /**
            ** JS code
            **/
        };
    </script>
</body>
</html>
```
Instead of hosting our own server and run the HTML file, we can use `emrun` to start a local server for us and run the HTML file.

>More info: https://emscripten.org/docs/compiling/Running-html-files-with-emrun.html


## **Next Step**

To Be Updated
