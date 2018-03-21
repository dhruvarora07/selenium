If you're only working with data contained inside nfldb, it's unlikely you'll 
ever need to worry about player names being inconsistent. However, if you have 
data from different sources (maybe from scraping a web site), then it might be 
useful if you can match players from another source to players in nfldb.

In an ideal world, this would be as simple as testing string equality on a 
player's full name. While this might work for the majority of cases, it can 
frustratingly fail on a small number of cases. For example, I recall a 
particularly nasty name matching bug where one source used `Stevie Johnson` 
while another used `Steve Johnson` to refer to the same wide receiver on the 
Buffalo Bills.

Even worse, different players can have precisely the same name. So there must 
be a way to filter based on other criteria like team and position.

nfldb attempts to partially solve this problem by providing the
[player_search](http://pdoc.burntsushi.net/nfldb#nfldb.player_search)
function, which takes a database handle and a full name and returns the closest 
matching player in the database along with a similarity score.
(Similarity is measured with the
[Levenshtein distance](http://en.wikipedia.org/wiki/Levenshtein_distance)
between a player's name in the database and the name provided to the
`player_search` function.)

For example, I'd much rather call my quarterback `tommy brady`, but his name in 
the database is `Tom Brady`. No matter! The `player_search` function doesn't 
care:

```python
player, dist = nfldb.player_search(db, 'tommy brady')
print 'Similarity score: %d, Player: %s' % (dist, player)
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
Similarity score: 4, Player: Tom Brady (NE, QB)
```

The similarity score (or more specifically, the edit or Levenshtein distance) 
is `4`. The lower the score, the better. Namely, the number refers to the 
minimum number of character changes between the target string (`Tom Brady`) and 
the query string (`tommy brady`). In this case, the `my` in `tommy` needs to be 
deleted, and the `t` and `b` need to be changed to their corresponding 
capitalized letters. A score of `0` represents an exact match.

Typically, when using data from other sources, there is more information about 
each player than just their name. Usually his current position or team is also 
available. When that's the case, `player_search` can restrict results returned 
to only players on a particular team and/or at a particular position.

For example, if we want to look up Robert Griffin III, we might try:

```python
player, dist = nfldb.player_search(db, 'robert griffin')
print 'Similarity: %d, Player: %s' % (dist, player)
```

But this outputs:

```bash
[andrew@Liger nflgame] python2 scratch.py
Similarity: 2, Player: Robert Griffin (UNK, UNK)
```

Which is probably not the player we're looking for since Robert Griffin III is 
currently playing for the Redskins and shouldn't have an unknown team or 
position. If we add some context, then we get better results:

```python
player, dist = nfldb.player_search(db, 'robert griffin', team='WAS')
print 'Similarity: %d, Player: %s' % (dist, player)

player, dist = nfldb.player_search(db, 'robert griffin', position='QB')
print 'Similarity: %d, Player: %s' % (dist, player)
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
Similarity: 6, Player: Robert Griffin III (WAS, QB)
Similarity: 6, Player: Robert Griffin III (WAS, QB)
```

One last parameter is `limit`, which allows you to specify that you want a 
*list* of best matching names. This is useful when simple filtering by team and 
position isn't enough. For example, we can see the best 5 results matching 
`robert griffin` with:

```python
matches = nfldb.player_search(db, 'robert griffin', limit=5)
for (player, dist) in matches:
    print 'Similarity score: %d, Player: %s' % (dist, player)
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
Similarity score: 2, Player: Robert Griffin (UNK, UNK)
Similarity score: 6, Player: Robert McClain (ATL, CB)
Similarity score: 6, Player: John Griffin (NYJ, RB)
Similarity score: 6, Player: Robert Griffin III (WAS, QB)
Similarity score: 6, Player: Robert Ortiz (UNK, UNK)
```

Finally, the results of `player_search` can be *correctly* integrated with the 
query interface by using the
[player_id](http://pdoc.burntsushi.net/nfldb#nfldb.Player.player_id)
attribute of the `Player` object returned. The reason we use `player_id` 
instead of a player's full name is that `player_id` is guaranteed to be unique 
by the database, while the player's full name might not be.

A simple example to find all of RGIII's rushing touchdowns:

```python
player, _ = nfldb.player_search(db, 'robert griffin', team='WAS')

q = nfldb.Query(db)
q.player(player_id=player.player_id)
for p in q.play(rushing_tds=1).sort([('gsis_id', 'asc'), ('time', 'asc')]).as_plays():
    print p
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
(WAS, OPP 5, Q1, 1 and 5) (:23) (Shotgun) R.Griffin left end for 5 yards, TOUCHDOWN.
(WAS, OPP 7, Q3, 2 and 7) (5:33) (Shotgun) R.Griffin up the middle for 7 yards, TOUCHDOWN.
(WAS, OPP 2, Q4, 2 and 1) (3:38) (No Huddle, Shotgun) R.Griffin up the middle for 2 yards, TOUCHDOWN.
(WAS, OPP 5, Q2, 1 and 5) (7:32) (Shotgun) R.Griffin up the middle for 5 yards, TOUCHDOWN.
(WAS, OPP 7, Q3, 1 and 7) (9:43) R.Griffin left tackle for 7 yards, TOUCHDOWN.
(WAS, OWN 24, Q4, 3 and 6) (2:56) (Shotgun) R.Griffin left guard for 76 yards, TOUCHDOWN.
(WAS, OPP 10, Q3, 1 and 10) (3:17) (Shotgun) R.Griffin left end for 10 yards, TOUCHDOWN.
```

