One of the goals of nfldb is for it to be simple to use. And for it to be 
simple to use, the data model should also be simple.

There are only 8 tables in the database. Here is a brief description of each:

* **meta** stores information about the database or about the state of the 
  world. For example, it keeps track of the version of the database and the
  current week of the current NFL season.
* **team** stores a row for each team in the league. There is also a row that 
  corresponds to an unknown team called `UNK`. This is used for players that 
  are not on any current roster.
* **player** stores ephemeral data about players. Namely, it is the most 
  current information about each player known by nfldb. The data is nearly a 
  total copy of the data in nflgame's JSON player database.
* **game** stores a row for each NFL game in the preseason, regular season and 
  postseason dating back to 2009. This includes games that are scheduled in the 
  future but have not been played.
* **drive** stores a row for each drive in a single game.
* **play** stores a row for each play in a single drive.
* **agg_play** stores a row for each play aggregating statistics from the
  corresponding rows in the `play_player` table.
* **play_player** stores a row for each player statistic in a single play.

You can get an overview of the entire database and the relationships between 
each table with the
[Entity-Relationship (ER) 
diagrams](The-data-model#er-diagrams)
section of this page.

The rest of this page describes the data stored in the database with SQL
examples. An explicit effort was made **not** to talk about the nfldb Python 
module.


### What kind of player meta data is stored?

The data in the `player` table corresponds to information scraped off of roster 
and player profile pages on NFL.com. (In fact, this is the only data in nfldb 
that is scraped.) NFL.com pages are used so that players can be matched with 
their statistical data via unique identifiers, rather than having to rely on a 
fuzzy name matching algorithm. (If you're interested in how the scraping is
done, please see the
[nflgame-update-players](https://github.com/BurntSushi/nflgame/blob/master/scripts/nflgame-update-players)
script in the nflgame repository.)

This data includes players who are no longer playing. In this case, their team 
is `UNK`. This leads to a nice property of the data in the `player` table: 
**any player with a team not equal to `UNK` is currently on that team's 
roster**. Therefore, the roster of a team (sorted by player status, position
and name) as known by nfldb can easily be accessed with the following query:

```sql
SELECT full_name, position, status
FROM player
WHERE team = 'NE'
ORDER BY status ASC, position ASC, full_name ASC
```

Similarly, you could get every player currently on a roster by using
`team != 'UNK'`.

Whether the data in the `player` table is current or not depends on how quickly 
NFL.com updates their data and whether you're [updating your 
database](Updating-nfldb-with-real-time-data)
frequently. In my experience, NFL.com's data can be slow to update during the 
offseason, but is relatively quick during the season.

Finally, it is important to note that most of the columns in the `player` table 
can be `NULL`. This means that not all data is available for all players. (We 
are at the mercy of the consistency of NFL.com's roster and player profile 
pages.) In my experience, the data is usually very complete for active players.


### What is the `play_player` table?

The `play_player` table is arguably the most important table in the entire 
database. Namely, it is **a single point of truth for player statistics**. Each 
row in this table corresponds to statistics recorded by **a single player in a 
single play**. (Aggregated player statistics for each play are stored in the
`agg_play` table. `agg_play` is a
[materialized view](http://en.wikipedia.org/wiki/Materialized_view).)

Let's look at a fairly complex play that occurred in the Eagles/Redskins game 
during the first week of the 2013 regular season in the 4th quarter with 13:55
remaining on the clock:

    M.Vick pass short right to J.Avant to PHI 33 for 6 yards (J.Wilson).
    FUMBLES (J.Wilson), RECOVERED by WAS-P.Riley at PHI 35.
    P.Riley to PHI 29 for 6 yards (J.Avant).

This particular play has a `gsis_id` of `2013090900`, a `drive_id` of `21` and 
a `play_id` of `3717`. We can then use that information to see the statistics 
recorded by each player in that play. (Note that I've restricted the `SELECT` 
fields in the query below to make the output readable here. You may want to try 
`SELECT * ...`.)

```sql
SELECT
    full_name, passing_yds, receiving_rec, receiving_yds,
    fumbles_forced, defense_ffum, defense_frec_yds
FROM play_player
LEFT JOIN player ON player.player_id = play_player.player_id
WHERE (gsis_id, drive_id, play_id) = ('2013090900', 21, 3717)
```

And the output of that query is:

```
  full_name   | passing_yds | receiving_rec | receiving_yds | fumbles_forced | defense_ffum | defense_frec_yds 
--------------+-------------+---------------+---------------+----------------+--------------+------------------
 Michael Vick |           6 |             0 |             0 |              0 |            0 |                0
 Jason Avant  |           0 |             1 |             6 |              1 |            0 |                0
 Josh Wilson  |           0 |             0 |             0 |              0 |            1 |                0
 Perry Riley  |           0 |             0 |             0 |              0 |            0 |                6
```

The statistics record that Michael Vick threw a pass for 6 yards, Jason
Avant caught a pass for 6 yards and had the ball stripped by Josh
Wilson, which was recovered by Perry Riley and returned for 6 yards.


### Aggregate statistics

To keep things simple and to avoid duplication of data, nfldb does not store 
aggregate statistics beyond the `play` level. Instead, they must be computed on 
the fly.  Thankfully, there is a fast way to do it with PostgreSQL's aggregate 
functions. For example, we could find the top quarterbacks in the 2012 regular 
season with more than 4500 passing yards:

```sql
SELECT player.full_name, SUM(play_player.passing_yds) AS passing_yds
FROM play_player
LEFT JOIN player ON player.player_id = play_player.player_id
LEFT JOIN game ON game.gsis_id = play_player.gsis_id
WHERE game.season_year = 2012 AND game.season_type = 'Regular'
GROUP BY player.full_name
HAVING SUM(play_player.passing_yds) >= 4500
ORDER BY passing_yds DESC
```

And the output is:

```
    full_name     | passing_yds 
------------------+-------------
 Drew Brees       |        5177
 Matthew Stafford |        4965
 Tony Romo        |        4903
 Tom Brady        |        4799
 Matt Ryan        |        4719
 Peyton Manning   |        4667
```

When writing aggregate queries, the `WHERE` clause specifies *what to 
aggregate* while the `HAVING` clause specifies *which aggregate results to 
return*. Namely, `WHERE` is used before aggregation while `HAVING` is used 
after aggregation.


### The `agg_play` table

It is a [materialized view](http://en.wikipedia.org/wiki/Materialized_view). A 
materialized view is like a normal view (which is just a saved `SELECT` query), 
except it actually stores the data.

Why does `nfldb` aggregate player statistics for each play? For the most part, 
it is a decision guided by the performance of searching statistics. If we 
didn't aggregate any data, then filtering plays based on statistics---like 
passing yards---requires joining with the `play_player` table and summing all 
the joining rows with `SUM`. On large data sets, this is a very expensive 
operation. If we aggregate statistics, no joining or summing is necessary when 
filtering. The costs are pretty meager by comparison: the database is a little 
bigger and it takes a little longer to insert data.

Since `agg_play` is a materialized view, it requires no maintenance. It is 
automatically updated whenver new data is added, modified or deleted. *It just 
works*.

Will other materialized views with aggregated data be added? I'm not sure. The 
`play` table really benefits because filtering statistics by plays is so 
common.

Finally, the `agg_play` table is completely hidden from the public interface. 
Even though `play` data is actually stored across two tables, users of `nfldb` 
never need to know this.


## ER diagrams

Entity-Relationship (ER) diagrams are used to graphically represent the schema 
of a database. They show each entity, its attributes and the relationships 
between each entity. In the ER diagrams for nfldb, entities correspond to 
tables and attributes correspond to columns in a table.

An example of a relationship would be one-to-many between games and drives. 
Namely, for each game, there can be *zero or more* drives associated with that 
game and for each drive, there must be *exactly one* game associated with that 
drive.

Note that the ER diagrams do not contain derived fields. You can see 
documentation for each statistical category (including derived fields) on the
[statistical 
categories](Statistical-categories) 
page.

There are two ER diagrams. The first is a condensed version that omits many of 
the statistical categories for plays and players. (Click on the image to get a 
full PDF of the ER diagram.)

[![Shortened ER diagram for nfldb](http://burntsushi.net/stuff/nfldb/nfldb-condensed.png)](http://burntsushi.net/stuff/nfldb/nfldb-condensed.pdf)

The second is a full ER diagram, with all of the statistical categories:

[![Full ER diagram for nfldb](http://burntsushi.net/stuff/nfldb/nfldb.png)](http://burntsushi.net/stuff/nfldb/nfldb.pdf)

If you're curious, the above ER diagrams are automatically generated with the
[nfldb-write-erd](../blob/master/scripts/nfldb-write-erd)
script using a utility I wrote called [erd](https://github.com/BurntSushi/erd).

