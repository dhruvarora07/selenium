NOTE: Getting your Python environment successfully configured is an important first step; if you are new to Python, then [Python & Pip installation steps](https://github.com/BurntSushi/nfldb/wiki/Python-&-pip-Windows-installation) may be helpful to you before continuing with these instructions.

In general, the steps for installing `nfldb` in Windows are the same as instructions detailed [here](Installation) with a few exceptions.

1. Trying to `pip install nfldb` will fail because installing dependency `psycopg2` fails

2. Manually install `psycopg` from http://www.lfd.uci.edu/~gohlke/pythonlibs/#psycopg
 (In Windows 8, it may be necessary to click "More Info" and "Run Anyways" options)

3. Now we can `pip install nfldb` (or `pip install --upgrade nfldb`)  
_For you visual learners out there, steps 4.-15. below are reproduced in greater detail [here](https://github.com/BurntSushi/nfldb/wiki/Detailed-Windows-PostgreSQL-installation)_

4. Download and install PostgreSQL:
  - here: https://www.postgresql.org/download/windows/
  - or here: http://www.enterprisedb.com/products-services-training/pgdownload#windows
    * Note: As part of the install process, you will be prompted to create a password for user `postgres`.
    * Your installation may offer you a wizard for configuring your database; we're going to ignore that and instead open a command prompt to configure manually.
    * Once complete, navigate to your install directory (i.e `C:\Program Files (x86)\PostgreSQL\9.3\bin`) which will be our working directory for next few steps.

5. Create a new user

      ```
      C:\Program Files (x86)\PostgreSQL\9.3\bin>createuser.exe -U postgres -E -P nfldb
      Enter password for new role:
      Enter it again:
      Password:
      ```
6. Create a new db

    ```
    C:\Program Files (x86)\PostgreSQL\9.3\bin>createdb.exe -U postgres -O nfldb nfldb
    Password:
    ```
7. Enable `fuzzystrmatch`

    ```
    C:\Program Files (x86)\PostgreSQL\9.3\bin>psql.exe -U postgres -c "CREATE EXTENSION fuzzystrmatch;" nfldb
    Password for user postgres:
    CREATE EXTENSION
    ```
8. Login and connect to `nfldb`
    ```
    C:\Program Files (x86)\PostgreSQL\9.3\bin>psql.exe -U nfldb nfldb
    Password for user nfldb:
    psql (9.3.0)
    WARNING: Console code page (437) differs from Windows code page (1252)
             8-bit characters might not work correctly. See psql reference
             page "Notes for Windows users" for details.
    Type "help" for help.

    nfldb=> 
    ```
9. Download and unzip zip file of the SQL to create the database from http://burntsushi.net/stuff/nfldb/nfldb.sql.zip
10. Import the database (on old Q9300 @ 2.5GHz, 4GB this took just under 6 minutes)

    ```
    C:\Program Files (x86)\PostgreSQL\9.3\bin>psql.exe -U nfldb nfldb < c:\Users\Ben\Downloads\nfldb.sql\nfldb.sql. 
    Password for user nfldb: xxxxxx
    ```
11. Create a copy of `c:\Python27\share\nfldb\config.ini.sample`. Name this copy `config.ini`.
12. Modify this new `config.ini` `nfldb` config file with your nfldb password and timezone
13. Create and save a file called `top-ten-qbs.py`

    ```python
    import nfldb

    db = nfldb.connect()
    q = nfldb.Query(db)

    q.game(season_year=2012, season_type='Regular')
    for pp in q.sort('passing_yds').limit(10).as_aggregate():
        print pp.player, pp.passing_yds
    ```
14. run `python top-ten-qbs.py`

    ```
    C:\Python27\Scripts>python "P:\Projects\Home Computer\Fantasy Football\nfldb\top-ten-qbs.py"
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
15. Finally, try running the `nfldb-update` script

    ```
    C:\Python27\Scripts>python nfldb-update
    -------------------------------------------------------------------------------
    STARTING NFLDB UPDATE AT 2013-09-23 11:40:49.144000
    Connecting to nfldb... done.
    Setting timezone to UTC... done.
    Updating player JSON database... (last update was 2013-09-22 14:48:38.196084+00:
    00)
    Loading games for REG 2013 week 3
    Downloading team rosters...
    32/32 complete. (100.00%)
    Done!
    done.
    Locking player table...
    Updating 5285 players... done.
    Locking write access to tables... done.
    Updating season phase, year and week... done.
    Bulk inserting data for 1 games...
            Sending batch of data to database...
    done.
    Closing database connection... done.
    FINISHED NFLDB UPDATE AT 2013-09-23 11:41:23.256000
    -------------------------------------------------------------------------------
    ```