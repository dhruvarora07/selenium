This is a guide for setting up `postgresql` on an Ubuntu system (which does not 
use `systemd`). It may vary a bit depending on your software versions, but 
hopefully it is close enough to get you started. To reduce duplication, I have 
removed most of the text before and after the instructions, since it applies to 
all linux distros, and have also removed notes about Windows or Mac 
installation, as the point of this wiki is to be Ubuntu-specific (though my 
guess would be it applies to most Debian-based systems). Thus, please start 
your installation by reading the beginning of the main [installation 
wiki](https://github.com/BurntSushi/nfldb/wiki/Installation).

**It is important that you install the database before the nfldb Python 
module**. This is to prevent nfldb from accidentally creating an empty database 
and inserting all of the data via the `nfldb-update` script. While this will 
work, it will take considerably longer than importing a ready-made database.


## A note about users

Many of the bash commands below need to be run as a specific user (i.e., `root` 
or `postgres`). To run a single command as root, start that line with `sudo`. 
To run a single command as `postgres`, precede it with `sudo -u postgres`.

For convenience, you can also run all commands as `postgres` by typing:

    sudo su - postgres

You should note the bash prompt change as a result.


## Setting up the nfldb database

If you don't already have PostgreSQL installed on your system, [you will need 
to install it](https://wiki.postgresql.org/wiki/Detailed_installation_guides). 
There are specific Ubuntu installation 
[instructions](https://help.ubuntu.com/community/PostgreSQL), but it should be 
as simple as running the following command as `root`.

    apt-get install postgresql postgresql-contrib


### Creating a database

Once PostgreSQL is installed, you will need to add a new database that you can 
import nfldb into. On Ubuntu, here are the rough steps that I followed:

As `root`:

    mkdir /var/lib/postgresql/data
    chown -c -R postgres:postgres /var/lib/postgresql

Start the postgresql server, login as `postgres` and set an encrypted password 
(the single quotes **do** need to surround your password):

    service postgresql enable
    service postgresql restart
    psql -U postgres
    # ALTER ROLE postgres WITH ENCRYPTED PASSWORD 'choose a superuser password';

_If you get an error about the lock file when you try to restart_ (I did not, 
but my friend did), you might need to run this command as `root`:

    chown -R postgres:postgres /var/run/postgresql/

Now edit `/etc/postgresql/9.1/main/pg_hba.conf` (as `root` or as `postgres`) 
replacing `9.1` with whatever you postgres version is and find the lines that 
read:

    local    all    postgres    peer
    local    all    all    peer

And change both `peer` to `md5`.

Then restart:

    service postgresql restart

Create a new user for your nfldb database:

    createuser -U postgres -E -P nfldb

Enter the new user's password twice and then enter the password for postgres
(that you set above with the `ALTER ROLE ...` SQL command). The -E parameter 
specifies that the password should be encrypted. The user does not need to be a 
superuser, nor do they need to be able to create new roles. I said that they 
should be allowed to create new databases, but to be honest I'm not sure if 
that matters or not.

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

Now you can head back to the main installation instructions and start with
[importing the 
database](Installation#importing-the-nfldb-database).

