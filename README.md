     _   _     _   ___      _
    |*|_|*|___|*|_|*  |___ |*|_ ___ ___ ___
    | | | |*_ |   | | |* _|   _|*  |* _|*__|
    | | | | __| | |   | |_ | |_| | | | |__ |
    |_____|___|___|_|_|___||___|___|_| |___|

WebActors is a simple library for managing concurrency
in JavaScript programs. It's heavily based on Erlang's
interpretation of the Actor Model.

For an introduction to actors in general, and WebActors in
particular, scroll down to the section entitled
"Tutorial".

# Building

## Build Requirements

WebActors doesn't have any special run-time requirements
-- the library is just a single JavaScript file -- but it
requires node.js, npm, and the "coffee-script" and "jsmin"
npm packages for development.

## Running Specs

Running `cake spec` will start a local webserver and print
a URL which can be used to run the Jasmine specs.  On most
Linux systems (anything that has xdg-open), it will also
open a browser window and run the specs.

## Building

Running `cake build` with no arguments will build
everything. 

The output files are:

  * dist/npm/webactors-#{version}.tar.gz - an npm package
  * dist/standalone/webactors-#{version}.js - for use in the browser
  * dist/standalone/webactors-#{version}.min.js - minfied version

# Tutorial

## Actors Explained

An "actor" is pretty much just a regular process or
thread with a mailbox attached.  In programming styles
based on the Actor Model, threads communicate with each
other by posting immutable messages to each others'
mailboxes, rather than by reading and writing fields of
mutually-shared objects.

Writing concurrent programs using message-passing can take
some getting used to, but actors can make programs simpler,
and they are also relatively safe from some common
programming errors which are endemic to event-driven
programs.

## Actors in JavaScript

JavaScript has neither processes nor threads (nor
coroutines), but in the absence of these, actors can still
be modeled by a chain of callbacks.  Indeed, actor-based
programming can be a good way to manage the inherent
complexities of callback-driven programming.

### Actors and the JavaScript Event Loop

In WebActors, actors are implemented in a non-reentrant
fashion.  Newly spawned actors won't run, and newly sent
messages won't be delivered, until the currently running
actor returns control to the event loop.

Before returning control to the event loop, an actor can
decide which messages will re-activate it by registering
callbacks for messages matching particular patterns.  If
an actor doesn't set itself up to receive any messages
before returning control to the event loop, or if it
returns control to the event loop by raising an exception,
then that actor will terminate.

### Creating Actors and Sending Messages

The `WebActors.spawn` function is used to
create a new actor.  This function takes a callback to run
in the new actor's context, and returns an id representing
the newly created actor.  This id can be used to submit
messages to the new actor's mailbox.

```javascript
    var actor = WebActors.spawn(aCallback); // create an actor
    WebActors.send(actor, "a message"); // send it a message
```

### Receiving Messages

To receive messages, use the `WebActors.receive`
function.  It takes a pattern and a callback to be invoked
when a matching message is received.

```javascript
    function aCallback() {
      // ANY matches anything
      WebActors.receive(WebActors.ANY, function (message) {
        alert(message);
      });
    }
```

If an actor callback sets up a new callback via receive,
then the actor will continue with the new callback once
a matching message becomes available.  Otherwise, if a
callback "breaks the chain", then the actor will terminate
as soon as the callback finishes.

In the above example, the actor sets up a callback to
receive one message. That callback, in turn, doesn't set
up any further callbacks, so the actor terminates at
that point.

If messages arrive which don't match any outstanding
receive patterns, they will be held until the actor asks
for them (by calling `receive` with a matching pattern).

### Patterns

The first argument to `receive` is a pattern which is used
to match an incoming message.  Patterns are ordinary
JavaScript values.

#### Primitive Values

Used as a pattern, any primitive value matches an
equivalent primitive value (as determined by `===`).

```
    3 matches 3
    "foo" matches "foo"

    3 does not match 4
    "3" does not match 3
    3 does not match "3"
    "foo" does not match "bar"
```

#### Arrays

When used as a pattern, an Array matches an array with
the same length and matching elements (as determined by the
pattern-matching rules)

```
    [1, 2, 3] matches [1, 2, 3]

    [1, 2, 3] does not match [4, 5, 6]
    [1, 2, 3] does not match [1, 2, 3, 4]
```

#### Objects

