The Query class is the focal point of the nfldb Python module, as it provides 
an interface to quickly search nfldb's database and return results in 
convenient Python types.

Before we get started, please *briefly familiarize yourself* with the
[API for the Query class](http://pdoc.burntsushi.net/nfldb#nfldb.Query). 
Namely, get a feel for the methods available and get your juices flowing by 
reading a few of the examples.

Also, I encourage you to tweak the examples you find here to satisfy your 
curiosity. In order to that though, you will need to know which statistical 
categories are available to you. Many of them are documented as instance 
variables on the following classes:
[Game](http://pdoc.burntsushi.net/nfldb#nfldb.Game),
[Drive](http://pdoc.burntsushi.net/nfldb#nfldb.Drive),
[Play](http://pdoc.burntsushi.net/nfldb#nfldb.Play),
[PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer) and
[Player](http://pdoc.burntsushi.net/nfldb#nfldb.Player).
But all of the play and player statistics are documented on the
[statistical
categories](Statistical-categories)
page to avoid cluttering the API documentation.


### Getting started

Before we can use the Query class, we first need to connect to the nfldb 
database. If you've followed the [installation 
instructions](Installation) 
successfully, then the following Python program should run successfully:

```python
import nfldb
db = nfldb.connect()
```

The
[nfldb.connect](http://pdoc.burntsushi.net/nfldb#nfldb.connect)
function will read your configuration file, connect to the database, perform 
any available schema upgrades to your database and return a handle to the 
database. `nfldb.connect` can also take specific connection information if you 
don't want to use a configuration file.

The next step is running a successful query. The
[nfldb.Query](http://pdoc.burntsushi.net/nfldb#nfldb.Query)
constructor takes a `db` parameter that it uses to connect to the database.
Search criteria can then be specified with the following methods:
[game](http://pdoc.burntsushi.net/nfldb#nfldb.Query.game),
[drive](http://pdoc.burntsushi.net/nfldb#nfldb.Query.drive),
[play](http://pdoc.burntsushi.net/nfldb#nfldb.Query.play),
[player](http://pdoc.burntsushi.net/nfldb#nfldb.Query.player) or
[aggregate](http://pdoc.burntsushi.net/nfldb#nfldb.Query.aggregate).
Once criteria have been specified, results can be retrieved with one of the 
following methods:
[as_games](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_games),
[as_drives](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_drives),
[as_plays](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_plays),
[as_play_players](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_play_players),
[as_players](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_players) or
[as_aggregate](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_aggregate).

If all is well, then the following should print the all of the preseason games 
of the 2013 season for the Patriots:

```python
import nfldb
db = nfldb.connect()

q = nfldb.Query(db)
q.game(season_year=2013, season_type='Preseason', team='NE')
for g in q.as_games():
    print g
```

And the output:

```bash
[andrew@Liger nflgame] python2 scratch.py 
Preseason 2013 week 1 on 08/09 at 07:30PM, NE (31) at PHI (22)
Preseason 2013 week 2 on 08/16 at 08:00PM, TB (21) at NE (25)
Preseason 2013 week 3 on 08/22 at 07:30PM, NE (9) at DET (40)
Preseason 2013 week 4 on 08/29 at 07:30PM, NYG (20) at NE (28)
```

From here on out, examples will omit the first two lines that import nfldb and 
connect to the database.


### Feeling the flexibility

The query interface can be composed in many different ways. For example, the 
above example could be written this way:

```python

q = nfldb.Query(db)
q.game(season_year=2013)
q.game(season_type='Preseason')
q.game(team='NE')
for g in q.as_games():
    print g
```

Or we could write it like this:

```python
q = nfldb.Query(db)
q.game(season_year=2013).game(season_type='Preseason').game(team='NE')
for g in q.as_games():
    print g
```

And you'll get the exact same output. This is because the Query class 
**combines all criteria conjunctively**. In other words, *all criteria must be 
satisfied for a result to be included*. (You can read about specifying criteria 
where *any one criterion* needs to be matched instead of all of them in the
[disjunctive 
criteria](Disjunctive-criteria)
article.)


### Mix and match criteria and results

Just because we used the
[game](http://pdoc.burntsushi.net/nfldb#nfldb.Query.game)
method to specify criteria doesn't mean we *have* to use the
[as_games](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_games)
method to retrieve results. For example, we could do a similar search as 
before, but get the results as drives:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular', team='NE', week=1)
for d in q.as_drives():
    print d
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py 
[End of Game ] BUF from OWN 20 to OWN 20 (lasted 00:05 - Q4 00:05 to Q4 00:00)
[Field Goal  ] NE  from OWN 34 to OPP 17 (lasted 04:26 - Q4 04:31 to Q4 00:05)
[Punt        ] BUF from OPP 42 to OPP 44 (lasted 01:11 - Q3 00:28 to Q4 14:17)
[End of Half ] NE  from OWN 20 to OWN 20 (lasted 00:34 - Q2 00:34 to Q2 00:00)
...
```

(If you're bothered by the order of the drives, see the
[sorting results](Sorting-results)
article.)

Or we could retrieve the results as plays:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular', team='NE', week=1)
for p in q.as_plays():
    print p
```

And the output is:

```bash
(UNK, N/A, Final) END GAME
(UNK, N/A, Final) End of game - 4.15 pm
(BUF, OWN 20, Q4, 1 and 10) (:05) (Shotgun) E.Manuel pass short middle to F.Jackson to BUF 32 for 12 yards. FUMBLES, recovered by BUF-E.Manuel at BUF 29. E.Manuel to BUF 29 for no gain (A.Dennard).
(NE, OWN 35, Q4) S.Gostkowski kicks 65 yards from NE 35 to end zone, Touchback. Kick through end zone.
(NE, OPP 17, Q4, 3 and 13) (:09) S.Gostkowski 35 yard field goal is GOOD, Center-D.Aiken, Holder-R.Allen.
...
```

Or we can flip it around and ask for any games in which Julian Edelman had a 
touchdown in the 2012 regular season:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.player(full_name='Julian Edelman').play_player(offense_tds=1)
for g in q.as_games():
    print g
```

(Note that the `offense_tds` field is actually a derived field and not part of 
the nfldb database schema. It is documented on the
[statistical categories](Statistical-categories)
page.)

And the output of the above code is:

```bash
[andrew@Liger nflgame] python2 scratch.py
Regular 2012 week 3 on 09/23 at 08:20PM, NE (30) at BAL (31)
Regular 2012 week 11 on 11/18 at 04:25PM, IND (24) at NE (59)
Regular 2012 week 12 on 11/22 at 08:20PM, NE (49) at NYJ (19)
```

Of course, we can also fetch the results as plays too:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.player(full_name='Julian Edelman').play_player(offense_tds=1)
for p in q.as_plays():
    print p
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py             
(NE, OPP 7, Q2, 2 and 7) (:07) (Shotgun) T.Brady pass short middle to J.Edelman for 7 yards, TOUCHDOWN.
(NE, OPP 2, Q3, 3 and 2) (11:10) (Shotgun) T.Brady pass short left to J.Edelman for 2 yards, TOUCHDOWN.
(NE, OWN 44, Q2, 3 and 5) (3:17) (Shotgun) T.Brady pass deep left to J.Edelman for 56 yards, TOUCHDOWN.
```


### Down with equality

Searching with comparison operators like `<=` or `>` is supported. You can use 
them by appending a special suffix to a field. The following are the valid 
suffixes:

* **__lt** - `<` or "less than"
* **__le** - `<=` or "less than or equal to"
* **__gt** - `>` or "greater than"
* **__ge** - `>=` or "greater than or equal to"
* **__ne** - `!=` or "does not equal"

And by default no suffix means `==` or "equal to".

To see them in action, we might want to find all of Tom Brady's passes for at 
least 60 yards in the 2012 regular season:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.player(full_name='Tom Brady').play_player(passing_yds__ge=60)
for p in q.as_plays():
    print p
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
(NE, OWN 17, Q2, 1 and 10) (9:56) (Shotgun) T.Brady pass short left to S.Vereen for 83 yards, TOUCHDOWN.
(NE, OWN 37, Q3, 3 and 10) (10:00) (Shotgun) T.Brady pass deep middle to D.Stallworth for 63 yards, TOUCHDOWN. NE 12-Brady 18th career game with 4+ TD passes, passing Johnny Unitas for 4th most all-time.
```

Or we could even look for all passes between 40 and 50 yards:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.player(full_name='Tom Brady')
q.play_player(passing_yds__ge=40, passing_yds__le=50)
for p in q.as_plays():
    print p
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
(NE, OWN 33, Q1, 2 and 5) (8:57) (Shotgun) T.Brady pass deep right to R.Gronkowski to BUF 26 for 41 yards (J.Rogers). Caught at BUF 32, slanting to sideline.
(NE, OPP 46, Q1, 1 and 10) (5:59) (No Huddle, Shotgun) T.Brady pass deep left to W.Welker for 46 yards, TOUCHDOWN.
(NE, OPP 44, Q3, 2 and 1) (2:09) (No Huddle, Shotgun) T.Brady pass deep right to M.Hoomanawanui to SF 3 for 41 yards (D.Whitner).
```


## More reading

This concludes a basic introduction to the Query interface. We've gone over how 
to run a simple query, mixing and matching search criteria with different kinds 
of results, and how to use comparison operators for broader searching.

While this is enough to get you started, there are a few follow up articles 
that go more in depth with some features:

* [Sorting results](Sorting-results)
* [Disjunctive criteria](Disjunctive-criteria)
* [Fuzzy player name matching](Fuzzy-player-name-matching)
* [Exploring secondary types: FieldPosition, PossessionTime and Clock](Exploring-secondary-types:-FieldPosition,-PossessionTime-and-Clock)
* [Aggregate searching](Aggregate-searching)
* [Breaking the shackles of the query interface with nfldb's types](Breaking-the-shackles-of-the-query-interface-with-nfldb's-types)
* [More examples](More-examples)

