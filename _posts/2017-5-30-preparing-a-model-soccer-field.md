---
layout: post
title:  "Preparing A Model Soccer Field"
date:   2017-5-30 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

My initial plans have changed a bit with respect to my personal project.
Following the path of scripting in lua while developing AI in c++ left me with few options.
I could roll my own rendering and physics for the purposes of visualization or I could find an engine that natively supports lua.
I tried my luck with the latter, exploring both Lumberyard and OGRE.
In the end I decided to plug the simulation into Unreal, as I am familiar with its usage so I wouldn't have to learn the engine in addition.
That also means that the lua scripting component of this project should probably be dropped, as its functionality could be provided by blueprint value modifications or even blueprint event visual scripting.

Moving on, the first portion of my plan involves modeling out the soccer field, and all entities that will be in play during the simulation.
I've written c++ classes to represent each type of entity and implemented some basic functions like the ASoccerBall::Kick method.
Each entity to be used in the field has been created as a blueprint instance that inherits from its corresponding c++ class, such as the blueprint BP_RedTeam that inherits from the ASoccerTeam class.
I've also set up collision on entities like the ball and walls.
Each entity also exists as an AChildActorComponent of what it physically belongs in so that I can easily change the components of one blueprint to affect all entities that contain that blueprint.

Here's what the current field looks like:
![field and components](images/field%20and%20components.PNG "Field and Components")

Example of :
```c

```