When used as a pattern, an object matches any object with
the same fields and matching values.  (The matched object
may have other fields in addition.)

```
    {a: 1} matches {a: 1}
    {a: 1} matches {a: 1, b: 2}

    {a: 1} does not match {}
    {a: 1} does not match {a: 3}
    {a: 1} does not match {b: 1}
```

#### Wildcards

`WebActors.ANY` matches any JavaScript value.

#### WebActors.match

If you want to, you can use WebActors pattern matching
directly in your own non-actor code by using
`WebActors.match`.  `match` takes a pattern and a value,
and returns a truthy value in case of a match, or a falsy
one otherwise.

### Saving Some Typing

If you aren't already in the habit of doing so, it can
be useful (and occasionally more readable) to define local
aliases for functions defined on library objects.

```javascript
    (function () {
    var spawn = WebActors.spawn;
    var receive = WebActors.receive;
    var send = WebActors.send;
    var ANY = WebActors.ANY;

    function aCallback() {
      receive(ANY, function (message) {
        alert(message);
      });
    }

    actor = spawn(aCallback); // create an actor
    send(actor, "a message"); // send it a message

    })();
```

Subsequent code samples will assume that such aliases have
already been defined.

### Simultaneous Receives (Choice)

An actor can also choose between alternatives based on
the specific message received. (This is a fragment,
rather than a complete example.)

```javascript
    receive("go left", function () {
      alert("You fall off a cliff.");
    });
    receive("go right", function () {
      alert("You stumble into a pit full of spikes.");
    });
```

In this case, if the actor receives "go left", it will
print the message about falling off a cliff and
terminate.  If it receives "go right", it will print
the message about falling into a pit and terminate.

Only one or the other of these callbacks will fire --
never both.  If a message matches multiple outstanding
receives (which is possible when wildcards are used), the
callback supplied to the first matching receive (in this
case, "go left") will be called.  In other words, earlier
receives take precedence.

### Chained Receives (Sequencing)

An actor can also specifically choose the order in which
it responds to messages by chaining calls to receive.

For example:

```javascript
    receive("up", function () {
      alert("Going up!");
      receive("down", function () {
        alert("Going down!");
      });
    });
```

An arrangement like this guarantees that the actor will 
receive (and act upon) the "up" message before it
receives "down", regardless of the order in which those
messages were originally sent.

### Loops and State Machines

Often, you'll want an actor to continue receiving the same
sort of messages indefinitely.  For example, you might
want to write something like this:

```javascript
    var running = true;
    while (running) {
      receive("say hello", function () {
        alert("Hello!")
      });
      receive("go away", function () {
        running = false;
      });
    }
```

Of course, this won't actually work, since receives are
non-blocking, and in any case no callbacks can get called
until we return to the browser event loop.

A working version looks something like this:

```javascript
    function helloLoop() {
      receive("say hello", function () {
        alert("Hello!");
        helloLoop(); // re-establish the same callbacks
      });
      receive("go away", function () {
        // do nothing, and let the actor exit
      });
    }
    helloLoop(); // start the loop
```

The observant will recognize this as a state machine with
a single non-terminal state.

```
    START -> helloLoop
    helloLoop -> ["say hello"] -> helloLoop
    helloLoop -> ["go away"] -> END
```

More complicated state machines can be built up in the
same way:

```javascript
    function entryHall() {
      receive(["go", "south"], function () { goTo(jungle); });
      receive(["go", "north"], function () { goTo(stairway); });
      desc = "You are in a dingy stone hallway. " +
             "Behind you, to the south, is the exit. " +
             "Ahead, to the north, stairs descend into darkness.";
      doDefaultTransitions(entryHall, desc);
    }

    function jungle() {
      alert("You escape into the jungle!")
    }

    function stairway() {
      receive(["go", "east"], function () { goTo(treasure); });
      receive(["go", "south"], function () { goTo(entryHall); });
      receive(["go", "north"], function () { goTo(pit); });
      desc = "The stairs run downwards, from south to north. " +
             "There is also a narrow doorway to the east.";
      doDefaultTransitions(stairway, desc);
    }

    function treasure() {
      alert("You found the treasure!");
    }

    function pit() {
      alert("The stairs give way beneath you and you fall into a pit!");
    }

    function goTo(location) {
      sendSelf("look");
      location();
    }

    function doDefaultTransitions(location, desc) {
      receive("look", function () {
        alert(desc);
        location();
      });
      receive(["go", ANY], function () {
        alert("You can't go that way!");
        location();
      });
    }

    // how you would actually start a game
    function gameActor {
      goTo(entryHall);
    }
    spawn(gameActor);
```

