---
title: Developing for the browser
description:
  In a recent blog post [Andy Mckay asked if javascript apps are ready for
  business](http://www.agmweb.ca/blog/andy/2350). Having recently spent a fair
  amount of writing JavaScript I think I can answer that question.
  Unfortunately the answer at this point in time is a resounding *no*.
---
In a recent blog post [Andy Mckay asked if javascript apps are ready for
business](http://www.agmweb.ca/blog/andy/2350). Having recently spent a fair
amount of writing JavaScript I think I can answer that question. Unfortunately
the answer at this point in time is a resounding *no*.

Platform incompatibilities
--------------------------

The most obvious problem that anyone who develops for a browser runs into is
that no two browsers are the same. For a developer that means that you develop
your app for the browser you are using, and when you are finished you get to
spend many days trying to get the same results in other browsers. This
generally means you need to debug CSS, modify your Javascript to work around
differences in DOM implementations, try to see why minified code runs on one
browser but not the other, etc. It is like having to write an application that
runs on not one or two but five different versions of Windows, and for each
version you must use a completely different set of debugging tools.

Now at this point people will say that modern browsers are much better at
standards compliance so this pain should just go away. I am sceptical of this.
First it will take a long time before enough people are using those
modern browsers that we can stop having to support the older browsers. How bad
this is will depend on your specific audience: if your target audience works in
large organisations that only update their standard desktop every couple of
years, or if they are in a country where people generally do not upgrade their
software (I'm looking at you China) you may be in for a long wait. I can
recommend looking at the excellent [Can I Use...](http://caniuse.com/) site
to check what your audience can support - the result is likely to be
depressing.

Furthermore I am not yet convinced that this standardisation will really work
out. Right now we are seeing that support for CSS 2, ECMAScript 5 and HTML 4
DOM is reaching a point where we can mostly assume they work on all browsers.
Those however are old standards and no longer suffice. For modern interfaces
and apps we need things like Web Storage, WebSockets, CSS animations, HTML 5
input types and other technologies which are still far from being supported by
all modern browsers, and if they are supported there are still way too many
inconsistencies. That means all the compatibility problems are still here
today; they just moved from the standards of five years ago to the standards of
last year.

I know that there are tools out there to help with this such as
[Modernizr](http://modernizr.com/),
[-prefix-free](http://leaverou.github.com/prefixfree/),
[jquery-placeholder](https://github.com/mathiasbynens/jquery-placeholder),
[Reset CSS](http://meyerweb.com/eric/tools/css/reset/),
[ExplorerCanvas](http://excanvas.sourceforge.net/) and many others. The fact
we must use these for every thing we do is a sign that things are really bad,
not a sign that things are going well. None of them should be needed.


No useful toolchain
-------------------

There is no decent way to manage dependencies between javascript files. Even
the terminology in this area seems to be unclear: people use terms like
widgets, modules, packages and libraries seemingly without defining exactly
what they are. I'll use the term *module* here. The most common approach is
that you write some javascript for your specific app and you manually include
dependencies that you need such as jQuery, jQuery tools, Modernizr, etc. When
you releasing your app you will then generally want to combine and minimise these
to generate a single file to reduce the number of HTTP requests a browser has
to make. There are many different tools to do that, all of them producing
slightly different results and everyone appears to just write their own scripts
or makefiles to manage that process.

[RequireJS](http://requirejs.org/) improves this situation by providing
a framework for declaring dependencies between javascript modules, loading
javascript modules on-demand as needed and supporting bundling and minimisation.
You do this by wrapping all your code with something like this:

```javascript
define([
    'require',
    'jquery.placeholder'
], function(require) {
    ...
});
```

RequireJS exposes some problems though: there is no standardisation in naming
of javascript modules. Your code might require jQuery, but the filename might be
jquery-1.8.2.min.js. Or jquery-1.8.2.js. Or jquery-min.js. You do not want
hardcode those filenames everywhere in your code, so you can tell RequireJS to
use a different filename. You do that with code like this:

```javascript
requirejs.config({
    paths: {
        "jquery": "3rdparty/require-jquery",
        "prefixfree": "3rdparty/prefixfree.min",
        "modernizr": "3rdparty/modernizr-2.0.6",
        "jquery.anythingslider": "3rdparty/jquery.anythingslider",
        "jquery.autoSuggest": "3rdparty/jquery.autosuggest",
        "jquery.fancybox": "3rdparty/jquery.fancybox-1.3.4",
        "jquery.form": "lib/jquery.form/jquery.form",
        "jquery.placeholder": "3rdparty/jquery.placeholder",
        "jquery.tools": "3rdparty/jquery.tools.min"
    }
});
```

Looks better, right? Ah, but there is a new problem lurking here: RequireJS
still assumes a model where you are writing a single application that pulls in
some third party javascript files. What if you are dealing with a larger
library that itself has dependencies and use that in your application? It turns
out that at that point things break down again: RequireJS has no way to merge
the dependencies from our library with those of the application. Currently that
seems to mean that you will either have to load things twice, or the
application needs to be intimately aware of library internals.

This appears to be an unsolvable problem unless there is a way to identify
javascript modules. Since we only have a filename and the filename is not
fixed there is no way to do that. Contrast this with how other languages work:
C/C++ has global library names (the lib*.so or *.dll files on your system),
Java has class paths, C# has assembly names, Python has module paths: all ways
to uniquely identify code. This is used by their linkers or interpreters to
resolve all dependencies and make sure everything needed is loaded, and never
loaded more than one time.

What is desperately needed is a standard toolchain for javascript that does
the following:

* Give a unique way to identify javascript modules.
* Allow you to specify dependencies between modules.
* Can generate single-file builds that include all dependencies.
* Can deal with single-file builds of libraries that include modules that are
  also included elsewhere.

This problem has been solved for (almost) every other language. It is time that
someone solves this for the javascript as well.


Tracking down errors
--------------------

When you are writing or deploying an application fixing errors is always
important. This can be broken down in two phases: during development you
use tests and debuggers, and after deployment you can use error logging.

The test situation is luckily pretty good now: with test frameworks such as
[Jasmine](http://pivotal.github.com/jasmine/) and [QUnit](http://qunitjs.com/)
and headless browsers like [PhantomJS](http://phantomjs.org/) all the necessary
tools are available.

Debugging is bit more problematic: Internet Explorer, Chrome and Safari include
a decent debugger. Firefox requires the third party Firebug extension which is
powerful but unfortunately has a slight tendency to crash the browser. Since
every browser version is essentially a different platform you will need to be
proficient with all debugging tools of all browsers. Doable, but annoying. This
only works during development: as soon as you start using minified code
debugging becomes almost impossible since you no longer have readable source
code. And it happens just a bit too often that something only breaks when
running after minimisation.

Error logging is very, very useful: by catching all errors and sending them
to a central log collection system you can get a great view of where your
software is breaking and why. There simply is no substitute for this. This is
also an area where browser support is minimal to non-existing: they do not
provide any way to catch unhandled errors in any meaningful way. You can
register a window.onerror handler but the only information you get is the URL
and a line number where the error happened. And since you always deploy with
minimised files this has no useful information at all. The alternative is that
you use lots of ```try..catch``` blocks in your code. This means many changes
throughout your code. Unfortunately browsers again do not give you much
information: you only get the exception object but no stack trace, which
makes it much much harder to figure out what went wrong.


Summary
-------

At this moment I can honestly say that after working on web apps for an entire
day I sometimes long for the time years ago when I was writing Windows
applications using
[MFC](http://en.wikipedia.org/wiki/Microsoft_Foundation_Class_Library). And
that is a very sad thought.
