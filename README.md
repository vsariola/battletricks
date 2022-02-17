# Battle tricks

This is an evolving list of various useful tricks for delivering the
most awesome fatalities in
[byte battles](https://battle.lovebyte.party/). Possibly also
interesting for TIC-80 sizecoding in general.


General notation
================

| Symbol | Definition              |
| ------ | ----------------------- |
| `x`    | x-coordinate of a pixel |
| `y`    | y-coordinate of a pixel |
| `i`    | index of a pixel        |
| `s`    | alias for math.sin      |


Basic optimizations
===================

- Functions, that are called 3 or more times should be aliased. For
  example, `e=elli` with `e()e()e()` is 3 bytes shorter than
  `elli()elli()elli()`. Functions with 5 characters may already benefit
  from aliasing with 2 calls: `r=rectb` with `r()r()` is 1 byte shorter
  than `rectb()rectb()`.

- `t=0` with `t=t+.1` is 3 bytes shorter than `t=time()/399`.

- `for i=0,32639 do x=i%240y=i/240 end` is 2-3 bytes shorter than
  `for y=0,135 do for x=0,239 do end end`.

- `(x*x+y*y)^.5` is 6 bytes shorter than `math.sqrt(x*x+y*y)`.

- `s(w-11)` and `s(w+8)` both approximate `math.cos(w)`, so only
  `math.sin` needs to be aliased. `s(w-11)` is slightly more accurate,
  with the cost of 1 byte.


One-lining
==========

Most whitespace can be removed from LUA code. For example: `x=0y=0` is
valid. All new lines can be removed or replaced with space, making the
whole code a single line:

```function TIC()cls()for i=0,32639 do poke4(i,i)end end```

> :warning: **Letters `a-f` and `A-F` after a number cause problems.**
> `a=0b=0` is not valid code. It is advisable to only used one letter
> variables in the ranges `g-z` and `G-Z` from the start; this will make
> eventual one-lining easier.


Load-function
=============

Function `load` takes a string of code and returns a function with no
named arguments, with the code as its body. It's particularly useful for
shortening the TIC function after it's onelined:

```lua
TIC=load'cls()for i=0,32639 do poke4(i,i)end'
```

As a rule of thumb, **one-lining and using the load trick can bring a ~
275 character code down to 256**.

`load` can be even used to minimize a function with parameters: `...`
returns the parameters. For example, the following saves 3 bytes:

```lua
SCN=load'r=...poke(16320,r)'
```

> :warning: **The backlash causes problems when using the load trick.**
> In particular, if you have a string with escaped characters in the
> original code i.e. `print("foo\nbar")`, then this needs to be
> double-escaped: `load'print("foo\\nbar")'`.

Dithering
=========

If you have a floating point color value, TIC-80 pix and poke functions
round it (toward zero). To add dithering, add a small value, between 0
and 1, to the color. The best technique depends whether you have `x` and
`y` available or only `i` and how many bytes you can spare:

| Expression          | Length  | Result                                        | Notes                                     |
| ------------------- | ------- | --------------------------------------------- | ----------------------------------------- |
|                     |         | ![No dithering](dither_none.png)              | No dithering                              |
| `s(i)*99%1`         | 9       | ![Random dithering](dither_random.png)        | "random" dithering, quite ugly            |
| `i*481/960%1`       | 11      | ![Chess dithering](dither_chess.png)          | chess horse                               |
| `(x*2-y%2)%4/4`     | 13      | ![Block dithering](dither_block.png)          | 2x2 block dithering                       |
| `(i*2-i//80%2)%4/4` | 17      | ![Block dithering from i](dither_block_i.png) | 2x2 block dithering (almost), from i only |

A quick example demonstrating the 2x2 block dithering:

```lua
function TIC()
 cls()
 for i=0,2399 do
  x=i%240
  y=i//240
  poke4(i,x/30+(x*2-y%2)%4/4)
 end
end
```


Palettes
========

The following palettes assume that `j` goes from 0 to 47. Usually
there's no need to make a new loop for this: just reuse another loop
with `j=i%48`.

| Expression                  | Length  | Result                                     | Notes                                           |
| --------------------------- | ------- | ------------------------------------------ | ----------------------------------------------- |
| `poke(16320+j,j*5)`         | 17      | ![Grayscale gradient](pal_gray.png)        |                                                 |
| `poke(16320+j,j%3*j/.4)`    | 22      | ![Blue gradient](pal_blue.png)             | Use `(j+1)%3` or `(j+2)%3` for different colors |
| `poke(16320+j,s(j)^2*255)`  | 24      | ![Rainbow](pal_rainbow.png)                |                                                 |
| `poke(16320+j,s(j/15)*255)` | 25      | ![Blue/Brown gradient](pal_blue_brown.png) | `s(j/15)^2` is less bright                      |

Code for testing palettes:

```lua
function TIC()
 cls()
 for j=0,47 do poke(16320+j,s(j/15)*255)end
 for c=0,15 do rect(c*5,0,5,5,c)end
end
s=math.sin
```


Motion blur
===========

In TIC-80 API, the `pix` and `poke4` functions round numbers towards
zero. This can be abused for a motion blur: `poke4(i,peek4(i)-.9)` maps
colors 1 to 15 into one lower value, but value 0 stays at it is. Like
so:

```lua
function TIC()
 t=time()/9
 circ(t%240,t%136,9,15)
 for i=0,32639 do poke4(i,peek4(i)-.9)end
end
```

Basic raymarcher
================

The basic structure for a raymarcher that has not been crunched to keep
it readable. The map is bunch of repeated spheres here:

```lua
function TIC()
 for i=0,32639 do
  -- ray (u,v,w), not normalized!
  u=i%240/120-1
  v=i/32639-.5
  w=1
  -- camera origo (x,y,z)
  x=3
  y=0
  z=time()/999 -- camera moves with time
  j=0
  repeat
   X=x%6-3 -- domain repetition
   Y=y%6-3
   Z=z%6-3
   -- ray not normalized=>reduce scale
   m=(X*X+Y*Y+Z*Z)^.5/2-1
   x=x+m*u
   y=y+m*v
   z=z+m*w
   j=j+1
  until j>15 or m<.1
  poke4(i,j)
 end
end
```