When things get more complicated, it's often a better idea
to represent states as objects, and explicitly keep track of
which state is current in order to faciliate unit testing
and debugging.

```javascript
    var JUNGLE_ENDING, PIT_ENDING, TREASURE_ENDING;

    function Game() {
      this.locations = {
        entryHall: new EntryHall(this),
        jungle: JUNGLE_ENDING,
        stairway: new Stairway(this),
        treasure: TREASURE_ENDING,
        pit: PIT_ENDING
      };
    }
    Game.prototype.start = function () {
      this.goTo("entryHall");
    }
    Game.prototype.goTo = function (label) {
      this.currentLocation = this.locations[label];
      this.currentLocation.enter();
    }

    function Location() {}
    Location.prototype.enter = function () {
      sendSelf("look");
      this.doTransitions();
    }

    function EntryHall(game) {
      this.game = game;
      this.desc = "You are in a dingy stone hallway. " +
                  "Behind you, to the south, is the exit. " +
                  "Ahead, to the north, stairs descend into darkness.";
    }
    EntryHall.prototype = new Location();
    EntryHall.prototype.doTransitions = function () {
      var game = this.game;
      receive(["go", "north"], function () { game.goTo("stairway"); });
      receive(["go", "south"], function () { game.goTo("jungle"); });
      doDefaultTransitions(this);
    }

    function Stairway(game) {
      this.game = game;
      this.desc = "The stairs run downwards, from south to north. " +
                  "There is also a narrow doorway to the east.";
    }
    Stairway.prototype = new Location();
    Stairway.prototype.doTransitions = function () {
      var game = this.game;
      receive(["go", "north"], function () { game.goTo("pit"); });
      receive(["go", "south"], function () { game.goTo("entryHall"); });
      receive(["go", "east"], function () { game.goTo("treasure"); });
      doDefaultTransitions(this);
    }

    function doDefaultTransitions(location) {
      receive("look", function () {
        alert(location.desc);
        location.doTransitions();
      });
      receive(["go", ANY], function () {
        alert("You can't go that way!");
        location.doTransitions();
      });
    }

    function Ending(message) {
      this.message = message;
    }
    Ending.prototype.enter = function () {
      alert(this.message);
    }

    JUNGLE_ENDING = new Ending("You escape into the jungle!");
    PIT_ENDING = new Ending("The stairs give way beneath you and you fall into a pit!");
    TREASURE_ENDING = new Ending("You found the treasure!");

    // how you would actually start a game
    function gameActor() {
      var game = new Game();
      game.start();
    }
    spawn(gameActor);
```

This sort of arrangement permits the rooms and endings to
be unit-tested individually, and as a bonus you can easily
inspect the current room from a debugger.

If you find yourself faced with a hierarchical state
machine, you may find it better to model as a set of
cooperating actors rather than trying to cram
everything into one.

### Supervision Trees

[write about supervision trees here]

# API Reference

WebActors defines a single object in the top-level
namespace, unsurprisingly called WebActors.  It has a
number of properties and methods attached to it.

## Actors

Most of the following functions in this section must be
called by an actor; the individual exceptions to this
rule are:

  * `WebActors.spawn`
  * `WebActors.send`
  * `WebActors.kill`
  * `WebActors.sendbackTo`

### WebActors.spawn(body) -> actorId

The `spawn` method spawns a new actor, returning
its id.

The actor will termiate after `body` returns,
unless `body` suspends the actor by calling
`receive`.

`spawn` may be called outside an actor.

### WebActors.spawnLinked(body) -> actorId

The `spawnLinked` method is similar to
`spawn`, except that it atomically links the
spawned actor with the current actor.

### WebActors.self()

The `self` method returns the id of the current
actor.

### WebActors.send(actorId, message)

Sends a message to another actor asynchronously. The
message is put in the receiving actor's mailbox, to
be retrieved with `receive`.  If the specified actor
doesn't exist, `send` has no effect.

`send` may be called outside an actor.

### WebActors.sendSelf(message)

Like `send`, but sends a message to the current
actor's mailbox.  Equivalent to
`WebActors.send(WebActors.self(), message)`

