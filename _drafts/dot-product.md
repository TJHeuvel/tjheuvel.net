---
title:  "Game Math: Dot product"
date:   2023-08-30 20:00:00 +0100
categories: programming math vector gamedev
---

After working on Quake, Michal Abrash wrote Graphics Programming Black Book, detailing all his knowledge. On the topic of math, [he's quite clear](https://www.jagregory.com/abrash-black-book/#the-fundamentals-of-the-math-behind-3-d-graphics):

> .. The truth is, there’s not all that much to 3-D math... You really need only two basic math tools beyond simple arithmetic: dot products and cross products, and really mostly just the former. 

Clearly since then games have gotten much more complex, but the simple dot product still lies at the heart of many algoritms. Lets explore the dot product!

# What is the dot product?

The dot product of two vectors produces a scalar, i.e. a single 'float' value. Its calculated by summing the multiplication of each vector component, as such:

> a.x * b.x + a.y * b.y

The range is proportional to the sum of the input length

# What can you use it for?

## Vector Length
We know in our favourite engine we can do something like this to get the distance between two points:

> (b - a).magnitude

Those who paid any attention in highschool might also remember what the Pythagoras is going on, the good old a^2+b^2=c^2. Why does this appear in this article about the dot product?

```
	d = b-a;
	dot(d, d)
```
Well hot damn, pythagoras is a dot! We can appreciate how often this is done to see its fundamental.

# Projections

dot(fwd, velocity) forward velocity

# Frustum Culling

We might want to know if an object is within the view frustum of our camera. Roughly, our object is invisible if for any frustum plane its entirely behind it. 

## Whats a plane?

An infinite flat surface, that divides a space into two half-spaces, in front and behind. In our our example it can help to think of it as an infinite line. A regular line has a start and an end, a ray has a start and a direction that goes on infinitely. Planes are taking this one step further, a line without a start and end, going on infinitely, with only a direction and offset distance. 


Whenever i try to understand, i simplify. Lets forget about our camera for a moment, and observe this plane. This plane is at 0,0,0, and points squarely up. Rememeber we store the normal, in this case 0,1,0.

Whats the distance of a point, say 2,5,0 to our line? Well we only care about the Y value, the rest can be discarded. We want 0% of x, 100% of y, and 0% of z. Lets go diagonal, we want 50% of x, and 50% of y. We can understand x*.5 + y*.5 gives our result, hence:

> distanceToPlane = dot(plane-normal, pos)

However our camera can also move, our line can be offset. This is a simple addition, we say we dont care about the distance. Distance here is signed, be aware if its origin-to-plane, or plane-to-origin!

> dot(norm, pos) + distance


## Actually behind
We can now get the distance to the object, and know when the *center* is behind. However we care about when the entire object, its bounds, are past. What we really want to know is how much bounds is there in this normal dir, project the bounds on the normal!

However this only works with absolute-normal, i.e. positive. The size of (-1, 0, ) and (1, 0, 0) should be the same, and positive. Likewise for (-.5, .5, 0) and (.5, .5, 0). This latter case explains why we need to do absulte of the normal rather than the result, plus minus is minus.

> rad = dot(abs(norm), bounds)

Meaning we end up:

```
distanceToPlane = dot(norm,pos)
projectedObjSize = dot(abs(norm), bounds)

return distanceToPlane - projectedObjSize > 0;
```