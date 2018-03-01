---
layout: post
title: "When is a bool not a bool?"
categories: [programming, python]
---
When is a `bool` not a `bool`? In Python, of course.

You see, once upon a time (specifically the late 80s and early 90s), some guy named [Guido](https://gvanrossum.github.io) started implementing a new programming language. One that would be pragmatic, uncluttered, and compact, yet highly opinionated. It's a neat language, but not without its drawbacks. 

Most of my day-to-day work is done in Python. Just yesterday I discovered a most interesting quirk of the language. Because Python is a [duck-typed](https://en.wikipedia.org/wiki/Duck_typing) language, values of one type can be frequently interpreted as values of another type. In fact, some of this is by design. In the Python "days of old", there wasn't even a `bool` type -- just `int` values of `0` or `1`.

What I was working with in my codebase was a bit of logic to read an optional config value, check if it was an `int` (intending for `None` or `False` as a value to mean that the config flag was disabled) and if so, act accordingly with that `int`[^existing-bug].

Anyway, here's an approximation of the code used:

```python
def scrambled_subset(query, config):
    if isinstance(config.QUERY_LIMIT, int):
        query = query.order_by(fn.RANDOM).limit(config.QUERY_LIMIT)

    # ...
    # More implementation
    # ...
    
    return query
```

There was more going on overall here, but that's the gist of it. The intended effect? If there was a limit configured, limit the query to that many rows. The limit was in play for some time, but then the need came to remove the limit -- so I set `config.QUERY_LIMIT` to `False`, thinking "welp, `False` definitely is not an `int`"[^hindsight].

So, there I was watching the consumer of that (so I thought) un-limited query have no rows to act on. What the what? So I did some poking around the code.

```python
>>> type(False)
<type 'bool'>
>>> isinstance(False, bool)
True
>>> isinstance(False, int))
True 
```

Um.

So after a little bit more research, I found that Python's `bool`s are totally acceptable `int`s. They even have prescribed values:

```python
>>> False + 1
1
>>> True + 1
1
>>> 2 ** (True + 1)
4
>>> 2 ** (- True)
0.5
>>> 2 ** False
1
>>> False ** 2
0
```

So there you have it, folks. In Python, a `bool` is really just a fancy fucking `int`.

[^existing-bug]: Interestingly, this bug should have shown itself from day one, but because the code inside the conditional block had its own issue (in particular, an ORM whose `LIMIT` clause for queries was broken when the query was "sorted" by `RANDOM`...) it remained hidden until the dependency's bug was actually fixed.

[^hindsight]: In hindsight, I probably should have set to `None` -- I think an initial version of this expected the config value to be a flag and the limit quantity was hardcoded to some value. The config just enabling or disabling the hardcoded limit.