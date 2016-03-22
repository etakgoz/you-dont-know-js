# Chapter 1

## Event Loop:

It's important to note that setTimeout(..) doesn't put your callback on the event loop queue. What it does is set up a timer; when the timer expires, the environment places your callback into the event loop, such that some future tick will pick it up and execute it.

What if there are already 20 items in the event loop at that moment? Your callback waits. It gets in line behind the others -- there's not normally a path for preempting the queue and skipping ahead in line. This explains why setTimeout(..) timers may not fire with perfect temporal accuracy. You're guaranteed (roughly speaking) that your callback won't fire before the time interval you specify, but it can happen at or after that time, depending on the state of the event queue.

So, in other words, your program is generally broken up into lots of small chunks, which happen one after the other in the event loop queue. And technically, other events not related directly to your program can be interleaved within the queue as well.

## Run-to-Completion:

Because of JavaScript's single-threading, the code inside of foo() (and bar() ) is atomic, which means that once foo()starts running, the entirety of its code will finish before any of the code in bar() can run, or vice versa. This is called "runto- completion" behavior.


## Concurrency

Concurrency is when two or more "processes" are executing simultaneously over the same period, regardless of whether their individual constituent operations happen in parallel (at the same instant on separate processors or cores) or not. You can think of concurrency then as "process"-level (or task-level) parallelism, as opposed to operation-level parallelism (separate-processor threads).

