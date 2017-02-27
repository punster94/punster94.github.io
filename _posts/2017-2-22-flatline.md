---
layout: post
title:  "Flatline"
date:   2017-02-22 00:00:01 +0200
tags: ['C#', 'Unity']
author: "Louis Hofer"
---

Our third rapid prototype project in the FIEA curriculum stole gave us some experience in something many game developers actively avoid.
This project required that we make a game designed for use on mobile devices.
I worked on a team of five as one of two programmers.
This game was my first game that made use of 3D animations (and 2D texture animations for that matter).
This gave me a general look at the pipline and usage of elements like this.
I was also exposed to new forms of input and, although Unity made it incredibly easy to grab that information, a lot of thinking and math went into translating screen-space touches to a 3D position on a projected plane.

Flatline is a frantic tower-defense game where you swipe to build temporary walls to protect your heart from bacterial attack.
The game procedurally generates six waves of three different enemy units to attack the heart.
Each wave is has more units than the last, and each unit has a particular chance of being picked when generated.
Most enemies shoot bullets towards the heart that must be deflected or the heart will take damage.
Deflected bullets bounce away from the wall, potentially hitting and destroying attacking units.
The wave is cleared when all attacking units are destroyed, and bonus points are awarded for completing in a timely manner.

Powerups randomly spawn out of enemy units and can be deflected into the heart to be picked up.
Some heal, some slow down the game, and some give a short time of limitless wall-drawing.
When the heart reaches a certain health level it can release a shockwave ability to destroy all enemies currently visible in the round.

Below is a video of the final game.
You can also find the code and assets we used in the game by following the link in the projects tab or clicking <a href="https://github.com/punster94/Flatline">here</a>.

<div style="position:relative;height:0;padding-bottom:56.25%"><iframe src="https://www.youtube.com/embed/e5VH0N9M0oc?ecver=2" style="position:absolute;width:100%;height:100%;left:0" width="640" height="360" frameborder="0" allowfullscreen></iframe></div>