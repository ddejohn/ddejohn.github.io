---
layout: post
title: "./overlapping_strings"
date: 2021-09-26 18:00:00 -0600
---

An extension of [one of my recent Stack Overflow answers](https://stackoverflow.com/questions/69337251/concatenate-strings-with-a-common-substring-in-python/69337268#69337268).

I'll show two pure Python solutions for concatenating an arbitrary list of strings based on suffix-prefix pairs. This means that EVERY possible pair of words must be checked to see if their suffix and prefixes match e.g.: `"hello world"` and `"world foo"`.

Here's an example of a (short) list of strings for which we can start working on a solution:

```python
strings = ["lazydog",
           "Thequick",
           "thelazy",
           "overthe",
           "foxjumps",
           "jumpsover",
           "brownfox",
           "quickbrown"]
 ```

 Notice that if the strings were in order, e.g., `"Thequick", "quickbrown", "brownfox", ...`, then it'd be pretty straight forward to join the strings and remove the overlaps to get the desired result `"Thequickbrownfox..."`. Of course, we can't make that assumption about an ordered list, so we'll have to figure out the ordering manually.

Since we need to check every pair of words, and since we can't assume *any* pair is ordered, we also have to take the reverse of every pair:

```python
pairs = set()
for a in strings:
    for b in strings:
        if a != b:
            pairs.add((a, b))
            pairs.add((b, a))
```

Output:

```python
{('Thequick', 'brownfox'),
 ('Thequick', 'foxjumps'),
 ('Thequick', 'jumpsover'),
 ('Thequick', 'lazydog'),
 ('Thequick', 'overthe'),
 ('Thequick', 'quickbrown'),
 ('Thequick', 'thelazy'),
 ('brownfox', 'Thequick'),
 ('brownfox', 'foxjumps'),
 ('brownfox', 'jumpsover'),
 ('brownfox', 'lazydog'),
 ('brownfox', 'overthe'),
 .
 .
 .
```

For this task, we can use permutations instead, which tends to be a bit quicker than the cartesian product:

```python
import itertools

set(itertools.permutations(strings, r=2))
```

Output:

```python
{('Thequick', 'brownfox'),
 ('Thequick', 'foxjumps'),
 ('Thequick', 'jumpsover'),
 ('Thequick', 'lazydog'),
 ('Thequick', 'overthe'),
 ('Thequick', 'quickbrown'),
 ('Thequick', 'thelazy'),
 ('brownfox', 'Thequick'),
 ('brownfox', 'foxjumps'),
 ('brownfox', 'jumpsover'),
 ('brownfox', 'lazydog'),
 ('brownfox', 'overthe'),
 .
 .
 .
```

Now for every pair, we need to find whether or not they overlap. Take, for example, the pair `('brownfox', 'foxjumps')`:

```python
brownfox
       foxjumps

brownfox
      foxjumps

brownfox
     foxjumps  # The end of 'brownfox' overlaps with the start of 'foxjumps'

brownfox
    foxjumps

brownfox
   foxjumps

brownfox
  foxjumps

brownfox
 foxjumps
```

Now take `('brownfox', 'jumpsover')`:

```python
brownfox
       jumpsover

brownfox
      jumpsover

brownfox
     jumpsover

brownfox
    jumpsover

brownfox
   jumpsover

brownfox
  jumpsover

brownfox
 jumpsover

 # No point in checking any further than this
```

We'll use this function to find a suffix-prefix match:

```python
def get_match(pair: Tuple[str, str], min_overlap: int = 3) -> str:
    a, b = pair
    for i in range(min_overlap, min(map(len, pair)) + 1):
        if a[-i:] == b[:i]:
            return b[:i]
    return ""
```

What this function does is essentially what is shown in the diagrams above: the second string `b` from `pair` is "slid" over the "top" of the first string `a` from `pair` until a match is found at which point the function returns immediately. The `min_overlap` parameter is optional and allows us to enforce that a match must be at least `min_overlap` number of characters, e.g., if `min_overlap == 2`, then `"hello", "one"` shouldn't match since the only suffix-prefix match is `"o"`.

Now, we need to iterate over every possible pairing, and record all pairs which match:

```python
for pair in itertools.permutations(strings, r=2):
    if (match := get_match(pair)):
        print(pair)
```

Output:

```python
('Thequick', 'quickbrown')
('thelazy', 'lazydog')
('overthe', 'thelazy')
('foxjumps', 'jumpsover')
('jumpsover', 'overthe')
('brownfox', 'foxjumps')
('quickbrown', 'brownfox')
```

Now here's where the two solutions diverge.
  
## First solution: constructing a DAG

So we know that the solution for our particular example should be `Thequickbrownfoxjumpsoverthelazydog`. This can be represented by a (boring) [Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph):

```python
Thequick -> quickbrown -> brownfox -> foxjumps -> jumpsover -> overthe -> thelazy -> lazydog
```

You may also recognize that as a singly-linked list, which is what it is in essence, save for the fact that we only have the nodes, and the rules for what can be connected. We still have to properly construct that singly-linked list.

This first solution uses a dictionary to track the adjacencies:

```python
def links_joiners(strings: List[str]) -> Tuple[Dict[str, str], Set[str]]:
    links = dict()
    joiners = set()
    for pair in itertools.permutations(strings, r=2):
        if (match := get_match(pair)):
            joiners.add(match)
            links.update((pair,))
    return links, joiners
```

Output:

```python
>>> links, joiners = links_joiners(strings)
>>> links
{'Thequick': 'quickbrown',
 'thelazy': 'lazydog',
 'overthe': 'thelazy',
 'foxjumps': 'jumpsover',
 'jumpsover': 'overthe',
 'brownfox': 'foxjumps',
 'quickbrown': 'brownfox'}

>>> joiners
{'brown', 'fox', 'jumps', 'lazy', 'over', 'quick', 'the'}
```

For now, we'll focus on the `links` dictionary, which shows us our adjacencies. Recall our target:

```python
Thequick -> quickbrown -> brownfox -> foxjumps -> jumpsover -> overthe -> thelazy -> lazydog
```

Now pick a random key in `links` and take its associated value. If that value is *also* a key, repeat that last step with the new key. If you get a value that *isn't* also a key in `links`, you've reached the end of the DAG. For example, consider an intial starting key, `"jumpsover"`.

* `"jumpsover"` points to `"overthe"`
* `"overthe"` points to `"thelazy"`
* `"thelazy"` points to `"lazydog"`
* `"lazydog"` is not a key in `links`, therefore we've reached the end of the DAG

This algorithm gives us the number of steps from any given key in the DAG (or "node") to the end. Our goal is to connect the nodes in the correct order without having to rerun another `get_match()` function on all the adjacent pairs of pairs. With that in mind, we can recursively traverse over `links`, finding the number of steps to the end of the DAG from every node. Since there's only one sequence which results in the longest list, and that will be achieved by ordering the nodes by their number of steps to the end of the DAG, in descending order:

```python
 nodes:   Thequick -> quickbrown -> brownfox -> foxjumps -> jumpsover -> overthe -> thelazy -> lazydog
 steps:       7           6            5           4            3           2          1          0
```

Here's the code:

```python
def sort_nodes(nodes: List[str], links: Dict[str, str]) -> List[str]:
    def dist(node: str) -> int:
        if node not in links:
            return 0
        return 1 + dist(links[node])
    return sorted(nodes, key=dist, reverse=True)
```

The inner function `dist()` is what's known as a [closure](https://en.wikipedia.org/wiki/Closure_(computer_programming)), which means that `dist()` encloses or "captures" any variables which are local to the scope of the closure's parent and which are used inside the closure. Here, the two variables in the parent scope are `nodes`, and `links`, the two parameters of the `sort_nodes()` function.

Notice that inside the closure `dist()`, the variable `links` is accessed even though `links` isn't within the scope of `dist()`. This is taking advantage of the fact that a function in Python will jump up the call stack to its parent's scope for a variable reference if it can't find it locally first.

The reason for using a closure here is so `dist` only needs to take one parameter. This is so we can use it as the sorting method for the built-in `sorted()` function. Under the hood, Python will take each `node` in the list of `nodes`, and use `dist` to determine its ordinality.

Let's look at the closure more closely:

```python
def dist(node: str) -> int:
    if node not in links:
        return 0
    return 1 + dist(links[node])
```

Take a minute to convince yourself that this function implements the algorithm outlined above. Let's do an example. Recall `links`:

```python
{'Thequick': 'quickbrown',
 'thelazy': 'lazydog',
 'overthe': 'thelazy',
 'foxjumps': 'jumpsover',
 'jumpsover': 'overthe',
 'brownfox': 'foxjumps',
 'quickbrown': 'brownfox'}
```

Now trace an example call, `dist("jumpsover")`:

```python
dist("jumpsover") = 1 + dist("overthe")
                  = 1 + (1 + dist("thelazy"))
                  = 1 + (1 + (1 + dist("lazydog")))  # base case; "lazydog" is not a key in `links`
                  = 1 + (1 + (1 + 0))
                  = 3
```

Now, by passing all of our `strings`, or "nodes", to `sorted()`, we will end up with a properly sorted list:

```python
strings = ["lazydog",
           "Thequick",
           "thelazy",
           "overthe",
           "foxjumps",
           "jumpsover",
           "brownfox",
           "quickbrown"]

nodes_in_order = sort_nodes(strings, links)
```

Output:

```python
>>> nodes_in_order
['Thequick',
 'quickbrown',
 'brownfox',
 'foxjumps',
 'jumpsover',
 'overthe',
 'thelazy',
 'lazydog']
```

Now, recall `joiners` from earlier:

```python
{'brown', 'fox', 'jumps', 'lazy', 'over', 'quick', 'the'}
```

Our sorted list, together with `joiners` are all we need in order to finally put the pieces together. However, there's one last hurdle, when we join `nodes_in_order`, all of the overlapping strings are present:

```python
>>> "".join(nodes_in_order)
'Thequickquickbrownbrownfoxfoxjumpsjumpsoveroverthethelazylazydog'
```

So instead of leaving it at that, we iterate over `joiners` and for each joiner `j`, we replace one copy of `j` with the empty string so as to get rid of the duplicates formed by joining:

```python
def join_strings(strings: List[str], joiners: Set[str]) -> str:
    result = "".join(strings)
    for j in joiners:
        result = result.replace(j, "", 1)
    return result
```

Output:

```python
>>> join_strings(nodes_in_order, joiners)
'Thequickbrownfoxjumpsoverthelazydog'
```

Done!

## Second solution: pure, unfettered recursion

This solution reuses `join_strings()` and `get_match()` from above. This solution foregoes collecting pairs, creating a dictionary of adjacency "links", finding the correct ordering -- and skips straight to joining pairs of strings:

```python
def join_overlapping_pairs(strings: Set[str]) -> str:
    if len(strings) == 1:
        return strings.pop()
    matches = set()
    for pair in itertools.permutations(strings, 2):
        if (match := get_match(pair)):
            matches.add(join_strings(pair, (match,)))
    return join_overlapping_pairs(filter_substrings(matches))


def filter_substrings(strings: Set[str]) -> Set[str]:
    if len(strings) == 1:
        return strings
    largest = set()
    for s1 in strings:
        for s2 in strings - {s1,}:
            if (s1 in s2):
                break
        else:
            largest.add(s1)
    if largest == strings:
        return strings
    return largest | filter_substrings(largest)
```

By adding `print(matches)` right before the recursive call of `join_overlapping_pairs()`, we can trace the behavior of the function:

```python
>>> join_overlapping_pairs(strings)
{'brownfoxjumps', 'foxjumpsover', 'jumpsoverthe', 'quickbrownfox', 'overthelazy', 'thelazydog', 'Thequickbrown'}
{'Thequickbrownfoxjumps', 'quickbrownfoxjumpsover', 'jumpsoverthelazydog', 'foxjumpsoverthelazy', 'brownfoxjumpsoverthe'}
{'Thequickbrownfoxjumpsoverthelazydog'}
```

The `filter_substrings()` function isn't wholly necessary, but it does reduce the size of the search space `join_overlapping_pairs()` has to work with. Without it, here's what the `matches` set looks like after the second iteration of `join_overlapping_pairs()`:

```python
{'overthelazydog', 'brownfoxjumpsoverthe', 'quickbrownfoxjumps', 'jumpsoverthelazydog', 'jumpsoverthelazy', 'Thequickbrownfox', 'foxjumpsoverthelazy', 'Thequickbrownfoxjumps', 'quickbrownfoxjumpsover', 'foxjumpsoverthe', 'brownfoxjumpsover'}
```

But if you look closely, many of these strings are actually substrings of others (e.g., `"jumpsoverthelazy"` and `"foxjumpsoverthelazy"`), which means we can get rid of a lot of redundancy. If we print `len(matches)` instead:

```python
7 42
11 110
15 210
10 90
6 30
3 6
1 0
```

The numbers on the left are how many strings were passed to `join_overlapping_pairs()`. The numbers on the right are the size of the search space. Notice that the number of strings and the size of the search space (the paired permutations of every string, which is `n * (n - 1)`) actually *increases* before it decreases. This isn't a good sign if we want to use this for larger sets of strings...

By first filtering out the substrings before recursing, we can not only reduce the search space for successive iterations, but even reduce the number of steps it takes to find a solution:

```python
7 42
5 20
1 0
```

The `filter_substrings()` function is pretty straightforward. It recursively iterates over the set, checking each string in the set to determine if it's a substring of any other string in the set.

## First solution: source code

Here's the source code for the first solution:

```python
import itertools
from typing import Set, Tuple, Dict, List


def get_match(pair: Tuple[str, str], min_overlap: int = 3) -> str:
    """
    For `a`, `b` in `pair`, this 'slides' `b` over the end of
    `a` from the left, starting at `min_overlap`. Returns as
    soon as a match is found, or the empty string otherwise.
    """
    a, b = pair
    for i in range(min_overlap, min(map(len, pair)) + 1):
        if a[-i:] == b[:i]:
            return b[:i]
    return ""


def join_strings(strings: List[str], joiners: Set[str]) -> str:
    """
    Joins an ordered sequence of `strings`, removing the extraneous `joiner`
    between each joined pair.
    """
    result = "".join(strings)
    for j in joiners:
        result = result.replace(j, "", 1)
    return result


def links_joiners(strings: List[str]) -> Tuple[Dict[str, str], Set[str]]:
    """
    Links will be key:value pairs of words that have a common
    substring where the end of `s1` overlaps with the start of `s2`.
    """
    links = dict()
    joiners = set()
    for pair in itertools.permutations(strings, r=2):
        if (match := get_match(pair)):
            joiners.add(match)
            links.update((pair,))
    return links, joiners


def sort_nodes(nodes: List[str], links: Dict[str, str]) -> List[str]:
    """
    A recursive closure which determines the length of each sequence
    starting at `node` and traversing the `links` dictionary.
    """
    def dist(node: str) -> int:
        if node not in links:
            return 0
        return 1 + dist(links[node])
    return sorted(nodes, key=dist, reverse=True)
```

## Second solution: source code:

```python
import itertools
from typing import Set, Tuple, List, Union


def get_match(pair: Tuple[str, str], min_overlap: int = 3) -> str:
    """
    For `a`, `b` in `pair`, this 'slides' `b` over the end of
    `a` from the left, starting at `min_overlap`. Returns as
    soon as a match is found, or the empty string otherwise.
    """
    a, b = pair
    for i in range(min_overlap, min(map(len, pair)) + 1):
        if a[-i:] == b[:i]:
            return b[:i]
    return ""


def join_strings(strings: List[str], joiners: Set[str]) -> str:
    """
    Joins an ordered sequence of `strings`, removing the extraneous `joiner`
    between each joined pair.
    """
    result = "".join(strings)
    for j in joiners:
        result = result.replace(j, "", 1)
    return result


def join_overlapping_pairs(strings: Set[str]) -> str:
    if len(strings) == 1:
        return strings.pop()
    matches = set()
    for pair in itertools.permutations(strings, 2):
        if (match := get_match(pair)):
            matches.add(join_strings(pair, (match,)))
    return join_overlapping_pairs(filter_substrings(matches))


def filter_substrings(strings: Set[str]) -> Set[str]:
    if len(strings) == 1:
        return strings
    largest = set()
    for s1 in strings:
        for s2 in strings - {s1,}:
            if s1 in s2:
                break
        else:
            largest.add(s1)
    if largest == strings:
        return strings
    return largest | filter_substrings(largest)
```
