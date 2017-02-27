---
layout: post
title:  "The Twisted Mystery of the Missing Brisket"
date:   2017-02-24 00:00:00 +0200
tags: ['C#', 'Unity']
author: "Louis Hofer"
---

Our fourth rapid prototype project in the FIEA curriculum began my dive into dialog systems.
The focus of this project was on telling a good story in a game.
I worked on a team of six as one of two programmers.
With this project, it was essential to our game that we have a dialog framework set up such that we can edit it outside of the engine.
Keeping this in mind, we approached the situation using XML as a structured input to the game.

The Twisted Mystery of the Missing Brisket is a who-done-it game about a bandit at FIEA.
The characters are all inspired by students in cohort 13 here at FIEA (even I, King Louie).
You, as the investigator Jack, travel around the cohort space and ask students about anything they may know regarding the events in question.
Statements discovered during the game carry certain keywords that unlock new dialog options for any character with knowledge pertaining to that keyword.
After talking to a certain number of characters in a round, you will be taken to the accusation menu where you can pick any person you suspect of being the bandit.
Of course until you play through the whole game you cannot be sure of your choice, but accusing a person may provide some enlightening facts about their innocence or someone elses guilt!

Additionally, we developed a menu where you can see a log of statements spoken by each character.
You are also provided an association map so that you may draw blame lines from facts revealed by one character to the character it affects.