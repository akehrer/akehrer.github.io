.. title: Getting Started With Nim
.. slug: getting-started-with-nim
.. date: 2015-01-05 10:18:00 UTC-04:00
.. tags: nim
.. category: 
.. link: 
.. description: 
.. type: text

Having been a long time Python user I often find myself reaching for it over languages. When I saw the announcement for version 0.10.2 of [Nim][nim-lang] on [Hacker News][hn-nim] it got me interested in learning another language, especially one that compiles to C and executables. I have some experience with C and C++ having done some Arduino projects, but my biggest memory of it is from a programming data structures class I took as an undergrad. All that memory management somehow didn't fit my brain's way of thinking and it turned me off to programming as a profession.

<!-- TEASER_END -->

Of course that didn't stop me from learning to program, it just drove me to interpreted languages like PHP and Python. I remember googling "garbage collection" when I learned about Python and saying "This looks like the language for me." What followed was 14+ years of enjoyable programming. Now my skills have improved to the point that I'm looking to broaden my language talents and Nim looks like a good one to add. 

Some of the things that attracted me to Nim:

  - The Python like syntax
  - Static typing with basic type inference
  - Garbage collector
  - Compiles to C and then to an executable
  
I'm hoping that the syntax and garbage collection will allow me to quickly getting programs more useful than "Hello World" running. The static typing and compiling should make things less error prone and fast, and the executable means working programs should be more portable than Python.

My first Python project involved creating a linear regression using least squares to fit a calibration line to data collected off a sensor array. Since SciPy and NumPy weren't around/mature at the time I cracked open my statistics textbook and wrote all the functions needed to fit the data. Since Nim [doesn't appear][nim-modules] to have a statistics module and given my statistics knowledge is a little rusty I thought starting a basic `nim-statistics` module would both give me a good understanding of Nim basics and refresh my stats.

I won't go over installing Nim or leaning the basics since that is covered in other places. A lot of what I will show here will be based on other people's code and reading the standard library. So let's start with something simple, a [Gaussian Distribution][gauss-wiki]. First we'll import the built-in `math` module since we know we'll be needing procedures from it. Then we'll define an object to hold the distribution's parameters.

``` nim
import math

type
  GaussDist* = object
    mu, sigma: float
```

Some things to note:

  - The `GaussDist` type inherits from the base `object` type and has two parameters with type `float`
  - Symbols that are marked with `*` are exported by the module (where another modules uses the `import` function). So `import math` imports everything mark with an asterisk in the [math](http://nim-lang.org/math.html) module and this module will export the `GaussDist` type.
  
Now let's add some functionality.

``` nim
import math

type
  GaussDist* = object
    mu, sigma: float
	
proc NormDist*(): GaussDist = 
  ## A Normal Distribution is a special form of the Gaussian Distribution with 
  ## mean 0.0 and standard deviation 1.0
  result.mu = 0.0
  result.sigma = 1.0
	
proc mean*(g: GaussDist): float =
  result = g.mu

proc standardDeviation*(g: GaussDist): float = 
  result = g.sigma

proc variance*(g: GaussDist): float = 
  result = math.pow(g.sigma, 2)
  
when isMainModule:
  var n = NormDist()
  var gnorm = GaussDist(mu: 0.0, sigma: 1.0)

  assert(n.mean == gnorm.mean)
  assert(n.standardDeviation == gnorm.standardDeviation)
  assert(n.variance == gnorm.variance)

  echo "SUCCESS: Tests passed!"
```

More notes:

  - The `mean`, `standardDeviation`, and `variance` procedures are all available from the `math` module but don't know how to handle the `GaussDist` type so we use Nim's ability to overload procedures to implement that here.
  - In the `NormDist` procedure you can see an example of the use of Nim's default retun value `result`. Since result is a `GaussDist` type due to the procedure's definition we can just assign the parameter values without needing to create an intermediate variable.
  - `math.pow` could be replaced with `pow` since the later is included in the import, but doing it this way makes it clear where the procedure is coming from.
  - The `when isMainModule` check gives us a convenient place to put tests since what's in the block won't be executed when the module is imported.
  
Finally, let's add the distribution functions.

``` nim
import math

## Some additional functions from math.h are needed 
## that aren't included in the math module

proc erf(x: float): float {.importc: "erf", header: "<math.h>".}
## computes the error function (also called the Gauss error function)

type
  GaussDist* = object
    mu, sigma: float

proc NormDist*(): GaussDist = 
  ## A Normal Distribution is a special form of the Gaussian Distribution with
  ## mean 0.0 and standard deviation 1.0
  result.mu = 0.0
  result.sigma = 1.0

proc mean*(g: GaussDist): float =
  result = g.mu

proc standardDeviation*(g: GaussDist): float = 
  result = g.sigma

proc variance*(g: GaussDist): float = 
  result = math.pow(g.sigma, 2)

## The distribution functions are from 
## http://en.wikipedia.org/wiki/Normal_distribution
proc pdf*(g: GaussDist, x: float): float = 
  var numer, denom: float

  numer = math.exp(-(math.pow((x - g.mu), 2)/(2 * math.pow(g.sigma, 2))))
  denom = g.sigma * math.sqrt(2 * math.PI)
  result = numer / denom

proc cdf*(g: GaussDist, x: float): float = 
  var z: float

  z = (x - g.mu) / (g.sigma * math.sqrt(2))
  result = 0.5 * (1 + erf(z))

when isMainModule:
  var n = NormDist()
  var gnorm = GaussDist(mu: 0.0, sigma: 1.0)

  assert(n.mean == gnorm.mean)
  assert(n.standardDeviation == gnorm.standardDeviation)
  assert(n.variance == gnorm.variance)
  assert(n.pdf(0.5) == 0.3520653267642995)
  assert(n.cdf(0.5) == 0.6914624612740131)

  echo "SUCCESS: Tests passed!"
```

Alright, so that's a good start. Some final notes:

  - The Nim `math` module doesn't include the `erf` procedure but it's available in the standard C library so it's  and easy import here.
  - There are multiple ways to call Nim procedures and in this post's example I've chosen to use the method call syntax (`obj.method()`) because it looks more natural to me.
  
This was a nice first go at learning Nim and although it didn't really exercise some of the 'wow' features it gave me a good introduction to the language and a starting point to grow from. Hopefully I'll be able to flesh out the statistics module and as I learn more about Nim I will post it here.

# Reference
[Nim Manual](http://nim-lang.org/manual.html)  
[Tutorial 1](http://nim-lang.org/tut1.html)  
[Tutorial 2](http://nim-lang.org/tut2.html)  
[Garbage Collector](http://nim-lang.org/gc.html)  
[Nim Standard Library][nim-modules]  


[hn-nim]: https://news.ycombinator.com/item?id=8809215
[nim-lang]: http://nim-lang.org/
[nim-modules]: http://nim-lang.org/lib.html
[gauss-wiki]: http://en.wikipedia.org/wiki/Normal_distribution