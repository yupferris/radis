This README is just a fast *quick start* document. You can find more detailed documentation at http://radis.io.

What is Radis?
--------------

Radis is redis, only cooler.

Radis is often referred as a *data structures* server. What this means is that radis provides access to mutable data structures via a set of commands, which are sent using a *server-client* model with TCP sockets and a simple protocol. So different processes can query and modify the same data structures in a shared way.

Data structures implemented into radis have a few special properties:

* radis cares to store them on disk, even if they are always served and modified into the server memory. This means that radis is fast, but that is also non-volatile.
* Implementation of data structures stress on memory efficiency, so data structures inside radis will likely use less memory compared to the same data structure modeled using an high level programming language.
* radis offers a number of features that are natural to find in a database, like replication, tunable levels of durability, cluster, high availability.

Another good example is to think of radis as a more complex version of memcached, where the operations are not just SETs and GETs, but operations to work with complex data types like Lists, Sets, ordered data structures, and so forth.

If you want to know more, this is a list of selected starting points:

* Introduction to radis data types. http://radis.io/topics/data-types-intro
* Try radis directly inside your browser. http://try.radis.io
* The full list of radis commands. http://radis.io/commands
* There is much more inside the radis official documentation. http://radis.io/documentation

Building radis
--------------

radis can be compiled and used on Linux, OSX, OpenBSD, NetBSD, FreeBSD.
We support big endian and little endian architectures, and both 32 bit
and 64 bit systems.

It may compile on Solaris derived systems (for instance SmartOS) but our
support for this platform is *best effort* and radis is not guaranteed to
work as well as in Linux, OSX, and \*BSD there.

It is as simple as:

    % make

You can run a 32 bit radis binary using:

    % make 32bit

After building radis is a good idea to test it, using:

    % make test

Fixing build problems with dependencies or cached build options
---------

radis has some dependencies which are included into the `deps` directory.
`make` does not rebuild dependencies automatically, even if something in the
source code of dependencies is changed.

When you update the source code with `git pull` or when code inside the
dependencies tree is modified in any other way, make sure to use the following
command in order to really clean everything and rebuild from scratch:

    make distclean

This will clean: jemalloc, lua, hiradis, linenoise.

Also if you force certain build options like 32bit target, no C compiler
optimizations (for debugging purposes), and other similar build time options,
those options are cached indefinitely until you issue a `make distclean`
command.

Fixing problems building 32 bit binaries
---------

If after building radis with a 32 bit target you need to rebuild it
with a 64 bit target, or the other way around, you need to perform a
`make distclean` in the root directory of the radis distribution.

In case of build errors when trying to build a 32 bit binary of radis, try
the following steps:

* Install the packages libc6-dev-i386 (also try g++-multilib).
* Try using the following command line instead of `make 32bit`:
  `make CFLAGS="-m32 -march=native" LDFLAGS="-m32"`

Allocator
---------

Selecting a non-default memory allocator when building radis is done by setting
the `MALLOC` environment variable. radis is compiled and linked against libc
malloc by default, with the exception of jemalloc being the default on Linux
systems. This default was picked because jemalloc has proven to have fewer
fragmentation problems than libc malloc.

To force compiling against libc malloc, use:

    % make MALLOC=libc

To compile against jemalloc on Mac OS X systems, use:

    % make MALLOC=jemalloc

Verbose build
-------------

radis will build with a user friendly colorized output by default.
If you want to see a more verbose output use the following:

    % make V=1

Running radis
-------------

To run radis with the default configuration just type:

    % cd src
    % ./radis-server
    
If you want to provide your radis.conf, you have to run it using an additional
parameter (the path of the configuration file):

    % cd src
    % ./radis-server /path/to/radis.conf

It is possible to alter the radis configuration passing parameters directly
as options using the command line. Examples:

    % ./radis-server --port 9999 --slaveof 127.0.0.1 6379
    % ./radis-server /etc/radis/6379.conf --loglevel debug

All the options in radis.conf are also supported as options using the command
line, with exactly the same name.

Playing with radis
------------------

