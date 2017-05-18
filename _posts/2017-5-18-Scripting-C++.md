---
layout: post
title:  "Scripting C++"
date:   2017-5-18 00:00:00 +0200
tags: ['c/c++', 'lua']
author: "Louis Hofer"
---

Starting in the third semester, I've been tasked with delving into a personal programming project.
In my previous work on the game engine I discovered the benefit of using script file inputs to a game.
The XML written in the game engine project (and SKKBP2D3D) allowed for reconfiguration without the slow compile times associated with C++ changes.

This was one of my favorite portions of the development process of the engine so I figured that I'd dive deeper into external scripting and how its used in conjunction to C++.
While I wrote some basic arithmetic functions that could be interpreted from text in XML using shunting yard, I'd rather not reinvent the wheel completely.

So I went online and found that Lua is used by several games and studios for describing behavior and tweaking values during game development.
It can also be used in production because it can be compiled into a human-unfriendly format to disuade cheating (unlike our XML file system).

I've also been lacking in exposure to AI techniques as practically all of my projects here at FIEA (aside from A Heart of Tin) fail to feature AI.
So I am intending to have the behavior that I specify in Lua control game entities in some way.

One simple and practical way to accomplish this is to create a finite state machine for entity behavior, whose state behaviors are defined in Lua.
These Lua scripts can be invoked by each C++ entity depending on which state the entity is currently in.
The state of the entity may even be determined by script execution.

My goal for this project is to have entities acting on screen differently depending on states described in Lua.
I would like to especially demonstrate the ability of the system to change behavior in Lua without recompiling the C++ code used to generate the game binary files.

As of now, I have selected a lightweight library named [Kaguya](https://github.com/satoren/kaguya) that allows for the easy wrapping of Lua states and values with C++ values.
I have created a LuaManager class that wraps their structures for easier usage of commonly used functionality but still exposes for when I do not wish to rewrite things myself (such as their type-agnostic data structures and template classes).

A test main method like this:
```c
using namespace LouisCrypt;

int main()
{
	LuaManager lua;

	lua.EvaluateScript("bindtesting.lua");

	kaguya::LuaTable entity = lua.CreateTable("entity");
	entity["x"] = 12;

	lua.GetState()["LuaFunction"].call<void>();

	std::cout << "Attempting to call a function in Lua script with arguments and return value:" << std::endl;

	std::cout << lua.GetState()["Sum"].call<int>(1, 4) << std::endl;

	system("PAUSE");
	
    return 0;
}
```

Outputs the following:
```
	This Lua function was invoked from C++
	The value from C++ is: 12
	Attempting to call a function in Lua script with arguments and return value:
	Adding the values 1 and 4 from C++ in Lua: 5
	Returning the value to C++
	5
Press any key to continue . . .
```