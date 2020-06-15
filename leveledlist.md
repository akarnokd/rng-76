# Fallout 76 Leveled List drop chances

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

One complication is that if **bit 2** of the flag is set (called *use all* in tools), the level evaluation will always work like the first method above.

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

More specifically, if this list is referenced by a parent leveled list, the current list is evaluatet over and over, specified by the
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

The inner list will be evaluated twice, each time picking Entry2 or Entry3 with 50% chance. Therefore, the output of the main list can be:

- 25% of the time, `Entry2` appears twice
- 25% of the time, `Entry3` appears twice
- 50% of the time, both `Entry2` and `Entry3` appears.

However, from a drop chance perspective, we consider the probability that an entry appears in the output at least once, therefore, 
the resulting distribution can be coalesced into the drop chance of 50% for `Entry2` and 50% for `Entry3`. Consequently, the
`Quantity` of the parent (indicating the reroll count) has no consequence for the probability of a sub-entry appearing at least once.

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


#### **bit 7** is set

If **bit 7** is set (named *first entry where conditions match* in  tools), only the first entry will be consulted.

This first entry can have its `chanceNone` defined and non-zero, the output may not contain that many entries after all. An entry's `chanceNone` is a percentage extracted from the

- `LVOV` attribute,
- `LVOC` global value or
- `LVOT` curve table where the indexer is the `LVOC` global value.

Thus, if a PRNG is rolled less than the entry's `chanceNone` percentage, the output will be **empty**. Otherwise, the output will be this first entry.

## Calculating exact drop chances