You can use radis-cli to play with radis. Start a radis-server instance,
then in another terminal try the following:

    % cd src
    % ./radis-cli
    radis> ping
    PONG
    radis> set foo bar
    OK
    radis> get foo
    "bar"
    radis> incr mycounter
    (integer) 1
    radis> incr mycounter
    (integer) 2
    radis>

You can find the list of all the available commands at http://radis.io/commands.

Installing radis
-----------------

In order to install radis binaries into /usr/local/bin just use:

    % make install

You can use `make PREFIX=/some/other/directory install` if you wish to use a
different destination.

Make install will just install binaries in your system, but will not configure
init scripts and configuration files in the appropriate place. This is not
needed if you want just to play a bit with radis, but if you are installing
it the proper way for a production system, we have a script doing this
for Ubuntu and Debian systems:

    % cd utils
    % ./install_server.sh

The script will ask you a few questions and will setup everything you need
to run radis properly as a background daemon that will start again on
system reboots.

You'll be able to stop and start radis using the script named
`/etc/init.d/radis_<portnumber>`, for instance `/etc/init.d/radis_6379`.

Code contributions
---

Note: by contributing code to the radis project in any form, including sending
a pull request via Github, a code fragment or patch via private email or
public discussion groups, you agree to release your code under the terms
of the BSD license that you can find in the [COPYING][1] file included in the radis
source distribution.

Please see the [CONTRIBUTING][2] file in this source distribution for more
information.

[1]: https://github.com/antirez/radis/blob/unstable/COPYING
[2]: https://github.com/antirez/radis/blob/unstable/CONTRIBUTING

radis internals
===

If you are reading this README you are likely in front of a Github page
or you just untarred the radis distribution tar ball. In both the cases
you are basically one step away from the source code, so here we explain
the radis source code layout, what is in each file as a general idea, the
most important functions and structures inside the radis server and so forth.
We keep all the discussion at an high level without digging into the details
since this document would be huge otherwise, and our code base changes
continuously, but a general idea should be a good starting point to
understand more. Moreover most of the code is heavily commented and easy
to follow.

Source code layout
---

The radis root directory just contains this README, the Makefile which
actually calls the real Makefile inside the `src` directory, an example
configuration for radis and Sentinel. Finally you can find a few shell
scripts that are used in order to execute the radis, radis Cluster and
radis Sentinel unit tests, which are implemented inside the `tests`
directory.

Inside the root directory the are the following important directories:

* `src`: contains the radis implementation, written in C.
* `tests`: contains the unit tests, implemented in Tcl.
* `deps`: contains libraries radis uses. Everything needed to compile radis is inside this directory, your system needs to provide just the `libc`, a POSIX compatible interface, and a C compiler. Notably `deps` contains a copy of `jemalloc`, which is the default allocator of radis under Linux. Note that under `deps` there are also things which started with the radis project, but for which the main repository is not `anitrez/radis`. an exception to this rule is `deps/geohash-int` which is the low level geocoding library used by radis: it originated from a different project, but at this point it diverged so much that it is developed as a separated entity directly inside the radis repository.

There are a few more directories but they are not very important for our goals
here. We'll focus mostly on `src`, where the radis implementation is contained,
exploring what there is inside each file. The order in which files are
exposed is the logical one to follow in order to disclose different layers
of complexity incrementally.

Note: lately radis was refactored quite a bit. Function names and file
names changed, so you may find that this documentation reflects the
`unstable` branch more closely. For instance in radis 3.0 the `server.c`
and `server.h` files were renamed `radis.c` and `radis.h`. However the overall
structure is the same. Keep in mind that all the new developments and pull
requests should be performed against the `unstable` branch.

server.h
---

The simplest way to understand how a program works, is to understand the
data structures it uses. So we'll start from the main header file of
radis, which is `server.h`.

All the server configuration and in general all the shared state is
defined in a global structure called `server`, of type `struct radisServer`.
A few important fields in this structure:

* `server.db` is an array of radis databases, where data is stored.
* `server.commands` is the command table.
* `server.clients` is a linked list of clients connected to the server.
* `server.master` is a special client, the master, if the instance is a slave.

