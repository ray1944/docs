# Summary
Core classes:
- `twisted.internet.protocol.Protocol`
- `twisted.internet.protocol.Factory` or `twisted.internet.protocol.ClientFactory`


# Protocol
## A simplest example:
```
from twisted.internet.protocol import Protocol
from sys import stdout

class Eacho(Protocol):
    def dataReceived(self, data):
        stdout.write(data)
```
It just writes whatever it reads from the connection to standard output.

## A example for a Protocol responding to another event
```
from twisted.internet.protocol import Protocol
class WelcomeMessage(Protocol):
    def connectionMade(self):
        self.transport.write("Hello server, I am the client!\r\n")
        self.transport.loseConnection()
    
    def connectionLost(self):
        pass
```
The `connectionMade` event is usually where set up of the `Protocol` object happens.
Any tearing down of `Protocol` - specific objects is done in `connectionLost`.

## Simple, single-use clients

