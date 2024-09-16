---
title:  "Game Math: Enter the matrices"
date:   2024-09-16 10:00:00 +0200
categories: programming math vector matrix gamedev
---

TODO: Interactive demos?


This is meant for 'intermediate' gamedevs that grasp Vectors already. Written by someone without a math backround.

# What problem are we solving
In our games we'd like to have believable worlds, with objects that have positions. Ideally we'd like to rotate, scale and translate (=move) those objects, and often in a hierarchy. It would be neat if we could do those transformations with a single data structure, and efficiency is key. 

Matrices as it turns out are a very neat fit here! Lets invent them together. 

# Flipping 

We have a bunch of vertices, and a loop transforming those

TODO: demo with arrow verts

What happens when we flip the verts?

```
vert[i] = -vert[i];
```

Intuitively we understand the arrow turns around. This is something we do often, however are there are more tricks like this!

```
vert[i] = new() { x = vert[i].y, y = -vert[i].x }
```
TODO: demo

Our vertices have now rotated 90 degrees, isnt that nice! Surely this is a lot simpler than having to get quaternions involved, do this instead! We can achieve rotation the other way by not negating x(TODO verify).

# Mathing

Those are neat, but lets introduce some variables so we can go from one to the other.

```
vert[i] = new () {
				x = -1 	* vert[i].x 	+ 0 	* vert[i].y,
				y = 0 	* vert[i].x 	+ -1 	* vert[i].y
};
```
Or our 90 deg rotation
```
vert[i] = new () {
				x = 0 * vert[i].x + 1 * vert[i].y,
				y = -1 * vert[i].x + 0 * vert[i].y
};
```
We're essentially doing the same. Now lets actually introduce some variables:

```
float2 up = float2.up;
float2 right = float2.down;

	vert[i] = new () {
					x = right.x * vert[i].x + right.y * vert[i].y,
					y = up.x * vert[i].x + up.y * vert[i].y
	};

```
Still doing the same, just thinking about it differently. We're saying, the up axis should become the right, and the right axis should become down. Hold your hand and rotate to confirm. 

This works for arbitrary directions!

```
float angle = radians(45);
float2 up = new () { x = cos(angle), y = sin(angle) };
float2 right = new() { x = -sin(angle), y = cos(angle) };
```
Observe we're using the same trick as before to make a right angle vector. 

# Matrixing

Alright thats nice, we'd probably often want to rotate. So lets abstract some things to make it easier to use. 

Firstly lets call all this multiply-and-add something, a `dot product` sounds great. Then our code just becomes: 

```
vert[i] = new () { x = dot(right, vert[i]), y = dot(up, vert[i]) };
```

Thats already smaller, but why dont we introduce a data structure to keep these up and right axis? Thats exactly what a float2x2 matrix is, each row has an axis of rotation. Also called base. Multiplying a vector, or another matrix, does what we've been doing, namely the dot product. Exactly how is each 'cell' is the dot product of the source row with the target column, but thats not very important.  

```
float2x2 rotation = new float2x2(up, right);
```

# Scaling

So far our axis have been direction vector, i.e. of unit length. If we make them longer we get scale. 

# Translating

todo game math reference, sheer in the 4th dimension. 

Translate is adding a position value, i.e. after all our work we just want to do:
```
vert[i] += pos;
```
And you could, and that works. However it would be neat if we could stay within our matrix-dot-product mindset. What we could do is cheat, we want to trick the maths to become this:

```
vert[i] = new () {
				x =  0 * vert[i].x 	+ 1 * vert[i].y   + 1 * pos[i].x,
				y = -1 * vert[i].x 	+ 0 * vert[i].y   + 1 * pos[i].y
};
```

If we invent some more dimensions we can achieve this:

```
float3 up = new() { x = 0, y = 1, z = pos.x };
float3 right = new() { x = 1, y = 0, z = pos.y };

vert[i] = new () {
		x = dot(float4(vert[i], 1), right),
		y = dot(float4(vert[i], 1), up),
};

```
Because vert.w is always 1, and we hid the position in up.w, we end up doing ` + 1 * pos.x`. This is why there is always an extra dimension, i.e. in a 3d game we have a 4x4 matrix. 

Now we know this we can apply a bunch of optimalisations. We dont *have* to send the w vector, or entire last row of a matrix, we know they are 1. When we want to transform but not get position, we can set these to 0. This is useful for direction vectors, which are rotated and scaled, but not translated. 