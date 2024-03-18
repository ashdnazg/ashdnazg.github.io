---
layout: post
title:  "Finding the Optimal Strategy by Minimising the Winning Probability"
date:   2024-03-18
visible: 0
usemathjax: 1
---

Super Six[^1] is not a good game, it's repetitive, heavily luck dependent, and repetitive.
Nevertheless, it has some qualities - it's short, simple and my nephew likes it. These (except the nephew one) translate into a much more interesting one - it's solvable for two players!

While the game itself is a mediocre subject for a blogpost, the maths behind solving it are not only interesting themselves, but they can also be used for solving other two-player games.

<!--more-->

## The game

As much as I would like to spare you from learning the rules of a game I don't recommend, I'm afraid understanding the game is a prerequisite for understanding the rest of the post. Luckily for you, there's not much to learn. While there are a few versions of the rules, the core mechanics of the game are the same in all of them.

In a game of Super Six, pegs are distributed between (in our case two) players and the first to get rid of all their pegs wins. Between the players sits a board with 5 peg-sized sockets numbered 1-5 and a bottomless pit numbered 6. In the beginning of their turn, a player rolls a 6-sided die which results in one of 3 outcomes:

<ol type="a">
<li>If they rolled a 6, they throw one of their pegs into the bottomless pit.</li>
<li>If they rolled a number for which the socket is empty, they put one of their pegs into that socket.</li>
<li>If they a number for which the socket is occupied by a peg, they take that peg.</li>
</ol>

After rolling that die, as long as they haven't taken a peg (outcome __c__), the player may decide whether they wish to roll again or to end their turn and let the next player play.

Therefore, every turn ends if one of the following happened:
1. The player took a peg.
2. The player decided to end their turn.
3. The player got rid of all their pegs and won the game.

And that's basically it! Here's a playable version if you'd like to have a go.

## This is a great game for analysis

The simplicity of the game is reflected in two mathematical properties.

First, the state space is quite small. If starting with 40 pegs, there are a total of N situations in which a player can find themselves.

Second, the player only has to decide between two options.

This will result later in the comparatively tiny complexity of solving the game.

## What is a solution?

[^1]: a.k.a Rio and plenty other names
