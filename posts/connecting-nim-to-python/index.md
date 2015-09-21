.. title: Connecting Nim to Python
.. slug: connecting-nim-to-python
.. date: 2015-01-24 10:18:00 UTC-04:00
.. tags: nim,python
.. category: 
.. link: 
.. description: 
.. type: text

In my [previous][part-2] post I ended with a statement inquiring about interfacing Nim code with Python and after some experimentation I've been able to get some things to work. So, let's get to the code.

<!-- TEASER_END -->

# Compiling a library
The first thing we need to touch on is the Nim compiler. In most cases you turn your Nim code in to an executable by running the command.

    nim c projectfile.nim

But to turn the code in to a library we'll need to look at the [compiler's documentation](http://nim-lang.org/nimc.html). There are quite a few options but the one we are interested in is `--app` and in our case we'll use the `--app:lib` command to create a shared library (`.dll` in Windows, `.so` in linux, `.dylib` in OSX).

    nim c --app:lib projectfile.nim

Running the above command should create a `projectfile.dll` or `libprojectfile.so` file in the same directory as the `.nim` file. That's great, but it doesn't quite get us what we want. The library has been created but none of our functions have been exposed. Not very useful.

# The `exportc` & `dynlib` pragmas
Nim has special methods called [pragmas](http://nim-lang.org/manual.html#pragmas) that give the compiler extra information when it's parsing specific pieces of code. We've already seen them in use in my previous posts. Remember the following code to load the `isnan` function from `math.h`?

``` nim
import math

proc cIsNaN(x: float): int {.importc: "isnan", header: "<math.h>".}
  ## returns non-zero if x is not a number
```

In the above the code between `{.` and `.}` contains the `importc` pragma that tells the compiler to use the code from the `isnan` funtion in `math.h` for the `cIsNaN` Nim procedure.

So, if Nim has a way to import code from C it should also have a way to export code to C; and it does, `exportc`. Using `exportc` we can tell the compiler to expose our function to the outside. This plus the [`dynlib`](http://nim-lang.org/manual.html#dynlib-pragma-for-import) pragma makes sure we can access our procedure from the library.

Let's start with something simple.

``` nim
proc summer*(x, y: float): float {. exportc, dynlib .} =
  result = x + y
```

Saving this as `test1.nim` and running `nim -c --app:lib test1.nim` gives us a `test1.dll` file in windows. Now let's see if we can use it.

# Python `ctypes`
To access our compiled library we'll use Python's [`ctypes`](https://docs.python.org/2/library/ctypes.html) module which has been part of the standard library since version 2.5. It allows us to access the C based code that's been compiled in to our library and convert between Python types and C types. Here's the code to access our `summer` function.

``` python
from ctypes import *

def main():
    test_lib = CDLL('test1')
    
    # Function parameter types
    test_lib.summer.argtypes = [c_float, c_float]
    
    # Function return types
    test_lib.summer.restype = c_float
    
    sum_res = test_lib.summer(1.0, 3.0)
    print('The sum of 1.0 and 3.0 is: %f'%sum_res)

if __name__ == '__main__':
    main()
```

We can see that after loading the library we set the argument and return types of the function and then call the function with a couple numbers. Let's see what happens.

    C:\Workspaces\nim-tests>python test1.py
    The sum of 1.0 and 3.0 is: 32.000008

Hmmm, that's not right. It looks like we may not have used the correct types for my arguments and return. Let's compare the Nim `float` type to the Python ctypes `c_float` type. According to the [Nim manual](http://nim-lang.org/manual.html#pre-defined-floating-point-types) it's `float` type is set to "the processor's fastest floating point type". The Python [ctypes manual](https://docs.python.org/2/library/ctypes.html#fundamental-data-types) says a `c_float` is the same as a `float` in C. Since I'm running this code using the 32-bit versions of both Nim and Python (2.7) on a 64-bit Windows machine is the Nim compiler making its `float` a `double`?

``` python
from ctypes import *

def main():
    test_lib = CDLL('test1')
    
    # Function parameter types
    test_lib.summer.argtypes = [c_double, c_double]
    
    # Function return types
    test_lib.summer.restype = c_double
    
    sum_res = test_lib.summer(1.0, 3.0)
    print('The sum of 1.0 and 3.0 is: %f'%sum_res)

if __name__ == '__main__':
    main()
```

    C:\Workspaces\nim-tests>python test1.py
    The sum of 1.0 and 3.0 is: 4.000000

That seems to have fixed it. When we know we are going to be using `exportc` or creating a shared library Nim has [types](http://nim-lang.org/manual.html#types) that let us add more constraint and reduce these types of mix-ups (e.g. `cfloat`, `cint`).

# `openArray` arguments & the header file
Now let's try something a little more complex and take the `median` function from the [`statistics`](https://github.com/akehrer/nim-statistics) module I created in my last two posts.

Nim code:
``` nim
proc median*(x: openArray[float]): float {. exportc, dynlib .} = 
  ## Computes the median of the elements in `x`. 
  ## If `x` is empty, NaN is returned.
  if x.len == 0:
    return NAN
  
  var sx = @x # convert to a sequence since sort() won't take an openArray
  sx.sort(system.cmp[float])
  
  if sx.len mod 2 == 0:
    var n1 = sx[(sx.len - 1) div 2]
    var n2 = sx[sx.len div 2]
    result = (n1 + n2) / 2.0
  else:
    result = sx[(sx.len - 1) div 2]
```

Python code:
``` python
from ctypes import *

def main():
    test_lib = CDLL('test1')
    
    # Function parameter types
    test_lib.summer.argtypes = [c_double, c_double]
    test_lib.median.argtypes = [POINTER(c_double), c_int]
    
    # Function return types
    test_lib.summer.restype = c_double
    test_lib.median.restype = c_double
    
    # Calc some numbers
    nums = [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]
    nums_arr = (c_double * len(nums))()
    for i,v in enumerate(nums):
        nums_arr[i] = c_double(v)
    
    sum_res = test_lib.summer(1.0, 3.0)
    print('The sum of 1.0 and 3.0 is: %f'%sum_res)

    med_res = test_lib.median(nums_arr, c_int(len(nums_arr)))
    print('The median of %s is: %f'%(nums, med_res))

if __name__ == '__main__':
    main()
```

There's some interesting stuff happening here to allow us to call the `median` procedure. In the Nim code you can see that it's only looking for one argument, an `openArray` of floats. Like we saw in [Part 2][part-2], `openArrays` make passing arrays of different sizes to procedures possible. But how does this translate into the C library? (You can probably guess from the Python code.) One thing that may help us is an additional argument that can be passed to the Nim compiler.

    nim c --app:lib --header test1.nim

The `--header` option will produce a C header file in the `nimcache` folder where the module is compiled. If we look at the header we'll see the following.

``` c
/* Generated by Nim Compiler v0.10.2 */
/*   (c) 2014 Andreas Rumpf */
/* The generated code is subject to the original license. */
/* Compiled for: Windows, i386, gcc */
/* Command for C compiler:
   gcc.exe -c  -w  -IC:\NIM\lib -o c:\workspaces\nim-tests\nimcache\test1.o c:\workspaces\nim-tests\nimcache\test1.h */
#ifndef __test1__
#define __test1__
#define NIM_INTBITS 32
#include "nimbase.h"
N_NOCONV(void, signalHandler)(int sig);
N_NIMCALL(NI, getRefcount)(void* p);
N_LIB_IMPORT N_CDECL(NF, median)(NF* x, NI xLen0);
N_LIB_IMPORT N_CDECL(NF, summer)(NF x, NF y);
N_LIB_IMPORT N_CDECL(void, NimMain)(void);
#endif /* __test1__ */
```

The important lines are 13 and 14 where our two exported procedures are called out. We can see that `summer` is looking for two `NF` type arguments which we can assume are Nim type floats. `median` on the other hand is looking for not one argument like in the Nim procedure we defined but two, a pointer to an `NF` and an `NI` which is a Nim integer. There's also a hint about what that integer is, a length value. So the `openArray` argument has been translated in to a standard way of passing arrays in C, a pointer to the array and the length of the array.

In the Python code you can see we set the correct arguments (`[POINTER(c_double), c_int]`) and return type (`c_double`) and in lines 15 through 18 we cast a Python list of floats in to a C array of doubles. Then when we call the function in line 23 we make sure to convert the list's length to a `c_int`. Let's check the results.

    C:\Workspaces\nim-tests>python test1.py
    The sum of 1.0 and 3.0 is: 4.000000
    The median of [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0] is: 4.500000

That looks good, and it's probably enough for this post. In future posts we'll look at interfacing with more types like strings, tuples, and custom objects.

# Reference
[Nim Compiler User Guide](http://nim-lang.org/nimc.html)  
[Nim Manual](http://nim-lang.org/manual.html)  
[Python ctypes](https://docs.python.org/2/library/ctypes.html)  
[Python ctypes Tutorial](http://jjd-comp.che.wisc.edu/index.php/PythonCtypesTutorial)  


(Thanks to dom96 for corrections.)

[part-2]: link://slug/getting-started-with-nim-2