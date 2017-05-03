# Illustrator Script Development (Advanced)

This section is all about making Illustrator script development easier and more enjoyable. We will learn how to get some more overview of our program and make our code reusable and reliable.

We will take the script that we created in the [beginner session][url-beginner] and make some improvements to it. It's a simple script that breaks the lines of a text frame into separate texts for each line:
```javascript
var doc = app.activeDocument,
  selection = doc.selection;

for (s = 0; s < selection.length; s++) {
  currentObject = selection[s];

  if (currentObject instanceof TextFrame) {
    var lines = currentObject.lines,
      lineHeight = currentObject.textRange.characterAttributes.leading;

    for (l = 0; l < lines.length; l++) {
      var newText = doc.textFrames.add();
      newText.left = currentObject.left;
      newText.top = currentObject.top - (l * lineHeight);
      lines[l].duplicate(newText.textRange, ElementPlacement.PLACEATBEGINNING);
    }

    currentObject.remove();
  }
}
```

## Snacksize Me
The first improvement to the script we be to make parts of it reusable by extracting them to individual functions. Since our script only does one thing, we can just create ourselves the obvious `createTextFramesFromLines`  function, which takes a multi-lined text frame as its argument:

```javascript
function createTextFramesFromLines(textWithLines) {
  var lines = textWithLines.lines,
    lineHeight = textWithLines.textRange.characterAttributes.leading;

  for (l = 0; l < lines.length; l++) {
    var newText = doc.textFrames.add();
    newText.left = textWithLines.left;
    newText.top = textWithLines.top - (l * lineHeight);

    lines[l].duplicate(newText.textRange, ElementPlacement.PLACEATBEGINNING);
  }

  textWithLines.remove();
}
```

We can then use this function in our main script like this:

```javascript
var doc = app.activeDocument,
  selection = doc.selection;

for (s = 0; s < selection.length; s++) {
  currentObject = selection[s];

  if (currentObject instanceof TextFrame) {
    createTextFramesFromLines(currentObject);
  }
}
```
That's way shorter and much more readable. But there's more we can do! Inside our `createTextFramesFromLines` function there's the part where we add a new text object for every line and copy the line's content to it. Let's extract that into a function, too. We'll call it `copyTextToPosition`, it takes a `TextRange` and `x, y` positions as its arguments.
```javascript
function copyTextToPosition(textRange, x, y) {
  var newText = doc.textFrames.add();
  newText.left = x;
  newText.top = y;
  textRange.duplicate(newPointText.textRange, ElementPlacement.PLACEATBEGINNING);
}
```
Our working script will then look like this:
```javascript
var doc = app.activeDocument,
  selection = doc.selection;

for (s = 0; s < selection.length; s++) {
  currentObject = selection[s];

  if (currentObject instanceof TextFrame) {
    createTextFramesFromLines(currentObject);
  }
}

function createTextFramesFromLines(textWithLines) {
  var lines = textWithLines.lines,
    lineHeight = textWithLines.textRange.characterAttributes.leading,
    x = textWithLines.left,
    y = textWithLines.top;

  for (l = 0; l < lines.length; l++) {
    copyTextToPosition(lines[l], x, y - (l * lineHeight));
  }

  textWithLines.remove();
}

function copyTextToPosition(textRange, x, y) {
  var newText = doc.textFrames.add();
  newText.left = x;
  newText.top = y;
  textRange.duplicate(newPointText.textRange, ElementPlacement.PLACEATBEGINNING);
}
```

Sweet, now we have also have a function that can copy a text frame to any other place on the artboard. Plus, we have more structure and reusability. But it's still quite a lot to read. Let's change that by extracting our functions into a separate file.

## Divide and conquer
From my experience, scripts tend to get big and hefty the more they can do, especially if you have created UI elements to interact with them.

In modern day web development, we split our code into several areas of concern and then use a bundler to combine them back together. We can do the same thing for Illustrator scripts. In the following, I will show you how to do that with `webpack`.

#### The webpack setup
Sorry in advance, if this section gives you a hard time. If you never got in touch with a bundler before, there might occur some confusion about the steps it takes to set it up.
But once you got it running, it will be totally worth it.

First, you need to have `node.js` installed on your system. If don't have it yet, go over to **node.js** and follow the instructions there. Once you're ready with that, go back to your script's project folder.

To install webpack, open a terminal at the folder where your script is located and type `$ npm init` into it. You will be asked to fill in some information about your project, but you can also leave it blank and just skip through it by hitting 'Enter'.

It will have created a `package.json` file in your folder that holds all information about your installed node packages.

Then type `$ npm install webpack --save-dev` to install webpack. It will be located inside the `node_modules` folder.