### WebActors.receive(pattern, cont)

Sets up a one-shot handler to be called if a message
matching the given pattern arrives.  The pattern will
be structurally matched against candidate messages
using `match`; a list of captured subvalues will
be passed to the supplied continuation callback.

The set of outstanding receives for an actor is
cleared each time the actor successfully receives
a message.

If an actor doesn't establish any receives before
returning to the event loop, or if it raises an
uncaught exception, the actor will terminate.

### WebActors.link(actorId)

The `link` method links the current actor with
the given actor, provided there wasn't already an existing
link between them.

If the specified actor doesn't exist, then `link` will
kill the current actor.

### WebActors.unlink(actorId)

The `unlink` method unlinks the current actor
from the given actor, if there was an existing link between
the two.

`unlink` has no effect if the specified actor doesn't
exist.

### WebActors.kill(recipientId, reason)

The `kill` method kills the given actor whether or not it
is linked with the current actor.  It has no effect if the
specified actor doesn't exist.

`kill` may be called outside an actor.

### WebActors.trapKill(function (killerId, reason) {...})

Normally, when an actor receives a kill, it will immediately
exit.  `trapKill` allows for more nuanced behavior
than the default.

This passed-in callback function receives two arguments:
the id of the originating (not the receiving!) actor, and
the reason for the kill.  It should return a message to
be delivered to the actor receiving the kill.  If the
function instead throws an exception, then the receiving
actor will die with that exception.

The callback should have no side-effects and should avoid
making state-modifying calls to the WebActors API.

Note that kills don't, in themselves, break links.  If an
actor is sent a kill message by an actor it is linked to,
the link will remain in place until the actor exits or
it calls `unlink`.

## Callbacks and "Sendbacks"

Most Javascript APIs are heavily callback-based, but it
isn't possible to run a callback in the context of an
actor.  However, WebActors includes utilities for bridging
the gap between callbacks and actors.

### WebActors.sendback(function (args...) { ... })

Generates a callback function which sends a message to the
current actor (at the time `sendback` is called).  The
content of the message is determined by what the function
returns passed to `sendback`; it receives the callback
arguments.

### WebActors.sendbackTo(actorId, function (args...) { ... })

Like `sendback`, but `sendbackTo` allows you to set up
the callback to send a message to a different actor. 

`sendbackTo` may be called outside an actor.

## Pattern Matching

WebActors also provides a utility function for performing
structural pattern matching on Javascript values.

### WebActors.match(pattern, value) -> result

`match` performs structural matching on
JavaScript values. It takes a pattern and a value,
returning an array of captured subvalues if the match is
successful, or `null` otherwise.

### WebActors.ANY

When used in a pattern, `ANY` matches any value.

## Web Workers

In browsers that support it, WebActors offers support for
true multicore parallelism through the use of HTML5 Web
Workers.

Not only can WebActors be used within a worker, WebActors
itself provides an actor-based API for managing Web
Workers in a way that largely abstracts the difference
between actors running in a web worker and actors running
in the page context.

### WebActors.spawnWorker(scriptUrl)

Starts a Web Worker running the given script and spawns
an actor inside it, returning the new actor's id.  This
actor will run in parallel with the parent VM and can
be used to perform background tasks without blocking the
browser UI.

In most respects, the spawned actor behaves like any
other WebActors actor; it can spawn more actors within
the worker, and can also interoperate with actors in the
worker's parent VM.  The main difference is that any
messages crossing the worker/parent boundary are
transferred using `postMessage`, so messages are copied
rather than sent by reference, and messages containing
non-POD objects may not be received as expected.

### WebActors.spawnLinkedWorker(scriptUrl)

Analagous to `spawnLinked`, but spawns a worker just as
`spawnWorker` does.  The spawned worker is immediately
linked to the actor that spawned it.

### WebActors.initWorker(function () {...})

The "bottom half" of `spawnWorker`, `initWorker` MUST
be called before any other WebActors API functions in
a script started via `spawnWorker`.

The passed-in function supplies the body of the actor
returned by `spawnWorker`.

When this actor exits, the worker will terminate, killing
any other actors inside the worker!

### WebActors.terminateWorker(actorId)

Forcibly terminates a worker spawned by `spawnWorker`,
even if it is stuck in an infinite loop.  This is only
effective when called from the worker's parent VM;
otherwise, it has no effect.
