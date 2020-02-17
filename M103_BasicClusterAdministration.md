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

# Chapter 2: Replication
## What is Replication
MongoDB uses async statement-based replication because it is platform independent and provides more flexibility within replica set.

Replication is the concept of maintaining mutliple copies of your data.
Necessary because you can never assume that all servers/nodes will be available.
In MongoDB, a group of nodes that each have copies of the data is called a __Replica Set__.
By default, all data is handled by one of the nodes (primary), and the others (seconary) must sync with it asynchronously.

During a failover (primary goes down), the nodes decide which one becomes the primary through an __Election__.

Data replication takes 1 of 2 forms:
- Binary Replication
  - examines the exact bytes that changed in a data file
  - the bytes are recorded in a binary log
  - the seconary nodes receive a copy of the binary log and make the same exact changes
  - easy on the secondary nodes
  - assumes the OS is consistent across the entire Replica Set
- Statement-Based Replication
  - after a write is completed on the primary, the statement is stored in the Oplog
  - secondaries then sync their Oplog with the primary, and run any new statements
  - idempotent: the write commands are transformed before being placed in the Oplog so the statement can be applied numerous times

Pros & Cons of the replication approaches:
- Binary Replication
  - less data
  - faster
- Statement-Based Replication
  - not bound by OS, or any machine level dependency
  - Oplog can grow

## MongoDB Replica Set
A Replica Set is a group of mongod(s) that share copies of the same info.
In MongoDB, the asynchronous replication protocol could be:
- pv1 - default
  - based on RAFT protocol
- pv0 - used by previous versions of MongoDB
The asynchronous replication protocol affects how availability and durability are handled throughout the Replica Set.

We will only be concerned with pv1.

Oplog is a statement based log tracking all write operations.
Every time a write applies to the primary, then it is recorded in an idempotent form in the Oplog.

A Replica Set node can be configured as an Arbiter.
It holds no data.
Serves as a tie-breaker in an election.
Cannot become primary.
Avoid Arbiters, they can lead to consistency issues.

You should strive to have an odd-number of nodes, although it is possible to have an even number.
No matter the number, you must always have a majority of the nodes available.
You can have up to 50 nodes.
Only a max of 7 nodes are voting members; the voting members is where the new primary will come from during a failover or anything else triggering an election.

Secondary nodes can also be:
- hidden: provides specific read-only workloads
  - still replicate data
  - can still vote
- delayed: allow resilience to application-level corruption (this node delays its replication process)

## Setting up a Replica Set
```yaml
# because this example has all 3 nodes running on the same machine, 
# all 3 nodes will use the same key file

# node1.conf
storage:
  dbPath: /var/mongodb/db/node1
net:
  bindIp: 192.168.103.100,localhost
  port: 27011
security:
  authorization: enabled
  keyFile: /var/mongodb/pki/m103-keyfile
systemLog:
  destination: file
  path: /var/mongodb/db/node1/mongod.log
  logAppend: true
processManagement:
  fork: true
replication:
  replSetName: m103-example

# node2.conf
storage:
  dbPath: /var/mongodb/db/node2
net:
  bindIp: 192.168.103.100,localhost
  port: 27012
security:
  keyFile: /var/mongodb/pki/m103-keyfile
systemLog:
  destination: file
  path: /var/mongodb/db/node2/mongod.log
  logAppend: true
processManagement:
  fork: true
replication:
  replSetName: m103-example

# node3.conf
storage:
  dbPath: /var/mongodb/db/node3
net:
  bindIp: 192.168.103.100,localhost
  port: 27013
security:
  keyFile: /var/mongodb/pki/m103-keyfile
systemLog:
  destination: file
  path: /var/mongodb/db/node3/mongod.log
  logAppend: true
processManagement:
  fork: true
replication:
  replSetName: m103-example
```

To create a keyfile:
```sh
sudo mkdir -p /var/mongodb/pki
sudo chown vagrant:vagrant -R /var/mongodb
openssl rand -base64 741 > /var/mongodb/pki/m103-keyfile
chmod 600 /var/mongodb/pki/m103-keyfile
```

To enable communication between the nodes:
- connect to a node `mongo --port 27011`
- `rs.initiate()`
- connect the other nodes to the replica set
  - if you have authenticate enabled, you must create a user for this
  ```javascript
    use admin
    db.createUser({
    user: "m103-admin",
    pwd: "m103-pass",
    roles: [
        {role: "root", db: "admin"}
    ]
    })
  ```
  - log in as the user and connect to the replica set
  ```sh
    mongo --host "m103-example/192.168.103.100:27011" -u "m103-admin"
    -p "m103-pass" --authenticationDatabase "admin"
  ```
  - add a node `rs.add("m103:27012")`
    - m103 is the hostname, in this case our vagrant box

One way to force an election is `rs.stepDown()`

## Replication Configuration Document
A BSON document, we manage using a JSON representation.
Can be configured manually from the shell.
It is shared across all the nodes.
There are some mongo shell replication helper methods.

Here are some of the configuration options:
- _id: the name of the replica set
  - Two ways to launch a mongod belonging to a replica set
    - `mongod --replSet m103-example`
    - `mongod -f /etc/mongodb.conf`
