# Battle tricks

This is an evolving list of various useful tricks for
[byte battles](https://battle.lovebyte.party/) and possibly also for
TIC-80 sizecoding in general.

General notation
================

| Symbol | Definition              |
| ------ | ----------------------- |
| `x`    | x-coordinate of a pixel |
| `y`    | y-coordinate of a pixel |
| `i`    | index of a pixel        |
| `s`    | alias for math.sin      |

Dithering
=========

If you have a floating point color value, TIC-80 pix and poke functions
round it (toward zero). To add dithering, add a small value, between 0
and 1, to the color. The best technique depends whether you have `x` and
`y` available or only `i` and how many bytes you can spare:

| Expression          | Length  | Notes                                     |
| ------------------- | ------- | ----------------------------------------- |
| `s(i)*99%1`         | 9       | "random" dithering, quite ugly            |
| `i*481/960%1`       | 11      | chess horse                               |
| `(x*2-y%2)%4/4`     | 13      | 2x2 block dithering                       |
| `(i*2-i//80%2)%4/4` | 17      | 2x2 block dithering (almost), from i only |

You can test the different dithering techniques with this simple tunnel:

```lua
function TIC()
 for i=0,32639 do
  x=i%240-120
  y=i/240-68
  u=math.atan2(x,y)*4
  v=99/(x*x+y*y)^.5+time()/99
  c=(s(u)+s(v))*2
  d=(x*2-y%2)%4/4 -- dithering
  poke4(i,c+d)
 end
end
s=math.sin
```

Palettes
========

Usually there's no need to make another loop for a palette; just do `j=i%48`.

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

Basic raymarcher
================

The basic structure for a raymarcher that has not been crunched to keep
it readable:

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