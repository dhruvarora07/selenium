Installing nfldb is a two step process. The first step is to install a PostgreSQL 
server on your system and import a nfldb database. The second step is to install
the nfldb Python module so that you can interface conveniently with the 
database installed in step one.

Note that this means that the name `nfldb` actually refers to two separate and 
distinct things: the database schema with the NFL data contained inside a
database with that schema, and the Python code that is used to update and query 
that data. In particular, the Python code is **not strictly necessary**. You 
could import the database and query it with any tool you like. However, the 
nfldb module provides several important things:

* A `nfldb-update` script that imports data from `nflgame` and efficiently 
  updates the database with data from active games.
* Convenient Python types for representing data in the database.
* Automatic schema migrations when the schema is changed.
* A convenient query interface for quickly searching the data in the database 
  without having to write any SQL.

Therefore, I strongly encourage the use of the nfldb Python module with the 
nfldb database schema.

**It is important that you install the database before the nfldb Python 
module**. This is to prevent nfldb from accidentally creating an empty database 
and inserting all of the data via the `nfldb-update` script. While this will 
work, it will take considerably longer than importing a ready-made database.

One last note before we begin: I am not a Windows user and have therefore not 
devoted any significant resources to making this guide friendly for Windows 
users. I wholeheartedly welcome contributions to make this guide more Windows 
(or Mac) friendly. [ochawkeye](https://github.com/ochawkeye) has written a
[Windows installation](Windows-Install)
guide that should be useful to read concurrently with this guide.


## Getting help

If you run into any kind of trouble while trying to get nfldb installed, please 
[open a new issue on nfldb's issue 
tracker](../issues/new). If you're thinking, 
"I'm not sure if this belongs on an issue tracker..." then my answer to you is: 
"Yes it does! Post it!" Not only does it help me stay organized, but it 
benefits others that come after you who run into similar problems.

Alternatively, please join us on the IRC channel `#nflgame` at FreeNode and I 
or someone else can help you as soon as we can.


## Setting up the nfldb database

If you don't already have PostgreSQL installed on your system, [you will need 
to install it](https://wiki.postgresql.org/wiki/Detailed_installation_guides).

I have been told that [Russ Brooks provides an excellent guide for setting up 
PostgreSQL on a 
Mac](http://www.russbrooks.com/2010/11/25/install-postgresql-9-on-os-x).


### Creating a database

If you're using Ubuntu, you may find the [Ubuntu 
Installation](Installation-on-Ubuntu) page useful for instructions on 
installing PostgreSQL. Once you're done on that page, you should come back here
for importing the database (skipping this section).

Once PostgreSQL is installed, you will need to add a new database that you can 
import nfldb into. On Archlinux with systemd, these are the rough steps that I
tend to follow:

As `root`:

    systemd-tmpfiles --create postgresql.conf
    mkdir /var/lib/postgres/data
    chown -c -R postgres:postgres /var/lib/postgres

As Linux user `postgres`, initialize the PostgreSQL data directory and 
configure it to always use UTF-8 when creating new databases:

    initdb -D /var/lib/postgres/data --locale=en_US.UTF-8 --encoding=UNICODE

Start the postgresql server, login as postgres and set an encrypted password:

    systemctl enable postgresql
    systemctl start postgresql
    psql -U postgres
    # ALTER ROLE postgres WITH ENCRYPTED PASSWORD 'choose a superuser password';

Now edit `/var/lib/postgres/data/pg_hba.conf` (as `root` or as `postgres`) and 
change all instances of `trust` to `md5`. Restart postgresql:

    systemctl restart postgresql

Create a new user for your nfldb database:

    createuser -U postgres -E -P nfldb

Enter the new user's password twice and then enter the password for postgres
(that you set above with the `ALTER ROLE ...` SQL command). The -E parameter 
specifies that the password should be encrypted.

Create a database with the same name as the user (you will have to enter the
password for the postgres user again):

    createdb -U postgres -O nfldb nfldb

And enable the `fuzzystrmatch` extension so that you can do fuzzy player 
searching with 
[player_search](http://pdoc.burntsushi.net/nfldb#nfldb.player_search):

    psql -U postgres -c 'CREATE EXTENSION fuzzystrmatch;' nfldb

Note that the above command requires superuser access, which is what the
`-U postgres` is for (it logs in as the `postgres` user).

You should now be able to login as that user and connect to `nfldb`:

    psql -U nfldb nfldb

The above instructions are very likely to vary depending on the Linux 
distribution that you're using. Also, the above options set up your PostgreSQL
server to be a bit more secure than is required. Finally, I have no idea what 
the steps are on Mac or Windows. Remember that all you need to do is create an 
empty database.


### Importing the nfldb database

[Download a zip file of the SQL to create the 
database](http://burntsushi.net/stuff/nfldb/nfldb.sql.zip).
Unzip it with whatever utility you like. Once **you've unzipped the downloaded 
file**, you can import it into the database you created in the previous step:

    psql -U nfldb nfldb < nfldb.sql

Depending on your machine, this step may take a while. On my considerably beefy 
machine, it takes a couple minutes. (Most of the time is spent creating 
indexes.)

To make sure it worked, try connecting to the database and viewing the list
of tables:

    [andrew@Liger nfldb] psql -U nfldb nfldb
    psql (9.2.4)
    Type "help" for help.
    
    nfldb=# \d
               List of relations
     Schema |    Name     | Type  | Owner  
    --------+-------------+-------+-------
     public | agg_play    | table | nfldb
     public | drive       | table | nfldb
     public | game        | table | nfldb
     public | meta        | table | nfldb
     public | play        | table | nfldb
     public | play_player | table | nfldb
     public | player      | table | nfldb
     public | team        | table | nfldb
    (7 rows)

If you're a Windows user, [follow this guide.](https://github.com/BurntSushi/nfldb/wiki/Detailed-Windows-PostgreSQL-installation)


## Installing the nfldb Python module

If you've made it this far, congratulations&mdash;the hard part is over!
The `nfldb` Python module should be installed through `pip` from
[PyPI](https://pypi.python.org):

    pip2 install nfldb

It will automatically install dependencies for you (which include `nflgame`,
`psycopg2`, `pytz` and `enum34`).

**NOTE:** If you are installing on a machine with limited memory and the install fails with a `MemoryError`, install using the no-cache-dir flag:

    pip2 install --no-cache-dir nfldb

**NOTE:** If install fails with a compile error such as (CentOS) `./psycopg/psycopg.h:31:22: fatal error: libpq-fe.h: No such file or directory`, you may need to install an additional postgres module

* On Ubuntu systems: `sudo apt-get install libpq-dev`
* On RHEL/CentOS systems: `sudo yum install postgresql-devel`
* On Mac: `brew install postgresql`

If you don't have `pip` on your machine, then you should first make sure that 
Python 2.7.x is installed (and **not** Python 3.x). Then, to install `pip`, 
follow the instructions in this
[StackOverflow answer](http://stackoverflow.com/a/12476379/619216).

Note that [nfldb's PyPI page](https://pypi.python.org/pypi/nfldb) also 
has a download link to a Windows installer that might work for you if you 
already have Python installed on your system. However, I don't believe it will
automatically install dependencies for you. Your best bet, I think, is to get
`pip` working on your machine.


### Configuring nfldb

Once the nfldb Python module is installed, only one step remains: we need to 
tell nfldb how to connect to the database you created earlier. You'll need
to find the default configuration file that was installed with nfldb, or use 
the sample in the 
[repository](https://github.com/BurntSushi/nfldb/blob/master/config.ini.sample). 
On Unix based systems, it will probably be in one of these locations:

    /etc/xdg/nfldb/config.ini.sample
    /usr/share/nfldb/config.ini.sample
    /usr/local/share/nfldb/config.ini.sample

If you used the `--user` flag with your `pip install`, it should be here:

    ~/.local/share/nfldb/config.ini.sample

If you can't find it, try running `locate config.ini.sample`. If that still
doesn't work, run `sudo updatedb` and try `locate config.ini.sample` again.

When you find it, you should copy it to your default configuration directory 
and name it `config.ini`:

    mkdir -p $HOME/.config/nfldb
    cp /etc/xdg/nfldb/config.ini.sample $HOME/.config/nfldb/config.ini
    # Change /etc/xdg/nfldb/config.ini.sample to wherever it is on your machine.

If you're on Windows, try to find where Python is installed in your machine 
(probably `C:\Python27\share\nfldb`) and look in there for a directory path 
similar to the one above. You should then copy `config.ini.sample` to 
`config.ini` in the same directory and ignore the stuff above about your 
"default configuration directory".

Once you have a `config.ini` file (and ***not*** a `config.ini.sample` file),
open it with your favorite text editor and fill in the information. If you've
followed the above instructions, then the only thing you must change is the
`password` setting. You may also want to change the `timezone` setting
or else game times will not be correct for you.


## Testing that nfldb is installed and working

If everything is up and running, then you should be able to run this program 
and see a list of the top ten quarterbacks by passing yards in the 2012 regular 
season. Save this to a file called `top-ten-qbs.py`:

```python
import nfldb

db = nfldb.connect()
q = nfldb.Query(db)

q.game(season_year=2012, season_type='Regular')
for pp in q.sort('passing_yds').limit(10).as_aggregate():
    print pp.player, pp.passing_yds
```

Then run `python2 top-ten-qbs.py`. The output I get (in about 1.3 seconds) is:

```bash
[andrew@Liger nfldb] python2 top-ten-qbs.py
Drew Brees (NO, QB) 5177
Matthew Stafford (DET, QB) 4965
Tony Romo (DAL, QB) 4903
Tom Brady (NE, QB) 4799
Matt Ryan (ATL, QB) 4719
Peyton Manning (DEN, QB) 4667
Andrew Luck (IND, QB) 4374
Aaron Rodgers (GB, QB) 4303
Josh Freeman (TB, QB) 4065
Carson Palmer (ARI, QB) 4018
```

Finally, you should try running the `nfldb-update` script:

```bash
nfldb-update
```

And this is the output I get at the moment:

```bash
[andrew@Liger nfldb] nfldb-update 
Connecting to nfldb... done.
Setting timezone to UTC... done.
-------------------------------------------------------------------------------
STARTING NFLDB UPDATE AT 2013-09-11 22:08:07.225584+00:00
Locking write access to tables... done.
Updating season phase, year and week... done.
FINISHED NFLDB UPDATE AT 2013-09-11 22:08:07.327157+00:00
-------------------------------------------------------------------------------
```

Note that you may have different output, particularly if games have been played 
since the SQL database you downloaded was last updated. In order to keep your 
database up to date with current statistics, please see
[updating nfldb with real time 
data](Updating-nfldb-with-real-time-data).

