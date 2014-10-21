# How to Create a Descriptor

Descriptors and Values are the two core architectural concepts in Quixote.
 
* **Descriptors** represent some *as yet uncalculated* aspect of CSS
* **Values** contain calculated values.

This directory is for descriptors.

A descriptor has the following key features, which should be implemented in this order.

* It has tests.
* It provides factory methods for construction.
* It converts itself to a Value object. (`value()`)
* It converts primitives to Value objects that are comparable to itself. (`convert(arg, type)`)
* It renders itself as a string. (`toString()`)
* It extends the `Descriptor` base class.
* Optional: It provides properties or methods that return other descriptors.

The following explanations use the (as yet fictional) example of a `BackgroundColor` descriptor. It represents the `background-color` CSS property.


## Tests

Start out by creating a simple testbed. In our case, we need an element with a background color.

```javascript
"use strict";

var assert = require("../util/assert.js");
var reset = require("../__reset.js");
var BackgroundColor = require("./background_color.js");

describe("BackgroundColor", function() {
  
  var COLOR = "#abcde0"       // our test element's background color
  
  var element;                // the test element
  var color;                  // the descriptor under test
  
  beforeEach(function() {
    // get the test frame
    var frame = reset.frame;
    
    // add our test element
    frame.addElement(
      "<p id='element' style='background-color: " + COLOR + "'>element</p>"
    );
    element = frame.getElement("#element");
    
    // create the test descriptor
    color = BackgroundColor.create(element);
  });
  
});
```


## Factory Methods

We have a convention of using factory methods, not constructors, to create all descriptors and values. The factory methods use a normal constructor under the covers, but other code is expected to use the factory.
 
```javascript
"use strict";

var ensure = require("../util/ensure.js");

var Me = module.exports = function BackgroundColor(element) {
  // Check that the constructor is called with the correct parameter types
  // Normally we would do the require at the top of the file, but we need to break a circular dependency with QElement. 
  var QElement = require("./q_element.js");
  ensure.signature(arguments, [ QElement ]);
  
  // store the element for later
  this._element = element;
};

Me.create = function create(element) {
  // Simply call the constructor. We don't call 'ensure.signature()' because the constructor already does that.
  return new BackgroundColor(element);
};
```


## Convert to Value (`value()`)

A descriptor is a *description* of a CSS property. Descriptors don't actually calculate the value of the property until `value()` is called, so this method is where the magic happens.

We start by testing that our descriptor gives us the actual background-color of our test element. We're assuming that we have an (also fictional) `Color` value object, and that we've required it at some point.

```javascript
it("resolves to value", function() {
  assert.objEqual(color.value(), Color.create(COLOR));
});
```

We implement it by getting `background-color` from our element. Note that `value()` always returns a value object, never a primitive.
 
```javascript
Me.prototype.value = function() {
  ensure.signature(arguments, []);      // more parameter checking
  
  // get the style
  var style = this._element.getRawStyle("background-color");
  
  // convert it to a value object and return
  return Color.create(style);
};
```


## Convert primitives (`convert()`)

In some cases, the user might try to compare our descriptor to a primitive type. By default, this results in an error. The `convert()` method allows us to handle whichever primitives we want.
 
Any primitives that aren't handled should be ignored, resulting in a return value of `undefined`.

Our test and production code:

```javascript
it("converts comparison arguments", function() {
  assert.objEqual(color.convert("#aabbcc", "string"), Color.create("#aabbcc"));
});
```

```javascript
Me.prototype.convert = function convert(arg, type) {
  // We don't check parameters because they're checked for us by the caller 
	if (type === "string") return Color.create(arg);
};
```


## Render to a string (`toString()`)

Remember, a descriptor is a *description* of a CSS property, not the value of the property, so when we render to a string, we want to *describe* the property. This human-readable description will be used when describing differences.

In this case, we represent the background color of an element, so a good value for `toString()` might be something like "background color of 'element'".

Here's our test and production code:

```javascript
it("converts to string", function() {
  assert.equal(color.toString(), "background color of " + element);
});
```

```javascript
Me.prototype.toString = function toString() {
  // check parameters
	ensure.signature(arguments, []);

  return "background color of " + this._element;
};
```


## Extend `Descriptor` base class

All descriptors have to extend Descriptor in order to work properly. We do this last because the tests will fail if the other methods aren't implemented.

Our tests and code:

```javascript
it("is a descriptor", function() {
  assert.implements(center, Descriptor);
});
```

```javascript
Descriptor.extend(Me);
```


## Optional properties or methods

You can also provide properties or methods that return other descriptors. For example, we might want to add a `darken()` method that provides a relative color. See `element_center.js` for an example.