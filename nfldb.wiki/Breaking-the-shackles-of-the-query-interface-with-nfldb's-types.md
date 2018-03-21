One of the main features of nfldb is to provide convenient types for 
representing data in the database. As a general rule, each table in the 
database corresponds to a single class in 
[nfldb/types.py](../blob/master/nfldb/types.py) and *each row* in that table 
corresponds to a single *instance* of that table's class.

For example, if we get the games played in week 1 of the 2013 season, the 
return value of
[as_games](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_games)
is a *list of instances of the Game class*. (Or equivalently, *Game objects*.)
We can demonstrate this with an example by inspecting the type of the return 
value:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular', week=1)
games = q.as_games()
print type(games[0])
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
<class 'nfldb.types.Game'>
```

We can go a little further and use `dir(games[0])` to look at the available 
attributes:

```bash
[andrew@Liger nflgame] python2 scratch.py
['__class__', '__delattr__', '__doc__', '__format__', '__getattribute__', 
'__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', 
'__repr__', '__setattr__', '__sizeof__', '__slots__', '__str__', 
'__subclasshook__', '_as_sql', '_db', '_drives', '_from_nflgame', 
'_from_schedule', '_row', '_save', '_sql_columns', '_sql_derived', 
'_sql_fields', '_table', 'away_score', 'away_score_q1', 'away_score_q2', 
'away_score_q3', 'away_score_q4', 'away_score_q5', 'away_team', 
'away_turnovers', 'day_of_week', 'drives', 'finished', 'from_id', 'from_row', 
'gamekey', 'gsis_id', 'home_score', 'home_score_q1', 'home_score_q2', 
'home_score_q3', 'home_score_q4', 'home_score_q5', 'home_team', 
'home_turnovers', 'loser', 'play_players', 'players', 'plays', 'season_type', 
'season_year', 'start_time', 'time_inserted', 'time_updated', 'week', 'winner']
```

While the `dir` function is useful for such things, nfldb has comprehensive API 
documentation that includes instance variables like `day_of_week` or 
`home_team`. You can view documentation for each attribute in the
[documentation for the Game 
class](http://pdoc.burntsushi.net/nfldb#nfldb.Game).
(Note that attributes starting with an underscore are not part of the public 
interface and should not be used.)

You can do similar things for the other `as_*` methods of the query interface. 
The following is a list of all the `as_*` methods with a link to the API 
documentation for the types of objects they return:

* [as_players](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_players)
  returns a list of
  [Player](http://pdoc.burntsushi.net/nfldb#nfldb.Player) objects.
* [as_games](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_games)
  returns a list of
  [Game](http://pdoc.burntsushi.net/nfldb#nfldb.Game) objects.
* [as_drives](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_drives)
  returns a list of
  [Drive](http://pdoc.burntsushi.net/nfldb#nfldb.Drive) objects.
* [as_plays](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_plays)
  returns a list of
  [Play](http://pdoc.burntsushi.net/nfldb#nfldb.Play) objects.
* [as_play_players](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_play_players) 
  returns a list of 
  [PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer) objects.
* [as_aggregate](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_aggregate) 
  returns a list of 
  [PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer) objects.

**Whenever you need to access data in the database, the aforementioned methods 
in the query interface should be the primary means of getting that data**. This 
is primarily because some of the `as_*` methods (like `as_plays` and 
`as_aggregate`) are optimized to be fast in the presence of a lot of data. 
Also, I believe the query interface is convenient for common use cases.

However, it is not mandatory to use the query interface. The rest of this 
article will be dedicated to accessing data *without* the query interface (or, 
at most, with help from the query interface). Since the query interface is 
meant to be fast and hide the details of writing SQL, this necessarily includes 
a discussion on performance and sometimes writing SQL by hand.


### Looking up data by primary key

Sometimes you just want to examine a particular data point like a single game. 
Each primary type in nfldb comes with a `from_id` static function that allows 
you to fetch a single row from the corresponding table by using its primary key 
identifier.

For example, for games, you can use the
[Game.from_id](http://pdoc.burntsushi.net/nfldb#nfldb.Game.from_id)
function to retrieve one game using its GSIS identifier:

```python
ne_buf = nfldb.Game.from_id(db, '2013090800')
print ne_buf
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
Regular 2013 week 1 on 09/08 at 01:00PM, NE (23) at BUF (21)
```

Note that each of the `from_id` functions takes a database and a value 
corresponding to a row's primary key value. For example, for drives, the 
primary key is the combination of the GSIS identifier and the drive sequence 
number.


### Walking the hierarchy of data

If you haven't read the
[article describing the data model](The-data-model)
or haven't
[perused the entity-relationship diagrams](The-data-model#er-diagrams),
you should do so now. In particular, pay attention to the relationships modeled 
in the ER diagrams.

The relationships modeled in the ER diagram imply a *hierarchy* of data between 
entities. For example, games contain drives, drives contain plays. The reverse 
is also true: each play belongs to a drive and each drive belongs to a game.

There are other hierarchies too. For example, a play contains zero or more 
player statistics where each player statistic corresponds to a single player. 
Said differently, each play contains zero or more players.

All of these relationships are manifest in the attributes of the types 
described earlier. For example, if you have a `Game` object, then you can
access the drives of that game via the
[drives](http://pdoc.burntsushi.net/nfldb#nfldb.Game.drives)
attribute:

```python
ne_buf = nfldb.Game.from_id(db, '2013090800')
for drive in ne_buf.drives:
    print drive
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
[Punt        ] NE  from OWN 14 to OPP 44 (lasted 02:55 - Q1 15:00 to Q1 12:05)
[Fumble      ] BUF from OWN 10 to OWN 11 (lasted 00:32 - Q1 12:05 to Q1 11:33)
[Touchdown   ] NE  from OPP 16 to OPP 9  (lasted 00:47 - Q1 11:33 to Q1 10:46)
...
```

The `drives` attribute is implemented as a
[Python property](http://docs.python.org/2/library/functions.html#property),
which means that accessing it actually calls a method, but that fact is hidden 
from you. Namely, it would be very expensive to populate *every* game object 
with every drive every time a game object is created. Instead, the `drives` 
property will fetch drives from the database *only when it is first accessed*. 
Therefore, if you never need the drives for a particular game, you don't need 
to pay the cost of fetching them. Of particular note is that the cost is only 
paid *once* for each object. So if you do:

```python
drives = game.drives
again = game.drives
```

then only the first access of `game.drives` will query the database. Subsequent 
accesses will return the same data immediately from a cache.

As you might have guessed, `drives` is not the only property like this. There 
are many others that are designed to let you walk the hierarchy of data. The 
following table shows all of them.

Type | Relationship | Attribute | Return type
--- | --- | --- | ---
[Game](http://pdoc.burntsushi.net/nfldb#nfldb.Game) | has many | [drives](http://pdoc.burntsushi.net/nfldb#nfldb.Game.drives) | list of [Drive](http://pdoc.burntsushi.net/nfldb#nfldb.Drive)
[Game](http://pdoc.burntsushi.net/nfldb#nfldb.Game) | has many | [plays](http://pdoc.burntsushi.net/nfldb#nfldb.Game.plays) | list of [Play](http://pdoc.burntsushi.net/nfldb#nfldb.Play)
[Game](http://pdoc.burntsushi.net/nfldb#nfldb.Game) | has many | [play_players](http://pdoc.burntsushi.net/nfldb#nfldb.Game.play_players) | list of [PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer)
[Game](http://pdoc.burntsushi.net/nfldb#nfldb.Game) | has many | [players](http://pdoc.burntsushi.net/nfldb#nfldb.Game.players) | list of pairs (team_name, [Player](http://pdoc.burntsushi.net/nfldb#nfldb.Player))
[Drive](http://pdoc.burntsushi.net/nfldb#nfldb.Drive) | belongs to a | [game](http://pdoc.burntsushi.net/nfldb#nfldb.Drive.game) | a [Game](http://pdoc.burntsushi.net/nfldb#nfldb.Game)
[Drive](http://pdoc.burntsushi.net/nfldb#nfldb.Drive) | has many | [plays](http://pdoc.burntsushi.net/nfldb#nfldb.Drive.plays) | list of [Play](http://pdoc.burntsushi.net/nfldb#nfldb.Play)
[Drive](http://pdoc.burntsushi.net/nfldb#nfldb.Drive) | has many | [play_players](http://pdoc.burntsushi.net/nfldb#nfldb.Drive.play_players) | list of [PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer)
[Play](http://pdoc.burntsushi.net/nfldb#nfldb.Play) | belongs to a | [drive](http://pdoc.burntsushi.net/nfldb#nfldb.Play.drive) | a [Drive](http://pdoc.burntsushi.net/nfldb#nfldb.Drive)
[Play](http://pdoc.burntsushi.net/nfldb#nfldb.Play) | has many | [play_players](http://pdoc.burntsushi.net/nfldb#nfldb.Drive.play_players) | list of [PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer)
[PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer) | belongs to a | [play](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer.play) | a [Play](http://pdoc.burntsushi.net/nfldb#nfldb.Play)
[PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer) | belongs to a | [player](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer.player) | a [Player](http://pdoc.burntsushi.net/nfldb#nfldb.Player)

This sort of access is particularly useful when you want to see the context of 
some statistics. For example, you've undoubtedly seen my example for computing 
the top quarterbacks in a season by passing yards:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
for pp in q.sort('passing_yds').limit(5).as_aggregate():
    print pp.player, pp.passing_yds
```

