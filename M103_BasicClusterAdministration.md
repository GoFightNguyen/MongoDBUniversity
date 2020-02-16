# Chapter 0: Introduction & Setup
This course used Vagrant and Virtual Box.
Turned off Hyper-v.
ran as admin: `bcdedit /set hypervisorlaunchtype off`

Some useful commands:
- vagrant up
- vagrant ssh
- vagrant halt
- vagrant provision
- vagrant status

If Vagrant prompts for username/password, it is vagrant/vagrant.

# Chapter 1: The Mongod
mongod is the main daemon process for MongoDB.
It is the core unit of the database, handling connections, requests and persisting data.

A daemon is a program/process meant to be run but not interacted with in a direct manner.

mongod is most easily started by running the command `mongod`.
Some defaults:
- port: 27017
- dbpath: /data/db

The command `mongo` connects you into the mongo shell you so you can interact.
Example `show dbs`.

To shut down the server and exit the mongo shell:
1. use the admin db: `use admin`
2. run the shutdown command: `db.shutdownServer()`
  - this will close/shutdown the `mongod` process
3. exit the shell: `exit`

Another way to shutdown the server: `mongo admin --eval 'db.shutdownServer()'`

To configure the `mongod` (there are more than we list here, `mongod help`):
- `--port`: the port mongod should listen on
- `--dbpath`: the location to store database files
- `--logpath`: the location for logging (the default is stdout)
- `--fork`: tells mongod to run as a background process, rather than as an active process which blocks the shell
  - if specifying, then you must specify `--logpath` too
  - fork does not work on Windows
  - outputs the process id

Another way to find the pid of the mongod process: `ps -ef | grep mongod`

## Configuration File
The MongoDB configuration file is a way to organize the options you want when running the mongod process.
It is a YAML file.

