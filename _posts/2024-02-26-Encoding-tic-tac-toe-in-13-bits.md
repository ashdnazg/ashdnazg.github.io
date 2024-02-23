---
layout: post
title:  "Encoding tic-tac-toe in 13 bits"
date:   2024-02-26
visible: 1
usemathjax: 1
---

The other day I happened upon a post by [Chris Barrick](https://cbarrick.dev/) detailing how to [encode a tic-tac-toe game state in 15 bits](https://cbarrick.dev/posts/2024/02/19/tic-tac-toe), itself a response to an earlier post by [Alejandra González](https://github.com/blyxyas) detailing how to [encode a tic-tac-toe state in 18 bits](https://blog.goose.love/posts/tictactoe/).

In their posts, Chris and Alejandra mention 10 bits as the absolute minimum, since there are only 765 possible game states.
This number, however, takes mirroring and rotations into account[^1], while Chris' and Alejandra's solutions do not. If you include mirrorings and rotations, the number of states [jumps to 5477](https://stackoverflow.com/a/25358690) that gives us a theoretical minimum of $$\left\lceil{\log_2{5477}}\right\rceil=\left\lceil{12.419...}\right\rceil=13$$ bits.

For physicists, the difference between 13 and 15 might look negligible, but hey, consuming 15% more space is no joke.

<!--more-->

## The task

Our goal is to find an efficient way to encode a tic-tac-toe game state into a number and decode it back to the same state, which is:
1. Human readable - It shouldn't be a bunch of neural network weights. We should be able to reason why it works.
2. Not a lookup table - Generating the 5477 states ruins the fun.
3. Efficient - uses as few bits as possible for the number.

## Why It should be possible

Encoding inefficiencies can stem from three causes:
1. Multiple numbers decode into the same state. This cause does not apply to the solutions analysed in this post, so feel free to forget it.
2. Some Numbers are invalid and can't be decoded.
3. Some Numbers decode into invalid states - states which can't appear in a tic-tac-toe game.

Alejandra's solution was to encode each square into two bits, where 00 is a blank square, 01 is an X and 10 is an O. Chris has noticed that a square is never encoded into the bits 11. In other words, any number that represents a square with 11 is invalid. Chris' solution - using base 3 - eliminates exactly these invalid numbers.

Since invalid numbers are gone, our first course of action should be to determine which numbers decode into invalid states in Chris' encoding.

Recall that our task is to only encode states which can appear during a tic-tac-toe game. The following state, for instance, cannot appear in a tic-tac-toe game since it has more Os than Xs.

```
  | O |
- + - + -
O | X | O
- + - + -
  | O |
```

It is, however, encodable by both Alejandra's and Chris' solutions, since they have no constraints over the number of Os compared to the number of Xs.

Now that we've determined a set of invalid states we can go on to eliminate them.

## Enforcing constraints

First, let's define a few symbols:
* $$\#_X$$ - The number of Xs on the board.
* $$\#_O$$ - The number of Os on the board.
* $$\#_{blank}$$ - The number of blank squares on the board.
* $$\#_{non\_blank}=\#_X+\#_O=9-\#_{blank}$$ - The number of non-blank squares on the board.

In a valid state of a tic-tac-toe game, the number of Xs is either equal to or one more than the number of Os. This is equivalent to saying that the number of Os is always half of the number of non-blank squares, rounded down:

$$\#_O = \left\lfloor{\frac{\#_{non\_blank}}{2}}\right\rfloor$$

Such a relationship cannot be expressed if we encode each square separately, instead, we should try to encode multiple squares together.

For instance, we could describe a board by first determining which squares contain any symbol in them and then which of these symbols is an O. Each of these two parts is a [combination](https://en.wikipedia.org/wiki/Combination).

Using combinatorics, we can calculate that given a certain $$\#_{non\_blank}$$, there are $${9 \choose \#_{non\_blank}}$$ possible combinations for the positions of the non-blank squares and $${\#_{non\_blank} \choose \#_O}={\#_{non\_blank} \choose \left\lfloor{\frac{\#_{non\_blank}}{2}}\right\rfloor}$$ possible combinations for the positions of the Os within these non-blank squares. Encoding these two combinations completely encodes the board.

For instance, if we know that the non-blanks are in positions (0,2,5,7) (zero-based), and of which (1,2) are Os, that's enough information to draw the following board (Note that the combination of the Os, refers to the indices in the non-blanks combination which refer to positions 2 and 5):

```
X |   | O
- + - + -
  |   | O
- + - + -
  | X |
```

Luckily for us, encoding and decoding a combination into/from a number in $$\left[0, {n \choose k} - 1\right]$$ [is a solved problem](https://stackoverflow.com/a/3143594), so let's use its solution as a black box to get these two numbers: $$enc_{non\_blanks} \in \left[0, {9 \choose \#_{non\_blank}} - 1\right]$$ and $$enc_O \in \left[0, {\#_{non\_blank} \choose \#_O} - 1\right]$$.

We can now combine these two numbers to encode the entire board:

$$enc_{non\_blanks}\times{\#_{non\_blank} \choose \#_O}+enc_O$$

For instance, in the example above, $$\#_{non\_blank}=4$$ and therefore $$\#_O=2$$ and $${\#_{non\_blank} \choose \#_O}={3 \choose 2}=6$$. Our black box encodes the positions of the non-blanks into $$enc_{non-blanks}=31$$, and the positions of the Os encode into $$enc_O=3$$, giving us $$31\times6+3=189$$ as the encoding of the entire board.

## Fine print

It feels like we've solved everything, but there's still a pesky issue - throughout the previous section we assumed we know $$\#_{non\_blank}$$, but we haven't encoded this information yet!

There are 10 possible values for this number (0-9), so the straightforward way to do this would be to just add it to the previous encoding:

$$enc_{board}=\left({enc_{non\_blanks}\times{\#_{non\_blank} \choose \#_O}+enc_O}\right)\times10+\#_{non\_blank}$$

In our example, we had $$\#_{non\_blank}=4$$, therefore the full encoding would be: $$enc_{board}=189 \times 10 + 4=1894$$.

Here's the Python code for encoding and decoding, where `encode_comb` and `decode_comb` are the black box functions that encode and decode a combination respectively:

```python
def encode_board_naive(board):
    non_blanks = []
    for i, c in enumerate(board):
        if c != " ":
            non_blanks.append(i)

    count_non_blank = len(non_blanks)
    enc_non_blank = encode_comb(9, non_blanks)

    os = []
    for i, j in enumerate(non_blanks):
        if board[j] == "O":
            os.append(i)

    count_o = len(os)
    enc_o = encode_comb(count_non_blank, os)
    comb_o = math.comb(count_non_blank, count_o)

    enc_board = enc_non_blank * comb_o + enc_o

    return 10 * enc_board + count_non_blank

def decode_board(enc_board):
    count_non_blank = enc_board % 10
    enc_board /= 10

    count_o = count_non_blank // 2
    comb_o = math.comb(count_non_blank, count_o)
    enc_o = enc_board % comb_o
    os = decode_comb(count_non_blank, count_o, enc_o)

    enc_non_blank = enc_board / comb_o
    non_blanks = decode_comb(9, count_non_blank, enc_non_blank)

    board = [" "] * 9
    for i in os:
        board[non_blanks[i]] = "O"

    for i in non_blanks:
        if board[i] == " ":
            board[i] = "X"

    return board
```

Alas, this results in a maximum encoding of $$\log_2{16796}\approx14.04$$ bits.

Like a technical job interview question, the straightforward way is rarely the efficient way. As we did before, let's find our invalid numbers.

Consider the case where $$\#_{non\_blank}=0$$, which is true for only one state - the empty board at the beginning of the game. Since there's only one possibility, it must hold that $$enc_{non\_blanks}=enc_O=0$$, and therefore $$enc_{board}=\left(0\times1+0\right)\times10+0=0$$. If we choose any number other than 0 for either $$enc_{non\_blanks}$$ or $$enc_O$$, we'll get an invalid number. e.g., if we choose $$enc_{non\_blanks}=2$$, we get $$enc_{board}=\left(2\times1+0\right)\times10+0=20$$.

The underlying reason is that the number of possible boards varies wildly depending on $$\#_{non\_blank}$$:

| $$\#_{non\_blank}$$ | Possible boards - $${9 \choose \#_{non\_blank}}\times{\#_{non\_blank} \choose \left\lfloor{\frac{\#_{non\_blank}}{2}}\right\rfloor}$$ |
|---|------|
| 0 | 1    |
| 1 | 9    |
| 2 | 72   |
| 3 | 252  |
| 4 | 756  |
| 5 | 1260 |
| 6 | 1680 |
| 7 | 1260 |
| 8 | 630  |
| 9 | 126  |

We need $$log_2{1680}\approx10.71$$ bits to represent a board with $$\#_{non\_blank}=6$$, but only $$log_2{9}\approx3.17$$ bits to represent a board with $$\#_{non\_blank}=1$$. The leftover $$10.71-3.17\approx7.54$$ bits are left unused. In fact, if they're not 0, our number is invalid.

## Improve efficiency

Based on the table above, there's a total of 6046 boards our system represents, so the most efficient function we could find would be an encoding into $$\left[0,6045\right]$$.

Why don't we do just that? Let's define the following mapping:

| $$\#_{non\_blank}$$ | Possible boards | Range
|---|------|-----|
| 0 | 1    |$$\left[0, 0\right]$$|
| 1 | 9    |$$\left[1, 9\right]$$|
| 2 | 72   |$$\left[10, 81\right]$$|
| 3 | 252  |$$\left[82, 333\right]$$|
| 4 | 756  |$$\left[334, 1089\right]$$|
| 5 | 1260 |$$\left[1090, 2349\right]$$|
| 6 | 1680 |$$\left[2350, 4029\right]$$|
| 7 | 1260 |$$\left[4030, 5289\right]$$|
| 8 | 630  |$$\left[5290, 5919\right]$$|
| 9 | 126  |$$\left[5920, 6045\right]$$|

Returning to our example, we had $$\#_{non\_blank}=4$$ and the combinations were encoded as 189. The table tells us the range we should use is $$\left[334, 1089\right]$$, so we map 189 into this range by adding the base offset, and we get the new full encoding: $$enc_{board}=334+189=523$$.

We can calculate this offset for each $$\#_{non\_blank}$$ by summing up the number of possible boards in the rows above it.
```python
def offset(count_non_blank):
    total_count = 0
    for i in range(count_non_blank):
        total_count += math.comb(9, i) * math.comb(i, i // 2)

    return total_count
```
and our final encoding function becomes:
```python
def encode_board(board):
    non_blanks = []
    for i, c in enumerate(board):
        if c != " ":
            non_blanks.append(i)

    count_non_blank = len(non_blanks)
    enc_non_blank = encode_comb(9, non_blanks)

    os = []
    for i, j in enumerate(non_blanks):
        if board[j] == "O":
            os.append(i)

    count_o = len(os)
    enc_o = encode_comb(count_non_blank, os)
    comb_o = math.comb(count_non_blank, count_o)

    enc_board = enc_non_blank * comb_o + enc_o

    return offset(count_non_blank) + enc_board
```

This successfully encodes 6046 boards into the range $$\left[0,6045\right]$$, consuming $$\left\lceil{\log_2{6046}}\right\rceil=\left\lceil{12.562...}\right\rceil=13$$ bits. We're just above the optimum of 5477, costing us ~0.143 bits, which luckily stays within the confines of 13 bits.

## Ideas for improvement

Now that we have an almost optimal encoding, let's analyse why it's only an "almost".

Consider the following board:

```
X | X | X
- + - + -
  |   |
- + - + -
O | O | O
```

Our encoding converts it to the number 2749, even though it could never occur during a tic-tac-toe game. The game would have ended when the X player won by putting their third X in a row, not allowing the O player to even play their third turn, let alone complete a row of their own.

I haven't checked it, but I think eliminating all states which are unreachable due to the victory conditions, should bring us to the optimum.

If you find an elegant way to do that, please share your insights, preferably as another blog post in the chain :)

The encoding/decoding code can be found [here](https://gist.github.com/ashdnazg/5cca7de6bac0eef4532d0c635c69184a#file-encode_tic_tac_toe-py)

Good luck!

[^1]: The 765 figure considers the following two boards (and a few others) as the same state:

    ```
    X |   | O     O |   | X
    - + - + -     - + - + -
      |   |         |   |
    - + - + -     - + - + -
    X |   |         |   | X
    ```
