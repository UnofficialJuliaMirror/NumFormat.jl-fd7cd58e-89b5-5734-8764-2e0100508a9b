# NumFormat

*IMPORTANT: all of this package’s functionalities are being merged into
[Formatting.jl](https://github.com/lindahua/Formatting.jl), please use that
instead*

[![Build Status](https://travis-ci.org/tonyhffong/NumFormat.jl.svg?branch=master)](https://travis-ci.org/tonyhffong/NumFormat.jl)

A way to get around the limitation that `@sprintf` has to take a literal string argument.
The core part is basically a c-style print formatter using the standard
`@sprintf` macro.
It also adds functionalities such as commas separator (thousands), parenthesis for negatives,
stripping trailing zeros, and mixed fractions.

## Usage and Implementation

The idea here is that the package compiles a function only once for each unique
format string within the `NumFormat.*` name space, so repeated use is faster.
Unrelated parts of a session using the same format string would reuse the same
function, avoiding redundant compilation. To avoid the proliferation of
functions, we limit the usage to only 1 argument. Practical consideration
would suggest that only dozens of functions would be created in a session, which
seems manageable.

Usage
```julia
using NumFormat

fmt = "%10.3f"
s = sprintf1( fmt, 3.14159 ) # usage 1. Quite performant. Easiest to switch to.

fmtrfunc = generate_formatter( fmt ) # usage 2. This bypass repeated lookup of cached function. Most performant.
s = fmtrfunc( 3.14159 )

s = format( 3.14159, precision=3 ) # usage 3. Most flexible, with some non-printf options. Least performant.
```

## Alternative approaches suggested by others:

* [Formatting.jl](https://github.com/lindahua/Formatting.jl).

* Put the macro in a quote block and eval it (very slow)
```julia
fmt = "%10d"
n = 1234
s = eval( Expr( :macrocall, symbol( "@sprintf" ), fmt, n ) ) # VERY slow, 1000x penalty
```
* `ccall` to libc sprintf. See [this gist](https://gist.github.com/dpo/11000433). The
example shows 6-7x speed penalty.

* Set up a lambda with the macro inside. Ok for repeated use. But the lambda
goes out of scope quickly so it cannot be reused. `@eval` would also repeat
compilation, even for the same format.
```julia
fmt = "%10d"
n = 1234

#method A lambda
l = :( x -> x ) # placeholder lambda
l.args[2].args[2] = Expr( :macrocall, symbol( "@sprintf" ), fmt, :x )
mfmtr = eval(l)

#method B @eval
@eval mfmtr(x) = @sprintf($fmt,x)

s = mfmtr( n ) # quite fast, but the definition is clunky
```

## Speed

`sprintf1`: Speed penalty is about 20% for floating point and 30% for integers.

If the formatter is stored and used instead (see the example using `generate_formatter` above),
the speed penalty reduces to 10% for floating point and 15% for integers.

## Commas

This package also supplements the lack of thousand separator e.g. `"%'d"`, `"%'f"`, `"%'s"`.

Note: `"%'s"` behavior is that for small enough floating point (but not too small),
thousand separator would be used. If the number needs to be represented by `"%e"`, no
separator is used.

## Flexible `format` function

This package contains a run-time number formatter `format` function, which goes beyond
the standard `sprintf` functionality.

An example:
```julia
s = format( 1234, commas=true ) # 1,234
s = format( -1234, commas=true, parens=true ) # (1,234)
```

The keyword arguments are (Bold keywards are not printf standard)

* width. Integer. Try to fit the output into this many characters. May not be successful.
   Sacrifice space first, then commas.
* precision. Integer. How many decimal places.
* leftjustified. Boolean
* zeropadding. Boolean
* commas. Boolean. Thousands-group separator.
* signed. Boolean. Always show +/- sign?
* positivespace. Boolean. Prepend an extra space for positive numbers? (so they align nicely with negative numbers)
* **parens**. Boolean. Use parenthesis instead of "-". e.g. `(1.01)` instead of `-1.01`. Useful in finance. Note that
  you cannot use `signed` and `parens` option at the same time.
* **stripzeros**. Boolean. Strip trailing '0' to the right of the decimal (and to the left of 'e', if any ).
   * It may strip the decimal point itself if all trailing places are zeros.
   * This is true by default if precision is not given, and vice versa.
* alternative. Boolean. See `#` alternative form explanation in standard printf documentation
* conversion. length=1 string. Default is type dependent. It can be one of `aAeEfFoxX`. See standard
  printf documentation.
* **mixedfraction**. Boolean. If the number is rational, format it in mixed fraction e.g. `1_1/2` instead of `3/2`
* **mixedfractionsep**. Default `_`
* **fractionsep**. Default `/`
* **fractionwidth**. Integer. Try to pad zeros to the numerator until the fractional part has this width
* **tryden**. Integer. Try to use this denominator instead of a smaller one. No-op if it'd lose precision.

See the test script for more examples.
