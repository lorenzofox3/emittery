# <img src="media/header.png" width="1000">

> Simple and modern async event emitter

[![Build Status](https://travis-ci.org/sindresorhus/emittery.svg?branch=master)](https://travis-ci.org/sindresorhus/emittery) [![codecov](https://codecov.io/gh/sindresorhus/emittery/branch/master/graph/badge.svg)](https://codecov.io/gh/sindresorhus/emittery)

It's only ~200 bytes minified and gzipped. [I'm not fanatic about keeping the size at this level though.](https://github.com/sindresorhus/emittery/pull/5#issuecomment-347479211)

It works in Node.js and the browser (using a bundler).

Emitting events asynchronously is important for production code where you want the least amount of synchronous operations.


## Install

```
$ npm install emittery
```


## Usage

```js
const Emittery = require('emittery');

const emitter = new Emittery();

emitter.on('🦄', data => {
	console.log(data);
	// '🌈'
});

emitter.emit('🦄', '🌈');
```


## API

### emitter = new Emittery()

#### on(eventName, listener)

Subscribe to an event.

Returns an unsubscribe method.

Using the same listener multiple times for the same event will result in only one method call per emitted event.

##### listener(data)

#### off(eventName, listener)

Remove an event subscription.

##### listener(data)

#### once(eventName)

Subscribe to an event only once. It will be unsubscribed after the first event.

Returns a promise for the event data when `eventName` is emitted.

```js
emitter.once('🦄').then(data => {
	console.log(data);
	//=> '🌈'
});

emitter.emit('🦄', '🌈');
```

#### events(eventName)

Get an async iterator which buffers data each time an event is emitted.

Call `return()` on the iterator to remove the subscription.

```js
const iterator = emitter.events('🦄');

emitter.emit('🦄', '🌈1'); // buffered
emitter.emit('🦄', '🌈2'); // buffered

iterator
	.next()
	.then( ({value, done}) => {
	// done is false
	// value === '🌈1'
		return iterator.next();
	})
	.then( ({value, done}) => {
		// done is false
		// value === '🌈2'
		// revoke subscription
		return iterator.return();
	})
	.then(({done}) => {
		// done is true
	});
```

In practice you would usually consume the events using the [for await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of) statement.
In that case, to revoke the subscription simply break the loop 

```js
// in an async context 
const iterator = emitter.events('🦄');

emitter.emit('🦄', '🌈1'); // buffered
emitter.emit('🦄', '🌈2'); // buffered

for await (const data of iterator){
	if(data === '🌈2')
		break; // revoke the subscription when we see the value '🌈2'
}

```

#### emit(eventName, [data])

Trigger an event asynchronously, optionally with some data. Listeners are called in the order they were added, but execute concurrently.

Returns a promise for when all the event listeners are done. *Done* meaning executed if synchronous or resolved when an async/promise-returning function. You usually wouldn't want to wait for this, but you could for example catch possible errors. If any of the listeners throw/reject, the returned promise will be rejected with the error, but the other listeners will not be affected.

#### emitSerial(eventName, data?)

Same as above, but it waits for each listener to resolve before triggering the next one. This can be useful if your events depend on each other. Although ideally they should not. Prefer `emit()` whenever possible.

If any of the listeners throw/reject, the returned promise will be rejected with the error and the remaining listeners will *not* be called.

#### onAny(listener)

Subscribe to be notified about any event.

Returns a method to unsubscribe.

##### listener(eventName, data)

#### offAny(listener)

Remove an `onAny` subscription.

#### anyEvent()

Get an async iterator which buffers a tuple of an event name and data each time an event is emitted.

Call `return()` on the iterator to remove the subscription.

```js
const iterator = emitter.anyEvent();

emitter.emit('🦄', '🌈1'); // buffered
emitter.emit('🌟', '🌈2'); // buffered

iterator.next()
	.then( ({value, done}) => {
		// done is false
		// value is ['🦄', '🌈1']
		return iterator.next();
	})
	.then( ({value, done}) => {
		// done is false
		// value is ['🌟', '🌈2']
		// revoke subscription
		return iterator.return();
	})
	.then(({done}) => {
		// done is true
	});
```

In the same way as for ``events`` you can subscribe by using the ``for await`` statement

#### clearListeners()

Clear all event listeners on the instance.

If `eventName` is given, only the listeners for that event are cleared.

#### listenerCount(eventName?)

The number of listeners for the `eventName` or all events if not specified.

#### bindMethods(target, methodNames?)

Bind the given `methodNames`, or all `Emittery` methods if `methodNames` is not defined, into the `target` object.

```js
import Emittery = require('emittery');

let object = {};

new Emittery().bindMethods(object);

object.emit('event');
```


## TypeScript

The default `Emittery` class does not let you type allowed event names and their associated data. However, you can use `Emittery.Typed` with generics:

```ts
import Emittery = require('emittery');

const ee = new Emittery.Typed<{value: string}, 'open' | 'close'>();

ee.emit('open');
ee.emit('value', 'foo\n');
ee.emit('value', 1); // TS compilation error
ee.emit('end'); // TS compilation error
```

### Emittery.mixin(emitteryPropertyName, methodNames?)

A decorator which mixins `Emittery` as property `emitteryPropertyName` and `methodNames`, or all `Emittery` methods if `methodNames` is not defined, into the target class.

```ts
import Emittery = require('emittery');

@Emittery.mixin('emittery')
class MyClass {}

let instance = new MyClass();

instance.emit('event');
```


## Scheduling details

Listeners are not invoked for events emitted *before* the listener was added. Removing a listener will prevent that listener from being invoked, even if events are in the process of being (asynchronously!) emitted. This also applies to `.clearListeners()`, which removes all listeners. Listeners will be called in the order they were added. So-called *any* listeners are called *after* event-specific listeners.

Note that when using `.emitSerial()`, a slow listener will delay invocation of subsequent listeners. It's possible for newer events to overtake older ones.


## FAQ

### How is this different than the built-in `EventEmitter` in Node.js?

There are many things to not like about `EventEmitter`: its huge API surface, synchronous event emitting, magic error event, flawed memory leak detection. Emittery has none of that.

### Isn't `EventEmitter` synchronous for a reason?

Mostly backwards compatibility reasons. The Node.js team can't break the whole ecosystem.

It also allows silly code like this:

```js
let unicorn = false;

emitter.on('🦄', () => {
	unicorn = true;
});

emitter.emit('🦄');

console.log(unicorn);
//=> true
```

But I would argue doing that shows a deeper lack of Node.js and async comprehension and is not something we should optimize for. The benefit of async emitting is much greater.

### Can you support multiple arguments for `emit()`?

No, just use [destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment):

```js
emitter.on('🦄', ([foo, bar]) => {
	console.log(foo, bar);
});

emitter.emit('🦄', [foo, bar]);
```


## Related

- [p-event](https://github.com/sindresorhus/p-event) - Promisify an event by waiting for it to be emitted
