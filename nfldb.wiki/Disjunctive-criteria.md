Sometimes you don't want to use all conjunctions, but allow some results to be 
returned if *any one criterion* matches the result. For example, how could you 
get the first four Patriots games of the 2012 regular season? Well, you could 
use what you know about the Query class already and filter the results returned 
in Python:

```python
q = nfldb.Query(db)
games = q.game(season_year=2012, season_type='Regular', team='NE').as_games()
first_four = filter(lambda g: g.week <= 4, games)
for g in first_four:
    print g
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py 
Regular 2012 week 1 on 09/09 at 01:00PM, NE (34) at TEN (13)
Regular 2012 week 2 on 09/16 at 01:00PM, ARI (20) at NE (18)
Regular 2012 week 3 on 09/23 at 08:20PM, NE (30) at BAL (31)
Regular 2012 week 4 on 09/30 at 01:00PM, NE (52) at BUF (28)
```

While that is perfectly OK, the Query interface will let you specify a 
*list* of values for a field instead of a single value, which will match if the 
field is equal to *any* of the values in the list. Using that knowledge, we can 
also get the first four games like so:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular', team='NE', week=[1, 2, 3, 4])
for g in q.as_games():
    print g
```

And the output will be the same as above.

This is nice and all, but what if you want to combine more than one field 
disjunctively? This can be done by using the
[QueryOR](http://pdoc.burntsushi.net/nfldb#nfldb.QueryOR)
function, which creates a regular Query class, but it combines *all* criteria 
disjunctively. For example, to get every game in nfldb where either the home 
or the away team had at least 59 points:

```python
q = nfldb.QueryOR(db)
q.game(home_score__ge=59, away_score__ge=59)
for g in q.as_games():
    print g
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py 
Regular 2009 week 6 on 10/18 at 04:15PM, TEN (0) at NE (59)
Preseason 2010 week 3 on 08/26 at 08:00PM, IND (24) at GB (59)
Regular 2010 week 7 on 10/24 at 04:15PM, OAK (59) at DEN (14)
Regular 2010 week 10 on 11/15 at 08:30PM, PHI (59) at WAS (28)
Regular 2011 week 7 on 10/23 at 08:20PM, IND (7) at NO (62)
Regular 2012 week 11 on 11/18 at 04:25PM, IND (24) at NE (59)
```

But wait! What if you only wanted the games from the 2012 regular season? You 
might try something like:

```python
q.game(season_year=2012, season_type='Regular')
q.game(home_score__ge=59, away_score__ge=59)
```

But this will end up giving you every game that matches *any* of those 
criteria. So that means all games in 2012, all regular season games (from any 
year), and any games where either team scored at least 59 points.

In order to remedy this, we can combine multiple queries where one query is 
conjunctive and the other is disjunctive. So to restrict our search for big 
scoring games to only the 2012 regular season, we can make one disjunctive 
query for the big scores, and another query for the conjunctive criteria that 
games be in the 2012 regular season. We can them combine the queries with the
[andalso](http://pdoc.burntsushi.net/nfldb#nfldb.Query.andalso)
method:

```python
big_scores = nfldb.QueryOR(db).game(home_score__ge=59, away_score__ge=59)

q = nfldb.Query(db).game(season_year=2012, season_type='Regular')
q.andalso(big_scores)
for g in q.as_games():
    print g
```

And this correctly outputs:

```bash
[andrew@Liger nflgame] python2 scratch.py 
Regular 2012 week 11 on 11/18 at 04:25PM, IND (24) at NE (59)
```

Finally, remember that we can specify as much additional criteria as we want. 
This includes providing more conditions on the same field. For example, instead 
of looking for big scores, perhaps we want to look for *extreme* scores that 
include games where a team had at least 59 points *or* games where a team was 
shutout:

```python
extreme_scores = nfldb.QueryOR(db)
extreme_scores.game(home_score__ge=59, away_score__ge=59)
extreme_scores.game(home_score=0, away_score=0)

q = nfldb.Query(db).game(season_year=2012, season_type='Regular')
q.andalso(extreme_scores)
for g in q.as_games():
    print g
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py 
Regular 2012 week 4 on 09/30 at 01:00PM, SF (34) at NYJ (0)
Regular 2012 week 11 on 11/18 at 04:25PM, IND (24) at NE (59)
Regular 2012 week 14 on 12/09 at 04:25PM, ARI (0) at SEA (58)
Regular 2012 week 15 on 12/16 at 01:00PM, NYG (0) at ATL (34)
Regular 2012 week 15 on 12/16 at 01:00PM, TB (0) at NO (41)
Regular 2012 week 15 on 12/16 at 04:25PM, KC (0) at OAK (15)
Regular 2012 week 17 on 12/30 at 04:25PM, MIA (0) at NE (28)
```

To illustrate the logic actually used here, we can dump an approximation of the 
WHERE clause used in the corresponding SQL with the
[show_where](http://pdoc.burntsushi.net/nfldb#nfldb.Query.show_where)
method.

```python
extreme_scores = nfldb.QueryOR(db)
extreme_scores.game(home_score__ge=59, away_score__ge=59)
extreme_scores.game(home_score=0, away_score=0)

q = nfldb.Query(db).game(season_year=2012, season_type='Regular')
q.andalso(extreme_scores)

print q.show_where()
```

And the output is (manually formatted):

```sql
game.season_type = 'Regular' AND game.season_year = 2012
AND (game.away_score >= 59
     OR game.home_score >= 59
     OR game.away_score = 0
     OR game.home_score = 0)
```

