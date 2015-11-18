---
layout: default

title:  "What every game programmer should know about vectors"
date:   2015-11-18 10:34:17 +0200
categories: programming math vector
---

This article tries to explain vector math related to game development. This article is written by a mere mortal, and might contain errors and shortcuts. 

## What is a vector?

A vector is a list of floats. The amount depends on your dimensions, for example an Vector2 has x/y while a Vector3 has x/y/z. Conceptually a vector might be 2 different things.

> People that are much smarter then me prefer to call floats [Scalars][scalars]. I however do not.

## Why do we need vectors?

To represent a location, dummy!

## How do i use them in my game?
What follows is a list of common scenarios in a 2d game. Most math is applicable in 3d too.

We're assuming the following base class. Every entity must have some sort of position defined by an X and Y. Whenever the `Update` method is called we will move him.

{% highlight C# %}
class Entity
{
    public Vector2 Position;

    void Update()
    {

    }
}
{% endhighlight %}

### Moving your guy

It might sound simple, but you can add vectors and multiply vectors.

{% highlight C# %}
Vector2 position;
float speed;

void Update() 
{
    this.Position += new Vector2(1,0) * speed *deltaTime;
}
{% endhighlight %}

After this we multiply it by our speed. 

You'll also notice we also multiply the displacement by a `deltaTime`, which is the time in seconds since the last Update. This makes sure we are moving at 1 unit per second and not per frame, because frame rates can differ greatly. It is very important to make sure your game is independant of the frame rate!

### Rotating our guy

Since moving only right would make for a pretty crabby game, lets see if we can free our protagonist and make him move in multiple directions. We're aiming for a classic 2D top down experience like GTA2

{% highlight C# %}
float rotation;

void Update() 
{
    if(isKeyDown(Keys.Left))
        rotation += Math.PI * deltaTime;
    else if(isKeyDown(Keys.Right))
        rotation -= Math.PI * deltaTime;

    Vector2 delta;

    if(isKeyDown(Keys.Up))
        delta = getForward();
    else if(isKeyDown(Keys.Down))
        delta = -getForward();
    else
        delta = Vector2.Zero;

    this.Position += delta * deltaTime;
}

Vector2 getForward() 
{
    return new Vector2(
        Math.Sin(this.rotation), 
        Math.Cos(this.rotation)
    );
}

{% endhighlight %}

### Distance between another entity

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

### Moving towards another enity

Say we want to move an enemy towards our hero, how would we do that?

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


### Am i looking towards another guy?

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

This concludes the first post!

[scalars]: http://en.wikipedia.org/wiki/Scalar_(mathematics)