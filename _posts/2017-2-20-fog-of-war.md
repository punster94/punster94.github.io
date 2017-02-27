---
layout: post
title:  "Fog of War"
date:   2017-02-20 00:00:00 +0200
tags: ['AS3', 'Animate']
author: "Louis Hofer"
---

Through my experiences at FIEA, I've had many opportunities to work on games with variable dialog tree structures.
I've worked on three games in particular using a dialog structure that I designed:
1. The Twisted Mystery of the Missing Brisket
2. Six Days Until Summer
3. OZ

My techniques in approaching this design challenge have changed as I gained more experience working on this particular type of feature.
In TTMMB, I began my approach by saving out XML file with two separate lists of structures:
* One for accusation dialog when you, the detective, accuse a particular character at key points in the game
* And one for storing general dialog you encounter when you speak with characters or objects around the environment to find clues

This game was interesting in that it stored keywords in each statement of a dialog.
These keywords would be appended to a list of seen keywords to determine if certain dialog options could be chosen in a scenerio.

Given that the demands of every game are different, my approach in Six Days Until Summer had to change.
We no longer cared about knowing which keywords were found, but rather wanted to attach a score making a certain choice in a dialog situation.
We also needed to transition our dialog options based on the state of the story, leading to the addition of several values on the statement structure.
This structure was also still stored in XML.

While XML is easier to read and write in than a piece of code, it is still quite unnatural to develop dialog in and as such is prone to frequent error.

In OZ, an Unreal game, I took the approach of using the capabilites of USTRUCTs to create a dialog system that was easily modified in a blueprint.
This allows the designers to more clearly edit without fear of syntactical failure.

<div style="position:relative;height:0;padding-bottom:56.25%"><iframe src="https://www.youtube.com/watch?v=yMx4sQ3Lrvs" style="position:absolute;width:100%;height:100%;left:0" width="640" height="360" frameborder="0" allowfullscreen></iframe></div>