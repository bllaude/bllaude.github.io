---
layout: post
title:  My favorite programming pattern
date:   2022-01-18 01:08:00 +0800
categories: document
tag: programming
topping: false
---

* content
{:toc}

So at work we were talking about interviewer questions for fresh graduates, and one of them was the standard test to weed out those who *faked it 'til they made it*. It went something like this:

<pre>You are given a list of characters, return the list with commas seperating each element. Do not leave a trailing comma.
</pre>

There might’ve been some language constraints or something — honestly, I wasn’t paying that much attention — but it got me thinking about what my honest answer would be. Here’s what I came up with:

In Haskell,

```haskell
join' :: [Char] -> ([Char] -> [Char])
join' (x:xs) = (x:) . dropLast . (join' xs)
   where
      dropLast = if (xs /= []) then (',':) else (id)
join' [] = id

answer :: [Char] -> [Char]
answer x = join' x $ []
```

For those unfamiliar with Haskell, here's a quick run down of the important parts.

The function will look at the first character in the list, and partially apply it to the prepend function `:`

In Haskell, you can partially apply functions — meaning if a function takes two arguments, you can provide just one and get back a new function that takes the remaining argument.

In the case of prepend, `x:` is a function that will take another list, and return that list prepended with `x`, ie.

`'a' : ['b', 'c', 'd'] == "abcd" == ['a', 'b', 'c', 'd']`

We then compose that function with one of two functions returned by `dropLast`, dependending on whether `x` is the last character in the list.

Composing is done with `.`, and is a way to chain functions together. ie:

`(f . g)(x) == f(g(x))`

If the character we're examining isn't the last one, we prepend a comma. If *it is* the last, we use the `id` function instead, which simply returns its input unchanged.

After that, we recursively call the function with the tail of the list.

At the end of this, we don’t get a comma-separated list like you might expect — instead, we end up with a function. For the input `"abcd"`, the resulting function looks something like this:

`'a' : ',' : 'b' : ',' : 'c' : ',' : 'd' : id`

Which is a function that takes a list of characters, and returns a list of characters. To get our answer we need to give this function an argument, and the argument we want is an empty list.

This function works by building the list from the back, prepending one character at a time through each step, until we arrive at the final result:

`"a,b,c,d"`

Why I like this pattern                                                    {#Why I like this pattern}
====================================
**It's not slow**

This one is more applicable to Haskell, but the naive way to do this (outside of using a prebuilt function) would be to use `++` and just go through each character adding it and a comma to a final output string.

Strings in Haskell are lists, and lists are linked lists under the hood. So appending anything requires traversing the entire list, and doing so in the way described above ends up being `O(n^2)`.

The prepend function method, however, is much faster. Prepending takes constant time, so doing so as shown only ends up being `O(n)` which is far far better.

**It uses `id`**

This is one of the few times I get to honestly use the `id` function for something other than teaching how lambda calculus works, so it gets points for that.

**It's fun and different**

You can't discount this, programming is a hobby and therefore should be fun. Not much more to say about that.