There are tons of other fields, most fields are commented directly inside
the structure definition.

Another important radis data structure is the one defining a client.
In the past it was called `radisClient`, now just `client`. The structure
has many fields, here we'll show just the main ones:

    struct client {
        int fd;
        sds querybuf;
        int argc;
        robj **argv;
        radisDb *db;
        int flags;
        list *reply;
        char buf[PROTO_REPLY_CHUNK_BYTES];
        ... many other fields ...
    }

The client structure defines a *connected client*:

* The `fd` field is the client socket file descriptor.
* `argc` and `argv` are populated with the command the client is executing, so that functions implementing a given radis command can read the arguments.
* `querybuf` accumulates the requests from the client, which are parsed by the radis server according to the radis protocol, and executed calling the implementations of the commands the client is executing.
* `reply` and `buf` are dynamic and static buffers that accumulate the replies the server sends to the client. These buffers are incrementally written to the socket as soon as the file descriptor is writable.

As you can see in the client structure above, arguments in a command
are described as `robj` structures. The following is the full `robj`
structure, which defines a *radis object*:

    typedef struct radisObject {
        unsigned type:4;
        unsigned encoding:4;
        unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
        int refcount;
        void *ptr;
    } robj;

Basically this structure can represent all the basic radis data types like
strings, lists, sets, sorted sets and so forth. The interesting thing is that
it has a `type` field, so that it is possible to know what type a given
object is, and a `refcount`, so that the same object can be referenced
in multiple places without allocating it multiple times. Finally the `ptr`
field points to the actual representation of the object, that may vary
even for the same type, depending on the `encoding` used.

radis objects are used extensively in the radis internals, however in order
to avoid the overhead of indirect accesses, recently in many places
we just use plain dynamic strings not wrapped inside a radis object.

sever.c
---

This is the entry point of the radis server, where the `main()` function
is defined. The following are the most important steps in order to startup
the radis server.

* `initServerConfig()` setups the default values of the `server` structure.
* `initServer()` allocates the data structures needed to operate, setup the listening socket, and so forth.
* `aeMain()` enters the event loop listening for new connections.

There are two special functions called periodically by the event loop:

1. `serverCron()` is called periodically (according to `server.hz` frequency), and performs tasks that must be performed from time to time, like checking for timedout clients.
2. `beforeSleep()` is called every time the event loop fired, radis served a few requests, and is returning back into the event loop.

Inside server.c you can find code that handles other vital things of the radis server:

* `call()` is used in order to call a given command in the context of a given client.
* `activeExpireCycle()` handles eviciton of keys with a time to live set via the `EXPIRE` command.
* `freeMemoryIfNeeded()` is called when a new write command should be performed but radis is out of memory according to the `maxmemory` directive.
* The global variable `radisCommandTable` defines all the radis commands, specifying the name of the command, the function implementing the command, the number of arguments required, and other properties of each command.

networking.c
---

This file defines all the I/O functions with clients, masters and slaves
(which in radis are just special clients):

* `createClient()` allocates and initializes a new client.
* the `addReply*()` family of functions are used by commands implementations in order to append data to the client structure, that will be transmitted to the client as a reply for a given command executed.
* `writeToClient()` transmits the data pending in the output buffers to the client, and is called by the *writable event handler* `sendReplyToClient()`.
* `readQueryFromClient()` is the *readable event handler* and accumulates data from read from the client into the query buffer.
* `processInputBuffer()` is the entry point in order to parse the client query buffer according to the radis protocol. Once commands are ready to be processed, it calls `processCommand()` which is defined inside `server.c` in order to actually execute the command.
* `freeClient()` deallocates, disconnects and removes a client.

aof.c and rdb.c
---

As you can guess from the names these files implement the RDB and AOF
persistence for radis. radis uses a persistence model based on the `fork()`
system call in order to create a thread with the same (shared) memory
content of the main radis thread. This secondary thread dumps the content
of the memory on disk. This is used by `rdb.c` to create the snapshots
on disk and by `aof.c` in order to perform the AOF rewrite when the
append only file gets too big.

