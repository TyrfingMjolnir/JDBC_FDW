# Installation instructions for "packaged" installation of PostgreSQL with a sample config of Apple FileMaker 16 Server as JDBC server through Postgres FDW for illumos, joyent smartos, or other open solaris distros

## Versions used in this example
+ Postgres 9.6
+ FileMaker 16 Server JDBC
+ OpenJDK 7

# How I did the [install](#install), [config](#config), and [testing](#testing)
## <a name="install"></a>Install

1) Login to a Joyent SmartOS Zone, mine is called sn07postgres04 installed prerequisits. 
```
zlogin <<someUUID>>
root@smartzone:~# pkgin update
root@smartzone:~# pkgin upgrade
root@smartzone:~# pkgin in openjdk7 build-essential postgresql96 postgresql96-contrib vim tmux 
root@smartzone:~$ updatedb
root@smartzone:~# vim $( pg_config --pgxs )
mkdir -p /opt/local/dev/
cd /opt/local/dev/
```

2) Get postgres 9.6 source.
```
root@smartzone:~$ git clone -b REL9_6_STABLE https://github.com/postgres/postgres
root@smartzone:~$ cd postgres
root@smartzone:~$ ./configure
```
3) Get JDBC_FDW source.
```
root@smartzone:~$ cd contrib
root@smartzone:~$ git clone https://github.com/atris/JDBC_FDW
```

4) Enter the JDBC_FDW folder.
```
root@smartzone:~$ cd JDBC_FDW
``` 

5) Execute make clean
```
root@smartzone:~$ sudo PATH=/opt/local/bin/:$PATH make USE_PGXS=1 clean
```
Assuming you have PostgreSQL installed in /usr/lib/postgresql.If not,please 
make appropriate changes to the $PATH value set in the command.

7) Execute make install
(You may have to change to root/installation privileges before executing Make 
Install)

### Important** : Before running Make Install,please execute the following command:
```
root@smartzone:~# ln -s /opt/local/java/openjdk7/jre/lib/amd64/server/libjvm.so /opt/local/lib/libjvm.so
```
Please replace the path in the command with the path to libjvm.so
(it should be in Java JRE folder).

For locating libjvm.so,you can use the following command:
```
root@smartzone:~# locate libjvm.so
```
Please give the path to libjvm.so in the 'server' folder if multiple libjvm.so 
files are being shown.

After running one of the above commands, please execute the following command:
```
root@smartzone:~# PATH=/opt/local/bin/:$PATH make USE_PGXS=1 install
```
Assuming you have PostgreSQL installed in /usr/lib/postgresql.If not,please 
make appropriate changes to the $PATH value set in the command.

8) Ensure ```make install``` executes successfully without any warning or errors.

# Config <a name="config"></a>

0) As a general design pattern I prefer to have a separate schema for different purposes, such as ```CREATE SCHEMA fm``` there is also an option to omit "fm." from the example config below.

1) Enter psql.
```
root@smartzone:~# psql -U postgres
```

2) Set up jdbc_fdw extension.
```
\c yourdesireddatabase;
CREATE EXTENSION jdbc_fdw;
```
Optionally you can list available FDWs on your server
```
SELECT * FROM pg_available_extensions WHERE name ~ 'fdw';
```
mine outputted
```
testdb=# SELECT * FROM pg_available_extensions WHERE name ~ 'fdw';
     name     | default_version | installed_version |                      comment
--------------+-----------------+-------------------+----------------------------------------------------
 mysql_fdw    | 1.1             | 1.1               | Foreign data wrapper for querying a MySQL server
 postgres_fdw | 1.0             |                   | foreign-data wrapper for remote PostgreSQL servers
 jdbc_fdw     | 1.0             | 1.0               | Foreign data wrapper for querying JDBC
 file_fdw     | 1.0             |                   | foreign-data wrapper for flat file access
(4 rows)
```
3) Create a server that uses jdbc_fdw.
### Make sure you cp [fmjdbc.jar](https://support.filemaker.com/s/answerview?anum=12921) /opt/local/libpostgresql/

