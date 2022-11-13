---
layout: posts

title: "CSAPP - Lab 1. (32-bit) Data Lab"

categories: arch
tags:
  - arch
  - CSAPP

date: 2022-10-05 17:28:00 +09:00
last_modified_at: 2022-10-05 17:28:00 +09:00

# published: true

---

# Data Lab

A lab assignment to implement primitive operations with (mostly) bitwise operations only. `int` type is assumed to be 32-bit.

## Integer Coding

### Common Techniques

* an all-set bit mask: `~0`
* a bit mask from `x`, full of its sign bit: `x >> 31`

### isTmax

> Check whether the argument `x` is `INT_MAX` or not. can use `! ~ & ^ | +`.

We can focus on a unique feature of `INT_MAX`: its successor is the exact bit negation of itself - every bits of `x` and `x+1` are different.
The bitwise comparison for `x` and `x+1` can be implemented as `(x+1)^(~x)` (`(~(x+1))^x` works also), and this will be zero when `x` is equal to `INT_MAX`.

Actually, this feature is not unique - `-1` also fits to this description.
So we have to detect `-1` in a different way.

```c
int isTmax(int x) {
    int y = ~x;
    int a = !y; // 1 when x == -1
    int b = (x+1)^y; // nonzero unless x == -1 or x == Tmax
    return !(a | b);
}
```

One of plausible - but wrong - answers exploits the fact that `INT_MAX` plus one, which has equivalent bit expression of `INT_MIN`, is one of the two solutions twice of which make zero (in an overflowing manner).
```
    return !(x + x + 2) & !!(x + 1)
```

This will fail the test anyhow, because **the compiler(gcc) mercilessly optimizes the whole term to zero!**
For historical reasons, it seems like the compiler assumes that [no overflow should occur in signed integer](https://stackoverflow.com/questions/18195715/why-is-unsigned-integer-overflow-defined-behavior-but-signed-integer-overflow-is) and there is no way to make the term nonzero unless `x` is the overflowing value `INT_MAX`.
Despite that the overflow is intended to detect `INT_MAX`, the only way to tell the compiler to do so is to turn off the optimization, which violates the lab rule.

### negate

> Arithmetical negation using `! ~ & ^ | + << >>`

This is straightforward:

```c
int negate(int x) {
  return (~x) + 1;
}
```
### isAsciiDigit

> return 1 if `x` is the ASCII code range for digits(characters `0` to `9` - range `0x30` to `0x39`)

We can split the given range based on the bit pattern of it:
* `x` matches with `0x30`, except possibly right 3 bits(`0x30 ~ 0x37`)
* `x` matches with `0x38`, except possibly right 1 bit(`0x38 ~ 0x39`)

```c
int isAsciiDigit(int x) {
  int a = !((x ^ 0x30) >> 3); //0x30 ~ 0x37
  int b = !((x ^ 0x38) >> 1); //0x38 ~ 0x39
  return a | b;
}
```

### isLessOrEqual

> Implement `x <= y` with operations `! ~ & ^ | + << >>` only.

The basic idea is to examine the sign of `y - x`, as well as the possibilities of overflowing or underflowing.
While we can't directly use subtraction, equivalently we can use `y + (~x) + 1`.

```c
int isLessOrEqual(int x, int y) {
  int sx = x >> 31;
  int sy = y >> 31;

  int a = (sx & ~sy) & 1;
  int b = (~sx & sy) + 1;
  int c = ((y + (~x) + 1) >> 31) + 1;
  return a | (b & c);
}
```

### logicalNeg

> implement `!x` with operations `~ & ^ | + << >>`.

If there are at least one bit set, then zero. Otherwise one.

The first attempt is to OR all the bits using bitshift.

```c
int logicalNeg(int x){
  x = x | (x >> 16);
  x = x | (x >> 8);
  x = x | (x >> 4);
  x = x | (x >> 2);
  x = x | (x >> 1);
  return (~x) & 1;
}
```

On the other hand, we should consider the max operation counts -- up to 12 operations are allowed, and this solution saturated the limit.

An alternative solution, inspired from previous `isTmax` function goes like this: `0` has unique property that its arithmetic negation has the same 0 sign bit to itself.

```c
int logicalNeg(int x){
  return ((x | (~x + 1)) >> 31) + 1;
}
```

### howManyBits

> return the minimum number of bits required to represent `x` in two's complement.

This was the trickiest one - considering so many edges cases, designing logic without too much exceptional cases...

Let's start with some observations.
* If `x` is positive, return the highest set bit index(0-based) + 2.
* If `x` is zero, return 1.
* If `x` is negative, return the lowest bit index of leading 1s + 1, which is equal to the highest unset bit index + 2. Hence the result is equal for `~x`, which is positive.

We can preprocess `x` to merge all 3 cases into one:
* If `x` is positive, then `y = 2*x or 2*x+1`
* If `x` is negative, then `y = 2*(~x) or 2*(~x)+1`
* If `x` is zero, then `y = 1`

so that we only have to find the highest set bit index of `y` and just add 1.
(when `x` is nonzero, LSB is irrelevant since it is obviously not the highest set bit.)

```c
y = ( (x>>31) ^ x ) | 1;
```

Now we're not only trying to identify the highest bit index of `y`, but also construct its bit representation.
Let the index stored in `r`(initially 0) and we will determine the bits of `r`.
So we start with querying whether there is a set bit larger than or equal to 16th.

If a set bit was found in 16th~31th bits, then we can move on to next query for finding set bit on 24th~31st bits to determine the next bit of `r`. Otherwise, find set bit on 8th~15th bits.

Overall, we want to implement this sort of code with restricted operations:

```c
z = y >> 16;
if(z){
    r = r + 16;
    y = z;
}
```

the translated results would be:
```c
z = y >> 16;
f = !z + (~0); // if >= 16th bit exists then 1..1, otherwise 0.
r = r | (f & 16);
y = (~f & y) | (f & z); // y = (z ? z : y);
```

and repeat this code, replacing `16` with `8, 4, 2, 1` respectively.

Now the only 'flaw' of this code is that since we're using signed integer arithmetic, the right shift has possibly carried the sign bit of initial `y`.
Usually, we can solve it with declaring `y` as unsigned, but typecasting is prohibited in this lab.

This is actually not a problem, since if the initial `y` has a sign bit, then there actually IS a set bit in 16th~31st, 24th~31st, 28th~31st, 30th~31st, 31st range, and the nonzero-ness of `z` for each iteration won't be change regardless of arithmetical or logical right shift we are using.

The above four-line code, can be more compactified as following.

```c
b = !!(y >> 16) << 4; // if >=16th bit exists then 16, otherwise 0
r = r | b;
y = y >> b;
```

The full code is omitted.

## Float Coding

Skip for this blog. Since float coding rules for this lab has much degrees of freedom(including control flow), we can solve them way more easily.
