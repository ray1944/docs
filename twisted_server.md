# Summary
A simple twisted server.
``` python
twisted.internet.protocol.Protocol
```
An instance of the protocol class is instantiated per-connection.
And will go away when the connection is finished.  

The persistent configuration is kept in a Factory class.
```python
twisted.internet.protocol.Factory
```
The buildProtocol method of Factory is used to create a Protocol for each new connection.  

```
IReactorTCP.listenTCP and IReactor*.listen* APIs for the lower level APIs that endpoints are based on.  
```

# Prtocols
```
from twisted.internet.protocol import Protocol
class Echo(Protocol):
    def dataReceived(self, data):
        self.transport.write(data)
```
above is the simplest protocols. it simply writes back whatever is written to it.

```
from twisted.internet.protocol import Protocol
class QOTD(Protocol):
    def connectionMade(self):
        self.transport.write("An apple a day keeps the doctor away\r\n")
        self.transport.loseConnection()
```
This protocol responds to the initial connection with a well know quote, and then terminates the connection.  

```
from twisted.internet.protocol import Protocol
class Echo(Protocol):
    def __init__(self, factory):
        self.factory = factory
    
    def connectionMade(self):
        self.factory.numProtocols = self.factory.numProtocols + 1
        self.transport.write("Welcome! There are currently %d open connections.\n" % 
        (self.factory.numProtocols,))
    
    def connectionLost(self, reason):
        self.factory.numProtocols = self.factory.numProtocols - 1
    
    def dataReceived(self, data):
        self.transport.write(data)
```
`connectionMake` and `connectionLost` cooperate to keep a count of the active protocols in a shared object, thef actory. The factory is used to share state that exists beyond the lifetime of any given connection.  

## lostConnection() and abortConnection()
`loseConnection` call will close the connection only when all the data has been written by Twisted out to the OS. so it is safe to use in this case without worrying about transport writes being lost. If a producer is being used with the transport, loseConnection will only close the connection once the producer is unregistered. 

`loseConnection` will wait for data transport complete, but in some abnoraml situation, network failures, or bugs or maliciousness etc. `loseConnection` called will not be lost. in this case, `abortConnection` can be used. it closes the connection immediately, regardless of buffered data.

## Using the Protocol
```
from twisted.internet.protocol import Factory
from twisted.internet.endpoints import TCP4ServerEndpoint
from twisted.internet import reactor

class QOTDFactory(Factory):
    def buildProtocol(self, addr):
        return QOTD()

endpoint = TCP4ServerEndpoint(reactor, 8007)
endpoint.listen(QOTDFactory())
reactor.run()
```

## Help Protocols
`LineReceiver` protocol. this protocol dispatches to two different event handlers:
- `lineReveived()`, `setLineMode()` called, then `lineReveived()` called;
- `rawDataReceived()`, `setRawMode()` called, then `rawDataReveived()` called.
- `sendLine()`, writes data to the transport along with the delimiter the class uses to split lines by default, \r\n
```
from twisted.protocols.basic import LineReceiver
class Answer(LineReveiver):
    answers = {'How are you?': 'Fine', None: 'I don't know what you mean'}
    def lineReveived(self, line):
        if line in self.answers:
            self.sendLine(self.answers[line])
        else:
            self.sendLine(self.answers[None])
```

# Factory
The default implementation of the `buildProtocol` method calls the protocol attribute of the factory to create a Protocol instance, and then sets an attribute on it called `factory`. 
Here is an example that uses these features instead of overriding `buildProtocol`:
```
from twisted.internet.protocol import Factory, Protocol
from twisted.internet.endpoints import TCP4ServerEndPoint
from twisted.internet import reactor

class QOTD(Protocol):
    def connectionMade(self):
        self.transport.write(self.factory.quote + '\r\n')
        self.transport.loseConnection()

class QOTDFactory(Factory):
    # default buildProtocol to create new protocols
    protocol = QOTD

    def __init__(self, quote=None):
        self.quote = quote or 'An apple a day keep the doctor away'

endpoint = TCP4ServerEndpoint(reactor, 8007)
endpoint.listen(QOTDFactory("configurabel quote"))
reactor.run()
```

## Factory startup and shutdown
it is often not appropritae to do them in __init__ or __del__, and would frequently be too early or too late.
Example:
```python
from twisted.internet.protocol import Factory
from twisted.protocols.basic import LineReceiver

class LoggingProtocol(LineReceiver):
    def lineReceived(self, line):
        self.factory.fp.write(line + '\n')

class LogfileFactory(Factory):
    protocol = LoggingProtocol

    def __init__(self, fileName):
        self.file = fileName
    
    def startFactory(self):
        self.fp = open(self.file, 'a')
    
    def stopFactory(self):
        self.fp.close()
```

# An full example
here is a simple char server that allows users to choose a username and then communicate with other users. it demonstrates the use of shared state in the factory, a state machine for each individual protocol, and communication between different protocols.
```python
from twisted.internet.protocol import Factory
from twisted.protocols.basic import LineReceiver
from twisted.internet import reactor

class Chat(LineReceiver):
    def __init__(self, users):
        self.users = users
        self.name = None
        self.state = "GETNAME"

    def connectionMade(self):
        self.sendLine("What's your name?")
    
    def connectionLost(self, reason):
        if self.name in self.users:
            del self.users[self.name]
    
    def lineReceived(self, line):
        if self.state == "GETNAME":
            self.handle_GETNAME(line)
        else:
            self.handle_CHAR(line)
    
    def handle_GETNAME(self, name):
        if name in self.users:
            self.sendLine("Name take, please choose another.")
            return
        self.sendLine("Welcome, %s!" % (name, ))
        self.name = name
        self.users[name] = self
        self.state = "CHAT"
    
    def handle_CHAR(self, message):
        message = "<%s> %s" % (self.name, message)
        for name, protocol in self.users.iteritems():
            if protocol != self:
                protocol.sendLine(message)
    
class ChatFactory(Factory):
    def __init__(self):
        self.users = {}
    
    def buildProtocol(self, addr):
        return Char(self.users)

reactor.listenTCP(8123, CharFactory())
reactor.run()

```
`listenTCP` is the method which connects a `Factory` to the network. This is the lower-level API that `endpoints` wraps for  you.

```shell
$ telnet 127.0.0.1 8123
 Trying 127.0.0.1...
 Connected to 127.0.0.1.
 Escape character is '^]'.
 What's your name?
 test
 Name taken, please choose another.
 bob
 Welcome, bob!
 hello
 <alice> hi bob
 twisted makes writing servers so easy!
 <alice> I couldn't agree more
 <carrol> yeah, it's great
```



