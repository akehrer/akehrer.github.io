.. title: Getting Started With Nim - Part 2
.. slug: getting-started-with-nim-2
.. date: 2015-01-14 13:18:00 UTC-04:00
.. tags: nim
.. category: 
.. link: 
.. description: 
.. type: text

[Last time][part-1] I wrote about my interest in learning Nim to broaden my skills and started by creating a simple statistics module. We went over some of the nuances I learned about the language and built the beginning of a model for the [Gaussian Distribution][gauss-wiki]. Today we'll be looking at some of the things I founds as I worked out other procedures in the module.

<!-- TEASER_END -->

### NaN
Some of the functions in the built-in `math` module return `NaN` as a value but I could not find an easy way in Nim to check for this. Luckily the standard `math.h` library includes an `isnan()` function that we can reuse. In this case I have created an additional procedure to convert the integer result from the C function to a boolean. You can see I also created a NAN constant that I used in some of my code. 

``` nim
import math

const
  NAN = 0.0/0.0 # floating point not a number (NaN)

proc cIsNaN(x: float): int {.importc: "isnan", header: "<math.h>".}
  ## returns non-zero if x is not a number

proc isNaN*(x: float): bool =
  ## converts the integer result from cIsNaN to a boolean
  if cIsNaN(x) != 0: true
  else: false
```

### More Statistics
With NaN taken care of I began writing more procedures for descriptive statistics like `median`, `skewness`, `kurtosis` and `quantile`. Here is the code for `median`:

``` nim
import algorithm  # needed for `sort`

proc median*(x: openArray[float]): float = 
  ## computes the median of the elements in `x`. 
  ## If `x` is empty, NaN is returned.
  
  var sx = @x # convert to a sequence since sort() won't take an openArray
  sx.sort(system.cmp[float])
  
  try:
    if sx.len mod 2 == 0:
      var n1 = sx[(sx.len - 1) div 2]
      var n2 = sx[sx.len div 2]
      result = (n1 + n2) / 2.0
    else:
      result = sx[(sx.len - 1) div 2]
  except IndexError:
    result = NAN
```

  - The `median` procedure takes an `openArray` type as it's input, similar to the `mean` procedure in the math module. [Open arrays](http://nim-lang.org/manual.html#open-arrays) in Nim make passing arrays of different sizes to procedures possible. The `openArray` type has some basic operations available, but we can't send one to the `sort` procedure so we need to convert it to a sequence first (by using the `@` function on `x`).
  - The use of the `div` procedure when computing the index of the sorted sequence is required since it does integer division, unlike `/` which returns a float.
  - Nim has a `try...except` syntax similar to Python and it's used here to catch when an empty array is send to the procedure.
  - Also note that we are using the `NAN` constant we created above.
  
Now that we have a working `median` procedure, let's add some tests:

``` nim
when isMainModule:
  # Setup some test data
  var data1: array[0..6, float]
  var data2: array[0..7, float]
  var data3 = newSeq[float]()
  var data4: array[1, float]
  var data5: array[2, float]
  data1 = [1.4, 3.6, 6.5, 9.3, 10.2, 15.1, 2.2]
  data2 = [1.4, 3.6, 6.5, 9.3, 10.2, 15.1, 2.2, 0.5]
  data4 = [2.3]
  data5 = [2.2, 2.5]
  
  # Test median()
  assert(abs(median(data1) - 6.5) < 1e-8)
  assert(abs(median(data2) - 5.05) < 1e-8)
  assert(isNaN(median(data3)))  # test an empty sequence
  assert(abs(median(data4) - 2.3) < 1e-8)
  assert(abs(median(data5) - 2.35) < 1e-8) 
```

Given this is floating point math we shouldn't use `==` tests.  I also went back and fixed the ones I used for the `GaussDist` test in [Part 1][part-1]. 

As I was working through other procedures I got caught by Nim's static typing. Sometimes the code looks enough like Python that I forget to check that the appropriate types are being used for procedures. Here's the code for the `quantile` procedure as an example:

``` nim
proc quantile*(x: openArray[float], frac: float): float = 
  ## Computes the quantile value of `x` determined by the fraction `frac`
  ## If `x` is empty, NaN is returned.
  ## `frac` must be between 0 and 1 so for the 25th quantile 
  ## the value should be 0.25, any other value returns `NaN`
  if x.len == 0:
    result = NAN  
  elif frac < 0.0 or frac > 1.0:
    result = NAN
  elif frac == 0.0:
    result = x.min
  else:
    var sx = @x # convert to a sequence since sort() won't take an openArray
    sx.sort(system.cmp[float])

    var n = sx.len - 1  # max index
    var i = int(math.floor(float(n) * frac))  # quantile index

    if i == n:
      result = sx[n]
    elif sx.len mod 2 == 0:
      # even length
      var n1 = sx[i]
      var n2 = sx[i+1]
      result = (n1 + n2) / 2.0
    else:
      # odd length
      result = sx[i]
```

You can see in line 17 that we need to convert `n` to a float before it can be multiplied by `frac` (a float). This is because Nim will not automatically convert an `int` to a `float` [[ref](http://nim-lang.org/tut1.html#floats)]. The result is then converted back to an `int` so it can be used as an index for the sequence. Thankfully Nim provides useful feedback when things are wrong and tries to point you in the right direction.  Here's the error when I try to compile without converting `n` to a float.

	Error: type mismatch: got (int, float)
	but expected one of:
	system.*(x: set[T], y: set[T]): set[T]
	system.*(x: int, y: int): int
	algorithm.*(x: int, order: SortOrder): int
	system.*(x: int64, y: int64): int64
	system.*(x: int32, y: int32): int32
	system.*(x: int8, y: int8): int8
	system.*(x: float, y: float): float
	system.*(x: float32, y: float32): float32
	system.*(x: int16, y: int16): int16
	
	
So the `*` procedure is looking for two parameters of the same type, but it's getting an `int` and a `float`. Fix that and things will work.

All in all I like what I've seen so far with Nim. I feel it was easy to pick up and get started. The documentation is adequate but not complete, which is not surprising for a young language. So far I've used it more like a complied Python and haven't gotten too deep into its unique features. It will be interesting to see if it can be used to create Python modules, maybe by interfacing through [cython](http://cython.org/). But that's for another time.

You can find the code for my `nim-statistics` module [here](https://github.com/akehrer/nim-statistics).

### Reference
[Part 1][part-1]  

[part-1]: link://slug/getting-started-with-nim
[gauss-wiki]: http://en.wikipedia.org/wiki/Normal_distribution