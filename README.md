## Operating System

- The installation procedure is based on Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-51-generic x86_64)
- Minimum 8GB RAM required

## MariaDB 10.3 Installation

- Refer to https://mariadb.org/download/

```bash
$ sudo apt install software-properties-common dirmngr apt-transport-https
$ sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'
$ sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] https://mirror.yongbok.net/mariadb/repo/10.3/ubuntu focal main'

$ sudo apt update
$ sudo apt install mariadb-server-10.3
```
- Set the MariaDB user name and password to root and 123456

## Package Installation 

```bash
$ sudo apt install libgraphviz-dev g++ libncurses5-dev libssl-dev
$ sudo apt install libboost-graph-dev libboost-program-options-dev
$ sudo apt install libz-dev pkg-config libncurses5-dev flex bison cmake g++ gnutls-dev
```

## Source Code Compilation

Download the source code.

```bash
# Unzip the source code archive
$ cd mariadb-server
$ mkdir build; cd build
$ cmake ..
$ make -j 10
```

## Environment Configuration

- Delete the existing configuration

```bash
# Stop and disable the service
$ sudo systemctl stop mariadb
$ sudo systemctl disable mariadb

# Delete the configuration file (to avoid confusion)
$ cd /etc/mysql
$ rm {ALL FILES EXCEPT FOR: debian-start, debian.cnf}

# Delete created data files
# Do not delete folders (do not touch the mysql/ and performance_schema/ folders and their contents)
$ rm /var/lib/mysql/*
```

- Create a new configuration file

```bash
$ vi my.cnf

[client]
port            = 3306
socket          = /var/lib/mysql/mysql.sock

[mysqld]
user            = mysql
socket          = /var/lib/mysql/mysql.sock
pid-file        = /var/run/mysqld/mysqld.pid
port            = 3306
lc_messages     = en_US
lc_messages_dir = /usr/share/mysql

#
# * IMPORTANT: Additional settings that can override those from this file!
#   The files must end with '.cnf', otherwise they'll be ignored.
#
!includedir /etc/my.cnf.d/
```

- Apply our new DB configuration file

```bash
$ mkdir -p /etc/my.cnf.d/
$ cp {mariadb-server directory path}/db_state_bench/base_script/*.cnf /etc/my.cnf.d/
```

- Create our new DB log folder

```bash
$ mkdir -p /var/lib/mysql/log/
$ chown mysql:mysql /var/lib/mysql/log
$ mkdir -p /var/run/mysqld/
$ chown mysql:mysql /var/run/mysqld
```

### Run our new DB server

```bash
$ cd {mariadb-server directory path}/build/sql
$ ./mysqld --user=mysql
```

- Check the client's connection

```bash
$ mysql -uroot -p123456
```

#### Error Case

- ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)

```bash
# Run mysql without the password
$ mysql -uroot
MariaDB [(none)]> alter user 'root'@'localhost' identified by '123456';
MariaDB [(none)]> commit;
# Run "mysql -uroot -p123456" to check connection
```

## oltpbench

### Installation

- Refer to https://github.com/oltpbenchmark/oltpbench/wiki

```bash
$ git clone https://github.com/oltpbenchmark/oltpbench.git
$ cd oltpbench
$ ant bootstrap
$ ant resolve
$ ant build
```

### Benchmark

- Change the script path (minimum 8GB RAM required)

```bash
$ cd {mariadb-server directory path}/db_state_bench/base_script
$ vi mysql_env.sh

MYSQL_SOURCE_PATH={mariadb-server directory path}/build
OLTPBENCH_PATH={oltpbench directory path}
```

- Retroactive Operation Test 1

```bash
$ cd {mariadb-server directory path}/db_state_bench/user_command_query_test

# TRANSACTION test (e.g., BEGIN ... COMMIT)
$./run_test_with_oltpbench_tpcc.sh trx

# PROCEDURE test (e.g., CALL ...)
$./run_test_with_oltpbench_tpcc.sh proc

# Newly generated retroactive ADD/DEL query file: ./user_query2.sql
# The mysqldump file before/after retroactive operation : /var/lib/mysql/log/tpcc_dump_{base, my}_modified
# The table SELECT results before/after retroactive operation: /var/lib/mysql/log/tpcc_dump_{base, my}_modified.{HISTORY, DISTRICT, CUSTOMER, WAREHOUSE}

# DB Comparison of before/after retroactive operation
$ diff /var/lib/mysql/log/tpcc_dump_*_modified
$ diff /var/lib/mysql/log/tpcc_dump_*_modified.HISTORY
$ diff /var/lib/mysql/log/tpcc_dump_*_modified.DISTRICT
$ diff /var/lib/mysql/log/tpcc_dump_*_modified.CUSTOMER
$ diff /var/lib/mysql/log/tpcc_dump_*_modified.WAREHOUSE

$ ./run_test_with_oltpbench_twitter.sh
```

- Retroactive Operation Test 2

```bash
$ cd {mariadb-server directory path}/db_state_bench/oltpbench

# Efficient retroactive operation test
$ ./tpcc_run.sh -m

# Naive (MariaDB) retroactive operation test
$ ./tpcc_run.sh -o
```


