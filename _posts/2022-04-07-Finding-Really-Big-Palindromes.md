---
layout: post
title:  "Finding Binary & Decimal Palindromes"
date:   2022-04-08
visible: 0
usemathjax: 1
---

Recently I've had my 33rd birthday and since I'm a nerd I also checked the binary representation of that number, which is 100001.

OMG, I thought to myself, 33 is a palindrome in both bases! I immediately checked when my next palindromtastic birthday would be:

_99_

That's a tad over my life expectancy, but arguably reachable. After that, the numbers get big quite fast. Exponentially fast. The [On-Line Encyclopedia of Integer Sequences](https://oeis.org/) has [an entry](https://oeis.org/A007632) for the sequence of these numbers, with a list of the [147 known elements](https://oeis.org/A007632/b007632.txt). The largest known number being 9335388324586156026843333486206516854238835339 with 46 decimal and 153 binary digits. That's quite large indeed.

Time to find even bigger palindromes!

<!--more-->

## Method

For quickly iterating over algorithms I've used python, only switching to a more performant language (Rust) when the algorithm stabilised. It is tempting to do all kinds of low level optimisations, but these make the code more complex and have a far lower benefit than improving the algorithm. In this problem - working less is orders of magnitude better than working faster.

In order to compare the performance of different algorithms I let them run for 10 minutes and recorded when each palindrome was found. The results are then plotted on a graph with logarithmic scales (logarithm of the number found as function of logarithm of the time it took until it was found).

## Naive approach

The most straightforward approach to the problem is simply to iterate over all numbers and check for each one whether it is a palindrome in both bases or not:

```python
def check_if_palindrome(num):
    bin_str = bin(num)[2:]
    dec_str = str(num)
    if bin_str == bin_str[::-1] and dec_str == dec_str[::-1]:
        palindrome_found(num)

def main():
    i = 1
    while True:
        check_if_palindrome(i)
        i += 1
```

<<<<add results>>>>

## Naive²

Instead of going through every number, we can only go through the decimal palindromes and check if each of them is a binary palindrome as well. Every even-lengthed palindrome can be constructed by taking a number, mirroring it and sticking the mirrored bit back on the original number.

For instance, the palindrome 123321 is 123 glued together with 321. For odd-lengthed palindromes we just need to drop a digit when gluing the numbers together: 12321 is 12 (123 without the last digit) glued together with 321.

So in order to iterate through all the decimal palindromes we still iterate through every number, but instead of checking if the number is itself a decimal palindrome, we use the above method to create two decimal palindromes out of it:

```python
def check_if_binary_palindrome(num):
    bin_str = bin(num)[2:]
    if bin_str == bin_str[::-1]:
        palindrome_found(num)

def main():
    i = 1
    while True:
        check_if_binary_palindrome(int(str(i) + str(i)[::-1]))
        check_if_binary_palindrome(int(str(i)[:-1] + str(i)[::-1]))
        i += 1
```

<<<<add results>>>>

## Pruning

That was a significant improvement, but obviously we want to be even faster, which means doing even less work. For this we are going to remodel the problem as a series of tree searches. Each such search is only looking for palindromes of a given decimal length.

In the tree, the root is empty and children are created by adding a digit on the right of the contents of their parent. For instance, the number 23 has 10 children - the numbers 230 to 239. The root is a special case, since it only has 9 children - we don't want numbers with 0 as their most significant digit.

Notice that the $$n$$-th layer of this tree (staring with the root on layer 0) contains all the numbers of decimal length $$n$$.

In the search for a palindrome of decimal length 6, we can do a depth first search up to a depth of 3 to generate all 3-decimal-digit numbers, and from them all 6-decimal-digit palindromes. The same process can be done for any other length $$n$$, by going up to depth $$\lceil \frac{n}{2} \rceil$$.

This search can be implemented using recursion in the following way:

```python
def find_palindromes(current_digits, decimal_length):
    if len(current_digits) * 2 >= decimal_length:
        digits_remaining = decimal_length - len(current_digits)
        decimal_palindrome = int(current_digits[:digits_remaining] + current_digits[::-1])
        check_if_binary_palindrome(decimal_palindrome)
        return

    if len(current_digits) == 0:
        next_digits = range(1, 10)
    else:
        next_digits = range(10)

    for next_digit in next_digits:
        new_digits = current_digits + str(next_digit)
        find_palindromes(new_digits, decimal_length)

def main():
    decimal_length = 1
    while True:
        find_palindromes("", decimal_length)
        decimal_length += 1
```