```
curl -kLO fmdl.filemaker.com/UPDT/13/FM13.0v1_xDBC_13.0.12.dmg
curl -kLO fmdl.filemaker.com/UPDT/13/FM13.0v2_xDBC_13.0.237.dmg
curl -kLO fmdl.filemaker.com/UPDT/13/FM13.0v5_xDBC_13.2.14.dmg
curl -kLO fmdl.filemaker.com/UPDT/14/FM14.0v1_xDBC_14.0.10.dmg
curl -kLO fmdl.filemaker.com/extras/FM15_xDBC_15.0.1.dmg
curl -kLO fmdl.filemaker.com/extras/FM15_xDBC_15.0.2.dmg
curl -kLO fmdl.filemaker.com/extras/FM15_xDBC_15.0.3.dmg
curl -kLO fmdl.filemaker.com/extras/FM16_xDBC_16.0.1.dmg
curl -kLO fmdl.filemaker.com/extras/FM17_xDBC_17.0.1.dmg
```

### Command to set up server that uses jdbc_fdw as the foreign data wrapper:
``` 
CREATE
  SERVER fm.jdbc_fm16s
  FOREIGN DATA WRAPPER jdbc_fdw
  OPTIONS(
    drivername 'com.filemaker.jdbc.Driver',
    url 'jdbc:filemaker://fmsa.homestead.lan/filemakerfilename',
    jarfile '/opt/local/lib/postgresql/fmjdbc.jar'
  );
```

### Explanation of setting up options for the server
```drivername``` : The drivername has to be given the value of the exact class name 
which has to be loaded for the JDBC driver e.g. org.postgresql.Driver

```url``` : The url has to be given the value of the JDBC URL of the database from 
which the data has to fetched by jdbc_fdw into PostgreSQL.

```jarfile``` : The jarfile has to be given the value of the path and name of the JAR 
file of the JDBC driver of the database from which the data has to fetched.

```querytimeout``` : The time after which a query will be terminated automatically.
This can be used for terminating hung queries.

```maxheapsize``` : The value of the option shall be set to the maximum heap size of 
the JVM which is being used in jdbc fdw. It can be set from 1 Mb onwards.
This option is used for setting the maximum heap size of the JVM manually.

### Important : Please note that setting the maximum heap size of the JVM 
manually can lead to decrease in performance of jdbc_fdw.It is recommended that
you double check the value you are setting in maxheapsize.It is not a 
compulsory option i.e. If you do net set any value for maxheapsize,the default 
value for the maximum heap size of the JVM being used in jdbc_fdw shall be used.
Please use it only if you want to set the maximum heap size of the JVM manually.
Setting it to a very low value can lead to drastic performance hit.
Also,since the JVM being used in jdbc fdw is created only once for the entire 
psql session,therefore,the first query issued that uses jdbc fdw shall set the
value of maximum heap size of the JVM(if the first query specifies a maximum heap value).

4) Create a user mapping for the server( pgUsername is any exisiting pg user for mapping to an existing fmUser with password )
```
CREATE USER MAPPING FOR pgUsername SERVER jdbc_fm16s OPTIONS(username 'fmUsername', password 'fmSecret');
```
5) Create a foreign table on the server.
```
CREATE FOREIGN TABLE fm16s.test1( a text, b text, c text ) SERVER jdbc_fm16s OPTIONS( table 'fmTableName' );
```

alternatively ```query``` can replace ```table``` as some sort of VIEW; primarily to avoid pulling all the data from FileMaker to Postgres as LIMIT will never be remotely; however an FQL query can be run pr as below. Even though you most likely would sned a timestamp from last rather than than a static LIMIT of 5.

Note that you are never really interested in running the query bareback as in the above utilizing ```table``` if there are more than 10 records in that table; as the query above will not be run remotely, postgres will pull down all the records from the FileMaker side and then do LIMIT 5. What you really want to do is design the ```query``` below in the optimum way for your solution to have some kind of performance.

```
CREATE FOREIGN TABLE fm16s.test2( a text, b text, c text ) SERVER jdbc_fm16s OPTIONS(
  query 'SELECT * FROM fmTable ORDER BY modificationTimestamp DESC FETCH FIRST 5 ROWS ONLY'
);
```

# Testing the configuration <a name="testing"></a>
1) Query the foreign table.
```
yourdesireddatabase=# SELECT * FROM fm.fm16s00users LIMIT 5;
   a   | b  | c
-------+----+----
 80    |    |
 14000 |    |
 17120 |    |
 25000 |    |
 22    | 27 | 16
(5 rows)
```
