Whenever you want to update your database, run the following command:

    nfldb-update

After it terminates, if there were no errors, your database will be completely 
up to date with current game data. If you want to continuously update nfldb at 
some interval, then you can use the `--interval` flag:

    nfldb-update --interval 30

Which will run the update routine every 30 seconds in an infinite loop. The 
only way to stop this process is by sending a `TERM` signal to the process 
(i.e., pressing `Ctrl-C` in a terminal).


### Run as a daemon

On my system, I run `nfldb-update --interval 30` as a daemon. (A daemon is just 
a long running process that executes in the background.) The advantage to 
running it as a daemon is that it's easy to start and stop, and have it 
automatically execute when you boot your machine.

Unfortunately, running a custom program as a daemon can vary depending on your 
distribution. I'll get it started with giving an example for `systemd`. If you 
get it working with a different init system, please add it below.


#### Systemd

I've put this in a file called `/etc/systemd/system/nfldb.service`:

```
[Unit]
Description=nfldb script to update the nfldb database
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/daemonize -u {LINUX_USER} -e /path/to/nfldb-update.log /full/path/to/nfldb-update --interval 30
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```

You can then start, stop and restart it with the following commands:

```bash
systemctl start nfldb
systemctl stop nfldb
systemctl restart nfldb
```

And enable it to automatically start on boot:

```bash
systemctl enable nfldb
```

Note that this needs `daemonize` to be installed on your machine. I recommend 
using `daemonize` with other init systems as well, since `nfldb-update` does 
not become a Linux daemon on its own.


#### A poor man's daemon

If adding a daemon using your init system just isn't working, then you can use 
the `nohup` command for a makeshift daemon: (`nohup` prevents your program from 
quitting after you close your terminal)

```bash
nohup nfldb-update --interval 30 > /path/to/nfldb-update.log 2>&1 &
```

This will run `nfldb-update` in the background and log its stdout and stderr
to `/path/to/nfldb-update.log`. You may close your terminal after running the 
above command without the program stopping. The only way to stop it from 
running is to kill it:

```bash
killall nfldb-update
```

Killing the `nfldb-update` script is perfectly safe. Even if it's in the middle 
of updating the database, the current transaction will fail and no changes will 
be made to the database. In other words, assuming there are no relevant bugs in 
nfldb, it is impossible to corrupt your database by killing `nfldb-update`.


### The grimy details

All data in `nfldb` is retrieved via 
[nflgame](https://github.com/BurntSushi/nflgame), which queries a JSON feed on 
NFL.com that is updated approximately every 15 seconds. Those data updates can 
be inserted in your `nfldb` database by running the `nfldb-update` command.

**Warning**: `nfldb-update` can be run on **any** database, including
an empty one; however, you should not initially load your database with
`nfldb-update` since it will be very slow. Instead, please refer to the
[installation](Installation)
guide for details on how to setup your database.

Running `nfldb-update` on a populated database **will only update data that has 
changed on NFL.com**, which means that it is typically very fast. 
`nfldb-update` will also update your player database periodically with 
up-to-date meta data (like team, position, etc.). If there are no new updates 
available, then `nfldb-update` will not modify your database (which makes it 
idempotent). Finally, `nfldb-update` acquires a write lock on each table, which 
means that it is the only program that can write to the database while it is 
running. (Other programs can still query the database while `nfldb-update` is 
running.) This means that it is actually permissible to have multiple
instances of `nfldb-update` running simultaneously on the same database;
they will not conflict with each other. (Although I can't currently think of
why you would want to do that.)

