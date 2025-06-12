---
layout: page
title: Batting Game Prototype
permalink: /batting-game-prototype/
---

Overview
--------
The Batting Game Prototype is a game made for the Unity Learn course Create with Code 2. The challange was to make a short
game utilizing a collision script that would increment "something". That's plenty vague so I went with a little physics game
of hitting a ball into some "gems" for points. I used the "Free Sports Kit" assets from Sports-Actions and used
a cricket bat and ball asset.

Getting the physics to work was actually the hardest part. The collisions between the bat and ball needed to be "Continuous Dynamic".
Otherwise, they may move through eachother. I also hacked a silly way to get the bat swinging controls correct: not good code.
I added a bouncy physics material and that also helped get the effect I wanted.

So as mentioned, there are gems to collect and that will give you points. Goal is to get the most points after 5 rounds.
However! There's a twist! In this game, the number of gems that spawn is based on the number collected in previous rounds.
This is cumulative too, so getting gems early can lead to an exponential growth of points!
As of right now, there is no information displayed to the player on this mechanic, add that to the todo.

<img src="/Batting_Game/images/battingGame.png" alt="Gameplay" title="Gameplay so far"/> 
--------------

Goals
=====
The goals for this project are to have the following features:

1. Player can swing the bat once per round

2. Player hits the ball to collect gems to earn points

3. More gems spawn for each gem earned in previous rounds

4. UI to display score, rounds left, and a game over screen to restart

5. Persistent high score

6. Powerups that will grow the ball or spawn multiple balls to hit

There are many other things that could be added, but these are the basic learning features that
would fit well with such a project. It would also be cool to have an online highscore feature
to compare to others but that would be a lot more work and trouble than it's worth.

---------------

Status
======

The game meets the requirement for the Unity Learn lesson, but goals 5 and 6 are not implemented.
I may come back to do these things in the future, but it's not a high priority.

Todo
====
1. Add some UI denoting the "bonus gems" that are available to spawn when the player picks up gems
2. Add persistent high score. For solo-play, don't need a "name" field for the high score I don't think.
3. Add base powerup abstract class and two powerup subclasses


[jekyll-organization]: https://github.com/jekyll
