API
===

```javascript
cw(Object or Function[, Number, Boolean])
	-> worker object
```

is the basic signature but it should be noted that

```javascript
cw(Function[, Number, Boolean])
```

is actually a shortcut to

```javascript
cw({data:Function}[, Number, Boolean])
```

if `Number` is specified, truthy, and greater than 1, then `cw()` is actually a shortcut to `new cw.Queue`,
otherwise it is a shortcut to `new cw.Worker()`

## cw.Worker

the object passed to the constructor is in the format of keys which map to functions
the keys of this object are turned into methods of the returned worker object
additionally if a key with the value 'initialize' is present then its function is
invoked in the worker as soon as the worker is created with the signature:

```javascript
scope.initialize(scope)
```

As a convenience, if a key named
'init' is present and a key named 'initialize' is not present then obj['initialize']
is set to obj['init'].

The methods of the worker object derived from keys have the following signature:

```javascript
worker.method(data[,transfer:Array])
	->promise
```

it is called with any arbitrary data that can be serialized by structured cloning
and and optional array of buffers to transfer and it returns a promise. 
In the worker, the function specified is called with the following signature:

```javascript
scope.method(data, callback, scope)
```

This function can resolve by either passing a value to the callback function or returning a defined value (aka typeof value !== 'undefined')
it may only resolve once, if it calls the callback more than once or returns a value and calls the callback one or more times an error will occur.

The worker object also has a method _close which may be used to close the worker

```javascript
worker._close()
	->promise
```

there is also a shortcut function of `worker.close()` which may be overwritten, 
`worker._close()` will overwrite the method name on the worker object but not inside the worker.
The promise always resolves successfully.

The functions in the worker are always called as methods of the 'scope' object, meaning 
that a function 'a' may call function 'b' as `this.b()` internally. As a convenience,
the scope is passed as the last parameter when functions are invoked (this is the 3rd parameter for most function except 
'initialize' where it is the first and only parameter). Function 'a' could thus do:

```javascript
function(data,callback,scope){
	scope.b();
}
```

the worker object and the scope object both have 3 additional methods
`on`, `one`, `off`, and `fire` for working with events. For these 3 methods the following definitions a being used:

'event name' can be any JavaScript string that does not include a space.

'event string' is one or more event names seperated by spaces.

```javascript
workerORscope.on('event string', listnener:Function[,contect:Object])
	->workerORscope
workerORscope.one('event string', listnener:Function[,contect:Object])
	->workerORscope
```

the listener function is called with the signature, on and one are identical except one is only called once.

```
listnener(data,scope);
```

by default, the scope and the context (third argument, `this` inside the listener) are the same
and be aware that changing the context does not change the scope (the scope object is always the same
even if `this` is changed).

```javascript
workerORscope.off('event string'[, func]);
	->workerORscope
```

Calling the `.off()` method on the worker removes the listener or listeners in the event string. If `func` is provided, it removes only that function.

```javascript
workerORscope.fire('event string'[,data,transfer:Array])
	->workerORscope
```

when `.fire()` is called in the main page, it sends the message with the data (if any) to the worker, transferring any buffers if they are specified when called in the worker it gets sent to the main page.

Internally, in before workers are created, any ImportScripts() declarations are hoisted into the global worker scope and 
deduped.

## cw.Queue

A queue which can be accessed as described above and has the signature:

```javascript
new cw.Queue(Object ,Number [,Boolean])
	-> CatilineQueue
```

A queue can be treated exactly like a worker and it will behave identically except
there will be a number of workers equal to the number specified in the second parameter `Number`.

If the third parameter `Boolean` is falsy (or omited) then it is a 'Managed Queue'. When methods are called
with the queue method, if a worker is free, the call is forwarded to that worker like normal, the worker is
then noted to be busy. When the worker finishes the promise is resolved and the worker is marked as being free.
Data is only given to free workers and if all workers are busy it is placed in a queue and will be
sent to workers on a 'first in, first out' basis as they become free.

If the third parameter `Boolean` is truthy then on each all to queue object a worker is picked at random and the data is sent to that one.

Queues also have properties called 'batch' and 'batchTransfer' which have the same methods as the provided object.

In other words if this queue was created

```javascript
var queue = cw({method:function});
```

not only would there be `queue.method()` method but also `queue.batch.method()` and `queue.batchTransfer.method()`.

`queue.batch.method` is like `queue.method` except it takes an array of values and divides them among the workers
based on the queue strategy. If they all resolve successfully then the promise resolves with an array of the results in the same order as the input values.
If a function has an error then the returned promises are rejected.

`queue.batchTransfer.method` is identical to `queue.batch.method` but instead of an array of values
it accepts an array of arrays of length 2 in which the first element is the value and the second an array of buffers to transfer, such as: `[value,[buffers]]`

if a function is passed to `batch` or `batchTransfer` when the method is called then that function will be called with each result as following: 

```javascript
queue.batch(Function).method([Array])
```

`Function` will be called with each result (it will not be called in case of an error).

if a string is passed to  `batch`, it will clear the managed queue and it will reject all callbacks, as follows: 


```javascript
queue.batch.method([Array])
queue.batch('stopit');
```

## Utility functions

```javascript
cw.makeUrl(reletiveUrl:String)
	->absoluteUrl
```

Takes a relative url and returns an absolute one, handy as relative urls will resolve badly inside a blob worker.

```javascript
cw.setImmediate(func)
```

executes `func` in the next event loop, think `setTimeout(func,0);` but faster.

```javascript
cw.deferred();//creates deffered object
cw.resolve(value);//create resolved promise
cw.reject(value);//create rejected promise
cw.all([promises]);
//returns a promise for an array of promsies
//resolved value is array of results in order.
```
