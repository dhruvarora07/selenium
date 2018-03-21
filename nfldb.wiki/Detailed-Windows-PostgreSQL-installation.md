# Step-by-step installation of PostgreSQL
## Windows 8 (64-bit) OS

1. Download [PostgreSQL-9.3.0-1-windows.exe](http://www.enterprisedb.com/postgresql-930-installers-win32?ls=Crossover&type=Crossover)
2. Try to run downloaded file but halted by too smart for its own good OS  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774385/b8f1f0c8-1929-11e4-99f2-c4d8f9316090.png" width="680px"/>
3. Click `More info` and `Run anyway`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774390/0287f980-192a-11e4-8494-8797233290ac.png" width="680px"/>
4. Setup wizard begins, click `Next`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774404/42963c4e-192a-11e4-93ac-ad35da1a9886.png" width="400px"/>
5. Specify installation directory, click `Next`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774418/8063fbc4-192a-11e4-979b-bf34264732f4.png" width="400px"/>
6. Specify data directory, click `Next`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774422/9bf8af7e-192a-11e4-8fea-97fc28d8e369.png" width="400px"/>
7. Assign password for superuser named `postgres`, click `Next`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774426/bc7f0590-192a-11e4-9524-893a930e2c9e.png" width="400px"/>
8. Leave port number with default `5432`, click `Next`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774436/ee746388-192a-11e4-9137-c63675fa9c10.png" width="400px"/>
9. Leave `Advanced Options` locale with `[Default locale]`, click `Next`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774442/15de9984-192b-11e4-9ffb-d3804f010e98.png" width="400px"/>
10. Wizard complete, ready to install, click `Next`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774445/3a50d372-192b-11e4-9f7c-53b915f31ba4.png" width="400px"/>
11. Wait while installation completes  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774454/5e2a4f12-192b-11e4-85dc-73f086acf070.png" width="400px"/>
12. Finished installing, click `Finish`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774466/a911f5ac-192b-11e4-86df-01ff8f7699eb.png" width="400px"/>
13. A new wizard opens up - Stack Builder 3.1.1; This wizard we are going to ignore. Click `Cancel` -> `Yes`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774482/d8a734f8-192b-11e4-8455-b19be898841e.png" width="400px"/>
<img src="https://cloud.githubusercontent.com/assets/2996205/3774487/f1db4b76-192b-11e4-8ed5-1dc3d703985c.png" width="240px"/>
14. Open Command Prompt and navigate to the install location selected in step 5. above  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774492/2727a00e-192c-11e4-9ccf-8f870ecacbc1.png" width="400px"/>
15. `createuser.exe -U postgres -E -P nfldb`  
Using superuser `postgres`, create a new user named `nfldb`.  Assign a password for user `nfldb`, enter it again to confirm, and finally enter the password for account `postgres` that you assigned in step 7. above  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774515/d3243db8-192c-11e4-8cff-689297fc6c0d.png" width="400px"/>
16. `createdb.exe -U postgres -O nfldb nfldb`  
Using superuser `postgres`, create a new database named `nfldb`.  Make the owner of this database user `nfldb`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774524/2ce7ff1a-192d-11e4-84a7-f6a3874ca894.png" width="400px"/>
17. `psql.exe -U postgres -c "CREATE EXTENSION fuzzystrmatch;" nfldb`  
Enable fuzzy string matching  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774537/9deff410-192d-11e4-91c0-6e2a6a2bac00.png" width="400px"/>
18. `psql.exe -U nfldb nfldb`  
Using user `nfldb`, log into database `nfldb`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774541/c3adf512-192d-11e4-9a13-7d6e0b6e8340.png" width="400px"/>  
`\q` to exit  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774568/5daa7a78-192e-11e4-92eb-c10d2a0d0c61.png" width="400px"/>  
19. Download [nfldb.sql.zip](http://burntsushi.net/stuff/nfldb/nfldb.sql.zip)  
20. Extract downloaded zip file  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774550/fbbc7da2-192d-11e4-9df4-dbd0e4e199ee.png" width="400px"/>
21. `psql.exe -U nfldb nfldb < x:\path\to\extracted\nfldb.sql\nfldb.sql`  
Using user `nfldb`, import the downloaded sql database into database `nfldb`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774586/b5d0e534-192e-11e4-9acd-66c68d5d87ca.png" width="400px"/>  
_Please note that you are importing the `nfldb.sql` *file* that you unzipped and not the `nfldb.sql` *folder* that the file resides in.  
If you are seeing `Access is denied.` it is likely that you are trying to incorrectly import the folder_
22.  Wait for database to import - took me about 6 minutes on older PC  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774631/d29c790c-192f-11e4-8725-a18d6ef6d1e1.png" width="400px"/>
23. Create a copy of the sample config file found in `x:\Python27\share\nfldb`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774651/19ae1c74-1930-11e4-8a89-e8a3f027372c.png" width="400px"/>  
Rename this copy `config.ini`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774665/4abd51fe-1930-11e4-9896-55a14d4cdf2e.png" width="400px"/>
<img src="https://cloud.githubusercontent.com/assets/2996205/3774661/444fc1bc-1930-11e4-8686-97181e54a034.png" width="280px"/>  
Edit two values in this config file: the timezone & the password for user `nfldb`; save and close the file  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774678/a7b7b066-1930-11e4-9747-970479bc8e97.png" width="400px"/>
24.  Create and save a file called `top-ten-qbs.py`
```python
import nfldb

db = nfldb.connect()
q = nfldb.Query(db)

q.game(season_year=2012, season_type='Regular')
for pp in q.sort('passing_yds').limit(10).as_aggregate():
    print pp.player, pp.passing_yds
```
<img src="https://cloud.githubusercontent.com/assets/2996205/3774695/3e3cf8de-1931-11e4-9c41-6b793e52a622.png" width="400px"/>  
25.  From a command prompt, run `python top-ten-qbs.py`  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774699/6cdd4ebe-1931-11e4-8c16-4472589f55bf.png" width="400px"/>  
26.  Finally, run the `nfldb-update` script found in the `Python\Scripts` folder  
<img src="https://cloud.githubusercontent.com/assets/2996205/3774710/e0f2a312-1931-11e4-96b9-ca531a999cbb.png" width="400px"/>  
27.  Congratulations - no one in the world has a more up-to-date `nfldb` database than you.


