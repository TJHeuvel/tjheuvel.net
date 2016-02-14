---
title:  "What every game programmer should know about vectors"
date:   2015-11-18 10:34:17 +0200
categories: programming math vectors gamedev
---

Mathmatics in game development is usually thought of as a difficult subject, and while this is might be true for matrixes and quaternions there is a whole lot about vectors that doesnt have to be for wizards only. They are much simpler, often taught in school and easy to visualise. This article explains several use cases you'll encounter while making games, and ways to solve them.

## What is a vector, and why do we need them

In order to simulate a game world we need to be able to give objects a position. We borrow a system called a [Cartesian Coordinate][cart-coords] system

A vector can represent two distinct things; positions and directions. Both are in our amazing grid. 

When you're making a 2D game you will mostly use a vector with the X and Y components, when working in 3d there is a third Z component added. Luckily most of the mathmatics easily convert to and from 3d.

In this tutorial we will be covering 2D only, and use Vector2. Luckily its easy to upgrade the math to 3d.

> People that are much smarter then me prefer to call floats [Scalars][scalars]. I however do not.

## How do i use them in my game
What follows is a list of common scenarios in a 2d game. Most math is applicable in 3d too.

We're assuming the following base class. Every entity must have some sort of position defined by an X and Y. Whenever the `Update` method is called we will move him.

{% highlight C# %}
class Entity
{
    public Vector2 Position;

    void Update();
}
{% endhighlight %}

## Moving our entity

Alright, we have an entity who has a position in the world lets try to move him around. 

You can add a vector to another vector to displace them, for example adding `1,0` will make him move right one unit. Its also possible to multiply or scale a vector, which does exactly what you think.

{% highlight C# %}
float speed = 5f;

void Update() 
{
    this.Position += new Vector2(1,0) * speed * deltaTime;
}
{% endhighlight %}

After this we multiply it by our speed. 

You'll also notice we also multiply the displacement by a `deltaTime`, which is the time in seconds since the last Update. This makes sure we are moving at 1 unit per second and not per frame, because frame rates can differ greatly. It is very important to make sure your game is independant of the frame rate!

## Rotating our guy

Since moving only right would make for a pretty crabby game, lets see if we can free our protagonist and make him move in multiple directions. We're aiming for a classic 2D top down experience like GTA2.

We will use sin and cos to create a happy little vector pointing in the direction we are facing in using the `rotation` variable. This is the rotation in radians, which go from 0 to pi * 2.

{% highlight C# %}
float rotation;

void Update() 
{
    rotation += Math.PI * 2f * deltaTime;
    this.Position += getForward() * speed * deltaTime;
}

Vector2 getForward() 
{
    return new Vector2(
        Math.Sin(this.rotation), 
        Math.Cos(this.rotation)
    );
}

{% endhighlight %}

In the example we constantly rotate, at a speed of 1 revolution a second because 360 degrees is 2 times pi. In your game you'd most likely want to rotate according to the users input though.


## Distance between vectors

Its often useful to know how far entities are away from each other.

{% highlight C# %}

float distance(Vector2 a, Vector2 b)
{
    return length(b - a);
}

float length(Vector2 a)
{
    return Math.Sqrt(a.x*b.x + a.y*b.y);
}
{% endhighlight %}

> In some situations where you dont need to know the exact distance you can omit the Square root.

## Moving towards another enity

Say we want to move an enemy towards our hero at a speed of 5 units per second, how would we do that?

Firstly we need to get the direction between me and my target and then multiply this by the speed. Basicly you take 1% out of it.

A normalized vector is a happy special little vector that always has the length of one.

{% highlight C# %}

void moveTowards(Entity target, float speed = 1f)
{
    this.Position += normalize(target.Position - this.Position) * speed * deltaTime;
}

Vector2 normalize(Vector2 a)
{
    return a / length(a);
}

{% endhighlight %}

> You can also invert direction vectors to go the opposite way. To move away from the target just use the opposite vector, -normalize(a,b).

## Am i looking towards another guy?

The last trick we'll pull out of our hat is checking to see if i am looking towards another entity. What we are interested in is the angle between the direction we are looking in (our `getForward`) and the direction between us and the other guy.

This is where the `dot product` comes in, when given two normalised vectors that are pointing in the same direction it returns 1, when they are pointing exactly opposite it returns -1. It achieves this by multiplying each component from the first vector by the second, and adding that up. 

A vector that is pointing all the way to the right has an X of 1, and a vector that is pointing left has an X of -1. 

{% highlight C# %}
bool isFacing(Vector2 otherPosition)
{
    return dot( 
            normalize(this.Position - otherPosition),
            getForward()
    ) > .9f; 
}


float dot(Vector2 a, Vector2 b)
{
    return a.x * b.x + a.y * b.y;
}

{% endhighlight %}

The dot product is an easy computationally not expensive way to get the angle. Some lighting models depend on just this factor, blinnpon works by dot(viewdirection, normal) or something. 

This concludes the first post!

[scalars]: http://en.wikipedia.org/wiki/Scalar_(mathematics)
[cart-coords]: https://en.wikipedia.org/wiki/Cartesian_coordinate_system