Open your `package.json` file and inside add the following line to the `scripts` object:
```json
"build": "webpack --display-error-details --watch"
```
It should look something like this:
```json
"scripts": {
  "test": "echo \"Error: no test specified\" && exit 1",
  "build": "webpack --display-error-details --watch",
},
```
Also add a `webpack.config.js` file to your folder with the following content
```javascript
const path = require('path');

module.exports = {
  entry: './index.js',
  output: {
    path: path.resolve(__dirname, 'build'),
    filename: 'myScript.js'
  }
};
 ```
That's a configuration file for webpack, telling it that our main (or entry) file will be named `index.js` and that we want our resulting script file to be in a `build` folder and renamed to `myScript.js`. Naming is up to you, of course.

That being done, let's extract our two functions into a separate file and call it something like `utility.js`. There's some changes we have to make to our functions here. First of all, we have to make them properties of the `
module.exports` object, so we can call them by their name from any other file that requires our utility code.
```javascript
// before
function copyTextToPosition(textRange, x, y) {
  var newText = doc.textFrames.add();
  newText.left = x;
  newText.top = y;
  textRange.duplicate(newPointText.textRange, ElementPlacement.PLACEATBEGINNING);
}
```
```javascript
// now
module.exports = {
  copyTextToPosition: function (textRange, x, y) {
    var newPointText = doc.textFrames.add();
    newPointText.left = x;
    newPointText.top = y;
    textRange.duplicate(newPointText.textRange, ElementPlacement.PLACEATBEGINNING);
  }
};
```


Also, we have to pass the `doc` object, our currently active document, as a parameter to our functions though. The `copyTextToPosition` function uses it to create a new text frame, but being isolated from our main script now, it does not know where to find it, unless we pass it in as an argument.
```javascript
// before
copyTextToPosition: function (textRange, x, y) {
  var newPointText = doc.textFrames.add(); // what's a 'doc'?
  //...
```
```javascript
// now
copyTextToPosition: function (doc, textRange, x, y) {
  var newPointText = doc.textFrames.add(); // ahh… now I see
  //...
```
Your files should look like this:
```javascript
// 'utility.js'

module.exports = {
  copyTextToPosition: function (doc, textRange, x, y) {
    var newPointText = doc.textFrames.add();
    newPointText.left = x;
    newPointText.top = y;
    textRange.duplicate(newPointText.textRange, ElementPlacement.PLACEATBEGINNING);
  },

  createTextFramesFromLines: function (doc, textWithLines) {
    var lines = textWithLines.lines,
      lineHeight = textWithLines.textRange.characterAttributes.leading,
      x = textWithLines.left,
      y = textWithLines.top;

    for (l = 0; l < lines.length; l++) {
      /* We need to add the 'this' keyword, referring to the
      copyTextToPosition function from THIS object (module.exports) */
      this.copyTextToPosition(doc, lines[l], x, y - (l * lineHeight));
    }

    textWithLines.remove();
  }
};
```
```javascript
// index.js (your main script file)

var util = require('./utility.js');
var doc = app.activeDocument,
  selection = doc.selection;

for (s = 0; s < selection.length; s++) {
  currentObject = selection[s];

  if (currentObject instanceof TextFrame) {
    util.createTextFramesFromLines(currentObject);
  }
}
```
Notice how small and readable our main script has now become, compared to the code from the beginning of the article. You can see at a glance what it does.

This is only a small script that does one tiny little thing. Imagine, what a headache a more complex code will give you, if you do not sum up your lines of code into readable little functions and transfer them to separate files.

The only thing left to do is type `$ npm run build` into the terminal to start the bundling process. Your final file should appear in the `build` folder. Take it for a test-drive in Illustrator and see that it still does exactly what we want it to. Except now we have a much more convenient and scalable development environment.

## Be sure that your code works
Knowing for sure that your script will still work after a change is crucial to any developer. You don't want to test your script manually over and over again with all possible edge cases after every little change.That's why in Web Development, you create automated tests. And again we can do the same thing with our Illustrator script.

#### Installing mocha and chai
We will add three more packages to our development environment. 
- `mocha` is a test framework, it will look at all the tests you have specified and check if they pass or fail. 
- `chai` is an assertion library, that enables you to write your tests in different ways.
(Personally, I prefer the [expect][url-expect] style of writing over the node.js [assert][url-assert] way that mocha uses by default. The tests here will be written in `expect` style.)
- `chai-spies` is a small addon for the `chai` library that allows us to trace function execution.

In your terminal, type `$ npm install mocha chai chai-spies --save-dev` to install the packages. 
Then add a task called 'test' to your `package.json` that will run mocha (or change the existing one.) It should look something like this:
```json
"scripts": {
  "test": "./node_modules/mocha/bin/mocha --watch",
  "build": "webpack --display-error-details --watch",
},
```
#### Writing tests
How do we write tests for our functions? Well, let's look at our `copyTextToPosition` function and what it actually does: It creates a new TextFrame in our document at a given position and with the content that we pass in.

