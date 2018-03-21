Searching NFL data isn't just about finding groups of plays, drives or games. 
Sometimes you want to gather aggregate statistics over a period of time. 
Whether that's only a couple plays, drives, games or seasons is up to you.

The recommended way to search and retrieve aggregate statistics is by using the
[aggregate](http://pdoc.burntsushi.net/nfldb#nfldb.Query.aggregate)
method to specify aggregate search criteria, and the
[as_aggregate](http://pdoc.burntsushi.net/nfldb#nfldb.Query.as_aggregate)
method to return aggregated statistics as
[PlayPlayer](http://pdoc.burntsushi.net/nfldb#nfldb.PlayPlayer)
objects.

In this article, "aggregating statistics" essentially means "adding 
player statistics". Namely, given two plays where Tom Brady threw for 5 and 16 
yards, the aggregation of that would be a single statistic of 21 passing yards 
(along with any other statistics accrued by Tom Brady including passing 
attempts, completions, interceptions, sacks, etc.).

The key to aggregate searching is that the regular `game`, `drive`, `play` and 
`player` methods specify *what to aggregate*, while the `aggregate` method 
specifies *which aggregate results to return*. Namely, the former methods 
specify search criteria before aggregation, while the `aggregate` method 
specifies search criteria after aggregation.

For this reason, whenever the `aggregate` method is used, the `as_aggregate` 
method is the **only way to retrieve results**. If you use any of the other 
`as_*` methods, an assertion error will be triggered.

Enough blabbering, let's take a look at an example. Say we want to see who 
throws for the most yards on third down. We can use what we learned in the
[sorting](Sorting-results)
article to limit the number of results:

```python
import nfldb
db = nfldb.connect()
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.play(third_down_att=1)
for pp in q.sort('passing_yds').limit(5).as_aggregate():
    print pp.player, pp.passing_yds
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
Drew Brees (NO, QB) 1452
Matthew Stafford (DET, QB) 1364
Aaron Rodgers (GB, QB) 1306
Andrew Luck (IND, QB) 1295
Peyton Manning (DEN, QB) 1236
```

Notice that we never actually used the `aggregate` method, and instead only 
called the `as_aggregate` method. This means that we didn't apply any search 
criteria on the aggregate results, but instead, only on what was aggregated.
For grins, let's restrict the results to QBs with more than 1300 yards passing
on third down:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.play(third_down_att=1)
q.aggregate(passing_yds__ge=1300)
for pp in q.sort('passing_yds').limit(5).as_aggregate():
    print pp.player, pp.passing_yds
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
Drew Brees (NO, QB) 1452
Matthew Stafford (DET, QB) 1364
Aaron Rodgers (GB, QB) 1306
```


### Aggregation in Python code

While I think the above examples are fairly cool, the query interface is 
inherently limited in its ability to express complex aggregation queries. The 
reason is to keep things simple and fast for the common use cases. It's simple 
because it only handles one large group of data to aggregate, and it's fast 
because the actual aggregation is done in PostgreSQL. (This is similar to how 
sorting with the query interface is typically faster than sorting in Python.)

nfldb also provides a way to aggregate statistics in Python code. Let's 
start by replicating the above example:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular')
q.play(third_down_att=1)
plays = q.as_plays()

aggregated = nfldb.aggregate(plays)  # Returns a list of PlayPlayer objects.
aggregated = sorted(aggregated, key=lambda p: p.passing_yds, reverse=True)
for pp in aggregated[0:5]:
    print pp.player, pp.passing_yds
```

There is quite a bit more code in this example, and it also runs more slowly on 
my machine. (It takes `5.48` seconds versus `0.3` seconds for the example 
above.) However, it's slow partially because it has to aggregate *every third 
down* play in the 2012 regular season. We can make it faster by throwing out
plays we don't need (like plays with 0 passing yards):

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.play(third_down_att=1, passing_yds__ne=0)
plays = q.as_plays()

aggregated = nfldb.aggregate(plays)  # Returns a list of PlayPlayer objects.
aggregated = sorted(aggregated, key=lambda p: p.passing_yds, reverse=True)
for pp in aggregated[0:5]:
    print pp.player, pp.passing_yds
```

Which runs in about `4` seconds on my machine.


### Fancier aggregation

Now let's consider a type of aggregation that requires multiple queries.

Let's say we want to track how a particular player does on a drive-by-drive 
basis in a particular game. For example, how does Tom Brady's passing 
statistics break down on a drive-by-drive basis in the 2013 season opener?

We can start by fetching the offensive drives for the Patriots in the 2013 
season opener:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular', week=1, team='NE')
drives = q.drive(pos_team='NE').sort(('start_time', 'asc')).as_drives()
for d in drives:
    print d
```

Which outputs:

```bash
[andrew@Liger nflgame] python2 scratch.py
[Punt        ] NE  from OWN 14 to OPP 44 (lasted 02:55 - Q1 15:00 to Q1 12:05)
[Touchdown   ] NE  from OPP 16 to OPP 9  (lasted 00:47 - Q1 11:33 to Q1 10:46)
[Field Goal  ] NE  from MIDFIELD to OPP 30 (lasted 03:43 - Q1 08:30 to Q1 04:47)
[Punt        ] NE  from OWN 25 to OPP 47 (lasted 03:35 - Q1 02:26 to Q2 13:51)
[Punt        ] NE  from OWN 20 to OWN 27 (lasted 01:00 - Q2 11:42 to Q2 10:42)
[Fumble      ] NE  from OWN 46 to OPP 24 (lasted 01:28 - Q2 09:53 to Q2 08:25)
[Punt        ] NE  from OWN 27 to OPP 37 (lasted 03:20 - Q2 08:25 to Q2 05:05)
[Touchdown   ] NE  from OPP 32 to OPP 8  (lasted 01:50 - Q2 03:45 to Q2 01:55)
[Interception] NE  from OWN 25 to OWN 31 (lasted 00:08 - Q2 01:11 to Q2 01:03)
[End of Half ] NE  from OWN 20 to OWN 20 (lasted 00:34 - Q2 00:34 to Q2 00:00)
[Fumble      ] NE  from OWN 20 to OPP 1  (lasted 06:51 - Q3 11:03 to Q3 04:12)
[Punt        ] NE  from OWN 3  to OWN 7  (lasted 00:47 - Q3 01:15 to Q3 00:28)
[Field Goal  ] NE  from OWN 15 to OPP 15 (lasted 03:29 - Q4 14:17 to Q4 10:48)
[Punt        ] NE  from OWN 14 to OWN 43 (lasted 02:50 - Q4 08:41 to Q4 05:51)
[Field Goal  ] NE  from OWN 34 to OPP 17 (lasted 04:26 - Q4 04:31 to Q4 00:05)
```

Now we need to fetch the player statistics for each drive, and aggregate them. 
Note that it might feel natural to do this:

```python
stats = nfldb.aggregate(drives)
```

But that would provide statistics for *all* of the drives. We want statistics 
for all of the plays in *each* drive. So we execute a new query for each drive 
and print a nice message containing the statistics we care about:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular', week=1, team='NE')
drives = q.drive(pos_team='NE').sort(('start_time', 'asc')).as_drives()

tpl = 'On drive {drive_num}, Tom Brady went {cmp}/{att} for {yds} yard(s) ' \
      'with {tds} TD(s), {int} INT(s) and was sacked {sack} time(s).'
for i, d in enumerate(drives):
    q = nfldb.Query(db).drive(gsis_id=d.gsis_id, drive_id=d.drive_id)
    tfb = q.player(full_name='Tom Brady').as_aggregate()[0]

    msg = tpl.format(drive_num=i+1, cmp=tfb.passing_cmp, att=tfb.passing_att,
                     yds=tfb.passing_yds, tds=tfb.passing_tds,
                     int=tfb.passing_int, sack=tfb.passing_sk)
    print msg
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
On drive 1, Tom Brady went 2/5 for 34 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 2, Tom Brady went 1/2 for 9 yard(s) with 1 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 3, Tom Brady went 4/6 for 20 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 4, Tom Brady went 1/2 for 6 yard(s) with 0 TD(s), 0 INT(s) and was sacked 1 time(s).
On drive 5, Tom Brady went 0/2 for 0 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 6, Tom Brady went 1/2 for 14 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 7, Tom Brady went 2/3 for 25 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 8, Tom Brady went 1/1 for 8 yard(s) with 1 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 9, Tom Brady went 1/2 for 6 yard(s) with 0 TD(s), 1 INT(s) and was sacked 0 time(s).
On drive 10, Tom Brady went 0/0 for 0 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 11, Tom Brady went 4/7 for 48 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 12, Tom Brady went 1/3 for 5 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 13, Tom Brady went 2/5 for 51 yard(s) with 0 TD(s), 0 INT(s) and was sacked 1 time(s).
On drive 14, Tom Brady went 2/5 for 26 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 15, Tom Brady went 7/7 for 36 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
```

We can of course also specify criteria on aggregate statistics. For example, we 
might only want to look at drives where Tom Brady threw for at least 30 yards 
combined:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular', week=1, team='NE')
drives = q.drive(pos_team='NE').sort(('start_time', 'asc')).as_drives()

tpl = 'On drive {drive_num}, Tom Brady went {cmp}/{att} for {yds} yard(s) ' \
      'with {tds} TD(s), {int} INT(s) and was sacked {sack} time(s).'
for i, d in enumerate(drives):
    q = nfldb.Query(db).drive(gsis_id=d.gsis_id, drive_id=d.drive_id)
    q.player(full_name='Tom Brady')
    results = q.aggregate(passing_yds__ge=30).as_aggregate()
    if len(results) == 0:
        continue

    tfb = results[0]
    msg = tpl.format(drive_num=i+1, cmp=tfb.passing_cmp, att=tfb.passing_att,
                     yds=tfb.passing_yds, tds=tfb.passing_tds,
                     int=tfb.passing_int, sack=tfb.passing_sk)
    print msg
```

Which gives us this output instead:

```bash
[andrew@Liger nflgame] python2 scratch.py
On drive 1, Tom Brady went 2/5 for 34 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 11, Tom Brady went 4/7 for 48 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
On drive 13, Tom Brady went 2/5 for 51 yard(s) with 0 TD(s), 0 INT(s) and was sacked 1 time(s).
On drive 15, Tom Brady went 7/7 for 36 yard(s) with 0 TD(s), 0 INT(s) and was sacked 0 time(s).
```

Note that we need the if statement `if len(results) == 0` since Tom Brady does 
not throw more than 30 yards in every drive.

