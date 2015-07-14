#CodeChecker Userguide

##CodeChecker usage
First of all, you have to setup the environment for CodeChecker.
Codechecker server uses PostgreSQL database to store the results which is also packed into the package.

~~~~~~~~~~~~~~~~~~~~~
cd $CODECHECKER_PACKAGE_ROOT/init
source init.sh
~~~~~~~~~~~~~~~~~~~~~

After sourcing the init script the 'CodeChecker --help' command should be available.


The next step is to start the CodeChecker main script.
The main script can be started with different options.

~~~~~~~~~~~~~~~~~~~~~
usage: CodeChecker.py [-h] {check,log,checkers,server,cmd,debug} ...

Run the codechecker script.

positional arguments:
  {check,log,checkers,server,cmd,debug}
                        commands
    check               Run codechecker for a project.
    log                 Build the project and only create a log file (no
                        checking).
    checkers            List available checkers.
    server              Start the codechecker database server.
    cmd                 Command line client
    debug               Create debug logs for failed actions

optional arguments:
  -h, --help            show this help message and exit
~~~~~~~~~~~~~~~~~~~~~


##Default configuration:

Used ports:
* 8764  - PostgreSql
* 11444 - CodeChecker result viewer

## 1. log mode:
Just build your project and create a log file but do not invoke the source code analysis.

~~~~~~~~~~~~~~~~~~~~~
$CodeChecker log --help
usage: CodeChecker.py log [-h] -o LOGFILE -b COMMAND

optional arguments:
  -h, --help            show this help message and exit
  -o LOGFILE, --output LOGFILE
                        Path to the log file.
  -b COMMAND, --build COMMAND
                        Build command.
~~~~~~~~~~~~~~~~~~~~~

You can change the compilers that should be logged.
Set CC_LOGGER_GCC_LIKE environment variable to a colon separated list.
For example (default):
~~~~~~~~~~~~~~~~~~~~~
export CC_LOGGER_GCC_LIKE="gcc:g++:clang"
~~~~~~~~~~~~~~~~~~~~~

Example:
~~~~~~~~~~~~~~~~~~~~~
CodeChecker log -o ../codechecker_myProject_build.log -b "make -j2"
~~~~~~~~~~~~~~~~~~~~~

Note:
In case you want to analyze your whole project, do not forget to clean your build tree before logging.

## 2. check mode:

### 2.1 basic usage
Database and connections will be automatically configured.
The main script starts and setups everything what is required for analyzing a project (database server, tables ...).

~~~~~~~~~~~~~~~~~~~~~
CodeChecker check -w codechecker_workspace -n myTestProject -b "make"
~~~~~~~~~~~~~~~~~~~~~

Static analysis can be started also by using an already generated buildlog (see log mode).
If log is not available the analyzer will automatically create it.
An already created CMake json compilation database can be used as well.

~~~~~~~~~~~~~~~~~~~~~
CodeChecker check -w ~/codechecker_wp -n myProject -l ~/codechecker_wp/build_log.json
~~~~~~~~~~~~~~~~~~~~~


### 2.2 advanced usage

The codechecker server can be started separately when desired.
In that case multiple clients can use the same database to store new results or view old ones.


### 2.2.1 Codechecker server and database on the same machine
Codechecker server and the database are running on the same machine but the database server is started manually.
In this case the database handler and the database can be started manually by running the server command.
The workspace needs to be provided for both the server and the check commands.

~~~~~~~~~~~~~~~~~~~~~
CodeChecker server -w ~/codechecker_wp --dbname myProjectdb --dbport 8764 --dbaddress localhost --view-port 11443
~~~~~~~~~~~~~~~~~~~~~

The checking process can be started separately on the same machine
~~~~~~~~~~~~~~~~~~~~~
CodeChecker check  -w ~/codechecker_wp -n myProject -b "make -j 4" --dbname myProjectdb --dbaddress localhost --dbport 8764
~~~~~~~~~~~~~~~~~~~~~

or on a different machine
~~~~~~~~~~~~~~~~~~~~~
CodeChecker check  -w ~/codechecker_wp -n myProject -b "make -j 4" --dbname myProjectdb --dbaddress 192.168.1.1 --dbport 8764
~~~~~~~~~~~~~~~~~~~~~


### 2.2.2 Codechecker server and database are on different machines
It is possible that the codechecker server and the PostgreSQL database that contains the analysis results are on different machines. To setup PostgreSQL see later section.

In this case the codechecker server can be started using the following command:
~~~~~~~~~~~~~~~~~~~~~
CodeChecker server --dbname myProjectdb --dbport 8764 --dbaddress 192.168.1.2 --view-port 11443
~~~~~~~~~~~~~~~~~~~~~
Start codechecker server locally which connects to a remote database (which is started separately). Workspace is not required in this case.