The motivation in this change is that now we can add logic to exclude entire parts of the tree as we go through it, if we know that they contain no binary palindrome. For instance, the most significant bit of all binary palindromes is 1 (it can't be 0, and there is no other option), so the least significant bit must also be 1, making all binary palindromes odd. All odd numbers, have an odd least significant decimal digit, and if they are palindromic, the same digit is their most significant decimal digit.

In other words, the most significant decimal digit of all binary-decimal palindromes has to be one of 1,3,5,7,9 and we can adjust our code accordingly, changing:
```python
    if len(current_digits) == 0:
        next_digits = range(1, 10)
```
to:
```python
    if len(current_digits) == 0:
        next_digits = range(1, 10, 2)
```

<<<<add results>>>>

## Pruning more

The cool thing about the above pruning is that it can be generalised to more than just the first bit/digit.
I've learned in school (and probably you have as well) that a number is divisible by 4 if its two least significant decimal digits form a number that is divisible by 4. For instance, we know that 123456 is divisible by 4, since 56 is. What I haven't learned in school is that this is also true for the remainder from division by 4: 1234567 mod 4 = 67 mod 4 = 3. This works because 100 is divisible by 4, and so are all higher powers of 10, so any change to decimal digits higher than the least significant 2, only changes the number in multiples of 4 - leaving the remainder unchanged.

Similarly, the remainder of any number from division by 8 is equal to the remainder from the division of just the 3 least significant digits. That's because 1000 is divisible by 8. This continues with 16, 32, 64,...,2<sup>$$k$$</sup>, requiring 4, 5, 6,...,$$k$$ digits respectively. In binary representation, the remainder from dividing by 2<sup>$$k$$</sup> is simply the $$k$$ least significant bits.

In the context of our search, this means that once we know the $$k$$ least (and most) significant decimal digits of a decimal palindrome, we also know its $$k$$ least (and most) significant bits if it is also a binary palindrome.

At every step as we go down the tree, our node determines the range of palindromes we will generate under it. If the decimal length of the palindrome is 7 and we are in the node containing the number 35, all the decimal palindromes we will generate below this node are in the range [3500053, 3599953]. The 2 least significant bits in 53 are 01, therefore we know that we are looking for a binary palindrome of the form 10...01.

All numbers in the range [3500053, 3599953] have 22 bits, so we know that we are looking for a binary palindrome of the same length, specifically, one in the range [1000000000000000000001<sub>b</sub>, 1011111111111111111101<sub>b</sub>] = [2097153, 3145725]. Here is where it gets interesting, the ranges do not overlap! Therefore no decimal palindrome in that range is also a binary palindrome, and we can prune all the subtree under 35 from our search.

{% comment %}
If not all numbers in the range have the same length in bits, we just take the minimal number of bits for the lower bound and the maximal number of bits for the upper bound. For the node containing the number 52 in a search with decimal length 6, we get the decimal range [520025, 529925], which contains numbers with both 19 and 20 bits. The 2 least significant bits in 25 are 01, therefore our range for the binary palindrome will be [1000000000000000001<sub>b</sub>, 10111111111111111101<sub>b</sub>] = [262145, 786429] (which have 19 and 20 bits respectively). This time the two ranges do overlap, therefore we continue the search as usual in the node's children.
{% endcomment %}

If not all numbers in the range have the same length in bits, we do not prune, as this scenario doesn't happen often enough to justify more complicated code.

In python, it looks like this:

```python
def is_pruned(decimal_digits, decimal_length):
    changing_decimal_length = decimal_length - 2 * len(decimal_digits)
    min_decimal_palindrome = int(decimal_digits + "0" * changing_decimal_length + decimal_digits[::-1])
    max_decimal_palindrome = int(decimal_digits + "9" * changing_decimal_length + decimal_digits[::-1])

    binary_length = min_decimal_palindrome.bit_length()
    if max_decimal_palindrome.bit_length() != binary_length:
        return False

    binary_digits = bin(min_decimal_palindrome)[-len(decimal_digits):]

    changing_binary_length = binary_length - 2 * len(binary_digits)
    min_binary_palindrome = int(binary_digits[::-1] + "0" * changing_binary_length + binary_digits, 2)
    max_binary_palindrome = int(binary_digits[::-1] + "1" * changing_binary_length + binary_digits, 2)

    return min_binary_palindrome > max_decimal_palindrome or max_binary_palindrome < min_decimal_palindrome
```

<<<<add results>>>>


## Extra Pruning

Our pruning method uses our knowledge of the least significant bits. This time, let's look at the most significant bits.

In the example, we deduced that any palindrome we might generate
