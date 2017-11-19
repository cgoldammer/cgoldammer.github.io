# Why use Either?

Haskell's `Either` is an unsual beast. I've used Haskell for two years now, and up to a week ago, `Either` has only felt like a bad version of `Maybe`. When I got an `Either`, my first instinct was to use a function like `EitherC.rightToMaybe` and turn it into a `Maybe`.

Does `Either` have good use cases? Yes! And I think I was missing those use cases because a long experience in Python made me debug my code ineffectively. I would've saved a lot of time using `Either`. So let's look at some real code.

I am currently building a library to parse chess games. Chess games get stored as a text file that provides individual moves. This format is called PGN and the input data is of type `[String]`: A list of `String`, with one `String` for each move. An example input with perfectly legal moves (in other words, this PGN should parse) is

```
pgn = ["e4", "e5", "Nf3", "Nc6", "Bb5"]
```

Now I have to parse those strings into a `Game`. Think of a `Game` as a Haskell object that understands chess. For instance, I could use a `Game` to tell me what moves are possible after each move.

The problem is that parsing the input can go wrong, either because the input data is corrupted or because my parser is wrong. The latter case is the real problem: Chess is not a trivial game, and if I have made a mistake in coding up the logic of the game, I can't fully parse the game.

Since this parsing can fail, I must somehow allow for failure. I started out with the function signature of `pgnGame :: [String] -> Maybe Game`. This looks great, doesn't it? If I try to parse, it tells me whether parsing succeeded or failed. But, in practice, it's not good at all: What do I do if I try to parse a game and I obtain `Nothing`? I know that I started with `[String]`, and somewhere along the line, this parsing broke. What went wrong? I opened a shell with `stack ghci` and then I ran the following tests:

```
>>> pgnGame $ take 1 pgn -- Does it work for the first move? It does, thus continue
>>> pgnGame $ take 2 pgn -- Second move is also fine
>>> pgnGame $ take 3 pgn -- Aah, it breaks at "Nf3". I've made a mistake in coding up the logic of a chess game!
```

If you're experienced in Haskell, you're likely to get angry at me for wasting so much time. But if you come from a language like Python, this seems completely natural. That's simply how it's done in Python: You use the shell to interactively diagnose errors. If needed, you'll also seek to prevent them using unit tests. But the shell is a natural place to start.

In Haskell, there's a better way. My way of thinking about it is that, if parsing fails, I obtain a `Nothing`. But that's not very informative. It tells me that something went wrong, but it doesn't tell my why and where it went wrong. But when I'm parsing a game, I already know when the parsing is going wrong, I was simply throwing this information away.

Let's look at the actual code for the function (simplified for exposition):

```
pgnGameFromGs :: [String] -> Either String Game
pgnGameFromGs pgnMoves = fmap (Game gs) eitherMoves
  where fs = pgnAsMoves pgnMoves
        startingGS = fromJust $ fst $ head fs
        movesWithPiece = tail $ fmap snd fs -- [Maybe (Piece, Move)]
        moves = ((fmap . fmap) snd) movesWithPiece -- [Maybe Move]
        eitherMoves = foldl foldGameEither (Right []) moves
```

The last two lines are most relevant: Im trying to parse a game into a list of `Move`. However, parsing can fail, therefore I need to work with `[Maybe Move]`. Once I have that, simply need a function that turns this list into a nice error description in case I ever encounter a `Nothing` along the way:

```
foldGameEither :: Either String [Move] -> Maybe Move -> Either String [Move] 
foldGameEither (Right mvs) Nothing = Left $ "Parsed moves until " ++ show mvs
foldGameEither (Right mvs) (Just mv) = Right $ mvs ++ [mv]
foldGameEither (Left err) _ = Left err
```

With these changes, I no longer need to guess where parsing failed. If it fails, the return type will let me know at which move this happened.

There's another takeaway for me. When you look at this code, you'll note that the `Either` could tell me more. It only tells me where the parsing failed. It'd be nicer if it also told me why parsing the move failed. Did the move contain wrong characters? Did the move try to move a piece that doesn't exist? Did the move use uppercase instead of lowercase? Deliberately, I decided that I would not include this information in the return type. I think this is the right decision because there are quickly diminishing returns: Yes, it would be nice to know why parsing didn't succeed, but it would be a lot of work to thread this logic through all my code, probably too much added code (and too much added complexity) to justify the time savings in debugging.

To me, here's the biggest takeaway: Haskell offers data abstractions that aren't easily available in a lot of other languages. It's tempting to demand that we use those abstractions maximally, but in practice it comes down to having experience and good judgment on what abstractions to use. In my own library, I learned that `Maybe Game` will cause me added work, that `Either String Game` solves most of these problems, and I'm stopping there with the guess that more advanced return type would likely be overkill. It's easy to see beginners (that includes me) struggling with too-simple types, but that's a normal and healthy part of advancing as a developer. 

`Either` is a beautiful tool, but using `Either` is part of a learning process that simply takes time. If you don't see the use yet, that's to be expected! It will happen naturally when you try to build more advanced libraries that deal with real-world use cases.


