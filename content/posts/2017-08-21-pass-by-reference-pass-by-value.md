---
title: "Pass by value (which is sometimes a reference)"
uuid: "5e5df17c-95fd-41d4-9ef7-b35603200e3e"
date: 2017-08-21
slug: "pass-by-reference-pass-by-value"
tags: ["programming", "python"]
categories: []
---
**tl;dr: this post just contrasts how things are passed to functions in C and Python.**

Here's a contrived Python function that takes two arguments and modifies both:

{{< highlight python3 >}}
def modify_arguments(a_list, a_number):
  a_list[0] += 42
  a_number += 42
{{< /highlight >}}

Let's try it out in the Python REPL:

{{< highlight python3 >}}
>>> the_list = [0]
>>> the_number = 0
>>> modify_arguments(the_list, the_number)
>>> the_list
[42]
>>> # ^ Ok, `the_list` function argument was modified
>>> the_number
0
>>> # ^ WHAT? Python didn't modify the argument `the_number`
{{< /highlight >}}

We see that `the_list` is modified but `the_number` doesn't change from `0`. I found this behaviour surprising the first time I encountered it in a language. I had guessed that both arguments would be modified or neither. As I learned more languages I saw this choice repeated in Ruby, Javascript, and (pre-dating those all) C.

## Did you say C?

Implementing something mostly equivalent C shows the exact same behaviour:

{{< highlight c >}}
#include <stdio.h>

int a_list[] = { 0 };
int a_number = 0;

void modify_arguments(int a_list[], int a_number) {
  a_list[0] = 42;
  a_number = a_number + 42;
}

int main(int argc, char *argv[]) {
   modify_arguments(a_list, a_number);
   printf("[%d], %d\n", a_list[0], a_number);
   return 0;
}

// This code outputs "[42], 0"
{{< /highlight >}}

This is because in C an array is passed around as a pointer to the first element of that array. Doing array assignment like `a_list[0] = 42` automatically dereference the pointer. In C you think about pointers a lot and this behaviour of arrays is explicitly covered so it doesn't seem incongruous. When `modify_arguments` is called the *pointer* to `a_list` is pushed to the stack and the *literal value* for `a_number` is pushed to the stack. When `modify_arguments` runs `a_list[0] = 42` automatically dereferences the pointer to `a_list` from the stack because it is an array assignment and `a_number = 42` updates *the copied value* of `the_number` which doesn't affect `the_number` outside `modify_arguments`.

The behaviour of copying a *pointer* to `the_list` to the stack makes sense, otherwise all the content in the array would need to be copied. Copying the literal value of `the_number` also makes sense given C is 45 years old and has a focus on speed.

In C we can explicitly pass pointers to write an alternative that does modify `the_number`:

{{< highlight c >}}
// Note: differences are pointed at with ^s

void modify_arguments(int a_list[], int *a_number) {
//--------------------------------------^
  a_list[0] = 42;
  *a_number = *a_number + 42;
//^
}

int main(int argc, char *argv[]) {
   modify_arguments(a_list, &a_number);
//--------------------------^
   printf("[%d], %d\n", a_list[0], a_number);
   return 0;
}

// This code outputs "[42], 42"
{{< /highlight >}}

## What about Python?

The stack copying business in C **is not** what is happening in Python. Python passes in `the_number` as a reference but doesn't allow basic types like `int` to be modified at all. The built-in `id` function returns a value that we can think of as a memory address. This makes it a bit more clear what is happening:

{{< highlight python3 >}}
def modify_arguments(a_list, a_number):
  print(id(a_list), id(a_number))
  a_list[0] += 42
  a_number += 42
  print(id(a_list), id(a_number))
{{< /highlight >}}

{{< highlight python3 >}}
>>> the_list = [0]
>>> the_number = 0
>>> print(id(the_list), id(the_number))
4420012616 4416354976
>>> modify_arguments(the_list, the_number)
4420012616 4416354976
4420012616 4416356320
>>> #------------^^^^ this id is different!
>>> print(id(the_list), id(the_number))
4420012616 4416354976
{{< /highlight >}}

What's happening is:

* `a_number += 42` creates a new `int` with an `id` of `4416356320`.
* `a_number` is set to refer to the newly created `int`.
* We leave `modify_arguments` and `a_number` is no longer used by anything and disappears. No modification of `the_number` happens at all.

### Passing references in Python

It is possible to pass a reference to `a_number` although it looks very different. We already know that objects like `list` can be modified inside a function. This means that wrapping an int in some mutable object allows it to be passed by reference. (Note: I don't recommend writing code like this. Almost certainly it would be better to return the newly calculated value.)

{{< highlight python3 >}}
class Ref:
  def __init__(self, object):
    self.object = object

def modify_arguments_with_ref(a_list, a_number_ref):
   a_list[0] += 42
   a_number_ref.object += 42

the_list = [0]
the_number_ref = Ref(0)

modify_arguments_with_ref(the_list, the_number_ref)

print(the_list, the_number_ref.object)

# This code outputs "[42] 42"
{{< /highlight >}}

Update: it is possible to edit the immutable value of an integer in Python. Doing so has [unintended consequences🔮]({{< ref "posts/2017-08-22-weird-python-integers.md" >}}).
