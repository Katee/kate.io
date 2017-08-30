---
title: "Weird Python Integers Part II: Constants in Bytecode"
date: 2017-08-24T22:05:50-04:00
uuid: "43ffbd20-75cb-433b-ac66-c542e33ff490"
slug: "python-constants-in-bytecode"
tags: ["python"]
categories: ["star"]
image: "images/posts/python-constants-in-bytecode.png"
image_width: 696
image_height: 355
---
This is a quick follow-up to [Weird Python Integers]({{< ref "posts/2017-08-22-weird-python-integers.md" >}}). While writing that post, I saw some Python behaviour I didn't understand. Then I read some comments by [kmill and squeaky-clean](https://news.ycombinator.com/item?id=15094345) in response to my original post, and now I can explain Python's behaviour.

If you don't know about the "small integers" table in Python, I recommend reading the original post first or this might be confusing.

<hr />

Given the small integers table, these lines make sense to me:

{{< highlight python3 >}}
>>> 100 is 100
True
>>> (10 ** 2) is (10 ** 2)
True
{{< /highlight >}}

* `100` is in the small integers table
* `(10 ** 2)` is evaluated to `100` which is the same number in the small integers table.

Even these lines make sense:

{{< highlight python3 >}}
>>> x = 1000
>>> y = 1000
>>> x is y
False
>>> (10 ** 3) is (10 ** 3)
False
{{< /highlight >}}

Both `x` and `y` are too large to be in the small integers table, so they are not the same object.

**However, this did not make sense to me:**

{{< highlight python3 >}}
>>> 1000 is 1000
True
{{< /highlight >}}


## Figuring it out with `dis`

Clearly something different is happening for `1000 is 1000` vs. `(100 ** 2) is (100 ** 2)`. I'm not familiar with Python's disassembler, but I know it exists so I tried it out:

{{< highlight python3 >}}
>>> import dis
>>> def disassemble(code_str):
...   dis.dis(compile(code_str, '', 'single'))
...
>>> disassemble('1000 is 1000')
  1           0 LOAD_CONST               0 (1000)
              2 LOAD_CONST               0 (1000)
              4 COMPARE_OP               8 (is)
              6 PRINT_EXPR
              8 LOAD_CONST               1 (None)
             10 RETURN_VALUE
>>> disassemble('(10 ** 3) is (10 ** 3)')
  1           0 LOAD_CONST               3 (1000)
              2 LOAD_CONST               4 (1000)
              4 COMPARE_OP               8 (is)
              6 PRINT_EXPR
              8 LOAD_CONST               2 (None)
             10 RETURN_VALUE
{{< /highlight >}}

Looking at this output, I was still confused why `(100 ** 2) is (100 ** 2)` wasn't `True`. The [Python peephole optimizer](https://github.com/python/cpython/blob/5fd33b5926eb8c9352bf5718369b4a8d72c4bb44/Python/peephole.c#L247-L249) changes my line to only be a comparison of constants just like `1000 is 1000`.

Do you see my mistake? I had completely missed the `0` in `0 (1000)` vs the `3` in `3 (1000)`. Both lines load and compare constants, just not the *same* constants. Now it is very clear that they are loading different constants:

{{< highlight python3 >}}
>>> def disassemble_with_constants(code_str):
...   code = compile(code_str, '', 'single')
...   dis.dis(code)
...   print("Constants:", code.co_consts)
...
>>> disassemble_with_constants('1000 is 1000')
  1           0 LOAD_CONST               0 (1000)
              2 LOAD_CONST               0 (1000)
              4 COMPARE_OP               8 (is)
              6 PRINT_EXPR
              8 LOAD_CONST               1 (None)
             10 RETURN_VALUE
Constants: (1000, None)
>>> disassemble_with_constants('(10 ** 3) is (10 ** 3)')
  1           0 LOAD_CONST               3 (1000)
              2 LOAD_CONST               4 (1000)
              4 COMPARE_OP               8 (is)
              6 PRINT_EXPR
              8 LOAD_CONST               2 (None)
             10 RETURN_VALUE
Constants: (10, 3, None, 1000, 1000)
{{< /highlight >}}

Even after being optimized the constants used in the calculation are still kept. Also there is no optimization to make two identical constants not in the small integers table use the same object.
