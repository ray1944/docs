# 1.building a simple finger service
This article uses **Factories**, **Protocols** and **Deferreds**.  
By the end of this sesion, out Finger server will answer TCP finger requests on port 1079, and will read data from the web.

# 2. Refuse Connections
If you want to observe the behavior of the server being developed. The easieast way to do this is with a telnet client.
```
telnet localhost 1079
```
will connect to the local host on port 1079.

# 3. The Reactor
Note that there are actually several different reactors to choose from.
`from twisted.internet import reactor` returns the current reactor.

## 3.1 Do nothing
```python
from twisted.internet import protocol, reactor, endpoints
class FingerProtocol(protocol.Protocol):
    pass

class FingerFactory(protocol.ServerFactory):
    protocol = FingerProtocol

fingerEndpoint = endpoints.serverFromString(reactor, "tcp:1079")
fingerEndpoint.listen(FingerFactory())
reactor.run()
```

### 3.1.1 How to choose endpoints
Basic usage of endpoints: you construct an appropriate type of server or client endpoint, and then call `listen`(servers) or `connect`(clients).
In most programs, you will want to allow the user to specify where to listen or connect, in a way which will allow the user to request different strategies, without having to adjust your program. In order to allow this, you should use `clientFromString` or `serverFromString`.

Each type of endpoint is just an interface with a single method that takes an argument. for ins.
```python
serverEndpoint.listen(factory)
```
or 
```python
clientEndpoint.connect(factory)
```
each of these APIs returns a value, Defferred.
`IStreamServerEndpoint.listen` returns a Deferred that fires with an `IListeningPort`. This deferred may errback(might be port be used by another program).

### 3.1.2 a example to demostrate relationship between endpoint, Facotry, protocol and service
```python
from twisted.internet.protocol import Factory
from twisted.internet.endpoints import clientFromString
from twisted.words.protocols.irc import IRCClient
from twisted.application.internet import ClientService
from twisted.internet import reactor

myEndpoint = clientFromString(reactor, "tls:example.com:6997") # base on second parameter string to define ssl mode  and destination client
myFactory = Factory.forProtocol(IRCClient) # by forProtocol method to get corresponding protocol Factory

myReconnectingService = ClientService(myEndpoint, myFactory) 
```
If you are writing an application and you need to construct endpoints yourself, you can allow users to specify arbitrary endpoints described **by a string** using the `clientFromString` and `serverFromString` APIs. Since these APIs just take a string, they provide flexibility: if Twisted adds support for new types of endpoints(IPv6 or WebSocket), your application will autoly be able to take advantage of them with no changes to its code.

In many use-cases, you might not want to use the endpoints APIs directly. Instead, you may want to construct an `IService`, using `strports.service`, which will fit neatly into the required structure of the twistd plugin API. 

In short, it is almost always preferable to use an endpoint rather than calling a lower-level APIs like `connectTCP`, `listenTCP`, etc, directly. By accepting an arbitrary endpoint rather than requiring a specific reactor interface, you leave your application open to lots of interesting transport-layer extensibility for the future.

A factory is the proper place for data that you want to make available to the protocol instances, since the protocol instances are garbage collected when the connection is closed.

## 3.2 full example
```python
from twisted.internet import protocol, reactor, endpoints
from twisted.protocols import basic
class FingerProtocol(basic.LineReceiver):
    def lineReceived(self, user):
        self.transport.write(b"No such user\r\n")
        self.transport.loseConnection()

class FingerFactory(protocol.ServerFactory):
    protocol = FingerProtocol

fingerEndpoint = endpoints.serverFromString(reactor, "tcp:1079")
fingerEndpoint.listen(FingerFactory())
reactor.run()
```