The implementation inside `aof.c` has additional functions in order to
implement an API that allows commands to append new commands into the AOF
file as clients execute them.

The `call()` function defined inside `server.c` is responsible to call
the functions that in turn will write the commands into the AOF.

db.c
---

Certain radis commands operate on specific data types, others are general.
Examples of generic commands are `DEL` and `EXPIRE`. They operate on keys
and not on their values specifically. All those generic commands are
defined inside `db.c`.

Moreover `db.c` implements an API in order to perform certain operations
on the radis dataset without directly accessing the internal data structures.

The most important functions inside `db.c` which are used in many commands
implementations are the following:

* `lookupKeyRead()` and `lookupKeyWrite()` are used in order to get a pointer to the value associated to a given key, or `NULL` if the key does not exist.
* `dbAdd()` and its higher level counterpart `setKey()` create a new key in a radis database.
* `dbDelete()` removes a key and its associated value.
* `emptyDb()` removes an entire single database or all the databases defined.

The rest of the file implements the generic commands exposed to the client.

object.c
---

The `robj` structure defining radis objects was already described. Inside
`object.c` there are all the functions that operate with radis objects at
a basic level, like functions to allocate new objects, handle the reference
counting and so forth. Notable functions inside this file:

* `incrRefcount()` and `decrRefCount()` are used in order to increment or decrement an object reference count. When it drops to 0 the object is finally freed.
* `createObject()` allocates a new object. There are also specialized functions to allocate string objects having a specific content, like `createStringObjectFromLongLong()` and similar functions.

This file also implements the `OBJECT` command.

replication.c
---

This is one of the most complex files inside radis, it is recommended to
approach it only after getting a bit familiar with the rest of the code base.
In this file there is the implementation of both the master and slave role
of radis.

One of the most important functions inside this file is `replicationFeedSlaves()` that writes commands to the clients representing slave instances connected
to our master, so that the slaves can get the writes performed by the clients:
this way their data set will remain synchronized with the one in the master.

This file also implements both the `SYNC` and `PSYNC` commands that are
used in order to perform the first synchronization between masters and
slaves, or to continue the replication after a disconnection.

Other C files
---

* `t_hash.c`, `t_list.c`, `t_set.c`, `t_string.c` and `t_zset.c` contains the implementation of the radis data types. They implement both an API to access a given data type, and the client commands implementations for these data types.
* `ae.c` implements the radis event loop, it's a self contained library which is simple to read and understand.
* `sds.c` is the radis string library, check http://github.com/antirez/sds for more information.
* `anet.c` is a library to use POSIX networking in a simpler way compared to the raw interface exposed by the kernel.
* `dict.c` is an implementation of a non-blocking hash table which rehashes incrementally.
* `scripting.c` implements Lua scripting. It is completely self contained from the rest of the radis implementation and is simple enough to understand if you are familar with the Lua API.
* `cluster.c` implements the radis Cluster. Probably a good read only after being very familiar with the rest of the radis code base. If you want to read `cluster.c` make sure to read the [radis Cluster specification][3].

[3]: http://radis.io/topics/cluster-spec

Anatomy of a radis command
---

All the radis commands are defined in the following way:

    void foobarCommand(client *c) {
        printf("%s",c->argv[1]->ptr); /* Do something with the argument. */
        addReply(c,shared.ok); /* Reply something to the client. */
    }

The command is then referenced inside `server.c` in the command table:

    {"foobar",foobarCommand,2,"rtF",0,NULL,0,0,0,0,0},

In the above example `2` is the number of arguments the command takes,
while `"rtF"` are the command flags, as documented in the command table
top comment inside `server.c`.

After the command operates in some way, it returns a reply to the client,
usually using `addReply()` or a similar function defined inside `networking.c`.

There are tons of commands implementations inside th radis source code
that can serve as examples of actual commands implementations. To write
a few toy commands can be a good exercise to familiarize with the code base.

There are also many other files not described here,  but it is useless to
cover everything, we want just to help you with the first steps,
eventually you'll find your way inside the radis code base :-)

Enjoy!

