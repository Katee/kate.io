---
title: "Weird Python Integers"
date: 2017-08-22T17:00:27-04:00
uuid: "2cff8cc5-8554-4ab2-a1c3-c15bfc4d41d4"
slug: "weird-python-integers"
tags: ["python"]
categories: ["star"]
---
Note: all of this code was run on my machine using Python 3.6.1. Not everything will work the same if you test using Python 2.

{{< highlight python3 >}}
>>> a = 42
>>> b = 42
>>> a is b
True
>>> a = 316
>>> b = 316
>>> a is b
False
{{< /highlight >}}

That is suprising! It turns out that all "small integers" with the same value point to the same memory. We can use the Python built-in function `id` which returns a value you can think of as a memory address to investigate.

{{< highlight python3 >}}
>>> a = 128
>>> b = 256
>>> id(a)
4504844960
>>> id(b)
4504849056
>>> (id(a) - id(b)) / (a - b)
32.0
{{< /highlight >}}

It looks like there is a table of tiny integers and each integer is takes up 32 bytes.

{{< highlight python3 >}}
>>> x = 1000
>>> y = 2000
>>> id(x)
4508143344
>>> id(y)
4508143312
>>> id(x) - id(y)
32
{{< /highlight >}}

It looks like integers that aren't in the small integers table also take up 32 bytes. The `id` for these is way larger than for the small integers which means they are stored somewhere else.

{{< highlight python3 >}}
>>> id(x) - id(256)
3294288
{{< /highlight >}}

## Editing Integers?

What happens if we change the value of an integer in this table? Python has a module called [ctypes](https://docs.python.org/3/library/ctypes.html) that can be misused to directly edit memory. (We could also use a debugger but this way all the examples are in Python.)

{{< highlight python3 >}}
>>> import ctypes
>>>
>>> def mutate_int(an_int, new_value):
...   ctypes.memmove(id(an_int) + 24, id(new_value) + 24, 8)
...
>>> a_number = 7
>>> another_number = 7
>>> mutate_int(a_number, 13)
>>> a_number
13
>>> another_number
13
{{< /highlight >}}

Not only have we changed `a_number` and `another_number` but all new references to `7`:

{{< highlight python3 >}}
>>> for i in range(0, 10):
...   print(i)
...
0
1
2
3
4
5
6
13
8
9
{{< /highlight >}}

Even doing math with `7` no longer works correctly 🎉

{{< highlight python3 >}}
>>> 7
13
>>> 6 + 1
13
>>> 7 + 1
14
>>> (7 + 1) - 1
13
>>> 7 * 2
26
>>> 0b1111 ^ 0b1000
13
{{< /highlight >}}
