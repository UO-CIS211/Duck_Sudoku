# Solving Simple Sudoku Puzzles

## Outline 

In this part you will construct
a solver that uses *constraint propagation* to 
solve simple puzzles.  In the second (and easier)
part, you will use the constraint propagation 
solver as one part of a solver that performs 
recursive guess-and-check search to solve 
*all* Sudoku puzzles.  

The instructions below begin with a description 
of Sudoku and of the solving strategy.  
Don't skip it, because the step-by-step 
instructions that follow won't make sense 
if you can't see where you are going. The 
step-by-step instructions begin in a section 
titled "Step 1", which is creating the 
Sudoku `Board` class.  It also describes how
to set up *logging* (very helpful in debugging), 
and once again we set up a Model-View-Controller
structure so that we can have a graphical 
view of the puzzle being solved. 

As in the FiveTwelve project, you will create a 
`Tile` class and a `Board` class, and a `Board` 
object will contain a list of lists of 
`Tile` objects.  The `Tile` objects for 
Sudoku are a good deal more complex than the 
`Tile` objects in FiveTwelve.  Each tile keeps 
track not only of the symbol it currently 
holds, but also of the symbols it *could* hold, 
constrained by the symbols currently held by 
other tiles in the same row, column, or block. 

A `Board` object will not only include a list
of lists of `Tile` objects, but will also 
contain a list containing lists of subsets of the tiles 
grouped in different ways.  You must devise 
part of the code to build these lists.  To 
understand what this structure is and why 
we are building it, be sure to read the 
[supplement on aliasing](https://uo-cis211.github.io/chapters/04_1_Alias).

You will also write a method `is_consistent` 
that determines whether a Sudoku board contains
any duplicate digits in a row, column, or block. 

The next part (in the section *Inferring Values*)
begins to provide the constraint propagation logic
used to solve simple puzzles.  You will write 
a method `naked_single` that implements a basic
solution tactic.  At this point you will have 
a working solver that can solve  
some very simple Sudoku puzzles. 

Next you will implement a tactic called 
*hidden single*.  I will provide you some 
high level pseudocode and guidance, but you 
will write the method.  When you have 
implemented method `hidden_single` as well as 
`naked_single`, you will have a complete 
solver for most of the Sudoku problems 
you find online that are marked *easy*. 

Although the solver you build in this part will 
not be able to completely solve many Sudoku 
puzzles ranked intermediate or hard, it is the 
major part of a complete solver.  The full solver
performs a 
systematic, recursive *guess-and-check* 
algorithm.  This part performs the 
*check* part and is a major component
of the *guess* part. 


## Completion with constraint propagation

If the board is consistent, then (and only then) your program
will apply two simple constraint propagation 
tactics to fill some of the empty tiles,
then print the resulting board.  These constraints are based
directly on the properties of a completed Sudoku puzzle,
viz., that each symbol must appear once but only once in each
row, column, and block. 

```
$ more data/board-sandiway-intermediate.sdk
.2.6.8...
58...97..
....4....
37....5..
6.......4
..8....13
....2....
..98...36
...3.6.9.
$ python3 sudoku.py data/board-sandiway-intermediate.sdk
.2.6.8...
58...97..
....4....
37....5..
6......74
..8....13
...92....
..98...36
...3.6.9.
```

In the example above, only a few tiles have been filled in,
  because only simple tactics have been used.  If you use the
   --display option, you can see progress in filling in tiles,
  including elimination of some candidates:

Constraint propagation alone is enough to solve
some easy puzzles: 

```
$ more board-incomplete-easy1.sdk
...26.7.1
68..7..9.
19...45..
82.1...4.
..46.29..
.5...3.28
..93...74
.4..5..36
7.3.18...
$ python3 sudoku.py data/board-incomplete-easy1.sdk
435269781
682571493
197834562
826195347
374682915
951743628
519326874
248957136
763418259
```

# Inferring Values

There are many tactics for inferring the value a tile must 
take.  We will use two of the most basic tactics described at
[The Sadman Sudoku site](http://www.sadmansoftware.com/sudoku/solvingtechniques.php). 
You may also find it useful to review the *pencil marks* 
technique that many people use to keep track of possible 
tile values, because our *candidates* structure is essentially
a Python representation of pencil marks;  
[Sadman's tutorial on pencil marks](https://www.learn-sudoku.com/pencil-marks.html)
is a good introduction. 

## Naked Single

The simplest of these is called "naked single", "sole candidate", or 
"singleton".  If you solve Sudoku by hand, you may use "pencil marks"
to indicate the values that can appear in a tile.  If a 3 appears
anywhere in the row, you mark off the 3 in the "pencil marks". 
If a 3 appears anywhere in the column or block, you likewise 
mark off the "3" in the pencil marks for tile.  If there is only 
one pencil mark that is not crossed off, then we can determine 
that the tile must hold that value. 

If we applied this tactic tile-by-tile, scanning the row, column, 
and block that the tile belongs to, we would scan each row, column, and block 
in a 9x9 puzzle up to 81 times.  We can do much better.  
Rather than focusing on one cell, we will work group-by-group. 
We will scan each group twice:  Once to get the set of symbols 
used in that group, and a second time to eliminate those symbols
from the candidate sets of all the unknown tiles in the group. 

What should the naked_single method return?  There are three 
possible outcomes: 

* We made some progress in solving the puzzle, by eliminating 
  at least one candidate in at least one tile.  (This includes 
  the case in which we were able to determine a tile 
  value.)
  
* We made no progress --- after applying Naked Single, the 
  candidate sets are exactly as they were before. 
  
* We eliminated all the candidates from some tile, showing that 
  this puzzle is not solvable.  Note that this can happen even if 
  there are no duplicates in any row, column, or block! 
  
It is most convenient to have the return value of the result 
be an indicator of whether or not there was progress.  We can 
treat the third possibility, an unsolvable puzzle, separately
next week.  

We will add to the Tile class a method for eliminating 
candidates and potentially changing the tile value from 
UNKNOWN to one of the CHOICES.  It will return an indication 
of whether the candidate list was changed: 

```python
    def remove_candidates(self, used_values: Set[str]):
        """The used values cannot be a value of this unknown tile.
        We remove those possibilities from the list of candidates.
        If there is exactly one candidate left, we set the
        value of the tile.
        Returns:  True means we eliminated at least one candidate,
        False means nothing changed (none of the 'used_values' was
        in our candidates set).
        """
        new_candidates = self.candidates.difference(used_values)
        if new_candidates == self.candidates:
            # Didn't remove any candidates
            return False
        self.candidates = new_candidates
        if len(self.candidates) == 1:
            self.set_value(new_candidates.pop())
        self.notify_all(TileEvent(self, EventKind.TileChanged))
        return True
```

```Tile.remove_candidates``` can be called from ```Board.naked_single```, 
with this header: 

```python
    def naked_single(self) -> bool:
        """Eliminate candidates and check for sole remaining possibilities.
        Return value True means we crossed off at least one candidate.
        Return value False means we made no progress.
        """
```

I leave writing ```naked_single``` to you.  We can test it 
with an example from Sadman Sudoku: 

```python
class TestNakedSingle(unittest.TestCase):
    """Simple test of Naked Single using row, column, and block
    constraints.  From Sadman Sudoku,
    http://www.sadmansoftware.com/sudoku/nakedsingle.php
    """
    def test_sadman_example(self):
        board = Board()
        board.set_tiles([".........", "......1..", "......7..",
                         "......29.", "........4", ".83......",
                         "......5..", ".........", "........."])
        progress = board.naked_single()
        self.assertTrue(progress, "Should resolve one tile")
        progress = board.naked_single()
        self.assertTrue(progress, "A few candidates should be eliminated from other tiles")
        progress = board.naked_single()
        self.assertFalse(progress, "No more progress on this simple example")
        self.assertEqual(str(board),
            ".........\n......1..\n......7..\n......29.\n........4\n.83...6..\n......5..\n.........\n.........")
```

## Solving a puzzle! 

Although we have only a single, simple technique, we can 
now solve some simple puzzles.  We will add a ```solve```
method to ```Board``` that just calls ```naked_single``` 
again and again as long as the Naked Single tactic makes 
some progress. 

```python
    def solve(self):
        progress = True
        while progress:
            progress = self.naked_single()
        return
```

Now we could write another test case (and we should), but it's 
more satisfying to see a puzzle solved.  In the terminal, we 
can try 

```
 python3 sudoku.py -d data/102-ns1-board.sdk 
```

(The command 'python', 'py',  or 'python3' may vary depending on your 
operating system.)  If all is well, we should be able to watch 
our Sudoku program solve a simple puzzle.

Of course we should also do this in the form of a test case, without 
graphical interaction: 

```python
    def test_naked_single_one(self):
        """This puzzle can be solved with multiple rounds of naked single."""
        board = Board()
        board.set_tiles(["...26.7.1", "68..7..9.", "19...45..",
                         "82.1...4.", "..46.29..", ".5...3.28",
                         "..93...74", ".4..5..36", "7.3.18..."])
        board.solve()
        self.assertEqual(str(board),
                         "\n".join(["435269781", "682571493", "197834562",
                                    "826195347", "374682915", "951743628",
                                    "519326874", "248957136", "763418259"]))

```

## Hidden Single

Suppose that after applying the hidden single tactic to eliminate 
some candidates, all of the unkown tiles still have at least 
two candidate values.  But suppose only one of those tiles has
candidate value 3.  Even though the tile that has candidate value 3
may have other candidate values, we know it must hold the 3 because 
there is no other place to put it.  This is the *hidden single* 
solving technique. 

In pseudocode, we can summarize the hidden single technique this way: 

```python
for each group of tiles (columns, rows, blocks):
    for each value in CHOICES: 
        if value is not already on some tile in the group: 
           count the number of tiles for which value is a candidate.
           If the number is 1, then put the value in that tile. 
```

We can break this down a little more. Let's first find the set 
of values in CHOICES that have not been placed on any tile. 
In pseudocode

```
for each group of tiles: 
    leftovers = copy of CHOICES
    for each tile in group:
        if tile.value in CHOICES, remove it from leftovers
    for each value in leftovers: 
        (count and place as before)
```

I will leave this method to you.  As before, it should return 
True if it makes progress (which will only be if it manages 
to place a value in a tile) and False otherwise.  One note: Be careful 
to make a copy of CHOICES before computing leftovers.  If you 
write 
```python
     leftovers = CHOICES
```
you will just be making a reference to the CHOICES list.  Copy 
it into a set with 

```python
    leftovers = set(CHOICES)
```

Hidden single works only in combination with another tactic that eliminates candidates, 
like naked single. Hidden single can only contribute progress if naked single makes 
progress, so in our ```solve``` method we don't need a separate check to see 
whether ```hidden single``` made progress.  ```solve``` can become

```python
    def solve(self):
        """Repeat solution tactics until we 
        don't make any progress, whether or not
        the board is solved. 
        """
        progress = True
        while progress:
            progress = self.naked_single()
            self.hidden_single()
        return
```

Here are test cases for the hidden single tactic: 

```python
class TestHiddenSingle(unittest.TestCase):
    """Test the Hidden Single tactic, which must be combined with the
    naked single tactic.
    """

    def test_hidden_single_example(self):
        """Simple example from Sadman Sudoku. Since 2 is blocked
        in two columns of the board, it must go into the middle
        column.
        """
        board = Board()
        board.set_tiles([".........", "...2.....",  ".........",
                         "....6....", ".........",  "....8....",
                         ".........", ".........", ".....2..."])
        board.naked_single()
        board.hidden_single()
        self.assertEqual(str(board),
                         "\n".join(
                        [".........", "...2.....",  ".........",
                         "....6....", "....2....",  "....8....",
                         ".........", ".........", ".....2..."]))


    def test_hidden_single_solve(self):
        """This puzzle can be solved with naked single
        and hidden single together.
        """
        board = Board()
        board.set_tiles(["......12.", "24..1....", "9.1..4...",
                         "4....365.", "....9....", ".364....1",
                         "...1..5.6", "....5..43", ".72......"])
        board.solve()
        self.assertEqual(str(board),
                         "\n".join(["687539124", "243718965", "951264387",
                                    "419873652", "725691438", "836425791",
                                    "394182576", "168957243", "572346819"]))
```

# Are we done?

We are done for now.  With naked single and hidden single, our 
Sudoku solver can handle all the puzzles that most online puzzle 
sources rank as "easy".  Next week we will add recursive 
guess-and-check to solve all other Sudoku puzzles, no matter how 
hard. 

## Summary of your parts 

Did you miss anything?  Here are the parts I left to you: 

* Write the ```naked_single``` method

* Write the ```hidden_single``` method 

## Next steps

When your constraint propagation is working correctly, you 
are ready to add a 
[guess-and-check procedure](HOWTO-GUESS.md) that will use 
your constraint propagation as a step between guesses. 







