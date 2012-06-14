# Welcome to Monterey!

Monterey is a tiny JavaScript library that neatly encapsulates some of my
favorite usage patterns and makes them a core part of the language. Some of
these features have to do with object-oriented programming, reflection, and
inheritance, while others are geared towards enabling a more fluent event-driven
interface.

This document explains each feature and gives some guidance as to why I've
included it and how to use it. If you're a newcomer to JavaScript from another
object-oriented language like Java, Python, or Ruby, you'll probably find most
of these additions quite welcome. If you're an experienced JavaScript programmer
you'll still probably enjoy reading the source to see how some of the new
features in ES5 are used to expose JavaScript's powerful object model and
meta-programming interface.

## Features

### Inheritance

Although it may be used in a mostly-functional style, JavaScript has a simple
and powerful object model at its heart built around prototypal inheritance. What
this means in practice is that every function has a `prototype` object attached.
When object instances are created using the `new` keyword in front of a function
they reference all the properties and methods of that function's prototype. As
such, all properties of the prototype object are also properties of that object.

Even though a function automatically comes with a prototype, a function may
actually use any object as its prototype. Thus, we can mimic a classical
inheritance strategy by setting the prototype of some function to an instance of
some other function, its "superclass".

Monterey provides the following properties and methods that make it easier to
use this pattern of inheritance in JavaScript:

  - `Object#class`
  - `Function#inherit(fn)`
  - `Function#isSuperclassOf(fn)`
  - `Function#isSubclassOf(fn)`
  - `Function#superclass`
  - `Function#ancestors`

`Object#class` is a simple getter alias for an object's `constructor`, except
that it's shorter to type and complements the "class"-style naming convention
better.

```javascript
function Person(name) {
  this.name = name;
}

var michael = new Person("Michael");

assert(Person.class === Function);
assert(michael.class === Person);
assert(michael.class.name === "Person");
```

`Function#inherit` is used to set up the prototype chain from one function to
another. In the following contrived example we have an `Employee` class inherit
from the `Person` class defined above.

```javascript
function Employee(name, title) {
  Person.call(this, name);
  this.title = title;
}

Employee.inherit(Person);

Employee.superclass; // Person
Employee.isSubclassOf(Person); // true
Person.isSuperclassOf(Employee); // true
Employee.ancestors; // [Person, Object]
```

Note: It is important to remember to call the parent function inside the child
constructor, otherwise you'll probably be missing some important initialization
logic the parent class provides.

### Events

Most JavaScript programs rely heavily on an event-driven architecture to know
when to do things. For example, in a web browser event handlers are used to run
a piece of code in response to things that the user does such as clicking on a
button or submitting a form.

Libraries like [jQuery](http://jquery.com) can be very useful for setting up
event listeners on a web page, but event-driven architecture can be a very
useful tool as a general method of organizing code.

In Monterey, any JavaScript object is able to register event handlers and notify
listeners when events happen. The following methods make this possible:

  - `Object#on(type, handler)`
  - `Object#off(type, [ handler ])`
  - `Object#trigger(type, args...)`

`Object#on` is used to register a handler function for a given type of event.
`Object#trigger` triggers an event of a given type and calls all handlers for
events of that type with an event object and any additional arguments given in
the scope of the receiver. `Object#off` un-registers a specific event handler if
given, or all event handlers for the given type if no handler is given.

Functions in Monterey make use of this feature to trigger "inherited" events
when another function inherits from them (see the Inheritance section above).

```javascript
function Person(name) {
  this.name = name;
}

// Use this array to keep track of subclasses when Person
// is inherited.
Person.subclasses = [];

Person.on("inherited", function (e, subclass) {
  e.type; // "inherited"
  e.time; // Date
  e.source; // Person
  this.subclasses.push(subclass);
});

function Employee(name, title) {
  Person.call(this, name);
  this.title = title;
}

Employee.inherit(Person);

Person.subclasses; // [Employee]
```

By default events have `type`, `time`, and `source` properties as in the
example above.

Note: If an event handler returns `false` all remaining handlers for that event
are not called.

### Mixins

Classical inheritance is the right solution for situations where the problem
domain can be neatly broken up into a single-inheritance hierarchy. However,
many real world scenarios do not fit this model well. Monterey attempts to
address this issue with "mixins".

A mixin is simply a function that is applied to some other object without being
inserted into that object's prototype chain. To achieve this the receiver is
first extended with all properties of the function prototype, and then the
function is called with the receiver as its `this` so that the object may be
initialized as any other object that uses that constructor function would be.

The following methods make this possible:

  - `Object#mixin(fn)`
  - `Object#mixins`
  - `Object#mixesIn(fn)`

These allow you to mimic multiple inheritance in JavaScript to some extent.
Consider the following:

```javascript
function View() {}

var view = new View;

// A mixin for views we want to be "scrollable".
function Scrollable() {
  this.isScrollable = true;
}

Scrollable.prototype.scroll = function () {
  // ...
};

// A mixin for views we want to be "scrollable".
function Draggable() {
  this.isDraggable = true;
}

Draggable.prototype.drag = function () {
  // ...
};

view.mixin(Scrollable);
view.mixin(Draggable);

view.isScrollable; // true
view.isDraggable; // true
view.mixins; // [Scrollable, Draggable]
view.mixesIn(Scrollable); // true
view.mixesIn(Array); // false
```

The caveat is that since the prototype of a mixin is not inserted into the
object's prototype chain that object does not automatically get any updates to
the prototype object. Also, `instanceof` doesn't work with mixins as it does
with normal inheritance. However, with carefully constructed code this pattern
can be very useful.

### Object#is

The `instanceof` operator can be used to check if an object inherits from a
function (i.e. if that function's prototype appears in the object's prototype
chain). However, this doesn't take into account mixins. `Object#is` solves
this problem for us.

Continuing from the example above:

```javascript
view.is(View); // true
view.is(Scrollable); // true
view.is(Draggable); // true
```

### Object#extend

It's extremely common to need to copy the own properties of one object to
another efficiently. This can be useful when cloning objects, for example, or
when mixing in methods of a function's prototype on an object (see the Mixins
section above).

To use it, simply call `extend` on any object and pass it another object to copy
properties from.

### Object#guid

In Monterey, every object has an `guid` property that is globally unique to
that object. This can be useful in many different situations. For example,
[jQuery](http://jquery.com)'s event subsystem assigns a globally unique id to
event handlers so that it can correctly identify a function later when removing
event handlers. Ruby's [Object class](http://ruby-doc.org/core-1.9.3/Object.html)
includes an `object_id` method that does the same thing.

This id is not generated until you need it so it doesn't slow down normal object
instantiation. Use it on any object.

```javascript
var a = {};
var b = {};

assert(a.guid !== b.guid);
```

## Compatibility

Monterey should work perfectly in any JavaScript environment that supports
ES5, specifically the "static" methods of `Object` including `Object.create`
and `Object.defineProperty`. Please see kangax's excellent [ECMAScript 5 compatibility table](http://kangax.github.com/es5-compat-table/)
for information on which browsers support ES5.

## Installation

Using [npm](http://npmjs.org):

    $ npm install monterey

Otherwise, [download](https://github.com/mjijackson/monterey.js/downloads) the
package from GitHub and include monterey.js as you would any other JavaScript
file.

## Testing

Monterey includes a test suite that runs on [node](http://nodejs.org) using the
[vows](http://vowsjs.org) testing framework. To run the tests, first install
node and vows, then use:

    $ vows monterey_test.js

## License

Copyright (c) 2012 Michael Jackson

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to use,
copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the
Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
