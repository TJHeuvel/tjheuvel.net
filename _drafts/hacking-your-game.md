---
title:  "Hacking your game"
date:   2015-12-08 10:34:17 +0200
categories: programming gamedev
---

#Lets do 2 parts, first one how to hack second how to fix. 

You have your awesome game all ready to go, though its being hacked. Lets see what we can do about this.

You should have some vague idea how computer memory works. Whenever you define a variable some block is reserved just for that, and you can read and write from this block. For example, when you write `int health` what is going on is that a block of 4 bytes is allocated. The whole problem is that not only you but also somebody else could write to this.

## Hacking yourself

You can try to do this on your own game but we'll use an example here. 

## What to do now?
The first step is mass paranoia. Although a bit difficult but accept the fact that this can be done. This can actually go even further, 

## Is all lost?

Well no, lets see what we can do about it. 

### Dont trust the client.
Do you have some server? Great, that guy is trustworthy. Dont trust anything sent by the client, it lies cheats and steals!