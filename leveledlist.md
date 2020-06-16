# Fallout 76 Leveled List drop chances

Table of contents

- [Terminology](#terminology)
  - [`chanceNone`](#chancenone)
  - [curve table lookup](#curve-table-lookup)
  - [condition evaluation](condition-evaluation)
- [How the game evaluates a leveled list](#how-the-game-evaluates-a-leveled-list)
  - [1. evaluate list conditions](#1-evaluate-list-conditions)
  - [2. get list `chanceNone`](#2-get-list-chanceNone)
  - [3. prepare the list entries](#3-prepare-the-list-entries)
    - [entry minimum levels](#entry-minimum-levels)
    - [entry conditions](#entry-conditions)
  - [4. pick an entry or entries](#4-pick-an-entry-or-entries)
    - [**bit 2** is set](#bit-2-is-set)
    - [**bit 1** is set](#bit-1-is-set)
    - [**bit 1** is clear](#bit-1-is-clear)
    - [**bit 6** is set](#bit-6-is-set)
- [Calculating the exact drop chances](#calculating-the-exact-drop-chances)
  - [Dealing with conditions](#dealing-with-conditions)
  - [Dealing with sublists](#dealing-with-sublists)
  - [Calculation for mode All](#calculation-for-mode-all)
    - [maximum = 0](#maximum-is-0)
    - [maximum = 1](#maximum-is-1)
    - [maximum more than 1](#maximum-is-more-than-1)
  - [Calculation for mode non-All](#calculation-for-mode-non-all)
    - [Uniform pick](#uniform-pick)
    - [Combinatorial pick](#combinatorial-pick)
    - [for-each repeated rolls](#for-each-repeated-rolls)
  - [Calculation for mode First](#calculation-for-mode-for-first)
    - [Pick first always](#pick-first-always)
    - [Pick one](#pick-one)
  - [Calculating the cascading chances](#calculating-the-cascading-chances)

## Terminology

### `chanceNone`

The game has properties specifying a probability (rolled against with a PRNG) that an entry or list **will not be used**.

For example, if a list has a `chanceNone` of 80, 80% of the time when evaluating the list, there won't be any results.

`chanceNone` defines the absense probability percentage (i.e, when calculating the `chanceNone` has to be divided by 100).

[Example list: LVLI:00064B92 &lt;LPI_Clothes_LocTheme&gt;](https://nukacrypt.com/database/json/00064B92)

### curve table lookup

Sometimes actual property values are looked up in a so-called curve table. A curve table is a list of `(x, y)` pairs where
an indexer value of `x` is used to get a `y` value.

For example, given a table of `(1, 0), (2, 10), (3, 20)` and an indexer `x = 2`, the lookup will return `20`.

#### lookup with inexact indexer

*This part is only assumed*

- If the indexer doesn't point to an exact match in the curve table, the lookup will do linear interpolation between the two nearby entries.

    For example, with a table of `(1, 0), (2, 10), (3, 20)` and an indexer `x = 2.5`, the lookup will return `15`.

- If the indexer is smaller than the smallest `x` value in the table, then the first entry's `y` is returned.

    For example, with a table of `(1, 0), (2, 10), (3, 20)` and an indexer `x = 0.5`, the lookup will return `0`.

- If the indexer is greater than the largest  `x` value in the table, then the last entry's `y` is returned.

   For example, with a table of `(1, 0), (2, 10), (3, 20)` and an indexer `x = 3.5`, the lookup will return `20`.

### condition evaluation

A list or an entry may have an associated condition list. A condition list is a series of function calls joined by either a *logical-and*
or a *logical-or* operation. These joins are defined on each condition element and affects what comes after.

Examples: `[HasKeyword(K1) == 1 OR HasKeyword(K2) == 1]`, `[GetRandomPercent >= 50 AND GetIsInRegion(R) == 1]`.

The last element's join operation is ignored.

In the condition list, the *logical-or* operations have higher precedence than the *logical-and* operations.

For example, given the following condition list `[C1 OR C2 AND C3 OR C4]`, 
adding parenthesis shows the evaluation order: `(C1 OR C2) AND (C3 OR C4)`.


## How the game evaluates a leveled list

### 1. evaluate list conditions

If the list has conditions, those conditions are evaluated. If they pass, the list is evaluated further.

If they don't pass, the list is considered **empty**.

### 2. get list `chanceNone`

First step is to get the list's `chanceNone` value extracted from the

- `LVCV` attribute,
- `LVLG` global value or
- `LVCT` curve table for which the indexer will be the global value of `LVLG`

If `LVLG` or `LVCT` are present, `LVCV` is ignored.

Next, a PRNG is rolled and if the rolled value is less than the `chanceNone` percentage, the list is considered **empty**.

### 3. prepare the list entries

#### entry minimum levels

The list entries can have a level requirement and if those are not met, the entry will be ignored in the subsequent steps.

Each entry can have a minimum level requirement, specified by a value:

- `LVLV` attribute
- `LVOG` gets a global value

If `LVOG` is present, `LVLV` is ignored.

However, the list's flags determines how to evaluate this minimum level.

Given the flag attribute `LVLF`:

- if **bit 0** is set (called *for each level &lt;= player level* in tools), all entries are kept which have minimum level less than or equal to the *subject/target level*

    Example: given a list of with flags = 1 or 

        [
            Entry1(minLevel = 1), 
            Entry2(minLevel = 10)
        ]

    - if *subject/target level* is 5, only `Entry1` is kept. 
    - If *subject/target level* is 10, both `Entry1` and `Entry2` are kept.

- if **bit 0** is clear, entries are kept which have a minimum level closest to the *subject/target level* and not exceeding it

    Example: given a list of 

        [
            Entry1(minLevel = 1), 
            Entry2(minLevel = 10), 
            Entry3(minLevel = 10), 
            Entry4(minLevel = 20)
        ]

    - if *subject/target level* is 5, only `Entry1` is kept. 
    - If *subject/target level* is 10, only `Entry2` and `Entry3` are kept.
    - If *subject/target level* is 19, only `Entry2` and `Entry3` are kept.
    - If *subject/target level* is 20, only `Entry4` is kept.

One complication is that if **bit 2** of the flag is set (called *use all* in tools), every entry is kept regardless of its minimum level.

#### entry conditions

The list entries can have their own conditions that determine if those entries should be considered at all. If an entry's
condition doesn't pass, it will be ignored in the subsequent step.

Thus, the game builds a pruned list by removing any entries whose conditions don't pass.

For example, given a list of 

    [
       Entry1 (HasKeyword(K1) == 1),
       Entry2 ()
    ]

- If keyword `K1` is not present on the subject/target, the pruned list will only hold `Entry2`. 
- If `K1` is present, the pruned list will hold both `Entry1` and `Entry2`.

### 4. pick an entry or entries

The evaluation of the pruned entry list can happen in multiple ways, depending on the flags associated with the leveled list.

The flag attribute `LVLF` is a bitfield indicating how to proceed.

Currently, 4 evaluation modes have been explored:

#### **bit 2** is set

If **bit 2** is set (named *all* in tools), every entry in the pruned list is consulted and collected up to be the result of this list.

For this mode, the *maximum* attribute can limit the total number of the entries in the results. The *maximum* number is extracted from the

- `LVMV` attribute,
- `LVMG` global value or
- `LVMT` curve table where the indexer is the `LVMG` global value.

When *maximum* is `0`, all items can be returned. If *maximum* is 1, only up to 1 entry is returned. If *maximum* is 2, up to 2 entries can be returned.

However, since each entry can have its `chanceNone` defined and non-zero, the output may not contain that many entries after all. An entry's `chanceNone` is a percentage extracted from the

- `LVOV` attribute,
- `LVOC` global value or
- `LVOT` curve table where the indexer is the `LVOC` global value.

Thus, for each entry in the pruned list, if a PRNG is rolled less than the entry's `chanceNone` percentage, the entry is ignored.

For example, given a list of

    [
        Entry1(chanceNone = 20),
        Entry2(chanceNone = 0),
        Entry3(chanceNone = 100)
    ]

- If PRNG rolls 50, `Entry1` is kept; `Entry2` is always present and `Entry3` is never present.
- If PRNG rolls 10, `Entry1` is ignored; `Entry2` is always present and `Entry3` is never present.

The entries are evaluated in order and the processing stops if the number of entries that passed is equal to the *maximum* specified.
(Note, since the stop condition is evaluated directly after an entry has determined to be passing, having *maximum* of `0` will never be true thus
every entry will be visited. Computationally, this is equivalent to having no limit on the size of the output.)

Having a non-zero *maximum* can result in non-intuitive outcomes, therefore, here are some examples with *maximum* and entries with `chanceNone`:

1. Example: *maximum* is `1` with guaranteed entry first entry:

        [
            Entry1,
            Entry2(chanceNone = 20)
        ]

    This list will always return `Entry1`.

2. Example: *maximum* is `1` with mixed entries:

        [
            Entry1(chanceNone = 20),
            Entry2
        ]

    - PRNG rolls X
    - If X >= 20, the output will be only `Entry1`.
    - If X < 20, `Entry1` is ignored and the output will be only `Entry2`.

3. Exmaple: *maximum* is `1` with chanced entries:

       [
           Entry1(chanceNone = 20),
           Entry2(chanceNone = 50)
       ]
    
    - PRNG rolls X
    - If X >= 20, the output will be only `Entry1`
    - If X < 10, `Entry1` is ignored and the next entry is evaluated
      - PRNG rolls Y
      - If Y >= 50, `Entry2` is ignored and the output will be **empty**
      - If Y < 50, the output will be only `Entry2`.

4. Example: *maximum* is `2` with chanced entries:

       [
           Entry1(chanceNone = 20),
           Entry2,
           Entry3
       ]

    - PRNG rolls X
    - If X >= 20, the output will be `Entry1` and `Entry2`
    - If X < 20, `Entry1` is ignored and the output will be `Entry2` and `Entry3`.

5. Example: *maximum* is `2` with chanced entries:

       [
           Entry1(chanceNone = 20),
           Entry2(chanceNone = 50),
           Entry3
       ]

    - PRNG rolls X
    - If X >= 20, then
      - PRNG rolls Y
      - If Y >= 50, the output will be `Entry1` and `Entry2`.
      - If < 50, the output will be `Entry1` and `Entry3`.
    - If X < 20
      - PRNG rolls Z
      - If Z >= 50, the output will be `Entry2` and `Entry3`.
      - If Z < 50, the output will be only `Entry3`.


#### **bit 1** is set

If **bit 1** is set (named *for each entry* in  tools), a single entry from the pruned list is picked at uniform random every time.

More specifically, if this list is referenced by a parent leveled list, the current list is evaluated over and over, specified by the
referencing entry's quantity amount.

For example, given the following nesting:

    [
        Entry1(Quantity = 2):
        [ 
            // list flags: 2
            Entry2,
            Entry3
        ]
    ]

The inner list will be evaluated twice, each time picking `Entry2` or `Entry3` with 50% chance. Therefore, the output of the main list can be:

- 25% of the time, `Entry2` appears twice
- 25% of the time, `Entry3` appears twice
- 50% of the time, both `Entry2` and `Entry3` appear.

The probability of `Entry2` appearing at least once can be calculated by calculating it not appearing at all for all the rolls, then subtracting that
value from 1. With the example, the chance `Entry2` is not rolled twice is `(1 - 0.5) * (1 - 0.5) = 0.25`, **25%**. Consequently, the chance
`Entry2` is rolled at least once is `1 - 0.25 = 0.75`, **75%**!

Regardless, a single evaluation of the list is done by picking one entry with uniform randomness.

For example, given a list:

    [
        Entry1,
        Entry2,
        Entry3
    ]

- 1/3 of the time, `Entry1` is chosen,
- 1/3 of the time, `Entry2` is chosen,
- 1/3 of the time, `Entry3` is chosen.

However, since each entry can have its `chanceNone` defined and non-zero, the output may be **empty** after all. An entry's `chanceNone` is a percentage extracted from the

- `LVOV` attribute,
- `LVOC` global value or
- `LVOT` curve table where the indexer is the `LVOC` global value.

Thus, if an entry was picked uniform random before, a PRNG is rolled and if less than the `chanceNone` percentage, the output will be **empty**.

1. Example: given a list of

        [
            Entry1(chanceNone = 20)
            Entry2
        ]

    - PRNG rolls X
    - If X < 50, then
      - PRNG rolls Y
      - If Y < 20, the output is **empty**
      - If Y >= 20, the output is `Entry1` 
    - If X >= 50, the output is `Entry2`.

2. Example: given a list of

        [
            Entry1(chanceNone = 20)
            Entry2
            Entry2(chanceNone = 50)
        ]

    - PRNG rolls X
    - If X < 33.33, then
      - PRNG rolls Y
      - If Y < 20, the output is **empty**
      - If Y >= 20, the output is `Entry1` 
    - If 33.33 <= X < 66.66, the output is `Entry2`.
    - If X >= 66.66
      - PRNG rolls Z
      - If Z < 50, the output is **empty**
      - If Z >= 50, the output is `Entry3` 



#### **bit 1** is clear

If **bit 1** is clear and if this list is embedded in another leveled list, the list itself is evaluated once, similar to the [**bit 1** set](#bit-1-is-set) case above,
and the item quantities are multiplied by the parent leveled list referencing entry's quantity.

Example:

    [
        Entry1(Quantity = 2):
        [ 
            // list flags: 0
            Entry2(Quantity = 3),
            Entry3(Quantity = 4)
        ]
    ]

In this setup, the inner list picks `Entry2` or `Entry3` with 50% probability, then multiplies their quantity, thus the main list will output
`Entry2(Quantity = 6)` 50% of the time and `Entry3(Quantity = 8)` 50% of the time.

From the drop chances perspective of the entries themselves, having **bit 1** set or not is an *equivalent setup*. 

The reason for this is that when we consider simulating the drops by running the main list N times, 
the inner list would be run 2 * N times if **bit 1** was set and N times if **bit 1** was clear, essentially running the simulation twice as long.


#### **bit 6** is set

If **bit 6** is set (named *first entry where conditions match* in  tools), only the first entry will be consulted.

This first entry can have its `chanceNone` defined and non-zero, the output may not contain that many entries after all. An entry's `chanceNone` is a percentage extracted from the

- `LVOV` attribute,
- `LVOC` global value or
- `LVOT` curve table where the indexer is the `LVOC` global value.

Thus, if a PRNG is rolled less than the entry's `chanceNone` percentage, the output will be **empty**. Otherwise, the output will be this first entry.

## Calculating the exact drop chances

Given the evaluation methods in the previous section, luckily, the drop chances for any entry can be calculated via exact formulas.

### Dealing with conditions

For the most part, conditions can be considered as pre-calculated. I.e., given a specific game state, the conditions and minimum level requirements
on the leveled lists and entries can be pre-processed, thus a tree of leveled list can be pruned upfront so that the chance calculations
only have to deal with present entries.

Therefore, an entry's chance information has only to consider its own `chanceNone` and, if it references another list, the chance that sublist evaluates to be empty itself, recursively applied if necessary.

However, one complication comes from conditions, such as `GetRandomPercent`, where the condition has to be converted into a probability on the list or on the entry it uses. Effectively, such conditions can act as another `chanceNone` value, to be combined with the list's/entry's own `chanceNone` value.

Consequently, the probability a list itself is considered in the first place is:

```javascript
var listSelfChance = (1 - list.chanceNone) * list.conditionChance;
```

where 

 - `list.chanceNone` is a value between 0 and 1 (i.e.`LVCV / 100.0`)
 - `list.conditionChance` is a value between 0 and 1 and is extracted from the condition
   -  `GetRandomPercent >= X` or `GetRandomPercent > X` as `conditionChance = 1 - X / 100.0`
   -  `GetRandomPercent < X` or `GetRandomPercent <= X` as `conditionChance = X / 100.0`

Similarly, an entry can have a `chanceNone` on its own, a `GetRandomPercent` and a possible sublist with an `emptyChance` evaluated.

```javascript
var entrySelfChance = (1 - entry.chanceNone) * entry.conditionChance;

// or

var entrySelfChance = (1 - entry.chanceNone) * entry.conditionChance * (1 - entry.sublist.emptyChance);
```

### Dealing with sublists

Since leveled lists can reference other leveled lists, those sublists can turn out to be empty with a certain probability, which has to be taken
into account with the various calculation methods.

For example, if the parent list uses mode *all* with *maximum* of `1`, the sublist of the first entry may turn out to be empty after all, thus the second
entry of the parent list has to be evaluated. Give the following nested lists:

    [ // flags = all, maximum = 1
        Entry1
        [
            Entry2(chanceNone = 20)
        ],
        Entry3
    ]

`Entry2` may not be chosen for thus `Entry3` is the output. 

When calculating the probabilities, the sublist has a 50% chance of being empty, thus the probability `Entry1` is selected is 80% as well.
Consequently, the probability of `Entry3` being selected is 20%.

Similarly, if the parent list entry has a `chanceNone`, all of them have to be combined:

    [
        // flags = all, maximum = 1
        Entry1(chanceNone = 10)
        [
            Entry2(chanceNone = 20)
        ],
        Entry3
    ]

Thus, the chance `Entry2` is the output is probability of chosing it (90%) multiplied by its sublist not being empty (80%): 72%. The chance
`Entry3` is picked is thus 28%.


### Calculation for mode All

The calculation for this mode depends on the value of the list's *maximum* attribute.

#### maximum is 0

This is perhaps the easiest configuration to calculate because the chance of each entry is independent of the other entries, thus can be calculated from just the entry's own properties.

```javascript
for (var entry of entries) {
    entry.chance = (1 - entry.chanceNone) * entry.conditionChance;
}
```

However, if the entire list has a `chanceNone` and/or `conditionChance`, those have to be included:

```javascript
var listSelfChance = (1 - list.chanceNone) * list.conditionChance;

for (var entry of entries) {
    entry.chance = listSelfChance * (1 - entry.chanceNone) * entry.conditionChance;
}
```

In addition, if the entry has a sublist, that sublist has to be evaluated recursively. This recursive evaluation can
return the **chance of the sublist becoming empty** and thus combined with the product above:

```javascript
var listSelfChance = (1 - list.chanceNone) * list.conditionChance;

for (var entry of entries) {
    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = listSelfChance * (1 - entry.chanceNone) * entry.conditionChance * sublistChance;
}
```

Finally, in case the current list is referenced by a parent list via the same recursive `evaluate()` function call, the chance for this list to be empty has to be calculated too. 

The chance of this list being empty is equal to the product of an entry not having anything by itself for some reason:


```javascript
var empty = 1;

for (var entry of entries) {
    empty *= 1 - entry.chance;
}

return empty;
```

(Of course, the two loops can be combined into one.)

One important property to consider with entries having a sublist is that when [cascading](#calculating-the-cascading-chances) the chances in the tree of leveled lists, the `sublistChance` should not be included
when again. Instead, an extra property called `aprioriChance` should be kept and used for the cascading calculation. 

```javascript
for (var entry of entries) {
    entry.aprioriChance = listSelfChance * (1 - entry.chanceNone) * entry.conditionChance;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;
}
```

If there is no sublist, the `chance` is equal to the `aprioriChance`. (See [this](#calculating-the-cascading-chances) for further details on cascading chances.)

#### maximum is 1

In this type of settings, the chance of picking an item is dependent upon not picking any of the previous items in the list  (because they are not there or have empty sublists for example).

Effectively, this means that when we keep track of the overal chance of emptyness, the entries have consider the chance of all previous items being empty. Luckily, we already track this in `empty`, thus we need to factor that in into the `entry.chance` calculation.

```javascript
var empty = 1;

for (var entry of entries) {
    entry.chance = (1 - entry.chanceNone) * empty;
    empty *= entry.chanceNone;

}
```

This setting is non-intuitive at first, therefore, here is a simplified formulae to demonstrate.

```
P[1] = (1 - ChanceNone[1])
P[2] = (1 - ChanceNone[2]) * ChanceNone[1]
P[3] = (1 - ChanceNone[3]) * ChanceNone[1] * ChanceNone[2]
...
P[n] = (1 - ChanceNone[n]) * ChanceNone[1] * ChanceNone[2] * ... * ChanceNone[n - 1]
```

So the chance of the first is just the presence chance. The chance of the second entry is its presence chance provided the first item was not selected. The second entry chance is its presence chance provided neither the first or second entry was selected before.

For example, if every entry has a 50% chance. The first entry is chosen with **50%** chance, the second entry **25%**, the third **12.5%**, the fourth **6.25%**, etc.


Let's express it via code:


```javascript
var empty = 1;

for (var entry of entries) {
    var selfChance = (1 - entry.chanceNone) * entry.conditionChance;
    entry.aprioriChance = selfChance * empty;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;

    empty *= 1 - selfChance * sublistChance;
}
```

To calculate the emptiness for the subsequent item, we can't include the previous empty value as it would "overcount" the empty cases.

Again, this calculation is non-intuitive, so let's see an example:

    list
    [
        Entry1(chanceNone = 20)
        [
            Entry2(chanceNone = 30)
        ],
        Entry3(chanceNone = 40)
    ]

1. We start out empty `1`
2. With `Entry1`
  - The self chance is `0.8`
  - Thee apriori chance is `0.8 * 1 = 0.8`.
  - The sublist chance of being empty is `0.7`.
  - The entry chance is thus `0.8 * 0.7 = 0.56`
  - The chance for this entry being empty is `1 - 0.8 * 0.7 = 1 - 0.56 = 0.44` 
3. The new overall empty is `1 * 0.44 = 0.44`
4. With `Entry3`
  - The self chance is `0.6`
  - The apriori chance is `0.6 * 0.44 = 0.264`
  - The entry chance is thus `0.264`
  - The chance for this entry being empty is `1 - 0.6 = 0.4`
5. The new overall empty is `0.44 * 0.4 = 0.176`

Therefore, the chances are `Entry1`: **56%**, `Entry3`: **26.4%** and the list being `empty` is **17.6%**. 56 + 26.4 + 17.6 = 100!

But what if the entire list has a `listSelfChance` less than 100% ? We can factor that into `entry.chance` via multiplication. We can't modify the running empty with it and we have to add its reverse to the overall empty value when returning the empty chance of the entire list:

```javascript
   // ...
   entry.chance = selfChance * sublistChance * listSelfChance;
   // ...

   return (1 - listSelfChance) + listSelfChance * empty;
```

In other terms, the chance the entire list is empty is the chance the list itself is to be considered empty (`1 - listSelfChance`) plus the chance of the list being empty after all provided the list shouldn't be empty (`listSelfChance * empty`).

With the given example above, having the list with 40% chance on its own, `Entry1`: **22.4%**, `Entry3`: **10.56%** and the list being `empty` is `0.6 + 0.4 * 0.176` = **67.04%**. Again, 22.4 + 10.56 + 67.04 = 100!


#### maximum is more than 1

If the list should return up to 2 or more items, that complicates things quite considerably. When evaluating an entry, one has to consider the pick chances of previous items in a way that would have lead to the picking of that entry.

For the first *maximum* number of entries, their pick chance is their presence chance because no matter what, those entries will be evaluated.

However, subsequent items are different. For example, if the *maximum is 2*, The 3rd item is only picked if neither the 1st nor 2nd item was picked. In other terms, the 3rd item is picked if both first and second items were *not* picked.

(Sidenote: remember the probability rule of `P(A or B) = 1 - P(!A and !B))`)

```
P[1] = 1 - ChanceNone[1]
P[2] = 1 - ChanceNone[2]
P[3] = (1 - ChanceNone[3]) * (1 - (1 - ChanceNone[1]) * (1 - ChanceNone[2]))
```

The fourth item is not picked if two of any of the previous 3 items have been picked, that is

- The 1st and 2nd items were picked, or
- The 1st and 3rd items were picked, or
- The 2nd and 3rd items were picked.

Let's call define `Q[i] := 1 - ChanceNone[i]` and `CN[i] = ChanceNone[i]` for short. The formula then looks like this:

```
P[3] = Q[3] * (1 - (Q[1] * Q[2]))
P[4] = Q[4] * (1 - (Q[1] * Q[2] + Q[1] * CN[2] * Q[3] + CN[1] * Q[2] * Q[3]))
P[5] = Q[5] * (1 - (Q[1] * Q[2] + Q[1] * CN[2] * Q[3] + CN[1] * Q[2] * Q[3]
                    + Q[1] * CN[2] * CN[3] * Q[4] + CN[1] * Q[2] * CN[3] * Q[4]
                    + CN[1] * CN[2] * Q[3] * Q[4]
                   ))
```

One can see that the that the `1 - ` part keeps expanding with more tags. If we encode `Q[i]` in a bitfield for being 1 at position `i`, and `CN[i]` being 0 at position `i`, we get the following bit strings for `P[i]`s:

```
3: 11
4: 11 101 011
5: 11 101 011 1001 0101 0011
```

The commonality is that these strings all have exactly 2 bits set to 1.

If we do the same process for *maximum* of 3, we get a similar pattern:

```
4: 111
5: 111 1101 1011 0111
6: 111 1101 1011 0111 11001 10101 10011 01101 01011 00111
```

Again, the number of bits set to 1 is always the same: 3. The pattern remains consistent for higher *maximum*s, thus we can generate that sum by using bit patterns.

For this, we'll need a method to count bits set to 1 in an integer. Some languages have direct support for this, others such as JavaScript, require bit-hacks:

```javascript
function bitCount(n) {
    n = n - ((n >> 1) & 0x55555555);
    n = (n & 0x33333333) + ((n >> 2) & 0x33333333);
    return ((n + (n >> 4) & 0xF0F0F0F) * 0x1010101) >> 24;
}
```

To begin, we have to assign the chances to the first *maximum* (`M`) items unconditionally:

```javascript
for (var i = 0; i < M; i++) {
    var entry = entries[i];
    entry.aprioriChance = (1 - entry.chanceNone) * entry.conditionChance;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;
}
```

Subsequent items have to start using the formula generated by the pattern. Let's introduce a `sum` that keeps track of the sum in the formula, thus we don't have calculate `11` and the rest again.

```javascript
var sum = 0;

for (var i = M; i < entries.length; i++) {
    var entry = entries[i];


}

```

Given the entry `i`, we'll have to consider bit strings of length `i - 1` which have as many bits set as `M`. Therefore, we need to loop between certain integer values and test how many bits they have.

Luckily, we don't have to loop for all integers, only between integers (of size `i - 1` bits) that have their highest order bit set up to integers that have all `i` bits of their set:

```javascript
    var minPattern = 1 << (i - 1);
    var maxPattern = (1 << i) - 1;
```

For example, `i = 2` gives (2, 3), `i = 3` gives (4, 7), `i = 4` gives (8, 15).

Then, we increment an integer between these numbers and test for how many bits were set:

```javascript
    
    for (var p = minPattern; p <= maxPattern; p++) {
        if (bitCount(p) == M) {

        }
    }
```

Now that we know `p` has only `M` bits set, we generate a product on this bit pattern, starting from bit index zero up to index `i`, using the entry's chance when it is 1 and using the opposite of the entry's chance if it is 0.

```javascript
            var product = 1;
            for (var b = 0; b < i; b++) {
                var mask = 1 << b;
                var e = entries[b];
                var chance = (1 - e.chanceNone) * e.conditionChance;
                
                if (e.sublist !== undefined) {
                    chance *= 1 - evaluate(e.sublist);
                }
                
                if ((p & mask) != 0) {
                    product *= chance;
                } else {
                    product *= 1 - chance;
                }
            }

            sum += product;
```

Now that we have the sum of the chance combinations from the previous entries, let's assign it to the current entry:

```javascript
    var entry = entries[i];
    entry.aprioriChance = (1 - entry.chanceNone) * entry.conditionChance * (1 - sum);

    var sublistChance = 1;
    if (e.sublist !== undefined) {
        sublistChance *= 1 - evaluate(e.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;
```

All in all, the entire algorithm looks like this:

```javascript
for (var i = 0; i < M; i++) {
    var entry = entries[i];
    entry.aprioriChance = (1 - entry.chanceNone) * entry.conditionChance;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;
}

var sum = 0;

for (var i = M; i < entries.length; i++) {
    var entry = entries[i];

    var minPattern = 1 << (i - 1);
    var maxPattern = (1 << i) - 1;
    
    for (var p = minPattern; p <= maxPattern; p++) {
        if (bitCount(p) == M) {
            var product = 1;
            for (var b = 0; b < i; b++) {
                var mask = 1 << b;
                var e = entries[b];
                var chance = (1 - e.chanceNone) * e.conditionChance;
                
                if (e.sublist !== undefined) {
                    chance *= 1 - evaluate(e.sublist);
                }
                
                if ((p & mask) != 0) {
                    product *= chance;
                } else {
                    product *= 1 - chance;
                }

            }
            sum += product;
        }
    }

    var entry = entries[i];
    entry.aprioriChance = (1 - entry.chanceNone) * entry.conditionChance * (1 - sum);

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance *= 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;
}
```

Of course, this could by itself result in an empty list, for which the probability can be calculated as the product
of none of the entries had a been selected:

```javascript
var empty = 1;

for (var entry of entries) {
    empty *= 1 - entry.chance;
}

return empty;
```

If we want to include the list's own chance:

```javascript
var empty = 1;

for (var entry of entries) {
    empty *= 1 - entry.chance;
    entry.chance *= listSelfChance;
}

return (1 - listSelfChance) + listSelfChance * empty;
```

### Calculation for mode non-All

This mode includes the *for each* and not *for each* settings both, effectively, when **bit 2** of the flags is not set, and **bit 1** might be set.

There are two sub-cases to consider though: 

- if every entry has `conditionChance == 1` and
- if some or all of the entries have a `conditionChance < 1`.

The reason for this is that since conditions are used for pruning the list upfront, an entry might not be on the list to begin with. Since this mode picks an entry with uniform random, that is, 1 / N chance, we don't have a constant N anymore. N could be 0, 1, ..., n based on the entry's `conditionChance`.

#### Uniform pick

When every entry has `conditionChance == 1`, the size of the list is constant, thus every entry has a 1 / N chance of being
picked. Then, the usual sublist and self chances have to be applied.

```javascript
var uniformChance = 1 / entries.length;
var sum = 0;

for (var entry of entries) {

    entry.aprioriChance = (1 - entry.chanceNone) * uniformChance;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;

    sum += entry.chance;
}

return 1 - sum;
```

Here, since the entries are chosen independently, the chance of an empty list is basically subtracting the sum of
overall entry chances from 1.

For example, given the following list

    [
        Entry1(chanceNone = 20),
        Entry2(chanceNone = 40),
        Entry3,
        Entry4
    ]

- `Entry1` is returned `0.25 * 0.8 = 0.2`, 20% of the time,
- `Entry2` is returned `0.25 * 0.6 = 0.15`, 15% of the time,
- `Entry3` is returned 25% of the time,
- `Entry4` is returned 25% of the time and
- The list ifself may be empty is `(1 - (0.2 + 0.15 + 0.25 + 0.25)) = 0.15`, **15%** of the time after all .

To factor in the lists' own chance, simply use it as the numerator in the `uniformChance` expression:

```javascript
var uniformChance = listSelfChance / entries.length;
```

The overall chance for an empty list will be still just the sum of entry chances subtracted from 1.

#### Combinatorial pick

In this mode, since the size of the list is probabilistic as well, we'll have to check for all possible combinations of entries being present to appropriately account for the size of the list to pick from uniform randomly.

For example, given a list of 2 entries `[Entry1, Entry2]` and the shorthand notation from [this](#maximum-is-more-than-1), `Entry1` is picked if the list has only `Entry1` itself (but not `Entry2`); or the list has both `Entry1` and `Entry2` present when `Entry1` is picked with 50% chance:

Note that these chances are only using the `conditionChance` and not the entry related chances because this mode picks an entry first, then determines if that entry should be empty or not independently.

```
C[i] := ConditionChance[i]
Z[i] := 1 - ConditionChance[i];

P[1] = C[1] * Z[2] + C[1] * C[2] / 2
```

Similarly, `Entry2` is picked if the list has only `Entry2` itself (but not `Entry1`); or the list has both `Entry1` and `Entry2` present when `Entry2` is picked with 50% chance:

```
P[2] = Z[1] * C[2] + C[1] * C[2] / 2
```

If we introduce the binary notation from [here](#maximum-is-more-than-1) to, we'll see a pattern emerge:

```
0: 10 11
1: 01 11
```

With 3 items to chose from:

```
0: 100 101 110 111
1: 010 110 011 111
2: 001 101 011 111
```

Again, if a bit is 1 multiply by the presence chance, if the bit is zero, multiply by the not present chance. Then divide each bit pattern by the number of 1 bits.

Let's build the formula. We'll have to consider 2 to the power of the size of the pruned list:

```javascript
var n = entries.length;

var bits = 1 << n;

for (var i = 0; i < bits; i++) {
    // ...
}
```

For each of such bit pattern, calculate the product of present/absent chances:

```javascript
    var product = 1;
    var nonZeroCount = 0;
    
    for (var j = 0; j < n; j++) {
        var mask = 1 << j;
        var entry = entries[j];
    
        var cr = entry.conditionChance;
        if ((i & mask) != 0) {
            product *= cr;
            nonZeroCount++;
        } else {
            product *= (1 - cr);
        }
    }

    // ...
```

Again the chance to pick an entry doesn't depend on the entry itself possibly empty. We count the number of non-zero bits as well to divide the sum later on. Consequently, we have to add the particular product to every entry's chance that were present in the product formula to form the entry's full sum:

```javascript
        if (nonZeroCount != 0) {
            product /= nonZeroCount;

            for (var j = 0; j < n; j++) {
                var mask = 1 << j;
                var entry = entries[j];
                if ((i & mask) != 0) {
                    entry.chance += product;
                }
            }
        }
```

Once the bit list has been evaluated, we can work out each entry's final chance and the chance of the list being empty:

```javascript
var sum = 0;
for (var entry of entries) {
    var baseChance = entry.chance;
    entry.aprioriChance = (1 - entry.chanceNone) * baseChance;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;


    sum += entry.chance;
}

return 1 - sum;
```

The list is empty if none of the entries were selected or were themselves empty due to their `chanceNone` or sublist evaluations.

Again, considering the list's own chance is a matter of multiplying by `listSelfChance`:

```javascript
for (var entry of entries) {
    entry.chance *= listSelfChance;
}

return (1 - sum) * listSelfChance + (1 - listSelfChance);
```

The entire algorithm looks as follows:

```javascript
var n = entries.length;

var bits = 1 << n;

for (var i = 0; i < bits; i++) {
    var product = 1;
    var nonZeroCount = 0;
    
    for (var j = 0; j < n; j++) {
        var mask = 1 << j;
        var entry = entries[j];
    
        var cr = entry.conditionChance;
        if ((i & mask) != 0) {
            product *= cr;
            nonZeroCount++;
        } else {
            product *= (1 - cr);
        }
    }

    if (nonZeroCount != 0) {
        product /= nonZeroCount;

        for (var j = 0; j < n; j++) {
            var mask = 1 << j;
            var entry = entries[j];
            if ((i & mask) != 0) {
                entry.chance += product;
            }
        }
    }
}

var sum = 0;
for (var entry of entries) {
    var baseChance = entry.chance;
    entry.aprioriChance = (1 - entry.chanceNone) * baseChance;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    var chance = entry.aprioriChance * sublistChance;

    entry.chance = chance * listSelfChance;

    sum += chance;
}

return (1 - sum) * listSelfChance + (1 - listSelfChance);
```

#### for-each repeated rolls

If the *for-each* flag is set (bit 1), the probabilities of the entries have to be adjusted in case the parent entry has *quantity* > 1 defined.

For example, let's consider the following list:

    [// flags: for-each
        Entry1,
        Entry2
    ]

If we roll this list once, we get `Entry1` 50% of the time and `Entry2` 50% of the time. However, if we roll it twice, we can get

- `Entry1` twice, with probability `0.5 * 0.5 = 0.25`, *25%**,
- `Entry2` twice, with probability `0.5 * 0.5 = 0.25`, *25%**,
- `Entry1` and `Entry2`, with probability `0.5 * 0.5 = 0.25`, *25%** or
- `Entry2` and `Entry1`, with probability `0.5 * 0.5 = 0.25`, *25%**.

Consequently, the chance of getting `Entry1` once is 25% + 25% + 25% = 75% just by summing up the cases where `Entry1` appears.

However, we can calculate the probability of getting `Entry1` at least once by calculating the chance we don't get `Entry1` at all, then subtract it from 1.
With the exmaple, we won't get `Entry1` if for both rolls, we roll `Entry2`, which has the probability of `0.5 * 0.5 = 0.25`, **25%**. Therefore,
The chance we do roll `Entry1` at least once is `1 - 0.25 = 0.75`, **75%**.

More generally, we can calculate the chance of an entry appearing at least once by calculating the chance the entry won't apper, raise it to the power of the roll count, then subtract it from 1:

```javascript
entry.chance = 1 - Math.pow(1 - entry.chance, rolls);
```

When we want to calculate chance the list is empty, we can no longer sum up the modified entry chances. Instead, we sum up the unmodified entry chances,
which indicates none of the entries were selected in the roll, then raise that amount to the power of the roll count, which indicates none of the rolls had any result.

```javascript
var sum = 0;

for (var entry of entries) {

    entry.chance = ... // calculate it according to the conditionChance status of the entries

    sum += entry.chance;

    entry.chance = 1 - Math.pow(1 - entry.chance, rolls);
}

return Math.pow(1 - sum, rolls);
```


### Calculation for mode First

Calculating the drop chanes for the mode *first* two sub-cases as well:

- If all entries have `conditionChance = 1` specified and
- if some or all of the entries have `conditionChance < 1`.

Both cases are relatively simple to calculate.

#### pick first always

The first case is really simple: the first entry is selected and returned if its chances allow it, or the list is empty. Consequently, all other entries will have zero chance of dropping.

```javascript
var entry = entry[0];

entry.aprioriChance = listSelfChance * (1 - entry.chanceNone);
var sublistChance = 1;
if (entry.sublist !== undefined) {
    sublistChance = 1 - evaluate(entry.sublist);
}

entry.chance = entry.aprioriChance * sublistChance;

return 1 - entry.chance;
```

Since only the first item can be picked, the list being empty is the chance the first item is not picked or is empty after all.

#### pick one

If any or all of the entries have `conditionChance < 1`, that means those entries can't be pruned upfront and their drop chance has to consider the probability of the entry present on the list to begin with.

Unlike the *all* mode with *maximum* of 1, this selection mode will stop when it finds an entry matching its conditions and not continue if the entry picked turned out to be empty itself. Consequently, now every item in the list can be considered for a potential drop.

Let's write down the simplified formulae for a better understanding:

```
P[1] = ConditionChance[1]
P[2] = ConditionChance[2] * (1 - ConditionChance[1])
P[3] = ConditionChance[3] * (1 - ConditionChance[1]) * (1 - ConditionChance[2])
...
P[i] = ConditionChance[i] * (1 - ConditionChance[1]) * (1 - ConditionChance[2]) * ... * (1 - ConditionChance[i - 1])
```

So the first item is picked with its `conditionChance` value. The second item is only picked if the first item was not picked, the third item is picked only if neither the first nor the second item was picked, etc.

For the empty case, we can sum up all the entry chances and subtract it from 1.

```javascript
var sum = 0;
var empty = 1;

for (var entry of entries) {
    var selfChance = (1 - entry.chanceNone) * entry.conditionChance;
    entry.apriori = selfChance * empty;

    var sublistChance = 1;
    if (entry.sublist !== undefined) {
        sublistChance = 1 - evaluate(entry.sublist);
    }

    entry.chance = entry.aprioriChance * sublistChance;

    empty *= 1 - entry.conditionChance;

    sum += entry.chance;
}

return 1 - sum;
```

Note that when cumulating the empty cases, we have to multiply by the entry's `conditionChance` only and not include the entry's own presence chance, because the selection criteria for an entry is only its `conditionChance`. If the entry turns out to be empty after all, that's not the problem of the subsequent entries like with the mode *all* and *maximum* of 1 was.

### Calculating the cascading chances

Given a tree of the leveled lists, we have to be careful when calculating the chances of the nodes in the tree by cascading a parent's probability to its chindren to form the basis for "if the node has been selected" chance.

For example, let's consider the following tree:

    [ // flags 0
        Entry1
        [
            Entry2(chanceNone = 20)
        ],
    ]

Here, the chance of selecting `Entry1` is 50% and getting `Entry2` is `0.5 * 0.8 = 0.4`, **40%**. Thus we could say the chance of `Entry1` being non-empty is 40% too.

However, when we try to cascade the chance of `Entry1` down to see given `Entry1`, what is the chance of `Entry2`, using this 40% would be wrong. It already included the probability of `Entry2` and we would calculate `0.5 * 0.8 * 0.8 = 0.32`, **32%** that is incorrect. We need a probability from `Entry1` that doesn't include the sublist outcome yet.

That's why we calculated the `aprioriChance` of every entry. Here `Entry1`'s apriori chance would be still 50% and the cascaded chance for `Entry2` would be `0.5 * 0.8 = 0.4`, **40%**, the correct value.