We do that by calling `app.activeDocument.textFrames.add()`, so if we want to assure it was actually created, we just have to make sure that the function was executed. Our test would be written like this:
```javascript
describe('copyTextToPosition', function () {
  it('should call add function', function() {
    // create a spy for the 'add' function of the
    // 'app.activeDocument.textFrames' object
    var add = chai.spy.on(app.activeDocument.textFrames, 'add');

    // execute the function
    app.activeDocument.textFrames.add();

    // see if the function was called
    expect(add).to.have.been.called();
  });
});
```
Let's try out our test and see if it works. Create an empty `utilitySpec.js` file and put it into a `test` subfolder of your project folder. Add these lines to the file:
```javascript
// utilitySpec.js
var chai = require('chai'),
 spies = require('chai-spies'),
 expect = chai.expect;
chai.use(spies);

// our test
describe('copyTextToPosition', function () {
  it('should call add function', function() {
    var add = chai.spy.on(app.activeDocument.textFrames, 'add');
    app.activeDocument.textFrames.add();

    expect(add).to.have.been.called();
  });
});
```
Then run `$ npm run test` in your terminal. You will see your test failing with a ReferenceError, because it doesn't know the global `app` variable that we have available when we run our script in Illustrator. So we will have to find a little workaround to get it running. We will pretend to have that global variable available and mock its properties.

This will be the minimal solution to get our test passing:
```javascript
// utilitySpec.js
var chai = require('chai'),
 spies = require('chai-spies'),
 expect = chai.expect;
chai.use(spies);

describe('copyTextToPosition', function () {
  // before we run the test, we will create a mock app variable,
  // that has the needed properties
  before(function() {
    global.app = {
      activeDocument: {
        textFrames: {
          add: function () {
            console.log('text frame was added');
          }
        }
      }
    };
  });
  
  it('should call add function', function() {
    var add = chai.spy.on(app.activeDocument.textFrames, 'add');
    app.activeDocument.textFrames.add();

    expect(add).to.have.been.called(); // will pass
  });
});
```
Since want to test our utility functions here, let's bring them in and see if our test passes this time:
```javascript
// utilitySpec.js
var chai = require('chai'),
 spies = require('chai-spies'),
 expect = chai.expect;
chai.use(spies);

// add utility.js
var util = require('../utility.js');

describe('copyTextToPosition', function () {
  before(function() {
    global.app = {
      activeDocument: {
        textFrames: {
          add: function () {
            console.log('text frame was added');
          }
        }
      }
    };
  });

  it('should call add function', function() {
    var add = chai.spy.on(app.activeDocument.textFrames, 'add');
    var doc = app.activeDocument;

    // the TextRange that we want to copy. Let's try leaving it blank
    var text = null;

    // call our utility function like we would do when we use our script.
    util.copyTextToPosition(doc, text, 100, 100);
    expect(add).to.have.been.called();
  });
});
```
The test will fail, because this time it `Cannot set property 'left' of undefined.`
Looking at our `copyTextToPosition` function, not only do we use the `doc.textFrames.add()` method, we also expect it to return a new `TextFrameItem` object with `left` and `top` properties that we can set.

Below that, it uses `textRange.duplicate()`, which we also need to mock in order to make our test pass. So here is the full annotated test for our `copyTextToPosition` function:

```javascript
var chai = require('chai'),
 spies = require('chai-spies'),
 expect = chai.expect;
chai.use(spies);

var util = require('../utility.js');

// define mock objects
var TextFrameItem = function() {
  this.left = 0;
  this.top = 0;
};

var TextRange = function() {
  this.duplicate = function (relativeObject,insertionLocation) {};
};

describe('copyTextToPosition', function () {
  // before we run our test, we will fill the app variable with some
  // mock content
  before(function() {
    global.app = {
      activeDocument: {
        textFrames: {
          add: function() {
            return new TextFrameItem();
          }
        }
      }
    };
    global.ElementPlacement = {
      PLACEATBEGINNING: {}
    };
  });

  it('should call add function', function() {
    // create a spy for the 'add' function of the
    // 'app.activeDocument.textFrames' object
    var add = chai.spy.on(app.activeDocument.textFrames, 'add');

    // get our active document
    var doc = app.activeDocument;

    // the TextRange that we want to copy to another position.
    var text = new TextRange();

    // call our utility function just like we would do when we use our script.
    util.copyTextToPosition(doc, text, 100, 100);

    // define our expectation
    expect(add).to.have.been.called(); // will pass
  });
});
```

