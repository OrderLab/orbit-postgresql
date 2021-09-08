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

Initialize the data directory (only for the first time):

```bash
export LD_LIBRARY_PATH=/home/ryan/userlib/build/lib
cd ../dist/postgres
./bin/initdb -D data/
```

Start the server:

```bash
bin/postgres -D data
```

Create test database and run shell:

```bash
./bin/createdb test
./bin/psql test
```
