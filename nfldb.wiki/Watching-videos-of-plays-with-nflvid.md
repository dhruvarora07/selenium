With nfldb and [nflvid](https://github.com/BurntSushi/nflvid), it is possible 
to search for a set of plays, open them as a playlist in a video player and 
watch the actual game footage of each play. Game footage can be all-22 coach 
video or the regular broadcast footage.

Here is a small working example that plays each of the 7 plays where Adrian 
Peterson rushed for more than 50 yards in the 2012 season:

```python
import nfldb
import nflvid.vlc

db = nfldb.connect()
q = nfldb.Query(db)

q.game(season_year=2012, season_type='Regular')
q.player(full_name='Adrian Peterson').play(rushing_yds__ge=50)

nflvid.vlc.watch(db, q.as_plays(), '/m/nfl/coach/pbp')
```

This simple example demonstrates a lot of power; the entire query interface of 
nfldb can be composed with a directory of play-by-play footage to watch any 
subset of plays imaginable.


### What is nflvid?

nflvid faciliates the downloading and slicing of NFL game footage from 
**publicly available sources**. In particular, everything that nflvid downloads 
is accessible to the public on NFL's content delivery network powered by 
Neulion. Note that nflvid **does not come with any footage** as I will not 
redistribute any video copyrighted by the NFL.

This means that nflvid's future is uncertain, since the public accessibility 
of the video could go away. When that happens, and if there are no other public 
sources, nflvid will be considered a dead project.

In addition to downloading footage, nflvid also downloads meta data (as XML) 
describing the start times of each play in broadcast and coach video. This data 
is then used to slice full game video into its component plays. This approach 
tends to consistently work, although sometimes the start timings in the meta 
data are off and can lead to incorrect results that are impossible to detect 
automatically. (Without video analysis.)


### Installation

To get the examples in this wiki to work, you must have
[nfldb installed](Installation). You must also
[install nflvid](https://github.com/BurntSushi/nflvid/blob/master/README.md#installation),
which is on PyPI. If you're on Windows, there are
[special instructions for you on the nflvid wiki](https://github.com/BurntSushi/nflvid/wiki/Windows-Installation).

Installing nflvid can be tricky because it requires run-time dependencies like 
`ffmpeg` and `rtmpdump`. Moreover, downloading huge video files can sometimes 
fail for unknown reasons. Note that if you want to download coach footage, you 
need `rtmpdump` **with KSV's patch**. Otherwise, coach footage might download 
successfully but you will be unable to play it. (If someone more knowledgeable 
than me on video playback would like to add something about why this is, that 
would be great.)
[I maintain a fork of rtmpdump with KSV's patch](https://github.com/BurntSushi/rtmpdump-ksv)
that I try to keep up to date with upstream changes from
`git://git.ffmpeg.org/rtmpdump`.

Since `rtmpdump` can be tricky to get working, we can verify that nflvid is 
installed by downloading the first 30 seconds of some broadcast footage:

```bash
mkdir -p /home/you/nfl/{broadcast,coach}/{full,pbp}
nflvid-footage --broadcast --dry-run --season 2012 --teams NE --weeks 1 -- /home/you/nfl/broadcast/full
```

And now you should be able to watch the first 30 seconds:

```bash
vlc /home/you/nfl/broadcast/full/2012090904.mp4
```

If that works, then try downloading the full game (remove the 30 second clip
from the previous step first). If you're impatient and want the download to go 
more quickly, then you can lower the quality to `800` (which is probably just 
barely watchable). The maximum quality is `4500`, which corresponds to 720p HD.

```bash
rm /home/you/nfl/broadcast/full/2012090904.mp4
nflvid-footage --broadcast --quality 800 --season 2012 --teams NE --weeks 1 -- /home/you/nfl/broadcast/full
```

Once the download completes, you can slice the video into play-by-play video 
files. Namely, there will be a single video file for each play in the game:

```bash
nflvid-slice --broadcast /home/you/nfl/broadcast/pbp /home/you/nfl/broadcast/full/2012090904.mp4
```

You should now be able to watch any of the plays individually:

```bash
vlc /home/you/nfl/broadcast/pbp/2012090904/3521.mp4
```

If you want to try your luck with coach video, then repeat the above steps 
except remove all occurrences of `--broadcast` and change your paths from
`/home/you/nfl/broadcast/...` to `/home/you/nfl/coach/...`. (Note that you can 
choose any path you like, but it's good to adhere to a sensible convention.)

You may find the `nflvid-incomplete` program useful. For example, I tried 
downloading the Sunday week 2 games in the 2013 regular season overnight. When 
I woke up, I ran this command:

```bash
[andrew@Liger nflvid] nflvid-incomplete --broadcast /m/nfl/broadcast/tmp/*.mp4
```

And the output was:

```bash
/m/nfl/broadcast/tmp/2013091505.mp4: Expected duration 02:49:10:14 but it has 00:59:40:030.
/m/nfl/broadcast/tmp/2013091506.mp4: Expected duration 02:22:58:98 but it has 00:31:10:019.
/m/nfl/broadcast/tmp/2013091511.mp4: Expected duration 02:27:04:96 but it has 00:41:19:427.
```

So I ran `rm /m/nfl/broadcast/tmp/20130915{05,06,11}.mp4` and restarted the 
download command from last night. It will automatically retry the downloads for 
the games I just deleted.

If you run into problems, please thoroughly read
[nflvid's README](https://github.com/BurntSushi/nflvid/blob/master/README.md)
to make sure you haven't missed anything. Then either join us on IRC/FreeNode 
at `#nflgame` or
[open a new issue on nflvid's issue tracker](https://github.com/BurntSushi/nflvid/issues/new).


### Watching footage

Once you have at least one game sliced into its component play videos, you can 
start trying to filter plays by some search criteria and watch them.

Note though that each play video file has the file path `gsis_id/play_id.mp4`
(where `play_id` is padded out to four places, e.g., `0001`), so that you could 
watch a subset of plays using any search mechanism that returns play 
identifiers found in `nflgame` or `nfldb`. However, the machinery to make this 
convenient for you depends on `nfldb`.

In particular, we can use the 
[nflvid.vlc](http://pdoc.burntsushi.net/nflvid.vlc)
module to open *any* list of plays. The advantage of using `nflvid.vlc` is that 
it can write the context of each play as a text overlay of the video using 
vlc's marquee feature. I personally find this useful for coach footage when 
there is typically nothing on the screen except for the play.

The main function that you'll use in `nflvid.vlc` is
[watch](http://pdoc.burntsushi.net/nflvid.vlc#watch),
which takes a database from `nfldb.connect()`, a list of `nfldb.Play` objects 
and a directory containing sliced play video. If successful, it will execute a 
command that opens vlc with a playlist and all the requisite meta data.

Here is one more example of one of my favorite plays:

```python
import nfldb
import nflvid.vlc

db = nfldb.connect()
q = nfldb.Query(db)

q.game(season_year=2011, season_type='Regular')
q.player(full_name='Wes Welker').play(receiving_yds__ge=90, receiving_tds=1)

nflvid.vlc.watch(db, q.as_plays(), '/m/nfl/coach/pbp')
```


### Searching on the command line

nflvid comes with a special program that makes watching play video from the 
command line simple. The name of the command is `nflvid-watch`. Before 
continuing, you should run `nflvid-watch --help` and glance over the different 
options available.

Of particular note is the `NFLVID_FOOTAGE_PLAY_DIR` environment variable. When 
set, you won't need to tell `nflvid-watch` the play footage directory on every 
invocation.

The basic idea is to provide a thin interface to nfldb's query interface. For 
example, you can restrict plays to games in the 2012 regular season with: (I've 
included a limit just in case you try to run the command.)

```bash
nflvid-watch --game 'season_year=2012' --game 'season_type="Regular"' --limit 10
```

You can use shorter flag names if that's your thing:

```bash
nflvid-watch -g 'season_year=2012' -g 'season_type="Regular"' -l 10
```

Each of the criteria we specify should be inside single quotes and **should be 
valid Python code**. The string you provide is passed directly to Python's 
`eval` function.

We can specify additional search criteria by subsequent uses of the `--game` or 
`-g` flag, which is similar to using the
[Query.game](http://pdoc.burntsushi.net/nfldb#nfldb.Query.game)
method to specify search criteria for games. We can do something similar for 
drives, plays and players. Moreover, we can make `Clock`, `PossessionTime` or 
`FieldPosition` values with the `Clock`, `Field` or `PTime` functions.

Finally, there are some shortcuts available to us for common constraints. The 
`-y` or `--year` flag is the same as using `-g 'season_year={YEAR}'`. Similarly 
for `-t` for `season_type` and `-w` for `week`.

Here's another example: watching all pass completions by Tom Brady in the last 
2 minutes of the 4th quarter in the 2012 regular season:

```bash
nflvid-watch -y 2012 -p 'time__ge=Clock("Q4", "2:00")' -p 'time__le=Clock("Q4", "0:00")' -p 'passing_cmp=1' -r 'full_name="Tom Brady"'
```

The `-c` or `--current` flag can also be useful, since it automatically 
restricts games to the current season and week. For example, on the Saturday 
after the abysmal 2013 week 2 game between the Jets and the Patriots, I can run 
the following command to watch the only two touchdowns in the game:

```bash
nflvid-watch -c -p 'offense_tds=1'
```