Alright, now we have a test proving to us, that our utility function `copyTextToPosition()` will call the `add` method. We should assure ourselves of the other effects of our function as well, i.e. that the new TextFrame will be at the expected position and that the content will be copied  using the `duplicate()` function:
```javascript
expect(add).to.have.been.called();
expect(newText.left).to.equal(100);
expect(newText.top).to.equal(100);
expect(duplicate).to.have.been.called();
```
Also, we haven't covered our other utility function `createTextFramesFromLines()` yet. To do so, we will have to create more mock objects, like  `Lines` and `CharacterAttributes`. This time, we will make them customizable, so that, for example, we can mock a TextFrame with 7 lines of text.

For better readability, we also extract our mock objects to a separate file, so that our test file will contain our test descriptions only. Here's what our two files will finally look like:

```javascript
// mock.js
var TextFrameItem = function(settings) {
  this.settings = settings ? settings : {};
  this.anchor = this.settings.anchor || [100,-100];
  this.left = this.settings.left || 0;
  this.top = this.settings.top || 0;
  this.lines = this.settings.lines || new Lines();
  this.textRange = this.settings.textRange || new TextRange();
  this.remove = function () {};
};

var TextRange = function(settings) {
  this.settings = settings ? settings : {};
  this.characterAttributes = this.settings.characterAttributes || new CharacterAttributes();
  this.duplicate = function (relativeObject,insertionLocation) {};
};

var Lines = function(settings) {
  this.settings = settings ? settings : {};
  this.length = this.settings.length || 1;
  this.parent = '[TextFrame]';
  this.typename = 'Lines';

  var lines = [];
  for (var l = 0; l < this.length; l++) {
    lines.push(new TextRange());
  }
  return lines;
};

var CharacterAttributes = function(settings) {
  this.settings = settings ? settings : {};
  this.leading = this.settings.leading ? this.settings.leading : 15;
};

module.exports = {
  TextFrameItem: TextFrameItem,
  TextRange: TextRange,
  Lines: Lines,
  CharacterAttributes: CharacterAttributes,
};
```
```javascript
// utilitySpec.js
var chai = require('chai'),
 spies = require('chai-spies'),
 expect = chai.expect;
chai.use(spies);

var util = require('../utility.js');
var mock = require('./mock.js');

describe('utility', function () {
  before(function() {
    global.ElementPlacement = {
      PLACEATBEGINNING: {}
    };

    global.app = {
      activeDocument: {
        textFrames: {
          add: function() {
            return new mock.TextFrameItem();
          }
        }
      }
    };
  });

  describe('copyTextToPosition', function () {
    it('should create a copy of a TextRange in position and duplicate content', function() {
      var text = new mock.TextRange();

      var add = chai.spy.on(app.activeDocument.textFrames, 'add'),
        duplicate = chai.spy.on(text, 'duplicate'),
        doc = app.activeDocument;

      // call our utility function like we would do when we use our script.
      var newText = util.copyTextToPosition(doc, text, 100, 100);

      // EXPECTATIONS:
      expect(add).to.have.been.called();
      expect(newText.left).to.equal(100);
      expect(newText.top).to.equal(100);
      expect(duplicate).to.have.been.called();
    });
  });

  describe('createTextFramesFromLines', function () {
    it('should create new TextFrame for every line', function() {
      // mock a TextFrame with 7 lines and a leading of 20 at x:50, y:100
      var mockTextFrame = new mock.TextFrameItem({
        lines: new mock.Lines({
          length: 7,
        }),
        textRange: new mock.TextRange({
          characterAttributes: new mock.CharacterAttributes({
            leading: 20,
          }),
        }),
        left: 50,
        top: 100,
      });

      var copyText = chai.spy.on(util, 'copyTextToPosition'),
        remove = chai.spy.on(mockTextFrame, 'remove'),
        doc = app.activeDocument;

      // call our utility function
      var newTextFrames = util.createTextFramesFromLines(app.activeDocument, mockTextFrame);

      // EXPECTATIONS:
      // should call copyTextToPosition function for every line
      expect(copyText).to.have.been.called.exactly(7);

      // should position new TextFrames correctly
      expect(newTextFrames[0].left).to.equal(50);
      expect(newTextFrames[0].top).to.equal(100);
      expect(newTextFrames[6].top).to.equal(-20);

      // should remove the original TextFrame
      expect(remove).to.have.been.called();
    });
  });
});
```
Congratulations, we're done! By now we have learned how to modularize our code, breaking it into small, focused functions that are covered with tests. That means we now have a solid codebase on which to rely for future script projects. We can reuse and extend our utility collection with every new script we design.

There's no more copying and pasting blocks of code from older scripts, just trusted and tested utility functions we can reuse without a worry.



[url-beginner]: http://www.example.com/
[url-expect]: http://chaijs.com/guide/styles/#expect
[url-assert]: https://nodejs.org/api/assert.html