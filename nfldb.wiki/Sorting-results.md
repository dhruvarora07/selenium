No searching interface is complete without the ability to sort and limit the 
results you get. While this can of course be done in Python, it can be 
significantly faster to do it in the database for two reasons. Firstly, the 
database is going to be much faster than Python at sorting data. Secondly,
limiting results can potentially prevent many unnecessary objects from being 
created.

Let's look at an example just in case you don't believe me. Let's consider 
finding the top 5 passing plays in the 2012 season. Using what you already know 
about the Query interface, this is easy:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
plays = sorted(q.as_plays(), key=lambda p: p.passing_yds, reverse=True)
for play in plays[0:5]:
    print play
```

And the output is:

```bash
[andrew@Liger nflgame] time python2 scratch.py
(TB, OWN 4, Q3, 2 and 10) (6:39) J.Freeman pass deep left to V.Jackson to NO 1 for 95 yards (M.Jenkins).
(WAS, OWN 12, Q1, 1 and 10) (3:41) R.Griffin pass deep left to P.Garcon for 88 yards, TOUCHDOWN [M.Jenkins]. Pass 16, YAC 72
(DAL, OWN 15, Q3, 1 and 10) (1:38) (Shotgun) T.Romo pass deep right to D.Bryant for 85 yards, TOUCHDOWN. Pass complete after Romo forced out of the pocked and threw on the run; Bryant caught the pass on a crossing pattern.
(NE, OWN 17, Q2, 1 and 10) (9:56) (Shotgun) T.Brady pass short left to S.Vereen for 83 yards, TOUCHDOWN.
(CAR, OWN 9, Q4, 1 and 10) (14:17) (Shotgun) C.Newton pass deep left to A.Edwards pushed ob at WAS 9 for 82 yards (C.Griffin).
```

On my system, the above took **7.186** seconds to complete. Now let's try 
sorting and limiting with the Query interface by using the
[sort](http://pdoc.burntsushi.net/nfldb#nfldb.Query.sort)
and
[limit](http://pdoc.burntsushi.net/nfldb#nfldb.Query.limit)
methods:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
for play in q.sort('passing_yds').limit(5).as_plays():
    print play
```

The output should be exactly the same as above. And on my system, it only
takes **0.183** seconds, which is **39 times faster** than fetching all of the 
plays and sorting them with Python.


### Sorting explained

Now that the introductory example is out of the way, we can look at sorting in 
more depth. In particular, the `limit` method is fairly uninteresting: it takes 
an integer, and restricts the number of results to be at most the integer 
given. If the integer is `0`, then no limiting happens. Specifying a limit can 
have a dramatic impact on performance if its absence results in a lot of Python 
objects being created. (And this is exactly how the first example on this page 
achieved such a speed up.)

The `sort` method however has more options. In the above example, we sorted by 
the `passing_yds` field by simply specifying a string. We can also specify an 
order, which must be `asc` for ascending (smallest to biggest) or `desc` for 
descending (biggest to smallest). Descending is the default ordering.

For example, let's adapt the last example so that it finds the five *smallest* 
passing plays:

```python
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
for play in q.sort(('passing_yds', 'asc')).limit(5).as_plays():
    print play
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py                                                 
(MIN, OPP 25, Q1, 1 and 10) (7:38) C.Ponder pass short right to C.Ponder to DET 40 for -15 yards (C.Avril).
(WAS, OWN 17, Q4, 1 and 10) (:17) (Shotgun) R.Griffin pass short right to B.Banks to WAS 8 for -9 yards (L.Kuechly).
(BUF, OWN 20, Q1, 3 and 10) (9:18) (Shotgun) R.Fitzpatrick pass short right to C.Spiller to BUF 11 for -9 yards (K.Wright). Screen pass, caught at BUF 11.
(SD, OPP 17, Q2, 3 and 5) (:14) (Shotgun) P.Rivers pass short right to J.Clary to CLE 25 for -8 yards (J.Parker). Pass Deflected by #97 Sheard and caught by #66 Clary
(TB, OPP 41, Q4, 3 and 3) (5:33) (Shotgun) J.Freeman pass short right to D.Ware to WAS 48 for -7 yards (R.Kerrigan).
```

We achieved this by calling `sort(('passing_yds', 'asc'))`. Namely, we passed a 
tuple where the first element is a string and the second element is the order.

It doesn't stop there, though. We can also sort by multiple fields at the same 
time. For example, if we wanted to see all defensive touchdowns in one week of 
football, we might want to sort them by game and then by the time the play 
occurred in the game. We can do this by providing a *list* of fields to sort 
by:

```python
q = nfldb.Query(db)
q.game(season_year=2013, season_type='Regular', week=1)
q.play(defense_tds=1)
q.sort([('gsis_id', 'asc'), ('time', 'asc')])
for play in q.as_plays():
    print play
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
(NE, OPP 24, Q2, 1 and 5) (8:40) S.Ridley up the middle to BUF 25 for -1 yards (K.Alonso). FUMBLES (K.Alonso), RECOVERED by BUF-D.Searcy at BUF 26. D.Searcy for 74 yards, TOUCHDOWN. The Replay Assistant challenged the fumble ruling, and the play was Upheld.
(JAC, OWN 12, Q4, 3 and 10) (12:51) (Shotgun) B.Gabbert pass short right intended for J.Forsett INTERCEPTED by T.Hali at JAC 10. T.Hali for 10 yards, TOUCHDOWN.
(STL, OWN 7, Q3, 2 and 10) (10:42) S.Bradford pass short right intended for L.Kendricks INTERCEPTED by D.Williams (M.Shaughnessy) at STL 2. D.Williams for 2 yards, TOUCHDOWN.
(NYG, OWN 24, Q3, 2 and 15) (12:42) (Shotgun) D.Wilson up the middle to NYG 27 for 3 yards (N.Hayden). FUMBLES (N.Hayden), RECOVERED by DAL-B.Church at NYG 27. B.Church for 27 yards, TOUCHDOWN. Church returned fumble along right side of the field for the touchdown.
(NYG, OWN 48, Q4, 1 and 10) (2:00) (Shotgun) E.Manning pass short right intended for D.Scott INTERCEPTED by B.Carr at NYG 49. B.Carr for 49 yards, TOUCHDOWN.
(PHI, OPP 4, Q1, 1 and 4) (12:14) (No Huddle, Shotgun) M.Vick to WAS 10 for -6 yards (R.Kerrigan). FUMBLES (R.Kerrigan), touched at WAS 10, RECOVERED by WAS-D.Hall at WAS 25. D.Hall for 75 yards, TOUCHDOWN. Lateral batted by 91 - Kerrigan The Replay Assistant challenged the backward pass ruling, and the play was Upheld.
(SD, OWN 13, Q4, 1 and 10) (9:38) (Shotgun) P.Rivers pass short left intended for D.Woodhead INTERCEPTED by B.Cushing at SD 18. B.Cushing for 18 yards, TOUCHDOWN. PENALTY on SD, Unnecessary Roughness, 15 yards, enforced between downs. The Replay Assistant challenged the incomplete pass ruling, and the play was Upheld.
```


### Restrictions

At this moment in time, the `sort` method can only be used to sort fields on 
the results given. For example, if you're retrieving plays, then you can't sort 
them by the
[start_time](http://pdoc.burntsushi.net/nfldb#nfldb.Game.start_time)
field, which is an attribute of a game. It is possible this restriction will be 
lifted in the future.