Since `as_aggregate` returns a list of `PlayPlayer` objects, we can use the 
`player` attribute to access data about the player that recorded those 
statistics. In this case, `pp.player` is a `Player` which has all available 
meta data on the player in `pp`. e.g., `pp.player.full_name` is the player's 
full name.

But this example serves as a cautionary tale for two reasons. Firstly, you 
might try using the `play` attribute here:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
for pp in q.sort('passing_yds').limit(5).as_aggregate():
    print pp.play, pp.passing_yds
```

But this outputs:

```bash
[andrew@Liger nflgame] python2 scratch.py
None 5177
None 4965
None 4903
None 4799
None 4719
```

Why? Because the `PlayPlayer` type is being abused to contain aggregate 
statistics **from multiple plays**. Therefore, the notion of it belonging to a 
single play no longer exists. (It remains to be seen whether this break in 
abstraction is worthy.)

The second cautionary tale here is that each access of the `player` attribute 
on distinct `PlayPlayer` objects is guaranteed to run a single SQL query. In a 
small loop, this is no big deal. But if you have a few hundred or a thousand 
results, you might start seeing a slow down.

This brings us to our next section.


### Walking can be slow

It may appear relatively benign to try something like this:

```python
ne_buf = nfldb.Game.from_id(db, '2013090800')
for play in ne_buf.plays:
    print play
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
(BUF, OWN 35, Q1) D.Carpenter kicks 64 yards from BUF 35 to NE 1. L.Blount to NE 14 for 13 yards (D.Searcy).
(NE, OWN 14, Q1, 1 and 10) (14:56) PENALTY on BUF-M.Dareus, Neutral Zone Infraction, 5 yards, enforced at NE 14 - No Play.
(NE, OWN 19, Q1, 1 and 5) (14:56) T.Brady pass short right to D.Amendola to NE 29 for 10 yards (J.Rogers). Slant pattern, caught at NE 25, crossing to middle.
(NE, OWN 29, Q1, 1 and 10) (14:24) S.Ridley up the middle to NE 32 for 3 yards (M.Lawson). Buffalo challenged the runner was down by contact ruling, and the play was Upheld. (Timeout #1.)
(NE, OWN 32, Q1, 2 and 7) (13:47) (Shotgun) T.Brady pass incomplete short middle to D.Amendola. Thrown wide of receiver at NE 38, crossing from left.
```

But let's look at the implementation of
[Game.plays](http://pdoc.burntsushi.net/nfldb#nfldb.Game.plays):

```python
@property
def plays(self):
    """
    A list of `nfldb.Play` objects in this game. Data is retrieved
    from the database if it hasn't been already.
    """
    plays = []
    for drive in self.drives:
        for play in drive.plays:
            plays.append(play)
    return plays
```

The access to `self.drives` executes one query to get all drives for this game. 
But *for each drive* the
[Drive.plays](http://pdoc.burntsushi.net/nfldb#nfldb.Drive.plays)
attribute is accessed, which itself runs a query. Therefore, the number of
queries in the above code is `1 game * # of drives`. This number can grow 
*very* quickly if one isn't careful. For example:

```python
q = nfldb.Query(db)
games = q.game(season_year=2012, season_type='Regular').as_games()
for game in games:
    for play in game.plays:
        for pp in play.play_players:
          print pp.player.full_name
```

Where the total number of queries executed here is equal to `(# of games in 2012 
season) * (# of drives) * (# of plays) * (# of player stats)`. Which ends up 
being about ~150,000 queries. You might be willing to pay for that on a local 
machine (the above code takes about 1 minute on mine), but if you're querying a 
remote database, the network latency is likely going to make it unfeasible to 
run the above code.

The simple solution to this problem is to be careful when traversing the 
hierarchy of data manually *when dealing with large data sets*. On small 
amounts of data (say, for a week's worth of games), you probably don't have to 
be too careful depending on your constraints.

Of course, the other solution is to just use the query interface, which 
naturally avoids queries in loops by allowing you to search the database 
without walking the hierarchy of data.


### Using regular SQL queries with nfldb's types

When you really need it, you can drop down into SQL to make things go faster. 
Typically, this is done when you have more specific needs than what the query 
interface gives you (or the hierarchy attributes).

The nice thing is that there are simple examples of doing this in nfldb 
already! For example, let's look at the implementation of the
[Game.drives](http://pdoc.burntsushi.net/nfldb#nfldb.Game.drives)
attribute, which fetches a list of drives from the database:

```python
@property
def drives(self):
    """
    A list of `nfldb.Drive`s for this game. They are automatically
    loaded from the database if they haven't been already.

    If there are no drives found in the game, then an empty list
    is returned.
    """
    if self._drives is None:
        self._drives = []
        with Tx(self._db) as cursor:
            cursor.execute('''
                SELECT %s FROM drive WHERE gsis_id = %s
                ORDER BY start_time ASC, drive_id ASC
            ''' % (select_columns(Drive), '%s'), (self.gsis_id,))
            for row in cursor.fetchall():
                d = Drive.from_row(self._db, row)
                d._game = self
                self._drives.append(d)
    return self._drives
```

We can narrow this code down to its relevant bits and provide a runnable 
example:

```python
gsis_id = '2013090800'  # the NE @ BUF 2013 season opener
drives = []
with nfldb.Tx(db) as cursor:
    cursor.execute('''
        SELECT %s FROM drive WHERE gsis_id = %%s
        ORDER BY start_time ASC, drive_id ASC
    ''' % nfldb.select_columns(nfldb.Drive), (gsis_id,))
    for row in cursor.fetchall():
        drives.append(nfldb.Drive.from_row(db, row))

for drive in drives:
    print drive
```

And the output is:

```bash
[andrew@Liger nflgame] time python2 scratch.py
[Punt        ] NE  from OWN 14 to OPP 44 (lasted 02:55 - Q1 15:00 to Q1 12:05)
[Fumble      ] BUF from OWN 10 to OWN 11 (lasted 00:32 - Q1 12:05 to Q1 11:33)
[Touchdown   ] NE  from OPP 16 to OPP 9  (lasted 00:47 - Q1 11:33 to Q1 10:46)
...
```

There are three important things to note in the above code. Firstly, we 
generate the columns to select automatically by using the
[select_columns](http://pdoc.burntsushi.net/nfldb#nfldb.select_columns)
function. This ensures that all of the necessary information to construct a 
`Drive` is included in each row. Secondly, we select drives in a particular 
game by searching for drives belonging to a particular `gsis_id`. Thirdly and
finally, we use the
[Drive.from_row](http://pdoc.burntsushi.net/nfldb#nfldb.Drive.from_row)
function to construct a `Drive` from a SQL result row. This function is
strictly a convenience and is a thin wrapper around the actual `Drive`
[constructor method](http://pdoc.burntsushi.net/nfldb#nfldb.Drive.__init__).
You may use the constructor method yourself, particularly when you want to
change the columns you select.

Note that dropping down into SQL means that you need to deal with the 
`psycopg2` module directly (which is a popular PostgreSQL driver for Python).
You should therefore read about
[basic module usage](http://initd.org/psycopg/docs/usage.html)
with psycopg2 if you haven't used it before.

A similar approach can be used with other types of data, like players, plays 
and player statistics.


### A case study: squeezing the juice with SQL

So, I've told you that dropping down into SQL is sometimes necessary and I've 
shown you how to do it. But *when* is it really necessary? The answer depends 
on your constraints and how many computing resources you're willing to devote
to nfldb. Whatever the case, I'll now show an example on how to improve the 
performance of aggregate queries by writing them yourself.

While you are probably very tired of the example showing the top quaterbacks 
by passing yards, it is once again instructive. Once more, here's the code with 
player names replaced with their identifiers to avoid a query inside the loop:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
for pp in q.sort('passing_yds').limit(5).as_aggregate():
    print pp.player_id, pp.passing_yds
```

And the output:

```bash
[andrew@Liger nflgame] python2 scratch.py
00-0020531 5177
00-0026498 4965
00-0021678 4903
00-0019596 4799
00-0026143 4719
```

On my system, despite the small result set, it takes about `1.3` seconds to 
complete. While the time it takes for the results to be returned is due to 
various things (like the size of the data to aggregate or the number of 
conditions to check), we can at least take a look at the actual aggregate 
query for the above code:

```sql
SELECT
  play_player.player_id,
  SUM(play_player.defense_ast) AS defense_ast, 
  SUM(play_player.defense_ffum) AS defense_ffum,
  SUM(play_player.defense_fgblk) AS defense_fgblk,
  SUM(play_player.defense_frec) AS defense_frec, 
  SUM(play_player.defense_frec_tds) AS defense_frec_tds, 
  SUM(play_player.defense_frec_yds) AS defense_frec_yds, 
  SUM(play_player.defense_int) AS defense_int,
  SUM(play_player.defense_int_tds) AS defense_int_tds,
  SUM(play_player.defense_int_yds) AS defense_int_yds, 
  SUM(play_player.defense_misc_tds) AS defense_misc_tds, 
  SUM(play_player.defense_misc_yds) AS defense_misc_yds, 
  SUM(play_player.defense_pass_def) AS defense_pass_def, 
  SUM(play_player.defense_puntblk) AS defense_puntblk, 
  SUM(play_player.defense_qbhit) AS defense_qbhit,
  SUM(play_player.defense_safe) AS defense_safe,
  SUM(play_player.defense_sk) AS defense_sk, 
  SUM(play_player.defense_sk_yds) AS defense_sk_yds,
  SUM(play_player.defense_tkl) AS defense_tkl,
  SUM(play_player.defense_tkl_loss) AS defense_tkl_loss, 
  SUM(play_player.defense_tkl_loss_yds) AS defense_tkl_loss_yds, 
  SUM(play_player.defense_tkl_primary) AS defense_tkl_primary, 
  SUM(play_player.defense_xpblk) AS defense_xpblk, 
  SUM(play_player.fumbles_forced) AS fumbles_forced, 
  SUM(play_player.fumbles_lost) AS fumbles_lost, 
  SUM(play_player.fumbles_notforced) AS fumbles_notforced, 
  SUM(play_player.fumbles_oob) AS fumbles_oob,
  SUM(play_player.fumbles_rec) AS fumbles_rec,
  SUM(play_player.fumbles_rec_tds) AS fumbles_rec_tds, 
  SUM(play_player.fumbles_rec_yds) AS fumbles_rec_yds, 
  SUM(play_player.fumbles_tot) AS fumbles_tot,
  SUM(play_player.kicking_all_yds) AS kicking_all_yds,
  SUM(play_player.kicking_downed) AS kicking_downed, 
  SUM(play_player.kicking_fga) AS kicking_fga,
  SUM(play_player.kicking_fgb) AS kicking_fgb,
  SUM(play_player.kicking_fgm) AS kicking_fgm, 
  SUM(play_player.kicking_fgm_yds) AS kicking_fgm_yds, 
  SUM(play_player.kicking_fgmissed) AS kicking_fgmissed, 
  SUM(play_player.kicking_fgmissed_yds) AS kicking_fgmissed_yds, 
  SUM(play_player.kicking_i20) AS kicking_i20,
  SUM(play_player.kicking_rec) AS kicking_rec,
  SUM(play_player.kicking_rec_tds) AS kicking_rec_tds, 
  SUM(play_player.kicking_tot) AS kicking_tot,
  SUM(play_player.kicking_touchback) AS kicking_touchback,
  SUM(play_player.kicking_xpa) AS kicking_xpa,
  SUM(play_player.kicking_xpb) AS kicking_xpb,
  SUM(play_player.kicking_xpmade) AS kicking_xpmade,
  SUM(play_player.kicking_xpmissed) AS kicking_xpmissed, 
  SUM(play_player.kicking_yds) AS kicking_yds,
  SUM(play_player.kickret_fair) AS kickret_fair,
  SUM(play_player.kickret_oob) AS kickret_oob, 
  SUM(play_player.kickret_ret) AS kickret_ret,
  SUM(play_player.kickret_tds) AS kickret_tds,
  SUM(play_player.kickret_touchback) AS kickret_touchback, 
  SUM(play_player.kickret_yds) AS kickret_yds,
  SUM(play_player.passing_att) AS passing_att,
  SUM(play_player.passing_cmp) AS passing_cmp, 
  SUM(play_player.passing_cmp_air_yds) AS passing_cmp_air_yds, 
  SUM(play_player.passing_incmp) AS passing_incmp, 
  SUM(play_player.passing_incmp_air_yds) AS passing_incmp_air_yds, 
  SUM(play_player.passing_int) AS passing_int,
  SUM(play_player.passing_sk) AS passing_sk,
  SUM(play_player.passing_sk_yds) AS passing_sk_yds, 
  SUM(play_player.passing_tds) AS passing_tds,
  SUM(play_player.passing_twopta) AS passing_twopta,
  SUM(play_player.passing_twoptm) AS passing_twoptm, 
  SUM(play_player.passing_twoptmissed) AS passing_twoptmissed, 
  SUM(play_player.passing_yds) AS passing_yds,
  SUM(play_player.punting_blk) AS punting_blk,
  SUM(play_player.punting_i20) AS punting_i20, 
  SUM(play_player.punting_tot) AS punting_tot,
  SUM(play_player.punting_touchback) AS punting_touchback,
  SUM(play_player.punting_yds) AS punting_yds, 
  SUM(play_player.puntret_downed) AS puntret_downed, 
  SUM(play_player.puntret_fair) AS puntret_fair,
  SUM(play_player.puntret_oob) AS puntret_oob,
  SUM(play_player.puntret_tds) AS puntret_tds,
  SUM(play_player.puntret_tot) AS puntret_tot,
  SUM(play_player.puntret_touchback) AS puntret_touchback,
  SUM(play_player.puntret_yds) AS puntret_yds, 
  SUM(play_player.receiving_rec) AS receiving_rec,
  SUM(play_player.receiving_tar) AS receiving_tar,
  SUM(play_player.receiving_tds) AS receiving_tds, 
  SUM(play_player.receiving_twopta) AS receiving_twopta, 
  SUM(play_player.receiving_twoptm) AS receiving_twoptm, 
  SUM(play_player.receiving_twoptmissed) AS receiving_twoptmissed, 
  SUM(play_player.receiving_yac_yds) AS receiving_yac_yds, 
  SUM(play_player.receiving_yds) AS receiving_yds,
  SUM(play_player.rushing_att) AS rushing_att,
  SUM(play_player.rushing_loss) AS rushing_loss, 
  SUM(play_player.rushing_loss_yds) AS rushing_loss_yds, 
  SUM(play_player.rushing_tds) AS rushing_tds,
  SUM(play_player.rushing_twopta) AS rushing_twopta,
  SUM(play_player.rushing_twoptm) AS rushing_twoptm, 
  SUM(play_player.rushing_twoptmissed) AS rushing_twoptmissed, 
  SUM(play_player.rushing_yds) AS rushing_yds,
  SUM(play_player.passing_yds + play_player.rushing_yds
      + play_player.receiving_yds + play_player.fumbles_rec_yds) AS offense_yds,
  SUM(play_player.passing_tds + play_player.receiving_tds
      + play_player.rushing_tds + play_player.fumbles_rec_tds) AS offense_tds,
  SUM(play_player.defense_frec_tds + play_player.defense_int_tds
      + play_player.defense_misc_tds) AS defense_tds
FROM play_player
LEFT JOIN game ON play_player.gsis_id = game.gsis_id
WHERE game.season_type = 'Regular' AND game.season_year = 2012
GROUP BY play_player.player_id
ORDER BY passing_yds DESC LIMIT 5
```

Holy crap! I bet you were not expecting that. It should perhaps be clear why 
the query might be slow: it is aggregating columns for **every statistical 
category** for each player statistic in the 2012 season. Why? This is the price 
that we must pay for generality: the query interface doesn't know which 
statistics you're going to inspect, so it must give you all of them.

But we can do better by writing a much simpler SQL query and adapting it to 
nfldb's types like we did in the last section:

```python
query = '''
SELECT
  play_player.player_id, SUM(play_player.passing_yds) AS passing_yds
FROM play_player
LEFT JOIN game ON play_player.gsis_id = game.gsis_id
WHERE game.season_type = 'Regular' AND game.season_year = 2012
GROUP BY play_player.player_id
ORDER BY passing_yds DESC LIMIT 5
'''
players = []
with nfldb.Tx(db) as cursor:
    cursor.execute(query)
    for row in cursor.fetchall():
        pp = nfldb.PlayPlayer(db, None, None, None, row['player_id'],
                              None, row)
        players.append(pp)

for pp in players:
    print pp.player_id, pp.passing_yds
```

And this provides exactly the same output as above, except it runs in `0.2` 
seconds on my machine (versus `1.3` seconds when using the query interface). 
The fact that we're only aggregating `passing_yds` saves a ton of time wasted 
on aggregating a whole bunch of other categories that are never used.

You can go even faster than this by restricting the rows to a particular player 
when you're aggregating statistics for a pre-determined set of players. For 
example, add the following to the `WHERE` clause of the SQL query in the 
last code sample:

```sql
play_player.player_id IN ('00-0020531', '00-0026498', '00-0021678', '00-0019596', '00-0026143')
```

Then the run time drops down to about `0.133` seconds on my machine. But of 
course, this particular optimization also makes using the query interface more 
performant:

```python
thebest = ['00-0020531', '00-0026498', '00-0021678', '00-0019596', '00-0026143']

q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.player(player_id=thebest)
for pp in q.sort('passing_yds').limit(5).as_aggregate():
    print pp.player_id, pp.passing_yds
```

Which runs in about `0.16` seconds on my machine. So it's not quite as fast as 
hand-written SQL, but it's much faster than the `1.3` seconds that we started 
with.

The moral of the story is to experiment and see what works best for you. But 
remember, in programming, premature optimization is the root of all evil.

