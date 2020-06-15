# Fallout 76 Leveled List drop chances

## Terminology

### `chanceNone`

The game has properties specifying a probability (rolled against with a PRNG) that an entry or list **will not be used**.

For example, if a list has a `chanceNone` of 80, 80% of the time when evaluating the list, there won't be any results.

`chanceNone` defines the absense probability percentage (i.e, when calculating the `chanceNone` has to be divided by 100).

[Example list: LVLI:00064B92 <LPI_Clothes_LocTheme>](https://nukacrypt.com/database/json/00064B92)

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

First step is to get the list's `chanceNone` value from:

- `LVCV` attribute
- `LVLG` gets a global value
- `LVCT` points to a curve table for which the indexer will be the global value of `LVLG`

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

Given the flag attribute `LVLF` and its bit 0 state:

- if set, all entries are kept which have minimum level less than or equal to the *subject/target level*

    Example: given a list of

        [
            Entry1(minLevel = 1), 
            Entry2(minLevel = 10)
        ]

    if *subject/target level* is 5, only `Entry1` is kept. 
    If *subject/target level* is 10, both `Entry1` and `Entry2` are kept.

- if clear, entries are kept which have a minimum level closest to the *subject/target level* and not exceeding it

    Example: given a list of 

        [
            Entry1(minLevel = 1), 
            Entry2(minLevel = 10), 
            Entry3(minLevel = 10), 
            Entry4(minLevel = 20)
        ]

    if *subject/target level* is 5, only `Entry1` is kept. 
    If *subject/target level* is 10, only `Entry2` and `Entry3` are kept.
    If *subject/target level* is 19, only `Entry2` and `Entry3` are kept.
    If *subject/target level* is 20, only `Entry4` is kept.


#### entry conditions

The list entries can have their own conditions that determine if those entries should be considered at all. If an entry's
condition doesn't pass, it will be ignored in the subsequent step.

Thus, the game builds a pruned list by removing any entries whose conditions don't pass.

For example, given a list of 

    [
       Entry1 (HasKeyword(K1) == 1),
       Entry2 ()
    ]

If keyword `K1` is not present on the subject/target, the pruned list will only hold `Entry2`. If `K1` is present,
the pruned list will hold both `Entry1` and `Entry2`



### 4. pick an entry or entries

The evaluation of the pruned entry list can happen in multiple ways, depending on the flags associated with the leveled list.

The flag attribute `LVLF` is a bitfield indicating how to proceed.

- If bit 0 is set, 

## Calculating exact drop chances