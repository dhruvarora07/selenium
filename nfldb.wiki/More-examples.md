The goal of this page is to provide a cookbook style list of examples. It is 
intended that examples illustrate more complex analysis of data than is covered 
in the other tutorial articles.

### Contents

* [Filtering plays based on where they end](More-examples#filtering-plays-based-on-where-they-end)


### Filtering plays based on where they end

One of the notable things absent from the data in nfldb is the *ending field 
position* of a play. As of now, this data doesn't exist because it doesn't 
exist in the source data. While it's possible that it will one day be added 
automatically by nfldb by computing it, we'll have to work around it on a case 
by case basis for the time being.

A nice way to illustrate this is to inspect the obscene number of plays in the 
2012 season where Calvin Johnson was tackled inside the 5 yard line.

```python
# Specify all of CJ's non-scoring receptions in the 2012 regular season.
q = nfldb.Query(db)
q.game(season_year=2012, season_type='Regular')
q.player(full_name="Calvin Johnson")
q.play(receiving_rec=1, receiving_tds=0)

def inside(field, stat):
    """
    inside takes a field position and a statistical category, and
    returns a predicate. The predicate takes a play and returns true if
    and only if that play ends inside the field position determined by
    adding the statistical category to the start of the play. (e.g.,
    play yardline + receiving yards determines the field position where
    CJ was tackled.)
    """
    cutoff = nfldb.FieldPosition.from_str(field)
    return lambda play: play.yardline + getattr(play, stat) >= cutoff
plays = filter(inside('OPP 5', 'receiving_yds'), q.as_plays())

for p in plays:
  print p
```

And the output is:

```bash
[andrew@Liger nflgame] python2 scratch.py
(DET, OPP 11, Q2, 2 and 8) (10:56) (Shotgun) M.Stafford pass short middle to C.Johnson to STL 1 for 10 yards (C.Dahl).
(DET, OPP 23, Q4, 2 and 7) (:30) (Shotgun) M.Stafford pass deep middle to C.Johnson to STL 5 for 18 yards (Q.Mikell).
(DET, OPP 10, Q3, 1 and 10) (7:10) (No Huddle, Shotgun) M.Stafford pass short right to C.Johnson to TEN 1 for 9 yards (J.McCourty; A.Ayers).
(DET, OPP 21, Q4, 2 and 20) (10:56) (Shotgun) M.Stafford pass short left to C.Johnson to PHI 1 for 20 yards (K.Coleman).
(DET, OPP 8, Q3, 3 and 2) (6:16) (Shotgun) M.Stafford pass short right to C.Johnson to CHI 2 for 6 yards (C.Tillman).
(DET, OPP 39, Q2, 3 and 8) (8:36) (Shotgun) M.Stafford pass deep left to C.Johnson to JAC 1 for 38 yards (C.Prosinski). Penalty on JAC-P.Posluszny, Defensive Offside, declined.
(DET, OPP 28, Q4, 1 and 10) (11:59) (Shotgun) M.Stafford pass deep middle to C.Johnson to MIN 3 for 25 yards (Team). PENALTY on MIN-J.Brinkley, Unnecessary Roughness, 2 yards, enforced at MIN 3.
(DET, OPP 12, Q2, 1 and 10) (13:37) (Shotgun) M.Stafford pass short left to C.Johnson to ARI 1 for 11 yards (R.Johnson).
```

Note that the `inside` function written above is higher-order. You should be 
able to copy it and use it as-in in your programs.