[mongod config file options](https://docs.mongodb.com/manual/reference/configuration-options/)

`mongod --config "/etc/mongod.conf"` or `mongod -f "/etc/mongod.conf"`

```yaml
storage:
    dbPath: "/data/db"

systemLog:
    path: "/data/log/mongod.log"
    destination: "file"

net:
    bindIp: "127.0.0.1,192.168.0.10"
    ssl:
        mode: "requireSSL"
        PEMKeyFile: "/etc/ssl/ssl.pem"
        CAFile: "/ect/ssl/SSLCA.pem"

processManagement:
    fork: true
```

## File Structure
- `diagnostic.data` and `log` files assist support in diagnostics
- do not modify files or folders in the MongoDB data directory (`dbPath`, defaults to /data/db)
- defer to MongoDB support or documentation for instructions on interacting with these files

## Basic Commands
Shell helpers (methods in the mongodb shell wrapping underlying db commands):
- `db.<method>()` methods wrapping commands interacting with the db
  - also has the additional collection extension `db.<collection>.<method>()`
- `rs.<method>()` methods wrapping commands controlling replica set deployment and management
- `sh.<method>()` methods wrapping commands controlling sharded cluster deployment and management

Some useful database commands:
- db.serverStatus()
- db.runCommand({<command>})
  - can be used instead of something like db.collection.createIndex, but it will be more verbose
- db.commandHelp("<command>")

## Logging Basics
Process Log - displays activity on the MongoDB instance
  - collects activity into one of the following components, each of which has an associated verbosity level
    - ACCESS: messages related to access control
    - COMMAND: messages related to database commands
    - CONTROL: messages related to control activities, such as initialization
    - FTDC: messages related to the diagnostic data collection mechanism
    - GEO: messages related to the parsing of geospatial shapes
    - INDEX: messages related to indexing operations
    - NETWORK: messages related to network activities, such as accepting connections
    - QUERY: messages related to queries, including query planner activities
    - REPL: messages related to replica sets, such as initial sync or heartbeats
    - REPL_HB
    - ROLLBACK
    - SHARDING
    - STORAGE
    - JOURNAL
    - WRITE
  - `db.getLogComponents()` outputs the process log configuration
    - includes default verbosity
    - -1: inherit from parent
    - 0: default verbosity, to include information messages
    - 1-5: increases the verbosity level to include debug messages
  - `db.adminCommand({"getLog": "global"})`
  - `db.setLogLevel(0, "INDEX")`
  - `tail -f /var/log/mongodb/mongod.log`
  - Log message severity levels:
    - F fatal
    - E error
    - W warning
    - I informational
    - D debug

By default, logs are written to standard output.
But you can configure them to be written somewhere else (doing so means you can fork the mongod process)

## Profiling the Database
Profilers are enabled at the database level.
Therefore, the operations on each database are separated.
When enabled, the profiler stores info in the `system.profile` collection.
It captures info on CRUD, admin operations, and config operations.
There are three levels:
- 0: default, profiler is off
- 1: collects data for operations taking longer than the value of slowms
  - By default, any operation taking longer than 100ms is considered slow
  - can specify the value you want by setting slowms
- 2: collects data for all operations

- `db.getProfilingLevel()`
- `db.setProfilingLevel(1)` or `db.setProfilingLevel(1, {slowms: 50})`
- `db.system.profile.find().pretty()` is one way to look at the profile data

## Basic Security
Supported authentication mechanisms for client:
- SCRAM (Salted Challenge Response Authentication Mechanism)
  - default
  - available in community/enterprise edition
- X.509
  - available in community/enterprise edition
  - uses cert for authentication
- LDAP (MongoDB enterprise only)
- KERBEROS (MongoDB enterprise only)

Uses RBACs (Role Based Access Control):
- each user has 1+ roles
- each role has 1+ privileges
- a privilege represents a groups of actions and the resources those actions apply to

### Localhost Exception
When creating a MongoDB cluster, it does not automatically include users.
The Localhost Exception allows you to access a MongoDB server enforcing authentication but not yet having a configured user for you to authenticate with.
You must run mongo shell from the __same host__ running the MongoDB server.
The localhost exception closes after you create your first user.
So __always__ create a user with admin privileges first.

## Built-in Roles
The following are categories of roles followed by some of the actual roles.
There are roles existing at a per-db level, and some at an any db level such as readAnyDatabase:
- database user: read, readWrite, readAnyDatabase
- database admin: dbAdmin, userAdmin, dbOwner, dbAdminAnyDatabase
- cluster admin: clusterAdmin, clusterManager, clusterMonitor, hostManager
- backup/restore: backup, restore
- super user: root

```javascript
// add a dbAdmin
db.createUser({
    user: 'dba',
    pwd:'c1lynd3rs',
    roles: [{db:'admin',role:'dbAdmin'}]})

// add another role
db.grantRolesToUser('dba',[{db:'playground', role: 'dbOwner'}])
```

## Server Tools
The tools you get when downloading the MongoDB package:
- mongod: core database process
- mongo: interactive, mongodb shell for connecting to mongod
- mongostat: provide stats on a running mongod process `mongostat --port 27000`
  - returns stats every second by default
- mongodump and mongorestore: used to export/import dump files from a collection
  - dump files are in BSON
  - must authenticate
  - `mongodump --port 27000 -u "m103-admin" -p "m103-pass" --authenticationDatabase "admin" --db exampleDB --collection students`
  - `mongorestore --drop --port 27000 -u "m103-admin" -p "m103-pass"   --authenticationDatabase "admin" dump/`
- mongoexport and mongoimport: similr to mongodump and mongorestore, but with JSON files
  - `mongoexport --port 27000 -u "m103-admin" -p "m103-pass"   --authenticationDatabase "admin" --db exampleDB --collection students`
    - mongoexport does not create a directory for the dump like mongodump; instead it writes to stdout
    - specify `-o students.json` to write to a file
  - `mongoimport --port 27000 -u "m103-admin" -p "m103-pass"   --authenticationDatabase "admin" students.json`
    - defaults to using the test database and a collection named after the json file

There are others: `find /usr/bin/ -name 'mongo*'`