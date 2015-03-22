# abroute_docker_autobahn - autobahn scripts
These are the scripts that run autobahn proof of concept
There are 3 of them, they are meant to be run in separate containers.

## Overview

There are two main scripts and one utility script.

* router, this is the main autobahn router with a couple of hooks to do authorization and authentication
* rpc, these are the main functions for doing authentication and authorization
* adm, this is simply the admin utility, like adding topics or users or roles. monitoring activity. etc.

Autobahn provides for registering functions with a name, and calling them remotely as well
as publish / subscribe functionality. This code is based upon the concept
that the names that are registered with Autobahn (topics) are logical domains. They
are dot separated subjects.  E.g. com is the logical parent of com.devices which in
turn is the logical parent of com.devices.fan.  Using this heirchy, We can map permissions
on top to control the actions we can take on the topic. The actions that can be controlled are
* register - register a function
* call - call an already registered function
* publish - publish a message to a topic
* subscribe - subscribe to a topic channel
* admin - grant these actions / permissions to others

A hook is needed in the autobahn router to check authentication and
authorization. Luckily, there is already a hook for authentication. Authorization
needs to be checked before doing each action, that is why I needed to modify the
router a little bit, to put in this authorization.  After I
wrote this code the Autobahn project removed router functionality from
the code and moved it to a different project (Crossbar). That's why
this code is on an older version of Autobahn.

The router has a dependency on the database. The database should be
listening at pg (hostname pg) port 5432. The router will in turn listen
on port 8080 for web socket connections from clients.

To start the router:
```
docker run -d -name ab tacodata/abroute_docker_autobahn router
```

The router has a companion process it needs.  I could put it in the same
process, but, I am exercising docker orchestration which makes me want to
have additional processes :-)  To start the supporting rpc:

```
docker run -d -name rpc tacodata/abroute_docker_autobahn rpc
```

The supporting rpc looks for the router at ab:8080.

The hostnames are populated by a discovery mechanism.  Docker-compose puts
these into the /etc/hosts file when starting containers based upon the links section.
Hostname can also be populated using the registrator/skydns mechanism, which
populates dns based upon containers starting/stopping.  I plan on testing
both of these methods.

## Docker Container Layout

![alt text][docker_containers]

The containers in this repo are indicated by sqlauthroute and sqlauthrpc in the
drawing. The rpc container has no public interfaces.  It is a part of the router.
The router itself has a single public interface, currently 8080, which web socket
connections are made with.  The router is stateless.  All of the state is held
in the postgres database.

## Scalability

With the version of Autobahn I am running I cannot connect routers to each other. So,
with this architecture the only thing that can be replicated is the rpc
container.  If the router is the bottleneck, then to replicate that will take
a little work to push calls and publishes through the database, and let it communicate
with the other routers.  If postgres turns out to be the bottleneck we can device a scheme
to mave one master and many replicated slaves (for read queries).

All that said, this framework is not the ultimate application.  This framework mearly
puts in place authentication and authorization for the application.  I am guessing
that the framework should handle millions of account histories, and perhaps tens of thousands of
active accounts.

[docker_containers]:https://github.com/lgfausak/sqlauth/raw/master/docs/docker_containers.png "Docker Containers"
