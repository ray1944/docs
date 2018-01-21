# Summary 
guide of `twisted.internet.defer.Deferred` object.
- what orts of things you can do when you get a Deferred from a funtion call;
- how you can write your code to robustly handle errors in Deferred code.

# Deferreds
The asynchronous library code calls the first callback when the result is available, or the first errback when an error occurs, and the `Deferred` object then hands the results of each callback or errback function to the next function in the chain.

# Callbacks 
for example:
```python
from twisted.internet import reactor, defer
def getDummyData(inputData):
    print('getDummyData called')
    deferred = defer.Deferred()
    reactor.callLater(2, deferred.callback, inputData * 3)
    return deferred

def cbPrintData(result):
    print('Result received: {}'.format(result))

reactor.callLater(4, reactor.stop)
print('Starting the reactor')
reactor.run()
```


## Multiple callbacks
```python
from twisted.internet import reactor, defer
class Getter:
    def gotResults(self, x):
        if self.d is None:
            print("Nowhere to put results")
            return
        d = self.d
        # set to None, avoids any change Getter.gotResults will accidentally fire the same Deferred more than once(AlreadyCalledError exception)
        self.d = None
        if x % 2 == 0:
            d.callback(x * 3)
        else:
            d.errback(ValueError("You used an odd number!"))

    def _toHTML(self, r):
        return "Result: %s" % r

    def getDummyData(self, x):
        self.d = defer.Deferred()
        reactor.callLater(2, self.getResults, x)
        self.d.addCallback(self._toHTML)
        return self.d

def cbPrintData(result):
    print(result)

def ebPrintError(failure):
    import sys
    sys.stderr.write(str(failure))

# this series of callbacks and errbacks will print an error message
g = Getter()
d = g.getDummyData(3)
d.addCallback(cbPrintData)
d.addErrback(ebPrintError)

g = Getter()
d = g.getDummyData(4)
d.addCallback(cbPrintData)
d.addErrback(ebPrintError)

reactor.callLater(4, reactor.stop)
reactor.run()
```
1. fires the Deferred object. `.callback(result)` if succeeded, `errback(failure)` if failed. failure is typically an instance of a `twisted.python.failure.Failure`;
2. triggers callbacks follows the rules:
    - result of the callback is always passed as the first argument to the next callback
    - if a callback raises an exception, switch to errback;
    - an unhandled failure gets passed down the line of errbacks, this creating an asynchronous analog to a series to a series of except: statements.
    - if an errback doesn't raise an exception or return a `twsited.python.failure.Failure` instance, switch to callback.


## Errbacks
If an errback doesn't return anything, then it effectively returns None, meaning that callbacks will continue to be executed after this errback.
## Unhandled Errors
## Handling either synchronous or asynchronous results
## Handling possible Deferreds in the library code


# Cancellation
## Motivation
## Cancellation for Applications which Consume Deferreds
## Default Cancellation Behavior
## Creating Cancellable Deferreds: Custom Cancellation Functions

# Timeouts

# DeferredList
## Other behaviours
## gatherResults

# Class Overview
## Basic Callback Functions
## Chaining Deferreds