Start the checking as explained previously.
~~~~~~~~~~~~~~~~~~~~~
CodeChecker check -w ~/codechecker_wp -n myProject -b "make -j 4" --dbname myProjectdb --dbaddress 192.168.1.2 --dbport 8764
~~~~~~~~~~~~~~~~~~~~~


## 3. checkers mode:

List all available checkers.

~~~~~~~~~~~~~~~~~~~~~
CodeChecker checkers
~~~~~~~~~~~~~~~~~~~~~


## 4. cmd mode:

A lightweigh command line interface to query the results of an analysis.
It is a suitable client to integrate with continous integration, schedule maintenance tasks and verifying correct analysis process.
The commands always need a viewer port of an already running CodeChecker server instance (which can be started using CodeChecker server command).

~~~~~~~~~~~~~~~~~~~~~
usage: CodeChecker.py cmd [-h] {runs,results,sum,del} ...

positional arguments:
  {runs,results,sum,del}
    runs                Get the run data.
    results             List results.
    sum                 Sum results.
    del                 Remove run results.

optional arguments:
  -h, --help            show this help message and exit
~~~~~~~~~~~~~~~~~~~~~

## 5. debug mode:

In debug mode CodeChecker can generate logs for failed build actions. The logs can be helpful debugging the checkers.

### Suppress file:

Suppress file can contain bug hashes and comments.
Suppressed bugs will not be showed in the viewer by default.
Usually a reason to suppress a bug is a false positive result (reporting a non-existent bug). Such false positives should be reported, so we can fix the checkers.
A comment can be added to suppressed reports that describes why that report is false positive. You should not edit suppress file by hand. The server should handle it.
The suppress file can be checked into the source code repository.
Bugs can be suppressed on the viewer even when suppress file was not set by command line arguments. This case the suppress will not be permanent. For this reason it is
advised to always provide (the same) suppress file for the checks.

### Skip file:
Paths and source files which will not be checked.

For example:
~~~~~~~~~~~~~~~~~~~~~
/skip/all/source/in/path
/do/not/check/this.file
~~~~~~~~~~~~~~~~~~~~~


## Example Usage
Checking with some extra checkers disabled and enabled

~~~~~~~~~~~~~~~~~~~~~
CodeChecker check -w ~/Develop/workspace -j 4 -b "cd ~/Libraries/myproject && make clean && make -j4" -s ~/Develop/skip.list -u ~/Develop/suppress.txt -e unix.Malloc -d core.uninitialized.Branch  -n MyLittleProject -c --dbport 9999 --dbname cctestdb
~~~~~~~~~~~~~~~~~~~~~

## View results

To view the results CodeChecker sever needs to be started.

~~~~~~~~~~~~~~~~~~~~~
CodeChecker server -w ~/codes/package/checker_ws/ --dbport 8764 --dbaddress localhost --view-port 11443
~~~~~~~~~~~~~~~~~~~~~

After the server has started open the outputed link to the browser (localhost:11443 in this example).

~~~~~~~~~~~~~~~~~~~~~
[11318] - WARNING! No suppress file was given, suppressed results will be only stored in the database.
[11318] - Checking for database
[11318] - Database is not running yet
[11318] - Starting database
[11318] - Waiting for client requests on [localhost:11443]
~~~~~~~~~~~~~~~~~~~~~

## Separate machine for the database
Create and start an empty database to which the codechecker server can connect

### Setup PostgreSQL (one time only)

This step is optional. You only need to set up PostgresSQL when you want to manage the database separately from CodeChecker.
If you skip this step, CodeChecker will manage Postgres itself.
Before the first use, you have to setup PostgreSQL.
PostgreSQL stores its data files in a data directory, so before you start the PostgreSQL server you have to create and init this data directory.
I will call the data directory to pgsql_data.

Do the following steps:
~~~~~~~~~~~~~~~~~~~~~
mkdir -p /path/to/pgsql_data
initdb -U codechecker -D /path/to/pgsql_data -E "SQL_ASCII"
~~~~~~~~~~~~~~~~~~~~~

### Starting PostgreSQL server
You can run the server with the following command:

~~~~~~~~~~~~~~~~~~~~~
postgres -U codechecker -D /path/to/pgsql_data -p 8764 &>pgsql_log &
~~~~~~~~~~~~~~~~~~~~~

### PostgreSQL authentication
If a CodeChecker is run with a user that is added later to the database authentication is required.
PGPASSFILE environment variable should be set to a pgpass file
For format and further information see PostgreSQL documentation:
http://www.postgresql.org/docs/current/static/libpq-pgpass.html

Debugging CodeChecker
-------------
Environment variables can be used to turn on CodeChecker debug mode.

Turn on CodeChecker debug level logging
~~~~~~~~~~~~~~~~~~~~~
export CODECHECKER_VERBOSE=debug
~~~~~~~~~~~~~~~~~~~~~

Turn on SQL_ALCHEMY debug level logging
~~~~~~~~~~~~~~~~~~~~~
export CODECHECKER_ALCHEMY_LOG=True
~~~~~~~~~~~~~~~~~~~~~