- version: incremented every time the configuration changes
- members: an array defining the topology and roles of the individual nodes in the replica set
  - each element is a subdocument containing:
    - _id: unique identifier of each element
    - host: this is the hostname:port
    - arbiterOnly
    - hidden
    - priority: [0,1000] higher priority => more likely to be primary
      - 0 means the member can never be the primary
      - a node with priority 0 is considered passive
    - slaveDelay: determines the replication delay interval in seconds
      - setting this implies hidden is true and priority 0

## Replication Commands
`rs.status()`
Reports health on replica set nodes.
Uses data from heartbeats, which are sent between the nodes.
Could be a few seconds out-of-date since it relies on heartbeats.

`rs.isMaster()`
Describes a node's role.

`rs.serverStatus()['repl']`
Section of the db.serverStatus() output.
Similar to the output of `rs.isMaster()`.
Also includes the `rbid` field, which is the number of rollbacks that have occurred on the node.

`rs.printReplicationInfo()`
Only returns oplog data relative to the current node.
Contains timestamps for first and last oplog events.

## Local DB
All MongoDB instances start with two default databases, __admin__ and __local__.

The `local` db starts with one collection: __startup_log__.
If in a replica set, there are more collections:
- most of the collections are maintained internally by the server, and are basically just configuration data : me, startup_log, system.replset, system.rollback.id, replset.election, replset.minvalid
- oplog.rs: the central point of the replication mechanism
  - keeps track of all statements being replicated
  - it is a capped collection, there is a size limit
    - `var stats = db.oplog.rs.stats()`, `stats.maxSize()`
    - `rs.printReplicationInfo()` also shows the max size
  - by default, the collection will take 5% of free disk

The size of the oplog.rs can also be configured:
```yaml
replication:
  oplogSizeMB: 5MB
```
Once the size is reached, the oldest operations are overwritten with newer operations.
The time it takes to fully fill in the oplog is what determines the replication window.
This value determines how long a replica set can afford a node to be down without requiring human intervention during recovery.

Every node has its own oplog.
Secondary nodes apply the data from the primary into their own oplogs.
If a node fails, when it recovers it will attempt to find a common oplog entry in the other nodes so it knows how to recover.
If it cannot find a common point, then the node cannot auto recover/sync with the rest.

Nodes can have different sized oplogs.
If the primary is larger, then it means a secondary can afford to be down longer.

Therefore:
- the replication window is proportional to the system load
- one operation may result in many oplog.rs entries in order to create idempotent operations

Any data written to local db will not be replicated, except for the things written to oplog.rs

## Reconfiguring a Replica Set
You can modify a replica set as it is running.
You can add/remove nodes and change their configuration while the replica set is running.

`rs.conf()` gives a complete configuration for the replica set.
```sh
cfg = rs.conf()
cfg.members[3].votes = 0
rs.reconfig(cfg) # reconfigure a running replica set
```

## Reads and Writes on a Replica Set
If connected to a secondary, in order to even read from it you must explicitly say so using `rs.slaveOk()`.
Even doing this, you cannot write to a secondary node.

If a replica set can no longer reach a majority, all remaining nodes will become secondaries, including the primary.
During such time, you cannot do writes.

## Failovers and Elections
A rolling upgrade starts with the secondary nodes.

`rs.stepDown()` is one way to have primary step down, which will force an election.

## Write Concerns
Additional write concern options:
- wtimeout: <int>
  - the time for the client to wait for the requested write concern before marking the operation as failed
  - the write can still succeed if the timeout is exceeded
- j: <true|false>
  - requires the node to commit the write operation to the journal before returning an ack
  - w:majority => j:true
  - if j:false it means the node only has to store the data in memory before reporting success

As an example, pretend you have a 3 node replica set.
One node goes down.
You execute db.new_data.insert({"m103": "very fun"}, {writeConcern: {w: 3, wtimeout: 1000}})
- when a writeConcernError occurs, the document is still written to the healthy nodes
  - a WriteResult object will indicate whether the writeConcern was successful or not, but it will not undo successful writes from any of the nodes
- An unhealthy node will have the write applied when it is brought back online
- If wtimeout is not specified, the write operation will be retried indefinitely
  - so if the writeConcern is impossible, it may never return anything to the client

## Read Concerns
Read Concern levels:
- local: return most recent data in the cluster
  - basically, anything existing in the primary
  - default for the primary
- available (sharded clusters): same as local but applies to the secondary and for sharded clusters
- majority: provides the strongest guarantee of data, but there's a chance you do not get the freshest/latest data in the cluster if it has yet to be replicated to the majority
- linearizable: similar to majority but also means read-your-own-write functionality

| | fast | latest | safe/durability guarantee |
| --- | --- | --- | --- |
| local | x | x | |
| available | x | x | |
| majority | x | | x |
| linearizable | | x | x |

## Read Preferences
Allow you to route read operations to specific members of a replica set.
Pretty much just a driver/client side setting.

Read Preference modes:
- primary (default)
- primaryPreferred: if the primary is unavailable then a secondary can be used instead
- secondary: only to secondary members
- secondaryPreferred: if no secondaries are available then the primary can be used
- nearest: to the node with the lowest network latency to the client