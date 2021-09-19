# Postgres with Orbits


### Build

We use environment variable `ORBITDIR`, which if set should point to the orbit 
userlib root, to decide if we should compile Postgres with orbits. 

```bash
export ORBITDIR=/home/ryan/userlib
mkdir -p ../build/postgres
mkdir -p ../dist/postgres
dist=$(cd ../dist/postgres && pwd)
cd ../build/postgres
../../postgres/configure --enable-debug CFLAGS=-ggdb -O0 -g3 -fno-omit-frame-pointer --prefix=$dist
make -j8
make install
```

### Run

#### 1. Initialize the data directory 

Only need to do this for the first time:

```bash
export LD_LIBRARY_PATH=/home/ryan/userlib/build/lib
cd ../dist/postgres
./bin/initdb -D data/
```

#### 2. Start the server

```bash
bin/postgres -D data
```

#### 3. Create test database

```bash
bin/createdb benchmark
bin/pgbench -i -s 50 benchmark
```

#### 4. Test Autovacuum orbits

Autovacuum has a launcher and a set of workers. When the launcher orbit starts
successfully, the console should print something like:

```bash
Created orbit <mpid 7706, lobid 1, gobid 7711>
LOG:  created orbit 7711 autovacuum launcher
LOG:  initialized AutoVacuumInfo (size 296) in orbit pool
LOG:  autovacuum orbit enters entry point
LOG:  av_launcherpid is 7711
LOG:  autovacuum launcher started
LOG:  database system is ready to accept connections
LOG:  allocated dblist from orbit pool!
LOG:  starts autovacuum worker from launcher
LOG:  allocated dblist from orbit pool!
LOG:  starts autovacuum worker from launcher
LOG:  allocated dblist from orbit pool!
...
```

Note that if autovacuum does not find a database with `pgstat` entry, it 
will sleep and check later. No actual worker orbits would be created 
despite the periodic `starts autovacuum worker from launcher` debug log message.
This is the case when no database other the default ones exist or there is no activity.

To trigger autovacuum, we need to interact with some database. For example, 
open a terminal:

```bash
$ bin/psql benchmark
psql (9.6.23)
Type "help" for help.

benchmark=# \list
                              List of databases
   Name    | Owner | Encoding |   Collate   |    Ctype    | Access privileges
-----------+-------+----------+-------------+-------------+-------------------
 benchmark | ryan  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | ryan  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | ryan  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/ryan          +
           |       |          |             |             | ryan=CTc/ryan
 template1 | ryan  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/ryan          +
           |       |          |             |             | ryan=CTc/ryan
 test      | ryan  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(5 rows)

benchmark=# \list
...
(5 rows)
benchmark=# \q
```

Then you should see new log messages in the `postgres` console indicating new 
worker orbits being created:

```bash
LOG:  sent orbit update of dboid 16385
LOG:  allocated dblist from orbit pool!
LOG:  postmaster starts autovacuum worker
LOG:  received dboid 16385 from launcher orbit
Created orbit <mpid 7706, lobid 2, gobid 7727>
LOG:  created avac_worker_1 orbit 7727
LOG:  autovac worker orbit entry: dboid 16385
^CLOG:  received fast shutdown request
```
