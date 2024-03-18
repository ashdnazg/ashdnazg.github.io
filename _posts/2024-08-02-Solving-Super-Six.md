---
layout: post
title:  "Solving Super Six"
date:   2024-08-02
image: 'assets/img/super-six/rio.jpeg'
visible: 1
usemathjax: 1
---

Super Six (a.k.a Rio and plenty of other names) is not a good game, it's repetitive, heavily luck dependent, and repetitive.
Nevertheless, it has two major qualities - it's short and simple. These qualities translate into a much more interesting one - it's solvable for two players!

While the game itself is a mediocre subject for a blog post, the maths behind solving it are not only interesting themselves, but they can also be used to solve other two-player games.

<!--more-->

## The game

As much as I would like to spare you from learning the rules of a game I don't recommend, I'm afraid understanding the game is a prerequisite for understanding the rest of the post. Luckily for you, there's not much to learn. While there are a few versions of the rules, the core mechanics of the game are the same in all of them.

{%
    include image.html src='assets/img/super-six/rio.jpeg'
    caption="The setup of <a href='https://boardgamegeek.com/boardgame/30640/rio'>Rio</a>, one of the alternative names of Super Six."
    alt="A board with five sockets, numbered 1 to 5 and a bigger hole numbered 6. In holes 3 and 4 there are short pegs. More pegs are next to the board, together with a 6-sided die."
%}

In a game of Super Six, (in our case 40) pegs are distributed between (in our case two) players and the first to get rid of all their pegs wins. Between the players sits a board with 5 peg-sized sockets numbered 1-5 and a bottomless pit numbered 6. At the beginning of their turn, a player rolls a 6-sided die which results in one of 3 outcomes:

<ol type="i">
<li>If they rolled a number for which the socket is empty, they put one of their pegs into that socket.</li>
<li>If they rolled a number for which the socket is occupied by a peg, they take that peg.</li>
<li>If they rolled a 6, they throw one of their pegs into the bottomless pit.</li>
</ol>

After rolling that die, as long as they haven't taken a peg (outcome __ii__), the player may decide whether they wish to roll again or to end their turn and let the next player play.

Therefore, every turn ends when one of the following happens:
1. The player took a peg.
2. The player decided to end their turn.
3. The player got rid of all their pegs and won the game.

And that's it!

## This is a great game for analysis

The simplicity of the game is reflected in two mathematical properties.

First, the state space is quite small. If starting with 40 pegs, there are a total of 4115 situations in which a player can find themselves.

Second, the player has only two options to decide between at any of these states.

This will later result in the comparatively tiny complexity of solving the game.

## What is a solution?

