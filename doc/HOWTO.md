# Sudoku Solver

Sudoku is a popular logic-based number placement puzzle.
For an example
and a bit of history, see
[the Wikipeda article](http://en.wikipedia.org/wiki/Sudoku)  One of the interesting bits of history is the role of
  of a Sudoku puzzle generating program in popularizing Sudoku.
  Creating good puzzles is much harder than solving them!

Your program will read a Sudoku board that may be partially
  completed.  A board file contains 9 lines of 9 symbols, each of
  which is either a digit 0-9 or the 
full-stop symbol '.' (also called 'period'  or 'dot')
  to indicate a tile that has not been filled. Your program will first
  check for violations of Sudoku rules.  If the board is valid, 
  then your program will apply two 
simple tactics to fill in an as many open tiles as possible.  

*Aside:  While standard sudoku puzzles are 9x9, rows and columns of any perfect square 
length
will work.  In practice, 4x4 is too simple, and 25x25 far too hard, 
so practical puzzle sizes are 9x9 (the standard) and 16x16 
(a more difficult variant, using hexadecimal digits
01234567890ABCDEF) are the only practical choices.  The rest of 
this HOWTO is written assuming 9x9 puzzles, but in fact our 
solution can be adapted to 16x16 puzzles by changing three 
lines of code in sdk_config.py.*

A valid Sudoku solution has the following properties:

   * In each of the 9 rows, each digit 1..9 appears
    exactly once.  (No duplicates, and no missing digits.)

   * In each of the 9 columns, each digit 1..9 appears
    exactly once.
 
   * The board can be divided into 9 subregion blocks, each 3x3.
    In each of these blocks, each digit 1..9 appears
    exactly once.

If the board contains only the symbols '1' through '9', the pigeonhole
principle ensures that these two properties are equivalent:

  * Each digit appears at least once in a row, column, and block

  * Each digit appears no more than once in a row, column, or block

If the board contains the symbols '1' through '9' and also the
wild-card symbol '.', we say it is incomplete.  We say an incomplete
board is *inconsistent* if any row, column, or block contains more
than one of the symbols '1' through '9', although it may contain
more than one wild-card symbol '.' indicating a choice yet to be
made. 

## The program

Your program will read an input file in the
basic form of the .sdk (Sadman Sudoku) format.
An input file will look like this:

```
...26.7.1
68..7..9.
19...45..
82.1...4.
..46.29..
.5...3.28
..93...74
.4..5..36
7.3.18...
```


If there are no duplicated entries in the board (and regardless
  of whether it is complete, with digits only, or has '.'
  characters indicating tiles yet to be filled), your program will
  proceed to the next step.  If there are duplicated elements,
  your program will reject the puzzle. 

For example, suppose the input board contained this:

```
435269781
682571493
197834562
826195347
374682915
951743628
519326874
248957136
963418257
```

Your program should reject this board. 

If you used the display option like this:
```
$ python3 sudoku.py --display boards/board1.sdk
```
then the program will display the board
as it attempts to solve it. 

## Consistency checking
  
Because of the pigeonhole principle of mathematics, if the board contains only the 
  digits 1..9, the following two statements are
  equivalent: 

  *  None of the 9 digits 1..9 appears more than once in a
  row. 

  * Each of the 9 digits 1..9 appear at least once in a row.

It is therefore enough to  check for duplicates.  Checking for
missing entries is similar but slightly more complicated,
particularly since we are allowing "open" tiles with the 
"." symbol. 

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

## What you must solve

This project comes in two parts.  In the first week we will write 
a simple solver that is efficient but can solve only a limited 
class of puzzle (roughly corresponding to those marked *easy* 
on online Sudoku puzzle sites).  In the second week we will a recursive 
guess-and-check component.  Recursive guess-and-check can 
solve any solvable Sudoku puzzle, but by itself it may be 
very slow.  Combined with the techniques we will program in the 
first week, it is very fast.  Even on a slow computer, I have yet
to find a puzzle that the combined approach cannot solve in 
under a second. 

## Model View Controller, Again

As in our previous tile game, FiveTwelve, we will use a 
Model-View-Controller architecture for the graphical display. 
In FiveTwelve we always connected the view component to 
the model component, because the game would be difficult and not much
fun if we couldn't see the board.  In Sudoku, on the other hand, 
we may or may not want to watch the puzzle be solved.  The solver is 
much faster than the graphics, so if we care most about speed we may 
run our Sudoku solver without graphics (only printing the solution 
when it is complete).   Other times we may wish to attach the 
graphical display and watch the solution in progress.  The main 
program sudoku.py allows both approaches depending on a command-line 
argument.  

## Incremental, but with a plan

Although we will develop our solver incrementally, piece by piece, we 
have a vision of where we are going, including the solution techniques we 
intend to use.  Those solution techniques center on associating with 
each tile a set of *possible values*. We will design with that in 
mind from the beginning, storing in each tile a Python set representing
the candidate values.   

# Step 1: Basic Board

We will create sdk_board.py to hold our Board and Tile classes. 

```python
"""
A Sudoku board holds a matrix of tiles.
Each row and column and also sub-blocks
are treated as a group (sometimes called
a 'nonet'); when solved, each group must contain
exactly one occurrence of each of the
symbol choices.
"""
```

Several basic constants are defined in ```sdk_config.py```. 
That is where we determine the size of the puzzle 
and the symbols to be used in these 
puzzles (digits 1-9 for the 9x9 version of the puzzle). 

```python
from sdk_config import CHOICES, UNKNOWN, ROOT
from sdk_config import NROWS, NCOLS
```

```CHOICES``` will be the symbols (e.g., "123456789"), and
```UNKNOWN``` will be a symbol that indicates a tile that 
has not been solved; we will use "." for that, following 
the standard for puzzle representation set by Sadman Sudoku. 
We describe the size of the puzzle in two ways, for convenience. 
A 9x9 puzzle has 3x3 blocks, so we define that size puzzle as 
having ```NROWS``` and ```NCOLS``` equal 9 but ```ROOT```
(the rows and columns in each block) equal 3.  For a 16x16 puzzle, ROOT 
would be 4 (the square root of 16). 

For debugging it is often handy to have "logging" messages 
that can be turned on or off.  The Python logging package 
provides this.  We could set it to print all debugging 
messages: 

```python
import logging
logging.basicConfig()
log = logging.getLogger(__name__)
log.setLevel(logging.DEBUG)
```

When we don't want so many messages, we can set it to 
to a lower level of verbosity: 

```python
log.setLevel(logging.INFO)
```

We will use the Model-View-Controller architecture, so we'll need events 
generated in the model component (mostly Tile objects in ```sdk_board.py```)
and acted upon in the view component (```sdk_display.py``). 

```python
# --------------------------------
#  The events for MVC
# --------------------------------

class Event(object):
    """Abstract base class of all events, both for MVC
    and for other purposes.
    """
    pass
```

The view component will register listeners, so we need to 
define a ```Listener``` class that it can use: 

```python
# ---------------
# Listeners (base class)
# ---------------

class Listener(object):
    """Abstract base class for listeners.
    Subclass this to make the notification do
    something useful.
    """

    def __init__(self):
        """Default constructor for simple listeners without state"""
        pass

    def notify(self, event: Event):
        """The 'notify' method of the base class must be
        overridden in concrete classes.
        """
        raise NotImplementedError("You must override Listener.notify")
```

Differently from FiveTwelve, there won't actually be any interesting events 
on the board ... our tiles are not moving around!  All the events that 
will need to be displayed will be from individual tiles.  We'll define two 
of them.  ```TileChanged``` will be the event that indicates almost 
any change to a tile (either to the value it holds, or to the set 
of candidate values an unsolved tile *might* hold).  We'll distinguish 
one special kind of tile change for when we add the guess-and-check 
part of the solver in week 2.  It's nice to be able to visually distinguish 
*solving* a tile from just *guessing* its value, so we'll define a 
different event for changes that are made by guessing. 

To use the ```enum.Enum``` class, we'll need to import it near 
the beginning of the file: 

```python
import enum
```

Then we can create the enumeration of kinds of events: 

```python
# --------------------------------------
# Events and listeners for Tile objects
# --------------------------------------

class EventKind(enum.Enum):
    TileChanged = 1
    TileGuessed = 2
```

And we can define a class of events that includes 
the enumeration element: 

```python
class TileEvent(Event):
    """Abstract base class for things that happen
    to tiles. We always indicate the tile.  Concrete
    subclasses indicate the nature of the event.
    """

    def __init__(self, tile: 'Tile', kind: EventKind):
        self.tile = tile
        self.kind = kind
        # Note 'Tile' type is a forward reference;
        # Tile class is defined below

    def __str__(self):
        """Printed representation includes name of concrete subclass"""
        return f"{repr(self.tile)}"
```

Listeners that need to receive TileEvents, like our view component, 
will be TileListeners: 

```python
class TileListener(Listener):
    def notify(self, event: TileEvent):
        raise NotImplementedError(
            "TileListener subclass needs to override notify(TileEvent)")
```

Objects that produce TileEvents, on the other hand, will be 
Listenable.  It is not an accident that class ```Listenable``` looks a lot like ```GameElement``` in our 
FiveTwelve project. 

```python
class Listenable:
    """Objects to which listeners (like a view component) can be attached"""

    def __init__(self):
        self.listeners = [ ]

    def add_listener(self, listener: Listener):
        self.listeners.append(listener)

    def notify_all(self, event: Event):
        for listener in self.listeners:
            listener.notify(event)
```

Finally we are ready to define the Tile class.  


We will want it to know 
what row and column it occupies on the board, not because we will use that 
information in solving puzzles, but because it may be useful for the graphical 
display and in debugging.  We also know that if the tile value is unknown, 
we will want to keep track of the possible values that it *could* hold; this
can be represented as a set, which is a built-in type in Python.  If the 
tile value was given at the start, or if we had determined it, should we 
just represent that as a set with one value, or should we separately 
indicate its current value?  Just for for convenience we will keep it in a separate
field.  The value field will always contain either the special value 
UNKNOWN (defined in ```sdk_config.py```) or one of the values in 
CHOICES (also from ```sdk_config.py```).  The ```candidates``` set should 
be consistent with the ```value```.  

```python
# ----------------------------------------------
#      Tile class
# ----------------------------------------------

class Tile(Listenable):
    """One tile on the Sudoku grid.
    Public attributes (read-only): value, which will be either
    UNKNOWN or an element of CHOICES; candidates, which will
    be a set drawn from CHOICES.  If value is an element of
    CHOICES,then candidates will be the singleton containing
    value.  If candidates is empty, then no tile value can
    be consistent with other tile values in the grid.
    value is a public read-only attribute; change it
    only through the access method set_value or indirectly 
    through method remove_candidates.   
    """
```

We could initialize the tile this way: 

```python
    def __init__(self, row: int, col: int, value=UNKNOWN):
        super().__init__()
        assert value == UNKNOWN or value in CHOICES
        self.row = row
        self.col = col
        self.value = value
        if self.value == UNKNOWN:
            self.candidates = set(CHOICES)
        else: 
            self.candidates = { value }
```

However, we will be setting tile values and candidates in other 
places as we solve the puzzle, so we'll factor out that part of the 
initialization into a separate ```set_value``` method: 

```python
    def __init__(self, row: int, col: int, value=UNKNOWN):
        super().__init__()
        assert value == UNKNOWN or value in CHOICES
        self.row = row
        self.col = col
        self.set_value(value)

    def set_value(self, value: str):
        if value in CHOICES:
            self.value = value
            self.candidates = {value}
        else:
            self.value = UNKNOWN
            self.candidates = set(CHOICES)
        self.notify_all(TileEvent(self, EventKind.TileChanged))
```

It will also need a ```__str__``` method and a ```__repr__``` method, which 
I'll leave to you.  The string form of a tile should just be its value 
(which might be the symbol for UNKNOWN or one of the characters from CHOICES). 
It's repr should look like a call to create a tile, e.g., ```Tile(2, 2, '.')```. 

There is one more method we should give a Tile object. When we 
display the tile graphically, we will want to show candidate values 
for unsolved tiles in the form of "pencil marks".  We'll also need 
 to check candidates later for the *hidden single* solution tactic.  
 We'll provide 
a ```could_be``` method that checks whether a particular symbol 
belongs to the candidates set: 

```python
    def could_be(self, value: str) -> bool:
        """True iff value is a candidate value for this tile"""
        return value in self.candidates

```

That's a lot of code to write before any testing.  There isn't much we 
can do with a Tile object yet, but let's start an ```test_sdk.py``` file 
and fill in a few very basic tests. 

```python
"""Test cases for sdk.py"""

import unittest
from sdk_board import *
from sdk_config import *

class TestTileBasic(unittest.TestCase):

    def test_init_unknown(self):
        tile = Tile(3, 2, UNKNOWN)
        self.assertEqual(tile.row, 3)
        self.assertEqual(tile.col, 2)
        self.assertEqual(tile.value, '.')
        self.assertEqual(tile.candidates, set(CHOICES))
        self.assertEqual(repr(tile), "Tile(3, 2, '.')")
        self.assertEqual(str(tile), ".")

    def test_init_known(self):
        tile = Tile(5, 7, '9')
        self.assertEqual(tile.row, 5)
        self.assertEqual(tile.col, 7)
        self.assertEqual(tile.value, '9')
        self.assertEqual(tile.candidates, {'9'})
        self.assertEqual(repr(tile), "Tile(5, 7, '9')")
        self.assertEqual(str(tile), "9")


if __name__ == "__main__":
    unittest.main()
```

I could have broken them down into a larger number of separate test cases, 
but I want to keep it short and seeing only the first error in each of these test cases is ok since I 
can simply debug and repair them one by one. 

Now we can build the most basic version of our Board class, with a way 
to initialize it from a list: 

```python
# ------------------------------
#  Board class
# ------------------------------

class Board(object):
    """A board has a matrix of tiles"""

    def __init__(self):
        """The empty board"""
        # Row/Column structure: Each row contains columns
        self.tiles: List[Tile] = [ ]
        for row in range(NROWS):
            cols = [ ]
            for col in range(NCOLS):
                cols.append(Tile(row, col))
            self.tiles.append(cols)
```

We would like to be able to "load" a board from a list of lists, 
as we did for FiveTwelve boards.  However, it is tedious to write 
a value like 
```python
[ ['.', '.', '.', '.', '.', '.', '.', '.', '.'], 
  ['.', '.', '.', '.', '.', '.', '.', '.', '.'],
  ['.', '.', '.', '.', '.', '.', '.', '.', '.'],
  ['.', '.', '.', '.', '.', '.', '.', '.', '.'],
  ['.', '.', '.', '.', '.', '.', '.', '.', '.'],
  ['.', '.', '.', '.', '.', '.', '.', '.', '.'],
  ['.', '.', '.', '.', '.', '.', '.', '.', '.'],
  ['.', '.', '.', '.', '.', '.', '.', '.', '.'],
  ['.', '.', '.', '.', '.', '.', '.', '.', '.']]
```

It would be convenient to also allow the list to 
look like 

```python
[".........", ".........", ".........", 
 ".........", ".........", ".........", 
 ".........", ".........", "........."]
```

Since both ```list``` and ```str``` fit the Python abstract 
type ```Sequence```, we will add 

```python
from typing import Sequence
```

Since we know we will be using the List and Set types as well, we might as 
well expand it to 

```python
from typing import Sequence, List, Set
```


near the beginning of the file and specify that any Sequence type 
can be used to set the tile values: 

```python           
    def set_tiles(self, tile_values: Sequence[Sequence[str]] ):
        """Set the tile values a list of lists or a list of strings"""
        for row_num in range(NROWS):
            for col_num in range(NCOLS):
                tile = self.tiles[row_num][col_num]
                tile.set_value(tile_values[row_num][col_num])
```

Now we can add a bit to our test suite: 

```python
class TestBoardBuild(unittest.TestCase):

    def test_initial_board(self):
        board = Board()
        sample_tile = board.tiles[0][0]
        self.assertEqual(sample_tile.value, '.')
        sample_tile = board.tiles[3][3]
        self.assertEqual(sample_tile.value, '.')
        sample_tile = board.tiles[8][8]
        self.assertEqual(sample_tile.value, '.')

    def test_load_board(self):
        board = Board()
        board.set_tiles(["123456789", "2345678991", "345678912",
                         "456789123", "567891234", "678912345",
                         "789123456", "891234567", "912345678"])
        sample_tile = board.tiles[0][0]
        self.assertEqual(sample_tile.value, '1')
        sample_tile = board.tiles[3][5]
        self.assertEqual(sample_tile.value, '9')
        sample_tile = board.tiles[8][8]
        self.assertEqual(sample_tile.value, '8')
```

We're going to want to read puzzles from files.  I have provided
```sdk_reader.py``` for that purpose.  It uses the 
```set_tiles``` method of ```sdk_board.Board```.  It reads 
boards in a subset of the "Sadman Sudoku" (sdk) format, to give us 
access to lots of sample puzzles. 

```
f = open("data/102-ns1-board.sdk")
board = sdk_reader.read(f)
print(board)
<sdk_board.Board object at 0x108ccfa90>
```

The printed form is not very satisfying.  We'll define a 
```__str__``` method for Board that produces a string in the 
same format as the sdk files we read.   It could probably 
be compressed to one line of code with nested list 
comprehensions, but I don't think I could read and 
understand that. On the other hand, I don't want a 20 line 
 method with nested loops.  Here is a reasonably compact definition 
that is (I think) fairly readable: 

```python
    def __str__(self) -> str:
        """In Sadman Sudoku format"""
        row_syms = [ ]
        for row in self.tiles:
            values = [tile.value for tile in row]
            row_syms.append("".join(values))
        return "\n".join(row_syms)
```

And of course a test case for it ... 

```python
class TestBoardIO(unittest.TestCase):

    def test_read_new_board(self):
        board = sdk_reader.read(open("data/00-nakedsubset1.sdk"))
        as_printed = str(board)
        self.assertEqual(as_printed,
            "32...14..\n9..4.2..3\n..6.7...9\n8.1..5...\n...1.6...\n...7..1.8\n1...9.5..\n2..8.4..7\n..45...31")
```

## Are we ready to solve something? 

So far we've got a representation for the board and for tiles, and we 
can read puzzle files and print them.  Before moving on to 
puzzle solving, it would be reassuring and possibly useful to 
build the simplest thing we can that uses the representation of 
candidate lists in tiles. 

One thing we can build now is a check to see whether a partially 
solved board contains any duplicate tiles in a row, column, or block. 
For each row, column, or block, we can build a set of used symbols.
We could do this in many ways, e.g., by counting how many times each 
symbol appears in the group, and rejecting a group if any of the counts 
is greater than 1. Since we are planning to use *sets* of candidates 
in tiles, we slightly prefer an approach that also uses that representation.  

In pseudocode, we can design an algorithm like this: 

``` 
for each group (row, column, or block): 
    used symbols = { }
    for each tile in the group: 
        if the tile is one of CHOICES (anything but UNKNOWN): 
           if the tile's symbol is already in used symbols: 
               return False (board is not consistent)
           else: 
               add the tile's symbol to the used symbols
return True  (the solved part of the board is ok so far)
```

This doesn't look too bad, but when we start trying to turn it 
into Python, we notice that the line

``` 
for each group (row, column, or block): 
```

is really hard to program in a nice way.  Looping through the rows, and tiles in each row
is easy enough.  Looping through the columns is a bit more complex, but 
not too bad.   Looping through the blocks is a good deal more 
complex, but we can figure it out.   But how in the world do we 
combine those into one loop?  Looping through the rows and looping
through the colunns and looping through the blocks all seem to 
require different logic.  We don't want to write the rest of the 
algorithm three times! 

The prospect of duplicating this logic for each kind of group 
is made doubly unattractive when we consider that we will almost 
certainly need to do something similar for each kind of solution 
tactic we apply.   We want to apply multiple techniques that
treat different kinds of groups (rows, columns, and blocks) in the 
same way.  We don't want to duplicate each technique three times! 

## Factoring out group formation

We could make 

``` 
   for each group (row, column, or block):
```

quite simple if we just had a list of groups, where each 
group was a list of the tiles in that group.  We'd still have 
to write the three kinds of loop (one for rows, one for columns, 
and one for blocks), but we could write them just once to build up
the list of groups.  That's what we'll do. 

We will add a new instance variable to the Board representation:  
A list of groups, i.e., a list of lists of tiles.   
We can create it in the constructor. The rows 
can be copied directly from the tiles variable: 

```python
        for row in self.tiles: 
            self.groups.append(row)
```

This is just a copy of self.tiles, but we need the copy because we 
will append additional groups to it.  (Note that ```set_tiles``` 
only changes the values of the tiles, without replacing the Tile objects, 
so it is ok for us to create the groups before the tile values 
have been set.) 

Adding groups for columns is not too difficult.  I'll leave that 
to you.  

The most complex groups to build are the blocks.  For example, 
in a 9x9 board, the blocks are 3x3 with upper left corners at 
(0,0), (0,3), (0,6), (3,0), (3,3), (3,6), (6,0), (6,3), (6,6). 
We want to do this in a way that will also work for other size 
boards.  The ```ROOT``` constant from ```sdk_config.py``` 
gives us the width and height 
of a single block.  The upper left corner of block r,c is 
ROOT * r, ROOT * c, and the tile at position i,j within a block 
is at position ROOT * r + i, ROOT * c + j within the tiles list. 

```python
        for block_row in range(ROOT):
            for block_col in range(ROOT):
                group = [ ] 
                for row in range(ROOT):
                    for col in range(ROOT):
                        row_addr = (ROOT * block_row) + row
                        col_addr = (ROOT * block_col) + col
                        group.append(self.tiles[row_addr][col_addr])
                self.groups.append(group)
```

## How are we going to test this? 

I'm not too worried about the row groups.  The column groups are
a bit more complex, and ought to be tested.  I think I got the logic 
for forming block groups right, but it certainly needs testing. 
But how am I going to do that?   For a 9x9 puzzle, there are 9+9+9 groups; 
I certainly don't want to write a test case in which I manually 
specify what should be in each group (and I wouldn't trust myself
to get it right).  What can I do? 

When we can't directly specify what a result should be, sometimes 
we can specify properties of a result that are easier to check.  We want 
to choose a property that would be likely to be violated if our 
code is buggy.   For example, I know that each tile 
 should appear 
in three different groups (one row, one column, and one block).  If 
I messed up the group formation, it seems likely that I would also 
mess up that property.  So, let's use it to write a test case. 

```python
class TestBoardGroups(unittest.TestCase):

    def test_count_tile_groups(self):
        """Every tile should appear in exactly three groups
        (regardless of board size).
        """
        board = Board()
        counts = { }
        for group in board.groups:
            for tile in group:
                if tile not in counts:
                    counts[tile] = 0
                counts[tile] += 1
        for tile in counts:
            self.assertEqual(counts[tile], 3)        
```

The test case is not perfect.  It's conceivable that I could 
mess up the group formation and still pass the test.  But it's
good enough to give me enough confidence to go on. 

## Detecting duplicates (again)

Now that we have a list of 'groups', we can revisit the 
algorithm we sketched before: 

``` 
for each group (row, column, or block): 
    used symbols = { }
    for each tile in the group: 
        if the tile is one of CHOICES (anything but UNKNOWN): 
           if the tile's symbol is already in used symbols: 
               return False (board is not consistent)
           else: 
               add the tile's symbol to the used symbols
return True  (the solved part of the board is ok so far)
```

Now it's easy!  The outer loop really is one line of code, 
looping through the groups.  I'll let you write the code for 
an ```is_consistent``` method with this signature: 

```python
    def is_consistent(self) -> bool:
```

Remember that ```{ }``` in Python is the empty dictionary rather than 
the empty set, so you'll need to initialize used_symbols as ```set()```. 

Note that ```is_consistent``` does not indicate which values were repeated. 
Later we may want to augment the ```is_consistent``` method with 
notification of a new event to communicate that information.  For now 
let's just make sure it works with a test case.  We'll want to test 
that accepts a proper, complete or incomplete board, and that it 
can reject a board with duplicate symbols in a row, a column, or a block. 

```python
class TestConsistent(unittest.TestCase):
    """Tests of the 'is_consistent' method"""

    def test_good_complete_board(self):
        """This one is from Wikipedia"""
        board = Board()
        board.set_tiles(["534678912", "672195348", "198342567",
                        "859761423", "426853791", "713924856",
                         "961537284", "287419635", "345286179"])
        self.assertTrue(board.is_consistent())

    def test_good_incomplete(self):
        """From Sadman Sudoku"""
        board = Board()
        board.set_tiles(["...26.7.1", "68..7..9.", "19...45..",
                        "82.1...4.", "..46.29..", ".5...3.28",
                        "..93...74", ".4..5..36", "7.3.18..."])
        self.assertTrue(board.is_consistent())

    def test_bad_column(self):
        board = Board()
        board.set_tiles(["1........", ".........", ".........",
                         ".........", ".........", ".........",
                         "1........", ".........", "........."])
        self.assertFalse(board.is_consistent())

    def test_bad_row(self):
        board = Board()
        board.set_tiles([".........", ".........", ".........",
                         ".........", ".2.....2.", ".........",
                         ".........", ".........", "........."])
        self.assertFalse(board.is_consistent())

    def test_bad_block(self):
        board = Board()
        board.set_tiles([".........", "......1..", "........1",
                         ".........", ".........", ".........",
                         ".........", ".........", "........."])
        self.assertFalse(board.is_consistent())
```

# Inferring Values

There are many tactics for inferring the value a tile must 
take.  We will use two of the most basic tactics described at
http://www.sadmansoftware.com/sudoku/solvingtechniques.php . 


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

(The command 'python' or 'python3' may vary depending on your 
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

* Add the column groups (I provided row groups and block groups)

* Write the ```is_consistent``` method

* Write the ```naked_single``` method

* Write the ```hidden_single``` method 






