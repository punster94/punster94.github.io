---
layout: post
title:  "Building A Game Engine"
date:   2017-28-4 00:00:00 +0200
tags: ['c/c++', 'XML']
author: "Louis Hofer"
---

I heard that the second semester here at FIEA would be tough on programmers.
We start a new class under a different instructor with high expectations in an attempt to whip our c++ skills into shape.
Our goal working through the semester was to learn the ins and outs of a game engine by implementing one ourselves.
While our engines did have to conform to particular API's, we were free to design how we saw fit.
This gave rise to some interesting solutions to the presented problems.

While the end result of this project is not flashy (since its composed merely of unit tests), I did learn a lot about what many game engines do behind the scenes.
I'll describe a few particularly insightful components of my engine.

Firstly, we built data structures from the ground up with a focus on templated classes that can be used to store many types of data.
These singly linked lists, vectors, and hashmap classes made possible the storage of any type if data but in itself was still statically typed.
We needed to know what type of data we were storing inside the class we were storing it in, which is not great when you want to specify information outside of code.
This is something that game engines are particularly useful for, as you can often specify the size and type of data to act on and contain in some external data file that is easy to reconfigure.
To accomplish this, a Datum class was created that holds a type-agnostic pointer to an array of memory.
This type can be defined after construction and can even be reset.
With this, every bit of primitive data can be constructed as a Datum and set after the fact for easy parsing.
The parsing in itself was a rewarding process to wrap my head around.
Type agnostic data containers come in handy when appending additional data that need not be prescribed in a c++ class.

Additionally, I build action clases that invokes a particular function on update.
To couple this, a concurrent reaction system was put into place using the observer pattern that can invoke functionality when an event is thrown.
Both of these systems are abstracted to the point where they can be specified in XML within the game world.
What was essentially created is a scripting language that can be evaluated based on type agnostic data structures at runtime.
I even created an expression system that uses a shunting yard algorithm to evaluate basic arithmetic and logic functions of name lookups to these type agnostic data structures.
The result is XML that looks like this:

<world name='World 1'>
	<float name='number' value='5.0'/>
	<float name='otherNumber' value='3.0'/>
	<if>
		<expression value='number > otherNumber'/>
		<then class='ActionList'>
			<action name='Creator' class='ActionCreateAction'>
				<string name='instance' value='Created Action'/>
				<string name='class' value='ActionList'/>
			</action>
		</then>
		<else class='ActionList'>
			<action class='ActionList' name='Created Action'/>
			<action name='Destroyer' class='ActionDestroyAction'>
				<string name='instance' value='Created Action'/>
			</action>
		</else>
	</if>
</world>

When the update method of that World instance is invoked, the then block of the if runs, creating a new Action named "Created Action".
If the value of otherNumber is set to 6.0, for instance, then the next update of the World will invoke the else block of the if, destroying its sibling Action named "Created Action".
All of this functionality can be implemented outside of c++, similarly to how an engine like Unreal would specify functionality in a blueprint.

The source for this project is available in the Projects tab or on my Github account page.