The idea of [solving a game](https://en.wikipedia.org/wiki/Solved_game) is to prove what choices a player should make to maximise their chance of winning, against an opponent who's also familiar with the solution and tries to maximise their own chance of winning.

In Super-Six, that means telling the player whether they should roll the die, based on the state of the game. The most straightforward way to do it is to calculate the probability of winning for both options (rolling and ending the turn), the best option would then be the one with the higher probability.

As we need to calculate these probabilities for every state, we will define them as functions on a given state $$S$$:

$$P_{roll}(S)$$ - The probability of winning if the player rolls the die when in state $$S$$.

$$P_{end}(S)$$ - The probability of winning if the player ends their turn when in state $$S$$.

## What is a state?

We can describe a state as a triplet $$(b, c, o)$$ of 3 numbers:
1. $$b$$ - The number of pegs on the board. Note that it doesn't really matter which sockets are full and which are empty since they're all symmetric.
2. $$c$$ - The number of pegs belonging to the current player, whose turn it is.
3. $$o$$ - The number of pegs belonging to the opponent.

For example, at the beginning of the game, there are no pegs on the board and each player has 20 pegs, therefore $$S=(0, 20, 20)$$. Later in the game, we can imagine a state where there are 4 pegs on the board, the current player has 6 pegs and the opponent has 2, giving us the state $$S=(4, 6, 2)$$.

## Calculations

We would like to find formulas for the two functions, with the hope of reaching some solvable equation system giving us the solution.

Let's start with the simpler case - $$P_{end}(b, c, o)$$ - the probability of winning when deciding to end your turn.

Since the game cannot end in a draw, the probability of winning equals the probability of the opponent losing. The end of your turn is exactly the beginning of the enemy's turn, therefore the probability of winning, when deciding to end your turn, equals the probability of your opponent losing, at the beginning of their turn.

Let's now derive the following:

$$P_{end}(b, c, o) = 1 - P_{roll}(b, o, c)$$

We are:

1. Using $$P_{roll}$$ since a player must roll at the beginning of their turn. Remember the opponent is also familiar with the solution,
so they have the same $$P_{roll}$$ as we do.
2. Swapping $$c$$ and $$o$$, since it's the opponent's turn.
3. Subtracting the probability from $$1$$, since we are interested in the probability of the opponent __losing__.

Now let's tackle the more complicated probability $$P_{roll}(b, c, o)$$ - the probability of winning when deciding to push your luck and roll the die. Recall the 3 outcomes from before:

<ol type="i">
<li>If they rolled a number for which the socket is empty, they put one of their pegs into that socket.</li>
<li>If they rolled a number for which the socket is occupied by a peg, they take that peg.</li>
<li>If they rolled a 6, they throw one of their pegs into the bottomless pit.</li>
</ol>

For each outcome we can describe the resulting state $$S'$$ and the probability of it occurring, $$p$$:

| Outcome              | p                 | S'                    |
|----------------------|-------------------|-----------------------|
| i - Rolled empty     | $$\frac{5-b}{6}$$ | $$(b + 1, c - 1, o)$$ |
| ii - Rolled occupied | $$\frac{b}{6}$$   | $$(b - 1, c + 1, o)$$ |
| iii - Rolled 6       | $$\frac{1}{6}$$   | $$(b, c - 1, o)$$     |

I recommend taking a few seconds to understand the contents of the table above and how they represent the outcomes.

In order to calculate $$P_{roll}$$, we also need the probability of winning for every outcome, $$P$$. That's where things start to become hairy.

After rolling a 6 (outcome __iii__), for instance, the player must _again_ decide whether they want to roll or not, which itself depends on whether the probability of winning when rolling is better than the one for ending the turn. We still don't know these probabilities, so let's define a new function:

$$P_{choice}(S)$$ - The probability of winning if the player chooses optimally whether they roll the die or not.

Mathematically, this means $$P_{choice}(S)=\max\left\{P_{roll}(S), P_{end}(S)\right\}$$

Now we can add the probability of winning, $$P$$, as a new column to the table:

|     | p                 | S'                    | P              |
|-----|-------------------|-----------------------|----------------|
| i   | $$\frac{5-b}{6}$$ | $$(b + 1, c - 1, o)$$ | $$P_{choice}$$ |
| ii  | $$\frac{b}{6}$$   | $$(b - 1, c + 1, o)$$ | $$P_{end}$$    |
| iii | $$\frac{1}{6}$$   | $$(b, c - 1, o)$$     | $$P_{choice}$$ |

Note that for outcome __ii__, we get $$P=P_{end}$$, since the player ends their turn if they rolled the number of an occupied slot.

Using conditional probabilities we get

$$P_{roll}(S) = p_i \times P_i(S'_i) + \\
p_{ii} \times P_{ii}(S'_{ii}) + \\
p_{iii} \times P_{iii}(S'_{iii})$$

and using our table:

$$P_{roll}(b, c, o) = \frac{5-b}{6} \times P_{choice}(b + 1, c - 1, o) + \\
\frac{b}{6} \times P_{end}(b - 1, c + 1, o)+ \\
\frac{1}{6} \times P_{choice}(b, c - 1, o)$$

Yeah... that's a bit of a mouthful. Don't worry though, we're not going to solve it manually, that's what computers are for.

## Examining our equations

In the previous section, we created 3 equations of the form $$P_{roll}=\ldots,\; P_{end}=...,\; P_{choice}=...$$, where each function depends on the other functions. Luckily the recursion is not endless, since if the player has no pegs left, the probability of winning is 1. We can represent it by setting $$P_{choice}(b, 0, o)=1$$ for all $$b, o$$.

If the equations were linear, our life would be simpler, but unfortunately for us, one of our equations involves a maximum operation:

$$P_{choice}(S)=\max\left\{P_{roll}(S), P_{end}(S)\right\}$$

Other than this pesky $$\max$$, though, it only involves linear terms, so we're gonna take a leap of faith and use a method for solving linear problems.

## Linear Programming

[Linear programming](https://en.wikipedia.org/wiki/Linear_programming) (or optimisation) is a nifty tool that tries to minimise or maximise a linear term, while preserving a set of linear constraints.

That might sound confusing, and the common notation uses all kinds of confusing matrices, but as hobbyist computer scientists we can just ignore all of that and look at the following example that hopefully makes things clearer.

For example, we could use this tool to find $$x$$ and $$y$$ for which $$2x+y$$ is minimal while $$x \geq 0,y \geq 0, x+y=1$$, and it would answer $$x=0,y=1$$.

In this example, we had:
1. Variables - $$x,y$$
2. A single objective - minimise $$2x+y$$
3. Constraints - $$x \geq 0, y \geq 0, x+y=1$$

We would like to map our problem into this model - define variables, create constraints in the form of linear equalities and inequalities, and choose the appropriate objective such that the solution gives us our probability functions.

We want to know the value of each of the three winning probability functions for each valid state, so for each valid state $$S$$, we define three variables:
1. $$p_{roll,S}$$ which represents $$P_{roll}(S)$$
2. $$p_{end,S}$$ which represents $$P_{end}(S)$$
3. $$p_{choice,S}$$ which represents $$P_{choice}(S)$$

The constraints for $$p_{roll,S}$$ and $$p_{end,S}$$ are easy to create - they're the equations we derived in the previous section, but what can we do about $$p_{choice,S}$$?

## That pesky $$max$$

We all know what a maximum of two numbers is - it's the larger number of the two. But we can also look at it from a different point of view.

* The maximum of two numbers is the smallest number that's larger or equal to both.

Let's sprinkle a little bit of mathematical notation:

* The maximum of two numbers $$x,y$$ is the smallest number $$m$$ such that $$m \geq x$$ and $$m \geq y$$.

Now that looks a bit like our model!
1. Variable - $$m$$ (note that $$x$$ and $$y$$ are constants here)
2. Objective - minimise $$m$$
3. Constraints - $$m \geq x, m \geq y$$

But that's just one maximum, and we have a bunch of $$p_{choice,S}$$ equations, each one containing its own $$\max$$.

We already have the variables for the maximums - $$p_{choice,S}$$ - and we can easily add the constraints $$p_{choice,S} \geq p_{roll,S},\; p_{choice,S} \geq p_{end,S}$$, so that's not a problem, but we can only choose a _single_ objective. We can't minimise each variable separately.

The intuitive solution is to minimise the sum of all maximums, which in our case $$\sum p_{choice,S}$$. It's not equivalent to minimising each one separately, but as hobbyist computer scientists, we have the superpower of not having to publish papers with rigorous proofs. We can just throw the pasta on the wall and see if it sticks.

After running the optimisation we can verify for each triplet of variables whether $$p_{choice,S}$$ is equal to either $$p_{roll,S}$$ or $$p_{end,S}$$. If this verification passes, we know that these are indeed maximums and we have solved the game.

## Did we succeed?

No :(

In the following plot, you can see the relative error of $$P_{choice}$$, or in other words, how far it is from the maximum of $$P_{roll}$$ and $$P_{end}$$ for the different values of $$b$$ (subplots), $$c$$ (x-axes) and $$o$$ (y-axes).

{% include plots/relative_error.html %}

Initially, everything is fine, but then the values of $$P_{choice}$$ when there are 4 pegs on the board start to drift from the maximum, reaching errors of more than 1%.

Due to the interdependence of the $$P_{choice}$$ variables (through the equations), minimising their sum as one unit did not constrain each one separately to be the maximum of $$P_{roll}$$ and $$P_{end}$$.

It seems that our superpower failed us. We forgot that even though we don't have to publish papers, we still need to find a valid solution if we want to salvage this more-than-half-written blog post.

## That pesky $$max$$, second attempt

Time to check if somebody encountered this issue before us. A quick search on Google brings us [this Stack Exchange answer](https://scicomp.stackexchange.com/a/20952) from 2015 (thanks Kevin!).

The answer suggests adding a binary variable, whose value can either be 1 or 0, representing the choice of whether we roll the die or not, respectively.

Let's denote this variable $$r_{b,c,o}$$.

We want to encode the following _if_ relationship into the constraints:

$$
p_{choice,S} = \begin{cases}
    p_{roll,S} & \text{if}\ \ r_{b,c,o}=1 \\
    p_{end,S} & \text{if}\ \ r_{b,c,o}=0
\end{cases}
$$

We start by adding the constraints from the previous section:

$$
p_{choice,S} \geq p_{roll,S} \\
p_{choice,S} \geq p_{end,S}
$$

as these are still true. Finally, we add the following magic constraints:

$$
p_{choice,S} \leq 1 - r_{b,c,o} + p_{roll,S} \\
p_{choice,S} \leq r_{b,c,o} + p_{end,S}
$$

Let's take a moment to digest this. If $$r_{b,c,o} = 0$$, these constraints become

$$
p_{choice,S} \leq p_{roll,S} \\
p_{choice,S} \leq 1 + p_{end,S}
$$

Together with $$p_{choice,S} \geq p_{roll,S}$$, this means that $$p_{choice,S}$$ must be equal to $$p_{roll,S}$$.

Conversely, If $$r_{b,c,o} = 1$$, the constraints become

$$
p_{choice,S} \leq 1 + p_{roll,S} \\
p_{choice,S} \leq p_{end,S}
$$

Similarly forcing $$p_{choice,S}$$ to equal $$p_{end,S}$$.

The crucial difference between this variable and the ones we defined before is that it's not continuous. Linear programming relies on the variables being continuous, therefore we have to change our model to Mixed Integer Linear Programming - linear programming with both continuous and integer variables. This has implications for the complexity of solving the problem, but that's the computer's problem, not mine.

What objective do we choose? It seems that we don't really need one. Our constraints already encode all of our equations, therefore we expect a single solution already without any optimisation. We can verify this by running the same solver multiple times with different objectives and getting the same result.

## Did we succeed this time?

Yes, we did! And here is a plot of our $$r_{b,c,o}$$ values, specifying whether we should or shouldn't roll for the different values of $$b$$ (subplots), $$c$$ (x-axes) and $$o$$ (y-axes).

{% include plots/should_roll.html %}

This post won't analyse these results as that would really overstay the post's welcome, but make sure not to miss the symmetry and the one surprising _Roll_ when there are 4 pegs on the board and both players have 1 peg.

The winning probability, $$P_{choice}$$ are also interesting:

{% include plots/p_choice.html %}

## Conclusions

Previously, a solution was available only for [up to 15 total pegs](https://arxiv.org/abs/2109.10700). Although the plots above go up to 40 total pegs, I've managed to run my code up to 130.

The Python code used for solving and creating the above plots is research-grade and not in the good sense. Nevertheless, it is
available [here](https://gist.github.com/ashdnazg/8c1f7510ac1087dcc54ee853390cc32c) in case anybody is curious. The first (failed) version of the code is [here](https://gist.github.com/ashdnazg/ec1a5129db7de039b69f9cda89ef15f1).

While it's nice that Mixed Integer Linear Programming can solve Super-Six and probably other simple games, I think it's more interesting that _regular_ Linear Programming is the wrong tool for the job. At first glance, it looked to fit the problem and the workarounds for its issues seemed plausible, but that wasn't enough. Math was right, I was wrong.

Hopefully, with the next project, I'll be able to find my errors before I write half the blog post. Time will tell :)
