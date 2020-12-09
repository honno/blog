---
layout:     post
title:      Implementing a defaultlist
summary:    We all love Python's defaultdict, but how about a list variant?
---

Python's 
[`defaultdict`](https://docs.python.org/3/library/collections.html#collections.defaultdict)
is an incredibly useful tool
to specify default values for missing keys
(RealPython has a
[great primer](https://realpython.com/python-defaultdict/)
on them).

I found I was taking up so much memory to store the value 0 for an algorithm which
counted rather infrequent elements in a huge sequence split into many chunks.
Using a `defaultdict(int)` for each distinct element
to store which indices had a non-zero count of the respective element
greatly optimised it,
but I yearned for a `defaultlist` equivalent to standardise this approach and
generally make my code more expressive.

## Humble beginnings

There are some opinionated decisions of what constitutes a "defaultlist",
so whilst an 
[excellent implementation already exists by c0fec0de](https://github.com/c0fec0de/defaultlist),
I disagreed with some behaviours that led me to implement my own.
My idea of a `defaultlist` is to be a glorified wrapper of the `defaultdict`.
We can use the
[`MutableSequence`](https://docs.python.org/3/library/collections.abc.html#collections.abc.MutableSequence)
abstract base class to mirror the interface one would expect from a builtin `list`.

```python
class defaultlist(MutableSequence):
    def __getitem__(self, i):
        ...

    def __setitem__(self, i, value):
        ...

    def __delitem__(self, i):
        ...

    def __len__(self):
        ...
        
    def insert(self):
        ...
```

Firstly we can initialise an internal `defaultdict` in `self._ddict`,
taking a `default_facotry` argument.
Each key of `self._ddict` will represent an index of a list.

Note that `defaultdict` does not require a `default_factory`
and will raise a `KeyError` when you try to access a missing key in such scenarios.
For a list variant this behaviour would be nonsensical as
missing elements between existing ones really aren't erroneous,
so we can just make `default_factory` return `None` if not specified.

```python
    def __init__(self, default_factory=None):
        self._ddict = defaultdict(default_factory or defaultlist._none_factory)

    @staticmethod
    def _none_factory():
        return None
```

The get and set item methods,
which determine the behaviour of square brackets notation (e.g. `my_dlist[i]`),
can also directly interface with the internal `defaultdict` we initialised.
This means that all elements of our `defaultlist` will default
just like its dictionary counter-part.

```python
    def __getitem__(self, i):
        return self._ddict[i]
        
    def __setitem__(self, i, v):
        self._ddict[i] = v
```

A delete item method
(called like `del my_dlist[i]`)
requires a bit more work.
The `list` builtin will reindex elements above the deleted index,
and so we can emulate that behaviour
by decrementing the keys of our internal `defaultdict`.
Conversely an insert method can increment the keys of `self._ddict`
before safely assigning the value at `i`.

```python
    def __delitem__(self, i):
        # I decided to be agnostic if the element exists or not
        try:
            del self._ddict[i]
        except KeyError:
            pass

        larger_keys = [k for k in self._ddict.keys() if k > i]
        reindexed_subdict = {k - 1: self._ddict[k] for k in larger_keys}
        for k in larger_keys:
            del self._ddict[k]
        self._ddict.update(reindexed_subdict)
        
    def insert(self, i, value):
        larger_keys = [k for k in self._ddict.keys() if k >= i]
        reindexed_subdict = {k + 1: self._ddict[k] for k in larger_keys}
        for k in larger_keys:
            del self._ddict[k]
        self._ddict.update(reindexed_subdict)

        self._ddict[i] = value
```

As we're treating keys in the `self._ddict` as indexes,
we can treat the highest existing index as the deciding factor
for what constitutes the length
(e.g. result of `len(my_dlist)`)
of our `defaultlist`.

```python
    def __len__(self):
        try:
            return max(self._ddict.keys()) + 1
        except ValueError:
            return 0
```

We are currently assuming positive indices are being passed,
but `list` does take in negative indices
that translate to the length of the list `n`
minus (or rather plus) the negative index.
We can support this behaviour by inspecting and modifying the passed index
before we actually use it on our internal `defaultdict`.

```python
    def _actualise_index(self, i):
        if i >= 0:
            return i
        else:
            n = len(self)
            if i >= -n:
                return n + i
            else:
                raise IndexError("negative list index larger than list length, unresolvable")

    def __getitem__(self, i):
        i = self._actualise_index(i)
        return self._ddict[i]
        
    def __setitem__(self, i, v):
        i = self._actualise_index(i)
        self._ddict[i] = v
        
    def __delitem__(self, key: Union[int, slice]):
        i = self._actualise_index(key)
        try:
            del self._ddict[i]
        except KeyError:
            pass
        ...
```

As we've successfully overridden the `MutableSequence` interface,
it will conveniently deal with second-order features of a list
such as iteration (`for x in my_dlist`)
and appending (`my_dlist.append(x)`).

## Bounding infinity

Currently, iterating over a `defaultlist` will go on forever.
Iteration of an object is determined by it's `__iter__()` method,
and inheriting `MutableSequence` comes with such a method
which will `__getitem__(i)` from 0 until an `IndexError` is raised.
As `defaultlist` is an infinite field of elements,
no such error is ever raised.

I decided its iteration capabilities should be bounded
to the length property (defined earlier).

```python
    def __iter__(self):
        for k in range(len(self)):
            yield self._ddict[k]
```

This provides an easy way for user to specify iteration bounds
using slice notation.
Say you were implementing an algorithm that divvied-up a sequence into 100 chunks
and counted in each chunk using `counts = defaultlist(int)`
(i.e. default the count to 0),
then you can specify the first 100 elements of `counts`
when it's time for a for loop.

```python
for count in counts[:100]:
    ...
```

...that is, if slicing is supported.

We can actually detect the `stop:start:step` notation---which
is infact a builtin 
[`slice`](https://docs.python.org/3/library/functions.html#slice)
object---in
our `__getitem__()` method
by inspecting the passed `key` with an instance check (i.e. `isinstance()`).

A builtin `list` handles slices as a range bounded by the list's length---for
the most part,
check out my
[list reference implementation](https://github.com/Honno/py-slicing-reference)
for a complete specification of how `list` handles slicing.
We however don't need to worry about going out-of-bounds
as the `defaultlist` is essentially an infinite field of elements.
All we need to do then is handle `None` values
before making a 1-to-1 translation from `slice` to the `range` builtin.

```python
    def __getitem__(self, key):
        if isinstance(key, int):
            i = self._actualise_index(key)
            return self._ddict[i]

        elif isinstance(key, slice):
            start = key.start or 0
            stop = key.start or 0
            step = key.step or 1

            if key.start is None:
                start = 0 if step > 0 else len(self)
            else:
                start = self._actualise_index(key.start)

            if key.stop is None:
                stop = len(self) if step > 0 else 0
            else:
                stop = self._actualise_index(key.stop)

            srange = range(start, stop, step)

            dlist = defaultlist(self._ddict.default_factory)
            dlist.extend(self._ddict[i] for i in srange)

            return dlist
```

At this point, we've got ourselves a pretty functional `defaultlist`!

{% gist b8d9f6aa23e87f95effbf9611ad883d9 %}

## Filling in the gaps

### Complete slicing support

We can also support slices for setting and deleting elements.
Like with the `insert` method,
it's simply a matter of reindexing the internal `defaultdict`
to accommodate for the addition and/or removal of existing elements
... but as you can see,
it's a bit of a headache to keep track of things.

```python
    def _determine_srange(self, slice_):
        step = slice_.step or 1

        if slice_.start is None:
            start = 0 if step > 0 else len(self)
        else:
            start = self._actualise_index(slice_.start)

        if slice_.stop is None:
            stop = len(self) if step > 0 else 0
        else:
            stop = self._actualise_index(slice_.stop)

        return range(start, stop, step)

    def __getitem__(self, key):
        if isinstance(key, int):
            i = self._actualise_index(key)

            return self._ddict[i]

        elif isinstance(key, slice):
            srange = self._determine_srange(key)
            dlist = defaultlist(self._ddict.default_factory)
            dlist.extend(self._ddict[i] for i in srange)

            return dlist

    def __setitem__(self, key, value):
        if isinstance(key, int):
            i = self._actualise_index(key)
            self._ddict[i] = value

        elif isinstance(key, slice):
            values = list(value) if isinstance(value, Iterable) else [value]
            nvalues = len(values)

            srange = self._determine_srange(key)
            if srange:
                srange_min = min(srange[0], srange[-1])
                srange_max = max(srange[0], srange[-1])
            else:
                srange_min = srange_max = srange.start

            for i in srange:
                try:
                    del self._ddict[i]
                except KeyError:
                    pass

            diff = nvalues - len(srange)
            if diff:
                keys = list(self._ddict.keys())

                threshold = srange_min + nvalues
                reindexed_subdict = {}

                mid_keys = [k for k in keys if srange_min <= k < srange_max]
                for i, k in enumerate(sorted(mid_keys), start=threshold):
                    reindexed_subdict[i] = self._ddict[k]

                larger_keys = [k for k in self._ddict.keys() if k > srange_max]
                for k in larger_keys:
                    i = k + diff
                    reindexed_subdict[i] = self._ddict[k]

                self._ddict.update(reindexed_subdict)

            for i, v in enumerate(values, start=srange_min):
                self[i] = v

    def __delitem__(self, key):
        if isinstance(key, int):
            i = self._actualise_index(key)
            try:
                del self._ddict[i]
            except KeyError:
                pass

            larger_keys = [k for k in self._ddict.keys() if k > i]
            reindexed_subdict = {k - 1: self._ddict[k] for k in larger_keys}
            for k in larger_keys:
                del self._ddict[k]
            self._ddict.update(reindexed_subdict)

        elif isinstance(key, slice):
            srange = self._determine_srange(key)
            if srange:
                for i in srange:
                    try:
                        del self._ddict[i]
                    except KeyError:
                        pass

                srange_min = min(srange[0], srange[-1])
                srange_max = max(srange[0], srange[-1])

                reindexed_subdict = {}

                mid_keys = [k for k in self._ddict.keys() if srange_min <= k < srange_max]
                for i, k in enumerate(values, start=srange_min):
                    reindexed_subdict[i] = self._ddict[k]

                larger_keys = [k for k in self._ddict.keys() if k > srange_max]
                for k in larger_keys:
                    i = k - len(srange)
                    reindexed_subdict[i] = self._ddict[k]

                for k in chain(mid_keys, larger_keys):
                    del self._ddict[k]
                self._ddict.update(reindexed_subdict)
```

### Equality checks

We might also want to emulate how `list`
can be equality checked (i.e. `a == b` notation)
with other `list` objects.
Equality check behaviour is specified
in the `__eq__(self, other)` dunder method
of the first instance (i.e. `a` in `a == b`).
Rather than caring if a `defaultlist` is being checked with another `defaultlist` however,
I found broadly check against all
[`Sequence`](https://docs.python.org/3/library/collections.abc.html#collections.abc.Sequence)-like types
to make it much more versatile.

```python
    def __eq__(self, other):
        if isinstance(other, Sequence):
            if len(self) != len(other):
                return False
            else:
                return all(a == b for a, b in zip(self, other))
        else:
            return False
```

Note that `zip(self, other)` is using the `__iter__()` methods
of both the `defaultlist` instance and `other`.
This means you can check if an iteration of a `defaultlist`
would actually match up with your expectations
using the nice brackets notation that Python uses for its `list`.

```python
>>> my_dlist = defaultlist(int)
>>> my_dlist[2] = 2
>>> my_dlist == [0, 0, 2]
True
```

### Finding first instances

`MutableSequence` also adds the 
[`index()`](https://docs.python.org/3/library/array.html#array.array.index)
method to our `defaultlist()` automatically,
but like with `__iter__()` it expects an `IndexError` from our get method---as
this never comes to fruition,
`index()` can go on forever.

All this means is we need to override `index()`
to bound the iteration to it's pseudo-length.
Like with our `__eq__()` method above,
note that `enumerate(self)` is calling `__iter__()`.

```python
    def index(self, x):
        for i, v in enumerate(self):
            if v == x:
                return i
        else:
            raise ValueError(f"'{x}' is not in defaultlist")
```

This also fixes the automatic `remove()` method too,
as that relies upon `index()` to find the `i` to then delete.

## Fin

A hopefully production-ready `defaultlist`
is available in my [coinflip](https://github.com/Honno/coinflip/) library,
available in the [`coinflip.collections.defaultlist`](https://coinflip.readthedocs.io/en/latest/reference/collections.html#coinflip.collections.defaultlist) namespace.
Its source code is available on 
[GitHub](https://github.com/Honno/coinflip/blob/main/src/coinflip/_randtests/common/collections.py#L122)
(and don't forget [tests](https://github.com/Honno/coinflip/blob/main/tests/collections/test_defaultlist.py)).

Huge thanks to [c0fec0de](https://github.com/c0fec0de) for their `defaultlist` implementation,
which really helped me think deeply about what I wanted the API to be for my own take.
Also I suppose a shout out to Guido, who I *think* implemented the much beloved `defaultdict`. Maybe it was Raymond Hettinger...?
I couldn't work this one out---git blame and all---so if anyone knows this then please let me know!