## 3.3 Use Deferreds
```python
# Read username, output from non-empty factory, drop connections
# Use deferreds, to minimize synchronicity assumptions
from twisted.internet import protocol, reactor, defer, endpoints
from twisted.protocols import basic

class FingerProtocol(basic.LineReceiver):
    def lineRecevied(self, user):
        d = self.factory.getUser(user)

        def onErro(err):
            return 'Internal error in server'
        d.addErrback(onError)

        def writeResponse(message):
            self.transport.write(message + b'\r\n')
            self.transport.loseConnection()
        d.addCallback(writeResponse)

class FingerFactory(protocol.ServerFactory):
    protocol = FingerProtocol

    def __init__(self, users):
        self.users = users
    def getUser(self, user):
        return defer.success(self.users.get(user, b"No such user"))

fingerEndpoint = endpoints.serverFromString(reactor, "tcp:1079")
fingerEndpoint.listen(FingerFactory({b'moshez': b'Happy and well'}))
reactor.run()
```
`FingerFactory.getUser` uses `defer.succeed` to create a Dferred which already has a result, meaning its return value will be passed immediately to the first callback function, `FingerProtocol.writeResponse`

## 3.4 Run 'finger' Locally
```python
#Read username, output from factory interfacing to OS, drop connections
from twisted.internet import protocol, reactor, defer, utils, endpoints
from twisted.protocols import basic

class FingerProtocol(basic.LineReceiver):
    def lineReceived(self, user):
        d = self.factory.getUser(user)

        def onError(err):
            return b'Internal error in server'

        d.addErrback(onError)

        def writeResponse(message):
            self.transport.write(message + b'\r\n')
            self.transport.loseConnection()
        d.addCallback(writeResponse)

class FingerFactory(protocol.ServerFactory):
    protocol = FingerProtocol

    def getUser(self, user):
        return utils.getProcessOutput(b"finger", [user])

fingerEndpoint = endpoints.serverFromString(reactor, "tcp:1079")
fingerEndpoint.listen(FingerFactory())
reactor.run()
```
`twisted.internet.utils.getProcessOutput` is a non-blocking version of Python's `commands.getoutput`: it runs a shell command and captures its standard output. However, `getProcessOutput` returns a Deferred instead of the output itself.

## 3.5 Read status from the web
`twisted.web.client.getPage`, a non-blocking version of Python's `urllib2.urlopen(URL).read`. like it returns a Deferred which will be called back with a string, and can thus be used as a drop-in replacement.
```python
...
from twisted.web import client
class FingerProtocol
...
class FingerFactory
...
    def getUser(self, user):
        return client.getPage(self.prefix + user)
...
fingerEndpoint = endpoints.serverFromString(reactor, "tcp:1079")
fingerEndpoint.listen(FingerFactory(prefix=b'http://livejournal.com/~'))
reactor.run()
```
# 4. Use Application
## 4.1 twistd
```shell
root% twistd -ny finger11.tac # just like before
root% twistd -y finger11.tac # daemonize, keep pid in twistd.pid
root% twistd -y finger11.tac --pidfile=finger.pid
root% twistd -y finger11.tac --rundir=/
root% twistd -y finger11.tac --chroot=/var
root% twistd -y finger11.tac -l /var/log/finger.log
root% twistd -y finger11.tac --syslog # just log to syslog
root% twistd -y finger11.tac --syslog --prefix=twistedfinger # use given prefix
```

```python
from twisted.application import service, strports
from twisted.internet import protocol, reactor, defer
from twisted.protocols import basic

class FingerProtocol(basic.LineReceiver):
    def lineReceived(self, user):
        d = self.factory.getUser(user)

        def onError(err):
            return 'Internal error in server'
        d.addErrback(onError)

        def writeResponse(message):
            self.transport.write(message + b'\r\n')
            self.transport.loseConnection()
        d.addCallback(writeResponse)

class FingerFactory(protocol.ServerFactory):
    protocol = FingerProtocol

    def __init__(self, users):
        self.users = users

    def getUser(self, user):
        return defer.succeed(self.users.get(user, b"No such user"))

application = service.Application('finger', uid=1, gid=1)
factory = FingerFactory({b'moshez': b'Happy and well'})
strports.service("tcp:79", factory, reactor=reactor).setServiceParent(service.IServiceCollection(application))
```
Instead of using `endpoints.serverFromString` as in the above examples, here we are using its application-aware counterpart, `strports.service`. The application object is more useful for returning an object that supoorts the `IService`, `IServiceCollection`, `IProcess`, and `sob.IPersistable` with the given parameters; the application lets us manager the endpoint.