## Description of our special DBs used for retroactive operation mode
- STATE_LOG_CHANGE_DB: A temporary DB for executing retroactive operation queries (needed only during retroactive operation)
- STATE_LOG_BACKUP_DB: A backup DB that keeps deleted tables (to retroactively restore them if needed)
- STATE_GROUP_DB: A DB that contains each query's time information separated by gid (group id)

## Execution

### Query Dependency Analysis Tool

- The mysql server should be running
- Analyze the input query file and generate a .json graph structure file and a .svg graph image file 

```bash
$ cd build/client
$
# $ query_analyzer -u root -p 123456 --query {INPUT QUERY FILE}  --key-name {COLUMN NAME} --info {OUTPUT JSON INFO FILE} --query {OUTPUT SVG GRAPH FILE}
$ query_analyzer -u root -p 123456 --query mysql-test/state-db-test/query_analyzer_col2.sql --key-name tpcc.DISTRICT.D_ID
```

### Choosing the clustering column and using it

```bash
# The final time for analysis is the last query's time in the binary log file
# $ db_state_change -u[mysql user ID] -p[mysql PW] --start-datetime [analysis starting time] --candidate-db [the target DB]
$ db_state_change -uroot -p123456 --start-datetime 2021-04-11 09:30:00 --candidate-db test_data > candidate_output
$ KEY_NAME=`cat candidate_output | grep 'STRIP CANDIDATE COLUMN' | awk -F ': ' '{print $2}'`
$ db_state_change -uroot -p123456 --key-name "$KEY_NAME" --key-type "int"

# - Each clustering key (column) type is regarded as follows
#    unit, unsigned -> uint
#    int, bigint, smallint, tinyint, bit -> int (or uint if there exists an unsigned flag)
#    float, double -> double
#    str, string, enum, char, varchar -> string
#    (Only above 4 final key types are considered to have ==, !=, >, >=, <, <= operations)
#    (TEXT, BLOB, etc are regarded as binary and thus cannot be used as a clustering key)
#    (Any other types are not allowed to be used as a clustering key)

```

### Retroactive Operation

```bash
# redo
# Rollback the target DB to the START DATE TIME and execute queries after the REDO DATE TIME, all "efficiently"
$ ./db_state_change -uroot -p123456 --start-datetime {START DATE TIME} --redo-datetime {REDO DATE TIME} --redo-db {REDO DB}


# Retroactively add the queries in the INPUT QUERY FILE as the DATE TIME (and generate the query dependency graph)
$ ./db_state_change --query {INPUT QUERY FILE} -uroot -p123456 --start-datetime {DATE TIME}
$ ./db_state_change --query {INPUT QUERY FILE} -uroot -p123456 --start-datetime {DATE TIME} --graph {SVG OUTPUT FILE}


# Retroactively add/deletes queries at specific moments as stated in the INPUT QUERY FILE (which uses our special syntax of "timestamp" and "add/del" commands)
$ ./db_state_change --query2 {INPUT QUERY FILE IN THE query2 FORMAT} -uroot -p123456
$ ./db_state_change --query2 ../../db_state_bench/user_command_query_test/user_query_sample.sql -uroot -p123456 --graph {SVG OUTPUT FILE}


# Group Query
# Retroactively delete all queries that belong to the GROUP ID
$ ./db_state_change --gid {GROUP ID} -uroot -p123456
$ ./db_state_change --gid {GROUP ID} -uroot -p123456 --graph {SVG OUTPUT FILE}

# Same as the above, and add new queries in the INPUT QUERY FILE at the earliest timestamp among the GID queries
$ ./db_state_change --gid {GROUP ID} --query {INPUT QUERY FILE} -uroot -p123456
$ ./db_state_change --gid {GROUP ID} --query {INPUT QUERY FILE} -uroot -p123456 --graph {SVG OUTPUT FILE}


# Retroactive operation with a manually chosen clustering key column
$ ./db_state_change --gid {GROUP ID} -uroot -p123456 --key-name {COLUMN NAME} --key-type {COLUMN DATA TYPE}
$ ./db_state_change --gid {GROUP ID} -uroot -p123456 --key-name {COLUMN NAME} --key-type {COLUMN DATA TYPE} --graph {SVG OUTPUT FILE}

# Compute the clustering key weight (Sigma sum) for all queries for all columns and return the lightest column
$ ./db_state_change -uroot -p123456 --start-datetime "2021-04-11 09:30:00" --candidate-db test_data | tee candidate_output
$ cat candidate_output | grep 'STRIP CANDIDATE COLUMN' | awk -F ': ' '{print $2}'
$ ./db_state_change -uroot -p123456 --key-name <key name from the above> --key-type <e.g., "int">
```



### Committed Queries Viewer
- Print so-far-committed queries (up to the retroactively updated state) as JSON or log. The log format prints time and query. The JSON format also prints the read/write table and column information

```bash
$ ./state_log_viewer --start-datetime {START TIME} --end-datetime {END TIME / if not set, up to now} --output {output path} --format {output format : log / json}
```