The single-threaded event loop is one expression of concurrency (there are certainly others, which we'll come back to later).

## Interaction

### Gate

```javascript
var a, b;

function foo(data) {
	a = data[a];
	if (typeof a !== "undefined" && typeof b !== "undefined") {
		baz();
	}
}

function bar(data) {
	b = data[b];
	if (typeof a !== "undefined" && typeof b !== "undefined") {
		baz();
	}
}

function baz() {
	console.log (at + b);
}

ajax("http://test.com/fetch1", foo);
ajax("http://test.com/fetch2", bar);

```

The if (a && b) conditional around the baz() call is traditionally called a "gate," because we're not sure what order a and b will arrive, but we wait for both of them to get there before we proceed to open the gate (call baz()).

### Latch

Another concurrency interaction condition you may run into is sometimes called a "race," but more correctly called a "latch." It's characterized by "only the first one wins" behavior. Here, nondeterminism is acceptable, in that you are explicitly saying it's OK for the "race" to the finish line to have only one winner.

```javascript
var a;

function foo(data) {
	if (typeof a !== "undefined") {
		a = data[a] * 2;
		baz();
	}
}

function bar(data) {
	if (typeof a !== "undefined") {
		a = data[a] / 2;
		baz();
	}
}

function baz() {
	console.log (a);
}

ajax("http://test.com/fetch1", foo);
ajax("http://test.com/fetch2", bar);

```

### Cooperation (Diving Data into Chunks Make the App More Performant / Responsive)


```javascript

var res = [];

function response (data) {
	var chunk = data.splice(0, 1000);

	res = res.concat( chunk.map(val => val * 2) );

	if (data.length > 0 ) {
		setTimeout(function () {
			response(data);
		}, 0);
	}
}

```

We process the data set in maximum-sized chunks of 1,000 items. By doing so, we ensure a short-running "process," even if that means many more subsequent "processes," as the interleaving onto the event loop queue will give us a much more responsive (performant) site/app.

Note: setTimeout(..0) is not technically inserting an item directly onto the event loop queue. The timer will insert the event at its next opportunity. For example, two subsequent setTimeout(..0) calls would not be strictly guaranteed to be processed in call order, so it is possible to see various conditions like timer drift where the ordering of such events isn't predictable. In Node.js, a similar approach is process.nextTick(..) . Despite how convenient (and usually more performant) it would be, there's not a single direct way (at least yet) across all environments to ensure async event ordering. We cover this topic in more detail in the next section.

## Jobs

```javascript

console.log( "A" );
setTimeout( function(){
	console.log( "B" );
}, 0 );

// theoretical "Job API"
schedule( function(){
	console.log( "C" );
	schedule( function(){
		console.log( "D" );
	} );
} );

```

You might expect this to print out A B C D , but instead it would print out A C D B , because the Jobs happen at the end of the current event loop tick, and the timer fires to schedule for the next event loop tick (if available!).

# Chapter 2: Callbacks

The most troublesome problem with callbacks is inversion of control leading to a complete breakdown along all those trust lines. ===> Trust but Verify

## Trying to Save Callbacks

Regarding more graceful error handling, some API designs provide for split callbacks (one for the success
notification, one for the error notification):


```javascript

function success (data) {
	// handle success...
}

function failure (error) {
	// handle error....
}

ajax("http://www.test.co", success, failure);

```

Another common callback pattern is called "error-first style" (sometimes called "Node style," as it's also the convention used across nearly all Node.js APIs), where the first argument of a single callback is reserved for an error object (if any). If success, this argument will be empty/falsy (and any subsequent arguments will be the success data), but if an error result is being signaled, the first argument is set/truthy (and usually nothing else is passed):

```javascript

function callback (err, data) {
	if (err) {
		console.log("error");
	} else {
		// handle success...
	}
}

ajax("http://www.test.co", callback);

```


What about the trust issue of never being called? If this is a concern (and it probably should be!), you likely will need to set up a timeout that cancels the event. You could make a utility (proof-of-concept only shown) to help you with that:

```javascript

function timeoutify(fn,delay) {
	var intv = setTimeout( function() {
					intv = null;
					fn( new Error( "Timeout!" ) );
			   }, delay );

	return function() {
		// timeout hasn't happened yet?
		if (intv) {
			clearTimeout( intv );
			fn.apply( this, arguments );
		}
	};
}

// using "error-first style" callback design
function foo(err,data) {
	if (err) {
		console.error( err );
	}
	else {
		console.log( data );
	}
}

ajax( "http://some.url.1", timeoutify( foo, 500 ) );

```

Another trust issue is being called "too early." In application-specific terms, this may actually involve being called before some critical task is complete. (TODO: Implement a function that handles this problem)


But more generally, the problem is evident in utilities that can either invoke the callback you provide now (synchronously), or later (asynchronously). ===> Don't release the Zalgo! Always behave async when expected!


TODO: Analyze this below in detail and implement in your own

```javascript

function asyncify(fn) {
	var orig_fn = fn,
		intv = setTimeout( function(){
				intv = null;
				if (fn) fn();
			}, 0 );

	fn = null;

	return function() {
		// firing too quickly, before `intv` timer has fired to
		// indicate async turn has passed?
		if (intv) {
			fn = orig_fn.bind.apply(
				 orig_fn,
				// add the wrapper's `this` to the `bind(..)`
				// call parameters, as well as currying any
				// passed in parameters
				[this].concat( [].slice.call( arguments ) )
			);
		}

		// already async
		else {
			// invoke original function
			orig_fn.apply( this, arguments );
		}
	};
}

function result(data) {
	console.log( a );
}

var a = 0;
ajax( "..pre-cached-url..", asyncify( result ) );

a++;

```

## Review

We need a way to express asynchrony in a more synchronous, sequential, blocking manner, just like our brains do.
Second, and more importantly, callbacks suffer from inversion of control in that they implicitly give control over to another party (often a third-party utility not in your control!) to invoke the continuation of your program. This control transfer leads us to a troubling list of trust issues, such as whether the callback is called more times than we expect.

Inventing ad hoc logic to solve these trust issues is possible, but it's more difficult than it should be, and it produces clunkier and harder to maintain code, as well as code that is likely insufficiently protected from these hazards until you get visibly bitten by the bugs.

We need a generalized solution to all of the trust issues, one that can be reused for as many callbacks as we create
without all the extra boilerplate overhead.



