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